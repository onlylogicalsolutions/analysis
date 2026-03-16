# PRD Idempotency Key Workflow

## Short answer to your doubt

Yes, every new Step Functions run gets a new execution ARN.
That is normal.

No, this does not create duplicates if you reuse the same idempotencyKey for retries.

Duplicate prevention must be based on idempotencyKey, not execution ARN.

## Why execution ARN changes every run

execution ARN identifies one specific Step Functions execution instance.
If you call StartExecution again, AWS creates a new execution instance, so the ARN changes.

Think of it this way:

- execution ARN = run identifier
- idempotencyKey = business request identifier

## Correct anti-duplicate design

Use idempotencyKey as the stable key for one business deployment request.

Rules:

1. Client or backend generates idempotencyKey once for a deployment request.
2. Save it immediately in DB with status PENDING.
3. Pass the same idempotencyKey to Step Functions input.
4. On retry, send the same idempotencyKey again.
5. Backend checks DB by idempotencyKey before starting a new deployment call.

If the key is reused, the system returns existing result or continues polling.
If a new key is generated, system treats it as a new request and may deploy again.

## End-to-end workflow

1. API request arrives to FastAPI.
2. FastAPI reads client idempotencyKey.
3. If key not provided, FastAPI generates one and returns it to client.
4. FastAPI checks deployments table by idempotencyKey.
5. Case A: record status SUCCEEDED
6. Return existing contract result, do not deploy again.
7. Case B: record status PENDING or TOKENY_SENT
8. Return execution status, continue poll flow, do not call deploy again.
9. Case C: no record found
10. Insert new row with idempotencyKey.
11. Start Step Functions execution, receive new execution ARN.
12. Store execution ARN in executions table for tracing.
13. Continue Tokeny deploy flow.

## Important clarification

A retry can have:

- Same idempotencyKey
- Different execution ARN

This is still safe because DB lookup is keyed by idempotencyKey.

## Decision table

| Retry scenario | execution ARN | idempotencyKey | Duplicate risk |
| -------------- | ------------- | -------------- | -------------- |
| Proper retry   | New           | Same           | No             |
| Wrong retry    | New           | New            | Yes            |

## Recommended key strategy

Use one stable key per business request, for example:

- UUID created once at first submit, then reused by client for retries
- Or deterministic hash from business fields like projectId + tokenSymbol + network + requestDateBucket

Practical recommendation:

- UI generates UUID once at first click
- UI stores it until terminal status
- Any retry uses same key

## DB constraints to enforce safety

1. Unique index on idempotencyKey in deployment status table.
2. Optional business unique constraint for extra safety, example:
   project_id + token_symbol + network + version

This protects against accidental new keys for the same intended deployment.

## Pseudologic

FastAPI receive request
If idempotencyKey exists in DB
If status SUCCEEDED
return previous result
Else
return in-progress state
Else
insert new idempotency row
start Step Functions execution
store execution ARN
return execution ARN + idempotencyKey

## What to store in each table

executions table:

- execution_arn
- idempotency_key
- status
- started_at
- updated_at

idempotency_records table:

- idempotency_key
- status IN_PROGRESS or COMPLETED
- response_data
- expires_at

step_events table:

- execution_arn
- step_name
- input
- output
- error_details

## Common mistake to avoid

Mistake:
Generate new idempotencyKey on every retry.

Impact:
System thinks each retry is a new deployment request.
Possible duplicate token deployments.

Fix:
Always reuse first idempotencyKey until the request reaches final state.

## Final conclusion

New execution ARN per retry is expected and not a problem.

The real duplicate guard is reusing the same idempotencyKey across retries and checking DB by that key before any deploy call.

---

## First-time deployment: table insert order with sample records

Scenario: User clicks "Deploy Token" for the very first time.
Assumed values used throughout this example:

```
idempotencyKey : "a1b2c3d4-0000-0000-0000-111122223333"
projectId      : "proj-9999-aaaa-bbbb"
executionArn   : "arn:aws:states:ap-southeast-1:123456789012:execution:TokenDeployStateMachine:deploy-20260316-001"
transactionId  : "tx_tokeny_abc123"       ← returned by Tokeny API
contractAddress: "0xDEADBEEF..."           ← returned after polling SUCCESS
```

---

### Step 1 — FastAPI receives request

FastAPI looks up `idempotency_records` by idempotencyKey.
Result: no row found → this is a brand new request.

**First INSERT → `idempotency_records`**

```sql
INSERT INTO idempotency_records (idempotency_key, status, response_data, expires_at)
VALUES (
  'a1b2c3d4-0000-0000-0000-111122223333',
  'IN_PROGRESS',
  NULL,
  NOW() + INTERVAL '24 hours'
);
```

| idempotency_key                      | status      | response_data | expires_at             |
| ------------------------------------ | ----------- | ------------- | ---------------------- |
| a1b2c3d4-0000-0000-0000-111122223333 | IN_PROGRESS | null          | 2026-03-17 10:00:00+00 |

> **Why first?** Lock the key immediately to prevent any parallel duplicate request from sneaking in before Step Functions even starts.

---

### Step 2 — FastAPI starts Step Functions execution

AWS returns a new `executionArn`.

**Second INSERT → `executions`** (Lambda: `pg-initialize-deployment`)

```sql
INSERT INTO executions (execution_arn, project_id, status, current_step, input_payload, started_at, updated_at)
VALUES (
  'arn:aws:states:ap-southeast-1:123456789012:execution:TokenDeployStateMachine:deploy-20260316-001',
  'proj-9999-aaaa-bbbb',
  'RUNNING',
  'InitializeRecord',
  '{"idempotencyKey":"a1b2c3d4-...","tokenName":"Real Estate Fund","symbol":"REF1","network":"polygon"}',
  NOW(),
  NOW()
);
```

| execution_arn                          | project_id          | status  | current_step     | started_at             |
| -------------------------------------- | ------------------- | ------- | ---------------- | ---------------------- |
| arn:aws:states:...:deploy-20260316-001 | proj-9999-aaaa-bbbb | RUNNING | InitializeRecord | 2026-03-16 10:00:00+00 |

---

### Step 3 — Lambda calls Tokeny API, receives transactionId

**Third INSERT → `step_events`** (Lambda: `tokeny-api-deploy`)

```sql
INSERT INTO step_events (event_id, execution_arn, step_name, input, output, error_details, retry_count)
VALUES (
  gen_random_uuid(),
  'arn:aws:states:...:deploy-20260316-001',
  'CallTokenyAPI',
  '{"tokenName":"Real Estate Fund","symbol":"REF1"}',
  '{"transactionId":"tx_tokeny_abc123"}',
  NULL,
  0
);
```

**UPDATE `executions`** to log transactionId (Lambda: `pg-update-tx-id`)

```sql
UPDATE executions
SET current_step = 'LogTransactionId', updated_at = NOW(),
    output_payload = '{"transactionId":"tx_tokeny_abc123"}'
WHERE execution_arn = 'arn:aws:states:...:deploy-20260316-001';
```

---

### Step 4 — Lambda polls Tokeny status (repeats until SUCCESS)

Each poll writes a row to `step_events`.

```sql
INSERT INTO step_events (event_id, execution_arn, step_name, input, output, error_details, retry_count)
VALUES (
  gen_random_uuid(),
  'arn:aws:states:...:deploy-20260316-001',
  'PollStatus',
  '{"transactionId":"tx_tokeny_abc123"}',
  '{"status":"PENDING"}',
  NULL,
  1   -- second poll attempt
);
```

---

### Step 5 — Tokeny returns SUCCESS, Lambda finalizes

**UPDATE `executions`** to SUCCEEDED (Lambda: `pg-finalize-deployment`)

```sql
UPDATE executions
SET status = 'SUCCEEDED',
    current_step = 'FinalizeDeployment',
    output_payload = '{"contractAddress":"0xDEADBEEF...","blockNumber":12345678}',
    updated_at = NOW()
WHERE execution_arn = 'arn:aws:states:...:deploy-20260316-001';
```

**INSERT final `step_events` row**

```sql
INSERT INTO step_events (event_id, execution_arn, step_name, input, output, error_details, retry_count)
VALUES (
  gen_random_uuid(),
  'arn:aws:states:...:deploy-20260316-001',
  'FinalizeDeployment',
  '{"transactionId":"tx_tokeny_abc123"}',
  '{"contractAddress":"0xDEADBEEF...","blockNumber":12345678}',
  NULL,
  0
);
```

**UPDATE `idempotency_records`** to COMPLETED, cache the result

```sql
UPDATE idempotency_records
SET status = 'COMPLETED',
    response_data = '{"contractAddress":"0xDEADBEEF...","transactionId":"tx_tokeny_abc123"}'
WHERE idempotency_key = 'a1b2c3d4-0000-0000-0000-111122223333';
```

---

### Insert order summary

| Order | Table                          | Lambda                          | Reason                                          |
| ----- | ------------------------------ | ------------------------------- | ----------------------------------------------- |
| 1     | `idempotency_records`          | FastAPI (before Step Functions) | Lock key to block duplicates immediately        |
| 2     | `executions`                   | `pg-initialize-deployment`      | Register execution run + initial RUNNING status |
| 3+    | `step_events`                  | Every Lambda step               | Audit log one row per step                      |
| Last  | `executions` (UPDATE)          | `pg-finalize-deployment`        | Write final SUCCEEDED + contractAddress         |
| Last  | `idempotency_records` (UPDATE) | `pg-finalize-deployment`        | Cache result, mark COMPLETED                    |

---

### Full first-time flow diagram

```
User clicks Deploy
       ↓
FastAPI generates idempotencyKey (UUID v4)
       ↓
INSERT idempotency_records  (status = IN_PROGRESS)   ← Table 3 first
       ↓
StartExecution → AWS returns execution_arn
       ↓
INSERT executions            (status = RUNNING)       ← Table 1
       ↓
Call Tokeny API → receive transactionId
       ↓
INSERT step_events (CallTokenyAPI)                    ← Table 2
UPDATE executions  (log transactionId)
       ↓
Poll Tokeny every N seconds
       ↓
INSERT step_events (PollStatus) per attempt           ← Table 2 repeated
       ↓
Tokeny returns SUCCESS
       ↓
UPDATE executions  (SUCCEEDED + contractAddress)      ← Table 1 final
INSERT step_events (FinalizeDeployment)               ← Table 2 final
UPDATE idempotency_records (COMPLETED + cached result) ← Table 3 final
       ↓
FastAPI returns contractAddress to client
```

---

## Should idempotencyKey live in a separate table?

### Short answer: Yes — store it in `executions`, not only in `idempotency_records`

`idempotency_records` is a **lock/cache table** — it is only alive during one step execution, has an `expires_at` TTL, and gets cleaned up.  
`executions` is a **permanent business record** — it lives forever for audit and status queries.

**Recommended design:**

| Table                 | Contains idempotencyKey?           | Why                                                         |
| --------------------- | ---------------------------------- | ----------------------------------------------------------- |
| `executions`          | Yes — add `idempotency_key` column | Permanent lookup: "has this project already been deployed?" |
| `idempotency_records` | Yes — primary key                  | Short-lived per-step lock and cache                         |

Add this column to `executions`:

```sql
ALTER TABLE executions ADD COLUMN idempotency_key VARCHAR(255) UNIQUE NOT NULL;
CREATE UNIQUE INDEX idx_executions_idempotency_key ON executions(idempotency_key);
```

Then on every API request, FastAPI queries `executions` first:

```sql
SELECT status, output_payload
FROM executions
WHERE idempotency_key = 'a1b2c3d4-0000-0000-0000-111122223333';
```

If found → return existing result or resume polling.  
If not found → safe to start a new deployment.

### Why NOT rely solely on `idempotency_records`

| Concern        | `idempotency_records` only                                                | `executions` column (recommended)                         |
| -------------- | ------------------------------------------------------------------------- | --------------------------------------------------------- |
| TTL expiry     | Record deleted after 24h → lookup returns "not found" → risk of re-deploy | Permanent — never expires                                 |
| Audit query    | Cannot query "what was deployed for project X?"                           | Can join to project_id easily                             |
| Phase coverage | Only set after FastAPI locks it, before Step Functions                    | Set at `pg-initialize-deployment` — covers full lifecycle |

---

## Is checking idempotencyKey before Step Functions starts correct?

**Yes, this is the correct and recommended pattern.**

Flow:

```
Client sends request (with or without idempotencyKey)
       ↓
FastAPI checks executions WHERE idempotency_key = ?
       ↓
┌─────────────────────────────┬────────────────────────────────────────────────┐
│ Record found                │ Record NOT found                               │
│                             │                                                │
│ status = SUCCEEDED          │ → Safe to proceed                              │
│ → return contractAddress    │ → INSERT idempotency_records (lock)            │
│   no Step Functions call    │ → INSERT executions (RUNNING)                  │
│                             │ → StartExecution → get execution_arn           │
│ status = RUNNING            │ → continue deployment flow                     │
│ → return "in progress"      │                                                │
│   no Step Functions call    │                                                │
│                             │                                                │
│ status = FAILED             │                                                │
│ → return error details      │                                                │
│   client can retry with     │                                                │
│   same key or new key       │                                                │
└─────────────────────────────┴────────────────────────────────────────────────┘
```

**The key principle:** Step Functions is only started if `executions` has no matching `idempotency_key`. This is the guard that prevents duplicate deployments on accidental re-trigger.

---

## How to generate a unique idempotencyKey for a token deployment

### Option 1 — UUID v4 (simplest, recommended for most cases)

Totally random, zero collision risk, no business logic required.

```python
import uuid

idempotency_key = str(uuid.uuid4())
# Example: "a1b2c3d4-5e6f-7890-abcd-ef1234567890"
```

**When to use:** Client generates once at first submit, stores it locally, reuses on retry.

**Risk:** If client loses the key (page refresh, session cleared), they cannot recover the key and may accidentally trigger a new deployment.

---

### Option 2 — Deterministic hash from business fields (safest, recommended for production)

Generate the key from fields that uniquely identify one deployment intent. Even if the client refreshes or reconnects, the same key is always reproduced.

```python
import hashlib

def generate_idempotency_key(project_id: str, token_symbol: str, network: str, version: int) -> str:
    raw = f"{project_id}:{token_symbol}:{network}:v{version}"
    return hashlib.sha256(raw.encode()).hexdigest()

# Example inputs
key = generate_idempotency_key(
    project_id="proj-9999-aaaa-bbbb",
    token_symbol="REF1",
    network="polygon",
    version=1
)
# Always produces the same key for the same inputs
# e.g. "3b4c1a9f2e..."
```

**When to use:** When you want the key to be reproducible — client can regenerate it any time from the same business data.

**Risk:** Two different projects cannot accidentally use the same key because `project_id` is part of the hash.

---

### Option 3 — Prefixed UUID (good for human readability + logs)

```python
import uuid
from datetime import datetime

def generate_idempotency_key(token_symbol: str) -> str:
    date_str = datetime.utcnow().strftime("%Y%m%d")
    unique_part = str(uuid.uuid4())[:8]
    return f"deploy-{token_symbol.lower()}-{date_str}-{unique_part}"

# Example: "deploy-ref1-20260316-a1b2c3d4"
```

**When to use:** When you want human-readable keys visible in logs and dashboards.

---

### Comparison

| Strategy      | Reproducible on retry | Human-readable | Collision risk | Recommended for              |
| ------------- | --------------------- | -------------- | -------------- | ---------------------------- |
| UUID v4       | No                    | No             | Near zero      | Simple flows, short sessions |
| SHA-256 hash  | Yes                   | No             | Extremely low  | Production, multi-session    |
| Prefixed UUID | No                    | Yes            | Near zero      | Dev, staging, demos          |

### Recommended for this project

Use **Option 2 deterministic hash** using:

```
project_id + token_symbol + network + deployment_version
```

`deployment_version` starts at 1 and increments only when a user explicitly wants to redeploy a token that already has a prior SUCCEEDED or FAILED record.

This means:

- Accidental re-triggers always produce the same key → blocked at `executions` check
- Intentional redeployment bumps version → new key → new deployment allowed

# Token Deployment Workflow

## Purpose

This document explains the Tokeny token deployment workflow in a simple sequence so you can implement it in your website.

## Pre-Request Checklist (Before Starting Deployment Workflow)

Complete these pre-requests before calling deployment APIs.

1. Confirm team context.
2. Confirm Tokeny environment (`sandbox` or `production`).
3. Confirm API account role and permissions (Token Creator for deployment).
4. Disable 2FA for agent account if that is required in your API flow.
5. Collect mandatory token fields:

- name
- symbol
- decimals
- network (chain)

6. Collect owner wallet address and verify it is correct.
7. Prepare logo file for multipart upload (`file` key).
8. Decide investor eligibility claim strategy.
9. Decide compliance rule strategy (jurisdiction, supply limit, investor limit, transfer controls).
10. Confirm internal business approval for draft creation and deployment.
11. Ensure your backend can store `tokenId`, `ruleId`, `actionId`, and `taskId`.

If these items are not ready, keep the token in preparation mode and do not start deployment.

## High-Level Workflow

```text
Authenticate
  -> Get networks
  -> Get issuer ID
  -> Create token draft
  -> Set owner
  -> Configure claims
  -> Configure compliance
  -> Deploy token
  -> Monitor deployment status
  -> Store deployed token details
```

## End-to-End Workflow

### Step 1: Authenticate

Use Tokeny credentials to get a JWT.

Endpoint:

`POST /servicing/api/auth/signin`

Output:

- JWT token

Use this JWT in the `Authorization: Bearer <token>` header for later requests.

### Step 2: Get Supported Networks

Fetch available blockchain networks before creating the token.

Endpoint:

`GET /v2/networks`

Output:

- Network name
- Chain ID
- Native currency

You should store the selected `chainId` for token creation.

### Step 3: Get Issuer ID

Token creation requires an issuer context.

Endpoint:

`GET /v2/issuer-portals/{portalId}/issuers/me`

Output:

- `issuerId`

### Step 4: Collect Token Input From Team

Before calling Tokeny, your website should collect:

- Team identifier
- Token name
- Token symbol
- Token decimals
- Network
- Token type
- Base currency
- Logo file
- Owner wallet address
- Optional compliance settings
- Optional claim settings

### Step 5: Create Token Draft

Create the token in Tokeny.

Endpoint:

`POST /v2/assets/servicing/tokens`

Request type:

- `multipart/form-data`

Important:

- Logo file must be sent with key `file`.
- Save the returned `tokenId`.

Locked After Deployment:

- Name
- Symbol
- Network
- Decimals

### Step 6: Set or Confirm Token Owner

Check current owner:

`GET /v2/assets/servicing/tokens/{tokenId}/owner`

Set owner if needed:

`PUT /v2/assets/servicing/tokens/{tokenId}/owner`

Rules:

- Owner controls agents and core token settings.
- Owner should be secured carefully.
- Owner can only be changed before deployment.

### Step 7: Configure Identity Eligibility

This defines which investors are allowed to hold the token.

Read current claims:

`GET /servicing/api/tokens/{tokenId}/required-claims`

Add or remove claims:

`POST /servicing/api/tokens/{tokenId}/required-claims/actions`

Typical logic:

- Investor must have ONCHAINID.
- Investor must have required claim.
- Claim must come from trusted issuer.

Important:

- New tokens may include a default KYC claim.
- If multiple tokens must share the same investor list, align the claim setup across those tokens.

### Step 8: Configure Compliance Rules

Read existing rules:

`GET /servicing/api/tokens/{tokenId}/compliance-rules`

Create rule:

`POST /servicing/api/tokens/{tokenId}/compliance-rules/actions`

Update or delete rule:

`POST /servicing/api/tokens/{tokenId}/compliance-rules/{ruleId}/actions`

Supported rule categories:

- Jurisdiction
- SupplyLimit
- InvestorLimit
- ConditionalTransfer
- TransferWhitelisting
- TransferLimit
- TransferFees
- Custom

Business meanings:

- `SupplyLimit`: maximum total token supply
- `InvestorLimit`: maximum holding per investor
- `Jurisdiction`: allowed countries
- `TransferLimit`: transfer amount restriction over time
- `TransferFees`: fee collection on transfer

Important rule:

- `ConditionalTransfer` and `TransferWhitelisting` cannot both be active at the same time.

### Step 9: Decide if Token Is Ready for Deployment

Before deployment, validate:

- Required fields are correct
- Owner is correct
- Claims are correct
- Compliance rules are correct
- Team approval is complete

If not ready, keep the token as draft in your system.

### Step 10: Deploy Token

Trigger blockchain deployment.

Endpoint:

`POST /v2/assets/servicing/tokens/{tokenId}/actions/deploy`

Important:

- Tokeny signs deployment internally.
- No external signing is needed for this deployment step.

### Step 11: Monitor Deployment Status

Check deployment progress.

Endpoint:

`GET /v2/assets/servicing/tokens/{tokenId}`

Typical statuses:

- `NotDeployed`
- `InProgress`
- `DEPLOYED`
- failure states if deployment does not complete successfully

### Step 12: Save Final Deployment Data

After successful deployment, your system should store:

- Team ID
- Tokeny `tokenId`
- Token name
- Symbol
- Owner wallet
- Network
- Chain ID
- Deployment status
- Onchain token address
- Created date
- Deployed date

## Post-Deployment Workflow

After deployment, token lifecycle continues.

### Claims Update After Deployment

If you update claims after deployment:

1. Call claim action endpoint.
2. Receive action metadata and task ID.
3. Sign blockchain transaction.
4. Broadcast it, or send signed payload to Tokeny.
5. Log transaction hash so Tokeny updates offchain state.

Relevant endpoints:

- `POST /servicing/api/tokens/{tokenId}/required-claims/actions`
- `POST /servicing/api/tokens/{tokenId}/required-claims/actions/{actionId}/tasks/{taskId}/transactions`

### Compliance Update After Deployment

If you update compliance after deployment:

1. Create or update compliance action.
2. Receive action ID, rule ID, and task ID.
3. Sign the blockchain transaction.
4. Broadcast or submit signed payload.
5. Log the resulting transaction.

Relevant endpoints:

- `POST /servicing/api/tokens/{tokenId}/compliance-rules/actions`
- `POST /servicing/api/tokens/{tokenId}/compliance-rules/{ruleId}/actions`
- `POST /servicing/api/tokens/{tokenId}/compliance-rules/{ruleId}/actions/{actionId}/tasks/{taskId}/transactions`

## Blockchain Signing Options

For post-deployment actions that need blockchain signing, you can choose one of these approaches.

### Option 1: Sign and Broadcast Yourself

Use when your app or backend signs and sends the transaction directly.

Flow:

1. Get unsigned payload.
2. Sign it in your frontend or backend.
3. Broadcast transaction.
4. Log `txHash` to Tokeny.

### Option 2: Sign Yourself, Let Tokeny Broadcast

Use when you want control of signing but do not want to manage broadcasting.

Flow:

1. Get unsigned payload.
2. Sign it.
3. Send signed payload to Tokeny.
4. Tokeny broadcasts and returns `txHash`.

### Option 3: Webhook-Based Wallet Provider Flow

Use when your external wallet provider handles signing.

Flow:

1. Subscribe wallet webhook.
2. Trigger Tokeny action.
3. Receive unsigned payload on webhook.
4. Sign through wallet provider.
5. Submit payload back to Tokeny.

## Suggested Website Implementation Workflow

For your project, this is the recommended implementation order.

### Phase 1

- Authentication
- Network retrieval
- Issuer retrieval
- Token draft creation
- Database storage of `tokenId`

### Phase 2

- Owner management
- Claims management
- Compliance management

### Phase 3

- Deploy action
- Status monitoring
- Deployment result storage

### Phase 4

- Post-deploy claim changes
- Post-deploy compliance changes
- Agent operations such as mint, burn, freeze, pause

## Simple Flow for One Team Token

```text
Team submits token form
  -> Backend authenticates with Tokeny
  -> Backend gets chainId and issuerId
  -> Backend creates token draft
  -> Backend sets owner
  -> Backend configures claims
  -> Backend configures compliance
  -> Backend waits for internal approval
  -> Backend triggers deployment
  -> Backend polls deployment status
  -> Backend saves final token address and status
```

## Key Implementation Rules

- Always save `tokenId` immediately after token creation.
- Do not deploy until name, symbol, decimals, and network are confirmed.
- Secure the owner wallet carefully.
- Use sandbox before production.
- Separate pre-deployment flow from post-deployment signed actions.
- Treat each team token as its own workflow instance.

# Token Deployment Use Cases

## Purpose

This document explains practical Tokeny token deployment use cases for your website, where multiple teams manage their own tokens.

Each team can be treated as an issuer context, and each token follows the same Tokeny deployment lifecycle.

## Common Pre-Requests (For All Use Cases)

Before executing any use case, prepare the following:

1. Team and portal context (`teamId`, `portalId`).
2. Environment selection (`sandbox` or `production`).
3. API account with valid deployment permissions.
4. Mandatory token data: name, symbol, decimals, network.
5. Owner wallet address.
6. Token logo file.
7. Initial claims and compliance strategy.
8. Internal approval to create and deploy token.

## Use Case 1: Deploy a New Token for a Team

### Situation

- A team wants to create a brand new token.
- The token does not exist yet on any blockchain.
- The team wants Tokeny to deploy the ERC-3643 token.

### Goal

Create and deploy a token draft, configure owner and compliance, and push the token onchain.

### Typical Example

- Team: Real Estate Team
- Token: Real Estate Growth Fund
- Symbol: REGF
- Network: Polygon
- Decimals: 2

### Flow

1. Authenticate and get JWT.
2. Retrieve supported blockchain networks.
3. Retrieve issuer ID.
4. Create token draft.
5. Set token owner wallet.
6. Configure required claims for investor eligibility.
7. Configure compliance rules.
8. Deploy token.
9. Monitor token deployment status until deployed.

### APIs Used

- `POST /servicing/api/auth/signin`
- `GET /v2/networks`
- `GET /v2/issuer-portals/{portalId}/issuers/me`
- `POST /v2/assets/servicing/tokens`
- `GET /v2/assets/servicing/tokens/{tokenId}/owner`
- `PUT /v2/assets/servicing/tokens/{tokenId}/owner`
- `GET /servicing/api/tokens/{tokenId}/required-claims`
- `POST /servicing/api/tokens/{tokenId}/required-claims/actions`
- `GET /servicing/api/tokens/{tokenId}/compliance-rules`
- `POST /servicing/api/tokens/{tokenId}/compliance-rules/actions`
- `POST /v2/assets/servicing/tokens/{tokenId}/actions/deploy`
- `GET /v2/assets/servicing/tokens/{tokenId}`

### Notes

- Name, symbol, network, and decimals should be finalized before deployment.
- If owner is not set explicitly, token creator becomes owner by default.
- Deployment signing is handled internally by Tokeny.

## Use Case 2: Deploy Tokens for Multiple Teams

### Situation

- Your website has multiple teams.
- Each team may have one or more tokens.
- You need a repeatable API flow for each team token.

### Goal

Standardize token deployment so every team follows the same process.

### Flow Per Team

1. Select the team.
2. Collect token input data.
3. Authenticate with the correct Tokeny account.
4. Create the token draft.
5. Configure owner, claims, and compliance.
6. Deploy the token.
7. Store returned `tokenId`, owner wallet, chain ID, and final token address in your system.

### Recommended Data to Store in Your Application

- Team ID
- Team name
- Token name
- Token symbol
- Token type
- Base currency
- Chain ID
- Network name
- Owner wallet
- Tokeny `tokenId`
- Deployment status
- Onchain token address

### Notes

- Treat each token deployment as an independent workflow.
- Store `tokenId` as the main reference for later Tokeny API operations.
- Keep audit data for who deployed the token and when.

## Use Case 3: Create Token Draft First, Deploy Later

### Situation

- A team is not ready to deploy immediately.
- They want to prepare token details and compliance first.

### Goal

Create the token draft now and deploy later when business approval is complete.

### Flow

1. Create token draft.
2. Set owner.
3. Configure required claims.
4. Configure compliance rules.
5. Save draft status in your application.
6. Deploy later when approved.

### Benefit

- Teams can review token setup before spending gas or creating the final onchain contract.

### Important Point

- Before deployment, changes are easier and cheaper.
- After deployment, some fields are locked and some updates require blockchain signing.

## Use Case 4: Deploy the Same Token on Multiple Networks

### Situation

- A team wants the same token available on multiple blockchains.
- They want the same contract address across networks.

### Goal

Use Tokeny create2-based deployment to keep the same address on different networks.

### Required Matching Fields

- Name
- Symbol
- Decimals
- Owner wallet

### Flow

1. Create token on network A.
2. Create token on network B with the exact same required values.
3. Deploy on both networks.
4. Verify that the resulting token address matches.

### Notes

- If any of the required fields differ, the deployed addresses will differ.
- This is useful for cross-network product consistency.

## Use Case 5: Add Compliance Before Deployment

### Situation

- A team already knows the business rules before launch.
- They want compliance active from day one.

### Goal

Attach compliance rules before deployment so the token launches with those restrictions already defined.

### Possible Rules

- Jurisdiction restrictions
- Total supply limit
- Investor balance limit
- Conditional transfer
- Transfer whitelisting
- Transfer limits
- Transfer fees

### Notes

- This is the cleanest setup for a new token.
- It reduces rework after deployment.

## Use Case 6: Update Claims or Compliance After Deployment

### Situation

- The token is already deployed.
- The owner needs to update claims or compliance rules.

### Goal

Change token rules while preserving onchain and offchain consistency.

### Flow

1. Call the relevant claims or compliance action endpoint.
2. Receive action information and task ID.
3. Sign the blockchain payload.
4. Broadcast the transaction, or send the signed payload to Tokeny.
5. Log the transaction so Tokeny reconciles offchain state.

### APIs Used

- `POST /servicing/api/tokens/{tokenId}/required-claims/actions`
- `POST /servicing/api/tokens/{tokenId}/required-claims/actions/{actionId}/tasks/{taskId}/transactions`
- `POST /servicing/api/tokens/{tokenId}/compliance-rules/actions`
- `POST /servicing/api/tokens/{tokenId}/compliance-rules/{ruleId}/actions`
- `POST /servicing/api/tokens/{tokenId}/compliance-rules/{ruleId}/actions/{actionId}/tasks/{taskId}/transactions`

### Notes

- Post-deployment claim and compliance updates are not fully automatic.
- They require blockchain signature handling.
- You can implement signing in frontend, backend, or webhook-based wallet flow.

## Use Case 7: Agent Manages Token After Deployment

### Situation

- The token is already deployed and live.
- Team operations continue after deployment.

### Goal

Allow agents to manage investors and supply through Tokeny.

### Common Agent Operations

- Authorize investors
- Revoke investors
- Mint tokens
- Burn tokens
- Freeze balances
- Unfreeze balances
- Force transfer
- Pause token
- Unpause token

### Notes

- Deployment is only the beginning of the token lifecycle.
- Operational actions after deployment are usually handled by agents.

## Use Case 8: Sandbox Testing Before Production

### Situation

- You are new to Tokeny.
- You want to validate the workflow before going live.

### Goal

Run full deployment flow in sandbox first.

### Environments

- Sandbox: `https://api-testing.tokeny.com`
- Production: `https://api.tokeny.com`

### Recommendation

1. Test authentication.
2. Test token draft creation.
3. Test owner setup.
4. Test claims and compliance setup.
5. Test deployment monitoring.
6. Only then switch to production.

## Recommended First Implementation for Your Website

For your project, the best first implementation is this:

1. Build one reusable token deployment service.
2. Pass team-specific input into that service.
3. Store `teamId -> tokenId` mapping in your database.
4. Support draft creation first.
5. Add deploy action after draft review.
6. Add claims and compliance update support after basic deployment works.

This gives you a simple and scalable design for multiple teams and multiple tokens.

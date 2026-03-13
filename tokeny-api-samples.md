# Tokeny API Samples

## Purpose

This document gives practical API request examples for token deployment in your website.

These examples are meant to help implementation. For exact request and response schemas, Swagger is still the source of truth.

## Base URLs

- Sandbox: `https://api-testing.tokeny.com`
- Production: `https://api.tokeny.com`

## Pre-Requests Before Running Samples

1. Use the correct environment base URL.
2. Ensure your account has the required role for the API action.
3. Keep `portalId` ready to resolve `issuerId`.
4. Keep token input ready: name, symbol, decimals, network, type, base currency.
5. Keep owner wallet address ready.
6. Keep logo file path ready for multipart upload.
7. Confirm your backend can persist `tokenId` and workflow identifiers.

## Common Headers

Most requests need:

```http
Authorization: Bearer <jwt>
Content-Type: application/json
```

For multipart requests, `Content-Type` is set automatically by the client.

## 1. Sign In and Get JWT

### Request

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/servicing/api/auth/signin \
  --header "Content-Type: application/json" \
  --data "{\"email\":\"you@company.com\",\"password\":\"your-password\"}"
```

### Example Response

```json
{
  "token": "<jwt>"
}
```

## 2. Get Supported Networks

### Request

```bash
curl --request GET \
  --url https://api-testing.tokeny.com/v2/networks \
  --header "Authorization: Bearer <jwt>"
```

### What to Save

- `chainId`
- network name

## 3. Get Issuer ID From Portal ID

### Request

```bash
curl --request GET \
  --url https://api-testing.tokeny.com/v2/issuer-portals/<portalId>/issuers/me \
  --header "Authorization: Bearer <jwt>"
```

### What to Save

- `issuerId`

## 4. Create Token Draft

This request is `multipart/form-data`.

The exact field names should be checked in Swagger, but the business inputs come from the docs: name, symbol, decimals, network, type, base currency, and logo file.

### Example Request

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/v2/assets/servicing/tokens \
  --header "Authorization: Bearer <jwt>" \
  --form "name=Real Estate Growth Fund" \
  --form "symbol=REGF" \
  --form "decimals=2" \
  --form "chainId=137" \
  --form "type=FUND" \
  --form "baseCurrency=EUR" \
  --form "issuerId=<issuerId>" \
  --form "file=@D:/logos/regf.png"
```

### Example Response Shape

```json
{
  "id": "3a369172-cd22-4e54-8f39-d852d716143d",
  "name": "Real Estate Growth Fund",
  "symbol": "REGF",
  "status": "NOT_DEPLOYED"
}
```

### Important

- Save the returned `id` as `tokenId`.
- Name, symbol, decimals, and network should be treated as final before deployment.

## 5. Get Current Token Owner

### Request

```bash
curl --request GET \
  --url https://api-testing.tokeny.com/v2/assets/servicing/tokens/<tokenId>/owner \
  --header "Authorization: Bearer <jwt>"
```

## 6. Set Token Owner

The exact body schema should be confirmed in Swagger. Business input is the owner wallet address.

### Example Request

```bash
curl --request PUT \
  --url https://api-testing.tokeny.com/v2/assets/servicing/tokens/<tokenId>/owner \
  --header "Authorization: Bearer <jwt>" \
  --header "Content-Type: application/json" \
  --data "{\"wallet\":\"0x1BA6791aBb346785c473313A70472bA1BCcf091B\"}"
```

## 7. Read Required Claims

### Request

```bash
curl --request GET \
  --url https://api-testing.tokeny.com/servicing/api/tokens/<tokenId>/required-claims \
  --header "Authorization: Bearer <jwt>"
```

### What to Inspect

- default KYC claim
- trusted issuer list
- claim topic
- current claim status

## 8. Add a Required Claim

The docs define the action model, but exact request fields should be verified in Swagger.

### Example Request

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/servicing/api/tokens/<tokenId>/required-claims/actions \
  --header "Authorization: Bearer <jwt>" \
  --header "Content-Type: application/json" \
  --data "{
    \"action\": \"CREATE\",
    \"topic\": 1,
    \"name\": \"KYC\",
    \"description\": \"Investor must be KYC verified\",
    \"trustedIssuers\": [\"<trustedIssuerId>\"]
  }"
```

### Example Response Shape

```json
{
  "actionId": "<actionId>",
  "taskId": "<taskId>"
}
```

## 9. Read Compliance Rules

### Request

```bash
curl --request GET \
  --url https://api-testing.tokeny.com/servicing/api/tokens/<tokenId>/compliance-rules \
  --header "Authorization: Bearer <jwt>"
```

## 10. Add Jurisdiction Rule

This shape is directly based on the documentation.

### Example Request

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/servicing/api/tokens/<tokenId>/compliance-rules/actions \
  --header "Authorization: Bearer <jwt>" \
  --header "Content-Type: application/json" \
  --data "{
    \"complianceRule\": {
      \"type\": \"Jurisdiction\",
      \"configuration\": {
        \"allowedCountries\": [\"LUX\", \"FRA\", \"ESP\"]
      }
    }
  }"
```

## 11. Add Total Supply Limit Rule

### Example Request

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/servicing/api/tokens/<tokenId>/compliance-rules/actions \
  --header "Authorization: Bearer <jwt>" \
  --header "Content-Type: application/json" \
  --data "{
    \"complianceRule\": {
      \"type\": \"SupplyLimit\",
      \"configuration\": {
        \"amount\": \"1000000.00\"
      }
    }
  }"
```

### Meaning

- This is where you define the maximum total token supply.
- The `amount` precision must match the token decimals.

## 12. Add Investor Limit Rule

### Example Request

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/servicing/api/tokens/<tokenId>/compliance-rules/actions \
  --header "Authorization: Bearer <jwt>" \
  --header "Content-Type: application/json" \
  --data "{
    \"complianceRule\": {
      \"type\": \"InvestorLimit\",
      \"configuration\": {
        \"amount\": \"50000.00\"
      }
    }
  }"
```

## 13. Add Transfer Fees Rule

### Example Request

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/servicing/api/tokens/<tokenId>/compliance-rules/actions \
  --header "Authorization: Bearer <jwt>" \
  --header "Content-Type: application/json" \
  --data "{
    \"complianceRule\": {
      \"type\": \"TransferFees\",
      \"configuration\": {
        \"fee\": \"2.00\",
        \"collectorWallet\": \"0x8fD26370598e1845b9C4972E4379a006968b83FE\"
      }
    }
  }"
```

## 14. Deploy Token

### Request

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/v2/assets/servicing/tokens/<tokenId>/actions/deploy \
  --header "Authorization: Bearer <jwt>"
```

### Important

- Deployment signing is handled by Tokeny.
- You still need to monitor status after this call.

## 15. Monitor Deployment Status

### Request

```bash
curl --request GET \
  --url https://api-testing.tokeny.com/v2/assets/servicing/tokens/<tokenId> \
  --header "Authorization: Bearer <jwt>"
```

### Example Response Shape

```json
{
  "id": "<tokenId>",
  "name": "Real Estate Growth Fund",
  "symbol": "REGF",
  "status": "DEPLOYED",
  "address": "0xTokenContractAddress"
}
```

## 16. Get My Tokens and Token IDs

### Request

```bash
curl --request GET \
  --url https://api-testing.tokeny.com/v2/assets/servicing/tokens/me \
  --header "Authorization: Bearer <jwt>"
```

### Why This Matters

- This returns the list of tokens visible to the current account.
- The `id` field is your Tokeny `tokenId`.

## 17. Post-Deployment Claim Update With Logged Transaction

If the token is already deployed, claim changes need blockchain signature handling.

### Step A: Create Claim Action

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/servicing/api/tokens/<tokenId>/required-claims/actions \
  --header "Authorization: Bearer <jwt>" \
  --header "Content-Type: application/json" \
  --data "{ ... }"
```

### Step B: Sign and Broadcast Transaction

- Sign with your wallet flow.
- Broadcast to blockchain.
- Get `txHash`.

### Step C: Log Transaction

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/servicing/api/tokens/<tokenId>/required-claims/actions/<actionId>/tasks/<taskId>/transactions \
  --header "Authorization: Bearer <jwt>" \
  --header "Content-Type: application/json" \
  --data "{\"txHash\":\"0x123...abc\"}"
```

## 18. Post-Deployment Compliance Update With Logged Transaction

### Step A: Create or Update Compliance Action

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/servicing/api/tokens/<tokenId>/compliance-rules/<ruleId>/actions \
  --header "Authorization: Bearer <jwt>" \
  --header "Content-Type: application/json" \
  --data "{ ... }"
```

### Step B: Sign and Broadcast Transaction

- Sign with your wallet flow.
- Broadcast.
- Get `txHash`.

### Step C: Log Transaction

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/servicing/api/tokens/<tokenId>/compliance-rules/<ruleId>/actions/<actionId>/tasks/<taskId>/transactions \
  --header "Authorization: Bearer <jwt>" \
  --header "Content-Type: application/json" \
  --data "{\"txHash\":\"0x123...abc\"}"
```

## Implementation Notes

- Use sandbox first.
- Save `tokenId` immediately after token creation.
- Keep exact payload schemas aligned with Swagger when building the real integration.
- Treat compliance examples here as implementation templates, not full schema documentation.

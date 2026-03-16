# PRD ONCHAINID API Flow

## ONCHAINID Creation and Investor Qualification Flow

This section covers how to create an ONCHAINID and qualify an investor after GBG KYC verification.

### When This Flow Runs

1. GBG KYC verification has passed for investor.
2. Token is already deployed.
3. Backend triggers investor qualification using Tokeny APIs.

### What the Flow Does

1. Imports investor data and creates ONCHAINID.
2. Creates an investor account on Tokeny.
3. Retrieves investor identity details.
4. Links and verifies investor wallet to ONCHAINID.

---

## 19. Import Investor and Create ONCHAINID

This call creates investor identity, deploys ONCHAINID, and whitelists the investor for the token in one operation.

Use agent JWT for this call. Agent 2FA must be disabled.

### Request

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/servicing/api/tokens/<tokenId>/import-investors \
  --header "Authorization: Bearer <jwt>" \
  --header "Content-Type: application/json" \
  --data "{
    \"holdersArray\": [
      {
        \"Type\": \"individual\",
        \"First Name\": \"Alice\",
        \"Last Name\": \"Smith\",
        \"Gender\": \"Female\",
        \"Date of Birth\": \"15/06/1990\",
        \"Place of Birth\": \"London\",
        \"Nationality\": \"GBR\",
        \"Passport #\": \"GB123456\",
        \"Phone Number\": \"+447911123456\",
        \"Occupation\": \"Software Engineer\",
        \"E-mail Address\": \"alice@example.com\",
        \"PEP\": \"N\",
        \"Building Number\": \"10\",
        \"Street\": \"Baker Street\",
        \"Zip Code\": \"NW1 6XE\",
        \"State\": \"England\",
        \"City\": \"London\",
        \"Country\": \"GBR\",
        \"KYC/AML\": \"Y\",
        \"ONCHAINID T&Cs\": \"Y\",
        \"Wallet Address\": \"0xAliceWalletAddress...\"
      }
    ]
  }"
```

### Key Fields From GBG

After GBG verification, your backend should map these fields from GBG result:

| Your Field      | Tokeny Field     | Notes                            |
| :-------------- | :--------------- | :------------------------------- |
| First name      | `First Name`     |                                  |
| Last name       | `Last Name`      |                                  |
| Date of birth   | `Date of Birth`  | format: DD/MM/YYYY               |
| Nationality     | `Nationality`    | ISO 3-letter                     |
| Passport number | `Passport #`     |                                  |
| Country         | `Country`        | ISO 3-letter                     |
| KYC passed      | `KYC/AML`        | set `Y` only if GBG passed       |
| T&Cs accepted   | `ONCHAINID T&Cs` | investor must accept before call |

### Example Response

```json
{
  "holdersArray": [
    {
      "email": "alice@example.com",
      "holderId": "holder-uuid-001",
      "result": "Success: Identity imported/updated."
    }
  ]
}
```

### What to Save

- `holderId`: used for future identity operations

---

## 20. Create Investor Account on Tokeny

After import, create a Tokeny account for the investor so they can use investor-facing APIs.

Use same email as Step 19.

### Request

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/servicing/api/accounts \
  --header "Authorization: Bearer <jwt>" \
  --header "Content-Type: application/json" \
  --data "{
    \"email\": \"alice@example.com\",
    \"firstName\": \"Alice\",
    \"lastName\": \"Smith\"
  }"
```

### Example Response

```json
{
  "accountId": "account-uuid-001",
  "email": "alice@example.com",
  "status": "CREATED"
}
```

---

## 20.1 Preferred SSO Flow (No Second Password)

Use your own website Identity Provider (OIDC/Cognito) and trade your IdP token for a Tokeny token.
This avoids creating or asking for a second investor password on Tokeny.

1. Investor signs in on your website (IdP).
2. Backend calls Token Trader account endpoint with IdP identity token.
3. Backend calls Token Trader token endpoint with IdP access token.
4. Use returned Tokeny token as `investor-tat` for Step 21 and Step 22.

### SSO Step A: Create/Link Shared Account on Token Trader

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/token-trader/accounts \
  --header "Authorization: Bearer <idp-identity-token>"
```

### SSO Step B: Trade IdP Access Token for Tokeny Access Token (TAT)

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/token-trader/token \
  --header "Authorization: Bearer <idp-access-token>"
```

### Token Trader Response

```json
{
  "token": "<investor-tat>"
}
```

### Important Note

- Agent JWT is used for servicing endpoints.
- investor-tat is used for investor endpoints in Step 21 and Step 22.

---

## 21. Retrieve Investor Identity and ONCHAINID Address

Call this with investor-tat to get identity details including ONCHAINID address.

### Request

```bash
curl --request GET \
  --url https://api-testing.tokeny.com/onchainid/v1/api/identity \
  --header "Authorization: Bearer <investor-tat>"
```

### Example Response

```json
{
  "identityId": "identity-uuid-001",
  "ssoId": "sso-uuid-001",
  "onchainIdAddress": "0xOnchainIdContractAddress...",
  "status": "DEPLOYED",
  "onchainIdAddresses": [
    {
      "address": "0xOnchainIdContractAddress...",
      "status": "DEPLOYED",
      "network": "polygon"
    }
  ]
}
```

### What to Save

- `ssoId`: required for wallet verification in next step
- `onchainIdAddress`: ONCHAINID contract address on blockchain

---

## 22. Verify and Link Investor Wallet to ONCHAINID

Links investor wallet to ONCHAINID with proof of ownership signature.

Investor must sign this message with their wallet:

```
"Proof of ownership: I hereby declare I am the owner of the wallet: 0xAliceWalletAddress..."
```

### Request

```bash
curl --request POST \
  --url https://api-testing.tokeny.com/onchainid/v1/api/wallets \
  --header "Authorization: Bearer <investor-tat>" \
  --header "Content-Type: application/json" \
  --data "{
    \"ssoId\": \"sso-uuid-001\",
    \"walletAddress\": \"0xAliceWalletAddress...\",
    \"provider\": \"USER_MANAGED\",
    \"signature\": \"0xSignedProofOfOwnership...\"
  }"
```

### Example Response

```json
{
  "identityId": "identity-uuid-001",
  "walletAddress": "0xAliceWalletAddress...",
  "provider": "USER_MANAGED",
  "verified": true
}
```

### After This Step

- Investor wallet is linked to ONCHAINID.
- Identity Registry Storage now has wallet -> ONCHAINID mapping.
- Investor is eligible to receive tokens (subject to claim and compliance checks).

---

## Full ONCHAINID Flow Summary

```text
GBG KYC passes
  -> Step 19: POST import-investors  (creates ONCHAINID + whitelists)
  -> Step 20: POST accounts          (creates investor Tokeny account)
  -> Step 20.1: SSO token trade      (get investor-tat from Token Trader)
  -> Step 21: GET identity           (get ssoId + onchainIdAddress)
  -> Step 22: POST wallets           (link + verify wallet)
  -> Investor is now token-eligible
```

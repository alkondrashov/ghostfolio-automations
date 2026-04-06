---
name: ghostfolio
description: Operate the Ghostfolio API - list, create, update, or delete activities (BUY, SELL, FEE, INTEREST, DIVIDEND, etc.), manage accounts, and query portfolio data. Use when the user asks to interact with their Ghostfolio portfolio.
tools: Bash
---

# Ghostfolio API Skill

Interact with the Ghostfolio instance at `https://ghostfol.io`.

## Authentication

Always authenticate first to obtain a JWT token:

```bash
TOKEN=$(curl -s -X POST "https://ghostfol.io/api/v1/auth/anonymous" \
  -H "Content-Type: application/json" \
  -d "{\"accessToken\": \"$GHOSTFOLIO_ACCESS_TOKEN\"}" \
  | jq -r '.authToken')
```

The access token is read from the `GHOSTFOLIO_ACCESS_TOKEN` environment variable. If it's not set, ask the user to provide it.

## Known Accounts

- **Vanguard Stocks & Shares ISA**: `be08db84-cbaf-4e99-ac40-87ce96992ec7`
- **Cash Revolut** (Investec savings, used for interest payments): `13993c80-84eb-4d6d-b42f-778410f5d92d` — currency: USD

## Key Endpoints

### List activities
```bash
curl -s "https://ghostfol.io/api/v1/order?accounts=<accountId>&type=<TYPE>" \
  -H "Authorization: Bearer $TOKEN" | jq '.activities[] | {id, date, type, comment, fee, unitPrice, quantity, "name": .SymbolProfile.name}'
```
Omit `type` to get all activity types. Supported types: `BUY`, `SELL`, `FEE`, `INTEREST`, `DIVIDEND`, `LIABILITY`.

### Create activity (import)
> **Bulk import limit**: The `/api/v1/import` endpoint accepts a maximum of **25 activities per request**. Split larger imports into batches.

```bash
curl -s -X POST "https://ghostfol.io/api/v1/import" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "activities": [{
      "accountId": "<accountId>",
      "currency": "<match account currency>",
      "dataSource": "MANUAL",
      "date": "2026-01-01T00:00:00.000Z",
      "fee": 0,
      "quantity": 1,
      "symbol": "<Name shown in UI>",
      "type": "FEE",
      "unitPrice": 0,
      "comment": ""
    }]
  }'
```

> **Note on Name vs Symbol**: For `MANUAL` dataSource activities, the `symbol` field becomes the display name shown in the Ghostfolio UI under "Name". Use a descriptive string (e.g. `"Account Fee 05-Jun-2021 to 04-Sep-2021"`). The `comment` field is a separate notes field.

> **Note on FEE activities**: The fee amount goes in the `fee` field (not `unitPrice`). Set `quantity: 0` and `unitPrice: 0`.

> **Note on INTEREST activities**: The interest amount = `unitPrice * quantity`. Typically `quantity: 1` and `unitPrice: <amount>`.

### Update account balance
```bash
curl -s -X PUT "https://ghostfol.io/api/v1/account/<accountId>" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "<accountId>",
    "balance": 1234.56,
    "currency": "<account currency>",
    "isExcluded": false,
    "name": "<account name>",
    "platformId": "<platformId>"
  }'
```
The account balance is a single current value (not historical). Always include all fields — it's a full replace.

### Update activity (PUT — replaces all fields)
```bash
curl -s -X PUT "https://ghostfol.io/api/v1/order/<id>" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "<id>",
    "accountId": "<accountId>",
    "currency": "<match account currency>",
    "dataSource": "MANUAL",
    "date": "...",
    "fee": 0,
    "quantity": 0,
    "symbol": "...",
    "type": "FEE",
    "unitPrice": 0,
    "comment": ""
  }'
```
Always include `id` in the body. The `symbol` field is **not** updated via PUT — to change the display name, delete and recreate the activity.

### Delete activity
```bash
curl -s -X DELETE "https://ghostfol.io/api/v1/order/<id>" \
  -H "Authorization: Bearer $TOKEN"
```

### Rename activity (change Name in UI)
The PUT endpoint does not update the symbol/name. To rename:
1. Note all fields from the existing activity
2. DELETE it
3. POST to `/api/v1/import` with the new `symbol` value

## Workflow Guidelines

- **Bulk operations**: Authenticate once, reuse `$TOKEN` across all requests
- **Before destructive operations**: Show the user what will be changed and confirm
- **After updates**: Verify by re-fetching the affected activity
- **Dates**: Always use ISO 8601 format: `2026-01-01T00:00:00.000Z`

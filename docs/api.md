# API Reference

Base URL: `/api` (Spring `server.servlet.context-path`).
Auth: JWT is issued as **HttpOnly cookies** (`access_token`, `refresh_token`) — the browser sends them automatically (`withCredentials`). There is no `Authorization` header to manage on the client.
Every response is wrapped in `ApiResponse<T>` → `{ success, message, data }`.

> Endpoints below are transcribed from the real controllers.

---

## Authentication — `/auth/v1`

| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/v1/register` | Create an account |
| POST | `/auth/v1/login` | Log in; sets the auth cookies |
| POST | `/auth/v1/refresh` | Rotate tokens from the refresh cookie |
| POST | `/auth/v1/logout` | Revoke the current session |
| POST | `/auth/v1/logout-all` | Revoke every session |
| GET | `/auth/v1/me` | Current user info |
| GET | `/auth/v1/login-history` | Recent login events |
| GET | `/auth/v1/sessions` | Active sessions |
| DELETE | `/auth/v1/sessions/{id}` | Revoke one session |
| POST | `/auth/v1/change-password` | Change password |
| POST | `/auth/v1/deactivate` | Deactivate the account |
| POST | `/auth/v1/email/send-verification` | Send an email-verification code |
| POST | `/auth/v1/email/verify` | Verify email |
| POST | `/auth/v1/email/change-request` | Request an email change |
| POST | `/auth/v1/email/change-confirm` | Confirm the new email |
| DELETE | `/auth/v1/me` | Delete the account |

## Users — `/users`

| Method | Endpoint | Description |
|---|---|---|
| GET | `/users` | List users |
| GET | `/users/{id}` | One user |
| PUT | `/users/{id}/profile` | Update profile |
| PUT | `/users/{id}/preferences` | Update preferences |
| DELETE | `/users/{id}` | Delete user |
| GET | `/users/{userId}/export` | Export all user data (file) |
| GET | `/users/{userId}/dashboard-report` | Generated dashboard report (file) |

## Wallets (Cartes)

| Method | Endpoint | Description |
|---|---|---|
| GET | `/users/{userId}/cartes` | List the user's wallets (with computed balances) |
| POST | `/users/{userId}/cartes` | Create a wallet (bank / cash / credit / foreign / mobile / other) |
| GET | `/cartes/{id}` | One wallet |
| PUT | `/cartes/{id}` | Update |
| DELETE | `/cartes/{id}` | Delete |

## Transactions

| Method | Endpoint | Description |
|---|---|---|
| GET | `/cartes/{carteId}/transactions` | Ledger for one wallet |
| GET | `/users/{userId}/transactions` | Ledger for the user |
| GET | `/transactions/all` | All transactions |
| GET | `/transactions/{id}` | One transaction |
| POST | `/users/{userId}/transactions/insertion` | Add income (Insertion) |
| POST | `/users/{userId}/transactions/purchase` | Add a purchase (in-person / online) |
| POST | `/users/{userId}/transactions/obligation` | Add an obligation |
| POST | `/users/{userId}/transactions/transfer` | Transfer between wallets |
| PUT | `/transactions/{id}` | Update |
| DELETE | `/transactions/{id}` | Delete |

## Loans & repayments

| Method | Endpoint | Description |
|---|---|---|
| GET | `/users/{userId}/transactions/loans/{loanId}/repayments` | Repayment schedule |
| POST | `/users/{userId}/transactions/loans/{loanId}/repayments` | Mark an instalment paid |

## Budgets — `/users/{userId}/budgets`

| Method | Endpoint | Description |
|---|---|---|
| GET | `/users/{userId}/budgets` | List (spent + remaining) |
| GET | `/users/{userId}/budgets/{id}` | One budget |
| POST | `/users/{userId}/budgets` | Create |
| PUT | `/users/{userId}/budgets/{id}` | Update |
| POST | `/users/{userId}/budgets/{id}/reset` | Reset the period |
| DELETE | `/users/{userId}/budgets/{id}` | Delete |

## Categories — `/categories`

| Method | Endpoint | Description |
|---|---|---|
| GET | `/categories` | List |
| GET | `/categories/{id}` | One |
| POST | `/categories` | Create |
| PUT | `/categories/{id}` | Update |
| DELETE | `/categories/{id}` | Delete |

## Zakat — `/zakat`

| Method | Endpoint | Description |
|---|---|---|
| GET | `/zakat/nisab` | Current Nisab threshold (live gold price) |
| POST | `/zakat/calculate` | Compute Zakat due |

## Payment methods — `/payment-methods`

| Method | Endpoint | Description |
|---|---|---|
| GET | `/payment-methods` | List |
| GET | `/payment-methods/{id}` | One |
| POST | `/payment-methods` | Create (online / in-real-time) |
| DELETE | `/payment-methods/{id}` | Delete |

## Notifications — `/users/{userId}/notifications`

| Method | Endpoint | Description |
|---|---|---|
| GET | `/users/{userId}/notifications` | All |
| GET | `/users/{userId}/notifications/unread` | Unread |
| GET | `/users/{userId}/notifications/unread/count` | Unread count |
| PATCH | `/users/{userId}/notifications/{id}/read` | Mark one read |
| PATCH | `/users/{userId}/notifications/read-all` | Mark all read |

## Notification preferences — `/users/{userId}/notification-preferences`

| Method | Endpoint | Description |
|---|---|---|
| GET | `/users/{userId}/notification-preferences` | List per-type preferences |
| PUT | `/users/{userId}/notification-preferences` | Update preferences |

## Statistics — `/statistics`

| Method | Endpoint | Description |
|---|---|---|
| GET | `/statistics/dashboard` | Financial summary |
| GET | `/statistics/{userId}/period` | Period summary |
| GET | `/statistics/{userId}/categories` | Category breakdown |
| GET | `/statistics/{userId}/categories/top` | Top category |
| GET | `/statistics/{userId}/budgets` | Budget stats |
| GET | `/statistics/{userId}/loans` | Loan stats |
| GET | `/statistics/{userId}/zakat` | Zakat stats |
| GET | `/statistics/{userId}/trend` | Monthly trend series |

## Files — `/files`

| Method | Endpoint | Description |
|---|---|---|
| POST | `/files/...` (avatar) | Upload a user avatar |
| POST | `/files/...` (transaction) | Upload a transaction image |
| DELETE | `/files/users/{userId}/avatar` | Remove avatar |
| DELETE | `/files/transactions/{kind}/{txId}` | Remove a transaction image |
| GET | `/files/users/{filename}` | Serve a user file |
| GET | `/files/transactions/{kind}/{filename}` | Serve a transaction file |

---

## Status codes

| Code | Meaning |
|---|---|
| 200 / 201 | Success |
| 400 | Validation failed |
| 401 | Missing / expired cookie — the client silently calls `/auth/v1/refresh` once, then retries |
| 403 | Authenticated but not permitted |
| 404 | Not found |
| 409 | Conflict |

## Error shape

```json
{ "success": false, "message": "Validation failed", "data": null }
```

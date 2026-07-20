# Database Schema

Relational schema (MySQL) via **Spring Data JPA**. Identifiers are **String UUIDs**; monetary values are **`BigDecimal`**. Schema changes ship as versioned SQL scripts (`Phase*.sql`).

## Inheritance (single-table)

`Transaction` and `Carte` are single-table hierarchies resolved by a discriminator column.

```
transaction (single table, discriminator = type)
  Insertion(INSERTION) · Transfer(TRANSFER)
  RealWorldPurchase(IN_PERSON) · OnlinePurchase(ONLINE)
  Loan(LOAN) · Zakat(ZAKAT)

carte (single table, discriminator)
  CarteBank(BANK_ACCOUNT) · CarteCash(CASH) · CarteCreditCard(CREDIT_CARD)
  CarteForeignCurrency(FOREIGN_CURRENCY) · CarteMobileWallet(MOBILE_WALLET) · CarteOther(OTHER)
```

## Relationships

```
User ─1:N─ Carte ─1:N─ Transaction ─N:1─ Category
  │                        │  └─N:1─ PaymentMethod
  ├─1:N─ Budget ──────────┘   (Budget → User, Category, Carte)
  ├─1:N─ Notification / NotificationPreference
  ├─1:N─ LoginEvent / RefreshToken
Purchases ─1:1─ Facture       Loan ─1:N─ LoanRepayment
```

## Core tables

### `user`
`id` (UUID), `email`, `password` (BCrypt), `name`, `prname`, `phone`, `image`, role (`@Enumerated STRING`), `city`, `country`, `dateOfBirth`, `gender`, `lastPasswordChange`, email-verification fields (`emailVerificationCode`, `emailVerificationExpires`), pending-email fields (`pendingEmail`, `pendingEmailCode`, `pendingEmailExpires`), `createdAt`, `updatedAt`.

### `carte` (abstract base)
`id`, `name`, `bank` (enum), `accountType` (enum), `initialBalance` (BigDecimal), `notes`, `color` (`WalletColor` enum), `balance` (BigDecimal), `monthlyIntake` (BigDecimal), `openedAt`, `currency`, `createdAt`, `updatedAt`, `user` (N:1), `transactions` (1:N).
Subtype columns: **Bank** `accountNumber`, `rib`, `agency`; **CreditCard** `cardNetwork` (enum), `creditLimit`, `statementDay`, `dueDay`; **Cash** `location`; **ForeignCurrency** `foreignCurrencyCode`, `baseExchangeRate`; **MobileWallet** `provider` (enum), `phoneNumber`; **Other** `customLabel`.

### `transaction` (abstract base)
`id`, `price` (BigDecimal), `title`, `notes`, `reference`, `image`, a status/type enum, `createdAt`, `updatedAt`, `carte` (N:1), `category` (N:1), `paymentMethod` (N:1).
Subtype columns: **Insertion** `source` (enum); **Output** `addressOfOutput`; **Purchases** `name`, `path`, `facture` (1:1); **RealWorldPurchase** `place`; **OnlinePurchase** `website`, `shippingCompany`; **Obligation** `obligationType` (enum), `name`, `phone`; **Loan** `repaymentDate`, `monthlyPayment`, `numberOfMonths`; **Zakat** `zakatDate`; **Transfer** `toCarte` (N:1).

### `loan_repayment`
`id`, `loan` (N:1), `instalmentIndex`, `amount` (BigDecimal), `paidAt`, `paidFromCarte` (N:1), `notes`, `createdAt`. Unique on `(loan, instalmentIndex)`.

### `category`
`id`, `name`, `color`, `icon`, type (enum), `createdAt`, `updatedAt`.

### `budget`
`id`, `limit` (BigDecimal), period (enum), status (enum), `startDate`, `endDate`, `createdAt`, `updatedAt`, `user` (N:1), `category` (N:1), `wallet` → `Carte` (N:1).

### `facture`
`id`, `purchases` (1:1), `name`, `filePath`, `createdAt`, `updatedAt`.

### `payment_method` (abstract)
`id`, `codeId`, `sender`, `receiver`, `createdAt`, `updatedAt`. Subtypes `Online`, `InRealTime`.

### `notification` / `notification_preference`
`notification`: `id`, `user` (N:1), `message`, `type` (enum), `createdAt`, `updatedAt`.
`notification_preference`: `id`, `userId`, `notificationType` (enum), `configJson`, `user` (N:1).

### `login_event`
`id`, `user` (N:1), `occurredAt`, `ipAddress`, `userAgent`, `deviceInfo`.

### `refresh_token`
`id`, `token`, `jwtToken`, `user` (N:1), `expiresAt`, `createdAt`, `deviceInfo`.

---

## Design decisions

**Single-table inheritance for the ledger.** One `transaction` table with a discriminator keeps every money movement queryable together, while each subtype (loan, zakat, transfer, purchases) contributes only its own columns.

**`BigDecimal`, not floating point.** All amounts use exact base-10 decimals, so sums do not drift.

**String UUID identifiers.** Non-sequential IDs are not enumerable.

**Wallet balances.** `Carte` keeps a `balance` (and `monthlyIntake`) maintained alongside the ledger of `transactions` for fast reads.

**Versioned SQL migrations.** Schema evolves through explicit, reviewable `Phase*.sql` scripts.

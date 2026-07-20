# Architecture

## Overview

```
┌───────────────────────────────────────────────────────┐
│  CLIENT  (React 19 + TypeScript, Vite)                │
│  Pages → feature components → TanStack Query hooks     │
│  Axios (withCredentials) → single-flight 401 refresh   │
└────────────────────────┬──────────────────────────────┘
                         │ REST /api, JWT in HttpOnly cookies
┌────────────────────────▼──────────────────────────────┐
│  SECURITY   JwtAuthenticationFilter · CookieService    │
├───────────────────────────────────────────────────────┤
│  CONTROLLERS  validation, HTTP in / HTTP out only      │
├───────────────────────────────────────────────────────┤
│  SERVICES     interface + impl per domain              │
│               budget · statistics · zakat · storage    │
├───────────────────────────────────────────────────────┤
│  REPOSITORIES Spring Data JPA                          │
└────────────────────────┬──────────────────────────────┘
             ┌───────────┼─────────────┐
        ┌────▼────┐  ┌────▼─────┐  ┌────▼────────┐
        │  MySQL  │  │Scheduled │  │Local file   │
        │         │  │  jobs    │  │storage      │
        └─────────┘  └──────────┘  └─────────────┘
```

Backend layering is strict: **controller → service → repository**. Almost every domain exposes a service **interface plus an `Impl`** (`BudgetInterface`/`BudgetService`, `CarteInterface`/`CarteService`, `TransactionInterface`/`TransactionService`, `StatisticsInterface`/`StatisticsService`, …), so controllers depend on abstractions. Controllers validate and map to HTTP; they hold no business logic.

## Modelling the ledger: single-table inheritance

Transactions are not uniform, so the domain uses a **JPA single-table inheritance hierarchy** rooted at the abstract `Transaction`, resolved by a discriminator column:

```
Transaction (abstract)
├── Insertion                    @DiscriminatorValue("INSERTION")   — income
└── Output (abstract)
    ├── Transfer                 @DiscriminatorValue("TRANSFER")    — → toCarte
    ├── Purchases (abstract)     — has a Facture (receipt)
    │   ├── RealWorldPurchase    @DiscriminatorValue("IN_PERSON")   — place
    │   └── OnlinePurchase       @DiscriminatorValue("ONLINE")      — website, shipping
    └── Obligation (abstract)    — obligationType, name, phone
        ├── Loan                 @DiscriminatorValue("LOAN")        — repaymentDate, monthlyPayment, numberOfMonths
        └── Zakat                @DiscriminatorValue("ZAKAT")       — zakatDate
```

Every subtype carries only its own columns while sharing one queryable ledger for balances and analytics. **Wallets (`Carte`) use the same pattern**: an abstract `Carte` with `CarteBank` (`BANK_ACCOUNT`), `CarteCash` (`CASH`), `CarteCreditCard` (`CREDIT_CARD`), `CarteForeignCurrency` (`FOREIGN_CURRENCY`), `CarteMobileWallet` (`MOBILE_WALLET`) and `CarteOther` (`OTHER`).

## Money representation

Money is stored as **`BigDecimal`** throughout (`Transaction.price`, `Carte.initialBalance` / `balance`, `Budget.limit`, `Loan.monthlyPayment`, …). `BigDecimal` gives exact base-10 arithmetic, so amounts never drift the way binary floating point does. Identifiers are **String UUIDs**, not sequential integers, which keeps IDs non-enumerable.

## Authentication — cookies, not headers

Spring Security issues **two JWTs as HttpOnly cookies**: a short-lived `access_token` and a longer-lived `refresh_token`. The `JwtAuthenticationFilter` reads the access cookie on each request; `CookieService`, `JwtService` and `JwtProperties` (under `service/User/JwtAndUserDetailsAndFilter`) manage issuing, signing and clearing them. Because the tokens are HttpOnly they are never exposed to JavaScript.

The client mirrors this: `apiClient` sets `withCredentials: true` and a response interceptor turns a `401` into a **single coordinated refresh** — the first failure calls `POST /auth/v1/refresh`, all concurrent failures queue behind it, and each is retried once the refresh resolves (auth endpoints are bypassed to avoid a loop). Registration supports email verification, and login events / sessions are tracked (`LoginEvent`, `RefreshToken` with `deviceInfo`).

## Request lifecycle — the dashboard

```
GET /api/statistics/dashboard
  → JwtAuthenticationFilter   read access_token cookie, set SecurityContext
  → StatisticsController      validate params
  → StatisticsService         aggregate totals, categories, trend, budget status
  → StatisticsRepository      Spring Data JPA queries
  ← 200 ApiResponse<FinancialSummaryDTO>
```

## Zakat and external prices

Zakat depends on the Nisab threshold, expressed in grams of gold. The `service/Zakat` package isolates this behind a `GoldPriceClient` interface with a concrete `GoldApiIoClient`, wrapped by a `GoldPriceService`. The `ZakatCalculatorService` is deterministic; the price source is swappable.

## Scheduled work

Off-request jobs live in `job/`: `MonthlyReportJob`, `FinancialTipJob` and `OrphanFileCleanupJob` (removing uploaded images no longer referenced by a transaction). Keeping these off the request path keeps user actions fast.

## Storage

`service/storage` provides a `StorageService` with a `LocalFilesystemStorageService`, plus `ImageValidator` and `StoragePathResolver`. Avatars and transaction images are written to disk and served back through the `/files` controllers.

## Security summary

- JWT access + refresh tokens as **HttpOnly cookies**
- BCrypt password hashing
- Ownership scoped to the authenticated user
- Bean Validation on write endpoints
- CORS restricted to the client origin (`CorsConfig`)
- Email verification on registration and email change
- Secrets in environment variables

## Trade-offs I would revisit

- **Aggregation per request.** Fine at personal-ledger scale; a materialised monthly rollup would be the next step.
- **Live gold price only.** Storing the rate at calculation time would make historical Zakat fully reproducible.
- **Local file storage.** Object storage (S3-compatible) would be needed to scale horizontally.
- **In-process scheduler.** Multiple instances would need a distributed lock.

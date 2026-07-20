# Project Structure

The tree below reflects the real codebase. The source itself is private;
this shows how the project is organised.

## Backend — Java / Spring Boot (Maven)

```
api_main/
├── fininacier/                              # Spring Boot application (Maven)
│   ├── pom.xml
│   ├── mvnw / mvnw.cmd
│   └── src/
│       ├── main/
│       │   ├── java/com/example/demo/
│       │   │   ├── FininacierApplication.java
│       │   │   ├── config/
│       │   │   │   ├── Security/
│       │   │   │   │   ├── AuthUtil.java
│       │   │   │   │   └── SecurityConfig.java
│       │   │   │   ├── storage/StorageProperties.java
│       │   │   │   └── zakat/ZakatProperties.java
│       │   │   ├── controller/
│       │   │   │   ├── Auth/AuthController.java
│       │   │   │   ├── File/                    # upload + serving
│       │   │   │   ├── Statistics/StatisticsController.java
│       │   │   │   ├── Transaction/             # Transaction, LoanRepayment
│       │   │   │   ├── User/                     # User, UserExport
│       │   │   │   ├── BudgetController.java
│       │   │   │   ├── CarteController.java      # wallets
│       │   │   │   ├── CategoryController.java
│       │   │   │   ├── NotificationController.java
│       │   │   │   ├── NotificationPreferenceController.java
│       │   │   │   ├── PaymentMethodController.java
│       │   │   │   └── ZakatController.java
│       │   │   ├── dto/                          # Request/Response per domain
│       │   │   │   ├── Auth/  Budget/  Carte/  Category/
│       │   │   │   ├── Transaction/  User/  Zakat/  Notification/
│       │   │   │   ├── Payment/  Period/  Projections/
│       │   │   │   └── API_wrapper/Response/ApiResponse.java
│       │   │   ├── entity/
│       │   │   │   ├── transaction/              # Transaction, Insertion,
│       │   │   │   │                             #   Purchases, Loan, Obligation,
│       │   │   │   │                             #   Transfer, Zakat, Output ...
│       │   │   │   ├── payment/                  # PaymentMethod, Online, InRealTime
│       │   │   │   ├── jwt/RefreshToken.java
│       │   │   │   ├── Carte*.java               # Bank, Cash, CreditCard,
│       │   │   │   │                             #   ForeignCurrency, MobileWallet ...
│       │   │   │   ├── Budget.java  Category.java  Facture.java
│       │   │   │   ├── Notification.java  NotificationPreference.java
│       │   │   │   ├── LoginEvent.java  User.java
│       │   │   ├── enums/                        # AccountType, CarteType, Bank,
│       │   │   │                                 #   BudgetPeriod, Role, Source ...
│       │   │   ├── exception/
│       │   │   │   ├── custom_exceptions/        # BadRequest, Forbidden,
│       │   │   │   │                             #   ResourceNotFound, Unauthorized ...
│       │   │   │   └── global_exception/         # GlobalExceptionHandler, ErrorResponse
│       │   │   ├── job/                          # scheduled: reports, tips, cleanup
│       │   │   ├── repository/                   # Spring Data JPA repositories
│       │   │   │   ├── Transaction/  Statistics/  user/  jwt/
│       │   │   │   └── Budget, Carte, Category, Notification, PaymentMethod ...
│       │   │   └── service/                      # interface + impl per domain
│       │   │       ├── Budget/  Cartes/  Category/  Notification/
│       │   │       ├── Payment/  Statistics/  Transaction/  Export/
│       │   │       ├── storage/                  # local filesystem storage
│       │   │       ├── User/
│       │   │       │   ├── AuthService.java  UserService.java
│       │   │       │   ├── EmailVerificationService.java
│       │   │       │   └── JwtAndUserDetailsAndFilter/   # JWT + filter + cookies
│       │   │       └── Zakat/                    # gold-price client + calculator
│       │   └── resources/application.properties
│       └── test/java/com/example/demo/FininacierApplicationTests.java
├── BugFix_Sweep.sql
├── Phase4_Obligation_Migration.sql
├── Phase5_Carte_Purchase_Migration.sql
├── Phase6_HealthCheck.sql
├── Phase6_Obligation_Split_Migration.sql
├── DOCUMENTATION_EN.md / DOCUMENTATION_AR.md
└── Financier_Postman_Collection.md
```

## Frontend — React + Vite + TypeScript

```
financier-ui/
├── index.html
├── package.json  vite.config.ts  tsconfig*.json
├── components.json                          # shadcn/ui config
├── public/                                  # favicon, icons, default svgs
└── src/
    ├── main.tsx
    ├── index.css
    ├── api/
    │   ├── client.ts                        # axios instance
    │   └── endpoints/                        # auth, transactions, budgets,
    │                                         #   cartes, categories, zakat ...
    ├── components/
    │   ├── ui/                               # shadcn/ui + animated primitives
    │   ├── layout/                           # Sidebar, Topbar, RootLayout,
    │   │                                     #   ProtectedRoute, mobile nav ...
    │   └── animate-ui/
    ├── features/                             # feature-scoped components
    │   ├── auth/  budgets/  categories/  dashboard/
    │   ├── loans/  purchases/  transactions/  wallets/
    │   ├── zakat/  notifications/  settings/  exports/
    ├── hooks/                                # useAuth, useTransactions,
    │                                         #   useBudgets, useZakat ...
    ├── pages/                                # Dashboard, Transactions, Budgets,
    │                                         #   Wallets, Loans, Zakat, Settings ...
    ├── providers/                            # Auth, Query, Direction, Motion
    ├── router/index.tsx
    ├── i18n/                                 # config + ar / en / fr locales
    ├── lib/                                  # currency, presets, utils
    └── types/                                # TS interfaces mirroring the DTOs
```

## Conventions

- Backend is layered controller → service (interface + impl) → repository; controllers stay thin and delegate to services.
- Entities never cross the API boundary; every request and response goes through a DTO under `dto/`.
- Ownership is enforced server-side — queries are scoped to the authenticated user.
- Auth is JWT-based with refresh tokens delivered via HTTP-only cookies.
- Schema changes ship as versioned SQL migration scripts, not auto-DDL.
- The frontend is feature-sliced: each domain owns its components, and shared UI lives in `components/ui`. Server state is fetched through the `api/endpoints` modules and TanStack Query hooks.
- The app is internationalised (Arabic / English / French) with RTL support.

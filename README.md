# My Financier — Personal Finance Manager

> A personal finance application for tracking income, expenses, wallets, budgets, loans and Zakat, with categorised analytics and spending insights. Backend in **Spring Boot**, frontend in **React + TypeScript**.
>
> **This repository is a technical showcase.** It documents the architecture and design decisions. Source code is private and available on request.

---

## Table of Contents
- [Overview](#overview)
- [Screenshots](#screenshots)
- [Tech Stack](#tech-stack)
- [System Architecture](#system-architecture)
- [Data Model](#data-model)
- [Design Patterns](#design-patterns)
- [Key Features](#key-features)
- [Technical Challenges](#technical-challenges)
- [Security](#security)
- [What I Learned](#what-i-learned)
- [Documentation](#documentation)

---

## Overview

Most people know roughly what they earn and have no idea where it goes. Spreadsheets work until you stop updating them.

My Financier keeps the friction low: adding a transaction takes two taps, categories are suggested from previous entries, wallets (cash, bank, cards, mobile money) track themselves, and the dashboard answers the only question that matters — *am I on track this month?* It also includes a **Zakat calculator** backed by live gold prices, and full loan/obligation tracking with repayment schedules.

| | |
|---|---|
| **Role** | Full-stack developer (solo) |
| **Duration** | *[fill in]* |
| **Type** | Personal project |
| **Status** | Functional |

---

## Screenshots

> Screenshots are stored in `docs/images/` in capture order; captions are indicative.

| | |
|:---:|:---:|
| ![Screen](docs/images/Capture%20d%E2%80%99%C3%A9cran%202026-07-20%20231328.png) | ![Screen](docs/images/Capture%20d%E2%80%99%C3%A9cran%202026-07-20%20231541.png) |
| ![Screen](docs/images/Capture%20d%E2%80%99%C3%A9cran%202026-07-20%20231640.png) | ![Screen](docs/images/Capture%20d%E2%80%99%C3%A9cran%202026-07-20%20231725.png) |
| ![Screen](docs/images/Capture%20d%E2%80%99%C3%A9cran%202026-07-20%20231812.png) | ![Screen](docs/images/Capture%20d%E2%80%99%C3%A9cran%202026-07-20%20231907.png) |

*Full gallery in [`docs/images/`](docs/images/).*

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Frontend | React 18 + TypeScript + Vite | Reactive UI for live-updating balances and charts, fast dev/build |
| UI | shadcn/ui (Radix) + Tailwind CSS + Framer Motion | Accessible primitives with polished, animated interactions |
| Server state | TanStack Query + Axios | Caching, background refetch, and derived server state in one place |
| Routing / i18n | React Router + i18next (AR / EN / FR, RTL) | Multi-language, right-to-left aware navigation |
| Backend | Java 17 + Spring Boot (Maven) | Layered services, strong typing, mature security ecosystem |
| Persistence | Spring Data JPA + relational DB (MySQL) | Referential integrity for money; versioned SQL migrations |
| Auth | Spring Security + JWT (access + refresh) | Stateless sessions; refresh token in an HTTP-only cookie |
| Validation | Bean Validation (Jakarta) | Reject bad data at the controller boundary |
| Integrations | Gold-price API (Zakat), local file storage | Live Nisab calculation; image uploads for transactions/avatars |
| Tooling | Git, Postman, Maven Wrapper | Version control, API testing, reproducible builds |

---

## System Architecture

```
┌───────────────────────────────────────────────────────┐
│  CLIENT  (React + TS)                                  │
│  Pages → feature components → TanStack Query hooks     │
│  Totals are DERIVED from server data, never stored     │
└────────────────────────┬──────────────────────────────┘
                         │ REST / JSON  (Bearer JWT + refresh cookie)
┌────────────────────────▼──────────────────────────────┐
│  SECURITY FILTER   JwtAuthenticationFilter · CORS     │
├───────────────────────────────────────────────────────┤
│  CONTROLLERS       validation, HTTP in / HTTP out     │
├───────────────────────────────────────────────────────┤
│  SERVICES          interface + impl per domain        │
│                    budget rules · statistics · zakat  │
├───────────────────────────────────────────────────────┤
│  REPOSITORIES      Spring Data JPA, indexed queries   │
└────────────────────────┬──────────────────────────────┘
             ┌───────────┼─────────────┐
        ┌────▼────┐  ┌────▼─────┐  ┌────▼────────┐
        │ MySQL   │  │ Scheduler│  │ File storage│
        └─────────┘  └──────────┘  └─────────────┘
```

**Core principle: the balance is never stored.** Every wallet balance and category total is derived from the transaction ledger. Storing a running balance would mean two sources of truth that drift apart after the first failed write. Aggregation happens in the database (indexed on user + date), so the dashboard stays fast as the ledger grows.

**Request flow — loading the dashboard:**

```
GET /api/statistics/summary?period=2026-07
  → JwtAuthenticationFilter   verify JWT, set SecurityContext (userId)
  → StatisticsController      parse and validate range
  → StatisticsService         run aggregations
  → StatisticsRepository      grouped sums by category, by day
  ← 200 { totals, byCategory, trend, budgetStatus }
```

---

## Data Model

```
User ──1:N── Carte (wallet) ──1:N── Transaction ──N:1── Category
  │                                     │
  ├──1:N── Budget ──────────────────────┘   (budget scoped to a category + period)
  ├──1:N── PaymentMethod
  ├──1:N── Notification / NotificationPreference
  └──1:N── LoginEvent / RefreshToken
```

`Transaction` is an inheritance hierarchy — `Insertion` (income), `Purchases` (`RealWorldPurchase` / `OnlinePurchase`), `Loan` (+ `LoanRepayment`), `Obligation`, `Transfer` and `Zakat` — so each type carries only the fields it needs while sharing one ledger. `Carte` is likewise specialised into bank, cash, credit-card, foreign-currency, mobile-wallet and other wallet types.

| Entity | Key fields |
|---|---|
| `User` | email, password hash, base currency, preferences |
| `Carte` (wallet) | name, type, colour, opening balance |
| `Transaction` | amount, type, date, category, wallet, note, image |
| `Category` | name, icon, colour, type (income/expense) |
| `Budget` | category, limit, period, status |
| `PaymentMethod` | online / in-real-time payment sources |
| `Zakat` | assets, nisab reference, computed due amount |

**Money is stored as `BigDecimal`** (exact base-10 arithmetic), and identifiers are String UUIDs. `0.1 + 0.2 !== 0.3` is not an acceptable property for a finance app.

Full details in [database-schema.md](docs/database-schema.md).

---

## Design Patterns

| Pattern | Where it's used | Problem it solved |
|---|---|---|
| **Layered architecture** | Backend | Business rules independent of the web framework |
| **Repository** | `repository/` | Query logic in one place; services stay persistence-agnostic |
| **DTO + Mapper** | `dto/` | Response shape is deliberate, not a leaked JPA entity |
| **Inheritance / polymorphism** | `entity/transaction`, `entity/Carte*` | Each transaction and wallet type carries only its own fields |
| **Interface + Impl services** | `service/*` | Controllers depend on abstractions, not concretions |
| **Strategy** | Zakat / gold-price clients | Swappable price source behind one interface |
| **Scheduled jobs** | `job/` | Monthly reports, financial tips and orphan-file cleanup run off-request |
| **Filter / Middleware** | `JwtAuthenticationFilter` | Auth is a cross-cutting concern, composed once |
| **Selector / Memoisation** | Frontend (TanStack Query) | Totals recompute only when server data actually changes |
| **Custom hooks** | Frontend | Logic reuse (`useTransactions`, `useBudgets`, `useZakat`…) |

---

## Key Features

- Multi-wallet tracking (cash, bank, credit card, foreign currency, mobile wallet) with per-wallet and global balances
- Rich transaction types: income, real-world and online purchases, loans with repayment schedules, obligations, transfers
- Hierarchical categories with custom icons and colours
- Monthly budgets per category, with progress rings and threshold alerts
- **Zakat calculator** driven by live gold prices and Nisab thresholds
- Loan and obligation tracking, including overdue detection and upcoming repayments
- Statistics: spend by category, month-over-month trend, income vs expense
- Notifications with per-type user preferences
- Image attachments on transactions; user data export
- Multi-language UI (Arabic / English / French) with full RTL support

---

## Technical Challenges

**Floating-point money.** Solved by using `BigDecimal` for every amount, so decimal arithmetic stays exact.

**One ledger, many transaction shapes.** A loan repayment, a Zakat payment and an online purchase share little beyond amount and date. A JPA inheritance hierarchy keeps them in one queryable ledger while giving each type its own fields and behaviour.

**Live Zakat calculation.** Nisab depends on the current gold price. A dedicated client fetches and caches the rate behind an interface, so the calculator is deterministic and the price source is swappable.

**Dashboard performance.** Summing every transaction client-side is fine at 200 records and unusable at 20,000. Aggregation moved into indexed database queries; response time became flat with respect to ledger size.

**RTL + i18n.** Supporting Arabic meant direction-aware layout, not just translated strings — handled through an i18next config and a direction provider on the client.

---

## Security

- Spring Security with JWT: short-lived access tokens, rotating refresh tokens in an **HTTP-only cookie**
- BCrypt password hashing
- **Ownership checks on every query** — reads and writes are scoped to the authenticated user; a valid token for user A can never reach user B's data by guessing an ID
- Bean Validation on every write endpoint
- CORS locked to the known client origin
- Email verification on registration
- Secrets in environment variables, never committed

---

## What I Learned

- Financial correctness is a data-representation problem before it is a code problem
- Derived state beats stored state — the bug you never have is the one where two numbers disagree
- Modelling a real domain (loans, obligations, Zakat) pushed me to use inheritance deliberately rather than one giant table
- Ownership filtering has to be systematic, not remembered per endpoint
- Aggregation belongs in the database

---

## Documentation

| Document | Contents |
|---|---|
| [Architecture](docs/ARCHITECTURE.md) | Layers, request lifecycle, design patterns, trade-offs |
| [API Reference](docs/api.md) | Every endpoint group, roles, payloads, status codes |
| [Database Schema](docs/database-schema.md) | Entities, indexes, modelling decisions |
| [Frontend](docs/front-end.md) | Client stack, routing, state, data layer |
| [Project Structure](STRUCTURE.md) | Folder layout of the real codebase |
| [.env.example](.env.example) | Required configuration (names only) |

---

## Contact

**BOUBAL ABDELLATIF** — [email] · [LinkedIn] · [Portfolio]

*Source code available on request for recruitment purposes.*

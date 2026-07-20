# Frontend

`financier-ui` — a single-page app in **React 19 + TypeScript**, built with **Vite**.

## Stack

| Concern | Library |
|---|---|
| Framework / build | React 19, Vite 8, TypeScript |
| Routing | react-router-dom 7 (`createBrowserRouter`, lazy routes) |
| Server state | TanStack Query 5 (+ Devtools), Axios |
| Tables | TanStack Table 8 |
| Forms / validation | react-hook-form 7 + `@hookform/resolvers` + Zod 4 |
| UI | Radix UI primitives (shadcn/ui style), Tailwind CSS 4 (`@tailwindcss/vite`) |
| Animation | `motion` (Framer Motion v12), `tw-animate-css` |
| Charts | Recharts 3 |
| i18n | i18next 26 + react-i18next (AR / EN / FR, RTL) |
| Theming | next-themes |
| Toasts / icons / dates | Sonner, lucide-react, date-fns, react-day-picker |
| Fonts | `@fontsource` — Inter, Geist, Cairo, IBM Plex Sans Arabic |

## Provider composition

Set up in `main.tsx`, outermost → innermost:

```
ThemeProvider (next-themes)
└─ QueryProvider (TanStack Query)
   └─ AuthProvider
      └─ DirectionProvider (LTR/RTL)
         └─ BreadcrumbProvider
            └─ MotionProvider
               └─ RouterProvider
```

`QueryProvider` configures the client with `staleTime: 60s`, `refetchOnWindowFocus: false`, `retry: 1`.

## Routing

Two shells (`src/router/index.tsx`):

- **Auth shell** (`AuthLayout`, public): `/login`, `/register`.
- **App shell** (`ProtectedRoute` → `RootLayout`): `/` (Dashboard), `reports`, `wallets`, `transactions`, `purchases`, `budgets` + `budgets/:id`, `categories` + `categories/:id`, `loans`, `zakat`, `notifications`, `settings`, `support` (coming soon).
- Catch-all → `NotFoundPage`.

Pages are **lazy-loaded** for route-level code splitting; the layouts wrap the `Outlet` in `<Suspense>` with a skeleton fallback, so first visits stream a chunk and later visits feel instant.

## Data layer

`src/api/client.ts` creates a single Axios instance:

- `baseURL = VITE_API_URL ?? http://localhost:9075/api`, `withCredentials: true`.
- JWTs live in **HttpOnly cookies** (`access_token`, `refresh_token`) — never touched in JS.
- A response interceptor coordinates a **single refresh** on `401`: the first failure calls `POST /auth/v1/refresh`, concurrent failures queue and retry once it resolves, and auth routes are bypassed to avoid a loop. On hard failure an `onAuthExpired` hook (wired from `AuthProvider`) redirects to login.

Per-domain endpoint modules live in `src/api/endpoints/` (auth, transactions, budgets, cartes, categories, zakat, notifications, statistics, …) and are consumed through TanStack Query hooks in `src/hooks/` (`useTransactions`, `useBudgets`, `useCartes`, `useZakat`, `useStatistics`, …).

## Structure

```
src/
├── api/          client + endpoints/
├── components/   ui/ (Radix primitives) · layout/ · animate-ui/
├── features/     auth · budgets · categories · dashboard · loans ·
│                 purchases · transactions · wallets · zakat ·
│                 notifications · settings · exports
├── hooks/        one query hook per domain
├── pages/        route entry points (lazy-loaded)
├── providers/    Auth · Query · Direction · Motion · Breadcrumb
├── router/       createBrowserRouter config
├── i18n/         config + locales/ (ar · en · fr)
├── lib/          currency, presets, utils
└── types/        TS interfaces mirroring the API DTOs
```

## Conventions

- **Feature-sliced:** each domain owns its components under `features/`; shared primitives live in `components/ui`.
- **Server state stays in TanStack Query**, not in global stores; derived totals come from the server, never duplicated client-side.
- **Forms** are react-hook-form + Zod schemas that mirror the backend's validation.
- **Internationalised and direction-aware:** Arabic support means RTL layout via `DirectionProvider`, not just translated strings.

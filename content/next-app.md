---
id: next-app
title: Build a Full-fledged Next.js App Step by Step
group: Full-stack Next.js
tagline: Build ledger-web, a production-style accounts and transactions app with the App Router, Auth.js, Prisma, server actions, route handlers and a typed API client.
covers: Next.js 16 (App Router, proxy.ts), React 19.2, TypeScript, Auth.js v5 (next-auth@beta), Prisma 7, PostgreSQL, Zod 4, TanStack Query 5, axios, pino, Vitest, Playwright, Docker
status: current
kind: guide
---

## 1. Plan the app

You are going to build **ledger-web**: a full-stack Next.js app where people sign in, open bank-like **accounts**, and record **transactions** that move money in and out. Money is stored as **integer cents**. The same project continues in "Real-time in Next.js with Socket.IO", so keep every name and path exactly as shown.

> **Outdated:** Versions move fast. This guide targets Next.js 16 (16.3.x was the current stable line in early October 2026, with 15.5 in maintenance). In Next.js 16, `middleware.ts` was renamed to `proxy.ts`, runs on the Node.js runtime, and `params`, `searchParams`, `cookies()` and `headers()` are async only. Auth.js v5 is still published as `next-auth@beta`, and the project is now maintained in maintenance mode by the Better Auth team. It works well for this guide, but check its status before you pick it for a brand new product.

### [Beginner] Step 1 — Decide what ledger-web does

Write the feature list before any code. It decides your folders, your tables and your tests.

| Feature | Who | Notes |
|---|---|---|
| Sign up, sign in, sign out | Everyone | Email + password. Optional GitHub or Okta (OIDC) sign-in |
| List, open, rename, archive accounts | Owner | An account can only be archived when its balance is zero |
| Record a credit or debit | Owner | Atomic: transaction row and balance update succeed or fail together. No overdraft |
| Browse transactions | Owner | Filter by type and text, paginate, all reflected in the URL |
| Admin overview | ADMIN role | Users, account counts, totals per currency |
| Public JSON API `/api/v1` | Owner, other clients | Same rules as the UI, consistent error JSON |

Non-goals for now: transfers between accounts, multi-currency conversion, statements. Writing non-goals down stops scope creep.

> **Finance tip:** Never use floating point for money. `0.1 + 0.2 === 0.30000000000000004` in JavaScript. Store integer cents (`1234` means 12.34) and only format at the edge, in the UI.

### [Beginner] Step 2 — Understand the architecture

Next.js is a React framework that also runs server code. In the App Router every component is a **Server Component** by default: it runs on the server, can talk to the database, and sends rendered output (not JavaScript) to the browser. You opt into browser code with `'use client'`.

There are four ways a request reaches your code:

1. **Proxy** (`src/proxy.ts`): runs before every matched request. Good for cheap checks: is there a session cookie, add a request id, redirect.
2. **Server Components** (`page.tsx`, `layout.tsx`): read data and render HTML.
3. **Server Actions** (`'use server'` functions): mutations called from forms and client components. Next.js turns them into POST endpoints for you.
4. **Route Handlers** (`route.ts`): plain HTTP endpoints, here under `/api/v1`, for client-side fetching and external clients.

All four call the same **service layer**, which owns business rules and talks to the database through Prisma.

```mermaid
flowchart TD
  A["Browser"] --> B["proxy.ts<br/>session cookie check, request id"]
  B --> C["Server Components<br/>pages and layouts"]
  B --> D["Server Actions<br/>form mutations"]
  B --> E["Route Handlers<br/>/api/v1"]
  C --> F["Service layer<br/>features/*/service.ts"]
  D --> F
  E --> F
  F --> G["Prisma Client"]
  G --> H["PostgreSQL"]
  A -.->|"TanStack Query + axios"| E
```

> **Why:** Putting the rules in one service layer means a debit is checked the same way whether it comes from a form, the JSON API or a test. The entry points only do three things: authenticate, validate input, call the service.

> **Interview tip:** If asked "where does business logic live in a Next.js app?", answer: "not in components or route files. In plain TypeScript service modules that take the user id as an argument, so they are testable without Next.js."

### [Beginner] Step 3 — Create the project and install dependencies

You need Node.js 20.9 or newer (22 LTS recommended), npm and Docker.

```bash
npx create-next-app@latest ledger-web --ts --app --src-dir --tailwind --eslint --import-alias "@/*" --use-npm
cd ledger-web
```

Turbopack is the default bundler in Next.js 16, so `npm run dev` already uses it.

Install runtime and dev dependencies:

```bash
npm install next-auth@beta @prisma/client @prisma/adapter-pg pg zod bcryptjs pino axios @tanstack/react-query sonner server-only dotenv
npm install -D prisma tsx @types/pg pino-pretty vitest @vitejs/plugin-react vite-tsconfig-paths jsdom @testing-library/react @testing-library/jest-dom @testing-library/user-event @playwright/test
```

> **Gotcha:** If npm reports a peer dependency conflict between `next-auth@beta` and `next@16`, install the newest beta (`npm install next-auth@beta`) first. As a last resort use `--legacy-peer-deps` and pin the versions in `package.json`.

> **Why:** `bcryptjs` is a pure JavaScript bcrypt. The native `bcrypt` package is faster but needs a compiler in your Docker image. For login volumes of a typical app, `bcryptjs` is fine.

Open `package.json` and set the module type and scripts. Prisma 7 expects an ESM project.

```json
{
  "name": "ledger-web",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "next dev",
    "build": "prisma generate && next build",
    "start": "next start",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "db:up": "docker compose up -d db",
    "db:migrate": "prisma migrate dev",
    "db:seed": "prisma db seed",
    "db:studio": "prisma studio",
    "test": "vitest run",
    "test:watch": "vitest",
    "e2e": "playwright test"
  }
}
```

Keep the `dependencies` and `devDependencies` blocks that npm wrote. Only add or change the keys shown.

**Check it works:**

```bash
npm run dev
```

```text
   ▲ Next.js 16.x (Turbopack)
   - Local:        http://localhost:3000
 ✓ Ready in 900ms
```

Open http://localhost:3000 and you see the starter page. Stop the server with Ctrl+C.

### [Beginner] Step 4 — Lay out a feature-based folder structure

Group code by **feature**, not by file type. Everything about accounts lives in `src/features/accounts`. The `app/` folder only holds routes, and route files stay thin.

```text
ledger-web/
├─ docker-compose.yml
├─ prisma.config.ts
├─ prisma/
│  ├─ schema.prisma
│  └─ seed.ts
├─ src/
│  ├─ app/                      routes only: pages, layouts, route handlers
│  │  ├─ (auth)/login/page.tsx
│  │  ├─ (auth)/signup/page.tsx
│  │  ├─ (app)/layout.tsx       protected area
│  │  ├─ (app)/accounts/...
│  │  ├─ (app)/admin/page.tsx
│  │  └─ api/
│  │     ├─ auth/[...nextauth]/route.ts
│  │     └─ v1/...
│  ├─ features/
│  │  ├─ auth/        actions.ts, guards.ts, schemas.ts, types.ts, components/
│  │  ├─ accounts/    actions.ts, queries.ts, schemas.ts, service.ts, dto.ts, components/
│  │  ├─ transactions/ actions.ts, queries.ts, schemas.ts, service.ts, dto.ts, components/
│  │  └─ admin/       service.ts
│  ├─ lib/            db/, api/, api-client/, errors.ts, logger.ts, money.ts, rate-limit.ts
│  ├─ generated/prisma/   generated client, git-ignored
│  ├─ auth.config.ts
│  ├─ auth.ts
│  ├─ env.ts
│  └─ proxy.ts
└─ tests/e2e/
```

Each feature has the same parts:

| File | Runs where | Job |
|---|---|---|
| `components/` | server or client | UI for the feature |
| `actions.ts` | server (`'use server'`) | mutations: auth, validate, call service, revalidate |
| `queries.ts` | server only | reads for Server Components: auth + service, cached per request |
| `schemas.ts` | both | Zod schemas, shared by forms, actions and route handlers |
| `service.ts` | server only | business rules and Prisma calls. Takes `userId` as an argument |
| `dto.ts` | both (types) | the JSON-safe shapes the UI and API use |

Import rules keep the graph clean: `app` may import `features`, `features` may import `lib`, `lib` never imports `features`. A feature may import another feature's `service.ts`, never its components.

```mermaid
flowchart LR
  A["app/ routes"] --> B["features/*"]
  B --> C["lib/"]
  A --> C
  C --> D["generated/prisma"]
  B --> D
```

> **Why:** Small features use one file per concern. When `service.ts` grows past a few hundred lines, turn it into a `service/` folder with an `index.ts`. The import path stays the same.

### [Intermediate] Step 5 — Validate environment variables with Zod

A missing `AUTH_SECRET` should crash the app at startup, not produce a confusing error on the first login. Parse `process.env` once with Zod and import the typed result everywhere.

```ts
// src/env.ts
import { z } from 'zod';

const schema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  DATABASE_URL: z.url(),
  AUTH_SECRET: z.string().min(32, 'AUTH_SECRET must be at least 32 characters'),
  AUTH_URL: z.url().optional(),
  AUTH_GITHUB_ID: z.string().optional(),
  AUTH_GITHUB_SECRET: z.string().optional(),
  AUTH_OKTA_ID: z.string().optional(),
  AUTH_OKTA_SECRET: z.string().optional(),
  AUTH_OKTA_ISSUER: z.url().optional(),
  LOG_LEVEL: z.enum(['fatal', 'error', 'warn', 'info', 'debug', 'trace']).default('info'),
});

export type Env = z.infer<typeof schema>;

function parseEnv(): Env {
  // Docker builds run `next build` without real secrets.
  if (process.env.SKIP_ENV_VALIDATION === '1') {
    return process.env as unknown as Env;
  }
  const result = schema.safeParse(process.env);
  if (!result.success) {
    const fields = z.flattenError(result.error).fieldErrors;
    console.error('Invalid environment variables:', fields);
    throw new Error('Invalid environment variables. See the log above.');
  }
  return result.data;
}

export const env = parseEnv();

export const features = {
  github: Boolean(env.AUTH_GITHUB_ID && env.AUTH_GITHUB_SECRET),
  okta: Boolean(env.AUTH_OKTA_ID && env.AUTH_OKTA_SECRET && env.AUTH_OKTA_ISSUER),
};
```

> **Gotcha:** This file reads secrets. Only import it from server code. Browser code only sees variables prefixed with `NEXT_PUBLIC_`, and Next.js inlines them at build time only when you write `process.env.NEXT_PUBLIC_X` literally.

> **Outdated:** Zod 4 moved string formats to the top level (`z.url()`, `z.email()`) and replaced `error.flatten()` with `z.flattenError(error)`. Zod 3 code with `z.string().url()` still runs but is deprecated.

Create the env files. `.env` is read by both Next.js and the Prisma CLI, so put local values there and keep it out of git.

```bash
# .env.example  (commit this one)
DATABASE_URL="postgresql://ledger:ledger@localhost:5432/ledger?schema=public"
AUTH_SECRET="replace-with-output-of-npx-auth-secret"
# AUTH_GITHUB_ID=""
# AUTH_GITHUB_SECRET=""
# AUTH_OKTA_ID=""
# AUTH_OKTA_SECRET=""
# AUTH_OKTA_ISSUER="https://your-org.okta.com/oauth2/default"
LOG_LEVEL="debug"
```

```bash
cp .env.example .env
openssl rand -base64 33   # paste the output into AUTH_SECRET in .env
printf "\n.env\nsrc/generated\n" >> .gitignore
```

**Check it works:** you will see the validation fire once the database layer imports `env` in Part 2. Remove `AUTH_SECRET` from `.env` at that point and the dev server prints `Invalid environment variables: { AUTH_SECRET: [ 'Invalid input: expected string, received undefined' ] }`. Put it back.

## 2. Database with Prisma and PostgreSQL

### [Beginner] Step 6 — Run PostgreSQL with docker-compose

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: ledger
      POSTGRES_PASSWORD: ledger
      POSTGRES_DB: ledger
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ledger"]
      interval: 5s
      timeout: 3s
      retries: 10
volumes:
  pgdata:
```

**Check it works:**

```bash
npm run db:up
docker compose ps
```

```text
NAME              SERVICE   STATUS
ledger-web-db-1   db        Up 10 seconds (healthy)
```

### [Beginner] Step 7 — Configure Prisma 7 and write the schema

Prisma 7 changed setup: the CLI is configured in `prisma.config.ts`, the client is generated into your source tree, and the client talks to Postgres through a **driver adapter** (`@prisma/adapter-pg`) instead of a bundled Rust engine.

```ts
// prisma.config.ts
import 'dotenv/config';
import { defineConfig, env } from 'prisma/config';

export default defineConfig({
  schema: 'prisma/schema.prisma',
  migrations: {
    path: 'prisma/migrations',
    seed: 'tsx prisma/seed.ts',
  },
  datasource: {
    url: env('DATABASE_URL'),
  },
});
```

Now the data model. Three tables: users, their accounts, and the transactions on each account.

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
}

enum Role {
  USER
  ADMIN
}

enum AccountType {
  CHECKING
  SAVINGS
}

enum TransactionType {
  CREDIT
  DEBIT
}

model User {
  id           String    @id @default(cuid())
  email        String    @unique
  name         String?
  passwordHash String?
  role         Role      @default(USER)
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt
  accounts     Account[]
}

model Account {
  id           String        @id @default(cuid())
  userId       String
  user         User          @relation(fields: [userId], references: [id], onDelete: Cascade)
  name         String
  type         AccountType   @default(CHECKING)
  currency     String        @default("USD") @db.Char(3)
  balanceCents Int           @default(0)
  archivedAt   DateTime?
  createdAt    DateTime      @default(now())
  updatedAt    DateTime      @updatedAt
  transactions Transaction[]

  @@unique([userId, name])
  @@index([userId])
}

model Transaction {
  id                String          @id @default(cuid())
  accountId         String
  account           Account         @relation(fields: [accountId], references: [id], onDelete: Restrict)
  type              TransactionType
  amountCents       Int
  description       String
  balanceAfterCents Int
  createdById       String
  idempotencyKey    String?         @unique
  createdAt         DateTime        @default(now())

  @@index([accountId, createdAt])
}
```

```mermaid
classDiagram
  class User {
    String id
    String email
    String passwordHash
    Role role
  }
  class Account {
    String id
    String userId
    String name
    String currency
    Int balanceCents
    DateTime archivedAt
  }
  class Transaction {
    String id
    String accountId
    TransactionType type
    Int amountCents
    Int balanceAfterCents
    String idempotencyKey
  }
  User "1" --> "many" Account
  Account "1" --> "many" Transaction
```

Design decisions worth saying out loud:

- `passwordHash` is optional because GitHub and Okta users have no password.
- `balanceCents` is stored on the account (a running balance) so reads are cheap. Each transaction also stores `balanceAfterCents`, which makes statements and audits easy.
- `onDelete: Restrict` on transactions: you cannot delete an account that has history. You **archive** it instead.
- `idempotencyKey` is unique so a retried POST cannot create the same payment twice.

> **Finance tip:** A Postgres `Int` holds up to 2,147,483,647 cents, about 21 million in major units. That is fine for this app. A real ledger uses `BigInt` (Postgres `bigint`) or `Decimal`, and then you must convert `bigint` to a string in JSON because `JSON.stringify` cannot serialize it.

> **Gotcha:** Prisma 7 moved the connection URL out of the schema and into `prisma.config.ts`. If your Prisma version still asks for `url = env("DATABASE_URL")` in the `datasource` block, you are on Prisma 6. Either upgrade or add that line.

### [Beginner] Step 8 — Run the first migration and add a safety constraint

```bash
npx prisma migrate dev --name init
```

```text
Applying migration `20261005090000_init`
Your database is now in sync with your schema.
✔ Generated Prisma Client to ./src/generated/prisma
```

Your service will prevent overdrafts, but a database constraint is a second wall that also stops bugs and manual SQL. Prisma has no syntax for `CHECK` constraints, so create an empty migration and write SQL:

```bash
npx prisma migrate dev --create-only --name balance_checks
```

Open the new `prisma/migrations/*_balance_checks/migration.sql` and paste:

```sql
-- prisma/migrations/<timestamp>_balance_checks/migration.sql
ALTER TABLE "Account" ADD CONSTRAINT "Account_balance_non_negative" CHECK ("balanceCents" >= 0);
ALTER TABLE "Transaction" ADD CONSTRAINT "Transaction_amount_positive" CHECK ("amountCents" > 0);
```

```bash
npx prisma migrate dev
```

**Check it works:**

```bash
docker compose exec db psql -U ledger -c '\d "Account"'
```

```text
Check constraints:
    "Account_balance_non_negative" CHECK ("balanceCents" >= 0)
```

### [Intermediate] Step 9 — The Prisma client singleton and the hot-reload gotcha

In development Next.js re-evaluates your modules on every save. If `db.ts` does `new PrismaClient()` at the top level, each save opens a new connection pool and after a few minutes Postgres says `too many clients already`. The fix is to cache the client on `globalThis`, which survives module reloads.

Split it in two files. `client.ts` has no Next.js-only imports, so plain Node scripts (the seed now, the custom Socket.IO server in the next guide) can use it. `index.ts` adds the `server-only` guard for app code.

```ts
// src/lib/db/client.ts
import { PrismaPg } from '@prisma/adapter-pg';
import { PrismaClient } from '../../generated/prisma/client';
import { env } from '../../env';

function createPrismaClient() {
  const adapter = new PrismaPg({ connectionString: env.DATABASE_URL });
  return new PrismaClient({
    adapter,
    log: env.NODE_ENV === 'development' ? ['warn', 'error'] : ['error'],
  });
}

const globalForPrisma = globalThis as unknown as {
  prisma?: ReturnType<typeof createPrismaClient>;
};

export const prisma = globalForPrisma.prisma ?? createPrismaClient();

if (env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}
```

```ts
// src/lib/db/index.ts
import 'server-only';

export { prisma } from './client';
export { Prisma } from '../../generated/prisma/client';
```

> **Why:** `import 'server-only'` makes the build fail if a Client Component imports this file, even indirectly. Without it, a stray import could try to bundle your database code (and secrets) into the browser.

> **Gotcha:** `server-only` throws when imported from plain Node (tsx scripts, Vitest), because those do not use the `react-server` export condition. That is exactly why the seed imports `client.ts` and why Vitest gets an alias in Part 7.

### [Beginner] Step 10 — Money helpers

Parse user input to cents with string maths, never `parseFloat(x) * 100` (try `parseFloat('19.99') * 100`).

```ts
// src/lib/money.ts
const AMOUNT_RE = /^\d{1,9}(\.\d{1,2})?$/;

/** "12.3" -> 1230, "1,000.05" -> 100005, invalid -> null */
export function parseAmountToCents(input: string): number | null {
  const s = input.trim().replace(/,/g, '');
  if (!AMOUNT_RE.test(s)) return null;
  const [whole = '0', frac = ''] = s.split('.');
  const cents = Number(whole) * 100 + Number(frac.padEnd(2, '0'));
  return Number.isSafeInteger(cents) ? cents : null;
}

export function formatCents(cents: number, currency = 'USD', locale = 'en-US'): string {
  return new Intl.NumberFormat(locale, { style: 'currency', currency }).format(cents / 100);
}
```

> **Finance tip:** Dividing by 100 only for display is safe. The rounding error of a float is far below one cent, and `Intl.NumberFormat` rounds to the currency's minor unit. Some currencies (JPY) have zero decimals and some (KWD) have three. A multi-currency ledger stores the minor-unit exponent per currency.

### [Beginner] Step 11 — Seed the database

```ts
// prisma/seed.ts
import bcrypt from 'bcryptjs';
import { prisma } from '../src/lib/db/client';

async function main() {
  const passwordHash = await bcrypt.hash('Password123!', 12);

  const alice = await prisma.user.upsert({
    where: { email: 'alice@example.com' },
    update: {},
    create: { email: 'alice@example.com', name: 'Alice', passwordHash },
  });

  await prisma.user.upsert({
    where: { email: 'admin@example.com' },
    update: { role: 'ADMIN' },
    create: { email: 'admin@example.com', name: 'Admin', passwordHash, role: 'ADMIN' },
  });

  const existing = await prisma.account.count({ where: { userId: alice.id } });
  if (existing > 0) return;

  const checking = await prisma.account.create({
    data: { userId: alice.id, name: 'Everyday', type: 'CHECKING', currency: 'USD' },
  });
  await prisma.account.create({
    data: { userId: alice.id, name: 'Rainy day', type: 'SAVINGS', currency: 'USD' },
  });

  // Build history through the same balance rule the app uses.
  const moves: Array<{ type: 'CREDIT' | 'DEBIT'; amountCents: number; description: string }> = [
    { type: 'CREDIT', amountCents: 250_000, description: 'Salary' },
    { type: 'DEBIT', amountCents: 4_599, description: 'Groceries' },
    { type: 'DEBIT', amountCents: 120_000, description: 'Rent' },
    { type: 'CREDIT', amountCents: 2_500, description: 'Refund' },
  ];

  let balance = 0;
  for (const m of moves) {
    balance += m.type === 'CREDIT' ? m.amountCents : -m.amountCents;
    await prisma.transaction.create({
      data: { ...m, accountId: checking.id, balanceAfterCents: balance, createdById: alice.id },
    });
  }
  await prisma.account.update({ where: { id: checking.id }, data: { balanceCents: balance } });
}

main()
  .then(() => console.log('Seed complete'))
  .catch((err) => {
    console.error(err);
    process.exitCode = 1;
  })
  .finally(() => prisma.$disconnect());
```

**Check it works:**

```bash
npm run db:seed
docker compose exec db psql -U ledger -c 'select name, "balanceCents" from "Account";'
```

```text
Seed complete
   name    | balanceCents
-----------+--------------
 Everyday  |       127901
 Rainy day |            0
```

Milestone tree:

```text
ledger-web/
├─ docker-compose.yml
├─ prisma.config.ts
├─ prisma/{schema.prisma, seed.ts, migrations/}
└─ src/
   ├─ env.ts
   ├─ generated/prisma/...
   └─ lib/{money.ts, db/{client.ts, index.ts}}
```

## 3. Authentication with Auth.js

### [Beginner] Step 12 — How Auth.js signs a user in

Auth.js (the `next-auth` package, version 5) handles the hard parts of sign-in: OAuth redirects, CSRF tokens for its own endpoints, encrypted cookies and session refresh. You choose providers and a **session strategy**:

- **JWT strategy** (used here): the session lives in an encrypted cookie (`authjs.session-token`, a JWE). No session table, no database read per request, and the proxy can read it cheaply.
- **Database strategy**: the cookie holds an id and every request looks it up. Easier to revoke, but needs an adapter.

```mermaid
sequenceDiagram
  participant B as Browser
  participant A as loginAction
  participant J as Auth.js
  participant P as Prisma
  B->>A: POST form with email and password
  A->>J: signIn credentials
  J->>P: authorize finds user by email
  P-->>J: user with passwordHash
  J->>J: bcrypt compare, then jwt callback adds id and role
  J-->>B: Set-Cookie authjs.session-token, redirect to /accounts
  B->>J: later requests send the cookie
  J-->>B: auth returns session with user id and role
```

> **Outdated:** In NextAuth v4 you used `getServerSession(authOptions)` and a `pages/api/auth/[...nextauth].ts` file. In v5 you call `auth()` everywhere (Server Components, actions, route handlers, proxy), and env vars are prefixed `AUTH_` instead of `NEXTAUTH_`.

### [Beginner] Step 13 — Type the session

Out of the box `session.user` has `name`, `email` and `image`. You need `id` and `role`. TypeScript module augmentation adds them.

```ts
// src/features/auth/types.ts
export type Role = 'USER' | 'ADMIN';

export type SessionUser = {
  id: string;
  role: Role;
  email?: string | null;
  name?: string | null;
};
```

```ts
// src/types/next-auth.d.ts
import type { DefaultSession } from 'next-auth';
import type { Role } from '@/features/auth/types';

declare module 'next-auth' {
  interface Session {
    user: { id: string; role: Role } & DefaultSession['user'];
  }
  interface User {
    role?: Role;
  }
}

declare module 'next-auth/jwt' {
  interface JWT {
    id?: string;
    role?: Role;
  }
}
```

> **Gotcha:** If `token.role` is still typed as `unknown` after this, your installed beta re-exports `JWT` from a different path. Augment `'@auth/core/jwt'` with the same interface instead. Restart the TypeScript server after editing `.d.ts` files.

### [Intermediate] Step 14 — A shared, proxy-safe config

The proxy runs on every request. You want it to only **decode the cookie**, not load bcrypt and Prisma. So split the config: `auth.config.ts` has settings that need no database, and `auth.ts` adds the providers.

```ts
// src/auth.config.ts
import type { NextAuthConfig } from 'next-auth';

export const authConfig = {
  pages: { signIn: '/login' },
  session: {
    strategy: 'jwt',
    maxAge: 60 * 60 * 8, // 8 hours, then sign in again
    updateAge: 60 * 15, // refresh the cookie at most every 15 minutes
  },
  trustHost: true, // required behind a reverse proxy or in a container
  providers: [],
  callbacks: {
    // Runs whenever a session is read. Copies our fields from the token.
    session({ session, token }) {
      if (token.id) session.user.id = token.id;
      if (token.role) session.user.role = token.role;
      return session;
    },
  },
} satisfies NextAuthConfig;
```

> **Why:** Before Next.js 16 the middleware ran on the Edge runtime, where Prisma and bcrypt did not work, so this split was mandatory. `proxy.ts` now runs on Node.js, so it would work either way, but keeping the proxy free of database code still keeps every request fast.

### [Intermediate] Step 15 — Providers and the jwt callback

```ts
// src/auth.ts
import NextAuth from 'next-auth';
import Credentials from 'next-auth/providers/credentials';
import GitHub from 'next-auth/providers/github';
import Okta from 'next-auth/providers/okta';
import type { Provider } from 'next-auth/providers';
import bcrypt from 'bcryptjs';
import { authConfig } from '@/auth.config';
import { prisma } from '@/lib/db';
import { env, features } from '@/env';
import { loginSchema } from '@/features/auth/schemas';

// Compare against a dummy hash when the user does not exist,
// so "unknown email" and "wrong password" take the same time.
let dummyHash: Promise<string> | undefined;
const getDummyHash = () => (dummyHash ??= bcrypt.hash('not-a-real-password', 12));

const providers: Provider[] = [
  Credentials({
    credentials: {
      email: { label: 'Email', type: 'email' },
      password: { label: 'Password', type: 'password' },
    },
    async authorize(raw) {
      const parsed = loginSchema.safeParse(raw);
      if (!parsed.success) return null;
      const { email, password } = parsed.data;

      const user = await prisma.user.findUnique({ where: { email } });
      if (!user?.passwordHash) {
        await bcrypt.compare(password, await getDummyHash());
        return null;
      }
      const ok = await bcrypt.compare(password, user.passwordHash);
      if (!ok) return null;

      return { id: user.id, email: user.email, name: user.name, role: user.role };
    },
  }),
];

if (features.github) {
  providers.push(GitHub({ clientId: env.AUTH_GITHUB_ID, clientSecret: env.AUTH_GITHUB_SECRET }));
}
if (features.okta) {
  providers.push(
    Okta({ clientId: env.AUTH_OKTA_ID, clientSecret: env.AUTH_OKTA_SECRET, issuer: env.AUTH_OKTA_ISSUER }),
  );
}

export const { handlers, auth, signIn, signOut } = NextAuth({
  ...authConfig,
  providers,
  callbacks: {
    ...authConfig.callbacks,
    // `user` and `account` are only present on the sign-in request.
    async jwt({ token, user, account }) {
      if (!user || !account) return token;

      if (account.provider === 'credentials') {
        token.id = user.id;
        token.role = user.role ?? 'USER';
        return token;
      }

      // GitHub or Okta: find or create our own user row by email.
      const email = user.email?.toLowerCase();
      if (!email) throw new Error('The identity provider did not return an email');
      const dbUser = await prisma.user.upsert({
        where: { email },
        update: {},
        create: { email, name: user.name ?? null },
      });
      token.id = dbUser.id;
      token.role = dbUser.role;
      return token;
    },
  },
});
```

> **Gotcha:** Linking accounts by email is only safe if the provider has **verified** the email. GitHub returns the primary verified email and Okta returns `email_verified`. With a provider that lets users type any email, an attacker could sign in as someone else. Auth.js blocks this kind of linking by default when you use an adapter, for the same reason.

> **Gotcha:** The role is baked into the JWT at sign-in. If an admin demotes a user, the old cookie still says `ADMIN` until it expires (8 hours here). For sensitive admin operations, re-read the role from the database. Part 3 Step 22 does that.

> **Why:** OIDC (OpenID Connect) is OAuth 2.0 plus a standard ID token that says who the user is. Okta, Azure AD (Entra ID), Auth0 and Google are all OIDC providers. With Auth.js, adding one is mostly client id, secret and issuer URL. Register `http://localhost:3000/api/auth/callback/okta` (or `/github`) as the redirect URI in the provider's console.

### [Beginner] Step 16 — Mount the Auth.js route handler

```ts
// src/app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/auth';

export const { GET, POST } = handlers;
```

**Check it works:** start `npm run dev` and open http://localhost:3000/api/auth/providers.

```text
{"credentials":{"id":"credentials","name":"Credentials","type":"credentials",...}}
```

### [Beginner] Step 17 — Schemas and auth server actions

The same Zod schema validates the login form, the Credentials `authorize` function and the sign-up action.

```ts
// src/features/auth/schemas.ts
import { z } from 'zod';

export const loginSchema = z.object({
  email: z.string().trim().toLowerCase().pipe(z.email('Enter a valid email')),
  password: z.string().min(1, 'Enter your password').max(200),
});

export const signupSchema = z.object({
  name: z.string().trim().min(2, 'At least 2 characters').max(60),
  email: z.string().trim().toLowerCase().pipe(z.email('Enter a valid email')),
  password: z
    .string()
    .min(10, 'At least 10 characters')
    .max(200)
    .regex(/[0-9]/, 'Include a number')
    .regex(/[A-Za-z]/, 'Include a letter'),
});
```

> **Why:** `trim()` and `toLowerCase()` run first and `pipe(z.email())` validates the cleaned value, so `" Alice@Example.com "` becomes `alice@example.com`. Normalizing emails before you store or look them up avoids duplicate users that differ only by case.

Every form in the app returns the same state shape. Define it once:

```ts
// src/lib/form-state.ts
export type FormState = {
  status: 'idle' | 'success' | 'error';
  message?: string;
  fieldErrors?: Record<string, string[] | undefined>;
};

export const idleState: FormState = { status: 'idle' };
```

```ts
// src/features/auth/actions.ts
'use server';

import { AuthError } from 'next-auth';
import bcrypt from 'bcryptjs';
import { z } from 'zod';
import { signIn, signOut } from '@/auth';
import { prisma } from '@/lib/db';
import type { FormState } from '@/lib/form-state';
import { loginSchema, signupSchema } from './schemas';

/** Only allow same-site relative paths. Prevents open redirects. */
function safeCallbackUrl(value: FormDataEntryValue | null): string {
  if (typeof value !== 'string' || !value.startsWith('/') || value.startsWith('//')) {
    return '/accounts';
  }
  return value;
}

export async function loginAction(_prev: FormState, formData: FormData): Promise<FormState> {
  const parsed = loginSchema.safeParse({
    email: formData.get('email'),
    password: formData.get('password'),
  });
  if (!parsed.success) {
    return { status: 'error', message: 'Check the highlighted fields', fieldErrors: z.flattenError(parsed.error).fieldErrors };
  }

  try {
    await signIn('credentials', {
      ...parsed.data,
      redirectTo: safeCallbackUrl(formData.get('callbackUrl')),
    });
    return { status: 'success' };
  } catch (err) {
    if (err instanceof AuthError) {
      return {
        status: 'error',
        message: err.type === 'CredentialsSignin' ? 'Invalid email or password' : 'Sign-in failed. Try again.',
      };
    }
    throw err; // signIn throws a redirect on success. Never swallow it.
  }
}

export async function signupAction(_prev: FormState, formData: FormData): Promise<FormState> {
  const parsed = signupSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { status: 'error', message: 'Check the highlighted fields', fieldErrors: z.flattenError(parsed.error).fieldErrors };
  }
  const { name, email, password } = parsed.data;

  const exists = await prisma.user.findUnique({ where: { email }, select: { id: true } });
  if (exists) {
    return { status: 'error', fieldErrors: { email: ['An account with this email already exists'] } };
  }

  const passwordHash = await bcrypt.hash(password, 12);
  await prisma.user.create({ data: { name, email, passwordHash } });

  await signIn('credentials', { email, password, redirectTo: '/accounts' });
  return { status: 'success' };
}

export async function oauthSignInAction(provider: 'github' | 'okta', formData: FormData) {
  await signIn(provider, { redirectTo: safeCallbackUrl(formData.get('callbackUrl')) });
}

export async function signOutAction() {
  await signOut({ redirectTo: '/login' });
}
```

> **Gotcha:** `signIn`, `signOut` and Next's `redirect()` work by **throwing** a special error that Next.js catches. If you wrap them in `try/catch`, rethrow anything you do not recognize, as above. Otherwise the redirect silently disappears.

> **Interview tip:** "Revealing that an email exists" on sign-up is a known trade-off. Saying "this email is taken" helps users and leaks account existence. High-security apps send an email instead and show the same message either way.

### [Beginner] Step 18 — Sign-in and sign-up pages

The pages are Server Components. They read `searchParams` (async in Next.js 16) and decide which buttons to show. The forms are Client Components because they use `useActionState`.

```tsx
// src/features/auth/components/field-error.tsx
export function FieldError({ errors }: { errors?: string[] }) {
  if (!errors?.length) return null;
  return <p className="mt-1 text-sm text-red-600">{errors[0]}</p>;
}
```

```tsx
// src/features/auth/components/login-form.tsx
'use client';

import { useActionState } from 'react';
import Link from 'next/link';
import { loginAction, oauthSignInAction } from '@/features/auth/actions';
import { idleState } from '@/lib/form-state';
import { FieldError } from './field-error';

type Props = { callbackUrl: string; showGithub: boolean; showOkta: boolean };

export function LoginForm({ callbackUrl, showGithub, showOkta }: Props) {
  const [state, formAction, pending] = useActionState(loginAction, idleState);

  return (
    <div className="space-y-6">
      <form action={formAction} className="space-y-4" noValidate>
        <input type="hidden" name="callbackUrl" value={callbackUrl} />
        <div>
          <label htmlFor="email" className="block text-sm font-medium">Email</label>
          <input id="email" name="email" type="email" autoComplete="email" required className="input" />
          <FieldError errors={state.fieldErrors?.email} />
        </div>
        <div>
          <label htmlFor="password" className="block text-sm font-medium">Password</label>
          <input id="password" name="password" type="password" autoComplete="current-password" required className="input" />
          <FieldError errors={state.fieldErrors?.password} />
        </div>
        {state.status === 'error' && state.message && (
          <p role="alert" className="text-sm text-red-600">{state.message}</p>
        )}
        <button type="submit" disabled={pending} className="btn-primary w-full">
          {pending ? 'Signing in...' : 'Sign in'}
        </button>
      </form>

      {showGithub && (
        <form action={oauthSignInAction.bind(null, 'github')}>
          <input type="hidden" name="callbackUrl" value={callbackUrl} />
          <button className="btn-secondary w-full">Continue with GitHub</button>
        </form>
      )}
      {showOkta && (
        <form action={oauthSignInAction.bind(null, 'okta')}>
          <input type="hidden" name="callbackUrl" value={callbackUrl} />
          <button className="btn-secondary w-full">Continue with Okta</button>
        </form>
      )}

      <p className="text-sm">No account? <Link href="/signup" className="underline">Create one</Link></p>
    </div>
  );
}
```

```tsx
// src/app/(auth)/login/page.tsx
import { LoginForm } from '@/features/auth/components/login-form';
import { features } from '@/env';

export const metadata = { title: 'Sign in - Ledger' };

export default async function LoginPage({
  searchParams,
}: {
  searchParams: Promise<{ callbackUrl?: string }>;
}) {
  const { callbackUrl } = await searchParams;
  return (
    <main className="mx-auto mt-24 max-w-sm">
      <h1 className="mb-6 text-2xl font-semibold">Sign in to Ledger</h1>
      <LoginForm callbackUrl={callbackUrl ?? '/accounts'} showGithub={features.github} showOkta={features.okta} />
    </main>
  );
}
```

```tsx
// src/features/auth/components/signup-form.tsx
'use client';

import { useActionState } from 'react';
import { signupAction } from '@/features/auth/actions';
import { idleState } from '@/lib/form-state';
import { FieldError } from './field-error';

export function SignupForm() {
  const [state, formAction, pending] = useActionState(signupAction, idleState);
  return (
    <form action={formAction} className="space-y-4" noValidate>
      {(['name', 'email', 'password'] as const).map((field) => (
        <div key={field}>
          <label htmlFor={field} className="block text-sm font-medium capitalize">{field}</label>
          <input
            id={field}
            name={field}
            type={field === 'password' ? 'password' : field === 'email' ? 'email' : 'text'}
            autoComplete={field === 'password' ? 'new-password' : field}
            className="input"
          />
          <FieldError errors={state.fieldErrors?.[field]} />
        </div>
      ))}
      {state.message && <p role="alert" className="text-sm text-red-600">{state.message}</p>}
      <button disabled={pending} className="btn-primary w-full">{pending ? 'Creating...' : 'Create account'}</button>
    </form>
  );
}
```

```tsx
// src/app/(auth)/signup/page.tsx
import { SignupForm } from '@/features/auth/components/signup-form';

export const metadata = { title: 'Create account - Ledger' };

export default function SignupPage() {
  return (
    <main className="mx-auto mt-24 max-w-sm">
      <h1 className="mb-6 text-2xl font-semibold">Create your Ledger account</h1>
      <SignupForm />
    </main>
  );
}
```

Add tiny utility classes so the forms look decent:

```css
/* src/app/globals.css (append below the Tailwind import) */
@layer components {
  .input { @apply mt-1 block w-full rounded border border-gray-300 px-3 py-2; }
  .btn-primary { @apply rounded bg-black px-4 py-2 text-white disabled:opacity-50; }
  .btn-secondary { @apply rounded border border-gray-300 px-4 py-2; }
}
```

> **Why:** A `<form action={serverAction}>` works **before JavaScript loads** (progressive enhancement). `useActionState` adds pending state and the returned errors once React hydrates.

**Check it works:** open http://localhost:3000/login, sign in as `alice@example.com` / `Password123!`. You are redirected to `/accounts` (a 404 for now, the page comes in Part 5). In DevTools, Application, Cookies, you see `authjs.session-token`. A wrong password shows "Invalid email or password".

### [Intermediate] Step 19 — Read the session on the server and the client

On the server, call `auth()`. Wrap it in guards so every page and action reads the same way:

```ts
// src/features/auth/guards.ts
import 'server-only';
import { notFound, redirect } from 'next/navigation';
import { auth } from '@/auth';
import { prisma } from '@/lib/db';
import type { Role, SessionUser } from './types';

export async function getCurrentUser(): Promise<SessionUser | null> {
  const session = await auth();
  if (!session?.user?.id) return null;
  const { id, role, email, name } = session.user;
  return { id, role, email, name };
}

/** For pages and server actions. Redirects to /login when signed out. */
export async function requireUser(): Promise<SessionUser> {
  const user = await getCurrentUser();
  if (!user) redirect('/login');
  return user;
}

/** Re-reads the role from the database, so a demoted admin loses access at once. */
export async function requireRole(role: Role): Promise<SessionUser> {
  const user = await requireUser();
  const fresh = await prisma.user.findUnique({ where: { id: user.id }, select: { role: true } });
  if (fresh?.role !== role) notFound(); // hide that the page exists
  return { ...user, role: fresh.role };
}
```

On the client, `useSession()` needs a `SessionProvider`. Root providers first (used by every page):

```tsx
// src/app/providers.tsx
'use client';

import { useState, type ReactNode } from 'react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Toaster } from 'sonner';

export function Providers({ children }: { children: ReactNode }) {
  // useState so each browser tab gets one client, and the server never shares one between users.
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: { queries: { staleTime: 30_000, retry: false, refetchOnWindowFocus: false } },
      }),
  );
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <Toaster richColors position="top-right" />
    </QueryClientProvider>
  );
}
```

```tsx
// src/app/layout.tsx
import type { Metadata } from 'next';
import type { ReactNode } from 'react';
import { Providers } from './providers';
import './globals.css';

export const metadata: Metadata = { title: 'Ledger', description: 'Accounts and transactions' };

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="en">
      <body className="min-h-screen bg-gray-50 text-gray-900">
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

> **Why:** `retry: false` in TanStack Query because the axios client you build in Part 4 already retries once. Two retry layers multiply requests.

A user menu in the client reads the session with `useSession` and signs out through the server action:

```tsx
// src/features/auth/components/user-menu.tsx
'use client';

import { useSession } from 'next-auth/react';
import { signOutAction } from '@/features/auth/actions';

export function UserMenu() {
  const { data: session, status } = useSession();
  if (status === 'loading') return <span className="text-sm text-gray-500">...</span>;
  if (!session) return null;
  return (
    <div className="flex items-center gap-3 text-sm">
      <span>{session.user.name ?? session.user.email}</span>
      {session.user.role === 'ADMIN' && <a href="/admin" className="underline">Admin</a>}
      <form action={signOutAction}>
        <button className="btn-secondary py-1">Sign out</button>
      </form>
    </div>
  );
}
```

> **Interview tip:** "Do I need `useSession` at all?" Often not. A Server Component can call `auth()` and pass `user` down as props. Use `useSession` for client components deep in the tree, or when the client must react to the session changing (for example after a 401 in Part 4).

### [Intermediate] Step 20 — Protect routes in proxy.ts

The proxy runs before rendering. It is the right place for a **fast, coarse** check: "is there a valid session cookie?" It also stamps every request with an id for logging.

```ts
// src/proxy.ts
import NextAuth from 'next-auth';
import { NextResponse } from 'next/server';
import { authConfig } from '@/auth.config';

const { auth } = NextAuth(authConfig);

const PUBLIC_PATHS = ['/login', '/signup'];
const REQUEST_ID_RE = /^[A-Za-z0-9-]{8,64}$/;

function withRequestId(res: NextResponse, requestId: string) {
  res.headers.set('x-request-id', requestId);
  return res;
}

export default auth((req) => {
  const { pathname, search } = req.nextUrl;
  const incoming = req.headers.get('x-request-id');
  const requestId = incoming && REQUEST_ID_RE.test(incoming) ? incoming : crypto.randomUUID();

  const user = req.auth?.user;
  const isPublic = PUBLIC_PATHS.some((p) => pathname === p || pathname.startsWith(`${p}/`));
  const isApi = pathname.startsWith('/api/');

  if (!user && !isPublic) {
    if (isApi) {
      return withRequestId(
        NextResponse.json(
          { error: { code: 'UNAUTHENTICATED', message: 'Sign in required', requestId } },
          { status: 401 },
        ),
        requestId,
      );
    }
    const login = new URL('/login', req.nextUrl);
    login.searchParams.set('callbackUrl', pathname + search);
    return withRequestId(NextResponse.redirect(login), requestId);
  }

  if (user && isPublic) {
    return withRequestId(NextResponse.redirect(new URL('/accounts', req.nextUrl)), requestId);
  }

  if (pathname.startsWith('/admin') && user?.role !== 'ADMIN') {
    return withRequestId(NextResponse.redirect(new URL('/accounts', req.nextUrl)), requestId);
  }

  // Forward the id to Server Components, actions and route handlers.
  const headers = new Headers(req.headers);
  headers.set('x-request-id', requestId);
  return withRequestId(NextResponse.next({ request: { headers } }), requestId);
});

export const config = {
  matcher: ['/((?!api/auth|api/health|_next/static|_next/image|favicon.ico|.*\\.(?:png|jpg|jpeg|svg|webp|ico)$).*)'],
};
```

```mermaid
flowchart TD
  A["Request"] --> B{"Matched by<br/>config.matcher?"}
  B -->|"no"| Z["Served directly<br/>static files, /api/auth"]
  B -->|"yes"| C{"Valid session<br/>cookie?"}
  C -->|"no, page"| D["Redirect to /login<br/>with callbackUrl"]
  C -->|"no, /api"| E["401 JSON"]
  C -->|"yes"| F{"Admin path and<br/>not ADMIN?"}
  F -->|"yes"| G["Redirect to /accounts"]
  F -->|"no"| H["Add x-request-id<br/>continue to route"]
```

> **Outdated:** Before Next.js 16 this file was `middleware.ts` exporting `middleware`, and ran on the Edge runtime. `middleware.ts` still works in 16 but is deprecated. Rename the file and the export (`npx @next/codemod@canary upgrade latest` does it for you).

> **Gotcha:** The proxy is **not** your security boundary. Matchers have gaps (a typo in the regex, a new route you forgot), and Server Actions are POSTs to the page URL that some setups skip. In 2025 a header-spoofing bug (CVE-2025-29927) let attackers bypass Next.js middleware entirely on unpatched versions. Treat the proxy as a UX optimization and re-check auth where the data is.

**Check it works:**

```bash
curl -i http://localhost:3000/api/v1/accounts
```

```text
HTTP/1.1 401 Unauthorized
x-request-id: 5f0c6c4e-...
{"error":{"code":"UNAUTHENTICATED","message":"Sign in required","requestId":"5f0c6c4e-..."}}
```

Open http://localhost:3000/accounts in a private window and you land on `/login?callbackUrl=%2Faccounts`.

### [Intermediate] Step 21 — Layout-level checks and the re-check rule

The protected area gets its own route group `(app)`. Its layout checks the session again and provides it to client components.

```tsx
// src/app/(app)/layout.tsx
import type { ReactNode } from 'react';
import Link from 'next/link';
import { SessionProvider } from 'next-auth/react';
import { auth } from '@/auth';
import { redirect } from 'next/navigation';
import { UserMenu } from '@/features/auth/components/user-menu';

export default async function AppLayout({ children }: { children: ReactNode }) {
  const session = await auth();
  if (!session?.user) redirect('/login');

  return (
    <SessionProvider session={session}>
      <header className="flex items-center justify-between border-b bg-white px-6 py-3">
        <Link href="/accounts" className="font-semibold">Ledger</Link>
        <UserMenu />
      </header>
      <main className="mx-auto max-w-4xl p-6">{children}</main>
    </SessionProvider>
  );
}
```

```tsx
// src/app/page.tsx
import { redirect } from 'next/navigation';

export default function Home() {
  redirect('/accounts');
}
```

> **Gotcha:** A layout check is **not** enough on its own. Layouts do not re-render when you navigate between pages that share them, and a Server Action or route handler never runs the layout at all. Someone can POST to a Server Action endpoint directly with `curl`. The rule for this codebase: **every page that reads private data, every Server Action and every route handler calls `requireUser()`, `requireRole()` or `withAuth()` itself.**

```mermaid
flowchart LR
  A["proxy.ts<br/>cookie present?"] --> B["layout.tsx<br/>session valid?"]
  B --> C["page, action or route<br/>requireUser or withAuth"]
  C --> D["service.ts<br/>where userId equals owner"]
  D --> E["Postgres CHECK<br/>balance not negative"]
```

> **Interview tip:** Call this "defense in depth". Each layer catches mistakes in the one before it, and the last two layers (ownership filter and database constraint) protect the data even if every UI layer is wrong.

### [Intermediate] Step 22 — Roles and resource ownership

**Authorization** has two questions. Role: "may this kind of user do this at all?" Ownership: "is this specific account theirs?" Ownership lives in the service: every query filters by `userId`. A missing or foreign account returns the same 404, so attackers cannot probe which ids exist.

First the shared error classes, used by services, actions and route handlers:

```ts
// src/lib/errors.ts
export class AppError extends Error {
  readonly code: string;
  readonly status: number;
  readonly details?: unknown;
  constructor(code: string, message: string, status: number, details?: unknown) {
    super(message);
    this.name = new.target.name;
    this.code = code;
    this.status = status;
    this.details = details;
  }
}

export class ValidationError extends AppError {
  constructor(details: unknown, message = 'Request validation failed') {
    super('VALIDATION_FAILED', message, 400, details);
  }
}
export class UnauthorizedError extends AppError {
  constructor() { super('UNAUTHENTICATED', 'Sign in required', 401); }
}
export class ForbiddenError extends AppError {
  constructor() { super('FORBIDDEN', 'You do not have access to this resource', 403); }
}
export class NotFoundError extends AppError {
  constructor(resource: string) { super('NOT_FOUND', `${resource} not found`, 404); }
}
export class ConflictError extends AppError {
  constructor(message: string) { super('CONFLICT', message, 409); }
}
export class InsufficientFundsError extends AppError {
  constructor() { super('INSUFFICIENT_FUNDS', 'Insufficient funds for this debit', 422); }
}
export class RateLimitError extends AppError {
  constructor(retryAfterSec: number) {
    super('RATE_LIMITED', 'Too many requests', 429, { retryAfterSec });
  }
}
```

The accounts DTO and schemas:

```ts
// src/features/accounts/dto.ts
import type { Account } from '@/generated/prisma/client';

export type AccountDTO = {
  id: string;
  name: string;
  type: 'CHECKING' | 'SAVINGS';
  currency: string;
  balanceCents: number;
  createdAt: string;
};

export function toAccountDTO(a: Account): AccountDTO {
  return {
    id: a.id,
    name: a.name,
    type: a.type,
    currency: a.currency,
    balanceCents: a.balanceCents,
    createdAt: a.createdAt.toISOString(),
  };
}
```

```ts
// src/features/accounts/schemas.ts
import { z } from 'zod';

export const CURRENCIES = ['USD', 'EUR', 'GBP', 'NPR'] as const;

export const createAccountSchema = z.object({
  name: z.string().trim().min(2, 'At least 2 characters').max(50),
  type: z.enum(['CHECKING', 'SAVINGS']),
  currency: z.enum(CURRENCIES),
});

export const renameAccountSchema = createAccountSchema.pick({ name: true });

export type CreateAccountInput = z.infer<typeof createAccountSchema>;
export type RenameAccountInput = z.infer<typeof renameAccountSchema>;
```

The service. It takes `userId` explicitly and never calls `auth()`, so it is easy to test and reuse:

```ts
// src/features/accounts/service.ts
import 'server-only';
import { prisma, Prisma } from '@/lib/db';
import { ConflictError, NotFoundError } from '@/lib/errors';
import { toAccountDTO, type AccountDTO } from './dto';
import type { CreateAccountInput, RenameAccountInput } from './schemas';

function isUniqueViolation(err: unknown) {
  return err instanceof Prisma.PrismaClientKnownRequestError && err.code === 'P2002';
}

export async function listAccounts(userId: string): Promise<AccountDTO[]> {
  const rows = await prisma.account.findMany({
    where: { userId, archivedAt: null },
    orderBy: { createdAt: 'asc' },
  });
  return rows.map(toAccountDTO);
}

export async function getAccount(userId: string, accountId: string): Promise<AccountDTO> {
  const row = await prisma.account.findFirst({ where: { id: accountId, userId, archivedAt: null } });
  if (!row) throw new NotFoundError('Account');
  return toAccountDTO(row);
}

export async function createAccount(userId: string, input: CreateAccountInput): Promise<AccountDTO> {
  try {
    const row = await prisma.account.create({ data: { ...input, userId } });
    return toAccountDTO(row);
  } catch (err) {
    if (isUniqueViolation(err)) throw new ConflictError('You already have an account with this name');
    throw err;
  }
}

export async function renameAccount(userId: string, accountId: string, input: RenameAccountInput): Promise<AccountDTO> {
  try {
    // updateMany lets us filter by owner. A plain update only accepts unique fields.
    const { count } = await prisma.account.updateMany({
      where: { id: accountId, userId, archivedAt: null },
      data: { name: input.name },
    });
    if (count === 0) throw new NotFoundError('Account');
  } catch (err) {
    if (isUniqueViolation(err)) throw new ConflictError('You already have an account with this name');
    throw err;
  }
  return getAccount(userId, accountId);
}

export async function archiveAccount(userId: string, accountId: string): Promise<void> {
  const { count } = await prisma.account.updateMany({
    where: { id: accountId, userId, archivedAt: null, balanceCents: 0 },
    data: { archivedAt: new Date() },
  });
  if (count === 1) return;
  await getAccount(userId, accountId); // throws NotFoundError if it is not theirs
  throw new ConflictError('Move the balance out before archiving. Balance must be zero.');
}
```

> **Gotcha:** `findUnique({ where: { id } })` followed by `if (account.userId !== userId)` also works, but it is easy to forget the second line. Putting `userId` **inside the where clause** makes "not yours" and "does not exist" the same query result.

Now the admin side: a role-protected page that sees all users.

```ts
// src/features/admin/service.ts
import 'server-only';
import { prisma } from '@/lib/db';

export type AdminUserRow = {
  id: string;
  email: string;
  name: string | null;
  role: 'USER' | 'ADMIN';
  accountCount: number;
  totals: Record<string, number>; // currency -> cents
};

export async function listUsersWithTotals(): Promise<AdminUserRow[]> {
  const [users, sums] = await Promise.all([
    prisma.user.findMany({
      orderBy: { createdAt: 'asc' },
      select: { id: true, email: true, name: true, role: true, _count: { select: { accounts: true } } },
    }),
    prisma.account.groupBy({
      by: ['userId', 'currency'],
      where: { archivedAt: null },
      _sum: { balanceCents: true },
    }),
  ]);

  const totals = new Map<string, Record<string, number>>();
  for (const s of sums) {
    const t = totals.get(s.userId) ?? {};
    t[s.currency] = s._sum.balanceCents ?? 0;
    totals.set(s.userId, t);
  }

  return users.map((u) => ({
    id: u.id,
    email: u.email,
    name: u.name,
    role: u.role,
    accountCount: u._count.accounts,
    totals: totals.get(u.id) ?? {},
  }));
}
```

> **Finance tip:** Never add balances in different currencies. Group by currency first, as above.

```tsx
// src/app/(app)/admin/page.tsx
import { requireRole } from '@/features/auth/guards';
import { listUsersWithTotals } from '@/features/admin/service';
import { formatCents } from '@/lib/money';

export const metadata = { title: 'Admin - Ledger' };

export default async function AdminPage() {
  await requireRole('ADMIN');
  const users = await listUsersWithTotals();

  return (
    <section>
      <h1 className="mb-4 text-xl font-semibold">Users</h1>
      <table className="w-full bg-white text-sm">
        <thead>
          <tr className="text-left"><th className="p-2">Email</th><th>Role</th><th>Accounts</th><th>Totals</th></tr>
        </thead>
        <tbody>
          {users.map((u) => (
            <tr key={u.id} className="border-t">
              <td className="p-2">{u.email}</td>
              <td>{u.role}</td>
              <td>{u.accountCount}</td>
              <td>
                {Object.entries(u.totals).map(([cur, cents]) => formatCents(cents, cur)).join(', ') || '-'}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </section>
  );
}
```

**Check it works:** signed in as Alice, http://localhost:3000/admin redirects you to `/accounts` (proxy). Sign out, sign in as `admin@example.com` and the table lists both users with `$1,279.01` for Alice. Now in psql run `update "User" set role='USER' where email='admin@example.com';` and refresh `/admin`: the proxy still lets you through (stale JWT), but `requireRole` reads the database and shows the 404 page. Set the role back to `ADMIN`.

Milestone tree:

```text
src/
├─ app/
│  ├─ (auth)/{login,signup}/page.tsx
│  ├─ (app)/{layout.tsx, admin/page.tsx}
│  ├─ api/auth/[...nextauth]/route.ts
│  ├─ layout.tsx, page.tsx, providers.tsx, globals.css
├─ features/
│  ├─ auth/{actions.ts, guards.ts, schemas.ts, types.ts, components/}
│  ├─ accounts/{dto.ts, schemas.ts, service.ts}
│  └─ admin/service.ts
├─ lib/{db/, errors.ts, form-state.ts, money.ts}
├─ types/next-auth.d.ts
├─ auth.config.ts, auth.ts, env.ts, proxy.ts
```

## 4. API layer and interceptors

### [Beginner] Step 23 — A structured logger

Route handlers need a logger before they need anything else. `pino` writes one JSON object per line, which every log platform can index.

```ts
// src/lib/logger.ts
import 'server-only';
import pino from 'pino';
import { env } from '@/env';

export const logger = pino({
  level: env.LOG_LEVEL,
  base: { service: 'ledger-web' },
  redact: { paths: ['password', '*.password', 'headers.cookie', 'headers.authorization'], censor: '[redacted]' },
  ...(env.NODE_ENV === 'development' ? { transport: { target: 'pino-pretty', options: { colorize: true } } } : {}),
});

export type Logger = typeof logger;
```

```ts
// next.config.ts (first version, extended in Part 6)
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  output: 'standalone',
  // Keep pino out of the bundler. Its worker-thread transports break when bundled.
  serverExternalPackages: ['pino', 'pino-pretty'],
};

export default nextConfig;
```

> **Gotcha:** `pino-pretty` as a transport runs in a worker thread. It is for local development only. In production log raw JSON to stdout and let the platform collect it.

### [Intermediate] Step 24 — A shared route handler wrapper

Every `/api/v1` handler repeats the same chores: read the request id, check the session, parse input, turn errors into JSON, log the result. Write them once as higher-order functions.

The error JSON contract is the API's most important type. Clients depend on it:

```ts
// src/lib/api/error-body.ts
import { Prisma } from '@/lib/db';
import { AppError } from '@/lib/errors';

export type ApiErrorBody = {
  error: { code: string; message: string; details?: unknown; requestId: string };
};

export function toErrorResponse(err: unknown, requestId: string): { status: number; body: ApiErrorBody } {
  if (err instanceof AppError) {
    return { status: err.status, body: { error: { code: err.code, message: err.message, details: err.details, requestId } } };
  }
  if (err instanceof Prisma.PrismaClientKnownRequestError) {
    if (err.code === 'P2025') return { status: 404, body: { error: { code: 'NOT_FOUND', message: 'Not found', requestId } } };
    if (err.code === 'P2002') return { status: 409, body: { error: { code: 'CONFLICT', message: 'Already exists', requestId } } };
  }
  // Never leak stack traces or SQL to clients.
  return { status: 500, body: { error: { code: 'INTERNAL', message: 'Something went wrong', requestId } } };
}
```

```ts
// src/lib/api/handler.ts
import 'server-only';
import { NextResponse, type NextRequest } from 'next/server';
import { z } from 'zod';
import { auth } from '@/auth';
import { logger, type Logger } from '@/lib/logger';
import { ForbiddenError, UnauthorizedError, ValidationError } from '@/lib/errors';
import type { Role, SessionUser } from '@/features/auth/types';
import { toErrorResponse } from './error-body';

type RouteContext<P> = { params: Promise<P> };
type NextRouteHandler<P> = (req: NextRequest, ctx: RouteContext<P>) => Promise<Response>;

export type HandlerArgs<P> = { req: NextRequest; params: P; requestId: string; log: Logger };
export type AuthedArgs<P> = HandlerArgs<P> & { user: SessionUser };

export function withErrorHandling<P = Record<string, never>>(
  fn: (args: HandlerArgs<P>) => Promise<Response>,
): NextRouteHandler<P> {
  return async (req, ctx) => {
    const requestId = req.headers.get('x-request-id') ?? crypto.randomUUID();
    const log = logger.child({ requestId, method: req.method, path: req.nextUrl.pathname });
    const started = performance.now();
    try {
      const params = ((await ctx?.params) ?? {}) as P;
      const res = await fn({ req, params, requestId, log });
      res.headers.set('x-request-id', requestId);
      log.info({ status: res.status, ms: Math.round(performance.now() - started) }, 'request completed');
      return res;
    } catch (err) {
      const { status, body } = toErrorResponse(err, requestId);
      if (status >= 500) log.error({ err }, 'request failed');
      else log.warn({ status, code: body.error.code }, 'request rejected');
      return NextResponse.json(body, { status, headers: { 'x-request-id': requestId } });
    }
  };
}

export function withAuth<P = Record<string, never>>(
  fn: (args: AuthedArgs<P>) => Promise<Response>,
  options: { role?: Role } = {},
): NextRouteHandler<P> {
  return withErrorHandling<P>(async (args) => {
    const session = await auth();
    if (!session?.user?.id) throw new UnauthorizedError();
    const user: SessionUser = { id: session.user.id, role: session.user.role, email: session.user.email };
    if (options.role && user.role !== options.role) throw new ForbiddenError();
    return fn({ ...args, user, log: args.log.child({ userId: user.id }) });
  });
}

export async function parseBody<S extends z.ZodType>(req: NextRequest, schema: S): Promise<z.infer<S>> {
  const json: unknown = await req.json().catch(() => {
    throw new ValidationError(null, 'Body must be valid JSON');
  });
  const result = schema.safeParse(json);
  if (!result.success) throw new ValidationError(z.flattenError(result.error));
  return result.data;
}

export function parseQuery<S extends z.ZodType>(req: NextRequest, schema: S): z.infer<S> {
  const result = schema.safeParse(Object.fromEntries(req.nextUrl.searchParams));
  if (!result.success) throw new ValidationError(z.flattenError(result.error));
  return result.data;
}

export const json = <T,>(data: T, init?: ResponseInit) => NextResponse.json(data, init);
```

```mermaid
flowchart LR
  A["GET /api/v1/accounts/abc"] --> B["withErrorHandling<br/>request id, timer, logger"]
  B --> C["withAuth<br/>auth, role check"]
  C --> D["parseBody or parseQuery<br/>Zod"]
  D --> E["service call<br/>userId passed in"]
  E --> F["json data"]
  C -->|"throws"| G["toErrorResponse<br/>error code message requestId"]
  D -->|"throws"| G
  E -->|"throws"| G
```

> **Why:** Higher-order functions are the Next.js answer to NestJS guards, pipes and filters. There is no framework-level pipeline for route handlers, so you compose one. Keep it small and typed.

> **Gotcha:** Next.js type-checks route handler exports during `next build`. If it complains about the second argument, type it with the generated helper instead: `ctx: RouteContext<'/api/v1/accounts/[id]'>` (a global type Next.js 15.5+ generates during `next dev`, `next build` or `next typegen`).

### [Intermediate] Step 25 — Accounts routes under /api/v1

Success responses are always `{ data }`, lists add `meta`. Errors are always `{ error }`.

```ts
// src/app/api/v1/accounts/route.ts
import { withAuth, parseBody, json } from '@/lib/api/handler';
import { createAccount, listAccounts } from '@/features/accounts/service';
import { createAccountSchema } from '@/features/accounts/schemas';

export const GET = withAuth(async ({ user }) => {
  return json({ data: await listAccounts(user.id) });
});

export const POST = withAuth(async ({ req, user, log }) => {
  const input = await parseBody(req, createAccountSchema);
  const account = await createAccount(user.id, input);
  log.info({ accountId: account.id }, 'account created');
  return json({ data: account }, { status: 201 });
});
```

```ts
// src/app/api/v1/accounts/[id]/route.ts
import { withAuth, parseBody, json } from '@/lib/api/handler';
import { archiveAccount, getAccount, renameAccount } from '@/features/accounts/service';
import { renameAccountSchema } from '@/features/accounts/schemas';

type Params = { id: string };

export const GET = withAuth<Params>(async ({ user, params }) => {
  return json({ data: await getAccount(user.id, params.id) });
});

export const PATCH = withAuth<Params>(async ({ req, user, params }) => {
  const input = await parseBody(req, renameAccountSchema);
  return json({ data: await renameAccount(user.id, params.id, input) });
});

export const DELETE = withAuth<Params>(async ({ user, params }) => {
  await archiveAccount(user.id, params.id);
  return new Response(null, { status: 204 });
});
```

```ts
// src/app/api/health/route.ts
import { prisma } from '@/lib/db';

export const dynamic = 'force-dynamic';

export async function GET() {
  try {
    await prisma.$queryRaw`SELECT 1`;
    return Response.json({ status: 'ok' });
  } catch {
    return Response.json({ status: 'db_unavailable' }, { status: 503 });
  }
}
```

**Check it works:** sign in in the browser, copy the `authjs.session-token` cookie value from DevTools, then:

```bash
export C="authjs.session-token=PASTE_VALUE"
curl -s -b "$C" http://localhost:3000/api/v1/accounts | jq '.data[].name'
curl -s -b "$C" -X POST http://localhost:3000/api/v1/accounts \
  -H 'content-type: application/json' -d '{"name":"x","type":"CHECKING","currency":"USD"}' | jq
curl -s -b "$C" http://localhost:3000/api/v1/accounts/does-not-exist | jq
```

```text
"Everyday"
"Rainy day"
{ "error": { "code": "VALIDATION_FAILED", "message": "Request validation failed",
  "details": { "formErrors": [], "fieldErrors": { "name": ["At least 2 characters"] } }, "requestId": "..." } }
{ "error": { "code": "NOT_FOUND", "message": "Account not found", "requestId": "..." } }
```

The terminal running `npm run dev` shows one log line per request with the same `requestId`.

### [Intermediate] Step 26 — A typed axios client with interceptors

Client components that fetch interactively (tables, polling, search-as-you-type) call `/api/v1` from the browser. Wrap axios once so every call gets the same behavior:

- **Request interceptor:** add `X-Request-Id` (so a support ticket can be matched to server logs) and, for external APIs, a bearer token.
- **Response interceptor:** turn every failure into one `ApiError` type; on 401, refresh the session once and retry, otherwise redirect to login; retry once on network errors and 502/503/504 for safe methods; toast on 5xx.

> **Why:** For same-origin calls you do **not** attach a token yourself. The session cookie is `httpOnly` (JavaScript cannot read it, so XSS cannot steal it) and the browser sends it automatically. The `getAccessToken` hook below is for calling a separate API, such as the NestJS `ledger-api`, with an OIDC access token.

```ts
// src/lib/api-client/api-error.ts
export class ApiError extends Error {
  readonly status: number;
  readonly code: string;
  readonly details?: unknown;
  readonly requestId?: string;
  constructor(init: { status: number; code: string; message: string; details?: unknown; requestId?: string }) {
    super(init.message);
    this.name = 'ApiError';
    this.status = init.status;
    this.code = init.code;
    this.details = init.details;
    this.requestId = init.requestId;
  }
}

export const isApiError = (e: unknown): e is ApiError => e instanceof ApiError;
```

```ts
// src/lib/api-client/http.ts
import axios, { AxiosError, type AxiosRequestConfig } from 'axios';
import { getSession } from 'next-auth/react';
import { toast } from 'sonner';
import { ApiError } from './api-error';
import type { ApiErrorBody } from '@/lib/api/error-body';

declare module 'axios' {
  interface AxiosRequestConfig {
    _authRetried?: boolean;
    _retried?: boolean;
    skipErrorToast?: boolean;
  }
}

let getAccessToken: (() => Promise<string | null>) | null = null;
/** Optional: provide a token getter when calling an external API. */
export function setAccessTokenProvider(fn: typeof getAccessToken) {
  getAccessToken = fn;
}

export const http = axios.create({
  baseURL: '/api/v1',
  timeout: 15_000,
  headers: { Accept: 'application/json' },
});

http.interceptors.request.use(async (config) => {
  config.headers.set('X-Request-Id', crypto.randomUUID());
  const token = getAccessToken ? await getAccessToken() : null;
  if (token) config.headers.set('Authorization', `Bearer ${token}`);
  return config;
});

// Many requests can 401 at once. Share one session refresh between them.
let refreshing: Promise<boolean> | null = null;
function refreshSession(): Promise<boolean> {
  refreshing ??= getSession()
    .then((s) => Boolean(s))
    .finally(() => {
      refreshing = null;
    });
  return refreshing;
}

const SAFE_METHODS = new Set(['get', 'head', 'options']);
const RETRYABLE_STATUS = new Set([502, 503, 504]);
const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

function toApiError(error: AxiosError<ApiErrorBody>): ApiError {
  const status = error.response?.status ?? 0;
  const body = error.response?.data?.error;
  if (body) return new ApiError({ status, ...body });
  if (error.code === 'ECONNABORTED') return new ApiError({ status, code: 'TIMEOUT', message: 'The request timed out' });
  if (!error.response) return new ApiError({ status, code: 'NETWORK', message: 'Network error. Check your connection.' });
  return new ApiError({ status, code: 'HTTP_' + status, message: error.message });
}

http.interceptors.response.use(
  (res) => res,
  async (error: unknown) => {
    if (axios.isCancel(error)) throw error; // aborted on purpose, not a failure
    if (!axios.isAxiosError<ApiErrorBody>(error) || !error.config) throw error;
    const config: AxiosRequestConfig = error.config;
    const status = error.response?.status;

    if (status === 401 && !config._authRetried) {
      config._authRetried = true;
      if (await refreshSession()) return http(config);
      const callbackUrl = encodeURIComponent(window.location.pathname + window.location.search);
      window.location.assign(`/login?callbackUrl=${callbackUrl}`);
      return new Promise(() => {}); // page is navigating away. Never settle.
    }

    const method = (config.method ?? 'get').toLowerCase();
    const retryable = !error.response || (status !== undefined && RETRYABLE_STATUS.has(status));
    if (retryable && SAFE_METHODS.has(method) && !config._retried) {
      config._retried = true;
      await sleep(300);
      return http(config);
    }

    const apiError = toApiError(error);
    if (apiError.status >= 500 && !config.skipErrorToast) {
      toast.error('Server error', { description: `Please try again. Reference: ${apiError.requestId ?? 'n/a'}` });
    }
    throw apiError;
  },
);
```

```mermaid
flowchart TD
  A["Response error"] --> B{"Cancelled?"}
  B -->|"yes"| C["Rethrow cancel"]
  B -->|"no"| D{"401 and not<br/>retried yet?"}
  D -->|"yes"| E{"getSession<br/>still valid?"}
  E -->|"yes"| F["Retry once"]
  E -->|"no"| G["Redirect to /login"]
  D -->|"no"| H{"Network or 502-504<br/>on GET?"}
  H -->|"yes"| I["Wait 300ms, retry once"]
  H -->|"no"| J["Normalize to ApiError"]
  J --> K{"5xx?"}
  K -->|"yes"| L["Toast with request id"]
  K -->|"no"| M["Throw ApiError"]
  L --> M
```

> **Gotcha:** Only retry **safe** methods automatically. Retrying a `POST /transactions` after a timeout can create a second payment if the first one actually succeeded. For POSTs, send an `Idempotency-Key` header and let the server deduplicate (Step 30).

> **Why a session refresh on 401?** With the JWT strategy, `getSession()` calls `/api/auth/session`, which re-issues the cookie when it is still within `maxAge`. If it returns `null`, the session really expired and the only fix is signing in again. If you used an OIDC access token, this is where you would call your token refresh endpoint.

### [Intermediate] Step 27 — Typed endpoints and abort support

Components should never build URLs. One module describes the API:

```ts
// src/lib/pagination.ts
export type Paginated<T> = { data: T[]; meta: { page: number; pageSize: number; total: number } };
```

```ts
// src/lib/api-client/endpoints.ts
import { http } from './http';
import type { AccountDTO } from '@/features/accounts/dto';
import type { TransactionDTO } from '@/features/transactions/dto';
import type { TransactionFilters, CreateTransactionBody } from '@/features/transactions/schemas';
import type { Paginated } from '@/lib/pagination';

export const api = {
  accounts: {
    list: (signal?: AbortSignal) =>
      http.get<{ data: AccountDTO[] }>('/accounts', { signal }).then((r) => r.data.data),
  },
  transactions: {
    list: (accountId: string, filters: TransactionFilters, signal?: AbortSignal) =>
      http
        .get<Paginated<TransactionDTO>>(`/accounts/${accountId}/transactions`, { params: filters, signal })
        .then((r) => r.data),
    create: (accountId: string, body: CreateTransactionBody, idempotencyKey: string) =>
      http
        .post<{ data: TransactionDTO }>(`/accounts/${accountId}/transactions`, body, {
          headers: { 'Idempotency-Key': idempotencyKey },
        })
        .then((r) => r.data.data),
  },
};
```

TanStack Query hands every `queryFn` an `AbortSignal`. When the query key changes (the user types another filter) or the component unmounts, the old request is cancelled. You just pass the signal through:

```ts
useQuery({
  queryKey: ['transactions', accountId, filters],
  queryFn: ({ signal }) => api.transactions.list(accountId, filters, signal),
});
```

Outside TanStack Query, use an `AbortController` yourself:

```ts
const controller = new AbortController();
api.accounts.list(controller.signal).catch((e) => { if (!axios.isCancel(e)) throw e; });
controller.abort(); // on unmount or when a newer request starts
```

> **Gotcha:** This file imports types from features. That is allowed because they are `import type`, which TypeScript erases. A runtime import of a feature's `service.ts` here would pull server code into the browser, and `server-only` would fail the build.

### [Beginner] Step 28 — Route handler, server action or direct service call?

| You need to... | Use | Why |
|---|---|---|
| Render data on a page | Server Component calls `queries.ts` (service directly) | No HTTP hop, no JSON, streams HTML |
| Submit a form from your own UI | Server Action | Progressive enhancement, typed args, `revalidatePath` refreshes the page in the same round trip |
| Interactive client data: filters, infinite scroll, polling | Route handler + TanStack Query | Cacheable GETs, abort, background refetch |
| Serve mobile apps, other teams, webhooks | Route handler | Stable URL, versioned (`/v1`), documented contract |
| Call your own route handler from a Server Component | Do not | It is an extra network hop to yourself. Call the service |

```mermaid
flowchart TD
  A["What are you building?"] --> B{"Read or write?"}
  B -->|"read on page load"| C["Server Component<br/>calls service"]
  B -->|"read, interactive in browser"| D["Route handler GET<br/>TanStack Query"]
  B -->|"write from own UI"| E["Server Action"]
  B -->|"write from other client"| F["Route handler POST"]
  E --> G["Same service function"]
  F --> G
  C --> G
  D --> G
```

> **Interview tip:** Server Actions are public HTTP endpoints with an unguessable id, not private function calls. Anyone can call them with any arguments. Always validate and authorize inside them, exactly like a route handler.

## 5. Features: accounts and transactions

### [Beginner] Step 29 — Accounts pages with Server Components

`queries.ts` is the read API for pages. It combines the auth check with the service call and uses React's `cache()` so two components that ask for the same account in one request hit the database once.

```ts
// src/features/accounts/queries.ts
import 'server-only';
import { cache } from 'react';
import { requireUser } from '@/features/auth/guards';
import { NotFoundError } from '@/lib/errors';
import { getAccount, listAccounts } from './service';

export const getMyAccounts = cache(async () => {
  const user = await requireUser();
  return listAccounts(user.id);
});

export const getMyAccount = cache(async (accountId: string) => {
  const user = await requireUser();
  try {
    return await getAccount(user.id, accountId);
  } catch (err) {
    if (err instanceof NotFoundError) return null;
    throw err;
  }
});
```

```tsx
// src/features/accounts/components/account-list.tsx
import Link from 'next/link';
import { formatCents } from '@/lib/money';
import type { AccountDTO } from '../dto';

export function AccountList({ accounts }: { accounts: AccountDTO[] }) {
  if (accounts.length === 0) {
    return <p className="text-gray-600">No accounts yet. Create your first one.</p>;
  }
  return (
    <ul className="grid gap-3 sm:grid-cols-2">
      {accounts.map((a) => (
        <li key={a.id}>
          <Link href={`/accounts/${a.id}`} className="block rounded border bg-white p-4 hover:shadow">
            <p className="text-sm text-gray-500">{a.type}</p>
            <p className="font-medium">{a.name}</p>
            <p className="mt-2 text-2xl tabular-nums">{formatCents(a.balanceCents, a.currency)}</p>
          </Link>
        </li>
      ))}
    </ul>
  );
}
```

```tsx
// src/app/(app)/accounts/page.tsx
import Link from 'next/link';
import { getMyAccounts } from '@/features/accounts/queries';
import { AccountList } from '@/features/accounts/components/account-list';

export const metadata = { title: 'Accounts - Ledger' };

export default async function AccountsPage() {
  const accounts = await getMyAccounts();
  return (
    <section className="space-y-4">
      <div className="flex items-center justify-between">
        <h1 className="text-xl font-semibold">Your accounts</h1>
        <Link href="/accounts/new" className="btn-primary">New account</Link>
      </div>
      <AccountList accounts={accounts} />
    </section>
  );
}
```

> **Why:** This page ships **zero** JavaScript for the list. The database query runs on the server and the browser receives HTML. Compare with a React SPA: download JS, render a spinner, call the API, render again.

**Check it works:** http://localhost:3000/accounts shows "Everyday $1,279.01" and "Rainy day $0.00".

### [Intermediate] Step 30 — Create and rename with Server Actions and useActionState

The action pattern is always the same five lines of intent: **authenticate, validate, call the service, revalidate, redirect or return state.**

```ts
// src/lib/form-errors.ts
import 'server-only';
import { AppError } from '@/lib/errors';
import type { FormState } from '@/lib/form-state';

/** Expected business errors become form messages. Bugs are rethrown to error.tsx. */
export function toFormError(err: unknown): FormState {
  if (err instanceof AppError && err.status < 500) return { status: 'error', message: err.message };
  throw err;
}
```

```ts
// src/features/accounts/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
import { z } from 'zod';
import { requireUser } from '@/features/auth/guards';
import type { FormState } from '@/lib/form-state';
import { toFormError } from '@/lib/form-errors';
import { createAccountSchema, renameAccountSchema } from './schemas';
import { archiveAccount, createAccount, renameAccount } from './service';

export async function createAccountAction(_prev: FormState, formData: FormData): Promise<FormState> {
  const user = await requireUser();
  const parsed = createAccountSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { status: 'error', message: 'Check the highlighted fields', fieldErrors: z.flattenError(parsed.error).fieldErrors };
  }

  let accountId: string;
  try {
    accountId = (await createAccount(user.id, parsed.data)).id;
  } catch (err) {
    return toFormError(err);
  }

  revalidatePath('/accounts');
  redirect(`/accounts/${accountId}`); // outside try/catch: redirect throws
}

export async function renameAccountAction(accountId: string, _prev: FormState, formData: FormData): Promise<FormState> {
  const user = await requireUser();
  const parsed = renameAccountSchema.safeParse({ name: formData.get('name') });
  if (!parsed.success) {
    return { status: 'error', fieldErrors: z.flattenError(parsed.error).fieldErrors };
  }
  try {
    await renameAccount(user.id, accountId, parsed.data);
  } catch (err) {
    return toFormError(err);
  }
  revalidatePath('/accounts');
  revalidatePath(`/accounts/${accountId}`);
  return { status: 'success', message: 'Account renamed' };
}

export async function archiveAccountAction(accountId: string, _prev: FormState): Promise<FormState> {
  const user = await requireUser();
  try {
    await archiveAccount(user.id, accountId);
  } catch (err) {
    return toFormError(err);
  }
  revalidatePath('/accounts');
  redirect('/accounts');
}
```

> **Why:** `revalidatePath` tells Next.js that cached render output for that path is stale. When it is called inside a Server Action, the response to the action already contains the fresh page, so the UI updates in the same round trip with no extra fetch.

> **Outdated:** In Next.js 16, `revalidateTag(tag)` with one argument is deprecated. Use `revalidateTag(tag, 'max')` for background refresh, or `updateTag(tag)` inside Server Actions when the user must see their own write immediately. `revalidatePath` is unchanged and is enough for this app, which does not use `'use cache'`.

```tsx
// src/features/accounts/components/create-account-form.tsx
'use client';

import { useActionState } from 'react';
import { createAccountAction } from '../actions';
import { CURRENCIES } from '../schemas';
import { idleState } from '@/lib/form-state';
import { FieldError } from '@/features/auth/components/field-error';

export function CreateAccountForm() {
  const [state, formAction, pending] = useActionState(createAccountAction, idleState);
  return (
    <form action={formAction} className="max-w-md space-y-4">
      <div>
        <label htmlFor="name" className="block text-sm font-medium">Name</label>
        <input id="name" name="name" className="input" />
        <FieldError errors={state.fieldErrors?.name} />
      </div>
      <div className="flex gap-4">
        <label className="block text-sm font-medium">
          Type
          <select name="type" className="input" defaultValue="CHECKING">
            <option value="CHECKING">Checking</option>
            <option value="SAVINGS">Savings</option>
          </select>
        </label>
        <label className="block text-sm font-medium">
          Currency
          <select name="currency" className="input" defaultValue="USD">
            {CURRENCIES.map((c) => <option key={c}>{c}</option>)}
          </select>
        </label>
      </div>
      {state.message && <p role="alert" className="text-sm text-red-600">{state.message}</p>}
      <button disabled={pending} className="btn-primary">{pending ? 'Creating...' : 'Create account'}</button>
    </form>
  );
}
```

```tsx
// src/app/(app)/accounts/new/page.tsx
import { CreateAccountForm } from '@/features/accounts/components/create-account-form';

export default function NewAccountPage() {
  return (
    <section>
      <h1 className="mb-4 text-xl font-semibold">New account</h1>
      <CreateAccountForm />
    </section>
  );
}
```

The edit page binds the account id into the action. `bind` is how you pass extra, trusted-looking arguments, but remember: the bound id comes from the client and is still checked by the service's `userId` filter.

```tsx
// src/features/accounts/components/edit-account-forms.tsx
'use client';

import { useActionState, useEffect } from 'react';
import { toast } from 'sonner';
import { archiveAccountAction, renameAccountAction } from '../actions';
import { idleState } from '@/lib/form-state';
import { FieldError } from '@/features/auth/components/field-error';
import type { AccountDTO } from '../dto';

export function EditAccountForms({ account }: { account: AccountDTO }) {
  const [renameState, renameAction, renaming] = useActionState(renameAccountAction.bind(null, account.id), idleState);
  const [archiveState, archiveAction, archiving] = useActionState(archiveAccountAction.bind(null, account.id), idleState);

  useEffect(() => {
    if (renameState.status === 'success') toast.success(renameState.message ?? 'Saved');
  }, [renameState]);

  return (
    <div className="space-y-8">
      <form action={renameAction} className="max-w-md space-y-2">
        <label htmlFor="name" className="block text-sm font-medium">Name</label>
        <input id="name" name="name" defaultValue={account.name} className="input" />
        <FieldError errors={renameState.fieldErrors?.name} />
        {renameState.status === 'error' && renameState.message && <p className="text-sm text-red-600">{renameState.message}</p>}
        <button disabled={renaming} className="btn-primary">Save</button>
      </form>

      <form action={archiveAction} className="rounded border border-red-200 bg-red-50 p-4">
        <p className="text-sm">Archiving hides the account. The balance must be zero.</p>
        {archiveState.message && <p role="alert" className="mt-2 text-sm text-red-700">{archiveState.message}</p>}
        <button disabled={archiving} className="mt-2 rounded bg-red-600 px-3 py-1 text-white">Archive account</button>
      </form>
    </div>
  );
}
```

```tsx
// src/app/(app)/accounts/[id]/edit/page.tsx
import { notFound } from 'next/navigation';
import { getMyAccount } from '@/features/accounts/queries';
import { EditAccountForms } from '@/features/accounts/components/edit-account-forms';

export default async function EditAccountPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const account = await getMyAccount(id);
  if (!account) notFound();
  return (
    <section>
      <h1 className="mb-4 text-xl font-semibold">Edit {account.name}</h1>
      <EditAccountForms account={account} />
    </section>
  );
}
```

```mermaid
sequenceDiagram
  participant U as Browser form
  participant SA as createAccountAction
  participant S as accounts service
  participant DB as Postgres
  U->>SA: POST with action id and FormData
  SA->>SA: requireUser and Zod safeParse
  SA->>S: createAccount with userId and input
  S->>DB: INSERT Account
  DB-->>S: row
  S-->>SA: AccountDTO
  SA->>SA: revalidatePath /accounts
  SA-->>U: redirect to /accounts/id with fresh RSC payload
```

**Check it works:** create an account called "Travel". You land on `/accounts/<id>` (a 404 until Step 35 builds that page). Go back to `/accounts/new` and try "Travel" again: the form shows "You already have an account with this name" and keeps your other choices. Try archiving "Everyday" from its edit page: "Move the balance out before archiving."

### [Advanced] Step 31 — Create a transaction atomically

This is the heart of the app. Recording a debit must:

1. confirm the account belongs to the user,
2. reject it if the balance would go below zero,
3. update the balance and insert the transaction row **together**,
4. stay correct when two debits arrive at the same millisecond,
5. not double-charge when the client retries.

The trick for points 2 and 4 is a **conditional update**: `UPDATE ... SET balance = balance - x WHERE id = ? AND balance >= x`. Postgres locks the row while updating and re-checks the condition, so two concurrent debits cannot both pass. If no row was updated, the funds were insufficient. No read-then-write race.

```ts
// src/features/transactions/dto.ts
import type { Transaction } from '@/generated/prisma/client';

export type TransactionDTO = {
  id: string;
  accountId: string;
  type: 'CREDIT' | 'DEBIT';
  amountCents: number;
  description: string;
  balanceAfterCents: number;
  createdAt: string;
};

export function toTransactionDTO(t: Transaction): TransactionDTO {
  return {
    id: t.id,
    accountId: t.accountId,
    type: t.type,
    amountCents: t.amountCents,
    description: t.description,
    balanceAfterCents: t.balanceAfterCents,
    createdAt: t.createdAt.toISOString(),
  };
}
```

```ts
// src/features/transactions/schemas.ts
import { z } from 'zod';
import { parseAmountToCents } from '@/lib/money';

export const TRANSACTION_TYPES = ['CREDIT', 'DEBIT'] as const;

/** JSON API body: amounts already in cents. */
export const createTransactionBodySchema = z.object({
  type: z.enum(TRANSACTION_TYPES),
  amountCents: z.number().int().positive().max(100_000_000),
  description: z.string().trim().min(1, 'Required').max(140),
});
export type CreateTransactionBody = z.infer<typeof createTransactionBodySchema>;

/** HTML form: the user types "12.34". */
export const createTransactionFormSchema = z.object({
  type: z.enum(TRANSACTION_TYPES),
  amount: z.string().transform((value, ctx) => {
    const cents = parseAmountToCents(value);
    if (cents === null || cents <= 0) {
      ctx.addIssue({ code: 'custom', message: 'Enter an amount like 12.34' });
      return z.NEVER;
    }
    return cents;
  }),
  description: z.string().trim().min(1, 'Required').max(140),
});

export type TransactionFilters = {
  type?: 'CREDIT' | 'DEBIT';
  q?: string;
  page: number;
  pageSize: number;
};

/** URL search params. `.catch` means bad input falls back to defaults instead of erroring. */
export const transactionFiltersSchema: z.ZodType<TransactionFilters> = z.object({
  type: z.enum(TRANSACTION_TYPES).optional().catch(undefined),
  q: z.string().trim().max(50).optional().catch(undefined).transform((v) => v || undefined),
  page: z.coerce.number().int().min(1).catch(1),
  pageSize: z.coerce.number().int().min(5).max(100).catch(20),
});

export function filtersToSearchParams(f: TransactionFilters): URLSearchParams {
  const sp = new URLSearchParams();
  if (f.type) sp.set('type', f.type);
  if (f.q) sp.set('q', f.q);
  if (f.page > 1) sp.set('page', String(f.page));
  if (f.pageSize !== 20) sp.set('pageSize', String(f.pageSize));
  return sp;
}

/** Next.js gives string | string[] | undefined per key. Keep the first value. */
export function parseFilters(raw: Record<string, string | string[] | undefined>): TransactionFilters {
  const flat = Object.fromEntries(Object.entries(raw).map(([k, v]) => [k, Array.isArray(v) ? v[0] : v]));
  return transactionFiltersSchema.parse(flat);
}
```

> **Why is `TransactionFilters` written by hand?** With `.catch()` and `.transform()` in the chain, the inferred type makes `type` and `q` required keys whose value may be `undefined`, so `{ page: 1, pageSize: 20 }` would not type-check. Annotating the schema as `z.ZodType<TransactionFilters>` keeps one readable type and still makes TypeScript check the schema's output against it.

```ts
// src/features/transactions/service.ts
import 'server-only';
import { prisma, Prisma } from '@/lib/db';
import { ConflictError, InsufficientFundsError, NotFoundError } from '@/lib/errors';
import type { Paginated } from '@/lib/pagination';
import { getAccount } from '@/features/accounts/service';
import { toTransactionDTO, type TransactionDTO } from './dto';
import type { CreateTransactionBody, TransactionFilters } from './schemas';

const isUniqueViolation = (err: unknown) =>
  err instanceof Prisma.PrismaClientKnownRequestError && err.code === 'P2002';

export type CreateResult = { transaction: TransactionDTO; replayed: boolean };

export async function createTransaction(
  userId: string,
  accountId: string,
  input: CreateTransactionBody,
  idempotencyKey?: string,
): Promise<CreateResult> {
  if (idempotencyKey) {
    const existing = await prisma.transaction.findUnique({
      where: { idempotencyKey },
      include: { account: { select: { userId: true } } },
    });
    if (existing) {
      if (existing.accountId !== accountId || existing.account.userId !== userId) {
        throw new ConflictError('Idempotency key was already used for a different request');
      }
      return { transaction: toTransactionDTO(existing), replayed: true };
    }
  }

  try {
    const row = await prisma.$transaction(async (tx) => {
      const owned = await tx.account.findFirst({
        where: { id: accountId, userId, archivedAt: null },
        select: { id: true },
      });
      if (!owned) throw new NotFoundError('Account');

      const delta = input.type === 'CREDIT' ? input.amountCents : -input.amountCents;
      const { count } = await tx.account.updateMany({
        where: {
          id: accountId,
          ...(input.type === 'DEBIT' ? { balanceCents: { gte: input.amountCents } } : {}),
        },
        data: { balanceCents: { increment: delta } },
      });
      if (count === 0) throw new InsufficientFundsError();

      // We hold the row lock until commit, so this read is our own write.
      const { balanceCents } = await tx.account.findUniqueOrThrow({
        where: { id: accountId },
        select: { balanceCents: true },
      });

      return tx.transaction.create({
        data: {
          accountId,
          type: input.type,
          amountCents: input.amountCents,
          description: input.description,
          balanceAfterCents: balanceCents,
          createdById: userId,
          idempotencyKey: idempotencyKey ?? null,
        },
      });
    });
    return { transaction: toTransactionDTO(row), replayed: false };
  } catch (err) {
    // Two identical requests raced. The other one won, so return its result.
    if (idempotencyKey && isUniqueViolation(err)) {
      return createTransaction(userId, accountId, input, idempotencyKey);
    }
    throw err;
  }
}

export async function listTransactions(
  userId: string,
  accountId: string,
  filters: TransactionFilters,
): Promise<Paginated<TransactionDTO>> {
  await getAccount(userId, accountId); // ownership: throws NotFoundError

  const where: Prisma.TransactionWhereInput = {
    accountId,
    ...(filters.type ? { type: filters.type } : {}),
    ...(filters.q ? { description: { contains: filters.q, mode: 'insensitive' } } : {}),
  };

  const [rows, total] = await prisma.$transaction([
    prisma.transaction.findMany({
      where,
      orderBy: [{ createdAt: 'desc' }, { id: 'desc' }],
      skip: (filters.page - 1) * filters.pageSize,
      take: filters.pageSize,
    }),
    prisma.transaction.count({ where }),
  ]);

  return { data: rows.map(toTransactionDTO), meta: { page: filters.page, pageSize: filters.pageSize, total } };
}
```

```mermaid
sequenceDiagram
  participant C as Caller
  participant S as createTransaction
  participant DB as Postgres
  C->>S: DEBIT 50.00 with idempotency key
  S->>DB: find by idempotencyKey
  DB-->>S: none
  S->>DB: BEGIN
  S->>DB: SELECT account WHERE id and userId
  S->>DB: UPDATE balance minus 5000 WHERE balance at least 5000
  alt row updated
    DB-->>S: count 1
    S->>DB: INSERT Transaction with balanceAfter
    S->>DB: COMMIT
    S-->>C: transaction, replayed false
  else no row updated
    DB-->>S: count 0
    S->>DB: ROLLBACK
    S-->>C: InsufficientFundsError 422
  end
```

> **Finance tip:** Throwing inside the `$transaction` callback rolls everything back. That is why `NotFoundError` and `InsufficientFundsError` are thrown, not returned.

> **Gotcha:** Interactive transactions (the callback form) hold a database connection until they finish. Never call external APIs, send emails or emit socket events **inside** the callback. Do those after it resolves. The real-time guide relies on this rule.

> **Interview tip:** Asked "how do you prevent double spending?", name three layers: a conditional atomic update (or `SELECT ... FOR UPDATE`), a database `CHECK` constraint, and idempotency keys for retries.

### [Intermediate] Step 32 — Transaction route handlers

```ts
// src/app/api/v1/accounts/[id]/transactions/route.ts
import { withAuth, parseBody, parseQuery, json } from '@/lib/api/handler';
import { ValidationError } from '@/lib/errors';
import { createTransaction, listTransactions } from '@/features/transactions/service';
import { createTransactionBodySchema, transactionFiltersSchema } from '@/features/transactions/schemas';

type Params = { id: string };

export const GET = withAuth<Params>(async ({ req, user, params }) => {
  const filters = parseQuery(req, transactionFiltersSchema);
  return json(await listTransactions(user.id, params.id, filters));
});

export const POST = withAuth<Params>(async ({ req, user, params, log }) => {
  const key = req.headers.get('idempotency-key');
  if (key !== null && !/^[A-Za-z0-9-]{8,64}$/.test(key)) {
    throw new ValidationError(null, 'Idempotency-Key must be 8-64 letters, digits or dashes');
  }
  const body = await parseBody(req, createTransactionBodySchema);
  const { transaction, replayed } = await createTransaction(user.id, params.id, body, key ?? undefined);
  log.info({ transactionId: transaction.id, replayed }, 'transaction created');
  return json({ data: transaction }, { status: replayed ? 200 : 201 });
});
```

**Check it works:**

```bash
ID=$(curl -s -b "$C" http://localhost:3000/api/v1/accounts | jq -r '.data[0].id')
curl -s -b "$C" -X POST "http://localhost:3000/api/v1/accounts/$ID/transactions" \
  -H 'content-type: application/json' -H 'Idempotency-Key: test-key-0001' \
  -d '{"type":"DEBIT","amountCents":1000,"description":"Coffee"}' -w '\n%{http_code}\n'
# run the exact same command again
curl -s -b "$C" -X POST "http://localhost:3000/api/v1/accounts/$ID/transactions" \
  -H 'content-type: application/json' \
  -d '{"type":"DEBIT","amountCents":99999999,"description":"Yacht"}' | jq .error.code
```

```text
{"data":{"id":"cm...","type":"DEBIT","amountCents":1000,"balanceAfterCents":126901,...}}
201
{"data":{"id":"cm...", same id ...}}
200
"INSUFFICIENT_FUNDS"
```

The second identical call returns `200` with the **same** id and the balance only dropped once.

### [Intermediate] Step 33 — URL search params for filters and pagination

Filters belong in the URL: the page is shareable, the back button works, and a refresh keeps state. Start with a version that needs no client JavaScript at all: a plain `GET` form. The page reads `searchParams`, parses them with the same Zod schema the API uses, and calls the service.

```ts
// src/features/transactions/queries.ts
import 'server-only';
import { cache } from 'react';
import { requireUser } from '@/features/auth/guards';
import { listTransactions } from './service';
import type { TransactionFilters } from './schemas';

export const getMyTransactions = cache(async (accountId: string, filters: TransactionFilters) => {
  const user = await requireUser();
  return listTransactions(user.id, accountId, filters);
});
```

```tsx
// src/features/transactions/components/filters-form.tsx
import type { TransactionFilters } from '../schemas';

/** Works without JavaScript: submitting a GET form rewrites the query string. */
export function FiltersForm({ filters }: { filters: TransactionFilters }) {
  return (
    <form method="get" className="flex gap-2" role="search">
      <select name="type" defaultValue={filters.type ?? ''} className="input w-auto">
        <option value="">All types</option>
        <option value="CREDIT">Credits</option>
        <option value="DEBIT">Debits</option>
      </select>
      <input name="q" defaultValue={filters.q ?? ''} placeholder="Search description" className="input" />
      <button className="btn-secondary">Apply</button>
    </form>
  );
}
```

> **Why start without JavaScript?** It proves the server side is complete and correct: parsing, filtering and pagination all work from the URL alone. The interactive table in the next step is an enhancement on top, not a replacement for that contract.

### [Advanced] Step 34 — An interactive table with TanStack Query

For a snappier table you want: no full page navigation on each filter change, previous rows kept on screen while the next page loads, aborted stale requests, and background refresh. TanStack Query does this. The server still renders the first page and passes it as `initialData`, so there is no loading spinner on first paint.

Filters are read from `useSearchParams()` and written with `window.history.replaceState`. Next.js integrates the native History API with its router (since 14.1): `useSearchParams` updates, but no server round trip happens.

```ts
// src/features/transactions/hooks.ts
'use client';

import { useCallback, useMemo } from 'react';
import { useSearchParams } from 'next/navigation';
import { filtersToSearchParams, transactionFiltersSchema, type TransactionFilters } from './schemas';

export const transactionKeys = {
  all: (accountId: string) => ['transactions', accountId] as const,
  list: (accountId: string, f: TransactionFilters) => ['transactions', accountId, f] as const,
};

export function useTransactionFilters() {
  const searchParams = useSearchParams();
  const filters = useMemo(
    () => transactionFiltersSchema.parse(Object.fromEntries(searchParams)),
    [searchParams],
  );

  const setFilters = useCallback(
    (next: Partial<TransactionFilters>) => {
      // Changing a filter resets to page 1 unless the caller sets page.
      const merged: TransactionFilters = { ...filters, page: 1, ...next };
      const qs = filtersToSearchParams(merged).toString();
      window.history.replaceState(null, '', qs ? `?${qs}` : window.location.pathname);
    },
    [filters],
  );

  return [filters, setFilters] as const;
}
```

```tsx
// src/features/transactions/components/transactions-table.tsx
'use client';

import { formatCents } from '@/lib/money';
import type { TransactionDTO } from '../dto';

export type TransactionRow = TransactionDTO & { pending?: boolean };

export function TransactionsTable({ rows, currency }: { rows: TransactionRow[]; currency: string }) {
  if (rows.length === 0) return <p className="p-4 text-gray-600">No transactions match.</p>;
  return (
    <table className="w-full bg-white text-sm">
      <thead>
        <tr className="text-left text-gray-500">
          <th className="p-2">Date</th><th>Description</th><th className="text-right">Amount</th><th className="p-2 text-right">Balance</th>
        </tr>
      </thead>
      <tbody>
        {rows.map((t) => (
          <tr key={t.id} className={`border-t ${t.pending ? 'opacity-50' : ''}`} aria-busy={t.pending || undefined}>
            <td className="p-2">{new Date(t.createdAt).toLocaleDateString()}</td>
            <td>{t.description}</td>
            <td className={`text-right tabular-nums ${t.type === 'DEBIT' ? 'text-red-700' : 'text-green-700'}`}>
              {t.type === 'DEBIT' ? '-' : '+'}{formatCents(t.amountCents, currency)}
            </td>
            <td className="p-2 text-right tabular-nums">{t.pending ? 'Saving...' : formatCents(t.balanceAfterCents, currency)}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

### [Advanced] Step 35 — Optimistic create with useOptimistic

`useOptimistic` shows a value immediately while an async transition is running, then snaps back to the real state when it ends. You add a greyed-out row at once, call the Server Action, and when the transition finishes the real row (from the refetched query) replaces it. If the action fails, the optimistic row simply disappears.

```ts
// src/features/transactions/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { z } from 'zod';
import { requireUser } from '@/features/auth/guards';
import type { FormState } from '@/lib/form-state';
import { toFormError } from '@/lib/form-errors';
import { createTransactionFormSchema } from './schemas';
import { createTransaction } from './service';

export async function createTransactionAction(
  accountId: string,
  idempotencyKey: string,
  formData: FormData,
): Promise<FormState> {
  const user = await requireUser();
  const parsed = createTransactionFormSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { status: 'error', message: 'Check the highlighted fields', fieldErrors: z.flattenError(parsed.error).fieldErrors };
  }
  const { amount, ...rest } = parsed.data;
  try {
    await createTransaction(user.id, accountId, { ...rest, amountCents: amount }, idempotencyKey);
  } catch (err) {
    return toFormError(err);
  }
  revalidatePath(`/accounts/${accountId}`); // refreshes the server-rendered balance
  return { status: 'success', message: 'Transaction recorded' };
}
```

```tsx
// src/features/transactions/components/transactions-panel.tsx
'use client';

import { useOptimistic, useState, useTransition, type FormEvent } from 'react';
import { keepPreviousData, useQuery, useQueryClient } from '@tanstack/react-query';
import { toast } from 'sonner';
import { api } from '@/lib/api-client/endpoints';
import type { Paginated } from '@/lib/pagination';
import { idleState, type FormState } from '@/lib/form-state';
import { FieldError } from '@/features/auth/components/field-error';
import { createTransactionAction } from '../actions';
import { createTransactionFormSchema, type TransactionFilters } from '../schemas';
import type { TransactionDTO } from '../dto';
import { transactionKeys, useTransactionFilters } from '../hooks';
import { TransactionsTable, type TransactionRow } from './transactions-table';

type Props = {
  accountId: string;
  currency: string;
  initialData: Paginated<TransactionDTO>;
  initialFilters: TransactionFilters;
};

const sameFilters = (a: TransactionFilters, b: TransactionFilters) =>
  a.type === b.type && a.q === b.q && a.page === b.page && a.pageSize === b.pageSize;

export function TransactionsPanel({ accountId, currency, initialData, initialFilters }: Props) {
  const [filters, setFilters] = useTransactionFilters();
  const queryClient = useQueryClient();
  const [formState, setFormState] = useState<FormState>(idleState);
  const [isPending, startTransition] = useTransition();

  const query = useQuery({
    queryKey: transactionKeys.list(accountId, filters),
    queryFn: ({ signal }) => api.transactions.list(accountId, filters, signal),
    initialData: sameFilters(filters, initialFilters) ? initialData : undefined,
    placeholderData: keepPreviousData,
  });

  const rows: TransactionRow[] = query.data?.data ?? [];
  const [optimisticRows, addOptimisticRow] = useOptimistic(rows, (state, row: TransactionRow) => [row, ...state]);

  function onSubmit(e: FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const form = e.currentTarget;
    const formData = new FormData(form);
    const idempotencyKey = crypto.randomUUID();
    const preview = createTransactionFormSchema.safeParse(Object.fromEntries(formData));

    startTransition(async () => {
      if (preview.success) {
        addOptimisticRow({
          id: `optimistic-${idempotencyKey}`,
          accountId,
          type: preview.data.type,
          amountCents: preview.data.amount,
          description: preview.data.description,
          balanceAfterCents: 0,
          createdAt: new Date().toISOString(),
          pending: true,
        });
      }
      const result = await createTransactionAction(accountId, idempotencyKey, formData);
      setFormState(result);
      if (result.status === 'error') {
        toast.error(result.message ?? 'Could not save the transaction');
        return; // optimistic row disappears when the transition ends
      }
      form.reset();
      toast.success('Transaction recorded');
      await queryClient.invalidateQueries({ queryKey: transactionKeys.all(accountId) });
    });
  }

  const total = query.data?.meta.total ?? 0;
  const lastPage = Math.max(1, Math.ceil(total / filters.pageSize));

  return (
    <div className="space-y-4">
      <form onSubmit={onSubmit} className="flex flex-wrap items-start gap-2 rounded border bg-white p-3" aria-label="New transaction">
        <select name="type" defaultValue="DEBIT" className="input w-auto" aria-label="Type">
          <option value="DEBIT">Debit</option>
          <option value="CREDIT">Credit</option>
        </select>
        <div>
          <input name="amount" inputMode="decimal" placeholder="0.00" className="input w-32" aria-label="Amount" />
          <FieldError errors={formState.fieldErrors?.amount} />
        </div>
        <div className="flex-1">
          <input name="description" placeholder="Description" className="input" aria-label="Description" />
          <FieldError errors={formState.fieldErrors?.description} />
        </div>
        <button disabled={isPending} className="btn-primary">{isPending ? 'Saving...' : 'Add'}</button>
      </form>

      <div className="flex gap-2">
        <select
          value={filters.type ?? ''}
          onChange={(e) => setFilters({ type: e.target.value === '' ? undefined : (e.target.value as 'CREDIT' | 'DEBIT') })}
          className="input w-auto"
          aria-label="Filter by type"
        >
          <option value="">All types</option>
          <option value="CREDIT">Credits</option>
          <option value="DEBIT">Debits</option>
        </select>
        <input
          key={filters.q ?? ''}
          defaultValue={filters.q ?? ''}
          onKeyDown={(e) => { if (e.key === 'Enter') setFilters({ q: e.currentTarget.value.trim() || undefined }); }}
          placeholder="Search, then Enter"
          className="input"
          aria-label="Search description"
        />
      </div>

      <div className={query.isFetching ? 'opacity-70 transition-opacity' : ''}>
        <TransactionsTable rows={optimisticRows} currency={currency} />
      </div>

      <nav className="flex items-center justify-between text-sm" aria-label="Pagination">
        <button className="btn-secondary" disabled={filters.page <= 1} onClick={() => setFilters({ page: filters.page - 1 })}>Previous</button>
        <span>Page {filters.page} of {lastPage} ({total} total)</span>
        <button className="btn-secondary" disabled={filters.page >= lastPage} onClick={() => setFilters({ page: filters.page + 1 })}>Next</button>
      </nav>
    </div>
  );
}
```

> **Gotcha:** This form uses `onSubmit` plus `startTransition`, not `<form action={fn}>`. In React 19 a form with a function `action` **auto-resets its fields** when the action finishes, even if the server returned validation errors, which wipes what the user typed. Calling the action yourself lets you reset only on success. `addOptimisticRow` must be called inside a transition, which `startTransition` provides.

> **Why invalidate if `revalidatePath` already ran?** `revalidatePath` refreshes **server-rendered** output (the balance header). The table's data lives in TanStack Query's client cache, which Next.js does not know about. Each cache needs its own invalidation.

Finally the account page ties it together:

```tsx
// src/app/(app)/accounts/[id]/page.tsx
import Link from 'next/link';
import { notFound } from 'next/navigation';
import { getMyAccount } from '@/features/accounts/queries';
import { getMyTransactions } from '@/features/transactions/queries';
import { parseFilters } from '@/features/transactions/schemas';
import { TransactionsPanel } from '@/features/transactions/components/transactions-panel';
import { formatCents } from '@/lib/money';

type Props = {
  params: Promise<{ id: string }>;
  searchParams: Promise<Record<string, string | string[] | undefined>>;
};

export default async function AccountPage({ params, searchParams }: Props) {
  const { id } = await params;
  const account = await getMyAccount(id);
  if (!account) notFound();

  const filters = parseFilters(await searchParams);
  const initialData = await getMyTransactions(id, filters);

  return (
    <section className="space-y-6">
      <div className="flex items-end justify-between">
        <div>
          <p className="text-sm text-gray-500">{account.type}</p>
          <h1 className="text-xl font-semibold">{account.name}</h1>
        </div>
        <div className="text-right">
          <p className="text-sm text-gray-500">Balance</p>
          <p className="text-3xl tabular-nums" data-testid="balance">{formatCents(account.balanceCents, account.currency)}</p>
          <Link href={`/accounts/${id}/edit`} className="text-sm underline">Edit</Link>
        </div>
      </div>
      <TransactionsPanel accountId={id} currency={account.currency} initialData={initialData} initialFilters={filters} />
    </section>
  );
}
```

`FiltersForm` from Step 33 is still useful on pages that do not need interactivity (an admin audit view, a printable statement). Here the panel's controls replace it.

```mermaid
stateDiagram-v2
  [*] --> Idle
  Idle --> Optimistic: submit, addOptimisticRow
  Optimistic --> Saving: server action running
  Saving --> Refetching: success, invalidateQueries
  Refetching --> Idle: real row replaces optimistic row
  Saving --> Idle: error, optimistic row removed, toast
```

**Check it works:** open "Everyday", add a debit of `12.50` "Lunch". The row appears instantly in grey with "Saving...", then turns solid with the new balance, and the header balance updates. Add a debit of `99999` and the grey row flashes and disappears with the toast "Insufficient funds for this debit". Choose "Debits" in the filter: the URL becomes `?type=DEBIT` without a page reload. Refresh: the filter is still applied (server rendered).

Milestone tree:

```text
src/
├─ app/(app)/accounts/
│  ├─ page.tsx
│  ├─ new/page.tsx
│  └─ [id]/{page.tsx, edit/page.tsx}
├─ app/api/v1/accounts/{route.ts, [id]/route.ts, [id]/transactions/route.ts}
├─ features/accounts/{actions.ts, dto.ts, queries.ts, schemas.ts, service.ts, components/}
├─ features/transactions/{actions.ts, dto.ts, hooks.ts, queries.ts, schemas.ts, service.ts, components/}
└─ lib/{api/, api-client/, errors.ts, form-errors.ts, form-state.ts, logger.ts, money.ts, pagination.ts}
```

## 6. Cross-cutting concerns

### [Beginner] Step 36 — Error boundaries and not-found pages

Next.js wraps route segments in React error boundaries for you. You supply the UI:

| File | Catches | Notes |
|---|---|---|
| `error.tsx` | errors thrown while rendering that segment and its children | must be a Client Component, gets `reset()` |
| `global-error.tsx` | errors in the root layout itself | replaces the whole document, so it renders `<html>` and `<body>` |
| `not-found.tsx` | `notFound()` calls and unknown URLs | can be a Server Component |

```tsx
// src/app/(app)/error.tsx
'use client';

import { useEffect } from 'react';

export default function AppError({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  useEffect(() => {
    console.error(error); // server-side details are already in the server log, matched by digest
  }, [error]);

  return (
    <div role="alert" className="rounded border border-red-200 bg-red-50 p-6">
      <h2 className="text-lg font-semibold">Something went wrong</h2>
      <p className="mt-1 text-sm">Try again. If it keeps happening, contact support with this reference: {error.digest ?? 'n/a'}</p>
      <button onClick={reset} className="btn-primary mt-4">Try again</button>
    </div>
  );
}
```

```tsx
// src/app/global-error.tsx
'use client';

export default function GlobalError({ error, reset }: { error: Error & { digest?: string }; reset: () => void }) {
  return (
    <html lang="en">
      <body style={{ fontFamily: 'system-ui', padding: 32 }}>
        <h1>Ledger is having trouble</h1>
        <p>Reference: {error.digest ?? 'n/a'}</p>
        <button onClick={reset}>Reload</button>
      </body>
    </html>
  );
}
```

```tsx
// src/app/not-found.tsx
import Link from 'next/link';

export default function NotFound() {
  return (
    <main className="mx-auto mt-24 max-w-md text-center">
      <h1 className="text-2xl font-semibold">Not found</h1>
      <p className="mt-2 text-gray-600">That page or account does not exist, or you do not have access to it.</p>
      <Link href="/accounts" className="mt-4 inline-block underline">Back to accounts</Link>
    </main>
  );
}
```

> **Why the digest?** In production Next.js replaces server error messages with a generic message and a `digest` hash, so secrets in error messages never reach the browser. The same digest appears in the server log, which links a user report to the stack trace.

Log every unhandled server error in one place with the `onRequestError` instrumentation hook:

```ts
// src/instrumentation.ts
import type { Instrumentation } from 'next';

export const onRequestError: Instrumentation.onRequestError = async (err, request, context) => {
  const { logger } = await import('@/lib/logger');
  logger.error(
    { err, path: request.path, method: request.method, routeType: context.routeType, requestId: request.headers['x-request-id'] },
    'unhandled server error',
  );
};
```

**Check it works:** temporarily add `throw new Error('boom')` at the top of `AccountsPage`. The page shows "Something went wrong" with the header still visible (the boundary is below the layout), and the terminal logs `unhandled server error` with the request id. Remove the line. Visit `/accounts/nope` and you see the not-found page.

### [Beginner] Step 37 — Loading skeletons and streaming

A `loading.tsx` file wraps the segment's page in `<Suspense>`. Next.js sends the layout and the skeleton immediately, then streams the page when its data is ready.

```tsx
// src/app/(app)/accounts/loading.tsx
export default function Loading() {
  return (
    <div className="grid gap-3 sm:grid-cols-2" aria-busy="true" aria-label="Loading accounts">
      {Array.from({ length: 4 }, (_, i) => (
        <div key={i} className="h-28 animate-pulse rounded border bg-gray-200" />
      ))}
    </div>
  );
}
```

```tsx
// src/app/(app)/accounts/[id]/loading.tsx
export default function Loading() {
  return (
    <div className="space-y-4" aria-busy="true">
      <div className="h-16 w-1/2 animate-pulse rounded bg-gray-200" />
      {Array.from({ length: 8 }, (_, i) => <div key={i} className="h-8 animate-pulse rounded bg-gray-200" />)}
    </div>
  );
}
```

For finer control, wrap one slow part in `<Suspense fallback={...}>` inside a page, so the rest renders first.

**Check it works:** add `await new Promise((r) => setTimeout(r, 1500));` at the top of `AccountsPage`, navigate to `/accounts`, and you see four grey cards before the real list. Remove the delay.

### [Beginner] Step 38 — Toasts as consistent feedback

`sonner`'s `<Toaster>` is already in `Providers`. The rules for this app:

- **Success of a user action:** `toast.success` from the client after the action returns `status: 'success'`.
- **Expected business error** (insufficient funds, duplicate name): an inline message next to the field or form, plus `toast.error` when the form is small.
- **Unexpected 5xx from the API:** the axios interceptor toasts once with the request id. Components do not repeat it.
- **Never** toast from Server Components. They have no browser. Return state and let the client decide.

### [Intermediate] Step 39 — Request-scoped logging in Server Actions

Route handlers get a request-scoped logger from `withErrorHandling`. Server Actions need one too. The proxy put `x-request-id` on the incoming request headers, so read it back:

```ts
// src/lib/request-logger.ts
import 'server-only';
import { headers } from 'next/headers';
import { logger } from '@/lib/logger';

export async function getRequestLogger(bindings: Record<string, unknown> = {}) {
  const h = await headers();
  return logger.child({ requestId: h.get('x-request-id') ?? 'none', ...bindings });
}
```

Use it in the transaction action, right after `requireUser()`:

```ts
// src/features/transactions/actions.ts (inside createTransactionAction)
const log = await getRequestLogger({ userId: user.id, accountId, action: 'createTransaction' });
// ...after a successful createTransaction:
log.info({ idempotencyKey }, 'transaction recorded via action');
// ...inside the catch, before return toFormError(err):
log.warn({ err }, 'transaction rejected');
```

Add `import { getRequestLogger } from '@/lib/request-logger';` at the top of the file.

**Check it works:** add a transaction in the UI. The dev terminal shows a pretty-printed line with `requestId`, `userId`, `accountId` and `action`. The same `requestId` appears on the response header `x-request-id` in the Network tab.

> **Finance tip:** Never log full card numbers, passwords or session tokens. The `redact` paths in `logger.ts` are a safety net, not a licence to log whole request bodies.

### [Intermediate] Step 40 — Rate limit the login

Credential stuffing tries thousands of passwords per minute. Limit attempts per IP and email. A simple fixed-window counter in memory is enough for one server:

```ts
// src/lib/rate-limit.ts
import 'server-only';

type Bucket = { count: number; resetAt: number };
type Result = { ok: true } | { ok: false; retryAfterSec: number };

const g = globalThis as unknown as { __ledgerRateLimit?: Map<string, Bucket> };
const buckets = (g.__ledgerRateLimit ??= new Map<string, Bucket>());

export function rateLimit(key: string, opts: { limit: number; windowMs: number }): Result {
  const now = Date.now();
  const bucket = buckets.get(key);

  if (!bucket || bucket.resetAt <= now) {
    buckets.set(key, { count: 1, resetAt: now + opts.windowMs });
    if (buckets.size > 10_000) {
      for (const [k, b] of buckets) if (b.resetAt <= now) buckets.delete(k);
    }
    return { ok: true };
  }
  if (bucket.count >= opts.limit) {
    return { ok: false, retryAfterSec: Math.ceil((bucket.resetAt - now) / 1000) };
  }
  bucket.count += 1;
  return { ok: true };
}
```

Wire it into `loginAction` right after the Zod check:

```ts
// src/features/auth/actions.ts (inside loginAction, after `if (!parsed.success) {...}`)
const h = await headers();
const ip = h.get('x-forwarded-for')?.split(',')[0]?.trim() ?? 'unknown';
const limit = rateLimit(`login:${ip}:${parsed.data.email}`, { limit: 5, windowMs: 60_000 });
if (!limit.ok) {
  return { status: 'error', message: `Too many attempts. Try again in ${limit.retryAfterSec} seconds.` };
}
```

Add the imports `import { headers } from 'next/headers';` and `import { rateLimit } from '@/lib/rate-limit';`.

> **Gotcha:** In-memory state is per process. With two containers, or on serverless where every instance is fresh, an attacker gets N times the limit. Use a shared store: `@upstash/ratelimit` with Upstash Redis (`Ratelimit.slidingWindow(5, '1 m')`) works from serverless functions, or Redis `INCR` plus `EXPIRE` in a container setup.

> **Gotcha:** Only trust `x-forwarded-for` when your own load balancer sets it. Otherwise the client can send any value and pick a fresh "IP" per request. Keying by email as well limits that damage.

**Check it works:** enter a wrong password six times quickly. The sixth attempt shows "Too many attempts. Try again in 5x seconds."

### [Intermediate] Step 41 — Security headers, CSP and CSRF

Add headers in `next.config.ts`. They cost nothing and close whole classes of attacks.

```ts
// next.config.ts
import type { NextConfig } from 'next';

const isDev = process.env.NODE_ENV !== 'production';

const csp = [
  "default-src 'self'",
  `script-src 'self' 'unsafe-inline'${isDev ? " 'unsafe-eval'" : ''}`,
  "style-src 'self' 'unsafe-inline'",
  "img-src 'self' data: https://avatars.githubusercontent.com",
  "font-src 'self'",
  "connect-src 'self'",
  "frame-ancestors 'none'",
  "base-uri 'self'",
  "form-action 'self' https://github.com https://*.okta.com",
  "object-src 'none'",
].join('; ');

const securityHeaders = [
  { key: 'Content-Security-Policy', value: csp },
  { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
];

const nextConfig: NextConfig = {
  output: 'standalone',
  serverExternalPackages: ['pino', 'pino-pretty'],
  poweredByHeader: false,
  async headers() {
    return [{ source: '/:path*', headers: securityHeaders }];
  },
};

export default nextConfig;
```

| Header | Stops |
|---|---|
| `Content-Security-Policy` | injected scripts loading from other origins, data exfiltration via `connect-src` |
| `frame-ancestors 'none'` / `X-Frame-Options` | clickjacking: your bank page inside an invisible iframe |
| `Strict-Transport-Security` | downgrade to plain HTTP |
| `X-Content-Type-Options` | browsers guessing a JSON file is a script |

> **Gotcha:** `'unsafe-inline'` in `script-src` is a compromise: Next.js injects inline scripts for hydration. The strict alternative is a **nonce**: generate one per request in `proxy.ts`, set `script-src 'nonce-...' 'strict-dynamic'` on the request and response headers, and Next.js adds the nonce to its own scripts automatically. Nonces force every page to render dynamically, which is fine for an authenticated app like this one. See the Next.js "Content Security Policy" guide for the exact proxy code.

> **Gotcha:** `form-action` also applies to redirects after a form submission in Chromium. Without the GitHub and Okta hosts, the OAuth sign-in button silently does nothing.

**CSRF (cross-site request forgery)**: another site makes the user's browser send a request with their cookies.

- **Server Actions** only accept `POST`, and Next.js compares the `Origin` header with the `Host` (or `X-Forwarded-Host`) and rejects mismatches. Behind a proxy that rewrites hosts, list trusted origins in `experimental.serverActions.allowedOrigins`.
- **Auth.js endpoints** use their own double-submit CSRF token, and the session cookie is `SameSite=Lax`, so cross-site `POST`s do not carry it.
- **Your route handlers** are protected by `SameSite=Lax` plus requiring `Content-Type: application/json` (a cross-site HTML form cannot send that without a CORS preflight). For extra safety, reject mutating requests whose `Origin` is not your own.

**Check it works:**

```bash
curl -sI http://localhost:3000/login | grep -i -E 'content-security|x-frame|strict-transport'
```

```text
content-security-policy: default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; ...
strict-transport-security: max-age=63072000; includeSubDomains; preload
x-frame-options: DENY
```

## 7. Testing and delivery

```mermaid
flowchart TD
  A["Playwright e2e<br/>login, create transaction"] --> B["Component tests<br/>RTL with mocked API and action"]
  B --> C["Unit tests<br/>services and money with mocked Prisma"]
  C --> D["Database constraints<br/>last line of defense"]
```

### [Intermediate] Step 42 — Vitest setup and service unit tests

```ts
// vitest.config.ts
import { fileURLToPath } from 'node:url';
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  resolve: {
    // `server-only` throws outside Next.js. Replace it with an empty module in tests.
    alias: { 'server-only': fileURLToPath(new URL('./tests/server-only-stub.ts', import.meta.url)) },
  },
  test: {
    environment: 'jsdom',
    setupFiles: ['./tests/setup.ts'],
    include: ['src/**/*.test.{ts,tsx}'],
    env: {
      NODE_ENV: 'test',
      DATABASE_URL: 'postgresql://test:test@localhost:5432/test',
      AUTH_SECRET: 'test-secret-test-secret-test-secret-123',
    },
  },
});
```

```ts
// tests/server-only-stub.ts
export {};
```

```ts
// tests/setup.ts
import '@testing-library/jest-dom/vitest';
```

Start with the pure function. No mocks needed:

```ts
// src/lib/money.test.ts
import { describe, expect, it } from 'vitest';
import { formatCents, parseAmountToCents } from './money';

describe('parseAmountToCents', () => {
  it.each([
    ['12', 1200],
    ['12.3', 1230],
    ['19.99', 1999],
    ['1,000.05', 100005],
    [' 0.01 ', 1],
  ])('parses %s to %i cents', (input, cents) => {
    expect(parseAmountToCents(input)).toBe(cents);
  });

  it.each(['', 'abc', '1.234', '-5', '1e3'])('rejects %s', (input) => {
    expect(parseAmountToCents(input)).toBeNull();
  });
});

describe('formatCents', () => {
  it('formats USD', () => {
    expect(formatCents(127901, 'USD')).toBe('$1,279.01');
  });
});
```

Then the service. Mock the Prisma module and make `$transaction` call the callback with a fake transaction client:

```ts
// src/features/transactions/service.test.ts
// @vitest-environment node
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { InsufficientFundsError, NotFoundError } from '@/lib/errors';

const { prismaMock, txMock } = vi.hoisted(() => {
  const txMock = {
    account: { findFirst: vi.fn(), updateMany: vi.fn(), findUniqueOrThrow: vi.fn() },
    transaction: { create: vi.fn() },
  };
  const prismaMock = {
    transaction: { findUnique: vi.fn() },
    $transaction: vi.fn(async (fn: (tx: typeof txMock) => unknown) => fn(txMock)),
  };
  return { prismaMock, txMock };
});

vi.mock('@/lib/db', () => ({
  prisma: prismaMock,
  Prisma: { PrismaClientKnownRequestError: class extends Error { code = ''; } },
}));

import { createTransaction } from './service';

const row = (over: Partial<Record<string, unknown>> = {}) => ({
  id: 'tx_1', accountId: 'acc_1', type: 'DEBIT', amountCents: 500, description: 'Coffee',
  balanceAfterCents: 9500, createdById: 'user_1', idempotencyKey: null, createdAt: new Date('2026-10-01'), ...over,
});

describe('createTransaction', () => {
  beforeEach(() => {
    vi.clearAllMocks();
    txMock.account.findFirst.mockResolvedValue({ id: 'acc_1' });
  });

  it('debits and records the balance after', async () => {
    txMock.account.updateMany.mockResolvedValue({ count: 1 });
    txMock.account.findUniqueOrThrow.mockResolvedValue({ balanceCents: 9500 });
    txMock.transaction.create.mockResolvedValue(row());

    const result = await createTransaction('user_1', 'acc_1', { type: 'DEBIT', amountCents: 500, description: 'Coffee' });

    expect(txMock.account.updateMany).toHaveBeenCalledWith({
      where: { id: 'acc_1', balanceCents: { gte: 500 } },
      data: { balanceCents: { increment: -500 } },
    });
    expect(result).toMatchObject({ replayed: false, transaction: { balanceAfterCents: 9500 } });
  });

  it('rejects an overdraft and writes nothing', async () => {
    txMock.account.updateMany.mockResolvedValue({ count: 0 });

    await expect(
      createTransaction('user_1', 'acc_1', { type: 'DEBIT', amountCents: 999_999, description: 'Yacht' }),
    ).rejects.toBeInstanceOf(InsufficientFundsError);
    expect(txMock.transaction.create).not.toHaveBeenCalled();
  });

  it('hides accounts the user does not own', async () => {
    txMock.account.findFirst.mockResolvedValue(null);
    await expect(
      createTransaction('intruder', 'acc_1', { type: 'CREDIT', amountCents: 100, description: 'x' }),
    ).rejects.toBeInstanceOf(NotFoundError);
  });

  it('replays an idempotent request without touching balances', async () => {
    prismaMock.transaction.findUnique.mockResolvedValue({ ...row({ idempotencyKey: 'key-12345' }), account: { userId: 'user_1' } });

    const result = await createTransaction('user_1', 'acc_1', { type: 'DEBIT', amountCents: 500, description: 'Coffee' }, 'key-12345');

    expect(result.replayed).toBe(true);
    expect(prismaMock.$transaction).not.toHaveBeenCalled();
  });
});
```

**Check it works:**

```bash
npm test
```

```text
 ✓ src/lib/money.test.ts (11 tests)
 ✓ src/features/transactions/service.test.ts (4 tests)
 Test Files  2 passed (2)
      Tests  15 passed (15)
```

> **Why mocks here and a real database elsewhere?** These tests check **your** branching logic in milliseconds. They cannot prove the SQL is right or that the row lock prevents races. Add a few integration tests that run against a throwaway Postgres (a second docker-compose service or Testcontainers) for that, and run them in CI.

### [Intermediate] Step 43 — A component test with React Testing Library

Test the panel the way a user uses it: type, click, see. Mock the network edges (the API module, the Server Action, the router hook), not React.

```tsx
// src/features/transactions/components/transactions-panel.test.tsx
import { describe, expect, it, vi, beforeEach } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { toast } from 'sonner';
import type { FormState } from '@/lib/form-state';
import { api } from '@/lib/api-client/endpoints';
import { createTransactionAction } from '../actions';
import { TransactionsPanel } from './transactions-panel';

vi.mock('next/navigation', () => ({ useSearchParams: () => new URLSearchParams() }));
vi.mock('@/lib/api-client/endpoints', () => ({ api: { transactions: { list: vi.fn() } } }));
vi.mock('../actions', () => ({ createTransactionAction: vi.fn() }));
vi.mock('sonner', () => ({ toast: { success: vi.fn(), error: vi.fn() } }));

const initialData = {
  data: [{ id: 't1', accountId: 'acc_1', type: 'CREDIT' as const, amountCents: 250000, description: 'Salary', balanceAfterCents: 250000, createdAt: '2026-10-01T00:00:00.000Z' }],
  meta: { page: 1, pageSize: 20, total: 1 },
};

function renderPanel() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(
    <QueryClientProvider client={client}>
      <TransactionsPanel accountId="acc_1" currency="USD" initialData={initialData} initialFilters={{ page: 1, pageSize: 20 }} />
    </QueryClientProvider>,
  );
}

describe('TransactionsPanel', () => {
  beforeEach(() => {
    vi.clearAllMocks();
    vi.mocked(api.transactions.list).mockResolvedValue(initialData);
  });

  it('renders the server-provided first page', () => {
    renderPanel();
    expect(screen.getByText('Salary')).toBeInTheDocument();
    expect(screen.getByText('Page 1 of 1 (1 total)')).toBeInTheDocument();
  });

  it('shows an optimistic row, then removes it when the action fails', async () => {
    let finish!: (s: FormState) => void;
    vi.mocked(createTransactionAction).mockReturnValue(new Promise<FormState>((r) => { finish = r; }));
    const user = userEvent.setup();
    renderPanel();

    await user.type(screen.getByLabelText('Amount'), '12.50');
    await user.type(screen.getByLabelText('Description'), 'Lunch');
    await user.click(screen.getByRole('button', { name: 'Add' }));

    expect(await screen.findByText('Lunch')).toBeInTheDocument();
    expect(screen.getByText('Saving...', { selector: 'td' })).toBeInTheDocument();
    expect(createTransactionAction).toHaveBeenCalledWith('acc_1', expect.any(String), expect.any(FormData));

    finish({ status: 'error', message: 'Insufficient funds for this debit' });

    await waitFor(() => expect(screen.queryByText('Lunch')).not.toBeInTheDocument());
    expect(toast.error).toHaveBeenCalledWith('Insufficient funds for this debit');
    expect(screen.getByLabelText('Description')).toHaveValue('Lunch'); // input kept on error
  });
});
```

**Check it works:** `npm test` now reports 3 files and 17 tests passing.

### [Intermediate] Step 44 — End-to-end test with Playwright

```bash
npx playwright install --with-deps chromium
```

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: 'tests/e2e',
  fullyParallel: false,
  retries: process.env.CI ? 1 : 0,
  use: { baseURL: 'http://localhost:3000', trace: 'on-first-retry' },
  webServer: {
    command: process.env.CI ? 'npm run build && npm run start' : 'npm run dev',
    url: 'http://localhost:3000/api/health',
    reuseExistingServer: !process.env.CI,
    timeout: 180_000,
  },
  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
});
```

```ts
// tests/e2e/ledger.spec.ts
import { expect, test } from '@playwright/test';

test('sign in and record a transaction', async ({ page }) => {
  await page.goto('/accounts');
  await expect(page).toHaveURL(/\/login\?callbackUrl=/);

  await page.getByLabel('Email').fill('alice@example.com');
  await page.getByLabel('Password').fill('Password123!');
  await page.getByRole('button', { name: 'Sign in' }).click();

  await expect(page.getByRole('heading', { name: 'Your accounts' })).toBeVisible();
  await page.getByRole('link', { name: /Everyday/ }).click();

  const balance = page.getByTestId('balance');
  const before = (await balance.textContent()) ?? '';
  const description = `E2E coffee ${Date.now()}`;

  await page.getByLabel('Amount').fill('1.00');
  await page.getByLabel('Description').fill(description);
  await page.getByRole('button', { name: 'Add' }).click();

  await expect(page.getByText('Transaction recorded')).toBeVisible();
  await expect(page.getByRole('cell', { name: description })).toBeVisible();
  await expect(balance).not.toHaveText(before);
});

test('rejects an overdraft', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('alice@example.com');
  await page.getByLabel('Password').fill('Password123!');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await page.getByRole('link', { name: /Rainy day/ }).click();

  await page.getByLabel('Amount').fill('5.00');
  await page.getByLabel('Description').fill('Should fail');
  await page.getByRole('button', { name: 'Add' }).click();

  await expect(page.getByText('Insufficient funds for this debit')).toBeVisible();
  await expect(page.getByRole('cell', { name: 'Should fail' })).toHaveCount(0);
});
```

**Check it works:** with the database up and seeded:

```bash
npm run e2e
```

```text
Running 2 tests using 1 worker
  ✓  1 [chromium] › ledger.spec.ts:3:1 › sign in and record a transaction (3.1s)
  ✓  2 [chromium] › ledger.spec.ts:30:1 › rejects an overdraft (1.4s)
  2 passed (6.2s)
```

> **Gotcha:** The rate limiter from Step 40 counts e2e logins too. Keep the number of login tests under the limit, or sign in once and reuse the cookies with Playwright's `storageState` setup project, which is also much faster.

### [Intermediate] Step 45 — A production Dockerfile with output: 'standalone'

`output: 'standalone'` makes `next build` copy only the files and `node_modules` your server actually uses into `.next/standalone`, with a minimal `server.js`. Images shrink from about 1 GB to about 150 MB.

```dockerfile
# Dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1 SKIP_ENV_VALIDATION=1
# Prisma generate reads the config. A dummy URL is enough at build time.
ENV DATABASE_URL=postgresql://build:build@localhost:5432/build
RUN npm run build

# One-off job image: runs migrations with the full Prisma CLI.
FROM builder AS migrate
CMD ["npx", "prisma", "migrate", "deploy"]

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1 PORT=3000 HOSTNAME=0.0.0.0
RUN addgroup -S nodejs && adduser -S nextjs -G nodejs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://127.0.0.1:3000/api/health || exit 1
CMD ["node", "server.js"]
```

```text
# .dockerignore
node_modules
.next
.git
.env
src/generated
playwright-report
test-results
```

**Check it works:**

```bash
docker build -t ledger-web .
docker build --target migrate -t ledger-web-migrate .
docker run --rm --network host -e DATABASE_URL="postgresql://ledger:ledger@localhost:5432/ledger" ledger-web-migrate
docker run --rm --network host -e DATABASE_URL="postgresql://ledger:ledger@localhost:5432/ledger" \
  -e AUTH_SECRET="$(grep AUTH_SECRET .env | cut -d'"' -f2)" ledger-web
```

```text
   ▲ Next.js 16.x
   - Local:        http://0.0.0.0:3000
 ✓ Ready in 120ms
```

> **Gotcha:** The standalone `server.js` does not run migrations, and the runner image has no Prisma CLI. Run `prisma migrate deploy` as a separate step (the `migrate` target, a Kubernetes Job, an ECS one-off task) **before** rolling out new app containers.

### [Beginner] Step 46 — Deploy: Vercel or a container

| | Vercel | Container (ECS, Cloud Run, Kubernetes, Fly) |
|---|---|---|
| Setup | `git push`, zero config | Dockerfile, registry, service, load balancer |
| Scaling | automatic per request, serverless functions | you set min and max instances |
| Long connections (WebSockets) | not supported in functions | supported |
| In-memory state (rate limit, caches) | not shared, instances come and go | shared within one instance |
| Database connections | many short-lived instances: use a pooler (PgBouncer, Prisma Postgres, Neon pooled URL) | a steady pool per instance |
| Preview deployments per PR | built in | build it yourself |
| Cost model | per usage, can spike | per instance hour, predictable |

Production checklist either way:

- Set `AUTH_SECRET`, `DATABASE_URL`, provider secrets and `AUTH_URL` (your public URL) in the platform's secret store.
- Run migrations before the new version takes traffic.
- Put a connection pooler in front of Postgres if you have many instances.
- Ship logs (stdout JSON) to your log platform and alert on `level >= 50` (error).
- Replace the in-memory rate limiter with Redis or Upstash when you run more than one instance.

> **Why does this matter for the next guide?** Real-time with Socket.IO needs a long-lived Node process. That pushes ledger-web towards the **container** column, or towards a separate realtime service next to a Vercel deployment.

## 8. Interview questions

#### Q: What is the difference between a Server Component, a Client Component and a Server Action?

A Server Component renders on the server only. It can read the database and secrets, and ships no JavaScript for itself. A Client Component (`'use client'`) is also pre-rendered to HTML on the server, then hydrated in the browser so it can use state, effects and event handlers. A Server Action (`'use server'`) is a function that runs on the server but can be called from the client. Next.js exposes it as a POST endpoint. Reads belong in Server Components, interactivity in Client Components, mutations in Server Actions.

#### Q: Why is protecting routes in proxy.ts (middleware) not enough?

The proxy is a coarse, early check. A matcher can miss routes, layouts are skipped on client navigation, and Server Actions and route handlers can be called directly with any HTTP client. Bugs like CVE-2025-29927 bypassed middleware completely. Every action, route handler and data read must call `auth()` (through a guard) and filter by owner. The proxy improves UX by redirecting early. It does not replace authorization.

#### Q: How do you prevent two concurrent debits from overdrawing an account?

Do the check and the write in one atomic statement: `UPDATE account SET balance = balance - x WHERE id = ? AND balance >= x`, then treat "zero rows updated" as insufficient funds. Postgres locks the row and re-evaluates the condition. Wrap it with the transaction insert in one database transaction. Add a `CHECK (balance >= 0)` constraint as a second wall, and idempotency keys so client retries do not double-charge.

#### Q: When would you use a route handler instead of a Server Action?

Route handlers are for callers other than your own React UI (mobile apps, partners, webhooks), for cacheable or interactive GETs used with TanStack Query, and when you need full control of HTTP (status codes, streaming, headers). Server Actions are for mutations from your own forms: they integrate with `useActionState`, progressive enhancement and `revalidatePath`. Both call the same service.

#### Q: How does Auth.js with the JWT strategy store the session, and what are the trade-offs?

The session is an encrypted JWT (JWE) in an `httpOnly` cookie signed with `AUTH_SECRET`. Reads need no database, so the proxy and every request can verify it cheaply, and it scales horizontally. The downside is revocation: a stolen cookie or a stale role stays valid until it expires. Mitigate with short `maxAge`, re-checking critical facts (role, account status) in the database, or switching to the database strategy when instant revocation matters.

#### Q: What does your API client's response interceptor do, and why only retry GETs?

It normalizes every failure into one `ApiError` shape with code, message and request id; on 401 it refreshes the session once (sharing one refresh across parallel requests) and retries, otherwise redirects to login; it retries once on network errors and 502, 503, 504; and it toasts on 5xx. Only safe methods are retried automatically because a POST that timed out may have succeeded on the server. Retrying it could create a duplicate payment. POSTs use idempotency keys instead.

#### Q: What is the Prisma client hot-reload problem in Next.js?

In development, Next.js re-evaluates server modules on each change. A top-level `new PrismaClient()` creates a new connection pool every time, and Postgres eventually rejects new connections. The fix is to store the client on `globalThis` in development, which survives module reloads. In production, modules load once, so a plain module-level instance is fine.

#### Q: How do useOptimistic and TanStack Query work together in your transactions page?

TanStack Query owns the server data for the table (with `initialData` from the Server Component). `useOptimistic` derives a temporary list from it with the new row prepended while a transition runs. The transition calls the Server Action, then invalidates the query and awaits the refetch. When the transition ends, React drops the optimistic state and shows the real data. On failure, the optimistic row disappears automatically, and the error is shown with a toast.

## Cheatsheet

```text
Rendering
  Server Component (default)      async, can await DB, no hooks, no JS shipped
  'use client'                    hooks, events, browser APIs, still SSR'd
  'use server'                    Server Action, a public POST endpoint
  params / searchParams           Promise in Next 16: const { id } = await params
  cookies() / headers()           async: const h = await headers()

Files in app/
  page.tsx layout.tsx loading.tsx error.tsx('use client') not-found.tsx
  global-error.tsx (renders html/body)  route.ts (GET POST PATCH DELETE)
  (group)/ folders do not change the URL   [id]/ dynamic segment

Data
  read on page         page -> queries.ts (cache) -> service(userId)
  write from UI        <form action> / useActionState -> action -> service -> revalidatePath
  interactive client   useQuery({ queryKey, queryFn: ({signal}) => api.x(signal) })
  optimistic           const [rows, add] = useOptimistic(data, reducer); add() inside startTransition
  cache invalidation   revalidatePath(path) | revalidateTag(tag,'max') | updateTag(tag) (actions only)

Auth.js v5 (next-auth@beta)
  export const { handlers, auth, signIn, signOut } = NextAuth({...})
  app/api/auth/[...nextauth]/route.ts   export const { GET, POST } = handlers
  server: const session = await auth()   client: useSession() inside <SessionProvider>
  jwt({ token, user, account }) runs at sign-in -> add id, role
  session({ session, token }) copies to session.user
  signIn/redirect THROW: rethrow unknown errors in try/catch

proxy.ts (was middleware.ts)
  export default auth((req) => ...)   req.auth?.user
  export const config = { matcher: ['/((?!api/auth|_next/static|_next/image|favicon.ico).*)'] }
  coarse check only. Re-check in every action, route and query.

Prisma 7
  prisma.config.ts (datasource url, seed)   generator provider = "prisma-client", output required
  new PrismaClient({ adapter: new PrismaPg({ connectionString }) })
  globalThis singleton in dev   updateMany({ where: { id, userId } }) for ownership
  $transaction(async (tx) => ...) throws -> rollback   no network calls inside

Money
  integer cents   parse with string maths   format with Intl.NumberFormat at the edge
  atomic: UPDATE ... WHERE balance >= x   CHECK (balance >= 0)   Idempotency-Key

API contract
  success { data, meta? }   error { error: { code, message, details?, requestId } }
  401 UNAUTHENTICATED 403 FORBIDDEN 404 NOT_FOUND 409 CONFLICT 422 INSUFFICIENT_FUNDS 429 RATE_LIMITED

Commands
  npm run db:up | npx prisma migrate dev --name x | npm run db:seed | npx prisma studio
  npm run dev | npm test | npm run e2e | docker build -t ledger-web .
```

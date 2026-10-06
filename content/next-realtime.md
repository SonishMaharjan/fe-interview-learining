---
id: next-realtime
title: Real-time in Next.js with Socket.IO
group: Full-stack Next.js
tagline: Add live balances and transaction feeds to ledger-web with a custom Node server running Next.js and Socket.IO, typed events, and an SSE fallback.
covers: Next.js 16 (custom server), React 19.2, Socket.IO 4, socket.io-client, Auth.js v5 JWT cookies, TanStack Query 5, Server-Sent Events, Redis adapter and emitter, Docker
status: current
kind: guide
---

## 1. Why WebSockets are tricky in Next.js

This guide continues **ledger-web** from "Build a Full-fledged Next.js App Step by Step". You start from its end state: Auth.js with JWT sessions, Prisma, `createTransaction` in `src/features/transactions/service.ts`, and the `TransactionsPanel` with TanStack Query. The goal: when a transaction is recorded in one tab (or by the API, or by another device), every open page for that account updates its balance and table within a second.

> **Outdated:** Version details move fast. This guide targets Next.js 16 and Socket.IO 4. Custom server behavior and hosting limits differ between versions and platforms. Where something is version-specific it is marked, so check your versions before copying a config value.

### [Beginner] Step 1 — Request/response vs a long-lived connection

Everything in ledger-web so far is **request/response**: the browser asks, the server answers, the connection is done. Pages, Server Actions and route handlers all fit that model, which is why they deploy well as **serverless functions**: a function starts, handles one request, and may be frozen or destroyed right after.

A WebSocket is the opposite. The browser opens one TCP connection and keeps it open for minutes or hours, and **the server** decides when to send. That needs:

1. a **process that stays alive** as long as the connection,
2. a place to keep **state**: which sockets are open, which rooms they joined,
3. a way for code that handles a mutation to **find** those sockets and write to them.

Serverless functions provide none of these. On Vercel, functions cannot act as WebSocket servers, and every function has a maximum duration (minutes, not hours, depending on plan). Edge runtimes are similar. Even if a connection could stay open, the next request that records a transaction may run in a completely different instance that has no idea the socket exists.

```mermaid
flowchart LR
  A["Tab 1 records a debit"] --> B["Function instance A<br/>runs the Server Action"]
  C["Tab 2 wants the update"] --> D["Function instance B<br/>or nothing at all"]
  B -.->|"no shared memory,<br/>no open socket"| D
```

> **Why:** Next.js itself is fine with WebSockets. The problem is **where you host it**. Run Next.js as a normal long-lived Node.js process (a container, a VM) and WebSockets work. Run it on serverless and you need the socket part to live somewhere else.

### [Beginner] Step 2 — The options, compared

| Option | How it works | Pros | Cons | Fits when |
|---|---|---|---|---|
| **A. Custom Node server** | `server.ts` starts Next.js and Socket.IO on the same HTTP server | one deployable, same auth cookie, same origin, full control | no Vercel, no `output: 'standalone'`, you own scaling (Redis adapter, sticky sessions) | you deploy containers anyway |
| **B. Separate realtime service** | a dedicated Socket.IO or NestJS gateway service, for example `ledger-api`'s `LedgerGateway` | Next.js can stay on Vercel, realtime scales on its own | two services, cross-origin auth (cookie domain or a short-lived token), shared event contract | you already have a backend team or service |
| **C. Managed service** | Pusher, Ably, Supabase Realtime, Liveblocks; your server calls their REST API to publish | no infrastructure, global edge, works from serverless | cost per message or connection, vendor lock-in, data leaves your network | small team, need it working this week |
| **D. Server-Sent Events** | a route handler returns a `text/event-stream` that stays open | plain HTTP, auto-reconnect built into `EventSource`, no library | one-way only (server to browser), function duration limits on serverless, needs a shared bus across instances | live feeds and notifications |

```mermaid
flowchart TD
  A["Need live updates"] --> B{"Must the client<br/>send messages too?"}
  B -->|"no, server push only"| C{"Hosting on serverless?"}
  C -->|"no"| D["SSE route handler<br/>Option D"]
  C -->|"yes"| E["Managed service or<br/>SSE with short reconnects"]
  B -->|"yes, two-way"| F{"Can you run a<br/>long-lived Node process?"}
  F -->|"yes, one app"| G["Custom server + Socket.IO<br/>Option A"]
  F -->|"yes, separate team or service"| H["Separate realtime service<br/>Option B"]
  F -->|"no"| I["Managed service<br/>Option C"]
```

> **Interview tip:** Do not answer "use WebSockets" by reflex. Most "real-time" finance UIs only need **server push** (balances, statuses, notifications). SSE or a managed service is often simpler. Reach for two-way sockets when clients send frequent messages: chat, collaborative editing, trading tickets, presence.

This guide builds **Option A** properly, because it teaches every moving part (connection lifecycle, auth, rooms, emitting from mutations, scaling). Part 5 shows **Option D** as the lightweight alternative. Options B and C reuse the same client code: only the place that emits changes.

Socket.IO in two sentences: it is a library on top of WebSockets that adds **reconnection**, **rooms** (named groups of sockets you can broadcast to), **acknowledgements** (request/response over the socket), and an HTTP long-polling fallback. Its client and server must both be Socket.IO. It is not a raw WebSocket protocol.

## 2. Option A: a custom server with Socket.IO

### [Beginner] Step 3 — Install the packages

```bash
npm install socket.io socket.io-client cookie pino @next/env
npm install -D esbuild
```

`cookie` parses the `Cookie` header during the socket handshake. `@next/env` loads `.env` files the same way `next dev` does (it already ships with Next.js, installing it makes the dependency explicit). `esbuild` bundles `server.ts` for production.

Target layout for this part:

```text
ledger-web/
├─ server.ts                         Next.js + Socket.IO entry point
├─ scripts/socket-smoke.ts           manual socket test client
└─ src/
   ├─ features/realtime/
   │  ├─ events.ts                   typed event contract, shared by server and client
   │  ├─ client-socket.ts            browser socket singleton
   │  ├─ socket-provider.tsx         connection status context
   │  ├─ use-account-channel.ts      subscribe hook
   │  └─ components/
   ├─ lib/realtime/
   │  ├─ io.ts                       globalThis io holder
   │  └─ publish.ts                  emit helpers used by services
   └─ server/
      ├─ load-env.ts                 loads .env before anything reads process.env
      └─ realtime/
         ├─ log.ts
         ├─ socket-auth.ts           handshake authentication
         └─ handlers.ts              rooms and client events
```

### [Beginner] Step 4 — Define the event contract first

Socket.IO accepts four generic types: events the client sends, events the server sends, events between server instances, and per-socket data. Write them once and import them on both sides. A typo in an event name then becomes a compile error instead of a silent bug.

```ts
// src/features/realtime/events.ts
import type { TransactionDTO } from '@/features/transactions/dto';
import type { Role } from '@/features/auth/types';

export type TransactionCreatedPayload = {
  accountId: string;
  transaction: TransactionDTO;
  /** The idempotency key of the request that created it. Lets the author's tab skip its own toast. */
  clientMutationId?: string;
};

export type BalanceUpdatedPayload = {
  accountId: string;
  balanceCents: number;
  currency: string;
  /** ISO time of the change. Clients ignore events older than what they already show. */
  at: string;
};

export type SubscribeAck = { ok: true } | { ok: false; error: 'NOT_FOUND' | 'SESSION_EXPIRED' | 'BAD_REQUEST' | 'INTERNAL' };

export interface ServerToClientEvents {
  'transaction:created': (payload: TransactionCreatedPayload) => void;
  'account:balance': (payload: BalanceUpdatedPayload) => void;
}

export interface ClientToServerEvents {
  'account:subscribe': (accountId: string, ack: (res: SubscribeAck) => void) => void;
  'account:unsubscribe': (accountId: string) => void;
}

export type InterServerEvents = Record<string, never>;

export type SocketData = {
  userId: string;
  role: Role;
  expiresAt: number; // ms epoch, from the session JWT
};

export const rooms = {
  user: (userId: string) => `user:${userId}`,
  account: (accountId: string) => `account:${accountId}`,
};
```

> **Why rooms per account and per user?** A room is a cheap broadcast group. `account:<id>` reaches every tab currently viewing that account. `user:<id>` reaches every tab of that user, wherever they are in the app, which is ideal for an "accounts list" or a notification badge.

> **Finance tip:** Events carry what changed, not secrets. The payload goes only to sockets that passed an ownership check, but treat it like an API response anyway: no internal ids of other users, no full card numbers.

### [Intermediate] Step 5 — A process-wide io holder on globalThis

Here is the subtle part. `server.ts` is compiled by `tsx` (dev) or `esbuild` (prod). Your Server Actions and services are compiled by **Next.js's bundler** into separate chunks. The same source file `src/lib/realtime/io.ts` ends up as **two module instances** in one process, so a module-level `let io` set by `server.ts` would be `undefined` inside a Server Action.

Both copies share one thing: the process's `globalThis`. Store the server there.

```ts
// src/lib/realtime/io.ts
import type { Server, Socket } from 'socket.io';
import type { ClientToServerEvents, InterServerEvents, ServerToClientEvents, SocketData } from '@/features/realtime/events';

export type LedgerServer = Server<ClientToServerEvents, ServerToClientEvents, InterServerEvents, SocketData>;
export type LedgerSocket = Socket<ClientToServerEvents, ServerToClientEvents, InterServerEvents, SocketData>;

const g = globalThis as unknown as { __ledgerIO?: LedgerServer };

export function setIO(io: LedgerServer): void {
  g.__ledgerIO = io;
}

/** Undefined when running under plain `next dev`, on Vercel, or in tests. Callers must cope. */
export function getIO(): LedgerServer | undefined {
  return g.__ledgerIO;
}
```

Only **types** are imported from `socket.io`, so this file adds nothing to any bundle.

```mermaid
flowchart TD
  A["Node.js process"] --> B["server.ts bundle<br/>creates io, calls setIO"]
  A --> C["Next.js server bundle<br/>Server Actions, route handlers"]
  B --> D["globalThis.__ledgerIO"]
  C -->|"getIO"| D
  D --> E["Socket.IO server<br/>rooms and sockets"]
```

> **Gotcha:** This works because Next.js renders in the same process as your custom server. If `getIO()` is always `undefined` inside actions on your Next.js version (some versions isolated rendering in worker processes), switch to the Redis emitter in Step 12. It works across processes and machines.

### [Intermediate] Step 6 — Authenticate the handshake from the Auth.js cookie

The browser sends cookies with the WebSocket upgrade request because it is same-origin. So the socket can reuse the exact session the pages use. Auth.js stores it as an encrypted JWT in `authjs.session-token` (or `__Secure-authjs.session-token` on HTTPS). `decode` from `next-auth/jwt` decrypts it with `AUTH_SECRET`. The cookie name is also the **salt**.

Plain Node code cannot import `src/lib/logger.ts` (it has `server-only`). Give the realtime server its own small logger:

```ts
// src/server/realtime/log.ts
import pino from 'pino';

export const rtLog = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  base: { service: 'ledger-web', component: 'realtime' },
});
```

```ts
// src/server/realtime/socket-auth.ts
import { parse } from 'cookie';
import { decode } from 'next-auth/jwt';
import { env } from '@/env';
import type { LedgerSocket } from '@/lib/realtime/io';
import { rtLog } from './log';

const COOKIE_NAMES = ['__Secure-authjs.session-token', 'authjs.session-token'] as const;

export async function authenticateSocket(socket: LedgerSocket, next: (err?: Error) => void): Promise<void> {
  try {
    const cookies = parse(socket.handshake.headers.cookie ?? '');
    const name = COOKIE_NAMES.find((n) => cookies[n]);
    const raw = name ? cookies[name] : undefined;
    if (!name || !raw) return next(new Error('UNAUTHENTICATED'));

    const token = await decode({ token: raw, secret: env.AUTH_SECRET, salt: name });
    const expiresAt = typeof token?.exp === 'number' ? token.exp * 1000 : 0;
    if (!token?.id || !token.role || expiresAt < Date.now()) return next(new Error('UNAUTHENTICATED'));

    socket.data.userId = token.id;
    socket.data.role = token.role;
    socket.data.expiresAt = expiresAt;
    next();
  } catch (err) {
    rtLog.warn({ err }, 'socket auth failed');
    next(new Error('UNAUTHENTICATED'));
  }
}
```

```mermaid
sequenceDiagram
  participant B as Browser
  participant S as Socket.IO server
  participant A as authenticateSocket
  B->>S: GET /socket.io upgrade with Cookie authjs.session-token
  S->>A: run io.use middleware
  A->>A: parse cookie, decode JWE with AUTH_SECRET
  alt valid and not expired
    A-->>S: next, socket.data has userId role expiresAt
    S-->>B: connect event
  else missing or invalid
    A-->>S: next with Error UNAUTHENTICATED
    S-->>B: connect_error UNAUTHENTICATED
  end
```

> **Gotcha:** When a session cookie grows past about 4 KB, Auth.js splits it into `authjs.session-token.0`, `.1` and so on. Our token is tiny, so this does not happen. If you put large claims in the JWT, join the chunks in order before calling `decode`.

> **Gotcha:** A cross-origin realtime service (Option B) does not receive this cookie unless both share a parent domain and the cookie's `Domain` allows it. The common fix is a short-lived signed token: a route handler mints it for the signed-in user, and the client passes it in `io(url, { auth: { token } })`.

> **Why `token.exp`?** A WebSocket can stay open for hours, longer than the 8-hour session. You record when the session ends and disconnect the socket at that moment (next step). Otherwise a signed-out or expired user keeps receiving balances.

### [Intermediate] Step 7 — Rooms and client events

```ts
// src/server/realtime/handlers.ts
import { prisma } from '@/lib/db/client';
import { rooms } from '@/features/realtime/events';
import type { LedgerServer, LedgerSocket } from '@/lib/realtime/io';
import { rtLog } from './log';

const ID_RE = /^[a-z0-9]{10,40}$/i;

export function registerHandlers(io: LedgerServer, socket: LedgerSocket): void {
  const { userId, expiresAt } = socket.data;
  const log = rtLog.child({ socketId: socket.id, userId });
  log.debug({ total: io.engine.clientsCount }, 'socket connected');

  void socket.join(rooms.user(userId));

  // Cut the connection when the session expires.
  const expiry = setTimeout(() => socket.disconnect(true), Math.max(0, expiresAt - Date.now()));

  socket.on('account:subscribe', async (accountId, ack) => {
    if (typeof ack !== 'function') return;
    if (typeof accountId !== 'string' || !ID_RE.test(accountId)) return ack({ ok: false, error: 'BAD_REQUEST' });
    if (Date.now() > socket.data.expiresAt) {
      ack({ ok: false, error: 'SESSION_EXPIRED' });
      socket.disconnect(true);
      return;
    }

    try {
      // Ownership check: the same rule as the HTTP layer.
      const owned = await prisma.account.findFirst({
        where: { id: accountId, userId, archivedAt: null },
        select: { id: true },
      });
      if (!owned) return ack({ ok: false, error: 'NOT_FOUND' });

      await socket.join(rooms.account(accountId));
      log.debug({ accountId }, 'joined account room');
      ack({ ok: true });
    } catch (err) {
      // An async listener that throws becomes an unhandled rejection. Always catch.
      log.error({ err, accountId }, 'subscribe failed');
      ack({ ok: false, error: 'INTERNAL' });
    }
  });

  socket.on('account:unsubscribe', (accountId) => {
    if (typeof accountId === 'string') void socket.leave(rooms.account(accountId));
  });

  socket.on('disconnect', (reason) => {
    clearTimeout(expiry);
    log.debug({ reason }, 'socket disconnected');
  });
}
```

> **Why validate `typeof accountId` when TypeScript already typed it?** Types exist only at compile time. A malicious client can emit anything. Every handler validates its input at runtime, exactly like a route handler.

> **Gotcha:** Never let a client join an arbitrary room name (`socket.on('join', (room) => socket.join(room))`). That is a classic data leak: anyone could join `account:<someone-else's-id>`. Rooms are joined only after an ownership check on the server.

### [Intermediate] Step 8 — server.ts: Next.js and Socket.IO on one HTTP server

`next dev` loads `.env` for you, but `server.ts` imports `src/env.ts` (through the socket auth and Prisma) **before** Next.js starts. ES modules evaluate imports in order, so a first, side-effect-only import that loads the env files fixes it:

```ts
// src/server/load-env.ts
import { loadEnvConfig } from '@next/env';

loadEnvConfig(process.cwd(), process.env.NODE_ENV !== 'production');
```

```ts
// server.ts
import './src/server/load-env'; // must stay the first import
import { createServer } from 'node:http';
import next from 'next';
import { Server } from 'socket.io';
import { setIO, type LedgerServer } from '@/lib/realtime/io';
import { authenticateSocket } from '@/server/realtime/socket-auth';
import { registerHandlers } from '@/server/realtime/handlers';
import { rtLog } from '@/server/realtime/log';

const dev = process.env.NODE_ENV !== 'production';
const port = Number(process.env.PORT ?? 3000);
const hostname = process.env.HOST ?? '0.0.0.0';

const app = next({ dev, hostname, port });
const handle = app.getRequestHandler();
await app.prepare();

const httpServer = createServer((req, res) => {
  void handle(req, res);
});

const io: LedgerServer = new Server(httpServer, {
  path: '/socket.io',
  serveClient: false,
  // Let Next.js handle its own upgrade requests (dev hot reload) instead of destroying them.
  destroyUpgrade: false,
  pingInterval: 25_000,
  pingTimeout: 20_000,
  connectionStateRecovery: { maxDisconnectionDuration: 2 * 60 * 1000, skipMiddlewares: false },
});

io.use((socket, nextFn) => {
  void authenticateSocket(socket, nextFn);
});
io.on('connection', (socket) => registerHandlers(io, socket));
setIO(io);

httpServer.listen(port, hostname, () => {
  rtLog.info({ port, dev }, `ledger-web ready on http://localhost:${port}`);
});

function shutdown(signal: string) {
  rtLog.info({ signal }, 'shutting down');
  io.close(); // closes sockets and the HTTP server
  setTimeout(() => process.exit(0), 5_000).unref();
}
process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

> **Why `destroyUpgrade: false`?** By default Socket.IO destroys any WebSocket upgrade request that is not for its path after one second. In development, Next.js opens its own WebSocket for hot module reloading on the same server. With the default, hot reload breaks with a confusing "WebSocket connection failed" in the console.

> **Why `skipMiddlewares: false`?** With connection state recovery, a socket that drops briefly gets its rooms and missed events back. Keeping middlewares on means the cookie is re-checked even on recovery.

> **Gotcha:** `HOSTNAME` is set by Docker to the container id. That is why this file reads `HOST`.

Update the scripts in `package.json`. `dev` now runs your server instead of `next dev`:

```json
{
  "scripts": {
    "dev": "tsx server.ts",
    "dev:next": "next dev",
    "build": "prisma generate && next build && npm run build:server",
    "build:server": "esbuild server.ts --bundle --platform=node --format=esm --target=node22 --packages=external --outfile=dist/server.js",
    "start": "NODE_ENV=production node dist/server.js"
  }
}
```

`--packages=external` keeps `node_modules` imports as runtime imports, while your own `@/` files (resolved through `tsconfig.json` paths) are bundled in. Add `dist` to `.gitignore`.

> **Gotcha:** A custom server cannot use `output: 'standalone'`: the standalone build writes its own minimal `server.js` and does not trace your `server.ts`. Remove `output: 'standalone'` from `next.config.ts` and use the Dockerfile in Part 6. Also, `tsx server.ts` does not restart when you edit `server.ts` itself. Next.js still hot-reloads pages and actions.

> **Gotcha:** The programmatic `next({ dev })` API may not pick the same bundler defaults as the `next dev` CLI on every version. If you see a webpack banner instead of Turbopack in development and care about the difference, check the custom server docs for your Next.js version.

**Check it works:** start it.

```bash
npm run dev
```

```text
{"level":30,"component":"realtime","port":3000,"dev":true,"msg":"ledger-web ready on http://localhost:3000"}
```

The app works as before. Now test the socket from a small Node client. Node's client can send a `Cookie` header, which a browser script cannot set by hand:

```ts
// scripts/socket-smoke.ts
import { io } from 'socket.io-client';
import type { ClientSocket } from '../src/features/realtime/client-socket';

const accountId = process.argv[2] ?? 'not-a-real-id-123';
const s: ClientSocket = io('http://localhost:3000', {
  transports: ['websocket'],
  extraHeaders: { cookie: process.env.C ?? '' }, // Node only: browsers send cookies themselves
});

s.on('connect', () => {
  console.log('connected', s.id);
  s.emit('account:subscribe', accountId, (res) => console.log('subscribe', res));
});
s.on('connect_error', (err) => {
  console.log('connect_error', err.message);
  process.exit(1);
});
s.on('account:balance', (p) => console.log('balance', p));
s.on('transaction:created', (p) => console.log('transaction', p.transaction.description));
```

(`ClientSocket` is defined in Part 4, Step 13. Create that file now or come back to this check after it.)

Sign in as Alice in the browser and copy the `authjs.session-token` cookie value from DevTools, as in the first guide:

```bash
export C="authjs.session-token=PASTE_VALUE"
npx tsx scripts/socket-smoke.ts
```

```text
connected 3fJk8sP0...
subscribe { ok: false, error: 'NOT_FOUND' }
```

Press Ctrl+C. Without the cookie it is rejected in the handshake:

```bash
C= npx tsx scripts/socket-smoke.ts
```

```text
connect_error UNAUTHENTICATED
```

> **Gotcha:** The CSP from the first guide has `connect-src 'self'`. Modern browsers treat same-origin `ws:` and `wss:` as matching `'self'`, so the bundled client in Part 4 just works. If you must support very old browsers, add `wss://your-domain` explicitly.

## 3. Emit after mutations

### [Intermediate] Step 9 — Where to emit, and when

Three rules:

1. **Emit after the database commit**, never inside `prisma.$transaction(...)`. If you emit inside and the transaction rolls back, clients show money that never moved.
2. **Emit from the service layer**, not from the Server Action. The JSON API (`POST /api/v1/.../transactions`), the Server Action and any future job all call the same `createTransaction`. Emitting there means every path notifies clients.
3. **Emitting is best effort.** A failed emit must not fail the payment. Clients also refetch on reconnect, so a missed event heals itself.

Write a small `publish` module so the service never touches Socket.IO directly. When `getIO()` is undefined (plain `next dev`, Vercel, tests), it logs and does nothing.

```ts
// src/lib/realtime/publish.ts
import 'server-only';
import { logger } from '@/lib/logger';
import { rooms, type BalanceUpdatedPayload, type TransactionCreatedPayload } from '@/features/realtime/events';
import { getIO } from './io';

export const publish = {
  transactionCreated(userId: string, payload: TransactionCreatedPayload, balance: BalanceUpdatedPayload): void {
    const io = getIO();
    if (!io) {
      logger.debug({ accountId: payload.accountId }, 'realtime not available, skipping emit');
      return;
    }
    try {
      io.to(rooms.account(payload.accountId)).emit('transaction:created', payload);
      // Balance goes to the account room and to every tab of the owner (accounts list, header).
      io.to(rooms.account(balance.accountId)).to(rooms.user(userId)).emit('account:balance', balance);
    } catch (err) {
      logger.error({ err }, 'realtime emit failed');
    }
  },
};
```

> **Why `.to(a).to(b)`?** Chaining rooms sends to the **union**, and each socket receives the event once even if it is in both rooms.

### [Intermediate] Step 10 — Emit from createTransaction

Change two things in `src/features/transactions/service.ts`: select the account's `currency` inside the transaction, and publish after it commits. Replace the `try { ... }` block of `createTransaction` with this version (the idempotency check above it is unchanged):

```ts
// src/features/transactions/service.ts (inside createTransaction)
  try {
    const { row, currency } = await prisma.$transaction(async (tx) => {
      const owned = await tx.account.findFirst({
        where: { id: accountId, userId, archivedAt: null },
        select: { id: true, currency: true },
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

      const { balanceCents } = await tx.account.findUniqueOrThrow({
        where: { id: accountId },
        select: { balanceCents: true },
      });

      const created = await tx.transaction.create({
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
      return { row: created, currency: owned.currency };
    });

    // Committed. Now tell the world.
    const transaction = toTransactionDTO(row);
    publish.transactionCreated(
      userId,
      { accountId, transaction, clientMutationId: idempotencyKey },
      { accountId, balanceCents: row.balanceAfterCents, currency, at: transaction.createdAt },
    );
    return { transaction, replayed: false };
  } catch (err) {
    if (idempotencyKey && isUniqueViolation(err)) {
      return createTransaction(userId, accountId, input, idempotencyKey);
    }
    throw err;
  }
```

Add the import at the top: `import { publish } from '@/lib/realtime/publish';`. A replayed request (`replayed: true`) returns early and emits nothing, which is correct: nothing changed.

The unit tests from the first guide still pass because `getIO()` returns `undefined` there. Add one assertion if you like: mock `@/lib/realtime/publish` and expect `publish.transactionCreated` to be called once on success and never on `InsufficientFundsError`.

```mermaid
sequenceDiagram
  participant T1 as Tab 1
  participant SA as Server Action
  participant S as createTransaction
  participant DB as Postgres
  participant IO as Socket.IO
  participant T2 as Tab 2 same account
  T1->>SA: submit debit 12.50
  SA->>S: createTransaction
  S->>DB: BEGIN, conditional UPDATE, INSERT, COMMIT
  DB-->>S: committed row
  S->>IO: emit transaction created to account room
  S->>IO: emit balance to account and user rooms
  IO-->>T2: transaction created
  IO-->>T2: account balance
  IO-->>T1: same events, toast skipped by clientMutationId
  S-->>SA: transaction DTO
  SA-->>T1: success, revalidatePath response
```

**Check it works:** copy the id of the "Everyday" account from its URL and run the smoke client with it:

```bash
npx tsx scripts/socket-smoke.ts PASTE_ACCOUNT_ID
```

Add a transaction in the browser. The terminal prints:

```text
connected 9xQ2...
subscribe { ok: true }
transaction Snack
balance { accountId: 'cm...', balanceCents: 126651, currency: 'USD', at: '2026-10-05T10:12:03.120Z' }
```

Record one through the JSON API with `curl` (Part 5 of the first guide) and the same lines appear: every entry point emits, because the service does.

### [Advanced] Step 11 — Why not emit from the Server Action, or from a database trigger?

| Emit from | Pros | Cons |
|---|---|---|
| Server Action | simple | the JSON API and jobs do not emit, so clients miss updates |
| **Service, after commit** (chosen) | every entry point emits, clear ordering | the service knows a tiny `publish` interface |
| Prisma extension or middleware hook | automatic | hard to know whether you are inside a transaction, emits before commit by accident |
| Postgres `LISTEN/NOTIFY` or change data capture | catches writes from any system, even manual SQL | more infrastructure, payload limits, harder to type |
| Transactional outbox table + worker | guaranteed delivery, survives crashes between commit and emit | most moving parts |

> **Finance tip:** For balances shown to customers, best effort plus "refetch on reconnect" is fine because the source of truth is always the database. For events that **trigger other systems** (send an email, post to a partner), use a transactional outbox: insert an `OutboxEvent` row in the same database transaction and let a worker publish it.

### [Advanced] Step 12 — The Redis emitter: emit from anywhere

`getIO()` only works in the process that owns the Socket.IO server. The **Redis emitter** lets any process publish to sockets held by any server, through Redis pub/sub. You need it when:

- Next.js runs on Vercel and Socket.IO runs in a container (Option B),
- a background worker records transactions,
- your Next.js version runs actions outside the custom server's process.

The Socket.IO servers must use the Redis **adapter** (Part 6, Step 20). Then:

```bash
npm install redis @socket.io/redis-emitter
```

```ts
// src/lib/realtime/redis-emitter.ts
import 'server-only';
import { createClient } from 'redis';
import { Emitter } from '@socket.io/redis-emitter';
import type { ServerToClientEvents } from '@/features/realtime/events';

const g = globalThis as unknown as { __ledgerEmitter?: Promise<Emitter<ServerToClientEvents>> };

export function getEmitter(): Promise<Emitter<ServerToClientEvents>> | undefined {
  const url = process.env.REDIS_URL;
  if (!url) return undefined;
  g.__ledgerEmitter ??= (async () => {
    const client = createClient({ url });
    client.on('error', () => {}); // logged by the caller. Avoid crashing the process.
    await client.connect();
    return new Emitter<ServerToClientEvents>(client);
  })();
  return g.__ledgerEmitter;
}
```

In `publish.ts`, fall back to the emitter when there is no local `io`:

```ts
// src/lib/realtime/publish.ts (inside transactionCreated, replacing the `if (!io)` block)
if (!io) {
  const emitterPromise = getEmitter();
  if (!emitterPromise) return;
  void emitterPromise
    .then((emitter) => {
      emitter.to(rooms.account(payload.accountId)).emit('transaction:created', payload);
      emitter.to(rooms.account(balance.accountId)).to(rooms.user(userId)).emit('account:balance', balance);
    })
    .catch((err: unknown) => logger.error({ err }, 'redis emit failed'));
  return;
}
```

Add `import { getEmitter } from './redis-emitter';` at the top.

> **Gotcha:** The emitter and the adapter must agree on the Redis channel prefix (default `socket.io`) and on the namespace (`/` here). If events never arrive, check those two first.

## 4. React client

### [Beginner] Step 13 — One socket per tab

The browser should hold **one** connection per tab, shared by every component. Create it lazily in a module so it is never created during server rendering, and with `autoConnect: false` so a provider controls when it connects.

```ts
// src/features/realtime/client-socket.ts
import { io, type Socket } from 'socket.io-client';
import type { ClientToServerEvents, ServerToClientEvents } from './events';

export type ClientSocket = Socket<ServerToClientEvents, ClientToServerEvents>;

let socket: ClientSocket | null = null;

export function getSocket(): ClientSocket {
  socket ??= io({
    path: '/socket.io',
    autoConnect: false,
    transports: ['websocket'], // skip long-polling, see Part 6
    reconnectionDelay: 1_000,
    reconnectionDelayMax: 10_000,
  });
  return socket;
}
```

Note the client generic order: `Socket<ListenEvents, EmitEvents>`, so server-to-client comes **first** on the client.

### [Intermediate] Step 14 — SocketProvider with connection status

```tsx
// src/features/realtime/socket-provider.tsx
'use client';

import { createContext, useContext, useEffect, useState, type ReactNode } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import { getSocket } from './client-socket';

export type ConnectionStatus = 'connecting' | 'connected' | 'reconnecting' | 'offline' | 'unauthorized';

const StatusContext = createContext<ConnectionStatus>('connecting');
export const useConnectionStatus = () => useContext(StatusContext);

export function SocketProvider({ children }: { children: ReactNode }) {
  const [status, setStatus] = useState<ConnectionStatus>('connecting');
  const queryClient = useQueryClient();

  useEffect(() => {
    const socket = getSocket();
    let wasConnected = false;

    const onConnect = () => {
      setStatus('connected');
      // After a real gap (not a recovered session), refetch what we may have missed.
      if (wasConnected && !socket.recovered) void queryClient.invalidateQueries({ queryKey: ['transactions'] });
      wasConnected = true;
    };
    const onDisconnect = (reason: string) => {
      // "io server disconnect" means the server kicked us (expired session). No auto-reconnect.
      setStatus(reason === 'io server disconnect' ? 'unauthorized' : 'reconnecting');
    };
    const onConnectError = (err: Error) => {
      setStatus(err.message === 'UNAUTHENTICATED' ? 'unauthorized' : 'reconnecting');
    };
    const onOffline = () => setStatus('offline');

    socket.on('connect', onConnect);
    socket.on('disconnect', onDisconnect);
    socket.on('connect_error', onConnectError);
    window.addEventListener('offline', onOffline);
    socket.connect();

    return () => {
      socket.off('connect', onConnect);
      socket.off('disconnect', onDisconnect);
      socket.off('connect_error', onConnectError);
      window.removeEventListener('offline', onOffline);
      socket.disconnect();
    };
  }, [queryClient]);

  return <StatusContext.Provider value={status}>{children}</StatusContext.Provider>;
}
```

> **Why connect in an effect, not at module load?** Effects run only in the browser and have a cleanup. On sign-out the `(app)` layout unmounts, the cleanup disconnects, and the next user in the same tab gets a fresh handshake with their own cookie.

> **Gotcha:** In development, React StrictMode runs every effect twice: mount, cleanup, mount. You will see the socket connect, disconnect and connect again, and the server logs two connections. That is expected and proves your cleanup works. It does not happen in production.

> **Gotcha:** When the server rejects the handshake in middleware (`connect_error` with `UNAUTHENTICATED`), Socket.IO keeps retrying with backoff. When the server calls `socket.disconnect(true)`, the client does **not** reconnect automatically. The `unauthorized` status lets the UI tell the user to sign in again.

### [Intermediate] Step 15 — useAccountChannel: subscribe on mount, clean up on unmount

```ts
// src/features/realtime/use-account-channel.ts
'use client';

import { useEffect, useEffectEvent } from 'react';
import { getSocket } from './client-socket';
import type { BalanceUpdatedPayload, TransactionCreatedPayload } from './events';

type Handlers = {
  onTransaction?: (p: TransactionCreatedPayload) => void;
  onBalance?: (p: BalanceUpdatedPayload) => void;
};

export function useAccountChannel(accountId: string, handlers: Handlers): void {
  // Effect Events always see the latest handlers without re-running the effect (React 19.2).
  const onTransaction = useEffectEvent((p: TransactionCreatedPayload) => handlers.onTransaction?.(p));
  const onBalance = useEffectEvent((p: BalanceUpdatedPayload) => handlers.onBalance?.(p));

  useEffect(() => {
    const socket = getSocket();

    const subscribe = () => {
      socket.emit('account:subscribe', accountId, (res) => {
        if (!res.ok) console.warn('account subscribe failed', accountId, res.error);
      });
    };
    const handleTransaction = (p: TransactionCreatedPayload) => {
      if (p.accountId === accountId) onTransaction(p);
    };
    const handleBalance = (p: BalanceUpdatedPayload) => {
      if (p.accountId === accountId) onBalance(p);
    };

    if (socket.connected) subscribe();
    socket.on('connect', subscribe); // rooms are per connection: re-join after every reconnect
    socket.on('transaction:created', handleTransaction);
    socket.on('account:balance', handleBalance);

    return () => {
      socket.off('connect', subscribe);
      socket.off('transaction:created', handleTransaction);
      socket.off('account:balance', handleBalance);
      if (socket.connected) socket.emit('account:unsubscribe', accountId);
    };
  }, [accountId]);
}
```

> **Why `useEffectEvent`?** Without it, passing a new `handlers` object every render would either re-run the effect (unsubscribe and resubscribe on every render) or capture stale closures. Effect Events are stable and read the latest props. On React versions before 19.2, store the handlers in a `useRef` updated in an effect instead.

> **Gotcha:** Always remove listeners with the **same function reference** you added. `socket.off('account:balance')` without a function removes every listener for that event, including other components' ones.

```mermaid
stateDiagram-v2
  [*] --> Connecting: provider mounts, socket.connect
  Connecting --> Connected: connect, hooks emit subscribe
  Connecting --> Unauthorized: connect_error UNAUTHENTICATED
  Connected --> Reconnecting: network drop or server restart
  Reconnecting --> Connected: connect, re-subscribe, refetch if not recovered
  Connected --> Unauthorized: server disconnect at session expiry
  Unauthorized --> [*]: user signs in again
```

### [Intermediate] Step 16 — Live balance and a live table

The balance is server-rendered. You have two ways to update it when an event arrives:

| Approach | Cost | When |
|---|---|---|
| `router.refresh()` | re-renders the whole route on the server: database queries, RSC payload | rare events, or many server-rendered parts change at once |
| Local state from the event payload (chosen) | zero requests | the payload already contains the new value |

```tsx
// src/features/realtime/components/live-balance.tsx
'use client';

import { useState } from 'react';
import { formatCents } from '@/lib/money';
import { useAccountChannel } from '../use-account-channel';

type Props = { accountId: string; currency: string; initialCents: number; initialAt: string };

export function LiveBalance({ accountId, currency, initialCents, initialAt }: Props) {
  const [state, setState] = useState({ cents: initialCents, at: initialAt });
  const [fromServer, setFromServer] = useState(initialAt);
  const [flash, setFlash] = useState(false);

  // When the server re-renders the page (revalidatePath), adopt its value.
  if (initialAt !== fromServer) {
    setFromServer(initialAt);
    if (initialAt > state.at) setState({ cents: initialCents, at: initialAt });
  }

  useAccountChannel(accountId, {
    onBalance: (p) => {
      // Events can arrive out of order. Never go back in time.
      if (p.at <= state.at) return;
      setState({ cents: p.balanceCents, at: p.at });
      setFlash(true);
      setTimeout(() => setFlash(false), 800);
    },
  });

  return (
    <p
      data-testid="balance"
      aria-live="polite"
      className={`text-3xl tabular-nums transition-colors ${flash ? 'text-blue-600' : ''}`}
    >
      {formatCents(state.cents, currency)}
    </p>
  );
}
```

> **Finance tip:** Comparing ISO timestamps as strings works because they are fixed-width UTC. Two changes in the same millisecond are rare but possible. A real ledger sends a monotonically increasing **version** per account (for example a `version` column incremented in the same `UPDATE`) and clients apply only higher versions.

The server page needs a timestamp for the initial value. Use the account's `updatedAt`. Add it to the DTO in `src/features/accounts/dto.ts`:

```ts
// src/features/accounts/dto.ts (add the field to the type and the mapper)
export type AccountDTO = {
  id: string;
  name: string;
  type: 'CHECKING' | 'SAVINGS';
  currency: string;
  balanceCents: number;
  createdAt: string;
  updatedAt: string;
};
// in toAccountDTO:
//   updatedAt: a.updatedAt.toISOString(),
```

In `src/app/(app)/accounts/[id]/page.tsx`, replace the `<p data-testid="balance">...</p>` element with:

```tsx
<LiveBalance accountId={id} currency={account.currency} initialCents={account.balanceCents} initialAt={account.updatedAt} />
```

and import it: `import { LiveBalance } from '@/features/realtime/components/live-balance';`. The Playwright test from the first guide still finds `data-testid="balance"`.

Now the table. In `TransactionsPanel`, write incoming transactions into the TanStack Query cache. Add this code after the `useQuery` call, and the imports shown:

```tsx
// src/features/transactions/components/transactions-panel.tsx (additions)
// add useRef to the existing import from 'react'
import { useAccountChannel } from '@/features/realtime/use-account-channel';
import type { TransactionCreatedPayload } from '@/features/realtime/events';
// ...

// Remember keys of our own submissions, so this tab does not toast its own transaction.
const myMutationIds = useRef(new Set<string>());

useAccountChannel(accountId, {
  onTransaction: ({ transaction, clientMutationId }: TransactionCreatedPayload) => {
    const isFirstUnfilteredPage = filters.page === 1 && !filters.type && !filters.q;
    if (isFirstUnfilteredPage) {
      queryClient.setQueryData<Paginated<TransactionDTO>>(transactionKeys.list(accountId, filters), (old) => {
        if (!old || old.data.some((t) => t.id === transaction.id)) return old;
        return {
          data: [transaction, ...old.data].slice(0, filters.pageSize),
          meta: { ...old.meta, total: old.meta.total + 1 },
        };
      });
    }
    // Other pages and filters: mark stale, refetch in the background.
    void queryClient.invalidateQueries({ queryKey: transactionKeys.all(accountId), refetchType: isFirstUnfilteredPage ? 'none' : 'active' });

    if (!clientMutationId || !myMutationIds.current.has(clientMutationId)) {
      toast.info('New transaction', { description: transaction.description });
    }
  },
});
```

And in `onSubmit`, right after `const idempotencyKey = crypto.randomUUID();`, add:

```ts
myMutationIds.current.add(idempotencyKey);
```

> **Why `setQueryData` for page one but `invalidateQueries` elsewhere?** On the first unfiltered page you know exactly where the new row goes: the top. On page 3, or with a "credits only" filter, inserting it correctly would mean re-implementing the server's sorting and filtering in the browser. Let the server do it.

> **Gotcha:** The event and the Server Action response race. Sometimes the socket event arrives first, sometimes the refetch after `invalidateQueries`. The `some((t) => t.id === transaction.id)` check makes the update **idempotent**, so the row never appears twice.

### [Beginner] Step 17 — Connection status indicator

```tsx
// src/features/realtime/components/connection-status.tsx
'use client';

import { useConnectionStatus, type ConnectionStatus } from '../socket-provider';

const LABELS: Record<ConnectionStatus, { text: string; dot: string }> = {
  connecting: { text: 'Connecting', dot: 'bg-gray-400' },
  connected: { text: 'Live', dot: 'bg-green-500' },
  reconnecting: { text: 'Reconnecting', dot: 'bg-amber-500 animate-pulse' },
  offline: { text: 'Offline', dot: 'bg-red-500' },
  unauthorized: { text: 'Session ended, sign in again', dot: 'bg-red-500' },
};

export function ConnectionStatusBadge() {
  const status = useConnectionStatus();
  const { text, dot } = LABELS[status];
  return (
    <span className="flex items-center gap-2 text-xs text-gray-600" role="status" aria-live="polite">
      <span className={`h-2 w-2 rounded-full ${dot}`} aria-hidden />
      {text}
    </span>
  );
}
```

Wire it into the protected layout. Wrap the content in `SocketProvider` inside `SessionProvider`, and show the badge in the header:

```tsx
// src/app/(app)/layout.tsx (updated return)
return (
  <SessionProvider session={session}>
    <SocketProvider>
      <header className="flex items-center justify-between border-b bg-white px-6 py-3">
        <Link href="/accounts" className="font-semibold">Ledger</Link>
        <div className="flex items-center gap-4">
          <ConnectionStatusBadge />
          <UserMenu />
        </div>
      </header>
      <main className="mx-auto max-w-4xl p-6">{children}</main>
    </SocketProvider>
  </SessionProvider>
);
```

Imports: `import { SocketProvider } from '@/features/realtime/socket-provider';` and `import { ConnectionStatusBadge } from '@/features/realtime/components/connection-status';`.

**Check it works:** run `npm run dev`, sign in as Alice and open the "Everyday" account in **two** browser windows side by side.

1. The header badge shows a green dot and "Live" in both.
2. Add a debit of `3.00` "Snack" in window 1. Window 2's balance flashes blue and changes, the row appears at the top of its table, and it shows a "New transaction" toast. Window 1 shows "Transaction recorded" only.
3. Stop the server (Ctrl+C). Both badges switch to "Reconnecting". Add nothing, start the server again: both return to "Live" within about 10 seconds.
4. In window 2 open DevTools, Network, and choose "Offline". Record a transaction in window 1, then set window 2 back to "No throttling". Window 2 reconnects. Within the 2-minute recovery window Socket.IO replays the missed events. After a longer gap `socket.recovered` is false and the provider refetches the table instead. Either way the new row shows up.

Milestone tree:

```text
ledger-web/
├─ server.ts
├─ dist/server.js                       (after npm run build)
├─ scripts/socket-smoke.ts
└─ src/
   ├─ app/(app)/layout.tsx               SocketProvider + status badge
   ├─ features/realtime/
   │  ├─ events.ts
   │  ├─ client-socket.ts
   │  ├─ socket-provider.tsx
   │  ├─ use-account-channel.ts
   │  └─ components/{live-balance.tsx, connection-status.tsx}
   ├─ lib/realtime/{io.ts, publish.ts, redis-emitter.ts}
   └─ server/{load-env.ts, realtime/{log.ts, socket-auth.ts, handlers.ts}}
```

## 5. Option B: Server-Sent Events for one-way updates

### [Intermediate] Step 18 — When SSE is enough

Server-Sent Events are a plain HTTP response that never ends. The server writes lines like `event: account:balance` and `data: {...}` followed by a blank line, and the browser's built-in `EventSource` parses them and **reconnects automatically**. No library, no upgrade, no special proxy settings beyond "do not buffer".

| | SSE | Socket.IO |
|---|---|---|
| Direction | server to browser only | both ways |
| Transport | normal HTTP, works through most proxies | WebSocket (or long-polling fallback) |
| Reconnect | built in, with `Last-Event-ID` | built in, with rooms re-joined by your code |
| Auth | cookies sent automatically (same origin) | cookies on handshake, or `auth` payload |
| Where it can run | route handler on any long-running Node server; on serverless only until the function time limit, then the browser reconnects | long-lived Node process only |
| Limits | about 6 connections per domain on HTTP/1.1 (not an issue on HTTP/2) | none of that kind |

For ledger-web's "live balance and feed", SSE does everything the UI needs. Writes still go through Server Actions.

### [Intermediate] Step 19 — An in-process event bus

The SSE route handler needs to hear about new transactions. Socket.IO had rooms. For SSE, a Node `EventEmitter` on `globalThis` plays that role inside one process.

```ts
// src/lib/realtime/bus.ts
import 'server-only';
import { EventEmitter } from 'node:events';
import type { BalanceUpdatedPayload, TransactionCreatedPayload } from '@/features/realtime/events';

export type AccountEvent =
  | { type: 'transaction:created'; payload: TransactionCreatedPayload }
  | { type: 'account:balance'; payload: BalanceUpdatedPayload };

const g = globalThis as unknown as { __ledgerBus?: EventEmitter };
const bus = (g.__ledgerBus ??= new EventEmitter().setMaxListeners(0)); // one listener per open stream

export function publishAccountEvent(accountId: string, event: AccountEvent): void {
  bus.emit(`account:${accountId}`, event);
}

export function subscribeAccountEvents(accountId: string, listener: (e: AccountEvent) => void): () => void {
  const key = `account:${accountId}`;
  bus.on(key, listener);
  return () => bus.off(key, listener);
}
```

Publish to the bus too. At the very top of `publish.transactionCreated`, before `const io = getIO();`, add:

```ts
// src/lib/realtime/publish.ts (first lines inside transactionCreated)
publishAccountEvent(payload.accountId, { type: 'transaction:created', payload });
publishAccountEvent(balance.accountId, { type: 'account:balance', payload: balance });
```

with `import { publishAccountEvent } from './bus';` at the top.

> **Gotcha:** This bus is per process, exactly like the in-memory rate limiter. With several instances, a transaction recorded on instance A never reaches a stream held by instance B. In production, back the bus with Redis pub/sub: `publishAccountEvent` does `PUBLISH`, and each instance `SUBSCRIBE`s once and re-emits locally.

### [Intermediate] Step 20 — The SSE route handler

```ts
// src/app/api/v1/accounts/[id]/events/route.ts
import { withAuth } from '@/lib/api/handler';
import { getAccount } from '@/features/accounts/service';
import { subscribeAccountEvents } from '@/lib/realtime/bus';

export const dynamic = 'force-dynamic';

type Params = { id: string };

export const GET = withAuth<Params>(async ({ req, user, params, log }) => {
  await getAccount(user.id, params.id); // ownership, throws 404 before the stream starts

  const encoder = new TextEncoder();
  let cleanup = () => {};

  const stream = new ReadableStream<Uint8Array>({
    start(controller) {
      let eventId = 0;
      const write = (chunk: string) => {
        try {
          controller.enqueue(encoder.encode(chunk));
        } catch {
          cleanup(); // stream already closed
        }
      };
      const send = (event: string, data: unknown) => {
        eventId += 1;
        write(`id: ${eventId}\nevent: ${event}\ndata: ${JSON.stringify(data)}\n\n`);
      };

      write('retry: 5000\n\n'); // ask the browser to wait 5s before reconnecting
      send('ready', { accountId: params.id });

      const unsubscribe = subscribeAccountEvents(params.id, (e) => send(e.type, e.payload));
      // Comments keep proxies and load balancers from closing an idle connection.
      const heartbeat = setInterval(() => write(': ping\n\n'), 15_000);

      cleanup = () => {
        clearInterval(heartbeat);
        unsubscribe();
        try {
          controller.close();
        } catch {
          // already closed
        }
        log.debug('sse stream closed');
      };
      req.signal.addEventListener('abort', cleanup, { once: true });
    },
    cancel() {
      cleanup();
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream; charset=utf-8',
      'Cache-Control': 'no-cache, no-transform',
      Connection: 'keep-alive',
      'X-Accel-Buffering': 'no', // tell nginx not to buffer
    },
  });
});
```

```mermaid
sequenceDiagram
  participant B as Browser EventSource
  participant R as SSE route handler
  participant Bus as Event bus
  participant S as createTransaction
  B->>R: GET /api/v1/accounts/id/events with cookie
  R->>R: withAuth and ownership check
  R-->>B: 200 text/event-stream, retry and ready
  R->>Bus: subscribe account id
  S->>Bus: publish transaction created after commit
  Bus-->>R: event
  R-->>B: event transaction created with JSON data
  R-->>B: ping comment every 15s
  B->>R: tab closed, request aborted
  R->>Bus: unsubscribe, clear heartbeat
```

> **Gotcha:** Cleanup is the whole game with long-lived responses. Without the `abort` listener, every closed tab leaves a listener on the bus and an interval running: a memory leak that only shows up after days in production.

> **Gotcha:** `withErrorHandling` logs "request completed" as soon as the `Response` is returned, which is when the stream **starts**. For SSE that is fine. Log the close separately, as above.

**Check it works:**

```bash
curl -N -b "$C" http://localhost:3000/api/v1/accounts/PASTE_ACCOUNT_ID/events
```

```text
retry: 5000

id: 1
event: ready
data: {"accountId":"cm..."}

: ping
```

Add a transaction in the browser and two more events appear in the terminal. Press Ctrl+C and the dev log prints `sse stream closed`.

### [Beginner] Step 21 — The EventSource hook

```ts
// src/features/realtime/use-account-events.ts
'use client';

import { useEffect, useEffectEvent } from 'react';
import type { BalanceUpdatedPayload, TransactionCreatedPayload } from './events';

type Handlers = {
  onTransaction?: (p: TransactionCreatedPayload) => void;
  onBalance?: (p: BalanceUpdatedPayload) => void;
};

export function useAccountEvents(accountId: string, handlers: Handlers): void {
  const onTransaction = useEffectEvent((p: TransactionCreatedPayload) => handlers.onTransaction?.(p));
  const onBalance = useEffectEvent((p: BalanceUpdatedPayload) => handlers.onBalance?.(p));

  useEffect(() => {
    const es = new EventSource(`/api/v1/accounts/${accountId}/events`);
    const handleTransaction = (e: MessageEvent<string>) => onTransaction(JSON.parse(e.data) as TransactionCreatedPayload);
    const handleBalance = (e: MessageEvent<string>) => onBalance(JSON.parse(e.data) as BalanceUpdatedPayload);

    es.addEventListener('transaction:created', handleTransaction);
    es.addEventListener('account:balance', handleBalance);
    es.onerror = () => {
      // The browser reconnects by itself. CLOSED means it gave up (for example a 401 or 404).
      if (es.readyState === EventSource.CLOSED) console.warn('event stream closed for', accountId);
    };

    return () => es.close();
  }, [accountId]);
}
```

It has the same signature as `useAccountChannel`. Swap one import in `LiveBalance` and `TransactionsPanel` and the UI works the same, with no custom server: plain `npm run dev:next` is enough.

> **Gotcha:** `EventSource` cannot set headers, so it cannot send a bearer token. Same-origin cookies work. For cross-origin streams, use `new EventSource(url, { withCredentials: true })` with CORS, or a `fetch()` stream reader instead of `EventSource`.

> **Interview tip:** If the interviewer asks "how would you add live updates to a Next.js app on Vercel?", a strong answer is: "SSE from a route handler for a quick win, knowing functions time out and the client reconnects; or a managed pub/sub service like Ably or Pusher; and a separate realtime service if we need two-way messaging at scale."

## 6. Production

### [Advanced] Step 22 — Scale out with the Redis adapter

Each Socket.IO server only knows its own sockets. With two instances behind a load balancer, `io.to('account:123').emit(...)` on instance A misses the tabs connected to instance B. The **Redis adapter** fixes it: every broadcast is also published to Redis, and every instance delivers it to its local sockets.

```mermaid
flowchart LR
  A["Tab 1"] --> LB["Load balancer"]
  B["Tab 2"] --> LB
  LB --> N1["ledger-web instance 1<br/>Next.js + Socket.IO"]
  LB --> N2["ledger-web instance 2<br/>Next.js + Socket.IO"]
  N1 <-->|"pub/sub"| R["Redis"]
  N2 <-->|"pub/sub"| R
  W["Worker or Vercel function<br/>redis-emitter"] -->|"publish"| R
  N1 --> P["Postgres"]
  N2 --> P
```

```bash
npm install redis @socket.io/redis-adapter
```

```yaml
# docker-compose.yml (add under services)
  redis:
    image: redis:7
    ports:
      - "6379:6379"
```

```ts
// server.ts (after `const io = new Server(...)`, before io.use)
import { createClient } from 'redis';
import { createAdapter } from '@socket.io/redis-adapter';

if (process.env.REDIS_URL) {
  const pubClient = createClient({ url: process.env.REDIS_URL });
  const subClient = pubClient.duplicate();
  pubClient.on('error', (err) => rtLog.error({ err }, 'redis pub error'));
  subClient.on('error', (err) => rtLog.error({ err }, 'redis sub error'));
  await Promise.all([pubClient.connect(), subClient.connect()]);
  io.adapter(createAdapter(pubClient, subClient));
  rtLog.info('socket.io redis adapter enabled');
}
```

Move the two `import` lines to the top of `server.ts` with the others (after `load-env`). Add `REDIS_URL="redis://localhost:6379"` to `.env` and `REDIS_URL: z.url().optional()` to the schema in `src/env.ts`.

> **Gotcha:** Connection state recovery (the `socket.recovered` flag) is supported by the default in-memory adapter and by some others (such as the Redis **Streams** adapter), but not by the classic Redis adapter used here. That is fine for ledger-web: the provider refetches whenever `socket.recovered` is false. Check the adapter documentation before relying on recovery.

**Check it works:** start two instances on different ports against the same Redis and database:

```bash
PORT=3000 npm run dev
PORT=3001 npm run dev   # second terminal
```

Open the "Everyday" account on `localhost:3000` in one window and on `localhost:3001` in another (cookies are not scoped by port, so signing in once on 3000 also signs you in on 3001). Record a transaction on 3000. The window on 3001 updates too. Stop Redis (`docker compose stop redis`) and the cross-instance update stops while same-instance updates continue.

### [Advanced] Step 23 — Sticky sessions, proxies and timeouts

Socket.IO starts with HTTP long-polling by default and then upgrades. A polling session is many HTTP requests that **must reach the same instance**. Behind a load balancer that needs **sticky sessions** (session affinity), or you see `400 Session ID unknown` errors.

Two ways out:

1. **WebSocket only** (what `client-socket.ts` does with `transports: ['websocket']`). One TCP connection, no stickiness needed. Downside: no fallback for networks that block WebSockets (rare today, but some corporate proxies do).
2. **Keep polling and enable stickiness** in the load balancer: cookie-based affinity on AWS ALB, `ip_hash` or a cookie in nginx, session affinity on Cloud Run.

An nginx front for either mode:

```nginx
# nginx.conf (excerpt)
upstream ledger_web {
  ip_hash;                       # stickiness for polling clients
  server app1:3000;
  server app2:3000;
}

server {
  listen 443 ssl;
  location /socket.io/ {
    proxy_pass http://ledger_web;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_read_timeout 75s;      # longer than pingInterval + pingTimeout
  }
  location / {
    proxy_pass http://ledger_web;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_buffering off;         # lets SSE responses through immediately
  }
}
```

> **Gotcha:** Every hop has an idle timeout: the load balancer (AWS ALB default 60 seconds), nginx, the platform. Socket.IO's heartbeat (`pingInterval` 25 seconds) keeps the connection active, but the timeout must be longer than `pingInterval + pingTimeout`. Some platforms also cap total request duration (for example Cloud Run's configurable request timeout), which closes sockets periodically. The client reconnects, so design for it rather than fighting it.

### [Intermediate] Step 24 — Containerize the custom server

The standalone output from the first guide does not include `server.ts`. Ship the regular `.next` build, the bundled `dist/server.js` and production dependencies. Remove `output: 'standalone'` from `next.config.ts`.

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
ENV DATABASE_URL=postgresql://build:build@localhost:5432/build
RUN npm run build

FROM builder AS migrate
CMD ["npx", "prisma", "migrate", "deploy"]

FROM node:22-alpine AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1 PORT=3000 HOST=0.0.0.0
RUN addgroup -S nodejs && adduser -S nextjs -G nodejs
COPY --from=prod-deps /app/node_modules ./node_modules
COPY --from=builder /app/package.json /app/next.config.ts ./
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next ./.next
COPY --from=builder /app/dist ./dist
USER nextjs
EXPOSE 3000
STOPSIGNAL SIGTERM
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://127.0.0.1:3000/api/health || exit 1
CMD ["node", "dist/server.js"]
```

**Check it works:**

```bash
docker build -t ledger-web-rt .
docker run --rm --network host --env-file .env -e NODE_ENV=production ledger-web-rt
```

```text
{"level":30,"component":"realtime","port":3000,"dev":false,"msg":"ledger-web ready on http://localhost:3000"}
```

Open two windows and repeat the two-window check from Step 17 against the container.

> **Why not Vercel?** Vercel runs Next.js as functions and does not run your `server.ts`, so there is no place for Socket.IO to live. If you want Vercel for the web app, keep Socket.IO in a separate container (Option B) and publish to it with the Redis emitter from Step 12, or use SSE or a managed service.

> **Gotcha:** On deploy, `SIGTERM` reaches `server.ts`, which calls `io.close()`. Every client disconnects and reconnects, possibly all at the same second, to the new instances. Socket.IO's client adds randomized backoff (`randomizationFactor`) to spread that thundering herd. Make sure the load balancer drains connections instead of cutting them.

### [Intermediate] Step 25 — Monitoring

What to watch, and how:

| Signal | Source | Alert when |
|---|---|---|
| Connected sockets | `io.engine.clientsCount` per instance | sudden drop to near zero (deploy, LB issue) |
| Handshake failures | `connect_error` / auth middleware logs | spike: secret mismatch after a deploy, or an attack |
| Disconnect reasons | `socket.on('disconnect', reason)` | many `ping timeout`: network or proxy timeouts |
| Emit failures | `realtime emit failed` / `redis emit failed` logs | any sustained rate |
| Redis health | adapter client `error` events | any |
| Event latency | `at` in the payload vs receive time on the client (sample via RUM) | p95 above a second |

Expose the socket count in the existing health route:

```ts
// src/app/api/health/route.ts
import { prisma } from '@/lib/db';
import { getIO } from '@/lib/realtime/io';

export const dynamic = 'force-dynamic';

export async function GET() {
  const realtime = getIO() ? { enabled: true, sockets: getIO()?.engine.clientsCount ?? 0 } : { enabled: false };
  try {
    await prisma.$queryRaw`SELECT 1`;
    return Response.json({ status: 'ok', realtime });
  } catch {
    return Response.json({ status: 'db_unavailable', realtime }, { status: 503 });
  }
}
```

```bash
curl -s http://localhost:3000/api/health
```

```text
{"status":"ok","realtime":{"enabled":true,"sockets":2}}
```

Also useful:

- `@socket.io/admin-ui` shows sockets, rooms and events live. Enable it only in development or behind admin auth.
- Load-test before launch: Artillery has a Socket.IO engine. Measure connections per instance and memory per socket.
- Log one line per connect and disconnect at `debug`, and aggregate counts at `info` every minute, so logs stay affordable.

## 7. Interview questions

#### Q: Why can't you just run Socket.IO inside a Next.js route handler on Vercel?

A WebSocket server needs a process that stays alive for the whole connection and keeps in-memory state about sockets and rooms. Vercel runs route handlers as functions that handle a request and can be frozen or destroyed afterwards, with a maximum duration, and they cannot accept WebSocket upgrades. Even if one could hold a connection, the instance that records a transaction is usually not the one holding the socket. You need a long-lived server (custom server, separate service), a managed realtime service, or SSE with reconnects.

#### Q: How do you authenticate a Socket.IO connection in a Next.js app that uses Auth.js?

Same origin means the browser sends the session cookie with the handshake. In `io.use` middleware, parse the `Cookie` header, take `authjs.session-token` (or the `__Secure-` variant), and decrypt it with `decode` from `next-auth/jwt` using `AUTH_SECRET` and the cookie name as salt. Reject with an error if it is missing, invalid or expired, and store `userId`, role and expiry on `socket.data`. Then authorize every room join with an ownership query, and disconnect the socket when the session expires.

#### Q: Where in the code do you emit "transaction created", and why there?

In the service layer, after the database transaction commits. After commit, so clients never see money that a rollback undid. In the service, so every entry point (Server Action, JSON API, background job) emits without duplicating code. The emit is best effort: it must not fail the payment, and clients refetch after reconnecting. For events that must never be lost, use a transactional outbox.

#### Q: Why store the Socket.IO server on globalThis?

The custom `server.ts` and the Next.js server code are compiled separately, so a shared source file becomes two module instances in the same process. A module-level variable set by `server.ts` is invisible to Server Actions. `globalThis` is shared by the whole process, so both sides see the same `io`. When the code runs in a different process or machine, use the Redis emitter instead.

#### Q: How do you scale Socket.IO to multiple instances?

Add the Redis adapter so broadcasts reach sockets on every instance. Either force the WebSocket transport, or enable sticky sessions so long-polling requests reach the same instance. Set load balancer idle timeouts above the heartbeat interval, drain connections on deploy, and let clients reconnect with randomized backoff. Publish from other processes with the Redis emitter.

#### Q: SSE or WebSockets for a live balance?

SSE, in most cases. Balances flow one way, from server to browser. SSE is plain HTTP, reconnects automatically, passes through proxies easily and needs no library. WebSockets win when the client sends frequent messages (chat, collaborative editing, presence) or when you need very high message rates. Both need a shared bus (Redis) once you have more than one instance.

#### Q: How do you keep the client cache correct when events and refetches race?

Make updates idempotent: insert a transaction into the cached list only if its id is not already there. Order by version or timestamp and ignore events older than what is shown. Only patch the cache where the position is obvious (first unfiltered page) and invalidate everything else. Refetch after a reconnect that could not recover missed events. And always clean up listeners in effects, which StrictMode verifies in development by mounting twice.

## Cheatsheet

```text
Options
  custom server (server.ts + Socket.IO)   one container, same cookie, no Vercel, no standalone
  separate realtime service                Next on Vercel ok, cross-origin auth token, Redis emitter
  managed (Pusher, Ably, Supabase, Liveblocks)  REST publish from server, client SDK
  SSE route handler                        one-way, EventSource, auto-reconnect, needs shared bus

server.ts
  import './src/server/load-env'           first import: loads .env before env.ts runs
  const app = next({ dev, hostname, port }); await app.prepare()
  createServer((req, res) => handle(req, res))
  new Server(httpServer, { destroyUpgrade: false, connectionStateRecovery: {...} })
  io.use(authenticateSocket); io.on('connection', registerHandlers); setIO(io)
  scripts: dev "tsx server.ts"   build:server esbuild --bundle --platform=node --packages=external

Types
  Server<ClientToServer, ServerToClient, InterServer, SocketData>
  client Socket<ServerToClient, ClientToServer>   (order flips)

Auth on handshake
  parse(socket.handshake.headers.cookie)
  decode({ token, secret: AUTH_SECRET, salt: cookieName })   from 'next-auth/jwt'
  socket.data = { userId, role, expiresAt }   disconnect at expiry
  join rooms only after an ownership query

Emitting
  after commit, in the service, best effort
  io.to(rooms.account(id)).to(rooms.user(uid)).emit(...)   union, no duplicates
  getIO() via globalThis   else Redis emitter: new Emitter(redisClient).to(room).emit(...)

Client
  one socket per tab: io({ autoConnect: false, transports: ['websocket'] })
  provider effect: on handlers, connect(); cleanup: off handlers, disconnect()
  hook: subscribe on mount and on every 'connect', off with same function refs
  useEffectEvent for latest handlers   StrictMode connects twice in dev
  setQueryData only where position is obvious, dedupe by id   else invalidateQueries
  !socket.recovered after reconnect -> refetch

SSE
  new Response(ReadableStream, { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache, no-transform' })
  "event: x\ndata: {...}\n\n"   ": ping\n\n" heartbeat   "retry: 5000"
  req.signal abort -> clearInterval, unsubscribe, close
  client: new EventSource(url); es.addEventListener('x', ...); es.close() on unmount

Production
  @socket.io/redis-adapter   sticky sessions OR websocket-only transport
  LB idle timeout > pingInterval + pingTimeout   drain on SIGTERM, io.close()
  no output: 'standalone' with a custom server   container, not Vercel
  watch: clientsCount, connect_error rate, disconnect reasons, emit failures
```

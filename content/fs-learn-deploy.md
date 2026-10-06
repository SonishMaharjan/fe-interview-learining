---
id: fs-learn-deploy
title: "Full-stack 3: Automate Testing and Deploy to Production"
group: "Full-stack Website: Next.js + Payload + Postgres"
tagline: Take my-site from "works on my laptop" to a tested, automatically deployed production website on Vercel + Neon or on AWS.
covers: "GitHub, GitHub Actions, Postgres service containers, Playwright, Vercel, Neon Postgres, Vercel Blob, Docker, Amazon ECR, ECS Express Mode, RDS Postgres, S3, Secrets Manager, GitHub OIDC, Sentry, Dependabot"
status: current
kind: guide
---

This is guide 3 of 3. In guide 1 you built `my-site`: a Next.js App Router website with Payload CMS 3.x living
inside it and a Postgres 16 database running in Docker. In guide 2 you added tests. Right now the whole thing
runs only on your laptop. In this guide you will put it on the internet, safely, and make every future change
flow there automatically.

Here is what you already have. If something in your project is named differently, use your name wherever this
guide uses the one below.

```text
my-site/
  docker-compose.yml          <- service "db": Postgres 16, user/password postgres, database my_site
  .env                        <- your real local values (git-ignored)
  .env.example                <- the same keys with placeholder values (committed)
  next.config.mjs             <- wrapped with withPayload()
  package.json                <- scripts: dev, build, start, lint, typecheck, test, e2e, migrate, seed
  playwright.config.ts
  vitest.config.mts
  src/
    payload.config.ts
    collections/              <- Users.ts, Media.ts, Pages.ts, Posts.ts, ContactSubmissions.ts
    migrations/               <- created by "npx payload migrate:create"
    seed.ts                   <- the seed script behind "npm run seed"
    app/
      (frontend)/             <- /, /[slug], /blog, /blog/[slug], /contact
      (payload)/admin/        <- the Payload admin panel at /admin
  tests/
    unit/                     <- Vitest
    e2e/                      <- Playwright
```

By the end you will have:

- A GitHub repository where nobody (including you) can push untested code to `main`.
- A CI workflow that spins up a real Postgres, runs your migrations, lint, type check, unit tests, a production
  build, seeds data and runs the Playwright tests on every pull request.
- **Path A (recommended):** the site on Vercel, with the database on Neon and uploaded images in Vercel Blob.
  Every pull request gets its own preview website **and its own copy of the database**.
- **Path B (for AWS shops):** the same site in a Docker container on Amazon ECS, with RDS Postgres, S3 for
  images and a GitHub Actions deploy pipeline that logs in to AWS without any stored keys.
- Monitoring, backups you have actually tested, automatic dependency updates and a release checklist.

You only need to do **one** of Path A or Path B. Read both if you are preparing for interviews.

> **Outdated:** Versions move fast. This guide was checked in October 2026 against Payload 3.x, Next.js 15/16
> (whichever `create-payload-app` installed for you), Node 22 LTS, GitHub Actions v4 to v6 action majors,
> Neon, Vercel and AWS as they were then. Where a detail is likely to change, the text says so. When a
> screen or flag looks different, trust the official docs over this guide.

## 1. Concepts: what "production" really means

Before typing anything, you need a mental model. Most deployment pain comes from not knowing *where* code is
running and *which* database and secrets it is using. These four short concepts fix that.

### [Beginner] Step 1 — Concept: what "production" means

**What we're doing:** defining the word everyone uses but rarely explains.

**Why:** "It works on my machine" is the most common sentence in software. Production is the place where it has
to work for everyone, all the time, without you watching.

**Production** is the copy of your website that real visitors use, at your real domain (for example
`https://www.my-site.com`), connected to the real database with real content and real contact-form messages.
It has three properties your laptop does not:

| Property | Your laptop (`npm run dev`) | Production |
| --- | --- | --- |
| Who uses it | Only you | Anyone on the internet |
| How the code runs | Dev server: compiles pages on demand, shows error overlays, hot reloads | `next build` then `next start` (or a platform like Vercel): optimised, cached, no overlays |
| Data | Throwaway data in a Docker Postgres | Precious data. Losing it is a real incident |
| Mistakes | Restart and move on | Visitors see errors, editors lose work, you may leak secrets |
| Machines | One, always on | Possibly many short-lived machines that start and stop on their own |

That last row matters a lot for a CMS. On Vercel your code runs in **serverless functions**: tiny machines that
start when a request arrives and disappear afterwards. On AWS ECS your code runs in **containers**: packaged
copies of your app that can be replaced at any moment (every deploy replaces them). In both cases **the disk
is temporary**. Anything written to it is gone soon. Keep that in mind; it comes back in Step 4.

**What just happened:** you now have a definition to test every decision against: "Would this be safe and
correct if many strangers used it at once, on machines that can vanish at any time?"

### [Beginner] Step 2 — Concept: environments (local, preview, production)

**What we're doing:** naming the three places your code will run.

**Why:** if you do not separate environments, you end up testing on production (scary) or shipping code that
was never tested anywhere that resembles production (also scary).

An **environment** is one complete running copy of your app: the code, the database it talks to, the secrets
it uses and the URL people reach it on.

- **Local:** your laptop. `npm run dev`, Postgres in Docker, `.env` file. Fast feedback, fake data.
- **Preview** (sometimes called staging): a temporary copy of the site built from a pull request, with its own
  URL. Teammates and reviewers click around before the change is merged. It should use its own database, never
  the production one.
- **Production:** built from the `main` branch. Real domain, real data.

```mermaid
flowchart LR
  Dev["Your laptop<br/>npm run dev"] -->|"git push branch"| GH["GitHub<br/>pull request"]
  GH -->|"CI checks"| CI["GitHub Actions<br/>throwaway Postgres"]
  GH -->|"auto deploy"| Prev["Preview<br/>pr-12.my-site URL"]
  Prev --> PrevDB["Preview database<br/>branch of prod"]
  GH -->|"merge to main"| Prod["Production<br/>www.my-site.com"]
  Prod --> ProdDB["Production database"]
  Dev --> LocalDB["Local Postgres<br/>in Docker"]
```

Notice that **every environment has its own database**. That is the single most important rule in this
diagram. A preview build that runs a migration against the production database can break the live site
before anyone has approved the change.

**What just happened:** you have three boxes to put every setting into. From now on, when you set an
environment variable, ask "for which of the three?"

### [Beginner] Step 3 — Concept: CI and CD, and why we automate

**What we're doing:** learning two acronyms you will hear in every interview.

**Why:** humans forget steps. A machine that runs the same checklist on every change never forgets.

- **CI, Continuous Integration:** every time someone pushes code, a machine automatically installs it, checks
  it (lint, types, tests) and builds it. If anything fails, the change cannot be merged. "Integration" means
  merging everyone's work into `main` often and safely.
- **CD, Continuous Delivery or Deployment:** after a change is merged and CI passed, a machine automatically
  deploys it. *Delivery* means it is ready to deploy with one click; *Deployment* means it deploys with no
  click at all. We will do the second.
- **Pipeline:** the ordered list of automated steps from "git push" to "live on the internet".
- **GitHub Actions:** GitHub's built-in robot that runs pipelines. You describe the steps in a YAML file
  inside `.github/workflows/`. Each run happens on a fresh, empty virtual machine called a **runner**.

```mermaid
flowchart TD
  Push["git push to a branch"] --> PR["Open pull request"]
  PR --> CI["CI: install, migrate, lint,<br/>typecheck, unit tests, build, e2e"]
  CI --> Pass{"All checks green?"}
  Pass -->|"no"| Fix["Fix and push again"]
  Fix --> CI
  Pass -->|"yes"| Review["Review the preview site<br/>and the code"]
  Review --> Merge["Merge to main"]
  Merge --> CD["CD: migrate prod DB,<br/>build, deploy"]
  CD --> Live["Live on www.my-site.com"]
```

Why bother, when you could just run `npm run build` and upload? Because automation gives you:

1. **Consistency.** The same steps, in the same order, every time.
2. **A safety net.** Broken code is stopped at the pull request, not discovered by a visitor.
3. **Speed.** Merging is deploying. No "release day".
4. **History.** Every deploy is linked to a commit and a log, so you can see what changed and roll back.

**What just happened:** you know what the robot does and why. The next parts build that robot one step at a
time.

### [Beginner] Step 4 — Concept: secrets vs config, and why you need external storage and a managed database

**What we're doing:** sorting settings into two buckets, and understanding the two services every CMS needs in
production.

**Why:** leaked secrets are the most common way small sites get hacked. And uploading an image that disappears
an hour later is the most common "why is my CMS broken" question.

**Config vs secrets.** Both are **environment variables**: named values the operating system hands to your
program, read in code as `process.env.NAME`.

| | Config | Secret |
| --- | --- | --- |
| Example | `NEXT_PUBLIC_SERVER_URL=https://www.my-site.com` | `PAYLOAD_SECRET`, `DATABASE_URI` (contains a password), `BLOB_READ_WRITE_TOKEN` |
| Harm if it leaks | None | Someone can read or wipe your data, or forge admin logins |
| Can be in git? | Yes, as a default or in `.env.example` | Never. Only placeholders in `.env.example` |
| Where it lives | Code, `.env.example`, GitHub "Variables" | `.env` (local only), GitHub "Secrets", Vercel env vars, AWS Secrets Manager |
| `NEXT_PUBLIC_` prefix allowed? | Yes | **Never.** That prefix copies the value into JavaScript sent to every browser |

`PAYLOAD_SECRET` deserves a word. Payload uses it to sign login tokens (JWTs) and encrypt some data. If someone
knows it, they can forge a login as any admin. Each environment gets its own long random value.

**Why serverless and containers need external storage for media.** In guide 1, uploads to the `media`
collection were saved to a folder on your disk. In production that breaks:

```mermaid
flowchart LR
  Editor["Editor uploads<br/>hero.jpg"] --> I1["Instance 1<br/>saves to its disk"]
  Visitor["Visitor requests<br/>hero.jpg"] --> I2["Instance 2<br/>empty disk"]
  I2 --> Miss["404 image missing"]
  Deploy["Next deploy"] --> Gone["Instance 1 replaced<br/>file lost forever"]
```

The fix is **object storage**: a separate service that stores files by name and serves them over HTTPS.
Vercel Blob and Amazon S3 are object storage. Payload has official **storage adapters**
(`@payloadcms/storage-vercel-blob`, `@payloadcms/storage-s3`) that upload to them instead of disk.

**Why a managed database.** Your Docker Postgres lives on your laptop. Production needs a database that is
always on, reachable from the app, backed up automatically, patched and monitored. A **managed database** is
Postgres run for you by a provider: Neon (Path A) or Amazon RDS (Path B). You get a **connection string**,
the same `postgres://user:password@host:port/database` shape as your local `DATABASE_URI`, and the provider
handles disks, backups and upgrades.

**What just happened:** you now know the three external pieces production needs (managed Postgres, object
storage, a secret store) and why each exists. Every later step plugs one of them in.

> **Interview tip:** "Why can't a serverless app save uploads to disk?" Because instances are ephemeral and
> not shared. Files must go to object storage, and state must go to a database. This is the "stateless app
> servers" principle from the Twelve-Factor App.

## 2. Git and GitHub, properly

Git tracks every change to your code. GitHub hosts that history online and adds pull requests, reviews and
Actions. You may have used both casually. Here we set them up the way a real team does.

### [Beginner] Step 5 — Create the repository and push

**What we're doing:** putting `my-site` on GitHub.

**Why:** Vercel, GitHub Actions and AWS all deploy from the GitHub repository. It becomes the single source of
truth for "what code is running".

**Do it:** first make sure secrets are not about to be committed. Open `.gitignore` and check it contains at
least these lines (`create-payload-app` adds most of them; add any that are missing):

```text
# .gitignore
node_modules
.next
out
build
coverage

# env files: real values never go to git
.env
.env*.local
.env.production

# local uploads from guide 1 (production uses object storage)
media

# test output
playwright-report
test-results
blob-report

# OS and editor
.DS_Store
.vercel
```

Now create the repo. The GitHub CLI (`gh`) is the easiest way; install it from cli.github.com and run
`gh auth login` once.

```bash
cd my-site
git status                      # shows files git sees. .env must NOT be listed
git add .
git commit -m "chore: initial commit of my-site"
gh repo create my-site --private --source=. --remote=origin --push
```

`gh repo create` makes an empty private repo on GitHub, adds it as the remote called `origin` and pushes your
commit. Without `gh`, create an empty repo on github.com, then run `git remote add origin <url>` and
`git push -u origin main`.

**Check it works:**

```bash
git log --oneline -1
gh repo view --web              # opens the repo in your browser
git ls-files | grep -E "^\.env$" || echo "good: .env is not tracked"
```

```text
a1b2c3d chore: initial commit of my-site
good: .env is not tracked
```

**What just happened:** your code now lives on GitHub. The `.gitignore` made sure the real `.env`, the build
output and the local `media` folder stayed on your laptop.

**If it breaks:**

- `.env` shows up in `git ls-files`: you committed it earlier. Run `git rm --cached .env`, commit, and **treat
  every value in it as leaked**: generate a new `PAYLOAD_SECRET`. Git history keeps old files forever.
- `error: src refspec main does not match any`: your branch is called `master`. Run `git branch -M main`.

### [Beginner] Step 6 — Concept: branches and pull requests

**What we're doing:** learning the everyday workflow.

**Why:** if everyone commits straight to `main`, and `main` is what deploys, every typo goes live instantly.

A **branch** is a parallel line of commits. You create one per change, work there, and `main` is untouched
until you are done. A **pull request (PR)** is a GitHub page that says "please merge my branch into `main`".
It shows the diff, runs CI, gets a preview deploy and collects review comments.

We use **trunk-based development**: `main` is the trunk and always deployable; branches are short (hours to a
couple of days) and named by purpose: `feat/blog-search`, `fix/contact-validation`, `chore/ci`.

```mermaid
sequenceDiagram
  participant You
  participant Git as Local git
  participant GH as GitHub
  participant CI as GitHub Actions
  You->>Git: git switch -c feat/blog-search
  You->>Git: commit changes
  Git->>GH: git push -u origin feat/blog-search
  You->>GH: gh pr create
  GH->>CI: run checks on the PR
  CI-->>GH: all green
  GH-->>You: preview URL and review
  You->>GH: squash and merge
  GH->>CI: run on main and deploy
```

**Do it:** practise the loop once with a harmless change.

```bash
git switch -c docs/readme-title
echo "" >> README.md
git commit -am "docs: touch README"
git push -u origin docs/readme-title
gh pr create --fill              # --fill uses your commit message as the PR title and body
```

**Check it works:** `gh pr view --web` opens the PR. You should see "1 commit" and "Files changed: 1". Leave it
open; we will use it in Step 9.

**What just happened:** you made a change without touching `main`. That small habit is what makes automatic
deploys safe.

### [Beginner] Step 7 — Document every variable in `.env.example`

**What we're doing:** making `.env.example` the complete, commented list of every setting the app needs, in all
environments.

**Why:** when a new teammate (or you in six months, or Vercel's settings page) asks "which variables do I need?",
the answer should be one file, not a hunt through the code. A missing variable in production is a classic
"works locally, crashes in prod" bug.

**Do it:** replace `.env.example` with this. It has placeholders only.

```bash
# .env.example
# Copy to .env for local development: cp .env.example .env
# NEVER put real secrets in this file. It is committed to git.

# --- Database (secret) ---------------------------------------------------
# Local: the Docker Postgres from docker-compose.yml
# Vercel + Neon: the Neon integration sets DATABASE_URL instead; the config reads either one
# AWS: stored in Secrets Manager
DATABASE_URI=postgres://postgres:postgres@localhost:5432/my_site

# --- Payload (secret) ----------------------------------------------------
# Signs admin login tokens. Use a different long random value per environment:
#   openssl rand -hex 32
PAYLOAD_SECRET=replace-me-with-a-long-random-string

# --- Public URL (config) -------------------------------------------------
# The URL people use to reach this environment. No trailing slash.
# Leave unset on Vercel previews; the code falls back to the preview URL.
NEXT_PUBLIC_SERVER_URL=http://localhost:3000

# --- Media storage (secret, production only) -----------------------------
# Path A: set automatically when you connect a Vercel Blob store
BLOB_READ_WRITE_TOKEN=
# Path B: the S3 bucket and region. No keys: the container uses its IAM task role
S3_BUCKET=
S3_REGION=

# --- Error tracking (config, optional, Part 6) ---------------------------
NEXT_PUBLIC_SENTRY_DSN=
```

**Check it works:**

```bash
diff <(grep -oE '^[A-Z_]+=' .env.example | sort) <(grep -oE '^[A-Z_]+=' .env | sort)
```

```text
> lines only for keys you have not set locally, such as BLOB_READ_WRITE_TOKEN=
```

Lines starting with `<` are keys in `.env.example` that your `.env` lacks. That is fine for the production-only
ones (blob, S3, Sentry). Anything else missing should be added to your `.env`.

**What just happened:** you created a contract. Every environment must provide these keys. When you add a new
variable later, the PR that uses it must also add it here.

> **Gotcha:** `NEXT_PUBLIC_*` variables are copied into the JavaScript bundle **at build time**. Changing them
> in Vercel or AWS has no effect until you rebuild and redeploy. Server-only variables are read at runtime.

### [Beginner] Step 8 — Write a README that gets a newcomer running

**What we're doing:** writing the front page of the repo.

**Why:** a README is the onboarding doc. Interviewers and teammates read it first. If they cannot start the
project in five minutes, they assume the code is messy too.

**Do it:** replace `README.md`:

````markdown
<!-- README.md -->
# my-site

A website built with Next.js (App Router) and Payload CMS 3, backed by Postgres.
Editors manage pages, blog posts and media at `/admin`.

## Requirements

- Node.js 22 LTS (see `.nvmrc`)
- Docker Desktop (for the local Postgres)

## Run it locally

```bash
cp .env.example .env          # then edit PAYLOAD_SECRET
docker compose up -d db       # start Postgres on localhost:5432
npm ci                        # install exact dependency versions
npm run migrate               # create the database tables
npm run seed                  # optional: sample pages and posts
npm run dev                   # http://localhost:3000 and http://localhost:3000/admin
```

## Scripts

| Script | What it does |
| --- | --- |
| `npm run dev` | Dev server with hot reload |
| `npm run build` / `npm start` | Production build / run the production build |
| `npm run lint` / `npm run typecheck` | ESLint / TypeScript with no output files |
| `npm test` | Vitest unit tests |
| `npm run e2e` | Playwright browser tests (needs a built app and a seeded DB) |
| `npm run migrate` | Apply pending database migrations |
| `npm run seed` | Insert sample content |

## Changing the database schema

1. Edit a collection in `src/collections`.
2. `npx payload migrate:create describe-the-change`
3. Commit the new file in `src/migrations`. CI and deploys run it automatically.

## Deploying

Merging to `main` deploys to production. Every pull request gets a preview deployment.
Database migrations run automatically during each deploy. To roll back, see "Rollback" below.

## Rollback

- Vercel: Deployments tab, pick the last good production deployment, "Instant Rollback".
- AWS: Actions > Deploy to AWS > Run workflow, enter the previous image tag (a git SHA).
````

**Check it works:** view the repo on GitHub. The README renders under the file list with working tables and
code blocks.

**What just happened:** the README now documents the same workflow CI will automate. If CI and the README ever
disagree, one of them is wrong.

### [Beginner] Step 9 — Plan branch protection (we switch it on after CI exists)

**What we're doing:** deciding the rules for `main`.

**Why:** a rule is only a rule if the system enforces it. Branch protection makes GitHub refuse direct pushes
and merges with failing checks, even from the repo owner (if you tick the box).

The rules we want on `main`:

| Rule | Why |
| --- | --- |
| Require a pull request before merging | No direct pushes; every change is visible and gets CI |
| Require status checks to pass (our CI jobs) | Red CI blocks the merge button |
| Require branches to be up to date before merging | The checks ran against the latest `main`, not an old one |
| Require conversation resolution | Review comments cannot be ignored |
| Do not allow bypassing (include administrators) | You are not above the rules at 6 pm on a Friday |
| Block force pushes and deletions | History of `main` cannot be rewritten |
| Squash merge only (repo setting) | One clean commit per PR on `main`, easy to revert |

Set the merge style now, under **Settings > General > Pull Requests**: tick **Allow squash merging**, untick
the other two, and tick **Automatically delete head branches**.

**Check it works:** on your open PR, the merge button now says **Squash and merge**.

**What just happened:** half of the safety net is planned. GitHub only lets you require a status check after
it has seen that check run once, so we enable protection in Step 15, right after CI runs.

## 3. Continuous Integration with GitHub Actions

Time to build the robot. Our CI must prove four things on every PR: the code is clean (lint), correct in shape
(types), correct in logic (unit tests), and actually works as a website against a real Postgres (build plus
end-to-end tests).

### [Beginner] Step 10 — Concept: runners, jobs and service containers

**What we're doing:** understanding what actually happens when a workflow runs, before writing one.

**Why:** YAML errors are confusing if you do not know what machine is running your commands, or why
`localhost:5432` works there.

- A **workflow** is one YAML file in `.github/workflows/`. It says *when* to run (`on:`) and *what* to run
  (`jobs:`).
- A **job** is a group of steps that run on one fresh **runner** (a Linux virtual machine GitHub lends you for
  a few minutes). Jobs run in parallel unless one `needs:` another.
- A **step** is either a shell command (`run:`) or a reusable **action** (`uses:`), which is a packaged script
  someone published, such as `actions/checkout`.
- A **service container** is a Docker container GitHub starts next to your job before the steps run. We use one
  to get a real, empty Postgres 16 for every run. With `ports: ["5432:5432"]` it is reachable at
  `localhost:5432`, exactly like your Docker Postgres at home.

```mermaid
flowchart TD
  Event["pull_request or push to main"] --> WF["Workflow ci.yml"]
  WF --> J1["Job: quality<br/>fresh runner"]
  WF --> J2["Job: e2e<br/>fresh runner"]
  J1 --> S1["lint, typecheck, unit tests"]
  J2 --> PG["Service container<br/>postgres:16 on 5432"]
  J2 --> S2["migrate, seed, build"]
  S2 --> S3["Playwright against next start"]
  S3 --> Rep["Upload HTML report"]
  PG -.->|"localhost:5432"| S2
```

**What just happened:** you know that each job starts from nothing, so it must check out code, install Node and
install packages itself. And the database it gets is brand new, so it must run migrations itself. That is a
feature: CI proves your migrations work on an empty database every single time.

### [Beginner] Step 11 — Pin the Node version and tidy the scripts

**What we're doing:** making CI, your laptop and production use the same Node version and the same script names.

**Why:** "works on Node 22, breaks on Node 20" bugs are real. And CI should call `npm run <name>` scripts, not
long commands, so that a developer can run exactly what CI runs.

**Do it:** create `.nvmrc` in the project root. Tools such as `nvm`, `fnm`, GitHub Actions and Vercel read it.

```text
22
```

> **Outdated:** Node 22 is a maintenance LTS line in late 2026 and Node 24 is the active LTS. Both work with
> Payload 3 and current Next.js. Pick one, put it in `.nvmrc`, and use the same number in the Dockerfile in
> Part 5. Check nodejs.org for the current LTS when you read this.

Now open `package.json` and make the `scripts` section look like this. Keep any extra scripts
`create-payload-app` gave you (for example `generate:types`). Two scripts are new: `build:deploy` (Part 4) and
`build:docker` (Part 5).

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "build:deploy": "payload migrate && next build",
    "build:docker": "next build --experimental-build-mode compile",
    "start": "next start",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "e2e": "playwright test",
    "migrate": "payload migrate",
    "migrate:create": "payload migrate:create",
    "seed": "payload run src/seed/index.ts",
    "generate:types": "payload generate:types",
    "generate:importmap": "payload generate:importmap"
  },
  "engines": {
    "node": ">=22"
  }
}
```

(Only the `scripts` and `engines` keys are shown. Leave `name`, `dependencies` and the rest as they are.)

| Script | Plain English |
| --- | --- |
| `build:deploy` | "Update the database schema, then build." Vercel will run this (Step 21) |
| `build:docker` | "Compile the app without pre-rendering pages", so a Docker build does not need a database (Step 27) |
| `test` | `vitest run` runs once and exits. Plain `vitest` would wait in watch mode forever in CI |
| `seed` | `payload run` executes a TypeScript file with Payload's config loaded. Use whatever your guide 1 seed script was |

> **Gotcha:** `create-payload-app` often wraps commands as `cross-env NODE_OPTIONS=--no-deprecation next build`.
> That is fine. Keep the wrapper and only add the new scripts. Also, recent Next.js versions removed the
> `next lint` command, which is why `lint` calls `eslint .` directly. If your template still uses `next lint`
> and it works, leave it.

**Check it works:**

```bash
node --version
npm run typecheck && npm run lint && npm test
```

```text
v22.x.x
... no TypeScript errors, no lint errors ...
 Test Files  4 passed (4)
      Tests  12 passed (12)
```

Your test counts will differ. What matters is that all three commands exit without errors, because CI is
about to run exactly these.

**What just happened:** every command CI needs is now a short script that also works on your laptop. When CI
fails, you reproduce it locally with the same command.

### [Intermediate] Step 12 — Add a health route and make Playwright CI-ready

**What we're doing:** adding a tiny `/healthz` URL that says "I am alive", and making sure Playwright tests the
**production** server.

**Why:** load balancers, uptime monitors and deploy smoke tests all need one cheap URL to poll. And end-to-end
tests must run against `next start`, not `next dev`. The dev server compiles pages on demand and hides bugs
that only appear in a production build.

**Do it:** create the health route. A `route.ts` file in the App Router is a **route handler**: it answers HTTP
requests directly with no page or layout.

```ts
// src/app/(frontend)/healthz/route.ts
export const dynamic = 'force-dynamic'

export function GET(): Response {
  return Response.json(
    {
      status: 'ok',
      commit: process.env.GIT_COMMIT_SHA ?? process.env.VERCEL_GIT_COMMIT_SHA ?? 'unknown',
      time: new Date().toISOString(),
    },
    { headers: { 'Cache-Control': 'no-store' } },
  )
}
```

`dynamic = 'force-dynamic'` and `no-store` make sure nobody caches the answer. `commit` tells you which version
is live, which is very handy right after a deploy. Vercel sets `VERCEL_GIT_COMMIT_SHA` itself; our Docker image
will set `GIT_COMMIT_SHA`.

> **Why:** the health route deliberately does **not** query the database. If Postgres has a two-second hiccup,
> you do not want the load balancer to decide every container is dead and restart them all. Liveness ("the
> process answers") and readiness ("the process can do real work") are different questions.

Next, check your `playwright.config.ts` from guide 2. It must start `npm run start` (not `dev`) and must be able
to point at a deployed URL instead. Make it look like this:

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

const PORT = Number(process.env.PORT ?? 3000)
// If E2E_BASE_URL is set, we test an already-deployed site and start nothing locally.
const externalBaseURL = process.env.E2E_BASE_URL
const baseURL = externalBaseURL ?? `http://localhost:${PORT}`

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: Boolean(process.env.CI),
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: process.env.CI ? [['github'], ['html', { open: 'never' }]] : 'list',
  use: {
    baseURL,
    trace: 'on-first-retry',
  },
  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
  webServer: externalBaseURL
    ? undefined
    : {
        command: 'npm run start',
        url: `${baseURL}/healthz`,
        reuseExistingServer: !process.env.CI,
        timeout: 120_000,
      },
})
```

| Setting | Why |
| --- | --- |
| `forbidOnly` in CI | A forgotten `test.only` would silently skip every other test. This fails the run instead |
| `retries: 2` in CI | Browsers on shared machines are occasionally flaky. A test that passes on retry is reported as "flaky" so you still notice |
| `workers: 1` in CI | Tests share one database. Running them one at a time avoids tests stepping on each other's data |
| `reporter` | `github` adds annotations on the PR diff; `html` produces the report we upload |
| `trace: 'on-first-retry'` | Records a full trace (DOM, network, console) when a test fails once. Gold for debugging CI-only failures |
| `webServer.url` | Playwright waits until this URL answers before running tests |

Finally, add a small smoke test file. Keep the detailed tests you wrote in guide 2; these three answer "did the
deploy basically work?" and we will also run them against production.

```ts
// tests/e2e/smoke.spec.ts
import { expect, test } from '@playwright/test'

test('health endpoint responds', async ({ request }) => {
  const response = await request.get('/healthz')
  expect(response.ok()).toBe(true)
  const body: { status: string } = await response.json()
  expect(body.status).toBe('ok')
})

test('home page renders', async ({ page }) => {
  const response = await page.goto('/')
  expect(response?.status()).toBe(200)
  await expect(page.locator('h1').first()).toBeVisible()
})

test('admin panel is served', async ({ page }) => {
  await page.goto('/admin')
  // Payload redirects to the login screen, or to "create first user" on an empty database.
  await expect(page).toHaveURL(/\/admin\/(login|create-first-user)/)
})
```

**Check it works:** with your local Docker Postgres running and seeded:

```bash
npm run build
npm run e2e
```

```text
Running 3 tests using 3 workers
  ✓  1 [chromium] › smoke.spec.ts:3:5 › health endpoint responds (95ms)
  ✓  2 [chromium] › smoke.spec.ts:10:5 › home page renders (640ms)
  ✓  3 [chromium] › smoke.spec.ts:16:5 › admin panel is served (2.4s)
  ... plus your guide 2 tests ...
```

**What just happened:** Playwright built nothing itself. It started your already-built app with `npm run start`,
waited for `/healthz`, ran the tests in a real Chromium browser, then shut the server down. The same config
can test a live URL by setting `E2E_BASE_URL`.

**If it breaks:**

- `Could not find a production build in the '.next' directory`: run `npm run build` first.
- The home page test gets a 404: the database has no Page with slug `home`. Run `npm run seed`.

### [Intermediate] Step 13 — Write the full `ci.yml`

**What we're doing:** writing the CI workflow.

**Why:** this file is the gatekeeper for everything that reaches production.

**Do it:** create `.github/workflows/ci.yml`:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

env:
  NEXT_TELEMETRY_DISABLED: '1'
  DATABASE_URI: postgres://postgres:postgres@localhost:5432/my_site
  PAYLOAD_SECRET: ci-only-secret-not-used-anywhere-else
  NEXT_PUBLIC_SERVER_URL: http://localhost:3000

jobs:
  quality:
    name: Lint, typecheck, unit tests
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Check out code
        uses: actions/checkout@v6

      - name: Set up Node
        uses: actions/setup-node@v6
        with:
          node-version-file: .nvmrc
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Typecheck
        run: npm run typecheck

      - name: Unit tests
        run: npm test

  e2e:
    name: Build and end-to-end tests
    runs-on: ubuntu-latest
    timeout-minutes: 25
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: my_site
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    steps:
      - name: Check out code
        uses: actions/checkout@v6

      - name: Set up Node
        uses: actions/setup-node@v6
        with:
          node-version-file: .nvmrc
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run database migrations
        run: npm run migrate

      - name: Seed test content
        run: npm run seed

      - name: Restore Next.js build cache
        uses: actions/cache@v4
        with:
          path: .next/cache
          key: nextjs-${{ runner.os }}-${{ hashFiles('package-lock.json') }}-${{ hashFiles('src/**') }}
          restore-keys: |
            nextjs-${{ runner.os }}-${{ hashFiles('package-lock.json') }}-

      - name: Build
        run: npm run build

      - name: Install Playwright browser
        run: npx playwright install --with-deps chromium

      - name: End-to-end tests
        run: npm run e2e

      - name: Upload Playwright report
        if: ${{ !cancelled() }}
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

**Check it works:** commit it on a branch and open a PR.

```bash
git switch -c chore/ci
git add .nvmrc package.json playwright.config.ts tests/e2e/smoke.spec.ts \
  "src/app/(frontend)/healthz" .github/workflows/ci.yml
git commit -m "ci: lint, typecheck, unit, build and e2e against Postgres"
git push -u origin chore/ci
gh pr create --fill
gh pr checks --watch
```

```text
Lint, typecheck, unit tests     pass   52s
Build and end-to-end tests      pass   4m08s
```

On the PR page you see both checks with green ticks. Click **Details** on the e2e job and expand "Run database
migrations": you will see Payload applying each file in `src/migrations` to the empty CI database.

**What just happened:** GitHub read the YAML, started two runners in parallel, started a Postgres container
next to the second one, and ran your scripts in order. A failure in any step turns the check red and stops that
job.

**If it breaks:**

- `ECONNREFUSED 127.0.0.1:5432` in the migrate step: the `services:` block is indented wrongly (it must be under
  the job, at the same level as `steps:`), or `ports` is missing.
- `npm ci` fails with "lockfile out of sync": you changed `package.json` without updating
  `package-lock.json`. Run `npm install` locally and commit the lockfile.
- The seed step fails with an error about `revalidatePath` or a "static generation store": your `afterChange`
  hooks call `revalidatePath` outside a Next.js request. Make the hook skip revalidation when
  `req.context.disableRevalidate` is true and pass `context: { disableRevalidate: true }` in the seed's
  `payload.create` calls (this is the pattern Payload's own website template uses).

### [Intermediate] Step 14 — Read the workflow line by line

**What we're doing:** explaining every line, because you should never commit YAML you cannot explain.

**Why:** in interviews and incidents, "I copied it" is not an answer.

| Line | What it does | Why |
| --- | --- | --- |
| `name: CI` | The workflow's display name | Deploy workflows in Part 5 refer to it by this name |
| `on: pull_request` | Run for every PR (against the PR merged with `main`) | Catch problems before merge |
| `on: push: branches: [main]` | Run again after merge | Proves `main` itself is green |
| `concurrency` + `cancel-in-progress` | A new push to the same branch cancels the older run | Saves minutes; stale results are useless |
| `permissions: contents: read` | The automatic `GITHUB_TOKEN` can only read the repo | Least privilege: a compromised dependency cannot push code |
| top-level `env:` | Environment variables for every step of every job | Same names as your `.env`, so scripts work unchanged |
| `PAYLOAD_SECRET: ci-only-...` | A throwaway value written in plain text | CI's database is destroyed after the run, so this protects nothing. Never reuse it elsewhere |
| `runs-on: ubuntu-latest` | Use GitHub's current Ubuntu runner | Clean Linux machine with Docker installed |
| `timeout-minutes` | Kill the job if it hangs | The default is 6 hours of wasted minutes |
| `actions/checkout@v6` | Clone your repo into the runner | Runners start empty |
| `actions/setup-node@v6` + `node-version-file` | Install the Node version in `.nvmrc` | One source of truth for the version |
| `cache: npm` | Cache npm's download folder, keyed on `package-lock.json` | `npm ci` becomes much faster. `node_modules` itself is not cached, on purpose |
| `npm ci` | Clean install exactly from the lockfile | Fails on lockfile drift, unlike `npm install`. Reproducible |
| `services: postgres` | Start `postgres:16` before the steps | A real database, same major version as local and production |
| `POSTGRES_USER/PASSWORD/DB` | The official image creates this user and database on start | Matches `DATABASE_URI` in `env:` |
| `ports: 5432:5432` | Map the container port to the runner | Steps reach it at `localhost:5432` |
| `options: --health-cmd ...` | Docker checks `pg_isready` until Postgres accepts connections | Steps wait for a ready database instead of failing on startup |
| `npm run migrate` | `payload migrate` applies every file in `src/migrations` | Proves migrations run cleanly from zero, as they will on a new environment |
| `npm run seed` | Inserts the home page, posts and other sample content | The e2e tests need content. It runs **before** the build on purpose (see below) |
| `actions/cache` on `.next/cache` | Restore Next.js's compiler cache from earlier runs | Faster builds; Next.js warns in CI without it |
| `hashFiles(...)` in the key | Cache key changes when dependencies or source change | `restore-keys` falls back to the newest partial match |
| `npm run build` | `next build`: compile and pre-render pages | Catches type and build errors only a production build finds |
| `playwright install --with-deps chromium` | Download Chromium plus the Linux libraries it needs | One browser keeps CI fast |
| `npm run e2e` | Playwright starts `next start`, runs the tests | The app is tested as visitors will use it |
| `if: ${{ !cancelled() }}` | Upload even when tests failed, skip if the run was cancelled | You need the report most when something failed |
| `upload-artifact` | Save `playwright-report/` as a downloadable zip for 7 days | Open it locally with `npx playwright show-report` |

Why seed **before** build? `next build` pre-renders pages such as `/` by querying the database. If the database
is empty at that moment, the home page is pre-rendered as a 404 and cached, and the e2e test would see that
404 even after seeding. Seeding first means the build sees real content, just like production.

> **Outdated:** action major versions change roughly once a year. At the time of writing, `actions/checkout`
> and `actions/setup-node` are on v6 and `actions/cache` and `actions/upload-artifact` on v4 or later.
> Dependabot (Part 6) opens PRs when new majors ship.

> **Gotcha:** secrets are not passed to workflows triggered by PRs from forks, and Dependabot PRs read from a
> separate "Dependabot secrets" store. This CI uses no real secrets at all, which is exactly why it works for
> every PR. Keep it that way.

> **Interview tip:** a useful extra check is "did someone change a collection but forget the migration?". Run
> `npx payload migrate:create ci-check --skip-empty` in CI and fail if `git status --porcelain src/migrations`
> is not empty. Test it locally first: if the schema change is ambiguous (a rename), the command asks an
> interactive question, which would hang CI.

### [Beginner] Step 15 — Make the checks required

**What we're doing:** switching on the branch protection planned in Step 9.

**Why:** CI that can be ignored will be ignored. Required checks turn the merge button grey until CI is green.

**Do it:**

1. Repo **Settings > Rules > Rulesets > New ruleset > New branch ruleset** (older UI: **Settings > Branches >
   Add branch protection rule**).
2. Name: `protect-main`. Enforcement: **Active**. Target branches: **Include default branch**.
3. Tick **Restrict deletions**, **Block force pushes**, **Require a pull request before merging** (with
   **Require conversation resolution**), and **Require status checks to pass** with **Require branches to be up
   to date**.
4. Under status checks, add **Lint, typecheck, unit tests** and **Build and end-to-end tests**. These are the
   `name:` values of the two jobs.
5. Leave the **Bypass list** empty. Save.

Working alone? Leave "required approvals" at 0, otherwise you can never merge your own PR. Everything else
still applies.

**Check it works:** try to push straight to `main`.

```bash
git switch main && git pull
git commit --allow-empty -m "test: direct push"
git push
```

```text
remote: error: GH013: Repository rule violations found for refs/heads/main.
remote: - Changes must be made through a pull request.
```

Undo the local commit with `git reset --hard origin/main`. Then merge your `chore/ci` PR with **Squash and
merge**, and watch CI run once more on `main`.

**What just happened:** `main` is now protected by the robot. From here on, every change goes branch, PR, green
CI, merge. That guarantee is what makes the automatic deploys in Parts 4 and 5 safe.

> **Interview tip:** "How do you stop broken code reaching production?" Answer in layers: required CI checks,
> branch protection with no bypass, preview deploys for human review, backward-compatible migrations, a smoke
> test after deploy, and a fast rollback for when all of that still is not enough.

## 4. Path A (recommended for beginners): Vercel + Neon Postgres + Vercel Blob

Vercel is the company behind Next.js, so it runs every Next.js feature with no configuration: server
components, caching and `revalidatePath`, image optimisation, route handlers, server actions. Neon is a
serverless Postgres provider whose killer feature is **branching**: a full copy of your database in about a
second, so every pull request can have its own. Vercel Blob is Vercel's object storage for uploads.

### [Beginner] Step 16 — Concept: what Path A looks like

**What we're doing:** drawing the production system before building it.

**Why:** when something breaks at 11 pm, you need to know which box to look at.

```mermaid
flowchart LR
  Visitor["Visitor or editor"] -->|"HTTPS"| Edge["Vercel edge network<br/>www.my-site.com"]
  Edge --> Fn["Vercel functions<br/>Next.js + Payload"]
  Edge --> Static["Cached pages and<br/>static assets"]
  Fn -->|"pooled connection"| Neon["Neon Postgres<br/>main branch"]
  Fn -->|"upload and read files"| Blob["Vercel Blob<br/>media files"]
  GH["GitHub main"] -->|"push triggers build"| Build["Vercel build<br/>payload migrate, next build"]
  Build -->|"migrate"| Neon
  Build --> Fn
```

Three services, each with one job:

| Piece | Job | Replaces on your laptop |
| --- | --- | --- |
| Vercel | Builds the app on every push, runs it, serves it over HTTPS on a global network | `npm run dev` |
| Neon | Managed Postgres, branchable | The `db` service in `docker-compose.yml` |
| Vercel Blob | Stores uploaded media files | The local `media/` folder |

**What just happened:** you have the map. Notice the build step talks to the database (to run migrations), and
the running functions talk to the database and Blob. Nothing writes to Vercel's own disk.

> **Finance tip:** Vercel's free **Hobby** plan is for personal, **non-commercial** projects. A company or
> client website needs **Pro** (per-member monthly fee). Neon and Vercel Blob both have free allowances that
> fit a small site. Check current pricing pages; see the cost table in Part 6.

### [Beginner] Step 17 — Import the repository into Vercel

**What we're doing:** creating the Vercel project that builds and hosts `my-site`.

**Why:** once connected, Vercel deploys automatically: every push to a PR branch makes a **preview deployment**,
every push to `main` makes a **production deployment**. That is the "CD" half of CI/CD, done for you.

**Do it:**

1. Sign in at vercel.com with GitHub. Click **Add New > Project** and **Import** the `my-site` repo. Grant the
   Vercel GitHub app access to that repo only.
2. **Framework Preset:** Next.js (detected automatically).
3. Open **Build and Output Settings** and override **Build Command** to:

   ```bash
   npm run build:deploy
   ```

   Leave Install Command (`npm install`, Vercel detects the lockfile) and Output Directory as default.
4. Open **Environment Variables** and add one for now: `PAYLOAD_SECRET` with a fresh value from
   `openssl rand -hex 32`. (We will scope it per environment in Step 20.)
5. Click **Deploy**.

**Check it works:** the first deployment **fails**, and that is expected. Open the build log:

```text
> my-site@1.0.0 build:deploy
> payload migrate && next build
Error: cannot connect to Postgres. Details: connect ECONNREFUSED 127.0.0.1:5432
Error: Command "npm run build:deploy" exited with 1
```

There is no database yet, so the build fell back to an empty connection string. Good: it proves migrations run
during the build, and that a broken migration stops the deploy instead of shipping half-migrated code.

**What just happened:** Vercel cloned your repo, installed dependencies, ran your build command and stopped at
the first failure. The live site is unchanged (there isn't one yet). Every future deploy follows this same
"build, and only switch traffic if the build succeeded" rule.

> **Gotcha:** Vercel reads the Node version from `engines.node` in `package.json` (Step 11) or the project
> setting **Settings > Build and Deployment > Node.js Version**. Make sure it matches `.nvmrc`.

### [Beginner] Step 18 — Create the Neon database and understand pooled connections

**What we're doing:** creating the production Postgres and connecting it to the Vercel project.

**Why:** the app needs an always-on database. Using the Vercel integration means Vercel receives the connection
string automatically, and (Step 22) creates a database branch for each preview.

**Concept: connection pooling.** Every Postgres connection uses memory on the server, so Postgres allows only a
limited number at once. Serverless functions are a problem here: a traffic spike can start 100 function
instances, each opening its own connections, and Postgres runs out. A **connection pooler** (Neon uses
PgBouncer) sits in front of Postgres, accepts thousands of client connections and shares a small set of real
ones between them.

```mermaid
flowchart LR
  F1["Function 1"] --> Pool["Neon pooler<br/>host has -pooler"]
  F2["Function 2"] --> Pool
  F3["Function 50"] --> Pool
  Pool -->|"few real connections"| PG["Postgres"]
  Mig["payload migrate<br/>during build"] -->|"pooled or direct"| PG
```

Neon gives you two strings for the same database:

```text
Pooled (use for the app):
postgresql://neondb_owner:abc123@ep-cool-sun-a1b2c3-pooler.us-east-1.aws.neon.tech/neondb?sslmode=require
Direct (unpooled):
postgresql://neondb_owner:abc123@ep-cool-sun-a1b2c3.us-east-1.aws.neon.tech/neondb?sslmode=require
```

The only difference is `-pooler` in the host name.

**Do it:**

1. In your Vercel project, open the **Storage** tab, click **Create Database** (or **Browse Marketplace**) and
   choose **Neon**.
2. Region: pick the one closest to where your Vercel functions run (by default Washington, D.C., `iad1`, which
   is next to AWS `us-east-1`). Database and app in the same region saves 10 to 80 ms per query.
3. Postgres version: choose **16** if offered, to match your local Docker and CI. (17 also works; then update
   `docker-compose.yml` and `ci.yml` to `postgres:17` so all environments match.)
4. Connect it to the project for **Production** and **Preview** environments. Leave **Development** off; your
   laptop keeps using Docker.
5. If the integration offers **"Create a database branch for each preview deployment"** (wording varies), turn
   it on. Also turn on **automatic deletion of obsolete branches** if offered.

The integration adds environment variables to the project, including `DATABASE_URL` (pooled) and
`DATABASE_URL_UNPOOLED` (direct), plus some legacy `PG*` and `POSTGRES_*` names.

> **Outdated:** Neon offers two integrations: one created from Vercel's Marketplace (billed through Vercel) and
> a "Neon-managed" one installed from the Neon console (billed by Neon). Both inject `DATABASE_URL`. Preview
> branching settings live in slightly different screens and have changed over time. If you cannot find the
> branch-per-preview option in one, check Neon's "Vercel integration" docs. You cannot use both integrations
> on the same Vercel project.

Now make the Payload config read Neon's variable name. Your local `.env` uses `DATABASE_URI`; Vercel provides
`DATABASE_URL`. Instead of copying values by hand (and forgetting to update them for each preview branch), let
the config accept either. Open `src/payload.config.ts` and change the `db` block:

```ts
// src/payload.config.ts (snippet: replace the existing db: postgresAdapter({...}) block)
  db: postgresAdapter({
    pool: {
      // Local and AWS set DATABASE_URI. The Neon + Vercel integration sets DATABASE_URL.
      connectionString: process.env.DATABASE_URI || process.env.DATABASE_URL || '',
    },
  }),
```

**Check it works:** in Vercel, **Settings > Environment Variables** lists `DATABASE_URL` for Production and
Preview. In the Neon console you see a project with a `main` branch (sometimes called `production`) and an
empty `neondb` database. Do not redeploy yet; Blob comes next.

**What just happened:** you created a managed Postgres and Vercel now injects its pooled connection string into
both builds and running functions. Payload's Postgres adapter uses the `pg` driver, which works through the
pooler for normal queries.

> **Gotcha:** if `payload migrate` ever fails during a Vercel build with an error mentioning prepared
> statements, advisory locks or the pooler, run migrations over the direct connection instead: change the
> Build Command to `DATABASE_URI=$DATABASE_URL_UNPOOLED npm run build:deploy`. Because `DATABASE_URI` wins in
> the config, migrate and build then use the direct host while the running app keeps the pooled one.

### [Intermediate] Step 19 — Store media in Vercel Blob

**What we're doing:** creating a Blob store and telling Payload to upload `media` files there.

**Why:** Step 4: function disks are temporary. Without this, an editor's upload would appear to work and then
404 minutes later.

**Do it:** in the Vercel project, **Storage > Create Database > Blob**. Name it `my-site-media` and connect it
to **Production** and **Preview**. Vercel adds `BLOB_READ_WRITE_TOKEN` to the project.

Install the adapter on a new branch:

```bash
git switch main && git pull
git switch -c feat/production-storage
npm install @payloadcms/storage-vercel-blob
```

> **Gotcha:** all `@payloadcms/*` packages must be on the **same version**. If `npm install` picks a newer
> version than your `payload` package, install a matching one: `npm install @payloadcms/storage-vercel-blob@<your payload version>`
> (check with `npm ls payload`), or upgrade all Payload packages together.

Now create a small helper for URLs. Preview deployments have a different URL for every deploy, so we cannot
hard-code it.

```ts
// src/lib/serverUrl.ts
/**
 * The public base URL of the environment we are running in.
 * - Local and production: NEXT_PUBLIC_SERVER_URL from the environment.
 * - Vercel previews: we leave NEXT_PUBLIC_SERVER_URL unset and use the URL Vercel provides.
 */
export function getServerUrl(): string {
  if (process.env.NEXT_PUBLIC_SERVER_URL) return process.env.NEXT_PUBLIC_SERVER_URL
  if (process.env.VERCEL_URL) return `https://${process.env.VERCEL_URL}`
  return 'http://localhost:3000'
}

/** Every origin that may legitimately call the Payload API (used for CORS and CSRF). */
export function getAllowedOrigins(): string[] {
  const origins = [getServerUrl()]
  if (process.env.VERCEL_URL) origins.push(`https://${process.env.VERCEL_URL}`)
  if (process.env.VERCEL_BRANCH_URL) origins.push(`https://${process.env.VERCEL_BRANCH_URL}`)
  return Array.from(new Set(origins))
}
```

`VERCEL_URL` (this deployment's unique URL) and `VERCEL_BRANCH_URL` (a stable URL for the Git branch) are
**system environment variables** Vercel sets automatically.

Here is the whole `src/payload.config.ts` after this step. Your collection imports should already look like
this from guide 1; the new parts are marked `NEW`.

```ts
// src/payload.config.ts
import path from 'path'
import { fileURLToPath } from 'url'
import { buildConfig } from 'payload'
import { postgresAdapter } from '@payloadcms/db-postgres'
import { lexicalEditor } from '@payloadcms/richtext-lexical'
import { vercelBlobStorage } from '@payloadcms/storage-vercel-blob' // NEW
import sharp from 'sharp'

import { Users } from './collections/Users'
import { Media } from './collections/Media'
import { Pages } from './collections/Pages'
import { Posts } from './collections/Posts'
import { ContactSubmissions } from './collections/ContactSubmissions'
import { getAllowedOrigins, getServerUrl } from './lib/serverUrl' // NEW

const filename = fileURLToPath(import.meta.url)
const dirname = path.dirname(filename)

export default buildConfig({
  serverURL: getServerUrl(), // NEW
  cors: getAllowedOrigins(), // NEW
  csrf: getAllowedOrigins(), // NEW
  admin: {
    user: Users.slug,
    importMap: { baseDir: path.resolve(dirname) },
  },
  collections: [Users, Media, Pages, Posts, ContactSubmissions],
  editor: lexicalEditor(),
  secret: process.env.PAYLOAD_SECRET || '',
  typescript: {
    outputFile: path.resolve(dirname, 'payload-types.ts'),
  },
  db: postgresAdapter({
    pool: {
      connectionString: process.env.DATABASE_URI || process.env.DATABASE_URL || '', // NEW fallback
    },
  }),
  sharp,
  plugins: [
    // NEW: in production and previews, uploads go to Vercel Blob.
    // Locally the token is empty, the adapter is disabled, and files go to ./media as before.
    vercelBlobStorage({
      enabled: Boolean(process.env.BLOB_READ_WRITE_TOKEN),
      collections: { media: true },
      token: process.env.BLOB_READ_WRITE_TOKEN || '',
      clientUploads: true,
    }),
  ],
})
```

| Option | Why |
| --- | --- |
| `enabled` | Switch the adapter on only where a token exists, so local development needs no Blob account |
| `collections: { media: true }` | Use Blob for the `media` collection (its `slug` is `media`) |
| `token` | The read/write token Vercel injected |
| `clientUploads: true` | The admin panel uploads the file **directly from the browser to Blob**. Vercel functions reject request bodies larger than about 4.5 MB, so without this, large images fail to upload |

The plugin automatically turns off local disk storage for `media` when enabled. Payload still records each
file in the `media` table (filename, size, alt text); only the bytes live in Blob. Payload serves file URLs
through its own `/api/media/file/...` route by default, so access control on `media` still applies, and
`next/image` keeps working with no extra `remotePatterns` config.

Regenerate the admin import map (storage adapters add an admin component for client uploads), then type-check:

```bash
npm run generate:importmap
npm run typecheck && npm run build
```

**Check it works:** commit and open a PR. CI should be green, because in CI there is no Blob token and the
adapter is disabled.

```bash
git add -A
git commit -m "feat: production config for Neon and Vercel Blob"
git push -u origin feat/production-storage
gh pr create --fill
```

**What just happened:** the same code now behaves correctly in three environments: local disk on your laptop,
no storage in CI, Blob on Vercel. That "one codebase, behaviour chosen by environment variables" idea is the
Twelve-Factor App principle "store config in the environment".

**If it breaks:**

- Admin shows a blank upload field or an import map error: you skipped `npm run generate:importmap`.
- `Cannot find module '@payloadcms/storage-vercel-blob'` on Vercel only: you installed it but did not commit
  `package-lock.json`.

### [Beginner] Step 20 — Set environment variables per environment

**What we're doing:** giving each environment exactly the settings it needs, and no more.

**Why:** a preview that holds the production `PAYLOAD_SECRET` can mint production admin tokens. A production
deploy with a missing variable crashes on its first request.

Vercel scopes every variable to one or more of **Production**, **Preview** and **Development**. Set them in
**Settings > Environment Variables**, or from your terminal with the Vercel CLI (`npm i -g vercel`, then
`vercel link` inside `my-site`):

```bash
# a different secret for each environment
openssl rand -hex 32 | vercel env add PAYLOAD_SECRET production
openssl rand -hex 32 | vercel env add PAYLOAD_SECRET preview
echo "https://www.my-site.com" | vercel env add NEXT_PUBLIC_SERVER_URL production
vercel env ls
```

Remove the unscoped `PAYLOAD_SECRET` you added in Step 17 if it applies to all environments.

The final picture:

| Variable | Production | Preview | Set by |
| --- | --- | --- | --- |
| `DATABASE_URL` | Neon `main` branch (pooled) | That preview's Neon branch | Neon integration |
| `DATABASE_URL_UNPOOLED` | Direct string for `main` | Direct string for the branch | Neon integration |
| `BLOB_READ_WRITE_TOKEN` | Blob store token | Same store (see Gotcha) | Blob store connection |
| `PAYLOAD_SECRET` | Random value A | Random value B | You |
| `NEXT_PUBLIC_SERVER_URL` | `https://www.my-site.com` (until Step 23, your `my-site.vercel.app` URL) | **Not set**, so `getServerUrl()` uses `VERCEL_URL` | You |
| `VERCEL_URL`, `VERCEL_ENV`, `VERCEL_GIT_COMMIT_SHA` | Automatic | Automatic | Vercel system variables |

> **Gotcha:** previews and production share one Blob store unless you create a second store for Preview. That
> is usually fine because file names are unique and the database decides which files a page uses. If you want
> strict separation, create `my-site-media-preview` and connect it to Preview only.

> **Gotcha:** environment variable changes apply to **new** deployments only. After editing variables, redeploy
> (Deployments tab, three dots, **Redeploy**).

**Check it works:** `vercel env ls` shows `PAYLOAD_SECRET` twice (production and preview) with different
"created" times, and `NEXT_PUBLIC_SERVER_URL` only for production.

**What just happened:** each environment is now isolated: its own secret, its own database, its own URL.

### [Beginner] Step 21 — Concept: how migrations run automatically during deploy

**What we're doing:** understanding the `build:deploy` script before trusting it with production.

**Why:** a migration changes the shape of the production database. You must know exactly when it runs and what
happens if it fails.

In guide 1 you learned that a **migration** is a file in `src/migrations` describing a schema change ("add
column `excerpt` to `posts`"), and that `payload migrate` applies every migration the database has not seen
yet, recording each one in a `payload_migrations` table.

Our Vercel Build Command is `payload migrate && next build`:

```mermaid
sequenceDiagram
  participant GH as GitHub
  participant V as Vercel build
  participant DB as Neon database
  participant Live as Live site
  GH->>V: push to main
  V->>DB: payload migrate
  DB-->>V: applied 1 new migration
  V->>DB: next build reads content to pre-render
  V->>V: next build succeeds
  V->>Live: switch traffic to the new deployment
  Note over Live: old deployment kept for rollback
```

Key facts:

- `&&` means "only build if migrate succeeded". A failed migration fails the deploy and the old version keeps
  serving.
- Migrations run **before** the new code is live. For a short moment the **old** code runs against the **new**
  schema. Your migrations must therefore be **backward compatible** (Step 25 shows how).
- In production, Payload never auto-changes the schema. The "push" mode that silently synced tables in
  development is off when `NODE_ENV` is `production`. That is why committed migrations are mandatory.

Payload also offers `prodMigrations`: you pass the migrations array to `postgresAdapter` and Payload runs them
when it starts up. Payload's docs recommend that for long-running servers, **not** for serverless platforms
like Vercel, where it would slow down cold starts. Running `payload migrate` in the build is the recommended
approach for Vercel.

**What just happened:** you know the deploy order and its one sharp edge (old code, new schema). Now you can
deploy.

### [Beginner] Step 22 — Deploy production and create the first admin user

**What we're doing:** merging the storage PR, letting Vercel deploy production, and claiming the admin account.

**Why:** an empty Payload installation lets **the first visitor** to `/admin` create the first admin user.
That visitor must be you, immediately.

**Do it:**

1. Wait for CI on the `feat/production-storage` PR to go green, then **Squash and merge**.
2. Vercel starts a production build. Watch it in the Vercel **Deployments** tab.
3. As soon as it is **Ready**, open `https://<your-project>.vercel.app/admin` and fill in the **Create First
   User** form with your real email and a long password from a password manager.
4. In the admin, create the content the site needs: at least a **Page** with slug `home`, a couple of posts and
   images. (Do not run `npm run seed` against production: sample content does not belong there.)

**Check it works:**

```bash
curl -s https://<your-project>.vercel.app/healthz
```

```text
{"status":"ok","commit":"9f8e7d6c...","time":"2026-10-06T10:12:44.120Z"}
```

The `commit` matches the merge commit on `main`. In the build log you should also see:

```text
> payload migrate && next build
[10:10:02] INFO: Migrating: 20261001_120000_initial
[10:10:03] INFO: Migrated:  20261001_120000_initial (812ms)
[10:10:03] INFO: Done.
   ▲ Next.js 16.x
   Creating an optimized production build ...
```

Upload an image in the admin, then open **Storage > my-site-media** in Vercel: the file is there. Open the
image URL in a private window: it loads.

Then run your smoke tests against production:

```bash
E2E_BASE_URL=https://<your-project>.vercel.app npx playwright test tests/e2e/smoke.spec.ts
```

```text
  3 passed (4.1s)
```

**What just happened:** the build migrated an empty Neon database from zero, built the site and switched it
live. Payload saw no users and offered the first-user form, which you claimed. Media now flows to Blob.

**If it breaks:**

- Admin login loops back to the login page: `PAYLOAD_SECRET` changed between the build and runtime, or
  `NEXT_PUBLIC_SERVER_URL` does not match the URL in the address bar. Fix the variable and redeploy.
- `/` returns 404: there is no Page with slug `home` yet. Create it. The `afterChange` hook calls
  `revalidatePath('/')`, so it appears without a redeploy.
- Image uploads fail around 4.5 MB: `clientUploads: true` is missing.

### [Intermediate] Step 23 — Preview deployments with their own database branch

**What we're doing:** seeing the most useful feature of Path A: every PR gets a live copy of the site **and** a
private copy of the database.

**Why:** reviewing a PR by reading code is good. Clicking around a running copy, with real-looking content,
after its migration has been applied, is much better. And because the copy has its own database branch, a bad
migration in a PR can never touch production.

**Concept: database branching.** A Neon **branch** is a copy-on-write clone of a database at a moment in time.
It starts identical to its parent but shares the storage underneath, so it is created in seconds and costs
almost nothing until it diverges. Writes to the branch never affect the parent.

```mermaid
sequenceDiagram
  participant Dev as You
  participant GH as GitHub
  participant V as Vercel
  participant N as Neon
  Dev->>GH: push branch feat/post-excerpt and open PR
  GH->>V: webhook for new commit
  V->>N: integration creates branch preview/feat/post-excerpt from main
  N-->>V: DATABASE_URL for the new branch
  V->>N: payload migrate on the branch only
  V->>V: next build
  V-->>GH: comment with the preview URL
  Dev->>GH: merge PR and delete git branch
  GH->>V: deploy production from main
  V->>N: payload migrate on main
  N->>N: delete the obsolete preview branch
```

**Do it:** make a real schema change to watch it work. Add an optional `excerpt` field to posts.

```bash
git switch main && git pull
git switch -c feat/post-excerpt
```

```ts
// src/collections/Posts.ts (snippet: add this object to the fields array)
    {
      name: 'excerpt',
      type: 'textarea',
      required: false,
      admin: { description: 'One or two sentences shown on the blog index.' },
    },
```

Create the migration while your local Docker Postgres is running:

```bash
npm run migrate:create -- add-post-excerpt
npm run migrate
git add -A && git commit -m "feat: add optional excerpt to posts"
git push -u origin feat/post-excerpt
gh pr create --fill
```

**Check it works:**

1. CI runs, and in the e2e job the migrate step now applies your new migration to an empty database.
2. The Vercel bot comments on the PR with a **Preview** link.
3. In the Neon console, **Branches** shows a new branch named after your git branch (for example
   `preview/feat/post-excerpt`).
4. Open the preview, go to `/admin`, log in with your production account (the branch copied the users table),
   and edit a post: the new **Excerpt** field is there.
5. Open production `/admin`: no Excerpt field. Production's database is untouched.

Merge the PR. Production builds, `payload migrate` adds the column to `main`, and the preview branch is deleted
(if you enabled automatic cleanup).

**What just happened:** you tested a schema change against a full copy of production data without risk. This
workflow is the main reason Path A is recommended for beginners.

> **Gotcha:** a branch of production contains production data, including **contact form submissions** (names,
> emails, messages) and admin users. Anyone who can open the preview can see what the data allows. Keep
> Vercel's **Deployment Protection** on for previews (the default requires a Vercel login), and if your data is
> sensitive, branch previews from a separate `staging` Neon branch with fake data instead of from `main`
> (an integration setting).

> **Gotcha:** previews only get a branch if the integration's branch-per-preview option is on. Without it,
> Preview deployments get the **production** `DATABASE_URL`, and a preview build would run your PR's
> migration against production. Check the Preview value of `DATABASE_URL` before your first schema-changing PR.

### [Intermediate] Step 24 — Add a custom domain with HTTPS

**What we're doing:** serving the site at `www.my-site.com` instead of `my-site.vercel.app`.

**Why:** a real domain is part of being in production. HTTPS (the padlock) is mandatory: browsers mark plain
HTTP as "Not secure", and Payload's login cookies should never travel unencrypted.

**Concept: DNS.** The **Domain Name System** is the internet's phone book. A **record** maps a name to a
destination. An **A record** maps a name to an IP address. A **CNAME record** maps a name to another name. The
**apex** domain is the bare `my-site.com`; `www.my-site.com` is a subdomain.

**Do it:**

1. Buy the domain from any registrar.
2. Vercel project **Settings > Domains > Add**, type `www.my-site.com`. Accept the suggestion to also add
   `my-site.com` and redirect it to `www`.
3. Vercel shows the records to create. They usually look like this (always copy the exact values Vercel shows
   you, as they can differ per project):

```text
Type   Name   Value
A      @      76.76.21.21
CNAME  www    cname.vercel-dns.com
```

4. Add them at your registrar's DNS settings. DNS changes take minutes to a few hours to spread.
5. Update `NEXT_PUBLIC_SERVER_URL` for Production to `https://www.my-site.com` and **redeploy** (it is baked in
   at build time).

**Check it works:**

```bash
dig +short www.my-site.com
curl -sI https://my-site.com | grep -i location
curl -s https://www.my-site.com/healthz
```

```text
cname.vercel-dns.com.
76.76.21.21
location: https://www.my-site.com/
{"status":"ok","commit":"...","time":"..."}
```

The Domains page shows a green **Valid Configuration** tick for both names.

**What just happened:** once DNS pointed at Vercel, Vercel proved you control the domain and got a free TLS
certificate from Let's Encrypt for it, which it renews automatically. HTTP requests are redirected to HTTPS.

### [Intermediate] Step 25 — Roll back safely

**What we're doing:** undoing a bad deploy in seconds, and understanding what rollback does **not** undo.

**Why:** every team ships a bug eventually. What separates a five-minute incident from a five-hour one is a
practised rollback.

**Do it:** in Vercel, **Deployments**, find the last good production deployment, open the three-dots menu and
choose **Instant Rollback** (or **Promote to Production**). With the CLI:

```bash
vercel ls my-site --prod          # list recent production deployments
vercel rollback <deployment-url>  # point the domain back at that deployment
```

Rollback does not rebuild anything. Vercel keeps every deployment and simply points your domain back at the old
one, so it takes seconds.

> **Outdated:** on the Hobby plan, Instant Rollback has historically been limited to the previous production
> deployment; Pro can go further back. Check Vercel's current docs.

> **Gotcha:** after a rollback, Vercel may pause automatic production deploys for that project so the next
> push does not immediately undo your rollback. Fix the bug with a normal PR, then re-enable or promote the
> fixed deployment.

**The big caveat: rollback does not undo migrations.** The old code now runs against the newer schema. This is
safe only if every migration is **backward compatible**. The standard technique is **expand and contract**:

```mermaid
stateDiagram-v2
  [*] --> Expand
  Expand: PR 1 expand - add new column as optional
  Expand --> Migrate
  Migrate: PR 2 code writes both and backfills data
  Migrate --> Switch
  Switch: PR 3 code reads only the new column
  Switch --> Contract
  Contract: PR 4 contract - drop the old column
  Contract --> [*]
```

Each PR is safe to roll back on its own, because the schema always supports both the current code and the
previous code. Rules of thumb:

- Adding an optional field: safe in one PR.
- Renaming a field: never in one step. Add new, copy, switch, then remove old.
- Making a field required: first fill every existing row, then make it required in a later PR.
- Deleting a field or collection: only after a release where no code uses it any more.

If data itself is damaged (for example a migration deleted rows), use Neon's **point-in-time restore**: Part 6
covers backups.

**Check it works:** practise once. Promote an older deployment, confirm `/healthz` shows the older commit, then
promote the newest one again.

**What just happened:** you can undo code in seconds and you know how to write migrations so that undoing code
is always safe.

> **Interview tip:** "How do you roll back a deploy that included a database migration?" You roll back the code,
> not the schema. Migrations are written expand-and-contract so the previous version still works with the new
> schema. Down migrations exist (`payload migrate:down`) but running them in production risks data loss, so
> they are a last resort.

## 5. Path B: AWS with Docker, ECS, RDS and S3

Choose this path when your company already runs on AWS, needs everything inside its own AWS account (security
reviews, private networking, one bill), or wants to avoid depending on a hosting platform. The price is more
pieces to own: the container, the network, the database server, the certificates. Everything here is also
classic interview material.

> **Outdated:** older tutorials use **AWS App Runner** for "just run my container". App Runner is closed to new
> customers (existing customers can keep using it). AWS points new users to **Amazon ECS Express Mode**: you
> give it a container image and two IAM roles, and it creates an ECS Fargate service, a load balancer with
> HTTPS, autoscaling and logs. Express Mode launched in late 2025, so its CLI flags may still change; each
> command below says what it does so you can map it to the current docs.

### [Beginner] Step 26 — Concept: the AWS pieces and how they fit

**What we're doing:** learning the vocabulary and drawing the system.

**Why:** AWS has a name for everything. Without the map, the commands look like noise.

| AWS term | Plain English | Path A equivalent |
| --- | --- | --- |
| **Docker image** | A frozen package of your app plus Node plus OS files. Runs the same anywhere | Vercel's build output |
| **Container** | A running instance of an image | A Vercel function instance |
| **ECR** (Elastic Container Registry) | Private storage for your images | (hidden inside Vercel) |
| **ECS** (Elastic Container Service) | Runs and restarts containers for you | Vercel's runtime |
| **Fargate** | ECS mode where AWS owns the servers; you only say "1 vCPU, 2 GB" | (same idea) |
| **Task definition** / **task** | The recipe for a container (image, CPU, env, secrets) / one running copy | Project settings / one instance |
| **Service** | Keeps N tasks running and replaces unhealthy ones | Production deployment |
| **ALB** (Application Load Balancer) | Receives HTTPS traffic and spreads it across healthy tasks | Vercel's edge |
| **RDS** | Managed Postgres server | Neon |
| **S3** | Object storage | Vercel Blob |
| **Secrets Manager** | Encrypted store for secrets, injected into containers at start | Vercel env vars |
| **IAM role** | An identity with permissions that a service (or GitHub) can temporarily become | (no equivalent) |
| **VPC** / **security group** | Your private network / a firewall around one resource | (hidden) |
| **CloudWatch Logs** | Where container output goes | Vercel logs |

```mermaid
flowchart LR
  User["Visitor"] -->|"HTTPS"| R53["Route 53<br/>www.my-site.com"]
  R53 --> ALB["Load balancer<br/>ACM certificate"]
  ALB --> Task["ECS Fargate task<br/>my-site container"]
  Task -->|"port 5432, app SG only"| RDS["RDS Postgres 16"]
  Task -->|"task role"| S3["S3 bucket<br/>media"]
  Task --> CW["CloudWatch Logs"]
  SM["Secrets Manager"] -->|"injected at start"| Task
  GHA["GitHub Actions"] -->|"OIDC, push image"| ECR["ECR my-site"]
  ECR --> Task
  GHA -->|"one-off migrate task"| RDS
```

**What just happened:** you can now read every box. The database accepts connections only from the app's
security group, the app reaches S3 through its IAM role (no keys), and GitHub reaches AWS through OIDC (no keys
either).

> **Finance tip:** unlike Path A, most of these pieces cost money every hour even with zero visitors: the RDS
> instance, the Fargate task, the load balancer and public IPv4 addresses. Expect tens of dollars per month.
> See the cost table in Part 6, and delete everything when you finish practising.

### [Intermediate] Step 27 — Write a multi-stage, standalone, non-root Dockerfile

**What we're doing:** packaging `my-site` into a small, secure Docker image, plus a second image that runs
migrations.

**Why:** a container is what ECS runs. A naive image (copy everything, `npm install`, `npm start`) is over 1 GB,
runs as root and contains your dev tools. We want a small image with only what production needs.

**Concept: three ideas in one file.**

- **Multi-stage build:** one Dockerfile with several `FROM` stages. Early stages install and build; the final
  stage copies only the results. Compilers, dev dependencies and source code never reach the final image.
- **Standalone output:** with `output: 'standalone'`, `next build` traces exactly which files in
  `node_modules` the server needs and writes them, plus a tiny `server.js`, to `.next/standalone`. No full
  `node_modules` needed.
- **Non-root:** the app runs as an unprivileged user, so a bug that lets an attacker run code does not give
  them root inside the container.

**Do it:** first, enable standalone output only for Docker builds. Open `next.config.mjs` (it may be
`next.config.ts` in your project; the change is the same):

```js
// next.config.mjs
import { withPayload } from '@payloadcms/next/withPayload'

/** @type {import('next').NextConfig} */
const nextConfig = {
  // Docker builds set BUILD_STANDALONE=true. Vercel and local builds are unaffected.
  output: process.env.BUILD_STANDALONE === 'true' ? 'standalone' : undefined,
  // keep any other options you already had here
}

export default withPayload(nextConfig, { devBundleServerPackages: false })
```

Make sure a `public` folder exists (the Dockerfile copies it):

```bash
mkdir -p public && touch public/.gitkeep
```

**Concept: building without a database.** `next build` normally pre-renders pages, and our pages query Payload,
so a normal build needs a database. A Docker build cannot reach your private RDS. Payload's docs give two
options: mark pages `dynamic = 'force-dynamic'` (slower site), or build with
`next build --experimental-build-mode compile`, which compiles everything but skips pre-rendering. Pages are
then rendered on their first request and cached as usual. That is our `build:docker` script from Step 11.

Now the Dockerfile:

```dockerfile
# Dockerfile
# syntax=docker/dockerfile:1
ARG NODE_VERSION=22

# ---- base: shared starting point --------------------------------------
FROM node:${NODE_VERSION}-alpine AS base
RUN apk add --no-cache libc6-compat
WORKDIR /app
ENV NEXT_TELEMETRY_DISABLED=1

# ---- deps: install exactly what the lockfile says ---------------------
FROM base AS deps
COPY package.json package-lock.json ./
RUN npm ci

# ---- builder: compile the Next.js + Payload app ----------------------
FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ARG NEXT_PUBLIC_SERVER_URL
ARG NEXT_PUBLIC_SENTRY_DSN=""
ENV NEXT_PUBLIC_SERVER_URL=${NEXT_PUBLIC_SERVER_URL} \
    NEXT_PUBLIC_SENTRY_DSN=${NEXT_PUBLIC_SENTRY_DSN} \
    BUILD_STANDALONE=true
RUN npm run build:docker

# ---- migrator: full source + node_modules, runs "payload migrate" ----
FROM base AS migrator
ENV NODE_ENV=production
COPY --from=deps --chown=node:node /app/node_modules ./node_modules
COPY --chown=node:node . .
USER node
CMD ["npm", "run", "migrate"]

# ---- runner: the small production image (last stage = default) ------
FROM base AS runner
ARG GIT_COMMIT_SHA=unknown
ENV NODE_ENV=production \
    PORT=3000 \
    HOSTNAME=0.0.0.0 \
    GIT_COMMIT_SHA=${GIT_COMMIT_SHA}
RUN addgroup --system --gid 1001 nodejs && adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
RUN mkdir -p media && chown nextjs:nodejs media
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

Keep junk and secrets out of the **build context** (the files Docker can see):

```text
# .dockerignore
node_modules
.next
.git
.github
.env
.env*.local
coverage
playwright-report
test-results
media
Dockerfile
.dockerignore
```

Line by line:

| Line | Why |
| --- | --- |
| `node:22-alpine` | Alpine Linux is tiny. `libc6-compat` helps some native packages (such as `sharp`) run on it |
| `COPY package.json package-lock.json` then `npm ci` | Docker caches each step. Dependencies are reinstalled only when these two files change, not on every code edit |
| `ARG NEXT_PUBLIC_SERVER_URL` | Public variables are compiled in at build time (Step 7), so they must be build arguments |
| `BUILD_STANDALONE=true` | Turns on `output: 'standalone'` in `next.config.mjs` |
| `npm run build:docker` | Compile without pre-rendering, so no database is needed during the build |
| `migrator` stage | `payload migrate` needs the Payload CLI, your config and `src/migrations`, which standalone output does not include. So migrations get their own image built from the same commit |
| `HOSTNAME=0.0.0.0` | `server.js` must listen on all network interfaces, or nothing outside the container can reach it |
| `adduser ... nextjs` + `USER nextjs` | Run as a non-root user |
| `--chown=nextjs:nodejs` | The server writes its page cache under `.next`; it needs permission |
| copy `public` and `.next/static` | Standalone output leaves these out. Without them, CSS, JS and images return 404 |
| `mkdir media` | Only used when S3 is not configured (the local compose test). In AWS, media goes to S3 |
| `.dockerignore` has `.env` | Your real secrets must never be copied into an image layer |

**Check it works:**

```bash
docker build --target runner --build-arg NEXT_PUBLIC_SERVER_URL=http://localhost:3000 -t my-site:local .
docker build --target migrator -t my-site:migrate-local .
docker images my-site --format "{{.Tag}}  {{.Size}}"
docker run --rm --entrypoint whoami my-site:local
```

```text
local          ~300MB
migrate-local  ~900MB
nextjs
```

Sizes vary. The runner image should be a few hundred MB. If it is over 1 GB, standalone output is not enabled.
The migrator is large because it has all of `node_modules`; that is fine, it only runs for seconds per deploy.

**What just happened:** one Dockerfile produced two images from the same commit: a small, non-root web server
and a one-shot migration runner.

**If it breaks:**

- `COPY failed: ... .next/standalone: not found`: `BUILD_STANDALONE` did not reach `next.config.mjs`. Check the
  `output:` line.
- The build fails trying to connect to Postgres: a page still forces pre-rendering. Check that `build:docker`
  uses `--experimental-build-mode compile`. As a fallback, start the compose database from Step 28 and build
  with `docker build --network host ...` and `DATABASE_URI` pointing at it.
- Errors about `sharp` on Alpine: switch both `FROM` lines to `node:22-bookworm-slim` and remove the `apk` line.

### [Beginner] Step 28 — Test a production-like stack locally with Docker Compose

**What we're doing:** running the real production images, a fresh Postgres and the migration step together on
your laptop.

**Why:** debugging a container on your laptop takes seconds; debugging it on AWS takes coffee breaks. If it
works here, the remaining AWS problems are networking and permissions only.

**Do it:** create a separate compose file so it does not clash with your dev database:

```yaml
# docker-compose.prod.yml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: my_site
    volumes:
      - prodlike-db:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U postgres']
      interval: 5s
      retries: 10

  migrate:
    build:
      context: .
      target: migrator
    environment:
      DATABASE_URI: postgres://postgres:postgres@db:5432/my_site
      PAYLOAD_SECRET: local-prodlike-secret
    depends_on:
      db:
        condition: service_healthy

  app:
    build:
      context: .
      target: runner
      args:
        NEXT_PUBLIC_SERVER_URL: http://localhost:3000
        GIT_COMMIT_SHA: local
    environment:
      DATABASE_URI: postgres://postgres:postgres@db:5432/my_site
      PAYLOAD_SECRET: local-prodlike-secret
    ports:
      - '3000:3000'
    depends_on:
      migrate:
        condition: service_completed_successfully

volumes:
  prodlike-db:
```

Inside a compose network, services reach each other by service name, so the database host is `db`, not
`localhost`. `service_completed_successfully` means "start the app only after migrations exited with code 0",
which is exactly the order we want in AWS.

Stop `npm run dev` (it also uses port 3000), then:

```bash
docker compose -f docker-compose.prod.yml up --build -d
docker compose -f docker-compose.prod.yml logs migrate
docker compose -f docker-compose.prod.yml run --rm migrate npm run seed
```

**Check it works:**

```bash
curl -s localhost:3000/healthz
E2E_BASE_URL=http://localhost:3000 npm run e2e
```

```text
{"status":"ok","commit":"local","time":"2026-10-06T11:02:13.551Z"}
  ... all tests passed ...
```

Open `http://localhost:3000/admin` and log in with the user your seed created (or create the first user).
When you are done:

```bash
docker compose -f docker-compose.prod.yml down -v   # -v also deletes the throwaway database volume
```

**What just happened:** you ran production images against an empty database, migrated it with the migrator
image and served the site from the standalone server. That is exactly the sequence AWS will run.

### [Intermediate] Step 29 — Switch media storage to S3 in the Payload config

**What we're doing:** telling Payload to store `media` in an S3 bucket on AWS.

**Why:** ECS tasks have temporary disks, just like Vercel functions (Step 4).

**Do it:** install the adapter (same version as your other `@payloadcms/*` packages):

```bash
npm install @payloadcms/storage-s3
```

In `src/payload.config.ts`, add the import and the plugin. If you are only doing Path B, replace the
`vercelBlobStorage(...)` entry; if you want the code to support both, keep both. Each one is disabled unless its
own variable is set.

```ts
// src/payload.config.ts (snippet: add next to the other imports)
import { s3Storage } from '@payloadcms/storage-s3'
```

```ts
// src/payload.config.ts (snippet: inside plugins: [ ... ])
    s3Storage({
      enabled: Boolean(process.env.S3_BUCKET),
      collections: { media: true },
      bucket: process.env.S3_BUCKET || '',
      config: {
        region: process.env.S3_REGION || 'us-east-1',
        // No credentials here on purpose. The AWS SDK finds them automatically:
        // on ECS from the task's IAM role, on your laptop from your AWS CLI profile.
      },
    }),
```

**Check it works:**

```bash
npm run typecheck && npm run build
```

Both pass. With `S3_BUCKET` unset locally, uploads still go to `./media`.

**What just happened:** Payload now uploads to S3 when `S3_BUCKET` is set. Leaving out `credentials` is the
important part: the container will borrow temporary credentials from its IAM **task role**, so no access key
ever exists to leak.

> **Gotcha:** Payload's docs show `credentials: { accessKeyId, secretAccessKey }` from env vars. That works,
> but long-lived access keys are exactly what we want to avoid on AWS. Use keys only for non-AWS hosts or
> S3-compatible services such as Cloudflare R2.

### [Intermediate] Step 30 — Create the network rules and RDS Postgres

**What we're doing:** creating a managed Postgres 16 that only your app can reach.

**Why:** a database open to the internet is scanned by bots within minutes. We put a firewall (security group)
around it that admits only the app's security group.

**Do it:** you need the AWS CLI v2, logged in to an account where you are allowed to create resources
(`aws configure sso` or `aws configure`). We use the **default VPC** to keep this short.

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

VPC_ID=$(aws ec2 describe-vpcs --filters Name=is-default,Values=true \
  --query 'Vpcs[0].VpcId' --output text)
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=$VPC_ID Name=default-for-az,Values=true \
  --query 'Subnets[].SubnetId' --output text | tr '\t' ',')

# Firewall for the app tasks (Express Mode adds its own rules for the load balancer)
APP_SG=$(aws ec2 create-security-group --group-name my-site-app \
  --description "my-site app tasks" --vpc-id $VPC_ID --query GroupId --output text)

# Firewall for the database: allow Postgres only from the app security group
DB_SG=$(aws ec2 create-security-group --group-name my-site-db \
  --description "my-site Postgres" --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $DB_SG \
  --protocol tcp --port 5432 --source-group $APP_SG

echo "SUBNET_IDS=$SUBNET_IDS APP_SG=$APP_SG DB_SG=$DB_SG"
```

Now the database. Pick the newest 16.x version RDS offers, generate a password, and create the instance:

```bash
PG_VERSION=$(aws rds describe-db-engine-versions --engine postgres \
  --query "DBEngineVersions[?starts_with(EngineVersion,'16.')].EngineVersion | [-1]" --output text)
DB_PASSWORD=$(openssl rand -hex 24)

aws rds create-db-instance \
  --db-instance-identifier my-site-db \
  --engine postgres --engine-version "$PG_VERSION" \
  --db-instance-class db.t4g.micro \
  --allocated-storage 20 --storage-type gp3 --storage-encrypted \
  --db-name my_site \
  --master-username mysite --master-user-password "$DB_PASSWORD" \
  --vpc-security-group-ids $DB_SG \
  --no-publicly-accessible \
  --backup-retention-period 7 \
  --copy-tags-to-snapshot \
  --deletion-protection

aws rds wait db-instance-available --db-instance-identifier my-site-db   # 5 to 15 minutes
DB_HOST=$(aws rds describe-db-instances --db-instance-identifier my-site-db \
  --query 'DBInstances[0].Endpoint.Address' --output text)
echo $DB_HOST
```

| Flag | Why |
| --- | --- |
| `db.t4g.micro`, 20 GB `gp3` | Smallest sensible size for a small site. Resize later with a short restart |
| `--storage-encrypted` | Data at rest is encrypted. Cannot be turned on later without a restore |
| `--no-publicly-accessible` | No public IP. Only resources inside the VPC can even try to connect |
| `--vpc-security-group-ids $DB_SG` | Only the app security group may connect on 5432 |
| `--backup-retention-period 7` | Daily automated backups plus point-in-time restore for 7 days |
| `--deletion-protection` | A stray `delete-db-instance` fails instead of destroying production |

Keep `DB_PASSWORD` in your shell for the next step; it goes into Secrets Manager and nowhere else.

**Check it works:**

```bash
aws rds describe-db-instances --db-instance-identifier my-site-db \
  --query 'DBInstances[0].[DBInstanceStatus,EngineVersion,PubliclyAccessible]' --output text
```

```text
available   16.x   False
```

**What just happened:** AWS created a Postgres server in your VPC with encrypted storage and daily backups.
Nothing outside the app's security group can reach it, not even your laptop. Migrations will therefore run
**inside** AWS as a one-off task (Step 33), not from GitHub's runners.

> **Gotcha:** for a real company setup, put tasks and the database in **private subnets** with a NAT gateway or
> VPC endpoints, and only the load balancer in public subnets. The default VPC (all public subnets) keeps this
> tutorial short; the security groups still keep the database closed.

### [Intermediate] Step 31 — Create the S3 bucket, ECR repository and secrets

**What we're doing:** creating storage for media, storage for images, and the secret store the containers read.

**Why:** these are the three remaining external pieces from Step 4: object storage, a registry and a secret
store.

**Do it:** the media bucket. Bucket names are global, so add your account ID:

```bash
export BUCKET=my-site-media-$ACCOUNT_ID
aws s3api create-bucket --bucket $BUCKET --region $AWS_REGION
aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
```

(Outside `us-east-1`, `create-bucket` also needs `--create-bucket-configuration LocationConstraint=$AWS_REGION`.)
The bucket stays **private**: Payload serves files through its own `/api/media/file/...` route, reading from S3
with the task role. Versioning keeps old copies when a file is overwritten or deleted.

The image registry:

```bash
aws ecr create-repository --repository-name my-site \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true

aws ecr put-lifecycle-policy --repository-name my-site --lifecycle-policy-text '{
  "rules": [{
    "rulePriority": 1,
    "description": "keep the 60 newest images (30 deploys, app + migrator)",
    "selection": {"tagStatus": "any", "countType": "imageCountMoreThan", "countNumber": 60},
    "action": {"type": "expire"}
  }]
}'
```

`IMMUTABLE` means a tag such as `my-site:3f9c2ab` can never be overwritten. That is what makes "roll back to tag
X" trustworthy.

The secrets. One Secrets Manager secret holds a small JSON object; ECS can pull individual keys out of it:

```bash
PAYLOAD_SECRET_PROD=$(openssl rand -hex 32)
aws secretsmanager create-secret --name my-site/prod --secret-string "{
  \"DATABASE_URI\": \"postgres://mysite:${DB_PASSWORD}@${DB_HOST}:5432/my_site?sslmode=no-verify\",
  \"PAYLOAD_SECRET\": \"${PAYLOAD_SECRET_PROD}\"
}"
export SECRET_ARN=$(aws secretsmanager describe-secret --secret-id my-site/prod --query ARN --output text)
unset DB_PASSWORD PAYLOAD_SECRET_PROD
```

> **Gotcha:** RDS for Postgres 15 and newer requires encrypted (SSL) connections by default. Node's `pg` driver
> does not trust Amazon's certificate authority out of the box, so `sslmode=require` can fail with a
> certificate error (recent `pg` versions treat `require` as "verify the certificate"). `sslmode=no-verify`
> encrypts the connection without verifying the server certificate, which is acceptable inside your own VPC.
> The stricter option is to download Amazon's RDS CA bundle into the image and pass it as `ssl: { ca }` in the
> `pool` options. Check the `pg` docs for the current behaviour of each `sslmode`.

**Check it works:**

```bash
aws s3api get-public-access-block --bucket $BUCKET --query 'PublicAccessBlockConfiguration.BlockPublicAcls'
aws ecr describe-repositories --repository-names my-site --query 'repositories[0].repositoryUri' --output text
aws secretsmanager describe-secret --secret-id my-site/prod --query Name --output text
```

```text
true
123456789012.dkr.ecr.us-east-1.amazonaws.com/my-site
my-site/prod
```

**What just happened:** the three stores exist. Notice the secret value never appeared in a file or in git; it
went from a shell variable straight into Secrets Manager, and we cleared the variables.

### [Intermediate] Step 32 — Create the IAM roles the containers use

**What we're doing:** creating three roles. Each one answers "who is allowed to do what".

**Why:** on AWS, nothing can do anything without a role that allows it. Mixing up these roles causes most
"AccessDenied" errors on ECS, so learn the difference once.

| Role | Used by | Allowed to |
| --- | --- | --- |
| `ecsTaskExecutionRole` | ECS itself, **before** your code starts | Pull the image from ECR, write logs, read `my-site/prod` from Secrets Manager |
| `my-site-task-role` | **Your code** while it runs | Read and write objects in the media bucket |
| `ecsInfrastructureRoleForExpressServices` | ECS Express Mode | Create and manage the load balancer, target groups, certificates and scaling |

**Do it:** two trust policies (who may assume the role):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ecs-tasks.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

Save that as `ecs-tasks-trust.json`. Save the same file with `"ecs.amazonaws.com"` as the service as
`ecs-trust.json`. Then:

```bash
# 1) Execution role: managed policy + permission to read our one secret
aws iam create-role --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://ecs-tasks-trust.json
aws iam attach-role-policy --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
aws iam put-role-policy --role-name ecsTaskExecutionRole --policy-name read-my-site-secret \
  --policy-document "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",
  \"Action\":\"secretsmanager:GetSecretValue\",\"Resource\":\"$SECRET_ARN\"}]}"

# 2) Task role: what the running app may do (only the media bucket)
aws iam create-role --role-name my-site-task-role \
  --assume-role-policy-document file://ecs-tasks-trust.json
aws iam put-role-policy --role-name my-site-task-role --policy-name media-bucket \
  --policy-document "{\"Version\":\"2012-10-17\",\"Statement\":[
    {\"Effect\":\"Allow\",\"Action\":[\"s3:GetObject\",\"s3:PutObject\",\"s3:DeleteObject\"],
     \"Resource\":\"arn:aws:s3:::$BUCKET/*\"},
    {\"Effect\":\"Allow\",\"Action\":\"s3:ListBucket\",\"Resource\":\"arn:aws:s3:::$BUCKET\"}]}"

# 3) Infrastructure role for Express Mode
aws iam create-role --role-name ecsInfrastructureRoleForExpressServices \
  --assume-role-policy-document file://ecs-trust.json
aws iam attach-role-policy --role-name ecsInfrastructureRoleForExpressServices \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSInfrastructureRoleforExpressGatewayServices
```

If `ecsTaskExecutionRole` already exists in your account, skip its `create-role` and only add the policies.

**Check it works:**

```bash
aws iam list-attached-role-policies --role-name ecsTaskExecutionRole --query 'AttachedPolicies[].PolicyName'
aws iam list-role-policies --role-name my-site-task-role
```

```text
[ "AmazonECSTaskExecutionRolePolicy" ]
{ "PolicyNames": [ "media-bucket" ] }
```

**What just happened:** you followed **least privilege**: the app can touch one bucket and nothing else; ECS can
read one secret and nothing else. If the app is ever compromised, the damage is limited to what its task role
allows.

### [Advanced] Step 33 — First deploy by hand: push images, migrate with a one-off task, create the service

**What we're doing:** doing the whole deploy once manually, so the automated version in Step 35 holds no
surprises.

**Why:** automation hides details. Doing it by hand once means you will understand every log line when the
robot does it.

**Concept: why migrations run as a one-off task.** The database is private, so GitHub's runners cannot reach
it. Instead, we ask ECS to start **one** temporary container (a **task**) from the migrator image, inside the
VPC, with the app's security group. It runs `payload migrate`, exits, and we check its exit code. Only if it is
0 do we roll out the new app version.

```mermaid
sequenceDiagram
  participant Op as You or GitHub Actions
  participant ECR
  participant ECS
  participant Mig as Migrate task
  participant DB as RDS
  participant Svc as my-site service
  Op->>ECR: push my-site:sha and my-site:migrate-sha
  Op->>ECS: register task definition with migrate image
  Op->>ECS: run-task once
  ECS->>Mig: start container in VPC
  Mig->>DB: payload migrate
  Mig-->>ECS: exit code 0
  Op->>ECS: update service to image my-site:sha
  ECS->>Svc: rolling deploy, health check /healthz
  Svc-->>Op: new commit visible at /healthz
```

**Do it.** Commit two small JSON templates to the repo. They contain ARNs (identifiers, not secrets). Replace
`123456789012`, the secret suffix and the bucket name with yours (`echo $SECRET_ARN` shows the full ARN,
including the random six-character suffix AWS adds).

```json
// aws/migrate-task-def.json  (remove this comment line; JSON has no comments)
{
  "family": "my-site-migrate",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "migrate",
      "image": "IMAGE_PLACEHOLDER",
      "essential": true,
      "environment": [{ "name": "NODE_ENV", "value": "production" }],
      "secrets": [
        { "name": "DATABASE_URI", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:my-site/prod-AbC123:DATABASE_URI::" },
        { "name": "PAYLOAD_SECRET", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:my-site/prod-AbC123:PAYLOAD_SECRET::" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-site-migrate",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "migrate"
        }
      }
    }
  ]
}
```

```json
// aws/app-container.json  (remove this comment line)
{
  "image": "IMAGE_PLACEHOLDER",
  "containerPort": 3000,
  "environment": [
    { "name": "S3_BUCKET", "value": "my-site-media-123456789012" },
    { "name": "S3_REGION", "value": "us-east-1" }
  ],
  "secrets": [
    { "name": "DATABASE_URI", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:my-site/prod-AbC123:DATABASE_URI::" },
    { "name": "PAYLOAD_SECRET", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:my-site/prod-AbC123:PAYLOAD_SECRET::" }
  ]
}
```

The `valueFrom` format is `<secret ARN>:<JSON key>::`. ECS reads that key at task start and exposes it as the
environment variable in `name`. The value never appears in the task definition or in logs.

Now build, push and migrate. For the first deploy, the "domain" is the URL Express Mode will give you, which you
do not know yet, so build with a placeholder and rebuild after Step 36.

```bash
export REGISTRY=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
export TAG=$(git rev-parse HEAD)

aws ecr get-login-password | docker login --username AWS --password-stdin $REGISTRY

# --platform: Fargate defaults to x86_64. Needed if your laptop is an Apple Silicon Mac.
docker build --platform linux/amd64 --target runner \
  --build-arg NEXT_PUBLIC_SERVER_URL=https://www.my-site.com \
  --build-arg GIT_COMMIT_SHA=$TAG -t $REGISTRY/my-site:$TAG .
docker build --platform linux/amd64 --target migrator -t $REGISTRY/my-site:migrate-$TAG .
docker push $REGISTRY/my-site:$TAG
docker push $REGISTRY/my-site:migrate-$TAG

# The cluster Express Mode uses, and log groups (with retention) for the migrate task
aws ecs create-cluster --cluster-name default
aws logs create-log-group --log-group-name /ecs/my-site-migrate
aws logs put-retention-policy --log-group-name /ecs/my-site-migrate --retention-in-days 30

# Register a task definition revision that uses this commit's migrate image
jq --arg img "$REGISTRY/my-site:migrate-$TAG" '.containerDefinitions[0].image = $img' \
  aws/migrate-task-def.json > /tmp/migrate.json
aws ecs register-task-definition --cli-input-json file:///tmp/migrate.json > /dev/null

# Run it once, inside the VPC, with the app security group (so RDS lets it in)
TASK_ARN=$(aws ecs run-task --cluster default --launch-type FARGATE \
  --task-definition my-site-migrate \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_IDS],securityGroups=[$APP_SG],assignPublicIp=ENABLED}" \
  --query 'tasks[0].taskArn' --output text)
aws ecs wait tasks-stopped --cluster default --tasks $TASK_ARN
aws ecs describe-tasks --cluster default --tasks $TASK_ARN \
  --query 'tasks[0].containers[0].exitCode' --output text
aws logs tail /ecs/my-site-migrate --since 15m
```

```text
0
... INFO: Migrating: 20261001_120000_initial
... INFO: Migrated:  20261001_120000_initial (1034ms)
... INFO: Done.
```

`assignPublicIp=ENABLED` lets the task pull its image from ECR in the default VPC's public subnets. (In private
subnets you would use a NAT gateway or VPC endpoints instead.)

Now create the Express Mode service:

```bash
jq --arg img "$REGISTRY/my-site:$TAG" '.image = $img' aws/app-container.json > /tmp/app.json

aws ecs create-express-gateway-service \
  --service-name my-site \
  --cluster default \
  --execution-role-arn arn:aws:iam::$ACCOUNT_ID:role/ecsTaskExecutionRole \
  --infrastructure-role-arn arn:aws:iam::$ACCOUNT_ID:role/ecsInfrastructureRoleForExpressServices \
  --task-role-arn arn:aws:iam::$ACCOUNT_ID:role/my-site-task-role \
  --primary-container file:///tmp/app.json \
  --network-configuration "{\"subnets\":[\"${SUBNET_IDS//,/\",\"}\"],\"securityGroups\":[\"$APP_SG\"]}" \
  --cpu 1 --memory 2 \
  --health-check-path /healthz \
  --scaling-target '{"minTaskCount":1,"maxTaskCount":1}' \
  --monitor-resources
```

| Flag | Why |
| --- | --- |
| `--primary-container` | Image, port 3000, plain env vars and secrets from our JSON |
| `--task-role-arn` | Gives the running app its S3 permissions |
| `--network-configuration` | Runs tasks in our subnets **with our app security group**, which RDS trusts |
| `--cpu 1 --memory 2` | 1 vCPU and 2 GB, plenty for Payload's admin. Units follow the Express Mode docs; check the current reference |
| `--health-check-path /healthz` | The load balancer only sends traffic to tasks that answer here |
| `--scaling-target` min 1, max 1 | **One** task, on purpose: see the Gotcha below |
| `--monitor-resources` | Prints progress while AWS creates the load balancer, certificate and service |

**Check it works:** the output (and the ECS console) shows the service's URL, of the form
`https://my-site.ecs.us-east-1.on.aws` (the exact format may differ). Save its ARN and URL:

```bash
export SERVICE_ARN=<serviceArn from the output>
export APP_URL=<the https URL from the output>
curl -s $APP_URL/healthz
```

```text
{"status":"ok","commit":"<your TAG>","time":"..."}
```

Open `$APP_URL/admin` and **immediately create the first admin user** (same reason as Step 22). Upload an image
and confirm it appears in the S3 bucket: `aws s3 ls s3://$BUCKET/ --recursive | head`.

**What just happened:** ECS pulled your image, injected secrets from Secrets Manager, started one Fargate task
in your security group, and Express Mode put an HTTPS load balancer in front of it. The migrate task ran first,
inside the VPC, so the schema was ready before the app started.

> **Gotcha:** `revalidatePath` clears the Next.js cache **of the container that handled the request**. On
> Vercel the cache is shared. With two or more containers, an editor's change refreshes one container while
> the others keep serving stale pages until their cache expires. Run one task (min = max = 1) until you
> configure a shared cache handler (Next.js `cacheHandler` with Redis, for example; check the current Next.js
> self-hosting docs), and keep a reasonable `revalidate` time as a safety net.

> **Gotcha:** Payload offers `prodMigrations` (migrations run when the server starts) and its docs recommend it
> for long-running containers. It is a simpler alternative to the one-off task. We prefer the separate task
> because a failed migration then blocks the deploy **before** any new container starts, and because with
> several containers starting at once you do not want each one racing to migrate.

**If it breaks:**

- Migrate task exit code `1` with `timeout` or `ECONNREFUSED`: the task is not in `$APP_SG`, or the DB security
  group rule is missing.
- `ResourceInitializationError: unable to pull secrets`: the execution role cannot read `$SECRET_ARN`, or the
  `valueFrom` has a typo. The format must end in `:KEY::`.
- `CannotPullContainerError`: no public IP and no NAT, or the image was built for `arm64` (use
  `--platform linux/amd64`).
- Tasks start then stop repeatedly: `aws logs tail` on the service's log group (Step 37) shows the crash. A
  missing env var is the usual cause.

> **Interview tip:** the classic alternative to Express Mode is **ECS Fargate behind an ALB you build
> yourself**: an ALB, target group, HTTPS listener with an ACM certificate, a task definition and an ECS
> service. You get full control (custom listener rules, WAF, blue/green deployments with CodeDeploy) for more
> setup. In GitHub Actions the standard pair is `aws-actions/amazon-ecs-render-task-definition` and
> `aws-actions/amazon-ecs-deploy-task-definition`. Express Mode creates the same kind of resources for you.

### [Advanced] Step 34 — Let GitHub Actions into AWS with OIDC (no stored keys)

**What we're doing:** allowing one GitHub workflow, in one repo, for one environment, to act in your AWS account
for about an hour at a time.

**Why:** the old way was an IAM user's access key pasted into GitHub secrets. Those keys never expire, get
copied around and leak. With **OIDC** (OpenID Connect), GitHub signs a short-lived token for each run that
says "I am a workflow in repo X, environment production", and AWS swaps it for temporary credentials only if
those claims match your trust policy.

```mermaid
sequenceDiagram
  participant W as Workflow run
  participant G as GitHub OIDC provider
  participant S as AWS STS
  participant A as ECR and ECS
  W->>G: request ID token for audience sts.amazonaws.com
  G-->>W: signed token with repo and environment
  W->>S: AssumeRoleWithWebIdentity with token and role ARN
  S->>S: check issuer, audience and sub claim
  S-->>W: temporary credentials, about 1 hour
  W->>A: push images, run migrate task, update service
```

**Do it:** register GitHub as an identity provider (once per AWS account):

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

(Older AWS CLI versions also demand `--thumbprint-list`. AWS no longer relies on it for GitHub, but if the CLI
insists, use the value from GitHub's "OIDC in AWS" docs.)

The trust policy. Replace the account ID and `your-user`:

```json
// github-trust.json  (remove this comment line)
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:your-user/my-site:environment:production"
      }
    }
  }]
}
```

The permissions: push to one repository, register and run the migrate task, update the service, and pass the
three roles to ECS.

```json
// deploy-policy.json  (remove this comment line)
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "ecr:GetAuthorizationToken", "Resource": "*" },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability", "ecr:BatchGetImage", "ecr:CompleteLayerUpload",
        "ecr:InitiateLayerUpload", "ecr:PutImage", "ecr:UploadLayerPart", "ecr:DescribeImages"
      ],
      "Resource": "arn:aws:ecr:us-east-1:123456789012:repository/my-site"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:RegisterTaskDefinition", "ecs:DescribeTaskDefinition", "ecs:RunTask", "ecs:DescribeTasks",
        "ecs:UpdateExpressGatewayService", "ecs:DescribeExpressGatewayService", "ecs:DescribeServices"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
        "arn:aws:iam::123456789012:role/my-site-task-role",
        "arn:aws:iam::123456789012:role/ecsInfrastructureRoleForExpressServices"
      ]
    },
    { "Effect": "Allow", "Action": ["logs:FilterLogEvents", "logs:DescribeLogGroups"], "Resource": "*" }
  ]
}
```

```bash
aws iam create-role --role-name my-site-github-deploy --assume-role-policy-document file://github-trust.json
aws iam put-role-policy --role-name my-site-github-deploy --policy-name deploy \
  --policy-document file://deploy-policy.json
```

> **Outdated:** Express Mode is new, and the exact IAM action names for it may differ from the ones above. If
> the deploy fails with `AccessDenied` naming an action, add that action. Tighten `"Resource": "*"` to your
> cluster and service ARNs once it works.

Now create the GitHub environment and its variables (none of these are secrets):

```bash
gh api -X PUT repos/your-user/my-site/environments/production
gh variable set AWS_DEPLOY_ROLE_ARN --env production --body "arn:aws:iam::$ACCOUNT_ID:role/my-site-github-deploy"
gh variable set ECS_SERVICE_ARN     --env production --body "$SERVICE_ARN"
gh variable set APP_URL             --env production --body "$APP_URL"
gh variable set SUBNET_IDS          --env production --body "$SUBNET_IDS"
gh variable set APP_SECURITY_GROUP  --env production --body "$APP_SG"
gh variable set NEXT_PUBLIC_SERVER_URL --env production --body "https://www.my-site.com"
```

In **Settings > Environments > production**, set **Deployment branches** to `main` only. Optionally add
yourself as a **required reviewer** to get a manual "approve deploy" button.

**Check it works:** `aws iam get-role --role-name my-site-github-deploy --query 'Role.Arn'` prints the ARN, and
`gh variable list --env production` lists six variables.

**What just happened:** you built the trust chain: GitHub proves who it is, AWS checks the `sub` claim, and the
role allows only deploy actions. There is no AWS key anywhere in GitHub.

> **Gotcha:** the `sub` condition is the whole security model. `repo:your-user/*` would let **any** of your
> repos deploy, and `StringLike` with `repo:your-user/my-site:*` would let **any branch or PR** deploy. Pin it
> to the `production` environment and restrict that environment to `main`.

### [Advanced] Step 35 — Write `deploy.yml`: build, push, migrate, deploy, verify

**What we're doing:** automating Step 33 so every green merge to `main` deploys itself.

**Why:** manual deploys get skipped, rushed or done slightly differently each time.

```mermaid
flowchart TD
  CI["CI succeeds on main"] --> Start["deploy.yml starts"]
  Manual["Manual run with image_tag"] --> Start
  Start --> Auth["Assume AWS role via OIDC"]
  Auth --> Q{"image_tag given?"}
  Q -->|"no"| Build["Build and push app and migrate images"]
  Build --> Mig["Run migrate task, require exit 0"]
  Q -->|"yes, rollback"| Exists["Check the tag exists in ECR"]
  Mig --> Deploy["Update Express service image"]
  Exists --> Deploy
  Deploy --> Wait["Poll /healthz until it reports the new commit"]
  Wait --> Smoke["Playwright smoke tests against APP_URL"]
```

**Do it:**

```yaml
# .github/workflows/deploy.yml
name: Deploy to AWS

on:
  workflow_run:
    workflows: ['CI']
    types: [completed]
    branches: [main]
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Existing image tag (a git SHA) to roll back to. Leave empty to build and deploy main.'
        required: false

concurrency:
  group: deploy-production
  cancel-in-progress: false

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    if: github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    timeout-minutes: 40
    environment:
      name: production
      url: ${{ vars.APP_URL }}
    env:
      AWS_REGION: us-east-1
      ECR_REPOSITORY: my-site
      IMAGE_TAG: ${{ inputs.image_tag || github.event.workflow_run.head_sha || github.sha }}
    steps:
      - name: Check out the commit CI tested
        uses: actions/checkout@v6
        with:
          ref: ${{ github.event.workflow_run.head_sha || github.sha }}

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v5
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Log in to ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set image names
        run: |
          echo "IMAGE=${{ steps.ecr.outputs.registry }}/${ECR_REPOSITORY}:${IMAGE_TAG}" >> "$GITHUB_ENV"
          echo "MIGRATE_IMAGE=${{ steps.ecr.outputs.registry }}/${ECR_REPOSITORY}:migrate-${IMAGE_TAG}" >> "$GITHUB_ENV"

      - name: Set up Docker Buildx
        if: ${{ !inputs.image_tag }}
        uses: docker/setup-buildx-action@v3

      - name: Build and push app image
        if: ${{ !inputs.image_tag }}
        uses: docker/build-push-action@v6
        with:
          context: .
          target: runner
          push: true
          tags: ${{ env.IMAGE }}
          build-args: |
            NEXT_PUBLIC_SERVER_URL=${{ vars.NEXT_PUBLIC_SERVER_URL }}
            GIT_COMMIT_SHA=${{ env.IMAGE_TAG }}
          cache-from: type=gha,scope=runner
          cache-to: type=gha,scope=runner,mode=max

      - name: Build and push migrate image
        if: ${{ !inputs.image_tag }}
        uses: docker/build-push-action@v6
        with:
          context: .
          target: migrator
          push: true
          tags: ${{ env.MIGRATE_IMAGE }}
          cache-from: type=gha,scope=migrator
          cache-to: type=gha,scope=migrator,mode=max

      - name: Check the rollback image exists
        if: ${{ inputs.image_tag }}
        run: aws ecr describe-images --repository-name "$ECR_REPOSITORY" --image-ids imageTag="$IMAGE_TAG"

      - name: Run database migrations as a one-off ECS task
        if: ${{ !inputs.image_tag }}
        env:
          SUBNET_IDS: ${{ vars.SUBNET_IDS }}
          APP_SG: ${{ vars.APP_SECURITY_GROUP }}
        run: |
          jq --arg img "$MIGRATE_IMAGE" '.containerDefinitions[0].image = $img' \
            aws/migrate-task-def.json > migrate.json
          aws ecs register-task-definition --cli-input-json file://migrate.json > /dev/null
          TASK_ARN=$(aws ecs run-task --cluster default --launch-type FARGATE \
            --task-definition my-site-migrate \
            --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_IDS],securityGroups=[$APP_SG],assignPublicIp=ENABLED}" \
            --query 'tasks[0].taskArn' --output text)
          echo "Migrate task: $TASK_ARN"
          aws ecs wait tasks-stopped --cluster default --tasks "$TASK_ARN"
          EXIT_CODE=$(aws ecs describe-tasks --cluster default --tasks "$TASK_ARN" \
            --query 'tasks[0].containers[0].exitCode' --output text)
          aws logs tail /ecs/my-site-migrate --since 20m || true
          if [ "$EXIT_CODE" != "0" ]; then
            echo "::error::Migration failed with exit code $EXIT_CODE. Not deploying."
            exit 1
          fi

      - name: Deploy the new image to ECS Express Mode
        env:
          SERVICE_ARN: ${{ vars.ECS_SERVICE_ARN }}
        run: |
          jq --arg img "$IMAGE" '.image = $img' aws/app-container.json > app.json
          aws ecs update-express-gateway-service \
            --service-arn "$SERVICE_ARN" \
            --primary-container file://app.json

      - name: Wait until the new version is live
        env:
          APP_URL: ${{ vars.APP_URL }}
        run: |
          for i in $(seq 1 60); do
            LIVE=$(curl -fsS "$APP_URL/healthz" | jq -r .commit || echo "none")
            echo "Attempt $i: live commit is $LIVE"
            if [ "$LIVE" = "$IMAGE_TAG" ]; then exit 0; fi
            sleep 15
          done
          echo "::error::New version did not go live within 15 minutes"
          exit 1

      - name: Set up Node for smoke tests
        uses: actions/setup-node@v6
        with:
          node-version-file: .nvmrc
          cache: npm

      - name: Smoke test production
        env:
          E2E_BASE_URL: ${{ vars.APP_URL }}
        run: |
          npm ci
          npx playwright install --with-deps chromium
          npx playwright test tests/e2e/smoke.spec.ts
```

The parts that are new compared with `ci.yml`:

| Line | Why |
| --- | --- |
| `workflow_run: workflows: ['CI']`, `branches: [main]` | Start only after CI finished on `main`. The `if:` skips the job when CI failed |
| `ref: ...head_sha` | Deploy exactly the commit CI tested, not whatever `main` is now |
| `workflow_dispatch` + `image_tag` | A manual "deploy this old tag" button: our rollback |
| `concurrency` with `cancel-in-progress: false` | Queue deploys. Never kill one halfway through a migration |
| `permissions: id-token: write` | Allows the job to request the OIDC token. Without it: "Credentials could not be loaded" |
| `environment: production` | Makes the OIDC `sub` match the trust policy, and enables required reviewers |
| `target: runner` / `target: migrator` | Two images from one Dockerfile, both tagged with the commit SHA |
| `cache-from/to: type=gha` | Reuse Docker layers between runs via the GitHub Actions cache |
| migrate step `exit 1` on non-zero | A failed migration stops the pipeline **before** new code goes live |
| `update-express-gateway-service` | ECS creates a new revision and does a rolling, zero-downtime replacement |
| poll `/healthz` for the commit | A simple, API-independent way to know the new version is actually serving |
| smoke tests with `E2E_BASE_URL` | Same tests as CI, now against the real thing |

> **Outdated:** the action majors shown (`configure-aws-credentials@v5`, `amazon-ecr-login@v2`,
> `setup-buildx-action@v3`, `build-push-action@v6`) were current when this was written. For extra supply-chain
> safety, pin third-party actions to a full commit SHA and let Dependabot update them. If AWS has published an
> official Express Mode deploy action by the time you read this, it can replace the `update-express-gateway-service`
> step.

> **Gotcha:** `update-express-gateway-service` should receive the **full** container definition (image, env,
> secrets), which is why we render `aws/app-container.json` instead of passing only the new image. Adding an
> environment variable later means editing that file in a PR, which also gives you a review trail.

**Check it works:** merge any small PR. In **Actions**, CI runs on `main`, then **Deploy to AWS** starts. The
log ends with:

```text
Attempt 7: live commit is 4be1c9a0d2...
  3 passed (5.2s)
```

**What just happened:** your merge button is now a deploy button for AWS too: build, push, migrate in the VPC,
rolling update, verification, smoke test, all without a single stored credential.

### [Intermediate] Step 36 — Custom domain with Route 53 and ACM

**What we're doing:** serving the AWS deployment at `www.my-site.com` over HTTPS.

**Why:** same as Step 24, but on AWS you assemble DNS and certificates yourself.

**Concept:** **Route 53** is AWS's DNS service. A **hosted zone** holds your domain's records. **ACM** (AWS
Certificate Manager) issues free TLS certificates for AWS load balancers and renews them automatically once you
prove domain ownership with a DNS record. An **alias record** is a Route 53 special record that points a name
(including the apex `my-site.com`) at an AWS resource such as a load balancer.

**Do it:**

```bash
# 1) Hosted zone (if you bought the domain elsewhere, set its name servers to the four NS values returned)
aws route53 create-hosted-zone --name my-site.com --caller-reference "$(date +%s)"

# 2) Certificate for both names, validated through DNS
CERT_ARN=$(aws acm request-certificate --domain-name www.my-site.com \
  --subject-alternative-names my-site.com --validation-method DNS \
  --query CertificateArn --output text)
aws acm describe-certificate --certificate-arn $CERT_ARN \
  --query 'Certificate.DomainValidationOptions[].ResourceRecord'
# Create the CNAME records it prints in the hosted zone (the ACM console has a
# "Create records in Route 53" button that does this for you), then:
aws acm wait certificate-validated --certificate-arn $CERT_ARN
```

3. Attach the certificate and your host name to the load balancer that Express Mode created: in the EC2
   console, **Load Balancers**, find the one serving `my-site`, add `$CERT_ARN` to its HTTPS listener's
   certificate list, and make sure a listener rule forwards host `www.my-site.com` to the `my-site` target
   group.
4. In Route 53, create an **A record, alias** for `www.my-site.com` pointing at that load balancer, and one for
   `my-site.com` (or redirect the apex to `www` at the listener).
5. Make sure the `NEXT_PUBLIC_SERVER_URL` GitHub variable is `https://www.my-site.com`, then redeploy, since it
   is a build-time value.

> **Outdated:** Express Mode manages its load balancer's listener rules itself, and custom domain support was
> still evolving when this guide was written. Before editing the load balancer by hand, check the ECS Express
> Mode docs for a built-in custom domain option, because manual changes to resources a service manages can be
> overwritten. If you need full control of the load balancer, use classic ECS Fargate + your own ALB (Step 33
> interview tip).

**Check it works:**

```bash
curl -sI https://www.my-site.com/healthz | head -1
curl -s https://www.my-site.com/healthz
```

```text
HTTP/2 200
{"status":"ok","commit":"4be1c9a0d2...","time":"..."}
```

**What just happened:** DNS now sends `www.my-site.com` to your load balancer, which terminates HTTPS with an
ACM certificate and forwards plain HTTP to the container inside AWS's network.

### [Intermediate] Step 37 — Logs and alarms in CloudWatch

**What we're doing:** finding your app's output and getting told when it breaks.

**Why:** on your laptop you watch the terminal. In production, nobody is watching unless you set it up.

Everything your container prints to stdout and stderr (every `console.log`, Payload's logger, Next.js errors)
goes to **CloudWatch Logs**. Express Mode creates a log group named like `/aws/ecs/default/my-site-<suffix>`.

```bash
aws logs describe-log-groups --log-group-name-prefix /aws/ecs/default/my-site \
  --query 'logGroups[].logGroupName'
aws logs tail /aws/ecs/default/my-site-1234 --follow --since 15m
aws logs put-retention-policy --log-group-name /aws/ecs/default/my-site-1234 --retention-in-days 30
```

Retention matters: by default CloudWatch keeps logs forever, and you pay for storage forever.

Add one alarm that emails you when the site returns server errors. The load balancer publishes
`HTTPCode_Target_5XX_Count`:

```bash
TOPIC_ARN=$(aws sns create-topic --name my-site-alerts --query TopicArn --output text)
aws sns subscribe --topic-arn $TOPIC_ARN --protocol email --notification-endpoint you@example.com
# Confirm the subscription from the email AWS sends you.

aws cloudwatch put-metric-alarm --alarm-name my-site-5xx \
  --namespace AWS/ApplicationELB --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/<alb-name>/<alb-id> \
  --statistic Sum --period 300 --evaluation-periods 1 --threshold 10 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching --alarm-actions $TOPIC_ARN
```

The `LoadBalancer` dimension is the part of the load balancer ARN after `loadbalancer/`. Express Mode also
creates its own alarm used to detect bad deployments; yours is for **you**.

**Check it works:** in CloudWatch **Logs Insights**, choose the log group and run:

```text
fields @timestamp, @message
| filter @message like /error/i
| sort @timestamp desc
| limit 20
```

You should see recent log lines (or none, if nothing has failed, which is the goal).

**What just happened:** logs are searchable and kept for 30 days, and a burst of 5xx errors emails you.

### [Beginner] Step 38 — Roll back to the previous image tag

**What we're doing:** undoing a bad AWS deploy.

**Why:** same reason as Step 25. Practise it before you need it.

Every image is tagged with its commit SHA and tags are immutable, so rollback is "deploy an older tag":

```bash
# The five newest app images (ignore the migrate-* ones)
aws ecr describe-images --repository-name my-site \
  --query 'reverse(sort_by(imageDetails,&imagePushedAt))[:10].[imageTags[0],imagePushedAt]' --output table

gh workflow run deploy.yml -f image_tag=<previous good sha>
gh run watch
```

The workflow skips the build and the migrations (the `if: ${{ !inputs.image_tag }}` lines), points the service
at the old image, waits for `/healthz` to report that SHA, and runs the smoke tests.

**Check it works:** `curl -s https://www.my-site.com/healthz` shows the older commit. Then fix forward with a
normal PR; the next merge deploys a new SHA as usual.

**What just happened:** the rollback reused an image that already passed CI and ran in production before.
Exactly as on Vercel, it does **not** undo migrations, so the expand-and-contract rules from Step 25 apply
here too.

> **Interview tip:** Express Mode also uses canary-style deployments with an alarm that can roll back
> automatically on 5xx errors. Automatic rollback is a safety net; a tested manual rollback is still required,
> because some bugs (wrong content, broken layout) never produce a 5xx.

## 6. Run it in production

Deploying is the start, not the end. A production site needs someone (or something) watching it, backups you
know how to restore, dependencies that stay patched and a repeatable release routine.

### [Beginner] Step 39 — Monitoring: uptime, errors and logs

**What we're doing:** setting up three kinds of monitoring, each answering a different question.

**Why:** without monitoring, your users are your monitoring. They will not email you; they will leave.

```mermaid
flowchart LR
  Up["Uptime monitor<br/>is it reachable?"] -->|"GET /healthz every minute"| Site["www.my-site.com"]
  Site -->|"exceptions with stack traces"| Sentry["Sentry<br/>what broke and where?"]
  Site -->|"stdout and stderr"| Logs["Vercel logs or CloudWatch<br/>what happened around it?"]
  Up --> Alert["Email or chat alert"]
  Sentry --> Alert
```

**1. Uptime.** Create a free account with an uptime service (UptimeRobot, Better Stack and others have free
tiers). Add an HTTP monitor for `https://www.my-site.com/healthz`, every 1 to 5 minutes, alerting your email.
Add a second **keyword** monitor for the home page that checks for a word that is always there (your site
name). The first catches "server down", the second catches "server up but broken".

**2. Error tracking with Sentry.** Sentry catches exceptions in the browser and on the server, groups them, and
shows the stack trace, the URL, the browser and the release (commit) where it started.

```bash
npx @sentry/wizard@latest -i nextjs
```

The wizard asks you to log in, creates the Sentry project, and adds files such as `instrumentation.ts`,
`instrumentation-client.ts` and `sentry.server.config.ts`, plus a `withSentryConfig` wrapper in
`next.config.mjs`. Because our config is already wrapped by Payload, make sure the result nests both:

```js
// next.config.mjs (snippet: the export at the bottom)
import { withSentryConfig } from '@sentry/nextjs'
// ...nextConfig as before...
export default withSentryConfig(withPayload(nextConfig, { devBundleServerPackages: false }), {
  org: 'your-org',
  project: 'my-site',
  silent: !process.env.CI,
})
```

Then add `NEXT_PUBLIC_SENTRY_DSN` (it is fine for this one to be public: it only allows **sending** events) to
Vercel Production and Preview, or to the `NEXT_PUBLIC_SENTRY_DSN` build argument on AWS. For readable stack
traces Sentry uploads source maps during the build using `SENTRY_AUTH_TOKEN`, which **is** a secret: add it
to Vercel as a secret, or pass it to the Docker build as a build secret, never as a plain build argument.

> **Outdated:** the Sentry wizard and the exact file names change between major versions of `@sentry/nextjs`.
> Follow what the wizard prints. Rerun `npm run typecheck && npm run build` afterwards.

**Check it works:** the wizard offers to create a test page (`/sentry-example-page`). Deploy, click its button,
and within a minute the error appears in Sentry tagged with your environment. Delete the example page
afterwards.

**3. Logs.** Vercel: **Logs** tab, filter by status `500` or search a message. AWS: `aws logs tail` and Logs
Insights (Step 37). Log useful context, never secrets or passwords. Payload already logs through its own
logger (`req.payload.logger.info(...)`), which writes structured JSON in production. Use it in your hooks and
server actions instead of `console.log`.

**What just happened:** you will now hear about downtime from a robot within minutes, see exceptions with stack
traces, and have logs to explain them.

### [Intermediate] Step 40 — Database backups and a restore test

**What we're doing:** confirming that you can get your data back, by actually doing it.

**Why:** "we have backups" means nothing until someone has restored one. Many teams discover during an
incident that backups were empty, too old or impossible to restore in time.

**What the platforms give you:**

| | Neon (Path A) | RDS (Path B) |
| --- | --- | --- |
| Automatic protection | History retention: restore to any moment within a window (length depends on plan) | Daily automated snapshots + point-in-time restore within the retention period (we set 7 days) |
| How a restore works | Create a **branch from a past timestamp**, or restore the branch in place | Restore into a **new** instance, then point the app at it |
| Media files | Vercel Blob (no versioning; deletes are permanent) | S3 with versioning on (Step 31) |

Platform backups live in the same provider account. Add an independent logical backup too: a `pg_dump` file
you store somewhere else.

**Do it, Path A restore test** (about 10 minutes, once a quarter):

1. In the Neon console, open your project, **Branches > Create branch**, choose `main` as parent and **a point
   in time**, for example one hour ago. Name it `restore-test`.
2. Copy its connection string and check the data with `psql` in a throwaway container:

```bash
docker run --rm -it postgres:16 psql "postgresql://...restore-test-host.../neondb?sslmode=require" \
  -c "select count(*) as posts from posts;" \
  -c "select max(updated_at) from posts;"
```

```text
 posts
-------
    12
          max
------------------------
 2026-10-06 09:58:12+00
```

3. Delete the `restore-test` branch.

**Do it, logical dump and restore into local Docker** (works for both paths; for RDS, run it from a one-off
task or a machine inside the VPC):

```bash
# Dump (the pg_dump major version must be >= the server's)
docker run --rm postgres:16 pg_dump "$PROD_DATABASE_URL" --format=custom --no-owner > my-site-$(date +%F).dump

# Restore into a new database in your local Docker Postgres
docker compose exec -T db createdb -U postgres my_site_restore
docker compose exec -T db pg_restore -U postgres -d my_site_restore --no-owner < my-site-$(date +%F).dump
docker compose exec -T db psql -U postgres -d my_site_restore -c "select count(*) from payload_migrations;"
```

Then point a local run at it to click around: `DATABASE_URI=postgres://postgres:postgres@localhost:5432/my_site_restore npm run start`
(after `npm run build`). Use `start`, not `dev`: dev mode's schema push could alter the restored copy.

**Do it, Path B restore test:**

```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier my-site-db \
  --target-db-instance-identifier my-site-restore-test \
  --use-latest-restorable-time \
  --db-instance-class db.t4g.micro \
  --vpc-security-group-ids $DB_SG --no-publicly-accessible
aws rds wait db-instance-available --db-instance-identifier my-site-restore-test
```

Verify it with a one-off ECS task (same pattern as the migrate task) whose `DATABASE_URI` points at the restored
host, running `npx payload migrate:status`; then delete it with
`aws rds delete-db-instance --db-instance-identifier my-site-restore-test --skip-final-snapshot`.

**Check it works:** write down in the README how long the restore took and the date you tested it. That number
is your real **RTO** (recovery time objective: how long you are down). The backup age is your **RPO**
(recovery point objective: how much data you can lose).

> **Gotcha:** dump files contain everything: admin password hashes, contact form messages. Store them
> encrypted, in a private location with restricted access, and delete old ones. Never commit them or upload them
> as public CI artifacts.

**What just happened:** you proved, with real commands, that a bad migration or an accidental delete is a
recoverable event, and you know how long recovery takes.

### [Beginner] Step 41 — Keep dependencies updated with Dependabot

**What we're doing:** letting GitHub open PRs that update your packages and actions.

**Why:** most security issues in web apps come from outdated dependencies. Small weekly updates are easy;
a year of skipped updates is a project.

**Do it:**

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
    groups:
      payload:
        patterns: ['payload', '@payloadcms/*']
      next-and-react:
        patterns: ['next', 'react', 'react-dom', '@types/react', '@types/react-dom']
      dev-dependencies:
        dependency-type: development
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
  - package-ecosystem: docker
    directory: /
    schedule:
      interval: monthly
```

The `payload` group matters: all `@payloadcms/*` packages must move together, and grouping them makes one PR
instead of six mismatched ones. Also turn on **Settings > Code security > Dependabot alerts** and **security
updates**.

**Check it works:** **Insights > Dependency graph > Dependabot** shows the three ecosystems with a "Last
checked" time. Within a week you get PRs titled like `chore(deps): bump the payload group with 7 updates`.

The routine: CI runs on each Dependabot PR. Green and a patch or minor version: open the preview, click around,
merge. Major version: read the changelog first.

> **Gotcha:** a Payload upgrade can change the database schema Payload expects. After merging a Payload update
> locally, run `npm run migrate:create -- payload-upgrade` with your dev database. If it generates a
> non-empty migration, commit it in the same PR, otherwise production will run new code against an old schema.

**What just happened:** updates now arrive as small, tested, reviewable PRs instead of a scary yearly upgrade.

### [Beginner] Step 42 — A release checklist

**What we're doing:** writing down what a careful engineer checks, so you do not rely on memory.

**Why:** automation covers the repeatable parts. Judgment calls (is this migration safe? did we tell the
editors?) still need a human, and a checklist makes sure they happen.

**Before merging a PR:**

- [ ] CI green (both jobs). No skipped or `.only` tests.
- [ ] Clicked through the preview deployment, including `/admin` if collections changed.
- [ ] Schema change? A migration file is committed, and it is **backward compatible** (Step 25).
- [ ] New environment variable? Added to `.env.example`, Vercel (Production and Preview) or `aws/app-container.json`
      and Secrets Manager, **before** merging.
- [ ] Nothing secret in the diff, nothing secret behind a `NEXT_PUBLIC_` name.

**Right after the deploy:**

- [ ] `/healthz` reports the new commit.
- [ ] Smoke tests passed (CI on Path B; run them by hand on Path A:
      `E2E_BASE_URL=https://www.my-site.com npx playwright test tests/e2e/smoke.spec.ts`).
- [ ] No new issues in Sentry for 15 minutes; uptime monitor green.
- [ ] If anything is wrong: roll back first (Step 25 or 38), investigate second.

**Once, before launch:**

- [ ] Custom domain and HTTPS work, apex redirects to `www`.
- [ ] First admin user created by you; any test users deleted; strong passwords.
- [ ] Contact form tested on production; submissions visible only to logged-in admins.
- [ ] Backups configured **and** a restore tested (Step 40).
- [ ] Uptime monitor and Sentry alerts reach a person.
- [ ] `robots.txt` allows indexing on production only (previews should not be indexed).

**What just happened:** you have a short, honest checklist. Put it in `.github/pull_request_template.md` so every
PR shows the "before merging" part automatically.

### [Beginner] Step 43 — Compare rough monthly costs

**What we're doing:** estimating what each path costs for a small site.

**Why:** "which platform?" is partly a money question, and interviewers like candidates who think about cost.

> **Finance tip:** these are rough, order-of-magnitude numbers for a small site with modest traffic in
> `us-east-1`, checked in October 2026. Free tiers, plan prices and included usage change often, and traffic,
> bandwidth and image optimisation can change the picture completely. Always check the current Vercel, Neon and
> AWS pricing pages (and the AWS Pricing Calculator) before deciding.

| Item | Path A: Hobby + free tiers | Path A: Pro | Path B: AWS (ECS Express + RDS + S3) |
| --- | --- | --- | --- |
| Hosting / compute | Vercel Hobby: $0, **personal and non-commercial only** | Vercel Pro: about $20 per member per month, with included usage | Fargate 1 vCPU / 2 GB running 24/7: roughly $30 to $40 |
| Load balancer and HTTPS | Included | Included | ALB: roughly $16 to $25 (shared by up to 25 Express services) |
| Database | Neon free plan: $0 within its limits (scales to zero when idle) | Neon paid plan: usage-based, often $5 to $30 for a small site | RDS `db.t4g.micro` + 20 GB: roughly $15 to $20 |
| Media storage | Vercel Blob free allowance | Small per-GB charges | S3: cents to a few dollars |
| Other | None | Overage on bandwidth or functions if traffic spikes | Public IPv4 addresses (about $3.60 each per month), Secrets Manager ($0.40 per secret), ECR, CloudWatch: roughly $5 to $15 |
| Your time | Very low | Very low | Medium: you own networking, IAM, patches of the setup |
| **Typical total** | **$0** | **About $20 to $50** | **Roughly $70 to $100** |

The honest summary: for a small content website, Path A is usually cheaper **and** far less work. Path B wins
when an organisation already runs on AWS (shared networking, security tooling, one bill, existing skills), has
compliance rules about where data lives, or has traffic large enough that platform usage pricing costs more
than running containers.

**What just happened:** you can justify the platform choice with numbers and trade-offs, not just preference.

## 7. Interview questions

#### Q: Walk me through what happens from `git push` to your code running in production.

I push a branch and open a pull request. GitHub Actions runs CI: one job runs lint, type check and unit tests;
another starts a Postgres service container, runs Payload migrations on the empty database, seeds content,
builds the app and runs Playwright tests against `next start`. Branch protection blocks merging until both
checks are green. In parallel, Vercel builds a preview deployment with its own Neon database branch, so
reviewers can click through the change. When I squash-merge, Vercel runs `payload migrate` against the
production database, then `next build`, and switches traffic only if both succeed. On AWS the equivalent is a
deploy workflow triggered after CI passes on `main`: it assumes an AWS role via OIDC, builds and pushes app and
migrator images tagged with the commit SHA, runs migrations as a one-off ECS task, updates the ECS service and
runs smoke tests against the live URL.

#### Q: How do you handle database migrations in an automated deployment, and what makes a migration safe?

Schema changes are versioned migration files committed with the code. CI applies all of them to an empty
database on every PR, which proves they run from zero. Deploys apply pending migrations **before** the new code
goes live (in the Vercel build, or as a one-off ECS task), and a failed migration stops the deploy. Because the
old code briefly runs against the new schema, and because a rollback restores old code but not the old schema,
every migration must be backward compatible. I use expand and contract: add new structures as optional, deploy
code that uses them, backfill, and only remove old columns in a later release once nothing reads them. Renames
and "make required" are always split across several PRs. Down migrations are a last resort in production
because they can delete data.

#### Q: Why can't a Next.js app on Vercel or ECS store uploaded files on its own disk?

Because the instances are ephemeral and not shared. Serverless functions and containers can be created,
replaced or scaled at any moment, every deploy replaces them, and two requests can hit two different
instances. A file saved on one instance's disk is invisible to the others and is lost when the instance goes
away. Uploads must go to object storage such as S3 or Vercel Blob, and all other state to a database. With
Payload this is a storage adapter (`@payloadcms/storage-s3` or `@payloadcms/storage-vercel-blob`) on the media
collection; Payload keeps the file's metadata in Postgres and the bytes in the bucket.

#### Q: How do you manage configuration and secrets across local, preview and production?

Every setting is an environment variable, documented with placeholders in a committed `.env.example`. Real
values live in `.env` locally (git-ignored), in Vercel's per-environment variables, or in AWS Secrets Manager
injected by ECS at task start. Each environment has its own database and its own `PAYLOAD_SECRET`, so a preview
can never mint production sessions. I keep the distinction between config and secrets clear, and never put a
secret behind a `NEXT_PUBLIC_` name, because those are compiled into the browser bundle at build time. For CI/CD
access to AWS I use GitHub OIDC with a role whose trust policy is pinned to the repo and the `production`
environment, so there are no long-lived keys to leak. The running app reaches S3 through its IAM task role, also
with no keys.

#### Q: Vercel with Neon, or AWS with ECS and RDS: how would you choose?

For a small or medium content site, Vercel and Neon: zero-config Next.js support including caching and
revalidation, preview deployments per PR with database branches, instant rollbacks, HTTPS and a global network,
at low cost and very low operational effort. I would choose AWS when the organisation already runs there and
needs one account for security and billing, requires private networking or data residency, wants to avoid
platform lock-in, or has traffic where container costs beat usage-based pricing. The trade-off is ownership: on
AWS we manage the Dockerfile, networking, IAM, certificates, scaling and a shared cache for revalidation across
multiple containers. Both use the same codebase; storage and database differences are switched by environment
variables.

#### Q: A deploy just broke production. What do you do?

First restore service, then investigate. I check the uptime alert, Sentry and logs to confirm the scope, and
roll back immediately: Instant Rollback on Vercel, or rerun the deploy workflow with the previous image SHA on
AWS. That is safe because migrations are backward compatible. I tell stakeholders what is happening. Once the
site is healthy I reproduce the bug locally or on a preview, fix it forward with a normal PR that includes a
test which would have caught it, and deploy through the usual pipeline. If data was damaged, I restore to a
point in time into a separate branch or instance, verify it, and copy back what is needed. Afterwards I write a
short blameless post-mortem: what happened, why our checks missed it, and which check or alert we are adding.

## Cheatsheet

**Everyday Git and GitHub**

```bash
git switch main && git pull                 # start from the latest main
git switch -c feat/short-description        # one branch per change
git add -A && git commit -m "feat: ..."
git push -u origin feat/short-description
gh pr create --fill                         # open the PR
gh pr checks --watch                        # watch CI
gh pr merge --squash --delete-branch        # merge when green
```

**Local checks (same as CI)**

```bash
npm run lint && npm run typecheck && npm test
docker compose up -d db && npm run migrate && npm run seed
npm run build && npm run e2e
E2E_BASE_URL=https://www.my-site.com npx playwright test tests/e2e/smoke.spec.ts
```

**Migrations**

```bash
npm run migrate:create -- describe-change   # generate a migration from your collection changes
npm run migrate                             # apply pending migrations
npx payload migrate:status                  # which ran, which are pending
```

**Path A: Vercel + Neon + Blob**

```bash
vercel link                                  # connect the folder to the Vercel project
vercel env ls                                # list variables per environment
openssl rand -hex 32 | vercel env add PAYLOAD_SECRET production
vercel ls my-site --prod                     # recent production deployments
vercel rollback <deployment-url>             # instant rollback
```

Build Command: `npm run build:deploy` (= `payload migrate && next build`). Neon integration sets
`DATABASE_URL` (pooled) per environment and a database branch per preview. Blob sets `BLOB_READ_WRITE_TOKEN`.

**Path B: AWS**

```bash
docker compose -f docker-compose.prod.yml up --build -d     # prod-like local test
aws ecr get-login-password | docker login --username AWS --password-stdin $REGISTRY
docker build --platform linux/amd64 --target runner   -t $REGISTRY/my-site:$TAG .
docker build --platform linux/amd64 --target migrator -t $REGISTRY/my-site:migrate-$TAG .
aws ecs run-task --cluster default --launch-type FARGATE --task-definition my-site-migrate ...
aws ecs update-express-gateway-service --service-arn $SERVICE_ARN --primary-container file://app.json
aws logs tail /aws/ecs/default/my-site-<suffix> --follow
gh workflow run deploy.yml -f image_tag=<previous sha>     # rollback
```

**Where each thing lives**

| Thing | Local | CI | Path A | Path B |
| --- | --- | --- | --- | --- |
| Postgres | Docker `db` | `services: postgres` | Neon (branch per preview) | RDS, private |
| Media | `./media` | none | Vercel Blob | S3, private, task role |
| Secrets | `.env` | dummy values in `ci.yml` | Vercel env vars per environment | Secrets Manager |
| Migrations | `npm run migrate` | before seed and build | in the Vercel build | one-off ECS task |
| Logs | terminal | Actions log | Vercel Logs | CloudWatch |
| Rollback | `git revert` | n/a | Instant Rollback | deploy previous image tag |

**Rules worth memorising**

- Every environment has its own database and its own `PAYLOAD_SECRET`.
- Never put a secret behind `NEXT_PUBLIC_`. Public values are baked in at build time.
- Migrations are committed, run before new code goes live, and are backward compatible (expand, then contract).
- Rollback restores code, not the schema.
- No long-lived cloud keys: OIDC for CI, IAM roles for the app.
- A backup you have not restored is not a backup.

### [Beginner] You built and shipped a full-stack app

Look at what you did across the three guides. You started with an empty folder and now have:

- A **Next.js App Router** website with server components that read content through the **Payload Local API**.
- A **Payload CMS** admin with pages, posts, media and a contact form protected by access control.
- A **Postgres** database whose schema is managed by versioned **migrations**.
- **Unit and end-to-end tests** that run against a real database on every pull request.
- A protected `main` branch, **CI** that blocks broken code, and **CD** that deploys every merge.
- A production deployment on **Vercel + Neon + Blob** with per-PR preview environments, or on **AWS** with
  Docker, ECS, RDS, S3, Secrets Manager and keyless OIDC deploys.
- Monitoring, tested backups, automatic dependency updates and a release checklist.

That is the full loop a professional full-stack developer works in every day. You can explain each piece, why
it exists and what breaks without it, which is exactly what interviews test.

```mermaid
flowchart LR
  Code["Write code<br/>and migration"] --> Test["Tests locally"]
  Test --> PR["Pull request"]
  PR --> CI["CI with real Postgres"]
  CI --> Preview["Preview with DB branch"]
  Preview --> Merge["Merge"]
  Merge --> Deploy["Migrate and deploy"]
  Deploy --> Watch["Monitor, back up, update"]
  Watch --> Code
```

**Ideas for your next features** (each one practises something new, and each goes through the same PR, CI and
deploy loop):

1. **Live preview and drafts:** enable Payload versions and drafts on Posts, and Next.js Draft Mode so editors
   can preview unpublished posts on the real site.
2. **Search:** add Payload's search plugin or a Postgres full-text index, and a `/search` page with a server
   component reading `searchParams`.
3. **SEO:** Payload's SEO plugin, `generateMetadata` per page and post, `sitemap.ts` and `robots.ts` using
   `NEXT_PUBLIC_SERVER_URL`.
4. **Email notifications:** send an email to the site owner from the contact submission `afterChange` hook
   with Payload's email adapter (Resend or SMTP), with the API key as a secret.
5. **Spam protection and rate limiting** on the contact form: a honeypot field plus a per-IP limit.
6. **Scheduled publishing:** a `publishAt` date on posts and a scheduled job (Payload jobs queue, or a Vercel
   Cron / EventBridge rule) that publishes due posts.
7. **Image performance:** responsive sizes on the Media collection and `next/image` with proper `sizes`.
8. **Accessibility and performance budgets in CI:** add `@axe-core/playwright` checks to the e2e tests and a
   Lighthouse CI job that fails the PR if performance drops.
9. **Multiple environments on AWS:** a `staging` ECS service and RDS instance with its own GitHub environment
   and approval rule, promoted to production by deploying the same image tag.
10. **Infrastructure as code:** rewrite the Path B CLI commands as Terraform, AWS CDK or SST so the whole
    environment can be recreated (and reviewed) from git.

---
id: fs-learn-setup
title: "Full-stack 1: Set Up and Model Your Content"
group: "Full-stack Website: Next.js + Payload + Postgres"
tagline: Go from an empty folder to a running Next.js + Payload CMS + Postgres project with a real content model, migrations and seed data, understanding every file and command on the way.
covers: Node.js 22 LTS, npm, Git, Docker Desktop, PostgreSQL 16, Next.js 16 (App Router), React 19, Payload 3.x, @payloadcms/db-postgres (Drizzle), @payloadcms/richtext-lexical, TypeScript
status: current
kind: guide
---

> **Version note (October 2026):** This guide was checked against Payload 3.x (the 3.8x releases at the time of writing), whose templates ship with Next.js 16.2.x and React 19. Payload's docs say it needs Node.js 20.9 or newer and only supports specific Next.js ranges (for example 15.4.11+ within 15.4, and 16.2.6+). We use Node 22 LTS; Node 24 LTS should also work. The `create-payload-app` prompts and the exact list of generated files change a little between releases. If your screen shows a slightly different prompt or an extra file, that is normal. The ideas in this guide do not change, only the spelling. When in doubt, the official docs at payloadcms.com/docs win.

This is guide 1 of 3. By the end of the three guides you will have built one real website, from an empty folder all the way to automatic deploys to production. In this first guide we set up your computer, create the project, and design the content: the "shape" of the pages, blog posts and contact messages the site will store. We will not write much visible frontend yet. That is guide 2. Here we lay a solid foundation, and you will understand every brick.

How to read each step. Every step has the same shape:

- **What we're doing** tells you the goal in a sentence or two.
- **Why** tells you why a real project needs this, and what would go wrong without it.
- **Do it** gives you the exact commands and complete files.
- **Check it works** tells you what to run or open, and what you should see.
- **What just happened** connects the step to how the tools work.
- **If it breaks** (sometimes) lists the most likely errors and their fixes.

Type the commands yourself instead of copying blindly. Reading the output is half the learning.

## 1. The big picture

### [Beginner] Step 1 — See what we will build

**What we're doing:** Agreeing on the finished product before we write any code.

**Why:** Beginners often get lost because they do not know where a step is heading. If you can picture the end result, every command has a purpose.

**Do it:** Read this description of the website. We will build it for a small, made-up business.

- A **home page** at `/` with a heading and some text that the business owner can edit.
- **Other pages** such as `/about` and `/services`, also editable, created without a developer.
- A **blog** at `/blog` that lists posts, and a page per post at `/blog/<slug>`. A **slug** is the short, URL-friendly name of a page, like `hello-world` in `/blog/hello-world`.
- A **contact form** at `/contact`. When a visitor sends it, the message is saved, and the owner can read it.
- An **admin panel** at `/admin`. The owner logs in here to write pages and posts, upload images, and read contact messages. Nobody has to edit code to change content.

The site will run on your laptop first. In guide 3 it goes live on the internet.

**Check it works:** You should be able to answer these three questions without looking: Where does the owner edit content? (At `/admin`.) Where do visitors read posts? (At `/blog`.) What happens to a contact form message? (It is saved and the owner reads it in the admin.)

**What just happened:** You defined the scope. A small, clear scope is how real teams ship. Everything in the next three guides serves one of these five features.

### [Beginner] Step 2 — Understand how the pieces fit

**What we're doing:** Drawing the path a request takes, from the visitor's browser to the database and back.

**Why:** A full-stack app has several layers. When something breaks, the first question is always "which layer?" You can only answer that if you know the layers.

**Do it:** Study this diagram. Read it left to right.

```mermaid
flowchart LR
  B["Browser<br/>visitor or editor"] -->|"HTTP request"| N["Next.js server<br/>App Router"]
  N -->|"renders pages with"| R["React components"]
  N -->|"Local API call"| P["Payload CMS<br/>inside the same app"]
  P -->|"SQL through Drizzle"| D[("Postgres database")]
  P -->|"saves uploads"| F["Media files<br/>disk or S3"]
  E["Editor at /admin"] -->|"uses"| P
```

Here is the same story in words:

1. A visitor types your address. The **browser** sends an **HTTP request**, which is just a structured message saying "please give me the page at `/blog`".
2. **Next.js** receives it. Next.js is the web server and the framework. It decides which React component should handle `/blog`.
3. That component needs the list of posts. It asks **Payload**. Payload is not a separate server here. It is a library that lives inside the same Next.js app, so the call is a normal function call (the "Local API").
4. Payload turns the question into **SQL**, the language databases speak, and sends it to **Postgres**.
5. Postgres returns rows. Payload turns them into JavaScript objects. React turns them into HTML. Next.js sends the HTML back to the browser.

Editors use the same Payload, but through the admin panel at `/admin`, which Payload provides for free.

**Check it works:** Cover the diagram and say the five steps out loud. If you can, you understand the architecture better than many people who have used these tools for months.

**What just happened:** You learned that this project is **one app** (Next.js with Payload inside) plus **one database** (Postgres). That is a simple and popular architecture. There is no separate backend service to deploy, which is a big reason teams choose Payload 3.

> **Interview tip:** If asked "how does Payload 3 differ from a traditional headless CMS?", say: Payload 3 installs into your Next.js app. The admin panel and the API run in the same process as your website, and server components can read content with a direct function call instead of an HTTP request.

### [Beginner] Step 3 — Meet every tool in one sentence

**What we're doing:** Learning the name and job of each tool before we install it.

**Why:** New words pile up fast. A short glossary now means you will not stop and wonder "what is this?" in the middle of a step.

**Do it:** Read this table once. Come back to it whenever a name confuses you.

| Tool | What it is, in one plain sentence |
| --- | --- |
| Terminal | A text window where you type commands to your computer instead of clicking. |
| Node.js | A program that runs JavaScript outside the browser, so it can power servers and tools. |
| npm | Node's package manager: it downloads libraries other people wrote and runs your project's scripts. |
| Next.js | A React framework that adds routing, server rendering and a web server on top of React. |
| React | A library for building user interfaces from small reusable pieces called components. |
| TypeScript | JavaScript with types, so your editor catches mistakes before you run the code. |
| Payload CMS | A content management system that gives you an admin panel, an API and a database layer from a config file. |
| PostgreSQL (Postgres) | A database: a program that stores your data safely in tables and answers questions about it. |
| Docker | A tool that runs programs (like Postgres) in isolated "containers", so you do not install them directly on your machine. |
| Git | A tool that records snapshots of your code over time, so you can see history and undo mistakes. |
| GitHub | A website that stores your Git history online, so you can back it up, share it and run automation. |
| Vercel | A hosting platform made by the Next.js team that builds and serves Next.js apps. |
| AWS | Amazon's cloud: rentable servers, databases and file storage (we may use its S3 file storage in guide 3). |

**Check it works:** Without looking, explain the difference between Git and GitHub. (Git is the tool on your machine. GitHub is a website that hosts Git repositories.)

**What just happened:** You now have names for every layer in the diagram from Step 2. Node runs Next.js. Next.js runs React and Payload. Payload talks to Postgres. Docker runs Postgres on your laptop. Git and GitHub keep your code safe. Vercel or AWS will run it in production.

### [Beginner] Step 4 — See the learning roadmap

**What we're doing:** Looking at what each of the three guides covers.

**Why:** Knowing the plan helps you see why we do some things now (like migrations) that only pay off later (in deploys).

**Do it:** Read the roadmap.

```mermaid
flowchart TD
  G1["Guide 1: Set up and model content<br/>tools, project, collections, migrations, seed"] --> G2["Guide 2: Build the website<br/>pages, blog, contact form, freshness, SEO"]
  G2 --> G3["Guide 3: Test and ship<br/>tests, CI, storage, production deploys"]
  G3 --> L["Live site with automatic deploys"]
```

- **Full-stack 1 (this guide):** Prepare your computer. Create the project with `create-payload-app`. Run Postgres in Docker. Design five collections: Users, Media, Pages, Posts and ContactSubmissions. Create the first migration and a seed script.
- **Full-stack 2:** Build the public website in Next.js: the home page, `/[slug]` pages, `/blog`, `/blog/[slug]`, and the `/contact` form with a server action and Zod validation. Keep pages fresh with `revalidatePath` hooks. Add metadata.
- **Full-stack 3:** Building on the tests from guide 2, set up GitHub Actions for continuous integration, move media to cloud storage, run migrations automatically, and deploy to production.

**Check it works:** You should know which guide teaches the contact form (guide 2) and which teaches deploys (guide 3).

**What just happened:** You have a map. Whenever this guide says "we will use this in guide 2", you will know where it fits.

## 2. Prepare your computer

### [Beginner] Step 5 — Learn just enough terminal

**What we're doing:** Opening a terminal and learning the eight commands and shortcuts you will use most.

**Why:** Every tool in this guide is installed and run from the terminal. If the terminal feels scary, every later step feels scary. Ten minutes here saves hours later.

**Do it:** Open a terminal.

- **macOS:** press Cmd+Space, type `Terminal`, press Enter.
- **Windows:** install **WSL 2** (Windows Subsystem for Linux) and use the Ubuntu terminal. Open PowerShell as administrator and run `wsl --install`, restart, then open "Ubuntu" from the Start menu. Doing everything inside WSL means the Linux commands in this guide work exactly as written.
- **Linux:** open your Terminal app.

The terminal shows a **prompt**, usually ending in `$`. You type a command after it and press Enter. Try these, one at a time:

```bash
pwd                 # "print working directory": which folder am I in?
ls                  # list the files and folders here
mkdir code          # make a new folder called code
cd code             # "change directory": move into it
cd ..               # move up one folder
cd ~/code           # ~ means your home folder, so this goes to /home/you/code
clear               # clear the screen
```

Also learn these keys:

- **Tab** completes a file or folder name. Type `cd co` then press Tab.
- **Up arrow** brings back the previous command.
- **Ctrl+C** stops the running program. You will use it to stop the dev server.
- Text after `#` in the examples above is a **comment**. You do not need to type it.

**Check it works:**

```bash
cd ~/code
pwd
```

```text
/home/you/code
```

On macOS it will look like `/Users/you/code`. That is fine.

**What just happened:** You learned that the terminal always has a **current folder**, and that commands act on it. Most "file not found" errors in this guide come from running a command in the wrong folder. When in doubt, run `pwd`.

### [Beginner] Step 6 — Install Node.js with nvm

**What we're doing:** Installing Node.js 22 (an LTS version) using **nvm**, the Node Version Manager.

**Why:** Next.js and Payload are JavaScript programs that run on Node. Different projects need different Node versions. nvm lets you install several and switch between them with one command. Installing Node from the website works too, but it makes upgrades painful and often needs `sudo`, which causes permission errors later. **LTS** means "long-term support": a version that gets security fixes for years, which is what you want for real projects.

**Do it:** Install nvm. Check the nvm GitHub README for the newest version number in the URL; the one below is an example.

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
```

Close the terminal and open a new one, so it picks up nvm. Then:

```bash
nvm install 22        # download and install the newest Node 22
nvm use 22            # use it in this terminal
nvm alias default 22  # use it in every new terminal too
```

On Windows without WSL you would use the separate "nvm-windows" project instead, but we recommend WSL.

**Check it works:**

```bash
node --version
npm --version
```

```text
v22.x.x
10.x.x
```

Any `v22` version is fine. Your npm number may be higher.

**What just happened:** nvm downloaded Node and put it in your home folder, so you never need administrator rights to install packages. npm came bundled with Node. From now on, `node` runs JavaScript files and `npm` installs libraries and runs scripts.

**If it breaks:**

- `nvm: command not found`: you did not open a new terminal, or your shell config was not updated. Run `source ~/.bashrc` (or `source ~/.zshrc` on macOS) and try again.
- `node --version` shows an old version: run `nvm use 22`, then `nvm alias default 22`.

### [Beginner] Step 7 — Install and configure Git

**What we're doing:** Installing Git and telling it your name and email.

**Why:** Git records every change to your code as a **commit** (a named snapshot). If you break something, you can go back. Without Git, one bad edit can cost a day. Git also needs your name and email to sign each commit.

**Do it:**

- **macOS:** run `git --version`. If Git is missing, macOS offers to install the Command Line Tools. Accept.
- **Ubuntu or WSL:** `sudo apt update && sudo apt install -y git`

Then configure it. Use the email you will use on GitHub.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

**Check it works:**

```bash
git --version
git config --global --list
```

```text
git version 2.4x.x
user.name=Your Name
user.email=you@example.com
init.defaultbranch=main
```

**What just happened:** Git is now installed and every commit you make will carry your name. `init.defaultBranch main` makes new repositories start on a branch called `main`, which is what GitHub expects.

### [Beginner] Step 8 — Install Docker Desktop

**What we're doing:** Installing Docker so we can run Postgres without installing it directly.

**Why:** Installing a database straight onto your laptop is messy. Versions clash, uninstalling is hard, and your setup will differ from your teammates'. Docker runs Postgres in a **container**: an isolated box with its own files, started and stopped with one command. Everyone on the team gets the exact same Postgres 16.

**Do it:**

- **macOS and Windows:** download Docker Desktop from docker.com, install it, and start it. On Windows, enable the "Use WSL 2 based engine" setting and turn on integration with your Ubuntu distro (Settings, Resources, WSL integration).
- **Linux:** install Docker Engine and the Docker Compose plugin by following the official docs for your distribution, then add yourself to the `docker` group so you do not need `sudo`.

Docker Desktop must be **running** (you will see the whale icon) whenever you work on this project.

**Check it works:**

```bash
docker --version
docker compose version
docker run --rm hello-world
```

```text
Docker version 2x.x.x
Docker Compose version v2.x.x
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

**What just happened:** `docker run hello-world` downloaded a tiny **image** (a template for a container), started a container from it, printed a message, and `--rm` deleted the container afterwards. We will do the same with the official Postgres image, but keep it running.

**If it breaks:**

- `Cannot connect to the Docker daemon`: Docker Desktop is not running. Start it and wait until it says "running".
- `docker compose` not found but `docker-compose` works: you have the old version 1. Update Docker Desktop. This guide uses the modern `docker compose` (with a space).

### [Beginner] Step 9 — Install VS Code and helpful extensions

**What we're doing:** Installing an editor that understands TypeScript, plus a few extensions.

**Why:** A good editor shows type errors as you type, formats code for you, and lets you look inside the database without leaving the window. That turns many confusing runtime errors into red squiggles you fix in seconds.

**Do it:** Install Visual Studio Code from code.visualstudio.com. On macOS, open the Command Palette (Cmd+Shift+P) and run "Shell Command: Install 'code' command in PATH" so you can open folders from the terminal with `code .`. On Windows with WSL, install the "WSL" extension and open folders from the Ubuntu terminal with `code .`.

Install these extensions from the Extensions panel (search by name):

- **ESLint**: shows lint problems inline.
- **Prettier - Code formatter**: formats files on save.
- **Docker** (Microsoft, now also published as "Container Tools"): see running containers.
- A **PostgreSQL client**: for example Microsoft's "PostgreSQL" extension or "SQLTools" with its PostgreSQL driver. Any of them is fine. Standalone apps such as TablePlus, DBeaver or pgAdmin also work.
- Optional: **Pretty TypeScript Errors**, which makes long type errors readable.

**Check it works:**

```bash
code --version
```

```text
1.xx.x
...
```

**What just happened:** Your editor is ready. You can now open the project with `code .` from inside its folder.

### [Beginner] Step 10 — Check every version in one go

**What we're doing:** Running one block of commands that confirms all tools are installed.

**Why:** It is much easier to fix a missing tool now than to discover it in the middle of Step 14 with a confusing error message.

**Do it:**

```bash
node --version && npm --version && git --version && docker --version && docker compose version
```

**Check it works:**

```text
v22.x.x
10.x.x
git version 2.4x.x
Docker version 2x.x.x
Docker Compose version v2.x.x
```

**What just happened:** `&&` runs the next command only if the previous one succeeded, so if a tool is missing, the chain stops right at the problem. Your computer now has everything the next 40 steps need.

```mermaid
flowchart LR
  T["Terminal"] --> N["Node 22 and npm<br/>run JavaScript and scripts"]
  T --> G["Git<br/>history of your code"]
  T --> D["Docker<br/>runs Postgres"]
  V["VS Code"] --> N
  V --> D
```

## 3. Create the project

### [Beginner] Step 11 — Create the app with create-payload-app

**What we're doing:** Generating a new Next.js + Payload project called `my-site`, using the blank template and PostgreSQL.

**Why:** Wiring Next.js and Payload together by hand involves a dozen files that must match exactly. The official generator does it correctly in a minute, and it stays in sync with the supported Next.js version. We pick the **blank** template on purpose: the bigger "website" template is great, but it hides too much for learning. We will add every collection ourselves.

**Do it:** Go to your code folder and run the generator. `npx` downloads a package and runs it once without installing it permanently; `@latest` makes sure you get the newest version.

```bash
cd ~/code
npx create-payload-app@latest
```

Answer the prompts like this. The wording may differ slightly in your version.

```text
? Project name? my-site
? Choose project template: blank
? Select a database: PostgreSQL
? Enter PostgreSQL connection string: postgres://postgres:postgres@localhost:5432/my_site
```

If it asks which package manager to use, choose **npm**. It then installs dependencies, which takes a minute or two.

If you prefer one line, the CLI also accepts flags. Run `npx create-payload-app@latest --help` to confirm the names in your version; at the time of writing this works:

```bash
npx create-payload-app@latest -n my-site -t blank --db postgres --db-connection-string postgres://postgres:postgres@localhost:5432/my_site
```

Then move into the project and open it in VS Code:

```bash
cd my-site
code .
```

**Check it works:**

```bash
ls
```

```text
README.md  docker-compose.yml  next.config.mjs  package.json  src  tsconfig.json  ...
```

You should see a `src` folder and a `package.json`. Your list will have a few more files; we tour them all in Step 12.

**What just happened:** The generator copied the blank template, filled in your project name and database connection string, wrote a `.env` file, and ran `npm install`, which downloaded every library listed in `package.json` into the `node_modules` folder. It has not connected to any database yet. That only happens when the app starts.

**If it breaks:**

- `EACCES` permission errors: you probably installed Node without nvm. Go back to Step 6.
- The prompt has no "PostgreSQL" option, only "Postgres": same thing. Pick it.

### [Beginner] Concept — What is the Next.js App Router?

Before we tour the files, you need one idea: in Next.js, **folders are URLs**.

The **App Router** is Next.js's routing system. Inside `src/app`, every folder is one segment of a URL, and a file named `page.tsx` inside it is the React component shown at that URL.

| File | URL |
| --- | --- |
| `src/app/page.tsx` | `/` |
| `src/app/about/page.tsx` | `/about` |
| `src/app/blog/[slug]/page.tsx` | `/blog/anything` (the `[slug]` part is a variable) |

A few other special file names matter:

- `layout.tsx` wraps every page below it. It is where the `<html>` and `<body>` tags and shared headers live.
- `route.ts` is an API endpoint instead of a page. It returns data, not HTML.
- `not-found.tsx` is shown when a page is missing.

Components in the App Router are **server components** by default. They run on the server, can be `async`, and can read the database directly. Only the HTML they produce is sent to the browser. A file that starts with `'use client'` is a **client component**: it also runs in the browser, so it can use `useState` and click handlers. You already know client components from plain React.

```mermaid
flowchart TD
  A["src/app"] --> L["layout.tsx<br/>wraps everything"]
  A --> P["page.tsx<br/>URL /"]
  A --> B["blog/page.tsx<br/>URL /blog"]
  A --> S["blog/[slug]/page.tsx<br/>URL /blog/any-slug"]
  A --> R["api/.../route.ts<br/>returns JSON"]
```

### [Beginner] Concept — Route groups: (frontend) and (payload)

A folder name in **parentheses**, like `(frontend)`, is a **route group**. It organises files but does **not** appear in the URL. So `src/app/(frontend)/blog/page.tsx` is still served at `/blog`.

Why bother? Because each route group can have its **own root layout**. Our project has two very different apps living side by side:

- `(frontend)`: the public website, with your own HTML, CSS and fonts.
- `(payload)`: the admin panel and the API, with Payload's own HTML and styles.

If they shared one layout, the admin's styles would leak into your website and the other way around. Route groups keep them apart while both run in one Next.js server.

```mermaid
flowchart LR
  U["Incoming URL"] --> Q{"Which group<br/>matches?"}
  Q -->|"/ or /blog or /contact"| F["(frontend) layout<br/>your website"]
  Q -->|"/admin"| A["(payload) layout<br/>admin panel"]
  Q -->|"/api/..."| API["(payload) route handlers<br/>REST and GraphQL"]
```

### [Beginner] Step 12 — Tour every generated file and folder

**What we're doing:** Walking through the project so no file is a mystery.

**Why:** You cannot change what you do not understand. Many beginners are afraid to touch generated files. After this step you will know which files you own (most of them) and which are generated for you (a few).

**Do it:** Compare this tree with your project. Yours may differ slightly between Payload releases; for example some versions include test files and a `Dockerfile`, some do not.

```text
my-site/
├── .env                        # your secrets and settings (never committed)
├── .env.example                # a template of .env without real secrets (committed)
├── .gitignore                  # files Git must ignore: node_modules, .env, media, .next
├── .vscode/                    # shared editor settings and recommended extensions
├── Dockerfile                  # instructions to build a production container (guide 3)
├── docker-compose.yml          # services for local development (we replace it in Step 13)
├── README.md                   # notes for humans
├── eslint.config.mjs           # lint rules
├── next.config.mjs             # Next.js settings, wrapped with withPayload()
├── next-env.d.ts               # Next.js type hints (generated, do not edit)
├── package.json                # dependencies and npm scripts
├── package-lock.json           # exact installed versions (generated, but commit it)
├── playwright.config.ts        # end-to-end test config (set up in guide 2), if present
├── vitest.config.mts           # unit test config (set up in guide 2), if present
├── tests/                      # example tests, if present
├── tsconfig.json               # TypeScript settings, including the @payload-config alias
└── src/
    ├── app/
    │   ├── (frontend)/
    │   │   ├── layout.tsx      # root layout of the public website
    │   │   ├── page.tsx        # the home page at /
    │   │   └── styles.css      # website styles
    │   ├── (payload)/
    │   │   ├── admin/
    │   │   │   ├── [[...segments]]/
    │   │   │   │   ├── page.tsx       # renders every /admin/... screen
    │   │   │   │   └── not-found.tsx  # admin 404
    │   │   │   └── importMap.js       # generated map of admin components
    │   │   ├── api/
    │   │   │   ├── [...slug]/route.ts         # the REST API at /api/...
    │   │   │   ├── graphql/route.ts           # the GraphQL API
    │   │   │   └── graphql-playground/route.ts
    │   │   ├── custom.scss     # your custom admin styles
    │   │   └── layout.tsx      # root layout of the admin
    │   └── my-route/route.ts   # an example custom API route (safe to delete later)
    ├── collections/
    │   ├── Media.ts            # the image upload collection
    │   └── Users.ts            # the login collection
    ├── payload-types.ts        # TypeScript types generated from your config
    └── payload.config.ts       # the heart of Payload: collections, database, editor
```

The most important ones, explained:

- **`package.json`** lists two things: **dependencies** (libraries such as `next`, `react`, `payload`, `@payloadcms/db-postgres`, `@payloadcms/richtext-lexical`, `sharp`) and **scripts** (named commands such as `dev` and `build`). `npm run dev` runs the `dev` script.
- **`next.config.mjs`** exports the Next.js config wrapped in `withPayload(...)`. That wrapper teaches Next.js how to bundle Payload correctly. Leave it in place.
- **`tsconfig.json`** has a `paths` entry mapping `@payload-config` to `./src/payload.config.ts`. That is why you will see `import config from '@payload-config'` everywhere.
- **`src/payload.config.ts`** is where you register collections, choose the database adapter and the rich text editor. We read it line by line in Step 15.
- **`[[...segments]]`** is a Next.js **optional catch-all** route. The double brackets mean it matches `/admin`, `/admin/collections/posts`, and any deeper path. Payload then decides which admin screen to show. You never edit this file.
- **`api/[...slug]/route.ts`** is a **catch-all** route handler. It gives you a full REST API, for example `GET /api/posts`, without writing any endpoint code.
- **`payload-types.ts`** and **`importMap.js`** are **generated**. Payload rewrites them. Do not edit them by hand. You regenerate them with `npm run generate:types` and `npm run generate:importmap`.

**Check it works:** Open `src/app/(payload)/admin/[[...segments]]/page.tsx` in VS Code. You should see a very short file that imports `RootPage` from `@payloadcms/next/views` and passes it your config and the import map. That is the whole admin panel: one component.

**What just happened:** You now know the layout of a Payload 3 project. The website lives in `(frontend)`, Payload lives in `(payload)`, your content model lives in `src/collections`, and everything is tied together by `src/payload.config.ts`.

> **Gotcha:** Do not move or rename the files inside `src/app/(payload)`. Payload upgrades sometimes ship changes to them, and the release notes tell you to copy the new versions. Keeping them untouched makes that easy.

### [Beginner] Concept — What is a database, and what is an ORM or adapter?

A **database** is a program whose only job is to store data safely and answer questions about it quickly. **Postgres** is a **relational** database: it stores data in **tables** made of **rows** and **columns**, like strict spreadsheets. A `posts` table has a column for `title`, a column for `slug`, and one row per post. Every column has a **type** (text, number, date), and Postgres refuses data of the wrong type.

You talk to Postgres in **SQL**, for example `SELECT title FROM posts WHERE slug = 'hello';`.

Writing SQL by hand for every feature is slow and error-prone. An **ORM** (Object-Relational Mapper) is a library that lets you work with JavaScript objects and turns them into SQL for you. Payload's Postgres support uses an ORM called **Drizzle** under the hood.

An **adapter** is the plug between Payload and a specific database. Payload itself does not know SQL. You give it `postgresAdapter(...)` from `@payloadcms/db-postgres`, and the adapter (using Drizzle) translates Payload's operations into Postgres SQL. Swap in the MongoDB adapter and the same collections are stored in MongoDB instead.

```mermaid
flowchart LR
  C["Your code<br/>payload.find posts"] --> P["Payload<br/>access, hooks, validation"]
  P --> A["postgresAdapter<br/>@payloadcms/db-postgres"]
  A --> Z["Drizzle ORM<br/>builds SQL"]
  Z --> D[("Postgres 16<br/>tables and rows")]
```

You will rarely write SQL in this project. But you will peek at the tables, because seeing where your data really lives removes a lot of "magic".

### [Beginner] Step 13 — Run Postgres with Docker Compose

**What we're doing:** Writing a `docker-compose.yml` that starts Postgres 16 in a container, and starting it.

**Why:** The app needs a running database before it can start. **Docker Compose** reads a YAML file that describes one or more containers and starts them with one command. Putting the database config in a file means every developer gets an identical database by running `docker compose up -d`. A **volume** keeps the data on disk, so stopping the container does not delete your content.

**Do it:** The template may ship its own `docker-compose.yml` (some versions describe MongoDB, or run the whole app in a container). Replace its whole content with this:

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: my_site
    ports:
      - '5432:5432'
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U postgres -d my_site']
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  pgdata:
```

Line by line:

- `services:` lists the containers. We have one, named `db`.
- `image: postgres:16` uses the official Postgres image, major version 16.
- `environment:` sets variables the image reads on first start: the user, password and database name to create.
- `ports: '5432:5432'` connects port 5432 on your laptop to port 5432 in the container, so your app can reach it at `localhost:5432`.
- `volumes:` stores the database files in a named volume called `pgdata`, which survives restarts.
- `healthcheck:` lets Docker report when Postgres is actually ready to accept connections.

Start it. `-d` means "detached": run in the background and give me my terminal back.

```bash
docker compose up -d
```

**Check it works:**

```bash
docker compose ps
```

```text
NAME             IMAGE         SERVICE   STATUS                    PORTS
my-site-db-1     postgres:16   db        Up 10 seconds (healthy)   0.0.0.0:5432->5432/tcp
```

Wait until the status says `(healthy)`. Then open a SQL prompt inside the container:

```bash
docker compose exec db psql -U postgres -d my_site -c 'SELECT version();'
```

```text
                          version
-----------------------------------------------------------
 PostgreSQL 16.x on x86_64-pc-linux-gnu, compiled by ...
(1 row)
```

**What just happened:** Docker downloaded the Postgres image, created a container, and Postgres created an empty database called `my_site`. `docker compose exec db psql ...` ran the `psql` command-line client inside that container. Useful commands for later: `docker compose stop` pauses the database, `docker compose up -d` starts it again, and `docker compose down -v` deletes it **including all data** (the `-v` removes the volume).

**If it breaks:**

- `port is already allocated` or `address already in use`: another Postgres is running on 5432 (maybe one you installed earlier). Stop it, or change the left side to `'5433:5432'` and use port 5433 in your connection string.
- `Cannot connect to the Docker daemon`: start Docker Desktop.

### [Beginner] Step 14 — Set up environment variables

**What we're doing:** Filling in `.env` with three settings, and keeping a safe copy in `.env.example`.

**Why:** An **environment variable** is a named setting that lives outside your code, such as a database password. Your laptop, your teammate's laptop and the production server all need different values, but the same code. Putting them in `.env` keeps secrets out of Git. If you commit a real secret to GitHub, assume it is stolen: bots scan public repositories within minutes.

**Do it:** First, generate a long random secret. Payload uses `PAYLOAD_SECRET` to sign login tokens, so it must be hard to guess.

```bash
openssl rand -hex 32
```

```text
3f9c1a...64 hex characters...b72e
```

Open `.env` and make it look like this, with your own secret. The generator may have written `DATABASE_URI` already; some versions name it differently. Use these exact names, because the config in Step 15 reads them.

```bash
# .env
DATABASE_URI=postgres://postgres:postgres@localhost:5432/my_site
PAYLOAD_SECRET=paste-your-64-character-secret-here
NEXT_PUBLIC_SERVER_URL=http://localhost:3000
```

Now write `.env.example` with the same names but no real secret. This file **is** committed, so a new teammate knows what to set.

```bash
# .env.example
# Copy this file to .env and fill in real values.
DATABASE_URI=postgres://postgres:postgres@localhost:5432/my_site
PAYLOAD_SECRET=generate-with-openssl-rand-hex-32
NEXT_PUBLIC_SERVER_URL=http://localhost:3000
```

What each one means:

- `DATABASE_URI` is a **connection string**: `postgres://USER:PASSWORD@HOST:PORT/DATABASE`. It must match `docker-compose.yml`.
- `PAYLOAD_SECRET` signs and encrypts authentication data.
- `NEXT_PUBLIC_SERVER_URL` is the public address of the site. Variables starting with `NEXT_PUBLIC_` are copied into the browser JavaScript bundle by Next.js, so **never** put a secret in a `NEXT_PUBLIC_` variable.

**Check it works:**

```bash
git check-ignore .env && echo "ignored, good"
```

```text
.env
ignored, good
```

If nothing prints, open `.gitignore` and add a line `.env` (keep `!.env.example` style exceptions if your template has them).

**What just happened:** Next.js automatically loads `.env` when it starts, and the Payload command-line tool does too. The values reach your code as `process.env.DATABASE_URI` and so on. Production servers do not use a `.env` file; you type the same names into the hosting dashboard (guide 3).

### [Beginner] Step 15 — Read payload.config.ts line by line

**What we're doing:** Understanding the central config file, and adjusting it to read our environment variables.

**Why:** Every Payload feature is switched on in this file. Knowing what each line does is the difference between "it works and I do not know why" and being able to debug it.

**Do it:** Open `src/payload.config.ts`. It should look close to this. Make sure yours matches, especially the `db` block and the `secret`.

```ts
// src/payload.config.ts
import { postgresAdapter } from '@payloadcms/db-postgres'
import { lexicalEditor } from '@payloadcms/richtext-lexical'
import path from 'path'
import { buildConfig } from 'payload'
import { fileURLToPath } from 'url'
import sharp from 'sharp'

import { Users } from './collections/Users'
import { Media } from './collections/Media'

const filename = fileURLToPath(import.meta.url)
const dirname = path.dirname(filename)

export default buildConfig({
  admin: {
    user: Users.slug,
    importMap: {
      baseDir: path.resolve(dirname),
    },
  },
  collections: [Users, Media],
  editor: lexicalEditor(),
  secret: process.env.PAYLOAD_SECRET || '',
  typescript: {
    outputFile: path.resolve(dirname, 'payload-types.ts'),
  },
  db: postgresAdapter({
    pool: {
      connectionString: process.env.DATABASE_URI || '',
    },
  }),
  sharp,
  plugins: [],
})
```

What each part does:

- `buildConfig({...})` validates your config and fills in defaults.
- `admin.user: Users.slug` tells the admin panel which collection holds the people who can log in.
- `admin.importMap.baseDir` tells Payload where to look for custom admin components when it writes `importMap.js`.
- `collections: [...]` is the list of content types. We add three more in Part 4.
- `editor: lexicalEditor()` makes **Lexical** (a rich text editor from Meta) the default editor for every `richText` field.
- `secret` is your `PAYLOAD_SECRET`.
- `typescript.outputFile` is where `npm run generate:types` writes your types.
- `db: postgresAdapter({ pool: { connectionString } })` connects to Postgres. `pool` means Payload keeps a few connections open and reuses them, which is much faster than reconnecting for every query.
- `sharp` is an image processing library. Payload uses it to resize uploads into the image sizes we define in Step 22.

`import.meta.url` and `fileURLToPath` simply compute "the folder this file is in", because this project uses modern **ES modules** where the old `__dirname` variable does not exist.

**Check it works:** Hover over `buildConfig` in VS Code. A tooltip should show its type. If the editor shows red squiggles everywhere, run `npm install` again and restart VS Code.

**What just happened:** You saw that Payload is "config as code". There is no database table that defines your content model. The TypeScript file is the source of truth, and Payload builds the admin, the API and the database schema from it.

### [Beginner] Step 16 — Start the dev server for the first time

**What we're doing:** Running the app in development mode.

**Why:** The **dev server** watches your files and reloads the app when you save, so you see changes in seconds. It also connects to Postgres and creates the tables Payload needs.

**Do it:** Make sure the database is running (`docker compose ps`), then:

```bash
npm run dev
```

**Check it works:** The terminal should print something like:

```text
> my-site@1.0.0 dev
> cross-env NODE_OPTIONS=--no-deprecation next dev

   ▲ Next.js 16.x.x
   - Local:        http://localhost:3000
   - Environments: .env

 ✓ Ready in 2.1s
```

Open http://localhost:3000. You should see the blank template's welcome page. The first load can take 10 to 30 seconds because Next.js compiles the page on demand. Later loads are fast.

Leave this terminal running. Open a **second** terminal tab for other commands. To stop the server later, press **Ctrl+C** in the first tab.

**What just happened:** `npm run dev` looked up the `dev` script in `package.json` and ran `next dev`. When the first request touched Payload, Payload connected to Postgres and, because we are in development, **pushed** the schema: it created the `users`, `media` and a few internal `payload_*` tables automatically. We will talk about this "push mode" and why production uses migrations instead in Part 5.

**If it breaks:**

- `ECONNREFUSED 127.0.0.1:5432`: Postgres is not running. Run `docker compose up -d`.
- `password authentication failed for user "postgres"`: the password in `.env` does not match `docker-compose.yml`. If you changed the password after the first start, the old one is still stored in the volume. Run `docker compose down -v` and `docker compose up -d` to start fresh.
- `Error: missing secret key`: `PAYLOAD_SECRET` is empty. Check `.env` and restart the dev server.

### [Beginner] Step 17 — Create the first admin user

**What we're doing:** Opening the admin panel and creating the first account.

**Why:** The admin is protected by login. When there are no users yet, Payload shows a special "create first user" screen. Whoever creates the first user owns the admin, so do this right away on any new environment.

**Do it:** Open http://localhost:3000/admin. You are redirected to `/admin/create-first-user`. Enter an email and a password (at least the length the form asks for) and submit.

**Check it works:** You land on the admin dashboard with two collections in the sidebar: **Users** and **Media**. Click Users: your account is listed. Now confirm it really is in Postgres. In your second terminal:

```bash
docker compose exec db psql -U postgres -d my_site -c 'SELECT id, email, created_at FROM users;'
```

```text
 id |      email        |         created_at
----+-------------------+----------------------------
  1 | you@example.com   | 2026-10-06 09:12:44.123+00
(1 row)
```

Your password is not stored. Payload stores a **hash** (a one-way scrambled version) and a **salt** (random data mixed in before hashing) in columns named `hash` and `salt`. Even someone who steals the database cannot read the password back.

**What just happened:** The browser posted the form to Payload's REST API under `/api/users/first-register`. Payload validated it, hashed the password, inserted a row with Drizzle, and set a login **cookie** (a small piece of data the browser sends back on every request) containing a signed token. That cookie is how the admin knows it is you on the next click.

```mermaid
sequenceDiagram
  participant B as Browser
  participant N as Next.js route /api
  participant P as Payload
  participant D as Postgres
  B->>N: POST /api/users/first-register
  N->>P: run the operation
  P->>P: validate fields and hash password
  P->>D: INSERT INTO users
  D-->>P: new row with id 1
  P-->>B: Set-Cookie payload-token and user JSON
  B->>N: GET /admin with cookie
  N-->>B: dashboard HTML
```

### [Beginner] Step 18 — Add the scripts we will use

**What we're doing:** Adding named npm scripts for type checking, migrations and seeding.

**Why:** Scripts give long commands short, memorable names that are the same for every developer and for CI (the automated checks in guide 3). Nobody has to remember `payload migrate:create --name ...`; they run `npm run migrate:create`.

**Do it:** Open `package.json` and make the `scripts` section look like this. Keep any extra scripts your template already has, such as `devsafe`, `test:int` or `test:e2e`.

```json
{
  "scripts": {
    "dev": "cross-env NODE_OPTIONS=--no-deprecation next dev",
    "build": "cross-env NODE_OPTIONS=--no-deprecation next build",
    "start": "cross-env NODE_OPTIONS=--no-deprecation next start",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "e2e": "playwright test",
    "payload": "cross-env NODE_OPTIONS=--no-deprecation payload",
    "generate:types": "cross-env NODE_OPTIONS=--no-deprecation payload generate:types",
    "generate:importmap": "cross-env NODE_OPTIONS=--no-deprecation payload generate:importmap",
    "migrate:create": "cross-env NODE_OPTIONS=--no-deprecation payload migrate:create",
    "migrate": "cross-env NODE_OPTIONS=--no-deprecation payload migrate",
    "migrate:status": "cross-env NODE_OPTIONS=--no-deprecation payload migrate:status",
    "seed": "cross-env NODE_OPTIONS=--no-deprecation payload run src/seed.ts"
  }
}
```

Only the `scripts` object is shown. Do not delete the rest of `package.json` (name, dependencies and so on).

- `cross-env` sets an environment variable in a way that works on every operating system. `NODE_OPTIONS=--no-deprecation` hides noisy deprecation warnings.
- `typecheck` runs the TypeScript compiler without writing files, only to find type errors.
- `test` and `e2e` are wired up properly in guide 2. If Vitest or Playwright are not installed yet, those two scripts will fail for now. That is expected.
- `payload run <file>` runs a TypeScript file with your Payload config and `.env` already loaded. We use it for the seed script in Part 5.

**Check it works:**

```bash
npm run typecheck
```

```text
> my-site@1.0.0 typecheck
> tsc --noEmit
```

No output after that means no type errors.

**What just happened:** Your project now has one consistent vocabulary of commands. You will see these names in every later guide and in the CI pipeline.

### [Beginner] Step 19 — Make the first Git commit

**What we're doing:** Saving the working project as the first snapshot in Git, and optionally pushing it to GitHub.

**Why:** You now have a working baseline. If anything in Part 4 goes wrong, you can always return here. Commit early, commit often.

**Do it:** The generator may have already run `git init`. Check:

```bash
git status
```

If it says `fatal: not a git repository`, run `git init` first. Then:

```bash
git add .
git status
```

Read the list of files in green. **Make sure `.env` and `node_modules` are not in it.** If they are, fix `.gitignore` and run `git rm -r --cached .env node_modules`. Then:

```bash
git commit -m "chore: scaffold my-site with Payload, Next.js and Postgres"
```

To back it up on GitHub: create a new, empty, private repository called `my-site` on github.com (no README, no .gitignore), then run the commands GitHub shows, which look like:

```bash
git remote add origin https://github.com/YOUR-USERNAME/my-site.git
git push -u origin main
```

**Check it works:**

```bash
git log --oneline
```

```text
a1b2c3d (HEAD -> main) chore: scaffold my-site with Payload, Next.js and Postgres
```

**What just happened:** `git add` chose which changes go into the next snapshot (the **staging area**). `git commit` saved the snapshot with a message. `git push` copied your history to GitHub. In guide 3, pushing to GitHub is what triggers tests and deploys.

## 4. Model your content

### [Beginner] Concept — What is a collection?

A **collection** in Payload is a type of content, such as "posts". You describe it once in a TypeScript file, with a **slug** (its machine name, like `posts`) and a list of **fields** (like `title` and `content`). From that one description, Payload creates:

- a **database table** in Postgres (`posts`), with one column per simple field,
- an **admin UI**: a list view and an edit form at `/admin/collections/posts`,
- a **REST API** at `/api/posts` (and GraphQL),
- the **Local API**, `payload.find({ collection: 'posts' })`, for your server code,
- **TypeScript types** (`Post`) in `payload-types.ts`.

So the short version is: a collection is a database table with an admin UI and an API attached. Here is the content model we will build in this part. The lines show which collections point to which.

```mermaid
classDiagram
  class Users {
    email
    name
    password hash
  }
  class Media {
    alt
    url
    sizes
  }
  class Pages {
    title
    slug unique
    layout richText
  }
  class Posts {
    title
    slug unique
    excerpt
    publishedAt
    status
    content richText
  }
  class ContactSubmissions {
    name
    email
    message
  }
  Posts --> Media : coverImage
```

A **global** is the one-of-a-kind cousin of a collection (for example site settings). We do not need one yet.

> **Why:** Modelling content before building pages is the most important design step in a CMS project. Pages are easy to change later. A content model that editors already filled with 200 posts is not.

### [Beginner] Step 20 — Write shared access helpers

**What we're doing:** Creating two tiny functions, `anyone` and `authenticated`, that every collection will use for access control.

**Why:** **Access control** decides who may create, read, update or delete each document. Without it, anyone on the internet could read contact messages or delete your posts through `/api/...`. Writing the rules once in shared helpers means every collection uses the same, tested logic, instead of five slightly different copies.

**Do it:** Create a new folder `src/access` with two files.

```ts
// src/access/anyone.ts
import type { Access } from 'payload'

// Everyone may do this, logged in or not.
export const anyone: Access = () => true
```

```ts
// src/access/authenticated.ts
import type { Access } from 'payload'

// Only logged-in users (our admins) may do this.
// Payload puts the logged-in user on req.user, or null when nobody is logged in.
export const authenticated: Access = ({ req: { user } }) => Boolean(user)
```

**Check it works:**

```bash
npm run typecheck
```

```text
> tsc --noEmit
```

No errors means the `Access` type accepted both functions.

**What just happened:** An access function receives the request (with `req.user`) and returns `true` (allowed), `false` (denied), or a **query** that filters which documents are allowed. We use the query form for posts in Step 25. In this project, every logged-in user is an admin, so "logged in" is the only role check we need.

### [Beginner] Step 21 — The Users collection (authentication)

**What we're doing:** Upgrading the template's `Users` collection: adding a `name` field and locking it down.

**Why:** `auth: true` turns a collection into a login system: Payload adds `email` and password fields, login and logout endpoints, password reset, and session cookies. You should never build that yourself in a learning project; it is easy to get subtly wrong. But the default access is "any logged-in user", which we make explicit so nobody wonders.

**Do it:** Replace the whole file:

```ts
// src/collections/Users.ts
import type { CollectionConfig } from 'payload'

import { authenticated } from '../access/authenticated'

export const Users: CollectionConfig = {
  slug: 'users',
  admin: {
    useAsTitle: 'email',
    defaultColumns: ['name', 'email', 'updatedAt'],
  },
  // auth: true adds email, password (stored as hash and salt), login, logout,
  // forgot-password and the session cookie.
  auth: true,
  access: {
    create: authenticated,
    read: authenticated,
    update: authenticated,
    delete: authenticated,
  },
  fields: [
    // email is added automatically by auth: true
    {
      name: 'name',
      type: 'text',
    },
  ],
}
```

**Check it works:** Save the file. The dev server reloads. Open http://localhost:3000/admin/collections/users and click your user. A new **Name** field appears. Fill it in and save. Then, in a terminal that is **not** logged in:

```bash
curl -s http://localhost:3000/api/users
```

```text
{"errors":[{"message":"You are not allowed to perform this action."}]}
```

**What just happened:** `useAsTitle: 'email'` tells the admin which field to show as the document's title in lists and relationship pickers. `defaultColumns` picks the list view columns. The `curl` request had no login cookie, so `req.user` was `null`, `authenticated` returned `false`, and Payload answered with HTTP 403. Creating the very first user is a special case: the `create-first-user` screen works even with these rules, because there is nobody to log in as yet.

### [Beginner] Step 22 — The Media collection (uploads and image sizes)

**What we're doing:** Configuring `Media` as an upload collection that only accepts images and creates resized copies.

**Why:** Editors upload one big photo. The website needs a small thumbnail for lists, a medium card image, and a large hero image. Sending a 5000-pixel photo to a phone wastes data and makes the page slow. Payload can resize on upload using `sharp`, so the frontend can pick the right size. The `alt` field holds the text description screen readers read aloud, so we make it required for accessibility.

**Do it:** Replace the whole file:

```ts
// src/collections/Media.ts
import type { CollectionConfig } from 'payload'

import { anyone } from '../access/anyone'
import { authenticated } from '../access/authenticated'

export const Media: CollectionConfig = {
  slug: 'media',
  admin: {
    useAsTitle: 'alt',
  },
  access: {
    // Images appear on the public website, so anyone may read them.
    read: anyone,
    create: authenticated,
    update: authenticated,
    delete: authenticated,
  },
  fields: [
    {
      name: 'alt',
      type: 'text',
      required: true,
      admin: {
        description: 'Describe the image for people using screen readers.',
      },
    },
  ],
  upload: {
    // Only accept images.
    mimeTypes: ['image/*'],
    // Let editors pick the most important point, so crops keep it in frame.
    focalPoint: true,
    // Which size to show in the admin list.
    adminThumbnail: 'thumbnail',
    imageSizes: [
      { name: 'thumbnail', width: 400, height: 300, position: 'centre' },
      { name: 'card', width: 768, height: 512, position: 'centre' },
      // No height: keep the original aspect ratio.
      { name: 'hero', width: 1600 },
    ],
  },
}
```

**Check it works:** Open http://localhost:3000/admin/collections/media, click **Create New**, upload any photo, fill in Alt, and save. Then look at the files on disk:

```bash
ls media
```

```text
my-photo.jpg  my-photo-400x300.jpg  my-photo-768x512.jpg  my-photo-1600x1067.jpg
```

The exact names depend on your photo. In some versions the folder is created elsewhere; the media document in the admin shows the URL of every size.

**What just happened:** Payload saved the original file to a local folder (by default named after the collection, `media`, and already in `.gitignore` in the template), ran `sharp` three times to make the sizes, and stored one row in the `media` table with the filename, dimensions, MIME type and the URL of each size. Local disk is fine for development. In production, servers are often temporary and their disks vanish, so guide 3 switches this collection to S3 or Vercel Blob storage with a storage adapter. Your collection code will not change.

**If it breaks:** `Error: Could not load the "sharp" module`: run `npm install sharp` and make sure `sharp` is passed in `payload.config.ts` (Step 15).

### [Beginner] Step 23 — A reusable slug field with an auto-format hook

**What we're doing:** Writing a `slugField()` helper that adds a unique, indexed `slug` field and fills it in from the title automatically.

**Why:** Pages and posts both need a slug, and their rules are identical. A slug must be unique (two posts at `/blog/hello` cannot both win), and it must be URL-safe (no spaces, no capitals, no accents). Editors forget. A **hook** is a function Payload runs at a certain moment, here "just before validation". Doing the formatting in a server-side hook means the rule is enforced no matter how the data arrives: the admin, the REST API, or a seed script.

**Do it:** First the pure formatting function. "Pure" means it only turns input into output, which makes it easy to unit test in guide 3.

```ts
// src/utilities/slugify.ts
// Turn any text into a URL-safe slug: "Hello, World!" -> "hello-world"
export const slugify = (input: string): string =>
  input
    .normalize('NFKD') // split letters from their accents: "é" -> "e" + accent
    .replace(/[̀-ͯ]/g, '') // drop the accents
    .toLowerCase()
    .trim()
    .replace(/[^a-z0-9\s-]/g, '') // remove anything that is not a letter, digit, space or dash
    .replace(/[\s_-]+/g, '-') // spaces and repeated dashes become one dash
    .replace(/^-+|-+$/g, '') // no dash at the start or end
```

Then the field helper with its hook:

```ts
// src/fields/slug.ts
import type { Field, FieldHook } from 'payload'

import { slugify } from '../utilities/slugify'

// Runs on the server before validation, on every create and update.
const formatSlug =
  (fallbackField: string): FieldHook =>
  ({ value, data, originalDoc }) => {
    // 1. The editor typed a slug: clean it up.
    if (typeof value === 'string' && value.trim().length > 0) {
      return slugify(value)
    }
    // 2. No slug: build one from the fallback field (usually the title).
    const fallback: unknown = data?.[fallbackField] ?? originalDoc?.[fallbackField]
    if (typeof fallback === 'string' && fallback.trim().length > 0) {
      return slugify(fallback)
    }
    // 3. Nothing to build from: leave it as it is.
    return value
  }

export const slugField = (fallbackField = 'title'): Field => ({
  name: 'slug',
  type: 'text',
  // unique: true creates a UNIQUE index in Postgres, so duplicates are impossible.
  unique: true,
  // index: true makes lookups by slug fast. That is exactly what /blog/[slug] does.
  index: true,
  admin: {
    position: 'sidebar',
    description: 'The URL part, for example "about-us". Leave empty to generate it from the title.',
  },
  hooks: {
    beforeValidate: [formatSlug(fallbackField)],
  },
})
```

**Check it works:** We test it for real once Pages exists in the next step. For now:

```bash
npm run typecheck
```

```text
> tsc --noEmit
```

**What just happened:** You wrote a **field hook**. Payload calls every `beforeValidate` hook of a field with the incoming `value`, the whole incoming document as `data`, and the stored version as `originalDoc` (on updates). Whatever the hook returns becomes the new value. We did not mark the field `required`, on purpose: the admin form checks required fields in the browser **before** the server hook can fill in the slug, so a required slug would force editors to type it. The hook plus the unique index give us the guarantee we need.

> **Gotcha:** Changing a slug changes a public URL. Old links and search results break. In a real project you would add redirects; for now, tell editors to pick slugs carefully.

### [Beginner] Step 24 — The Pages collection

**What we're doing:** Creating the `pages` collection: a title, a slug, and a rich text `layout` field for the body.

**Why:** Pages like Home, About and Services are what the business owner edits most. We name the body field `layout` because many Payload projects later turn it into a **blocks** field (a page builder where editors stack sections like "hero", "text", "call to action"). Starting with rich text keeps guide 2 simple, and the name stays correct when you upgrade.

**Do it:**

```ts
// src/collections/Pages.ts
import type { CollectionConfig } from 'payload'

import { anyone } from '../access/anyone'
import { authenticated } from '../access/authenticated'
import { slugField } from '../fields/slug'

export const Pages: CollectionConfig = {
  slug: 'pages',
  admin: {
    useAsTitle: 'title',
    defaultColumns: ['title', 'slug', 'updatedAt'],
    description: 'Website pages. The page with slug "home" is shown at /.',
  },
  access: {
    read: anyone,
    create: authenticated,
    update: authenticated,
    delete: authenticated,
  },
  fields: [
    {
      name: 'title',
      type: 'text',
      required: true,
      maxLength: 120,
    },
    slugField('title'),
    {
      name: 'layout',
      type: 'richText',
      // No editor here, so it uses lexicalEditor() from payload.config.ts.
    },
  ],
}
```

Register it in `src/payload.config.ts`. Only two lines change: one import, and the `collections` array.

```ts
// src/payload.config.ts (only the changed lines)
import { Pages } from './collections/Pages'

// inside buildConfig({ ... })
  collections: [Users, Media, Pages],
```

**Check it works:** The dev server reloads and, in push mode, creates the `pages` table. In the admin, open **Pages**, click **Create New**, type the title `About Us`, leave the slug empty, write a sentence in Layout, and save. The slug in the sidebar now says `about-us`. Now try to create a second page with the title `About us!`. Saving fails with an error that the slug value must be unique.

**What just happened:** Your hook turned "About Us" into `about-us` on the server, and the Postgres unique index rejected the duplicate. Two layers, two different jobs: the hook makes the value **correct**, the index makes duplicates **impossible**, even if two editors click save in the same millisecond.

### [Beginner] Concept — How access control decides

Every request to Payload passes through the access function for that operation. The function can answer in three ways.

```mermaid
flowchart TD
  R["Request to read posts"] --> A{"Access function<br/>result?"}
  A -->|"true"| ALL["Return all matching documents"]
  A -->|"false"| E["403 not allowed"]
  A -->|"a query object"| W["Add the query as an extra WHERE<br/>return only allowed documents"]
```

The third answer is powerful. For posts, we will say: logged-in users see everything; everyone else sees only documents where `status` equals `published`. Payload adds that condition to the SQL, so drafts never leave the database for anonymous visitors.

> **Gotcha:** The **Local API** (`payload.find(...)` in your server code) **skips access control by default** (`overrideAccess` defaults to `true`), because it assumes your server code is trusted. That is why the seed script in Part 5 works without logging in. In guide 2, when the website reads posts, we will filter by `status` ourselves, or pass `overrideAccess: false`. The REST API at `/api/...` always enforces access.

### [Intermediate] Step 25 — The Posts collection

**What we're doing:** Creating `posts` with a title, slug, excerpt, cover image, publish date, status and rich text content, plus a hook and a filtering access rule.

**Why:** The blog is the most "data-like" part of the site. Posts are listed, sorted by date, filtered by status, and shown with an image. Each of those needs a field with the right type. The status field lets editors save unfinished work as a **draft** that the public cannot see.

**Do it:**

```ts
// src/collections/Posts.ts
import type { Access, CollectionBeforeChangeHook, CollectionConfig } from 'payload'

import { authenticated } from '../access/authenticated'
import { slugField } from '../fields/slug'

// Logged-in users see every post. Visitors only see published ones.
const publishedOrLoggedIn: Access = ({ req: { user } }) => {
  if (user) return true
  return {
    status: {
      equals: 'published',
    },
  }
}

// When a post is published without a date, stamp it with "now".
const setPublishedAt: CollectionBeforeChangeHook = ({ data }) => {
  if (data.status === 'published' && !data.publishedAt) {
    data.publishedAt = new Date().toISOString()
  }
  return data
}

export const Posts: CollectionConfig = {
  slug: 'posts',
  admin: {
    useAsTitle: 'title',
    defaultColumns: ['title', 'status', 'publishedAt', 'updatedAt'],
  },
  access: {
    read: publishedOrLoggedIn,
    create: authenticated,
    update: authenticated,
    delete: authenticated,
  },
  // Newest first in the admin list.
  defaultSort: '-publishedAt',
  hooks: {
    beforeChange: [setPublishedAt],
  },
  fields: [
    {
      name: 'title',
      type: 'text',
      required: true,
      maxLength: 160,
    },
    slugField('title'),
    {
      name: 'excerpt',
      type: 'textarea',
      admin: {
        description: 'A short summary shown in the blog list. Up to 40 words.',
      },
      // A custom validator: return true when valid, or an error message.
      validate: (value: string | null | undefined) => {
        if (!value) return true
        const words = value.trim().split(/\s+/).length
        return words <= 40 || `Keep the excerpt to 40 words or fewer (now ${words}).`
      },
    },
    {
      // An upload field is a relationship to an upload collection.
      name: 'coverImage',
      type: 'upload',
      relationTo: 'media',
    },
    {
      name: 'publishedAt',
      type: 'date',
      index: true,
      admin: {
        position: 'sidebar',
        date: {
          pickerAppearance: 'dayAndTime',
        },
      },
    },
    {
      name: 'status',
      type: 'select',
      required: true,
      defaultValue: 'draft',
      index: true,
      options: [
        { label: 'Draft', value: 'draft' },
        { label: 'Published', value: 'published' },
      ],
      admin: {
        position: 'sidebar',
      },
    },
    {
      name: 'content',
      type: 'richText',
      required: true,
    },
  ],
}
```

Register it:

```ts
// src/payload.config.ts (only the changed lines)
import { Posts } from './collections/Posts'

// inside buildConfig({ ... })
  collections: [Users, Media, Pages, Posts],
```

**Check it works:** In the admin, create a post titled `Draft idea`, leave status as **Draft**, add content, and save. Create a second post `Hello world`, pick the cover image you uploaded, set status to **Published**, and save. Its **Published At** date fills in by itself. Now ask the REST API as an anonymous visitor:

```bash
curl -s 'http://localhost:3000/api/posts?depth=0'
```

You can also open http://localhost:3000/api/posts in a **private browser window** (where you are not logged in). You should see `"totalDocs": 1` and only `Hello world`. In your normal, logged-in window, the same URL shows `"totalDocs": 2`.

```text
{"docs":[{"id":2,"title":"Hello world","slug":"hello-world","status":"published","coverImage":1, ...}],"totalDocs":1, ...}
```

**What just happened:** The access function returned a query for the anonymous request, and Payload turned it into `WHERE status = 'published'`. The `beforeChange` **collection hook** ran after validation and before the database write, and it set the date. `?depth=0` told the API to return the cover image as its plain id (`1`); with the default depth, Payload **populates** relationships and returns the whole media document instead.

> **Interview tip:** Payload also has built-in drafts (`versions: { drafts: true }`), which adds a `_status` field, full version history and autosave. We used a plain `status` select to keep the moving parts visible. Mention both in an interview and explain the trade-off: built-in drafts give history and preview, the plain field is simpler and has fewer tables.

### [Beginner] Step 26 — The ContactSubmissions collection

**What we're doing:** Creating `contact-submissions` to store messages from the contact form, where the public can create but only admins can read.

**Why:** A contact form is the one place where anonymous visitors **write** to your database. That is the opposite of every other collection, so the access rules are inverted: open to create, closed to read. If you got this backwards, anyone could download every customer's name, email and message, which is a serious privacy leak.

**Do it:**

```ts
// src/collections/ContactSubmissions.ts
import type { CollectionConfig } from 'payload'

import { anyone } from '../access/anyone'
import { authenticated } from '../access/authenticated'

export const ContactSubmissions: CollectionConfig = {
  slug: 'contact-submissions',
  labels: {
    singular: 'Contact submission',
    plural: 'Contact submissions',
  },
  admin: {
    useAsTitle: 'name',
    defaultColumns: ['name', 'email', 'createdAt'],
    description: 'Messages sent through the /contact form.',
  },
  access: {
    // Visitors may send a message.
    create: anyone,
    // Only admins may read or delete them.
    read: authenticated,
    delete: authenticated,
    // Nobody edits a customer's message after it was sent.
    update: () => false,
  },
  fields: [
    {
      name: 'name',
      type: 'text',
      required: true,
      maxLength: 100,
    },
    {
      name: 'email',
      type: 'email',
      required: true,
    },
    {
      name: 'message',
      type: 'textarea',
      required: true,
      minLength: 10,
      maxLength: 5000,
    },
  ],
}
```

Register it. This is the final `collections` list for this guide:

```ts
// src/payload.config.ts (only the changed lines)
import { ContactSubmissions } from './collections/ContactSubmissions'

// inside buildConfig({ ... })
  collections: [Users, Media, Pages, Posts, ContactSubmissions],
```

**Check it works:** Send a message as an anonymous visitor, exactly as a form would:

```bash
curl -s -X POST http://localhost:3000/api/contact-submissions \
  -H 'Content-Type: application/json' \
  -d '{"name":"Ada","email":"ada@example.com","message":"Hello, I would like a quote."}'
```

```text
{"doc":{"id":1,"name":"Ada","email":"ada@example.com","message":"Hello, I would like a quote.", ...},"message":"Contact submission successfully created."}
```

Now try to read them anonymously:

```bash
curl -s http://localhost:3000/api/contact-submissions
```

```text
{"errors":[{"message":"You are not allowed to perform this action."}]}
```

Then send an invalid one:

```bash
curl -s -X POST http://localhost:3000/api/contact-submissions \
  -H 'Content-Type: application/json' \
  -d '{"name":"Ada","email":"not-an-email","message":"Hi"}'
```

```text
{"errors":[{"name":"ValidationError","data":{"errors":[{"message":"Please enter a valid email address.","path":"email"},{"message":"This value must be longer than the minimum length of 10 characters.","path":"message"}]}, ...}]}
```

The exact error wording may differ between versions. In the admin, **Contact submissions** shows Ada's message.

**What just happened:** The same collection gives different answers to different people, based only on the access functions. The `email` field type and `minLength`/`maxLength` gave us validation without writing code. In guide 2, the `/contact` page will call Payload from a **server action** and validate with Zod first, so visitors get friendly errors before anything reaches Payload.

### [Beginner] Step 27 — Generate TypeScript types

**What we're doing:** Asking Payload to write TypeScript types for all collections.

**Why:** In guide 2, your pages will read posts. With generated types, `post.title` autocompletes and `post.titel` is a red error in the editor, instead of `undefined` on the live site.

**Do it:**

```bash
npm run generate:types
```

**Check it works:** Open `src/payload-types.ts` and search for `interface Post`. You should see something close to:

```ts
// src/payload-types.ts (generated, excerpt)
export interface Post {
  id: number;
  title: string;
  slug?: string | null;
  excerpt?: string | null;
  coverImage?: (number | null) | Media;
  publishedAt?: string | null;
  status: 'draft' | 'published';
  content: {
    root: {
      type: string;
      children: {
        type: any;
        version: number;
        [k: string]: unknown;
      }[];
      direction: ('ltr' | 'rtl') | null;
      format: 'left' | 'start' | 'center' | 'right' | 'end' | 'justify' | '';
      indent: number;
      version: number;
    };
    [k: string]: unknown;
  };
  updatedAt: string;
  createdAt: string;
}
```

**What just happened:** Payload read your config and wrote one interface per collection. Notice three things. `id` is a `number` because Postgres uses auto-incrementing integer ids. `coverImage` is `number | Media` because it is either an id or the populated document, depending on `depth`. `slug` is optional because we did not mark it required. Rerun this command every time you change a collection. Some Payload versions also regenerate types automatically when the dev server starts; running it yourself is always safe.

### [Beginner] Step 28 — Field types and validation, in one place

**What we're doing:** Reviewing the field types you used and the ones you will meet next, and how validation works.

**Why:** Choosing the right field type gives you the right admin input, the right database column, and free validation. Choosing the wrong one (say, `text` for a date) means sorting and filtering break later.

**Do it:** Read this reference table. Postgres column types are approximate and may vary by version.

| Field type | Admin input | Stored in Postgres as | Used for |
| --- | --- | --- | --- |
| `text` | single line | `varchar` column | titles, names, slugs |
| `textarea` | multi line | `varchar` column | excerpts, messages |
| `email` | email input | `varchar` column, email format checked | contact email |
| `number` | number input | `numeric` column | prices, counts |
| `checkbox` | toggle | `boolean` column | "featured" flags |
| `date` | date picker | `timestamp with time zone` | publishedAt |
| `select` | dropdown | a Postgres `enum` type | status |
| `richText` | Lexical editor | `jsonb` column (a JSON tree) | page and post bodies |
| `upload` | file picker | `<name>_id` foreign key to the upload table | coverImage |
| `relationship` | document picker | `<name>_id` foreign key, or a `_rels` table for many | author, categories |
| `array` | repeatable rows | a separate child table | gallery items, FAQ |
| `group` | nested fields | prefixed columns | SEO fields |
| `blocks` | page builder | one table per block type | flexible layouts |

Validation happens in layers:

1. **Built-in options**: `required`, `unique`, `minLength`, `maxLength`, `min`, `max`, and the type itself (`email`).
2. **Custom `validate` functions**: return `true` or an error message, like our excerpt rule. A custom `validate` **replaces** the built-in validator of that field, so if you add one to a `required` field, check for an empty value yourself.
3. **Database constraints**: `NOT NULL` and `UNIQUE` in Postgres, the last line of defence.

Payload runs validation in the admin form (fast feedback for editors) and **always again on the server** (the only check you can trust, because anyone can call the API directly).

**Check it works:** In the admin, edit `Hello world` and paste a 50-word excerpt. Click Save. You should see your custom message under the field:

```text
Keep the excerpt to 40 words or fewer (now 50).
```

**What just happened:** You saw the whole validation story: field options, a custom rule, and the database index from Step 24, each catching a different kind of mistake.

```mermaid
sequenceDiagram
  participant E as Editor
  participant A as Admin form
  participant P as Payload server
  participant D as Postgres
  E->>A: click Save
  A->>A: client validation
  A->>P: PATCH /api/posts/2
  P->>P: access check
  P->>P: beforeValidate hooks then validate
  P->>P: beforeChange hooks
  P->>D: UPDATE posts
  D-->>P: ok or unique violation
  P->>P: afterChange hooks
  P-->>A: updated document or errors
```

The last box, **afterChange hooks**, is where guide 2 calls `revalidatePath` so the public website updates right after an editor saves.

### [Beginner] Step 29 — Peek at the tables in Postgres

**What we're doing:** Looking at what Payload created in the database.

**Why:** Payload is not magic. It writes ordinary tables. Seeing them helps you debug ("is the data actually saved?"), write reports, and answer interview questions about how a CMS maps to SQL.

**Do it:** Open `psql` inside the container. This time without `-c`, so you get an interactive prompt:

```bash
docker compose exec db psql -U postgres -d my_site
```

At the `my_site=#` prompt, type these. Commands that start with a backslash are `psql` shortcuts, not SQL.

```sql
\dt
\d posts
SELECT id, title, slug, status, published_at FROM posts;
SELECT id, filename, sizes_thumbnail_url FROM media;
\q
```

**Check it works:** `\dt` lists the tables. You should see your collections plus some internal ones:

```text
                 List of relations
 Schema |              Name              | Type  |  Owner
--------+--------------------------------+-------+----------
 public | contact_submissions            | table | postgres
 public | media                          | table | postgres
 public | pages                          | table | postgres
 public | payload_locked_documents       | table | postgres
 public | payload_locked_documents_rels  | table | postgres
 public | payload_migrations             | table | postgres
 public | payload_preferences            | table | postgres
 public | payload_preferences_rels       | table | postgres
 public | posts                          | table | postgres
 public | users                          | table | postgres
 ...
```

`\d posts` describes the `posts` table:

```text
      Column      |            Type             | Nullable
------------------+-----------------------------+----------
 id               | integer                     | not null
 title            | character varying           | not null
 slug             | character varying           |
 excerpt          | character varying           |
 cover_image_id   | integer                     |
 published_at     | timestamp(3) with time zone |
 status           | enum_posts_status           | not null
 content          | jsonb                       | not null
 updated_at       | timestamp(3) with time zone | not null
 created_at       | timestamp(3) with time zone | not null
Indexes:
    "posts_pkey" PRIMARY KEY, btree (id)
    "posts_slug_idx" UNIQUE, btree (slug)
    ...
Foreign-key constraints:
    "posts_cover_image_id_media_id_fk" FOREIGN KEY (cover_image_id) REFERENCES media(id) ON DELETE SET NULL
```

Your output may show a few more indexes or slightly different names. If you prefer clicking, connect your VS Code PostgreSQL extension or a GUI such as TablePlus to `localhost:5432`, user `postgres`, password `postgres`, database `my_site`.

**What just happened:** You saw the mapping from Step 28 for real. Field names in camelCase (`coverImage`, `publishedAt`) became snake_case columns (`cover_image_id`, `published_at`). The select became a Postgres **enum** type. Rich text is a JSON tree in a `jsonb` column. The upload field became a **foreign key**: Postgres itself guarantees that `cover_image_id` points at a real media row. Image sizes became flattened columns such as `sizes_thumbnail_url`. The `payload_*` tables hold admin preferences, document locking (so two editors do not overwrite each other) and, soon, migration history.

### [Beginner] Step 30 — Commit the content model

**What we're doing:** Saving the content model in Git.

**Why:** The content model is the most valuable code in the project. Commit it as one clear, reviewable change.

**Do it:**

```bash
git add .
git commit -m "feat: add users, media, pages, posts and contact submissions collections"
```

**Check it works:** Your project now looks like this:

```text
my-site/
├── docker-compose.yml
├── .env.example
├── package.json
└── src/
    ├── access/
    │   ├── anyone.ts
    │   └── authenticated.ts
    ├── app/
    │   ├── (frontend)/...
    │   └── (payload)/...
    ├── collections/
    │   ├── ContactSubmissions.ts
    │   ├── Media.ts
    │   ├── Pages.ts
    │   ├── Posts.ts
    │   └── Users.ts
    ├── fields/
    │   └── slug.ts
    ├── utilities/
    │   └── slugify.ts
    ├── payload-types.ts
    └── payload.config.ts
```

**What just happened:** Notice what is **not** committed: your database. The tables exist only in your local Postgres, created by push mode. A teammate who clones the repo, or a production server, has no way to recreate them exactly. That is the problem Part 5 solves.

## 5. Migrations and seed data

### [Beginner] Concept — What is a database migration?

Your code lives in Git, so every copy of the project has the same code. But each **database** is separate: your laptop's Postgres, a teammate's, the production one. When you add a field, every one of those databases needs a new column, and in the same order.

A **migration** is a small file of code that changes a database's structure one step forward, for example "create table posts" or "add column excerpt". Migrations have a timestamp in their name, so they run in order, and the database records which ones it already ran (Payload uses the `payload_migrations` table for that). Each migration has two functions:

- `up`: apply the change.
- `down`: undo it.

Because migration files are committed to Git, every database can be brought to exactly the same structure by running "all migrations I have not run yet".

```mermaid
flowchart LR
  C["Change a collection<br/>in TypeScript"] --> M["npm run migrate:create<br/>writes a migration file"]
  M --> G["Commit the file<br/>to Git"]
  G --> T["Teammate or CI<br/>npm run migrate"]
  G --> P["Production deploy<br/>npm run migrate"]
  T --> S["Same schema everywhere"]
  P --> S
```

### [Beginner] Step 31 — Understand dev push mode versus migrations

**What we're doing:** Learning the two ways Payload can change your Postgres schema, and when to use each.

**Why:** Mixing them up is the most common Payload + Postgres mistake. Using push in production can **drop columns and their data** without asking. Using only migrations locally makes quick experiments slow.

**Do it:** Read and compare.

| | Push mode | Migrations |
| --- | --- | --- |
| How it works | On dev server start, Drizzle compares your config with the database and changes the database to match | You generate a file with the exact SQL, review it, commit it, and run it |
| When it runs | Automatically, in development only | When you run `npm run migrate` |
| Good for | Fast local experiments | Teammates, CI, staging, production |
| Danger | Can drop a column (and its data) if you rename or remove a field | Little: you can read the SQL before it runs |
| Recorded where | A special `dev` entry in `payload_migrations` | One row per migration in `payload_migrations` |

The `postgresAdapter` uses push mode automatically when `NODE_ENV` is not `production`. You can turn it off with `push: false` in the adapter options if you want migrations only, even locally.

The workflow we will use for the rest of the series:

1. Change collections freely while the dev server runs. Push keeps your local database in sync.
2. When the change is finished, run `npm run migrate:create` to capture it in a migration file.
3. Commit the migration together with the collection change.
4. Every other environment runs `npm run migrate`. In guide 3, the deploy does this automatically.

```mermaid
stateDiagram-v2
  [*] --> Editing: change a collection
  Editing --> Pushed: dev server pushes schema
  Pushed --> Editing: keep experimenting
  Pushed --> Captured: npm run migrate create
  Captured --> Committed: git commit
  Committed --> Deployed: npm run migrate in CI or prod
  Deployed --> [*]
```

**Check it works:** Answer this: you renamed `excerpt` to `summary` locally and the dev server warned about data loss. What protects production? (A migration you generate and **read** before committing. You can edit it to rename the column instead of dropping it.)

**What just happened:** You learned the rule: **push for your laptop, migrations for everything else.**

> **Gotcha:** If you run `npm run migrate` against a database that was changed by push mode, Payload notices the `dev` entry and asks whether to continue, because the tables may already exist. For a clean result, run migrations against a fresh database, which is exactly what we do in Step 33.

### [Beginner] Step 32 — Create the first migration

**What we're doing:** Generating a migration that creates every table for our five collections.

**Why:** Right now, the only description of the schema that a new database could use is "start the dev server and hope push does the right thing". A migration file is an exact, reviewable, repeatable recipe.

**Do it:** Stop the dev server (Ctrl+C in its terminal). Then:

```bash
npm run migrate:create -- initial
```

The `--` passes `initial` through npm to the Payload command, as the migration name.

**Check it works:**

```text
[09:15:02] INFO: Migration created at /home/you/code/my-site/src/migrations/20261006_091502_initial.ts
```

```bash
ls src/migrations
```

```text
20261006_091502_initial.json  20261006_091502_initial.ts  index.ts
```

Open the `.ts` file. It looks like this (shortened):

```ts
// src/migrations/20261006_091502_initial.ts (generated, shortened)
import { MigrateDownArgs, MigrateUpArgs, sql } from '@payloadcms/db-postgres'

export async function up({ db, payload, req }: MigrateUpArgs): Promise<void> {
  await db.execute(sql`
   CREATE TYPE "public"."enum_posts_status" AS ENUM('draft', 'published');
   CREATE TABLE "posts" (
     "id" serial PRIMARY KEY NOT NULL,
     "title" varchar NOT NULL,
     "slug" varchar,
     ...
   );
   CREATE UNIQUE INDEX "posts_slug_idx" ON "posts" USING btree ("slug");
   ...`)
}

export async function down({ db, payload, req }: MigrateDownArgs): Promise<void> {
  await db.execute(sql`
   DROP TABLE "posts" CASCADE;
   ...`)
}
```

**What just happened:** Payload compared your config with an empty starting point (there were no previous migrations) and wrote the SQL to build everything. The `.json` file is a **snapshot** of the schema; the next `migrate:create` compares against it, so the next migration only contains the difference. `index.ts` lists all migrations in order, which guide 3 uses to run them on deploy. Never edit a migration after it has run somewhere other than your laptop; write a new one instead.

### [Beginner] Step 33 — Run the migration on a fresh database

**What we're doing:** Deleting the local database, recreating it empty, and building it with the migration alone.

**Why:** This proves the migration is complete. It is exactly what will happen on a new teammate's laptop or a new production database. If something is missing from the migration, you want to find out now, not during a deploy.

**Do it:** This deletes all local data (your test posts and your admin user). That is fine; the seed script will recreate content.

```bash
docker compose down -v
docker compose up -d
docker compose ps
```

Wait until the status says `(healthy)`. Then:

```bash
npm run migrate
```

**Check it works:**

```text
[09:17:40] INFO: Reading migration files from /home/you/code/my-site/src/migrations
[09:17:40] INFO: Migrating: 20261006_091502_initial
[09:17:41] INFO: Migrated:  20261006_091502_initial (412ms)
[09:17:41] INFO: Done.
```

```bash
npm run migrate:status
```

```text
┌───────────────────────────────┬───────┬─────┐
│ Name                          │ Batch │ Ran │
├───────────────────────────────┼───────┼─────┤
│ 20261006_091502_initial       │ 1     │ Yes │
└───────────────────────────────┴───────┴─────┘
```

Start the dev server again with `npm run dev`, open http://localhost:3000/admin, and create your admin user again, since the old database is gone.

**What just happened:** `payload migrate` read `src/migrations`, saw that `payload_migrations` had no record of `initial`, ran its `up` function, and recorded it in **batch 1**. A batch is a group of migrations run together; `npm run payload migrate:down` would undo the last batch. When the dev server started afterwards, push mode compared the config with the database, found nothing to change, and did nothing.

**If it breaks:**

- `relation "posts" already exists`: the database was not fresh. Run `docker compose down -v` again, and make sure the dev server was stopped so it could not push in between.
- `ECONNREFUSED`: Postgres was still starting. Wait for `(healthy)` and retry.

### [Beginner] Concept — What is seed data?

**Seed data** is starter content inserted by a script: a home page, a couple of posts. It means that any fresh database (yours after a reset, a teammate's, a preview environment, or the end-to-end tests in guide 3) has something to show immediately. Without it, every developer clicks through the admin by hand to create test content, and everyone's test content is different.

A good seed script is **idempotent**: running it twice does not create duplicates. Ours checks whether a document with the same slug exists before creating it.

```mermaid
sequenceDiagram
  participant T as Terminal
  participant S as seed.ts
  participant P as Payload Local API
  participant D as Postgres
  T->>S: npm run seed
  S->>P: getPayload with config
  S->>P: find pages where slug is home
  P->>D: SELECT from pages
  D-->>P: no rows
  S->>P: create page home
  P->>D: INSERT INTO pages
  S->>P: find and create two posts
  S-->>T: Seed complete
```

### [Intermediate] Step 34 — Write the seed script

**What we're doing:** Writing `src/seed.ts`, which creates the home page and two published posts if they do not exist.

**Why:** Guide 2 renders the page with slug `home` at `/` and lists posts at `/blog`. With seed data, those pages show real content the first time you open them.

**Do it:**

```ts
// src/seed.ts
import config from '@payload-config'
import { getPayload } from 'payload'

import type { Post } from './payload-types'

// Lexical stores rich text as a JSON tree. This helper builds a tree
// with one paragraph per string, so we can write content in plain text.
type RichText = Post['content']

const richText = (...paragraphs: string[]): RichText => ({
  root: {
    type: 'root',
    format: '',
    indent: 0,
    version: 1,
    direction: 'ltr',
    children: paragraphs.map((text) => ({
      type: 'paragraph',
      format: '',
      indent: 0,
      version: 1,
      direction: 'ltr',
      textFormat: 0,
      children: [
        {
          type: 'text',
          text,
          format: 0,
          style: '',
          mode: 'normal',
          detail: 0,
          version: 1,
        },
      ],
    })),
  },
})

type SeedPost = Pick<Post, 'title' | 'slug' | 'excerpt' | 'status' | 'publishedAt' | 'content'>

const seedPosts: SeedPost[] = [
  {
    title: 'Welcome to our new website',
    slug: 'welcome-to-our-new-website',
    excerpt: 'We rebuilt our website to make it faster and easier to find what you need.',
    status: 'published',
    publishedAt: '2026-10-01T09:00:00.000Z',
    content: richText(
      'We are excited to share our new website with you.',
      'You can now read our latest news on the blog and reach us through the contact form.',
    ),
  },
  {
    title: 'Five tips for choosing the right service',
    slug: 'five-tips-for-choosing-the-right-service',
    excerpt: 'A short checklist to help you decide what you really need before you ask for a quote.',
    status: 'published',
    publishedAt: '2026-10-03T09:00:00.000Z',
    content: richText(
      'Start with the problem, not the product. Write down what you want to change.',
      'Then set a budget, ask for references, and compare at least two quotes.',
    ),
  },
]

async function seed(): Promise<void> {
  const payload = await getPayload({ config })

  // 1. The home page, shown at / in guide 2.
  const home = await payload.find({
    collection: 'pages',
    where: { slug: { equals: 'home' } },
    limit: 1,
  })

  if (home.docs.length === 0) {
    await payload.create({
      collection: 'pages',
      data: {
        title: 'Home',
        slug: 'home',
        layout: richText(
          'Welcome to My Site. We help small businesses grow.',
          'Read our blog for news and tips, or get in touch through the contact page.',
        ),
      },
    })
    payload.logger.info('Created page: home')
  } else {
    payload.logger.info('Page "home" already exists, skipping')
  }

  // 2. Two published posts, shown at /blog in guide 2.
  for (const post of seedPosts) {
    const existing = await payload.find({
      collection: 'posts',
      where: { slug: { equals: post.slug } },
      limit: 1,
    })

    if (existing.docs.length > 0) {
      payload.logger.info(`Post "${post.slug}" already exists, skipping`)
      continue
    }

    await payload.create({
      collection: 'posts',
      data: post,
    })
    payload.logger.info(`Created post: ${post.slug}`)
  }
}

try {
  await seed()
  console.log('Seed complete')
  process.exit(0)
} catch (error) {
  console.error('Seed failed', error)
  process.exit(1)
}
```

A few details worth reading twice:

- `import config from '@payload-config'` uses the path alias from `tsconfig.json`.
- `getPayload({ config })` starts Payload **without** Next.js and gives you the Local API. It is the same call your pages will use in guide 2.
- We never log in. The Local API skips access control by default, which is right for a trusted script.
- `process.exit(...)` closes the database pool. Without it, the script can hang after finishing because open connections keep Node alive.
- `Post['content']` reuses the generated type, so if Lexical's shape changes after an upgrade, `npm run typecheck` tells you.

**Check it works:**

```bash
npm run typecheck
```

```text
> tsc --noEmit
```

**What just happened:** You wrote a small, typed program that talks to your CMS through the same API your website will use. The `richText` helper shows that rich text is just data: a tree of nodes with a `root`, `paragraph` children and `text` leaves. That is why it is stored in a `jsonb` column.

### [Beginner] Step 35 — Run the seed and look at the result

**What we're doing:** Running the seed script, twice, and checking the data in the admin and in Postgres.

**Why:** Running it twice proves it is idempotent. Checking both the admin and the database proves the data is really there and correctly shaped.

**Do it:**

```bash
npm run seed
```

**Check it works:**

```text
[09:21:10] INFO: Created page: home
[09:21:10] INFO: Created post: welcome-to-our-new-website
[09:21:11] INFO: Created post: five-tips-for-choosing-the-right-service
Seed complete
```

Run it again:

```bash
npm run seed
```

```text
[09:21:30] INFO: Page "home" already exists, skipping
[09:21:30] INFO: Post "welcome-to-our-new-website" already exists, skipping
[09:21:30] INFO: Post "five-tips-for-choosing-the-right-service" already exists, skipping
Seed complete
```

Check the database:

```bash
docker compose exec db psql -U postgres -d my_site -c 'SELECT slug, status, published_at FROM posts ORDER BY published_at DESC;'
```

```text
                   slug                   |  status   |        published_at
------------------------------------------+-----------+----------------------------
 five-tips-for-choosing-the-right-service | published | 2026-10-03 09:00:00+00
 welcome-to-our-new-website               | published | 2026-10-01 09:00:00+00
(2 rows)
```

Open http://localhost:3000/admin/collections/posts: both posts are there, and you can edit their content in the Lexical editor like any other post.

**What just happened:** `payload run src/seed.ts` loaded `.env`, compiled the TypeScript file, and ran it. Every `payload.create` went through the full pipeline from Step 28: the slug hook, validation and the `beforeChange` hook all ran, just as if an editor had clicked Save. That is a big advantage of seeding through the Local API instead of raw SQL: your business rules are never bypassed.

**If it breaks:**

- `Cannot find module '@payload-config'`: check the `paths` entry in `tsconfig.json` (Step 12).
- `relation "pages" does not exist`: you skipped `npm run migrate` after resetting the database.
- The script prints nothing and never exits: make sure the `process.exit` lines are present.

### [Beginner] Step 36 — The daily workflow, and the final commit

**What we're doing:** Writing down the routine you will follow every day, and committing the migration and seed.

**Why:** Good habits keep the three environments (your laptop, CI, production) in sync without thinking about it.

**Do it:** Learn this routine.

```bash
# Start of the day
docker compose up -d        # start Postgres
npm run dev                 # start the app

# After changing a collection
npm run generate:types      # refresh TypeScript types
npm run migrate:create -- describe-the-change   # capture the schema change
npm run typecheck           # make sure nothing broke

# Fresh database (new laptop, or after docker compose down -v)
npm run migrate
npm run seed

# End of the day
git add . && git commit -m "feat: ..."
docker compose stop         # optional, frees memory
```

Commit what you built in this part:

```bash
git add .
git commit -m "feat: add initial migration and seed script"
git push
```

**Check it works:**

```bash
git log --oneline
```

```text
c3d4e5f (HEAD -> main, origin/main) feat: add initial migration and seed script
b2c3d4e feat: add users, media, pages, posts and contact submissions collections
a1b2c3d chore: scaffold my-site with Payload, Next.js and Postgres
```

The final tree for this guide:

```text
my-site/
├── .env                     # not committed
├── .env.example
├── docker-compose.yml
├── next.config.mjs
├── package.json
├── tsconfig.json
└── src/
    ├── access/
    │   ├── anyone.ts
    │   └── authenticated.ts
    ├── app/
    │   ├── (frontend)/      # guide 2 builds the website here
    │   └── (payload)/       # admin and API, generated
    ├── collections/
    │   ├── ContactSubmissions.ts
    │   ├── Media.ts
    │   ├── Pages.ts
    │   ├── Posts.ts
    │   └── Users.ts
    ├── fields/
    │   └── slug.ts
    ├── migrations/
    │   ├── 20261006_091502_initial.json
    │   ├── 20261006_091502_initial.ts
    │   └── index.ts
    ├── utilities/
    │   └── slugify.ts
    ├── payload-types.ts
    ├── payload.config.ts
    └── seed.ts
```

**What just happened:** A new teammate can now clone the repository and, with four commands (`npm install`, `docker compose up -d`, `npm run migrate`, `npm run seed`), get a database identical to yours, with content in it. That property, "any environment can be rebuilt from the repo", is what makes automated deploys in guide 3 possible.

## 6. Interview questions

#### Q: How does Payload 3 fit into a Next.js app, and what is the Local API?

Payload 3 is installed into the Next.js App Router project itself. The admin panel is rendered by a route under `src/app/(payload)/admin`, and the REST and GraphQL APIs are Next.js route handlers under `src/app/(payload)/api`. There is no separate CMS server to run or deploy. The **Local API** is Payload's JavaScript interface, `const payload = await getPayload({ config })` followed by `payload.find(...)` or `payload.create(...)`. Server components, server actions and scripts call it directly, as a function call in the same process, so there is no HTTP round trip and no API token to manage. The trade-off is that the website and the CMS scale and deploy together.

#### Q: What is the difference between Payload's push mode and migrations, and why not use push in production?

Push mode lets the Postgres adapter (via Drizzle) compare the config with the live database and alter the database to match, automatically, in development. It is fast for experiments but it is not reviewed, not recorded as discrete steps, and it can drop columns and data when a field is renamed or removed. Migrations are generated files with explicit `up` and `down` SQL, committed to Git and run in order with `payload migrate`, with each run recorded in `payload_migrations`. Production needs migrations because changes must be repeatable across environments, reviewable before they run, and safe for existing data.

#### Q: How does access control work in Payload, and what happens when an access function returns a query?

Each collection defines functions for `create`, `read`, `update` and `delete` (and auth collections have a few more). They receive the request, including `req.user`. Returning `true` allows, `false` denies with a 403, and returning a `Where` query allows the operation only on matching documents. Payload merges that query into the database query, so for posts we return `{ status: { equals: 'published' } }` for anonymous users and drafts never leave the database. One important detail: the Local API runs with `overrideAccess: true` by default, so server code must filter itself or pass `overrideAccess: false` (and a `user`) when it acts on behalf of a visitor.

#### Q: Why give the slug a unique index and format it in a hook, instead of validating it in the form?

The form only protects one entry point. The REST API, the Local API and scripts can all write documents, so the rule must live on the server. A `beforeValidate` field hook normalises any input into a URL-safe slug (or builds one from the title) no matter where the data came from. The unique index in Postgres is the final guarantee: even two simultaneous saves cannot produce duplicate slugs, because the database rejects the second one. The index also makes `WHERE slug = ...` lookups fast, which is exactly what `/blog/[slug]` does on every request.

#### Q: What are route groups, and why does a Payload project have (frontend) and (payload)?

A route group is an App Router folder whose name is in parentheses. It organises routes without adding a URL segment, and each group can have its own root layout. Payload uses this to run two separate applications in one Next.js server: the public site in `(frontend)` with its own HTML, CSS and fonts, and the admin plus API in `(payload)` with Payload's layout and styles. This prevents style and layout leaks between them, while both share the same config, database connection and deployment.

#### Q: Where should uploaded media be stored in production, and why not on the server's disk?

On a cloud object store such as S3 or Vercel Blob, using a Payload storage adapter like `@payloadcms/storage-s3` or `@payloadcms/storage-vercel-blob`. Serverless functions and most container platforms have temporary file systems: files written there disappear on the next deploy or are not shared between instances, so images would randomly vanish. Object storage is durable, shared by all instances, and can be put behind a CDN. In Payload, switching is a config change: the upload collection keeps its fields and image sizes, and only the storage plugin changes.

## Cheatsheet

**Commands**

| Task | Command |
| --- | --- |
| Create the project | `npx create-payload-app@latest` (blank, PostgreSQL) |
| Start or stop Postgres | `docker compose up -d` / `docker compose stop` |
| Delete the database and its data | `docker compose down -v` |
| Open a SQL prompt | `docker compose exec db psql -U postgres -d my_site` |
| Dev server | `npm run dev` then http://localhost:3000 and /admin |
| Type check | `npm run typecheck` |
| Regenerate types | `npm run generate:types` |
| Regenerate the admin import map | `npm run generate:importmap` |
| New migration | `npm run migrate:create -- <name>` |
| Run pending migrations | `npm run migrate` |
| Migration status | `npm run migrate:status` |
| Seed content | `npm run seed` |
| Secret for PAYLOAD_SECRET | `openssl rand -hex 32` |

**Environment variables**

| Name | Local value | Notes |
| --- | --- | --- |
| `DATABASE_URI` | `postgres://postgres:postgres@localhost:5432/my_site` | must match `docker-compose.yml` |
| `PAYLOAD_SECRET` | 64 random hex characters | never commit it |
| `NEXT_PUBLIC_SERVER_URL` | `http://localhost:3000` | sent to the browser, so never a secret |

**Collections in this project**

| Slug | Key fields | Read access |
| --- | --- | --- |
| `users` | email, password, name (`auth: true`) | logged in |
| `media` | alt, upload with thumbnail, card, hero sizes | anyone |
| `pages` | title, slug (unique, hook), layout (richText) | anyone |
| `posts` | title, slug, excerpt, coverImage, publishedAt, status, content | published, or logged in |
| `contact-submissions` | name, email, message | logged in (anyone may create) |

**Collection skeleton**

```ts
// src/collections/Example.ts
import type { CollectionConfig } from 'payload'

import { anyone } from '../access/anyone'
import { authenticated } from '../access/authenticated'

export const Example: CollectionConfig = {
  slug: 'examples',
  admin: { useAsTitle: 'title' },
  access: { read: anyone, create: authenticated, update: authenticated, delete: authenticated },
  fields: [{ name: 'title', type: 'text', required: true }],
}
```

**Rules of thumb**

- Folders are URLs in `src/app`; `(group)` folders are not.
- Never edit `payload-types.ts`, `importMap.js` or files in `src/app/(payload)` by hand.
- Push mode for your laptop, migrations for everything else. Read every generated migration before committing it.
- Access control protects the REST API automatically. The Local API skips it unless you pass `overrideAccess: false`.
- Validate on the server. Client checks are for convenience only.
- Rich text is a JSON tree stored in `jsonb`.
- Any environment should be rebuildable with `npm install`, `docker compose up -d`, `npm run migrate`, `npm run seed`.

> **What's next:** In **Full-stack 2: Build the Website** you will turn this content model into real pages. You will render the `home` page at `/`, other pages at `/[slug]`, the blog at `/blog` and `/blog/[slug]` with server components and the Local API, render Lexical rich text with Payload's React component, build the `/contact` form with a server action and Zod that creates `contact-submissions` documents, and add `afterChange` hooks that call `revalidatePath` so the site updates the moment an editor clicks Save.


---
name: miget-api
description: Deploy and manage apps, databases, buckets, and services on Miget PaaS. Covers authentication, resource provisioning, deployments, addons, domains, environment variables, and all API endpoints.
---

# Miget API - Guide for AI Agents

**Skill version:** `0.6.0` — see the [changelog](https://github.com/migetapp/agent-skills/blob/main/CHANGELOG.md).

## Overview

Miget is a Kubernetes-based Platform-as-a-Service (PaaS) similar to Heroku or Render. It allows developers to deploy and manage applications, databases, and services in the cloud with minimal infrastructure management.

**Base URL:** `https://app.miget.com/api/v1`

**API Documentation:** `https://app.miget.com/api/v1/docs` (Swagger/OpenAPI)

---

## Platform Constraints

These are the platform's fixed rules. Most failed first deployments trace back to one of them, and none is discoverable from a repository — read them before you plan anything.

| Constraint | What it means for you |
|---|---|
| **HTTP is always port 5000** | The app's public URL is served from port `5000`. It is created automatically, cannot be deleted, and no API parameter changes it. An app listening on 3000/8000/8080 will build and start, then never answer. Most frameworks read a `PORT` variable, so setting `PORT=5000` on the app is usually the whole fix; otherwise change the start command. |
| **No implicit release phase** | Nothing runs between the build finishing and the new replicas starting. Database migrations belong in `pre_deploy_command` (`public_git` and `github` only), or they run on every replica boot. |
| **Extra ports are private, and TCP/UDP only** | Additional ports default to private — pass `public: true` at creation or `PATCH .../expose_publicly` afterwards. They carry TCP/UDP, not HTTP. Port management is unavailable on the free plan (403). |
| **App-to-app traffic is off by default** | `allow_connections` is `false` until set on the **destination** app (`PUT /api/v1/apps/{uuid}/security`). Until then it is not reachable at its `internal_url` from other apps on the same resource. |
| **Runtime logs and metrics are not on the REST API** | The REST API serves build/deploy logs and cron run logs only. Application logs and metrics come from the Loki/Prometheus APIs at `metrics.miget.com`, using the *same* `miget_live_` token and an `X-Workspace-Id` header — there is no separate Grafana credential. |
| **Basic Auth credentials are never returned** | The app response tells you whether Basic Auth is on, never what the credentials are. Do not try to read them back. |
| **`quota` is in bytes** | `quota.ram_size` is bytes (`134217728` = 128 MiB), `quota.cpu_size` is a fractional core count. There are no top-level `ram_size`/`cpu_size` fields on the response. |
| **CPU is never the capacity constraint** | Placement and capacity are checked against **RAM and disk only** — CPU is not a quota and is never the reason an app or addon cannot be created. On dev plans (the free plan included) `cpu_size` is a *ceiling*, not a reservation: the **Miget Fair Scheduler** distributes the resource's CPU dynamically across every app and addon on it, so an idle process holds nothing back from a busy one. Guaranteed, dedicated CPU is a Pro-plan feature. Never tell a user that a database "will eat the CPU" of a small resource, and never refuse to co-locate an app and its database on that ground — decide on RAM. |
| **A database addon injects two variables, not one** | A `postgres` or `mysql` addon sets **both** `<ADDON_NAME>_URL` (e.g. `POSTGRES_YZQUW_URL`) and `DATABASE_URL` to the same connection string; `valkey` sets `<ADDON_NAME>_URL` and `REDIS_URL`. The generic alias is skipped only if the app already has a variable of that name, which is never overwritten. Read `GET /api/v1/apps/{uuid}/vars` instead of guessing, and do not add a duplicate `DATABASE_URL` "because the framework needs one". |
| **`name` is server-assigned and always suffixed** | The server appends a random suffix to whatever `name` you send — `wall` comes back as `wall-pfhqv`, and a resource is `migetmxq`. Apps, addons, services, stacks, buckets and secret files all work this way. **The suffixed `name` is the identifier everywhere**: public URLs, Git remote paths, connection variable keys. `service_name` (the unsuffixed form) appears in exactly one place, `internal_url`, and `label` is display text with no addressing role. Never reconstruct any of these from the name you sent, and never strip a suffix that looks like noise — read the field back and use it verbatim. |
| **`state` mixes two vocabularies** | On apps, addons and services `state` reports the platform lifecycle while the object is provisioning and the raw **Kubernetes** status once it is up. There is no `active`, and a healthy database reports **`healthy`** (storage reports **`bound`**) — only an *app* reports `running`. Never poll for one hard-coded string; see "Reading `state`". |
| **Addon vs standalone service** | An addon's lifecycle is tied to its app and is the right default — deleting the app deletes it. A standalone service outlives any single app. Note that explicit mounting works only for `shared_storage`; a shared database is shared by pointing several apps at its connection variables, not by mounting it. |
| **A compose file means a Stack** | A repository with `docker-compose.yml` is a Stack, not an App, and uses an entirely different set of endpoints. Do not try to force it into a single app. |

---

## How to Work

Miget is meant to be driven, not to conduct an interview. Nearly everything needed to deploy a project is discoverable — from the repository in front of you and from the user's own account. Derive what you can, say what you inferred and from which file, and ask only about what is genuinely unknowable or irreversible.

A deploy request always has the same shape:

1. **Read the project** — language, framework, databases, ports, migrations. See "Reading the Project".
2. **Read the account** — `GET /api/v1/resources`, `/api/v1/projects`, `/api/v1/plans`, `/api/v1/regions`, so you build on what already exists instead of duplicating it.
3. **Present one plan** — everything that will be created, what each choice was inferred from, what is a guess, and the monthly cost. See "The Plan Card".
4. **Take one confirmation** — a single yes, on the whole picture.
5. **Execute** — create, configure, deploy.
6. **Verify** — confirm the app actually answers. See "Verifying a Deployment".

This replaces asking for region, plan, deployment method, addon type, RAM and CPU one field at a time. Six questions before anything happens fails both audiences: a first-time user cannot answer them, and an experienced user resents being asked what the repo already says.

**Ask a real question only when:**
- the choice cannot be derived from the repo or the account — for example which of three existing projects to use, when none matches the repository,
- the choice is expensive or hard to undo — a plan materially pricier than the obvious default, or anything destructive,
- the request is ambiguous about *intent*, not merely about mechanism.

**An explicit instruction always wins.** If the user says "deploy this to us-east-1 on the pro plan with a postgres addon", use those values verbatim — do not re-derive them, and do not present a plan card nobody asked for. Honour "just do it, don't explain" and "walk me through every step" equally.

**Never assume consent for:** anything that costs more than the plan the user confirmed, anything destructive (deleting apps, addons, buckets, or data), and anything involving the user's credentials.

### Session Setup

Do these once, before your first API call.

**1. Find the API token.** All API calls require authentication. The token is a secret — your job is to get it into the environment, not to see it. Never ask the user to paste a token or a password into the conversation, and never put a token value on a command line you run: both end up in your transcript, and the second also lands in the user's shell history.

   **Step 1: Check environment variables.**
   Check whether `MIGET_API_TOKEN` is set without printing it:

   ```bash
   [ -n "$MIGET_API_TOKEN" ] && echo "token set" || echo "token missing"
   ```

   If it is set, use it for every API call by *referencing the variable*, never by substituting its value:

   ```bash
   curl -H "Authorization: Bearer $MIGET_API_TOKEN" https://app.miget.com/api/v1/resources
   ```

   **Step 2: If no token is set, have the user install one.**
   Ask which case applies and give them the matching instructions to run themselves:

   - **"I have an API token"** - go straight to Step 3.
   - **"I have an account but no token"** - guide them to generate one:
     1. Go to **https://app.miget.com/my_account#api_tokens**
     2. Click **"Create new token"**
     3. Give it a name (e.g., `cli-agent`)
     4. Copy the token (it starts with `miget_live_`), then go to Step 3.
   - **"I don't have an account"** - direct them to sign up first:
     1. Go to **https://app.miget.com/users/sign_up**
     2. Create an account and verify email
     3. Once signed in, generate an API token at **https://app.miget.com/my_account#api_tokens**, then go to Step 3.

   **Step 3: The user stores the token themselves.**
   Give them the snippet for their shell and ask them to run it in their own terminal. It prompts for the token without echoing it, appends it to their shell config, and exports it into the current session — so the value never passes through you.

   For **zsh** (default on macOS):
   ```zsh
   read -rs "tok?Miget API token: " && printf 'export MIGET_API_TOKEN=%q\n' "$tok" >> ~/.zshrc && export MIGET_API_TOKEN="$tok" && unset tok
   ```

   For **bash**:
   ```bash
   read -rsp 'Miget API token: ' tok && printf 'export MIGET_API_TOKEN=%q\n' "$tok" >> ~/.bashrc && export MIGET_API_TOKEN="$tok" && unset tok
   ```

   For **fish**:
   ```fish
   read --silent --prompt-str='Miget API token: ' tok; and set -Ux MIGET_API_TOKEN $tok; and set -e tok
   ```

   Then ask them to restart the session (or tell you once the command has run) and repeat Step 1. Once stored, the token is detected automatically on every future session.

   **Step 4: Do not proceed without a token.** Attempting API calls without authentication wastes time and confuses the user with 401 errors.

   If the user pastes a token into the conversation anyway, tell them it is now in the transcript, use it for this session if they want to continue, and recommend they rotate it at **https://app.miget.com/my_account#api_tokens** afterwards.

**2. Confirm this skill is current.** This API changes often, and a stale copy will describe fields that no longer match it. Once per session, alongside finding the token, fetch the latest published release and read its `tag_name`:

   ```bash
   curl -s https://api.github.com/repos/migetapp/agent-skills/releases/latest
   ```

   Compare it with the **Skill version** at the top of this file. **Only if the published version is newer than yours**, tell the user once and walk them through the update. First refresh everything the CLI manages:

   ```bash
   npx skills update
   ```

   Then **verify the copy your own agent reads**. A general update only refreshes agents the skill was installed for, so it can silently leave the current agent on an old version. Re-read the `Skill version` line in this agent's own skill directory — for example `~/.claude/skills/miget-api/SKILL.md` (global) or `./.claude/skills/miget-api/SKILL.md` (project). If it still shows the old version, install it for this agent explicitly:

   ```bash
   npx skills add migetapp/agent-skills -a claude-code
   ```

   Use whichever agent you are in place of `claude-code` (`codex`, `cursor`, `gemini-cli`, …). Tell the user that the copy already loaded in the current session does not change — the new version takes effect in the next session.

   If the version matches, say nothing. Never let this check block or delay the user's actual request: if it fails for any reason (offline, rate-limited, unexpected response), skip it silently and carry on.

**3. Know which workspace you are in.** Miget uses workspace-based multi-tenancy. Include the `X-Workspace-Id` header when the user works with multiple workspaces. If omitted, the API uses the user's default workspace.

### Reading the Project

Before asking anything, read the repository. Most of a deployment plan is already written down in it.

| Signal | What it tells you |
|--------|-------------------|
| `package.json` with `next`, `nest`, `express`, `remix` | Node app, `builder: "auto"`; framework drives sizing |
| `requirements.txt`, `pyproject.toml`, `Pipfile` | Python; read deps for `django`, `flask`, `fastapi` |
| `Gemfile` + `config/database.yml` | Rails; the adapter in `database.yml` names the database it needs |
| `go.mod` / `composer.json` / `Cargo.toml` / `pom.xml`, `build.gradle` | Go / PHP / Rust / JVM |
| `prisma/schema.prisma` | Read `datasource.provider` → `postgres` or `mysql` addon. Prisma migrations need `builder: "dockerfile"` — see Build Settings |
| `drizzle.config.*`, `knexfile.*`, `alembic.ini`, `db/migrate/` | Migration tooling → set `pre_deploy_command` |
| `ioredis`, `redis`, `redis-py`, `sidekiq`, `celery`, `bullmq` in deps | Needs a Valkey addon |
| `.env.example`, `.env.local`, `.env.sample` | The variable **names** the app expects |
| `Dockerfile` | `builder: "dockerfile"`; read its `EXPOSE` and `CMD` |
| **`docker-compose.yml` / `compose.yaml`** | **This is a Stack, not an App** — use the stacks endpoints instead |
| `compose.miget.yaml` | A stack whose platform overrides are already tuned — prefer it over a plain compose file |
| Code listening on `:3000`, `:8000`, `:8080` | Must serve on **5000** — see Platform Constraints |
| A nested app directory, or `workspaces` in `package.json` | Monorepo → set `project_path` |
| `.github/workflows/`, `Procfile`, `render.yaml`, `fly.toml`, `app.json` | Prior deployment intent — read it for build/start commands and env vars |
| `git remote -v` | **Decides the deployment method** — a GitHub remote means `github`, another host means `public_git`, no remote at all means `git_push`. Check this before you plan; see "Choosing Defaults" |

**Report what you found, not that you looked.** "Next.js with Drizzle pointing at PostgreSQL, migrations via `drizzle-kit migrate`" is useful. "I have analysed your repository" is not.

**Handling `.env` files — read names, never expose values.**
- Read `.env*` to learn which variables the app expects and to spot which look like secrets.
- Send values to Miget with `POST /api/v1/apps/{uuid}/vars` (or `app_vars_attributes` at creation).
- **Never print a secret value** in chat, in a summary, in a log line, or in a commit. Refer to variables by name only.
- Never commit a `.env` file, and never copy secrets into a compose file or a Dockerfile.
- If a required variable has no value anywhere (a `.env.example` placeholder like `changeme` or an empty string), that is one of the few things genuinely worth asking about — or generate a strong random value when it is clearly an app-internal secret such as `SESSION_SECRET` or `JWT_SECRET`, and tell the user you did.

### Choosing Defaults

Derive these rather than asking. Read live values from `GET /api/v1/plans` and `GET /api/v1/regions` — never quote a price from memory.

**Region.** The platform has no default region. Prefer, in order: the region of a resource the user already owns; then the region implied by where they are (North America → `us-east-1`, everywhere else → `eu-east-1`, which is the same fallback the platform uses at signup). Say which you picked and offer to change it. This does not apply to static sites, which accept `eu-east-1` only — see "Static Sites".

**Resource.** Reuse an existing resource with enough free RAM before creating a new one — `GET /api/v1/resources`. A new resource is a new monthly charge; reusing one is free. When you do need a new one, pick the **cheapest plan from `GET /api/v1/plans` whose `ram_size` and `disk_size` cover the app plus every addon you are about to attach**, with a little headroom — the app and its databases all draw on the same resource. Size on RAM and disk only: CPU is not part of this arithmetic (see Platform Constraints). Do not reach for a larger plan speculatively; resizing later is easy.

**Always set `ram_size` and `cpu_size` explicitly.** If you omit them, the app is given *the entire remaining RAM and CPU of the resource*, leaving no room for anything else on it. This is the single most common way to quietly wedge an account. Values are in MiB and fractional cores; the floor for placing an app is 128 MiB. **Mind the asymmetry:** sizes you *send* (`ram_size`, `disk_size` on create/update) are MiB/GiB, while sizes the API *returns* (`quota.*`, plan `ram_size`, resource `total_/available_*`) are bytes. Never compare a value you sent against one you read back without converting.

| Workload | `ram_size` | `cpu_size` |
|---|---|---|
| Static site or SPA build | 256 | 0.25 |
| Go / Rust binary | 256 | 0.25 |
| Node API (Express, Fastify, NestJS) | 512 | 0.5 |
| Next.js / Nuxt / Remix (SSR) | 512–1024 | 0.5 |
| Django / Flask / FastAPI | 512 | 0.5 |
| Rails | 512–1024 | 0.5 |
| JVM (Spring Boot) | 1024+ | 1.0 |

Treat these as starting points, not platform rules — raise them if the app is memory-hungry, and check the **RAM** column against the resource's free capacity before sending. The `cpu_size` column is a ceiling the Fair Scheduler works under, not a slice carved out of the resource: the numbers may sum past the plan's core count without anything failing, and on a dev plan CPU does not cap replica counts either.

**Deployment method.** Read it off the repository's own remote — `git remote -v` — rather than defaulting to `git_push`. `git_push` is the fallback for code that lives nowhere else, and it is the only method that ends with you asking the user to fetch a credential from the dashboard. Reaching for it while the repo sits on GitHub trades a working auto-deploy for a manual one.

| What you found | Use | Why |
|---|---|---|
| A GitHub remote, or the user names a GitHub repo | `github` | The only method with **auto-deploy on push** and **review apps** for pull requests. Needs a `credential_id` from `GET /api/v1/git_credentials`; if the workspace has none, having the user install the Miget GitHub App once is a smaller ask than a token they must re-issue whenever it is lost. |
| A remote on another host, or a public repo with no GitHub App | `public_git` | Deploys straight from the URL, no credential exchange. Private repos take a `credential_id`. |
| The image is already built and pushed to a registry | `container_registry` | There is nothing left to build. |
| A Dockerfile the user builds and ships themselves | `kamal` or `container_registry` | Their pipeline already produces the artifact. |
| No remote at all — local-only code | `git_push` | The fallback. See the `git_push` notes for the SSH-first flow. |

Review apps are configured in the dashboard, not over the API, and only for `github` apps — so a repo that will want PR previews should be on `github` from the start rather than migrated later.

**Addons.** Their defaults are sensible; supply a size only when the app clearly needs more.

| Addon | Default RAM | Default disk | Default CPU |
|---|---|---|---|
| `postgres` | 128 MiB | 0.5 GiB | 0.1 |
| `mysql` | 128 MiB | 0.5 GiB | 0.1 |
| `valkey` | 32 MiB | 0.1 GiB | 0.1 |

Prefer an **addon** over a standalone service unless more than one app genuinely needs the same database.

**The free plan.** One free resource per user, personal workspaces only, and it is small: 0.1 core, 256 MiB RAM, 1 GiB disk — so the app **and every addon on it** must together fit inside 256 MiB of RAM and 1 GiB of disk. It also cannot use public custom ports, autoscaling, cron jobs, or Postgres backups, and addon CPU is pinned to 0.1 regardless of what you request. It suits a first deploy or a demo; say plainly when a project has outgrown it rather than trying to squeeze it in.

**An app plus a database does fit on the free plan.** A 128 MiB app (the floor) and a Postgres addon at its 128 MiB default come to exactly 256 MiB, with the addon's 0.5 GiB disk inside the 1 GiB allowance. That is tight and worth saying out loud — no headroom to raise either later without upgrading — but it is a valid configuration and the platform will create it. The 0.1 core is *not* divided between them, so it is never the reason to refuse: if you push back on a free-plan Node + Postgres deploy, push back on the 256 MiB, not on the CPU.

A free resource holding **no apps and no services** is deleted after 30 days of inactivity; one with an app on it is never auto-deleted. Deployed apps currently run continuously on every plan — nothing is idled or put to sleep. Treat that as today's behaviour rather than a promise, and do not build an argument for the free plan on guaranteed uptime.

### The Plan Card

Present exactly one of these before creating anything, then ask once.

> **Deploying `acme-storefront` to Miget**
>
> | | | Why |
> |---|---|---|
> | Resource | new, `Miget Hobby Tier 2` (1 core, 1 GiB) in `eu-east-1` | no existing resource in this workspace; 1 GiB fits the app plus the database |
> | App | `acme-storefront`, builder `auto`, 512 MiB / 0.5 core | `package.json` → Next.js 15 |
> | Deploy from | GitHub `acme/storefront`, branch `main` | current repo remote |
> | Database | PostgreSQL addon | `drizzle.config.ts` → `dialect: "postgresql"` |
> | Migrations | `pre_deploy_command: npx drizzle-kit migrate` | migrations in `drizzle/` |
> | Env vars | 6 imported from `.env.local`; `NEXTAUTH_SECRET` generated | — |
> | **Cost** | **$7.00/month** | the resource plan; the addon draws on its capacity, not a separate charge |
>
> Guessing on: region (no signal in the repo — say the word if you want somewhere else).
>
> Deploy this?

Rules for the card:
- **Every row names its evidence.** A row you cannot justify is a row you should be asking about.
- **Always state the cost**, read live from `GET /api/v1/plans` — never from memory, and never from the example above. Addons and apps consume the resource's capacity rather than being billed separately, so the monthly figure is the resource plan (plus any components). If it is free, say so.
- **Separate inference from guesswork.** Anything with no signal behind it goes in an explicit "guessing on" line.
- **One card, one question.** Do not follow it with more questions unless the user's answer opens a genuinely new choice.

### Two Registers

The same decisions get explained at different densities depending on who is asking. Infer the register — never ask "are you experienced?", which is both insulting and unreliable.

**Read as new to the platform when:** the request is goal-level ("put this online", "push this to miget"), the repo has no Dockerfile, CI config, or infrastructure files, the workspace is empty, or the user asks what a term means.

**Read as fluent when:** the request is mechanism-level ("pro resource in us-east-1, 2 vCPU, postgres addon"), platform vocabulary is used correctly, or the account already holds several resources and projects.

A correction using platform vocabulary shifts the register up for the rest of the session. A question like "what's an addon?" shifts it down.

The same moment, explanatory:

> Your app needs somewhere to run, so I'll create a **resource** — a small slice of compute in Miget's `eu-east-1` region — and put the app on it. Drizzle is pointing at PostgreSQL, so I'll attach a **Postgres addon**: a managed database whose connection string lands in your app's environment automatically, and whose lifecycle follows the app. Your migrations will run once before each release rather than on every restart.
>
> That comes to $7.00/month — the database runs inside the same resource, so it is not billed separately. Shall I go ahead?

and terse:

> `miget_hobby_2` in `eu-east-1`, app `acme-storefront` (builder `auto`, 512 MiB/0.5, GitHub `acme/storefront@main`), Postgres addon, `pre_deploy_command: npx drizzle-kit migrate`, 6 vars from `.env.local`. $7/mo. Go?

**Register changes how much is explained. It never changes how much is done without asking.** Both versions above make the same decisions and both stop at the same single confirmation. Inverting that — acting more freely for users who seem inexperienced — would give the least oversight to the people least able to catch a mistake.

### Verifying a Deployment

A deployment reaching `completed` means the image built and the pods started. It does not mean the app works. **You are done when the app's URL answers with a non-5xx status** — check it before telling the user it is live.

1. Poll `GET /api/v1/apps/{uuid}/deployments/{id}` until the status settles.
2. Request the app's URL.
3. If that fails, pull build logs from `GET /api/v1/apps/{uuid}/deployments/{id}/logs`, and runtime logs from the Loki API (see "Monitoring & Observability" — runtime logs are not on the REST API).
4. Work the table below.
5. Fix and redeploy, or tell the user precisely what is wrong. Never report a deployment as successful without having checked the URL.

**A failed deployment tells you nothing by itself.** `GET /api/v1/apps/{uuid}/deployments/{id}` returns `state: "failed"` and no reason — there is no error field, no failing phase, no exit code. The log body is the only source of truth, and the platform will not summarize it for you.

So when a deployment fails, **fetch the logs and read them yourself** before you say anything beyond "it failed". Tell the user you are doing it and then do it — "The deployment failed. Let me pull the build logs and find out why." — rather than reporting the bare status and waiting, or pasting a log dump for them to read. What the user wants back is the cause and the fix, in a sentence or two, with the relevant few lines quoted. They asked you to deploy the app; diagnosing why it did not deploy is part of that job, not a follow-up request.

Two things about the log body worth knowing before you read it:
- **Logs 404 while the deployment is still running.** They are uploaded once the run ends, and `logs_stored_at` on the deployment turns non-null at that moment. Read the 404 body before giving up: *"Logs are not available yet"* means finish polling step 1 and retry, while *"Logs not found in storage"* means they aged out under the plan's retention window and are not coming back.
- **It is one blob covering every phase.** Build output first, then `-----> Release`. A failure in the release phase is still labelled a build failure (see the table), so read to the end before concluding anything.

| Symptom | Probe | Likely cause | Fix |
|---------|-------|--------------|-----|
| Deploy succeeded, URL times out or 502s | Runtime logs show the server listening on 3000/8080 | App is not on port 5000 | Make the app bind `5000` (usually `PORT`/`process.env.PORT`), redeploy |
| Container starts then exits immediately | Runtime logs show a config or connection error at boot | Missing environment variable | Compare `.env.example` against `GET /api/v1/apps/{uuid}/vars`, add what's missing |
| App 500s on every request touching data | Runtime logs show "relation does not exist" / "no such table" | Database never migrated | Set `pre_deploy_command`, redeploy |
| Pod restarts repeatedly | Metrics show memory at the quota ceiling | Out of memory | Raise `ram_size` on the app, or reduce the app's footprint |
| Build fails early | Build logs show a missing command or failed install | Build image lacks the tool, or the wrong builder | Check the builder, `build_command`, and `project_path` for monorepos |
| Build fails with a missing module the app depends on | Build logs show the build step, not the install step, failing | `NODE_ENV=production` is set before `build_command`, so `npm ci` skipped `devDependencies` | Use `npm ci --include=dev && …` |
| Deployment reads `Build failed: Release job failed` | The build log ends with `Built in …s` and an image push, then a `-----> Release` section | The **build succeeded**; `pre_deploy_command` failed | Read the release output at the **bottom** of the same log body — not the build section |
| App reachable, but cannot reach another app | Target app's `allow_connections` is `false` | Internal traffic is off by default | Enable `allow_connections` on the target app |
| A non-HTTP port refuses connections from outside | `GET /api/v1/apps/{uuid}/ports` shows it as private | Extra ports are private by default | Expose it publicly — see the ports endpoints |


### Reading `state` on an app, addon, or service

**`state` is not one vocabulary.** For apps, addons and services the field is computed, not stored: while the object is still being provisioned it reports the platform's own lifecycle state, and once it is up it reports **whatever Kubernetes says**, downcased. The two sets barely overlap, and nothing in the response tells you which one you are looking at. Polling for a single hard-coded string is the most common way to hang forever.

**There is no `active`, and a healthy database is not `running`.** A working Postgres, MySQL or Valkey addon reports **`healthy`**. A working storage addon reports **`bound`**. Only an *app* reports `running`.

| Object | Provisioning / stopped | Up and healthy | Trouble |
|---|---|---|---|
| App | `pending`, `assigned`, `start_scheduled`, `started`, `deploying`, `cloning`, `stop_scheduled`, `restart_scheduled`, `stopped`, `blocked` | `running` | `failed`, `problem` (pods failing while replicas are still up) |
| `postgres` / `mysql` / `valkey` addon or service | `pending`, `processing`, `creating`, `stopped` | **`healthy`**, or `running` | `failed`, `failing`, `degraded`, `degradated`, `crashloopbackoff` |
| `storage` addon or service | `pending`, `processing`, `stopped` | **`bound`** | `failed`, `lost` |

**Why a healthy addon sometimes says `running` instead of `healthy`.** The Kubernetes status is fetched live over the network and cached for ten seconds. When that call fails or returns nothing, the value falls back to the platform's lifecycle state — which for a working addon is `running`. Both answers mean the same thing. Treat them as equivalent rather than waiting for the "real" one to appear.

**So test the set, never the string.** Wait on membership, and always give up eventually:

```bash
# Correct: any of these means the addon is usable.
case "$state" in healthy|bound|running) echo up ;; esac
```

- **Ready:** `healthy`, `bound`, `running`, `active`
- **Still working:** `pending`, `processing`, `creating`, `assigned`, `started`, `start_scheduled`, `deploying`, `cloning`
- **Stop polling and report:** `failed`, `failing`, `lost`, `crashloopbackoff`, `problem`, `stopped`, `blocked`
- **Anything else:** treat as still working, but bound the wait — an unrecognised value is not a reason to loop indefinitely.

Two more things worth knowing before you build a wait loop on this field:

- **`degraded` / `degradated` is not a failure.** A Postgres HA cluster reports it while a replica catches up; the primary is serving. Surface it, don't block on it.
- **Stacks are different.** `GET /api/v1/stacks/{uuid}` returns a *normalised* state — `pending`, `validating`, `publishing`, `building`, `deploying`, `running`, `degraded`, `failed`, `stopped` — computed across the stack's items. Kubernetes vocabulary does not leak through there, so a stack really does settle on `running`.
- **On a *deployment*, `running` means the opposite.** A deployment's `state` is its own small enum — `pending`, `running`, `completed`, `failed`, `cancelling`, `cancelled` — where `running` means *still building* and `completed` is the terminal success. Don't carry the app's reading of the word across to it.

### General Best Practices

**1. Use the OpenAPI spec as a fallback.** If you encounter an endpoint or parameter not covered in this guide, consult the OpenAPI spec at `https://app.miget.com/docs/openapi.json` for the exact schema. You don't need to load it proactively - this guide covers the common cases.

**2. Handle async operations.** Deployments and resource provisioning are asynchronous. Poll the relevant status endpoint to track progress rather than assuming immediate completion.

**3. Handle errors gracefully.** Parse error responses and provide helpful feedback to the user:
   - `400` - Validation error, check required fields
   - `401` - Authentication failed, check token
   - `403` - Permission denied, check user permissions
   - `404` - Resource not found, verify IDs/UUIDs
   - `422` - Validation failed, check field values

**4. Validate before creating.** Before creating resources, verify that dependencies exist:
   - Verify project exists (if creating app)
   - Verify resource exists (if assigning to app)
   - Verify region exists (if creating resource)
   - Check if names are available (apps, projects must be unique within a workspace)
   - **Validate user-supplied links.** When the user gives a source URL — a `public_git` repository, a `container_registry` image reference, or a stack `repository_url` — check its format before sending the request. The platform rejects malformed or unreachable sources at creation, so catching it first lets you correct the user instead of surfacing a 422. See the per-method format rules under "Deployment Configuration by Method" (public Git URLs and container image references).

**5. Provide helpful follow-up.** After creating resources, confirm what was created, provide the UUID/ID, and suggest logical next steps.

**6. Run database migrations in `pre_deploy_command`.** Miget has no implicit release phase. For `public_git` and `github` apps, set `deployment_config.pre_deploy_command` so migrations run once before the new release starts — putting them in the start command runs them on every replica boot. See "Build Settings for `public_git` and `github`".

**7. Never handle secrets in the clear.** Reference `$MIGET_API_TOKEN` instead of its value, and don't echo tokens into logs, error messages, commands, or anything shown to the user. The same applies to every other secret the platform hands you — Git tokens, registry credentials, addon connection strings and environment-variable values. If a response body contains one, summarise it rather than printing it.

---

## Authentication

Miget API supports two authentication methods. **If you are an agent, use Method 1.** Method 2 requires the user's account password and is documented only for interactive clients.

### Method 1: API Token (use this)

1. **Generate API token** in the web UI:
   - Go to `https://app.miget.com/my_account#api_tokens`
   - Create a new API token
   - Copy the token (starts with `miget_live_` prefix)

2. **Store it in `MIGET_API_TOKEN`** — see "Session Setup" for shell snippets that do this without exposing the value.

3. **Use API token** in requests, referencing the variable rather than the value:
   ```http
   Authorization: Bearer $MIGET_API_TOKEN
   ```

   - An API token expires only if it was given an expiry date; otherwise it runs until revoked
   - Scoped and individually revocable, so a leak is contained and traceable
   - Better for long-running automation and CI/CD

   There are **two kinds**, and which one you hold changes what the API answers:

   - A **user token** is created at `https://app.miget.com/my_account#api_tokens` and acts as its owner, carrying their role in whichever workspace the request targets.
   - A **workspace token** is created in workspace settings under Developers, and acts for that one workspace. It carries its own permission list and its own project list — neither the creator's role nor workspace ownership widens it. Three refusals follow from that, and none of them mean the token is broken: an `X-Workspace-Id` naming a different workspace answers **403**, `/api/v1/users/me` and everything under it answers **403**, and any workspace administration endpoint is simply not grantable to it. If you hit these, you are holding a workspace token and the request needs a different credential, not a retry.

### Method 2: Username/Password (JWT Tokens) — not for agents

This exchanges the user's **account password** for a short-lived JWT. It grants everything the account can do, and the password itself cannot be revoked without a reset. Do not collect, transmit, or store a user's password on their behalf — direct them to Method 1 instead. Documented here for completeness, and for interactive clients where the user types their own password.

1. **Sign in** to get access and refresh tokens:
   ```http
   POST /api/v1/auth/sign_in
   Content-Type: application/json

   {
     "email": "user@example.com",
     "password": "your-password"
   }
   ```

   **Response:**
   ```json
   {
     "access_token": "eyJhbGc...",
     "refresh_token": "eyJhbGc..."
   }
   ```

2. **Use access token** in subsequent requests:
   ```http
   Authorization: Bearer {access_token}
   ```

   - Access tokens expire after **30 minutes**
   - Use refresh token to get a new access token when expired

3. **Refresh token** (if access token expired):
   ```http
   POST /api/v1/auth/refresh_token
   Content-Type: application/json

   {
     "refresh_token": "eyJhbGc..."
   }
   ```

---

## Core Concepts

### Workspaces

Miget uses **workspace-based multi-tenancy**. Each user can belong to multiple workspaces (organizations/teams).

- **Default workspace:** If no `X-Workspace-Id` header is provided, API uses the user's default workspace
- **Multi-workspace:** Include `X-Workspace-Id` header to specify which workspace to operate on:
  ```http
  X-Workspace-Id: {workspace-uuid}
  ```

### Resources (Migets)

A **Resource** (internally called "Miget") is a compute resource that provides CPU, RAM, and disk space. Resources are assigned to applications and services.

- Resources have **plans** (free, dev, pro tiers)
- Resources can have **components** (extra RAM, CPU, disk)
- Resources can have **labels** (user-defined strings like "production", "staging", "sandbox" for identification)
- Resources are region-specific
- Resource capacity is reported as `total_ram_size` / `total_used_ram_size` / `available_ram_size` (and the `disk_size` equivalents) — all in **bytes**, the same unit as `quota.ram_size` on apps and `ram_size` on plans. `*_cpu_size` fields are fractional core counts. When you check whether a resource can host another app, compare bytes to bytes; treating these as MiB is the usual cause of "it should have fit".
- A resource is a fixed-capacity compute unit that hosts as many workloads as fit. It is **not** one-per-app, and by default it is not owned by any project
- A resource can be **assigned** to one or more projects (`POST /api/v1/projects/{project_id}/resources`). Once assigned, only those projects may place workloads on it; anything else is refused with **422**. Assigning restricts the **resource**, never the project — a project can always use any unassigned resource, whether or not it has assignments of its own. `GET /api/v1/projects/{project_id}` lists a project's assigned resources under `resources`
- Every resource reports its own assignments as `assigned` (a boolean) plus `project_ids` — an array of project UUIDs. `assigned: false` means nobody has assigned it and any project may deploy on it. `project_ids` lists the projects it is assigned to that **you can access**; a resource assigned solely to projects you cannot access is not returned to you at all, so you will not see it in `GET /api/v1/resources` and cannot address it by UUID
- **Resource selection can be refused.** Before offering a `resource_id`, read `assigned` and `project_ids`: the resource is usable when `assigned` is false, or when `project_ids` contains the target project's UUID. Prefer a resource assigned to the target project, then an unassigned one. A 422 mentioning an assignment means the resource is closed to that project — pick another one, or assign it to the project first
- **Assigning is itself refused while the resource still runs workloads the assignment would lock out.** The 422 names the blocking projects with their workload counts — except that it **redacts every project you cannot reach**, reporting only how many there are. Send `with_hosting_projects: true` on `POST /api/v1/projects/{project_id}/resources` to assign the resource to those projects too and let the call through; it only ever widens access, and every workload already there stays where it is. When a blocker is redacted this flag is the *only* way forward, because you cannot name a project you cannot see — do not retry the bare call and do not report the assignment as impossible

### Applications (Apps)

An **Application** is a deployable service (web app, API, worker, etc.).

- Apps are deployed to a **Resource** (Miget)
- Apps can have multiple **deployment methods**:
  - `git_push` - Push to Miget-hosted Git remote
  - `public_git` - Public Git repository
  - `github` - GitHub repository (via GitHub App)
  - `container_registry` - Pre-built container image from a registry (Docker Hub, GHCR, etc.)
  - `parent_image` - Inherit image from parent app
  - `kamal` - Deploy from local machine using Kamal
- Apps belong to a **Project** (for organization)
- Apps can have **addons** (databases, caches, storage)
- Apps can have **cronjobs** (scheduled tasks)
- Apps can have **domains** (custom domains)
- Apps can have **environment variables** (vars)
- Apps can have **ports** (exposed ports). Port `5000` is fixed: HTTP traffic on the app's `*.migetapp.com` URL is always served from port `5000` — the app must listen on `5000`, and this port cannot be removed or changed. Additional TCP/UDP ports can be added for custom protocols; they are **private by default** and can be exposed publicly via the expose endpoint (see workflow 9, and https://docs.miget.com/networking/ports for the full list of supported ports).
- Apps can be **public or private** (`private_access`): a private app has no public ingress and is reachable only inside the workspace network. Settable on create/update (default `false`); returned in the app response.
- Apps have an **internal URL** for app-to-app and addon connections, returned as `internal_url` on the app response in the form `<service_name>.<resource-name>.<region-code>.migetapp.internal:5000`. Traffic from other apps requires `allow_connections: true` on the **destination** app (default `false`, set via `PUT /apps/{uuid}/security`); once enabled, other applications on the same resource (miget) can reach it at its `internal_url`.
- Apps also have a **public URL**, returned as `public_url` on the app response in the form `https://<name>.<region-code>.migetapp.com`. The region comes from the compute resource the app runs on, and since `resource_id` is required at creation, every app has one. It is built from `name`, not `label` or `service_name` — and because the server appends a random suffix to `name` at creation, the only reliable way to learn an app's URL is to read `public_url` back from the response. An app with `private_access: true` has no public ingress, so its `public_url` will not answer. Custom domains are **not** included here; they are listed separately under `GET /apps/{uuid}/domains`, which returns `[]` for an app that only has its platform URL.
- Resource limits are reported under `quota` on the app response: `quota.ram_size` is in **bytes** (e.g. `134217728` = 128 MiB) and `quota.cpu_size` is a fractional core count. There are no top-level `ram_size`/`cpu_size` fields.
- The app response also returns `basic_auth_enabled` (whether HTTP Basic Auth is enforced at the ingress). Basic Auth credentials are **never** returned by the API.
- Every app automatically gets **monitoring** — Grafana dashboards, metrics, and logs, with Prometheus/Loki-compatible query APIs at `metrics.miget.com`. Runtime metrics and app logs are **not** on the REST API; see the Monitoring & Observability section.

### Projects

**Projects** are logical groupings of apps and services.

- A project holds applications, static sites, services, stacks and buckets. Applications, static sites, services and stacks always belong to exactly one project; buckets may belong to one
- Ownership (which project a workload belongs to) and placement (which resource it runs on) are separate. Static sites sit on no resource at all
- **Ownership can be changed on every kind; placement cannot.** Send `project_id` to `PUT /apps/{uuid}`, `PUT /services/{id}`, `PUT /stacks/{uuid}`, `PUT /static_sites/{uuid}` or `PUT /buckets/{uuid}` to move a workload between projects — it keeps running on the same resource throughout. Moving a stack moves every application and service in it. There is no way to move a workload to a different resource; that requires deleting and recreating it
- Because the resource cannot change, a move is refused with **422** when the resource is assigned to projects that do not include the destination. The way through is to assign the resource to the destination project as well, not to pick a different resource
- A `project_id` that does not exist — or that exists but you cannot reach — is a **404** carrying `{"error": "Project not found"}`, on every one of those endpoints. Those two cases answer identically on purpose: a status code that told them apart would confirm that a restricted project is there. Read this 404 as "the destination could not be resolved", never as "the workload is gone" — the workload is untouched and still in its original project. Re-read `GET /api/v1/projects` to see which destinations you may actually use
- Projects can have **project-level environment variables** (shared across apps)
- Projects can have **project secret files** (shared files)
- Projects can have **assigned resources**, returned as `resources` on the project response. An empty list means the project simply uses the shared pool — which it may do even when it has assignments. The same relationship reads from the other side as `assigned` and `project_ids` on the resource response, which is the cheaper check when you are choosing a resource
- A project can also be **restricted** to specific people, returned as `restrictions` on the project response. An empty list means the project is open to the whole workspace; any entry closes it to everyone but the listed members, everyone holding a listed role, the workspace owner and holders of the **built-in `admin` role**. A custom role never bypasses a restriction, whatever permissions it carries — including `workspace:members`. Restricting projects is available on organization and enterprise workspaces only, and that plan refusal is checked **before** the subject you passed, so on a smaller plan you get the plan message even when the email is also wrong
- **A project you cannot reach simply does not appear.** It is absent from `GET /api/v1/projects`, and so are its applications, services, stacks and buckets in their own listings; addressing any of them by UUID returns **404**, not 403. So a short project list is not proof that the workspace holds nothing else — it is what you are allowed to see. The same is true of resources assigned solely to projects you cannot reach; an unassigned resource stays visible to everyone, because any project may deploy on it.

### Buckets

A **Bucket** is an S3-compatible object storage container.

- Buckets are attached to a **Resource** (Miget) for quota management
- Buckets may belong to a **Project**, but the field is optional — a bucket with no project stays at workspace level. It is never inferred: omit `project_id` and the bucket is unassigned, even when the workspace has exactly one project. A bucket without a project satisfies no assignment, so it cannot be created on — or left sitting on — an assigned resource
- Buckets can be **public** or **private** visibility
- Buckets have **S3 credentials** (access key / secret key) returned in the `GET /api/v1/buckets/{uuid}` response - use these for direct S3 API access via any S3-compatible client
- Buckets support **policies** (S3-compatible JSON bucket policies)
- Buckets support **ACLs** (S3-compatible XML access control lists)
- Bucket objects can be managed via presigned upload/download URLs

### Services

**Services** are standalone managed services (databases, caches, storage) that can be shared across multiple apps.

- Service types: `postgres`, `shared_storage`
- Services can be mounted to apps as addons
- Services have their own lifecycle independent of any app

### Stacks (Docker Compose)

A **Stack** deploys a multi-service application from a single `docker-compose.yml` in a Git repository.

- A stack is pinned to a **Resource** (Miget) and belongs to a **Project**
- Miget detects the compose services and provisions the underlying **apps** and **managed services** for you
- The stack tracks a Git **branch**; each deploy re-reads the compose file and reconciles changes
- Stack `state` and per-service status are computed from the underlying apps/services

---

### Static Sites

A **static site** hosts prebuilt HTML, CSS and JavaScript from object storage. It is
**not an application** and is not reachable through `/api/v1/apps` — it has its own
endpoints under `/api/v1/static`. There is no compute resource, no replicas, no ports
and no environment variables, and it costs nothing to run.

Three things differ from applications and will bite you if you assume otherwise:

- **The name is exact and globally unique.** No random suffix is appended, so the name
  you send is the name the site is served under: `https://{name}.static.onmiget.com`.
  A taken name is a `422`, not a silent rename. Pick something specific.
- **The content source is fixed at creation.** It decides what gets provisioned, so it
  cannot be changed later. To switch sources, create another site.
- **`region` accepts `eu-east-1` only.** It is where the content is stored; serving is
  region-less, so a site is equally fast everywhere regardless. Omit it and you get
  `eu-east-1`. Sending anything else — including `us-east-1`, which is a valid region
  for every other resource — is rejected with `400`, so do not carry the region you
  picked for the user's apps over to their static site.

There are four sources, in two families:

| Source | Content you supply | Built? | Deploys when |
|---|---|---|---|
| `github` | the repository | yes, generator auto-detected | you call the deploy endpoint, or on push with auto-deploy |
| `git_push` | the repository | yes, generator auto-detected | you push to the site's git remote |
| `zip` | the finished site | no | you upload an archive |
| `sftp` | the finished site | no | the SFTP session closes |

The two git sources are built for you: the generator is detected automatically across
33+ frameworks (Next.js, Hugo, Astro, Jekyll, Eleventy, SvelteKit and so on), and you
can override `build_command`, `output_dir` or `project_path` when detection is wrong.

The two upload sources take **already-built output**. Uploading a Hugo *project* as a
zip publishes its source files, not a rendered site — build first, upload the output
directory.

#### Every source starts by creating the site

**Create first, then deploy.** There is no way to push, upload or connect before the
site exists: the git remote, the SFTP host and the deploy endpoint are all derived from
the site, and the daemon has to provision its storage and routing before any of them
answer.

**Do not wait for `state` to leave `pending` — it will not.** A static site is `pending`
until content has been deployed to it, so waiting for anything else before the first
deploy hangs forever. What you actually wait for is the field your source needs, which
the daemon fills in during provisioning (a few seconds, normally):

| Source | Poll `GET /api/v1/static/{uuid}` until |
|---|---|
| `zip`, `sftp` | `deployment_config.bucket_name` is set |
| `git_push` | `deployment_config.git_ssh_url` is set |
| `github` | nothing — deploy as soon as the site is created |

Do not construct the git remote or the SFTP host yourself from the site name. Read them
back from `deployment_config`; they are empty until the daemon fills them in.

After the first successful deploy the site leaves `pending`, and from then on `state`
tells you what you expect it to.

**Finish by giving the user the URL.** The site response carries `url`
(`https://{name}.static.onmiget.com`) — fetch the site once the deploy settles, check
the URL answers, and hand it over. Custom domains are not included there; they are
listed under `GET /api/v1/static/{uuid}/domains`.

`url` is built from the name, so it is filled in from the moment the site is created and
says nothing about whether anything is deployed. A site that is still `pending`, or whose
deploy failed, returns the same URL as a working one — read `state`, and check the URL
actually responds, before telling the user the site is live.

#### Upload a zip

1. `POST /api/v1/static` with `deployment_config: {"source_type": "zip"}`.
2. Wait until `deployment_config.bucket_name` is set.
3. Build the site locally. Zip the **contents of the output directory** — `index.html`
   must sit at the root of the archive, not inside a `dist/` folder.
4. `POST /api/v1/static/{uuid}/deployments` as `multipart/form-data` with the archive in
   the `archive` field. Max 1 GB.

```bash
curl -X POST "https://miget.com/api/v1/static/{uuid}/deployments" \
  -H "Authorization: Bearer $MIGET_API_TOKEN" \
  -F "archive=@site.zip"
```

Each upload **replaces** the whole site; files from the previous deploy that are absent
from the archive are removed.

#### Deploy over SFTP

1. `POST /api/v1/static` with `deployment_config: {"source_type": "sftp"}`.
2. Wait until `deployment_config.bucket_name` is set.
3. Make sure the user has an SSH key on their Miget account — the gateway authenticates
   with it, and there is no password. Check with `GET /api/v1/users/me/ssh_keys`. If the
   list is empty, read their public key (`~/.ssh/id_ed25519.pub`, or `ssh-keygen -t ed25519`
   if they have none) and add it with `POST /api/v1/users/me/ssh_keys`. Do not send a
   private key, and do not generate a key without telling them.
4. Read `deployment_config.sftp_username` and the site's `region` from
   `GET /api/v1/static/{uuid}`, then connect (`deployment_config.sftp_endpoint` carries the
   same `user@host` target ready-made):

```bash
sftp {sftp_username}@ssh.{region}.migetapp.com
```

5. Upload the **built** site into the session root (`put -r ./dist/*`, not `put -r ./dist`).
6. Disconnect. Closing the session is what triggers the deploy — nothing is published
   while you are still connected.

#### Deploy with git push

1. `POST /api/v1/static` with `deployment_config: {"source_type": "git_push"}`, plus any
   build overrides (`generator`, `build_command`, `output_dir`, `project_path`).
2. Wait until `deployment_config.git_ssh_url` is set.
3. Read `deployment_config.git_ssh_url` from `GET /api/v1/static/{uuid}` and add it as a
   remote. The user needs an SSH key on their account here too — same check as step 3 of
   the SFTP runbook, via `GET`/`POST /api/v1/users/me/ssh_keys`.

```bash
git remote add miget {git_ssh_url}
git push miget main
```

4. Every push builds the site and deploys the output. There is no deploy endpoint for
   this source — pushing *is* the deploy.

#### Deploy from GitHub

1. Get a `credential_id` from `GET /api/v1/git_credentials` (the GitHub App install is a
   browser flow done in the dashboard; you cannot create one over the API).
2. `POST /api/v1/static` with `source_type: "github"`, `credential_id`, `repository`
   (`owner/repo`) and `branch`. Set `auto_deploy_enabled: true` to build on every push.
3. `POST /api/v1/static/{uuid}/deployments` with **no** archive to build and deploy the
   configured branch. With auto-deploy on, pushes do this for you.

---

## API Structure

### Base Path

All endpoints are under: `/api/v1/`

### Common Headers

```http
Authorization: Bearer $MIGET_API_TOKEN
X-Workspace-Id: {workspace-uuid}  # Optional, uses default if omitted
Content-Type: application/json
```

### Response Format

- **Success:** JSON object or array
- **Error:** JSON object with error details. The format varies by error type:

  Single error (most common):
  ```json
  {
    "error": "Error message"
  }
  ```

  Validation errors (422 responses from app creation/update endpoints) use the
  key `errors`, but the value is a **single comma-joined string**, not an array:
  ```json
  {
    "errors": "Label is too long (maximum is 40 characters), Name is invalid"
  }
  ```

  Handle both keys, and do not assume `errors` is iterable — treat it as a string
  and split on `", "` only if you need the individual messages.

### Common HTTP Status Codes

- `200` - Success
- `201` - Created
- `204` - No Content (successful deletion)
- `400` - Bad Request (validation errors)
- `401` - Unauthorized (invalid/missing token)
- `403` - Forbidden (insufficient permissions)
- `404` - Not Found
- `422` - Unprocessable Entity (validation failed)
- `500` - Internal Server Error

---

## Common Workflows

### 1. Create and Deploy an Application

All deployment methods follow the same initial steps: create a resource, create a project, then create the app with a `deployment_method` and its corresponding `deployment_config`. The final deploy step varies by method.

```http
# Step 1: Create a resource (if needed)
# Pick plan_code_name from GET /api/v1/plans — never invent one.
POST /api/v1/resources
{
  "plan_code_name": "miget_hobby_0",
  "region_code": "eu-east-1"
}

# Step 2: Create a project (if needed)
POST /api/v1/projects
{
  "name": "my-project",
  "description": "My project description"
}

# Step 3: Create the application (example: GitHub deployment)
POST /api/v1/apps
{
  "name": "my-app",
  "label": "My Application",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "builder": "auto",
  "ram_size": 256,
  "cpu_size": 0.5,
  "deployment_method": "github",
  "deployment_config": {
    "credential_id": "{github-credential-uuid}",
    "repository": "username/repo",
    "branch": "main",
    "auto_deploy_enabled": true
  }
}

# Step 4: Deploy the application
POST /api/v1/apps/{app-uuid}/deploy
{
  "custom_tag": "v1.2.3",  # Optional: deploy a specific image tag
  "commit_sha": "abc123",  # Optional: deploy specific commit
  "branch": "main"         # Optional: deploy specific branch
}
```

**Alternative: Create with Docker Registry deployment**
```http
POST /api/v1/apps
{
  "name": "my-nginx",
  "label": "My Nginx",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "builder": "dockerfile",
  "ram_size": 256,
  "cpu_size": 0.5,
  "deployment_method": "container_registry",
  "deployment_config": {
    "image_url": "docker.io/library/nginx",
    "tag": "latest"
  }
}
```

**Alternative: Create with Kamal deployment**
```http
POST /api/v1/apps
{
  "name": "my-rails-app",
  "label": "My Rails App",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "builder": "dockerfile",
  "ram_size": 512,
  "cpu_size": 0.5,
  "deployment_method": "kamal",
  "deployment_config": {
    "ssh_keys": ["ssh-ed25519 AAAA... user@machine"]
  }
}
# Note: Kamal apps are deployed from the user's machine via `kamal deploy`, not via the API
# The registry password is auto-generated - retrieve it via GET /api/v1/apps/{uuid}
```

### 2. Monitor Deployment

```http
# List recent deployments
GET /api/v1/apps/{app-uuid}/deployments?period=7days&status=running

# Get deployment details
GET /api/v1/apps/{app-uuid}/deployments/{deployment-id}

# Stream build logs (SSE)
GET /api/v1/apps/{app-uuid}/deployments/{deployment-id}/stream_logs

# Or get stored logs (after deployment completes)
GET /api/v1/apps/{app-uuid}/deployments/{deployment-id}/logs

# Cancel a running deployment
POST /api/v1/apps/{app-uuid}/deployments/{deployment-id}/cancel

# Rollback to a previous deployment
POST /api/v1/apps/{app-uuid}/deployments/{deployment-id}/rollback
```

### 3. Manage Environment Variables

```http
# List variables
GET /api/v1/apps/{app-uuid}/vars

# Create variable
POST /api/v1/apps/{app-uuid}/vars
{
  "key": "DATABASE_URL",
  "value": "postgresql://..."
}

# Update variable (identified by key)
PUT /api/v1/apps/{app-uuid}/vars
{
  "key": "DATABASE_URL",
  "value": "new-value"
}

# Delete variable (identified by key)
DELETE /api/v1/apps/{app-uuid}/vars
{
  "key": "DATABASE_URL"
}
```

### 4. Add a Database Addon

```http
# Create addon
POST /api/v1/apps/{app-uuid}/addons
{
  "type": "postgres",
  "label": "Primary database",
  "postgres_version": "17"
}

# `label` and the type's version field are REQUIRED even though the API
# marks them optional — omitting either returns 422.
#
# The addon injects TWO variables into the app, both holding the same
# connection string:
#
#   1. <ADDON_NAME>_URL — the addon's own name, upcased with dashes turned
#      into underscores. A postgres addon named "postgres-mwvzq" yields
#      POSTGRES_MWVZQ_URL.
#   2. A generic alias — DATABASE_URL for postgres and mysql, REDIS_URL
#      for valkey.
#
# The alias is created only when the app has no variable of that name yet
# (compared case-insensitively). An existing DATABASE_URL is left alone,
# so an app pointed at some other database keeps pointing at it.
#
# There are no broken-out components — no DB_HOST, DB_PORT, DB_USER.
#
# So a framework reading DATABASE_URL works with no extra step. Confirm
# with GET /apps/{uuid}/vars rather than assuming either way, and do not
# create a second DATABASE_URL — you would be duplicating a variable the
# platform already set.
```

### 5. Add Custom Domain

```http
# Add domain
POST /api/v1/apps/{app-uuid}/domains
{
  "domain": "api.example.com"
}

# Update domain (e.g., enable SSL)
PUT /api/v1/apps/{app-uuid}/domains/{domain-uuid}
{
  "ssl_enabled": true
}
```

### 6. Create and Manage a Storage Bucket

```http
# Step 1: Create a bucket (requires an existing resource)
POST /api/v1/buckets
{
  "label": "My Assets Bucket",
  "resource_id": "01H...resource-uuid...",
  "project_id": "01H...project-uuid...",
  "visibility": "private_access",
  "disk_size": 1.0
}

# Step 2: Get bucket details (S3 endpoint, credentials, usage)
GET /api/v1/buckets/{bucket-uuid}

# Step 3: Upload a file (get presigned URL, then upload directly to S3)
POST /api/v1/buckets/{bucket-uuid}/objects/upload_url
{
  "key": "images/logo.png",
  "size": 102400,
  "content_type": "image/png"
}
# Response contains presigned URL - upload file directly via HTTP PUT to that URL

# Step 4: List objects in bucket
GET /api/v1/buckets/{bucket-uuid}/objects/list?prefix=images/&limit=50

# Step 5: Download a file (get presigned URL)
POST /api/v1/buckets/{bucket-uuid}/objects/download_url
{
  "key": "images/logo.png"
}

# Update bucket settings
PUT /api/v1/buckets/{bucket-uuid}
{
  "label": "Updated Label",
  "visibility": "public_access",
  "disk_size": 5.0
}

# Move the bucket to another project, or send null to unassign it
PUT /api/v1/buckets/{bucket-uuid}
{
  "project_id": "01H...project-uuid..."
}

# Delete a bucket
DELETE /api/v1/buckets/{bucket-uuid}
```

### 7. Configure Bucket Access (Policy & ACL)

Bucket policies and ACLs control who can access bucket contents and how. Users rarely know the exact format - your job is to understand their intent and build the configuration for them. Policies use JSON format; ACLs use XML format. See the "Bucket Policy & ACL" section under Required Fields for templates, step-by-step guidance, and example interactions.

```http
# Set a bucket policy
PUT /api/v1/buckets/{bucket-uuid}/policy
{
  "policy": "<S3-compatible policy JSON string>"
}

# Set a bucket ACL
PUT /api/v1/buckets/{bucket-uuid}/acl
{
  "acl": "<S3-compatible ACL XML string>"
}

# Remove policy or ACL (reverts to default access rules)
DELETE /api/v1/buckets/{bucket-uuid}/policy
DELETE /api/v1/buckets/{bucket-uuid}/acl
```

### 8. Create a PostgreSQL Read Replica

Read replicas provide read-only copies of a PostgreSQL database for scaling read workloads. Replicas share credentials with their primary and are managed as CloudNativePG replica clusters.

```http
# For app addons:
# Step 1: Get the primary addon details
GET /api/v1/apps/{app-uuid}/addons/{addon-id}
# Verify role is "primary" and type is "postgres"

# Step 2: Create the read replica
POST /api/v1/apps/{app-uuid}/addons/{addon-id}/create_replica

# For standalone services:
# Step 1: Get the primary service details
GET /api/v1/services/{service-id}
# Verify role is "primary" and service_type is "postgres"

# Step 2: Create the read replica
POST /api/v1/services/{service-id}/create_replica
```

**Important notes about replicas:**
- Only PostgreSQL **standalone** primary addons/services can have replicas (cluster/HA databases do not support read replicas - HA clusters already provide redundancy)
- Cannot create a replica of a replica
- Replicas share credentials (database name, username, password) with their primary
- Replicas use the same resource allocation (CPU, RAM, disk) as their primary
- Replicas do **not** support backups, restore, or database reset - these operations are only available on the primary
- Replicas do **not** have their own ports or environment variables
- Public access setting is inherited from the primary and cannot be changed independently - if the primary has public access enabled, the replica's `connection_details.external` will include external connection URLs
- Deleting a primary automatically deletes all its replicas
- Replica creation is asynchronous - poll the addon/service `state` to track provisioning, testing it against the ready set in "Reading `state`" (`healthy`, `bound`, `running`) rather than a single string
- Replicas can be promoted to standalone instances using the promote endpoint - this is irreversible and the promoted instance will no longer receive updates from the primary
- The `create_replica` endpoint returns the full serialized replica entity (same shape as `GET /api/v1/apps/{uuid}/addons/{id}` or `GET /api/v1/services/{id}`), including `uuid`, `role: "replica"`, `primary_addon_uuid`, and `connection_details` - no follow-up `GET` is required to discover the new replica

**Response fields for PostgreSQL addons/services:**
- `role` (string) - `"primary"` or `"replica"` (null for non-PostgreSQL)
- `primary_addon_uuid` (string) - UUID of the primary addon (only present on replicas)
- `replicas` (array) - List of replicas with `uuid`, `name`, `label`, `state` (only present on primaries, in show endpoints)

### 9. Manage App Ports

App ports are managed through the standard app-UUID-scoped endpoints. Port `5000` is auto-created for HTTP traffic; use these endpoints to add extra TCP/UDP ports.

```http
# List all ports
GET /api/v1/apps/{app-uuid}/ports

# Create a new port
POST /api/v1/apps/{app-uuid}/ports
{
  "internal_port": 8080,
  "protocol": "tcp",
  "public": false
}

# Get port details
GET /api/v1/apps/{app-uuid}/ports/{port-id}

# Expose port publicly
PATCH /api/v1/apps/{app-uuid}/ports/{port-id}/expose_publicly

# Make port private
PATCH /api/v1/apps/{app-uuid}/ports/{port-id}/make_private

# Delete port
DELETE /api/v1/apps/{app-uuid}/ports/{port-id}
```

### 10. Create and Deploy with Kamal

Kamal is a deployment method where the user deploys from their local machine using `kamal deploy`. Unlike other methods, Miget does not build or deploy the app - it provides the infrastructure (SSH endpoint, container registry) and the user runs Kamal locally.

**Important constraints:**
- Apps must be created with Kamal from the start - you cannot switch an existing app to Kamal
- The deploy button is not used for Kamal apps (the user deploys from their machine)
- The registry password is auto-generated by the platform - retrieve it via `GET /api/v1/apps/{uuid}`

```http
# Step 1: Create resource and project (same as other methods)

# Step 2: Create app with Kamal deployment method
POST /api/v1/apps
{
  "name": "my-rails-app",
  "label": "My Rails App",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "builder": "dockerfile",
  "deployment_method": "kamal",
  "deployment_config": {
    "ssh_keys": ["ssh-ed25519 AAAA... user@machine"]
  },
  "ram_size": 512,
  "cpu_size": 0.5
}

# Step 3: Get app details to retrieve deploy.yml values
GET /api/v1/apps/{app-uuid}
# Response includes deployment_config with all values needed for config/deploy.yml:
# {
#   "deployment_method": "kamal",
#   "deployment_config": {
#     "type": "kamal",
#     "ssh_keys": ["ssh-ed25519 AAAA... user@machine"],
#     "registry_image": "registry.eu-east-1.miget.io/my-resource/my-rails-app",
#     "registry_hostname": "registry.eu-east-1.miget.io",
#     "registry_username": "my-resource",
#     "registry_password": "auto-generated-password"
#   }
# }

# Step 4: Update SSH keys (if needed)
PUT /api/v1/apps/{app-uuid}/deployment
{
  "deployment_method": "kamal",
  "deployment_config_attributes": {
    "ssh_keys": ["ssh-ed25519 AAAA... user@machine", "ssh-rsa AAAA... other@machine"]
  }
}

# Step 5: User deploys from their machine using Kamal CLI
# kamal deploy (from the user's local machine, not via API)
```

### 11. Create a Compose Stack (Docker Compose)

```http
# Step 1: Analyze the repo to discover services and required env vars (creates nothing)
POST /api/v1/stacks/analyze
{
  "repository_url": "https://github.com/acme/shop.git",
  "branch": "main",
  "compose_path": "."
}
# Response: { "manifest": { "apps": [...], "managed_services": [...] }, "warnings": [] }
# Inspect each service's env_vars for entries with "required": true and a blank "value",
# then ask the user whether to supply them or auto-populate (good for secrets).

# Step 2: Create the stack, supplying required env vars
POST /api/v1/stacks
{
  "repository_url": "https://github.com/acme/shop.git",
  "branch": "main",
  "resource_id": "{miget-uuid}",
  "project_id": "{project-uuid}",
  "label": "Shop",
  "env_var_overrides": { "web": { "STRIPE_API_KEY": "sk_live_..." } },
  "auto_populate_required_vars": true
}

# Step 3: Watch deployment progress
GET /api/v1/stacks/{stack-uuid}/deployments
```

### 11a. Deploy a Known App from the Miget Catalogue (deployable.sh)

Miget curates ready-to-run, platform-tuned Compose stacks — WordPress, Ghost, n8n, Kafka,
Supabase, Metabase, and many more — in the public **deployable.sh** catalogue (repo
`deployable-sh/stacks`). Each stack directory ships a `compose.miget.yaml` carrying the
platform overrides a raw compose file lacks (port 5000 ingress, `private: true` defaults,
RAM sizing, managed-service wiring), so these deploy correctly out of the box.

**When the user asks to run a well-known self-hostable app (e.g. "deploy WordPress", "spin up
Ghost"), prefer the catalogue over an arbitrary compose file found on the web** — a random
`docker-compose.yml` from the internet almost never has Miget's overrides and will likely fail
to deploy.

1. **Find it in the catalogue.** Look the app up in `deployable-sh/stacks`: list the repo's
   top-level directories (or browse https://deployable.sh) and match the app to a directory. The
   slug is the app's lowercased name (e.g. `wordpress`, `ghost`, `n8n`). Do **not** search the
   internet for a compose file when the app exists here.
2. **Deploy it** as a normal Compose Stack (section 11), pointing `repository_url` at the
   catalogue repo and `compose_path` at the stack directory:

```http
POST /api/v1/stacks/analyze
{
  "repository_url": "https://github.com/deployable-sh/stacks.git",
  "branch": "main",
  "compose_path": "wordpress"
}
# Then POST /api/v1/stacks with the same source plus any required env vars (Step 2 above).
```

**If the app is not in the catalogue**, don't grab a random compose file off the web — ask the
user for a repository (public Git or GitHub) instead.

### 11b. Repos with a compose file but no `compose.miget.yaml`

When deploying a user's **own** repository whose base `docker-compose.yml` has no
`compose.miget.yaml` beside it, the stack still deploys, but without Miget's per-service tuning —
services fall back to default sizing and exposure. **Offer to create a `compose.miget.yaml`
overlay** (a sibling of the compose file) and, once the user confirms, generate it and add it to
their repo. Miget merges it onto the base compose at detect/deploy time; it carries only
`x-miget` overrides:

```yaml
# compose.miget.yaml — Miget overlay, merged onto your docker-compose at deploy time
services:
  web:                 # the public HTTP entry — must listen on port 5000 (Miget's only ingress port)
    x-miget:
      ram: "1024"      # memory: plain MB, or a unit like "1Gi"
  worker:
    x-miget:
      ram: "512"
      private: true    # internal only, not publicly exposed
  db:                  # a database/cache -> provision as a managed add-on, not a raw container
    x-miget:
      managed: postgres  # supported: postgres, valkey
      cpu: "500m"
      ram: "1Gi"
      storage: "5Gi"
  cache:
    x-miget:
      managed: valkey
      ram: "256Mi"
volumes:
  webdata:
    x-miget: { size: "5000", type: RWO }   # disk: MB or a unit; RWO (default) or RWX (shared)
```

Rules when generating it:
- Give every service an `x-miget.ram` (plain MB or a unit like `1Gi`); add `cpu` (e.g. `500m`) when known.
- Exactly one service is the public HTTP entry and must listen on **port 5000**; mark every other
  service `x-miget: { private: true }`.
- For databases/caches, use `x-miget.managed: <postgres|valkey>` (with `storage`) so Miget runs them
  as managed add-ons and injects their connection variables — don't run them as raw containers.
- Give every named volume an `x-miget: { size: "<MB or unit>", type: RWO }` (`RWX` only when the
  volume is shared across replicas).

Re-run `POST /api/v1/stacks/analyze` after adding the file to confirm the detected services and
sizing. Full field reference: the "Docker Compose Stacks" page in the Miget docs.

---

## Key Endpoints

### Authentication

- `POST /api/v1/auth/sign_in` - Authenticate with email/password
- `POST /api/v1/auth/refresh_token` - Refresh access token

### Applications

- `GET /api/v1/apps` - List all applications
- `POST /api/v1/apps` - Create new application
- `GET /api/v1/apps/{uuid}` - Get application details (includes `deployment_method` and `deployment_config` with method-specific fields; for Kamal apps this includes `registry_password`, `registry_hostname`, `registry_username`, `registry_image`, and `ssh_keys`; also includes a nested `region` object with `id`, `name`, and `code`, plus a `private_access` boolean)
- `PUT /api/v1/apps/{uuid}` - Update application
- `DELETE /api/v1/apps/{uuid}` - Delete application
- `PUT /api/v1/apps/{uuid}/security` - Update security settings (network connectivity, Basic Authentication)
- `PATCH /api/v1/apps/{uuid}/state` - Change app state (schedule_start/schedule_stop/schedule_restart)
- `POST /api/v1/apps/{uuid}/clone` - Clone an application. Copies nothing by default beyond the app's own settings — env vars, secret files, scaling, health checks, security, add-ons and cronjobs are each opt-in. See the runbook under Endpoint Reference
- `PUT /api/v1/apps/{uuid}/deployment` - Update deployment method and configuration (switch methods, update Kamal SSH keys). `deployment_config_attributes` is a **patch**: fields you omit keep their stored value, and a field sent as `""` is cleared. Sending a *different* `deployment_method` builds the config from scratch, so supply every field that method needs.
- `POST /api/v1/apps/{uuid}/deploy` - Trigger deployment (optional: custom_tag, commit_sha, branch). Not used for Kamal apps. Returns `409 Conflict` if a deployment is already in progress — poll `GET /apps/{uuid}/deployments` and retry once it settles. On a `github` app, a `commit_sha` that does not exist in the configured repository is rejected with `422` before any build starts, so push the commit first and pass a SHA from the same repository the app is configured with (a SHA from a fork or a squashed/force-pushed branch will not resolve). Other deployment methods do not check the SHA.
- `PUT /api/v1/apps/{uuid}/health_checks` - Update health check probes (liveness, readiness, startup)
- `PUT /api/v1/apps/{uuid}/scaling_profile` - Update scaling profile (replicas, auto-scaling, thresholds). Not available on free plan.
- `GET /api/v1/apps/{uuid}/deployments` - List deployments
- `GET /api/v1/apps/{uuid}/deployments/{id}/logs` - Get build logs
- `GET /api/v1/apps/{uuid}/activity` - Get paginated activity feed (deployments, config changes, audit events). Query params: `page`, `limit`. Returns an envelope `{ "activities": [{ action, description, resource, actor, timestamp, source }], "pagination": { page, limit, total } }`.

### Resources

- `GET /api/v1/resources` - List all resources
- `POST /api/v1/resources` - Create new resource
- `GET /api/v1/resources/{uuid}` - Get resource details
- `PUT /api/v1/resources/{uuid}` - Update resource (change plan, add components)
- `PATCH /api/v1/resources/{uuid}/labels` - Update resource labels
- `DELETE /api/v1/resources/{uuid}` - Delete resource

### Projects

- `GET /api/v1/projects` - List all projects
- `POST /api/v1/projects` - Create new project
- `GET /api/v1/projects/{project_id}` - Get project details
- `PUT /api/v1/projects/{project_id}` - Update project
- `DELETE /api/v1/projects/{project_id}` - Delete project
- `GET /api/v1/projects/{project_id}/apps` - List applications in project
- `POST /api/v1/projects/{project_id}/resources` - Assign a resource to the project (body: `resource_id`, optional `with_hosting_projects`). Needs **both** `projects:manage` and `resources:manage`, because assigning narrows who may use the resource. Refused with **422** while the resource still runs workloads the assignment would lock out — pass `with_hosting_projects: true` to assign it to those projects as well instead of being refused
- `DELETE /api/v1/projects/{project_id}/resources/{resource_id}` - Remove the assignment, returning the resource to the shared pool. Needs only `projects:manage`: releasing only ever widens access, and every workload already on the resource stays legal. **422** when that resource is not assigned to that project
- `POST /api/v1/projects/{project_id}/restrictions` - Grant access to the project. Needs `projects:manage`, and you must be able to reach the project yourself. Body carries **exactly one** of `user_email` (a workspace member) or `role_name` (a workspace role — everyone holding it reaches the project); sending both or neither is a **400**. Adding the first entry is what closes an otherwise open project. The **422** replies are distinct, so read the message rather than retrying: `Restricting projects is available on organization and enterprise plans` (checked first, before anything about the subject), `No member of this workspace has the email <address>`, `This workspace has no role named <name>`, `Subject already has access to this project`, and — for a workspace owner who holds no membership row — `The workspace owner already reaches every project`, which means no entry is needed rather than that something failed
- `DELETE /api/v1/projects/{project_id}/restrictions/{id}` - Revoke one entry, where `{id}` is the numeric `id` from the project's `restrictions` list. Removing the last entry reopens the project to the whole workspace

### Project Environment Variables

- `GET /api/v1/projects/{project_id}/vars` - List project environment variables
- `POST /api/v1/projects/{project_id}/vars` - Create project environment variable
- `PUT /api/v1/projects/{project_id}/vars` - Update project environment variable
- `DELETE /api/v1/projects/{project_id}/vars` - Delete project environment variable

### App Domains

- `GET /api/v1/apps/{uuid}/domains` - List app domains
- `POST /api/v1/apps/{uuid}/domains` - Add domain
- `GET /api/v1/apps/{uuid}/domains/{domain_uuid}` - Get domain details (returns `verification_status`, `verification_token`, `dns_target`)
- `PUT /api/v1/apps/{uuid}/domains/{domain_uuid}` - Update domain
- `DELETE /api/v1/apps/{uuid}/domains/{domain_uuid}` - Remove domain
- `POST /api/v1/apps/{uuid}/domains/{domain_uuid}/verify` - Trigger DNS verification. Caller is expected to have already published `TXT _migetapp-verify.<domain> = <verification_token>` (returned by GET). The check runs async (Fibonacci backoff up to 60min); poll the GET endpoint until `verification_status` becomes `verified` and `dns_target` is populated.

### App Environment Variables

- `GET /api/v1/apps/{uuid}/vars` - List app variables
- `POST /api/v1/apps/{uuid}/vars` - Create variable
- `PUT /api/v1/apps/{uuid}/vars` - Update variable (identified by `key` in body)
- `DELETE /api/v1/apps/{uuid}/vars` - Delete variable (identified by `key` in body)

### App Addons

- `GET /api/v1/apps/{uuid}/addons` - List app addons (includes `role` and `primary_addon_uuid` fields for PostgreSQL addons)
- `POST /api/v1/apps/{uuid}/addons` - Create addon (PostgreSQL supports `creation_mode`: `fresh` or `external_replica`, and `instances` [1, 3, 5, 7] for HA clusters)
- `GET /api/v1/apps/{uuid}/addons/{id}` - Get addon details (includes `role`, `primary_addon_uuid`, and `replicas` for PostgreSQL primaries)
- `PUT /api/v1/apps/{uuid}/addons/{id}` - Update addon (PostgreSQL supports `instances` [1, 3, 5, 7] for scaling cluster nodes)
- `DELETE /api/v1/apps/{uuid}/addons/{id}` - Delete addon (deleting a primary cascades to all its replicas)
- `PATCH /api/v1/apps/{uuid}/addons/{id}/state` - Change addon state (process_start/process_stop/process_restart)
- `POST /api/v1/apps/{uuid}/addons/{id}/rotate_password` - Rotate addon password (only for certain addon types - databases like PostgreSQL/MySQL, caches like Valkey)
- `POST /api/v1/apps/{uuid}/addons/{id}/create_replica` - Create a read replica of a PostgreSQL addon (primary only, optional `cpu_size`/`ram_size` params, requires `apps:operate`)
- `POST /api/v1/apps/{uuid}/addons/{id}/promote_replica` - Promote a read replica to a standalone PostgreSQL instance (replica only, requires `apps:operate`)
- `POST /api/v1/apps/{uuid}/addons/{id}/promote_external` - Promote an external replica to standalone instance or cluster by disconnecting from external source (preserves current mode, requires `apps:operate`)
- `GET /api/v1/apps/{uuid}/addons/{id}/backups` - Get addon backups (PostgreSQL primary only, not available for replicas)
- `POST /api/v1/apps/{uuid}/addons/{id}/restore_backup` - Restore addon from backup (PostgreSQL primary only, not available for replicas)
- `POST /api/v1/apps/{uuid}/addons/{id}/reset_database` - Reset addon database (PostgreSQL primary only, not available for replicas)

### App Cronjobs

- `GET /api/v1/apps/{uuid}/cronjobs` - List cronjobs
- `POST /api/v1/apps/{uuid}/cronjobs` - Create cronjob
- `GET /api/v1/apps/{uuid}/cronjobs/{id}` - Get cronjob details
- `PUT /api/v1/apps/{uuid}/cronjobs/{id}` - Update cronjob (**only `label` and `command` are updatable**; the schedule cannot be changed via PUT — DELETE and recreate the cronjob to reschedule)
- `DELETE /api/v1/apps/{uuid}/cronjobs/{id}` - Delete cronjob
- `GET /api/v1/apps/{uuid}/cronjobs/{id}/stream_logs` - Stream the most recent run's logs in real-time (SSE, text/event-stream; returns 404 until the job has run at least once)

### App Deployments

- `GET /api/v1/apps/{uuid}/deployments` - List deployments (optional filters: `status` (pending/running/completed/failed/cancelling/cancelled), `period` (7days/30days/90days/all)). Each record includes `commit_sha`, `commit_message`, and `branch` for git-based deployment methods (null otherwise).
- `GET /api/v1/apps/{uuid}/deployments/{id}` - Get deployment details
- `GET /api/v1/apps/{uuid}/deployments/{id}/logs` - Get build logs (text/plain, available after deployment completes)
- `GET /api/v1/apps/{uuid}/deployments/{id}/stream_logs` - Stream logs in real-time (SSE, text/event-stream, for running deployments)
- `POST /api/v1/apps/{uuid}/deployments/{id}/cancel` - Cancel running deployment (only deployments in 'running' state can be cancelled)
- `POST /api/v1/apps/{uuid}/deployments/{id}/rollback` - Rollback to a previous deployment (only rollbackable deployments can be rolled back - must have image URL, same deployment method as app, and not be running/failed)

### App Ports

- `GET /api/v1/apps/{uuid}/ports` - List all ports for an app
- `POST /api/v1/apps/{uuid}/ports` - Create a new port (requires apps:manage, not available on free plan)
- `GET /api/v1/apps/{uuid}/ports/{port_id}` - Get port details
- `DELETE /api/v1/apps/{uuid}/ports/{port_id}` - Delete a port
- `PATCH /api/v1/apps/{uuid}/ports/{port_id}/expose_publicly` - Expose a port publicly (requires apps:manage, not available on free plan)
- `PATCH /api/v1/apps/{uuid}/ports/{port_id}/make_private` - Make a port private (requires apps:manage)

### Buckets (S3-Compatible Object Storage)

- `GET /api/v1/buckets` - List all buckets
- `POST /api/v1/buckets` - Create new bucket
- `GET /api/v1/buckets/{uuid}` - Get bucket details (includes S3 endpoint, S3 credentials, usage stats)
- `PUT /api/v1/buckets/{uuid}` - Update bucket (label, project, visibility, disk size)
- `DELETE /api/v1/buckets/{uuid}` - Delete bucket and all its contents
- `PUT /api/v1/buckets/{uuid}/policy` - Set or update bucket policy (S3-compatible JSON)
- `DELETE /api/v1/buckets/{uuid}/policy` - Remove bucket policy
- `PUT /api/v1/buckets/{uuid}/acl` - Set or update bucket ACL (S3-compatible XML)
- `DELETE /api/v1/buckets/{uuid}/acl` - Remove bucket ACL

### Bucket Objects (Files & Folders)

- `GET /api/v1/buckets/{uuid}/objects/list` - List objects in bucket (supports `prefix`, `limit`, `cursor`)
- `POST /api/v1/buckets/{uuid}/objects/upload_url` - Generate presigned upload URL
- `POST /api/v1/buckets/{uuid}/objects/download_url` - Generate presigned download URL
- `POST /api/v1/buckets/{uuid}/objects/create_folder` - Create a folder
- `PUT /api/v1/buckets/{uuid}/objects/{key}/rename` - Rename an object
- `DELETE /api/v1/buckets/{uuid}/objects/{key}` - Delete an object or folder

### Services

- `GET /api/v1/services` - List all services (includes `role` and `primary_addon_uuid` fields for PostgreSQL services)
- `POST /api/v1/services` - Create service (types: postgres, shared_storage. PostgreSQL supports `creation_mode`: `fresh` or `external_replica`, and `instances` [1, 3, 5, 7] for HA clusters)
- `GET /api/v1/services/{id}` - Get service details (includes `role`, `primary_addon_uuid`, and `replicas` for PostgreSQL primaries)
- `PUT /api/v1/services/{id}` - Update service (PostgreSQL supports `instances` [1, 3, 5, 7] for scaling cluster nodes). `project_id` moves the service to another project (requires `services:manage`); a service that belongs to a stack moves with the stack instead
- `DELETE /api/v1/services/{id}` - Delete service (deleting a primary cascades to all its replicas)
- `PATCH /api/v1/services/{id}/state` - Change service state (process_start/process_stop/process_restart)
- `POST /api/v1/services/{id}/rotate_password` - Rotate service password (databases and caches only, returns new password)
- `POST /api/v1/services/{id}/mount_app` - Mount an application to a service (creates storage addon on app linked to this service)
- `POST /api/v1/services/{id}/unmount_app` - Unmount an application from a service
- `POST /api/v1/services/{id}/create_replica` - Create a read replica of a PostgreSQL service (primary only, optional `cpu_size`/`ram_size` params, requires `services:operate`)
- `POST /api/v1/services/{id}/promote_replica` - Promote a read replica to a standalone PostgreSQL instance (replica only, requires `services:operate`)
- `POST /api/v1/services/{id}/promote_external` - Promote a service external replica to standalone instance or cluster by disconnecting from external source (preserves current mode, requires `services:operate`)
- `GET /api/v1/services/{id}/backups` - Get service backups (PostgreSQL primary only, not available for replicas)
- `POST /api/v1/services/{id}/restore_backup` - Restore service from backup (PostgreSQL primary only, not available for replicas)
- `POST /api/v1/services/{id}/reset_database` - Reset service database (PostgreSQL primary only, not available for replicas)

### Stacks (Docker Compose)

Stacks reuse `apps:*` permissions (read = `apps:view`, create/delete = `apps:manage`, deploy/config = `apps:deploy`/`operate`/`manage`).

- `GET /api/v1/stacks` - List all stacks
- `POST /api/v1/stacks/analyze` - Detect compose services and required env vars from a repo (creates nothing; call before creating)
- `POST /api/v1/stacks` - Create a stack (analyzes the repo server-side, then provisions the apps/services)
- `GET /api/v1/stacks/{uuid}` - Get stack details (computed `state`, `services`, `latest_deployment`, `deployment_config`)
- `PUT /api/v1/stacks/{uuid}` - Update a stack (`label`, `compose_path`). `project_id` moves the stack, and every application and service it runs, to another project (requires `apps:manage`)
- `DELETE /api/v1/stacks/{uuid}` - Delete a stack (cascades to its apps and services)
- `POST /api/v1/stacks/{uuid}/deploy` - Trigger a redeploy (optional: `commit_sha`)
- `PUT /api/v1/stacks/{uuid}/deployment` - Update the GitHub deployment config (`branch`, `auto_deploy_enabled`, `repository`)
- `GET /api/v1/stacks/{uuid}/deployments` - List deployment history
- `GET /api/v1/stacks/{uuid}/deployments/{id}` - Get a single deployment (`{id}` is the deployment UUID)

### Plans, Regions & Components

- `GET /api/v1/plans` - List available plans. Each plan returns `code_name` (the opaque identifier you pass to `POST /api/v1/resources`), `plan_type` (`dev` for development/hobby, `pro` for production), `ram_size` and `disk_size` in **bytes**, `cpu_size` as a fractional core count, and `unit_price` in **cents**. Sizing note: `ram_size: 268435456` is 256 MiB, not 256 MB — compare it against `quota.ram_size` on apps, which uses the same unit.
- `GET /api/v1/regions` - List available regions. Current regions: `eu-east-1` (Warsaw), `us-east-1` (Vint Hill)
- `GET /api/v1/components` - List available resource components (extra_ram, extra_cpu, extra_disk)

### Users

- `GET /api/v1/users/me` - Get current user profile
- `GET /api/v1/users/me/credits` - Get credit balance, referral code, and credit operation history

### SSH Keys

- `GET /api/v1/users/me/ssh_keys` - List SSH keys
- `POST /api/v1/users/me/ssh_keys` - Add an SSH key
- `GET /api/v1/users/me/ssh_keys/{id}` - Get SSH key details
- `DELETE /api/v1/users/me/ssh_keys/{id}` - Remove an SSH key

### Container Registry Credentials

Workspace-level credentials for pulling images from private registries. The returned `uuid` is what you pass as `deployment_config.credential_id` for `container_registry` (and `github`/`public_git`) deployments. The `token` is encrypted at rest and **never returned** by the API.

- `GET /api/v1/container_registry_credentials` - List credentials
- `POST /api/v1/container_registry_credentials` - Create a credential
- `GET /api/v1/container_registry_credentials/{uuid}` - Get credential details
- `PUT /api/v1/container_registry_credentials/{uuid}` - Update a credential (rotate token, change registry/username)
- `DELETE /api/v1/container_registry_credentials/{uuid}` - Delete a credential

### Git Credentials

Workspace-level Git credentials (GitHub App installs / personal access tokens) used to clone private repositories. The returned `uuid` is what you pass as `credential_id` when creating a **stack** (`POST /api/v1/stacks`) or a `github`/`public_git` **app**. Read-only over the API; the access token is encrypted at rest and **never returned**. (Creating a GitHub App credential is a browser-based install flow, done in the dashboard.)

- `GET /api/v1/git_credentials` - List git credentials (returns `uuid`, `name`, `provider`, `installation_id`)
- `GET /api/v1/git_credentials/{uuid}` - Get credential details

### Static Sites

Separate from applications: a static site never appears in `GET /api/v1/apps` and cannot
be created or read there. Read "Static Sites" under Core Concepts first — the name rules
and the fixed content source are the two things that trip up a first attempt.

- `GET /api/v1/static` - List static sites
- `POST /api/v1/static` - Create a static site
- `GET /api/v1/static/{uuid}` - Get a static site
- `PUT /api/v1/static/{uuid}` - Update the label, or move the site to another project
- `DELETE /api/v1/static/{uuid}` - Delete the site, its content and its domains
- `PUT /api/v1/static/{uuid}/deployment` - Update build and routing settings. Applied as
  a **patch**: omitted fields keep their stored value. `source_type` is not accepted here
- `POST /api/v1/static/{uuid}/deployments` - Deploy. Send multipart `archive` (zip, max
  1 GB) for a `zip` site; omit it to rebuild a `github` site. Returns `409 Conflict` while
  a deployment is in flight
- `GET /api/v1/static/{uuid}/deployments` - Deployment history
- `POST /api/v1/static/{uuid}/deployments/{id}/cancel` - Cancel a running build. Only a
  `git_push` or `github` site has a build to cancel; a `zip` or `sftp` deployment is a
  direct sync with nothing to stop
- `POST /api/v1/static/{uuid}/deployments/{id}/rollback` - Rebuild and republish the site
  from the commit that deployment shipped. **`github` only** — a static site produces no
  image, so this is a fresh build of that commit rather than a redeploy of a stored
  artifact. A `git_push` site is rebuilt by pushing, and `zip`/`sftp` deployments carry no
  commit; both return `422`
- `GET /api/v1/static/{uuid}/files` - Browse the deployed content, one directory level per
  call (`prefix`, `limit`, `cursor`). Read-only — deploy to change content. Returns `422`
  while the site's storage is still being provisioned
- `GET|POST /api/v1/static/{uuid}/domains` and `PUT|DELETE .../domains/{domain_uuid}`,
  `POST .../domains/{domain_uuid}/verify` - Custom domains, same shape as an app's

**Creating one.** `label`, `name` and `deployment_config.source_type` are required. Supply
either `project_id` or `new_project_name`:

```json
{
  "label": "Marketing site",
  "name": "acme-marketing",
  "project_id": "{project-uuid}",
  "deployment_config": {
    "source_type": "github",
    "credential_id": "{git-credential-uuid}",
    "repository": "acme/marketing",
    "branch": "main",
    "auto_deploy_enabled": true,
    "generator": "auto",
    "spa_mode": false
  }
}
```

For `zip` or `sftp`, `deployment_config` is just `{"source_type": "zip"}` — there is
nothing to build. Set `spa_mode: true` for a client-side router, so unknown paths serve
`/index.html` instead of a 404.

The site is provisioned asynchronously and stays `pending` until content has been
deployed to it, so do not treat `pending` as "not ready yet" and do not wait for it to
change before deploying — see "Every source starts by creating the site" for what to
poll instead.

### Webhooks

Workspace-level outbound webhooks. Miget POSTs a signed JSON payload to your endpoint when a subscribed event occurs, so you can react to deploys without polling `GET /api/v1/apps/{uuid}/deployments`. Requires `workspace:general` (admin only).

- `GET /api/v1/webhooks` - List webhooks
- `POST /api/v1/webhooks` - Create a webhook (**the only response containing `secret`**)
- `GET /api/v1/webhooks/{uuid}` - Get a webhook
- `PUT /api/v1/webhooks/{uuid}` - Update a webhook (send only the fields you want changed)
- `DELETE /api/v1/webhooks/{uuid}` - Delete a webhook
- `POST /api/v1/webhooks/{uuid}/test` - Send a test event and get the result back immediately

**Testing an endpoint.** `POST /api/v1/webhooks/{uuid}/test` POSTs a signed event to the URL right away and returns the resulting delivery, so you can verify an endpoint without waiting for a deployment. It is signed exactly like a real delivery, so it also exercises your signature verification. It responds `200` whether or not the endpoint accepted it — read `status` and `response_code` on the returned delivery. It is not retried, works on a disabled webhook, and is recorded in the delivery history.

**Handle `type: "ping"`.** The test event carries `"type": "ping"` with an empty `data` object. `ping` is *not* one of the subscribable event types, so a consumer that rejects unknown types will fail the test even though the endpoint is otherwise fine. Either accept `ping` explicitly or ignore unknown types.

**Create fields:** `name` (unique per workspace), `url` (http or https), `event_filter` (array), optional `app_uuids` (array) and `enabled` (default `true`).

**The endpoint must be publicly reachable.** A `url` pointing into private address space — loopback, RFC1918, link-local (including `169.254.169.254`), CGNAT, or the `localhost`, `.local` and `.internal` hostnames — is rejected with `422`. The host is re-resolved before every delivery, so a name that later starts resolving to a private address stops being delivered to rather than being retried.

**Scoping to specific apps.** By default a webhook receives events from **every app in the workspace**, including apps created later. Pass `app_uuids` to narrow it to specific apps; send an empty array to widen it back. Apps must belong to the same workspace or the request is rejected with `422`. Either way the payload carries `data.app_id`, so a consumer can filter on its own as well.

**Event types:** `deploy_started` (a deployment enters `running`) and `deploy_ended` (a deployment reaches `completed`, `failed`, or `cancelled`). There is no separate build event — on Miget the build and the deployment are one lifecycle, and `build_id` *is* the deployment UUID.

**Static site deployments fire these events too**, with `data.app_id` set to the static site's UUID — the same UUID `GET /api/v1/static` returns, and the one to pass in `app_uuids` to scope a webhook to a site. A `zip` or `sftp` deployment carries no commit, so `commit_sha`, `commit_message` and `branch` come back `null`.

**The secret is returned exactly once.** `POST /api/v1/webhooks` is the only response carrying `secret`; every other endpoint omits it, and there is no rotation endpoint. Store it when you create the webhook — to replace it, delete the webhook and create a new one.

**Payload:**

```json
{
  "id": "01952d3f-...",
  "type": "deploy_ended",
  "timestamp": "2026-08-05T12:00:00Z",
  "data": {
    "id": "<deployment uuid>",
    "app_id": "<app uuid>",
    "app_name": "my-api-x7k2p",
    "status": "completed",
    "commit_sha": "abc123",
    "commit_message": "Fix the thing",
    "branch": "main",
    "created_at": "2026-08-05T11:58:00Z"
  }
}
```

**Verifying a delivery.** Requests follow the [Standard Webhooks](https://standardwebhooks.com) specification, so any Standard Webhooks library verifies them as-is. Three headers accompany every POST:

| Header | Value |
|---|---|
| `webhook-id` | The event UUID (also `id` in the body) |
| `webhook-timestamp` | Unix seconds at signing time |
| `webhook-signature` | `v1,<base64>` |

The signature is `HMAC-SHA256(key, "{webhook-id}.{webhook-timestamp}.{body}")`, base64-encoded, where `key` is the base64-decoded portion of the secret after the `whsec_` prefix. Compare with a constant-time comparison, and reject deliveries whose `webhook-timestamp` is more than a few minutes old — that is what makes a captured request non-replayable.

**Retries.** A non-2xx response is retried with exponential backoff over roughly **24 hours** — 9 attempts in total, spaced 30s, 2m, 8m, 30m, 1h, 3h, 8h and 12h apart with jitter. Endpoints should be idempotent: deduplicate on `id`, which is stable across retries of the same event *and* across manual replays. A failing endpoint is never auto-disabled.

**Delivery history.**

- `GET /api/v1/webhooks/{uuid}/deliveries` — the recent history, newest first. One entry per event, updated in place across retries; `attempts` is how many times it has been tried, `status` is `pending`, `delivered` or `failed`, `response_code` and `message` hold what the endpoint returned (`message` is truncated to 500 characters, and carries the transport error when no response arrived). `payload` is the exact JSON body that was POSTed, so you can compare what you received against what was sent. Only the last 50 events per webhook are kept.
- `POST /api/v1/webhooks/{uuid}/deliveries/{delivery_uuid}/retry` — replay a delivery from its stored payload, reusing the original event id. Returns the delivery with status `pending`; poll the list for the outcome. `422` if the webhook is disabled.

Use these to diagnose a webhook that appears silent: a `failed` entry with `response_code: 502` is a problem on your side, while no entries at all means no subscribed event has fired.

---

## Permissions & Authorization

Miget uses **role-based access control (RBAC)** within workspaces.

### Common Permissions

- `apps:view` - View applications
- `apps:manage` - Create, update and delete applications
- `apps:deploy` - Deploy applications
- `apps:operate` - Operate applications (start/stop, manage addons)
- `resources:view` - View resources
- `resources:operate` - Operate resources
- `resources:manage` - Create, update and delete resources
- `projects:view` - View projects
- `projects:operate` - Operate projects
- `projects:manage` - Create, update and delete projects
- `services:view` - View services
- `services:operate` - Operate services
- `services:manage` - Create, update and delete services
- `buckets:view` - View buckets and list objects
- `buckets:operate` - Operate buckets (upload, download, manage policy/ACL)
- `buckets:manage` - Create and delete buckets
- `workspace:general` - Manage general workspace settings, API tokens and webhooks
- `workspace:security` - Manage workspace security settings
- `workspace:credentials` - Manage git and registry credentials
- `workspace:integrations` - Manage workspace integrations
- `workspace:billing` - Manage the plan and billing
- `workspace:members` - Manage workspace members
- `workspace:roles` - Manage workspace roles

Workspace owners have all permissions automatically.

If you get a `403 Forbidden` error, the user doesn't have the required permission for that operation. Let them know which permission is needed so they can request it from a workspace admin.

---

## Required Fields for Creation Endpoints

This section lists what each endpoint needs. It is a schema reference, not an interview script — derive what you can from the repository and the account first (see "How to Work"), fold the result into a single plan card, and ask only about what is left. What you must not do is *guess silently*: an unstated assumption about region or plan produces resources in the wrong place that then have to be deleted and recreated. Infer, say what you inferred and from where, and confirm once.

### Create Application (`POST /api/v1/apps`)

**Required fields:**
- `name` (string) - Service name seed (lowercase, alphanumeric with hyphens). **The server appends a random suffix**, so the app you get back is named `my-api-x7k2p`, not `my-api`. Never build a URL or a Git remote from the name you sent — read `name` and `public_url` back from the create response. The unsuffixed form is kept separately as `service_name`, and `internal_url` is the **only** place it is used; every other identifier — `public_url`, the `git_push` repository path, the addon's `<ADDON_NAME>_URL` key — is built from the suffixed `name`. Reaching for `service_name` anywhere else produces a path that does not resolve. The suffix costs 6 characters and is applied *before* the 40-character limit is checked, so keep what you send to 34 characters or fewer — otherwise you get "Name is too long (maximum is 40 characters)" for a name that looked well under it. **`service_name` gets no suffix, and it must be unique across the whole workspace** — two apps seeded `api` collide even on different resources and in different projects, and the second is refused with a 422 naming the codename as already taken. Disambiguate the seed (`shop-api`, `blog-api`) rather than relying on the suffix, which protects `name` but not `service_name`.
- `label` (string) - Human-readable display name
- `project_id` (string) - UUID of the project to create the application in (get from `GET /api/v1/projects`)
- `resource_id` (string) - UUID of the compute resource (Miget) to assign (get from `GET /api/v1/resources`). The app's region is derived from this resource.
- `builder` (string) - Build strategy: `"auto"`, `"dockerfile"`, or `"custom"`. `"custom"` additionally requires `language` and `build_command` in `deployment_config` (see "Build Settings for `public_git` and `github`").

**Optional but important:**
- `ram_size` (float) - RAM allocation in MiB
- `cpu_size` (float) - CPU allocation in cores
- `deployment_method` (string) - `"git_push"`, `"public_git"`, `"github"`, `"container_registry"`, `"parent_image"`, or `"kamal"`. Note: the enum value is `container_registry` (not `docker_registry`).
- `deployment_config` (object) - Configuration specific to the chosen deployment method (see table below)
- `app_vars_attributes` (array) - Environment variables to set at creation (array of `{key, value}` objects)
- `private_access` (boolean) - Restrict the app to private access only — no public ingress, reachable only inside the workspace network (default `false`). Also accepted on `PUT /api/v1/apps/{uuid}`.

#### Deployment Configuration by Method

Each `deployment_method` requires different fields in `deployment_config`:

**`git_push`** - Deploy by pushing code to a Miget-hosted Git remote. No `deployment_config` fields are required at creation.

```json
{
  "deployment_method": "git_push",
  "deployment_config": {
    "dockerfile_path": "./Dockerfile",
    "build_context": "."
  }
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `dockerfile_path` | string | `"./Dockerfile"` | Path to Dockerfile in the repository |
| `build_context` | string | `"."` | Docker build context directory |

**Before you choose `git_push` at all.** It is the method for code with no remote. If the repository already has one, `github` or `public_git` deploys from it with no credential handling, and `github` additionally gives auto-deploy and review apps — see "Choosing Defaults". Check `git remote -v` first.

**Helping the user deploy a `git_push` app — SSH first.** There are two ways to authenticate, and they are not equally convenient. **Prefer SSH.** It uses the SSH key already on the user's account, needs no Git token, and every step is doable over the API. Fall back to HTTPS only when SSH genuinely will not work.

*Read `region.code`, `miget.name`, and `name` from `GET /api/v1/apps/{uuid}` — both remote URLs are built from them, verbatim.* **The repository path is the app's `name`, with its random suffix — `wall-pfhqv`, not `wall`.** Do not substitute `service_name` or `label`, and do not strip the suffix because it looks like noise: `service_name` is the unsuffixed form and belongs to `internal_url` only. Pushing to the unsuffixed path fails to authenticate against a repository that does not exist. The resource segment is likewise `miget.name` as returned (`migetmxq`), never the resource's display label.

**Option A — SSH (default).**

1. **Check the account for a key:** `GET /api/v1/users/me/ssh_keys`. If one is registered, there is nothing to set up — go to step 3.
2. **If the list is empty, register one.** A public key is not a secret, so you can read the user's own and send it: look for `~/.ssh/id_ed25519.pub` (or `id_rsa.pub`), and `POST /api/v1/users/me/ssh_keys` with `{"public_key": "ssh-ed25519 AAAA... user@host"}`. If they have no key pair at all, have them run `ssh-keygen -t ed25519` themselves, then read the `.pub` file. **Never read, print, or send the private key** — it is the file *without* the `.pub` suffix.
3. **Add the remote and push:**

   ```bash
   git config push.autoSetupRemote true
   # Both segments come from the API response: {miget.name} and {name}.
   git remote add miget git@ssh.{region.code}.migetapp.com:{miget.name}/{name}.git
   git push miget
   ```

   A filled-in example, so the shape is unambiguous:

   ```bash
   git remote add miget git@ssh.eu-east-1.migetapp.com:migetmxq/wall-pfhqv.git
   ```

**Option B — HTTPS with a Git token.** Git tokens are not on the API in any form — they cannot be listed, read, or created through it — so this route always ends in the dashboard. Use it only if SSH is unavailable (a network that blocks port 22, or a user who cannot add a key).

1. Send the user to `https://app.miget.com/apps/{APP_UUID}/settings#git_tokens` to reveal or create a token.
   - Default token: the **username** is the resource (miget) name, the **password** is the token value.
   - Any token the user adds: the **username** is the token's name, the **password** is the token value.
   - A token is viewable exactly once. "Token already seen" means it is gone for good and they must create a new one — revoking the default token disables HTTPS deploys entirely.
2. Add the remote and push:

   ```bash
   git config push.autoSetupRemote true
   git remote add miget https://git.{region.code}.miget.io/{miget.name}/{name}
   git push miget
   ```

   Git prompts for the username and password from step 1. Note the host differs from the SSH one: HTTPS is `git.{region}.miget.io`, SSH is `ssh.{region}.migetapp.com`. The path segments are the same suffixed `name` and `miget.name` as the SSH remote.

**For a directory that is not yet a repository**, prefix either option with `git init`, `git add .`, and `git commit -m "initial"`.

Either way, the push triggers a build and deploy — monitor it via `GET /api/v1/apps/{uuid}/deployments` (see workflow 2).

**`public_git`** - Deploy from a public Git repository URL.

```json
{
  "deployment_method": "public_git",
  "deployment_config": {
    "credential_id": "{git-credential-uuid}",
    "repository_url": "https://github.com/user/repo.git",
    "branch": "main",
    "dockerfile_path": "./Dockerfile",
    "build_context": "."
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `repository_url` | string | Yes | - | Full HTTPS Git repository URL (note: `public_git` uses `repository_url`, whereas `github` uses `repository`) |
| `branch` | string | No | `"main"` | Branch to deploy from |
| `credential_id` | string | No | - | UUID of Git credentials (for private repos) |
| `dockerfile_path` | string | No | `"./Dockerfile"` | Path to Dockerfile |
| `build_context` | string | No | `"."` | Docker build context directory |
| `project_path` | string | No | `""` | Subdirectory to build from, for monorepos |
| `run_command` | string | No | - | Override the start command (builder `auto` or `custom`) |
| `language` | string | No | - | Language to build with — **required** when `builder` is `custom` |
| `build_command` | string | No | - | Build command — **required** when `builder` is `custom` |
| `pre_deploy_command` | string | No | - | Command run once before the new release starts (e.g. database migrations) |
| `post_deploy_command` | string | No | - | Command run once after a successful deploy |
| `use_dhi` | boolean | No | `false` | Build on Docker Hardened Images; ignored when `builder` is `dockerfile` |

**Validate the URL before you send it.** The platform enforces the format below and rejects a malformed or unreachable repo at creation — check it yourself first so you can correct the user instead of surfacing a 422:
- Must be an **HTTPS** URL shaped `https://<host>/<owner>/<repo>` (the trailing `.git` is optional — it is normalized server-side). Examples: `https://github.com/rails/rails`, `https://gitlab.com/group/project.git`.
- **Rejected:** SSH URLs (`git@github.com:owner/repo.git`), plain `http://`, a host with a port, and extra path depth such as GitLab subgroups (`host/group/subgroup/repo`) — the check expects exactly host + owner + repo. If the user gives an SSH or browser URL, convert it to the `https://<host>/<owner>/<repo>` form before sending.
- The repo **and** the `branch` must be publicly reachable: on create the platform verifies the repository is accessible and the branch exists. A private repository without a `credential_id`, or a non-existent branch, fails validation. For a private repo, pass a `credential_id` (see Git Credentials).

Three failures come back from that reachability check, and they mean different things — do not treat them all as a bad URL:

| Message | What it means | What to do |
|---|---|---|
| `The repository or branch does not exist.` | The repo is private, the URL is wrong, or the branch name is wrong | Fix the URL or branch, or pass a `credential_id` |
| `The repository host rate limit was exceeded. Please try again later.` | The Git host is throttling the platform — **the URL is fine** | Wait and retry; do not "correct" a URL that was already right |
| `The branch '<name>' has no commits.` | The branch exists but is empty | Push a commit, or point `branch` at one that has history |

**`github`** - Deploy from a GitHub repository using the Miget GitHub App integration.

```json
{
  "deployment_method": "github",
  "deployment_config": {
    "credential_id": "{github-credential-uuid}",
    "repository": "username/repo",
    "branch": "main",
    "auto_deploy_enabled": true,
    "auto_deploy_branch": "main",
    "dockerfile_path": "./Dockerfile",
    "build_context": "."
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `credential_id` | string | Yes | - | UUID of GitHub App credentials (from workspace credentials) |
| `repository` | string | Yes | - | GitHub repository in `owner/repo` format |
| `branch` | string | No | `"main"` | Branch to deploy from |
| `auto_deploy_enabled` | boolean | No | `false` | Automatically deploy when code is pushed |
| `auto_deploy_branch` | string | No | same as `branch` | Branch that triggers auto-deploy |
| `dockerfile_path` | string | No | `"./Dockerfile"` | Path to Dockerfile |
| `build_context` | string | No | `"."` | Docker build context directory |
| `project_path` | string | No | `""` | Subdirectory to build from, for monorepos |
| `run_command` | string | No | - | Override the start command (builder `auto` or `custom`) |
| `language` | string | No | - | Language to build with — **required** when `builder` is `custom` |
| `build_command` | string | No | - | Build command — **required** when `builder` is `custom` |
| `pre_deploy_command` | string | No | - | Command run once before the new release starts (e.g. database migrations) |
| `post_deploy_command` | string | No | - | Command run once after a successful deploy |
| `use_dhi` | boolean | No | `false` | Build on Docker Hardened Images; ignored when `builder` is `dockerfile` |

#### Build Settings for `public_git` and `github`

The two Git-based methods share a set of build fields. They are also accepted on `PUT /api/v1/apps/{uuid}/deployment` under `deployment_config_attributes`, where they behave as a patch — see **Adding a build setting to an existing app** below.

**Adding a build setting to an existing app.** Send the app's current `deployment_method` plus only the fields you want to change; everything you leave out is preserved. To add migrations to a live app you do not have to restate its branch, project path, or commands:

```json
{
  "deployment_method": "public_git",
  "deployment_config_attributes": {
    "repository_url": "https://github.com/user/repo.git",
    "pre_deploy_command": "bin/rails db:migrate"
  }
}
```

To clear a field, send it as an empty string (`"pre_deploy_command": ""`). Note that `repository_url` (for `public_git`) and `repository` + `credential_id` (for `github`) must be present on every request even when unchanged.

**Running database migrations.** Miget has no implicit release phase — nothing runs between the build finishing and the new replicas starting. Put migrations in `pre_deploy_command` so they run **once** before the new release goes live, rather than in the start command where every replica would run them on boot:

```json
{
  "deployment_method": "public_git",
  "deployment_config": {
    "repository_url": "https://github.com/user/repo.git",
    "branch": "main",
    "pre_deploy_command": "npx drizzle-kit migrate"
  }
}
```

Typical values: `npx drizzle-kit migrate` (Drizzle), `alembic upgrade head` (Alembic), `bin/rails db:migrate` (Rails), `python manage.py migrate` (Django). Prisma is the exception — see below. The other Git-based deployment methods have no equivalent field — for those, migrations have to run from the start command or a cronjob.

**What the release phase can and cannot do.** `pre_deploy_command` runs in the runtime image as the unprivileged user `node`, with no package manager and no root. Nothing can be installed from it — `apt-get install …` fails with `Permission denied`. Write the command against what the image already ships.

**Prisma specifically.** The default `auto` runtime image (`node:22.16.0-slim`) has no OpenSSL, which Prisma's migration engine requires. `npx prisma migrate deploy` reaches the database and then fails with `prisma:warn Prisma failed to detect the libssl/openssl version` followed by an empty `Error: Migration engine error:` and `Release failed`. There is no `pre_deploy_command` workaround (see above). For Prisma, use `builder: "dockerfile"` with a base image that includes OpenSSL, or run migrations from outside the platform.

**`NODE_ENV` is `production` during the build.** The generated runtime Dockerfile exports it before `build_command` runs, so a plain `npm ci` skips `devDependencies` — any build needing a bundler or compiler then dies with a module-resolution error such as `Could not find Nx modules`. Use `npm ci --include=dev && npm run build` when the build needs dev dependencies.

**Using `builder: "custom"`.** The `custom` builder needs `language` **and** `build_command` in `deployment_config`; without them the build has nothing to run, and the request is rejected with `422`. `run_command` is optional but usually wanted, since `custom` does not infer a start command:

```json
{
  "builder": "custom",
  "deployment_method": "github",
  "deployment_config": {
    "credential_id": "{github-credential-uuid}",
    "repository": "user/repo",
    "language": "nodejs",
    "build_command": "npm run build",
    "run_command": "node server.js"
  }
}
```

**Monorepos.** Set `project_path` to the subdirectory holding the app (for example `apps/api`). The build then treats that directory as its root.

**`container_registry`** - Deploy a pre-built container image from a registry (Docker Hub, GHCR, etc.).

```json
{
  "deployment_method": "container_registry",
  "deployment_config": {
    "credential_id": "{registry-credential-uuid}",
    "image_url": "docker.io/library/nginx",
    "tag": "latest",
    "command": ["/opt/keycloak/bin/kc.sh"],
    "args": ["start", "--optimized"]
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `image_url` | string | Yes | - | Container image reference **without scheme and without tag** (e.g., `docker.io/library/nginx`) |
| `tag` | string | No | `"latest"` | Container image tag (separate field — do not append it to `image_url`) |
| `credential_id` | string | No | - | UUID of container registry credentials (for private registries) |
| `command` | array of strings | No | - | Override the image's `ENTRYPOINT` (Kubernetes `container.command`). Leave unset to use the image default. |
| `args` | array of strings | No | - | Override the image's `CMD` (Kubernetes `container.args`). Leave unset to use the image default. Use this when an image's ENTRYPOINT is set but no CMD is supplied (e.g. Keycloak prints help on bare run; pass `["start"]` to start the server). |

**Validate the image reference before you send it.** The platform enforces the format below and rejects a malformed reference at creation — check it yourself first:
- Provide `image_url` **without a scheme and without a tag**. Format: `[registry-host[:port]/]namespace/name`, with at least one `/`. Examples: `docker.io/library/nginx`, `ghcr.io/org/app`, `registry.example.com:5000/team/app`.
- **Rejected:** a bare name with no namespace (`nginx` → use `library/nginx`), anything containing `://`, and an image with the tag embedded.
- **Split off the tag.** If the user gives `ghcr.io/org/app:1.2`, send `image_url: "ghcr.io/org/app"` and `tag: "1.2"`. If no tag is given, it defaults to `latest`.
- **Private images** require a matching `credential_id` (a container registry credential — see Container Registry Credentials). Public images (e.g. Docker Hub official images) need none.

**`parent_image`** - Inherit the container image from another app on the platform. When the parent app deploys, this app can auto-sync.

```json
{
  "deployment_method": "parent_image",
  "deployment_config": {
    "parent_app_id": "{parent-app-uuid}",
    "parent_image_auto_sync": true
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `parent_app_id` | string | Yes | - | UUID of the parent application to inherit image from |
| `parent_image_auto_sync` | boolean | No | `false` | Automatically deploy when parent image updates |

**`kamal`** - Deploy from the user's local machine using the Kamal CLI (`kamal deploy`). Miget provides SSH infrastructure and a container registry; the user runs Kamal locally.

```json
{
  "deployment_method": "kamal",
  "deployment_config": {
    "ssh_keys": ["ssh-ed25519 AAAA... user@machine"]
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `ssh_keys` | array of strings | No | `[]` | SSH public keys for Kamal deployment access |

**Kamal-specific requirements:**
- The `builder` should be `"dockerfile"` (Kamal builds images locally using Docker).
- The **registry password** is auto-generated by the platform. After creation, retrieve it via `GET /api/v1/apps/{uuid}` - the response includes the deployment config with the password. The user must set this password as the `MIGET_REGISTRY_PASSWORD` environment variable on their local machine for `kamal deploy` to authenticate with the registry.
- After creation, the app's deployment config will also contain registry hostname, username, image path, and SSH endpoint info needed for the user's `config/deploy.yml`. Retrieve it via `GET /api/v1/apps/{uuid}`.
- The deploy button / `POST /api/v1/apps/{uuid}/deploy` is **not used** for Kamal apps - the user deploys from their machine.
- You **cannot switch** an existing app to Kamal. The app must be created with `deployment_method: "kamal"` from the start.

**Example interaction:**
```
User: "Create an app for me"

Agent: "I'll help you create an application. I need some information first:

Required:
1. Application name (lowercase, alphanumeric with hyphens, e.g., 'my-api-server')
2. Display label (human-readable name, e.g., 'My API Server')
3. Project (which project should this app belong to?)
4. Region (which region should I deploy to?)
5. Builder type (auto/dockerfile/custom)

Optional but recommended:
- Resource (which Miget resource should I assign?)
- Deployment method (git_push/public_git/github/container_registry/parent_image/kamal). Note: the enum value is `container_registry` (not `docker_registry`).

Please provide these details so I can create the application."
```

---

### Create Stack (`POST /api/v1/stacks`)

A stack deploys a multi-service app from a `docker-compose.yml` in a Git repository. The compose source is analyzed server-side, so **call `POST /api/v1/stacks/analyze` first** to discover the services and which environment variables are required.

**Required fields:**
- `repository_url` (string) - Git repository URL (GitHub or public HTTPS Git)
- `branch` (string) - Git branch to deploy
- `resource_id` (string) - UUID of the compute resource (Miget) to deploy onto
- One of `project_id` (string, existing project UUID) **or** `new_project_name` (string, creates a new project)

**Optional:**
- `compose_path` (string) - Path to the compose file in the repo (default `"."`)
- `credential_id` (string) - UUID of a stored Git credential (for private repositories)
- `label` (string) - Display name; `name` (string) - codename seed (derived from the repo if omitted)
- `new_project_description` (string) - description when creating a new project
- `env_var_overrides` (object) - Values for required env vars, shaped `{ "<service>": { "<KEY>": "<value>" } }`
- `auto_populate_required_vars` (boolean) - Fill any required env var left without a custom value (default `false`)

**Discover-then-supply flow (handling env vars):**
1. `POST /api/v1/stacks/analyze` with `{repository_url, branch, compose_path?}`. Each app/standalone service in the returned `manifest` has `env_vars: [{ key, value, required }]`.
2. Required vars are those with `required: true` and a blank `value`. Ask the user whether each should be a **custom** value or **auto-populated** (good for secrets).
3. `POST /api/v1/stacks` with custom values in `env_var_overrides` and/or `auto_populate_required_vars: true`. Managed services (databases/caches) are auto-configured — never supply env vars for them.

**Derived variables:** some stacks declare a variable that must be computed from another rather than
chosen freely — for example a Supabase `ANON_KEY` is a JWT signed with that stack's `JWT_SECRET`. The
platform always computes those from their source, so a value you send for one in `env_var_overrides`
is ignored. Supply the source variable (or let `auto_populate_required_vars` generate it) and the
dependent values are derived to match.

**Error responses to handle:**
- `422 { "error": "Missing required environment variables: web.SECRET, ..." }` - a required var was neither supplied nor auto-populated; go back to step 2.
- `422 { "error": "Not enough capacity on the resource: ..." }` - the manifest needs more RAM/disk than the resource has; pick a larger `resource_id`.

---

## Creating Addons and Services

When a user asks to create a database, cache, or storage, you must first clarify *how* they want to create it. There are two primary methods:

1.  **App Addon (`POST /api/v1/apps/{uuid}/addons`)**: An addon is attached directly to a *specific, existing application*. The addon's lifecycle is tied to the app. Connection details are automatically injected as environment variables into that app.
2.  **Standalone Service (`POST /api/v1/services`)**: A service is a standalone resource that can be shared across *multiple applications* within a project. It has its own lifecycle and is not tied to any single app.

Ask which method they prefer - creating the wrong one wastes time and may require starting over.

**Example Interaction:**

> **User:** "I need a PostgreSQL database."
>
> **Agent:** "I can create a PostgreSQL database in two ways. Which do you prefer?
>
> 1.  **App Addon**: Attach it to a specific, existing app. Connection details will be injected as env vars automatically.
> 2.  **Standalone Service**: Create it as a shared resource for multiple apps in a project.
>
> If you choose 'App Addon', please provide the UUID of the application it should be attached to."

Once the user chooses, proceed to collect the required parameters for the chosen method.

For PostgreSQL databases, also ask about:
- **Creation mode**: Fresh database (default) or external replica (replicating from an existing external PostgreSQL).
- **Deployment type**: Standalone instance (default, 1 instance) or High Availability cluster (3, 5, or 7 instances with automatic failover).

---

### Create App Addon (`POST /api/v1/apps/{uuid}/addons`)

An addon is attached to a specific application and its lifecycle is managed alongside the app.

**Common Parameters (for all addon types):**

*   **Required:**
    *   `uuid` (string, in path): The UUID of the application to attach the addon to.
    *   `type` (string): The type of addon. Must be one of `postgres`, `mysql`, `valkey`, `storage`.
    *   `label` (string): A human-readable display name for the addon.
*   **Optional:**
    *   `ram_size` (float): RAM allocation in MiB (e.g., 64, 128, 256).
    *   `disk_size` (float): Disk storage in GiB (e.g., 1, 5, 10).
    *   `cpu_size` (float): CPU allocation in cores (e.g., 0.1, 0.25, 0.5). A ceiling, not a reservation — and ignored on dev plans, where it is pinned to `0.1`.

**Connection variables.** Creating a `postgres`, `mysql`, or `valkey` addon writes **two** variables to the app, both set to the same connection string: `<ADDON_NAME>_URL`, and a generic alias — `DATABASE_URL` for `postgres`/`mysql`, `REDIS_URL` for `valkey`. The alias is skipped when the app already has a variable of that name (case-insensitive), and an existing one is never overwritten. Verify with `GET /api/v1/apps/{uuid}/vars` after creating the addon.

---

#### Addon Type: `postgres`

A PostgreSQL database addon. Supports two creation modes: fresh database or external replica.

*   **Type-specific Parameters:**
    *   `postgres_version` (string, **required**): The major version of PostgreSQL. Accepted values: `'18'`, `'17'`, `'16'`, `'15'`, `'14'`, `'13'`. Any other value is rejected with `400`.
    *   `public_access` (string): Enable public internet access. Use `'1'` for enabled, `'0'` for disabled.
    *   `instances` (integer): Number of database instances. Allowed values: `1` (standalone, default), `3`, `5`, or `7` for a High Availability cluster.
    *   `creation_mode` (string): `'fresh'` (new empty database, default) or `'external_replica'` (replica of an external PostgreSQL database).

*   **External Replica Parameters** (required when `creation_mode` is `'external_replica'`):
    *   `external_host` (string): Hostname of the external PostgreSQL source database.
    *   `external_port` (integer): Port of the external PostgreSQL source (default: 5432).
    *   `auth_type` (string): `'password'` or `'tls'`.
    *   `replication_username` (string): Username for replication connection.
    *   `replication_password` (string): Password (required when `auth_type` is `'password'`).
    *   `ca_crt` (string): CA certificate (required when `auth_type` is `'tls'`).
    *   `tls_crt` (string): TLS client certificate (required when `auth_type` is `'tls'`).
    *   `tls_key` (string): TLS client key (required when `auth_type` is `'tls'`).
    *   `s3_enabled` (string): `'1'` to enable optional S3 WAL archive fallback.
    *   `s3_endpoint`, `s3_bucket`, `s3_path`, `s3_access_key`, `s3_secret_key` (strings): S3 configuration (required when `s3_enabled` is `'1'`).

**Example questions:**

> "What version of PostgreSQL would you like? (18, 17, 16, 15, 14 or 13)"
> "Should this database be accessible from the public internet? (yes/no)"
> "Do you want a standalone instance or a High Availability ha_cluster? (standalone/cluster)"
> "Do you want to create a fresh database or replicate from an external PostgreSQL? (fresh/external_replica)"

---

#### Addon Type: `mysql`

A MySQL database addon.

*   **Type-specific Parameters:**
    *   `mysql_version` (string, **required**): The major version of MySQL. Accepted values: `'8.2'`, `'8.0'`. Any other value is rejected with `400`.

**Example questions:**

> "What version of MySQL would you like? (8.2 or 8.0)"

---

#### Addon Type: `valkey`

A Valkey (Redis-compatible) cache addon.

*   **Type-specific Parameters:**
    *   `valkey_version` (string, **required**): The version of Valkey. Accepted values: `'7'`, `'7.2'`. Any other value is rejected with `400`.

**Example questions:**

> "What version of Valkey would you like? (7 or 7.2)"

---

#### Addon Type: `storage`

A persistent storage volume addon.

*   **Type-specific Parameters:**
    *   `service_id` (integer): To attach an existing shared storage service, provide its ID. When given, `mount_point` and `storage_access` are inherited from that service — omit them.
    *   `mount_point` (string, **required unless `service_id` is given**): The path inside the container where the volume should be mounted (e.g., `/data`).
    *   `storage_access` (string, **required unless `service_id` is given**): The access mode. Must be one of `RWO` (ReadWriteOnce) or `RWX` (ReadWriteMany).
    *   `sub_path` (string, optional): Mount only this subdirectory of the volume instead of its root — a relative path like `media/uploads`, created automatically if missing. Only valid for `RWX` storage and shared-storage mounts; sending it with `RWO` is refused with a `422`. Each app mounting the same shared volume can use a different `sub_path`, which is how several apps share one volume without seeing each other's files.

**Example questions:**

> "Where should the storage be mounted inside the container? (e.g., /data)"
> "What access mode do you need? `RWO` (for a single running app instance) or `RWX` (for multiple app instances)?"

---

### Create Service (`POST /api/v1/services`)

A service is a standalone resource (e.g., database, shared storage) that can be used by multiple applications.

**Common Parameters (for all service types):**

*   **Required:**
    *   `service_type` (string): The type of service. Must be one of `postgres`, `shared_storage`.
    *   `project_id` (string): The UUID of the project this service will belong to.
    *   `label` (string): A human-readable display name for the service.
*   **Optional:**
    *   `resource_id` (string): UUID of the compute resource to provision the service on. Optional at the API level, but a service needs a resource to run on — supply this (or the legacy alias `miget_id`, deprecated) in practice.
    *   `ram_size` (float): RAM allocation in MiB.
    *   `disk_size` (float): Disk storage in GiB.
    *   `cpu_size` (float): CPU allocation in cores.

---

#### Service Type: `postgres`

A standalone PostgreSQL database service. Supports two creation modes: fresh database or external replica.

*   **Type-specific Parameters:**
    *   `postgres_version` (string, **required**): The major version of PostgreSQL. Accepted values: `'18'`, `'17'`, `'16'`, `'15'`, `'14'`, `'13'`. Any other value is rejected with `400`.
    *   `public_access` (string): Enable public internet access. Use `'1'` for enabled, `'0'` for disabled.
    *   `environment_variables` (boolean): If `true`, writes the connection variables to the **project** the service belongs to — `<SERVICE_NAME>_URL` and `DATABASE_URL` — so every app in that project inherits them. An existing project variable of the same name is not overwritten.
    *   `instances` (integer): Number of database instances. Allowed values: `1` (standalone, default), `3`, `5`, or `7` for a High Availability cluster.
    *   `creation_mode` (string): `'fresh'` (new empty database, default) or `'external_replica'` (replica of an external PostgreSQL database).

*   **External Replica Parameters** (required when `creation_mode` is `'external_replica'`):
    *   `external_host` (string): Hostname of the external PostgreSQL source database.
    *   `external_port` (integer): Port of the external PostgreSQL source (default: 5432).
    *   `auth_type` (string): `'password'` or `'tls'`.
    *   `replication_username` (string): Username for replication connection.
    *   `replication_password` (string): Password (required when `auth_type` is `'password'`).
    *   `ca_crt` (string): CA certificate (required when `auth_type` is `'tls'`).
    *   `tls_crt` (string): TLS client certificate (required when `auth_type` is `'tls'`).
    *   `tls_key` (string): TLS client key (required when `auth_type` is `'tls'`).
    *   `s3_enabled` (string): `'1'` to enable optional S3 WAL archive fallback.
    *   `s3_endpoint`, `s3_bucket`, `s3_path`, `s3_access_key`, `s3_secret_key` (strings): S3 configuration (required when `s3_enabled` is `'1'`).

**Example questions:**

> "Which project should this service belong to? (Please provide the Project UUID)"
> "Which compute resource (Miget) should I provision this on? (Please provide the Miget UUID)"
> "What version of PostgreSQL would you like? (18, 17, 16, 15, 14 or 13)"
> "Should this database be publicly accessible? (yes/no)"
> "Do you want a standalone instance or a High Availability ha_cluster? (standalone/cluster)"
> "Do you want to create a fresh database or replicate from an external PostgreSQL? (fresh/external_replica)"

---

#### Service Type: `shared_storage`

A standalone shared storage volume service.

*   **Type-specific Parameters:**
    *   `mount_point` (string, **required**): The default mount path (e.g., `/shared-data`). Apps attaching to this service inherit it.
    *   `storage_access` (string): Ignored — a shared storage service is always provisioned as `RWX`.

**Example questions:**

> "Which project should this service belong to? (Please provide the Project UUID)"
> "Which compute resource (Miget) should I provision this on? (Please provide the Miget UUID)"
> "What access mode do you need? `RWO` (for a single app) or `RWX` (for multiple apps)?"

---

### Create Bucket (`POST /api/v1/buckets`)

**Required fields:**
- `label` (string) - Human-readable display name for the bucket

**Optional but important:**
- `resource_id` (string) - UUID of the compute resource to attach the bucket to (get from `GET /api/v1/resources`). Optional at the API level, but a bucket needs a resource — supply this (or the legacy alias `miget_id`, deprecated) in practice.
- `project_id` (string) - UUID of the project the bucket belongs to (get from `GET /api/v1/projects`). Never inferred: omit it and the bucket is created with no project, even in a workspace with a single project. Send `null` on `PUT /api/v1/buckets/{uuid}` to unassign an existing bucket.
- `visibility` (string) - Bucket visibility: `"public_access"` or `"private_access"` (default: `"private_access"`)
- `disk_size` (float) - Disk allocation in GiB (default: 0.1)

**Ask only what you cannot derive:**
- "What should be the bucket's display name?"
- "Which resource (Miget) should the bucket be attached to? (provide resource ID)"
- "Which project should the bucket belong to, or should it stay outside any project?"
- "Should the bucket be public or private? (default: private)"
- "How much storage do you need in GiB? (default: 0.1 GiB)"

---

### Bucket Policy & ACL (AI-Assisted Configuration)

Users rarely know S3 policy or ACL formats. Your role is to understand what they want in plain language, build the correct configuration, and send it via the API. Policies use JSON format; ACLs use XML format. Asking a user to write raw JSON/XML creates friction - instead, ask about their intent and construct the configuration yourself.

#### Step-by-step: How to handle a bucket access request

1. **Understand intent** - Ask what the user wants to achieve:
   - "Who should have access?" (everyone, specific IPs, specific users)
   - "What kind of access?" (read-only, read-write, full control)
   - "To what?" (all objects, a specific path/prefix)

2. **Choose the right mechanism:**
   - Recommend **policy** for most cases (broad rules, IP restrictions, public access)
   - Recommend **ACL** only when the user needs per-user/per-group granular S3 permissions

3. **Fetch the bucket first** - Call `GET /api/v1/buckets/{uuid}` to get the bucket `name` (needed for policy ARNs) and to check the current `policy`/`acl` state

4. **Build the configuration** - Construct JSON (for policies) or XML (for ACLs) from the templates below, substituting the actual bucket name and user-provided values

5. **Show the user what you built** - Display the formatted configuration and explain what it does before sending

6. **Send it** - `PUT /api/v1/buckets/{uuid}/policy` or `PUT /api/v1/buckets/{uuid}/acl`

#### Update Bucket Policy (`PUT /api/v1/buckets/{uuid}/policy`)

**Required fields:**
- `uuid` (string, path) - Bucket UUID
- `policy` (string) - S3-compatible bucket policy as a JSON string

**Constraints:** Must be valid JSON. Requires `buckets:operate` permission.

**Policy templates** (replace `{bucket-name}` with the actual bucket name from the GET response):

- **Public read access to all objects:**
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": "*",
        "Action": ["s3:GetObject"],
        "Resource": ["arn:aws:s3:::{bucket-name}/*"]
      }
    ]
  }
  ```

- **Restrict access to specific IP range:**
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:*",
        "Resource": ["arn:aws:s3:::{bucket-name}", "arn:aws:s3:::{bucket-name}/*"],
        "Condition": {
          "NotIpAddress": {
            "aws:SourceIp": "203.0.113.0/24"
          }
        }
      }
    ]
  }
  ```

- **Read-only access for a specific user:**
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam:::{account-id}:user/{username}"},
        "Action": ["s3:GetObject", "s3:ListBucket"],
        "Resource": ["arn:aws:s3:::{bucket-name}", "arn:aws:s3:::{bucket-name}/*"]
      }
    ]
  }
  ```

- **Allow uploads to a specific prefix:**
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": "*",
        "Action": ["s3:PutObject"],
        "Resource": ["arn:aws:s3:::{bucket-name}/uploads/*"]
      }
    ]
  }
  ```

You can combine multiple statements in one policy. For non-standard requests, build the JSON using S3 policy conventions - `Version` is always `"2012-10-17"`, each statement needs `Effect`, `Principal`, `Action`, and `Resource`.

**Example interactions:**

```
User: "Make my bucket publicly readable"

Agent: "I'll create a policy that allows anyone to read objects from your
bucket. Let me fetch the bucket details first."
-> GET /api/v1/buckets/{uuid}  (gets bucket name: "my-assets-abc123")
-> Builds policy JSON with Resource: "arn:aws:s3:::my-assets-abc123/*"
-> Shows the user the formatted policy and explains it
-> PUT /api/v1/buckets/{uuid}/policy
```

```
User: "Only allow access from our office network 10.0.0.0/8"

Agent: "I'll set up a policy that denies all access except from your
10.0.0.0/8 network range."
-> Fetches bucket, builds deny policy with NotIpAddress condition
-> Shows and confirms, then sends PUT
```

```
User: "Remove the bucket policy"

Agent: -> DELETE /api/v1/buckets/{uuid}/policy
"Done - the bucket policy has been removed."
```

#### Update Bucket ACL (`PUT /api/v1/buckets/{uuid}/acl`)

**Required fields:**
- `uuid` (string, path) - Bucket UUID
- `acl` (string) - S3-compatible ACL as an XML string

**Constraints:** Must be valid XML. Requires `buckets:operate` permission.

**Available ACL permissions:** `READ`, `WRITE`, `READ_ACP`, `WRITE_ACP`, `FULL_CONTROL`

**ACL templates:**

- **Public read access:**
  ```xml
  <AccessControlPolicy>
    <Owner>
      <ID>{owner-canonical-id}</ID>
    </Owner>
    <AccessControlList>
      <Grant>
        <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="Group">
          <URI>http://acs.amazonaws.com/groups/global/AllUsers</URI>
        </Grantee>
        <Permission>READ</Permission>
      </Grant>
    </AccessControlList>
  </AccessControlPolicy>
  ```

- **Grant full control to another user:**
  ```xml
  <AccessControlPolicy>
    <Owner>
      <ID>{owner-canonical-id}</ID>
    </Owner>
    <AccessControlList>
      <Grant>
        <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="CanonicalUser">
          <ID>{grantee-canonical-id}</ID>
        </Grantee>
        <Permission>FULL_CONTROL</Permission>
      </Grant>
    </AccessControlList>
  </AccessControlPolicy>
  ```

**Example interactions:**

```
User: "Grant read access to everyone on my assets bucket"

Agent: "I'll set an ACL granting public read access. Let me fetch the
bucket details to get the owner ID."
-> GET /api/v1/buckets/{uuid}
-> Builds ACL XML with owner ID from response
-> Shows user the formatted ACL, explains what it does
-> PUT /api/v1/buckets/{uuid}/acl
```

```
User: "I want to set up an ACL for my bucket"

Agent: "I can help with that. What access do you need to configure?
For example:
- Grant public read access to everyone
- Grant a specific user read or write access
- Grant full control to another S3 user

What would you like to set up?"
```

---

### Create Resource (`POST /api/v1/resources`)

**Required fields:**
- `plan_code_name` (string) - Plan identifier. **Do not guess it** — call `GET /api/v1/plans` and use a `code_name` from the response verbatim. Code names are opaque identifiers (e.g. `miget_hobby_0`, `miget_pro_1`), not friendly words like `"starter"` or `"professional"`, and they differ between environments. A wrong value returns a generic 422.
- `region_code` (string) - Deployment region code. Available: `eu-east-1` (Warsaw), `us-east-1` (Vint Hill)

**Optional:**
- `plan_type` (string) - Ignored. The plan is resolved from `plan_code_name` alone; this field is accepted only for backward compatibility. Read `plan_type` off the plan object instead to tell `dev` and `pro` plans apart.
- `components` (array) - Additional resource components (extra RAM, CPU, disk)

**Ask only what you cannot derive:**
- "What plan type? (dev for development, pro for production)"
- "What plan code name? (e.g., free, starter, professional)"
- "Which region? (eu-east-1, us-east-1)"
- "Do you want to add any extra components? (RAM, CPU, disk)"

### Create Project (`POST /api/v1/projects`)

**Required fields:**
- `name` (string) - Project name (must be unique within workspace)

**Optional:**
- `description` (string) - Brief description of the project's purpose

**Ask only what you cannot derive:**
- "What should be the project name?"
- "Do you want to add a description?"

### Create App Domain (`POST /api/v1/apps/{uuid}/domains`)

**Required fields:**
- `domain.name` (string) - Fully qualified domain name (e.g., `"app.example.com"`)

**Ask only what you cannot derive:**
- "What domain name should I add? (e.g., app.example.com)"

### Create App Environment Variable (`POST /api/v1/apps/{uuid}/vars`)

**Required fields:**
- `key` (string) - Variable name (use SCREAMING_SNAKE_CASE, e.g., `DATABASE_URL`)
- `value` (string) - Variable value

**Ask only what you cannot derive:**
- "What's the variable name? (use SCREAMING_SNAKE_CASE)"
- "What's the variable value?"

### Create App Cronjob (`POST /api/v1/apps/{uuid}/cronjobs`)

**Required fields:**
- `name` (string) - Unique cron job identifier

**Optional but important:**
- `label` (string) - Human-readable display name
- `schedule_type` (string) - `"cron"` for custom cron expression, `"interval"` for predefined intervals
- `interval_type` (string) - `"every_10_minutes"`, `"hourly"`, or `"daily"` (for interval type)
- `cron` (string) - Cron expression (e.g., `"0 * * * *"` for hourly, for cron type)
- `command` (string) - Shell command to execute
- `daily_time` (string) - Execution time for daily jobs in HH:MM format (24-hour)
- `minute` (string) - Minute component for scheduling (0-59)
- `hour` (string) - Hour component for scheduling (0-23)

**Important notes:**
- Updating a cronjob (`PUT`) only changes `label` and `command`. The **schedule cannot be changed in place** — to reschedule, DELETE the cronjob and create a new one.
- Per-run logs are available via `GET /api/v1/apps/{uuid}/cronjobs/{id}/stream_logs` (SSE) once the job has run at least once.

**Ask only what you cannot derive:**
- "What should be the cronjob name?"
- "What display label should I use?"
- "What schedule type? (cron for custom expression, interval for predefined)"
- "If interval, which interval? (every_10_minutes/hourly/daily)"
- "If cron, what cron expression? (e.g., 0 * * * * for hourly)"
- "If daily, what time? (HH:MM format)"
- "What command should be executed?"

### Create App Port (`POST /api/v1/apps/{uuid}/ports`)

**Required fields:**
- `internal_port` (integer) - Internal port number (1-65535)
- `protocol` (string) - Protocol: `"tcp"` or `"udp"`

**Optional but important:**
- `public` (boolean) - Whether the port should be publicly accessible (default: `false`)

**Important notes:**
- Requires `apps:manage` permission
- Port management is not available on free plan resources
- Ports can be exposed publicly or made private after creation using separate endpoints
- Port `5000` is fixed for HTTP traffic on the app's `*.migetapp.com` URL — it is auto-created, cannot be removed or changed, and the app must listen on it. Use this endpoint to add extra TCP/UDP ports for custom protocols; they are **private by default** — use `expose_publicly` to make them reachable from outside the cluster. See https://docs.miget.com/networking/ports for the full list of supported ports.

**Ask only what you cannot derive:**
- "What internal port number? (1-65535)"
- "What protocol? (tcp or udp)"
- "Should this port be publicly accessible? (true/false)"

### Create Container Registry Credential (`POST /api/v1/container_registry_credentials`)

Stores credentials for pulling images from a private registry. The returned `uuid` is passed as `deployment_config.credential_id` when creating an app with `deployment_method: container_registry` (and is also usable for `github`/`public_git` private repositories). The `token` is encrypted at rest and never returned.

**Required fields:**
- `name` (string) - Display name (must be unique within the workspace)
- `registry` (string) - Provider: `docker_hub`, `github`, `gitlab`, `aws_ecr`, `azure`, `digitalocean`, `quay`, or `generic`
- `username` (string) - Registry username
- `token` (string) - Registry password or access token

**Optional:**
- `registry_hostname` (string) - Registry hostname URL (required for `generic`, `aws_ecr`, and `azure`; optional otherwise)
- `skip_validation` (boolean) - Skip live credential validation (non-production environments only)

**Response fields:** `uuid`, `name`, `registry`, `username`, `registry_hostname`, `created_at`, `updated_at` (the `token` is never returned).

**Ask only what you cannot derive:**
- "Which registry provider? (docker_hub, github, gitlab, aws_ecr, azure, digitalocean, quay, generic)"
- "What's the registry username and access token/password?"
- "For generic/aws_ecr/azure registries: what's the registry hostname?"

### Rollback Deployment (`POST /api/v1/apps/{uuid}/deployments/{id}/rollback`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Deployment UUID

**Constraints:**
- Deployment must be rollbackable (has image URL, same deployment method as app, not running/failed)
- Application must be in deployable state
- Requires `apps:deploy` permission

**Ask only what you cannot derive:**
- "Which deployment should I rollback to? (provide deployment UUID)"

### Rotate Addon Password (`POST /api/v1/apps/{uuid}/addons/{id}/rotate_password`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Addon UUID

**Constraints:**
- Password rotation only supported for certain addon types (databases like PostgreSQL, MySQL, and caches like Valkey)
- Requires `apps:operate` permission
- New password will be returned in response - you must update ENV variables manually

**Ask only what you cannot derive:**
- "Which addon should I rotate the password for? (provide addon UUID)"

### Create Read Replica - App Addon (`POST /api/v1/apps/{uuid}/addons/{id}/create_replica`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Addon ID or UUID (must be a PostgreSQL primary addon)

**Optional fields:**
- `cpu_size` (float) - CPU allocation for the replica (defaults to primary's value)
- `ram_size` (integer) - RAM allocation in megabytes (defaults to primary's value)

**Constraints:**
- Addon must be PostgreSQL type
- Addon must be a primary (not a replica itself)
- Resource must have sufficient capacity for the replica
- Requires `apps:operate` permission

**Ask only what you cannot derive:**
- "Which PostgreSQL addon should I create a replica for? (provide addon UUID)"
- "What CPU and RAM should the replica use? (defaults to primary's values)"

### Promote Replica to Standalone - App Addon (`POST /api/v1/apps/{uuid}/addons/{id}/promote_replica`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Addon ID or UUID (must be a PostgreSQL replica addon)

**Constraints:**
- Addon must be PostgreSQL type
- Addon must be a replica (not a primary)
- Requires `apps:operate` permission

**Ask only what you cannot derive:**
- "Which replica should I promote to a standalone instance? (provide addon UUID)"

### Promote External Replica - App Addon (`POST /api/v1/apps/{uuid}/addons/{id}/promote_external`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Addon ID or UUID (must be a PostgreSQL external replica addon)

**Constraints:**
- Addon must be PostgreSQL type
- Addon must be an external replica (created with `creation_mode: external_replica`)
- Promotes by disconnecting from external source
- Preserves current mode (standalone or cluster)
- Requires `apps:operate` permission

**Ask only what you cannot derive:**
- "Which external replica addon should I promote to a standalone instance? (provide addon UUID)"

### Create Read Replica - Service (`POST /api/v1/services/{id}/create_replica`)

**Required fields:**
- `id` (string, path) - Service ID (must be a PostgreSQL primary service)

**Optional fields:**
- `cpu_size` (float) - CPU allocation for the replica (defaults to primary's value)
- `ram_size` (integer) - RAM allocation in megabytes (defaults to primary's value)

**Constraints:**
- Service must be PostgreSQL type
- Service must be a primary (not a replica itself)
- Resource must have sufficient capacity for the replica
- Requires `services:operate` permission

**Ask only what you cannot derive:**
- "Which PostgreSQL service should I create a replica for? (provide service ID)"
- "What CPU and RAM should the replica use? (defaults to primary's values)"

### Promote Replica to Standalone - Service (`POST /api/v1/services/{id}/promote_replica`)

**Required fields:**
- `id` (string, path) - Service ID (must be a PostgreSQL replica service)

**Constraints:**
- Service must be PostgreSQL type
- Service must be a replica (not a primary)
- Requires `services:operate` permission

**Ask only what you cannot derive:**
- "Which replica service should I promote to a standalone instance? (provide service ID)"

### Promote External Replica - Service (`POST /api/v1/services/{id}/promote_external`)

**Required fields:**
- `id` (string, path) - Service ID (must be a PostgreSQL external replica service)

**Constraints:**
- Service must be PostgreSQL type
- Service must be an external replica (created with `creation_mode: external_replica`)
- Promotes by disconnecting from external source
- Preserves current mode (standalone or cluster)
- Requires `services:operate` permission

**Ask only what you cannot derive:**
- "Which external replica service should I promote to a standalone instance? (provide service ID)"

### Update Security Settings (`PUT /api/v1/apps/{uuid}/security`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- At least one of the following optional fields must be provided

**Optional fields:**
- `allow_connections` (boolean) - Allow other applications on the same resource (miget) to connect to this app over the internal network
- `basic_auth_enabled` (boolean) - Enable Basic Authentication for the application
- `basic_auth_username` (string) - Username for Basic Authentication (required when `basic_auth_enabled` is true)
- `basic_auth_password` (string) - Password for Basic Authentication (required when `basic_auth_enabled` is true, leave blank to keep current password)

**Constraints:**
- Requires `apps:manage` permission
- When `basic_auth_enabled` is true, both `basic_auth_username` and `basic_auth_password` are required (unless password already exists and you want to keep it)
- The app response returns `basic_auth_enabled` so you can tell whether Basic Auth is enforced, but Basic Auth **credentials are never returned** by the API.

**Ask only what you cannot derive:**
- "Should I enable Basic Authentication? (true/false)"
- "If enabling Basic Auth, what username should I use?"
- "If enabling Basic Auth, what password should I use? (leave blank to keep current)"
- "Should I allow internal network connections? (true/false)"

### Clone Application (`POST /api/v1/apps/{uuid}/clone`)

Creates a **new application** from an existing one. It is not a snapshot and not a
backup: it is a fresh app that starts out configured like the source.

**Every copy flag defaults to `false`.** A clone with no flags gets the source app's
own settings (ports, resource limits, and the like) and nothing else — no environment
variables, no secret files, no add-ons, no cronjobs. Ask the user what they want
carried over rather than guessing, and pass the flags explicitly.

#### The one decision that changes everything: where the code comes from

**The source app's deployment configuration is not copied.** A clone therefore has no
source to build from, and its `deployment_method` falls back to `git_push`. You have
two ways to finish the job, and you must pick one:

- **`use_parent_image: true`** — the clone runs the *same container image* as the
  source. This is the fast path: nothing to build, nothing to configure, and adding
  `parent_image_auto_sync: true` redeploys the clone whenever the source's image
  changes. Use it for extra environments or extra regions of the same code.
- **Leave it `false`** — the clone is an independent app, and you must give it a
  source afterwards with `PUT /api/v1/apps/{uuid}/deployment` (see that endpoint;
  changing `deployment_method` rebuilds the config, so send every field it needs).
  Use it when the clone will diverge from the source.

If you skip this decision, the user ends up with an app that cannot deploy.

#### Collect the UUIDs first

`addons` and `cronjobs` take UUIDs from the **source** app, so read them before you
clone — you cannot discover them afterwards:

```bash
curl -H "Authorization: Bearer $MIGET_API_TOKEN" https://app.miget.com/api/v1/apps/{source-uuid}/addons
curl -H "Authorization: Bearer $MIGET_API_TOKEN" https://app.miget.com/api/v1/apps/{source-uuid}/cronjobs
```

`clone_data` on an add-on copies its **contents** — database rows, stored files. Without
it you get an empty add-on of the same type. Confirm this one with the user explicitly:
copying a production database into a new app is rarely what someone means by "clone my
app", and it is not reversible from here.

**Required fields:**
- `uuid` (string, path) - Source application UUID to clone from
- `label` (string) - Display name for the cloned application
- `name` (string) - Unique service name for the cloned application (lowercase, alphanumeric with hyphens)
- `project_id` (string) - UUID of the project the clone is created in
- `resource_id` (string) - UUID of the compute resource (Miget) the clone runs on

**Optional fields:**
- `use_parent_image` (boolean, default: false) - Run the source app's image instead of building
- `parent_image_auto_sync` (boolean, default: false) - Redeploy the clone when the parent's image changes. Only meaningful with `use_parent_image`
- `clone_variables` (boolean, default: false) - Copy environment variables
- `clone_secret_files` (boolean, default: false) - Copy secret files
- `clone_scaling_settings` (boolean, default: false) - Copy the scaling profile (replicas, autoscaling)
- `clone_health_checks` (boolean, default: false) - Copy liveness/readiness/startup probe config
- `clone_security` (boolean, default: false) - Copy security settings (allowed connections and Basic Auth)
- `addons` (array, default: []) - Add-ons to clone, each `{uuid: String, clone_data: Boolean}`
- `cronjobs` (array, default: []) - Cronjob UUIDs to clone

```json
{
  "label": "Acme API (staging)",
  "name": "acme-api-staging",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "use_parent_image": true,
  "parent_image_auto_sync": true,
  "clone_variables": true,
  "clone_secret_files": true,
  "clone_security": true,
  "addons": [{"uuid": "{addon-uuid}", "clone_data": false}],
  "cronjobs": []
}
```

#### After the call

The clone comes back in state `cloning` and is provisioned in the background. Poll
`GET /api/v1/apps/{uuid}` until the state settles, then:

- If you did **not** use `use_parent_image`, configure its deployment source now — the
  app cannot deploy until you do.
- Environment variables often name the source app (hostnames, callback URLs, database
  names). Read them back with `GET /api/v1/apps/{uuid}/vars` and tell the user which
  ones look like they need changing. Do not silently rewrite them.
- The clone gets its own URL, derived from its `name`. Hand it over the same way you
  would for a new app.

### Update Health Checks (`PUT /api/v1/apps/{uuid}/health_checks`)

Configures Kubernetes health probes (liveness, readiness, startup).

**Required fields:**
- `uuid` (string, path) - Application UUID

**Optional fields (all optional, provide at least one):**
- `liveness_probe_enabled` (boolean) - Enable liveness probe
- `readiness_probe_enabled` (boolean) - Enable readiness probe
- `startup_probe_enabled` (boolean) - Enable startup probe
- `liveness_probe_path` (string) - Liveness probe HTTP path
- `readiness_probe_path` (string) - Readiness probe HTTP path
- `startup_probe_path` (string) - Startup probe HTTP path
- `*_probe_initial_delay_seconds` (integer) - Seconds before first probe check
- `*_probe_timeout_seconds` (integer) - Seconds before probe times out
- `*_probe_period_seconds` (integer) - Seconds between probe checks
- `*_probe_failure_threshold` (integer) - Consecutive failures before marking unhealthy
- `*_in_app_failure_notification_enabled` (boolean) - In-app notifications on probe failure
- `*_email_failure_notification_enabled` (boolean) - Email notifications on probe failure

Replace `*` with `liveness`, `readiness`, or `startup`.

### Update Scaling Profile (`PUT /api/v1/apps/{uuid}/scaling_profile`)

Configures auto-scaling. Not available on free plan.

**Required fields:**
- `uuid` (string, path) - Application UUID

**Optional fields:**
- `replicas` (integer) - Fixed number of running instances
- `auto_scaling_enabled` (boolean) - Enable automatic horizontal scaling
- `auto_min_replicas` (integer) - Minimum instances when auto-scaling
- `auto_max_replicas` (integer) - Maximum instances when auto-scaling
- `cpu_threshold` (integer) - CPU usage % that triggers scale-up (1-100)
- `memory_threshold` (integer) - Memory usage % that triggers scale-up (1-100)
- `period_enabled` (boolean) - Enable time-based scaling windows
- `scaling_start_time` (string) - Start time for scaling window (HH:MM, 24-hour)
- `scaling_end_time` (string) - End time for scaling window (HH:MM, 24-hour)
- `within_resources` (boolean) - Not implemented yet: scaling is always limited to the resource's allocation. Accepted but ignored.

### Change Application State (`PATCH /api/v1/apps/{uuid}/state`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `state` (string) - Target state: `schedule_start`, `schedule_stop`, or `schedule_restart`

Note: apps use `schedule_*` values. Addons and services use `process_*` values (see below) — they are not interchangeable.

### Change Addon State (`PATCH /api/v1/apps/{uuid}/addons/{id}/state`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Addon ID or UUID
- `state` (string) - Target state: `process_start`, `process_stop`, or `process_restart`

### Change Service State (`PATCH /api/v1/services/{id}/state`)

**Required fields:**
- `id` (string, path) - Service ID
- `state` (string) - Target state: `process_start`, `process_stop`, or `process_restart`

### Rotate Service Password (`POST /api/v1/services/{id}/rotate_password`)

**Required fields:**
- `id` (string, path) - Service ID

**Constraints:**
- Only supported for database/cache services (PostgreSQL, MySQL, Valkey)
- Requires `services:operate` permission
- New password returned in response - update ENV variables manually

### Mount App to Service (`POST /api/v1/services/{id}/mount_app`)

Creates a storage addon on the app linked to this service.

**Required fields:**
- `id` (string, path) - Service UUID
- `app_id` (string) - UUID of the application to mount

**Optional fields:**
- `mount_point` (string) - Container mount path (e.g., /data)
- `sub_path` (string) - Subdirectory of the shared volume to mount instead of its root (relative path, e.g. `media/uploads`; created automatically if missing). Each mounted app can use a different `sub_path` to keep its files apart on the same shared volume.
- `label` (string) - Display label for the mounted addon

### Unmount App from Service (`POST /api/v1/services/{id}/unmount_app`)

**Required fields:**
- `id` (string, path) - Service UUID
- `app_id` (string) - UUID of the application to unmount

---

## Monitoring & Observability

Every app on Miget automatically gets metrics, logs, and pre-built Grafana dashboards — no setup. For **any** observability question (resource usage, request rates, restarts, errors, why a pod or cron run misbehaved), look here and in the Miget monitoring docs first: the REST API deliberately does **not** expose runtime metrics or app logs.

**Docs:** https://docs.miget.com/monitoring/overview · `/metrics` · `/metrics-api` · `/logs`

### In Grafana (UI)

Click **Monitoring** on an app's dashboard to open Grafana (automatic login — no credentials). Three pre-built dashboards ship with every app: **App Overview**, **Pod Details**, and **Logs**; custom dashboards can be built in Grafana.

### Metrics API (Prometheus-compatible)

Base URL `https://metrics.miget.com`. Auth: `Authorization: Bearer $MIGET_API_TOKEN` — the **same Miget API token** you already use for the REST API (see Authentication); there is no separate Grafana credential. Scope with the `X-Workspace-Id: <workspace-uuid>` header. Query language: **PromQL**. Subject to fair-use rate limits (`429` on throttle).

- Instant query: `GET /prometheus/api/v1/query?query=<PromQL>[&time=<ts>]`
- Range query: `GET /prometheus/api/v1/query_range?query=<PromQL>&start=<ts>&end=<ts>&step=<e.g. 60s>`
- Label discovery: `GET /prometheus/api/v1/labels` · `GET /prometheus/api/v1/label/{name}/values`

Metrics are prefixed `miget_` and carry common labels `namespace`, `app`, `addon`, `addon_type`, `instance` (HTTP metrics add `status`/`method`; disk adds `device`). Common series:
- **App:** `miget_app_replicas_desired`, `miget_app_replicas_available`, `miget_app_http_responses_total`, `miget_app_http_response_time_seconds_bucket`
- **Instance (pod):** `miget_instance_cpu_usage`, `miget_instance_memory_used_bytes`, `miget_instance_net_recv_bytes_total`, `miget_instance_disk_read_bytes_total`, `miget_instance_status_phase`, `miget_instance_restarts_total`
- **Volume:** `miget_volume_used_bytes`, `miget_volume_size_bytes`, `miget_volume_iops_limit`
- **Addon (PostgreSQL):** connections, database size, replication lag

Example — last hour of CPU for an app:
`GET https://metrics.miget.com/prometheus/api/v1/query_range?query=miget_instance_cpu_usage{app="my-app"}&start=<ts>&end=<ts>&step=60s`

### Logs API (Loki-compatible)

Same host and auth as the Metrics API. Query language: **LogQL**.
- `GET https://metrics.miget.com/loki/api/v1/query_range?query={app="my-app"}&limit=100`
- Narrow to a specific cron run by pod: `{app="my-app", pod="<last_job_name>"}`.

See https://docs.miget.com/monitoring/logs#logs-via-api.

### Retention (by plan)

| Plan | Metrics | Logs |
|---|---|---|
| Free | 30 days | 3 days |
| Pay as you grow | 13 months | 7 days |

Short-lived cron pods may fall between metric scrapes, so their `miget_instance_*` series can be sparse or absent; their **logs** are still queryable via the Logs API (and the cron `stream_logs` endpoint) while retained.

---

## Best Practices

1. **Use API Tokens for Automation**
   - API tokens are better for CI/CD and automation; they expire only if given an expiry date
   - Generate a user token at `https://app.miget.com/my_account#api_tokens`, or a workspace token in workspace settings under Developers
   - Read them from `MIGET_API_TOKEN` — never ask a user to paste a token or password into a conversation, and never write a token value into a command or config file on their behalf

2. **Deployment Workflow**
   - Create resource -> Create project -> Create app -> Deploy
   - Monitor deployments using `/deployments` endpoints
   - Use `stream_logs` for real-time build monitoring

3. **Environment Variables**
   - Use app-level vars for app-specific configuration
   - Use project-level vars for shared configuration across apps

4. **Deployment Methods**
   - `git_push` - Push to Miget-hosted Git remote
   - `github` - Best for GitHub repositories (supports auto-deploy on push)
   - `public_git` - For public Git repositories
   - `container_registry` - For pre-built container images from a registry (Docker Hub, GHCR, etc.)
   - `parent_image` - For inheriting images from parent apps
   - `kamal` - For deploying from local machine using Kamal (`kamal deploy`). The app must be created with Kamal from the start - you cannot switch an existing app to Kamal. Registry password is auto-generated.

5. **Resource Management**
   - Resources are region-specific
   - Choose appropriate plan type (`dev` for development, `pro` for production)
   - Add components (extra RAM/CPU) as needed

6. **Troubleshooting a "URL not reachable" complaint**
   - **Check the ports first.**
     - If the user is hitting the default `*.migetapp.com` URL: the app must listen on port `5000` (HTTP is always served from `5000` and cannot be changed). If it's listening on a different port, tell the user to change the app to listen on `5000`.
     - If the user is hitting a custom TCP/UDP port directly: list ports via `GET /api/v1/apps/{uuid}/ports` and confirm the port exists and is public. Extra ports are **private by default** — expose them with `expose_publicly`.
   - Only after ports look right, check deployment status, domains, and logs.

7. **Observability — metrics, logs, dashboards**
   - For resource usage, request rates, restarts, errors, or any runtime "what's happening" question, use the **Monitoring & Observability** section above (Grafana dashboards + Prometheus/Loki query APIs at `metrics.miget.com`) and https://docs.miget.com/monitoring/overview.
   - The REST API only serves **build/deploy** logs (`GET /api/v1/apps/{uuid}/deployments/{id}/logs`, `/stream_logs`) and **cron run** logs (`GET /api/v1/apps/{uuid}/cronjobs/{id}/stream_logs`). App **runtime** logs and all metrics live in the monitoring APIs, not the REST API.

8. **In-container HTTP tooling**
   - The default build image is minimal and may not include `curl`/`wget`. For outbound HTTP from your app or a cron command, prefer your language's native HTTP client, or install the tool in your Dockerfile.

---

## Example: Complete Application Setup (GitHub)

```http
# 1. Authenticate — every request below carries this header
Authorization: Bearer $MIGET_API_TOKEN

# 2. Get or create resource
GET /api/v1/resources
# If none exists (plan_code_name comes from GET /api/v1/plans):
POST /api/v1/resources
{
  "plan_code_name": "miget_hobby_0",
  "region_code": "eu-east-1"
}

# 3. Create project
POST /api/v1/projects
{
  "name": "my-api",
  "description": "REST API project"
}

# 4. Create application with GitHub deployment
POST /api/v1/apps
{
  "name": "api-server",
  "label": "API Server",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "builder": "auto",
  "ram_size": 256,
  "cpu_size": 0.5,
  "deployment_method": "github",
  "deployment_config": {
    "credential_id": "{github-credential-uuid}",
    "repository": "username/api-repo",
    "branch": "main",
    "auto_deploy_enabled": true
  }
}

# 5. Add environment variables
POST /api/v1/apps/{app-uuid}/vars
{
  "key": "NODE_ENV",
  "value": "production"
}

# 6. Add database addon (label and postgres_version are required)
POST /api/v1/apps/{app-uuid}/addons
{
  "type": "postgres",
  "label": "Primary database",
  "postgres_version": "17"
}

# 7. Deploy (optional: specify commit_sha, branch, or custom_tag)
POST /api/v1/apps/{app-uuid}/deploy
{
  "commit_sha": "abc123",  # Optional: deploy specific commit (for Git/GitHub)
  "branch": "main"         # Optional: deploy specific branch (for GitHub)
}

# 8. Monitor deployment
GET /api/v1/apps/{app-uuid}/deployments?period=7days&status=running
# Wait for status: "completed"

# 8b. Rollback if needed (to a previous deployment)
POST /api/v1/apps/{app-uuid}/deployments/{previous-deployment-id}/rollback

# 9. Add custom domain
POST /api/v1/apps/{app-uuid}/domains
{
  "domain": "api.example.com"
}
```

## Example: Complete Kamal Application Setup

```http
# 1. Authenticate (same as above)

# 2. Get or create resource (same as above)

# 3. Create project (same as above)

# 4. Create application with Kamal deployment
POST /api/v1/apps
{
  "name": "my-rails-app",
  "label": "My Rails App",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "builder": "dockerfile",
  "ram_size": 512,
  "cpu_size": 0.5,
  "deployment_method": "kamal",
  "deployment_config": {
    "ssh_keys": ["ssh-ed25519 AAAA... user@machine"]
  }
}

# 5. Retrieve app details for deploy.yml configuration
GET /api/v1/apps/{app-uuid}
# Response includes deployment_config with:
#   - registry hostname, username, image path, and registry password
#   - SSH endpoint hostname and port
# Use these values to create your local config/deploy.yml
# Set the registry password as MIGET_REGISTRY_PASSWORD env var locally

# 6. (Optional) Add environment variables for the app
POST /api/v1/apps/{app-uuid}/vars
{"key": "RAILS_ENV", "value": "production"}

POST /api/v1/apps/{app-uuid}/vars
{"key": "SECRET_KEY_BASE", "value": "your-secret-key"}

# 7. User deploys from their local machine:
#    kamal setup   (first time)
#    kamal deploy  (subsequent deploys)
# These commands are run locally, NOT via the API
```

---

## Additional Resources

- **API Documentation:** `https://app.miget.com/api/v1/docs`
- **OpenAPI Spec:** `https://app.miget.com/docs/openapi.json` (use as fallback when this guide doesn't cover a specific endpoint or parameter)
- **Documentation:** `https://docs.miget.com`
- **Website:** `https://migetapp.com`
- **Support:** `hello@miget.com` or `https://migetapp.com/join-discord`

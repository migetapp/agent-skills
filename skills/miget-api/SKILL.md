---
name: miget-api
description: Deploy and manage apps, databases, buckets, private networks and services on Miget PaaS. Covers authentication, resource provisioning, deployments, add-ons, domains, environment variables, VPCs and VPN, and every API endpoint. Use this skill whenever the user mentions Miget, deploying an app to Miget, a miget/resource, or asks to ship, host, scale or debug an application on the Miget platform — including when they only describe the goal ("get this Rails app online", "give my database a private address") without naming Miget.
version: 1.0.0
---

# Miget API - Guide for AI Agents

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
| **`name` is server-assigned and always suffixed** | The server appends a random suffix to whatever `name` you send — `wall` comes back as `wall-pfhqv`, and a resource is `migetmxq`. Apps, addons, services, stacks and buckets all work this way. **Secret files go further** — you cannot send a name at all, only `filename` and `text`, and the whole name is generated server-side (`secret-file-gfeey`). **The suffixed `name` is the identifier everywhere**: public URLs, Git remote paths, connection variable keys. `service_name` (the unsuffixed form) appears in exactly one place, `internal_url`, and `label` is display text with no addressing role. Never reconstruct any of these from the name you sent, and never strip a suffix that looks like noise — read the field back and use it verbatim. |
| **`state` mixes two vocabularies** | On apps, addons and services `state` reports the platform lifecycle while the object is provisioning and the raw **Kubernetes** status once it is up. There is no `active`, and a healthy database reports **`healthy`** (storage reports **`bound`**) — only an *app* reports `running`. Never poll for one hard-coded string; see "Reading `state`". |
| **Addon vs standalone service** | An addon's lifecycle is tied to its app and is the right default — deleting the app deletes it. A standalone service outlives any single app. Note that explicit mounting works only for `shared_storage`; a shared database is shared by pointing several apps at its connection variables, not by mounting it. |
| **Billing is not on the API — and deleting is not cancelling** | There is no endpoint that cancels a subscription, changes a plan's payment, suspends or deletes a workspace. Those are dashboard-only, on purpose. The trap is `DELETE /api/v1/resources/{uuid}`: it destroys the resource and everything on it, but the **subscription keeps charging** for the workspace plan and any other resource. Deleting things over the API never stops a bill. Send the user to the dashboard — see "Stopping the bill" in `references/workspace.md`. |
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

   Compare it with the `version` in this file's frontmatter. **Only if the published version is newer than yours**, tell the user once and walk them through the update. First refresh everything the CLI manages:

   ```bash
   npx skills update
   ```

   Then **verify the copy your own agent reads**. A general update only refreshes agents the skill was installed for, so it can silently leave the current agent on an old version. Re-read the frontmatter `version` in this agent's own skill directory — for example `~/.claude/skills/miget-api/SKILL.md` (global) or `./.claude/skills/miget-api/SKILL.md` (project). If it still shows the old version, install it for this agent explicitly:

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

**Region.** The platform has no default region. Prefer, in order: the region of a resource the user already owns; then the region implied by where they are (North America → `us-east-1`, everywhere else → `eu-east-1`, which is the same fallback the platform uses at signup). Say which you picked and offer to change it. This does not apply to static sites, which accept `eu-east-1` only — see `references/static-sites.md`.

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

Preview environments (review apps) work only for `github` apps — so a repo that will want PR previews should be on `github` from the start rather than migrated later. You configure them over the API with `PUT /api/v1/apps/{uuid}/preview_environments/config`; the environments themselves are created by GitHub webhooks, never by you.

**Addons.** Their defaults are sensible; supply a size only when the app clearly needs more.

| Addon | Default RAM | Default disk | Default CPU |
|---|---|---|---|
| `postgres` | 128 MiB | 0.5 GiB | 0.1 |
| `mysql` | 128 MiB | 0.5 GiB | 0.1 |
| `valkey` | 32 MiB | 0.1 GiB | 0.1 |

Prefer an **addon** over a standalone service unless more than one app genuinely needs the same database.

**The free plan needs a card on file.** Nothing is charged; the card is how Miget tells a person from a throwaway account, and each card is good for one free resource. A request for a free resource from an account with no saved card is rejected with `422` and a message saying so; a card already used for somebody else's free resource is rejected the same way, pointing at support. Cards added through Apple Pay or Google Pay count. This applies to creating a resource and to moving an existing one down to the free plan.

**A new account gets 30 days free on Hobby 5 — and only there.** An account that has never paid for a resource, and has not used a trial before, gets a 30-day free trial when it buys plan `miget_hobby_5` **on its own, with no `components`**. The checkout link comes back as usual and charges nothing today; billing starts at the end of the trial at the plan's normal price, and cancelling before then costs nothing. Any other plan is an ordinary purchase, cheaper ones included, and so is Hobby 5 with a paid component on the same request — there is no partial version of this. The trial is spent once per account and once per workspace.

It needs **a saved payment card**, exactly like the free plan and with the same `422`: no card on file, or a card already used for someone else's trial, and the request is rejected before any resource exists. Reading the plan the user asked for and quietly buying it without the trial would cost them money they did not expect, so relay the rejection and point at billing settings instead.

Once a trial is running, adding a resource or moving up to a dearer plan **ends it immediately and bills the new total that day** — a downgrade or a sideways move does not. Say this before an agent "upgrades while it is free anyway".

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
3. If that fails, pull build logs from `GET /api/v1/apps/{uuid}/deployments/{id}/logs`, and runtime logs from the Loki API (see `references/observability.md` — runtime logs are not on the REST API).
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

Short definitions so you can read a request and know which noun it is about. Each
one has a reference file with the endpoints, the required fields and the traps.

- **Workspace** — the tenant everything belongs to and the thing that gets billed.
  Requests carry `X-Workspace-Id`, or fall back to the user's default. → `references/workspace.md`
- **Resource (Miget)** — the compute a workload runs on. This is what costs money;
  applications and add-ons draw RAM, CPU and disk from its quota. → `references/workspace.md`
- **Project** — a grouping inside a workspace, and the unit access can be
  restricted by. A project you cannot reach is invisible, not forbidden. → `references/workspace.md`
- **Application** — one deployed service. Six deployment methods, from a Git push
  to a container image. → `references/apps.md`
- **Add-on** — a database, cache or volume owned by one application, dying with it.
  → `references/addons-services.md`
- **Service** — the same types standing on their own, shareable across
  applications and outliving any of them. → `references/addons-services.md`
- **Bucket** — S3-compatible object storage on a resource. → `references/buckets.md`
- **Stack** — a multi-service application deployed from one `docker-compose.yml`.
  A compose file in the repo means a stack, never a hand-built set of
  applications. → `references/stacks.md`
- **Static site** — prebuilt files served from storage. Not an application, not
  under `/api/v1/apps`, and free to run. → `references/static-sites.md`
- **VPC** — a private network workloads join to reach each other by name, with a
  VPN to reach it from outside. → `references/networking.md`
- **Webhook** — a signed POST to your endpoint when something happens, so nobody
  has to poll. → `references/webhooks.md`

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

## Where the details live

This file is the part that changes how you work. Everything specific to one kind
of object — endpoints, required fields, worked examples, and the step-by-step for
leading somebody through a setup — lives in a reference file next to it.

**Open the file before the first write call against that object.** The required
fields are deliberately not repeated here, so working from memory means a `400` or
a `422` and a guess at which of a dozen fields it meant. Reading costs one tool
call; a wrong `POST /api/v1/apps` costs a rebuild.

| Read | Before you | Because |
|---|---|---|
| `references/apps.md` | Create, deploy or configure an application | The deployment method decides which `deployment_config` fields are mandatory, and the name you send is not the name you get back |
| `references/addons-services.md` | Add a database, cache or volume | `label` and the version field are required despite being marked optional, and the platform writes the connection variables for you |
| `references/buckets.md` | Create a bucket or touch a policy | Policies and ACLs take raw S3 JSON and XML, and refuse anything else with a parser error |
| `references/networking.md` | Create a VPC or enable a VPN | A new VPC is `pending` with null addresses and already owns a subnet; one VPN terminator costs $199/mo and takes a scarce address |
| `references/stacks.md` | Deploy anything with a compose file | Building the applications by hand produces something that will not reconcile |
| `references/static-sites.md` | Put static files online | It is not an application and does not answer on `/api/v1/apps` |
| `references/webhooks.md` | Wire up notifications | An empty `app_uuids` means every application, and preview environments are excluded unless asked for |
| `references/workspace.md` | Buy a resource, or explain a 403 | Creating a resource spends the user's money, and an unreachable project answers 404 rather than 403 |
| `references/observability.md` | Debug something that deployed but misbehaves | Metrics are Prometheus-shaped and logs are Loki-shaped |

A new reference file has to be added to this table, or nothing will ever open it.

These files are schema references, not interview scripts — derive what you can from
the repository and the account first, fold the result into a single plan card, and
ask only about what is left. What you must not do is *guess silently*: an unstated
assumption about region or plan produces resources in the wrong place that then have
to be deleted and recreated. Infer, say what you inferred and from where, and
confirm once.

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

## Additional Resources

- **API Documentation:** `https://app.miget.com/api/v1/docs`
- **OpenAPI Spec:** `https://app.miget.com/docs/openapi.json` (use as fallback when this guide doesn't cover a specific endpoint or parameter)
- **Documentation:** `https://docs.miget.com`
- **Website:** `https://migetapp.com`
- **Support:** `hello@miget.com` or `https://migetapp.com/join-discord`

# Changelog

## 0.2.1 — 2026-07-27

Keep the user's credentials out of the agent. The setup flow previously had the agent ask for an API token in chat and write it into a shell config inline, which put a live secret into the conversation transcript and the user's shell history.

- **The agent never sees the token:** it checks whether `MIGET_API_TOKEN` is set without printing it, and if it is missing, hands the user a snippet to run in their own terminal that prompts for the token without echoing it, appends it to their shell config and exports it for the current session. The instruction to "share the token back" is gone, along with the `echo 'export MIGET_API_TOKEN="..."'` snippets that placed a secret on a command line
- **Reference the variable, not the value:** every example now sends `Authorization: Bearer $MIGET_API_TOKEN`, so a token value has no reason to appear in a command, a header example or a log
- **Password authentication is no longer the headline method:** signing in with an account email and password was documented as Method 1, ahead of API tokens. It is now Method 2, marked as not for agents, and states why — it grants everything the account can do and cannot be revoked without a password reset. API tokens, which are individually revocable, are Method 1
- **The end-to-end walkthrough no longer starts with a password:** it opens with the token header that the remaining requests carry, instead of a `POST /api/v1/auth/sign_in` carrying an email and password
- **Secret hygiene covers more than the API token:** the handling rule now names Git tokens, registry credentials, addon connection strings and environment-variable values, and says to summarise a response body containing a secret rather than printing it
- **If a token is pasted anyway:** the skill tells the agent to say so plainly and recommend rotating it, rather than silently carrying on

## 0.2.0 — 2026-07-27

Rewrite the skill to deploy from what a project already says, document the Git build settings the API now accepts, and close the gaps that a full end-to-end deployment turned up — units, names, required fields and how to read a failure.

- **Deploy without the interrogation:** the skill previously told agents to ask the user for every required field before creating anything, which turned "deploy this repo" into six questions before any work happened. It now reads the repository and the account first, presents a single plan covering everything that will be created — each row naming the evidence behind it, plus the monthly cost — and asks for one confirmation. Explicit user instructions are still used verbatim, and questions are reserved for choices that genuinely cannot be derived or that are expensive to undo
- **Platform constraints in one place:** the fixed rules that break a first deployment — HTTP always served on port 5000, no implicit release phase, extra ports private and TCP/UDP only, app-to-app traffic off until enabled, runtime logs and metrics living outside the REST API, `quota` reported in bytes — are now stated up front instead of being scattered through the reference
- **Reading a project:** a signal-to-conclusion table mapping `package.json`, `requirements.txt`, `Gemfile`, `go.mod`, `prisma/schema.prisma`, `Dockerfile`, `docker-compose.yml` and friends to the deployment method, builder, addons and migration command they imply — including that a compose file means a stack rather than an app. Environment files are read for variable *names*; values are imported through the vars endpoint and never printed
- **Defaults worth deriving:** sizing guidance per workload, how to choose a region and the cheapest plan that fits, and addon defaults. Notably, omitting `ram_size`/`cpu_size` when creating an app hands it the resource's entire remaining RAM and CPU — the skill now tells agents to always set them explicitly
- **Deployment is verified, not assumed:** a deployment reaching `completed` only means the image built and the pods started. The skill now defines "done" as the app's URL answering, with a symptom-to-fix table covering the wrong listen port, missing variables, an unmigrated database, out-of-memory restarts and unexposed ports
- **Git build settings:** `POST /api/v1/apps` and `PUT /api/v1/apps/{uuid}/deployment` now accept `project_path`, `run_command`, `language`, `build_command`, `pre_deploy_command`, `post_deploy_command` and `use_dhi` in the `public_git` and `github` deployment configuration, and they are returned on the app response
- **Release-phase migrations:** there is no implicit release phase, so database migrations belong in `pre_deploy_command` (`npx drizzle-kit migrate`, `alembic upgrade head`, `bin/rails db:migrate`, …) where they run once before a release rather than on every replica boot
- **`builder: "custom"` is now usable:** it requires `language` and `build_command`, which previously could not be sent — the builder was selectable but not configurable. A request that selects it without both is rejected with `422`, rather than creating an app that cannot build
- **Monorepos:** set `project_path` to the subdirectory holding the app and the build treats it as the root
- **App response for `public_git`:** `GET /api/v1/apps/{uuid}` returns the repository under `repository_url`, matching the field name used to create and update it. This request previously failed for `public_git` apps
- **Deployment config in the response:** `POST /api/v1/apps` and `PUT /api/v1/apps/{uuid}/deployment` now return the configuration they just saved, instead of a null `deployment_config`, so reading back your own configuration no longer costs a follow-up request
- **Telling three Git failures apart:** the reachability check that runs when a `public_git` repository is set has three distinct outcomes — the repository or branch does not exist, the Git host is rate-limiting the platform, and the branch exists but is empty. Only the first is something the caller can fix by editing the URL, so all three are now tabled with what to do about each; a throttled host previously read as a bad URL
- **Deployment updates are a patch:** `deployment_config_attributes` is applied over the stored configuration — fields you omit keep their value, and a field sent as `""` is cleared. Adding a `pre_deploy_command` to a live app no longer requires restating its branch, project path and commands, and no longer silently drops them. Sending a *different* `deployment_method` still builds the configuration from scratch, so every field that method needs must be supplied
- **Derived stack variables:** some stacks declare a variable computed from another (a Supabase `ANON_KEY` signed with the stack's `JWT_SECRET`, for example). The platform always derives those from their source, so a value supplied for one in `env_var_overrides` is ignored — supply the source variable, or let `auto_populate_required_vars` generate it
- **Sizes you send and sizes you read back use different units:** `ram_size` and `disk_size` on create and update are MiB and GiB, while every size the API *returns* — `quota.*` on apps, `ram_size`/`disk_size` on plans, and `total_`/`total_used_`/`available_` on resources — is bytes. Comparing one against the other without converting is the usual cause of "it should have fit"; the resource and plan fields now say so explicitly
- **Every app has a public URL, and you cannot predict it:** the app response now carries `public_url` (`https://<name>.<region-code>.migetapp.com`). It matters because the server appends a random six-character suffix to the `name` you send, so the URL is never the one you would have constructed — read `public_url` back from the create response instead. The same suffix is applied before the 40-character name limit is checked, so a name has to be 34 characters or fewer. Custom domains are not included here; they are listed under `GET /apps/{uuid}/domains`
- **Plan codes are opaque and must be looked up:** `plan_code_name` on `POST /api/v1/resources` takes a `code_name` from `GET /api/v1/plans` verbatim (`miget_hobby_0`, `miget_pro_1`), not a friendly word like `starter`, and the values differ between environments — guessing returns a generic `422`. `plan_type` on that request is ignored and kept only for backward compatibility; the plan object itself now returns `plan_type` so `dev` and `pro` plans can still be told apart
- **Addon and service creation state what they actually require:** `label` and the type's version field are required on addons, `mount_point` and `storage_access` are required for standalone storage (and inherited when `service_id` is given), and `mount_point` is required for a `shared_storage` service. Accepted versions are now listed rather than exemplified — PostgreSQL `18`–`13`, MySQL `8.2` and `8.0`, Valkey `7` and `7.2` — and anything else is rejected with `400`. MySQL `8.4` appeared in earlier examples but was never supported
- **What an addon actually injects:** a database addon adds exactly one variable, named after the addon — `<ADDON_NAME>_URL`, upcased with dashes turned into underscores, so an addon named `postgres-mwvzq` yields `POSTGRES_MWVZQ_URL`. There is no `DATABASE_URL` and no broken-out host/port/user variables. Frameworks that expect `DATABASE_URL` need it set as a second variable pointing at the same value
- **Reading a failed deployment:** a failed deployment carries no reason — no error field, no failing phase, no exit code — so the log body is the only source of truth and the skill now treats fetching and interpreting it as part of deploying, not a follow-up request. Logs `404` while a run is in progress and the body distinguishes "not available yet" from "aged out". The log is one blob covering build *and* release, and a release-phase failure is still reported as `Build failed`, so it has to be read to the end
- **What the release phase can and cannot do:** `pre_deploy_command` runs in the runtime image as an unprivileged user with no package manager, so nothing can be installed from it. Two concrete consequences are documented: Prisma's migration engine needs OpenSSL, which the default `auto` image lacks, so Prisma requires `builder: "dockerfile"`; and `NODE_ENV` is `production` during the build, so a plain `npm ci` skips `devDependencies` and any build needing a bundler dies with a module-resolution error — use `npm ci --include=dev`
- **`errors` is a string, not an array:** validation failures return the messages comma-joined in a single string (`"Label is too long (maximum is 40 characters), Name is invalid"`). Iterating it yields characters; split on `", "` if the individual messages are needed
- **Corrected endpoint:** `allow_connections` is set through `PUT /api/v1/apps/{uuid}/security`

---

## 0.1.10 — 2026-07-21

Document new app, deployment, and cron-job response fields, the deploy-in-progress conflict, and Grafana monitoring (metrics & logs APIs).

- **App internal URL & auth state:** `GET /api/v1/apps/{uuid}` now returns `internal_url` (the `<service>.<resource>.<region>.migetapp.internal:5000` address for app-to-app and addon connections, null until a compute resource is assigned) and `basic_auth_enabled` (whether HTTP Basic Auth is enforced at the ingress — credentials are never returned)
- **Deployment commit metadata:** `GET /api/v1/apps/{uuid}/deployments` records now include `commit_sha`, `commit_message`, and `branch` for git-based deployment methods (null otherwise), so you can confirm which commit is live
- **Cron run logs:** documented `GET /api/v1/apps/{uuid}/cronjobs/{id}/stream_logs` (SSE) for streaming the most recent run's logs
- **Deploy while busy:** `POST /api/v1/apps/{uuid}/deploy` now returns `409 Conflict` when a deployment is already in progress — poll `GET /api/v1/apps/{uuid}/deployments` and retry once it settles
- **Cron reschedule and resource units:** clarified that a cron job's schedule cannot be changed via `PUT` (delete and recreate to reschedule), and that `quota.ram_size` is reported in bytes
- **Monitoring & Observability:** added a section covering the Grafana dashboards every app ships with and the Prometheus-compatible Metrics API and Loki Logs API at `https://metrics.miget.com` (PromQL/LogQL, auth, common `miget_*` series, retention) — where agents should look for resource usage, request rates, restarts, errors, and runtime logs
- **Self-updating:** the skill now states its own version and tells the agent to compare it once per session against the latest published release, walking the user through `npx skills update` and, when the running agent still reads an old copy, an explicit per-agent `npx skills add … -a <agent>` — only when a newer release exists — a stale copy otherwise keeps describing endpoints that no longer match the API

---

## 0.1.9 — 2026-07-16

Correct the `within_resources` scaling field: it is accepted but no longer does anything.

- **`within_resources`:** `PUT /api/v1/apps/{uuid}/scaling_profile` still accepts the field, but it is now ignored — scaling is always limited to the resource's allocation. Scaling beyond the resource is not implemented

---

## 0.1.8 — 2026-07-07

Add guidance for deploying catalogue stacks and authoring `compose.miget.yaml`.

- **Catalogue stacks:** deploy well-known self-hostable apps (WordPress, Ghost, n8n, …) from the curated deployable.sh catalogue (`deployable-sh/stacks`) instead of an arbitrary compose file found on the web — point `repository_url` at the catalogue repo and set `compose_path` to the stack directory
- **`compose.miget.yaml`:** documented the `x-miget` overlay for tuning a stack on Miget — per-service `ram`/`cpu`/`private`, managed Postgres/Valkey add-ons with `storage`, volume `size`/`type`, and the port-5000 public-entry convention — and how to author one when a repo has only a base compose file

---

## 0.1.7 — 2026-07-05

Add Compose Stacks and Git Credentials, plus source-link validation and var-path fixes.

- **Docker Compose Stacks:** documented the full `/api/v1/stacks` surface — analyze, create, list/get, update, deploy, deployment config, deployment history, and delete — plus the analyze-then-create workflow and required env-var resolution
- **Git Credentials:** added the read-only `GET /api/v1/git_credentials` (list/get); the returned `uuid` is the `credential_id` for stacks and `github`/`public_git` apps
- **Container Registry Credentials:** added the workspace-scoped credentials endpoints and creation fields
- Fixed the `public_git` deployment config field: it is `repository_url` (a full HTTPS URL), not `repository`. `repository` remains the `github` field (`owner/repo` format)
- Added URL validation guidance for `public_git` (HTTPS-only `https://host/owner/repo`, no SSH/`http://`/ports/nested paths, repo + branch must be reachable) and for the container `image_url` (no scheme, no embedded tag — put the tag in the separate `tag` field)
- Corrected the app/project variable endpoint paths (flat `/vars` routes; removed the non-existent `/vars/{var-uuid}` segments)
- Documented the app `private_access` field and PostgreSQL read-replica / promote endpoints

---

## 0.1.6 — 2026-05-25

Fix ports API endpoint URLs — remove broken `workspace/app-name` path segments.

The ports API was using a non-standard nested URL that required `workspace_name` and `app_name` alongside the app UUID. Endpoints are now standardised to `/api/v1/apps/{uuid}/ports/{port_id}/`. Also added the missing `GET /api/v1/apps/{uuid}/ports/{port_id}` show endpoint.

---

## 0.1.5 — 2026-05-15

Document `resource_id` params, domain verification, and app state endpoint.

- Buckets and services now use `resource_id` (UUID); `miget_id` is documented as a deprecated legacy alias
- Added `POST /apps/{uuid}/domains/{domain_uuid}/verify` with the DNS TXT workflow and async polling guidance
- Expanded `GET` domain response to include `verification_status`, `verification_token`, and `dns_target`
- Added `PATCH /apps/{uuid}/state`; noted that `schedule_*` values (apps) and `process_*` values (addons/services) are not interchangeable

---

## 0.1.4 — 2026-05-07

- Dropped `region_id` from app creation examples (`resource_id` is now required)
- Documented `command` and `args` overrides for container deployments
- Noted that `create_replica` responses now include the full replica entity

---

## 0.1.3 — 2026-04-18

Added `git_push` deployment workflow and port 5000 guidance to the miget-api skill.

---

## 0.1.2 — 2026-04-07

Renamed API token prefix from `miget_api_*` to `miget_live_*`. Update any stored tokens or shell exports to use the new prefix.

---

## 0.1.1 — 2026-03-27

- Added `CONTRIBUTING.md` and `CODE_OF_CONDUCT.md`
- Added authentication setup instructions (token creation and `MIGET_API_TOKEN` export) to README

---

## 0.1.0

Initial release. Follows the [Agent Skills Open Standard](https://agentskills.io/home).

- Added Miget API agent skill covering authentication, apps, projects, resources, services, buckets, addons, domains, environment variables, deployments, and cronjobs
- JWT and API token authentication support
- Agent behavioral guidelines for safe and predictable API usage
- Complete endpoint reference with request/response examples
- Required fields documentation for all creation endpoints

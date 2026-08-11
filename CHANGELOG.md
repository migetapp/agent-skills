# Changelog

## 0.4.0 — 2026-08-11

Document outbound webhooks, a subsystem the skill had no coverage of at all. An agent asked to "tell me when a deploy finishes" previously had nothing to reach for but a polling loop, and had no way to find out why an endpoint that looked configured was receiving nothing.

- **Deployment events reach an endpoint instead of a polling loop:** register a URL with `POST /api/v1/webhooks` and Miget POSTs a signed JSON payload to it when a subscribed event occurs. There are two event types — `deploy_started` when a deployment enters `running`, and `deploy_ended` when it reaches `completed`, `failed` or `cancelled`. There is no separate build event, because build and deployment are one lifecycle and `build_id` *is* the deployment UUID. The full payload shape is documented, along with the `webhook-id`, `webhook-timestamp` and `webhook-signature` headers: deliveries follow the [Standard Webhooks](https://standardwebhooks.com) specification, so any Standard Webhooks library verifies them as-is, and rejecting a timestamp older than a few minutes is what makes a captured request non-replayable
- **The signing secret is returned exactly once:** the create response is the only one carrying `secret`, every other endpoint omits it, and there is no rotation endpoint. An agent that does not store it at creation cannot recover it — the only route back is deleting the webhook and creating another. Called out at the point of creation rather than buried in a field table
- **Scoping is inverted from what you would guess:** an empty `app_uuids` means *every* app in the workspace, including apps created later, not "no apps". Narrowing is opt-in, and the payload carries `data.app_id` either way so a consumer can filter on its own
- **Static sites emit the same two events:** a deployment of a static site is reported exactly like an application's, with `data.app_id` holding the UUID that `GET /api/v1/static` returns — which is also what scopes a webhook to a single site. The one difference is on the payload rather than the flow: a `zip` or `sftp` deployment has no commit behind it, so `commit_sha`, `commit_message` and `branch` arrive as `null`, and a consumer that reads them unconditionally breaks on the first upload-based site
- **Verify an endpoint before it matters:** `POST /api/v1/webhooks/{uuid}/test` POSTs a signed event immediately and returns the resulting delivery, so a URL and its signature verification can be checked without waiting for a real deployment. It works on a disabled webhook and answers `200` whether or not the endpoint accepted it, so the outcome is read from `status` on the returned delivery rather than the HTTP code. One trap is called out explicitly: the test carries `"type": "ping"`, which is **not** one of the subscribable types, so a consumer that rejects unknown types fails a test its endpoint would otherwise pass
- **Diagnose a delivery that never arrived:** `GET /api/v1/webhooks/{uuid}/deliveries` returns the recent history with `status`, `response_code`, the response body or transport error in `message`, an `attempts` count, and `payload` — the exact body that was sent, so what was received can be compared against what left. One entry per event, updated in place across retries, last 50 per webhook. `POST .../deliveries/{delivery_uuid}/retry` replays one under its original event id. The skill says how to read the two silences apart: a `failed` entry is a problem at the consumer, while no entries at all means no subscribed event has fired
- **Retries span about a day, not a moment:** 9 attempts spaced 30s, 2m, 8m, 30m, 1h, 3h, 8h and 12h apart with jitter. Endpoints therefore have to be idempotent over a far longer window than a short backoff would imply, deduplicating on the event `id`, which is stable across automatic retries *and* manual replays. A failing endpoint is never disabled automatically
- **A URL that is not publicly reachable is rejected:** loopback, RFC1918, link-local including `169.254.169.254`, CGNAT, and the `localhost`, `.local` and `.internal` hostnames all answer `422`. The host is re-resolved before every delivery, so a name that later starts resolving to a private address stops being delivered to rather than being retried. Worth knowing before pointing a webhook at a tunnel or a development machine

## 0.3.0 — 2026-08-11

Document static sites, a resource type the skill had no coverage of at all, and fix three things that stall an agent part-way through a deploy: a state value it is told to wait for that never arrives before the first deploy, a cloning endpoint documented as a bare parameter list, and a claim that SSH keys cannot be registered over the API.

- **A static site deployment can be stopped or replayed:** `POST /api/v1/static/{uuid}/deployments/{id}/cancel` stops a build that is still running, and `POST .../rollback` republishes the site from the commit a previous deployment shipped. Both are narrower than their application counterparts, and the shape of a static site is why. Cancelling only means something for `git_push` and `github`, which run a real build; a `zip` or `sftp` deployment is a direct sync that finishes in seconds, with nothing to interrupt. Rolling back is `github` only: a static site produces no image, so there is no stored artifact to redeploy — the platform rebuilds that commit instead, which it can only do for a source it can fetch on demand. A `git_push` site is rolled back by pushing again, and upload sources carry no commit at all; both answer `422`
- **The SFTP target comes back ready to use:** `deployment_config.sftp_endpoint` carries the complete `user@host`, so there is no need to assemble it from `sftp_username` and the site's region
- **Static sites are their own resource, not a kind of application:** they host prebuilt HTML, CSS and JavaScript from object storage, and they are unreachable through `/api/v1/apps` — the endpoints live under `/api/v1/static`. There is no compute resource, no replicas, no ports, no environment variables and no cost. Three rules trip up a first attempt and are called out up front: the name is exact and globally unique, so a taken name is a `422` rather than a silent rename, the content source is fixed at creation because it decides what gets provisioned, and the region is not yours to pick (below). Four sources get a runbook each — `github` and `git_push` are built for you with the generator auto-detected across 33+ frameworks, while `zip` and `sftp` take already-built output, so uploading a project rather than its output directory publishes the source files
- **A static site lives in `eu-east-1` and nowhere else:** the general advice on choosing a region — take the one the user's existing resources are in, or the one their location implies, which sends a North American user to `us-east-1` — is exactly wrong here, and following it answers `400` on the create call. Content is stored in one region while serving is region-less, so the choice buys nothing anyway; omitting `region` gives the right answer. The constraint now sits with the other two static site rules, and the region heuristic points at it rather than contradicting it
- **`pending` means the site has no content yet, not that it is still provisioning:** the previous guidance — poll until `state` leaves `pending`, then deploy — describes a condition the caller itself has to cause. Nothing else moves a site out of `pending`, so an agent following it before the first deploy waits forever. Each runbook now names the field its source actually needs and the platform actually fills in: `bucket_name` for `zip` and `sftp`, `git_ssh_url` for `git_push`, nothing at all for `github`. After the first successful deploy, `state` reads the way you would expect
- **Cloning an application is a decision, not a parameter list:** every copy flag defaults to `false`, so a clone carries the source app's own settings and nothing else — no environment variables, no secret files, no add-ons, no cronjobs. More to the point, the deployment configuration is not copied either, which leaves the clone with no source to build from and its `deployment_method` fallen back to `git_push`. `use_parent_image` is the fast path, running the source's image with optional auto-sync; leaving it off means configuring a source afterwards. Add-on and cronjob UUIDs have to be collected from the source before the call, and `clone_data` copies a database's contents — worth confirming out loud, since that is rarely what "clone my app" means
- **A `commit_sha` is validated before the build starts, on a `github` app:** `POST /api/v1/apps/{uuid}/deploy` rejects a SHA that does not exist in the repository the app is configured with, answering `422` rather than starting a build that cannot check the commit out. Push first, and pass a SHA from that same repository — one from a fork, or from a branch that has since been squashed or force-pushed, will not resolve. The check is named for the one deployment method that performs it; the others take the SHA as given
- **`clone_security` is documented:** the flag copies allowed connections and Basic Auth from the source application. It was accepted before but appeared in neither the reference nor the API document, so there was no way to know it existed
- **SSH keys are manageable over the API for static sites too:** the SFTP runbook stated that a key "cannot be added over the API" while the same document listed `POST /api/v1/users/me/ssh_keys` further down. Both the SFTP and `git_push` runbooks now check `GET /api/v1/users/me/ssh_keys` and register a public key when the list is empty, with the obvious guards — never send a private key, and never generate one without telling the user

## 0.2.2 — 2026-07-28

Correct five things the skill got wrong or left unsaid, each of which sent an agent down the wrong path during a real deployment: refusing a valid free-plan layout on CPU grounds, duplicating a connection variable the platform had already set, reaching for the one deployment method that needs a dashboard visit, waiting forever on a state value that never arrives, and building a Git remote from the wrong name field.

- **CPU is never the capacity constraint:** the skill told agents to size a plan on its RAM *and CPU*, which led to refusing to put an app and its database on the same small resource because "the database will eat the CPU". Placement and capacity are checked against RAM and disk only. On dev plans `cpu_size` is a ceiling rather than a reservation, and the Fair Scheduler distributes the resource's CPU dynamically across everything running on it, so an idle process holds nothing back from a busy one. Sizing now reads on `ram_size` and `disk_size`, and the free plan carries a worked example: a 128 MiB app plus a Postgres addon at its 128 MiB default is exactly the plan's 256 MiB — tight, but valid, and the platform will create it
- **A database addon injects two variables, not one:** 0.2.0 stated that an addon adds exactly one variable and that "there is no `DATABASE_URL`". It adds two, both holding the same connection string — `<ADDON_NAME>_URL` and a generic alias, `DATABASE_URL` for `postgres` and `mysql` or `REDIS_URL` for `valkey`. The alias is created only when the app has no variable of that name yet, compared case-insensitively, and an existing one is never overwritten. A framework reading `DATABASE_URL` therefore works with no extra step, and adding a second one by hand duplicates what the platform already set
- **Choose the deployment method from the repository, not by default:** `git_push` was presented as the way to deploy, and its only documented flow ended with the user fetching a view-once token from the dashboard — even when the repository already had a GitHub remote. The method is now derived from `git remote -v`: a GitHub remote means `github`, the only method with auto-deploy on push and review apps for pull requests; another host means `public_git`; a prebuilt image means `container_registry`; and `git_push` is the fallback for code that lives nowhere else. Review apps are configured in the dashboard and only for `github` apps, so a repository that will want pull-request previews should start there
- **`git_push` authenticates over SSH first:** SSH keys are fully manageable through the API, so the whole flow can be completed without leaving the terminal — check `GET /api/v1/users/me/ssh_keys`, register a public key with `POST` if the list is empty, add the remote, push. Git tokens are not exposed by the API in any form and can be viewed exactly once, so the HTTPS route is documented as the fallback for cases where SSH will not work
- **`state` is two vocabularies, and `active` is not one of them:** apps, addons and services report the platform lifecycle while provisioning and the raw Kubernetes status once running, with nothing in the response marking which. A healthy Postgres, MySQL or Valkey reports `healthy`; storage reports `bound`; only an app reports `running` — which is also the fallback any of them returns when the status lookup comes back empty. An agent polling for `"active"` or `"running"` on a database therefore waits forever on something already up. The skill now carries the full value space per object type and tells agents to test set membership, with three traps called out: `degraded` is a replica catching up rather than a failure, stacks return a normalised state that does settle on `running`, and on a *deployment* `running` means still building while `completed` is the success
- **The suffixed `name` is the identifier everywhere:** the server appends a random suffix to the name you send, and the unsuffixed form is kept separately as `service_name`. Reading that as an invitation to use `service_name` in a Git remote produces `resource/wall.git` instead of `resource/wall-pfhqv.git` and a push against a repository that does not exist. `service_name` appears in exactly one place, `internal_url`; `label` addresses nothing; every other identifier — public URL, Git remote path, connection variable key — is built from `name` as returned. Both remote templates now name the fields they read and the SSH one carries a filled-in example

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

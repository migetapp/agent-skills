# Changelog

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

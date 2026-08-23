# Workspace, resources, projects and people

The container everything else sits in: the workspace and its bill, the compute
resources that cost money, the projects that partition them, and the permissions
that decide what a token may do.

Read this before creating a resource — that is the call that spends money — and
whenever a `403` needs explaining or a project turns out to be invisible rather
than forbidden.

## Contents

- [Workspaces and the bill](#workspaces)
- [Resources](#resources-migets)
- [Projects](#projects)
- [Endpoints](#endpoints)
- [Permissions](#permissions--authorization)
- [Users and SSH keys](#users)

## Workspaces

Miget uses **workspace-based multi-tenancy**. Each user can belong to multiple workspaces (organizations/teams).

- **Default workspace:** If no `X-Workspace-Id` header is provided, API uses the user's default workspace
- **Multi-workspace:** Include `X-Workspace-Id` header to specify which workspace to operate on:
  ```http
  X-Workspace-Id: {workspace-uuid}
  ```

## Stopping the bill

**None of this is on the API, and you must not try to approximate it by deleting things.** When a user wants to stop paying, downgrade, leave, or wind a workspace down, your job is to tell them exactly where to click and what will happen — then stop.

| What the user wants | Where they go | What it does |
|---|---|---|
| Stop paying entirely | **Billing** in the sidebar → **Cancel Subscription** on the subscription card | Charges stop now. Everything keeps running until the end of the paid period, then the resources and all their applications, services, buckets and databases are **permanently deleted**. Resumable from the same page until that date. |
| Stop paying, **enterprise workspace** | Their account manager, or support@miget.com | There is no cancel button on an enterprise plan and no endpoint behind it — notice period, final invoice and data handover follow the signed agreement. Billing shows an "Ending your contract" note instead. Do not tell an enterprise user to look for a Cancel button; they will not find one. |
| Drop one resource, keep the rest | **Resources** → pick it → **Settings** → **Delete** | Deletes that resource and everything on it, and takes it off the bill. The subscription continues for the workspace plan and any other resources. |
| Change plan | **Settings** → **Plan** | |
| Remove the workspace | **Settings** → **Delete** | Owner only — the tab is not shown to anyone else. Revokes the subscription immediately with no refund for the remaining period, and deletes every project and application. The last remaining workspace cannot be deleted. |
| Payment method or invoices | **Billing** → **Update Payment Methods** | Opens the payment provider's portal. |

Two things to warn about, because a user will not expect either:

- **Deleting applications does not reduce the bill.** Billing is per resource — the capacity applications run on — not per application. An empty resource costs exactly what a full one does. This is the single most common billing surprise, so say it before the user starts deleting apps to save money.
- **Cancelling ends in deletion, not in a frozen account.** A cancelled workspace is suspended and its workloads stopped, and the resources are then destroyed. Tell the user to export anything they want to keep — there is no undo once it happens.

A workspace can also be **suspended** by the platform — after a cancellation completes, for non-payment, or when an application is blocked for abuse. It is a consequence, never something a user or an agent triggers, and there is no endpoint to lift it.

While suspended, **every** API request for that workspace answers `403`, reads included, with one of two messages: `Your workspace has been suspended. Please contact support to reactivate it.` or, when an application was blocked, `...due to suspicious activity in one of your applications...`. Treat either as terminal — do not retry, do not switch endpoints, and do not report it as a permission problem the user can fix by changing a role. Relay which of the two it was, since they need different people: billing for the first, support for the second.

## Resources (Migets)

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

## Projects

**Projects** are logical groupings of apps and services.

- A project holds applications, static sites, services, stacks and buckets. Applications, static sites, services and stacks always belong to exactly one project; buckets may belong to one
- Ownership (which project a workload belongs to) and placement (which resource it runs on) are separate. Static sites sit on no resource at all
- **Ownership can be changed on every kind; placement cannot.** Send `project_id` to `PUT /apps/{uuid}`, `PUT /services/{id}`, `PUT /stacks/{uuid}`, `PUT /static_sites/{uuid}` or `PUT /buckets/{uuid}` to move a workload between projects — it keeps running on the same resource throughout. Moving a stack moves every application and service in it. There is no way to move a workload to a different resource; that requires deleting and recreating it
- Because the resource cannot change, a move is refused with **422** when the resource is assigned to projects that do not include the destination. The way through is to assign the resource to the destination project as well, not to pick a different resource
- A `project_id` that does not exist — or that exists but you cannot reach — is a **404** carrying `{"error": "Project not found"}`, on every one of those endpoints. Those two cases answer identically on purpose: a status code that told them apart would confirm that a restricted project is there. Read this 404 as "the destination could not be resolved", never as "the workload is gone" — the workload is untouched and still in its original project. Re-read `GET /api/v1/projects` to see which destinations you may actually use
- Projects can have **project-level environment variables** and **project secret files**, shared across the apps in the project. Neither is automatic: an app receives them only while its own `project_variables_enabled` / `project_files_enabled` toggle is on. Both are managed over the API — see "Project Environment Variables" and "Project Secret Files"
- Projects can have **assigned resources**, returned as `resources` on the project response. An empty list means the project simply uses the shared pool — which it may do even when it has assignments. The same relationship reads from the other side as `assigned` and `project_ids` on the resource response, which is the cheaper check when you are choosing a resource
- A project can also be **restricted** to specific people, returned as `restrictions` on the project response. An empty list means the project is open to the whole workspace; any entry closes it to everyone but the listed members, everyone holding a listed role, the workspace owner and holders of the **built-in `admin` role**. A custom role never bypasses a restriction, whatever permissions it carries — including `workspace:members`. Restricting projects is available on organization and enterprise workspaces only, and that plan refusal is checked **before** the subject you passed, so on a smaller plan you get the plan message even when the email is also wrong
- **A project you cannot reach simply does not appear.** It is absent from `GET /api/v1/projects`, and so are its applications, services, stacks and buckets in their own listings; addressing any of them by UUID returns **404**, not 403. So a short project list is not proof that the workspace holds nothing else — it is what you are allowed to see. The same is true of resources assigned solely to projects you cannot reach; an unassigned resource stays visible to everyone, because any project may deploy on it.

## Endpoints

## Resources

- `GET /api/v1/resources` - List all resources
- `POST /api/v1/resources` - Create new resource
- `GET /api/v1/resources/{uuid}` - Get resource details
- `PUT /api/v1/resources/{uuid}` - Update resource (change plan, add components)
- `PATCH /api/v1/resources/{uuid}/labels` - Update resource labels
- `DELETE /api/v1/resources/{uuid}` - Delete the resource and everything on it — applications, services, buckets, stacks — permanently. Takes that resource off the bill; **does not cancel the subscription**, which continues for the workspace plan and any other resources. There is no endpoint that cancels a subscription; see "Stopping the bill"

## Projects

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

## Project Environment Variables

- `GET /api/v1/projects/{project_id}/vars` - List project environment variables
- `POST /api/v1/projects/{project_id}/vars` - Create project environment variable
- `PUT /api/v1/projects/{project_id}/vars` - Update project environment variable
- `DELETE /api/v1/projects/{project_id}/vars` - Delete project environment variable

## Project Secret Files

Shared across every app in the project that has project secret files enabled. Same write-only rule as app secret files. **Creating one changes nothing for an app until that app's `project_files_enabled` toggle is on** — the parallel of `project_variables_enabled` for env vars.

- `GET /api/v1/projects/{project_id}/secret_files` - List project secret files (metadata only)
- `POST /api/v1/projects/{project_id}/secret_files` - Create one (body: `filename`, `text`). Returns the server-assigned `name`
- `GET /api/v1/projects/{project_id}/secret_files/{name}` - Get one file's metadata
- `PUT /api/v1/projects/{project_id}/secret_files/{name}` - Update `filename`, `text`, or both. At least one is required
- `DELETE /api/v1/projects/{project_id}/secret_files/{name}` - Remove it

## Plans, Regions & Components

- `GET /api/v1/plans` - List available plans. Each plan returns `code_name` (the opaque identifier you pass to `POST /api/v1/resources`), `plan_type` (`dev` for development/hobby, `pro` for production), `ram_size` and `disk_size` in **bytes**, `cpu_size` as a fractional core count, and `unit_price` in **cents**. Sizing note: `ram_size: 268435456` is 256 MiB, not 256 MB — compare it against `quota.ram_size` on apps, which uses the same unit.
- `GET /api/v1/regions` - List available regions. Current regions: `eu-east-1` (Warsaw), `us-east-1` (Vint Hill)
- `GET /api/v1/components` - List available resource components (extra_ram, extra_cpu, extra_disk)

## Users

- `GET /api/v1/users/me` - Get current user profile
- `GET /api/v1/users/me/credits` - Get credit balance, referral code, and credit operation history

## SSH Keys

- `GET /api/v1/users/me/ssh_keys` - List SSH keys
- `POST /api/v1/users/me/ssh_keys` - Add an SSH key
- `GET /api/v1/users/me/ssh_keys/{id}` - Get SSH key details
- `DELETE /api/v1/users/me/ssh_keys/{id}` - Remove an SSH key

## Authentication

- `POST /api/v1/auth/sign_in` - Authenticate with email/password
- `POST /api/v1/auth/refresh_token` - Refresh access token

## Create Resource (`POST /api/v1/resources`)

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

## Create Project (`POST /api/v1/projects`)

**Required fields:**
- `name` (string) - Project name (must be unique within workspace)

**Optional:**
- `description` (string) - Brief description of the project's purpose

**Ask only what you cannot derive:**
- "What should be the project name?"
- "Do you want to add a description?"

## Permissions & Authorization

Miget uses **role-based access control (RBAC)** within workspaces.

## Common Permissions

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

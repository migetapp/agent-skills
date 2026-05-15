---
name: miget-api
description: Deploy and manage apps, databases, buckets, and services on Miget PaaS. Covers authentication, resource provisioning, deployments, addons, domains, environment variables, and all API endpoints.
---

# Miget API - Guide for AI Agents

## Overview

Miget is a Kubernetes-based Platform-as-a-Service (PaaS) similar to Heroku or Render. It allows developers to deploy and manage applications, databases, and services in the cloud with minimal infrastructure management.

**Base URL:** `https://app.miget.com/api/v1`

**API Documentation:** `https://app.miget.com/api/v1/docs` (Swagger/OpenAPI)

---

## Authentication

Miget API supports two authentication methods:

### Method 1: Username/Password (JWT Tokens)

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

### Method 2: API Token (Recommended for Automation)

1. **Generate API token** in the web UI:
   - Go to `https://app.miget.com/my_account#api_tokens`
   - Create a new API token
   - Copy the token (starts with `miget_live_` prefix)

2. **Use API token** in requests:
   ```http
   Authorization: Bearer miget_live_xxxxxxxxxxxxx
   ```

   - API tokens don't expire (unless manually revoked)
   - Better for long-running automation and CI/CD

**Recommendation:** For AI agents and automation, use API tokens as they don't expire and provide better security tracking. Store the token in the `MIGET_API_TOKEN` environment variable so it can be detected automatically.

---

## Agent Behavioral Guidelines

These guidelines shape how you interact with the Miget API and with users. Read them before making any API calls.

### Before You Make Any API Call

These are non-negotiable - skipping them leads to failed requests or broken resources.

**1. Find the API token first.** All API calls require authentication. Follow this sequence:

   **Step 1: Check environment variables.**
   Look for `MIGET_API_TOKEN` in the user's shell environment (e.g., run `echo $MIGET_API_TOKEN`). If the variable is set and non-empty, use it as the `Authorization: Bearer` token for all API calls and skip ahead to guideline 2.

   **Step 2: If no token is found, ask the user.**
   Ask whether they already have a Miget account and an API token. Present these scenarios:

   - **"I have an API token"** - Ask them to provide it, then suggest storing it (see Step 4).
   - **"I have an account but no token"** - Guide them to generate one:
     1. Go to **https://app.miget.com/my_account#api_tokens**
     2. Click **"Create new token"**
     3. Give it a name (e.g., `cli-agent`)
     4. Copy the token (it starts with `miget_live_`)
     5. Share it back so you can proceed, then suggest storing it (see Step 4).
   - **"I don't have an account"** - Direct them to sign up first:
     1. Go to **https://app.miget.com/users/sign_up**
     2. Create an account and verify email
     3. Once signed in, generate an API token at **https://app.miget.com/my_account#api_tokens**
     4. Share the token back, then suggest storing it (see Step 4).

   **Step 3: Do not proceed without a valid token.** Attempting API calls without authentication wastes time and confuses the user with 401 errors.

   **Step 4: Suggest persisting the token in shell config.**
   After receiving a token, recommend the user store it so it's available automatically in future sessions:

   For **zsh** (default on macOS):
   ```bash
   echo 'export MIGET_API_TOKEN="miget_live_xxxxxxxxxxxxx"' >> ~/.zshrc
   source ~/.zshrc
   ```

   For **bash**:
   ```bash
   echo 'export MIGET_API_TOKEN="miget_live_xxxxxxxxxxxxx"' >> ~/.bashrc
   source ~/.bashrc
   ```

   For **fish**:
   ```fish
   set -Ux MIGET_API_TOKEN "miget_live_xxxxxxxxxxxxx"
   ```

   Once stored, the token will be detected automatically in Step 1 on every future session.

**2. Ask for all required fields before creating resources.** Guessing values for fields like region, plan, or project leads to failed deployments or unexpected costs. Before making any creation API call:
   - Ask the user explicitly for each required field
   - Explain what each field is for and why it's needed
   - If there are enum values (like plan types or deployment methods), list them so the user can choose
   - See the "Required Fields for Creation Endpoints" section for per-endpoint details

**3. Ask about database/cache deployment mode.** When a user asks to create a database (PostgreSQL, MySQL) or cache (Valkey), there are two fundamentally different approaches - creating the wrong one wastes time and may require starting over:
   - **Standalone Service** (`POST /api/v1/services`) - A standalone managed service that can be shared across multiple apps
   - **App Addon** (`POST /api/v1/apps/{uuid}/addons`) - Attached to a specific existing app, lifecycle tied to the app

   Ask which they prefer before proceeding. See "Creating Addons and Services" section for details.

**4. Understand which workspace you're in.** Miget uses workspace-based multi-tenancy. Include the `X-Workspace-Id` header when the user works with multiple workspaces. If omitted, the API uses the user's default workspace.

### General Best Practices

**5. Use the OpenAPI spec as a fallback.** If you encounter an endpoint or parameter not covered in this guide, consult the OpenAPI spec at `https://app.miget.com/docs/openapi.json` for the exact schema. You don't need to load it proactively - this guide covers the common cases.

**6. Handle async operations.** Deployments and resource provisioning are asynchronous. Poll the relevant status endpoint to track progress rather than assuming immediate completion.

**7. Handle errors gracefully.** Parse error responses and provide helpful feedback to the user:
   - `400` - Validation error, check required fields
   - `401` - Authentication failed, check token
   - `403` - Permission denied, check user permissions
   - `404` - Resource not found, verify IDs/UUIDs
   - `422` - Validation failed, check field values

**8. Validate before creating.** Before creating resources, verify that dependencies exist:
   - Verify project exists (if creating app)
   - Verify resource exists (if assigning to app)
   - Verify region exists (if creating resource)
   - Check if names are available (apps, projects must be unique within a workspace)

**9. Provide helpful follow-up.** After creating resources, confirm what was created, provide the UUID/ID, and suggest logical next steps (e.g., "App created! Would you like me to deploy it?").

**10. Store tokens securely.** Do not expose tokens in logs, error messages, or outputs shown to the user.

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

### Projects

**Projects** are logical groupings of apps and services.

- Projects can have **project-level environment variables** (shared across apps)
- Projects can have **project secret files** (shared files)

### Buckets

A **Bucket** is an S3-compatible object storage container.

- Buckets are attached to a **Resource** (Miget) for quota management
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

---

## API Structure

### Base Path

All endpoints are under: `/api/v1/`

### Common Headers

```http
Authorization: Bearer {token}
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

  Multiple validation errors (422 responses from creation/update endpoints):
  ```json
  {
    "errors": ["Field X is required", "Field Y is invalid"]
  }
  ```

  Parse both `error` (string) and `errors` (array) when handling error responses.

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
POST /api/v1/resources
{
  "plan_type": "dev",
  "plan_code_name": "starter",
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

# Update variable
PUT /api/v1/apps/{app-uuid}/vars/{var-uuid}
{
  "value": "new-value"
}

# Delete variable
DELETE /api/v1/apps/{app-uuid}/vars/{var-uuid}
```

### 4. Add a Database Addon

```http
# Create addon
POST /api/v1/apps/{app-uuid}/addons
{
  "type": "postgres"
}

# The addon will be automatically configured with environment variables
# like DATABASE_URL, DB_HOST, etc.
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
- Replica creation is asynchronous - poll the addon/service status to track provisioning
- Replicas can be promoted to standalone instances using the promote endpoint - this is irreversible and the promoted instance will no longer receive updates from the primary
- The `create_replica` endpoint returns the full serialized replica entity (same shape as `GET /api/v1/apps/{uuid}/addons/{id}` or `GET /api/v1/services/{id}`), including `uuid`, `role: "replica"`, `primary_addon_uuid`, and `connection_details` - no follow-up `GET` is required to discover the new replica

**Response fields for PostgreSQL addons/services:**
- `role` (string) - `"primary"` or `"replica"` (null for non-PostgreSQL)
- `primary_addon_uuid` (string) - UUID of the primary addon (only present on replicas)
- `replicas` (array) - List of replicas with `uuid`, `name`, `label`, `state` (only present on primaries, in show endpoints)

### 9. Manage App Ports (requires workspace_name and app_name)

App ports use a different URL pattern than other endpoints - they require `workspace_name` and `app_name` path segments in addition to the app UUID. This is because port routing operates at the Kubernetes namespace level, which maps to workspace and app names rather than UUIDs.

```http
# List all ports
GET /api/v1/apps/{app-uuid}/workspaces/{workspace-name}/apps/{app-name}/ports

# Create a new port
POST /api/v1/apps/{app-uuid}/workspaces/{workspace-name}/apps/{app-name}/ports
{
  "internal_port": 8080,
  "protocol": "tcp",
  "public": false
}

# Expose port publicly
PATCH /api/v1/apps/{app-uuid}/workspaces/{workspace-name}/apps/{app-name}/ports/{port-id}/expose_publicly

# Make port private
PATCH /api/v1/apps/{app-uuid}/workspaces/{workspace-name}/apps/{app-name}/ports/{port-id}/make_private

# Delete port
DELETE /api/v1/apps/{app-uuid}/workspaces/{workspace-name}/apps/{app-name}/ports/{port-id}
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

---

## Key Endpoints

### Authentication

- `POST /api/v1/auth/sign_in` - Authenticate with email/password
- `POST /api/v1/auth/refresh_token` - Refresh access token

### Applications

- `GET /api/v1/apps` - List all applications
- `POST /api/v1/apps` - Create new application
- `GET /api/v1/apps/{uuid}` - Get application details (includes `deployment_method` and `deployment_config` with method-specific fields; for Kamal apps this includes `registry_password`, `registry_hostname`, `registry_username`, `registry_image`, and `ssh_keys`)
- `PUT /api/v1/apps/{uuid}` - Update application
- `DELETE /api/v1/apps/{uuid}` - Delete application
- `PUT /api/v1/apps/{uuid}/security` - Update security settings (network connectivity, Basic Authentication)
- `PATCH /api/v1/apps/{uuid}/state` - Change app state (schedule_start/schedule_stop/schedule_restart)
- `POST /api/v1/apps/{uuid}/clone` - Clone an application (copy env vars, secret files, scaling, health checks, addons, cronjobs)
- `PUT /api/v1/apps/{uuid}/deployment` - Update deployment method and configuration (switch methods, update Kamal SSH keys)
- `POST /api/v1/apps/{uuid}/deploy` - Trigger deployment (optional: custom_tag, commit_sha, branch). Not used for Kamal apps.
- `PUT /api/v1/apps/{uuid}/health_checks` - Update health check probes (liveness, readiness, startup)
- `PUT /api/v1/apps/{uuid}/scaling_profile` - Update scaling profile (replicas, auto-scaling, thresholds). Not available on free plan.
- `GET /api/v1/apps/{uuid}/deployments` - List deployments
- `GET /api/v1/apps/{uuid}/deployments/{id}/logs` - Get build logs

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
- `PUT /api/v1/apps/{uuid}/vars/{var_uuid}` - Update variable
- `DELETE /api/v1/apps/{uuid}/vars/{var_uuid}` - Delete variable

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
- `PUT /api/v1/apps/{uuid}/cronjobs/{id}` - Update cronjob
- `DELETE /api/v1/apps/{uuid}/cronjobs/{id}` - Delete cronjob

### App Deployments

- `GET /api/v1/apps/{uuid}/deployments` - List deployments (optional filters: `status` (pending/running/completed/failed/cancelling/cancelled), `period` (7days/30days/90days/all))
- `GET /api/v1/apps/{uuid}/deployments/{id}` - Get deployment details
- `GET /api/v1/apps/{uuid}/deployments/{id}/logs` - Get build logs (text/plain, available after deployment completes)
- `GET /api/v1/apps/{uuid}/deployments/{id}/stream_logs` - Stream logs in real-time (SSE, text/event-stream, for running deployments)
- `POST /api/v1/apps/{uuid}/deployments/{id}/cancel` - Cancel running deployment (only deployments in 'running' state can be cancelled)
- `POST /api/v1/apps/{uuid}/deployments/{id}/rollback` - Rollback to a previous deployment (only rollbackable deployments can be rolled back - must have image URL, same deployment method as app, and not be running/failed)

### App Ports

Note: Port endpoints use a nested path that includes workspace and app names alongside the UUID. See workflow section 8 for details on why.

- `GET /api/v1/apps/{uuid}/workspaces/{workspace_name}/apps/{app_name}/ports` - List all ports for an app
- `POST /api/v1/apps/{uuid}/workspaces/{workspace_name}/apps/{app_name}/ports` - Create a new port (requires apps:manage, not available on free plan)
- `GET /api/v1/apps/{uuid}/workspaces/{workspace_name}/apps/{app_name}/ports/{id}` - Get port details
- `DELETE /api/v1/apps/{uuid}/workspaces/{workspace_name}/apps/{app_name}/ports/{id}` - Delete a port
- `PATCH /api/v1/apps/{uuid}/workspaces/{workspace_name}/apps/{app_name}/ports/{id}/expose_publicly` - Expose a port publicly (requires apps:manage, not available on free plan)
- `PATCH /api/v1/apps/{uuid}/workspaces/{workspace_name}/apps/{app_name}/ports/{id}/make_private` - Make a port private (requires apps:manage)

### Buckets (S3-Compatible Object Storage)

- `GET /api/v1/buckets` - List all buckets
- `POST /api/v1/buckets` - Create new bucket
- `GET /api/v1/buckets/{uuid}` - Get bucket details (includes S3 endpoint, S3 credentials, usage stats)
- `PUT /api/v1/buckets/{uuid}` - Update bucket (label, visibility, disk size)
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
- `PUT /api/v1/services/{id}` - Update service (PostgreSQL supports `instances` [1, 3, 5, 7] for scaling cluster nodes)
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

### Plans, Regions & Components

- `GET /api/v1/plans` - List available plans (plan types: `dev`, `pro`)
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

---

## Permissions & Authorization

Miget uses **role-based access control (RBAC)** within workspaces.

### Common Permissions

- `apps:view` - View applications
- `apps:create` - Create applications
- `apps:manage` - Manage applications (update, delete)
- `apps:deploy` - Deploy applications
- `apps:operate` - Operate applications (start/stop, manage addons)
- `resource:view` - View resources
- `resource:manage` - Manage resources
- `projects:view` - View projects
- `projects:manage` - Manage projects
- `buckets:view` - View buckets and list objects
- `buckets:operate` - Operate buckets (upload, download, manage policy/ACL)
- `buckets:manage` - Create and delete buckets
- `workspace:settings` - Manage workspace settings

Workspace owners have all permissions automatically.

If you get a `403 Forbidden` error, the user doesn't have the required permission for that operation. Let them know which permission is needed so they can request it from a workspace admin.

---

## Required Fields for Creation Endpoints

When creating resources, ask the user for all required fields before making the API call. Guessing values leads to failed requests or resources created in the wrong region/plan, which then need to be deleted and recreated.

### Create Application (`POST /api/v1/apps`)

**Required fields:**
- `name` (string) - Unique service name (lowercase, alphanumeric with hyphens, used in URLs)
- `label` (string) - Human-readable display name
- `project_id` (string) - UUID of the project to create the application in (get from `GET /api/v1/projects`)
- `resource_id` (string) - UUID of the compute resource (Miget) to assign (get from `GET /api/v1/resources`). The app's region is derived from this resource.
- `builder` (string) - Build strategy: `"auto"`, `"dockerfile"`, or `"custom"`

**Optional but important:**
- `ram_size` (float) - RAM allocation in MiB
- `cpu_size` (float) - CPU allocation in cores
- `deployment_method` (string) - `"git_push"`, `"public_git"`, `"github"`, `"container_registry"`, `"parent_image"`, or `"kamal"`. Note: the enum value is `container_registry` (not `docker_registry`).
- `deployment_config` (object) - Configuration specific to the chosen deployment method (see table below)
- `app_vars_attributes` (array) - Environment variables to set at creation (array of `{key, value}` objects)

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

**Helping the user deploy a `git_push` app.** The API does not expose the Git token value, so after the app is created you must walk the user through pushing from their machine:

1. **Get a token.** Direct the user to their app's Git Tokens settings page to view (or create) a token:

   `https://app.miget.com/apps/{APP_UUID}/settings#git_tokens`

   - For the auto-created default token, the Git **username** is the resource (miget) name and the **password** is the token value.
   - For any additional token the user creates, the **username** is the token's name and the **password** is the token value.
   - A token can only be viewed once — if it says "Token already seen", the user must create a new one on that page.

2. **Build the remote URL.** The remote is region-specific: `https://git.{region_code}.miget.io/{miget_name}/{app_name}` (for example, `https://git.eu-east-1.miget.io/my-resource/my-app`). You can read `region.code`, `miget.name`, and `name` from `GET /api/v1/apps/{uuid}`.

3. **Guide the user through the push.**

   New repository:
   ```bash
   git init
   git config push.autoSetupRemote true
   git remote add miget https://git.{region_code}.miget.io/{miget_name}/{app_name}
   git add .
   git commit -a -m "initial"
   git push miget
   ```

   Existing repository:
   ```bash
   git remote add miget https://git.{region_code}.miget.io/{miget_name}/{app_name}
   git push miget
   ```

   When git prompts for credentials, use the username/password from step 1. The push triggers a build and deploy — monitor it via `GET /api/v1/apps/{uuid}/deployments` (see workflow 2).

**`public_git`** - Deploy from a public Git repository URL.

```json
{
  "deployment_method": "public_git",
  "deployment_config": {
    "credential_id": "{git-credential-uuid}",
    "repository": "https://github.com/user/repo.git",
    "branch": "main",
    "dockerfile_path": "./Dockerfile",
    "build_context": "."
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `repository` | string | Yes | - | Full Git repository URL |
| `branch` | string | No | `"main"` | Branch to deploy from |
| `credential_id` | string | No | - | UUID of Git credentials (for private repos) |
| `dockerfile_path` | string | No | `"./Dockerfile"` | Path to Dockerfile |
| `build_context` | string | No | `"."` | Docker build context directory |

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
| `image_url` | string | Yes | - | Full container image URL (e.g., `docker.io/library/nginx`) |
| `tag` | string | No | `"latest"` | Container image tag |
| `credential_id` | string | No | - | UUID of container registry credentials (for private registries) |
| `command` | array of strings | No | - | Override the image's `ENTRYPOINT` (Kubernetes `container.command`). Leave unset to use the image default. |
| `args` | array of strings | No | - | Override the image's `CMD` (Kubernetes `container.args`). Leave unset to use the image default. Use this when an image's ENTRYPOINT is set but no CMD is supplied (e.g. Keycloak prints help on bare run; pass `["start"]` to start the server). |

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
*   **Optional:**
    *   `label` (string): A human-readable display name for the addon.
    *   `ram_size` (float): RAM allocation in MiB (e.g., 64, 128, 256).
    *   `disk_size` (float): Disk storage in GiB (e.g., 1, 5, 10).
    *   `cpu_size` (float): CPU allocation in cores (e.g., 0.1, 0.25, 0.5).

---

#### Addon Type: `postgres`

A PostgreSQL database addon. Supports two creation modes: fresh database or external replica.

*   **Type-specific Parameters (Optional):**
    *   `postgres_version` (string): The major version of PostgreSQL (e.g., `'15'`, `'16'`, `'17'`).
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

> "What version of PostgreSQL would you like? (e.g., 15, 16, 17)"
> "Should this database be accessible from the public internet? (yes/no)"
> "Do you want a standalone instance or a High Availability ha_cluster? (standalone/cluster)"
> "Do you want to create a fresh database or replicate from an external PostgreSQL? (fresh/external_replica)"

---

#### Addon Type: `mysql`

A MySQL database addon.

*   **Type-specific Parameters (Optional):**
    *   `mysql_version` (string): The major version of MySQL (e.g., `'8.0'`, `'8.4'`).

**Example questions:**

> "What version of MySQL would you like? (e.g., 8.0, 8.4)"

---

#### Addon Type: `valkey`

A Valkey (Redis-compatible) cache addon.

*   **Type-specific Parameters (Optional):**
    *   `valkey_version` (string): The version of Valkey (e.g., `'7.2'`).

**Example questions:**

> "What version of Valkey would you like? (e.g., 7.2)"

---

#### Addon Type: `storage`

A persistent storage volume addon.

*   **Type-specific Parameters (Optional):**
    *   `mount_point` (string): The path inside the container where the volume should be mounted (e.g., `/data`).
    *   `storage_access` (string): The access mode. Must be one of `RWO` (ReadWriteOnce) or `RWX` (ReadWriteMany).
    *   `service_id` (integer): To attach an existing shared storage service, provide its ID.

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
    *   `resource_id` (string): The UUID of the compute resource where the service will be provisioned. (Legacy alias: `miget_id` — deprecated, still accepted for backward compatibility.)
    *   `label` (string): A human-readable display name for the service.
*   **Optional:**
    *   `ram_size` (float): RAM allocation in MiB.
    *   `disk_size` (float): Disk storage in GiB.
    *   `cpu_size` (float): CPU allocation in cores.

---

#### Service Type: `postgres`

A standalone PostgreSQL database service. Supports two creation modes: fresh database or external replica.

*   **Type-specific Parameters (Optional):**
    *   `postgres_version` (string): The major version of PostgreSQL (e.g., `'15'`, `'16'`, `'17'`).
    *   `public_access` (string): Enable public internet access. Use `'1'` for enabled, `'0'` for disabled.
    *   `environment_variables` (boolean): If `true`, automatically injects connection variables into the parent application.
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
> "What version of PostgreSQL would you like? (e.g., 15, 16, 17)"
> "Should this database be publicly accessible? (yes/no)"
> "Do you want a standalone instance or a High Availability ha_cluster? (standalone/cluster)"
> "Do you want to create a fresh database or replicate from an external PostgreSQL? (fresh/external_replica)"

---

#### Service Type: `shared_storage`

A standalone shared storage volume service.

*   **Type-specific Parameters (Optional):**
    *   `mount_point` (string): The default mount path (e.g., `/shared-data`).
    *   `storage_access` (string): The access mode. Must be one of `RWO` or `RWX`.

**Example questions:**

> "Which project should this service belong to? (Please provide the Project UUID)"
> "Which compute resource (Miget) should I provision this on? (Please provide the Miget UUID)"
> "What access mode do you need? `RWO` (for a single app) or `RWX` (for multiple apps)?"

---

### Create Bucket (`POST /api/v1/buckets`)

**Required fields:**
- `label` (string) - Human-readable display name for the bucket
- `resource_id` (string) - UUID of the compute resource to attach the bucket to (get from `GET /api/v1/resources`). Legacy alias `miget_id` is still accepted but deprecated.

**Optional but important:**
- `visibility` (string) - Bucket visibility: `"public_access"` or `"private_access"` (default: `"private_access"`)
- `disk_size` (float) - Disk allocation in GiB (default: 0.1)

**Example questions to ask:**
- "What should be the bucket's display name?"
- "Which resource (Miget) should the bucket be attached to? (provide resource ID)"
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
- `plan_type` (string) - Plan category: `"dev"` for development/hobby, `"pro"` for production
- `plan_code_name` (string) - Specific plan identifier (e.g., `"free"`, `"starter"`, `"professional"`)
- `region_code` (string) - Deployment region code. Available: `eu-east-1` (Warsaw), `us-east-1` (Vint Hill)

**Optional:**
- `components` (array) - Additional resource components (extra RAM, CPU, disk)

**Example questions to ask:**
- "What plan type? (dev for development, pro for production)"
- "What plan code name? (e.g., free, starter, professional)"
- "Which region? (eu-east-1, us-east-1)"
- "Do you want to add any extra components? (RAM, CPU, disk)"

### Create Project (`POST /api/v1/projects`)

**Required fields:**
- `name` (string) - Project name (must be unique within workspace)

**Optional:**
- `description` (string) - Brief description of the project's purpose

**Example questions to ask:**
- "What should be the project name?"
- "Do you want to add a description?"

### Create App Domain (`POST /api/v1/apps/{uuid}/domains`)

**Required fields:**
- `domain.name` (string) - Fully qualified domain name (e.g., `"app.example.com"`)

**Example questions to ask:**
- "What domain name should I add? (e.g., app.example.com)"

### Create App Environment Variable (`POST /api/v1/apps/{uuid}/vars`)

**Required fields:**
- `key` (string) - Variable name (use SCREAMING_SNAKE_CASE, e.g., `DATABASE_URL`)
- `value` (string) - Variable value

**Example questions to ask:**
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

**Example questions to ask:**
- "What should be the cronjob name?"
- "What display label should I use?"
- "What schedule type? (cron for custom expression, interval for predefined)"
- "If interval, which interval? (every_10_minutes/hourly/daily)"
- "If cron, what cron expression? (e.g., 0 * * * * for hourly)"
- "If daily, what time? (HH:MM format)"
- "What command should be executed?"

### Create App Port (`POST /api/v1/apps/{uuid}/workspaces/{workspace_name}/apps/{app_name}/ports`)

**Required fields:**
- `workspace_name` (string) - Workspace name
- `app_name` (string) - Application name
- `internal_port` (integer) - Internal port number (1-65535)
- `protocol` (string) - Protocol: `"tcp"` or `"udp"`

**Optional but important:**
- `public` (boolean) - Whether the port should be publicly accessible (default: `false`)

**Important notes:**
- Requires `apps:manage` permission
- Port management is not available on free plan resources
- Ports can be exposed publicly or made private after creation using separate endpoints
- Port `5000` is fixed for HTTP traffic on the app's `*.migetapp.com` URL — it is auto-created, cannot be removed or changed, and the app must listen on it. Use this endpoint to add extra TCP/UDP ports for custom protocols; they are **private by default** — use `expose_publicly` to make them reachable from outside the cluster. See https://docs.miget.com/networking/ports for the full list of supported ports.

**Example questions to ask:**
- "What workspace name?"
- "What application name?"
- "What internal port number? (1-65535)"
- "What protocol? (tcp or udp)"
- "Should this port be publicly accessible? (true/false)"

### Rollback Deployment (`POST /api/v1/apps/{uuid}/deployments/{id}/rollback`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Deployment UUID

**Constraints:**
- Deployment must be rollbackable (has image URL, same deployment method as app, not running/failed)
- Application must be in deployable state
- Requires `apps:deploy` permission

**Example questions to ask:**
- "Which deployment should I rollback to? (provide deployment UUID)"

### Rotate Addon Password (`POST /api/v1/apps/{uuid}/addons/{id}/rotate_password`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Addon UUID

**Constraints:**
- Password rotation only supported for certain addon types (databases like PostgreSQL, MySQL, and caches like Valkey)
- Requires `apps:operate` permission
- New password will be returned in response - you must update ENV variables manually

**Example questions to ask:**
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

**Example questions to ask:**
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

**Example questions to ask:**
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

**Example questions to ask:**
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

**Example questions to ask:**
- "Which PostgreSQL service should I create a replica for? (provide service ID)"
- "What CPU and RAM should the replica use? (defaults to primary's values)"

### Promote Replica to Standalone - Service (`POST /api/v1/services/{id}/promote_replica`)

**Required fields:**
- `id` (string, path) - Service ID (must be a PostgreSQL replica service)

**Constraints:**
- Service must be PostgreSQL type
- Service must be a replica (not a primary)
- Requires `services:operate` permission

**Example questions to ask:**
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

**Example questions to ask:**
- "Which external replica service should I promote to a standalone instance? (provide service ID)"

### Update Security Settings (`PUT /api/v1/apps/{uuid}/security`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- At least one of the following optional fields must be provided

**Optional fields:**
- `allow_connections` (boolean) - Allow internal network connections from other Miget applications in this workspace
- `basic_auth_enabled` (boolean) - Enable Basic Authentication for the application
- `basic_auth_username` (string) - Username for Basic Authentication (required when `basic_auth_enabled` is true)
- `basic_auth_password` (string) - Password for Basic Authentication (required when `basic_auth_enabled` is true, leave blank to keep current password)

**Constraints:**
- Requires `apps:manage` permission
- When `basic_auth_enabled` is true, both `basic_auth_username` and `basic_auth_password` are required (unless password already exists and you want to keep it)

**Example questions to ask:**
- "Should I enable Basic Authentication? (true/false)"
- "If enabling Basic Auth, what username should I use?"
- "If enabling Basic Auth, what password should I use? (leave blank to keep current)"
- "Should I allow internal network connections? (true/false)"

### Clone Application (`POST /api/v1/apps/{uuid}/clone`)

**Required fields:**
- `uuid` (string, path) - Source application UUID to clone from
- `label` (string) - Display name for the cloned application
- `name` (string) - Unique service name for the cloned application (lowercase, alphanumeric with hyphens)
- `project_id` (string) - UUID of the project
- `resource_id` (string) - UUID of the compute resource (Miget)

**Optional fields:**
- `use_parent_image` (boolean, default: false) - Use source app as parent image
- `parent_image_auto_sync` (boolean, default: false) - Auto-deploy when parent image updates
- `clone_variables` (boolean, default: false) - Copy environment variables
- `clone_secret_files` (boolean, default: false) - Copy secret files
- `clone_scaling_settings` (boolean, default: false) - Copy auto-scaling config
- `clone_health_checks` (boolean, default: false) - Copy health check config
- `addons` (array, default: []) - Addons to clone, each: `{uuid: String, clone_data: Boolean}`
- `cronjobs` (array, default: []) - Cronjob UUIDs to clone

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
- `within_resources` (boolean) - Limit scaling to available resource allocation

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
- `label` (string) - Display label for the mounted addon

### Unmount App from Service (`POST /api/v1/services/{id}/unmount_app`)

**Required fields:**
- `id` (string, path) - Service UUID
- `app_id` (string) - UUID of the application to unmount

---

## Best Practices

1. **Use API Tokens for Automation**
   - API tokens don't expire and are better for CI/CD and automation
   - Generate tokens at `https://app.miget.com/my_account#api_tokens`

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
     - If the user is hitting a custom TCP/UDP port directly: list ports via `GET /api/v1/apps/{uuid}/workspaces/{workspace_name}/apps/{app_name}/ports` and confirm the port exists and is public. Extra ports are **private by default** — expose them with `expose_publicly`.
   - Only after ports look right, check deployment status, domains, and logs.

---

## Example: Complete Application Setup (GitHub)

```http
# 1. Authenticate
POST /api/v1/auth/sign_in
{
  "email": "user@example.com",
  "password": "password"
}
# Save access_token

# 2. Get or create resource
GET /api/v1/resources
# If none exists:
POST /api/v1/resources
{
  "plan_type": "dev",
  "plan_code_name": "starter",
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

# 6. Add database addon
POST /api/v1/apps/{app-uuid}/addons
{
  "type": "postgres"
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

# Static sites

Prebuilt HTML, CSS and JavaScript served from object storage. A static site is
**not an application**: it has its own endpoints under `/api/v1/static`, no
compute resource, no replicas, no ports and no environment variables, and it
costs nothing to run.

Read this before reaching for `/api/v1/apps` with something that is only files.
Three things differ from applications in ways that bite silently — the name takes
no suffix, rollback rebuilds rather than redeploys, and `zip` and `sftp` sources
carry no commit.

## Static Sites

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

## Endpoints

## Static Sites

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

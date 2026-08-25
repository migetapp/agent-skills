# Webhooks

Workspace-level outbound webhooks. Miget POSTs a signed JSON payload to your
endpoint when a subscribed event happens, so a consumer can react to deploys
without polling.

Read this before building any polling loop over `GET /api/v1/apps/{uuid}/deployments`,
and before telling a customer their webhook is broken — the two defaults that
surprise people are that an empty `app_uuids` means *every* application, and that
preview environments are excluded unless asked for.

## Webhooks

Workspace-level outbound webhooks. Miget POSTs a signed JSON payload to your endpoint when a subscribed event occurs, so you can react to deploys without polling `GET /api/v1/apps/{uuid}/deployments`. Requires `workspace:general` (admin only).

- `GET /api/v1/webhooks` - List webhooks
- `POST /api/v1/webhooks` - Create a webhook (**the only response containing `secret`**)
- `GET /api/v1/webhooks/{uuid}` - Get a webhook
- `PUT /api/v1/webhooks/{uuid}` - Update a webhook (send only the fields you want changed)
- `DELETE /api/v1/webhooks/{uuid}` - Delete a webhook
- `POST /api/v1/webhooks/{uuid}/test` - Send a test event and get the result back immediately

**Testing an endpoint.** `POST /api/v1/webhooks/{uuid}/test` POSTs a signed event to the URL right away and returns the resulting delivery, so you can verify an endpoint without waiting for a deployment. It is signed exactly like a real delivery, so it also exercises your signature verification. It responds `200` whether or not the endpoint accepted it — read `status` and `response_code` on the returned delivery. It is not retried, works on a disabled webhook, and is recorded in the delivery history.

**Handle `type: "ping"`.** The test event carries `"type": "ping"` with an empty `data` object. `ping` is *not* one of the subscribable event types, so a consumer that rejects unknown types will fail the test even though the endpoint is otherwise fine. Either accept `ping` explicitly or ignore unknown types.

**Create fields:** `name` (unique per workspace), `url` (http or https), `event_filter` (array), optional `app_uuids` (array), `enabled` (default `true`) and `include_review_apps` (default `false`).

**The endpoint must be publicly reachable.** A `url` pointing into private address space — loopback, RFC1918, link-local (including `169.254.169.254`), CGNAT, or the `localhost`, `.local` and `.internal` hostnames — is rejected with `422`. The host is re-resolved before every delivery, so a name that later starts resolving to a private address stops being delivered to rather than being retried.

**Scoping to specific apps.** By default a webhook receives events from **every app in the workspace**, including apps created later. Pass `app_uuids` to narrow it to specific apps; send an empty array to widen it back. Apps must belong to the same workspace or the request is rejected with `422`. Either way the payload carries `data.app_id`, so a consumer can filter on its own as well.

**Event types:**

| Event | Fires when |
|---|---|
| `deploy_started` | A deployment enters `running` |
| `deploy_ended` | A deployment reaches `completed`, `failed`, or `cancelled` |
| `app_unhealthy` | An application fails a liveness, readiness, or startup check |
| `app_crash_loop` | An application is restarting repeatedly and not recovering |
| `app_stopped` | The platform stopped an application after repeated crashes |
| `app_state_changed` | Somebody started, stopped, or restarted an application |
| `app_blocked` | An application was blocked for a security issue |
| `scaling_limit_reached` | An application could not scale to the replicas it asked for |
| `certificate_expiring` | An SSL certificate is 30, 7, or 1 day from expiring without renewing |
| `domain_verified` | Domain verification finished, whether it verified or failed |
| `quota_alert` | Observability ingestion crossed a warning or critical threshold |

There is no separate build event — on Miget the build and the deployment are one lifecycle, and `build_id` *is* the deployment UUID.

**Preview environments are separate apps.** A preview environment is created as its own application with its own UUID, so a webhook narrowed with `app_uuids` does not receive its events even when the parent application is listed. Set `include_review_apps: true` to have those events matched against the parent as well. A webhook covering every app in the workspace already receives them, which is worth knowing before opening a busy repository's pull requests.

`quota_alert` belongs to the workspace rather than to any one app, so it is delivered to every subscribed webhook regardless of `app_uuids`. Every other event carries `data.app_id` and respects app scoping.

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
    "message": null,
    "commit_sha": "abc123",
    "commit_message": "Fix the thing",
    "branch": "main",
    "created_at": "2026-08-05T11:58:00Z"
  }
}
```

`message` carries why a deployment failed, in the platform's own wording rather than the raw cluster error where one is recognised. It is `null` on a deployment that did not fail.

**App lifecycle events name the resource.** `app_state_changed`, `app_unhealthy`, `app_crash_loop`, `app_stopped`, `app_blocked` and `scaling_limit_reached` carry `data.resource_name` and `data.labels` — the resource the application runs on and the labels set on it, an empty array when it has none. `app_state_changed` also carries `data.state`, one of `started`, `stopped`, `failed` or `restart_scheduled`; `restart_scheduled` is sent when a restart is requested, and the `started` that follows once it is running is a second event.

**Verifying a delivery.** Requests follow the [Standard Webhooks](https://standardwebhooks.com) specification, so any Standard Webhooks library verifies them as-is. Three headers accompany every POST:

| Header | Value |
|---|---|
| `webhook-id` | The event UUID (also `id` in the body) |
| `webhook-timestamp` | Unix seconds at signing time |
| `webhook-signature` | `v1,<base64>` |

The signature is `HMAC-SHA256(key, "{webhook-id}.{webhook-timestamp}.{body}")`, base64-encoded, where `key` is the base64-decoded portion of the secret after the `whsec_` prefix. Compare with a constant-time comparison, and reject deliveries whose `webhook-timestamp` is more than a few minutes old — that is what makes a captured request non-replayable.

**Retries.** A non-2xx response is retried with exponential backoff over roughly **24 hours** — 9 attempts in total, spaced 30s, 2m, 8m, 30m, 1h, 3h, 8h and 12h apart with jitter. Endpoints should be idempotent: deduplicate on `id`, which is stable across retries of the same event *and* across manual replays. Three consecutive events that exhaust every attempt disable the webhook and email the workspace; re-enable it with `PATCH /api/v1/webhooks/{uuid}` and `enabled: true` once the endpoint is back.

**Delivery history.**

- `GET /api/v1/webhooks/{uuid}/deliveries` — the recent history, newest first. One entry per event, updated in place across retries; `attempts` is how many times it has been tried, `status` is `pending`, `delivered` or `failed`, `response_code` and `message` hold what the endpoint returned (`message` is truncated to 500 characters, and carries the transport error when no response arrived). `payload` is the exact JSON body that was POSTed, so you can compare what you received against what was sent. Only the last 50 events per webhook are kept.
- `POST /api/v1/webhooks/{uuid}/deliveries/{delivery_uuid}/retry` — replay a delivery from its stored payload, reusing the original event id. Returns the delivery with status `pending`; poll the list for the outcome. `422` if the webhook is disabled.

Use these to diagnose a webhook that appears silent: a `failed` entry with `response_code: 502` is a problem on your side, while no entries at all means no subscribed event has fired.

---

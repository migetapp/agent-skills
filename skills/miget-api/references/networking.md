# Private networks (VPC) and VPN

A **VPC** is a private network a workspace's applications, services and add-ons
join to reach each other by name, with nothing outside able to reach them. The
**VPN** half is how something outside the platform — a laptop, an office, an AWS
account — gets in.

Read this before the first `POST /api/v1/vpcs`. Two things about a new VPC are
not guessable and cost a wasted call each: it comes back `pending` with its
addresses still `null`, and it already owns a subnet. Read the VPN section before
enabling a gateway — one of the four spends $199/mo and takes a public address
that the region has a small, finite number of.

## Contents

- [What a VPC is](#what-a-vpc-is)
- [Endpoints](#endpoints)
- [The two surprises](#the-two-surprises)
- [Walking a customer through a private network](#walking-a-customer-through-a-private-network)
- [Reaching a VPC from outside: the four terminators](#reaching-a-vpc-from-outside-the-four-terminators)
- [VPN endpoints](#vpn-endpoints)
- [Peer routes](#peer-routes)
- [Site-to-site tunnels](#site-to-site-tunnels)
- [Billing](#billing)

## What a VPC is

- A VPC belongs to **one region** and cannot span regions. A workspace working in
  two regions gets one VPC in each, and only workloads running on a resource in
  that region can attach.
- The range defaults to `10.224.0.0/16`. It must be RFC1918, `/22` or larger, and
  must not overlap the platform's own networks — the create call refuses with the
  specific conflict rather than failing later.
- Two workspaces may sit on the same range: VPCs are separate routers and never
  see each other. The consequence is that two VPCs on the same range can never be
  peered, so pick your own range if you plan to connect them.
- Subnets are carved out of the VPC. The only thing that distinguishes them is
  `public`: public keeps the platform default route, private moves it to the VPC
  so egress leaves through the workspace's own gateway.
- **Attaching restarts the workload.** An interface cannot be added to a running
  pod, so for an add-on this is a database restart. Say so before you do it.
- An attached workload answers to `fqdn`. That name resolves to a **pod address**,
  not a fixed one, and the record refreshes on the next attach, detach or route
  change — treat it as a name to reach, never as a stable address. A cron job gets
  no `fqdn` at all, because its pods are transient.
- `resolver_v4` is the VPC's own DNS resolver. Point an external resolver at it to
  resolve `migetapp.internal` names from outside the platform.

## Endpoints

VPCs use their own `network:*` permissions: read = `network:view`, attach/detach =
`network:operate`, everything else = `network:manage`. Site-to-site connections
are the exception and need the workspace **owner**.

- `GET /api/v1/vpcs` - List the workspace's private networks
- `POST /api/v1/vpcs` - Create one. `region_id` is required; `name` defaults to `default` and `cidr_v4` to `10.224.0.0/16`. A range that is not RFC1918, is smaller than `/22`, or overlaps a platform network is refused with **422** naming the conflict
- `GET /api/v1/vpcs/{uuid}` - Get one, including `resolver_v4` and `vpn_pool_v4`
- `PUT /api/v1/vpcs/{uuid}` - Change `label` or `is_default`. The name and the ranges are fixed at creation, because the platform derives the whole address plan from them on every action. Making this the region's default while another VPC holds it is **422** — clear that one first
- `DELETE /api/v1/vpcs/{uuid}` - Delete it. **422** while any subnet remains — remove those first
- `GET /api/v1/vpcs/{uuid}/subnets` - List subnets
- `POST /api/v1/vpcs/{uuid}/subnets` - Create one (`name`, plus `cidr_v4` unless `family` is `ipv6`). Optional `family` — `dual` (default), `ipv4` for no IPv6 at all, `ipv6` for no IPv4 — and `public`, default `true`. `cidr_v6` is derived from the VPC's own when omitted. The range must sit inside the VPC's own and must not overlap a sibling
- `DELETE /api/v1/vpcs/{uuid}/subnets/{subnet_uuid}` - Delete a subnet. **422** while a workload is still attached to it
- `GET /api/v1/vpcs/{uuid}/attachments` - List attached workloads with the `fqdn` each answers to
- `POST /api/v1/vpcs/{uuid}/attachments` - Attach a workload (`subnet_uuid`, `attachable_type` one of `App`/`Service`/`Addon`, `attachable_uuid`). **The workload restarts** — for an add-on that is a database restart. **422** when the workload runs in a different region from the VPC
- `DELETE /api/v1/vpcs/{uuid}/attachments/{attachment_uuid}` - Detach. The workload restarts again

An application can also join at creation, with `vpc_uuid` or `vpc_subnet_uuid` on
`POST /api/v1/apps` — worth preferring, because it skips the restart that
attaching afterwards costs.

## The two surprises

**A new VPC comes back `pending`.** `POST /api/v1/vpcs` answers immediately with
`status: "pending"` and `cidr_v6`, `resolver_v4` and `vpn_pool_v4` all `null` —
the platform derives those and answers a moment later. Poll
`GET /api/v1/vpcs/{uuid}` until `status` is `active`, then read them.

**It already owns a subnet.** That same reply carries a ready-made subnet named
`default`. It is not the first `/24` — the platform keeps the start of the range for
the router, the resolver and the VPN terminators — so on a `10.224.0.0/16` VPC the
default subnet is `10.224.1.0/24`. List the subnets before creating
one: attaching a workload usually needs no subnet call at all, and carving
`10.224.1.0/24` by hand is refused with `422` for overlapping the subnet you
already have.

## Walking a customer through a private network

1. **Ask what should stop being public**, not whether they want a VPC. Usually the
   answer is a database, and the fix is one attach rather than a whole network
   design.
2. **Create the VPC**, or find the region's existing default. Only argue about the
   range if they plan to peer it with something — otherwise the default is right,
   and a range they picked to be "safe" is the thing that later makes peering
   impossible.
3. **Wait for `active`**, then list subnets and use the `default` one.
4. **Attach the database first, then the application.** Warn before each: both
   restart, and for the database that is real downtime. If the application is not
   created yet, pass `vpc_uuid` on create instead and skip its restart entirely.
5. **Verify from inside**, not from the outside. Read `fqdn` off the attachment
   and have the application reach the database by that name. Tell them plainly it
   is a name, not an address — it moves on the next rollout.

### What to suggest next

- **`private: true` on the application's subnet**, if egress should leave through
  their own gateway rather than the platform's.
- **WireGuard**, the moment somebody says "I need to connect to the database from
  my laptop" — that is the whole reason the VPN half exists.
- **Their own resolver pointed at `resolver_v4`**, if they want `migetapp.internal`
  names to resolve on their office network.

## Reaching a VPC from outside: the four terminators

| Terminator | What it is for | Costs |
|---|---|---|
| **WireGuard** | Individual people and laptops. The platform books the address and port and issues one config per device | Nothing scarce — shares one address across ports |
| **Tailscale** | A workspace already running a tailnet; the VPC joins it as a node | Nothing scarce — dials out |
| **Cloudflare WARP** | A workspace already on Cloudflare Zero Trust | Nothing scarce — dials out |
| **IPsec (site-to-site)** | A whole remote network, e.g. an AWS VPC | **One public IPv4 per tunnel, $199/mo** |

WireGuard, Tailscale and WARP are enabled directly. IPsec is not: it comes up with
the first site-to-site connection, which is what pays for its address.

## VPN endpoints

- `GET /api/v1/vpcs/{uuid}/vpn_gateways` - List the terminators enabled on the VPC
- `POST /api/v1/vpcs/{uuid}/vpn_gateways` - Enable one. `kind` is `wireguard`, `tailscale` or `cloudflare`; one of each per VPC. Cloudflare needs `organization`, `client_id` and `client_secret`; Tailscale needs `auth_key` and takes optional `tag` and `accept_routes`. Those credentials are stored encrypted and **never returned by any endpoint**
- `DELETE /api/v1/vpcs/{uuid}/vpn_gateways/{gateway_uuid}` - Disable it, revoking every device on it. **422** while site-to-site connections still terminate on it
- `GET /api/v1/vpcs/{uuid}/vpn_users` - List the WireGuard devices, with the address each answers to
- `POST /api/v1/vpcs/{uuid}/vpn_users` - Register a device. `name` is lowercase letters, numbers and hyphens. **422** if WireGuard is not enabled first
- `GET /api/v1/vpcs/{uuid}/vpn_users/{user_uuid}/config` - The device's WireGuard config file. **Readable exactly once** — see below
- `DELETE /api/v1/vpcs/{uuid}/vpn_users/{user_uuid}` - Revoke the device

**The device config works once and cannot be recovered.** The platform generates
the keypair, keeps only the public half, and holds the private key in a 15-minute
vault that this read empties. So:

1. `POST .../vpn_users` returns the device `pending`, with no address and no config.
2. Poll it until `status` is `active` — the address and the server's key only
   arrive when the platform has registered the peer.
3. `GET .../config` **once**, and write the file straight to disk or hand it
   straight to the person. A second call answers `410`, and so does a call made
   before the device is active, or more than 15 minutes after it was created.
4. A device whose config expired unread is not recoverable. Delete it and add it
   again.

Tell the customer this before you create the device, not after they lose the file.

## Peer routes

Only **Cloudflare WARP and Tailscale** need explicit routes: WireGuard's client
pool and an IPsec connection's remote selectors are derived by the platform from
the gateways themselves.

- `GET /api/v1/vpcs/{uuid}/peer_routes` - List staged and applied routes
- `POST /api/v1/vpcs/{uuid}/peer_routes` - Stage one (`cidr`, `gateway_kind` of `cloudflare` or `tailscale`, optional `description`). Staging changes nothing yet
- `POST /api/v1/vpcs/{uuid}/peer_routes/apply` - Send every staged route at once. **This restarts every workload attached to the VPC.** **422** when nothing is staged
- `DELETE /api/v1/vpcs/{uuid}/peer_routes/{route_uuid}` - Remove one. A staged route goes outright; an applied one is withdrawn from the cluster, which restarts the workloads again

Routes are staged rather than applied one at a time precisely because applying is
disruptive. Add every range the customer needs, then apply once, and say what it
will cost before you do.

## Site-to-site tunnels

- `GET /api/v1/vpcs/{uuid}/site_connections` - List the tunnels, each with the public address it holds
- `POST /api/v1/vpcs/{uuid}/site_connections` - Create one. `name`, `remote_cidrs`, and `tunnels` (each `remote_gateway` plus `psk` from the far side's configuration); optional `peer_asn`
- `DELETE /api/v1/vpcs/{uuid}/site_connections/{connection_uuid}` - Tear it down. This stops the $199/mo charge but does **not** release the public address — that is held by the IPsec gateway, which goes only with the VPC

Three things to say out loud before calling this:

- It costs **$199/mo** for as long as the tunnel exists, because IPsec needs fixed
  UDP ports and so one public IPv4 carries exactly one peer.
- Only the **workspace owner** may call it. Anyone else gets `403`.
- A region has a small pool of these addresses. When it is empty the answer is
  `422`, not a queue.

Read `public_ip` off the response — that is the address the customer enters on
their side, and the tunnel does not come up until they do. The pre-shared keys are
stored encrypted and never returned.

## Billing

The VPC itself is included on paid plans. The VPN is the add-on:

- **$29/mo per enabled gateway.** Counted per gateway, not per VPC — a VPC running
  both WireGuard and Tailscale is charged twice, as are two VPCs each running
  WireGuard.
- **$199/mo per site-to-site tunnel**, counted from the tunnels that exist.

Enterprise has both included. Deleting a tunnel stops its charge; the public
address it held is only returned to the region's pool when the VPC is deleted.

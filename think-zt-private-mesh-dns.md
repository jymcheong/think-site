# Scratchpad: Private Mesh Network + DNS — Building a Zero-Trust Lab on ZeroTier

**Date:** 2026-07-28
**Status:** Raw material — not edited

---

## Core Idea

Most people run ZeroTier as a VPN convenience layer. You can go further: self-host the controller, automate membership via API, bolt on private DNS — and end up with a proper private network where nodes find each other by name, the controller is yours, and nothing leaks to the public internet.

---

## What We Actually Built / Did Today

### 1. Dual-daemon ZeroTier on one VPS
- contabo1 runs TWO zerotier-one daemons simultaneously
- Host daemon (port 29993): contabo1 itself as a mesh CLIENT
- Docker container `zu-controller` (port 9993): the CONTROLLER for all managed networks
- They coexist — host shifted to 29993 because container claimed 9993 first
- `sudo zerotier-cli` talks to host daemon only, NOT the controller

### 2. Controller API is local, not cloud
- ZeroTier Central (`my.zerotier.com`) = cloud-managed, costs money, someone else holds the keys
- Local API at `zerotier:9993` inside the container = you own it entirely
- Auth via `X-ZT1-Auth` header (API token, not user auth)
- BusyBox wget inside container does GET/POST fine but no DELETE — use `nc` for raw HTTP

### 3. Full node lifecycle via API
Live-tested today:
```
create network (with IP pool + route) 
  → register node (authorized=false, access blocked)
  → approve node (authorized=true, IP assigned)
  → ping between nodes: 0% loss
```
- claudeVM: node `699df46196`, got `10.200.1.46`
- deepseekVM: node `27ff79e36d`, got `10.200.1.163`

### 4. DNS isolation — the clean way
- dnsmasq on contabo1, bound ONLY to ZT IP (`listen-address` + `bind-interfaces`)
- Port 53 socket never opens on eth0 / public IP
- ZT IPs (`10.246.231.x`) have no public route — unreachable even if socket existed
- Two structural layers, no firewall rules needed
- Members query `hostname.qlab.zt` → resolves to ZT IP → stays inside mesh

---

## Angles for the Article

### Angle A: "Why self-host your ZeroTier controller"
- You own the keys, no vendor lock-in
- API-driven automation (no web UI clicks)
- Can tie into Telegram bot / onboarding flow
- Controller goes down → existing peers still communicate (ZT is P2P after handshake)

### Angle B: "The two-layer DNS isolation trick"
- bind-interfaces vs firewall rules (structural > policy)
- Why split-DNS is the right model (not replacing public resolver, just augmenting for private domain)
- dnsmasq as the right tool (2MB, one config line per host, no overkill)
- Pattern: join network → get IP → get name → reachable by all peers immediately

### Angle C: "Building a private agent mesh"
- Multiple VMs / bots that form a private network
- Names instead of IPs for inter-bot communication
- Controller as the authority: only approved nodes get in
- Extend: onboarding flow (HMAC token → curl | bash → auto-approve → DNS record)

### Angle D: "Things that surprised us"
- curl was BROKEN in the container (SSL symbol not found) — BusyBox nc saved it
- `sudo zerotier-cli` hits the WRONG daemon (host client, not controller) — common trap
- `bind-interfaces` is NOT optional — without it dnsmasq still opens 0.0.0.0
- ZT assigns IPs only after node syncs — slight delay after approve

---

## Quotable Moments / Lines

- "The controller is just an API. The mesh is the network."
- "bind-interfaces: the line that means your DNS server doesn't exist to the outside world."
- "Two zerotier-one processes, same box, different ports, different jobs."

---

## Technical Facts to Keep Accurate

- Local API base: `http://zerotier:9993` (docker service name inside compose network)
- Auth header: `X-ZT1-Auth: <token>` — token from `/var/lib/zerotier-one/authtoken.secret`
- Network creation: POST to `/controller/network/<nodeID>______` (6 underscores = random suffix)
- Member approval: POST to `/controller/network/<nwid>/member/<node_id>` with `{"authorized":true}`
- ZT Central vs local: Central uses `Authorization: Bearer <token>`, local uses `X-ZT1-Auth`
- dnsmasq critical config: `listen-address=<ZT-IP>` AND `bind-interfaces` (both required)

---

## Related / Link to

- zerotier-control-plane-automation (onboarding flow — natural next post)
- jym.sg AI attack surface article (agentic systems need private comms infrastructure)

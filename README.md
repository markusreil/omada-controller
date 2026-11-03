# Omada Controller

TP-Link Omada SDN controller in Docker, wired as a downstream service of the
shared nginx reverse proxy (LAN / self-signed variant).

Upstream image: `mbentley/omada-controller` (see `omada/README.md`). No custom
build — the upstream image *is* the service.

## Architecture

- `omada-controller` — Omada Software Controller (management UI + device
  adoption). Management UI is reached through the proxy at
  `omada.<BASE_DOMAIN>`; device discovery/adoption ports are published
  directly on the host (see below).
- Volumes: `omada-data`, `omada-logs`, `omada-work` (named volumes; state
  survives restarts and rebuilds).
- Network: joins the proxy's external network (`NGINX_PROXY_NETWORK`,
  default `web-proxy`), which must exist before startup.

## Why `ports:` exist here (rule 4 exception)

Rule 4 says no host port publishing — everything through the proxy. That
covers browser traffic: the management UI is proxy-only (proxy terminates
client TLS and talks to the backend over HTTPS, `VIRTUAL_PORT: 8043` +
`VIRTUAL_PROTO: https` — required because the controller 302-redirects its
plain-HTTP endpoint to its own HTTPS port, which would otherwise bounce
browsers out to `:8043` with the controller's TP-Link self-signed cert).

But devices talk to the controller directly, and no HTTP reverse proxy can
carry that traffic. These ports are published so APs/switches can reach
the controller:

| Port | Purpose | Who needs it |
| --- | --- | --- |
| 8088/tcp | Inform + portal HTTP | Every device (inform URL), guest portal |
| 8043/tcp | Management HTTPS, doubles as firmware download for non-ES families (EAPs etc. — tracks `MANAGE_HTTPS_PORT`) | Browsers (direct hits show the controller's own cert — use the proxy instead), EAP firmware upgrades |
| 8843/tcp | Portal HTTPS | Guest portal |
| 8044/tcp | Dedicated HTTPS firmware download for ES-series switches (`UPGRADE_ES_HTTPS_PORT`) | Switch firmware upgrades |
| 29810/udp | Discovery | Device discovery |
| 29811–29817/tcp | Management, adopt, upgrade, remote terminal | Adoption and management |
| 27001/udp | App discovery | Omada app |

Only the management UI goes through the proxy; everything in this table
stays direct. If WAN exposure worries you, restrict these ports to LAN
subnets at the host firewall instead of unpublishing them — removing any
one of them breaks adoption or firmware upgrades for some device family.

## Firmware upgrades

Devices download firmware **from the controller**, not from the internet:
the controller fetches the image from TP-Link cloud and each device pulls
it over HTTPS. Per-family ports apply (see table: ES-series → 8044, all
other families → 8043), so a failure that hits only APs or only switches
points at the corresponding port. After any recreate or version bump, wait
3–5 minutes for the controller (Java + MongoDB) to fully start before
attempting upgrades — the upgrade endpoints only come online at the end of
startup.

## Quickstart

```sh
cp env.example .env
# edit .env (BASE_DOMAIN, PUID/PGID, TZ, OMADA_VERSION)
docker compose config   # fails fast on missing vars
docker compose up -d
docker compose ps
```

Start the proxy cluster first so the external `web-proxy` network exists.

Open `https://omada.<BASE_DOMAIN>` (self-signed certificate on LAN) and
follow the controller's setup wizard. Default credentials are set on first
run via the wizard, not via env vars.

## Operational notes

- Adopt devices by pointing them at the Docker host on the LAN (L3
  adoption): `set inform url http://<host>:8088/inform` via SSH, where
  `<host>` is the LAN IP or a LAN-resolvable hostname (e.g.
  `omada.<BASE_DOMAIN>`). The published device ports must be reachable
  from the devices; if adoption times out with "device does not respond
  to adopt commands", the return path (server → device) is broken —
  check gateway/VLAN/firewall, then factory-reset and re-adopt.
- Match the site's Device Account (site view → Settings) to your devices'
  logins before adopting — matching credentials adopt with no prompt;
  a wrong entry at the prompt is cached, so fix it on the device or
  delete the Pending entry and re-discover to get the prompt back.
- Upgrade: bump `OMADA_VERSION` in `.env`, then
  `docker compose pull && docker compose up -d`. Named volumes keep state.
- Logs: `docker compose logs omada-controller`.
- Reset (reseed): `docker compose down -v` removes all controller state
  (devices, settings), then `up -d` starts fresh.

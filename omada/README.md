# omada (upstream image, no custom build)

This service uses the upstream `mbentley/omada-controller` image directly —
there is no `Dockerfile`, `docker/` context, or `entrypoint.sh` here.

## Why upstream

Per compose spec rule 6, an upstream image is acceptable when it *is* the
service. The Omada Controller is a closed-source TP-Link application; there
is nothing to build from source. `mbentley/omada-controller` is the
community-standard containerization of it (multi-arch, version-tagged,
handles PUID/PGID and port env vars internally).

## Why no entrypoint

Per rules 8–9, a custom entrypoint is only needed for first-run seeding in
custom-built images. The upstream image already seeds its own state on first
start and manages its own user/privileges via `PUID`/`PGID`. Adding another
entrypoint layer would only add failure modes, so there is none — and
therefore no `su-exec` privilege-drop logic of ours either.

## Environment variables

| Variable | Required | Purpose |
| --- | --- | --- |
| `OMADA_VERSION` | yes | Image tag (e.g. `5.14`, `latest`). |
| `PUID` / `PGID` | yes | IDs the controller process runs as. |
| `TZ` | yes | Timezone (e.g. `Europe/Berlin`). |
| `MANAGE_HTTP_PORT` | yes | Management UI HTTP port (default `8088`). |
| `MANAGE_HTTPS_PORT` | yes | Management UI HTTPS port (default `8043`). |
| `PORTAL_HTTP_PORT` | yes | Captive portal HTTP port. |
| `PORTAL_HTTPS_PORT` | yes | Captive portal HTTPS port (`8843`). |
| `SHOW_SERVER_LOGS` | no | Verbose controller logs (`true`/`false`). |
| `SHOW_MONGODB_LOGS` | no | Verbose MongoDB logs (`true`/`false`). |
| `OMADA_GEN_SELF_SIGNED_CERT` | no | Self-signed TLS opt-in for the proxy (`GEN_SELF_SIGNED_CERT`), default `true` in compose (LAN/self-signed deployment). |

Proxy vars (`VIRTUAL_HOST`, `VIRTUAL_PORT`, `ACME_HOST`) are wired in
`docker-compose.yml` and need no per-deployment tuning beyond `BASE_DOMAIN`.

## Security notes

- No secrets in env: initial admin credentials are created via the setup
  wizard on first run, not via environment variables.
- The controller runs as `PUID`/`PGID`, not root, per the upstream image.
- Device adoption ports are LAN-reachable by design; do not expose this host
  to the internet without firewalling everything except the proxied UI.

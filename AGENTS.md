# AGENTS.md

Working rules for this repo (summary of the compose spec as it applies here).

- Config: everything deployment-specific lives in `.env` (never tracked);
  `env.example` is the tracked source of truth. Keep both in the same
  order/shape (globals, then one `# --- <service> ---` section per service),
  every variable with its own comment. Setup: `cp env.example .env`.
- Validation: every required var uses `${VAR:?...}` so
  `docker compose config` fails fast; optional vars use `${VAR:-default}`.
  Never add silent defaults for required config.
- Profiles: `COMPOSE_PROFILES` is steered via `.env`, never via `--profile`.
- Proxy: management UI is proxy-only — no `ports:` for it. Each proxied
  service declares its complete contract (`VIRTUAL_HOST` from the `x-hosts`
  anchor, `VIRTUAL_PORT` where needed, `ACME_HOST` plus `GEN_SELF_SIGNED_CERT`
  wired from `.env` defaulting to `true` for HTTPS). Exception: Omada device
  adoption ports (raw TCP/UDP) are published directly — see README.md.
  The proxy network is external (`NGINX_PROXY_NETWORK`, default `web-proxy`);
  start the proxy cluster first. No per-service `ACME_EMAIL` override.
- Hostnames: anchored once in `x-hosts`, derived from `BASE_DOMAIN`; never
  copy-paste literal hostnames.
- Images: upstream `mbentley/omada-controller` *is* the service (no custom
  build, no entrypoint — see `omada/README.md` for why).
- Volumes: named volumes for state (`omada-data`, `omada-logs`,
  `omada-work`); no bind mounts for stateful data.
- No `container_name:` — rely on default naming. `restart: unless-stopped`.
- Homepage labels on the service (`group`, `name`, `icon`, `href`,
  `description`).
- Changelog: Keep a Changelog, `## [Unreleased]` filled in as changes are
  made (newest first, ISO dates); never create a release section unprompted.
- Verify: `docker compose config`, then build/up and check the UI.
  Details in `README.md` and `omada/README.md`.

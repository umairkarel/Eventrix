# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Server-side GTM (Google Tag Manager) and GA4 tracking infrastructure. Proxies analytics requests through a first-party domain to bypass ad blockers, with multi-tenant support via Traefik routing and HTTPS-first local development using self-signed certs.

## Services

| Service | Directory | Tech | Purpose |
|---------|-----------|------|---------|
| Proxy Server | `proxy-server/` | Python / FastAPI | Proxies GA4 `/g/collect` requests, injects real client IP |
| GTM Server | `gtm-server/` | Docker / Google Cloud Tagging image | Runs GTM container + preview server |
| Traefik | `gtm-server/traefik/` | Traefik v3.6.5 | Multi-tenant routing, TLS termination |
| NGINX | `nginx/` | NGINX | Legacy proxy with JS domain rewriting via `sub_filter` |
| WordPress | `wordpress/` | WordPress 6 + MySQL 8 | Test environment for GTM integration |

## Running Services

**Proxy Server** (FastAPI on port 8000):
```bash
cd proxy-server
uv run uvicorn server:app --host 0.0.0.0 --port 8000 --reload
```

**GTM + Preview Server** (with specific client env):
```bash
cd gtm-server
docker compose --env-file ./client_envs/client1.env up -d
```

**Traefik** (must start before GTM server):
```bash
cd gtm-server/traefik
docker compose up -d
```

**WordPress** (port 8080):
```bash
cd wordpress
docker compose up -d
```

**NGINX** (ports 80/443):
```bash
cd nginx
docker compose up -d
```

## Architecture: Data Flow

```
Browser → NGINX/Traefik (TLS termination)
       → FastAPI Proxy (extracts real IP, logs, forwards)
       → google-analytics.com
```

GTM container runs via the official `gcr.io/cloud-tagging-10302018/gtm-cloud-image` image. `CONTAINER_CONFIG` is a Base64-encoded GTM workspace config passed as an env var. `PREVIEW_SERVER_URL` must point to the HTTPS preview endpoint for GTM preview mode to work.

## Multi-Tenant Setup

Client configs live in `gtm-server/client_envs/<clientname>.env`. Each file sets:
- `CLIENT_NAME` — used as Traefik routing label and subdomain prefix
- `BASE_DOMAIN` — root domain for routing rules
- `CONTAINER_CONFIG` — Base64 GTM config for this client
- `PREVIEW_SERVER_URL` — HTTPS URL to preview server

Traefik discovers containers via Docker labels and routes by hostname based on these vars.

## Key Files

- [proxy-server/server.py](proxy-server/server.py) — Core proxy logic (IP extraction, forwarding, CORS)
- [proxy-server/pyproject.toml](proxy-server/pyproject.toml) — Python deps (FastAPI, httpx, uvicorn)
- [gtm-server/docker-compose.yml](gtm-server/docker-compose.yml) — GTM + preview + NGINX SSL sidecar
- [gtm-server/traefik/config/traefik.yml](gtm-server/traefik/config/traefik.yml) — Traefik static config (entrypoints, Docker provider)
- [nginx/conf.d/default.conf](nginx/conf.d/default.conf) — NGINX routing with `sub_filter` JS rewriting
- [Notes/proxy.md](Notes/proxy.md) — Proxy implementation reference (Node.js, Cloudflare Workers, NGINX examples)
- [idea.md](idea.md) — Project roadmap and phase breakdown

## TLS / Certs

Self-signed certificates are stored in `gtm-server/certs/`. NGINX uses them for the preview server HTTPS sidecar. Traefik uses its own TLS config under `gtm-server/traefik/config/`.

## Notes

- Traefik connects to Docker via TCP on port 2375 — Docker must expose this.
- The `traefik_gtm` Docker network must exist before starting the GTM server stack; Traefik creates it.
- NGINX `sub_filter` rewrites GTM JS to replace Google's domain with your proxy domain — see `nginx/conf.d/default.conf` for the rewrite rules.

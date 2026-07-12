# Self-Hosting Jackpoll

Jackpoll runs as a small set of containers: the **frontend** (Next.js) and
**backend** (Quarkus) images from GHCR, plus **Postgres**, **MinIO** (file
uploads), **Keycloak** (auth), **Redis** (rate limiting) and **Mailpit** (dev
email). This guide covers two deployment paths:

- **[Path A — Single host (`docker compose`)](#path-a--single-host-docker-compose)** —
  one machine, published ports, no TLS proxy. Best for trying Jackpoll, small
  internal instances, or a box behind your own reverse proxy.
- **[Path B — Docker Swarm + Traefik](#path-b--docker-swarm--traefik)** —
  multi-replica, automatic TLS, external secrets. Best for a public production
  deployment.

## Prerequisites

- Docker Engine 24+ with the Compose v2 plugin (`docker compose version`).
- ~2 GB RAM free (4 GB recommended), ~5 GB disk.
- For a public deployment: a domain and DNS pointing at your host.

---

## Path A — Single host (`docker compose`)

### 1. Clone

```bash
git clone https://github.com/jackpoll-org/jackpoll-selfhost.git
cd jackpoll-selfhost
```

### 2. Configure secrets

```bash
cp .env.simple.example .env
```

Open `.env` and replace **every** `CHANGE_ME` with a strong, unique value:

| Variable | What it is | How to generate |
|---|---|---|
| `DB_PASSWORD` | Postgres password | `openssl rand -base64 24` |
| `MINIO_SECRET_KEY` | Object-storage secret key | `openssl rand -base64 24` |
| `KEYCLOAK_CLIENT_SECRET` | OIDC client secret (backend ↔ Keycloak) | `openssl rand -base64 32` |
| `KEYCLOAK_ADMIN_PASSWORD` | Keycloak admin console password | `openssl rand -base64 24` |
| `SURVEY_SPAM_SECRET` | HMAC secret for spam/bot protection | `openssl rand -base64 32` |

The backend **refuses to boot** if any of these are left unset — there is no
production fallback.

### 3. Start

```bash
docker compose -f docker-compose.simple.yml up -d
```

First boot pulls images and imports the Keycloak realm; give it a minute. Watch
progress with:

```bash
docker compose -f docker-compose.simple.yml logs -f backend
```

The backend is healthy once `/q/health/ready` returns `UP` (the frontend waits
for this automatically).

### 4. Open

| Service | URL | Notes |
|---|---|---|
| **App** | http://localhost:3000 | The Jackpoll UI |
| Backend API | http://localhost:8080 | Health at `/q/health` |
| Keycloak admin | http://localhost:8180 | Login `admin` / your `KEYCLOAK_ADMIN_PASSWORD` |
| MinIO console | http://localhost:9001 | Login `minioadmin` / your `MINIO_SECRET_KEY` |
| Mailpit (email) | http://localhost:8025 | Catches verification / reset emails |

Register a user in the app — the verification email appears in Mailpit.

### 5. Going public (own domain)

The simple stack serves plain HTTP. To expose it safely:

1. Put a reverse proxy with TLS (Caddy, Nginx, or Traefik) in front of the
   frontend (`:3000`) and Keycloak (`:8180`).
2. In `.env` set the public URLs so redirects and CORS line up:
   ```env
   APP_BASE_URL=https://app.example.com
   CORS_ORIGINS=https://app.example.com
   KEYCLOAK_HOSTNAME=https://auth.example.com
   ```
3. Point a real SMTP server via `MAIL_HOST` / `MAIL_PORT` / `MAIL_USERNAME` /
   `MAIL_PASSWORD` / `MAIL_TLS` and drop the Mailpit dependency.

For a hands-off TLS + multi-node setup, use **Path B** instead.

---

## Path B — Docker Swarm + Traefik

Use `docker-compose.swarm.yml`. It publishes **no host ports** — routing and TLS
are handled by an existing [Traefik](https://traefik.io/) on a shared external
`traefik` network (via the `le-prod` cert resolver). Secrets are **external
Swarm secrets**, never stored in the repo.

### 1. One-time host setup

```bash
docker swarm init                 # on the manager node
docker network create --driver overlay --attachable traefik   # if not present
```

You need a running Traefik instance attached to that `traefik` network with a
cert resolver named `le-prod` (or edit the labels in the stack file to match
yours).

### 2. Create the external secrets

Create these four Swarm secrets (Portainer → Swarm → Secrets, or the CLI). The
names must match exactly:

```bash
printf '%s' 'STRONG_DB_PASSWORD'        | docker secret create survey_db_password -
printf '%s' 'STRONG_MINIO_SECRET'       | docker secret create survey_minio_secret -
printf '%s' 'STRONG_KEYCLOAK_SECRET'    | docker secret create survey_keycloak_client_secret -
printf '%s' 'STRONG_SPAM_SECRET'        | docker secret create survey_spam_secret -
```

### 3. Configure non-secret variables

```bash
cp .env.swarm.example .env.swarm
```

Edit `.env.swarm` — set your hostnames (`FRONTEND_HOST`, `KEYCLOAK_HOST`,
`API_HOST`), image tags, replica counts, and any optional features (ClamAV,
push, Web Push). `docker stack deploy` ignores `env_file`, so the variables must
be loaded into your shell before deploying (next step).

### 4. Deploy

```bash
set -a; . ./.env.swarm; set +a          # load variables into the shell
docker stack deploy -c docker-compose.swarm.yml survey
```

Check rollout:

```bash
docker stack services survey
```

### Notes for Path B

- **Realm changes:** Swarm configs are immutable. When
  `keycloak/survey-school-realm.json` changes, bump the config key
  (`keycloak_realm_v2` → `_v3`) in the stack file, or the deploy fails with
  *"only updates to Labels are allowed"*.
- **WebSockets:** the stack exposes `/collab-ws` (live co-editing) and
  `/results-ws` (live results) as dedicated Traefik routers at priority 100 so
  they aren't shadowed by the frontend catch-all. Keep those labels intact.
- **Stateful services** (Postgres, MinIO) are pinned to the manager node — keep
  the manager stable or move to named volumes on shared storage.

---

## Upgrading

Both paths run tagged GHCR images. To update:

```bash
# Path A
docker compose -f docker-compose.simple.yml pull
docker compose -f docker-compose.simple.yml up -d

# Path B
docker service update --image ghcr.io/jackpoll-org/backend:<tag>  survey_backend
docker service update --image ghcr.io/jackpoll-org/frontend:<tag> survey_frontend
```

Pin `FRONTEND_IMAGE` / `BACKEND_IMAGE` to a release tag (not `:latest`) for
reproducible deploys. Database schema migrations run automatically on backend
startup (Flyway).

## Backups

- **Postgres** — the `pgdata` volume holds all surveys, responses and users:
  ```bash
  docker compose -f docker-compose.simple.yml exec postgres \
    pg_dump -U survey survey > backup-$(date +%F).sql
  ```
- **MinIO** — the `miniodata` volume holds uploaded images/files. Back up the
  volume or mirror the bucket with `mc mirror`.

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Backend restarts, logs `refuse to boot` | A required secret is unset in `.env` — fill every `CHANGE_ME`. |
| Login redirects fail / "invalid redirect" | `KEYCLOAK_HOSTNAME` doesn't match the URL the browser uses. Set it to the public Keycloak URL. |
| CORS errors in the browser console | `CORS_ORIGINS` must equal the exact origin serving the app (scheme + host + port). |
| No verification email | Path A: check Mailpit at `:8025`. Prod: verify `MAIL_*` SMTP settings. |
| Frontend never starts | It waits for the backend healthcheck — check `docker compose ... logs backend`. |
| Keycloak won't import realm | Ensure `keycloak/survey-school-realm.json` exists and the DB is healthy first. |

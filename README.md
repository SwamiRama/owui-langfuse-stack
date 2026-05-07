# owui-langfuse-stack

Production-grade `docker-compose` template for a self-hosted
**Open WebUI + Langfuse v3 + Caddy** proof-of-concept.

Pinned versions, healthchecks, memory limits, security defaults,
auto-HTTPS, optional OAuth/SSO. One server, one `docker compose up -d`.

> **Status:** POC template. Battle-tested patterns, but **not** a
> drop-in production deployment — see [Disclaimer](#disclaimer).

---

## What's in the box

```
                    ┌──────────────────────┐
                    │   Caddy (80/443)     │  auto-HTTPS via Let's Encrypt
                    └──────┬────────┬──────┘
                           │        │
                  chat.…   │        │   langfuse.…
                           ▼        ▼
                    ┌──────────┐  ┌────────────┐
                    │ OpenWebUI│  │Langfuse Web│
                    └────┬─────┘  └──────┬─────┘
                         │               │
                  ┌──────▼─────┐         │
                  │ Pipelines  │  (LLM-as-a-judge,
                  │ (filters)  │   token cost, etc.)
                  └────────────┘         │
                                         │
              ┌────────────┬─────────────┼──────────────┐
              ▼            ▼             ▼              ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐  ┌────────────┐
        │ Postgres │ │ClickHouse│ │  Redis   │  │   MinIO    │
        │  (txn)   │ │  (OLAP)  │ │ (queue)  │  │ (S3 blobs) │
        └──────────┘ └──────────┘ └──────────┘  └────────────┘
                              ▲
                              │
                        Langfuse Worker
                        (async ingest)
```

| Component | Image | Purpose |
|---|---|---|
| Caddy | `caddy:2.11-alpine` | Reverse proxy + auto-HTTPS |
| Open WebUI | `ghcr.io/open-webui/open-webui:v0.9.2` | Chat frontend |
| Pipelines | `ghcr.io/open-webui/pipelines:main@sha256:b48e…` | Plugin host (digest-pinned) |
| Langfuse Web | `langfuse/langfuse:3.172.1` | Tracing UI + API |
| Langfuse Worker | `langfuse/langfuse-worker:3.172.1` | Async event processing |
| PostgreSQL | `postgres:17-alpine` | Langfuse transactional DB |
| ClickHouse | `clickhouse/clickhouse-server:24.12-alpine` | Langfuse OLAP traces |
| Redis | `redis:7-alpine` | Langfuse queue + cache |
| MinIO | `minio/minio:RELEASE.2025-09-07…` | S3-compatible blob store |

---

## Quickstart

```bash
git clone https://github.com/SwamiRama/owui-langfuse-stack.git
cd owui-langfuse-stack

# 1) Generate secrets and prepare .env
cp .env.example .env
# edit .env — see "Generating Secrets" below

# 2) Point DNS A-records of DOMAIN_OWUI and DOMAIN_LANGFUSE at this server

# 3) Validate compose syntax
docker compose config

# 4) Start it
docker compose up -d

# 5) Watch Langfuse boot (initial ClickHouse migrations take ~2 min)
docker compose logs -f langfuse-web   # wait for "Ready"
```

Once Caddy has issued certificates (≈10 s after first hit), open:

- `https://${DOMAIN_OWUI}` — Open WebUI (sign up → admin approves)
- `https://${DOMAIN_LANGFUSE}` — Langfuse (login with the headless-init credentials)

---

## Generating Secrets

The stack needs seven 256-bit hex secrets and three strong passwords.
Generate everything in one shot:

```bash
{
  echo "WEBUI_SECRET_KEY=$(openssl rand -hex 32)"
  echo "NEXTAUTH_SECRET=$(openssl rand -hex 32)"
  echo "LANGFUSE_SALT=$(openssl rand -hex 32)"
  echo "ENCRYPTION_KEY=$(openssl rand -hex 32)"
  echo "PIPELINES_API_KEY=$(openssl rand -hex 32)"
  echo "REDIS_AUTH=$(openssl rand -hex 32)"
  echo "MINIO_ROOT_PASSWORD=$(openssl rand -hex 32)"
  echo "POSTGRES_PASSWORD=$(openssl rand -base64 32 | tr -d '=+/')"
  echo "CLICKHOUSE_PASSWORD=$(openssl rand -base64 32 | tr -d '=+/')"
  echo "LANGFUSE_INIT_USER_PASSWORD=$(openssl rand -base64 24 | tr -d '=+/')"
  echo "LANGFUSE_INIT_PROJECT_PUBLIC_KEY=pk-lf-$(openssl rand -hex 16)"
  echo "LANGFUSE_INIT_PROJECT_SECRET_KEY=sk-lf-$(openssl rand -hex 32)"
} >> .env
```

Then open `.env` and:

- Replace the existing `CHANGEME_*` placeholders with the new values
- Set `DOMAIN_OWUI`, `DOMAIN_LANGFUSE`, `OPENWEBUI_URL`, `LANGFUSE_URL`
- Set `ACME_EMAIL` to a real address (Let's Encrypt expiry warnings)
- Set `OPENROUTER_API_KEY` (or replace OpenRouter with another provider)

> **`ENCRYPTION_KEY` must be exactly 64 hex chars.** Langfuse refuses to
> start otherwise — you'll see `Error: ENCRYPTION_KEY must be 32 bytes`.

---

## First Login

### Open WebUI

1. Open `https://${DOMAIN_OWUI}`
2. The first user to sign up becomes admin
3. Subsequent signups land in `pending` state — admin approves them
4. **After the first admin exists, set `ENABLE_SIGNUP=false`** in `.env`
   and `docker compose up -d openwebui` to disable open registration

### Langfuse

The headless-init variables in `.env` (`LANGFUSE_INIT_*`) bootstrap the
first organization, project, and admin user automatically. Log in with:

- Email: `LANGFUSE_INIT_USER_EMAIL`
- Password: `LANGFUSE_INIT_USER_PASSWORD`

The project's API keys are exactly the ones you put in
`LANGFUSE_INIT_PROJECT_PUBLIC_KEY` / `LANGFUSE_INIT_PROJECT_SECRET_KEY`.

> **Security:** after the first successful start, **remove all
> `LANGFUSE_INIT_*` lines from `.env`** so the admin password is no
> longer sitting in plaintext on disk. The values stay configured in
> the Langfuse Postgres DB.

---

## Connecting Open WebUI to Langfuse

The `pipelines` service hosts filters that intercept OWUI chat
completions and forward traces to Langfuse.

1. Drop a Langfuse filter pipeline into `pipelines_data` volume:
   ```bash
   docker compose exec pipelines bash -c '
     curl -fsSL https://raw.githubusercontent.com/open-webui/pipelines/main/examples/filters/langfuse_filter_pipeline.py \
       -o /app/pipelines/langfuse_filter_pipeline.py
   '
   docker compose restart pipelines
   ```
2. In OWUI: **Admin Panel → Settings → Pipelines → Langfuse Filter**
3. Set:
   - `Secret Key` = `LANGFUSE_INIT_PROJECT_SECRET_KEY`
   - `Public Key` = `LANGFUSE_INIT_PROJECT_PUBLIC_KEY`
   - `Host` = `http://langfuse-web:3000` (in-cluster) or your public URL
4. Send a chat message — it should appear as a trace in Langfuse within
   ~5 s

Reference: <https://langfuse.com/docs/integrations/openwebui>

---

## Optional: OAuth / SSO

Out of the box, OWUI uses local username/password auth. To switch to
Okta / Azure AD / Google / Authentik, uncomment the OAuth block in
`.env` and fill in the values:

```dotenv
ENABLE_OAUTH_SIGNUP=true
OAUTH_PROVIDER_NAME=Okta
OAUTH_CLIENT_ID=<from your IdP>
OAUTH_CLIENT_SECRET=<from your IdP>
OPENID_PROVIDER_URL=https://your-tenant.okta.com/.well-known/openid-configuration
OAUTH_SCOPES=openid email profile groups

ENABLE_OAUTH_GROUP_MANAGEMENT=true
OAUTH_GROUPS_CLAIM=groups
ENABLE_OAUTH_GROUP_CREATION=false
OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true
```

Then `docker compose up -d openwebui`. Group management requires that
your IdP emits a `groups` claim and that you pre-create matching
groups in OWUI's admin panel (or set `ENABLE_OAUTH_GROUP_CREATION=true`
to auto-create them).

**Redirect URI** to register at the IdP:
`https://${DOMAIN_OWUI}/oauth/<provider>/callback`

---

## Backups

The state lives in named volumes — **back them up**:

| Volume | Contains |
|---|---|
| `postgres_data` | Langfuse users, projects, traces metadata |
| `clickhouse_data` | Langfuse traces / observations / scores (the bulk of the data) |
| `redis_data` | Langfuse queue (ephemeral, but AOF) |
| `minio_data` | Langfuse blob uploads (events, media, exports) |
| `openwebui_data` | OWUI users, chats, RAG documents, settings |
| `pipelines_data` | Pipelines code + their per-pipeline state |
| `caddy_data` | Let's Encrypt certs + ACME state |

A simple offsite backup loop:

```bash
docker run --rm \
  -v owui-langfuse-stack_postgres_data:/src:ro \
  -v /your/backup/path:/dst \
  alpine tar czf /dst/postgres-$(date +%F).tgz -C /src .
```

For Postgres specifically, `pg_dump` inside the container is more
robust than tar'ing the data dir.

---

## Upgrades

```bash
# Bump image tags in docker-compose.yml, then:
docker compose pull
docker compose up -d
docker compose logs -f langfuse-web   # watch migrations
```

For Langfuse major-version upgrades, **always read the upstream
release notes first** — schema changes are common.

---

## Troubleshooting

**`langfuse-web` boot loop with `ENCRYPTION_KEY must be 32 bytes`**
→ Your key isn't exactly 64 hex chars. `openssl rand -hex 32` produces
the right format.

**Pipelines returns 404 on `/v1/models`**
→ The Langfuse filter file isn't in `pipelines_data`, or `pipelines`
needs a restart after dropping it in.

**Caddy certificate fails: `no such host`**
→ DNS A-record hasn't propagated yet. Wait a few minutes and check
`dig +short ${DOMAIN_OWUI}`. Use ACME staging (commented line in
`Caddyfile`) while iterating to avoid Let's Encrypt rate limits.

**ClickHouse refuses to start: `Cannot lock file …`**
→ Stale lock from an unclean shutdown. Stop everything, remove the
container, restart:
```bash
docker compose stop clickhouse && docker compose rm -f clickhouse
docker compose up -d clickhouse
```

**OWUI shows `Error: Model not found` for a custom model**
→ Custom models have access-control. Open *Workspace → Models →
edit model → Access* and grant either `Public` or specific groups.

**MinIO console at `http://YOUR-SERVER:9091/` doesn't load**
→ It's bound to `127.0.0.1` only. SSH-tunnel:
```bash
ssh -L 9091:127.0.0.1:9091 user@your-server
# then open http://localhost:9091
```

---

## Production hardening to-dos (out of scope for POC)

- Move Postgres + ClickHouse to managed services (RDS / Aiven / Cloud)
- Replace MinIO with cloud S3 (avoids AGPLv3, gets versioning + lifecycle)
- Add a Watchtower or Renovate flow for image updates
- Add Prometheus exporters + a Grafana dashboard
- Configure `caddy` log shipping (currently json-file with rotation)
- Run as non-root in containers where the upstream image allows
- Network-segment per service (db-net / app-net) instead of one bridge

---

## Credits / Upstream

This stack is just configuration glue. The actual software is built by:

- **Open WebUI** — <https://github.com/open-webui/open-webui> (BSD-3 + branding)
- **Open WebUI Pipelines** — <https://github.com/open-webui/pipelines> (MIT)
- **Langfuse** — <https://github.com/langfuse/langfuse> (MIT + commercial EE modules)
- **Caddy** — <https://github.com/caddyserver/caddy> (Apache-2.0)
- **PostgreSQL** — <https://www.postgresql.org/> (PostgreSQL License)
- **ClickHouse** — <https://github.com/ClickHouse/ClickHouse> (Apache-2.0)
- **Redis** — <https://redis.io/> (RSALv2/SSPLv1 since 7.4)
- **MinIO** — <https://github.com/minio/minio> (AGPLv3 — see `NOTICE`)

See `NOTICE` for full attribution.

---

## Disclaimer

This is a **proof-of-concept template**. It has sane defaults but
hasn't been audited for any specific production environment. Before
running it in front of real users, you should at minimum:

1. Review every image's CVE feed against the pinned versions
2. Replace headless-init credentials with rotated values via the UI
3. Configure offsite backups and test restores
4. Add monitoring + alerting
5. Decide whether you're comfortable with MinIO's AGPLv3 obligations
   (or swap it for cloud S3)

No warranty, express or implied. See `LICENSE`.

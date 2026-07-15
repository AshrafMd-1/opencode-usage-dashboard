# OpenCode Usage Dashboard

A self-hosted dashboard for OpenCode usage trends. A Node.js collector retrieves all paginated usage records, stores them in PostgreSQL, and refreshes a [Rill](https://github.com/rilldata/rill) dashboard backed by DuckDB.

```text
OpenCode → collector → PostgreSQL → Rill/DuckDB → dashboard
```

The stack supports Linux ARM64 (`aarch64`) and AMD64. It is intended for Ubuntu 22.04 and also works with Docker Desktop.

## Dashboard contents

The **OpenCode Usage** dashboard provides time trends and comparisons for:

- requests and normalized USD cost
- input, output, reasoning, cache-read, and cache-write tokens
- total input/cache and total token volume
- average cost per request and cache-read ratio
- model, provider, plan, workspace, and session dimensions
- today, week/month to date, 7-day, 30-day, 3-month, all-time, and custom ranges

Quota/reset data is intentionally not included because it has a different data grain.

## Requirements

- Docker Engine with the Compose plugin
- an OpenCode workspace usage URL
- the workspace's secret `auth` cookie

No local Node.js or Rill installation is required.

## Setup

1. Create local configuration:

   ```bash
   cp .env.example .env
   ```

2. Edit `.env` and set at least:

   ```dotenv
   OPENCODE_USAGE_URL=https://opencode.ai/workspace/<workspace-id>/usage
   OPENCODE_AUTH=<raw-auth-cookie-value>
   POSTGRES_PASSWORD=<long-random-password>
   ```

3. Build and start:

   ```bash
   docker compose up --build -d
   ```

4. Follow startup:

   ```bash
   docker compose logs -f collector rill
   ```

5. Open:

   - controls: <http://localhost:3000>
   - dashboard: <http://localhost:9009>

The initial collector run performs a complete OpenCode refetch. Existing PostgreSQL volumes use incremental refreshes.

## Refresh behavior

A refresh has two stages:

1. the collector fetches OpenCode records and upserts them into PostgreSQL;
2. it asks Rill to reload the `opencode_usage` model into DuckDB.

The controls page offers:

- **Refresh now** — run the complete pipeline;
- **Retry Rill only** — reload Rill after PostgreSQL succeeded but Rill was unavailable;
- **Enable/disable auto-refresh** — persisted in PostgreSQL.

Auto-refresh defaults to every 6 hours. `AUTO_REFRESH_ENABLED` initializes a new database only; afterward, the controls-page setting is authoritative. Dashboard filter changes merely re-query data already loaded into Rill.

The incremental collector preserves the five-sequential-request fingerprint overlap. A healthy no-change run normally fetches only the first 50-row OpenCode page. If no overlap is found, it safely scans all pages and upserts by fingerprint.

## Security

`OPENCODE_AUTH` is a live authentication secret. Never commit, print, or share `.env`.

Compose binds both web ports only to loopback:

```text
127.0.0.1:3000
127.0.0.1:9009
```

PostgreSQL has no published host port. **Rill Preview is not an authentication boundary.** Keep both web services behind an authenticated HTTPS proxy such as Cloudflare Access. Never assign a public domain to PostgreSQL.

## Deploy on Coolify with Cloudflare Access

This is the simplest production path for an ARM64 Ampere server. `Dockerfile.rill` automatically downloads the ARM64 Rill release, and the named PostgreSQL and Rill volumes persist across Coolify redeployments.

1. Push this repository to a private Git repository.
2. In Coolify, create a **Docker Compose** resource from that repository and select `compose.yaml`.
3. Add these environment variables in Coolify. Mark `OPENCODE_AUTH` and `POSTGRES_PASSWORD` as runtime secrets, not build-only values:

   ```dotenv
   OPENCODE_USAGE_URL=https://opencode.ai/workspace/<workspace-id>/usage
   OPENCODE_AUTH=<raw-auth-cookie-value>
   POSTGRES_DB=opencode_usage
   POSTGRES_USER=opencode
   POSTGRES_PASSWORD=<long-random-password>
   AUTO_REFRESH_ENABLED=true
   AUTO_REFRESH_INTERVAL_MINUTES=360
   RILL_INTERNAL_URL=http://rill:9009
   RILL_VERSION=0.87.8
   ```

4. Assign domains to the two web services in Coolify:

   - Rill service: `https://usage.example.com:9009`
   - collector service: `https://usage-control.example.com:3000`

   The suffixes tell Coolify which container port to proxy; visitors still use normal HTTPS on port 443. Do not assign a domain to `postgres`.
5. In Cloudflare DNS, point both hostnames to the Coolify server and enable the orange-cloud proxy. Use **Full (strict)** TLS.
6. In **Cloudflare Zero Trust → Access controls → Applications**, create a self-hosted application covering both hostnames. Add an **Allow** policy containing only your Google account and enable Google as the identity provider. Access denies all unmatched users by default.
7. Deploy, then verify both URLs, service health, and the first collector refresh.

The Compose host-port mappings bind only to `127.0.0.1`, while Coolify's proxy reaches the services over its private Docker network. This avoids exposing ports 3000 or 9009 directly. Cloudflare Access protects requests arriving through Cloudflare; to prevent origin bypass, restrict origin traffic to Cloudflare IP ranges or publish through a Cloudflare Tunnel.

## Operations

### Status and logs

```bash
docker compose ps
docker compose logs --tail=200 collector
docker compose logs --tail=200 rill
docker compose logs --tail=200 postgres
```

### Validate PostgreSQL totals

```bash
docker compose exec postgres psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c \
  'SELECT COUNT(*) AS requests, SUM(cost_usd) AS cost_usd FROM usage_requests;'
```

If those shell variables are not exported, use the values from `.env` directly.

### Stop or restart

```bash
docker compose stop
docker compose restart collector rill
```

### Backup

```bash
docker compose exec -T postgres pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" > opencode-usage.sql
```

The dump contains usage metadata such as workspace/session/key IDs. Store it securely.

Restore into an empty database with `psql` after recreating the stack.

### Reset all stored data

```bash
docker compose down -v
docker compose up --build -d
```

**Warning:** `down -v` permanently deletes PostgreSQL history, refresh settings, and Rill runtime state. The next start performs a full OpenCode refetch.

### Upgrade Rill

Rill is pinned to `v0.87.8`. `Dockerfile.rill` contains the official SHA-256 digest for each AMD64/ARM64 release archive. To upgrade:

1. change the version;
2. obtain both archive digests from the official GitHub release;
3. update both checksum build arguments;
4. rebuild with `docker compose build --no-cache rill`;
5. run the verification steps below.

Changing only `RILL_VERSION` intentionally causes the image build to fail checksum verification.

## ARM64 verification

On the Ubuntu host:

```bash
uname -m
docker compose build --no-cache
docker compose up -d
docker compose ps
```

`uname -m` should report `aarch64`. The Rill Dockerfile maps Docker's `arm64` target to the signed `rill_linux_arm64.zip` release and does not require emulation.

Confirm published bindings:

```bash
docker compose port collector 3000
docker compose port rill 9009
```

Both should report `127.0.0.1`.

## Troubleshooting

### Collector reports HTTP 401/403

The OpenCode cookie has likely expired. Replace `OPENCODE_AUTH` in `.env`, then recreate the collector:

```bash
docker compose up -d --force-recreate collector
```

Existing PostgreSQL and dashboard data remain available.

### OpenCode bundle discovery fails

OpenCode may have changed its internal SolidStart assets or `usage.list` reference. Check sanitized collector logs. Do not paste cookies or raw `.env` contents into an issue.

### PostgreSQL schema is missing after editing `postgres/init.sql`

Initialization scripts run only for a new PostgreSQL volume. Apply a migration manually or reset the volume after taking a backup.

### PostgreSQL refreshed but Rill did not

Use **Retry Rill only** on <http://localhost:3000>. Then inspect:

```bash
docker compose logs --tail=200 rill collector
```

### Rill starts with a YAML/model error

Run:

```bash
docker compose exec rill rill project status --local
```

Then inspect the Rill logs and the files under `rill/`.

## Development

Run unit tests locally:

```bash
npm install
npm test
```

Validate Compose interpolation without starting services:

```bash
docker compose config --quiet
```

Core files:

```text
src/                    collector, PostgreSQL, scheduler, and controls
postgres/init.sql       durable schema and constraints
rill/                    connector, model, metrics view, and dashboard YAML
compose.yaml             localhost-only service topology
Dockerfile.collector     Node collector image
Dockerfile.rill          verified multi-architecture Rill image
```

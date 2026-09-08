# PostureRMM Production Deployment

<!-- Mirrored verbatim to PostureRMM/PostureRMM; never edit on the hub.
`just public-docs-sync` — deploy/release/check-public-docs.sh reds on a difference. -->

[quickstart.md](quickstart.md) installs the stack and
[configuration.md](configuration.md) is the settings reference. This is the
runbook for what comes after: a Bastion in a DMZ, the segment proxy, agent
rollout, TLS and agent trust, upgrades, metrics, retention, backup and
restore.

## Architecture

```
Browser / Agent → HTTPS/443 → backend → db:5432
                                (terminates TLS and serves /api, /health,
                                 /install, /downloads, /winget, and the
                                 MiniJinja + HTMX admin UI at /)
HTTP bootstrap → HTTP/80 ───────┘

bastion → HTTPS → Feed API
bastion → HTTP  → backend:3000 (admin API, status, sync)
```

No separate web or reverse-proxy tier: the backend serves both the JSON API and
the admin UI. Its `:80` listener redirects to HTTPS except for the plaintext
`/install.ps1` and `/downloads/*` bootstrap paths, and its `:3000` plane is
never published to the host. Only the Bastion talks to the internet.

## License

PostureRMM is proprietary software, licensed under the
[End-User License Agreement](../LICENSE) shipped with every release; installing
or running it accepts those terms. Community is free and perpetual up to 50
endpoints, and a paid entitlement raises the cap. Only builds published by us —
GitHub Release assets and images in our own GHCR namespace — are covered, so
pull what the quickstart names and nothing else.

## TLS and agent trust

**The default install is self-signed, and agents pin the certificate
authority.** There is no Let's Encrypt, no ACME, and nothing to obtain before
your first endpoint enrolls — a fresh install must work offline on a LAN with
no domain.

Agents do *not* use the system root store on a default install. The install
script stamps the CA's SHA-256 fingerprint into the agent, which then does real
chain, validity and hostname checking against a trust anchor of exactly one
certificate.

Two consequences worth knowing before you operate this:

- **Rotating the leaf is safe.** Changing `POSTURERMM_DOMAIN` re-mints only the
  leaf; the CA is persisted and reused, so enrolled agents keep working with no
  action on the endpoints.
- **Replacing the CA is a fleet event, but not a manual one.** It happens on
  first boot, and otherwise only when you remove `ca.crt` + `ca.key` or move
  between a public certificate and a private one. Enrolled agents lose their
  transport at that moment and recover on their own: after a few failed
  check-ins each asks the `:80` bootstrap listener for the current authority and
  adopts it **only** if the server can prove it knows that endpoint's own
  enrollment secret. Expect the fleet to reconverge within minutes.

  Losing both files outright is survivable for the *agents* for the same reason.
  What you lose is everything that trusted the old CA by hand: browsers, OS
  trust stores, external tooling. Both files stay in the backup set.

**The pin is delivered over plain HTTP.** Port 80 serves `/install.ps1`
deliberately: PowerShell 5.1 on a fresh Windows Server 2016 cannot complete an
HTTPS fetch at all. Enrollment therefore roots in one unauthenticated fetch on
your LAN. Pinning narrows the trust-on-first-contact window to that single
request; it does not remove it. Across a network you do not control, distribute
the bootstrap through a trusted software-management channel and use a real
certificate — the approval queue, not a shared secret, is what gates admission.

Bringing your own certificate is covered in
[configuration.md](configuration.md). One case it cannot detect: if your *edge*
terminates TLS with a certificate the backend never generated (a reverse proxy,
a tunnel), set `POSTURERMM_SERVER__AGENT_TRUST_MODE=system_roots` explicitly.

## Fronting the backend with your own proxy

[configuration.md](configuration.md) explains why
`POSTURERMM_SERVER__TRUSTED_PROXIES` defaults to empty; running the stack as
shipped, there is nothing to do. Put your own terminator in front and two
settings become required, not optional:

1. **`POSTURERMM_SERVER__TRUSTED_PROXIES` = your terminator's address.** Skip
   it and every request resolves to the proxy: each audit row records the proxy
   rather than the actor, and the per-IP login limiter collapses to one bucket,
   so ten failed logins from anywhere return 429 to every user. Both failures
   are silent. The backend warns once when a forwarding header arrives from an
   address it does not trust; that line means this setting is missing.
2. **`Strict-Transport-Security` is your terminator's to send.** The backend
   emits HSTS only when it terminates TLS itself, so under
   `POSTURERMM_TLS__ENABLED=false` it sends none — a promise about the edge is
   only the edge's to make.

A PostureRMM **proxy relay** is governed by the same rule: to attribute agent
audit rows to the *agent* rather than to the relay, list the relay's address
here.

## Bastion in a DMZ

By default the Bastion runs beside the backend at `http://bastion:8300`, sharing
a bearer secret the backend generates on first boot through a read-only volume.
No operator input is required, including for an offline install.

For a split topology — Bastion on a DMZ host, backend/db/admin UI in the
protected zone — the hosts cannot share that volume. Generate one secret with
`openssl rand -hex 32` and put the same value in the `.env` beside each compose
file.

1. **On the DMZ host** — take
   [`docker-compose.bastion.yml`](https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.bastion.yml)
   from the release page (that file alone; it bind-mounts nothing), then create
   `.env` beside it with `POSTURERMM_VERSION`, the generated secret, and the
   deployment id from the backend's boot log — the compose refuses whichever is
   absent:
   ```bash
   POSTURERMM_BASTION__SECRET=<generated-64-character-hex-value>
   POSTURERMM_FEED__DEPLOYMENT_ID=<uuid-from-the-backend-boot-log>
   docker compose -f docker-compose.bastion.yml up -d
   docker compose -f docker-compose.bastion.yml ps   # should report bastion healthy
   ```
2. **On the protected-zone host** — leave `docker-compose.yml` as-is and set
   both values in `.env`:
   ```bash
   POSTURERMM_BASTION__URL=https://bastion.dmz.example:8443
   POSTURERMM_BASTION__SECRET=<the-same-generated-64-character-hex-value>
   docker compose up -d backend   # or `docker compose up -d` to recreate everything
   ```
3. **Verify authentication** — every `/api/v1/*` route requires
   `Authorization: Bearer <secret>`; a wrong or missing value returns `401`
   naming `POSTURERMM_BASTION__SECRET`.
4. **Verify feed reachability** —
   ```bash
   # /health answers only "the process is listening", which is all the container
   # healthcheck may ever mean. /health/ready reports whether the Bastion can
   # reach the feed and where its identity came from. Both /health endpoints are
   # public — the container healthcheck and any load balancer need them without
   # the shared secret — and neither names a secret: /health/ready reports THAT
   # an identity is set, never its value. /health/ready ALWAYS answers 200: a
   # Bastion behind a closed maintenance window is doing what it was told, and a
   # probe in front of it must not restart it.
   curl -s http://localhost:8300/health/ready
   {"verdict":"rejected",
    "deployment_id":{"configured":false,"last_attempt_source":"unset"},
    "upstream":{"feed_origin":"https://feed.posturermm.com",
                "last_success_unix":null,"last_success_seconds_ago":null,
                "last_attempt_unix":1756500000,"last_attempt_seconds_ago":12,
                "last_attempt_outcome":"http_error","last_attempt_status":401}}
   ```

   | `verdict` | What it means |
   |---|---|
   | `contacted` | a feed fetch has succeeded; `last_success_seconds_ago` says how long ago |
   | `not_yet_contacted` | an identity exists but nothing has succeeded yet — the normal reading behind a closed maintenance window, and not an alarm |
   | `rejected` | the feed refused the identity; `last_attempt_status` carries the `401`/`403` |
   | `misconfigured` | no identity from any source and no successful contact ever — set `POSTURERMM_FEED__DEPLOYMENT_ID` from the backend's boot log |

5. **Firewall** — protected zone → DMZ, outbound only, destination port matching
   whatever the Bastion (or its reverse proxy) listens on. No inbound rule from
   the DMZ to the protected zone is required or supported — the Bastion never
   initiates a connection inward.
6. **Cross-zone TLS** — the hop must be HTTPS and the Bastion has no built-in
   TLS termination. Run a reverse proxy on the DMZ host, terminate TLS with your
   own certificate, and forward to the Bastion's plain-HTTP `:8300`. The shared
   secret authenticates the backend; it does not encrypt traffic.

Changing the split URL or secret means updating `.env` and recreating the
backend container, on both hosts; during a mismatch the backend fails closed
with a named authentication error.

## Proxy container

A proxy is optional and exists for **segmented networks**: agents that cannot
reach the backend directly point at a proxy in their own segment, which relays
for them, caches downloads so a hundred endpoints pull a patch once, and queues
writes durably while the backend is unreachable.

A proxy runs where its agents are, not beside the backend, so the stock
`docker-compose.yml` defines none. It ships as its own release download,
`docker-compose.proxy.yml`, version-stamped like the stock one and carried in
the offline bundle with its image.

### Registering it

1. In the admin UI, **Proxies → New**. You get an ID and a secret; the secret is
   shown once.
2. On a Docker host in the segment, fetch it and write `.env` beside it:

```bash
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.proxy.yml
cat > .env <<'EOF'
POSTURERMM_PROXY_BACKEND_URL=https://posturermm.example.com
POSTURERMM_PROXY_ID=<id from step 1>
POSTURERMM_PROXY_SECRET=<secret from step 1>
EOF
docker compose -f docker-compose.proxy.yml up -d
```

3. Point that segment's agents at `http://<proxy-host>:8443` instead of the
   backend. The proxy injects `X-Forwarded-For` and `X-Proxy-ID`, so the audit
   log still records the real endpoint — provided
   `POSTURERMM_SERVER__TRUSTED_PROXIES` names the proxy's address. Omit it and
   every row from that segment shows the proxy instead.

### Configuring it

Built-in defaults, an optional TOML file, then `POSTURERMM_PROXY_*` environment
variables. The image ships no config file; an absent one is not an error.

| Variable | Default | Meaning |
|---|---|---|
| `POSTURERMM_PROXY_BACKEND_URL` | — | Where to relay. Required. |
| `POSTURERMM_PROXY_ID` | — | Proxy record ID from the UI. Required. |
| `POSTURERMM_PROXY_SECRET` | — | Shared secret from the UI. Required. |
| `POSTURERMM_PROXY_LISTEN_ADDRESS` | `0.0.0.0:8443` | Listener. |
| `POSTURERMM_PROXY_TLS_ENABLED` | `false` | See the TLS note below. |
| `POSTURERMM_PROXY_LOG_LEVEL` | `info` | |
| `POSTURERMM_PROXY_CONFIG` | `./config/posturermm-proxy.toml` | Optional TOML, mounted by you. |

**TLS.** The listener is plain HTTP on `:8443` by default — the port number is
conventional, not a promise. Across any boundary you do not control, either set
`POSTURERMM_PROXY_TLS_ENABLED=true` with `cert_path`/`key_path` in a mounted
TOML, or front the proxy with your own TLS terminator. If you enable TLS,
**override the image healthcheck too** — it curls
`http://localhost:8443/health` and reports the container unhealthy against an
HTTPS listener.

**Liveness.** `GET /health` answers **503** with
`{"status":"degraded","backend_reachable":false}` when the backend is
unreachable. That is a real signal but *not* a reason to restart the proxy:
serving its segment from cache and queueing writes for replay is exactly what it
should be doing during an outage. The image's healthcheck therefore probes
liveness only. If you wire this endpoint into your own monitoring, treat
503-degraded as "backend down", not "proxy down". The proxy also heartbeats the
backend every 60s, so one that stops checking in shows as stale in the UI.

## Agent rollout

1. **Create an install key** — Admin UI → Settings → Install keys → Create key.
   The key is optional and buys one thing: skipping the approval queue, which
   unattended rollouts need. Without it the same one-liner still enrolls, inert,
   until you admit the endpoint from Endpoints.
   `POSTURERMM_SERVER__ALLOW_AUTO_ENROLLMENT=true` admits keyless enrollments
   instantly — a trusted-LAN convenience, and an open door on anything
   internet-reachable.
2. **Run the one-liner** on the target Windows endpoint (elevated PowerShell):
   ```powershell
   irm "http://posturermm.example.com/install.ps1?key=..." | iex
   ```
   The shim writes `C:\ProgramData\PostureRMM\install-key.txt`, downloads the
   MSI, and runs `msiexec /i /qn SERVERURL=https://posturermm.example.com`. The
   MSI registers and starts the agent service; the agent reads the key file on
   first boot, enrolls, and self-cleans it.

**Where the MSI on the server comes from.** There is never a step for you to
place it. An **offline install** gets it from the bundle — `install.sh` seeds
the downloads volume before the stack starts. An **online install** gets it from
the product feed on the Bastion's first pull. Until that pull completes,
Endpoints → Install first endpoint shows *Fetching agent v…* and deliberately
offers no install command, because the one-liner would download an MSI that is
not there yet. Behind a maintenance window that is what you will see until the
window opens; the request is queued durably, so nothing needs re-asking.

For SCCM / Intune / GPO flows, take the MSI URL and checksum from Admin UI →
Endpoints → Install agent → More install options and pass `SERVERURL` on the
`msiexec` command line; drop `install-key.txt` beside the MSI so the agent picks
it up from the captured source dir.

### Changing the agent's backend URL

`ServerUrl` is registry-canonical — the agent reads
`HKLM\SOFTWARE\PostureRMM\Agent\ServerUrl` at boot and ignores TOML and env. Two
supported paths to change it:

1. **Per-machine** — elevated PowerShell on the endpoint:
   ```powershell
   posturermm-agent.exe config set server.url https://posturermm.example.com
   posturermm-agent.exe config get     # sanity check
   ```
   Validates the URL, writes the registry value, restarts `PostureRMM-Agent`
   and emits an `audit.config` log line.

2. **Fleet-wide (SCCM / Intune / GPO)** — push an explicit reinstall:
   ```cmd
   msiexec /x posturermm-agent.msi /qn
   msiexec /i posturermm-agent.msi SERVERURL=https://new-host.example.com /qn
   ```
   A command-line `SERVERURL=` always wins, on upgrade as on fresh install; omit
   it and the MSI recovers the operator's existing value rather than stranding
   the agent.

## Operations

```bash
# Status + logs
docker compose ps
docker compose logs -f backend
docker compose logs -f bastion

# Restart a single service
docker compose restart backend

# Upgrade — replace docker-compose.yml with the new release's copy, which
# carries that release's version baked in, then pull and recreate. Nothing to
# edit; add POSTURERMM_VERSION to .env only to pin a version against the file.
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.yml
docker compose pull
docker compose up -d

# Drop into the DB shell
docker compose exec db psql -U posturermm posturermm
```

Read the target version's release notes. A release needing a one-time manual
step says so there, and the backend diagnoses that class of problem itself — it
refuses to start, names the paths and identity it needs, and prints the recipe.

### Pushing the images into a private registry

Never required — `docker load` from the offline bundle on each host is the
supported path. Sites already running Harbor / Artifactory / Nexus can seed one
from the same bundle so hosts pull internally:

```bash
# Option A — docker load + retag + push
docker load -i images.tar
docker tag ghcr.io/posturermm/posturermm-backend:X.Y.Z registry.internal/posturermm-backend:X.Y.Z
docker push registry.internal/posturermm-backend:X.Y.Z
# ...repeat for bastion and postgres...

# Option B — skopeo, no local docker daemon needed
skopeo copy docker-archive:images.tar:ghcr.io/posturermm/posturermm-backend:X.Y.Z \
  docker://registry.internal/posturermm-backend:X.Y.Z
```

Then point the image refs at the internal registry — convenience, not product
surface.

### Prometheus metrics

`/metrics` is served on the backend's internal `:3000` plane only, which is
never published to the host. It takes no credentials, so `:443` and `:80` 404
it. Scrape it from the same Compose network (`docker network connect
posturermm_default <your-scraper>` if your Prometheus runs elsewhere):

```yaml
# prometheus.yml
scrape_configs:
  - job_name: posturermm
    static_configs:
      - targets: ["backend:3000"]
```

```bash
# Confirm it answers, from the host running the stack:
docker compose exec backend curl -s http://localhost:3000/metrics | head
```

With `POSTURERMM_TLS__ENABLED=false` that single `:3000` listener *is* the
internal plane and carries `/metrics`; do not proxy the path through to the
public side.

### The backend turns PostgreSQL's JIT off

Every pooled connection runs `SET jit = off`, so `SHOW jit` in a `psql` shell
reports `on` while the application runs with it off. It is set in the pool, not
in `docker-compose.yml`, so it holds against your own PostgreSQL too. Measured
on a 10-endpoint fleet with JIT as the only variable:

| query | with JIT | without |
|---|---|---|
| compliance fleet rollup | 940 ms | 191 ms |
| patching fleet queue | 178 ms | 72 ms |

No query in the product was measured faster with JIT on.

### Data retention

Settings → Retention (SuperAdmin) controls four windows, all enforced daily by
the same scheduler job:

| Setting | Default | Notes |
|---|---|---|
| Task history | 90 days | Terminal `tasks` rows. Deployment stats are materialized to the parent `deployments` row first, so fleet-level history survives the prune. |
| Compliance snapshot | 365 days | Daily per-endpoint compliance percentage, feeds trend charts and evidence packs. |
| Posture snapshot | 365 days | Daily per-endpoint posture score, feeds fleet sparklines. Same shape and growth rate as the compliance snapshot, and the same default window: **upgrading an existing deployment begins pruning rows older than 365 days immediately**. There is no unlimited option for this window. |
| Audit log | **0 (keep forever)** | The one window that defaults to unlimited, and the only one of the four with an unlimited option at all, so upgrading never starts deleting an existing operator's history on its own. Set a positive number of days to opt into pruning. |

The first three accept 30–3650 days with no "never prune" option; audit-log
retention accepts 0–3650. Pruning runs in bounded batches so a first run against
years of backlog does not hold a long-lived lock on a live table, and pruning
the audit log writes one more audit row (`audit_log_prune`) naming the row count
and the cutoff — the trail records its own trimming.

A run that cannot read all four windows — a database blip, or a value edited
into something that is not a day count — prunes nothing and logs the key at
fault, instead of falling back to the defaults above. The next daily run
recovers on its own; repair a bad value by saving a real number on the
Retention page.

Three telemetry histories are pruned on fixed windows you do not set, hourly
rather than daily: performance metrics and endpoint reachability at **30 days**,
and disk SMART history at **2 years** — affordable because that history is
stored on change plus a daily heartbeat, not on a cadence.

## Security Checklist

- [x] TLS termination by the backend itself
- [x] The backend publishes 80/443 only; its `:3000` plane, the DB and the Bastion stay on the internal Docker network
- [x] The backend makes zero outbound internet connections — the Bastion is the only internet-facing component
- [x] JWT signing secret auto-provisioned to `posturermm-data` volume (or externally supplied via `JWT_SECRET`, 32+ chars)
- [x] Keyless enrollment waits in the approval queue; install keys are the opt-in unattended lane
- [x] 2FA enforced for admin accounts (Settings → Security → Enforce 2FA)
- [x] Audit-log source IP is trustworthy: forwarding headers are believed only from a peer named in `POSTURERMM_SERVER__TRUSTED_PROXIES` — a client cannot forge its own address into the trail
- [x] Signing out revokes the session's refresh token server-side and records a `logout` event

## Backup & Restore

Treat the database, secret-bearing volumes and `.env` as one backup set. A
database dump alone appears to restore, but the backend then silently
auto-provisions a new internal CA, JWT signing secret and TOTP key on first
boot. The UI and database look healthy while enrolled agents reject the new
certificate and existing TOTP enrollments and backup codes stop working.

Run the backup from the directory holding `docker-compose.yml` and `.env`. It
briefly stops the backend so its volumes cannot change mid-archive; PostgreSQL
and the Bastion stay up. The PostgreSQL image already present is the archive
helper, so this works from an offline bundle too.

```bash
# Create one private directory for the entire backup set.
BACKUP_DIR="$(pwd)/posturermm-backup-$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -m 700 "$BACKUP_DIR"
cp --preserve=mode .env "$BACKUP_DIR/.env"

# Quiesce the service that mounts the volumes being archived.
docker compose stop backend

# Back up PostgreSQL logically; do not copy its live data-directory volume.
docker compose exec -T db pg_dump -U posturermm posturermm \
  > "$BACKUP_DIR/posturermm.sql"

# Archive every non-regenerable application volume, preserving dotfiles,
# ownership, and permissions. Exclude the separately-mounted downloads cache
# from /app/data.
ARCHIVE_IMAGE="$(docker inspect posturermm-db --format '{{.Config.Image}}')"
docker run --rm --volumes-from posturermm-backend \
  --mount "type=bind,src=$BACKUP_DIR,dst=/backup" "$ARCHIVE_IMAGE" \
  tar --exclude=./downloads -czf /backup/posturermm-data.tar.gz -C /app/data .
docker run --rm --volumes-from posturermm-backend \
  --mount "type=bind,src=$BACKUP_DIR,dst=/backup" "$ARCHIVE_IMAGE" \
  tar -czf /backup/posturermm-tls.tar.gz -C /app/tls .

# Make corruption detectable, then return the application to service.
(
  cd "$BACKUP_DIR"
  sha256sum .env posturermm.sql posturermm-data.tar.gz \
    posturermm-tls.tar.gz > SHA256SUMS
)
docker compose start backend
until docker compose exec -T backend curl -sf http://localhost:3000/health/ready \
  >/dev/null; do sleep 2; done
```

Copy the whole directory off-host as one unit. It holds database contents,
private keys, authentication secrets and the database password — store it with
the same controls as production credentials.

| Artifact | What it preserves | What breaks if it is omitted |
|---|---|---|
| `posturermm.sql` | The portable backup of `posturermm-db-data`: users, endpoints, configuration, audit history, and all other relational state | The application data is gone. A raw copy of the live PostgreSQL volume is not a substitute for `pg_dump`. |
| `posturermm-tls.tar.gz` | `posturermm-tls`: the internal CA and key, leaf certificate and key, and auto-generation sentinel | Every agent pinned to the old leaf/CA rejects the replacement certificate and stops connecting. |
| `posturermm-data.tar.gz` | `posturermm-data`: the auto-provisioned `jwt-secret`, `totp-secret`, and other backend-persisted secret material | Existing access tokens no longer validate. More importantly, stored TOTP secrets cannot be decrypted and MFA backup-code hashes no longer verify, locking 2FA users out. |
| `.env` | `POSTGRES_PASSWORD` and any operator-supplied `JWT_SECRET` or `POSTURERMM_CREDENTIAL_KEY` | PostgreSQL credentials no longer match, explicitly supplied JWT keys rotate, and stored credentials/evidence-signing keys encrypted under the credential key become unusable. Neither value is in `pg_dump`. |

Four volumes are deliberately excluded; none re-keys the installation or holds
authoritative data. Three are caches — `posturermm-bastion-data`,
`posturermm-binary-store` (refetchable installer blobs, which an offline
deployment may need to repopulate before deploying software) and
`posturermm-downloads`. The fourth, `posturermm-bastion-auth`, holds a secret
but still needs no backup: co-located, the backend regenerates it on first boot
and shares it with the Bastion; DMZ-split, it comes from
`POSTURERMM_BASTION__SECRET` in `.env`, which *is* in the backup set.

Restore only onto a fresh host with no existing PostureRMM containers or named
volumes. Install the same release's compose files, put the backup directory
beside them, and **do not run `docker compose up` yet** — the secret and TLS
volumes must be populated before the backend's first boot. Boot it against empty
volumes and it silently writes new secrets and a new CA, which a later restore
cannot undo.

```bash
# Point this at the complete backup directory and verify it before use.
BACKUP_DIR="$(pwd)/posturermm-backup-YYYYMMDDTHHMMSSZ"
(cd "$BACKUP_DIR" && sha256sum -c SHA256SUMS)

# Restore configuration first; Compose needs these values even to create the
# stopped containers. This intentionally precedes every Compose command.
install -m 600 "$BACKUP_DIR/.env" .env

# Create all three containers and empty named volumes without starting them.
docker compose create db backend bastion
ARCHIVE_IMAGE="$(docker inspect posturermm-db --format '{{.Config.Image}}')"

# Populate every archived volume BEFORE the backend first starts.
docker run --rm --volumes-from posturermm-backend \
  --mount "type=bind,src=$BACKUP_DIR,dst=/backup" "$ARCHIVE_IMAGE" \
  tar -xzf /backup/posturermm-data.tar.gz -C /app/data
docker run --rm --volumes-from posturermm-backend \
  --mount "type=bind,src=$BACKUP_DIR,dst=/backup" "$ARCHIVE_IMAGE" \
  tar -xzf /backup/posturermm-tls.tar.gz -C /app/tls

# Restore PostgreSQL while the backend is still stopped.
docker compose start db
until docker compose exec -T db pg_isready -U posturermm -d posturermm \
  >/dev/null; do sleep 2; done
docker compose exec -T db psql -v ON_ERROR_STOP=1 -U posturermm posturermm \
  < "$BACKUP_DIR/posturermm.sql"

# Only now may the backend boot; bring the application services back together.
docker compose start backend bastion
until docker compose exec -T backend curl -sf http://localhost:3000/health/ready \
  >/dev/null; do sleep 2; done
docker compose ps
```

After restore, verify an existing login and a 2FA-enabled account before
declaring recovery complete, and compare the restored CA fingerprint against a
known-good value:

```bash
docker cp posturermm-backend:/app/tls/ca.crt ./restored-ca.crt
openssl x509 -in ./restored-ca.crt -noout -fingerprint -sha256
rm ./restored-ca.crt
```

## Troubleshooting

- **Backend won't start:** `docker compose logs backend`. Migrations log themselves on startup.
- **502 from your own upstream proxy:** the backend container is down or not ready. Check `docker compose ps` — `backend` should be `healthy` — then `docker compose logs backend`.
- **Backend not serving HTTPS:** verify the `posturermm-tls` volume is mounted and `server.crt` / `server.key` exist; if not, check `docker compose logs backend` for certificate or listener errors.
- **UI loads blank / 404:** the admin UI is served at `/`; if it does not render the login screen the backend is not serving traffic yet (`docker compose logs backend`).
- **Agent can't connect:** verify DNS resolution, port 443 reachable, cert valid. `POSTURERMM_SERVER__PUBLIC_URL` must match what the agent sees.
- **Bastion returning 401 to the backend:** the bearer secret disagrees. The Bastion checks `POSTURERMM_BASTION__SECRET` (or, unset, the file at `[bastion] secret_path`) against what the backend presents; both sides must hold the same 32+ character value.
- **Bastion returning 502 on feed pulls:** it could not reach the feed origin. Check egress from the DMZ host to `[feed] api_url` (default `https://feed.posturermm.com`) and `docker compose logs bastion` for the upstream error.

---

Back to the [README](../README.md), the [quickstart](quickstart.md), or
[configuration.md](configuration.md).

# PostureRMM Production Deployment

<!-- Mirrored verbatim to PostureRMM/PostureRMM; never edit on the hub.
Each release publishes it there — deploy/release/check-public-docs.sh. -->

[quickstart.md](quickstart.md) installs the stack and
[configuration.md](configuration.md) is the settings reference. This is the
runbook for what comes after: a Bastion in a DMZ, the segment proxy, agent
rollout, TLS and agent trust, upgrades, metrics, retention, audit forwarding,
backup and restore.

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

No separate web or reverse-proxy tier: the backend serves the JSON API and the
admin UI. Its `:80` listener redirects to HTTPS except for the plaintext
`/install.ps1`, `/install-agent.sh` and `/downloads/*` bootstrap paths, and its
`:3000` plane is never published. Only the Bastion talks to the internet.

## License

PostureRMM is proprietary software, licensed under the
[End-User License Agreement](../LICENSE) shipped with every release; installing
or running it accepts those terms. Community is free and perpetual up to 50
endpoints, and a paid entitlement raises the cap. Only builds we publish (GitHub
Release assets and images in our GHCR namespace) are covered.

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

  Losing both files is survivable for agents; browsers, OS trust stores and
  external tooling that trusted the old CA by hand are what you lose. Both files
  stay in the backup set.

**The pin is delivered over plain HTTP.** Port 80 serves `/install.ps1`
deliberately: PowerShell 5.1 on a fresh Windows Server 2016 cannot complete an
HTTPS fetch. Enrollment therefore roots in one unauthenticated fetch on your LAN;
pinning narrows that trust-on-first-contact window to a single request. Across a
network you do not control, distribute the bootstrap through a trusted
software-management channel and use a real certificate. The approval queue gates
admission.

[configuration.md](configuration.md) covers bringing your own certificate. If your
*edge* (a reverse proxy, a tunnel) terminates TLS with a certificate the backend
never generated, set `POSTURERMM_SERVER__AGENT_TRUST_MODE=system_roots`
explicitly.

## Fronting the backend with your own proxy

[configuration.md](configuration.md) explains why
`POSTURERMM_SERVER__TRUSTED_PROXIES` defaults to empty; running the stack as
shipped, there is nothing to do. With your own terminator in front, three
settings are required:

1. **`POSTURERMM_SERVER__TRUSTED_PROXIES` = your terminator's address.** Skip
   it and every request resolves to the proxy: each audit row records the proxy
   rather than the actor, and ten failed logins from anywhere return 429 to every
   user. Both failures are silent; the backend warns once when a forwarding
   header arrives from an untrusted address, and that line means this setting is
   missing.
2. **`Strict-Transport-Security` is your terminator's to send.** The backend
   sends HSTS only when it terminates TLS itself.
3. **Forward `Host` unchanged** (nginx: `proxy_set_header Host $host;`).
   Browsers lacking `Sec-Fetch-Site` must send an `Origin` matching `Host`, or
   their forms get 403 `cross-origin request refused`.

List a PostureRMM **proxy relay**'s address here too, so agent audit rows name
the agent, not the relay.

## Bastion in a DMZ

By default the Bastion runs beside the backend at `http://bastion:8300`, sharing
a bearer secret the backend generates on first boot; nothing to do, offline
installs included.

For a split topology (Bastion on a DMZ host, backend/db/admin UI in the
protected zone) the hosts cannot share that volume. Generate one secret with
`openssl rand -hex 32` and put it in the `.env` beside each compose file.

1. **On the DMZ host** — take
   [`docker-compose.bastion.yml`](https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.bastion.yml)
   from the release page (that file alone), then create
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

5. **Firewall** — protected zone → DMZ, outbound only, to the port the Bastion
   (or its reverse proxy) listens on. No inbound rule from the DMZ is required or
   supported; the Bastion never initiates a connection inward.
6. **Cross-zone TLS** — the hop must be HTTPS and the Bastion has no built-in
   TLS termination. Run a reverse proxy on the DMZ host, terminate TLS with your
   own certificate, and forward to the Bastion's plain-HTTP `:8300`. The shared
   secret authenticates the backend; it does not encrypt.

Changing the split URL or secret means updating `.env` and recreating the
backend container on both hosts; during a mismatch the backend fails closed with
a named authentication error.

## Proxy container

A proxy is optional and exists for **segmented networks**: agents that cannot
reach the backend directly point at a proxy in their own segment, which relays
for them, caches downloads, and queues writes durably while the backend is
unreachable. It ships as the release download `docker-compose.proxy.yml`, also in
the offline bundle with its image.

### Registering it

1. In the admin UI, **Proxies → New**. Copy the ID, the one-shot secret and,
   on a self-signed server, the server CA fingerprint.
2. On a Docker host in the segment, fetch it and write `.env` beside it:

```bash
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.proxy.yml
cat > .env <<'EOF'
POSTURERMM_PROXY_BACKEND_URL=https://posturermm.example.com
POSTURERMM_PROXY_ID=<id from step 1>
POSTURERMM_PROXY_SECRET=<secret from step 1>
POSTURERMM_PROXY_BACKEND_CA_PIN=<server CA fingerprint from step 1>
POSTURERMM_PROXY_TLS_HOSTNAME=<proxy-host>
EOF
docker compose -f docker-compose.proxy.yml up -d
```

3. Point that segment's agents at `https://<proxy-host>:8443`, the host named
   in `POSTURERMM_PROXY_TLS_HOSTNAME`. The proxy injects `X-Forwarded-For` and `X-Proxy-ID`, so the audit
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
| `POSTURERMM_PROXY_BACKEND_CA_PIN` | — | Server CA fingerprint from the UI; omit for a real certificate. |
| `POSTURERMM_PROXY_TLS_HOSTNAME` | — | The host agents reach the proxy by; its certificate is issued for it. Required. |
| `POSTURERMM_PROXY_LISTEN_ADDRESS` | `0.0.0.0:8443` | Listener. |
| `POSTURERMM_PROXY_TLS_ENABLED` | `true` | `false` is the plaintext opt-out. See the TLS note below. |
| `POSTURERMM_PROXY_TLS_CERT` / `_KEY` | — | Your own certificate chain and key, served instead of the minted one. |
| `POSTURERMM_PROXY_LOG_LEVEL` | `info` | |
| `POSTURERMM_PROXY_CONFIG` | `./config/posturermm-proxy.toml` | Optional TOML, mounted by you. |

**TLS.** The proxy serves HTTPS from a CA it mints and keeps in the
`posturermm-proxy-tls` volume; agents pin it, and its page in the admin UI shows
the **TLS CA fingerprint**. Losing the volume means re-pinning every agent behind
the proxy. `TLS_ENABLED=false` is only for a proxy behind your own TLS
terminator. With no `BACKEND_CA_PIN` the proxy trusts only a public certificate,
and changing the server's CA means re-pinning every proxy.

**Liveness.** `GET /health` answers **503** with
`{"status":"degraded","backend_reachable":false}` when the backend is
unreachable; treat that as "backend down", not "proxy down", since serving from
cache is correct then. A proxy that misses its 60s heartbeat shows as stale in
the UI.

## Agent rollout

1. **Create an install key** — Admin UI → Settings → Enrollment → Create key.
   The key is optional and only skips the approval queue, which unattended
   rollouts need; without it the one-liner still enrolls, inert, until you admit
   the endpoint from Endpoints.
   `POSTURERMM_SERVER__ALLOW_AUTO_ENROLLMENT=true` admits keyless enrollments
   instantly: a trusted-LAN convenience, an open door on anything
   internet-reachable.
2. **Run the one-liner** on the target Windows endpoint (elevated PowerShell):
   ```powershell
   irm "http://posturermm.example.com/install.ps1?key=..." | iex
   ```
   The shim writes `C:\ProgramData\PostureRMM\install-key.txt`, downloads the
   MSI, and runs `msiexec /i /qn SERVERURL=https://posturermm.example.com`. The
   MSI registers and starts the agent service; the agent reads the key file on
   first boot, enrolls, and self-cleans it.

**Where the MSI on the server comes from.** You never place it. An **offline
install** gets it from the bundle; an **online install** from the product feed on
the Bastion's first pull. Until then Endpoints → Install first endpoint shows
*Fetching agent v…* and offers no install command. Behind a maintenance window
that lasts until the window opens; the request is queued durably.

For SCCM / Intune / GPO flows, take the MSI URL and checksum from Admin UI →
Endpoints → Install agent → More install options, pass `SERVERURL` on the
`msiexec` command line, and drop `install-key.txt` beside the MSI.

### Linux endpoints

Debian 12/13, Ubuntu 22.04/24.04 LTS, RHEL, Rocky Linux and AlmaLinux 9/10, on
x86_64 with systemd. Run the install screen's command as a user who can `sudo`:

```bash
curl -fsSL "http://posturermm.example.com/install-agent.sh?key=..." | sudo sh
wget -qO- "http://posturermm.example.com/install-agent.sh?key=..." | sudo sh   # no curl
curl -fsSL http://posturermm.example.com/install-agent.sh | sudo sh            # keyless
```

It is the same plain-HTTP bootstrap as Windows, with its
[trust caveat](#tls-and-agent-trust). The script refuses an unsupported host,
checks the package's SHA-256, writes `/etc/posturermm/agent.conf` (mode 0600) and
the install key, and installs the `.deb` or `.rpm`, which starts
`posturermm-agent.service`. Re-run over the same version, it re-writes the
server settings and restarts the agent.

The console offers the command once both packages are on the server, from the
product feed or offline bundle.

| Path | Holds |
|---|---|
| `/etc/posturermm/agent.conf` | `ServerUrl`, `TrustMode`, `ServerCertPin` and the [local policy](#local-endpoint-policy) switches, as `Name=Value` lines |
| `/var/lib/posturermm` | Credentials and state; `upgrade/` keeps one previous package for rollback |
| `/var/cache/posturermm` | Rebuildable cache: the agent's private apt or dnf tree |
| `/var/log/posturermm/agent.log` | The agent log. Panics also reach the journal: `journalctl -u posturermm-agent` |

A failed upgrade rolls back to the kept package. `apt purge posturermm-agent` or
`dnf remove posturermm-agent` deletes config, state and cache and keeps the
logs; `apt remove` keeps everything.

Not on Linux: the tray companion, remote desktop and tamper protection. Scripts
run in `sh` or `bash`. Updates are the distribution packages the server carries,
installed from the agent's private source list, never the host's repositories.
RHEL itself, and AlmaLinux 10 on a CPU without x86-64-v3, are scanned but not
patched.

### Local endpoint policy

| | Windows | Linux |
|---|---|---|
| Refuse scripts | `msiexec /i … REFUSESCRIPTS=1`, or `RefuseScripts` = `1` (REG_SZ or REG_DWORD) under `HKLM\SOFTWARE\PostureRMM\Agent` or the Group Policy key `HKLM\SOFTWARE\Policies\PostureRMM\Agent` | `RefuseScripts=1` in `/etc/posturermm/agent.conf`, before or after the install script |
| Refuse the remote terminal | `REFUSETERMINAL=1`, or `RefuseTerminal` in the same keys | `RefuseTerminal=1` in the same file |
| Refuse remediation | `REFUSEREMEDIATION=1`, or `RefuseRemediation` in the same keys. Every compliance fix and undo; the endpoint is still scanned and scored | `RefuseRemediation=1` in the same file |
| Who can change it | Administrators. No server setting, task or upgrade overrides it (ADR-0161) | root |
| Effect | The agent refuses at the point of execution, from the next use, with no restart. From the next check-in the console stops offering it (Run script, the terminal, Fix, Apply all safe fixes) and says the endpoint's local policy refuses it | Same |
| Allowing again | `0`, `false`, `no`, `off` or blank. Any other value in any of those places refuses, and so does one the agent cannot read | Same |

### Changing the agent's backend URL

The agent reads `ServerUrl` at boot from
`HKLM\SOFTWARE\PostureRMM\Agent\ServerUrl` on Windows, or from
`/etc/posturermm/agent.conf` on Linux, and ignores TOML and env. Change it
one of two ways:

1. **Per-machine** — elevated PowerShell on the endpoint:
   ```powershell
   posturermm-agent.exe config set server.url https://posturermm.example.com
   posturermm-agent.exe config get     # sanity check
   ```
   Validates the URL, writes the registry value, restarts `PostureRMM-Agent`
   and emits an `audit.config` log line. On Linux, as root:
   ```bash
   sudo posturermm-agent config set server.url https://posturermm.example.com
   ```
   It writes `ServerUrl` into `agent.conf`, keeping every other line, and
   restarts `posturermm-agent`.

2. **Fleet-wide (SCCM / Intune / GPO)** — push an explicit reinstall:
   ```cmd
   msiexec /x posturermm-agent.msi /qn
   msiexec /i posturermm-agent.msi SERVERURL=https://new-host.example.com /qn
   ```
   A command-line `SERVERURL=` always wins, on upgrade as on fresh install; omit
   it and the MSI recovers the operator's existing value rather than stranding
   the agent. On Linux, run the new server's one-liner: it re-writes the server
   settings in `agent.conf` and restarts the agent.

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
# Every `up -d` first runs the one-shot db-setup as the database superuser
# (`posturermm`, whose password only `db` and `db-setup` receive). It creates
# `posturermm_owner` (no SUPERUSER, BYPASSRLS, CREATEROLE or CREATEDB), gives it
# the database and everything in it, creates `posturermm_app` with DML grants
# only, and writes both generated passwords to /app/data/pgpass. The backend
# migrates as the owner, serves requests as the app role and refuses a
# superuser, so an install from before db-setup converts on its first `up -d`.
# On your own PostgreSQL, give the backend two such roles: DATABASE_OWNER_URL
# owning its database and a member of pg_read_all_stats, DATABASE_URL owning
# nothing. The pgcrypto extension must be available.
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.yml
docker compose pull
docker compose up -d

# Drop into the DB shell, as the superuser
docker compose exec db psql -U posturermm posturermm
```

Read the target version's release notes: a release needing a one-time manual
step says so there, and the backend itself refuses to start, names what it needs
and prints the recipe.

### Verifying a release

`SHA256SUMS.txt` lists every release file, SBOMs included. In the directory
holding your downloads:

```bash
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/SHA256SUMS.txt
sha256sum -c --ignore-missing SHA256SUMS.txt
```

Each release attaches CycloneDX 1.5 SBOMs: `sbom-<binary>-<version>-<target>.cdx.json`
per binary and `sbom-image-<name>-<version>.cdx.json` per image, which also
lists its Debian packages. Load them into your scanner, or read one directly:

```bash
jq -r '.components[] | "\(.name) \(.version)"' sbom-image-backend-X.Y.Z.cdx.json
```

A binary with no SBOM beside it embeds its dependency list; the exit code is
non-zero when a RustSec advisory matches:

```bash
cargo install cargo-audit --locked
cargo audit bin --max-binary-size 1000000000 posturermm-agent-X.Y.Z.exe
```

### Pushing the images into a private registry

Never required: `docker load` from the offline bundle on each host is the
supported path. A site running Harbor, Artifactory or Nexus can seed one from the
same bundle:

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

Then point the image refs at the internal registry.

### Prometheus metrics

`/metrics` is served on the backend's internal `:3000` plane only, which is
never published to the host; `:443` and `:80` 404 it. Scrape it from the same
Compose network (`docker network connect posturermm_default <your-scraper>`):

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

With `POSTURERMM_TLS__ENABLED=false` the single `:3000` listener carries
`/metrics`; do not proxy that path to the public side.

### The backend turns PostgreSQL's JIT off

Every pooled connection runs `SET jit = off` (so a `psql` shell still reports
`on`), against your own PostgreSQL too. JIT was the only variable here:

| query | with JIT | without |
|---|---|---|
| compliance fleet rollup | 940 ms | 191 ms |
| patching fleet queue | 178 ms | 72 ms |

No query was faster with JIT on.

### Data retention

Settings → Retention (SuperAdmin) controls four windows, all enforced daily by
the same scheduler job:

| Setting | Default | Notes |
|---|---|---|
| Task history | 90 days | Terminal `tasks` rows. Deployment stats are materialized to the parent `deployments` row first, so fleet-level history survives the prune. |
| Compliance snapshot | 365 days | Daily per-endpoint compliance percentage, feeds trend charts and evidence packs. |
| Posture snapshot | 365 days | Daily per-endpoint posture score, feeds fleet sparklines. Same shape and growth rate as the compliance snapshot, and the same default window: **upgrading an existing deployment begins pruning rows older than 365 days immediately**. There is no unlimited option for this window. |
| Audit log | **0 (keep forever)** | The one window that defaults to unlimited, and the only one of the four with an unlimited option at all, so upgrading never starts deleting an existing operator's history on its own. Set a positive number of days to opt into pruning. |

The first three accept 30–3650 days; audit-log retention accepts 0–3650. Pruning
runs in bounded batches, and pruning the audit log writes one more audit row
(`audit_log_prune`) naming the row count and the cutoff. Rows a configured audit
sink has not yet received are never pruned; the run logs how many.

A run that cannot read all four windows prunes nothing and logs the key at
fault, rather than falling back to the defaults. Save a real number on the
Retention page and the next daily run recovers.

Three telemetry histories are pruned hourly on fixed windows you do not set:
performance metrics and endpoint reachability at **30 days**, disk SMART history
at **2 years**.

### Forwarding the audit log

Settings → System → Audit forwarding (SuperAdmin) sends every audit row to one
syslog receiver: RFC 5424 over TLS, RFC 5425 octet-counted framing, standard port
6514. Rows go in order and queue while the receiver is down. Each is one JSON
object (facility 13, severity notice, APP-NAME `posturermm`, MSGID the action):

```
<109>1 2026-10-05T11:07:37.123456Z rmm.example.com posturermm - login_success - {"seq":42,"instance_id":"…","organization":{"id":"…","name":"…"},"id":"…","timestamp":"…","action":"login_success","user_id":…,"user_email":…,"resource_type":…,"resource_id":…,"details":…,"ip_address":…,"user_agent":…}
```

The receiver's certificate must name the host you enter in a subject alternative
name; a common name alone is refused. Paste the CA that signed it, or leave the
box blank for a public CA. **Send test message** tries the form before you save
it (action `audit_forwarding_test`, `seq` 0). Every recipe below terminates the
TLS in rsyslog.

#### rsyslog

Install `rsyslog-gnutls` and add `/etc/rsyslog.d/posturermm.conf`, which writes
one JSON object per line:

```
global(
  DefaultNetstreamDriver="gtls"
  DefaultNetstreamDriverCAFile="/etc/ssl/posturermm/ca.pem"
  DefaultNetstreamDriverCertFile="/etc/ssl/posturermm/server.pem"
  DefaultNetstreamDriverKeyFile="/etc/ssl/posturermm/server.key"
)
module(load="imtcp" StreamDriver.Name="gtls" StreamDriver.Mode="1"
       StreamDriver.AuthMode="anon")
template(name="posturermm-json" type="string" string="%msg%\n")
ruleset(name="posturermm") {
  action(type="omfile" file="/var/log/posturermm-audit.json" template="posturermm-json")
}
input(type="imtcp" port="6514" ruleset="posturermm")
```

To require a client certificate, paste one with its key into the form and set
`StreamDriver.AuthMode="x509/name" PermittedPeer="<name in that certificate>"`;
the global CA file must be the CA that signed it.

#### Splunk

Run the rsyslog recipe and have a Splunk forwarder on that host read the file
(Splunk's TCP inputs do not unwrap RFC 5425 framing):

```
[monitor:///var/log/posturermm-audit.json]
sourcetype = _json
index = posturermm
```

#### Microsoft Sentinel

Sentinel reads syslog through Azure Monitor Agent (AMA) on a Linux forwarder.
Give that forwarder a data collection rule with a **Linux Syslog** source
(facility `audit`, minimum level Notice) and the rsyslog `global()` and
`module()` lines above. AMA forwards only rsyslog's default ruleset, so drop the
template and ruleset and bind the input to it: `input(type="imtcp" port="6514")`.

Rows land in the `Syslog` table with the JSON in `SyslogMessage`; ingestion
strips semicolons from `user_agent`, and the JSON stays valid.

```kusto
Syslog
| where SyslogMessage has "\"instance_id\""
| extend j = parse_json(extract(@"(\{.*\})", 1, SyslogMessage))
| project TimeGenerated, action = tostring(j.action), user = tostring(j.user_email), details = j.details
```

#### Wazuh

Wazuh's syslog listener has no TLS. Run the rsyslog recipe on the Wazuh server or
an agent host and read the file:

```
<localfile>
  <log_format>json</log_format>
  <location>/var/log/posturermm-audit.json</location>
</localfile>
```

No stock rule matches this stream; write rules on the JSON fields.

#### Spotting a gap, and re-sending

`seq` is gapless per `instance_id`, so a hole at the receiver is a row it lost
(TCP has no acknowledgement). Numbering starts at the first row after you save
the sink, a repeat after a retry is normal, and `seq` 0 is the test message. On
an rsyslog host:

```bash
jq -r 'select(.seq > 0) | "\(.instance_id) \(.seq)"' /var/log/posturermm-audit.json |
  sort -u -k1,1 -k2,2n |
  awk '$1==i && $2!=p+1 {print i": missing "p+1"-"$2-1} {i=$1; p=$2}'
```

In Sentinel:

```kusto
Syslog
| where SyslogMessage has "\"instance_id\""
| extend j = parse_json(extract(@"(\{.*\})", 1, SyslogMessage))
| project instance = tostring(j.instance_id), seq = tolong(j.seq)
| where seq > 0
| distinct instance, seq
| sort by instance asc, seq asc
| extend prev_instance = prev(instance), prev_seq = prev(seq)
| where instance == prev_instance and seq - prev_seq > 1
| project instance, missing_from = prev_seq + 1, missing_to = seq - 1
```

To fill a hole, choose **Re-send from a sequence number** on the page's health
panel and enter the first missing number. Every row from there on goes out again,
duplicates included, and the re-send is itself audited. The panel also shows the
backlog and last error; the `audit_forwarding_stalled` alert fires when rows have
waited and nothing has been sent for 30 minutes.

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
boot: the UI looks healthy while enrolled agents reject the new certificate and
TOTP enrollments and backup codes stop working.

Run the backup from the directory holding `docker-compose.yml` and `.env`. It
briefly stops the backend so its volumes cannot change mid-archive, and works
from an offline bundle too.

```bash
# Create one private directory for the entire backup set.
BACKUP_DIR="$(pwd)/posturermm-backup-$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -m 700 "$BACKUP_DIR"
cp --preserve=mode .env "$BACKUP_DIR/.env"

# Record the CA fingerprint now, so the restore has something to compare against.
docker compose exec -T backend cat /app/tls/ca.crt |
  openssl x509 -noout -fingerprint -sha256 > "$BACKUP_DIR/ca-fingerprint.txt"

# Quiesce the service that mounts the volumes being archived.
docker compose stop backend

# Back up PostgreSQL logically; do not copy its live data-directory volume.
# The dump lands under a .part name and is renamed only once its completion
# marker is present. A dump that dies part-way — the database stopped, the disk
# full — leaves no posturermm.sql at all, so every step below fails loudly
# instead of checksumming and shipping a backup that restores to nothing.
docker compose exec -T db pg_dump -U posturermm posturermm \
  > "$BACKUP_DIR/posturermm.sql.part"
tail -n 5 "$BACKUP_DIR/posturermm.sql.part" | grep -q 'PostgreSQL database dump complete' &&
  mv "$BACKUP_DIR/posturermm.sql.part" "$BACKUP_DIR/posturermm.sql"

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

# Make later corruption detectable, then return the application to service.
# These sums cover only what happens to the files after this point; the dump's
# own completeness was settled above, by the rename.
(
  cd "$BACKUP_DIR"
  sha256sum .env posturermm.sql posturermm-data.tar.gz \
    posturermm-tls.tar.gz ca-fingerprint.txt > SHA256SUMS
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
| `posturermm-data.tar.gz` | `posturermm-data`: the auto-provisioned `jwt-secret`, `totp-secret`, and other backend-persisted secret material | Existing access tokens no longer validate. More importantly, stored TOTP secrets cannot be decrypted and MFA backup-code hashes no longer verify, locking 2FA users out. Agents cannot re-anchor to a new CA until each checks in again, so losing this with `posturermm-tls` strands them. |
| `.env` | `POSTGRES_PASSWORD` and any operator-supplied `JWT_SECRET` or `POSTURERMM_CREDENTIAL_KEY` | PostgreSQL credentials no longer match, explicitly supplied JWT keys rotate, and stored credentials/evidence-signing keys encrypted under the credential key become unusable. Neither value is in `pg_dump`. |

Four volumes are deliberately excluded; none holds authoritative data. Three are
caches: `posturermm-bastion-data`, `posturermm-binary-store` (refetchable
installer blobs; an offline deployment may need to repopulate them) and
`posturermm-downloads`. The fourth, `posturermm-bastion-auth`, is regenerated on
first boot, or comes from `POSTURERMM_BASTION__SECRET` in `.env` when the Bastion
is DMZ-split.

Restore only onto a fresh host with no existing PostureRMM containers or named
volumes, and **do not run `docker compose up` yet**: a backend booted against
empty volumes silently writes new secrets and a new CA, which a later restore
cannot undo.

Fetch the compose file for the release the backup came from — not `latest`,
which would restore an older dump under a newer backend — and put the backup
directory beside it:

```bash
mkdir posturermm && cd posturermm
curl -LO https://github.com/PostureRMM/PostureRMM/releases/download/vX.Y.Z/docker-compose.yml
```

Agents reach this server by the address baked into their install, so give the
replacement host the lost host's address before you restore.

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

# Restore PostgreSQL while the backend is still stopped. db-setup first: the
# dump hands its objects to the backend's role, which must exist.
docker compose start db
until docker compose exec -T db pg_isready -U posturermm -d posturermm \
  >/dev/null; do sleep 2; done
docker compose run --rm db-setup &&
docker compose exec -T db psql -v ON_ERROR_STOP=1 -U posturermm posturermm \
  < "$BACKUP_DIR/posturermm.sql" &&

# Only now may the backend boot; bring the application services back together.
# Chained to the load above on purpose: a backend that boots over a half-loaded
# database migrates it, and the restore can no longer simply be repeated.
docker compose up -d
until docker compose exec -T backend curl -sf http://localhost:3000/health/ready \
  >/dev/null; do sleep 2; done
docker compose ps
```

After restore, verify an existing login and a 2FA-enabled account before
declaring recovery complete, and check the restored CA against the fingerprint
the backup recorded — if they differ, the backend booted before the TLS volume
was populated and every enrolled agent is about to stop connecting:

```bash
docker cp posturermm-backend:/app/tls/ca.crt ./restored-ca.crt
openssl x509 -in ./restored-ca.crt -noout -fingerprint -sha256
cat "$BACKUP_DIR/ca-fingerprint.txt"
rm ./restored-ca.crt
```

Then confirm an enrolled agent checks in without a reinstall.

## Troubleshooting

- **Backend won't start:** `docker compose logs backend`. Migrations log themselves on startup.
- **502 from your own upstream proxy:** the backend container is down or not ready; `docker compose ps` should show it `healthy`.
- **Backend not serving HTTPS:** verify the `posturermm-tls` volume is mounted and `server.crt` / `server.key` exist.
- **UI loads blank / 404:** the backend is not serving traffic yet.
- **Agent can't connect:** verify DNS resolution, port 443 reachable, cert valid. `POSTURERMM_SERVER__PUBLIC_URL` must match what the agent sees.
- **Bastion returning 401 to the backend:** the bearer secret disagrees. Both sides must hold the same 32+ character `POSTURERMM_BASTION__SECRET` (or, unset on the Bastion, the file at `[bastion] secret_path`).
- **Bastion returning 502 on feed pulls:** it could not reach the feed origin. Check egress from the DMZ host to `[feed] api_url` (default `https://feed.posturermm.com`) and `docker compose logs bastion`.
- **Lost the admin password:** `/app/data/bootstrap-admin-password` is one-time — once changed, the next restart overwrites it with a note saying so. Reset from the Docker host:

  ```bash
  docker compose exec backend ./posturermm-backend reset-admin-password admin@localhost
  ```

  The new password goes to your terminal's stdout, never the container log, and the next login must change it. Live sessions end; API keys survive. It is **deliberately unauthenticated** — anyone who can `docker compose exec` here can already read the JWT secret and write the `users` table — and needs the backend running.

  **Authenticator and backup codes lost too?** Add `--clear-2fa`: it deletes the account's TOTP enrolment and backup codes and writes a `totp_admin_reset` audit row. Without it, 2FA is untouched.

---

Back to the [README](../README.md), the [quickstart](quickstart.md), or
[configuration.md](configuration.md).

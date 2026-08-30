# PostureRMM Production Deployment

## Architecture

```
Browser / Agent → HTTPS/443 → backend → db:5432
                                (terminates TLS and serves /api, /health,
                                 /install, /downloads, /winget, and the
                                 MiniJinja + HTMX admin UI at /)
HTTP bootstrap → HTTP/80 ───────┘

bastion → HTTPS → Feed API (Cloudflare Workers + R2)
bastion → HTTP  → backend:3000 (admin API, status, sync)
```

Three containers via Docker Compose: `posturermm-db`, `posturermm-backend`, and `posturermm-bastion`. The backend terminates TLS and serves both the JSON API and the server-rendered admin UI — there is no separate web or reverse-proxy tier. It publishes HTTPS on `:443` and an HTTP listener on `:80` that redirects to HTTPS except for the plaintext `/install.ps1` and `/downloads/*` bootstrap paths. Its plain-HTTP `:3000` plane is internal to the Compose network for the Bastion and container healthcheck and is not published to the host. The backend makes zero outbound internet connections; only the Bastion talks to the Feed.

## Prerequisites

- Linux on amd64 or arm64 with Docker Engine **20.10.0 or newer** and Docker
  Compose **2.0.0 or newer**. The stack relies on Compose-Spec required-value
  interpolation and health-conditioned dependencies; legacy Python Compose
  v1 is not supported.
- At least **4 GiB RAM** visible to Docker and **10 GiB free disk**. The RAM
  floor is measured: the first feed sync peaks at 1.5 GiB above the OS,
  Docker and PostgreSQL shared buffers.
- TCP ports 443 and 80 available by default. The conflict occurs at Docker's
  publish layer before the backend starts. Set `POSTURERMM_HTTPS_PORT=8443` to
  remap HTTPS. To put an existing host reverse proxy at the public edge,
  disable the backend TLS listeners with `POSTURERMM_TLS__ENABLED=false` and
  proxy to its plain-HTTP `:3000` plane instead.

Do not use `apt install docker-compose` as a version check: Ubuntu 22.04/24.04
and Debian 12 provide legacy v1 there, while Debian 13 provides v2. The
preflight probes both `docker compose` and `docker-compose`, selects the newer
implementation, and prints repository instructions specific to the host OS
when either Docker or Compose is missing or too old. It never installs Docker.

## License

PostureRMM is proprietary software, licensed under the
[End-User License Agreement](../LICENSE) shipped with every release. Installing
or running it accepts those terms. Community is free and perpetual up to 50
endpoints; a paid entitlement raises the cap. Only builds published by us —
GitHub Release assets and container images in our own GHCR namespace — are
covered, so pull the images and assets the Quick Start below names and nothing
else.

## Quick Start

**You edit nothing.** No values to choose, no editor, no `openssl`, no DNS.
Install is pull-only — no build tools, no crates.io/Docker Hub
access beyond pulling the published images, nothing compiled on your hardware.

```bash
mkdir posturermm && cd posturermm
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.yml
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/preflight.sh
chmod +x preflight.sh
./preflight.sh
```

First boot writes the generated `admin@localhost` password to
`/app/data/bootstrap-admin-password`, mode 0600, and logs that path rather than
the value — JSON logging is on, so a credential the log sees once it keeps. Read
it with `docker compose exec backend cat /app/data/bootstrap-admin-password`,
then log in at `https://<host>/`.

Fetch the compose file **from the release, not from `main`** — that is what
makes the version question disappear. A release's compose has the version it
shipped with baked in as the default; a copy taken from anywhere else belongs to
no release and deliberately refuses to start without one. `/releases/latest/` resolves to
the newest release, which is the only supported one; to install a
specific version, swap `latest/download` for `download/v<version>`.

What the preflight does, in order: checks the host (architecture, Engine and
Compose versions, disk, container-visible RAM, port 443), **seeds
`POSTGRES_PASSWORD` and `POSTURERMM_DOMAIN` into `.env`**, pulls any image it
does not have locally, then starts the stack. Everything else self-provisions on
first boot — JWT secret, TOTP key, the TLS CA and leaf, and the backend↔Bastion
shared secret.

Both seeded values are **write-once**; a re-run rewrites neither. The password,
because Postgres initialises its volume with the value it first sees. The domain
— this host's primary address, read out of the routing table — because it names
the server in every agent install command; unset, that name is `localhost`.

`.env` is a knob surface, not an install step. Take the optional
[`env.example`](https://github.com/PostureRMM/PostureRMM/releases/latest/download/env.example)
from the same release if you want a real DNS name, a non-default port, or
externally-managed keys — every line in it is commented out, because every line
in it is optional.

Once every image is local the preflight hands off with
`docker compose up -d --pull never`, so startup cannot block on a registry. For
a fully offline install, use the bundle below instead — it is this same path
with `docker load` in front of it.

Backend runs migrations automatically on startup. On first boot, it generates a self-signed CA and leaf cert, writes them to the `posturermm-tls` volume, and serves the chain directly — no domain registration or ACME setup required.

## Offline / air-gapped install

For a target host with no internet, every release attaches a self-contained
bundle `posturermm-offline-vX.Y.Z.tar.gz` to its GitHub Release, next to the
agent MSI/EXEs and the online install assets. It carries a `docker save` of exactly the three images the
shipped compose pins (backend, bastion, and digest-pinned `postgres`), so a
`docker load` is byte-for-byte what an online `docker compose pull` would
fetch — the offline path is the online path with one command in front.

On an internet-connected machine:

```bash
# Download the bundle + its checksum from the release, verify, then move it
# across (SFTP, USB, or whatever crosses your air gap).
curl -LO https://github.com/PostureRMM/PostureRMM/releases/download/vX.Y.Z/posturermm-offline-vX.Y.Z.tar.gz
curl -LO https://github.com/PostureRMM/PostureRMM/releases/download/vX.Y.Z/SHA256SUMS.txt
sha256sum -c SHA256SUMS.txt --ignore-missing
```

On the air-gapped target (Docker + the Compose v2 plugin already installed):

```bash
tar xzf posturermm-offline-vX.Y.Z.tar.gz
cd posturermm-offline-vX.Y.Z
./install.sh        # docker load images.tar, then seeds .env from .env.example
./preflight.sh      # checks the host, generates the DB password, starts without pulling
```

Nothing to edit here either: the bundled `docker-compose.yml`, `.env.example`
and `preflight.sh` are byte-identical to the online quickstart's — same
release, so the version is already baked in and the database password is
generated the same way.

`install.sh` never touches the network: `docker load` populates the local
image store so the compose pull-only refs resolve locally, and the bundled
agent installers are copied straight into the downloads volume. The product installs
and enrolls after load — the agent is in the tarball (deliberately *not* in the
backend image), so first login → first endpoint enrolled needs zero
fetches. **Benchmarks are not**: they arrive through the feed, and
Prevention reads *not yet measured* until the first sync.
Upgrades are symmetric: load the next release's `images.tar`, swap in that
release's `docker-compose.yml`, `docker compose up -d`. (A
`POSTURERMM_VERSION` set in `.env` overrides the compose default — bump it
too, or drop the line.)

### Optional: pushing the bundle into a private registry

A private registry is **never required** — `docker load` on each host is the
supported path. But sites that already run Harbor / Artifactory / Nexus can
seed it from the same bundle so hosts pull from the internal registry instead:

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

Then point `POSTURERMM_VERSION` / the image refs at the internal registry.
This is documented convenience, not product surface.

## Configuration (.env)

**Nothing in `.env` is required.** The release's compose carries its own
version, the preflight generates the database password, and the backend
self-provisions its keys and certificate at boot — so this is the surface for
knobs you *choose* to turn, and an install that turns none is a supported
install. The shipped `.env.example` is a short starter carrying only the
settings most installs touch, and
[configuration.md](configuration.md) covers the ones most installs reach for.
This table is the full list:

| Variable | Required | Purpose |
|---|---|---|
| `POSTURERMM_VERSION` | baked | Released version tag to pull for the backend and bastion images — e.g. `1.0.0`, matching a `v1.0.0` GitHub Release (one version covers backend, agent, bastion and proxy). The release's `docker-compose.yml` and `.env.example` both ship it baked in, so a fresh install never types it; a copy taken from anywhere but a release hard-requires it, because it belongs to no release. No `latest` fallback — under `restart: unless-stopped` a floating tag would silently upgrade the stack on container recreation. Upgrading is the explicit act of downloading the new release's compose file; set this key only to pin a version against whatever compose file you hold, and note that it then overrides the baked default until you remove it. |
| `POSTGRES_PASSWORD` | auto | Postgres password for the `posturermm` role. Generated write-once by `preflight.sh` (64 random hex chars) and never rewritten: the volume is initialised with the first value it sees, so a later change locks the backend out. Postgres publishes no host port — it is reachable only on the Compose network. Set it by hand only for an externally-managed credential, and only before the first start. |
| `POSTURERMM_DOMAIN` | seeded | Hostname agents and browsers use to reach the server. `preflight.sh` writes this host's primary address here on first run, write-once; unset, the server calls itself `localhost` and aims agent installs at the endpoint. Drives the backend certificate SAN and the public URL embedded in agent installs. A DNS name or bare IP; configure the port separately. |
| `POSTURERMM_HTTPS_PORT` | optional | Backend HTTPS publish port (default `443`). A non-default value is also included in the derived agent public URL. |
| `POSTURERMM_HTTPS_BIND` | optional | Host address for the backend HTTPS publish. Unset listens on all host addresses. |
| `POSTURERMM_TLS__ENABLED` | optional | Enables the backend's built-in TLS and `:80`/`:443` listeners (default `true`). Set `false` only when an operator-managed TLS terminator fronts the backend's plain-HTTP `:3000` plane. |
| `JWT_SECRET` | auto | Self-provisioned on first boot — backend writes a random 64-char hex to `/app/data/jwt-secret` in the `posturermm-data` volume and reuses it thereafter. Set explicitly only to rotate or use an externally-managed key. |
| `POSTURERMM_AUTH__TOTP_SECRET` | auto | Dedicated key for encrypting TOTP secrets at rest and peppering MFA backup-code hashes. Self-provisioned on first boot to `/app/data/totp-secret`, independent of `JWT_SECRET` so rotating the JWT secret does not brick stored 2FA secrets. Set explicitly only to rotate or externally manage. |
| `LOG_LEVEL` | optional | `trace \| debug \| info \| warn \| error` (default `info`) |
| `POSTURERMM_AUTH__REMOTE_SESSION_REAUTH_INTERVAL_SECS` | optional | How often an **already-open** remote session (terminal, RDP, screen share, file transfer, live script output) re-checks that the operator driving it is still authorized — still active, still in an active Organization, still holding the role and group scope (default `20`). Deactivating a user *also* pushes an immediate teardown of their live sessions, so this is the worst-case window for a session that raced that sweep, plus the mechanism behind revocations that have no push door: deletion, demotion, Organization deactivation. Cost is one indexed lookup per live socket per tick. Lower it for a tighter window; there is little reason to raise it. |
| `POSTURERMM_SERVER__PUBLIC_URL` | escape hatch | Overrides the agent-install URL when the external URL differs from `POSTURERMM_DOMAIN` (Cloudflare tunnel, reverse proxy, split DNS). Defaults to `https://${POSTURERMM_DOMAIN}:${POSTURERMM_HTTPS_PORT:-443}`. |
| `POSTURERMM_SERVER__CORS_ORIGINS` | optional | Comma-separated browser origins allowed to call this server cross-origin. Default is *empty*, which means **same-origin**: the origin of `POSTURERMM_SERVER__PUBLIC_URL` is the whole allow-list, which is all the admin UI needs, and every other origin is refused. Set it only when a browser app you host elsewhere must call this server — e.g. `https://tools.example.com,http://localhost:5173`. A single `*` is the explicit opt-in to allowing *every* origin; it warns on every boot and cannot carry session cookies (the CORS spec forbids credentials with a wildcard), so a bearer-token caller is the only thing it enables. |
| `POSTURERMM_SERVER__TRUSTED_PROXIES` | **required behind a proxy** | Comma-separated CIDRs/IPs whose `X-Forwarded-For` / `X-Real-Ip` the backend believes when stamping the audit log's source IP. Default is *empty*: trust nothing, record the socket peer. Correct for the shipped direct-edge topology; **mandatory** the moment a reverse proxy fronts the backend — see "Source IP in the audit log" below. |
| `POSTURERMM_BASTION__URL` | optional | URL the backend uses to reach the Bastion. Defaults to the co-located compose-network neighbour, `http://bastion:8300`. Set to retarget a DMZ-split Bastion (see "Bastion in a DMZ" below). |
| `POSTURERMM_BASTION__STREAM_PACING_MS` | optional | Pause between per-stream feed imports on a pull tick (default `2000`). A fresh install's first tick is a full backfill — ~26 NVD year-streams, ~372k rows — and running them back to back outpaces Postgres checkpointing and kernel writeback on a memory-constrained host. At 38 streams the default costs ~76s on a worst-case backfill and nothing in the steady state, since unchanged streams are skipped by content hash before the pause. Raise it if a small host still struggles during a backfill; `0` disables pacing. |
| `POSTURERMM_BASTION__COMMIT_PACING_MS` | optional | Pause between the bounded commits *within* one feed stream, in milliseconds (default `500`). `STREAM_PACING_MS` above paces the gaps between streams; this paces the gaps inside one. A first tick moves the whole corpus — roughly a million rows across the NVD, EPSS and MSRC streams — in 5,000-row commits, and running those back to back produces dirty pages faster than a 4 GiB host writes them back, which stalled such a host outright with no OOM kill and no error. The pause sits *between* commits, so a stream small enough to commit once — every steady-state delta — never reaches it: the default costs the first sync roughly 95 seconds and every later sync nothing. Raise it if a host at the 4 GiB floor still struggles during its first sync; `0` disables pacing. |
| `POSTURERMM_FEED__API_URL` | optional | Feed origin the **Bastion** pulls manifests, stream blobs and binaries from. Unset uses the shipped origin; set it only to point at your own mirror. |
| `POSTURERMM_FEED__DEPLOYMENT_ID` | split Bastion only | Identity the Bastion sends as `X-Deployment-Id`, empty by default. **The Feed rejects an empty one with 401.** On the co-located stack you leave it unset: the backend mints and persists a UUID on first boot and stamps it on every Bastion request, which the Bastion forwards. A DMZ-split Bastion has no backend beside it to mint one, so `docker-compose.bastion.yml` refuses to start without it — take the UUID from the backend's boot log. |
| `POSTURERMM_FEED__TIER` | optional | Feed channel: `community` \| `pro` (default `community`). |
| `POSTURERMM_FEED__SYNC_INTERVAL_HOURS` | optional | How often the Bastion checks the Feed for a new manifest (default `1`). |

`DATABASE_URL` is constructed automatically by the backend container from `POSTGRES_PASSWORD` and the Compose network — you do not set it manually.

`NVD_API_KEY` and `GITHUB_TOKEN` are **not** backend env vars. The backend consumes pre-built Feed bundles from the Bastion container and never calls NVD or GitHub directly. Neither variable does anything if you set it.

### Source IP in the audit log

Every audit row records the client's IP. Which address that is depends on **who the backend's socket peer is**:

- The peer matches `POSTURERMM_SERVER__TRUSTED_PROXIES` → it is a proxy relaying for someone, so its forwarding headers are believed: `X-Real-Ip`, else the **right-most** `X-Forwarded-For` hop (the one the proxy appended and the client could not prepend).
- The peer does **not** match → the peer *is* the client, and any forwarding header on the request is ignored. This is what makes the column trustworthy: without it, a client talking straight to the backend could set `X-Forwarded-For: 1.2.3.4` and write its own source IP into your forensic record. Believing such a header is invisible rather than merely wrong, which is why the empty default stands.

**Running the bundled compose stack? Nothing to do** — clients reach the backend's own `:443`/`:80`, so the socket peer is the real client.

**Fronting the backend with your own reverse proxy? Two settings are required, not optional:**

1. **`POSTURERMM_SERVER__TRUSTED_PROXIES` = your terminator's address.** Skip it and every request resolves to the proxy: each audit row records the proxy rather than the actor, and the per-IP login limiter collapses to one shared bucket, so ten failed logins from anywhere return 429 to every user. Both failures are silent. The backend warns once when a forwarding header arrives from an address it does not trust; that line means this setting is missing.
2. **`Strict-Transport-Security` is your terminator's to send.** The backend emits HSTS only when it terminates TLS itself, so under `POSTURERMM_TLS__ENABLED=false` it sends none — a promise about the edge is only the edge's to make.

A PostureRMM **proxy relay** is governed by the same rule: to attribute agent audit rows to the *agent* rather than to the relay, list the relay's address here.

### TLS

**The default install is self-signed, and agents pin the certificate authority.** There is no Let's Encrypt, no ACME, and nothing to obtain before your first endpoint enrolls. That is deliberate — a fresh install must work offline on a LAN with no domain.

On first boot the backend generates an internal CA and a leaf certificate signed by it, writes both to the `posturermm-tls` volume, and serves the chain itself. Your browser will warn on first visit; that is expected for a self-hosted internal tool and is not a sign of misconfiguration. Accept the warning or import the CA into the browser or operating-system trust store.

**What the agents do.** They do *not* use the system root store on a default install. The install script stamps the CA's SHA-256 fingerprint into the agent, and the agent validates the server's certificate chain against that one CA — real chain, validity and hostname checking, with a trust anchor of exactly one certificate.

Two consequences worth knowing before you operate this:

- **Rotating the leaf is safe.** Changing `POSTURERMM_DOMAIN` re-mints only the leaf; the CA is persisted and reused, so enrolled agents keep working with no action on the endpoints.
- **Replacing the CA is a fleet event, but no longer a manual one.** Every agent is pinned to it. It happens on first boot and otherwise only when you deliberately remove `ca.crt` + `ca.key`, or when you move between a public certificate and a private one. Enrolled agents lose their transport at that moment — and then recover on their own: after a few failed check-ins each one asks the `:80` bootstrap listener for the current authority and adopts it **only** if the server can prove it knows that endpoint's own enrollment secret. Expect the fleet to reconverge within a few minutes, not to need a visit.

  Losing `ca.crt` + `ca.key` outright is survivable for the *agents* for the same reason — the backend that mints the replacement is still the one holding their enrollment secrets, so they will re-anchor onto it. What you lose is everything that trusted the old CA by hand: browsers, any operating-system trust store you imported it into, and any external tooling pinned to it. Both files stay in the backup set (see Backup & Restore) for that reason.

**The pin is delivered over plain HTTP.** The install one-liner is fetched from `http://<server>/install.ps1` — port 80 serves that path deliberately, because PowerShell 5.1 on a fresh Windows Server 2016 defaults to TLS 1.0 and cannot complete an HTTPS fetch at all. So enrollment still roots in one unauthenticated fetch on your LAN. Pinning narrows the trust-on-first-contact window to that single request; it does not remove it, and it does not make enrollment safe across a hostile network. For internet-reachable servers, distribute the bootstrap through a trusted software-management channel and use a real certificate (below); the approval queue, not a shared secret, is what gates admission.

`POSTURERMM_TLS__CA_KEY_PATH` must be an absolute path inside the
`posturermm-tls` volume — `docker-compose.yml` sets `/app/tls/ca.key`. Unset, it
falls back to a *relative* default resolved against the container's working
directory, which puts the CA private key outside the volume that holds the rest
of the TLS material and loses it on the next container recreation.

**Bringing your own certificate (BYOC).** Obtain a certificate externally (acme.sh, certbot, a commercial CA) and drop `server.crt` + `server.key` into the `posturermm-tls` volume **without** the `.auto-generated` sentinel file. The backend detects the missing sentinel, leaves your files untouched, serves them directly, and stamps `system_roots` trust into the install flow so agents validate against the platform trust store instead.

If your edge terminates TLS with a certificate the backend never generated (a reverse proxy, a tunnel), the backend cannot infer that. Set `POSTURERMM_SERVER__AGENT_TRUST_MODE=system_roots` explicitly.

## Bastion container

The Bastion runs in its own container. It is the *only* component that reaches the internet. The backend drives feed sync itself — it pulls the manifest, diffs per-stream hashes, and fetches changed blobs directly from the Feed (pull mode). The Bastion serves the feed data; the backend is the consumer.

### Configuring it

**There is nothing to create.** The Bastion has no configuration file and the compose stack bind-mounts nothing into it: it starts on its built-in defaults and takes every override from the `POSTURERMM_*` variables in your `.env`, which `docker-compose.yml` passes through **only when they are set** — an unset variable is left unset in the container, so the shipped default applies. The four Bastion-specific ones are in the environment table above (`POSTURERMM_FEED__*`).

A TOML file is still supported if you prefer one — mount it at `/app/config/bastion.toml`, the path `POSTURERMM_BASTION_CONFIG` names, and it layers between the defaults and the environment. Nothing requires it, an absent file is not an error, and a **directory** at that path fails at startup with a message naming it. (That last case is what Docker leaves behind when a compose file bind-mounts a host file that does not exist — it creates an empty directory rather than failing.)

Authentication between the backend and the Bastion is the shared bearer secret described under "Bastion in a DMZ" below, generated automatically for the co-located topology. It is not an admin API key, and the Bastion holds no backend credentials at all.

### Bastion in a DMZ

By default the Bastion runs co-located with the backend (the `bastion` service in `docker-compose.yml`), reached at `http://bastion:8300` over the internal compose network. The backend generates a random bearer secret on first boot, persists it in the `posturermm-bastion-auth` volume, and shares that volume read-only with the Bastion. No operator input is required, including for an offline install.

For a split topology, where the Bastion sits on a separate DMZ host and the backend/db/admin UI stay in the protected zone, the hosts cannot share that volume. Generate one secret and put the same value in the `.env` file beside each compose file:

```bash
openssl rand -hex 32
```

1. **On the DMZ host** — take [`docker-compose.bastion.yml`](https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.bastion.yml) from the release page (that file alone; it bind-mounts nothing), then create `.env` beside it with `POSTURERMM_VERSION`, the generated secret, and the deployment id from the backend's boot log — the compose refuses whichever is absent:
   ```bash
   POSTURERMM_BASTION__SECRET=<generated-64-character-hex-value>
   POSTURERMM_FEED__DEPLOYMENT_ID=<uuid-from-the-backend-boot-log>
   docker compose -f docker-compose.bastion.yml up -d
   docker compose -f docker-compose.bastion.yml ps   # should report bastion healthy
   ```
2. **On the protected-zone host** — leave the main `docker-compose.yml` as-is and set both values in `.env`:
   ```bash
   POSTURERMM_BASTION__URL=https://bastion.dmz.example:8443
   POSTURERMM_BASTION__SECRET=<the-same-generated-64-character-hex-value>
   docker compose up -d backend   # or `docker compose up -d` to recreate everything
   ```
3. **Verify authentication** — `/health` deliberately stays public for the container healthcheck and load balancers. Every `/api/v1/*` route requires `Authorization: Bearer <secret>`; a missing or wrong value returns `401` with `missing or incorrect Bastion shared secret`. Backend logs name `POSTURERMM_BASTION__SECRET` when the two hosts do not match.
4. **Firewall** — protected zone → DMZ, outbound only, destination port matching whatever the Bastion (or its reverse proxy) listens on (`:8300` plain, or your TLS-terminating proxy's port, e.g. `:8443`). No inbound rule from the DMZ to the protected zone is required or supported — the Bastion never initiates a connection inward.
5. **Cross-zone TLS** — co-located, backend→bastion is plain HTTP on the trusted compose network; across a zone boundary it must be HTTPS. The Bastion binary has no built-in TLS termination today. You must run an operator-managed reverse proxy (nginx, Traefik, HAProxy, etc.) on the DMZ host, terminate TLS with an operator-supplied certificate, and forward to the Bastion's plain-HTTP `:8300`. The shared secret authenticates the backend; it does not encrypt traffic.

**Configuration precedence.** Every backend Bastion call reads `config.bastion.url` and `config.bastion.secret`, resolved at process startup through the standard defaults < TOML < `POSTURERMM_*` environment precedence. Changing the split URL or secret means updating `.env` and restarting/recreating the backend container. Rotating the split secret requires updating both hosts; during a mismatch the backend fails closed with a named authentication error.

## Proxy container

A proxy is optional. It exists for **segmented networks**: agents that cannot reach the backend directly point at a proxy inside their own segment, and it relays for them, caches downloads so a hundred endpoints pull a patch once, and queues writes durably while the backend is unreachable.

A proxy runs where its agents are, not beside the backend, so the stock `docker-compose.yml` defines none. It ships as its own release download, `docker-compose.proxy.yml`, version-stamped like the stock one. The offline bundle carries it and the proxy image.

### Registering it

1. In the admin UI, **Proxies → New**. You get an ID and a secret; the secret is shown once.
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

3. Point that segment's agents at `http://<proxy-host>:8443` instead of the backend. The proxy injects `X-Forwarded-For` and `X-Proxy-ID`, so the audit log still records the real endpoint — provided `POSTURERMM_SERVER__TRUSTED_PROXIES` on the backend names the proxy's address (see "Source IP in the audit log" above). Omit it and every row from that segment shows the proxy's address.

### Configuring it

Same shape as the Bastion: built-in defaults, an optional TOML file, then `POSTURERMM_PROXY_*` environment variables. The image ships no config file, and an absent one is not an error.

| Variable | Default | Meaning |
|---|---|---|
| `POSTURERMM_PROXY_BACKEND_URL` | — | Where to relay. Required. |
| `POSTURERMM_PROXY_ID` | — | Proxy record ID from the UI. Required. |
| `POSTURERMM_PROXY_SECRET` | — | Shared secret from the UI. Required. |
| `POSTURERMM_PROXY_LISTEN_ADDRESS` | `0.0.0.0:8443` | Listener. |
| `POSTURERMM_PROXY_TLS_ENABLED` | `false` | See the TLS note below. |
| `POSTURERMM_PROXY_LOG_LEVEL` | `info` | |
| `POSTURERMM_PROXY_CONFIG` | `./config/posturermm-proxy.toml` | Optional TOML, mounted by you. |

**TLS.** The listener is plain HTTP on `:8443` by default — the port number is conventional, not a promise. Agents in a segment you control may be fine with that; across any boundary you do not control, either set `POSTURERMM_PROXY_TLS_ENABLED=true` with `cert_path`/`key_path` in a mounted TOML, or front the proxy with your own TLS-terminating reverse proxy. If you enable TLS, **override the image healthcheck too** — it curls `http://localhost:8443/health` and will report the container unhealthy against an HTTPS listener.

**Liveness.** `GET /health` on the listener. Note it answers **503** with `{"status":"degraded","backend_reachable":false}` when the backend is unreachable — that is a real and useful signal, but it is *not* a reason to restart the proxy: serving its segment from cache and queueing writes for replay is exactly what it should be doing during a backend outage. The image's healthcheck therefore probes **liveness only** (any response counts), so the container stays healthy through an outage. If you wire this endpoint into your own monitoring, treat 503-degraded as "backend down", not "proxy down".

The proxy also heartbeats the backend every 60s, so one that stops checking in shows as stale in the UI without you polling it.


## Agent Enrollment

1. **Create an install key** — Admin UI → Settings → Install keys → Create key. The key is optional and buys one thing: skipping the approval queue, which unattended rollouts need. Without it the same one-liner still enrolls, inert, until you admit the endpoint from Endpoints. `POSTURERMM_SERVER__ALLOW_AUTO_ENROLLMENT=true` admits keyless enrollments instantly instead — a trusted-LAN convenience, and an open door on anything internet-reachable.
2. **Run the one-liner** on the target Windows endpoint (elevated PowerShell):
   ```powershell
   irm "http://posturermm.example.com/install.ps1?key=..." | iex
   ```
   The shim writes `C:\ProgramData\PostureRMM\install-key.txt`, downloads the current MSI from the server, and runs `msiexec /i /qn SERVERURL=https://posturermm.example.com`. The MSI registers the agent service and starts it; the agent reads the key file on first boot, enrolls, and self-cleans the file.

**Where the MSI on the server comes from.** There is never a step for you to place it. An **offline install** gets it from the bundle — `install.sh` seeds the downloads volume before the stack starts. An **online install** gets it from the product feed: the Bastion fetches the binaries on first feed pull and the server indexes them. Until that first pull completes, Endpoints → Install first endpoint shows *Fetching agent v…* and deliberately offers no install command — the one-liner would download an MSI that isn't there yet. If your Bastion only has egress during a maintenance window, that notice is what you'll see until the window opens; the request is queued in the database and survives restarts, so nothing needs re-asking.

For SCCM / Intune / GPO software-distribution flows, grab the current MSI URL and checksum from Admin UI → Endpoints → Install agent → More install options, and pass `SERVERURL` on the `msiexec` command line; drop `install-key.txt` next to the MSI so the agent picks it up via the captured source dir.

### Changing the agent's backend URL

`ServerUrl` is registry-canonical — the agent reads `HKLM\SOFTWARE\PostureRMM\Agent\ServerUrl` at boot and ignores TOML and env. Two supported operator paths to change it:

1. **Per-machine, canonical** — elevated PowerShell on the endpoint:
   ```powershell
   posturermm-agent.exe config set server.url https://posturermm.example.com
   posturermm-agent.exe config get     # sanity check
   ```
   Validates the URL, writes the registry value, restarts `PostureRMM-Agent`. Emits an `audit.config` log line.

2. **Fleet-wide (SCCM / Intune / GPO)** — push an explicit reinstall with the new URL:
   ```cmd
   msiexec /x posturermm-agent.msi /qn
   msiexec /i posturermm-agent.msi SERVERURL=https://new-host.example.com /qn
   ```
   The MSI writes the registry value only on fresh install, so an in-place upgrade that passes `SERVERURL=` does *not* clobber operator changes — uninstall + reinstall is the explicit "I really want this URL everywhere" gesture.

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

### The backend turns PostgreSQL's JIT off

Worth knowing before you go looking for it: every pooled connection runs `SET jit = off`, so `SHOW jit` in a `psql` shell reports `on` while the application is running with it off. That is deliberate and set in the connection pool rather than in `docker-compose.yml`, so it holds whether you use the bundled Postgres or point `DATABASE_URL` at your own.

JIT compilation costs time proportional to how many expressions a plan contains and repays it proportional to how many rows those expressions run over. This product's heavy queries are the wrong side of that trade — wide `CASE` ladders and many filtered aggregates over a page's worth of rows. Measured on a 10-endpoint fleet, JIT as the only variable:

| query | with JIT | without |
|---|---|---|
| compliance fleet rollup | 940 ms | 191 ms |
| patching fleet queue | 178 ms | 72 ms |

The rollup compiled 169 functions to return 35 rows. No query in the product was measured faster with JIT on. Raising `jit_above_cost` instead would have to sit above 14.2M to spare the worst case, at which point nothing would ever JIT anyway.

### Prometheus metrics

`/metrics` is served on the backend's internal `:3000` plane only — the one the Bastion and the healthcheck already use, deliberately never published to the host. It takes no credentials, so `:443` and `:80` 404 it. Scrape it from the same Compose network (`docker network connect posturermm_default <your-scraper>` if your Prometheus runs elsewhere):

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

With `POSTURERMM_TLS__ENABLED=false` that single `:3000` listener *is* the internal plane and carries `/metrics`; do not proxy the path through to the public side.

### Data retention

Settings → Retention (SuperAdmin) controls four independent windows, all enforced daily by the same scheduler job:

| Setting | Default | Notes |
|---|---|---|
| Task history | 90 days | Terminal `tasks` rows. Deployment stats are materialized to the parent `deployments` row first, so fleet-level history survives the prune. |
| Compliance snapshot | 365 days | Daily per-endpoint compliance percentage, feeds trend charts and evidence packs. |
| Posture snapshot | 365 days | Daily per-endpoint posture score, feeds fleet sparklines. Same shape and growth rate as the compliance snapshot — and the same default window: **upgrading an existing deployment begins pruning rows older than 365 days immediately**, exactly as it already does for compliance snapshots. There is no unlimited option for this window. |
| Audit log | **0 (keep forever)** | The one window that defaults to unlimited, and the only one of the four with an unlimited option at all — matches the product's original "audit log is permanent" guarantee, so upgrading to this setting never starts deleting an existing operator's history on its own. Set a positive number of days to opt into pruning. |

Three telemetry histories are pruned on fixed windows the operator does not set, by hourly scheduler tasks rather than the daily job above: performance metrics and endpoint reachability at **30 days**, and disk SMART history at **2 years**. The long disk window is affordable because that history is stored on change plus a daily heartbeat, not on a fixed cadence.

#### Upgrade note — disk SMART history is compacted in place

The migration that introduces store-on-change also **deletes rows from `disk_health_snapshots`** on upgrade. Read this before upgrading if you have been running long enough to have a large history there.

Until this release the agent appended one row per physical disk **every five minutes, forever**, with nothing pruning the table — a 500-endpoint fleet accrues roughly 105 million rows and 31 GB a year. Worse, four of the columns that history existed for (`temperature_celsius`, `nvme_percentage_used`, `read_errors_uncorrected`, `write_errors_uncorrected`) were never populated, because the agent read the SMART reliability counter with a query that returns nothing on every machine. Both halves are fixed together: the agent now reaches the counter correctly, and the backend only writes a row when a value moves.

What the migration deletes is every row **identical to the row before it** — precisely the history store-on-change would have produced. For each (endpoint, disk) it keeps the oldest row of every identical run, so "first seen healthy" survives, and always keeps the newest row, so "last seen" survives for endpoints that are offline at upgrade time. Measured against the development fleet: 209,844 rows became 26, in 2.5 seconds, with **zero** distinct readings lost and the posture evaluator's per-disk read byte-identical before and after.

Two operational points:

- **Space is not returned immediately.** The migration deletes rows inside its transaction; the table stays its old size until autovacuum reclaims it. In the measurement above the table read 56 MB after the delete and 64 kB after a manual `VACUUM FULL disk_health_snapshots` — worth running by hand if the reclaim matters to you and you can take the exclusive lock.
- **Surviving rows are stamped `smart_status = 'not_collected'`**, a new column that records *why* a SMART value is absent — nobody asked (`not_collected`), the disk exposes no reliability counter (`unavailable`), the probe failed (`probe_error`), or the counter was read and the device simply leaves that field blank (`reported`). Pre-upgrade rows get `not_collected` because no probe ever established anything about that hardware. Real values start arriving as endpoints check in on the new agent.

Task, compliance-snapshot, and posture-snapshot retention all accept 30–3650 days with no "never prune" option — a zero-day window there would mean "delete everything immediately," which the Settings UI and the API both refuse. Audit-log retention accepts 0–3650 (0 = never prune); it is the one compliance record in the product where indefinite retention is a legitimate default, not a missing feature. Pruning runs in bounded batches so a first run against years of backlog doesn't hold a long-lived lock on a live table. Because the audit log is the record used for security-incident review, pruning it writes one more audit row (`audit_log_prune`) recording how many rows were removed and the cutoff — the trail records its own trimming.

## Security Checklist

- [x] TLS termination by the backend itself
- [x] The backend publishes ports 80/443; its plain-HTTP `:3000` plane, DB, and Bastion remain on the internal Docker network
- [x] Backend makes zero outbound internet connections — the Bastion container is the only internet-facing component
- [x] JWT signing secret auto-provisioned to `posturermm-data` volume (or externally supplied via `JWT_SECRET`, 32+ chars)
- [x] Keyless enrollment waits in the approval queue; install keys are the opt-in unattended lane
- [x] 2FA enforced for admin accounts (Settings → Security → Enforce 2FA)
- [x] Audit-log source IP is trustworthy: forwarding headers are believed only from a peer named in `POSTURERMM_SERVER__TRUSTED_PROXIES` — a client cannot forge its own address into the trail
- [x] Signing out revokes the session's refresh token server-side and records a `logout` event

## Backup & Restore

Treat the database, secret-bearing volumes, and `.env` as one backup set. A
database dump alone appears to restore successfully, but the backend then
silently auto-provisions a new internal CA, JWT signing secret, and TOTP key on
its first boot. The UI and database will look healthy while enrolled agents
reject the new certificate and existing TOTP enrollments and backup codes no
longer work.

The following is the complete backup procedure. Run it from the directory that
contains `docker-compose.yml` and `.env`. It briefly stops the backend so its
volumes cannot change while they are archived; PostgreSQL and Bastion remain
running. The PostgreSQL image already present for this deployment is used as
the archive helper, so this also works from an offline release bundle.

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

Protect and copy the whole backup directory off-host as one unit. It contains
database contents, private keys, authentication secrets, and the database
password, so store it with the same controls as production credentials.

Every included artifact has a defined recovery role; the omission consequences
are:

| Artifact | What it preserves | What breaks if it is omitted |
|---|---|---|
| `posturermm.sql` | The portable backup of `posturermm-db-data`: users, endpoints, configuration, audit history, and all other relational state | The application data is gone. A raw copy of the live PostgreSQL volume is not a substitute for `pg_dump`. |
| `posturermm-tls.tar.gz` | `posturermm-tls`: the internal CA and key, leaf certificate and key, and auto-generation sentinel | Every agent pinned to the old leaf/CA rejects the replacement certificate and stops connecting. |
| `posturermm-data.tar.gz` | `posturermm-data`: the auto-provisioned `jwt-secret`, `totp-secret`, and other backend-persisted secret material | Existing access tokens no longer validate. More importantly, stored TOTP secrets cannot be decrypted and MFA backup-code hashes no longer verify, locking 2FA users out. |
| `.env` | `POSTGRES_PASSWORD` and any operator-supplied `JWT_SECRET` or `POSTURERMM_CREDENTIAL_KEY` | PostgreSQL credentials no longer match, explicitly supplied JWT keys rotate, and stored credentials/evidence-signing keys encrypted under the credential key become unusable. Neither value is in `pg_dump`. |

Four volumes are deliberately excluded. Three are caches: `posturermm-bastion-data`
is a feed cache and regenerates; `posturermm-binary-store` contains refetchable
installer blobs (offline deployments may need to repopulate it before deploying
software); and `posturermm-downloads` contains staged agent installers that are
reproduced from the bundled image on boot.

The fourth, `posturermm-bastion-auth`, holds a secret but still does not need
backing up. In the co-located topology the backend regenerates the Bastion bearer
secret on first boot and shares the volume read-only with the Bastion, so both
sides pick up the new value together and stay in agreement. In the DMZ-split
topology the secret comes from `POSTURERMM_BASTION__SECRET` in `.env` — which
*is* in the backup set — and the volume is not used at all.

None of the four re-key the installation or lose authoritative application data.

Restore only onto a fresh host with no existing PostureRMM containers or named
volumes. Install the same release's compose files, put the backup directory
beside them, and **do not run `docker compose up` yet**. The secret and TLS
volumes must be populated before the backend's first boot. If the backend boots
against empty volumes, it silently writes new secrets and a new CA; subsequent
volume restore cannot undo sessions or agent trust already rotated to those new
values.

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
declaring recovery complete. Also compare the restored CA fingerprint with the
pre-migration value (or a value recorded from a known-good backup):

```bash
docker cp posturermm-backend:/app/tls/ca.crt ./restored-ca.crt
openssl x509 -in ./restored-ca.crt -noout -fingerprint -sha256
rm ./restored-ca.crt
```

## Troubleshooting

- **Backend won't start:** `docker compose logs backend`. Migrations log themselves on startup.
- **502 Bad Gateway from an operator-managed upstream proxy:** the backend container is down / not ready. Check `docker compose ps` — `backend` should be `healthy` — then inspect `docker compose logs backend`.
- **Backend not serving HTTPS:** verify the `posturermm-tls` volume is mounted and `server.crt` / `server.key` exist. If not, check `docker compose logs backend` for certificate generation or listener errors on startup.
- **UI loads blank / 404:** the admin UI is served at `/` by the backend; if `/` doesn't render the login screen, the backend isn't serving traffic yet (`docker compose logs backend`).
- **Agent can't connect:** verify DNS resolution, port 443 reachable, cert valid. `POSTURERMM_SERVER__PUBLIC_URL` must match what the agent sees.
- **Bastion returning 401 to the backend:** the backend-to-Bastion bearer secret disagrees. The Bastion checks `POSTURERMM_BASTION__SECRET` (or, when that is unset, the file at `[bastion] secret_path`) against what the backend presents; both sides must hold the same 32+ character value.
- **Bastion returning 502 on feed pulls:** it could not reach the feed origin. Check egress from the DMZ host to `[feed] api_url` (default `https://feed.posturermm.com`) and `docker compose logs bastion` for the upstream error.

---

Back to the [README](../README.md), the [quickstart](quickstart.md), or
[configuration.md](configuration.md).

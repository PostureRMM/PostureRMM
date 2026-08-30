# Quickstart

<!-- Mirrored verbatim to PostureRMM/PostureRMM; never edit on the hub.
`just public-docs-sync` — deploy/release/check-public-docs.sh reds on a difference. -->

## Requirements

Docker Engine 20.10+, Compose v2 and `curl` on any Linux host, with at least 4 GB of RAM and
10 GB free. **Nothing to edit** — no values to choose, no certificates to obtain, no DNS.

Debian and Ubuntu minimal images ship without `curl` (`sudo apt install curl` first), and without
Docker. If you need Docker, its own installer covers every supported distribution:

```bash
curl -fsSL https://get.docker.com | sh
```

Do not use `apt install docker-compose` as a version check: Ubuntu 22.04/24.04 and Debian 12
provide the legacy Python v1 there, which is not supported. `preflight.sh` probes both
`docker compose` and `docker-compose`, picks the newer, and prints instructions for your OS when
either is missing or too old.

Windows endpoints supported: **11, 10, and Server 2016 / 2019 / 2022 / 2025.**

## Install

```bash
mkdir posturermm && cd posturermm
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.yml
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/preflight.sh
chmod +x preflight.sh && ./preflight.sh
```

`preflight.sh` checks the host, generates your database password, records this host's address as
the name agents will use to reach it, pulls the images and starts the stack.

## First login

First boot writes the generated `admin@localhost` password to a 0600 file and logs that path
rather than the value. Read it, then log in at `https://<host>/`:

```bash
docker compose exec backend cat /app/data/bootstrap-admin-password
```
 Add your first endpoint from **Endpoints → Install first endpoint**, which
hands you a one-line PowerShell command to run on it.

## Air-gapped install

Take `posturermm-offline-vX.Y.Z.tar.gz` from the same release, carry it across however your air gap
works, then:

```bash
tar xzf posturermm-offline-vX.Y.Z.tar.gz
cd posturermm-offline-vX.Y.Z
./install.sh
./preflight.sh
```

Same stack, with `docker load` in front of the pull. `install.sh` loads the bundled images and puts
the agent installer in place; `preflight.sh` is the same script the online quickstart runs, and it
is what checks the host, generates the database password and starts the stack. The agent installer
travels inside the bundle, so first enrollment works with no internet at any point.

## Next

Every setting is optional. See [configuration.md](configuration.md) for hostnames, ports, your own
certificate, split deployments, the content feed, and log and audit tuning.

[PRODUCTION.md](PRODUCTION.md) is the operator's runbook: a Bastion in a DMZ, the segment proxy,
backup and restore, upgrades, metrics and troubleshooting.

---

Back to the [README](../README.md).

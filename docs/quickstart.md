# Quickstart

## Requirements

Docker Engine 20.10+, Compose v2 and `curl` on any Linux host. **Nothing to edit** — no values
to choose, no certificates to obtain, no DNS. (Debian and Ubuntu minimal images ship without
`curl`: `sudo apt install curl` first.)

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

Watch `docker compose logs -f backend` for the generated `admin@localhost` password, then log in
at `https://<host>/`. Add your first endpoint from **Endpoints → Install first endpoint**, which
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

---

Back to the [README](../README.md).

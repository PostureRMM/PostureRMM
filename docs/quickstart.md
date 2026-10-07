# Quickstart

<!-- Mirrored verbatim to PostureRMM/PostureRMM; never edit on the hub.
Each release publishes it there — deploy/release/check-public-docs.sh. -->

## Requirements

A Linux host with at least 4 GB of RAM, 10 GB free and `curl`, running Docker Engine 20.10+ with
Compose v2, or Podman 5+ as root. **Nothing to edit** — no values to choose, no certificates to
obtain, no DNS.

A host with no container runtime needs nothing done by hand: run the install below with
`sudo ./preflight.sh` and it offers to install one. On RHEL, Rocky and AlmaLinux that is Podman
from the OS's own repositories; on Debian and Ubuntu it is Docker, through Docker's installer. It
shows the commands and asks first, and it never replaces a runtime that is already there. SELinux
can stay enforcing, and firewalld needs no change.

Debian and Ubuntu minimal images ship without `curl`: `sudo apt install curl` first.

Do not use `apt install docker-compose` as a version check: Ubuntu 22.04/24.04 and Debian 12
provide the legacy Python v1 there, which is not supported. `preflight.sh` probes both
`docker compose` and `docker-compose`, picks the newer, and prints instructions for your OS when
either is missing or too old.

Windows endpoints supported: **11, 10, and Server 2016 / 2019 / 2022 / 2025.** Linux endpoints
supported: **Debian 12/13, Ubuntu 22.04/24.04 LTS, RHEL, Rocky Linux and AlmaLinux 9/10**, on x86_64
with systemd.

## Install

```bash
mkdir posturermm && cd posturermm
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.yml
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/preflight.sh
chmod +x preflight.sh && ./preflight.sh
```

`preflight.sh` checks the host, generates your database password, records this host's address as
the name agents will use to reach it, pulls the images and starts the stack. Podman runs the stack as
root, so on a Podman host run it as `sudo ./preflight.sh`, and prefix `sudo` to the `docker`
commands in these docs.

## First login

First boot writes the generated `admin@localhost` password to a 0600 file and logs that path
rather than the value. Run from a terminal, `preflight.sh` waits for the backend and prints the
password with the URL, so you can log in at `https://<host>/` straight away. Only a terminal is shown
the value: run from CI or a config-management tool, whose output is logged, it prints this command
instead, which works at any time:

```bash
docker compose exec backend cat /app/data/bootstrap-admin-password
```

This password is one-time: once you change it, the next restart replaces that file with a note
saying so. If you lose it, [PRODUCTION.md](PRODUCTION.md#troubleshooting) has the reset command.

Add your first endpoint from **Endpoints → Install first endpoint**, which
hands you a one-line PowerShell command to run on it. For a Linux endpoint the same screen shows:

```bash
curl -fsSL http://<host>/install-agent.sh | sudo sh
```

or, on a host with no `curl`, `wget -qO- http://<host>/install-agent.sh | sudo sh`. It shows them
only once the Linux agent packages are on the server; until then it says why.

These commands carry no install key, so the endpoint arrives in the approval queue rather than
straight into the fleet. Admit it from the same screen — or at **Settings → Enrollment → Endpoints
awaiting approval** — and its first compliance scan finishes a few minutes later.

## Air-gapped install

Take `posturermm-offline-vX.Y.Z.tar.gz` from the same release, carry it across however your air gap
works, then:

```bash
tar xzf posturermm-offline-vX.Y.Z.tar.gz
cd posturermm-offline-vX.Y.Z
./install.sh
```

On a Podman host, run it with `sudo`. A host with no runtime at all gets one from `sudo ./preflight.sh`
first, installed from your OS repositories' local mirror; then run `./install.sh`. The bundle carries
the Compose plugin Podman needs.

Same stack, with `docker load` in front of the pull. `install.sh` loads the bundled images, puts
the agent installer in place and then runs `preflight.sh`, the same script the online quickstart
runs, which checks the host, generates the database password and starts the stack. The agent installer
travels inside the bundle, so first enrollment works with no internet at any point.

## Next

Every setting is optional. See [configuration.md](configuration.md) for hostnames, ports, your own
certificate, split deployments, the content feed, and log and audit tuning.

[PRODUCTION.md](PRODUCTION.md) is the operator's runbook: a Bastion in a DMZ, the segment proxy,
backup and restore, upgrades, metrics and troubleshooting.

---

Back to the [README](../README.md).

# Quickstart

<!-- Mirrored verbatim to PostureRMM/PostureRMM; never edit on the hub.
Each release publishes it there — deploy/release/check-public-docs.sh. -->

Already running PostureRMM? Jump to [Upgrade](#upgrade).

## Requirements

A Linux host (Windows 10 or 11: [see below](#install-on-windows)) with at least 4 GB of RAM, 10 GB free and `curl`, running Docker Engine 20.10+ with
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
supported: **Debian 12/13, Ubuntu 22.04/24.04/26.04 LTS, RHEL, Rocky Linux and AlmaLinux 9/10**, on x86_64
with systemd.

## Install

```bash
mkdir posturermm && cd posturermm
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.yml
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/preflight.sh
chmod +x preflight.sh && sudo ./preflight.sh
```

`preflight.sh` checks the host and that ports 443 and 80 are free, generates your database password, records this host's address as
the name agents will use to reach it, pulls the images and starts the stack. It needs `sudo` to install a
runtime and to run Podman, which runs the stack as root. Prefix `sudo` to the `docker` commands in these
docs too: the `.env` it writes is root's, and a fresh Docker install has not put your user in the `docker` group.

Running it again is safe. The 10 GB free-space check applies to a first install only, not to a re-run once the images are local.

## Install on Windows

A Windows PC is for trying PostureRMM. Docker Desktop runs only while someone is signed in, so
after a restart (Windows Update restarts a PC overnight) the server is down until someone signs in
and Docker Desktop, a few minutes later, reports "Engine running"; enrolled agents wait meanwhile.
For a server that runs unattended, use a Linux host, or a small Linux VM in Hyper-V, and the Linux
install above.

**Before you start**, on a Windows 10 or 11 PC with 8 GB of RAM (Docker gives containers half of
it by default) and 10 GB free:

1. Turn on hardware virtualisation (Intel VT-x or AMD SVM Mode) in the PC's firmware settings if it is
   off. WSL 2 needs it.
2. Download [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/) (about
   600 MB) and run the installer as an administrator.
3. Keep "Use WSL 2" ticked and accept the Docker Subscription Service Agreement.
4. Restart Windows when the installer asks.
5. If Docker Desktop says WSL is missing or out of date, run `wsl --install --no-distribution` in an
   administrator PowerShell and restart if it asks. Close the "Welcome to WSL" window if one opens.
6. Open Docker Desktop from the Start menu. It does not start on its own the first time.
7. On the "Welcome to Docker" screen, choose **Skip** (or sign in). Docker Desktop answers nothing
   while that screen waits.
8. Wait for "Engine running", about 3 minutes the first time.

`preflight.ps1` checks these in order and, if one is missing, prints the next one. It also turns on
Docker Desktop's "Start Docker Desktop when you sign in", so signing in brings the server back.
Then, in PowerShell, no administrator prompt:

```powershell
mkdir posturermm; cd posturermm
curl.exe -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.yml
curl.exe -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/preflight.ps1
powershell -ExecutionPolicy Bypass -File .\preflight.ps1
```

Windows Server can't host the PostureRMM server: Docker Desktop runs on Windows 10 and 11 only. Run a small Linux VM in
Hyper-V and use the Linux install above; Windows Server stays a supported endpoint.

`preflight.ps1` does what `preflight.sh` does, and `-ExecutionPolicy Bypass` applies to this one run
only; nothing machine-wide changes. If Docker has too little RAM it stops within seconds, before
anything downloads, and prints the `.wslconfig` fix. If another program holds port 80 or 443 (IIS often holds 80),
it names the program. Windows then asks whether to let Docker Desktop Backend through its firewall,
in a window that can hide behind Docker's: choose Allow, or other machines cannot reach the server.
If you dismissed it, `preflight.ps1` says so when it finds the block, and this administrator PowerShell
line fixes it:

```powershell
Get-NetFirewallRule -DisplayName 'Docker Desktop Backend' | Set-NetFirewallRule -Action Allow
```

Log in as below, with `docker compose` commands run in the same folder.

Docker Desktop is free for personal use, education, non-commercial open source and businesses with
fewer than 250 employees and under $10 million in annual revenue; larger organisations need a paid
Docker subscription. The air-gapped install is not supported on Windows; use a Linux host.

## First login

First boot writes the generated `admin@localhost` password to a 0600 file and logs that path
rather than the value. Run from a terminal, `preflight.sh` waits for the backend and prints the
password with the URL, so you can log in at `https://<host>/` straight away. Only a terminal is shown
the value: run from CI or a config-management tool, whose output is logged, it prints this command
instead, which works at any time:

```bash
docker compose exec backend cat /app/data/bootstrap-admin-password
```

Your browser will warn that the certificate is not trusted: the server made its own on first start, and
`preflight.sh` prints its fingerprint for you to check. To use your own certificate instead, see
[PRODUCTION.md](PRODUCTION.md#tls-and-agent-trust).

You will then choose your own password (12 characters or more) and set up an authenticator app, so have
your phone or password manager ready and keep the backup codes it shows.

This password is one-time: once you change it, the next restart replaces that file with a note
saying so. If you lose it, [PRODUCTION.md](PRODUCTION.md#troubleshooting) has the reset command.

Add your first endpoint with **Install first endpoint** (the button on an empty Dashboard, or
under **Endpoints**), which hands you a one-line PowerShell command to run on it. For a Linux endpoint the same screen shows:

```bash
curl -fsSL http://<host>/install-agent.sh | sudo sh
```

or, on a host with no `curl`, `wget -qO- http://<host>/install-agent.sh | sudo sh`. It shows them
only once the Linux agent packages are on the server; until then it says why.

These commands carry no install key, so the endpoint arrives in the approval queue rather than
straight into the fleet. Admit it from the same screen — or from **Pending approval** on
**Settings → Enrollment** — and its first compliance scan finishes a few
minutes later.

To do the same without the web UI, use an API key with the Admin role:
`GET /api/v1/enrollment-requests` lists the waiting endpoints,
`POST /api/v1/enrollment-requests/<id>/approve` admits one (send `{}`, or a `group_id` and
`tag_ids` to place it), and `DELETE /api/v1/enrollment-requests/<id>` discards it. **Settings → API
reference** has the details.

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

## Upgrade

**Back up first** ([how](PRODUCTION.md#backup--restore)): database migrations cannot be undone.
Then, in the folder holding your `docker-compose.yml` (in PowerShell, `curl.exe -LO`; on Linux,
`sudo` before `docker`):

```bash
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.yml
docker compose pull
docker compose up -d
```

The new `docker-compose.yml` carries its release's version, so there is nothing to edit.
Agents upgrade themselves once the server has the new agent, wherever an upgrade policy
(Policies → Upgrade) covers them. What changed and any manual step, for every release you are
skipping: the [release notes](https://github.com/PostureRMM/PostureRMM/blob/main/docs/release-notes.md). Pinning a
version: [configuration.md](configuration.md).

## Next

Every setting is optional. See [configuration.md](configuration.md) for hostnames, ports, your own
certificate, split deployments, the content feed, and log and audit tuning.

[PRODUCTION.md](PRODUCTION.md) is the operator's runbook: a Bastion in a DMZ, the segment proxy,
backup and restore, upgrades, metrics and troubleshooting.

---

Back to the [README](../README.md).

<div align="center">

<img src="./docs/brand/logo-lockup.svg" alt="PostureRMM" width="380" />

**Endpoint management, security and compliance in one self-hosted platform.**

Manage, patch, deploy, remote into, harden, audit and remediate your Windows fleet —
from one console, on your own hardware, with nothing phoning home.

**Free forever up to 50 endpoints. Every feature. No asterisks.**

</div>

<div align="center">
<img src="./docs/screenshots/dashboard.png" alt="PostureRMM dashboard" width="900" />
</div>

---

## What it does

Most tools in this space stop at telling you what is wrong. PostureRMM closes the loop: it finds
the problem, fixes it, then **re-measures to prove the fix worked** — and if it didn't, it says so
rather than reporting the API call as a success.

| Area | Capabilities |
|---|---|
| **Compliance** | Audit against DISA STIG, Microsoft Security Baselines and a curated hardening profile. Failing checks carry authored fixes that apply per-endpoint, per-check across the fleet, or as "apply all safe fixes" — behind a safety gate that refuses anything which would cut your own access. Every fix is verified by re-running the check, and every fix is revertible, because each executor captures the state it overwrote. |
| **Patching** | Windows updates scanned **offline** against Microsoft's own applicability engine — no internet needed on the endpoint or the server. Third-party app updates from a curated catalogue. CVE correlation across NVD, EPSS, MSRC, CISA KEV and GHSA with version-range-aware matching. Deployment policies with canary pauses, maintenance windows and an optional supply-chain bake period. |
| **Software** | Curated app catalogue with one-click install and uninstall, custom `.msi`/`.exe`/`.msix` upload, registry-verified installs with exit-code classification and tiered retry, full software inventory, and an application blocklist for rival agents and unapproved remote-access tools. |
| **Remote access** | Browser-based remote desktop, a real terminal, file browse/upload/download, and PowerShell execution with an approval workflow and live output — no separate tool, no separate agent. |
| **Fleet** | Rule-based dynamic groups, ARP network discovery for unmanaged devices, hardware and disk-health inventory, per-endpoint reachability history, maintenance windows, reboot policies with end-user deferral, and alerting over webhook and email with delivery health you can see. |
| **Access & audit** | OIDC single sign-on — free, in every tier — five-role RBAC with group scoping, TOTP 2FA, scoped API keys, a permanent audit trail, multi-tenant data model, and a read-only "view as" lens so you can check RBAC is what you think it is. |

## The Posture Score

Three pillars, equally weighted, composing one number that drills all the way down to the check or
telemetry sample that produced it.

| Pillar | Covers |
|---|---|
| **Prevention** | DISA STIG, Microsoft Security Baseline, PostureRMM Recommended hardening, mandate crosswalks |
| **Patching** | CVE exposure, patch latency and cadence, OS and application end-of-life |
| **Performance** | Disk health and fill projection, SMART/NVMe wear, thermal throttling, battery health, reboot age, time-sync drift, service state |

Unmeasured surface **shrinks the maximum** rather than counting as failure, so a score is never a
guess dressed up as a number. An empty fleet reads "not yet established", not zero.

## What's in the box

**2,850 atomic compliance checks** across ten platforms, imported from upstream DISA SCAP/XCCDF and
the Microsoft Security Compliance Toolkit by dedicated importers — not hand-transcribed:

| Platform | Checks |
|---|---|
| Windows 11 · 10 | 577 · 496 |
| Windows Server 2022 · 2019 · 2016 · 2025 | 443 · 384 · 329 · 286 |
| Microsoft 365 Apps · Edge · Chrome · Firefox | 201 · 73 · 38 · 23 |

Plus **nine mandate crosswalks** — HIPAA, PCI DSS v4, SOC 2 CC, ISO 27001 Annex A, CMMC L1 and L2,
Cyber Essentials, cyber-insurance questionnaires and customer security questionnaires — mapping
regulatory controls back to the checks that answer them.

Checks are YAML, not code. So is every posture rule. You can read them, and so can your auditor.

## Offline by design

**The product runs with no internet at all.** Enrollment, compliance scanning, remediation, patch
installation, software deployment, remote desktop, reporting — all of it happens on your network. Your
endpoints never reach out. Neither does the backend. Even Windows Update scanning is done offline,
against Microsoft's own applicability engine, so an endpoint never contacts Microsoft either.

The internet buys exactly one thing: **fresh content.** New CVEs, patch payloads, updated
compliance baselines, the app catalogue. That arrives by *hydration* — a single component, the
Bastion, pulls from a static feed on Cloudflare R2 on a schedule you set. Open egress for a
maintenance window, let it fill up, close it again. Anything the backend asks for while the door
is shut queues durably and drains the moment it opens, with no operator action and nothing to
re-request.

Nothing goes the other way. There is no PostureRMM server that knows your fleet exists — the feed
is a file store and your Bastion downloads files. Community needs no activation and no licence
token at all; Business activates once and then never needs to reach us again.

First install and first enrollment work with the door shut too: the agent installer travels inside
the offline bundle.

## Architecture

RMM is the most abused software category there is — see [LOLRMM](https://lolrmm.io). PostureRMM is
built so it cannot become another entry on that list: your backend accepts **no inbound connection
from us**, and cannot be reached by us at all.

```
        ┌────────────────────┐
        │  PostureRMM Feed   │   threat intel, patch metadata, app catalog
        └─────────┬──────────┘
                  │ HTTPS, outbound only, on your schedule
┌─────────────────┼───────────────────────────────────────────┐
│ YOUR NETWORK    ▼                                           │
│   Bastion (Rust) ── the only component that reaches out     │
│      │                                                      │
│      ▼                                                      │
│   Backend (Rust) ── TLS edge, API, admin UI, scheduler      │
│      │              zero outbound internet                  │
│      ├── PostgreSQL                                         │
│      └── Agents (Rust) ── Windows service, no runtime deps  │
└─────────────────────────────────────────────────────────────┘
```

Run the Bastion always-on and your fleet stays current continuously; run it behind a door you open
once a month and your fleet stays current as far back as your last sync. Both are first-class —
neither is a degraded mode.

There is no cloud version to migrate you to.

## Quick Start

Docker Engine 20.10+ and Compose v2 on any Linux host. **Nothing to edit** — no values to choose,
no certificates to obtain, no DNS.

```bash
mkdir posturermm && cd posturermm
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/docker-compose.yml
curl -LO https://github.com/PostureRMM/PostureRMM/releases/latest/download/preflight.sh
chmod +x preflight.sh && ./preflight.sh
```

`preflight.sh` checks the host, generates your database password, pulls the images and starts the
stack. Then watch `docker compose logs -f backend` for the generated `admin@localhost` password and
log in at `https://<host>/`. Add your first endpoint from **Devices → Install Agent**, which hands
you a one-line PowerShell command to run on it.

Windows endpoints supported: **11, 10, and Server 2016 / 2019 / 2022 / 2025.**

<details>
<summary><b>Air-gapped install</b></summary>

Take `posturermm-offline-vX.Y.Z.tar.gz` from the same release, carry it across however your air gap
works, then:

```bash
tar xzf posturermm-offline-vX.Y.Z.tar.gz
cd posturermm-offline-vX.Y.Z
./install.sh
```

Same stack, with `docker load` in front of the pull. The agent installer travels inside the bundle,
so first enrollment works with no internet at any point.

</details>

## Pricing

**Free, forever, up to 50 endpoints — with every feature above.** Not a trial, not a
feature-limited edition, not "free for personal use". SSO, the full posture model, all compliance
content, remote access and every report are in it.

A paid tier for larger fleets arrives in **2027**, once the free one has had real time in the
field. It will raise the endpoint cap and change nothing else — your first 50 stay free at any
scale, and there will be no feature matrix, because there are no features to withhold.

## Support and downloads

Everything is here on GitHub for now — there is no support portal, no ticket system, and no
separate download site.

- [**Discussions**](https://github.com/PostureRMM/PostureRMM/discussions) — questions, how-tos,
  anything you would otherwise ask a colleague
- [**Issues**](https://github.com/PostureRMM/PostureRMM/issues) — bugs and feature requests
- [**Releases**](https://github.com/PostureRMM/PostureRMM/releases) — every download: the compose
  file, the preflight script, the offline bundle, the agent installers
- [**Security policy**](SECURITY.md) — report a vulnerability here, never as a public issue

## License

Proprietary — see [LICENSE](LICENSE). Free and perpetual up to 50 endpoints, no redistribution or
rebranding of our builds. It is short and written to be read.

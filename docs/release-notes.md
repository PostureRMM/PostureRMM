# PostureRMM release notes

What changed in every release, newest first. The hub keeps only the newest release's files, so upgrade to it directly; the notes of earlier releases stay on this page. To upgrade, follow the [upgrade steps](../README.md#upgrade).

Jump to: [1.31.0](#v1.31.0) · [1.30.0](#v1.30.0) · [1.29.0](#v1.29.0) · [1.28.1](#v1.28.1) · [1.28.0](#v1.28.0)

<a id="v1.31.0"></a>

## Version 1.31.0

Sign in with your directory, publish your own apps and certificates, keep devices on patch baselines, watch installs as they run, see a scorecard per benchmark, and use a cleaner, more consistent interface.

### Upgrading

Back up first, because database migrations cannot be undone. Then, in the folder holding your `docker-compose.yml`:

```bash
docker compose pull && docker compose up -d
```

There are no manual steps. The first start can take a few minutes on a large history. Duplicate open alerts are merged into the oldest (marked cleared, not deleted). If a sign-in method was deleted, people who used it are re-linked by verified email at their next sign-in.

Agents upgrade through your upgrade policy, and nothing needs forcing. Upgrade your agents promptly: certificates and tamper protection need an agent on this version, and agents older than 1.31.0 don't verify release signatures until they've upgraded once.

### New

- **Sign in with Active Directory or LDAP.** Settings → Directory sign-in connects your directory over LDAPS or StartTLS. People sign in with their account name, UPN or `DOMAIN\name`, nested groups map to roles as they do for single sign-on, and a Test connection button checks the setup. The sign-in page offers local, single sign-on and directory sign-in.
- **Provision users from Entra ID or Okta.** SCIM 2.0 creates and updates users and groups before anyone signs in, deactivates leavers, and sets role and add-on permissions from group membership. Each Organization gets its own token, shown once.
- **Publish your own apps and keep them updated.** Software now has three tabs: Installed, Custom apps and Catalog. A custom app has versions you can retire, an uninstall command and a detection rule. Upload a newer version and every device holding an older copy is offered it on its Updates tab, in Software → Updates and in the Deploy wizard. Nothing is downgraded. An Admin can give a user, or a single sign-on group, the Publish apps permission to manage their own apps. Publishing never includes deploying, and every action is audited.
- **See, deploy and remove certificates.** Each device lists the certificates in its machine stores on a Certificates tab, and Settings → Automation → Certificates lists them fleet-wide. A Certificate expiring alert fires per certificate, 30 days ahead by default. Upload a public certificate (private keys are refused), deploy it to Windows or Linux trust stores from the Deploy wizard, and remove one by thumbprint from one device or many.
- **A scorecard per benchmark.** Prevention → Benchmarks lists each benchmark that applies to your fleet with a plain result such as "Passes · 211 of 211 automated rules" and how many devices pass, have gaps or aren't measured yet. Every count opens the list it counted. STIG rules that no scanner can check are counted as needing a person and linked. Fix gaps previews what it would change and applies safe changes by default. Unsafe changes are added only when you opt in and type the device count, and they go to canary devices first.
- **Assign benchmarks and keep them fixed.** Assign a benchmark to the whole Organization, to groups or to single endpoints. When a rule that passed on an assigned device fails again, an assignment set to fix automatically re-applies safe settings. Anything else, including unsafe settings and two benchmarks wanting different values, raises a Benchmark drift alert. An accepted risk silences both.
- **Patch baselines.** A baseline is a reusable list of OS update families and apps that tracks the latest release. Its apps can come from the catalog or be your own published apps. Pin an app to hold it at a version: a pin is a ceiling, and nothing is downgraded. A policy's Apply baseline rule keeps its devices on the baseline. Each device and each baseline shows "N of M lines current", and which lines are behind, held by a release-age hold, ahead of a pin or in conflict with another baseline. Every count opens the rows it counted.
- **Know what a deploy will do before you approve it.** Review asks each Linux device's own apt or dnf what it would install, and lists Windows updates in the order they will install. Servicing stack updates always install first.
- **Watch installs as they run, and stop them honestly.** The Change page streams the output of apt, dnf and Windows Installer while an install runs, with a Stop button from the moment it starts. Stop says exactly what it did. Stopped before installing means nothing was installed, and an install already under way finishes and reports what it installed.
- **Alerts in Slack, Teams, Discord, Mattermost and ntfy.** Each is a notification channel next to email and webhooks.
- **Tamper protection is back.** Settings → Deployment sets one password that guards stopping the Windows agent, changing its server settings and uninstalling it, even for an administrator. Wrong attempts are audited against the device.
- **A Bastion page that answers the first questions.** It shows whether the Bastion is working, the last and next sync, how often it syncs (15 minutes, 1, 6, 12 or 24 hours; hourly by default) and the package cache. Remove a recommended app and the Bastion stops pre-caching it. A deploy that needs it still fetches it, and Allow again undoes the removal. Signatures, streams and the queue are one click away on Diagnostics.
- **Clearer alerts and a dashboard that leads with vulnerabilities.** A triggered alert names its endpoint, device, CVE or installation and links to it. Vulnerability age and Severity trend come first on the dashboard, and the page follows one range: 7 days, 30 days (default), 90 days or 1 year. CPU, memory and volume meters use the limits of your performance rules, and Live → Processes sorts by every column.
- **A cleaner, more consistent interface.** Dates read like `7 Nov 2026, 13:42 UTC` everywhere, numbers carry thousands separators, and from 1700px wide, pages use the extra space instead of leaving it empty. The agent's tray portal matches.

### Fixed

- **The same open alert was listed again each time its cooldown passed.** A condition that keeps holding now updates its open alert instead of adding a row, a notification and an audit entry. The cooldown only applies after the alert clears.
- **Someone who signed in by both single sign-on and LDAP was re-linked on every sign-in.** Each sign-in method now keeps its own link.
- **The tamper protection switch could not be turned on, and the right password still did not stop the agent.** Both work now, and `msiexec /x` on a protected agent asks for the password.
- **A reinstalled Windows agent refused to run an uninstall with "cannot prepare ...\hive".** It now creates the folder it was missing.
- **Updates for a custom app were missing from the device list's pending-update count, the Software page and the Deploy wizard.** They now count everywhere the device's Updates tab shows them.
- **A Linux install waited for the device's next check-in.** It is sent to the device as soon as it is created, or as soon as its files reach the server.
- **Custom app updates were missing from the update notification on the device and from the Patch Status report.** Both include them now.
- **Tables opened in no particular order, and rows could repeat or go missing between pages.** Lists open by name A to Z, with vulnerability, update and compliance lists worst first and history newest first. The sorted column shows its arrow, and paging never repeats or skips a row.
- **A bulk agent upgrade could choose an older release as the newest.** After the server indexed several agent releases at once, the bulk upgrade, the endpoints page and the drift count could pick the oldest one. They now always pick the highest version.
- **Package versions were listed out of order, with 9.9.9 above 10.0.0.** They are ordered by version now.
- **The dashboard took about a second to draw the 1-year trend.** It loads quickly now.
- **Score bands disagreed between pages.** Healthy, At risk and Critical, with a default of 80 and 60, are used everywhere.
- **Smoother pages.** A tall dialog's Create button stays in view, a New dialog opened by its address lands on the page that owns it, the Bastion page no longer jumps as it loads, phones use the full width, and a column you tick in a table shows at any width.

### Worth knowing

- **CSV and JSON report exports write dates as RFC 3339** (`2026-09-20T10:00:00Z`) rather than `2026-09-20 10:00:00 UTC`, so check anything that reads them.
- **Two pages are gone and return 404.** `/admin/app-catalogue` moved to Software → Custom apps and Catalog, and the Prevention baseline changelog page was removed. The top of the Prevention page still says when the baseline moved.
- **Wording changed.** "Fires" is now "Triggered alerts", "catalogue" is "catalog" in the interface, and the risk register's "Operations" category is "Coverage". A fresh install seeds a leaner set of default alert rules; existing rules are untouched.
- **The access log is also written to the data volume** as one JSON line per request, in `logs/access.<date>.jsonl`, kept for 90 days or 500 MB, so it survives recreating the container. `POSTURERMM_LOGGING__ACCESS_LOG_RETENTION_DAYS` and `POSTURERMM_LOGGING__ACCESS_LOG_MAX_MB` tune it, and `0` for either turns the file off; the container log keeps the lines. The lines hold client IPs and user emails.
- **Releases are signed.** A release whose signature is missing or wrong is no longer published, and agents verify it.
- **Moving agents to a new server or proxy is simpler.** An agent given the new server's certificate pin switches to it at its next start, and keeps trying every five minutes if the new server is down then. It never trusts a certificate the pin does not name. The production guide has a table of what to set for each move.
- **Release notes for every version are on one page.** The update notice in the app links to it, so after skipping versions you can read everything that changed since yours.
- **Several features need a device's agent on this version:** certificate inventory, deploy and removal (Windows and Linux), live install output and Stop, the Linux dry run in Review, and tamper protection (Windows only). Until then the device shows them as not reported or refuses the task with the reason.

<a id="v1.30.0"></a>

## Version 1.30.0

- **See who is signed in to a device and when they were last active.** Windows devices now report each signed-in account and when it last used keyboard or mouse. The Endpoints list has an Activity column showing the last active time, green within the last ten minutes and yellow after that. The device Overview lists each account with its status: Active, Away, Locked, Disconnected or Signed out. Input through PostureRMM's own Remote Desktop doesn't count. A device shows this once its agent is on this version; Linux doesn't report it.
- **User column on wider screens.** To make room for Activity, the Endpoints list shows User from 1728px wide. The account is still in the row's slide-over and on the device page.
- **Settings tree, finished.** The category tree stays in view as you scroll, a hub's lit row lists that hub's pages, and group headings no longer look like links. Lists are tree rows, and adding, editing or deleting an item leaves you on the refreshed list. Card headers state what is set, with their actions beside them.
- **Notification channels open inside Settings.** Channels and a channel's delivery health render in the Settings tree, and the Alerts page links there.
- **System health fixes.** "All N binaries" opens a real list, half-width Bastion tables fit, and pages with tabs name their sibling pages in the page head.
- **Smaller fixes.** A rejected licence code comes back as you typed it, and one Settings section's request no longer discards another's. The People & access card lists the first ten users of the Users page rather than the newest. Dependency updates, including one for a library release that was withdrawn.

Upgrading: no manual steps. Back up, then follow the upgrade steps; agents follow through your upgrade policy.

<a id="v1.29.0"></a>

## Version 1.29.0

- **Settings, redesigned.** Settings is now five long pages (Personal, Access, Fleet, Automation, System) under a category tree inside the page, and nothing is hidden behind a fold-out. People & access shows Users, Organizations and SSO together. Fleet setup puts enrollment keys, groups, tags and custom properties on one page, with a link to endpoints waiting for approval. Each section's actions sit in its header. Old settings links redirect.
- **System health has its own sidebar entry.** Feed status, Bastion and Scheduler health are tabs under it, and each role sees only the tabs it can use. Long ledgers like feed hydration and the reconciliation log show the latest 10 entries, with a link to the full list.
- **License in one place.** Your licence, installation ID and deactivation now live in Settings › Deployment, beside the catalogue age warning. Feed sync health shows the licence as a reading.
- **Vulnerability age and trend.** The dashboard and the fleet report show a vulnerability age × severity matrix beside a severity trend, with a strip of affected endpoints. The report adds CVE age and trend tables before the findings list.
- **One Patch-all for a CVE on Windows and Linux.** A CVE fixed on both platforms gets one Patch-all and one Deploy, which files a deployment per platform. Its count includes Linux distribution fixes. The CVE link opens exactly the endpoints the count included, and no Windows machine is offered a Linux package.
- **RHEL 10 compliance.** RHEL 10 is imported from DISA's released V1R1 STIG, and PostureRMM Recommended scores the same 46 requirements on RHEL 10 as on RHEL 9. A missing `grub.cfg` now reads not applicable on RHEL 9/10 and Debian 12/13.
- **Group-scoped users see only their groups' findings.** A user limited to some groups no longer sees vulnerability findings or trends from endpoints outside those groups.
- **Smaller fixes.** .NET 10 runtimes are listed once and matched to their CVEs, and a removed .NET runtime leaves no inventory row. The DISA STIG download reads DISA's current catalogue page. Webhook delivery holds no idle connections. App catalogue no longer has saved views. The audit log always shows who did each action.

Upgrading: no manual steps. Back up, then follow the upgrade steps; agents follow through your upgrade policy.

<a id="v1.28.1"></a>

## Version 1.28.1

- **Agent version on the Endpoints list.** A new Agent column shows each endpoint's agent version, sorts oldest first in version order (1.9 before 1.10), and marks an agent that is behind the current stable release. On a 1280px screen the IP column now appears from 1536px to make room.
- **You can see which version you are running.** Every signed-in user sees the version beside their email. Super admins get a notice on every page when a newer release is out, with a link to the upgrade steps.
- **Move two-factor authentication to a new phone yourself.** My account now lets you re-enrol your authenticator and regenerate your backup codes without an administrator. My account itself has a calmer layout and a more compact API key list.
- **Updates that are blocked by an open app ask instead of failing.** When an app update needs an app closed, the agent asks the user to close it and postpones; installers that are blocked wait rather than fail. The tray's progress notification stays on screen until the operation ends, names the app, and says what happened.
- **Clearer Bastion status.** When a content source cannot be reached, the egress card and its tooltip say "source unavailable" rather than counting it as a permanent refusal, and after a Bastion restart they say "not checked yet".
- **PostureRMM is in beta.** It is free for up to 50 endpoints, with every feature. Paid plans are not available yet. The licence page and documentation now say this and offer nothing for sale.
- **Smaller fixes.** Each page request uses far less memory, bulk actions announce their result reliably to screen readers, rows on a live list flash once when their state changes even while the page loads other content, preflight re-runs pass on a near-full disk, and the release page keeps only the latest release.

Upgrading: no manual steps. Back up, then follow the upgrade steps; agents follow through your upgrade policy.

<a id="v1.28.0"></a>

## Version 1.28.0

- **Agent version on the Endpoints list.** A new Agent column shows each endpoint's agent version, sorts oldest first in version order (1.9 before 1.10), and marks an agent that is behind the current stable release. On a 1280px screen the IP column now appears from 1536px to make room.
- **You can see which version you are running.** Every signed-in user sees the version beside their email. Super admins get a notice on every page when a newer release is out, with a link to the upgrade steps.
- **Move two-factor authentication to a new phone yourself.** My account now lets you re-enrol your authenticator and regenerate your backup codes without an administrator. My account itself has a calmer layout and a more compact API key list.
- **Updates that are blocked by an open app ask instead of failing.** When an app update needs an app closed, the agent asks the user to close it and postpones; installers that are blocked wait rather than fail. The tray's progress notification stays on screen until the operation ends, names the app, and says what happened.
- **Clearer Bastion status.** When a content source cannot be reached, the egress card and its tooltip say "source unavailable" rather than counting it as a permanent refusal, and after a Bastion restart they say "not checked yet".
- **PostureRMM is in beta.** It is free for up to 50 endpoints, with every feature. Paid plans are not available yet. The licence page and documentation now say this and offer nothing for sale.
- **Smaller fixes.** Bulk actions announce their result reliably to screen readers, rows on a live list flash once when their state changes even while the page loads other content, preflight re-runs pass on a near-full disk, and the release page keeps only the latest release.

Upgrading: no manual steps. Back up, then follow the upgrade steps; agents follow through your upgrade policy.

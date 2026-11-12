# PostureRMM release notes

What changed in every release, newest first. The hub keeps only the newest release's files, so upgrade to it directly; the notes of earlier releases stay on this page. To upgrade, follow the [upgrade steps](../README.md#upgrade).

Jump to: [1.32.0](#v1.32.0) · [1.31.0](#v1.31.0) · [1.30.0](#v1.30.0) · [1.29.0](#v1.29.0) · [1.28.1](#v1.28.1) · [1.28.0](#v1.28.0)

<a id="v1.32.0"></a>

## Version 1.32.0

Connect an AI client to PostureRMM, automate most of the console over the API, see how every posture score is built, and get a smoother first install and a clearer tray portal.

### Upgrading

Back up first, because database migrations cannot be undone. Then, in the folder holding your `docker-compose.yml`:

```bash
docker compose pull && docker compose up -d
```

There are no manual steps. The first start can take a minute or two on a large vulnerability history. Agents upgrade through your upgrade policy, and nothing needs forcing. Upgrade your agents promptly: the tray portal changes, the clean Windows stop, the kept Windows Update catalogue and the repaired feature list need an agent on this version.

If a script or AI client that uses Python 3.13 or newer refuses your install's certificate, see the optional step under Worth knowing.

### New

- **Connect an AI client.** PostureRMM answers the Model Context Protocol at `/mcp`, so an AI client such as Claude Code can read your fleet: posture, endpoints, vulnerabilities, compliance, alerts, changes and the audit log. There is nothing to install, and it works with no internet. Make a key under My account → API access. By default it can only look. A second key made with Allow AI writes can scan, apply safe fixes, cancel, stop or revert a Change, release or abort a canary, restart a service, tag endpoints and approve or reject a waiting computer, always within the key's role and groups. Revealing a stored credential, deleting an endpoint, managing users, Organizations or API keys, and security settings are never offered to an AI client. The guide is Connect an AI client.
- **Most console actions now have a documented API route.** Device actions, reverting or stopping a Change, enrollment approvals, groups and proxies, policies and maintenance windows, patch baselines, custom apps, the app catalog's icons and bulk enable, tags, property definitions, the script library and its approvals, alert rules, monitors, notification channels, messages, invitations, reports, network discovery, Bastion settings, licence sync, install keys, user permissions, stored credential rotation, scoring, retention and brand settings, and Organization delete can all be driven by a key. Creating a Change takes an `Idempotency-Key`, so a retried call never dispatches twice. Every key call now records its outcome (succeeded, refused or failed) in the audit log, and a refusal says why. Security settings stay in the console. Anything that runs code on your endpoints, such as scripts and installer uploads, is never offered to an AI client either.
- **Posture scores show their working.** Each pillar's coverage sits beside its score, and a line on the dashboard says how the score is built. Prevention weighs checks by severity (high 50%, medium 35%, low 15%). Patching and Performance count each fact once, with partial credit where a measurement has several thresholds, so one problem no longer fails several rules. A CVE with no fix is labelled "No fix available". The letter grade is gone from the console.
- **See and fix end-of-life apps on a device.** The device's Patching tab lists every end-of-life app with its end date, the supported line, whether that line is already installed, what the rule costs the device in Patching points, and Install or Uninstall buttons.
- **A tray portal that matches the console.** It uses the dashboard's score card, buttons and tables, draws a 7-day posture trend and shows each app's own icon on update rows. Check in, Run scan and Restart agent say what happened ("Checked in 20:41", "Just ran. Try again at 20:43"). When an update had to wait for a program you closed, the toast says it is installing, and the Updated toast offers Reopen for that program.
- **The server's certificate renews itself.** The generated certificate is checked daily and replaced, with the same CA and no restart, once fewer than 60 days remain, so an install left running for years never goes dark. Agents notice nothing. If you brought your own certificate, a banner warns Super Admins in the last 30 days.
- **A smoother first install.** A keyless endpoint waiting for approval now shows on Endpoints, the dashboard and the Install page, and notices approval in about 20 seconds instead of up to 5 minutes. The first feed sync starts at once and brings the compliance content first, so Prevention no longer waits behind the vulnerability data. The first-run button installs the first endpoint, a Linux endpoint shows its real distribution while it waits, and the Windows installer script checks Docker Desktop's memory and turns on start-at-sign-in.
- **New app versions reach the feed within the hour.** Winget updates are collected hourly instead of nightly. Your Bastion's sync interval decides when they reach you.

### Fixed

- **A second, different script sent to the same device within seconds was silently dropped**, while you were told it was queued. Tasks are now told apart by what they carry, so a different script or different reboot arguments is queued, and a double click still yields one task.
- **The Bastion's "Auto pre-cache" switch did nothing.** Turned off, the Bastion stops pre-caching recommended apps automatically and keeps the cache as it is, instead of letting it age out a week later. Allow again, Retry all pending and the apps a deploy needs still fetch. The switch is on the API too.
- **Schedules created on the Schedules tab never ran.** They run now, and schedules already stored with no next run are given their next occurrence on upgrade, without all firing at once.
- **A group-limited account could reach beyond its groups.** Approving an enrollment, moving an endpoint, approving a reinstall and sending a message to "all" now stay inside the groups the person is assigned to, and the refusal says so.
- **A renamed or moved group kept its old path, and a group switched to static could refill itself.** The path and everything under it update with the change, and a late evaluation no longer adds members.
- **A new endpoint raised dozens of critical-vulnerability alerts within a minute.** Its first scan is now one alert, "N critical CVEs, K on KEV", linking to that endpoint's Critical vulnerabilities. Later CVEs still alert one by one.
- **An endpoint nothing had measured counted as fine.** It reads "Not yet measured", and the dashboard's empty score spread names what it is waiting for.
- **Windows 10 and 11 PCs were scored on checks written for servers, and a fully scanned PC could read about 21% coverage.** Prevention counts only checks for the endpoint's own platform.
- **A third-party app on an older release line read as outdated.** Java 17 is compared with the newest Java 17, and a .NET 8 install that is end of life no longer fails twice.
- **Pinned per-user app copies were counted current while the device listed them behind.** The baseline view and the device now agree.
- **A stopped Windows Time service on a standalone PC failed the time rule.** A trigger-start service that synced recently is healthy.
- **Stopping the Windows agent logged an error and reported a failed exit.** Stop always finishes its drain and exits cleanly, and a reboot gets the same grace.
- **The Windows agent fetched its offline Windows Update catalogue (about 680 MB) again every day.** It is kept and reused, and an upgrade keeps the copy an older agent left.
- **A failed Windows features query wiped the device's feature list**, and feature-policy drift read Unknown until the next good collection. The last list is kept.
- **The agent log was noisy and unhelpful.** Probing the control pipe and a long outage no longer flood it, an announced restart is not a warning, and it says when the live connection is up, down or restored and what a deferred task is waiting for. Each signed-in user's tray logs to its own file.
- **`msiexec /x` failed on an agent with tamper protection off**, and an agent moved to a new server could be refused for good because of the old server's install key. Both work now, on Windows and Linux.
- **Message counts lost people as they answered, and pressing OK was not counted.** Recipients, Delivered, Dismissed and Clicked now read as a funnel.
- **Scores were printed on different scales.** The All Organizations leaderboard and the evidence pack print out of 100, and every score surface uses the same bands and rounding.
- **Large fleets were slow on the Vulnerabilities, Software, Changes and report pages.** They load faster.

### Worth knowing

- **Scores can move on upgrade without anything changing on your fleet.** The scoring model changed. Trend lines and 7-day changes that cross the change are withheld, and say so ("Scoring model updated to v10 on a date"). The coverage figure beside each pillar score on the Endpoints list is gone. It remains on the dashboard, the pillar pages and a device's own page.
- **Only Super Admins can change scoring weights, health bands and the self-service opt-in.** A group-limited Admin's Save is greyed out with the reason, in the console and over the API.
- **A group-limited Admin can no longer change network discovery.** Scan configurations and discovered devices cover the whole Organization, so creating, editing, deleting and scanning need an Admin who is not limited to groups. Helpdesk users see only network devices whose probe is in their groups.
- **A scheduled report runs with its owner's access as it is now.** If the owner was removed or disabled, it sends nothing and shows Failed with the reason. If the owner was demoted to Helpdesk, it covers only what that person can read. Create the report again under an account that should own it.
- **Optional: strict clients and an existing certificate.** A certificate this install generated before this release lacks an identifier that Python 3.13 and newer require, so those clients refuse it even with the CA trusted. A new install has it, and so does any certificate minted from now on. To re-mint yours now, follow "Reissuing the generated certificate" in the configuration guide. Enrolled agents keep working. Skip it if no strict client connects.
- **A Windows PC is for trying PostureRMM.** The server runs only while someone is signed in with Docker Desktop running. Put a server you leave unattended on Linux, or in a small Linux VM on Hyper-V.
- **Several features need the agent on this version:** the tray portal changes, the clean Windows stop, the kept catalogue, the repaired feature list and the improved Windows Time verdict. Until then the device behaves as before.

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

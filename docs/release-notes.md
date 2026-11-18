# PostureRMM release notes

What changed in every release, newest first. The hub keeps only the newest release's files, so upgrade to it directly; the notes of earlier releases stay on this page. To upgrade, follow the [upgrade steps](../README.md#upgrade).

Jump to: [1.35.0](#v1.35.0) · [1.34.1](#v1.34.1) · [1.34.0](#v1.34.0) · [1.33.2](#v1.33.2) · [1.33.1](#v1.33.1) · [1.33.0](#v1.33.0) · [1.32.1](#v1.32.1) · [1.32.0](#v1.32.0) · [1.31.0](#v1.31.0) · [1.30.0](#v1.30.0) · [1.29.0](#v1.29.0) · [1.28.1](#v1.28.1) · [1.28.0](#v1.28.0)

<a id="v1.35.0"></a>

## Version 1.35.0

Pending updates now count every copy that needs updating, including copies one user installed, and false .NET vulnerabilities are gone.

### Upgrading

Back up first, because database migrations cannot be undone. Then, in the folder holding your `docker-compose.yml`:

```bash
docker compose pull && docker compose up -d
```

There are no manual steps. Agents upgrade through your upgrade policy; the tray fix below needs an agent on this version.

### New

- **Per-user copies count as pending.** An app one user installed for themselves now counts in a device's "Pending (N)", in the endpoint list's patch count and its export, in the Patch Status report and on the device Overview, with the split between machine and per-user copies shown.
- **Node.js Current moves to Node.js LTS.** When winget stops publishing a package and another one carries the same product line on, as Node.js Current did with Node.js LTS, the device is offered the newer version as an in-place upgrade. A link is made only when the new installer itself proves it replaces the old one, never from the name.
- **A warning before an install path moves.** Before an update swaps an MSI install for an MSIX, the deploy wizard, the device's Pending updates and the tray say which copy is removed and that the new one lives under `C:\Program Files\WindowsApps`.
- **Pending update details.** Clicking an update on a device's Pending updates opens a side panel with its versions, owner and the CVEs it closes.
- **CVEs with no fix are grouped by product.** On a device's Vulnerabilities tab, CVEs that have no fix yet are grouped under the installed product they affect.
- **The disk fill forecast explains itself.** The rule shows which volume is growing, how fast, and when it would fill.
- **The device Overview matches the Endpoints page.** The posture header is the same one the Endpoints page uses, with each pillar's score and top finding.

### Fixed

- **.NET showed dozens of vulnerabilities it did not have.** Each .NET release line is now matched against its own version numbers.
- **The Vulnerabilities panel offered an update the Updates tab did not show.** Both read the same catalogue decision.
- **The tray's update list went blank during "Update all".** Updates it started stay listed while they install.
- **A device's Pending updates said "Deploy".** It says "Install".
- **A stray scrollbar appeared after the last tab on a device page.** It is gone.

### Security

This release includes security hardening. All installs before 1.35.0 should upgrade.

### Worth knowing

- **Pending counts may rise after upgrading.** Copies that a single user installed now count, as they already did on the device's Updates tab.
- **An MSIX update moves the app's install path.** Repoint anything pinned to the old path, such as a scheduled task or a firewall rule.

<a id="v1.34.1"></a>

## Version 1.34.1

Updates now reach apps that only ship as a zip or an MSIX, apps installed for a single user, and CVEs the National Vulnerability Database has not scored yet.

### Upgrading

Back up first, because database migrations cannot be undone. Then, in the folder holding your `docker-compose.yml`:

```bash
docker compose pull && docker compose up -d
```

There are no manual steps. Agents upgrade through your upgrade policy. Zip-packaged apps and the move from an MSI install to an MSIX install need an agent on this version; older agents are told to upgrade instead of trying.

### New

- **Zip-packaged apps can be updated.** About 500 apps that winget publishes as a zip around an installer, such as MobaXterm and Microsoft GameInput, can now be deployed. The agent checks the download, unpacks it, runs the installer inside, and cleans up.
- **Apps that moved to MSIX update cleanly.** When a new version ships only as an MSIX, as PowerShell 7.7 does, the agent removes the older MSI install and installs the MSIX in its place, instead of leaving two copies.
- **Updates for apps installed by one user.** Store apps and other apps a user installed for themselves, such as WinDbg, are offered their updates in the tray and on the device's Updates tab, and install in that user's session. Deploys, the deploy wizard and policies update each user's own copy the same way. The device shows these as "Update available (user)" with the user's name.
- **A severity for every CVE.** When the National Vulnerability Database has not scored a CVE yet, common for Linux kernel CVEs, PostureRMM uses the rating from Ubuntu, Debian or Red Hat, the rating Rocky Linux and AlmaLinux also follow. It is shown with its source, for example "Medium · Ubuntu", and is never presented as a CVSS score.
- **Each machine uses its own distribution's rating.** A device's vulnerabilities, counts and Patching score use the rating from that machine's own distribution. Fleet-wide lists show the highest rating any distribution gives.
- **SQL Server drivers by major version.** OLE DB Driver 18 and 19, and ODBC Driver 17 and 18, are listed and updated separately.

### Fixed

- **Hovering the Severity trend chart hid some of its layers.** The chart stays fully drawn while you hover.
- **Linux kernel CVEs were listed against a small tool instead of the kernel.** They are listed against the kernel, once per machine, on Debian, Ubuntu and Red Hat-family systems.
- **Thousands of CVEs read "Not yet scored".** They now carry their distribution's rating, and CVEs with only a CVSS 4.0 or 2.0 score show that score's severity.
- **A Windows Update scan sometimes failed while the update service was restarting.** The agent waits and tries once more.
- **An OLE DB Driver 18 install was offered Driver 19.** Each major version is offered its own updates.
- **On a phone, the search box cut off its placeholder text.** It shows in full.

### Security

Nothing.

### Worth knowing

- **Patching scores may drop after upgrading.** CVEs that were "Not yet scored" now have a severity and count toward Patching, as they would in other scanners.
- **"Medium · Ubuntu" and similar labels are distribution ratings, not CVSS scores.** Risk scores still use CVSS only.

<a id="v1.34.0"></a>

## Version 1.34.0

Updates now reach apps that only ship as a zip or an MSIX, apps installed for a single user, and CVEs the National Vulnerability Database has not scored yet.

### Upgrading

Back up first, because database migrations cannot be undone. Then, in the folder holding your `docker-compose.yml`:

```bash
docker compose pull && docker compose up -d
```

There are no manual steps. Agents upgrade through your upgrade policy. Zip-packaged apps and the move from an MSI install to an MSIX install need an agent on this version; older agents are told to upgrade instead of trying.

### New

- **Zip-packaged apps can be updated.** About 500 apps that winget publishes as a zip around an installer, such as MobaXterm and Microsoft GameInput, can now be deployed. The agent checks the download, unpacks it, runs the installer inside, and cleans up.
- **Apps that moved to MSIX update cleanly.** When a new version ships only as an MSIX, as PowerShell 7.7 does, the agent removes the older MSI install and installs the MSIX in its place, instead of leaving two copies.
- **Updates for apps installed by one user.** Store apps and other apps a user installed for themselves, such as WinDbg, are offered their updates in the tray and on the device's Updates tab, and install in that user's session. Deploys, the deploy wizard and policies update each user's own copy the same way. The device shows these as "Update available (user)" with the user's name.
- **A severity for every CVE.** When the National Vulnerability Database has not scored a CVE yet, common for Linux kernel CVEs, PostureRMM uses the rating from Ubuntu, Debian or Red Hat, the rating Rocky Linux and AlmaLinux also follow. It is shown with its source, for example "Medium · Ubuntu", and is never presented as a CVSS score.
- **Each machine uses its own distribution's rating.** A device's vulnerabilities, counts and Patching score use the rating from that machine's own distribution. Fleet-wide lists show the highest rating any distribution gives.
- **SQL Server drivers by major version.** OLE DB Driver 18 and 19, and ODBC Driver 17 and 18, are listed and updated separately.

### Fixed

- **Hovering the Severity trend chart hid some of its layers.** The chart stays fully drawn while you hover.
- **Linux kernel CVEs were listed against a small tool instead of the kernel.** They are listed against the kernel, once per machine, on Debian, Ubuntu and Red Hat-family systems.
- **Thousands of CVEs read "Not yet scored".** They now carry their distribution's rating, and CVEs with only a CVSS 4.0 or 2.0 score show that score's severity.
- **A Windows Update scan sometimes failed while the update service was restarting.** The agent waits and tries once more.
- **An OLE DB Driver 18 install was offered Driver 19.** Each major version is offered its own updates.
- **On a phone, the search box cut off its placeholder text.** It shows in full.

### Security

Nothing.

### Worth knowing

- **Patching scores may drop after upgrading.** CVEs that were "Not yet scored" now have a severity and count toward Patching, as they would in other scanners.
- **"Medium · Ubuntu" and similar labels are distribution ratings, not CVSS scores.** Risk scores still use CVSS only.

<a id="v1.33.2"></a>

## Version 1.33.2

A new look for the whole console, one-click vulnerability scans, wider vulnerability coverage from CVSS 4.0 and 2.0 scores and vendor-supplied product data, and pages that explain every finding they show.

### Upgrading

Back up first, because database migrations cannot be undone. Then, in the folder holding your `docker-compose.yml`:

```bash
docker compose pull && docker compose up -d
```

There are no manual steps. Agents upgrade through your upgrade policy. The tray portal's new look needs an agent on this version.

### New

- **A new look.** The console uses the Inter typeface, a darker and calmer palette, a single indigo accent for actions, and coloured status pills for scores and states. The sidebar lists only the main areas, with labels. Each page switches between its views from tabs under its title, and the sign-in page carries the product's branding. Endpoint lists show each machine's operating system as an icon, and search results and slide-overs show what kind of thing you are looking at. Every page now uses the same summary figures, tabs, tables, buttons, empty states and coloured notifications, and clicking a row in any list opens its details in a side panel beside the list. The agent's tray window, where users see and run their updates, takes the same typeface and colours once the endpoint runs this release's agent.
- **Endpoints at a glance.** The Endpoints page opens with your fleet's posture score and a card for each pillar naming its top finding. Click a pillar's health band to filter the list to those endpoints. The list refreshes every 30 seconds, and the endpoint count and Export sit in the page header.
- **One-click vulnerability scan.** A device's Collect actions include Vulnerability scan. It asks the agent for a fresh inventory and, on Windows, a Windows Update scan, then re-checks everything for CVEs. It is also in the Endpoints bulk actions and on the API. If one part cannot run, for example because a Windows Update scan is already running, the other still runs and you are told what was skipped.
- **Vulnerability scores for more CVEs.** When the National Vulnerability Database has no CVSS 3 score for a CVE, PostureRMM now uses its CVSS 4.0 or 2.0 score. Each score is labelled with its version in the console, exports and the API. New CVEs the database has not analysed yet are matched from the affected products their publisher listed.
- **Every finding explains itself.** The rule drawers on the Prevention, Patching and Performance pages say what was measured, what was expected and how to fix it. Each change names what started it and who, and you can retry a change's failed targets as a new change.
- **Sort any list.** Column headers sort on the remaining lists, including network discovery, baselines, the risk register, certificates, monitors, messaging, Bastion feed streams and Reports. Long lists such as scripts, alert rules and policies page through your whole history and state the true total.
- **One Reports list.** Report definitions and generated reports are one list, and a row click opens it.
- **A device page that groups by pillar.** A device has one Patching tab (pending updates, vulnerabilities, deployments and baselines) and one Performance tab with its live views. It also shows the device's last boot time, and a Linux endpoint's point release.
- **A redesigned Install agent page.** The install command sits beside a live checklist: checked in, enrolled, first scan done. When the endpoint arrives you see its name and address and can open it. The page opens on the platform most of your fleet runs.
- **Software opens on what your fleet runs most.** The Software page sorts by how many endpoints run each app and hides Windows' own built-in apps such as Calculator, Camera and Clock. They are one click away under Built-in apps. The Status column sorts, and an app with known CVEs but no update in the catalog now says so.
- **The whole winget catalog.** Beyond the apps we curate, PostureRMM now knows every app in winget, about 16,000 of them. It shows update available, and CVEs where the product match is certain, for any of them you run. An app links to your installed software only on its exact product or package code, never on its name. Apps winget publishes a silent install for can be deployed; the rest show as detect only. The Catalog tab pages through them with Curated, Discovered and All views.

### Fixed

- **Windows Update installs showed no progress.** A device now reports downloading, downloaded and installing.
- **A device with a fast clock read "Restart in 7h", and some times were off.** Every time a device reports is stored on server time, and clock skew is measured.
- **The CVE page said "No fix available" next to a fix it could not send.** It now says what applies, and Linux findings name their advisory.
- **An empty Updates tab claimed "no missing updates" before any scan had run.** It says so only after a complete scan.
- **Fleet Patching pages did not say how many endpoints had never been scanned.** They do now.
- **Signing in with a backup code from the two-factor form failed.** It works.
- **Dialogs holding a table were cramped and pushed Use off screen.** They are wider, and their cells wrap.
- **Counts were printed inconsistently.** Thousands are grouped everywhere, and open CVEs and findings use one vocabulary and one rounding.
- **A flat disk volume showed a fill projection.** It no longer does.
- **Dialogs, menus and toasts were a lighter grey than the pages behind them.** They now sit on the dark palette, one step above the panel.
- **Score columns had uneven gaps.** List columns are aligned consistently: text on the left, scores, counts and actions centred.
- **The dashboard trend chart carried a vertical "Model v10" label.** The chart is clean; the caption beneath it says when the scoring model changed.

### Security

This release includes security hardening. All installs before 1.33.2 should upgrade.

### Worth knowing

- **The console looks different.** Navigation moved: the sidebar holds the main areas, and the tabs under each page title switch between views. Nothing was removed.
- **Reboot dialogs now ask for a reason of at least ten characters.** The reason is kept with the change.
- **Exports and the endpoint API gained columns** for the CVSS figure and its version. A script that reads columns by position should be checked.
- **Built-in Windows apps are hidden from the Software page by default.** Use the Built-in apps view to see them. Agents on this version also recognise apps your hardware vendor or administrator pre-installed.
- **Clickable counts in tables are plain until you point at them.** They turn the link colour on hover or keyboard focus.
- **Users can no longer change their own email address.** An administrator changes it for them.
- **Administrators limited to groups have a narrower reach.** Creating groups, rolling out agents to the whole Organization, creating install keys not tied to a group, changing proxies, and linking an Organization to a PSA company need a super admin. An administrator with no assigned groups counts as limited.
- **API keys cannot sign in, change a password or manage two-factor sign-in** for anyone, at any role. Use the console for those.
- **Evidence packs exported before this version no longer verify against your server.** Export them again.
- **Remote sessions end after 12 hours.** Change it with `POSTURERMM_AUTH__REMOTE_SESSION_MAX_SECS`; `0` turns the limit off.
- **Behind a reverse proxy, the client address now comes from `X-Forwarded-For` only.** If your proxy sends only `X-Real-Ip`, set `POSTURERMM_SERVER__TRUST_X_REAL_IP=true`.
- **The installer download helper refuses addresses inside your network.** To use an internal mirror, add its IP address or range to the allowed list; `POSTURERMM_SERVER__DOWNLOAD_RELAY_ALLOWLIST` now sets that list.
- **Report exports prefix formula-like cells with an apostrophe.** A cell starting with `=`, `+`, `-` or `@` that is not a plain number shows a leading `'` in a spreadsheet.

<a id="v1.33.1"></a>

## Version 1.33.1

A new look for the whole console, one-click vulnerability scans, wider vulnerability coverage from CVSS 4.0 and 2.0 scores and vendor-supplied product data, pages that explain every finding they show, and a round of security fixes across the API, agent connections and stored secrets.

### Upgrading

Back up first, because database migrations cannot be undone. Then, in the folder holding your `docker-compose.yml`:

```bash
docker compose pull && docker compose up -d
```

There are no manual steps. Agents upgrade through your upgrade policy. The tray portal's new look needs an agent on this version.

### New

- **A new look.** The console uses the Inter typeface, a darker and calmer palette, a single indigo accent for actions, and coloured status pills for scores and states. The sidebar lists only the main areas, with labels. Each page switches between its views from tabs under its title, and the sign-in page carries the product's branding. Endpoint lists show each machine's operating system as an icon, and search results and slide-overs show what kind of thing you are looking at. Every page now uses the same summary figures, tabs, tables, buttons, empty states and coloured notifications, and clicking a row in any list opens its details in a side panel beside the list. The agent's tray window, where users see and run their updates, takes the same typeface and colours once the endpoint runs this release's agent.
- **Endpoints at a glance.** The Endpoints page opens with your fleet's posture score and a card for each pillar naming its top finding. Click a pillar's health band to filter the list to those endpoints. The list refreshes every 30 seconds, and the endpoint count and Export sit in the page header.
- **One-click vulnerability scan.** A device's Collect actions include Vulnerability scan. It asks the agent for a fresh inventory and, on Windows, a Windows Update scan, then re-checks everything for CVEs. It is also in the Endpoints bulk actions and on the API. If one part cannot run, for example because a Windows Update scan is already running, the other still runs and you are told what was skipped.
- **Vulnerability scores for more CVEs.** When the National Vulnerability Database has no CVSS 3 score for a CVE, PostureRMM now uses its CVSS 4.0 or 2.0 score. Each score is labelled with its version in the console, exports and the API. New CVEs the database has not analysed yet are matched from the affected products their publisher listed.
- **Every finding explains itself.** The rule drawers on the Prevention, Patching and Performance pages say what was measured, what was expected and how to fix it. Each change names what started it and who, and you can retry a change's failed targets as a new change.
- **Sort any list.** Column headers sort on the remaining lists, including network discovery, baselines, the risk register, certificates, monitors, messaging, Bastion feed streams and Reports. Long lists such as scripts, alert rules and policies page through your whole history and state the true total.
- **One Reports list.** Report definitions and generated reports are one list, and a row click opens it.
- **A device page that groups by pillar.** A device has one Patching tab (pending updates, vulnerabilities, deployments and baselines) and one Performance tab with its live views. It also shows the device's last boot time, and a Linux endpoint's point release.
- **A redesigned Install agent page.** The install command sits beside a live checklist: checked in, enrolled, first scan done. When the endpoint arrives you see its name and address and can open it. The page opens on the platform most of your fleet runs.
- **Software opens on what your fleet runs most.** The Software page sorts by how many endpoints run each app and hides Windows' own built-in apps such as Calculator, Camera and Clock. They are one click away under Built-in apps. The Status column sorts, and an app with known CVEs but no update in the catalog now says so.
- **The whole winget catalog.** Beyond the apps we curate, PostureRMM now knows every app in winget, about 16,000 of them. It shows update available, and CVEs where the product match is certain, for any of them you run. An app links to your installed software only on its exact product or package code, never on its name. Apps winget publishes a silent install for can be deployed; the rest show as detect only. The Catalog tab pages through them with Curated, Discovered and All views.

### Fixed

- **API keys could reach beyond their role.** A key could set up two-factor sign-in for its owner and get a session with the owner's full role, reset another user's two-factor sign-in or password, or create a deployment policy that ran with its owner's rights. A key now acts only within its own role, and sign-in, password and two-factor changes need a signed-in person.
- **Administrators limited to some groups could act outside them.** Agent rollouts, install keys, proxies, groups, posture fixes and the custom app preview now keep to the administrator's assigned groups.
- **An enrolled agent could affect other devices.** An agent could answer data requests meant for other devices, and an enrolling device could take over another device's enrollment. Each agent now affects only its own device, and a decommissioned device's live connection is closed at once.
- **Secrets could be read or reused.** Any signed-in user could read the Organization's evidence-pack signing key. A notification channel kept its stored password or token when its destination changed. A local-account password waiting for an offline device was stored readable. All three are fixed.
- **Organization administrators could reach shared settings.** They could change the install command of apps shared by every Organization, list every company in your PSA, and link their Organization to another customer's PSA company. Those are now super admin tasks.
- **The installer download helper could be steered.** It now checks every redirect against your allowed list and refuses addresses inside your network.
- **Remote sessions outlived revoked access.** Shell, remote desktop and file sessions now close when the password, sign-in or API key that opened them is revoked.
- **A client could fake its address behind a reverse proxy.** Address-based limits and the local network check now use the address your proxy records.
- **Report exports could carry spreadsheet formulas from device data.** Cells that would run as a formula are made safe.

- **Windows Update installs showed no progress.** A device now reports downloading, downloaded and installing.
- **A device with a fast clock read "Restart in 7h", and some times were off.** Every time a device reports is stored on server time, and clock skew is measured.
- **The CVE page said "No fix available" next to a fix it could not send.** It now says what applies, and Linux findings name their advisory.
- **An empty Updates tab claimed "no missing updates" before any scan had run.** It says so only after a complete scan.
- **Fleet Patching pages did not say how many endpoints had never been scanned.** They do now.
- **Signing in with a backup code from the two-factor form failed.** It works.
- **An administrator could reset two-factor authentication for a user in another Organization.** That is refused and changes nothing.
- **Dialogs holding a table were cramped and pushed Use off screen.** They are wider, and their cells wrap.
- **Counts were printed inconsistently.** Thousands are grouped everywhere, and open CVEs and findings use one vocabulary and one rounding.
- **A flat disk volume showed a fill projection.** It no longer does.
- **Dialogs, menus and toasts were a lighter grey than the pages behind them.** They now sit on the dark palette, one step above the panel.
- **Score columns had uneven gaps.** List columns are aligned consistently: text on the left, scores, counts and actions centred.
- **The dashboard trend chart carried a vertical "Model v10" label.** The chart is clean; the caption beneath it says when the scoring model changed.

### Worth knowing

- **The console looks different.** Navigation moved: the sidebar holds the main areas, and the tabs under each page title switch between views. Nothing was removed.
- **Reboot dialogs now ask for a reason of at least ten characters.** The reason is kept with the change.
- **Exports and the endpoint API gained columns** for the CVSS figure and its version. A script that reads columns by position should be checked.
- **Built-in Windows apps are hidden from the Software page by default.** Use the Built-in apps view to see them. Agents on this version also recognise apps your hardware vendor or administrator pre-installed.
- **Clickable counts in tables are plain until you point at them.** They turn the link colour on hover or keyboard focus.
- **Users can no longer change their own email address.** An administrator changes it for them.
- **Administrators limited to groups have a narrower reach.** Creating groups, rolling out agents to the whole Organization, creating install keys not tied to a group, changing proxies, and linking an Organization to a PSA company need a super admin. An administrator with no assigned groups counts as limited.
- **API keys cannot sign in, change a password or manage two-factor sign-in** for anyone, at any role. Use the console for those.
- **Evidence packs exported before this version no longer verify against your server.** Export them again.
- **Remote sessions end after 12 hours.** Change it with `POSTURERMM_AUTH__REMOTE_SESSION_MAX_SECS`; `0` turns the limit off.
- **Behind a reverse proxy, the client address now comes from `X-Forwarded-For` only.** If your proxy sends only `X-Real-Ip`, set `POSTURERMM_SERVER__TRUST_X_REAL_IP=true`.
- **The installer download helper refuses addresses inside your network.** To use an internal mirror, add its IP address or range to the allowed list; `POSTURERMM_SERVER__DOWNLOAD_RELAY_ALLOWLIST` now sets that list.
- **Report exports prefix formula-like cells with an apostrophe.** A cell starting with `=`, `+`, `-` or `@` that is not a plain number shows a leading `'` in a spreadsheet.

<a id="v1.33.0"></a>

## Version 1.33.0

A new look for the whole console, one-click vulnerability scans, wider vulnerability coverage from CVSS 4.0 and 2.0 scores and vendor-supplied product data, pages that explain every finding they show, and a round of security fixes across the API, agent connections and stored secrets.

### Upgrading

Back up first, because database migrations cannot be undone. Then, in the folder holding your `docker-compose.yml`:

```bash
docker compose pull && docker compose up -d
```

There are no manual steps. Agents upgrade through your upgrade policy. The tray portal's new look needs an agent on this version.

### New

- **A new look.** The console uses the Inter typeface, a darker and calmer palette, a single indigo accent for actions, and coloured status pills for scores and states. The sidebar lists only the main areas, with labels. Each page switches between its views from tabs under its title, and the sign-in page carries the product's branding. Endpoint lists show each machine's operating system as an icon, and search results and slide-overs show what kind of thing you are looking at. Every page now uses the same summary figures, tabs, tables, buttons, empty states and coloured notifications, and clicking a row in any list opens its details in a side panel beside the list. The agent's tray window, where users see and run their updates, takes the same typeface and colours once the endpoint runs this release's agent.
- **Endpoints at a glance.** The Endpoints page opens with your fleet's posture score and a card for each pillar naming its top finding. Click a pillar's health band to filter the list to those endpoints. The list refreshes every 30 seconds, and the endpoint count and Export sit in the page header.
- **One-click vulnerability scan.** A device's Collect actions include Vulnerability scan. It asks the agent for a fresh inventory and, on Windows, a Windows Update scan, then re-checks everything for CVEs. It is also in the Endpoints bulk actions and on the API. If one part cannot run, for example because a Windows Update scan is already running, the other still runs and you are told what was skipped.
- **Vulnerability scores for more CVEs.** When the National Vulnerability Database has no CVSS 3 score for a CVE, PostureRMM now uses its CVSS 4.0 or 2.0 score. Each score is labelled with its version in the console, exports and the API. New CVEs the database has not analysed yet are matched from the affected products their publisher listed.
- **Every finding explains itself.** The rule drawers on the Prevention, Patching and Performance pages say what was measured, what was expected and how to fix it. Each change names what started it and who, and you can retry a change's failed targets as a new change.
- **Sort any list.** Column headers sort on the remaining lists, including network discovery, baselines, the risk register, certificates, monitors, messaging, Bastion feed streams and Reports. Long lists such as scripts, alert rules and policies page through your whole history and state the true total.
- **One Reports list.** Report definitions and generated reports are one list, and a row click opens it.
- **A device page that groups by pillar.** A device has one Patching tab (pending updates, vulnerabilities, deployments and baselines) and one Performance tab with its live views. It also shows the device's last boot time, and a Linux endpoint's point release.
- **A redesigned Install agent page.** The install command sits beside a live checklist: checked in, enrolled, first scan done. When the endpoint arrives you see its name and address and can open it. The page opens on the platform most of your fleet runs.
- **Software opens on what your fleet runs most.** The Software page sorts by how many endpoints run each app and hides Windows' own built-in apps such as Calculator, Camera and Clock. They are one click away under Built-in apps. The Status column sorts, and an app with known CVEs but no update in the catalog now says so.
- **The whole winget catalog.** Beyond the apps we curate, PostureRMM now knows every app in winget, about 16,000 of them. It shows update available, and CVEs where the product match is certain, for any of them you run. An app links to your installed software only on its exact product or package code, never on its name. Apps winget publishes a silent install for can be deployed; the rest show as detect only. The Catalog tab pages through them with Curated, Discovered and All views.

### Fixed

- **API keys could reach beyond their role.** A key could set up two-factor sign-in for its owner and get a session with the owner's full role, reset another user's two-factor sign-in or password, or create a deployment policy that ran with its owner's rights. A key now acts only within its own role, and sign-in, password and two-factor changes need a signed-in person.
- **Administrators limited to some groups could act outside them.** Agent rollouts, install keys, proxies, groups, posture fixes and the custom app preview now keep to the administrator's assigned groups.
- **An enrolled agent could affect other devices.** An agent could answer data requests meant for other devices, and an enrolling device could take over another device's enrollment. Each agent now affects only its own device, and a decommissioned device's live connection is closed at once.
- **Secrets could be read or reused.** Any signed-in user could read the Organization's evidence-pack signing key. A notification channel kept its stored password or token when its destination changed. A local-account password waiting for an offline device was stored readable. All three are fixed.
- **Organization administrators could reach shared settings.** They could change the install command of apps shared by every Organization, list every company in your PSA, and link their Organization to another customer's PSA company. Those are now super admin tasks.
- **The installer download helper could be steered.** It now checks every redirect against your allowed list and refuses addresses inside your network.
- **Remote sessions outlived revoked access.** Shell, remote desktop and file sessions now close when the password, sign-in or API key that opened them is revoked.
- **A client could fake its address behind a reverse proxy.** Address-based limits and the local network check now use the address your proxy records.
- **Report exports could carry spreadsheet formulas from device data.** Cells that would run as a formula are made safe.

- **Windows Update installs showed no progress.** A device now reports downloading, downloaded and installing.
- **A device with a fast clock read "Restart in 7h", and some times were off.** Every time a device reports is stored on server time, and clock skew is measured.
- **The CVE page said "No fix available" next to a fix it could not send.** It now says what applies, and Linux findings name their advisory.
- **An empty Updates tab claimed "no missing updates" before any scan had run.** It says so only after a complete scan.
- **Fleet Patching pages did not say how many endpoints had never been scanned.** They do now.
- **Signing in with a backup code from the two-factor form failed.** It works.
- **An administrator could reset two-factor authentication for a user in another Organization.** That is refused and changes nothing.
- **Dialogs holding a table were cramped and pushed Use off screen.** They are wider, and their cells wrap.
- **Counts were printed inconsistently.** Thousands are grouped everywhere, and open CVEs and findings use one vocabulary and one rounding.
- **A flat disk volume showed a fill projection.** It no longer does.
- **Dialogs, menus and toasts were a lighter grey than the pages behind them.** They now sit on the dark palette, one step above the panel.
- **Score columns had uneven gaps.** List columns are aligned consistently: text on the left, scores, counts and actions centred.
- **The dashboard trend chart carried a vertical "Model v10" label.** The chart is clean; the caption beneath it says when the scoring model changed.

### Worth knowing

- **The console looks different.** Navigation moved: the sidebar holds the main areas, and the tabs under each page title switch between views. Nothing was removed.
- **Reboot dialogs now ask for a reason of at least ten characters.** The reason is kept with the change.
- **Exports and the endpoint API gained columns** for the CVSS figure and its version. A script that reads columns by position should be checked.
- **Built-in Windows apps are hidden from the Software page by default.** Use the Built-in apps view to see them. Agents on this version also recognise apps your hardware vendor or administrator pre-installed.
- **Clickable counts in tables are plain until you point at them.** They turn the link colour on hover or keyboard focus.
- **Users can no longer change their own email address.** An administrator changes it for them.
- **Administrators limited to groups have a narrower reach.** Creating groups, rolling out agents to the whole Organization, creating install keys not tied to a group, changing proxies, and linking an Organization to a PSA company need a super admin. An administrator with no assigned groups counts as limited.
- **API keys cannot sign in, change a password or manage two-factor sign-in** for anyone, at any role. Use the console for those.
- **Evidence packs exported before this version no longer verify against your server.** Export them again.
- **Remote sessions end after 12 hours.** Change it with `POSTURERMM_AUTH__REMOTE_SESSION_MAX_SECS`; `0` turns the limit off.
- **Behind a reverse proxy, the client address now comes from `X-Forwarded-For` only.** If your proxy sends only `X-Real-Ip`, set `POSTURERMM_SERVER__TRUST_X_REAL_IP=true`.
- **The installer download helper refuses addresses inside your network.** To use an internal mirror, add its IP address or range to the allowed list; `POSTURERMM_SERVER__DOWNLOAD_RELAY_ALLOWLIST` now sets that list.
- **Report exports prefix formula-like cells with an apostrophe.** A cell starting with `=`, `+`, `-` or `@` that is not a plain number shows a leading `'` in a spreadsheet.

<a id="v1.32.1"></a>

## Version 1.32.1

1.32.1 is the first published build of the 1.32 line: 1.32.0 was tagged but its publishing step failed, so everything below is new since 1.31.0.

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

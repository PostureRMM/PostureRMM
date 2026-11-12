# Connect an AI client (MCP)

<!-- Mirrored verbatim to PostureRMM/PostureRMM; never edit on the hub.
Each release publishes it there — deploy/release/check-public-docs.sh. -->

PostureRMM answers the Model Context Protocol (MCP) at `https://<your-host>/mcp`,
so an AI client such as Claude Code can read your fleet: its
posture score, endpoints, vulnerabilities, compliance results, alerts, changes
and audit log. It is part of the server. There is nothing to install, it works
on a network with no internet, and it is as current as your install.

**By default an AI client can look and cannot change anything.** It can act on
your fleet only through a key you made for that, and only for the few
operations listed under [What the AI client can change](#what-the-ai-client-can-change).

The rest of the operator guide is [PRODUCTION.md](PRODUCTION.md).

## 1. Make a key

The endpoint accepts an API key and nothing else. A sign-in session does not work
on it.

1. Sign in as the person the AI client should act as.
2. Open **My account**, find the **API access** card and choose **Create API key**.
3. Set **Role** to **Auditor**. An Auditor key reads the whole Organization and
   can never write, so it is the right key for an AI client that only looks. A
   key never exceeds its owner's role, and it is limited to its owner's groups,
   so a Helpdesk user's key sees only what that user sees in the console.
4. Leave **Allow AI writes** unticked (see [Letting an AI client make
   changes](#letting-an-ai-client-make-changes)).
5. Copy the key (`frt_…`). It is shown once.

Give the key an expiry, and revoke it from the same card when you are done.

Text in the results, such as hostnames, software names and log lines, comes from
the managed computers themselves. A model reading it can be misled by text
someone wrote into it, so give an AI client a key that cannot change anything
unless you want it to.

## 2. Trust the install's certificate

An install serves its own certificate authority (CA) unless you gave it a
certificate from a public one. If yours uses a public certificate, skip this
step.

Otherwise download the CA from your install. It needs no sign-in:

```bash
curl -k -o posturermm-ca.crt https://<your-host>/api/v1/tls/ca.crt
openssl x509 -in posturermm-ca.crt -noout -fingerprint -sha256
```

Before you trust the file, compare that fingerprint, ignoring colons and case,
with one you got from the server itself and not over the connection you are
about to trust:

- the fingerprint the installer printed when you installed (`preflight.sh`
  prints it with "check its certificate authority has this SHA-256
  fingerprint"), or
- on the server, `docker compose exec backend curl -s
  http://localhost:3000/api/v1/tls/fingerprint`. That route answers with no sign-in;
  it is the installer's own source.

A SuperAdmin can also read it in the console, under **Settings → License & this
install → Certificate trust**, which has a **Download CA certificate** button
for the file.

Node-based clients, which includes Claude Code, read extra authorities from
`NODE_EXTRA_CA_CERTS`:

```bash
export NODE_EXTRA_CA_CERTS=/path/to/posturermm-ca.crt
```

Set it in the shell that starts the client. Other clients take the file through
their own trust store: add it to the operating system's, or ask the client's
documentation for its CA setting.

## 3. Add the server to the client

**Claude Code:**

```bash
claude mcp add --transport http posturermm https://<your-host>/mcp \
  --header "Authorization: Bearer frt_…"
```

**Any client that reads a JSON configuration** takes the same two facts, the URL
and the header. In Claude Code's `.mcp.json` form:

```json
{
  "mcpServers": {
    "posturermm": {
      "type": "http",
      "url": "https://<your-host>/mcp",
      "headers": { "Authorization": "Bearer frt_…" }
    }
  }
}
```

A client that only speaks standard input and output can reach the same URL
through a bridge such as `mcp-remote`, passing the same header.

Then ask: *"What is our posture score, and what are the three fixes that would
raise it most?"*

## What the AI client can ask

| Tool | Answers |
|---|---|
| `fleet_posture` | The Posture Score, its grade and pillars, and how it moved |
| `posture_trend` | The score over 7 days to a year |
| `posture_actions` | The fixes that would raise the score, and accepted risks |
| `fleet_stats` | Counts: endpoints, open vulnerabilities, compliance, protection |
| `agent_health` | How well the agents are reporting |
| `list_endpoints`, `get_endpoint`, `endpoint_facets` | The endpoints, one endpoint, and the fleet's make-up |
| `fleet_vulnerabilities` | Open findings across the fleet, filterable to the worst or to known-exploited |
| `endpoint_services` | One endpoint's services or systemd units, to name one for a service change |
| `list_enrollment_requests` | Computers waiting to be admitted. Admin keys only |
| `endpoint_vulnerabilities`, `endpoint_vulnerability_summary` | One endpoint's findings, and their counts |
| `fleet_compliance_failures` | Compliance results across the fleet, filterable to failures |
| `endpoint_compliance`, `endpoint_compliance_summary` | One endpoint's results, and their counts |
| `list_alerts` | Triggered alerts |
| `list_changes`, `get_change` | Work dispatched to endpoints, and how far it got |
| `audit_log` | Who did what. Auditor and SuperAdmin keys only |

Lists come a page at a time, 25 rows by default. Every tool answers as the key's
owner would see it in the console: the key's role, its groups and its
Organization apply. The Posture Score is quoted the way the console shows it, as
a figure out of 100; until every pillar has measured something it has no grade
and the tool says it is not yet measured.

## Letting an AI client make changes

Make a second key for this, so the look-only key stays look-only. In **My account →
API access → Create API key**, tick **Allow AI writes**. The key can then
do what the tools below do, **within its role and groups**: the same checks the
console and the API apply. Give it the lowest role that can do the job, which
must be Operator, Admin or SuperAdmin: an Auditor or Helpdesk key cannot change
anything, so ticking the box on one is refused. You cannot tick the box on a key
that exists: make another and revoke the old one. The same choice is
`"allow_ai_writes": true` on `POST /api/v1/auth/api-keys`.

**A write-enabled key reaches only its owner's groups.** An Admin or Operator
key may change only machines in the groups its owner is assigned to. An owner
with no groups, even a SuperAdmin who made an Admin key, gets a key that is
refused every change, and the refusal says the account has no assigned groups.
Give the key the SuperAdmin role, which reaches every group, or make it as an
owner who has groups.

A key without the box sees no change tools, and asking for one is refused with a
message saying how to get such a key. The same holds for a key whose owner has
since been demoted to a role that cannot write: the key acts as the lower of its
own role and its owner's role right now, so it stops seeing change tools.

**Your AI client's approval prompt is the safeguard.** Every change tool is marked
as one that modifies things, so a client asks you before each call. Read what it
is about to do. Do not choose "always allow" for these tools, and do not run a
client with its approvals switched off against a key made for AI writes. Text from
the managed computers (a hostname, a software name, a log line) can contain
instructions aimed at an AI; an approval prompt in front of a person is what
stops them, and nothing the server could ask the model to confirm would.

### What the AI client can change

| Tool | What happens to real machines |
|---|---|
| `scan_endpoint` | Re-matches what one endpoint last reported against known vulnerabilities: applications by CPE, a Windows endpoint's missing updates, a Linux endpoint's packages against its distribution's advisories. Does not ask the agent to collect again, and changes nothing on the machine |
| `remediate_endpoint_safe` | Without `confirm`, counts the safe fixes. With it, queues them as a Change that edits the endpoint's configuration |
| `run_posture_action` | Runs a recommended action's automatic fix on every endpoint it is open on |
| `remediate_posture_action` | Applies a compliance or hardening fix on every endpoint it is failing on, or on those listed. A dangerous fix is never run: it waits for you in the console |
| `cancel_change` | Cancels a Change's pending tasks. A running task is not stopped |
| `stop_change_task` | Stops one running install of a Change, and cancels the steps queued behind it. An install that has started is not killed: it finishes and its result says what was installed |
| `revert_change`, `revert_change_target` | Undoes the fixes a Change applied, or the one on a machine, as a new Change that edits the machines' configuration back. A revert of several machines, of a fix that was verified as working, or that could cut off remote access waits for you in the console |
| `approve_canary`, `abort_canary` | Releases, or discards, the held remainder of a canary Change |
| `restart_service`, `start_service`, `stop_service` | Acts on one service or unit on one endpoint now. A vendor-owned service is left to the console |
| `tag_endpoints` | Adds, removes or replaces tags on up to 500 endpoints, which can change which policies reach them |
| `approve_enrollment`, `reject_enrollment` | Admits a waiting computer, or discards its request |

Some things are never tools, whatever the key: decrypting a stored credential,
revealing an escrowed secret, setting a local account's password, deleting an
endpoint, uninstalling an endpoint's agent, writing or approving a script, managing
users, Organizations or API keys, and publishing custom apps or changing the silent
arguments an installer runs with (scripts and installers are code that runs on
endpoints). Security settings (single sign-on, LDAP, SCIM, the
tamper password, licence activation) stay in the console.

## Limits and records

- Each key may make 600 requests a minute, as on the API. A tool call counts as
  two. Past the limit the answer is `429` with a `Retry-After`.
- Reads are not written to the audit log, as with the API. The access log
  records each request to `/mcp` with the key's user. A change tool call writes
  an audit entry before it runs, the same one a script's API call with that key
  writes, and it also names the tool. A call that is then refused, or fails, has
  one too. When the call is answered, a second entry records the outcome:
  `succeeded`, `refused` (a 4xx answer) or `failed` (a 5xx), with the HTTP
  status. The audit log shows it beside the first entry. A call with no outcome
  entry reads as unknown: the server stopped before it could record one.
- To act in another Organization, send an `X-Tenant-Id` header with the key, as
  on the API. A key made inside a client Organization stays in it.
- A rejected key, or no key, answers `401` before anything else is read.

# Single sign-on

PostureRMM signs administrators in through any OpenID Connect (OIDC) identity
provider, directly: Microsoft Entra ID, Okta, Google Workspace, or any other
compliant provider such as Authentik or Keycloak. It is free on every install.
SAML is not supported. Groups can set each user's role; see
**Map groups to roles** below.

Your own settings live in [configuration.md](configuration.md); the rest of the
operator guide is [PRODUCTION.md](PRODUCTION.md).

## What you need

- A **SuperAdmin** account. SSO is configured at **Settings → Single
  sign-on** (`/settings/sso`).
- `POSTURERMM_SERVER__PUBLIC_URL` set to the address your browser uses. The
  default is `https://<POSTURERMM_DOMAIN>:<POSTURERMM_HTTPS_PORT>`; see
  [configuration.md](configuration.md). The panel shows a warning while it is
  unset.
- A server that can reach your provider over HTTPS. Discovery and the token
  exchange are calls from the PostureRMM server to the provider, not from the
  browser, and redirects are not followed. On a network with no internet, run
  an internal provider and point **Issuer URL** at it (see the certificate
  note under generic OIDC).

## Rules that apply to every provider

- **Redirect URI.** Register `<public URL>/auth/oidc/callback` exactly, for
  example `https://rmm.example.com/auth/oidc/callback`. The panel shows it as
  **Redirect URI** with a copy button. A scheme, port or trailing-slash
  difference fails the sign-in.
- **Flow and scopes.** Authorization code with PKCE (S256), confidential
  client, scopes `openid email profile`.
- **One organization per install.** Only one organization may have SSO
  enabled. Saving **Enable single sign-on** while another organization has it
  on is refused. Sign-in is bound to the organization that configured the
  provider: it can reach that organization's users and no one else's.
- **Verified email.** The ID token must carry `sub`, and an `email` claim with
  `email_verified` set to `true` (Entra's `xms_edov`, below, stands in for it).
  A missing `email_verified` counts as `false`.
  The only exception is a user already linked to that `sub`, who signs in
  without either. Each section below says how its provider behaves.
- **Who gets in.** PostureRMM looks for the user by linked `sub`; failing that,
  by verified email among the organization's users, and links them on the way.
  If nobody matches, **Auto-provision new users on first sign-in** creates a
  user with the **Auto-provision role** (Admin, Operator, Helpdesk or Auditor);
  with it off, only existing users can sign in. An email that belongs to a user
  of another organization is refused. A deactivated user or organization is
  refused.
- **Allowed email domains.** A comma or newline separated list, required with
  auto-provision (a save without one is refused). Matching by email and
  auto-provisioning accept only an email whose domain, after the last `@`, is
  listed exactly: `example.com` does not admit `sub.example.com` or
  `evil-example.com`. A user already linked to their `sub` is not checked. With
  the list empty and auto-provision off, there is no domain check. A provider
  can still authenticate people outside your domains, so also restrict who it
  authenticates.
- **MFA belongs to the provider.** An SSO sign-in does not ask for a PostureRMM
  authenticator code. Password sign-in stays available, and an SSO-created
  user has no usable password.
- **Failures say one thing.** The browser always shows "Single sign-on failed.
  Try again, or contact your administrator." The reason is in the server log
  and in the audit log as a failed `oidc` sign-in with a class such as
  `email_unverified`, `no_local_account`, `tenant_mismatch` or
  `email_domain_not_allowed`. Read that,
  not the browser.
- **Set it up with a second session open.** Keep a password-signed-in SuperAdmin
  tab until a test sign-in works.

In the panel, fill **Button label** (shown on the sign-in page), **Issuer URL**
(its `/.well-known/openid-configuration` is read at sign-in), **Client ID** and
**Client secret** (stored encrypted, never shown again), choose the
auto-provision settings, tick **Enable single sign-on** and save.

## Map groups to roles

Under **Groups to roles** add rows of provider group and role. On every sign-in
PostureRMM reads the **Groups claim** (default `groups`) from the ID token and
gives the user the role of the first row, top to bottom, whose group they hold.
Order matters, so put the most important row first. Match the group exactly as
the token spells it: Entra sends group object IDs, not names.

- **No rows means the mapping is off.** New users get the auto-provision role
  and existing users keep theirs.
- **No match** gives the auto-provision role, or refuses the sign-in when
  **Refuse sign-in when no group matches** is ticked.
- **Super Admin cannot be mapped**, and a Super Admin's role is never changed
  by a sign-in.
- If the token carries no groups (Entra drops them past 150), the user matches
  nothing.

## Microsoft Entra ID

1. In the Microsoft Entra admin center, **Entra ID → App registrations → New
   registration**. Choose **Single tenant only** for **Supported account
   types**. Under **Redirect URI** pick **Web** and paste the PostureRMM
   redirect URI.
2. Copy **Application (client) ID** into **Client ID**.
3. **Certificates & secrets → New client secret**; copy the secret **Value**
   (shown once) into **Client secret**.
4. **Issuer URL**: `https://login.microsoftonline.com/<Directory (tenant) ID>/v2.0`.
   Use your tenant ID, not `common` or `organizations`.
5. **Token configuration → Add optional claim → Token type: ID**, tick
   `xms_edov`, and save.

**Verified email.** Entra's v2.0 ID token never carries `email_verified`.
PostureRMM reads `xms_edov` instead and treats `true` as a verified email. A
missing or `false` `xms_edov` is unverified, so skip step 5 and every first
sign-in is refused as `email_unverified`. Entra's plain `email` claim is
user-editable, so PostureRMM never trusts it alone. Microsoft sets `xms_edov`
only when the user's email domain is a domain the tenant admin has verified
(**Entra ID → Custom domain names**), or the user is a Microsoft account,
Google account or one-time-passcode guest. Facebook and SAML/WS-Fed
accounts never have a verified domain. The claim is returned only when the
token carries `email`. If a token carries both `xms_edov` and `email_verified`
and they disagree, the email counts as unverified.

Sources: [register an app](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app),
[OIDC discovery and issuer](https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols-oidc),
[ID token claims](https://learn.microsoft.com/en-us/entra/identity-platform/id-token-claims-reference),
[optional claims (`xms_edov`)](https://learn.microsoft.com/en-us/entra/identity-platform/optional-claims-reference).

## Okta

1. In the Okta Admin Console, **Applications → Applications → Create App
   Integration**. **Sign-in method** **OIDC - OpenID Connect**, **Application
   type** **Web Application**.
2. Paste the PostureRMM redirect URI into **Sign-in redirect URIs** and save.
   The **Authorization Code** grant is preselected.
3. On the app's **General** tab copy **Client ID** and **Client secret**.
4. **Issuer URL**: `https://<your-okta-domain>` for the org authorization
   server. A custom authorization server (`https://<your-okta-domain>/oauth2/<id>`)
   also works but needs Okta's API Access Management in production.

**Verified email.** With the `email` scope Okta sends `email_verified`, true
when the user's Okta primary email has been verified. Okta documents no
setting that changes it, so a user whose primary email is unverified in Okta
is refused here until Okta verifies it.

**Gotcha.** Assign people or groups on the app's **Assignments** tab, or Okta
refuses the sign-in before PostureRMM sees it.

Sources: [authorization code flow](https://developer.okta.com/docs/guides/implement-grant-type/authcode/main/),
[OIDC claims and scopes](https://developer.okta.com/docs/reference/api/oidc/).

## Google Workspace

1. In the Google Cloud console, **Google Auth Platform → Clients → Create
   client**, application type **Web application**. Add the PostureRMM redirect
   URI under **Authorized redirect URIs**.
2. Copy the client ID and secret into **Client ID** and **Client secret**.
3. **Issuer URL**: `https://accounts.google.com`.

**Verified email.** Google's ID token carries `email_verified`; nothing to
configure.

**Gotcha.** `https://accounts.google.com` authenticates every Google account,
Gmail included. For this issuer, PostureRMM also requires Google's Workspace
domain claim (`hd`) to be present and listed in **Allowed email domains**
whenever auto-provision is on or the list is set, so a Gmail account is
refused. Still set the user type to **Internal** under **Google Auth Platform →
Audience**, so only members of your Cloud organization can authorize the app.

Sources: [OpenID Connect](https://developers.google.com/identity/openid-connect/openid-connect),
[OAuth client setup](https://developers.google.com/identity/protocols/oauth2/openid-connect),
[Internal and External audiences](https://support.google.com/cloud/answer/15549945).

## Any other OIDC provider (Authentik, Keycloak, ...)

Create a confidential OIDC client using the authorization code flow, with the
PostureRMM redirect URI and scopes `openid email profile`.

- **Issuer URL** is the provider's issuer, exactly as its
  `/.well-known/openid-configuration` reports it. Keycloak:
  `https://<host>/realms/<realm>`. Authentik:
  `https://<host>/application/o/<application-slug>/`.
- The ID token must include `email` and `email_verified: true`. A provider that
  lets users edit their own email should not mark it verified. Check what your
  provider emits by decoding a test ID token.
- The issuer must be reachable from the PostureRMM server. A certificate from
  a public CA works as is. For an internal CA, paste its certificate (PEM) into
  **IdP CA certificate**. It is trusted in addition to the public CAs, and a
  save is refused if it holds no valid certificate. Leave it blank otherwise.
  There is no option to skip certificate verification.

## Provision users and groups with SCIM

SSO signs people in. SCIM makes the provider create their accounts and groups
ahead of time, and deactivate them when they leave, with no one touching
PostureRMM. It is free on every install and inbound only: the provider calls
PostureRMM, which never calls out.

1. In **Settings → User provisioning** (`/settings/scim`, SuperAdmin) choose
   **Generate token**, copy it (it is shown once; only a hash is stored), tick
   **Enable user provisioning** and save. **Rotate token** replaces it and the
   old one stops working at once.
2. At the provider, use the **SCIM base URL** shown on that page
   (`<public URL>/scim/v2`) as the tenant URL and the token as the secret
   (bearer). In Entra ID this is an enterprise application's Provisioning →
   Automatic; in Okta, an application's Provisioning → SCIM 2.0 with
   HTTP Header authentication.
3. Push users with `userName` set to their email address, and push the groups
   that should grant access. **A group's name decides the role**, exactly as in
   **Map groups to roles** above: a mapping row names the group, and so does its
   `externalId` (Entra's object id, which is what its tokens carry). The first
   matching row sets the role and every matching row adds its permissions, as at
   sign-in, but at once and on every membership change. A person in no mapped
   group gets the auto-provision role, or read-only if SSO is not set up.

What it does and does not do:

- **Deactivating** a user (Entra's `active: false`, or a delete) disables the
  account and revokes its sessions and API keys immediately. Reactivating turns
  the account on; the old credentials stay revoked.
- A person who already has an account is **matched by email** inside the
  organization and adopted, never duplicated. Their first sign-in through your
  OIDC provider uses the same account.
- **Super Admin accounts are never created, changed or deactivated** by the
  provider, and no group can grant the role.
- Supported: Users and Groups with create, read, filter on `userName`,
  `externalId` or `displayName`, replace, patch and delete. Bulk, sort,
  password changes and the `.search` form answer 501.

## Sign in with Active Directory or LDAP

For a site with no OIDC provider. People type their directory name and
password; the directory checks the password and PostureRMM maps their groups to a
role through the same rows as **Map groups to roles** above. It is free on every
install and stays on your network: PostureRMM connects to your domain controller
and nothing else.

1. Create a **read-only service account** in the directory. It only looks users
   and groups up.
2. In **Settings → Directory sign-in** (`/settings/ldap`, SuperAdmin) enter the
   domain controller, port (636 for LDAPS, 389 for StartTLS), base DN, the
   service account's DN and password. Plain-text LDAP is not offered, and the
   domain controller's certificate is always verified. If it was issued by an
   internal CA, paste that CA's certificate (PEM); with a CA set, only that CA is
   trusted. The certificate must name the host you typed.
3. **Test connection** runs the real connection with the boxes' values and
   saves nothing. It reports whether the host answered, the certificate was
   trusted, the service account bound and the base DN was readable.
4. Add mapping rows. Name a group by its name (`PMM-Operators`) or its full DN;
   case does not matter. A user belongs to a group **through any depth of
   nesting**: if `Helpdesk` is inside `PMM-Operators`, its members match a row
   for `PMM-Operators`. The first matching row sets the role, every matching row
   adds its permissions, and no row can grant Super Admin.
5. Tick **Enable directory sign-in**. The login page gains a **Sign in with**
   link that opens the directory form (`/login?method=directory`); the local
   form and the OIDC button stay.

How it behaves:

- Users type their `sAMAccountName`, `DOMAIN\name` or UPN.
- **Two-factor authentication still applies.** A directory sign-in is a password
  sign-in, so a user without an authenticator enrols one first.
- A directory user is matched to an existing PostureRMM user by **mail address**
  inside the organization, or created if you tick **Create a PostureRMM user on
  first sign-in** (which needs allowed email domains). The link then follows the
  account's object GUID, so a rename in the directory does not orphan it.
- Failures read "Invalid username or password." in the browser whatever the
  cause; the reason (`ldap_tls_failed`, `ldap_user_not_found`, ...) is in the
  audit log and the server log.
- Only one organization can have directory sign-in enabled at a time.
- Kerberos single sign-on (no password prompt on a domain-joined PC) is not
  supported.

# Security Policy

**Contact:** security@posturermm.com
**Acknowledgment:** within 5 business days · **Status update:** within 30 days

PostureRMM welcomes responsible disclosure of security vulnerabilities, and commits to working
with researchers who report issues in good faith. This is a **disclosure program** — there are no
monetary bounties. Researchers who disclose valid vulnerabilities are credited below, with their
permission.

## Reporting a vulnerability

Email **security@posturermm.com** with:

- a clear description of the vulnerability and its impact
- reproduction steps — a proof-of-concept is strongly encouraged
- any relevant screenshots, logs or HTTP traffic captures
- the name or alias you would like credited, or a request to stay anonymous

Please do not open a public issue for a security report.

We acknowledge receipt within **5 business days** and give a status update or remediation timeline
within **30 days**. For critical vulnerabilities we aim to ship a patch within **14 days**.

## Safe harbour

We will not pursue civil or criminal action against researchers who:

1. Report the vulnerability to security@posturermm.com before any public disclosure.
2. Do not access, modify or exfiltrate data beyond what is minimally necessary to demonstrate it.
3. Do not test against systems they neither own nor have explicit written permission to test.
4. Do not conduct denial-of-service testing against any PostureRMM-operated or customer system.
5. Allow a reasonable remediation window — 30 days minimum, negotiable for complex issues — before
   public disclosure.

## Scope

**In scope**

| Component | Examples |
|---|---|
| Backend API | authentication bypass, authorisation failure, privilege escalation, injection (SQL, command), IDOR, SSRF, XXE, cryptographic weakness |
| Admin UI | XSS, CSRF, exposed secrets, clickjacking, open redirects that bypass authentication |
| Windows agent | privilege escalation to SYSTEM, agent secret exposure, unauthenticated RCE over the agent channel, binary tampering or signature bypass |
| Enrollment and agent protocol | MITM, replay, session fixation, weak cryptographic parameters |
| posturermm.com infrastructure | subdomain takeover, DNS misconfiguration, exposed credentials, email spoofing (DMARC/SPF/DKIM bypass) |

**Out of scope**

- Anything requiring physical access to the machine running the agent or the server.
- Social engineering.
- Denial of service, at the application or network layer.
- Third-party dependency issues, unless you can demonstrate exploitability in a PostureRMM
  context. Report those upstream and tell us — we will track them.
- Scanner output with no demonstrated impact, including missing security headers.
- TLS or cipher-suite recommendations with no demonstrated weakness.
- Anything in a self-hosted operator's own environment — their infrastructure hardening, their
  rate limiting, their network exposure.
- Anything requiring a compromised administrator account. Administrator access is already a trust
  boundary.

## Self-hosted deployments

PostureRMM is self-hosted: operators run it in their own environments. If you find a vulnerability
in **someone else's** PostureRMM deployment, contact that operator — we have no access to and no
control over third-party installations.

If the vulnerability is in the PostureRMM software itself and reproduces on a standard
installation, report it here.

## Disclosure

We follow coordinated disclosure:

1. You report privately to security@posturermm.com.
2. We confirm receipt and investigate.
3. We give you a remediation timeline.
4. We patch and ship a release.
5. We agree a disclosure date with you — typically 90 days from the report, or sooner once patched.
6. You publish.

If we fail to respond within 30 days, or miss an agreed deadline without explanation, you may
disclose at your discretion after giving reasonable further notice.

## Acknowledgments

*(None yet — be the first.)*

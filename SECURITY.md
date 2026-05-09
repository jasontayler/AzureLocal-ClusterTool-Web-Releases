# Security Policy

## Supported Versions

Only the latest release receives security fixes.

| Version | Supported |
|---|---|
| Latest release | Yes |
| Older releases | No — please upgrade |

---

## Reporting a Vulnerability

**Please do not open a public GitHub Issue for security vulnerabilities.**

Report security issues by emailing: **security@jasontayler.com**

Include as much detail as you can:
- A description of the vulnerability and its potential impact
- Steps to reproduce or a proof-of-concept
- The app version (visible in the left nav bar footer)
- Your environment (Windows Server version, IIS version, browser)

You will receive an acknowledgement within **5 business days**. If a fix is warranted, a patched release will be issued and you will be credited in the release notes (unless you prefer to remain anonymous).

---

## Scope

This tool runs **on-premises** inside your own network. The following are in scope:

- Authentication bypass or privilege escalation (Entra ID, Windows Auth, custom RBAC)
- Audit log tampering or bypass
- Sensitive data exposure (credentials, secrets stored in the database)
- Remote code execution via the web interface
- Insecure defaults in the setup scripts

The following are out of scope:

- Vulnerabilities that require physical access to the app server
- Vulnerabilities in third-party dependencies (report directly to those projects)
- Denial-of-service attacks against a single-tenant on-premises deployment

---

## Disclosure Policy

Once a fix is released, vulnerability details may be disclosed publicly after **30 days**
to allow users time to upgrade.

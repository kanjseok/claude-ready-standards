# Security Policy

> Guidelines for reporting vulnerabilities and the security update process for Claude Ready Standards.

---

## Supported Versions

| Version | Supported |
|---------|-----------|
| Latest (main branch) | Yes |
| Older commits | No |

Since this is a documentation repository, the "latest" version on the `main` branch is always the supported version.

---

## Reporting a Vulnerability

If you discover a security issue (e.g., exposed secrets in examples, insecure configuration recommendations, or misleading security guidance), please report it responsibly.

### How to Report

1. **GitHub Security Advisories (preferred)**: Use [GitHub Security Advisories](https://github.com/kanjseok/claude-ready-standards/security/advisories/new) to privately report the issue.
2. **GitHub Issues**: For non-sensitive issues (e.g., outdated security recommendations), open a [GitHub Issue](https://github.com/kanjseok/claude-ready-standards/issues).

### What to Include

- Which document contains the issue
- The specific section or line number
- A description of the security concern
- Suggested correction (if applicable)

### Response Timeline

| Action | Timeline |
|--------|----------|
| Acknowledgment of report | Within 48 hours |
| Initial assessment | Within 5 business days |
| Fix published | Within 30 days for critical issues |

---

## Disclosure Policy

We follow a responsible disclosure process:

1. **Report received** — acknowledgment sent to reporter within 48 hours.
2. **Assessment** — maintainers evaluate severity and impact.
3. **Fix developed** — correction prepared and reviewed.
4. **Fix published** — updated documentation merged to `main`.
5. **Public disclosure** — the issue and fix are documented in the commit history.

We ask that reporters refrain from public disclosure until a fix has been published.

---

## Security Update Process

- Security-related fixes are prioritized and merged directly to `main`.
- Fixes are communicated through commit messages prefixed with `docs(security):`.
- For critical issues affecting downstream projects that follow these standards, a notice will be added to the relevant document's header.

---

## Scope

This security policy covers:

- Configuration examples and code snippets in all standard documents
- Recommended security practices and checklists
- Authentication, authorization, and encryption guidance
- CI/CD workflow configurations

If you believe a recommended practice in this repository could lead to a security vulnerability in a downstream project, that qualifies as a reportable issue.

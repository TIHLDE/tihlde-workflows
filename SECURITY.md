# Security Policy

## Supported versions

Only the latest release on the `main` branch is supported with security updates.

## Reporting a vulnerability

If you discover a security vulnerability in this repository, **please do not open a public issue**.

Instead, report it privately:

1. **GitHub Security Advisories (preferred):** Go to the [Security Advisories](https://github.com/TIHLDE/workflows/security/advisories) tab and click **"Report a vulnerability"**.
2. **Email:** Send details to TIHLDE via <https://tihlde.org>.

## Scope

This repository contains reusable GitHub Actions workflows. Security concerns include:

- Credential leakage in workflow logs
- Unsafe defaults that could affect caller repositories
- Supply-chain risks in pinned Action versions

## Best practices for callers

- Pin workflow references to a specific SHA or major tag (`@v1`).
- Never pass secrets as plain-text inputs — use the `secrets` context.
- Review Dependabot PRs that update pinned Action versions in this repo.

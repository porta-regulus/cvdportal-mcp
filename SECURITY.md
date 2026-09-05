# Security

## Reporting a vulnerability in CVD Portal

Do not open a public issue for a security finding.

Use the contact published in our RFC 9116 file at
[cvdportal.com/.well-known/security.txt](https://cvdportal.com/.well-known/security.txt),
or file through our own disclosure portal. We run the same coordinated
disclosure process we sell.

## Reporting a vulnerability to a manufacturer

This repository is not a disclosure channel for third-party products. To report
a finding to a manufacturer that uses CVD Portal, follow
[skills/cvd-portal-disclosure/SKILL.md](skills/cvd-portal-disclosure/SKILL.md).

## Handling API keys

API keys are bearer credentials. Read them from an environment variable such as
`CVDPORTAL_API_KEY`. Never write one into a file, a log line, a commit, or a
chat message.

Keys are stored as SHA-256 hashes. A lost key cannot be recovered, only
replaced. Revoke and reissue in the dashboard under Settings, Developer.

A key can carry a scope list and a source IP allowlist. Narrow both to what the
integration needs.

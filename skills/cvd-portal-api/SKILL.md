---
name: cvd-portal-api
description: Manage your own organisation's CVD Portal workspace through the v1 REST API. Read CRA compliance status, file and export vulnerability records, check Article 14 reporting deadlines, upload an SBOM or SARIF scan, and register webhooks. Use when the user is the manufacturer operating the portal. Requires an Enterprise-plan API key.
---

# CVD Portal v1 API

Base URL: `https://cvdportal.com/api/v1`

Every route is scoped to the organisation that owns the API key. There is no way to read or write another tenant's data, and no parameter accepts a company identifier.

## Authentication

```
Authorization: Bearer <api-key>
```

Keys are generated in the dashboard under Settings, Developer. They are stored as SHA-256 hashes, so a lost key cannot be recovered, only replaced.

API access is gated to the Enterprise plan. A `403` with an upgrade message means the plan, not the key.

Never write a key into a file, a log line, or a chat message. Read it from an environment variable such as `CVDPORTAL_API_KEY`. If the user pastes one directly, use it for the call and do not repeat it back.

## Scopes

Keys carry scopes. A call fails with `API key lacks the required scope: <scope>` when the key is too narrow. Ask the user to widen the key rather than working around it.

| Endpoint | Method | Scope |
|---|---|---|
| `/compliance/status` | GET | `read` |
| `/vulnerabilities` | POST | `submissions:write` |
| `/vulnerabilities/export` | GET | `read` |
| `/vulnerabilities/{id}/article14` | GET | `read` |
| `/products` | GET | `read` |
| `/products` | POST | `products:write` |
| `/sbom` | GET | `read` |
| `/sbom` | POST | `sbom:write` |
| `/scan/sarif` | POST | `sarif:write` |
| `/webhooks/register` | POST | `webhooks:write` |
| `/gdpr/export` | GET | `gdpr:export` |

Keys created before scopes existed carry an empty scope list and pass every check.

## Common tasks

**Where does our CRA compliance stand?**

```
GET /compliance/status
```

**File a vulnerability into our own workspace.** `description` is the only required field.

```
POST /vulnerabilities
{ "description": "...", "productName": "...", "vulnerabilityType": "RCE" }
```

**Are we inside the Article 14 clock on a given report?**

```
GET /vulnerabilities/{id}/article14
```

Article 14 sets a 24-hour early warning, a 72-hour notification, and a 14-day final report for actively exploited vulnerabilities. Report what the endpoint returns. Do not compute deadlines yourself, and do not tell the user a filing has been made to ENISA or a national CSIRT — this API prepares filing packages, it does not transmit them to an authority.

**Export the vulnerability record**, for an auditor or a CSAF advisory:

```
GET /vulnerabilities/export
```

**Upload an SBOM** (CycloneDX or SPDX) or a **SARIF scan result**:

```
POST /sbom
POST /scan/sarif
```

Components without a resolvable package URL ecosystem cannot be matched against vulnerability feeds, so coverage of an uploaded SBOM is usually lower than its component count. Check what the response reports rather than assuming full coverage.

**Register a webhook** for submission events:

```
POST /webhooks/register
```

## Machine-readable references

- OpenAPI 3.0 spec: `https://cvdportal.com/openapi.json`
- API catalog (RFC 9727 linkset): `https://cvdportal.com/.well-known/api-catalog`
- Health: `https://cvdportal.com/api/health`
- Developer guide: `https://docs.cvdportal.com/cvd/api-overview`

The spec covers a subset of the routes listed above. Where they disagree, the table in this skill is the current surface.

## Reporting results back

When summarising submissions, audit entries, or exports in a chat interface, omit reporter email addresses and IP addresses unless the user has asked for a specific record. These are personal data under GDPR and the audit log is append-only, so anything surfaced cannot be unlogged.

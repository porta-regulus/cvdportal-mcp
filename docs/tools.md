# Tool reference

Two servers. The researcher server at `/api/mcp/public` needs no key. The manufacturer server at `/api/mcp` needs an Enterprise-plan API key.

---

# Researcher server

`https://cvdportal.com/api/mcp/public`

No authentication. These three tools mirror endpoints an unauthenticated browser already reaches.

## `find_vendor_portal`

Resolve a manufacturer's disclosure portal. Call this before submitting, so a report reaches the right vendor. Never guess a slug.

### Input

| Field | Type | Required | Notes |
|---|---|---|---|
| `query` | string | Yes | Portal slug (`acme`), a custom domain (`disclose.acme.com`), or a URL. |

### Output

```json
{
  "companyName": "Example Manufacturing GmbH",
  "slug": "example-mfg",
  "portalUrl": "https://example-mfg.cvdportal.com",
  "securityTxtUrl": "https://example-mfg.cvdportal.com/.well-known/security.txt",
  "policyUrl": "https://example-mfg.cvdportal.com/policy",
  "disclosureContact": "security@example-mfg.example",
  "submitWith": { "tool": "submit_vulnerability_to_vendor", "slug": "example-mfg" }
}
```

A verified custom domain wins over the `.cvdportal.com` subdomain. Rate limit is 30 lookups per minute per IP.

## `submit_vulnerability_to_vendor`

File a report to a manufacturer's public portal.

A disclosure is a permanent record on the vendor's side and cannot be retracted through the API. Confirm the vendor and the finding with the person you are acting for before calling this.

### Input

| Field | Type | Required | Notes |
|---|---|---|---|
| `slug` | string | Yes | From `find_vendor_portal` or the vendor's `security.txt`. |
| `description` | string | Yes | 10 to 10,000 characters. |
| `productName` | string | No | Up to 255 characters. |
| `vulnerabilityType` | enum | No | See the list below. |
| `stepsToReproduce` | string | No | Up to 10,000 characters. |
| `impact` | string | No | Up to 5,000 characters. |
| `contactEmail` | string | No | Omit it and the report is anonymous, but the vendor cannot come back with questions and no email updates are sent. |
| `pgpKey` | string | No | Public key, for an encrypted reply. |

### Output

```json
{
  "referenceNumber": "CVD-CLX8F2K9",
  "trackingToken": "<opaque token>",
  "trackingUrl": "https://example-mfg.cvdportal.com/status/<token>",
  "createdAt": "2026-09-05T09:41:22.118Z",
  "status": "NEW"
}
```

Record the tracking token. It is the only way back to an anonymous report and it cannot be recovered.

Rate limit is 5 reports per minute, per portal, per IP. It is the same key and ceiling the web form uses, so switching transport does not raise it.

## `track_vulnerability_report`

Read the status of a report already filed.

### Input

| Field | Type | Required |
|---|---|---|
| `trackingToken` | string | Yes |

### Output

```json
{
  "referenceNumber": "CVD-CLX8F2K9",
  "status": "ACKNOWLEDGED",
  "vendor": "Example Manufacturing GmbH",
  "productName": "Gateway 4000",
  "vulnerabilityType": "RCE",
  "filedAt": "2026-09-05T09:41:22.118Z",
  "lastUpdatedAt": "2026-09-06T11:02:44.900Z"
}
```

It never returns the report body, reproduction steps, impact, contact email or PGP key. A tracking token proves possession, not authorship, so it reveals progress only. Rate limit is 10 lookups per minute per IP.

---

# Manufacturer server

`https://cvdportal.com/api/mcp`

Three tools. Each mirrors an existing v1 REST endpoint, so an agent over MCP can do what a script does over HTTP.

Every tool derives its tenant from the verified API key. No argument accepts a company identifier. A tool that fails returns a readable error string rather than raising, so an agent can recover.

---

## `submit_vulnerability`

File a coordinated vulnerability disclosure report against the authenticated company's portal.

Mirrors `POST /api/v1/vulnerabilities`. Requires the `submissions:write` scope.

### Input

| Field | Type | Required | Notes |
|---|---|---|---|
| `description` | string | Yes | What the vulnerability is. Minimum 1 character. |
| `productName` | string | No | Affected product name. |
| `vulnerabilityType` | enum | No | See the list below. |
| `stepsToReproduce` | string | No | |
| `impact` | string | No | |
| `contactEmail` | string | No | Researcher contact email. Must be a valid address. |
| `pgpKey` | string | No | Public key for an encrypted reply. |

`vulnerabilityType` is one of `XSS`, `SQL_INJECTION`, `CSRF`, `SSRF`, `RCE`, `IDOR`, `AUTH_BYPASS`, `INFO_DISCLOSURE`, `DOS`, `OTHER`.

### Output

```json
{
  "id": "clx8f2k9p0001abcd1234efgh",
  "status": "NEW",
  "createdAt": "2026-09-05T09:41:22.118Z",
  "message": "Vulnerability report received."
}
```

`status` starts at `NEW` and moves through `ACKNOWLEDGED`, `IN_PROGRESS`, then `RESOLVED` or `DISMISSED`.

### Note on direction

This tool files into your own workspace. It is the vendor recording a finding. A researcher disclosing to a third-party vendor uses `submit_vulnerability_to_vendor` on the researcher server instead, which needs no key.

---

## `get_compliance_status`

Return the authenticated company's CRA compliance summary and open recommendations.

Mirrors `GET /api/v1/compliance/status`. Requires the `read` scope. Takes no arguments.

### Output

```json
{
  "companyId": "clx8f2k9p0000abcd1234efgh",
  "companyName": "Example Manufacturing GmbH",
  "timestamp": "2026-09-05T09:41:22.118Z",
  "status": "NON_COMPLIANT",
  "checks": {
    "craVerified": false,
    "identityVerified": true,
    "iso29147Alignment": true,
    "secureEncryption": true
  },
  "recommendations": [
    "Complete the CRA Verification checklist in the dashboard."
  ]
}
```

`status` is `COMPLIANT` only when both `craVerified` and `identityVerified` are true.

This is a disclosure-readiness summary. It is not a conformity assessment and it is not a Declaration of Conformity.

---

## `generate_csaf_advisory`

Build the CSAF 2.0 VEX document for the advisory linked to a submission the company owns.

Requires the `read` scope, and the `csaf_export` feature, which sits on the Reporting plan and above.

### Input

| Field | Type | Required | Notes |
|---|---|---|---|
| `submissionId` | string | Yes | Id of a submission owned by the authenticated company. |

### Output

A CSAF 2.0 document with the VEX profile, returned as structured content.

The advisory must already exist for that submission. Create it in the dashboard first. If it does not exist, or belongs to another tenant, the tool returns `No advisory found for that submission.` rather than an empty document.

---

## Errors

| Message | Cause |
|---|---|
| `Unauthorized: no valid API key.` | The key is missing, expired, revoked, or outside its IP allowlist. |
| `This API key lacks the required scope: <scope>` | The key is too narrow. Widen it in the dashboard. |
| `Company not found for this API key.` | The tenant behind the key no longer exists. |
| `Machine-readable CSAF export requires the Reporting plan.` | Plan gate on `csaf_export`. |
| `No advisory found for that submission. Create the advisory in the dashboard first, then retry.` | The advisory does not exist, or is not yours. |

A 401 at the HTTP layer, before any tool runs, means the key failed validation or the company is below the Enterprise plan.

## Scopes

Keys carry an optional scope list. An empty list is a legacy key created before scopes existed, and it passes every check.

| Scope | Grants |
|---|---|
| `read` | Read vulnerabilities, products, compliance and SBOM metadata |
| `submissions:write` | Submit vulnerability reports |

The full scope list, including the write scopes used by the REST API but not by any MCP tool, is in [the API skill](../skills/cvd-portal-api/SKILL.md).

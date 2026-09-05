# Tool reference

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

This tool files into your own workspace. It is the vendor recording a finding. A researcher disclosing to a third-party vendor uses the public portal endpoint instead, which needs no key. See [the disclosure skill](../skills/cvd-portal-disclosure/SKILL.md).

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

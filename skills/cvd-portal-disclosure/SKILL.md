---
name: cvd-portal-disclosure
description: Report a security vulnerability to an EU manufacturer through their CVD Portal disclosure portal, track the report afterwards, and check whether a vendor publishes a coordinated disclosure contact. Use when disclosing a finding to a vendor, filing the same finding with several vendors at once, or checking a company's CRA Article 14 posture. No API key needed.
---

# Reporting a vulnerability through CVD Portal

CVD Portal hosts coordinated vulnerability disclosure (CVD) portals for EU manufacturers subject to the Cyber Resilience Act. Each manufacturer has a portal at `https://<slug>.cvdportal.com`, and some map a custom domain onto it.

Everything in this skill is public and unauthenticated. Do not send an API key to these endpoints.

## Before you submit

Confirm with the person you are acting for before filing anything. A disclosure is a permanent record on the vendor's side, it is not retractable through the API, and filing to the wrong vendor is a real cost to both parties.

Check that the vendor actually uses CVD Portal:

```
GET https://cvdportal.com/<slug>/.well-known/security.txt
```

An RFC 9116 response confirms the portal and gives the disclosure contact and policy URL. If it 404s, the slug is wrong or the vendor is elsewhere. Do not guess slugs.

## Filing one report

```
POST https://cvdportal.com/api/portal/<slug>/submit
Content-Type: application/json
```

Body fields:

| Field | Required | Notes |
|---|---|---|
| `description` | Yes | 10 to 10,000 characters. The finding itself. |
| `productName` | No | Free text, up to 255 characters. |
| `vulnerabilityType` | No | One of `XSS`, `SQL_INJECTION`, `CSRF`, `SSRF`, `RCE`, `IDOR`, `AUTH_BYPASS`, `INFO_DISCLOSURE`, `DOS`, `OTHER`. |
| `stepsToReproduce` | No | Up to 10,000 characters. |
| `impact` | No | Up to 5,000 characters. |
| `contactEmail` | No | Omit it and the report is anonymous, but the vendor cannot come back with questions and you get no email updates. |
| `pgpKey` | No | Your public key, if you want an encrypted reply. |

A `201` returns `submission.referenceNumber` and `submission.trackingToken`. Record both. The tracking token is the only way back to an anonymous report, and it is not recoverable.

Rate limit is 5 submissions per minute per portal per IP. A `429` carries the reset time; wait for it rather than retrying immediately.

## Filing the same finding with several vendors

For a shared component affecting multiple manufacturers:

```
POST https://cvdportal.com/api/portal/batch
Content-Type: application/json

{ "submissions": [ { "slug": "vendor-a", "description": "..." }, ... ] }
```

Each item takes the same fields as a single submission plus `slug`. Maximum 20 per request. Confirm the full vendor list with the user before sending, since one call can reach 20 organisations at once.

## Tracking a report

```
https://cvdportal.com/<slug>/status/<trackingToken>
```

This is a page rather than a JSON endpoint. It shows the current state and the vendor's acknowledgment against the 48-hour target.

## Checking a vendor's CRA posture

```
GET https://cvdportal.com/<slug>/.well-known/cra-compliance
```

Returns the vendor's own published disclosure-handling status against CRA Article 14, including whether they meet the 11 September 2026 minimum. It is a self-declaration by the manufacturer, not an audit or a certification, and should be reported that way.

## What not to do

- Do not file a report the user has not seen and approved.
- Do not include credentials, live customer data, or personal data of third parties in `description` or `stepsToReproduce`. Describe the class of data instead.
- Do not use the batch endpoint to probe for which slugs exist. Use security.txt.
- Do not treat a `cra-compliance` response as verification of anything beyond what the vendor asserts.

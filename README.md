# CVD Portal MCP Server and Agent Skills

Model Context Protocol server and agent skills for [CVD Portal](https://cvdportal.com), the coordinated vulnerability disclosure and EU Cyber Resilience Act compliance platform for manufacturers.

This repository holds the public interface. It is documentation, manifests and skills. The server itself is hosted, so there is nothing to install and nothing to build.

- **Endpoint** `https://cvdportal.com/api/mcp`
- **Transport** Streamable HTTP
- **Auth** Bearer API key
- **Tools** 3

## What it does

CVD Portal gives an EU manufacturer a branded disclosure portal where security researchers file vulnerability reports, and carries those reports through to the Cyber Resilience Act obligations that follow. Article 14 of the CRA requires a manufacturer to notify its CSIRT within 24 hours of learning that a vulnerability in its product is actively exploited.

This MCP server lets an AI agent do three things inside that workflow.

| Tool | Purpose | Scope required |
|---|---|---|
| `submit_vulnerability` | File a disclosure report into your own workspace | `submissions:write` |
| `get_compliance_status` | Read your CRA compliance summary and open recommendations | `read` |
| `generate_csaf_advisory` | Export a CSAF 2.0 VEX document for an advisory you own | `read` |

Full input and output schemas are in [docs/tools.md](docs/tools.md).

## Connect

### Claude Code

```bash
claude mcp add --transport http cvd-portal https://cvdportal.com/api/mcp \
  --header "Authorization: Bearer $CVDPORTAL_API_KEY"
```

### Claude Desktop, Cursor, and other clients that read a JSON config

```json
{
  "mcpServers": {
    "cvd-portal": {
      "type": "http",
      "url": "https://cvdportal.com/api/mcp",
      "headers": {
        "Authorization": "Bearer ${CVDPORTAL_API_KEY}"
      }
    }
  }
}
```

Generate a key in the CVD Portal dashboard under Settings, then Developer. Keys are stored as SHA-256 hashes, so a lost key cannot be recovered, only replaced.

## Access

The MCP server requires an Enterprise-plan API key. A 401 means the key, the plan, or the source IP. Keys carry an optional scope list and an optional IP allowlist.

Every tool derives its tenant from the verified key. No argument accepts a company identifier, and no tool can read or write another tenant's data.

Rate limiting is applied per key.

## Agent skills

Two skills in [skills/](skills/) describe the wider platform in a form an agent can load. They are served live at `https://cvdportal.com/.well-known/agent-skills/<name>/SKILL.md`.

**[cvd-portal-disclosure](skills/cvd-portal-disclosure/SKILL.md)** is for reporting a vulnerability to a manufacturer. It covers finding a vendor's portal through RFC 9116 `security.txt`, filing one report or a batch across several vendors, and tracking a report afterwards. It needs no API key and every endpoint it uses is public.

**[cvd-portal-api](skills/cvd-portal-api/SKILL.md)** is for the manufacturer operating the portal. It covers the v1 REST API, which is a wider surface than the three MCP tools, including SBOM upload, SARIF ingest, Article 14 deadlines, product registration and webhooks. It needs an Enterprise-plan API key.

Use the disclosure skill when you are the researcher. Use the API skill when you are the vendor.

## Standards

CVD Portal implements or produces these.

- EU Cyber Resilience Act, Regulation (EU) 2024/2847, Article 14 reporting
- CSAF 2.0 with VEX profile
- RFC 9116 `security.txt`
- CycloneDX and SPDX software bills of materials
- SARIF for scan findings
- ISO/IEC 29147 vulnerability disclosure and ISO/IEC 30111 handling

## Links

- Product [cvdportal.com](https://cvdportal.com)
- MCP server documentation [docs.cvdportal.com/cvd/mcp-server](https://docs.cvdportal.com/cvd/mcp-server)
- Agent skills documentation [docs.cvdportal.com/cvd/ai-agent-skills](https://docs.cvdportal.com/cvd/ai-agent-skills)
- REST API overview [docs.cvdportal.com/cvd/api-overview](https://docs.cvdportal.com/cvd/api-overview)
- OpenAPI specification [cvdportal.com/openapi.json](https://cvdportal.com/openapi.json)
- Security contact [cvdportal.com/.well-known/security.txt](https://cvdportal.com/.well-known/security.txt)

## Reporting an issue with this server

Report vulnerabilities in CVD Portal itself through the contact in our `security.txt`. Do not open a public issue for a security finding.

## Licence

MIT. See [LICENSE](LICENSE). The licence covers this documentation and the skills. It does not cover the hosted service, which is governed by the CVD Portal terms of service.

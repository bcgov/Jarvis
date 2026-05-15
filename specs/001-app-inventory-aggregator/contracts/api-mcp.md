# MCP Server Contract

**Feature**: 001-app-inventory-aggregator | **Date**: 2026-05-13

## Overview

The MCP (Model Context Protocol) server exposes application inventory tools to AI assistants and automation tools. It uses **Streamable HTTP transport** hosted as middleware within the ASP.NET Core API, and communicates via JSON-RPC 2.0 as defined by the MCP specification (2025-03-26).

All MCP traffic is served by the same ASP.NET Core process as the REST API.

## Server Info

```json
{
  "name": "jarvis-inventory",
  "version": "1.0.0",
  "protocolVersion": "2025-03-26"
}
```

## Transport

**Endpoint**: `POST /mcp`

The MCP server uses Streamable HTTP transport:
- Client sends JSON-RPC requests via HTTP POST to `/mcp`
- Server responds with JSON-RPC results, or streams via Server-Sent Events (SSE) for long-running operations
- All requests are served by the same IIS/Kestrel process as the REST API

## Authentication

The MCP server uses the same dual-mode authentication as the REST API. AI clients include a Bearer token in the HTTP `Authorization` header:

```
Authorization: Bearer <token>
```

**Recommended for MCP clients**: Use a Personal Access Token (PAT) rather than a KeyCloak JWT. PATs are stable (up to 90 days), don't require browser-based OAuth flows, and work with every MCP client.

**Creating a PAT for MCP use**:
1. Log into the Jarvis web UI
2. Navigate to your profile → Personal Access Tokens
3. Create a new token (e.g., name: "VS Code MCP", expiry: 90 days)
4. Copy the token (shown once) and configure your MCP client

**Example: GitHub Copilot in VS Code** (`.vscode/mcp.json`):
```json
{
  "servers": {
    "jarvis-inventory": {
      "type": "http",
      "url": "https://jarvis.example.gov.bc.ca/mcp",
      "headers": {
        "Authorization": "Bearer ${JARVIS_TOKEN}"
      }
    }
  }
}
```

Set `JARVIS_TOKEN` as an environment variable containing your PAT. Do not commit tokens to source control — the `jrv_` prefix is detectable by secret scanning tools.

## Tools

---

### search_applications

**Description**: Search the application inventory by various criteria. Returns summary information for matching applications.

**Input Schema**:

```json
{
  "type": "object",
  "properties": {
    "query": {
      "type": "string",
      "description": "Free-text search across acronym, name, and aliases"
    },
    "ministry": {
      "type": "string",
      "description": "Filter by ministry name (exact match)"
    },
    "status": {
      "type": "string",
      "enum": ["active", "dormant", "retired", "unknown", "maintenance"],
      "description": "Filter by application lifecycle status"
    },
    "technology": {
      "type": "string",
      "description": "Filter by technology tag (partial match)"
    },
    "isCritical": {
      "type": "boolean",
      "description": "Filter by critical system flag"
    },
    "limit": {
      "type": "integer",
      "default": 20,
      "maximum": 100,
      "description": "Maximum number of results to return"
    }
  },
  "required": []
}
```

**Output**: Text content with formatted application summaries.

```
Found 3 applications matching "payment":

1. PAY-GW (Payment Gateway)
   Ministry: Finance | Status: active | Critical: yes
   Tech: Java, Spring Boot | Last activity: 2026-05-10

2. PAY-RPT (Payment Reporting)
   Ministry: Finance | Status: active | Critical: no
   Tech: Python, Django | Last activity: 2026-04-28

3. PAY-OLD (Legacy Payment System)
   Ministry: Finance | Status: retired | Critical: no
   Tech: COBOL | Last activity: 2024-01-15
```

**Authorization**: reader, maintainer, administrator

---

### get_application

**Description**: Get complete details for a single application by its acronym.

**Input Schema**:

```json
{
  "type": "object",
  "properties": {
    "acronym": {
      "type": "string",
      "description": "The unique application acronym"
    }
  },
  "required": ["acronym"]
}
```

**Output**: Text content with full application details, formatted for readability.

```
Application: PAY-GW (Payment Gateway)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Ministry: Finance
Branch: Revenue Services
Status: active (confidence: high, source: jira_activity)
Critical System: YES

Contacts:
  Business Owner: Jane Smith (jane.smith@gov.bc.ca) [source: cmdb]
  Technical Lead: John Doe (john.doe@gov.bc.ca) [source: cmdb]

Tech Stack:
  Language: Java | Runtime: JDK 17
  Frameworks: Spring Boot, Spring Security
  Confidence: high

Code Location: Bitbucket / FINANCE
  - pay-gw-api (main) last commit: 2026-05-10
  - pay-gw-ui (main) last commit: 2026-05-08

Hosting: OpenShift
  PROD: pod-pay-gw-1, pod-pay-gw-2 (v3.2.1)
  TEST: pod-pay-gw-test-1 (v3.3.0-rc1)

URLs:
  PROD: https://pay-gw.gov.bc.ca
  TEST: https://pay-gw-test.gov.bc.ca

Links:
  CMDB: https://cmdb.gov.bc.ca/apps/PAY-GW
  Jira: https://jira.gov.bc.ca/projects/PAYGW

Provenance: CMDB, Jira, Bitbucket, Confluence (tier: green)

Notes:
  - PCI-DSS compliant since 2024
  - Scheduled for framework upgrade Q4 2026

Risks:
  - [medium] End-of-life framework: Spring Boot 2.x reaches EOL Nov 2026 (status: open)

Data collected: 2026-05-13T08:00:00Z
```

**Authorization**: reader, maintainer, administrator

---

### update_application

**Description**: Update fields on an existing application or create a new application entry. Performs field-level merge (only specified fields are changed).

**Input Schema**:

```json
{
  "type": "object",
  "properties": {
    "acronym": {
      "type": "string",
      "description": "The unique application acronym (creates new if not found)"
    },
    "source": {
      "type": "string",
      "description": "Name of the data source making this update (for provenance tracking)"
    },
    "fields": {
      "type": "object",
      "description": "Key-value pairs of fields to update. Only specified fields are modified.",
      "properties": {
        "fullName": { "type": "string" },
        "ministry": { "type": "string" },
        "branch": { "type": "string" },
        "status": { "type": "string", "enum": ["active", "dormant", "retired", "unknown", "maintenance"] },
        "isCritical": { "type": "boolean" },
        "notes": { "type": "array", "items": { "type": "string" }, "description": "Notes to append (not replace)" },
        "risks": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "title": { "type": "string" },
              "description": { "type": "string" },
              "severity": { "type": "string", "enum": ["low", "medium", "high", "critical"] },
              "status": { "type": "string" }
            },
            "required": ["title"]
          },
          "description": "Risks to append (not replace)"
        },
        "contacts": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "role": { "type": "string" },
              "name": { "type": "string" },
              "email": { "type": "string" }
            },
            "required": ["role", "name"]
          }
        },
        "technologies": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Technology tags to set"
        }
      }
    }
  },
  "required": ["acronym", "source", "fields"]
}
```

**Output**: Confirmation text with summary of changes.

```
Updated PAY-GW:
  - status: active -> maintenance
  - Added note: "Undergoing security patching"
  Source: security_scan | Updated at: 2026-05-13T14:30:00Z
```

**Authorization**: maintainer, administrator

---

### list_ministries

**Description**: List all distinct ministries that have applications in the inventory. Useful for discovering valid filter values.

**Input Schema**:

```json
{
  "type": "object",
  "properties": {},
  "required": []
}
```

**Output**: Text list of ministries with application counts.

```
Ministries in inventory:
  - Citizens' Services (45 applications)
  - Finance (32 applications)
  - Health (28 applications)
  - Education (15 applications)
  ...
```

**Authorization**: reader, maintainer, administrator

---

### list_technologies

**Description**: List all distinct technology tags in use across the inventory. Useful for discovering valid filter values and understanding the technology landscape.

**Input Schema**:

```json
{
  "type": "object",
  "properties": {
    "query": {
      "type": "string",
      "description": "Optional filter prefix for technology names"
    }
  },
  "required": []
}
```

**Output**: Text list of technologies with usage counts.

```
Technologies in inventory:
  - Java (89 applications)
  - Spring Boot (67 applications)
  - Angular (43 applications)
  - Python (38 applications)
  - .NET (31 applications)
  ...
```

**Authorization**: reader, maintainer, administrator

---

### get_application_provenance

**Description**: Get the data source history for a specific application, showing which sources provided which fields and when.

**Input Schema**:

```json
{
  "type": "object",
  "properties": {
    "acronym": {
      "type": "string",
      "description": "The unique application acronym"
    },
    "fieldName": {
      "type": "string",
      "description": "Optional: filter to a specific field's history"
    }
  },
  "required": ["acronym"]
}
```

**Output**: Formatted provenance history.

```
Provenance history for PAY-GW:

Field: status
  2026-05-13 08:00 [jira_activity] active -> maintenance
  2026-01-10 12:00 [cmdb_sync] unknown -> active

Field: contacts
  2026-05-01 10:00 [cmdb_sync] Updated business_owner
  2025-06-15 14:00 [manual_review] Added technical_lead

Sources present: CMDB, Jira, Bitbucket, Confluence
Confidence tier: green
```

**Authorization**: reader, maintainer, administrator

---

## Resources

The MCP server also exposes resources for read-only access to summary data.

### inventory://summary

**Description**: High-level inventory statistics.

**Content**:
```
Application Inventory Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total applications: 4,832
  Active: 3,210 | Dormant: 892 | Retired: 534 | Unknown: 156 | Maintenance: 40
  Critical systems: 127
Ministries: 24
Data sources: 8
Last updated: 2026-05-13T08:00:00Z
```

### inventory://recent-changes

**Description**: Applications modified in the last 7 days.

**Content**: List of recently modified applications with change summaries.

## Error Handling

Tool errors are returned as MCP tool results with `isError: true`:

```json
{
  "content": [
    {
      "type": "text",
      "text": "Error: Application 'INVALID' not found. Use search_applications to find valid acronyms."
    }
  ],
  "isError": true
}
```

Common errors:
- Application not found (invalid acronym)
- Validation failure (missing required fields in update)
- Authorization denied (insufficient role for operation)
- Concurrency conflict (record modified by another request)

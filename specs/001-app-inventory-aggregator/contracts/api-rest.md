# REST API Contract

**Feature**: 001-app-inventory-aggregator | **Date**: 2026-05-13

## Base URL

```
https://{host}/api/v1
```

## Authentication

All endpoints (except health checks) require a valid Bearer token. The API supports two token types:

**KeyCloak JWT** — for browser-based SSO (Blazor WASM frontend):
```
Authorization: Bearer <jwt-token>
```

**Personal Access Token (PAT)** — for API clients, MCP connections, and automation:
```
Authorization: Bearer jrv_<token>
```

The middleware detects the token type by prefix:
1. `jrv_` prefix → PAT: hash with SHA-256, look up in PersonalAccessToken table, verify not expired/revoked, resolve user's current role.
2. Otherwise → JWT: validate signature against KeyCloak, extract email claim, resolve role from User table.

Both paths produce the same identity — downstream authorization is token-type agnostic.

Roles are resolved from the application's user table (matched by email claim from JWT or by token ownership for PATs).

## Error Response Format (RFC 7807 ProblemDetails)

```json
{
  "type": "https://jarvis.gov.bc.ca/errors/{error-type}",
  "title": "Human-readable error title",
  "status": 400,
  "detail": "Specific explanation of what went wrong and how to fix it",
  "instance": "/api/v1/applications/ABC",
  "errors": {
    "fieldName": ["Validation error message"]
  }
}
```

## Endpoints

---

### GET /applications

**Description**: List applications with search, filter, sort, and pagination.

**Authorization**: reader, maintainer, administrator

**Query Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| search | string | null | Full-text search on acronym, name, aliases |
| status | string | null | Filter: active, dormant, retired, unknown, maintenance |
| ministry | string | null | Filter by ministry (exact match) |
| isCritical | bool | null | Filter by critical system flag |
| technology | string | null | Filter by technology tag (partial match) |
| sortBy | string | "acronym" | Sort field: acronym, fullName, ministry, status, updatedAt |
| sortDir | string | "asc" | Sort direction: asc, desc |
| page | int | 1 | Page number (1-based) |
| pageSize | int | 50 | Items per page (max 200) |

**Response**: 200 OK

```json
{
  "items": [
    {
      "acronym": "APP1",
      "fullName": "Application One",
      "ministry": "Citizens' Services",
      "branch": "Digital Services",
      "status": "active",
      "isCritical": true,
      "updatedAt": "2026-05-13T10:00:00Z"
    }
  ],
  "page": 1,
  "pageSize": 50,
  "totalCount": 342,
  "totalPages": 7
}
```

---

### GET /applications/{acronym}

**Description**: Get full details for a single application.

**Authorization**: reader, maintainer, administrator

**Response**: 200 OK

```json
{
  "acronym": "APP1",
  "fullName": "Application One",
  "aliases": ["AppOne", "A1"],
  "jiraCategory": "DIGITAL",
  "ministry": "Citizens' Services",
  "branch": "Digital Services",
  "section": "Platform Team",
  "status": "active",
  "statusSource": "jira_activity",
  "statusConfidence": "high",
  "isCritical": true,
  "primaryUrl": "https://app1.gov.bc.ca",
  "urlSource": "cmdb",
  "dataQuality": "green",
  "lastDeployment": "2026-05-01T14:30:00Z",
  "lastCommit": "2026-05-10T09:15:00Z",
  "recentJiraActivity90d": 23,
  "recentJiraActivity365d": 87,
  "collectedAt": "2026-05-13T08:00:00Z",
  "lastActivity": "2026-05-13T08:00:00Z",
  "contacts": [
    {
      "role": "business_owner",
      "name": "Jane Smith",
      "email": "jane.smith@gov.bc.ca",
      "source": "cmdb"
    },
    {
      "role": "technical_lead",
      "name": "John Doe",
      "email": "john.doe@gov.bc.ca",
      "source": "cmdb"
    }
  ],
  "techStack": {
    "primaryLanguage": "Java",
    "runtime": "JDK 17",
    "frameworks": ["Spring Boot", "Angular"],
    "raw": "Java, Spring Boot, Angular",
    "source": "bitbucket",
    "confidence": "high"
  },
  "codeLocation": {
    "platform": "Bitbucket",
    "project": "DIGITAL",
    "repos": [
      {
        "slug": "app1-backend",
        "url": "https://bitbucket.org/bcgov/app1-backend",
        "defaultBranch": "main",
        "lastCommit": "2026-05-10T09:15:00Z",
        "scm": "bitbucket"
      }
    ]
  },
  "hosting": {
    "platform": "OpenShift",
    "environments": [
      {
        "name": "PROD",
        "servers": ["pod-app1-prod-1", "pod-app1-prod-2"],
        "deployedVersion": "2.3.1",
        "source": "openshift"
      }
    ]
  },
  "environmentUrls": [
    { "environment": "PROD", "url": "https://app1.gov.bc.ca" },
    { "environment": "TEST", "url": "https://app1-test.gov.bc.ca" },
    { "environment": "DLV", "url": "https://app1-dev.gov.bc.ca" }
  ],
  "links": {
    "cmdb": "https://cmdb.gov.bc.ca/apps/APP1",
    "jira": "https://jira.gov.bc.ca/projects/APP1",
    "confluence": "https://confluence.gov.bc.ca/display/APP1",
    "bitbucket": "https://bitbucket.org/bcgov/app1-backend"
  },
  "provenance": {
    "inCmdb": true,
    "inJira": true,
    "inBitbucket": true,
    "inConfluence": true,
    "inImisServers": false,
    "inTomcatInventory": false,
    "confidenceTier": "green"
  },
  "notes": ["Migration to OpenShift planned for Q3 2026"],
  "risks": [
    {
      "id": 1,
      "title": "End-of-life framework",
      "description": "Spring Boot 2.x reaches EOL in Nov 2026. Upgrade to 3.x required.",
      "severity": "medium",
      "status": "open",
      "source": "security_scan",
      "createdAt": "2026-04-15T10:00:00Z"
    }
  ],
  "fieldProvenance": [
    {
      "fieldName": "status",
      "source": "jira_activity",
      "updatedAt": "2026-05-13T08:00:00Z"
    }
  ],
  "createdAt": "2025-01-15T10:00:00Z",
  "updatedAt": "2026-05-13T08:00:00Z"
}
```

**Response**: 404 Not Found (if acronym does not exist)

---

### PUT /applications/{acronym}

**Description**: Create or update (upsert) a full application record. Creates if acronym does not exist; replaces all fields if it does.

**Authorization**: maintainer, administrator

**Request Body**: Same structure as GET response (excluding computed fields: createdAt, updatedAt, fieldProvenance).

**Response**: 200 OK (updated) or 201 Created (new)

**Response**: 409 Conflict (concurrent modification detected)

```json
{
  "type": "https://jarvis.gov.bc.ca/errors/concurrency-conflict",
  "title": "Concurrent modification conflict",
  "status": 409,
  "detail": "The record was modified by another request. Reload and retry.",
  "conflictingFields": ["status"]
}
```

---

### PATCH /applications/{acronym}

**Description**: Partial update — only specified fields are modified. Supports field-level merge for concurrent updates to different fields.

**Authorization**: maintainer, administrator

**Request Headers**:
```
If-Match: "{rowVersion}"  (optional, enables conflict detection)
```

**Request Body**: Partial update object — only include fields to change. Scalar fields are replaced; collection fields (`notes`, `risks`) are **appended** to existing values (not replaced).

```json
{
  "status": "retired",
  "notes": ["Retired as of May 2026"],
  "source": "manual_review"
}
```

The `source` field (top-level in the patch) is used to record provenance for all fields in this update.

**Note on collections**: `notes` and `risks` arrays in a PATCH are appended to the existing collection. To remove a note or risk, use the dedicated DELETE sub-resource endpoints (future). This matches the MCP `update_application` tool's append semantics.

**Response**: 200 OK

**Response**: 404 Not Found

**Response**: 409 Conflict (if If-Match provided and specific fields conflict)

---

### DELETE /applications/{acronym}

**Description**: NOT SUPPORTED. Returns 405 Method Not Allowed. Applications are retired via PATCH (soft delete only).

**Response**: 405 Method Not Allowed

```json
{
  "type": "https://jarvis.gov.bc.ca/errors/soft-delete-only",
  "title": "Deletion not supported",
  "status": 405,
  "detail": "Applications cannot be permanently deleted. Use PATCH to set status to 'retired'."
}
```

---

### POST /applications/bulk

**Description**: Bulk upsert multiple application records. Used for data source imports.

**Authorization**: maintainer, administrator

**Request Body**:

```json
{
  "source": "cmdb_sync",
  "applications": [
    { /* same structure as PUT body */ },
    { /* ... */ }
  ]
}
```

**Response**: 200 OK

```json
{
  "processed": 150,
  "created": 12,
  "updated": 138,
  "errors": [
    {
      "acronym": "BAD1",
      "error": "Validation failed: fullName is required"
    }
  ]
}
```

---

### GET /applications/{acronym}/provenance

**Description**: Get field-level provenance history for an application.

**Authorization**: reader, maintainer, administrator

**Query Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| fieldName | string | null | Filter to specific field |
| page | int | 1 | Page number |
| pageSize | int | 50 | Items per page |

**Response**: 200 OK

```json
{
  "items": [
    {
      "fieldName": "status",
      "source": "jira_activity",
      "updatedAt": "2026-05-13T08:00:00Z",
      "previousValue": "active",
      "newValue": "maintenance"
    }
  ],
  "totalCount": 5
}
```

---

### GET /technologies/suggest

**Description**: Auto-suggest existing technology tags for typeahead.

**Authorization**: reader, maintainer, administrator

**Query Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| q | string | Partial match query (min 2 chars) |
| limit | int | Max results (default 10, max 50) |

**Response**: 200 OK

```json
{
  "suggestions": ["Spring Boot", "Spring Security", "Spring Cloud"]
}
```

---

### GET /users

**Description**: List all provisioned users.

**Authorization**: administrator

**Response**: 200 OK

```json
{
  "items": [
    {
      "email": "jane.smith@gov.bc.ca",
      "displayName": "Jane Smith",
      "role": "administrator",
      "isActive": true,
      "lastLoginAt": "2026-05-13T09:00:00Z"
    }
  ]
}
```

---

### POST /users

**Description**: Pre-provision a user account with a role.

**Authorization**: administrator

**Request Body**:

```json
{
  "email": "new.user@gov.bc.ca",
  "role": "reader"
}
```

**Response**: 201 Created

**Response**: 409 Conflict (email already exists)

---

### POST /users/bulk

**Description**: Bulk provision multiple user accounts.

**Authorization**: administrator

**Request Body**:

```json
{
  "users": [
    { "email": "user1@gov.bc.ca", "role": "reader" },
    { "email": "user2@gov.bc.ca", "role": "maintainer" }
  ]
}
```

**Response**: 200 OK

```json
{
  "processed": 2,
  "created": 1,
  "skipped": 1,
  "errors": []
}
```

---

### PUT /users/{email}/role

**Description**: Change a user's role.

**Authorization**: administrator

**Request Body**:

```json
{
  "role": "maintainer"
}
```

**Response**: 200 OK

**Response**: 400 Bad Request (attempting to remove last administrator)

```json
{
  "type": "https://jarvis.gov.bc.ca/errors/last-admin",
  "title": "Cannot remove last administrator",
  "status": 400,
  "detail": "At least one active administrator must exist. Assign another administrator before changing this role."
}
```

---

### DELETE /users/{email}

**Description**: Deactivate a user account (soft delete - sets isActive to false). Also revokes all of the user's active Personal Access Tokens.

**Authorization**: administrator

**Response**: 204 No Content

**Response**: 400 Bad Request (last administrator)

---

### GET /users/me/tokens

**Description**: List the authenticated user's own Personal Access Tokens. Returns metadata only (not the token value). Includes active, expired, and revoked tokens.

**Authorization**: reader, maintainer, administrator (own tokens only)

**Response**: 200 OK

```json
{
  "items": [
    {
      "id": 1,
      "name": "VS Code MCP",
      "tokenPrefix": "jrv_a3",
      "expiresAt": "2026-08-11T00:00:00Z",
      "lastUsedAt": "2026-05-13T14:30:00Z",
      "createdAt": "2026-05-13T10:00:00Z",
      "isExpired": false,
      "isRevoked": false
    }
  ]
}
```

---

### POST /users/me/tokens

**Description**: Create a new Personal Access Token for the authenticated user. The plaintext token is returned **once** in the response and cannot be retrieved again.

**Authorization**: reader, maintainer, administrator (own tokens only)

**Request Body**:

```json
{
  "name": "VS Code MCP",
  "expiresInDays": 90
}
```

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| name | string | Required, 1-100 chars, unique per user | User-chosen label |
| expiresInDays | int | Required, 1-90 | Days until expiry |

**Response**: 201 Created

```json
{
  "id": 1,
  "name": "VS Code MCP",
  "token": "jrv_a3Bx9kLm2nPqRs4tUv6wXy8zA1cD3eF5gH7iJ",
  "tokenPrefix": "jrv_a3",
  "expiresAt": "2026-08-11T10:00:00Z",
  "createdAt": "2026-05-13T10:00:00Z"
}
```

**Response**: 400 Bad Request (max 10 active tokens, invalid expiry, duplicate name)

```json
{
  "type": "https://jarvis.gov.bc.ca/errors/token-limit",
  "title": "Token limit reached",
  "status": 400,
  "detail": "You have 10 active tokens. Revoke or let one expire before creating another."
}
```

---

### DELETE /users/me/tokens/{tokenId}

**Description**: Revoke one of the authenticated user's own tokens. Immediate and irreversible.

**Authorization**: reader, maintainer, administrator (own tokens only)

**Response**: 204 No Content

**Response**: 404 Not Found (token doesn't exist or belongs to another user)

---

### GET /users/{email}/tokens

**Description**: List another user's Personal Access Tokens. For administrator oversight and offboarding.

**Authorization**: administrator

**Response**: 200 OK (same shape as GET /users/me/tokens)

---

### DELETE /users/{email}/tokens/{tokenId}

**Description**: Revoke a specific user's token. For security incidents or offboarding.

**Authorization**: administrator

**Response**: 204 No Content

---

### DELETE /users/{email}/tokens

**Description**: Revoke all of a user's active tokens at once. For offboarding or credential rotation.

**Authorization**: administrator

**Response**: 200 OK

```json
{
  "revoked": 3
}
```

---

### GET /health

**Description**: Liveness check. Returns 200 if the process is running.

**Authorization**: None (public)

**Response**: 200 OK

```json
{ "status": "healthy" }
```

---

### GET /health/ready

**Description**: Readiness check. Returns 200 if SQL Server is reachable and the database schema is current.

**Authorization**: None (public)

**Response**: 200 OK / 503 Service Unavailable

```json
{
  "status": "ready",
  "checks": {
    "database": "healthy"
  }
}
```

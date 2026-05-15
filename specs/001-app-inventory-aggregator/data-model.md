# Data Model: Application Inventory Aggregator

**Feature**: 001-app-inventory-aggregator | **Date**: 2026-05-13

## Entity Relationship Overview

```
┌──────────────────┐       ┌───────────────────┐
│       User       │       │    Application    │
│                  │       │                   │
│  Id (PK)        │       │  Id (PK)          │
│  Email (UK)     │       │  Acronym (UK)     │
│  Role            │       │  FullName         │
│  CreatedAt       │       │  Ministry         │
│  IsActive        │       │  Branch           │
└──────────────────┘       │  Status           │
                           │  IsCritical       │
                           │  RowVersion       │
                           └────────┬──────────┘
                                    │
                 ┌──────────────────┼──────────────────────┐
                 │                  │                       │
    ┌────────────▼───┐   ┌────────▼────────┐   ┌─────────▼──────────┐
    │   Contact      │   │   Repository    │   │   Environment      │
    │                │   │                 │   │                    │
    │  Role          │   │  Slug           │   │  Name (DEV/TEST..) │
    │  Name          │   │  Url            │   │  Servers[]         │
    │  Email         │   │  Scm            │   │  DeployedVersion   │
    │  Source        │   │  Platform       │   │  Url               │
    └────────────────┘   │  Project        │   │  Source            │
                         └─────────────────┘   └────────────────────┘
                 │                  │                       │
    ┌────────────▼───┐   ┌────────▼────────┐   ┌─────────▼──────────┐
    │   TechStack    │   │     Link        │   │   FieldProvenance  │
    │                │   │                 │   │                    │
    │  Language      │   │  Type (cmdb..)  │   │  FieldName         │
    │  Runtime       │   │  Url            │   │  Source -> DataSrc │
    │  Frameworks[]  │   └─────────────────┘   │  UpdatedAt         │
    │  Confidence    │                         │  PreviousValue     │
    └────────────────┘            │            └────────────────────┘
                       ┌──────────▼──────────┐
                       │    Note             │
                       │    Risk             │
                       │                     │
                       │  Content            │
                       │  CreatedAt          │
                       │  Source             │
                       └─────────────────────┘
```

## Entities

### Application (Primary Entity)

The central entity. Uniquely identified by acronym. Supports soft-delete only (Status = retired).

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | Internal surrogate key |
| Acronym | string(50) | Unique, Required, Indexed | Business key, used for upsert matching |
| FullName | string(200) | Required | Display name |
| Ministry | string(100) | Required, Indexed | Organizational unit |
| Branch | string(100) | Nullable | Sub-organizational unit |
| Section | string(100) | Nullable | Sub-branch |
| JiraCategory | string(100) | Nullable | Jira project category |
| Status | enum | Required, Indexed | active, dormant, retired, unknown, maintenance |
| StatusSource | string(100) | Nullable | What determined the status |
| StatusConfidence | enum | Nullable | low, medium, high |
| IsCritical | bool | Default: false | Critical system flag |
| PrimaryUrl | string(500) | Nullable | Main application URL |
| UrlSource | string(100) | Nullable | Source of URL data |
| DataQuality | string(50) | Nullable | e.g., "ghost" or quality descriptor |
| ConfidenceTier | enum | Nullable | red, yellow, green |
| LastDeployment | DateTimeOffset | Nullable | Most recent deployment |
| LastCommit | DateTimeOffset | Nullable | Most recent code commit |
| RecentJiraActivity90d | int | Nullable | Jira tickets in last 90 days |
| RecentJiraActivity365d | int | Nullable | Jira tickets in last 365 days |
| CollectedAt | DateTimeOffset | Required | When data was last collected |
| LastActivity | DateTimeOffset | Required, Auto | Updated whenever the application record is modified (FR-010). Reflects the last time any change was made to this application's data. |
| CreatedAt | DateTimeOffset | Required, Auto | Record creation timestamp |
| UpdatedAt | DateTimeOffset | Required, Auto | Last modification timestamp |
| RowVersion | byte[] | Concurrency token | SQL Server native `rowversion` — auto-incremented on every update, mapped by EF Core as concurrency token |

**Validation Rules**:
- Acronym: alphanumeric + hyphens, 1-50 chars, case-insensitive unique
- Status: must be one of the defined enum values
- FullName: required, 1-200 chars
- Ministry: required, 1-100 chars

**State Transitions**:
- Any status -> retired (soft delete, irreversible through app)
- active <-> dormant <-> maintenance <-> unknown (bidirectional)
- retired -> (no transition back through app)

### ApplicationAlias

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| ApplicationId | int | FK -> Application, Required | |
| Alias | string(200) | Required | Alternative name |

### Contact

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| ApplicationId | int | FK -> Application, Required | |
| Role | string(50) | Required | business_owner, technical_lead, or custom role |
| Name | string(200) | Required | Contact display name |
| Email | string(254) | Nullable | Contact email |
| Source | string(100) | Nullable | Where this contact info came from |

**Validation Rules**:
- Role: required, 1-50 chars
- Name: required, 1-200 chars
- Email: valid email format if provided

### TechStack

One-to-one with Application (stored as owned entity or separate table for query flexibility).

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| ApplicationId | int | FK -> Application, Required, Unique | |
| PrimaryLanguage | string(100) | Nullable | Main programming language |
| Runtime | string(100) | Nullable | Runtime environment |
| Raw | string(1000) | Nullable | Original comma-separated value |
| Source | string(100) | Nullable | Data source |
| Confidence | enum | Nullable | low, medium, high |

### TechStackFramework

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| TechStackId | int | FK -> TechStack, Required | |
| Name | string(100) | Required | Framework name (free-text tag) |

### CodeLocation

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| ApplicationId | int | FK -> Application, Required, Unique | |
| Platform | string(100) | Nullable | e.g., Bitbucket, GitHub |
| Project | string(200) | Nullable | Project/org within the platform |

### Repository

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| CodeLocationId | int | FK -> CodeLocation, Required | |
| Slug | string(200) | Required | Repository identifier |
| Url | string(500) | Required | Full URL to repo |
| DefaultBranch | string(100) | Nullable | e.g., main, master |
| LastCommit | DateTimeOffset | Nullable | Last commit date |
| Scm | enum | Required | svn, bitbucket, github |

### HostingEnvironment

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| ApplicationId | int | FK -> Application, Required | |
| Platform | string(100) | Nullable | Hosting platform name |
| EnvironmentName | string(50) | Required | e.g., DEV, TEST, PROD |
| DeployedVersion | string(100) | Nullable | Current deployed version |
| Source | string(100) | Nullable | Data source |

### HostingServer

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| HostingEnvironmentId | int | FK -> HostingEnvironment, Required | |
| ServerName | string(200) | Required | Server hostname or identifier |

### EnvironmentUrl

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| ApplicationId | int | FK -> Application, Required | |
| EnvironmentName | string(50) | Required | DLV, PROD, TEST, TRAIN, etc. |
| Url | string(500) | Required | URL for this environment |

### ApplicationLink

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| ApplicationId | int | FK -> Application, Required | |
| LinkType | string(50) | Required | cmdb, jira, confluence, bitbucket |
| Url | string(500) | Required | Full URL |

### ApplicationNote

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| ApplicationId | int | FK -> Application, Required | |
| Content | string(2000) | Required | Note text |
| CreatedAt | DateTimeOffset | Required, Auto | When note was added |
| Source | string(100) | Nullable | Data source that provided note |

### ApplicationRisk

A known risk or vulnerability associated with an application (FR-007, FR-014).

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| ApplicationId | int | FK -> Application, Required | |
| Title | string(200) | Required | Short risk/vulnerability description |
| Description | string(2000) | Nullable | Detailed description, mitigation, or impact notes |
| Severity | enum | Nullable | low, medium, high, critical |
| Status | string(50) | Nullable | open, mitigated, accepted, resolved |
| CreatedAt | DateTimeOffset | Required, Auto | When risk was recorded |
| Source | string(100) | Nullable | Data source that provided this risk |

### DataSource

Central registry of data source systems that feed application inventory data (spec Key Entity).

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| Name | string(100) | Required, Unique, Indexed | Canonical source name (e.g., "cmdb", "jira", "bitbucket") |
| Description | string(500) | Nullable | Human-readable description of this data source |
| LastSyncAt | DateTimeOffset | Nullable | When this source last provided data |
| IsActive | bool | Default: true | Whether this source is currently feeding data |

### Provenance

One-to-one with Application. Tracks which data sources contain this application.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| ApplicationId | int | FK -> Application, Required, Unique | |
| InCmdb | bool | Default: false | Present in CMDB |
| InJira | bool | Default: false | Present in Jira |
| InBitbucket | bool | Default: false | Present in Bitbucket |
| InConfluence | bool | Default: false | Present in Confluence |
| InImisServers | bool | Default: false | Present in IMIS servers |
| InTomcatInventory | bool | Default: false | Present in Tomcat inventory |
| ConfidenceTier | enum | Nullable | red, yellow, green |

### FieldProvenance

Tracks which data source provided each field update (supports FR-009).

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| ApplicationId | int | FK -> Application, Required | |
| FieldName | string(100) | Required | Which field was updated |
| Source | string(100) | Required | Data source name |
| UpdatedAt | DateTimeOffset | Required | When the update occurred |
| PreviousValue | string(2000) | Nullable | Value before this update |
| NewValue | string(2000) | Nullable | Value after this update |

### User

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| Email | string(254) | Required, Unique, Indexed | Business key from SSO |
| DisplayName | string(200) | Nullable | Friendly name from SSO claims |
| Role | enum | Required | administrator, maintainer, reader |
| IsActive | bool | Default: true | Can be deactivated without deletion |
| CreatedAt | DateTimeOffset | Required, Auto | When account was provisioned |
| LastLoginAt | DateTimeOffset | Nullable | Last successful authentication |

**Validation Rules**:
- Email: required, valid email format, case-insensitive unique
- Role: must be one of the three defined values
- System constraint: at least one active administrator must exist at all times (FR-020)

### PersonalAccessToken

Self-service tokens for API and MCP authentication. Follows the GitHub/Azure DevOps PAT model.

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| Id | int | PK, auto-increment | |
| UserId | int | FK -> User, Required | Token owner |
| Name | string(100) | Required | User-chosen label (e.g., "VS Code MCP") |
| TokenHash | string(64) | Required, Unique, Indexed | SHA-256 hash of the token (hex). Plaintext never stored. |
| TokenPrefix | string(10) | Required | First 6 chars of the token (for identification in lists without exposing the full token) |
| ExpiresAt | DateTimeOffset | Required | Maximum 90 days from creation |
| RevokedAt | DateTimeOffset | Nullable | Set when revoked by user or admin |
| RevokedBy | string(254) | Nullable | Email of the user who revoked (self or admin) |
| LastUsedAt | DateTimeOffset | Nullable | Updated on each successful authentication |
| CreatedAt | DateTimeOffset | Required, Auto | When the token was created |

**Token format**: `jrv_` + 32 random bytes base62-encoded (~43 chars total). The `jrv_` prefix enables secret scanning.

**Validation Rules**:
- Name: required, 1-100 chars, unique per user
- ExpiresAt: required, must be ≤ 90 days from CreatedAt
- A user can have at most 10 active (non-expired, non-revoked) tokens
- Revocation is immediate and irreversible (no un-revoke)

**Auth resolution**: On each request with a `jrv_`-prefixed Bearer token:
1. Hash the token with SHA-256
2. Look up by TokenHash
3. Verify not expired (`ExpiresAt > now`) and not revoked (`RevokedAt IS NULL`)
4. Verify the owning User is active (`IsActive = true`)
5. Resolve the user's current Role (not a cached role from token creation time)

## Indexes

| Table | Index | Type | Purpose |
|-------|-------|------|---------|
| Application | IX_Application_Acronym | Unique | Business key lookup, upsert matching |
| Application | IX_Application_Ministry | Non-unique | Filter by ministry |
| Application | IX_Application_Status | Non-unique | Filter by status |
| Application | IX_Application_IsCritical | Non-unique | Filter by critical flag |
| Application | IX_Application_FullName | Non-unique | Search by name |
| User | IX_User_Email | Unique | Login lookup |
| PersonalAccessToken | IX_PAT_TokenHash | Unique | Token authentication lookup |
| PersonalAccessToken | IX_PAT_UserId | Non-unique | List tokens by user |
| Contact | IX_Contact_ApplicationId | Non-unique | Join performance |
| FieldProvenance | IX_FieldProvenance_ApplicationId | Non-unique | History lookup |
| FieldProvenance | IX_FieldProvenance_Source | Non-unique | Source attribution queries |
| ApplicationAlias | IX_ApplicationAlias_Alias | Non-unique | Alias search |
| ApplicationRisk | IX_ApplicationRisk_ApplicationId | Non-unique | Join performance |
| DataSource | IX_DataSource_Name | Unique | Source lookup by name |

## Enumerations

### ApplicationStatus
```
active | dormant | retired | unknown | maintenance
```

### UserRole
```
administrator | maintainer | reader
```

### Confidence
```
low | medium | high
```

### ConfidenceTier
```
red | yellow | green
```

### ScmType
```
svn | bitbucket | github
```

### RiskSeverity
```
low | medium | high | critical
```

## JSON-to-Entity Mapping

The following maps the provided JSON structure to the entity model:

| JSON Path | Entity.Field |
|-----------|-------------|
| `acronym` | Application.Acronym |
| `full_name` | Application.FullName |
| `aliases[]` | ApplicationAlias.Alias |
| `jira_category` | Application.JiraCategory |
| `ownership.ministry` | Application.Ministry |
| `ownership.branch` | Application.Branch |
| `ownership.section` | Application.Section |
| `ownership.business_owner` | Contact (Role="business_owner") |
| `ownership.technical_lead` | Contact (Role="technical_lead") |
| `ownership.team` | Contact (Role="team") |
| `ownership.contacts[]` | Contact (various roles) |
| `tech_stack.*` | TechStack entity |
| `tech_stack.frameworks[]` | TechStackFramework.Name |
| `code_location.platform` | CodeLocation.Platform |
| `code_location.project` | CodeLocation.Project |
| `code_location.repos[]` | Repository entities |
| `hosting.platform` | HostingEnvironment.Platform |
| `hosting.environments{}` | HostingEnvironment per key |
| `hosting.environments.*.servers[]` | HostingServer.ServerName |
| `urls.primary` | Application.PrimaryUrl |
| `urls.by_environment{}` | EnvironmentUrl per key |
| `urls.source` | Application.UrlSource |
| `lifecycle.status` | Application.Status |
| `lifecycle.last_deployment` | Application.LastDeployment |
| `lifecycle.last_commit` | Application.LastCommit |
| `lifecycle.recent_jira_activity_90d` | Application.RecentJiraActivity90d |
| `lifecycle.recent_jira_activity_365d` | Application.RecentJiraActivity365d |
| `lifecycle.status_source` | Application.StatusSource |
| `lifecycle.confidence` | Application.StatusConfidence |
| `provenance.*` | Provenance entity |
| `links.*` | ApplicationLink per type |
| `notes[]` | ApplicationNote.Content |
| `risks[]` | ApplicationRisk entities |
| `data_quality` | Application.DataQuality |
| `_collected_at` | Application.CollectedAt |

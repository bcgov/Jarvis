# Feature Specification: Application Inventory Aggregator

**Feature Branch**: `001-app-inventory-aggregator`

**Created**: 2026-05-13

**Status**: Draft

**Input**: User description: "Create an app that aggregates application inventories from various sources. For each app, I need to collect: acronym, name, ministry, branch, technologies, hosting details, status (active, dormant, retired, unknown), data sources, last activity, urls (app URL for each environment, source code repo, CI/CD link), contacts (business owner, technical lead), Critical system flag, notes, known risks and vulnerabilities. Employees will log in to the app via corporate SSO but I need various user roles in the app: administrator, maintainer, reader. Initially the UI will be readonly, with updates happening via an API and MCP server."

## Clarifications

### Session 2026-05-13

- Q: What happens when a user authenticates but has no pre-assigned role? → A: They see "Access Denied" — no default reader role is assigned.
- Q: How is the first administrator created after initial deployment? → A: First authenticated user automatically becomes admin when no users exist in the system (bootstrapping).
- Q: How do administrators pre-populate user accounts? → A: Via both the web UI (user management screen) and the API (for bulk/automated provisioning), using email as the unique identifier.
- Q: How is admin orphaning prevented? → A: System prevents demotion/removal of the last administrator account.
- Q: How are concurrent API updates to the same record handled (FR-015)? → A: Field-level merge — only conflicting fields are rejected; non-overlapping fields are merged automatically.
- Q: How does the MCP server authenticate? → A: Same token-based authentication as the REST API (shared identity provider).
- Q: What is the structure of the technologies field? → A: Free-text tags with auto-suggest from existing values (flexible but encourages consistency).
- Q: What is the application deletion policy? → A: Soft delete only — applications can only be marked "retired"; no permanent removal through the application. Preserves audit history.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - View Application Inventory (Priority: P1)

A reader logs into the application using their corporate SSO credentials and browses the consolidated inventory of all tracked applications. They can search, filter, and sort the list to find specific applications by name, acronym, ministry, status, or technology. They can view detailed information about any application including its contacts, URLs, hosting details, and known risks.

**Why this priority**: This is the core value proposition of the system. Without the ability to view and search the aggregated inventory, no other feature has meaning. This represents the primary read-only UI that all users interact with.

**Independent Test**: Can be fully tested by logging in, browsing the inventory list, searching for applications, and viewing application detail pages. Delivers immediate value to anyone needing to look up application information.

**Acceptance Scenarios**:

1. **Given** a reader has valid corporate credentials, **When** they authenticate via SSO, **Then** they are granted access to the application with reader permissions and can see the inventory dashboard.
2. **Given** applications exist in the inventory, **When** a reader searches by application acronym or name, **Then** matching results are displayed with key summary information (name, acronym, ministry, status).
3. **Given** a reader is viewing the application list, **When** they select an application, **Then** they see the full detail view including all tracked fields (contacts, URLs, technologies, hosting, risks, notes).
4. **Given** applications with various statuses exist, **When** a reader filters by status (e.g., "active"), **Then** only applications matching that status are shown.

---

### User Story 2 - Update Inventory via API (Priority: P1)

A maintainer or automated data source pushes application inventory updates through the API. This includes creating new application entries, updating existing fields, and marking applications with new statuses. The API validates incoming data and rejects malformed requests with clear error messages.

**Why this priority**: Since the UI is initially read-only, the API is the sole mechanism for populating and maintaining the inventory. Without it, the system has no data to display.

**Independent Test**: Can be tested by making API calls to create, update, and retrieve application records. Delivers value by enabling automated data feeds to populate the inventory.

**Acceptance Scenarios**:

1. **Given** a valid API credential with maintainer role, **When** a request is made to create a new application entry with all required fields, **Then** the application is created and a success response is returned.
2. **Given** an existing application in the inventory, **When** an API request updates specific fields (e.g., status changes from "active" to "retired"), **Then** the changes are persisted and the last activity timestamp is updated.
3. **Given** an API request with missing required fields or invalid data, **When** the request is submitted, **Then** a clear error response is returned indicating which fields are invalid and why.
4. **Given** a request from an unauthenticated or unauthorized client, **When** the API endpoint is called, **Then** the request is rejected with an appropriate error response.

---

### User Story 3 - Update Inventory via MCP Server (Priority: P2)

An AI assistant or automation tool connects to the MCP (Model Context Protocol) server to query and update the application inventory. The MCP server exposes tools for searching applications, reading details, and pushing updates, enabling AI-driven inventory maintenance workflows.

**Why this priority**: The MCP server extends the system's automation capabilities beyond traditional API integrations, enabling AI-assisted inventory management. It is a secondary data input channel after the REST API.

**Independent Test**: Can be tested by connecting an MCP client, invoking available tools to query and update inventory records, and verifying the changes are reflected in the system.

**Acceptance Scenarios**:

1. **Given** an MCP client connects with valid credentials, **When** it requests the list of available tools, **Then** it receives tool definitions for searching, reading, and updating application records.
2. **Given** an MCP client invokes a search tool, **When** it provides search criteria (e.g., ministry name), **Then** matching application records are returned.
3. **Given** an MCP client invokes an update tool with valid data, **When** the update is processed, **Then** the application record is updated and the change is visible through the UI and API.

---

### User Story 4 - Administer User Roles and Accounts (Priority: P2)

An administrator manages user accounts and roles within the application. Administrators can pre-populate user accounts (using email as the unique identifier) before users first log in, assigning them one of three roles: administrator, maintainer, or reader. Users who authenticate via corporate SSO without a pre-assigned role are denied access. Role assignments control what actions each user can perform.

**Why this priority**: Role-based access control ensures data integrity by preventing unauthorized modifications. Users must be explicitly provisioned by an administrator before they can access the system, ensuring only authorized personnel gain entry.

**Independent Test**: Can be tested by logging in as an administrator, pre-populating user accounts, viewing the user list, assigning roles to users, and verifying that role changes affect the permissions of those users. Can also verify that users without a pre-assigned role are denied access.

**Acceptance Scenarios**:

1. **Given** a user authenticates via SSO for the first time, **When** they have no pre-assigned role in the system, **Then** they are shown an "Access Denied" message and cannot access any application features.
2. **Given** an administrator is logged in, **When** they assign the maintainer role to a user, **Then** that user can make API updates on subsequent requests.
3. **Given** a user with reader role, **When** they attempt to call a write API endpoint, **Then** the request is rejected with an authorization error.
4. **Given** an administrator views the user management screen, **When** they search for a user, **Then** they see the user's current role and can modify it.
5. **Given** an administrator is on the user management screen, **When** they pre-populate a new user account by entering an email address and selecting a role, **Then** the account is created and ready for the user's first SSO login.
6. **Given** an administrator or automated system uses the API, **When** they submit a bulk user provisioning request with email addresses and roles, **Then** the user accounts are created and ready for first SSO login.
7. **Given** the system has just been deployed with no users, **When** the first user authenticates via SSO, **Then** they are automatically assigned the administrator role (bootstrapping).

---

### User Story 5 - Aggregate from Multiple Data Sources (Priority: P3)

The system tracks where each piece of application data originated. When the same application is reported by multiple data sources, the system reconciles the information and tracks provenance. Users can see which data source provided each piece of information and when it was last updated.

**Why this priority**: Data provenance and multi-source reconciliation ensure data quality and auditability. This is important for trust but is an enhancement to the core inventory viewing and updating capabilities.

**Independent Test**: Can be tested by submitting data for the same application from two different sources via the API, then viewing the application detail to confirm source attribution is displayed.

**Acceptance Scenarios**:

1. **Given** an application record exists from Source A, **When** Source B submits updated data for the same application (matched by acronym), **Then** the system records both sources and shows the most recent data with provenance.
2. **Given** an application has data from multiple sources, **When** a reader views the application detail, **Then** they can see which source provided each field and when it was last updated.

---

### Edge Cases

- What happens when two sources provide conflicting data for the same application field (e.g., different status values)? The most recent update takes precedence (last-write-wins), and the previous value is retained in source history.
- How does the system handle an application that exists in one source but not another? The application record is maintained with data from all available sources; missing fields from a particular source are simply not attributed to that source.
- What happens when a user's corporate SSO account is deactivated while they have an active session? The session is invalidated on the next request requiring authentication verification.
- How does the system behave when the API receives an update for an application acronym that does not yet exist? The system creates a new application record (upsert behavior).
- What happens when a data source submits an application with an acronym that matches an existing record but a different name? The name is updated (last-write-wins) and the change is tracked in the source history with both the old and new values.
- What happens when the system is deployed for the first time and no users exist? The first user to authenticate via SSO is automatically assigned the administrator role (bootstrapping), enabling them to provision other users.
- What happens when an administrator attempts to demote or remove the last administrator account? The system prevents the operation and returns an error indicating at least one administrator must exist at all times.
- What happens when an administrator attempts to delete an application? Applications cannot be permanently deleted; they can only be marked as "retired" (soft delete). This preserves audit history and data provenance.
- What happens when two API clients simultaneously update different fields of the same application? The system performs field-level merge — non-overlapping fields are merged automatically; only conflicting fields (same field updated by both) are rejected with a conflict error.
- What happens when a user authenticates via SSO but has no pre-assigned role? They are shown an "Access Denied" message and cannot access the system. An administrator must pre-populate their account with a role before they can gain access.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST authenticate users via corporate SSO as the sole authentication mechanism for the web UI.
- **FR-002**: System MUST enforce role-based access control with three roles: administrator (full access including user management), maintainer (can create and update application records via API/MCP), and reader (view-only access).
- **FR-003**: System MUST deny access to users who authenticate via SSO but have no pre-assigned role, displaying an "Access Denied" message. Users must be explicitly provisioned by an administrator before gaining access.
- **FR-004**: System MUST provide a read-only user interface that displays the consolidated application inventory with search, filter, and sort capabilities.
- **FR-005**: System MUST provide an API that allows authenticated clients with maintainer or administrator roles to create and update application records.
- **FR-006**: System MUST provide an MCP server that exposes tools for querying and updating the application inventory.
- **FR-007**: System MUST store the following fields for each application: acronym, name, ministry, branch, technologies (free-text tags with auto-suggest from existing values), hosting details, status (active/dormant/retired/unknown), data sources, last activity date, URLs (per-environment app URLs, source code repository, CI/CD link), contacts (business owner, technical lead), critical system flag, notes, and known risks and vulnerabilities.
- **FR-008**: System MUST validate that required fields are provided when creating or updating application records and return clear error messages for invalid submissions.
- **FR-009**: System MUST track the data source origin for each application record and field update.
- **FR-010**: System MUST update the "last activity" timestamp whenever an application record is modified.
- **FR-011**: System MUST allow administrators to assign and change user roles through the user interface.
- **FR-012**: System MUST support filtering applications by status, ministry, technology, and critical system flag.
- **FR-013**: System MUST support searching applications by name, acronym, and ministry.
- **FR-014**: System MUST display known risks and vulnerabilities for each application in the detail view.
- **FR-015**: System MUST handle concurrent API updates to the same application record using field-level merge: non-overlapping field updates are merged automatically, and only updates to the same field by concurrent requests result in a conflict rejection.
- **FR-016**: System MUST support the API performing upsert operations (create if not exists, update if exists) based on application acronym.
- **FR-017**: System MUST allow multiple URLs per application, organized by environment (e.g., development, staging, production).
- **FR-018**: System MUST automatically assign the administrator role to the first authenticated user when no user accounts exist in the system (bootstrapping).
- **FR-019**: System MUST allow administrators to pre-populate user accounts by specifying an email address and role, via both the web UI (user management screen) and the API (for bulk/automated provisioning).
- **FR-020**: System MUST prevent the demotion or removal of the last administrator account to avoid an unrecoverable state where no one can manage users.
- **FR-021**: System MUST enforce soft delete for applications — applications can only be marked as "retired" and cannot be permanently removed through the application, preserving audit history.
- **FR-022**: System MUST provide auto-suggest functionality for the technologies field, suggesting existing tag values as users or API clients enter new values.
- **FR-023**: System MUST authenticate MCP server connections using the same token-based authentication mechanism as the REST API (shared identity provider).

### Key Entities

- **Application**: The central entity representing a tracked software application. Contains all inventory fields (acronym, name, ministry, branch, technologies, hosting, status, URLs, contacts, critical flag, notes, risks). Uniquely identified by acronym. Cannot be permanently deleted; can only be marked "retired" (soft delete).
- **Data Source**: Represents an origin system or feed that provides application inventory data. Tracks which source provided each piece of information and when.
- **User**: A person who has authenticated via corporate SSO. Uniquely identified by email address. Has an assigned role (administrator, maintainer, or reader) that determines their permissions within the system. Users must be pre-populated by an administrator before they can access the system (except for the bootstrapping case where the first user becomes admin).
- **Contact**: A person associated with an application in a specific capacity (business owner or technical lead). Contains name and contact information.
- **Risk/Vulnerability**: A known risk or vulnerability associated with an application. Contains description and any relevant notes about mitigation or impact.
- **Environment URL**: A URL associated with a specific deployment environment (development, staging, production) for an application.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can find any application in the inventory within 30 seconds using search or filters.
- **SC-002**: The system supports at least 5,000 application records without noticeable performance degradation for end users.
- **SC-003**: API-driven inventory updates are reflected in the read-only UI within 5 seconds of submission.
- **SC-004**: 95% of authenticated users can successfully navigate to an application's detail page on their first attempt without assistance.
- **SC-005**: The system can ingest updates from at least 10 distinct data sources simultaneously without errors.
- **SC-006**: Pre-provisioned employees can access the inventory and find relevant applications within 2 minutes of their first login (no training required beyond SSO authentication). Administrators can pre-populate a new user account in under 1 minute.
- **SC-007**: Administrators can assign or change a user role in under 1 minute.
- **SC-008**: The system maintains 99.5% availability during business hours.

## Assumptions

- Users have existing corporate SSO credentials and the SSO provider supports standard federation protocols (e.g., SAML or OIDC).
- The application inventory will initially contain fewer than 5,000 applications, with moderate growth over time.
- "Read-only UI" means the initial web interface does not provide forms for creating or editing applications; all write operations happen through the API or MCP server. The UI does include user management screens for administrators.
- The API uses token-based authentication tied to the same identity provider as the SSO, allowing automated systems to authenticate programmatically.
- The MCP server uses the same token-based authentication as the REST API (shared identity provider).
- The MCP server follows the Model Context Protocol specification and is consumed by AI assistants or automation tools, not directly by end users.
- Application acronyms are unique identifiers within the system and are used for matching records across data sources.
- When conflicting data arrives from multiple sources, the most recent update takes precedence unless an administrator manually overrides it (last-write-wins by default).
- The "critical system" flag is a boolean indicator and its definition aligns with the organization's existing criticality classification.
- Mobile-responsive design is expected but a dedicated mobile application is out of scope for the initial version.
- Data retention follows the organization's standard records management policies.
- The initial deployment target is an internal corporate network accessible to all employees.
- Users must be explicitly pre-provisioned by an administrator (with email and role) before they can access the system. There is no default role assignment for unknown users.
- The first authenticated user on a fresh deployment is automatically granted administrator privileges (bootstrapping), after which all other users must be pre-provisioned.

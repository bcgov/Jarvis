# Tasks: Application Inventory Aggregator

**Input**: Design documents from `/specs/001-app-inventory-aggregator/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/api-rest.md, contracts/api-mcp.md, quickstart.md

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **API + MCP**: `src/Jarvis.Api/`
- **Blazor WASM**: `src/Jarvis.Web/`
- **Shared models/DTOs**: `src/Jarvis.Shared/`
- **Tests**: `tests/Jarvis.Api.Tests/`, `tests/Jarvis.Web.Tests/`, `tests/Jarvis.Mcp.Tests/`
- **Deployment**: `deploy/`

---

## Phase 1: Setup (Project Scaffolding)

**Purpose**: Create the .NET solution, projects, NuGet dependencies, and base configuration files.

- [ ] T001 Create solution file at src/Jarvis.sln with projects Jarvis.Api, Jarvis.Web, Jarvis.Shared and test projects Jarvis.Api.Tests, Jarvis.Web.Tests, Jarvis.Mcp.Tests
- [ ] T002 Configure src/Jarvis.Shared/Jarvis.Shared.csproj with .NET 10 target framework (class library)
- [ ] T003 [P] Configure src/Jarvis.Api/Jarvis.Api.csproj with .NET 10, NuGet packages: Microsoft.EntityFrameworkCore.SqlServer, Microsoft.AspNetCore.Authentication.JwtBearer, Microsoft.AspNetCore.Authentication.OpenIdConnect, FluentValidation.AspNetCore, Serilog.AspNetCore, Serilog.Sinks.File, Serilog.Sinks.EventLog, ModelContextProtocol, Microsoft.AspNetCore.Diagnostics.HealthChecks, project reference to Jarvis.Shared
- [ ] T004 [P] Configure src/Jarvis.Web/Jarvis.Web.csproj with .NET 10 Blazor WebAssembly standalone, NuGet packages: Microsoft.AspNetCore.Components.WebAssembly, Microsoft.AspNetCore.Components.WebAssembly.Authentication, Microsoft.Extensions.Http, project reference to Jarvis.Shared
- [ ] T005 [P] Configure tests/Jarvis.Api.Tests/Jarvis.Api.Tests.csproj with xUnit, Microsoft.AspNetCore.Mvc.Testing, Microsoft.EntityFrameworkCore.InMemory, FluentAssertions, project references to Jarvis.Api and Jarvis.Shared
- [ ] T006 [P] Configure tests/Jarvis.Web.Tests/Jarvis.Web.Tests.csproj with xUnit, bUnit, FluentAssertions, project reference to Jarvis.Web
- [ ] T007 [P] Configure tests/Jarvis.Mcp.Tests/Jarvis.Mcp.Tests.csproj with xUnit, FluentAssertions, project references to Jarvis.Api and Jarvis.Shared
- [ ] T008 [P] Create .gitignore entries for appsettings.Development.json, bin/, obj/, .vs/, *.user
- [ ] T009 [P] Create src/Jarvis.Api/appsettings.json with production-safe defaults (no secrets) and src/Jarvis.Api/appsettings.Development.template.json with LocalDB connection string and local KeyCloak settings
- [ ] T010 [P] Create deploy/docker-compose.yml with KeyCloak service for local development (pre-configured jarvis realm)
- [ ] T011 [P] Create deploy/install.ps1 PowerShell deployment script stub for IIS site setup on Windows Server 2025

**Checkpoint**: Solution builds with `dotnet build src/Jarvis.sln`. No runtime functionality yet.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented. Includes domain entities, database schema, authentication, error handling, and logging.

**CRITICAL**: No user story work can begin until this phase is complete.

### Domain Entities (Shared)

- [ ] T012 [P] Create enumerations in src/Jarvis.Shared/Models/Enums.cs: ApplicationStatus (active, dormant, retired, unknown, maintenance), UserRole (administrator, maintainer, reader), Confidence (low, medium, high), ConfidenceTier (red, yellow, green), ScmType (svn, bitbucket, github), RiskSeverity (low, medium, high, critical)
- [ ] T013 [P] Create Application entity in src/Jarvis.Shared/Models/Application.cs with all fields per data-model.md (Id, Acronym, FullName, Ministry, Branch, Section, JiraCategory, Status, StatusSource, StatusConfidence, IsCritical, PrimaryUrl, UrlSource, DataQuality, ConfidenceTier, LastDeployment, LastCommit, RecentJiraActivity90d, RecentJiraActivity365d, CollectedAt, LastActivity, CreatedAt, UpdatedAt, RowVersion) plus navigation properties
- [ ] T014 [P] Create ApplicationAlias entity in src/Jarvis.Shared/Models/ApplicationAlias.cs
- [ ] T015 [P] Create Contact entity in src/Jarvis.Shared/Models/Contact.cs
- [ ] T016 [P] Create TechStack entity in src/Jarvis.Shared/Models/TechStack.cs and TechStackFramework in src/Jarvis.Shared/Models/TechStackFramework.cs
- [ ] T017 [P] Create CodeLocation entity in src/Jarvis.Shared/Models/CodeLocation.cs and Repository in src/Jarvis.Shared/Models/Repository.cs
- [ ] T018 [P] Create HostingEnvironment entity in src/Jarvis.Shared/Models/HostingEnvironment.cs and HostingServer in src/Jarvis.Shared/Models/HostingServer.cs
- [ ] T019 [P] Create EnvironmentUrl entity in src/Jarvis.Shared/Models/EnvironmentUrl.cs
- [ ] T020 [P] Create ApplicationLink entity in src/Jarvis.Shared/Models/ApplicationLink.cs
- [ ] T021 [P] Create ApplicationNote entity in src/Jarvis.Shared/Models/ApplicationNote.cs
- [ ] T021a [P] Create ApplicationRisk entity in src/Jarvis.Shared/Models/ApplicationRisk.cs (Id, ApplicationId, Title, Description, Severity, Status, CreatedAt, Source)
- [ ] T021b [P] Create DataSource entity in src/Jarvis.Shared/Models/DataSource.cs (Id, Name, Description, LastSyncAt, IsActive)
- [ ] T022 [P] Create Provenance entity in src/Jarvis.Shared/Models/Provenance.cs
- [ ] T023 [P] Create FieldProvenance entity in src/Jarvis.Shared/Models/FieldProvenance.cs
- [ ] T024 [P] Create User entity in src/Jarvis.Shared/Models/User.cs
- [ ] T025 [P] Create PersonalAccessToken entity in src/Jarvis.Shared/Models/PersonalAccessToken.cs

### DTOs (Shared)

- [ ] T026 [P] Create application list DTO in src/Jarvis.Shared/Dtos/ApplicationListItemDto.cs (acronym, fullName, ministry, branch, status, isCritical, updatedAt)
- [ ] T027 [P] Create application detail DTO in src/Jarvis.Shared/Dtos/ApplicationDetailDto.cs (full response structure per api-rest.md GET /applications/{acronym}, including risks array and lastActivity)
- [ ] T028 [P] Create application upsert request DTO in src/Jarvis.Shared/Dtos/ApplicationUpsertRequest.cs and patch request DTO in src/Jarvis.Shared/Dtos/ApplicationPatchRequest.cs (notes and risks arrays use append semantics on PATCH)
- [ ] T029 [P] Create bulk import DTOs in src/Jarvis.Shared/Dtos/BulkImportRequest.cs and src/Jarvis.Shared/Dtos/BulkImportResult.cs
- [ ] T030 [P] Create user DTOs in src/Jarvis.Shared/Dtos/UserDto.cs, src/Jarvis.Shared/Dtos/CreateUserRequest.cs, src/Jarvis.Shared/Dtos/BulkUserRequest.cs, src/Jarvis.Shared/Dtos/ChangeRoleRequest.cs
- [ ] T031 [P] Create PAT DTOs in src/Jarvis.Shared/Dtos/TokenDto.cs, src/Jarvis.Shared/Dtos/CreateTokenRequest.cs, src/Jarvis.Shared/Dtos/CreateTokenResponse.cs
- [ ] T032 [P] Create pagination DTOs in src/Jarvis.Shared/Dtos/PagedResult.cs (generic) and src/Jarvis.Shared/Dtos/PaginationParams.cs

### Database Infrastructure

- [ ] T033 Create JarvisDbContext in src/Jarvis.Api/Infrastructure/JarvisDbContext.cs with DbSet properties for all entities (including ApplicationRisk, DataSource), configure entity relationships, indexes (per data-model.md Indexes section), and RowVersion concurrency token
- [ ] T034 Create entity configurations in src/Jarvis.Api/Infrastructure/Configurations/ (ApplicationConfiguration.cs, UserConfiguration.cs, PersonalAccessTokenConfiguration.cs, ApplicationRiskConfiguration.cs, DataSourceConfiguration.cs) with Fluent API for constraints, indexes, and relationships
- [ ] T035 Create initial EF Core migration using dotnet-ef tools (creates src/Jarvis.Api/Infrastructure/Migrations/ folder)

### Authentication and Authorization

- [ ] T036 Implement dual-mode authentication middleware in src/Jarvis.Api/Authentication/DualAuthHandler.cs that detects jrv_ prefix for PAT lookup or delegates to KeyCloak JWT validation
- [ ] T037 Implement PAT authentication logic in src/Jarvis.Api/Authentication/PatAuthenticationService.cs (SHA-256 hash lookup, expiry check, revocation check, active user verification, role resolution)
- [ ] T038 Implement role resolution service in src/Jarvis.Api/Authentication/RoleResolutionService.cs (resolve role from User table by email for JWT, by token ownership for PAT)
- [ ] T039 Configure authentication and authorization in src/Jarvis.Api/Program.cs (AddAuthentication with JWT Bearer + PAT scheme, AddAuthorization with role-based policies for administrator, maintainer, reader)

### Error Handling and Logging

- [ ] T040 [P] Implement global exception handling middleware in src/Jarvis.Api/Middleware/ExceptionHandlingMiddleware.cs returning RFC 7807 ProblemDetails format
- [ ] T041 [P] Configure Serilog in src/Jarvis.Api/Program.cs with File sink (rolling daily, JSON structured) and EventLog sink, PII masking for email fields

### Program.cs Bootstrap

- [ ] T042 Wire up src/Jarvis.Api/Program.cs with all services: DbContext (SQL Server), authentication, authorization, Serilog, CORS (for Blazor WASM), health checks, static file serving (Blazor WASM), Swagger/OpenAPI

**Checkpoint**: Foundation ready. `dotnet ef database update` creates the schema. Authentication middleware rejects unauthenticated requests. Health endpoints respond. User story implementation can begin.

---

## Phase 3: User Story 1 - View Application Inventory (Priority: P1) - MVP

**Goal**: Authenticated readers can browse, search, filter, and sort the application inventory through a Blazor WASM UI. They can view full details for any application.

**Independent Test**: Log in via SSO, browse the inventory list, search by acronym/name, filter by status/ministry, view application detail page with all tracked fields.

### Tests for User Story 1

- [ ] T043 [P] [US1] Integration test for GET /applications endpoint (search, filter, pagination) in tests/Jarvis.Api.Tests/Integration/ApplicationsListTests.cs
- [ ] T044 [P] [US1] Integration test for GET /applications/{acronym} endpoint in tests/Jarvis.Api.Tests/Integration/ApplicationsDetailTests.cs
- [ ] T045 [P] [US1] Integration test for GET /technologies/suggest endpoint in tests/Jarvis.Api.Tests/Integration/TechnologiesSuggestTests.cs
- [ ] T046 [P] [US1] Unit test for ApplicationReadService in tests/Jarvis.Api.Tests/Services/ApplicationReadServiceTests.cs
- [ ] T047 [P] [US1] bUnit component test for application list page in tests/Jarvis.Web.Tests/Components/ApplicationListPageTests.cs
- [ ] T048 [P] [US1] bUnit component test for application detail page in tests/Jarvis.Web.Tests/Components/ApplicationDetailPageTests.cs

### Implementation for User Story 1

- [ ] T049 [US1] Implement ApplicationReadService in src/Jarvis.Api/Services/ApplicationReadService.cs (list with search/filter/sort/pagination, get detail by acronym, map entities to DTOs)
- [ ] T050 [US1] Implement TechnologyService in src/Jarvis.Api/Services/TechnologyService.cs (suggest endpoint - query distinct TechStackFramework names with partial match)
- [ ] T051 [US1] Implement ApplicationsController GET endpoints in src/Jarvis.Api/Controllers/ApplicationsController.cs (GET /applications with query params, GET /applications/{acronym}, GET /applications/{acronym}/provenance)
- [ ] T052 [US1] Implement TechnologiesController in src/Jarvis.Api/Controllers/TechnologiesController.cs (GET /technologies/suggest)
- [ ] T053 [US1] Configure Blazor WASM OIDC authentication in src/Jarvis.Web/Auth/OidcAuthStateProvider.cs and src/Jarvis.Web/Program.cs (KeyCloak authorization code flow with PKCE)
- [ ] T054 [US1] Create ApplicationApiClient in src/Jarvis.Web/Services/ApplicationApiClient.cs (typed HttpClient for calling API endpoints with Bearer token)
- [ ] T055 [P] [US1] Create main layout with BC Design System tokens in src/Jarvis.Web/wwwroot/css/app.css (BC Sans font, color tokens, spacing) and src/Jarvis.Web/Shared/MainLayout.razor
- [ ] T056 [US1] Create application list page in src/Jarvis.Web/Pages/Applications/Index.razor with search bar, status/ministry/critical filters, sortable columns, pagination
- [ ] T057 [US1] Create application detail page in src/Jarvis.Web/Pages/Applications/Detail.razor showing all tracked fields (contacts, tech stack, hosting, URLs, links, notes, risks, provenance, lastActivity)
- [ ] T058 [P] [US1] Create reusable filter components in src/Jarvis.Web/Components/SearchBar.razor, src/Jarvis.Web/Components/StatusFilter.razor, src/Jarvis.Web/Components/MinistryFilter.razor
- [ ] T059 [US1] Create navigation and access denied page in src/Jarvis.Web/Pages/AccessDenied.razor and src/Jarvis.Web/Shared/NavMenu.razor

**Checkpoint**: User Story 1 complete. A reader can log in, see the inventory list, search/filter, and view application details. All read-only UI is functional.

---

## Phase 4: User Story 2 - Update Inventory via API (Priority: P1)

**Goal**: Maintainers and administrators can create, update (full and partial), and bulk-import application records through the REST API. Validation rejects malformed requests with clear errors. Concurrent updates use field-level merge.

**Independent Test**: Make API calls with a maintainer token to create new applications (PUT), partially update fields (PATCH), bulk import, and verify validation errors for invalid data.

### Tests for User Story 2

- [ ] T060 [P] [US2] Integration test for PUT /applications/{acronym} (create + update) in tests/Jarvis.Api.Tests/Integration/ApplicationsUpsertTests.cs
- [ ] T061 [P] [US2] Integration test for PATCH /applications/{acronym} (partial update, field-level merge, concurrency conflict) in tests/Jarvis.Api.Tests/Integration/ApplicationsPatchTests.cs
- [ ] T062 [P] [US2] Integration test for POST /applications/bulk in tests/Jarvis.Api.Tests/Integration/ApplicationsBulkTests.cs
- [ ] T063 [P] [US2] Unit test for ApplicationWriteService in tests/Jarvis.Api.Tests/Services/ApplicationWriteServiceTests.cs
- [ ] T064 [P] [US2] Unit test for FluentValidation validators in tests/Jarvis.Api.Tests/Validation/ApplicationValidatorTests.cs
- [ ] T065 [P] [US2] Integration test for DELETE /applications/{acronym} returns 405 in tests/Jarvis.Api.Tests/Integration/ApplicationsDeleteTests.cs

### Implementation for User Story 2

- [ ] T066 [US2] Implement FluentValidation validators in src/Jarvis.Api/Validation/ApplicationUpsertValidator.cs and src/Jarvis.Api/Validation/ApplicationPatchValidator.cs (required fields, acronym format, enum values, string lengths per data-model.md)
- [ ] T067 [US2] Implement ApplicationWriteService in src/Jarvis.Api/Services/ApplicationWriteService.cs (upsert logic: create if not exists/replace if exists, partial update with field-level merge, soft delete via Status=retired, update LastActivity on every modification, append semantics for notes and risks arrays)
- [ ] T068 [US2] Implement concurrency handling in ApplicationWriteService: on DbUpdateConcurrencyException, re-read the current row, compare only the fields this request modifies against what changed in the database — if no field overlap, merge and retry; if same field was changed by both, return 409 Conflict listing the conflicting fields. Use RowVersion as the row-level trigger for the retry loop.
- [ ] T069 [US2] Implement bulk import logic in src/Jarvis.Api/Services/BulkImportService.cs (iterate applications, upsert each, collect errors, return summary)
- [ ] T070 [US2] Add write endpoints to src/Jarvis.Api/Controllers/ApplicationsController.cs: PUT /applications/{acronym}, PATCH /applications/{acronym}, DELETE /applications/{acronym} (405), POST /applications/bulk
- [ ] T071 [US2] Wire FluentValidation into the request pipeline in src/Jarvis.Api/Program.cs and configure automatic 400 responses with ProblemDetails for validation failures

**Checkpoint**: User Story 2 complete. API clients can create, update, and bulk-import applications. Validation rejects bad data. Concurrent updates merge cleanly or return 409.

---

## Phase 5: User Story 3 - Update Inventory via MCP Server (Priority: P2)

**Goal**: AI assistants and automation tools connect to the MCP server (Streamable HTTP at /mcp) to search, read, and update application records using the Model Context Protocol.

**Independent Test**: Connect an MCP client, list available tools, invoke search_applications, get_application, update_application, list_ministries, list_technologies, and verify results.

### Tests for User Story 3

- [ ] T072 [P] [US3] Unit test for MCP tool: search_applications in tests/Jarvis.Mcp.Tests/Tools/SearchApplicationsToolTests.cs
- [ ] T073 [P] [US3] Unit test for MCP tool: get_application in tests/Jarvis.Mcp.Tests/Tools/GetApplicationToolTests.cs
- [ ] T074 [P] [US3] Unit test for MCP tool: update_application in tests/Jarvis.Mcp.Tests/Tools/UpdateApplicationToolTests.cs
- [ ] T075 [P] [US3] Unit test for MCP tools: list_ministries and list_technologies in tests/Jarvis.Mcp.Tests/Tools/ListToolsTests.cs
- [ ] T076 [P] [US3] Unit test for MCP tool: get_application_provenance in tests/Jarvis.Mcp.Tests/Tools/GetProvenanceToolTests.cs

### Implementation for User Story 3

- [ ] T077 [US3] Configure MCP Streamable HTTP middleware in src/Jarvis.Api/Program.cs (map /mcp endpoint, register tools, configure server info: name=jarvis-inventory, version=1.0.0, protocolVersion=2025-03-26)
- [ ] T078 [P] [US3] Implement search_applications MCP tool in src/Jarvis.Api/Mcp/SearchApplicationsTool.cs (delegates to ApplicationReadService, formats results as text)
- [ ] T079 [P] [US3] Implement get_application MCP tool in src/Jarvis.Api/Mcp/GetApplicationTool.cs (delegates to ApplicationReadService, formats full detail as text)
- [ ] T080 [P] [US3] Implement update_application MCP tool in src/Jarvis.Api/Mcp/UpdateApplicationTool.cs (delegates to ApplicationWriteService, returns change summary as text)
- [ ] T081 [P] [US3] Implement list_ministries MCP tool in src/Jarvis.Api/Mcp/ListMinistriesTool.cs (distinct ministry query with application counts)
- [ ] T082 [P] [US3] Implement list_technologies MCP tool in src/Jarvis.Api/Mcp/ListTechnologiesTool.cs (distinct framework names with usage counts, optional prefix filter)
- [ ] T083 [US3] Implement get_application_provenance MCP tool in src/Jarvis.Api/Mcp/GetProvenanceTool.cs (delegates to ProvenanceService, formats history as text)
- [ ] T084 [US3] Implement MCP resources in src/Jarvis.Api/Mcp/InventoryResources.cs (inventory://summary with status counts, inventory://recent-changes with 7-day modifications)
- [ ] T085 [US3] Implement MCP authentication integration: ensure /mcp endpoint validates Bearer tokens via the same dual-mode auth handler and resolves roles for authorization checks within tools

**Checkpoint**: User Story 3 complete. MCP clients can connect, discover tools, query the inventory, and push updates. Same auth and data layer as the REST API.

---

## Phase 6: User Story 4 - Administer User Roles and Accounts (Priority: P2)

**Goal**: Administrators manage user accounts and roles. Users are pre-provisioned with email + role before first SSO login. Bootstrap logic assigns admin to the first user on fresh deployment. Self-service PAT management for all users.

**Independent Test**: Log in as admin, provision users via UI and API, change roles, verify access denied for unprovisioned users, create/revoke PATs, verify last-admin protection.

### Tests for User Story 4

- [ ] T086 [P] [US4] Integration test for POST /users and POST /users/bulk in tests/Jarvis.Api.Tests/Integration/UsersProvisionTests.cs
- [ ] T087 [P] [US4] Integration test for PUT /users/{email}/role and DELETE /users/{email} in tests/Jarvis.Api.Tests/Integration/UsersRoleTests.cs
- [ ] T088 [P] [US4] Integration test for last-admin protection (cannot demote/deactivate last admin) in tests/Jarvis.Api.Tests/Integration/LastAdminProtectionTests.cs
- [ ] T089 [P] [US4] Integration test for bootstrap logic (first user becomes admin) in tests/Jarvis.Api.Tests/Integration/BootstrapTests.cs
- [ ] T090 [P] [US4] Integration test for PAT CRUD (create, list, revoke, admin revoke) in tests/Jarvis.Api.Tests/Integration/PersonalAccessTokenTests.cs
- [ ] T091 [P] [US4] Unit test for UserService in tests/Jarvis.Api.Tests/Services/UserServiceTests.cs
- [ ] T092 [P] [US4] Unit test for PersonalAccessTokenService in tests/Jarvis.Api.Tests/Services/PersonalAccessTokenServiceTests.cs
- [ ] T093 [P] [US4] bUnit tests for user management page in tests/Jarvis.Web.Tests/Components/UserManagementPageTests.cs

### Implementation for User Story 4

- [ ] T094 [US4] Implement UserService in src/Jarvis.Api/Services/UserService.cs (create user, bulk create, change role, deactivate, list users, get by email, last-admin check)
- [ ] T095 [US4] Implement bootstrap logic in src/Jarvis.Api/Authentication/BootstrapService.cs (on authenticated request: if zero users exist, create current user as administrator)
- [ ] T096 [US4] Implement PersonalAccessTokenService in src/Jarvis.Api/Services/PersonalAccessTokenService.cs (generate jrv_ token, hash with SHA-256, store hash only, enforce 10-token limit, enforce 90-day max, revoke, list)
- [ ] T097 [US4] Implement UsersController in src/Jarvis.Api/Controllers/UsersController.cs (GET /users, POST /users, POST /users/bulk, PUT /users/{email}/role, DELETE /users/{email})
- [ ] T098 [US4] Implement TokensController in src/Jarvis.Api/Controllers/TokensController.cs (GET /users/me/tokens, POST /users/me/tokens, DELETE /users/me/tokens/{id}, GET /users/{email}/tokens, DELETE /users/{email}/tokens/{id}, DELETE /users/{email}/tokens)
- [ ] T099 [US4] Create user management page in src/Jarvis.Web/Pages/Admin/Users.razor (list users, search, add user with email+role, change role, deactivate)
- [ ] T100 [US4] Create PAT management page in src/Jarvis.Web/Pages/Profile/Tokens.razor (list own tokens, create new token with name+expiry, copy token on creation, revoke)
- [ ] T101 [US4] Implement UserApiClient in src/Jarvis.Web/Services/UserApiClient.cs (typed HttpClient for user management endpoints)
- [ ] T102 [US4] Add role-based navigation visibility in src/Jarvis.Web/Shared/NavMenu.razor (Admin menu items visible only to administrators)

**Checkpoint**: User Story 4 complete. Administrators can manage users and roles. All users can manage their own PATs. Bootstrap works on fresh deployment. Last-admin protection enforced.

---

## Phase 7: User Story 5 - Aggregate from Multiple Data Sources (Priority: P3)

**Goal**: The system tracks data provenance at the field level. When data arrives from multiple sources, the system records which source provided each field and when. Users can see source attribution in the application detail view.

**Independent Test**: Submit data for the same application from two different sources via API, view the application detail to confirm source attribution is displayed for each field.

### Tests for User Story 5

- [ ] T103 [P] [US5] Integration test for field provenance recording on PATCH in tests/Jarvis.Api.Tests/Integration/ProvenanceTrackingTests.cs
- [ ] T104 [P] [US5] Integration test for GET /applications/{acronym}/provenance in tests/Jarvis.Api.Tests/Integration/ProvenanceQueryTests.cs
- [ ] T105 [P] [US5] Unit test for ProvenanceService in tests/Jarvis.Api.Tests/Services/ProvenanceServiceTests.cs
- [ ] T106 [P] [US5] bUnit test for provenance display component in tests/Jarvis.Web.Tests/Components/ProvenanceDisplayTests.cs

### Implementation for User Story 5

- [ ] T107 [US5] Implement ProvenanceService in src/Jarvis.Api/Services/ProvenanceService.cs (record field changes with source, previousValue, newValue, timestamp; query provenance history with filters)
- [ ] T108 [US5] Integrate provenance recording into ApplicationWriteService: on every field update via PUT, PATCH, or bulk, call ProvenanceService to record the source and change for each modified field
- [ ] T109 [US5] Integrate provenance recording into Provenance entity updates: when data arrives, update the Application's Provenance flags (inCmdb, inJira, etc.) based on the source parameter
- [ ] T110 [US5] Implement provenance query endpoint in ApplicationsController: GET /applications/{acronym}/provenance (paginated, optional fieldName filter)
- [ ] T111 [US5] Create provenance display component in src/Jarvis.Web/Components/ProvenancePanel.razor (shows field history with source, date, old/new values per field)
- [ ] T112 [US5] Integrate ProvenancePanel into the application detail page src/Jarvis.Web/Pages/Applications/Detail.razor (collapsible section showing field provenance)

**Checkpoint**: User Story 5 complete. Field-level provenance is tracked on every update. Readers can see which data source provided each piece of information.

---

## Phase 8: Polish and Cross-Cutting Concerns

**Purpose**: Deployment, CI/CD, performance, security hardening, and final validation.

- [ ] T113 [P] Create Azure DevOps pipeline in azure-pipelines.yml (build, test, publish self-contained win-x64, artifact upload)
- [ ] T114 [P] Finalize deploy/install.ps1 with IIS site creation, app pool configuration, service installation, and database migration execution
- [ ] T115 [P] Implement health check endpoints in src/Jarvis.Api/Controllers/HealthController.cs: GET /health (liveness), GET /health/ready (SQL Server connectivity + schema validation)
- [ ] T116 [P] Add Swagger/OpenAPI documentation configuration in src/Jarvis.Api/Program.cs with endpoint descriptions and example responses
- [ ] T117 [P] Add response caching for GET /applications list and GET /technologies/suggest in src/Jarvis.Api/Program.cs (short cache, invalidated on writes)
- [ ] T118 [P] Add database query performance indexes verification: ensure all indexes from data-model.md are present in the migration
- [ ] T119 Configure Blazor WASM publishing to include compiled output in API wwwroot (MapFallbackToFile in src/Jarvis.Api/Program.cs for SPA routing)
- [ ] T120 Add rate limiting middleware for API endpoints in src/Jarvis.Api/Program.cs (protect against abuse)
- [ ] T121 Security hardening: ensure HTTPS enforcement, HSTS headers, anti-forgery for state-changing operations, CORS restricted to same origin
- [ ] T122 Run quickstart.md validation: verify clone, restore, migrate, run flow works end-to-end on a clean machine
- [ ] T123 Create sample-data.json at repository root with representative test applications for the bulk import endpoint

**Checkpoint**: Application is production-ready. CI/CD pipeline builds and tests. Deployment script installs on Windows Server 2025. All endpoints secured and performant.

---

## Dependencies and Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies - start immediately
- **Phase 2 (Foundational)**: Depends on Phase 1 completion - BLOCKS all user stories
- **Phase 3 (US1)**: Depends on Phase 2 - read-only UI and API
- **Phase 4 (US2)**: Depends on Phase 2 - write API (can run in parallel with US1)
- **Phase 5 (US3)**: Depends on Phase 2 + shares services with US2 (run after or parallel with US2)
- **Phase 6 (US4)**: Depends on Phase 2 - user management (can run in parallel with US1/US2)
- **Phase 7 (US5)**: Depends on Phase 4 (needs write service to integrate provenance tracking)
- **Phase 8 (Polish)**: Depends on all user stories being complete

### User Story Dependencies

- **US1 (P1)**: Depends only on Phase 2. Can start immediately after foundation.
- **US2 (P1)**: Depends only on Phase 2. Can start in parallel with US1.
- **US3 (P2)**: Depends on Phase 2. Reuses services from US1 and US2 but can be built independently using the same service interfaces.
- **US4 (P2)**: Depends only on Phase 2. Fully independent of US1/US2/US3.
- **US5 (P3)**: Depends on US2 (integrates into the write service). Must follow US2.

### Within Each User Story

- Tests written first (fail before implementation)
- Models/entities before services
- Services before controllers/endpoints
- API endpoints before UI pages
- Core logic before integration/polish

### Parallel Opportunities

**Phase 1**: T003-T011 all run in parallel (different files)

**Phase 2**: T012-T025 (entities), T026-T032 (DTOs) all run in parallel. T033-T035 (database) sequential. T036-T039 (auth) sequential. T040-T041 in parallel.

**After Phase 2**: US1, US2, US3, and US4 can all start in parallel (if team capacity allows). US5 must wait for US2.

**Within US1**: T043-T048 (tests) in parallel. T055/T058 in parallel with other US1 tasks. T049-T052 sequential (service before controller).

**Within US2**: T060-T065 (tests) in parallel. T066-T071 sequential (validators before service before controller).

**Within US3**: T072-T076 (tests) in parallel. T078-T082 (tools) in parallel.

**Within US4**: T086-T093 (tests) in parallel. T094-T096 sequential (services). T097-T098 in parallel (different controllers). T099-T102 in parallel (different UI pages).

---

## Parallel Example: User Stories 1 and 2 Simultaneously

```text
Developer A (US1 - Read):                    Developer B (US2 - Write):
  T049 ApplicationReadService                  T066 Validators
  T050 TechnologyService                       T067 ApplicationWriteService
  T051 ApplicationsController (GET)            T068 Concurrency handling
  T052 TechnologiesController                  T069 BulkImportService
  T053 Blazor OIDC auth                        T070 ApplicationsController (PUT/PATCH/POST)
  T054 ApplicationApiClient                    T071 Wire validation
  T055-T059 Blazor UI pages
```

Both developers work on different files and different controller methods. No conflicts.

---

## Implementation Strategy

### MVP First (User Stories 1 + 2 Only)

1. Complete Phase 1: Setup (solution structure)
2. Complete Phase 2: Foundational (entities, database, auth, error handling)
3. Complete Phase 3: User Story 1 (read-only UI)
4. Complete Phase 4: User Story 2 (write API)
5. **STOP and VALIDATE**: Search works, API creates/updates records, UI displays them
6. Deploy MVP to the Windows Server VM

### Incremental Delivery

1. Setup + Foundational = runnable skeleton
2. US1 = readers can browse inventory (requires seed data or US2)
3. US2 = maintainers can populate data via API (MVP complete with US1)
4. US3 = AI assistants can interact via MCP
5. US4 = administrators manage users and PATs (enables self-service)
6. US5 = full provenance tracking for data quality
7. Polish = production-hardened deployment

### Parallel Team Strategy

With 2-3 developers after Phase 2:
- Developer A: US1 (Blazor UI) + US4 (admin UI)
- Developer B: US2 (write API) + US3 (MCP tools)
- Developer C: US5 (provenance) after US2 complete, then Polish

---

## Notes

- [P] tasks = different files, no dependencies on incomplete tasks in same phase
- [Story] label maps task to specific user story for traceability
- Each user story is independently completable and testable after Phase 2
- RowVersion concurrency uses SQL Server native rowversion type (auto-incremented)
- PAT format: `jrv_` + 32 random bytes base62 (~43 chars). SHA-256 hash stored, plaintext shown once
- Soft delete only: Application.Status = retired (no separate IsRetired flag), DELETE returns 405
- All DateTime fields use DateTimeOffset for timezone safety
- BC Design System: use CSS custom properties + BC Sans font, not React components
- MCP transport: Streamable HTTP (POST /mcp), not stdio
- Commit after each task or logical group of parallel tasks

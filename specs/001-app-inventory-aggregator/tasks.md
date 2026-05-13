# Tasks: Application Inventory Aggregator

**Input**: Design documents from `/specs/001-app-inventory-aggregator/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/api-rest.md, contracts/api-mcp.md, quickstart.md

**Tests**: Not explicitly requested in the feature specification. Test tasks are omitted.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **API project**: `src/Jarvis.Api/`
- **Web project**: `src/Jarvis.Web/`
- **Shared project**: `src/Jarvis.Shared/`
- **Tests**: `tests/Jarvis.Api.Tests/`, `tests/Jarvis.Web.Tests/`, `tests/Jarvis.Mcp.Tests/`
- **Deployment**: `deploy/`
- **CI/CD**: `.github/workflows/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create the .NET solution structure, configure projects, and establish local development environment.

- [ ] T001 Create solution file and project structure: `src/Jarvis.sln`, `src/Jarvis.Api/Jarvis.Api.csproj`, `src/Jarvis.Web/Jarvis.Web.csproj`, `src/Jarvis.Shared/Jarvis.Shared.csproj`
- [ ] T002 Configure NuGet dependencies for Jarvis.Api in `src/Jarvis.Api/Jarvis.Api.csproj` (EF Core SQLite, FluentValidation, Serilog, ModelContextProtocol, KubernetesClient, Authentication.JwtBearer)
- [ ] T003 [P] Configure NuGet dependencies for Jarvis.Web in `src/Jarvis.Web/Jarvis.Web.csproj` (Blazor WebAssembly, Microsoft.AspNetCore.Components.WebAssembly.Authentication)
- [ ] T004 [P] Configure NuGet dependencies for Jarvis.Shared in `src/Jarvis.Shared/Jarvis.Shared.csproj`
- [ ] T005 [P] Create test projects: `tests/Jarvis.Api.Tests/Jarvis.Api.Tests.csproj`, `tests/Jarvis.Web.Tests/Jarvis.Web.Tests.csproj`, `tests/Jarvis.Mcp.Tests/Jarvis.Mcp.Tests.csproj` (xUnit, bUnit, Microsoft.AspNetCore.Mvc.Testing)
- [ ] T006 [P] Create `deploy/docker-compose.yml` with KeyCloak service for local development
- [ ] T007 [P] Create `src/Jarvis.Api/appsettings.Development.template.json` with local dev defaults (SQLite path, KeyCloak localhost, Leadership disabled)
- [ ] T008 [P] Update `.gitignore` to exclude `appsettings.Development.json`, `*.db`, `*.db-wal`, `*.db-shm`, and local dev artifacts
- [ ] T009 Create minimal `src/Jarvis.Api/Program.cs` with builder configuration placeholder and Blazor WASM static file hosting
- [ ] T010 [P] Create minimal `src/Jarvis.Web/Program.cs` with Blazor WASM bootstrap and HttpClient configuration
- [ ] T011 [P] Create `src/Jarvis.Web/wwwroot/index.html` with BC Sans font and BC Design Tokens CSS references

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented. Includes data model, authentication, error handling, and shared services.

**CRITICAL**: No user story work can begin until this phase is complete.

### Shared Models and Enumerations

- [ ] T012 [P] Create enumerations in `src/Jarvis.Shared/Models/Enums.cs` (ApplicationStatus, UserRole, Confidence, ConfidenceTier, ScmType)
- [ ] T013 [P] Create Application entity in `src/Jarvis.Shared/Models/Application.cs` with all fields per data-model.md
- [ ] T014 [P] Create ApplicationAlias entity in `src/Jarvis.Shared/Models/ApplicationAlias.cs`
- [ ] T015 [P] Create Contact entity in `src/Jarvis.Shared/Models/Contact.cs`
- [ ] T016 [P] Create TechStack and TechStackFramework entities in `src/Jarvis.Shared/Models/TechStack.cs`
- [ ] T017 [P] Create CodeLocation and Repository entities in `src/Jarvis.Shared/Models/CodeLocation.cs`
- [ ] T018 [P] Create HostingEnvironment and HostingServer entities in `src/Jarvis.Shared/Models/HostingEnvironment.cs`
- [ ] T019 [P] Create EnvironmentUrl entity in `src/Jarvis.Shared/Models/EnvironmentUrl.cs`
- [ ] T020 [P] Create ApplicationLink entity in `src/Jarvis.Shared/Models/ApplicationLink.cs`
- [ ] T021 [P] Create ApplicationNote entity in `src/Jarvis.Shared/Models/ApplicationNote.cs`
- [ ] T022 [P] Create Provenance entity in `src/Jarvis.Shared/Models/Provenance.cs`
- [ ] T023 [P] Create FieldProvenance entity in `src/Jarvis.Shared/Models/FieldProvenance.cs`
- [ ] T024 [P] Create User entity in `src/Jarvis.Shared/Models/User.cs`
- [ ] T025 [P] Create PersonalAccessToken entity in `src/Jarvis.Shared/Models/PersonalAccessToken.cs`

### DTOs

- [ ] T026 [P] Create Application DTOs in `src/Jarvis.Shared/Dtos/ApplicationDtos.cs` (ApplicationListItemDto, ApplicationDetailDto, ApplicationUpsertDto, ApplicationPatchDto, BulkImportRequestDto, BulkImportResultDto)
- [ ] T027 [P] Create User DTOs in `src/Jarvis.Shared/Dtos/UserDtos.cs` (UserListItemDto, CreateUserDto, BulkCreateUsersDto, ChangeRoleDto)
- [ ] T028 [P] Create Token DTOs in `src/Jarvis.Shared/Dtos/TokenDtos.cs` (TokenListItemDto, CreateTokenRequestDto, CreateTokenResponseDto)
- [ ] T029 [P] Create common DTOs in `src/Jarvis.Shared/Dtos/CommonDtos.cs` (PagedResultDto, ProblemDetailsDto, ProvenanceDto, TechnologySuggestDto)

### Database Infrastructure

- [ ] T030 Create EF Core DbContext in `src/Jarvis.Api/Infrastructure/AppDbContext.cs` with DbSets for all entities, relationship configuration, indexes, and concurrency token setup
- [ ] T031 Create initial EF Core migration in `src/Jarvis.Api/Infrastructure/Migrations/` (generate via `dotnet ef migrations add InitialCreate`)
- [ ] T032 Create SQLite RowVersion trigger SQL in `src/Jarvis.Api/Infrastructure/SqlScripts/row_version_trigger.sql` and apply in migration

### Authentication and Authorization

- [ ] T033 Implement dual-mode authentication middleware in `src/Jarvis.Api/Authentication/DualAuthenticationHandler.cs` (detect `jrv_` prefix for PAT, otherwise validate JWT from KeyCloak)
- [ ] T034 Implement PAT validation service in `src/Jarvis.Api/Authentication/PatAuthenticationService.cs` (SHA-256 hash lookup, expiry check, revocation check, user active check)
- [ ] T035 [P] Create role-based authorization policies in `src/Jarvis.Api/Authentication/AuthorizationPolicies.cs` (AdminOnly, MaintainerOrAdmin, AllAuthenticated)
- [ ] T036 [P] Implement user role resolution from database in `src/Jarvis.Api/Authentication/UserRoleResolver.cs` (email claim to role lookup)
- [ ] T037 Configure authentication and authorization services in `src/Jarvis.Api/Program.cs` (register KeyCloak JWT, PAT handler, policies)

### Error Handling and Logging

- [ ] T038 [P] Implement global exception handling middleware in `src/Jarvis.Api/Middleware/ExceptionHandlingMiddleware.cs` (RFC 7807 ProblemDetails responses)
- [ ] T039 [P] Configure Serilog structured logging in `src/Jarvis.Api/Infrastructure/LoggingConfiguration.cs` (JSON stdout, PII masking enricher)

### Health Endpoints

- [ ] T040 Implement health check endpoints in `src/Jarvis.Api/Controllers/HealthController.cs` (GET /health for liveness, GET /health/ready for readiness with SQLite and replication status)

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - View Application Inventory (Priority: P1) MVP

**Goal**: A reader logs in via corporate SSO, browses the inventory, searches/filters/sorts applications, and views detailed application information.

**Independent Test**: Log in, browse inventory list, search by name/acronym, filter by status/ministry, view application detail page with all fields.

### API Implementation for User Story 1

- [ ] T041 Implement ApplicationService in `src/Jarvis.Api/Services/ApplicationService.cs` (list with search/filter/sort/pagination, get by acronym with all related entities eager-loaded)
- [ ] T042 Implement ApplicationsController in `src/Jarvis.Api/Controllers/ApplicationsController.cs` (GET /api/v1/applications with query params, GET /api/v1/applications/{acronym})
- [ ] T043 [P] Implement TechnologyService in `src/Jarvis.Api/Services/TechnologyService.cs` (suggest endpoint logic - query distinct framework names matching prefix)
- [ ] T044 [P] Implement TechnologiesController in `src/Jarvis.Api/Controllers/TechnologiesController.cs` (GET /api/v1/technologies/suggest)

### Blazor WASM Frontend for User Story 1

- [ ] T045 [P] Create BC Design System tokens CSS in `src/Jarvis.Web/wwwroot/css/bc-design-tokens.css` (colors, spacing, typography, border-radius from BC Design System)
- [ ] T046 [P] Create base layout component in `src/Jarvis.Web/Components/Layout/MainLayout.razor` (BC Gov header, navigation, footer, BC Sans typography)
- [ ] T047 Configure OIDC authentication in `src/Jarvis.Web/Auth/AuthenticationStateConfiguration.cs` and `src/Jarvis.Web/Program.cs` (KeyCloak OIDC code flow with PKCE)
- [ ] T048 [P] Create API client service in `src/Jarvis.Web/Services/ApplicationApiService.cs` (typed HttpClient for application list and detail endpoints)
- [ ] T049 Create inventory list page in `src/Jarvis.Web/Pages/Inventory.razor` (paginated table, search bar, status/ministry/technology/critical filters, sort controls)
- [ ] T050 [P] Create search and filter component in `src/Jarvis.Web/Components/Inventory/SearchFilterPanel.razor` (text search input, filter dropdowns, clear filters button)
- [ ] T051 Create application detail page in `src/Jarvis.Web/Pages/ApplicationDetail.razor` (full detail view: contacts, tech stack, hosting, URLs, links, notes, risks, provenance summary)
- [ ] T052 [P] Create application list item component in `src/Jarvis.Web/Components/Inventory/ApplicationCard.razor` (summary row: acronym, name, ministry, status badge, critical flag indicator)
- [ ] T053 [P] Create pagination component in `src/Jarvis.Web/Components/Common/Pagination.razor` (page navigation, page size selector)
- [ ] T054 Create redirect-to-login and access-denied pages in `src/Jarvis.Web/Pages/Authentication/Login.razor` and `src/Jarvis.Web/Pages/Authentication/AccessDenied.razor`

**Checkpoint**: At this point, User Story 1 should be fully functional - users can log in and browse/search the inventory

---

## Phase 4: User Story 2 - Update Inventory via API (Priority: P1)

**Goal**: Maintainers and automated sources push application updates through the REST API with validation, upsert, field-level merge, and provenance tracking.

**Independent Test**: Make API calls to create a new application (PUT), update specific fields (PATCH), perform bulk import, and verify validation errors for malformed requests.

### Implementation for User Story 2

- [ ] T055 Implement FluentValidation validators in `src/Jarvis.Api/Validation/ApplicationUpsertValidator.cs` and `src/Jarvis.Api/Validation/ApplicationPatchValidator.cs` (required fields, acronym format, status enum, field lengths per data-model.md)
- [ ] T056 Implement ApplicationWriteService in `src/Jarvis.Api/Services/ApplicationWriteService.cs` (upsert logic with acronym matching, field-level merge for PATCH, RowVersion concurrency detection, provenance recording)
- [ ] T057 Implement PUT /api/v1/applications/{acronym} in `src/Jarvis.Api/Controllers/ApplicationsController.cs` (full upsert - create or replace)
- [ ] T058 Implement PATCH /api/v1/applications/{acronym} in `src/Jarvis.Api/Controllers/ApplicationsController.cs` (partial update with If-Match header support, field-level merge, 409 Conflict for same-field concurrent updates)
- [ ] T059 [P] Implement DELETE /api/v1/applications/{acronym} in `src/Jarvis.Api/Controllers/ApplicationsController.cs` (returns 405 Method Not Allowed with explanation)
- [ ] T060 Implement BulkImportService in `src/Jarvis.Api/Services/BulkImportService.cs` (batch upsert with per-record validation, source tracking, error collection)
- [ ] T061 Implement POST /api/v1/applications/bulk in `src/Jarvis.Api/Controllers/ApplicationsController.cs` (bulk upsert endpoint with processed/created/updated/errors response)
- [ ] T062 [P] Register FluentValidation in `src/Jarvis.Api/Program.cs` and configure validation filter for automatic 400 responses

**Checkpoint**: At this point, User Stories 1 AND 2 should both work - the system can be populated via API and browsed via UI

---

## Phase 5: User Story 3 - Update Inventory via MCP Server (Priority: P2)

**Goal**: AI assistants connect to the MCP server via Streamable HTTP to query and update the application inventory using defined tools.

**Independent Test**: Connect an MCP client, call search_applications, get_application, update_application, list_ministries, list_technologies, and verify changes are reflected.

### Implementation for User Story 3

- [ ] T063 Configure MCP server middleware in `src/Jarvis.Api/Program.cs` (register ModelContextProtocol services, map Streamable HTTP endpoint at /mcp, configure server info: name "jarvis-inventory", version "1.0.0", protocolVersion "2025-03-26")
- [ ] T064 Implement search_applications MCP tool in `src/Jarvis.Api/Mcp/SearchApplicationsTool.cs` (query, ministry, status, technology, isCritical, limit parameters; returns formatted text summaries)
- [ ] T065 [P] Implement get_application MCP tool in `src/Jarvis.Api/Mcp/GetApplicationTool.cs` (acronym parameter; returns full formatted application details)
- [ ] T066 Implement update_application MCP tool in `src/Jarvis.Api/Mcp/UpdateApplicationTool.cs` (acronym, source, fields parameters; performs field-level update via ApplicationWriteService; returns change summary)
- [ ] T067 [P] Implement list_ministries MCP tool in `src/Jarvis.Api/Mcp/ListMinistriesTool.cs` (returns distinct ministries with application counts)
- [ ] T068 [P] Implement list_technologies MCP tool in `src/Jarvis.Api/Mcp/ListTechnologiesTool.cs` (optional query prefix; returns technology tags with usage counts)
- [ ] T069 [P] Implement get_application_provenance MCP tool in `src/Jarvis.Api/Mcp/GetApplicationProvenanceTool.cs` (acronym, optional fieldName; returns formatted provenance history)
- [ ] T070 [P] Implement MCP resources in `src/Jarvis.Api/Mcp/InventoryResources.cs` (inventory://summary with status counts, inventory://recent-changes with 7-day modifications)
- [ ] T071 Implement MCP error handling in `src/Jarvis.Api/Mcp/McpErrorHandler.cs` (convert service exceptions to MCP isError: true responses with helpful messages)

**Checkpoint**: At this point, User Stories 1, 2, AND 3 should all work independently - AI clients can interact with the inventory

---

## Phase 6: User Story 4 - Administer User Roles and Accounts (Priority: P2)

**Goal**: Administrators manage user accounts and roles. Users without pre-assigned roles are denied access. First user bootstraps as admin. PATs enable self-service API/MCP access.

**Independent Test**: Log in as admin, pre-provision users, assign/change roles, verify access denied for unprovisioned users, create/revoke PATs, verify last-admin protection.

### API Implementation for User Story 4

- [ ] T072 Implement UserService in `src/Jarvis.Api/Services/UserService.cs` (list users, create user, bulk create, change role, deactivate, last-admin guard, bootstrap logic for first user)
- [ ] T073 Implement UsersController in `src/Jarvis.Api/Controllers/UsersController.cs` (GET /api/v1/users, POST /api/v1/users, POST /api/v1/users/bulk, PUT /api/v1/users/{email}/role, DELETE /api/v1/users/{email})
- [ ] T074 Implement TokenService in `src/Jarvis.Api/Services/TokenService.cs` (create PAT with jrv_ prefix + 32 random bytes base62, SHA-256 hash storage, list tokens, revoke own token, admin revoke, admin revoke-all, 10-token limit, 90-day max expiry)
- [ ] T075 Implement token endpoints in `src/Jarvis.Api/Controllers/TokensController.cs` (GET /api/v1/users/me/tokens, POST /api/v1/users/me/tokens, DELETE /api/v1/users/me/tokens/{tokenId}, GET /api/v1/users/{email}/tokens, DELETE /api/v1/users/{email}/tokens/{tokenId}, DELETE /api/v1/users/{email}/tokens)
- [ ] T076 Implement first-user bootstrap middleware in `src/Jarvis.Api/Authentication/BootstrapMiddleware.cs` (on first authenticated request when User table is empty, create admin record for authenticated email)

### Blazor UI Implementation for User Story 4

- [ ] T077 Create user management page in `src/Jarvis.Web/Pages/Admin/UserManagement.razor` (user list table, search, add user form with email + role picker, role change dropdown, deactivate button)
- [ ] T078 [P] Create user API client service in `src/Jarvis.Web/Services/UserApiService.cs` (typed HttpClient for user management endpoints)
- [ ] T079 [P] Create PAT management page in `src/Jarvis.Web/Pages/Profile/PersonalAccessTokens.razor` (list own tokens, create new token form with name + expiry, revoke button, show-once token display modal)
- [ ] T080 [P] Create token API client service in `src/Jarvis.Web/Services/TokenApiService.cs` (typed HttpClient for token endpoints)
- [ ] T081 [P] Create FluentValidation validators in `src/Jarvis.Api/Validation/UserValidators.cs` (CreateUserValidator: email format + role enum; CreateTokenValidator: name required + expiresInDays 1-90)

**Checkpoint**: At this point, User Stories 1-4 should all work - full user lifecycle and self-service tokens are functional

---

## Phase 7: User Story 5 - Aggregate from Multiple Data Sources (Priority: P3)

**Goal**: The system tracks data provenance at the field level. Users can see which source provided each field and when it was last updated.

**Independent Test**: Submit data for the same application from two different sources via API, view the application detail, confirm source attribution and history are displayed.

### Implementation for User Story 5

- [ ] T082 Implement ProvenanceService in `src/Jarvis.Api/Services/ProvenanceService.cs` (record field-level changes with source attribution, query provenance history with pagination and field filtering)
- [ ] T083 Implement provenance endpoint in `src/Jarvis.Api/Controllers/ApplicationsController.cs` (GET /api/v1/applications/{acronym}/provenance with fieldName and pagination query params)
- [ ] T084 Integrate provenance recording into ApplicationWriteService in `src/Jarvis.Api/Services/ApplicationWriteService.cs` (on each field update, record FieldProvenance with source, previousValue, newValue, timestamp)
- [ ] T085 [P] Add provenance detail section to application detail page in `src/Jarvis.Web/Pages/ApplicationDetail.razor` (collapsible panel showing field-level source attribution and change history)
- [ ] T086 [P] Create provenance API client method in `src/Jarvis.Web/Services/ApplicationApiService.cs` (fetch provenance data for display)
- [ ] T087 [P] Add provenance data source indicators to application detail fields in `src/Jarvis.Web/Components/Inventory/ProvenanceBadge.razor` (small badge showing source name and last update date next to each field)

**Checkpoint**: All user stories should now be independently functional

---

## Phase 8: Infrastructure and Deployment

**Purpose**: Leadership election, write-forwarding, Litestream replication, container build, and OpenShift manifests.

- [ ] T088 Implement Kubernetes Lease leader election in `src/Jarvis.Api/Leadership/LeaderElectionService.cs` (BackgroundService using KubernetesClient to acquire/renew Lease, expose IsLeader property, configurable via Leadership:Enabled setting)
- [ ] T089 Implement Litestream coordination in `src/Jarvis.Api/Leadership/LitestreamCoordinator.cs` (start Litestream replication on leader, periodic restore from RWX PVC on followers via BackgroundService every ~5s)
- [ ] T090 Implement write-forwarding middleware in `src/Jarvis.Api/Middleware/WriteForwardingMiddleware.cs` (if not leader and request is mutating, proxy to leader pod IP discovered from Lease holder identity)
- [ ] T091 [P] Create Litestream configuration in `deploy/litestream.yml` (file replica type, path to RWX PVC mount, snapshot interval)
- [ ] T092 Create multi-stage Dockerfile in `deploy/Dockerfile` (Stage 1: .NET SDK restore+build+publish all projects; Stage 2: aspnet:10.0 runtime with published output + Litestream binary + entrypoint script)
- [ ] T093 [P] Create OpenShift Deployment manifest in `deploy/openshift/deployment-api.yaml` (replicas: 2, emptyDir at /data, RWX PVC at /replicas, readiness/liveness probes, resource limits 512MB, Litestream entrypoint)
- [ ] T094 [P] Create PodDisruptionBudget in `deploy/openshift/pdb-api.yaml` (minAvailable: 1)
- [ ] T095 [P] Create RWX PersistentVolumeClaim in `deploy/openshift/pvc-replicas.yaml` (ReadWriteMany, ~100MB, NFS-backed)
- [ ] T096 [P] Create Service and Route in `deploy/openshift/service.yaml` and `deploy/openshift/route.yaml` (ClusterIP service, HTTPS edge-terminated Route)
- [ ] T097 [P] Create ConfigMap and Secret templates in `deploy/openshift/configmap.yaml` and `deploy/openshift/secret.yaml` (non-secret config, KeyCloak client secret placeholder)

---

## Phase 9: CI/CD

**Purpose**: GitHub Actions workflows for build, test, and deployment.

- [ ] T098 Create CI workflow in `.github/workflows/ci.yml` (trigger on push/PR, restore, build, test all projects, publish test results)
- [ ] T099 [P] Create deploy workflow in `.github/workflows/deploy.yml` (trigger on main merge, build Docker image, push to registry, deploy to OpenShift via oc CLI)

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories and final validation.

- [ ] T100 [P] Add structured request/response logging in `src/Jarvis.Api/Middleware/RequestLoggingMiddleware.cs` (correlation ID, request duration, status code, PII masking for email fields)
- [ ] T101 [P] Implement response caching headers for GET /applications and GET /technologies/suggest in `src/Jarvis.Api/Controllers/ApplicationsController.cs` (ETag based on UpdatedAt, Cache-Control for static resources)
- [ ] T102 [P] Add loading states and error boundaries to Blazor pages in `src/Jarvis.Web/Components/Common/LoadingSpinner.razor` and `src/Jarvis.Web/Components/Common/ErrorBoundary.razor`
- [ ] T103 [P] Create responsive mobile layout adjustments in `src/Jarvis.Web/wwwroot/css/responsive.css` (BC Design breakpoints, collapsible filter panel, stacked cards on mobile)
- [ ] T104 Validate quickstart.md workflow end-to-end (clone, restore, docker compose up, run migrations, start app, first login bootstrap, create PAT, API call, MCP connection)
- [ ] T105 [P] Add OpenAPI/Swagger documentation to API endpoints via endpoint metadata in `src/Jarvis.Api/Program.cs` (configure Swashbuckle/NSwag with JWT auth scheme)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Story 1 (Phase 3)**: Depends on Foundational phase (read-only API + Blazor UI)
- **User Story 2 (Phase 4)**: Depends on Foundational phase (write API). Can run in parallel with US1.
- **User Story 3 (Phase 5)**: Depends on Foundational phase. Benefits from US2 (reuses ApplicationWriteService) but can stub/implement independently.
- **User Story 4 (Phase 6)**: Depends on Foundational phase (auth infrastructure already in place).
- **User Story 5 (Phase 7)**: Depends on US2 completion (provenance is recorded during writes).
- **Infrastructure (Phase 8)**: Can begin after Foundational, but full integration requires US2 (write-forwarding needs write endpoints).
- **CI/CD (Phase 9)**: Can begin after Setup (Phase 1) for basic CI; deploy workflow after Infrastructure (Phase 8).
- **Polish (Phase 10)**: Depends on all desired user stories being complete.

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational - No dependencies on other stories
- **User Story 2 (P1)**: Can start after Foundational - No dependencies on other stories. Shares ApplicationService with US1 (read operations).
- **User Story 3 (P2)**: Can start after Foundational - Benefits from US2's ApplicationWriteService but can be developed in parallel by implementing its own write logic if needed
- **User Story 4 (P2)**: Can start after Foundational - Independent of inventory stories. Bootstrap middleware integrates with auth flow.
- **User Story 5 (P3)**: Depends on US2 - Provenance recording hooks into the write service pipeline

### Within Each User Story

- Models before services (for US1/US2, models are in Foundational)
- Services before controllers/endpoints
- API before Blazor UI (UI consumes API)
- Core implementation before integration points
- Story complete before moving to next priority

### Parallel Opportunities

- All Setup tasks marked [P] can run in parallel (T003-T008, T010-T011)
- All entity definitions (T012-T025) can run in parallel
- All DTO definitions (T026-T029) can run in parallel
- Auth components T035-T036 can run in parallel with error handling T038-T039
- Within US1: API (T041-T044) and Blazor static assets (T045-T046) can start in parallel
- US1 and US2 can proceed in parallel after Foundational
- US3 and US4 can proceed in parallel after Foundational
- Infrastructure (Phase 8) tasks T093-T097 are all independent manifests
- All Polish tasks marked [P] can run in parallel

---

## Parallel Example: User Story 1

```
# Launch API service and Blazor styling in parallel:
Task: T041 "Implement ApplicationService in src/Jarvis.Api/Services/ApplicationService.cs"
Task: T045 "Create BC Design System tokens CSS in src/Jarvis.Web/wwwroot/css/bc-design-tokens.css"
Task: T046 "Create base layout component in src/Jarvis.Web/Components/Layout/MainLayout.razor"

# After T041 completes, launch controllers in parallel:
Task: T042 "Implement ApplicationsController in src/Jarvis.Api/Controllers/ApplicationsController.cs"
Task: T043 "Implement TechnologyService in src/Jarvis.Api/Services/TechnologyService.cs"
Task: T044 "Implement TechnologiesController in src/Jarvis.Api/Controllers/TechnologiesController.cs"
```

## Parallel Example: Foundational Phase

```
# All entity models can be created simultaneously:
Task: T012 through T025 (14 entity files, all independent)

# All DTOs can be created simultaneously:
Task: T026 through T029 (4 DTO files, all independent)

# After entities, these can run in parallel:
Task: T033 "Dual-mode auth middleware"
Task: T038 "Exception handling middleware"
Task: T039 "Serilog configuration"
```

---

## Implementation Strategy

### MVP First (User Story 1 + User Story 2)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1 (view inventory)
4. Complete Phase 4: User Story 2 (populate inventory via API)
5. **STOP and VALIDATE**: Test both stories together - can populate data via API and browse via UI
6. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational -> Foundation ready
2. Add User Story 1 + User Story 2 -> Test independently -> Deploy/Demo (MVP!)
3. Add User Story 3 (MCP) -> Test independently -> Deploy/Demo
4. Add User Story 4 (User admin) -> Test independently -> Deploy/Demo
5. Add User Story 5 (Provenance) -> Test independently -> Deploy/Demo
6. Add Infrastructure (Phase 8) -> Enables multi-pod deployment
7. Add CI/CD (Phase 9) -> Automated pipeline
8. Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: User Story 1 (Blazor UI)
   - Developer B: User Story 2 (API writes)
   - Developer C: User Story 4 (User management)
3. After US2 completes:
   - Developer B: User Story 3 (MCP - reuses write service)
   - Developer A: User Story 5 (Provenance - hooks into writes)
4. Infrastructure and CI/CD can be done by any developer in parallel with US3-US5

---

## Notes

- [P] tasks = different files, no dependencies on incomplete tasks in same phase
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Leadership/Litestream (Phase 8) is not needed for local dev or single-pod deployment - it enables multi-pod HA
- BC Design tokens CSS approach avoids React interop complexity while achieving visual compliance
- SQLite RowVersion uses a trigger-updated BLOB - must be configured in migration
- PAT auth middleware detects `jrv_` prefix to route to PAT validation vs JWT validation
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence

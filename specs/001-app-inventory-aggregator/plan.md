# Implementation Plan: Application Inventory Aggregator

**Branch**: `001-app-inventory-aggregator` | **Date**: 2026-05-13 | **Spec**: `specs/001-app-inventory-aggregator/spec.md`

**Input**: Feature specification from `specs/001-app-inventory-aggregator/spec.md`

## Summary

Build an application inventory aggregator that consolidates data from multiple sources into a searchable, read-only web UI. Updates flow exclusively through a REST API and an MCP server. The system uses .NET 10, Blazor WebAssembly for the frontend, an ASP.NET Core API backend, SQL Server 2019 for storage, Keycloak for SSO/OIDC authentication, and deploys to a single Windows Server 2025 VM. Role-based access control (admin/maintainer/reader) governs who can view or modify inventory data. Self-service Personal Access Tokens enable API and MCP access.

## Technical Context

**Language/Version**: .NET 10 (C# 14)

**Primary Dependencies**: ASP.NET Core 10, Blazor WebAssembly (standalone), Entity Framework Core 10 (SQL Server + Design packages), Microsoft.AspNetCore.Authentication.OpenIdConnect and JwtBearer (Keycloak OIDC), ModelContextProtocol SDK for .NET (Streamable HTTP transport), FluentValidation, Serilog (File + EventLog sinks)

**Storage**: SQL Server 2019 (production), LocalDB (development)

**Testing**: xUnit, bUnit (Blazor component tests), Microsoft.AspNetCore.Mvc.Testing (integration tests backed by LocalDB / SQL Server behavior)

**Target Platform**: Windows Server 2025, IIS reverse proxy or Windows Service (Kestrel)

**Project Type**: Web application (Blazor WASM frontend + ASP.NET Core API backend + MCP server)

**Performance Design Targets**: Internal tuning targets only (not acceptance criteria): search results < 500ms, API response < 200ms p95, UI first contentful paint < 2s

**Constraints**: Single-server deployment, SQL Server 2019 (existing on the VM), LocalDB for local database development, Dockerized Keycloak or an approved development OIDC provider for local authentication, pinned NuGet package versions with lock files, warnings treated as errors, and CI gates for locked restore, vulnerability scan, static analysis/lint, build, and tests on every push

**Scale/Scope**: ~5000 application records, ~50 concurrent users, 3 user roles, ~10 data source integrations

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Data Security | PASS | Secrets supplied only via environment variables or an approved secret manager. `appsettings` files contain placeholders or other non-secret defaults only. No PII in logs (structured logging with masking). `.gitignore` excludes local settings and environment files. |
| II. Simplicity First | PASS | Single solution with 3 projects (API+MCP, Blazor WASM, Shared). SQL Server already on the VM — no additional infrastructure. No repository pattern — direct EF Core DbContext. |
| III. Adaptability | PASS | Loose coupling via clean service interfaces. EF Core allows DB swap later. MCP hosted as HTTP middleware in API. Blazor WASM decoupled from API via HTTP client. |
| IV. Regression Safety | PASS | xUnit + bUnit + LocalDB-backed integration tests. CI gates on locked restore, vulnerability scan, static analysis/lint, build, and test pass. Acceptance scenarios from spec map directly to tests. |
| V. Ease of Use | PASS | LocalDB removes the need for a separate dev database. Local auth uses Dockerized Keycloak or an approved dev OIDC provider. Clear API error messages with ProblemDetails. Minimal setup: clone, `dotnet tool restore`, start local Keycloak or configure the dev IdP, `dotnet ef database update`, `dotnet run`. |

**Gate Result**: PASS - No violations. Proceed to Phase 0.

### Post-Design Re-evaluation (after Phase 1)

| Principle | Status | Post-Design Assessment |
|-----------|--------|------------------------|
| I. Data Security | PASS | Keycloak secrets are supplied only via environment variables or an approved secret manager. `appsettings` files carry placeholders or non-secret defaults only. SQL Server connection strings use Windows Authentication where possible. Serilog masks PII. `.gitignore` blocks local settings and environment files. JWT tokens are validated server-side; no secrets live in the WASM client. |
| II. Simplicity First | PASS | 3 projects is the minimum for clean separation (API+MCP, WASM, Shared). SQL Server already on the VM — zero additional infrastructure. No repository pattern — direct EF Core. No event sourcing. BC Design tokens reused rather than wrapping React. Single process deployment. |
| III. Adaptability | PASS | EF Core provider swap is a NuGet + connection string change. MCP transport can switch between HTTP and stdio via configuration. WASM frontend decoupled from API. Field-level provenance allows data source changes without model changes. |
| IV. Regression Safety | PASS | xUnit for API logic, bUnit for Blazor components, and LocalDB-backed integration tests with WebApplicationFactory. CI gates on locked restore, vulnerability scan, static analysis/lint, build, and tests. Acceptance scenarios from the spec map to test cases. |
| V. Ease of Use | PASS | LocalDB keeps local storage lightweight, while local auth is provided by Dockerized Keycloak or an approved dev IdP. ProblemDetails errors include corrective suggestions. Health endpoints support monitoring. Quickstart covers `dotnet tool restore`, local auth setup, migrations, and run/publish flows. |

**Post-Design Gate Result**: PASS - No violations introduced during design phase.

## Project Structure

### Documentation (this feature)

```text
specs/001-app-inventory-aggregator/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output (API contracts)
│   ├── api-rest.md      # REST API contract
│   └── api-mcp.md       # MCP server tools contract
└── tasks.md             # Phase 2 output (not created by /speckit.plan)
```

### Source Code (repository root)

```text
src/
├── Jarvis.Api/                    # ASP.NET Core Web API + hosts Blazor WASM + MCP
│   ├── Controllers/               # API controllers
│   ├── Services/                  # Business logic services
│   ├── Infrastructure/            # EF Core DbContext, migrations, config
│   ├── Authentication/            # Dual-mode auth (Keycloak JWT + PAT), role resolution
│   ├── Middleware/                # Error handling, logging
│   ├── Mcp/                      # MCP tool definitions (Streamable HTTP)
│   ├── Program.cs
│   └── Jarvis.Api.csproj
│
├── Jarvis.Web/                    # Blazor WebAssembly frontend
│   ├── Pages/                     # Routable page components
│   ├── Components/                # Reusable UI components (BC Design System styled)
│   ├── Services/                  # HTTP client services
│   ├── Auth/                      # OIDC auth state provider
│   ├── wwwroot/                   # Static assets, BC Sans font, design tokens CSS
│   ├── Program.cs
│   └── Jarvis.Web.csproj
│
├── Jarvis.Shared/                 # Shared models and DTOs
│   ├── Models/                    # Domain entities
│   ├── Dtos/                      # API request/response DTOs
│   └── Jarvis.Shared.csproj
│
└── Jarvis.sln

tests/
├── Jarvis.Api.Tests/              # API unit + integration tests
│   ├── Controllers/
│   ├── Services/
│   └── Integration/               # Uses LocalDB
├── Jarvis.Web.Tests/              # Blazor component tests (bUnit)
│   └── Components/
└── Jarvis.Mcp.Tests/              # MCP tool tests
    └── Tools/

deploy/
├── docker-compose.yml             # Local Keycloak for development authentication
└── install.ps1                    # Production deployment script (IIS site setup, service install)

azure-pipelines.yml                    # Azure DevOps build + test + publish pipeline
```

**Structure Decision**: Blazor WASM runs as a separate project served as static files by the API host. MCP tools are hosted as Streamable HTTP middleware within the API. Deployment remains a single Windows Server 2025 process (IIS or Windows Service) with SQL Server 2019 on the same VM. LocalDB supports local database development, while local authentication uses Dockerized Keycloak or an approved shared dev IdP. A Shared project holds DTOs and models. Azure DevOps on-prem handles CI/CD with locked restore, vulnerability scanning, static analysis/lint, build, and test gates.

## Complexity Tracking

No constitution violations requiring justification. The 3-project solution structure (Api+MCP, Web, Shared) is the minimum needed to cleanly separate:
- API + MCP concerns (authentication, persistence, business logic, MCP tool protocol)
- Frontend concerns (UI rendering, client-side state)
- Shared concerns (models, DTOs reused across projects)

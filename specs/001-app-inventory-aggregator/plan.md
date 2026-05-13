# Implementation Plan: Application Inventory Aggregator

**Branch**: `001-app-inventory-aggregator` | **Date**: 2026-05-13 | **Spec**: `specs/001-app-inventory-aggregator/spec.md`

**Input**: Feature specification from `specs/001-app-inventory-aggregator/spec.md`

## Summary

Build an application inventory aggregator that consolidates data from multiple sources into a searchable, read-only web UI. Updates flow exclusively through a REST API and an MCP server. The system uses .NET 10, Blazor WebAssembly for the frontend, a minimal ASP.NET Core API backend, SQLite with WAL mode for storage, KeyCloak for SSO/OIDC authentication, and deploys to BC Government OpenShift Gold. Role-based access control (admin/maintainer/reader) governs who can view or modify inventory data.

## Technical Context

**Language/Version**: .NET 10 (C# 14)

**Primary Dependencies**: ASP.NET Core 10, Blazor WebAssembly (standalone), Entity Framework Core 10 (SQLite provider), Microsoft.AspNetCore.Authentication.OpenIdConnect (KeyCloak OIDC), ModelContextProtocol SDK for .NET (Streamable HTTP transport), FluentValidation, Litestream (sidecar/entrypoint binary for SQLite replication), KubernetesClient (for Lease-based leader election)

**Storage**: SQLite with WAL mode embedded in each API pod. Litestream replicates WAL segments to a shared RWX PVC. Kubernetes Lease-based leader election designates one pod as the single writer.

**Testing**: xUnit, bUnit (Blazor component tests), Microsoft.AspNetCore.Mvc.Testing (integration tests), Verify (snapshot testing for contracts)

**Target Platform**: Linux containers on BC Government OpenShift Gold (OKD/Kubernetes)

**Project Type**: Web application (Blazor WASM frontend + ASP.NET Core API backend + MCP server)

**Performance Goals**: Search results < 500ms, API response < 200ms p95, UI first contentful paint < 2s

**Constraints**: Multi-pod deployment with PodDisruptionBudget (minAvailable: 1) on ALL Deployments, minimum 2 API replicas, all traffic through HTTP proxy (OpenShift router), < 512MB memory per pod, no external S3, all storage within namespace

**Scale/Scope**: ~5000 application records, ~50 concurrent users, 3 user roles, ~10 data source integrations

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Data Security | PASS | Secrets via env vars (KeyCloak client secret). No PII in logs (structured logging with masking). .gitignore excludes .env, appsettings.Development.json. |
| II. Simplicity First | PASS | Single solution with 3 projects (API+MCP, Blazor WASM, Shared). Embedded SQLite — zero database pods. No repository pattern abstraction — direct EF Core DbContext. |
| III. Adaptability | PASS | Loose coupling via clean service interfaces. EF Core allows DB swap later. MCP hosted as HTTP middleware in API. Blazor WASM decoupled from API via HTTP client. |
| IV. Regression Safety | PASS | xUnit + bUnit + integration tests. CI pipeline gates on test pass. Acceptance scenarios from spec map directly to integration tests. |
| V. Ease of Use | PASS | Docker Compose for local dev. Clear API error messages with ProblemDetails. BC Design System styling for consistent UX. Minimal setup: clone, docker-compose up. |

**Gate Result**: PASS - No violations. Proceed to Phase 0.

### Post-Design Re-evaluation (after Phase 1)

| Principle | Status | Post-Design Assessment |
|-----------|--------|------------------------|
| I. Data Security | PASS | KeyCloak secrets in env vars / OpenShift Secrets. SQLite on local emptyDir (not network-exposed). Serilog with PII masking. .gitignore blocks appsettings.Development.json. JWT tokens validated server-side; no secrets in WASM client. |
| II. Simplicity First | PASS | 3 projects is the minimum for clean separation (API+MCP, WASM, Shared). Embedded SQLite — zero database pods, zero operators. No repository pattern — direct EF Core. No event sourcing. BC Design tokens reused rather than wrapping React. Single Deployment for the entire app. |
| III. Adaptability | PASS | EF Core provider swap is a NuGet + connection string change. MCP transport can switch between HTTP and stdio via configuration. WASM frontend decoupled from API. Field-level provenance allows data source changes without model changes. Leader election is a BackgroundService — removable if storage changes. |
| IV. Regression Safety | PASS | xUnit for API logic, bUnit for Blazor components, integration tests with WebApplicationFactory. Acceptance scenarios from spec map to test cases. CI gates on test pass. |
| V. Ease of Use | PASS | docker compose up for full local stack. ProblemDetails errors with corrective suggestions. Health endpoints for ops. Quickstart documented. Minimal env vars with safe defaults. |

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
│   ├── Authentication/            # Dual-mode auth (KeyCloak JWT + PAT), role resolution
│   ├── Middleware/                # Error handling, logging, write-forwarding proxy
│   ├── Mcp/                      # MCP tool definitions (Streamable HTTP)
│   ├── Leadership/               # Kubernetes Lease leader election + Litestream coordination
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
│   ├── Leadership/                # Leader election + write-forwarding tests
│   └── Integration/
├── Jarvis.Web.Tests/              # Blazor component tests (bUnit)
│   └── Components/
└── Jarvis.Mcp.Tests/              # MCP tool tests
    └── Tools/

deploy/
├── Dockerfile                     # Multi-stage build for API + WASM + MCP + Litestream
├── docker-compose.yml             # Local development (API + KeyCloak, SQLite local)
├── litestream.yml                 # Litestream configuration (replica path)
└── openshift/                     # OpenShift deployment manifests
    ├── deployment-api.yaml        # API Deployment (replicas: 2+, emptyDir + RWX mounts)
    ├── pdb-api.yaml               # PodDisruptionBudget (minAvailable: 1)
    ├── pvc-replicas.yaml          # RWX PersistentVolumeClaim for Litestream replicas
    ├── service.yaml               # API Service (ClusterIP)
    ├── route.yaml                 # OpenShift Route (HTTPS)
    ├── configmap.yaml             # Non-secret configuration
    └── secret.yaml                # Template for secrets (not committed)

.github/
└── workflows/
    ├── ci.yml                     # Build + test on push/PR
    └── deploy.yml                 # Build image + deploy to OpenShift
```

**Structure Decision**: Blazor WASM as a separate project served as static files by the API host. MCP tools hosted as Streamable HTTP middleware within the API (no separate container). This gives clean code separation while deploying a single container. SQLite is embedded in each pod (emptyDir) with Litestream replicating to a shared RWX PVC — zero database Deployments. A Shared project holds DTOs and models. GitHub Actions handles CI/CD.

## Complexity Tracking

No constitution violations requiring justification. The 3-project solution structure (Api+MCP, Web, Shared) is the minimum needed to cleanly separate:
- API + MCP + leadership concerns (authentication, persistence, business logic, MCP tool protocol, leader election)
- Frontend concerns (UI rendering, client-side state)
- Shared concerns (models, DTOs reused across projects)

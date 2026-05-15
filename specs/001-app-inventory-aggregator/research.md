# Research: Application Inventory Aggregator

**Feature**: 001-app-inventory-aggregator | **Date**: 2026-05-13

## Decision 1: Blazor Server vs Blazor WebAssembly

**Decision**: Blazor WebAssembly (WASM) hosted by the ASP.NET Core API

**Rationale**:
- The UI is read-only (spec assumption). WASM offloads rendering to the browser, keeping server load minimal.
- The frontend is served as static files from the API; no separate hosting needed.
- Clean separation: the WASM client communicates with the backend exclusively via the REST API, making the API the single source of truth for both UI and programmatic consumers.
- ~5000 records with search/filter is well within WASM capability.
- For a single-server deployment, WASM reduces server resource consumption since rendering happens client-side.

**Alternatives considered**:
- **Blazor Server**: Simpler initial development (no separate WASM project), but maintains persistent SignalR connections per user. On a single VM this is fine, but WASM is lighter on server resources and keeps the architecture consistent if scaling is ever needed.
- **Blazor WASM (standalone, separate host)**: Adds a second site/binding to manage. Hosting WASM from the API project is simpler.

## Decision 2: Data Storage

**Decision**: SQL Server 2019, with LocalDB for local development

**Rationale**:
- The production server runs Windows Server 2025 with SQL Server 2019 already available. No additional infrastructure to provision.
- SQL Server handles concurrent reads and writes natively — connection pooling, MVCC, row-level locking. No leader election, write forwarding, or replication needed.
- ~5000 records is a trivial dataset for SQL Server.
- EF Core's SQL Server provider (`Microsoft.EntityFrameworkCore.SqlServer`) is the most mature and feature-complete EF Core provider. Native `rowversion` support for optimistic concurrency.
- **LocalDB for development**: Ships with Visual Studio and the .NET SDK. Zero external dependencies — no Docker, no database server to install. Connection string: `Server=(localdb)\\MSSQLLocalDB;Database=Jarvis;Trusted_Connection=true`. Developers can `dotnet ef database update` and be running immediately.
- SQL Server Management Studio (SSMS) available for database inspection on both dev and production.

**Alternatives considered**:
- **SQLite + Litestream**: Was the previous choice for OpenShift multi-pod. Unnecessary complexity for a single-server deployment. SQL Server is already available on the target VM.
- **PostgreSQL**: Would require installing and maintaining a separate database server on Windows. SQL Server is native to the Windows ecosystem and already present.
- **SQLite (simple, no replication)**: Viable for the dataset size, but loses native concurrency handling, `rowversion`, and full-text search capabilities that SQL Server provides. LocalDB gives the same zero-dependency dev experience.

## Decision 3: Deployment Architecture

**Decision**: Single Windows Server 2025 VM running the application as a Windows Service (Kestrel) or behind IIS as a reverse proxy

**Rationale**:
- The target is a single Windows Server 2025 VM with SQL Server 2019. No container orchestration, no clustering.
- ASP.NET Core runs natively on Windows as a self-contained deployment. Two hosting options:
  - **IIS reverse proxy** (recommended): IIS handles TLS termination, static file caching, and process management. The app runs behind IIS via the ASP.NET Core Module (ANCM). Automatic process restart on crash.
  - **Windows Service (Kestrel direct)**: The app runs as a Windows Service via `Microsoft.Extensions.Hosting.WindowsServices`. Simpler setup but no built-in TLS termination or process management features of IIS.
- `dotnet publish` produces a self-contained deployment folder. Deployment is a file copy + service restart.
- Health checks via `/health` (liveness) and `/health/ready` (readiness, verifies SQL Server connectivity).
- Blazor WASM is published as static files in the API's `wwwroot/` — served by the same process.
- MCP Streamable HTTP endpoint hosted at `/mcp` in the same process.

**Alternatives considered**:
- **OpenShift / Kubernetes**: Was the previous deployment target. Removed in favor of a simpler single-VM deployment. No container orchestration needed for a single-server app.
- **Docker on Windows**: Adds a container runtime dependency for no benefit on a single server. Native deployment is simpler.
- **Azure App Service**: Not available — the target is an on-premises VM.

## Decision 4: Authentication Architecture

**Decision**: KeyCloak OIDC with JWT tokens for both UI and API

**Rationale**:
- Blazor WASM uses OIDC authorization code flow with PKCE to authenticate via KeyCloak.
- API validates JWT bearer tokens from KeyCloak on every request.
- MCP server validates the same JWT tokens (passed as tool arguments or environment).
- Role mapping: KeyCloak provides identity; application maintains its own role table (email-to-role mapping). This allows the "pre-provisioned users" requirement without modifying KeyCloak groups.
- Token refresh handled by the WASM OIDC library (Microsoft.AspNetCore.Components.WebAssembly.Authentication).

**Alternatives considered**:
- **KeyCloak roles/groups for RBAC**: Requires KeyCloak admin access to manage roles. The spec requires in-app role management by administrators. Keeping roles in the application database is simpler and self-contained.
- **Cookie-based auth (Blazor Server)**: Not applicable since we chose WASM.

## Decision 5: BC Design System Integration

**Decision**: Use BC Design Tokens (CSS custom properties) + BC Sans font to style native Blazor components

**Rationale**:
- BC Design System components are React-based; wrapping them for Blazor adds significant complexity (JS interop for each component).
- The design tokens (colors, spacing, typography, border radius) are available as CSS custom properties and can be applied directly to HTML elements rendered by Blazor.
- BC Sans font is freely available and easily loaded via CSS.
- Standard HTML elements styled with BC tokens achieve visual compliance without React dependency.
- Blazor's built-in form components (InputText, InputSelect, EditForm) map cleanly to the BC Design System's form patterns.

**Alternatives considered**:
- **React component JS interop wrappers**: High development cost, fragile interop, maintenance burden when BC updates components. Violates Simplicity First.
- **Embedding React components via iframes**: Poor UX, accessibility issues, complex state management.
- **Using a third-party Blazor component library (MudBlazor, Radzen) then theming**: Adds dependency weight and the theme would still need customization to match BC. Closer match but still not identical, and adds a large dependency.

## Decision 6: MCP Server Implementation

**Decision**: Streamable HTTP transport hosted as middleware within the ASP.NET Core API, using the official ModelContextProtocol NuGet package

**Rationale**:
- Streamable HTTP is the current MCP transport standard (spec 2025-03-26). It uses HTTP POST for client→server requests and Server-Sent Events (SSE) for server→client streaming.
- Hosting MCP as middleware in the API means no separate process or deployment. The MCP endpoint is just another path on the same server (`/mcp`).
- Shares the same authentication (KeyCloak JWT + PATs), authorization (role resolution), and database services. No duplication.
- The `ModelContextProtocol` .NET SDK supports both stdio and HTTP transports. Switching transports is a configuration change in `Program.cs`.
- AI clients (Claude, Copilot, etc.) connect via the server URL (e.g., `https://jarvis.example.gov.bc.ca/mcp`).

**Alternatives considered**:
- **stdio transport**: Would require users to have direct access to the server to run the binary. Streamable HTTP is accessible from any network-connected MCP client.
- **Separate MCP process on the same VM**: Adds process management complexity for no benefit. Same-process hosting shares auth and DB access.
- **Legacy SSE transport (pre-2025-03-26)**: Being superseded by Streamable HTTP. Use the current spec.

## Decision 7: Concurrency and Field-Level Merge

**Decision**: Optimistic concurrency with field-level merge using EF Core's change tracking + SQL Server's native `rowversion` column

**Rationale**:
- SQL Server's `rowversion` (timestamp) type is automatically updated on every row modification. EF Core maps this natively as a concurrency token — no triggers or manual management needed.
- Each application record has a `RowVersion` column. EF Core includes it in the `WHERE` clause on updates, detecting conflicts automatically.
- For field-level merge: the API accepts partial updates (PATCH semantics). Only fields included in the request are updated. Two concurrent updates to different fields both succeed because they modify different columns.
- Two concurrent updates to the same field: second request gets a 409 Conflict for that specific field.
- SQL Server handles concurrent reads and writes from multiple API requests natively via MVCC (READ COMMITTED SNAPSHOT isolation).

**Alternatives considered**:
- **Last-write-wins (no conflict detection)**: Spec explicitly requires field-level merge with conflict rejection for same-field updates. LWW would lose data silently.
- **Event sourcing**: Massive complexity for a CRUD app with ~50 users. Violates Simplicity First.
- **Pessimistic locking**: Unnecessary for this workload. Optimistic concurrency is simpler and performs better for read-heavy scenarios.

## Decision 8: CI/CD

**Decision**: Azure DevOps on-prem for CI/CD. Deployment via `dotnet publish` producing a self-contained folder, deployed to the Windows Server VM.

**Rationale**:
- Azure DevOps Server (on-prem) is the project's CI/CD platform. Pipelines run on push/PR to build, test, and produce a publish artifact.
- `dotnet publish --self-contained --runtime win-x64` produces a standalone deployment folder with no .NET runtime dependency on the server.
- Blazor WASM is published as static files and included in the API's wwwroot.
- Deployment to the VM via a self-hosted agent on the target server, or via artifact download + PowerShell deployment script (`deploy/install.ps1`).
- No container images, no registry, no Dockerfile.
- YAML pipelines (`azure-pipelines.yml`) checked into the repo for pipeline-as-code.

**Alternatives considered**:
- **Docker on Windows Server**: Adds container runtime complexity for a single-server deployment. Native `dotnet publish` is simpler.
- **GitHub Actions**: Requires internet-accessible runners or self-hosted runner infrastructure. Azure DevOps on-prem is already available and keeps CI/CD within the on-prem network.
- **Manual build on the server**: Not repeatable. CI ensures every build is tested and consistent.

## Decision 9: Logging and Observability

**Decision**: Structured logging with Serilog, writing to file (rolling) and Windows Event Log

**Rationale**:
- On a Windows Server VM, the natural log destinations are the filesystem and Windows Event Log.
- Serilog with `Serilog.Sinks.File` (rolling daily, size-capped) for detailed structured JSON logs.
- Serilog with `Serilog.Sinks.EventLog` for critical errors and application lifecycle events, making them visible in Windows Event Viewer.
- Serilog destructuring policies mask PII fields (email addresses in non-debug contexts).
- Health check endpoints (`/health`, `/health/ready`) for monitoring tools or load balancer probes.
- No APM agent needed initially; can add OpenTelemetry later if performance investigation is needed.

**Alternatives considered**:
- **stdout only**: The twelve-factor app pattern, but less useful on a Windows VM where there's no centralized log collector by default. File + Event Log is more practical.
- **Application Insights**: Azure-specific, adds a cloud dependency for an on-premises deployment.
- **ELK/EFK stack**: Overkill for a single-server deployment.

## Decision 10: Data Migration and Seeding

**Decision**: EF Core Migrations for schema management. Initial data load via API bulk import endpoint. SQL Server native backup for disaster recovery.

**Rationale**:
- EF Core Migrations provide versioned, repeatable schema changes.
- Migrations run on application startup (in Development) or via `dotnet ef database update` as a deployment step (in Production).
- Initial inventory data (from the existing JSON structure) is loaded via a bulk import API endpoint that maintainers can call.
- No seed data in migrations — keeps schema changes separate from data changes.
- Backup via SQL Server native backup (full + differential), scheduled via SQL Server Agent or Windows Task Scheduler. Standard, well-understood, no additional tooling.

**Alternatives considered**:
- **Manual SQL scripts**: Less integrated with the ORM, harder to track in source control alongside model changes.
- **DbUp or similar**: Another dependency when EF Core Migrations already handle this well.

## Decision 11: Personal Access Token Authentication for API and MCP

**Decision**: Self-service Personal Access Tokens (PATs) with 90-day maximum lifetime, dual-mode auth middleware (JWT or PAT)

**Rationale**:
- MCP clients (GitHub Copilot in VS Code, Copilot CLI) need a stable credential for the Streamable HTTP endpoint. KeyCloak JWTs expire in minutes and require browser-based OAuth flows that most MCP clients don't support yet.
- Following the GitHub/Azure DevOps model: any authorized user can create PATs for themselves. No administrator involvement in token issuance.
- Tokens are scoped to the user's current role — if the role changes, the token reflects the new permissions on next use.
- Maximum lifetime of 90 days. Users set an expiry up to this limit. Expired tokens are rejected.
- Administrators can list and revoke any user's tokens (for offboarding, security incidents). Users can revoke their own tokens.
- Token format: `jrv_` prefix + 32 random bytes (base62-encoded). The prefix allows secret scanning tools to identify leaked tokens.
- Storage: only the SHA-256 hash is stored in the database. The plaintext token is shown once at creation time and never retrievable again.
- Auth middleware checks the `Authorization: Bearer <token>` header:
  1. If the token starts with `jrv_` → look up hash in the PersonalAccessToken table, verify not expired/revoked, resolve user+role.
  2. Otherwise → validate as a KeyCloak JWT, resolve user+role from the User table by email claim.
  3. Both paths produce the same `ClaimsPrincipal` — downstream code is auth-method agnostic.

**Security considerations**:
- Hashed storage prevents token leakage from database compromise.
- `jrv_` prefix enables GitHub secret scanning and pre-commit hooks to catch accidental commits.
- 90-day cap forces periodic rotation.
- Revocation is immediate (checked on every request, no cached valid-token list).
- PATs inherit the user's role — if an admin demotes a user, existing PATs immediately reflect the reduced permissions.

**Alternatives considered**:
- **MCP Protocol OAuth (built-in)**: Cleanest long-term solution, but GitHub Copilot's MCP client doesn't reliably support MCP's auth spec yet. Can be added later as a secondary auth method.
- **Token helper CLI**: Requires distributing and maintaining a separate binary. Higher friction than self-service PATs.
- **Administrator-issued tokens**: Creates a bottleneck. The GitHub/Azure DevOps model (self-service) is proven at scale and expected by developers.
- **No expiry / unlimited tokens**: Security risk. 90-day cap balances convenience with rotation hygiene.

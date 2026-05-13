# Research: Application Inventory Aggregator

**Feature**: 001-app-inventory-aggregator | **Date**: 2026-05-13

## Decision 1: Blazor Server vs Blazor WebAssembly

**Decision**: Blazor WebAssembly (WASM) hosted by the ASP.NET Core API

**Rationale**:
- Multi-pod deployment is straightforward because WASM runs entirely in the browser; no SignalR connections to manage across pods.
- OpenShift round-robin load balancing works without sticky sessions.
- The API is stateless, so horizontal scaling is trivial.
- The frontend is served as static files; CDN/caching friendly.
- For a read-heavy inventory browser, client-side rendering reduces server load.
- ~5000 records with search/filter is well within WASM capability.

**Alternatives considered**:
- **Blazor Server**: Simpler initial development (no API layer needed for UI), but requires SignalR sticky sessions or a Redis backplane for multi-pod. Adds infrastructure complexity that violates Simplicity First (Principle II).
- **Blazor Server with Redis backplane**: Works multi-pod but introduces a Redis dependency for what is fundamentally a read-only UI. Overkill for this use case.
- **Blazor WASM (standalone, separate host)**: Adds another deployment unit. Hosting WASM from the API project is simpler and reduces container count.

## Decision 2: Data Storage

**Decision**: SQLite embedded in each API pod, with Litestream replication to a shared RWX PVC, and Kubernetes Lease-based leader election for single-writer coordination

**Rationale**:
- ~5000 records is a tiny dataset (~10-20MB SQLite file). Embedding the database in each API pod eliminates any database Deployment entirely — zero extra pods.
- Every Deployment in this namespace must have PDB minAvailable: 1 (requiring ≥2 replicas). Running a database as a Deployment would force either HA PostgreSQL (too complex) or a single-instance PostgreSQL with no PDB (violates the constraint). Embedded SQLite sidesteps this.
- **Leader election via Kubernetes Lease**: one API pod acquires a Lease and becomes the write leader. Only the leader accepts writes and runs Litestream in continuous replication mode, shipping WAL segments to a shared RWX (ReadWriteMany) PVC.
- **Follower pods** serve reads from a local SQLite copy. A BackgroundService periodically restores from the RWX PVC (every ~5s), keeping read staleness within the spec's 5-second SLA (SC-003).
- **No S3 required**: Litestream's `file` replica type writes to a filesystem path. The RWX PVC (NFS-backed via NetApp, standard in OpenShift Gold) provides the shared filesystem. No MinIO, no external S3.
- SQLite WAL mode on the leader pod provides concurrent reads during writes (local to that pod).
- EF Core SQLite provider is mature and well-supported.
- Litestream binary is included in the Docker image (~8MB, single static binary).

**Write forwarding**:
- OpenShift Route distributes requests via round-robin. A write request may hit any pod.
- If a follower pod receives a write request, it proxies it to the leader pod's IP (discovered from the Lease object, which stores the holder's identity). This is a simple HTTP reverse-proxy middleware — ~20 lines of code.
- The leader processes the write against its local SQLite, Litestream replicates the WAL to the RWX PVC, and the follower returns the response to the client.

**Volume layout per pod**:
- `emptyDir` at `/data/` — local SQLite database (ephemeral, rebuilt from RWX on startup)
- RWX PVC at `/replicas/` — shared Litestream WAL segments and snapshots

**Failure modes**:
- Leader pod evicted: another pod acquires the Lease (~5-10s), restores latest state from RWX PVC, becomes the new leader. Writes return 503 during the failover window. Reads continue from all follower pods.
- Follower pod evicted: new pod starts, restores from RWX PVC in init container, begins serving reads. No impact to writes.
- Both scenarios: WASM frontend is already loaded in the browser and continues to function; only API calls are briefly affected.

**Alternatives considered**:
- **Simple single-instance PostgreSQL (no PDB)**: Violates the requirement that all pods must have PDB minAvailable: 1. Accepting no PDB on the database means OpenShift can kill the sole database pod with no replacement ready, causing full write+read outage.
- **HA PostgreSQL (Patroni/CrunchyData)**: Explicitly ruled out as overkill. Adds operator dependency, 3+ pods, complex failover.
- **MinIO for S3-compatible storage**: No external S3 available. MinIO in distributed mode requires 4+ pods. MinIO in standalone mode has the same PDB problem as PostgreSQL. RWX PVC eliminates the need for any S3.
- **SQLite directly on RWX PVC (no Litestream)**: SQLite's documentation warns against running on NFS due to unreliable locking semantics. Litestream avoids this by keeping SQLite on local emptyDir and only storing WAL replicas on NFS.
- **In-memory with S3 persistence**: No S3 available. In-memory with RWX file persistence would lose relational query power and EF Core integration.

## Decision 3: OpenShift Deployment Architecture

**Decision**: Single Deployment (API + WASM + MCP) with multiple replicas, PDB, and embedded SQLite. No database pods.

**Rationale**:
- One Deployment: `jarvis-api`, replicas ≥ 2, PDB minAvailable: 1.
- Each pod carries its own SQLite database (emptyDir) and syncs via Litestream to a shared RWX PVC.
- API container serves Blazor WASM static files, handles REST API requests, and hosts MCP Streamable HTTP endpoints.
- Zero additional Deployments — no database pod, no MinIO, no separate MCP server.
- Health checks via `/health` (liveness) and `/health/ready` (readiness, verifies local SQLite is accessible and Litestream replication is current) on the API pods.
- Standard Kubernetes Deployment (not legacy DeploymentConfig) as OpenShift 4.x supports it natively.
- All ingress/egress goes through OpenShift router (HAProxy) which acts as the HTTP proxy boundary.

**OpenShift resources**:
- `Deployment` (jarvis-api): replicas: 2, Litestream sidecar or entrypoint wrapper
- `PodDisruptionBudget`: minAvailable: 1
- `PersistentVolumeClaim` (RWX): shared Litestream replica store (~50MB, NFS-backed)
- `Service` + `Route`: HTTPS ingress
- `ConfigMap`: non-secret configuration
- `Secret`: KeyCloak client secret, Litestream config (if any)

**Alternatives considered**:
- **API + separate PostgreSQL Deployment**: PostgreSQL also needs PDB minAvailable: 1, which forces HA PostgreSQL (too complex). Embedded SQLite eliminates the problem.
- **Separate MCP container (stdio)**: stdio transport cannot traverse the HTTP proxy. MCP as Streamable HTTP middleware in the API eliminates a second Deployment.
- **Single replica**: Violates the requirement for PodDisruptionBudget of 1. A single pod would cause downtime during eviction.

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
- All traffic into and out of OpenShift traverses an HTTP proxy (OpenShift router). stdio transport cannot pass through this boundary — external AI clients have no way to exec into pods or attach to stdin.
- Streamable HTTP is the current MCP transport standard (spec 2025-03-26). It uses HTTP POST for client→server requests and Server-Sent Events (SSE) for server→client streaming. Both work through HTTP proxies.
- Hosting MCP as middleware in the API eliminates a separate container, Deployment, and Route. The MCP endpoint is just another path on the same pods (`/mcp`).
- Shares the same authentication (KeyCloak JWT), authorization (role resolution), and database services. No duplication.
- The `ModelContextProtocol` .NET SDK supports both stdio and HTTP transports. Switching from stdio to HTTP is a configuration change in `Program.cs`.
- AI clients (Claude, Copilot, etc.) connect via the public route URL (e.g., `https://jarvis.apps.gold.devops.gov.bc.ca/mcp`).

**Alternatives considered**:
- **stdio transport in a sidecar container**: AI clients would need `oc exec` access to the pod. Requires OpenShift RBAC for pod exec, which is a security concern and operational complexity. Not viable through HTTP proxy.
- **stdio transport as a separate Deployment**: Same proxy problem. Also adds a container, Deployment, and Service that serve no purpose since stdio doesn't listen on a port.
- **Legacy SSE transport (pre-2025-03-26)**: Separate SSE endpoint + HTTP POST. Functionally similar to Streamable HTTP but being superseded. Use the current spec.
- **Separate HTTP MCP container**: Works through proxy but adds a second Deployment for no benefit. The MCP tools call the same services as the REST API — hosting in the same process avoids network hops and auth duplication.

## Decision 7: Concurrency and Field-Level Merge

**Decision**: Optimistic concurrency with field-level merge using EF Core's change tracking + a RowVersion column. Single-writer guarantee simplifies conflict handling.

**Rationale**:
- The leader election pattern (Decision 2) guarantees a single writer process at any time. This eliminates database-level write contention entirely — SQLite's single-writer model is a feature, not a limitation.
- Concurrency conflicts arise only from *API clients* submitting overlapping updates, not from database-level race conditions between pods.
- Each application record has a `RowVersion` (concurrency token, stored as a SQLite BLOB updated via trigger).
- On update, the API loads the current record, applies field-level changes, and saves. EF Core detects concurrency conflicts via the version column.
- For field-level merge: the API accepts partial updates (PATCH semantics). Only fields included in the request are updated. Two concurrent updates to different fields both succeed because they modify different columns.
- Two concurrent updates to the same field: second request gets a 409 Conflict for that specific field.

**Alternatives considered**:
- **Last-write-wins (no conflict detection)**: Spec explicitly requires field-level merge with conflict rejection for same-field updates. LWW would lose data silently.
- **Event sourcing**: Massive complexity for a CRUD app with ~50 users. Violates Simplicity First.
- **Distributed locking**: Unnecessary — there is literally one writer process.

## Decision 8: CI/CD and Container Build

**Decision**: GitHub Actions for CI/CD. Multi-stage Dockerfile with .NET SDK build + runtime image. Push to OpenShift internal registry or external registry (e.g., GitHub Container Registry).

**Rationale**:
- GitHub Actions is the project's CI platform. Workflows run on push/PR to build, test, and optionally deploy.
- Multi-stage Dockerfile: Stage 1 restores + builds + publishes all projects. Stage 2 copies published output to a minimal runtime image (`mcr.microsoft.com/dotnet/aspnet:10.0`).
- Blazor WASM is published as static files and included in the API's wwwroot.
- Image size optimization: trim unused assemblies, use alpine-based runtime where possible.
- Health check endpoints enable OpenShift readiness/liveness probes.
- ConfigMaps for non-secret configuration, Secrets for KeyCloak client credentials and database connection string.
- GitHub Actions workflow stages: restore → build → test → publish → docker build → push → (optional) deploy to OpenShift via `oc` CLI or Helm.

**Alternatives considered**:
- **OpenShift BuildConfig / S2I**: OpenShift-specific, less portable. Standard Dockerfile + GitHub Actions is more universal and keeps CI close to the code.
- **Buildpacks**: Less control over build process. .NET Dockerfiles are well-understood.
- **Jenkins**: No indication of Jenkins infrastructure. GitHub Actions is already available.

## Decision 9: Logging and Observability

**Decision**: Structured logging with Serilog, writing to stdout (OpenShift collects via Fluentd/EFK)

**Rationale**:
- OpenShift Gold clusters typically have EFK (Elasticsearch, Fluentd, Kibana) or Loki for log aggregation.
- Writing structured JSON logs to stdout is the twelve-factor app pattern and works with any log collector.
- Serilog with destructuring policies to mask PII fields (email addresses in non-debug contexts).
- Health check endpoints (/health, /health/ready) for Kubernetes probes.
- No APM agent needed initially; can add OpenTelemetry later if performance investigation is needed.

**Alternatives considered**:
- **Application Insights**: Azure-specific, not appropriate for OpenShift.
- **Direct EFK integration**: Coupling to infrastructure. stdout is simpler and portable.

## Decision 10: Data Migration and Seeding

**Decision**: EF Core Migrations for schema management. Initial data load via API bulk import endpoint. Litestream replication to RWX PVC serves as continuous backup.

**Rationale**:
- EF Core Migrations provide versioned, repeatable schema changes.
- Migrations run on application startup (in Development) or as part of the leader pod's init sequence (in Production).
- Initial inventory data (from the existing JSON structure) is loaded via a bulk import API endpoint that maintainers can call.
- No seed data in migrations — keeps schema changes separate from data changes.
- Backup is inherent in the architecture: Litestream continuously replicates WAL segments to the RWX PVC. To restore, a new pod simply runs Litestream restore. No separate backup CronJob needed.

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

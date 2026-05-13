# Quickstart: Application Inventory Aggregator

**Feature**: 001-app-inventory-aggregator | **Date**: 2026-05-13

## Prerequisites

- .NET 10 SDK (https://dotnet.microsoft.com/download/dotnet/10.0)
- Docker Desktop (for local KeyCloak and optional container builds)
- Git

## 1. Clone and Restore

```bash
git clone <repo-url> jarvis
cd jarvis
dotnet restore src/Jarvis.sln
```

## 2. Start Local Infrastructure

```bash
docker compose -f deploy/docker-compose.yml up -d keycloak
```

This starts:
- **KeyCloak** on http://localhost:8080 (admin/admin) with a pre-configured `jarvis` realm, client, and test users

Note: In local development, the app runs as a single process with SQLite — no leader election, no Litestream, no shared PVC. The multi-pod replication architecture only applies in OpenShift.

## 3. Configure the Application

Copy the template settings file:

```bash
cp src/Jarvis.Api/appsettings.Development.template.json src/Jarvis.Api/appsettings.Development.json
```

The template contains working defaults for local development:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=jarvis.db"
  },
  "Authentication": {
    "Authority": "http://localhost:8080/realms/jarvis",
    "ClientId": "jarvis-api",
    "RequireHttpsMetadata": false
  },
  "Leadership": {
    "Enabled": false
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

Note: `Leadership.Enabled = false` disables Kubernetes Lease leader election and Litestream replication for local development. The app runs as a direct single-writer SQLite process.

## 4. Run Database Migrations

```bash
cd src/Jarvis.Api
dotnet ef database update
```

This creates the SQLite database file (`jarvis.db`) with all tables.

## 5. Run the API (serves both API and Blazor WASM)

```bash
dotnet run --project src/Jarvis.Api
```

The application starts at:
- **Web UI**: https://localhost:5001
- **API**: https://localhost:5001/api/v1
- **MCP**: https://localhost:5001/mcp (Streamable HTTP)
- **Health**: https://localhost:5001/health

## 6. First Login (Bootstrap)

1. Navigate to https://localhost:5001
2. You are redirected to KeyCloak login
3. Sign in with a test user (e.g., `admin@test.local` / `password`)
4. Since no users exist in the system, the first authenticated user is automatically assigned the administrator role
5. You now have full access and can provision other users

## 7. Create a Personal Access Token (for API/MCP use)

After logging in, create a PAT for programmatic access:

**Via the web UI**: Profile → Personal Access Tokens → Create New Token

**Via the API**:

```bash
curl -X POST https://localhost:5001/api/v1/users/me/tokens \
  -H "Authorization: Bearer <jwt-from-login>" \
  -H "Content-Type: application/json" \
  -d '{"name": "VS Code MCP", "expiresInDays": 90}'
```

Response includes the token (shown once — save it):

```json
{
  "id": 1,
  "name": "VS Code MCP",
  "token": "jrv_a3Bx9kLm2nPqRs4tUv6wXy8zA1cD3eF5gH7iJ",
  "expiresAt": "2026-08-11T10:00:00Z"
}
```

## 8. Connect an MCP Client (optional)

The MCP server is hosted at `https://localhost:5001/mcp` using Streamable HTTP transport. Use your PAT for authentication.

**GitHub Copilot in VS Code** (`.vscode/mcp.json`):

```json
{
  "servers": {
    "jarvis-inventory": {
      "type": "http",
      "url": "https://localhost:5001/mcp",
      "headers": {
        "Authorization": "Bearer ${JARVIS_TOKEN}"
      }
    }
  }
}
```

Set the environment variable: `export JARVIS_TOKEN="jrv_a3Bx9k..."`

**Claude Desktop** (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "jarvis-inventory": {
      "transport": "streamable-http",
      "url": "https://localhost:5001/mcp",
      "headers": {
        "Authorization": "Bearer jrv_a3Bx9k..."
      }
    }
  }
}
```

## 9. Run Tests

```bash
dotnet test src/Jarvis.sln
```

Individual test projects:

```bash
dotnet test tests/Jarvis.Api.Tests
dotnet test tests/Jarvis.Web.Tests
dotnet test tests/Jarvis.Mcp.Tests
```

Note: Integration tests use SQLite in-memory databases and do not require Docker.

## 10. Full Docker Build (optional)

Build and run the complete stack in containers:

```bash
docker compose -f deploy/docker-compose.yml up --build
```

This starts:
- KeyCloak (http://localhost:8080)
- Jarvis API + Web + MCP (http://localhost:5000, SQLite local, no leader election)

## Common Tasks

All examples below use a PAT. Replace `$JARVIS_TOKEN` with your token or set it as an environment variable.

### Import sample data

```bash
curl -X POST https://localhost:5001/api/v1/applications/bulk \
  -H "Authorization: Bearer $JARVIS_TOKEN" \
  -H "Content-Type: application/json" \
  -d @sample-data.json
```

### Add a new user via API

```bash
curl -X POST https://localhost:5001/api/v1/users \
  -H "Authorization: Bearer $JARVIS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email": "new.user@gov.bc.ca", "role": "reader"}'
```

### Update an application via API

```bash
curl -X PATCH https://localhost:5001/api/v1/applications/APP1 \
  -H "Authorization: Bearer $JARVIS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "maintenance", "source": "manual_update"}'
```

### Revoke a token

```bash
curl -X DELETE https://localhost:5001/api/v1/users/me/tokens/1 \
  -H "Authorization: Bearer $JARVIS_TOKEN"
```

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `ConnectionStrings__DefaultConnection` | No | `Data Source=jarvis.db` | SQLite connection string |
| `Authentication__Authority` | Yes | - | KeyCloak realm URL |
| `Authentication__ClientId` | Yes | - | OIDC client ID |
| `Authentication__ClientSecret` | In Prod | - | OIDC client secret (confidential client) |
| `Leadership__Enabled` | No | `true` | Enable Kubernetes Lease leader election (disable for local dev) |
| `Leadership__LeaseName` | No | `jarvis-leader` | Kubernetes Lease resource name |
| `Litestream__ReplicaPath` | In Prod | - | Path to shared RWX PVC for Litestream replicas |
| `ASPNETCORE_ENVIRONMENT` | No | `Production` | Runtime environment |

## Project Structure Quick Reference

```
src/Jarvis.Api/          -> ASP.NET Core API + Blazor WASM host + MCP (Streamable HTTP)
src/Jarvis.Web/          -> Blazor WebAssembly frontend
src/Jarvis.Shared/       -> Shared models and DTOs
tests/                   -> xUnit + bUnit tests
deploy/                  -> Dockerfile + OpenShift manifests
.github/workflows/       -> GitHub Actions CI/CD
```

## Troubleshooting

**"Access Denied" after login**: Your email is not provisioned in the system. If this is a fresh deployment, the first user should automatically become admin. Check that the database migration has run.

**"Connection refused" to KeyCloak**: Ensure the KeyCloak container is running: `docker compose -f deploy/docker-compose.yml ps`

**Database locked errors**: Should not occur in local dev (single writer, Leadership disabled). If running multiple instances locally, ensure only one has write access.

**WASM not loading**: Clear browser cache. Blazor WASM downloads .NET assemblies which can be aggressively cached.

**Write requests returning 503 in OpenShift**: The leader pod may be restarting or transitioning. Wait ~10 seconds for a new leader to be elected. If persistent, check that the Lease exists: `oc get lease jarvis-leader`.

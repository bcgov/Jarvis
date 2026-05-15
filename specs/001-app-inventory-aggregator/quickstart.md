# Quickstart: Application Inventory Aggregator

**Feature**: 001-app-inventory-aggregator | **Date**: 2026-05-13

## Prerequisites

- .NET 10 SDK (https://dotnet.microsoft.com/download/dotnet/10.0)
- Git
- Docker Desktop (optional — only needed for local KeyCloak)

LocalDB ships with the .NET SDK and Visual Studio. No database server to install.

## 1. Clone and Restore

```bash
git clone <repo-url> jarvis
cd jarvis
dotnet restore src/Jarvis.sln
```

## 2. Start Local KeyCloak (optional)

```bash
docker compose -f deploy/docker-compose.yml up -d keycloak
```

This starts **KeyCloak** on http://localhost:8080 (admin/admin) with a pre-configured `jarvis` realm, client, and test users.

If you skip KeyCloak, you can still use the API with a Personal Access Token after bootstrapping the first user via another method.

## 3. Configure the Application

Copy the template settings file:

```bash
cp src/Jarvis.Api/appsettings.Development.template.json src/Jarvis.Api/appsettings.Development.json
```

The template contains working defaults for local development:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\MSSQLLocalDB;Database=Jarvis;Trusted_Connection=true"
  },
  "Authentication": {
    "Authority": "http://localhost:8080/realms/jarvis",
    "ClientId": "jarvis-api",
    "RequireHttpsMetadata": false
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

## 4. Run Database Migrations

```bash
cd src/Jarvis.Api
dotnet ef database update
```

This creates the `Jarvis` database in LocalDB with all tables and indexes.

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
  "token": "jrv_*****",
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

Integration tests use LocalDB and do not require Docker or external services.

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
| `ConnectionStrings__DefaultConnection` | No | LocalDB | SQL Server connection string |
| `Authentication__Authority` | Yes | - | KeyCloak realm URL |
| `Authentication__ClientId` | Yes | - | OIDC client ID |
| `Authentication__ClientSecret` | In Prod | - | OIDC client secret (confidential client) |
| `ASPNETCORE_ENVIRONMENT` | No | `Production` | Runtime environment |

## Project Structure Quick Reference

```
src/Jarvis.Api/          -> ASP.NET Core API + Blazor WASM host + MCP (Streamable HTTP)
src/Jarvis.Web/          -> Blazor WebAssembly frontend
src/Jarvis.Shared/       -> Shared models and DTOs
tests/                   -> xUnit + bUnit tests
deploy/                  -> docker-compose.yml (KeyCloak) + install.ps1 (production)
azure-pipelines.yml      -> Azure DevOps CI/CD pipeline
```

## Troubleshooting

**"Access Denied" after login**: Your email is not provisioned in the system. If this is a fresh deployment, the first user should automatically become admin. Check that the database migration has run.

**"Connection refused" to KeyCloak**: Ensure the KeyCloak container is running: `docker compose -f deploy/docker-compose.yml ps`

**Cannot connect to LocalDB**: Verify LocalDB is installed: `sqllocaldb info MSSQLLocalDB`. If the instance doesn't exist, create it: `sqllocaldb create MSSQLLocalDB`. Start it: `sqllocaldb start MSSQLLocalDB`.

**Migration fails**: Ensure the .NET EF Core tools are installed: `dotnet tool install --global dotnet-ef`. Verify the connection string in `appsettings.Development.json` points to your LocalDB instance.

**WASM not loading**: Clear browser cache. Blazor WASM downloads .NET assemblies which can be aggressively cached.

# TodoApp Copilot Instructions

## Architecture Overview

This is a **distributed Todo application** built with **.NET 9 + Aspire** featuring a **Backend-for-Frontend (BFF) pattern**:
- `TodoApp.AppHost` - Aspire orchestrator that runs the entire app stack
- `Todo.Api` - REST API backend with minimal APIs, Entity Framework, SQLite
- `Todo.Web.Server` - Blazor Server (BFF) that proxies requests using YARP's IHttpForwarder
- `Todo.Web.Client` - Blazor WebAssembly frontend
- `TodoApp.ServiceDefaults` - shared Aspire services (OpenTelemetry, health checks, service discovery)

## Key Development Patterns

### Running the Application
Always run via the **AppHost project** (`TodoApp.AppHost`), not individual projects:
```bash
cd TodoApp.AppHost
dotnet run
```
This starts all services with proper service discovery and observability via Aspire Dashboard.

### Authentication & Authorization Architecture
- **Dual auth system**: Bearer tokens (API) + Cookie auth (Web)
- **CurrentUser pattern**: Scoped service injected into minimal API endpoints
- **Data protection shared** between BFF and API via `ApplicationDiscriminator = "TodoApp"`
- APIs require `RequireCurrentUser()` authorization policy, not standard [Authorize]
- Per-user rate limiting using `RequirePerUserRateLimit()`

### Minimal API Patterns
Follow the established pattern in `TodoApi.cs`:
```csharp
group.RequireAuthorization(pb => pb.RequireCurrentUser());
group.RequirePerUserRateLimit();
group.WithParameterValidation(typeof(TodoItem));
```
- Use `Results<T, U>` return types for clear API contracts
- CurrentUser dependency injection for user context
- Row-level security via `todo.OwnerId == owner.Id` checks

### Database & Migrations
- **SQLite** with Entity Framework Core
- Migrations auto-applied via `AddTodoDbMigration()` in AppHost
- Connection string: `Data Source=.db/Todos.db`
- Use `AsNoTracking()` for read-only queries

### Testing Strategy
- **WebApplicationFactory pattern** in `TodoApplication.cs`
- In-memory SQLite for isolated tests: `"Filename=:memory:"`
- Custom `AuthHandler` for authenticated test clients
- Always call `CreateUserAsync()` before testing authenticated endpoints

### Service Discovery & HTTP Clients
- Internal service calls use service names: `http://todoapi`
- BFF proxies to API using `AddHttpForwarderWithServiceDiscovery()`
- Configure HttpClient with base address pointing to service name

### Blazor Client Patterns
- `TodoClient` service handles API communication
- Handle both server-side and client-side execution via `client.BaseAddress` checks
- Return tuple patterns: `(HttpStatusCode, T?)` for error handling

### Project Structure Conventions
- **ServiceDefaults**: shared Aspire configuration across all projects
- **Global.json**: pins .NET version for team consistency
- **Directory.Build.props**: common MSBuild properties (net9.0, nullable enabled)
- **Directory.Packages.props**: centralized package version management

### Development Tools
- **Scalar API docs** at root endpoint (not Swagger)
- **SQLite Web UI** for database inspection (containerized)
- **Aspire Dashboard** for observability (traces, metrics, logs)
- **OpenTelemetry** configured by default in ServiceDefaults

## Debugging & Troubleshooting
- Check Aspire Dashboard for distributed traces when requests fail
- Verify service discovery is working via dashboard endpoints
- Database issues: inspect via SQLite Web container or connection string
- Authentication: ensure data protection keys are shared (same ApplicationDiscriminator)

## Common Operations
- **Add new API endpoint**: Follow TodoApi pattern with CurrentUser injection
- **Database changes**: Add migration, AppHost will auto-apply
- **New service**: Reference TodoApp.ServiceDefaults, register in AppHost
- **Testing**: Use TodoApplication factory, create test users, use AuthHandler for auth
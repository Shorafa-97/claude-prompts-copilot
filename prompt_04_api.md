# Prompt 04 — API Layer (Controllers, JWT Auth, Middleware)

## Role
You are continuing the `ContactsX` .NET 9 backend. Core, Infrastructure, and Application are complete. Now implement the **API layer** (`ContactsX.API`).

---

## Few-Shot Examples (Chain-of-Thought)

### Example A — Thin controller dispatching MediatR

**Input:** "How should I write the ContactsController?"

**Reasoning (CoT):**
1. Controllers in Clean Architecture are thin — they receive HTTP input, dispatch a MediatR command/query, and return an HTTP response.
2. No business logic lives in controllers. Mapping from request body to command happens at the controller boundary.
3. I use `[ApiController]` so model binding errors automatically return 400 without manual checks.
4. I inject `ISender` (MediatR) not `IMediator`, because controllers only need to send — not publish events.

**Output:**
```csharp
[ApiController]
[Route("api/contacts")]
[Authorize]
public class ContactsController(ISender sender) : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetAll(
        [FromQuery] string? search,
        [FromQuery] ContactType? type,
        [FromQuery] bool? isActive,
        [FromQuery] int page = 1,
        [FromQuery] int limit = 20,
        CancellationToken ct = default)
    {
        var result = await sender.Send(
            new GetContactsQuery(search, type, isActive, page, limit), ct);
        return Ok(result);
    }

    [HttpPost]
    public async Task<IActionResult> Create(
        [FromBody] CreateContactCommand cmd,
        CancellationToken ct)
    {
        var contact = await sender.Send(cmd, ct);
        return CreatedAtAction(nameof(GetById), new { id = contact.Id }, contact);
    }
}
```

---

### Example B — Global exception handler middleware

**Input:** "How do I handle `NotFoundException` and `ValidationException` globally without try/catch in every controller?"

**Reasoning (CoT):**
1. .NET 9 has `IExceptionHandler` — I implement it and register via `app.UseExceptionHandler()`.
2. `NotFoundException` → 404 with `{ error: "message" }`
3. `ValidationException` (FluentValidation) → 400 with `{ errors: { field: ["message"] } }`
4. All other exceptions → 500 with a generic message (never expose stack traces in production).

**Output:**
```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext ctx, Exception ex, CancellationToken ct)
    {
        var (status, body) = ex switch
        {
            NotFoundException nfe => (404, new { error = nfe.Message }),
            ValidationException ve => (400, (object)new
            {
                errors = ve.Errors
                    .GroupBy(e => e.PropertyName)
                    .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage))
            }),
            _ => (500, new { error = "An unexpected error occurred." })
        };
        ctx.Response.StatusCode = status;
        await ctx.Response.WriteAsJsonAsync(body, ct);
        return true;
    }
}
```

---

## Your Task

Generate the complete **API layer** for `ContactsX`.

### Step 1 — Program.cs
Generate the full `Program.cs` using .NET 9 minimal hosting model:
- Call `builder.Services.AddApplication()` and `builder.Services.AddInfrastructure(config)`
- Register `GlobalExceptionHandler`
- Configure JWT Bearer authentication (secret from `config["Jwt:Secret"]`, issuer, audience)
- Add `Swagger/OpenAPI` with JWT security definition
- Add CORS (allow all origins for development, configurable for production)
- Add `app.UseAuthentication()` and `app.UseAuthorization()`
- Run EF migrations on startup in development (using `IServiceScope`)

### Step 2 — Auth controller
Create `AuthController` with:
```
POST /api/auth/login
  body: { username, password }
  returns: { token: "JWT...", user: { id, username } }
```
- For this phase, accept credentials from `appsettings.json` (future: database users per BRD out-of-scope note)
- Generate a signed JWT with claims: `sub`, `name`, `role`
- Token expiry: 8 hours

Create `Application/Auth/LoginCommand.cs` and `LoginHandler.cs` returning `AuthResponseDto { Token, UserId, Username }`.

### Step 3 — All REST controllers
Generate one controller per resource, exactly matching the API endpoints in `API_ENDPOINTS.md`:

| Controller | Route prefix | Key endpoints |
|-----------|-------------|--------------|
| `ContactsController` | `/api/contacts` | GET (paged+search+filter), GET `:id`, POST, PUT `:id`, DELETE `:id` |
| `EntitiesController` | `/api/entities` | GET, GET `:id`, POST, PUT `:id`, DELETE `:id`, GET `:id/children` |
| `RelationshipsController` | `/api/relationships` | GET (filter by contactId/entityId), POST, DELETE `:id` |
| `SearchController` | `/api/search` | GET `?query=` |
| `DuplicatesController` | `/api/duplicates` | GET (filter by type+status), POST `/detect`, POST `:id/merge`, POST `:id/dismiss` |
| `ImportController` | `/api/import` | POST (body: `{ type, records }`) |
| `AuditLogsController` | `/api/audit-logs` | GET (paged, filtered by entityType + action) |
| `AnalyticsController` | `/api/dashboard/kpis` | GET |

Rules for all controllers:
- Every endpoint requires `[Authorize]` except login
- Use `CancellationToken ct` on every async action
- Return `201 Created` with `Location` header on POST, `204 No Content` on DELETE
- Accept `PUT` (full replace) for contacts and entities, not `PATCH`, matching `API_ENDPOINTS.md`

### Step 4 — Middleware and filters
Create:
- `GlobalExceptionHandler` (shown in Example B above) — register as `IExceptionHandler`
- `RequestLoggingMiddleware` — logs `{Method} {Path} → {StatusCode} in {ElapsedMs}ms` using `ILogger`

### Step 5 — appsettings.json
Generate a complete `appsettings.json` and `appsettings.Development.json` with:
```json
{
  "ConnectionStrings": { "Postgres": "Host=localhost;Database=contactsx;Username=...;Password=..." },
  "Jwt": { "Secret": "...", "Issuer": "contactsx-api", "Audience": "contactsx-client", "ExpiryHours": 8 },
  "Auth": { "Username": "admin", "Password": "changeme" },
  "AllowedHosts": "*"
}
```

Think step by step. For each controller action, state the MediatR command/query it dispatches and the HTTP status code it returns on success and on failure.

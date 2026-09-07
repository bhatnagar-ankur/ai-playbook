---
name: dotnet-webapi-development
version: 1.0.0
technology: dotnet
author: Ankur Bhatnagar
last_updated: 2026-09-07
description: >
  ASP.NET Core Web API development skill for .NET 8 LTS.
  Covers controllers, Minimal APIs, dependency injection, EF Core,
  validation, JWT authentication, HTTP client patterns, options,
  logging, performance, and xUnit testing.
  Example trigger phrases: "create a controller", "add a repository",
  "scaffold an endpoint", "add JWT authentication", "write a unit test for a service".
references:
  - references/controllers.md
  - references/data-layer.md
  - references/auth.md
  - references/testing.md
  - references/examples.md
---

# .NET Web API Development Skill

This skill guides Claude to produce idiomatic, production-ready ASP.NET Core Web API code
targeting .NET 8 LTS. Follow every rule in this document unless the **Customizing** section
overrides it.

---

## 1. When to Use This Skill

Use this skill when working on:
- ASP.NET Core Web API projects (controller-based or Minimal API)
- .NET class libraries consumed by an API
- EF Core data access, migrations, and repositories
- JWT authentication and policy-based authorization
- Typed HTTP clients and service integrations
- xUnit unit and integration tests for API projects

Do **not** apply to: Blazor, WPF, WinForms, MAUI, Azure Functions, or projects targeting
.NET Framework (< .NET 5).

---

## 2. Project Structure

```
src/
├── YourApp.Api/                    # Entry point — hosts the ASP.NET Core app
│   ├── Program.cs                  # Composition root: builder + middleware pipeline
│   ├── appsettings.json
│   ├── appsettings.Development.json
│   ├── Controllers/                # Controller-based endpoints
│   │   └── OrdersController.cs
│   ├── Endpoints/                  # Minimal API endpoint classes (IEndpoint pattern)
│   │   └── OrderEndpoints.cs
│   ├── Middleware/                 # Custom middleware
│   │   └── ExceptionHandlingMiddleware.cs
│   └── Extensions/                 # WebApplication / IServiceCollection extensions
│       ├── ServiceCollectionExtensions.cs
│       └── WebApplicationExtensions.cs
├── YourApp.Application/            # Business logic — no infrastructure dependencies
│   ├── Interfaces/                 # Service contracts
│   ├── Services/                   # Service implementations
│   ├── Models/                     # Request/response DTOs
│   └── Validators/                 # FluentValidation validators
├── YourApp.Domain/                 # Domain entities, enums, value objects
│   ├── Entities/
│   └── Enums/
├── YourApp.Infrastructure/         # EF Core, HTTP clients, external integrations
│   ├── Data/
│   │   ├── AppDbContext.cs
│   │   ├── Configurations/         # IEntityTypeConfiguration<T> classes
│   │   └── Migrations/
│   ├── Repositories/
│   └── HttpClients/
└── YourApp.Tests/                  # xUnit test project
    ├── Unit/
    └── Integration/
```

**Solution file** at the repo root: `YourApp.sln`.

---

## 3. Controllers & Minimal APIs

### Controller-based (default for CRUD resources)

```csharp
// Controllers/OrdersController.cs

[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public sealed class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;

    public OrdersController(IOrderService orderService)
    {
        _orderService = orderService;
    }

    [HttpGet]
    [ProducesResponseType<IReadOnlyList<OrderResponse>>(StatusCodes.Status200OK)]
    public async Task<IActionResult> GetAll(CancellationToken ct)
    {
        var orders = await _orderService.GetAllAsync(ct);
        return Ok(orders);
    }

    [HttpGet("{id:int}")]
    [ProducesResponseType<OrderResponse>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(int id, CancellationToken ct)
    {
        var order = await _orderService.GetByIdAsync(id, ct);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpPost]
    [ProducesResponseType<OrderResponse>(StatusCodes.Status201Created)]
    [ProducesResponseType<ValidationProblemDetails>(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create(
        [FromBody] CreateOrderRequest request,
        CancellationToken ct)
    {
        var order = await _orderService.CreateAsync(request, ct);
        return CreatedAtAction(nameof(GetById), new { id = order.Id }, order);
    }

    [HttpPatch("{id:int}/status")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> UpdateStatus(
        int id,
        [FromBody] UpdateOrderStatusRequest request,
        CancellationToken ct)
    {
        var success = await _orderService.UpdateStatusAsync(id, request, ct);
        return success ? NoContent() : NotFound();
    }

    [HttpDelete("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id, CancellationToken ct)
    {
        var success = await _orderService.DeleteAsync(id, ct);
        return success ? NoContent() : NotFound();
    }
}
```

**Controller rules:**
- Always `[ApiController]` + `[Route]` — never route via `[HttpGet("path")]` alone
- Return `IActionResult` (not `ActionResult<T>`) — every action in this skill's controllers uses a consistent return type regardless of how many possible outcomes it has (Ok/NotFound/BadRequest/…), so callers and filters don't need to special-case `ActionResult<T>`'s dual nature; document the success payload separately with `[ProducesResponseType<T>]` for Swagger. (Note: current Microsoft guidance favors `ActionResult<T>` for its compile-time payload typing — teams that value that over return-type uniformity may reasonably override this rule via §15.)
- Never put business logic in controllers — delegate to a service
- Always accept `CancellationToken ct` and pass it to all async calls
- Use `sealed` on controllers — they are not designed for inheritance

### CORS

Required before any browser-based frontend can call this API cross-origin.

- Define explicit allowed origins via `AddCors` — never combine `AllowAnyOrigin()` with `AllowCredentials()` (the combination is rejected by browsers and, if it somehow worked, would let any site make credentialed requests)
- Scope policies per environment — permissive/localhost origins in dev, an explicit allow-list of real production domains in prod
- Apply the policy with `app.UseCors("PolicyName")` before `app.UseAuthorization()` in the middleware pipeline

### Minimal API (for lightweight or versioned endpoints)

```csharp
// Endpoints/OrderEndpoints.cs

public static class OrderEndpoints
{
    public static IEndpointRouteBuilder MapOrderEndpoints(
        this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        group.MapGet("/", GetAllOrders)
            .Produces<IReadOnlyList<OrderResponse>>();

        group.MapGet("/{id:int}", GetOrderById)
            .Produces<OrderResponse>()
            .Produces(StatusCodes.Status404NotFound);

        group.MapPost("/", CreateOrder)
            .Produces<OrderResponse>(StatusCodes.Status201Created)
            .ProducesValidationProblem();

        return app;
    }

    private static async Task<IResult> GetAllOrders(
        IOrderService orderService, CancellationToken ct)
    {
        var orders = await orderService.GetAllAsync(ct);
        return Results.Ok(orders);
    }

    private static async Task<IResult> GetOrderById(
        int id, IOrderService orderService, CancellationToken ct)
    {
        var order = await orderService.GetByIdAsync(id, ct);
        return order is null ? Results.NotFound() : Results.Ok(order);
    }

    private static async Task<IResult> CreateOrder(
        CreateOrderRequest request,
        IOrderService orderService,
        CancellationToken ct)
    {
        var order = await orderService.CreateAsync(request, ct);
        return Results.CreatedAtRoute("GetOrderById", new { id = order.Id }, order);
    }
}
```

For full endpoint patterns, action filters, and problem details see `references/controllers.md`.

---

## 4. Dependency Injection & Services

### Service interface

```csharp
// Application/Interfaces/IOrderService.cs

public interface IOrderService
{
    Task<IReadOnlyList<OrderResponse>> GetAllAsync(CancellationToken ct = default);
    Task<OrderResponse?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<OrderResponse> CreateAsync(CreateOrderRequest request, CancellationToken ct = default);
    Task<bool> UpdateStatusAsync(int id, UpdateOrderStatusRequest request, CancellationToken ct = default);
    Task<bool> DeleteAsync(int id, CancellationToken ct = default);
}
```

### Service implementation

```csharp
// Application/Services/OrderService.cs

public sealed class OrderService : IOrderService
{
    private readonly IOrderRepository _repository;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IOrderRepository repository, ILogger<OrderService> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<OrderResponse?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        var order = await _repository.GetByIdAsync(id, ct);
        if (order is null)
        {
            _logger.LogWarning("Order {OrderId} not found", id);
            return null;
        }
        return OrderMapper.ToResponse(order);
    }

    // ...
}
```

### Registration (extension method pattern)

```csharp
// Extensions/ServiceCollectionExtensions.cs

public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddApplicationServices(
        this IServiceCollection services)
    {
        services.AddScoped<IOrderService, OrderService>();
        services.AddScoped<IOrderRepository, OrderRepository>();
        return services;
    }
}
```

**DI rules:**
- Default lifetime is `Scoped` for services touching DbContext; `Transient` for stateless helpers; `Singleton` for caches and typed HTTP clients
- Never inject `IServiceProvider` directly — use constructor injection
- Register services in extension methods, not directly in `Program.cs`
- Always inject interfaces, not concrete types

---

## 5. Data Layer (EF Core)

### Entity

```csharp
// Domain/Entities/Order.cs

public sealed class Order
{
    public int Id { get; init; }
    public string CustomerId { get; init; } = string.Empty;
    public OrderStatus Status { get; private set; } = OrderStatus.Pending;
    public decimal TotalAmount { get; init; }
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; private set; }

    private Order() { }   // EF Core constructor

    public static Order Create(string customerId, decimal totalAmount)
        => new() { CustomerId = customerId, TotalAmount = totalAmount };

    public void UpdateStatus(OrderStatus newStatus)
    {
        Status = newStatus;
        UpdatedAt = DateTime.UtcNow;
    }
}
```

### Entity configuration

```csharp
// Infrastructure/Data/Configurations/OrderConfiguration.cs

public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.CustomerId).HasMaxLength(100).IsRequired();
        builder.Property(o => o.TotalAmount).HasColumnType("decimal(18,2)");
        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(50);
    }
}
```

**Why Repository/Unit-of-Work instead of `DbContext`/`DbSet` directly in services:** it keeps
services testable — a fake `IRepository` can be swapped in for unit tests without spinning up a
real database — centralizes query logic in one place instead of scattering LINQ across services,
and keeps EF Core-specific concerns (change tracking, `DbSet` mechanics) out of the application
layer.

For the repository pattern, unit of work, migrations, and query patterns see `references/data-layer.md`.

---

## 6. Validation & Error Handling

### FluentValidation validator

```csharp
// Application/Validators/CreateOrderRequestValidator.cs

public sealed class CreateOrderRequestValidator
    : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderRequestValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty()
            .MaximumLength(100);

        RuleFor(x => x.TotalAmount)
            .GreaterThan(0)
            .WithMessage("Total amount must be positive.");

        RuleFor(x => x.Items)
            .NotEmpty()
            .WithMessage("Order must have at least one item.");
    }
}
```

### Global exception handling middleware

```csharp
// Middleware/ExceptionHandlingMiddleware.cs

public sealed class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (NotFoundException ex)
        {
            _logger.LogWarning(ex, "Resource not found: {Message}", ex.Message);
            await WriteProblemAsync(context, StatusCodes.Status404NotFound, ex.Message);
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning("Validation failed: {Errors}", ex.Errors);
            await WriteValidationProblemAsync(context, ex);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            await WriteProblemAsync(
                context,
                StatusCodes.Status500InternalServerError,
                "An unexpected error occurred.");
        }
    }

    // WriteProblemAsync / WriteValidationProblemAsync: see references/controllers.md
}
```

**Validation rules:**
- Use FluentValidation for all request validation — never validate manually in controllers
- Register validators with `services.AddValidatorsFromAssemblyContaining<Program>()`
- Return `ProblemDetails` (RFC 7807) for all errors — never return plain strings
- Map domain exceptions to HTTP status codes in middleware, not controllers

---

## 7. Authentication & Authorization

```csharp
// Program.cs — JWT registration

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer           = true,
            ValidateAudience         = true,
            ValidateLifetime         = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer              = builder.Configuration["Jwt:Issuer"],
            ValidAudience            = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey         = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!)),
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly",
        policy => policy.RequireClaim("role", "admin"));
    options.AddPolicy("OrderRead",
        policy => policy.RequireClaim("permission", "orders:read"));
});
```

```csharp
// Controller usage
[Authorize(Policy = "OrderRead")]
[HttpGet]
public async Task<IActionResult> GetAll(CancellationToken ct) { ... }
```

For token generation, refresh tokens, and custom authorization handlers see `references/auth.md`.

---

## 8. HTTP Client (Typed Client Pattern)

```csharp
// Infrastructure/HttpClients/InventoryClient.cs

public sealed class InventoryClient
{
    private readonly HttpClient _http;
    private readonly ILogger<InventoryClient> _logger;

    public InventoryClient(HttpClient http, ILogger<InventoryClient> logger)
    {
        _http = http;
        _logger = logger;
    }

    public async Task<InventoryItem?> GetItemAsync(
        string sku,
        CancellationToken ct = default)
    {
        try
        {
            var response = await _http.GetAsync($"/items/{sku}", ct);
            if (response.StatusCode == HttpStatusCode.NotFound) return null;
            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<InventoryItem>(
                cancellationToken: ct);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Failed to get inventory item {Sku}", sku);
            throw;
        }
    }
}
```

```csharp
// Registration
services.AddHttpClient<InventoryClient>(client =>
{
    client.BaseAddress = new Uri(configuration["InventoryApi:BaseUrl"]!);
    client.DefaultRequestHeaders.Add("Accept", "application/json");
})
.AddStandardResilienceHandler();   // Polly retry + circuit breaker (.NET 8)
```

---

## 9. Configuration (Options Pattern)

```csharp
// Application/Models/JwtOptions.cs

public sealed class JwtOptions
{
    public const string SectionName = "Jwt";

    [Required] public string Key { get; init; } = string.Empty;
    [Required] public string Issuer { get; init; } = string.Empty;
    [Required] public string Audience { get; init; } = string.Empty;
    public int ExpiryMinutes { get; init; } = 60;
}
```

```csharp
// Registration
builder.Services
    .AddOptions<JwtOptions>()
    .BindConfiguration(JwtOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

```csharp
// Consumption
public sealed class TokenService
{
    private readonly JwtOptions _jwt;

    public TokenService(IOptions<JwtOptions> options)
    {
        _jwt = options.Value;
    }
}
```

**Options rules:**
- Always use `IOptions<T>` for singleton services, `IOptionsSnapshot<T>` for scoped services
- Add `[Required]` data annotations and call `.ValidateDataAnnotations().ValidateOnStart()` — fail fast on startup if config is missing
- Never read `IConfiguration` directly in application services — bind to typed options

---

## 10. Logging

```csharp
// Use structured logging with message templates — never string interpolation in log calls

_logger.LogInformation("Order {OrderId} created for customer {CustomerId}",
    order.Id, order.CustomerId);

_logger.LogError(ex, "Failed to process payment for order {OrderId}", orderId);

// Log levels:
// LogTrace    — highly detailed, internal state (dev only)
// LogDebug    — diagnostic, flow tracing
// LogInformation — milestones (order created, user logged in)
// LogWarning  — recoverable anomaly (not found, retry)
// LogError    — unhandled exceptions, data integrity issues
// LogCritical — system failure, startup errors
```

**Logging rules:**
- Use structured logging templates, not `$"..."` interpolation — Serilog/Application Insights reads properties
- Never log passwords, tokens, PII, or connection strings
- Add `Serilog` for production-grade sinks (file, console JSON, Application Insights)

---

## 11. Performance

- Always use `async`/`await` with `CancellationToken` for every I/O operation
- Use `AsNoTracking()` for read-only EF Core queries
- Use projection (`Select(x => new Dto { ... })`) — never return entities from the API
- Cache with `IMemoryCache` or `IDistributedCache` for expensive, infrequently-changing data
- Use `IAsyncEnumerable<T>` for large result sets (`yield return` in repositories)
- Avoid `ToList()` before `Select` — compose the query, then materialise once
- Enable response compression: `services.AddResponseCompression()`
- Use `record` types for immutable DTOs — they are allocation-friendly

---

## 12. Testing

Full patterns in `references/testing.md`.

**Stack:**
- **xUnit** — test runner
- **Moq** — mocking service dependencies in unit tests
- **WebApplicationFactory** — integration tests against the real HTTP pipeline
- **FluentAssertions** — readable assertion syntax

```csharp
// Unit/Services/OrderServiceTests.cs

public sealed class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _repoMock = new();
    private readonly Mock<ILogger<OrderService>> _loggerMock = new();
    private readonly OrderService _sut;

    public OrderServiceTests()
    {
        _sut = new OrderService(_repoMock.Object, _loggerMock.Object);
    }

    [Fact]
    public async Task GetByIdAsync_WhenOrderExists_ReturnsResponse()
    {
        // Arrange
        var order = Order.Create("cust-1", 99.99m);
        _repoMock.Setup(r => r.GetByIdAsync(1, It.IsAny<CancellationToken>()))
                 .ReturnsAsync(order);

        // Act
        var result = await _sut.GetByIdAsync(1);

        // Assert
        result.Should().NotBeNull();
        result!.CustomerId.Should().Be("cust-1");
    }
}
```

---

## 13. Naming Conventions

| Artefact | Convention | Example |
|---|---|---|
| Class | PascalCase | `OrderService` |
| Interface | `I` prefix + PascalCase | `IOrderService` |
| Method | PascalCase, async suffix `Async` | `GetByIdAsync` |
| Private field | `_camelCase` | `_orderService` |
| Constant | PascalCase | `MaxRetryCount` |
| Property | PascalCase | `CustomerId` |
| Parameter / local | camelCase | `orderId` |
| DTO suffix | `Request` / `Response` | `CreateOrderRequest`, `OrderResponse` |
| Entity | Singular noun | `Order`, `Customer` |
| DbContext | `AppDbContext` | — |
| Migration | Timestamp + description | `20240101_AddOrdersTable` |
| Test class | `{Sut}Tests` | `OrderServiceTests` |
| Test method | `MethodName_Scenario_ExpectedResult` | `GetByIdAsync_WhenNotFound_ReturnsNull` |

---

## 14. Code Quality

- Nullable reference types are **mandatory** — target `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>` in every `.csproj`
- **Formatter:** `dotnet format` — run locally before committing; enforce in CI with `dotnet format --verify-no-changes` so unformatted code fails the build
- **Analyzers:** rely on the built-in Roslyn analyzers (on by default in .NET 8 SDK-style projects) plus a repo-root `.editorconfig` for naming/style rules; treat analyzer warnings as errors in CI (`dotnet build /warnaserror`)
- Use `sealed` on classes not designed for inheritance (most classes)
- Use `record` for immutable DTOs and value objects; `record struct` for small value types
- Use `init` properties on DTOs — never public setters
- Never use `dynamic` or `object` as a return type in application code
- Throw domain exceptions (`NotFoundException`, `ConflictException`) from services — never `HttpResponseException`
- Never swallow exceptions with an empty `catch` block
- Dispose `IDisposable` resources with `using` declarations
- Keep methods under 40 lines; extract private helpers freely

---

## 15. Customizing This Skill

### Reference File Lookup

Claude reads this file first; open a reference file only when the task needs deeper detail:

| Topic | File | When to read |
|---|---|---|
| Controllers, Minimal APIs, filters, versioning, Swagger/OpenAPI | `references/controllers.md` | When scaffolding endpoints, adding action filters, API versioning, or Swagger config |
| Repository pattern, Unit of Work, migrations, query patterns | `references/data-layer.md` | When adding a repository, wiring EF Core migrations, or writing paginated/complex queries |
| JWT auth, refresh tokens, policies, resource-based authorization | `references/auth.md` | When issuing tokens, adding a policy, or writing custom/resource-based authorization handlers |
| xUnit, Moq, WebApplicationFactory, test data builders | `references/testing.md` | When writing unit or integration tests for services, controllers, or repositories |
| Full examples | `references/examples.md` | When producing a complete feature or needing a production-ready, end-to-end pattern |

### Project Overrides

```markdown
## Project Overrides — [Project Name]

- Auth: External identity provider (Azure AD B2C) — JWT validation uses OIDC metadata endpoint
- Database: SQL Server 2022 with Dapper for read queries (not EF Core projections)
- Validation: Built-in DataAnnotations only (no FluentValidation installed)
- Logging: Application Insights SDK (no Serilog)
- API style: Minimal APIs only — no controller-based endpoints
```

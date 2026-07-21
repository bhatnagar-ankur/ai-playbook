---
author: Ankur Bhatnagar
---

# Controllers & Endpoints — Full Reference

Detailed patterns for ASP.NET Core controllers, Minimal APIs, filters, problem details,
versioning, and Swagger/OpenAPI.
For the overview see the **Controllers & Minimal APIs** section in `SKILL.md`.

---

## Table of Contents
1. [Controller Base Patterns](#controller-base-patterns)
2. [Route Constraints & Parameter Binding](#route-constraints--parameter-binding)
3. [Problem Details (RFC 7807)](#problem-details-rfc-7807)
4. [Action Filters](#action-filters)
5. [Minimal API — IEndpoint Pattern](#minimal-api--iendpoint-pattern)
6. [API Versioning](#api-versioning)
7. [Swagger / OpenAPI Configuration](#swagger--openapi-configuration)
8. [Global Exception Handling](#global-exception-handling)

---

## Controller Base Patterns

### Full CRUD controller template

```csharp
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[Produces("application/json")]
[Authorize]
public sealed class OrdersController : ControllerBase
{
    private readonly IOrderService _service;

    public OrdersController(IOrderService service) => _service = service;

    /// <summary>Returns all orders for the authenticated user.</summary>
    [HttpGet]
    [ProducesResponseType<PagedResponse<OrderResponse>>(StatusCodes.Status200OK)]
    public async Task<IActionResult> GetAll(
        [FromQuery] OrderFilterRequest filter,
        CancellationToken ct)
    {
        var result = await _service.GetAllAsync(filter, ct);
        return Ok(result);
    }

    [HttpGet("{id:int}", Name = "GetOrderById")]
    [ProducesResponseType<OrderResponse>(StatusCodes.Status200OK)]
    [ProducesResponseType<ProblemDetails>(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(int id, CancellationToken ct)
    {
        var order = await _service.GetByIdAsync(id, ct);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpPost]
    [ProducesResponseType<OrderResponse>(StatusCodes.Status201Created)]
    [ProducesResponseType<ValidationProblemDetails>(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create(
        [FromBody] CreateOrderRequest request,
        CancellationToken ct)
    {
        var order = await _service.CreateAsync(request, ct);
        return CreatedAtRoute("GetOrderById", new { id = order.Id }, order);
    }

    [HttpPut("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType<ValidationProblemDetails>(StatusCodes.Status400BadRequest)]
    [ProducesResponseType<ProblemDetails>(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Update(
        int id,
        [FromBody] UpdateOrderRequest request,
        CancellationToken ct)
    {
        var success = await _service.UpdateAsync(id, request, ct);
        return success ? NoContent() : NotFound();
    }

    [HttpDelete("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType<ProblemDetails>(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id, CancellationToken ct)
    {
        var success = await _service.DeleteAsync(id, ct);
        return success ? NoContent() : NotFound();
    }
}
```

---

## Route Constraints & Parameter Binding

```csharp
// Route constraint examples
[HttpGet("{id:int}")]            // int only
[HttpGet("{slug:alpha}")]        // letters only
[HttpGet("{code:length(6)}")]    // exactly 6 chars
[HttpGet("{date:datetime}")]     // DateTime

// Parameter binding attributes
public IActionResult Search(
    [FromQuery]  string term,         // ?term=foo
    [FromRoute]  int    id,           // /api/orders/42
    [FromBody]   CreateOrderRequest r,// JSON body
    [FromHeader] string correlationId,// X-Correlation-ID header
    [FromForm]   IFormFile file)      // multipart/form-data
```

### Paged query parameters

```csharp
public sealed record OrderFilterRequest
{
    [Range(1, int.MaxValue)]
    public int Page { get; init; } = 1;

    [Range(1, 100)]
    public int PageSize { get; init; } = 20;

    public string? Status { get; init; }
    public string? CustomerId { get; init; }
    public DateTime? From { get; init; }
    public DateTime? To { get; init; }
}

public sealed record PagedResponse<T>
{
    public IReadOnlyList<T> Items { get; init; } = [];
    public int TotalCount { get; init; }
    public int Page { get; init; }
    public int PageSize { get; init; }
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
}
```

---

## Problem Details (RFC 7807)

```csharp
// Program.cs
builder.Services.AddProblemDetails();

// Custom ProblemDetails factory extension
public static class ProblemDetailsExtensions
{
    public static ProblemDetails NotFound(string detail)
        => new()
        {
            Status = StatusCodes.Status404NotFound,
            Title  = "Resource Not Found",
            Detail = detail,
        };

    public static ValidationProblemDetails ValidationFailed(
        IDictionary<string, string[]> errors)
        => new(errors)
        {
            Status = StatusCodes.Status400BadRequest,
            Title  = "Validation Failed",
        };
}

// In controllers: return Problem() helpers
return NotFound(ProblemDetailsExtensions.NotFound($"Order {id} not found."));
```

---

## Action Filters

### Request logging filter

```csharp
public sealed class RequestLoggingFilter : IActionFilter
{
    private readonly ILogger<RequestLoggingFilter> _logger;

    public RequestLoggingFilter(ILogger<RequestLoggingFilter> logger)
        => _logger = logger;

    public void OnActionExecuting(ActionExecutingContext context)
    {
        _logger.LogInformation(
            "Executing {Controller}.{Action} with args {Arguments}",
            context.Controller.GetType().Name,
            context.ActionDescriptor.DisplayName,
            context.ActionArguments);
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        if (context.Exception is not null)
            _logger.LogError(context.Exception, "Action threw an exception");
    }
}

// Registration — apply globally
builder.Services.AddControllers(options =>
{
    options.Filters.Add<RequestLoggingFilter>();
});
```

### Idempotency filter (POST safety)

```csharp
public sealed class IdempotencyFilter : IAsyncActionFilter
{
    private readonly IDistributedCache _cache;

    public IdempotencyFilter(IDistributedCache cache) => _cache = cache;

    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)
    {
        var key = context.HttpContext.Request.Headers["Idempotency-Key"].ToString();
        if (string.IsNullOrEmpty(key))
        {
            await next();
            return;
        }

        var cached = await _cache.GetStringAsync(key);
        if (cached is not null)
        {
            context.Result = new ContentResult
            {
                Content     = cached,
                ContentType = "application/json",
                StatusCode  = StatusCodes.Status200OK,
            };
            return;
        }

        var executed = await next();

        if (executed.Result is ObjectResult { Value: not null } result)
        {
            var json = JsonSerializer.Serialize(result.Value);
            await _cache.SetStringAsync(
                key, json,
                new DistributedCacheEntryOptions
                    { AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24) });
        }
    }
}
```

---

## Minimal API — IEndpoint Pattern

```csharp
// Endpoints/IEndpoint.cs
public interface IEndpoint
{
    void MapEndpoints(IEndpointRouteBuilder app);
}

// Endpoints/OrderEndpoints.cs
public sealed class OrderEndpoints : IEndpoint
{
    public void MapEndpoints(IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/orders")
            .WithTags("Orders")
            .RequireAuthorization()
            .WithOpenApi();

        group.MapGet("/",       GetAll);
        group.MapGet("/{id:int}", GetById);
        group.MapPost("/",      Create);
        group.MapPatch("/{id:int}/status", UpdateStatus);
        group.MapDelete("/{id:int}", Delete);
    }

    private static async Task<IResult> GetAll(
        IOrderService svc, CancellationToken ct)
        => Results.Ok(await svc.GetAllAsync(ct));

    private static async Task<IResult> GetById(
        int id, IOrderService svc, CancellationToken ct)
    {
        var order = await svc.GetByIdAsync(id, ct);
        return order is null ? Results.NotFound() : Results.Ok(order);
    }

    private static async Task<IResult> Create(
        CreateOrderRequest request, IOrderService svc, CancellationToken ct)
    {
        var order = await svc.CreateAsync(request, ct);
        return Results.CreatedAtRoute("GetOrderById", new { id = order.Id }, order);
    }

    private static async Task<IResult> UpdateStatus(
        int id, UpdateOrderStatusRequest request,
        IOrderService svc, CancellationToken ct)
    {
        var ok = await svc.UpdateStatusAsync(id, request, ct);
        return ok ? Results.NoContent() : Results.NotFound();
    }

    private static async Task<IResult> Delete(
        int id, IOrderService svc, CancellationToken ct)
    {
        var ok = await svc.DeleteAsync(id, ct);
        return ok ? Results.NoContent() : Results.NotFound();
    }
}

// Auto-registration in Program.cs
var endpointTypes = typeof(Program).Assembly
    .GetTypes()
    .Where(t => t.IsAssignableTo(typeof(IEndpoint)) && !t.IsInterface);

foreach (var type in endpointTypes)
{
    var endpoint = (IEndpoint)Activator.CreateInstance(type)!;
    endpoint.MapEndpoints(app);
}
```

---

## API Versioning

```xml
<!-- .csproj -->
<PackageReference Include="Asp.Versioning.Mvc" Version="8.*" />
<PackageReference Include="Asp.Versioning.Mvc.ApiExplorer" Version="8.*" />
```

```csharp
// Program.cs
builder.Services
    .AddApiVersioning(options =>
    {
        options.DefaultApiVersion         = new ApiVersion(1);
        options.AssumeDefaultVersionWhenUnspecified = true;
        options.ReportApiVersions         = true;
        options.ApiVersionReader          = ApiVersionReader.Combine(
            new UrlSegmentApiVersionReader(),
            new HeaderApiVersionReader("X-Api-Version"));
    })
    .AddApiExplorer(options =>
    {
        options.GroupNameFormat           = "'v'VVV";
        options.SubstituteApiVersionInUrl = true;
    });

// Controller
[ApiController]
[ApiVersion(1)]
[ApiVersion(2)]
[Route("api/v{version:apiVersion}/orders")]
public sealed class OrdersController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion(1)]
    public IActionResult GetV1() => Ok("v1");

    [HttpGet]
    [MapToApiVersion(2)]
    public IActionResult GetV2() => Ok("v2 — enhanced");
}
```

---

## Swagger / OpenAPI Configuration

```csharp
// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title   = "YourApp API",
        Version = "v1",
    });

    // JWT bearer auth in Swagger UI
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name         = "Authorization",
        Type         = SecuritySchemeType.Http,
        Scheme       = "Bearer",
        BearerFormat = "JWT",
        In           = ParameterLocation.Header,
        Description  = "Enter: Bearer {token}",
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id   = "Bearer",
                },
            },
            Array.Empty<string>()
        },
    });

    // Include XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);
});

// Middleware
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "v1");
        options.RoutePrefix = string.Empty;  // Serve at root
    });
}
```

---

## Global Exception Handling

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
        _next   = next;
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
            await WriteProblemAsync(context, StatusCodes.Status404NotFound,
                "Resource Not Found", ex.Message);
        }
        catch (ConflictException ex)
        {
            await WriteProblemAsync(context, StatusCodes.Status409Conflict,
                "Conflict", ex.Message);
        }
        catch (UnauthorizedException ex)
        {
            await WriteProblemAsync(context, StatusCodes.Status401Unauthorized,
                "Unauthorised", ex.Message);
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning("Validation failed: {Errors}", ex.Errors);
            var errors = ex.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(
                    g => g.Key,
                    g => g.Select(e => e.ErrorMessage).ToArray());

            context.Response.StatusCode  = StatusCodes.Status400BadRequest;
            context.Response.ContentType = "application/problem+json";
            var problem = new ValidationProblemDetails(errors)
            {
                Status = StatusCodes.Status400BadRequest,
                Title  = "Validation Failed",
            };
            await context.Response.WriteAsJsonAsync(problem);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception for {Method} {Path}",
                context.Request.Method, context.Request.Path);
            await WriteProblemAsync(context,
                StatusCodes.Status500InternalServerError,
                "Internal Server Error",
                "An unexpected error occurred. Please try again later.");
        }
    }

    private static async Task WriteProblemAsync(
        HttpContext context, int statusCode, string title, string detail)
    {
        context.Response.StatusCode  = statusCode;
        context.Response.ContentType = "application/problem+json";
        var problem = new ProblemDetails
        {
            Status = statusCode,
            Title  = title,
            Detail = detail,
        };
        await context.Response.WriteAsJsonAsync(problem);
    }
}

// Domain exceptions
public sealed class NotFoundException    : Exception { public NotFoundException(string msg) : base(msg) {} }
public sealed class ConflictException    : Exception { public ConflictException(string msg) : base(msg) {} }
public sealed class UnauthorizedException : Exception { public UnauthorizedException(string msg) : base(msg) {} }

// Registration (before UseRouting)
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

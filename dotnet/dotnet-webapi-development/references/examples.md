---
author: Ankur Bhatnagar
---

# Examples — Full Reference

Complete worked examples for ASP.NET Core Web API (.NET 8 LTS).
Each example is self-contained and production-ready.

---

## Table of Contents
1. [Project Bootstrap (Program.cs)](#example-1-project-bootstrap)
2. [Full CRUD Feature — Orders](#example-2-full-crud-feature--orders)
3. [Typed HTTP Client Integration](#example-3-typed-http-client-integration)
4. [Background Service (Worker)](#example-4-background-service-worker)

---

## Example 1: Project Bootstrap

```csharp
// Program.cs — composition root

var builder = WebApplication.CreateBuilder(args);

// ── Options ────────────────────────────────────────────────────────────
builder.Services
    .AddOptions<JwtOptions>()
    .BindConfiguration(JwtOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();

// ── Infrastructure ─────────────────────────────────────────────────────
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Default"),
        sql => sql.EnableRetryOnFailure(maxRetryCount: 5,
            maxRetryDelay: TimeSpan.FromSeconds(30), errorNumbersToAdd: null)));

// ── Application services ────────────────────────────────────────────────
builder.Services.AddApplicationServices();   // IServiceCollection extension

// ── Auth ────────────────────────────────────────────────────────────────
builder.Services.AddJwtAuthentication(builder.Configuration);
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("OrderRead",
        p => p.RequireClaim("permission", "orders:read"));
    options.AddPolicy("OrderWrite",
        p => p.RequireClaim("permission", "orders:write"));
});

// ── HTTP ────────────────────────────────────────────────────────────────
builder.Services.AddHttpContextAccessor();
builder.Services.AddHttpClient<InventoryClient>(client =>
{
    client.BaseAddress = new Uri(
        builder.Configuration["InventoryApi:BaseUrl"]!);
}).AddStandardResilienceHandler();

// ── API ─────────────────────────────────────────────────────────────────
builder.Services
    .AddControllers(o => o.Filters.Add<RequestLoggingFilter>())
    .AddJsonOptions(o =>
    {
        o.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        o.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
    });

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(SwaggerConfig.Configure);
builder.Services.AddResponseCompression();
builder.Services.AddProblemDetails();
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// ── Logging ─────────────────────────────────────────────────────────────
builder.Host.UseSerilog((ctx, cfg) =>
    cfg.ReadFrom.Configuration(ctx.Configuration));

// ─────────────────────────────────────────────────────────────────────────
var app = builder.Build();

// Apply migrations in dev
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    await scope.ServiceProvider
               .GetRequiredService<AppDbContext>()
               .Database
               .MigrateAsync();

    app.UseSwagger();
    app.UseSwaggerUI(o => o.RoutePrefix = string.Empty);
}

app.UseResponseCompression();
app.UseMiddleware<ExceptionHandlingMiddleware>();
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

await app.RunAsync();

// Expose Program for WebApplicationFactory in tests
public partial class Program { }
```

---

## Example 2: Full CRUD Feature — Orders

### Domain entity

```csharp
// Domain/Entities/Order.cs

public sealed class Order
{
    public int          Id          { get; init; }
    public string       CustomerId  { get; init; } = string.Empty;
    public OrderStatus  Status      { get; private set; } = OrderStatus.Pending;
    public decimal      TotalAmount { get; init; }
    public DateTime     CreatedAt   { get; init; } = DateTime.UtcNow;
    public DateTime?    UpdatedAt   { get; private set; }

    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    private readonly List<OrderItem> _items = [];

    private Order() { }

    public static Order Create(string customerId, decimal totalAmount)
        => new() { CustomerId = customerId, TotalAmount = totalAmount };

    public void AddItem(string sku, int quantity, decimal unitPrice)
        => _items.Add(OrderItem.Create(sku, quantity, unitPrice));

    public void UpdateStatus(OrderStatus newStatus)
    {
        Status    = newStatus;
        UpdatedAt = DateTime.UtcNow;
    }
}
```

### DTOs

```csharp
// Application/Models/Orders/CreateOrderRequest.cs

public sealed record CreateOrderRequest
{
    [Required]
    [MaxLength(100)]
    public string CustomerId { get; init; } = string.Empty;

    [Range(0.01, double.MaxValue, ErrorMessage = "Total amount must be positive.")]
    public decimal TotalAmount { get; init; }

    [MinLength(1, ErrorMessage = "Order must have at least one item.")]
    public IReadOnlyList<OrderItemRequest> Items { get; init; } = [];
}

public sealed record OrderItemRequest
{
    [Required] public string Sku      { get; init; } = string.Empty;
    [Range(1, 1000)] public int Quantity { get; init; }
}

public sealed record OrderResponse
{
    public int                          Id          { get; init; }
    public string                       CustomerId  { get; init; } = string.Empty;
    public string                       Status      { get; init; } = string.Empty;
    public decimal                      TotalAmount { get; init; }
    public DateTime                     CreatedAt   { get; init; }
    public IReadOnlyList<OrderItemResponse> Items  { get; init; } = [];
}
```

### Mapper

```csharp
// Application/Mappers/OrderMapper.cs

public static class OrderMapper
{
    public static OrderResponse ToResponse(Order order)
        => new()
        {
            Id          = order.Id,
            CustomerId  = order.CustomerId,
            Status      = order.Status.ToString(),
            TotalAmount = order.TotalAmount,
            CreatedAt   = order.CreatedAt,
            Items       = order.Items.Select(ToItemResponse).ToList(),
        };

    public static OrderItemResponse ToItemResponse(OrderItem item)
        => new()
        {
            Sku       = item.Sku,
            Quantity  = item.Quantity,
            UnitPrice = item.UnitPrice,
        };
}
```

### Service

```csharp
// Application/Services/OrderService.cs

public sealed class OrderService : IOrderService
{
    // IOrderRepository/IRepository<T> only expose data access (Add/Get/Update/Remove) —
    // persistence is triggered through IUnitOfWork.SaveChangesAsync, not the repository
    // itself. See references/data-layer.md for the Unit-of-Work definition.
    private readonly IUnitOfWork _uow;
    private readonly ICurrentUser _user;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        IUnitOfWork uow,
        ICurrentUser user,
        ILogger<OrderService> logger)
    {
        _uow    = uow;
        _user   = user;
        _logger = logger;
    }

    public async Task<IReadOnlyList<OrderResponse>> GetAllAsync(
        CancellationToken ct = default)
    {
        var orders = await _uow.Orders.GetByCustomerIdAsync(_user.Id, ct);
        return orders.Select(OrderMapper.ToResponse).ToList();
    }

    public async Task<OrderResponse?> GetByIdAsync(
        int id, CancellationToken ct = default)
    {
        var order = await _uow.Orders.GetByIdAsync(id, ct);
        if (order is null) return null;
        if (order.CustomerId != _user.Id && _user.Role != "admin")
            throw new UnauthorizedException("You do not own this order.");
        return OrderMapper.ToResponse(order);
    }

    public async Task<OrderResponse> CreateAsync(
        CreateOrderRequest request, CancellationToken ct = default)
    {
        var order = Order.Create(request.CustomerId, request.TotalAmount);
        foreach (var item in request.Items)
            order.AddItem(item.Sku, item.Quantity, 0m);   // Price fetched from inventory

        await _uow.Orders.AddAsync(order, ct);
        await _uow.SaveChangesAsync(ct);

        _logger.LogInformation("Order {OrderId} created for {CustomerId}",
            order.Id, order.CustomerId);

        return OrderMapper.ToResponse(order);
    }

    public async Task<bool> UpdateStatusAsync(
        int id,
        UpdateOrderStatusRequest request,
        CancellationToken ct = default)
    {
        var order = await _uow.Orders.GetByIdAsync(id, ct);
        if (order is null) return false;

        if (!Enum.TryParse<OrderStatus>(request.Status, out var newStatus))
            throw new ValidationException("Invalid order status.");

        order.UpdateStatus(newStatus);
        _uow.Orders.Update(order);
        await _uow.SaveChangesAsync(ct);
        return true;
    }

    public async Task<bool> DeleteAsync(int id, CancellationToken ct = default)
    {
        var order = await _uow.Orders.GetByIdAsync(id, ct);
        if (order is null) return false;
        _uow.Orders.Remove(order);
        await _uow.SaveChangesAsync(ct);
        return true;
    }
}
```

### Controller

```csharp
// Controllers/OrdersController.cs

[ApiController]
[Route("api/orders")]
[Produces("application/json")]
[Authorize]
public sealed class OrdersController : ControllerBase
{
    private readonly IOrderService _service;

    public OrdersController(IOrderService service) => _service = service;

    [HttpGet]
    [Authorize(Policy = "OrderRead")]
    [ProducesResponseType<IReadOnlyList<OrderResponse>>(StatusCodes.Status200OK)]
    public async Task<IActionResult> GetAll(CancellationToken ct)
        => Ok(await _service.GetAllAsync(ct));

    [HttpGet("{id:int}", Name = "GetOrderById")]
    [Authorize(Policy = "OrderRead")]
    [ProducesResponseType<OrderResponse>(StatusCodes.Status200OK)]
    [ProducesResponseType<ProblemDetails>(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(int id, CancellationToken ct)
    {
        var order = await _service.GetByIdAsync(id, ct);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpPost]
    [Authorize(Policy = "OrderWrite")]
    [ProducesResponseType<OrderResponse>(StatusCodes.Status201Created)]
    [ProducesResponseType<ValidationProblemDetails>(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create(
        [FromBody] CreateOrderRequest request, CancellationToken ct)
    {
        var order = await _service.CreateAsync(request, ct);
        return CreatedAtRoute("GetOrderById", new { id = order.Id }, order);
    }

    [HttpPatch("{id:int}/status")]
    [Authorize(Policy = "OrderWrite")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType<ProblemDetails>(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> UpdateStatus(
        int id, [FromBody] UpdateOrderStatusRequest request, CancellationToken ct)
    {
        var success = await _service.UpdateStatusAsync(id, request, ct);
        return success ? NoContent() : NotFound();
    }

    [HttpDelete("{id:int}")]
    [Authorize(Policy = "OrderWrite")]
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

## Example 3: Typed HTTP Client Integration

```csharp
// Infrastructure/HttpClients/InventoryClient.cs

public sealed class InventoryClient
{
    private readonly HttpClient _http;
    private readonly ILogger<InventoryClient> _logger;

    public InventoryClient(HttpClient http, ILogger<InventoryClient> logger)
    {
        _http   = http;
        _logger = logger;
    }

    public async Task<InventoryItem?> GetItemAsync(
        string sku, CancellationToken ct = default)
    {
        try
        {
            var response = await _http.GetAsync($"/api/items/{sku}", ct);

            if (response.StatusCode == HttpStatusCode.NotFound)
                return null;

            response.EnsureSuccessStatusCode();

            return await response.Content.ReadFromJsonAsync<InventoryItem>(
                cancellationToken: ct);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Inventory API unavailable for SKU {Sku}", sku);
            throw;
        }
    }

    public async Task<bool> ReserveStockAsync(
        string sku, int quantity, CancellationToken ct = default)
    {
        var payload  = new ReserveStockRequest { Sku = sku, Quantity = quantity };
        var response = await _http.PostAsJsonAsync("/api/inventory/reserve", payload, ct);
        return response.IsSuccessStatusCode;
    }
}
```

```csharp
// Infrastructure/HttpClients/Models/InventoryItem.cs

public sealed record InventoryItem
{
    public string Sku         { get; init; } = string.Empty;
    public string Name        { get; init; } = string.Empty;
    public int    StockLevel  { get; init; }
    public decimal UnitPrice  { get; init; }
    public bool   IsAvailable => StockLevel > 0;
}
```

---

## Example 4: Background Service (Worker)

```csharp
// Infrastructure/Workers/OrderExpiryWorker.cs

public sealed class OrderExpiryWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OrderExpiryWorker> _logger;
    private static readonly TimeSpan Interval = TimeSpan.FromHours(1);

    public OrderExpiryWorker(
        IServiceScopeFactory scopeFactory,
        ILogger<OrderExpiryWorker> logger)
    {
        _scopeFactory = scopeFactory;
        _logger       = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("OrderExpiryWorker started");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ExpireOldOrdersAsync(stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "OrderExpiryWorker encountered an error");
            }

            await Task.Delay(Interval, stoppingToken);
        }
    }

    private async Task ExpireOldOrdersAsync(CancellationToken ct)
    {
        // Always use a scope for DbContext — never inject as singleton
        using var scope  = _scopeFactory.CreateScope();
        var db           = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var cutoff        = DateTime.UtcNow.AddDays(-7);

        var expiring = await db.Orders
            .Where(o => o.Status == OrderStatus.Pending && o.CreatedAt < cutoff)
            .ToListAsync(ct);

        foreach (var order in expiring)
            order.UpdateStatus(OrderStatus.Expired);

        if (expiring.Count > 0)
        {
            await db.SaveChangesAsync(ct);
            _logger.LogInformation("Expired {Count} stale orders", expiring.Count);
        }
    }
}
```

```csharp
// Registration
builder.Services.AddHostedService<OrderExpiryWorker>();
```

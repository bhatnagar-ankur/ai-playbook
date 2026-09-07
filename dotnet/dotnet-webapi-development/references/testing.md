---
author: Ankur Bhatnagar
---

# Testing — Full Reference

xUnit unit tests, Moq mocking, WebApplicationFactory integration tests,
and test project setup for ASP.NET Core 8.
For the overview see the **Testing** section in `SKILL.md`.

---

## Table of Contents
1. [Test Project Setup](#test-project-setup)
2. [Unit Tests — Services](#unit-tests--services)
3. [Unit Tests — Validators](#unit-tests--validators)
4. [Mocking with Moq](#mocking-with-moq)
5. [Integration Tests — WebApplicationFactory](#integration-tests--webapplicationfactory)
6. [Database Integration Tests (Sqlite In-Memory)](#database-integration-tests-sqlite-in-memory)
7. [Auth Helpers](#auth-helpers)
8. [Test Data Builders](#test-data-builders)

---

## Test Project Setup

```xml
<!-- YourApp.Tests.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="xunit"                          Version="2.*" />
    <PackageReference Include="xunit.runner.visualstudio"      Version="2.*" />
    <PackageReference Include="Microsoft.NET.Test.Sdk"         Version="17.*" />
    <PackageReference Include="Moq"                            Version="4.*" />
    <PackageReference Include="FluentAssertions"               Version="6.*" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.*" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="8.*" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\YourApp.Api\YourApp.Api.csproj" />
    <ProjectReference Include="..\YourApp.Application\YourApp.Application.csproj" />
    <ProjectReference Include="..\YourApp.Infrastructure\YourApp.Infrastructure.csproj" />
  </ItemGroup>
</Project>
```

---

## Unit Tests — Services

```csharp
// Unit/Services/OrderServiceTests.cs

public sealed class OrderServiceTests
{
    // SaveChangesAsync lives on IUnitOfWork, not IOrderRepository — the repository mock
    // only handles data access (Get/Add/Update/Remove); persistence is mocked separately.
    private readonly Mock<IOrderRepository> _repoMock   = new(MockBehavior.Strict);
    private readonly Mock<IUnitOfWork> _uowMock         = new();
    private readonly Mock<ILogger<OrderService>> _logMock = new();
    private readonly OrderService _sut;

    public OrderServiceTests()
    {
        _uowMock.Setup(u => u.Orders).Returns(_repoMock.Object);
        _sut = new OrderService(_uowMock.Object, _logMock.Object);
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
        result.TotalAmount.Should().Be(99.99m);
        _repoMock.Verify(r => r.GetByIdAsync(1, It.IsAny<CancellationToken>()), Times.Once);
    }

    [Fact]
    public async Task GetByIdAsync_WhenOrderNotFound_ReturnsNull()
    {
        // Arrange
        _repoMock.Setup(r => r.GetByIdAsync(99, It.IsAny<CancellationToken>()))
                 .ReturnsAsync((Order?)null);

        // Act
        var result = await _sut.GetByIdAsync(99);

        // Assert
        result.Should().BeNull();
    }

    [Fact]
    public async Task CreateAsync_ValidRequest_ReturnsCreatedOrder()
    {
        // Arrange
        var request = new CreateOrderRequest { CustomerId = "cust-1", TotalAmount = 50m };
        _repoMock.Setup(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
                 .Returns(Task.CompletedTask);
        _uowMock.Setup(u => u.SaveChangesAsync(It.IsAny<CancellationToken>()))
                 .ReturnsAsync(1);

        // Act
        var result = await _sut.CreateAsync(request, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result.CustomerId.Should().Be("cust-1");
        result.TotalAmount.Should().Be(50m);
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    [InlineData(-100)]
    public async Task CreateAsync_WithInvalidAmount_ThrowsValidationException(decimal amount)
    {
        var request = new CreateOrderRequest { CustomerId = "cust-1", TotalAmount = amount };

        var act = () => _sut.CreateAsync(request, CancellationToken.None);

        await act.Should().ThrowAsync<ValidationException>();
    }
}
```

---

## Unit Tests — Validators

```csharp
// Unit/Validators/CreateOrderRequestValidatorTests.cs

public sealed class CreateOrderRequestValidatorTests
{
    private readonly CreateOrderRequestValidator _validator = new();

    [Fact]
    public void Validate_ValidRequest_PassesValidation()
    {
        var request = new CreateOrderRequest
        {
            CustomerId  = "cust-1",
            TotalAmount = 99.99m,
            Items       = [new OrderItemRequest { Sku = "SKU-1", Quantity = 1 }],
        };

        var result = _validator.Validate(request);

        result.IsValid.Should().BeTrue();
    }

    [Fact]
    public void Validate_MissingCustomerId_FailsWithError()
    {
        var request = new CreateOrderRequest
        {
            CustomerId  = string.Empty,
            TotalAmount = 10m,
            Items       = [new OrderItemRequest { Sku = "SKU-1", Quantity = 1 }],
        };

        var result = _validator.Validate(request);

        result.IsValid.Should().BeFalse();
        result.Errors.Should().ContainSingle(e =>
            e.PropertyName == nameof(CreateOrderRequest.CustomerId));
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-1)]
    public void Validate_ZeroOrNegativeAmount_FailsWithError(decimal amount)
    {
        var request = new CreateOrderRequest
        {
            CustomerId  = "cust-1",
            TotalAmount = amount,
            Items       = [new OrderItemRequest { Sku = "SKU-1", Quantity = 1 }],
        };

        var result = _validator.Validate(request);

        result.IsValid.Should().BeFalse();
        result.Errors.Should().ContainSingle(e =>
            e.PropertyName == nameof(CreateOrderRequest.TotalAmount));
    }
}
```

---

## Mocking with Moq

```csharp
// Mock behaviour modes
var strict  = new Mock<IOrderRepository>(MockBehavior.Strict);   // Throws on unexpected calls
var loose   = new Mock<IOrderRepository>(MockBehavior.Loose);    // Returns default on unexpected
var default_ = new Mock<IOrderRepository>();                     // Loose by default

// Setup return values
mock.Setup(r => r.GetByIdAsync(1, It.IsAny<CancellationToken>()))
    .ReturnsAsync(order);

// Setup with callback
mock.Setup(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
    .Callback<Order, CancellationToken>((entity, _) => entity.GetType())
    .Returns(Task.CompletedTask);

// Setup to throw
mock.Setup(r => r.GetByIdAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()))
    .ThrowsAsync(new DbUpdateException("Connection failed"));

// Verify calls
mock.Verify(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()), Times.Once);
mock.Verify(r => r.GetByIdAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()), Times.Never);
mock.VerifyAll();  // Verify all setups were called

// Capture argument for assertion
Order? captured = null;
mock.Setup(r => r.AddAsync(It.IsAny<Order>(), It.IsAny<CancellationToken>()))
    .Callback<Order, CancellationToken>((o, _) => captured = o)
    .Returns(Task.CompletedTask);

// After act:
captured.Should().NotBeNull();
captured!.CustomerId.Should().Be("expected");
```

---

## Integration Tests — WebApplicationFactory

```csharp
// Integration/ApiFactory.cs

public sealed class ApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove real DbContext
            var descriptor = services.SingleOrDefault(d =>
                d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor is not null)
                services.Remove(descriptor);

            // Add test DbContext pointing at container
            services.AddDbContext<AppDbContext>(options =>
                options.UseNpgsql(_db.GetConnectionString()));
        });
    }

    public async Task InitializeAsync()
    {
        await _db.StartAsync();

        // Migrate on startup
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public new async Task DisposeAsync()
    {
        await _db.DisposeAsync();
    }
}
```

```csharp
// Integration/Orders/GetOrdersTests.cs

public sealed class GetOrdersTests : IClassFixture<ApiFactory>
{
    private readonly HttpClient _client;
    private readonly ApiFactory _factory;

    public GetOrdersTests(ApiFactory factory)
    {
        _factory = factory;
        _client  = factory.CreateClient();
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", AuthHelper.GenerateToken("user-1"));
    }

    [Fact]
    public async Task GetAll_WhenAuthenticated_Returns200WithOrders()
    {
        // Arrange — seed data
        await _factory.SeedAsync(async db =>
        {
            db.Orders.Add(Order.Create("user-1", 50m));
            await db.SaveChangesAsync();
        });

        // Act
        var response = await _client.GetAsync("/api/orders");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);

        var body = await response.Content
            .ReadFromJsonAsync<List<OrderResponse>>();

        body.Should().NotBeNull();
        body!.Should().HaveCountGreaterThanOrEqualTo(1);
    }

    [Fact]
    public async Task GetById_WhenOrderNotFound_Returns404()
    {
        var response = await _client.GetAsync("/api/orders/99999");

        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task Create_ValidRequest_Returns201()
    {
        var request = new CreateOrderRequest
        {
            CustomerId  = "user-1",
            TotalAmount = 75m,
            Items = [new OrderItemRequest { Sku = "SKU-1", Quantity = 2 }],
        };

        var response = await _client.PostAsJsonAsync("/api/orders", request);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
    }

    [Fact]
    public async Task Create_InvalidRequest_Returns400WithValidationErrors()
    {
        var request = new CreateOrderRequest
        {
            CustomerId  = string.Empty,  // Invalid
            TotalAmount = -1m,           // Invalid
        };

        var response = await _client.PostAsJsonAsync("/api/orders", request);

        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);

        var problem = await response.Content
            .ReadFromJsonAsync<ValidationProblemDetails>();

        problem!.Errors.Should().ContainKey(nameof(CreateOrderRequest.CustomerId));
        problem.Errors.Should().ContainKey(nameof(CreateOrderRequest.TotalAmount));
    }

    [Fact]
    public async Task GetAll_WhenUnauthenticated_Returns401()
    {
        var client   = _factory.CreateClient();  // No auth header
        var response = await client.GetAsync("/api/orders");

        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }
}
```

---

## Database Integration Tests (Sqlite In-Memory)

```csharp
// Use Sqlite for lightweight integration tests without a container

public abstract class DbTestBase : IDisposable
{
    protected readonly AppDbContext Context;

    protected DbTestBase()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite("Data Source=:memory:")
            .Options;

        Context = new AppDbContext(options);
        Context.Database.EnsureCreated();
    }

    public void Dispose() => Context.Dispose();
}

public sealed class OrderRepositoryTests : DbTestBase
{
    private readonly OrderRepository _sut;

    public OrderRepositoryTests()
        => _sut = new OrderRepository(Context);

    [Fact]
    public async Task GetByIdAsync_WhenSaved_ReturnsEntity()
    {
        // Arrange
        var order = Order.Create("cust-1", 50m);
        Context.Orders.Add(order);
        await Context.SaveChangesAsync();

        // Act
        var found = await _sut.GetByIdAsync(order.Id);

        // Assert
        found.Should().NotBeNull();
        found!.CustomerId.Should().Be("cust-1");
    }
}
```

---

## Auth Helpers

```csharp
// Integration/Helpers/AuthHelper.cs

public static class AuthHelper
{
    private const string TestKey    = "test-secret-key-32-chars-minimum!";
    private const string TestIssuer = "test-issuer";
    private const string TestAudience = "test-audience";

    public static string GenerateToken(
        string userId,
        string role        = "user",
        string[]? perms    = null)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub,   userId),
            new(JwtRegisteredClaimNames.Email, $"{userId}@test.com"),
            new("role",                        role),
        };

        foreach (var perm in perms ?? [])
            claims.Add(new Claim("permission", perm));

        var key   = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(TestKey));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        var token = new JwtSecurityToken(
            issuer:             TestIssuer,
            audience:           TestAudience,
            claims:             claims,
            expires:            DateTime.UtcNow.AddHours(1),
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

## Test Data Builders

```csharp
// Tests/Builders/OrderBuilder.cs

public sealed class OrderBuilder
{
    private string  _customerId  = "cust-default";
    private decimal _total       = 99.99m;
    private OrderStatus _status  = OrderStatus.Pending;

    public OrderBuilder WithCustomerId(string id)   { _customerId = id;  return this; }
    public OrderBuilder WithTotal(decimal amount)   { _total = amount;   return this; }
    public OrderBuilder WithStatus(OrderStatus s)   { _status = s;       return this; }

    public Order Build()
    {
        var order = Order.Create(_customerId, _total);
        if (_status != OrderStatus.Pending)
            order.UpdateStatus(_status);
        return order;
    }

    public static Order Default() => new OrderBuilder().Build();
}

// Usage in tests
var order = new OrderBuilder()
    .WithCustomerId("user-42")
    .WithTotal(250m)
    .WithStatus(OrderStatus.Shipped)
    .Build();

var defaultOrder = OrderBuilder.Default();
```

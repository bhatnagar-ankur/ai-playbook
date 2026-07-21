---
author: Ankur Bhatnagar
---

# Data Layer — Full Reference

EF Core 8 configuration, repository pattern, unit of work, migrations, and query patterns.
For the entity and configuration overview see the **Data Layer** section in `SKILL.md`.

---

## Table of Contents
1. [DbContext Setup](#dbcontext-setup)
2. [Entity Configurations](#entity-configurations)
3. [Repository Pattern](#repository-pattern)
4. [Unit of Work](#unit-of-work)
5. [Query Patterns](#query-patterns)
6. [Pagination](#pagination)
7. [Soft Delete](#soft-delete)
8. [Migrations](#migrations)

---

## DbContext Setup

```csharp
// Infrastructure/Data/AppDbContext.cs

public sealed class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }

    public DbSet<Order>    Orders    => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        // Apply all IEntityTypeConfiguration<T> classes in this assembly
        builder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
        base.OnModelCreating(builder);
    }
}
```

```csharp
// Registration in Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Default"),
        sql => sql.EnableRetryOnFailure(
            maxRetryCount:       5,
            maxRetryDelay:       TimeSpan.FromSeconds(30),
            errorNumbersToAdd:   null)));
```

---

## Entity Configurations

```csharp
// Infrastructure/Data/Configurations/OrderConfiguration.cs

public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");
        builder.HasKey(o => o.Id);

        builder.Property(o => o.CustomerId)
            .HasMaxLength(100)
            .IsRequired();

        builder.Property(o => o.TotalAmount)
            .HasColumnType("decimal(18,2)")
            .IsRequired();

        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(50)
            .IsRequired();

        builder.Property(o => o.CreatedAt)
            .IsRequired();

        // Navigation: one order has many items
        builder.HasMany(o => o.Items)
            .WithOne(i => i.Order)
            .HasForeignKey(i => i.OrderId)
            .OnDelete(DeleteBehavior.Cascade);

        // Owned type for address value object
        builder.OwnsOne(o => o.ShippingAddress, addr =>
        {
            addr.Property(a => a.Street).HasMaxLength(200);
            addr.Property(a => a.City).HasMaxLength(100);
            addr.Property(a => a.PostCode).HasMaxLength(20);
        });

        // Index for common queries
        builder.HasIndex(o => o.CustomerId);
        builder.HasIndex(o => o.Status);
        builder.HasIndex(o => o.CreatedAt);
    }
}
```

```csharp
// Value object (Owned type)
public sealed record ShippingAddress
{
    public string Street   { get; init; } = string.Empty;
    public string City     { get; init; } = string.Empty;
    public string PostCode { get; init; } = string.Empty;
}
```

---

## Repository Pattern

### Generic repository interface

```csharp
// Application/Interfaces/IRepository.cs

public interface IRepository<TEntity> where TEntity : class
{
    Task<TEntity?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<IReadOnlyList<TEntity>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(TEntity entity, CancellationToken ct = default);
    void Update(TEntity entity);
    void Remove(TEntity entity);
}
```

### Generic repository implementation

```csharp
// Infrastructure/Repositories/Repository.cs

public class Repository<TEntity> : IRepository<TEntity>
    where TEntity : class
{
    protected readonly AppDbContext Context;
    protected readonly DbSet<TEntity> DbSet;

    public Repository(AppDbContext context)
    {
        Context = context;
        DbSet   = context.Set<TEntity>();
    }

    public virtual async Task<TEntity?> GetByIdAsync(
        int id, CancellationToken ct = default)
        => await DbSet.FindAsync([id], ct);

    public virtual async Task<IReadOnlyList<TEntity>> GetAllAsync(
        CancellationToken ct = default)
        => await DbSet.AsNoTracking().ToListAsync(ct);

    public virtual async Task AddAsync(
        TEntity entity, CancellationToken ct = default)
        => await DbSet.AddAsync(entity, ct);

    public virtual void Update(TEntity entity)
        => DbSet.Update(entity);

    public virtual void Remove(TEntity entity)
        => DbSet.Remove(entity);
}
```

### Specialised order repository

```csharp
// Application/Interfaces/IOrderRepository.cs

public interface IOrderRepository : IRepository<Order>
{
    Task<IReadOnlyList<Order>> GetByCustomerIdAsync(
        string customerId, CancellationToken ct = default);

    Task<IReadOnlyList<Order>> GetByStatusAsync(
        OrderStatus status, CancellationToken ct = default);

    Task<PagedResult<Order>> GetPagedAsync(
        OrderFilterRequest filter, CancellationToken ct = default);
}
```

```csharp
// Infrastructure/Repositories/OrderRepository.cs

public sealed class OrderRepository
    : Repository<Order>, IOrderRepository
{
    public OrderRepository(AppDbContext context) : base(context) { }

    public override async Task<Order?> GetByIdAsync(
        int id, CancellationToken ct = default)
        => await DbSet
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task<IReadOnlyList<Order>> GetByCustomerIdAsync(
        string customerId, CancellationToken ct = default)
        => await DbSet
            .AsNoTracking()
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.CreatedAt)
            .ToListAsync(ct);

    public async Task<PagedResult<Order>> GetPagedAsync(
        OrderFilterRequest filter, CancellationToken ct = default)
    {
        var query = DbSet.AsNoTracking().AsQueryable();

        if (!string.IsNullOrEmpty(filter.Status))
            query = query.Where(o => o.Status.ToString() == filter.Status);

        if (!string.IsNullOrEmpty(filter.CustomerId))
            query = query.Where(o => o.CustomerId == filter.CustomerId);

        if (filter.From.HasValue)
            query = query.Where(o => o.CreatedAt >= filter.From.Value);

        if (filter.To.HasValue)
            query = query.Where(o => o.CreatedAt <= filter.To.Value);

        var totalCount = await query.CountAsync(ct);

        var items = await query
            .OrderByDescending(o => o.CreatedAt)
            .Skip((filter.Page - 1) * filter.PageSize)
            .Take(filter.PageSize)
            .ToListAsync(ct);

        return new PagedResult<Order>(items, totalCount, filter.Page, filter.PageSize);
    }
}
```

---

## Unit of Work

```csharp
// Application/Interfaces/IUnitOfWork.cs

public interface IUnitOfWork : IDisposable
{
    IOrderRepository Orders { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

```csharp
// Infrastructure/Data/UnitOfWork.cs

public sealed class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;

    public UnitOfWork(AppDbContext context, IOrderRepository orders)
    {
        _context = context;
        Orders   = orders;
    }

    public IOrderRepository Orders { get; }

    public Task<int> SaveChangesAsync(CancellationToken ct = default)
        => _context.SaveChangesAsync(ct);

    public void Dispose() => _context.Dispose();
}
```

```csharp
// Service using UoW
public sealed class OrderService : IOrderService
{
    private readonly IUnitOfWork _uow;

    public OrderService(IUnitOfWork uow) => _uow = uow;

    public async Task<OrderResponse> CreateAsync(
        CreateOrderRequest request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId, request.TotalAmount);
        await _uow.Orders.AddAsync(order, ct);
        await _uow.SaveChangesAsync(ct);
        return OrderMapper.ToResponse(order);
    }
}
```

---

## Query Patterns

```csharp
// Always use AsNoTracking() for read-only queries
var orders = await context.Orders
    .AsNoTracking()
    .Where(o => o.Status == OrderStatus.Pending)
    .ToListAsync(ct);

// Always project to DTOs — never return entities from the API
var responses = await context.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderResponse
    {
        Id          = o.Id,
        CustomerId  = o.CustomerId,
        Status      = o.Status.ToString(),
        TotalAmount = o.TotalAmount,
        CreatedAt   = o.CreatedAt,
    })
    .ToListAsync(ct);

// Eager loading with Include
var order = await context.Orders
    .Include(o => o.Items)
    .Include(o => o.Customer)
    .FirstOrDefaultAsync(o => o.Id == id, ct);

// Split queries for multiple collections (avoids cartesian explosion)
var order = await context.Orders
    .AsSplitQuery()
    .Include(o => o.Items)
    .Include(o => o.Tags)
    .FirstOrDefaultAsync(o => o.Id == id, ct);

// Streaming large result sets
await foreach (var order in context.Orders
    .AsNoTracking()
    .AsAsyncEnumerable()
    .WithCancellation(ct))
{
    await ProcessAsync(order, ct);
}
```

---

## Pagination

```csharp
// Application/Models/PagedResult.cs

public sealed record PagedResult<T>
{
    public IReadOnlyList<T> Items      { get; init; }
    public int              TotalCount { get; init; }
    public int              Page       { get; init; }
    public int              PageSize   { get; init; }
    public int              TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
    public bool             HasNext    => Page < TotalPages;
    public bool             HasPrev    => Page > 1;

    public PagedResult(IReadOnlyList<T> items, int totalCount, int page, int pageSize)
    {
        Items      = items;
        TotalCount = totalCount;
        Page       = page;
        PageSize   = pageSize;
    }
}

// Reusable extension
public static class QueryableExtensions
{
    public static async Task<PagedResult<T>> ToPagedResultAsync<T>(
        this IQueryable<T> query,
        int page,
        int pageSize,
        CancellationToken ct = default)
    {
        var total = await query.CountAsync(ct);
        var items = await query
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync(ct);

        return new PagedResult<T>(items, total, page, pageSize);
    }
}
```

---

## Soft Delete

```csharp
// Domain/Entities/Base/SoftDeletableEntity.cs
public abstract class SoftDeletableEntity
{
    public bool      IsDeleted { get; private set; }
    public DateTime? DeletedAt { get; private set; }

    public void Delete()
    {
        IsDeleted = true;
        DeletedAt = DateTime.UtcNow;
    }
}

// Global query filter in AppDbContext
protected override void OnModelCreating(ModelBuilder builder)
{
    builder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);

    // Apply soft delete filter to all entities that inherit SoftDeletableEntity
    foreach (var entityType in builder.Model.GetEntityTypes())
    {
        if (!typeof(SoftDeletableEntity).IsAssignableFrom(entityType.ClrType))
            continue;

        var parameter = Expression.Parameter(entityType.ClrType, "e");
        var filter    = Expression.Lambda(
            Expression.Equal(
                Expression.Property(parameter, nameof(SoftDeletableEntity.IsDeleted)),
                Expression.Constant(false)),
            parameter);

        builder.Entity(entityType.ClrType).HasQueryFilter(filter);
    }
}
```

---

## Migrations

```bash
# Add a new migration (from solution root)
dotnet ef migrations add AddOrdersTable \
  --project src/YourApp.Infrastructure \
  --startup-project src/YourApp.Api

# Apply migrations to the database
dotnet ef database update \
  --project src/YourApp.Infrastructure \
  --startup-project src/YourApp.Api

# Generate a SQL script for production (idempotent)
dotnet ef migrations script \
  --idempotent \
  --output migrations.sql \
  --project src/YourApp.Infrastructure \
  --startup-project src/YourApp.Api
```

```csharp
// Apply migrations on startup (only in dev / staging)
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    if (app.Environment.IsDevelopment())
        await db.Database.MigrateAsync();
}
```

**Migration rules:**
- Never modify an existing migration — add a new one
- Always review generated migration SQL before applying to production
- Use idempotent SQL scripts for production deployments, not `dotnet ef database update`
- Add database indexes in entity configurations, not as raw SQL in migrations

---
author: Ankur Bhatnagar
---

# Authentication & Authorization — Full Reference

JWT token generation, refresh tokens, policy-based authorization, custom requirements,
and resource-based authorization patterns for ASP.NET Core 8.
For the overview and middleware registration see the **Auth** section in `SKILL.md`.

---

## Table of Contents
1. [JWT Configuration](#jwt-configuration)
2. [Token Service](#token-service)
3. [Refresh Tokens](#refresh-tokens)
4. [Policy-Based Authorization](#policy-based-authorization)
5. [Custom Authorization Requirements](#custom-authorization-requirements)
6. [Resource-Based Authorization](#resource-based-authorization)
7. [Current User Context](#current-user-context)
8. [External Identity Providers (OIDC)](#external-identity-providers-oidc)

---

## JWT Configuration

```json
// appsettings.json
{
  "Jwt": {
    "Key": "replace-with-a-256-bit-secret-never-commit-to-source-control",
    "Issuer": "https://yourapp.example.com",
    "Audience": "yourapp-api",
    "ExpiryMinutes": 60,
    "RefreshTokenExpiryDays": 7
  }
}
```

```csharp
// Application/Models/JwtOptions.cs

public sealed class JwtOptions
{
    public const string SectionName = "Jwt";

    [Required] public string Key              { get; init; } = string.Empty;
    [Required] public string Issuer           { get; init; } = string.Empty;
    [Required] public string Audience         { get; init; } = string.Empty;
    public int               ExpiryMinutes    { get; init; } = 60;
    public int               RefreshTokenExpiryDays { get; init; } = 7;
}
```

```csharp
// Extensions/ServiceCollectionExtensions.cs

public static IServiceCollection AddJwtAuthentication(
    this IServiceCollection services,
    IConfiguration configuration)
{
    services
        .AddOptions<JwtOptions>()
        .BindConfiguration(JwtOptions.SectionName)
        .ValidateDataAnnotations()
        .ValidateOnStart();

    var jwtOptions = configuration
        .GetSection(JwtOptions.SectionName)
        .Get<JwtOptions>()!;

    services
        .AddAuthentication(options =>
        {
            options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
            options.DefaultChallengeScheme    = JwtBearerDefaults.AuthenticationScheme;
        })
        .AddJwtBearer(options =>
        {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer           = true,
                ValidateAudience         = true,
                ValidateLifetime         = true,
                ValidateIssuerSigningKey = true,
                ValidIssuer              = jwtOptions.Issuer,
                ValidAudience            = jwtOptions.Audience,
                IssuerSigningKey         = new SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(jwtOptions.Key)),
                ClockSkew                = TimeSpan.FromSeconds(30),
            };

            // Support token in query string for SignalR / WebSocket hubs
            options.Events = new JwtBearerEvents
            {
                OnMessageReceived = context =>
                {
                    var token = context.Request.Query["access_token"];
                    var path  = context.HttpContext.Request.Path;
                    if (!string.IsNullOrEmpty(token) && path.StartsWithSegments("/hubs"))
                        context.Token = token;
                    return Task.CompletedTask;
                },
            };
        });

    return services;
}
```

---

## Token Service

```csharp
// Application/Interfaces/ITokenService.cs

public interface ITokenService
{
    string  GenerateAccessToken(AppUser user);
    string  GenerateRefreshToken();
    ClaimsPrincipal? GetPrincipalFromExpiredToken(string token);
}
```

```csharp
// Application/Services/TokenService.cs

public sealed class TokenService : ITokenService
{
    private readonly JwtOptions _jwt;

    public TokenService(IOptions<JwtOptions> options)
        => _jwt = options.Value;

    public string GenerateAccessToken(AppUser user)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub,   user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email!),
            new(JwtRegisteredClaimNames.Jti,   Guid.NewGuid().ToString()),
            new("role",                        user.Role),
        };

        // Add permission claims
        foreach (var permission in user.Permissions)
            claims.Add(new Claim("permission", permission));

        var key         = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwt.Key));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer:             _jwt.Issuer,
            audience:           _jwt.Audience,
            claims:             claims,
            notBefore:          DateTime.UtcNow,
            expires:            DateTime.UtcNow.AddMinutes(_jwt.ExpiryMinutes),
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken()
    {
        var bytes = RandomNumberGenerator.GetBytes(64);
        return Convert.ToBase64String(bytes);
    }

    public ClaimsPrincipal? GetPrincipalFromExpiredToken(string token)
    {
        var parameters = new TokenValidationParameters
        {
            ValidateIssuer           = true,
            ValidateAudience         = true,
            ValidateIssuerSigningKey = true,
            ValidateLifetime         = false,   // Allow expired token for refresh
            ValidIssuer              = _jwt.Issuer,
            ValidAudience            = _jwt.Audience,
            IssuerSigningKey         = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(_jwt.Key)),
        };

        var handler   = new JwtSecurityTokenHandler();
        var principal = handler.ValidateToken(token, parameters, out var validatedToken);

        if (validatedToken is not JwtSecurityToken jwt ||
            !jwt.Header.Alg.Equals(
                SecurityAlgorithms.HmacSha256,
                StringComparison.OrdinalIgnoreCase))
            return null;

        return principal;
    }
}
```

---

## Refresh Tokens

```csharp
// Domain/Entities/RefreshToken.cs

public sealed class RefreshToken
{
    public int      Id         { get; init; }
    public string   Token      { get; init; } = string.Empty;
    public string   UserId     { get; init; } = string.Empty;
    public DateTime ExpiresAt  { get; init; }
    public DateTime CreatedAt  { get; init; } = DateTime.UtcNow;
    public bool     IsRevoked  { get; private set; }
    public string?  ReplacedBy { get; private set; }

    private RefreshToken() { }

    public static RefreshToken Create(string token, string userId, int expiryDays)
        => new()
        {
            Token     = token,
            UserId    = userId,
            ExpiresAt = DateTime.UtcNow.AddDays(expiryDays),
        };

    public bool IsExpired  => DateTime.UtcNow >= ExpiresAt;
    public bool IsActive   => !IsRevoked && !IsExpired;

    public void Revoke(string? replacedBy = null)
    {
        IsRevoked  = true;
        ReplacedBy = replacedBy;
    }
}
```

```csharp
// Controllers/AuthController.cs

[ApiController]
[Route("api/auth")]
public sealed class AuthController : ControllerBase
{
    private readonly IAuthService _authService;

    public AuthController(IAuthService authService) => _authService = authService;

    [HttpPost("login")]
    [AllowAnonymous]
    [ProducesResponseType<TokenResponse>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    public async Task<IActionResult> Login(
        [FromBody] LoginRequest request, CancellationToken ct)
    {
        var result = await _authService.LoginAsync(request, ct);
        return result is null ? Unauthorized() : Ok(result);
    }

    [HttpPost("refresh")]
    [AllowAnonymous]
    [ProducesResponseType<TokenResponse>(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    public async Task<IActionResult> Refresh(
        [FromBody] RefreshTokenRequest request, CancellationToken ct)
    {
        var result = await _authService.RefreshAsync(request, ct);
        return result is null ? Unauthorized() : Ok(result);
    }

    [HttpPost("revoke")]
    [Authorize]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    public async Task<IActionResult> Revoke(
        [FromBody] RevokeTokenRequest request, CancellationToken ct)
    {
        await _authService.RevokeAsync(request.RefreshToken, ct);
        return NoContent();
    }
}
```

---

## Policy-Based Authorization

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    // Role-based
    options.AddPolicy("AdminOnly",
        policy => policy.RequireRole("admin"));

    // Claim-based
    options.AddPolicy("OrderRead",
        policy => policy.RequireClaim("permission", "orders:read"));

    options.AddPolicy("OrderWrite",
        policy => policy.RequireClaim("permission", "orders:write"));

    // Combined
    options.AddPolicy("SeniorStaff", policy =>
    {
        policy.RequireRole("staff");
        policy.RequireClaim("department", "operations", "management");
    });

    // Default policy — require authenticated user
    options.DefaultPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});
```

```csharp
// Controller / endpoint usage
[Authorize(Policy = "OrderRead")]
[HttpGet]
public async Task<IActionResult> GetAll(CancellationToken ct) { ... }

[Authorize(Policy = "OrderWrite")]
[HttpPost]
public async Task<IActionResult> Create(
    [FromBody] CreateOrderRequest request, CancellationToken ct) { ... }
```

---

## Custom Authorization Requirements

```csharp
// Application/Authorization/MinimumAgeRequirement.cs

public sealed class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }
    public MinimumAgeRequirement(int minimumAge) => MinimumAge = minimumAge;
}

public sealed class MinimumAgeHandler
    : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var birthDateClaim = context.User.FindFirst("birthdate");

        if (birthDateClaim is null ||
            !DateTime.TryParse(birthDateClaim.Value, out var birthDate))
        {
            context.Fail();
            return Task.CompletedTask;
        }

        var age = DateTime.Today.Year - birthDate.Year;
        if (birthDate.Date > DateTime.Today.AddYears(-age)) age--;

        if (age >= requirement.MinimumAge)
            context.Succeed(requirement);
        else
            context.Fail();

        return Task.CompletedTask;
    }
}

// Registration
services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
services.AddAuthorization(options =>
{
    options.AddPolicy("Over18",
        policy => policy.AddRequirements(new MinimumAgeRequirement(18)));
});
```

---

## Resource-Based Authorization

```csharp
// Application/Authorization/OrderOperations.cs

public static class OrderOperations
{
    public static readonly OperationAuthorizationRequirement Read   = new() { Name = "Read" };
    public static readonly OperationAuthorizationRequirement Update = new() { Name = "Update" };
    public static readonly OperationAuthorizationRequirement Delete = new() { Name = "Delete" };
}

public sealed class OrderAuthorizationHandler
    : AuthorizationHandler<OperationAuthorizationRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OperationAuthorizationRequirement requirement,
        Order resource)
    {
        var userId = context.User.FindFirstValue(JwtRegisteredClaimNames.Sub);

        // Admins can do anything
        if (context.User.IsInRole("admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // Owners can read and update their own orders
        if (resource.CustomerId == userId &&
            (requirement == OrderOperations.Read ||
             requirement == OrderOperations.Update))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Controller usage
[HttpDelete("{id:int}")]
public async Task<IActionResult> Delete(
    int id,
    [FromServices] IAuthorizationService authService,
    CancellationToken ct)
{
    var order = await _service.GetEntityByIdAsync(id, ct);
    if (order is null) return NotFound();

    var authResult = await authService.AuthorizeAsync(
        User, order, OrderOperations.Delete);

    if (!authResult.Succeeded) return Forbid();

    await _service.DeleteAsync(id, ct);
    return NoContent();
}
```

---

## Current User Context

```csharp
// Application/Interfaces/ICurrentUser.cs

public interface ICurrentUser
{
    string  Id          { get; }
    string  Email       { get; }
    string  Role        { get; }
    bool    IsAuthenticated { get; }
    bool    HasPermission(string permission);
}
```

```csharp
// Infrastructure/Services/CurrentUser.cs

public sealed class CurrentUser : ICurrentUser
{
    private readonly IHttpContextAccessor _http;

    public CurrentUser(IHttpContextAccessor http) => _http = http;

    private ClaimsPrincipal User =>
        _http.HttpContext?.User
        ?? throw new InvalidOperationException("No HttpContext available.");

    public string Id    => User.FindFirstValue(JwtRegisteredClaimNames.Sub) ?? string.Empty;
    public string Email => User.FindFirstValue(JwtRegisteredClaimNames.Email) ?? string.Empty;
    public string Role  => User.FindFirstValue("role") ?? string.Empty;
    public bool   IsAuthenticated => User.Identity?.IsAuthenticated ?? false;

    public bool HasPermission(string permission)
        => User.Claims
               .Where(c => c.Type == "permission")
               .Any(c => c.Value == permission);
}

// Registration
services.AddHttpContextAccessor();
services.AddScoped<ICurrentUser, CurrentUser>();
```

---

## External Identity Providers (OIDC)

```csharp
// Azure AD / Entra ID
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = $"https://login.microsoftonline.com/{tenantId}/v2.0";
        options.Audience  = configuration["AzureAd:ClientId"];
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer   = true,
            ValidateAudience = true,
            NameClaimType    = "preferred_username",
            RoleClaimType    = "roles",
        };
    });
```

```csharp
// Keycloak
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority            = "https://keycloak.example.com/realms/myrealm";
        options.Audience             = "yourapp-api";
        options.RequireHttpsMetadata = true;
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer   = true,
            ValidateAudience = true,
            RoleClaimType    = "roles",
        };
    });
```

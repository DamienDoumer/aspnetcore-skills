---
name: aspnetcore-jwt-auth
description: Use when adding JWT-based email/password + refresh-token authentication (with optional Google/Firebase social login) to an ASP.NET Core project, or when replacing hand-rolled Login/Register/Refresh/ForgotPassword/ResetPassword minimal-API endpoints — enforces use of The.Jwt.Auth.Endpoints NuGet package instead of rolling the endpoints by hand.
---

# ASP.NET Core JWT Authentication with `The.Jwt.Auth.Endpoints`

## Overview

`The.Jwt.Auth.Endpoints` is a JWT-only alternative to the built-in
.NET 8 Identity API endpoints. It ships a fixed set of pre-built
minimal-API endpoints under `/api/auth/*` and requires ASP.NET Core
Identity in the host project. This skill enforces its use for every
new project that needs standard email/password authentication with
refresh tokens.

Illustrative names in the snippets below — `User`,
`ApplicationDbContext`, `JwtSettings`, `WelcomeActionService` — are
placeholders. Substitute your own domain types.

## When to use / when not to use

Use when:

- Adding authentication to a new ASP.NET Core service that needs
  email/password login, registration with email confirmation,
  password reset, and refresh-token rotation.
- Adding Google/Firebase social login on top of email/password.
- Migrating away from the built-in Identity API endpoints because
  their proprietary token format does not fit your clients.

Do **not** use when:

- The auth flow is highly custom (MFA-only, passkeys-only, custom
  federation). The package's endpoints are fixed and will not stretch
  that far.
- The user store is not `IdentityUser`-derived (custom user tables).

## Prerequisites

1. Reference `Microsoft.AspNetCore.Identity.EntityFrameworkCore` from
   your Infrastructure project.
2. A `DbContext` that inherits from `IdentityDbContext<TUser>` (or
   `IdentityDbContext<TUser, IdentityRole, string>` when you
   customise roles). All Identity tables (`AspNetUsers`,
   `AspNetRoles`, ...) must be part of your EF Core migrations.
3. A running EF Core provider (SQL Server, PostgreSQL, SQLite, ...).
4. A `User` class that inherits `IdentityUser` and carries the fields
   the package touches:

```csharp
public class User : IdentityUser
{
    public string? FirstName { get; set; }
    public string? LastName  { get; set; }
    public string  PictureUrl              { get; set; } = string.Empty;
    public DateTime CreatedAt              { get; set; }
    public DateTime? LastLoginAt           { get; set; }
    public string  RefreshToken            { get; set; } = string.Empty;
    public DateTime RefreshTokenExpiryTime { get; set; }
}
```

`RefreshToken` + `RefreshTokenExpiryTime` are how the reference
`IRefreshTokenRepository` implementation persists rotation state on
the user row. If you keep refresh tokens in a separate entity — for
multi-device support or revocation history — map them there
instead; the interface does not care where the storage lives.

## Endpoints exposed

Once wired up, the package maps these routes:

| Endpoint | Method | Description |
| --- | --- | --- |
| `/api/auth/register` | POST | User registration with email confirmation |
| `/api/auth/login` | POST | User login with email/password |
| `/api/auth/refresh` | POST | Refresh JWT access token |
| `/api/auth/confirmEmail` | GET | Confirm user email address |
| `/api/auth/forgotPassword` | POST | Initiate password reset process |
| `/api/auth/resetPassword` | POST | Complete password reset |
| `/api/auth/social/google` | POST | Google Firebase authentication *(optional)* |

The Google endpoint is only mapped when
`opts.GoogleFirebaseAuthOptions` is set in the DI configuration
(see §4 below).

## Step-by-step setup

### 1. Install the NuGet package

Add `The.Jwt.Auth.Endpoints` to every project that either wires it
into DI or maps its endpoints — typically both the API project (for
`app.MapJwtAuthEndpoints<TUser>()` in `Program.cs`) and the
Infrastructure project (for `services.AddJwtAuthEndpoints<TUser>()`
and the four consumer implementations described in §5):

```bash
dotnet add package The.Jwt.Auth.Endpoints
```

Run the command from each project's directory, or from the solution
root with `--project <Project>.csproj` to target a specific project.
Pin every reference to the **same** version — a version drift between
the API and Infrastructure projects will either fail to compile or
silently pull two copies of the package.

The package targets .NET 9. Verify each consuming project's
`<TargetFramework>` is `net9.0` (or higher).

### 2. Strongly-typed `JwtSettings`

Define one POCO with a `Tag` constant, bind it via `IOptions<T>`, and
fail fast if the secret is missing (see also
`aspnetcore-safe-code` §6):

```csharp
public class JwtSettings
{
    public const string Tag = "JwtSettings";

    public string Secret   { get; set; } = string.Empty;
    public string Issuer   { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public int TokenLifetimeInMinutes        { get; set; }
    public int RefreshTokenLifetimeInMinutes { get; set; }
}
```

```csharp
var jwt = new JwtSettings();
builder.Configuration.Bind(JwtSettings.Tag, jwt);

if (string.IsNullOrWhiteSpace(jwt.Secret))
    throw new InvalidOperationException(
        "JwtSettings:Secret is missing from configuration.");

builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection(JwtSettings.Tag));
```

### 3. ASP.NET Core Identity

Register Identity **before** the package's DI extension so its
stores are ready when the JWT auth chain is built:

```csharp
builder.Services
    .AddIdentity<User, IdentityRole>(options =>
    {
        options.Password.RequiredLength      = 8;
        options.Password.RequireDigit        = true;
        options.Password.RequireUppercase    = true;
        options.User.RequireUniqueEmail      = true;

        options.Lockout.DefaultLockoutTimeSpan  = TimeSpan.FromMinutes(15);
        options.Lockout.MaxFailedAccessAttempts = 5;
        options.Lockout.AllowedForNewUsers      = true;
    })
    .AddEntityFrameworkStores<ApplicationDbContext>()
    .AddDefaultTokenProviders();
```

Do not disable password complexity without a written trade-off and a
compensating control (see `aspnetcore-safe-code` §13).

### 4. Register the package

Call `AddJwtAuthEndpoints<TUser>()` from
`The.Jwt.Auth.Endpoints.Extensions`. Copy every setting from the
strongly-typed `JwtSettings` — never type the secret twice at
separate call sites:

```csharp
using The.Jwt.Auth.Endpoints.Extensions;
using Microsoft.IdentityModel.Tokens;

var key = Encoding.UTF8.GetBytes(jwt.Secret);

builder.Services.AddJwtAuthEndpoints<User>(opts =>
{
    opts.JwtSettings.Secret   = jwt.Secret;
    opts.JwtSettings.Issuer   = jwt.Issuer;
    opts.JwtSettings.Audience = jwt.Audience;
    opts.JwtSettings.TokenLifeSpanInMinutes        = jwt.TokenLifetimeInMinutes;
    opts.JwtSettings.RefreshTokenLifeSpanInMinutes = jwt.RefreshTokenLifetimeInMinutes;

    opts.JwtAuthSchemeOptions.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer           = true,
        ValidateAudience         = true,
        ValidateLifetime         = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer              = jwt.Issuer,
        ValidAudience            = jwt.Audience,
        IssuerSigningKey         = new SymmetricSecurityKey(key),

        // Pick an explicit skew. TimeSpan.Zero for a single-node
        // deployment; 1–5 minutes when clock drift across nodes is
        // a concern. Never leave the framework default silently.
        ClockSkew = TimeSpan.Zero,
    };

    opts.JwtAuthSchemeOptions.RequireHttpsMetadata = !builder.Environment.IsDevelopment();
    opts.JwtAuthSchemeOptions.SaveToken            = true;

    // Optional Google/Firebase social login. Uncomment and point at
    // the credential JSON to also expose /api/auth/social/google.
    // opts.GoogleFirebaseAuthOptions = new AppOptions
    // {
    //     Credential = GoogleCredential.FromFile(firebaseCredentialPath),
    // };
});

builder.Services.AddAuthorization();
```

If Google/Firebase login is enabled, add the credential JSON to
`.gitignore` and load it from a path outside source control.

`AddJwtAuthEndpoints<TUser>()` also registers an
`IValidateOptions<JwtAuthEndpointsConfigOptions>` that fails startup
if `Secret`/`Issuer`/`Audience` are blank or either lifespan is `0`.
The same validator rejects any non-default authentication-scheme
name — do not touch `opts.AuthenticationScheme`.

### 5. Required consumer implementations

Four contracts from `The.Jwt.Auth.Endpoints.Helpers` must be
registered by the host. The package `TryAddScoped`s a default
`IWelcomeActionService` and `IJwtTokenProvider`, so a production app
only *has* to register the first three — but the default welcome
service is a no-op, so almost every app overrides it too.

```csharp
services.AddScoped<IIdentityUserFactory<User>, ApplicationUserFactory>();
services.AddScoped<IRefreshTokenRepository, RefreshTokenRepository>();
services.AddScoped<IEmailSender<User>, IdentityEmailSender>();
services.AddScoped<IWelcomeActionService, WelcomeActionService>();
```

**`IIdentityUserFactory<TUser>`** — build a `TUser` from
firstName/lastName/email/picture:

```csharp
public class ApplicationUserFactory : IIdentityUserFactory<User>
{
    public User CreateUser(
        string firstName, string lastName, string email, string? picture = null)
        => new()
        {
            UserName   = email,
            Email      = email,
            FirstName  = firstName,
            LastName   = lastName,
            PictureUrl = picture ?? string.Empty,
            CreatedAt  = DateTime.UtcNow,
        };
}
```

**`IRefreshTokenRepository`** — three-method contract that persists
the current refresh token per user. A minimal EF Core implementation
storing the token on the user row:

```csharp
public class RefreshTokenRepository(ApplicationDbContext db)
    : IRefreshTokenRepository
{
    public async Task<bool> AddOrUpdateRefreshToken(
        string userId, string refreshToken, DateTime expiryTime)
    {
        var user = await db.Users.FirstOrDefaultAsync(u => u.Id == userId);
        if (user is null) return false;

        user.RefreshToken           = refreshToken;
        user.RefreshTokenExpiryTime = expiryTime;
        await db.SaveChangesAsync();
        return true;
    }

    public async Task<bool> DeleteRefreshToken(string userId, string refreshToken)
    {
        var user = await db.Users
            .FirstOrDefaultAsync(u => u.Id == userId && u.RefreshToken == refreshToken);
        if (user is null) return false;

        user.RefreshToken           = string.Empty;
        user.RefreshTokenExpiryTime = DateTime.MinValue;
        await db.SaveChangesAsync();
        return true;
    }

    public async Task<(string refreshToken, DateTime expiryTime)> GetRefreshToken(
        string userId)
    {
        var user = await db.Users.FirstOrDefaultAsync(u => u.Id == userId);
        return user is null || string.IsNullOrEmpty(user.RefreshToken)
            ? (string.Empty, DateTime.MinValue)
            : (user.RefreshToken, user.RefreshTokenExpiryTime);
    }
}
```

**`IEmailSender<TUser>`** — the interface from
`Microsoft.AspNetCore.Identity.UI.Services`. Register a thin adapter
over whatever email provider the app already uses.

**`IWelcomeActionService`** — fires after successful *first-time*
account creation (both `/api/auth/register` and
`/api/auth/social/google`). It is **not** fired on login. Use it to
send a welcome email, seed defaults, or emit an audit event:

```csharp
public class WelcomeActionService(
    ILogger<WelcomeActionService> logger,
    IEmailService emails) : IWelcomeActionService
{
    public async Task PerformWelcomeActionsAsync(
        string userId, string userEmail, string username)
    {
        logger.LogInformation("Welcome actions for {UserId}", userId);
        await emails.SendWelcomeEmailAsync(userEmail, username);
    }
}
```

## Middleware order in `Program.cs`

The pipeline is order-sensitive. Three constraints matter:

- `app.UseCors(...)` must run **before** `app.UseAuthentication()`.
  Otherwise the CORS preflight (`OPTIONS`) is bounced as `401` and
  browsers fail silently.
- `app.UseAuthentication()` runs before `app.UseAuthorization()`.
- `app.MapJwtAuthEndpoints<User>()` and `app.MapApiEndpoints()`
  (see `aspnetcore-clean-architecture`) both come **after**
  `app.UseAuthorization()`. Their order relative to each other does
  not matter.

The canonical pipeline:

```csharp
var app = builder.Build();

app.UseSerilogRequestLogging();
app.UseStatusCodePages();
app.UseExceptionHandler();
app.UseHttpsRedirection();

app.UseCors(CorsPolicyName);         // BEFORE authentication
app.UseAuthentication();
app.UseAuthorization();

app.MapJwtAuthEndpoints<User>();     // package endpoints
app.MapApiEndpoints();               // your feature endpoints

app.Run();
```

## Configuration reference

**`JwtSettings`** — the POCO you bind from configuration:

| Field | Recommended range |
| --- | --- |
| `Secret` | ≥ 32 chars, high-entropy. Load from user-secrets / env / vault. |
| `Issuer` | Stable URL you control (`https://api.yourapp.com`). |
| `Audience` | Stable name of the API (`YourApp`). |
| `TokenLifeSpanInMinutes` | 15 – 60 minutes for access tokens. |
| `RefreshTokenLifeSpanInMinutes` | 1440 – 20160 (1 – 14 days) for refresh tokens. |

**`JwtAuthEndpointsConfigOptions`** — the shape of the delegate
passed to `AddJwtAuthEndpoints<TUser>()`, from
`The.Jwt.Auth.Endpoints.Settings`:

| Field | Purpose |
| --- | --- |
| `JwtSettings` | The five fields above. |
| `JwtAuthSchemeOptions` | Full `JwtBearerOptions` — set `TokenValidationParameters`, `RequireHttpsMetadata`, `SaveToken`, `Events`. |
| `GoogleFirebaseAuthOptions` | Set to a `FirebaseAdmin.AppOptions` to enable `/api/auth/social/google`; leave `null` to skip. |
| `AuthenticationScheme` | Deliberately internal — the options validator rejects any non-default scheme name. Do not touch. |

## Rules

- Load `JwtSettings.Secret` from user-secrets, environment variables,
  or a vault — never from `appsettings.json`
  (see `aspnetcore-safe-code` §6).
- Bind the JWT settings once (POCO + `IOptions<T>`) and reuse the
  same values inside `opts.JwtSettings` and inside
  `TokenValidationParameters`. Never type the secret twice.
- Set `RequireHttpsMetadata = !builder.Environment.IsDevelopment()`
  so production always requires HTTPS metadata (see
  `aspnetcore-safe-code` §8).
- Choose an **explicit** `ClockSkew`. `TimeSpan.Zero` for single-node
  deployments; `TimeSpan.FromMinutes(1)` to `TimeSpan.FromMinutes(5)`
  when NTP drift across nodes is a concern. Never leave the framework
  default silently.
- Register `AddIdentity<TUser, IdentityRole>()` **before**
  `AddJwtAuthEndpoints<TUser>()`.
- Register all four required consumer services
  (`IIdentityUserFactory<TUser>`, `IRefreshTokenRepository`,
  `IEmailSender<TUser>`, `IWelcomeActionService`).
- Add the Firebase credential JSON (if used) to `.gitignore` and load
  it from a path outside source control.
- Enforce `.RequireAuthorization()` on every non-auth endpoint, or
  set a fallback policy that requires an authenticated user
  (see `aspnetcore-safe-code` §9).
- Access-token lifetime ≤ 1 hour; refresh-token lifetime chosen from
  your rotation policy (≤ 14 days is a common ceiling).

## Anti-patterns

- Hand-rolling `LoginEndpoint.cs` / `RegisterEndpoint.cs` /
  `RefreshTokenEndpoint.cs` / `ForgotPasswordEndpoint.cs` /
  `ResetPasswordEndpoint.cs` when this package covers the same shape.
- Storing `JwtSettings.Secret` in `appsettings.json`.
- Renaming any of the `AuthenticationScheme.Default*` scheme names —
  the options validator will fail startup.
- Registering `AddJwtAuthEndpoints<TUser>` before
  `AddIdentity<TUser, IdentityRole>` — the Identity stores end up
  unwired.
- Duplicating the secret string across `opts.JwtSettings.Secret` and
  `TokenValidationParameters.IssuerSigningKey` from separate string
  literals — bind once, reuse.
- Forgetting `app.UseCors(...)` before `app.UseAuthentication()`.
- Skipping the `IWelcomeActionService` override when the default
  no-op is not enough — new users get no welcome email.
- Committing the Firebase credential JSON (or any other credential
  file the package loads at startup) to source control.

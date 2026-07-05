---
name: aspnetcore-safe-code
description: Use when writing, reviewing, or refactoring C# code in an ASP.NET Core solution — nullable-analyzer promotion, structured ILogger usage with correct levels, boundary-only try/catch with a global IExceptionHandler, CancellationToken threading, IOptions<T> and never-commit-secrets, safe cryptography, hardened HTTP surface (HTTPS, scoped CORS, per-route authorization, API versioning), and immutability-first access modifiers.
---

# ASP.NET Core Safe C# Code

## Overview

This skill defines the safe-coding rules every C# file in an ASP.NET
Core solution must follow. Treat all rules as **MUST** unless a rule
is explicitly tagged *guidance*. Illustrative type names (`Foo`,
`Product`, `<Service>`) are placeholders — substitute your own domain
concepts.

The companion file `editorconfig.template` in this skill's folder is
a ready-to-use `.editorconfig`. Copy it to the solution root as
`.editorconfig` and commit it. It turns the most important nullable
rules into build-breaking errors.

## 1. Nullability — make the compiler your bodyguard

- Enable nullable reference types at the solution level:
  `<Nullable>enable</Nullable>` in every csproj (or a shared
  `Directory.Build.props`).
- Promote the nullable analyzer warnings to `error` in `.editorconfig`
  so unsafe code fails the build:

  ```editorconfig
  [*.{cs,vb}]
  dotnet_diagnostic.CS8618.severity = error   # non-nullable not initialised in ctor
  dotnet_diagnostic.CS8601.severity = error   # possible null-ref assignment
  dotnet_diagnostic.CS8602.severity = error   # possible null-ref dereference
  dotnet_diagnostic.CS8604.severity = error   # possible null-ref argument
  dotnet_diagnostic.CS8619.severity = error   # nullability mismatch on generic
  ```

- Guard every public entry point:

  ```csharp
  public FooService(IRepository repository, ILogger<FooService> logger)
  {
      ArgumentNullException.ThrowIfNull(repository);
      ArgumentNullException.ThrowIfNull(logger);
      _repository = repository;
      _logger = logger;
  }

  public Task<Foo> Handle(string id, CancellationToken cancellationToken)
  {
      if (string.IsNullOrWhiteSpace(id))
          throw new BadRequestException("id is required", ErrorCodes.MissingId);
      // ...
  }
  ```

- Prefer `string.IsNullOrWhiteSpace` over `string.IsNullOrEmpty` when
  the input comes from a user — whitespace-only strings are almost
  always garbage.
- Use `??` in two distinct ways:
  - Fail-fast for invariants:
    `var value = source.Field ?? throw new InvalidOperationException("Field expected");`
  - Safe defaults in DTO projections: `Name = entity.Name ?? "";`.
- Avoid the null-forgiving `!` operator. Its only legitimate use is
  when the invariant is proven **immediately above** the call site.
  In particular, `_userManager.FindByIdAsync(id)!` is a bug in
  waiting — that API returns `null` when the id is unknown, and the
  `!` merely delays the `NullReferenceException` until the next
  member access.

## 2. Logging

- Inject `ILogger<T>` via constructor. Do **not** call the static
  Serilog `Log.Xxx` anywhere except the bootstrap `try/catch` in
  `Program.cs`.
- Structured logging only. Use named placeholders `{Foo}`; never
  interpolate or concatenate values into the message template:

  ```csharp
  // Structured — Serilog/Seq/ES can index UserId
  _logger.LogInformation("Product {ProductId} created for {UserId}", productId, userId);

  // Interpolated — a single opaque string, unindexable
  _logger.LogInformation($"Product {productId} created for {userId}");
  ```

- Pick the level from the caller's intent, not from the exception
  type:

  | Level | When to use |
  | --- | --- |
  | `LogCritical` | Unrecoverable failure — data loss, integrity break, or a job pipeline that terminates. Always paired with a rethrow. |
  | `LogError` | Caught exception at a boundary (external API, DB write, email/payment gateway) or the global exception handler. Always include the exception object. |
  | `LogWarning` | Expected-but-abnormal business situation (unsupported event, missing optional resource, retry about to happen). No throw, or a controlled throw immediately after. |
  | `LogInformation` | Successful business milestone — entity created, phase completed, job step finished. |
  | `LogDebug` / `LogTrace` | Developer diagnostics only. Never ship enabled at an Information-level sink. |

- Enable request logging (`app.UseSerilogRequestLogging()` or the
  built-in `UseHttpLogging`) and register a JSON-formatted sink with
  `FromLogContext` enrichment so scoped properties flow through async
  calls.

**Global exception handler.** Register a single `IExceptionHandler`
and route every unhandled exception through it. Its responsibilities:

1. Log at `LogError`, including the resolved status code, the
   message, and the inner-exception message via named placeholders:

   ```csharp
   _logger.LogError(exception,
       "Handled exception resulted in HTTP {StatusCode}: {Message}. Inner: {InnerException}",
       statusCode, exception.Message, exception.InnerException?.Message);
   ```

2. Write a `ProblemDetails` response with a `trace-id` and
   `instance` in `Extensions` so log entries can be correlated with
   client-side error reports.

3. Only add `innerException` / `stackTrace` to the response when
   `IWebHostEnvironment.IsDevelopment()`. In production those leak
   internal detail to callers.

See the `aspnetcore-clean-architecture` skill for the recommended
`BaseException` hierarchy the handler consumes.

## 3. `.editorconfig`

Every solution ships **one** `.editorconfig` at the repository root
and every contributor's IDE (Rider, Visual Studio, VS Code with the
C# extension) honours it. Copy the companion `editorconfig.template`
from this skill and commit it as `.editorconfig` next to the `.sln`.

The template enforces:

- The five nullable-analyzer diagnostics as **build errors** (see §1).
- `csharp_style_namespace_declarations = file_scoped:warning`
  (file-scoped namespaces).
- `I`-prefix on interfaces; PascalCase for types, properties, events,
  methods.
- Modern-language preferences (collection expressions, primary
  constructors, `using` declarations, null propagation, coalesce
  expression).

Do not create per-project overrides unless a specific project (e.g.
generated code) genuinely needs different rules.

## 4. Async and cancellation

- Every async method that can be cancelled takes a
  `CancellationToken` parameter and forwards it:

  ```csharp
  public interface IProductRepository
  {
      Task<Product?> GetById(string id, CancellationToken cancellationToken);
      Task<Product>  Create(Product product, CancellationToken cancellationToken);
  }
  ```

- MediatR handlers, repositories, HTTP client calls, and EF Core
  queries all take the same token from the top of the request.
  `CancellationToken.None` appears only at real edges (background
  worker start-up, fire-and-forget publish).
- No `Task.Run` / `_ = Task.…` from the request path. If work must
  outlive the request, delegate to a real job scheduler (Hangfire,
  Quartz, hosted service) with a retry policy and an explicit
  failure sink.
- Skip `ConfigureAwait(false)` in ASP.NET Core code. Kestrel has no
  synchronization context, so the call is noise.

## 5. Exception handling

- Define a `BaseException(message, errorCode, statusCode)` and typed
  subclasses (`BadRequestException`, `NotFoundException`,
  `UnAuthorizedException`, `ForbiddenException`,
  `BusinessRuleValidationException`, `InternalServerErrorException`).
  The global handler translates them to `ProblemDetails`.
- `try/catch` only at real boundaries:
  - `Program.cs` bootstrap
  - background job entry points
  - external side-effects (email, payments, storage, third-party HTTP)
- Never `catch (Exception) { }` (silent swallow). If a swallow is
  genuinely required — for example a best-effort welcome email that
  must not fail user registration — log the exception at `LogError`
  and add a comment on the line explaining why the failure is
  non-fatal.
- Never `throw ex;` — it resets the stack trace. Use `throw;` to
  rethrow, or wrap: `throw new DomainException(..., innerException: ex);`.
- Use `catch (SpecificException ex) when (predicate)` when a filter
  makes intent clearer than a nested `if`.

## 6. Secrets and configuration

- One POCO per JSON section, with a `public const string Tag` naming
  the section, consumed via `IOptions<T>` /
  `IOptionsSnapshot<T>` / `IOptionsMonitor<T>`:

  ```csharp
  public class JwtSettings
  {
      public const string Tag = "JwtSettings";
      public string Secret   { get; set; } = "";
      public string Issuer   { get; set; } = "";
      public string Audience { get; set; } = "";
  }

  // In DI:
  services.Configure<JwtSettings>(configuration.GetSection(JwtSettings.Tag));
  ```

- Never sprinkle `configuration["Path:Sub"]` across services — settings
  belong on a strongly-typed POCO.
- **Never commit secrets.** Enable `UserSecretsId` in the API csproj
  for local development, and read production secrets from environment
  variables or a real vault (Azure Key Vault, AWS Secrets Manager,
  HashiCorp Vault). Encryption keys, JWT signing keys, database
  passwords, and third-party API keys never belong in
  `appsettings.json` or any committed file.

## 7. Disposal and HTTP resources

- `using var` for every `IDisposable` / `IAsyncDisposable` at method
  scope. Fall back to a `using` block only when you need explicit
  early disposal (e.g. flushing a `StreamWriter` before reading the
  underlying `MemoryStream`).
- Never `new HttpClient()`. Register with `services.AddHttpClient()`
  (typed or named) so socket handlers are pooled and DNS refreshes
  cleanly.

## 8. Cryptography

- **Passwords**: delegate to ASP.NET Core Identity's
  `PasswordHasher<TUser>` (PBKDF2). Do not implement your own. Do
  not lower the complexity requirements without a written trade-off
  and a compensating control (mandatory MFA, social-only
  authentication, etc.).
- **Symmetric encryption** (AES): generate a fresh random IV per
  message with `RandomNumberGenerator.GetBytes(16)`, prepend it to
  the ciphertext, and store the key in a secret store. A static IV
  makes ciphertext deterministic and leaks plaintext equality — do
  not use one.
- **Key derivation from a passphrase**: use `Rfc2898DeriveBytes`
  (PBKDF2) or `Argon2`. Plain `SHA256` of a passphrase is not a KDF.
- **Random tokens** (refresh tokens, one-time codes):
  `RandomNumberGenerator.GetBytes(N)`, not `System.Random`.
- **JWT**: `ClockSkew = TimeSpan.Zero`; enable `ValidateIssuer`,
  `ValidateAudience`, and `ValidateIssuerSigningKey`; set
  `RequireHttpsMetadata = true` in production; sign with at least
  `HmacSha256`, or an asymmetric algorithm (`RS256` / `ES256`) when
  the signing key must be shared with other services.

## 9. HTTP surface hardening

- `app.UseHttpsRedirection()` and HSTS enabled in the pipeline.
- CORS: **explicit `WithOrigins(...)`** with the allowed hosts.
  `AllowAnyOrigin()` is not an acceptable production configuration,
  even hidden behind an `Enabled` flag.
- Authorisation is opt-in per route (`.RequireAuthorization()`) or
  via a fallback policy that requires an authenticated user.
  "Unauthenticated by default" does not survive a refactor.
- API versioning declared explicitly (`Asp.Versioning`) so old
  clients are never silently upgraded.
- JSON options hardened: `PropertyNameCaseInsensitive = true`,
  `DefaultIgnoreCondition = WhenWritingNull`, and a strict enum
  converter (`JsonStringEnumConverter` if you want string enums;
  otherwise leave enums as integers).
- Admin dashboards (Hangfire, MiniProfiler, Swagger UI) restricted
  to Development or gated behind an admin-only authorisation policy.

## 10. Immutability and access modifiers

- `record` for DTOs, MediatR commands, and MediatR queries — value
  equality plus positional immutability.
- `private readonly` for every injected dependency.
- `{ get; }` or `{ get; init; }` for immutable state;
  `{ get; private set; }` where entity mutation is actually needed.
  No public setters on entities.
- `sealed class` for polymorphic hierarchies you own (webhook payload
  subtypes, notification classes) — prevents accidental inheritance
  from external code.
- `internal class` for infrastructure adapters that must not leak
  across project boundaries.
- No `public` fields anywhere. Ever.

## 11. Validation (cross-reference)

- Shape validation: `System.ComponentModel.DataAnnotations` on
  request DTOs. Fails with `400` before the handler runs.
- Business-rule validation: `IBusinessRule` + `IValidator<TRequest>`
  + a MediatR `BusinessRuleBehavior`. See the
  `aspnetcore-clean-architecture` skill for the full pattern.

## 12. Persistence — strong types across the database boundary

When the persistence layer is Postgres (Npgsql + EF Core), map every
C# enum to a native Postgres `ENUM` type. A free-form string column
(`entity.Property(e => e.Status).HasMaxLength(100)` on a string-
backed enum, or a plain `HasConversion<string>()`) silently accepts
garbage: any INSERT with an unknown value succeeds, a typo in the
C# source lands in the database, and there is no constraint the DBA
can rely on. A native Postgres enum is a first-class type — the
server rejects unknown values at write time, storage is compact
(4 bytes), and adding a label requires an explicit migration that is
easy to review.

Registering an enum takes **two** steps, both required:

**1. Map at the Npgsql data-source level** (wire-level marshalling).
Centralise the mapping and the name translator behind one static
class so the same list is applied to both the raw data source and
the `DbContext` options:

```csharp
using Npgsql;
using Npgsql.EntityFrameworkCore.PostgreSQL.Infrastructure;

public static class NpgsqlDataSourceExtensions
{
    // Single source of truth for enum naming: PascalCase CLR → camelCase PG.
    public static readonly CamelCaseNameTranslator EnumTranslator = new();

    public static NpgsqlDataSourceBuilder MapDomainEnums(
        this NpgsqlDataSourceBuilder builder)
    {
        var t = EnumTranslator;
        builder.MapEnum<OrderStatus>(nameTranslator: t);
        builder.MapEnum<PaymentMethod>(nameTranslator: t);
        // one line per enum
        return builder;
    }

    public static NpgsqlDbContextOptionsBuilder MapDomainEnums(
        this NpgsqlDbContextOptionsBuilder builder)
    {
        var t = EnumTranslator;
        builder.MapEnum<OrderStatus>(nameTranslator: t);
        builder.MapEnum<PaymentMethod>(nameTranslator: t);
        return builder;
    }
}
```

Wire it up when you register the data source and the `DbContext`:

```csharp
services.AddNpgsqlDataSource(connectionString,
    b => b.EnableDynamicJson().MapDomainEnums());

services.AddDbContext<ApplicationDbContext>((sp, options) =>
{
    var dataSource = sp.GetRequiredService<NpgsqlDataSource>();
    options.UseNpgsql(dataSource, b => b.MapDomainEnums());
});
```

**2. Declare at the EF model level** so migrations emit
`CREATE TYPE ... AS ENUM (...)`:

```csharp
protected override void OnModelCreating(ModelBuilder builder)
{
    base.OnModelCreating(builder);

    var t = NpgsqlDataSourceExtensions.EnumTranslator;
    builder.HasPostgresEnum<OrderStatus>(nameTranslator: t);
    builder.HasPostgresEnum<PaymentMethod>(nameTranslator: t);

    builder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
}
```

**Rules for the translator.** Use one shared `INpgsqlNameTranslator`
instance (typically `CamelCaseNameTranslator`) so the Npgsql runtime,
the EF model, and generated migrations agree on the PG identifier
for every label. A per-call translator or a mismatched pair produces
cryptic mapping errors at runtime.

**Rules for evolving enums.**

- *Adding a label* is safe: migration is
  `ALTER TYPE ... ADD VALUE 'newLabel'`.
- *Removing or renaming a label* is expensive: Postgres does not
  support dropping enum values, so you must recreate the type and
  migrate the column. Treat enums as stable value sets.
- If ordering matters (`ORDER BY status`), add new labels with
  `BEFORE` / `AFTER` so the sort order stays meaningful.

**When not to use Postgres enums** *(guidance)*.

- If the schema must ship to non-Postgres engines, use a string
  conversion with a `CHECK` constraint instead so the constraint
  travels with the schema.
- If the value set changes weekly and unpredictably, a lookup table
  with a foreign key is more agile than repeated `ALTER TYPE`.

On engines other than Postgres, apply the same principle: use the
engine's native typed equivalent (SQL Server: a lookup table with a
foreign key and, if desired, a computed check constraint). Never let
the database accept an unconstrained string in place of an enum.

## 13. Anti-patterns — do not copy these, ever

A blunt list. Every one is a security or reliability bug.

- Never load all the database data in a dbset to memory before filtering. Always filter in the query! So as to avoid
memory leaks and performance issues. For example, avoid `dbContext.Products.ToList().Where(p => p.IsActive)`; instead, use `dbContext.Products.Where(p => p.IsActive).ToList()`.
- When registering services in DI, never use reflection with typeof(...) instead, use generic type parameters. For example, avoid `services.AddScoped(typeof(IRepository<>), typeof(Repository<>))`; instead, use `services.AddScoped<IRepository<Foo>, Repository<Foo>>()`.
- Hard-coded API keys, encryption keys, JWT secrets, or database
  passwords in `appsettings.json` (or anywhere in source control).
- Static AES IV. Every message needs its own random IV.
- `AllowAnyOrigin()` in a CORS policy.
- `RequireHttpsMetadata = false` in production.
- `catch { }` — silent exception swallow.
- `!` (null-forgiving) after a nullable-returning API without a
  proven guard immediately above.
- `throw ex;` — resets the stack trace.
- `new HttpClient()` at a call site — leaks sockets under load.
- All password requirements disabled (`RequiredLength = 0`,
  `RequireDigit = false`, ...) without a documented compensating
  control.
- Logging via `$"…{value}"` interpolation — the template becomes an
  opaque string and structured search stops working.
- MediatR handlers or repositories without a `CancellationToken`.
- Business logic in Minimal API endpoint handlers instead of MediatR
  commands.
- Storing enums as free-form `varchar` columns on Postgres — the DB
  will accept any value the C# code happens to emit, including typos.
  Map to a native `ENUM` type instead (§12).

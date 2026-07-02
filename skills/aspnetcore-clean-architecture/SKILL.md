---
name: aspnetcore-clean-architecture
description: Use when scaffolding a new ASP.NET Core solution or adding features to an existing one that must follow Clean Architecture (Domain / Application / Infrastructure / API). Enforces MediatR-based use cases, Minimal APIs organised by feature, route constants, per-layer DI extension methods, command validation via a pipeline behavior, and business-rule objects.
---

# ASP.NET Core Clean Architecture

## Overview

Enforce a strict separation of concerns by splitting the solution into layers.
The API (UI) must never contain business rules or persistence code, and
business rules must never depend on ASP.NET Core, EF Core, or any third-party
SDK. Dependencies always point **inward** toward the `Domain` layer.

```
API (Minimal APIs)  ──▶  Application  ──▶  Domain
                              ▲
                              │
                        Infrastructure
```

- `Domain` depends on **nothing**.
- `Application` depends only on `Domain`.
- `Infrastructure` depends on `Application` (to implement its contracts) and
  on `Domain`.
- The API project depends on `Application` and `Infrastructure` and wires
  everything together at startup.

Throughout this document, `<Product>` is the name of the solution and the
sample entity/use case names (`Product`, `CreateProduct`, ...) are
**illustrative stand-ins** — substitute your own domain concepts when you
apply the pattern.

## Solution structure

Create one C# project per layer plus a test project:

```
<Product>.sln
├── <Product>/                          # ASP.NET Core Web API (Minimal APIs) — composition root
│   ├── Program.cs                      # Only calls extension methods, no logic
│   ├── ApiEndpoints.cs                 # Static class holding ALL route constants
│   ├── ApiVersioning.cs
│   ├── HostingServices.cs              # Host-level extension methods (logging, versioning, ...)
│   ├── GlobalExceptionHandler.cs       # IExceptionHandler implementation
│   ├── CORS/
│   │   ├── CorsOptions.cs
│   │   └── CorsConfigExtensions.cs
│   ├── OpenApi/
│   │   └── SwaggerDefaultValues.cs
│   └── Endpoints/
│       ├── EndpointsExtensions.cs      # Master mapper — MapApiEndpoints()
│       └── <Feature>/                  # One folder per feature (Products, Orders, ...)
│           ├── <Feature>EndpointExtensions.cs   # Feature-level mapper
│           ├── <UseCase>Endpoint.cs             # One endpoint per file
│           └── Models/
│               └── <UseCase>Request.cs
│
├── <Product>.Application/
│   ├── ApplicationServicesExtensions.cs         # public AddApplicationServices()
│   ├── Behaviors/                               # MediatR pipeline behaviors
│   │   ├── IBusinessRule.cs
│   │   ├── IValidator.cs
│   │   ├── BaseValidator.cs
│   │   └── BusinessRuleBehavior.cs
│   ├── Contracts/                               # Interfaces implemented by Infrastructure
│   │   ├── Persistence/                         # Repositories
│   │   │   ├── IRepository.cs
│   │   │   └── I<Entity>Repository.cs
│   │   └── <Concern>/                           # One folder per concern (mailing, payments, ...)
│   ├── Dtos/                                    # Cross-layer data transfer objects
│   ├── Exceptions/                              # BaseException + typed exceptions
│   │   ├── BaseException.cs
│   │   ├── BadRequestException.cs
│   │   ├── NotFoundException.cs
│   │   ├── UnAuthorizedException.cs
│   │   ├── ForbiddenException.cs
│   │   ├── BusinessRuleValidationException.cs
│   │   └── InternalServerErrorException.cs
│   ├── Helpers/
│   │   ├── ErrorCodes.cs                        # Central error-code constants
│   │   ├── PagedListResponse.cs
│   │   ├── GetManyQueryParameters.cs
│   │   └── Settings/                            # Strongly-typed options
│   └── UseCases/
│       └── <Feature>/
│           ├── <UseCase>.cs                     # record Command + nested Handler
│           ├── RequestModels/                   # Use-case-specific inputs (optional)
│           ├── ResponseModels/                  # Use-case-specific outputs (optional)
│           ├── Validation/
│           │   └── <UseCase>Validator.cs
│           └── BusinessRules/
│               └── <Invariant>Rule.cs
│
├── <Product>.Infrastructure/
│   ├── InfrastructureServicesExtensions.cs      # public AddInfrastructureServices()
│   ├── Persistence/
│   │   ├── PersistenceExtensions.cs             # internal AddPersistenceServices()
│   │   ├── ApplicationDbContext.cs
│   │   ├── BaseRepository.cs
│   │   ├── <Entity>Repository.cs
│   │   └── Migrations/                          # EF Core migrations live HERE
│   └── <Concern>/                               # One folder per concern (mailing, payments, ...)
│
├── <Product>.Domain/                            # No dependencies on anything
│   ├── IBaseEntity.cs
│   ├── BaseEntity.cs
│   └── <Entity>.cs
│
└── <Product>.Tests/
    ├── Integration/
    ├── TestFixtures/
    └── Fakes/
```

Notes on the tree:

- `Contracts/<Concern>/` and `Infrastructure/<Concern>/` are grouped by
  concern (mailing, payments, external APIs, security, background jobs, ...).
  Create one subfolder per concern that your project actually has. Only
  `Persistence/` is shown by name because virtually every ASP.NET Core app
  needs it; the rest is up to you.
- **EF Core migrations belong to the Infrastructure layer**, under
  `<Product>.Infrastructure/Persistence/Migrations/`. They are generated
  against the `DbContext`, and the `DbContext` lives in Infrastructure — so
  the migrations must live there too. The API project stays free of any
  persistence-specific folders.

## Step-by-step implementation

### Step 1 — Create the Domain layer

Create a class library (`<Product>.Domain`) with **no** external dependencies
(no NuGet packages, no framework references beyond `netstandard`/`net`).

- Define an `IBaseEntity` interface exposing `Id`, `CreationDate`,
  `LastModified`.
- Define an abstract `BaseEntity` that implements it and is inherited by every
  persisted entity.
- Add plain POCO entity classes. No annotations, no EF attributes.
- Add value objects and enums that describe the business.

### Step 2 — Create the Application layer

Create `<Product>.Application` and reference `<Product>.Domain`. Add the
following NuGet packages:

- `MediatR` — mediator pattern (commands, queries, pipeline behaviors).
- `Microsoft.Extensions.DependencyInjection.Abstractions` — to expose the DI
  extension method.
- `Microsoft.Extensions.Configuration.Abstractions` — for strongly-typed
  settings.

Then add:

1. `Contracts/` — interfaces the Infrastructure must implement, grouped by
   concern (`Persistence/`, plus one folder per external concern the project
   actually uses).
2. `Dtos/` — Data Transfer Objects returned by use cases (never expose Domain
   entities to the API).
3. `Exceptions/` — a `BaseException` with `ErrorCode` and `StatusCode`, then
   typed subclasses (`BadRequestException`, `NotFoundException`,
   `UnAuthorizedException`, `ForbiddenException`,
   `BusinessRuleValidationException`, `InternalServerErrorException`).
4. `Helpers/` — `ErrorCodes` (constants), pagination helpers, settings
   classes.
5. `Behaviors/` — validation infrastructure (see step 5).
6. `UseCases/<Feature>/` — one file per use case. Each file contains a static
   class that groups a `record Command` and its nested `Handler` (see step 4).

### Step 3 — Create the Infrastructure layer

Create `<Product>.Infrastructure` and reference `<Product>.Application` (which
transitively pulls Domain).

- Implement every interface declared in `Application/Contracts/`.
- Group implementations by concern in subfolders matching the contract groups
  (`Persistence/` plus one folder per external concern).
- Any third-party SDK (EF Core, cloud SDKs, payment providers, mail providers,
  identity providers, AI providers, ...) belongs **only** in this project.
- Never let Application or Domain reference these packages directly.
- Place EF Core migrations under `Persistence/Migrations/`. Migration
  generation must target the Infrastructure project.

### Step 4 — Model each use case with MediatR

Each use case lives in its own file inside
`Application/UseCases/<Feature>/` and follows the same shape (the names
`CreateProduct`, `ProductDto`, `IProductRepository` are illustrative — swap
in your own domain):

```csharp
public class CreateProduct
{
    public record Command(string Name, string Description)
        : IRequest<ProductDto>;

    public class Handler : IRequestHandler<Command, ProductDto>
    {
        private readonly IProductRepository _productRepository;
        private readonly ICurrentUserService _currentUserService;
        private readonly ILogger<CreateProduct> _logger;

        public Handler(
            IProductRepository productRepository,
            ICurrentUserService currentUserService,
            ILogger<CreateProduct> logger)
        {
            _productRepository = productRepository;
            _currentUserService = currentUserService;
            _logger = logger;
        }

        public async Task<ProductDto> Handle(
            Command request, CancellationToken cancellationToken)
        {
            // orchestrate the work — call repositories via Application contracts only
        }
    }
}
```

Rules:

- The outer class is a *container*; its name matches the use case
  (`CreateProduct`, `ArchiveProduct`, `RenameProduct`, ...).
- The command is always a `record` implementing `IRequest<TResponse>`.
- Handlers depend on Application contracts (`ICurrentUserService`,
  `IProductRepository`, ...), never on Infrastructure types directly.
- Return DTOs, never Domain entities.
- Throw the typed exceptions from `Application/Exceptions/` — the API layer
  converts them into `ProblemDetails` responses.

### Step 5 — Add command validation with a MediatR pipeline behavior

Validation is split in two layers, each with a well-defined responsibility.

**Layer A — Input (shape) validation** using
`System.ComponentModel.DataAnnotations` on the request DTO:

```csharp
public class CreateProductRequest
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(200, MinimumLength = 2,
        ErrorMessage = "Name must be between 2 and 200 characters")]
    public string Name { get; set; } = string.Empty;

    [Required(ErrorMessage = "Description is required")]
    [StringLength(1000, MinimumLength = 10,
        ErrorMessage = "Description must be between 10 and 1000 characters")]
    public string Description { get; set; } = string.Empty;
}
```

Data annotations validate *shape* only: required fields, string length, email
format, min length, etc. They live on the endpoint request model (or on
`Application/UseCases/<Feature>/RequestModels/`) and fail with `400 Bad
Request` before the handler is invoked.

**Layer B — Business-rule validation** using the `IValidator`/`IBusinessRule`
contracts and a MediatR pipeline behavior.

1. `IBusinessRule` describes a single rule that can be broken:

    ```csharp
    public interface IBusinessRule
    {
        Task<bool> IsBroken();
        string Message { get; }
        string ErrorCode { get; }
        Exception? Exception { get; }
    }
    ```

2. Each rule is a small class that receives its dependencies via the
   constructor:

    ```csharp
    public class ProductNameMustBeUniqueRule : IBusinessRule
    {
        private readonly IProductRepository _productRepository;
        private readonly string _name;

        public ProductNameMustBeUniqueRule(
            IProductRepository productRepository, string name)
        {
            _productRepository = productRepository;
            _name = name;
        }

        public string Message =>
            "A product with the same name already exists.";
        public string ErrorCode => ErrorCodes.ProductNameNotUnique;
        public Exception? Exception => new BadRequestException(Message, ErrorCode);

        public async Task<bool> IsBroken()
        {
            if (string.IsNullOrWhiteSpace(_name)) return true;
            return await _productRepository.ExistsWithName(_name);
        }
    }
    ```

3. `IValidator<TRequest>` and `BaseValidator<TRequest>` provide the shared
   plumbing:

    ```csharp
    public interface IValidator<TRequest>
    {
        Task Validate(TRequest request);
        Task CheckRule(IBusinessRule rule);
    }

    public abstract class BaseValidator<TRequest> : IValidator<TRequest>
        where TRequest : MediatR.IBaseRequest
    {
        public async Task CheckRule(IBusinessRule rule)
        {
            if (await rule.IsBroken())
            {
                if (rule.Exception != null) throw rule.Exception;
                throw new BusinessRuleValidationException(rule);
            }
        }

        public abstract Task Validate(TRequest request);
    }
    ```

4. One validator per command, in `UseCases/<Feature>/Validation/`:

    ```csharp
    public class CreateProductValidator : BaseValidator<CreateProduct.Command>
    {
        private readonly IProductRepository _productRepository;
        private readonly ICurrentUserService _currentUserService;

        public CreateProductValidator(
            IProductRepository productRepository,
            ICurrentUserService currentUserService)
        {
            _productRepository = productRepository;
            _currentUserService = currentUserService;
        }

        public override async Task Validate(CreateProduct.Command request)
        {
            if (!_currentUserService.IsAuthenticated)
                throw new UnAuthorizedException("User not authenticated");

            await CheckRule(new ProductNameMustBeUniqueRule(
                _productRepository, request.Name));
        }
    }
    ```

5. A single MediatR pipeline behavior runs the matching validator before
   every handler:

    ```csharp
    public class BusinessRuleBehavior<TRequest, TResponse>
        : IPipelineBehavior<TRequest, TResponse>
        where TRequest : IRequest<TResponse>
    {
        private readonly IValidator<TRequest> _validator;

        public BusinessRuleBehavior(IValidator<TRequest> validator)
            => _validator = validator;

        public async Task<TResponse> Handle(
            TRequest request,
            RequestHandlerDelegate<TResponse> next,
            CancellationToken cancellationToken)
        {
            await _validator.Validate(request);
            return await next();
        }
    }
    ```

6. Register the validator and the pipeline behavior in
   `ApplicationServicesExtensions`:

    ```csharp
    private static void RegisterValidators(IServiceCollection services)
    {
        services.AddScoped<
            IValidator<CreateProduct.Command>, CreateProductValidator>();
    }

    private static void RegisterPipelineBehaviors(IServiceCollection services)
    {
        services.AddScoped(
            typeof(IPipelineBehavior<CreateProduct.Command, ProductDto>),
            typeof(BusinessRuleBehavior<CreateProduct.Command, ProductDto>));
    }
    ```

**Guidelines for writing rules and validators**

- One rule = one invariant. Prefer several small rules over one big rule with
  many branches.
- Rules must return a `bool` from `IsBroken()` and carry their own `Message`,
  `ErrorCode`, and (optionally) the `Exception` to throw. This gives the
  client a stable error code and a human-readable message.
- Every error code is a constant in `Application/Helpers/ErrorCodes.cs` —
  never hard-code strings at the throw site.
- The validator's `Validate` method is the only place that decides *which*
  rules apply, in *which* order, for a given command. Handlers must stay free
  of validation logic.
- Business-rule failures raise `BusinessRuleValidationException` (or the
  rule's supplied `Exception`), which the global exception handler maps to
  `400 Bad Request` with `ProblemDetails`.
- Cross-cutting checks that are not domain rules (authentication,
  authorisation) are handled by ASP.NET Core middleware or
  `RequireAuthorization()` on the endpoint, not by validators.

### Step 6 — Expose the API with Minimal APIs

The web project uses **Minimal APIs only** — no MVC controllers for feature
endpoints.

**Route constants** — All routes live in `ApiEndpoints.cs` as `const string`
values, grouped by feature and versioned:

```csharp
public static class ApiEndpoints
{
    private const string ApiBase = "api/v{version:apiVersion}";

    public static class Products
    {
        public const string Base    = $"{ApiBase}/products";
        public const string ById    = $"{Base}/{{productId}}";
        public const string Archive = $"{Base}/{{productId}}/archive";
    }
}
```

Endpoints reference these constants directly — no inline strings.

**One endpoint per file.** Each file is a static class that defines the
endpoint's `Name`, its mapping extension, its route (from `ApiEndpoints`),
its OpenAPI metadata, and its authorisation:

```csharp
public static class CreateProductEndpoint
{
    public const string Name = "CreateProduct";

    public static IEndpointRouteBuilder MapCreateProductEndpoint(
        this IEndpointRouteBuilder app)
    {
        app.MapPost(ApiEndpoints.Products.Base, async (
                [FromBody] CreateProductRequest request,
                [FromServices] IMediator mediator) =>
            {
                var result = await mediator.Send(new CreateProduct.Command(
                    request.Name, request.Description));

                return Results.Created(
                    $"{ApiEndpoints.Products.Base}/{result.Id}", result);
            })
            .WithName(Name)
            .RequireAuthorization()
            .Produces<ProductDto>(StatusCodes.Status201Created)
            .Produces(StatusCodes.Status401Unauthorized)
            .Produces(StatusCodes.Status400BadRequest)
            .WithTags(ProductsEndpointExtensions.Tag);

        return app;
    }
}
```

Handlers must be *thin*: parse the request, call `IMediator`, translate the
result to `Results.*`. No business logic. No repository calls.

**Feature-level mapper.** Each feature folder has a
`<Feature>EndpointExtensions.cs` that maps every endpoint in the feature and
exposes the OpenAPI `Tag`:

```csharp
public static class ProductsEndpointExtensions
{
    public static IEndpointRouteBuilder MapProductsEndpoints(
        this IEndpointRouteBuilder app)
    {
        app.MapCreateProductEndpoint();
        // app.MapGetProductByIdEndpoint();
        // app.MapArchiveProductEndpoint();
        return app;
    }

    public static string[] Tag { get; set; } = { "Products" };
}
```

**Top-level mapper.** `Endpoints/EndpointsExtensions.cs` composes all feature
mappers under a single versioned group:

```csharp
public static class EndpointsExtensions
{
    public static IEndpointRouteBuilder MapApiEndpoints(
        this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("").WithApiVersionSet(ApiVersioning.VersionSet!);

        // one call per feature
        group.MapProductsEndpoints();

        return app;
    }
}
```

### Step 7 — Write DI extension methods per layer

Each layer owns a static class with a **single public** entry-point extension
method that registers everything the layer contributes. Any internal helpers
used to organise the registrations are kept `private static` (or
`internal static` for cross-file helpers such as `PersistenceExtensions`).

**Application layer:**

```csharp
public static class ApplicationServicesExtensions
{
    public static IServiceCollection AddApplicationServices(
        this IServiceCollection services)
    {
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));

        RegisterValidators(services);
        RegisterPipelineBehaviors(services);

        return services;
    }

    private static void RegisterValidators(IServiceCollection services) { /* ... */ }
    private static void RegisterPipelineBehaviors(IServiceCollection services) { /* ... */ }
}
```

**Infrastructure layer** (public entry-point + internal per-concern helpers):

```csharp
public static class InfrastructureServicesExtensions
{
    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services,
        IConfiguration configuration,
        string environment)
    {
        // Bind strongly-typed settings for this layer
        services.Configure<SomeSettings>(configuration.GetSection(SomeSettings.Tag));

        // Persistence — see PersistenceExtensions below
        services.AddPersistenceServices(configuration, environment);

        // One line per concern: register the interface (from Application/Contracts)
        // against the Infrastructure implementation.
        services.AddScoped<ISomeConcernService, SomeConcernService>();

        return services;
    }
}

internal static class PersistenceExtensions
{
    internal static IServiceCollection AddPersistenceServices(
        this IServiceCollection services,
        IConfiguration configuration,
        string environment)
    {
        // DbContext + repositories + connection setup.
        // Configure the DbContext so migrations resolve to this assembly:
        //     builder.MigrationsAssembly("<Product>.Infrastructure");
        return services;
    }
}
```

**API layer.** `HostingServices.cs` groups host-level extensions (logging,
API versioning, exception handling, and any optional cross-cutting concerns
your project happens to use — observability, background jobs, feature
flags, ...). `Program.cs` becomes a short pipeline of extension calls:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder
    .AddSerilog()                    // structured logging
    .AddApiVersioning()
    .AddGlobalExceptionHandler();
    // Add optional cross-cutting concerns here (observability, background jobs, ...).

builder.AddCoreServices();           // internally calls AddApplicationServices()
builder.ConfigureServices();         // internally calls AddInfrastructureServices() + auth setup

var app = builder.Build();
app.ConfigureApp();                  // pipeline + MapApiEndpoints()
app.Run();
```

Rules for extension methods:

- Exactly **one** `public static` entry-point per class. Every other
  registration helper is `private static` or `internal static` and never
  called from another project.
- Extension methods return the builder they extend (`IServiceCollection`,
  `WebApplicationBuilder`, `IEndpointRouteBuilder`) so calls can be chained.
- Extension classes are named `<Concern>Extensions` (`PersistenceExtensions`,
  `CorsConfigExtensions`, `ProductsEndpointExtensions`).
- Cross-cutting host extensions (logging, versioning, Swagger, exception
  handling) live in `HostingServices.cs`.

### Step 8 — Wire the global exception handler

Register a `GlobalExceptionHandler : IExceptionHandler` in the API project.
It should:

- Inspect the exception; if it derives from `BaseException`, use its
  `StatusCode` and `ErrorCode` for the `ProblemDetails.Status` and `Type`.
- Otherwise fall back to `500 Internal Server Error` and a generic error
  code.
- Add `trace-id` and `instance` to `ProblemDetails.Extensions` for
  correlation.
- Only expose `innerException` / `stackTrace` in Development.

Enable it in `Program.cs` via `app.UseExceptionHandler()` and register it
with `services.AddExceptionHandler<GlobalExceptionHandler>()` +
`services.AddProblemDetails(...)`.

Give every error code a stable name in `Application/Helpers/ErrorCodes.cs`,
following the convention `<FEATURE>-NNN`:

```csharp
public static class ErrorCodes
{
    public const string ProductNameNotUnique = "PRODUCT-001";
    // ...
}
```

### Step 9 — Add tests

Create `<Product>.Tests` with three folders:

- `Integration/` — end-to-end tests hitting real endpoints through
  `WebApplicationFactory`.
- `TestFixtures/` — shared fixtures (test containers, factories, auth
  helpers).
- `Fakes/` — in-memory or scripted fakes for external services behind
  `Application/Contracts/` interfaces, swapped in via `services.Replace(...)`
  in the test factory.

## Checklist when adding a new feature

- [ ] Add the domain entity (if new) to `<Product>.Domain`.
- [ ] Add or extend a repository interface in
      `Application/Contracts/Persistence/`.
- [ ] Implement the repository in `Infrastructure/Persistence/` and register
      it in `PersistenceExtensions`.
- [ ] Create the use case at `Application/UseCases/<Feature>/<UseCase>.cs`
      (record command + handler).
- [ ] If business rules apply, add rule classes under `BusinessRules/` and a
      validator under `Validation/`, then register both in
      `ApplicationServicesExtensions`.
- [ ] Add the request DTO with data annotations under the endpoint's
      `Models/` folder.
- [ ] Add the route constant to `ApiEndpoints.<Feature>`.
- [ ] Create one endpoint file per HTTP verb in `Endpoints/<Feature>/`, wire
      it in `<Feature>EndpointExtensions.Map<Feature>Endpoints`, and make sure
      the feature mapper is called from `EndpointsExtensions.MapApiEndpoints`.
- [ ] Add error codes to `Application/Helpers/ErrorCodes.cs` if the use case
      introduces new failure modes.
- [ ] Add an integration test in `<Product>.Tests/Integration/<Feature>/`.

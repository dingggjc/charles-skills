---
name: backend-dotnet
description: >
  Apply this skill for any .NET backend task — scaffolding a new project, creating
  commands, queries, handlers, controllers, entities, DTOs, validators, or wiring
  dependencies. Trigger whenever the user mentions .NET, C#, Clean Architecture,
  MediatR, CQRS, EF Core, handlers, commands, queries, controllers, or asks to
  build/add/create anything on the backend side. Also trigger when the user says
  "new feature", "add an endpoint", "create a handler", "scaffold a backend", or
  "add a command/query".
---

# Backend .NET Skill

Standards and scaffold guide for all .NET Clean Architecture projects.

## Stack

| Layer | Tool |
|---|---|
| Framework | .NET 10, ASP.NET Core Web API |
| Architecture | Clean Architecture |
| CQRS | MediatR |
| ORM | EF Core |
| Database | PostgreSQL (Neon) |
| Validation | FluentValidation |
| Auth | JWT (cookie-based) |
| Password Hashing | ASP.NET Core Identity |

---

## Solution Structure

```
Solution/
├── Solution.Domain/          ← Entities, enums, base classes only
├── Solution.Application/     ← CQRS handlers, interfaces, validators, DTOs
├── Solution.Infrastructure/  ← JWT, email, external services
├── Solution.Persistence/     ← EF Core, DbContext, migrations, seeders, services
└── Solution.WebApi/          ← Controllers, helpers, DI wiring, startup
```

**Layer rules:**
- `Domain` has zero dependencies on other layers
- `Application` depends only on `Domain`
- `Infrastructure` and `Persistence` depend on `Application`
- `WebApi` depends on all layers for DI wiring only
- Never reference `WebApi` or `Persistence` from `Application`
- Never reference `Infrastructure` or `Persistence` from `Domain`

---

## Domain Layer

### Folder Structure

```
Solution.Domain/
├── Common/
│   ├── BaseEntity.cs
│   └── PhilippineTime.cs (or TimeProvider equivalent)
├── Constants/
│   └── Roles.cs
├── Entities/
│   └── [Entity].cs
└── Enums/
    └── [Enum].cs
```

### BaseEntity

```csharp
namespace Solution.Domain.Common
{
    public abstract class BaseEntity
    {
        public Guid Id { get; set; } = Guid.NewGuid();
        public DateTime CreatedAt { get; set; }
        public DateTime? UpdatedAt { get; set; }
        public bool IsDeleted { get; set; } = false;
    }
}
```

### Entity Standards

- Always inherit from `BaseEntity`
- No business logic in entities — pure data only
- Navigation properties are allowed
- No DTOs, no MediatR references, no EF Core references in Domain

```csharp
namespace Solution.Domain.Entities
{
    public class Category : BaseEntity
    {
        public string Name { get; set; } = string.Empty;
        public string? CreatedById { get; set; }
        public ApplicationUser? CreatedBy { get; set; }
    }
}
```

### Roles Constants

```csharp
namespace Solution.Domain.Constants
{
    public static class Roles
    {
        public const string SuperAdmin = "SuperAdmin";
        public const string Admin = "Admin";
        public const string Cashier = "Cashier";
    }
}
```

---

## Application Layer

### Folder Structure

```
Solution.Application/
├── Common/
│   ├── Models/
│   │   ├── BaseResult.cs
│   │   ├── CommandResult.cs
│   │   ├── QueryPageResult.cs
│   │   ├── PageResult.cs
│   │   └── Error.cs
│   └── DTO/
│       ├── ExtendedParameters.cs
│       └── ExtendedPageDetails.cs
├── Interfaces/
│   ├── IApplicationDbContext.cs
│   ├── ICurrentUserService.cs
│   ├── IJwtTokenGenerator.cs
│   └── ITenantProvider.cs
├── Categories/               ← plural domain name, one folder per domain
│   ├── Commands/
│   │   ├── CreateCategoryCommand.cs
│   │   └── CreateCategoryHandler.cs   ← co-located with its command
│   ├── Queries/
│   │   ├── GetCategoriesQuery.cs
│   │   └── GetCategoriesHandler.cs    ← co-located with its query
│   ├── Validators/
│   │   └── CreateCategoryValidator.cs
│   └── DTO/
│       ├── CreateCategoryDTO.cs
│       └── CategoryResultDTO.cs
├── Products/
│   ├── Commands/
│   ├── Queries/
│   ├── Validators/
│   └── DTO/
└── Users/
    ├── Commands/
    ├── Queries/
    ├── Validators/
    └── DTO/

**Scaffolding rule:** When adding a new feature, ask for the domain name first.
Use the plural form for the folder (`Categories`, `Products`, `Users`),
and singular for commands/handlers/DTOs (`CreateCategoryCommand`, `CategoryResultDTO`).
```

**Feature rules:**
- Handlers are always co-located with their command or query — never a separate `Handlers/` folder
- Validators live inside the feature folder, not a global `Validators/` folder
- One command/query per file, one handler per file

---

## Common Models

### BaseResult

```csharp
namespace Solution.Application.Common.Models
{
    public class BaseResult<T>
    {
        public ValidationError? ValidatorError { get; set; }
        public bool IsSuccess => ValidatorError == null;
        public HttpStatusCode StatusCode { get; set; } = HttpStatusCode.OK;
        public T? Response { get; set; }
    }
}
```

### CommandResult — for write operations

```csharp
namespace Solution.Application.Common.Models
{
    public class CommandResult<T> : BaseResult<T> {}
}
```

### QueryPageResult — for paginated reads

```csharp
namespace Solution.Application.Common.Models
{
    public class QueryPageResult<T> : BaseResult<T>
    {
        public ExtendedPageDetails? PageDetails { get; set; }
    }
}
```

### Error and ValidationError

```csharp
namespace Solution.Application.Common.Models
{
    public class Error
    {
        public int StatusCode { get; set; } = 400;
        public string Title { get; set; } = "Bad Request";
        public string Message { get; set; } = "Something went wrong";
        public object? Payload { get; set; }
    }

    public class ValidationError : Error
    {
        public ValidationError()
        {
            Title = "Validation Failed";
        }
        public Dictionary<string, string[]>? Errors { get; set; }
    }
}
```

### PageResult — pagination engine

```csharp
namespace Solution.Application.Common.Models
{
    public class PageResult<T>
    {
        public ExtendedPageDetails ExtendedPageDetails { get; set; }
        public List<T> DataResult { get; set; }

        public PageResult(List<T> items, int count, int pageNumber, int pageSize)
        {
            ExtendedPageDetails = new ExtendedPageDetails
            {
                TotalCount = count,
                PageSize = pageSize,
                CurrentPage = pageNumber,
                TotalPages = (int)Math.Ceiling(count / (double)pageSize),
                HasPrevious = pageNumber > 1,
                HasNext = pageNumber < (int)Math.Ceiling(count / (double)pageSize)
            };

            DataResult = items.ToList();
        }

        public static async Task<PageResult<T>> ToPagedListAsync(
            IQueryable<T> source, int pageNumber, int pageSize)
        {
            var count = await source.CountAsync();
            var items = await source
                .Skip((pageNumber - 1) * pageSize)
                .Take(pageSize)
                .ToListAsync();
            return new PageResult<T>(items, count, pageNumber, pageSize);
        }

        public static async Task<PageResult<T>> EnumerableToPagedListAsync(
            IEnumerable<T> source, int pageNumber, int pageSize)
        {
            var count = source.Count();
            var items = await Task.Run(() =>
                source.Skip((pageNumber - 1) * pageSize).Take(pageSize).ToList());
            return new PageResult<T>(items, count, pageNumber, pageSize);
        }
    }
}
```

### ExtendedParameters

```csharp
namespace Solution.Application.Common.DTO
{
    public class ExtendedParameters
    {
        private const int maxPageSize = 500;
        public int PageNumber { get; set; } = 1;
        private int _pageSize = 10;
        public int PageSize
        {
            get => _pageSize;
            set => _pageSize = value > maxPageSize ? maxPageSize : value;
        }
    }
}
```

### ExtendedPageDetails

```csharp
namespace Solution.Application.Common.DTO
{
    public class ExtendedPageDetails
    {
        public int CurrentPage { get; set; }
        public int TotalPages { get; set; }
        public int PageSize { get; set; }
        public int TotalCount { get; set; }
        public bool HasPrevious { get; set; }
        public bool HasNext { get; set; }
    }
}
```

---

## ICurrentUserService — never pass UserId/UserRole in commands

Never pass `UserId` or `UserRole` as command parameters. Always inject `ICurrentUserService` into the handler:

```csharp
// ❌ wrong
public record CreateCategoryCommand(
    CreateCategoryDTO Category,
    string UserId,
    string UserRole
) : IRequest<CommandResult<Guid>>;

// ✅ correct
public record CreateCategoryCommand(
    CreateCategoryDTO Category
) : IRequest<CommandResult<Guid>>;
```

```csharp
// Interface
namespace Solution.Application.Interfaces
{
    public interface ICurrentUserService
    {
        string? UserId { get; }
        string? UserRole { get; }
    }
}
```

```csharp
// Usage in handler
public class CreateCategoryHandler : IRequestHandler<CreateCategoryCommand, CommandResult<Guid>>
{
    private readonly IApplicationDbContext _context;
    private readonly IValidator<CreateCategoryDTO> _validator;
    private readonly ICurrentUserService _currentUser;

    public CreateCategoryHandler(
        IApplicationDbContext context,
        IValidator<CreateCategoryDTO> validator,
        ICurrentUserService currentUser)
    {
        _context = context;
        _validator = validator;
        _currentUser = currentUser;
    }

    public async Task<CommandResult<Guid>> Handle(
        CreateCategoryCommand request,
        CancellationToken cancellationToken)
    {
        var result = new CommandResult<Guid>();

        if (_currentUser.UserRole != Roles.Admin)
        {
            result.ValidatorError = new ValidationError
            {
                Message = "Unauthorized. Only Admin can create categories."
            };
            result.StatusCode = HttpStatusCode.Forbidden;
            return result;
        }

        var validationResult = await _validator.ValidateAsync(
            request.Category, cancellationToken);

        if (!validationResult.IsValid)
        {
            result.ValidatorError = new ValidationError
            {
                Errors = validationResult.Errors
                    .GroupBy(e => e.PropertyName)
                    .ToDictionary(
                        g => g.Key,
                        g => g.Select(e => e.ErrorMessage).ToArray())
            };
            result.StatusCode = HttpStatusCode.BadRequest;
            return result;
        }

        var exists = _context.Categories.Any(c =>
            c.Name.ToLower() == request.Category.Name.ToLower() && !c.IsDeleted);

        if (exists)
        {
            result.ValidatorError = new ValidationError
            {
                Message = "Category name already exists."
            };
            result.StatusCode = HttpStatusCode.BadRequest;
            return result;
        }

        var category = new Category
        {
            Name = request.Category.Name,
            CreatedById = _currentUser.UserId
        };

        await _context.Categories.AddAsync(category, cancellationToken);
        await _context.SaveChangesAsync(cancellationToken);

        result.Response = category.Id;
        result.StatusCode = HttpStatusCode.Created;
        return result;
    }
}
```

---

## Query Handler Pattern

```csharp
// Query
public record GetCategoriesQuery(
    string? SearchKey,
    ExtendedParameters ExtendedParameters
) : IRequest<QueryPageResult<List<CategoryResultDTO>>>;
```

```csharp
// Handler co-located in Queries/
public class GetCategoriesHandler : IRequestHandler<GetCategoriesQuery, QueryPageResult<List<CategoryResultDTO>>>
{
    private readonly IApplicationDbContext _context;

    public GetCategoriesHandler(IApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<QueryPageResult<List<CategoryResultDTO>>> Handle(
        GetCategoriesQuery request,
        CancellationToken cancellationToken)
    {
        var query = _context.Categories
            .Where(c => !c.IsDeleted)
            .AsNoTracking();

        if (!string.IsNullOrWhiteSpace(request.SearchKey))
        {
            var search = request.SearchKey.ToLower();
            query = query.Where(c => c.Name.ToLower().Contains(search));
        }

        query = query.OrderByDescending(c => c.CreatedAt);

        var pagedResult = await PageResult<Category>.ToPagedListAsync(
            query,
            request.ExtendedParameters.PageNumber,
            request.ExtendedParameters.PageSize);

        var userIds = pagedResult.DataResult
            .Where(c => c.CreatedById != null)
            .Select(c => c.CreatedById!)
            .Distinct()
            .ToList();

        var users = await _context.Users
            .Where(u => userIds.Contains(u.Id))
            .Select(u => new { u.Id, u.FirstName, u.LastName })
            .ToListAsync(cancellationToken);

        return new QueryPageResult<List<CategoryResultDTO>>
        {
            Response = pagedResult.DataResult.Select(c =>
            {
                var creator = users.FirstOrDefault(u => u.Id == c.CreatedById);
                return new CategoryResultDTO
                {
                    Id = c.Id,
                    Name = c.Name,
                    CreatedAt = c.CreatedAt,
                    CreatedBy = creator != null
                        ? $"{creator.FirstName} {creator.LastName}"
                        : "Unknown"
                };
            }).ToList(),
            PageDetails = pagedResult.ExtendedPageDetails,
            StatusCode = HttpStatusCode.OK
        };
    }
}
```

---

## Controller Pattern

- One controller per feature
- Only inject `IMediator` — nothing else
- Never put business logic in controllers
- Always use `ResultHelper.ToObjectResult` for responses
- Never pass `UserId` or `UserRole` from controller to command — handler gets it from `ICurrentUserService`

```csharp
[ApiController]
[Route("[controller]")]
[Authorize]
public class CategoriesController : ControllerBase
{
    private readonly IMediator _mediator;

    public CategoriesController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost]
    public async Task<ActionResult<CommandResult<Guid>>> Create(
        [FromBody] CreateCategoryDTO model)
    {
        var result = await _mediator.Send(new CreateCategoryCommand(model));
        return ResultHelper.ToObjectResult(result, result.StatusCode);
    }

    [HttpGet]
    public async Task<ActionResult<QueryPageResult<List<CategoryResultDTO>>>> Get(
        [FromQuery] string? searchKey,
        [FromQuery] ExtendedParameters extendedParameters)
    {
        var result = await _mediator.Send(
            new GetCategoriesQuery(searchKey, extendedParameters));
        return ResultHelper.ToObjectResult(result, result.StatusCode);
    }

    [HttpPut("{id}")]
    public async Task<ActionResult<CommandResult<Guid>>> Update(
        Guid id, [FromBody] UpdateCategoryDTO model)
    {
        var result = await _mediator.Send(new UpdateCategoryCommand(id, model));
        return ResultHelper.ToObjectResult(result, result.StatusCode);
    }

    [HttpDelete("{id}")]
    public async Task<ActionResult<CommandResult<Guid>>> Delete(Guid id)
    {
        var result = await _mediator.Send(new DeleteCategoryCommand(id));
        return ResultHelper.ToObjectResult(result, result.StatusCode);
    }
}
```

---

## FluentValidation Pattern

Validators live inside the feature's `Validators/` folder:

```csharp
// Application/Categories/Validators/CreateCategoryValidator.cs
public class CreateCategoryValidator : AbstractValidator<CreateCategoryDTO>
{
    public CreateCategoryValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Category name is required.")
            .MaximumLength(100).WithMessage("Category name must not exceed 100 characters.");
    }
}
```

---

## Naming Conventions

| Type | Convention | Example |
|---|---|---|
| Classes | PascalCase | `CategoryHandler` |
| Interfaces | `I` prefix + PascalCase | `IApplicationDbContext` |
| Methods | PascalCase | `Handle`, `ToPagedListAsync` |
| Properties | PascalCase | `IsSuccess`, `PageSize` |
| Local variables | camelCase | `validationResult`, `pagedResult` |
| Private fields | camelCase with `_` prefix | `_context`, `_currentUser` |
| Constants | camelCase | `maxPageSize` |
| Commands | `[Action][Entity]Command` | `CreateCategoryCommand`, `DeleteProductCommand` |
| Queries | `Get[Entity/Entities]Query` | `GetCategoriesQuery`, `GetCategoryByIdQuery` |
| Handlers | `[Action][Entity]Handler` | `CreateCategoryHandler`, `GetCategoriesHandler` |
| DTOs | `[Action][Entity]DTO` / `[Entity]ResultDTO` | `CreateCategoryDTO`, `CategoryResultDTO` |
| Validators | `[Action][Entity]Validator` | `CreateCategoryValidator` |
| Controllers | `[Entities]Controller` | `CategoriesController`, `ProductsController` |

---

## General Code Quality

- No inline comments — code should be self-explanatory, never narrate what the code is doing
- Remove all unused `using` statements — only import what you use
- No business logic in controllers — mediator call + return only
- No business logic in entities — pure data only
- Never pass `UserId` or `UserRole` as command/query parameters
- Always use `ICurrentUserService` for current user context in handlers
- Always use `AsNoTracking()` for read-only queries
- Always filter soft-deleted records with `!c.IsDeleted`
- Use object initializer syntax — never set properties line by line after construction
- One command/query + its handler per file pair, co-located in the same folder

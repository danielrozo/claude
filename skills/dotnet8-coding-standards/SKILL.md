---
name: dotnet8-coding-standards
description: Use when writing C# code, implementing modern .NET 8 features, or reviewing code for best practices.
---

# .NET 8 Coding Standards

Use this skill when writing C# code, implementing modern .NET 8 features, or reviewing code for best practices.

## Primary Constructors

- **ALWAYS** use primary constructors for dependency injection (C# 12/.NET 8+)
- **ALWAYS** use primary constructors for controllers, handlers, services, and repositories
- **PREFER** primary constructors over traditional constructor patterns
- **ONLY** use traditional constructors when you need complex initialization logic

```csharp
// ✅ Correct - Primary Constructor with logging
public class ResourceController(IMediator mediator, ILogger<ResourceController> logger)
    : ApiControllerBase
{
    [HttpGet("{id}")]
    public async Task<IActionResult> GetResource(Guid id, CancellationToken cancellationToken = default)
    {
        logger.LogInformation("Retrieving resource with ID {ResourceId}", id);

        var result = await mediator.Send(new GetResourceQuery(id), cancellationToken);

        if (result.IsSuccess)
        {
            logger.LogInformation("Successfully retrieved resource with ID {ResourceId}", id);
            return HandleSuccess(result.Data!);
        }

        logger.LogWarning("Failed to retrieve resource with ID {ResourceId}", id);
        return CreateProblemDetailsResponse("Failed to retrieve resource", "Resource not found");
    }
}

// ✅ Correct - Primary Constructor with multiple dependencies including logging
public class ResourceService(
    IResourceRepository repository,
    IResourceDomainService domainService,
    ILogger<ResourceService> logger) : IResourceService
{
    public async Task<ProcessResult<Resource>> CreateResourceAsync(CreateResourceRequest request)
    {
        logger.LogInformation("Creating resource with name {ResourceName}", request.Name);

        // Implementation with proper logging
        var result = await domainService.CreateResourceAsync(request.Name, request.Description);

        if (result.IsSuccess)
        {
            logger.LogInformation("Successfully created resource with ID {ResourceId}", result.Data?.Id);
        }
        else
        {
            logger.LogWarning("Failed to create resource with name {ResourceName}", request.Name);
        }

        return result;
    }
}

// ❌ Incorrect - Traditional Constructor
public class ResourceController : ApiControllerBase
{
    private readonly IMediator _mediator;

    public ResourceController(IMediator mediator)
    {
        _mediator = mediator;
    }

    // ... rest of implementation
}
```

## Performance Guidelines

### Database Queries

- **ALWAYS** measure query performance with profiling
- **ALWAYS** use `AsNoTracking()` for read-only operations
- **ALWAYS** limit result sets with pagination
- **CONSIDER** caching for frequently accessed reference data

### Memory Management

- **ALWAYS** dispose of resources properly
- **ALWAYS** use `using` statements for disposable objects
- **ALWAYS** avoid memory leaks in long-running operations

## Security Guidelines

### Authentication & Authorization

- **ALWAYS** use JWT Bearer tokens
- **ALWAYS** validate tokens on all endpoints
- **ALWAYS** use role-based authorization where appropriate
- **NEVER** expose sensitive data in logs

### Input Validation

- **ALWAYS** validate all input parameters
- **ALWAYS** sanitize user input
- **ALWAYS** use parameterized queries
- **NEVER** trust client-side validation alone

## Documentation Standards

### API Documentation

- **ALWAYS** document all endpoints with Swagger/OpenAPI
- **ALWAYS** include example requests and responses
- **ALWAYS** document error codes and messages
- **ALWAYS** keep documentation up to date

### Code Documentation

- **ALWAYS** document complex business logic
- **ALWAYS** use XML documentation for public APIs
- **ALWAYS** explain "why" not just "what"
- **NEVER** document obvious code

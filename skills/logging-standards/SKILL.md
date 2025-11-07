---
name: logging-standards
description: Use when implementing logging, structured logging, or debugging issues.
---

# Logging Standards

Use this skill when implementing logging, structured logging, or debugging issues.

## General Logging Standards

- **ALWAYS** use structured logging with Serilog
- **ALWAYS** include correlation IDs
- **ALWAYS** log at appropriate levels (Error, Warning, Information, Debug)
- **NEVER** log sensitive data (passwords, tokens, PII)

## Controller Logging Requirements

- **ALWAYS** inject `ILogger<T>` in primary constructor
- **ALWAYS** log operation start with Information level
- **ALWAYS** log success with relevant data (counts, IDs, etc.)
- **ALWAYS** log failures with Warning level
- **ALWAYS** use structured logging with parameters

```csharp
// ✅ Correct - Controller with proper logging
public class ResourcesController(IMediator mediator, ILogger<ResourcesController> logger)
    : ApiControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetResources(CancellationToken cancellationToken = default)
    {
        logger.LogInformation("Retrieving all resources");

        var processResult = await mediator.Send(new GetResourcesQuery(), cancellationToken);

        if (processResult.IsSuccess)
        {
            logger.LogInformation("Successfully retrieved {Count} resources",
                processResult.Data?.Count() ?? 0);
            return HandleSuccess(processResult.Data!);
        }

        logger.LogWarning("Failed to retrieve resources");
        return CreateProblemDetailsResponse("Failed to retrieve resources", "An error occurred");
    }
}

// ❌ Incorrect - No logging
public class ResourcesController(IMediator mediator) : ApiControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetResources(CancellationToken cancellationToken = default)
    {
        var processResult = await mediator.Send(new GetResourcesQuery(), cancellationToken);
        return processResult.IsSuccess
            ? HandleSuccess(processResult.Data!)
            : CreateProblemDetailsResponse(...);
    }
}
```

## Handler Logging Requirements

- **ALWAYS** inject `ILogger<T>` in primary constructor
- **ALWAYS** log database operations with Debug level
- **ALWAYS** log success with relevant data (counts, IDs, etc.)
- **ALWAYS** log exceptions with Error level and full exception details
- **ALWAYS** use structured logging with parameters

```csharp
// ✅ Correct - Handler with proper logging
public class GetResourcesQueryHandler(
    ApplicationDbContext dbContext,
    ILogger<GetResourcesQueryHandler> logger)
    : IRequestHandler<GetResourcesQuery, ProcessResult<IEnumerable<ResourceType>>>
{
    public async Task<ProcessResult<IEnumerable<ResourceType>>> Handle(
        GetResourcesQuery request,
        CancellationToken cancellationToken = default)
    {
        try
        {
            logger.LogDebug("Starting to retrieve resources from database");

            var resourceTypes = await dbContext
                .Resources
                .AsNoTracking()
                .OrderBy(et => et.Name)
                .Select(et => new ResourceType { Id = et.ResourceTypeId, Name = et.Name })
                .ToListAsync(cancellationToken);

            logger.LogInformation("Successfully retrieved {Count} resources from database",
                resourceTypes.Count);
            return ProcessResult.Success(resourceTypes.AsEnumerable());
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Failed to retrieve resources from database");
            return ProcessResult.Failure<IEnumerable<ResourceType>>();
        }
    }
}

// ❌ Incorrect - No logging or poor logging
public class GetResourcesQueryHandler(ApplicationDbContext dbContext)
    : IRequestHandler<GetResourcesQuery, ProcessResult<IEnumerable<ResourceType>>>
{
    public async Task<ProcessResult<IEnumerable<ResourceType>>> Handle(
        GetResourcesQuery request,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var resourceTypes = await dbContext.Resources.ToListAsync(cancellationToken);
            return ProcessResult.Success(resourceTypes.AsEnumerable());
        }
        catch (Exception)
        {
            return ProcessResult.Failure<IEnumerable<ResourceType>>();
        }
    }
}
```

## Logging Level Guidelines

- **Debug**: Detailed operation steps, database queries, internal state
- **Information**: Normal operation flow, successful operations with data
- **Warning**: Business logic failures, recoverable errors, validation failures
- **Error**: Technical exceptions, system failures, unrecoverable errors

## Structured Logging Best Practices

- **ALWAYS** use parameterized messages: `"Retrieved {Count} items"` not `"Retrieved " + count + " items"`
- **ALWAYS** include relevant context: IDs, counts, operation names
- **ALWAYS** use consistent property names: `{Count}`, `{Id}`, `{Operation}`
- **NEVER** log sensitive data: passwords, tokens, PII, connection strings
- **ALWAYS** log exceptions with full details in Error level

## Example with Full Pattern

```csharp
// Complete example with proper logging
[HttpGet("{id:guid}")]
public async Task<IActionResult> GetResource(
    [FromRoute] Guid id,
    CancellationToken cancellationToken = default)
{
    logger.LogInformation("Retrieving resource with ID {ResourceId}", id);

    try
    {
        var processResult = await mediator.Send(new GetResourceQuery(id), cancellationToken);

        if (processResult.IsSuccess)
        {
            logger.LogInformation("Successfully retrieved resource with ID {ResourceId}", id);
            return HandleSuccess(processResult.Data!);
        }

        logger.LogWarning("Resource with ID {ResourceId} not found", id);
        return CreateProblemDetailsResponse(
            "Resource not found",
            $"No resource exists with ID {id}",
            StatusCodes.Status404NotFound);
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Unexpected error occurred while retrieving resource with ID {ResourceId}", id);
        return CreateProblemDetailsResponse(
            "Internal server error",
            "An unexpected error occurred while processing the request",
            StatusCodes.Status500InternalServerError);
    }
}
```

---
name: api-design
description: Use when implementing API endpoints, working with ApiResult/ProcessResult patterns, or designing API responses.
---

# API Design Patterns

Use this skill when implementing API endpoints, working with ApiResult/ProcessResult patterns, or designing API responses.

## ApiResult Pattern (API Layer)

- **ALWAYS** use `ApiResult` types for all API responses
- **NEVER** return raw types or exceptions from controllers
- **ALL** `ApiResult` types live in `Shared.Contracts` project (shared across all microservices)
- Use `ApiResult<T>` for operations that return data
- Use `ApiPagedResult<T>` for paginated responses
- **ALWAYS** use Problem Details (RFC 7807) for error responses
- **ApiResult** is a pure data container - success/failure determined by HTTP status codes

## ProcessResult Pattern (Application Layer)

- **ALWAYS** use `ProcessResult` types in Application layer (handlers, services)
- **NEVER** use `ApiResult` directly in Application layer
- **ALL** `ProcessResult` types live in `Shared.Application` project
- Use `ProcessResult<T>` for operations that return data
- Use `ProcessResult` for operations that don't return data
- Use `ProcessPagedResult<T>` for paginated results
- **ProcessResult** focuses on business logic success/failure only
- **Controllers** convert `ProcessResult` to `ApiResult` for API responses

## Correct Usage Examples

```csharp
// ✅ Correct - Controller uses ProcessResult and creates ApiResult
public async Task<IActionResult> GetResource([FromRoute] Guid id)
{
    var processResult = await _mediator.Send(new GetResourceQuery(id));

    if (processResult.IsSuccess)
    {
        return HandleSuccess(processResult.Data!);
    }

    return CreateProblemDetailsResponse(
        "Failed to retrieve resource",
        "An error occurred while processing the request");
}

// ✅ Alternative - Direct object initializer (if not using HandleSuccess)
public async Task<IActionResult> GetResource([FromRoute] Guid id)
{
    var processResult = await _mediator.Send(new GetResourceQuery(id));

    if (processResult.IsSuccess)
    {
        return Ok(new ApiResult<Resource> { Data = processResult.Data! });
    }

    return CreateProblemDetailsResponse(
        "Failed to retrieve resource",
        "An error occurred while processing the request");
}

// ✅ Correct - Handler uses ProcessResult (Application layer)
public async Task<ProcessResult<Resource>> Handle(GetResourceQuery request)
{
    try
    {
        var resource = await _repository.GetByIdAsync(request.Id);
        return ProcessResult.Success(resource);
    }
    catch (Exception)
    {
        return ProcessResult.Failure<Resource>();
    }
}
```

## Architectural Separation

**ProcessResult vs ApiResult:**

- **ProcessResult**: Used in Application layer for business logic success/failure
- **ApiResult**: Used in API layer as pure data container for HTTP responses
- **Controllers**: Check ProcessResult.IsSuccess and create appropriate response
- **Handlers**: Return ProcessResult (not ApiResult)
- **Clean separation**: API concerns stay in API, business logic stays in Application

**Project Dependencies:**

- **Shared.Application**: Only references `Shared.Contracts`
- **Shared.Api**: References both `Shared.Application` and `Shared.Contracts`
- **Dependency direction**: Application → Contracts, API → Application + Contracts

## Controller Standards

- **ALL** V2 controllers extend `ApiControllerBase`
- **ALL** V2 routes use prefix `api/v2/[controller]`
- **NO** try-catch blocks in controllers (handled by middleware)
- **NO** validation logic in controllers (handled by pipeline)
- **ALWAYS** use `CancellationToken` on async operations
- **ALWAYS** convert `ProcessResult` to `ApiResult` in controllers
- **ALWAYS** return `IActionResult` with `ApiResult<T>` responses

## Route Conventions

- **ALWAYS** use kebab-case for multi-word routes: `api/v2/resource-types`, `api/v2/field-sets`, `api/v2/option-lists`
- **ALWAYS** use plural nouns for collections: `entities`, `countries`, `aspects`, `resource-types`
- **ALWAYS** use singular nouns for individual resources: `resource/{id}`, `field/{identifier}`
- **ALWAYS** use action verbs for commands: `import`, `validate`, `process`
- **NEVER** use camelCase or PascalCase in route segments: ❌ `api/v2/resourceTypes`, ✅ `api/v2/resource-types`

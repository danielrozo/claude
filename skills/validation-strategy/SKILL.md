---
name: validation-strategy
description: Use when implementing validation logic, FluentValidation, or business rule validation.
---

# Validation Strategy

Use this skill when implementing validation logic, FluentValidation, or business rule validation.

## Input Validation (API Layer)

- **ALWAYS** use FluentValidation for request DTOs
- **ALWAYS** place validators in `{Feature}/Validators/` folder
- **ALWAYS** use descriptive error messages
- **ALWAYS** validate required fields, formats, and ranges
- **AUTOMATIC**: MediatR ValidationBehavior validates all requests automatically

```csharp
// ✅ Correct - FluentValidation for input validation
public class CreateResourceTypeRequestValidator : AbstractValidator<CreateResourceTypeRequest>
{
    public CreateResourceTypeRequestValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .WithMessage("Name is required")
            .MaximumLength(100)
            .WithMessage("Name cannot exceed 100 characters");
    }
}
```

## Business Validation (Application Layer)

- **ALWAYS** use ProcessResult for business rule violations
- **NEVER** throw exceptions for business rule violations
- **ALWAYS** return ProcessResult.Failure() for validation failures
- **ALWAYS** handle ProcessResult.IsSuccess in controllers

```csharp
// ✅ Correct - Business validation in handler
public async Task<ProcessResult<ResourceType>> Handle(CreateResourceTypeCommand request)
{
    if (await _repository.ExistsByNameAsync(request.Name))
    {
        return ProcessResult.Failure<ResourceType>();
    }

    var resourceType = new ResourceType(request.Name);
    await _repository.SaveAsync(resourceType);
    return ProcessResult.Success(resourceType);
}

// ✅ Correct - Controller handles ProcessResult
public async Task<IActionResult> CreateResourceType(CreateResourceTypeRequest request)
{
    var processResult = await _mediator.Send(new CreateResourceTypeCommand(request));

    if (processResult.IsSuccess)
    {
        return HandleSuccess(processResult.Data!);
    }

    return CreateProblemDetailsResponse(
        "Failed to create resource",
        "Resource type with this name already exists");
}
```

## Domain Validation (Domain Layer)

- **ALWAYS** throw domain exceptions for invariant violations
- **NEVER** use ProcessResult in domain layer
- **ALWAYS** use specific domain exception types
- **ALWAYS** let middleware handle domain exceptions

## External API Validation Pattern

**FOLLOW THIS EXACT PATTERN** for validation requiring external API calls:

### Layer Responsibilities

- **Domain/Ports**: Define `IValidationAdapter` interface (no external dependencies)
- **Infrastructure/Adapters**: Implement `ValidationAdapter` with API client dependencies
- **Application**: Create FluentValidation validators that depend on `IValidationAdapter`
- **Application/Behaviors**: Use `ValidationBehavior` for automatic MediatR pipeline validation

### Implementation Structure

```
Domain/Ports/
  └── IValidationAdapter.cs

Infrastructure/Adapters/
  └── ValidationAdapter.cs

Application/Behaviors/
  └── ValidationBehavior.cs

Application/[Feature]/[Command]/
  └── [Command]Validator.cs
```

### Example Implementation

```csharp
// Domain/Ports/IValidationAdapter.cs
public interface IValidationAdapter
{
    Task<bool> IsValidCurrencyAsync(string currencyCode, CancellationToken cancellationToken);
    Task<bool> IsValidPartyAsync(string partyId, CancellationToken cancellationToken);
}

// Infrastructure/Adapters/ValidationAdapter.cs
public class ValidationAdapter : IValidationAdapter
{
    private readonly ICurrencyApiClient _currencyClient;
    private readonly IPartyApiClient _partyClient;

    public async Task<bool> IsValidCurrencyAsync(string currencyCode, CancellationToken cancellationToken)
    {
        return await _currencyClient.ValidateCurrencyAsync(currencyCode, cancellationToken);
    }
}

// Application/[Command]/[Command]Validator.cs
public class CreateResourceCommandValidator : AbstractValidator<CreateResourceCommand>
{
    private readonly IValidationAdapter _validationAdapter;

    public CreateResourceCommandValidator(IValidationAdapter validationAdapter)
    {
        _validationAdapter = validationAdapter;

        RuleFor(x => x.CurrencyCode)
            .MustAsync(async (code, cancellationToken) =>
                await _validationAdapter.IsValidCurrencyAsync(code, cancellationToken))
            .WithMessage("Invalid currency code");
    }
}
```

### Configuration

```csharp
// In Program.cs or ServiceCollectionExtensions
services.AddScoped<IValidationAdapter, ValidationAdapter>();
services.AddValidatorsFromAssemblyContaining<CreateResourceCommandValidator>();
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
```

### Key Principles

- ✅ FluentValidation validators in Application layer
- ✅ External API dependencies abstracted via Adapter Pattern
- ✅ Domain interfaces have no external dependencies
- ✅ Infrastructure adapters handle API client dependencies
- ✅ MediatR ValidationBehavior for automatic validation
- ✅ Proper dependency inversion maintained

## Anti-Patterns to Avoid

```csharp
// ❌ Incorrect - Validation in controller
public async Task<IActionResult> CreateResourceType(CreateResourceTypeRequest request)
{
    if (string.IsNullOrEmpty(request.Name))
    {
        return BadRequest("Name is required"); // Don't do this
    }
    // ... rest of logic
}

// ❌ Incorrect - Throwing exceptions for business rules
public async Task<ProcessResult<ResourceType>> Handle(CreateResourceTypeCommand request)
{
    if (await _repository.ExistsByNameAsync(request.Name))
    {
        throw new InvalidOperationException("Name exists"); // Don't do this
    }
    // ... rest of logic
}
```

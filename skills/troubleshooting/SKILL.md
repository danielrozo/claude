---
name: troubleshooting
description: Use when tests fail, builds break, MCPs disconnect, or when uncertain about implementation. Provides systematic debugging and problem-solving strategies.
trigger: test failure, build error, debugging, MCP error, TypeScript error, troubleshooting
---

# Troubleshooting Skill

## When Tests Fail

### Process
1. **Read the error message carefully**
   - Don't just scan - read the full error
   - Note the file, line number, and test name
   - Identify if it's a test issue or code issue

2. **Show the failing test output**
   - Display the actual vs expected values
   - Show the stack trace if relevant
   - Point out the specific assertion that failed

3. **Identify root cause (not just symptoms)**
   - Is it a logic error in the code?
   - Is the test incorrect or outdated?
   - Is it a mock/fixture data issue?
   - Is it a timing/async issue?
   - Is it a type mismatch?

4. **Fix the underlying issue**
   - Fix the code if logic is wrong
   - Update the test if expectations changed
   - Update mocks if API changed
   - Add await if async issue
   - Fix types at the source

5. **Verify fix with re-run**
   - Run the specific test that failed
   - Run related tests
   - Run full test suite if significant change

6. **Check for related failures**
   - Look for similar patterns in other tests
   - Update related tests proactively
   - Document breaking changes

### Common Test Issues

**Type Errors in Tests:**
```typescript
// Issue: Mock data doesn't match updated type
const mockRelationship = { id: 1, percentage: 50 }; // Missing new fields

// Fix: Update mock to match current type
const mockRelationship: DataModel = {
  id: 1,
  percentage: 50,
  title: null,
  startDate: null,
  endDate: null,
  notes: null,
  // ... other required fields
};
```

**Async Issues:**
```typescript
// Issue: Not awaiting async operation
const result = service.updateRelationship(data);
expect(result.percentage).toBe(50);

// Fix: Await the operation
const result = await service.updateRelationship(data);
expect(result.percentage).toBe(50);
```

**Mock Configuration:**
```typescript
// Issue: Mock not configured for new parameter
jest.spyOn(service, 'validate').mockResolvedValue(true);

// Fix: Handle new parameters
jest.spyOn(service, 'validate')
  .mockImplementation(async (resourceId, percentage) => {
    return percentage <= 100;
  });
```

## When Build Fails

### Process
1. **Show the compilation error**
   - Display the full TypeScript error
   - Note the file and line number
   - Identify the error code (TS2345, TS2339, etc.)

2. **Explain TypeScript errors in plain language**
   - "Property 'percentage' doesn't exist on type 'DataModel'"
   - → The type definition is outdated, needs 'percentage'

3. **Fix type mismatches at source**
   - Update type definitions first
   - Then update all usages
   - Don't use `any` or `@ts-ignore` to bypass

4. **Verify related code still type-checks**
   - Check imports of changed types
   - Verify interfaces that extend changed types
   - Check function signatures using changed types

### Common Build Issues

**Missing Properties:**
```typescript
// Error: Property 'percentage' does not exist on type 'DataModel'

// Fix: Add to type definition
interface DataModel {
  id: number;
  percentage?: number; // Add missing property
  // ... other properties
}
```

**Import Errors:**
```typescript
// Error: Module '"./types"' has no exported member 'DataModel'

// Fix: Check export in types file
export interface DataModel { ... } // Ensure it's exported
```

**Circular Dependencies:**
```typescript
// Error: Circular dependency detected

// Fix: Extract shared types to separate file
// types/shared.ts - common types
// types/resource.ts - resource-specific types (imports from shared)
// types/relationship.ts - relationship types (imports from shared)
```

## When MCP Fails

### Process
1. **Check MCP connection status**
   - Use `/mcp` command to list status
   - Verify which MCPs are connected/disconnected

2. **Verify environment variables**
   - Check if required env vars are set
   - Verify credentials are not expired
   - Check network connectivity

3. **Test with simple query first**
   - Try a basic operation (e.g., get single ticket)
   - Verify permissions
   - Check rate limits

4. **Fallback to manual operations**
   - Provide instructions for manual steps
   - Offer alternative approaches
   - Continue with demo smoothly

5. **Explain what went wrong**
   - Be transparent about the issue
   - Show how to debug MCP issues
   - Demonstrate resilience

### Common MCP Issues

**Atlassian MCP:**
```
Error: fetch failed

Troubleshooting:
1. Check: /mcp atlassian
2. Verify JIRA_API_TOKEN in environment
3. Test: Try fetching a known ticket
4. Fallback: Manually provide ticket details
```

**Context7 MCP:**
```
Error: Library not found

Troubleshooting:
1. Check library name spelling
2. Use resolve-library-id first
3. Verify library is in Context7 database
4. Fallback: Use WebFetch for official docs
```

## When Uncertain

### What to Do
- **Ask clarifying questions:** "Should the percentage field be required or optional?"
- **Research codebase patterns:** Use Explore agent to find similar implementations
- **Use agents for discovery:** "Launch Explore agent to find all validation patterns"
- **Admit when you need user input:** "I need to confirm the business rule before implementing"
- **Don't guess - investigate:** Read the code, check tests, explore related files

### Questions to Ask
- "What should happen if the total exceeds 100%?"
- "Should we validate immediately or on save?"
- "Are there existing similar implementations I should follow?"
- "What's the expected behavior for legacy data?"
- "Should this be a breaking change or backward compatible?"

### Investigation Techniques
1. **Use Explore agent** for broad discovery
2. **Use Grep** for specific patterns
3. **Read related tests** to understand expected behavior
4. **Check existing implementations** for similar features
5. **Review git history** for context on decisions

## Demo Recovery Strategies

### If Implementation Takes Too Long
- "Let me show you the plan first, then we can implement key parts"
- Focus on the most interesting/complex part
- Skip boilerplate, focus on unique logic

### If Tests Keep Failing
- "This is actually a great learning opportunity - let's debug this together"
- Show systematic troubleshooting process
- Explain why each fix is needed
- Turn failure into teaching moment

### If MCP Disconnects
- "Let me show you what we'd normally do, but I'll provide the data manually"
- Show the hook that would have automated it
- Demonstrate the command that uses MCP
- Continue smoothly with manual approach

### If Build Breaks
- "TypeScript caught an issue - this is exactly what we want"
- Explain the error clearly
- Show the fix
- "This prevents runtime errors in production"

## Logging and Debugging

### Console Logging
```typescript
// Temporary debug logging
console.log('Total ownership:', { current: total, new: percentage, sum: total + percentage });

// Remove before commit or use proper logging
logger.debug('Validating ownership', { resourceId, percentage, currentTotal: total });
```

### Debugging Validation
```typescript
// Log intermediate steps
const relationships = await getRelationships(resourceId);
console.log('Found relationships:', relationships.length);

const total = relationships.reduce((sum, r) => sum + (r.percentage ?? 0), 0);
console.log('Current total:', total);

const newTotal = total + percentage;
console.log('New total would be:', newTotal);

if (newTotal > 100) {
  console.log('Validation failed');
  return error;
}
```

### React Query Debugging
```typescript
// Enable React Query DevTools
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

// In App component
<ReactQueryDevtools initialIsOpen={false} />
```

## Preventing Issues

### Before Implementing
- Read ticket carefully
- Check for existing patterns
- Verify types are up to date
- Review related tests

### During Implementation
- Run tests frequently
- Check types after each change
- Commit working increments
- Keep changes focused

### After Implementation
- Run full test suite
- Check for TypeScript errors
- Test manually in browser
- Review changes for quality

### Code Review Checklist
- [ ] Tests passing
- [ ] No TypeScript errors
- [ ] Follows existing patterns
- [ ] Validation logic correct
- [ ] Error handling complete
- [ ] Performance considered
- [ ] Accessibility maintained
- [ ] Documentation updated

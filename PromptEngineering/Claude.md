
# Claude.md


1. CLAUDE.md is a special Markdown file that becomes part of Claude’s system prompt. Every time you start a Claude Code session in a directory containing this file, its contents are automatically loaded into context.

2. Without CLAUDE.md, Claude starts every session blind. It doesn’t know your project structure, your coding conventions, your preferred patterns, or even how to run your tests. You end up repeating the same context over and over.

3. With a well-crafted CLAUDE.md, Claude hits the ground running. It knows your architecture, respects your patterns, and makes decisions aligned with your codebase from the first prompt.


## How Context Works

When you start a session, Claude doesn’t load your entire codebase into memory immediately. Instead, it:

1. Reads your CLAUDE.md file
2. Scans the project structure
3. Loads files on-demand as needed
4. Maintains conversation history


## Memory Levels (Highest to Lowest Priority)

Enterprise Policy - Organization-level rules (if using Claude for Teams)
Project Memory - CLAUDE.md in your repository root
Project Rules - Files in .claude/rules/ directory
User Memory - ~/.claude/CLAUDE.md for personal global preferences

## Directory Structure

.claude/
├── rules/
│   ├── architecture.md      # Layer rules and dependencies
│   ├── api-conventions.md   # Endpoint patterns
│   ├── testing.md           # Test patterns and naming
│   ├── database.md          # EF Core conventions
│   └── modules/
│       ├── identity.md      # Identity module specifics
│       ├── catalog.md       # Catalog module specifics
│       └── orders.md        # Orders module specifics

@.claude/rules/architecture.md
@.claude/rules/testing.md
@.claude/rules/api-conventions.md

## What to Include (And What NOT to Include)
This is where most developers go wrong. They either include too little (Claude operates blind) or too much (wastes context tokens on irrelevant details).

### Include These
Tech stack and versions - Be specific: “Entity Framework Core 10” not just “EF Core”
Project structure - Map folders to their purpose
Common commands - Build, test, run, migrations
Coding conventions - Naming, patterns, style preferences
Patterns you use - Mediator, Result pattern, CQRS
Patterns you DON’T use - Equally important to prevent unwanted suggestions
Domain terminology - Business terms that map to code entities
Testing instructions - How to run tests, what frameworks you use
Repository workflow - Branch naming, commit conventions, PR process

### Do NOT Include
Secrets or credentials - Never put API keys, connection strings, or passwords
Code style rules that linters handle - Use Prettier/ESLint/dotnet format instead
Obvious framework knowledge - Claude knows how ASP.NET Core works
Excessive documentation - Link to docs instead of copying content
Historical context - Focus on current state, not project history



## Claude.md Example
```
# CLAUDE.md - [Project Name]

## Overview
[One-sentence description of what this project does]

## Tech Stack
- .NET 10, ASP.NET Core Minimal APIs
- Entity Framework Core 10 with PostgreSQL
- Mediator for CQRS (source-generated, https://github.com/martinothamar/Mediator)
- FluentValidation for request validation
- Scalar for OpenAPI documentation
- xUnit + FluentAssertions for testing

## Project Structure
- `src/Api/` - Endpoints, middleware, DI configuration
- `src/Application/` - Commands, queries, handlers, validators
- `src/Domain/` - Entities, value objects, enums, domain events
- `src/Infrastructure/` - EF Core, external services, repositories
- `tests/UnitTests/` - Domain and application layer tests
- `tests/IntegrationTests/` - API and database tests

## Commands
- Build: `dotnet build`
- Test: `dotnet test`
- Run API: `dotnet run --project src/Api`
- Add Migration: `dotnet ef migrations add <Name> -p src/Infrastructure -s src/Api`
- Update Database: `dotnet ef database update -p src/Infrastructure -s src/Api`
- Format: `dotnet format`

## Architecture Rules
- Domain layer has ZERO external dependencies
- Application layer defines interfaces, Infrastructure implements them
- All database access goes through EF Core DbContext (no repository pattern)
- Use Mediator for all command/query handling
- API layer is thin - endpoint definitions only

## Code Conventions

### Naming
- Commands: `Create[Entity]Command`, `Update[Entity]Command`
- Queries: `Get[Entity]Query`, `List[Entities]Query`
- Handlers: `[Command/Query]Handler`
- DTOs: `[Entity]Dto`, `Create[Entity]Request`

### Patterns We Use
- Primary constructors for DI
- Records for DTOs and commands
- Result<T> pattern for error handling (no exceptions for flow control)
- File-scoped namespaces
- Always pass CancellationToken to async methods

### Patterns We DON'T Use (Never Suggest)
- Repository pattern (use EF Core directly)
- AutoMapper (write explicit mappings)
- Exceptions for business logic errors
- Stored procedures

## Validation
- All request validation in FluentValidation validators
- Validators auto-registered via assembly scanning
- Validation runs in Mediator pipeline behavior

## Testing
- Unit tests: Domain logic and handlers
- Integration tests: Full API endpoint testing with WebApplicationFactory
- Use FluentAssertions for readable assertions
- Test naming: `[Method]_[Scenario]_[ExpectedResult]`

## Git Workflow
- Branch naming: `feature/`, `bugfix/`, `hotfix/`
- Commit format: `type: description` (feat, fix, refactor, test, docs)
- Always create a branch before changes
- Run tests before committing

## Domain Terms
- [Term 1] - [Maps to Entity/Concept]
- [Term 2] - [Maps to Entity/Concept]

## REFACTOR
- “Never refactor code unless explicitly asked.”

```


## Claude Code Custom Commands

You can create custom commands to streamline repetitive tasks. These live in the .claude/commands/ folder as Markdown files.

For example, create .claude/commands/test.md:
```
Run all tests for the solution and report any failures.
If tests fail, analyze the error and suggest fixes.

$ARGUMENTS
``` 

Now you can type /test in Claude, and it executes your predefined prompt.

Some ideas for custom commands:
/pr - Create a pull request with a structured description
/review - Review the current changes for code quality
/migrate - Generate an EF Core migration


## Create Skill
A skill is a reusable Markdown file that Claude Code loads on demand as part of its prompt. You write instructions in a SKILL.md file, give it a name, and invoke it with /skill-name - just like a built-in slash command. Claude reads the instructions and executes them.

Store project-specific skills in **.claude/skills/** inside your repository so they are version-controlled and shared with your team.

**Skills vs Rules vs Hooks: Skills for what to do (workflows), Rules for how things are (conventions), Hooks for what happens automatically (triggers).**


https://codewithmukesh.com/blog/skills-claude-code/ 
```
mkdir -p .claude/skills/scaffold-controller

create skill.md -> 
.claude/skills/scaffold-controller/skill.md

```

### Skill.md
```
---
name: scaffold-controller
description: Scaffold an ASP.NET Core controller with constructor injection, proper response types, and XML documentation comments. Use when the user asks to create a new controller or API endpoint.
argument-hint: [entity-name]
---
# Scaffold ASP.NET Core Controller

Generate a new API controller for the `$ARGUMENTS` entity.

## Requirements

1. **File location**: `src/Api/Controllers/{EntityName}Controller.cs`
2. **Namespace**: Match the project's root namespace + `.Controllers`
3. **Base class**: Inherit from `ControllerBase`
4. **Attributes**: `[ApiController]` and `[Route("api/[controller]")]`
5. **Constructor injection**: Accept the service interface via constructor DI
6. **Response types**: Use `ActionResult<T>` return types with proper status codes
7. **XML comments**: Add `<summary>` docs on every public method
8. **Endpoints to generate**:
   - `GET /api/{entity}` - list all (with pagination parameters)
   - `GET /api/{entity}/{id}` - get by ID
   - `POST /api/{entity}` - create
   - `PUT /api/{entity}/{id}` - update
   - `DELETE /api/{entity}/{id}` - delete

## Code Style
- Use `IActionResult` for delete, `ActionResult<T>` for everything else
- Return `NotFound()` when entity is null, not an empty 200
- Use cancellation tokens on all async methods
- Follow the project's existing naming conventions from CLAUDE.md

```

### Invoke Skill
```
/scaffold-controller Product
```

### Skill Example: .NET Endpoint Generator
 - .claude/skills/gen-endpoint/SKILL.md
 ```
 ---
name: gen-endpoint
description: Generate a complete ASP.NET Core Minimal API endpoint with DTOs, validation, service interface, and integration test. Use when asked to create a new endpoint or API route.
argument-hint: [HTTP-method] [route] [entity-name]
allowed-tools: Read Write Edit Grep Glob Bash(dotnet build *)
---
# Generate Minimal API Endpoint

Create a complete endpoint for **$2** using **$0 $1**.

## Step 1: Analyze the Project Structure

Before generating anything:
- Read the existing project structure using Glob
- Find existing endpoint registrations to match the pattern
- Check for existing DTOs, services, and validation patterns
- Read CLAUDE.md for project-specific conventions

## Step 2: Generate Request and Response DTOs

Location: `src/Api/Contracts/{EntityName}/`

- `{Method}{EntityName}Request.cs` - request DTO with data annotations
- `{EntityName}Response.cs` - response DTO (reuse if it already exists)

Use records for DTOs. Include XML documentation.

## Step 3: Add FluentValidation Validator

Location: `src/Api/Validators/`

- `{Method}{EntityName}RequestValidator.cs`
- Validate all required fields, string lengths, and business rules
- Use `.WithMessage()` for every rule - no default messages

## Step 4: Create or Update Service Interface and Implementation

- Interface: `src/Api/Services/I{EntityName}Service.cs`
- Implementation: `src/Api/Services/{EntityName}Service.cs`
- Add only the method needed for this endpoint
- Use CancellationToken on all async methods
- Return a Result type if the project uses one, otherwise throw on not-found

## Step 5: Register the Endpoint

Location: Follow the project's existing endpoint registration pattern.

```csharp
app.Map{Method}("{route}", async (
    [{FromBody/FromRoute/FromQuery}] {RequestType} request,
    I{EntityName}Service service,
    CancellationToken ct) =>
{
    // implementation
})
.WithName("{Method}{EntityName}")
.WithOpenApi()
.Produces<{ResponseType}>(StatusCodes.Status{XXX})
.ProducesValidationProblem();
```

## Step 6: Register in DI Container

Add the service registration in the appropriate DI extension method.

## Step 7: Generate Integration Test

Location: `tests/Api.IntegrationTests/{EntityName}/`

- Test the happy path
- Test validation failures (400)
- Test not-found scenarios (404) where applicable
- Use WebApplicationFactory

## Step 8: Build Verification

Run `dotnet build` on the solution. Fix any compilation errors before reporting done.

 ```
 ```

 ### Skill Example : Architecture Review
 ```
---
name: architecture-review
description: Deep architecture review with extended reasoning
context: fork
agent: general-purpose
---

# Architecture Review (ultrathink)

Analyze the architecture of this .NET solution:
1. Map the dependency graph between projects
2. Identify circular dependencies
3. Evaluate adherence to Clean Architecture principles
4. Flag any violations of SOLID principles
5. Recommend specific refactoring steps
 ```

 ### .Net Skill Creator Plugin

 - Inside a Claude Code session

```
    /plugin install skill-creator
    /skill-creator I want a skill that scaffolds an EF Core entity with a configuration class and a migration
``` 




## Commands - /PR-Review
- create .claude\commands\Pr-Review\command.md

```
---
name: pr-review
description: Review a pull request with full context
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---
# Pull Request Review

## Context (live data)
- **Changed files:** !`gh pr diff --name-only`
- **PR description:** !`gh pr view --json body --jq .body`
- **Test status:** !`gh pr checks`

## Your Task

Review this PR for:
1. Code quality and .NET best practices
2. Missing null checks or validation
3. Performance concerns
4. Test coverage gaps

Provide feedback as a numbered list with file:line references.

```

## Ready Skills
https://codewithmukesh.com/blog/skills-claude-code/

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

mkdir -p .claude/skills/scaffold-controller
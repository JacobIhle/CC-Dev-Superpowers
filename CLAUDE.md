# cc-dev-superpowers

This file ships with the `cc-dev-superpowers` plugin. It restates the conventions the plugin's skills enforce, as ambient reinforcement when CLAUDE.md is loaded.

## Default workflow for code-writing tasks

For any non-trivial production code change, follow the `feature-development-workflow` skill:

1. Write a failing test first (`test-driven-development` skill).
2. Write the minimum code to pass.
3. Refactor with green tests.
4. Loop until the unit is complete.
5. Dispatch the `code-reviewer` subagent for a clean-context review.
6. Address critical findings.
7. Dispatch the `docs-maintainer` subagent if the change affected anything user-visible.

Skip the pipeline for throwaway scripts, doc-only changes, and trivial mechanical edits. Honor explicit user opt-out.

## Temporary workspaces

When you need to clone an external repo, download reference material, or create scratch files for analysis, put them in `./tmp/` at the project root — never in `/tmp/` or other system directories. Reads inside the project are auto-allowed; reads outside aren't, so keeping scratch work in-repo avoids permission churn and makes cleanup straightforward. The consuming project should have `tmp/` in its `.gitignore`.

## Code style

For C# / .NET code, the canonical invariants:

- **Project**: nested project dir; `global.json` pins the SDK; csproj has `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>`.
- **Layout**: feature folders by role (`Endpoints/`, `Models/`, `Services/`, `Extensions/`). One primary type per file. File-scoped namespaces. Namespace mirrors folder hierarchy. Filename matches class name exactly.
- **Endpoints**: Minimal API. `public static class` with `Map<Name>Endpoint` extension; handlers return `Results.*`; chain `.WithName().WithOpenApi().RequireAuthorization(...)`.
- **Services**: constructor-injected instance classes by default; static classes only for pure orchestration with no per-call state. Concrete-type DI registration — no interface abstractions for internal services.
- **HTTP**: always `IHttpClientFactory.CreateClient()`. Never `new HttpClient()`.
- **DTOs**: `public class`, mutable `{ get; set; }`, `Dto` suffix (PascalCase). `required` on non-nullable properties; `?` on optionals.
- **Configuration**: `GetRequiredValue` extension for fail-fast on missing keys; no silent null defaults.
- **Async**: all I/O async; method names end with `Async`. No `ConfigureAwait`.
- **Errors**: services log + rethrow; endpoints log + `Results.Problem` with a generic external message.
- **Logging**: structured templates only with PascalCase placeholders. Never `$"..."` interpolation in `LogXxx(...)` calls.
- **Naming**: PascalCase types / methods / records / constants. `_camelCase` private fields. camelCase locals / parameters. `I`-prefix interfaces. `Async` suffix on async methods.
- **Tests**: xUnit + FluentAssertions, AAA layout, name `Method_Should_X_When_Y`, one behavior per test.
- **XML docs**: opt-in; if you open a tag, fill it in.

The `coding-style` skill has the *why*, the edge cases, and the contrasts.

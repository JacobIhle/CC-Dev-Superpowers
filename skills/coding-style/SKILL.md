---
name: coding-style
description: Use when writing or modifying C# / .NET code. Encodes structural conventions (project layout, endpoints, services, DTOs), async/error/logging patterns, naming, and test approach. Derived from a canonical reference codebase; the plugin's CLAUDE.md restates the hard invariants so they stay in context.
---

# Coding Style (.NET)

This skill is the elaboration — the *why*, the edge cases, and the contrasts. Hard invariants are restated in the plugin's CLAUDE.md so they're always in context.

## Project & file layout

- Solution structure: `<repo>/<project>/<project>.csproj` — nested project dir inside the solution root.
- `global.json` at repo root pins the SDK. Use `rollForward: "latestMinor"` and `allowPrerelease: false` so minor SDK upgrades roll automatically but prereleases are opt-in.
- `csproj` enables nullable and implicit usings: `<Nullable>enable</Nullable>`, `<ImplicitUsings>enable</ImplicitUsings>`. Web projects use `Microsoft.NET.Sdk.Web`.
- Source folders are organized by **role**, not feature: `Endpoints/`, `Models/`, `Services/`, `Extensions/`. Flat within each — don't add deeper nesting until volume actually warrants it.
- **File-scoped namespaces** (`namespace X;`). Namespace mirrors folder hierarchy.
- **One primary type per file.** Tightly-coupled small helper types may sit alongside (e.g. a `Result` record next to the service that returns it). Separate files when the helper is useful independently.
- **Filename matches class name exactly.** `EmployeeDto.cs` houses `EmployeeDto`, not `EmployeeDTO`. Casing matters.

## Endpoints (Minimal API)

- Each endpoint module is a `public static class` with a `Map<Name>Endpoint(this IEndpointRouteBuilder app[, IConfiguration configuration])` extension method.
- The extension is invoked from `Program.cs`. There are no MVC controllers.
- Handler is an inline lambda taking dependencies as parameters — they're resolved per-request from DI.
- When the lambda grows beyond a few statements, extract a `private static async Task<IResult> Handle*Async(...)` and call it from the lambda.
- Required roles are fetched from configuration up front, then passed to `.RequireAuthorization(p => p.RequireRole(role))`.
- Standard chain: `.WithName(...).WithOpenApi().RequireAuthorization(...)`.
- Return `Results.Ok / BadRequest / Problem / NotFound / AcceptedAtRoute / StatusCode((int)code)` — the typed factory methods, not raw status codes.

## Services

There are two shapes. Default to (1) for new services.

**1. Instance classes with constructor injection.** Registered via `AddScoped<T>()` or `AddSingleton<T>()` in `Program.cs`. Constructor takes `ILogger<T>`, `IHttpClientFactory`, `IConfiguration`, and other dependencies. Private readonly fields with `_camelCase` prefix. This is the canonical shape — anything that holds per-call state, has multiple methods that share dependencies, or might want a testing seam later.

**2. Static classes** for pure orchestration with no state, where the class only routes between other services. Methods take their dependencies (logger, config, services) as parameters. Tolerated when it's genuinely cleaner than the instance shape, but not the default.

DI registration:
- Use **concrete-type registration**: `builder.Services.AddScoped<EmployeeService>()`.
- **Don't introduce interface abstractions** for internal services unless there's a real reason (testing seam that can't be done another way, multiple implementations). The reference codebase has zero internal interfaces.
- Lifetimes: `Singleton` for stateless / shared resources (e.g. storage clients); `Scoped` for per-request state.

HTTP clients:
- **Always** `IHttpClientFactory.CreateClient()`. Inject `IHttpClientFactory`, call `CreateClient()` once in the constructor, store in `_httpClient`.
- **Never** `new HttpClient()`. It's a known socket-exhaustion footgun: each instance leaves a TCP connection in `TIME_WAIT` for ~4 minutes after dispose; under any meaningful load the host runs out of ports.

## DTOs / Models

- `public class` with `{ get; set; }` properties. Mutable, public setters. No factory / constructor pattern.
- **`Dto` suffix** in PascalCase: `EmployeeDto`, not `EmployeeDTO`.
- **All non-nullable properties use `required`.** Optional properties use `?`. Don't leave a non-nullable `string` property with no default and no `required` — that's a warning under `Nullable=enable` and a runtime null surprise waiting to happen.

```csharp
public class EmployeeDto
{
    public required string EmployeeId { get; set; }
    public required string Email { get; set; }
    public bool? IsActive { get; set; }
}
```

## Configuration

- `IConfiguration` is accessed inline within methods. We don't use the strongly-typed Options pattern by default — it's added when a config block is consumed in many places and benefits from a typed contract.
- A `GetRequiredValue` extension provides fail-fast on missing keys:

```csharp
public static class ConfigurationExtensions
{
    public static string GetRequiredValue(this IConfiguration configuration, string key)
        => configuration[key] ?? throw new InvalidOperationException($"Configuration value '{key}' is not set");
}
```

- **Use `GetRequiredValue` for any required setting.** A silent null default is worse than a startup exception — it pushes the failure to whichever runtime path first dereferences the value, often deep in a request handler.

## Async

- All I/O is async. Method names **end with `Async`** suffix.
- **No `ConfigureAwait` calls.** ASP.NET Core has no synchronization context to capture, so `ConfigureAwait(false)` is noise.
- Use `Task` / `Task<T>`. No `ValueTask` unless profiling shows the allocation actually matters.
- **Fire-and-forget** uses `Task.Run` with a try/catch wrapper that logs failures. Never silently lose an exception in a fire-and-forget — even when the work is intentionally detached, the failure deserves a log line.

## Error handling

- **Services**: try/catch around external calls (HTTP, storage, key vault). Log with context, then rethrow:
  ```csharp
  _logger.LogError(ex, "Failed to load employee {EmployeeId}", employeeId);
  throw;
  ```
  The caller decides what to do with the failure.
- **Endpoints**: try/catch at the top of the handler. Log, then return `Results.Problem(...)` with a *generic* external message. Don't leak exception details to clients.
- **Validation failures** throw `InvalidOperationException` (or a more specific exception type) with a descriptive message that names what was missing.
- Use `System.Diagnostics.UnreachableException` for "this branch is logically impossible" assertions — better than `throw new Exception(...)` for documenting intent.
- Configuration access goes through `GetRequiredValue` so missing config fails fast at first use.

## Logging

- `ILogger<T>` is injected. In Minimal API endpoint handlers, the parameter is `ILogger<DtoType>` for the request DTO.
- **Structured logging only.** Placeholders are PascalCase named tokens:
  ```csharp
  logger.LogInformation("Loading employee {EmployeeId}", request.EmployeeId);
  ```
- **Never** use `$"..."` interpolation in `LogXxx(...)` calls. It defeats structured logging — log aggregators can't filter on `EmployeeId` if it's been baked into the message string.
- **Mask sensitive data** before logging (phone numbers, tokens, secrets, partial PII). A `Mask*` helper next to the consumer is fine.

## Naming

- **Types** (class, struct, enum, delegate, record), **methods**, **records**, **constants**: PascalCase.
- **Interfaces**: `I`-prefixed PascalCase (`IUserService`).
- **Private fields**: `_camelCase` (`_logger`, `_httpClient`).
- **Local variables, parameters**: camelCase.
- **Async methods**: end with `Async` suffix.
- **DTOs**: end with `Dto` (PascalCase, not `DTO`).
- **Constants**: PascalCase — not SCREAMING_SNAKE_CASE.

## XML doc comments

- **Opt-in.** No requirement to add XML docs to public APIs.
- **If you write one, fill in every tag you open.** A `<summary>` with an empty `<returns>` is worse than no XML doc at all — it suggests the doc was started and abandoned.

## Tests

(Defaults — the reference codebase had no test project, so adjust if your project's existing tests already use a different stack.)

- **Framework**: xUnit.
- **Assertions**: FluentAssertions.
- **Layout**: AAA (Arrange / Act / Assert), with blank lines between sections.
- **Test name**: `MethodUnderTest_Should_DoX_When_Y` — describe the behavior, not the implementation.
- **One behavior per test.** Multiple assertions are fine when they describe one behavior; separate tests for separate behaviors.
- **Bug fixes start with a regression test.** Write the failing test that reproduces the bug, then fix. Leave the test as a guard against regression. (See the `test-driven-development` skill for the broader discipline.)

## Anti-patterns to avoid

- `new HttpClient()` — use `IHttpClientFactory`.
- `$"...{x}..."` in `LogXxx(...)` calls — use structured templates with PascalCase placeholders.
- Silent catch (`try { ... } catch { }`) — log, then either rethrow or return a typed failure.
- Non-nullable string DTO property without `required` — either mark it `required` or make it `string?`.
- Filename diverging from class name (`EmployeeDTO.cs` for class `EmployeeDto`) — fix on touch.
- Bare configuration access (`configuration["X"]`) without a null check — use `GetRequiredValue`.
- Folder nesting that mirrors aspirational architecture rather than actual file count — flatten until volume justifies the split.
- Interface abstractions for internal services with one implementation — adds indirection without payoff.

# Contribution [#1]: Implement roslyn_find_unused and supporting analysis tools

**Contribution Number:** 1
**Student:** Rubina Shaik
**Issue:** https://github.com/MadQ/RoslynMcp/issues/33
**Status:** Phase II completed

---

## Why I Chose This Issue

I chose this issue because it connects directly to my interest in AI developer tools and Model Context Protocol (MCP), which is an area I expect to work with in my upcoming job. RoslynMcp is a good fit because it already has an MCP server in place, so the project is focused on adding a useful new tool rather than building the entire system from scratch. The `roslyn_find_unused` tool would help AI coding agents inspect C# projects more intelligently by finding private or internal symbols with no references. I also think the scope is appropriate for a 3-4 week project because it includes learning the existing MCP tool structure, refreshing my C# knowledge, implementing the tool, adding tests, documenting the behavior, and responding to review feedback.

---

## Understanding the Issue

### Problem Description

Currently, agents often rely on grep or keyword matching to understand C# code, which is limited because text search does not understand symbols, overloads, references, or type relationships. RoslynMcp gives AI agents compiler-backed tools so they can reason about code semantically instead of matching strings.

This issue involves implementing three new semantic analysis tools:

- `roslyn_find_unused`: finds private/internal symbols that have no references and may be safe dead-code cleanup candidates.
- `roslyn_get_type_dependencies`: given a type, returns the other types it directly depends on, such as base types, interfaces, field types, property types, method return types, parameter types, and generic constraints.
- `roslyn_find_overloads`: given a method name and containing type, returns all overloads with their full method signatures.

Together, these tools help agents understand unused code, type coupling, and overloaded methods before making refactoring or editing decisions.

### Expected Behavior

AI agents should be able to use RoslynMcp to answer these semantic C# questions directly:

- Which private/internal symbols are unused?
- What types does a given type directly depend on?
- What overloads exist for a method on a containing type?

The system should provide dedicated tools for these questions: `roslyn_find_unused`, `roslyn_get_type_dependencies`, and `roslyn_find_overloads`.

### Current Behavior

These dedicated tools do not currently exist. Agents must approximate the answers using grep, keyword search, file reads, or combinations of existing broader tools like `roslyn_find_references` and `roslyn_get_type_members`.

This is less reliable because text search does not understand C# symbols, overloads, inheritance, interfaces, or type relationships.

### Affected Components

- `src/RoslynMcp/Tools/Analysis/` — where the new analysis tools should be implemented
- `src/RoslynMcp/Tools/RoslynMcpTool.cs` — shared helper patterns for resolving compilations and symbols
- `src/RoslynMcp/Tools/ToolResults.cs` — structured result types, if shared result records are added
- `src/TestHarness/Tests/` — tests for the new tool behavior
- `README.md` — tool catalog documentation
- `docs/AGENT-INSTRUCTIONS.md` — agent guidance for when to use the new tools
- `docs/plans/tool-suggestions.md` and `ROADMAP.md` — planning/status documentation

---
## Reproduction Process

### Environment Setup

I cloned my fork locally:

`https://github.com/rubinashaik2022/RoslynMcp.git`

Local path:

`/Users/rubinashaik/RoslynMcp`

Working branch:

`issue-19-semantic-analysis-tools`

I built the MCP server locally using `net8.0` because my machine has .NET 8/9 installed, but not .NET 10. The README shows a `net10.0` publish example, so I adjusted the target framework for my local environment.

Commands used:

dotnet restore src/RoslynMcp/RoslynMcp.csproj -p:TargetFrameworks=net8.0 -r osx-arm64
dotnet restore src/RoslynMcp.Analyzers/RoslynMcp.Analyzers.csproj
dotnet publish src/RoslynMcp/RoslynMcp.csproj -c Release -f net8.0 -o ./publish/net8.0 -p:TargetFrameworks=net8.0 --no-restorel 

### Steps to Reproduce / Explore Current Functionality

1. Built the RoslynMcp server locally to confirm the development environment works.
2. Ran the published server with `./publish/net8.0/RoslynMcp --version` to confirm the MCP executable starts successfully.
3. Reviewed the existing analysis tools under `src/RoslynMcp/Tools/Analysis/`.
4. Confirmed that similar semantic tools already exist, including `roslyn_find_references`, `roslyn_find_callers`, `roslyn_get_call_graph`, and `roslyn_get_type_members`.
5. Searched the codebase for `roslyn_find_unused`, `roslyn_get_type_dependencies`, and `roslyn_find_overloads`.
6. Confirmed that these tools are mentioned in planning docs but are not currently implemented as MCP tools.

### Observed Result

The project already has the architecture needed for semantic C# analysis tools, but the three requested tools are missing from the implementation.

### Reproduction Evidence

- **Commit showing reproduction:** N/A - This is not a reproducible bug but a feature.
- Branch link: https://github.com/rubinashaik2022/RoslynMcp/tree/issue-19-semantic-analysis-tools
- **Screenshots/logs:** [If applicable]
- **My findings:**
This is a feature gap rather than a runtime bug. Existing tools demonstrate the implementation pattern:

- `FindReferencesTool.cs` shows how to use `SymbolFinder.FindReferencesAsync`.
- `TypeMembersTool.cs` shows how to resolve a type and inspect members.
- `GetCallGraphTool.cs` shows how to inspect semantic dependencies using Roslyn.
- `FindCallersTool.cs` shows async Roslyn lookup and structured/paged results.

The new tools should follow these existing patterns.
---

## Solution Plan

### Understand

Add independent Roslyn-backed tools that follow existing tool conventions. Each tool should be read-only, idempotent, structured, and token-efficient.

### Match

Use these existing tools as implementation references:

- `FindReferencesTool.cs` for symbol reference lookup
- `FindCallersTool.cs` for async semantic lookup and result paging
- `GetCallGraphTool.cs` for dependency-style semantic traversal
- `TypeMembersTool.cs` for type resolution and member enumeration

### Plan

1. Implement `roslyn_find_unused`

   Find private/internal symbols with zero references.

   Suggested logic:

   - Enumerate source-declared private/internal types and members.
   - Exclude implicitly declared members, generated accessors, constructors unless intentionally supported, overrides, interface implementations, and public API.
   - Use `SymbolFinder.FindReferencesAsync` for each candidate.
   - Return only candidates with no reference locations outside the declaration.

  Result fields:

   - `name`
   - `kind`
   - `signature`
   - `accessibility`
   - `file`
   - `line`

2. Implement `roslyn_get_type_dependencies`

   Given a type, return directly referenced types.

   Logic:

   - Resolve the type using the same pattern as `TypeMembersTool`.
   - Collect dependencies from base type, interfaces, fields, properties, method return types, method parameters, generic arguments, and generic constraints.
   - Deduplicate with `SymbolEqualityComparer.Default`.
   - Exclude the containing type itself.

   Suggested result fields:

   - `typeName`
   - `totalDependencies`
   - `dependencies`
   - each dependency: `name`, `namespace`, `kind`, `source`, `reason`

3. Implement `roslyn_find_overloads`

   Given a method name and containing type, return all overloads.

   Logic:

   - Require `containingType`.
   - Resolve the containing type.
   - Filter `GetMembers(methodName)` to `IMethodSymbol`.
   - Exclude property/event accessors and implicitly declared methods.
   - Format signatures with `SymbolFormatter.FormatMethod`.

   Suggested result fields:

   - `containingType`
   - `methodName`
   - `totalOverloads`
   - `overloads`
   - each overload: `signature`, `returnType`, `parameters`, `accessibility`, `isStatic`, `isGeneric`, `file`, `line`

4. Add tests

   Cover these minimum cases:

   - `roslyn_find_unused` reports unused private members.
   - `roslyn_find_unused` does not report referenced private members.
   - `roslyn_get_type_dependencies` reports dependencies from fields, parameters, return types, base type, and interfaces.
   - `roslyn_find_overloads` returns all overloads for a method and excludes unrelated methods.

5. Update docs

   After implementation, update:

   - `README.md` Tool Catalog
   - `docs/AGENT-INSTRUCTIONS.md`
   - `ROADMAP.md`
   - `docs/plans/tool-suggestions.md`

### Review

Check that the new tools:

- Use the `roslyn_*` naming convention.
- Are marked read-only and idempotent.
- Use `TryGetCompilation`.
- Return structured result records.
- Use project-relative file paths.
- Page large result sets.
- Avoid text search for semantic questions.
- Include useful tool descriptions that tell agents when to use them.

### Evaluate

Run:

```bash
dotnet publish src/RoslynMcp/RoslynMcp.csproj -c Release -f net8.0 -o ./publish/net8.0 -p:TargetFrameworks=net8.0
```

Then manually smoke-test against a small C# sample project:

- A deliberately unused private method appears in `roslyn_find_unused`.
- A referenced private method does not appear in `roslyn_find_unused`.
- A type with fields, parameters, return types, base type, and interfaces returns expected dependencies.
- A method with multiple overloads returns every overload signature.

## Clarification

Some older text refers to "four new analysis tools." In the current repository state, three related tools remain unshipped:

- `roslyn_find_unused`
- `roslyn_get_type_dependencies`
- `roslyn_find_overloads`

`roslyn_check_syntax` appears to have been part of the earlier suggestion set, but it has already shipped in v0.8.0-beta.

---

## Testing Strategy

# Testing

## Integration Testing

Verified both projects compile and type-check with Roslyn MCP diagnostic/build tools.

### `src/RoslynMcp/RoslynMcp.csproj`

```text
roslyn_get_diagnostics
Result: 0 errors, 0 warnings
```

```text
roslyn_build_project
Result: build succeeded
```

### `src/TestHarness/TestHarness.csproj`

```text
roslyn_get_diagnostics
Result: 0 errors, 0 warnings
```

```text
roslyn_build_project
Result: build succeeded
```

## End-to-End Testing

Ran the full MCP `TestHarness`:

```bash
dotnet run --project src/TestHarness/TestHarness.csproj
```

The harness exercises the full MCP client/server flow:

1. Builds `src/RoslynMcp/RoslynMcp.csproj`.
2. Starts the RoslynMcp server as a child process.
3. Sends the MCP `initialize` request.
4. Sends the MCP `notifications/initialized` notification.
5. Invokes tools through MCP JSON-RPC requests.
6. Validates the JSON responses returned by the server.
7. Runs teardown for scratch files and local-history fixtures.

Final result:

```text
All 82 tests passed
```

The new and updated tools were exercised through the MCP protocol as part of the `Type Understanding Tools` group.

## `roslyn_find_overloads` Coverage

The harness verifies that `roslyn_find_overloads` handles the following cases.

### Simple Containing Type Name

```text
containingType = "RoslynMcpTool"
```

Verifies overload lookup works when the caller provides a simple type name.

### Fully-Qualified Containing Type Name

```text
containingType = "RoslynMcp.Tools.RoslynMcpTool"
```

Verifies overload lookup works when the caller provides a fully-qualified type name.

### Multiple Overloads

```text
methodName = "BeginTool"
```

Verifies all overload signatures are returned.

### Generic Method Signatures

```text
BeginTool<T>(...)
```

Verifies generic overload signatures are included.

### Optional/Default Parameter Values

```text
string? subject = null
```

Verifies default parameter values are preserved in the returned signatures.

### `out` Parameters

```text
out Compilation? compilation
out ToolResult? error
```

Verifies `out` parameter modifiers are included in full signatures.

### `ref` Parameters

```text
ref int skip
```

Verifies `ref` parameter modifiers are included in full signatures.

### Missing Method

```text
methodName = "DefinitelyNotAMethod"
```

Verifies the tool returns an empty overload result instead of an error:

```text
total_overloads == 0
overloads is empty
```

## `roslyn_get_type_dependencies` Coverage

The harness verifies direct dependencies from the type declaration and member signatures only.

## Covered Dependency Categories

### Base Type

```csharp
internal sealed class FindOverloadsTool : RoslynMcpTool
```

Verifies `RoslynMcpTool` is reported as:

```text
dependency_kind = "base_type"
```

### Directly Implemented Interface

```csharp
internal sealed class _TypeDependenciesMemberFixture_<TItem> : _TypeDependenciesDirectInterface_
```

Verifies `_TypeDependenciesDirectInterface_` is reported as:

```text
dependency_kind = "interface"
```

### Field Type

```csharp
private FileLogger? logger;
```

Verifies `FileLogger` is reported as:

```text
dependency_kind = "field"
member = "logger"
```

### Property Type

```csharp
public Dictionary<string, TItem[]> Items { get; } = [];
```

Verifies the constructed dictionary type is reported as:

```text
dependency_kind = "property"
member = "Items"
```

### Generic Type Arguments

```csharp
Dictionary<string, TItem[]>
```

Verifies `string` is reported as a dependency from the generic type argument list.

### Event Type

```csharp
public event Action<ErrorResult>? Changed;
```

Verifies `ErrorResult` is reported as:

```text
dependency_kind = "event"
member = "Changed"
```

### Constructor Parameter

```csharp
public FindOverloadsTool(
    WorkspaceResolver workspace,
    FileLogger logger,
    PaginationCache paginationCache)
```

Verifies `WorkspaceResolver` is reported as:

```text
dependency_kind = "constructor_parameter"
```

### Method Return Type

```csharp
public Project? Build<TResult>(Compilation compilation)
```

Verifies `Project` is reported as:

```text
dependency_kind = "method_return"
member = "Build"
```

### Method Parameter Type

```csharp
public Project? Build<TResult>(Compilation compilation)
```

Verifies `Compilation` is reported as:

```text
dependency_kind = "method_parameter"
member = "Build"
```

### Type Generic Constraint

```csharp
internal sealed class _TypeDependenciesMemberFixture_<TItem>
    where TItem : ToolResult
```

Verifies `ToolResult` is reported as:

```text
dependency_kind = "generic_constraint"
member = "TItem"
```

### Method Generic Constraint

```csharp
public Project? Build<TResult>(Compilation compilation)
    where TResult : ToolResult
```

Verifies `ToolResult` is reported as:

```text
dependency_kind = "generic_constraint"
member = "Build"
```

### User-Defined Operator Dependencies

```csharp
public static _TypeDependenciesOperatorFixture_ operator +(
    _TypeDependenciesOperatorFixture_ left,
    _TypeDependenciesOperatorFixture_ right)
```

Verifies `op_Addition` reports return and parameter dependencies:

```text
dependency_kind = "method_return"
member = "op_Addition"
```

```text
dependency_kind = "method_parameter"
member = "op_Addition"
```

### Conversion Operator Dependencies

```csharp
public static explicit operator string(_TypeDependenciesOperatorFixture_ value)
```

Verifies `op_Explicit` reports the conversion return dependency:

```text
type_name = "string"
dependency_kind = "method_return"
member = "op_Explicit"
```

## Negative and Guard Coverage

### Generic Placeholder Types Are Not Reported

```csharp
TItem
```

Verifies bare generic placeholder types are not returned as dependencies.

`TItem` is only a placeholder. The real dependency is the constraint:

```csharp
where TItem : ToolResult
```

So `ToolResult` is returned, but bare `TItem` is not.

### Function Pointer Dependencies Are Out of Scope

Function pointer dependencies are intentionally not covered in v1.

Reason:

```text
Function pointers require unsafe code.
The v1 fixture set stays in safe C#.
```

### Intentionally Out of Scope

The following are intentionally not reported by `roslyn_get_type_dependencies`:

```text
nested type dependencies
attributes
method body locals
method body calls
called method return types
implicit object base type
implicit ValueType base type
implicit Enum base type
unsafe function pointer internals
```

### Manual Testing

[]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]

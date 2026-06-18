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

#### Integration Testing

The new `roslyn_find_overloads` tool was tested through the project's existing `TestHarness`. These tests exercise the real MCP server rather than calling the tool method directly.

For each test, the harness:

1. Builds and starts the RoslynMcp server as a child process.
2. Opens an MCP session through the server's standard input and output streams.
3. Sends a JSON-RPC `tools/call` request for `roslyn_find_overloads`.
4. Provides a real project path, containing type, and method name.
5. The tool loads the project into a Roslyn `Compilation`.
6. It resolves the containing type and finds matching overloads.
7. The server serializes the result into JSON and returns it through MCP.
8. The test harness parses and validates the returned JSON.

The integration tests cover:

- Finding multiple overloads with the same method name.
- Preserving generic type parameters such as `<T>`.
- Preserving nullable annotations such as `string?`.
- Preserving optional/default parameter values such as `= null`.
- Preserving `out` parameter modifiers.
- Resolving fully qualified containing type names.
- Returning an empty overload list when the method does not exist.

### Integration Tests

- **Multiple overloads:** Calls `roslyn_find_overloads` for `RoslynMcpTool.BeginTool` and confirms that both overloads are returned.

- **Full method signatures:** Confirms that the returned signatures preserve:
  - Generic type parameters such as `<T>`
  - Nullable markers such as `string?`
  - Default values such as `subject = null`

  In C#, `string?` means that the value may be `null`. The parameter is optional because it has a default value.

- **Fully qualified type name:** Calls the tool using `RoslynMcp.Tools.RoslynMcpTool` instead of the shorter `RoslynMcpTool` name and confirms that the type still resolves correctly.

- **`out` parameters:** Calls the tool for `TryGetCompilation` and confirms that its signature preserves `out Compilation? compilation` and `out ToolResult? error`. An `out` parameter is assigned by the method and returned to the caller.

- **`ref` parameters:** Calls the tool for a method such as `Paginate` and confirms that `ref int skip` is preserved. A `ref` parameter allows the method to read and modify the caller's existing variable.

- **Missing method:** Requests a method that does not exist and confirms that the tool returns zero overloads and an empty list instead of failing.


### Manual Testing

[What you tested manually and results]

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

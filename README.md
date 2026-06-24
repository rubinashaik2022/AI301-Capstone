# Contribution [#1]: Implement `roslyn_find_unused` and Supporting Analysis Tools

**Contribution Number:** 1  
**Student:** Rubina Shaik  
**Issue:** https://github.com/MadQ/RoslynMcp/issues/33  
**Status:** Phase IV completed; all three analysis tools implemented

---

## Why I Chose This Issue

I chose this issue because it connects directly to my interest in AI developer tools and Model Context Protocol (MCP), which is an area I expect to work with in my upcoming job. RoslynMcp is a good fit because it already has an MCP server in place, so the project is focused on adding useful tools rather than building the entire system from scratch.

The `roslyn_find_unused` tool would help AI coding agents inspect C# projects more intelligently by finding private or internal symbols with no references. I also think the scope is appropriate for a 3-4 week project because it includes learning the existing MCP tool structure, refreshing my C# knowledge, implementing the tools, adding tests, documenting behavior, and responding to review feedback.

---

## Understanding the Issue

### Problem Description

Currently, agents often rely on grep or keyword matching to understand C# code. This is limited because text search does not understand symbols, overloads, references, or type relationships. RoslynMcp gives AI agents compiler-backed tools so they can reason about code semantically instead of matching strings.

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

At the start of this contribution, these dedicated tools did not exist. Agents had to approximate the answers using grep, keyword search, file reads, or combinations of existing broader tools like `roslyn_find_references` and `roslyn_get_type_members`.

This is less reliable because text search does not understand C# symbols, overloads, inheritance, interfaces, or type relationships.

### Affected Components

- `src/RoslynMcp/Tools/Analysis/` - where the new analysis tools should be implemented
- `src/RoslynMcp/Tools/RoslynMcpTool.cs` - shared helper patterns for resolving compilations and symbols
- `src/RoslynMcp/Tools/ToolResults.cs` - structured result types, if shared result records are added
- `src/TestHarness/Tests/` - tests for the new tool behavior
- `README.md` - tool catalog documentation
- `docs/AGENT-INSTRUCTIONS.md` - agent guidance for when to use the new tools
- `docs/plans/tool-suggestions.md` and `ROADMAP.md` - planning/status documentation

---

## Reproduction Process

### Environment Setup

I cloned my fork locally:

```text
https://github.com/rubinashaik2022/RoslynMcp.git
```

Local path:

```text
/Users/rubinashaik/RoslynMcp
```

Working branch:

```text
issue-19-semantic-analysis-tools
```

I built the MCP server locally using `net8.0` because my machine has .NET 8/9 installed, but not .NET 10. The README shows a `net10.0` publish example, so I adjusted the target framework for my local environment.

Commands used:

```bash
dotnet restore src/RoslynMcp/RoslynMcp.csproj -p:TargetFrameworks=net8.0 -r osx-arm64
dotnet restore src/RoslynMcp.Analyzers/RoslynMcp.Analyzers.csproj
dotnet publish src/RoslynMcp/RoslynMcp.csproj -c Release -f net8.0 -o ./publish/net8.0 -p:TargetFrameworks=net8.0 --no-restore
```

### Steps to Reproduce / Explore Current Functionality

1. Built the RoslynMcp server locally to confirm the development environment works.
2. Ran the published server with `./publish/net8.0/RoslynMcp --version` to confirm the MCP executable starts successfully.
3. Reviewed the existing analysis tools under `src/RoslynMcp/Tools/Analysis/`.
4. Confirmed that similar semantic tools already exist, including `roslyn_find_references`, `roslyn_find_callers`, `roslyn_get_call_graph`, and `roslyn_get_type_members`.
5. Searched the codebase for `roslyn_find_unused`, `roslyn_get_type_dependencies`, and `roslyn_find_overloads`.
6. Confirmed that these tools are mentioned in planning docs but are not currently implemented as MCP tools.

### Observed Result

The project already had the architecture needed for semantic C# analysis tools, but the three requested tools were missing from the implementation at the start of the contribution.

### Reproduction Evidence

- **Commit showing reproduction:** N/A - this is a feature gap rather than a reproducible bug.
- **Branch link:** https://github.com/rubinashaik2022/RoslynMcp/tree/issue-19-semantic-analysis-tools
- **My findings:** existing tools demonstrate the implementation pattern:
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

Some older text refers to "four new analysis tools." For this contribution, the relevant issue scope is three related tools:

- `roslyn_find_unused`
- `roslyn_get_type_dependencies`
- `roslyn_find_overloads`

`roslyn_check_syntax` appears to have been part of the earlier suggestion set, but it has already shipped in v0.8.0-beta.

Phase II completed `roslyn_get_type_dependencies` and `roslyn_find_overloads`. The PR for those two tools was merged. Week 4 completed `roslyn_find_unused`.

---

## Testing Strategy

### Integration Testing

Verified both projects compile and type-check with Roslyn MCP diagnostic/build tools.

#### `src/RoslynMcp/RoslynMcp.csproj`

```text
roslyn_get_diagnostics
Result: 0 errors, 0 warnings
```

```text
roslyn_build_project
Result: build succeeded
```

#### `src/TestHarness/TestHarness.csproj`

```text
roslyn_get_diagnostics
Result: 0 errors, 0 warnings
```

```text
roslyn_build_project
Result: build succeeded
```

### End-to-End Testing

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
All 84 tests passed
```

The new and updated tools were exercised through the MCP protocol as part of the `Type Understanding Tools` group.

### `roslyn_find_overloads` Coverage

The harness verifies that `roslyn_find_overloads` handles the following cases.

#### Simple Containing Type Name

```text
containingType = "RoslynMcpTool"
```

Verifies overload lookup works when the caller provides a simple type name.

#### Fully-Qualified Containing Type Name

```text
containingType = "RoslynMcp.Tools.RoslynMcpTool"
```

Verifies overload lookup works when the caller provides a fully-qualified type name.

#### Multiple Overloads

```text
methodName = "BeginTool"
```

Verifies all overload signatures are returned.

#### Generic Method Signatures

```text
BeginTool<T>(...)
```

Verifies generic overload signatures are included.

#### Optional/Default Parameter Values

```text
string? subject = null
```

Verifies default parameter values are preserved in the returned signatures.

#### `out` Parameters

```text
out Compilation? compilation
out ToolResult? error
```

Verifies `out` parameter modifiers are included in full signatures.

#### `ref` Parameters

```text
ref int skip
```

Verifies `ref` parameter modifiers are included in full signatures.

#### Missing Method

```text
methodName = "DefinitelyNotAMethod"
```

Verifies the tool returns an empty overload result instead of an error:

```text
total_overloads == 0
overloads is empty
```

### `roslyn_get_type_dependencies` Coverage

The harness verifies direct dependencies from the type declaration and member signatures only.

### Covered Dependency Categories

#### Base Type

```csharp
internal sealed class FindOverloadsTool : RoslynMcpTool
```

Verifies `RoslynMcpTool` is reported as:

```text
dependency_kind = "base_type"
```

#### Directly Implemented Interface

```csharp
internal sealed class _TypeDependenciesMemberFixture_<TItem> : _TypeDependenciesDirectInterface_
```

Verifies `_TypeDependenciesDirectInterface_` is reported as:

```text
dependency_kind = "interface"
```

#### Field Type

```csharp
private FileLogger? logger;
```

Verifies `FileLogger` is reported as:

```text
dependency_kind = "field"
member = "logger"
```

#### Property Type

```csharp
public Dictionary<string, TItem[]> Items { get; } = [];
```

Verifies the constructed dictionary type is reported as:

```text
dependency_kind = "property"
member = "Items"
```

#### Generic Type Arguments

```csharp
Dictionary<string, TItem[]>
```

Verifies `string` is reported as a dependency from the generic type argument list.

#### Event Type

```csharp
public event Action<ErrorResult>? Changed;
```

Verifies `ErrorResult` is reported as:

```text
dependency_kind = "event"
member = "Changed"
```

#### Constructor Parameter

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

#### Method Return Type

```csharp
public Project? Build<TResult>(Compilation compilation)
```

Verifies `Project` is reported as:

```text
dependency_kind = "method_return"
member = "Build"
```

#### Method Parameter Type

```csharp
public Project? Build<TResult>(Compilation compilation)
```

Verifies `Compilation` is reported as:

```text
dependency_kind = "method_parameter"
member = "Build"
```

#### Type Generic Constraint

```csharp
internal sealed class _TypeDependenciesMemberFixture_<TItem>
    where TItem : ToolResult
```

Verifies `ToolResult` is reported as:

```text
dependency_kind = "generic_constraint"
member = "TItem"
```

#### Method Generic Constraint

```csharp
public Project? Build<TResult>(Compilation compilation)
    where TResult : ToolResult
```

Verifies `ToolResult` is reported as:

```text
dependency_kind = "generic_constraint"
member = "Build"
```

#### User-Defined Operator Dependencies

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

#### Conversion Operator Dependencies

```csharp
public static explicit operator string(_TypeDependenciesOperatorFixture_ value)
```

Verifies `op_Explicit` reports the conversion return dependency:

```text
type_name = "string"
dependency_kind = "method_return"
member = "op_Explicit"
```

### Negative and Guard Coverage

#### Generic Placeholder Types Are Not Reported

```csharp
TItem
```

Verifies bare generic placeholder types are not returned as dependencies.

`TItem` is only a placeholder. The real dependency is the constraint:

```csharp
where TItem : ToolResult
```

So `ToolResult` is returned, but bare `TItem` is not.

#### Function Pointer Dependencies Are Out of Scope

Function pointer dependencies are intentionally not covered in v1.

Reason:

```text
Function pointers require unsafe code.
The v1 fixture set stays in safe C#.
```

#### Intentionally Out of Scope

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

### `roslyn_find_unused` Coverage

This section summarizes the integration and end-to-end test coverage added for `roslyn_find_unused`.

#### Harness Location

The tests live in:

- `src/TestHarness/Tests/FindUnusedTests.cs`

The test group is registered in:

- `src/TestHarness/TestHarnessProgram.cs`

The tests run through the existing MCP stdio harness, so they exercise the server process, tool registration, JSON-RPC `tools/call`, workspace loading, Roslyn reference analysis, result serialization, and response validation.

#### Fixture Strategy

The harness writes temporary fixture files into `src/RoslynMcp` before the test group runs, then deletes them during teardown.

Temporary files:

- `0_FindUnusedFixture_.cs`
- `0_FindUnusedFixture_.generated.cs`
- `0_FindUnusedSuffixFixture.g.cs`
- `0_FindUnusedDesignerFixture.designer.cs`

The `0_` prefix keeps fixture results early in sorted output so the default paged response includes them.

The fixture contains uniquely named symbols so tests can assert inclusion or exclusion without colliding with real project code.

#### Positive Coverage

The first harness test verifies that expected cleanup candidates are reported with confidence metadata.

Covered reported cases:

- Private unused method is reported.
- Private unused method has `confidence = high`.
- Internal unused method is reported.
- Internal unused method has `confidence = medium`.
- Internal unused method reason mentions friend assemblies when `InternalsVisibleToAttribute` is present.
- Internal unused constructor is reported.
- Internal unused constructor has `confidence = medium`.
- Public method inside an internal type is reported as effectively internal.
- Effectively internal public method has `confidence = medium`.
- Unreferenced internal containing type is reported.
- Reported entries include non-empty `reason`.

#### Conservative Skip Coverage

The second harness test verifies that high false-positive-risk symbols are not reported.

Covered skipped cases:

- Protected method.
- Member inside an attributed type.
- Method with its own attribute.
- Member inside a generated partial type.
- Partial method declaration/implementation pair.
- Interface implementation.
- Method in `.g.cs` generated suffix file.
- Method in `.designer.cs` generated suffix file.
- Virtual method.
- Abstract method.
- Override method, through shared unique virtual/abstract method names.
- Static `Main` convention entry point.
- Enum member.
- Child method suppressed when its containing type is already reported unused.
- Getter-referenced property.
- Setter-referenced property.
- Subscribed event.
- Member declared in one partial type part when another partial part is attributed.

#### End-To-End Verification

The full harness command:

```bash
dotnet run --project src/TestHarness/TestHarness.csproj -f net10.0
```

Final result:

```text
All 84 tests passed
```

Before the final assertion fix, the harness showed one failure in `roslyn_find_unused: reports conservative candidates with confidence metadata`. The failing assertion expected the constructor result to mention friend assemblies. That was incorrect because constructor results intentionally use the reflection/dependency-injection reason. The test was fixed to assert friend-assembly wording on the internal method instead.

After that correction, the full harness passed locally.

#### Build And Diagnostic Verification

During development, these checks were run:

- `roslyn_get_diagnostics` on `src/TestHarness/TestHarness.csproj`.
- `roslyn_build_project` on `src/TestHarness/TestHarness.csproj`.
- `roslyn_build_project` on `src/RoslynMcp/RoslynMcp.csproj`.
- Earlier multi-target verification included `RoslynMcp` default target and `net8.0`.

Results:

- Diagnostics were clean.
- Builds succeeded.
- `NU1900` warnings appeared because NuGet vulnerability metadata could not be fetched from `https://api.nuget.org/v3/index.json`.

The `NU1900` warnings were network/metadata warnings under restricted or unavailable network access; restore/build still succeeded.

#### Known Gaps

The current harness coverage is sufficient for the v1 conservative design.

Possible future tests:

- Pagination-specific behavior for `roslyn_find_unused`.
- Dedicated property/event result-shape tests if accessor-level behavior is ever added.
- A focused fixture proving `.generated.cs` suffix behavior separately from `<auto-generated` header behavior. The generated partial case currently uses both.
- `extern` method skip, if a practical fixture is added.

These are not required for the current tool contract.

#### Manual Testing

No separate manual UI/client testing was performed.

Validation was done through:

- Roslyn MCP diagnostics/build checks
- The automated MCP `TestHarness`, which runs end-to-end through MCP JSON-RPC

Manual exploratory testing was not needed because these are MCP tool responses with deterministic JSON output, and the new behavior is covered by targeted harness assertions for overload signatures, direct type dependency categories, and unused-symbol analysis.

---

## Implementation Notes

### Week 3 Progress

Implemented `roslyn_find_overloads` and `roslyn_get_type_dependencies` for RoslynMCP, added end-to-end tests, and opened a PR for these two tools only. That PR has been merged.

### Week 4 Progress

Implemented and tested `roslyn_find_unused`, which reports private/internal symbols with zero direct static references in the loaded solution. The tool was designed conservatively so agents get practical cleanup candidates without treating "zero references" as absolute proof that code is safe to delete.

Key design decisions:

- Prefer false negatives over false positives.
- Skip symbols that are generated, compiler-created, attributed, reflection-sensitive, inheritance-based, or convention-based.
- Use a confidence model instead of presenting every result as equally safe.
- Include caution text explaining that dynamic usage, reflection, external projects, and friend assemblies can still use symbols that have no direct static references.

The detailed implementation notes below document the final v1 behavior for `roslyn_find_unused`, including skipped symbol categories, generated code handling, reflection-sensitive cases, constructors, internal symbols, confidence levels, result shape, caution text, and the false-negative-over-false-positive design bias.

Testing was also completed in Week 4 through the MCP `TestHarness`. The new `FindUnusedTests.cs` coverage verifies both reported cleanup candidates and conservative skip cases, and the full harness passed with all 84 tests.

### Design Decisions

- **Direct dependencies only:** Defined dependencies as types directly referenced by a type's declaration, members, and signatures. The tool reports directly related types, but it does not recursively inspect those types to find their own dependencies. For example, if `OrderService` depends on `OrderRepository`, the tool reports `OrderRepository`; it does not then inspect `OrderRepository` and return every type that `OrderRepository` depends on. This keeps results scoped, predictable, and useful for AI consumption.
- **Generic type parameters:** Excluded generic placeholders such as `T`, `TKey`, and `TItem` because they are not concrete dependencies. Included constraint types because they represent actual type requirements and relationships.
- **Unsafe constructs:** Excluded function pointer internals because they require unsafe C#, while the project and current harness fixtures stay in safe C#.
- **Ordinary method overloads only:** Scoped `roslyn_find_overloads` to ordinary named methods. Constructors, operators, conversion operators, property accessors, event accessors, and compiler-generated methods are intentionally out of scope.
- **Empty results instead of errors:** Made missing-method overload lookup return an empty result instead of failing, so agents can safely ask exploratory questions.
- **Paged structured responses:** Followed the existing RoslynMcp pattern for structured, paged results so responses stay token-efficient and consistent with other tools.
- **Conservative unused-symbol analysis:** Designed `roslyn_find_unused` to report fewer, more explainable cleanup candidates. The tool skips cases where C# runtime behavior, generated code, reflection, inheritance, or framework conventions could make a zero-reference symbol unsafe to report as unused.

### `roslyn_find_unused`

`roslyn_find_unused` finds private/internal symbols with zero direct static references in the loaded solution.

#### Skipped Symbols

The tool skips compiler and language artifacts that are not useful cleanup targets:

- Implicitly declared symbols.
- Symbols without source locations.
- Enum fields.
- Associated symbols such as property accessors, event accessors, and backing fields.

#### Generated Code

Generated code is skipped, including:

- Symbols declared in `.g.cs`.
- Symbols declared in `.generated.cs`.
- Symbols declared in `.designer.cs`.
- Symbols declared in files whose first 1024 characters contain `<auto-generated`.
- Members of generated containing types.

Partial types are not skipped just because they are partial. Instead, the tool checks the symbol's source locations and containing types. If any relevant declaration is generated, the type or member is skipped.

#### Attributes and Reflection-Sensitive Code

The tool skips symbols with attributes and members of attributed containing types. This avoids reporting members that may be discovered by serializers, converters, dependency injection containers, test frameworks, command frameworks, source generators, or other reflection/convention-based systems.

For partial types, Roslyn aggregates attributes across partial declarations, so an attribute on any partial part causes the type and its members to be skipped.

#### Partial Methods

The tool skips methods with `PartialDefinitionPart` or `PartialImplementationPart`.

Partial methods can be split between declarations, implementations, and generated code. The conservative v1 rule is to skip both halves instead of trying to prove the pair is unused.

#### Protected and Dispatch-Based Members

The tool skips:

- `protected`, `protected internal`, and `private protected` members.
- Abstract methods.
- Virtual methods.
- Override methods.
- Extern methods.
- Interface implementations.
- Static `Main` methods.

These members may be used through inheritance, interface dispatch, runtime dispatch, external implementations, or runtime entry-point conventions.

#### Properties and Events

Properties and events are treated at symbol level. If either getter-style or setter-style usage exists, the property is considered referenced. The tool does not separately report unused getters, setters, add accessors, or remove accessors because accessor-level unused analysis would be noisier and would require a different result model.

#### Constructors

Constructors are included, but reported with medium confidence. They may have zero direct static references while still being activated by reflection or dependency injection.

```csharp
services.AddSingleton<UserService>();
```

In this example, a dependency injection container can inspect and call `UserService` constructors at runtime, even if source code never calls `new UserService(...)`.

#### Internal Symbols and Friend Assemblies

Internal symbols are included because the tool description explicitly includes internals. They are reported with medium confidence because other projects outside the loaded solution or friend assemblies declared through `InternalsVisibleToAttribute` may use them.

When the current assembly has `InternalsVisibleToAttribute`, internal-symbol reasons mention friend assemblies.

#### Confidence Model

The tool uses `high` and `medium` confidence.

- **High confidence:** private non-constructor symbols with zero direct static references.
- **Medium confidence:** internal symbols, effectively internal public members, constructors, and symbols in assemblies with friend assemblies.

There is no `low` confidence. If a result would be low confidence, the symbol is skipped instead.

#### Result Shape

Each unused entry includes:

- `kind`
- `name`
- `accessibility`
- `confidence`
- `reason`
- `file`
- `line`

The `reason` field explains why the symbol was reported at that confidence level.

#### Caution Text

The tool returns a caution explaining that only direct static references in the loaded solution are considered. The caution calls out dynamic usage, reflection-based usage, external projects, and external friend assemblies. If the workspace is loaded in Adhoc mode, the normal Adhoc caution is appended.

#### Design Bias

The tool intentionally prefers false negatives over false positives. It may miss real dead code when a symbol is reflection-sensitive, generated, attributed, partial, protected, inherited, or convention-based. This is preferable to encouraging agents to delete code that may be used through runtime mechanisms Roslyn reference search cannot fully prove.

The goal is practical refactoring cleanup guidance:

- Report fewer things.
- Keep findings explainable.
- Avoid common C# runtime and framework traps.
- Preserve the distinction between "zero direct static references" and "definitely unused."

### `roslyn_find_overloads`

`roslyn_find_overloads` returns the overload signatures for a method declared on a specific containing type.

#### Included

- Ordinary methods only.
- Multiple overloads with the same method name.
- Generic method signatures.
- Fully-qualified containing type names.
- Simple containing type names.
- Full parameter modifiers, including:
  - `ref`
  - `out`
- Nullable annotations in signatures.
- Default parameter values where Roslyn includes them in the formatted signature.

#### Not Included / Out of Scope

- Constructors.
- Operators.
- Conversion operators.
- Property accessors.
- Event accessors.
- Compiler-generated methods.
- Methods on unrelated types with the same name.

This tool is intentionally scoped to ordinary named method overloads. Operators/conversions are handled as type dependencies in `roslyn_get_type_dependencies`, but they are not treated as normal overloads here.

#### Tests Added

The harness verifies:

- Lookup by simple containing type:
  - `RoslynMcpTool`
- Lookup by fully-qualified containing type:
  - `RoslynMcp.Tools.RoslynMcpTool`
- Multiple overloads for `BeginTool`
- Generic overload signature:
  - `BeginTool<T>(...)`
- Optional/default parameter signature:
  - `string? subject = null`
- `out` parameters:
  - `out Compilation? compilation`
  - `out ToolResult? error`
- `ref` parameters:
  - `ref int skip`
- Missing method:
  - returns `total_overloads == 0`
  - returns an empty `overloads` array

### `roslyn_get_type_dependencies`

`roslyn_get_type_dependencies` returns the types directly referenced by a type declaration and its direct member signatures.

The key design decision is that this tool reports **direct dependencies only**. It does not recursively walk into each dependency and return that dependency's dependencies.

For example:

```csharp
class Example
{
    public Dictionary<string, Foo> Items { get; }
}
```

The tool reports:

```text
Dictionary<string, Foo>
string
Foo
```

It does **not** inspect `Foo` and return every type that `Foo` depends on.

#### Included

The tool includes direct type references from:

- Base type
- Directly implemented interfaces
- Field types
- Property types
- Event types
- Constructor parameter types
- Method return types
- Method parameter types
- Type generic constraints
- Method generic constraints
- Generic type arguments
- Array element types
- Pointer element types
- User-defined operator return/parameter types
- Conversion operator return/parameter types

#### Direct Interfaces Only

The tool uses direct interfaces, not inherited/transitive interfaces.

Example:

```csharp
interface IA { }
interface IB : IA { }

class Example : IB { }
```

The tool reports:

```text
IB
```

It does not report:

```text
IA
```

because `IA` is indirect.

#### Generic Placeholders vs Constraint Types

Generic placeholder types are not reported as dependencies.

Example:

```csharp
class Store<TItem>
{
    public TItem Item { get; }
}
```

The tool does not report:

```text
TItem
```

because `TItem` is only a placeholder.

Constraint types are reported.

Example:

```csharp
class Store<TItem>
    where TItem : ToolResult
{
}
```

The tool reports:

```text
ToolResult
```

because `ToolResult` is a real type dependency declared in the constraint.

#### Operators and Conversions

User-defined operators and conversion operators are included because they are declared directly on the type and have real parameter/return types.

Example:

```csharp
public static Money operator +(Money left, Money right)
```

The tool reports the operator return and parameter types as direct dependencies.

Example:

```csharp
public static explicit operator string(UserId id)
```

The tool reports:

```text
string
UserId
```

These are represented by Roslyn as special method kinds, not ordinary methods, but they still contribute direct type dependencies.

#### Function Pointers

Function pointer internals are intentionally not included in v1.

Reason:

- Function pointers require unsafe code.
- This project does not allow unsafe blocks.
- The current fixture set stays in safe C#.
- Supporting function pointer internals would require additional unsafe test infrastructure.
- This is out of scope for the current implementation.

This can be reconsidered in a future version if the project adds an unsafe fixture or decides to support unsafe dependency analysis explicitly.

#### Not Included / Out of Scope

The tool intentionally does not include:

- Nested type dependencies
- Attributes
- Method body locals
- Method body calls
- Called method return types
- Dependencies of dependencies
- Transitive interfaces
- Bare generic placeholder types such as `TItem`
- Implicit `object` base type
- Implicit `ValueType` base type
- Implicit `Enum` base type
- Function pointer internals

### Response Shape

`roslyn_get_type_dependencies` follows the same paged result shape used by other tools:

```json
{
  "type_name": "Example",
  "type_kind": "class",
  "total_dependencies": 10,
  "skip": 0,
  "take": 50,
  "dependencies": [
    {
      "type_name": "SomeDependency",
      "dependency_kind": "field",
      "member": "someField"
    }
  ],
  "page_token": null,
  "has_more": false
}
```

Each dependency records:

- `type_name`: the referenced type
- `dependency_kind`: why the type was referenced
- `member`: the member where the dependency was found, or `null` for type-level dependencies

### Tests Added

The harness verifies `roslyn_get_type_dependencies` reports:

- Base type dependencies
- Direct interface dependencies
- Field dependencies
- Property dependencies
- Event dependencies
- Constructor parameter dependencies
- Method return dependencies
- Method parameter dependencies
- Type generic constraint dependencies
- Method generic constraint dependencies
- Generic type argument dependencies
- User-defined operator dependencies
- Conversion operator dependencies

The harness also verifies:

- Bare generic placeholders such as `TItem` are not returned
- Missing or indirect dependency categories remain out of scope

### Code Changes

- **Files modified:** implementation, tests, and documentation files for `roslyn_find_overloads`, `roslyn_get_type_dependencies`, and `roslyn_find_unused`.
- **Current status:** `roslyn_find_unused` is implemented; continue review/iteration as needed.
- **Key commits for PR #211 (`roslyn_find_overloads` and `roslyn_get_type_dependencies`):**
  - [`a0c5d56`](https://github.com/MadQ/RoslynMcp/commit/a0c5d568db00dc0f89ceb704b1a3b073e3760b65) - implemented `roslyn_find_overloads`
  - [`94687ac`](https://github.com/MadQ/RoslynMcp/commit/94687ac50933fb97b205350aa01ce5a217223349) - added helper support for full overloaded method signatures
  - [`f5dcd87`](https://github.com/MadQ/RoslynMcp/commit/f5dcd878da6454b7dc0a206fcc531a759a153e0a) - added tests for the overload tool
  - [`ed4859e`](https://github.com/MadQ/RoslynMcp/commit/ed4859ef840ab2c87ef0e37da91763cc97e6904e) - added `roslyn_get_type_dependencies` and tests
  - [`fb82415`](https://github.com/MadQ/RoslynMcp/commit/fb82415267ea183e6e0907f54550788b1a2b8dea) - included operator and conversion dependencies
  - [`76a4180`](https://github.com/MadQ/RoslynMcp/commit/76a41804ae50fd8b18362e03e90a787c4a9e93bb) - fixed tests and added coverage for `roslyn_get_type_dependencies`
  - [`103ae6b`](https://github.com/MadQ/RoslynMcp/commit/103ae6bb81e0e422f10571bbf401462605f242b4) - updated docs for the new tools
  - [`016919b`](https://github.com/MadQ/RoslynMcp/commit/016919b1d46c76803c943d0eb60a4729fef9d712) - updated changelog
- **Key commits for PR #212 (`roslyn_find_unused`):**
  - [`10e53db`](https://github.com/MadQ/RoslynMcp/commit/10e53dbd7c84ee6e94af9a00a4137c96e85735e0) - hardened `roslyn_find_unused` edge-case handling, including generated/attributed symbols, effective internal visibility, and confidence/reason metadata
  - [`5b5cb9f`](https://github.com/MadQ/RoslynMcp/commit/5b5cb9f964f6f64de6c2bb58d328c39fa189da80) - added tests for `roslyn_find_unused`
  - [`82102f3`](https://github.com/MadQ/RoslynMcp/commit/82102f35a062a48e1db767899c98a1caebd0781d) - updated documentation for `roslyn_find_unused`
- **Approach decisions:** kept `roslyn_find_overloads` scoped to ordinary named methods, and kept `roslyn_get_type_dependencies` focused on direct dependencies from declarations and member signatures.

---

## Pull Requests

### PR #211

**PR Link:** https://github.com/MadQ/RoslynMcp/pull/211

**PR Description:** Adds `roslyn_find_overloads` and `roslyn_get_type_dependencies`, including implementation, targeted MCP `TestHarness` coverage, and documentation updates. This PR only covers those two tools.

**Status:** Merged on June 22, 2026

### PR #212

**PR Link:** https://github.com/MadQ/RoslynMcp/pull/212

**PR Description:** Adds `roslyn_find_unused`, including conservative unused-symbol analysis, confidence/reason metadata, TestHarness coverage, and documentation updates.

**Status:** Merged on June 23, 2026

Positive feedback received on both PRs and was merged. 

---

## Learnings & Reflections

### Technical Skills Gained

I expanded my C# knowledge by working with Roslyn's semantic model and symbol APIs. This included resolving symbols, finding references with `SymbolFinder`, formatting method signatures, identifying direct type dependencies, and distinguishing source-declared symbols from compiler-generated or generated-code artifacts.

I also gained a better understanding of how MCP servers work and how MCP tools are structured. This included tool registration, request/response models, paged results, JSON-RPC tool calls, and end-to-end validation through the existing `TestHarness`.

### Challenges Overcome

The hardest part was defining the correct scope before coding. Planning out what the tool should and should not include turned out to be just as important as the implementation itself.

For example, with `roslyn_get_type_dependencies`, the first instinct was to let the tool find every nested or transitive dependency. However, the intended scope was direct dependencies only. Making that distinction early kept the tool predictable and aligned with the issue description.

Another challenge was thinking through edge cases for `roslyn_find_unused`. A symbol can have zero direct static references but still be used through reflection, dependency injection, inheritance, generated code, or framework conventions. To handle that safely, I designed the tool to be conservative, include confidence metadata, and skip cases that had a high false-positive risk.

### What I'd Do Differently Next Time

Next time, I would spend even more time writing down scope decisions before implementing. AI can help generate ideas and code, but I should not over-rely on it to decide the behavior of the tool. The developer still needs to define the contract, review whether suggestions match the issue, and think carefully about edge cases before coding.

I would also create the test fixture plan earlier. The `roslyn_find_unused` tests required many edge cases, and planning those cases upfront would make the implementation path clearer.


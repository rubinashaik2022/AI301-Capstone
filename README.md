# Contribution [#1]: Implement roslyn_find_unused and supporting analysis tools

**Contribution Number:** 1
**Student:** Rubina Shaik
**Issue:** https://github.com/MadQ/RoslynMcp/issues/33
**Status:** Phase I completed

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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

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
- **Screenshots/logs:** [If applicable]
- **My findings:**
This is a feature gap rather than a runtime bug. Existing tools demonstrate the implementation pattern:

- `FindReferencesTool.cs` shows how to use `SymbolFinder.FindReferencesAsync`.
- `TypeMembersTool.cs` shows how to resolve a type and inspect members.
- `GetCallGraphTool.cs` shows how to inspect semantic dependencies using Roslyn.
- `FindCallersTool.cs` shows async Roslyn lookup and structured/paged results.

The new tools should follow these existing patterns.
---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

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

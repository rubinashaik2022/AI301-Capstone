# Contribution [2]: feat: roslyn_apply_code_fix — bulk-apply Roslyn CodeFix providers without agent reasoning

**Contribution Number:** [2]  
**Student:** [Rubina Shaik]  
**Issue:** [https://github.com/MadQ/RoslynMcp/issues/110]  
**Status:** [Phase II completed]
---

## Why I Chose This Issue

I chose this issue because it connects directly to my interest in AI developer tools and Model Context Protocol (MCP), which is an area I expect to work with in my upcoming job. RoslynMcp is a good fit because it already has an MCP server in place, so the project is focused on adding useful tools rather than building the entire system from scratch.

---

## Understanding the Issue

### Problem Description

Right now, roslyn_get_diagnostics surfaces diagnostics, but fixing them requires for an AI agent to reason about the fixes and call replace_in_code. This is slow and expensive. roslyn_apply_code_fix will apply the Roslyn CodeFixProvider fixes automatically, without the agent having to do any reasoning about the edits. This lowers expenses significantly and is highly efficient. 

### Expected Behavior
RoslynMcp should discover, preview, and apply Roslyn code fixes outside VS Code, including Fix All where supported.

### Current Behavior
Diagnostics are available, but their matching CodeFixProviders are not exposed or applied by RoslynMcp.

### Affected Components
MSBuildWorkspace, diagnostics, analyzer loading, .editorconfig options, solution diffs, and the preview/apply workflow.

---

## Reproduction Process

### Environment Setup

I used my local fork of RoslynMcp:

```text
https://github.com/rubinashaik2022/RoslynMcp.git
```

Local path:

```text
/Users/rubinashaik/RoslynMcp
```

Working branch:

```text
issue-110-apply-bulk-code-fix
```

The issue being researched is:

```text
https://github.com/MadQ/RoslynMcp/issues/110
```

### Steps to Reproduce

1. Open the RoslynMcp issue for `roslyn_apply_code_fix`: https://github.com/MadQ/RoslynMcp/issues/110.
2. Read the current behavior described in the issue: `roslyn_get_diagnostics` can surface diagnostics, but RoslynMcp does not currently expose or apply the matching Roslyn `CodeFixProvider` fixes.
3. In the local RoslynMcp repo, inspect the diagnostics implementation to confirm the current path only reports diagnostics and does not collect code fix actions.
4. Inspect the in-repo analyzer project and confirm that Roslyn code fix providers already exist, including providers for `RMCP001` / `RMCP002` and `RMCP003` through `RMCP006`.
5. Confirm that these code fix providers expose `FixableDiagnosticIds` and register suggested `CodeAction`s through Roslyn's `CodeFixContext` / `RegisterCodeFixesAsync` pattern.
6. Compare this with the existing preview/apply tools, especially rename and signature-change, to confirm RoslynMcp already has a safe pattern for previewing a changed `Solution`, storing an approval token, and applying changes later.
7. Observe the gap: RoslynMcp has diagnostics and safe apply infrastructure, but it does not yet have a host/wrapper that asks matching `CodeFixProvider`s for their available fixes and turns a selected `CodeAction` into a previewable diff.

### Reproduction Evidence

- **Branch link:** https://github.com/rubinashaik2022/RoslynMcp/tree/issue-110-apply-bulk-code-fix
- **Issue link:** https://github.com/MadQ/RoslynMcp/issues/110
- **My findings:** CodeFixProviders can be used outside VS Code with `MSBuildWorkspace`. Analyzer NuGet packages are detected, but RoslynMcp still needs to explicitly load and run their fixes. `.editorconfig` settings feed through correctly. FixAll is possible, but should be deferred until after the single-fix pipeline is working because large FixAll runs may touch many files and need caps, preview summaries, and performance testing.

---

## Solution Approach

### Analysis

The root issue is that RoslynMcp can currently report diagnostics, but it does not expose the matching Roslyn code fixes as MCP tools. Right now, when an AI agent sees a diagnostic, it has to read the surrounding code, reason about the correct edit, and then call a lower-level editing tool such as `roslyn_replace_in_code`. That uses more tokens, takes more steps, and increases the risk that the agent makes a weaker edit than Roslyn's own fix provider would make.

Roslyn already has `CodeFixProvider`s that know how to fix many diagnostics. These are the same kind of suggested fixes shown by IDE lightbulbs. The missing piece is a RoslynMcp wrapper/tool that can find the providers for a diagnostic, collect their available fixes, preview the selected fix, and apply it safely.

### Proposed Solution

Add a new preview/apply code-fix workflow to RoslynMcp. The first phase should focus on applying a single selected fix for one diagnostic at a specific file location. The tool will ask matching `CodeFixProvider`s for available `CodeAction`s, return choices when more than one fix exists, generate a diff for the selected fix, and only write files after approval.

This makes the workflow more token efficient because the AI agent no longer has to manually reason through every code edit. Instead, the agent can use Roslyn's built-in code-fix logic and RoslynMcp's existing safe preview/apply pattern.

### Implementation Plan

For the first PR, I plan to focus on the simple single-fix pipeline instead of FixAll. The goal is to prove RoslynMcp can behave like the IDE lightbulb for one diagnostic at one location.

- Add a small `CodeFixHost` service that wraps Roslyn's `CodeFixProvider` API behind a simpler method such as `GetFixesAsync(document, diagnostic, cancellationToken)`.
- The wrapper will loop through known `CodeFixProvider`s, check whether each provider supports the diagnostic ID, create a `CodeFixContext`, and use the context callback to collect the provider's registered `CodeAction`s into a list.
- Add a preview tool that targets one diagnostic by project path, file path, line, column, and optional diagnostic ID. If one fix exists, it will preview that fix. If multiple fixes exist, it will return the choices and require an `actionIndex` instead of guessing.
- Execute the selected `CodeAction` in memory, extract the changed `Solution`, build a unified diff, and store it behind an approval token using the existing preview/apply pattern already used by rename and signature-change tools.
- Add an apply tool that consumes the token, supports approval values `y`, `n`, and `session`, saves backups before writing, and applies the changed solution safely.
- Test first with the in-repo analyzer code fixes, especially `RMCP001` / `RMCP002`, because those providers are already available and give a clear single-fix case.
- Defer FixAll to a second phase. FixAll should build on the selected single-fix action and use the provider's FixAll support, but it needs preview counts, changed-file caps, timeout/cancellation behavior, and explicit opt-in for large project or solution-wide edits.

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

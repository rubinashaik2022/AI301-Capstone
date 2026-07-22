# Contribution 2: Roslyn code-fix preview and apply workflow

**Contribution Number:** 2  
**Student:** Rubina Shaik  
**Issue:** [MadQ/RoslynMcp #110](https://github.com/MadQ/RoslynMcp/issues/110)  
**Development Branch:** [issue-110-apply-bulk-code-fix](https://github.com/rubinashaik2022/RoslynMcp/tree/issue-110-apply-bulk-code-fix)  
**Project Status:** Phase III — In Progress  
**Tool Status:** Phase I targeted code-fix workflow is implemented but still requires final verification, partial-write reporting, workspace-token binding, focused tests, and cleanup before the first PR.  
**Delivery Plan:** PR 1 will complete and submit Phase I of the tool. A separate PR 2 will implement Phase II Fix All support after the targeted preview/apply pipeline is fully validated.

> **Development note:** This contribution is not complete yet. The current branch contains the main Phase I implementation and stale-file safety work, but the first review round should not begin until the remaining Phase I completion items are addressed. Fix All is intentionally separated into a later PR so the first PR stays focused on a safe, fully tested single-diagnostic workflow.

---

## Why I Chose This Issue

I chose this issue because it combines AI developer tooling, the Model Context Protocol (MCP), and Roslyn's semantic code-analysis APIs. RoslynMcp already exposes diagnostics and code-intelligence tools to AI agents, but agents still have to interpret many diagnostics themselves and construct edits through lower-level tools. Exposing Roslyn's own `CodeFixProvider` actions allows an agent to use the same semantic fixes that power IDE lightbulbs while retaining RoslynMcp's explicit preview, approval, backup, and recovery workflow.

This work is relevant to my interest in MCP-based development tools and to the kind of AI-assisted engineering systems I expect to encounter professionally. It also required working across several layers of an existing codebase: analyzer discovery, Roslyn workspaces, immutable solutions, MCP tool design, disk safety, approval state, backups, and integration testing.

---

## Understanding the Issue

### Problem Description

`roslyn_get_diagnostics` can report compiler and analyzer diagnostics, but RoslynMcp previously did not expose the matching `CodeFixProvider` actions. An AI agent therefore had to:

1. Read the diagnostic and surrounding source.
2. Infer the intended repair.
3. Construct an edit manually.
4. Call a lower-level editing tool such as `roslyn_replace_in_code`.
5. Recheck diagnostics and potentially revise the edit.

That workflow costs additional requests and tokens, duplicates logic already encoded in Roslyn analyzers, and may produce a weaker edit than the analyzer author's own fix.

### Expected Behavior

RoslynMcp should be able to discover code fixes for a diagnostic, present available actions, calculate the selected change in memory, show a unified diff, and write the exact reviewed change only after approval.

Phase I targets one diagnostic at a specific source location. Phase II will add Fix All at document, project, and solution scope after the shared apply pipeline has sufficiently strong multi-file verification and partial-failure reporting.

### Previous Behavior

RoslynMcp exposed diagnostics but did not ask matching `CodeFixProvider` instances to register their actions. It had no MCP tool that translated a selected `CodeAction` into a previewable and approvable changed solution.

### Affected Components

- `MSBuildWorkspace` and `AdhocWorkspace`
- Compiler and analyzer diagnostics
- Analyzer and code-fix provider loading
- Roslyn `CodeFixContext`, `CodeAction`, and `ApplyChangesOperation`
- Immutable `Solution` snapshots
- Unified solution diffs
- Approval tokens and session approval
- Local-history backups
- Physical file creation, modification, and deletion
- Workspace invalidation and recovery
- Test harness integration

---

## Roslyn Concepts Behind the Feature

### Diagnostic

A Roslyn diagnostic is a compiler or analyzer finding associated with source code. It can represent an error, warning, style issue, or project-specific rule. For example, RoslynMcp's `RMCP001` diagnostic reports the use of `IntPtr` where the project prefers `nint`.

A diagnostic explains what Roslyn found, but it does not itself contain the repair.

### CodeFixProvider

A `CodeFixProvider` is the component that knows how to offer fixes for one or more diagnostic IDs. It is similar to the provider behind an IDE lightbulb suggestion.

The provider declares which diagnostics it supports through `FixableDiagnosticIds`. When RoslynMcp gives it a diagnostic and document through `CodeFixContext`, the provider may register one or more possible `CodeAction` objects.

In plain language:

```text
Diagnostic       = the problem Roslyn found
CodeFixProvider  = the component that knows possible repairs
CodeAction       = one specific repair offered by that provider
```

### CodeAction

A `CodeAction` represents one proposed fix. It is not the diff and it does not immediately edit a file. It is a recipe that Roslyn can execute in memory to calculate one or more operations.

For the normal case used by this feature, executing the action produces an `ApplyChangesOperation` containing a changed Roslyn `Solution`.

Example:

```text
Diagnostic:
    RMCP001: Prefer nint over IntPtr

CodeFixProvider:
    PreferNintOverIntPtrCodeFixProvider

CodeAction:
    Replace IntPtr with nint
```

RoslynMcp calculates the action during preview. No physical source file is written at that point.

### BaseSolution and ChangedSolution

A Roslyn `Solution` is an immutable in-memory snapshot containing projects, documents, and source state.

`BaseSolution` is the original solution snapshot before the selected fix is calculated.

`ChangedSolution` is the new in-memory solution snapshot produced by the selected `CodeAction`. It represents what the solution should look like after the proposed fix.

```text
BaseSolution
    original in-memory state
        |
        | execute CodeAction in memory
        v
ChangedSolution
    computed in-memory state after the proposed fix
```

The word "changed" does not mean the physical files have already changed. Both snapshots are immutable and remain in memory until the apply tool explicitly persists the proposed solution.

RoslynMcp compares the two snapshots to determine:

- Which documents were modified
- Which documents were added
- Which documents were removed
- What unified diff should be shown to the caller
- Which physical paths the action may eventually touch

### Diff

The unified diff is a human-readable comparison between `BaseSolution` and `ChangedSolution`. It is returned so the caller can review the proposal.

The diff is not replayed as a text patch. RoslynMcp stores and later applies the exact `ChangedSolution` that produced the reviewed diff.

---

## Code-Fix Selection and Approval Cases

### No matching diagnostic

If no diagnostic matches the requested file location and optional diagnostic ID, preview returns a structured error. No action is calculated and no approval token is created.

### Diagnostic has no loaded fix provider

The diagnostic may be valid even when none of RoslynMcp's loaded providers supports it. Preview returns `no fixes`. No token is created.

### Exactly one fix is available

RoslynMcp selects the only action, calculates its changed solution, generates the diff, captures affected file states, and returns an approval token.

### More than one fix is available

Different providers—or one provider—can offer several valid repairs for the same diagnostic. RoslynMcp does not guess which one the caller intended.

Instead, preview returns choices containing:

- Action index
- Action title
- Provider name
- Diagnostic ID and message
- Equivalence key, when supplied

No approval token is created yet. The caller reruns preview with the selected `actionIndex`, and only that action is calculated and tokenized.

### Action calculates no file changes

If the selected action produces no changed solution or no meaningful diff, preview returns a structured `no changes` result. No token is created.

### Preview cannot capture a safe file baseline

If an affected file disappears, becomes inaccessible, or conflicts with a proposed new path while preview is calculating hashes, preview stops without issuing a token. This prevents approval of a proposal whose original disk state is already unknown.

---

## Current Approval-Token Lifecycle

The current implementation stores tokens in memory for the lifetime of the server process.

```text
Preview succeeds
    -> Pending token
    -> approved/rejected/error
```

### Cases where the token is not consumed

The token remains pending when apply stops before the current point of no return:

- Approval value is invalid
- The preview is stale because an affected path changed
- Backup creation fails before source mutation

The stale token technically remains stored, but retrying it is normally not useful unless the affected path returns exactly to its preview state. Generating a fresh preview is safer.

### Cases where the token is removed without applying

Approval `n` rejects and removes the token. No files are changed.

### Current point of consumption

For approval `y` or `session`, apply currently performs:

1. Token lookup
2. Affected-file stale validation
3. Pre/post backup preparation
4. Token consumption
5. Workspace writes and file deletions

The token is consumed after backups succeed but before the first source write. This prevents two simultaneous requests from applying the same token.

### Successful application

The token remains consumed after all intended changes are applied.

### Write failure before any file changes

Because the token was already consumed at the current point of no return, it cannot be reused even if the workspace fails before changing a physical file. The caller must generate a new preview.

This is safe but not ideal for retry experience.

### Partial multi-file write

If some files are written before another file fails, the token remains consumed. Reusing the original token would be unsafe because the workspace no longer matches the original preview baseline.

Backups allow recovery, but exact per-file partial-result reporting remains Phase I work.

### Deletion failure

If content files are written but a later physical deletion fails, the token remains consumed. The current result reports the written-file count, completed deletion count, failed deletion path, and local-history recovery guidance.

### Planned token improvement

A lower-priority future improvement is a three-state lifecycle:

```text
Pending -> Applying -> Consumed
              |
              +--> Pending only when zero physical changes are verified
```

This would make transient zero-write failures retryable while still consuming tokens after successful, partial, or uncertain writes. It is deferred until exact post-write verification can reliably prove whether any physical change occurred.

---

## Reproduction Process

### Environment Setup

Fork:

```text
https://github.com/rubinashaik2022/RoslynMcp.git
```

Local checkout:

```text
/Users/rubinashaik/RoslynMcp
```

Working branch:

```text
issue-110-apply-bulk-code-fix
```

Issue:

```text
https://github.com/MadQ/RoslynMcp/issues/110
```

### Steps Used to Confirm the Gap

1. Inspected the diagnostics pipeline and confirmed it reported diagnostics without collecting code-fix actions.
2. Inspected the in-repository analyzer project and confirmed that code-fix providers already existed for RoslynMcp diagnostics.
3. Confirmed that the providers expose `FixableDiagnosticIds` and register `CodeAction` instances through `CodeFixContext`.
4. Compared the missing code-fix workflow with the existing rename and signature-change preview/apply tools.
5. Confirmed that Roslyn's immutable changed `Solution` could be previewed without writing files.
6. Identified the required wrapper: match a diagnostic to providers, collect actions, calculate the selected action, store the changed solution, and apply it only after approval.

### Reproduction Evidence

- [Issue #110](https://github.com/MadQ/RoslynMcp/issues/110)
- [Active development branch](https://github.com/rubinashaik2022/RoslynMcp/tree/issue-110-apply-bulk-code-fix)
- [Initial targeted code-fix implementation commit](https://github.com/rubinashaik2022/RoslynMcp/commit/06733d5)
- [Stale-file protection commit](https://github.com/rubinashaik2022/RoslynMcp/commit/e1ecdc6)

---

## Solution Approach

### Phase I Scope

Phase I implements a targeted, single-diagnostic workflow rather than Fix All. This keeps the first version understandable and provides a safe foundation for later multi-file and solution-wide operations.

The two MCP tools are:

```text
roslyn_preview_code_fix
roslyn_apply_code_fix
```

The workflow deliberately mirrors rename and signature change:

```text
Diagnostic and source location
        -> discover CodeFixProvider actions
        -> select a CodeAction
        -> calculate ChangedSolution in memory
        -> compare BaseSolution and ChangedSolution
        -> return unified diff and approval token
        -> validate affected disk state
        -> save recovery snapshots
        -> apply only after explicit approval
```

### Why Preview and Apply Are Separate

A code fix may affect more than the source token where the diagnostic appears. The two-step workflow provides several safeguards:

- The caller reviews a unified diff before mutation.
- The exact `ChangedSolution` reviewed during preview is stored with the token.
- Apply does not recalculate the action or replay a text patch.
- Explicit approval is required before disk writes.
- Backups are created before source mutation.
- Stale affected files are rejected rather than overwritten.

### Why the Changed Solution Is Stored

Roslyn `Solution` objects are immutable snapshots. Executing a `CodeAction` produces a proposed `ChangedSolution`, but it does not alter physical files. The diff is generated by comparing:

```text
BaseSolution -> ChangedSolution
```

The diff is for review; the stored `ChangedSolution` remains the semantic source of truth. This avoids text-patch context failures and preserves Roslyn's representation of modified, added, and removed documents.

---

## Architecture and Pipeline

### 1. Resolve the Source Document and Diagnostic

`roslyn_preview_code_fix` accepts a project path, file path, line, column, optional diagnostic ID, and optional action index. The tool resolves the project through `WorkspaceResolver`, locates the document, gathers compiler and analyzer diagnostics, and selects the diagnostic closest to the requested position.

When several diagnostics match the location, the tool returns the choices and asks the caller to provide a diagnostic ID rather than guessing.

### 2. Discover Matching Code Fixes

`CodeFixHost` owns the loaded `CodeFixProvider` instances. For each provider it:

1. Checks whether `FixableDiagnosticIds` contains the diagnostic ID.
2. Creates a Roslyn `CodeFixContext`.
3. Calls `RegisterCodeFixesAsync`.
4. Captures the provider's registered `CodeAction` instances.
5. Records the provider name, title, equivalence key, diagnostic ID, and action index.

If several actions are available, preview returns them and requires an explicit `actionIndex`. This prevents RoslynMcp from silently choosing between fixes with different semantics.

### 3. Calculate the Code Action in Memory

A `CodeAction` is a recipe, not a diff and not a file write. Preview calls `GetOperationsAsync` and extracts the `ChangedSolution` from an `ApplyChangesOperation`.

The current providers produce the standard single `ApplyChangesOperation` shape. A small remaining hardening task is to explicitly reject zero, multiple, or custom operations rather than relying on `SingleOrDefault` behavior.

### 4. Generate the Unified Diff

`SolutionDiff.BuildAsync` compares the immutable base and changed solutions. The caller reviews this diff, but apply later uses the stored changed solution directly.

### 5. Translate Roslyn Documents into Physical File Effects

Roslyn reports document-level changes:

```csharp
GetChangedDocuments()
GetAddedDocuments()
GetRemovedDocuments()
```

A Roslyn document is not always a unique disk file. Linked files and multi-targeted projects can contain several `Document` objects that point to one physical path.

```text
Project A document ----+
                       +--> /repo/Shared.cs
Project B document ----+
```

The implementation therefore reasons about physical paths across the entire solution:

- A modified document normally rewrites an existing file.
- An added document may create a new file or add another link to an existing file.
- A removed document may remove one project link or remove the final solution reference.
- A physical file is deleted only when the changed solution contains no remaining document for that path.
- Duplicate physical paths are grouped so one file is processed once.

### 6. Capture Affected File States

The preview stores an expected physical state for every path that the action intends to touch:

```csharp
internal enum ExpectedFileState
{
    Exists,
    Absent
}

internal sealed record PreviewFileState(
    ExpectedFileState ExpectedState,
    string? ContentHash);
```

| Proposed operation | Preview expectation |
|---|---|
| Modify a file | Exists with its original SHA-256 |
| Add a new file | Remains absent |
| Remove a file | Exists with its original SHA-256 |
| Add another link to an existing file | Exists with original SHA-256 |
| Remove one of several links | No physical deletion expectation |

SHA-256 is calculated over exact disk bytes, so source, comments, whitespace, line endings, and encoding changes are all detected.

Only affected paths are tracked. Unrelated repository changes do not invalidate a preview. The goal is to prevent destructive overlap with concurrent work, not to freeze the entire solution.

### 7. Store the Pending Operation

`ApprovalStore` keeps the following data in memory:

- Base solution
- Changed solution
- Unified diff
- Session-approval key
- Affected physical-file states

The returned token identifies the exact proposal reviewed by the caller.

### 8. Reject Stale Previews

Before backups, token consumption, or writes, apply rechecks every affected path.

For an expected existing path:

```text
The path must exist.
Its current SHA-256 must equal its preview SHA-256.
```

For an expected absent path:

```text
The path must still be absent.
```

If an affected file changed, disappeared, or unexpectedly appeared, apply returns `stale preview` and writes nothing.

### 9. Create Recovery Snapshots

Backups are created before source mutation:

| Physical operation | Pre snapshot | Post snapshot |
|---|---:|---:|
| Modify | Original contents | Intended contents |
| Create | None | Intended contents |
| Delete | Original contents | None |

Pre snapshots support rollback. Post snapshots can restore or complete intended content after a failed write. A newly created file has no pre snapshot, so rolling it back means deleting it.

### 10. Apply Changes in Both Workspace Modes

In MSBuild mode, the changed solution is passed through the workspace apply path. In Adhoc mode, RoslynMcp writes changed and added documents explicitly through `SolutionDiff`, `FileWriter`, and workspace invalidation.

Removed paths are collected from `BaseSolution`, because they no longer exist in `ChangedSolution`. A path is physically deleted only after its final solution reference disappears. Deleted paths are invalidated from the workspace, and the response reports a deleted-file count.

### 11. Truncation Recovery

The existing RoslynMcp write pipeline protects against catastrophic truncation, where a workspace write appears successful but leaves a file missing or empty. RoslynMcp can rewrite the intended bytes through a temporary file and atomically replace the damaged target.

The current check is size-oriented. It detects catastrophic empty-file damage but does not yet prove that every non-empty file exactly matches the intended content. Exact post-write SHA-256 verification is the highest-priority remaining Phase I improvement.

---

## Implementation Notes

### Initial Implementation

The first implementation added:

- `CodeFixHost` for loading providers and collecting registered actions.
- `roslyn_preview_code_fix` for locating diagnostics, returning action choices, calculating a selected `CodeAction`, building a unified diff, and issuing an approval token.
- `roslyn_apply_code_fix` for approval handling, backups, workspace application, truncation checks, and recovery guidance.
- Dependency-injection registration for `CodeFixHost`.
- A runtime project reference to the analyzer project so the server can instantiate its code-fix provider types while retaining the existing analyzer reference for diagnostic execution.
- Test-harness registration and a focused end-to-end test for `RMCP001` (`IntPtr` to `nint`).

### Stale-File Protection and File Removal

The follow-up implementation added:

- Typed expected file states instead of a hash-only dictionary.
- Original SHA-256 capture for modified and removed paths.
- Explicit absence tracking for genuinely new paths.
- Protection against a new file appearing after preview.
- Protection against modified or removed files changing or disappearing after preview.
- Linked-file handling that distinguishes a project-link change from a physical file change.
- Final-reference detection before physical deletion.
- Support for deletion-only actions.
- Pre-change backups for physically removed files.
- Physical file deletion and workspace invalidation.
- `FilesDeleted` in the apply result.
- Structured preview errors when a safe physical baseline cannot be captured.
- A regression test confirming that a concurrent user edit causes `stale preview` and remains untouched.

### Key Design Decisions

1. **Separate preview from apply.** Multi-file semantic changes require explicit review and approval.
2. **Store the changed solution.** The reviewed Roslyn solution, not a rendered text patch, is applied.
3. **Track only affected paths.** Unrelated work should not invalidate the preview.
4. **Hash exact bytes.** Byte-level comparison detects formatting, encoding, and line-ending changes in addition to source edits.
5. **Represent absence explicitly.** New-file collision protection is part of the state model rather than encoded with a magic hash value.
6. **Reason about paths across the complete solution.** Roslyn documents are not guaranteed to correspond one-to-one with disk files.
7. **Back up before mutation.** Failed backup preparation cannot leave source files partially changed.
8. **Do not automatically roll back partial changes.** Automatic rollback can fail or overwrite concurrent work; snapshots and explicit recovery are safer.
9. **Self-heal only obvious truncation.** Missing or empty output strongly indicates persistence damage, while non-empty unexpected content may be user work.
10. **Build Fix All on the same apply pipeline.** Phase II should change how the combined action is calculated, not introduce a second disk-write implementation.

---

## Code Changes

### Active Branch

- [issue-110-apply-bulk-code-fix](https://github.com/rubinashaik2022/RoslynMcp/tree/issue-110-apply-bulk-code-fix)

### Meaningful Commits

- [`06733d5` - added preview code fix tool + apply code fix tool + focused tests](https://github.com/rubinashaik2022/RoslynMcp/commit/06733d5)
- [`e1ecdc6` - fix: protect code-fix apply from stale file changes](https://github.com/rubinashaik2022/RoslynMcp/commit/e1ecdc6)

The commits are currently ahead of the remote branch and should be pushed before submitting a draft PR.

### Primary Files

- `src/RoslynMcp/Tools/CodeFix/CodeFixHost.cs`
- `src/RoslynMcp/Tools/CodeFix/PreviewCodeFixTool.cs`
- `src/RoslynMcp/Tools/CodeFix/ApplyCodeFixTool.cs`
- `src/RoslynMcp/ApprovalStore.cs`
- `src/RoslynMcp/Program.cs`
- `src/RoslynMcp/RoslynMcp.csproj`
- `src/TestHarness/TestHarnessProgram.cs`
- `src/TestHarness/Tests/CodeFixTests.cs`
- `src/RoslynMcp/Tools/CodeFix/README.md` (architecture notes, currently local/untracked)

---

## Testing Strategy

### Implemented Integration Coverage

- End-to-end MCP preview and apply test for `RMCP001`.
- Confirms preview returns a token and a diff containing the `nint` replacement.
- Confirms apply writes the intended change and reports at least one written file.
- Regression test previews a fix, performs a simulated concurrent edit, applies the old token, and verifies:
  - The result is `stale preview`.
  - The concurrent edit remains byte-for-byte intact.

### Static and Build Validation Performed

- Ran `roslyn_get_diagnostics` against the server project: 0 compiler errors and 0 warnings.
- Ran `roslyn_get_diagnostics` against the test harness: 0 compiler errors and 0 warnings.
- Ran `roslyn_build_project` for `src/RoslynMcp/RoslynMcp.csproj`: succeeded.
- Ran `roslyn_build_project` for `src/TestHarness/TestHarness.csproj`: succeeded.
- The only reported build warnings were `NU1900` warnings because NuGet vulnerability metadata could not be reached from the restricted environment; these were unrelated to source correctness.
- Reviewed the semantic diff after edits and caught/restored MCP registration attributes that had been removed during an intermediate method replacement.

### Environment Limitation

The shell environment used during the final local pass did not expose `dotnet` or `pwsh` on `PATH`. As a result, the full executable harness and `scripts/Test-CodeStyle.ps1 -Fix` could not be launched directly from that shell. The RoslynMcp build tool performed SDK discovery independently and successfully built both projects.

### Remaining Focused Tests Before Phase II

- Added path remains absent and is created successfully.
- Added path appears after preview and is not overwritten.
- Removed file is backed up and deleted.
- Removed file changes or disappears after preview.
- Removing one linked document keeps the physical file.
- Removing the final linked document deletes the physical file.
- Mixed add/modify/remove action.
- Multi-file partial write with exact successful and failed paths.
- Non-empty incorrect post-write content.
- Hash-verified truncation recovery success and failure.
- Unsupported or failing `CodeAction` operations.

---

## Remaining Phase I Work

The targeted feature should stay intentionally smaller than a general transaction system. The next work should focus on making the existing preview/apply pipeline verifiable, easy to recover, and difficult to misuse.

### 1. Exact Post-Write Verification

The current truncation checks detect missing, empty, or extremely short files. They do not prove that a non-empty file contains the complete intended fix.

For each modified or added path, RoslynMcp should calculate:

```text
Original preview hash
Intended ChangedSolution hash
Actual disk hash after apply
```

The final state can then be classified:

| Actual disk state | Meaning |
|---|---|
| Matches intended hash | Successfully written |
| Matches original hash | Write did not occur |
| Missing or empty | Missing or truncated |
| Matches neither | Partial write, interference, or uncertain content |

For a removed file, absence means successful deletion; the original hash means deletion did not occur; and a different existing hash means the file changed unexpectedly.

Truncation self-healing should remain, but a recovered file should count as successful only after its hash matches the intended content. A non-empty unexpected file should not be overwritten automatically because it may contain concurrent user work.

### 2. Clean Partial-Write Results and Recovery Messages

When a multi-file operation fails after some files were already written, the result should identify exact paths instead of returning only an error or aggregate count.

The response should clearly separate:

- Files that reached the intended content
- Files that were successfully deleted
- Files that remained unchanged
- Files whose final content is uncertain
- The path where the failure occurred, when known

Example:

```json
{
  "error": "partial apply",
  "writtenFiles": ["A.cs", "B.cs"],
  "deletedFiles": ["OldHelper.cs"],
  "untouchedFiles": ["D.cs"],
  "uncertainFiles": ["C.cs"],
  "failedFile": "C.cs"
}
```

Recovery guidance should also be specific:

- Written file: apply its `pre` snapshot to roll back.
- Deleted file: apply its `pre` snapshot to recreate it.
- Untouched file: no recovery is needed.
- Uncertain file: inspect it, then use `pre` to roll back or `post` to restore the intended fix.
- Newly created file: rollback means deleting it because no pre-existing snapshot exists.

This work is required before Fix All because a project- or solution-wide fix can touch many files and must explain exactly what happened if one operation fails.

### 3. Bind Approval Tokens to Their Original Workspace

A token created from one project or workspace should not be applied through another `projectPath`.

The pending operation should store the canonical workspace identity resolved during preview. Apply should resolve its supplied path to the same canonical identity and reject a mismatch before stale checks, backups, or writes.

The identity should not be the raw input string because several inputs may legitimately resolve to the same workspace:

```text
/repo
/repo/App.csproj
/repo/src/Widget.cs
```

This protects against mixing a changed solution from workspace A with backup metadata, workspace mode, or invalidation calls from workspace B.

### 4. Harden CodeAction Calculation

The selected action can fail while Roslyn calculates its operations, even if the provider successfully advertised the action.

RoslynMcp should:

- Catch non-cancellation exceptions from `GetOperationsAsync`.
- Return a clean message containing the diagnostic ID, provider, action title, and failure reason.
- Allow `OperationCanceledException` to continue through the normal cancellation path.
- Replace the current `SingleOrDefault` assumption with a clear validation check.

For the first version, RoslynMcp should support exactly one `ApplyChangesOperation`. Zero, multiple, or custom operations should return a structured unsupported-action response rather than throwing or silently ignoring part of the provider's request.

### 5. Handle File-Access Races Cleanly

A file can disappear or become inaccessible between an existence check and the later hash read. Expected `IOException` and `UnauthorizedAccessException` cases during stale validation should return a clean stale/unavailable response instead of escaping as a generic tool failure.

### 6. Complete Focused Integration Coverage

The current tests cover a successful existing-file fix and rejection of a stale modified file. Additional focused tests should cover:

- Added path remains absent and is created.
- Added path appears after preview and is not overwritten.
- Removed file is backed up and deleted.
- Removed file changes or disappears after preview.
- Removing one linked document keeps the physical file.
- Removing the final linked document deletes it.
- Mixed add/modify/remove action.
- Multi-file failure reports exact successful and failed paths.
- Non-empty incorrect post-write content.
- Truncation recovery succeeds only when the final hash matches.
- Unsupported and failing code actions return clean messages.
- A token cannot be applied through another workspace.

### 7. Documentation, Validation, and Cleanup

- Update the main README tool table and public tool counts.
- Document result fields and recovery behavior.
- Include the code-fix architecture README.
- Run the full executable test harness.
- Run `scripts/Test-CodeStyle.ps1 -Fix`.
- Run Roslyn diagnostics and full builds.
- Remove `.DS_Store` files and add the pattern to `.gitignore`.

### Lower-Priority Future Work

The following improvements are useful but should not block the initial targeted feature:

- Three-state approval tokens (`Pending -> Applying -> Consumed`) for retrying verified zero-write failures.
- A shared `SolutionChangeApplier` used by code fix, rename, and signature-change tools.
- General-purpose injectable provider/test infrastructure.
- Automatic multi-file rollback.
- Support for executing custom or combining multiple `CodeActionOperation` types rather than rejecting them.
- Detailed per-file public state models beyond successful and failed paths.
- Perfect linked-file and multi-target-framework test coverage.
- Large-result pagination and caps, which primarily belong to Phase II Fix All.

---

## Phase II: Fix All

Fix All should reuse the exact same safe pipeline:

```text
FixAllProvider + FixAllContext
        -> combined CodeAction
        -> ChangedSolution
        -> unified diff
        -> affected physical states
        -> approval token
        -> stale validation
        -> backups
        -> apply and verify
```

Phase II adds:

- `FixAllProvider` discovery
- Document, project, and solution scopes
- Diagnostic aggregation
- Cancellation and timeouts
- Affected diagnostic, project, and file counts
- Changed-file and diff-size safeguards
- Fix All-specific integration tests

It should not add a separate file-writing path. Completing Phase I's exact verification and partial-result reporting first ensures that a large Fix All operation can explain exactly what reached disk and how to recover from failure.

---

## Pull Request

**Draft PR:** Not created yet. The active branch and commits are linked above and should be pushed before opening the draft PR.

Suggested PR summary:

> Adds targeted Roslyn code-fix preview and apply tools. The preview tool discovers matching `CodeFixProvider` actions for a diagnostic, returns action choices, calculates the selected `ChangedSolution`, and produces an approval token with a unified diff. The apply tool validates affected file states, creates local-history snapshots, applies the reviewed solution, handles final-reference file deletion, and rejects stale previews so concurrent edits are not overwritten. Fix All is intentionally deferred until exact post-write verification and partial multi-file reporting are complete.

**Status:** Active development; Phase I follow-up validation remains before draft review.

---

## Learnings and Reflections

### Technical Skills Gained

- Using Roslyn diagnostics and `CodeFixProvider` APIs outside an IDE.
- Understanding `CodeFixContext`, `CodeAction`, `CodeActionOperation`, and immutable `Solution` snapshots.
- Translating Roslyn document changes into safe physical file operations.
- Handling linked source files and multi-project document representations.
- Designing two-step MCP workflows for destructive actions.
- Using SHA-256 file-state checks to protect concurrent user edits.
- Integrating backup, recovery, and workspace invalidation behavior.
- Dogfooding RoslynMcp's own diagnostics, semantic reads, builds, and editing tools.

### Challenges Overcome

The hardest part of this contribution was understanding and defining the problem before writing the code. The issue initially sounded like a single feature—apply Roslyn code fixes—but it crossed diagnostic discovery, provider execution, immutable solutions, approval workflows, backups, physical file safety, partial failures, and future Fix All behavior. I had to separate what Roslyn already provided from what RoslynMcp still needed to own.

Defining the first-version scope required the most judgment. Several ideas were valuable—Fix All, retryable three-state tokens, a shared transaction-style apply engine, automatic rollback, exhaustive linked-file testing, and generalized provider infrastructure—but implementing all of them in Phase I would have made one feature unnecessarily large. I had to decide which work was required for a safe and useful targeted version, which work depended on earlier foundations, and which work should be documented as future improvement.

Understanding and planning for the full set of normal cases, failure cases, and edge cases was also difficult. A seemingly simple code fix can modify one file, create a file, delete a file, affect several projects, or operate on multiple Roslyn documents that share one physical path. The workspace can change between preview and apply; a new path can unexpectedly appear; a file intended for deletion can be edited; a multi-file write can fail partway through; a workspace write can leave an empty file; a provider can fail while calculating its action; and an approval token can be retried or submitted through the wrong workspace. Working through these cases was necessary to understand the design, even when some of them were ultimately classified as lower-priority future work rather than Phase I requirements.

The tricky judgment was not simply identifying every edge case, but deciding how each one should affect the implementation plan. Some cases represented immediate data-safety requirements, such as preventing stale previews from overwriting concurrent edits. Others required small defensive handling, while broader retry, rollback, and generalized operation support could be deferred. This case-by-case prioritization helped keep the first version safe without allowing the feature to grow indefinitely.

The final Phase I boundary prioritized the targeted preview/apply workflow, explicit approval, affected-file stale protection, backups, deletion handling, and focused integration coverage. Exact post-write verification, honest multi-file partial-result reporting, and small action-execution hardening were identified as the remaining completion work. Broader infrastructure and Fix All were deliberately deferred until the targeted pipeline could provide trustworthy physical results.

The linked-file and disk-state problems were still important technical challenges. A Roslyn `Document` is not always equivalent to one physical file, and a `ChangedSolution` describes intended in-memory state without proving what reached disk. Understanding those distinctions informed the design, but the larger challenge was deciding how much of that complexity had to be solved immediately and in what order.

### What I Would Do Differently

I would spend more time at the beginning writing a short problem statement, explicit Phase I completion criteria, and a separate future-work list. That would make it easier to judge proposals against the actual first-version goal and prevent useful long-term ideas from being mistaken for immediate requirements.

I would also define the physical-state model earlier—original presence and hash, intended presence and hash, and actual post-write state—because it clarifies stale validation, verification, partial reporting, token retry behavior, and Fix All. Most importantly, I would preserve the sequencing used by the final plan: complete the smallest safe targeted workflow first, verify it, and add generalized infrastructure only when the next phase demonstrates that it is necessary.

---

## Resources Used

- [RoslynMcp issue #110](https://github.com/MadQ/RoslynMcp/issues/110)
- [RoslynMcp development branch](https://github.com/rubinashaik2022/RoslynMcp/tree/issue-110-apply-bulk-code-fix)
- [Microsoft.CodeAnalysis.CodeFixes.CodeFixProvider documentation](https://learn.microsoft.com/dotnet/api/microsoft.codeanalysis.codefixes.codefixprovider)
- [Microsoft.CodeAnalysis.CodeActions.CodeAction documentation](https://learn.microsoft.com/dotnet/api/microsoft.codeanalysis.codeactions.codeaction)
- [Microsoft.CodeAnalysis.Solution documentation](https://learn.microsoft.com/dotnet/api/microsoft.codeanalysis.solution)
- [Model Context Protocol specification](https://modelcontextprotocol.io/)
- Existing RoslynMcp rename, signature-change, backup, and local-history implementations

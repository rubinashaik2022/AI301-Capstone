# Contribution 2: Roslyn code-fix preview and apply workflow

**Contribution Number:** 2  
**Student:** Rubina Shaik  
**Issue:** [MadQ/RoslynMcp #110](https://github.com/MadQ/RoslynMcp/issues/110)  
**Development Branch:** [issue-110-apply-bulk-code-fix](https://github.com/rubinashaik2022/RoslynMcp/tree/issue-110-apply-bulk-code-fix)  
**Project Status:** Phase I implementation complete; final end-to-end startup verification is blocked by an MCP server initialization hang  
**Tool Status:** The targeted code-fix workflow now has strict provider, file, workspace, encoding, approval, backup, concurrency, persistence, and partial-failure boundaries. Both projects build successfully and focused boundary tests are implemented.  
**Delivery Plan:** PR 1 will complete and submit Phase I of the tool. A separate PR 2 will implement Phase II Fix All support after the targeted preview/apply pipeline is fully validated.

> **Development note:** The Phase I safety scope is now deliberately narrow: only bundled RoslynMcp code-fix providers may run, and only existing writable, non-generated `.cs` files inside the resolved workspace may be modified. File creation, deletion, moves, linked or external files, project-system changes, references, additional documents, and analyzer-config changes are rejected. Fix All remains a separate Phase II contribution.

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
- Physical file modification, encoding preservation, atomic replacement, and verification
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

## Approval-Token Lifecycle

The current implementation stores tokens in memory for the lifetime of the server process.

```text
Preview succeeds
    -> Pending
    -> Applying
    -> Consumed
```

### Cases where the token is not consumed

The token remains pending when apply stops before claiming the operation:

- Approval value is invalid
- The preview is stale because an affected path changed
- Backup creation fails before source mutation
- The supplied workflow or workspace does not match the preview

The approval token is bound to both the code-fix workflow and the canonical workspace identity. A token from rename, signature change, or another workspace cannot be applied through `roslyn_apply_code_fix`.

### Cases where the token is removed without applying

Approval `n` atomically rejects and removes a pending token. If another request has already claimed it for application, rejection reports that the token is unavailable instead of falsely claiming that an in-progress apply was rejected.

### Current point of consumption

For approval `y` or `session`, apply performs:

1. Token, workflow, and workspace validation
2. Affected-file stale validation
3. Pre/post backup preparation using exact bytes
4. Atomic transition from `Pending` to `Applying`
5. A second validation inside the global physical-apply gate
6. Same-directory temporary-file staging
7. A final target-hash check immediately before atomic replacement
8. Exact post-write hash verification
9. Transition to `Consumed`

The atomic claim prevents two simultaneous requests from applying the same token. The process-wide apply gate also serializes physical solution application, which closes cross-token races between validation and persistence.

### Successful application

The token remains consumed after all intended changes are applied.

### Write failure before any file changes

Once claimed, the token cannot be reused. The response reports the verified state of each file so the caller can distinguish an unchanged file from a successful or uncertain write and generate a fresh preview when appropriate.

### Partial multi-file write

If some files are written before another file fails, the token remains consumed. Reusing the original token would be unsafe because the workspace no longer matches the original preview baseline.

The result reports each physical operation as written, deleted, unchanged, or uncertain and includes operation-specific recovery guidance. Backups provide exact pre-change and intended post-change snapshots.

### Phase I deletion policy

Deletion is rejected during preview and checked again during apply. Phase I cannot create, delete, or move `.cs` files, so it cannot silently break SDK globs or explicit MSBuild `<Compile Include>` entries.

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

Phase I deliberately supports modification only. A proposed action is rejected unless every physical effect is a modification to an existing writable, non-generated `.cs` file inside the workspace. This policy avoids project-system ambiguity and keeps the first release truthful about what it can safely persist.

### Why Preview and Apply Are Separate

A code fix may affect more than the source token where the diagnostic appears. The two-step workflow provides several safeguards:

- The caller reviews a unified diff before mutation.
- The exact intended bytes derived from the reviewed `ChangedSolution` are stored with the token.
- Apply does not recalculate the action or replay a text patch.
- Explicit approval is required before disk writes.
- Backups are created before source mutation.
- Stale affected files are rejected rather than overwritten.
- The original encoding and byte-order-mark choice are preserved.

### Why the Changed Solution Is Calculated Once

Roslyn `Solution` objects are immutable snapshots. Executing a `CodeAction` produces a proposed `ChangedSolution`, but it does not alter physical files. The diff is generated by comparing:

```text
BaseSolution -> ChangedSolution
```

The diff is for review. During preview, the changed source text is encoded into the exact bytes that should reach disk. Those intended bytes remain the persistence source of truth during backup, write, and verification. This avoids recalculating text during apply and prevents encoding or byte-order-mark drift.

---

## Architecture and Pipeline

### 1. Resolve the Source Document and Diagnostic

`roslyn_preview_code_fix` accepts a project path, file path, line, column, optional diagnostic ID, and optional action index. The tool resolves the project through `WorkspaceResolver`, locates the document, gathers compiler and analyzer diagnostics, and selects the diagnostic closest to the requested position.

When several diagnostics match the location, the tool returns the choices and asks the caller to provide a diagnostic ID rather than guessing.

### 2. Discover Matching Code Fixes

`CodeFixHost` owns an explicit allowlist of code-fix providers bundled with RoslynMcp. It does not load arbitrary providers from workspace analyzer packages. For each allowed provider it:

1. Checks whether `FixableDiagnosticIds` contains the diagnostic ID.
2. Creates a Roslyn `CodeFixContext`.
3. Calls `RegisterCodeFixesAsync`.
4. Captures the provider's registered `CodeAction` instances.
5. Records the provider name, title, equivalence key, diagnostic ID, and action index.

If several actions are available, preview returns them and requires an explicit `actionIndex`. This prevents RoslynMcp from silently choosing between fixes with different semantics. Restricting Phase I to bundled providers also prevents external analyzer packages from introducing unreviewed code execution or operations that bypass the intended safety model.

### 3. Calculate the Code Action in Memory

A `CodeAction` is a recipe, not a diff and not a file write. Preview calls `GetOperationsAsync` and extracts the `ChangedSolution` from an `ApplyChangesOperation`.

Phase I requires exactly one `ApplyChangesOperation`. Zero operations, multiple operations, or custom operation types return a structured unsupported-action result. Cancellation propagates normally, while non-cancellation provider failures are converted into clean tool errors containing the diagnostic, provider, and action context.

### 4. Generate the Unified Diff

`SolutionDiff.BuildAsync` compares the immutable base and changed solutions. The caller reviews this diff, but apply later uses the stored changed solution directly.

### 5. Enforce the Phase I Physical Boundary

Roslyn reports document- and project-level changes. Preview enumerates the complete solution change set and rejects:

- Added, removed, or moved documents
- Linked files and files outside the workspace
- Non-`.cs` files
- Generated files
- Read-only or otherwise non-writable files
- Project changes, metadata references, project references, analyzer references, additional documents, and analyzer-config documents

The same modification-only `.cs` boundary is checked again during apply. A malformed, stale, or wrong-workflow token therefore cannot bypass preview validation.

### 6. Capture Affected File States

For every allowed modified file, preview reads the original bytes and stores:

```csharp
internal sealed record PreviewFileState(
    string OriginalHash,
    byte[] IntendedBytes,
    string IntendedHash);
```

SHA-256 is calculated over exact disk bytes, so source, comments, whitespace, line endings, and encoding changes are all detected.

The changed Roslyn `SourceText` is encoded with the original file's encoding. Preview detects whether the original bytes contained that encoding's preamble, commonly called a byte-order mark or BOM, and preserves that exact choice. The intended bytes are stored once and reused for the post-backup, physical write, and verification.

### 7. Store the Pending Operation

`ApprovalStore` keeps the following data in memory:

- Base solution
- Unified diff
- Workflow kind and canonical workspace identity
- Session-approval key
- Original hashes and exact intended bytes

The returned token identifies the exact proposal reviewed by the caller.

### 8. Reject Stale Previews

Before backups and again inside the global physical-apply gate, apply rechecks every affected path.

The path must still exist, remain writable, remain an allowed `.cs` file, and match its preview SHA-256. If it changed, disappeared, or became unavailable, apply returns a clean stale or unavailable result without overwriting it.

### 9. Create Recovery Snapshots

Backups are created before source mutation:

| Physical operation | Pre snapshot | Post snapshot |
|---|---:|---:|
| Modify existing `.cs` file | Exact original bytes | Exact intended bytes |

Pre snapshots support rollback. Post snapshots restore or complete the exact approved output, including its original encoding and BOM choice.

### 10. Persist Exact Bytes Atomically

Code-fix persistence no longer delegates to `Workspace.ApplyChanges`. The physical applier writes the already-approved intended bytes to a uniquely named temporary file in the target directory, preserves the target's Unix mode when applicable, performs a final target-hash check, and atomically replaces the existing file.

The same-directory temporary file keeps the final rename on the same filesystem. This dramatically narrows the time-of-check/time-of-use gap and prevents a failed or interrupted write from truncating the original target.

### 11. Verify and Report the Final State

After replacement, the applier hashes the actual disk bytes and compares them with the stored intended hash. Every operation receives a verified state such as written, unchanged, or uncertain. Partial failures report exact per-file outcomes and recovery guidance rather than only an aggregate count.

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

The original add/remove implementation was subsequently narrowed for the first release. Creation, deletion, moves, and linked-file operations are now rejected because they can require `.csproj` edits and project-system reasoning that Phase I does not yet support.

### Phase I Hardening Pass

The final hardening work added:

- Atomic token claiming and workflow-specific token validation.
- Canonical workspace binding so a token cannot be applied through another project.
- Exact-one-`ApplyChangesOperation` enforcement.
- Per-file partial-write states and recovery messages.
- An explicit allowlist containing only bundled RoslynMcp providers.
- Rejection of arbitrary providers from external analyzer packages.
- Preview and apply enforcement for existing writable, non-generated in-workspace `.cs` files only.
- Rejection of project, reference, additional-document, analyzer-config, create, delete, move, linked, and external changes.
- Exact preservation of the source encoding and BOM choice.
- Storage of exact intended bytes in the approved physical plan.
- Use of the same intended bytes for post-backup, write, and hash verification.
- Cancellation propagation instead of converting cancellation into ordinary provider or I/O failures.
- Removal of inaccurate preview idempotency metadata.
- A process-wide physical-apply gate, a second stale check inside the gate, same-directory temporary-file staging, and atomic replacement.
- Accurate rejection reporting when another request is already applying the token.

### Key Design Decisions

1. **Separate preview from apply.** Multi-file semantic changes require explicit review and approval.
2. **Store the changed solution.** The reviewed Roslyn solution, not a rendered text patch, is applied.
3. **Track only affected paths.** Unrelated work should not invalidate the preview.
4. **Hash exact bytes.** Byte-level comparison detects formatting, encoding, and line-ending changes in addition to source edits.
5. **Represent absence explicitly.** New-file collision protection is part of the state model rather than encoded with a magic hash value.
6. **Reject ambiguous physical effects.** Linked files, create/delete/move operations, and project-system edits are deferred instead of approximated.
7. **Back up before mutation.** Failed backup preparation cannot leave source files partially changed.
8. **Do not automatically roll back partial changes.** Automatic rollback can fail or overwrite concurrent work; snapshots and explicit recovery are safer.
9. **Write exact bytes atomically.** Temporary-file staging protects the existing target, and exact hashes—not file size—prove the final result.
10. **Build Fix All on the same apply pipeline.** Phase II should change how the combined action is calculated, not introduce a second disk-write implementation.

---

## Code Changes

### Active Branch

- [issue-110-apply-bulk-code-fix](https://github.com/rubinashaik2022/RoslynMcp/tree/issue-110-apply-bulk-code-fix)

### Meaningful Commits

- [`06733d5` - added preview code fix tool + apply code fix tool + focused tests](https://github.com/rubinashaik2022/RoslynMcp/commit/06733d5)
- [`e1ecdc6` - fix: protect code-fix apply from stale file changes](https://github.com/rubinashaik2022/RoslynMcp/commit/e1ecdc6)
- `755b83f` - make token consumption atomic
- `301be22` - attach token to workspace
- `0c06fb6` - require exactly one `ApplyChangesOperation`
- `4a23c0f` - add partial-write handling and reporting
- `1ef7088` - harden the apply pipeline and restrict providers
- `b4d2e6e` - harden preview boundaries and preserve file encoding
- `7b603fd` - harden apply concurrency and file persistence

The local branch is ahead of the remote branch and should be pushed before submitting the first-round PR.

### Primary Files

- `src/RoslynMcp/Tools/CodeFix/CodeFixHost.cs`
- `src/RoslynMcp/Tools/CodeFix/PreviewCodeFixTool.cs`
- `src/RoslynMcp/Tools/CodeFix/ApplyCodeFixTool.cs`
- `src/RoslynMcp/ApprovalStore.cs`
- `src/RoslynMcp/PhysicalSolutionApply.cs`
- `src/RoslynMcp/Program.cs`
- `src/RoslynMcp/RoslynMcp.csproj`
- `src/TestHarness/TestHarnessProgram.cs`
- `src/TestHarness/Tests/CodeFixTests.cs`
- `src/RoslynMcp/Tools/CodeFix/README.md` (architecture notes, currently local/untracked)

---

## Testing Strategy

### Implemented Integration and Boundary Coverage

- End-to-end MCP preview and apply test for `RMCP001`.
- Confirms preview returns a token and a diff containing the `nint` replacement.
- Confirms apply writes the intended change and reports at least one written file.
- Regression test previews a fix, performs a simulated concurrent edit, applies the old token, and verifies:
  - The result is `stale preview`.
  - The concurrent edit remains byte-for-byte intact.
- Rejects create, delete, move, linked/external, generated, read-only, non-C#, project, reference, additional-document, and analyzer-config effects.
- Rejects wrong-workflow and wrong-workspace tokens.
- Rejects unsupported provider-operation shapes.
- Verifies modification-only policy again during apply.
- Covers UTF-8 and other supported source encodings with and without a BOM.
- Verifies the exact intended bytes are used for backup, persistence, and final hash comparison.
- Covers atomic approval races, partial-result reporting, cancellation propagation, and clean file-access failure reporting.

### Static and Build Validation Performed

- Ran `roslyn_get_diagnostics` against the server project: 0 compiler errors and 0 warnings.
- Ran `roslyn_get_diagnostics` against the test harness: 0 compiler errors and 0 warnings.
- Ran `roslyn_build_project` for `src/RoslynMcp/RoslynMcp.csproj`: succeeded.
- Ran `roslyn_build_project` for `src/TestHarness/TestHarness.csproj`: succeeded.
- The only reported build warnings were `NU1900` warnings because NuGet vulnerability metadata could not be reached from the restricted environment; these were unrelated to source correctness.
- Reviewed the semantic diff after edits and caught/restored MCP registration attributes that had been removed during an intermediate method replacement.

### End-to-End Harness Blocker

The compiled server and test harness both build successfully under `net10.0`. The full executable harness was then launched with an explicit .NET path, but the child MCP server did not answer the initial `initialize` request within 60 seconds, so no tool calls ran.

Direct reproduction showed that the server process remained alive, consumed approximately 1.4 GB of memory, and stalled before `ApplicationStarted` or the normal start log. A macOS process sample showed heavy assembly/reflection/JIT activity while the main thread waited. The machine had approximately 11 GB free, so storage pressure was worth addressing but did not explain this specific initialization hang.

This is currently the last verification blocker. It occurs before a code-fix request reaches preview or apply, so the focused code-fix pipeline has not yet been implicated. The next diagnostic step is to isolate MCP tool discovery/schema registration and determine which registered tool or startup component causes the initialization explosion.

---

## Remaining Phase I Work

The safety and persistence items previously listed as must-fix work are implemented. The remaining pre-PR work is finite:

1. Diagnose and fix the MCP initialization hang.
2. Run the complete executable harness after initialization works.
3. Run the repository style auditor when `pwsh` is available.
4. Review and add the untracked code-fix architecture README.
5. Remove untracked `.DS_Store` files.
6. Push the branch and open the first-round PR.

### Lower-Priority Future Work

The following improvements are useful but should not block the initial targeted feature:

- A shared `SolutionChangeApplier` used by code fix, rename, and signature-change tools.
- Controlled support for additional reviewed/bundled providers.
- Project-system-aware creation, deletion, moves, and `.csproj` edits.
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

**Draft PR:** Not created yet. The local branch contains the completed Phase I hardening commits and should be pushed after the startup blocker is resolved and the full harness passes.

Suggested PR summary:

> Adds targeted Roslyn code-fix preview and apply tools using bundled RoslynMcp providers. Preview discovers matching actions, returns choices, calculates the selected solution change, enforces an existing writable in-workspace `.cs`-only boundary, preserves encoding and BOM, and stores exact intended bytes with a workspace-bound approval token. Apply atomically claims the token, creates exact pre/post recovery snapshots, revalidates inside a global apply gate, stages same-directory temporary files, atomically replaces targets, verifies final hashes, and reports per-file outcomes. Creation, deletion, moves, linked/external files, project-system changes, and Fix All are intentionally deferred.

**Status:** Phase I implementation and focused boundary coverage are complete. Full end-to-end validation remains blocked by the server initialization hang described above.

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

Defining the first-version scope required the most judgment. Several ideas were valuable—Fix All, project-system edits, creation and deletion, a shared transaction-style apply engine, automatic rollback, exhaustive linked-file testing, and generalized external-provider infrastructure—but implementing all of them in Phase I would have made one feature unnecessarily large. I had to decide which work was required for a safe and useful targeted version, which work depended on earlier foundations, and which work should be documented as future improvement.

Understanding and planning for the full set of normal cases, failure cases, and edge cases was also difficult. A seemingly simple code fix can modify one file, create or delete files, affect several projects, or operate on multiple Roslyn documents that share one physical path. The final Phase I design handles the safe modification case and rejects the ambiguous cases explicitly. It also accounts for workspace changes between preview and apply, partial multi-file failures, encoding drift, provider failures, cancellation, concurrent token requests, and wrong-workspace approval attempts.

The tricky judgment was not simply identifying every edge case, but deciding how each one should affect the implementation plan. Some cases represented immediate data-safety requirements, such as preventing stale previews from overwriting concurrent edits. Others required small defensive handling, while broader retry, rollback, and generalized operation support could be deferred. This case-by-case prioritization helped keep the first version safe without allowing the feature to grow indefinitely.

The final Phase I boundary prioritizes the targeted preview/apply workflow, bundled providers, explicit approval, existing writable in-workspace `.cs` files, affected-file stale protection, exact-byte backups, encoding preservation, atomic replacement, final hash verification, honest partial-result reporting, and focused boundary coverage. Broader provider loading, project-system edits, add/remove support, and Fix All are deliberately deferred.

The linked-file and disk-state problems were still important technical challenges. A Roslyn `Document` is not always equivalent to one physical file, and a `ChangedSolution` describes intended in-memory state without proving what reached disk. Understanding those distinctions led to an explicit linked-file rejection boundary and a physical plan containing the exact bytes and hashes needed to prove the approved result.

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

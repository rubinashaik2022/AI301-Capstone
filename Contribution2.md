# Contribution [2]: feat: roslyn_apply_code_fix — bulk-apply Roslyn CodeFix providers without agent reasoning

**Contribution Number:** [2]  
**Student:** [Rubina Shaik]  
**Issue:** [https://github.com/MadQ/RoslynMcp/issues/110]  
**Status:** [Phase I]
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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

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

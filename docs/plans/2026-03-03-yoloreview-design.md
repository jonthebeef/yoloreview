# YOLO Review - Design Document

**Date:** 2026-03-03
**Status:** Approved

## Overview

A `/yoloreview` slash command for Claude Code that fully automates the PR lifecycle: creates a branch, raises a PR, runs isolated code review via a separate `claude -p` process, autonomously fixes issues, monitors CI, and merges when everything is green.

## Requirements

- **Starting state:** Local code changes already exist (staged, unstaged, or untracked)
- **Review tool:** Existing `/review` skill (code-review:code-review) via isolated `claude -p` subprocess
- **CI monitoring:** GitHub Actions via `gh` CLI (primary), adaptable to other CI systems
- **Fix mode:** Fully autonomous - no user intervention in the loop
- **Merge strategy:** Auto-detect from repo configuration
- **Naming:** Branch name and PR title auto-generated from diff analysis

## Architecture

**Type:** Single slash command (`~/.claude/commands/yoloreview.md`)
**Approach:** Single skill with bash orchestration - all git/gh operations via CLI, review isolation via `claude -p`

## Workflow Phases

### Phase 1: Setup

1. Check git status - confirm there are local changes
2. If already on a feature branch, use it. If on main/master, create a new branch (auto-named from diff analysis, e.g. `feat/add-user-validation`)
3. Stage all changes, create a commit with auto-generated message
4. Push the branch to origin

### Phase 2: PR Creation

1. Create a PR via `gh pr create` with auto-generated title and body
2. Post a comment on the PR: "YOLO Review initiated - automated review cycle starting"
3. Capture the PR number for subsequent steps

### Phase 3: Review Loop (repeats until LGTM)

1. Launch isolated reviewer: `claude -p "Run /review on PR #N in this repo"`
2. Capture review output to `/tmp/yoloreview-output-$PR_NUMBER.txt`
3. Post the review feedback as a PR comment
4. If LGTM -> exit loop, go to Phase 4
5. If issues found:
   - Parse the review feedback
   - Make autonomous code fixes
   - Commit and push with message like "fix: address review feedback (round N)"
   - Post a PR comment summarizing what was fixed
   - Loop back to step 1

**Max iterations:** 5 rounds

### Phase 4: Pipeline Watch (repeats until green)

1. Wait for CI checks: `gh pr checks <PR> --watch`
2. If all checks pass -> go to Phase 5
3. If checks fail:
   - Fetch failure logs: `gh run view <run-id> --log-failed`
   - Diagnose and fix the failures
   - Commit and push fixes
   - Post a PR comment: "CI fix: [description]"
   - Loop back to step 1

**Max iterations:** 3 rounds

### Phase 5: Merge & Cleanup

1. Merge the PR: `gh pr merge <PR> --auto --delete-branch` (uses repo's default merge strategy)
2. Post a final PR comment: "YOLO Review complete - merged successfully"
3. Switch back to the default branch locally
4. Pull latest

## Isolated Review Invocation

```bash
claude -p "Review PR #<PR_NUMBER> in this repo using /review. The repo is at $(pwd)." 2>&1
```

- Subprocess gets a clean context (no knowledge of implementation)
- **Key insight:** The `/review` command posts its findings directly as a PR comment via `gh pr comment`. The yoloreview loop reads PR comments to parse results rather than relying on `claude -p` stdout.
- LGTM detection: comment contains "No issues found" or zero numbered items
- Issues detection: comment contains "Found N issues:" with a numbered list

## PR Comment Formats

### Review Feedback

```markdown
## YOLO Review - Round N

### Review Feedback
[paste of review output]

### Status
[emoji] Issues found - fixing autonomously
```

### Fix Summary

```markdown
## YOLO Review - Round N (Fixes)

### Changes Made
- Fixed X in file Y
- Updated Z

### Status
[emoji] Re-reviewing...
```

### Final

```markdown
## YOLO Review - Complete

All reviews passed, CI green, PR merged successfully.
```

## Failure Protocol

When the review loop (5 rounds) or CI loop (3 attempts) is exhausted without reaching a mergeable state, the skill:

1. **Accepts failure explicitly** - does not try to work around the limits
2. **Posts a detailed failure comment on the PR** with:
   - Root cause analysis (why the issues couldn't be resolved)
   - Timeline of what was tried each round
   - Specific, actionable recommendations for the user
3. **Reports to the user** in terminal with a summary and next steps
4. **Leaves the PR open** for the user to continue manually

## Safety & Guardrails

- **Review loop max:** 5 rounds before stopping and reporting to user
- **CI fix loop max:** 3 rounds before stopping and reporting to user
- **No force pushes** - all changes are additive commits
- **Branch protection:** `gh pr merge --auto` queues merge for when requirements are met
- **Each PR comment timestamped** with round number for audit trail

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Skill vs Command | Command (`~/.claude/commands/`) | User-invocable slash command |
| Review isolation | `claude -p` subprocess | Clean reviewer context, no implementation bias |
| Fix automation | Fully autonomous | User wants hands-off loop |
| Loop limits | 5 review / 3 CI | Prevent infinite loops while allowing reasonable attempts |
| Merge strategy | Auto-detect | Respect repo configuration |
| Branch naming | Auto-generated | Faster workflow, no user prompts |

# YOLO Review Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a `/yoloreview` slash command that fully automates the PR lifecycle - branch, PR, isolated review loop, CI monitoring, and merge - with graceful failure reporting.

**Architecture:** Single command file at `~/.claude/commands/yoloreview.md` that orchestrates git/gh CLI operations and spawns isolated `claude -p` subprocesses for code review. The `/review` command posts findings directly as PR comments, so the loop reads PR comments to determine status.

**Tech Stack:** Claude Code commands, `gh` CLI, `claude -p`, git

---

### Task 1: Create the command file with frontmatter and phase structure

**Files:**
- Create: `~/.claude/commands/yoloreview.md`

**Step 1: Write the command file**

Create `~/.claude/commands/yoloreview.md` with the complete command. The full content follows:

```markdown
---
name: yoloreview
description: Fully automated PR lifecycle - creates branch, raises PR, runs isolated code review, fixes issues autonomously, monitors CI, and merges when green. Invoke when local changes are ready for review.
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, Task
---

# YOLO Review

Fully automated PR lifecycle. You have local changes ready to go. Take them all the way to merge with zero user intervention.

## CRITICAL: Read the review command output

The `/review` command (code-review:code-review) posts its findings **directly as a PR comment** via `gh pr comment`. After running the isolated review, read the latest PR comment to parse the results. Do NOT rely on stdout from `claude -p`.

## Phase 1: Setup

1. Run `git status` to confirm there are local changes (staged, unstaged, or untracked). If no changes exist, stop and tell the user.
2. Run `git branch --show-current` to get the current branch.
3. If on `main` or `master`:
   - Run `git diff --stat` and `git diff --cached --stat` to understand the changes.
   - Generate a short descriptive branch name from the changes (e.g. `feat/add-input-validation`, `fix/null-pointer-in-parser`).
   - Run `git checkout -b <branch-name>`.
4. If already on a feature branch, use it as-is.
5. Stage all changes: `git add -A`
6. Generate a commit message by analyzing the staged diff (`git diff --cached`).
7. Commit: `git commit -m "<generated message>"`
8. Push: `git push -u origin HEAD`

Post a brief status update to the user after this phase: "Branch `<name>` created and pushed."

## Phase 2: PR Creation

1. Detect the default branch: `gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'`
2. Generate a PR title (short, under 70 chars) and body (summary of changes) from the diff.
3. Create the PR:
   ```bash
   gh pr create --title "<title>" --body "<body>" --base <default-branch>
   ```
4. Capture the PR number from the output.
5. Post an initial comment:
   ```bash
   gh pr comment <PR> --body "## YOLO Review - Initiated

   Automated review cycle starting. Will review, fix issues, monitor CI, and merge when ready."
   ```

Post status to user: "PR #<number> created: <title>"

## Phase 3: Review Loop

**Max iterations: 5 rounds.**

For each round (N = 1, 2, 3, ...):

### Step 3a: Run isolated review

Launch an isolated Claude process to review the PR. This reviewer has NO context from the current session:

```bash
claude -p "Review PR #<PR_NUMBER> in this repo using /review. The repo is at $(pwd)." 2>&1
```

Wait for it to complete. The `/review` command posts its findings directly as a PR comment.

### Step 3b: Read the review results

Fetch the latest comment on the PR to get the review findings:

```bash
gh pr view <PR> --comments --json comments --jq '.comments[-1].body'
```

### Step 3c: Determine LGTM vs issues

Parse the latest review comment:
- **LGTM signals:** The comment contains "No issues found" or the comment body has zero numbered issues.
- **Issues signals:** The comment contains numbered issues (e.g. "Found N issues:" followed by a numbered list).

If LGTM: Post a comment confirming, then proceed to Phase 4.

If issues found and this is round N < 5:
- Post a comment:
  ```
  ## YOLO Review - Round N

  Issues found. Fixing autonomously...
  ```

### Step 3d: Fix the issues

Read the review comment carefully. For each issue:
1. Read the referenced file and lines.
2. Understand the issue and make the fix.
3. Be conservative - only fix what was called out, don't refactor.

After all fixes:
```bash
git add -A
git commit -m "fix: address review feedback (round N)"
git push
```

Post a comment summarizing what was fixed:
```
## YOLO Review - Round N (Fixes Applied)

### Changes Made
- [list each fix briefly]

Re-reviewing...
```

Loop back to Step 3a.

### Step 3e: Loop exhausted

If round 5 is reached and issues persist, **do NOT continue**. Proceed to the Failure Protocol (below).

## Phase 4: Pipeline Watch

**Max iterations: 3 rounds.**

### Step 4a: Wait for checks

```bash
gh pr checks <PR> --watch --fail-fast
```

If all checks pass: proceed to Phase 5.

### Step 4b: Diagnose failures

If checks fail:
1. Get the failed run ID:
   ```bash
   gh pr checks <PR> --json name,state,link --jq '.[] | select(.state == "FAILURE" or .state == "ERROR")'
   ```
2. Get the run ID from the check URL and fetch logs:
   ```bash
   gh run view <run-id> --log-failed
   ```
3. Analyze the failure logs and fix the issue.
4. Commit and push:
   ```bash
   git add -A
   git commit -m "ci: fix <brief description> (attempt N)"
   git push
   ```
5. Post a comment:
   ```
   ## YOLO Review - CI Fix (Attempt N)

   **Failure:** [what failed]
   **Fix:** [what was changed]

   Watching CI again...
   ```

Loop back to Step 4a.

### Step 4c: CI loop exhausted

If 3 CI fix attempts fail, **do NOT continue**. Proceed to the Failure Protocol (below).

## Phase 5: Merge & Cleanup

1. Merge the PR using the repo's default strategy:
   ```bash
   gh pr merge <PR> --auto --delete-branch
   ```
   If `--auto` is not available (no branch protection rules), merge directly:
   ```bash
   gh pr merge <PR> --delete-branch
   ```

2. Post a final comment:
   ```
   ## YOLO Review - Complete

   All reviews passed. CI green. PR merged and branch cleaned up.
   ```

3. Switch back locally:
   ```bash
   git checkout <default-branch>
   git pull
   ```

4. Report success to the user: "YOLO Review complete. PR #<number> merged successfully."

## Failure Protocol

When the review loop or CI loop is exhausted without success, the skill MUST:

1. **Accept failure explicitly.** Do not try to work around the limits.

2. **Post a detailed failure comment on the PR:**
   ```
   ## YOLO Review - Failed

   Automated review was unable to reach a mergeable state after [N review rounds / N CI fix attempts].

   ### Root Cause Analysis
   [Deep analysis of WHY the issues couldn't be resolved. Be specific:
   - Which issues kept recurring and why the fixes didn't stick
   - Whether the issues are architectural (can't be fixed with small patches)
   - Whether the review is flagging false positives
   - Whether CI failures are environmental vs code issues]

   ### What Was Tried
   [Chronological list of each round and what was attempted]

   ### Recommendations
   [Specific, actionable next steps for the user:
   - "The review keeps flagging X because Y. Consider redesigning the approach to Z."
   - "CI fails on test_foo because of a flaky dependency on service X. Consider mocking it."
   - "The reviewer flags this as a bug but it may be a false positive. Verify manually and consider adding a comment explaining the intent."
   - "Run /review manually to see if the remaining issues are real or false positives."]
   ```

3. **Report to the user in the terminal:**
   ```
   YOLO Review failed after N rounds. PR #<number> is still open.

   Root cause: [1-2 sentence summary]

   The PR has a detailed failure analysis in the comments. Recommended next steps:
   - [most important recommendation]
   - [second recommendation if applicable]
   ```

4. **Leave the PR open** - do NOT close it. The user may want to continue manually.

## Rules

- Never force push. All changes are additive commits.
- Never skip or silence pre-commit hooks.
- If `gh` commands fail due to auth issues, stop and tell the user.
- If the repo has no remote, stop and tell the user.
- Always post PR comments at each stage for audit trail.
- Be conservative with fixes - only change what the review calls out.
```

**Step 2: Verify the file was created correctly**

Run: `cat ~/.claude/commands/yoloreview.md | head -5`
Expected: The YAML frontmatter starting with `---`

**Step 3: Commit**

```bash
cd ~/yoloreview
git init
git add -A
git commit -m "feat: add yoloreview command - automated PR lifecycle with review loop"
```

---

### Task 2: Test the command is recognized by Claude Code

**Files:**
- Verify: `~/.claude/commands/yoloreview.md`

**Step 1: Verify command appears in Claude Code**

The user should be able to type `/yoloreview` in Claude Code and see it as an available command. This is a manual verification step - the user types `/yolo` and checks autocomplete suggests `yoloreview`.

**Step 2: Verify in a test repo**

To properly test, the user needs a repo with:
- A GitHub remote
- Some local uncommitted changes
- `gh` CLI authenticated

Suggest the user creates a test repo or uses an existing one to do a dry run.

---

### Task 3: Update the design doc with the review-output insight

**Files:**
- Modify: `~/yoloreview/docs/plans/2026-03-03-yoloreview-design.md`

**Step 1: Add the review output insight**

Add a note to the design doc under "Isolated Review Invocation":

> **Key insight:** The `/review` command posts its findings directly as a PR comment via `gh pr comment`. The yoloreview loop reads PR comments to parse results rather than relying on `claude -p` stdout.

**Step 2: Add the failure protocol to the design doc**

Add a new section "Failure Protocol" documenting the graceful failure behavior with root cause analysis and recommendations.

**Step 3: Commit**

```bash
cd ~/yoloreview
git add -A
git commit -m "docs: update design with review output insight and failure protocol"
```

---
name: yoloreview
description: Fully automated PR lifecycle - creates branch, raises PR, runs isolated code review, fixes issues autonomously, monitors CI, and merges when green. Invoke when local changes are ready for review.
user-invocable: true
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, Task
---

# YOLO Review

Fully automated PR lifecycle. You have local changes ready to go. Take them all the way to merge with zero user intervention.

## CRITICAL: How the isolated review works

The review is run by dispatching a **Task subagent** (type: `general-purpose`). The subagent has a fresh context with zero knowledge of the implementation. It reviews the PR diff via `gh`, posts its findings as a PR comment, and returns a summary.

**Do NOT use `claude -p`** — nested Claude Code sessions are not supported.

## Phase 1: Setup

1. Run `git status` to confirm there are local changes (staged, unstaged, or untracked). If no changes exist, stop and tell the user "No local changes found. Make some changes first, then run /yoloreview."
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

Post a brief status update to the user: "Phase 1 complete. Branch `<name>` created and pushed."

## Phase 2: PR Creation

1. Detect the default branch: `gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'`
2. Generate a PR title (short, under 70 chars) and body (summary of changes) from the diff against the default branch.
3. Create the PR:
   ```bash
   gh pr create --title "<title>" --body "$(cat <<'EOF'
   <generated body>

   ---
   🤖 PR created by YOLO Review
   EOF
   )"
   ```
4. Capture the PR number from the output. Extract it with: `gh pr view --json number --jq '.number'`
5. Post an initial comment:
   ```bash
   gh pr comment <PR> --body "## YOLO Review - Initiated

   Automated review cycle starting. Will review, fix issues, monitor CI, and merge when ready.

   **Limits:** 5 review rounds, 3 CI fix attempts."
   ```

Post status to user: "Phase 2 complete. PR #<number> created."

## Phase 3: Review Loop

**Max iterations: 5 rounds.**

Set ROUND=1. For each round:

### Step 3a: Run isolated review

Dispatch a **Task subagent** to review the PR. The subagent has a fresh context with no knowledge of the implementation. Use the Task tool with these parameters:

- **subagent_type:** `general-purpose`
- **description:** `Review PR #<PR_NUMBER>`
- **prompt:** (include ALL of the following in the prompt)

```
You are an independent code reviewer. Review PR #<PR_NUMBER> in the repo at <REPO_PATH>.

Do the following:

1. Run: gh pr diff <PR_NUMBER>
   Read the full diff carefully.

2. Run: gh pr view <PR_NUMBER> --json title,body --jq '.title + "\n" + .body'
   Understand what the PR is trying to do.

3. Check for any CLAUDE.md files in the repo root and in directories touched by the PR.

4. Review the diff for:
   a. Bugs - logic errors, off-by-one, null/undefined issues, race conditions
   b. CLAUDE.md compliance - if CLAUDE.md files exist, check the changes comply
   c. Security issues - injection, XSS, credential exposure
   d. Obvious mistakes a senior engineer would flag

5. For each issue found, score your confidence from 0-100:
   - 0: False positive, doesn't stand up to scrutiny
   - 25: Might be an issue, can't verify
   - 50: Real issue but minor or unlikely in practice
   - 75: Verified real issue, important, will impact functionality
   - 100: Certain, confirmed, will happen frequently

6. Filter out anything below 80 confidence.

7. Post your review as a PR comment using:
   gh pr comment <PR_NUMBER> --body "<your review>"

   Format:
   If issues found:
   ### Code review
   Found N issues:
   1. <description>
   2. <description>

   If no issues:
   ### Code review
   No issues found. Checked for bugs and CLAUDE.md compliance.

8. Return a one-line summary: either "LGTM" or "ISSUES: N issues found"
```

Wait for the subagent to complete and read its return value.

### Step 3b: Determine LGTM vs issues

Check the subagent's return value AND fetch the latest PR comment to confirm:

```bash
gh api repos/{owner}/{repo}/issues/<PR>/comments --jq '.[-1].body'
```

- **LGTM signals:** Subagent returned "LGTM" or latest comment contains "No issues found".
- **Issues signals:** Subagent returned "ISSUES:" or latest comment contains "Found N issues:".

**If LGTM:**
Post a comment:
```
## YOLO Review - Round ROUND - PASSED

No issues found. Proceeding to CI monitoring.
```
Then proceed to Phase 4.

**If issues found and ROUND < 5:**
Post a comment:
```
## YOLO Review - Round ROUND

Issues found. Fixing autonomously...
```
Continue to Step 3d.

**If ROUND == 5 and still issues:**
Proceed to the Failure Protocol.

### Step 3d: Fix the issues

Read the review comment carefully. For each numbered issue:
1. Read the referenced file and lines.
2. Understand the issue and make the minimal fix.
3. Be conservative - only fix what was called out, don't refactor surrounding code.

After all fixes:
```bash
git add -A
git commit -m "fix: address review feedback (round ROUND)"
git push
```

Post a comment summarizing what was fixed:
```
## YOLO Review - Round ROUND (Fixes Applied)

### Changes Made
- [list each fix briefly]

Re-reviewing...
```

Increment ROUND. Loop back to Step 3a.

## Phase 4: Pipeline Watch

**Max iterations: 3 attempts.**

Set CI_ATTEMPT=1.

### Step 4a: Wait for checks

```bash
gh pr checks <PR> --watch --fail-fast
```

If all checks pass: proceed to Phase 5.

### Step 4b: Diagnose failures

If checks fail and CI_ATTEMPT < 3:

1. Get the failed checks:
   ```bash
   gh pr checks <PR> --json name,state,link --jq '.[] | select(.state == "FAILURE" or .state == "ERROR")'
   ```
2. Extract the run ID from the check URL and fetch logs:
   ```bash
   gh run view <run-id> --log-failed
   ```
3. Analyze the failure logs and fix the issue.
4. Commit and push:
   ```bash
   git add -A
   git commit -m "ci: fix <brief description> (attempt CI_ATTEMPT)"
   git push
   ```
5. Post a comment:
   ```
   ## YOLO Review - CI Fix (Attempt CI_ATTEMPT)

   **Failure:** [what failed]
   **Fix:** [what was changed]

   Watching CI again...
   ```

Increment CI_ATTEMPT. Loop back to Step 4a.

### Step 4c: CI loop exhausted

If CI_ATTEMPT == 3 and still failing, proceed to the Failure Protocol.

## Phase 5: Merge & Cleanup

1. Merge the PR. Try squash first (cleanest history), fall back to merge commit:
   ```bash
   gh pr merge <PR> --squash --delete-branch
   ```
   If squash is not allowed by the repo, try:
   ```bash
   gh pr merge <PR> --merge --delete-branch
   ```

2. Post a final comment:
   ```
   ## YOLO Review - Complete

   All reviews passed. CI green. PR merged and branch cleaned up.
   ```

3. Switch back locally:
   ```bash
   DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
   git checkout $DEFAULT_BRANCH
   git pull
   ```

4. Report success to the user: "YOLO Review complete! PR #<number> merged successfully."

## Failure Protocol

When the review loop (5 rounds) or CI loop (3 attempts) is exhausted without reaching a mergeable state:

1. **Accept failure explicitly.** Do not try to work around the limits or continue.

2. **Analyze the failure deeply.** Look at:
   - All review comments posted during the loop
   - All commits made during the loop
   - The pattern of recurring issues
   - Whether issues are architectural vs superficial
   - Whether the reviewer is flagging false positives
   - Whether CI failures are environmental vs code-related

3. **Post a detailed failure comment on the PR:**

   ```
   ## YOLO Review - Failed

   Automated review was unable to reach a mergeable state after [N review rounds / N CI fix attempts].

   ### Root Cause Analysis

   [Deep analysis of WHY. Be specific:
   - Which specific issues kept recurring and why the fixes didn't resolve them
   - Whether the issues are architectural (can't be fixed with small patches)
   - Whether the reviewer is flagging potential false positives
   - Whether CI failures are environmental, flaky tests, or genuine code issues
   - Any dependency or configuration issues discovered]

   ### Timeline

   [Chronological summary:
   - Round 1: Found X issues, fixed Y
   - Round 2: Found X issues (N recurring), fixed Y
   - ...]

   ### Recommendations

   [Specific, actionable next steps:
   - "The review keeps flagging X because Y. Consider redesigning the approach to Z."
   - "CI fails on test_foo due to flaky dependency on service X. Consider mocking it."
   - "The reviewer flags issue X but it may be a false positive. Verify manually and add a comment explaining the intent."
   - "Run /review manually to see if the remaining issues are real."
   - "Consider breaking this PR into smaller, more focused changes."]
   ```

4. **Report to the user in the terminal:**

   ```
   YOLO Review failed after N rounds. PR #<number> is still open.

   Root cause: [1-2 sentence summary]

   The PR has a detailed failure analysis in the comments. Recommended next steps:
   - [most important recommendation]
   - [second recommendation if applicable]
   ```

5. **Leave the PR open.** Do NOT close or delete it. The user may want to continue manually.

## Rules

- Never force push. All changes are additive commits.
- Never skip or silence pre-commit hooks or `--no-verify`.
- If `gh` commands fail due to auth issues, stop and tell the user to run `gh auth login`.
- If the repo has no remote, stop and tell the user to add one first.
- Always post PR comments at each stage for a clear audit trail.
- Be conservative with fixes - only change what the review calls out.
- Do not refactor, improve, or clean up code that wasn't flagged.
- Each commit message should clearly indicate which round/attempt it belongs to.

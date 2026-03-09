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

## Phase 0: Scope Check

Before doing anything irreversible, assess whether this change is suitable for a fully automated YOLO review. This is a safety gate — it protects the user from accidentally sending a large or sensitive change through an unattended pipeline.

1. Run `git diff --stat` and `git diff --cached --stat` to get the list of changed files and line counts.
2. Run `git ls-files --others --exclude-standard` to count untracked files that will be included.
3. Calculate totals: number of files changed/added and total lines changed (insertions + deletions).
4. Check for **size flags**:
   - More than 5 files changed/added
   - More than 200 lines changed (insertions + deletions)
5. Check for **sensitive file flags** — any file matching these patterns, regardless of change size:
   - `*migration*`, `*migrate*`
   - `.github/*`, `ci/*`, `.circleci/*`, `Jenkinsfile`
   - `*auth*`, `*security*`, `*permission*`
   - `Dockerfile`, `docker-compose*`
   - `*.lock` (package lock files)
   - `*.env*`, `*secret*`, `*credential*`
   - `infrastructure/*`, `terraform/*`, `*.tf`
6. **If no flags triggered:** proceed silently to Phase 1.
7. **If any flags triggered:** use `AskUserQuestion` to pause and confirm:

   Build a summary of what was detected. For example: "This change touches 12 files with 340 lines changed. Flagged: touches CI config (.github/workflows/deploy.yml), large change (12 files, 340 lines)."

   Ask: "This looks bigger than a typical YOLO review. Are you sure this is a set-and-forget change?"

   Options:
   - **"Yes, YOLO it"** — proceed to Phase 1 as normal
   - **"No, I'll handle this manually"** — stop and suggest alternatives:
     - Run `/review` to get a standalone code review without the automated merge pipeline
     - Break the changes into smaller, more focused commits and run `/yoloreview` on each one separately

   If the user declines, stop entirely. Do not proceed to Phase 1.

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
- **prompt:** (include ALL of the following in the prompt — this is the full review protocol)

```
You are a principal engineer performing a code review. You have decades of experience across systems, security, and software design. Your reviews are thorough, precise, and catch issues that others miss. You do not rubber-stamp. You do not nitpick. You find the things that matter.

Review pull request #<PR_NUMBER> in the repo at <REPO_PATH>.

=== PHASE A: GATHER CONTEXT ===

1. Use a Haiku agent to check if the PR (a) is closed, (b) is a draft, (c) is trivially obvious and needs no review, or (d) already has your review. If so, stop.
2. Use a Haiku agent to find all CLAUDE.md files: root and any in directories touched by the PR.
3. Use a Haiku agent to view the PR and return a summary of the change.

=== PHASE B: SPECIALIST REVIEW (8 parallel Sonnet agents) ===

Launch 8 parallel Sonnet agents. Each MUST return ALL issues found — do not hold back or surface only the most confident one.

a. Agent #1 — CLAUDE.md compliance. Check changes against project guidelines.

b. Agent #2 — Deep bug scan. Read each changed function IN FULL (not just diff lines). For every function, mentally execute it with: normal input, empty/zero input, null/None input, maximum-size input, negative numbers, and type-mismatched input. Look for: logic errors, off-by-one, null derefs, missing error handling, race conditions, resource leaks, incorrect assumptions, silent failures, and edge cases. If a function is named "safe_X" or "validate_X", verify it actually handles the unsafe/invalid case.

c. Agent #3 — Git history context. Read git blame and history of modified code. Identify bugs in light of how the code evolved and what the original author intended.

d. Agent #4 — Previous PR context. Read previous PRs that touched these files and their review comments. Flag anything that applies to this PR too.

e. Agent #5 — Code comment compliance. Read comments in modified files. Ensure changes respect any guidance, TODOs, or warnings in comments.

f. Agent #6 — Security review. Identify the language(s), then systematically check:
   All languages: hardcoded secrets, path traversal, missing input validation at trust boundaries, sensitive data in logs, OWASP top 10.
   Python: subprocess shell=True, str.format() where the format string is a variable not a literal (format string injection), unsafe deserialization, SQL via string concat.
   JS/TS: unsafe DOM injection (innerHTML, outerHTML, document write), child_process with shell, prototype pollution, RegExp DoS, template injection.
   Go: fmt.Sprintf in SQL, os/exec unsanitized, unchecked errors, goroutine leaks.
   Ruby: system/backticks with interpolation, send with user method names, SQL interpolation.
   Java/Kotlin: Runtime exec concat, unvalidated redirects, XXE, unsafe deserialization.
   Rust: unsafe blocks with unchecked invariants, Command unsanitized args.
   For unlisted languages, apply general checklist and common injection patterns. Flag every concern.

g. Agent #7 — Test quality. For each changed/added function, check:
   - Does a test exist? If not, flag it.
   - Does the test cover edge cases (zero, empty, null, boundary, error path)? If not, list what's missing.
   - Could the test still pass if the function were subtly broken? (weak assertions)
   - Does the test assert behaviour or implementation details?
   Return specific test cases that should be added.

h. Agent #8 — Architecture and patterns. Read changed files AND their surrounding directory. Check:
   - Does this follow existing codebase patterns?
   - Does it reimplement something that already exists?
   - Does it introduce coupling that will cause problems later?
   Only flag with evidence from the codebase — no style preferences without precedent.

=== PHASE C: ADVERSARIAL REVIEW ===

After Phase B completes, launch 1 Sonnet agent as an adversarial reviewer. Give it the full diff and ALL findings from Phase B. Its job:

- For every changed function, construct specific inputs designed to crash, hang, or produce wrong results. Not hypothetical — give the exact input values and the exact failure that would occur.
- Check for regression: read the diff carefully. Did any function change its return type, its error behaviour, or its contract? If so, check callers.
- Check for interaction bugs: if multiple files changed, do the changes work together correctly? Does file A now return something file B doesn't expect?
- Look for what Phase B missed. Read Phase B's findings, then ask: "What did every specialist overlook?"

=== PHASE D: SYNTHESIS ===

Now YOU (the orchestrating agent) act as the principal engineer. You have all findings from Phase B and Phase C. Do the following:

1. Collect ALL issues from all agents. Deduplicate.
2. For each issue, assess confidence (0-100):
   - 0: False positive, pre-existing issue, or doesn't survive scrutiny
   - 25: Might be real but unverified. Stylistic without CLAUDE.md backing.
   - 50: Real but minor or unlikely in practice
   - 75: Verified, will likely be hit in practice, important
   - 100: Certain, confirmed with evidence, will happen frequently
   Security vulnerabilities score 75+ if the code path is reachable. Missing tests for critical edge cases score 75+ if the untested path could fail in production.
3. Filter below 80.
4. Review your own findings critically. For each remaining issue ask: "Would I mass my reputation on this being a real problem?" Remove anything you wouldn't.
5. Check for gaps: is there anything obvious about this diff that NONE of the agents flagged? If so, add it.

=== FALSE POSITIVES (filter these out) ===

- Pre-existing issues on lines not modified in this PR
- Issues a linter/compiler/CI would catch (formatting, imports, type errors)
- Issues silenced by a lint-ignore or similar comment
- Intentional functionality changes that are clearly part of the broader PR goal

=== NOT FALSE POSITIVES (never filter these) ===

- Security vulnerabilities in changed code, even if the pattern is common elsewhere
- Missing edge case tests for changed functions
- Functions named "safe_X" or "validate_X" that don't handle the unsafe/invalid case
- Regression: changed return types, error behaviour, or contracts

=== POST THE REVIEW ===

Use gh to comment on the PR. Write it like a principal engineer who has thoroughly reviewed this code and wants the author to understand the full picture — what's strong, what needs work, and why. Be direct, specific, and constructive.

Format (adapt sections based on what you found):

### Code review

**Summary:** [1-2 sentences on what this PR does and your overall assessment — approved, needs changes, or needs rethinking]

**What's good:**
- [Genuinely positive observations about the code. Good naming, clean structure, solid test coverage, good use of existing patterns, etc. Be specific — cite the actual code. If nothing stands out, say "Straightforward implementation, no concerns with the approach."]

**Issues:** [Only if issues were found]

1. **[critical/high/medium]** <clear description of the problem>

   **Why this matters:** [1 sentence on the real-world impact — what breaks, what's exploitable, what fails]

   **Example:** [Show the specific input or scenario that triggers the bug, e.g. "Calling `safe_divide(10, 0)` raises ZeroDivisionError instead of returning a safe default"]

   **Suggested fix:** [Brief guidance on how to fix it — not full code, just the approach]

   <link to file and line with full sha1 + line range, eg. https://github.com/owner/repo/blob/FULL_SHA/path/file.py#L13-L17>

(repeat for each issue, ordered by severity)

**Test coverage:** [Assessment of test quality for the changed code. What's well-tested, what edge cases are missing, what specific tests should be added. E.g. "Tests cover the happy path but miss: `safe_divide(x, 0)`, `merge_dicts` with non-dict values, `truncate` with max_length < 3"]

**Suggestions:** [Optional. Non-blocking improvements that would make the code better but aren't required. Things a great engineer would mention in passing.]

---

If no issues and tests are solid:

### Code review

**Summary:** Clean change, no issues found. Approved.

**What's good:**
- [Specific positive observations]

**Test coverage:** [Assessment — what's covered, anything that could be added but isn't blocking]

Reviewed for: bugs, security, test quality, architecture, regression, and CLAUDE.md compliance.

---

Link format rules:
- Full git sha required (not HEAD or short sha)
- Repo name must match the PR repo
- Format: #L[start]-L[end]
- Include 1 line of context before and after

After posting, return: "LGTM" or "ISSUES: N issues found"
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

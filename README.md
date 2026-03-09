# yoloreview

You've just finished a small change. Dinner's on the table, the kids need bathing, or it's time for the bedtime story. Type `/yoloreview` and leave your laptop. When you come back, your PR has been reviewed, fixed, passed CI, and merged.

That's it. That's the whole idea.

### When to use it - and when not to

This is for small, confident changes. A bug fix. A new util function. A config tweak. Something you've already looked at and know is solid.

Don't make this part of your everyday workflow. Don't use it on a massive migration, a complex refactor, or anything that touches half the codebase. If the change needs deep context, careful sequencing, or your full attention to review - that's not a yoloreview. That's real work.

Use it when you're confident it's not something that will ruin your day when you next open your laptop.

## What it does

1. **Checks the scope of your changes** — counts files, lines, and checks for sensitive file patterns. If the change looks too big or touches risky files (migrations, CI config, auth, infrastructure), it pauses and asks "Are you sure this is a set-and-forget change?" before proceeding. Small, clean changes sail through without a prompt.
2. Creates a branch and commits your local changes
3. Raises a PR with an auto-generated title and description
4. Spawns an **isolated** Claude Code process to run `/review` (the reviewer has zero knowledge of your implementation context)
5. Posts review feedback as PR comments
6. Autonomously fixes any issues found
7. Pushes fixes and re-reviews in a loop until the reviewer gives an LGTM
8. Watches the CI pipeline until it goes green, fixing failures along the way
9. Merges the PR and cleans up the branch

If it can't get to a merge within the limits (5 review rounds, 3 CI fix attempts), it accepts failure gracefully - posting a detailed root cause analysis with actionable recommendations on the PR and reporting back to you.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed and authenticated
- [GitHub CLI (`gh`)](https://cli.github.com/) installed and authenticated
- The [code-review](https://github.com/anthropics/claude-code-plugins) plugin installed (provides the `/review` command)
- A git repo with a GitHub remote

## Install

Copy the command file to your Claude Code commands directory:

```bash
# Create the commands directory if it doesn't exist
mkdir -p ~/.claude/commands

# Copy the command file
cp commands/yoloreview.md ~/.claude/commands/yoloreview.md
```

That's it. The `/yoloreview` command will be available in your next Claude Code session.

## Usage

1. Make your code changes in any repo
2. Open Claude Code in that repo
3. Run `/yoloreview`
4. Walk away

The command works whether you're on `main`/`master` (it'll create a branch) or already on a feature branch.

## How the review loop works

```
Local changes
    |
    v
Scope check (files, lines, sensitive patterns)
    |
    +-- Too big / sensitive --> "Are you sure?" prompt
    |       |                       |
    |       +-- "Yes, YOLO it" -----+
    |       |                       |
    |       +-- "No" --> Stop       |
    |                   (suggests /review or splitting)
    +-- Small & clean --------------+
    |
    v
Create branch + commit + push
    |
    v
Create PR
    |
    v
+-> Run /review (isolated claude -p process)
|       |
|       v
|   Parse PR comment for results
|       |
|       +-- LGTM --> Monitor CI --+--> Green --> Merge + cleanup
|       |                         |
|       |                         +--> Fail --> Fix + push (max 3x)
|       |                                   |
|       |                                   +--> Exhausted --> Failure report
|       |
|       +-- Issues found
|               |
|               v
|           Fix issues autonomously
|           Commit + push
|               |
+---------------+ (max 5 rounds)
        |
        +--> Exhausted --> Failure report
```

## Safety

- **Scope check before anything irreversible** — before creating branches, commits, or PRs, yoloreview assesses whether the change is suitable for a fully automated pipeline. If it detects a large change (>5 files or >200 lines) or sensitive files (migrations, CI config, auth, Dockerfiles, infrastructure, secrets), it pauses and asks you to confirm. You can always override, but it makes sure you've thought about it first. Small, confident changes proceed without interruption.
- Never force pushes - all changes are additive commits
- Never skips pre-commit hooks
- Posts PR comments at every stage for a clear audit trail
- Leaves the PR open on failure so you can continue manually
- Review loop capped at 5 rounds
- CI fix loop capped at 3 attempts
- Conservative fixes - only changes what the reviewer flags

## File structure

```
yoloreview/
  commands/
    yoloreview.md    # The command file (copy to ~/.claude/commands/)
  docs/
    plans/
      2026-03-03-yoloreview-design.md   # Design document
      2026-03-03-yoloreview-plan.md     # Implementation plan
  README.md
```

## License

MIT

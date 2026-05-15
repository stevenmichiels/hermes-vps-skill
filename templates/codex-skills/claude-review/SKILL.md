---
name: claude-review
description: Run Claude Code CLI as a read-only reviewer from Codex and save a Markdown review report. Use when the user asks Claude to review a diff, create claude-review.md, get a second opinion from Claude, compare Codex/Claude work, or use one agent to develop while the other reviews. Supports plan-aware reviews using .codex/plan/*.md and guarded handling of large or sensitive diffs.
---

# Claude Review

## Core Rule
Use Claude Code as a reviewer, not a co-editor. One agent develops in the worktree; the other produces a Markdown review. The reviewer must not stage, commit, or edit source files.

## Workflow
1. Make sure the implementation agent has produced a normal git diff.
2. Locate this skill's bundled helper at `scripts/codex-claude-review` relative to this `SKILL.md`.
3. Run the helper from the target repository root:
   ```bash
   /path/to/claude-review/scripts/codex-claude-review "Review the current diff as a strict senior engineer."
   ```
4. Read the generated Markdown report under `.codex/reviews/`.
5. Decide which findings to apply. Apply changes as the primary agent.
6. If the review report should be versioned, inspect it first and commit only the Markdown report in a separate commit.

## Output Path
Default output is timestamped:

```bash
.codex/reviews/claude-review-YYYYMMDDTHHMMSSZ.md
```

If that default location is not writable, for example because a sandbox mounts
`.codex/reviews` read-only, the helper falls back to:

```bash
${TMPDIR:-/tmp}/codex-claude-review-YYYYMMDDTHHMMSSZ.md
```

Set `CLAUDE_REVIEW_FALLBACK_DIR` to choose a different fallback directory.

Use `-o` for a stable path:

```bash
/path/to/claude-review/scripts/codex-claude-review \
  -o claude-review.md \
  "Check whether this implementation is overengineered."
```

## Context Included
The helper sends Claude:
- tracked changes from `git diff HEAD`
- untracked files, excluding previous `.codex/reviews/` reports
- up to three recently modified `.codex/plan/*.md` files

The plan does not need a fixed name. To omit plan context:

```bash
CLAUDE_REVIEW_INCLUDE_PLAN=0 \
  /path/to/claude-review/scripts/codex-claude-review
```

## Safety Controls
- Claude runs with `--print`, `--tools ""`, `--permission-mode plan`, and `--no-session-persistence`.
- The helper refuses likely secret-bearing paths by default, such as `.env`, `secrets/`, `config/credentials.json`, and private key filenames.
- The helper refuses diffs larger than `CLAUDE_REVIEW_MAX_DIFF_BYTES` (default `200000`) unless explicitly overridden.
- `CLAUDE_REVIEW_MAX_BUDGET_USD` is optional. Leave it unset for normal Claude Code subscription usage.

Useful overrides:

```bash
CLAUDE_REVIEW_MAX_DIFF_BYTES=400000 \
  /path/to/claude-review/scripts/codex-claude-review

CLAUDE_REVIEW_ALLOW_LARGE_DIFF=1 \
  /path/to/claude-review/scripts/codex-claude-review
```

Use `CLAUDE_REVIEW_ALLOW_SENSITIVE_DIFF=1` only after manually confirming the diff contains no secrets.

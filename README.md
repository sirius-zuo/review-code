# review-code

A agentic skill that reviews a pull request, a local branch/diff, or an explicit commit range or diff file for correctness bugs and `CLAUDE.md` compliance — by dispatching one subagent per review lens, either in parallel (default) or one at a time (`--sequential`). Dispatching one subagent at a time never requires multiple agents to run concurrently, so it still works against models (including local/offline LLMs) that can't run multiple agents in parallel — only true parallel dispatch needs that capability, and `--sequential` opts out of it.

## Why this exists

Claude Code ships an official bundled `/code-review` skill that dispatches several parallel subagents (and can offload to a cloud multi-agent `ultra` mode, or a richer max-mode 10-angle taxonomy via one subagent per angle — see `docs/claude/code-review-claude.md`). This skill borrows from that design — one subagent per review lens — with two differences that matter for portability and report quality:

- **`--sequential` is always available**, dispatching one lens-subagent at a time instead of in parallel — useful for platforms that support subagent dispatch but not multiple agents running concurrently.
- **Scoring stays centralized**, in one fresh-eyes pass over every lens's combined findings (Step 4) — never delegated to the lens-subagents themselves, so the same underlying issue flagged by two different lenses doesn't become two separate rows in the report.

It's named `review-code` rather than `code-review` specifically so it doesn't shadow the official bundled `/code-review` skill — both can be installed side by side.

## What it checks

**Default mode** dispatches 5 lenses (4 outside GitHub PR mode), one subagent each:

1. CLAUDE.md compliance
2. Shallow bug scan (diff-only, no extra context)
3. Git blame / history context
4. Prior PR review comments (GitHub PR mode only)
5. Code-comment compliance

**`--full` mode** additionally dispatches 7 more lenses, covering both deeper bug classes and cleanup-style findings:

6. Removed-behavior auditor — invariants dropped along with deleted/replaced code
7. Cross-file tracer — call-site breakage from a changed function (the one lens that reads beyond the diff hunk)
8. Language-pitfall specialist — classic footguns for the diff's language/framework
9. Reuse — new code reimplementing an existing helper
10. Simplification — unnecessary complexity the diff adds
11. Efficiency — wasted work the diff introduces
12. Altitude — fixes implemented as a bandaid instead of at the right depth

Lenses 9–12 overlap with the separate `simplify` skill (which also auto-fixes what it finds) — review-code reports them alongside bugs in one unified review instead of requiring a second pass; it never auto-fixes them itself.

Each finding gets a confidence score (0–100, only ≥80 survives) and an independent severity (Critical / Major / Minor) — see `SKILL.md` for the full rubric. Findings are ordered by severity then confidence, with correctness-bug lenses (1–8) breaking ties ahead of cleanup lenses (9–12).

## Usage

```
/review-code [--full] [--sequential] [--fix] [--focus "<area>"] [PR-number|URL|A..B|diff-file]
```

**Input modes** (auto-detected from the argument, in priority order):

| Mode | Trigger | Diff source |
|---|---|---|
| GitHub PR | A PR number/URL, or no argument with a PR open for the current branch | `gh pr diff` |
| Explicit | A commit range (`A..B`) or a path to a diff/patch file | that range or file |
| Local diff | Default — no argument, no PR, or `gh` unavailable | `git diff` against the detected base branch, including uncommitted changes |

**Flags:**

- `--full` — dispatch all 12 lenses instead of the default 5.
- `--sequential` — dispatch lens-subagents one at a time instead of the default parallel dispatch. Available with or without `--full`.
- `--focus "<area>"` — bias every dispatched lens toward an area (e.g. `"security"`, `"performance"`) without skipping any of them.
- `--fix` — after reporting, apply the surviving findings directly to the working tree. Anything ambiguous is reported as "not auto-fixed" instead of guessed at.

## Output

A markdown table (`# | Severity | Location | Impact | Description`) is printed to the console and written to `docs/review-code/<timestamp>-<slug>.md` **and** `docs/review-code/<timestamp>-<slug>.html` in the reviewed repo, where `<slug>` is `pr-<number>`, the sanitized commit range/diff filename, or the current branch name. The markdown file is the source of truth; the HTML file is a styled rendering of the same content for human reading. The skill never posts anywhere (no `gh pr comment`) — output is always local.

## Design notes

See `SKILL.md` for the full step-by-step pipeline. Key departures from the official `/code-review`'s subagent design:

- **`--sequential` is always available**, not just a fallback for low-capability platforms — one lens-subagent dispatched at a time still satisfies "no parallel multi-agent execution required."
- **Three input modes** instead of GitHub-PR-only, so it's useful before a PR even exists.
- **No posting.** The original posted a GitHub PR comment; this one only ever prints/writes locally.
- **Confidence + severity as separate axes**, scored centrally in one pass over every lens's combined findings — never delegated per-lens — so one issue flagged by two lenses doesn't become two rows in the report.
- **Cleanup lenses (9–12) report only, never auto-fix** — that overlap with `simplify` is intentional; `simplify` is the tool that applies fixes.

## Installation

This directory is a self-contained skill (`SKILL.md` at its root). To make it available across all your projects, symlink it into your personal skills directory:

```bash
ln -s /Users/jinzuo/projects/skills/review-code ~/.agents/skills/review-code
```

(Claude Code reads `~/.claude/skills/`; Codex, Copilot CLI, and Gemini CLI read the cross-runtime `~/.agents/skills/` alias.)

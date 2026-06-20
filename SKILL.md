---
name: review-code
description: Use when asked to review a pull request, code review the current branch or diff, or review a specific commit range or diff file, for correctness bugs and CLAUDE.md compliance.
argument-hint: '[--full] [--sequential] [--fix] [--focus "<area>"] [PR-number|URL|A..B|diff-file]'
allowed-tools: Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(git:*), Bash(mkdir:*), Bash(date:*), Agent, Edit, Write
---

# Code Review

## Overview

Reviews a pull request, a local branch/diff, or an explicit commit range / diff file for correctness bugs and CLAUDE.md compliance by dispatching one subagent per review lens (in parallel or sequentially via `--sequential`). Works against any model that supports subagent dispatch — `--sequential` opts out of parallel dispatch if the platform doesn't support concurrent agents.

Follow these steps precisely, in order. Make a todo list first, with one item per step below.

## Step 0: Parse arguments and detect input mode

Parse `$ARGUMENTS` for these optional flags, in any order, and strip them from the string before continuing:

- `--full` — dispatch all 12 lenses in Step 3 instead of the default 5. See Step 3 for the full roster.
- `--sequential` — dispatch Step 3's lens-subagents one at a time (wait for each to fully return before starting the next) instead of the default parallel dispatch. Applies identically whether or not `--full` is also given — both modes dispatch subagents, only the lens-set size differs between them.
- `--fix` — apply the surviving findings (after Step 5) to the working tree instead of only reporting them.
- `--focus "<area>"` — bias the analysis passes in Step 3 toward this area (e.g. `"security"`, `"performance"`, `"accessibility"`). Every dispatched lens still runs regardless — focus changes emphasis and depth, not which lenses execute.

Whatever remains after stripping flags is the target argument. Use it to detect the mode, in this priority order:

1. **GitHub PR mode** — the remaining argument is a PR number/URL, or empty but `gh pr view --json number -q .number` succeeds for the current branch. Diff source: `gh pr diff`.
2. **Explicit mode** — the remaining argument is a commit range (`A..B`) or a path to an existing diff/patch file. Diff source: that range or file directly.
3. **Local diff mode** (default — including when `gh` is unavailable/unauthenticated and no explicit argument was given) — diff the current branch/working tree against its detected base branch (`git merge-base`), including uncommitted changes.

State which mode was detected, whether `--full` and/or `--sequential` were given, and the focus area if one was given, before continuing.

## Step 0.5: Eligibility check

Check directly, without dispatching any agent:

- **GitHub PR mode only:** check `gh pr view --json state,isDraft`. Stop if state is `CLOSED` (abandoned, not merged) or `isDraft` is true, or if the PR already has a code review comment from you from earlier (check existing PR comments). A `MERGED` state is eligible — do not stop for it.
- **All modes:** is the diff empty, or is it obviously trivial (e.g. pure formatting, lockfile-only, generated-file-only)? If so, stop and say why.

## Step 1: Find relevant CLAUDE.md files

List the file paths (not contents) of any relevant CLAUDE.md files: the root CLAUDE.md (if one exists), plus any CLAUDE.md files in directories whose files were modified in the diff.

## Step 2: Summarize the change

Read the diff and write a short summary of what it changes, for reference in later steps.

## Step 3: Dispatch lens-subagents

Before dispatching, assemble the **scope block** every lens-subagent will receive verbatim:

- The diff command/target detected in Step 0 (mode, and how to reproduce the diff: `gh pr diff`, the explicit range/file, or the local-diff command).
- The CLAUDE.md file paths found in Step 1 (paths only — each subagent reads the contents itself).
- The change summary written in Step 2.
- The `--focus` area from Step 0, or "none".

Dispatch one subagent per applicable lens from the roster below. Default mode dispatches lenses 1–5 (4 outside GitHub PR mode, since lens 4 is PR-mode-only). `--full` mode additionally dispatches lenses 6–12.

- **Default (parallel):** dispatch all applicable lens-subagents at once, capped at 6 concurrent (batch the rest).
- **`--sequential`:** dispatch one lens-subagent at a time, waiting for it to fully return before starting the next. Lens instructions are identical either way — only dispatch timing changes.

Each subagent's task: run the diff command from the scope block, review through its assigned lens only, and return a findings list tagged with the lens name, each with a file:line citation. If a `--focus` area was given, the subagent should weight issues matching that area higher and look harder for it specifically — but still complete its full lens; it must not skip its assigned lens just because it seems unrelated to the focus. Collect every lens's findings into one running list before continuing to Step 4.

**Dig deeper when a finding is uncertain but would matter if true.** Each lens-subagent should, when something looks high-impact but can't be confirmed from the diff alone, follow the lead before finalizing it: open the function being called, check its other call sites, or read the type/schema it touches. Reserve this extra reading for findings that would otherwise stay uncertain and matter — not for every minor finding.

### Lenses 1–5 (default and `--full`)

1. **CLAUDE.md compliance** — audit the changes against the CLAUDE.md files from the scope block. CLAUDE.md is guidance for writing code, so not all instructions apply during review.
2. **Shallow bug scan** — read only the file changes in the diff (avoid extra context beyond the changes) and scan for obvious large bugs. Avoid small issues and nitpicks; ignore likely false positives.
3. **Git blame / history context** — read the git blame and history of the modified code, and identify any bugs visible only in light of that history.
4. **Prior PR review comments** — **GitHub PR mode only.** Read previous pull requests that touched these files, and check for comments on those PRs that may also apply here. In other modes, skip this lens entirely — do not dispatch it — and note "skipped — no PR host in this mode."
5. **Code-comment compliance** — read code comments in the modified files, and check the changes comply with any guidance in those comments.

### Lenses 6–12 (`--full` mode only)

6. **Removed-behavior auditor** — for every line the diff deletes or replaces, name the invariant or behavior it enforced, then check whether the new code re-establishes it. Flag a candidate if you can't find where it's re-established: a removed guard, a dropped error path, a narrowed validation, a deleted test that covered a real case.
7. **Cross-file tracer** — for each function the diff changes, find its callers and callees (grep for the symbol) and check whether the change breaks any call site: a new precondition, a changed return shape, a new exception, a timing/ordering dependency. Also check whether a parallel change elsewhere in the same diff makes a call unsafe. (This is the one lens allowed to read beyond the diff hunk into caller/callee files — lens 2 stays diff-only by design.)
8. **Language-pitfall specialist** — scan for classic pitfalls of the diff's language/framework (JS falsy-zero/`==` coercion/closure capture; Python mutable defaults/late-binding closures; Go nil-map writes/range-var capture; SQL injection; timezone/DST drift; float equality). Flag any instance the diff introduces.
9. **Reuse** — flag new code that reimplements something the codebase already has; grep shared/utility modules and files adjacent to the change, and name the existing helper to call instead.
10. **Simplification** — flag unnecessary complexity the diff adds: redundant or derivable state, copy-paste with slight variation, deep nesting, dead code left behind. Name the simpler form that does the same job.
11. **Efficiency** — flag wasted work the diff introduces: redundant computation or repeated I/O, independent operations run sequentially, blocking work added to hot paths, or closures that keep an enclosing scope alive longer than needed. Name the cheaper alternative.
12. **Altitude** — check each change is implemented at the right depth, not as a fragile bandaid; flag special cases layered on shared infrastructure as a sign the fix isn't deep enough, and suggest generalizing the underlying mechanism instead.

**Scoping rule for lenses 9–12:** only flag complexity/waste/reuse-misses the diff itself introduces — never pre-existing complexity elsewhere in the file.

Examples of false positives to exclude from any lens:
- Pre-existing issues
- Something that looks like a bug but is not actually a bug
- Pedantic nitpicks that a senior engineer wouldn't call out
- Issues a linter, typechecker, or compiler would catch (missing/incorrect imports, type errors, broken tests, formatting). Assume CI runs these separately — do not build or typecheck yourself.
- General code-quality issues (test coverage, security, docs) unless explicitly required by CLAUDE.md — lenses 9–12 (Reuse, Simplification, Efficiency, Altitude) are an explicit exception to this line; they are in scope by design whenever they're dispatched.
- Issues called out in CLAUDE.md but explicitly silenced in the code (e.g. a lint-ignore comment)
- Changes in functionality that are likely intentional or directly related to the broader change
- Real issues, but on lines the change did not modify

## Step 4: Score and classify each issue

Re-read the merged issue list from Step 3 with fresh eyes. For each issue, assign two independent things:

**A confidence score, 0-100, applied verbatim:**

- **0** — Not confident at all. False positive that doesn't survive light scrutiny, or a pre-existing issue.
- **25** — Somewhat confident. Might be real, might be a false positive; not verified. If stylistic, not explicitly called out in the relevant CLAUDE.md.
- **50** — Moderately confident. Verified as a real issue, but may be a nitpick or rare in practice; not very important relative to the rest of the change.
- **75** — Highly confident. Double-checked; very likely to be hit in practice. The existing approach is insufficient, or it is directly called out in the relevant CLAUDE.md.
- **100** — Absolutely certain. Double-checked and confirmed real, will happen frequently, evidence directly confirms it.

**A severity, independent of confidence:**

- **Critical** — breaks functionality, a security or data-loss risk, or causes incorrect behavior on a common path.
- **Major** — a real bug, but on a less common path or with a workaround.
- **Minor** — real but low-impact (a rare edge case, or a cosmetic correctness issue).

Confidence and severity don't move together: a Critical issue can carry low confidence (e.g. a suspected race condition you can't fully verify), and a Minor issue can be 100% certain.

For issues flagged via a CLAUDE.md, double check the CLAUDE.md actually calls out that issue specifically before scoring it high.

For findings from lenses 9–12 (Reuse, Simplification, Efficiency, Altitude), frame the eventual Impact description (Step 7) as the concrete cost — what's duplicated, wasted, or harder to maintain — rather than a crash or failure scenario; these lenses don't have a crash to describe.

## Step 5: Filter

Split the scored issue list in two: those scored 80 or above on confidence (regardless of severity) become the findings reported in Step 7's main table. Everything below 80 is not discarded — keep it for the appendix in Step 7.

## Step 6: Re-check eligibility

Repeat Step 0.5's eligibility check, in case anything changed while reviewing (e.g. the PR was closed, or someone else already posted a review, in the meantime). If no longer eligible, stop.

## Step 7: Report the result

Report as a markdown table, written to the console and to two files — never posted anywhere (no `gh pr comment`, no other API calls). The markdown file is the source of truth (for future LLM consumption); the HTML file is a presentation-only rendering of the exact same content (for human reading).

**Determine the output file path:**

1. Build a target slug: GitHub PR mode → `pr-<number>`; explicit mode → the commit range with `..` and `/` replaced by `-`, or the diff file's basename without extension; local diff mode → the current branch name with `/` replaced by `-`.
2. Build a timestamp: `date +%Y-%m-%d-%H%M%S`.
3. File path stem: `docs/review-code/<timestamp>-<slug>` (relative to the repo root being reviewed). Create the `docs/review-code/` directory if it doesn't exist. Write the markdown report to `<stem>.md` and the HTML report to `<stem>.html`.

**Citation format** for the Location column depends on mode:
- **GitHub PR mode:** full-SHA GitHub permalink, e.g. `[path:L40-L44](https://github.com/owner/repo/blob/<full-sha>/path#L40-L44)`. Requires the actual full git SHA (not a `$(...)` command substitution — resolve it directly, since the output is rendered as-is).
- **Local diff / explicit mode:** `path:line` relative to the repo root — no hosted URL needed.

**Ordering:** sort findings by severity (Critical, then Major, then Minor), then by confidence descending within each severity. When two findings tie on both severity and confidence, list correctness-bug findings (lenses 1–8) before cleanup findings (lenses 9–12).

**Table format**, if issues were found (example with 3):

```markdown
### Code review

Mode: <GitHub PR / local diff / explicit> — Target: <PR number, branch, or range> — Focus: <area, or "none">

Found 3 issues:

| # | Severity | Location | Impact | Description |
|---|----------|----------|--------|--------------|
| 1 | Critical | [path:L40-L44](<citation>) | <one-line consequence if this is real> | <brief description> (CLAUDE.md says "<...>") |
| 2 | Major | [path:L10-L13](<citation>) | <one-line consequence> | <brief description> |
| 3 | Minor | path:L88 | <one-line consequence> | <brief description> (bug due to <code snippet>) |
```

Or, if no issues remain after filtering:

```markdown
### Code review

Mode: <GitHub PR / local diff / explicit> — Target: <PR number, branch, or range> — Focus: <area, or "none">

No issues found. Checked for bugs and CLAUDE.md compliance.
```

**Appendix — filtered candidates.** If Step 5 set aside any sub-80 issues, append this section after the main table (or after the "No issues found" line). Omit the section entirely if nothing was filtered out.

```markdown
<details>
<summary>Appendix: candidates below the confidence threshold (not reported above)</summary>

| # | Confidence | Severity | Location | Rationale |
|---|------------|----------|----------|-----------|
| A | 50 | Minor | path:L40-L44 | <why it's real but didn't clear the bar> |
| B | 25 | Minor | path:L88 | <why it's real but didn't clear the bar> |

</details>
```

Use letters (A, B, C...) instead of numbers for appendix rows, so they're never confused with the numbered main findings.

**HTML rendering.** Render the exact same content (main table or "no issues found" line, plus appendix if non-empty) into this self-contained template — replace every `{PLACEHOLDER}`, drop the `{APPENDIX}` block entirely if there is no appendix:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Code Review — {TARGET}</title>
<style>
  body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif; background:#f5f5f7; color:#1d1d1f; margin:0; padding:2.5rem 3rem; line-height:1.6; }
  h1 { font-size:1.4rem; margin-bottom:0.25rem; }
  .meta { color:#6e6e73; font-size:0.875rem; margin-bottom:1.5rem; }
  table { width:100%; border-collapse:collapse; background:#fff; border-radius:8px; overflow:hidden; box-shadow:0 1px 3px rgba(0,0,0,0.08); margin-bottom:2rem; }
  th, td { padding:0.65rem 0.9rem; text-align:left; border-bottom:1px solid #e5e5ea; font-size:0.9rem; vertical-align:top; }
  th { background:#fafafa; font-weight:600; color:#3a3a3c; }
  tr:last-child td { border-bottom:none; }
  .sev-critical { color:#d70015; font-weight:600; }
  .sev-major { color:#bf5700; font-weight:600; }
  .sev-minor { color:#6e6e73; font-weight:600; }
  code { background:#f2f2f7; padding:0.1rem 0.35rem; border-radius:4px; font-size:0.85em; }
  details { margin-top:1rem; }
  summary { cursor:pointer; font-weight:600; color:#3a3a3c; }
</style>
</head>
<body>
  <h1>Code Review</h1>
  <div class="meta">Mode: {MODE} — Target: {TARGET} — Focus: {FOCUS}</div>
  {FINDINGS_TABLE_OR_NO_ISSUES_LINE}
  {APPENDIX}
</body>
</html>
```

For each finding row, set the `<td>` for Severity to `<span class="sev-critical">Critical</span>` (or `sev-major`/`sev-minor`) instead of plain text. Render the appendix (if present) as a `<details><summary>Appendix: candidates below the confidence threshold</summary><table>...</table></details>` block using the same row data as the markdown appendix table.

Write the markdown content (main table or "no issues found" line, plus the appendix if non-empty) to `<stem>.md`, write the HTML rendering of the same content to `<stem>.html`, then print the markdown content to the console. Keep it brief, no emojis.

## Step 8: Apply fixes (only if `--fix` was passed)

Skip this step entirely if `--fix` was not given — issues are reported only, never modified.

If `--fix` was given: after printing the review, fix each issue from Step 5 directly in the working tree, one at a time, using Edit. After all fixes, print a short summary: file changed and a one-line description, per issue. If the right fix for an issue is ambiguous or requires a design decision, don't guess — report it as "not auto-fixed: <reason>" instead.

## Notes

- Do not check build signal, or attempt to build or typecheck the app — assume CI handles this separately.
- Use `gh` to interact with GitHub (e.g. fetching a PR diff), rather than WebFetch.
- Make a todo list first, one item per step in this skill.

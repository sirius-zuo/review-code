# review-code — Test Scenarios & Evidence

Manual verification scenarios. No automation — invoke the skill against the described input and compare against expected output.

## TS-1: Dual output — local diff mode

**Input:** Any local git repo with at least one uncommitted change that introduces an obvious bug (e.g. an off-by-one in a loop bound) plus a root `CLAUDE.md` with one explicit rule the change violates.

**Invocation:** `/review-code` (no arguments — local diff mode against the repo's default base branch)

**Expected output:**
- `docs/review-code/<timestamp>-<branch>.md` exists, with the standard findings table.
- `docs/review-code/<timestamp>-<branch>.html` exists, opens in a browser without errors, and contains a `<table>` with the same row count and same Severity/Location/Description content as the `.md` file.
- Severity cells in the HTML render with the correct `sev-critical`/`sev-major`/`sev-minor` class and visually distinct color.
- Console output matches the `.md` file content (HTML is not printed to console).

**Fail signals:**
- Only one of the two files is created.
- HTML file's table row count or content diverges from the markdown file's table.
- Console output includes raw HTML.

## TS-2: Dual output — no issues found

**Input:** A trivial, compliant change (e.g. a comment-only edit) with no CLAUDE.md violations and no bugs.

**Invocation:** `/review-code`

**Expected output:**
- Both `.md` and `.html` files exist.
- Both contain the "No issues found" line (HTML rendered as plain text, no table).
- No appendix block rendered in either file (no `<details>` element in the HTML).

## TS-3: Default mode — subagent dispatch, 5 lenses

**Input:** `/tmp/review-code-fixture` with the uncommitted diff from the implementation plan (Task 5, Step 2).

**Invocation:** `cd /tmp/review-code-fixture && /review-code` (no flags — local diff mode, default lens set)

**Expected output:**
- Findings for lenses 1, 2, 3, 5 appear (the CLAUDE.md violation, the removed-guard bug, the git-history confirmation, and the misleading comment) — these may be merged into fewer rows than 4 if Step 4's scoring recognizes they're the same underlying issue from converging lenses.
- Lens 4 is explicitly noted as "skipped — no PR host in this mode," not silently omitted.
- None of lenses 6–12's findings appear (`format_dollars` reuse, the `total`/loop simplification/efficiency findings, the `== 0.0` pitfall, or the `handler.py` bandaid) — `--full` was not given.
- Both `docs/review-code/<timestamp>-<branch>.md` and `.html` exist with matching content.

**Fail signals:**
- Any lens-6–12 finding appears without `--full`.
- Lens 1, 2, 3, or 5's finding is missing.

## TS-4: `--full` mode — all 12 lenses

**Input:** Same fixture and diff as TS-3.

**Invocation:** `cd /tmp/review-code-fixture && /review-code --full`

**Expected output:**
- Everything from TS-3, plus findings for lenses 7, 8, 9, 10, 11, 12 (lens 6's removed-behavior finding likely merges with lens 1/2's same-line finding during Step 4 scoring, same converging-evidence pattern as TS-3).
- The `format_dollars` (lens 9), simplification/efficiency (lenses 10–11), and altitude (lens 12) findings have an Impact description stating a concrete cost (duplicated logic, wasted loop, harder-to-maintain bandaid) — not a crash scenario.
- In the report's ordering, any correctness-bug finding (lenses 1–8) that ties on severity and confidence with a cleanup finding (lenses 9–12) is listed first.

**Fail signals:**
- A lens 9–12 finding's Impact column describes a crash/failure scenario instead of a concrete cost.
- A cleanup finding is listed ahead of a tied correctness-bug finding.

## TS-5: `--sequential` — dispatch timing only, no behavior change

**Input:** Same fixture and diff.

**Invocation:** `cd /tmp/review-code-fixture && /review-code --sequential` and separately `/review-code --full --sequential`

**Expected output:** Identical findings to TS-3 and TS-4 respectively — `--sequential` changes lens-subagent dispatch timing only, never which lenses run or what they find.

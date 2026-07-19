# AG Code Updates
<!-- Created: 2026-04-04 | Format reference — see REFS/AG-Code-Updates.template.md -->

================================================================================
AG UPDATE - 18JUL26 - Fix: .lintignore exclusion word-split on spaces (files never removed)
================================================================================

## 18JUL26: .lintignore Exclusion Broke on Filenames Containing Spaces

**Problem Report:** Tetra Lint Fix Wizard on ASC/plt_drugs excluded 2 non-code files via `.lintignore`, but Code Check kept failing on those exact files, keeping the Git Ops pull blocked. The 2 files: `!requirements/ox_inventory items.lua`, `!requirements/qb-inventory items.lua` — both contain a space.

**Root Cause:** The "Exclude Non-Lintable Files" step iterated `for f in $(find . -not -path './.git/*' -path "./$line" 2>/dev/null)`. The unquoted `$(...)` word-splits `find` output on IFS whitespace, so a path with a space (`./!requirements/ox_inventory items.lua`) split into two bogus tokens (`./!requirements/ox_inventory` + `items.lua`); `rm -f` no-op'd on both, the real file survived, got linted, and failed — defeating the exclusion entirely. `xargs`-based trim compounded the fragility (mangles quotes/backslashes).

**What Changed:**
- `lint-reusable.yml` (Exclude Non-Lintable Files step) — trim comment/whitespace with bash parameter expansion (no `xargs`), and iterate `find … -print0 | while IFS= read -r -d ''` so paths containing spaces or globs are removed correctly.

**Verified locally:** temp tree with a spaced `.lua` file + `.lintignore` — OLD logic left the file (removed=2 bogus); NEW logic removes exactly the spaced file, leaves real `.lua` intact, exits 0 under `set -eo pipefail`.

**Pairs with:** `Architected-Gaming/tetra-distributed` PR #299 (hub 7.98.17) — makes the Lint Fix Wizard actually re-trigger a lint run after pushing `.lintignore` (a `.lintignore`-only push didn't match the caller `paths:` filter). Both needed for the ASC block to clear.

================================================================================
AG UPDATE - 04APR26 - Root cause fix: --ignore arg order consuming file path
================================================================================

## 04APR26: Root Cause — --ignore Consuming File Path Argument

**Problem Report:** Discord lint notifications showed no error details for syntax errors. Multiple fix attempts (fallback parsers, `<error>` element handling) failed because the actual problem was upstream: `junit.xml` contained luacheck's usage/help text instead of JUnit XML.

**Investigation:** Downloaded `junit.xml` as a GH Actions artifact [tool: gh run download]. File contained `Usage: luacheck ... Error: missing argument 'files'` — not XML at all. Traced to the action's entrypoint which constructs: `luacheck ... --formatter JUnit --ignore 1 2 3 4 5 6 .` — the `--ignore` flag takes multiple args (`--ignore <patt> [<patt>] ...`) and was consuming the `.` file path as an ignore pattern. The second exec worked because the entrypoint appends `--formatter default` between `--ignore` patterns and `.`, breaking the chain. [read: entrypoint.sh, tool: GH Actions log exec commands]

**Root Cause:** Arg order. `--ignore 1 2 3 4 5 6` was the last option before the `.` file path. Luacheck's argparse consumed `.` as a 7th ignore pattern, leaving no file argument. This was introduced when `--ignore 1 2 3 4 5 6` was added to suppress warnings.

**What Changed:**
- `lint-reusable.yml:57` — Reordered args from `-t --formatter JUnit --ignore 1 2 3 4 5 6` to `-t --ignore 1 2 3 4 5 6 --formatter JUnit`. Now `--formatter` breaks the `--ignore` chain before the file path in both execs.

**Additional changes in this session:**
- Webhook split: `DISCORD_LINT_WEBHOOK` fires on all runs, `DISCORD_LINT_ERRORS_WEBHOOK` on failures only
- Parser checks both `<failure>` and `<error>` JUnit elements
- Raw-text fallback + generic message as last resort
- `timeout=10` on webhook urlopen
- Errors webhook gate uses `failed > 0 or job_status == "failure"`
- Simplified false-positive hint text

---

================================================================================
AG UPDATE - 04APR26 - Fix: JUnit fallback parsing + errors webhook job_status gate
================================================================================

## 04APR26: Fix Malformed JUnit XML Handling + Errors Webhook Gate

**Problem Report:** On a syntax error (`config/config.lua:7:26: expected '=' near 'CODE'`), the all-messages webhook showed a bare "Failed / 0 passed" with no error details. The errors-only webhook didn't fire at all.

**Investigation:** GH Actions log showed `##[error] Failed to parse file (junit.xml) with error Error: Text data outside of root node.` [tool: GH Actions log]. Luacheck's JUnit formatter produces invalid XML on syntax errors. The Python `ET.parse()` hit `ET.ParseError`, the `except` block silently swallowed it [read: lint-reusable.yml:112], leaving `failed=0` — so no error details were collected and the errors webhook gate (`if failed > 0`) blocked it.

**Root Cause:** Two issues: (1) No fallback when JUnit XML is malformed — errors silently lost. (2) Errors webhook gated solely on JUnit `failed` count, not on actual job failure status.

**What Changed:**
- `lint-reusable.yml:112-124` — Added raw-text fallback in the `except` block: reads `junit.xml` as plain text, extracts lines matching `.lua:` + error patterns, populates `errors[]` and increments `failed`.
- `lint-reusable.yml:176` — Errors webhook gate changed from `if failed > 0` to `if failed > 0 or job_status == "failure"` so it fires even if JUnit parsing completely fails.

---

================================================================================
AG UPDATE - 04APR26 - Lint webhook: main WH fires on all runs, errors WH on failures only
================================================================================

## 04APR26: Split Lint Discord Webhook Behavior

**Problem Report:** Both `DISCORD_LINT_WEBHOOK` and `DISCORD_LINT_ERRORS_WEBHOOK` were gated behind `if failed > 0`, meaning successful lint runs produced no Discord notification at all. User wanted one webhook to report all results (pass + fail) for visibility.

**Investigation:** Reading `lint-reusable.yml:155-172` confirmed both webhooks shared a single `if failed > 0` gate with a loop over both URLs [read: lint-reusable.yml:156-158].

**Root Cause:** Original implementation treated both webhooks identically — both errors-only. No webhook covered passing runs.

**What Changed:**
- `lint-reusable.yml:155-175` — Extracted a `send()` helper function. `DISCORD_LINT_WEBHOOK` now fires unconditionally (every run). `DISCORD_LINT_ERRORS_WEBHOOK` remains gated behind `if failed > 0`.

---
<!-- Prepend new updates above this line -->

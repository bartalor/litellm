# pre-commit-issue notes

Context: investigating whether `scripts/pre_commit_lint.sh` (invoked by `make pre-commit`) has a real issue worth raising, or whether its design is coherent as-is.

## What the script does

1. Line 22: `git diff --cached --name-only --diff-filter=ACMR` — produces the staged file list. Source of "what to lint".
2. Lines 28-39: pattern-match that list into per-target buckets (litellm python, tests/e2e python, dashboard files, proxy/types spec files).
3. Lines 48-55: warn if the working tree has unstaged or untracked changes, since the tools read the working tree and results may not correspond to a commit of only the staged set.
4. Lines 97-152: run the per-target checks — `make lint`, `ruff format --check`, `make lint-e2e-basedpyright`, prettier + eslint + budgets, dashboard api types regen.
5. Line 144: after `npm run gen:api`, `git diff --quiet -- schema.d.ts` — checks whether regen produced a file that differs from the staged version.

## State model — is it consistent?

Initial suspicion: the script mixes state models (file list from index, tools read working tree, drift check compares working tree to index, warning papers over the mismatch), and pre-commit should pick one.

Retracted after tracing. The design is coherent:

- The tools (`make lint`, `ruff format --check`, prettier, eslint, `gen:api`) all read the working tree. They can't read the index without a temp checkout. Running against the working tree is not a choice, it's the only option.
- `--cached` isn't about "what state to lint"; it's about "which files to bother with" — scope the work to files that are part of this commit.
- The drift check at line 144 compares regen output (working tree) to the staged `schema.d.ts` (index). That's exactly the right question: the user is about to commit the staged version; if regen produced something different, the commit will contain stale types. Comparing to HEAD instead would miss the case where the user regenerated and staged, then edited proxy sources without re-staging.
- The warning at 48-55 exists precisely because working-tree-vs-index can't be fully reconciled without a temp checkout, which the script comment (41-47) explicitly rejects as "no safe way to lint the index in place". The warning surfaces the residual gap instead of hiding it.

So: file list from index (what's being committed), tools run against working tree (only option), drift check compares regen to index (what will be committed). Coherent.

## The remaining actual gap

The script only supports "predict CI for what you have staged". There's no supported way to run the same checks over an already-committed change or an arbitrary file list. The `--cached` on line 22 is the only reason for that limitation; everything downstream operates on a file list and works against the working tree regardless of where the list came from.

Two of the pieces are genuinely pre-commit-specific and don't generalize:
- The unstaged/untracked warning (48-55) — working-tree-vs-index by definition.
- The schema.d.ts drift check (144) — compares to the index specifically because that's what's about to be committed.

Everything else (the per-target check blocks) would work verbatim given a different file list.

## Proposal shape (not yet drafted)

Extract the file-list-consuming checks into a standalone script that takes a file list on stdin (or argv). `pre_commit_lint.sh` keeps its exact current behavior and interface: computes the staged file list with `git diff --cached --name-only --diff-filter=ACMR`, passes it to the new script, keeps the warning and drift check locally. No new env vars, no new flags on the pre-commit path, no behavior change for the hook. The same checks become directly callable with any file list.

(Considered piping a diff instead of a file list. Rejected: the tools read the working tree, not the diff, so the diff content would be thrown away and only the filename extraction used. Passing a file list matches what the checks actually consume; piping a diff would require reimplementing git's filename extraction including `--diff-filter` and rename handling.)

## Open questions

- Is the extraction worth the churn given the only concrete use case so far is "I forgot to run pre-commit before committing, want to run it retroactively"?
- If yes, does the new script live under `scripts/` with a name that reflects "checks for a file list" rather than pre-commit specifically?

## Strict gates hardcode `origin/litellm_internal_staging` as the base

`Makefile` invokes `scripts/ruff_strict_gate.py`, `scripts/type_check_gate.py`, and `scripts/type_discipline_gate.py` with `--base origin/litellm_internal_staging` (lines 180, 188, 203), and `ruff_strict_gate.py` sets `DEFAULT_BASE = "origin/litellm_internal_staging"`. This assumes every contributor's `origin` is the upstream repository. For a fork workflow — where `origin` is your personal fork and `upstream` is the project — `origin/litellm_internal_staging` is stale (a snapshot of the last time your fork was synced), so the gate blames every violation introduced upstream since your last fork-sync on your change. Concrete data point: my fork was 158 commits behind, and the gate reported 32 added UP045 hits for a change whose actual diff added 1.

The right base to compare against is "wherever this branch's changes started diverging from the intended target," which for a fork PR is `upstream/<target-branch>`. Hardcoding `origin/…` bakes in a monorepo/direct-push assumption. Options that would fix this without forcing every contributor to keep their fork in sync:

- Read the base from an env var (e.g. `STRICT_GATE_BASE`) with the current value as fallback, so fork contributors can set it once.
- Use `git for-each-ref` (or a repo-local config knob) to resolve the target branch on whichever remote actually has the freshest tip.
- Have the Makefile fetch the base ref before running the gate, but from a configurable remote.

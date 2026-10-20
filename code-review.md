# Code Review

Run every pass below, in order. One pass at a time, to completion, with its own individual commit — its own findings, its own test run, and its own labelled commit — before starting the next.

## Mandatory Commit Discipline

**Every pass MUST result in exactly one individual commit. No exceptions.**

This requirement applies even when:

- the pass finds no bugs;
- the pass makes no code changes;
- the pass is not applicable;
- the pass only produces findings that are deliberately deferred;
- the pass only produces a comment documenting an observation, investigation, or conclusion;
- the pass determines that an existing implementation is correct;
- the pass produces only tests;
- the pass is stopped early by a gate.

**Do not skip a commit because there is "nothing to commit." There must still be a commit recording the completion and result of the pass.**

**The commit message is the record.** Do not write review documents into the repository — no `docs/review/`, no `REVIEW-n.md`, no report file. A file duplicating the commit message is a second copy to keep in sync, it detaches from the change the moment anyone cherry-picks or rebases, and it goes stale against the diff sitting next to it. `git log <base>..HEAD` is the review.

If a pass produces no source-code change, commit anyway:

```bash
git commit --allow-empty -F -
```

An empty commit is the correct mechanism here, not a loophole — the message is the deliverable, and there is nothing else the pass needed to leave behind.

Whether or not it changes code, the message must contain:

- what the pass examined;
- what was found — with `file:line`;
- what was NOT found, when that is the result;
- the evidence supporting the conclusion;
- the disposition of each finding;
- tests/checks that were run, with their before/after counts;
- anything deliberately deferred to a later pass, and why.

Write the message for someone reading `git log` a year from now with no other context. A one-line message is not a record.

### Commit Ordering Is Mandatory

The following sequence is required for **every** pass:

1. Start the pass.
2. Perform the pass completely.
3. Run the required tests/checks.
4. Record all findings and dispositions.
5. Make only the changes permitted by that pass.
6. Run the required post-change tests/checks.
7. Write the pass's record into the commit message.
8. Create **exactly one commit for that pass** — `--allow-empty` if it changed no code.
9. Verify the commit exists and contains only that pass's work.
10. **Only then may the next pass begin.**

Do not begin pass `N+1` while pass `N` has uncommitted work.

Do not accumulate changes from multiple passes and commit them together.

Do not amend a previous pass's commit with findings or fixes discovered during a later pass. If a later pass discovers something that belongs to an earlier pass, record it as a finding in the current pass and follow its disposition rules.

### No Skipping

**Every listed pass must be executed and committed.**

A pass may be:

- `PASS` — reviewed and no blocking findings remain;
- `N/A` — the pass genuinely does not apply;
- `DEFERRED` — a finding is intentionally left for a later pass, with the reason documented;
- `RETURN` — a gate requires the review to stop.

None of these statuses permits skipping the commit.

For example, this is valid:

```text
[REVIEW-7][accuracy] N/A — no numeric logic is present in this change.
```

That still requires a commit documenting that determination.

This is also valid:

```text
[REVIEW-3][scale-honesty] No findings.
Expected production N: unknown; no production volume data was supplied.
Checked all collection construction and external-call loops.
```

That still requires a commit.

This is **not** valid:

```text
Pass 3 doesn't seem relevant, so skip it.
```

### Findings Must Not Be Lost

If a pass produces even a single observation, it must be recorded.

A finding does **not** need to result in a code change.

For example:

```text
Finding:
`foo.py:142` performs an unbounded read.

Disposition:
Deferred to Pass 9 because the current pass is restricted to
scale analysis and changing the data-loading abstraction would
constitute a structural simplification.

Commit:
[REVIEW-3][scale-honesty] Document unbounded export read
```

Likewise, if the conclusion is that something is correct, document the evidence:

```text
Finding:
None.

Evidence:
`foo.py:142-168` was traced for empty, single-item, and
10,000-item inputs. The collection is paginated before being
materialized.

Commit:
[REVIEW-3][scale-honesty] Record scale review
```

### Commit Isolation

Each commit must represent **one and only one pass**.

Do not:

- fix a Pass 6 finding during Pass 3;
- include Pass 4 changes in the Pass 3 commit;
- combine two passes because their changes touch the same files;
- squash the review commits afterward;
- amend an earlier review commit with later findings;
- hide review findings exclusively in the final report.

The Git history itself must tell the story of the review:

```text
REVIEW-1 → REVIEW-2 → REVIEW-3 → REVIEW-4 → ... → REVIEW-12 → REVIEW-13
```

A reviewer must be able to inspect any individual `REVIEW-n` commit and determine exactly what happened during that pass.

Do not merge two passes into one commit. Do not fix something out of turn: if pass 3 makes you notice a boundary bug, write it down and handle it in pass 6.

## Commit format

```
[REVIEW-<n>][<slug>] <imperative summary>

<what this pass looked for>
<what it found — file:line>
<what changed, and what was deliberately left alone>
```

If a pass changes nothing, still commit — `git commit --allow-empty` — and put the pass's result in the message. The commit is how the pass is reported.

## Rules for every pass

- A finding needs a **location** and a **failure**. `file.py:142` plus concrete inputs that produce a wrong result. "This feels fragile" is not a finding.
- **Mark confidence.** CONFIRMED = you ran it or traced every branch. SUSPECTED = you reasoned about it. Never present the second as the first.
- **Finding nothing is a real result.** Say so plainly. Manufacturing a finding to justify a pass buries the real ones.
- Every finding gets a **disposition**: fixed, deferred with a reason, or rejected with a reason.
- Never fix and refactor in the same commit.
- Run the tests before and after each pass.
- **Every "Produce:" below goes in that pass's commit message**, not into a file. The message is the only place a pass's findings live.
- **Stop early and say so** if a pass invalidates the whole approach — a criterion that can't be implemented as designed, an operation that fundamentally can't be made idempotent. Reviewing seven more passes of code that's about to be rewritten buries the finding that mattered. The stopping pass must still produce its required individual commit, documenting the reason for the stop and which subsequent passes were not executed.

---

## Pass 1 — Human Readability · `[REVIEW-1][readability]`

*Can someone who has never seen this code work out what it does — and be right?*

The most important pass. Every later pass is bottlenecked on comprehension — you cannot spot a boundary error in a function you cannot follow. And the judgment is perishable: you only read this code for the first time once, and passes 2–12 will teach you how it works and destroy your ability to tell whether it was legible to begin with.

Do this cold — before the tests, before the ticket, before the call graph.

- **The stranger test.** Read it once, top to bottom, at normal speed. Write down what you think it does. Then check. Every place your summary was wrong is a finding — at the line that misled you, not the line you misunderstood.
- **The surprise inventory.** Every "huh?" and every re-read, written down. Each is either a real bug or a missing explanation, and you won't remember one of them once the code is familiar.
- **Time to competence.** More than ~10 minutes to feel safe changing a single function is a finding.
- **Names.** Names you must read the body to understand. Abbreviations only the author can expand. Booleans that read backwards (`disable_no_cache`). Missing units (`timeout` vs `timeout_ms`). Generic names (`data`, `handle`, `manager`) doing specific work.
- **One thing, one level.** Business logic beside byte manipulation in one function body — a finding even when both halves are correct.
- **Control flow you can hold in your head.** Nesting depth, pyramids that guard clauses would flatten, loops with several exits, flags steering a branch a hundred lines away.
- **Comments.** A comment explaining *what* means the code should be clearer — fix the code, delete the comment. A *why* comment is valuable — keep it, verify it's still true. A missing *why* on a non-obvious decision is itself a finding. A stale comment is worse than none.
- **Implicit knowledge.** Undocumented invariants, magic numbers, required call ordering, assumptions about the caller. Each is a trap for the next person.
- **Constants that depend on each other.** For example, a lease that must be shorter than a window, or a threshold set at three intervals. If each is its own literal and only a comment ties them together, changing one silently breaks the others. Derive them from one source in code. Judge here, perform in pass 10.
- **Errors as documentation.** `raise ValueError("invalid")` tells a reader at 3am nothing.
- **The diff itself reads.** Formatting churn mixed into logic, or a file move combined with edits, makes a diff unreviewable no matter how good the result is.

**Produce:** your first-read summary plus the list of places it was wrong, in the commit message. That list is the finding set, and it's the only thing here you cannot reconstruct later.

Behavior-preserving only. Renames, comments, guard clauses, extracted helpers. Tests pass unchanged. A readability fix that needs a behavior change is a finding for a later pass — log it, don't do it. Large structural moves: judge here, perform in pass 10.

### Gate — this pass stops the review

RETURN the change and do not run passes 2–12 if:

1. Your first-read summary was wrong about *what the change does* — its purpose or primary effect, not a detail.
2. You could not write a summary without first reading the tests, ticket, or call graph.
3. More than one comprehension failure per ~50 lines of diff, where the failures were missing explanations rather than real bugs.
4. Any single function took more than ~10 minutes to feel confident changing.
5. The diff is unreadable — logic buried in formatting churn, or a move combined with edits.

Running nine more passes over code nobody can read produces a review that looks thorough, finds nothing, and ships with a green check. That's why this stops rather than warns.

A RETURN must be itemized — the wrong summary, the surprise inventory, the function and its timer. If you can't list them, it's a PASS with findings, not a RETURN. Hand back the artifacts, not the verdict: "unclear, please clean up" is useless and invites a cosmetic resubmission.

---

## Pass 2 — Acceptance Criteria · `[REVIEW-2][acceptance]`

*Does this do everything that was asked, and hold up when the input isn't the happy path?*

Write the acceptance criteria down as a checklist first. If they were never written, write them now and confirm them — you can't review completeness against criteria that don't exist.

- Map every criterion to the code path that satisfies it. A criterion with no code path is a gap. A code path with no criterion is scope creep — also a finding.
- Partial implementations: happy path only, TODO, stubbed returns, a hardcoded value standing in for real logic, a parameter accepted and ignored.
- Every input boundary: empty, null, single element, max size, malformed, duplicated, out of order, wrong type, unexpected unicode, very long strings.
- Every external call: timeout, 4xx, 5xx, empty body, malformed payload, unexpected schema, slow-but-successful.
- Everything swallowed silently: bare `except`/`catch`, `|| default`, ignored return values, errors logged at debug and discarded.
- The inverse of each criterion. "Only admins can X" — find the path where a non-admin reaches X.
- **Limits against the criterion's wording.** A floor, cap or validation that refuses part of what a criterion promises narrows that criterion. For example, a 10-minute minimum on a criterion that promises minutes. That's the ticket owner's decision, not the reviewer's. If the limit stays, the user must see it where they make the choice, not only as an error on save.
- **Every starting state.** Run each criterion from a new record, from an existing record, and from a new or edited rule or config reaching records that already exist. A path tested only from "already in the pipeline" misses the record that never enters it.

**Produce:** criterion → code location → met / partial / missing, every row filled in.

---

## Pass 3 — Scale and Honesty · `[REVIEW-3][scale-honesty]`

*What breaks at 100x, and does the code tell the truth about what it does?*

One pass because they fail together — code that misrepresents itself is how a scale problem stays hidden.

### Scale

- Unbounded collections in memory. Full result sets loaded to count them. Files read whole.
- Per-item network or DB calls inside a loop (N+1).
- Missing pagination, missing `LIMIT`, missing cursor.
- No timeout, no backpressure, unbounded queue depth or concurrency.
- Nested loops over the same growing collection; repeated lookups that should be a dict.
- State the real N. Expected production volume, versus the volume this was actually tested at. If nobody knows, that's the finding.

### Honesty

- Names that overstate: `validate_all()` checking one field, `sync()` that fires and forgets.
- Logs and metrics reporting success on partial failure.
- Fallbacks substituting a degraded result without marking it degraded.
- Caught exceptions turned into a success return.
- Progress counts and percentages that don't reflect real work.
- "Retry" that doesn't retry, "cache" that never hits, "batch" that processes one at a time.
- Docstrings and READMEs describing behavior the code doesn't have.

**Produce:** the N at which each scale finding bites; the specific claim beside the specific reality for each honesty finding.

---

## Pass 4 — Survivability · `[REVIEW-4][survivability]`

*If this dies halfway through, what does the world look like — and can you just run it again?*

- **Kill it at every step boundary.** What's left half-written? Is that state detectable, or does it look complete?
- **Idempotency.** Running twice gives the same end state — no duplicates, no double-charges, no doubled counters.
- **Resumability.** Checkpoint or cursor, or does a failure at 95% mean starting from zero?
- **Transaction boundaries.** Writes that should be atomic but aren't. A transaction held open across a network call. A commit before the side effect it was supposed to include.
- **Retries.** Bounded? Backoff? Jitter? Anything non-idempotent being retried?
- **Poison input.** Does one bad record halt the batch forever, or get quarantined?
- **Cleanup on the failure path.** Temp files, locks, connections, handles released when the happy path doesn't run?
- **Two instances at once.** Cron overlap, duplicate consumers, a retry racing the original.
- **Two versions at once.** A rolling deploy runs old and new code side by side, and a rollback runs the old code against what the new code wrote. A step that removes "anything that isn't mine" (another interval, another schema, another worker's lock) lets the two versions undo each other on every restart. Make such steps one-way.
- **One field, one meaning.** Writing a lease, a lock or a retry marker into a field that already means something (a due time, a status) destroys the original value. Ordering, health signals and recovery then all read the marker instead. Keep the marker in its own field.
- **When a value stops being overwritten, find who relied on the overwrite.** Moving a lease out of the due-time field meant only the evaluation moved the due time. Every path that returns before writing it, such as a record with nothing left to evaluate (canceled, deleted, skipped), was then stuck for good. List those paths and give each a write or a cleanup.
- **Dependency down.** DB, queue, or third-party API unavailable: degrade, fail loudly, or hang?

**Produce:** the failure injection you'd run for each, what you expect, what the code actually does. Run the cheap ones and say that you did.

---

## Pass 5 — Two Empirical Checks and a Regression · `[REVIEW-5][empirical]`

*Enough reasoning. What happens when you run it?*

Passes 1–4 produce claims, and claims are cheap. Prove exactly two and leave a test behind.

1. Pick the two claims with the **highest cost if wrong** — not the two easiest to check. Most expensive to be wrong about in either direction: a real bug dismissed, or a non-bug "fixed".
2. **Design a check that can fail.** If you can't describe an outcome that would disprove the claim, the check is worthless.
3. **Run it against real code and realistic data.** If you mock the thing under test, you proved nothing about it.
4. **Record what you ran, what you expected, what happened** — especially when the last two disagree.
5. **Write one regression test of your own choosing.** Not a test the author would write — a test for the thing *you* suspect. It must fail against the buggy version and pass after the fix. If it passes immediately, the bug wasn't there; say so and keep the test.

"Both checks confirmed the code is correct and my regression passed first try" is a legitimate result. Report it plainly. The commit includes the test either way.

---

## Pass 6 — Boundaries and Order of Enforcement · `[REVIEW-6][boundaries]`

*Are limits, guards and checks applied at the right moment — or one iteration too late?*

Named for the cap enforced *after* the batch that exceeded it: technically present, quietly wrong by up to one batch.

- Every cap, limit, quota, budget: evaluated before the work or after? Before the append or after? That distinction is the entire bug.
- Loop boundaries: off-by-one in ranges, slices, indices, `<` vs `<=`.
- Check placement: a guard outside the loop that belongs inside, or inside when it belongs outside.
- Accumulators: does the counter increment before or after the guard reads it?
- Early exit that fires late — break condition evaluated at the top of the next iteration instead of the end of the current one, overshooting by a full unit of work.
- Rate limits and token budgets checked once at startup instead of per call.
- Validation after mutation. Authorization after the side effect — after the write, the email, the charge.
- Pagination edges: is the last page processed twice, or dropped? Is the cursor inclusive or exclusive, and does the code agree with itself?
- Deadlines: enforced per item or only per batch, so one slow item blows the window?
- **A threshold that grows with load needs a ceiling.** An allowance per queued item, a timeout per batch size, a retry budget per record: under enough load, a failure never trips it. Cap it, and state the worst-case delay the cap gives.
- **Every guarantee against every recovery path.** When the change promises a bound (the shortest window that works, the longest delay before something happens), check that each retry, lease, timeout and backoff fits inside it. A recovery slower than the guarantee breaks it on the first failure. Re-check whenever a later pass changes one of those values.

**Produce:** the exact overshoot. "Processes up to `batch_size - 1` records past the cap" is a finding. "The cap logic looks off" is not.

---

## Pass 7 — Accuracy · `[REVIEW-7][accuracy]`

*Are the numbers right?*

- **Arithmetic:** integer division truncating, floats holding money, rounding direction and half-rules, operation order.
- **Units:** ms vs s, bytes vs KB, cents vs dollars, basis points vs percent — and where each conversion happens.
- **Time:** timezones, DST, UTC vs local, inclusive vs exclusive range ends, and where "now" is evaluated.
- **Scheduled work runs late, by a different amount each time.** A cron or repeatable job fires on its boundary plus a pickup delay. Anything that compares one run to the next, such as a lease meant to expire before the next tick, needs margin for that jitter. A guarantee built by adding intervals must count every delay on the path (pickup, queue lag, processing), or be stated as best effort.
- **Aggregation:** nulls in averages, dedup before or after counting, `GROUP BY` mismatched to the select list, counts over a joined set.
- **Comparison:** float equality, string sort on numbers, case and locale sensitivity, null semantics.
- **Mapping:** similar field names crossed, silent type coercion, a default masking a missing value.
- **Ratios:** right denominator, divide-by-zero, percentage of the wrong base.
- **SQL:** a `JOIN` multiplying rows, `NOT IN` against a nullable column, a `WHERE` turning an outer join back into an inner one.

**Produce:** one real record worked end to end by hand, compared against the code's output. Use the most-used path, not the easiest one.

---

## Pass 8 — Cost · `[REVIEW-8][cost]`

*What does one run cost, what does it cost at 100x, and is there a ceiling that holds?*

The second arithmetic pass — pass 7 checks the numbers the code produces, this one the numbers it spends.

- **Put a number on one run:** API calls, tokens in and out, compute seconds, storage, egress. Not "it's cheap" — a figure. Then project it at expected volume and 100x. Linear, or worse?
- Anything billable inside a loop, a retry, or a hot path. A retry wrapper around a paid call multiplies the bill by the retry count, and nobody notices until the invoice.
- **LLM-specific:** prompt growing with input length, full context resent per item instead of batched, a bigger model than the task needs, no caching of identical calls, retries on non-deterministic output, output limits unset.
- **Redundant spend:** the same call twice in one run, a cache that never hits, data fetched and discarded.
- **Work that grows with the square of the input.** Per-item work that re-scans the whole collection, such as each breakpoint re-evaluating every element. Time it at the largest realistic N (a shipment with 100 stops, not the 4-stop fixture).
- **Idle cost:** polling intervals, always-on resources, connections held between runs.
- **The cap.** Is there one? And — see pass 6 — is it checked before the spend or after? A budget evaluated after the call was billed is the cap-a-batch-late bug with money attached.
- What happens at the cap: hard fail, degrade, queue, or silently keep spending?
- **Visibility.** Can anyone answer "what did this cost last month" without guessing?

**Produce:** a per-run figure and a projected monthly figure, with the assumption behind each. "No findings" still requires the numbers — that's what makes it finding-free rather than unexamined.

---

## Pass 9 — Reusability and Standards · `[REVIEW-9][reusability]`

*Before adding a new abstraction, utility, helper, pattern, or dependency, did you first look for an existing standard, function, component, or established pattern that should be reused?*

This pass exists to prevent the codebase from accumulating multiple ways to solve the same problem. **Our existing standards and functions take priority over inventing a new implementation.** A new abstraction is justified only when the existing one cannot reasonably satisfy the requirement.

### Repository-First Reuse

Before creating something new:

- Search the repository for an existing function, helper, service, utility, component, hook, module, abstraction, or pattern that already solves the problem.
- Check the project's established conventions and standards before choosing an implementation.
- Prefer an existing internal function over writing an equivalent local implementation.
- Prefer an existing shared abstraction over creating a one-off helper.
- Prefer the project's existing libraries and dependencies over adding another dependency that provides overlapping functionality.
- Follow existing naming, error-handling, logging, validation, configuration, and data-access conventions.
- Check nearby code and analogous implementations, not just exact name matches. The right reusable pattern may have a different name.
- Search for existing tests before adding new test utilities or fixtures.
- If an existing function is close but insufficient, determine whether it should be extended rather than duplicated.
- Do not create a new abstraction merely because the existing one has a slightly different interface or requires a small adapter.
- If a new abstraction is genuinely necessary, document why the existing standard or function could not be reused.

### Duplication

Look specifically for:

- New code duplicating an existing utility.
- Reimplemented validation that already exists elsewhere.
- Repeated API, database, HTTP, retry, logging, parsing, formatting, or error-handling logic.
- Multiple implementations of the same business rule.
- New constants or configuration values that duplicate existing ones.
- New wrappers around a dependency already wrapped by an internal standard.
- A new helper whose behavior substantially overlaps an existing helper.
- New test setup that duplicates existing fixtures, factories, builders, or harnesses.
- A new dependency introduced solely to perform work the codebase already supports.

### Standards Take Priority

If the repository has an established standard, use it unless there is a documented reason not to.

Examples:

```text
Existing validation utility
        ↓
Use it

Existing date/time utility
        ↓
Use it

Existing API client
        ↓
Use it

Existing logging abstraction
        ↓
Use it

Existing error type / error handler
        ↓
Use it

Existing database/repository pattern
        ↓
Use it
```

Do not introduce a competing implementation simply because it is personally preferred, more familiar, or easier to write locally.

### Produce

For every new function, helper, abstraction, dependency, or pattern introduced by the change, record:

```text
New thing:
<name and location>

Existing alternatives searched:
<functions, utilities, modules, standards, or patterns examined>

Why they were not reused:
<specific reason>

Decision:
<reused / extended / adapted / intentionally new>

Evidence:
<searches or references supporting the decision>
```

If nothing new was introduced:

```text
Result:
No new abstractions or overlapping implementations.

Repository reuse:
<what existing standards/functions were used>
```

A finding is required when a new implementation duplicates an existing capability without a documented reason.

**Behavior-preserving reuse is preferred.** If reusing or extending an existing function would require a behavior change, record that as a finding and defer the behavior change to the appropriate pass rather than quietly changing the shared implementation here.

---

## Pass 10 — Simplicity · `[REVIEW-10][simplicity]`

*What can be deleted?*

Late on purpose — only simplify code already proven correct, or you're rearranging bugs into harder-to-find shapes. This is also where structural moves deferred from pass 1 get performed.

- Abstractions with one implementation, or one caller.
- Config options nobody sets; feature flags permanently on.
- Dead code, unreachable branches, commented-out blocks, unused imports and exports.
- Duplicated logic that should be one function — and prematurely shared helpers that should be two.
- Pass-through layers that add no behavior.
- A dependency pulled in for one trivial function.
- Defensive code guarding impossible states. If you can't prove it's impossible, it stays — that's pass 2's territory, and pass 2 wins.

When simplicity and readability conflict, readability wins. An extracted, well-named helper adds a layer and is better. Don't collapse a clear three-step function into a dense one-liner and call it progress.

Behavior-preserving. Tests pass unchanged. If a test has to change, it isn't a simplicity change.

---

## Pass 11 — Linter · `[REVIEW-11][lint]`

*Does the changed code pass the project's own linter, run the way CI runs it?*

Late on purpose. Passes 1–10 and the situational passes all change code, so a lint run before them is already out of date by the time they finish. It runs before pass 12 because coverage is measured against the final diff, and lint fixes are part of that diff.

### Run what CI runs

- Find the real command instead of guessing it. Check the CI workflow, `package.json` scripts, the lint-staged or pre-commit config, and the `Makefile`. Use the repo's config and the repo's linter version. A linter run with your own defaults is linting a different project.
- Include formatters in check mode (`prettier --check`, `black --check`, `gofmt -l`). If the pre-commit hook or CI would rewrite a file, that is a failure even when the linter reports zero errors.
- Include the type checker if CI runs one as part of lint (`tsc --noEmit`, `mypy`).
- If the linter can't run (missing deps, a broken config), that is a finding. N/A means only that the project has no linter configured. Record what you checked to establish that.
- **Afterwards, `git diff HEAD` must be empty.** Run the CI command from the directory CI runs it in. lint-staged run that way (`lint-staged --diff="origin/<base>...HEAD"`) exits non-zero with "prevented an empty git commit" when it had nothing to change. That is the clean result; the empty diff is the check.
- **Every app the change touches, each with its own config.** In a monorepo one change can span apps with different linters, and one may have no CI lint job at all. Lint it with its own `lint` script and type checker anyway.
- **Every branch the change ships on.** Cherry-picks land on different bases, so lint each one. Don't infer one branch's result from another's.

### Scope to the diff

```bash
git diff --name-only --diff-filter=ACMR <base>...HEAD
```

Lint those files, then sort every result into one of two buckets:

- **On touched lines:** added or modified in this diff. These are this change's findings. Each one gets fixed or gets a written exception.
- **Pre-existing:** on lines the diff didn't touch. Count them and leave them alone. Fixing them is formatting churn in code this change doesn't own, which pass 1 calls unreviewable. Exception: if CI lints whole files (lint-staged usually does), a pre-existing error in a touched file fails the build anyway. Fix it here and say so. Check the CI config before sorting. A file an earlier review pass touched for the first time (adding a test to an old spec, say) brings its old errors into the diff with it.

### Fixing

- Use autofix only for mechanical rules: formatting, import order, quote style. Read the autofix diff before committing it.
- Some lint fixes change behavior, for example `eqeqeq` (`==` → `===`), `no-floating-promises` (adding an `await`) and `radix`. Treat each one as a behavior change. Run the tests, and record it as its own finding. It may be a real bug the earlier passes missed, so name the pass that should have caught it.
- **Suppressions are findings.** Every new `eslint-disable`, `# noqa`, `@ts-ignore` or `//nolint` needs a written reason in the commit message. A file-level disable added only to make the pass green is not a fix.
- Never loosen the linter config to make the diff pass.

**Produce:** the exact commands run, the files linted, and before/after counts split into touched lines and pre-existing. List each fix and whether it preserved behavior, and each suppression with its reason. The bar is zero lint errors on touched lines. Each one that remains needs its own written exception.

---

## Pass 12 — Test Quality and Diff Coverage · `[REVIEW-12][tests]`

*Is every touched line covered — and does any of those tests actually check anything?*

Last of the numbered passes, because coverage can only be measured against the final diff, and passes 10 and 11 just changed it. (The General Pass that follows re-runs this pass, and pass 11, on anything it changes.)

### Coverage: 100% of the touched lines

The diff, not the project.

- Every line added or modified is executed by at least one test.
- Every branch introduced is taken both ways. 100% line coverage with an untested `else` isn't 100%.
- **Error paths count.** The `except` block, the timeout handler, the early return — those are the lines most likely to be wrong, because they're the ones nobody ran by hand.
- **Deleted code:** were its tests deleted too, or are they still testing nothing?

Use diff-aware coverage rather than eyeballing a percentage — `diff-cover`, `jest --changedSince` with thresholds, `nyc` with a diff filter, `go test -coverprofile` filtered to the diff.

Uncovered lines need a written exception, one per line. "Hard to test" is not an exception — it's a finding about the design.

### Coverage is necessary, not sufficient

100% proves the lines ran. It proves nothing was *checked* — a suite that executes everything and asserts nothing scores 100%. So:

- **Break things on purpose.** Mutate the touched lines that matter — flip a comparison, change a constant, delete a line, invert a boolean. Does a test go red? A covered line that survives mutation is covered and unverified, and the coverage number is lying about it.
- **Tests asserting their own mocks.** If the only assertion is that a mock was called, you tested the test.
- **Assertions that can't fail:** asserting a value the test just set, auto-updating snapshots, `assert response is not None` as the whole check.
- **Do the boundaries have tests?** Every pass 6 finding should have a test pinning the correct side.
- **The pass 5 regression test** — confirm it still fails against the pre-fix code.
- **Tests coupled to implementation.** Rename a private method and see what breaks. Tests that fail on a pure refactor will block every future simplification.
- **Isolation.** Run the suite in random order and the new tests alone.
- **Speed and flake.** A test too slow or flaky to run gets skipped, and a skipped test is worse than a missing one because it reads as coverage.
- **Pin constants exactly.** `expect(lease).to.equal(3 * interval)`, not `.to.be.above(interval)`. An inequality lets a value drift past the thing it protects with CI green. Where two apps mirror a rule, a test CI actually runs must pin one to the other.
- **The failures the change claims to survive.** If it claims to recover from a lost message, a failed publish or a crashed worker, a test must lose the message, fail the publish or kill the worker, end to end.
- **Coverage the project's config hides.** Projects often exclude UI or other folders from their coverage report. For the diff, override those excludes. A changed file with "no coverage data" is a gap, not an exception.
- **Clock-dependent tests that pass by accident.** A path that falls back to the real clock can pass against fixtures dated in the past. Check that a time test fails when the code reads the wrong clock.

**Produce:** the diff-coverage number with every uncovered line named, plus the mutation results — which lines you broke and whether a test caught each.

This pass gates too. If a touched line is uncovered without a written exception, or a mutated line survived without turning a test red, the change is not reviewed — however clean passes 1–11 were.

---

## Situational passes

Run these when they apply, before pass 11, so the linter and the coverage check both see their changes.

### `[REVIEW-A][security]`

When the change touches auth, input parsing, queries, shell, file paths, or anything a third party reaches. Injection at every boundary (SQL, shell, template, path, deserialization), parameterized at the point of use. Authorization on every entry point, not just the one the UI uses. The hostile authenticated user — what do they reach by changing an ID? Secrets in code, logs, stack traces, or commit history. Unverified webhooks, redirect URLs, uploaded filenames used as paths. New dependencies with advisories.

**Produce:** the specific attacker, their starting position, what they reach.

### `[REVIEW-B][observability]`

When the code runs unattended. If this breaks at 3am, what tells you? Name the alert or record that there isn't one. A log line at every failure branch with an ID, not "processing failed". Errors distinguishable from noise. A metric for what matters, not what was easy to instrument. "Did it run, and did it finish?" answerable without opening the database. A run that processes 400 of 500 records and exits zero is the thing this pass exists to catch.

Also check:
- **Recovery code can reset the signal an alert reads.** For example, putting a stuck item back at "now" keeps the oldest-item age fresh through an outage, so the alert never fires.
- **An alert can fire on normal operation,** such as a backlog that is draining at full speed. Walk both: the real failure must trip the alert, and normal operation must not.

**Produce:** one real failure walked to the human who'd notice it.

### `[REVIEW-C][interface]`

When the change touches an API, schema, queued message, stored record, or persisted config. Enumerate existing callers rather than assuming. In-flight data — messages already queued, rows already written — still parse? Old client/new server and new client/old server, since both happen during a rollout. New fields optional, enums extended not replaced. If this is reverted after an hour of traffic, what breaks?

- **Stored data outlives a rollback.** New code can write something the old code misreads: a new enum value, a new operator, a new status. That is the rollback hazard. Keep the writer behind a feature flag until the reader has been deployed past its rollback window.
- **A rule enforced in two places** (a frontend check mirroring a backend validation, a constant copied into both): one source, or a test CI runs that fails when they drift. Both sides must use the same parser as the payload. Two parsers of one format will disagree on some input.

**Produce:** the caller list and a rollout/rollback ordering that works.

### `[REVIEW-D][data]`

When the change includes a migration, backfill, or new constraint. Constraints in the schema or only in application code? Existing rows that already violate the new rule — have you counted them? Is the migration reversible, and has the rollback actually been run? Will the backfill or index build lock a table, and for how long at production row count? Nullable columns the code assumes are populated.

**Produce:** the count of violating rows and the measured lock duration.

---

## General Pass — Completion and Bugs · `[REVIEW-13][general]`

*If this shipped right now, what would the first bug report be?*

The final pass, after pass 12. The other passes each look through one lens. This one reads the whole final diff, review commits included, top to bottom one more time, for two things: is the change *finished*, and is anything still *wrong*. It catches what fell between the lenses.

Read the review's own commits as hard as the original. Fixes written late in a review get the least scrutiny of any code in the change, and a fix in one pass can quietly break what an earlier pass established.

### Completion

- **Leftovers:** `TODO`, `FIXME`, debug logging, commented-out code, hardcoded test values, a temporary flag left on.
- **Half-applied changes:** a rename that still has the old name in a test, a doc, a config, an error message or a log line. A new field written in one place but not read, or read but never written. Every call site updated, not just the one you were looking at.
- **Everything that ships with it:** README, docs, PR description, changelog, config and environment variables, migrations, exports, build and package entries. Each matches the *final* code, not the code as it was before the review changed it.
- **Claims the review made:** comments and commit messages written in earlier passes that later passes made untrue, such as a comment justifying code that pass 10 deleted.
- **Repo rules about what the change adds:** build config comments, design-pattern docs, CI path filters. A new kind of file (a DB-backed test, a worker, a migration) may come with rules elsewhere in the repo and checks to re-run. Find them, run them, and update any count or claim the change made stale.
- **Loose ends:** every finding marked "deferred" has a reason and a home (a ticket or a line in the PR). Nothing was fixed on one branch, or in one repo, and forgotten on another.

### Bugs

- **The defects no single lens owns:** the wrong variable, a copy-paste slip, an inverted condition, a missing `await`, a shadowed name, a mutated shared or global value, a resource opened and never closed, an error message that names the wrong thing.
- **Interactions between fixes:** two passes changing the same code in ways that conflict, or a guard added in one pass that blocks a legitimate path another pass relied on.
- **Fresh eyes:** by now you know the code too well to see it. Where you can, hand the final diff to a reader who hasn't seen it, as in pass 1, and ask only for bugs.

Findings follow the same rules as every other pass: a location, a failure, a confidence and a disposition. When a finding is something an earlier pass should have caught, name that pass. That is feedback on the review, not only on the code.

The same goes for what reviewers find after the review is handed back. Validate each comment like any other finding, then name the pass that should have caught it, and add the check to this guide.

Fixes made in answer to review comments get the full passes too, not only tests: especially 4 (survivability), 6 (boundaries), 7 (time) and B (observability). A redesign that answers one comment can open new failure paths. The first fix to one PR's claim lease introduced three new bugs, all caught only on re-review.

Anything this pass changes goes through pass 11 and pass 12 again for the touched lines: lint-clean, covered, both branches taken, and at least one mutation caught. Run the full suite before and after. Like every pass it ends in its own commit (`--allow-empty` if it changed nothing), and the Produce below is that commit's message.

**Produce:** the completion checklist with every item marked done, not applicable, or found; then the bug list, each with location, failure and disposition. "Read the whole final diff and found nothing" is a legitimate result, but say how you read it: alone, or with a fresh reader.

---

## Report at the end

Write this as your closing message to the reader — do not commit it as a file. It is an index into the commits, which hold the detail.

For each pass: the findings, their disposition, and the commit. Both gates need an explicit verdict. "No findings", "not applicable", and "deferred, and here's why" are all complete entries.

```
[REVIEW-1][readability]   GATE: PASS. Summary wrong about retry scope. 4 renames, 2 why-comments.  <sha>
[REVIEW-2][acceptance]    3 findings, 3 fixed.  <sha>
[REVIEW-3][scale-honesty] N+1 on export, fixed. Honesty: clean.  <sha>
[REVIEW-4][survivability] No findings. Verified by re-running from a killed state.  <sha>
[REVIEW-5][empirical]     N+1 under 50k rows, cap at the boundary. Both held. Regression test added.  <sha>
[REVIEW-6][boundaries]    Cap enforced one batch late. Fixed.  <sha>
[REVIEW-7][accuracy]      N/A — no numeric logic in this change.  <sha>
[REVIEW-8][cost]          $0.11/run, ~$340/mo at 100k runs. Embedding calls batched 1→50.  <sha>
[REVIEW-A][security]      N/A — no auth, parsing, or query changes.  <sha>
[REVIEW-9][reusability]  Reused existing RetryPolicy rather than adding a duplicate.  <sha>
[REVIEW-10][simplicity]  Deleted unused abstraction (one impl, one caller).  <sha>
[REVIEW-11][lint]         eslint + prettier --check on 6 touched files. Touched lines 4 → 0 (2 autofixed, 1 eqeqeq behavior fix + test, 1 justified disable). 9 pre-existing left alone.  <sha>
[REVIEW-12][tests]        GATE: PASS. Diff coverage 100% (47/47 lines, 12/12 branches). Mutated 6, all caught.  <sha>
[REVIEW-13][general]      Completion: README still named the old flag, fixed. Bugs: one, a missing await in the retry fix from pass 4. Re-covered and re-linted.  <sha>
```

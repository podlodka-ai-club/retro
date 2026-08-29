---
name: j-qa
description: Write tests for a merge request — read the diff, work out what can actually break, and add the smallest set of tests that catches it. Use when the user asks to cover an MR/branch/diff with tests, says "напиши тесты", "покрой тестами", "какие тесты нужны на этот MR", or types /j-qa.
---

# j-qa

Write tests for a change. Not every test that could exist — the few that earn their place.

The reader is the author of the MR and whoever reviews it. A test that duplicates an existing
one, or that pins an implementation detail nobody will keep stable, costs more than it returns.

Run the steps in order. Do not skip to step 6.

## 1. Get the diff

- MR id or GitLab URL given → `glab mr diff <id>` (add `--repo <group/project>` for another repo).
- Nothing given → the current branch against its base: `git diff main...HEAD` (`...`, three dots —
  it excludes commits that landed on main after the branch started).
- A path or commit range given → use it as-is.
- Also read `git diff --stat` first: a diff that touches 40 files needs scoping before anything
  else, and the user should be told which part you are covering.

Then collect what a human has already found on this change — this is rank 1 in step 5, and it is
invisible in the diff:

- Review comments on the MR: `glab mr view <id> --comments`.
- A bug ticket linked from the MR or the branch name.
- Anything the user said in their own message ("оно падает, когда промокод истёк").

Read only. Never push, commit, or touch the MR itself in this flow.

## 2. Read the change, not the diff

A diff shows lines. You need the behaviour.

- Open the changed files in full, not just the hunks — the surrounding function decides what a
  changed line means.
- Trace who calls the changed code, and what calls out of it. A changed helper with nine callers
  is a different risk from a new leaf function.
- State the change in one sentence before going further: *"retries are now scheduled from the
  webhook instead of the cron, and `cancel` is a terminal status."* If you cannot write that
  sentence, you have not read enough.

## 3. Find what already covers it

Before proposing anything, find the tests that already touch this code.

- Locate the test files for the changed modules (mirror path, `test_*`, `*_test`, `__tests__` —
  whatever the repo uses).
- Grep the test suite for the changed function, class, endpoint and status names.
- For each existing test, note what it actually asserts. Many tests exercise a path without
  asserting the thing that just changed — those count as *not covered*.

Output of this step is a short list: covered / touched-but-not-asserted / untouched.

## 4. List the risks — what can break

Enumerate candidate failures before choosing any. Go wider than the happy path; write them as
concrete failures, not areas ("`retrying` → `cancel` leaves the invoice payable", not "status
handling").

Prompts that usually find real ones:

- **Boundaries** — empty collection, single element, zero, negative, missing optional field, the
  maximum the caller can send.
- **Error branches** — every `raise`/`except`/early return the diff adds or moves. These are where
  most escaped bugs live and where coverage is usually thinnest.
- **State transitions** — a status or flag that changed meaning: which transitions must now be
  impossible, and what happens if one is attempted twice.
- **Idempotency and retries** — the same webhook, task or request delivered twice.
- **Contract compatibility** — a field renamed, removed, or made optional, with existing clients
  still sending the old shape.
- **Permissions and roles** — a path now reachable by someone who could not reach it before.
- **Ordering and concurrency** — two callers hitting the changed code at once, when the code
  reads-then-writes.
- **Data the change silently stops writing** — a column, log row or event that no longer happens.

For each risk, note the class of bug it represents. The class matters more than the instance:
two risks in the same class rarely need two tests.

## 5. Choose what to write — the budget rule

There is no fixed number of tests. You decide how many — and the default is zero. A risk has to
argue its way into a test, not out of one.

### Rank

1. **A bug a human actually found** — outranks everything else. A linked bug ticket, a review
   comment, a failure the user described, a regression that has escaped before. Write that test
   even if it is slow and even if it is the only one you write.
2. **Delta to the existing suite** — of the remaining risks, only the ones no current test would
   catch. If a test in the *touched-but-not-asserted* bucket would catch the risk by gaining one
   assertion, add the assertion instead of a new test: it is the cheapest coverage there is.
3. **Everything else gets nothing.** Not a weaker test, not a smoke test — nothing. It goes in the
   skipped block of the report.

### Cost

Ranked outcomes, best to worst:

    no test  >  one fast test  >  a few fast tests  >  one slow test  >  several slow tests

**Fast** means: direct calls into the changed unit, in-memory fixtures, no DB, no network, no
sleeps, no app bootstrap. **Slow** is everything else. Several slow tests is the worst outcome
available — it is the state where the suite grows, the pipeline drags, and each individual test is
too expensive to ever be deleted.

So before accepting a slow test, try to make it fast: push the risk down to the unit that actually
decides the behaviour. A branch can almost always be tested where it is written rather than through
the endpoint that reaches it. If it still needs the DB or the network to reproduce at all, keep it —
but it now has to be worth the whole rest of the set.

### Tie-break

Two candidates left, one slot of patience: the faster one wins. Equally fast: the one that catches
the wider class of bugs wins — a test that fails for any broken transition beats one that pins a
single value.

Writing nothing is a legitimate result of this step. A rename-only diff, or one whose every risk is
already asserted, gets zero new tests and a report that says why.

## 6. Write the tests

- Match the repo's existing test style exactly: same framework, same fixtures, same factories,
  same naming, same file layout. Read two neighbouring tests before writing one.
- Reuse the fixtures and factories that exist. A new fixture needs a reason.
- One behaviour per test. If a test needs "and" in its name, it is two tests — or the wrong test.
- Assert the observable outcome (returned value, stored row, emitted event, raised error), never
  the internal call sequence. Mock only what leaves the process.
- Name the test after the behaviour it protects, not the function it calls:
  `test_cancelled_invoice_is_not_retried`, not `test_process_invoice_2`.

## 7. Run them

- Run the new tests. Then run the whole test file, then the module's suite — a new test that
  breaks a neighbour is a finding, not a nuisance.
- Every new test must fail against the pre-change behaviour and pass with the change. If it passes
  both ways it tests nothing about this MR. Verify it, do not assume it: `git stash` the source
  change (never the test), rerun, unstash.
- Report failures with their real output. Never describe a test as passing without having run it.

## 8. Report

Short. Three blocks:

1. **Written** — one line per test: the risk it catches, where it lives, and fast or slow.
   For every slow test, one clause on why it could not be made fast.
2. **Considered and skipped** — the risks from step 4 you did not cover, each with the reason
   (already covered by X / same bug class as Y / not reachable / costs more than it protects).
   This block is not optional. It is the part that gets reviewed, argued with, and — later — fed
   back when a bug escapes.
3. **Not testable here** — anything that needs QA through the interface, a provider sandbox, or a
   staging run. Name it plainly instead of writing a test that pretends to cover it.

## Never

- Write a test that asserts what the code says rather than what it should do — copying the
  implementation into an assertion pins the bug in place.
- Add a test whose only justification is a coverage number.
- Change source code to make a test pass. If the change is genuinely wrong, say so in the report
  and stop.
- Delete or weaken an existing failing test to get green.

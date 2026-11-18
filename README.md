# HARUWE — Fast Fix Studio

**Process case study:** how focused GitHub bug fixes and CI repairs are scoped, delivered, and verified.

This repository is not a product demo with inflated metrics. It documents the **working method** used for paid, reviewable patches: narrow scope, regression coverage, and a clear handoff. Profile: [thangtran7314](https://github.com/thangtran7314).

---

## What this studio does

| Situation | Outcome |
| --- | --- |
| One clear, reproducible bug | Minimal patch + regression test + verification note |
| Red or flaky CI | Baseline log, smallest fix, green check documented |
| Small workflow / API automation | Secret-safe config, short map, handoff |

**Not offered here:** full rewrites, open-ended “improve the codebase,” or work without confirmed scope and payment terms.

---

## Case study A — Flaky CI on a small Python package

*Illustrative process (pattern from typical CI repair work). Client and private repo details omitted.*

### Context

- **Symptom:** `pytest` passed locally; GitHub Actions failed intermittently on Python 3.11.
- **Constraint:** Maintainer wanted the pipeline green without changing product behavior.
- **Pack:** CI / Test Repair.

### Scope confirmed before code

1. Stabilize the failing job only.
2. No dependency major bumps unless required for the failure.
3. Leave a note on how to re-run and what “green” means.

### Investigation

1. Re-ran the failing workflow and captured the **first** red line (not the last noise in the log).
2. Compared local vs CI: different timezone handling in a date helper used by one test.
3. Confirmed the product path was fine; only the test assumed a fixed local offset.

### Change (shape of the fix)

- Adjusted the test to freeze time / use an explicit timezone.
- Left production code untouched.
- Diff stayed inside the test module.

### Verification

```text
Command: pytest tests/test_dates.py -q
Result:  passed (local)
CI:      workflow re-run on the PR branch — job green
```

### Handoff note (what the maintainer received)

```text
What failed: intermittent pytest on CI (timezone-sensitive assertion)
What changed: test only — explicit clock/timezone
What did not change: application date logic, dependencies, other jobs
How to verify: re-run the PR checks; or pytest tests/test_dates.py -q
Risk: low — no runtime path modified
```

**Why this builds trust:** the write-up matches the diff. Reviewers can check the claim in minutes.

---

## Case study B — Single bug with a missing edge case

*Illustrative process for a One Bug Fix Sprint.*

### Context

- **Symptom:** API returned `500` when an optional query field was an empty string (not omitted).
- **Constraint:** Public clients already sent `""`; changing the client was out of scope.
- **Pack:** One Bug Fix Sprint.

### Scope confirmed before code

1. Treat empty string as “not provided” for that field only.
2. Add a regression test for `""`, `null`/omitted, and a valid value.
3. No broader validation rewrite.

### Investigation

1. Reproduced with a minimal request (empty string vs omitted param).
2. Traced to a parse step that assumed non-empty when the key was present.
3. Fixed the empty-string branch; left other validation rules alone.

### Verification

```text
Command: pytest tests/test_query_params.py -q
Cases:   omit field | field="" | field="ok"
Result:  all passed
```

### Handoff note

```text
What failed: empty string on optional query field → 500
What changed: normalize "" to missing for that field; regression tests
What did not change: auth, other params, response schema for happy path
How to verify: run tests above; optional manual curl with ?field=
```

---

## Working method (every engagement)

```text
1. Read the issue → confirm scope + acceptance + price
2. Reproduce or document the blocker
3. Patch only the failing path
4. Add or adjust a regression check
5. Write a short verification + handoff note
```

### What “done” means

- Diff is reviewable without a guided tour of the whole repo.
- A maintainer can re-run the stated command and see the same result.
- Out-of-scope items are listed, not silently “improved.”

---

## Templates you can reuse

### Proposal comment (before work starts)

```text
Hi — I can take this as a narrow fix.

Plan:
1. Reproduce from the steps in the issue (or flag what’s missing).
2. Change only the path that fails — no unrelated refactor.
3. Add a regression check that fails before the patch and passes after.
4. Comment with the verification command and result.

I start after scope and payment terms are confirmed.
```

### Verification block (on the PR or issue)

```text
## Verification
- Command:
- Result:
- Environment: (OS / runtime / CI job name)
- Out of scope this change:
```

---

## Service packs (summary)

| Pack | Best for |
| --- | --- |
| **One Bug Fix Sprint** | One reproducible defect |
| **CI / Test Repair** | Red CI, flaky tests, release checks |
| **Automation Patch** | Small GitHub/API workflow, secret-safe |

Details and contact: [github.com/thangtran7314](https://github.com/thangtran7314) · PayPal: [paypal.me/tranvanthang14](https://paypal.me/tranvanthang14)

---

## Repo status

| Item | Note |
| --- | --- |
| Purpose | Public process / case-study write-up |
| Client code | Not published here (private by default) |
| Claims | No star counts, conversion rates, or accuracy percentages |

When a real engagement is cleared for public mention, a short entry can be added under **Public references** with a link to the issue or PR only — never private data.

---

*HARUWE · Fast Fix Studio — small diffs, clear tests, honest handoff.*

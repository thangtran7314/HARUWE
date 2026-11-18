# Regression test template

Use this when delivering a **narrow bug fix** or **CI repair**. Goal: one focused check that **fails before the patch** and **passes after**.

Not a full test-suite rewrite.

---

## Checklist

### Before writing

- [ ] Reproduced the bug (or documented the blocker)
- [ ] Named the exact input / step that fails
- [ ] Named the expected behavior after the fix
- [ ] Confirmed scope: only this behavior is under test

### While writing

- [ ] Test fails on current `main` / before patch
- [ ] Test passes with the patch applied
- [ ] Name describes behavior + condition (not `test_bug` / `test_issue_12`)
- [ ] One primary assertion (plus at most 1–2 edge cases)
- [ ] No reliance on machine timezone, absolute local paths, or live secrets

### Before handoff

- [ ] Command to run is one copy-paste line
- [ ] Noted environment (runtime, CI job name if relevant)
- [ ] Listed what this change did **not** touch

### Handoff block (paste into PR / issue)

```text
## Verification
- Command:
- Result before patch:
- Result after patch:
- Environment:
- Out of scope:
```

---

## Naming

```text
test_<subject>_<condition>_<expected>
```

Examples:

- `test_optional_filter_empty_string_does_not_500`
- `test_checkout_zero_quantity_rejected`
- `test_format_date_uses_explicit_utc`

Avoid: `test_new`, `test_fix`, `test_issue_123`.

---

## Python snippets

### HTTP / API edge case

```python
def test_optional_filter_empty_string_does_not_500():
    """Regression: empty string on optional filter used to return 500."""
    r = client.get("/items", params={"filter": ""})
    assert r.status_code != 500
    assert r.status_code in (200, 400)  # lock one behavior in scope


def test_optional_filter_omitted_still_ok():
    r = client.get("/items")
    assert r.status_code == 200


def test_optional_filter_valid_value():
    r = client.get("/items", params={"filter": "active"})
    assert r.status_code == 200
```

### Pure function / unit

```python
from app.query import normalize_optional


def test_normalize_empty_string_to_missing():
    """Regression: '' was passed through and crashed downstream."""
    assert normalize_optional("") is None
    assert normalize_optional(None) is None
    assert normalize_optional("active") == "active"
```

### Timezone-sensitive test (CI vs local)

```python
from datetime import datetime, timezone

# Prefer project helpers if they exist; add freezegun only if already in use.

def test_format_date_uses_explicit_utc():
    """Regression: CI (UTC) failed when host TZ differed."""
    from app.dates import format_report_day

    dt = datetime(2024, 6, 15, 12, 0, 0, tzinfo=timezone.utc)
    assert format_report_day(dt) == "2024-06-15"
```

### Run commands

```bash
pytest tests/test_query_params.py -q
pytest tests/test_query_params.py::test_optional_filter_empty_string_does_not_500 -q
```

---

## JavaScript / TypeScript snippets

### Function unit (Jest / Vitest style)

```ts
import { normalizeOptional } from "../src/query";

test("normalizeOptional: empty string becomes missing (regression)", () => {
  // Regression: "" was passed through and crashed downstream
  expect(normalizeOptional("")).toBeNull();
  expect(normalizeOptional(null)).toBeNull();
  expect(normalizeOptional("active")).toBe("active");
});
```

### HTTP handler / supertest-style

```ts
import request from "supertest";
import { app } from "../src/app";

test("GET /items?filter= does not 500 (regression)", async () => {
  const res = await request(app).get("/items").query({ filter: "" });
  expect(res.status).not.toBe(500);
  expect([200, 400]).toContain(res.status);
});

test("GET /items without filter still 200", async () => {
  const res = await request(app).get("/items");
  expect(res.status).toBe(200);
});
```

### Run commands

```bash
npm test -- --testPathPattern=query
npx vitest run src/query.test.ts
```

---

## What to avoid

| Avoid | Why |
| --- | --- |
| Happy-path only | Does not lock the bug |
| Broad end-to-end for a one-line fix | Hard to review; slow CI |
| `sleep` to hide races | Flake remains |
| Hard-coded local paths / host TZ | Fails on CI |
| Asserting unrelated fields | Out of scope; noisy diffs |

---

## Fit in the 5-step process

1. Scope confirmed  
2. Reproduced  
3. Narrow patch  
4. **This template** — red → green regression  
5. Handoff with the exact command  

See also: [README case studies](../README.md).

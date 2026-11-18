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

### 1. HTTP / API edge case

```python
def test_optional_filter_empty_string_does_not_500():
    """Regression: empty string on optional filter used to return 500."""
    r = client.get("/items", params={"filter": ""})
    assert r.status_code != 500
    assert r.status_code in (200, 400)  # lock one behavior agreed in scope


def test_optional_filter_omitted_still_ok():
    r = client.get("/items")
    assert r.status_code == 200


def test_optional_filter_valid_value():
    r = client.get("/items", params={"filter": "active"})
    assert r.status_code == 200
```

### 2. Pure function / unit

```python
from app.query import normalize_optional


def test_normalize_empty_string_to_missing():
    """Regression: '' was passed through and crashed downstream."""
    assert normalize_optional("") is None
    assert normalize_optional(None) is None
    assert normalize_optional("active") == "active"
```

### 3. Timezone-sensitive (CI vs local)

```python
from datetime import datetime, timezone

# Prefer project helpers if they exist; add freezegun only if already in use.

def test_format_date_uses_explicit_utc():
    """Regression: CI (UTC) failed when host TZ differed."""
    from app.dates import format_report_day

    dt = datetime(2024, 6, 15, 12, 0, 0, tzinfo=timezone.utc)
    assert format_report_day(dt) == "2024-06-15"
```

### 4. Expected exception (`pytest.raises`)

```python
import pytest
from app.checkout import parse_quantity, InvalidQuantity


def test_parse_quantity_rejects_zero():
    """Regression: qty=0 used to pass and create empty orders."""
    with pytest.raises(InvalidQuantity):
        parse_quantity(0)


def test_parse_quantity_rejects_negative():
    with pytest.raises(InvalidQuantity):
        parse_quantity(-1)


def test_parse_quantity_accepts_positive():
    assert parse_quantity(3) == 3
```

### 5. Parametrize edge cases (one behavior, many inputs)

```python
import pytest
from app.query import normalize_optional


@pytest.mark.parametrize(
    "raw, expected",
    [
        ("", None),
        (None, None),
        ("   ", None),       # if scope treats whitespace as missing
        ("active", "active"),
    ],
)
def test_normalize_optional_edge_cases(raw, expected):
    """Regression: empty / blank optional values must not reach downstream."""
    assert normalize_optional(raw) == expected
```

Only include rows that match **confirmed scope**. Delete the whitespace case if the client did not ask for it.

### 6. Path handling (no hard-coded OS paths)

```python
from pathlib import Path
from app.reports import resolve_report_path


def test_resolve_report_path_under_base(tmp_path: Path):
    """Regression: absolute client paths escaped the reports directory."""
    base = tmp_path / "reports"
    base.mkdir()
    out = resolve_report_path(base, "team/q1.csv")
    assert out == base / "team" / "q1.csv"
    assert base in out.parents or out.parent == base


def test_resolve_report_path_rejects_escape(tmp_path: Path):
    base = tmp_path / "reports"
    base.mkdir()
    with pytest.raises(ValueError):
        resolve_report_path(base, "../secrets.txt")
```

Use `tmp_path` (pytest) so CI and Windows/macOS/Linux stay aligned.

### 7. Environment variable with `monkeypatch`

```python
from app.config import max_upload_mb


def test_max_upload_mb_falls_back_when_env_invalid(monkeypatch):
    """Regression: non-integer MAX_UPLOAD_MB caused startup crash."""
    monkeypatch.setenv("MAX_UPLOAD_MB", "not-a-number")
    assert max_upload_mb() == 5  # documented default


def test_max_upload_mb_reads_valid_env(monkeypatch):
    monkeypatch.setenv("MAX_UPLOAD_MB", "25")
    assert max_upload_mb() == 25
```

### 8. Off-by-one / boundary

```python
from app.page import slice_page


def test_slice_page_last_item_included():
    """Regression: last index of a full page was dropped (off-by-one)."""
    items = list(range(10))
    page = slice_page(items, offset=0, limit=10)
    assert page == items
    assert len(page) == 10


def test_slice_page_empty_when_offset_past_end():
    items = list(range(10))
    assert slice_page(items, offset=10, limit=10) == []
```

### 9. Stable ordering (flake on CI)

```python
from app.labels import unique_labels


def test_unique_labels_preserves_first_seen_order():
    """Regression: set() ordering differed between local and CI."""
    assert unique_labels(["b", "a", "b", "c"]) == ["b", "a", "c"]
```

Prefer explicit sort or “first seen” lists over relying on `set` iteration order.

### 10. External call failure (mock, no network)

```python
from unittest.mock import patch
from app.webhooks import deliver


def test_deliver_returns_false_on_timeout():
    """Regression: timeout raised and broke the request handler."""
    with patch("app.webhooks.http_post", side_effect=TimeoutError):
        assert deliver("https://example.test/hook", {"ok": True}) is False
```

Do not hit real URLs in regression tests for a narrow fix.

### Run commands

```bash
# whole file
pytest tests/test_query_params.py -q

# one test
pytest tests/test_query_params.py::test_optional_filter_empty_string_does_not_500 -q

# show values on assert failure
pytest tests/test_query_params.py -q --tb=short
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
| Real network calls | Slow, flaky, may need secrets |

---

## Fit in the 5-step process

1. Scope confirmed  
2. Reproduced  
3. Narrow patch  
4. **This template** — red → green regression  
5. Handoff with the exact command  

See also: [README case studies](../README.md).

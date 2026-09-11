# TASK-04-MATCHER-TESTS

This checkout has no receipt-to-handoff matcher implementation. The suite below is the contract. Pairing is by identity (`receipt.re == handoff.name`), not by the `YYYYMMDD-` name prefix. These tests fail against a naive date-prefix matcher.

## Matcher contract

- A **handoff** has a unique `name` (example: `20260910-alpha`).
- A **receipt** has a unique `name` and a `re` field naming exactly one handoff.
- `match(handoffs, receipts)` returns a list of `(handoff, receipt)` pairs.
- A pair is valid only when `receipt.re == handoff.name`.
- Receipt names may use a different UTC calendar date than the handoff they refer to. Date prefixes are not a key.

## Naive matcher these tests reject

```python
def naive_date_prefix_match(handoffs, receipts):
    """Wrong: join on YYYYMMDD name prefix, first handoff wins."""
    by_prefix = {}
    for h in handoffs:
        by_prefix.setdefault(h.name[:8], []).append(h)
    pairs = []
    for r in receipts:
        candidates = by_prefix.get(r.name[:8], [])
        if candidates:
            pairs.append((candidates[0], r))
    return pairs
```

That implementation misses `20260911-*` receipts whose `re` names a `20260910-*` handoff, and can bind a receipt to a same-date handoff that is not `receipt.re`.

## Property-based suite

Requires `hypothesis`. Point `match` at the implementation under test. Pointing it at `naive_date_prefix_match` must fail.

```python
from dataclasses import dataclass
from hypothesis import given, settings
from hypothesis import strategies as st


@dataclass(frozen=True)
class Handoff:
    name: str


@dataclass(frozen=True)
class Receipt:
    name: str
    re: str


DATES = ["20260909", "20260910", "20260911", "20260912"]
SLUGS = ["alpha", "beta", "gamma", "delta"]


def token(date, slug):
    return f"{date}-{slug}"


@st.composite
def worlds(draw):
    handoff_names = draw(st.lists(
        st.tuples(st.sampled_from(DATES), st.sampled_from(SLUGS)).map(lambda p: token(*p)),
        min_size=1,
        max_size=4,
        unique=True,
    ))
    handoffs = [Handoff(n) for n in handoff_names]
    n_receipts = draw(st.integers(min_value=0, max_value=5))
    receipts = []
    for i in range(n_receipts):
        r_date = draw(st.sampled_from(DATES))
        r_slug = draw(st.sampled_from(SLUGS))
        target = draw(st.sampled_from(handoff_names + [token(draw(st.sampled_from(DATES)), draw(st.sampled_from(SLUGS)))]))
        receipts.append(Receipt(name=f"{r_date}-{r_slug}-r{i}", re=target))
    return handoffs, receipts


def pair_key(pairs):
    return {(h.name, r.name, r.re) for h, r in pairs}


@given(worlds())
@settings(max_examples=80)
def test_symmetric(world):
    handoffs, receipts = world
    forward = pair_key(match(handoffs, receipts))
    reversed_inputs = pair_key(match(list(reversed(handoffs)), list(reversed(receipts))))
    assert forward == reversed_inputs


@given(worlds())
@settings(max_examples=80)
def test_idempotent(world):
    handoffs, receipts = world
    once = match(handoffs, receipts)
    assert pair_key(once) == pair_key(match(handoffs, receipts))
    if not once:
        return
    matched_h = [h for h, _ in once]
    matched_r = [r for _, r in once]
    assert pair_key(match(matched_h, matched_r)) == pair_key(once)


@given(worlds())
@settings(max_examples=80)
def test_never_returns_receipt_for_different_handoff(world):
    handoffs, receipts = world
    names = {h.name for h in handoffs}
    for h, r in match(handoffs, receipts):
        assert r.re == h.name
        assert r.re in names
```

Properties:

1. **Symmetric.** The pair set is independent of input order.
2. **Idempotent.** `match(H, R)` is stable; matching the already-paired subset returns that same subset.
3. **`re` identity.** No returned receipt has `re` naming a different handoff than the one it is paired with.

## UTC date-rollover regression

Handoff named `20260910-*`, receipt named `20260911-*`, `re` pointing at the 20260910 handoff. A correct matcher pairs them. A date-prefix matcher does not.

```python
def test_utc_date_rollover_receipt_next_day_still_binds():
    handoff = Handoff("20260910-alpha")
    receipt = Receipt(name="20260911-alpha-r0", re="20260910-alpha")
    assert match([handoff], [receipt]) == [(handoff, receipt)]


def test_utc_date_rollover_same_prefix_wrong_handoff_is_rejected():
    intended = Handoff("20260910-alpha")
    distractor = Handoff("20260911-beta")
    receipt = Receipt(name="20260911-beta-r0", re="20260910-alpha")
    pairs = match([intended, distractor], [receipt])
    assert pairs == [(intended, receipt)]
    assert all(r.re == h.name for h, r in pairs)


def test_naive_date_prefix_fails_rollover():
    """This assertion is the naive matcher’s failure mode; keep it to document the bug."""
    handoff = Handoff("20260910-alpha")
    receipt = Receipt(name="20260911-alpha-r0", re="20260910-alpha")
    naive = naive_date_prefix_match([handoff], [receipt])
    assert naive == [(handoff, receipt)]  # fails: prefixes 20260910 vs 20260911
```

## Expected naive failures

| Test | Naive date-prefix result |
|---|---|
| `test_utc_date_rollover_receipt_next_day_still_binds` | empty pair list (`20260911` ≠ `20260910`) |
| `test_utc_date_rollover_same_prefix_wrong_handoff_is_rejected` | pairs the receipt with `20260911-beta` |
| `test_naive_date_prefix_fails_rollover` | empty pair list |
| `test_never_returns_receipt_for_different_handoff` | same-date prefix, `re` names another handoff |
| `test_symmetric` / `test_idempotent` | extra or order-dependent pairs when several handoffs share a date prefix |

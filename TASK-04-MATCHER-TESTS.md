# TASK-04-MATCHER-TESTS

This checkout has no receipt-to-handoff matcher implementation. The suite below is the contract. Pairing is by identity (`receipt.re == handoff.name`), not by the `YYYYMMDD-` name prefix.

These tests **fail** if `match` is bound to a naive date-prefix join. They **pass** if `match` is bound to `re`-identity. Do not treat an empty pair list as success: a prefix matcher drops the UTC rollover case and the `re` property is then vacuously true.

## Matcher contract

- A **handoff** has a unique `name` (example: `20260910-alpha`).
- A **receipt** has a unique `name` and a `re` field naming exactly one handoff.
- `match(handoffs, receipts)` returns a list of `(handoff, receipt)` pairs.
- A pair is valid only when `receipt.re == handoff.name`.
- Receipt names may use a different UTC calendar date than the handoff they refer to. Date prefixes are not a key (`TASK-07-DATE-ROLLOVER.md`).

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

## Reference matcher these tests accept

```python
def identity_match(handoffs, receipts):
    """Correct: pair only when receipt.re == handoff.name."""
    by_name = {h.name: h for h in handoffs}
    return [(by_name[r.re], r) for r in receipts if r.re in by_name]
```

Point `match` at the implementation under test. `match = naive_date_prefix_match` must fail. `match = identity_match` must pass. `identity_match` is the predicate, not a product matcher to land here.

## Property-based suite

Requires `hypothesis`.

```python
from dataclasses import dataclass
from hypothesis import example, given, settings
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
        target = draw(st.sampled_from(
            handoff_names + [token(draw(st.sampled_from(DATES)), draw(st.sampled_from(SLUGS)))]
        ))
        receipts.append(Receipt(name=f"{r_date}-{r_slug}-r{i}", re=target))
    return handoffs, receipts


def pair_key(pairs):
    return {(h.name, r.name, r.re) for h, r in pairs}


# Forced cases the prefix matcher gets wrong (do not rely on draws alone).
ROLLOVER = (
    [Handoff("20260910-alpha")],
    [Receipt(name="20260911-alpha-r0", re="20260910-alpha")],
)
WRONG_SAME_PREFIX = (
    [Handoff("20260910-alpha"), Handoff("20260911-beta")],
    [Receipt(name="20260911-beta-r0", re="20260910-alpha")],
)
SHARED_PREFIX = (
    [Handoff("20260910-alpha"), Handoff("20260910-beta")],
    [Receipt(name="20260910-gamma-r0", re="20260910-beta")],
)


@example(ROLLOVER)
@example(WRONG_SAME_PREFIX)
@example(SHARED_PREFIX)
@given(worlds())
@settings(max_examples=80)
def test_symmetric(world):
    handoffs, receipts = world
    forward = pair_key(match(handoffs, receipts))
    reversed_inputs = pair_key(match(list(reversed(handoffs)), list(reversed(receipts))))
    assert forward == reversed_inputs


@example(ROLLOVER)
@example(WRONG_SAME_PREFIX)
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


@example(ROLLOVER)
@example(WRONG_SAME_PREFIX)
@given(worlds())
@settings(max_examples=80)
def test_never_returns_receipt_for_different_handoff(world):
    handoffs, receipts = world
    names = {h.name for h in handoffs}
    pairs = match(handoffs, receipts)
    for h, r in pairs:
        assert r.re == h.name
        assert r.re in names
    # Vacuous pass on [] is not enough: a named target in-set must be returned.
    present = names
    for r in receipts:
        if r.re in present:
            assert any(h.name == r.re and rec.name == r.name for h, rec in pairs)
```

Properties:

1. **Symmetric.** The pair set is independent of input order.
2. **Idempotent.** `match(H, R)` is stable; matching the already-paired subset returns that same subset.
3. **`re` identity.** No returned receipt has `re` naming a different handoff than the one it is paired with. If `re` names a handoff that is present, that pair must appear (so a prefix miss cannot hide as “no pairs”).

## UTC date-rollover regression

Handoff named `20260910-*`, receipt named `20260911-*`, `re` pointing at the 20260910 handoff. A correct matcher pairs them. A date-prefix matcher returns `[]`.

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
```

## Expected naive failures

| Test | Naive date-prefix result |
|---|---|
| `test_utc_date_rollover_receipt_next_day_still_binds` | empty pair list (`20260911` ≠ `20260910`) |
| `test_utc_date_rollover_same_prefix_wrong_handoff_is_rejected` | pairs the receipt with `20260911-beta` |
| `test_never_returns_receipt_for_different_handoff` `@example(ROLLOVER)` | `[]` while `re` names a present handoff |
| `test_never_returns_receipt_for_different_handoff` `@example(WRONG_SAME_PREFIX)` | pair with `20260911-beta`, `re` is `20260910-alpha` |
| `test_symmetric` `@example(SHARED_PREFIX)` | first-wins flips when handoff order reverses |
| `test_idempotent` | may pass on a wrong pair; do not treat it as sufficient alone |

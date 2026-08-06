# Golden Comment Verification Report

**Repository:** saleor__saleor-clone__lyxor__PR21__20260602
**PR:** #1 — Add voucher usage summary query
**Files evaluated:** `resolvers.py`, `filters.py`, `types/vouchers.py`, `schema.py`, `schema.graphql`, `test_vouchers.py`, `types/__init__.py`
**Methodology:** Verdicts based solely on visible diff content. Unverifiable claims default to Incorrect.

---

## Comment 1

**Golden Comment:** Searching through voucher codes joins the codes table but does not call `distinct()`. A voucher with multiple matching codes can be counted multiple times in the summary totals.

**Verdict:** Correct

**Reason:** `filter_voucher_usage_summary` applies `Q(codes__code__icontains=search)` when a `search` term is provided, traversing the `Voucher → codes` reverse foreign key. If a voucher has multiple codes matching the search term, the filtered queryset returns duplicate voucher rows — one per matching code — before reaching `.aggregate()`. No `.distinct()` call appears anywhere in `filter_voucher_usage_summary` or `resolve_voucher_usage_summary`.

**Evidence:** `saleor/graphql/discount/filters.py`
```python
def filter_voucher_usage_summary(qs, filter_data):
    if not filter_data:
        return qs
    if search := filter_data.get("search"):
        qs = qs.filter(Q(name__icontains=search) | Q(codes__code__icontains=search))
    if started := filter_data.get("started"):
        qs = filter_range_field(qs, "start_date", started)
    return qs
```
No `distinct()` call is present anywhere in the diff's queryset chain.

**Confidence:** Medium-High

---

## Comment 2

**Golden Comment:** The active and expired conditions overlap at the exact `end_date` timestamp, so a voucher ending at `now` can be counted as both active and expired.

**Verdict:** Correct

**Reason:** `active_count` uses `Q(end_date__isnull=True) | Q(end_date__gte=now)`, and `expired_count` uses `Q(end_date__lte=now)`. Both use inclusive comparisons against the same `now` value, so a voucher whose `end_date` equals `now` satisfies both conditions simultaneously.

**Evidence:** `saleor/graphql/discount/resolvers.py`
```python
active_count=Count(
    "id",
    filter=Q(start_date__lte=now)
    & (Q(end_date__isnull=True) | Q(end_date__gte=now)),
),
expired_count=Count("id", filter=Q(end_date__lte=now)),
```

**Confidence:** High

---

## Comment 3

**Golden Comment:** The resolver always counts all voucher codes and ignores the `includeInactiveCodes` filter, so callers cannot exclude inactive codes from summary totals.

**Verdict:** Correct

**Reason:** `VoucherUsageSummaryFilterInput` defines an `includeInactiveCodes` boolean field, but `filter_voucher_usage_summary` only reads `search` and `started` from `filter_data` — it never inspects `includeInactiveCodes`. `code_count=Count("codes")` and `used_count=Sum("codes__used")` aggregate all codes unconditionally, with no `is_active` filtering anywhere in the diff.

**Evidence:** `filter_voucher_usage_summary` body only branches on `search` and `started`. The test `test_voucher_usage_summary_includes_inactive_codes` sets `includeInactiveCodes: True` and observes `codeCount == 2` (including an `is_active=False` code) — but since the flag is never read, the result would be identical regardless of its value.

**Confidence:** High

---

## Comment 4

**Golden Comment:** `usage_limit` is summed after joining voucher codes, so vouchers with multiple codes have their limit counted multiple times. Remaining uses can be heavily overstated.

**Verdict:** Correct

**Reason:** All aggregate annotations (`code_count=Count("codes")`, `used_count=Sum("codes__used")`, `remaining_uses=Sum("usage_limit") - Sum("codes__used")`) are computed in a single `.aggregate()` call on a queryset joined via the `codes` relation. Combining a `Sum` over a parent-table field (`usage_limit`, on `Voucher`) with a `Sum`/`Count` over a joined child relation (`codes`) in the same aggregate causes row duplication per related code — inflating `Sum("usage_limit")` by the number of matching codes per voucher.

**Evidence:** `saleor/graphql/discount/resolvers.py`
```python
totals = qs.aggregate(
    total_count=Count("id"),
    ...
    code_count=Count("codes"),
    used_count=Sum("codes__used"),
    remaining_uses=Sum("usage_limit") - Sum("codes__used"),
)
```
No subquery isolation or `distinct` counting is used to prevent join-induced row multiplication.

**Confidence:** High

---

## Summary Table

| # | Golden Comment (short) | Verdict | Confidence |
|---|---|---|---|
| 1 | Missing `distinct()` on code search join | Correct | Medium-High |
| 2 | Active/expired boundary overlap at `now` | Correct | High |
| 3 | `includeInactiveCodes` filter ignored | Correct | High |
| 4 | `usage_limit` sum inflated by code join | Correct | High |

**Total Correct:** 4
**Total Incorrect / Partially Correct:** 0

## Overall Assessment

This is an unusually strong batch of golden comments. All four identify genuine, verifiable defects rooted in real Django ORM aggregation pitfalls (join-induced row duplication affecting `Sum`/`Count`, boundary-condition overlap in date filters, and a filter field declared in the schema but never wired into resolver logic). Each comment is specific, technically precise, and directly traceable to the diff without requiring assumptions about code outside the PR. None rely on speculative or unverifiable claims — a notably high signal-to-noise batch relative to typical review comment sets.

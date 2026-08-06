# Golden Comment Verification Report

**Repository:** saleor/saleor-clone
**PR:** #1 — Add stock summary query (PR22)
**Author:** jaykaduofficial
**Files changed:** `resolvers.py`, `types.py`, `schema.py`, `schema.graphql`, `test_stocks.py`
**Evaluation method:** Diff-only verification (no assumptions about code outside the provided PR)

---

## Golden Comment 1

> "Low-stock counting uses raw `quantity` instead of available quantity. A stock with high quantity but nearly all units allocated will not be counted as low stock."

- **Verdict:** Correct
- **Reason:** The `low_stock_count` aggregate filters directly on the `quantity` field, not on an "available" (quantity − quantity_allocated) value. A stock with, e.g., `quantity=1000` and `quantity_allocated=990` (only 10 truly available) would never trigger `low_stock_count`, even though its effective available stock is far below `low_stock_threshold`.
- **Evidence:** `saleor/graphql/warehouse/resolvers.py`
  ```python
  totals = qs.aggregate(
      total_count=Count("id"),
      total_quantity=Coalesce(Sum("quantity"), 0),
      total_allocated=Coalesce(Sum("quantity_allocated"), 0),
      low_stock_count=Count("id", filter=Q(quantity__lte=low_stock_threshold)),
  )
  ```
  The filter is `quantity__lte=low_stock_threshold` — raw `quantity`, never `quantity - quantity_allocated`.
- **Confidence:** High

---

## Golden Comment 2

> "The available total subtracts allocations but ignores active reservations, so the summary can overstate inventory available to sell."

- **Verdict:** Correct
- **Reason:** `total_available` is computed purely as `total_quantity - total_allocated`, with no reference to reservations anywhere in the resolver, the aggregate query, or the `StockSummary` type fields. The diff confirms reservations are a distinct concept from allocations in this codebase — the pre-existing `Stock` type in `schema.graphql` already exposes a separate `quantityReserved: Int!` field. Since `resolve_stock_summary` only touches `quantity` and `quantity_allocated`, any reserved-but-not-yet-allocated stock is not deducted, matching the comment's claim.
- **Evidence:** `saleor/graphql/warehouse/resolvers.py`
  ```python
  return {
      "total_count": totals["total_count"] or 0,
      "total_quantity": total_quantity,
      "total_allocated": total_allocated,
      "total_available": total_quantity - total_allocated,
      "low_stock_count": totals["low_stock_count"] or 0,
      "low_stock_threshold": low_stock_threshold,
  }
  ```
  No `quantity_reserved` or reservation reference anywhere in the new code. `schema.graphql` shows `quantityReserved` exists as a separate, pre-existing field on `Stock`, confirming reservations ≠ allocations in this domain model.
- **Confidence:** Medium-High (the arithmetic gap is directly verifiable in the diff; the only caveat is the `Stock` model definition itself is not visible, so we cannot see exactly how "reserved" is computed elsewhere — this does not affect the validity of the claim about the new resolver, which simply never references reservations).

---

## Summary Statistics

| Metric | Count |
|---|---|
| Total golden comments evaluated | 2 |
| Correct | 2 |
| Partially Correct | 0 |
| Incorrect | 0 |

## Overall Quality Assessment

Both golden comments are strong, diff-grounded observations rather than generic boilerplate. They correctly identify a real semantic gap in the new `resolve_stock_summary` logic: it conflates "quantity" with "availability" in the low-stock filter, and it only accounts for allocations (not reservations) when computing `total_available`. Both claims are precise and testable against exact lines in the aggregate query — exactly the kind of catch a competent reviewer would flag on a PR introducing inventory-summary math. High-quality golden set with no hallucinated or unverifiable claims.

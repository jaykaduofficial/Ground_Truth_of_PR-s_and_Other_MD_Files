# Golden Comment Evaluation Report

**PR:** #1 — "Add shipping zone summary query" (pr-23)
**Repository:** saleor/saleor-clone (annotated)
**Files Changed (5):**
- `saleor/graphql/shipping/tests/queries/test_shipping_zones.py`
- `saleor/graphql/shipping/resolvers.py`
- `saleor/graphql/shipping/schema.py`
- `saleor/graphql/shipping/types.py`
- `saleor/graphql/schema.graphql`

**Methodology:** Diff-only verification. Verdicts based solely on visible code changes in the provided PR. Unverifiable claims default to Incorrect.

---

## Golden Comment 1

> "Channel scoping filters through shipping method listings instead of the shipping zone's channel assignment. A zone assigned to the channel but without channel-listed methods is excluded from the summary."

- **Verdict:** Correct
- **Reason:** The resolver filters zones for the summary using a path that traverses `shipping_methods → channel_listings → channel`, rather than the zone's own `channels` relation (which is used separately for `channel_count=Count("channels")`). A `ShippingZone` with a channel assigned directly via `zone.channels` but no channel-listed shipping methods for that channel would not match the filter and would be silently excluded from the summary.
- **Evidence:**
  ```python
  # saleor/graphql/shipping/resolvers.py
  def resolve_shipping_zone_summary(info, channel_slug=None):
      qs = models.ShippingZone.objects.using(get_database_connection_name(info.context))
      if channel_slug:
          qs = qs.filter(shipping_methods__channel_listings__channel__slug=channel_slug)
  ```
  Compare to `channel_count=Count("channels")` in the same aggregate block, which references a different relation than the one used for filtering.
- **Confidence:** High

---

## Golden Comment 2

> "The aggregate counts are performed after joins introduced by channel filtering, so zones and methods can be counted multiple times unless distinct counts are used."

- **Verdict:** Correct
- **Reason:** When `channel_slug` is supplied, the queryset is built via a join across `shipping_methods__channel_listings__channel__slug`. If a zone has more than one shipping method (or a method has more than one matching channel listing), the join produces multiple rows for the same zone. The subsequent `.aggregate()` call uses plain `Count()` calls with no `distinct=True`, so these counts are vulnerable to inflation whenever the channel filter is applied. Added tests only exercise a single zone with a single matching method/channel listing, so this path is uncovered.
- **Evidence:**
  ```python
  # saleor/graphql/shipping/resolvers.py
  totals = qs.aggregate(
      total_count=Count("id"),
      default_count=Count("id", filter=Q(default=True)),
      method_count=Count("shipping_methods"),
      channel_count=Count("channels"),
  )
  ```
  No `distinct=True` anywhere in these `Count()` calls.
- **Confidence:** High

---

## Golden Comment 3

> "Country counting iterates over the joined queryset directly. If the queryset contains duplicate zone rows from joins, the same zone's countries are counted more than once."

- **Verdict:** Correct
- **Reason:** `country_count` is computed by iterating `qs` in Python (`sum(len(zone.countries) for zone in qs)`), using the same `qs` potentially duplicated by the channel-filter join (Comment 2). The loop does not deduplicate by zone `id` (no `.distinct()` or dict/set keyed by pk), so any zone appearing multiple times in `qs` due to the join has its `countries` field summed multiple times, inflating `country_count`.
- **Evidence:**
  ```python
  # saleor/graphql/shipping/resolvers.py
  if channel_slug:
      qs = qs.filter(shipping_methods__channel_listings__channel__slug=channel_slug)
  ...
  country_count = sum(len(zone.countries) for zone in qs)
  ```
  No `.distinct()` call is present on `qs` at any point before this line.
- **Confidence:** High

---

## Summary Statistics

| # | Golden Comment (short) | Verdict | Confidence |
|---|---|---|---|
| 1 | Channel scoping via method listings excludes valid zones | Correct | High |
| 2 | Aggregate counts inflated by join duplication | Correct | High |
| 3 | Country count inflated by join duplication (Python loop) | Correct | High |

- **Total Correct:** 3
- **Total Incorrect / Partially Correct:** 0

## Overall Quality Assessment

These three golden comments are strong and technically well-grounded. They all trace a single root cause — the channel-scoping join (`shipping_methods__channel_listings__channel__slug`) in `resolve_shipping_zone_summary` — through to its distinct downstream consequences: (1) incorrect filtering semantics relative to the zone's actual channel assignment, (2) inflated SQL-level aggregate counts, and (3) inflated Python-level country counts. All three are precisely supported by the diff in `resolvers.py` with no reliance on code outside the PR. Notably, the PR's own added tests (`test_shipping_zone_summary`, `test_shipping_zone_summary_with_channel`) only cover a single-zone, single-method scenario, so none of these bugs would be caught by the PR's test suite as written — reinforcing that these are legitimate, currently-unverified findings.

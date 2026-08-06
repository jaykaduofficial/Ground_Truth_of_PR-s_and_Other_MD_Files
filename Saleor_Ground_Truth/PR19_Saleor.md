# Golden Comment Evaluation Report

**PR:** #1 — Add latest order lookup by customer email
**Repo:** saleor__saleor-clone__lyxor__PR19__20260602
**Author:** jaykaduofficial
**Files changed:** 6
**Evaluation method:** Diff-only verification (no assumptions about code outside the visible PR diff)

---

## Golden Comment 1

> "This new order lookup is exposed as a BaseField without declaring MANAGE_ORDERS permissions. It allows callers to query orders by customer email through a top-level field without the same permission gate used by the existing external-reference lookup."

**Verdict:** Partially Correct

**Reason:** The diff shows the new field definition in `schema.py`:

```python
latest_order_by_customer_email = BaseField(
    Order,
    description=(...),
    email=graphene.Argument(graphene.String, ..., required=True),
    channel=graphene.Argument(graphene.String, ...),
    doc_category=DOC_CATEGORY_ORDERS,
)
```

No `permissions=` argument appears anywhere in this hunk, so on its face the claim is accurate. However, the accompanying test (`test_query_latest_order_by_customer_email`) explicitly passes `permissions=[permission_manage_orders]` to `staff_api_client.post_graphql(...)`, which is the standard Saleor test pattern used **only when a field enforces a permission check**. This suggests a permission gate likely exists somewhere (e.g., a resolver-level check or class-level `permission_classes`) that simply isn't included in this diff. Since the full `schema.py` file isn't visible and we can't confirm or deny a hidden permission mechanism, we can't fully validate "no permission gate exists" — only that none is *visible* in the changed lines.

**Evidence:** `saleor/graphql/order/schema.py` (new `BaseField` block, lines ~198–214); `test_order.py` test using `permissions=[permission_manage_orders]`.

**Confidence:** Medium

---

## Golden Comment 2

> "The resolver builds an unrestricted Order queryset directly instead of using resolve_orders, so it bypasses channel-access filtering for staff users and apps. A caller can request orders from channels they should not be able to access."

**Verdict:** Correct

**Reason:** The new resolver in `resolvers.py` builds the queryset directly from `models.Order.objects`, applying only `non_draft()` and an optional `channel__slug` filter if `channel_slug` is explicitly passed — there is no channel-permission/visibility filtering applied for the requesting staff user or app, and `resolve_orders` (the existing, presumably permission-aware resolver used elsewhere) is never called or referenced. If `channel` isn't supplied, the query searches across **all channels** with no access restriction shown in the diff.

**Evidence:**

```python
def resolve_latest_order_by_customer_email(info, email, channel_slug=None):
    database_connection_name = get_database_connection_name(info.context)
    qs = models.Order.objects.using(database_connection_name).non_draft()
    if channel_slug:
        qs = qs.filter(channel__slug=str(channel_slug))
    order = filter_customer_email(qs, email).order_by("-created_at").first()
```
`saleor/graphql/order/resolvers.py`

**Confidence:** High

---

## Golden Comment 3

> "The email lookup uses icontains rather than validating and matching the full email address. This enables broad enumeration-style searches such as domain fragments and can return another customer's latest order."

**Verdict:** Correct

**Reason:** `filter_customer_email` is implemented with `icontains`, not an exact match, and the PR's own test explicitly demonstrates domain-fragment matching returning another customer's order.

**Evidence:**

```python
def filter_customer_email(qs, value):
    if not value:
        return qs
    return qs.filter(Q(user_email__icontains=value) | Q(user__email__icontains=value))
```
`saleor/graphql/order/filters.py`. Confirmed by the test `test_query_latest_order_by_customer_email_with_fragment`, where `email = "example.com"` (a bare domain, not a full address) successfully matches `order.user_email == "customer@example.com"`.

**Confidence:** High

---

## Golden Comment 4

> "The customerEmailDomain resolver derives data from user_email without applying the existing userEmail permission and obfuscation rules. It leaks customer email-derived information even in contexts where the full email would normally be protected."

**Verdict:** Partially Correct

**Reason:** The new resolver directly reads `root.node.user_email` and returns the domain with no permission check, no call to any obfuscation helper, and no `@one_of_permissions_required`-style decorator visible:

```python
@staticmethod
def resolve_customer_email_domain(root, _info):
    email = root.node.user_email
    if not email and root.node.user_id:
        return None
    return email.split("@")[-1] if email else None
```

This is a fair concern — unlike the neighboring `_resolve_user_email` (referenced in the diff context just above/below it, implying it has separate handling logic), this new resolver contains no equivalent gating. However, the diff does **not** include the actual body of `_resolve_user_email`/`resolve_user_email`, so we can't confirm exactly what "existing permission and obfuscation rules" consist of, or definitively prove they're being bypassed rather than simply not needed (a domain is arguably less sensitive than a full email). The core technical claim (no visible permission/obfuscation logic on the new resolver) holds, but the comparative claim about what the existing `userEmail` rules actually do is unverifiable from this diff alone.

**Evidence:** `saleor/graphql/order/types.py`, new `resolve_customer_email_domain` method (lines ~2523–2530); adjacent `_resolve_user_email` reference not fully shown in diff.

**Confidence:** Medium

---

## Summary Statistics

| # | Golden Comment (short) | Verdict | Confidence |
|---|---|---|---|
| 1 | Missing MANAGE_ORDERS permission gate | Partially Correct | Medium |
| 2 | Unrestricted queryset bypasses channel filtering | Correct | High |
| 3 | `icontains` enables email enumeration | Correct | High |
| 4 | `customerEmailDomain` leaks data without permission rules | Partially Correct | Medium |

- **Total Correct:** 2
- **Total Incorrect / Partially Correct:** 2 (0 Incorrect, 2 Partially Correct)

## Overall Quality Assessment

This is a strong, high-signal set of golden comments — all four flag genuine, verifiable security/design issues rather than stylistic nitpicks, and all are grounded in code actually changed by the PR (no hallucinated file references). Comments 2 and 3 are cleanly and fully verifiable from the diff alone (missing channel-access filtering, and `icontains`-based enumeration, the latter even self-demonstrated by the PR's own test). Comments 1 and 4 are directionally correct but rely partly on comparisons to code *outside* the diff (the "existing external-reference lookup" permission gate, and the "existing userEmail permission and obfuscation rules") that isn't visible here — so while the underlying technical observation in the diff holds, the comparative framing can't be fully confirmed under strict diff-only verification. Overall this looks like a well-constructed golden set focused on a coherent theme (authorization/data-exposure gaps in a new customer-lookup feature), with realistic and specific evidence rather than generic boilerplate concerns.

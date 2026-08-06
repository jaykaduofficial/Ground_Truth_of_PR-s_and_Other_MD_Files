# Golden Comment Evaluation Report

**PR:** #1 — "Add page excerpt fields"
**Repo:** saleor/saleor-clone (annotated PR set)
**Source file:** saleor-clone__lyxor__PR25__20260602.pdf
**Branch:** `pr-25` → `golden-pr-10`

---

## Comment 1: Private metadata leak via public excerpt field

> The public excerpt field can fall back to private metadata without checking MANAGE_PAGES or private metadata permissions, which can leak private CMS content to storefront/API clients.

**Verdict:** Correct

**Reason:** The `excerpt` field is exposed as a public `graphene.String` field on the `Page` type with no permission decorator visible anywhere in the diff (it's a brand-new field, so any permission check would have to appear in this same diff). Its resolver calls `get_page_excerpt()`, which checks `page.metadata` first and, if empty, silently falls back to `page.private_metadata` before returning the excerpt to the caller — with no gating on `MANAGE_PAGES` or similar.

**Evidence:**

`saleor/page/utils.py`:
```python
metadata_excerpt = page.metadata.get(PAGE_EXCERPT_METADATA_KEY)
if not metadata_excerpt:
    metadata_excerpt = page.private_metadata.get(PAGE_EXCERPT_METADATA_KEY)
if metadata_excerpt:
    return _truncate_excerpt(str(metadata_excerpt).strip(), max_length)
```

`saleor/graphql/page/types.py`, `resolve_excerpt` — plain `@staticmethod`, no permission check, feeding directly into the public `excerpt` field defined in the same diff hunk (`types.py` ~lines 179–190, `schema.graphql` ~lines 13219–13227).

**Confidence:** High

---

## Comment 2: Excerpt can exceed `maxLength` for small values

> For maxLength values below 3, the returned excerpt can be longer than the requested limit because the suffix is always appended after slicing with a negative index.

**Verdict:** Correct

**Reason:** `_truncate_excerpt` slices with `value[: max_length - 3]` and unconditionally appends `"..."`. When `max_length < 3`, `max_length - 3` is negative, so the slice removes characters from the *end* rather than producing a short prefix, and the appended ellipsis adds 3 more characters — the result can end up far longer than `max_length`. E.g. for `max_length=1`, `value[:-2]` keeps almost the whole string, then `"..."` is added on top.

**Evidence:**

`saleor/page/utils.py`:
```python
def _truncate_excerpt(value: str, max_length: int) -> str:
    if len(value) <= max_length:
        return value
    return f"{value[: max_length - 3].rstrip()}..."
```

No lower-bound guard on `max_length` exists anywhere in the diff, and the GraphQL schema only enforces `PositiveInt` (≥1), not ≥3.

**Confidence:** High

---

## Comment 3: N+1 query pattern on `pageTypeName` resolver

> Resolving pageTypeName performs a separate PageType query for every Page node, creating an N+1 query pattern on page listings instead of reusing the existing PageTypeByIdLoader.

**Verdict:** Correct

**Reason:** The new `resolve_page_type_name` resolver issues a direct synchronous ORM `.get()` call per `Page` node, whereas the pre-existing `resolve_page_type` resolver on the same class uses `PageTypeByIdLoader` (a batching dataloader) for the equivalent lookup. Since `pageTypeName` is on `Page` and would be resolved per-node in a listing query (e.g. `PAGES_QUERY_WITH_EXCERPTS` added in the tests), this produces one query per page instead of one batched query.

**Evidence:**

`saleor/graphql/page/types.py`:
```python
@staticmethod
def resolve_page_type_name(root: ChannelContext[models.Page], info: ResolveInfo):
    return (
        models.PageType.objects.using(get_database_connection_name(info.context))
        .get(pk=root.node.page_type_id)
        .name
    )
```

Compared directly above it in the same diff:
```python
def resolve_page_type(root: ChannelContext[models.Page], info: ResolveInfo):
    return PageTypeByIdLoader(info.context).load(root.node.page_type_id)
```

**Confidence:** High

---

## Summary

| # | Golden Comment | Verdict | Confidence |
|---|-----------------|---------|-----------|
| 1 | Private metadata leak via public excerpt field | Correct | High |
| 2 | maxLength < 3 truncation bug | Correct | High |
| 3 | N+1 query on pageTypeName resolver | Correct | High |

**Total correct:** 3
**Total incorrect / partially correct:** 0

### Overall Quality Assessment

All three golden comments are well-grounded, diff-verifiable findings — this is a strong batch. Each points to a distinct, real issue directly traceable to added code in this PR (an authorization/data-exposure gap, an off-by-construction truncation bug, and a performance regression from bypassing an existing dataloader), and none require assumptions about code outside the shown diff. No fabricated or unverifiable claims were present.

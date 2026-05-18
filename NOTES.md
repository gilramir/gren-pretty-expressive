# NOTES

Deferred design decisions and known optimization opportunities. None of these
are bugs or correctness issues — they are places where the library could be
faster or more ergonomic, written down so they're not forgotten and so
revisiting them is a paste-and-implement away.

---

## Deferred: general structural-equality dedup in `choice`

### What's there today

`flatten`'s `DocChoice` arm dedupes identical branches:

```gren
DocChoice { a = d1, b = d2 } ->
    let fa = flatten d1; fb = flatten d2 in
    if docStructEqual fa fb then fa else choice fa fb
```

This catches the hot pattern `flatten (group d) = flatten (choice d (flatten
d)) = choice (flatten d) (flatten d)`, which arises naturally when a `group` is
embedded inside another `group`'s flat branch. The dedup is invisible to
correctness — both branches resolve to the same measure set — but it cuts the
renderer's wasted work in half at every level of group nesting.

The `choice` smart constructor itself does *not* dedupe. It only collapses on
`DocFail`:

```gren
choice d1 d2 =
    when { d1 = d1, d2 = d2 } is
        { d1 = DocFail, d2 = _ } -> d2
        { d1 = _, d2 = DocFail } -> d1
        _ -> DocChoice { a = d1, b = d2, nlCnt = ... }
```

### Why not dedupe in `choice` too

A foot-gun remains: user code that builds `P.choice d d` directly — for
example, `group d` where `d` happens to be already-flat (no soft newlines or
nested choices), so `flatten d == d` and `group d = choice d d` — produces a
`DocChoice` with two identical branches that the renderer evaluates twice.

The straightforward fix is to extend the `choice` smart constructor with the
same `docStructEqual` check:

```gren
_ ->
    if docStructEqual d1 d2 then
        d1
    else
        DocChoice { a = d1, b = d2, nlCnt = ... }
```

`docStructEqual` already exists, exposed for use inside `flatten`. Reusing it
here is mechanically trivial.

### What's holding it back

Cost. `docStructEqual` is `O(|d1| + |d2|)` worst case (a structural walk of
both subtrees). For workloads with many `choice` calls — most non-trivial
documents — every call would do a structural compare, almost always
returning `False`, almost always wasted. With `N` choice nodes spanning trees
of average size `S`, the overhead is `O(N · S)` at construction.

In contrast, `flatten`'s dedup fires in a context where the two branches were
*just* recomputed by recursive flatten calls; the structural compare is on
already-walked subtrees, and the case where they match is common enough
(every flattened group) to dominate the check cost.

### When to revisit

Add the dedup to `choice`'s smart constructor if profiling shows a real
workload hitting the `P.choice d d` foot-gun — typically, a user-built doc
where some branch happens to be already-flat and the construction pattern
naturally produces a symmetric choice. The fix is two lines of code; the
question is purely whether the up-front compare cost is worth it for the
specific workload.

A cheaper compromise, if needed, is a *shallow* equality fast path: dedup
when both branches are `DocText` with equal contents, or both are the same
`DocShared` id. That catches the common trivial-already-flat case without
walking deep subtrees. Implementation hint: pattern-match on the two-branch
shape directly in `choice` before falling back to the full `docStructEqual`.

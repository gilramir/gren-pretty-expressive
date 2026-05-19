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

---

## Deferred: remaining stack-safety hot spots

Three recursive code paths could in principle overflow Node's call stack
on synthetic or pathological inputs. None has been observed in real
workloads (including the gren-format formatter on a 4485-line embedded-
JSON file, the original repro). They share the same shape as the
already-fixed `resolve` / `flatten` / `docStructEqual` DocConcat arms
(left-spine recursion bounded by chain depth), and the same fix
technique (unroll the spine via `unrollLeftConcatChain`, then iterate)
would apply mechanically.

### `resolve`'s `DocChoice` arm

```gren
DocChoice { a = d1, b = d2 } ->
    let
        useReverseOrder = nlCntOf d1 < nlCntOf d2
        ...
        r1 = self firstD c i memo
        r2 = self secondD c i r1.memo
    in
    ...
```

Recursion depth equals the *nesting depth* of explicit `P.choice` (or
`P.group`) nodes. Idiomatic user code keeps this shallow because
Pareto pruning bounds the frontier — but `doTwoColumns` constructs a
deeply nested `DocChoice` tree internally (see below), which is the
realistic path to hitting this in practice.

Fix shape: same as the DocConcat unroll, but the iteration would need
to merge measure sets across all branches rather than appending them.
Materially more complex than the DocConcat case.

### `doTwoColumns`' choice tree

The separator search builds a `DocChoice` tree of depth roughly
`rows * measures-per-row`. The resolver then walks that tree via the
DocChoice arm above, so a table with thousands of rows could overflow.
Real-world tables (record-field alignment, struct definitions, oncall
schedules) almost never approach this size — but a generated-code
caller could.

Two fix options:
  1. Build the choice tree *balanced* rather than spine-leaning. The
     existing construction recurses through `loopLimit` and
     `innerLoop`; both produce left-leaning choice chains.
  2. Convert the separator search to an iterative loop that maintains
     the running-best measure set directly, skipping the choice-tree
     construction entirely. This is the cleaner long-term design and
     would also let us share more across separator candidates.

### `resolve` / `flatten` on wrapper chains

`DocNest`, `DocAlign`, `DocReset`, and `DocAddCost` each recurse into
their single child. A user-built chain like
`P.nest 4 (P.nest 4 (P.nest 4 (...)))` 5000 levels deep would
overflow. No idiomatic Gren code produces such a chain — `nest`
typically wraps one logical block at a time, and the smart
constructors don't auto-collapse adjacent wrappers — but a code
generator targeting the API could conceivably build them.

Fix shape: a tail-recursive "unwrap chain" helper analogous to
`unrollLeftConcatChain`, returning the leaf doc plus an accumulator
of (indent | align | reset | cost-tag) entries to re-apply.
Significantly more variant cases than the DocConcat unroll; only
worth doing if a real workload reports it.

### When to revisit

The same trigger as #1: a real workload reports stack overflow on
input the recursive form can't handle. The fixes are mechanical
extensions of patterns already in the codebase. Until then, the
existing `StackSafety` test suite covers the cases that have actually
been hit in practice.

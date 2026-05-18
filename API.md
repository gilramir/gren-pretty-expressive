# PrettyExpressive — API reference for LLMs

A Gren port of the *Pretty Expressive* optimal pretty-printer (Sorawee
Porncharoenwase et al., ICFP 2023). Given a page width and a cost model,
the renderer searches all possible layouts and returns the lowest-cost
one. This file is a dense reference; for prose explanations read the doc
comment in `src/PrettyExpressive.gren`.

## Imports

    import PrettyExpressive as P
    import PrettyExpressive.Builder as B

The Builder import is **mandatory** even for documents that don't use
sharing — every rendering entry point takes `B.Builder (P.Doc cost)`.

## Setup

    cf : P.CostFactory P.DefaultCostTuple
    cf =
        P.defaultCostFactory { pageWidth = 80, computationWidth = Nothing }

`computationWidth = Nothing` defaults to `1.2 * pageWidth`. Set it
explicitly if you want a different limit beyond which the renderer
collapses to a single representative measure (`isTainted = True`).

## Mental model

A `Doc cost` is an **abstract description** of a piece of formatted text.
Leaves are `text`/newline atoms; internal nodes are concatenations,
choices, indentation wrappers, and cost annotations. The renderer walks
the tree, threading a current column `c` and indentation level `i`,
produces a Pareto-optimal set of `(last_column, cost, layout_fragments)`
candidates, then picks the cheapest one. Failure (`Nothing`) only happens
when every layout branch is unreachable.

You never write costs while constructing; the `CostFactory` does that. You
just describe what's possible (`choice`) and what's required (`hardNl`).

## Combinator table

| Use case | Combinator |
|---|---|
| Literal token | `P.text "if"` |
| Empty doc | `P.empty` |
| Space, comma, brackets | `P.space`, `P.comma`, `P.lparen`, `P.rparen`, `P.lbrace`, `P.rbrace`, `P.lbrack`, `P.rbrack`, `P.dquote` |
| Soft break (→ space when flat) | `P.nl` |
| Soft break (→ "" when flat) | `P.breakDoc` |
| Mandatory line break | `P.hardNl` |
| Custom flat substitute | `P.newline (Just ", ")` |
| Concatenate two docs | `P.concat a b` |
| Concatenate an array | `P.concat_array [a, b, c]` |
| Custom fold | `P.foldDoc f [docs]` |
| Try flat first, fall back broken | `P.group d` |
| Pick the cheaper of two layouts | `P.choice a b` |
| Indent block by N | `P.nest n d` |
| Hang at current column | `P.align d` |
| Reset indent to 0 | `P.reset d` |
| Attach extra cost | `P.addCost c d` |
| Aligned concat (`concat a (align b)`) | `P.alignedConcat a b` |
| Concat with mandatory newline between | `P.hardNlConcat a b` |
| Flatten left, then aligned concat | `P.flattenAlignedConcat a b` |
| Stack vertically (hard newlines) | `P.vcat [docs]` |
| Stack horizontally (flatten-aligned) | `P.hcat [docs]` |
| Two-column table | `P.twoColumns [{ a, b }, ...]` |
| Unreachable layout marker | `P.failDoc` |

All of the above are **pure** functions returning `Doc cost`. The smart
constructors normalize at build time (text fusion, fail propagation,
cost-annotation floating, drop-nests on text/align/reset).

## Sharing — when and how

Two references to the same `Doc` value in the tree don't automatically
share work — Gren has no value identity. To get the DAG-style memoization
the paper describes, wrap repeated sub-documents with `share`:

    P.share : P.Doc cost -> B.Builder (P.Doc cost)

The shared sub-doc resolves once per `(column, indent)` it's visited
from; subsequent visits at the same position hit the memo cache.

### Pattern: one shared sub-doc

    shared : B.Builder (P.Doc cost)
    shared =
        P.share (P.text "exit();")
            |> B.map (\exitD ->
                P.choice
                    (P.concat P.space exitD)
                    (P.nest 4 (P.concat P.nl exitD))
            )

### Pattern: multiple shared sub-docs

    twoShares : B.Builder (P.Doc cost)
    twoShares =
        P.share leftSubdoc
            |> B.andThen (\l ->
                P.share rightSubdoc
                    |> B.map (\r ->
                        ... build doc using l and r ...
                    )
            )

### When sharing actually helps

- Only when the shared value is **referenced more than once** in the
  resulting tree. One reference: zero benefit.
- The references must be the same Gren binding, e.g. one `let exitD =
  ...` used twice. Two independent `P.share (P.text "exit();")` calls
  produce two different ids.
- The win scales with (a) the size of the shared subtree and (b) the
  number of choice branches that visit it. Shared leaves like `text "x"`
  rarely pay for themselves.

### When you don't need sharing

If no subtree is repeated, wrap with `B.return` and move on:

    doc : P.Doc cost
    doc = P.concat (P.text "hello") (P.text " world")

    builder : B.Builder (P.Doc cost)
    builder = B.return doc

The renderer with an empty memo cache behaves the same as a pre-DAG tree
walk.

## Rendering

| Function | Returns | Notes |
|---|---|---|
| `P.prettyFormat cf builder` | `Maybe String` | Most common |
| `P.prettyFormatAt cf initC builder` | `Maybe String` | Output begins at column `initC` |
| `P.prettyFormatInfo cf builder` | `Maybe { string, info }` | Adds `isTainted`, `cost` |
| `P.prettyFormatInfoAt cf initC builder` | `Maybe { string, info }` | |
| `P.prettyFormatDebug cf builder` | `Maybe String` | Adds column ruler + cost line |
| `P.prettyFormatDebugAt cf initC builder` | `Maybe String` | |

`Nothing` means the document is genuinely unrenderable — typically because
every `choice` branch contains a `hardNl` inside a context that requires
flattening, or because the user inserted `failDoc` everywhere. For
documents built from ordinary constructors this never happens.

## Common building blocks

### Function call: name + arguments that may wrap

    callDoc : String -> Array (P.Doc cost) -> P.Doc cost
    callDoc name args =
        P.alignedConcat (P.text (name ++ "("))
            (P.concat (P.group (foldArgs args)) (P.text ")"))

    foldArgs args =
        P.foldDoc
            (\a b -> P.concat a (P.concat (P.text ",") (P.concat P.nl b)))
            args

### Indented block

    blockDoc : P.Doc cost -> P.Doc cost -> P.Doc cost
    blockDoc header body =
        P.concat header
            (P.concat
                (P.nest 4 (P.concat P.hardNl body))
                (P.concat P.hardNl (P.text "}"))
            )

### If/then/else with optional flat form

    ifDoc : P.Doc cost -> P.Doc cost -> P.Doc cost -> P.Doc cost
    ifDoc cond thenD elseD =
        P.group
            (P.concat (P.text "if ")
                (P.concat cond
                    (P.concat (P.concat P.nl (P.text "then "))
                        (P.concat thenD
                            (P.concat (P.concat P.nl (P.text "else "))
                                elseD)))))

### Record-style two-column table

    P.twoColumns
        [ { a = P.text "name",  b = P.text " : String" }
        , { a = P.text "age",   b = P.text " : Int" }
        , { a = P.text "email", b = P.text " : Maybe String" }
        ]

The renderer searches for the best column-separator column. Single-row
input degenerates to `P.align (P.concat a b)`; empty input is `P.empty`.

### Stacked declarations

    P.vcat
        [ P.text "module Foo exposing (..)"
        , P.text ""
        , P.text "import Bar"
        , P.text ""
        , bodyDoc
        ]

## Gotchas

- **`P.text` must not contain `\n`.** Use `P.hardNl` between segments.
- **`Builder.return` is required at the entry point.** Forgetting it is a
  type error: entry points take `B.Builder (P.Doc cost)`, not
  `P.Doc cost`.
- **Sharing is only useful for multi-reference subtrees.** Wrapping a
  one-use sub-document with `share` adds a memo lookup with no benefit.
- **Don't construct `DocContext`, `DocEvaled`, or `DocBlank` directly.**
  These are internal renderer nodes used by `twoColumns`; user-facing
  code should never need them. `flatten` of these returns `failDoc`.
- **Cost-annotation floating:** `concat`, `nest`, `align`, `reset` lift
  `DocAddCost` outward. This is a normalization, not a bug — the tree
  printer (`PrettyExpressive.Debug.debugShowDoc`) may show a different
  shape than you literally constructed.
- **`hardNl` inside `group`:** `group d = choice d (flatten d)`, and
  `flatten` on a tree containing `hardNl` produces `failDoc`. The
  flattened branch is pruned, so the group degrades silently to the
  unflattened form. This is usually what you want, but be aware.
- **Tainted output:** if `c` or `i` exceeds `cf.limit` during
  resolution, the result is marked `isTainted = True` and only one
  representative layout is tracked beyond that point. The output is
  still valid, just potentially suboptimal. Increase `computationWidth`
  to push this limit out.
- **Choice ordering:** when two branches have equal cost, `choice a b`
  prefers `a`. The renderer evaluates the higher-newline branch first
  (heuristic) but tie-breaking still favours the first argument.

## Cost model (default)

`P.DefaultCostTuple = { badness : Int, columnOverflow : Int, height : Int }`.

Lexicographic comparison: `badness` first, then `columnOverflow`, then
`height`. The renderer thus prefers (in order):

1. Layouts that don't overflow the page width.
2. Layouts that don't overflow column separators in `twoColumns`.
3. Layouts with fewer newlines.

`badness` is the sum of squared overflows past `pageWidth`, so one big
overflow is penalised more than several small ones.

For a custom cost type, implement the `P.CostFactory cost` record:

    type alias CostFactory cost =
        { text : Int -> Int -> cost           -- text at col `c`, len `l`
        , newline : Int -> cost               -- newline + `i` indent spaces
        , combine : cost -> cost -> cost      -- associative
        , le : cost -> cost -> Bool           -- total order
        , twoColumnsBias : Int -> cost        -- per-separator penalty
        , twoColumnsOverflow : Int -> cost    -- per-column-overflow penalty
        , limit : Int                         -- taint threshold
        , stringOfCost : cost -> String       -- for debug output
        , debugFormat : String -> Bool -> String -> String
        }

Required invariants (see `CostFactory` docs for the full list): `le` is
total, `combine` is associative with identity `text 0 0`, and `le` is
monotonic under `combine`.

## Decision flowchart

- Need a literal? → `P.text`.
- Need a space-or-newline? → `P.nl` (usually inside `P.group`).
- Need a guaranteed line break? → `P.hardNl`.
- Need to indent the inside of a block? → `P.nest n`.
- Want to anchor child newlines to the current column? → `P.align`.
- Have two layout options, want the cheaper one? → `P.choice`.
- Want "fit on one line if possible, otherwise break"? → `P.group`.
- Have a list of items to stack vertically? → `P.vcat`.
- Have a list of items to lay out horizontally? → `P.hcat`.
- Have a table with two columns? → `P.twoColumns`.
- Same subtree used twice and the subtree is non-trivial? → `P.share`.
- Want to bias the optimizer? → `P.addCost`.
- Need to render? → `P.prettyFormat cf (B.return doc)` (or with `share`,
  `P.prettyFormat cf builder`).

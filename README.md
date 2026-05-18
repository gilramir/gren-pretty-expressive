# Pretty Expressive Printer

A Gren implementation of the Pretty Expressive Printer, ported
from the OCaml implementation.

This library implements an *optimal* pretty printer. Given a page-width limit
and a cost model, it searches all possible layouts and selects the one with
the lowest cost. Documents are constructed from combinators and rendered with
`prettyFormat` or `prettyFormatInfo`.

Basic usage:

    import PrettyExpressive as P
    import PrettyExpressive.Builder as B

    -- Create a CostFactory, for measuring optimality ("prettiness")
    cf : P.CostFactory P.DefaultCostTuple
    cf =
        P.defaultCostFactory { pageWidth = 80, computationWidth = Nothing }

    -- Create the document (sequences of pretty-printing directives)
    document : P.Doc cost
    document =
        P.text "hello" |> P.concat (P.text " world")

    -- Render the document to a String. Entry points take a `Builder`;
    -- wrap a pure Doc with `B.return` when sharing is not needed.
    printed : Maybe String
    printed =
        P.prettyFormat cf (B.return document)

**Sharing sub-documents:** When the same sub-document is referenced from
multiple parents — typically two branches of a `choice` — the renderer can
resolve it once per `(column, indent)` position and reuse the result.
OCaml gets this for free from value identity; Gren has no equivalent, so
sharing is opt-in via `share`. `share : Doc cost -> Builder (Doc cost)`
stamps a unique id onto a sub-document; every reference participates in
the renderer's memo cache.

    sharedExample : B.Builder (P.Doc P.DefaultCostTuple)
    sharedExample =
        P.share (P.text "exit();")
            |> B.map
                (\exitD ->
                    -- Two references → one resolution per (c, i).
                    P.choice
                        (P.concat P.space exitD)
                        (P.nest 4 (P.concat P.nl exitD))
                )

For documents with no repeated sub-trees, `B.return document` is all you
need and the renderer behaves identically to a tree walk. See the docs in
`PrettyExpressive` for the full Builder API and when sharing pays off.

# Notes

The ACM article describing the algorithm presents only a few Doc constructors.
They are the most important:
* text
* nl
* concat
* nest
* group

In the article, the CostFactory has a very small, simple interface.

The OCaml implementation provides many more constructors, and its CostFactory
has a much more complicated interface, to support these new constructors.
This Gren package provides all the same constructors as the OCaml version does, and
has the same more-complicated CostFactory interface.

This Gren package also adds another constructor, for convenience:
* concat\_array

# References
* [ACM Reference Page](https://dl.acm.org/doi/abs/10.1145/3622837)
* [ACM PDF](https://arxiv.org/pdf/2310.01530) from October 2023

# Other implementations
* [OCaml information](https://sorawee.github.io/pretty-expressive-ocaml/pretty_expressive/index.html)
* [OCaml sources](https://github.com/sorawee/pretty-expressive-ocaml/tree/main)
* [Haskell](https://github.com/ennocramer/floskell) in the src/Floskell directory
* [Racket](https://docs.racket-lang.org/pretty-expressive/index.html)
* [Rust information](https://docs.rs/pretty-expressive/latest/pretty_expressive/)
* [Rust sources](https://git.midna.dev/mjm/mjl) in the pretty-expressive directory

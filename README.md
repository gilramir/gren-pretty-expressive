# Pretty Expressive Printer for Gren

A Gren implementation of the Pretty Expressive Printer, ported
from the OCaml implementation.

This library implements an *optimal* pretty printer. Given a page-width limit
and a cost model, it searches all possible layouts and selects the one with
the lowest cost. Documents are constructed from combinators and rendered with
`prettyFormat` or `prettyFormatInfo`.

Basic usage:

    import PrettyExpressive as P

    -- Create a CostFactory, for measuring optimality ("prettiness")
    cf : P.CostFactory P.DefaultCostTuple
    cf =
        P.defaultCostFactory { pageWidth = 80, computationWidth = Nothing }

    -- Create the document (sequences of pretty-printing directives)
    document : P.Doc
    document =
            (P.text "hello" |> P.concat (P.text " world"))

    -- Render the document to a String
    printed : Maybe String
    printed =
        P.prettyFormat cf document

**Gren-specific design note:** The OCaml original used per-node mutable hash
tables (indexed by `(column, indent)` pairs) to memoize intermediate results,
and a global counter to assign unique IDs to nodes for sharing detection.
Neither technique is implemented here in Gren, as it greatly increases
the complexity of the code. The omission is unlikely to matter
for typical documents; only extremely large or heavily shared document trees
may be noticeably slower.

# Notes

The ACM article describing the algorithm presents only a few Doc constructors.
They are the most important:
* text
* nl
* concat
* nest
* group

In the article, the CostFactory has a very small, simple itnerface.

The OCaml implementation provides many more constructors, and its CostFactory
has a much more complicated interfae, to support these new constructors.
This Gren package provides all the same constructors as the OCaml version does, and
has the same more-complicated CostFactory interface.

This Gren package also adds another helper constructor, and is open to adding
more as needed:
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

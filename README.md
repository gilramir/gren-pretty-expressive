# Pretty Expressive Printer

A Gren implementation of the
[Pretty Expressive Printer](https://arxiv.org/pdf/2310.01530).

This library implements an *optimal* pretty printer. Given a page-width limit
and a cost model, it searches all possible layouts for the Doc you give it
and selects the one with the lowest cost.
"Lowest cost" according to your cost model equates to "prettiest".
Documents are first constructed from combinator functions and then
rendered into a string with another function.

# Basic Usage

## Creating a Doc

    import PrettyExpressive as P
    import PrettyExpressive.Builder as B

    -- Create a CostFactory, for measuring optimality ("prettiness")
    cf : P.CostFactory P.DefaultCostTuple
    cf =
        P.defaultCostFactory { pageWidth = 80, computationWidth = Nothing }

    -- Build a document for the Gren declaration `pi = 3.14`.
    -- `group` lets the renderer pick the flat form `pi = 3.14` when
    -- it fits, and otherwise break to:
    --     pi =
    --         3.14
    document : P.Doc cost
    document =
        P.concat
            (P.text "pi =")
            (P.nest 4 (P.group (P.concat P.nl (P.text "3.14"))))

`group d` desugars to `choice d (flatten d)`, and `flatten` rewrites
`nl` to its flat-form string (a single space) and then fuses the result
into the adjacent `DocText`. In other words, this Doc has a choice when
redndering, depending on the cost function; if the page width has been reached,
it can redner to the 2-line format, otherwise, it can choose the single-line
format. The constructed `document` looks like:

    DocConcat {
        a = DocText { string = "pi =", width = 4 },
        b = DocNest {
            indent = 4,
            doc = DocChoice {
                a = DocConcat {
                    a = DocNewline (Just " "),
                    b = DocText { string = "3.14", width = 4 }
                },
                b = DocText { string = " 3.14", width = 5 }
            }
        }
    }

It is rendered as:

    pi = 3.14

but if the renderer needs one more space to fit it nicely on the page, this
two-line form is chosen:

    pi =
        3.14

## Sharing sub-documents

A PretteyExpressive Doc is actually a directed acyclic graph (DAG), not a tree.
When the same sub-document is referenced from
multiple parents — typically two branches of a `choice` — the renderer can
resolve it once per `(column, indent)` position and reuse the result.
OCaml gets this for free from value identity; Gren has no equivalent, so
sharing is opt-in via `share`. `share : Doc cost -> Builder (Doc cost)`
stamps a unique id onto a sub-document; every reference participates in
the renderer's memo cache.

Here, the "exit();" acts as a node in the DAG that has two parents:

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

Running the Builder stamps a fresh id (say `0`) on the shared subtree;
both references carry that same id, which is what enables the renderer's
memo cache to reuse one resolution per `(column, indent)`:

    DocChoice {
        a = DocConcat {
            a = DocText { string = " ", width = 1 },
            b = DocShared {
                id = 0,
                nlCnt = 0,
                doc = DocText { string = "exit();", width = 7 }
            }
        },
        b = DocNest {
            indent = 4,
            doc = DocConcat {
                a = DocNewline (Just " "),
                b = DocShared {
                    id = 0,
                    nlCnt = 0,
                    doc = DocText { string = "exit();", width = 7 }
                }
            }
        }
    }

For documents with no repeated sub-trees, `B.return document` is all you
need and the renderer behaves identically to a tree walk. See the docs in
`PrettyExpressive` for the full Builder API and when sharing pays off.


## Rendering the document

Once the P.Doc is built, use P.prettyFormat to convert it to a String

    -- Render the document to a String. Entry points take a `Builder`;
    -- wrap a pure Doc with `B.return` when sharing is not needed.
    printed : Maybe String
    printed =
        P.prettyFormat cf (B.return document)


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

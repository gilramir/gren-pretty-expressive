# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Run tests:**
```sh
cd tests && ./run-tests.sh
```
This compiles the test application (`gren make Main --output=app`) and runs it with Node (`node app`). All test suites live in `tests/src/`.

**Build the package (type-check only, no output artifact):**
```sh
gren make
```

## Architecture

This is a Gren package (`gilramir/gren-pretty-expressive`, v1.1.0) implementing the *Pretty Expressive* optimal pretty-printer algorithm, ported from OCaml. It targets `gren-lang/core` only (platform: common).

### Exposed modules

- **`PrettyExpressive`** (`src/PrettyExpressive.gren`) — the entire library. One large file (~2360 lines) organized into clearly marked sections.
- **`PrettyExpressive.Debug`** (`src/PrettyExpressive/Debug.gren`) — a single `debugShowDoc` function that pretty-prints a `Doc` tree for inspection.

### Key types

| Type | Role |
|---|---|
| `Doc cost` | Abstract document tree. Parameterized by cost type so callers can supply their own cost algebra. |
| `CostFactory cost` | A record of functions defining the cost algebra (text, newline, combine, le, twoColumns hooks, limit). |
| `DefaultCostTuple` | The built-in cost record `{ badness, columnOverflow, height }`, used by `defaultCostFactory`. |
| `MeasureSet cost` | Either `Measures (Array Measure)` (Pareto-optimal candidates) or `Tainted ({} -> Measure)` (computation limit exceeded). |
| `Measure cost` | A single layout candidate: `{ last : Int, cost, layout : Array String }`. |

### Rendering pipeline

1. Caller builds a `Doc cost` tree using combinators (`text`, `concat`, `nl`, `nest`, `group`, etc.).
2. `resolve cf doc c i` recursively evaluates the tree, threading column `c` and indentation `i`. For each node it returns a `MeasureSet` — a Pareto-optimal set of `(ending_column, cost, layout_fragments)` triples.
3. `DocConcat` is handled by `processConcat`, which pairs every left measure with every right measure and Pareto-filters the results.
4. `DocChoice` merges two `MeasureSet`s with `mergeSets`/`mergeLists`.
5. `DocTwoColumns` is handled by `doTwoColumns`, which searches all candidate separator columns by building a `DocChoice` tree and re-evaluating it.
6. When `c` or `i` exceeds `cf.limit`, the result becomes `Tainted` — a single representative layout is tracked rather than the full candidate set.
7. Entry points (`prettyFormat`, `prettyFormatInfo`, `prettyFormatInfoAt`, `prettyFormatDebug`, …) call `resolve` at column 0, extract the best measure, and join layout fragments with `String.join ""`.

### Smart constructors

`concat`, `choice`, `nest`, `align`, `reset`, and `addCost` are *smart constructors* that apply normalization at build time (e.g. fusing adjacent `DocText` nodes, dropping `DocText` wrapping for `nest`/`align`, propagating `DocFail`, floating `DocAddCost` outward).

### Tests

Tests are a separate Gren application in `tests/` with its own `gren.json`. Source directories include both `tests/src/` and `../src/` (the library source). Test suites:
- `Constructors` — tests for smart constructor normalization
- `Costs` — cost model tests
- `OCamlExamples` / `OCamlTests` — parity tests against the OCaml reference implementation
- `ViewItemsTest` — integration-style rendering tests

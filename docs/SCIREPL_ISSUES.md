# TypR Issues for SciREPL Integration

Issues discovered while integrating TypR as a kernel in SciREPL.
Each can be a separate PR to upstream (fabriceHategekimana/typr).

## PR 1: Expand R Standard Library (ready — on feat/expand-r-stdlib)
**Status:** Merged to fork main, not yet PR'd upstream

- `cat()` not in std library — most common R output function
- `print()` only accepts `char`, not `int`/`num`
- Missing: math functions (abs, sqrt, log, sin, cos, etc.)
- Missing: vector ops (length, sort, unique, head, tail, etc.)
- Missing: string ops (nchar, substr, gsub, trimws, etc.)
- Missing: type checking (is.numeric, is.character, etc.)
- Missing: functional (Map, Filter, Reduce, sapply, etc.)
- Missing: comparison operators (==, !=, <, >, <=, >=)
- Missing: control flow (ifelse, tryCatch, invisible, return)

**Files changed:** `configs/std/std_R.ty`, `configs/std/default.ty`,
`configs/std/io.ty`, `crates/typr-cli/src/standard_library.rs`

**Note:** `io.ty` is created but parser skips it — investigate why
new .ty files aren't picked up even with identical syntax to std_R.ty.

## PR 2: @{ }@ Raw-R Blocks in WASM Build
**Status:** Not started — needs Rust work in typr-core/typr-wasm

The TypR CLI supports `@{ ... }@` for embedding raw R code that
bypasses the type checker. The WASM build (`typr-wasm`) does not
support this syntax — the parser throws an error.

**Impact:** UnifyWeaver's TypR target generates code with `@{ }@`
blocks for operations TypR can't express natively (environment
manipulation, complex R closures). This code can't run in the
browser via typr-wasm.

**Proposed fix:** In `typr-wasm/src/lib.rs`, preprocess the source
to extract `@{ ... }@` blocks before parsing, replace with
placeholders, compile the TypR portions, then splice raw R back
into the output.

**Alternative:** Handle in typr-core's parser so both CLI and WASM
benefit.

## PR 3: Semicolon Requirement Documentation
**Status:** Documentation only

TypR requires semicolons at end of statements. This isn't obvious
from the README. Without semicolons, multi-line code fails with
confusing "type Empty doesn't match type any" errors.

```
let x <- 42     # fails
let x <- 42;    # works
```

## PR 4: Function Body Braces Required
**Status:** Documentation only

Single-expression function bodies without braces silently drop
the body in the transpiled output:

```
let f <- function(x) x * x;    # transpiles to: function(x)  (no body!)
let f <- function(x) { x * x };  # works correctly
```

This is a transpiler bug, not just a syntax requirement.

## PR 5: Variadic Function Support
**Status:** Not started — significant parser/type-checker work

R functions like `cat()`, `paste()`, `print()` are variadic.
TypR only supports fixed-arity function declarations. This forces
workarounds like `cat(paste("x =", x))` instead of `cat("x =", x)`.

**Workaround:** Use `paste()` to combine arguments before passing
to single-arg functions.

## PR 6: Literal Type Inference
**Status:** Investigation needed

`42` is inferred as `any`, not `int`. `"hello"` is inferred as
`any`, not `char`. This causes type annotation errors:

```
let x: int <- 42;  # Error: int doesn't match any
```

Works without annotation: `let x <- 42;` (inferred as `any`, used as `int` via operators).

## Priority Order
1. PR 1 (std library) — immediate usability impact, ready to submit
2. PR 2 (@{ }@ WASM) — enables UnifyWeaver TypR target in browser
3. PR 4 (function body bug) — silent data loss, should be high priority
4. PR 3 (semicolons) — documentation
5. PR 6 (literal inference) — type system improvement
6. PR 5 (variadic) — significant feature work

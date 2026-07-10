# TypR / SciREPL integration status

Updated 2026-07-09 after refreshing from `we-data-ch/typr` at `5f8f2a9`.

## Current integration baseline

The fork had been based on `f4151f0` and was 109 upstream commits behind. The
refresh branch `integrate/upstream-2026-07-07` starts at current upstream and
ports only the sciREPL behavior that remains necessary.

Do not replay `0648c01` (the old standard-library expansion) verbatim. Upstream
deleted the duplicate top-level `configs/std` tree and reorganized the active
standard library under `crates/typr-cli/configs/std`. Its signatures should be
re-audited against the current type system instead of resolving the deleted
files back into existence.

## Variadic function inputs: implemented upstream

**Status:** Available upstream since commit `53e5f0d` (2026-05-25), and present
in releases v0.5.3 and v0.5.6.

TypR now accepts a typed trailing variadic parameter:

```typr
let total <- fn(...xs: num): num {
    sum(xs)
};

let collect <- fn(prefix: char, ...xs: char): [#N, char] {
    xs
};

let answer <- total(1.0, 2.0, 3.0);
```

The implementation:

- parses `...name: T` only in trailing parameter position;
- allows zero or more trailing arguments of type `T`;
- exposes the parameter as a typed vector `[#N, T]` inside the function body;
- emits native R `...` and collects it into `typed_vec(...)` for the body; and
- supports fixed leading parameters followed by the variadic parameter.

Focused upstream coverage currently has seven passing variadic tests, including
zero arguments, multiple arguments, fixed-plus-variadic parameters, and a type
error for an incompatible trailing value.

### Fork extensions: heterogeneous dots and forwarding

The refresh branch builds on the upstream homogeneous rest parameter with the
R-facing cases needed by sciREPL:

```typr
let capture <- fn(...args: Any): [#N, Any] {
    args
};

let forward <- fn(prefix: char, ...args: num): num {
    destination(prefix, ...args)
};

let values <- capture(count = 1, label = "x", enabled = true);
```

- `...Any` accepts heterogeneous named arguments. R names are retained by the
  `typed_vec` collector.
- `target(prefix, ...args)` forwards a collected vector, tuple, or record into
  a trailing variadic parameter.
- Forwarded packs must be trailing, and required fixed arguments must remain
  explicit. Scalar spread sources and calls to non-variadic functions are
  rejected by the type checker.
- The R backend emits `do.call` with named list entries and uses
  `typr_call_args` to recover the collected values and names.
- Imported and `extern` calls use the same spread path; extern values are
  converted with `to_native`.

The active standard library now declares variadic `cat`, `print`, `paste`,
`paste0`, `sprintf`, `message`, `warning`, and `c`. The generated
`.std_r_typed.bin` is refreshed from the current configuration tree.

No matching upstream issue or pull request was found under the terms `variadic`,
`varargs`, `variable arguments`, or `dots`; the implementation landed directly
in the commit history.

## Arbitrary `@{ ... }@` raw-R blocks

**Status:** Forward-ported on `integrate/upstream-2026-07-07`.

Upstream already has a `VecBlock` AST node for `@{ ... }@`, so the fork-specific
`RawRBlock` enum variant is no longer needed. The remaining sciREPL gaps were:

- the parser only accepted a small expression subset inside the delimiters;
- markers inside raw `function(...)` bodies leaked into emitted R.

The forward port reuses `VecBlock`, captures arbitrary content up to `}@`, and
strips delimiters from raw R function bodies during transpilation. It retains
the upstream opaque `Empty` sentinel so a raw R result can flow into an explicitly
typed sciREPL binding; using `Any` here is incorrect because TypR treats it as a
top type rather than a dynamic value. Regression coverage includes a multi-line
R closure using an environment and `$` field access, plus the exact typed alias
assignment emitted by sciREPL.

## Standard-library expansion

**Status:** Reviewed subset ported to the active upstream layout.

The old fork commit added useful typed declarations, but most edits targeted a
configuration tree upstream has since removed. Some signatures were fixed-arity
workarounds created before variadic support existed. Reintroducing those files
would fight the upstream reorganization. This branch instead updates
`crates/typr-cli/configs/std/std_R.ty` and regenerates the embedded binary.
Declarations use `Any` for heterogeneous R dots and preserve the fixed leading
argument for `print` and `sprintf`.

## Other previously recorded issues

- Semicolon diagnostics: re-test against the current parser and LSP.
- Single-expression function bodies: re-test; upstream has substantially
  refactored function parsing and now reports empty function bodies explicitly.
- Literal type inference: re-test on v0.5.6 before carrying forward the old
  report that literals became `Any`.

## Validation notes

- All 516 `typr-core` unit tests and all 31 snapshots pass. The deepest
  upstream tests require a larger Rust test-thread stack; they also overflow on
  pristine `5f8f2a9` with the default stack.
- Twelve focused variadic tests pass, including heterogeneous named arguments,
  forwarding, invalid scalar spreads, and active standard-library calls.
- The arbitrary raw-R parser, transpiler, and typed-assignment regressions pass.
- `cargo test -p typr-wasm` and the canonical release
  `wasm-pack build --release --target web crates/typr-wasm` pass.
- Focused UnifyWeaver tests for native `cat` and typed raw-R alias assignment
  pass against the rebuilt CLI.

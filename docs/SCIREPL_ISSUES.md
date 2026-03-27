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

### Remaining variadic opportunities

The current feature is a homogeneous typed rest parameter. It does not yet model
R dots as a heterogeneous named argument row, and there is no obvious syntax for
forwarding a collected vector back into another variadic call. Also, many active
standard-library declarations still describe R variadic functions with fixed
arity (or rely on the untyped R function list). Good follow-up work would be:

1. add variadic typed signatures for common functions such as `cat`, `paste`,
   `paste0`, `sprintf`, and `c`;
2. specify named and defaulted values inside `...`;
3. add a forwarding/spread form for calls; and
4. add WASM tests for declaring, calling, and transpiling variadic functions.

No matching upstream issue or pull request was found under the terms `variadic`,
`varargs`, `variable arguments`, or `dots`; the implementation landed directly
in the commit history.

## Arbitrary `@{ ... }@` raw-R blocks

**Status:** Forward-ported on `integrate/upstream-2026-07-07`.

Upstream already has a `VecBlock` AST node for `@{ ... }@`, so the fork-specific
`RawRBlock` enum variant is no longer needed. The remaining sciREPL gaps were:

- the parser only accepted a small expression subset inside the delimiters;
- the block was typed as `Empty`, even when used as a value; and
- markers inside raw `function(...)` bodies leaked into emitted R.

The forward port reuses `VecBlock`, captures arbitrary content up to `}@`, types
it as opaque `Any`, and strips delimiters from raw R function bodies during
transpilation. Regression coverage includes a multi-line R closure using an
environment and `$` field access.

## Standard-library expansion

**Status:** Needs a clean redesign on the new upstream layout.

The old fork commit added useful typed declarations, but most edits targeted a
configuration tree upstream has since removed. Some signatures were fixed-arity
workarounds created before variadic support existed. Reintroducing those files
would fight the upstream reorganization. Port only reviewed declarations to the
active CLI standard library and use variadic types where they match R semantics.

## Other previously recorded issues

- Semicolon diagnostics: re-test against the current parser and LSP.
- Single-expression function bodies: re-test; upstream has substantially
  refactored function parsing and now reports empty function bodies explicitly.
- Literal type inference: re-test on v0.5.6 before carrying forward the old
  report that literals became `Any`.

## Validation notes

- Seven focused variadic tests pass.
- Both new raw-R regressions pass.
- `cargo test -p typr-wasm` and the release `wasm-pack build --target web` pass.
- A full `typr-core` run reached an existing stack overflow in
  `test_record_kinded_generic_param_transpiles_without_panic`; the same failure
  reproduces on a pristine `5f8f2a9` worktree, so it is not caused by the
  sciREPL forward port.

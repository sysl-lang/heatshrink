# What vendoring heatshrink found in sysl

Found against sysl 0.1.0-alpha.12. Nothing below is worked around in the package.

## 1. A package's C cannot be compiled against cstd's headers (gap)

heatshrink's C includes `<stdlib.h>` and `<string.h>`. On a freestanding target nothing lets the
package's carried C be compiled against `sysl-lang/cstd`'s `include/`:

```
sysl build-lib . --target thumb-freestanding
sysl: error: sh/sysl/heatshrink/c/heatshrink_decoder.c did not compile (clang, exit 1):
sh/sysl/heatshrink/c/heatshrink_decoder.c:1:10: fatal error: 'stdlib.h' file not found
```

The same gap littlefs found (its FINDINGS 1).

**Status: accepted for alpha.14** (sysl branch `vendored-c`).

## 2. `sysl test` has no per-test timeout (gap)

During the mutation pass, a mutated binding (the encoder's `finish` never called) looped for ever
inside a test. The run held until the test binary was killed by hand, and the runner then reported
exit 143. `sysl test --help` lists no timeout. A mutation pass, or any test of code that can loop,
needs one. Wanted: a `--timeout` for the run, or a per-test limit on `@test`, that fails the test
naming it rather than hanging the suite.

**Status: fixed in sysl 0.1.0, checked on 0.1.1** (`test-timeout`): `--timeout <s>` per test,
`@test(timeout: <s>)`, a default of 300 s (`--timeout 0` turns it off). A test that runs out fails by
name and the run continues, and the runner kills its test binaries when it exits or is killed.

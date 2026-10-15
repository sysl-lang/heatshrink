# heatshrink for sysl

LZSS compression for small machines, bound to
[heatshrink](https://github.com/atomicobject/heatshrink) 0.4.1, vendored unchanged.

- **It is small.** The encoder takes about 1.6 KiB and the decoder about 300 bytes, whatever they are
  given: a 256-byte window, matches of up to 16 bytes.
- **It needs no heap.** The encoder's window and search index and the decoder's input buffer and window
  are arrays inside the sysl values that own them. The package allocates nothing.
- **It streams.** Both sides work a few bytes at a time (`sink`, `poll`, `finish`), or over whole
  buffers in one call (`compress`, `decompress`).

```hocon
dependencies {
  heatshrink { git = "github.com/sysl-lang/heatshrink", version = "<the first tag>" }
}
```

The module is `sh.sysl.heatshrink`.

## Using it

```sysl
import sh.sysl.heatshrink.{compress, decompress}

var packed: [256]u8 = [0; 256]
val n = compress(data, packed[0..<256]).unwrap()
var back: [256]u8 = [0; 256]
val m = decompress(packed[0..<n], back[0..<256]).unwrap()
```

Both answer the number of bytes written, or `Error.OutputFull` when the output slice is too small.
Incompressible input grows by one bit in eight, so `input.len + input.len / 8 + 2` bytes always fit a
`compress`.

The streaming interface is for data that does not fit in memory at once, or that arrives in pieces:

```sysl
import sh.sysl.heatshrink.encoder

var enc = encoder()
var out: [64]u8 = [0; 64]
// offer input: it takes what its buffer holds, so alternate with poll
val taken = enc.sink(chunk).unwrap()
loop
    val p = enc.poll(out[0..<64]).unwrap()    // p.written bytes are ready
    if !p.more then break                     // `more`: the slice filled up, poll again
// ... and when the input is over:
loop
    val more = enc.finish().unwrap()          // true while output remains
    // poll until it is dry
    if !more then break
```

`decoder()` is the same with the roles reversed; its input buffer is 32 bytes (`INPUT_BUFFER_SIZE`),
and `sink` answers 0 when it is full until a `poll` has made room. `reset` starts either over.

| | |
|---|---|
| one shot | `compress`, `decompress` |
| streaming | `encoder`, `decoder`; on each `sink`, `poll`, `finish`, `reset` |
| errors | `Error.OutputFull`, `Error.Misuse` (a `sink` after `finish`, a `poll` into an empty slice), `Error.Unknown(code)` |
| parameters | `WINDOW_BITS` (8), `LOOKAHEAD_BITS` (4), `INPUT_BUFFER_SIZE` (32) |

**The format has no length and no checksum.** A decoder given damaged or truncated input does not
fail; it answers whatever those bits decode to. Frame the data if that matters.

## What it is compiled with, and what is left out

`package.hocon` sets `HEATSHRINK_DYNAMIC_ALLOC 0`, static allocation: heatshrink never calls `malloc`,
which is what lets the encoder and decoder live inline in sysl values. That is the only option upstream's
`heatshrink_config.h` takes from the command line. With static allocation upstream fixes the sizes in
the header itself, **unguarded**: window bits 8, lookahead bits 4, input buffer 32 bytes. A `-D` for
any of them is a macro redefinition and the header's value wins, and the header is vendored unchanged,
so these are the package's values (and the usual small-machine choice). Data made with other
parameters, such as the `heatshrink` command-line tool with `-w 11`, cannot be decoded here. Making them
configurable would mean patching the vendored header with `#ifndef`; upstream could take that change.

Not bound: the dynamic-allocation API (`_alloc`, `_free`, window and lookahead as arguments), and the
debugging logs, which need `<stdio.h>`. Both are off.

## Bare metal

The package needs no operating system and no allocator, but its C includes `<stdlib.h>` and `<string.h>`,
and a sysl package has no way yet to hand it `sysl-lang/cstd`'s headers (the gap littlefs hit; accepted for
alpha.14). Until then:

```
sysl build-lib . --target thumb-freestanding
sysl: error: sh/sysl/heatshrink/c/heatshrink_decoder.c did not compile (clang, exit 1):
sh/sysl/heatshrink/c/heatshrink_decoder.c:1:10: fatal error: 'stdlib.h' file not found
```

A target with a C library (the Pico SDK's newlib, Zephyr, FreeRTOS) has the headers already.

## Tests

`sysl test .` runs 18 tests:

- **Round trips** of nine inputs (empty, one byte, two bytes, a run of one byte, text, noise, a pattern
  longer than the window, noise with repeats near and far, every byte value), through the one-shot
  helpers and through the streaming interface fed 1, 3, 7, 33 and 1000 bytes at a time and drained 1, 2,
  5 and 64 at a time. The streaming bytes must equal the one-shot bytes.
- **The error paths**: an output slice one byte short, empty, and exactly the right size, for both
  `compress` and `decompress`; a `sink` after `finish`; a `poll` into an empty slice.
- **Against heatshrink driven from C.** A C program written straight against the headers, built with
  `clang` from the same vendored source with the same options, compresses and decompresses the same
  nine inputs (feeding 7 bytes, polling 5). The binding's compressed bytes must be identical to the
  driver's; the driver must decompress the binding's output, and the binding the driver's. A second test
  checks `sizeof` of both state structs and the window, lookahead and input-buffer parameters against the
  driver's.
- **A golden vector**: a known text compresses to the 17 bytes the C library gives.

Mutations, each run and each red: the encoder's `sink` count off by one; the decoder's likewise; the
encoder's `more` mapped to `EMPTY`; the encoder `finish` not called in `compress`; the decoder's `finish`
mapping swapped; the encoder's error mapping swapped; the overflow probe removed (an exact fit and a
short buffer then disagree); `OutputFull` answered as `Misuse`; the window bits changed from 8 to 7 in
the header (the oracle agrees, since both sides read the same header; the golden vector and the
parameters test are what see it). One survivor, which is equivalent: dropping the decoder `finish` call
from `decompress` changes nothing, because `poll` has already drained everything that was sunk;
`finish` itself is pinned by a test of its own.

**AddressSanitizer**: `SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .` passes 18/18, with 10
`__asan_report` references linked into the test binary.

## Findings

What this package found in sysl is in `FINDINGS.md`.

## Vendored

`sh/sysl/heatshrink/c/` holds `heatshrink_encoder.c`, `heatshrink_encoder.h`, `heatshrink_decoder.c`,
`heatshrink_decoder.h`, `heatshrink_common.h` and `heatshrink_config.h` from
[atomicobject/heatshrink](https://github.com/atomicobject/heatshrink) v0.4.1, unchanged (the command-line
tool and tests are not carried), from the tarball whose SHA-256 is
`7529a1c8ac501191ad470b166773364e66d9926aad632690c72c63a1dea7e9a6`. Vendoring means upstream fixes do
not arrive on their own: watch heatshrink's releases and re-vendor.

Licensing: heatshrink is ISC; the rest of this package is ISC. Both notices are in `LICENSE`.

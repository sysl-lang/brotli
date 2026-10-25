# brotli

**`Content-Encoding: br` for sysl, at both ends.** Brotli is what a browser asks for first — every
one of them sends `Accept-Encoding: br, gzip` — and on text it is worth roughly a sixth over gzip in
the same speed class, more at the quality settings a server can afford for something it compresses
once and serves many times.

*An independent sysl package for Brotli. It is not affiliated with or endorsed by Brotli's authors or trademark holders.*

```
dependencies {
  brotli { git = "github.com/sysl-lang/brotli", version = "0.1.2" }
}
```

`sh.sysl.miniz` is the package for gzip and deflate. This is the one for the header that comes first.

## Compressing and decompressing

```sysl
import sh.sysl.brotli.*

val small = compress(page)?               // a Buf[u8] — nothing to size
val back  = decompress(small.view(), 1 << 20)?
```

**`compress` needs no output buffer at all.** Offered no output space, brotli accumulates into its
own and hands that buffer over, so this asks for none and copies what comes back — there is no
bound to compute, no retry, and no `Short` on the path a caller normally takes.

**`decompress` takes a limit, and it is not optional.** A compressed stream says nothing about how
large it becomes, and a crafted brotli stream of a few hundred bytes expands without bound. A server
decompressing a request body it did not create has to say how much it is prepared to hold, and a
bound that is easy to leave out is one that gets left out — so there is no form that omits it.

Quality is `0` to `11` and defaults to `11`. The scale is not linear: `0`–`4` are fast enough to
compress a response on the way out, `5` is the usual on-the-fly choice, and `9`–`11` are for
something compressed once. Quality 11 on a megabyte of text costs about half a second and **60 MB**
of working memory, which is why it is not what a request handler should reach for.

```sysl
val quick = compress(body, 5)?                          // on the way out
val once  = compress(asset, 11, DEFAULT_WINDOW, Text)?  // built ahead of time
```

## Into storage the caller owns

```sysl
val n = compress_into(page, response[at..])?
```

**`Short` carries the exact size wanted, not a ceiling.** That is the one thing this package can say
that `miniz` cannot: brotli has already done the work into its own buffer by the time the shortfall
is known, so there is nothing to estimate. `bound(len)` is still there for a caller who wants a slice
guaranteed to fit up front — it is generous, 100,030 bytes for 100,000.

## A chunk at a time

`Encoder` and `Decoder` take a chunk of input and a slice to write into, and are deliberately the
same shape `miniz`'s `Deflater` and `Inflater` have — so a program choosing an encoding at run time
writes the two branches the same way.

```sysl
var e = encoder(5)?

while more()
    e.push(next_chunk(), out, last())?

send(out[0..<e.written])
```

An encoder is entitled to hold everything back until it is told to finish, so a `produced` of zero is
ordinary rather than a failure. A `consumed` short of the input means the destination filled.

## Errors say which thing happened

`Truncated` and `Corrupt` are different answers and a caller acts differently on each: one is worth
waiting for more bytes and the other is not. `Corrupt` carries brotli's own code and sentence, which
distinguishes a bad window size from a bad Huffman table from a truncated dictionary. `TooLarge`
names the limit that was passed rather than how large the answer would have been — finding that out
is exactly what the limit exists to refuse to do.

## Requirements

Brotli has to be installed, and so do its headers. **Nothing is vendored**: brotli is a C project
with its own build system and a 120 KB static dictionary, and every platform ships it as one package.

```
brew install brotli
sudo apt install libbrotli-dev
sudo pacman -S brotli
```

The build needs no flags — `pkg-config` is asked where the libraries are.

**`requires { heap = true }`, and there is no way around it.** Both state machines are allocated by
the library, and the encoder allocates and frees repeatedly *during* a compression: 63 allocations
against 57 frees for a one-megabyte input at quality 11, peaking at 60 MB. Several bindings here
redirect a library's allocator so its state lands in storage the program owns; that does not work
here, because the allocator has to recycle rather than bump, and
`BrotliEncoderEstimatePeakMemoryUsage` — which would let a caller size one — is declared in the
public header and **not exported from the shared library**. Nothing here needs `posix`: compression
is arithmetic over two buffers.

## One package rather than two

The encoder is 653 KB and the decoder 87 KB, which looks like an argument for splitting them the way
SDL3 is split here — a program that only decompresses should not carry an encoder it never calls.
**It is not the same case.** SDL3's companions are separate formulae a machine may not have; brotli's
three libraries ship as one package on macOS, Debian and Arch alike, so the second `-l` costs nothing
a user has to act on and nothing a build can fail for. Splitting would buy a smaller link line and
cost every consumer a second coordinate.

## Testing

```
sysl test .
```

22 tests, no files and no network.

**The fixtures were produced by the Rust `brotli` crate**, which is a from-scratch port rather than
another binding of the C library this package calls — so a stream in the suite agreeing with the
plaintext beside it says the *format* was read, not that one library agrees with itself. Each was
offered to the `brotli` command line as well before being written down, and both accepted it.

**The encode direction cannot be pinned that way in a unit test**, because a compressed stream is a
choice rather than an answer: the Rust crate and the C library produce different bytes for the same
input at the same quality and both are correct. Two things stand in for it. `libbrotlienc` and
`libbrotlidec` are separate libraries, so a round trip in the suite is one of them checking the
other rather than code checking itself; and outside the suite, eight streams this package produced —
from an empty input to 100,000 bytes — were read back correctly by **both** the Rust decoder and the
command line.

Each test was also checked to be capable of failing: removing the limit check, never setting the
quality parameter, and treating a truncated stream as a success each redden the tests that claim
those things and nothing else.

**AddressSanitizer** (`SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .`) is clean, on the suite
and on a program compressing 200 KB, with instrumentation confirmed rather than assumed —
`nm -u <binary> | grep asan` reports twelve symbols including `__asan_report_load8`. **What it can
say about this package is limited, and knowing why matters**: sysl's own slice bounds checking
catches the sysl half first, and brotli sub-allocates from large blocks, so an over-read of a
`take_output` pointer stays inside a live allocation that no sanitizer can object to. A deliberate
over-read of four megabytes does fault; one of a hundred kilobytes runs clean. ASan is worth running
here and is not the whole story.

## What is missing

Shared and prepared dictionaries — brotli lets both ends agree on a corpus ahead of time, which is
worth a great deal for many small similar responses and is a second API. The decoder's metadata
callbacks. `BROTLI_OPERATION_FLUSH`, which matters for a long-lived stream that must be readable
before it ends. None of these has been asked for.

## Checking the handles are released

A `BrotliEncoderState` is about half a megabyte, and nothing in a suite notices one being leaked --
the compression is still correct. What notices is the resident set over a loop:

```sysl
for i in 0..<50000
    val out = compress("hello, this is a payload worth compressing a little".bytes)?
```

```
/usr/bin/time -l ./probe
```

**4.8 MB, against 3.3 MB for the same program at a thousand iterations.** Before `encoder` and `decoder`
returned `&Encoder` and `&Decoder` it reached **26 GB**, which is the same program leaking one state
per call.

That is why the constructors hand back a box. A destructor runs when a box's strong count reaches
zero and for nothing else, so a `Drop` on a type nobody boxes is never called -- and a type
annotation at the call site does not fix it, because reading a value into a `&Encoder` binding boxes
a *copy*.

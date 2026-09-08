# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.4 — 2026-09-08

- **Declares its layer**: `layer = "core"` in the manifest — the public API requires no effects, and `novo pkg publish` now checks the code against that budget.  No code changed.  The layers are described under Design in the [publishing guide](https://novo-lang.org/docs/publishing.html#design).
- **Sources reformatted to the canonical form** `novo fmt` prints today (spacing and alignment only; no code changed).

## Unreleased

- **`max_encoded_len` no longer claims its bound is exact everywhere.**
  It is reached exactly by a zero-free payload except when the payload
  is a positive multiple of 254, where a full block ends the input,
  opens no successor, and the encoding comes out one byte shorter than
  the formula counted — 255 rather than 256 at 254 bytes, and the same
  at every step of 254 after.  The bound was always sound (nothing
  exceeds it, no caller ever sized a buffer too small); only the claim
  of exactness was wrong.  A new test asserts both halves over every
  length up to 800.  No code changed.

## 0.1.3

Documentation: the reference is generated from the code, and the
examples in it are doctests.  No code changed — every byte on the wire
is what 0.1.2 produced.

- **Every `pub` item is documented under Go's rule**, the comment block
  directly above the declaration, its first sentence the summary a
  reader meets before opening anything.  `CobsError.message`, which had
  no comment at all, has one.  `novo doc` turns the lot into
  [the package's page](https://novo-lang.org/packages/cobs-nv).
- **Five worked examples, and they run.**  Both pairs are shown where
  they are declared, with the encoding of a payload that contains
  zeros written out in hex, and both malformed-frame answers.  A fenced
  `novo` block in a documentation comment is compiled by `novo doc` and
  run by `novo test src/cobs.nv`, so an example that stopped being true
  is a failing test rather than a reader's afternoon.

## 0.1.2

Developed in its own repository from this version.  `novolang/cobs-nv` is
where the sources live, where CI runs and where releases are tagged;
the novo-lang monorepo no longer carries a copy.  No code changed —
every signature and every byte on the wire is what 0.1.1 shipped.

- **`LICENSE` ships with the package.**  It is on the publish
  allow-list, so the tarball now carries the Apache-2.0 text rather
  than only naming it in the manifest.

## 0.1.1

A patch: byte for byte the same encoder and decoder.  The COBS paper's
own vectors, the bound at every multiple of 254, and both crossings
with `std.codec` all still pass.

- **The test module moved out of `src/`.**  A package's `src/` ships
  whole and a consumer compiles every module in it, so the suite is
  under `tests/` where it is not published.  Run it with
  `novo test tests/cobs_tests.nv`.
- **The manifest carries the fields the registry browses by.**
  `category`, `tags`, `repository` and `maintainers` were added after
  `0.1.0` was published, and a published version is never replaced, so
  this release is the first one the packages page can shelve and
  filter.

There is no bit arithmetic here to rewrite onto the new operators: COBS
counts bytes and copies them, and the one mask in the format is the
`0xff` block length, which is a comparison rather than a bit
operation.

## 0.1.0

First release: `max_encoded_len`, `encode_into`, `encode`,
`decode_into`, `decode`, and the `CobsError` pair.

- **The in-place pair allocates nothing.**  `encode_into` and
  `decode_into` work over a cursor the caller owns, with the encoder's
  per-block back-patch written in place.
- **A malformed frame is a value.**  `ZeroInFrame` and `Truncated` each
  carry the offset; nothing about bad input panics.
- **The full-block case is the paper's.**  A 254-byte block at the end
  of the input opens no successor, which is the one place
  implementations of COBS disagree, and the three long examples are
  asserted by exact encoded length.
- **No delimiter is added or expected.**  Placing the `0x00` that
  separates frames belongs to the caller, who is usually writing into a
  larger buffer anyway.

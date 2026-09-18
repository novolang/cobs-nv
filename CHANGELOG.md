# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**.  See [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.5 — 2026-09-18

The documentation and comments in plain prose; no signature changed.

- **`max_encoded_len` is an upper bound rather than an exact length.**
  A payload with no zeros in it reaches the bound exactly, except where
  its length is a positive multiple of 254.  There a full block ends
  the input and opens no successor, so the encoding comes out one byte
  shorter than the formula counted.  It is 255 rather than 256 at 254
  bytes, and the same at every step of 254 after.  The bound is sound
  either way, and no caller ever sized a buffer too small.  A test
  asserts both halves over every length up to 800.  No code changed.

## 0.1.4 — 2026-09-08

- **The layer is declared.**  `layer = "core"` in the manifest.  The
  public API requires no effects, and `novo pkg publish` checks the
  code against that layer.  No code changed.  The layers are described
  under Design in the [publishing
  guide](https://novo-lang.org/docs/publishing.html#design).
- **The sources are in the canonical form** `novo fmt` prints today.
  Spacing and alignment only, and no code changed.

## 0.1.3

The reference generated from the code, with the examples in it run as
tests.  No code changed, and every byte on the wire is what 0.1.2
produced.

- **Every `pub` item is documented under Go's rule**, the comment block
  directly above the declaration, its first sentence the summary a
  reader meets before opening anything.  `CobsError.message` is
  documented as well.  `novo doc` turns the lot into [the package's
  page](https://novo-lang.org/packages/cobs-nv).
- **Five worked examples, and they run.**  Both pairs are shown where
  they are declared, with the encoding of a payload that contains zeros
  written out in hex, and both malformed-frame answers.  A fenced
  `novo` block in a documentation comment is compiled by `novo doc` and
  run by `novo test src/cobs.nv`, so an example that has stopped being
  true is a failing test.

## 0.1.2

The Apache-2.0 text in the tarball.  No code changed, and every
signature and every byte on the wire is what 0.1.1 shipped.

- **`LICENSE` ships with the package.**  It is on the publish
  allow-list, so the tarball carries the licence text rather than only
  naming it in the manifest.

## 0.1.1

A patch, with the same encoder and decoder byte for byte.

- **The test module moved out of `src/`.**  A package's `src/` ships
  whole and a consumer compiles every module in it, so the suite is
  under `tests/` where it is not published.  Run it with
  `novo test tests/cobs_tests.nv`.
- **The manifest carries the fields the registry browses by.**
  `category`, `tags`, `repository` and `maintainers` were added after
  `0.1.0` was published, and a published version is never replaced, so
  this release is the first one the packages page can shelve and
  filter.

## 0.1.0

The first release.  `max_encoded_len`, `encode_into`, `encode`,
`decode_into`, `decode`, and the `CobsError` pair.

- **The in-place pair allocates nothing.**  `encode_into` and
  `decode_into` work over a cursor the caller owns, with the encoder's
  per-block back-patch written in place.
- **A malformed frame is a value.**  `ZeroInFrame` and `Truncated` each
  carry the offset, and nothing about bad input panics.
- **The full-block case is the paper's.**  A 254-byte block at the end
  of the input opens no successor, which is the one place
  implementations of COBS disagree, and the three long examples are
  asserted by exact encoded length.
- **No delimiter is added or expected.**  Placing the `0x00` that
  separates frames belongs to the caller, who is usually writing into a
  larger buffer anyway.

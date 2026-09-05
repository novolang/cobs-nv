# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

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

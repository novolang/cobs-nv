# cobs-nv

Consistent Overhead Byte Stuffing is a framing for a byte stream. It
removes every zero byte from a payload, so that a zero byte is free to
mark the end of a frame. It is described by Cheshire and Baker in
"Consistent Overhead Byte Stuffing", IEEE/ACM Transactions on
Networking, volume 7 number 2. This package brings the format to
novo-lang, and
[frame-nv](https://novo-lang.org/packages/frame-nv) is built on it.

## What COBS is

A stream of frames needs a delimiter, and a delimiter only works if it
cannot occur inside a frame. COBS makes `0x00` such a byte. It replaces
each zero in the payload with the distance to the next one.

The encoded stream is a run of **blocks**. A block begins with a
**header**, which is a length byte saying how many bytes the block
covers, itself included. The bytes after the header are payload bytes,
and none of them is zero. A header below 255 stands for a zero the
encoder removed, and the decoder puts that zero back. A header of 255
is a full block, which stands for no zero.

A full block at the very end of the input opens no successor. That is
the one place implementations of COBS differ from each other, so the
paper's three long examples are in the test suite by their exact
encoded lengths.

The overhead is at most one byte per 254 bytes of payload, plus one,
whatever the data. There is no input that doubles a frame, which is
what lets a link size its buffers in advance.

| Quantity | Value |
| --- | --- |
| The delimiter | `0x00` |
| Payload bytes one block carries, at most | 254 |
| Bytes one header covers, itself included, at most | 255 |
| Overhead on `n` bytes of payload, at most | `n / 254 + 1` |
| Encoded length of `n` bytes, at most | `max_encoded_len(n)` |
| Bytes of room `decode_into` needs, at most | the length of the frame |

## Install

```
novo pkg add cobs-nv
```

## Example

```novo
use std.bytes
use cobs

fn main() [io]
    // A payload with zeros in it. The encoding contains none.
    let payload = bytes.from_hex("110000ff") ?? bytes.zeros(0)
    println(bytes.to_hex(cobs.encode(payload)))   // 02110102ff

    // The same bytes on the way back, zeros included.
    match cobs.decode(cobs.encode(payload))
        Ok(back) => println(bytes.to_hex(back))   // 110000ff
        Err(e)   => println(e.message())
```

Build and test with `novo pkg build` and `novo test tests/cobs_tests.nv`.

## What the package contains

| Module | Contents |
| --- | --- |
| `cobs` | The encoder and the decoder, each in a form that allocates its answer and a form that writes into a cursor the caller owns, the size a caller reserves, and the two ways a frame can be malformed. |

Every `pub` declaration is documented where it is declared, and
[the package's page](https://novo-lang.org/packages/cobs-nv) is that
reference.

## How to choose an entry point

**`cobs.encode` and `cobs.decode` take the whole payload.** Each
allocates its answer and hands it back. This is the call for a program
that is assembling a message rather than filling a frame.

**`cobs.encode_into` and `cobs.decode_into` write into a `Cursor` you
already own.** They allocate nothing. Size the destination with
`cobs.max_encoded_len` before encoding, and with the length of the
frame before decoding. This is the call for a program that writes into
a transmit buffer it holds already.

## The rules a user needs

1. **No delimiter is written and none is expected.** This module reads
   and writes the bytes of one frame. Appending the `0x00` that
   separates frames belongs to the caller, who is usually placing it in
   a larger buffer anyway. On the way back, split the received bytes at
   each `0x00` and hand each piece to `decode`.
2. **The destination is a cursor rather than a `Bytes`.** A `Bytes`
   passed as a parameter is borrowed, so a write through it lands in a
   copy the caller never sees. A cursor owns its buffer, so a write
   through one lands where the caller can read it. The standard
   library's own writers take a cursor for the same reason.
3. **`max_encoded_len` is an upper bound rather than an exact length.**
   A payload with no zeros in it reaches the bound exactly, except
   where its length is a positive multiple of 254. There the last block
   is a full one that opens no successor, and the encoding is one byte
   shorter than the bound. `max_encoded_len(254)` is 256 and `encode`
   writes 255.
4. **Decoding never grows a frame.** A destination as long as the frame
   is always enough room for `decode_into`.
5. **A destination that is too small is a panic.** `encode_into` panics
   when the cursor has less room than `max_encoded_len` asks for,
   exactly as `c.put_u8` and `xs[i]` do. A buffer the caller sized
   wrong is a mistake in the program.
6. **A malformed frame is never a panic.** `CobsError` has two
   variants, and both are input faults. `ZeroInFrame` is a `0x00` among
   the encoded bytes, which the encoding never produces. `Truncated` is
   a block header covering more bytes than the frame has. Each carries
   the offset, so a receiver dropping a bad frame can say where it went
   wrong.
7. **The allocating pair allocates one buffer each.** `encode` and
   `decode` allocate the answer they hand back. `encode_into` and
   `decode_into` allocate nothing at all.

Writing the delimiter is one line after the frame.

```novo
use cobs
use std.bytes

// One frame into a transmit buffer, delimiter included.
fn put_frame(var out: Cursor, payload: Bytes) -> Int
    let n = cobs.encode_into(out, payload)
    out.put_u8(0)
    n + 1
```

## What is not included

- **The reduced-overhead variant, COBS/R.** It changes the bytes on the
  wire, so it belongs in its own package rather than behind a flag
  here.
- **Zero-run compression, rzCOBS.** It is the framing the `defmt`
  logging format uses, and it is a different wire format.
  [rzcobs-nv](https://novo-lang.org/packages/rzcobs-nv) is the package
  for it.
- **Decoding across buffer boundaries.** A frame is decoded whole, and
  a receiver holding half a frame waits for the rest.
- **The delimiter.** Nothing here writes or reads the `0x00` between
  frames, and `max_encoded_len` does not count it.
- **A check over the frame.** COBS finds the boundary between frames
  and detects nothing, so a frame that arrives with a flipped bit
  decodes to a payload with a flipped bit.
- **A claim that the package builds for a microcontroller.** It ships
  no device probe, so nothing here is checked on a board.
- **Any input or output.** Every function in this package works over
  bytes the caller already holds.

## Related packages

- [frame-nv](https://novo-lang.org/packages/frame-nv) is a whole framed
  packet: a length prefix, the payload, a CRC-32C trailer over both,
  the lot COBS-encoded, and a streaming reader. It is what to reach for
  when a frame needs a length and a check as well as a boundary.
- [rzcobs-nv](https://novo-lang.org/packages/rzcobs-nv) is reverse
  zero-compressing COBS, the framing the `defmt` logging library sends
  its logs in. A payload full of zeros comes out shorter than it went
  in, and the bytes on the wire are not these ones.
- [crc-nv](https://novo-lang.org/packages/crc-nv) has the cyclic
  redundancy checks, which is the part of a frame this package does not
  do.

## Test vectors

```
novo test tests/cobs_tests.nv
```

The vectors are Cheshire and Baker's own table from the paper that
introduced the format. They include the three 254-byte cases above,
asserted by exact encoded length. Beyond those, the suite runs a round
trip over every payload length up to 300 for both a zero-free and an
all-zero payload. It asserts that an encoded frame contains no zero at
all, that the bound is reached where the rules above say it is, and
that each of the two ways a frame can be malformed is reported with its
offset.

## Licence

Apache-2.0. See `LICENSE`.

# cobs-nv

Consistent Overhead Byte Stuffing. A stream of frames needs a
delimiter, and a delimiter only works if it cannot occur inside a
frame. COBS removes every `0x00` from a payload so that `0x00` is free
to end a frame — and it does so at a cost of at most one byte per 254
bytes plus one, whatever the data. That bound is the whole point: there
is no input that doubles a frame, so a link can size its buffers in
advance.

```novo
use cobs

fn main() [io]
    let framed = cobs.encode(payload)     // no 0x00 anywhere in it
    match cobs.decode(framed)
        Ok(back) => println("${bytes.len(back)} bytes")
        Err(e)   => println(e.message())
```

```
novo pkg add cobs-nv
```

## What it gives you

| Function | |
|---|---|
| `cobs.max_encoded_len(len: Int) -> Int` | the buffer to reserve for `len` bytes of payload |
| `cobs.encode_into(dst: Cursor, src: Bytes) -> Int` | `src` encoded at the cursor; the byte count is the return |
| `cobs.encode(src: Bytes) -> Bytes` | a fresh buffer of exactly the length it needed |
| `cobs.decode_into(dst: Cursor, wire: Bytes) -> Result<Int, CobsError>` | `wire` decoded at the cursor |
| `cobs.decode(wire: Bytes) -> Result<Bytes, CobsError>` | a fresh buffer of the payload |

`CobsError` has two variants, both input faults: `ZeroInFrame` when the
encoded bytes contain a `0x00`, which the encoding never produces, and
`Truncated` when a block header covers more bytes than the frame has.
Each carries the offset, so a receiver dropping a bad frame can say
where it went wrong.

## The delimiter is yours

This module writes and reads bytes and nothing else: it adds no
delimiter and expects none. Appending the `0x00` that separates frames
belongs to the caller, who is usually placing it in a larger buffer
anyway:

```novo
use cobs
use std.bytes

// One frame into a transmit buffer, delimiter included.
fn put_frame(out: Cursor, payload: Bytes) -> Int
    let n = cobs.encode_into(out, payload)
    out.put_u8(0)
    n + 1
```

On the way back, split the received bytes at each `0x00` and hand each
piece to `decode`.

## Writing into a buffer you already hold

`encode_into` and `decode_into` take a cursor and allocate nothing. The
offset lives in the cursor rather than in a third parameter because a
`Bytes` passed as a parameter is borrowed — the caller still holds it,
so a write through it lands in a copy the caller never sees. A cursor
owns its buffer, so a write through one lands where the caller can read
it. That is the same rule the standard library's own writers follow.

```novo
var out = bytes.cursor_le(bytes.zeros(cobs.max_encoded_len(bytes.len(payload))))
let written = cobs.encode_into(out, payload)
```

For decoding, `bytes.len(wire)` bytes of room is always enough:
decoding never grows a frame.

## What it costs

One pass over the input, one output byte per input byte plus the block
headers, and one back-patch per block — a block's length is only known
once the block ends, so the encoder reserves the byte, keeps writing,
and seeks back to fill it in. The seek costs nothing: the buffer is the
cursor's own, so the patch is a write in place.

`encode_into` and `decode_into` allocate nothing at all. `encode` and
`decode` allocate one buffer each.

`encode_into` panics when the cursor has less room than
`max_encoded_len` asks for, exactly as `c.put_u8` and `xs[i]` do: a
buffer the caller sized wrong is a mistake in the program. Malformed
**input** is never a panic — that is what `CobsError` is for.

## The 254-byte cases

Implementations of COBS differ from each other in one place: what to do
when a full 254-byte block lands at the very end of the input. A full
block stands for no zero, so it needs no successor — an encoder that
opens one anyway writes a byte nobody else writes. This one does not,
which is why the three long examples from the original paper are in the
test suite by their exact lengths.

## What it does not do

No reduced-overhead variant (COBS/R), and no zero-run compression
(rzCOBS, which the `defmt` logging format uses). Both change the wire
format, so both belong in their own package rather than behind a flag
here. No streaming across buffer boundaries: a frame is decoded whole,
and a receiver holding half a frame should wait for the rest.

## Tests

```
novo test src/cobs_tests.nv
```

The vectors are Cheshire and Baker's own table from the paper that
introduced the format, including the three 254-byte cases above,
asserted by exact encoded length. Beyond those: a round trip over every
payload length up to 300 for both a zero-free and an all-zero payload,
the property that an encoded frame contains no zero at all, and each of
the two ways a frame can be malformed.

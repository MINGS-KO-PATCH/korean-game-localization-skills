# Type 1 — 1-byte length-prefixed stream

### Identification and layout

After an optional preserved block header, records repeat cleanly as:

```text
[len:u8][text:len bytes][len:u8][text:len bytes]...
next_record = current_record + 1 + len
```

A convincing candidate walks through many consecutive decodable records without boundary drift. The clean run identifies the stream start; bytes before it remain protected header or index data until separately understood.

### Rewrite rule

Serialize each record with its new byte length. There is no separate pointer table inside this stream. Following record positions result from concatenation. Preserve the total block size and fill only the established padding range with the established padding value.

Fail when one encoded string exceeds 255 bytes or when the serialized stream exceeds the fixed block capacity.

### Verified case

- `ScpPackFile.fbe`, block `[0xD89, 0x2694)`, capacity 6,411 bytes; stream begins at `0xDB9`, leaving a 48-byte header.
- 224 consecutive strings parsed with zero decode failures.
- Unchanged extraction and reinsertion matched all 333,732 source-file bytes.
- Individual-length and block-capacity overflow paths both stopped with errors.



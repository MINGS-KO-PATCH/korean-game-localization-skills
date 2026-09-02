# Type 2 — 2-byte cumulative offsets with null-terminated strings

### Identification and layout

```text
[header:8]
  count:u16le
  text_size:u32le
  padding:2
[cumulative_next_starts:u16le × (count-1)]
[null-terminated text area:text_size]

block_size = 8 + 2 × (count - 1) + text_size
table[i] = sum(serialized_size(text[0..i]))
```

The first string start is implicit at text-relative offset zero. Each table entry is the next string start, so the entry count is one less than the string count. Confirm monotonicity, every null-terminated slice, and the exact block-size equation.

### Rewrite rule

Rebuild the text area, derive every cumulative value from its serialized prefixes, and rewrite `text_size`. If the population changes, also rewrite the count and table extent. A change to an early string therefore affects all later table values.

Fail when a cumulative value exceeds `0xFFFF` or the complete rebuilt layout exceeds its enclosing block capacity.

### Verified case

- `TextPack.fbe` block 0 at `[0x228, 0x7CC)`, size `0x5A4`.
- Count 58, 57 table entries, text size `0x52A`.
- `8 + 114 + 0x52A = 0x5A4`; all stored values matched measured cumulative starts.
- Unchanged reconstruction matched all 1,444 block bytes.



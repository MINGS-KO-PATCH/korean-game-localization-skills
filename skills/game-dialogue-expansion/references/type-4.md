# Type 4 — nested absolute and record-local offsets

### Identification and layout

The outer file follows Type 3, but each pointer selects a block rather than a single string:

```text
[count:u32le][block_start:u32le × count][blocks]

block:
  p1:u16le
  p2:u16le
  fixed_metadata:16 bytes
  text_fields

fixed_region_size = 20
p1,p2 = block-relative starts of later text fields
```

For ordinary verified records whose fields are separated by `00 00`:

```text
local_start_after_separator = 20 + separator_position_in_text + 2
```

The stored `p1` and `p2` values are authoritative. Delimiter positions help explain or rebuild them only for the record shapes where the relation is verified.

### Rewrite rule

Rebuild two dependency layers:

1. Recalculate `p1` and `p2` from the new subfield boundaries for supported record shapes.
2. Recalculate every outer absolute block start from final serialized block sizes.

Preserve the other 16 fixed bytes. Do not apply the ordinary two-separator formula blindly to exceptional records.

### Verified case and exceptions

- Sample size `0x7343`; count 210; `4 + 210 × 4 = 0x34C`, equal to the first block start.
- Each block has a 20-byte fixed region. The first four bytes hold two `u16le` local starts; the remaining 16 include metadata such as dates, identifiers, and flags.
- The formula matched 197 of 210 blocks. Twelve blocks had one `00 00` separator and one had three; their stored local offsets and original fixed region must remain the governing evidence until their variants are established.



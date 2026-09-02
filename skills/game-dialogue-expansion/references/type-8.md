# Type 8 — reordered 4-byte cumulative offset references (provisional)

### Identification and layout

This form has no adjacent ID field. A reference region is an array of little-endian 4-byte values whose storage order follows the game's lookup order rather than text order:

```text
canonical text boundaries:
  boundary[i] = sum(serialized_length(text[0..i]))

reference array:
  slot[j]:u32le = boundary[target_index[j]]
```

Values are text-area-relative cumulative ends. A sequence such as `boundary[5], boundary[4], boundary[3], boundary[2], boundary[1]` is valid when that is the original reference order. It must not be sorted or regenerated as `1, 2, 3...`. Unlike Type 7, the four bytes before a cumulative value are not an ID/value pair; every 4-byte word is an independent reference slot.

### Historical reconstruction workflow

The Tengoku-hen work used the following practical method:

1. Extract text in canonical text order and calculate each serialized byte length, including the established terminator.
2. Build the cumulative boundary list from those lengths.
3. Dump every aligned 4-byte word from the candidate reference file, keeping its file offset.
4. Match each original reference value to the corresponding original cumulative boundary.
5. Link that reference slot to the new cumulative boundary in a spreadsheet, preserving the original slot order.
6. Serialize the resulting 4-byte little-endian words back to the reference file.

The historical `SRVC_35.xlsx` makes this relationship explicit: `SHEET1` contains 549 four-byte slots and `SHEET2` contains 378 text rows. Of the 549 slots, 547 are formulas and two remain literal zero. The formulas reference 373 distinct text boundaries, with 172 repeated references beyond the first occurrence and as many as six slots targeting one boundary. There are 112 descending transitions in 544 adjacent formula targets, proving that storage order is not text order. The first targets are rows `6, 5, 4, 3, 2, 8, 9, 119, 10, 139...`.

### Safe rewrite rule

Create and protect an explicit slot catalog:

```text
reference_slot | original_file_offset | original_value | target_boundary
```

After encoding all translated text, recompute the canonical boundary list once. For each original slot, write `new_boundary[target_boundary]` at the same slot. Preserve literal zeros and any proven non-reference fields. Never globally replace every occurrence of an old 4-byte value: empty strings, shared boundaries, duplicate lengths, and unrelated integers can have identical byte patterns.

The catalog must resolve identity from structural evidence such as a validated table extent, consumer lookup, or an approved per-revision map. A value match alone is only a candidate. Unknown characters must fail encoding; silently dropping them changes cumulative lengths and corrupts later references.

### Automatic discovery of the reference-table start

Do not require the user to supply a known start address. When the first text address and canonical cumulative boundaries can be established, derive a candidate table immediately preceding the text:

```text
allowed = set(original_cumulative_boundaries) union {proven_literals}
position = text_start - 4

while position >= 0 and read_u32le(position) in allowed:
    position -= 4

candidate_start = position + 4
candidate_end = text_start
```

For original `SRVC_35.bin`, the first valid text starts at `0x2B08`. Every aligned word from `0x2B04` backward through `0x2274` is an original cumulative boundary or allowed zero. The preceding word at `0x2270` is `0x00000003`, which is not in the allowed set, so the derived start is `0x2274`. The extent is exact:

```text
0x2B08 - 0x2274 = 0x894 = 2,196 = 549 × 4
```

Report this as a strong candidate only after checking text decoding at `candidate_end`, four-byte alignment, slot coverage, and consistency with the extracted population. If an unrelated preceding word happens to equal a boundary, backward scanning can overrun the real table. Resolve that case with a header/count, neighboring record grammar, cross-file comparison, consumer evidence, or controlled differential testing. If no unique extent can be established, return `NOT YET ESTABLISHED` and request the next evidence instead of inventing an address.

### Current evidence

- Historical calculated point file `포인트35`: 2,196 bytes, exactly 549 little-endian words; SHA-256 `9ee8217d595fb141c4daa0806df5511aba19fd14ec12431f23df0cb8fc546752`.
- Original `SRVC_35.bin`: the Type 8 table is automatically derivable as `[0x2274, 0x2B08)` by the backward boundary scan above; the user need not know `0x2274` in advance.
- Historical point workbook `SRVC_35.xlsx`: `SHEET1` maps reference slots to `SHEET2` cumulative boundaries with formulas such as slot offsets `0x00..0x10` mapping to text rows `6, 5, 4, 3, 2`.
- The saved calculated point file agrees with 544 of 549 formula/literal results reconstructed from the current workbook. Five slots differ, so the workbook and output are useful evidence of the method but not a byte-identical final provenance pair. Treat the discrepancies as unresolved revision or manual-edit history.
- Earlier `SRVC_2.bin` inspection also showed cumulative boundaries reused by reordered/subset arrays, including repeated equal boundaries. This supports the relation but does not by itself establish every SRVC file's complete reference population.

### Provisional limits and promotion conditions

Keep Type 8 `CONDITIONAL` until all applicable items are closed for the target revision:

1. Establish the exact reference-array extent and separate real slots from coincidental 4-byte matches.
2. Preserve a revision-bound slot-to-boundary catalog and define how duplicate/shared boundaries are disambiguated.
3. Explain or eliminate every workbook/output mismatch and require an unchanged full-file round trip.
4. Perform a controlled one-string growth test and verify every predicted slot changes while unrelated words remain identical.
5. Prove the runtime consumer meaning of the reordered slots and test representative battle-dialogue routes.
6. Reject decreasing/out-of-range boundaries, unmapped characters, width overflow, output capacity overflow, and enclosing-container violations.

Do not convert this form to Type 7 unless an actual paired ID field is demonstrated. Do not sort slots, deduplicate repeated references, or infer target identity solely from the numeric value.



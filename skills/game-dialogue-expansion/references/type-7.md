# Type 7 — 4-byte ID plus 4-byte cumulative text length (provisional)

### Identification and layout

After a 12-byte header, the verified candidates use 8-byte table entries followed by a contiguous text region:

```text
[raw_count:u32le][fixed1:u32le][fixed2:u32le]
[entry × table_count]

entry:
  text_id:u32le
  cumulative_end:u32le

text_range(0) = [0, cumulative_end[0])
text_range(i) = [cumulative_end[i-1], cumulative_end[i])
```

`cumulative_end` is relative to the text-area start and must be nondecreasing and within the established text extent. It is the serialized byte sum through the current entry, not a file-absolute pointer.

The first four bytes are an ID/key, even when they happen to equal `1, 2, 3...`. Do not infer that they are disposable row numbers.

### Variants

- **Type 7-A — continuous ID:** IDs appear as a continuous sequence. Preserve them anyway; continuity is observed data, not permission to regenerate them.
- **Type 7-B — irregular ID:** IDs use gaps or namespaces such as `1–31`, `50`, `1000–1034`, `10000...`. Preserve every original ID and entry order exactly.

### Rewrite rule

Encode entries in original table order. Copy each original `text_id` unchanged and derive `cumulative_end` from the final serialized byte lengths:

```text
running = 0
for entry in original_entry_order:
    running += len(serialized_text(entry))
    write_u32le(entry.text_id)
    write_u32le(running)
```

Changing an early string affects its cumulative end and every following cumulative value. It does not change later IDs. Reject duplicate, missing, reordered, or newly sequentialized IDs unless a separately established schema transformation explicitly changes the population.

### Current evidence

- `UICOMMONTEXT.BIN_08`: `raw_count=216`; 215 observed table IDs form `1–215`; cumulative values are monotonic.
- `TOWNTEXT.BIN_11`: `raw_count=127`; 126 observed IDs are unique but irregular, including the runs `1–31`, `50`, `1000–1034`, several `10000` ranges, `100000–100019`, `1000000–1000002`, and `10000000–10000004`.
- Every observed ID in the two files is identical between the compared source and Korean output. This establishes preservation, not automatic regeneration.
- Existing scripts that write `struct.pack('<I', i)` would destroy Type 7-B IDs. An improved extractor must export the stored ID, and the reinserter must consume the protected original ID.

### Provisional limits and promotion conditions

- Confirm the exact relation between `raw_count`, table entry count, and any implicit last entry for every file variant.
- Separate the final indexed text from unindexed tail data and file padding. Do not assume that every byte through EOF belongs to the last string.
- Prove unchanged full-file round trips for both continuous and irregular-ID examples.
- Fail on decreasing/out-of-range cumulative values, ID population changes, unknown text bytes, and output/container capacity violations.
- Verify that the game looks up Type 7-B text by stored ID or otherwise establish the consumer meaning of the first field.

Until these conditions are closed, use Type 7 to preserve and rebuild the observed table safely, but report the expansion gate as `CONDITIONAL` rather than universally `EXPANDABLE`.



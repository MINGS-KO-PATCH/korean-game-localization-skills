# Verified dialogue-expansion structures

This is the integrated, extensible record of dialogue-expansion structures established during Korean localization work. Types 1–5 are verified patterns. Types 6–8 are intentionally provisional because parts of their exception handling, boundary interpretation, or runtime proof may be improved. The type numbers are local working names. Match the relations and evidence, not the label.

## Comparison map

| Type | Boundary authority | Coordinate basis | Change propagation | Main constraint |
|---|---|---|---|---|
| 1. Length-prefixed stream | Per-record 1-byte length | Sequential within a fixed block | Rewrite that record length; following positions emerge from serialization | Record ≤ 255 bytes and serialized stream ≤ block capacity |
| 2. Cumulative-offset block | Header count, 2-byte cumulative table, null terminators | Text-area relative | Recompute all later cumulative values and text-area size | Offset ≤ `0xFFFF` and complete layout ≤ block capacity |
| 3. Absolute-offset file | Header count and 4-byte pointer table | File absolute | Recompute all later starts; table-size changes move every start | Preserve duplicate offsets and count; enclosing container may impose capacity |
| 4. Nested-offset blocks | Outer 4-byte table plus record-local 2-byte offsets | File absolute outside, block relative inside | Recompute local field offsets and all later outer block starts | Preserve fixed metadata and handle exceptional delimiter populations |
| 5. Lua 5.1 size-prefixed string | Per-string 4-byte size including trailing null | Sequential Lua binary chunk | Rewrite that string size; following structures move through serialization | Size is not an offset; preserve chunk grammar and reject unmapped characters |
| 6. Two-level section directory (provisional) | Top-level section directory and complete inner pointer population | File-absolute section starts; section-relative inner targets | Recompute section sizes, later section starts, and every affected inner relative target | 4-byte text alignment, 16-byte file alignment, protected commands, and explicit embedded-reference exceptions |
| 7. 4-byte ID plus cumulative length (provisional) | 8-byte entries: original ID/key plus cumulative text end | Text-area relative | Preserve every ID and entry order; recompute cumulative ends from serialized text | Continuous and irregular-ID variants; final tail, padding, and `raw_count` meaning need full confirmation |
| 8. Reordered 4-byte cumulative references (provisional) | 4-byte slots select cumulative text boundaries in consumer-defined order | Text-area relative values stored in an independent reference array | Preserve every slot and its target boundary; rewrite each slot from that boundary's new cumulative end | Duplicate boundary values make value-only matching ambiguous; table population and consumer meaning require proof |

## Shared analysis sequence

1. Read the declared count or find a candidate repeating region without editing it.
2. Test layout equations against exact file or block boundaries.
3. Determine whether values represent byte lengths, current starts, next starts, or subfield starts.
4. State the origin for each value and verify it against several entries, including the last entry and exceptions.
5. Establish how empty, duplicate, shared, and final entries are represented.
6. Serialize unchanged content from protected source fields and require byte identity.
7. Change one representative entry length and predict every affected field before runtime testing.

## Type 1 — 1-byte length-prefixed stream

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

## Type 2 — 2-byte cumulative offsets with null-terminated strings

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

## Type 3 — 4-byte absolute starts

### Identification and layout

```text
[count:u32le]
[start:u32le × count]
[string payloads]

first_start = 4 + 4 × count
range(i) = [start[i], start[i+1])
range(last) = [start[last], EOF)
```

Starts are file-absolute and nondecreasing rather than strictly increasing. Equal adjacent starts represent empty entries or duplicate references and must remain in the authoritative population. Payloads in the verified case end with `00 00`, but pointer ranges—not delimiter scans—define entry identity and boundaries.

### Rewrite rule

Serialize exactly `count` entries, including zero-length entries, and derive every absolute start from the final table end and preceding serialized payloads. If count changes, the table changes by four bytes per entry, shifting the base of every payload in addition to payload-length deltas.

The standalone file has no intrinsic fixed-size ceiling in the verified case. Recheck any archive, filesystem, loader, or buffer that encloses it.

### Verified case and trap

- `texts_jp` sample size `0x7419`; count 517; `4 + 517 × 4 = 0x818`, equal to the first start.
- 18 starts duplicate their predecessor. Pointer-based extraction produced 517 entries with zero decode failures.
- Sequential `00 00` scanning found only 499 entries and lost the 18 empty positions, proving it unsuitable as the population authority.
- Unchanged reconstruction matched all 29,721 bytes.

## Type 4 — nested absolute and record-local offsets

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

## Type 5 — Lua 5.1 4-byte size-prefixed strings

### Identification and layout

The verified `script_1.x` begins with a Lua 5.1 binary-chunk header:

```text
1B 4C 75 61 51 00 01 04 04 04 04 00
\e  L  u  a  5.1       int=4, size_t=4, instruction=4, number=4
```

Its serialized strings use a 4-byte little-endian `size_t` immediately before the payload. For string constants, a Lua constant tag `04` commonly precedes that size:

```text
[optional string tag:04][size:u32le][payload:size-1 bytes][terminator:00]
size = encoded_payload_length + 1
next_structure = payload_start + size
```

The four bytes are a **length**, not a file offset or pointer. Establish this by comparing the stored integer against several exact payload extents. Do not classify every four-byte value before text as Type 5 merely because it is little-endian or small.

### Rewrite rule

Encode the replacement using the one approved character table, append the required `00`, and write `len(encoded_payload) + 1` as `u32le` immediately before the payload. Rebuild or splice the enclosing sequential chunk so later bytes move by the payload delta. There is no Type 2 or Type 3 table whose later values must be increased merely because their file positions changed.

Preserve the surrounding Lua grammar, tags, counts, instructions, nested prototypes, debug data, and non-dialogue string constants. A Shift-JIS scan alone does not establish that a candidate is player-visible dialogue.

Fail when a character is absent from the approved mapping. Do not fall back to UTF-8 because doing so changes both the encoding and the computed size without proving that the game renderer accepts those bytes.

### Verified case and tool-chain cautions

- File: `부르냥맨/PSP_GAME/USRDIR/대사/script_1.x`, size 826,778 bytes; header identifies a Lua 5.1 binary chunk with 4-byte `size_t` and little-endian representation.
- `System_StartDrawLoaded`: payload 22 bytes, stored prefix `17 00 00 00` = 23, followed by `00`.
- `Sprite_Load`: payload 11 bytes, stored prefix `0C 00 00 00` = 12, followed by `00`.
- `boss\\cuko\\cuko.ssf`: payload 18 bytes, stored prefix `13 00 00 00` = 19, followed by `00`.
- `BGM_Play`: payload 8 bytes, stored prefix `09 00 00 00` = 9, followed by `00`.
- `1_TEXT_1ST_UN.py` records the four preceding bytes as `길이포인트`; `3_POINT_RE.PY` recomputes that field as a 4-byte little-endian length from the edited text and adjustment value.
- `1_TEXT_1ST_RE.py` writes the `길이포인트` supplied by the workbook rather than deriving it from `new_bytes`. Therefore its input must already contain the recomputed value, and a robust successor should derive and validate `len(new_bytes) + 1` during reinsertion.
- The current extractor scans Shift-JIS-like null-terminated sequences and may collect code constants or other non-dialogue strings. Treat its output as candidates until usage or context establishes the dialogue population.

## Type 6 — two-level section directory with aligned dialogue blocks (provisional)

### Applicability and status

This pattern is established from `나의 여름방학1` `_0001` script-like files and the current `text_editor_tool#23.py`. Its two-level table relocation has strong static evidence across the compared Japanese and Spanish populations. Keep the type **provisional** because three embedded references still use file-specific manual targets, padding policy may receive runtime-driven changes, and full release-scope runtime proof is not represented by the current offset records alone.

### Identification and layout

The file begins with a top-level section directory:

```text
[section_count:u32le]
[section entry × section_count]

section entry (8 bytes):
  identifier:u16le
  section_size:u16le
  section_absolute_offset:u32le
```

Each selected section starts with its own relative-offset table:

```text
section_base:
  pointer_count:u32le
  relative_offset:u32le × pointer_count

absolute_target = section_base + relative_offset
section_limit = section_base + section_size
```

Section starts are unique, ascending, 4-byte aligned, at or after the top-level directory, and within the file. Every declared section extent and inner target must remain within its section boundary.

Dialogue discovery is pointer-driven. A recognized dialogue header is 12 bytes; its body begins at `align_up(header_offset + 12, 4)`. In the verified tool, both the header location and aligned body location must occur in the section's authoritative relative-pointer population. The visible seven-character code in the header is metadata, not the discovery source.

### Rewrite and propagation rules

Replace each dialogue through its original aligned end and align the rebuilt block end to four bytes. Let each edited block provide `(original_text_offset, byte_delta)`. Derive every relocation from the original file and the complete sorted delta map:

```text
delta_before(position) = sum(delta for edited_start < position)

new_section_base = old_section_base + delta_before(old_section_base)
new_section_size = old_section_size + sum(delta inside old section extent)

new_target = old_target + delta_before(old_target)
new_relative = new_target - new_section_base
new_pointer_slot = new_section_base + 4 + pointer_index × 4
```

Rewrite the top-level `section_size:u16le`, top-level `section_absolute_offset:u32le`, and every affected inner `relative_offset:u32le`. Do not filter inner pointers by whether they appear to target text: the same table also reaches commands, branches, and auxiliary data whose positions move with earlier dialogue.

After rebuilding the decompressed `_0001` file, align its end to `0x10` with the established file padding. When the file is returned to its GZIP/GZX or higher container, treat compression, member alignment, stored size, and following outer offsets as another dependency layer; expansion of the inner file alone does not update that container.

### Protected data and exceptions

- Preserve every 12-byte dialogue header and every non-translatable command/control byte after mapping it through the delta function.
- Editable byte classes are restricted to the approved character table, display space `00 00`, line break `01 80`, and parameterized page break `02 80 XX XX`. The parameterized page breaks preserve original values and order.
- `00 80` and `FF FF` act as observed dialogue-ending or separating values in this scope and remain protected unless the selected short-dialogue policy places padding after an established ending.
- A four-byte aligned value that falls inside the file is not automatically a pointer. Earlier heuristic adjustment corrupted ordinary command arguments.
- Only three currently verified command-internal references are adjusted outside the two structural tables: `M_C18100_0001` slot `0x03CC`, and `M_I44000_0001` / `M_I44001_0001` slot `0x037C`. Their current translated targets are file-specific overrides, not a reusable structural formula.

### Verification evidence

- Japanese and Spanish files with matching names: 3,857.
- Files successfully parsed as offset containers: 437; all retained matching section counts, identifiers, and pointer populations.
- Compared sections: 2,230; compared inner pointers: 23,768; structural population mismatches: zero.
- Full MAP15 reinsertion scope: 3,857 files, 389 matched sheets, 8,076 changed dialogues; reported offset/header/protected-control failures: zero.
- Type 6 verification reparses the final file and checks every section start, section size, and inner target against the original plus the complete delta map. It also checks protected bytes outside translated ranges.
- Automated record reports include 26 tests and 792 lower-level round trips. These establish the current static/tool scope, not every runtime route.

### Improvement backlog and promotion conditions

Keep Type 6 provisional until later work addresses the relevant items below:

1. Replace the three manual embedded targets with consumer-proven structural anchors or document why a fixed per-revision catalog is the permanent specification.
2. Make width overflow an immediate hard failure at calculation time, especially the 16-bit section size, rather than relying on a skipped write to be caught by final verification.
3. Establish the complete population of direct strings or dialogue variants that may not satisfy the current dual-pointer header/body rule.
4. Runtime-test short-dialogue policies, especially post-terminator `FF` padding and un-terminated direct strings, across representative menus, choices, paging, and transitions.
5. Bind decompressed-file validation to the final compressed member, outer container offsets and sizes, and emulator consumption in one reproducible build proof.
6. Record the supported source revision hashes and make every manual exception revision-specific.

Do not transfer the section-header constants, 12-byte dialogue header, control values, or manual slots to another title or revision without re-establishing them.

## Type 7 — 4-byte ID plus 4-byte cumulative text length (provisional)

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

## Type 8 — reordered 4-byte cumulative offset references (provisional)

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

## Evidence standard for future additions

Add a type or variant only with all of the following:

- fixed source identity and exact analyzed extent;
- byte diagram, widths, endianness, and coordinate origins;
- entry-count and boundary authority, including empty or shared cases;
- equations that reproduce stored values and total layout;
- propagation rules for a controlled length change;
- constraints and explicitly counted exceptions; and
- unchanged full-extent round-trip evidence plus tested rejection of overflow or malformed input where applicable.

Do not infer a universal engine format from one file. Keep file names, addresses, counts, and measurements as evidence for the relation; re-establish them on another revision or title.


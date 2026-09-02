# Dialogue-expansion Type router

Read this file first and choose by boundary authority and propagation, not by game title. Inspect the target bytes before choosing. Read only the selected Type file; read two only when the evidence remains genuinely ambiguous.

| Type | Discriminator | Coordinate or length basis | Changed-length propagation | Status |
|---|---|---|---|---|
| 1 | Repeating `[len:u8][text:len]` stream | Per-record byte length | Rewrite that length; later positions emerge from serialization | Verified |
| 2 | Header plus `u16le` cumulative table and null-terminated text | Text-area relative cumulative next starts | Recompute later cumulative values and text-area size | Verified |
| 3 | Count plus `u32le` start table | File-absolute starts | Recompute payload starts, including duplicate or empty entries | Verified |
| 4 | Outer absolute block table plus inner `u16le` field starts | File absolute outside, record local inside | Recompute local starts and later outer block starts | Verified |
| 5 | Lua 5.1 chunk with 4-byte `size_t` string length including null | Per-string serialized size | Rewrite Lua string size and preserve chunk grammar | Verified |
| 6 | Top-level section directory plus section-relative pointer tables | File-absolute section starts and section-relative targets | Recompute section sizes, later starts, and affected inner targets | Provisional |
| 7 | Paired `[text_id:u32le][cumulative_end:u32le]` entries | Text-area relative cumulative ends | Preserve IDs and order; rebuild cumulative ends | Provisional |
| 8 | Independent `u32le` slots reuse cumulative boundaries in consumer-defined order | Text-area relative values in a reordered reference array | Preserve slot-to-boundary identity; rewrite each referenced boundary | Provisional |
| 9 | Repeating `[tag:4][payload_size:u32le][payload]` plus aggregate wrappers | Chunk-local lengths and enclosing extents | Rewrite payload length, later placement, and every enclosing size | Provisional |

## Fast rejection and ambiguity rules

- A small value before text is not a length until consecutive layout equations reproduce real boundaries.
- Monotonic values are not sufficient evidence for a cumulative or pointer table.
- A four-byte value before text is Type 5 only when the enclosing file is a compatible Lua 5.1 binary chunk.
- Type 7 requires an actual paired ID field. Type 8 has independent reference slots and may be reordered or repeated.
- Type 9 requires a structural chunk walk to EOF; ASCII marker searches alone do not establish it.
- If no relation uniquely explains the declared extent, report `NOT YET ESTABLISHED` and state the next evidence needed instead of reading all Type files speculatively.

## Output after selection

Report the selected Type, coordinate basis, authoritative population, affected fields, capacity constraints, round-trip status, controlled-growth status, and unresolved evidence. When implementing spreadsheet interchange, keep stable entry IDs and original coordinates as protected evidence; calculate new addresses and derived values from final serialized bytes during reinsertion.


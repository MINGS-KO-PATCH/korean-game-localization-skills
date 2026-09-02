# Type 9 — tagged nested 4-byte length chunks (provisional)

### Identification and layout

The Tengoku-hen dictionary sample uses a 32-byte fixed header followed by sequential tagged chunks:

```text
file header: 0x20 bytes

chunk:
  tag:4 bytes ASCII
  payload_size:u32le
  payload:payload_size bytes

next_chunk = chunk_start + 8 + payload_size
```

The sample `ID00001` is 2,232 bytes (`0x8B8`) and has SHA-256 `668bb9c73eb8aa1013378557cf9f9e2d9eaa24ed13787187e0e01e8719dd995f`. A structural walk from `0x20` reaches EOF exactly:

| Start | Tag | Payload size | Payload extent |
|---:|---|---:|---:|
| `0x0020` | `CHFN` | `0x0E` | `[0x0028,0x0036)` |
| `0x0036` | `CHNN` | `0x06` | `[0x003E,0x0044)` |
| `0x0044` | `PRDC` | `0x12` | `[0x004C,0x005E)` |
| `0x005E` | `ACTR` | `0x10` | `[0x0066,0x0076)` |
| `0x0076` | `VOIC` | `0x08` | `[0x007E,0x0086)` |
| `0x0086` | `LOOK` | `0x08` | `[0x008E,0x0096)` |
| `0x0096` | `DSCR` | `0x409` | `[0x009E,0x04A7)` |
| `0x04A7` | `DSC2` | `0x409` | `[0x04AF,0x08B8)` |

`DSCR` therefore ends exactly at the `DSC2` tag, and `DSC2` ends exactly at EOF. Identify chunks by walking the length equations from the established start, not by searching the payload for tag byte sequences.

### Enclosing aggregate sizes

The header carries two nested aggregate lengths:

```text
0x10: "DSIZ"
0x14: dsiz_size:u32le = file_size - 0x18

0x18: "DATA"
0x1C: data_size:u32le = file_size - 0x20
```

For `ID00001`, `dsiz_size=0x8A0=2,208` and `data_size=0x898=2,200`. These equal the bytes from their respective payload origins through EOF.

### Rewrite and propagation rules

Serialize edited payload bytes first and derive every length from that final byte string:

```text
chunk.payload_size = len(serialized_payload)
next_chunk_start = chunk_start + 8 + chunk.payload_size
data_size = final_file_size - 0x20
dsiz_size = final_file_size - 0x18
```

A changed early payload relocates all later chunks even though later chunk-local lengths remain unchanged. Reparse the rebuilt file from `0x20`, require every tag/length/payload extent to remain within the file, and require the final chunk end to equal EOF. Preserve the fixed header fields whose meanings are not established.

### Historical scripts and required corrections

- `STEP_6_길이#1.PY` estimates text length as two bytes per character except `㊥` as one byte and writes the result to the workbook. This is not the authoritative rule because it does not tokenize `[XX]` as one byte and can disagree with the reinserter on encoding failures.
- `STEP_6_길이#2.PY` rewrites `DSCR` and `DSC2` sizes by searching for their ASCII markers. Replace this with the structural chunk walk; marker bytes may occur inside payloads.
- `STEP_7_RE.PY` copies the first 32 bytes, then writes workbook-supplied code, length, and encoded text. It must derive and verify `payload_size == len(text_bytes)` during serialization rather than trusting the workbook length cell.
- `STEP_8_전체크기.PY` writes `file_size-24` at `0x14` and `file_size-32` at `0x1C`; these formulas are verified for this sample layout.
- The reinserter encodes `[XX]` as one raw byte, `㊥` as `0A`, selected Roman numerals as fixed Shift-JIS pairs, and pads ordinary one-byte Shift-JIS characters with `5E`. Its current fallback writes `5E` on encoding failure. A safe successor must fail on unmapped input instead of silently substituting a byte.

### Provisional limits and promotion conditions

Keep Type 9 `CONDITIONAL` until the target revision satisfies all applicable items:

1. Establish whether the 32-byte header and the `DSIZ`/`DATA` nesting are invariant across the complete file population.
2. Confirm the authoritative chunk population and permitted tag set, including optional, repeated, empty, or reordered chunks.
3. Use one serializer for both payload bytes and stored lengths; remove the separate character-count approximation.
4. Reject malformed lengths, integer overflow, unknown encodings, accidental tag matches, missing mandatory chunks, and final extents that do not equal EOF.
5. Prove unchanged full-file round trips and controlled growth for an early chunk, verifying later chunk relocation and both aggregate sizes.
6. Verify representative dictionary display and lookup routes in the runtime consumer and any enclosing archive capacity.

Do not classify this as Type 5 merely because both use a four-byte length. Type 5 is a Lua 5.1 serialized string grammar; Type 9 is a generic tagged chunk stream with multiple enclosing aggregate sizes.



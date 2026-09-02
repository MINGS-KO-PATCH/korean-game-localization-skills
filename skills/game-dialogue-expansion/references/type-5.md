# Type 5 — Lua 5.1 4-byte size-prefixed strings

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



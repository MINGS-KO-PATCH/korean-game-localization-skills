# Type 6 — two-level section directory with aligned dialogue blocks (provisional)

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



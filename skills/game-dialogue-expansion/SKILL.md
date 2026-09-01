---
name: game-dialogue-expansion
description: >-
  Expand game dialogue safely by analyzing and rebuilding length fields,
  cumulative offsets, absolute pointer tables, and nested record-local offsets.
  Use for Korean-patch work when translated dialogue changes byte length or an
  unknown binary text layout must be identified before reinsertion. 게임 한글화의
  대사 확장과 오프셋·포인터·길이 필드·중첩 구조 재계산에 사용한다.
---

# Expand game dialogue safely

Expand dialogue only after determining how each byte-length change propagates through the containing structure. Treat the four documented types as verified expansion patterns and comparison models, not universal format names.

Read [references/dialogue-expansion-types.md](references/dialogue-expansion-types.md) whenever identifying, extracting, rebuilding, or expanding dialogue. It contains Types 1–5 as verified patterns and Type 6 as a provisional pattern with an explicit improvement backlog.

## Working rules

- Identify the coordinate system of every value: file absolute, block relative, text-area relative, or record-local.
- Distinguish a length from a start position by testing consecutive records and exact layout equations. Monotonic values alone are insufficient.
- Establish the authoritative boundary source. Prefer the verified length or pointer population over delimiter scanning; scanning can miss empty or shared entries.
- Model every dependency layer affected by changed byte length. A nested structure may require both local offsets and outer pointers to be rebuilt.
- Preserve headers, metadata, duplicate pointers, empty entries, terminators, padding, and exceptional records unless their meanings and rewrite rules are established.
- Rebuild derived values from final serialized output. Do not accumulate patches against already modified offsets.
- Reject unrepresentable length, offset, block-capacity, or container-capacity values instead of truncating or wrapping them.
- Require an unchanged extract-and-reinsert round trip before translated reinsertion. Compare the complete declared byte range, not only decoded text.

## Adding a verified type

Append it to the integrated reference only after recording:

1. a discriminating byte-layout equation and coordinate basis;
2. the authoritative entry population and boundary source;
3. exact rewrite propagation when one entry changes length;
4. width, capacity, alignment, container, and exceptional-record constraints;
5. concrete measurements from a fixed source revision; and
6. unchanged round-trip and failure-path evidence.

Keep target-specific constants in the evidence section. Promote only the reusable relation to the type definition. If a new case differs in dependency propagation or boundary authority, record a new type or variant instead of forcing it into the nearest pattern.

Do not present a provisional type as generally complete. Preserve its known-good scope, unresolved runtime claims, hard-coded exceptions, and promotion conditions until later evidence closes them.


---
name: game-dialogue-expansion
description: >-
  Analyze and implement game-dialogue byte-length expansion by rebuilding
  length fields, offsets, pointers, and nested sizes. Use this focused skill
  alone for dialogue expansion, binary text-layout classification, or an
  expansion-safe extractor/reinserter. Do not co-apply create-kr-patch to the
  same structure analysis; use create-kr-patch instead for fonts, graphics,
  hooks, full patch builds, or end-to-end localization. 게임 대사 확장·오프셋
  분석과 확장형 추출기·재삽입기 제작에 단독으로 사용한다.
---

# Expand game dialogue safely

Expand dialogue only after determining how each byte-length change propagates through the containing structure. Treat the documented types as comparison models, not universal format names.

## Exclusive routing and selective references

Do not load `create-kr-patch` for the same dialogue-structure analysis. Use this skill alone when the requested result is expansion feasibility, Type classification, offset or length rebuilding, a dialogue extractor/reinserter, or spreadsheet interchange for dialogue. Use `create-kr-patch` alone when the request is broader and primarily concerns fonts, graphics, executable hooks, complete patch production, emulator QA, or release management. In a full localization project, finish the bounded dialogue-expansion subtask here and return only its findings and artifacts to the broader workflow.

For every new binary, read [references/type-index.md](references/type-index.md) first. It is the only reference loaded before classification. Then read only the one matching Type file, or at most two Type files when the evidence genuinely leaves two candidates. Do not read every Type file and do not read the legacy integrated reference during ordinary analysis.

| Candidate | Detailed reference |
|---|---|
| Type 1 | [references/type-1.md](references/type-1.md) |
| Type 2 | [references/type-2.md](references/type-2.md) |
| Type 3 | [references/type-3.md](references/type-3.md) |
| Type 4 | [references/type-4.md](references/type-4.md) |
| Type 5 | [references/type-5.md](references/type-5.md) |
| Type 6 | [references/type-6.md](references/type-6.md) |
| Type 7 | [references/type-7.md](references/type-7.md) |
| Type 8 | [references/type-8.md](references/type-8.md) |
| Type 9 | [references/type-9.md](references/type-9.md) |

## Expansion feasibility gate

Before extracting dialogue for translation, determine whether byte-length expansion can be rebuilt safely. Do not begin bulk extraction merely because text can be decoded.

1. Identify the authoritative entry population and boundary source: length field, cumulative table, absolute pointer table, nested table, or consumer-defined structure.
2. Determine each value's width, endianness, origin, and meaning. Distinguish IDs from positions and lengths from offsets.
3. Predict every field affected when one representative string grows and shrinks. Include inner records, sections, compressed members, archives, filesystem metadata, load buffers, and runtime consumers when present.
4. Prove an unchanged extraction-and-reinsertion round trip over the declared extent.
5. Perform a controlled one-entry length change and verify all derived values, final readback, and the real consumer path when available.

Report the gate as one of:

- `EXPANDABLE`: the population, propagation rules, capacity, rebuild path, and verification scope are established.
- `CONDITIONAL`: a bounded expansion policy is established, but named capacity, container, exception, or runtime conditions remain.
- `NOT YET ESTABLISHED`: decoding or extraction works, but safe length-changing reinsertion is not proven.

Only scale extraction for translation after `EXPANDABLE`, or after the user explicitly accepts the stated limitations of `CONDITIONAL`. Never report `NOT YET ESTABLISHED` as impossible; state the next evidence needed.

## Working rules

- Identify the coordinate system of every value: file absolute, block relative, text-area relative, or record-local.
- Distinguish a length from a start position by testing consecutive records and exact layout equations. Monotonic values alone are insufficient.
- Establish the authoritative boundary source. Prefer the verified length or pointer population over delimiter scanning; scanning can miss empty or shared entries.
- When the user does not know a pointer-table address, derive candidate extents from established text starts and boundary relations. For a table immediately before text, scan backward in field-width steps while every value satisfies the type's allowed set, then validate the stopping boundary and full extent. Never require a user-supplied address when it can be derived, and never guess one when the evidence is ambiguous.
- When another game is proven to match a registered type, use that relation to build or adapt an expansion-safe extractor and reinserter. When spreadsheet exchange is requested, export stable entry identity, protected structural fields, original text, translation text, and derived-value evidence, then recompute lengths and offsets from final serialized bytes during reinsertion rather than trusting editable spreadsheet formulas or addresses.
- Model every dependency layer affected by changed byte length. A nested structure may require both local offsets and outer pointers to be rebuilt.
- Preserve headers, metadata, duplicate pointers, empty entries, terminators, padding, and exceptional records unless their meanings and rewrite rules are established.
- Rebuild derived values from final serialized output. Do not accumulate patches against already modified offsets.
- Reject unrepresentable length, offset, block-capacity, or container-capacity values instead of truncating or wrapping them.
- Require an unchanged extract-and-reinsert round trip before translated reinsertion. Compare the complete declared byte range, not only decoded text.

## Adding a verified type

Add or update its individual Type reference and the short index only after recording:

1. a discriminating byte-layout equation and coordinate basis;
2. the authoritative entry population and boundary source;
3. exact rewrite propagation when one entry changes length;
4. width, capacity, alignment, container, and exceptional-record constraints;
5. concrete measurements from a fixed source revision; and
6. unchanged round-trip and failure-path evidence.

Keep target-specific constants in the evidence section. Promote only the reusable relation to the type definition. If a new case differs in dependency propagation or boundary authority, record a new type or variant instead of forcing it into the nearest pattern.

Do not present a provisional type as generally complete. Preserve its known-good scope, unresolved runtime claims, hard-coded exceptions, and promotion conditions until later evidence closes them.


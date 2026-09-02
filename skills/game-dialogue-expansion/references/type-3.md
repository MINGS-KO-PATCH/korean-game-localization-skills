# Type 3 — 4-byte absolute starts

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



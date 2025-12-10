# **FID Specification**

**Pronounced faɪˈdiː**

**Version 0.1 — 2025-12-09**

**Status: Informational**

FID is a compact, time-ordered, human-friendly, coordination-free 64-bit identifier format.

It provides:

* Chronological ordering within a semantic *kind*
* Local, deterministic, collision-free generation (no central authority)
* Minimal binary and textual footprint
* URL-safe, human-readable Base32 encoding

FID fits within a single unsigned 64-bit integer while preserving structure suitable for introspection and traceability.

---

# 1. Purpose

FID is intended for distributed systems that require:

* Time-sortable identifiers
* Independent generation across nodes
* Compact cross-system serialization
* Manual transcribability and URL safety

The format encodes both temporal and structural information while remaining small enough to embed directly in databases, network protocols, and event logs.

---

# 2. Bit Layout and Field Semantics

### **64-bit layout**

```
[kind:8 | time:43 | node:8 | counter:5]
```

| Field       | Bits | Description                                       |
| ----------- | ---- | ------------------------------------------------- |
| **kind**    | 8    | Category / entity type for grouping and ordering. |
| **time**    | 43   | Milliseconds since fixed epoch (2025-05-15 UTC).  |
| **node**    | 8    | Generator node identifier (0–255).                |
| **counter** | 5    | Monotonic counter within a millisecond (0–31).    |

### Field Notes

* **Kind**
  Provides stable grouping. Ordering is meaningful *within* a kind and meaningless *across* kinds.

* **Time**
  Represents milliseconds since epoch.
  Range: ~278 years (valid until ~2303 CE).

* **Node**
  Differentiates independent generators running concurrently.

* **Counter**
  Ensures per-node monotonicity during bursts (up to 32 IDs/ms).
  If the counter would overflow, the generator **must wait** for the next millisecond.

### Clock Behavior

If the system clock moves backward, generators **SHOULD** delay issuance until the clock catches up.
FID **MUST NOT** encode a timestamp earlier than the last emitted millisecond.

### Global Capacity

* Per node: **32 FIDs per ms**
* Across 256 nodes: **8192 FIDs per ms** (theoretical)

### Bit Representation

* Bits numbered MSB (bit 63) → LSB (bit 0)
* All arithmetic uses unsigned 64-bit big-endian interpretation

---

# 3. Epoch

```
epoch = 2025-05-15T00:00:00Z
```

* All timestamps are milliseconds since this epoch.
* Timestamps before the epoch are invalid.
* Format overflows around year 2303 CE.

---

# 4. Text Encoding

FID values are encoded in **Crockford Base32**:

```
<13-character body><1-character check digit>
```

### Rules

* Canonical textual FIDs contain **exactly 14 characters**.
* Body is zero-padded Base32 representation of the 64-bit value.
* Check digit = `value % 37`, indexing the check alphabet.

### Alphabets

* **Body alphabet:**
  `0123456789ABCDEFGHJKMNPQRSTVWXYZ`

* **Check alphabet:**
  `0123456789ABCDEFGHJKMNPQRSTVWXYZ*~$=U`

### Normalization

Parsers must be case-insensitive and may ignore: `- _ .  `
Characters `O → 0`, `I → 1`, `L → 1`.

### Example

```
FID: 04P7Z8K5GHN2M2U
```

---

# 5. Node Identification

The *node* field identifies the local generator instance.

The specification does not prescribe a specific method, but requires:

1. **Local Uniqueness**
   Distinct generators within a deployment should use distinct node IDs.

2. **Determinism**
   Node IDs should remain stable across restarts of the same generator.

3. **Range**
   Node IDs are one byte: `0–255`.

4. **Derivation Methods**
   May include:

   * Environment variable / configuration
   * Hash of machine ID or hostname
   * Container/runtime identity
   * Manual assignment

If two generators use the same node ID, collisions are possible and undetectable.

---

# 6. Ordering

FID values sort as:

1. By **kind**
2. Then by **time**
3. Then by **node**
4. Then by **counter**

Properties:

* FIDs of the same kind sort in strict chronological order.
* Ordering across kinds is **not** semantically meaningful.
* FIDs are naturally ordered when treated as `uint64`.

This yields stable, time-grouped buckets per kind.

---

# 7. Binary and Text Forms

* **Binary**: 8-byte big-endian unsigned integer (no check digit).
* **Text**: 14-character canonical Base32 form (includes check digit).

Text canonicalization rules apply only to the textual form.

---

# 8. Error Detection

Implementations **must** reject invalid input.

### Required Validation

1. **Length**
   After normalization, the string must contain **14 characters**.

2. **Alphabet Compliance**
   All body characters must be in the Crockford Base32 alphabet.

3. **Check Digit**
   The final character must match the computed check digit.

### Behavior

* Invalid FIDs must be rejected.
* Implementations **should not** attempt automatic correction.

---

# 9. Example Breakdown

Example FID:

```
0C3RD4GJQ0AK8Y
```

Numeric payload:

```
0x05F57F48D13F4288
```

Field extraction:

```
kind    = (value >> (43 + 8 + 5)) & 0xFF
time    = (value >> (8 + 5))      & ((1 << 43) - 1)
node    = (value >> 5)            & 0xFF
counter =  value                  & 0x1F
```

Values in this example are illustrative only.

---

# 10. Security Considerations

FID provides:

* uniqueness
* ordering
* structure

FID does **not** provide:

* secrecy
* authentication
* access control
* collision resistance under malicious conditions

FID **must not** be used as a credential or authorization token.

---

# 11. Reference Implementations

Official implementations:

* **Go:** [https://github.com/fid-project/fid-go](https://github.com/fid-project/fid-go)
* Additional languages (Python, Node, Rust, Java) planned.

---

# 12. Version and Status

* **Specification Version:** 0.1
* **Status:** Informational
* **Date:** 2025-12-10

Updates will maintain backwards compatibility within major versions.

---

13. Conformance

Normative test vectors are provided in conformance/vectors.json.
Implementations should use these to verify packing, encoding, and check-digit behavior.

---

# License

This specification is licensed under the **Apache License 2.0**.
See `LICENSE` for details.

© 2025 Fred Bairn
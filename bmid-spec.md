# BMID — Book Metadata Identifier

**A structured, self-describing identifier for LRM entities**

Specification v1.0 — DRAFT | April 2026

---

## Executive Summary

The Book Metadata Identifier (BMID) is a structured, self-describing identifier designed for use in book metadata ecosystems aligned with the IFLA Library Reference Model (LRM). It addresses critical limitations of plain UUIDs by embedding routing metadata, source attribution, and error detection directly into the identifier itself.

A BMID is a 25-byte value encoded as a 40-character Crockford Base32 string. It is globally unique, instantly routable, and resilient to transcription and transmission errors. It is designed for distributed metadata systems where a small consortium of partner organizations independently mint identifiers that can coexist without runtime coordination.

---

## Scope

BMID is intended for entities that lack established standard identifiers — Works, Expressions, Manifestations, Items, Agents (contributors, publishers), Series, and Subjects under the LRM model. It does not replace ISBN, ISTC, ISNI, DOI, or other established identifiers. Those continue to be carried as parallel metadata on the records they apply to.

---

## The Problem with Plain UUIDs

UUIDs are excellent at one thing: global uniqueness. But in a book metadata ecosystem, uniqueness alone is not enough. Consider the following real-world scenarios that a plain UUID cannot address:

- **Routing:** A user pastes an identifier into a search field. The system must determine whether this refers to a Work, an Agent, a Series, or a Manifestation. With a UUID, a lookup against every entity table is required. With a BMID, the type is encoded in the prefix — routing is instantaneous.

- **Source tracing:** When multiple partners supply metadata, knowing which partner created an identifier is essential for conflict resolution, quality assessment, and provenance tracking. A UUID carries no such information.

- **Environment awareness:** Metadata systems typically run in parallel environments (production, staging, development). When an identifier from one environment is looked up in another, plain UUIDs can only answer "not found," which is ambiguous. BMIDs carry an environment flag so consumers can answer "not found — this identifier belongs to a staging environment," which is actionable.

- **Error detection:** When identifiers are transmitted across systems, copy-pasted, or manually entered, corruption is inevitable. UUID has no built-in detection. BMID includes a 2-byte CRC that catches transcription errors, truncation, and bit-flips.

- **Readability:** Hexadecimal UUIDs are prone to transcription errors (0/O, 1/l, 8/B). Crockford Base32 eliminates ambiguous characters and is case-insensitive — a BMID is designed to survive transcription, verbal communication, and OCR.

---

## Identifier Structure

A BMID consists of 25 bytes (200 bits) organized into seven fields. The non-UUID fields precede the payload to enable prefix-based routing and sharding on the decoded byte array without reading the entire identifier.

### Byte Layout

| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | **Version** | 1 byte | BMID format version. Allows future evolution without breaking existing parsers. |
| 1 | **Entity Type** | 1 byte | LRM entity class. See *Entity Type Registry*. |
| 2 | **Vendor ID** | 1 byte | Partner organization that minted the identifier. See *Vendor Registry*. |
| 3 | **Environment** | 1 byte | Operational environment. See *Environment Registry*. |
| 4–6 | **Reserved** | 3 bytes | Reserved for future use. Must be `0x00 0x00 0x00` in version 1. |
| 7–22 | **UUID Payload** | 16 bytes | A UUIDv7 (RFC 9562). Mandatory. |
| 23–24 | **Check Bytes** | 2 bytes | CRC-16/XMODEM over bytes 0–22, big-endian. |

**Total:** 25 bytes = 200 bits = 40 Crockford Base32 characters.

### Canonical Form

The canonical form of a BMID is exactly **40 Crockford Base32 characters**, uppercase, with no separators or surrounding whitespace. Two BMIDs are equal if and only if their canonical forms are byte-equal. Implementations must normalize input to canonical form before comparison, hashing, or use as a key.

There is no hyphenated or segmented display form. Field boundaries fall on byte offsets in the decoded buffer, not on character offsets in the encoded string. **Implementations parse fields by Base32-decoding the entire string and indexing into the resulting 25-byte array.**

### Worked Example

A Manifestation minted by the testing vendor (`0xFF`) in the production environment:

```
041ZY080000033SD9A600W93K8NKRKAYDXX8PHR4
```

Decoded bytes (hex):

```
01 03 FF 01 00 00 00  01 8F 2D 4A 8C 00 71 23 9A 2B 3C 4D 5E 6F 7A 8B  47 04
```

| Bytes | Value | Meaning |
|-------|-------|---------|
| `[0]` | `0x01` | Version = 1 |
| `[1]` | `0x03` | Entity type = Manifestation |
| `[2]` | `0xFF` | Vendor ID = testing |
| `[3]` | `0x01` | Environment = Production |
| `[4–6]` | `0x00 0x00 0x00` | Reserved |
| `[7–22]` | `01 8F 2D 4A 8C 00 71 23 9A 2B 3C 4D 5E 6F 7A 8B` | UUIDv7 payload |
| `[23–24]` | `0x47 0x04` | CRC-16/XMODEM = `0x4704` |

### URN Form

A BMID may be expressed as a URN for use in RDF, BIBFRAME, and other linked-data contexts:

```
urn:bmid:041ZY080000033SD9A600W93K8NKRKAYDXX8PHR4
```

The URN form is optional. The canonical 40-character form is the authoritative representation.

---

## Registries

The entity type, vendor ID, and environment fields are each allocated from a registry maintained as part of this specification. **New allocations are made by submitting a pull request against this document** with the requested code, the name of the requesting organization, and a contact address.

### Entity Type Registry

| Code | Entity | Description |
|------|--------|-------------|
| `0x00` | *Reserved* | Invalid; must not be used. |
| `0x01` | **Work** | An abstract intellectual creation (IFLA LRM). |
| `0x02` | **Expression** | A realization of a Work in a specific form. |
| `0x03` | **Manifestation** | A physical or digital embodiment. |
| `0x04` | **Item** | A specific exemplar of a Manifestation. |
| `0x05` | **Agent** | A person or corporate body (contributor, publisher). |
| `0x06` | **Series** | An ordered collection of related Works. |
| `0x07` | **Subject** | A topic, theme, or classification heading. |
| `0x08–0x0F` | *Reserved* | Headroom for additional core LRM types. |
| `0x10–0x7F` | *Reserved* | Reserved for future IFLA/LRM-aligned types. |
| `0x80–0xFE` | *Available* | Registry-assigned extensions. |
| `0xFF` | *Reserved* | Reserved for testing and examples. |

### Vendor Registry

| Code | Vendor |
|------|--------|
| `0x00` | *Reserved — invalid* |
| `0x01–0xFE` | *Available — registry-assigned* |
| `0xFF` | *Reserved for testing and examples* |

(No partner organizations are allocated in this draft.)

The vendor ID identifies the **minter** at the moment of creation. It is fixed and represents provenance — not authoritative current ownership or custody. Systems that need to track current custody (after acquisitions, rights transfers, or catalog resale) must carry that information in a separate mutable metadata field.

### Environment Registry

| Code | Environment |
|------|-------------|
| `0x00` | *Reserved — invalid* |
| `0x01` | Production |
| `0x02` | Staging |
| `0x03` | Development |
| `0x04` | Test |
| `0x05–0xFF` | *Reserved* |

---

## Encoding

### Crockford Base32

BMID uses Crockford Base32 encoding. This encoding was chosen for the following properties:

- Case-insensitive on input; canonical output is uppercase
- No ambiguous characters: excludes I, L, O, and U to prevent confusion with 1, 0, and V
- URL-safe and filename-safe without escaping
- Compact: 40 characters for 200 bits (compared to 50 in hex)
- 5 bits per character aligns perfectly with the 200-bit total — no padding

On input, implementations must accept both upper and lower case, may map `I`/`i`/`L`/`l` to `1` and `O`/`o` to `0` (per Crockford), and must reject any characters outside the alphabet.

### Check Bytes

The trailing 2 bytes are a **CRC-16/XMODEM** checksum computed over the first 23 decoded bytes (version through UUID payload), stored big-endian.

CRC-16/XMODEM parameters:

| Parameter | Value |
|-----------|-------|
| Polynomial | `0x1021` |
| Initial value | `0x0000` |
| Reflect input | false |
| Reflect output | false |
| XOR output | `0x0000` |

**Test vector:** CRC-16/XMODEM of the ASCII string `"123456789"` is `0x31C3`.

The CRC algorithm and parameters are **invariant across all versions of this specification.** Future version bumps may change any other field, but CRC computation remains bit-compatible. This eliminates a chicken-and-egg situation in validation: CRC validation is always possible without first parsing the version byte.

The CRC provides:

- Detection of any single-character transcription error
- Detection of transposed adjacent characters
- Detection of truncation or padding errors
- A 1-in-65,536 false-acceptance rate for random corruption

### UUID Payload

The 16-byte payload is a **UUIDv7** (RFC 9562). UUIDv7 is mandatory in BMID v1, for the following reasons:

- The embedded millisecond timestamp enables temporal ordering and time-range queries directly from the identifier
- K-sortable insertion order substantially improves B-tree and index locality compared to random UUIDv4
- The timestamp is extractable for audit and debugging without external metadata
- A consistent UUID version across the ecosystem allows downstream systems to rely on these properties

Trade-offs accepted by mandating UUIDv7:

- The minting time is observable to anyone who holds the BMID. For book metadata this is rarely sensitive, but consider embargoes or pre-announcement workflows.
- Slightly less entropy than UUIDv4 (62 random bits vs. 122). This remains safe for uniqueness; do not rely on UUID values for unguessability in any case.
- Within-millisecond burst minting requires a UUIDv7 implementation that handles monotonicity correctly.

---

## Comparison with Plain UUIDs

| Capability | BMID | Plain UUID |
|------------|------|-----------|
| Entity type routing | Encoded in prefix | Not supported |
| Source attribution | Vendor ID field | Not supported |
| Environment awareness | Built-in flag | Not supported |
| Error detection | CRC-16 check bytes | None |
| Temporal ordering | Mandatory UUIDv7 | UUIDv7 only if chosen |
| Global uniqueness | UUID-based | UUID-based |
| Transcription resilience | Crockford Base32 + CRC | Hex (easy errors) |
| Independent minting | Native multi-vendor | No coordination |
| Prefix-based sharding | Natural support | Random distribution |
| Self-describing | Yes (version + type) | No |

---

## Key Advantages

### Instant Routing from a Search Field

When a user pastes a BMID into a search field, the application can immediately determine the entity type from the prefix and route the query to the correct service or database partition. There is no need for a multi-table lookup, a central registry, or any network call to determine what kind of entity the identifier refers to.

### Coordinated Federation with Independent Minting

BMIDs are minted independently by a small, explicitly registered set of partner organizations. Global uniqueness is guaranteed by the UUIDv7 payload without runtime coordination or central authority in the mint path. Provenance is preserved by the vendor ID field, whose values are allocated from the registry maintained with this specification.

This model fits trusted-consortium metadata ecosystems on the order of tens of partners. It does not claim to be an open, zero-coordination system — vendor and entity-type allocations are coordinated at specification-evolution time, not at mint time. Partners submit a PR to request an allocation.

### Environment-Aware Lookup Errors

The environment flag allows a system that receives a foreign-environment BMID to return a clear, actionable error ("this identifier belongs to a staging environment") instead of an ambiguous "not found." This is a diagnostic aid, not a security boundary. Environment isolation must continue to be enforced at the network, credential, and data-plane layers — the flag improves the diagnostic experience when those boundaries leak.

### Self-Describing and Future-Proof

The version byte ensures that the identifier format can evolve without breaking existing parsers. A system encountering a BMID with an unknown version can reject it gracefully or route it to a version-aware handler. The reserved bytes allow modest field additions in future versions.

### Designed to Survive Transcription, Verbal Communication, and OCR

Crockford Base32 excludes the most commonly confused characters (I/1, L/1, O/0, U/V) and is case-insensitive. Combined with the CRC, a BMID communicated over a phone call, printed on a shipping label, or extracted via OCR will either arrive intact or fail validation — silent corruption is unlikely.

### Database and Infrastructure Friendly

The fixed-length, prefix-structured format is naturally suited to partitioning. Identifiers for the same entity type and vendor cluster together, improving cache locality and enabling prefix-based sharding. Stored as the raw 25 bytes, the overhead compared to a 16-byte UUID is modest and is offset by the elimination of auxiliary lookup tables for type, source, and environment.

---

## Implementation Notes

### Storage

BMIDs can be stored in two forms:

- **Binary (25 bytes):** Most compact. Suitable for database primary keys and binary protocols.
- **Crockford Base32 (40 chars):** Canonical string representation. Use for APIs, logs, URLs, and user-facing display.

There is no other display form. Hyphens, spaces, and other separators are not part of the BMID format.

### Validation

A conforming implementation must validate the following on input, in this order:

1. **Length:** exactly 40 Base32 characters after stripping whitespace and normalizing case.
2. **Character set:** all characters must be valid Crockford Base32 symbols (after applying optional `I`/`L`→`1`, `O`→`0` normalization).
3. **CRC:** Base32-decode the input, recompute CRC-16/XMODEM over the first 23 bytes, and compare to bytes 23–24.
4. **Version:** the version byte must be a recognized version. Unknown versions should be rejected or routed to a version-aware handler.
5. **Entity type, vendor ID, environment:** codes should be within known ranges. Unknown codes may be accepted with a warning to support forward compatibility with newly-registered values.
6. **Reserved bytes:** in version 1, bytes 4–6 should be zero. Non-zero reserved bytes from a v1 producer indicate a bug or corruption; consumers may accept and ignore them for forward compatibility.

The CRC algorithm is fixed across all versions of the specification, so step 3 never depends on step 4.

### Revocation and Deprecation

BMIDs are **never reused or reassigned.** When an entity is withdrawn, merged into another, or otherwise deprecated, its record is marked deprecated and carries a pointer to the canonical successor BMID (if any). The identifier itself remains valid as a historical reference and continues to resolve to its deprecation record.

Implementations that resolve BMIDs must distinguish three states: live, deprecated-with-successor, and unknown. Deprecation handling lives in the metadata layer, not in the identifier itself.

### Why a Structured Identifier

The most common objection to structured identifiers is that a plain UUID is simpler and sufficient. This argument holds in systems with a single entity type, a single source of truth, and no need for human interaction with identifiers. In a federated book metadata ecosystem, none of these conditions hold:

- Multiple entity types share the same namespace and must be distinguished without a lookup
- Multiple partners mint identifiers independently and must be traceable to their source
- Identifiers appear in UIs, emails, support tickets, and printed materials where transcription matters
- Environments share infrastructure and must be diagnosable when crossed
- Data integrity across system boundaries demands built-in error detection

A BMID is not a replacement for UUID — it *contains* a UUID. It is a thin, fixed-cost envelope around a UUID that adds routing, provenance, diagnostic context, and integrity. The cost is 9 additional bytes. The value is the elimination of multiple auxiliary systems, lookup tables, and failure modes.

---

## Security Considerations

The CRC-16 check bytes are designed for error detection, not security. They do not protect against deliberate tampering. If cryptographic integrity is required, implementations should layer HMAC or digital signatures over the BMID rather than extending the identifier itself.

The vendor ID and environment fields are informational and must not be treated as access control mechanisms. Authorization decisions must be made by the receiving system based on its own trust model.

The vendor ID field reveals the original minter of an identifier, which may carry commercial sensitivity (sourcing relationships, catalog provenance). Partners exposing BMIDs to external consumers should consider whether this disclosure is acceptable in their use case.

The UUIDv7 payload reveals the millisecond timestamp at which the identifier was minted. This is generally not sensitive for book metadata but should be considered for embargoed or pre-announcement workflows.

---

## Glossary

| Term | Definition |
|------|------------|
| **BMID** | Book Metadata Identifier. The structured identifier format specified in this document. |
| **LRM** | IFLA Library Reference Model. The conceptual model for bibliographic entities (Work, Expression, Manifestation, Item, Agent). |
| **Crockford Base32** | A base-32 encoding alphabet designed by Douglas Crockford. Uses 0–9 and A–Z excluding I, L, O, U. |
| **CRC-16/XMODEM** | A 16-bit cyclic redundancy check with polynomial `0x1021`, initial value `0x0000`, and no input/output reflection. Used by BMID for error detection over the first 23 bytes. |
| **UUIDv7** | A UUID version defined in RFC 9562 that embeds a Unix millisecond timestamp for temporal ordering. Mandatory in BMID v1. |
| **Vendor ID** | A 1-byte identifier assigned from the central registry to a partner organization that mints BMIDs. |
| **Minter** | The vendor that originally created a BMID. Identified by the vendor ID field. Distinct from current custodian. |
| **Custodian** | The current owner of a record. Tracked outside the BMID in mutable metadata. |

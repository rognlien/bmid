# BMID — Book Metadata Identifier

**A structured, self-describing identifier for LRM entities**

Specification v1.0 — DRAFT | April 2026

---

## Executive Summary

The Book Metadata Identifier (BMID) is a structured, self-describing identifier designed for use in book metadata ecosystems aligned with the IFLA Library Reference Model (LRM). It addresses critical limitations of plain UUIDs and legacy identifiers like ISBN by embedding routing metadata, source attribution, and error detection directly into the identifier itself.

A BMID is a 25-byte value encoded as a 40-character Crockford Base32 string. It is globally unique, human-friendly, instantly routable, and tamper-evident. It is designed for modern, distributed, and federated metadata systems where multiple vendors must independently mint identifiers that can coexist without coordination.

---

## The Problem with Plain UUIDs

UUIDs are excellent at one thing: global uniqueness. But in a book metadata ecosystem, uniqueness alone is not enough. Consider the following real-world scenarios that a plain UUID cannot address:

- **Routing:** A user pastes an identifier into a search field. The system must determine whether this refers to a Work, an Agent, a Series, or a Manifestation. With a UUID, a lookup against every entity table is required. With a BMID, the type is encoded in the prefix — routing is instantaneous.

- **Source tracing:** When multiple vendors supply metadata, knowing which vendor created an identifier is essential for conflict resolution, quality assessment, and provenance tracking. A UUID carries no such information.

- **Environment safety:** Staging data accidentally entering production is a common and costly failure mode in distributed systems. BMIDs carry an environment flag that allows systems to reject cross-environment identifiers at the boundary.

- **Error detection:** When identifiers are transmitted across systems, copy-pasted, or manually entered, corruption is inevitable. ISBN has a single check digit. UUID has nothing. BMID includes a 2-byte CRC that catches transcription errors, truncation, and bit-flips.

- **Readability:** Hexadecimal UUIDs are prone to transcription errors (0/O, 1/l, 8/B). Crockford Base32 eliminates ambiguous characters and is case-insensitive, making BMIDs safer to communicate verbally, print, or type.

---

## Identifier Structure

A BMID consists of 25 bytes (200 bits) organized into six fields. The non-UUID fields precede the payload to enable prefix-based routing, sharding, and filtering without parsing the entire identifier.

### Byte Layout

| Field | Size | Description | Example |
|-------|------|-------------|---------|
| **Version** | 1 byte | Identifier format version. Allows future evolution of the encoding scheme without breaking existing parsers. | `0x01` |
| **Entity Type** | 2 bytes | Encodes the LRM entity class: Work, Expression, Manifestation, Item, Agent, Series, or custom domain types. Enables instant routing and type-aware processing. | `0x0001` |
| **Vendor ID** | 3 bytes | Identifies the organization or system that minted the identifier. Supports up to 16.7 million distinct sources in a federated ecosystem. | `0x00A1F3` |
| **Environment** | 1 byte | Operational environment flag. Distinguishes production, staging, development, and test identifiers to prevent cross-environment contamination. | `0x01` |
| **UUID Payload** | 16 bytes | Globally unique payload. UUIDv7 recommended for embedded timestamp and natural temporal ordering. UUIDv4 also supported. | — |
| **Check Bytes** | 2 bytes | CRC-16/CCITT or similar checksum computed over the preceding 23 bytes. Catches transcription errors, truncation, and corruption. | — |

**Total:** 25 bytes = 200 bits = 40 Crockford Base32 characters

### Wire Format

The encoded BMID is segmented with hyphens for human readability. The segments align with the semantic fields:

```
V-TTTT-PPPPPP-EE-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-CCCC
```

| Segment | Meaning |
|---------|---------|
| `V` | Version (2 chars, 1 byte) |
| `TTTT` | Entity type (4 chars, 2 bytes) |
| `PPPPPP` | Vendor ID (5–6 chars, 3 bytes) |
| `EE` | Environment flag (2 chars, 1 byte) |
| `XX…X` | UUID payload (26 chars, 16 bytes) |
| `CCCC` | Check bytes (4 chars, 2 bytes) |

**Example:** `2-0002-01N7R-04-01J3Q6S8G1K9V4B0DPWXYZ5MN0-3FA8`

### Entity Type Registry

The entity type field uses a 2-byte code space. The lower ranges are reserved for standard LRM entity types, while upper ranges are available for vendor-specific extensions.

| Code | Entity | Description |
|------|--------|-------------|
| `0x0001` | **Work** | An abstract intellectual creation (IFLA LRM) |
| `0x0002` | **Expression** | A realization of a Work in a specific form |
| `0x0003` | **Manifestation** | A physical or digital embodiment |
| `0x0004` | **Item** | A specific exemplar of a Manifestation |
| `0x0010` | **Agent** | A person or corporate body (contributor, publisher) |
| `0x0011` | **Series** | An ordered collection of related Works |
| `0x0020` | **Subject** | A topic, theme, or classification heading |
| `0x0100–0x0FFF` | *Reserved* | Reserved for future LRM extensions |
| `0x1000–0xFFFF` | *Vendor-defined* | Available for domain-specific entity types |

---

## Encoding

### Crockford Base32

BMID uses Crockford Base32 encoding exclusively. This encoding was chosen for the following properties:

- Case-insensitive: input can be upper, lower, or mixed case
- No ambiguous characters: excludes I, L, O, and U to prevent confusion with 1, 0, and V
- URL-safe and filename-safe without escaping
- Compact: 40 characters for 200 bits (compared to 50 characters in hex)
- 5 bits per character aligns perfectly with the 200-bit total (no padding needed)

### Check Bytes

The trailing 2 bytes are a CRC-16/CCITT checksum computed over the first 23 bytes (version through UUID payload). This provides:

- Detection of any single-character transcription error
- Detection of transposed adjacent characters
- Detection of truncation or padding errors
- A 1-in-65,536 false acceptance rate for random corruption

### UUID Payload Recommendations

The 16-byte payload is a standard UUID. Implementations are free to choose their UUID version, but UUIDv7 (RFC 9562) is strongly recommended for the following reasons:

- Embedded millisecond-precision timestamp enables temporal ordering of identifiers
- Natural sort order within the same entity type and vendor partition
- Improved database index locality compared to random UUIDv4
- Timestamp is extractable without external metadata, useful for debugging and auditing

UUIDv4 remains fully supported and interoperable. The version byte in the BMID header does not constrain the UUID version in the payload.

---

## Comparison with Existing Identifiers

| Capability | BMID | Plain UUID | ISBN / ISTC |
|------------|------|-----------|-------------|
| Entity type routing | Encoded in prefix | Not supported | Not supported |
| Source attribution | Vendor ID field | Not supported | Publisher prefix only |
| Environment safety | Built-in flag | Not supported | Not supported |
| Error detection | CRC-16 check bytes | None | Check digit (1 char) |
| Temporal ordering | UUIDv7 payload | UUIDv7 only if chosen | Not supported |
| Global uniqueness | UUID-based | UUID-based | Limited namespace |
| Human readability | Crockford Base32 | Hex (easy errors) | Numeric only |
| Federated minting | Native multi-vendor | No coordination | Central authority required |
| Prefix-based sharding | Natural support | Random distribution | Not designed for this |
| Self-describing | Yes (version + type) | No | Partially (format implies book) |

---

## Key Advantages

### Instant Routing from a Search Field

When a user pastes a BMID into a search field, the application can immediately determine the entity type from the prefix and route the query to the correct service or database partition. There is no need for a multi-table lookup, a central registry, or any network call to determine what kind of entity the identifier refers to. This is the single most impactful operational advantage over plain UUIDs.

### Federated Minting Without Coordination

Multiple vendors can independently generate BMIDs without a central authority. The UUID payload guarantees global uniqueness, while the vendor ID field preserves provenance. When metadata from different sources is merged, the vendor field enables source-aware conflict resolution, quality scoring, and audit trails. This is a fundamental requirement for any metadata ecosystem that spans organizational boundaries.

### Environment Boundary Enforcement

The environment flag allows systems to reject identifiers from non-matching environments at the API gateway level. A staging BMID can never silently corrupt a production database. This is a defensive design choice that eliminates an entire class of operational incidents that are common in distributed systems with shared identifier namespaces.

### Self-Describing and Future-Proof

The version byte ensures that the identifier format can evolve without breaking existing parsers. A system encountering a BMID with an unknown version can reject it gracefully or route it to a version-aware handler. This provides a clean upgrade path that plain UUIDs fundamentally cannot offer.

### Human-Friendly Encoding

Crockford Base32 is specifically designed for environments where identifiers are read, spoken, printed, or manually entered. The exclusion of ambiguous characters (I/1, O/0, L/1) and the case-insensitivity mean that a BMID communicated verbally over a phone call, printed on a shipping label, or typed from a screenshot will arrive intact. The appended CRC provides a safety net for the cases where it does not.

### Database and Infrastructure Friendly

The fixed-length, prefix-structured format is naturally suited to partitioning strategies. Identifiers for the same entity type and vendor cluster together, improving cache locality and enabling prefix-based sharding. When stored as the raw 25 bytes, the overhead compared to a 16-byte UUID is modest (56% increase) and is offset by the elimination of auxiliary lookup tables for type and source.

---

## Implementation Notes

### Storage

BMIDs can be stored in three forms depending on the context:

- **Binary (25 bytes):** Most compact. Suitable for database primary keys and binary protocols.
- **Crockford Base32 (40 chars):** Canonical string representation. Use for APIs, logs, and user-facing display.
- **Hyphenated (44–46 chars):** Human-readable form with hyphens between semantic segments. Hyphens are stripped during parsing.

### Validation

A conforming implementation must validate the following on input:

1. **Length:** exactly 40 Base32 characters (after stripping hyphens and normalizing case)
2. **Character set:** all characters must be valid Crockford Base32 symbols
3. **CRC:** recompute the CRC-16 over the first 23 decoded bytes and compare to the last 2 bytes
4. **Version:** the version byte must be a recognized version
5. **Entity type:** the type code should be within a known range (unknown types may be accepted with a warning)

### Addressing the "Just Use a UUID" Argument

The most common objection to structured identifiers is that a plain UUID is simpler and sufficient. This argument holds in systems with a single entity type, a single source of truth, and no need for human interaction with identifiers. In a federated book metadata ecosystem, none of these conditions hold:

- Multiple entity types share the same namespace and must be distinguished without a lookup
- Multiple vendors mint identifiers independently and must be traceable to their source
- Identifiers appear in UIs, emails, support tickets, and printed materials where human readability matters
- Environments (prod/staging/dev) share infrastructure and must be kept strictly separate
- Data integrity across system boundaries demands built-in error detection

A BMID is not a replacement for UUID — it *contains* a UUID. It is a thin, fixed-cost envelope around a UUID that adds routing, provenance, safety, and integrity. The cost is 9 additional bytes. The value is the elimination of multiple auxiliary systems, lookup tables, and failure modes.

---

## Security Considerations

The CRC-16 check bytes are designed for error detection, not security. They do not protect against deliberate tampering. If cryptographic integrity is required, implementations should layer HMAC or digital signatures over the BMID rather than extending the identifier itself.

The vendor ID and environment fields are informational and should not be treated as access control mechanisms. Authorization decisions must be made by the receiving system based on its own trust model.

---

## Glossary

| Term | Definition |
|------|------------|
| **BMID** | Book Metadata Identifier. The structured identifier format specified in this document. |
| **LRM** | IFLA Library Reference Model. The conceptual model for bibliographic entities (Work, Expression, Manifestation, Item, Agent). |
| **Crockford Base32** | A base-32 encoding alphabet designed by Douglas Crockford. Uses 0–9 and A–Z excluding I, L, O, U. |
| **CRC-16/CCITT** | A 16-bit cyclic redundancy check commonly used for error detection in data transmission. |
| **UUIDv7** | A UUID version defined in RFC 9562 that embeds a Unix timestamp for temporal ordering. |
| **Vendor ID** | A 3-byte identifier assigned to an organization or system that mints BMIDs. |

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository status

Specification-only. The only substantive file is `bmid-spec.md` (the BMID v1.0 draft). There is no implementation, no build system, and no tests yet. Do not invent commands or pretend scaffolding exists.

## What BMID is

A **Book Metadata Identifier**: a 25-byte (200-bit) structured identifier that wraps a UUIDv7 with routing and integrity metadata, encoded as 40 Crockford Base32 characters. Designed for IFLA LRM entities (Work, Expression, Manifestation, Item, Agent, Series, Subject) in a coordinated consortium of partner organizations.

The key framing to preserve in any writing or code: **a BMID is not a replacement for UUID — it contains a UUID.** It is a thin, fixed-cost envelope that adds routing, provenance, environment awareness, and integrity.

BMID is for entities that lack standard identifiers (Work, Manifestation, Contributor, Series, etc.). It is **not** a replacement for ISBN, ISTC, ISNI, or DOI — those continue to be carried as parallel metadata.

## Load-bearing invariants

Any implementation or spec edit must respect these — they are the points where ambiguity would break interop:

- **Byte layout is fixed and ordered**:

  | Offset | Field | Size |
  |--------|-------|------|
  | 0 | Version | 1 byte |
  | 1 | Entity Type | 1 byte |
  | 2 | Vendor ID | 1 byte |
  | 3 | Environment | 1 byte |
  | 4–6 | Reserved (must be zero in v1) | 3 bytes |
  | 7–22 | UUIDv7 payload | 16 bytes |
  | 23–24 | CRC-16/XMODEM, big-endian | 2 bytes |

  Total: 25 bytes / 40 Crockford Base32 characters.

- **CRC-16/XMODEM** over the first 23 bytes. Parameters: poly `0x1021`, init `0x0000`, refin/refout false, xorout `0x0000`. Test vector: CRC of `"123456789"` is `0x31C3`.
- **The CRC algorithm is invariant across all BMID versions.** Future version bumps may change anything else, but never the CRC. This is what lets validation compute CRC before reading the version byte.
- **Crockford Base32 exclusively** (excludes I, L, O, U). Case-insensitive on input. Canonical form is uppercase. 5 bits/char × 40 chars = 200 bits with no padding.
- **No hyphens, no separators, no segmented display form.** Canonical = 40 unbroken uppercase chars. URN form `urn:bmid:<40-char-id>` is optional but supported.
- **UUIDv7 is mandatory.** Not v4. Mandating v7 is what makes temporal ordering and index locality real BMID guarantees rather than incidental.
- **Parse fields from the decoded byte array, not from character offsets in the string.** Field boundaries do not align with Base32 character boundaries.
- **Validation order**: length → charset → CRC → version → field codes → reserved-bytes-zero check.

## Registries

All allocations are made by submitting a PR against `bmid-spec.md`. There is no runtime registry service.

- **Entity Type** (1 byte): `0x01–0x07` core LRM (Work, Expression, Manifestation, Item, Agent, Series, Subject); `0x08–0x7F` reserved for future LRM/IFLA-aligned types; `0x80–0xFE` available for registry-assigned extensions; `0x00` and `0xFF` reserved.
- **Vendor ID** (1 byte): `0x01–0xFE` available for assignment to partner organizations; `0x00` invalid; `0xFF` reserved for testing/examples. No allocations in the current draft.
- **Environment** (1 byte): `0x01` Production, `0x02` Staging, `0x03` Development, `0x04` Test.

## Editing the spec

- The spec is a **DRAFT v1.0**. Changes to byte layout, field sizes, the CRC algorithm, or the mandatory UUID version are breaking and should prompt a version-byte discussion rather than silent edits.
- The vendor ID identifies the **minter at mint time** — it is provenance, not current custody. Custody belongs in mutable metadata. Don't conflate them in spec edits.
- The environment flag is **diagnostic, not security**. Don't reframe it as a defense or boundary enforcement mechanism. Network/credential isolation is the real boundary.
- Keep the two storage forms distinct in any discussion: binary (25 bytes) and canonical Base32 (40 chars). There is no third form.
- BMIDs are **never reused or reassigned.** Deprecation is handled via a separate metadata field pointing to a successor BMID.
- The "Comparison with Plain UUIDs" and "Why a Structured Identifier" sections are the rhetorical core — edits there change the spec's pitch, not just its prose.

## Worked example (for sanity-checking implementations)

Canonical: `041ZY080000033SD9A600W93K8NKRKAYDXX8PHR4`
Decoded (hex): `01 03 FF 01 00 00 00 01 8F 2D 4A 8C 00 71 23 9A 2B 3C 4D 5E 6F 7A 8B 47 04`
CRC: `0x4704`
Decoded fields: version=1, entity_type=Manifestation (0x03), vendor=testing (0xFF), environment=Production (0x01).

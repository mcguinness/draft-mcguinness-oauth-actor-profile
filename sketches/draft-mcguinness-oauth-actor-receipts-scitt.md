# OAuth Actor Receipts SCITT Profile (SCITT-aligned variant)

> **Status:** Early-stage architectural sketch.  Not a published I-D.
> Companion to [draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md).
> See [README](./README.md) for context and the [JWT-native variant](./draft-mcguinness-oauth-actor-receipts-transparency.md).
> Last updated: 2026-05-10

## Abstract

This profile augments [draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md)
by submitting each actor receipt to a SCITT Transparency Service (TS)
and including the resulting SCITT Receipt alongside the receipt in the
token.  Recipients verify SCITT Receipts against trusted Transparency
Services, shifting trust from per-receipt-issuer enumeration to a small
set of trusted TS operators.  This document defines the SCITT Signed
Statement format for actor receipts, the `actor_receipts_scitt` token
claim, and the issuer/consumer processing rules required to integrate
JWS-based actor receipts with COSE-based SCITT primitives.

## 1. Introduction

The [SCITT working group](https://datatracker.ietf.org/wg/scitt/about/)
is developing the IETF's general-purpose architecture for transparency
over signed statements.  Its model (Issuer signs Statement, submits
to Transparency Service, receives a Receipt that vouches for inclusion
in an append-only ledger) is structurally identical to what the
receipts spec would need to escape per-issuer-trust enumeration.

Aligning with SCITT, rather than defining a parallel JWT-based
mechanism, gets:

- Reuse of SCITT's verifiable data structure ([draft-ietf-cose-merkle-tree-proofs](https://datatracker.ietf.org/doc/draft-ietf-cose-merkle-tree-proofs/))
  and Receipt format.
- Reuse of SCITT's witness protocols and multi-TS redundancy.
- Integration with the SCITT registry of Transparency Services as it
  develops.
- Shared tooling and verifier libraries with other SCITT-bearing
  artifact ecosystems (software supply chain, content provenance,
  etc.).

The cost of alignment is the JOSE/COSE format boundary: actor receipts
are JWS, SCITT artifacts are COSE.  This profile addresses that
boundary explicitly: the actor receipt remains a JWS (unchanged by
this profile), and the SCITT Signed Statement is a COSE_Sign1 envelope
whose payload is the JWS's compact serialization.

## 2. Terminology

This document uses SCITT terms as defined in
[draft-ietf-scitt-architecture](https://datatracker.ietf.org/doc/draft-ietf-scitt-architecture/):

- **Signed Statement:** a COSE_Sign1 envelope conveying a claim from
  an Issuer.  In this profile, the Signed Statement's payload is an
  actor receipt's compact-serialized JWS.
- **Transparency Service (TS):** the operator of a verifiable
  append-only data structure that records Signed Statements and issues
  Receipts.
- **SCITT Receipt:** a signed artifact returned by the TS proving
  inclusion of a Signed Statement in the TS's data structure.  In this
  document, "SCITT Receipt" always refers to this construct;
  "actor receipt" refers to the JWS defined by the receipts spec.

## 3. Architecture

```
+-------------+   actor receipt (JWS)   +----------------+
|     AS      |------------------------>|  Receipts spec |
|             |                         |   consumers    |
|             |   wrap as Signed        +----------------+
|             |   Statement (COSE)
|             |   submit
|             v
|             +---->  +--------------------+
|             |       |  SCITT Transparency|
|             |       |     Service        |
|             |       +--------------------+
|             |                |
|             |                | SCITT Receipt
|             |<---------------+
|             |
|             | bundle (actor receipt + SCITT Receipt)
|             v
+-------------+   delegated token (JWT)   +-------------+
                  with actor_receipts AND  | Recipient   |
                  actor_receipts_scitt     | (verifier)  |
                                           +-------------+
```

The AS signs the actor receipt as JWS (per receipts spec), wraps it as
a SCITT Signed Statement, submits to the TS, receives a SCITT Receipt,
and carries both the actor receipt and the SCITT Receipt in the token.
Recipients have three layers of trust available: (a) the AS's JWS
signature on the actor receipt, (b) the AS's COSE signature on the
Signed Statement, (c) the TS's signature on the SCITT Receipt.
Different trust modes use different subsets of these.

## 4. Signed Statement Format

Each actor receipt that is submitted to a SCITT TS is wrapped in a
COSE_Sign1 Signed Statement.  The protected header contains:

- `alg` (label 1): asymmetric digital signature algorithm; MUST NOT be
  a MAC algorithm.
- `kid` (label 4): key identifier, when the AS publishes multiple
  verification keys.
- `cty` (label 3): content type, set to `application/actor-receipt+jwt`
  (the existing receipts-spec media type).
- `iss` (CWT_Claims label 33 → 1, per [RFC 9597](https://www.rfc-editor.org/rfc/rfc9597)):
  the AS issuer identifier.  MUST equal the actor receipt's `iss`
  claim.

The unprotected header is empty.

The payload is the compact-serialized actor receipt
(header.payload.signature), carried as a CBOR byte string.

The signature is over the COSE_Sign1 sig_structure as defined in
[RFC 9052](https://www.rfc-editor.org/rfc/rfc9052), computed by the AS
using the algorithm named in `alg`.  In typical deployments the AS
uses the same key for both the JWS receipt and the COSE Signed
Statement; nothing in this profile prevents using distinct keys.

## 5. The `actor_receipts_scitt` Claim

### `actor_receipts_scitt`

OPTIONAL.  An array parallel to `actor_receipts`.
`actor_receipts_scitt[i]` corresponds to `actor_receipts[i]`.  Each
non-null entry is a JSON object with two members:

- `statement`: REQUIRED.  Base64url encoding of the COSE_Sign1 Signed
  Statement defined in Section 4.
- `receipt`: REQUIRED.  Base64url encoding of one or more SCITT
  Receipts ([draft-ietf-cose-merkle-tree-proofs](https://datatracker.ietf.org/doc/draft-ietf-cose-merkle-tree-proofs/))
  covering the Signed Statement.  When multiple Receipts are present
  (e.g., from different TSes), they are encoded as a CBOR array.

Entries MAY be `null` for receipts the issuer did not submit to a TS.
The array MUST contain at least one non-null entry; otherwise the
claim MUST be omitted.  The array length MUST equal the length of
`actor_receipts`.

### `actor_receipts_scitt_complete`

OPTIONAL.  A boolean.  When `true`, every receipt in `actor_receipts`
has a corresponding non-null SCITT entry.  Recipients MUST verify
this when set to `true`.

## 6. Issuer Processing

When an issuer creates a new actor receipt under the receipts spec
and chooses SCITT-based attestation:

1. Construct and sign the actor receipt JWS per the receipts spec.
2. Construct a COSE_Sign1 Signed Statement per Section 4 of this
   profile, with the actor receipt JWS as payload.
3. Submit the Signed Statement to one or more configured Transparency
   Services using [SCRAPI](https://datatracker.ietf.org/doc/draft-ietf-scitt-scrapi/)
   or an equivalent submission interface.
4. Receive a SCITT Receipt from each TS.
5. Verify each SCITT Receipt locally before relying on it: parse the
   Receipt, validate the TS signature against the TS's published
   verification key, verify the Merkle audit path against the TS's
   signed log root.
6. Construct the `actor_receipts_scitt[i]` entry containing the Signed
   Statement and the verified SCITT Receipt(s).

Issuers MAY submit a Signed Statement to multiple TSes for redundancy.
Submission to a TS MAY be performed asynchronously after token
issuance; in that case the issuer holds the token until SCITT Receipts
are returned, then issues with `actor_receipts_scitt` populated.
Synchronous-submission deployments add latency; asynchronous
deployments require operator tooling for retries and durable
submission queues.

Issuers MUST NOT include an entry in `actor_receipts_scitt` whose
Signed Statement payload differs from the corresponding
`actor_receipts[i]` JWS bytes.

## 7. Consumer Processing

Consumer processing extends the receipts spec's Consumer Processing.
In addition to the existing rules:

1. If the recipient operates in SCITT-required mode, verify that
   `actor_receipts_scitt` is present.  If absent, reject under local
   policy.
2. For each receipt at index `i` requiring SCITT verification:
   1. Verify that `actor_receipts_scitt[i]` is non-null.
   2. Decode the Signed Statement.  Verify that its `iss`
      (CWT_Claims) matches the actor receipt's `iss`.  Verify that its
      payload bytes equal the compact-serialized JWS from
      `actor_receipts[i]`.
   3. Verify the COSE_Sign1 signature on the Signed Statement.  This
      is OPTIONAL but RECOMMENDED in belt-and-suspenders mode; the JWS
      signature on the inner actor receipt provides equivalent
      integrity if the two are signed by the same key.
   4. For each SCITT Receipt present:
      - parse the Receipt;
      - verify the TS signature against a trusted TS operator's
        published verification key;
      - verify the Merkle audit path against the Receipt's log root;
      - (optional) verify the log root against witness signatures.
   5. If at least one SCITT Receipt from a trusted TS validates, the
      actor receipt is SCITT-attested.
3. If SCITT verification is required and no Receipt validates, reject
   the receipt chain under local policy.

## 8. Trust Modes

Three modes, parallel to the JWT-native variant:

**Per-issuer mode.**  SCITT artifacts ignored.  Per-issuer trust per
the receipts spec.

**SCITT-only mode.**  Recipient trusts a configured set of TS
operators.  An actor receipt is accepted if a SCITT Receipt from a
trusted TS validates; the receipt's direct issuer signature is
verified for integrity but the issuer need not be in any per-issuer
trust set.  This is the federation-friendly mode.

**Belt-and-suspenders mode.**  Recipient requires both: receipt issuer
in per-issuer trust set, AND a valid SCITT Receipt from a trusted TS.

## 9. Discovery

### 9.1 Authorization Server Metadata

- `actor_receipts_scitt_supported`: OPTIONAL.  Boolean indicating that
  the AS submits actor receipts to one or more SCITT Transparency
  Services.

- `actor_receipts_scitt_services_supported`: OPTIONAL.  Array of TS
  identifiers (URIs) the AS uses.

### 9.2 Protected Resource Metadata

- `actor_receipts_scitt_required`: OPTIONAL.  Boolean indicating the
  resource server requires SCITT attestation for actor receipts.

- `actor_receipts_scitt_services_accepted`: OPTIONAL.  Array of
  trusted TS identifiers.  Issuers SHOULD include SCITT Receipts from
  at least one listed TS.

### 9.3 SCITT Service Discovery

This profile does not standardize Transparency Service discovery;
deployments use SCITT-WG-defined mechanisms (SCRAPI metadata,
configured TS endpoints, or equivalent).

## 10. Privacy Considerations

The privacy considerations of the receipts spec apply.  In addition:

**SCITT logs are public to log consumers by design.**  Submitting
actor receipts to a public SCITT log makes the receipt content
(issuers, actors, and `cnf` if present) visible to anyone with log
access.  Issuers SHOULD evaluate whether the chosen TS's access model
is compatible with the deployment's privacy posture.  Federation-
internal or sectoral TSes are typical mitigations; bilateral TSes are
also valid but lose the multi-recipient benefit.

**Right to deletion.**  Append-only logs do not honor deletion
requests.  Deployments under data-protection regimes that require
deletability MUST use TSes whose retention model is compatible
(consortium TSes with member-controlled retention, deployment-private
TSes, etc.) or restrict logged content to data that is not subject to
deletion.

**Submission identity.**  The Signed Statement's `iss` claim names
the AS.  TSes typically log the submitter's identity; deployments
should treat TS access logs as part of the privacy surface.

## 11. Format Coexistence

This profile sits at the JOSE/COSE boundary intentionally:

- **Inner artifact:** actor receipt JWS, defined and unchanged by this
  profile.  Receipts-spec consumers process it without any awareness
  of SCITT.
- **Outer artifact:** SCITT Signed Statement (COSE_Sign1) and SCITT
  Receipt.  COSE-aware consumers process them.
- **Carrier:** JWT outer token with `actor_receipts` (JWS strings) and
  `actor_receipts_scitt` (base64-encoded COSE bytes).

A recipient that implements only the receipts spec validates
`actor_receipts` and ignores `actor_receipts_scitt`.  A recipient that
implements this profile additionally validates the SCITT artifacts.
Implementations need both JOSE and COSE libraries; this is the
principal cost of alignment with SCITT.

This profile does not require COSE-encoding of the actor receipt
content itself.  An issuer using this profile signs their receipts as
JWS once (per receipts spec) and as COSE_Sign1 once (per Section 4,
with the JWS as payload).  Both signatures are typically by the same
key; deployments needing key separation MAY use distinct keys.

## 12. Open Design Questions

**Q1: Single-signature or double-signature.**  The drafted approach
has the AS sign twice (JWS + COSE wrapper), with the COSE signature
over the JWS bytes.  Alternative: the COSE Signed Statement is a
"delivery envelope" that doesn't need its own signature, since the
inner JWS is already AS-signed and the SCITT Receipt vouches for the
bytes.  The `iss` claim in the COSE protected header would still
identify the AS.  This would require SCITT to accept Signed Statements
with an empty signature field, which the SCITT architecture currently
doesn't (the signature is required).  Lobbying for SCITT to accept
this would be a separate WG conversation.

**Q2: Signed Statement payload format.**  Drafted as the JWS compact
serialization (a CBOR byte string).  Alternative: a COSE re-encoding
of the receipt's JSON payload, with the inner JWS signature dropped.
More uniform on the SCITT side but breaks compatibility with non-SCITT
receipts consumers.  Drafted approach is correct but the question may
recur in WG.

**Q3: SCITT registration of `actor-receipt+jwt` content type.**  The
Signed Statement's `cty` is set to the actor-receipt media type.
This requires the SCITT ecosystem to recognize this content type; in
practice this is just a media-type registration (already done by the
receipts spec) and a SCITT WG note that COSE-wrapped content with
this type is the actor-receipts profile.

**Q4: Carriage of multiple SCITT Receipts.**  Drafted as a CBOR array
within the `receipt` field.  Alternative: separate array entries for
each TS, with explicit TS identifiers.  More verbose but easier for
consumers to filter to trusted TSes without parsing each Receipt.

**Q5: SCRAPI dependency vs. private submission interfaces.**  SCRAPI
is the SCITT submission API standard but is not yet finalized.
Deployments may use private submission interfaces in the interim.
This profile is silent on the submission protocol; the only
interoperability constraint is the SCITT Receipt format.

## 13. Comparison to JWT-Native Variant

| Concern | [JWT-native variant](./draft-mcguinness-oauth-actor-receipts-transparency.md) | SCITT-aligned variant |
|---|---|---|
| Inclusion proof format | JWS-based, defined in this profile family | COSE-based, defined in [draft-ietf-cose-merkle-tree-proofs](https://datatracker.ietf.org/doc/draft-ietf-cose-merkle-tree-proofs/) (reused) |
| Witness/gossip protocol | Not defined; deployment-specific | Reuses SCITT's witness model (when ratified) |
| Library dependency | JOSE only | JOSE + COSE |
| Ecosystem reuse | OAuth-only | Cross-ecosystem (software supply chain, content provenance, etc.) |
| Maturity dependency | Self-contained | Depends on SCITT WG progress |
| Token size | Smaller (JWS proofs) | Larger (COSE Signed Statement + COSE Receipt encoded as base64) |
| Format friction | None | JOSE/COSE boundary at issuer and consumer |

## 14. Recommendation

For an actual companion-profile draft, the SCITT-aligned variant is
the right architectural target because:

- The transparency / verifiable data structure / witness gossip work
  is genuinely shared infrastructure.  Building a JWS-parallel doubles
  the implementation surface and divides community effort.
- SCITT's governance model and Transparency Service registry are the
  kind of cross-ecosystem coordination OAuth alone won't provide.
- The format friction (JOSE vs. COSE) is real but bounded:
  implementations need both library families anyway in modern
  deployments (COSE shows up in payment, supply chain, IoT, web
  authentication, and increasingly in the OAuth/SCITT/SD-JWT triad).

The JWT-native variant is the right *fallback* if SCITT progresses
too slowly or if the OAuth WG decides the COSE dependency is
unacceptable for OAuth-stack deployments.  Even then, structuring the
JWT-native variant as "SCITT-shaped, JWS-encoded" (semantic alignment
without format alignment) preserves the option to converge later.

## 15. Net

If this companion is to ship, open a coordination thread between the
OAuth WG and the SCITT WG before drafting in earnest.  The
architectural conversation about whether OAuth Actor Receipts are an
in-scope SCITT use case is the highest-leverage decision.  If yes,
the draft is mostly profile-of-SCITT bookkeeping (defining the Signed
Statement payload and JWT carriage); if no, the JWT-native variant is
the path.

## See Also

- [draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md)
 : companion this is built on.
- [JWT-native variant](./draft-mcguinness-oauth-actor-receipts-transparency.md)
 : alternative architectural approach.
- [draft-ietf-scitt-architecture](https://datatracker.ietf.org/doc/draft-ietf-scitt-architecture/)
 : the IETF transparency framework this aligns with.
- [draft-ietf-scitt-scrapi](https://datatracker.ietf.org/doc/draft-ietf-scitt-scrapi/)
 : SCITT submission API.
- [draft-ietf-cose-merkle-tree-proofs](https://datatracker.ietf.org/doc/draft-ietf-cose-merkle-tree-proofs/)
 : Merkle proof format used by SCITT Receipts.
- [RFC 9052 (COSE)](https://www.rfc-editor.org/rfc/rfc9052)
 : COSE_Sign1 format used by Signed Statements.
- [RFC 9597 (CWT Claims in COSE Headers)](https://www.rfc-editor.org/rfc/rfc9597)
 : for the `iss` claim in protected headers.

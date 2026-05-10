# OAuth Actor Receipt Transparency (JWT-native variant)

> **Status:** Early-stage architectural sketch.  Not a published I-D.
> Companion to [draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md).
> See [README](./README.md) for context and the [SCITT-aligned variant](./draft-mcguinness-oauth-actor-receipts-scitt.md).
> Last updated: 2026-05-10

## Abstract

This document defines OAuth Actor Receipt Transparency, an optional
companion profile that adds transparency-log-based verification to OAuth
Actor Receipts.  Each actor receipt is logged in an append-only
transparency log at issuance time; the log returns a signed inclusion
proof that the issuer carries inline with the receipt.  Recipients
verify receipts via log-inclusion proofs against trusted log operators,
in addition to or instead of validating each receipt's issuer signature
directly.  This shifts the trust frontier from per-issuer enumeration
to a small set of trusted log operators, supporting federation and
open-world delegation.

## 1. Introduction

The actor-receipts profile establishes per-issuer non-transitive trust:
each receipt is validated against the recipient's pre-configured
trusted-issuer set.  This is operationally tractable in small bilateral
federations (a recipient configures trust for a known finite set of
issuers) but breaks in large or dynamic ones.  Cross-organizational AI
agent networks, open identity federations, and unattended workload-to-
workload delegation all push toward a model where the recipient cannot
(and should not have to) enumerate every issuer that may appear in an
inbound chain.

Transparency-log-based mechanisms ([Certificate Transparency](https://www.rfc-editor.org/rfc/rfc9162),
Sigstore Rekor, the [SCITT architecture](https://datatracker.ietf.org/doc/draft-ietf-scitt-architecture/))
address this by introducing a small number of trusted log operators
that vouch for issuer behavior over time.  An issuer that signs different
statements for different parties (a "split-view" attack) is detectable
through log monitoring; revoked or compromised issuers are visible to
all log consumers; and recipients can shift trust from "do I trust this
issuer?" to "do I trust that this log faithfully records issuer
activity?"

This document specifies how OAuth Actor Receipts are submitted to a
transparency log, how inclusion proofs are carried alongside receipts
in tokens, and how recipients verify them.  The mechanism is additive:
a token MAY carry receipts only, inclusion proofs only, or both.  The
receipts spec is unchanged; this profile defines new claims and
processing rules.

## 2. Terminology

- **Log** or **Transparency Log:** an append-only, publicly-auditable
  data structure (typically a Merkle tree) operated by a Log Operator,
  into which signed statements are recorded and inclusion proofs are
  returned.  This profile does not standardize the log construction;
  it requires only that the log support inclusion proofs verifiable
  against a signed log root.
- **Inclusion Proof:** a cryptographic proof that a specific statement
  (here, an actor receipt JWT string) is included in the log at a
  specific position, signed by the log operator.  Equivalent to a
  SCITT Receipt or a Certificate Transparency Signed Certificate
  Timestamp + Merkle audit path.
- **Log Root:** the signed Merkle tree head that the log operator
  publishes; a specific root identifies the log state at a point in
  time.
- **Witness:** an external party that signs the log root to attest the
  log operator is presenting a consistent view to all observers;
  multi-witness models defend against split-view attacks.

The "receipt" terminology of the receipts profile refers to the
AS-signed per-hop JWT.  This profile uses "inclusion proof" for the
log-side artifact to avoid collision with SCITT's use of "Receipt" for
the same construct.

## 3. Architecture Overview

### 3.1 Issuance flow

When an issuer creates an actor receipt under the receipts profile:

1. The issuer constructs and signs the receipt JWT as defined in the
   receipts spec's Receipt Claims section.
2. The issuer submits the compact-serialized receipt to a configured
   transparency log.
3. The log operator records the receipt as a leaf in its Merkle tree,
   advances the tree, and signs the new root.
4. The log operator returns an inclusion proof that vouches for the
   receipt's presence at a specific tree position under a specific
   signed root.
5. The issuer prepends the inclusion proof to the corresponding entry
   of a new top-level token claim, `actor_receipts_log` (parallel to
   `actor_receipts`).

The issuer MAY submit a receipt to multiple logs and include multiple
inclusion proofs per receipt (one per log).

### 3.2 Recipient flow

When a recipient validates a receipt-bearing token under the receipts
profile and inclusion proofs are present:

1. Validate the receipt itself per the receipts spec's Consumer
   Processing rules.
2. For each receipt, validate the corresponding inclusion proof:
   - parse the inclusion proof per Section 5;
   - verify the log root signature against a trusted log operator's
     published verification key;
   - verify the Merkle path from the receipt's leaf hash to the log
     root;
   - optionally verify the log root against witness signatures (where
     witness gossip is in use).
3. Apply the trust-model rules of Section 6 to determine whether the
   receipt is acceptable.

Recipients MAY accept tokens with valid receipt signatures but no
inclusion proofs (falling back to the receipts spec's per-issuer trust).
Recipients MAY accept tokens with valid inclusion proofs but receipts
whose direct signers are not in the per-issuer trust set (relying on
the log operator).  Recipients MAY require both (defense in depth).

### 3.3 Log operator role

A log operator MUST:

- accept receipt submissions from authorized issuers;
- record each accepted receipt as a leaf in an append-only Merkle tree;
- publish signed log roots at regular intervals;
- produce inclusion proofs on demand for any leaf;
- (recommended) participate in a witness/gossip protocol so that
  split-view attacks are detectable.

The log operator does NOT validate receipt content semantics (whether
the actor is correct, whether the chain is valid, etc.); it only
attests inclusion in the log.  Semantic validation remains the
recipient's responsibility.

This profile does not standardize log governance, log-operator
selection, or witness protocols; those are deployment-specific or
governed by frameworks such as SCITT.

## 4. The `actor_receipts_log` Claim

### `actor_receipts_log`

OPTIONAL.  An array of arrays of inclusion proofs.  The outer array is
parallel to `actor_receipts`: `actor_receipts_log[i]` corresponds to
`actor_receipts[i]`.  The inner array contains zero or more inclusion
proofs for that receipt, each from a different log operator.

Each inclusion proof is a compact-serialized JWT
(`typ: actor-receipt-inclusion+jwt`) signed by the log operator, with
payload as defined in Section 5.  When present, the array:

- MUST have the same outer length as `actor_receipts`;
- MAY contain empty inner arrays for receipts the issuer did not log;
- MUST contain at least one non-empty inner array, otherwise the claim
  MUST be omitted.

### `actor_receipts_log_complete`

OPTIONAL.  A boolean.  When `true`, the issuer attests that every
receipt in `actor_receipts` has at least one inclusion proof in
`actor_receipts_log`.  Recipients MUST verify this when set to `true`.
When `false` or absent, partial coverage is permitted.

## 5. Inclusion Proof Format

Each inclusion proof is a JWT signed by the log operator, carrying the
following claims:

- `iss`: REQUIRED.  Identifier of the log operator (URL or other
  deployment-specified identifier).

- `sub`: REQUIRED.  The base64url-encoded SHA-256 hash of the
  compact-serialized actor receipt that this proof attests.  Recipients
  MUST compute this hash from the receipt as carried in
  `actor_receipts[i]` and reject any inclusion proof whose `sub` does
  not match.

- `log_id`: REQUIRED.  Stable identifier for the specific log instance
  (since one operator may run multiple logs).

- `tree_size`: REQUIRED.  The size of the Merkle tree at the log root
  under which this proof is valid.

- `leaf_index`: REQUIRED.  Position of the receipt's leaf in the tree.

- `audit_path`: REQUIRED.  Array of base64url-encoded sibling hashes
  forming the Merkle audit path from the leaf to the log root.

- `log_root`: REQUIRED.  The signed log root hash under which the proof
  is valid.  This value MUST be the root of the tree at `tree_size`.

- `iat`: REQUIRED.  Time the inclusion proof was issued.

- `exp`: OPTIONAL.  When present, the proof is structurally valid only
  until this time.  Inclusion proofs MAY be long-lived, since the log
  root is a stable witness; deployments SHOULD set `exp` only when they
  intend to require fresh proofs.

This format is loosely modeled on Certificate Transparency SCT
semantics adapted to JWT.  Alignment with [SCITT Receipts](https://datatracker.ietf.org/doc/draft-ietf-scitt-architecture/)
is an open design question (see Section 11); the COSE-based SCITT
format may be preferable for deployments already using SCITT
infrastructure.

## 6. Trust Model

This profile introduces a second trust layer alongside per-issuer trust.

### 6.1 Three trust modes

Recipients MAY operate in one of three modes:

**Per-issuer mode (no transparency).**  Recipients trust each receipt
issuer individually as in the receipts spec.  Inclusion proofs, if
present, are ignored.  Equivalent to receipts without this profile.

**Log-only mode.**  Recipients trust a configured set of log operators.
A receipt is accepted if a valid inclusion proof from a trusted log is
present; the receipt's direct issuer signature is verified for
integrity, but the issuer need not be in the recipient's per-issuer
trust set.  This is the federation-friendly mode.

**Belt-and-suspenders mode.**  Recipients require both: the receipt
issuer is in the per-issuer trust set AND a valid inclusion proof from
a trusted log is present.  Strongest mode; defends against compromise
of either trust layer alone.

The mode is per-recipient deployment policy; this profile does not
standardize the choice.

### 6.2 What log-based trust does and does not provide

Log-based trust provides:

- **Federation scalability.**  A recipient configures trust for a small
  number of log operators rather than enumerating every issuer.
- **Misbehavior detection.**  An issuer that signs different receipts
  for different recipients (split-view) is detectable through log
  monitoring, since both receipts must be logged or one must be
  unlogged.
- **Auditability.**  Anyone with log access can audit issuer behavior
  over time.
- **Detached verification.**  Recipients can verify against the log
  even when the issuer is offline.

Log-based trust does NOT provide:

- **Validity attestation.**  Logs attest inclusion, not semantic
  correctness.  A logged receipt for a fabricated actor is still
  logged (and detectable, but only by parties auditing the log).
- **Revocation.**  Inclusion is permanent; subsequent revocation must
  be expressed as a separate log entry that consumers monitor.
- **Online freshness.**  Inclusion proofs are point-in-time; they do
  not assert that the receipt has not been retroactively repudiated.
- **Defense against compromised log operator.**  A compromised log
  operator can issue inclusion proofs for arbitrary content.
  Multi-log redundancy and witness protocols mitigate this.

### 6.3 Witness gossip

For high-assurance deployments, recipients SHOULD verify log roots
against witness signatures, where multiple independent witnesses sign
each log root.  A log operator that presents different roots to
different parties (split-view) is detectable when witnesses gossip
about the roots they have countersigned.

This profile does not define a witness protocol; deployments adopt
SCITT's witness model or an equivalent.

## 7. Issuer Processing

When an issuer creates a new receipt under the receipts spec's
"Creating the First Receipt" or "Extending an Existing Receipt Chain"
sections, and chooses to support log-based verification:

1. Construct the receipt JWT per the receipts spec.
2. For each transparency log the issuer is configured to use:
   1. Submit the compact-serialized receipt JWT to the log.
   2. Receive an inclusion proof.
   3. Verify the inclusion proof against the log's published
      verification key (the issuer should not trust the proof received
      in step 2 without verification).
3. Append the verified inclusion proofs to the corresponding entry of
   `actor_receipts_log[i]`.
4. Set `actor_receipts_log_complete` according to whether every
   receipt has at least one proof.

Issuers SHOULD log every receipt they issue, even if recipients are
not yet requesting log-based verification.  This builds the audit
trail before it is needed.

## 8. Consumer Processing

Consumer processing extends the receipts spec's Consumer Processing.
In addition to the existing rules:

1. If the recipient operates in log-only or belt-and-suspenders mode,
   verify that `actor_receipts_log` is present.  If absent, reject
   under local policy.
2. For each receipt at index `i` for which the recipient requires log
   verification:
   1. Verify that `actor_receipts_log[i]` contains at least one
      inclusion proof.
   2. For each candidate inclusion proof:
      - parse the proof JWT;
      - verify the log operator's signature against a trusted log
        operator's published key;
      - verify that the proof's `sub` equals the SHA-256 hash of
        `actor_receipts[i]`;
      - verify the Merkle audit path from the leaf to the log root;
      - (optional) verify the log root against witness signatures.
   3. If at least one proof from a trusted log validates, the receipt
      is log-attested.
3. If log verification is required and no proof validates, reject the
   receipt chain under local policy.

This profile adds rejection conditions; per the receipts spec's
Extensibility section, it does not relax any of the receipts spec's
rejection conditions.

## 9. Discovery and Capability Signaling

### 9.1 Authorization Server Metadata

- `actor_receipts_log_supported`: OPTIONAL.  A boolean.  When `true`,
  the AS advertises that it logs receipts to one or more transparency
  logs and can include inclusion proofs in `actor_receipts_log`.

- `actor_receipts_log_operators_supported`: OPTIONAL.  An array of
  identifiers naming the log operators the AS uses.  Recipients use
  this to anticipate which proofs they will need to validate.

### 9.2 Protected Resource Metadata

- `actor_receipts_log_required`: OPTIONAL.  A boolean.  When `true`,
  the resource server requires inclusion proofs for receipts.

- `actor_receipts_log_operators_accepted`: OPTIONAL.  An array of
  trusted log operator identifiers.  Issuers SHOULD include proofs
  from at least one of the listed operators.

### 9.3 Introspection

When introspection is used (per the receipts spec's Introspection
section), the introspection response MAY include `actor_receipts_log`
alongside `actor_receipts`.

## 10. Privacy Considerations

Transparency logs trade privacy for auditability.

**What logs disclose publicly.**  Receipts in a transparency log are
visible to anyone with log access.  This means delegation chains,
issuer identities, actor identities, and (where `cnf` is included)
presenter-key bindings are visible to log auditors, log consumers, and
the public, depending on log access policy.  Logs operated as fully
public infrastructure (CT-style) make this disclosure essentially
permanent and global.

**What logs are appropriate for actor receipts.**  Public CT-style logs
are likely too disclosure-heavy for general OAuth deployments.  More
appropriate models:

- **Federation-internal logs:** logs operated by a federation,
  accessible only to federation members.  Restricts disclosure while
  providing federation-scale trust.
- **Bilateral logs:** logs co-operated by issuer and recipient,
  accessible to both.  Smallest scope but loses the multi-recipient
  benefit.
- **Sectoral logs:** logs operated for a specific industry or use case
  (financial sector, healthcare delegation, etc.).

This profile does not standardize the log access model; deployments
choose based on their privacy and compliance requirements.

**`cnf` in logged receipts.**  Receipt `cnf` values are particularly
sensitive in logs.  Issuers SHOULD apply the `cnf` privacy guidance
from the receipts spec's Historical `cnf` Disclosure section with
extra weight when receipts will be logged: cnf in a logged receipt is
essentially a permanent public record of the historical key binding.

**Subject privacy.**  Receipt `sub` values that contain user
identifiers (rather than service or workload identifiers) become
visible to log auditors.  Deployments handling personally identifiable
subjects SHOULD evaluate whether logging is compatible with their
privacy posture; subject pseudonymization at issuance time is one
mitigation.

**Right to deletion.**  Append-only logs cannot honor deletion
requests.  Deployments operating under data-protection regimes that
require deletability (GDPR right-to-be-forgotten, etc.) MUST evaluate
whether transparency logging is legally compatible with their use
case.  Pseudonymous identifiers in receipts are one mitigation;
ephemeral identifiers that lose meaning over time are another.

## 11. Open Design Questions

This sketch leaves several questions deliberately open for working-
group discussion.

**Q1: Alignment with SCITT.**  The SCITT WG is developing a near-
identical mechanism: signed statements (≈ our actor receipts)
submitted to a transparency service that returns receipts (≈ our
inclusion proofs).  Two paths:

- *Reuse SCITT directly.*  Our `actor_receipts_log` claim becomes
  "carries SCITT Receipts."  Reuses SCITT infrastructure, COSE-based
  formats, and witness protocols.  Costs JWT-native ergonomics;
  recipients must implement COSE.  See the
  [SCITT-aligned variant](./draft-mcguinness-oauth-actor-receipts-scitt.md).
- *Define a JWT-native parallel.*  This sketch.  Our inclusion proof
  is a JWT, reusing JOSE rather than COSE.  Better fit for OAuth/JWT
  ecosystems; misses out on shared SCITT infrastructure.

**Q2: Mandatory vs optional logging.**  Should recipients in log-only
mode reject receipts that are not in the log, or accept them with
reduced assurance?

**Q3: Online verification vs inline carriage.**  Inclusion proofs add
token size.  Alternative: token carries only a reference (log entry
index); recipients fetch proofs from the log on demand.  Smaller
tokens, online dependency.

**Q4: Log operator metadata format.**  Standardize as part of OAuth
AS metadata, or punt to a dedicated transparency-services discovery
mechanism?

**Q5: Witness signatures format.**  In-band (with the inclusion
proof), out-of-band (via separate witness-gossip protocol), or
deployment-specific?

**Q6: Submission authentication.**  How does a log operator
authenticate that a receipt-submission request is from a legitimate
issuer?  Pre-registered issuer keys?  Issuer signature on the
submission?  This is operational but affects what the log meaningfully
attests.

## 12. Relationship to SCITT

The [SCITT architecture](https://datatracker.ietf.org/doc/draft-ietf-scitt-architecture/)
is the IETF's general-purpose transparency framework.  Its Signed
Statement / Transparency Service / Receipt model is structurally
identical to what this profile sketches.

Two architectures suggest themselves for the eventual companion
profile:

- **SCITT-aligned.**  Specifies how actor receipts map to SCITT
  Signed Statements, with SCITT Receipts replacing the inclusion
  proofs sketched in Section 5.  Maximum reuse; OAuth-specific
  surface is small.  See the
  [SCITT-aligned variant](./draft-mcguinness-oauth-actor-receipts-scitt.md).
- **JWT-native parallel.**  This sketch.  Defines a JWT-based
  transparency mechanism in the OAuth ecosystem, parallel to but not
  requiring SCITT.  Smaller dependency footprint; potentially
  divergent from SCITT.

The recommended path for an actual published companion is
SCITT-aligned: the SCITT WG is doing the architectural heavy lifting,
and aligning means OAuth actor receipt transparency benefits from
SCITT's witness protocols, multi-log gossip, and governance work.
The COSE-vs-JWT format friction is real but bounded; both communities
have format-conversion patterns.

## See Also

- [draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md)
  — companion this is built on.
- [SCITT-aligned variant](./draft-mcguinness-oauth-actor-receipts-scitt.md)
  — alternative architectural approach.
- [draft-ietf-scitt-architecture](https://datatracker.ietf.org/doc/draft-ietf-scitt-architecture/)
  — the IETF transparency framework this references.
- [RFC 9162 (Certificate Transparency)](https://www.rfc-editor.org/rfc/rfc9162)
  — model this draws on.

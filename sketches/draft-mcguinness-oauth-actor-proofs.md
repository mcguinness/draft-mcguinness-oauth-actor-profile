# OAuth Actor-Signed Hop Proofs

> **Status:** Early-stage architectural sketch.  Not a published I-D.
> Companion to [draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md)
> and to the core [draft-mcguinness-oauth-actor-profile](../draft-mcguinness-oauth-actor-profile.md).
> See [README](./README.md) for context.
> Last updated: 2026-05-10

## Abstract

This document defines OAuth Actor-Signed Hop Proofs, an optional
companion profile that adds actor-side cryptographic non-repudiation
to delegation chains.  At each hop, the actor being added to the
chain signs a proof attesting their own participation and the target
binding they authorize.  Proofs travel with the token in a top-level
`actor_proofs` claim parallel to `actor_receipts`, are linked into a
hash chain via `prh`, and may cross-reference corresponding receipts
by `jti`.  Where receipts mitigate a compromised *downstream* issuer
fabricating prior-hop provenance, proofs additionally mitigate a
compromised *current* outer-token issuer fabricating actor
participation.

## 1. Introduction

The receipts profile establishes that the AS at each hop attests
which actor was added.  The threat model receipts address is a
compromised downstream issuer that wants to retroactively rewrite
prior-hop provenance: that issuer cannot forge the prior issuers'
receipt signatures.

Receipts do not address a different but adjacent threat: a
compromised AS at the current hop, or a colluding AS, can fabricate
the participation of an actor that did not actually authorize the
delegation.  AS-signed receipts say "AS X attests that actor Y was
the actor at this hop."  Nothing in that statement requires Y's
own participation; only the AS attests.  In federated and especially
agentic deployments, where actors (AI agents, workloads, services)
have their own credentials and act on behalf of subjects, this gap
is operationally significant.

Actor-signed proofs close that gap.  At each hop, the actor being
added to the chain signs a proof JWT attesting "I, this actor,
participated at this hop and authorized delegation to this target."
A compromised AS still cannot forge actor participation without
compromising the actor's signing key.

This profile is independent of receipts: a token MAY carry proofs
only, receipts only, both, or neither.  Belt-and-suspenders deployments
require both.

## 2. Terminology

This document uses terminology from
[draft-mcguinness-oauth-actor-profile](../draft-mcguinness-oauth-actor-profile.md)
and [draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md).

- **Actor Proof:** a signed JWT issued and signed by the actor at a
  delegation hop, attesting the actor's own participation and the
  target binding for that hop.
- **Actor Signing Key:** an asymmetric key controlled by the actor
  and bound to the actor's identity through deployment-specific means
  (registered client key, workload identity certificate, DPoP key
  registration, etc.).  This profile does not standardize how actor
  signing keys are established.
- **Target Binding:** the audience and (optional) resource
  identifier the actor authorizes for the delegation at this hop.
  The proof's target binding constrains what the issued token may
  authorize downstream.

## 3. Architecture

### 3.1 Threat-model coverage

| Adversary | Receipts | Proofs |
|---|---|---|
| Compromised downstream issuer fabricating prior-hop provenance | Mitigated | Mitigated |
| Compromised current outer-token issuer fabricating actor participation | NOT mitigated | Mitigated |
| Compromised current issuer misrepresenting target binding | NOT mitigated | Mitigated (when proof binds target) |
| Compromised actor at a hop | NOT mitigated | NOT mitigated (signing key is the trust anchor) |
| Replay of an entire token plus its receipts/proofs | NOT mitigated | NOT mitigated (outer-token replay defense applies) |

### 3.2 Proof creation flow

When an issuer adds a new outermost actor hop:

1. Actor authenticates to the AS using its own credential
   (mTLS client certificate, DPoP-bound key, JWT client assertion,
   etc.).
2. As part of the token exchange or grant request, the actor signs
   a proof JWT (per Section 5) using its actor signing key.  The
   proof attests:
   - the actor's own identity (matches the new outermost `act.sub`);
   - the subject the actor is acting on behalf of;
   - the target binding (intended audience and resource);
   - chain linkage to any previous proof in the chain.
3. AS validates the proof: signature against the actor's known
   verification key, target binding within authorized scope,
   structural alignment with the new outermost `act` object.
4. AS includes the proof in `actor_proofs` of the issued token.

When an issuer is also creating a receipt under the receipts profile,
the receipt and the proof are siblings: both attest the same hop
from different signers.  The proof MAY include the receipt's `jti`
in `receipt_jti` for explicit cross-reference.

### 3.3 Proof verification flow

Recipients validate the outer token, then for each proof:

1. Verify the proof's `typ` is `actor-proof+jwt`.
2. Verify the proof's signature against the actor's verification key.
   Key resolution is deployment-specific; see Section 9.
3. Verify the proof's target binding against the outer token's `aud`
   and `resource`: the outer token's audience MUST be a subset of
   the proof's authorized target.  An outer token with broader
   audience than the proof authorizes is not valid under this
   profile.
4. Verify the proof's `act` matches the corresponding visible `act`
   chain entry by structure (same as receipts hop alignment).
5. Verify the `prh` chain linkage to the next older proof.
6. When `receipt_jti` is present and receipts are required, verify
   the corresponding receipt is present in `actor_receipts` and has
   the named `jti`.

## 4. The `actor_proofs` Claim

### `actor_proofs`

OPTIONAL.  An array of strings.  Each string is the compact
serialization of a signed JWT proof as defined in Section 5.  The
array:

- MUST NOT be empty when present;
- MUST be ordered from newest covered hop to oldest covered hop
  (parallel to `actor_receipts`);
- MUST NOT contain more entries than the visible actor-chain depth
  of the outer token's `act` claim;
- MUST represent a contiguous outermost prefix of the visible `act`
  chain.

### `actor_proofs_complete`

OPTIONAL.  A boolean.  When `true`, the issuer attests that
`actor_proofs` covers every visible hop in the outer token's `act`
chain.  Recipients MUST verify this when set to `true`.

The claim-pair convention follows the receipts spec's Discovery and
Capability Signaling section.

## 5. Actor Proof JWT Format

### 5.1 JOSE Header

The JOSE header of an actor proof:

- MUST include an asymmetric digital-signature `alg` value;
- MUST NOT use `alg: none` or a MAC-based symmetric algorithm;
- MUST include `typ` with the value `actor-proof+jwt`;
- SHOULD include `kid` when the actor publishes multiple
  verification keys.

### 5.2 Proof Claims

The JWT payload of an actor proof uses the following claims.

#### Identity

- `iss`: REQUIRED.  Identifier of the actor that signed the proof.
  MUST equal the corresponding visible `act.sub` value at the
  represented hop.  In contrast to receipts (where `iss` is the AS),
  in proofs `iss` is the actor itself.

- `sub`: REQUIRED.  The subject identifier on whose behalf the actor
  is delegating.  MUST equal the outer token's top-level `sub` for
  `actor_proofs[0]`.  Older proofs MAY carry differing `sub` values
  per the receipts spec's Subject Re-Expression Across Hops rules.

- `act`: REQUIRED.  Single-hop actor object identifying the actor
  (redundant with `iss` but preserved for hop-alignment with the
  visible `act` chain and for `act.iss` namespace context).
  Same structure as receipt `act` (no nested `act`, no `cnf`).

#### Target binding

- `aud`: REQUIRED.  The audience(s) the actor authorizes for the
  delegation at this hop.  Recipients MUST verify the outer token's
  `aud` is a subset of this value (after the receipts spec's audience
  rules apply).

- `resource`: OPTIONAL.  Resource indicator(s)
  ([RFC 8707](https://www.rfc-editor.org/rfc/rfc8707)) the actor
  authorizes.  When present, narrows the target binding beyond `aud`.

#### Chain linkage

- `prh`: OPTIONAL.  Previous proof hash, computed identically to the
  receipts spec's `prh`: base64url-encoded hash of the next older
  proof's compact serialization.  The oldest proof in the chain MUST
  omit `prh`.

- `prh_alg`: OPTIONAL.  Hash algorithm identifier; same semantics and
  default (`sha-256`) as the receipts spec's `prh_alg`.  All proofs
  in a single chain MUST use the same algorithm.

#### Cross-companion reference

- `receipt_jti`: OPTIONAL.  When the issuer is also producing
  receipts for this hop and the proof and receipt are siblings, this
  claim is the receipt's `jti`.  Lets recipients align proofs with
  receipts at the same hop.  Per the receipts spec's Extensibility
  section, cross-companion alignment is by `jti`.

#### Time and identity

- `iat`: REQUIRED.  Time the proof was signed.

- `exp`: REQUIRED.  Expiration time.  Same sizing rules as receipt
  `exp`: MUST cover the maximum lifetime of any token that may carry
  or inherit this proof.

- `jti`: REQUIRED.  Unique identifier for the proof.

- `origin_jti`: OPTIONAL.  The outer token's `jti` at the time of
  proof creation.  Same semantics as receipt `origin_jti`: independently
  verifiable only for `actor_proofs[0]` in the originating-issuance
  case.

#### Workflow correlation

- The proof MAY carry the delegation correlation claim
  (`acti` or `dlg_id`, pending core-profile resolution; see
  [issue #3](https://github.com/mcguinness/draft-mcguinness-oauth-actor-profile/issues/3))
  if the deployment uses one.  When present, MUST equal the outer
  token's value.  Provides actor-attested workflow membership in
  addition to the AS-attested membership in the outer token.

### 5.3 Extension claims

A proof MAY contain additional claims defined by another
specification or by deployment policy.  Consumers MUST ignore
unrecognized claims unless another specification or local agreement
defines their meaning.

## 6. Issuer Processing

### 6.1 Creating the first proof

When an issuer creates a delegated token with a new outermost actor
hop, no inbound `actor_proofs` are being preserved, and the deployment
uses actor-signed proofs:

1. Authenticate the actor through deployment-specific means.
2. Obtain a proof JWT from the actor.  This typically happens
   in-band with the token exchange or grant request: the actor
   signs and submits the proof as part of authenticating itself.
3. Validate the proof:
   - signature against the actor's verification key;
   - `iss` matches the new outermost `act.sub`;
   - `act` matches the new outermost `act` object structurally;
   - target binding (`aud`, optional `resource`) is consistent with
     the requested token's authorized scope;
   - `iat` and `exp` are sane and `exp` covers the issued token's
     lifetime;
   - `prh` is omitted (first proof in chain).
4. Include the proof JWT as `actor_proofs[0]` of the issued token.
5. Set `actor_proofs_complete: true` if no inner hops are uncovered.

### 6.2 Extending an existing proof chain

When an issuer adds a new outermost actor hop and preserves an
inbound `actor_proofs` array:

1. Validate the inbound proof chain by applying consumer processing
   rules before relying on it or carrying it forward.
2. Verify each inbound proof's `exp` covers the issued outer token's
   `exp`.
3. Preserve each inbound proof byte-for-byte unchanged.
4. Obtain a new proof JWT from the new outermost actor (per 6.1).
5. The new proof MUST set `prh` to the hash of the inbound array's
   first element, computed using the inherited `prh_alg`.
6. Prepend the new proof to the inherited array.

Issuers MUST NOT reserialize, resign, normalize, or alter inbound
proofs.

### 6.3 Reissuance without a new actor hop

Same rules as receipts: an issuer that reissues without adding a new
hop MAY carry `actor_proofs` forward unchanged, MUST NOT create a
new proof, MUST NOT change the outer token's `sub`.

### 6.4 Sibling-receipt creation

When the deployment uses both receipts and proofs:

1. Create the proof and the receipt for the same hop in the same
   issuance operation.
2. Set the proof's `receipt_jti` to the receipt's `jti`.
3. Both go into their respective top-level claims (`actor_proofs`,
   `actor_receipts`).

The receipt and proof attest the same hop from different signers
(AS and actor); both are included for belt-and-suspenders trust.

## 7. Consumer Processing

Consumer processing extends the receipts spec's Consumer Processing.
In addition to (or instead of) receipts validation:

1. Validate the outer token per the core actor profile.
2. If `actor_proofs` is required by local policy or by Protected
   Resource Metadata signals (Section 9), verify it is present.
3. Verify the array is non-empty, ordered newest-first, and does not
   exceed the visible actor-chain depth.
4. For each proof, in order:
   - parse the JWT;
   - verify `typ` equals `actor-proof+jwt`;
   - resolve the actor's verification key per Section 8;
   - validate the JWT signature;
   - enforce `exp`, `iat`, and standard JWT validity rules;
   - verify `iss` matches the corresponding visible `act.sub` and
     the proof's `act` matches the visible hop structurally;
   - verify the target binding (`aud`, `resource`) covers the outer
     token's `aud` and `resource`;
   - verify the `prh` chain linkage and `prh_alg` consistency.
5. Verify that `actor_proofs[0].sub` equals the outer token's
   top-level `sub`.  Older proofs MAY carry differing `sub` values
   under the receipts spec's Subject Re-Expression rules.
6. When proofs and receipts are both present, verify cross-companion
   alignment for each pair: `actor_proofs[i].receipt_jti` (if
   present) MUST equal `actor_receipts[i].jti`.
7. If `actor_proofs_complete: true` is set, verify the proof count
   equals the visible actor-chain depth.

## 8. Trust Model

### 8.1 Actor key resolution

A recipient validates a proof by verifying its signature against
the actor's verification key.  Key resolution is deployment-specific
and outside this profile's scope.  Common patterns:

- Pre-registered actor keys (the actor is a registered OAuth client
  with a published JWKS at the AS).
- Workload identity certificates ([draft-ietf-wimse-arch](https://datatracker.ietf.org/doc/draft-ietf-wimse-arch/)
  and related work).
- DPoP-key-bound actors where the proof signing key is the same as
  the DPoP key bound in the outer token's `cnf`.
- Identity-document-based key resolution
  ([draft-ietf-oauth-identity-chaining](https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-chaining/)
  and related work).

For a recipient to validate a proof, the actor's verification key
MUST be discoverable by the recipient through one of these means.
Recipients SHOULD pre-configure the means by which actor keys are
resolved for actors in their trust set.

### 8.2 Trust modes

Three modes:

**Receipts-only mode.**  Proofs ignored.  Trust per the receipts
spec.

**Proofs-only mode.**  Receipts ignored.  Trust rests on actor key
resolution and direct actor signatures.  Suitable for deployments
where receipts are not in use but actor non-repudiation is required.

**Belt-and-suspenders mode.**  Both required.  Strongest mode;
defends against compromise of either AS or actor (but not both
simultaneously).

The mode is per-recipient deployment policy; this profile does not
standardize the choice.

### 8.3 What proofs do and do not provide

Proofs provide:

- **Actor non-repudiation.**  An actor cannot plausibly deny having
  participated in a delegation if their proof is present and
  validates.
- **Target-binding integrity.**  A compromised AS cannot
  retroactively broaden the audience or resource scope of an actor's
  delegation, since the actor signed the target binding.
- **Cross-AS portability.**  An actor's proof is valid against any
  recipient that can resolve the actor's key, independent of which
  AS issued the outer token.

Proofs do NOT provide:

- **Defense against compromised actor key.**  If the actor's signing
  key is compromised, proofs signed with it are forgeable.
  Standard key-rotation and revocation hygiene applies.
- **Online freshness.**  Proofs are point-in-time attestations; they
  do not assert that the represented delegation is still active.
- **Actor non-participation in pre-existing chains.**  An actor that
  has signed proofs in past delegations cannot retroactively
  repudiate them.

## 9. Discovery

### 9.1 Authorization Server Metadata

- `actor_proofs_supported`: OPTIONAL.  Boolean.  When `true`, the AS
  accepts actor-signed proofs in token exchange flows and includes
  them in `actor_proofs` of issued tokens.

- `actor_proofs_complete_supported`: OPTIONAL.  Boolean.  When
  `true`, the AS can produce complete-coverage proof chains.

### 9.2 Protected Resource Metadata

- `actor_proofs_required`: OPTIONAL.  Boolean.  When `true`, the
  resource server requires actor-signed proofs covering at minimum
  the outermost visible actor hop.

- `actor_proofs_complete_required`: OPTIONAL.  Boolean.  When `true`,
  the resource server requires complete-coverage proof chains.

### 9.3 Introspection

When introspection is used, the introspection response MAY include
`actor_proofs` and `actor_proofs_complete` alongside the receipts-spec
introspection members.

## 10. Security Considerations

### 10.1 Actor key compromise

If an actor's signing key is compromised, all proofs signed with
that key become forgeable.  Recipients SHOULD treat tokens carrying
proofs from a compromised actor as lacking trusted proof-based
provenance for those hops.  Standard key-rotation procedures and
revocation mechanisms (e.g., updated JWKS, certificate revocation)
apply.

This profile does not define an in-band revocation mechanism for
proofs.

### 10.2 Replay

Proofs are designed to be carried by tokens that themselves have
replay-detection or sender-constraint properties.  Replay of an
entire token plus its proofs is governed by the outer token's replay
characteristics, not by the proof chain.  Recipients that need
proof-level replay defense SHOULD treat the proof's `jti` as
single-use.

### 10.3 Cross-hop reuse

A proof signed for hop N MUST NOT be reused at hop M without the
actor's re-signing.  The proof's `act` claim and `prh` chain linkage
prevent structural transplantation, but recipients SHOULD verify
that proofs are not reused across delegation flows by tracking
`jti` values against expected hop positions.

### 10.4 Compromised AS at proof creation

A compromised AS that participates in proof creation can:

- Refuse to include valid proofs (denial-of-service).
- Submit incorrect proofs to the recipient (rejected by signature
  validation).
- Forge proofs only if it also has the actor's signing key
  (out of scope; see Section 10.1).

A compromised AS cannot fabricate actor participation absent the
actor's signing key.  This is the principal threat-model improvement
over receipts.

### 10.5 Proof and receipt mismatch

When both proofs and receipts are present and `receipt_jti`
cross-references are in use, mismatch between a proof's `receipt_jti`
and any receipt's `jti` indicates either tampering or a misconfigured
issuer.  Recipients in belt-and-suspenders mode MUST reject such
mismatches.

## 11. Privacy Considerations

The privacy considerations of the receipts spec apply.  In addition:

- Actor proofs reveal actor signing key identifiers (via `kid` or
  via the public key resolved for verification), which are
  potentially long-lived correlation handles.  Issuers SHOULD treat
  proof presence as a deployment decision with similar disclosure
  characteristics to receipt `cnf` values.
- Proof target bindings (`aud`, `resource`) reveal the actor's
  authorized scope.  This may expose deployment-internal resource
  identifiers to recipients that don't need them.  Issuers SHOULD
  evaluate target-binding disclosure when deciding which audiences
  receive proof-bearing tokens.

## 12. Open Design Questions

**Q1: Actor key resolution standardization.**  This profile leaves
key resolution to deployments.  A normative recommendation
(e.g., "actors MUST be registered as OAuth clients with published
JWKS") would simplify interoperability but constrain the deployment
space.

**Q2: Same key as `cnf` or distinct.**  The proof's signing key
COULD be required to match the outer token's `cnf.jkt`, tightly
binding proof to current PoP.  Drafted approach allows distinct
keys for flexibility.  Worth WG discussion.

**Q3: Target binding granularity.**  The proof binds `aud` and
`resource`.  Should it also bind specific scopes or actions?
Finer-grained binding gets closer to action-level attestation, which
is a different scope from this profile.

**Q4: Mandatory `receipt_jti` when receipts are present.**  When a
deployment uses both proofs and receipts, should `receipt_jti` be
REQUIRED?  Drafted as OPTIONAL for deployment flexibility.

**Q5: Pre-issuance proof submission.**  Drafted assumes proofs are
submitted as part of token exchange.  Some deployments may want
pre-issuance proof creation (actor signs a proof template that the
AS later instantiates).  Workflow-engine integration question;
out of scope for v1.

**Q6: Multi-actor proofs.**  In some delegation models, multiple
actors at the same hop may need to co-sign (e.g., multi-party
authorization).  Drafted as single-actor; multi-actor would be a
separate companion or future revision.

## 13. Relationship to Other Companions

### 13.1 Receipts

Proofs and receipts are siblings: AS-signed and actor-signed
attestations of the same hop.  A token MAY carry either, both, or
neither.  When both are present, `receipt_jti` provides cross-
reference per the receipts spec's Extensibility section.

### 13.2 Transparency companions

The transparency companion sketches
([JWT-native](./draft-mcguinness-oauth-actor-receipts-transparency.md),
[SCITT-aligned](./draft-mcguinness-oauth-actor-receipts-scitt.md))
log receipts to a transparency service.  In principle the same
mechanism could log proofs.  This profile does not specify proof
logging; deployments that want transparency for proofs should
extend the relevant transparency companion.

### 13.3 Disclosure profiles

Disclosure profiles (subset, actor-only) defined in a future
companion would apply uniformly to proofs and receipts: the
recipient sees whichever subset the issuer chose to disclose.

## See Also

- [draft-mcguinness-oauth-actor-profile](../draft-mcguinness-oauth-actor-profile.md)
 : core profile this builds on.
- [draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md)
 : receipts companion this composes with.
- [Transparency JWT-native variant](./draft-mcguinness-oauth-actor-receipts-transparency.md)
- [Transparency SCITT-aligned variant](./draft-mcguinness-oauth-actor-receipts-scitt.md)
- [RFC 8693 (OAuth Token Exchange)](https://www.rfc-editor.org/rfc/rfc8693)
- [RFC 8707 (Resource Indicators)](https://www.rfc-editor.org/rfc/rfc8707)
- [draft-mw-oauth-actor-chain](https://datatracker.ietf.org/doc/draft-mw-oauth-actor-chain/)
 : separate work proposing actor-signed step proofs in a different
  layering; this sketch is the actor-signed-proofs portion of the
  decomposition discussed in repo issues.

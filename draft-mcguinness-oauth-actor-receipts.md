---
title: "OAuth Actor Receipts for Delegation Provenance"
abbrev: "OAuth Actor Receipts"
category: std
docname: draft-mcguinness-oauth-actor-receipts-latest
submissiontype: IETF
number:
date: 2026-07-03
ipr: "trust200902"
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - oauth
 - delegation
 - actor
 - provenance
 - receipt
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-actor-profile"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-receipts.html"

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC6749:
  RFC6750:
  RFC6838:
  RFC6920:
  RFC7515:
  RFC7519:
  RFC7662:
  RFC7800:
  RFC8259:
  RFC8414:
  RFC8693:
  RFC8705:
  RFC8725:
  RFC9449:
  RFC9728:
  I-D.ietf-oauth-transaction-tokens:
  I-D.mcguinness-oauth-actor-profile:

informative:
  RFC9493:
  RFC9700:
  I-D.mw-oauth-actor-chain:
  I-D.liu-oauth-chain-delegation:
  I-D.liu-oauth-authorization-evidence:

...

--- abstract

This document defines OAuth Actor Receipts, an optional companion provenance profile for delegated OAuth tokens that conform to the OAuth Actor Profile for Delegation.  It introduces the `actor_receipts` claim, a signed per-hop receipt chain that records which issuer added each visible actor hop, optionally preserves the historical top-level `cnf` value associated with that hop subject to deployment disclosure policy, and links receipts together so recipients can validate prior-hop provenance without relying solely on the current outer token issuer.  This document also defines metadata and introspection parameters for advertising and consuming actor-receipt support.

--- middle

# Introduction

The OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}} makes actor identity visible in delegated tokens through a common `act` claim.  A relying party can read that chain but cannot, on the basis of the outer token alone, verify prior hops independently of the current token issuer.

This document defines OAuth Actor Receipts, an optional companion profile that adds independently signed per-hop provenance.  Each issuer that adds an actor hop signs a receipt for that hop; receipts travel with the token, are linked into a hash chain, and are validated against each issuer's published keys.  The design center is:

*  keep the visible actor chain in `act`;
*  keep active presenter proof of possession in the token's top-level `cnf`;
*  carry prior-hop provenance, including historical top-level `cnf` values, in separately signed hop receipts.

Receipts add a top-level JWT claim and a small set of metadata signals on top of the existing OAuth ({{RFC6749}}, {{RFC8693}}) and core-actor-profile trust model; deployments opt in per resource server or per trust domain.  Scope is detailed in [Design Goals and Non-Goals](#design-goals-and-non-goals).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Unless otherwise specified, OAuth terms such as client, authorization server, resource server, access token, refresh token, grant, `subject_token`, and `actor_token` are used as defined in {{RFC6749}} and {{RFC8693}}.  Transaction Token and Transaction Token Service (TTS) are used as defined in {{I-D.ietf-oauth-transaction-tokens}}.

The following terms are used in this document:

Actor Receipt:
: A signed JWT that attests one visible actor hop in a delegated token chain.

Outer Token:
: The access token or Transaction Token (JWT-formatted or opaque) with which a receipt chain is associated.  JWT outputs carry the `actor_receipts` claim inline; opaque tokens have their receipts returned via introspection (see {{consumer-introspection}}).  Distinguished from the receipt JWTs nested within it.

Receipt Chain:
: The ordered `actor_receipts` array carried in a token or introspection response.

Historical Presenter Binding:
: The presenter-binding state of a hop at the time the hop was created.  When recorded, it appears as the receipt's `cnf` claim, equal to the top-level `cnf` value of the token issued at that hop.  Historical presenter binding is informational provenance; it does not create an active proof-of-possession obligation for the current request.

Complete Receipt Coverage:
: A condition in which the number of receipts in `actor_receipts` equals the number of visible actor hops in the token's `act` chain, and every receipt aligns with the corresponding visible hop.

Examples in this document are illustrative and omit unrelated claims, signatures, and validation steps that a complete deployment would need.

# Relationship to the Core Actor Profile

This document is an extension of {{I-D.mcguinness-oauth-actor-profile}}.  A token that uses the `actor_receipts` claim defined here:

*  MUST conform to the actor-chain representation rules of the core actor profile;
*  MUST use the top-level `cnf` claim, when present, only for the current token presenter;
*  MUST NOT use nested `act` objects to carry independently trusted prior-hop key history.

Actor receipts do not replace the visible `act` chain.  The `act` chain remains the interoperable representation of current delegated identity.  Receipts are an additional provenance layer that can be validated by issuers and recipients that support this profile.

This document also does not redefine the request semantics of {{RFC8693}} or any Transaction Token request semantics.  It defines only:

*  the `actor_receipts` claim;
*  the signed JWT format of each receipt;
*  issuer and consumer processing for those receipts;
*  associated metadata and introspection parameters.

## Relationship to Token Introspection

Receipt-based provenance and OAuth Token Introspection ({{RFC7662}}) address overlapping but distinct needs:

*  **Introspection** returns the current AS-controlled state of a token at the time of the introspection call.  It tells the resource server whether the token is currently active and reflects the AS's current view of the token's claims.
*  **Receipts** preserve signed cross-hop provenance that travels with the token, survives across trust boundaries, may remain useful after intermediate systems become unreachable or are decommissioned, and supports detached verification by parties that do not have access to the issuing AS's introspection endpoint.

A deployment may use either, both, or neither:

*  Receipts MAY be returned via introspection, providing both signals in a single response (see {{consumer-introspection}}).
*  Receipts MAY be carried inline in JWT tokens for deployments where introspection is not available or where detached verification is required.
*  Introspection MAY be used for active-status checks even when receipts are not in use.

This document does not standardize the choice between receipts, introspection, and inline-only receipt-bearing tokens; that choice is a deployment decision driven by the recipient's trust framework, the network availability of introspection endpoints, and the audit and verification requirements of the deployment.

## Relationship to Other Delegation-Evidence Work

Several contemporaneous efforts record delegation evidence for OAuth tokens; they differ from this profile chiefly in where evidence lives and who can verify it.  This section is informative.

{{I-D.mw-oauth-actor-chain}} carries an issuer-signed cumulative commitment in the token while retaining the underlying per-hop evidence at the authorization server; recipients verify commitment continuity but depend on issuer retention and out-of-band access for the evidence itself.  Receipts under this profile are the evidence: each hop's attestation travels with the token and is validated by recipients directly against the attesting issuer's keys, with no retention or retrieval dependency, at the cost of per-hop token growth.

{{I-D.liu-oauth-authorization-evidence}} records a single authorization-server-signed consent event inline using Rich Authorization Requests machinery, and {{I-D.liu-oauth-chain-delegation}} extends the same approach to per-hop delegation records.  Those records are not chained to one another and are re-signed at trust-domain boundaries, whereas receipts are byte-preserved standalone JWTs linked by `prh`, so recipients verify each hop against the issuer that created it rather than against the most recent re-signing issuer.  These designs address overlapping needs; convergence is a working-group discussion this document aims to inform rather than preempt.

# Design Goals and Non-Goals

The goals of this document are:

*  preserve independently signed provenance for each visible actor hop;
*  preserve historical top-level `cnf` values without putting them back into the core `act` claim;
*  allow downstream recipients to validate prior-hop provenance against the issuers that created those hops;
*  defer to OAuth ({{RFC6749}}, {{RFC8693}}) and the core actor profile for current-token trust establishment, authorization, audience scoping, and sender-constraint validation;
*  add provenance through additive top-level claims and metadata signals, with no required changes to client request flows;
*  support progressive deployment, including tokens with partial receipt coverage.

The non-goals of this document are:

*  replacing the outer token's own signature or issuer trust model;
*  redefining OAuth audience semantics, scope evaluation, or AS-to-RS trust establishment;
*  redefining sender-constrained token validation for the current presenter;
*  requiring clients to change request flows, or requiring resource servers that do not consume receipts to change resource-protection logic;
*  proving that a particular historical scope, audience, or token lifetime was in force when a receipt was created;
*  reconciling subject identifiers that differ across receipts (see {{subject-re-expression-across-hops}});
*  defining transparency logs, non-repudiation systems, or public audit infrastructure.

## Deployment Fit

This profile's value scales with the number of distinct issuers in the trust set.  In federated or cross-domain deployments where multiple authorization servers and Transaction Token Services each contribute hops, receipts let recipients validate each hop against its own issuer rather than relying on the current outer token issuer's faithful preservation of the chain.  In single-domain deployments where the outer token issuer is the only trusted source for delegation state, the outer token's signature already conveys that attestation; receipts add overhead without meaningful additional provenance and SHOULD be weighed before adoption.  The trust-configuration cost scales with the same issuer count that drives the value: every receipt issuer must be within the recipient's pre-configured trusted-issuer set ({{trust-in-receipt-issuers}}), so deployments beyond bilaterally negotiated or federation-managed scale require trust-distribution infrastructure outside the scope of this document.

Authorization servers that mint refresh tokens for receipt-bearing access tokens incur additional issuer-side cost: receipts must be retained in issuer-controlled storage across refresh ({{reissuance-without-a-new-actor-hop}}).  Fully stateless JWT issuance does not satisfy this unless the receipts are embedded in the refresh-token state itself, for example in a self-contained refresh token that carries the receipt array.

Receipts mitigate a compromised or dishonest *downstream* issuer fabricating prior-hop provenance (see {{trust-in-receipt-issuers}} and {{compromised-outer-issuer}}).  They do not mitigate a compromised current outer token issuer and do not provide actor non-repudiation; those properties require mechanisms outside this document.

# Actor Receipts Overview

An actor receipt records one actor hop.  The issuer that adds a new outermost actor hop signs a receipt describing that hop and, when the issued token is sender-constrained, may copy the token's top-level `cnf` value into the receipt as historical presenter-binding context subject to the disclosure considerations in {{receipt-claims}}.

The token then carries an `actor_receipts` array:

*  one array entry per covered actor hop;
*  newest receipt first;
*  older receipts preserved unchanged.

This ordering aligns directly with the visible `act` chain in the outer token.  `actor_receipts[0]` corresponds to the outermost `act` object, `actor_receipts[1]` corresponds to `act.act`, and so on.

Receipts can cover either:

*  the full visible chain; or
*  a contiguous outermost prefix of the visible chain.

When a deployment requires full provenance, local policy or resource requirements enforce complete receipt coverage.

Receipts are historical provenance, not authority transfer.  A valid receipt chain documents prior delegation state but does not by itself convey authority, authorization, entitlement, or delegation rights.  Validation of a receipt does not imply that the represented delegation remains active, authorized, or acceptable under current policy; current authorization decisions MUST evaluate the current outer token, current policy, and current state, not the receipt chain alone.

# The `actor_receipts` Claim {#actor-receipts-claim}

`actor_receipts` is a new top-level JWT claim for tokens that conform to the core actor profile and this companion provenance profile.

`actor_receipts`:
: OPTIONAL.  An array of strings.  Each string MUST be the compact serialization of a signed JWT receipt as defined in {{actor-receipt-jwt-format}}.  When present, the array:

  *  MUST NOT be empty; issuers MUST omit the claim rather than including an empty array;
  *  MUST be ordered from newest covered hop to oldest covered hop;
  *  MUST NOT contain more entries than the visible actor-chain depth of the token's `act` claim;
  *  MUST represent a contiguous outermost prefix of the visible `act` chain.

If a token carries `actor_receipts`, it MUST also carry an `act` claim conforming to the core actor profile.

`actor_receipts_complete`:
: OPTIONAL.  A boolean JWT claim in the outer token.  When `true`, the issuer attests that `actor_receipts` covers every visible hop in the token's `act` chain.

  The attestation is relative to the visible chain at issuance time; it does not attest that the visible chain is itself unfiltered (see `chain_complete` in the core actor profile {{I-D.mcguinness-oauth-actor-profile}}).  Consumer enforcement, including the count-equality check, is defined in step 4 of {{consumer-processing}}.

  Issuers SHOULD set `actor_receipts_complete: true` when they emit complete coverage, to enable consumers to detect chain truncation.  Issuers SHOULD set `actor_receipts_complete: false` when they emit partial coverage.  Omitting the claim is observationally equivalent to `false` for consumers that test only for the literal value `true`, but does not provide a positive attestation that coverage is partial.

This document does not require every delegated token to carry `actor_receipts`.  A deployment that requires provenance receipts uses local policy or the metadata defined in {{discovery-capability-signaling}} to express that requirement.

# Actor Receipt JWT Format {#actor-receipt-jwt-format}

Each element of `actor_receipts` is a signed JWT represented using JWS compact serialization {{RFC7515}}.

## JOSE Header

The JOSE header of an actor receipt:

*  MUST include an asymmetric digital-signature `alg` value;
*  MUST NOT use `alg: none` or a MAC-based symmetric algorithm;
*  MUST include `typ` with the value `actor-receipt+jwt`;
*  SHOULD include `kid` when the issuer publishes multiple verification keys;
*  MAY include `crit` per {{RFC7515}}; consumers MUST reject a receipt whose `crit` header lists an extension header the consumer does not understand.

Receipt issuers and consumers MUST apply the JWT best practices in {{RFC8725}}.

## Receipt Claims

The JWT payload of an actor receipt uses the claims defined below, grouped by purpose.

### Identity Claims

`iss`:
: REQUIRED.  The issuer that created and signed the receipt for the corresponding actor hop.  This value identifies the receipt signer.

  The `act.iss` value inside the receipt identifies the namespace authority for `act.sub`, using the meaning defined by the core actor profile.  These two values MAY be the same entity or different entities.  When they differ, consumers MUST evaluate two distinct trust questions:

  *  trust in `iss` as a receipt signer: whether this issuer's signature attests receipts under local policy;
  *  trust in `act.iss` as the namespace authority for `act.sub`: whether this issuer's namespace produces actor identifiers the recipient accepts.

  These evaluations are independent even when the same entity holds both roles.  The difference between `iss` and `act.iss` alone does not make the receipt invalid under this profile.

`sub`:
: REQUIRED.  The top-level `sub` value that was present in the token issued at this hop.

  Receipt `iss` is the receipt signer; it is NOT the namespace authority for receipt `sub`.  Receipt `sub` records the subject identifier as it appeared in the token at the represented hop, interpreted under the namespace authority that governed that token.  That namespace authority is contextual to the represented hop and MAY differ across receipts in the same chain (see {{subject-re-expression-across-hops}}).  Recipients reconciling differing `sub` values across receipts apply trusted local mapping rules; this profile does not define an in-band mechanism for cross-namespace subject reconciliation.

`sub_iss`:
: OPTIONAL.  The namespace authority under which the receipt `sub` value is interpreted.

  When present, `sub_iss` identifies the issuer or namespace authority for receipt `sub` in the same way that `act.iss` identifies the namespace authority for `act.sub` under the core actor profile.  When absent, the subject namespace authority is not independently expressed by this profile and MUST be determined, if needed, from trusted local context for the represented hop.  This profile deliberately uses a flat issuer-plus-subject claim pair, mirroring the (`act.iss`, `act.sub`) convention of the core actor profile, rather than the structured Subject Identifier formats of {{RFC9493}}: receipt claims record token claim values as issued, and the represented token carries its subject as a flat `sub` claim.

  This profile and the core actor profile {{I-D.mcguinness-oauth-actor-profile}} do not define a top-level claim on the outer token that names the namespace authority for the outer token's `sub`.  Recipients reconciling `receipt[0].sub_iss` with the outer token's top-level `sub` therefore obtain the outer-token subject namespace authority from trusted local context, an inbound subject token's claims, or another deployment-defined source.

`sub_profile`:
: OPTIONAL.  The top-level `sub_profile` value, when the token issued at this hop carried one.

`act`:
: REQUIRED.  A single-hop actor object.  This object:

  *  MUST conform to the core actor profile's actor-object rules;
  *  MUST include `act.sub` and `act.iss`;
  *  MAY include `act.sub_profile`;
  *  MUST NOT contain `cnf`;
  *  MUST NOT contain a nested `act`.

  `act.cnf` is prohibited to avoid ambiguity between historical key binding and a current PoP requirement on the actor; historical presenter binding belongs at the receipt's top level (the `cnf` claim), where its role as provenance-only is unambiguous.  Additional claims permitted by the core actor profile in `act` objects MAY appear in receipt `act` objects unless explicitly prohibited here.  A receipt that carries `act.cnf` is invalid under this profile.

### Historical Presenter Binding

`cnf`:
: OPTIONAL.  A confirmation claim as defined in {{RFC7800}}.  When present, it MUST equal the top-level `cnf` claim of the token issued at this hop.

  Receipt `cnf` records historical presenter-binding information for the hop represented by the receipt.  It does not create a current proof-of-possession obligation for the current request.

  Issuers SHOULD NOT include `cnf` in a receipt unless the relying parties that will receive the token have been evaluated for the associated disclosure risk; omitting `cnf` does not invalidate the receipt.

### Chain Linkage

`prh`:
: OPTIONAL.  Previous receipt hash.  When present, `prh` MUST be the base64url encoding without padding ({{RFC7515}}) of the hash of the ASCII octets of the complete compact serialization of the next older receipt in the chain, computed using the algorithm identified by `prh_alg` (defaulting to SHA-256 when `prh_alg` is absent).  The oldest receipt in the chain, including a single-element chain in which the sole receipt is both newest and oldest, MUST omit `prh`.

`prh_alg`:
: OPTIONAL.  Hash algorithm identifier naming the algorithm used to compute `prh`.

  *  Values MUST be drawn from the IANA "Named Information Hash Algorithm Registry" {{RFC6920}}, which uses lowercase forms such as `sha-256`, `sha-384`, and `sha-512`.
  *  When absent, the default is `sha-256`.
  *  When present, the value MUST identify a hash algorithm whose collision and preimage resistance is at least equivalent to `sha-256`.
  *  All receipts in a single `actor_receipts` array MUST use the same `prh_alg` value, so that recipients can validate the chain without per-receipt algorithm negotiation.  This uniformity rule is deliberately syntactic: a chain in which some receipts omit `prh_alg` and others carry an explicit `sha-256` is rejected even though the algorithm is the same, so that consumers never reconcile explicit values against defaults.  A chain that omits `prh_alg` from every receipt uses the SHA-256 default for every `prh` value.  A single-element chain MAY carry `prh_alg`, but the value has no effect unless a later receipt links to it.
  *  An issuer extending an inbound chain MUST either preserve the inbound `prh_alg` or reject the chain.

### Time and Uniqueness

`iat`:
: REQUIRED.  The time at which the receipt was created, as defined in {{RFC7519}}.

`exp`:
: REQUIRED.  Expiration time for the receipt, as defined in {{RFC7519}}.

  *  `exp` MUST be set to a value that covers the expected maximum token lifetime of any token that will carry or inherit this receipt, so that consumer validation of older receipts in a valid chain is not prematurely rejected.
  *  Issuers SHOULD set `exp` to the maximum delegated-token lifetime permitted under local policy for tokens that may inherit this receipt.

  Downstream issuers reject inbound receipts whose `exp` precedes the issued outer token's `exp` ({{extending-an-existing-receipt-chain}}), so an under-set `exp` causes propagation failure rather than mid-token-lifetime consumer rejection.  Short `exp` values on receipts limit the window during which a compromised receipt signing key can be exploited.  Because the originating issuer cannot enumerate every downstream issuer that may inherit a receipt, deployments typically derive the originating `exp` from a bounded delegated-session lifetime ({{reissuance-without-a-new-actor-hop}}) coordinated across the trust set.

`jti`:
: REQUIRED.  A unique identifier for the receipt, as defined in {{RFC7519}}.

### Outer-Token Binding

`origin_jti`:
: RECOMMENDED.  The `jti` of the outer token at the time this receipt was created (the receipt's origin outer token).  This value is fixed at receipt creation; after reissuance the current outer token's `jti` may differ.

  This value is a historical record of which outer token the receipt was originally issued with.  It does not bind the receipt to the current outer token.

  There is one exception.  When `receipt[0].iss` equals the outer token's `iss` and `receipt[0].origin_jti` equals the outer token's `jti`, the value binds `receipt[0]` to the current outer-token instance.  This is the originating-issuance case.

  Issuers SHOULD include `origin_jti` whenever the token they are issuing carries a `jti` claim.  Recipients that require current-token-instance binding in strict mode MUST require `origin_jti` on `receipt[0]` when the outer token carries `jti`; otherwise the receipt chain is valid only as issuer-signed provenance and lacks the `origin_jti` binding property described in {{receipt-to-token-binding-limits}}.

  Consumer verification of `origin_jti` is defined in step 5 of {{consumer-processing}}, with strict-mode rejection of `receipt[0]` divergence in {{strict-mode-validation}}.

### Excluded Standard Claims

`aud`:
: NOT RECOMMENDED.  Issuers SHOULD omit `aud` from receipts.

  Receipts are validated as part of outer-token processing, not as independent JWTs against an audience; the outer token carries the audience scoping for the request.  This profile diverges from the audience-validation guidance in {{RFC8725}} Section 3.9 on those grounds.  Including `aud` in a receipt has no defined meaning under this profile and would create ambiguity about whether the receipt asserts an independent audience constraint, which it does not.

### Extension Claims

A receipt MAY contain additional claims defined by another specification or by deployment policy.  Consumers MUST ignore unrecognized claims unless another specification or local agreement defines their meaning.

## Receipt-Chain Linkage

When the issuer creates a new receipt and prepends it to an inherited receipt chain:

*  if there is an older receipt immediately following it in the array, the new receipt MUST include `prh`, and that value MUST be the base64url encoding without padding of the hash of the ASCII octets of the exact compact JWT string of that next receipt;
*  if the new receipt is the only receipt in the array, it MUST omit `prh`.

No JSON {{RFC8259}} canonicalization is applied.  `prh` hashes the ASCII octets of the exact compact-serialized JWS string of the next older receipt as carried in the array, and the hash output is base64url-encoded without padding.  Systems that carry, store, or forward `actor_receipts` arrays MUST preserve each compact JWT string byte-for-byte; any modification, including semantically equivalent re-encoding, invalidates `prh` for any receipt that references it.

# Issuer Processing

This section defines how an authorization server or Transaction Token Service creates, preserves, and extends `actor_receipts`.

When an issuer adds a new outermost actor hop and creates the corresponding receipt, that issuer is the same entity that signs the outer token.  Consequently `receipt[0].iss` equals the outer token's `iss` for tokens emitted under {{creating-the-first-receipt}} or {{extending-an-existing-receipt-chain}}.  When the issuer includes `origin_jti`, `receipt[0].origin_jti` equals the outer token's `jti` in those originating-issuance cases.

Reissuance without a new actor hop ({{reissuance-without-a-new-actor-hop}}) is the only case in which `receipt[0]` may legitimately diverge from the current outer-token instance, either by `receipt[0].iss` differing from `outer.iss` or by `receipt[0].iss` matching but `receipt[0].origin_jti` differing from `outer.jti`.  Consumer rules in {{consumer-processing}} use this property to scope the bind-to-current checks for `receipt[0]`.

## Creating the First Receipt

When an issuer creates a delegated token with a new outermost actor hop and no inbound `actor_receipts` are being preserved, the issuer MAY create a new one-element `actor_receipts` array.

If it does so, the new receipt:

*  MUST describe the new outermost actor hop;
*  MUST set `sub` to the issued token's top-level `sub`;
*  MUST set `act.sub` and `act.iss` to the new outermost actor;
*  MAY copy the issued token's top-level `cnf`, if any, into the receipt `cnf`, subject to the disclosure considerations in {{receipt-claims}}; when copied, the receipt `cnf` MUST equal the outer token's `cnf` value;
*  SHOULD set `origin_jti` to the issued token's `jti`, if the issued token carries a `jti`;
*  MUST omit `prh`.

When the one-element array covers every visible hop (a visible `act` chain of depth 1), the issuer SHOULD set `actor_receipts_complete: true`; when inner visible hops remain uncovered, it SHOULD set `actor_receipts_complete: false`, per {{actor-receipts-claim}}.

## Extending an Existing Receipt Chain

When an issuer adds a new outermost actor hop and also preserves an inbound `actor_receipts` array, it:

1.  MUST validate the inbound receipt chain by applying the consumer processing rules in {{consumer-processing}} before relying on it or carrying it forward.
2.  MUST verify that each inbound receipt's `exp` is no earlier than the issued outer token's `exp`.  Carrying a receipt forward whose `exp` precedes the outer token's `exp` would cause consumers to reject the chain mid-token-lifetime under {{consumer-processing}}; an inbound receipt that fails this check is treated as failing validation under step 1.  Issuers MAY apply a small clock-skew margin to this comparison, consistent with the consumer-side skew tolerance in {{consumer-processing}}, but MUST NOT broadly accept inbound receipts whose `exp` precedes the issued outer token's `exp` by more than a deployment-defined skew bound.
3.  MUST preserve each inbound receipt byte-for-byte unchanged.
4.  MUST create exactly one new receipt for the new outermost actor hop.
5.  MUST prepend that new receipt to the inherited array.
6.  When the inherited array is non-empty, MUST set the new receipt's `prh` to the hash of the exact compact serialization of the receipt now at the next array index, computed using the algorithm named by `prh_alg` (defaulting to SHA-256 when `prh_alg` is absent).
7.  MUST set the new receipt's `prh_alg` to the inherited value, or omit `prh_alg` if the inherited chain omits it (preserving the SHA-256 default for the chain).  An issuer that does not support the inbound `prh_alg` value MUST reject the chain rather than rehash; rehashing would invalidate prior issuers' signatures.
8.  MUST set the issued outer token's `actor_receipts_complete` to `true` when the validated inbound chain carried `actor_receipts_complete: true` and the new receipt covers the new outermost actor hop; downgrading to absent or `false` in that case would falsely signal a coverage reduction to recipients that test for the literal value `true`.  When the issuer cannot attest complete coverage for the issued chain (for example, the inbound chain was partial or carried no completeness attestation), it MUST NOT set `actor_receipts_complete: true` and SHOULD set `actor_receipts_complete: false`, per {{actor-receipts-claim}}.

An issuer MUST NOT reserialize, resign, normalize, trim, or otherwise alter a prior receipt.

If inbound receipts fail validation, the issuer MUST NOT propagate them.  It MAY continue without `actor_receipts` only when local policy permits partial coverage; otherwise it MUST fail the request under the error model of the underlying protocol.

## Reissuance Without a New Actor Hop

An issuer that reissues, translates, or introspects and re-emits a token without adding a new outermost actor hop:

*  MAY carry an inbound `actor_receipts` array forward unchanged;
*  MUST NOT create a new receipt;
*  MUST carry the inbound `actor_receipts_complete` value forward unchanged when the inbound array is carried forward unchanged; an issuer that cannot continue to attest the inbound coverage value MUST drop the inbound array entirely rather than keep the array and silently downgrade `actor_receipts_complete`.  Receipt disclosure is all-or-nothing for a given token: a strict subset of an inherited array cannot validate under {{consumer-processing}} (see {{consumer-introspection}});
*  MUST NOT continue to carry an inherited `actor_receipts` array if it cannot preserve the visible hop alignment required by {{consumer-processing}};
*  MUST NOT change the outer token's top-level `sub` while carrying inherited receipts forward; a `sub` change re-expresses subject identity and breaks the alignment between `receipt[0].sub` and the outer token's top-level `sub` that consumers verify under {{consumer-processing}}.  A reissuing issuer that re-expresses `sub` into another namespace (for example, a token translator at a domain boundary) therefore drops inherited receipts; receipt-based provenance does not survive subject re-expression at reissuance.

If such an issuer changes the visible outermost actor, it has added a new hop and MUST follow {{extending-an-existing-receipt-chain}}.

A change in the outer token's top-level `cnf` value, by itself, does not invalidate carried-forward receipts.  Receipt `cnf` records the historical presenter binding in effect when that hop was created.  The current token's top-level `cnf` can later change, for example because of key rotation or token reissuance, without requiring a new receipt so long as the visible actor hop itself is unchanged.

Reissuance MAY change the outer token's `aud`, `scope`, `cnf`, `exp`, and other claims that bind the current request without requiring updates to inherited receipts.  Receipts attest hop history at the time of original issuance and are unaffected by later mutations of current-request bindings.  Only `sub` is structurally constrained, because changing `sub` while carrying inherited receipts would break the alignment between `receipt[0].sub` and the outer token's top-level `sub` that consumers verify under {{consumer-processing}}.

Reissuance under this section is the only case in which `receipt[0]` may legitimately diverge from the current outer-token instance.  Two patterns of divergence are possible:

*  **Different-issuer reissuance**: `receipt[0].iss` differs from the outer token's `iss`.  For example, an introspection endpoint operated as a separate trust principal re-emits the token, or a token translator at a domain boundary re-issues under its own issuer identity.
*  **Same-issuer reissuance**: `receipt[0].iss` matches the outer token's `iss`, but `receipt[0].origin_jti` differs from the outer token's `jti`.  For example, an authorization server refreshes its own access token: the refreshed token is signed by the same AS but carries a new `jti`, while inherited receipts (carried forward unchanged) retain the original outer-token `jti` in `origin_jti`.

Consumer rules in {{consumer-processing}} treat `receipt[0].origin_jti` as historical provenance, not as a current-token binding, in both reissuance patterns.  Recipients distinguish legitimate reissuance from a re-wrapping attack through local policy or out-of-band trust framework, as described in {{receipt-to-token-binding-limits}}.

Refresh-token reissuance is a special case of reissuance under this section.  Receipt-bearing refresh is interoperable only when local policy defines a bounded maximum delegated-session lifetime for tokens that may inherit the receipts.

An AS that supports refresh tokens for delegated access tokens:

*  MUST retain the `actor_receipts` array associated with the original access token in issuer-controlled state across refresh, either in durable storage (for example, a token-state database or refresh-token state) or embedded in a self-contained refresh token, so each refreshed access token can carry the receipts forward unchanged.
*  MUST set receipt `exp` values under {{receipt-claims}} to accommodate the bounded maximum delegated-session lifetime.  Otherwise downstream issuers reject inbound chains under {{extending-an-existing-receipt-chain}} as receipts approach expiry, and refresh loses receipt-based provenance.
*  When that bounded lifetime would be exceeded, MUST either obtain fresh delegation state and start a new receipt chain or stop emitting `actor_receipts`, unless local policy permits partial or absent receipt coverage.

## Partial Coverage and Full Coverage

This document permits partial receipt coverage for progressive deployment.  An issuer MAY begin a new receipt chain even when older inner actor hops remain visible but uncovered.

However:

*  a partial chain MUST still cover a contiguous outermost prefix of the visible actor chain;
*  an issuer MUST NOT skip an outer visible hop and receipt only an inner visible hop;
*  when local policy or resource requirements require full provenance, the issuer MUST either emit complete receipt coverage or fail the request under the error model of the underlying protocol.

Because coverage is a contiguous outermost prefix, partial coverage always omits the innermost (oldest) hops first.  In many delegation chains the innermost hop is the original subject-to-actor delegation, the hop with the greatest audit and accountability value.  Deployments that value evidence for that originating hop SHOULD deploy receipt support at the origin issuer first: once the origin issuer emits the first receipt, every downstream issuer that supports this profile extends the chain, and coverage is complete by construction.  Resource servers that require evidence for the originating hop enforce it through `actor_receipts_complete_required` ({{discovery-capability-signaling}}) or equivalent local policy.

When the issuer also filters the visible `act` chain (see the `chain_complete` introspection member defined in the core actor profile {{I-D.mcguinness-oauth-actor-profile}}), `actor_receipts` covers only the visible filtered chain.  In that case `actor_receipts_complete` describes coverage relative to the visible filtered chain, not the unfiltered delegation chain; recipients that need true-chain completeness MUST evaluate `chain_complete` separately.

For inline JWT tokens, this document defines no `chain_complete` JWT claim.  A recipient that needs true-chain completeness for inline JWT tokens MUST obtain that signal from trusted deployment context, introspection, or another profile; `actor_receipts_complete: true` alone attests only complete receipt coverage for the visible `act` chain.

## Transaction Token Service Rebinding

A Transaction Token Service that establishes a new presenter and makes that presenter the new outermost actor follows the same receipt rules as any other issuer that adds a new outermost actor hop, as defined in {{extending-an-existing-receipt-chain}} (or {{creating-the-first-receipt}} when no inbound `actor_receipts` exist).  TTS presenter rebinding inherently changes the outer token's top-level `cnf`; the new receipt records the new presenter's `cnf` when included (subject to {{receipt-claims}}), and inherited receipts retain their historical `cnf` values unchanged.  This profile does not define additional receipt claims specific to Transaction Tokens; any transaction-specific semantics remain governed by the Transaction Token itself and its deployment profile.

# Consumer Processing {#consumer-processing}

An issuer, resource server, or other recipient that relies on `actor_receipts` MUST perform the following steps.

1.  Validate the outer token according to its token type and the core actor profile.
2.  If `actor_receipts` is absent, treat the token as lacking receipt-based provenance.  Whether that is acceptable is determined by local policy or by Protected Resource Metadata signals such as `actor_receipts_required` and `actor_receipts_complete_required` defined in {{discovery-capability-signaling}}.  If `actor_receipts_complete` is present with the value `true` while `actor_receipts` is absent, the combination is malformed; the recipient MUST treat this as a failed required check and apply the rejection rule following step 11.
3.  Verify that `actor_receipts`, if present, is a non-empty JSON array of strings.  Verify that `actor_receipts_complete`, if present, is a JSON boolean.
4.  Verify that the number of receipts does not exceed the visible actor-chain depth of the outer token.  If the outer token carries `actor_receipts_complete: true`, verify that the receipt count exactly equals the visible actor-chain depth; if it does not, reject the token.
5.  For each receipt, in array order:
    *  parse the string as a compact JWT;
    *  verify that the receipt issuer is within the recipient's pre-configured trusted-issuer set before performing any network retrieval for that issuer's metadata or keys;
    *  resolve the signing key from the receipt issuer's authorization server metadata `jwks_uri` {{RFC8414}} (where the receipt issuer is identified by the receipt's `iss` claim, which may differ from the outer token's issuer) or from local configuration;
    *  validate the JWT signature;
    *  verify that the JOSE header uses an asymmetric digital-signature `alg` value accepted for that receipt issuer, and reject receipts that use `alg: none` or a MAC-based symmetric algorithm;
    *  verify that `typ` equals `actor-receipt+jwt`;
    *  reject a receipt whose `crit` header lists an extension header the consumer does not understand;
    *  verify that all REQUIRED receipt claims are present and have the expected JSON types, including `iss`, `sub`, `act`, `iat`, `exp`, and `jti`;
    *  verify that OPTIONAL claims used by this profile have the expected JSON types when present, including `sub_iss`, `sub_profile`, `cnf`, `prh`, `prh_alg`, and `origin_jti`;
    *  verify that the receipt `act` object is single-hop, contains no nested `act`, and contains no `cnf`;
    *  enforce `exp`, `iat`, and other JWT validity rules.  Because `exp` is REQUIRED on receipts and MUST cover the expected outer token lifetime, an expired receipt SHOULD be treated as invalid even for older hops.  Local policy MAY permit continued use of a receipt that is expired by a small clock-skew margin, but MUST NOT relax `exp` enforcement broadly as a workaround for issuers that failed to set adequate `exp` values.
    *  for `receipt[0]`: anchor it to the current outer-token instance when `receipt[0].iss` equals the outer token's `iss` and `receipt[0].origin_jti` is present and equals the outer token's `jti` (the originating-issuance case).  When `receipt[0].iss` equals the outer token's `iss` and the outer token carries no `jti`, accept the chain as issuer-signed provenance without current-token-instance binding; any `receipt[0].origin_jti` value is then informational only.  If `receipt[0].iss` equals the outer token's `iss` but `origin_jti` is absent while the outer token carries `jti`, the recipient MAY accept the chain as issuer-signed provenance under local policy, but MUST NOT treat it as bound to the current outer-token instance.  Otherwise, the recipient MUST reject the chain unless local policy designates the current outer token issuer (`outer.iss`) as a trusted reissuing issuer for receipt chains whose leading receipt issuer is `receipt[0].iss`, per {{receipt-to-token-binding-limits}}.  For receipts other than `receipt[0]`, `origin_jti` is informational provenance only.
6.  Verify receipt-chain linkage:
    *  each receipt other than the oldest MUST include `prh`;
    *  each non-oldest receipt's `prh` MUST hash the next older receipt using the algorithm named by `prh_alg`, defaulting to `sha-256` when `prh_alg` is absent;
    *  all receipts in the chain MUST carry the same `prh_alg` value (or all omit it); a mixed-algorithm chain MUST be rejected;
    *  the named algorithm MUST be one the recipient supports; a chain naming an unsupported algorithm MUST be rejected;
    *  the oldest receipt MUST omit `prh`.
7.  Verify visible-hop alignment:
    *  `receipt[0].act.sub` MUST equal the outer token's `act.sub`, and `receipt[0].act.iss` MUST equal the outer token's `act.iss`;
    *  `receipt[1].act.sub` MUST equal the outer token's `act.act.sub`, and `receipt[1].act.iss` MUST equal the outer token's `act.act.iss`;
    *  and so on for the number of receipts present;
    *  when `act.sub_profile` is present in the receipt `act` object, the corresponding visible `act` object MUST contain `act.sub_profile` with the same value;
    *  when `act.sub_profile` is present only in the visible `act` object, the receipt remains aligned for this profile.  The visible value is not independently attested by that receipt, and recipients that require receipt coverage for actor classification MUST reject the receipt chain or apply explicit local mapping rules.
8.  Verify subject alignment:
    *  `receipt[0].sub` MUST equal the outer token's top-level `sub`;
    *  when `receipt[0].sub_iss` is present and the recipient has a top-level subject namespace authority for the outer token's `sub` from local configuration, an inbound subject token's claims, or another deployment-defined source, the two MUST identify the same namespace authority, evaluated by case-sensitive string comparison; treating lexically distinct identifiers as the same authority requires explicit trusted local mapping rules;
    *  when `receipt[0].sub_profile` is present and the outer token contains top-level `sub_profile`, the values MUST match;
    *  when `receipt[0].sub_profile` is present but the outer token does not contain top-level `sub_profile`, recipients that require receipt coverage for subject classification MUST reject the receipt chain or apply explicit local mapping rules;
    *  when `receipt[0].sub_profile` is absent but the outer token contains top-level `sub_profile`, the receipt remains aligned for this profile.  The visible value is not independently attested by that receipt, and recipients that require receipt coverage for subject classification MUST reject the receipt chain or apply explicit local mapping rules;
    *  older receipts MAY carry differing `sub`, `sub_iss`, or `sub_profile` values; see {{subject-re-expression-across-hops}}.
9.  Treat each receipt `cnf` value, if present, only as historical provenance for that hop.  A mismatch between the current outer token's top-level `cnf` and the outermost receipt `cnf` MUST NOT by itself invalidate the receipt chain under this profile.
10.  Receipt `cnf` values MUST NOT replace validation of the current request against the outer token's top-level `cnf`.
11.  Apply any additional consumer-processing rules defined by companion profiles whose claims appear in the receipt or outer token (see {{extensibility}}).  Companion-profile rules MUST NOT relax any requirement in steps 1 through 10; they MAY add additional rejection conditions.

If any required check fails, the recipient MUST reject the receipt chain for the purposes of this profile and MUST apply the underlying protocol's error handling for the stage at which the failure occurred.

A recipient that has rejected a receipt chain under this profile MAY, under explicit local policy, extract structural information from the chain for use by companion profiles (for example, applying a companion's verification rules to the trusted prefix of an otherwise-invalid chain).  The recipient MUST NOT treat such partial validation as conformance with this profile, and MUST NOT relax the rejection requirements defined above.  Companion profiles defining partial-validation modes MUST do so under their own normative scope.

## Subject Re-Expression Across Hops {#subject-re-expression-across-hops}

Older receipts can carry a different `sub` value from the current outer token when the subject has been re-expressed across issuer namespaces.  This document does not define a universal subject-mapping algorithm.

Accordingly:

*  only `receipt[0].sub` is required to equal the current outer token `sub`;
*  older receipt `sub` values MAY differ;
*  a recipient that applies stronger continuity requirements across older `sub` values MUST do so under explicit trusted local mapping rules.

Recipients MUST be aware that permitting differing `sub` values across receipts creates a cross-subject insertion risk: a receipt from an unrelated subject chain that happens to share the same actor identity could satisfy the structural hop-alignment check.

This risk is not merely accidental.  An attacker who compromises any single upstream issuer can deliberately mint receipts for any subject in that issuer's namespace and graft them onto a downstream chain whose re-expressed `sub` points to a target subject; the graft satisfies structural hop-alignment because the actor identity at the grafted hop matches a hop that legitimately occurred for the target.

This profile provides no in-band mechanism for cross-namespace subject reconciliation.

Deployments where subject continuity is a security requirement SHOULD adopt one of the following:

*  require consistent `sub` values across all receipts in the chain, rejecting re-expressed chains; or
*  enforce explicit trusted subject-mapping rules that can positively confirm each distinct `sub` value refers to the same underlying entity.

When neither condition is met, the recipient MUST treat the differing `sub` values as unverified subject continuity and MUST NOT rely on those older receipts for authorization decisions.

## Complete Receipt Coverage

A recipient can infer structural complete receipt coverage by comparing receipt count with visible actor depth.  If the number of receipts equals the visible actor depth and all validation rules above succeed, the token has structurally complete receipt coverage for the visible chain.

When local policy only needs structural completeness, that inferred count match is sufficient.  When Protected Resource Metadata declares `actor_receipts_complete_required: true`, the token or introspection response MUST also carry `actor_receipts_complete: true`; a count match without that explicit issuer attestation does not satisfy the metadata-declared requirement.

If local policy or resource requirements require full provenance, the recipient MUST reject tokens that do not satisfy the applicable complete-coverage requirement.  For `actor_receipts_complete_required: true`, this requires both structurally complete receipt coverage and `actor_receipts_complete: true`.

## Use by Resource Servers

Resource servers can use validated actor receipts as provenance input for authorization, diagnostics, and audit.  However, a valid receipt chain:

*  proves only that trusted issuers attested specific visible actor hops;
*  does not prove that the current token's audience, scope, or expiration were in force when older receipts were created;
*  does not replace the need to authorize the current token itself;
*  does not convey authority, authorization, entitlement, or delegation rights;
*  does not imply the represented delegation remains active or acceptable under current policy.

A resource server that bases an authorization decision on receipt content alone, without re-evaluating the current outer token and current policy, mis-uses this profile.

## Introspection {#consumer-introspection}

When an authorization server returns actor-receipt information in an OAuth Token Introspection response {{RFC7662}}, it:

*  MAY return `actor_receipts` using the same array format defined in {{actor-receipts-claim}};
*  MAY return `actor_receipts_complete` to indicate whether the returned array provides complete coverage for the visible chain as known to the introspection server.

The registered introspection response members are defined in {{introspection-response-members}}; introspection-server failure handling is addressed in {{introspection-errors}}.

Introspection is the primary delivery mechanism for receipts associated with opaque (non-JWT) outer tokens.  Such tokens cannot carry an inline `actor_receipts` claim; the issuer instead retains the receipts in its token store and surfaces them to authorized resource servers via introspection.  The receipt format and consumer processing rules above apply unchanged in this case, with the introspection response substituting for the outer token's claim set.

An introspection response that includes `actor_receipts` MUST include the members needed to perform the consumer processing in {{consumer-processing}}: the token's top-level `sub`, the visible `act` chain, and the token's `iss`, which the `receipt[0]` anchoring check in step 5 compares against `receipt[0].iss`.  When the introspection server maintains a `jti` for the token, the response SHOULD include it so that recipients can evaluate `origin_jti` anchoring; when `jti` is absent from the response, recipients apply the no-`jti` branch of step 5.  If `actor_receipts_complete` is present, it MUST be a JSON boolean.  A resource server that receives both inline receipts in a JWT token and receipts in an introspection response MUST apply local policy to choose the authoritative source; if both sources are consumed together, mismatched `actor_receipts` or `actor_receipts_complete` values MUST cause the resource server to reject receipt-based provenance for the token.

Receipt disclosure through introspection is all-or-nothing for a given token: a strict subset of the stored array cannot validate under {{consumer-processing}}, because removing an older receipt leaves the next newer receipt's `prh` without a target and removing the newest receipt breaks visible-hop alignment.  An introspection server that cannot disclose the full stored array for privacy or policy reasons MUST omit `actor_receipts` from the response entirely.  A stored array that itself has partial coverage is returned in full, with `actor_receipts_complete: false`.

When the introspected token is revoked or otherwise inactive, the introspection response follows the core actor profile's suppression rule for delegation claims: an introspection server MUST NOT return `actor_receipts` or `actor_receipts_complete` for a token it reports as inactive.

The core actor profile {{I-D.mcguinness-oauth-actor-profile}} defines a separate `chain_complete` introspection member that indicates whether the visible `act` chain itself has been filtered.  These two completeness signals are distinct: `chain_complete: false` means the introspection server has suppressed inner `act` hops from the chain representation, while `actor_receipts_complete: false` means receipt coverage of the visible chain is partial.  A token can have `chain_complete: true` and `actor_receipts_complete: false`, or vice versa.

Consumers that rely on both signals MUST evaluate them independently.  When `chain_complete: false`, the receipt array may cover only part of the true delegation chain even when `actor_receipts_complete: true`; receipt coverage is then complete only for the visible filtered chain, not the full chain.

# Discovery and Capability Signaling {#discovery-capability-signaling}

This section defines metadata for advertising support for actor receipts.

## Authorization Server Metadata

The following parameter is defined for use in Authorization Server Metadata {{RFC8414}}:

`actor_receipts_supported`:
: OPTIONAL.  A boolean.  When `true`, the authorization server advertises that it can validate inbound actor receipts and can originate, preserve, or extend receipt chains according to this document.  This value does not guarantee complete historical coverage for every visible hop in every resulting token.  When `false` or absent, clients and relying parties MUST NOT assume such support.

This parameter applies equally to an authorization server that issues delegated JWT outputs and to a Transaction Token Service publishing metadata through the same framework.

## Protected Resource Metadata

The following parameters are defined for use in Protected Resource Metadata {{RFC9728}}:

`actor_receipts_required`:
: OPTIONAL.  A boolean.  When `true`, the resource server indicates that delegated requests are expected to carry valid actor receipts covering at minimum the outermost visible actor hop.  When `false` or absent, the resource server makes no metadata declaration about receipt-based provenance requirements.

  This parameter is a policy declaration for deployment coordination, not a request-time protocol signal: this document defines no parameter by which a client requests receipt issuance, so the declaration is satisfied by configuring receipt issuance at the authorization servers that serve the resource.  Clients MAY use it, together with `actor_receipts_supported`, to select an authorization server that can issue receipt-bearing tokens.

`actor_receipts_complete_required`:
: OPTIONAL.  A boolean.  When `true`, the resource server indicates that it requires complete receipt coverage: the receipt count must equal the visible actor-chain depth and `actor_receipts_complete` must be `true` in the outer token or the introspection response.  This parameter refines `actor_receipts_required`; a resource server SHOULD NOT set `actor_receipts_complete_required: true` without also setting `actor_receipts_required: true`.  When `false` or absent, partial receipt coverage is acceptable to the resource server, subject to any further local policy.

## Introspection Response Members {#introspection-response-members}

The following members are defined for use in OAuth Token Introspection responses {{RFC7662}}:

`actor_receipts`:
: OPTIONAL.  An array of strings using the same syntax as the JWT claim of the same name.

`actor_receipts_complete`:
: OPTIONAL.  A boolean.  When `true`, the introspection response indicates that the returned `actor_receipts` cover every visible hop in the token chain as known to the introspection server.  When `false`, the response indicates that the returned receipts provide only partial coverage of the visible chain.

Consumer use of these members is described in {{consumer-introspection}}; introspection-server failure handling is addressed in {{introspection-errors}}.

## Out-of-Scope Discovery Signals

This document does not define a metadata signal for "this resource server requires `cnf` to be present in receipts."  Issuers default to omitting receipt `cnf` for privacy reasons (see {{receipt-claims}} and {{historical-cnf-disclosure}}); resource servers that need historical sender-constraint provenance MUST coordinate that requirement with issuers through deployment policy or a future companion profile, rather than through metadata defined here.

## Claim-Pair Convention for Sibling Profiles

This document defines the per-hop artifact array `actor_receipts` together with the coverage attestation `actor_receipts_complete`, and the discovery triple (`actor_receipts_supported`, `actor_receipts_required`, `actor_receipts_complete_required`).  Companion profiles that define their own per-hop signed artifacts (for example, actor-signed proofs or recipient acknowledgments) SHOULD follow the same `<name>` plus `<name>_complete` claim-pair convention, advertise support with `<name>_supported` in Authorization Server Metadata {{RFC8414}}, and advertise resource-side requirements with `<name>_required` and (if applicable) `<name>_complete_required` in Protected Resource Metadata {{RFC9728}}.  Following this convention lets recipients evaluate independent companions through the same coverage and capability machinery defined here.

Companion profiles add per-hop signal in two distinct patterns, and SHOULD pick the one that matches their signer:

*  **Parallel artifact arrays**: a companion outer-token claim parallel to `actor_receipts`, following the `<name>` plus `<name>_complete` convention above.  Each entry is a separately signed JWT.  Use this pattern when the artifact needs its own signer trust independent of the receipt issuer (for example, actor-signed proofs whose threat model differs from AS-signed receipts, or recipient-signed acknowledgments).
*  **Per-receipt extension claims**: a claim added to each receipt JWT and verified as part of the receipt's signature.  Use this pattern when the assertion is something the receipt issuer is already attesting (for example, per-hop authority bounds, delegation-flow correlation, or lifecycle-state snapshots).  Recognized per the unrecognized-claims rule in {{receipt-claims}}; cross-receipt verification follows {{extensibility}}.

The two patterns serve different threat models and have different completeness semantics: parallel-array companions inherit the claim-pair coverage machinery defined here; per-receipt-claim companions define their own completeness rules over the set of receipts that carry the claim.

# Error Handling {#error-handling}

This section defines how receipt-related processing failures map to OAuth error responses.  Receipt validation extends the underlying OAuth or Transaction Token validation rather than replacing it; failures should be reported through the error-response mechanism applicable to the stage at which validation occurred.

## Authorization Server and Transaction Token Service Errors

When an authorization server or Transaction Token Service rejects a token-exchange request because inbound `actor_receipts` cannot be validated under {{extending-an-existing-receipt-chain}} (signature failure, expired receipt, unsupported `prh_alg`, broken `prh` chain, hop or subject misalignment, or untrusted receipt issuer), it SHOULD return `invalid_grant`, constructed per {{RFC8693}} Section 2.2.2 and {{RFC6749}} Section 5.2, consistent with the core actor profile's error mapping for actor information that fails validation.

When the failure reflects an actor-authorization decision rather than a structural validation failure, an issuer MAY use `actor_unauthorized` as defined in the core actor profile {{I-D.mcguinness-oauth-actor-profile}} where applicable.

## Resource Server Errors

When a resource server rejects a request because `actor_receipts` validation fails under {{consumer-processing}}, it SHOULD return `invalid_token` per the bearer-token error model in {{RFC6750}} Section 3.1.

When the failure is specifically that required receipts are absent or coverage is incomplete (per `actor_receipts_required` or `actor_receipts_complete_required`), the resource server SHOULD include an `error_description` value identifying receipt-coverage failure so that clients and operators can distinguish it from generic token-validation failures.

## Introspection Server Behavior {#introspection-errors}

When an introspection server cannot return receipts that the requesting resource server requires, it returns the introspection response per {{RFC7662}} with `actor_receipts` absent or with `actor_receipts_complete: false`; the resource server then applies its local policy to decide whether to accept the token.

The introspection server itself does not return an OAuth error for missing receipts; receipt presence is a property of the introspection response, not a precondition for it.

Consumer use of introspection-returned receipts is described in {{consumer-introspection}}; the registered introspection response members are defined in {{introspection-response-members}}.

## No New Error Codes

This document does not define new OAuth error codes.  The mapping above reuses existing codes from {{RFC6749}}, {{RFC6750}}, and the core actor profile.

# Extensibility {#extensibility}

This profile is designed to compose with sibling companion profiles that build on the OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}}.  Companion profiles have five standard extension surfaces:

*  **New claims inside a receipt JWT** for additional per-hop attributes (for example, historical scope, additional binding data, or extension-specific provenance).  Consumers ignore unrecognized claims under {{receipt-claims}} unless another specification or local agreement defines their meaning, so additive claims do not break the validation rules of this document.
*  **New top-level claims on the outer token, parallel to `actor_receipts`**, for per-hop artifacts that need their own signature semantics (for example, actor-signed proofs whose threat model differs from AS-signed receipts, or recipient-signed acknowledgments).  Profiles that define such claims SHOULD follow the `<name>` plus `<name>_complete` claim-pair convention described in {{discovery-capability-signaling}}.
*  **New JOSE `typ` values** for receipt-shaped artifacts that are not AS-signed receipts conforming to this document.  The `typ` value `actor-receipt+jwt` defined here is reserved for receipts conforming to this document and MUST NOT be used by other artifacts.
*  **New outer-token binding claims**, analogous to `origin_jti`, that record an outer-token field other than `jti` (for example, a workflow correlation identifier).  Such claims are independently verifiable as current-token bindings only on the same terms as `origin_jti`; see {{receipt-to-token-binding-limits}}.
*  **Non-hop event artifacts** for signal that is not tied to the introduction of a new visible actor hop, including re-authorization events without a hop change, lifecycle-state changes on a governing authority object, receiver acknowledgments, and sender-constraint rotations.  Companion profiles defining such artifacts SHOULD: use a signed JWT with a `typ` value distinct from `actor-receipt+jwt`; carry the events in a top-level outer-token claim parallel to `actor_receipts` (for example, `<companion>_events`); anchor each event either to a specific receipt by carrying that receipt's `jti` in a claim the companion profile defines, or to the delegation flow by a correlation identifier the companion profile specifies (for example, the `txn` claim of {{I-D.ietf-oauth-transaction-tokens}} in Transaction Token deployments); OPTIONALLY chain events among themselves via `prh` / `prh_alg` using the linkage construction in {{receipt-claims}}; OPTIONALLY follow the `<name>` plus `<name>_complete` claim-pair convention from {{discovery-capability-signaling}}.  Companion profiles MUST NOT add event-shaped entries to `actor_receipts`; that array is reserved for per-hop AS-signed attestations defined by this document, and its byte-preservation invariant cannot accommodate entries added after the chain was formed.

Companion profile authoring rules:

*  Companion profiles MAY extend consumer processing under {{consumer-processing}} by adding rejection conditions; they MUST NOT relax any rejection condition defined here.
*  Companion-profile claims and discovery metadata MUST be registered with IANA in the registries used by this document.
*  Companion profiles MAY reuse the `prh` and `prh_alg` chain-linkage construction defined in {{receipt-claims}} when their per-hop signed artifacts form a similar chain structure, so that recipients can apply a single chain-validation routine across companions.
*  Companions whose artifacts do not form a chain (for example, independent per-hop attestations or recipient acknowledgments that are not linked to one another) MAY define their own integrity structure.
*  Companion profiles MAY define cross-receipt verification rules (for example, monotonicity rules over per-hop authority bounds, alignment rules between per-hop attestations, or aggregation rules over per-hop assertions) that compare claims across receipts in the chain.  The chain structure preserved by `prh` and the byte-for-byte preservation requirement make such cross-receipt verification possible.  Companion profiles defining cross-receipt rules MUST tolerate sparse coverage (not every receipt is required to carry the companion's claims) unless they explicitly require completeness.

Cross-companion alignment: companion artifacts that need to reference a specific receipt (for example, an actor-signed proof at hop N referencing the corresponding AS-signed receipt at hop N) SHOULD do so by the receipt's `jti`, which is REQUIRED on receipts and unique within the issuer's namespace.  This profile does not define a hop-index claim; cross-companion alignment is established through `jti` reference plus the `prh` chain's structural integrity, not through array-position metadata.

Conflict resolution: when a recipient implements multiple companion profiles whose rules conflict, local policy determines precedence.  Companion profiles SHOULD be designed to add, not contradict, other profiles' rejection conditions, so that conflicts arise only between profiles whose threat models are genuinely incompatible.

# Security Considerations

Actor receipts strengthen provenance for visible actor hops, but they do not replace ordinary token validation.  The general OAuth 2.0 Security Best Current Practice {{RFC9700}} and the JWT best practices in {{RFC8725}} apply to systems implementing this profile.

## Threat Model {#threat-model}

This section indexes the adversary classes addressed (and not addressed) by this profile.  Detailed mitigations live in {{trust-in-receipt-issuers}}, {{receipt-to-token-binding-limits}}, {{compromised-outer-issuer}}, and {{subject-re-expression-across-hops}}.

### Adversaries Mitigated by This Profile

*  **Compromised downstream issuer fabricating prior-hop provenance.**  Cannot forge prior issuers' receipt signatures; `prh` chain prevents dropping or reordering inner receipts.  Primary value proposition.
*  **Token mutation in transit.**  Each receipt is independently signed; modification invalidates the receipt's signature and any newer receipt's `prh`.
*  **Receipt transplantation between tokens with matching visible `act` chains.**  Outer-token signature prevents non-issuer parties from constructing a substitute outer token to host transplanted receipts.  `receipt[0].origin_jti` provides diagnostic confirmation in the originating-issuance case (see {{receipt-to-token-binding-limits}}); it does not extend the threat model beyond the outer-token-signature defense, since outer-token issuer compromise is out of scope ({{compromised-outer-issuer}}).
*  **Partial-coverage misclaim.**  An issuer cannot drop an inner receipt without breaking the `prh` chain; `actor_receipts_complete: true` cannot be claimed without a count matching visible chain depth.

### Adversaries NOT Mitigated

*  **Compromised current outer token issuer.**  Can assemble a new outer token wrapping previously harvested valid receipts for the same visible chain prefix.  Defense requires external transparency, transaction binding, or replay detection.
*  **Compromised receipt signing key for any one issuer.**  Forged receipts indistinguishable from legitimate ones cannot be revoked individually.  Remediation: remove the compromised issuer from the trusted-issuer set; short receipt `exp` bounds the exposure window.
*  **Compromised actor at a hop.**  Receipts attest issuer assertions, not actor non-repudiation.  Companion profiles ({{extensibility}}) can address this with actor-signed proofs.
*  **Cross-namespace subject graft with a compromised upstream issuer.**  An attacker who compromises one upstream issuer can mint receipts for any subject in that issuer's namespace and graft them onto a re-expressed downstream chain.  Mitigation: consistent `sub` across the chain or trusted out-of-band subject mapping ({{subject-re-expression-across-hops}}).
*  **Replay of an entire token plus its receipts.**  This profile does not define replay detection; receipts inherit the outer token's replay characteristics.

### Trust Model Summary

Trust is per-issuer and per-deployment, and not transitive across the chain.  A receipt chain breaks at the first inner receipt whose issuer is not trusted, even when the outer token's issuer and earlier receipts are trusted.  Companion profiles ({{extensibility}}) can extend the addressed adversary set; for example, an actor-signed-proofs companion can mitigate the compromised-current-outer-token-issuer adversary.

## Current Presenter Validation

The current request is always validated against the outer token's top-level `cnf` ({{RFC7800}}), when present, using the proof mechanism appropriate to the token type and deployment, such as DPoP {{RFC9449}} or mutual-TLS {{RFC8705}}.

Receipt `cnf` values are historical only:

*  A recipient MUST NOT treat an older receipt `cnf` value as sufficient proof for the current request, regardless of which proof mechanism the historical `cnf` was bound to.
*  Recipients MUST distinguish receipt JWTs (identified by `typ` value `actor-receipt+jwt`) from outer tokens that carry `cnf` for current-request proof-of-possession; receipt `cnf` records historical binding and never satisfies a current-request PoP requirement under {{RFC7800}}, {{RFC9449}}, or {{RFC8705}}.

The current top-level `cnf` can differ from the outermost receipt `cnf` after a later reissuance or key rotation that does not add a new actor hop.  That difference does not by itself invalidate the receipt chain under this profile.

## Trust in Receipt Issuers {#trust-in-receipt-issuers}

Receipt validation is meaningful only if the recipient trusts the issuers that signed the receipts.

Trust establishment requirements:

*  A recipient MUST establish which issuers it trusts for receipt validation before relying on `actor_receipts`.
*  Trust MUST be established through explicit pre-configuration, bilateral agreement, federation policy, or another explicit trust framework.
*  A recipient MUST NOT treat the presence of a syntactically valid signed receipt as sufficient grounds to trust its issuer.
*  Authorization servers that support this document SHOULD advertise `actor_receipts_supported: true` in their AS metadata {{RFC8414}}.
*  Consumers SHOULD use that metadata signal as one input to trust establishment, but MUST NOT treat metadata advertisement alone as sufficient grounds to trust a receipt issuer; the issuer must also be within the recipient's configured trust boundary.

Key resolution requirements:

*  To avoid attacker-controlled key resolution, a recipient MUST determine whether a receipt `iss` is within its trusted-issuer set before performing any network retrieval for that issuer's metadata or keys.
*  A recipient that uses dynamic discovery for receipt validation MUST do so only within an existing trust framework or equivalent local policy that defines which issuers are permitted.

Trust is per-issuer and not transitive: each receipt is validated against the recipient's own trusted-issuer set, independent of the outer token's issuer or neighboring receipts.  If any receipt in the presented `actor_receipts` array is signed by an issuer that is not trusted for receipt validation, the recipient MUST reject the receipt chain for the purposes of this profile.  This document does not define trusted-prefix validation across an untrusted inner receipt.  Deployments needing uniform trust across an extended chain MUST establish trust explicitly with every receipt issuer that may appear in tokens they accept.

The trust evaluation in this section covers receipt signers (the `iss` claim of each receipt).  Recipients separately evaluate trust in each receipt's `act.iss` as the namespace authority for `act.sub` as described in {{receipt-claims}}; that evaluation is independent of receipt-signer trust, even when the same entity holds both roles.

## Receipt-to-Token Binding Limits {#receipt-to-token-binding-limits}

Receipts prove that trusted issuers attested particular actor hops and, optionally, historical presenter bindings.  They do not prove that the current outer token's audience, scope, expiration, or other authorization details were in force when older receipts were created.  A recipient MUST NOT treat a valid receipt chain as evidence of historical authorization scope or audience beyond what the current outer token authorizes.

In the originating-issuance case, receipt-chain integrity rests on two anchors when the outer token carries `jti` and `receipt[0].origin_jti` is present:

*  `receipt[0].origin_jti`, when present, signed by the same issuer that signed the outer token, and equal to the outer token's `jti`, binds `receipt[0]` to the specific outer-token instance and prevents transplantation from a different token whose visible `act` structure happens to match.
*  `prh` chains each receipt cryptographically to its older neighbor, so all inner receipts inherit the originating-issuance binding from `receipt[0]` through the hash chain.

Inner receipts' `origin_jti` values are not independently verifiable (see {{consumer-processing}}); deployments depending on current-token-instance binding MUST rely on `prh` plus a verifiable `receipt[0].origin_jti` in the originating-issuance case, not on inner `origin_jti` values.  If `receipt[0].origin_jti` is absent, the chain can still provide issuer-signed hop provenance, but it does not provide the current-token-instance binding property.  An inner receipt has no independent binding to the current request, audience, scope, or token instance; it is bound only to its older neighbor through `prh` and ultimately to the current outer token through the chain's single anchor when that anchor is verifiable.

This construction makes coverage tamper-evident at the structural level:

*  An issuer cannot drop an inner receipt without breaking the `prh` chain: the next-newer receipt's `prh` value would no longer match the receipt now in the next array position, and consumer step 6 of {{consumer-processing}} rejects the chain.
*  An issuer can withhold coverage only from the innermost (oldest) end of the chain, and only by beginning a new chain under {{creating-the-first-receipt}} rather than trimming an inherited one; a trimmed chain leaves the surviving oldest receipt carrying a `prh` with no target, which step 6 also rejects.  The result is partial coverage that `actor_receipts_complete: true` then forbids the issuer from claiming.

Coverage is therefore truthful within the limits of the trusted-issuer set: a compromised issuer can omit some or all of its own receipts and any outermost receipts from issuers it controls, but it cannot fabricate, reorder, or selectively drop receipts signed by other trusted issuers.

When `receipt[0]` diverges from the current outer-token instance (either `receipt[0].iss` differs from `outer.iss`, or `receipt[0].iss` matches but `receipt[0].origin_jti` differs from `outer.jti`), `receipt[0].origin_jti` becomes historical provenance only and does not bind the receipt chain to the current outer-token instance.  Per step 5 of {{consumer-processing}} and {{strict-mode-validation}}, consumers reject divergent chains by default and accept them only when local policy designates the current outer token issuer (`outer.iss`) as a trusted reissuing issuer for chains whose leading receipt issuer is `receipt[0].iss`.

This profile provides no in-band signal distinguishing legitimate reissuance from a re-wrapping attack by a compromised issuer.  Recipients that opt into accepting reissued tokens MUST establish that policy through local configuration or an out-of-band trust framework, and SHOULD treat unexpected reissuance divergence as cause for additional scrutiny.

The `origin_jti` strict-equality check therefore provides binding only in the originating-issuance case (same-issuer, same-`jti`).  In all other cases, transplantation defense rests on the outer token's signature: a non-issuer party cannot construct a substitute outer token to host transplanted receipts.  Compromise of the outer-token issuer falls under {{compromised-outer-issuer}} and is out of scope.

Companion profiles MAY define additional outer-token binding claims following the `origin_jti` pattern: each records an identifier from the outer token at receipt creation, with consumer verifiability conditioned on issuer alignment and equality with the current outer-token field.  Such claims provide parallel anchors against other outer-token fields and do not weaken the `origin_jti` anchor.

### Strict-Mode Validation {#strict-mode-validation}

Recipients that have not explicitly configured a set of trusted reissuing issuers operate in *strict-mode validation* by default: per step 5 of {{consumer-processing}}, any divergence between `receipt[0]` and the current outer-token instance (whether by differing `iss` or differing `origin_jti`) causes the recipient to reject the receipt chain.  When the outer token carries `jti`, strict-mode recipients that require current-token-instance binding also reject same-issuer chains whose leading receipt omits `origin_jti`.

Strict mode is the conservative default and the recommended posture absent specific deployment requirements.  Deployments that need to accept reissued tokens (refresh-token reissuance, introspection-and-re-emission, or token translation across trust principals) MUST configure permissive validation with an explicit set of trusted reissuing issuers, established through local policy or out-of-band trust framework.  Without such configuration, receipts on reissued tokens will be rejected.

## Hash Algorithm Agility

`prh` defaults to a base64url-encoded `sha-256` hash of the next older receipt.  The `prh_alg` claim ({{receipt-claims}}) signals an alternative hash algorithm by reference to the IANA Named Information Hash Algorithm Registry {{RFC6920}}, without requiring a successor specification.

Algorithm coordination requirements:

*  All receipts in a single chain MUST use the same algorithm.
*  Consumers MUST reject chains that mix algorithms or that name an algorithm the recipient does not support.
*  An issuer extending an inbound chain MUST preserve the inbound `prh_alg`.

Migration is whole-chain, not partial: chains begun under one algorithm remain on that algorithm for their lifetime; new chains can adopt a different algorithm independently.  This profile does not define rehashing of inbound receipts, because rehashing would invalidate prior signers' `prh` values and require re-signing receipts the extending issuer did not originate.

Deployments SHOULD begin issuing new chains under the target algorithm well before any indication that the legacy algorithm is reaching end of life, so legacy chains expire naturally.

## Compromised Outer Issuer {#compromised-outer-issuer}

Receipts mitigate a compromised or dishonest *downstream* issuer attempting to fabricate prior-hop provenance: that issuer cannot forge prior issuers' receipt signatures.

A compromised current outer token issuer is a different threat.  Such an issuer can assemble a new outer token wrapping previously harvested valid receipts for the same visible chain prefix.  This document does not solve that class of attack; deployments needing stronger guarantees can combine this profile with transparency, transaction binding, or replay-detection mechanisms outside the scope of this document.

## Receipt Freshness and Replay {#receipt-freshness}

Receipts are historical attestations of past delegation state.  They MAY outlive the validity period of the outer token they were originally issued for, and MAY be carried forward across reissuance and refresh as long as their `exp` permits ({{receipt-claims}}).

This profile takes a deliberately limited view of receipt freshness:

*  Receipts attest hop history at the time of original issuance; they do not assert that the represented delegation is still active.
*  Runtime policy evaluation, including current authorization, current scope, and current revocation state, is separate from receipt validation.  A valid receipt does not guarantee that the represented delegation has not been revoked since.
*  Receipt `exp` bounds the window during which the receipt is structurally usable; it does not bound the lifetime of the underlying delegation.
*  Replay of a receipt within its `exp` window is not in itself an attack.  Receipts are designed to be carried by tokens that themselves have replay-detection or sender-constraint properties; replay of an entire token plus its receipts is governed by the outer token's replay characteristics, not by the receipt chain.

Deployments needing freshness signals beyond receipt `exp`, such as active delegation status, fresh authorization confirmation, or current revocation state, MUST obtain those signals from the AS via introspection ({{RFC7662}}), fresh token issuance, or another mechanism outside the scope of this profile.

## Receipt Signing Key Compromise

If a receipt issuer's signing key is compromised, previously issued receipts signed with that key cannot be individually revoked.  The primary remediation is to remove the compromised issuer from the trusted-issuer set; once removed, consumers will reject all receipts signed by that issuer regardless of their content.

Deployments SHOULD set short `exp` values on receipts, consistent with the REQUIRED `exp` defined in {{receipt-claims}}, to limit the window during which receipts signed with a compromised key remain valid.  When a key compromise is detected, deployments SHOULD treat all tokens carrying receipts from the affected issuer as lacking trusted provenance for those hops and SHOULD require re-issuance through a trusted issuer.

## Receipt Chain Size

Each receipt is a full signed JWT, and the chain grows linearly with delegation depth.  A typical signed receipt is 400 to 800 bytes after JWS compact serialization and base64url encoding (the upper end when `cnf` or larger `act` objects are present).  Chains beyond approximately 10 hops therefore approach the 8 KB Authorization header budget common in HTTP infrastructure; chains beyond approximately 20 hops approach a 16 KB practical ceiling.  Figures are illustrative and depend on the deployment.

Deployments SHOULD verify that the outer token plus its `actor_receipts` array fits within the header-size budget of every component on the request path.  When introspection is available, deployments MAY return receipts via introspection rather than embedding them, to avoid header pressure for bearer-token clients.

## Historical `cnf` Disclosure {#historical-cnf-disclosure}

Receipt `cnf` values reveal prior-hop public-key identifiers or certificate thumbprints to any party that receives the token or introspection response.  These are stable identifiers that enable cross-request and cross-service correlation of actors and services over time.  Issuers SHOULD NOT include `cnf` in receipts unless the relying parties that will receive the token have been evaluated for that disclosure risk and the risk is acceptable.  Omitting `cnf` does not invalidate the receipt; it means that hop lacks independently attested historical presenter binding, which is acceptable for many deployments.

# Privacy Considerations

Actor receipts materially increase delegation transparency and observability.  This is useful for audit and provenance, but it is privacy-sensitive: receipts disclose information that a non-receipt-bearing token does not, and the disclosure can be hard to retract once the token has reached recipients.

## What Receipts Disclose

Receipts can expose, to any party that receives the token or introspection response:

*  the set of issuers that participated in the delegation chain (receipt `iss` values);
*  historical presenter-key identifiers across requests, enabling cross-session correlation of actors and services;
*  internal service identities and intermediary actors that a deployment might otherwise have kept visible only to intermediate issuers;
*  workload identifiers (e.g., `act.sub` values) that may reveal organizational structure or orchestration topology;
*  subject re-expression patterns across namespaces, which can reveal cross-domain identity mappings.

## Minimization

Deployments SHOULD minimize receipt disclosure when full provenance is not required:

*  Issuers and introspection servers MAY suppress `actor_receipts` entirely when policy does not permit disclosure.
*  Introspection servers returning a stored partial-coverage chain SHOULD set `actor_receipts_complete` to `false`; disclosure of a stored chain is otherwise all-or-nothing (see {{consumer-introspection}}).
*  Resource servers SHOULD request or require actor receipts only when they materially improve authorization, audit, or risk controls.
*  Issuers SHOULD omit `cnf` from receipts by default when relying parties have not been evaluated for historical presenter-key disclosure risk (see {{historical-cnf-disclosure}}).
*  Deployments SHOULD prefer per-resource-server policy on receipt requirements over blanket inclusion in every token.

## Selective Disclosure

This profile does not define a per-claim selective-disclosure mechanism for receipts: chain integrity requires byte-for-byte preservation of each receipt JWT, so selective omission of individual claims within a receipt would break the chain.  Selective disclosure is therefore coarse-grained:

*  Issuers MAY emit partial-coverage chains that cover only the outermost hops (see {{partial-coverage-and-full-coverage}}); this is the only mechanism for omitting individual hops, and it operates at issuance time.
*  Issuers and introspection servers MAY withhold the `actor_receipts` array entirely; a strict subset of an existing array cannot validate under {{consumer-processing}} (see {{consumer-introspection}}).

Deployments needing finer-grained selective disclosure require a future companion profile.  Such a companion must alter the chain-linkage construction (for example, by linking against a stable hash that survives claim redaction); a companion that only adds a selective-disclosure claim cannot achieve per-claim disclosure within the current `prh` construction.

## Audience Restriction

A receipt travels with the outer token to whichever audiences the outer token serves; receipts have no independent audience scoping ({{receipt-claims}}).  Deployments needing audience-specific disclosure constraints SHOULD partition receipt issuance by audience at issuance time (for example, issue receipt-bearing tokens only to audiences with adequate disclosure agreements) rather than relying on receipt-level audience restriction, which this profile does not provide.

## Unnecessary Hop Disclosure

Receipts expose every hop the issuer chose to include.  Some hops may be deployment-internal (orchestration layers, internal workload-identity services) that the deployment would not otherwise expose to relying parties.  Issuers SHOULD evaluate, at issuance time, which hops are appropriate to expose to which audiences.  Where inner hops are not appropriate to expose, issuers SHOULD use partial coverage (omitting the inner-hop receipts) rather than fabricating, suppressing, or rewriting visible `act` chain entries; the latter would violate the core actor profile.

## Cross-Service Correlation

Stable identifiers in receipts (`iss`, `act.sub`, `cnf`, and any companion correlation claim such as a delegation-flow identifier) enable cross-service correlation of actors, subjects, and workflows over time.  Deployments operating in privacy-sensitive contexts SHOULD evaluate the correlation risk before enabling receipts:

*  An audit pipeline that aggregates receipts across services builds a graph of who delegated to whom and when, across organizational boundaries.
*  Receipts from a single workflow are tied together via `prh` chain hashes, exposing the delegation graph even when individual hops are routed through privacy-preserving infrastructure.
*  Receipts persist longer than the outer tokens they were issued for and may be retained in audit logs indefinitely; correlation risk is not bounded by token lifetime.

Companion profiles defining per-receipt extension claims (per the patterns in {{discovery-capability-signaling}}) may introduce additional stable identifiers or correlation surfaces beyond those listed above.  Examples include per-hop authority-bounds content (which exposes per-hop scope, audience, and resource detail), references to a governing authority object, and lifecycle-state snapshots.  The privacy considerations of this section (cross-service correlation, retention beyond token lifetime, detached-verification disclosure) apply to such companion claims wherever they share these properties.  Companion profiles defining extension claims SHOULD evaluate the correlation and disclosure characteristics specific to their claims and document any deployment guidance their claims warrant.

## Detached Verification Privacy

Receipts support detached verification (validation by parties that do not have access to the issuing AS's introspection endpoint).  This is a feature for auditability but a risk for privacy: any party with the token plus the receipt issuer's published verification keys can validate the chain, including parties the issuer did not anticipate.  Deployments SHOULD treat receipt-bearing tokens as carrying their full provenance to anywhere the token reaches, and SHOULD scope token distribution accordingly.

# IANA Considerations

## Media Type Registration

This document requests registration of the following media type in the "Media Types" registry {{RFC6838}}:

*  Type name: `application`
*  Subtype name: `actor-receipt+jwt`
*  Required parameters: N/A
*  Optional parameters: N/A
*  Encoding considerations: 8bit; an actor receipt is a JWS compact-serialized JWT {{RFC7515}} {{RFC7519}} consisting of base64url-encoded segments separated by period (`.`) characters.
*  Security considerations: See {{security-considerations}} of this document and {{RFC8725}}.
*  Interoperability considerations: N/A
*  Published specification: This document
*  Applications that use this media type: Applications that issue, exchange, or validate OAuth Actor Receipts.
*  Fragment identifier considerations: N/A
*  Additional information:
   *  Deprecated alias names for this type: N/A
   *  Magic number(s): N/A
   *  File extension(s): N/A
   *  Macintosh file type code(s): N/A
*  Person & email address to contact for further information: Karl McGuinness, public@karlmcguinness.com
*  Intended usage: COMMON
*  Restrictions on usage: None
*  Author: Karl McGuinness, public@karlmcguinness.com
*  Change controller: IETF

The JOSE `typ` value `actor-receipt+jwt` used by this document is the media type subtype name without the `application/` prefix, following common JWT typing practice.

## JSON Web Token Claims Registration

This document requests registration of the following JWT Claims in the "JSON Web Token Claims" registry {{RFC7519}}:

*  Claim Name: `actor_receipts`
*  Claim Description: Array of signed actor-hop receipts providing delegation provenance
*  Change Controller: IESG
*  Specification Document(s): This document

*  Claim Name: `actor_receipts_complete`
*  Claim Description: Boolean indicating whether actor_receipts covers every visible hop in the token's act chain
*  Change Controller: IESG
*  Specification Document(s): This document

*  Claim Name: `sub_iss`
*  Claim Description: Issuer or namespace authority for the subject in an Actor Receipt JWT
*  Change Controller: IESG
*  Specification Document(s): This document

*  Claim Name: `prh`
*  Claim Description: Base64url-encoded hash of the immediately preceding (older) receipt in an Actor Receipt JWT chain
*  Change Controller: IESG
*  Specification Document(s): This document

*  Claim Name: `prh_alg`
*  Claim Description: Hash algorithm identifier (from the IANA Named Information Hash Algorithm Registry) naming the algorithm used to compute prh in an Actor Receipt JWT
*  Change Controller: IESG
*  Specification Document(s): This document

*  Claim Name: `origin_jti`
*  Claim Description: The jti of the outer token at the time an Actor Receipt JWT was created (the receipt's origin outer token)
*  Change Controller: IESG
*  Specification Document(s): This document

## OAuth Authorization Server Metadata Registration

This document requests registration of the following metadata name in the "OAuth Authorization Server Metadata" registry {{RFC8414}}:

*  Metadata Name: `actor_receipts_supported`
*  Metadata Description: Indicates support for validating, originating, preserving, or extending actor-receipt chains
*  Change Controller: IESG
*  Specification Document(s): This document

## OAuth Protected Resource Metadata Registration

This document requests registration of the following metadata names in the "OAuth Protected Resource Metadata" registry {{RFC9728}}:

*  Metadata Name: `actor_receipts_required`
*  Metadata Description: Indicates that the resource expects delegated requests to carry valid actor receipts covering at minimum the outermost visible actor hop
*  Change Controller: IESG
*  Specification Document(s): This document

*  Metadata Name: `actor_receipts_complete_required`
*  Metadata Description: Indicates that the resource requires complete receipt coverage for all visible actor hops
*  Change Controller: IESG
*  Specification Document(s): This document

## OAuth Token Introspection Response Registration

This document requests registration of the following names in the "OAuth Token Introspection Response" registry {{RFC7662}}:

*  Name: `actor_receipts`
*  Description: Array of signed actor-hop receipts returned by introspection
*  Change Controller: IESG
*  Specification Document(s): This document

*  Name: `actor_receipts_complete`
*  Description: Indicates whether the returned actor receipts provide complete visible-hop coverage
*  Change Controller: IESG
*  Specification Document(s): This document

# Acknowledgments

This document builds on the OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}}, on the OAuth 2.0 Token Exchange specification {{RFC8693}}, on the OAuth 2.0 Transaction Tokens work {{I-D.ietf-oauth-transaction-tokens}}, and on prior OAuth Working Group discussion of delegation transparency, sender-constrained tokens, and proof-of-possession mechanisms ({{RFC7800}}, {{RFC8705}}, {{RFC9449}}).  The author thanks the working group for that foundation.

Individual contributors and reviewers will be acknowledged in subsequent revisions of this document as feedback accumulates.

--- back

# Examples

The examples in this appendix show decoded receipt contents.  Real receipts are compact-signed JWT strings carried in the `actor_receipts` array.  The `iat` and `exp` values shown are illustrative only; in deployments, receipt `exp` is set per {{receipt-claims}} and {{extending-an-existing-receipt-chain}} so that no inbound receipt expires before the outer token that carries it.

Both examples below illustrate the explicit-disclosure pattern: receipts include `cnf` to demonstrate historical sender-constraint provenance.  Per {{receipt-claims}}, issuers SHOULD omit `cnf` from receipts unless the relying parties that will receive the token have been evaluated for the associated disclosure risk.  Privacy-conservative deployments produce receipts that are structurally identical to those shown but with the `cnf` claim omitted.

## Example: Two-Hop Delegation Chain

The outer token carries the following visible actor chain:

~~~json
{
  "jti": "d3a1b2c0-9f4e-4a1d-b8e7-12345678abcd",
  "iss": "https://as.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://tools.example.com/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service",
    "act": {
      "sub": "https://agents.example.com/travel-assistant",
      "iss": "https://as.enterprise.example",
      "sub_profile": "ai_agent"
    }
  },
  "cnf": {
    "jkt": "ToolJKT"
  },
  "actor_receipts": [
    "<receipt-0>",
    "<receipt-1>"
  ],
  "actor_receipts_complete": true
}
~~~

`actor_receipts[0]` is the newest receipt, created by the travel-provider AS when it added the booking tool as the new outermost actor:

~~~json
{
  "iss": "https://as.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://tools.example.com/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service"
  },
  "cnf": {
    "jkt": "ToolJKT"
  },
  "prh": "0QvKZr5A4XW7N9LQW0u4e7z8k2Kqz6I7xL4V4Vh2nRc",
  "iat": 1776745200,
  "exp": 1776832000,
  "jti": "c8e29c11-0c3a-4e6f-a0a6-30a52c4a8149",
  "origin_jti": "d3a1b2c0-9f4e-4a1d-b8e7-12345678abcd"
}
~~~

`actor_receipts[1]` is the older receipt, created by the enterprise AS when it first added the AI agent:

~~~json
{
  "iss": "https://as.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_iss": "https://idp.enterprise.example",
  "act": {
    "sub": "https://agents.example.com/travel-assistant",
    "iss": "https://as.enterprise.example",
    "sub_profile": "ai_agent"
  },
  "cnf": {
    "jkt": "AgentJKT"
  },
  "iat": 1776741600,
  "exp": 1776832000,
  "jti": "1d4c4d30-fb6d-4172-b7eb-775b6b9c2b85",
  "origin_jti": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
~~~

This example shows the key provenance property of this profile: the current token is bound to `ToolJKT`, while the older receipt preserves that the earlier actor hop was bound to `AgentJKT` when it was created.  The `sub_iss` claim on `receipt[1]` records that the subject identifier `https://idp.enterprise.example/users/alice` is interpreted under the enterprise IdP's namespace authority, distinct from the receipt's signer (`https://as.enterprise.example`, the enterprise AS).

## Example: Transaction Token Service Rebinding

Suppose the booking tool exchanges the access token above at a TTS, and the TTS rebinds the issued Transaction Token to an internal workload identified as `https://wimse.travel-provider.example/payments`.

The resulting Transaction Token can carry:

~~~json
{
  "jti": "f0e1d2c3-b4a5-6789-cdef-012345678901",
  "iss": "https://tts.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://wimse.travel-provider.example/payments",
    "iss": "https://tts.travel-provider.example",
    "sub_profile": "service",
    "act": {
      "sub": "https://tools.example.com/booking-tool",
      "iss": "https://as.travel-provider.example",
      "sub_profile": "service",
      "act": {
        "sub": "https://agents.example.com/travel-assistant",
        "iss": "https://as.enterprise.example",
        "sub_profile": "ai_agent"
      }
    }
  },
  "cnf": {
    "jkt": "PaymentsJKT"
  },
  "actor_receipts": [
    "<receipt-tts>",
    "<receipt-0>",
    "<receipt-1>"
  ],
  "actor_receipts_complete": true
}
~~~

The new leading receipt created by the TTS is:

~~~json
{
  "iss": "https://tts.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://wimse.travel-provider.example/payments",
    "iss": "https://tts.travel-provider.example",
    "sub_profile": "service"
  },
  "cnf": {
    "jkt": "PaymentsJKT"
  },
  "prh": "C4zv2FK0kPjxzJz8F7G3mslmbb0TQmVQvls0gA1lV3Q",
  "iat": 1776747000,
  "exp": 1776832000,
  "jti": "8b1ab6d1-c345-4bd3-8af2-f302d54444b7",
  "origin_jti": "f0e1d2c3-b4a5-6789-cdef-012345678901"
}
~~~

The inherited receipts for the booking tool and the AI agent are carried forward unchanged.

## Example: Partial Receipt Coverage

When receipt support is rolled out progressively across issuers, downstream tokens may carry coverage for only the outermost hops.  Suppose the enterprise AS has not yet deployed receipt support, and the travel-provider AS has.  The enterprise AS issues a delegated token introducing the AI agent without a receipt.  The travel-provider AS exchanges that token, adds the booking tool as the new outermost actor, and creates a single receipt for that hop.

The resulting access token carries:

~~~json
{
  "jti": "b6d94f2a-3c81-47e5-9a0d-5f6e7a8b9c0d",
  "iss": "https://as.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://tools.example.com/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service",
    "act": {
      "sub": "https://agents.example.com/travel-assistant",
      "iss": "https://as.enterprise.example",
      "sub_profile": "ai_agent"
    }
  },
  "cnf": {
    "jkt": "ToolJKT"
  },
  "actor_receipts": [
    "<receipt-0>"
  ],
  "actor_receipts_complete": false
}
~~~

The single receipt covers the outermost hop:

~~~json
{
  "iss": "https://as.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://tools.example.com/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service"
  },
  "cnf": {
    "jkt": "ToolJKT"
  },
  "iat": 1776745200,
  "exp": 1776832000,
  "jti": "9b7a4e30-2c1f-4d8a-9b5e-f0e8a3c4b6d2",
  "origin_jti": "b6d94f2a-3c81-47e5-9a0d-5f6e7a8b9c0d"
}
~~~

`prh` is omitted because this is a single-element chain.  `actor_receipts_complete: false` signals to recipients that the inner AI-agent hop is uncovered.  Resource servers that set `actor_receipts_complete_required: true` in their Protected Resource Metadata reject this token; resource servers that accept partial coverage validate the receipt-attested outermost hop and treat the inner hop as carried solely by the visible `act` chain, with no independent receipt-level provenance.

## Example: Reissuance Without a New Actor Hop

Suppose the access token from the Two-Hop Delegation Chain example is introspected by an introspection endpoint operated as a separate trust principal from the originating travel-provider AS, and re-emitted as a JWT for an internal service.  Re-emission does not add a new outermost actor hop; the visible `act` chain is unchanged.  Per {{reissuance-without-a-new-actor-hop}}, the re-emitting issuer carries the inbound `actor_receipts` array forward unchanged and does not create a new receipt.

The re-emitted token's claims:

~~~json
{
  "jti": "f4a7b9c2-1d3e-4f5a-8b6c-7d8e9f0a1b2c",
  "iss": "https://introspection.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://tools.example.com/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service",
    "act": {
      "sub": "https://agents.example.com/travel-assistant",
      "iss": "https://as.enterprise.example",
      "sub_profile": "ai_agent"
    }
  },
  "cnf": {
    "jkt": "ToolJKT"
  },
  "actor_receipts": [
    "<receipt-0>",
    "<receipt-1>"
  ],
  "actor_receipts_complete": true
}
~~~

The receipts are bit-identical to those in the Two-Hop Delegation Chain example.  Two divergences from the originating-issuance pattern are visible at the outer-token level:

*  `outer.iss` is `https://introspection.travel-provider.example`, while `receipt[0].iss` remains `https://as.travel-provider.example`.  This divergence is legitimate under {{reissuance-without-a-new-actor-hop}}.
*  `outer.jti` is `f4a7b9c2-1d3e-4f5a-8b6c-7d8e9f0a1b2c`, while `receipt[0].origin_jti` remains `d3a1b2c0-9f4e-4a1d-b8e7-12345678abcd` (the original outer token's `jti`).  This divergence is also legitimate.

Per consumer step 5 of {{consumer-processing}}, the consumer treats `receipt[0].origin_jti` as historical provenance rather than a current-token binding because `receipt[0]` diverges from the current outer-token instance.  This example illustrates the different-issuer pattern (`receipt[0].iss` differs from `outer.iss`); the same treatment applies to the same-issuer pattern, in which an authorization server refreshes its own token and the new outer-token `jti` differs from `receipt[0].origin_jti` while `iss` matches.  In both patterns, trust that the divergence reflects legitimate reissuance rather than a re-wrapping attack rests on the recipient's local policy regarding which issuers in its trust set perform reissuance under this profile, as described in {{receipt-to-token-binding-limits}}.

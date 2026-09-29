---
title: "OAuth Actor Receipts for Delegation Provenance"
abbrev: "OAuth Actor Receipts"
category: std
docname: draft-mcguinness-oauth-actor-receipts-latest
submissiontype: IETF
number:
date: 2026-07-04
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

This document defines OAuth Actor Receipts, an optional companion to the OAuth Actor Profile for Delegation.  Each receipt is a signed JSON Web Token (JWT) attesting which issuer added an actor hop and, optionally, its historical presenter binding.  The `actor_receipts` claim carries a hash-linked chain of receipts that recipients verify against each hop's issuer.  This document specifies receipt processing, metadata, and introspection parameters.

--- middle

# Introduction

The OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}} makes actor identity visible in delegated tokens through a common `act` claim.  A relying party can read that chain but cannot, on the basis of the outer token alone, verify prior hops independently of the current token issuer.

Each issuer adding a covered actor hop signs a receipt.  Receipts travel with the token in `actor_receipts`, linked by hashes and verifiable against the individual issuers' keys.  The visible actor chain remains in `act`; active presenter binding remains in top-level `cnf`.  Receipt `cnf`, when disclosed, records historical binding only.

Deployments enable receipts per resource or trust domain without changing client request flows.  [Design Goals and Non-Goals](#design-goals-and-non-goals) defines the scope.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses OAuth terminology from {{RFC6749}} and {{RFC8693}}, and Transaction Token terminology from {{I-D.ietf-oauth-transaction-tokens}}.  AS, RS, and TTS denote authorization server, resource server, and Transaction Token Service.

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

This profile adds signed receipts, processing rules, metadata, and introspection parameters to the core actor representation.  The underlying Token Exchange and Transaction Token request semantics continue to apply.

## Relationship to Token Introspection

Introspection {{RFC7662}} returns the AS's current view of token status and claims.  Receipts preserve signed hop history that can be verified without access to an introspection endpoint, including after intermediate systems become unavailable.

A deployment may use either, both, or neither:

*  Receipts MAY be returned via introspection, providing both signals in a single response (see {{consumer-introspection}}).
*  Receipts MAY be carried inline in JWT tokens for deployments where introspection is not available or where detached verification is required.
*  Introspection MAY be used for active-status checks even when receipts are not in use.

The choice depends on trust, availability, and audit requirements.

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
*  defining a workflow or delegation-flow correlation identifier across tokens; cross-token correlation uses receipt `jti` and `origin_jti` values together with deployment audit infrastructure;
*  defining transparency logs, non-repudiation systems, or public audit infrastructure.

## Deployment Fit

Receipts are most useful when several issuers contribute hops: recipients can verify each attestation independently of the current outer issuer.  In a single-issuer deployment, the outer signature already supplies that issuer's attestation, so deployments SHOULD weigh receipt overhead against that limited benefit.

Recipients must configure trust for every receipt issuer ({{trust-in-receipt-issuers}}).  Large deployments therefore need a trust-distribution mechanism, such as federation, outside this profile.

Receipt-bearing refresh requires issuer-controlled storage of the receipt array across refresh.  This can be durable server state or a self-contained refresh token; see {{reissuance-without-a-new-actor-hop}}.

Receipts mitigate a compromised or dishonest *downstream* issuer fabricating prior-hop provenance (see {{trust-in-receipt-issuers}} and {{compromised-outer-issuer}}).  They do not mitigate a compromised current outer token issuer and do not provide actor non-repudiation; those properties require mechanisms outside this document.

# Actor Receipts Overview

An actor receipt records one actor hop.  The issuer that adds a new outermost actor hop signs a receipt describing that hop and, when the issued token is sender-constrained, may copy the token's top-level `cnf` value into the receipt as historical presenter-binding context subject to the disclosure considerations in {{receipt-claims}}.

The `actor_receipts` array is ordered newest first and preserves older entries unchanged.  Index 0 corresponds to outermost `act`, index 1 to `act.act`, and so on.  Coverage is either complete or a contiguous outermost prefix; local policy or resource requirements determine whether partial coverage is acceptable.

A valid receipt chain records issuer attestations of past delegation.  It conveys no authority and does not establish that delegation remains active.  Current authorization decisions MUST evaluate the current outer token, current policy, and current state, not the receipt chain alone.

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

  Issuers SHOULD set `actor_receipts_complete` to `true` for complete coverage and `false` for partial coverage.  An absent value provides no completeness attestation; consumers requiring the literal value `true` treat absence like `false`.

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

  Receipt `iss` identifies the signer, not the subject namespace.  Interpret `sub` in the namespace of the represented hop, which MAY differ across receipts.  Reconciliation uses trusted local mappings; see {{subject-re-expression-across-hops}}.

`sub_iss`:
: OPTIONAL.  The namespace authority under which the receipt `sub` value is interpreted.

  The (`sub_iss`, `sub`) pair identifies the subject as (`act.iss`, `act.sub`) identifies the actor.  If `sub_iss` is absent, recipients MUST determine the namespace, when needed, from trusted local context for the represented hop.  This flat pair records the token's issued claims; it does not use the structured Subject Identifier formats in {{RFC9493}}.

  Neither this profile nor the core profile defines an outer-token subject-namespace claim.  To compare with `receipt[0].sub_iss`, recipients obtain that namespace from trusted local context, an inbound subject token, or another deployment-defined source.

`sub_profile`:
: OPTIONAL.  The top-level `sub_profile` value, when the token issued at this hop carried one.

`act`:
: REQUIRED.  A single-hop actor object.  This object:

  *  MUST conform to the core actor profile's actor-object rules;
  *  MUST include `act.sub` and `act.iss`;
  *  MUST NOT contain `cnf`;
  *  MUST NOT contain a nested `act`.

  Historical binding belongs in receipt-level `cnf`.  Other core-profile actor extensions MAY appear unless prohibited here; a receipt containing `act.cnf` is invalid.

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
  *  All receipts in an array MUST carry the same `prh_alg` value or all omit it.  Mixing omission with explicit `sha-256` is invalid even though both select SHA-256.  A single-element chain MAY carry `prh_alg`, but the value has no effect unless a later receipt links to it.
  *  An issuer extending an inbound chain MUST either preserve the inbound `prh_alg` or reject the chain.

### Time and Uniqueness

`iat`:
: REQUIRED.  The time at which the receipt was created, as defined in {{RFC7519}}.

`exp`:
: REQUIRED.  Expiration time for the receipt, as defined in {{RFC7519}}.

  `exp` needs to cover the lifetime of any token that will carry or inherit this receipt; otherwise consumers reject older receipts in a valid chain prematurely.

  Downstream issuers reject receipts that expire before the issued outer token.  Deployments typically coordinate a bounded delegated-session lifetime to avoid propagation failure while limiting signing-key exposure; see {{reissuance-without-a-new-actor-hop}}.

`jti`:
: REQUIRED.  A unique identifier for the receipt, as defined in {{RFC7519}}.

### Outer-Token Binding

`origin_jti`:
: RECOMMENDED.  The `jti` of the outer token at the time this receipt was created (the receipt's origin outer token).  This value is fixed at receipt creation; after reissuance the current outer token's `jti` may differ.

  It identifies the originating token.  It binds to the current token only when both `receipt[0].iss` and `receipt[0].origin_jti` match that token's `iss` and `jti`.

  Issuers SHOULD include `origin_jti` when the issued token has `jti`.  Strict-mode recipients requiring instance binding MUST require `origin_jti` on `receipt[0]` when the outer token has `jti`.  See {{receipt-instance-binding}} and {{strict-mode-validation}} for validation and reissuance handling.

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

The hash input is the exact compact JWS string, without JSON {{RFC8259}} canonicalization.  Systems that carry, store, or forward `actor_receipts` arrays MUST preserve each receipt byte-for-byte.  Re-encoding changes the hash even if the claims remain equivalent.

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
2.  MUST verify that every inbound receipt's `exp` is no earlier than the issued outer token's `exp`.  A failure is an inbound validation failure.  Issuers MAY allow a small, deployment-defined clock-skew margin consistent with consumer validation, but MUST NOT accept a larger expiry gap.
3.  MUST preserve each inbound receipt byte-for-byte unchanged.
4.  MUST create exactly one new receipt for the new outermost actor hop.
5.  MUST prepend that new receipt to the inherited array.
6.  When the inherited array is non-empty, MUST set the new receipt's `prh` to the hash of the exact compact serialization of the receipt now at the next array index, computed using the algorithm named by `prh_alg` (defaulting to SHA-256 when `prh_alg` is absent).
7.  MUST set the new receipt's `prh_alg` to the inherited value, or omit `prh_alg` if the inherited chain omits it (preserving the SHA-256 default for the chain).  An issuer that does not support the inbound `prh_alg` value MUST reject the chain rather than rehash; rehashing would invalidate prior issuers' signatures.
8.  MUST preserve `actor_receipts_complete: true` when the inbound attestation is valid and the new receipt covers the added hop.  Otherwise, the issuer MUST NOT set it to `true` and SHOULD set it to `false`.

An issuer MUST NOT reserialize, resign, normalize, trim, or otherwise alter a prior receipt.

If inbound receipts fail validation, the issuer MUST NOT propagate them.  It MAY continue without `actor_receipts` only when local policy permits partial coverage; otherwise it MUST fail the request under the error model of the underlying protocol.

## Reissuance Without a New Actor Hop

An issuer that reissues, translates, or introspects and re-emits a token without adding a new outermost actor hop:

*  MAY carry an inbound `actor_receipts` array forward unchanged;
*  MUST NOT create a new receipt;
*  MUST preserve `actor_receipts_complete` when carrying the array unchanged.  If the issuer cannot attest that value, it MUST drop the array entirely; disclosure is all-or-nothing ({{consumer-introspection}}).
*  MUST NOT continue to carry an inherited `actor_receipts` array if it cannot preserve the visible hop alignment required by {{consumer-processing}};
*  MUST NOT change top-level `sub` while retaining receipts.  Subject re-expression breaks alignment with `receipt[0].sub` and requires dropping the array.

If such an issuer changes the visible outermost actor, it has added a new hop and MUST follow {{extending-an-existing-receipt-chain}}.

Reissuance MAY change `aud`, `scope`, `cnf`, `exp`, and other current-request claims without changing receipts.  Receipt `cnf` remains historical, so key rotation alone does not invalidate the chain.  The `sub` and hop-alignment restrictions above still apply.

Reissuance is the only case in which `receipt[0]` may legitimately diverge from the current outer-token instance.  Two patterns of divergence are possible:

*  **Different-issuer reissuance**: `receipt[0].iss` differs from the outer token's `iss`, for example when an introspection endpoint operated as a separate trust principal re-emits the token, or a token translator at a domain boundary re-issues it.
*  **Same-issuer reissuance**: `receipt[0].iss` matches the outer token's `iss`, but a present `receipt[0].origin_jti` differs from the outer token's `jti`, for example when an AS refreshes its own access token.

In either case, `origin_jti` remains historical and no longer binds the chain to the current instance.  Recipients accept such divergence only under the reissuing-issuer policy in {{receipt-to-token-binding-limits}}.

Refresh-token reissuance is a special case of reissuance under this section.  Receipt-bearing refresh is interoperable only when local policy defines a bounded maximum delegated-session lifetime for tokens that may inherit the receipts.

An AS that supports refresh tokens for delegated access tokens:

*  needs to retain the `actor_receipts` array associated with the original access token in issuer-controlled state across refresh, either in durable storage (for example, a token-state database or refresh-token state) or embedded in a self-contained refresh token, so each refreshed access token can carry the receipts forward unchanged.
*  MUST set receipt `exp` values under {{receipt-claims}} to accommodate the bounded maximum delegated-session lifetime.  Otherwise downstream issuers reject inbound chains under {{extending-an-existing-receipt-chain}} as receipts approach expiry, and refresh loses receipt-based provenance.
*  When that bounded lifetime would be exceeded, MUST either obtain fresh delegation state and start a new receipt chain or stop emitting `actor_receipts`, unless local policy permits partial or absent receipt coverage.

## Partial Coverage and Full Coverage

This document permits partial receipt coverage for progressive deployment.  An issuer MAY begin a new receipt chain even when older inner actor hops remain visible but uncovered.

However:

*  a partial chain MUST still cover a contiguous outermost prefix of the visible actor chain;
*  an issuer MUST NOT skip an outer visible hop and receipt only an inner visible hop;
*  when local policy or resource requirements require full provenance, the issuer MUST either emit complete receipt coverage or fail the request under the error model of the underlying protocol.

Partial coverage leaves the oldest hops uncovered, including the original subject-to-actor delegation.  Deployments needing evidence for that hop SHOULD enable receipt support at the origin issuer first.  Resource servers can require full coverage through `actor_receipts_complete_required` or local policy.

When the issuer also filters the visible `act` chain (see the `chain_complete` introspection member defined in the core actor profile {{I-D.mcguinness-oauth-actor-profile}}), `actor_receipts` covers only the visible filtered chain.  In that case `actor_receipts_complete` describes coverage relative to the visible filtered chain, not the unfiltered delegation chain; recipients that need true-chain completeness MUST evaluate `chain_complete` separately.

For inline JWT tokens, this document defines no `chain_complete` JWT claim.  A recipient that needs true-chain completeness for inline JWT tokens MUST obtain that signal from trusted deployment context, introspection, or another profile; `actor_receipts_complete: true` alone attests only complete receipt coverage for the visible `act` chain.

## Transaction Token Service Rebinding

A TTS that adds a presenter as the new outermost actor follows {{extending-an-existing-receipt-chain}}, or {{creating-the-first-receipt}} if no receipts exist.  The new receipt can record the presenter's `cnf` under {{receipt-claims}}; inherited receipts retain their historical bindings.

This profile defines no Transaction Token-specific receipt claims.  Transaction semantics follow the underlying specification and deployment profile.

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
    *  enforce `exp`, `iat`, and other JWT validity rules.  An expired receipt is invalid even for an older hop; only the small clock-skew leeway of {{RFC7519, Section 4.1.4}} applies.
    *  for `receipt[0]`, apply {{receipt-instance-binding}}.  For older receipts, `origin_jti` is historical information only.
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

## Receipt Instance Binding {#receipt-instance-binding}

Consumer step 5 applies the following cases in order to `receipt[0]`:

1.  If its `iss` matches the outer issuer and its `origin_jti` is present and matches the outer token's `jti`, the chain is bound to that token instance.
2.  If the issuers match but the outer token has no `jti`, the chain supplies provenance without instance binding.  Any `origin_jti` is informational.
3.  If the issuers match and the outer token has `jti`, but `origin_jti` is absent, the recipient MAY accept provenance under local policy.  It MUST NOT treat the chain as instance-bound.
4.  Otherwise, the recipient MUST reject the chain unless local policy trusts the outer issuer to reissue chains led by this receipt issuer ({{receipt-to-token-binding-limits}}).

## Subject Re-Expression Across Hops {#subject-re-expression-across-hops}

Only `receipt[0].sub` must match the outer token.  Older receipts MAY carry different subject identifiers.  A recipient requiring continuity across them MUST use explicit trusted local mapping rules.

Matching actors alone do not establish subject continuity.  A receipt from an unrelated subject chain that shares the same actor identity can satisfy the hop-alignment check, whether by accident or because a compromised upstream issuer minted it for insertion.  Recipients MUST account for this cross-subject insertion risk.

Deployments requiring subject continuity SHOULD either require identical subject identifiers throughout or positively reconcile them through trusted mappings.  When neither applies, recipients MUST treat continuity as unverified and MUST NOT use the differing older receipts for authorization.

## Complete Receipt Coverage

Coverage is structurally complete when all validation succeeds and the receipt count equals the visible actor depth.  This suffices when local policy requires only structural completeness.

With `actor_receipts_complete_required: true`, the token or introspection response MUST also carry `actor_receipts_complete: true`.  Recipients MUST reject tokens that fail the applicable completeness requirement.

## Use by Resource Servers

Resource servers can use validated receipts as provenance input for authorization, diagnostics, and audit, subject to the limits in {{threat-model}}.  However, a valid receipt chain:

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

For opaque tokens, the issuer stores receipts and returns them to authorized resource servers through introspection.  The same format and consumer rules apply, using the response as the outer token's claim set.

An introspection response carrying receipts MUST include the members needed for {{consumer-processing}}: the token's top-level `sub`, the visible `act` chain, and the token's `iss`.  It SHOULD include the token's `jti` when maintained by the server; when the response omits `jti`, recipients apply the no-`jti` case in {{receipt-instance-binding}}.  If present, `actor_receipts_complete` MUST be a boolean.

An RS receiving both inline and introspected receipts MUST select an authoritative source under local policy.  If it consumes both, differing arrays or completeness values MUST cause rejection of receipt-based provenance.

An introspection server MUST return the full stored array or omit `actor_receipts`.  Removing an older entry breaks `prh`; removing the newest breaks hop alignment.  A stored array with partial coverage is returned in full with `actor_receipts_complete: false`.

For an inactive token, the introspection server MUST NOT return `actor_receipts` or `actor_receipts_complete`.

Consumers that rely on both signals MUST evaluate `chain_complete` and `actor_receipts_complete` independently.  The former describes filtering of the visible `act` chain; the latter describes receipt coverage of that chain.  Complete receipt coverage therefore does not prove that no actors were filtered.

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

Companion profiles that define their own per-hop signed artifact arrays SHOULD use the same naming convention, SHOULD advertise support in AS metadata {{RFC8414}}, and SHOULD advertise requirements in Protected Resource Metadata {{RFC9728}}:

| Purpose | Name |
|---------|------|
| Artifact array | `<name>` |
| Coverage attestation | `<name>_complete` |
| AS support | `<name>_supported` |
| Resource requirement | `<name>_required` |
| Complete-coverage requirement, if applicable | `<name>_complete_required` |

Companion profiles add per-hop signal in two distinct patterns, and SHOULD pick the one that matches their signer:

*  **Parallel artifact arrays**: a companion outer-token claim parallel to `actor_receipts`, following the `<name>` plus `<name>_complete` convention above.  Each entry is a separately signed JWT.  Use this pattern when the artifact needs its own signer trust independent of the receipt issuer (for example, actor-signed proofs whose threat model differs from AS-signed receipts, or recipient-signed acknowledgments).
*  **Per-receipt extension claims**: a claim added to each receipt JWT and verified as part of the receipt's signature.  Use this pattern when the assertion is something the receipt issuer is already attesting (for example, per-hop authority bounds, delegation-flow correlation, or lifecycle-state snapshots).  Recognized per the unrecognized-claims rule in {{receipt-claims}}; cross-receipt verification follows {{extensibility}}.

The two patterns serve different threat models and have different completeness semantics: parallel-array companions inherit the claim-pair coverage machinery defined here; per-receipt-claim companions define their own completeness rules over the set of receipts that carry the claim.

# Error Handling {#error-handling}

Receipt validation failures use the underlying protocol's error mechanism for the stage at which validation occurs.

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
*  **Events not tied to a new visible actor hop**, such as re-authorization without a hop change, lifecycle-state changes, receiver acknowledgments, or sender-constraint rotation.  Companion profiles SHOULD use a separate JWT type and a parallel outer-token array.  They SHOULD anchor each event to a receipt `jti` or a defined flow identifier, such as Transaction Token `txn` {{I-D.ietf-oauth-transaction-tokens}}.  Event chaining with `prh` / `prh_alg` and the completeness convention are optional.  Companion profiles MUST NOT add event entries to `actor_receipts`, which is reserved for the AS-signed hop receipts defined by this document.

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

The following threats and limits assume the trust and validation rules in this document.

### Adversaries Mitigated by This Profile

*  **Compromised downstream issuer fabricating prior-hop provenance.**  Cannot forge prior issuers' receipt signatures; `prh` chain prevents dropping or reordering inner receipts.
*  **Token mutation in transit.**  Each receipt is independently signed; modification invalidates the receipt's signature and any newer receipt's `prh`.
*  **Receipt transplantation between tokens with matching visible `act` chains.**  Outer-token signature prevents non-issuer parties from constructing a substitute outer token to host transplanted receipts.  `receipt[0].origin_jti` provides diagnostic confirmation in the originating-issuance case (see {{receipt-to-token-binding-limits}}); it does not extend the threat model beyond the outer-token-signature defense, since outer-token issuer compromise is out of scope ({{compromised-outer-issuer}}).
*  **Partial-coverage misclaim.**  An issuer cannot drop an inner receipt without breaking the `prh` chain; `actor_receipts_complete: true` cannot be claimed without a count matching visible chain depth.

### Adversaries Not Mitigated

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

*  A recipient needs to establish which issuers it trusts for receipt validation before relying on `actor_receipts`.
*  Trust MUST be established through explicit pre-configuration, bilateral agreement, federation policy, or another explicit trust framework.
*  A recipient MUST NOT treat the presence of a syntactically valid signed receipt as sufficient grounds to trust its issuer.
*  Authorization servers that support this document SHOULD advertise `actor_receipts_supported: true` in their AS metadata {{RFC8414}}.
*  Consumers SHOULD use that metadata signal as one input to trust establishment, but MUST NOT treat metadata advertisement alone as sufficient grounds to trust a receipt issuer; the issuer must also be within the recipient's configured trust boundary.

Key resolution requirements:

*  To avoid attacker-controlled key resolution, a recipient MUST determine whether a receipt `iss` is within its trusted-issuer set before performing any network retrieval for that issuer's metadata or keys.
*  A recipient that uses dynamic discovery for receipt validation MUST do so only within an existing trust framework or equivalent local policy that defines which issuers are permitted.

Trust is per-issuer and not transitive: each receipt is validated against the recipient's own trusted-issuer set, independent of the outer token's issuer or neighboring receipts.  If any receipt in the presented `actor_receipts` array is signed by an issuer that is not trusted for receipt validation, the recipient MUST reject the receipt chain for the purposes of this profile.  This document does not define trusted-prefix validation across an untrusted inner receipt.  Deployments needing uniform trust across an extended chain need to establish trust explicitly with every receipt issuer that may appear in tokens they accept.

The trust evaluation in this section covers receipt signers (the `iss` claim of each receipt).  Recipients separately evaluate trust in each receipt's `act.iss` as the namespace authority for `act.sub` as described in {{receipt-claims}}; that evaluation is independent of receipt-signer trust, even when the same entity holds both roles.

## Receipt-to-Token Binding Limits {#receipt-to-token-binding-limits}

Receipts prove that trusted issuers attested particular actor hops and, optionally, historical presenter bindings.  They do not prove that the current outer token's audience, scope, expiration, or other authorization details were in force when older receipts were created.  A recipient MUST NOT treat a valid receipt chain as evidence of historical authorization scope or audience beyond what the current outer token authorizes.

In the originating-issuance case, receipt-chain integrity rests on two anchors when the outer token carries `jti` and `receipt[0].origin_jti` is present:

*  `receipt[0].origin_jti`, when present, signed by the same issuer that signed the outer token, and equal to the outer token's `jti`, binds `receipt[0]` to the specific outer-token instance and prevents transplantation from a different token whose visible `act` structure happens to match.
*  `prh` chains each receipt cryptographically to its older neighbor, so all inner receipts inherit the originating-issuance binding from `receipt[0]` through the hash chain.

Inner `origin_jti` values are historical and cannot independently bind the current request.  Deployments requiring instance binding MUST rely on `prh` and a verifiable leading `origin_jti`.  Without that anchor, the chain supplies issuer-signed hop provenance only.

This construction makes coverage tamper-evident at the structural level:

*  An issuer cannot drop an inner receipt without breaking the `prh` chain: the next-newer receipt's `prh` value would no longer match the receipt now in the next array position, and consumer step 6 of {{consumer-processing}} rejects the chain.
*  An issuer can withhold coverage only from the innermost (oldest) end of the chain, and only by beginning a new chain under {{creating-the-first-receipt}} rather than trimming an inherited one; a trimmed chain leaves the surviving oldest receipt carrying a `prh` with no target, which step 6 also rejects.  The result is partial coverage that `actor_receipts_complete: true` then forbids the issuer from claiming.

Coverage is therefore truthful within the limits of the trusted-issuer set: a compromised issuer can omit some or all of its own receipts and any outermost receipts from issuers it controls, but it cannot fabricate, reorder, or selectively drop receipts signed by other trusted issuers.

Divergence in issuer or token identifier removes current-instance binding.  {{receipt-instance-binding}} rejects such chains unless local policy explicitly trusts the outer issuer to reissue them.  This profile cannot distinguish legitimate reissuance from malicious rewrapping in band.

Recipients accepting reissuance configure that trust locally or through an out-of-band framework, and unexpected divergence warrants investigation.  Outside the originating-issuance case, protection against non-issuer transplantation depends on the outer signature.  A compromised outer issuer can create a replacement token; see {{compromised-outer-issuer}}.

Companion profiles MAY define additional outer-token binding claims following the `origin_jti` pattern: each records an identifier from the outer token at receipt creation, with consumer verifiability conditioned on issuer alignment and equality with the current outer-token field.  Such claims provide parallel anchors against other outer-token fields and do not weaken the `origin_jti` anchor.

### Strict-Mode Validation {#strict-mode-validation}

Without configured trusted reissuing issuers, recipients use strict mode: issuer or `origin_jti` divergence causes rejection.  If the outer token has `jti`, a recipient requiring instance binding also rejects a missing leading `origin_jti`.

Strict mode is the recommended default.  Deployments that need to accept reissued tokens, such as refreshed, re-emitted, or translated tokens, need to configure the trusted reissuing issuers explicitly, through local policy or an out-of-band trust framework.

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

Receipt expiration bounds use of the artifact, not the delegation's lifetime.  Reuse of a receipt within its `exp` window, including in extended, fanned-out, and reissued tokens, is not in itself an attack; replay protection for the whole token follows its token type.  Current authorization and revocation checks remain separate.

Deployments needing freshness signals beyond receipt `exp`, such as active delegation status, fresh authorization confirmation, or current revocation state, MUST obtain those signals from the AS via introspection ({{RFC7662}}), fresh token issuance, or another mechanism outside the scope of this profile.

## Receipt Signing Key Compromise

If a receipt issuer's signing key is compromised, previously issued receipts signed with that key cannot be individually revoked.  The primary remediation is to remove the compromised issuer from the trusted-issuer set; once removed, consumers will reject all receipts signed by that issuer regardless of their content.

Deployments SHOULD keep receipt `exp` no longer than the delegated-session lifetime it needs to cover ({{receipt-claims}}), to limit the window during which receipts signed with a compromised key remain valid.  When a key compromise is detected, deployments SHOULD treat all tokens carrying receipts from the affected issuer as lacking trusted provenance for those hops and SHOULD require re-issuance through a trusted issuer.

## Receipt Chain Size

Each receipt is a full signed JWT, and the chain grows linearly with delegation depth.  A typical signed receipt is 400 to 800 bytes after JWS compact serialization and base64url encoding (the upper end when `cnf` or larger `act` objects are present).  Chains beyond approximately 10 hops therefore approach the 8 KB Authorization header budget common in HTTP infrastructure; chains beyond approximately 20 hops approach a 16 KB practical ceiling.  Figures are illustrative and depend on the deployment.

Deployments SHOULD verify that the outer token plus its `actor_receipts` array fits within the header-size budget of every component on the request path.  When introspection is available, deployments MAY return receipts via introspection rather than embedding them, to avoid header pressure for bearer-token clients.

## Historical `cnf` Disclosure {#historical-cnf-disclosure}

Receipt `cnf` values reveal prior-hop public-key identifiers or certificate thumbprints to any party that receives the token or introspection response.  These are stable identifiers that enable cross-request and cross-service correlation of actors and services over time.  Issuers SHOULD NOT include `cnf` in receipts unless the relying parties that will receive the token have been evaluated for that disclosure risk and the risk is acceptable.  Omitting `cnf` does not invalidate the receipt; it means that hop lacks independently attested historical presenter binding, which is acceptable for many deployments.

# Privacy Considerations

Receipts expose delegation history that recipients can retain and verify after the outer token expires.

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

Extension claims can expose additional scope, audience, resource, or lifecycle information.  Companion profiles defining extension claims SHOULD document the disclosure and correlation risks specific to their claims, including retention beyond token lifetime.

## Detached Verification Privacy

Any holder with the issuers' public keys can verify receipts, including unintended recipients.  Deployments SHOULD treat token distribution as disclosure of the full carried provenance and SHOULD limit distribution accordingly.

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

Contributors and reviewers will be acknowledged in future revisions.

--- back

# Examples

The examples in this appendix show decoded receipt contents.  Real receipts are compact-signed JWT strings carried in the `actor_receipts` array.  The `iat` and `exp` values shown are illustrative only; in deployments, receipt `exp` is set per {{receipt-claims}} and {{extending-an-existing-receipt-chain}} so that no inbound receipt expires before the outer token that carries it.

Examples containing `cnf` illustrate explicit disclosure of historical binding.  Issuers SHOULD omit it unless recipient disclosure risk has been evaluated ({{receipt-claims}}).

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

Under {{receipt-instance-binding}}, `origin_jti` is historical here because the outer issuer and token identifier have changed.  The same rule applies when an AS refreshes its own token with a new `jti`.  In either case, acceptance requires explicit trust in the reissuing issuer ({{receipt-to-token-binding-limits}}).

# Document History
{:numbered="false"}

[[ To be removed from the final specification ]]

-01

* Consolidated and tightened the text throughout; the claim-pair naming convention now uses a table.
* Gathered the receipt instance-binding rules for `origin_jti`, strict mode, and reissuance into one section.
* Defined reissuance divergence as a mismatch between `receipt[0]` and the outer token's `iss` or `jti`.
* Clarified that the claim-pair naming convention and its metadata apply to companion profiles that define parallel per-hop artifact arrays.

-00

* Initial version.

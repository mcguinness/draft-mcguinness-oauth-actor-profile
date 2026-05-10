---
title: "OAuth Actor Receipts for Delegation Provenance"
abbrev: "OAuth Actor Receipts"
category: exp
docname: draft-mcguinness-oauth-actor-receipts-latest
submissiontype: IETF
number:
date: 2026-05-09
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
  RFC9700:

...

--- abstract

This document defines OAuth Actor Receipts, an optional companion provenance profile for delegated OAuth tokens that conform to the OAuth Actor Profile for Delegation.  It introduces the `actor_receipts` claim, a signed per-hop receipt chain that records which issuer added each visible actor hop, preserves the historical top-level `cnf` value associated with that hop, and links receipts together so recipients can validate prior-hop provenance without relying solely on the current outer token issuer.  This document also defines metadata and introspection parameters for advertising and consuming actor-receipt support.

--- middle

# Introduction

The OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}} makes actor identity visible in delegated tokens by standardizing the `act` claim across JWT assertion grants, JWT access tokens, and Transaction Tokens.  That core profile is intentionally narrow: it defines who the current subject is, who the visible actors are, and how the current presenter proves possession of the key bound in the token's top-level `cnf` claim.  It does not attempt to make prior-hop actor-key history independently verifiable across trust boundaries.

Some deployments need stronger provenance.  A relying party can often see an `act` chain, but without independently signed prior-hop evidence it still depends on the current token issuer to have preserved that chain faithfully.  The problem is sharper when deployments want historical sender-constraint context: carrying key history inline in nested `act` objects makes that history readable, but it does not make it independently trustworthy.

This document defines an optional companion profile, OAuth Actor Receipts, for deployments that need signed prior-hop provenance.  The design center is:

*  keep the visible actor chain in `act`, as defined by the core actor profile;
*  keep active presenter proof of possession in the token's top-level `cnf`;
*  carry prior-hop provenance, including historical top-level `cnf` values, in separately signed hop receipts.

This document applies to tokens that already conform to the OAuth Actor Profile for Delegation.  It supplements that profile; it does not replace it.  Receipts add an additive top-level JWT claim and a small set of metadata signals on top of the existing OAuth ({{RFC6749}}, {{RFC8693}}) and core-actor-profile trust model; deployments opt in per resource server or per trust domain.  Detailed scope is captured in [Design Goals and Non-Goals](#design-goals-and-non-goals).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Unless otherwise specified, OAuth terms such as client, authorization server, resource server, access token, refresh token, grant, `subject_token`, and `actor_token` are used as defined in {{RFC6749}} and {{RFC8693}}.  Transaction Token and Transaction Token Service (TTS) are used as defined in {{I-D.ietf-oauth-transaction-tokens}}.

The following terms are used in this document:

Actor Receipt:
: A signed JWT that attests one visible actor hop in a delegated token chain.

Outer Token:
: The JWT access token or Transaction Token whose top-level `actor_receipts` claim carries the receipt chain.  Distinguished from the receipt JWTs nested within it.

Receipt Chain:
: The ordered `actor_receipts` array carried in a token or introspection response.

Historical Presenter Binding:
: The top-level `cnf` value that was present in the token issued at the hop represented by a receipt.  Historical presenter binding is informational provenance for that hop; it does not create an active proof-of-possession obligation for the current request.

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
*  defining transparency logs, non-repudiation systems, or public audit infrastructure.

## Deployment Fit

This profile's value scales with the number of distinct issuers in the trust set.  In federated or cross-domain deployments, where multiple authorization servers and Transaction Token Services participate in a delegation chain and each is independently trusted by the recipient, receipts let the recipient validate prior-hop attestations against the issuers that actually performed those hops, rather than relying solely on the current outer token issuer's faithful preservation of the chain.  In single-domain deployments where the outer token issuer is the only trusted source for delegation state, the outer token's own signature already conveys that issuer's attestation; in those deployments, receipts add overhead without adding meaningful provenance, and SHOULD be weighed against that overhead before adoption.

The threat model receipts address is summarized in {{trust-in-receipt-issuers}} and {{compromised-outer-issuer}}: receipts mitigate a compromised or dishonest *downstream* issuer that attempts to fabricate prior-hop provenance.  They do not mitigate a compromised current outer token issuer, and they do not provide actor non-repudiation; deployments needing those properties require additional mechanisms outside the scope of this document.

# Actor Receipts Overview

An actor receipt records one actor hop.  The issuer that adds a new outermost actor hop signs a receipt describing that hop and, when the issued token is sender-constrained, copies the token's top-level `cnf` value into the receipt as historical presenter-binding context.

The token then carries an `actor_receipts` array:

*  one array entry per covered actor hop;
*  newest receipt first;
*  older receipts preserved unchanged.

This ordering aligns directly with the visible `act` chain in the outer token.  `actor_receipts[0]` corresponds to the outermost `act` object, `actor_receipts[1]` corresponds to `act.act`, and so on.

Receipts can cover either:

*  the full visible chain; or
*  a contiguous outermost prefix of the visible chain.

When a deployment requires full provenance, local policy or resource requirements enforce complete receipt coverage.

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
: OPTIONAL.  A boolean JWT claim in the outer token.  When `true`, the issuer attests that `actor_receipts` covers every visible hop in the token's `act` chain.  This attestation is relative to the visible chain at the time of token issuance; it does not attest that the visible chain is itself unfiltered (see `chain_complete` in the core actor profile {{I-D.mcguinness-oauth-actor-profile}}).  When `actor_receipts_complete` is `true` in the outer token, the consumer MUST verify that the receipt count equals the visible actor-chain depth; if it does not, the consumer MUST reject the token.  Issuers SHOULD set `actor_receipts_complete: true` when they emit complete coverage, to enable consumers to detect chain truncation.  Issuers SHOULD set `actor_receipts_complete: false` when they emit partial coverage; omitting the claim is observationally equivalent to `false` for consumers that test only for the literal value `true`, but does not provide a positive attestation that coverage is partial.

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

`sub_profile`:
: OPTIONAL.  The top-level `sub_profile` value, when the token issued at this hop carried one.

`act`:
: REQUIRED.  A single-hop actor object.  This object:

  *  MUST conform to the core actor profile's actor-object rules;
  *  MUST include `act.sub` and `act.iss`;
  *  MAY include `act.sub_profile`;
  *  MUST NOT contain `cnf`;
  *  MUST NOT contain a nested `act`.

  The prohibition on `act.cnf` exists because a `cnf` claim inside an `act` object would create ambiguity about whether it represents a historical key binding or a current proof-of-possession requirement for the actor.  Historical presenter binding belongs at the receipt's top level in the `cnf` claim, where its role as provenance-only information is unambiguous.  Additional claims permitted by the core actor profile in `act` objects MAY appear in receipt `act` objects unless explicitly prohibited here.  A receipt that carries `act.cnf` is invalid under this profile.

### Historical Presenter Binding

`cnf`:
: OPTIONAL.  A confirmation claim as defined in {{RFC7800}}.  When present, it MUST equal the top-level `cnf` claim of the token issued at this hop.

  Receipt `cnf` records historical presenter-binding information for the hop represented by the receipt.  It does not create a current proof-of-possession obligation for the current request.

  Issuers SHOULD NOT include `cnf` in a receipt unless the relying parties that will receive the token have been evaluated for the associated disclosure risk; omitting `cnf` does not invalidate the receipt.

### Chain Linkage

`prh`:
: OPTIONAL.  Previous receipt hash.  When present, `prh` MUST be the base64url-encoded hash of the complete compact serialization of the next older receipt in the chain, computed using the algorithm identified by `prh_alg` (defaulting to SHA-256 when `prh_alg` is absent).  The oldest receipt in the chain, including a single-element chain in which the sole receipt is both newest and oldest, MUST omit `prh`.

`prh_alg`:
: OPTIONAL.  Hash algorithm identifier naming the algorithm used to compute `prh`.

  *  Values MUST be drawn from the IANA "Named Information Hash Algorithm Registry" {{RFC6920}}, which uses lowercase forms such as `sha-256`, `sha-384`, and `sha-512`.
  *  When absent, the default is `sha-256`.
  *  When present, the value MUST identify a hash algorithm whose collision and preimage resistance is at least equivalent to `sha-256`.
  *  All receipts in a single `actor_receipts` array MUST use the same `prh_alg` value, so that recipients can validate the chain without per-receipt algorithm negotiation.
  *  An issuer extending an inbound chain MUST either preserve the inbound `prh_alg` or reject the chain.

### Time and Uniqueness

`iat`:
: REQUIRED.  The time at which the receipt was created, as defined in {{RFC7519}}.

`exp`:
: REQUIRED.  Expiration time for the receipt, as defined in {{RFC7519}}.

  *  `exp` MUST be set to a value that covers the expected maximum token lifetime of any token that will carry or inherit this receipt, so that consumer validation of older receipts in a valid chain is not prematurely rejected.
  *  Issuers SHOULD set `exp` to the maximum delegated-token lifetime permitted under local policy for tokens that may inherit this receipt.

  Downstream issuers reject inbound receipts whose `exp` precedes the issued outer token's `exp` ({{extending-an-existing-receipt-chain}}), so an under-set `exp` causes propagation failure rather than mid-token-lifetime consumer rejection.  Short `exp` values on receipts limit the window during which a compromised receipt signing key can be exploited.

`jti`:
: REQUIRED.  A unique identifier for the receipt, as defined in {{RFC7519}}.

### Outer-Token Binding

`token_id`:
: RECOMMENDED.  The `jti` value of the specific outer token that this receipt was created for.

  When present, `token_id` binds the receipt to the specific token instance issued at this hop, enabling consumers to detect receipts transplanted from a different token whose visible `act` structure happens to match.  Issuers SHOULD include `token_id` whenever the token they are issuing carries a `jti` claim.

  Consumer verification of `token_id` is defined in {{consumer-processing}}:

  *  for `receipt[0]`, `token_id` is independently verifiable when `receipt[0].iss` equals the outer token's `iss` (the originating-issuance case);
  *  for receipts other than `receipt[0]`, `token_id` records the historical outer-token `jti` at the hop where the receipt was created and is informational provenance only.

### Excluded Standard Claims

`aud`:
: NOT RECOMMENDED.  Issuers SHOULD omit `aud` from receipts.

  A receipt travels with the outer token to whichever audiences the outer token serves.  Receipt validity is anchored to trust in `iss`, the binding to the outer token via `token_id` and `prh`, and the outer token's own audience scoping.  Including `aud` in a receipt has no defined meaning under this profile.

  This profile diverges from the audience-validation guidance in {{RFC8725}} Section 3.10 because receipts are not validated as independent JWTs against an audience; they are validated as part of outer-token processing, and the outer token carries the audience scoping.  Including `aud` in a receipt would create ambiguity about whether the receipt asserts an audience constraint independent of the outer token, which it does not.

### Extension Claims

A receipt MAY contain additional claims defined by another specification or by deployment policy.  Consumers MUST ignore unrecognized claims unless another specification or local agreement defines their meaning.

## Receipt-Chain Linkage

When the issuer creates a new receipt and prepends it to an inherited receipt chain:

*  if there is an older receipt immediately following it in the array, the new receipt MUST include `prh`, and that value MUST hash the exact compact JWT string of that next receipt;
*  if the new receipt is the only receipt in the array, it MUST omit `prh`.

No JSON canonicalization is applied.  `prh` hashes the exact compact-serialized JWS string of the next older receipt as carried in the array.  Systems that carry, store, or forward `actor_receipts` arrays MUST preserve each compact JWT string byte-for-byte without parsing, re-serializing, normalizing whitespace, or re-encoding base64url segments.  Any modification to a receipt string, including semantically equivalent re-encoding, invalidates `prh` for any receipt that references it.

# Issuer Processing

This section defines how an authorization server or Transaction Token Service creates, preserves, and extends `actor_receipts`.

When an issuer adds a new outermost actor hop and creates the corresponding receipt, that issuer is the same authorization server or Transaction Token Service that signs the outer token carrying the new hop.  As a result, in tokens emitted under {{creating-the-first-receipt}} or {{extending-an-existing-receipt-chain}}, `receipt[0].iss` is equal to the outer token's `iss`.  The only case in which `receipt[0].iss` legitimately differs from the outer token's `iss` is reissuance of an existing token without adding a new actor hop, as defined in {{reissuance-without-a-new-actor-hop}}.  Consumer rules in {{consumer-processing}} use this property to scope the bind-to-current checks for `receipt[0]`.

## Creating the First Receipt

When an issuer creates a delegated token with a new outermost actor hop and no inbound `actor_receipts` are being preserved, the issuer MAY create a new one-element `actor_receipts` array.

If it does so, the new receipt:

*  MUST describe the new outermost actor hop;
*  MUST set `sub` to the issued token's top-level `sub`;
*  MUST set `act.sub` and `act.iss` to the new outermost actor;
*  MAY copy the issued token's top-level `cnf`, if any, into the receipt `cnf`, subject to the disclosure considerations in {{receipt-claims}}; when copied, the receipt `cnf` MUST equal the outer token's `cnf` value;
*  SHOULD set `token_id` to the issued token's `jti`, if the issued token carries a `jti`;
*  MUST omit `prh`.

## Extending an Existing Receipt Chain

When an issuer adds a new outermost actor hop and also preserves an inbound `actor_receipts` array, it:

1.  MUST validate the inbound receipt chain by applying the consumer processing rules in {{consumer-processing}} before relying on it or carrying it forward.
2.  MUST verify that each inbound receipt's `exp` is no earlier than the issued outer token's `exp`.  Carrying a receipt forward whose `exp` precedes the outer token's `exp` would cause consumers to reject the chain mid-token-lifetime under {{consumer-processing}}; an inbound receipt that fails this check is treated as failing validation under step 1.  Issuers MAY apply a small clock-skew margin to this comparison, consistent with the consumer-side skew tolerance in {{consumer-processing}}, but MUST NOT broadly accept inbound receipts whose `exp` precedes the issued outer token's `exp` by more than a deployment-defined skew bound.
3.  MUST preserve each inbound receipt byte-for-byte unchanged.
4.  MUST create exactly one new receipt for the new outermost actor hop.
5.  MUST prepend that new receipt to the inherited array.
6.  MUST set the new receipt's `prh`, if the inherited array is non-empty, to the hash, computed using the algorithm named by `prh_alg` (defaulting to SHA-256 when `prh_alg` is absent), of the exact compact serialization of the receipt that is now at the next array index.
7.  MUST propagate `prh_alg`: if the inherited chain carries `prh_alg`, the new receipt MUST carry the same `prh_alg` value; if the inherited chain omits `prh_alg`, the new receipt MUST also omit it (preserving the SHA-256 default for the entire chain).  An issuer that does not support the inbound `prh_alg` value MUST reject the chain rather than rehash, since rehashing would invalidate prior issuers' signatures.

An issuer MUST NOT reserialize, resign, normalize, trim, or otherwise alter a prior receipt.

If inbound receipts fail validation, the issuer MUST NOT propagate them.  It MAY continue without `actor_receipts` only when local policy permits partial coverage; otherwise it MUST fail the request under the error model of the underlying protocol.

## Reissuance Without a New Actor Hop

An issuer that reissues, translates, or introspects and re-emits a token without adding a new outermost actor hop:

*  MAY carry an inbound `actor_receipts` array forward unchanged;
*  MUST NOT create a new receipt;
*  MUST NOT continue to carry an inherited `actor_receipts` array if it cannot preserve the visible hop alignment required by {{consumer-processing}};
*  MUST NOT change the outer token's top-level `sub` while carrying inherited receipts forward; a `sub` change re-expresses subject identity and breaks the alignment between `receipt[0].sub` and the outer token's top-level `sub` that consumers verify under {{consumer-processing}}.

If such an issuer changes the visible outermost actor, it has added a new hop and MUST follow {{extending-an-existing-receipt-chain}}.

A change in the outer token's top-level `cnf` value, by itself, does not invalidate carried-forward receipts.  Receipt `cnf` records the historical presenter binding in effect when that hop was created.  The current token's top-level `cnf` can later change, for example because of key rotation or token reissuance, without requiring a new receipt so long as the visible actor hop itself is unchanged.

Reissuance MAY change the outer token's `aud`, `scope`, `cnf`, `exp`, and other claims that bind the current request without requiring updates to inherited receipts.  Receipts attest hop history at the time of original issuance and are unaffected by later mutations of current-request bindings.  Only `sub` is structurally constrained, because changing `sub` while carrying inherited receipts would break the alignment between `receipt[0].sub` and the outer token's top-level `sub` that consumers verify under {{consumer-processing}}.

Reissuance under this section is the only case in which `receipt[0]` may legitimately diverge from the current outer-token instance.  Two patterns of divergence are possible:

*  **Different-issuer reissuance**: `receipt[0].iss` differs from the outer token's `iss`.  For example, an introspection endpoint operated as a separate trust principal re-emits the token, or a token translator at a domain boundary re-issues under its own issuer identity.
*  **Same-issuer reissuance**: `receipt[0].iss` matches the outer token's `iss`, but `receipt[0].token_id` differs from the outer token's `jti`.  For example, an authorization server refreshes its own access token: the refreshed token is signed by the same AS but carries a new `jti`, while inherited receipts (carried forward unchanged) retain the original outer-token `jti` in `token_id`.

Consumer rules in {{consumer-processing}} relax the bind-to-current `token_id` check for `receipt[0]` in both reissuance patterns.  Recipients distinguish legitimate reissuance from a re-wrapping attack through local policy or out-of-band trust framework, as described in {{receipt-to-token-binding-limits}}.

Refresh-token reissuance is a special case of reissuance under this section.  An AS that supports refresh tokens for delegated access tokens MUST persist the `actor_receipts` array associated with the original access token in its token store, so that each refreshed access token can carry the receipts forward unchanged.  Receipt `exp` values set under {{receipt-claims}} MUST accommodate the maximum refresh-extended access-token lifetime permitted under local policy; otherwise downstream issuers will reject inbound chains under {{extending-an-existing-receipt-chain}} as receipts approach expiry, and refresh will silently lose receipt-based provenance.

## Partial Coverage and Full Coverage

This document permits partial receipt coverage for progressive deployment.  An issuer MAY begin a new receipt chain even when older inner actor hops remain visible but uncovered.

However:

*  a partial chain MUST still cover a contiguous outermost prefix of the visible actor chain;
*  an issuer MUST NOT skip an outer visible hop and receipt only an inner visible hop;
*  when local policy or resource requirements require full provenance, the issuer MUST either emit complete receipt coverage or fail the request under the error model of the underlying protocol.

When the issuer also filters the visible `act` chain (see the `chain_complete` introspection member defined in the core actor profile {{I-D.mcguinness-oauth-actor-profile}}), `actor_receipts` covers only the visible filtered chain.  In that case `actor_receipts_complete` describes coverage relative to the visible filtered chain, not the unfiltered delegation chain; recipients that need true-chain completeness MUST evaluate `chain_complete` separately.

## Transaction Token Service Rebinding

A Transaction Token Service that establishes a new presenter and makes that presenter the new outermost actor follows the same receipt rules as any other issuer that adds a new outermost actor hop, as defined in {{extending-an-existing-receipt-chain}} (or {{creating-the-first-receipt}} when no inbound `actor_receipts` exist).  This profile does not define additional receipt claims specific to Transaction Tokens; any transaction-specific semantics remain governed by the Transaction Token itself and its deployment profile.

# Consumer Processing {#consumer-processing}

An issuer, resource server, or other recipient that relies on `actor_receipts` MUST perform the following steps.

1.  Validate the outer token according to its token type and the core actor profile.
2.  If `actor_receipts` is absent, treat the token as lacking receipt-based provenance.  Whether that is acceptable is determined by local policy or by Protected Resource Metadata signals such as `actor_receipts_required` and `actor_receipts_complete_required` defined in {{discovery-capability-signaling}}.
3.  Verify that `actor_receipts`, if present, is a non-empty JSON array of strings.
4.  Verify that the number of receipts does not exceed the visible actor-chain depth of the outer token.  If the outer token carries `actor_receipts_complete: true`, verify that the receipt count exactly equals the visible actor-chain depth; if it does not, reject the token.
5.  For each receipt, in array order:
    *  parse the string as a compact JWT;
    *  verify that the receipt issuer is within the recipient's pre-configured trusted-issuer set before performing any network retrieval for that issuer's metadata or keys;
    *  resolve the signing key from the receipt issuer's authorization server metadata `jwks_uri` {{RFC8414}} (where the receipt issuer is identified by the receipt's `iss` claim, which may differ from the outer token's issuer) or from local configuration;
    *  validate the JWT signature;
    *  verify that `typ` equals `actor-receipt+jwt`;
    *  verify that the receipt `act` object is single-hop, contains no nested `act`, and contains no `cnf`;
    *  enforce `exp`, `iat`, and other JWT validity rules.  Because `exp` is REQUIRED on receipts and MUST cover the expected outer token lifetime, an expired receipt SHOULD be treated as invalid even for older hops.  Local policy MAY permit continued use of a receipt that is expired by a small clock-skew margin, but MUST NOT relax `exp` enforcement broadly as a workaround for issuers that failed to set adequate `exp` values.
    *  for `receipt[0]` only, when `token_id` is present: if `receipt[0].iss` equals the outer token's `iss` AND `receipt[0].token_id` equals the outer token's `jti`, the receipt is anchored to the current outer-token instance (the originating-issuance case).  Otherwise — whether because `receipt[0].iss` differs from `outer.iss`, or because `receipt[0].iss` matches but `receipt[0].token_id` differs from `outer.jti` — the divergence reflects reissuance per {{reissuance-without-a-new-actor-hop}}, and `receipt[0].token_id` is not required to equal the current outer token's `jti`.  Same-issuer `token_id` divergence typically reflects refresh-token-driven reissuance; different-issuer divergence reflects introspection-and-re-emission or similar paths.  In either reissuance case, the consumer relies on the trust framework in {{receipt-to-token-binding-limits}} to distinguish legitimate reissuance from a re-wrapping attack.  For receipts other than `receipt[0]`, `token_id` records the historical outer-token `jti` at the hop where the receipt was created, is not independently verifiable by the consumer, and is informational provenance only.
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
8.  Verify that `receipt[0].sub` equals the outer token's top-level `sub`.  Older receipts MAY carry differing `sub` values; see {{subject-re-expression-across-hops}}.
9.  Treat each receipt `cnf` value, if present, only as historical provenance for that hop.  A mismatch between the current outer token's top-level `cnf` and the outermost receipt `cnf` MUST NOT by itself invalidate the receipt chain under this profile.
10.  Receipt `cnf` values MUST NOT replace validation of the current request against the outer token's top-level `cnf`.
11.  Apply any additional consumer-processing rules defined by companion profiles whose claims appear in the receipt or outer token (see {{extensibility}}).  Companion-profile rules MUST NOT relax any requirement in steps 1 through 10; they MAY add additional rejection conditions.

If any required check fails, the recipient MUST reject the receipt chain for the purposes of this profile and MUST apply the underlying protocol's error handling for the stage at which the failure occurred.

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

A recipient determines complete receipt coverage by comparing receipt count with visible actor depth.  If the number of receipts equals the visible actor depth and all validation rules above succeed, the token has complete receipt coverage for the visible chain.

If local policy or resource requirements require full provenance, the recipient MUST reject tokens that do not have complete receipt coverage.

## Use by Resource Servers

Resource servers can use validated actor receipts as provenance input for authorization, diagnostics, and audit.  However, a valid receipt chain:

*  proves only that trusted issuers attested specific visible actor hops;
*  does not prove that the current token's audience, scope, or expiration were in force when older receipts were created;
*  does not replace the need to authorize the current token itself.

## Introspection {#consumer-introspection}

When an authorization server returns actor-receipt information in an OAuth Token Introspection response {{RFC7662}}, it:

*  MAY return `actor_receipts` using the same array format defined in {{actor-receipts-claim}};
*  MAY return `actor_receipts_complete` to indicate whether the returned array provides complete coverage for the visible chain as known to the introspection server.

The registered introspection response members are defined in {{introspection-response-members}}; introspection-server failure handling is addressed in {{introspection-errors}}.

Introspection is the primary delivery mechanism for receipts associated with opaque (non-JWT) outer tokens.  Such tokens cannot carry an inline `actor_receipts` claim; the issuer instead retains the receipts in its token store and surfaces them to authorized resource servers via introspection.  The receipt format and consumer processing rules above apply unchanged in this case, with the introspection response substituting for the outer token's claim set.

An introspection server that suppresses one or more receipts for privacy or policy reasons and still returns `actor_receipts` SHOULD return `actor_receipts_complete` with the value `false`.

The core actor profile {{I-D.mcguinness-oauth-actor-profile}} defines a separate `chain_complete` introspection member that indicates whether the visible `act` chain itself has been filtered.  These two completeness signals are distinct: `chain_complete: false` means the introspection server has suppressed inner `act` hops from the chain representation, while `actor_receipts_complete: false` means receipt coverage is partial or filtered.  A token can have `chain_complete: true` (full act chain visible) and `actor_receipts_complete: false` (some receipts suppressed), or vice versa.  Consumers that rely on both signals MUST evaluate them independently.  A response with `chain_complete: false` means the receipt array may cover only part of the true delegation chain even when `actor_receipts_complete: true`; in that case the receipt coverage is complete only for the visible filtered chain, not the full chain.

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

`actor_receipts_complete_required`:
: OPTIONAL.  A boolean.  When `true`, the resource server indicates that it requires complete receipt coverage: the receipt count must equal the visible actor-chain depth and `actor_receipts_complete` must be `true` in the outer token or the introspection response.  This parameter refines `actor_receipts_required`; a resource server SHOULD NOT set `actor_receipts_complete_required: true` without also setting `actor_receipts_required: true`.  When `false` or absent, partial receipt coverage is acceptable to the resource server, subject to any further local policy.

## Introspection Response Members {#introspection-response-members}

The following members are defined for use in OAuth Token Introspection responses {{RFC7662}}:

`actor_receipts`:
: OPTIONAL.  An array of strings using the same syntax as the JWT claim of the same name.

`actor_receipts_complete`:
: OPTIONAL.  A boolean.  When `true`, the introspection response indicates that the returned `actor_receipts` cover every visible hop in the token chain as known to the introspection server.  When `false`, the response indicates that coverage is partial or that the server has filtered or withheld one or more receipts.

Consumer use of these members is described in {{consumer-introspection}}; introspection-server failure handling is addressed in {{introspection-errors}}.

## Out-of-Scope Discovery Signals

This document does not define a metadata signal for "this resource server requires `cnf` to be present in receipts."  Issuers default to omitting receipt `cnf` for privacy reasons (see {{receipt-claims}} and {{historical-cnf-disclosure}}); resource servers that need historical sender-constraint provenance MUST coordinate that requirement with issuers through deployment policy or a future companion profile, rather than through metadata defined here.

## Claim-Pair Convention for Sibling Profiles

This document defines the per-hop artifact array `actor_receipts` together with the coverage attestation `actor_receipts_complete`, and the discovery triple (`actor_receipts_supported`, `actor_receipts_required`, `actor_receipts_complete_required`).  Companion profiles that define their own per-hop signed artifacts (for example, actor-signed proofs or recipient acknowledgments) SHOULD follow the same `<name>` plus `<name>_complete` claim-pair convention, advertise support with `<name>_supported` in Authorization Server Metadata {{RFC8414}}, and advertise resource-side requirements with `<name>_required` and (if applicable) `<name>_complete_required` in Protected Resource Metadata {{RFC9728}}.  Following this convention lets recipients evaluate independent companions through the same coverage and capability machinery defined here.

# Error Handling {#error-handling}

This section defines how receipt-related processing failures map to OAuth error responses.  Receipt validation extends the underlying OAuth or Transaction Token validation rather than replacing it; failures should be reported through the error-response mechanism applicable to the stage at which validation occurred.

## Authorization Server and Transaction Token Service Errors

When an authorization server or Transaction Token Service rejects a token-exchange request because inbound `actor_receipts` cannot be validated under {{extending-an-existing-receipt-chain}} (signature failure, expired receipt, unsupported `prh_alg`, broken `prh` chain, hop misalignment, or untrusted receipt issuer), it SHOULD return `invalid_grant` per {{RFC8693}} Section 2.4.

When the failure reflects an actor-authorization decision rather than a structural validation failure, an issuer MAY use `actor_unauthorized` as defined in the core actor profile {{I-D.mcguinness-oauth-actor-profile}} where applicable.

## Resource Server Errors

When a resource server rejects a request because `actor_receipts` validation fails under {{consumer-processing}}, it SHOULD return `invalid_token` per the bearer-token error model in {{RFC6750}} Section 3.1.

When the failure is specifically that required receipts are absent or coverage is incomplete (per `actor_receipts_required` or `actor_receipts_complete_required`), the resource server SHOULD include an `error_description` value identifying receipt-coverage failure so that clients and operators can distinguish it from generic token-validation failures.

## Introspection Server Behavior {#introspection-errors}

When an introspection server cannot return receipts that the requesting resource server requires, it returns the introspection response per {{RFC7662}} with `actor_receipts` absent or with `actor_receipts_complete: false`; the resource server then applies its local policy to decide whether to accept the token.

The introspection server itself does not return an OAuth error for missing receipts; receipt presence is a property of the introspection response, not a precondition for it.

Consumer use of introspection-returned receipts is described in {{consumer-introspection}}; the registered introspection response members are defined in {{introspection-response-members}}.

## No New Error Codes

This document does not define new OAuth error codes.  The mapping above reuses existing codes from {{RFC8693}}, {{RFC6750}}, and the core actor profile.

# Extensibility {#extensibility}

This profile is designed to compose with sibling companion profiles that build on the OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}}.  Companion profiles have four standard extension surfaces:

*  **New claims inside a receipt JWT** for additional per-hop attributes (for example, historical scope, additional binding data, or extension-specific provenance).  Consumers ignore unrecognized claims under {{receipt-claims}} unless another specification or local agreement defines their meaning, so additive claims do not break the validation rules of this document.
*  **New top-level claims on the outer token, parallel to `actor_receipts`**, for per-hop artifacts that need their own signature semantics (for example, actor-signed proofs whose threat model differs from AS-signed receipts, or recipient-signed acknowledgments).  Profiles that define such claims SHOULD follow the `<name>` plus `<name>_complete` claim-pair convention described in {{discovery-capability-signaling}}.
*  **New JOSE `typ` values** for receipt-shaped artifacts that are not AS-signed receipts conforming to this document.  The `typ` value `actor-receipt+jwt` defined here is reserved for receipts conforming to this document and MUST NOT be used by other artifacts.
*  **New outer-token binding claims**, analogous to `token_id`, that bind a receipt to an outer-token field other than `jti` (for example, a workflow correlation identifier).  Such claims are independently verifiable on the same terms as `token_id`; see {{receipt-to-token-binding-limits}}.

Companion profile authoring rules:

*  Companion profiles MAY extend consumer processing under {{consumer-processing}} by adding rejection conditions; they MUST NOT relax any rejection condition defined here.
*  Companion-profile claims and discovery metadata MUST be registered with IANA in the registries used by this document.
*  Companion profiles MAY reuse the `prh` and `prh_alg` chain-linkage construction defined in {{receipt-claims}} when their per-hop signed artifacts form a similar chain structure, so that recipients can apply a single chain-validation routine across companions.
*  Companions whose artifacts do not form a chain (for example, independent per-hop attestations or recipient acknowledgments that are not linked to one another) MAY define their own integrity structure.

Cross-companion alignment: companion artifacts that need to reference a specific receipt (for example, an actor-signed proof at hop N referencing the corresponding AS-signed receipt at hop N) SHOULD do so by the receipt's `jti`, which is REQUIRED on receipts and unique within the issuer's namespace.  This profile does not define a hop-index claim; cross-companion alignment is established through `jti` reference plus the `prh` chain's structural integrity, not through array-position metadata.

Conflict resolution: when a recipient implements multiple companion profiles whose rules conflict, local policy determines precedence.  Companion profiles SHOULD be designed to add, not contradict, other profiles' rejection conditions, so that conflicts arise only between profiles whose threat models are genuinely incompatible.

# Security Considerations

Actor receipts strengthen provenance for visible actor hops, but they do not replace ordinary token validation.  The general OAuth 2.0 Security Best Current Practice {{RFC9700}} and the JWT best practices in {{RFC8725}} apply to systems implementing this profile.

## Threat Model {#threat-model}

This section consolidates the adversary classes addressed (and not addressed) by this profile.  Detailed mitigations are described in {{trust-in-receipt-issuers}}, {{receipt-to-token-binding-limits}}, {{compromised-outer-issuer}}, and {{subject-re-expression-across-hops}}; this section provides a reviewer's index.

### Adversaries Mitigated by This Profile

*  **Compromised downstream issuer fabricating prior-hop provenance.**  A dishonest issuer that adds a new hop and signs the outer token cannot forge prior issuers' receipt signatures.  The `prh` chain prevents that issuer from dropping or reordering inner receipts.  This is the primary value proposition of the profile.
*  **Token mutation in transit.**  Each receipt is independently signed.  Modification of any receipt invalidates that receipt's signature and any newer receipt's `prh` value.
*  **Receipt transplantation between tokens with matching visible `act` chains.**  The outer token's own signature prevents non-issuer parties from constructing a substitute outer token to host transplanted receipts; this is the primary transplantation defense.  `receipt[0].token_id`, when matching the current outer-token `jti` in the originating-issuance case, provides additional diagnostic confirmation of receipt-to-token binding (see {{receipt-to-token-binding-limits}}); the strict equality check does not extend the threat model beyond the outer-token-signature defense, since outer-token issuer compromise is the threat in {{compromised-outer-issuer}} and is out of scope.
*  **Partial-coverage misclaim.**  An issuer cannot drop an inner receipt without breaking the `prh` chain; coverage is structurally tamper-evident, and `actor_receipts_complete: true` cannot be claimed without a count matching visible chain depth.

### Adversaries NOT Mitigated

*  **Compromised current outer token issuer.**  A compromised issuer can assemble a new outer token wrapping previously harvested valid receipts for the same visible chain prefix.  Defense requires external transparency, transaction binding, or replay detection outside the scope of this document.
*  **Compromised receipt signing key for any one issuer.**  Forged receipts indistinguishable from legitimate ones cannot be revoked individually.  The remediation is removal of the compromised issuer from the trusted-issuer set; short receipt `exp` values bound the exposure window.
*  **Compromised actor at a hop.**  Receipts attest issuer assertions, not actor non-repudiation.  Companion profiles defined under {{extensibility}} can address this with actor-signed proofs.
*  **Cross-namespace subject graft with a compromised upstream issuer.**  An attacker who compromises a single upstream issuer can mint receipts for any subject in that issuer's namespace and graft them onto a re-expressed downstream chain.  Mitigation requires consistent `sub` values across the chain or trusted out-of-band subject mapping (see {{subject-re-expression-across-hops}}).
*  **Replay of an entire token plus its receipts.**  This profile does not define replay detection; receipts inherit the outer token's replay characteristics.

### Trust Model Summary

Recipients establish trust per-issuer and per-deployment.  Trust under this profile is not transitive across the chain: a receipt chain breaks at the first inner receipt whose issuer is not in the recipient's trusted-issuer set, even when the outer token's issuer and earlier receipts are trusted.  Companion profiles built on this document (see {{extensibility}}) can extend the addressed adversary set; for example, an actor-signed-proofs companion can mitigate the compromised-current-outer-token-issuer adversary by requiring an actor-side signature on each hop.

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

Trust under this profile is per-issuer and not transitive: each receipt is validated against the recipient's own trusted-issuer set, regardless of which issuer minted the surrounding outer token or any neighboring receipt.  A receipt chain therefore breaks at the first inner receipt whose issuer is not in the recipient's trust set, even when the outer token's issuer and earlier receipts are trusted.  Recipients SHOULD treat receipts whose issuers are not in the trusted set as if they were absent and apply partial-coverage policy ({{discovery-capability-signaling}}) accordingly.  Deployments that need uniform trust across an extended chain MUST establish trust explicitly with every receipt issuer that may appear in tokens they accept.

## Receipt-to-Token Binding Limits {#receipt-to-token-binding-limits}

Receipts prove that trusted issuers attested particular actor hops and, optionally, historical presenter bindings.  They do not by themselves prove that the current outer token's audience, scope, expiration, or other authorization details were in force when older receipts were created.

Accordingly, a recipient MUST NOT treat a valid receipt chain as evidence of historical authorization scope or audience beyond what the current outer token itself authorizes.

The integrity of the receipt chain rests on two anchors:

*  `receipt[0].token_id`, when present and signed by the same issuer that signed the outer token, binds `receipt[0]` to the specific outer token instance and prevents transplantation of a receipt chain from a different token whose visible `act` structure happens to match.
*  `prh` chains each receipt cryptographically to its older neighbor, so that all inner receipts inherit their binding from `receipt[0]` through the hash chain.

Inner receipts' `token_id` values are not independently verifiable by the consumer (see {{consumer-processing}}); deployments that depend on receipt-chain integrity MUST rely on `prh` plus a verifiable `receipt[0].token_id`, not on inner `token_id` values.

A receipt chain therefore has exactly one anchor to the current outer token (`receipt[0].token_id`, when issuer alignment holds) and exactly one chain of cryptographic linkage (`prh`).  All inner-receipt integrity flows from these two properties.  An inner receipt has no independent binding to the current request, audience, scope, or token instance; it is bound only to its older neighbor through `prh` and ultimately to the current outer token through the chain's anchor.

This construction makes coverage tamper-evident at the structural level:

*  An issuer cannot drop an inner receipt without breaking the `prh` chain: the next-newer receipt's `prh` value would no longer match the receipt now in the next array position, and consumer step 6 of {{consumer-processing}} rejects the chain.
*  An issuer can only fail to include receipts from the outermost end of the chain, producing partial coverage that `actor_receipts_complete: true` then forbids the issuer from claiming.

Coverage is therefore truthful within the limits of the trusted-issuer set: a compromised issuer can omit some or all of its own receipts and any outermost receipts from issuers it controls, but it cannot fabricate, reorder, or selectively drop receipts signed by other trusted issuers.

When `receipt[0]` diverges from the current outer-token instance — either because `receipt[0].iss` differs from `outer.iss`, or because `receipt[0].iss` matches but `receipt[0].token_id` differs from `outer.jti` — the consumer relaxes the `token_id` bind-to-current check ({{consumer-processing}}) on the assumption that the divergence reflects legitimate reissuance per {{reissuance-without-a-new-actor-hop}}.  In this case:

*  Recipients rely on the current outer token issuer's authority to have performed a legitimate reissuance.
*  This profile provides no in-band signal distinguishing legitimate reissuance from a re-wrapping attack by a compromised issuer, and does not define metadata identifying which issuers in a recipient's trust set perform reissuance versus only originating tokens.
*  Recipients that need to distinguish reissuing from originating issuers MUST establish that distinction through local policy or out-of-band trust framework, and SHOULD treat unexpected reissuance divergence as cause for additional scrutiny under that policy.

Because the relaxation applies whenever divergence is observed, the `token_id` strict-equality check provides binding only in the originating-issuance case (same-issuer, same-`jti`).  In all other cases, transplantation defense rests on the outer token's own signature: a party that is not the trusted outer-token issuer cannot construct a substitute outer token to host transplanted receipts, regardless of `token_id`.  An outer token issuer that is compromised falls under the threat in {{compromised-outer-issuer}}, which is explicitly out of scope for this profile.

Companion profiles MAY define additional outer-token binding claims that follow the `token_id` pattern: each such claim records an identifier from the outer token at the time of receipt creation, and consumer verifiability against the current outer token is conditioned on issuer alignment in the same way `token_id` is conditioned on `receipt[0].iss` matching `outer.iss`.  An additional binding claim does not weaken the `token_id` anchor; it provides a parallel anchor against a different outer-token field.

## Hash Algorithm Agility

`prh` defaults to a base64url-encoded `sha-256` hash of the next older receipt.  The `prh_alg` claim ({{receipt-claims}}) signals an alternative hash algorithm by reference to the IANA Named Information Hash Algorithm Registry {{RFC6920}}, without requiring a successor specification.

Algorithm coordination requirements:

*  All receipts in a single chain MUST use the same algorithm.
*  Consumers MUST reject chains that mix algorithms or that name an algorithm the recipient does not support.
*  An issuer extending an inbound chain MUST preserve the inbound `prh_alg`.

Migration semantics: algorithm migration under this profile is whole-chain, not partial.  Chains begun under one algorithm remain on that algorithm for their entire lifetime; new chains can adopt a different algorithm independently.  This profile does not define rehashing of inbound receipts under a new algorithm, because rehashing would invalidate the original signers' `prh` values and require re-signing receipts the extending issuer did not originate.

Deployments planning a migration SHOULD begin issuing new chains under the target algorithm well before any indication that the legacy algorithm is reaching end of life, so that legacy chains expire naturally without forcing partial-migration patterns this profile does not support.

## Compromised Outer Issuer {#compromised-outer-issuer}

Receipts provide their strongest additional assurance against a compromised or dishonest downstream issuer attempting to fabricate prior-hop provenance.  That downstream issuer cannot forge prior issuers' receipt signatures.

However, if the current outer token issuer is compromised, that issuer can still assemble a new outer token around previously harvested valid receipts for the same visible chain prefix.  This document does not attempt to solve that class of attack.  Deployments that need stronger guarantees can combine this profile with additional transparency, transaction binding, or replay-detection mechanisms outside the scope of this document.

## Receipt Signing Key Compromise

If a receipt issuer's signing key is compromised, previously issued receipts signed with that key cannot be individually revoked.  The primary remediation is to remove the compromised issuer from the trusted-issuer set; once removed, consumers will reject all receipts signed by that issuer regardless of their content.

Deployments SHOULD set short `exp` values on receipts, consistent with the REQUIRED `exp` defined in {{receipt-claims}}, to limit the window during which receipts signed with a compromised key remain valid.  When a key compromise is detected, deployments SHOULD treat all tokens carrying receipts from the affected issuer as lacking trusted provenance for those hops and SHOULD require re-issuance through a trusted issuer.

## Receipt Chain Size

Each receipt is a full signed JWT, and the receipt chain grows linearly with delegation depth.  A typical signed receipt is in the 400 to 800 byte range after JWS compact serialization and base64url encoding, depending on signature algorithm and which optional claims are present; receipts that include `cnf` or larger `act` objects sit at the upper end of that range.  Chains beyond approximately 10 hops therefore approach the 8 KB Authorization header budget that is common in HTTP infrastructure, and chains beyond approximately 20 hops approach a 16 KB practical ceiling.  These figures are illustrative and depend on the specific deployment.

Tokens carrying deep chains can exceed HTTP header size limits in proxies, gateways, or downstream services.  Deployments SHOULD verify that the resulting outer token plus its `actor_receipts` array fits within the header size budget of every component on the request path that handles the token.  When introspection is available, deployments MAY return receipts via introspection rather than embedding them in the access token to avoid header pressure for bearer-token clients.

## Historical `cnf` Disclosure {#historical-cnf-disclosure}

Receipt `cnf` values can reveal prior-hop public-key identifiers or certificate thumbprints to any party that receives the token or introspection response.  These are stable identifiers that can enable cross-request and cross-service correlation of actors and services over time.  Issuers SHOULD NOT include `cnf` in receipts unless the relying parties that will receive the token have been evaluated for that disclosure risk and the risk is acceptable.  Omitting `cnf` from a receipt does not invalidate the receipt; it means that hop lacks independently attested historical presenter binding, which is acceptable for many deployments.

# Privacy Considerations

Actor receipts increase delegation transparency, but they also increase information disclosure:

*  they expose which issuers created visible actor hops;
*  they can reveal historical presenter-key identifiers across requests, enabling cross-session correlation;
*  they can reveal internal service identities that a deployment might otherwise have kept visible only to intermediate issuers.

Deployments SHOULD minimize receipt disclosure when full provenance is not required.  In particular:

*  an issuer or introspection server MAY suppress `actor_receipts` entirely when policy does not permit disclosure;
*  an introspection server that returns only partial receipt information SHOULD set `actor_receipts_complete` to `false`;
*  resource servers SHOULD request or require actor receipts only when they materially improve authorization, audit, or risk controls;
*  issuers SHOULD omit `cnf` from receipts by default when the relying parties that will receive the token have not been evaluated for historical presenter-key disclosure risk.

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

*  Claim Name: `prh`
*  Claim Description: Base64url-encoded hash of the immediately preceding (older) receipt in the actor_receipts chain
*  Change Controller: IESG
*  Specification Document(s): This document

*  Claim Name: `prh_alg`
*  Claim Description: Hash algorithm identifier (from the IANA Named Information Hash Algorithm Registry) naming the algorithm used to compute prh
*  Change Controller: IESG
*  Specification Document(s): This document

*  Claim Name: `token_id`
*  Claim Description: The jti of the specific token instance that this actor receipt was created for
*  Change Controller: IESG
*  Specification Document(s): This document

## OAuth Authorization Server Metadata Registration

This document requests registration of the following metadata name in the "OAuth Authorization Server Metadata" registry {{RFC8414}}:

*  Metadata Name: `actor_receipts_supported`
*  Metadata Description: Indicates support for validating, issuing, preserving, or extending actor-receipt chains
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
  "token_id": "d3a1b2c0-9f4e-4a1d-b8e7-12345678abcd"
}
~~~

`actor_receipts[1]` is the older receipt, created by the enterprise AS when it first added the AI agent:

~~~json
{
  "iss": "https://as.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
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
  "token_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
~~~

This example shows the key provenance property of this profile: the current token is bound to `ToolJKT`, while the older receipt preserves that the earlier actor hop was bound to `AgentJKT` when it was created.

## Example: Transaction Token Service Rebinding

Suppose the booking tool exchanges the access token above at a TTS, and the TTS rebinds the issued Transaction Token to an internal workload identified as `https://wimse.travel-provider.example/workloads/payments`.

The resulting Transaction Token can carry:

~~~json
{
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://wimse.travel-provider.example/workloads/payments",
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
    "sub": "https://wimse.travel-provider.example/workloads/payments",
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
  "token_id": "f0e1d2c3-b4a5-6789-cdef-012345678901"
}
~~~

The inherited receipts for the booking tool and the AI agent are carried forward unchanged.

## Example: Partial Receipt Coverage

When receipt support is rolled out progressively across issuers, downstream tokens may carry coverage for only the outermost hops.  Suppose the enterprise AS has not yet deployed receipt support, and the travel-provider AS has.  The enterprise AS issues a delegated token introducing the AI agent without a receipt.  The travel-provider AS exchanges that token, adds the booking tool as the new outermost actor, and creates a single receipt for that hop.

The resulting access token carries:

~~~json
{
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
  "token_id": "d3a1b2c0-9f4e-4a1d-b8e7-12345678abcd"
}
~~~

`prh` is omitted because this is a single-element chain.  `actor_receipts_complete: false` signals to recipients that the inner AI-agent hop is uncovered.  Resource servers that set `actor_receipts_complete_required: true` in their Protected Resource Metadata reject this token; resource servers that accept partial coverage validate the receipt-attested outermost hop and treat the inner hop as carried solely by the visible `act` chain, with no independent receipt-level provenance.

## Example: Reissuance Without a New Actor Hop

Suppose the access token from the Two-Hop Delegation Chain example is introspected by an introspection endpoint operated as a separate trust principal from the originating travel-provider AS, and re-emitted as a JWT for an internal service.  Re-emission does not add a new outermost actor hop; the visible `act` chain is unchanged.  Per {{reissuance-without-a-new-actor-hop}}, the re-emitting issuer carries the inbound `actor_receipts` array forward unchanged and does not create a new receipt.

The re-emitted token's claims:

~~~json
{
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
  "actor_receipts_complete": true,
  "jti": "f4a7b9c2-1d3e-4f5a-8b6c-7d8e9f0a1b2c"
}
~~~

The receipts are bit-identical to those in the Two-Hop Delegation Chain example.  Two divergences from the originating-issuance pattern are visible at the outer-token level:

*  `outer.iss` is `https://introspection.travel-provider.example`, while `receipt[0].iss` remains `https://as.travel-provider.example`.  This divergence is legitimate under {{reissuance-without-a-new-actor-hop}}.
*  `outer.jti` is `f4a7b9c2-1d3e-4f5a-8b6c-7d8e9f0a1b2c`, while `receipt[0].token_id` remains `d3a1b2c0-9f4e-4a1d-b8e7-12345678abcd` (the original outer token's `jti`).  This divergence is also legitimate.

Per consumer step 5 of {{consumer-processing}}, the consumer relaxes the bind-to-current `token_id` check for `receipt[0]` because `receipt[0]` diverges from the current outer-token instance.  This example illustrates the different-issuer pattern (`receipt[0].iss` differs from `outer.iss`); the same relaxation applies to the same-issuer pattern, in which an authorization server refreshes its own token and the new outer-token `jti` differs from `receipt[0].token_id` while `iss` matches.  In both patterns, trust that the divergence reflects legitimate reissuance rather than a re-wrapping attack rests on the recipient's local policy regarding which issuers in its trust set perform reissuance under this profile, as described in {{receipt-to-token-binding-limits}}.

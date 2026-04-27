---
title: "OAuth Actor Receipts for Delegation Provenance"
abbrev: "OAuth Actor Receipts"
category: exp
docname: draft-mcguinness-oauth-actor-receipts-latest
submissiontype: IETF
number:
date: 2026-04-27
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
  RFC6838:
  RFC7515:
  RFC7519:
  RFC7662:
  RFC7800:
  RFC8414:
  RFC8693:
  RFC8725:
  RFC9728:
  I-D.ietf-oauth-transaction-tokens:
  I-D.mcguinness-oauth-actor-profile:
    title: "OAuth Actor Profile for Delegation"
    author:
     -
        fullname: Karl McGuinness
        organization: Independent
    date: 2026-04-27
    target: https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-profile.html

informative:
  RFC6749:
  RFC7638:
  RFC8126:
  RFC9068:
  RFC9449:
  RFC9700:

...

--- abstract

The OAuth Actor Profile for Delegation makes delegated actor identity explicit in OAuth tokens, but it intentionally limits the core profile to current-token semantics.  In particular, the core profile uses the top-level `cnf` claim only for the current presenter and does not standardize signed prior-hop key history inside the `act` chain.

This document defines OAuth Actor Receipts, an optional companion provenance profile for delegated tokens.  It introduces the `actor_receipts` claim, a signed per-hop receipt chain that records which issuer added each visible actor hop, preserves the historical top-level `cnf` value associated with that hop, and links receipts together so recipients can validate prior-hop provenance without relying solely on the current outer token issuer.  This document also defines metadata and introspection parameters for advertising and consuming actor-receipt support.

--- middle

# Introduction

The OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}} makes actor identity visible in delegated tokens by standardizing the `act` claim across JWT assertion grants, JWT access tokens, and Transaction Tokens.  That core profile is intentionally narrow: it defines who the current subject is, who the visible actors are, and how the current presenter proves possession of the key bound in the token's top-level `cnf` claim.  It does not attempt to make prior-hop actor-key history independently verifiable across trust boundaries.

Some deployments need stronger provenance.  A relying party can often see an `act` chain, but without independently signed prior-hop evidence it still depends on the current token issuer to have preserved that chain faithfully.  The problem is sharper when deployments want historical sender-constraint context: carrying key history inline in nested `act` objects makes that history readable, but it does not make it independently trustworthy.

This document defines an optional companion profile, OAuth Actor Receipts, for deployments that need signed prior-hop provenance.  The design center is:

*  keep the visible actor chain in `act`, as defined by the core actor profile;
*  keep active presenter proof of possession in the token's top-level `cnf`;
*  carry prior-hop provenance, including historical top-level `cnf` values, in separately signed hop receipts.

This document applies to tokens that already conform to the OAuth Actor Profile for Delegation.  It supplements that profile; it does not replace it.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Unless otherwise specified, OAuth terms such as client, authorization server, resource server, access token, refresh token, grant, `subject_token`, and `actor_token` are used as defined in {{RFC6749}} and {{RFC8693}}.  Transaction Token and Transaction Token Service (TTS) are used as defined in {{I-D.ietf-oauth-transaction-tokens}}.

The following terms are used in this document:

Actor Receipt:
: A signed JWT that attests one visible actor hop in a delegated token chain.

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
*  support progressive deployment, including tokens with partial receipt coverage.

The non-goals of this document are:

*  replacing the outer token's own signature or issuer trust model;
*  redefining sender-constrained token validation for the current presenter;
*  proving that a particular historical scope, audience, or token lifetime was in force when a receipt was created;
*  defining transparency logs, non-repudiation systems, or public audit infrastructure.

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
: OPTIONAL.  A boolean JWT claim in the outer token.  When `true`, the issuer attests that `actor_receipts` covers every visible hop in the token's `act` chain.  Because this claim is covered by the outer token's signature, consumers can detect if it has been tampered with.  When `actor_receipts_complete` is `true` in the outer token, the consumer MUST verify that the receipt count equals the visible actor-chain depth; if it does not, the consumer MUST reject the token.  Issuers SHOULD set `actor_receipts_complete: true` when they emit complete coverage, to enable consumers to detect chain truncation.

This document does not require every delegated token to carry `actor_receipts`.  A deployment that requires provenance receipts uses local policy or the metadata defined in {{discovery-capability-signaling}} to express that requirement.

# Actor Receipt JWT Format {#actor-receipt-jwt-format}

Each element of `actor_receipts` is a signed JWT represented using JWS compact serialization.

## JOSE Header

The JOSE header of an actor receipt:

*  MUST include an asymmetric digital-signature `alg` value;
*  MUST NOT use `alg: none` or a MAC-based symmetric algorithm;
*  MUST include `typ` with the value `actor-receipt+jwt`;
*  SHOULD include `kid` when the issuer publishes multiple verification keys.

Receipt issuers and consumers MUST apply the JWT best practices in {{RFC8725}}.

## Receipt Claims

The JWT payload of an actor receipt uses the following claims:

`iss`:
: REQUIRED.  The issuer that created and signed the receipt for the corresponding actor hop.  This value identifies the receipt signer.  The `act.iss` value inside the receipt identifies the namespace authority for `act.sub`, using the meaning defined by the core actor profile.  These values MAY be the same entity or different entities.  When they differ, consumers MUST validate trust in the receipt signer and namespace authority according to their respective roles under local policy; the difference alone does not make the receipt invalid under this profile.

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

`cnf`:
: OPTIONAL.  A confirmation claim as defined in {{RFC7800}}.  When present, it MUST equal the top-level `cnf` claim of the token issued at this hop.  This value records historical presenter-binding information for the hop represented by the receipt.  It does not create a current proof-of-possession obligation for the current request.  Issuers SHOULD NOT include `cnf` in a receipt unless the relying parties that will receive the token have been evaluated for the associated disclosure risk; omitting `cnf` does not invalidate the receipt.

`prh`:
: OPTIONAL.  Previous receipt hash.  When present, `prh` MUST be the base64url-encoded SHA-256 hash of the complete compact serialization of the next older receipt in the chain.  The oldest receipt in the chain MUST omit `prh`.

`iat`:
: REQUIRED.  The time at which the receipt was created, as defined in {{RFC7519}}.

`exp`:
: REQUIRED.  Expiration time for the receipt, as defined in {{RFC7519}}.  `exp` MUST be set to a value that covers the expected maximum token lifetime of any token that will carry or inherit this receipt, so that consumer validation of older receipts in a valid chain is not prematurely rejected.  Short `exp` values on receipts limit the window during which a compromised receipt signing key can be exploited.

`jti`:
: REQUIRED.  A unique identifier for the receipt, as defined in {{RFC7519}}.

`token_id`:
: RECOMMENDED.  The `jti` value of the specific outer token that this receipt was created for.  When present, `token_id` binds the receipt to the specific token instance issued at this hop, enabling consumers to detect receipts transplanted from a different token whose visible `act` structure happens to match.  Issuers SHOULD include `token_id` whenever the token they are issuing carries a `jti` claim.

A receipt MAY contain additional claims defined by another specification or by deployment policy.  Consumers MUST ignore unrecognized claims unless another specification or local agreement defines their meaning.

## Receipt-Chain Linkage

When the issuer creates a new receipt and prepends it to an inherited receipt chain:

*  if there is an older receipt immediately following it in the array, the new receipt MUST include `prh`, and that value MUST hash the exact compact JWT string of that next receipt;
*  if the new receipt is the only receipt in the array, it MUST omit `prh`.

No JSON canonicalization is applied.  `prh` hashes the exact compact-serialized JWS string of the next older receipt as carried in the array.  Systems that carry, store, or forward `actor_receipts` arrays MUST preserve each compact JWT string byte-for-byte without parsing, re-serializing, normalizing whitespace, or re-encoding base64url segments.  Any modification to a receipt string — even a semantically equivalent one — invalidates `prh` for any receipt that references it.

# Issuer Processing

This section defines how an authorization server or Transaction Token Service creates, preserves, and extends `actor_receipts`.

## Creating the First Receipt

When an issuer creates a delegated token with a new outermost actor hop and no inbound `actor_receipts` are being preserved, the issuer MAY create a new one-element `actor_receipts` array.

If it does so, the new receipt:

*  MUST describe the new outermost actor hop;
*  MUST set `sub` to the issued token's top-level `sub`;
*  MUST set `act.sub` and `act.iss` to the new outermost actor;
*  MUST copy the issued token's top-level `cnf`, if any, into the receipt `cnf`, subject to the disclosure considerations in {{receipt-claims}};
*  SHOULD set `token_id` to the issued token's `jti`, if the issued token carries a `jti`;
*  MUST omit `prh`.

## Extending an Existing Receipt Chain

When an issuer adds a new outermost actor hop and also preserves an inbound `actor_receipts` array, it:

1.  MUST validate the inbound receipt chain before relying on it or carrying it forward.
2.  MUST preserve each inbound receipt byte-for-byte unchanged.
3.  MUST create exactly one new receipt for the new outermost actor hop.
4.  MUST prepend that new receipt to the inherited array.
5.  MUST set the new receipt's `prh`, if the inherited array is non-empty, to the SHA-256 hash of the exact compact serialization of the receipt that is now at the next array index.

An issuer MUST NOT reserialize, resign, normalize, trim, or otherwise alter a prior receipt.

If inbound receipts fail validation, the issuer MUST NOT propagate them.  It MAY continue without `actor_receipts` only when local policy permits partial coverage; otherwise it MUST fail the request under the error model of the underlying protocol.

## Reissuance Without a New Actor Hop

An issuer that reissues, translates, or introspects and re-emits a token without adding a new outermost actor hop:

*  MAY carry an inbound `actor_receipts` array forward unchanged;
*  MUST NOT create a new receipt;
*  MUST NOT continue to carry an inherited `actor_receipts` array if it cannot preserve the visible hop alignment required by {{consumer-processing}}.

If such an issuer changes the visible outermost actor, it has added a new hop and MUST follow {{extending-an-existing-receipt-chain}}.

A change in the outer token's top-level `cnf` value, by itself, does not invalidate carried-forward receipts.  Receipt `cnf` records the historical presenter binding in effect when that hop was created.  The current token's top-level `cnf` can later change, for example because of key rotation or token reissuance, without requiring a new receipt so long as the visible actor hop itself is unchanged.

## Partial Coverage and Full Coverage

This document permits partial receipt coverage for progressive deployment.  An issuer MAY begin a new receipt chain even when older inner actor hops remain visible but uncovered.

However:

*  a partial chain MUST still cover a contiguous outermost prefix of the visible actor chain;
*  an issuer MUST NOT skip an outer visible hop and receipt only an inner visible hop;
*  when local policy or resource requirements require full provenance, the issuer MUST either emit complete receipt coverage or fail the request under the error model of the underlying protocol.

## Transaction Token Service Rebinding

When a Transaction Token Service establishes a new presenter and makes that presenter the new outermost actor, it follows the same receipt rules as any other issuer that adds a new outermost actor hop:

*  it validates and preserves any inbound `actor_receipts`;
*  it creates one new receipt for the newly established outermost actor;
*  it copies the issued Transaction Token's top-level `cnf`, if any, into the new receipt, subject to the disclosure considerations in {{receipt-claims}}.

This profile does not define additional receipt claims specific to Transaction Tokens.  Any transaction-specific semantics remain governed by the Transaction Token itself and its deployment profile.

# Consumer Processing {#consumer-processing}

An issuer, resource server, or other recipient that relies on `actor_receipts` MUST perform the following steps.

1.  Validate the outer token according to its token type and the core actor profile.
2.  If `actor_receipts` is absent, treat the token as lacking receipt-based provenance.  Whether that is acceptable is determined by local policy or metadata.
3.  Verify that `actor_receipts`, if present, is a non-empty JSON array of strings.
4.  Verify that the number of receipts does not exceed the visible actor-chain depth of the outer token.  If the outer token carries `actor_receipts_complete: true`, verify that the receipt count exactly equals the visible actor-chain depth; if it does not, reject the token.
5.  For each receipt, in array order:
    *  parse the string as a compact JWT;
    *  verify that the receipt issuer is within the recipient's pre-configured trusted-issuer set before performing any network retrieval for that issuer's metadata or keys;
    *  resolve the signing key from the trusted issuer's `jwks_uri` metadata {{RFC8414}} or from local configuration;
    *  validate the JWT signature;
    *  verify that `typ` equals `actor-receipt+jwt`;
    *  verify that the receipt `act` object is single-hop, contains no nested `act`, and contains no `cnf`;
    *  enforce `exp`, `iat`, and other JWT validity rules.  Because `exp` is REQUIRED on receipts and MUST cover the expected outer token lifetime, an expired receipt SHOULD be treated as invalid even for older hops.  Local policy MAY permit continued use of a receipt that is expired by a small clock-skew margin, but MUST NOT relax `exp` enforcement broadly as a workaround for issuers that failed to set adequate `exp` values.
    *  if `token_id` is present in the receipt, verify that it equals the `jti` of the outer token at the corresponding hop position.
6.  Verify receipt-chain linkage:
    *  each receipt other than the oldest MUST include `prh`;
    *  each non-oldest receipt's `prh` MUST hash the next older receipt;
    *  the oldest receipt MUST omit `prh`.
7.  Verify visible-hop alignment:
    *  `receipt[0].act.sub` MUST equal the outer token's `act.sub`, and `receipt[0].act.iss` MUST equal the outer token's `act.iss`;
    *  `receipt[1].act.sub` MUST equal the outer token's `act.act.sub`, and `receipt[1].act.iss` MUST equal the outer token's `act.act.iss`;
    *  and so on for the number of receipts present;
    *  when `act.sub_profile` is present in the receipt `act` object, the corresponding visible `act` object MUST contain `act.sub_profile` with the same value;
    *  when `act.sub_profile` is present only in the visible `act` object, the receipt remains aligned for this profile.  The visible value is not independently attested by that receipt, and recipients that require receipt coverage for actor classification MUST reject the receipt chain or apply explicit local mapping rules.
8.  Verify that `receipt[0].sub` equals the outer token's top-level `sub`.
9.  Treat each receipt `cnf` value, if present, only as historical provenance for that hop.  A mismatch between the current outer token's top-level `cnf` and the outermost receipt `cnf` MUST NOT by itself invalidate the receipt chain under this profile.
10.  Receipt `cnf` values MUST NOT replace validation of the current request against the outer token's top-level `cnf`.

If any required check fails, the recipient MUST reject the receipt chain for the purposes of this profile and MUST apply the underlying protocol's error handling for the stage at which the failure occurred.

## Subject Re-Expression Across Hops

Older receipts can carry a different `sub` value from the current outer token when the subject has been re-expressed across issuer namespaces.  This document does not define a universal subject-mapping algorithm.

Accordingly:

*  only `receipt[0].sub` is required to equal the current outer token `sub`;
*  older receipt `sub` values MAY differ;
*  a recipient that applies stronger continuity requirements across older `sub` values MUST do so under explicit trusted local mapping rules.

Recipients MUST be aware that permitting differing `sub` values across receipts creates a cross-subject insertion risk: a receipt from an unrelated subject chain that happens to share the same actor identity could satisfy the structural hop-alignment check.  Deployments where subject continuity is a security requirement SHOULD require consistent `sub` values across all receipts in the chain, or enforce explicit trusted subject-mapping rules that can positively confirm each distinct `sub` value refers to the same underlying entity.  When neither condition is met, the recipient MUST treat the differing `sub` values as unverified subject continuity and MUST NOT rely on those older receipts for authorization decisions.

## Complete Receipt Coverage

A recipient determines complete receipt coverage by comparing receipt count with visible actor depth.  If the number of receipts equals the visible actor depth and all validation rules above succeed, the token has complete receipt coverage for the visible chain.

If local policy or resource requirements require full provenance, the recipient MUST reject tokens that do not have complete receipt coverage.

## Use by Resource Servers

Resource servers can use validated actor receipts as provenance input for authorization, diagnostics, and audit.  However, a valid receipt chain:

*  proves only that trusted issuers attested specific visible actor hops;
*  does not prove that the current token's audience, scope, or expiration were in force when older receipts were created;
*  does not replace the need to authorize the current token itself.

## Introspection

When an authorization server returns actor-receipt information in an OAuth Token Introspection response {{RFC7662}}, it:

*  MAY return `actor_receipts` using the same array format defined in {{actor-receipts-claim}};
*  MAY return `actor_receipts_complete` to indicate whether the returned array provides complete coverage for the visible chain as known to the introspection server.

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
: OPTIONAL.  A boolean.  When `true`, the resource server indicates that it requires complete receipt coverage — the receipt count must equal the visible actor-chain depth and `actor_receipts_complete` must be `true` in the outer token or the introspection response.  This parameter refines `actor_receipts_required`; a resource server SHOULD NOT set `actor_receipts_complete_required: true` without also setting `actor_receipts_required: true`.  When `false` or absent, partial receipt coverage is acceptable to the resource server, subject to any further local policy.

## Introspection Response Members

The following members are defined for use in OAuth Token Introspection responses {{RFC7662}}:

`actor_receipts`:
: OPTIONAL.  An array of strings using the same syntax as the JWT claim of the same name.

`actor_receipts_complete`:
: OPTIONAL.  A boolean.  When `true`, the introspection response indicates that the returned `actor_receipts` cover every visible hop in the token chain as known to the introspection server.  When `false`, the response indicates that coverage is partial or that the server has filtered or withheld one or more receipts.

# Security Considerations

Actor receipts strengthen provenance for visible actor hops, but they do not replace ordinary token validation.

## Current Presenter Validation

The current request is always validated against the outer token's top-level `cnf`, when present, using the proof mechanism appropriate to the token type and deployment.  Receipt `cnf` values are historical only.  A recipient MUST NOT treat an older receipt `cnf` value as sufficient proof for the current request.

The current top-level `cnf` can differ from the outermost receipt `cnf` after a later reissuance or key rotation that does not add a new actor hop.  That difference does not by itself invalidate the receipt chain under this profile.

## Trust in Receipt Issuers

Receipt validation is meaningful only if the recipient trusts the issuers that signed the receipts.  A recipient MUST establish which issuers it trusts for receipt validation before relying on `actor_receipts`.  Trust MUST be established through explicit pre-configuration, bilateral agreement, federation policy, or another explicit trust framework.  A recipient MUST NOT treat the presence of a syntactically valid signed receipt as sufficient grounds to trust its issuer.

Authorization servers that support this document SHOULD advertise `actor_receipts_supported: true` in their AS metadata {{RFC8414}}.  Consumers SHOULD use this metadata signal as one input to trust establishment, but MUST NOT treat metadata advertisement alone as sufficient grounds to trust a receipt issuer; the issuer must also be within the recipient's configured trust boundary.

To avoid attacker-controlled key resolution, a recipient MUST determine whether a receipt `iss` is within its trusted-issuer set before performing any network retrieval for that issuer's metadata or keys.  A recipient that uses dynamic discovery for receipt validation MUST do so only within an existing trust framework or equivalent local policy that defines which issuers are permitted.

## Receipt-to-Token Binding Limits

Receipts prove that trusted issuers attested particular actor hops and, optionally, historical presenter bindings.  They do not by themselves prove that the current outer token's audience, scope, expiration, or other authorization details were in force when older receipts were created.

Accordingly, a recipient MUST NOT treat a valid receipt chain as evidence of historical authorization scope or audience beyond what the current outer token itself authorizes.

## Compromised Outer Issuer

Receipts provide their strongest additional assurance against a compromised or dishonest downstream issuer attempting to fabricate prior-hop provenance.  That downstream issuer cannot forge prior issuers' receipt signatures.

However, if the current outer token issuer is compromised, that issuer can still assemble a new outer token around previously harvested valid receipts for the same visible chain prefix.  This document does not attempt to solve that class of attack.  Deployments that need stronger guarantees can combine this profile with additional transparency, transaction binding, or replay-detection mechanisms outside the scope of this document.

## Receipt Signing Key Compromise

If a receipt issuer's signing key is compromised, previously issued receipts signed with that key cannot be individually revoked.  The primary remediation is to remove the compromised issuer from the trusted-issuer set; once removed, consumers will reject all receipts signed by that issuer regardless of their content.

Deployments SHOULD set short `exp` values on receipts, consistent with the REQUIRED `exp` defined in {{receipt-claims}}, to limit the window during which receipts signed with a compromised key remain valid.  When a key compromise is detected, deployments SHOULD treat all tokens carrying receipts from the affected issuer as lacking trusted provenance for those hops and SHOULD require re-issuance through a trusted issuer.

## Historical `cnf` Disclosure

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
*  Encoding considerations: 8bit; an actor receipt is a JWS compact-serialized JWT consisting of base64url-encoded segments separated by period (`.`) characters.
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
*  Author: IETF
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
*  Claim Description: Base64url-encoded SHA-256 hash of the immediately preceding (older) receipt in the actor_receipts chain
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

# Examples

The examples in this section show decoded receipt contents.  Real receipts are compact-signed JWT strings carried in the `actor_receipts` array.

## Two-Hop Delegation Chain

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

## Transaction Token Service Rebinding

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

# Acknowledgments

This companion draft builds on the OAuth Actor Profile for Delegation and on prior working-group discussion around delegation transparency, token exchange, Transaction Tokens, and sender-constrained tokens.

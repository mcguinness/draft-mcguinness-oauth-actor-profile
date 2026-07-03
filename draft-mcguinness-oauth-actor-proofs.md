---
title: "OAuth Actor-Signed Hop Proofs"
abbrev: "OAuth Actor Proofs"
category: exp
docname: draft-mcguinness-oauth-actor-proofs-latest
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
 - non-repudiation
 - proof
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-actor-profile"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-proofs.html"

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
  RFC8707:
  RFC8725:
  RFC9449:
  RFC9728:
  I-D.ietf-oauth-transaction-tokens:
  I-D.mcguinness-oauth-actor-profile:
  ACTOR-RECEIPTS:
    title: "OAuth Actor Receipts for Delegation Provenance"
    author:
      -
        fullname: Karl McGuinness
        organization: Independent
    date: 2026-05
    target: "https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-receipts.html"

informative:
  RFC9700:

...

--- abstract

This document defines OAuth Actor-Signed Hop Proofs, an optional companion profile for delegated OAuth tokens that conform to the OAuth Actor Profile for Delegation.  It introduces the `actor_proofs` claim, a signed per-hop proof chain in which the actor added at each visible hop signs its own participation and the target binding it authorized for that hop.  Proofs are linked into a hash chain, are validated against actor verification keys resolved through pre-established trust, and optionally cross-reference sibling actor receipts.  This document also defines a token request parameter for conveying proofs at issuance, and metadata and introspection parameters for advertising and consuming actor-proof support.

--- middle

# Introduction

The OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}} makes actor identity visible in delegated tokens through a common `act` claim.  The OAuth Actor Receipts companion {{ACTOR-RECEIPTS}} adds authorization-server-signed per-hop provenance.  Both are issuer assertions: an authorization server attests that an actor was added at a hop.  Nothing in either profile requires the actor's own cryptographic participation, so a compromised or dishonest issuer can fabricate the participation of an actor that never authorized the delegation.

This document defines OAuth Actor-Signed Hop Proofs, an optional companion profile that adds actor-side evidence.  At each covered hop, the actor being added signs a proof attesting its own participation and the target binding it authorizes; proofs travel with the token, are linked into a hash chain, and are validated against actor verification keys.  The design center is:

*  keep the visible actor chain in `act`;
*  keep authorization-server-signed provenance in `actor_receipts`, when the receipts companion is in use;
*  carry actor-signed participation and hop-time target consent in separately signed hop proofs.

Proofs add a top-level JWT claim, one token request parameter, and a small set of metadata signals on top of the existing OAuth ({{RFC6749}}, {{RFC8693}}) and core-actor-profile trust model; deployments opt in per resource server or per trust domain.  Scope is detailed in [Design Goals and Non-Goals](#design-goals-and-non-goals).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Unless otherwise specified, OAuth terms such as client, authorization server, resource server, access token, refresh token, grant, `subject_token`, and `actor_token` are used as defined in {{RFC6749}} and {{RFC8693}}.  Transaction Token and Transaction Token Service (TTS) are used as defined in {{I-D.ietf-oauth-transaction-tokens}}.  Actor Receipt, Receipt Chain, and Outer Token are used as defined in {{ACTOR-RECEIPTS}}; in this document, Outer Token refers to the token with which a proof chain is associated, whether or not that token also carries receipts.

The following terms are used in this document:

Actor Proof:
: A signed JWT created and signed by the actor added at one visible actor hop, attesting that actor's participation and the target binding it authorized for that hop.

Proof Chain:
: The ordered `actor_proofs` array carried in a token or introspection response.

Actor Signing Key:
: An asymmetric key controlled by an actor and used to sign actor proofs.  This document does not standardize how actor signing keys are established; see [Actor Key Resolution and Trust](#actor-key-resolution).

Actor-Key Source:
: A mechanism, trusted by a recipient under explicit local policy, that resolves an actor identifier pair (`act.iss`, `act.sub`) to one or more actor verification keys.

Target Binding:
: The audience and optional resource constraints that the actor authorized for the token issued at its hop, carried in the proof's `target` claim.  A target binding records hop-time consent; it is not an audience restriction on the proof artifact itself.

Sibling Receipt:
: The actor receipt, if any, created for the same visible actor hop as a proof, under {{ACTOR-RECEIPTS}}.

Complete Proof Coverage:
: A condition in which the number of proofs in `actor_proofs` equals the number of visible actor hops in the token's `act` chain, and every proof aligns with the corresponding visible hop.

Examples in this document are illustrative and omit unrelated claims, signatures, and validation steps that a complete deployment would need.

# Relationship to the Core Actor Profile

This document is an extension of {{I-D.mcguinness-oauth-actor-profile}}.  A token that uses the `actor_proofs` claim defined here:

*  MUST conform to the actor-chain representation rules of the core actor profile;
*  MUST use the top-level `cnf` claim, when present, only for the current token presenter;
*  MUST NOT treat any proof as satisfying a proof-of-possession requirement for the current request.

Actor proofs do not replace the visible `act` chain.  The `act` chain remains the interoperable representation of current delegated identity.  Proofs are an additional evidence layer that can be validated by issuers and recipients that support this profile.

This document does not redefine the request semantics of {{RFC8693}} or any Transaction Token request semantics.  It defines only:

*  the `actor_proofs` claim;
*  the signed JWT format of each proof;
*  the `actor_proof` token request parameter for conveying a proof at issuance;
*  issuer and consumer processing for proofs;
*  associated metadata and introspection parameters.

## Relationship to the Actor Receipts Companion {#relationship-to-receipts}

Proofs and receipts attest the same hops from different signers with different trust anchors: a receipt is signed by the authorization server that added the hop, while a proof is signed by the actor that was added.  The two companions are carried independently.  A token MAY carry either, both, or neither, and recipients select a validation posture by policy:

*  **Receipts-only**: proofs absent or ignored; trust per {{ACTOR-RECEIPTS}}.
*  **Proofs-only**: receipts absent or ignored; trust rests on actor-key resolution and actor signatures.
*  **Belt-and-suspenders**: both validated; the token carries independent issuer-side and actor-side evidence for covered hops, and sibling references bind the two chains together ({{sibling-receipt-issuance}}).

The choice of posture is per-recipient deployment policy; this document does not standardize it.

The two companions remain separate artifacts rather than a single multi-signature artifact because their signers, adoption prerequisites, and threat models differ: receipts require only issuer-side implementation, while proofs additionally require actor signing keys and recipient-side actor-key resolution.  A JWS structure carrying both signatures over one payload (JWS JSON Serialization, {{RFC7515}} Section 7.2) would express the sibling relationship more directly but is poorly supported in common OAuth token processing libraries; two parallel arrays of compact-serialized JWTs compose with existing tooling.

The receipts companion's discussion of receipt-based provenance versus token introspection applies equally to proofs; see {{ACTOR-RECEIPTS}}.

# Design Goals and Non-Goals

The goals of this document are:

*  carry actor-signed participation evidence for visible actor hops, independently verifiable against actor keys rather than issuer keys;
*  record the target binding the actor authorized at each covered hop, and prevent the issuing authorization server from issuing beyond that binding at the covered hop;
*  allow downstream recipients to validate actor participation through actor-key sources established independently of the outer token issuer;
*  compose with the actor receipts companion without requiring it;
*  add evidence through additive top-level claims, one additive token request parameter, and metadata signals, with no changes required of deployments that do not use proofs;
*  support progressive deployment, including tokens with partial proof coverage.

The non-goals of this document are:

*  establishing, distributing, or rotating actor signing keys; this document profiles how recipients resolve keys through pre-established trust, not how that trust is created;
*  action-level or scope-level consent semantics; the target binding operates at audience and resource granularity;
*  proof revocation or online freshness signals;
*  replay detection beyond the outer token's own replay characteristics;
*  multi-actor co-signed hops;
*  actor-side events between hops, such as re-consent to a broader target without a new actor hop (see {{extensibility}});
*  reconciling subject identifiers that differ across proofs (see {{subject-re-expression-across-hops}});
*  transparency logging of proofs.

## Deployment Fit

This profile's value is concentrated in deployments where actors hold signing keys: AI agents, workloads, and services with their own credentials.  Where actors cannot sign, actor receipts {{ACTOR-RECEIPTS}} remain the available provenance layer, and this profile adds nothing.  The two companions serve complementary deployment populations and are strongest together.

Proofs shift trust configuration from per-issuer enumeration to per-actor key resolution.  This is a different burden, not a smaller one: the population of actors is typically larger than the population of issuers, and a recipient must establish a trusted actor-key source for every actor whose proofs it relies on ({{actor-key-resolution}}).

The anti-fabrication property of this profile is conditional on recipients requiring proofs.  An issuer that fabricates actor participation simply omits proofs; recipients that accept proof-less delegated tokens receive no protection from this profile ({{downgrade-by-omission}}).

# Actor Proofs Overview

An actor proof records one actor hop from the actor's side.  The actor being added at a hop signs a proof naming itself, the subject on whose behalf it acts, and the target binding it authorizes for the token issued at that hop.  The issuing authorization server validates the proof against the actor's verification key and against the token it is about to issue, then embeds the proof in the issued token.

The token then carries an `actor_proofs` array:

*  one array entry per covered actor hop;
*  newest proof first;
*  older proofs preserved unchanged.

This ordering aligns directly with the visible `act` chain in the outer token.  `actor_proofs[0]` corresponds to the outermost `act` object, `actor_proofs[1]` corresponds to `act.act`, and so on.  It also aligns index-for-index with the `actor_receipts` array when the receipts companion is in use, because both arrays are contiguous outermost prefixes of the same visible chain.

Proofs can cover either:

*  the full visible chain; or
*  a contiguous outermost prefix of the visible chain.

When a deployment requires full actor-side evidence, local policy or resource requirements enforce complete proof coverage.

Proofs are participation and consent evidence, not authority transfer.  A valid proof chain documents that each covered actor signed its participation and its hop-time target binding, but does not by itself convey authority, authorization, entitlement, or delegation rights.  Validation of a proof does not imply that the represented delegation remains active, authorized, or acceptable under current policy; current authorization decisions MUST evaluate the current outer token, current policy, and current state, not the proof chain alone.

# The `actor_proofs` Claim {#actor-proofs-claim}

`actor_proofs` is a new top-level JWT claim for tokens that conform to the core actor profile and this companion profile.

`actor_proofs`:
: OPTIONAL.  An array of strings.  Each string MUST be the compact serialization of a signed JWT proof as defined in {{actor-proof-jwt-format}}.  When present, the array:

  *  MUST NOT be empty; issuers MUST omit the claim rather than including an empty array;
  *  MUST be ordered from newest covered hop to oldest covered hop;
  *  MUST NOT contain more entries than the visible actor-chain depth of the token's `act` claim;
  *  MUST represent a contiguous outermost prefix of the visible `act` chain.

If a token carries `actor_proofs`, it MUST also carry an `act` claim conforming to the core actor profile.

`actor_proofs_complete`:
: OPTIONAL.  A boolean JWT claim in the outer token.  When `true`, the issuer attests that `actor_proofs` covers every visible hop in the token's `act` chain.

  The attestation is relative to the visible chain at issuance time; it does not attest that the visible chain is itself unfiltered (see `chain_complete` in the core actor profile {{I-D.mcguinness-oauth-actor-profile}}).  Consumer enforcement, including the count-equality check, is defined in step 4 of {{consumer-processing}}.

  Issuers SHOULD set `actor_proofs_complete: true` when they emit complete coverage, to enable consumers to detect chain truncation.  Issuers SHOULD set `actor_proofs_complete: false` when they emit partial coverage.  Omitting the claim is observationally equivalent to `false` for consumers that test only for the literal value `true`, but does not provide a positive attestation that coverage is partial.

This document does not require every delegated token to carry `actor_proofs`.  A deployment that requires actor-signed evidence uses local policy or the metadata defined in {{discovery-capability-signaling}} to express that requirement.

# Actor Proof JWT Format {#actor-proof-jwt-format}

Each element of `actor_proofs` is a signed JWT represented using JWS compact serialization {{RFC7515}}.

## JOSE Header

The JOSE header of an actor proof:

*  MUST include an asymmetric digital-signature `alg` value;
*  MUST NOT use `alg: none` or a MAC-based symmetric algorithm;
*  MUST include `typ` with the value `actor-proof+jwt`;
*  SHOULD include `kid` when the actor's key source publishes multiple verification keys;
*  MAY include `crit` per {{RFC7515}}; consumers MUST reject a proof whose `crit` header lists an extension header the consumer does not understand.

Actors, issuers, and consumers MUST apply the JWT best practices in {{RFC8725}}.

## Proof Claims {#proof-claims}

The JWT payload of an actor proof uses the claims defined below, grouped by purpose.

### Identity Claims

`iss`:
: REQUIRED.  The identifier of the actor that signed the proof.  It MUST equal the proof's `act.sub` value.

  In contrast to actor receipts, where `iss` identifies the attesting authorization server, the proof `iss` is the actor itself.  The value is interpreted under the namespace authority named by the proof's `act.iss`, exactly as `act.sub` is interpreted under `act.iss` in the core actor profile.  A bare proof `iss` value MUST NOT be used as a key-resolution or trust index on its own; recipients resolve keys and evaluate trust using the (`act.iss`, `act.sub`) pair ({{actor-key-resolution}}).

`sub`:
: REQUIRED.  The subject identifier on whose behalf the actor authorized the delegation, as known to the actor at signing time.  `actor_proofs[0].sub` MUST equal the outer token's top-level `sub`.  Older proofs MAY carry differing `sub` values when the subject has been re-expressed across issuer namespaces (see {{subject-re-expression-across-hops}}).

`sub_iss`:
: OPTIONAL.  The namespace authority under which the proof `sub` value is interpreted, with the semantics defined for the `sub_iss` claim in {{ACTOR-RECEIPTS}}.  When absent, the subject namespace authority is not independently expressed by this profile and MUST be determined, if needed, from trusted local context for the represented hop.

`act`:
: REQUIRED.  A single-hop actor object identifying the signing actor.  This object:

  *  MUST conform to the core actor profile's actor-object rules;
  *  MUST include `act.sub` and `act.iss`;
  *  MAY include `act.sub_profile`;
  *  MUST NOT contain `cnf`;
  *  MUST NOT contain a nested `act`.

  The `act` object is deliberately redundant with the proof `iss`: it carries the `act.iss` namespace context that a bare `iss` string lacks, and it aligns the proof with the corresponding visible `act` chain entry using the same structural rules as receipt `act` objects.  A proof whose `iss` does not equal its `act.sub` is invalid under this profile.

This profile deliberately defines no proof counterpart to the receipt `sub_profile` claim of {{ACTOR-RECEIPTS}}: subject classification is asserted by issuers rather than by acting parties, so an actor-signed copy would add no actor-attested information.  Actor classification travels in the proof's `act.sub_profile` when present.

### Target Binding

`target`:
: REQUIRED.  A JSON object recording the target binding the actor authorized for the token issued at this hop.  Members:

  `target.aud`:
  : REQUIRED.  A string or array of strings.  The audiences the actor authorizes for the token issued at this hop.

  `target.resource`:
  : OPTIONAL.  An array of URIs with the semantics of the `resource` request parameter of {{RFC8707}}.  When present, it narrows the target binding beyond `target.aud`.

  A target binding records hop-time consent for the hop the proof covers.  Issuer-side enforcement at the covered hop is defined in {{accepting-a-proof}}; consumer evaluation, including the distinction between the newest proof and older proofs, is defined in step 9 of {{consumer-processing}}.

  Extension members MAY be defined by other specifications.  An extension member MUST be defined with constraining semantics only: its presence narrows what the actor authorized and its absence leaves the binding as expressed by the defined members.  Consumers MUST ignore unrecognized `target` members unless another specification or local agreement defines their meaning; actors MUST NOT rely on unrecognized extension members being enforced.

This document deliberately defines the target binding as a first-class `target` claim rather than reusing the JWT `aud` claim.  The binding describes the audiences of the token issued at the hop, not the recipients of the proof artifact itself, and reusing `aud` would cause generic JWT audience validation ({{RFC7519}} Section 4.1.3) to misfire on older proofs whose hop-time audiences legitimately differ from the current recipient.

### Chain Linkage

`prh`:
: OPTIONAL.  Previous proof hash.  When present, `prh` MUST be the base64url encoding without padding ({{RFC7515}}) of the hash of the ASCII octets of the complete compact serialization of the next older proof in the chain, computed using the algorithm identified by `prh_alg` (defaulting to SHA-256 when `prh_alg` is absent).  The oldest proof in the chain, including a single-element chain in which the sole proof is both newest and oldest, MUST omit `prh`.

  The `prh` and `prh_alg` claims are reused from {{ACTOR-RECEIPTS}} with the same construction, applied to proof JWTs.  The proof chain is linked independently of any receipt chain carried in the same token: each companion's `prh` values hash that companion's own artifacts.

`prh_alg`:
: OPTIONAL.  Hash algorithm identifier naming the algorithm used to compute `prh`.

  *  Values MUST be drawn from the IANA "Named Information Hash Algorithm Registry" {{RFC6920}}, which uses lowercase forms such as `sha-256`, `sha-384`, and `sha-512`.
  *  When absent, the default is `sha-256`.
  *  When present, the value MUST identify a hash algorithm whose collision and preimage resistance is at least equivalent to `sha-256`.
  *  All proofs in a single `actor_proofs` array MUST use the same `prh_alg` value, so that recipients can validate the chain without per-proof algorithm negotiation.  This uniformity rule is deliberately syntactic: a chain in which some proofs omit `prh_alg` and others carry an explicit `sha-256` is rejected even though the algorithm is the same, so that consumers never reconcile explicit values against defaults.  A chain that omits `prh_alg` from every proof uses the SHA-256 default for every `prh` value.  A single-element chain MAY carry `prh_alg`, but the value has no effect unless a later proof links to it.
  *  An issuer extending an inbound chain MUST either preserve the inbound `prh_alg` or reject the chain.
  *  The proof chain's `prh_alg` is independent of the receipt chain's `prh_alg` in the same token; the two chains MAY use different algorithms.

### Sibling Receipt Reference

`receipt_jti`:
: OPTIONAL.  The `jti` of the sibling receipt created for the same hop under {{ACTOR-RECEIPTS}}.

  Because the actor signs the proof before the issuer signs the sibling receipt, this claim can be populated only when the issuance flow provides the prospective receipt identifier to the actor before signing (for example, in a challenge step).  Most deployments will omit it and rely instead on the issuer-populated `proof_jti` receipt claim defined in {{sibling-receipt-issuance}}, which binds in the feasible direction.  Consumer verification of sibling references is defined in step 10 of {{consumer-processing}}.

### Time and Uniqueness

`iat`:
: REQUIRED.  The time at which the proof was signed, as defined in {{RFC7519}}.

`exp`:
: REQUIRED.  Expiration time for the proof, as defined in {{RFC7519}}.

  *  `exp` MUST be set to a value that covers the expected maximum token lifetime of any token that will carry or inherit this proof, so that consumer validation of older proofs in a valid chain is not prematurely rejected.
  *  Actors SHOULD set `exp` to the maximum delegated-token lifetime permitted under local policy for tokens that may inherit this proof.

  Downstream issuers reject inbound proofs whose `exp` precedes the issued outer token's `exp` ({{extending-an-existing-proof-chain}}), so an under-set `exp` causes propagation failure rather than mid-token-lifetime consumer rejection.  Short `exp` values bound two exposure windows: the window during which a compromised actor signing key can be exploited, and the window during which a party holding a previously valid proof can re-embed it in another token ({{proof-to-token-binding-limits}}).  Deployments typically derive `exp` from a bounded delegated-session lifetime coordinated across the trust set.

`jti`:
: REQUIRED.  A unique identifier for the proof, as defined in {{RFC7519}}.  Recipients and auditors can use `jti` uniqueness across observed tokens to detect proof re-embedding ({{proof-to-token-binding-limits}}).

### Outer-Token Binding

`origin_jti`:
: OPTIONAL.  The `jti` of the outer token issued at the hop this proof covers, following the pattern of the `origin_jti` claim defined in {{ACTOR-RECEIPTS}}.

  Because the actor signs the proof before the outer token exists, this claim can be populated only when the issuance flow provides the prospective outer-token `jti` to the actor before signing (for example, in a challenge step).  When `actor_proofs[0].origin_jti` is present and equals the outer token's `jti`, it binds the proof chain to the current outer-token instance.  When absent, the proof chain carries no instance binding of its own; deployments that require instance binding obtain it through composition with receipts ({{proof-to-token-binding-limits}}).  Consumer evaluation is defined in step 9 of {{consumer-processing}}.

### Excluded Standard Claims

`aud`:
: NOT RECOMMENDED.  Actors SHOULD omit `aud` from proofs.

  Proofs are validated as part of outer-token processing, not as independent JWTs against an audience; the outer token carries the audience scoping for the request, and the actor's consented audiences live in `target.aud`.  This profile diverges from the audience-validation guidance in {{RFC8725}} Section 3.9 on those grounds.  Including `aud` in a proof would create ambiguity between an audience restriction on the proof artifact and the target binding, which are different statements.

### Extension Claims

A proof MAY contain additional claims defined by another specification or by deployment policy.  Consumers MUST ignore unrecognized claims unless another specification or local agreement defines their meaning.

## Proof-Chain Linkage {#proof-chain-linkage}

When an actor signs a new proof that extends an inherited proof chain:

*  if there is an older proof immediately following it in the array, the new proof MUST include `prh`, and that value MUST be the base64url encoding without padding of the hash of the ASCII octets of the exact compact JWT string of that next proof;
*  if the new proof is the only proof in the array, it MUST omit `prh`.

No JSON {{RFC8259}} canonicalization is applied.  `prh` hashes the ASCII octets of the exact compact-serialized JWS string of the next older proof as carried in the array, and the hash output is base64url-encoded without padding.  Systems that carry, store, or forward `actor_proofs` arrays MUST preserve each compact JWT string byte-for-byte; any modification, including semantically equivalent re-encoding, invalidates `prh` for any proof that references it.

# Conveying Proofs at Issuance {#actor-proof-parameter}

This document defines one token request parameter:

`actor_proof`:
: OPTIONAL.  The compact serialization of a single actor proof JWT for the new outermost actor hop of the requested token.  A token request MUST NOT include more than one `actor_proof` parameter.

The parameter is defined for token endpoint requests that produce delegated tokens under the core actor profile, including OAuth 2.0 Token Exchange {{RFC8693}} requests and JWT assertion grants.  Transaction Token Service deployments convey the proof equivalently in the Transaction Token request, subject to {{I-D.ietf-oauth-transaction-tokens}}.

The proof is consent and participation evidence, not an authentication credential.  It is not an `actor_token`, and its presence does not alter how the authorization server authenticates the requester or derives actor identity under the core actor profile.  The authorization server authenticates the actor through the mechanisms the core actor profile already defines and separately validates the proof as described in {{accepting-a-proof}}.

This document does not define a challenge mechanism by which an authorization server provides prospective values (such as the outer token's `jti`, a receipt's `jti`, or the newest inbound proof for opaque inbound tokens) to the actor before signing.  Deployments and companion profiles MAY define such mechanisms; the `origin_jti` and `receipt_jti` claims are the designed insertion points.

# Issuer Processing

This section defines how an authorization server or Transaction Token Service accepts, validates, embeds, preserves, and extends `actor_proofs`.

## Accepting a Proof for a New Actor Hop {#accepting-a-proof}

When an issuer adds a new outermost actor hop and the token request carries `actor_proof`, the issuer:

1.  MUST validate the proof's structure per {{actor-proof-jwt-format}}: `typ` value, asymmetric `alg`, presence and JSON types of the REQUIRED claims `iss`, `sub`, `act`, `target` (including `target.aud`), `iat`, `exp`, and `jti`, and the single-hop `act` rules.
2.  MUST verify that the proof's (`act.iss`, `act.sub`) pair equals the actor identifier pair the issuer will emit as the new outermost visible `act` object, and that the proof `iss` equals the proof `act.sub`.
3.  MUST verify that the proof `sub` equals the top-level `sub` of the token being issued.  An issuer that re-expresses the subject at this hop MUST NOT embed the proof; re-expression breaks the alignment between `actor_proofs[0].sub` and the outer token's top-level `sub` that consumers verify under {{consumer-processing}}.
4.  MUST resolve the actor's verification key through an actor-key source trusted under the issuer's local policy and validate the proof's signature ({{actor-key-resolution}}).
5.  MUST verify that the proof's `exp` is no earlier than the issued outer token's `exp`, and that `iat` is plausible under the issuer's clock-skew policy.
6.  MUST NOT issue an outer token whose `aud`, or whose effective resource indicators when the request expresses them, exceed the proof's target binding.  Every audience of the issued token MUST be present in `target.aud`, and every effective resource indicator MUST be within `target.resource` when that member is present.  For Token Exchange requests, an issuer that cannot satisfy the requested target within the proof's target binding SHOULD reject with `invalid_target` per {{RFC8693}} Section 2.2.2.
7.  MUST include the validated proof as `actor_proofs[0]` of the issued token, subject to the chain rules below.

When no inbound `actor_proofs` are being preserved, the proof starts a new chain and MUST omit `prh`.  When the one-element array covers every visible hop (a visible `act` chain of depth 1), the issuer SHOULD set `actor_proofs_complete: true`; when inner visible hops remain uncovered, it SHOULD set `actor_proofs_complete: false`, per {{actor-proofs-claim}}.

If proof validation fails, the issuer MUST NOT embed the proof.  When local policy or the deployment's resource requirements require actor-signed evidence for the issuance, the issuer MUST fail the request under the error model of {{error-handling}}; otherwise it MAY issue the token without `actor_proofs`.

## Extending an Existing Proof Chain {#extending-an-existing-proof-chain}

When an issuer adds a new outermost actor hop and also preserves an inbound `actor_proofs` array, it:

1.  MUST validate the inbound proof chain by applying the consumer processing rules in {{consumer-processing}} before relying on it or carrying it forward.
2.  MUST verify that each inbound proof's `exp` is no earlier than the issued outer token's `exp`.  An inbound proof that fails this check is treated as failing validation under step 1.  Issuers MAY apply a small clock-skew margin to this comparison, consistent with the consumer-side skew tolerance in {{consumer-processing}}, but MUST NOT broadly accept inbound proofs whose `exp` precedes the issued outer token's `exp` by more than a deployment-defined skew bound.
3.  MUST preserve each inbound proof byte-for-byte unchanged.
4.  MUST accept exactly one new proof, conveyed per {{actor-proof-parameter}} and validated per {{accepting-a-proof}}, for the new outermost actor hop.
5.  MUST verify that the new proof's `prh` equals the hash of the exact compact serialization of the inbound array's newest proof, computed using the algorithm named by the inherited `prh_alg` (defaulting to SHA-256 when absent), and MUST verify that the new proof's `prh_alg` matches the inherited chain's value or is omitted when the chain omits it.  An issuer that does not support the inbound `prh_alg` MUST reject the chain rather than rehash; rehashing would invalidate prior actors' signatures.
6.  MUST prepend the new proof to the inherited array.
7.  MUST set the issued outer token's `actor_proofs_complete` to `true` when the validated inbound chain carried `actor_proofs_complete: true` and the new proof covers the new outermost actor hop; downgrading to absent or `false` in that case would falsely signal a coverage reduction to recipients that test for the literal value `true`.  When the issuer cannot attest complete coverage for the issued chain (for example, the inbound chain was partial or carried no completeness attestation), it MUST NOT set `actor_proofs_complete: true` and SHOULD set `actor_proofs_complete: false`, per {{actor-proofs-claim}}.

An issuer MUST NOT reserialize, resign, normalize, trim, or otherwise alter a prior proof.

Chain linkage requires the signing actor to know the exact compact serialization of the newest inbound proof, because the actor computes and signs `prh`.  When the inbound subject token is a JWT, the actor reads `actor_proofs[0]` from the token it presents.  When the inbound subject token is opaque, the deployment MUST convey the newest inbound proof, or its hash and the chain's `prh_alg`, to the actor before signing; when it cannot, the issuer MUST NOT accept a `prh`-less proof as an extension of the inbound chain, and MAY instead start a new chain under {{accepting-a-proof}} where local policy permits partial coverage ({{partial-coverage-and-full-coverage}}).

If inbound proofs fail validation, the issuer MUST NOT propagate them.  It MAY continue without `actor_proofs` only when local policy permits partial or absent coverage; otherwise it MUST fail the request under the error model of the underlying protocol.

## Reissuance Without a New Actor Hop {#reissuance-without-a-new-actor-hop}

An issuer that reissues, translates, or introspects and re-emits a token without adding a new outermost actor hop:

*  MAY carry an inbound `actor_proofs` array forward unchanged;
*  MUST NOT accept or embed a new proof;
*  MUST carry the inbound `actor_proofs_complete` value forward unchanged when the inbound array is carried forward unchanged; an issuer that cannot continue to attest the inbound coverage value MUST drop the inbound array entirely rather than keep the array and silently downgrade `actor_proofs_complete`.  Proof disclosure is all-or-nothing for a given token: a strict subset of an inherited array cannot validate under {{consumer-processing}};
*  MUST NOT continue to carry an inherited `actor_proofs` array if it cannot preserve the visible hop alignment required by {{consumer-processing}};
*  MUST NOT change the outer token's top-level `sub` while carrying inherited proofs forward; a `sub` change re-expresses subject identity and breaks the alignment between `actor_proofs[0].sub` and the outer token's top-level `sub` that consumers verify under {{consumer-processing}}.

If such an issuer changes the visible outermost actor, it has added a new hop and MUST follow {{extending-an-existing-proof-chain}}.

Reissuance interacts with the target binding.  A reissued token whose `aud` or effective resource indicators exceed `actor_proofs[0]`'s target binding is rejected by default-posture recipients under step 9 of {{consumer-processing}}.  An issuer that retargets the audience beyond the newest proof's target binding MUST drop the inherited `actor_proofs` array unless the deployment's recipients are configured to accept reissuance divergence under {{target-binding-strict-mode}}; a retargeted token carrying proofs is interoperable only inside such a deployment.  Reissuance that narrows or preserves the audience within `actor_proofs[0].target` does not disturb the proof chain.

When an issuer drops an inherited `actor_proofs` array while carrying an `actor_receipts` array forward, any `proof_jti` values in the retained receipts ({{sibling-receipt-issuance}}) name proofs that no longer travel with the token.  Those references become informational only; recipients that require bound siblings enforce proof presence through `actor_proofs_required` or local policy, not through receipt-side references alone.

An AS that supports refresh tokens for delegated access tokens carrying proofs:

*  MUST retain the `actor_proofs` array in issuer-controlled state across refresh, either in durable storage (for example, a token-state database or refresh-token state) or embedded in a self-contained refresh token, so each refreshed access token can carry the proofs forward unchanged.
*  MUST rely on proof `exp` values set per {{proof-claims}} to accommodate the bounded maximum delegated-session lifetime.  Otherwise downstream issuers reject inbound chains under {{extending-an-existing-proof-chain}} as proofs approach expiry, and refresh loses actor-signed evidence.
*  When that bounded lifetime would be exceeded, MUST either obtain fresh delegation state with fresh proofs or stop emitting `actor_proofs`, unless local policy permits partial or absent coverage.

## Partial Coverage and Full Coverage {#partial-coverage-and-full-coverage}

This document permits partial proof coverage for progressive deployment.  An issuer MAY begin a new proof chain even when older inner actor hops remain visible but uncovered.

However:

*  a partial chain MUST still cover a contiguous outermost prefix of the visible actor chain;
*  an issuer MUST NOT skip an outer visible hop and carry a proof only for an inner visible hop;
*  when local policy or resource requirements require full actor-signed evidence, the issuer MUST either emit complete proof coverage or fail the request under the error model of the underlying protocol.

When the issuer also filters the visible `act` chain (see the `chain_complete` introspection member defined in the core actor profile {{I-D.mcguinness-oauth-actor-profile}}), `actor_proofs` covers only the visible filtered chain.  In that case `actor_proofs_complete` describes coverage relative to the visible filtered chain, not the unfiltered delegation chain; recipients that need true-chain completeness MUST evaluate `chain_complete` separately.

## Transaction Token Service Rebinding

A Transaction Token Service that establishes a new presenter and makes that presenter the new outermost actor follows the same proof rules as any other issuer that adds a new outermost actor hop, as defined in {{extending-an-existing-proof-chain}} (or {{accepting-a-proof}} when no inbound `actor_proofs` exist).  The new presenter is the signing actor for the new proof; inherited proofs are carried forward unchanged.  This profile does not define additional proof claims specific to Transaction Tokens.

## Sibling Receipt Issuance {#sibling-receipt-issuance}

When a deployment uses both this profile and {{ACTOR-RECEIPTS}}, the issuer that adds a hop creates the receipt and embeds the proof for that hop in the same issuance operation.  The two artifacts are siblings: independent attestations of the same hop by different signers.

This document defines the following extension claim for Actor Receipt JWTs, under the extension-claims rule of {{ACTOR-RECEIPTS}}:

`proof_jti`:
: OPTIONAL.  The `jti` of the proof the receipt issuer validated for the same hop.  When the issuer embeds a proof and creates a sibling receipt for one hop, the receipt SHOULD include `proof_jti` equal to that proof's `jti`.

`proof_jti` binds in the feasible direction: the issuer signs the receipt after the proof exists, so it can always name the proof, whereas the actor can name the receipt in `receipt_jti` only when the receipt identifier was provisioned before signing ({{proof-claims}}).  Because receipts are preserved byte-for-byte, `proof_jti` is fixed at receipt creation and travels with the receipt chain; a later substitution of the proof array for a different harvested proof chain is detectable through the mismatch ({{threat-model}}).

Consumer verification of sibling references is defined in step 10 of {{consumer-processing}}.

# Consumer Processing {#consumer-processing}

An issuer, resource server, or other recipient that relies on `actor_proofs` MUST perform the following steps.

1.  Validate the outer token according to its token type and the core actor profile.
2.  If `actor_proofs` is absent, treat the token as lacking actor-signed evidence.  Whether that is acceptable is determined by local policy or by Protected Resource Metadata signals such as `actor_proofs_required` and `actor_proofs_complete_required` defined in {{discovery-capability-signaling}}.  If `actor_proofs_complete` is present with the value `true` while `actor_proofs` is absent, the combination is malformed; the recipient MUST treat this as a failed required check and apply the rejection rule following step 11.
3.  Verify that `actor_proofs`, if present, is a non-empty JSON array of strings.  Verify that `actor_proofs_complete`, if present, is a JSON boolean.
4.  Verify that the number of proofs does not exceed the visible actor-chain depth of the outer token.  If the outer token carries `actor_proofs_complete: true`, verify that the proof count exactly equals the visible actor-chain depth; if it does not, reject the token.
5.  For each proof, in array order:
    *  parse the string as a compact JWT;
    *  verify that the proof's (`act.iss`, `act.sub`) pair is within the scope of an actor-key source the recipient trusts, before performing any network retrieval keyed by the proof's content;
    *  resolve the actor's verification key from that source ({{actor-key-resolution}});
    *  validate the JWT signature;
    *  verify that the JOSE header uses an asymmetric digital-signature `alg` value accepted for that actor, and reject proofs that use `alg: none` or a MAC-based symmetric algorithm;
    *  verify that `typ` equals `actor-proof+jwt`;
    *  reject a proof whose `crit` header lists an extension header the consumer does not understand;
    *  verify that all REQUIRED proof claims are present and have the expected JSON types, including `iss`, `sub`, `act`, `target` with `target.aud`, `iat`, `exp`, and `jti`;
    *  verify that OPTIONAL claims used by this profile have the expected JSON types when present, including `sub_iss`, `target.resource`, `prh`, `prh_alg`, `receipt_jti`, and `origin_jti`;
    *  verify that the proof `act` object is single-hop, contains no nested `act`, and contains no `cnf`, and that the proof `iss` equals the proof `act.sub`;
    *  enforce `exp`, `iat`, and other JWT validity rules.  Because `exp` is REQUIRED on proofs and MUST cover the expected outer token lifetime, an expired proof SHOULD be treated as invalid even for older hops.  Local policy MAY permit continued use of a proof that is expired by a small clock-skew margin, but MUST NOT relax `exp` enforcement broadly as a workaround for actors that failed to set adequate `exp` values.
6.  Verify proof-chain linkage:
    *  each proof other than the oldest MUST include `prh`;
    *  each non-oldest proof's `prh` MUST hash the next older proof using the algorithm named by `prh_alg`, defaulting to `sha-256` when `prh_alg` is absent;
    *  all proofs in the chain MUST carry the same `prh_alg` value (or all omit it); a mixed-algorithm chain MUST be rejected;
    *  the named algorithm MUST be one the recipient supports; a chain naming an unsupported algorithm MUST be rejected;
    *  the oldest proof MUST omit `prh`.
7.  Verify visible-hop alignment:
    *  `actor_proofs[0].act.sub` MUST equal the outer token's `act.sub`, and `actor_proofs[0].act.iss` MUST equal the outer token's `act.iss`;
    *  `actor_proofs[1].act.sub` MUST equal the outer token's `act.act.sub`, and `actor_proofs[1].act.iss` MUST equal the outer token's `act.act.iss`;
    *  and so on for the number of proofs present;
    *  when `act.sub_profile` is present in the proof `act` object, the corresponding visible `act` object MUST contain `act.sub_profile` with the same value;
    *  when `act.sub_profile` is present only in the visible `act` object, the proof remains aligned for this profile.  The visible value is not attested by the actor, and recipients that require actor-signed evidence for actor classification MUST reject the proof chain or apply explicit local mapping rules.
8.  Verify subject alignment:
    *  `actor_proofs[0].sub` MUST equal the outer token's top-level `sub`;
    *  when `actor_proofs[0].sub_iss` is present and the recipient has a top-level subject namespace authority for the outer token's `sub` from local configuration, an inbound subject token's claims, or another deployment-defined source, the two MUST identify the same namespace authority, evaluated by case-sensitive string comparison; treating lexically distinct identifiers as the same authority requires explicit trusted local mapping rules;
    *  older proofs MAY carry differing `sub` or `sub_iss` values; see {{subject-re-expression-across-hops}}.
9.  Evaluate outer-token binding and target binding:
    *  when `actor_proofs[0].origin_jti` is present and equals the outer token's `jti`, the proof chain is bound to the current outer-token instance; when it is present and differs, the recipient MUST reject the chain unless local policy designates the outer token issuer as a trusted reissuing issuer per {{target-binding-strict-mode}}, in which case the value is historical provenance;
    *  when `actor_proofs[0].origin_jti` is absent, the proof chain carries no instance binding of its own; this is not by itself a validation failure;
    *  verify that every audience of the outer token is present in `actor_proofs[0].target.aud`, and, when the outer token's effective resource indicators are determinable from token claims, the introspection response, or trusted local context, that each is within `actor_proofs[0].target.resource` when that member is present.  A recipient MAY accept a token whose audience or resources exceed the newest proof's target binding only under {{target-binding-strict-mode}}, and MUST then treat the chain as participation evidence only, not as actor consent to the current target;
    *  target bindings of proofs other than `actor_proofs[0]` are historical consent for their own hops.  The recipient MUST NOT evaluate them against the current outer token's audience or resources.
10.  Verify sibling references, when the token also carries `actor_receipts` validated under {{ACTOR-RECEIPTS}}:
     *  for each index i covered by both arrays, when `actor_receipts[i]` carries `proof_jti`, it MUST equal `actor_proofs[i].jti`, and when `actor_proofs[i]` carries `receipt_jti`, it MUST equal `actor_receipts[i].jti`;
     *  a mismatched sibling reference MUST cause the recipient to reject both receipt-based and proof-based provenance for the token;
     *  a sibling reference that names an artifact at an index not covered by the other array is unverifiable; recipients whose policy requires bound siblings MUST reject the token's proof-based provenance, and other recipients MUST treat the reference as informational only;
     *  when receipts are absent or not validated, `receipt_jti` values are informational only.
11.  Apply any additional consumer-processing rules defined by companion profiles whose claims appear in the proof or outer token (see {{extensibility}}).  Companion-profile rules MUST NOT relax any requirement in steps 1 through 10; they MAY add additional rejection conditions.

If any required check fails, the recipient MUST reject the proof chain for the purposes of this profile and MUST apply the underlying protocol's error handling for the stage at which the failure occurred.

A recipient that has rejected a proof chain under this profile MAY, under explicit local policy, extract structural information from the chain for use by companion profiles.  The recipient MUST NOT treat such partial validation as conformance with this profile, and MUST NOT relax the rejection requirements defined above.

## Subject Re-Expression Across Hops {#subject-re-expression-across-hops}

Older proofs can carry a different `sub` value from the current outer token when the subject has been re-expressed across issuer namespaces between hops.  This document does not define a universal subject-mapping algorithm.

Accordingly:

*  only `actor_proofs[0].sub` is required to equal the current outer token `sub`;
*  older proof `sub` values MAY differ;
*  a recipient that applies stronger continuity requirements across older `sub` values MUST do so under explicit trusted local mapping rules.

Recipients MUST be aware that permitting differing `sub` values across proofs creates a cross-subject insertion risk: a proof signed by a legitimate actor for an unrelated subject's delegation could satisfy the structural hop-alignment check when the actor identity at that hop matches.  An attacker who compromises any single actor signing key can deliberately sign proofs naming any subject and any target, and graft them onto a downstream chain whose re-expressed `sub` points at a victim subject.

This profile provides no in-band mechanism for cross-namespace subject reconciliation.

Deployments where subject continuity is a security requirement SHOULD adopt one of the following:

*  require consistent `sub` values across all proofs in the chain, rejecting re-expressed chains; or
*  enforce explicit trusted subject-mapping rules that can positively confirm each distinct `sub` value refers to the same underlying entity.

When neither condition is met, the recipient MUST treat the differing `sub` values as unverified subject continuity and MUST NOT rely on those older proofs for authorization decisions.

## Complete Proof Coverage

A recipient can infer structural complete proof coverage by comparing proof count with visible actor depth.  If the number of proofs equals the visible actor depth and all validation rules above succeed, the token has structurally complete proof coverage for the visible chain.

When local policy only needs structural completeness, that inferred count match is sufficient.  When Protected Resource Metadata declares `actor_proofs_complete_required: true`, the token or introspection response MUST also carry `actor_proofs_complete: true`; a count match without that explicit issuer attestation does not satisfy the metadata-declared requirement.

If local policy or resource requirements require full actor-signed evidence, the recipient MUST reject tokens that do not satisfy the applicable complete-coverage requirement.

## Use by Resource Servers

Resource servers can use validated actor proofs as evidence input for authorization, diagnostics, and audit.  However, a valid proof chain:

*  proves only that the covered actors signed their participation and hop-time target bindings;
*  does not prove that the represented delegation remains active, authorized, or acceptable under current policy;
*  does not prove that any authorization server validated the hop; that attestation is the receipts companion's role;
*  does not replace the need to authorize the current token itself;
*  does not convey authority, authorization, entitlement, or delegation rights;
*  is not actor consent to the current request; it is actor consent to the hop-time issuance within the recorded target binding.

A resource server that bases an authorization decision on proof content alone, without re-evaluating the current outer token and current policy, mis-uses this profile.

## Introspection {#consumer-introspection}

When an authorization server returns actor-proof information in an OAuth Token Introspection response {{RFC7662}}, it:

*  MAY return `actor_proofs` using the same array format defined in {{actor-proofs-claim}};
*  MAY return `actor_proofs_complete` to indicate whether the returned array provides complete coverage for the visible chain as known to the introspection server.

The registered introspection response members are defined in {{introspection-response-members}}; introspection-server failure handling is addressed in {{introspection-errors}}.

Introspection is the primary delivery mechanism for proofs associated with opaque (non-JWT) outer tokens.  Such tokens cannot carry an inline `actor_proofs` claim; the issuer instead retains the proofs in its token store and surfaces them to authorized resource servers via introspection.  The proof format and consumer processing rules above apply unchanged in this case, with the introspection response substituting for the outer token's claim set.

An introspection response that includes `actor_proofs` MUST include the members needed to perform the consumer processing in {{consumer-processing}}: the token's top-level `sub`, the visible `act` chain, the token's `aud`, and the token's `iss`.  When the introspection server maintains a `jti` for the token, the response SHOULD include it so that recipients can evaluate `origin_jti` binding; when `jti` is absent from the response, recipients treat the chain as carrying no instance binding per step 9.  If `actor_proofs_complete` is present, it MUST be a JSON boolean.  A resource server that receives both inline proofs in a JWT token and proofs in an introspection response MUST apply local policy to choose the authoritative source; if both sources are consumed together, mismatched `actor_proofs` or `actor_proofs_complete` values MUST cause the resource server to reject proof-based provenance for the token.

Proof disclosure through introspection is all-or-nothing for a given token: a strict subset of the stored array cannot validate under {{consumer-processing}}, because removing an older proof leaves the next newer proof's `prh` without a target and removing the newest proof breaks visible-hop alignment.  An introspection server that cannot disclose the full stored array for privacy or policy reasons MUST omit `actor_proofs` from the response entirely.  A stored array that itself has partial coverage is returned in full, with `actor_proofs_complete: false`.

When the introspected token is revoked or otherwise inactive, the introspection response follows the core actor profile's suppression rule for delegation claims: an introspection server MUST NOT return `actor_proofs` or `actor_proofs_complete` for a token it reports as inactive.

The core actor profile's `chain_complete` introspection member and `actor_proofs_complete` are distinct signals, exactly as described for receipts in {{ACTOR-RECEIPTS}}: when `chain_complete: false`, proof coverage is complete only for the visible filtered chain, not the full delegation chain.

# Discovery and Capability Signaling {#discovery-capability-signaling}

This section defines metadata for advertising support for actor proofs.  It follows the claim-pair and discovery conventions defined in {{ACTOR-RECEIPTS}}.

## Authorization Server Metadata

The following parameter is defined for use in Authorization Server Metadata {{RFC8414}}:

`actor_proofs_supported`:
: OPTIONAL.  A boolean.  When `true`, the authorization server advertises that it accepts the `actor_proof` token request parameter, validates proofs against actor keys, and embeds, preserves, or extends proof chains according to this document.  This value does not guarantee complete coverage for every visible hop in every resulting token.  When `false` or absent, clients and relying parties MUST NOT assume such support.

This parameter applies equally to an authorization server that issues delegated JWT outputs and to a Transaction Token Service publishing metadata through the same framework.

## Protected Resource Metadata

The following parameters are defined for use in Protected Resource Metadata {{RFC9728}}:

`actor_proofs_required`:
: OPTIONAL.  A boolean.  When `true`, the resource server indicates that delegated requests are expected to carry valid actor proofs covering at minimum the outermost visible actor hop.  When `false` or absent, the resource server makes no metadata declaration about actor-signed evidence requirements.

  Unlike receipt issuance, proof creation involves the actor directly: an actor that can sign proofs MAY use this declaration, together with `actor_proofs_supported` in Authorization Server Metadata, to decide to include `actor_proof` in its token requests.  The declaration also serves deployment coordination and expresses the enforcement posture under which this profile's anti-fabrication property holds ({{downgrade-by-omission}}).

`actor_proofs_complete_required`:
: OPTIONAL.  A boolean.  When `true`, the resource server indicates that it requires complete proof coverage: the proof count must equal the visible actor-chain depth and `actor_proofs_complete` must be `true` in the outer token or the introspection response.  This parameter refines `actor_proofs_required`; a resource server SHOULD NOT set `actor_proofs_complete_required: true` without also setting `actor_proofs_required: true`.  When `false` or absent, partial proof coverage is acceptable to the resource server, subject to any further local policy.

## Introspection Response Members {#introspection-response-members}

The following members are defined for use in OAuth Token Introspection responses {{RFC7662}}:

`actor_proofs`:
: OPTIONAL.  An array of strings using the same syntax as the JWT claim of the same name.

`actor_proofs_complete`:
: OPTIONAL.  A boolean.  When `true`, the introspection response indicates that the returned `actor_proofs` cover every visible hop in the token chain as known to the introspection server.  When `false`, the response indicates that the returned proofs provide only partial coverage of the visible chain.

Consumer use of these members is described in {{consumer-introspection}}; introspection-server failure handling is addressed in {{introspection-errors}}.

## Out-of-Scope Discovery Signals

This document does not define metadata for actor-key source discovery; recipients establish actor-key sources through explicit trust frameworks ({{actor-key-resolution}}), not through metadata defined here.  It also does not define a metadata signal for requiring bound siblings (`proof_jti` on receipts); deployments that need bound siblings coordinate that requirement through deployment policy.

# Error Handling {#error-handling}

This section defines how proof-related processing failures map to OAuth error responses.  Proof validation extends the underlying OAuth or Transaction Token validation rather than replacing it; failures should be reported through the error-response mechanism applicable to the stage at which validation occurred.

## Authorization Server and Transaction Token Service Errors

When an authorization server or Transaction Token Service rejects a token request because an inbound `actor_proofs` chain or a newly submitted proof cannot be validated (signature failure, key-resolution failure for an actor outside the trusted key sources, expired proof, unsupported `prh_alg`, broken `prh` chain, hop or subject misalignment), it SHOULD return `invalid_grant`, constructed per {{RFC8693}} Section 2.2.2 and {{RFC6749}} Section 5.2, consistent with the core actor profile's error mapping for actor information that fails validation.

When the requested token's audience or resources cannot be satisfied within the submitted proof's target binding in a Token Exchange request, the issuer SHOULD return `invalid_target` per {{RFC8693}} Section 2.2.2.

When the failure reflects an actor-authorization decision rather than a structural validation failure, an issuer MAY use `actor_unauthorized` as defined in the core actor profile {{I-D.mcguinness-oauth-actor-profile}} where applicable.

## Resource Server Errors

When a resource server rejects a request because `actor_proofs` validation fails under {{consumer-processing}}, it SHOULD return `invalid_token` per the bearer-token error model in {{RFC6750}} Section 3.1.

When the failure is specifically that required proofs are absent or coverage is incomplete (per `actor_proofs_required` or `actor_proofs_complete_required`), the resource server SHOULD include an `error_description` value identifying proof-coverage failure so that clients and operators can distinguish it from generic token-validation failures.

## Introspection Server Behavior {#introspection-errors}

When an introspection server cannot return proofs that the requesting resource server requires, it returns the introspection response per {{RFC7662}} with `actor_proofs` absent or with `actor_proofs_complete: false`; the resource server then applies its local policy to decide whether to accept the token.

The introspection server itself does not return an OAuth error for missing proofs; proof presence is a property of the introspection response, not a precondition for it.

## No New Error Codes

This document does not define new OAuth error codes.  The mapping above reuses existing codes from {{RFC6749}}, {{RFC6750}}, {{RFC8693}}, and the core actor profile.

# Extensibility {#extensibility}

This profile composes with the extensibility framework defined in {{ACTOR-RECEIPTS}} and adds proof-specific extension surfaces:

*  **New claims inside a proof JWT** for additional per-hop actor-attested attributes.  Consumers ignore unrecognized claims under {{proof-claims}} unless another specification or local agreement defines their meaning.
*  **New `target` extension members** with constraining semantics, per the extension rule in {{proof-claims}}.  Specifications needing actor consent at scope or action granularity extend `target` rather than redefining it.
*  **Challenge and provisioning mechanisms** that supply prospective identifiers (the outer token's `jti`, a receipt's `jti`, or the newest inbound proof for opaque inbound tokens) to the actor before signing.  The `origin_jti` and `receipt_jti` claims are the designed insertion points; such mechanisms strengthen instance binding without changing proof processing.
*  **Actor-side non-hop event artifacts**, such as an actor re-consenting to a different target binding without a new actor hop.  Companion profiles defining such artifacts SHOULD follow the non-hop event artifact pattern of {{ACTOR-RECEIPTS}}: a signed JWT with a `typ` value distinct from `actor-proof+jwt`, carried in a parallel outer-token claim, anchored to a specific proof by the proof's `jti` in a claim the companion defines, or to the delegation flow by a correlation identifier the companion profile specifies.  Companion profiles MUST NOT add event-shaped entries to `actor_proofs`; that array is reserved for per-hop actor-signed proofs defined by this document, and its byte-preservation invariant cannot accommodate entries added after the chain was formed.
*  **Multi-actor co-signed hops** are out of scope for this document and would require a successor or companion profile with its own artifact structure.

Companion profile authoring rules:

*  Companion profiles MAY extend consumer processing under {{consumer-processing}} by adding rejection conditions; they MUST NOT relax any rejection condition defined here.
*  Companion-profile claims and discovery metadata MUST be registered with IANA in the registries used by this document.
*  Companion profiles that define per-hop signed artifacts SHOULD follow the claim-pair and discovery conventions of {{ACTOR-RECEIPTS}}, and MAY reuse the `prh` and `prh_alg` chain-linkage construction.

Conflict resolution: when a recipient implements multiple companion profiles whose rules conflict, local policy determines precedence.  Companion profiles SHOULD be designed to add, not contradict, other profiles' rejection conditions.

# Security Considerations

Actor proofs strengthen delegation evidence with actor-side signatures, but they do not replace ordinary token validation.  The general OAuth 2.0 Security Best Current Practice {{RFC9700}} and the JWT best practices in {{RFC8725}} apply to systems implementing this profile.

## Threat Model {#threat-model}

This section indexes the adversary classes addressed (and not addressed) by this profile.  Detailed mitigations live in {{actor-key-resolution}}, {{proof-to-token-binding-limits}}, {{downgrade-by-omission}}, and {{subject-re-expression-across-hops}}.

### Adversaries Mitigated by This Profile

*  **Current outer-token issuer fabricating actor participation.**  Cannot forge the actor's proof signature at proof-covered hops.  Primary value proposition.  Conditional on two recipient-side requirements: the recipient requires proofs for the tokens it accepts ({{downgrade-by-omission}}), and the recipient resolves the actor's key through a source independent of the issuer being defended against ({{actor-key-resolution}}).
*  **Current issuer exceeding the actor-authorized target at the covered hop.**  Issuer-side enforcement in {{accepting-a-proof}} and consumer step 9 detect an outer token whose audience or resources exceed the newest proof's signed target binding, subject to {{target-binding-strict-mode}}.
*  **Compromised downstream issuer fabricating prior-hop participation.**  Cannot forge prior actors' proof signatures; the `prh` chain prevents dropping or reordering inner proofs.
*  **Token mutation in transit.**  Each proof is independently signed; modification invalidates the proof's signature and any newer proof's `prh`.
*  **Partial-coverage misclaim.**  An issuer cannot drop an inner proof without breaking the `prh` chain; `actor_proofs_complete: true` cannot be claimed without a count matching visible chain depth.  An issuer can withhold coverage only from the innermost end of the chain, and only by beginning a new chain rather than trimming an inherited one, exactly as for receipts.
*  **Proof-chain substitution, when receipts with `proof_jti` are present.**  Replacing the proof array with a different harvested proof chain for the same visible hops mismatches the byte-preserved `proof_jti` values in the receipt chain and is rejected under step 10 of {{consumer-processing}}.

### Adversaries NOT Mitigated

*  **Compromised actor signing key.**  Forged proofs are indistinguishable from legitimate ones and cannot be revoked individually.  Remediation: remove the key or actor from the trusted actor-key sources; short proof `exp` bounds the exposure window.
*  **Issuer omission of proofs.**  An issuer that fabricates participation simply omits `actor_proofs`.  Omission is a downgrade, not merely denial of service; the anti-fabrication property exists only for recipients that require proofs ({{downgrade-by-omission}}).
*  **Proof re-embedding within the validity window.**  A party that received a valid proof, including the issuer it was submitted to, can embed it in a different token with a matching subject, actor chain position, and target within the proof's `exp` window.  See {{proof-to-token-binding-limits}} for the binding limits and mitigations.
*  **Malicious or coerced actor.**  Proofs attest that the actor's key signed the participation; they do not attest intent, and they do not protect against an actor that colludes with a compromised issuer.  When issuer and actor are the same adversary, neither companion detects it.
*  **Cross-subject graft with a compromised actor key.**  Analogous to the receipts-side graft: see {{subject-re-expression-across-hops}}.
*  **Replay of an entire token plus its proofs.**  This profile does not define replay detection; proofs inherit the outer token's replay characteristics.

### Trust Model Summary

Trust is per-actor-key and per-deployment, and not transitive across the chain.  A proof chain breaks at the first proof whose signing key cannot be resolved through an actor-key source the recipient trusts, even when the outer token's issuer and other proofs are trusted.  Proofs and receipts have independent trust anchors; validating both yields evidence that survives compromise of either the issuer side or the actor side, but not simultaneous compromise of both.

## Current Presenter Validation

The current request is always validated against the outer token's top-level `cnf` ({{RFC7800}}), when present, using the proof mechanism appropriate to the token type and deployment, such as DPoP {{RFC9449}} or mutual-TLS {{RFC8705}}.

An actor proof is never a substitute for that validation:

*  A recipient MUST NOT treat a proof signature as satisfying a proof-of-possession requirement for the current request, regardless of whether the proof signing key is the same key as a presenter key.
*  Recipients MUST distinguish proof JWTs (identified by the `typ` value `actor-proof+jwt`) from artifacts that carry current-request proof-of-possession semantics under {{RFC7800}}, {{RFC9449}}, or {{RFC8705}}.

## Actor Key Resolution and Trust {#actor-key-resolution}

Proof validation is meaningful only if the recipient resolves actor verification keys through sources it trusts.

Trust establishment requirements:

*  A recipient MUST establish its trusted actor-key sources before relying on `actor_proofs`.  Trust MUST be established through explicit pre-configuration, bilateral agreement, federation policy, or another explicit trust framework.
*  A recipient MUST NOT treat the presence of a syntactically valid signed proof as sufficient grounds to trust the key that signed it.
*  A recipient MUST determine that a proof's (`act.iss`, `act.sub`) pair is within the scope of a trusted actor-key source before performing any network retrieval keyed by the proof's content, and MUST NOT dereference key references supplied by the proof itself (such as `jku` or `x5u` header parameters) outside a pre-established trust framework, per {{RFC8725}}.
*  Key resolution and trust evaluation use the (`act.iss`, `act.sub`) pair.  The bare proof `iss` string MUST NOT be the sole resolution index; actor identifiers are namespaced by `act.iss`, and identical `act.sub` strings under different namespace authorities are different actors.

This document profiles the following resolution patterns; a deployment may support any subset:

*  **Pre-established keys.**  The actor's verification keys are registered with the recipient or its trust framework in advance, for example as the JWKS of a registered OAuth client at the authorization server, or as locally configured keys at a resource server.  This pattern is self-contained and provides the strongest independence properties.
*  **Receipt-attested presenter keys.**  When the sibling receipt for a hop carries the historical presenter binding `cnf` defined in {{ACTOR-RECEIPTS}}, and the deployment binds proof signing keys to presenter keys, the recipient MAY verify the proof signature against the key identified by the aligned receipt's `cnf`.  This pattern requires no separate key registry, but the key attestation derives from the receipt issuer: proofs verified this way provide no independence from that issuer.  In particular, at the newest hop, whose receipt is signed by the current outer token issuer in the originating-issuance case, receipt-attested key resolution provides no anti-fabrication protection against that issuer.
*  **Federation and workload identity systems.**  Deployment-defined resolution through workload identity or federation infrastructure.  The trust and freshness properties are those of the underlying system; this document does not profile them.

The independence requirement follows from the threat model: for the anti-fabrication property against a given issuer to hold at a hop, the recipient MUST resolve the actor's key for that hop through a source independent of that issuer.

Actor keys, like receipt-issuer trust, are not transitive: each proof is validated against the recipient's own actor-key sources, independent of the outer token's issuer and of neighboring proofs.  If any proof in the presented `actor_proofs` array is signed by a key the recipient cannot resolve through a trusted source, the recipient MUST reject the proof chain for the purposes of this profile.

## Proof-to-Token Binding Limits {#proof-to-token-binding-limits}

A proof is signed before the token that carries it exists.  It therefore names the delegation context (subject, actor, target binding, time window, unique `jti`), not a token instance.  Within a proof's validity window, a party holding the proof, including the issuer it was legitimately submitted to, can embed it in a different outer token with the same subject, the same actor chain position, and a target within the proof's binding, and no signature check detects the reuse.  The actor's non-repudiation statement is scoped accordingly: the actor authorized this delegation relationship toward this target within this window, not this specific token issuance.

Available bindings, strongest first:

*  **Receipts composition.**  When the token also carries receipts, the receipt chain's `origin_jti` anchoring and strict-mode rules in {{ACTOR-RECEIPTS}} bind the token instance, and `proof_jti` ({{sibling-receipt-issuance}}) binds the proof chain to that anchored receipt chain.  A re-embedded proof would require a matching fabricated receipt, which the receipt trust model prevents for issuers that cannot sign trusted receipts.  This is the RECOMMENDED posture for deployments that require instance binding.
*  **Provisioned `origin_jti`.**  When the issuance flow provides the prospective outer-token `jti` to the actor before signing, `actor_proofs[0].origin_jti` binds the proof to that token instance directly, per step 9 of {{consumer-processing}}.
*  **`jti` uniqueness monitoring.**  Recipients and audit pipelines MAY track proof `jti` values and flag the same proof appearing in more than one outer-token instance.  This is stateful and deployment-specific; this document does not define the mechanism.
*  **Short `exp`.**  Bounds the re-embedding window unconditionally, at the cost of shorter delegated-session lifetimes ({{proof-claims}}).

Inner proofs have no independent binding to the current token; they are bound to their newer neighbor through `prh` and inherit whatever binding `actor_proofs[0]` has.

### Target-Binding Strict Mode {#target-binding-strict-mode}

Recipients that have not explicitly configured a set of trusted reissuing issuers operate in strict mode by default: per step 9 of {{consumer-processing}}, an outer token whose audience or effective resources exceed `actor_proofs[0]`'s target binding, or whose `jti` differs from a present `actor_proofs[0].origin_jti`, causes the recipient to reject the proof chain.

Strict mode is the conservative default and the recommended posture absent specific deployment requirements.  Deployments that reissue tokens with retargeted audiences while carrying proofs forward MUST configure permissive validation with an explicit set of trusted reissuing issuers, established through local policy or an out-of-band trust framework.  A recipient operating permissively MUST treat the proof chain as participation evidence only and MUST NOT treat the current audience or resources as actor-authorized.  When receipts are also present, the recipient SHOULD apply a single reissuance-trust decision across both companions rather than accepting divergence for one chain and rejecting it for the other.

## Hash Algorithm Agility

The `prh` and `prh_alg` agility rules of {{ACTOR-RECEIPTS}} apply to proof chains unchanged: one algorithm per chain, whole-chain migration only, no rehashing of inherited artifacts, and rejection of mixed or unsupported algorithms.  The proof chain's algorithm is independent of the receipt chain's algorithm in the same token.

## Downgrade by Omission {#downgrade-by-omission}

This profile's anti-fabrication property is conditional on enforcement.  A compromised issuer that wants to fabricate actor participation does not submit a forged proof, which would fail signature validation; it omits `actor_proofs` entirely.  A recipient that accepts delegated tokens without proofs has no protection from this profile against that issuer.

Accordingly:

*  Resource servers that rely on actor-signed evidence MUST require proofs, through `actor_proofs_required` (and `actor_proofs_complete_required` where inner hops matter) or equivalent local policy, for the delegated tokens they accept.
*  Recipients SHOULD treat the absence of proofs from an issuer that advertises `actor_proofs_supported: true`, for a resource that requires them, as a signal warranting scrutiny rather than silent acceptance.

Issuers cannot protect recipients that do not ask; the enforcement locus of this profile is the recipient.

## Actor Key Compromise

If an actor's signing key is compromised, previously signed proofs and newly forged proofs under that key are indistinguishable.  The primary remediation is to remove the compromised key or actor from the recipient's trusted actor-key sources; once removed, consumers will reject all proofs attributed to that actor's key regardless of content.

Deployments SHOULD set short `exp` values on proofs, consistent with the REQUIRED `exp` defined in {{proof-claims}}, to limit the window during which proofs signed with a compromised key remain valid.  When a key compromise is detected, deployments SHOULD treat tokens carrying proofs from the affected actor as lacking trusted actor-signed evidence for those hops and SHOULD require fresh delegation with fresh proofs.

## Proof Chain Size

Each proof is a full signed JWT, and the chain grows linearly with delegation depth.  A typical signed proof is 400 to 800 bytes after JWS compact serialization and base64url encoding, comparable to a receipt.  A token carrying both companions in belt-and-suspenders mode carries roughly twice the per-hop artifact bytes of either companion alone.  Deployments SHOULD verify that the outer token plus its companion arrays fits within the header-size budget of every component on the request path; the receipts companion's size guidance applies, and introspection delivery avoids header pressure for bearer-token clients.

## Proof Freshness and Replay {#proof-freshness}

Proofs are historical attestations of hop-time consent.  They MAY outlive the validity period of the outer token they were originally embedded in, and MAY be carried forward across reissuance and refresh as long as their `exp` permits.

*  Proofs attest participation and target consent at signing time; they do not assert that the represented delegation is still active or that the actor would consent today.
*  Runtime policy evaluation, including current authorization and current revocation state, is separate from proof validation.
*  Replay of an entire token plus its proofs is governed by the outer token's replay characteristics.  Re-embedding of an individual proof into a different token is a distinct threat, bounded as described in {{proof-to-token-binding-limits}}.

Deployments needing freshness signals beyond proof `exp` MUST obtain those signals from the authorization server via introspection ({{RFC7662}}), fresh token issuance, or another mechanism outside the scope of this profile.

## Sibling Revocation Independence

A proof does not inherit revocation or trust state from its sibling receipt.  When a deployment removes a receipt issuer from its trusted-issuer set under {{ACTOR-RECEIPTS}}, proofs for the same hops remain valid under their own actor keys, and vice versa.  Recipients that require issuer-side revocation semantics for a hop MUST require receipt validation alongside proof validation rather than relying on the proof alone; the sibling references of {{sibling-receipt-issuance}} identify the receipt whose trust state applies to a hop.

# Privacy Considerations

Actor proofs increase delegation transparency in a specific way that receipts do not: they make actor participation provable to third parties, not merely visible.  A signature is transferable evidence; any holder of a proof-bearing token can demonstrate to anyone with access to the actor's public key that the actor participated in the delegation and consented to the recorded target.  Non-repudiation is the purpose of this profile and simultaneously its principal privacy cost.

## What Proofs Disclose

Proofs can expose, to any party that receives the token or introspection response:

*  cryptographically transferable evidence of each covered actor's participation, retained and provable beyond token lifetime;
*  actor key identifiers (`kid` values and resolved public keys), which are stable correlation handles across proofs, flows, and services;
*  target bindings (`target.aud`, `target.resource`), which may reveal internal audience and resource identifiers a deployment would not otherwise expose to all recipients;
*  the delegation graph of a workflow, tied together by `prh` chain hashes;
*  subject re-expression patterns across namespaces, as with receipts.

## Minimization

Deployments SHOULD minimize proof disclosure when actor-signed evidence is not required:

*  Issuers and introspection servers MAY withhold `actor_proofs` entirely when policy does not permit disclosure; a strict subset of an existing array cannot validate ({{consumer-introspection}}), so disclosure of an existing chain is all-or-nothing.
*  Actors SHOULD omit `target.resource` when audience-level consent is sufficient, since resource URIs are often the most deployment-revealing values in a proof.
*  Resource servers SHOULD require actor proofs only when they materially improve authorization, audit, or risk controls.
*  Deployments SHOULD prefer per-resource-server policy on proof requirements over blanket inclusion in every token.
*  Deployments SHOULD evaluate actor key lifetimes with correlation in mind: long-lived actor keys make every proof signed under them linkable.

## Selective Disclosure

This profile does not define a per-claim selective-disclosure mechanism for proofs: chain integrity requires byte-for-byte preservation of each proof JWT, so selective omission of individual claims within a proof would break the chain.  Selective disclosure is coarse-grained, exactly as for receipts: issuance-time partial coverage of outermost hops, or whole-array omission.

## Audience Restriction

A proof travels with the outer token to whichever audiences the outer token serves; proofs have no independent audience scoping ({{proof-claims}}).  Deployments needing audience-specific disclosure constraints SHOULD partition proof issuance by audience at issuance time rather than relying on proof-level audience restriction, which this profile does not provide.

## Detached Provability

Proofs support detached verification by any party that can resolve the actor's public key, including parties the actor and issuer did not anticipate.  Unlike receipts, whose evidentiary weight depends on the verifier trusting the receipt issuer, a proof's signature is self-contained evidence of the actor's participation.  Deployments SHOULD treat proof-bearing tokens as carrying durable, transferable participation evidence to anywhere the token reaches, and SHOULD scope token distribution and retention accordingly.

# IANA Considerations

## Media Type Registration

This document requests registration of the following media type in the "Media Types" registry {{RFC6838}}:

*  Type name: `application`
*  Subtype name: `actor-proof+jwt`
*  Required parameters: N/A
*  Optional parameters: N/A
*  Encoding considerations: 8bit; an actor proof is a JWS compact-serialized JWT {{RFC7515}} {{RFC7519}} consisting of base64url-encoded segments separated by period (`.`) characters.
*  Security considerations: See {{security-considerations}} of this document and {{RFC8725}}.
*  Interoperability considerations: N/A
*  Published specification: This document
*  Applications that use this media type: Applications that create, exchange, or validate OAuth Actor-Signed Hop Proofs.
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

The JOSE `typ` value `actor-proof+jwt` used by this document is the media type subtype name without the `application/` prefix, following common JWT typing practice.

## JSON Web Token Claims Registration

This document requests registration of the following JWT Claims in the "JSON Web Token Claims" registry {{RFC7519}}:

*  Claim Name: `actor_proofs`
*  Claim Description: Array of actor-signed hop proofs providing delegation participation evidence
*  Change Controller: IESG
*  Specification Document(s): This document

*  Claim Name: `actor_proofs_complete`
*  Claim Description: Boolean indicating whether actor_proofs covers every visible hop in the token's act chain
*  Change Controller: IESG
*  Specification Document(s): This document

*  Claim Name: `target`
*  Claim Description: Target binding (audience and resource constraints) authorized by the signer of an Actor Proof JWT
*  Change Controller: IESG
*  Specification Document(s): This document

*  Claim Name: `receipt_jti`
*  Claim Description: jti of the sibling Actor Receipt JWT created for the same delegation hop as an Actor Proof JWT
*  Change Controller: IESG
*  Specification Document(s): This document

*  Claim Name: `proof_jti`
*  Claim Description: jti of the sibling Actor Proof JWT validated for the same delegation hop as an Actor Receipt JWT
*  Change Controller: IESG
*  Specification Document(s): This document

This document reuses the `prh`, `prh_alg`, `origin_jti`, and `sub_iss` claims registered by {{ACTOR-RECEIPTS}}, with the semantics defined there, applied to Actor Proof JWTs as profiled in this document.  This document requests that IANA add this document to the Specification Document(s) entries for those four registrations, and requests that their Claim Description entries be updated to cover both artifact types:

*  `prh`: Base64url-encoded hash of the immediately preceding (older) entry in a chained array of Actor Receipt or Actor Proof JWTs
*  `prh_alg`: Hash algorithm identifier (from the IANA Named Information Hash Algorithm Registry) naming the algorithm used to compute prh in an Actor Receipt or Actor Proof JWT
*  `origin_jti`: The jti of the outer token associated with the hop at which an Actor Receipt or Actor Proof JWT was created
*  `sub_iss`: Issuer or namespace authority for the subject in an Actor Receipt or Actor Proof JWT

## OAuth Parameters Registration

This document requests registration of the following parameter in the "OAuth Parameters" registry established by {{RFC6749}}:

*  Parameter name: `actor_proof`
*  Parameter usage location: token request
*  Change Controller: IESG
*  Specification Document(s): This document

## OAuth Authorization Server Metadata Registration

This document requests registration of the following metadata name in the "OAuth Authorization Server Metadata" registry {{RFC8414}}:

*  Metadata Name: `actor_proofs_supported`
*  Metadata Description: Indicates support for accepting, validating, embedding, preserving, and extending actor-signed hop proofs
*  Change Controller: IESG
*  Specification Document(s): This document

## OAuth Protected Resource Metadata Registration

This document requests registration of the following metadata names in the "OAuth Protected Resource Metadata" registry {{RFC9728}}:

*  Metadata Name: `actor_proofs_required`
*  Metadata Description: Indicates that the resource expects delegated requests to carry valid actor proofs covering at minimum the outermost visible actor hop
*  Change Controller: IESG
*  Specification Document(s): This document

*  Metadata Name: `actor_proofs_complete_required`
*  Metadata Description: Indicates that the resource requires complete proof coverage for all visible actor hops
*  Change Controller: IESG
*  Specification Document(s): This document

## OAuth Token Introspection Response Registration

This document requests registration of the following names in the "OAuth Token Introspection Response" registry {{RFC7662}}:

*  Name: `actor_proofs`
*  Description: Array of actor-signed hop proofs returned by introspection
*  Change Controller: IESG
*  Specification Document(s): This document

*  Name: `actor_proofs_complete`
*  Description: Indicates whether the returned actor proofs provide complete visible-hop coverage
*  Change Controller: IESG
*  Specification Document(s): This document

# Acknowledgments

This document builds on the OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}}, on the OAuth Actor Receipts companion {{ACTOR-RECEIPTS}}, on the OAuth 2.0 Token Exchange specification {{RFC8693}}, on the OAuth 2.0 Transaction Tokens work {{I-D.ietf-oauth-transaction-tokens}}, and on prior OAuth Working Group discussion of delegation transparency, sender-constrained tokens, and proof-of-possession mechanisms ({{RFC7800}}, {{RFC8705}}, {{RFC9449}}).  Separate working group contributions have explored actor-signed step artifacts in different layerings; this document profiles the actor-signed per-hop portion of that design space as a composable companion.

Individual contributors and reviewers will be acknowledged in subsequent revisions of this document as feedback accumulates.

--- back

# Examples

The examples in this appendix show decoded proof contents.  Real proofs are compact-signed JWT strings carried in the `actor_proofs` array.  The `iat` and `exp` values shown are illustrative only; in deployments, proof `exp` is set per {{proof-claims}} and {{extending-an-existing-proof-chain}} so that no inbound proof expires before the outer token that carries it.  The delegation scenario continues the two-hop travel example of {{ACTOR-RECEIPTS}}: the subject alice delegates to an AI travel-assistant agent through the enterprise AS, and the agent's token is exchanged at the travel-provider AS, which adds a booking tool as the new outermost actor.

## Example: Two-Hop Delegation Chain with Sibling Receipts

The outer token carries both companions:

~~~json
{
  "jti": "e8f4a2d6-3b1c-4d7e-9f5a-0c2b4d6e8f0a",
  "iss": "https://as.travel-provider.example",
  "aud": "https://api.travel-provider.example",
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
  "actor_proofs": [
    "<proof-0>",
    "<proof-1>"
  ],
  "actor_proofs_complete": true
}
~~~

`actor_proofs[0]` was signed by the booking tool when it requested the exchange at the travel-provider AS, before that AS issued the outer token:

~~~json
{
  "iss": "https://tools.example.com/booking-tool",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://tools.example.com/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service"
  },
  "target": {
    "aud": ["https://api.travel-provider.example"]
  },
  "prh": "Xm3VqLr8pTzKNdY5W2uEbc4gHf7jAsQ9R6vBnC1oD0k",
  "iat": 1776745180,
  "exp": 1776832000,
  "jti": "5f2e8d91-4a6b-4c3d-8e2f-1a9b8c7d6e5f"
}
~~~

`actor_proofs[1]` was signed earlier by the AI agent when the enterprise AS added it as the first actor hop:

~~~json
{
  "iss": "https://agents.example.com/travel-assistant",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_iss": "https://idp.enterprise.example",
  "act": {
    "sub": "https://agents.example.com/travel-assistant",
    "iss": "https://as.enterprise.example",
    "sub_profile": "ai_agent"
  },
  "target": {
    "aud": ["https://as.travel-provider.example"]
  },
  "iat": 1776741580,
  "exp": 1776832000,
  "jti": "7a1c9e42-3b5d-4f6a-9c8e-2d4f6a8b0c1e"
}
~~~

The sibling receipts follow the same construction as the examples of {{ACTOR-RECEIPTS}}, with the `proof_jti` claim defined in {{sibling-receipt-issuance}} included at receipt creation.  Because these receipts carry `proof_jti`, they are different byte strings from the receipts shown in that document's examples: they carry their own `jti` values, and the newest receipt's `prh` differs because it hashes a different older receipt.  The newest receipt, signed by the travel-provider AS, carries:

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
  "proof_jti": "5f2e8d91-4a6b-4c3d-8e2f-1a9b8c7d6e5f",
  "prh": "K9mPvXq2LwTnR7dYcE5uHb8jZa4gFs6iOk1rC3xW0eA",
  "iat": 1776745200,
  "exp": 1776832000,
  "jti": "b7d1f3a5-8c2e-4a6b-9d0f-1e3a5c7b9d1f",
  "origin_jti": "e8f4a2d6-3b1c-4d7e-9f5a-0c2b4d6e8f0a"
}
~~~

This example illustrates the properties this profile adds.  The outer token's `aud` is within `actor_proofs[0].target.aud`, so the current audience is actor-authorized.  `actor_proofs[1].target.aud` names the travel-provider AS, the target the agent authorized for its own hop; per step 9 of {{consumer-processing}} that historical binding is not evaluated against the current outer token's audience.  The receipt's `proof_jti` binds the proof chain to the receipt chain, whose `origin_jti` anchors the current token instance; a compromised issuer re-embedding `<proof-0>` in a different token could not produce a matching trusted receipt.  The proofs in this example are signed by the keys whose thumbprints appear as the historical presenter bindings in the sibling receipts (`ToolJKT`, `AgentJKT`), illustrating receipt-attested key resolution; per {{actor-key-resolution}}, a recipient relying on that pattern gains no anti-fabrication protection against the receipt issuer itself at that hop.

## Example: Proofs-Only Partial Coverage

Suppose the enterprise AS has not yet deployed proof support, so no proof exists for the AI-agent hop, and the travel-provider AS accepts a proof from the booking tool when it adds the tool as the new outermost actor.  The resulting access token carries a one-element proof chain and no receipts:

~~~json
{
  "jti": "a4c8e2f6-1b3d-4e5f-9a7c-8d6e4f2a0b1c",
  "iss": "https://as.travel-provider.example",
  "aud": "https://api.travel-provider.example",
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
  "actor_proofs": [
    "<proof-0>"
  ],
  "actor_proofs_complete": false
}
~~~

The single proof covers the outermost hop:

~~~json
{
  "iss": "https://tools.example.com/booking-tool",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://tools.example.com/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service"
  },
  "target": {
    "aud": ["https://api.travel-provider.example"],
    "resource": ["https://api.travel-provider.example/bookings"]
  },
  "iat": 1776745180,
  "exp": 1776832000,
  "jti": "9c3b7f15-6d2e-4a8b-b1f4-e5a7c9d1b3f6"
}
~~~

`prh` is omitted because this is a single-element chain.  `actor_proofs_complete: false` signals to recipients that the inner AI-agent hop carries no actor-signed evidence.  Resource servers that set `actor_proofs_complete_required: true` in their Protected Resource Metadata reject this token; resource servers that accept partial coverage validate the booking tool's signed participation and target consent, and treat the agent hop as carried solely by the visible `act` chain.  Because no receipts are present, the proof chain carries no outer-token instance binding; per {{proof-to-token-binding-limits}}, a recipient requiring instance binding would require the receipts companion or a provisioned `origin_jti`.

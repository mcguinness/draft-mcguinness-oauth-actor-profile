---
title: "OAuth Actor-Signed Hop Proofs"
abbrev: "OAuth Actor Proofs"
category: std
docname: draft-mcguinness-oauth-actor-proofs-latest
submissiontype: IETF
number:
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
  RFC3986:
  RFC6749:
  RFC6750:
  RFC6838:
  RFC7515:
  RFC7519:
  RFC7523:
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
  I-D.mcguinness-oauth-actor-receipts:

informative:
  RFC9700:
  I-D.mw-oauth-actor-chain:
  I-D.liu-oauth-chain-delegation:
  I-D.jiang-oauth-intent-admission:
  I-D.ietf-oauth-attestation-based-client-auth:
  I-D.ietf-oauth-spiffe-client-auth:

...

--- abstract

This document defines OAuth Actor-Signed Hop Proofs, an optional companion to the OAuth Actor Profile for Delegation.  Each proof is a signed JSON Web Token (JWT) recording an actor's participation and authorized target for one hop.  The `actor_proofs` claim carries a hash-linked chain verified through trusted actor-key sources.  This document specifies proof submission, validation, optional links to actor receipts, metadata, and introspection parameters.

--- middle

# Introduction

The OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}} makes actor identity visible in delegated tokens through a common `act` claim.  The OAuth Actor Receipts companion {{I-D.mcguinness-oauth-actor-receipts}} adds authorization-server-signed per-hop provenance.  Both are issuer assertions: an authorization server attests that an actor was added at a hop.  Nothing in either profile requires the actor's own cryptographic participation, so a compromised or dishonest issuer can fabricate the participation of an actor that never authorized the delegation.

This document defines OAuth Actor-Signed Hop Proofs, an optional companion profile that adds actor-side evidence.  At each covered hop, the actor signs its participation and authorized target; the AS validates the proof and includes it in `actor_proofs`, and recipients verify the actor's signature through trusted key sources.  The design center is:

*  keep the visible actor chain in `act`;
*  keep authorization-server-signed provenance in `actor_receipts` when the receipts companion is in use;
*  carry actor-signed participation and hop-time target consent in separately signed proofs.

This profile adds the `actor_proof` request parameter, an output claim, and discovery metadata.  [Design Goals and Non-Goals](#design-goals-and-non-goals) defines the scope.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses OAuth terminology from {{RFC6749}} and {{RFC8693}}, and Transaction Token terminology from {{I-D.ietf-oauth-transaction-tokens}}.  AS, RS, and TTS denote authorization server, resource server, and Transaction Token Service.  Actor Receipt and Receipt Chain follow {{I-D.mcguinness-oauth-actor-receipts}}.  Outer Token is the token associated with a proof chain, whether or not it also carries receipts.

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
: The actor receipt, if any, created for the same visible actor hop as a proof, under {{I-D.mcguinness-oauth-actor-receipts}}.

Complete Proof Coverage:
: A condition in which the number of proofs in `actor_proofs` equals the number of visible actor hops in the token's `act` chain, and every proof aligns with the corresponding visible hop.

Examples in this document are illustrative and omit unrelated claims, signatures, and validation steps that a complete deployment would need.

# Relationship to the Core Actor Profile

This document is an extension of {{I-D.mcguinness-oauth-actor-profile}}.  A token that uses the `actor_proofs` claim defined here:

*  conforms to the actor-chain representation rules of the core actor profile ({{actor-proofs-claim}});
*  uses the top-level `cnf` claim, when present, only for the current token presenter, as the core actor profile defines;
*  gains no proof-of-possession semantics from its proofs for the current request ({{current-presenter-validation}}).

This profile does not redefine the request semantics of {{RFC8693}} or of Transaction Tokens.  It defines only:

*  the `actor_proofs` claim;
*  the signed JWT format of each proof;
*  the `actor_proof` token request parameter for conveying a proof at issuance;
*  issuer and consumer processing for proofs;
*  associated metadata and introspection parameters.

## Relationship to the Actor Receipts Companion {#relationship-to-receipts}

Receipts are signed by the AS; proofs are signed by the actor.  A token MAY carry either, both, or neither, and recipients select a validation posture by local policy:

*  **Receipts-only**: proofs absent or ignored; trust per {{I-D.mcguinness-oauth-actor-receipts}}.
*  **Proofs-only**: receipts absent or ignored; trust rests on actor-key resolution and actor signatures.
*  **Belt-and-suspenders**: both validated; independent issuer-side and actor-side attestations for covered hops, linked by the sibling references in {{sibling-receipt-issuance}}.

Receipts and proofs remain separate compact JWTs, rather than one JWS with both signatures over a shared payload (JWS JSON Serialization, {{RFC7515, Section 7.2}}), because their signers, adoption prerequisites, and threat models differ.  Receipts require only issuer support, while proofs also require actor signing keys and trusted key resolution.  Separate artifacts keep the two trust anchors independent ({{threat-model}}) and let deployments adopt, validate, and hash-chain issuer and actor evidence independently.

The receipts companion's distinction between historical evidence and current introspection status also applies to proofs.

## Relationship to Other Actor-Evidence Work {#related-work}

Several contemporaneous efforts add actor-side or issuer-side delegation evidence to OAuth deployments; they differ from this profile chiefly in where evidence lives, who signs it, and who can verify it.  This section is informative.

*  {{I-D.mw-oauth-actor-chain}} retains actor-signed step proofs at the AS and carries an issuer-signed cumulative commitment in the token.  Actor-signature verification depends on AS retention.  This profile carries the proofs for direct recipient verification, increasing token size at each hop.
*  {{I-D.liu-oauth-chain-delegation}} carries AS-signed hop records inline, optionally countersigned by the delegator.  Those fields carry no target binding, and records are re-signed at domain boundaries.
*  {{I-D.jiang-oauth-intent-admission}} defines a single-hop intent artifact signed by the admission authority.

The distinguishing property of this profile relative to each is a recipient-verifiable artifact signed by the actor itself, before issuance, over an explicit target binding.  These designs address overlapping needs; convergence is a working-group discussion this document aims to inform rather than preempt.

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

This profile requires actors capable of signing, such as agents, workloads, and services with their own keys.  Actors without signing keys can use issuer-signed receipts instead.

Recipients need a trusted key source for every actor whose proof they use ({{actor-key-resolution}}).  This can require more configuration than trusting the smaller set of receipt issuers.

The anti-fabrication property of this profile is conditional on recipients requiring proofs.  An issuer that fabricates actor participation simply omits proofs; recipients that accept proof-less delegated tokens receive no protection from this profile ({{downgrade-by-omission}}).

# Actor Proofs Overview

An actor proof records one actor hop from the actor's side.  The actor being added at a hop signs a proof naming itself, the subject on whose behalf it acts, and the target binding it authorizes for the token issued at that hop.  The issuing authorization server validates the proof against the actor's verification key and against the token it is about to issue, then embeds the proof in the issued token.

The `actor_proofs` array is ordered newest first and preserves older entries unchanged.  Index 0 corresponds to outermost `act`, index 1 to `act.act`, and so on.  Coverage is either complete or a contiguous outermost prefix; local policy or resource requirements determine whether partial coverage is acceptable.  Proofs and receipts align at indexes covered by both arrays.

A valid proof chain records actor participation and hop-time target consent.  It conveys no authority and does not establish that delegation remains active.  Current authorization decisions MUST evaluate the current outer token, current policy, and current state, not the proof chain alone.

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

  Issuers SHOULD set `actor_proofs_complete` to `true` for complete coverage and `false` for partial coverage.  An absent value provides no completeness attestation; consumers requiring the literal value `true` treat absence like `false`.

This document does not require every delegated token to carry `actor_proofs`.  A deployment that requires actor-signed evidence uses local policy or the metadata defined in {{discovery-capability-signaling}} to express that requirement.

# Actor Proof JWT Format {#actor-proof-jwt-format}

Each element of `actor_proofs` is a signed JWT represented using JWS compact serialization {{RFC7515}}.

## JOSE Header

The JOSE header of an actor proof:

*  MUST include an asymmetric digital-signature `alg` value;
*  MUST NOT use `alg: none` or a MAC-based symmetric algorithm;
*  MUST include `typ` with the value `actor-proof+jwt`;
*  SHOULD include `kid` when the actor's key source publishes multiple verification keys;
*  MAY include `crit`; a proof whose `crit` header lists an extension header the consumer does not understand is invalid per {{RFC7515, Section 4.1.11}}.

Actors, issuers, and consumers MUST apply the JWT best practices in {{RFC8725}} when creating and validating proofs, except for the audience requirements of {{RFC8725, Section 3.9}}, from which this profile departs by prohibiting `aud` as described in {{proof-claims}}.

## Proof Claims {#proof-claims}

The JWT payload of an actor proof uses the claims defined below, grouped by purpose.

### Identity Claims

`iss`:
: REQUIRED.  The identifier of the actor that signed the proof.  It MUST equal the proof's `act.sub` value.

  Interpret `iss` within the namespace given by `act.iss`.  A bare `iss` MUST NOT serve as the sole key-resolution or trust index; use (`act.iss`, `act.sub`) as specified in {{actor-key-resolution}}.

`sub`:
: REQUIRED.  The subject identifier on whose behalf the actor authorized the delegation, as known to the actor at signing time.  `actor_proofs[0].sub` MUST equal the outer token's top-level `sub`.  Older proofs can carry differing `sub` values, which step 8 of {{consumer-processing}} accepts structurally; authorization that depends on subject equivalence across them is subject to the continuity rules of {{subject-re-expression-across-hops}}.

`sub_iss`:
: OPTIONAL.  The namespace authority under which the proof `sub` value is interpreted, with the semantics defined for the `sub_iss` claim in {{I-D.mcguinness-oauth-actor-receipts}}.  When absent, the namespace is determined as for an absent receipt `sub_iss`.

`act`:
: REQUIRED.  A single-hop actor object identifying the signing actor.  This object:

  *  MUST conform to the core actor profile's actor-object rules;
  *  MUST include `act.sub` and `act.iss`;
  *  MUST NOT contain `cnf`;
  *  MUST NOT contain a nested `act`.

  `act` supplies the namespace context and visible-hop alignment.  A proof is invalid if `iss` differs from `act.sub`.

  These restrictions apply to the proof's actor object.  The token's `act` chain can retain confirmation members as extension data under the core actor profile.  The proof's actor object omits those members while satisfying visible-hop alignment (step 7 of {{consumer-processing}}); this does not modify the token's actor chain.

Proofs define no subject `sub_profile` claim; subject classification remains issuer-asserted.  Actor classification can appear in `act.sub_profile`.

### Target Binding

`target`:
: REQUIRED.  A JSON object recording the target binding the actor authorized for the token issued at this hop.  Members:

  `target.aud`:
  : REQUIRED.  A string or array of strings.  The audiences the actor authorizes for the token issued at this hop.

  `target.resource`:
  : OPTIONAL.  An array of URIs with the semantics of the `resource` request parameter of {{RFC8707}}.  When present, it narrows the target binding beyond `target.aud`.

  A target binding records hop-time consent for the hop the proof covers.  Issuer-side enforcement at the covered hop is defined in {{accepting-a-proof}}; consumer evaluation, including the distinction between the newest proof and older proofs, is defined in step 9 of {{consumer-processing}}.

  Extension members MAY be defined by other specifications.  An extension member MUST be defined with constraining semantics only: its presence narrows what the actor authorized and its absence leaves the binding as expressed by the defined members.  Consumers MUST ignore unrecognized `target` members unless another specification or local agreement defines their meaning; actors MUST NOT rely on unrecognized extension members being enforced.

`target.aud` describes the token audiences authorized at this hop, not the proof's recipients.  Using JWT `aud` for this purpose would incorrectly apply current-recipient audience checks to historical proofs.

### Chain Linkage

`prh`:
: OPTIONAL.  Previous proof hash of the next older proof in the chain; the oldest proof, including the sole proof of a single-element chain, omits it.

  The `prh` and `prh_alg` claims are reused from {{I-D.mcguinness-oauth-actor-receipts}} with the same construction, applied to proof JWTs.  The proof chain is linked independently of any receipt chain carried in the same token: each companion's `prh` values hash that companion's own artifacts.

`prh_alg`:
: OPTIONAL.  Hash algorithm identifier naming the algorithm used to compute `prh`.  The value, consistency, and extension rules of the receipt `prh_alg` claim apply to proof chains.

  *  When absent, the default is `sha-256`.
  *  The proof chain's `prh_alg` is independent of the receipt chain's `prh_alg` in the same token; the two chains MAY use different algorithms.

### Sibling Receipt Reference

`receipt_jti`:
: OPTIONAL.  The `jti` of the sibling receipt created for the same hop under {{I-D.mcguinness-oauth-actor-receipts}}.

  The actor needs a prospective receipt identifier to include this claim.  Most deployments instead use the receipt's `proof_jti`, which the issuer sets after validating the proof ({{sibling-receipt-issuance}}).  Consumer step 10 validates both references.

### Time and Uniqueness

`iat`:
: REQUIRED.  The time at which the proof was signed, as defined in {{RFC7519}}.

`exp`:
: REQUIRED.  Expiration time for the proof, as defined in {{RFC7519}}.

  `exp` needs to cover the lifetime of any token that will carry or inherit this proof; otherwise consumers reject older proofs in a valid chain prematurely.

  A proof expiring before the outer token an issuer would issue caps that token's `exp` or ends the proof's propagation ({{issuer-processing}}).  Longer validity supports delegated sessions but also extends exposure to key compromise and proof reuse ({{proof-to-token-binding-limits}}).

  With instance binding through receipts in strict mode or a provisioned `origin_jti` ({{proof-to-token-binding-limits}}), `exp` MAY cover the delegated session only while the outer token stays instance-bound.  Refresh or reissuance ends instance binding, so issuers that refresh tokens carrying proofs SHOULD keep proof `exp` short.  Without instance binding, `exp` SHOULD be short to limit proof reuse.

`jti`:
: REQUIRED.  A unique identifier for the proof, as defined in {{RFC7519}}.  Recipients and auditors can use `jti` uniqueness across observed tokens to detect proof re-embedding ({{proof-to-token-binding-limits}}).

### Outer-Token Binding

`origin_jti`:
: OPTIONAL.  The `jti` of the outer token issued at the hop this proof covers, following the pattern of the `origin_jti` claim defined in {{I-D.mcguinness-oauth-actor-receipts}}.

  This requires provisioning the prospective token identifier before signing.  A matching `actor_proofs[0].origin_jti` binds the chain to that token instance; a present but differing value causes rejection unless the outer issuer is a trusted reissuing issuer.  Without `origin_jti`, instance binding requires receipt composition; see {{proof-to-token-binding-limits}} and consumer step 9.

  A Transaction Token Service MAY include `jti` in a Transaction Token, because {{I-D.ietf-oauth-transaction-tokens, Section 9.2}} permits additional claims; without it, a Transaction Token's proof chain is never instance-bound.

### Excluded Standard Claims

`aud`:
: Prohibited.  Actors MUST NOT include `aud` in a proof, and consumers MUST reject a proof that carries it (step 5 of {{consumer-processing}}).

  Proofs are validated as part of outer-token processing, not as independent JWTs against an audience; the outer token carries the audience scoping for the request, and the actor's consented audiences live in `target.aud`.  This profile departs from {{RFC8725, Section 3.9}} on those grounds.  Including `aud` in a proof would create ambiguity between an audience restriction on the proof artifact and the target binding, which are different statements; rejecting it gives the result that {{RFC7519, Section 4.1.3}} requires when the processing principal does not identify itself with the `aud` value.

### Extension Claims

A proof MAY contain additional claims defined by another specification or by deployment policy.  Consumers ignore unrecognized claims unless another specification or local agreement defines their meaning, per {{RFC7519, Section 4}}.

## Proof-Chain Linkage {#proof-chain-linkage}

When an actor signs a new proof that extends an inherited proof chain:

*  if there is an older proof immediately following it in the array, the new proof MUST include `prh`, and that value MUST be the base64url encoding without padding of the hash of the ASCII octets of the exact compact JWT string of that next proof;
*  if the new proof is the only proof in the array, it MUST omit `prh`.

The hash input is the exact compact JWS string, without JSON {{RFC8259}} canonicalization.  Systems that carry, store, or forward `actor_proofs` arrays MUST preserve each proof byte-for-byte.  Re-encoding changes the hash even if the claims remain equivalent.

# Conveying Proofs at Issuance {#actor-proof-parameter}

This document defines one token request parameter:

`actor_proof`:
: OPTIONAL.  The compact serialization of a single actor proof JWT for the new outermost actor hop of the requested token.  A request carries at most one `actor_proof` parameter ({{RFC6749, Section 3.2}}).

The parameter is defined for token endpoint requests that produce delegated tokens under the core actor profile, including OAuth 2.0 Token Exchange {{RFC8693}} requests and JWT assertion grants.  A Transaction Token request made over HTTP is a Token Exchange request ({{I-D.ietf-oauth-transaction-tokens, Section 11}}) and carries the proof in `actor_proof`; other Transaction Token Service interfaces convey it equivalently.

The AS authenticates the actor and derives its identity under the core profile, then separately validates the proof ({{accepting-a-proof}}).  `actor_proof` supplies participation and consent evidence; it does not serve as `actor_token` or authenticate the request.

This document does not define a challenge mechanism by which an authorization server provides prospective values (such as the outer token's `jti`, a receipt's `jti`, or the newest inbound proof for opaque inbound tokens) to the actor before signing.  Deployments and companion profiles MAY define such mechanisms; the `origin_jti` and `receipt_jti` claims are the designed insertion points.

# Issuer Processing {#issuer-processing}

This section defines how an authorization server or Transaction Token Service accepts, validates, embeds, preserves, and extends `actor_proofs`.

When a proof the issuer retains from an inbound token or refresh state has an `exp` earlier than the `exp` the issuer would set for the issued token, the issuer:

1.  MAY lower the issued token's `exp` to the earliest `exp` among the retained proofs;
2.  otherwise, where local policy permits absent coverage, MUST drop the `actor_proofs` array;
3.  otherwise, MUST fail the request: `invalid_grant` on a refresh or JWT bearer grant request ({{RFC6749, Section 5.2}}), `invalid_request` on a Token Exchange request ({{RFC8693, Section 2.2.2}}).

## Accepting a Proof for a New Actor Hop {#accepting-a-proof}

When an issuer adds a new outermost actor hop and the token request carries `actor_proof`, the issuer:

1.  MUST validate the proof's structure per {{actor-proof-jwt-format}}: `typ` value, asymmetric `alg`, presence and JSON types of the REQUIRED claims `iss`, `sub`, `act`, `target` (including `target.aud`), `iat`, `exp`, and `jti`, the absence of `aud`, and the single-hop `act` rules.
2.  MUST verify that the proof's (`act.iss`, `act.sub`) pair equals the actor identifier pair the issuer will emit as the new outermost visible `act` object, and that the proof `iss` equals the proof `act.sub`.  Deployment configuration supplies the actor with the `act.iss` value the issuer will emit.
3.  MUST verify that the proof `sub` equals the top-level `sub` of the token being issued.  An issuer that re-expresses the subject at this hop MUST NOT embed the proof; re-expression breaks the alignment between `actor_proofs[0].sub` and the outer token's top-level `sub` that consumers verify under {{consumer-processing}}.
4.  MUST resolve the actor's verification key through an actor-key source trusted under the issuer's local policy and validate the proof's signature ({{actor-key-resolution}}).
5.  MUST verify that the proof's `exp` is no earlier than the issued outer token's `exp`, and that `iat` is plausible under the issuer's clock-skew policy.
6.  MUST NOT issue an outer token whose `aud` or effective resource indicators exceed the proof's target binding.  Every audience of the issued token MUST be present in `target.aud`, and every effective resource indicator MUST equal an entry of `target.resource` under simple string comparison ({{RFC3986, Section 6.2.1}}) when that member is present.  When `target.resource` is present and the request supplies no resource indicators, the issuer uses `target.resource` as the effective resource indicators and MUST NOT issue a token whose effective resources exceed it.  The issuer MAY narrow the issued resource indicators to fit the target binding, as {{RFC8707, Section 2.2}} leaves acceptable resources to its policy, but does not drop a requested audience; {{error-handling}} gives the error to return when the requested target cannot be issued within the target binding after any narrowing.
7.  MUST verify, when the proof carries `origin_jti`, that it equals the issued token's `jti`, and, when the proof carries `receipt_jti` and the issuer creates a sibling receipt for this hop ({{sibling-receipt-issuance}}), that it equals that receipt's `jti`.
8.  MUST include the validated proof as `actor_proofs[0]` of the issued token, subject to the chain rules below.

When no inbound `actor_proofs` are being preserved, the proof starts a new chain and MUST omit `prh`.  The one-element array is complete coverage only when the visible `act` chain has depth 1; {{actor-proofs-claim}} governs how the issuer sets `actor_proofs_complete` in each case.

If proof validation fails, the issuer MUST NOT embed the proof.  When local policy or the deployment's resource requirements require actor-signed evidence for the issuance, the issuer MUST fail the request under the error model of {{error-handling}}; otherwise it MAY issue the token without `actor_proofs`.

## Extending an Existing Proof Chain {#extending-an-existing-proof-chain}

When an issuer adds a new outermost actor hop and also preserves an inbound `actor_proofs` array, it:

1.  MUST validate the inbound proof chain by applying the consumer processing rules in {{consumer-processing}} before relying on it or carrying it forward.
2.  MUST verify that each inbound proof's `exp` is no earlier than the issued outer token's `exp`, applying the lifetime rule in {{issuer-processing}} when an inbound proof's `exp` is earlier than the `exp` the issuer would set.  Issuers MAY apply a small clock-skew margin to this comparison, consistent with the consumer-side skew tolerance in {{consumer-processing}}, but MUST NOT broadly accept inbound proofs whose `exp` precedes the issued outer token's `exp` by more than a deployment-defined skew bound.
3.  preserves each inbound proof byte-for-byte unchanged, as required by {{proof-chain-linkage}}.
4.  MUST accept exactly one new proof, conveyed per {{actor-proof-parameter}} and validated per {{accepting-a-proof}}, for the new outermost actor hop.  Without a valid new proof, the issuer MUST NOT carry the inbound `actor_proofs` array forward; it continues without proofs where local policy permits absent coverage, and otherwise MUST fail the request under {{error-handling}}.
5.  MUST verify that the new proof's `prh` equals the hash of the exact compact serialization of the inbound array's newest proof, computed using the algorithm named by the inherited `prh_alg` (defaulting to SHA-256 when absent), and MUST verify that the new proof's `prh_alg` matches the inherited chain's value or is omitted when the chain omits it.  An issuer that does not support the inbound `prh_alg` MUST reject the chain rather than rehash; rehashing would invalidate prior actors' signatures.
6.  MUST prepend the new proof to the inherited array.
7.  MUST NOT set `actor_proofs_complete` to `true` unless every inbound proof validated and the issued token's proof count equals its visible actor-chain depth, and SHOULD set it to `false` otherwise.

Byte-for-byte preservation ({{proof-chain-linkage}}) rules out reserializing, re-signing, normalizing, trimming, or otherwise altering a prior proof.

The actor must know the newest inbound proof's exact serialization, or its hash and `prh_alg`, before signing.  For JWT inputs it can read `actor_proofs[0]`.  For opaque inputs, the deployment needs to supply that information through a mechanism it defines ({{actor-proof-parameter}}).  If unavailable, the issuer MUST NOT accept a proof without `prh` as a chain extension; it MAY instead start a new chain under {{accepting-a-proof}} where local policy permits partial coverage ({{partial-coverage-and-full-coverage}}).

If inbound proofs fail validation, the issuer MUST NOT propagate them.  It MAY continue without them only when local policy permits partial or absent coverage, and MAY then begin a new chain at its own hop under {{accepting-a-proof}}; the result is partial coverage and MUST NOT carry `actor_proofs_complete: true`.  Otherwise it MUST fail the request under the error model of the underlying protocol.

## Reissuance Without a New Actor Hop {#reissuance-without-a-new-actor-hop}

An issuer that reissues, translates, or introspects and re-emits a token without adding a new outermost actor hop:

*  MAY carry an `actor_proofs` array received in an inbound token or its introspection response forward unchanged, and MUST first validate it against that token under {{consumer-processing}}, as step 1 of {{extending-an-existing-proof-chain}} requires for extension.  If the array fails validation, the issuer MUST NOT carry it forward, and MUST fail the request under the error model of the underlying protocol unless local policy permits the issued token to lack it;
*  MUST NOT accept or embed a new proof, and MUST reject with `invalid_request` ({{RFC6749, Section 5.2}}) a request that carries an `actor_proof` parameter;
*  MUST preserve `actor_proofs_complete` when carrying the array unchanged.  If it cannot attest that value, the issuer MUST drop the whole array.
*  MUST NOT continue to carry an inherited `actor_proofs` array if it cannot preserve the visible hop alignment required by {{consumer-processing}};
*  MUST NOT change top-level `sub` while retaining proofs; doing so breaks alignment with `actor_proofs[0].sub`.
*  MUST NOT set the outer token's `exp` later than the earliest `exp` among the retained proofs, and applies the lifetime rule in {{issuer-processing}} when it would otherwise set a later `exp`.

If such an issuer changes the visible outermost actor, it has added a new hop and MUST follow {{extending-an-existing-proof-chain}}.

If reissuance exceeds the newest proof's target, the issuer MUST drop `actor_proofs` unless deployment agreement designates it, for the recipients of the reissued token, as a trusted reissuing issuer permitted to retarget ({{target-binding-strict-mode}}).  Narrowing or preserving the target keeps the token within the target binding.  A reissued token with a new `jti` diverges from a present `actor_proofs[0].origin_jti` ({{target-binding-strict-mode}}).

If proofs are dropped while receipts remain, inherited `proof_jti` references become informational.  Recipients requiring bound siblings enforce proof presence through `actor_proofs_required`, `actor_proofs_complete_required`, or local policy.

An AS that supports refresh tokens for delegated access tokens carrying proofs:

*  needs to retain the `actor_proofs` array in issuer-controlled state across refresh, either in durable storage (for example, a token-state database or refresh-token state) or embedded in a self-contained refresh token, so each refreshed access token can carry the proofs forward unchanged.
*  applies the lifetime rule in {{issuer-processing}} to each refreshed access token.  Refresh ends instance binding, so proof `exp` sizing for refreshed tokens follows the short-`exp` guidance in {{proof-claims}}.
*  after dropping `actor_proofs` under that rule, restores actor-signed evidence only through a new delegated issuance that adds a hop with a fresh proof, because refresh adds no actor hop.

## Partial Coverage and Full Coverage {#partial-coverage-and-full-coverage}

This document permits partial proof coverage for progressive deployment.  An issuer MAY begin a new proof chain even when older inner actor hops remain visible but uncovered.

However:

*  a partial chain still covers a contiguous outermost prefix of the visible actor chain, as {{actor-proofs-claim}} requires;
*  an issuer therefore cannot skip an outer visible hop and carry a proof only for an inner visible hop;
*  when local policy or resource requirements require full actor-signed evidence, the issuer MUST either emit complete proof coverage or fail the request under the error model of the underlying protocol.

Partial coverage leaves the oldest hops uncovered, including the original subject-to-actor delegation.  Deployments needing evidence for that hop should enable proof support at the origin and its actors first.  Resource servers can require full coverage through `actor_proofs_complete_required` or local policy.

When an introspection server filters the visible `act` chain (see the `chain_complete` introspection member defined in the core actor profile {{I-D.mcguinness-oauth-actor-profile}}), `actor_proofs` covers only the visible filtered chain.  In that case `actor_proofs_complete` describes coverage relative to the visible filtered chain, not the unfiltered delegation chain; recipients that need true-chain completeness MUST evaluate `chain_complete` separately.

## Transaction Token Service Rebinding

A Transaction Token Service that establishes a new presenter and makes that presenter the new outermost actor follows the same proof rules as any other issuer that adds a new outermost actor hop, as defined in {{extending-an-existing-proof-chain}} (or {{accepting-a-proof}} when no inbound `actor_proofs` exist).  The new presenter is the signing actor for the new proof; inherited proofs are carried forward unchanged.  This profile does not define additional proof claims specific to Transaction Tokens.

## Sibling Receipt Issuance {#sibling-receipt-issuance}

When a deployment uses both this profile and {{I-D.mcguinness-oauth-actor-receipts}}, the issuer that adds a hop creates the receipt and embeds the proof for that hop in the same issuance operation.  The two artifacts are siblings: independent attestations of the same hop by different signers.

This document defines the following extension claim for Actor Receipt JWTs, under the extension-claims rule of {{I-D.mcguinness-oauth-actor-receipts}}:

`proof_jti`:
: OPTIONAL.  The `jti` of the proof the receipt issuer validated for the same hop.  When the issuer embeds a proof and creates a sibling receipt for one hop, the receipt SHOULD include `proof_jti` equal to that proof's `jti`.

The issuer knows the proof identifier before signing the receipt; the actor knows a prospective receipt identifier only if it was provisioned.  Byte preservation keeps `proof_jti` fixed, making later proof substitution detectable through sibling validation.

Consumer verification of sibling references is defined in step 10 of {{consumer-processing}}.

# Consumer Processing {#consumer-processing}

An issuer, resource server, or other recipient that relies on `actor_proofs` MUST perform the following steps.

1.  Validate the outer token according to its token type and the core actor profile.
2.  If `actor_proofs` is absent, treat the token as lacking actor-signed evidence.  Whether that is acceptable is determined by local policy or by Protected Resource Metadata signals such as `actor_proofs_required` and `actor_proofs_complete_required` defined in {{discovery-capability-signaling}}.  If `actor_proofs_complete` is present with the value `true` while `actor_proofs` is absent, the combination is malformed; the recipient MUST treat this as a failed required check and apply the rejection rule following step 11.
3.  Verify that `actor_proofs`, if present, is a non-empty JSON array of strings.  Verify that `actor_proofs_complete`, if present, is a JSON boolean.
4.  Verify that the number of proofs does not exceed the visible actor-chain depth of the outer token.  If the outer token carries `actor_proofs_complete: true`, verify that the proof count exactly equals the visible actor-chain depth; if it does not, the check fails.
5.  For each proof, in array order:
    *  parse the string as a compact JWT;
    *  verify that the JOSE header uses an asymmetric digital-signature `alg` value accepted for that actor, and reject proofs that use `alg: none` or a MAC-based symmetric algorithm ({{RFC8725, Section 3.1}});
    *  verify that `typ` equals `actor-proof+jwt`;
    *  verify that the proof's (`act.iss`, `act.sub`) pair is within the scope of an actor-key source the recipient trusts, before performing any network retrieval keyed by the proof's content;
    *  resolve the actor's verification key from that source ({{actor-key-resolution}});
    *  validate the JWT signature;
    *  reject a proof whose `crit` header lists an extension header the consumer does not understand;
    *  verify that all REQUIRED proof claims are present and have the expected JSON types, including `iss`, `sub`, `act`, `target` with `target.aud`, `iat`, `exp`, and `jti`;
    *  verify that OPTIONAL claims used by this profile have the expected JSON types when present, including `sub_iss`, `target.resource`, `prh`, `prh_alg`, `receipt_jti`, and `origin_jti`;
    *  reject a proof that carries an `aud` claim ({{proof-claims}});
    *  verify that the proof `act` object is single-hop, contains no nested `act`, and contains no `cnf`, and that the proof `iss` equals the proof `act.sub`;
    *  enforce `exp`, `iat`, and other JWT validity rules.  An expired proof is invalid even for an older hop; only the small clock-skew leeway of {{RFC7519, Section 4.1.4}} applies.
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
    *  older proofs MAY carry differing `sub` or `sub_iss` values.  This acceptance is structural only: authorization that depends on subject equivalence across those proofs is subject to the continuity rules of {{subject-re-expression-across-hops}}.
9.  Evaluate outer-token binding and target binding:
    *  when `actor_proofs[0].origin_jti` is present and equals the outer token's `jti`, the proof chain is bound to the current outer-token instance; when it is present and differs, the chain has diverged, {{target-binding-strict-mode}} decides whether the recipient rejects it (rejection is the default), and an accepted value is historical provenance;
    *  when `actor_proofs[0].origin_jti` is absent, the proof chain carries no instance binding of its own; this is not by itself a validation failure;
    *  verify that every audience of the outer token is present in `actor_proofs[0].target.aud`, and, when the outer token's effective resource indicators are determinable from token claims, the introspection response, or trusted local context, that each equals an entry of `actor_proofs[0].target.resource` under simple string comparison ({{RFC3986, Section 6.2.1}}) when that member is present.  A recipient that cannot determine the outer token's effective resources treats the proof as consent to the audience only, not as divergence, and MUST NOT infer resource-level consent.  A token whose audience or resources exceed the newest proof's target binding has also diverged; {{target-binding-strict-mode}} decides whether the recipient rejects the chain (rejection is the default) and limits an accepted chain to participation evidence, not actor consent to the current target;
    *  target bindings of proofs other than `actor_proofs[0]` are historical consent for their own hops.  The recipient MUST NOT evaluate them against the current outer token's audience or resources.
10.  Verify sibling references, when the token also carries `actor_receipts` validated under {{I-D.mcguinness-oauth-actor-receipts}}:
     *  for each index i covered by both arrays, when `actor_receipts[i]` carries `proof_jti`, it MUST equal `actor_proofs[i].jti`, and when `actor_proofs[i]` carries `receipt_jti`, it MUST equal `actor_receipts[i].jti`;
     *  a mismatched sibling reference MUST cause the recipient to reject both receipt-based and proof-based provenance for the token;
     *  a sibling reference that names an artifact at an index not covered by the other array is unverifiable; recipients whose policy requires bound siblings MUST reject the token's proof-based provenance, and other recipients MUST treat the reference as informational only;
     *  when receipts are absent or not validated, `receipt_jti` values are informational only.
11.  Apply any additional consumer-processing rules defined by companion profiles whose claims appear in the proof or outer token (see {{extensibility}}).  Companion-profile rules can add rejection conditions but cannot relax any requirement needed for conformance to this profile.

Step 1 is a prerequisite: an outer token that fails its own validation is rejected under the rules for its token type, not treated as lacking proofs.  If any later required check fails, the recipient MUST reject the proof chain and treat the token as lacking actor-signed evidence (step 2).  It rejects the token only when local policy or Protected Resource Metadata requires that evidence, using the underlying protocol's error handling for the stage at which the failure occurred.

A recipient that has rejected a proof chain under this profile MAY, under explicit local policy, extract structural information from the chain for use by companion profiles.  The recipient MUST NOT treat such partial validation as conformance with this profile, and MUST NOT relax the rejection requirements defined above.

## Subject Re-Expression Across Hops {#subject-re-expression-across-hops}

Older proofs can carry a different `sub` value from the current outer token when the subject has been re-expressed across issuer namespaces between hops.  This document does not define a universal subject-mapping algorithm.

Accordingly:

*  only `actor_proofs[0].sub` is required to equal the current outer token `sub`;
*  older proof `sub` values can differ (step 8 of {{consumer-processing}}).

Recipients need to be aware that permitting differing `sub` values across proofs creates a cross-subject insertion risk: a proof signed by a legitimate actor for an unrelated subject's delegation could satisfy the structural hop-alignment check when the actor identity at that hop matches.  An attacker who compromises any single actor signing key can deliberately sign proofs naming any subject and any target, and graft them onto a downstream chain whose re-expressed `sub` points at a victim subject.

This profile provides no in-band mechanism for cross-namespace subject reconciliation.

Deployments where subject continuity is a security requirement SHOULD adopt one of the following:

*  require exact, namespace-aware matching of subject identifiers across all proofs (the same `sub` under the same namespace authority; see `sub_iss` in {{identity-claims}}); or
*  enforce explicit trusted subject-mapping rules that can positively confirm each distinct subject identifier refers to the same underlying entity.

When neither condition is met, the recipient MUST treat subject continuity as unverified and MUST NOT rely on older proofs whose subject identifiers (`sub` or `sub_iss`) differ to support authorization that requires subject continuity (for example, a decision that treats every covered hop as having acted for the current token's subject).

## Complete Proof Coverage

Coverage is structurally complete when all validation succeeds and the proof count equals the visible actor depth.  This suffices when local policy requires only structural completeness.

When Protected Resource Metadata sets `actor_proofs_complete_required: true`, the token or introspection response MUST also carry `actor_proofs_complete: true`.  Recipients MUST reject tokens that fail the applicable completeness requirement.

## Use by Resource Servers

Resource servers can use validated proofs as evidence input for authorization, diagnostics, and audit, subject to the limits in {{threat-model}}.  However, a valid proof chain:

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

For opaque tokens, the issuer stores proofs and returns them to authorized resource servers through introspection.  The same format and consumer rules apply, using the response as the outer token's claim set.

An introspection response carrying proofs MUST include the members needed for {{consumer-processing}}: the token's top-level `sub`, the visible `act` chain, the token's `aud`, and the token's `iss`.  It SHOULD include `jti` when maintained by the server; without it, the response supplies no instance binding under step 9.  If present, `actor_proofs_complete` MUST be a boolean.

An RS receiving both inline and introspected proofs MUST select an authoritative source under local policy.  If it consumes both, differing arrays or completeness values MUST cause rejection of proof-based provenance.

An introspection server MUST return the full stored array or omit `actor_proofs`.  Removing an older entry breaks `prh`; removing the newest breaks hop alignment.  A server that filters the visible `act` chain can still return the full array when it filters only inner actors that no proof covers; if it filters a covered actor, it MUST omit both `actor_proofs` and `actor_proofs_complete`.  When the introspection server returns a stored array that it knows has partial coverage, it MUST include `actor_proofs_complete: false`.

For an inactive token, the introspection server MUST NOT return `actor_proofs` or `actor_proofs_complete`.

The core actor profile's `chain_complete` introspection member and `actor_proofs_complete` are distinct signals, exactly as described for receipts in {{I-D.mcguinness-oauth-actor-receipts}}: when `chain_complete: false`, proof coverage is complete only for the visible filtered chain, not the full delegation chain.

# Discovery and Capability Signaling {#discovery-capability-signaling}

This section defines metadata for advertising support for actor proofs.  It follows the claim-pair and discovery conventions defined in {{I-D.mcguinness-oauth-actor-receipts}}.

## Authorization Server Metadata

The following parameter is defined for use in Authorization Server Metadata {{RFC8414}}:

`actor_proofs_supported`:
: OPTIONAL.  A boolean.  When `true`, the authorization server advertises that it accepts the `actor_proof` token request parameter, validates proofs against actor keys, and embeds, preserves, or extends proof chains according to this document.  This value does not guarantee complete coverage for every visible hop in every resulting token.  When `false` or absent, the AS makes no claim of such support.

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
: OPTIONAL.  A boolean.  When `true`, the introspection response indicates that the returned `actor_proofs` cover every visible hop in the token chain as known to the introspection server.  When `false`, the response makes no attestation of complete coverage.

Consumer use of these members is described in {{consumer-introspection}}; introspection-server failure handling is addressed in {{introspection-errors}}.

## Out-of-Scope Discovery Signals

This document does not define metadata for actor-key source discovery; recipients establish actor-key sources through explicit trust frameworks ({{actor-key-resolution}}), not through metadata defined here.  It also does not define a metadata signal for requiring bound siblings (`proof_jti` on receipts); deployments that need bound siblings coordinate that requirement through deployment policy.

# Error Handling {#error-handling}

Proof validation failures use the underlying protocol's error mechanism for the stage at which validation occurs.

## Authorization Server and Transaction Token Service Errors

When an authorization server or Transaction Token Service rejects a token request because an inbound `actor_proofs` chain or a newly submitted proof cannot be validated (signature failure, key-resolution failure for an actor outside the trusted key sources, expired proof, unsupported `prh_alg`, broken `prh` chain, hop or subject misalignment), it returns an error response per {{RFC6749, Section 5.2}}: `invalid_request` for a Token Exchange request, as {{RFC8693, Section 2.2.2}} requires, or `invalid_grant` for a JWT bearer grant request ({{RFC7523, Section 3.1}}), consistent with the core actor profile's error mapping for actor information that fails validation.

When the requested target cannot be issued within the submitted proof's target binding after any narrowing under {{accepting-a-proof}}, the issuer SHOULD return `invalid_target`, per {{RFC8693, Section 2.2.2}} for a Token Exchange request and {{RFC8707, Section 2}} for other token requests.

When the failure reflects an actor-authorization decision rather than a structural validation failure, the issuer uses `actor_unauthorized`, as the core actor profile {{I-D.mcguinness-oauth-actor-profile}} requires.  An absent required proof, whether an `actor_proof` parameter or an inbound `actor_proofs` array, is an input-validation failure: the issuer returns `invalid_request` on a Token Exchange request and `invalid_grant` on a JWT bearer grant or refresh request.

## Resource Server Errors

When a resource server rejects a request because `actor_proofs` validation fails under {{consumer-processing}}, it SHOULD return `invalid_token` per the bearer-token error model in {{RFC6750}} Section 3.1.  For a Transaction Token, the recipient instead rejects the token through the deployment's Txn-Token handling, because {{I-D.ietf-oauth-transaction-tokens}} defines no error response for a rejected Transaction Token.

When the failure is specifically that required proofs are absent or coverage is incomplete (per `actor_proofs_required` or `actor_proofs_complete_required`), the resource server SHOULD include an `error_description` value identifying proof-coverage failure so that clients and operators can distinguish it from generic token-validation failures.

## Introspection Server Behavior {#introspection-errors}

When an introspection server cannot return proofs that the requesting resource server requires, it returns the introspection response per {{RFC7662}} with `actor_proofs` absent, or with a partial array and `actor_proofs_complete: false`; the resource server then applies its local policy to decide whether to accept the token.

The introspection server itself does not return an OAuth error for missing proofs; proof presence is a property of the introspection response, not a precondition for it.

## No New Error Codes

This document does not define new OAuth error codes.  The mapping above reuses existing codes from {{RFC6749}}, {{RFC6750}}, {{RFC8693}}, and the core actor profile.

# Extensibility {#extensibility}

This profile composes with the extensibility framework defined in {{I-D.mcguinness-oauth-actor-receipts}} and adds proof-specific extension surfaces:

*  **New claims inside a proof JWT** for additional per-hop actor-attested attributes.  Consumers ignore unrecognized claims under {{proof-claims}} unless another specification or local agreement defines their meaning.
*  **New `target` extension members** with constraining semantics, per the extension rule in {{proof-claims}}.  Specifications needing actor consent at scope or action granularity extend `target` rather than redefining it.
*  **Challenge and provisioning mechanisms** that supply prospective identifiers (the outer token's `jti`, a receipt's `jti`, or the newest inbound proof for opaque inbound tokens) to the actor before signing.  The `origin_jti` and `receipt_jti` claims are the designed insertion points; such mechanisms strengthen instance binding without changing proof processing.
*  **Actor events between hops**, such as consent to a changed target.  Companion profiles SHOULD use a JWT type distinct from `actor-proof+jwt`, a parallel outer-token array, and an anchor to a proof `jti` or defined flow identifier, following the non-hop event pattern of {{I-D.mcguinness-oauth-actor-receipts}}.  Companion profiles MUST NOT add event entries to `actor_proofs`, which is reserved for the actor-signed hop proofs defined by this document.
*  **Multi-actor co-signed hops** are out of scope for this document and would require a successor or companion profile with its own artifact structure.

Companion profile authoring rules:

*  Companion profiles MAY extend consumer processing under {{consumer-processing}} by adding rejection conditions; they MUST NOT relax any requirement needed for conformance to this profile.  This does not change the separately scoped partial-validation rule that follows step 11 of {{consumer-processing}}.
*  Companion profiles that define per-hop signed artifacts SHOULD follow the claim-pair and discovery conventions of {{I-D.mcguinness-oauth-actor-receipts}}, and MAY reuse the `prh` and `prh_alg` chain-linkage construction.

Conflict resolution: when a recipient implements multiple companion profiles whose rules conflict, local policy determines precedence.

# Security Considerations

Actor proofs strengthen delegation evidence with actor-side signatures, but they do not replace ordinary token validation.  The general OAuth 2.0 Security Best Current Practice {{RFC9700}} and the JWT best practices in {{RFC8725}}, except its audience requirements for proof JWTs (see `aud` in {{proof-claims}}), apply to systems implementing this profile.

## Threat Model {#threat-model}

The following threats and limits assume the trust and validation rules in this document.

### Adversaries Mitigated by This Profile

*  **Current outer-token issuer fabricating actor participation.**  Cannot forge the actor's proof signature at proof-covered hops.  Conditional on two recipient-side requirements: the recipient requires proofs for the tokens it accepts ({{downgrade-by-omission}}), and the recipient resolves the actor's key through a source independent of the issuer being defended against ({{actor-key-resolution}}).
*  **Current issuer exceeding the actor-authorized target at the covered hop.**  Issuer-side enforcement in {{accepting-a-proof}} and consumer step 9 detect an outer token whose audience or resources exceed the newest proof's signed target binding, subject to {{target-binding-strict-mode}}.
*  **Compromised downstream issuer fabricating prior-hop participation.**  Cannot forge prior actors' proof signatures; the `prh` chain prevents dropping or reordering inner proofs.
*  **Token mutation in transit.**  Each proof is independently signed; modification invalidates the proof's signature and any newer proof's `prh`.
*  **Partial-coverage misclaim.**  An issuer cannot drop an inner proof without breaking the `prh` chain; `actor_proofs_complete: true` cannot be claimed without a count matching visible chain depth.  An issuer can withhold coverage only from the innermost end of the chain, and only by beginning a new chain rather than trimming an inherited one, exactly as for receipts.
*  **Proof-chain substitution, when receipts with `proof_jti` are present.**  Replacing the proof array with a different harvested proof chain for the same visible hops mismatches the byte-preserved `proof_jti` values in the receipt chain and is rejected under step 10 of {{consumer-processing}}.

### Adversaries Not Mitigated

*  **Compromised actor signing key.**  Forged proofs are indistinguishable from legitimate ones and cannot be revoked individually.  Remediation: remove the key or actor from the trusted actor-key sources; short proof `exp` bounds the exposure window.
*  **Issuer omission of proofs.**  An issuer that fabricates participation simply omits `actor_proofs`.  Omission is a downgrade, not merely denial of service; the anti-fabrication property exists only for recipients that require proofs ({{downgrade-by-omission}}).
*  **Proof re-embedding within the validity window.**  A party that received a valid proof, including the issuer it was submitted to, can embed it in a different token with a matching subject, actor chain position, and target within the proof's `exp` window.  See {{proof-to-token-binding-limits}} for the binding limits and mitigations.
*  **Malicious or coerced actor.**  Proofs attest that the actor's key signed the participation; they do not attest intent, and they do not protect against an actor that colludes with a compromised issuer.  When issuer and actor are the same adversary, neither companion detects it.
*  **Cross-subject graft with a compromised actor key.**  Analogous to the receipts-side graft: see {{subject-re-expression-across-hops}}.
*  **Replay of an entire token plus its proofs.**  This profile does not define replay detection; proofs inherit the outer token's replay characteristics.

### Trust Model Summary

Trust is per-actor-key and per-deployment, and not transitive across the chain.  A proof chain fails validation if any proof's signing key cannot be resolved through an actor-key source the recipient trusts, even when the outer token's issuer and other proofs are trusted.  Proofs and receipts have independent trust anchors; validating both yields evidence that survives compromise of either the issuer side or the actor side, but not simultaneous compromise of both.

## Current Presenter Validation

The current request is always validated against the outer token's top-level `cnf` ({{RFC7800}}), when present, using the proof mechanism appropriate to the token type and deployment, such as DPoP {{RFC9449}} or mutual-TLS {{RFC8705}}.

An actor proof is never a substitute for that validation:

*  A recipient MUST NOT treat a proof signature as satisfying a proof-of-possession requirement for the current request, regardless of whether the proof signing key is the same key as a presenter key.
*  Recipients MUST distinguish proof JWTs (identified by the `typ` value `actor-proof+jwt`) from artifacts that carry current-request proof-of-possession semantics under {{RFC7800}}, {{RFC9449}}, or {{RFC8705}}.

## Actor Key Resolution and Trust {#actor-key-resolution}

Proof validation is meaningful only if the recipient resolves actor verification keys through sources it trusts.

Trust establishment requirements:

*  A recipient needs to establish its trusted actor-key sources before relying on `actor_proofs`, through explicit pre-configuration, bilateral agreement, federation policy, or another explicit trust framework.
*  A recipient MUST NOT treat the presence of a syntactically valid signed proof as sufficient grounds to trust the key that signed it.
*  Step 5 of {{consumer-processing}} checks that a proof's (`act.iss`, `act.sub`) pair is within the scope of a trusted actor-key source before any network retrieval keyed by the proof's content, and a recipient MUST NOT dereference key references supplied by the proof itself (such as `jku` or `x5u` header parameters) outside a pre-established trust framework, per {{RFC8725}}.
*  Key resolution and trust evaluation use the (`act.iss`, `act.sub`) pair.  The bare proof `iss` string is not a resolution index on its own ({{identity-claims}}); actor identifiers are namespaced by `act.iss`, and identical `act.sub` strings under different namespace authorities are different actors.

This document profiles the following resolution patterns; a deployment may support any subset:

*  **Pre-established keys.**  The actor's verification keys are registered with the recipient or its trust framework in advance, for example as the JWKS of a registered OAuth client at the authorization server, or as locally configured keys at a resource server.  This pattern is self-contained and provides the strongest independence properties.  Attestation-based client authentication {{I-D.ietf-oauth-attestation-based-client-auth}} provides an interoperable way to establish such keys, with an attester vouching for the actor's key binding.
*  **Federation and workload identity systems.**  Deployment-defined resolution through workload identity or federation infrastructure.  The trust and freshness properties are those of the underlying system; this document does not profile them.  OAuth SPIFFE client authentication {{I-D.ietf-oauth-spiffe-client-auth}} is an example of workload-identity key establishment that deployments can apply to actor signing keys.

The independence requirement follows from the threat model: for the anti-fabrication property against a given issuer to hold at a hop, the recipient MUST resolve the actor's key for that hop through a source independent of that issuer.

Actor keys, like receipt-issuer trust, are not transitive: each proof is validated against the recipient's own actor-key sources, independent of the outer token's issuer and of neighboring proofs.  If any proof in the presented `actor_proofs` array is signed by a key the recipient cannot resolve through a trusted source, step 5 of {{consumer-processing}} fails and the proof chain is rejected for the purposes of this profile.

## Proof-to-Token Binding Limits {#proof-to-token-binding-limits}

Without instance binding, a proof records a subject, actor, target, and validity window.  Any party holding it, including the issuer it was legitimately submitted to, can reuse it in another token matching that context without invalidating its signature.  The signature therefore evidences consent to that context, not to a particular token issuance.

Available bindings, strongest first:

*  **Receipts composition.**  When the token also carries receipts, the receipt chain's `origin_jti` anchoring and strict-mode rules in {{I-D.mcguinness-oauth-actor-receipts}} bind the token instance, and `proof_jti` ({{sibling-receipt-issuance}}) binds the proof chain to that anchored receipt chain.  A re-embedded proof would require a matching fabricated receipt, which the receipt trust model prevents for issuers that cannot sign trusted receipts.  This is the RECOMMENDED posture for deployments that require instance binding.
*  **Provisioned `origin_jti`.**  When the issuance flow provides the prospective outer-token `jti` to the actor before signing, `actor_proofs[0].origin_jti` binds the proof to that token instance directly, per step 9 of {{consumer-processing}}.
*  **`jti` uniqueness monitoring.**  Recipients and audit pipelines MAY track proof `jti` values and flag the same proof appearing in more than one outer-token instance.  This is stateful and deployment-specific; this document does not define the mechanism.
*  **Short `exp`.**  Bounds the re-embedding window unconditionally, at the cost of shorter delegated-session lifetimes ({{proof-claims}}).

Inner proofs have no independent binding to the current token; they are bound to their newer neighbor through `prh` and inherit whatever binding `actor_proofs[0]` has.

### Target-Binding Strict Mode {#target-binding-strict-mode}

An outer token diverges from its proof chain when its audience or effective resources exceed `actor_proofs[0]`'s target binding, or when its `jti` differs from a present `actor_proofs[0].origin_jti`.  A recipient MUST reject a divergent proof chain unless local policy designates the outer token issuer as a trusted reissuing issuer.  Designation as a trusted reissuing issuer excuses only `jti` divergence unless local policy also permits that issuer to retarget.  Recipients that have not explicitly configured a set of trusted reissuing issuers therefore operate in strict mode by default, rejecting every divergent chain.

Strict mode is the recommended default.  Deployments accepting retargeted reissuance need an explicit set of trusted reissuing issuers, configured through local policy or an out-of-band trust framework.  A recipient accepting target divergence MUST treat proofs only as participation evidence and MUST NOT infer consent to the current audience or resources.  With receipts, it SHOULD apply one reissuance-trust decision to both companions.

## Hash Algorithm Agility

The `prh` and `prh_alg` agility rules of {{I-D.mcguinness-oauth-actor-receipts}} apply to proof chains unchanged: one algorithm per chain, whole-chain migration only, no rehashing of inherited artifacts, and rejection of mixed or unsupported algorithms.  The proof chain's algorithm is independent of the receipt chain's algorithm in the same token.

## Downgrade by Omission {#downgrade-by-omission}

This profile's anti-fabrication property is conditional on enforcement.  A compromised issuer that wants to fabricate actor participation does not submit a forged proof, which would fail signature validation; it omits `actor_proofs` entirely.  A recipient that accepts delegated tokens without proofs has no protection from this profile against that issuer.

Accordingly:

*  Resource servers that rely on actor-signed evidence MUST require proofs, through `actor_proofs_required` (and `actor_proofs_complete_required` where inner hops matter) or equivalent local policy, for the delegated tokens they accept.
*  Recipients SHOULD treat the absence of proofs from an issuer that advertises `actor_proofs_supported: true`, for a resource that requires them, as a signal warranting scrutiny rather than silent acceptance.

Issuers cannot protect recipients that do not ask; the enforcement locus of this profile is the recipient.

## Actor Key Compromise

If an actor's signing key is compromised, previously signed proofs and newly forged proofs under that key are indistinguishable.  The primary remediation is to remove the compromised key or actor from the recipient's trusted actor-key sources; once removed, consumers will reject all proofs attributed to that actor's key regardless of content.

Proof `exp` is sized under the conditional rule in {{proof-claims}}, which calls for short values without instance binding; a shorter `exp` limits the window during which proofs signed with a compromised key remain valid.  When a key compromise is detected, deployments SHOULD treat tokens carrying proofs from the affected actor as lacking trusted actor-signed evidence for those hops and SHOULD require fresh delegation with fresh proofs.

## Proof Chain Size

Each proof is a full signed JWT, and the chain grows linearly with delegation depth.  A typical signed proof is 400 to 800 bytes after JWS compact serialization and base64url encoding, comparable to a receipt.  A token carrying both companions carries roughly twice the per-hop artifact bytes of either companion alone.  Deployments SHOULD verify that the outer token plus its companion arrays fits within the header-size budget of every component on the request path; the receipts companion's size guidance applies, and introspection delivery avoids header pressure for bearer-token clients.

## Proof Freshness and Replay {#proof-freshness}

Proofs are historical attestations of hop-time consent.  They can outlive the validity period of the outer token they were originally embedded in, and can be carried forward across reissuance and refresh only in tokens that expire no later than they do ({{reissuance-without-a-new-actor-hop}}).

*  Proofs attest participation and target consent at signing time; they do not assert that the represented delegation is still active or that the actor would consent today.
*  Runtime policy evaluation, including current authorization and current revocation state, is separate from proof validation.
*  Replay of an entire token plus its proofs is governed by the outer token's replay characteristics.  Re-embedding of an individual proof into a different token is a distinct threat, bounded as described in {{proof-to-token-binding-limits}}.

Deployments needing freshness signals beyond proof `exp` MUST obtain those signals from the authorization server via introspection ({{RFC7662}}), fresh token issuance, or another mechanism outside the scope of this profile.

## Sibling Revocation Independence

A proof does not inherit revocation or trust state from its sibling receipt.  When a deployment removes a receipt issuer from its trusted-issuer set under {{I-D.mcguinness-oauth-actor-receipts}}, proofs for the same hops remain valid under their own actor keys, and vice versa.  Recipients that require issuer-side revocation semantics for a hop MUST require receipt validation alongside proof validation rather than relying on the proof alone; the sibling references of {{sibling-receipt-issuance}} identify the receipt whose trust state applies to a hop.

# Privacy Considerations

Proof signatures provide transferable evidence of actor participation and target consent.  Any holder able to resolve the actor's public key can verify that evidence, including after token expiration.

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

Unintended recipients can also verify proofs if they can resolve trusted actor keys.  Deployments SHOULD treat proof-bearing tokens as carrying durable, transferable participation evidence to anywhere the token reaches, and SHOULD scope token distribution and retention accordingly.

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
*  Change Controller: IETF
*  Specification Document(s): This document

*  Claim Name: `actor_proofs_complete`
*  Claim Description: Boolean indicating whether actor_proofs covers every visible hop in the token's act chain
*  Change Controller: IETF
*  Specification Document(s): This document

*  Claim Name: `target`
*  Claim Description: Target binding (audience and resource constraints) authorized by the signer of an Actor Proof JWT
*  Change Controller: IETF
*  Specification Document(s): This document

*  Claim Name: `receipt_jti`
*  Claim Description: jti of the sibling Actor Receipt JWT created for the same delegation hop as an Actor Proof JWT
*  Change Controller: IETF
*  Specification Document(s): This document

*  Claim Name: `proof_jti`
*  Claim Description: jti of the sibling Actor Proof JWT validated for the same delegation hop as an Actor Receipt JWT
*  Change Controller: IETF
*  Specification Document(s): This document

This document reuses the `prh`, `prh_alg`, `origin_jti`, and `sub_iss` claims registered by {{I-D.mcguinness-oauth-actor-receipts}}, with the semantics defined there, applied to Actor Proof JWTs as profiled in this document.  This document requests that IANA add this document to the Specification Document(s) entries for those four registrations, and requests that their Claim Description entries be updated to cover both artifact types:

*  `prh`: Base64url-encoded hash of the immediately preceding (older) entry in a chained array of delegation-evidence JWTs (Actor Receipts, Actor Proofs, or companion event artifacts)
*  `prh_alg`: Hash algorithm identifier (from the IANA Named Information Hash Algorithm Registry) naming the algorithm used to compute prh in a delegation-evidence JWT
*  `origin_jti`: The jti of the outer token associated with the hop at which an Actor Receipt or Actor Proof JWT was created
*  `sub_iss`: Issuer or namespace authority for the subject in an Actor Receipt or Actor Proof JWT

## OAuth Parameters Registration

This document requests registration of the following parameter in the "OAuth Parameters" registry established by {{RFC6749}}:

*  Parameter name: `actor_proof`
*  Parameter usage location: token request
*  Change Controller: IETF
*  Specification Document(s): This document

## OAuth Authorization Server Metadata Registration

This document requests registration of the following metadata name in the "OAuth Authorization Server Metadata" registry {{RFC8414}}:

*  Metadata Name: `actor_proofs_supported`
*  Metadata Description: Indicates support for accepting, validating, embedding, preserving, and extending actor-signed hop proofs
*  Change Controller: IETF
*  Specification Document(s): This document

## OAuth Protected Resource Metadata Registration

This document requests registration of the following metadata names in the "OAuth Protected Resource Metadata" registry {{RFC9728}}:

*  Metadata Name: `actor_proofs_required`
*  Metadata Description: Indicates that the resource expects delegated requests to carry valid actor proofs covering at minimum the outermost visible actor hop
*  Change Controller: IETF
*  Specification Document(s): This document

*  Metadata Name: `actor_proofs_complete_required`
*  Metadata Description: Indicates that the resource requires complete proof coverage for all visible actor hops
*  Change Controller: IETF
*  Specification Document(s): This document

## OAuth Token Introspection Response Registration

This document requests registration of the following names in the "OAuth Token Introspection Response" registry {{RFC7662}}:

*  Name: `actor_proofs`
*  Description: Array of actor-signed hop proofs returned by introspection
*  Change Controller: IETF
*  Specification Document(s): This document

*  Name: `actor_proofs_complete`
*  Description: Indicates whether the returned actor proofs provide complete visible-hop coverage
*  Change Controller: IETF
*  Specification Document(s): This document

# Acknowledgments

This document builds on the OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}}, on the OAuth Actor Receipts companion {{I-D.mcguinness-oauth-actor-receipts}}, on the OAuth 2.0 Token Exchange specification {{RFC8693}}, on the OAuth 2.0 Transaction Tokens work {{I-D.ietf-oauth-transaction-tokens}}, and on prior OAuth Working Group discussion of delegation transparency, sender-constrained tokens, and proof-of-possession mechanisms ({{RFC7800}}, {{RFC8705}}, {{RFC9449}}).  Related actor-evidence efforts and their relationship to this document are discussed in {{related-work}}.

Contributors and reviewers will be acknowledged in future revisions.

--- back

# Examples

The examples in this appendix show decoded proof contents.  Real proofs are compact-signed JWT strings carried in the `actor_proofs` array.  The `iat` and `exp` values shown are illustrative only; in deployments, proof `exp` is set per {{proof-claims}} and {{extending-an-existing-proof-chain}} so that no inbound proof expires before the outer token that carries it.  The delegation scenario continues the two-hop travel example of {{I-D.mcguinness-oauth-actor-receipts}}: the subject alice delegates to an AI travel-assistant agent through the enterprise AS, and the agent's token is exchanged at the travel-provider AS, which adds a booking tool as the new outermost actor.

## Example: Two-Hop Delegation Chain with Sibling Receipts

The outer token carries both companions:

~~~json
{
  "jti": "e8f4a2d6-3b1c-4d7e-9f5a-0c2b4d6e8f0a",
  "iss": "https://as.travel-provider.example",
  "aud": "https://api.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://tools.travel-provider.example/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service",
    "act": {
      "sub": "https://agents.enterprise.example/travel-assistant",
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
  "iss": "https://tools.travel-provider.example/booking-tool",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://tools.travel-provider.example/booking-tool",
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
  "iss": "https://agents.enterprise.example/travel-assistant",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_iss": "https://idp.enterprise.example",
  "act": {
    "sub": "https://agents.enterprise.example/travel-assistant",
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

The sibling receipts follow the same construction as the examples of {{I-D.mcguinness-oauth-actor-receipts}}, with the `proof_jti` claim defined in {{sibling-receipt-issuance}} included at receipt creation.  Because these receipts carry `proof_jti`, they are different byte strings from the receipts shown in that document's examples: they carry their own `jti` values, and the newest receipt's `prh` differs because it hashes a different older receipt.  The newest receipt, signed by the travel-provider AS, carries:

~~~json
{
  "iss": "https://as.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://tools.travel-provider.example/booking-tool",
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

The example verifies as follows:

*  The current audience is within `actor_proofs[0].target.aud`.
*  The older proof's target records consent for its own hop and is not compared with the current audience.
*  Receipt `proof_jti` links the two chains, and the leading receipt's `origin_jti` anchors the current token instance.

The recipient verifies each proof against a pre-established key registered for its actor ({{actor-key-resolution}}).  Receipt binding does not prevent a compromised issuer from signing a replacement receipt.

## Example: Proofs-Only Partial Coverage

Suppose the enterprise AS has not yet deployed proof support, so no proof exists for the AI-agent hop, and the travel-provider AS accepts a proof from the booking tool when it adds the tool as the new outermost actor.  The resulting access token carries a one-element proof chain and no receipts:

~~~json
{
  "jti": "a4c8e2f6-1b3d-4e5f-9a7c-8d6e4f2a0b1c",
  "iss": "https://as.travel-provider.example",
  "aud": "https://api.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://tools.travel-provider.example/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service",
    "act": {
      "sub": "https://agents.enterprise.example/travel-assistant",
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
  "iss": "https://tools.travel-provider.example/booking-tool",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://tools.travel-provider.example/booking-tool",
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

`prh` is omitted because this is a single-element chain.  `actor_proofs_complete: false` signals to recipients that the inner AI-agent hop carries no actor-signed evidence.  Resource servers that set `actor_proofs_complete_required: true` in their Protected Resource Metadata reject this token; resource servers that accept partial coverage validate the booking tool's signed participation and its consent to the token's audience, and treat the agent hop as carried solely by the visible `act` chain.  The token carries no resource claim, so a recipient that cannot determine its effective resources from other context does not infer consent to `target.resource` (step 9 of {{consumer-processing}}).  Because no receipts are present, the proof chain carries no outer-token instance binding; per {{proof-to-token-binding-limits}}, a recipient requiring instance binding would require the receipts companion or a provisioned `origin_jti`.

# Document History
{:numbered="false"}

[[ To be removed from the final specification ]]

-01

* Consolidated and tightened the text throughout.
* An issuer now drops inherited proofs when reissuance exceeds any part of the newest proof's target, not only its audience.
* Distinguished a mismatched `origin_jti`, which consumer processing rejects unless the outer issuer is a trusted reissuer, from an absent one.
* Removed an example claim that receipt composition stops a compromised issuer from re-embedding a proof.
* Reconciled `exp` guidance and removed BCP 14 keywords from storage, trust-setup, and rollout guidance.
* An expired proof, including one for an older hop, is now invalid; only the clock-skew leeway of {{RFC7519, Section 4.1.4}} applies.
* Consolidated duplicated requirements into single homes and cited dependencies instead of restating them.
* Resolved the remaining duplicate-rule conflicts: companion rules cannot relax conformance requirements, and Strict Mode governs every divergence.
* Removed the unconditional recommendation for short proof `exp` in favor of the claim's conditional sizing rule.
* Aligned subject-continuity handling and the proof actor object's `sub_profile` rule with Receipts.
* An introspection server that returns a stored array it knows has partial coverage is now required to include `actor_proofs_complete: false`.
* Used the base profile's example identifiers for the travel assistant and booking tool.
* Clarified that proof actor-object restrictions apply separately from confirmation extensions in the token's actor chain.
* Prohibited `aud` in proofs (-00 discouraged it); consumers reject a proof that carries it, and {{RFC8725}} applies except {{RFC8725, Section 3.9}}.
* One lifetime rule now governs hop extension, reissuance, and refresh when a retained proof expires before the token the issuer would set: lower the token's `exp`, otherwise drop the proofs where policy permits absent coverage, otherwise fail with `invalid_grant` on refresh or a JWT bearer grant or `invalid_request` on Token Exchange.
* Proof validation failures on Token Exchange requests now use `invalid_request`, as {{RFC8693, Section 2.2.2}} requires; JWT bearer grant requests use `invalid_grant` ({{RFC7523, Section 3.1}}).
* A resource indicator is within a proof's target binding only when it equals an entry of `target.resource` under simple string comparison ({{RFC3986, Section 6.2.1}}).
* When `target.resource` is present and the request supplies no resource indicators, the issuer uses `target.resource` as the effective resources; a recipient that cannot determine the token's resources treats the proof as audience-level consent.
* Named the IETF, rather than the IESG, as change controller for the claim, parameter, metadata, and introspection registrations.
* A failed proof check, including a false `actor_proofs_complete: true`, now drops only the actor-signed evidence; an outer token that fails its own validation is still rejected, and otherwise the recipient rejects the token only when policy or metadata requires that evidence, and an issuer that cannot propagate inbound proofs can begin a partial chain at its own hop.
* A reissued or refreshed token with a new `jti` diverges from a present `actor_proofs[0].origin_jti`; trusted-reissuer designation excuses only that divergence unless policy also permits retargeting, and proof `exp` covers a delegated session only while the outer token stays instance-bound.
* Removed receipt-attested presenter keys as an actor-key resolution pattern; the examples use pre-established keys.
* An issuer that adds a hop without a valid new proof drops the inbound proofs, and a request that adds no hop but carries `actor_proof` is rejected with `invalid_request`.
* A Transaction Token Service can include `jti` so that a Transaction Token's proof chain can be instance-bound; resource servers reject a Transaction Token through the deployment's Txn-Token handling.
* An extending issuer sets `actor_proofs_complete: true` only when every inbound proof validated and the proof count equals the visible actor-chain depth.
* A proof chain fails validation when any proof's signing key is untrusted.
* Consumers check `alg` and `typ` before key resolution and signature validation ({{RFC8725, Section 3.1}}).
* Issuers use `actor_unauthorized` for actor-authorization failures, as the core actor profile requires, and treat an absent required proof as an input-validation failure.
* An introspection server that filters a proof-covered actor from the visible `act` chain omits `actor_proofs` and `actor_proofs_complete`.

-00

* Initial version.

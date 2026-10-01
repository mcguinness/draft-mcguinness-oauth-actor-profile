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

  I-D.mora-oauth-entity-profiles:
    title: "OAuth Entity Profiles"
    author:
     -
        fullname: Sreyantha Chary Mora
        organization: Microsoft
     -
        fullname: Pamela Dingle
        organization: Microsoft
     -
        fullname: Karl McGuinness
        organization: Independent
    date: 2026-04-17
    seriesinfo:
      Internet-Draft: draft-mora-oauth-entity-profiles-01
    target: https://www.ietf.org/archive/id/draft-mora-oauth-entity-profiles-01.txt
informative:
  RFC9700:
  I-D.mw-oauth-actor-chain:
  I-D.liu-oauth-chain-delegation:
  I-D.jiang-oauth-intent-admission:
  I-D.ietf-oauth-attestation-based-client-auth:
  I-D.ietf-oauth-spiffe-client-auth:

...

--- abstract

This document defines OAuth Actor-Signed Hop Proofs, an optional companion to the OAuth Actor Profile for Delegation.  Each proof is a signed JSON Web Token (JWT) recording an actor's participation and authorized target for one hop.  The `actor_proofs` claim carries a hash-linked chain verified through trusted actor-key sources.  This document specifies proof conveyance, validation, optional links to Actor Receipts, metadata parameters, and introspection response members.

--- middle

# Introduction

The OAuth Actor Profile for Delegation {{I-D.mcguinness-oauth-actor-profile}} makes actor identity visible in delegated tokens through a common `act` claim.  The OAuth Actor Receipts companion {{I-D.mcguinness-oauth-actor-receipts}} adds authorization-server-signed per-hop provenance.  Both are issuer assertions: an authorization server attests that an actor was added at a hop.  Nothing in either profile requires the actor's own cryptographic participation, so a compromised or dishonest issuer can fabricate the participation of an actor that never authorized the delegation.

This document defines OAuth Actor-Signed Hop Proofs, an optional companion profile that adds actor-side evidence.  At each covered hop, the actor signs its participation and authorized target; the AS validates the proof and includes it in `actor_proofs`, and recipients verify the actor's signature through trusted key sources.  The design center is:

*  keep the visible actor chain in `act`;
*  keep authorization-server-signed provenance in `actor_receipts` when the receipts companion is in use;
*  carry actor-signed participation and hop-time target consent in separately signed proofs.

This profile adds the `actor_proof` token request parameter, the `actor_proofs` and related claims, and discovery metadata.  It does not define transparency logging of proofs.

## Relationship to the Actor Receipts Companion {#relationship-to-receipts}

The AS signs receipts; the actor signs proofs.  A token MAY carry either, both, or neither, and recipients select a validation posture by local policy:

*  **Receipts-only**: proofs are absent or ignored; trust follows {{I-D.mcguinness-oauth-actor-receipts}}.
*  **Proofs-only**: receipts are absent or ignored; trust rests on actor-key resolution and actor signatures.
*  **Belt-and-suspenders**: both are validated, providing independent issuer-side and actor-side attestations for covered hops, linked by the sibling references in {{sibling-receipt-issuance}}.

Receipts and proofs are separate compact JWTs, rather than one JWS with both signatures over a shared payload (JWS JSON Serialization, {{Section 7.2 of RFC7515}}), because their signers, adoption prerequisites, and threat models differ.  Receipts require only issuer support, whereas proofs also require actors capable of signing and a trusted actor-key source for every actor whose proof a recipient uses ({{actor-key-resolution}}), which can require more configuration than trusting the smaller set of receipt issuers; deployments can rely on receipts for actors without signing keys.  Separate artifacts keep the two trust anchors independent ({{threat-model}}) and let deployments adopt, validate, and hash-chain issuer and actor evidence independently.

## Relationship to Other Actor-Evidence Work {#related-work}

Several concurrent efforts add actor-side or issuer-side delegation evidence to OAuth deployments; they differ from this profile chiefly in where the evidence is carried, who signs it, and who can verify it.

*  {{I-D.mw-oauth-actor-chain}} retains actor-signed step proofs at the AS and carries an issuer-signed cumulative commitment in the token, so actor-signature verification depends on AS retention.  This profile instead carries the proofs for direct recipient verification, increasing token size at each hop.
*  {{I-D.liu-oauth-chain-delegation}} carries AS-signed hop records inline, optionally countersigned by the delegator.  Those fields carry no target binding, and records are re-signed at domain boundaries.
*  {{I-D.jiang-oauth-intent-admission}} defines a single-hop intent artifact signed by the admission authority.

The distinguishing property of this profile is a recipient-verifiable artifact, signed by the actor itself before issuance, over an explicit target binding.  This comparison is intended to inform, not preempt, working group discussion of convergence.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses OAuth terminology from {{RFC6749}} and {{RFC8693}}, and Transaction Token terminology from {{I-D.ietf-oauth-transaction-tokens}}.  AS, RS, and TTS denote authorization server, resource server, and Transaction Token Service, respectively.  Actor Receipt and Receipt Chain are defined in {{I-D.mcguinness-oauth-actor-receipts}}.  Outer Token denotes the token associated with a proof chain, whether or not it also carries receipts.

The following terms are used in this document:

Actor Proof:
: A JWT created and signed by the actor added at one visible actor hop, attesting to that actor's participation and the target binding it authorized for that hop.

Proof Chain:
: The ordered `actor_proofs` array carried in a token or introspection response.

Actor Signing Key:
: An asymmetric key controlled by an actor and used to sign actor proofs.  This document does not standardize how actor signing keys are established; see [Actor Key Resolution and Trust](#actor-key-resolution).

Actor-Key Source:
: A mechanism, trusted by a recipient under explicit local policy, that resolves an actor identifier pair (`act.iss`, `act.sub`) to one or more actor verification keys.

Target Binding:
: The audience and optional resource constraints that the actor authorized for the token issued at its hop, carried in the proof's `target` claim.  A target binding records hop-time consent; it is not an audience restriction on the proof artifact itself.

Sibling Receipt:
: The Actor Receipt, if any, created under {{I-D.mcguinness-oauth-actor-receipts}} for the same visible actor hop as a proof.

Complete Proof Coverage:
: The condition in which the number of proofs in the `actor_proofs` claim equals the number of visible actor hops in the token's `act` chain, and every proof aligns with the corresponding visible hop.

Examples in this document are illustrative and omit unrelated claims, signatures, and validation steps that a complete deployment would need.

# The `actor_proofs` Claim {#actor-proofs-claim}

The `actor_proofs` claim is a new top-level JWT claim for tokens that conform to the core actor profile and to this companion profile.

`actor_proofs`:
: OPTIONAL.  An array of strings.  Each string MUST be the compact serialization of a signed JWT proof as defined in {{actor-proof-jwt-format}}.  When present, the array:

  *  MUST NOT be empty; issuers MUST omit the claim rather than including an empty array;
  *  MUST be ordered from newest covered hop to oldest covered hop;
  *  MUST NOT contain more entries than the visible actor-chain depth of the token's `act` claim;
  *  MUST represent a contiguous outermost prefix of the visible `act` chain.

If a token carries the `actor_proofs` claim, it MUST also carry an `act` claim conforming to the core actor profile.

`actor_proofs_complete`:
: OPTIONAL.  A boolean JWT claim in the outer token.  When `true`, the issuer attests that the `actor_proofs` claim covers every visible hop in the token's `act` chain.

  The attestation is relative to the visible chain at issuance time; it does not attest that the visible chain is itself unfiltered (see `chain_complete` in the core actor profile {{I-D.mcguinness-oauth-actor-profile}}).  Consumer enforcement, including the count-equality check, is defined in step 4 of {{consumer-processing}}.

  Issuers SHOULD set `actor_proofs_complete` to `true` for complete coverage and `false` for partial coverage.  An absent value provides no completeness attestation; consumers that require the literal value `true` treat absence as `false`.

# Actor Proof JWT Format {#actor-proof-jwt-format}

Each element of the `actor_proofs` array is a signed JWT represented using the JWS compact serialization {{RFC7515}}.

## JOSE Header

The JOSE header of an actor proof:

*  MUST include an asymmetric digital-signature `alg` value;
*  MUST NOT use `alg: none` or a MAC-based symmetric algorithm;
*  MUST include `typ` with the value `actor-proof+jwt`;
*  SHOULD include `kid` when the actor's key source publishes multiple verification keys;
*  MAY include `crit`; a proof whose `crit` header lists an extension header the consumer does not understand is invalid per {{Section 4.1.11 of RFC7515}}.

Actors, issuers, and consumers MUST apply the JWT best practices in {{RFC8725}} when creating and validating proofs, except for the audience requirements of {{Section 3.9 of RFC8725}}, from which this profile departs by prohibiting `aud` as described in {{proof-claims}}.

## Proof Claims {#proof-claims}

### Identity Claims

`iss`:
: REQUIRED.  The identifier of the actor that signed the proof.  It MUST equal the proof's `act.sub` value.

  The `iss` value is interpreted within the namespace given by `act.iss`.  A bare `iss` MUST NOT serve as the sole key-resolution or trust index; use (`act.iss`, `act.sub`) as specified in {{actor-key-resolution}}.

`sub`:
: REQUIRED.  The subject identifier on whose behalf the actor authorized the delegation, as known to the actor at signing time.  `actor_proofs[0].sub` MUST equal the outer token's top-level `sub`.  Older proofs' `sub` values can differ ({{subject-re-expression-across-hops}}).

`sub_iss`:
: OPTIONAL.  The namespace authority under which the proof's `sub` value is interpreted, with the semantics defined for the `sub_iss` claim in {{I-D.mcguinness-oauth-actor-receipts}}.  When `sub_iss` is absent, the namespace is determined as for a receipt that omits it.

`act`:
: REQUIRED.  A single-hop actor object identifying the signing actor.  This object:

  *  MUST conform to the core actor profile's actor-object rules;
  *  MUST include `act.sub` and `act.iss`;
  *  MUST NOT contain `cnf`;
  *  MUST NOT contain a nested `act`.

  The `act` claim supplies the namespace context and visible-hop alignment.

  The token's `act` chain can retain confirmation members as extension data under the core actor profile; the proof's actor object omits them and still satisfies visible-hop alignment (step 7 of {{consumer-processing}}).

This profile defines no subject `sub_profile` claim for proofs; subject classification remains issuer-asserted.  Actor classification can appear in `act.sub_profile`.

### Target Binding

`target`:
: REQUIRED.  A JSON object recording the target binding the actor authorized for the token issued at this hop.  Its members are:

  `target.aud`:
  : REQUIRED.  A string or array of strings.  The audiences the actor authorizes for the token issued at this hop.

  `target.resource`:
  : OPTIONAL.  An array of URIs with the semantics of the `resource` request parameter of {{RFC8707}}.  When present, it narrows the target binding beyond `target.aud`.

  Issuer-side enforcement at the covered hop is defined in {{accepting-a-proof}}, and consumer evaluation in step 9 of {{consumer-processing}}.

  Other specifications MAY define extension members.  An extension member MUST be defined with constraining semantics only: its presence narrows what the actor authorized and its absence leaves the binding as expressed by the defined members.  Consumers MUST ignore unrecognized `target` members unless another specification or local agreement defines their meaning; actors MUST NOT rely on unrecognized extension members being enforced.

### Chain Linkage

`prh`:
: OPTIONAL.  The previous proof hash, computed over the next older proof in the chain; the oldest proof, including the sole proof of a single-element chain, omits it.

  This profile reuses the `prh` and `prh_alg` claims from {{I-D.mcguinness-oauth-actor-receipts}} with the same construction, applied to proof JWTs.  The proof chain is linked independently of any receipt chain carried in the same token: each companion's `prh` values hash that companion's own artifacts.

`prh_alg`:
: OPTIONAL.  An identifier naming the hash algorithm used to compute `prh`.  The value, consistency, and extension rules of the receipt `prh_alg` claim apply to proof chains.

  *  When absent, the default is `sha-256`.
  *  The proof chain's `prh_alg` is independent of the receipt chain's `prh_alg` in the same token; the two chains MAY use different algorithms.

### Sibling Receipt Reference

`receipt_jti`:
: OPTIONAL.  The `jti` of the sibling receipt created for the same hop under {{I-D.mcguinness-oauth-actor-receipts}}.

  To include this claim, the actor needs a prospective receipt identifier.  Most deployments instead use the receipt's `proof_jti` claim, which the issuer sets after validating the proof ({{sibling-receipt-issuance}}).  Consumer step 10 validates both references.

### Time and Uniqueness

`iat`:
: REQUIRED.  The time at which the proof was signed, as defined in {{RFC7519}}.

`exp`:
: REQUIRED.  Expiration time for the proof, as defined in {{RFC7519}}.

  The `exp` value needs to cover the lifetime of any token that will carry or inherit this proof ({{issuer-processing}}).  Longer validity supports delegated sessions but also extends exposure to key compromise and proof reuse ({{proof-to-token-binding-limits}}).

  With instance binding through receipts in strict mode or a provisioned `origin_jti` ({{proof-to-token-binding-limits}}), `exp` MAY cover the delegated session only while the outer token stays instance-bound.  Refresh or reissuance ends instance binding, so issuers that refresh tokens carrying proofs SHOULD keep proof `exp` short.  Without instance binding, `exp` SHOULD be short to limit proof reuse.

`jti`:
: REQUIRED.  A unique identifier for the proof, as defined in {{RFC7519}}.

### Outer-Token Binding

`origin_jti`:
: OPTIONAL.  The `jti` of the outer token issued at the hop this proof covers, following the pattern of the `origin_jti` claim defined in {{I-D.mcguinness-oauth-actor-receipts}}.

  This requires provisioning the prospective token identifier before signing.  Consumer step 9 evaluates it; see also {{proof-to-token-binding-limits}}.

  A Transaction Token Service MAY include `jti` in a Transaction Token, because {{Section 9.2 of I-D.ietf-oauth-transaction-tokens}} permits additional claims; without it, a Transaction Token's proof chain is never instance-bound.

### Excluded Standard Claims

`aud`:
: Prohibited.  Actors MUST NOT include `aud` in a proof, and consumers MUST reject a proof that carries it (step 5 of {{consumer-processing}}).

  Proofs are validated as part of outer-token processing, not as independent JWTs against an audience; the outer token carries the audience scoping for the request, and the actor's consented audiences are carried in `target.aud`.  This profile departs from {{Section 3.9 of RFC8725}} on those grounds.  Rejecting a proof that carries `aud` produces the result that {{Section 4.1.3 of RFC7519}} requires when the processing principal does not identify itself with the `aud` value.

### Extension Claims

A proof MAY contain additional claims defined by another specification or by deployment policy.  Consumers ignore unrecognized claims unless another specification or local agreement defines their meaning, per {{Section 4 of RFC7519}}.

## Proof-Chain Linkage {#proof-chain-linkage}

When an actor signs a new proof that extends an inherited proof chain:

*  if there is an older proof immediately following it in the array, the new proof MUST include `prh`, and that value MUST be the base64url encoding, without padding, of the hash of the ASCII octets of the exact compact JWT string of that next proof;
*  if the new proof is the only proof in the array, it MUST omit `prh`.

The hash input is the exact compact JWS string, without JSON {{RFC8259}} canonicalization.  Systems that carry, store, or forward `actor_proofs` arrays MUST preserve each proof byte-for-byte.

# Conveying Proofs at Issuance {#actor-proof-parameter}

This document defines one token request parameter:

`actor_proof`:
: OPTIONAL.  The compact serialization of a single actor proof JWT for the new outermost actor hop of the requested token.  A request carries at most one `actor_proof` parameter ({{Section 3.2 of RFC6749}}).

The parameter is defined for token endpoint requests that produce delegated tokens under the core actor profile, including OAuth 2.0 Token Exchange {{RFC8693}} requests and JWT assertion grants.  A Transaction Token request made over HTTP is a Token Exchange request ({{Section 11 of I-D.ietf-oauth-transaction-tokens}}) and carries the proof in the `actor_proof` parameter; other Transaction Token Service interfaces convey the proof by equivalent means.

The AS authenticates the actor and derives its identity under the core actor profile, then separately validates the proof ({{accepting-a-proof}}).  The `actor_proof` parameter supplies participation and consent evidence; it does not serve as the `actor_token` parameter and does not authenticate the request.

This document does not define a challenge mechanism by which an authorization server provides prospective values (such as the outer token's `jti`, a receipt's `jti`, or the newest inbound proof for opaque inbound tokens) to the actor before signing.  Deployments and companion profiles MAY define such mechanisms; the `origin_jti` and `receipt_jti` claims are the intended insertion points.

# Issuer Processing {#issuer-processing}

This section defines how an authorization server or Transaction Token Service accepts, validates, embeds, preserves, and extends `actor_proofs`.

When a proof that the issuer retains from an inbound token or refresh state has an `exp` earlier than the `exp` the issuer would set for the issued token, the issuer:

1.  MAY lower the issued token's `exp` to the earliest `exp` among the retained proofs;
2.  otherwise, where local policy permits absent coverage, MUST drop the `actor_proofs` array;
3.  otherwise, MUST fail the request with the `invalid_grant` error code on a refresh or JWT bearer grant request ({{Section 5.2 of RFC6749}}), or the `invalid_request` error code on a Token Exchange request ({{Section 2.2.2 of RFC8693}}).

## Accepting a Proof for a New Actor Hop {#accepting-a-proof}

When an issuer adds a new outermost actor hop and the token request carries the `actor_proof` parameter, the issuer:

1.  MUST validate the proof's structure per {{actor-proof-jwt-format}}: `typ` value, asymmetric `alg`, presence and JSON types of the REQUIRED claims `iss`, `sub`, `act`, `target` (including `target.aud`), `iat`, `exp`, and `jti`, the absence of `aud`, and the single-hop `act` rules.
2.  MUST verify that the proof's (`act.iss`, `act.sub`) pair equals the actor identifier pair the issuer will emit as the new outermost visible `act` object, and that the proof's `iss` equals the proof's `act.sub`.  Deployment configuration supplies the actor with the `act.iss` value the issuer will emit.  When the proof carries `act.sub_profile`, the issuer MUST verify that it matches, under the set comparison of step 7 of {{consumer-processing}}, the `act.sub_profile` the issuer emits for the new outermost actor.
3.  MUST verify that the proof's `sub` equals the top-level `sub` of the token being issued.  An issuer that re-expresses the subject at this hop MUST NOT embed the proof; re-expression breaks the alignment between `actor_proofs[0].sub` and the outer token's top-level `sub` that consumers verify under {{consumer-processing}}.
4.  MUST resolve the actor's verification key through an actor-key source trusted under the issuer's local policy and validate the proof's signature ({{actor-key-resolution}}).
5.  MUST verify that the proof's `exp` is no earlier than the issued outer token's `exp`, and that `iat` is plausible under the issuer's clock-skew policy.
6.  MUST NOT issue an outer token whose `aud` or effective resource indicators exceed the proof's target binding.  Every audience of the issued token MUST be present in `target.aud`, and every effective resource indicator MUST equal an entry of `target.resource` under simple string comparison ({{Section 6.2.1 of RFC3986}}) when that member is present.  When `target.resource` is present and the request supplies no resource indicators, the issuer uses `target.resource` as the effective resource indicators and MUST NOT issue a token whose effective resources exceed it.  On a request other than Token Exchange, the issuer MAY narrow the issued resource indicators to fit the target binding, as {{Section 2.2 of RFC8707}} leaves acceptable resources to its policy.  On a Token Exchange request, it does not drop a requested audience or resource, because both name targets where the requested token must be usable ({{Section 2.1 of RFC8693}}); {{error-handling}} gives the error to return when the requested target cannot be issued within the target binding after any narrowing.
7.  MUST verify, when the proof carries `origin_jti`, that it equals the issued token's `jti`, and, when the proof carries `receipt_jti` and the issuer creates a sibling receipt for this hop ({{sibling-receipt-issuance}}), that it equals that receipt's `jti`.
8.  MUST include the validated proof as `actor_proofs[0]` of the issued token, subject to the chain rules below.

When no inbound `actor_proofs` are being preserved, the proof starts a new chain and MUST omit `prh`.  The resulting one-element array constitutes complete coverage only when the visible `act` chain has depth 1; {{actor-proofs-claim}} governs how the issuer sets `actor_proofs_complete` in each case.

If proof validation fails, the issuer MUST NOT embed the proof.  When local policy or the deployment's resource requirements require actor-signed evidence for the issuance, the issuer MUST fail the request under the error model of {{error-handling}}; otherwise it MAY issue the token without `actor_proofs`.

## Extending an Existing Proof Chain {#extending-an-existing-proof-chain}

When an issuer adds a new outermost actor hop and also preserves an inbound `actor_proofs` array, it:

1.  MUST validate the inbound proof chain by applying the consumer processing rules in {{consumer-processing}} before relying on it or carrying it forward.
2.  MUST verify that each inbound proof's `exp` is no earlier than the issued outer token's `exp`, applying the lifetime rule in {{issuer-processing}} when an inbound proof's `exp` is earlier than the `exp` the issuer would set.  Issuers MAY apply a small clock-skew margin to this comparison, consistent with the consumer-side skew tolerance in {{consumer-processing}}, but MUST NOT broadly accept inbound proofs whose `exp` precedes the issued outer token's `exp` by more than a deployment-defined skew bound.
3.  preserves each inbound proof byte-for-byte, as required by {{proof-chain-linkage}}.
4.  MUST accept exactly one new proof, conveyed per {{actor-proof-parameter}} and validated per {{accepting-a-proof}}, for the new outermost actor hop.  Without a valid new proof, the issuer MUST NOT carry the inbound `actor_proofs` array forward; it continues without proofs where local policy permits absent coverage, and otherwise MUST fail the request under {{error-handling}}.
5.  MUST verify that the new proof's `prh` equals the hash of the exact compact serialization of the inbound array's newest proof, computed using the algorithm named by the inherited `prh_alg` (defaulting to SHA-256 when absent), and MUST verify that the new proof's `prh_alg` matches the inherited chain's value or is omitted when the chain omits it.  An issuer that does not support the inbound `prh_alg` MUST reject the chain rather than rehash; rehashing would invalidate prior actors' signatures.
6.  MUST prepend the new proof to the inherited array.
7.  MUST NOT set `actor_proofs_complete` to `true` unless every inbound proof validated and the issued token's proof count equals its visible actor-chain depth, and SHOULD set it to `false` otherwise.

Byte-for-byte preservation ({{proof-chain-linkage}}) precludes reserializing, re-signing, normalizing, trimming, or otherwise altering a prior proof.

The actor needs to know the newest inbound proof's exact serialization, or its hash and `prh_alg`, before signing.  For an inbound JWT, the actor can read `actor_proofs[0]`.  For an opaque inbound token, the deployment needs to supply that information through a mechanism it defines ({{actor-proof-parameter}}).  If that information is unavailable, the issuer MUST NOT accept a proof without `prh` as a chain extension; it MAY instead start a new chain under {{accepting-a-proof}} where local policy permits partial coverage ({{partial-coverage-and-full-coverage}}).

If inbound proofs fail validation, the issuer MUST NOT propagate them.  It MAY continue without them only when local policy permits partial or absent coverage, and MAY then begin a new chain at its own hop under {{accepting-a-proof}}; the result is partial coverage and MUST NOT carry `actor_proofs_complete: true`.  Otherwise it MUST fail the request under the error model of the underlying protocol.

## Reissuance Without a New Actor Hop {#reissuance-without-a-new-actor-hop}

An issuer that reissues, translates, or introspects and re-emits a token without adding a new outermost actor hop:

*  MAY carry an `actor_proofs` array received in an inbound token or its introspection response forward unchanged, and MUST first validate it against that token under {{consumer-processing}}, as step 1 of {{extending-an-existing-proof-chain}} requires for extension; an array the issuer retained across refresh follows the refresh rules below instead.  If the array fails validation, the issuer MUST NOT carry it forward, and MUST fail the request under the error model of the underlying protocol unless local policy permits the issued token to lack it;
*  MUST NOT accept or embed a new proof, and MUST reject, with the `invalid_request` error code ({{Section 5.2 of RFC6749}}), a request that carries an `actor_proof` parameter;
*  MUST preserve `actor_proofs_complete` when carrying the array unchanged.  If it cannot attest that value, the issuer MUST drop the whole array;
*  MUST NOT continue to carry an inherited `actor_proofs` array if it cannot preserve the visible hop alignment required by {{consumer-processing}};
*  MUST NOT change the top-level `sub` claim while retaining proofs; doing so breaks alignment with `actor_proofs[0].sub`;
*  MUST NOT set the outer token's `exp` later than the earliest `exp` among the retained proofs, and applies the lifetime rule in {{issuer-processing}} when it would otherwise set a later `exp`.

If such an issuer changes the visible outermost actor, it has added a new hop and MUST follow {{extending-an-existing-proof-chain}}.

If reissuance exceeds the newest proof's target binding, the issuer MUST drop `actor_proofs` unless deployment agreement designates it, for the recipients of the reissued token, as a trusted reissuing issuer permitted to retarget ({{target-binding-strict-mode}}).  A reissued token with a new `jti` diverges from a present `actor_proofs[0].origin_jti` ({{target-binding-strict-mode}}).

An actor that intends its proof to remain usable after a later redemption without a new hop, for example an Identity Assertion JWT Authorization Grant that a Resource Authorization Server redeems ({{I-D.mcguinness-oauth-actor-profile}}), can include the known downstream audiences in `target.aud`, subject to its consent policy.  This addresses audience divergence only: a new `jti` still diverges from a present `origin_jti` ({{target-binding-strict-mode}}), and a changed subject or the proof's expiry still ends the proof's use for the redeemed token.

If proofs are dropped while receipts remain, inherited `proof_jti` references become informational.  Recipients that require bound siblings enforce proof presence through the `actor_proofs_required` or `actor_proofs_complete_required` metadata parameter, or through local policy.

An AS that supports refresh tokens for delegated access tokens carrying proofs:

*  needs to retain the `actor_proofs` array in issuer-controlled state across refresh, either in durable storage (for example, a token-state database or refresh-token state) or embedded in a self-contained refresh token, so that each refreshed access token can carry the proofs forward unchanged.
*  takes the array from that retained state rather than from the previous access token: it validates the refresh request per {{Section 6 of RFC6749}}, checks the retained proofs against its issuance state, and neither requires the previous access token to remain unexpired nor re-runs {{consumer-processing}} against it.
*  applies the lifetime rule in {{issuer-processing}} to each refreshed access token.  Refresh ends instance binding, so proof `exp` sizing for refreshed tokens follows the short-`exp` guidance in {{proof-claims}}.
*  after dropping `actor_proofs` under that rule, restores actor-signed evidence only through a new delegated issuance that adds a hop with a fresh proof, because refresh adds no actor hop.

## Partial Coverage and Full Coverage {#partial-coverage-and-full-coverage}

This document permits partial proof coverage to support progressive deployment.  An issuer MAY begin a new proof chain even when older inner actor hops remain visible but uncovered.  When local policy or resource requirements require full actor-signed evidence, the issuer MUST either emit complete proof coverage or fail the request under the error model of the underlying protocol.

Partial coverage leaves the oldest hops uncovered, including the original subject-to-actor delegation.  Deployments needing evidence for that hop should enable proof support at the origin and its actors first.

When an introspection server filters the visible `act` chain (see the `chain_complete` introspection member defined in the core actor profile {{I-D.mcguinness-oauth-actor-profile}}), the `actor_proofs` member covers only the visible filtered chain.  In that case, the `actor_proofs_complete` member describes coverage relative to the visible filtered chain, not the unfiltered delegation chain; recipients that need true-chain completeness MUST evaluate the `chain_complete` member separately.

## Transaction Token Service Rebinding

A Transaction Token Service that establishes a new presenter and makes that presenter the new outermost actor follows the same proof rules as any other issuer that adds a new outermost actor hop, as defined in {{extending-an-existing-proof-chain}} (or {{accepting-a-proof}} when no inbound `actor_proofs` array exists).  The new presenter is the signing actor for the new proof.

## Sibling Receipt Issuance {#sibling-receipt-issuance}

When a deployment uses both this profile and {{I-D.mcguinness-oauth-actor-receipts}}, the issuer that adds a hop creates the receipt and embeds the proof for that hop in the same issuance operation.  The two artifacts are siblings: independent attestations of the same hop by different signers.

This document defines the following extension claim for Actor Receipt JWTs, under the extension-claims rule of {{I-D.mcguinness-oauth-actor-receipts}}:

`proof_jti`:
: OPTIONAL.  The `jti` of the proof the receipt issuer validated for the same hop.  When the issuer embeds a proof and creates a sibling receipt for one hop, the receipt SHOULD include `proof_jti` equal to that proof's `jti`.

Byte-for-byte preservation keeps the `proof_jti` value fixed, making later proof substitution detectable through sibling validation (step 10 of {{consumer-processing}}).

# Consumer Processing {#consumer-processing}

An issuer, resource server, or other recipient that relies on `actor_proofs` MUST perform the following steps.

1.  Validate the outer token according to its token type and the core actor profile.
2.  If `actor_proofs` is absent, treat the token as lacking actor-signed evidence.  Local policy or Protected Resource Metadata parameters such as `actor_proofs_required` and `actor_proofs_complete_required` defined in {{discovery-capability-signaling}} determine whether that is acceptable.  If `actor_proofs_complete` is present with the value `true` while `actor_proofs` is absent, the combination is malformed; the recipient MUST treat this as a failed required check and apply the rejection rule following step 11.
3.  Verify that `actor_proofs`, if present, is a non-empty JSON array of strings.  Verify that `actor_proofs_complete`, if present, is a JSON boolean.
4.  Verify that the number of proofs does not exceed the visible actor-chain depth of the outer token.  If the outer token carries `actor_proofs_complete: true`, verify that the proof count exactly equals the visible actor-chain depth; if it does not, the check fails.
5.  For each proof, in array order:
    *  parse the string as a compact JWT;
    *  verify that the JOSE header uses an asymmetric digital-signature `alg` value accepted for that actor, and reject proofs that use `alg: none` or a MAC-based symmetric algorithm ({{Section 3.1 of RFC8725}});
    *  verify that the `typ` header parameter equals `actor-proof+jwt`;
    *  verify that the proof's (`act.iss`, `act.sub`) pair is within the scope of an actor-key source the recipient trusts, before performing any network retrieval keyed by the proof's content;
    *  resolve the actor's verification key from that source ({{actor-key-resolution}});
    *  validate the proof signature;
    *  reject a proof whose `crit` header parameter lists an extension header parameter that the recipient does not understand;
    *  verify that all REQUIRED proof claims are present and have the expected JSON types, including `iss`, `sub`, `act`, `target` with `target.aud`, `iat`, `exp`, and `jti`;
    *  verify that OPTIONAL claims used by this profile have the expected JSON types when present, including `sub_iss`, `target.resource`, `prh`, `prh_alg`, `receipt_jti`, and `origin_jti`;
    *  reject a proof that carries an `aud` claim ({{proof-claims}});
    *  verify that the proof `act` object is single-hop, contains no nested `act`, and contains no `cnf`, and that the proof `iss` equals the proof `act.sub`;
    *  enforce the `exp` and `iat` claims and other JWT validity rules.  An expired proof is invalid even for an older hop; only the small clock-skew leeway of {{Section 4.1.4 of RFC7519}} applies.
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
    *  when `act.sub_profile` is present in the proof `act` object, the corresponding visible `act` object MUST contain `act.sub_profile` with the same value.  `sub_profile` values are compared as sets: the space-delimited values are compared case-insensitively, their order is insignificant, and duplicate values are ignored ({{Section 3.3 of I-D.mora-oauth-entity-profiles}}); comparison never rewrites a signed proof;
    *  when `act.sub_profile` is present only in the visible `act` object, the proof remains aligned for this profile.  The actor does not attest the visible value, and recipients that require actor-signed evidence for actor classification MUST reject the proof chain or apply explicit local mapping rules.
8.  Verify subject alignment:
    *  `actor_proofs[0].sub` MUST equal the outer token's top-level `sub`;
    *  when `actor_proofs[0].sub_iss` is present and the recipient has a top-level subject namespace authority for the outer token's `sub` from local configuration, an inbound subject token's claims, or another deployment-defined source, the two MUST identify the same namespace authority, evaluated by case-sensitive string comparison; treating lexically distinct identifiers as the same authority requires explicit trusted local mapping rules;
    *  older proofs MAY carry differing `sub` or `sub_iss` values.  This acceptance is structural only: authorization that depends on subject equivalence across those proofs is subject to the continuity rules of {{subject-re-expression-across-hops}}.
9.  Evaluate outer-token binding and target binding:
    *  when `actor_proofs[0].origin_jti` is present and equals the outer token's `jti`, the proof chain is bound to the current outer-token instance; when it is present and differs, the chain has diverged: {{target-binding-strict-mode}} decides whether the recipient rejects it (rejection is the default), and an accepted value is historical provenance;
    *  when `actor_proofs[0].origin_jti` is absent, the proof chain carries no instance binding of its own; this is not by itself a validation failure;
    *  verify that every audience of the outer token is present in `actor_proofs[0].target.aud`, and, when the outer token's effective resource indicators are determinable from token claims, the introspection response, or trusted local context, that each equals an entry of `actor_proofs[0].target.resource` under simple string comparison ({{Section 6.2.1 of RFC3986}}) when that member is present.  A recipient that cannot determine the outer token's effective resources treats the proof as consent to the audience only, not as divergence, and MUST NOT infer resource-level consent.  A token whose audience or resources exceed the newest proof's target binding has also diverged; {{target-binding-strict-mode}} decides whether the recipient rejects the chain (rejection is the default) and limits an accepted chain to participation evidence, not actor consent to the current target;
    *  target bindings of proofs other than `actor_proofs[0]` are historical consent for their own hops.  The recipient MUST NOT evaluate them against the current outer token's audience or resources.
10.  Verify sibling references, when the token also carries `actor_receipts` validated under {{I-D.mcguinness-oauth-actor-receipts}}:
     *  for each index i covered by both arrays, when `actor_receipts[i]` carries `proof_jti`, it MUST equal `actor_proofs[i].jti`, and when `actor_proofs[i]` carries `receipt_jti`, it MUST equal `actor_receipts[i].jti`;
     *  a mismatched sibling reference MUST cause the recipient to reject both receipt-based and proof-based provenance for the token;
     *  a sibling reference that names an artifact at an index not covered by the other array is unverifiable; recipients whose policy requires bound siblings MUST reject the token's proof-based provenance, and other recipients MUST treat the reference as informational only;
     *  when receipts are absent or not validated, `receipt_jti` values are informational only.
11.  Apply any additional consumer-processing rules defined by companion profiles whose claims appear in the proof or outer token (see {{extensibility}}).

Step 1 is a prerequisite: an outer token that fails its own validation is rejected under the rules for its token type, not treated as lacking proofs.  If any later required check fails, the recipient MUST reject the proof chain and treat the token as lacking actor-signed evidence (step 2).  The recipient rejects the token only when local policy or Protected Resource Metadata requires that evidence, using the underlying protocol's error handling for the stage at which the failure occurred.

A recipient that has rejected a proof chain under this profile MAY, under explicit local policy, extract structural information from the chain for use by companion profiles.  The recipient MUST NOT treat such partial validation as conformance with this profile, and MUST NOT relax the rejection requirements defined above.

## Subject Re-Expression Across Hops {#subject-re-expression-across-hops}

Older proofs can carry a `sub` value that differs from that of the current outer token when the subject has been re-expressed across issuer namespaces between hops.  This document does not define a universal subject-mapping algorithm; step 8 of {{consumer-processing}} requires only `actor_proofs[0].sub` to equal the current outer token's `sub`.

Permitting differing `sub` values across proofs creates a cross-subject insertion risk: a proof signed by a legitimate actor for an unrelated subject's delegation could satisfy the structural hop-alignment check when the actor identity at that hop matches.  An attacker who compromises any single actor signing key can sign proofs naming any subject and any target, and graft them onto a downstream chain whose re-expressed `sub` identifies a victim subject.

Deployments where subject continuity is a security requirement SHOULD adopt one of the following:

*  require exact, namespace-aware matching of subject identifiers across all proofs (the same `sub` under the same namespace authority; see `sub_iss` in {{identity-claims}}); or
*  enforce explicit trusted subject-mapping rules that can positively confirm that each distinct subject identifier refers to the same underlying entity.

When neither condition is met, the recipient MUST treat subject continuity as unverified and MUST NOT rely on older proofs whose subject identifiers (`sub` or `sub_iss`) differ to support authorization that requires subject continuity (for example, a decision that treats every covered hop as having acted for the current token's subject).

## Complete Proof Coverage

Coverage is structurally complete when all validation succeeds and the proof count equals the visible actor-chain depth.  This suffices when local policy requires only structural completeness.

When Protected Resource Metadata sets `actor_proofs_complete_required: true`, the token or introspection response MUST also carry `actor_proofs_complete: true`.  Recipients MUST reject tokens that fail the applicable completeness requirement.

## Use by Resource Servers

Resource servers can use validated proofs as evidence for authorization, diagnostics, and audit, subject to the limits in {{threat-model}}.  However, a valid proof chain:

*  proves only that the covered actors signed their participation and hop-time target bindings;
*  does not prove that the represented delegation remains active, authorized, or acceptable under current policy;
*  does not prove that any authorization server validated the hop; that attestation is the receipts companion's role;
*  does not remove the need to authorize the current token itself;
*  does not convey authority, authorization, entitlement, or delegation rights;
*  is not actor consent to the current request; it is actor consent to the hop-time issuance within the recorded target binding.

Current authorization decisions MUST evaluate the current outer token, current policy, and current state, not the proof chain alone.

## Introspection {#consumer-introspection}

When an authorization server returns actor-proof information in an OAuth Token Introspection response {{RFC7662}}, it:

*  MAY return the `actor_proofs` member using the same array format defined in {{actor-proofs-claim}};
*  MAY return the `actor_proofs_complete` member to indicate whether the returned array provides complete coverage for the visible chain as known to the introspection server.

For opaque tokens, the issuer stores proofs and returns them to authorized resource servers through introspection.  The same format and consumer rules apply, with the introspection response serving as the outer token's claim set.

An introspection response carrying proofs MUST include the members needed for {{consumer-processing}}: the token's top-level `sub`, the visible `act` chain, the token's `aud`, and the token's `iss`.  It SHOULD include the `jti` member when maintained by the server; without it, the response supplies no instance binding under step 9.  If present, the `actor_proofs_complete` member MUST be a boolean.

An RS receiving both inline and introspected proofs MUST select an authoritative source under local policy.  If it consumes both, differing arrays or completeness values MUST cause rejection of proof-based provenance.

An introspection server MUST return the full stored array or omit `actor_proofs`.  Removing an older entry breaks the `prh` chain; removing the newest entry breaks hop alignment.  A server that filters the visible `act` chain can still return the full array when it filters only inner actors that no proof covers; if it filters a covered actor, it MUST omit both `actor_proofs` and `actor_proofs_complete`.  When the introspection server returns a stored array that it knows has partial coverage, it MUST include `actor_proofs_complete: false`.

For an inactive token, the introspection server MUST NOT return `actor_proofs` or `actor_proofs_complete`.

# Discovery and Capability Signaling {#discovery-capability-signaling}

This section follows the claim-pair and discovery conventions defined in {{I-D.mcguinness-oauth-actor-receipts}}.

## Authorization Server Metadata

The following parameter is defined for use in Authorization Server Metadata {{RFC8414}}:

`actor_proofs_supported`:
: OPTIONAL.  A boolean.  When `true`, the authorization server advertises that it accepts the `actor_proof` token request parameter, validates proofs against actor keys, and embeds, preserves, or extends proof chains according to this document.  This value does not guarantee complete coverage for every visible hop in every resulting token.  When `false` or absent, the AS makes no claim of such support.

This parameter applies equally to an authorization server that issues delegated JWT outputs and to a Transaction Token Service publishing metadata through the same framework.

## Protected Resource Metadata

The following parameters are defined for use in Protected Resource Metadata {{RFC9728}}:

`actor_proofs_required`:
: OPTIONAL.  A boolean.  When `true`, the resource server indicates that delegated requests are expected to carry valid actor proofs covering at minimum the outermost visible actor hop.  When `false` or absent, the resource server makes no metadata declaration about actor-signed evidence requirements.

  Unlike receipt issuance, proof creation involves the actor directly: an actor that can sign proofs MAY use this declaration, together with `actor_proofs_supported` in Authorization Server Metadata, to decide whether to include the `actor_proof` parameter in its token requests.  The declaration also serves deployment coordination and expresses the enforcement posture under which this profile's anti-fabrication property holds ({{downgrade-by-omission}}).

`actor_proofs_complete_required`:
: OPTIONAL.  A boolean.  When `true`, the resource server indicates that it requires complete proof coverage: the proof count needs to equal the visible actor-chain depth and `actor_proofs_complete` needs to be `true` in the outer token or the introspection response.  This parameter refines `actor_proofs_required`; a resource server SHOULD NOT set `actor_proofs_complete_required: true` without also setting `actor_proofs_required: true`.  When `false` or absent, partial proof coverage is acceptable to the resource server, subject to any further local policy.

## Introspection Response Members {#introspection-response-members}

The following members are defined for use in OAuth Token Introspection responses {{RFC7662}}:

`actor_proofs`:
: OPTIONAL.  An array of strings using the same syntax as the JWT claim of the same name.

`actor_proofs_complete`:
: OPTIONAL.  A boolean.  When `true`, the introspection response indicates that the returned `actor_proofs` cover every visible hop in the token chain as known to the introspection server.  When `false`, the response makes no attestation of complete coverage.

Consumer use of these members is described in {{consumer-introspection}}; introspection-server failure handling is addressed in {{introspection-errors}}.

# Error Handling {#error-handling}

Proof validation failures use the underlying protocol's error mechanism for the stage at which validation occurs.

## Authorization Server and Transaction Token Service Errors

When an authorization server or Transaction Token Service rejects a token request because an inbound `actor_proofs` chain or a newly submitted proof cannot be validated (signature failure, key-resolution failure for an actor outside the trusted key sources, expired proof, unsupported `prh_alg`, broken `prh` chain, hop or subject misalignment), it returns an error response per {{Section 5.2 of RFC6749}}: `invalid_request` for a Token Exchange request, as {{Section 2.2.2 of RFC8693}} requires, or `invalid_grant` for a JWT bearer grant request ({{Section 3.1 of RFC7523}}), consistent with the core actor profile's error mapping for actor information that fails validation.

When the requested target cannot be issued within the submitted proof's target binding after any narrowing under {{accepting-a-proof}}, the issuer SHOULD return `invalid_target`, per {{Section 2.2.2 of RFC8693}} for a Token Exchange request and {{Section 2 of RFC8707}} for other token requests.

When the failure reflects an actor-authorization decision rather than a structural validation failure, the issuer uses the `actor_unauthorized` error code, as the core actor profile {{I-D.mcguinness-oauth-actor-profile}} requires.  An absent required proof, whether an `actor_proof` parameter or an inbound `actor_proofs` array, is an input-validation failure: the issuer returns the `invalid_request` error code for a Token Exchange request and the `invalid_grant` error code for a JWT bearer grant or refresh request.

## Resource Server Errors

When a resource server rejects a request because `actor_proofs` validation fails under {{consumer-processing}}, it SHOULD return `invalid_token` per the bearer-token error model in {{Section 3.1 of RFC6750}}.  For a Transaction Token, the recipient instead rejects the token through the deployment's Transaction Token handling, because {{I-D.ietf-oauth-transaction-tokens}} defines no error response for a rejected Transaction Token.

When the failure is specifically that required proofs are absent or coverage is incomplete (per `actor_proofs_required` or `actor_proofs_complete_required`), the resource server SHOULD include an `error_description` value identifying proof-coverage failure so that clients and operators can distinguish it from generic token-validation failures.

## Introspection Server Behavior {#introspection-errors}

When an introspection server cannot return proofs that the requesting resource server requires, it returns the introspection response per {{RFC7662}} with `actor_proofs` absent, or with a partial array and `actor_proofs_complete: false`; the resource server then applies its local policy to decide whether to accept the token.

# Extensibility {#extensibility}

This profile composes with the extensibility framework defined in {{I-D.mcguinness-oauth-actor-receipts}} and adds proof-specific extension surfaces:

*  **New claims inside a proof JWT** for additional per-hop actor-attested attributes.  Consumers ignore unrecognized claims under {{proof-claims}} unless another specification or local agreement defines their meaning.
*  **New `target` extension members** with constraining semantics, per the extension rule in {{proof-claims}}.  Specifications needing actor consent at scope or action granularity extend `target` rather than redefining it.
*  **Challenge and provisioning mechanisms** that supply prospective values to the actor before signing ({{actor-proof-parameter}}); such mechanisms strengthen instance binding without changing proof processing.
*  **Actor events between hops**, such as consent to a changed target.  Companion profiles SHOULD use a JWT type distinct from `actor-proof+jwt`, a parallel outer-token array, and an anchor to a proof `jti` or defined flow identifier, following the non-hop event pattern of {{I-D.mcguinness-oauth-actor-receipts}}.  Companion profiles MUST NOT add event entries to `actor_proofs`, which is reserved for the actor-signed hop proofs defined by this document.
*  **Multi-actor co-signed hops** are outside the scope of this document and would require a successor or companion profile with its own artifact structure.

Companion profile authoring rules:

*  Companion profiles MAY extend consumer processing under {{consumer-processing}} by adding rejection conditions; they MUST NOT relax any requirement needed for conformance to this profile.  This does not change the separately scoped partial-validation rule that follows step 11 of {{consumer-processing}}.
*  Companion profiles that define per-hop signed artifacts SHOULD follow the claim-pair and discovery conventions of {{I-D.mcguinness-oauth-actor-receipts}}, and MAY reuse the `prh` and `prh_alg` chain-linkage construction.

Conflict resolution: when a recipient implements multiple companion profiles whose rules conflict, local policy determines precedence.

# Security Considerations

The general OAuth 2.0 Security Best Current Practice {{RFC9700}} and the JWT best practices in {{RFC8725}}, except its audience requirements for proof JWTs (see `aud` in {{proof-claims}}), apply to systems implementing this profile.

## Threat Model {#threat-model}

### Adversaries Mitigated by This Profile

*  **Current outer-token issuer fabricating actor participation.**  The issuer cannot forge the actor's signature at proof-covered hops, provided the recipient requires proofs ({{downgrade-by-omission}}) and resolves actor keys independently of that issuer ({{actor-key-resolution}}).
*  **Current issuer exceeding the actor-authorized target at the covered hop.**  Issuer-side enforcement ({{accepting-a-proof}}) and step 9 of {{consumer-processing}} detect a token exceeding the newest proof's target binding, subject to {{target-binding-strict-mode}}.
*  **Compromised downstream issuer fabricating prior-hop participation.**  Such an issuer cannot forge prior actors' proof signatures, and the `prh` chain prevents it from dropping or reordering inner proofs.
*  **Token mutation in transit.**  Each proof is independently signed; modification invalidates the proof's signature and any newer proof's `prh`.
*  **Partial-coverage misclaim.**  An issuer cannot drop an inner proof without breaking the `prh` chain, and it cannot claim `actor_proofs_complete: true` unless the proof count matches the visible actor-chain depth.  It can withhold coverage only at the innermost end, and only by beginning a new chain rather than trimming an inherited one.
*  **Proof-chain substitution, when receipts with `proof_jti` are present.**  A harvested proof chain for the same visible hops fails the sibling check in step 10 of {{consumer-processing}}.

### Adversaries Not Mitigated

*  **Compromised actor signing key.**  Forged proofs are indistinguishable from legitimate ones and cannot be revoked individually ({{actor-key-compromise}}).
*  **Issuer omission of proofs.**  Omission is a downgrade, not merely denial of service, against recipients that do not require proofs ({{downgrade-by-omission}}).
*  **Proof re-embedding within the validity window.**  Any holder of a valid proof, including the issuer it was submitted to, can embed it in another token with matching context ({{proof-to-token-binding-limits}}).
*  **Malicious or coerced actor.**  Proofs attest that the actor's key signed the participation, not the actor's intent; neither companion detects an actor colluding with a compromised issuer.
*  **Cross-subject graft with a compromised actor key.**  Mitigation: exact, namespace-aware subject matching across proofs or trusted subject mapping ({{subject-re-expression-across-hops}}).
*  **Replay of an entire token plus its proofs.**  This profile does not define replay detection; proofs inherit the outer token's replay characteristics.

## Current Presenter Validation

When the outer token carries a top-level `cnf` claim ({{RFC7800}}), the current request is always validated against it, using a mechanism such as DPoP {{RFC9449}} or mutual-TLS {{RFC8705}}.

An actor proof does not substitute for that validation:

*  A recipient MUST NOT treat a proof signature as satisfying a proof-of-possession requirement for the current request, regardless of whether the proof signing key is the same key as a presenter key.
*  Recipients MUST distinguish proof JWTs (identified by the `typ` value `actor-proof+jwt`) from artifacts that carry current-request proof-of-possession semantics under {{RFC7800}}, {{RFC9449}}, or {{RFC8705}}.

## Actor Key Resolution and Trust {#actor-key-resolution}

Trust is per-actor-key and per-deployment, and is not transitive: a proof chain fails validation if any proof's signing key cannot be resolved through an actor-key source the recipient trusts (step 5 of {{consumer-processing}}).  Proofs and receipts have independent trust anchors; validating both survives compromise of either the issuer side or the actor side, but not of both.

Trust establishment requirements:

*  A recipient needs to establish its trusted actor-key sources before relying on `actor_proofs`, through explicit pre-configuration, bilateral agreement, federation policy, or another explicit trust framework.
*  A recipient MUST NOT treat the presence of a syntactically valid signed proof as sufficient grounds to trust the key that signed it.
*  A recipient MUST NOT dereference key references supplied by the proof itself (such as `jku` or `x5u` header parameters) outside a pre-established trust framework, per {{RFC8725}}.
*  Key resolution and trust evaluation use the (`act.iss`, `act.sub`) pair, checked in step 5 of {{consumer-processing}} before any retrieval; identical `act.sub` strings under different namespace authorities are different actors ({{identity-claims}}).

This document profiles two resolution patterns; a deployment can support either or both:

*  **Pre-established keys**, registered in advance, for example as a registered OAuth client's JWKS at the authorization server or as keys configured at a resource server.  This pattern provides the strongest independence properties; attestation-based client authentication {{I-D.ietf-oauth-attestation-based-client-auth}} is an interoperable way to establish such keys.
*  **Federation and workload identity systems**, with the trust and freshness properties of the underlying system, such as OAuth SPIFFE client authentication {{I-D.ietf-oauth-spiffe-client-auth}}.

The independence requirement follows from the threat model: for the anti-fabrication property against a given issuer to hold at a hop, the recipient MUST resolve the actor's key for that hop through a source independent of that issuer.

## Proof-to-Token Binding Limits {#proof-to-token-binding-limits}

Without instance binding, any holder of a proof, including the issuer it was submitted to, can reuse it in another token with the same subject, actor, and target within its validity window; the signature evidences consent to that context, not to a particular token issuance.

Available bindings:

*  **Receipts composition.**  When the token also carries receipts, the receipt chain's `origin_jti` anchoring and strict-mode rules in {{I-D.mcguinness-oauth-actor-receipts}} bind the token instance, and `proof_jti` ({{sibling-receipt-issuance}}) binds the proof chain to it; re-embedding then requires a fabricated receipt, which only an issuer able to sign trusted receipts can produce.  This is the RECOMMENDED posture for deployments that already use receipts and require instance binding.
*  **Provisioned `origin_jti`.**  Binds the proof to the outer-token instance directly when the actor receives the prospective `jti` before signing (step 9 of {{consumer-processing}}).
*  **`jti` uniqueness monitoring.**  Recipients and audit pipelines MAY track proof `jti` values and flag the same proof appearing in more than one outer-token instance.
*  **Short `exp`.**  Bounds the re-embedding window unconditionally, at the cost of shorter delegated sessions ({{proof-claims}}).

Receipts composition relies on the receipt issuer's assertion and a provisioned `origin_jti` on the actor's own signature; neither binding signs the outer token's other contents.  Inner proofs inherit whatever binding `actor_proofs[0]` has through `prh`.

### Target-Binding Strict Mode {#target-binding-strict-mode}

An outer token diverges from its proof chain when its audience or effective resources exceed `actor_proofs[0]`'s target binding, or when its `jti` differs from a present `actor_proofs[0].origin_jti`.  A recipient MUST reject a divergent proof chain unless local policy designates the outer token issuer as a trusted reissuing issuer.  Designation as a trusted reissuing issuer excuses only `jti` divergence unless local policy also permits that issuer to retarget.  Recipients that have not explicitly configured a set of trusted reissuing issuers therefore operate in strict mode by default, rejecting every divergent chain.

Strict mode is the recommended default.  A recipient accepting target divergence MUST treat proofs only as participation evidence and MUST NOT infer consent to the current audience or resources.  With receipts, it SHOULD apply one reissuance-trust decision to both companions.

## Hash Algorithm Agility

The `prh` and `prh_alg` agility rules of {{I-D.mcguinness-oauth-actor-receipts}} apply to proof chains unchanged: one algorithm per chain, whole-chain migration only, no rehashing of inherited artifacts, and rejection of mixed or unsupported algorithms.

## Downgrade by Omission {#downgrade-by-omission}

An issuer fabricating actor participation omits `actor_proofs` rather than forging a proof, so a recipient that accepts delegated tokens without proofs has no protection from this profile against it.

Accordingly:

*  Resource servers that rely on actor-signed evidence MUST require proofs, through `actor_proofs_required` (and `actor_proofs_complete_required` where inner hops matter) or equivalent local policy, for the delegated tokens they accept.
*  Recipients SHOULD treat the absence of proofs from an issuer that advertises `actor_proofs_supported: true`, for a resource that requires them, as a signal warranting scrutiny rather than silent acceptance.

## Proof Freshness and Replay {#proof-freshness}

Proofs are historical attestations of hop-time consent, carried forward only in tokens that expire no later than they do ({{reissuance-without-a-new-actor-hop}}); they do not show that the delegation is still active or that the actor would consent today.  Deployments that need freshness signals beyond proof `exp` obtain them via introspection ({{RFC7662}}), fresh token issuance, or another mechanism outside the scope of this document.

## Actor Key Compromise

Remediation for a compromised actor signing key is to remove the key or actor from the recipient's trusted actor-key sources; a short proof `exp` ({{proof-claims}}) limits how long proofs signed with it remain valid.  When a key compromise is detected, deployments SHOULD treat tokens carrying proofs from the affected actor as lacking trusted actor-signed evidence for those hops and SHOULD require fresh delegation with fresh proofs.

## Proof Chain Size

Each proof is a signed JWT of typically 400 to 800 bytes, comparable to a receipt, and the chain grows linearly with delegation depth; carrying both companions roughly doubles the per-hop bytes.  Deployments SHOULD verify that the outer token plus its companion arrays fits within the header-size budget of every component on the request path; the receipts companion's size guidance applies, and introspection delivery avoids header pressure for bearer-token clients.

## Sibling Revocation Independence

A proof does not inherit revocation or trust state from its sibling receipt, or the reverse: removing a receipt issuer from the trusted-issuer set under {{I-D.mcguinness-oauth-actor-receipts}} leaves proofs for the same hops valid under their own actor keys.  Recipients that require issuer-side revocation semantics for a hop MUST require receipt validation alongside proof validation rather than relying on the proof alone; the sibling references of {{sibling-receipt-issuance}} identify the receipt whose trust state applies to a hop.

# Privacy Considerations

## What Proofs Disclose

Proofs can expose, to any party that receives the token or introspection response:

*  cryptographically transferable evidence of each covered actor's participation, retained and provable beyond token lifetime;
*  actor key identifiers (`kid` values and resolved public keys), which are stable correlation handles across proofs, flows, and services;
*  target bindings (`target.aud`, `target.resource`), which can reveal internal audience and resource identifiers a deployment would not otherwise expose to all recipients;
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

This profile does not define a per-claim selective-disclosure mechanism for proofs: chain integrity requires byte-for-byte preservation of each proof JWT, so selective omission of individual claims within a proof would break the chain.  Selective disclosure is therefore coarse-grained: issuance-time partial coverage of outermost hops, or whole-array omission.

## Audience Restriction

A proof is carried with the outer token to whichever audiences the outer token serves; proofs have no independent audience scoping ({{proof-claims}}).  Deployments needing audience-specific disclosure constraints SHOULD partition proof issuance by audience at issuance time rather than relying on proof-level audience restriction, which this profile does not provide.

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

The examples in this appendix show decoded proof contents.  The `iat` and `exp` values are illustrative; proof `exp` is set so that no inbound proof expires before the outer token that carries it ({{extending-an-existing-proof-chain}}).  The delegation scenario continues the two-hop travel example of {{I-D.mcguinness-oauth-actor-receipts}}: the subject alice delegates to an AI travel-assistant agent through the enterprise AS, and the agent's token is exchanged at the travel-provider AS, which adds a booking tool as the new outermost actor.

## Example: Two-Hop Delegation Chain with Sibling Receipts

The following example shows an outer token that carries both companions:

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

The booking tool signed `actor_proofs[0]` when it requested the exchange at the travel-provider AS, before that AS issued the outer token:

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

The AI agent signed `actor_proofs[1]` earlier, when the enterprise AS added it as the first actor hop:

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

The sibling receipts follow the same construction as the examples of {{I-D.mcguinness-oauth-actor-receipts}}, with the `proof_jti` claim defined in {{sibling-receipt-issuance}} included at receipt creation.  Because they carry `proof_jti`, these receipts have their own `jti` values, and the newest receipt's `prh` hashes a different older receipt.  The newest receipt, signed by the travel-provider AS, carries:

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
*  Receipt `proof_jti` values link the two chains, and the newest receipt's `origin_jti` anchors the current token instance.

The recipient verifies each proof against a pre-established key registered for its actor ({{actor-key-resolution}}).  Receipt binding does not prevent a compromised issuer from signing a replacement receipt.

## Example: Proofs-Only Partial Coverage

In this example, the enterprise AS has not yet deployed proof support, so no proof exists for the AI-agent hop, and the travel-provider AS accepts a proof from the booking tool when it adds the tool as the new outermost actor.  The resulting access token carries a one-element proof chain and no receipts:

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

The `prh` claim is omitted because this is a single-element chain.  `actor_proofs_complete: false` signals that the inner AI-agent hop carries no actor-signed evidence.  Resource servers that set `actor_proofs_complete_required: true` reject this token; others validate the booking tool's signed participation and its consent to the token's audience, and treat the agent hop as carried solely by the visible `act` chain.  The token carries no resource claim, so a recipient that cannot determine its effective resources from other context does not infer consent to `target.resource` (step 9 of {{consumer-processing}}).  Because no receipts are present, the proof chain carries no outer-token instance binding; per {{proof-to-token-binding-limits}}, a recipient requiring instance binding would require the receipts companion or a provisioned `origin_jti`.

# Document History
{:numbered="false"}

[[ To be removed from the final specification ]]

-01

* Restructured and tightened the text: each rule has one home, dependencies are cited rather than restated, scope and related work are in the Introduction, and Security Considerations point to the rules they rely on.
* Defined one lifetime rule for extension, reissuance, and refresh, and made an expired older proof invalid; retained proofs are validated against the issuer's state on refresh.
* Clarified instance binding: a new `jti` diverges from a provisioned `origin_jti`, trusted-reissuer designation excuses only that divergence unless retargeting is permitted, and the binding options are described by the trust each relies on.
* Tightened target binding: resource indicators match by simple string comparison, `target.resource` supplies the effective resources when a request names none, consent is audience-only when the token's resources are unknown, Token Exchange targets are not narrowed, and the issuer checks `origin_jti` and `receipt_jti`.
* Added guidance for proofs that need to survive assertion-grant redemption.
* An issuer adding a hop without a valid new proof drops the inbound proofs, a request that adds no hop but carries `actor_proof` is rejected, and a reissuer validates proofs before carrying them forward.
* A failed proof check removes only actor-signed evidence unless policy or metadata requires proofs.
* Prohibited `aud` in proofs.
* Removed receipt-attested presenter keys as an actor-key source, and rejected a chain with any untrusted signing key.
* Aligned error codes with {{RFC8693}} and {{RFC7523}}, and required `actor_unauthorized` for actor-authorization failures.
* Clarified completeness and introspection: `actor_proofs_complete` after extension depends on the proof count, an introspection `false` makes no completeness attestation, and filtering a covered actor omits the proofs.
* Compared `sub_profile` values as sets with a matching issuer check, and had deployment configuration supply the `act.iss` the actor signs.
* Allowed a TTS to include `jti`, carried `actor_proof` in Transaction Token requests, and deferred Transaction Token rejection at the resource server to the deployment.
* Checked `alg` and `typ` before key resolution.
* Removed BCP 14 keywords from guidance no other party can observe, named the IETF as change controller, and aligned the examples with the base profile.

-00

* Initial version.

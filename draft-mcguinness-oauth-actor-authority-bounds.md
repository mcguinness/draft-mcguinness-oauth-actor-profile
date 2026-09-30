---
title: "OAuth Actor Chain Authority Bounds"
abbrev: "OAuth Actor Bounds"
category: std
docname: draft-mcguinness-oauth-actor-authority-bounds-latest
submissiontype: IETF
number:
ipr: "trust200902"
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - oauth
 - delegation
 - actor
 - authority
 - attenuation
 - monotonicity
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-actor-profile"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-authority-bounds.html"

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
  RFC7662:
  RFC8414:
  RFC8693:
  RFC8707:
  RFC8725:
  RFC9396:
  RFC9728:
  I-D.mcguinness-oauth-actor-profile:
  I-D.mcguinness-oauth-actor-receipts:
  I-D.mcguinness-oauth-actor-proofs:

informative:
  RFC9700:
  PIC-MODEL:
    title: "PIC Model Specification (Provenance Identity Continuity)"
    author:
      -
        organization: "PIC Protocol Project"
    date: 2026
    target: "https://github.com/pic-protocol/pic-spec"

...

--- abstract

This document defines OAuth Actor Chain Authority Bounds, an optional companion to the OAuth Actor Profile for Delegation and Actor Receipts.  Receipt claims record authority at each hop so recipients can detect expansion in `scope`, `resource`, and `authorization_details`, and in `aud` where audience is governed, unless an explicit re-authorization establishes new bounds.  Signed JSON Web Token (JWT) events record re-authorization between hops.  This document specifies comparison rules, issuer self-attestation, metadata, and introspection parameters.

--- middle

# Introduction

The OAuth Actor Profile {{I-D.mcguinness-oauth-actor-profile}} identifies delegated actors.  Actor Receipts {{I-D.mcguinness-oauth-actor-receipts}} attest issuer participation, and Actor Proofs {{I-D.mcguinness-oauth-actor-proofs}} attest actor participation and target consent.  OAuth 2.0 Token Exchange {{RFC8693}} does not define a record of authority changes across hops, and these profiles do not add one.  Actor Receipts lists historical authority as a non-goal.  An actor proof binds only the target its actor authorized at one hop.  A recipient can therefore verify who participated at every hop without detecting that an intermediate issuer widened the authority flowing through the chain.

This document defines OAuth Actor Chain Authority Bounds, an optional companion profile that closes that gap for deployments that use actor receipts.  Receipt claims record the authority in effect at each hop; recipients compare those values across hops and against the current token, and expansion requires an explicit re-authorization, recorded either in a new receipt or in a signed event between hops.  The design center is:

*  keep the visible actor chain in `act` and per-hop provenance in `actor_receipts`;
*  carry per-hop authority as receipt claims, so it inherits the receipt issuer's signature and the chain's integrity;
*  make authority expansion an explicit, signed, auditable event rather than a silent change.

The profile adds bounds claims, an event array, issuer self-attestation, and discovery metadata; deployments opt in per resource or trust domain.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses OAuth terminology from {{RFC6749}} and {{RFC8693}}.  Actor Receipt, Receipt Chain, and Outer Token follow {{I-D.mcguinness-oauth-actor-receipts}}.  `receipt[i]` denotes entry i of the outer token's `actor_receipts` array, which that document orders newest first: `receipt[0]` is the receipt for the newest hop, whose actor is the outermost `act`.  AS, RS, and TTS denote authorization server, resource server, and Transaction Token Service.

The following terms are used in this document:

Governed Dimension:
: An authority dimension whose per-hop values are recorded and compared under this profile.  This document defines four: `scope`, `aud`, `resource`, and `authorization_details`.

Monotonic Dimension:
: A governed dimension whose recorded values are required not to expand across hops.  `scope`, `resource`, and `authorization_details` are monotonic by default; `aud` is recorded but not monotonic by default (see {{audience-governance}}).

Authority Bounds:
: The values of governed dimensions in effect for the token issued at a given hop, recorded in the receipt's `bounds` claim.

Re-Authorization Event:
: An explicit event in which an authorized principal consents to new, possibly broader, authority.  Recorded either in a receipt's `reauthorized` claim (when the event coincides with a new actor hop) or in a `bounds_events` entry (when it does not).

Monotonicity Basis:
: The point in the chain from which non-expansion is measured.  The origin hop is the initial basis; each re-authorization event establishes a new basis.

Effective Upper Bound:
: For a given dimension and comparison point, the value that the newer artifact's authority must not exceed: the older receipt's recorded bound, as superseded by any re-authorization events anchored to that receipt ({{consumer-processing}}).

Dense Coverage:
: A condition in which every receipt in the chain carries `bounds` for a given dimension, so that monotonicity for that dimension is verified across every adjacency.

Examples in this document are illustrative and omit unrelated claims, signatures, and validation steps that a complete deployment would need.

# Relationship to the Receipts Companion

This profile uses three extension points in {{I-D.mcguinness-oauth-actor-receipts}}:

*  Receipt claims `bounds` and `reauthorized`, protected by the receipt signature.
*  Comparisons across receipts, allowing sparse coverage unless completeness is required.
*  Events between hops, carried in `bounds_events` with a distinct JWT type and receipt-identifier anchors.

Receipt signing, linkage, byte preservation, and coverage rules continue to apply.  Receipts without bounds remain valid.  Independent bounds evidence requires a validated receipt chain; without one, only the weaker {{issuer-attestation}} is available.

# Design Goals and Non-Goals

The goals of this document are:

*  record the authority values in effect at each receipt-covered hop, signed by that hop's issuer;
*  let recipients verify offline that monotonic dimensions never expanded across the covered chain except at explicit re-authorization events;
*  make re-authorization a signed, anchored, ordered artifact rather than an out-of-band assumption, including re-authorization that occurs between hops;
*  compose with the receipts companion's coverage, disclosure, and introspection machinery, and with the proofs companion's actor-consented target bindings;
*  support progressive deployment, including sparse per-dimension coverage.

The non-goals of this document are:

*  interpreting scope grammars: comparison is syntactic set membership, and semantic subsumption is out of scope ({{scope-dimension}});
*  constraining the origin issuer's initial authority choice; no upstream value exists to compare against;
*  defining cross-domain equivalence of authority vocabularies; a trust-domain boundary is an explicit basis reset ({{domain-transitions}});
*  asserting that recorded authority remains active, authorized, or acceptable under current policy;
*  governing `exp`, `cnf`, `sub`, or `sub_profile`, which are lifecycle, presenter, and identity concerns handled by the base profiles;
*  replacing current-token authorization at the resource server.

## Deployment Fit

This profile is most useful where issuers share authority vocabularies.  Changes to scope registries or resource namespaces at domain boundaries require explicit resets ({{domain-transitions}}), limiting comparisons to each domain segment.  Deployments whose chains cross domains at every hop gain recording and audit value from this profile but little enforcement value.

Detecting an intermediate expansion requires bounds on both sides of each compared hop.  Deployments needing that guarantee enforce dense coverage through {{discovery-capability-signaling}}.  Sparse recording still supports audit but leaves gaps in verification.

# Authority Bounds Overview

An issuer that adds an actor hop and supports this profile records, inside the receipt it signs for that hop, the authority values it applied to the issued token.  Recipients walk the validated receipt chain from oldest to newest, comparing recorded values for each governed dimension:

*  values may narrow or stay the same across each adjacency;
*  values may not expand, unless a signed re-authorization event establishes a new basis;
*  the current outer token's values may narrow further relative to the newest recorded bounds, but may not expand.

Two artifacts carry re-authorization:

*  **`reauthorized` receipt claim**: for expansion that coincides with a new actor hop; carried on the new receipt, signed by the hop's issuer.
*  **`bounds_events` outer-token array**: for expansion between hops (a refresh grant with broader scope, a step-up mid-flow, an approver widening a governing authority object); each event is a signed JWT anchored to the receipt whose issued authority it supersedes, and events are hash-chained so their order is cryptographic.

Verified bounds establish non-expansion of recorded authority, subject to re-authorization.  They confer no authority and do not establish that the current request is authorized.

# Governed Dimensions and Comparison Rules {#governed-dimensions}

This section defines the four governed dimensions and the comparison rule for each.  Consumer verification ({{consumer-processing}}) applies these rules.  A comparison is evaluated only when both compared values are present; absence of a recorded bound is missing evidence, not a successful comparison ({{bounds-claim}}).

## `scope` {#scope-dimension}

The `scope` dimension records the space-separated scope string of {{RFC6749}} Section 3.3.  Comparison:

*  parse both values into sets of distinct scope tokens (separator: ASCII space, U+0020);
*  `scope_a` is within `scope_b` if and only if every token in `scope_a` is also in `scope_b`;
*  the empty string is the empty set and is within every scope set.

Comparison does not interpret scope semantics: `read:user/*` does not automatically cover `read:user/123`, and preserved strings may acquire broader meanings through configuration changes ({{scope-subsumption-gaps}}).  Deployments whose scope grammars carry hierarchy or wildcard semantics follow {{scope-subsumption-gaps}} rather than relying on grammar-dependent subsumption.

## `aud` and Audience Governance {#audience-governance}

The `aud` dimension records the audience of the token issued at the hop, as a string or array of strings with the value space of {{RFC7519}} Section 4.1.3.  Comparison treats a single string as a one-element set; `aud_a` is within `aud_b` if and only if every value in `aud_a` is in `aud_b`.

Audience is recorded without monotonicity enforcement by default because token exchange commonly retargets tokens.  Recorded audiences remain useful for audit and comparison with actor-consented targets ({{composition-with-proofs}}).

A deployment whose chains do not retarget, or that treats retargeting as a policy-controlled event, MAY require audience governance through the metadata in {{discovery-capability-signaling}} or local policy.  It then applies the same monotonicity rules to `aud`; retargeting MUST be covered by re-authorization or verification fails.

## `resource` {#resource-dimension}

`bounds.resource` records the effective resource-indicator set applied by the issuer: an array of absolute URIs using {{RFC8707}} semantics.  It records the set even when the token has no corresponding claim.

Comparison:

*  compare URIs by canonical form per {{RFC3986}} Section 6.2; implementations SHOULD apply case normalization for scheme and host, percent-encoding normalization, and path-segment normalization before comparison;
*  `resource_a` is within `resource_b` if and only if every canonical URI in `resource_a` is also in `resource_b`;
*  an empty array is the empty set and is within every resource set.

URI prefix subsumption (for example, treating `https://api.travel-provider.example/v1/` as covering `https://api.travel-provider.example/v1/users`) is NOT applied.  Issuers wishing to express prefix relationships MUST emit explicit URIs at each hop.

## `authorization_details` {#rar-dimension}

The `authorization_details` dimension records the Rich Authorization Requests array of {{RFC9396}}.  Refinement is per-object and per-type:

*  an array `ad_a` refines `ad_b` if and only if every object in `ad_a` refines some object in `ad_b`; objects present in `ad_b` but absent from `ad_a` represent narrowing and are permitted; objects in `ad_a` that refine no object in `ad_b` represent expansion and fail the comparison;
*  two objects refine only when they share the same `type`;
*  for the common members defined by {{RFC9396, Section 2.2}}, refinement requires: `actions` a subset, `locations` a subset under the URI rules of {{resource-dimension}}, `datatypes` a subset, `privileges` a subset, and `identifier` equal; a common member present in the `ad_b` object but absent from the `ad_a` object is expansion and fails the comparison, and one absent from the `ad_b` object but present in the `ad_a` object is narrowing.

For type-specific members the recipient cannot evaluate, the recipient MUST reject verification of that dimension by default, or skip the object's refinement under explicit local policy.  RAR type specifications SHOULD define their own refinement rules; see {{extensibility}}.

## Dimensions Explicitly Not Governed

This profile does not govern token lifetime (`exp`), presenter binding (`cnf`), subject identity (`sub`), or entity classification (`sub_profile`).  The base profiles define those rules.

# Receipt Extension Claims

This section defines two extension claims for Actor Receipt JWTs, under the extension-claims rule of {{I-D.mcguinness-oauth-actor-receipts}}.  Both inherit the receipt's signature and byte-preservation.

## The `bounds` Claim {#bounds-claim}

`bounds`:
: OPTIONAL.  A JSON object recording the authority bounds in effect for the token issued at this hop.  Members correspond to the governed dimensions:

  *  `scope`: a string of space-separated scope tokens;
  *  `aud`: a string or array of strings;
  *  `resource`: an array of URI strings;
  *  `authorization_details`: an array of {{RFC9396}} objects.

  Each member, when present, MUST equal the corresponding effective value applied to the token issued at this hop: for `scope`, `aud`, and `authorization_details`, ordinarily the issued token's top-level claim of the same name; for `resource`, the resource-indicator set the issuer applied whether or not the token carries a claim.

  An absent member is missing evidence, not an assertion that authority was unconstrained or unchanged.  Recipients MUST treat it accordingly.  Additional dimensions MAY be registered under {{iana-dimensions}}; consumers MUST ignore unrecognized members unless a specification or local agreement supplies their comparison rule.

A receipt MAY omit `bounds` entirely, and a chain MAY mix receipts with and without it.  Consumer processing defines both sparse verification and dense enforcement ({{consumer-processing}}).

## The `reauthorized` Claim {#reauthorized-claim}

`reauthorized`:
: OPTIONAL.  A JSON object recording that the authority issued at this hop was expanded relative to the inbound authority under an explicit re-authorization.  Members:

  `sub`:
  : REQUIRED.  Identifier of the principal that re-authorized the delegation.  It MUST equal the top-level `sub` of the token issued at this hop, or an upstream subject identifier recorded in the chain.

  `iss`:
  : REQUIRED.  Identifier of the authorization server or other authority that captured the re-authorization.

  `method`:
  : REQUIRED.  A value from the re-authorization methods registry established in {{iana-methods}}, or a collision-resistant URI.  Initial registered values:

    *  `interactive_consent`: the principal re-authorized through an interactive prompt;
    *  `step_up`: re-authorization through step-up authentication;
    *  `refresh_grant`: re-issuance under a refresh grant carrying broader authority than the prior access token;
    *  `policy_grant`: programmatic re-authorization under a deployment policy artifact;
    *  `domain_transition`: the hop crosses a trust-domain boundary at which authority vocabularies change ({{domain-transitions}}).

  `iat`:
  : REQUIRED.  Time of the re-authorization event.

  `artifact`:
  : OPTIONAL.  A URI or token identifier referencing an external artifact evidencing the re-authorization (for example, a consent record or step-up assertion).  Recipients MAY resolve and validate the artifact under local policy; {{reauthorization-abuse}} covers when deployments require it.

When `reauthorized` is present on a receipt, that hop is a new monotonicity basis: the hop's bounds are not compared against older bounds, and newer artifacts are compared against the post-re-authorization bounds ({{consumer-processing}}).  A receipt carrying `reauthorized` MUST also carry `bounds` recording the post-event value for every governed dimension the event expanded, and SHOULD carry `bounds` for every governed dimension in effect at the hop.

# Bounds Events {#bounds-events}

Re-authorization at a new hop is recorded in that hop's receipt.  Between hops, it is recorded in the `bounds_events` array, using the event pattern from {{I-D.mcguinness-oauth-actor-receipts}}.

## The `bounds_events` and `bounds_events_complete` Claims

`bounds_events`:
: OPTIONAL.  A top-level JWT claim on the outer token.  An array of strings, each the compact serialization {{RFC7515}} of a signed bounds-event JWT as defined below.  When present, the array:

  *  MUST NOT be empty;
  *  MUST be ordered from newest event to oldest event;
  *  is preserved and extended as described in {{event-lifecycle}}.

`bounds_events_complete`:
: OPTIONAL.  A boolean JWT claim on the outer token.  When `true`, the issuer attests that `bounds_events` contains every non-hop bounds-changing event that occurred during the delegation lifetime as of issuance.  When `true` and `bounds_events` is absent, the issuer attests that no such event occurred.  When `false` or absent, coverage may be partial and recipients MUST NOT infer from the absence of events that no re-authorization occurred.  This is the `<name>_complete` member of the receipts companion's claim-pair convention, except that `true` without `bounds_events` attests a complete history with no events.

## Bounds-Event JWT Format

The JOSE header of a bounds event:

*  MUST include an asymmetric digital-signature `alg` value, and MUST NOT use `alg: none` or a MAC-based symmetric algorithm;
*  MUST include `typ` with the value `bounds-event+jwt`;
*  SHOULD include `kid` when the event issuer publishes multiple verification keys;
*  MAY include `crit`; an event whose `crit` header lists an extension header the consumer does not understand is invalid per {{RFC7515, Section 4.1.11}}.

The JWT payload of a bounds event:

`iss`:
: REQUIRED.  The authority that captured the re-authorization and signed the event.  Recipients evaluate event issuers under the trust rules of {{reauthorization-abuse}}.

`event_type`:
: REQUIRED.  The value `bounds_reauth` for events defined by this document.  Companion profiles defining other event types use their own event arrays per the receipts companion's non-hop pattern, not this array.

`receipt_jti`:
: REQUIRED.  The `jti` of the receipt to which this event is anchored: the newest receipt in the chain at the time of the event.  The event supersedes that receipt's recorded bounds for everything newer than that receipt.  This claim is reused from {{I-D.mcguinness-oauth-actor-proofs}} with its registered semantics generalized to event anchoring ({{iana-jwt-claims}}).

`new_bounds`:
: REQUIRED.  A JSON object with the same shape and member rules as the `bounds` receipt claim ({{bounds-claim}}), recording the authority in effect after the event.  At least one governed dimension MUST be present.

`reauthorized`:
: REQUIRED.  A JSON object with the same structure and requirements as the `reauthorized` receipt claim ({{reauthorized-claim}}), recording who authorized the change, how, and when.

`prh` / `prh_alg`:
: The linkage construction from {{I-D.mcguinness-oauth-actor-receipts}}, applied independently to the event array.

  *  Each non-oldest event MUST include `prh` hashing the next older event's exact compact serialization; the oldest MUST omit `prh`.
  *  Events MUST use the same `prh_alg` value or all omit it.  The SHA-256 default and agility rules apply unchanged.
  *  Recipients MUST use chain order, not `iat`, to order events.

`iat`, `exp`, `jti`:
: REQUIRED, as defined in {{RFC7519}}.  `exp` MUST cover the expected maximum lifetime of any token that will carry this event.

An event MAY contain additional claims; consumers ignore unrecognized claims unless a specification or local agreement defines their meaning, per {{RFC7519, Section 4}}.

## Event Lifecycle

The re-authorizing authority creates and signs an event, and the issuer of the next token carrying the array prepends it.  Every party that carries, stores, or forwards the array MUST preserve each inherited event string byte-for-byte and keep the inherited events in their inherited order; byte preservation applies to the event strings, not to the encoding of the enclosing JSON array.  Parties MUST NOT otherwise add events, and MUST NOT remove them.  An issuer unable to preserve the inherited array MUST omit it and `bounds_events_complete` entirely.  Removing entries breaks linkage or conceals a change of bounds.

# Issuer Self-Attestation {#issuer-attestation}

`authority_bounds_enforced`:
: OPTIONAL.  A top-level JWT claim on the outer token: a non-empty array of governed-dimension names.  For each named dimension, the issuer attests that, at issuance, the issued token's effective value was no broader than the corresponding effective inbound value, and that any expansion was covered by a re-authorization event recorded per this profile.

This claim is an unverifiable self-attestation by the same issuer that signed the outer token; it adds no independent evidence, and a compromised issuer can assert it freely ({{issuer-attestation-limits}}).  Its defined uses are:

*  **Consistency check.**  When the token also carries receipt-attested bounds for a named dimension, step 7 of {{consumer-processing}} rejects the token's bounds-based evidence unless chain verification succeeds for that dimension.
*  **Deployment coordination.**  In deployments without receipt-attested bounds, the claim records which dimensions the issuer applied monotonicity to, for recipients whose local policy chooses to rely on issuer trust alone.

Absence of the claim, or of a dimension from it, does not assert that authority expanded; it means the issuer made no attestation for that dimension.

# Issuer Processing

This section defines how an authorization server or Transaction Token Service records, checks, and re-bases authority bounds.  It extends the issuer processing of {{I-D.mcguinness-oauth-actor-receipts}}; all receipt creation, extension, preservation, and reissuance rules of that document apply unchanged.

## Recording Bounds at a New Hop {#recording-bounds}

When an issuer adds a new outermost actor hop and creates the receipt for it, and the deployment uses this profile, the issuer:

1.  MUST determine the issued token's effective `scope`, `aud`, `resource`, and `authorization_details` under the underlying grant rules.
2.  MUST include in the new receipt's `bounds` each dimension it attests, with each member equal to the effective issued value per {{bounds-claim}}.  An issuer that declares a dimension in `authority_bounds_enforced` MUST record that dimension's bound in the receipt when receipts are in use.
3.  For each monotonic dimension it enforces, MUST verify that the issued value is within the effective inbound value, and, when validated inbound receipts carry bounds for the dimension, within the effective upper bound derived from the newest applicable inbound receipt and any anchored events ({{consumer-processing}}).
4.  When step 3 fails and the deployment holds an authoritative re-authorization for the expansion, MAY proceed by recording `reauthorized` on the new receipt per {{reauthorized-claim}}; otherwise MUST reject the request under {{error-handling}}.
5.  MAY set `authority_bounds_enforced` on the issued token, listing exactly the dimensions for which it performed step 3 or recorded a covering re-authorization.

## Reissuance and Refresh Without a New Hop {#reissuance-and-refresh}

Reissuance without a new actor hop creates no receipt, so recorded bounds cannot change through the receipt chain.  An issuer that reissues or refreshes while carrying a bounds-bearing receipt chain forward:

*  MUST NOT issue an outer token whose value for any monotonic dimension exceeds the effective upper bound derived from `receipt[0]` and any events anchored to it; narrowing further is always permitted;
*  when a broader value is authorized (for example, a refresh grant following step-up or an approver widening a governing authority object), MUST record the expansion as a `bounds_reauth` event in `bounds_events`, anchored to the inherited `receipt[0]` by its `jti`, signed by the authority that captured the re-authorization, and prepended per {{bounds-events}}; the issued token's values are then measured against the event's `new_bounds`;
*  when it can neither stay within the effective upper bound nor record a covering event, MUST fail the request or, where local policy and resource requirements permit absent receipt coverage, drop the inherited `actor_receipts` array and with it the bounds evidence, omitting `bounds_events` and `bounds_events_complete` as well.

An AS that supports refresh tokens for bounds-bearing delegated tokens needs to retain, in the same issuer-controlled state that the receipts companion requires for `actor_receipts`, the inherited `bounds_events` array and enough state to compute the effective upper bound at refresh time.

## Domain Transitions {#domain-transitions}

A domain boundary can change the authority vocabulary, making syntactic comparison unsuitable.  This profile records an explicit reset rather than assuming equivalence.

An issuer that adds a hop whose authority vocabulary differs from the inbound token's:

*  MUST record `reauthorized` on the new receipt with `method: domain_transition`, establishing a new monotonicity basis at the boundary;
*  MUST record the new domain's authority values in the new receipt's `bounds`;
*  SHOULD reference, via `reauthorized.artifact`, the policy or agreement under which the cross-domain translation is authorized.

Comparison applies within each domain segment.  A recipient requiring end-to-end monotonicity MUST reject chains containing `domain_transition` bases unless explicit trusted mappings establish cross-domain equivalence.

## Partial and Sparse Coverage

A partial receipt chain can record bounds on any subset of its receipts.  Consumer processing compares only covered adjacencies; required dimensions need dense coverage.  Deployments needing origin-hop evidence should enable recording at the origin issuer first.

# Consumer Processing {#consumer-processing}

An issuer, resource server, or other recipient relying on this profile MUST perform the following steps:

1.  Validate the outer token and receipt chain under {{I-D.mcguinness-oauth-actor-receipts}}.  Bounds in receipts that fail that validation MUST NOT be used.

2.  Check claim types:
    *  `bounds` and `new_bounds` are objects whose recognized members have the types defined in {{governed-dimensions}}.
    *  `reauthorized` contains its required, correctly typed members.
    *  `authority_bounds_enforced` and `bounds_events`, when present, are non-empty arrays of strings.
    *  `bounds_events_complete`, when present, is a boolean.  A value of `true` without `bounds_events` attests a complete history with no events ({{bounds-events}}).

3.  Validate events in array order:
    *  Parse each compact JWT and verify that the event issuer is acceptable under the recipient's re-authorization trust policy ({{reauthorization-abuse}}) before any network retrieval keyed by event content.
    *  Resolve the key and validate the signature, `typ: bounds-event+jwt`, and `crit`; reject unrecognized critical headers.
    *  Check required claims and their types: `iss`, `event_type`, `receipt_jti`, `new_bounds`, `reauthorized`, `iat`, `exp`, and `jti`.
    *  Enforce `iat` and `exp`, require `event_type: bounds_reauth`, and verify that `receipt_jti` names a receipt present in the validated chain.  Reject unresolved anchors.
    *  Verify linkage under {{bounds-events}}: each non-oldest event hashes the next older event, all use one `prh_alg` or omit it, and the oldest omits `prh`.

4.  Compute the effective upper bound for each dimension D at each receipt `receipt[k]` that carries `bounds[D]`.  Start with `receipt[k].bounds[D]`.  If events anchored to that receipt contain D, use `new_bounds[D]` from the newest such event in chain order.

5.  Compare adjacent receipts for each monotonic dimension, including `aud` when governed:
    *  Compare only pairs where both receipts record D.
    *  Skip comparison when the newer receipt carries `reauthorized`, establishing a new basis.
    *  Otherwise, the newer receipt's `bounds[D]` must be within the older receipt's effective upper bound under {{governed-dimensions}}.  Failure MUST reject bounds evidence.

6.  Compare the current token with `receipt[0]` for each D that `receipt[0]` carries in `bounds` and whose effective token value is available from claims, introspection, or trusted context.  The token's value MUST be within the receipt's effective upper bound.  Skip `resource` comparison when its effective value cannot be determined.

7.  Check `authority_bounds_enforced`, if present.  Every named dimension MUST be recognized; unlike an unrecognized `bounds` member, which is ignored, an unrecognized name here is an enforcement claim the recipient cannot verify, as with an unrecognized `crit` header parameter ({{RFC7515, Section 4.1.11}}).  For each named dimension for which the chain carries any receipt-attested bounds, steps 4 through 6 MUST succeed; inconsistency MUST reject bounds evidence.

8.  Enforce dimensions required by `authority_bounds_required` or local policy.  Each required D must appear in `authority_bounds_enforced`, be recorded on every receipt, and pass steps 4 through 6.  Sparse coverage does not satisfy this requirement.  Full-chain enforcement also needs complete receipt and event coverage ({{protected-resource-metadata}}).

9.  Apply any additional rules defined by companion profiles whose claims appear in the artifacts ({{extensibility}}).  They can add rejection conditions but cannot relax any requirement needed for conformance to this profile.

If any required check fails, the recipient MUST reject the token's bounds-based evidence and MUST apply the underlying protocol's error handling for the stage at which the failure occurred.  Rejection of bounds-based evidence does not by itself invalidate the receipt chain under {{I-D.mcguinness-oauth-actor-receipts}}; whether the token remains acceptable without bounds evidence is local policy, except where step 8 applies.

## Composition with Actor-Signed Hop Proofs {#composition-with-proofs}

When the token also carries `actor_proofs` validated under {{I-D.mcguinness-oauth-actor-proofs}}, recorded bounds and actor-consented target bindings are comparable at each hop covered by both artifacts.  For each index i covered by a bounds-bearing receipt and a proof:

*  when `receipt[i].bounds.aud` is present, it MUST be within `actor_proofs[i].target.aud`;
*  when both `receipt[i].bounds.resource` and `actor_proofs[i].target.resource` are present, the recorded set MUST be within the consented set;
*  when both `receipt[i].bounds.scope` and `actor_proofs[i].target.scope` (defined below) are present, the recorded scope set MUST be within the consented scope set.

A failed comparison means the issuer recorded authority broader than the actor consented to at that hop; recipients validating both companions MUST treat it as a failed required check for both artifacts' evidence.

This document defines one extension member for the proof `target` object, under the constraining-extension rule of {{I-D.mcguinness-oauth-actor-proofs}}:

`target.scope`:
: OPTIONAL.  A string of space-separated scope tokens the actor authorizes for the token issued at its hop, compared under the rules of {{scope-dimension}}.  As a constraining member, its presence narrows the actor's target binding; consumers that do not recognize it ignore it per {{I-D.mcguinness-oauth-actor-proofs}}.

## Use by Resource Servers

Bounds evidence records non-expansion across covered hops, with explicit re-authorization for expansion.  An RS MUST still evaluate the current token under current policy; verified history alone does not authorize access.

## Introspection {#consumer-introspection}

Receipt-attested bounds travel inside receipts and are returned wherever receipts are returned; the introspection rules of {{I-D.mcguinness-oauth-actor-receipts}} apply unchanged, including all-or-nothing receipt disclosure and the requirement list for outer-token members.

An introspection response MAY include `authority_bounds_enforced`, `bounds_events`, and `bounds_events_complete` using their JWT syntax.  It MUST return the full stored event array or omit both event claims, except that, for a token whose stored history is complete and contains no events, it MAY return `bounds_events_complete: true` without `bounds_events`.  A subset would break linkage or conceal a change of bounds.

For inactive tokens, introspection servers MUST NOT return `authority_bounds_enforced`, `bounds_events`, or `bounds_events_complete`.

# Discovery and Capability Signaling {#discovery-capability-signaling}

This section defines metadata for advertising authority-bounds support.  It follows the discovery conventions of {{I-D.mcguinness-oauth-actor-receipts}}, with dimension-valued parameters where a boolean would hide which dimensions are covered.

## Authorization Server Metadata

The following parameters are defined for use in Authorization Server Metadata {{RFC8414}}:

`authority_bounds_supported`:
: OPTIONAL.  A non-empty array of governed-dimension names.  The authorization server advertises that, for each named dimension, it can record receipt-attested bounds and enforce issuance-time monotonicity per {{recording-bounds}}.  Absence, or absence of a dimension from the array, means no such advertisement; omission makes no claim of support.

`bounds_events_supported`:
: OPTIONAL.  A boolean.  When `true`, the authorization server can create, preserve, and return `bounds_events` per this document.

Both parameters apply equally to a Transaction Token Service publishing metadata through the same framework.

## Protected Resource Metadata

The following parameters are defined for use in Protected Resource Metadata {{RFC9728}}:

`authority_bounds_required`:
: OPTIONAL.  A non-empty array of governed-dimension names.  For each named dimension, the resource server requires the dense receipt-attested enforcement of consumer step 8: `authority_bounds_enforced` naming the dimension, `bounds` for the dimension on every receipt, and successful verification.  Naming `aud` enables audience governance ({{audience-governance}}).  This is a deployment policy declaration, satisfied by configuring the authorization servers that serve the resource; clients MAY combine it with `authority_bounds_supported` to select an AS.

`bounds_events_complete_required`:
: OPTIONAL.  A boolean.  When `true`, the resource server requires `bounds_events_complete: true` on the outer token or introspection response whenever bounds evidence is presented, so that the absence of events is itself attested ({{bounds-events}}).  This document deliberately defines no `bounds_events_required` parameter: a recipient cannot observe whether unrecorded events occurred, so the only testable requirement is the completeness attestation.

A resource server that needs full-chain rather than covered-prefix enforcement SHOULD pair `authority_bounds_required` with both the receipts companion's `actor_receipts_complete_required` and `bounds_events_complete_required`.  Neither completeness signal attests that the visible `act` chain is itself unfiltered; that separate assurance is `chain_complete` ({{I-D.mcguinness-oauth-actor-profile}}).

## Introspection Response Members {#introspection-response-members}

The following members are defined for use in OAuth Token Introspection responses {{RFC7662}}, each with the same syntax as the JWT claim of the same name: `authority_bounds_enforced`, `bounds_events`, and `bounds_events_complete`.  Consumer use is described in {{consumer-introspection}}.

# Error Handling {#error-handling}

Bounds validation extends the underlying OAuth or Transaction Token validation.  Failures are reported through the error mechanism applicable to the stage at which they occur.

When an authorization server or Transaction Token Service rejects a token request because inbound bounds evidence fails validation under {{consumer-processing}} (signature failure on an event, broken event chain, unresolvable anchor, monotonicity failure in the inbound chain), it SHOULD return `invalid_grant`, constructed per {{RFC8693}} Section 2.2.2 and {{RFC6749}} Section 5.2, consistent with the core actor profile's error mapping for actor information that fails validation.

When requested authority exceeds the effective upper bound without re-authorization, the issuer SHOULD return:

| Dimension | Error |
|-----------|-------|
| `scope` | `invalid_scope` ({{RFC6749, Section 5.2}}) |
| `aud` or `resource` | `invalid_target` ({{RFC8693, Section 2.2.2}}) |
| `authorization_details` | `invalid_authorization_details` {{RFC9396}} |

When the failure reflects an authorization-policy decision about the actor or delegation rather than a structural failure, an issuer MAY use `actor_unauthorized` as defined in the core actor profile {{I-D.mcguinness-oauth-actor-profile}}.

When a resource server rejects a request because bounds verification fails or required dimensions are unsatisfied, it SHOULD return `invalid_token` per {{RFC6750}} Section 3.1, and SHOULD include an `error_description` identifying bounds-verification failure so operators can distinguish it from generic token validation.

An introspection server does not return an OAuth error for missing bounds artifacts; their presence is a property of the response.  This document defines no new OAuth error codes.

# Extensibility {#extensibility}

This profile composes with the extensibility framework of {{I-D.mcguinness-oauth-actor-receipts}} and adds its own surfaces:

*  **New governed dimensions**, registered in the dimension registry ({{iana-dimensions}}) with a defined comparison rule and a declared governance class (monotonic by default, or record-only).  Consumers ignore unregistered `bounds` members they do not recognize.
*  **New re-authorization methods**, registered in the methods registry ({{iana-methods}}) or expressed as collision-resistant URIs.
*  **Per-type RAR refinement rules**, defined by the specifications that define RAR types; such rules extend {{rar-dimension}} for their types without modifying this document.
*  **New event types** are NOT added to `bounds_events`; companion profiles defining other non-hop events use their own parallel arrays per the receipts companion's pattern, so that each array has one verification routine and one completeness attestation.

Companion rules MUST NOT relax any requirement needed for conformance to this profile; they MAY add rejection conditions.  A companion's partial-validation mode, defined under its own normative scope as {{I-D.mcguinness-oauth-actor-receipts}} requires, is not conformance to this profile.

# Security Considerations

Authority bounds strengthen authority provenance for receipt-covered hops, but they do not replace token validation or authorization.  The general OAuth 2.0 Security Best Current Practice {{RFC9700}} and the JWT best practices in {{RFC8725}} apply.

## Threat Model {#threat-model}

### Adversaries Mitigated by This Profile

*  **Intermediate authority expansion.**  Detection requires both adjacent receipts to record the dimension.  Recording a wider value fails comparison; recording less than issued fails the next enforcing issuer's check or the terminal token comparison; omission fails dense-coverage enforcement.  Sparse coverage leaves gaps in this protection.
*  **Silent basis change.**  Expansion requires a signed artifact: a `reauthorized` claim inside a trusted issuer's receipt or a signed, anchored, chained event.  Dropping an event breaks the event `prh` chain; reordering is prevented by the chain; `bounds_events_complete: true` attests that no events are withheld.
*  **Issuance beyond actor consent, when proofs are present.**  The cross-checks of {{composition-with-proofs}} detect recorded authority broader than the actor-signed target binding at the same hop.

### Adversaries Not Mitigated

*  **Origin issuer choosing broad initial bounds.**  Monotonicity is relative; no upstream value constrains the origin.  Constraint on origin authority requires policy at the origin issuer, pre-authorization artifacts, or transparency mechanisms outside this document.
*  **Fabricated re-authorization by a trusted issuer.**  Any issuer trusted to record re-authorization can convert detected expansion into authorized expansion ({{reauthorization-abuse}}).
*  **Full-chain collusion.**  Colluding issuers fabricate a monotonic chain at any level; this matches the receipts companion's trust boundary.
*  **Semantic expansion within syntactic subsets.**  See {{scope-subsumption-gaps}}.
*  **Cross-domain expansion.**  A `domain_transition` basis reset is exactly an unverified re-expression of authority; recipients requiring end-to-end guarantees must reject or map it ({{domain-transitions}}).
*  **Compromised current outer-token issuer.**  Out of scope here as in the receipts companion; a compromised outer issuer can omit this profile's claims entirely.  Absence of bounds evidence is a downgrade recipients detect only by requiring the evidence ({{discovery-capability-signaling}}).

### Trust Model Summary

Bounds inherit the receipts companion's per-issuer, non-transitive trust model, and add one axis: trust to record re-authorization.  A recipient can trust an issuer's receipts while refusing its `reauthorized` claims and events ({{reauthorization-abuse}}).  Composition with proofs adds an actor-side check with an independent trust anchor.

## Issuer Self-Attestation Limits {#issuer-attestation-limits}

An issuer can assert `authority_bounds_enforced` without independent evidence.  It never satisfies a requirement for receipt-attested verification, such as required-dimension enforcement in step 8 of {{consumer-processing}}.  Deployments requiring offline evidence that authority did not expand MUST use receipt-attested bounds; the claim alone supports only coordination and consistency checking.

## Re-Authorization Abuse {#reauthorization-abuse}

A compromised or over-trusted issuer that is trusted to record re-authorization can use it to justify arbitrary expansion.

*  Recipients MUST evaluate re-authorization trust separately from receipt trust: which issuers are trusted to capture re-authorization, for which subjects, and by which methods, is explicit local policy.  A recipient MAY accept an issuer's receipts while rejecting its re-authorization records; a rejected re-authorization is a failed basis reset, and the chain is then evaluated without it, which typically fails monotonicity and rejects the token's bounds evidence.
*  Deployments needing strong re-authorization integrity SHOULD require `reauthorized.artifact` and SHOULD validate the referenced artifact against the authority that captured the event (for example, verifying a consent receipt's signature), rather than accepting the recording issuer's bare assertion.
*  `domain_transition` bases deserve the most scrutiny: they legitimize non-comparability, and an attacker who can insert one launders any expansion.  Recipients SHOULD restrict which issuers may record domain transitions to the deployment's known boundary issuers.

## Scope Subsumption Gaps {#scope-subsumption-gaps}

Set-membership comparison catches verbatim expansion only.  A scope token that is lexically new at a hop fails the subset check even when semantically narrower, and a lexically preserved token can be semantically broadened by configuration changes at the AS that defines it.  Deployments whose scope grammars carry hierarchy or wildcard semantics MUST either emit explicit narrowest-form scopes at every hop, or define and apply a deployment-specific comparison rule; this document does not define scope subsumption, and a general solution belongs in its own specification.

## Recorded Values and Token Reality

`bounds` members are attested copies of issued-token values, signed by the issuer that produced both.  An issuer that records values differing from what it actually issued produces either a detectable mismatch (the outer-token comparison at the terminal hop, or the next enforcing issuer's inbound check) or a consistent lie spanning its receipt and its token, which is the intermediate authority expansion case in {{threat-model}}.  Step 6 of {{consumer-processing}} compares `bounds` against the effective values the token actually carries, not request-time values.

## Event Chain Size and Retention

The event array grows with each re-authorization.  Deployments need to account for the combined token, receipt, proof, and event size; introspection can avoid carrying this evidence in HTTP request headers.  Retained events also disclose consent and authentication history ({{privacy-considerations}}).

# Privacy Considerations {#privacy-considerations}

The privacy considerations of {{I-D.mcguinness-oauth-actor-receipts}} apply, including cross-service correlation and retention beyond token lifetime.  Bounds add authority-shaped disclosure:

*  `bounds` exposes per-hop scope, audience, resource, and authorization-detail values to every recipient of the token or introspection response, revealing internal permission vocabulary, resource topology, and orchestration structure.  Issuers SHOULD record only the dimensions recipients need, and MAY enforce monotonicity at issuance without recording bounds where disclosure outweighs evidence value.
*  `reauthorized` and `bounds_events` reveal consent prompts, step-up authentication, and policy decisions, with timing; this is sensitive activity metadata.  `bounds_events` is visible at the outer-token level even when receipts are suppressed, so its presence is a distinct disclosure decision.  Deployments that need event history for audit but not for relying parties SHOULD return `bounds_events` via introspection only.
*  A fully bounds-covered chain is a detailed narrative of authority narrowing across an organization; deployments SHOULD scope disclosure to audiences with adequate agreements, using the receipts companion's all-or-nothing granularity deliberately.

# IANA Considerations

## Media Type Registration

This document requests registration of the following media type in the "Media Types" registry {{RFC6838}}:

*  Type name: `application`
*  Subtype name: `bounds-event+jwt`
*  Required parameters: N/A
*  Optional parameters: N/A
*  Encoding considerations: 8bit; a bounds event is a JWS compact-serialized JWT {{RFC7515}} {{RFC7519}} consisting of base64url-encoded segments separated by period (`.`) characters.
*  Security considerations: See {{security-considerations}} of this document and {{RFC8725}}.
*  Interoperability considerations: N/A
*  Published specification: This document
*  Applications that use this media type: Applications that issue, exchange, or validate OAuth Actor Chain Authority Bounds events.
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

The JOSE `typ` value `bounds-event+jwt` is the media type subtype name without the `application/` prefix, following common JWT typing practice.

## JSON Web Token Claims Registration {#iana-jwt-claims}

This document requests registration of the following JWT Claims in the "JSON Web Token Claims" registry {{RFC7519}}:

*  Claim Name: `bounds`
*  Claim Description: Authority bounds in effect for the token issued at the hop attested by an Actor Receipt JWT
*  Change Controller: IETF
*  Specification Document(s): This document

*  Claim Name: `reauthorized`
*  Claim Description: Record of explicit re-authorization of delegated authority at a hop or event
*  Change Controller: IETF
*  Specification Document(s): This document

*  Claim Name: `bounds_events`
*  Claim Description: Array of signed bounds-event JWTs recording non-hop re-authorization of delegated authority
*  Change Controller: IETF
*  Specification Document(s): This document

*  Claim Name: `bounds_events_complete`
*  Claim Description: Boolean indicating whether bounds_events covers every non-hop bounds-changing event as of issuance
*  Change Controller: IETF
*  Specification Document(s): This document

*  Claim Name: `authority_bounds_enforced`
*  Claim Description: Array of authority-dimension names for which the issuer attests issuance-time monotonicity enforcement
*  Change Controller: IETF
*  Specification Document(s): This document

*  Claim Name: `event_type`
*  Claim Description: Type discriminator for a delegation-evidence event JWT
*  Change Controller: IETF
*  Specification Document(s): This document

*  Claim Name: `new_bounds`
*  Claim Description: Authority bounds in effect after the re-authorization recorded by a bounds-event JWT
*  Change Controller: IETF
*  Specification Document(s): This document

This document reuses the `prh` and `prh_alg` claims registered by {{I-D.mcguinness-oauth-actor-receipts}} and the `receipt_jti` claim registered by {{I-D.mcguinness-oauth-actor-proofs}}, applied to bounds-event JWTs as profiled in this document.  This document requests that IANA add this document to the Specification Document(s) entries for those three registrations.  The Claim Description wording requested by {{I-D.mcguinness-oauth-actor-proofs}} for `prh` and `prh_alg` already covers the delegation-evidence artifact family; for `receipt_jti`, this document requests that the Claim Description be updated to:

*  `receipt_jti`: jti of the Actor Receipt JWT that a sibling or anchored delegation-evidence JWT references for the same delegation hop

This document does not request separate registration for the members of the `bounds`, `new_bounds`, `reauthorized`, and proof `target` objects it defines; sub-object keys within a registered claim are scoped to that claim's JSON object, following the convention of {{I-D.mcguinness-oauth-actor-profile}}.

## OAuth Actor Authority Bounds Dimensions Registry {#iana-dimensions}

This document requests that IANA establish a registry titled "OAuth Actor Authority Bounds Dimensions", with the registration policy Specification Required.  Each entry records: Dimension Name, Governance Class (`monotonic` or `record-only`), Comparison Rule reference, and Specification Document(s).  Initial contents:

*  `scope`, monotonic, {{scope-dimension}} of this document
*  `aud`, record-only, {{audience-governance}} of this document
*  `resource`, monotonic, {{resource-dimension}} of this document
*  `authorization_details`, monotonic, {{rar-dimension}} of this document

Designated experts SHOULD verify that a requested dimension has a deterministic comparison rule, a declared governance class, and semantics that do not overlap an existing entry.

## OAuth Actor Re-Authorization Methods Registry {#iana-methods}

This document requests that IANA establish a registry titled "OAuth Actor Re-Authorization Methods", with the registration policy Specification Required.  Each entry records: Method Name, Description, and Specification Document(s).  Initial contents: `interactive_consent`, `step_up`, `refresh_grant`, `policy_grant`, and `domain_transition`, as defined in {{reauthorized-claim}}.  Values not in the registry MUST be collision-resistant URIs.

## OAuth Authorization Server Metadata Registration

This document requests registration of the following metadata names in the "OAuth Authorization Server Metadata" registry {{RFC8414}}:

*  Metadata Name: `authority_bounds_supported`
*  Metadata Description: Array of authority-dimension names for which the server records receipt-attested bounds and enforces issuance-time monotonicity
*  Change Controller: IETF
*  Specification Document(s): This document

*  Metadata Name: `bounds_events_supported`
*  Metadata Description: Indicates support for creating, preserving, and returning bounds-event arrays
*  Change Controller: IETF
*  Specification Document(s): This document

## OAuth Protected Resource Metadata Registration

This document requests registration of the following metadata names in the "OAuth Protected Resource Metadata" registry {{RFC9728}}:

*  Metadata Name: `authority_bounds_required`
*  Metadata Description: Array of authority-dimension names for which the resource requires dense receipt-attested bounds enforcement
*  Change Controller: IETF
*  Specification Document(s): This document

*  Metadata Name: `bounds_events_complete_required`
*  Metadata Description: Indicates that the resource requires an issuer attestation of complete bounds-event coverage
*  Change Controller: IETF
*  Specification Document(s): This document

## OAuth Token Introspection Response Registration

This document requests registration of the following names in the "OAuth Token Introspection Response" registry {{RFC7662}}:

*  Name: `authority_bounds_enforced`
*  Description: Array of authority-dimension names for which the issuer attests issuance-time monotonicity enforcement
*  Change Controller: IETF
*  Specification Document(s): This document

*  Name: `bounds_events`
*  Description: Array of signed bounds-event JWTs returned by introspection
*  Change Controller: IETF
*  Specification Document(s): This document

*  Name: `bounds_events_complete`
*  Description: Indicates whether the returned bounds events provide complete coverage of non-hop re-authorization
*  Change Controller: IETF
*  Specification Document(s): This document

# Acknowledgments

The author thanks the OAuth Working Group for the specifications underlying this profile.  The PIC Model {{PIC-MODEL}} also informed the non-expansion property, which this document represents through signed OAuth evidence.

Contributors and reviewers will be acknowledged in future revisions.

--- back

# Examples

The examples in this appendix show decoded contents; real receipts and events are compact-signed JWT strings.  Timestamps are illustrative.  The scenario continues the two-hop travel example of {{I-D.mcguinness-oauth-actor-receipts}}: Alice delegates to an AI travel-assistant agent through the enterprise AS, and the agent's token is exchanged at the travel-provider AS, which adds a booking tool as the outermost actor.  Because these receipts carry `bounds`, they are different byte strings from the receipts shown in that document's examples and carry their own identifiers.

## Example: Two-Hop Chain with Narrowing Bounds

The outer token:

~~~json
{
  "jti": "3c9f5a1e-7d2b-4e8c-a6f0-1b4d7e0a3c6f",
  "iss": "https://as.travel-provider.example",
  "aud": "https://api.travel-provider.example",
  "scope": "trips:book",
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
  "actor_receipts": [
    "<receipt-0>",
    "<receipt-1>"
  ],
  "actor_receipts_complete": true,
  "authority_bounds_enforced": ["scope", "resource"]
}
~~~

`actor_receipts[1]`, signed by the enterprise AS at the first hop, records the authority granted to the agent:

~~~json
{
  "iss": "https://as.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://agents.enterprise.example/travel-assistant",
    "iss": "https://as.enterprise.example",
    "sub_profile": "ai_agent"
  },
  "bounds": {
    "scope": "trips:read trips:book profile:read",
    "aud": ["https://as.travel-provider.example"],
    "resource": [
      "https://api.travel-provider.example/bookings",
      "https://api.travel-provider.example/trips"
    ]
  },
  "iat": 1776741600,
  "exp": 1776832000,
  "jti": "6e2a8c4f-9b1d-4f7a-8e3c-5a0b2d9f7e1a"
}
~~~

`actor_receipts[0]`, signed by the travel-provider AS when it added the booking tool, records narrower authority:

~~~json
{
  "iss": "https://as.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "act": {
    "sub": "https://tools.travel-provider.example/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service"
  },
  "bounds": {
    "scope": "trips:book",
    "aud": ["https://api.travel-provider.example"],
    "resource": ["https://api.travel-provider.example/bookings"]
  },
  "prh": "Vt7RcW2yQm9ZpKd4Xa6bEu8sHf1jLn3iTg5oAw0eCkY",
  "iat": 1776745200,
  "exp": 1776832000,
  "jti": "8d4b2f6a-1c3e-4a5d-9f7b-0e2c4a6d8f0b",
  "origin_jti": "3c9f5a1e-7d2b-4e8c-a6f0-1b4d7e0a3c6f"
}
~~~

The example verifies as follows:

*  Scope narrows to `trips:book`.
*  Resources narrow to the bookings endpoint.  URI prefixes do not imply containment.
*  The outer token's scope equals the newest recorded scope.
*  Audience changes are recorded without comparison under the default audience rules.

The issuer attests enforcement of `scope` and `resource`, consistent with those checks.  No re-authorization occurred, so the token carries no `bounds_events`; adding `bounds_events_complete: true` would attest that complete, empty history ({{bounds-events}}).

## Example: Refresh Widening Recorded as a Bounds Event

Suppose the travel-provider AS later refreshes the token above with broader scope after Alice completes step-up authentication: the refreshed token carries `scope: "trips:book trips:cancel"`.  No new actor hop exists, so the expansion is recorded as a bounds event anchored to `receipt[0]`:

~~~json
{
  "iss": "https://as.travel-provider.example",
  "event_type": "bounds_reauth",
  "receipt_jti": "8d4b2f6a-1c3e-4a5d-9f7b-0e2c4a6d8f0b",
  "new_bounds": {
    "scope": "trips:book trips:cancel"
  },
  "reauthorized": {
    "sub": "https://idp.enterprise.example/users/alice",
    "iss": "https://as.travel-provider.example",
    "method": "step_up",
    "iat": 1776747800,
    "artifact": "https://as.travel-provider.example/events/su-91f4"
  },
  "iat": 1776747810,
  "exp": 1776832000,
  "jti": "2f8c6e4a-0b1d-4c3e-8a5f-7d9b1e3c5a7f"
}
~~~

The refreshed outer token carries the inherited receipts unchanged, `scope: "trips:book trips:cancel"`, and:

~~~json
{
  "bounds_events": [
    "<event-0>"
  ],
  "bounds_events_complete": true
}
~~~

Verification: the event validates against the AS's key, its `receipt_jti` resolves to `receipt[0]`, and it is the sole event, so it omits `prh`.  The effective upper bound presented by `receipt[0]` for `scope` becomes `trips:book trips:cancel`, and the refreshed outer token's scope is within it.  Without the event, the refreshed token would fail step 6 of {{consumer-processing}}, since `trips:cancel` is not in `receipt[0].bounds.scope`.  A recipient whose re-authorization trust policy does not accept `step_up` events from this AS rejects the basis change, and with it the refreshed token's bounds evidence.

# Document History
{:numbered="false"}

[[ To be removed from the final specification ]]

-00

* Initial version.

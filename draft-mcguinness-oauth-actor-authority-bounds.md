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
  RFC7519:
  RFC7523:
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
  I-D.ietf-oauth-transaction-tokens:
  I-D.niyikiza-oauth-attenuating-agent-tokens:
  PIC-MODEL:
    title: "PIC Model Specification (Provenance Identity Continuity)"
    author:
      -
        organization: "PIC Protocol Project"
    date: 2026
    target: "https://github.com/pic-protocol/pic-spec"

...

--- abstract

This document defines OAuth Actor Chain Authority Bounds, an optional companion to the OAuth Actor Profile for Delegation and Actor Receipts.  Receipt claims record authority at each hop so recipients can detect expansion in `scope` and `authorization_details`, and in `aud` and `resource` where a deployment requires it, unless an explicit re-authorization establishes new bounds.  This document specifies comparison rules and metadata.

--- middle

# Introduction

The OAuth Actor Profile {{I-D.mcguinness-oauth-actor-profile}} identifies delegated actors.  Actor Receipts {{I-D.mcguinness-oauth-actor-receipts}} attest issuer participation, and Actor Proofs {{I-D.mcguinness-oauth-actor-proofs}} attest actor participation and target consent.  OAuth 2.0 Token Exchange {{RFC8693}} does not define a record of authority changes across hops, and these profiles do not add one.  Actor Receipts lists historical authority as a non-goal.  An actor proof binds only the target its actor authorized at one hop.  A recipient can therefore verify who participated at every hop without detecting that an intermediate issuer widened the authority flowing through the chain.

This document defines OAuth Actor Chain Authority Bounds, an optional companion profile that closes that gap for deployments that use actor receipts.  Receipt claims record the authority in effect at each hop; recipients compare those values across hops and against the current token, and expansion requires an explicit re-authorization recorded in the receipt for a new hop.  The design center is:

*  keep the visible actor chain in `act` and per-hop provenance in `actor_receipts`;
*  carry per-hop authority as receipt claims, so it inherits the receipt issuer's signature and the chain's integrity;
*  make authority expansion an explicit, signed, auditable event rather than a silent change.

The profile adds bounds claims and discovery metadata; deployments opt in per resource or trust domain.

Attenuating Authorization Tokens {{I-D.niyikiza-oauth-attenuating-agent-tokens}} address a related problem with a different model: a token holder derives a token with equal or narrower tool-level authority offline, and any enforcement point holding the root issuer's trust anchor verifies the derivation chain.  This profile instead adds evidence to issuance by authorization servers and Transaction Token Services, recording the authority each issuer applied in its signed receipt and permitting expansion only under a signed re-authorization.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses OAuth terminology from {{RFC6749}} and {{RFC8693}}.  Actor Receipt, Receipt Chain, and Outer Token follow {{I-D.mcguinness-oauth-actor-receipts}}.  `receipt[i]` denotes entry i of the outer token's `actor_receipts` array, which that document orders newest first: `receipt[0]` is the receipt for the newest hop, whose actor is the outermost `act`.  AS, RS, and TTS denote authorization server, resource server, and Transaction Token Service.

The following terms are used in this document:

Governed Dimension:
: An authority dimension whose per-hop values are recorded under this profile and, when the dimension is monotonic, compared across hops.  This document defines four: `scope`, `aud`, `resource`, and `authorization_details`.

Monotonic Dimension:
: A governed dimension whose recorded values are required not to expand across hops.  `scope` and `authorization_details` are monotonic by default; `aud` and `resource` are recorded but not monotonic unless a resource server names them in `authority_bounds_required` or local policy requires it ({{audience-governance}}, {{resource-dimension}}).

Authority Bounds:
: The values of governed dimensions in effect for the token issued at a given hop, recorded in the receipt's `bounds` claim.

Re-Authorization Event:
: An explicit event in which an authorized principal consents to new, possibly broader, authority.  Recorded in the `reauthorized` claim of the receipt for the new actor hop at which it occurs.

Monotonicity Basis:
: The point in the chain from which non-expansion of a dimension is measured.  The origin hop is the initial basis; each re-authorization recorded at a hop establishes a new basis for the dimensions it lists.

Dense Coverage:
: A condition in which every receipt in the chain carries `bounds` for a given dimension, so that monotonicity for that dimension is verified across every adjacency.

Examples in this document are illustrative and omit unrelated claims, signatures, and validation steps that a complete deployment would need.

# Relationship to the Receipts Companion

This profile uses two extension points in {{I-D.mcguinness-oauth-actor-receipts}}:

*  Receipt claims `bounds` and `reauthorized`, protected by the receipt signature.
*  Comparisons across receipts, tolerating sparse coverage by verifying a dimension only when every receipt records it.

Receipt signing, linkage, byte preservation, and coverage rules continue to apply.  Receipts without bounds remain valid.  Bounds evidence requires a validated receipt chain.

# Design Goals and Non-Goals

The goals of this document are:

*  record the authority values in effect at each receipt-covered hop, signed by that hop's issuer;
*  let recipients verify offline that monotonic dimensions never expanded across the covered chain except at explicit re-authorizations recorded at a hop;
*  make re-authorization at a hop an explicit, signed, dimension-scoped record rather than an out-of-band assumption;
*  compose with the receipts companion's coverage, disclosure, and introspection machinery, and with the proofs companion's actor-consented target bindings;
*  support progressive deployment: sparse per-dimension recording supports audit, and verifying a dimension needs every receipt to record it.

The non-goals of this document are:

*  interpreting scope grammars: comparison is syntactic set membership, and semantic subsumption is out of scope ({{scope-dimension}});
*  constraining the origin issuer's initial authority choice; no upstream value exists to compare against;
*  defining cross-domain equivalence of authority vocabularies; a trust-domain boundary is an explicit basis reset ({{domain-transitions}});
*  recording re-authorization between hops (a refresh or step-up without a new hop), which a later extension can add ({{extensibility}});
*  asserting that recorded authority remains active, authorized, or acceptable under current policy;
*  governing `exp`, `cnf`, `sub`, or `sub_profile`, which are lifecycle, presenter, and identity concerns handled by the base profiles;
*  replacing current-token authorization at the resource server.

## Deployment Fit

This profile is most useful where issuers share authority vocabularies.  Changes to scope registries or resource namespaces at domain boundaries require explicit resets ({{domain-transitions}}), limiting comparisons to each domain segment.  Deployments whose chains cross domains at every hop gain recording and audit value from this profile but little enforcement value.

Verifying a dimension across the chain requires every receipt to record it.  Deployments needing that guarantee enforce dense coverage through {{discovery-capability-signaling}}.  Sparse recording supports audit, but a dimension recorded on only some receipts is not verified.

# Authority Bounds Overview

An issuer that adds an actor hop and supports this profile records, inside the receipt it signs for that hop, the authority values it applied to the issued token.  Recipients walk the validated receipt chain from oldest to newest, comparing recorded values for each monotonic dimension:

*  values may narrow or stay the same across each adjacency;
*  values may not expand, unless a signed re-authorization recorded at a hop establishes a new basis for that dimension;
*  the current outer token's values may narrow further relative to the newest recorded bounds, but may not expand.

Re-authorization is carried by the `reauthorized` receipt claim, recorded on the receipt for a new actor hop, signed by the hop's issuer, and scoped to the dimensions it lists.

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

`bounds.resource` records the effective resource-indicator set applied by the issuer: an array of absolute URIs using {{RFC8707}} semantics.  It records the set even when the token has no corresponding claim.  An issuer that applied no resource indicator omits `bounds.resource` rather than recording an empty array; where a resource server requires `resource` ({{protected-resource-metadata}}), the omission fails that dimension.

Comparison:

*  compare URIs by simple string comparison ({{RFC3986, Section 6.2.1}}); issuers need to record each resource indicator in the same form at every hop;
*  `resource_a` is within `resource_b` if and only if every URI in `resource_a` is also in `resource_b`;

URI prefix subsumption (for example, treating `https://api.travel-provider.example/v1/` as covering `https://api.travel-provider.example/v1/users`) is NOT applied.  Issuers wishing to express prefix relationships MUST emit explicit URIs at each hop.

`resource` is recorded without monotonicity enforcement by default because Token Exchange uses `resource` to retarget tokens ({{RFC8693, Section 2.1}}).  A deployment MAY require monotonicity for `resource` through the same metadata or local policy as audience governance ({{audience-governance}}).  It then applies the same monotonicity rules to `resource`; retargeting MUST be covered by re-authorization or verification fails.

## `authorization_details` {#rar-dimension}

The `authorization_details` dimension records the Rich Authorization Requests array of {{RFC9396}}.  Refinement is per-object and per-type:

*  an array `ad_a` refines `ad_b` if and only if every object in `ad_a` refines some object in `ad_b`; objects present in `ad_b` but absent from `ad_a` represent narrowing and are permitted; objects in `ad_a` that refine no object in `ad_b` represent expansion and fail the comparison;
*  two objects refine only when they share the same `type`;
*  for the common members defined by {{RFC9396, Section 2.2}}, refinement requires: `actions` a subset, `locations` a subset under the URI rules of {{resource-dimension}}, `datatypes` a subset, `privileges` a subset, and `identifier` equal; a common member present in the `ad_b` object but absent from the `ad_a` object is expansion and fails the comparison, and one absent from the `ad_b` object but present in the `ad_a` object also fails the comparison unless the refinement rules for that `type` establish that it preserves or narrows authority, because the effect of a member depends on the API's semantics ({{RFC9396, Section 6.1}}).

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

A receipt MAY omit `bounds` entirely, and a chain MAY mix receipts with and without it.  A dimension recorded on only some receipts is recorded for audit but not verified across the chain ({{consumer-processing}}).

## The `reauthorized` Claim {#reauthorized-claim}

`reauthorized`:
: OPTIONAL.  A JSON object recording that the authority issued at this hop was expanded relative to the inbound authority under an explicit re-authorization.  Members:

  `sub`:
  : REQUIRED.  Identifier of the principal or authority that re-authorized the delegation, such as the subject, an approver, or the authority whose policy or agreement applies.  When the subject itself re-authorized, it equals the top-level `sub` of the token issued at this hop.  Recipients evaluate whether to trust the re-authorization under {{reauthorization-abuse}}.

  `iss`:
  : REQUIRED.  Identifier of the authorization server or other authority that captured the re-authorization.

  `method`:
  : REQUIRED.  A value from the re-authorization methods registry established in {{iana-methods}}, or a collision-resistant URI.  Initial registered values:

    *  `interactive_consent`: the principal re-authorized through an interactive prompt;
    *  `step_up`: re-authorization through step-up authentication;
    *  `policy_grant`: programmatic re-authorization under a deployment policy artifact;
    *  `domain_transition`: the hop crosses a trust-domain boundary at which authority vocabularies change ({{domain-transitions}}).

  `iat`:
  : REQUIRED.  Time of the re-authorization.

  `dimensions`:
  : REQUIRED.  A non-empty array of the governed dimension names this re-authorization resets.  The receipt MUST carry `bounds` for each listed dimension.

  `artifact`:
  : OPTIONAL.  A URI or token identifier referencing an external artifact evidencing the re-authorization (for example, a consent record or step-up assertion).  Recipients MAY resolve and validate the artifact under local policy; {{reauthorization-abuse}} covers when deployments require it.

When `reauthorized` is present on a receipt, that hop is a new monotonicity basis only for the dimensions listed in `reauthorized.dimensions`: the hop's bounds for a listed dimension are not compared against older bounds, and newer artifacts are compared against the post-re-authorization bounds ({{consumer-processing}}).  Every other dimension is compared as usual.  A receipt carrying `reauthorized` SHOULD carry `bounds` for every governed dimension in effect at the hop.

# Issuer Processing

This section defines how an authorization server or Transaction Token Service records, checks, and re-bases authority bounds.  It extends the issuer processing of {{I-D.mcguinness-oauth-actor-receipts}}; all receipt creation, extension, preservation, and reissuance rules of that document apply unchanged.

## Recording Bounds at a New Hop {#recording-bounds}

When an issuer adds a new outermost actor hop and creates the receipt for it, and the deployment uses this profile, the issuer:

1.  MUST determine the issued token's effective `scope`, `aud`, `resource`, and `authorization_details` under the underlying grant rules.
2.  MUST include in the new receipt's `bounds` each dimension it attests, with each member equal to the effective issued value per {{bounds-claim}}.
3.  For each monotonic dimension it enforces, MUST verify that the issued value is within the effective inbound value, and, when the inbound token's validated `receipt[0]` carries bounds for the dimension, within that receipt's recorded bound.
4.  When the requested authority would fail step 3, MAY narrow the issued `scope` or `resource` value to fit, as {{RFC6749, Section 3.3}} permits for scope and {{RFC8707, Section 2.2}} leaves acceptable resources to its policy, but does not drop a requested audience; steps 1 to 3 then apply to the narrowed value.  When the deployment holds an authoritative re-authorization for the expansion, it MAY instead proceed by recording `reauthorized` on the new receipt, listing each expanded dimension in `reauthorized.dimensions`, per {{reauthorized-claim}}.  An issuer that does neither, or whose narrowing leaves nothing permitted, MUST reject the request under {{error-handling}}.

## Reissuance and Refresh Without a New Hop {#reissuance-and-refresh}

Reissuance without a new actor hop creates no receipt, so recorded bounds cannot change through the receipt chain.  This document does not define recording re-authorization between hops; {{extensibility}} lets another specification define it.  An issuer that reissues or refreshes while carrying a bounds-bearing receipt chain forward:

*  MUST NOT issue an outer token whose value for any monotonic dimension exceeds `receipt[0]`'s recorded bound; narrowing further is always permitted;
*  when the issued value would exceed the recorded bound for a monotonic dimension, even because broader authority was authorized without a new hop (for example, a refresh grant following step-up or an approver widening a governing authority object), MUST narrow the issued value to fit, fail the request, or, where local policy and resource requirements permit absent receipt coverage, drop the inherited `actor_receipts` array and with it the bounds evidence.

## Domain Transitions {#domain-transitions}

A domain boundary can change the authority vocabulary, making syntactic comparison unsuitable.  This profile records an explicit reset rather than assuming equivalence.

An issuer that adds a hop whose authority vocabulary differs from the inbound token's:

*  MUST record `reauthorized` on the new receipt with `method: domain_transition` and with `dimensions` listing every governed dimension, establishing a new monotonicity basis at the boundary;
*  MUST record the new domain's authority values in the new receipt's `bounds`;
*  SHOULD reference, via `reauthorized.artifact`, the policy or agreement under which the cross-domain translation is authorized.

Comparison applies within each domain segment.  A recipient requiring end-to-end monotonicity MUST reject chains containing `domain_transition` bases unless explicit trusted mappings establish cross-domain equivalence.

## Partial and Sparse Coverage

A partial receipt chain can record bounds on any subset of its receipts.  Such sparse recording supports audit, but consumer processing verifies a dimension across the chain only when every receipt records it.  Deployments needing origin-hop evidence should enable recording at the origin issuer first.

# Consumer Processing {#consumer-processing}

An issuer, resource server, or other recipient relying on this profile MUST perform the following steps:

1.  Validate the outer token and receipt chain under {{I-D.mcguinness-oauth-actor-receipts}}.  Bounds in receipts that fail that validation MUST NOT be used.

2.  Check claim types:
    *  `bounds` is an object whose recognized members have the types defined in {{governed-dimensions}}.
    *  `reauthorized` contains its required, correctly typed members, and the receipt carries `bounds` for each dimension named in `reauthorized.dimensions`.

3.  Compare adjacent receipts for each monotonic dimension D (`aud` and `resource` are monotonic only when required):
    *  Apply this step to D only when every receipt in the chain records `bounds[D]`; a dimension recorded on only some receipts is recorded for audit but not verified.
    *  Skip comparison for D when the newer receipt carries `reauthorized` listing D in `dimensions`, establishing a new basis for D.
    *  Otherwise, the newer receipt's `bounds[D]` must be within the older receipt's `bounds[D]` under {{governed-dimensions}}.  Failure MUST reject bounds evidence.

4.  Compare the current token with `receipt[0]` for each monotonic D that `receipt[0]` carries in `bounds` and whose effective token value is available from claims, introspection, or trusted context.  The token's value MUST be within `receipt[0].bounds[D]`.  Skip `resource` comparison when its effective value cannot be determined.

5.  Enforce dimensions required by `authority_bounds_required` or local policy.  Each required D must be recorded on every receipt and pass steps 3 and 4; a step-4 comparison that cannot be made because the token's effective value for D cannot be determined fails D.  Sparse coverage does not satisfy this requirement.  Full-chain enforcement also needs complete receipt coverage ({{protected-resource-metadata}}).

6.  Apply any additional rules defined by companion profiles whose claims appear in the artifacts ({{extensibility}}).  They can add rejection conditions but cannot relax any requirement needed for conformance to this profile, other than through a replacement bound ({{extensibility}}).

If any required check fails, the recipient MUST reject the token's bounds-based evidence and MUST apply the underlying protocol's error handling for the stage at which the failure occurred.  Rejection of bounds-based evidence does not by itself invalidate the receipt chain under {{I-D.mcguinness-oauth-actor-receipts}}; whether the token remains acceptable without bounds evidence is local policy, except where step 5 applies.

## Composition with Actor-Signed Hop Proofs {#composition-with-proofs}

When the token also carries `actor_proofs` validated under {{I-D.mcguinness-oauth-actor-proofs}}, recorded bounds and actor-consented target bindings are comparable at each hop covered by both artifacts.  For each index i covered by a bounds-bearing receipt and a proof:

*  when `receipt[i].bounds.aud` is present, it MUST be within `actor_proofs[i].target.aud`;
*  when both `receipt[i].bounds.resource` and `actor_proofs[i].target.resource` are present, the recorded set MUST be within the consented set;
*  when both `receipt[i].bounds.scope` and `actor_proofs[i].target.scope` (defined below) are present, the recorded scope set MUST be within the consented scope set.

A failed comparison means the issuer recorded authority broader than the actor consented to at that hop; recipients validating both companions MUST treat it as a failed required check for both artifacts' evidence.

An issuer that supports this profile and accepts a proof carrying `target.scope` MUST NOT embed that proof in a token whose scope exceeds `target.scope`.  A recipient that supports this profile and relies on the proof chain MUST verify, independently of receipt coverage, that the current token's effective scope is within `actor_proofs[0].target.scope` when that member is present, under {{scope-dimension}}; a token whose scope exceeds it has diverged from the proof chain, and the target-binding strict mode of {{I-D.mcguinness-oauth-actor-proofs}} decides whether the recipient rejects the chain.  A recipient that cannot determine the token's effective scope MUST NOT infer scope-level consent from the proof.

This document defines one extension member for the proof `target` object, under the constraining-extension rule of {{I-D.mcguinness-oauth-actor-proofs}}:

`target.scope`:
: OPTIONAL.  A string of space-separated scope tokens the actor authorizes for the token issued at its hop, compared under the rules of {{scope-dimension}}.  As a constraining member, its presence narrows the actor's target binding; consumers that do not recognize it ignore it per {{I-D.mcguinness-oauth-actor-proofs}}.

## Use by Resource Servers

Bounds evidence records non-expansion across covered hops, with explicit re-authorization for expansion.  An RS MUST still evaluate the current token under current policy; verified history alone does not authorize access.

## Introspection {#consumer-introspection}

Receipt-attested bounds travel inside receipts and are returned wherever receipts are returned; the introspection rules of {{I-D.mcguinness-oauth-actor-receipts}} apply unchanged, including all-or-nothing receipt disclosure and the requirement list for outer-token members.

# Discovery and Capability Signaling {#discovery-capability-signaling}

This section defines metadata for advertising authority-bounds support.  It follows the discovery conventions of {{I-D.mcguinness-oauth-actor-receipts}}, with dimension-valued parameters where a boolean would hide which dimensions are covered.

## Authorization Server Metadata

The following parameters are defined for use in Authorization Server Metadata {{RFC8414}}:

`authority_bounds_supported`:
: OPTIONAL.  A non-empty array of governed-dimension names.  The authorization server advertises that, for each named dimension, it can record receipt-attested bounds and enforce issuance-time monotonicity per {{recording-bounds}}.  Absence, or absence of a dimension from the array, means no such advertisement; omission makes no claim of support.

This parameter applies equally to a Transaction Token Service publishing metadata through the same framework.

## Protected Resource Metadata

The following parameters are defined for use in Protected Resource Metadata {{RFC9728}}:

`authority_bounds_required`:
: OPTIONAL.  A non-empty array of governed-dimension names.  For each named dimension, the resource server requires the dense receipt-attested enforcement of consumer step 5: `bounds` for the dimension on every receipt and successful verification.  Naming `aud` or `resource` makes that dimension monotonic ({{audience-governance}}, {{resource-dimension}}).  This is a deployment policy declaration, satisfied by configuring the authorization servers that serve the resource; clients MAY combine it with `authority_bounds_supported` to select an AS.

A resource server that needs full-chain rather than covered-prefix enforcement SHOULD pair `authority_bounds_required` with the receipts companion's `actor_receipts_complete_required`.  That completeness signal does not attest that the visible `act` chain is itself unfiltered; that separate assurance is `chain_complete` ({{I-D.mcguinness-oauth-actor-profile}}).

# Error Handling {#error-handling}

Bounds validation extends the underlying OAuth or Transaction Token validation.  Failures are reported through the error mechanism applicable to the stage at which they occur.

When an authorization server or Transaction Token Service rejects a token request because inbound bounds evidence fails validation under {{consumer-processing}} (for example, a monotonicity failure in the inbound chain), it returns an error response per {{RFC6749, Section 5.2}}: `invalid_request` for a Token Exchange request, as {{RFC8693, Section 2.2.2}} requires, or `invalid_grant` for a JWT bearer grant request ({{RFC7523, Section 3.1}}), consistent with the core actor profile's error mapping for actor information that fails validation.

When requested authority exceeds the recorded bound without re-authorization and the issuer does not narrow the issued value to fit, or narrowing leaves nothing permitted ({{recording-bounds}}), the issuer SHOULD return:

| Dimension | Error |
|-----------|-------|
| `scope` | `invalid_scope` ({{RFC6749, Section 5.2}}) |
| `aud` or `resource` | `invalid_target` ({{RFC8693, Section 2.2.2}} for Token Exchange; {{RFC8707, Section 2}} otherwise) |
| `authorization_details` | `invalid_authorization_details` {{RFC9396}} |

The issuer uses `actor_unauthorized` as defined in the core actor profile {{I-D.mcguinness-oauth-actor-profile}} when the failure reflects an actor-authorization decision.  An absent required artifact is an input-validation failure: `invalid_request` on a Token Exchange request, `invalid_grant` on a JWT bearer grant or refresh request.

When a resource server rejects a request because bounds verification fails or required dimensions are unsatisfied, it SHOULD return `invalid_token` per {{RFC6750}} Section 3.1, and SHOULD include an `error_description` identifying bounds-verification failure so operators can distinguish it from generic token validation.  For a Transaction Token, the resource server rejects it through the deployment's Txn-Token handling, because {{I-D.ietf-oauth-transaction-tokens}} defines no error response.

An introspection server does not return an OAuth error for missing bounds artifacts; their presence is a property of the response.  This document defines no new OAuth error codes.

# Extensibility {#extensibility}

This profile composes with the extensibility framework of {{I-D.mcguinness-oauth-actor-receipts}} and adds its own surfaces:

*  **New governed dimensions**, registered in the dimension registry ({{iana-dimensions}}) with a defined comparison rule and a declared governance class (monotonic by default, or record-only).  Consumers ignore unregistered `bounds` members they do not recognize.
*  **New re-authorization methods**, registered in the methods registry ({{iana-methods}}) or expressed as collision-resistant URIs.
*  **Per-type RAR refinement rules**, defined by the specifications that define RAR types; such rules extend {{rar-dimension}} for their types without modifying this document.
*  **Re-authorization without a new hop.**  This profile defines no mechanism for recording re-authorization without a new actor hop.  Another specification can define one, including how a validated re-authorization establishes a replacement bound for the dimensions it covers, and the evidence, trust, ordering, expiry, preservation, and discovery rules it needs.  An issuer or recipient that supports such a mechanism uses the replacement bound in place of the affected receipt's `bounds[D]` wherever this document compares against it, for the covered dimensions only.  One that does not support it MUST apply the recorded bounds defined here.

Companion rules MUST NOT relax any requirement needed for conformance to this profile, other than through a replacement bound as described above; they MAY add rejection conditions.  A companion's partial-validation mode, defined under its own normative scope as {{I-D.mcguinness-oauth-actor-receipts}} requires, is not conformance to this profile.

# Security Considerations

Authority bounds strengthen authority provenance for receipt-covered hops, but they do not replace token validation or authorization.  The general OAuth 2.0 Security Best Current Practice {{RFC9700}} and the JWT best practices in {{RFC8725}} apply.

## Threat Model {#threat-model}

### Adversaries Mitigated by This Profile

*  **Intermediate authority expansion.**  Detection requires every receipt in the chain to record the dimension.  Recording a wider value fails comparison; recording less than issued fails the next enforcing issuer's check or the terminal token comparison; omission fails dense-coverage enforcement.  Sparse coverage leaves gaps in this protection.
*  **Silent basis change.**  Expansion requires a signed artifact: a `reauthorized` claim inside a trusted issuer's receipt, scoped to the dimensions it lists.
*  **Issuance beyond actor consent, when proofs are present.**  The cross-checks of {{composition-with-proofs}} detect recorded authority broader than the actor-signed target binding at the same hop.

### Adversaries Not Mitigated

*  **Origin issuer choosing broad initial bounds.**  Monotonicity is relative; no upstream value constrains the origin.  Constraint on origin authority requires policy at the origin issuer, pre-authorization artifacts, or transparency mechanisms outside this document.
*  **Fabricated re-authorization by a trusted issuer.**  Any issuer trusted to record re-authorization can convert detected expansion into authorized expansion ({{reauthorization-abuse}}).
*  **Full-chain collusion.**  Colluding issuers fabricate a monotonic chain at any level; this matches the receipts companion's trust boundary.
*  **Semantic expansion within syntactic subsets.**  See {{scope-subsumption-gaps}}.
*  **Cross-domain expansion.**  A `domain_transition` basis reset is exactly an unverified re-expression of authority; recipients requiring end-to-end guarantees must reject or map it ({{domain-transitions}}).
*  **Compromised current outer-token issuer.**  Out of scope here as in the receipts companion; a compromised outer issuer can omit this profile's claims entirely.  Absence of bounds evidence is a downgrade recipients detect only by requiring the evidence ({{discovery-capability-signaling}}).

### Trust Model Summary

Bounds inherit the receipts companion's per-issuer, non-transitive trust model, and add one axis: trust to record re-authorization.  A recipient can trust an issuer's receipts while refusing its `reauthorized` claims ({{reauthorization-abuse}}).  Composition with proofs adds an actor-side check with an independent trust anchor.

## Re-Authorization Abuse {#reauthorization-abuse}

A compromised or over-trusted issuer that is trusted to record re-authorization can use it to justify arbitrary expansion.

*  Recipients MUST evaluate re-authorization trust separately from receipt trust: which issuers are trusted to capture re-authorization, for which subjects, and by which methods, is explicit local policy.  A recipient MAY accept an issuer's receipts while rejecting its re-authorization records; a rejected re-authorization is a failed basis reset, and the chain is then evaluated without it, which typically fails monotonicity and rejects the token's bounds evidence.
*  Deployments needing strong re-authorization integrity SHOULD require `reauthorized.artifact` and SHOULD validate the referenced artifact against the authority that captured the event (for example, verifying a consent receipt's signature), rather than accepting the recording issuer's bare assertion.
*  `domain_transition` bases deserve the most scrutiny: they legitimize non-comparability, and an attacker who can insert one launders any expansion.  Recipients SHOULD restrict which issuers may record domain transitions to the deployment's known boundary issuers.

## Scope Subsumption Gaps {#scope-subsumption-gaps}

Set-membership comparison catches verbatim expansion only.  A scope token that is lexically new at a hop fails the subset check even when semantically narrower, and a lexically preserved token can be semantically broadened by configuration changes at the AS that defines it.  Deployments whose scope grammars carry hierarchy or wildcard semantics MUST either emit explicit narrowest-form scopes at every hop, or define and apply a deployment-specific comparison rule; this document does not define scope subsumption, and a general solution belongs in its own specification.

## Recorded Values and Token Reality

`bounds` members are attested copies of issued-token values, signed by the issuer that produced both.  An issuer that records values differing from what it actually issued produces either a detectable mismatch (the outer-token comparison at the terminal hop, or the next enforcing issuer's inbound check) or a consistent lie spanning its receipt and its token, which is the intermediate authority expansion case in {{threat-model}}.  Step 4 of {{consumer-processing}} compares `bounds` against the effective values the token actually carries, not request-time values.

# Privacy Considerations {#privacy-considerations}

The privacy considerations of {{I-D.mcguinness-oauth-actor-receipts}} apply, including cross-service correlation and retention beyond token lifetime.  Bounds add authority-shaped disclosure:

*  `bounds` exposes per-hop scope, audience, resource, and authorization-detail values to every recipient of the token or introspection response, revealing internal permission vocabulary, resource topology, and orchestration structure.  Issuers SHOULD record only the dimensions recipients need, and MAY enforce monotonicity at issuance without recording bounds where disclosure outweighs evidence value.
*  `reauthorized` reveals consent prompts, step-up authentication, and policy decisions, with timing; this is sensitive activity metadata.
*  A fully bounds-covered chain is a detailed narrative of authority narrowing across an organization; deployments SHOULD scope disclosure to audiences with adequate agreements, using the receipts companion's all-or-nothing granularity deliberately.

# IANA Considerations

## JSON Web Token Claims Registration {#iana-jwt-claims}

This document requests registration of the following JWT Claims in the "JSON Web Token Claims" registry {{RFC7519}}:

*  Claim Name: `bounds`
*  Claim Description: Authority bounds in effect for the token issued at the hop attested by an Actor Receipt JWT
*  Change Controller: IETF
*  Specification Document(s): This document

*  Claim Name: `reauthorized`
*  Claim Description: Record of explicit re-authorization of delegated authority at a hop
*  Change Controller: IETF
*  Specification Document(s): This document

This document does not request separate registration for the members of the `bounds`, `reauthorized`, and proof `target` objects it defines; sub-object keys within a registered claim are scoped to that claim's JSON object, following the convention of {{I-D.mcguinness-oauth-actor-profile}}.

## OAuth Actor Authority Bounds Dimensions Registry {#iana-dimensions}

This document requests that IANA establish a registry titled "OAuth Actor Authority Bounds Dimensions", with the registration policy Specification Required.  Each entry records: Dimension Name, Governance Class (`monotonic` or `record-only`), Comparison Rule reference, and Specification Document(s).  Initial contents:

*  `scope`, monotonic, {{scope-dimension}} of this document
*  `aud`, record-only, {{audience-governance}} of this document
*  `resource`, record-only, {{resource-dimension}} of this document
*  `authorization_details`, monotonic, {{rar-dimension}} of this document

Designated experts SHOULD verify that a requested dimension has a deterministic comparison rule, a declared governance class, and semantics that do not overlap an existing entry.

## OAuth Actor Re-Authorization Methods Registry {#iana-methods}

This document requests that IANA establish a registry titled "OAuth Actor Re-Authorization Methods", with the registration policy Specification Required.  Each entry records: Method Name, Description, and Specification Document(s).  Initial contents: `interactive_consent`, `step_up`, `policy_grant`, and `domain_transition`, as defined in {{reauthorized-claim}}.  Values not in the registry MUST be collision-resistant URIs.

## OAuth Authorization Server Metadata Registration

This document requests registration of the following metadata names in the "OAuth Authorization Server Metadata" registry {{RFC8414}}:

*  Metadata Name: `authority_bounds_supported`
*  Metadata Description: Array of authority-dimension names for which the server records receipt-attested bounds and enforces issuance-time monotonicity
*  Change Controller: IETF
*  Specification Document(s): This document

## OAuth Protected Resource Metadata Registration

This document requests registration of the following metadata names in the "OAuth Protected Resource Metadata" registry {{RFC9728}}:

*  Metadata Name: `authority_bounds_required`
*  Metadata Description: Array of authority-dimension names for which the resource requires dense receipt-attested bounds enforcement
*  Change Controller: IETF
*  Specification Document(s): This document

# Acknowledgments

The author thanks the OAuth Working Group for the specifications underlying this profile.  The PIC Model {{PIC-MODEL}} also informed the non-expansion property, which this document represents through signed OAuth evidence.

Contributors and reviewers will be acknowledged in future revisions.

--- back

# Examples

The examples in this appendix show decoded contents; real receipts are compact-signed JWT strings.  Timestamps are illustrative.  The scenario continues the two-hop travel example of {{I-D.mcguinness-oauth-actor-receipts}}: Alice delegates to an AI travel-assistant agent through the enterprise AS, and the agent's token is exchanged at the travel-provider AS, which adds a booking tool as the outermost actor.  Because these receipts carry `bounds`, they are different byte strings from the receipts shown in that document's examples and carry their own identifiers.

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
  "actor_receipts_complete": true
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
*  Resources narrow to the bookings endpoint, so the chain also verifies where `resource` is required.  URI prefixes do not imply containment.
*  The outer token's scope equals the newest recorded scope.
*  Audience changes are recorded without comparison under the default audience rules.

# Document History
{:numbered="false"}

[[ To be removed from the final specification ]]

-00

* Initial version.

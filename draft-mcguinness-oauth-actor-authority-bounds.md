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

This document defines OAuth Actor Chain Authority Bounds, an optional companion to the OAuth Actor Profile for Delegation and Actor Receipts.  Its receipt claims record the authority in effect at each hop, so that recipients can detect expansion of `scope` and `authorization_details`, and of `aud` and `resource` where a deployment requires it, unless an explicit re-authorization establishes new bounds.  It also specifies comparison rules and discovery metadata.

--- middle

# Introduction

The OAuth Actor Profile {{I-D.mcguinness-oauth-actor-profile}} identifies delegated actors.  Actor Receipts {{I-D.mcguinness-oauth-actor-receipts}} attest issuer participation, and Actor Proofs {{I-D.mcguinness-oauth-actor-proofs}} attest actor participation and target consent.  OAuth 2.0 Token Exchange {{RFC8693}} does not define a record of authority changes across hops, and these profiles do not add one.  Actor Receipts lists historical authority as a non-goal.  An actor proof binds only the target its actor authorized at one hop.  A recipient can therefore verify who participated at every hop without detecting that an intermediate issuer widened the authority flowing through the chain.

This document defines OAuth Actor Chain Authority Bounds, an optional companion profile that closes that gap for deployments that use actor receipts.  Receipt claims record the authority in effect at each hop; recipients compare those values across hops and against the current token, and expansion requires an explicit re-authorization recorded in the receipt for a new hop.  The design center is:

*  keep the visible actor chain in `act` and per-hop provenance in `actor_receipts`;
*  carry per-hop authority as receipt claims, so it inherits the receipt issuer's signature and the chain's integrity;
*  make authority expansion an explicit, signed, auditable event rather than a silent change.

This profile adds bounds claims and discovery metadata; deployments opt in per resource or trust domain.

Recipients walk the validated receipt chain from oldest to newest and verify offline that each monotonic dimension narrows or stays the same, except where a re-authorization recorded at a hop establishes a new basis for the dimensions it lists, and that the current outer token does not exceed the newest recorded bounds ({{consumer-processing}}).  Verifying a dimension requires every receipt to record it; deployments that need that guarantee enforce dense coverage through {{discovery-capability-signaling}}.

This profile does not interpret scope grammars ({{scope-dimension}}), constrain the origin issuer's initial choice of authority, or define cross-domain equivalence of authority vocabularies.  A change of authority vocabulary, as at a trust-domain boundary, is an explicit basis reset ({{domain-transitions}}), so deployments whose chains change vocabulary at every hop gain recording and audit value but little enforcement value.  Recording re-authorization between hops (a refresh or step-up without a new hop) is left to a later extension ({{extensibility}}).

Attenuating Authorization Tokens {{I-D.niyikiza-oauth-attenuating-agent-tokens}} address a related problem with a different model: a token holder derives a token with equal or narrower tool-level authority offline, and any enforcement point holding the root issuer's trust anchor verifies the derivation chain.  This profile instead adds evidence to issuance by authorization servers and Transaction Token Services, recording the authority each issuer applied in its signed receipt and permitting expansion only under a signed re-authorization.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses OAuth terminology from {{RFC6749}} and {{RFC8693}}.  The terms Actor Receipt, Receipt Chain, and Outer Token are used as defined in {{I-D.mcguinness-oauth-actor-receipts}}.  `receipt[i]` denotes entry i of the outer token's `actor_receipts` array, which that document orders newest first: `receipt[0]` is the receipt for the newest hop, whose actor is the outermost `act`.  The abbreviations AS, RS, and TTS denote authorization server, resource server, and Transaction Token Service, respectively.

The following terms are used in this document:

Governed Dimension:
: An authority dimension whose per-hop values are recorded under this profile and, when the dimension is monotonic, compared across hops.  This document defines four: `scope`, `aud`, `resource`, and `authorization_details`.

Monotonic Dimension:
: A governed dimension whose recorded values are required not to expand across hops.  `scope` and `authorization_details` are monotonic by default; `aud` and `resource` are recorded but not monotonic unless a resource server names them in `authority_bounds_required` or local policy requires it ({{audience-governance}}, {{resource-dimension}}).

Authority Bounds:
: The values of governed dimensions in effect for the token issued at a given hop, recorded in the receipt's `bounds` claim.

Re-Authorization Event:
: An explicit event in which an authorized principal consents to new, possibly broader, authority.  It is recorded in the `reauthorized` claim of the receipt for the new actor hop at which it occurs.

Monotonicity Basis:
: The point in the chain from which non-expansion of a dimension is measured.  The origin hop is the initial basis; each re-authorization recorded at a hop establishes a new basis for the dimensions it lists.

Dense Coverage:
: A condition in which every receipt in the chain carries `bounds` for a given dimension, so that monotonicity for that dimension is verified across every adjacency.

Examples in this document are illustrative and omit unrelated claims, signatures, and validation steps that a complete deployment would need.

# Relationship to the Receipts Companion

This profile uses two extension points in {{I-D.mcguinness-oauth-actor-receipts}}:

*  Receipt extension claims: `bounds` and `reauthorized`, protected by the receipt signature.
*  Cross-receipt verification: comparisons across receipts.

The receipt signing, linkage, byte-preservation, and coverage rules of that document continue to apply.

# Governed Dimensions and Comparison Rules {#governed-dimensions}

Consumer verification ({{consumer-processing}}) applies the comparison rules below.  A comparison is evaluated only when both compared values are present; absence of a recorded bound is missing evidence, not a successful comparison ({{bounds-claim}}).

## `scope` {#scope-dimension}

The `scope` dimension records the space-separated scope string of {{Section 3.3 of RFC6749}}.  Values are compared as follows:

*  parse both values into sets of distinct scope tokens (separator: ASCII space, U+0020);
*  `scope_a` is within `scope_b` if and only if every token in `scope_a` is also in `scope_b`;
*  the empty string is the empty set and is within every scope set.

Comparison does not interpret scope semantics: `read:user/*` is not treated as covering `read:user/123` ({{scope-subsumption-gaps}}).

## `aud` and Audience Governance {#audience-governance}

The `aud` dimension records the audience of the token issued at the hop, as a string or array of strings with the value space of {{Section 4.1.3 of RFC7519}}.  Comparison treats a single string as a one-element set; `aud_a` is within `aud_b` if and only if every value in `aud_a` is in `aud_b`.

By default, audience is recorded without monotonicity enforcement because Token Exchange commonly retargets tokens.  Recorded audiences remain useful for audit and comparison with actor-consented targets ({{composition-with-proofs}}).

A deployment whose chains do not retarget, or that treats retargeting as a policy-controlled event, MAY require audience governance through the metadata in {{discovery-capability-signaling}} or local policy.  It then applies the same monotonicity rules to `aud`; retargeting MUST be covered by re-authorization or verification fails.

## `resource` {#resource-dimension}

The `resource` dimension records the effective set of resource indicators that the issuer applied, as an array of absolute URIs with {{RFC8707}} semantics.  An issuer that applied no resource indicator omits `bounds.resource` rather than recording an empty array; where a resource server requires `resource` ({{protected-resource-metadata}}), the omission fails that dimension.

Values are compared as follows:

*  compare URIs by simple string comparison ({{Section 6.2.1 of RFC3986}}); issuers need to record each resource indicator in the same form at every hop;
*  `resource_a` is within `resource_b` if and only if every URI in `resource_a` is also in `resource_b`.

URI prefix subsumption (for example, treating `https://api.travel-provider.example/v1/` as covering `https://api.travel-provider.example/v1/users`) is not applied.  Issuers wishing to express prefix relationships MUST emit explicit URIs at each hop.

By default, `resource` is recorded without monotonicity enforcement because Token Exchange uses the `resource` parameter to retarget tokens ({{Section 2.1 of RFC8693}}).  A deployment MAY require monotonicity for `resource` through the same metadata or local policy as audience governance ({{audience-governance}}).  It then applies the same monotonicity rules to `resource`; retargeting MUST be covered by re-authorization or verification fails.

## `authorization_details` {#rar-dimension}

The `authorization_details` dimension records the Rich Authorization Requests array of {{RFC9396}}.  Refinement is evaluated per object and per type:

*  an array `ad_a` refines `ad_b` if and only if every object in `ad_a` refines some object in `ad_b`; objects present in `ad_b` but absent from `ad_a` represent narrowing and are permitted; objects in `ad_a` that refine no object in `ad_b` represent expansion and fail the comparison;
*  an object refines another object only when both have the same `type`;
*  for the common members defined by {{Section 2.2 of RFC9396}}, refinement requires: `actions` a subset, `locations` a subset under the URI rules of {{resource-dimension}}, `datatypes` a subset, `privileges` a subset, and `identifier` equal; a common member present in the `ad_b` object but absent from the `ad_a` object is expansion and fails the comparison, and one absent from the `ad_b` object but present in the `ad_a` object also fails the comparison unless the refinement rules for that `type` establish that it preserves or narrows authority, because the effect of a member depends on the API's semantics ({{Section 6.1 of RFC9396}}).

The common-member rules above apply to objects of a `type` whose members the recipient can evaluate, either because it knows that the `type` defines no other members or because it has a refinement rule for them.  For any other `type`, an object refines another object of the same `type` only when the two are equal as whole JSON objects (the same member names with equal values; member order is insignificant), and any change requires a type-specific refinement rule.  RAR type specifications SHOULD define their own refinement rules; see {{extensibility}}.

## Dimensions Explicitly Not Governed

This profile does not govern token lifetime (`exp`), presenter binding (`cnf`), subject identity (`sub`), or entity classification (`sub_profile`).  The base profiles define those rules.

# Receipt Extension Claims

This section defines two extension claims for Actor Receipt JWTs, under the extension-claims rule of {{I-D.mcguinness-oauth-actor-receipts}}.  Both inherit the receipt's signature and byte-preservation protections.

## The `bounds` Claim {#bounds-claim}

`bounds`:
: OPTIONAL.  A JSON object recording the authority bounds in effect for the token issued at this hop.  Members correspond to the governed dimensions:

  *  `scope`: a string of space-separated scope tokens;
  *  `aud`: a string or array of strings;
  *  `resource`: an array of URI strings;
  *  `authorization_details`: an array of {{RFC9396}} objects.

  Each member, when present, MUST equal the corresponding effective value applied to the token issued at this hop: for `scope`, `aud`, and `authorization_details`, ordinarily the issued token's top-level claim of the same name; for `resource`, the resource-indicator set the issuer applied whether or not the token carries a claim.

  An absent member is missing evidence, not an assertion that authority was unconstrained or unchanged.  Recipients MUST treat it accordingly.  Additional dimensions MAY be registered under {{iana-dimensions}}; consumers MUST ignore unrecognized members unless a specification or local agreement supplies their comparison rule.

A receipt MAY omit `bounds` entirely, and a chain MAY mix receipts with and without it.

## The `reauthorized` Claim {#reauthorized-claim}

`reauthorized`:
: OPTIONAL.  A JSON object recording that the authority issued at this hop was expanded relative to the inbound authority under an explicit re-authorization.  Members:

  `sub`:
  : REQUIRED.  Identifier of the principal or authority that re-authorized the delegation, such as the subject, an approver, or the authority whose policy or agreement applies.  When the subject itself re-authorized, it equals the top-level `sub` of the token issued at this hop.  Recipients evaluate whether to trust the re-authorization under {{reauthorization-abuse}}.

  `iss`:
  : REQUIRED.  Identifier of the authorization server or other authority that captured the re-authorization.

  `method`:
  : REQUIRED.  A value from the re-authorization methods registry established in {{iana-methods}}, or a collision-resistant URI.  The initial registered values are:

    *  `interactive_consent`: re-authorization by the principal through an interactive prompt;
    *  `step_up`: re-authorization through step-up authentication;
    *  `policy_grant`: programmatic re-authorization under a deployment policy artifact;
    *  `domain_transition`: the hop changes the authority vocabulary of one or more dimensions, as at a trust-domain boundary or when a TTS issues Transaction Token scope ({{domain-transitions}}).

  `iat`:
  : REQUIRED.  Time of the re-authorization.

  `dimensions`:
  : REQUIRED.  A non-empty array of the governed dimension names this re-authorization resets.  The receipt MUST carry `bounds` for each listed dimension.

  `artifact`:
  : OPTIONAL.  A URI or token identifier referencing an external artifact that evidences the re-authorization (for example, a consent record or step-up assertion).  Recipients MAY resolve and validate the artifact under local policy; {{reauthorization-abuse}} covers when deployments require it.

Recording `reauthorized` resets comparison under this profile only; it does not let an issuer exceed a limit that the underlying grant or the core actor profile imposes, such as the Token Exchange scope ceiling.  When `reauthorized` is present on a receipt, that hop is a new monotonicity basis only for the dimensions listed in `reauthorized.dimensions`: the hop's bounds for a listed dimension are not compared against older bounds, and newer artifacts are compared against the post-re-authorization bounds ({{consumer-processing}}).  A receipt carrying `reauthorized` SHOULD carry `bounds` for every governed dimension in effect at the hop.

# Issuer Processing

This section extends the issuer processing of {{I-D.mcguinness-oauth-actor-receipts}}, whose receipt creation, extension, preservation, and reissuance rules apply unchanged.

## Recording Bounds at a New Hop {#recording-bounds}

When an issuer adds a new outermost actor hop and creates the receipt for it, and the deployment uses this profile, the issuer:

1.  MUST determine the issued token's effective `scope`, `aud`, `resource`, and `authorization_details` under the underlying grant rules.
2.  MUST include in the new receipt's `bounds` each dimension it attests, with each member equal to the effective issued value per {{bounds-claim}}.
3.  For each monotonic dimension it enforces, MUST verify that the issued value is within the inbound token's effective value for that dimension, and, when the inbound token's validated `receipt[0]` carries bounds for the dimension, within that receipt's recorded bound.
4.  When the requested authority would fail step 3, MAY narrow the issued `scope` to fit ({{Section 3.3 of RFC6749}}; a Token Exchange response then reports the issued scope, {{Section 2.2.1 of RFC8693}}), and, on a request other than Token Exchange, the issued `resource` value ({{Section 2.2 of RFC8707}}); it does not drop a requested audience, or a requested resource on a Token Exchange request, and steps 1 to 3 then apply to the narrowed value.  When the issuer has a re-authorization for the expansion, captured under one of the methods of {{reauthorized-claim}} by the issuer itself or by an authority it trusts, it MAY instead proceed by recording `reauthorized` on the new receipt, listing each expanded dimension in `reauthorized.dimensions`, provided the underlying grant rules and the core actor profile permit the expanded value.  An issuer that does neither, or whose narrowing leaves nothing permitted, MUST reject the request under {{error-handling}}.

## Reissuance and Refresh Without a New Hop {#reissuance-and-refresh}

Reissuance without a new actor hop creates no receipt, so recorded bounds cannot change through the receipt chain.  An issuer that reissues or refreshes while carrying a bounds-bearing receipt chain forward:

*  MUST NOT issue an outer token whose value for any monotonic dimension exceeds `receipt[0]`'s recorded bound; narrowing further is always permitted;
*  when the issued value would exceed the recorded bound for a monotonic dimension, even because broader authority was authorized without a new hop (for example, a refresh grant following step-up or an approver widening the underlying grant), MUST narrow the issued value to fit, fail the request, or, where local policy and resource requirements permit absent receipt coverage, drop the inherited `actor_receipts` array and with it the bounds evidence.

## Domain Transitions {#domain-transitions}

A trust-domain boundary, or a TTS issuing Transaction Token scope ({{Section 9.2 of I-D.ietf-oauth-transaction-tokens}}), can change the authority vocabulary of a dimension, making syntactic comparison unsuitable.

An issuer that adds a hop at which the authority vocabulary of a dimension the new receipt records in `bounds` differs from the inbound token's:

*  MUST record `reauthorized` on the new receipt with `method: domain_transition` and with `dimensions` listing each such dimension, establishing a new monotonicity basis for those dimensions;
*  MUST record the new authority values for those dimensions in the new receipt's `bounds`;
*  SHOULD reference, via `reauthorized.artifact`, the policy or agreement under which the translation is authorized.

Comparison applies within each segment between vocabulary changes.  A recipient requiring end-to-end monotonicity MUST reject chains containing `domain_transition` bases unless explicit trusted mappings establish equivalence across the change.

A TTS that changes the authority vocabulary in presenter continuation adds no hop and therefore cannot record a new basis.  For a monotonic dimension whose vocabulary changes, it fails the request or, where local policy and resource requirements permit absent receipt coverage, drops the inherited `actor_receipts` array ({{reissuance-and-refresh}}).

## Partial and Sparse Coverage

A partial receipt chain can carry bounds on any subset of its receipts.  Deployments needing origin-hop evidence should enable recording at the origin issuer first.

# Consumer Processing {#consumer-processing}

An issuer, resource server, or other recipient relying on this profile MUST perform the following steps:

1.  Validate the outer token and receipt chain under steps 1 to 10 of the consumer processing of {{I-D.mcguinness-oauth-actor-receipts}} and, when the token carries `actor_proofs`, the consumer processing of {{I-D.mcguinness-oauth-actor-proofs}} through its sibling-reference check.  Bounds in receipts that fail that validation MUST NOT be used.

2.  Check claim types:
    *  `bounds` is an object whose recognized members have the types defined in {{governed-dimensions}}.
    *  `reauthorized` contains its required, correctly typed members, and the receipt carries `bounds` for each dimension named in `reauthorized.dimensions`.

3.  Compare adjacent receipts for each monotonic dimension D (`aud` and `resource` are monotonic only when required):
    *  Apply this step to D only when every receipt in the chain records `bounds[D]`; a dimension recorded on only some receipts is recorded for audit but not verified.
    *  Skip comparison for D when the newer receipt carries `reauthorized` listing D in `dimensions`, establishing a new basis for D.
    *  Otherwise, the newer receipt's `bounds[D]` must be within the older receipt's `bounds[D]` under {{governed-dimensions}}.  Failure MUST reject bounds evidence.

4.  Compare the current token with `receipt[0]` for each monotonic D that `receipt[0]` carries in `bounds` and whose effective token value is available from claims, introspection, or trusted context.  The token's value MUST be within `receipt[0].bounds[D]`.  An effective `resource` value available from claims, introspection, or trusted context takes precedence.  Only when none is available, a resource server MAY use the resource identifier of the protected resource receiving the request; that comparison shows only that the receiving resource is within the recorded bound, not that the token's whole resource set is.  When neither is available, skip `resource` comparison.

5.  Enforce dimensions required by `authority_bounds_required` or local policy.  Each required D must be recorded on every receipt and pass steps 3 and 4; a step-4 comparison that cannot be made because the token's effective value for D cannot be determined fails D.  Sparse coverage does not satisfy this requirement.  Full-chain enforcement also needs complete receipt coverage ({{protected-resource-metadata}}).

6.  Apply any additional rules defined by companion profiles whose claims appear in the artifacts ({{extensibility}}).

If any required check fails, the recipient MUST reject the token's bounds-based evidence.  It rejects the token only when step 5 or local policy requires bounds evidence, using the underlying protocol's error handling for the stage at which the failure occurred.  Rejection of bounds-based evidence does not by itself invalidate the receipt chain under {{I-D.mcguinness-oauth-actor-receipts}}; whether the token remains acceptable without bounds evidence is local policy, except where step 5 applies.

## Composition with Actor-Signed Hop Proofs {#composition-with-proofs}

When the token also carries `actor_proofs` validated under {{I-D.mcguinness-oauth-actor-proofs}}, recorded bounds and actor-consented target bindings are comparable at each hop covered by both artifacts.  For each index i covered by a bounds-bearing receipt and a proof:

*  when `receipt[i].bounds.aud` is present, it MUST be within `actor_proofs[i].target.aud`;
*  when both `receipt[i].bounds.resource` and `actor_proofs[i].target.resource` are present, the recorded set MUST be within the consented set;
*  when both `receipt[i].bounds.scope` and `actor_proofs[i].target.scope` (defined below) are present, the recorded scope set MUST be within the consented scope set.

A failed comparison means the issuer recorded authority broader than the actor consented to at that hop; recipients validating the receipts and proofs companions MUST reject both the bounds-based evidence and the proof-based evidence for the token; the receipt chain's hop provenance is unaffected.

An issuer that supports this profile and accepts a proof carrying `target.scope` MUST NOT embed that proof in a token whose scope exceeds `target.scope`.  A recipient that supports this profile and relies on the proof chain MUST verify, independently of receipt coverage, that the current token's effective scope is within `actor_proofs[0].target.scope` when that member is present, under {{scope-dimension}}; a token whose scope exceeds it has diverged from the proof chain, and the target-binding strict mode of {{I-D.mcguinness-oauth-actor-proofs}} decides whether the recipient rejects the chain.  A recipient that cannot determine the token's effective scope MUST NOT infer scope-level consent from the proof.

This document defines one extension member for the proof `target` object, under the constraining-extension rule of {{I-D.mcguinness-oauth-actor-proofs}}:

`target.scope`:
: OPTIONAL.  A string of space-separated scope tokens the actor authorizes for the token issued at its hop, compared under the rules of {{scope-dimension}}.  As a constraining member, its presence narrows the actor's target binding; consumers that do not recognize it ignore it per {{I-D.mcguinness-oauth-actor-proofs}}.

## Use by Resource Servers

An RS MUST still evaluate the current token under current policy; verified history alone does not authorize access.  Bounds verification compares values recorded in validated receipts, which the outer token carries as a top-level claim, with the current token's own claims and, when present, validated proofs; it does not make nested `act` objects inputs to access-control decisions ({{Section 4.1 of RFC8693}}).

## Introspection {#consumer-introspection}

Bounds travel inside receipts, so the introspection rules of {{I-D.mcguinness-oauth-actor-receipts}} apply unchanged.  An introspection response whose `receipt[0]` carries `bounds` MUST also include the token's `scope`, `aud`, and `authorization_details` ({{Section 9.2 of RFC9396}}) members for each of those dimensions that `receipt[0].bounds` records, so that step 4 of {{consumer-processing}} can be applied.

# Discovery and Capability Signaling {#discovery-capability-signaling}

This metadata follows the discovery conventions of {{I-D.mcguinness-oauth-actor-receipts}}, using dimension-valued parameters where a boolean would not convey which dimensions are covered.

## Authorization Server Metadata

The following parameters are defined for use in Authorization Server Metadata {{RFC8414}}:

`authority_bounds_supported`:
: OPTIONAL.  A non-empty array of governed-dimension names.  The authorization server advertises that, for each named dimension, it can record receipt-attested bounds and enforce issuance-time monotonicity per {{recording-bounds}}.  Absence of the parameter, or of a dimension from the array, makes no claim of support.

This parameter applies equally to a Transaction Token Service that publishes metadata through the same framework.

## Protected Resource Metadata

The following parameters are defined for use in Protected Resource Metadata {{RFC9728}}:

`authority_bounds_required`:
: OPTIONAL.  A non-empty array of governed-dimension names.  For each named dimension, the resource server requires the dense receipt-attested enforcement of consumer step 5: `bounds` for the dimension on every receipt and successful verification.  Naming `aud` or `resource` makes that dimension monotonic ({{audience-governance}}, {{resource-dimension}}).  This is a deployment policy declaration, satisfied by configuring the authorization servers that serve the resource; clients MAY combine it with `authority_bounds_supported` to select an AS.

A resource server that needs full-chain rather than covered-prefix enforcement SHOULD pair `authority_bounds_required` with the receipts companion's `actor_receipts_complete_required`.  That completeness signal does not attest that the visible `act` chain is itself unfiltered; that separate assurance is the `chain_complete` introspection response member ({{I-D.mcguinness-oauth-actor-profile}}).

# Error Handling {#error-handling}

When an authorization server or Transaction Token Service rejects a token request because inbound bounds evidence fails validation under {{consumer-processing}} (for example, a monotonicity failure in the inbound chain), it returns an error response per {{Section 5.2 of RFC6749}}: `invalid_request` for a Token Exchange request, as {{Section 2.2.2 of RFC8693}} requires, or `invalid_grant` for a JWT bearer grant request ({{Section 3.1 of RFC7523}}).

When requested authority exceeds the recorded bound without re-authorization and the issuer does not narrow the issued value to fit, or narrowing leaves nothing permitted ({{recording-bounds}}), the issuer SHOULD return:

| Dimension | Error |
|-----------|-------|
| `scope` | `invalid_scope` ({{Section 5.2 of RFC6749}}) |
| `aud` or `resource` | `invalid_target` ({{Section 2.2.2 of RFC8693}} for Token Exchange; {{Section 2 of RFC8707}} otherwise) |
| `authorization_details` | `invalid_authorization_details` {{RFC9396}} |

When the failure reflects an actor-authorization decision, the issuer uses the `actor_unauthorized` error code defined in the core actor profile {{I-D.mcguinness-oauth-actor-profile}}.  An absent required artifact is an input-validation failure: the `invalid_request` error code for a Token Exchange request and the `invalid_grant` error code for a JWT bearer grant or refresh request.

When a resource server rejects a request because bounds verification fails or required dimensions are unsatisfied, it SHOULD return `invalid_token` ({{Section 3.1 of RFC6750}}) in a challenge that uses the authentication scheme the core actor profile's resource server processing selects, and SHOULD include an `error_description` identifying bounds-verification failure so operators can distinguish it from generic token validation.  For a Transaction Token, the resource server rejects it through the deployment's Transaction Token handling, because {{I-D.ietf-oauth-transaction-tokens}} defines no error response.

An introspection server does not return an OAuth error for missing bounds artifacts; their presence is a property of the response.  This document defines no new OAuth error codes.

# Extensibility {#extensibility}

This profile composes with the extensibility framework of {{I-D.mcguinness-oauth-actor-receipts}} and adds its own surfaces:

*  **New governed dimensions**, registered in the dimension registry ({{iana-dimensions}}) with a defined comparison rule and a declared governance class (monotonic by default, or record-only).
*  **New re-authorization methods**, registered in the methods registry ({{iana-methods}}) or expressed as collision-resistant URIs.
*  **Per-type RAR refinement rules**, defined by the specifications that define RAR types; such rules extend {{rar-dimension}} for their types without modifying this document.
*  **Re-authorization without a new hop.**  Another specification can define a mechanism for recording it, including how a validated re-authorization establishes a replacement bound for the dimensions it covers, and the evidence, trust, ordering, expiry, preservation, and discovery rules it needs.  An issuer or recipient that supports such a mechanism uses the replacement bound in place of the affected receipt's `bounds[D]` wherever this document compares against it, for the covered dimensions only.  One that does not support it MUST apply the recorded bounds defined here.

Companion rules MUST NOT relax any requirement needed for conformance to this profile, other than through a replacement bound as described above; they MAY add rejection conditions.  A companion's partial-validation mode, defined under its own normative scope as {{I-D.mcguinness-oauth-actor-receipts}} requires, is not conformance to this profile.

# Security Considerations

Authority bounds do not replace token validation or authorization ({{consumer-processing}}).  The general OAuth 2.0 Security Best Current Practice {{RFC9700}} and the JWT best practices in {{RFC8725}} apply.

## Threat Model {#threat-model}

Bounds inherit the receipts companion's per-issuer, non-transitive trust model; composition with proofs adds an actor-side check with an independent trust anchor.

### Adversaries Mitigated by This Profile

*  **Intermediate authority expansion.**  Detection requires every receipt in the chain to record the dimension.  Recording a wider value fails comparison; recording a narrower value than was issued fails the next enforcing issuer's inbound check or step 4 of {{consumer-processing}}, which compares against the effective values the token carries, not request-time values; omission fails dense-coverage enforcement.
*  **Silent basis change.**  Expansion requires a signed `reauthorized` claim inside a trusted issuer's receipt, scoped to the dimensions it lists.
*  **Issuance beyond actor consent, when proofs are present.**  {{composition-with-proofs}} detects recorded authority broader than the actor-signed target binding at the same hop.

### Adversaries Not Mitigated

*  **Origin issuer choosing broad initial bounds.**  No upstream value constrains the origin; constraining it requires origin-issuer policy, pre-authorization artifacts, or transparency mechanisms.
*  **Fabricated re-authorization by a trusted issuer.**  See {{reauthorization-abuse}}.
*  **Full-chain collusion.**  Colluding issuers can fabricate a monotonic chain at any level; this matches the receipts companion's trust boundary.
*  **Semantic expansion within syntactic subsets.**  See {{scope-subsumption-gaps}}.
*  **Expansion at a vocabulary change.**  A `domain_transition` basis reset is an unverified re-expression of authority ({{domain-transitions}}).
*  **Compromised current outer-token issuer.**  As in the receipts companion, a compromised outer issuer can omit this profile's claims entirely; recipients detect that downgrade only by requiring the evidence ({{discovery-capability-signaling}}).

## Re-Authorization Abuse {#reauthorization-abuse}

If an issuer trusted to record re-authorization is compromised or over-trusted, it can use those records to justify arbitrary expansion.

*  Recipients MUST evaluate re-authorization trust separately from receipt trust: which issuers are trusted to capture re-authorization, for which subjects, and by which methods, is explicit local policy.  A recipient MAY accept an issuer's receipts while rejecting its re-authorization records; a rejected re-authorization is a failed basis reset, and the chain is then evaluated without it, which typically fails monotonicity and rejects the token's bounds evidence.
*  Deployments needing strong re-authorization integrity SHOULD require `reauthorized.artifact` and SHOULD validate the referenced artifact against the authority that captured the event (for example, verifying a consent receipt's signature), rather than accepting the recording issuer's bare assertion.
*  A `domain_transition` basis makes values non-comparable, so an attacker who can insert one can present any expansion as authorized.  Recipients SHOULD restrict which issuers may record domain transitions to the deployment's known boundary issuers and TTSs.

## Scope Subsumption Gaps {#scope-subsumption-gaps}

Set-membership comparison detects only verbatim expansion: a lexically new token fails the subset check even when semantically narrower, and a preserved token can be broadened by configuration changes at the AS that defines it.  Deployments whose scope grammars carry hierarchy or wildcard semantics MUST either emit explicit narrowest-form scopes at every hop, or define and apply a deployment-specific comparison rule; this document does not define scope subsumption, and a general solution belongs in its own specification.

# Privacy Considerations {#privacy-considerations}

The privacy considerations of {{I-D.mcguinness-oauth-actor-receipts}} apply, including cross-service correlation and retention beyond token lifetime.  Bounds add disclosure of authority information:

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

This document requests registration of the following metadata name in the "OAuth Authorization Server Metadata" registry {{RFC8414}}:

*  Metadata Name: `authority_bounds_supported`
*  Metadata Description: Array of authority-dimension names for which the server records receipt-attested bounds and enforces issuance-time monotonicity
*  Change Controller: IETF
*  Specification Document(s): This document

## OAuth Protected Resource Metadata Registration

This document requests registration of the following metadata name in the "OAuth Protected Resource Metadata" registry {{RFC9728}}:

*  Metadata Name: `authority_bounds_required`
*  Metadata Description: Array of authority-dimension names for which the resource requires dense receipt-attested bounds enforcement
*  Change Controller: IETF
*  Specification Document(s): This document

# Acknowledgments

The author thanks the OAuth Working Group for the specifications underlying this profile.  The PIC Model {{PIC-MODEL}} also informed the non-expansion property, which this document represents through signed OAuth evidence.

Contributors and reviewers will be acknowledged in future revisions.

--- back

# Examples

The examples in this appendix show decoded contents; actual receipts are signed JWTs in compact serialization.  Timestamps are illustrative.  The scenario continues the two-hop travel example of {{I-D.mcguinness-oauth-actor-receipts}}: Alice delegates to an AI travel-assistant agent through the enterprise AS, and the agent's token is exchanged at the travel-provider AS, which adds a booking tool as the outermost actor.  Because these receipts carry `bounds`, they are different byte strings from the receipts shown in that document's examples and carry their own identifiers.  The later examples vary {{example-narrowing}} and show only the members that change; a token or receipt that changes is a different byte string with its own `jti`, and the `origin_jti` or `prh` that references it changes with it.

## Example: Two-Hop Chain with Narrowing Bounds {#example-narrowing}

The following example shows the outer token:

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

A recipient verifies this chain as follows:

*  Scope narrows to `trips:book`.
*  The resource set narrows to the bookings endpoint, so the chain also passes verification where `resource` is required, when the resource server uses the identifier of the protected resource receiving the request as the token's effective value (step 4).  URI prefixes do not imply containment.
*  The outer token's scope equals the newest recorded scope.
*  Audience changes are recorded without comparison under the default audience rules.

## Example: Re-Authorization at a Hop {#example-reauthorization}

In this example, Alice completes step-up authentication at the travel-provider AS before that AS adds the booking tool, so that the tool can also reach the payments resource.  Token Exchange lets the AS issue for that additional requested resource, while the scope stays within the inbound token's.  `actor_receipts[1]` is unchanged; `actor_receipts[0]` records the broader resource set in `bounds` and adds `reauthorized`:

~~~json
{
  "bounds": {
    "scope": "trips:book",
    "aud": ["https://api.travel-provider.example"],
    "resource": [
      "https://api.travel-provider.example/bookings",
      "https://api.travel-provider.example/payments"
    ]
  },
  "reauthorized": {
    "sub": "https://idp.enterprise.example/users/alice",
    "iss": "https://as.travel-provider.example",
    "method": "step_up",
    "iat": 1776745140,
    "dimensions": ["resource"]
  }
}
~~~

A recipient that requires `resource` monotonicity ({{protected-resource-metadata}}) and whose policy trusts the travel-provider AS to record `step_up` re-authorization for Alice ({{reauthorization-abuse}}) verifies this chain as follows:

*  Step 3 of {{consumer-processing}} skips the `resource` comparison between the receipts because `reauthorized.dimensions` lists `resource`; without that listing, the payments resource would fail it.
*  Step 4 passes when the token's effective resource is within the newly recorded set.
*  Other dimensions compare as in {{example-narrowing}}.

## Example: Expansion Not Covered by Re-Authorization {#example-unlisted-expansion}

This variant of {{example-reauthorization}} also records `authorization_details` on both receipts, and the travel-provider AS widens that dimension without listing it in `reauthorized.dimensions`.  `actor_receipts[1].bounds` adds:

~~~json
{
  "authorization_details": [
    {
      "type": "trip_booking",
      "actions": ["read", "book"],
      "locations": ["https://api.travel-provider.example/bookings"]
    }
  ]
}
~~~

`actor_receipts[0].bounds` adds the following member, and the outer token carries it as a top-level claim:

~~~json
{
  "authorization_details": [
    {
      "type": "trip_booking",
      "actions": ["book", "cancel"],
      "locations": ["https://api.travel-provider.example/bookings"]
    }
  ]
}
~~~

Step 3 of {{consumer-processing}} fails for `authorization_details`: `reauthorized.dimensions` lists only `resource`, and the newer object adds the `cancel` action, so it refines no older object, whether or not the recipient has a refinement rule for `trip_booking` ({{rar-dimension}}).  The recipient therefore rejects the token's bounds evidence, including the re-authorized `resource` bound.

# Document History
{:numbered="false"}

[[ To be removed from the final specification ]]

-00

* Initial version.

# OAuth Actor Chain Authority Bounds

> **Status:** Early-stage architectural sketch.  Not a published I-D.
> Companion to [draft-mcguinness-oauth-actor-profile](../draft-mcguinness-oauth-actor-profile.md)
> and [draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md).
> See [README](./README.md) for context.
> Last updated: 2026-05-18

## Abstract

This document defines OAuth Actor Chain Authority Bounds, an optional
companion profile that records and verifies how authority changes
across the hops of a delegated token chain.  The core actor profile
defines the visible actor representation, and the receipts companion
attests which issuer added each visible hop.  This profile adds a
separate invariant: governed authority (`scope`, `aud`, `resource`,
and `authorization_details`) does not expand across hops within a
delegation chain, except at explicit re-authorization events.  Bounds
are carried as receipt claims so monotonicity is offline-verifiable
for recipients that trust the receipt issuers.  An issuer-attested
flag is defined for deployments that do not use receipt-carried
bounds.

Where receipts mitigate fabrication of *past* hop provenance and
proofs mitigate fabrication of *actor participation*, bounds mitigate
undetected *authority expansion* by an intermediate issuer whose
receipt is visible to the recipient and whose receipt assertions can
be compared to the surrounding chain.

## 1. Introduction

### 1.1 The gap

The core actor profile defines visible-chain rules.  The receipts
companion defines per-hop AS-signed provenance.  The proofs sketch
defines per-hop actor-signed attestation.  None of these constrain
how authority can change across hops.

[RFC 8693](https://www.rfc-editor.org/rfc/rfc8693) Token Exchange
explicitly permits the issued token to carry different `scope`,
`aud`, `resource`, and `authorization_details` from the inbound
`subject_token` and `actor_token`, subject only to AS local policy.
The receipts spec is explicit that "receipts attest hop history at
the time of original issuance; they do not assert that the represented
delegation is still active" and that this profile "does not prove
that the current token's audience, scope, or expiration were in force
when older receipts were created."

A recipient inspecting a delegated token therefore cannot determine
whether authority was preserved or restricted across exchange hops,
or whether some intermediate issuer expanded authority beyond what
the upstream principal originally consented to.  Without recorded
bounds, adversarial or misconfigured intermediates can broaden scope,
audience, resources, or authorization details at a hop and the
receipt chain only proves that the hop existed; it does not prove
that the hop preserved upstream authority limits.

### 1.2 What this profile adds

This profile defines an **authority monotonicity** property over a
set of governed dimensions across the hops of a delegation chain:

- **Scope monotonicity**: each newer hop's `scope` is a subset of
  the immediately older hop's `scope`.
- **Audience monotonicity**: each newer hop's `aud` is a subset of
  the immediately older hop's `aud`.
- **Resource monotonicity**: each newer hop's `resource` is a subset
  of the immediately older hop's `resource`.
- **Authorization-details refinement**: each newer hop's
  `authorization_details` is a refinement of the immediately older
  hop's `authorization_details` per the rules in Section 4.

Authority can decrease or stay the same; it cannot expand without an
explicit re-authorization event signaled by the `reauthorized` claim
defined in Section 5.

Bounds at each hop are recorded as a receipt claim
(Section 5).  Recipients verify monotonicity offline by walking the
receipt chain (Section 7).  Deployments that do not use receipts
can use an issuer-attested flag, `authority_bounds_enforced`, on the
outer token (Section 6); this is a weaker current-issuer assertion,
not independent chain evidence.

This primitive is also the wire-format enforcement layer that
[Mission-Bound OAuth](https://notes.karlmcguinness.com/series/mission-bound-oauth/)
deployments need at the protocol boundary: per-hop attestation that
derived tokens have not widened beyond what the Mission approved.
See Section 13.6 for the composition pattern.

### 1.3 Threat-model coverage

| Adversary | Receipts | Proofs | Bounds |
|---|---|---|---|
| Compromised downstream issuer fabricating prior-hop provenance | Mitigated | Not the primary control | Not the primary control |
| Compromised current outer-token issuer fabricating actor participation | NOT mitigated | Mitigated | NOT mitigated |
| Compromised intermediate issuer silently expanding scope / audience / resources | NOT mitigated | NOT mitigated | Mitigated when the recipient validates receipt-attested bounds from trusted issuers |
| Compromised origin issuer choosing broad initial bounds | NOT mitigated | NOT mitigated | NOT mitigated |
| Full-chain collusion | NOT mitigated | NOT mitigated | NOT mitigated |

Bounds compose with receipts and proofs; a token MAY carry any
combination.  Receipts-only mode attests history but not authority
monotonicity; issuer-flag-only mode records a current-issuer
assertion but is not independently verifiable across the chain;
receipts + bounds attest history AND constrain change; receipts +
proofs + bounds is the strongest combination.

## 2. Terminology

This document uses terminology from
[draft-mcguinness-oauth-actor-profile](../draft-mcguinness-oauth-actor-profile.md)
and [draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md).

- **Governed dimension:** an authority claim whose value across hops
  is constrained by this profile.  This profile defines four governed
  dimensions: `scope`, `aud`, `resource`, and `authorization_details`.
- **Authority bounds:** the values of governed dimensions in effect
  at a given hop.
- **Monotonicity:** the property that authority bounds at a newer
  hop are a subset (or refinement) of the bounds at the immediately
  older hop.  In the receipts array ordering, `receipt[i]` is newer
  than `receipt[i+1]`, so `receipt[i].bounds[D]` is checked against
  `receipt[i+1].bounds[D]`.
- **Re-authorization:** an explicit event in which the upstream
  principal consents to new (possibly broader) authority bounds.
  Signaled by the `reauthorized` claim and breaks the monotonicity
  chain at that hop.

## 3. Relationship to the Receipts Spec

This profile is an extension of
[draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md).
It defines:

- A new optional receipt claim, `bounds` (Section 5.1).
- A new optional receipt claim, `reauthorized` (Section 5.2).
- A new optional outer-token claim, `authority_bounds_enforced` (Section 6).
- A new optional outer-token array, `bounds_events`, for non-hop
  re-authorization events (Section 5.3).
- Consumer-processing rules extending the receipts spec's
  §Consumer Processing (Section 7).
- New metadata names following the receipts spec's claim-pair
  convention (Section 8).

This profile uses three receipts-spec extension surfaces:

- **Per-receipt extension claims** (`bounds`, `reauthorized`) per
  the per-receipt pattern described in the receipts spec's
  §Claim-Pair Convention.  These claims live inside each receipt
  JWT and inherit the receipt issuer's signature.
- **Cross-receipt verification rules** (monotonicity) per the
  receipts spec's §Extensibility authoring rule for cross-receipt
  rules.  Recipients walk the receipt chain comparing bounds
  across adjacent receipts.
- **Non-hop event artifacts** (`bounds_events`) per the receipts
  spec's §Extensibility extension surface for non-hop events.
  These cover re-authorization events that occur without a new
  visible actor hop.

The receipts spec's chain-linkage rules (`prh`, byte-for-byte
preservation, partial coverage, originating-issuance binding) apply
unchanged.  This profile does not alter how receipts are signed,
linked, or carried.

## 4. Governed Dimensions and Subset Semantics

This section defines the four governed dimensions and the subset
relation for each.  Consumer verification (Section 7) applies these
relations to determine monotonicity.

The subset relation is evaluated only when both compared values are
present.  For required dimensions, absence is not treated as a
successful subset relation: if an older receipt lacks `bounds[D]`,
the recipient cannot verify that a newer value of `D` is non-
expanding relative to that hop.  Section 7 defines how sparse bounds
coverage is handled.

### 4.1 `scope` ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749) Section 3.3)

A space-separated string of scope tokens.  Subset semantics:

- Parse both sides into the set of distinct scope tokens
  (separator: ASCII space, U+0020).
- `scope_a ⊆ scope_b` if and only if every token in `scope_a` is
  also in `scope_b`.
- The empty string represents the empty set and is a subset of every
  scope set.

This profile applies set-membership subset semantics only.  It does
NOT attempt to interpret scope-language semantics (e.g.,
`read:user/*` subsuming `read:user/123`); such subsumption is
deployment-specific and depends on the AS's scope grammar.  Issuers
that wish to express subsumed scopes MUST emit the explicit, narrowest
scope at each hop.

### 4.2 `aud` ([RFC 7519](https://www.rfc-editor.org/rfc/rfc7519) Section 4.1.3)

A single string or an array of strings.  Subset semantics:

- A single-string `aud` is treated as a one-element set.
- `aud_a ⊆ aud_b` if and only if every value in `aud_a` is also in
  `aud_b`.
- An empty array represents the empty set and is a subset of every
  audience set.

### 4.3 `resource` ([RFC 8707](https://www.rfc-editor.org/rfc/rfc8707))

An array of absolute-URI strings recording the effective resource
indicator set for the issued token.  RFC 8707 defines `resource` as
a request parameter rather than a standard JWT claim; this profile
uses `bounds.resource` to record the resource indicators that the
issuer applied when shaping the token.  If a deployment also carries
a top-level `resource` claim, Section 7 applies the same comparison
rules to that claim.

Subset semantics:

- Compare URIs by canonical form per
  [RFC 3986](https://www.rfc-editor.org/rfc/rfc3986) Section 6.2.
  Implementations SHOULD normalize before comparison (case
  normalization for scheme and host, percent-encoding normalization,
  path normalization).
- `resource_a ⊆ resource_b` if and only if every canonical URI in
  `resource_a` is also in `resource_b`.
- An empty array represents the empty set and is a subset of every
  resource set.

URI prefix subsumption (e.g., `https://api.example.com/v1/` covering
`https://api.example.com/v1/users`) is NOT applied by this profile.
Issuers wishing to express prefix subsumption MUST emit explicit URIs
at each hop.

### 4.4 `authorization_details` ([RFC 9396](https://www.rfc-editor.org/rfc/rfc9396))

An array of RAR objects, each tagged with a `type` field.  Refinement
semantics are per-type:

- Two RAR objects refine if they share the same `type` and every
  member of the refining object is consistent with the corresponding
  member of the broader object under the type's refinement rules.
- An array `ad_a` refines `ad_b` if and only if every object in
  `ad_a` refines some object in `ad_b`.  (Objects in `ad_b` MAY be
  absent in `ad_a`; this is narrowing.  Objects in `ad_a` that do not
  refine any object in `ad_b` represent expansion and are rejected.)

This profile defines refinement rules for the following type-agnostic
members (when present):

- `actions` (array of strings): refining set MUST be a subset.
- `locations` (array of URI strings): refining set MUST be a subset
  (URI subset semantics per Section 4.3).
- `datatypes` (array of strings): refining set MUST be a subset.
- `identifier` (string): refining value MUST equal the broader value
  (no narrowing of a single identifier).

Type-specific members beyond these four are NOT given normative
refinement rules by this profile.  Future companion profiles or
RAR-type registrations MAY define per-type refinement.  Recipients
processing RAR types without defined refinement rules MUST either:

1. Reject the bounds chain (conservative), or
2. Skip refinement verification for those types under explicit local
   policy (permissive).

The default is conservative rejection.

### 4.5 Dimensions explicitly NOT governed

- **`exp`**: receipt expiration is governed by the receipts spec.
  Token lifetime affects replay and freshness, but it is not an
  authority dimension in this profile.
- **`cnf`**: presenter rebinding is legitimate at TTS hops; not
  governed.
- **`sub`**: already constrained by receipts spec's hop-alignment
  rules.
- **`sub_profile`**: subject classification is not authority;
  not governed.

## 5. Receipt Extensions

### 5.1 `bounds`

OPTIONAL.  A JSON object recording the authority bounds in effect at
this hop.  Contains any combination of the following members
corresponding to the governed dimensions (Section 4):

- `scope`: a string (space-separated scope tokens).
- `aud`: a string or an array of strings.
- `resource`: an array of URI strings.
- `authorization_details`: an array of RAR objects.

Each member, when present, MUST equal the corresponding effective
value used for the token issued at this hop.  For `scope`, `aud`,
and `authorization_details`, that is ordinarily the top-level token
claim when the token format carries that claim.  For `resource`, it
is the resource-indicator set applied by the issuer when shaping the
token, whether or not the token format carries a top-level
`resource` claim.

An absent member means the issuer did not record a bound for that
dimension in this receipt.  It does not prove that the dimension was
unconstrained or absent on the issued token.  Recipients that require
verification for a dimension MUST treat an absent `bounds[D]` as
missing evidence for that dimension.

A receipt MAY omit `bounds` entirely.  A chain MAY contain receipts
with and without `bounds`; consumer-processing rules (Section 7)
define both a sparse verification mode and a stricter required-
dimension mode.

### 5.2 `reauthorized`

OPTIONAL.  A JSON object recording explicit re-authorization of the
delegation at this hop.  Contains:

- `sub`: REQUIRED.  Identifier of the principal that re-authorized
  the delegation.  MUST equal either the outer token's top-level
  `sub` at this hop or an upstream subject in the chain.
- `iss`: REQUIRED.  Identifier of the AS or other authority that
  captured the re-authorization event.
- `method`: REQUIRED.  A URI or registered keyword identifying the
  re-authorization method.  Registered values:
  - `interactive_consent`: end-user re-authorized through an
    interactive prompt.
  - `step_up`: re-authorization via step-up authentication.
  - `refresh_grant`: re-issuance under a refresh token with bounds
    different from the prior access token.
  - `policy_grant`: programmatic re-authorization under a deployment-
    specific policy artifact.
  - `mission_lifecycle`: re-authorization triggered by a Mission
    lifecycle event in a Mission-Bound OAuth deployment (Mission
    approval, scope extension by approver, resumption from suspended
    state).  When this method is used, `reauthorized.artifact` SHOULD
    reference the corresponding Mission event record.  See Section 13.6.
- `iat`: REQUIRED.  Time of re-authorization.
- `artifact`: OPTIONAL.  A URI or JWT identifier referencing the
  external artifact (e.g., consent receipt JWT, step-up assertion)
  evidencing the re-authorization.  Recipients MAY fetch and validate
  this artifact under local policy.

When `reauthorized` is present on a receipt, monotonicity verification
treats that hop as a new origin: bounds at this hop are not required
to be a subset of older bounds (Section 7).

A receipt that records `reauthorized` MUST also carry `bounds`
recording the post-reauthorization bounds for every governed
dimension that was expanded by the re-authorization event.  It SHOULD
carry `bounds` for every governed dimension in effect at that hop.

### 5.3 `bounds_events`

OPTIONAL.  A top-level outer-token claim parallel to `actor_receipts`.
An array of strings, each the compact serialization of a signed JWT
"bounds event" representing a re-authorization that changed authority
bounds without producing a new visible actor hop.  Follows the
non-hop event artifact pattern in the receipts spec's §Extensibility.

The `reauthorized` claim in Section 5.2 covers re-authorization that
occurs *at* a new hop (and so lives on the receipt for that hop).
`bounds_events` covers re-authorization that occurs *between* hops
(refresh-grant with broader scope, step-up authentication mid-flow
that widens, Mission lifecycle scope extension by an approver
without a new actor): no new receipt is created, so the event needs
its own carrier.

Each event JWT:

- MUST have `typ` equal to `bounds-event+jwt`.
- MUST include `iss`, `iat`, `exp`, and `jti` per JWT conventions.
- MUST include `event_type` from a registered value.  The initial
  registered value is `bounds_reauth`.  Future companions MAY define
  additional event types.
- MUST anchor to either a specific receipt (via a `receipt_jti`
  claim equal to that receipt's `jti`) or to the delegation flow
  (via the delegation correlator defined by the core actor profile,
  when present).  At least one anchor MUST be present.
- MUST include `new_bounds`: a JSON object recording the bounds in
  effect after the event, using the same shape as the `bounds`
  receipt claim defined in Section 5.1.  At least one governed
  dimension MUST be present.
- MUST include `reauthorized`: a JSON object with the same structure
  as the `reauthorized` receipt claim defined in Section 5.2.

Events MAY chain among themselves via `prh` / `prh_alg` if a
deployment needs ordered event verification; this profile does not
REQUIRE event chaining.

The array, when present:

- MUST NOT be empty.
- MAY be unordered, but recipients SHOULD process events in `iat`
  order for consumer verification (Section 7.5).
- MUST be byte-for-byte preserved across reissuance, refresh, and
  forwarding by intermediate issuers, following the same preservation
  rule as `actor_receipts`.

### 5.4 `bounds_events_complete`

OPTIONAL.  A boolean JWT claim in the outer token.  When `true`, the
issuer attests that `bounds_events` covers every non-hop bounds-
changing event that occurred during the delegation lifetime as of
token issuance.  When `false` or absent, coverage may be partial and
the recipient cannot rely on the absence of events to mean no
re-authorization occurred.

Per the claim-pair convention in the receipts spec, this is the
`<name>_complete` member for the `bounds_events` artifact.

## 6. Outer-Token Claim

### `authority_bounds_enforced`

OPTIONAL.  A boolean JWT claim in the outer token.  When `true`, the
issuer attests that, at the moment of token issuance:

- The issued token's `scope`, `aud`, `resource`, and
  `authorization_details` are no broader than the corresponding
  effective values of the inbound credentials or policy artifacts
  that produced this token.
- Any change to a governed dimension that resulted in expansion was
  accompanied by a re-authorization event recorded in the issued
  token's `actor_receipts` chain (where receipts are in use).

When `authority_bounds_enforced` is `true` and the token carries
`actor_receipts` with `bounds` claims, the receipt-chain monotonicity
verification in Section 7 MUST also succeed; the two signals reinforce
each other and inconsistency MUST cause the token to be rejected.

When `authority_bounds_enforced` is `true` and receipts are absent
or carry no `bounds` claims, the attestation is issuer-only; the
recipient relies on trusting the current issuer to have enforced
monotonicity at issuance.

Absence of `authority_bounds_enforced` does not assert that bounds
were violated; it simply means the issuer has not made the attestation.

## 7. Consumer Processing

This section extends the receipts spec's §Consumer Processing.

### 7.1 Receipt-chain monotonicity verification

When `actor_receipts` is present and any receipt carries `bounds`,
the recipient MUST perform the following additional steps after
validating each receipt's signature, structure, and chain linkage
per the receipts spec.

For each governed dimension `D` defined in Section 4:

1. Identify each adjacent receipt pair `(receipt[i], receipt[i+1])`
   for which both receipts carry `bounds[D]`.
2. For each such pair:
   - If `receipt[i]` carries `reauthorized`, skip this pair.
   - Otherwise, verify `receipt[i].bounds[D]` is a subset of (or
     refines) `receipt[i+1].bounds[D]` per the subset semantics
     defined in Section 4.
   - If verification fails, reject the receipt chain.

This sparse verification mode never proves complete monotonicity for
dimension `D` unless every adjacent pair in the relevant visible chain
segment carries `bounds[D]`, except across a `reauthorized` boundary.
Missing `bounds[D]` on an intermediate receipt creates an evidence
gap.  Recipients MAY accept such gaps only when local policy does not
require complete bounds verification for `D`.

If `receipt[0]` carries `bounds[D]` and the outer token carries `D`
at the top level:

3. Verify `outer.D` is a subset of (or refines) `receipt[0].bounds[D]`
   per the subset semantics defined in Section 4.  This permits
   reissuance to narrow further beyond receipt-attested bounds but
   not to expand.
4. If verification fails, reject the receipt chain.

For the `resource` dimension, step 3 compares the current token's
effective resource-indicator set against `receipt[0].bounds.resource`.
If the effective resource set cannot be determined from the token,
introspection response, or trusted local context, the recipient cannot
perform outer-token bounds verification for `resource`.

### 7.2 Issuer-attested enforcement

If `authority_bounds_enforced` is present on the outer token:

5. Verify it is a JSON boolean.
6. When `true` and `actor_receipts` is present with `bounds` claims,
   step 1 through step 4 MUST also succeed; the combined signal is
   the recipient's basis for trust.
7. When `true` and `actor_receipts` is absent or carries no `bounds`
   claims, the attestation is recorded but no further verification is
   performed; recipients evaluate the attestation per their local
   trust policy for the current outer-token issuer.

### 7.3 Required-dimensions enforcement

When Protected Resource Metadata declares `authority_bounds_required`
with the value `true`, the recipient MUST verify
`authority_bounds_enforced: true`.
When it also declares `authority_bounds_dimensions`, the recipient
MUST verify that for each declared dimension:

- `bounds[D]` appears on `receipt[0]`, AND
- when the outer token carries `D` or its effective value can be
  determined, `outer.D` is a subset of `receipt[0].bounds[D]`, AND
- for every adjacent receipt pair within the visible receipt chain
  that does not cross a `reauthorized` boundary, both receipts carry
  `bounds[D]` and monotonicity succeeds for `D` per Section 7.1.

A token that fails any required dimension MUST be rejected.  A token
with sparse bounds coverage does not satisfy a required dimension
unless local policy explicitly accepts partial receipt coverage for
that request.  A resource server that needs full-chain authority
monotonicity SHOULD also require complete actor-receipt coverage via
the receipts profile's `actor_receipts_complete_required` metadata.

### 7.4 Reauthorization scope

A `reauthorized` claim on `receipt[k]` causes monotonicity to "reset"
at that hop: receipts older than hop *k* (receipt[k+1] and beyond)
are not constrained to be supersets of receipt[k]'s bounds.  Receipts
newer than hop *k* (receipt[k-1] and toward receipt[0]) remain
constrained to be subsets of receipt[k]'s post-reauth bounds.

Multiple `reauthorized` events in a chain are permitted.  Each
defines a new monotonicity boundary; the chain is verified piecewise
between re-authorization events.

### 7.5 `bounds_events` verification

When `bounds_events` is present, the recipient MUST perform the
following steps as part of monotonicity verification:

1. Validate each event JWT individually: parse, verify `typ` equals
   `bounds-event+jwt`, verify the issuer is in the recipient's
   trusted-issuer set, resolve the signing key, validate the
   signature, enforce `exp` and `iat`, verify the presence of
   required claims (`event_type`, `new_bounds`, `reauthorized`,
   and at least one anchor).  Reject any event that fails.
2. Group events by their anchor.  Receipt-anchored events
   (`receipt_jti`) apply to the named receipt; flow-anchored events
   (delegation correlator) apply to the chain as a whole.
3. For each governed dimension `D` and each adjacent receipt pair
   `(receipt[i], receipt[i+1])`, determine the *effective upper
   bound* applicable to `receipt[i].bounds[D]` as follows:
   - Start with `receipt[i+1].bounds[D]` as the default.
   - If a `bounds_reauth` event anchored to `receipt[i+1]` has
     `iat` strictly later than `receipt[i+1].iat` and no later
     than `receipt[i].iat`, replace the default with the event's
     `new_bounds[D]`.  If multiple such events exist, apply them
     in `iat` order; the latest one wins.
   - If a `bounds_reauth` event is flow-anchored and has `iat`
     within the same window as above, apply it the same way; later
     receipts-anchored events override flow-anchored events at the
     same `iat` only if the deployment specifies that precedence,
     otherwise events are processed in `iat` order with the latest
     applicable event winning.
4. Verify `receipt[i].bounds[D]` is a subset of the effective upper
   bound.  Reject if not.
5. The relation between the outer token and `receipt[0]` is governed
   the same way: events anchored to `receipt[0]` (or flow-anchored)
   with `iat` between `receipt[0].iat` and the outer token's `iat`
   modify the effective upper bound the outer token's value must
   satisfy.

Sparse `bounds_events` (with `bounds_events_complete: false` or
absent) does not invalidate the receipt chain by itself, but
recipients requiring complete event coverage MUST reject tokens that
lack `bounds_events_complete: true` when Protected Resource Metadata
requires it.

Event-driven re-authorization is conceptually equivalent to a
`reauthorized` boundary on a receipt: the chain monotonicity
"resets" at the event's anchor point with the event's `new_bounds`
as the new upper bound for newer receipts.

## 8. Discovery and Capability Signaling

### 8.1 Authorization Server Metadata

- `authority_bounds_supported`: OPTIONAL.  Boolean.  When `true`, the
  AS enforces bounds monotonicity at token-exchange, refresh, and
  reissuance time for at least one governed dimension.  An AS that
  sets this value to `true` SHOULD emit `authority_bounds_enforced`
  with the value `true` on tokens for which enforcement was applied
  and MAY include `bounds` in actor receipts when receipts are in use.
- `authority_bounds_dimensions_supported`: OPTIONAL.  Array of
  strings.  Names the governed dimensions the AS enforces.  Drawn
  from `{"scope", "aud", "resource", "authorization_details"}`.
  Absence means the AS does not advertise dimension-specific
  coverage; clients and resource servers MUST NOT infer support for
  all four dimensions from omission.

### 8.2 Protected Resource Metadata

- `authority_bounds_required`: OPTIONAL.  Boolean.  When `true`, the
  resource server requires `authority_bounds_enforced: true` on
  inbound delegated tokens for request paths where authority bounds
  are enforced.
- `authority_bounds_dimensions`: OPTIONAL.  Array of strings.  Names
  the dimensions the resource server requires to be governed and
  bounds-attested on receipts or equivalent introspection-returned
  receipts.  Drawn from the same set as the AS metadata above.  A
  resource server SHOULD NOT set this parameter unless it also sets
  `authority_bounds_required: true`.
- `bounds_events_required`: OPTIONAL.  Boolean.  When `true`, the
  resource server requires `bounds_events`, when any non-hop
  re-authorization occurred during the delegation lifetime, to be
  present on the inbound token.
- `bounds_events_complete_required`: OPTIONAL.  Boolean.  When `true`,
  the resource server additionally requires `bounds_events_complete:
  true` on the inbound token.  A resource server SHOULD NOT set this
  parameter unless it also sets `bounds_events_required: true`.

### 8.3 Authorization Server Metadata for Events

- `bounds_events_supported`: OPTIONAL.  Boolean.  When `true`, the AS
  emits `bounds_events` for non-hop re-authorization that affects
  governed bounds.
- `bounds_events_complete_supported`: OPTIONAL.  Boolean.  When `true`,
  the AS is capable of emitting `bounds_events_complete: true` for
  tokens whose event coverage is complete as of issuance.

### 8.4 Introspection

When introspection is used, the introspection response MAY include
`authority_bounds_enforced`, `bounds_events`, and
`bounds_events_complete` alongside the receipts-spec introspection
members.  Receipts returned via introspection carry `bounds` and
`reauthorized` claims in their JWT payload as defined in Section 5.

## 9. Issuer Processing

### 9.1 First receipt with bounds

When an issuer creates a delegated token with a new outermost actor
hop, no inbound `actor_receipts` are being preserved, and the
deployment uses authority bounds:

1. Determine the issued token's `scope`, `aud`, `resource`, and
   `authorization_details` per the underlying grant rules.
2. Include `bounds` in the receipt carrying any of those dimensions
   the issuer is willing to attest.  Each `bounds` member MUST equal
   the corresponding effective value used for the issued token, as
   defined in Section 5.1.
3. Optionally set `authority_bounds_enforced: true` on the outer
   token if the AS enforces monotonicity at exchange time.

### 9.2 Extending a chain with bounds

When an issuer adds a new outermost actor hop and preserves an
inbound `actor_receipts` array:

1. Apply the receipts spec's extension rules (validate inbound chain,
   preserve byte-for-byte, prepend new receipt).
2. Compute the issued token's governed dimensions.
3. **Monotonicity check at issuance:** for each governed dimension
   `D` the issuer enforces, verify the issued token's effective `D`
   is a subset of (or refines) the corresponding effective inbound
   value for `D`.  When inbound receipts carry `bounds[D]`, the new
   bounds MUST also be checked against the newest applicable inbound
   receipt bound, except across a re-authorization boundary.  If
   verification fails:
   - If the deployment has an authoritative re-authorization event
     for the requesting subject, the issuer MAY proceed and set
     `reauthorized` on the new receipt with details of that event.
   - Otherwise, the issuer MUST fail the token-exchange request
     under the underlying protocol's error model.
4. Include `bounds` in the new receipt recording the issued token's
   effective bounds for every dimension the issuer enforces and any
   additional dimensions the issuer is willing to attest.
5. Optionally set `reauthorized` on the new receipt if the broadening
   was authorized.
6. Set `authority_bounds_enforced: true` on the outer token only if
   the issuer performed the applicable monotonicity checks for the
   dimensions it enforces and any expansion was covered by a valid
   re-authorization event.

### 9.3 Reissuance without a new actor hop

The receipts spec permits reissuance to change `aud`, `scope`, `cnf`,
`exp` without creating a new receipt.  When this profile is in use:

- Reissuance MUST NOT expand any governed dimension beyond the bounds
  recorded on `receipt[0]` of the inherited chain.
- Reissuance MAY narrow further; this is permitted by the
  `outer.D ⊆ receipt[0].bounds[D]` consumer rule (Section 7.1
  step 3).
- An issuer that needs to expand bounds during reissuance MUST NOT
  carry the inherited receipt chain forward unchanged while also
  claiming bounds enforcement for the expanded dimension.  It MUST
  either fail the request, issue without receipt-attested bounds where
  local policy and resource requirements permit, or perform a fresh
  issuance that records a `reauthorized` event in a new receipt under
  rules defined by a future revision of this profile or another
  companion profile.

### 9.4 Refresh-token reissuance

The receipts spec requires refresh-token AS to persist `actor_receipts`
in issuer-controlled storage.  When this profile is in use, the AS
MUST also persist sufficient state to verify monotonicity at refresh
time.  Specifically:

- The bounds recorded on the inherited `receipt[0]` are the upper
  bound for any refreshed access token.
- A refresh grant that produces a token with bounds expanding beyond
  `receipt[0].bounds` is treated as a re-authorization with `method`
  set to `refresh_grant`.  Because the base receipts profile does not
  create a new receipt for ordinary reissuance without a new actor
  hop, the issuer MUST NOT present the inherited receipt chain as
  satisfying receipt-attested bounds for the expanded token unless a
  companion rule defines how the new reauthorization receipt is
  created and aligned.

## 10. Security Considerations

### 10.1 Bounds without receipts

`authority_bounds_enforced: true` on an outer token without receipt-
attested bounds is an unverifiable issuer assertion.  Recipients in
this mode rely entirely on trusting the current outer-token issuer
to have enforced monotonicity at issuance.  A compromised issuer can
set the flag to `true` while expanding bounds without detection.

Deployments needing offline evidence that authority did not expand
relative to prior trusted issuers MUST use receipt-attested bounds
(Section 5.1).  Receipt-attested bounds still do not make a
compromised current issuer trustworthy: such an issuer can omit
bounds where policy permits, fabricate its own `reauthorized` claim,
or issue a token that recipients reject only if they enforce this
profile strictly.

### 10.2 Origin issuer not constrained

This profile constrains bounds *relative to a predecessor*.  The
origin issuer (the AS that issues the innermost / oldest receipt)
chooses the initial bounds at its discretion; no upstream credential
exists to constrain it.  Deployments needing constraint on origin
bounds MUST establish that constraint through external means:
policy at the origin AS, pre-authorization artifacts, or transparency
logging of origin issuances.

### 10.3 Full-chain collusion

If every issuer in a chain is compromised or colluding, they can
fabricate a monotonic chain at any level.  Bounds defend against
*individual* compromised intermediates, not collusion across all
issuers.  This matches the receipts spec's threat boundary.

### 10.4 Re-authorization abuse

A compromised or over-trusted issuer can fabricate `reauthorized`
events to justify arbitrary bounds expansion.  Mitigations:

- Recipients MUST evaluate the trustworthiness of issuers that record
  `reauthorized` events.  An issuer not trusted to record valid
  re-authorization SHOULD have its `reauthorized` events rejected
  even when its receipts are otherwise valid.
- The `reauthorized.artifact` member, when present, lets recipients
  validate an out-of-band re-authorization artifact independently.
- Deployments needing strong re-authorization integrity SHOULD require
  `reauthorized.artifact` and SHOULD verify the artifact's signature
  against a trusted re-authorization authority.

### 10.5 Scope subsumption gaps

Section 4.1 applies string-set subset semantics over scope tokens.
This catches verbatim expansion but does NOT catch semantic expansion
where a narrow-looking scope token grants broader authority than its
predecessor under the AS's scope grammar (e.g., a custom
`admin:read:all` token might broaden a `read:user` token under
deployment-specific scope semantics).

Deployments using non-set-membership scope semantics MUST either:

1. Emit explicit, narrowest-form scope at every hop so set semantics
   suffice, or
2. Define a deployment-specific subset rule and apply it at consumer
   verification time.

This profile does not define a general scope subsumption rule.

### 10.6 RAR refinement gaps

Section 4.4 defines refinement rules only for `actions`,
`locations`, `datatypes`, and `identifier` members of RAR objects.
Other type-specific members are not constrained.  An AS that supports
a custom RAR type with type-specific members SHOULD define refinement
rules for those members and SHOULD publish them in deployment
documentation or a future companion profile.

Recipients in conservative mode (Section 4.4) reject RAR refinement
they cannot evaluate, which prevents undetected expansion at the cost
of rejecting tokens with un-evaluable extensions.

### 10.7 Reissuance after key rotation

A reissuance that changes `cnf` (key rotation, presenter rebinding)
does not by itself violate bounds.  However, if the reissuing AS
also narrows governed dimensions, the narrower bounds become the
outer-token bounds and any subsequent reissuance is constrained by
those narrower bounds (consumer rule
`outer.D ⊆ receipt[0].bounds[D]`).  Reissuance cannot undo earlier
narrowing without a `reauthorized` event.

## 11. Privacy Considerations

The privacy considerations of the receipts spec apply, including the
specific guidance on companion extension claims and correlation
surfaces in the receipts spec's §Cross-Service Correlation.  In
addition:

- `bounds` exposes per-hop authority detail (scope, audience,
  resources, RAR details).  This may reveal deployment topology and
  internal resource identifiers to any recipient of the token.
  Issuers SHOULD evaluate disclosure carefully; `bounds` MAY be
  omitted from receipts even when bounds-monotonicity is enforced at
  exchange time (the receipts spec's whole-receipt suppression
  applies to `bounds` claims as well, since they live within a
  receipt JWT that is preserved byte-for-byte).
- `reauthorized` reveals re-authorization events, which may indicate
  privileged actions (step-up authentication, consent prompts) and
  enable correlation across requests.  Issuers SHOULD treat
  `reauthorized.artifact` as carrying similar disclosure risk to
  receipt `cnf`.
- `bounds_events` is independently visible at the outer-token level
  (not inside a receipt) and therefore exposes re-authorization
  history to every recipient of the token even when the
  `actor_receipts` array is suppressed.  Issuers SHOULD treat
  `bounds_events` presence and content as a deployment-level
  disclosure decision distinct from receipt disclosure.  Deployments
  that need re-authorization history for audit but not for relying
  parties SHOULD return `bounds_events` via introspection only.
- Per-receipt `bounds` disclosure across an entire chain provides a
  detailed picture of authority narrowing, which is informative for
  audit but increases correlation surface.  Deployments SHOULD scope
  disclosure to audiences with adequate disclosure agreements.

## 12. Open Design Questions

**Q1: Subset semantics for `scope`.**  Section 4.1 applies
string-set subset.  Some deployments have scope grammars with
hierarchical semantics (`read:*` ⊇ `read:user`).  This profile takes
a conservative stance.  WG discussion: should a more capable scope
subsumption rule be defined?  If so, this is likely a separate
specification on its own, not a member of this profile.

**Q2: RAR refinement extensibility.**  Section 4.4 defines refinement
for four common members.  Adding type-specific refinement requires
either per-type companion profiles or a new IANA registry of RAR
refinement rules.  The latter is more scalable but heavier-weight.

**Q3: Bounds on the outer token vs. receipts.**  This profile carries
`bounds` on receipts and `authority_bounds_enforced` on the outer
token.  An alternative design carries explicit `bounds` on the outer
token (analogous to `act` carrying current-actor identity).  That
design is simpler but does not provide cross-hop monotonicity
without further per-hop attestation.  Drafted version optimizes for
the strong property at the cost of requiring receipts.

**Q4: `reauthorized` as outer-token claim vs. receipt claim.**
Drafted as receipt claim because the re-auth event happens at a
specific hop.  An outer-token alternative, such as a
`reauthorized_at` claim carrying a hop index, is simpler but less
informative.  Trade-off worth WG discussion.

**Q5: Reauthorization without a new actor hop.**  The base receipts
profile creates receipts when actor hops are added and otherwise
preserves receipts byte-for-byte across reissuance.  Bounds expansion
during refresh or same-actor reissuance creates a tension: the
authority state changed, but the visible actor chain did not.  This
sketch treats that as requiring a fresh reauthorization receipt under
rules still to be nailed down.  WG discussion: should authority-
changing reissuance create a new receipt for the same visible actor
hop, start a new chain, or be represented only by the outer-token
issuer assertion?

**Q6: Belongs in core, receipts, or its own companion.**  Three
options:

- **Core.** A SHOULD-not-expand rule baked into the core profile's
  exchange/refresh sections.  Cheapest; no per-hop attestation; just
  a normative recommendation.
- **Receipts extension.** This document's approach: `bounds` as a
  receipt claim, monotonicity check as a consumer-processing step.
  Naturally fits where per-hop attestation lives.
- **Standalone companion.** Drafted as a standalone companion that
  extends receipts.  Cleanest layering but more cross-references.

Drafted version takes the receipts-extension approach in a standalone
document; the core SHOULD-not-expand rule could be added separately
as a non-normative guidance even if this companion is not adopted.

**Q7: Legitimate up-scoping cases.**  The `reauthorized` carve-out
covers step-up authentication, refresh grants with new scope, and
explicit consent re-prompts.  Token translation across trust domains
where the inbound represents only a partial view is a trickier case;
the `method: policy_grant` value is a placeholder for deployment-
specific carve-outs.  WG discussion: are there cases this typology
doesn't cover?

**Q8: Interaction with selective-disclosure companion.**  A future
selective-disclosure companion (subset / actor-only) MAY suppress
entire receipts or define a new selectively disclosable receipt
format.  It cannot remove `bounds` from an existing compact receipt
without invalidating that receipt's signature.  Recipients receiving
a disclosure-narrowed chain MUST be aware that suppressed bounds
material does not invalidate monotonicity; it means the recipient
cannot verify it for those hops.  This profile's verification rules
accommodate sparse `bounds` coverage by design.

**Q9: Time sanity beyond receipt expiration.**  The receipts spec
requires receipt `exp` to cover the token lifetime for chains that
carry the receipt.  Some deployments may also want `iat` monotonicity
(newer receipts have later `iat`) as a sanity check.  Not included in
this profile; could be a separate rule in the receipts spec.

**Q10: Default subset semantics for RAR `type`.**  Section 4.4
requires the same `type` for refinement.  Some RAR types may have
sub-type hierarchies that could refine across distinct `type` values.
Drafted version is conservative; per-type refinement could relax
this.

**Q11: Integration with Mission-Bound OAuth.**  Mission-Bound OAuth
defines a durable AS-resident authority object that bounds is a
natural protocol-layer projection of (Section 13.6).  Drafted version
treats Mission as a deployment context without dependency.  A tighter
integration could define richer rules: the origin receipt's `bounds`
MUST equal the projected Mission Authority Model at issuance;
`mission_ref` on the outer token correlates with the bounds chain;
Mission lifecycle events drive `reauthorized.method` values such as
`mission_lifecycle` automatically.  Drafted as loose composition for
now; tighter integration depends on Mission specs stabilizing.

## 13. Relationship to Other Companions

### 13.1 Receipts

This profile extends receipts.  The receipts spec's chain-linkage,
byte-preservation, originating-issuance, and reissuance rules apply
unchanged.  `bounds` and `reauthorized` are additive claims; receipts
without them remain valid under the receipts spec.

### 13.2 Proofs

Actor-signed proofs (the proofs sketch) attest actor participation
and a target binding (`aud`, `resource`).  This profile constrains
*all* governed dimensions across hops.  When both are in use:

- The proof's target binding represents what the actor *consented*
  to at its hop.
- The receipt's `bounds` represents what the AS *issued* at that hop.
- A recipient in belt-and-suspenders mode MUST verify that
  `bounds.aud` and `bounds.resource`, when present, are subsets of
  the proof's target binding for each hop where both artifacts are
  present.  Scope and `authorization_details` alignment requires a
  proof profile that signs those dimensions or a deployment-specific
  policy mapping.  An AS that issues target bounds broader than the
  actor authorized is detectable.

This composition is stronger than either profile alone: receipts
attest "issuer said this," proofs attest "actor authorized that,"
bounds verify "issuer did not expand authority across the chain."

### 13.3 Transparency

Receipts logged to a transparency log (per the transparency
companion sketches) carry their `bounds` and `reauthorized` claims
along with the rest of the receipt JWT.  Bounds become independently
auditable across the receipt issuer's history.  No additional design
is required.

### 13.4 Selective disclosure (future)

A future selective-disclosure companion may suppress entire receipts
or define a new selectively disclosable receipt format.  It cannot
simply remove claims from the compact JWT receipts defined by the
receipts spec, because that would invalidate the receipt signature
and any `prh` linkage.  Sparse `bounds` coverage is already
accommodated by the consumer rules in Section 7.  A disclosure
companion that hides bounds for a specific recipient MUST clearly
signal the resulting evidence gap so the recipient does not falsely
conclude monotonicity over suppressed material.

### 13.5 Workflow correlation (`acti` / `dlg_id`)

The delegation correlation claim (pending core-profile resolution;
see [issue #3](https://github.com/mcguinness/draft-mcguinness-oauth-actor-profile/issues/3))
identifies the delegation flow.  Bounds verification operates within
a single delegation flow; a `reauthorized` event with a new `acti`
value would indicate a new delegation flow, not a re-auth within the
existing one.  This profile assumes `acti` stability across the
chain and treats `acti` change as out-of-scope (the chain itself has
changed identity).

### 13.6 Mission-Bound OAuth

[Mission-Bound OAuth](https://notes.karlmcguinness.com/series/mission-bound-oauth/)
defines a durable, AS-resident authority object (the Mission) from
which delegated tokens derive.  Mission and bounds address
complementary aspects of authority governance:

- **Mission** is the *control-plane* object: a governed authority
  record with lifecycle states (`active`, `suspended`, `completed`,
  `revoked`, `expired`), independent of any token, terminable by
  business event.  Tokens carry `mission_ref` as a projected
  reference; the Mission Authority Model is the AS's machine-
  evaluable form, attenuated from a proposal through approval.
- **Bounds** is the *wire-format* attestation: per-hop record of
  authority shape on receipts, monotonicity enforced across hops,
  offline-verifiable against the receipt chain.

The composition pattern:

- The origin receipt's `bounds` projects the Mission Authority Model
  into the chain.  Subsequent receipts attest narrowing from that
  origin.  A recipient verifying the chain learns both *what authority
  was approved* (via `mission_ref` and Mission introspection if
  available) and *that derivation never widened it* (via the bounds
  chain, offline).
- Mission lifecycle events that legitimately broaden authority (for
  example, an approver extending a Mission's scope after a request
  from the agent) appear in the bounds chain in one of two places
  depending on whether a new actor hop was introduced:
  - When the broadening is issued *at* a new hop, a `reauthorized`
    entry on that new receipt with `method: mission_lifecycle`
    (Section 5.2) and `artifact` referencing the Mission event record.
  - When the broadening occurs *between* hops (no new actor hop;
    for example, an approver extends a Mission's scope while an
    existing access token is still in use), a `bounds_reauth` event
    in `bounds_events` (Section 5.3) with `method: mission_lifecycle`,
    anchored to the receipt at the hop the change applies to and
    referencing the Mission event record.
- A Mission's bounded delegation rules constrain who may extend the
  chain.  Bounds is the wire-level evidence that delegation respected
  those rules.  An AS that issues a derived token outside its
  Mission's delegation bounds produces a chain that fails bounds
  verification at the recipient.

The relationship inverts at the lifecycle layer: bounds attests
authority shape but does not assert whether the Mission is still
active.  Recipients needing freshness on Mission state perform a
separate check (introspection against the Mission AS, Shared Signals
events, or a similar mechanism); bounds verification is offline and
historical.

Bounds does not require Mission.  A deployment without Mission state
still benefits from monotonicity enforcement at exchange time.
Mission deployments are a target consumer of bounds, not its
prerequisite.

### 13.7 Non-hop event artifacts (receipts §Extensibility)

This profile instantiates the non-hop event artifact pattern defined
in the receipts spec's §Extensibility with `bounds_events`
(Section 5.3) carrying `bounds-event+jwt` events.  Other companions
MAY define their own event arrays under the same pattern (for
example, future receiver-acknowledgment or sender-constraint-rotation
artifacts).  Each companion's events occupy a parallel outer-token
claim and follow the same anchoring rules (by `receipt_jti` or by
delegation correlator).  This profile's verification rules
(Section 7.5) apply only to its own `bounds_events`; companion
events are evaluated under their own consumer-processing rules.

## 14. Why bounds is a separate companion

A natural design question is whether monotonicity should be a
property of the core profile or of receipts rather than a standalone
companion.  This sketch keeps it separate.  Rationale:

- **Different adoption gate.** The receipts spec is OPTIONAL and many
  deployments will adopt it without needing monotonicity enforcement.
  Bundling monotonicity into receipts would either force every
  receipts deployment to also implement subset semantics (which is
  the substantive design work here) or weaken the receipts spec's
  guarantees by making it conditional.
- **Subset semantics are the hard problem.** The substantive design
  is Section 4.  Whether to define string-set vs. semantic subset
  for `scope`, refinement rules for RAR types, URI canonicalization
  for `resource`: each of these is a design question on its own.
  Keeping them in a focused companion lets WG discussion converge
  without entangling the receipts spec.
- **Threat model is distinct.** Receipts mitigate fabrication of
  past hop state.  Bounds mitigate authority expansion at the
  current hop.  These are different adversary classes; keeping the
  specs separate keeps the threat models legible.
- **Composability is the right model.** Receipts-only, issuer-flag-
  only, receipts-plus-bounds, full belt-and-suspenders: each is a
  deployment choice with a different assurance level.  Layered
  companions express these cleanly; a bundled spec would force every
  reader to reason about all the combinations even if they only need
  one.

The principal argument *for* unification with the core profile is a
SHOULD-not-expand rule: even without per-hop attestation, the core
profile could RECOMMEND that delegated-token reissuance and exchange
not expand governed dimensions.  That's a non-normative SHOULD that
could live in the core profile alongside the verifiable companion
defined here.  The two are not in conflict.

## 15. Acknowledgements

This profile is motivated in part by the
[PIC Model](https://github.com/pic-protocol/pic-spec) (Provenance
Identity Continuity), which enforces authority monotonicity
(`ops_i ⊆ ops_{i-1}`) as a structural invariant.  PIC operates at a
different abstraction level (execution model, not JWT profile) and
with a different architecture (online CAT validation rather than
offline receipt verification), but its formal property of authority
non-expansion across hops is the gap this profile addresses for
OAuth-native deployments.  This profile does not depend on or
require PIC; the property is valuable independent of PIC's broader
model.

This profile is also motivated by
[Mission-Bound OAuth](https://notes.karlmcguinness.com/series/mission-bound-oauth/),
which defines a durable AS-resident authority object from which
delegated tokens derive.  Mission addresses the control-plane
question of *what authority was approved and is it still in force*;
bounds addresses the wire-format question of *did the chain preserve
that authority across hops*.  The two are designed to compose;
Section 13.6 describes the composition pattern.  Mission deployments
are a target consumer of bounds without being a prerequisite.

## See Also

- [draft-mcguinness-oauth-actor-profile](../draft-mcguinness-oauth-actor-profile.md)
 : core profile this builds on.
- [draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md)
 : receipts companion this extends.
- [Actor-signed hop proofs](./draft-mcguinness-oauth-actor-proofs.md)
 : sibling companion adding actor-side non-repudiation.
- [Transparency JWT-native variant](./draft-mcguinness-oauth-actor-receipts-transparency.md)
- [Transparency SCITT-aligned variant](./draft-mcguinness-oauth-actor-receipts-scitt.md)
- [RFC 6749 (OAuth 2.0)](https://www.rfc-editor.org/rfc/rfc6749)
- [RFC 7519 (JWT)](https://www.rfc-editor.org/rfc/rfc7519)
- [RFC 8693 (OAuth Token Exchange)](https://www.rfc-editor.org/rfc/rfc8693)
- [RFC 8707 (Resource Indicators)](https://www.rfc-editor.org/rfc/rfc8707)
- [RFC 9396 (Rich Authorization Requests)](https://www.rfc-editor.org/rfc/rfc9396)
- [PIC Model Specification](https://github.com/pic-protocol/pic-spec)
 : related model-layer work on authority monotonicity in distributed
  execution.
- [Mission-Bound OAuth](https://notes.karlmcguinness.com/series/mission-bound-oauth/)
 : control-plane work on durable AS-resident authority objects;
  bounds is a wire-format primitive this architecture can consume.

# OAuth Actor Capabilities (Waffles-Derived OAuth Profile)

> **Status:** Early-stage architectural sketch.  Not a published I-D.
> OAuth Actor Profile integration derived from
> [draft-hingnikar-cecchetti-wimse-waffles](https://clawdrey.com/drafts/waffles/)
> for the OAuth Actor Profile family.  Alternative architecture to
> the [receipts](../draft-mcguinness-oauth-actor-receipts.md) +
> [proofs](../draft-mcguinness-oauth-actor-proofs.md) +
> [authority-bounds](../draft-mcguinness-oauth-actor-authority-bounds.md)
> companion stack.  See [README](./README.md) for family context.
> Last updated: 2026-05-25

## Abstract

This sketch defines a Waffles-derived OAuth profile for use with
the OAuth Actor Profile family.  It reuses Waffles'
({{I-D.hingnikar-cecchetti-wimse-waffles}}) per-layer JWT shape,
JOSE hash binding, depth marker, issuer constraint, and monotonic
attenuation rules, but replaces Waffles' native Stack carriage and
presentation proof with OAuth access-token carriage and OAuth
sender-constrained-token validation.  This profile carries the
capability stack inline in an OAuth access token alongside the
visible `act` chain, constrains the root layer's signer to a
principal-authoritative party for the subject, aligns each layer
with the corresponding visible `act` object, adds resource-
indicator attenuation for OAuth deployments, and defines Token
Exchange and refresh-token integration.

Within the actor-profile family, this profile is an alternative to
the receipts + proofs + bounds companion stack rather than a
companion to it.  A token carrying `actor_caps` does not carry
`actor_receipts`, `actor_proofs`, or `bounds`.  The two
architectures are alternative answers to the same question of how
to attest a delegation chain in transit.

## 1. Introduction

### 1.1 Why a separate profile

The receipts, proofs, and authority-bounds companions each address
a distinct adversary class but overlap structurally: each carries a
per-hop signed artifact and verifies a per-hop alignment against the
visible `act` chain of the outer token.  Their composition
introduces cross-companion alignment by `jti`, `prh`-chain
coordination, and sparse-coverage rules that braid the three
threat models together.

The bet of this profile is that one artifact, signed by the right
party, can carry the same evidence.  Waffles already defines that
artifact: a JWT signed by the delegator at the hop, hash-bound to
its parent, monotonically attenuating authority forward.  This
profile applies the Waffles mechanism to the OAuth Actor Profile
family and adds the integration that the actor-profile family
requires: outer-token wrapping, `act`-chain alignment, principal-
authoritative trust root, Token Exchange flow, and refresh-token
behavior.

### 1.2 Relationship to Waffles

The relationship is derived-from, not strict profile-of.  Each
per-layer JWT in `actor_caps` uses the Waffles layer structure and
registered header parameters, but this document defines a distinct
OAuth carriage and verification profile.

This profile inherits these Waffles mechanics:

- compact JWS layer format with `typ` value `waffle+jwt`;
- `wfl_parent` hash binding over the parent layer's exact compact
  serialization (the chain-linkage role that `prh` plays in the
  receipts spec; this profile uses `wfl_parent` and does not carry
  `prh`);
- `wfl_dep` depth signaling;
- `wfl_iss` child-issuer constraint;
- common `sub` and `aud` across the chain;
- monotonic attenuation of `scope` and `authorization_details`;
- per-layer `iat`, `exp`, `jti`, and `cnf` claims (with `cnf`
  broadened per the override below).

This profile overrides these Waffles mechanics:

- Stack carriage is the `actor_caps` JSON array inside an OAuth
  access token, not an ampersand-joined Authorization header value.
- Presentation proof is the OAuth sender-constrained access-token
  proof over the outer token, not Waffles' Stack-level DPoP proof or
  `ath` calculation over the Stack string.
- Sender constraint MAY use DPoP or mutual TLS; `cnf` is
  consequently broadened from Waffles' `jkt`-only model to also
  accept the certificate confirmation method of
  {{RFC7800, Section 3.3}}.  Confirmation method matches between
  the tip cap and the outer token.
- Key-to-actor identity binding is defined by actor-profile metadata
  in this document, not solely by Waffles issuer metadata.
- Issuance, extension, reissuance, refresh, and error handling are
  OAuth-specific.

This profile adds beyond Waffles:

- Monotonic attenuation of an additional authority dimension,
  `resource` ({{RFC8707}}), with per-dimension semantics defined in
  {{cap-jwt-format}}.
- A principal-authoritative root-cap signer constraint that ties the
  root cap to the subject's identity provider (or equivalent
  principal-authoritative party).
- An `act`-chain alignment that maps each cap's `cnf` to a specific
  visible actor.

Where this document explicitly invokes a Waffles requirement, that
requirement applies.  Otherwise, this document's OAuth-specific
processing rules govern.

### 1.3 What this profile trades

Authorization servers lose their role as the trust mediator of
prior-hop attestation: the artifact does not say "the authorization
server validated this," it says "the delegator authorized this."
Deployments that rely on per-hop authorization-server attestation
(for example, environments where the actor cannot hold a signing
key but the authorization server can) are better served by the
receipts companion.

## 2. Terminology

This document uses the terminology of the core actor profile
{{I-D.mcguinness-oauth-actor-profile}} (including `hop` and
`visible hop`) and of Waffles
{{I-D.hingnikar-cecchetti-wimse-waffles}} (including `Layer`,
`Stack`, `Root Waffle`).  In addition:

- **Actor capability layer (cap):** a Waffles Layer profiled by
  this document.  Used as a shorthand for "the per-hop JWT this
  profile carries."
- **Cap chain:** a Waffles Stack profiled by this document.
  Carried as `actor_caps`.
- **Delegator for cap *i*:** the entity that signed `actor_caps[i]`.
  For the root cap, this is a principal-authoritative party for
  the outer token's `sub` (see {{principal-authoritative}}).  For
  every other cap, this is the entity bound by the parent cap's
  `cnf` (the entity that the parent cap delegated to).
- **Delegatee for cap *i*:** the entity bound by `actor_caps[i].cnf`.
  The current presenter of the outer token is the delegatee for the
  tip cap (`actor_caps[0]`).
- **Root cap:** the innermost (oldest) capability layer in the
  chain; the Waffles Root Waffle.  In this profile, signed by the
  principal-authoritative key for `sub`.
- **Tip cap:** the outermost (newest) capability layer in the
  chain; corresponds to Waffles' `L_0`.  When the chain has depth
  1, the tip cap and root cap are the same layer.
- **Principal-authoritative key:** a signing key whose recipient
  trust scope includes asserting delegations on behalf of a subject
  namespace.  Typically an identity provider's signing key.
- **Outer-token issuer:** the authorization server that signs the
  outer token wrapping the cap chain.  Not a member of the cap
  chain.

## 3. Architecture

### 3.1 The bet

The receipts spec's primary value proposition is that a compromised
downstream issuer cannot fabricate prior-hop provenance, because it
cannot forge prior issuers' signatures.  The proofs sketch adds
that a compromised current issuer cannot fabricate actor
participation, because actor signatures are independently
verifiable.  The bounds sketch adds that no intermediate issuer can
silently widen authority.

Waffles' layered-JWT design with hash-binding and monotonic
attenuation makes all three properties fall out of a single
signature chain:

- The delegator's signature on each layer directly attests prior-
  hop provenance.  No downstream party can forge it.
- Actor non-repudiation is a property of the same artifact, because
  the delegator for a non-root layer is the delegatee named by the
  parent layer.
- Attenuation is structural: a child layer's `scope` and
  `authorization_details` must be subsets/refinements of its
  parent's, and this profile applies the same structural treatment
  to `resource`, enforced by hash-chained signatures.

This profile adapts that mechanism to the actor-profile family by
wrapping the Stack in an outer OAuth access token whose `act` chain
provides the visible-identity surface, and by constraining the root
Layer's signer to a principal-authoritative party for `sub`.

### 3.2 Chain layout

The cap chain mirrors the visible `act` chain.  A token with `n`
visible actor hops carries `actor_caps` with `n` capability JWTs,
ordered newest hop to oldest hop (the same convention as Waffles'
`L_0..L_n` and as the receipts spec's `actor_receipts`).

- `actor_caps[0]` is the tip cap.  It corresponds to the outermost
  `act` object and was signed by the delegator for the tip cap.
- `actor_caps[n-1]` is the root cap.  It corresponds to the
  innermost `act` object and was signed by the principal-
  authoritative key for `sub`.

Each non-root cap is hash-bound to its parent via Waffles' header
parameter `wfl_parent`.  Depth is recorded in Waffles' `wfl_dep`
header parameter, strictly decreasing from parent to child.

### 3.3 Relationship to the outer token

The outer token's `act` chain provides identity (who is at each
hop).  The cap chain provides authority (what was granted at each
hop) and is signed by the delegator at each hop.  The outer token's
top-level `cnf` binds the current presenter to a key, as in the
base actor profile.

Two binding properties tie the outer token to the cap chain:

- **cnf binding.**  The tip cap's `cnf` MUST be consistent with the
  outer token's top-level `cnf` under the sender-constraining proof
  mechanism in effect for the outer token.  This guarantees the
  current presenter is the delegatee the tip cap authorizes.
- **Authority binding.**  The outer token's effective authority
  (`scope`, `aud`, `resource`, `authorization_details`) MUST be
  consistent with the cap chain per
  {{outer-token-authority-binding}}.

The outer-token issuer's role is to validate the inbound cap chain,
optionally narrow runtime authority within tip-cap bounds, and emit
an outer token wrapping the validated chain.  The outer-token
issuer is not a member of the cap chain and its signature does not
extend the chain.

## 4. The `actor_caps` Claim {#actor-caps-claim}

`actor_caps` is a new top-level JWT claim for tokens that conform
to the core actor profile and this companion.

`actor_caps`:
: OPTIONAL.  An array of strings.  Each string MUST be the compact
  serialization of a Waffles Layer (a signed JWT with `typ`
  `waffle+jwt`).  When present, the array:

  - MUST NOT be empty;
  - MUST be ordered from newest hop (`actor_caps[0]`) to oldest hop
    (`actor_caps[n-1]`);
  - MUST have length equal to the visible actor-chain depth of the
    outer token's `act` claim (complete coverage; partial coverage
    is not defined by this profile);
  - MUST satisfy Waffles' hash-chain requirements: each non-root
    cap's `wfl_parent` header equals the base64url-encoded SHA-256
    of the next-inner cap's compact serialization, and the root cap
    omits `wfl_parent`.

If a token carries `actor_caps`, it MUST also carry an `act` claim
conforming to the core actor profile, and MUST NOT carry
`actor_receipts`, `actor_proofs`, or `bounds` claims from the
receipts, proofs, or authority-bounds companions.  See
{{coexistence}}.

The `actor_caps` array length is bounded above by the configured
maximum delegation depth defined in the core actor profile
{{I-D.mcguinness-oauth-actor-profile}} and by Waffles'
recommended-default maximum Stack depth of 10
({{I-D.hingnikar-cecchetti-wimse-waffles}} §7.3).  A recipient
whose configured maximum delegation depth is less than the array
length MUST reject the token.

This profile carries the stack as a JSON array inside the outer
access token rather than as the ampersand-joined string in the
Authorization header defined by
{{I-D.hingnikar-cecchetti-wimse-waffles}} §4.2.  This addresses
Waffles Open Issue #3 by taking the JSON-array position for this
profile's deployments; the underlying per-layer format is
unchanged.

## 5. Capability JWT Format {#cap-jwt-format}

Each cap uses the Waffles Layer JWT structure.  The requirements in
{{I-D.hingnikar-cecchetti-wimse-waffles}} §3 apply only as profiled
below; sender presentation, key-to-actor binding, and OAuth
transport are defined by this document.

### 5.1 JOSE Header

Inherited from {{I-D.hingnikar-cecchetti-wimse-waffles}} §3.1:

- `typ`: REQUIRED.  Value MUST be `waffle+jwt`.
- `alg`: REQUIRED.  Asymmetric signing algorithm.  ES256 per
  Waffles; deployments that require other asymmetric algorithms
  MUST follow Waffles' future updates rather than re-profiling
  here.
- `kid`: REQUIRED when the delegator publishes multiple keys.
- `wfl_parent`: REQUIRED for non-root caps, MUST be omitted on the
  root cap.
- `wfl_dep`: OPTIONAL but RECOMMENDED.  When present, this
  profile's recipients SHOULD enforce strict decrease per Waffles.
- `wfl_iss`: OPTIONAL.  Per Waffles, restricts the set of issuer
  identifiers acceptable on child layers.  Useful in deployments
  that want to bound which intermediates may extend the chain.

Inline key carriage (`jwk` or `x5c`):
: OPTIONAL.  For root caps, recipients MUST verify any inline key
  against the configured principal-authoritative key source for the
  outer token's `sub` (see {{principal-authoritative}}).  For non-
  root caps, recipients MUST resolve the signer's verification key
  per Waffles' metadata-based key resolution
  ({{I-D.hingnikar-cecchetti-wimse-waffles}} §5) and MAY use an
  inline key only after that resolution confirms its publication.

### 5.2 Payload Claims

Inherited from {{I-D.hingnikar-cecchetti-wimse-waffles}} §3.2:

- `iss`: REQUIRED.  The identifier of the delegator that signed
  this cap.  For the root cap, MUST identify a principal-
  authoritative party for the outer token's `sub` (see
  {{principal-authoritative}}).  For non-root caps, MUST identify
  the entity bound by the parent cap's `cnf` (the entity the parent
  delegated to).
- `sub`: REQUIRED.  Per Waffles, the subject the chain is acting on
  behalf of, identical at every layer.  In this profile, `sub` MUST
  equal the outer token's top-level `sub`.  Delegatee identity at
  each hop is NOT carried in `sub` (it is carried by alignment with
  the visible `act` chain; see {{act-alignment}}).
- `aud`: REQUIRED.  Per Waffles, the audience this chain may target,
  identical at every layer.  In this profile, `aud` MUST equal the
  outer token's `aud`.  This profile does not define per-cap
  audience attenuation; audience is a property of the chain as a
  whole, fixed at the root.
- `iat`, `exp`, `jti`: per Waffles.
- `cnf`: REQUIRED.  Waffles defines `cnf` with a `jkt` member (JWK
  thumbprint of a DPoP key).  This profile broadens `cnf` to also
  permit the certificate confirmation method of
  {{RFC7800, Section 3.3}} when sender constraint is mutual TLS
  (see {{cap-jwt-format}} §5.3).  The confirmation method used on
  the tip cap MUST match the confirmation method used on the outer
  token.
- `scope`: OPTIONAL.  Per Waffles, monotonically reducing: every
  scope token on a child layer MUST appear on every ancestor's
  `scope`.
- `authorization_details`: OPTIONAL.  Per Waffles, child elements
  MUST be subsets or refinements of parent elements per
  {{RFC9396, Section 7}}.
- `resource`: OPTIONAL.  An absolute URI string or array of absolute
  URI strings representing the resource indicators {{RFC8707}} the
  cap authorizes.  Per-dimension semantics are:

  - Parent absent, child present: forbidden.  A child cap MUST NOT
    introduce a `resource` constraint that no ancestor recorded;
    adding a dimension absent from the parent is not attenuation.
    A child cap whose parent omits `resource` MUST also omit
    `resource`.
  - Parent present, child present: every URI in the child's
    `resource` MUST be an exact URI-set subset of the parent's
    `resource` after canonicalization per {{RFC3986, Section 6.2}}.
    URI prefix subsumption is not applied by this profile.
  - Parent present, child absent: the child confers no
    `resource`-dimension authority at its hop.

The `resource` claim is an OAuth Actor Capabilities extension to the
base Waffles layer claim set.  It is included so this profile can
replace the authority-bounds companion for deployments that use
resource indicators separately from audience.

#### Actor-profile constraints {#act-alignment}

These constraints apply on top of the Waffles claim definitions.
They define the key-to-actor binding that Waffles does not by
itself provide.

- `actor_caps[i].cnf` MUST identify a confirmation key for the actor
  named by the visible `act` object at depth *i*+1
  (`actor_caps[0]` covers the outermost `act`, `actor_caps[1]`
  covers the next-inner `act`, and so on).
- Recipients MUST verify the `act.sub`-to-`cnf` binding through an
  actor-key binding source trusted under local policy.  Acceptable
  sources include authorization-server client metadata, workload
  identity metadata, a DID document, federation metadata, or another
  configured registry that maps the visible actor identifier to
  confirmation keys.
- For DPoP, the binding source MUST associate the actor identifier
  with the JWK thumbprint in `cnf.jkt`.  For mTLS, the binding source
  MUST associate the actor identifier with the certificate binding
  carried in `cnf` (for example, `x5t#S256`).
- For non-root caps, `actor_caps[i].iss` MUST identify the same
  actor whose key is bound by `actor_caps[i+1].cnf`.  The recipient
  verifies this by resolving the parent cap's `cnf` to an actor
  identifier using the same actor-key binding source and comparing
  that identifier to the child cap's `iss`.

If the recipient cannot resolve a cap's `cnf` to the corresponding
visible actor identifier, or cannot resolve a parent cap's `cnf` to
the child cap's `iss`, it MUST reject the token.

### 5.3 Sender Constraint

Waffles requires DPoP at every layer and binds the DPoP `ath` value
to the Waffles Stack presentation.  This profile replaces that
presentation model.  A recipient validates the sender-constrained
OAuth access token and verifies that the outer token's top-level
`cnf` is consistent with the tip cap's `cnf`; it does not require a
separate Waffles Stack-level DPoP proof or an `ath` value over the
`actor_caps` array.

This profile permits DPoP {{RFC9449}} and mutual-TLS sender
constraint {{RFC8705}}.  When DPoP is used, the tip cap's `cnf` and
the outer token's top-level `cnf` MUST carry the same `jkt`.  When
mTLS is used:

- the layer's `cnf` carries the certificate confirmation method
  defined in {{RFC7800, Section 3.3}} rather than `jkt`;
- the outer token's top-level `cnf` MUST use the same confirmation
  method as the tip cap;
- the request to the recipient MUST be presented over a TLS
  connection bound by the certificate the tip cap's `cnf`
  identifies.

This addresses Waffles Open Issue #1 by taking the "mTLS also
acceptable" position for this profile's deployments.

## 6. Issuer Processing

### 6.1 Creating the root cap

When the principal-authoritative party (typically the identity
provider for `sub`) authorizes the first actor, it creates a root
cap (a Waffles Root Waffle) with `wfl_parent` omitted, `iss` equal
to its own identifier, `sub` equal to the subject, and `cnf` set to
the first actor's confirmation key.  The root cap's `scope`,
`resource`, and `authorization_details`, if present, define the
maximum authority the chain may convey.

The root cap MAY carry `wfl_iss` to restrict which actor
identifiers may sign the immediate child cap.  Per Waffles, a child
cap's own `wfl_iss` MUST be a subset of its parent's `wfl_iss` (if
the parent carries one), so further-downstream constraint follows
by propagation rather than by direct reach from the root.  A
principal-authoritative party that wants to forbid unconstrained
onward delegation SHOULD include `wfl_iss` naming the permitted
immediate-child delegators.  If `wfl_iss` is omitted, any child cap
whose signer satisfies the parent-`cnf` binding and actor-profile
alignment rules can extend the chain.

The principal-authoritative party MAY require interactive consent,
authentication assurance, or other policy-defined preconditions
before issuing a root cap.  Recording the preconditions that led
to root-cap issuance is deployment-specific; this profile does not
standardize a "consent record" structure.

### 6.2 Extending the chain {#extending-chain}

When the current outermost actor sub-delegates to a new actor, the
current actor signs a child cap that:

- carries `wfl_parent` set per Waffles to the SHA-256 of the parent
  cap's compact serialization;
- carries `iss` equal to its own identifier (the delegating actor);
- carries `sub` equal to the parent cap's `sub` (the principal);
- carries `aud` equal to the parent cap's `aud`;
- carries `cnf` set to the new delegatee's confirmation key;
- carries `scope` that is a subset of the parent's `scope` and
  `authorization_details` that is a refinement of the parent's,
  per Waffles;
- carries `resource`, when present, as a subset of the parent's
  `resource` per {{cap-jwt-format}};
- carries `iat` no earlier than the parent cap's `iat`;
- carries `exp` no later than the parent cap's `exp`.

The delegating actor (or the new delegatee acting on the delegating
actor's behalf) then presents the child cap to an authorization
server that will issue an outer token bound to the new delegatee's
key.  The authorization server validates the chain (see
{{consumer-processing}}), prepends the new cap to `actor_caps`, and
issues an outer token whose `act` chain is extended by one hop.

The authorization server MUST NOT reserialize, resign, normalize,
trim, or otherwise alter any inbound cap.  `wfl_parent` is a hash
over the exact compact serialization of the parent cap; any
modification, including ostensibly equivalent re-encoding, breaks
the chain.  The authorization server MUST NOT issue an outer token
whose authority exceeds the bounds the tip cap establishes per
{{outer-token-authority-binding}}.

### 6.3 Reissuance without a new actor hop

An authorization server reissuing or refreshing a token without
adding a new actor hop MUST carry `actor_caps` forward unchanged.
The cap chain is not affected by changes to the outer token's
`scope`, `resource`, or `exp` that remain within tip-cap authority
and within tip-cap expiry.  The outer token's `aud` MUST equal the
tip cap's `aud` (since `aud` is identical across the chain).  The
outer token's top-level `cnf` MUST remain consistent with the tip
cap's `cnf`; rotating the presenter key requires a new cap from the
tip cap's delegator.

The outer token's `sub`, `act` chain, and any actor-identity claims
MUST NOT change under reissuance without a new hop.

### 6.4 Re-authorization {#re-authorization}

A party with principal-authoritative scope (typically the identity
provider) widening authority for a subject creates a new root cap.
Subsequent re-delegation builds a new chain.  Re-authorization is
not represented as a mid-chain widening event; the new chain
supersedes the prior chain at the principal level.  Mid-chain
widening is not supported (Waffles already forbids it through
monotonic attenuation).

### 6.5 Token Exchange flow integration {#token-exchange}

This profile composes with OAuth Token Exchange {{RFC8693}} for the
extension flow defined in {{extending-chain}}, complementing the
Token-Exchange-to-Waffles relationship in
{{I-D.hingnikar-cecchetti-wimse-waffles}} §6.2.

When an actor sub-delegates to a new actor and seeks an outer token
for the new delegatee from an authorization server that supports
this profile:

1. The delegating actor signs a child cap as in
   {{extending-chain}}.  The delegating actor MUST hold or obtain
   a valid inbound outer token whose tip cap names it as delegatee.
2. The new delegatee (or the delegating actor acting on the new
   delegatee's behalf) sends a Token Exchange request to the
   authorization server containing:
   - `subject_token`: the inbound outer token (a token issued under
     this profile, carrying `actor_caps`);
   - `subject_token_type`: the registered type identifier of that
     token;
   - `actor_token`: the compact serialization of the new child cap
     itself;
   - `actor_token_type`:
     `urn:ietf:params:oauth:token-type:actor-cap`
     (registered per {{iana-token-types}});
   - other Token Exchange parameters as defined in {{RFC8693}}.
3. The authorization server validates the inbound `subject_token`
   per {{consumer-processing}}.  Validation includes verifying that
   the new child cap's `wfl_parent` hash equals the hash of the
   inbound tip cap, that the new child cap's authority is
   monotonically reduced from the inbound tip cap per Waffles, and
   that the new child cap's `iss` corresponds to the entity bound
   by the inbound tip cap's `cnf`.
4. The authorization server's chain validation in step 3 substitutes
   for the actor-token trust check that {{RFC8693}} would otherwise
   apply: the authorization server does not need to independently
   trust the child cap's `iss` as an ordinary identity-assertion
   issuer, because the cap chain is the assertion and is verified
   structurally.  This substitution is profile-specific; an
   authorization server MAY additionally apply the {{RFC8693}}
   actor-token trust check alongside chain validation when local
   policy requires both signals.
5. The authorization server issues a new outer token whose `act`
   chain is extended by one hop and whose `actor_caps` is
   `[new_cap, ...inbound_actor_caps]`, where `inbound_actor_caps`
   is the validated `actor_caps` array read from the `subject_token`
   outer JWT.

An authorization server that does not support this profile MUST
reject the request with `unsupported_token_type` for the
`actor_token_type`.

### 6.6 Refresh tokens

A refresh token issued for an outer token carrying `actor_caps` is
subject to the same constraints as the outer token:

- The refresh exchange MUST carry the same `actor_caps` array
  forward unchanged.
- The refreshed outer token MUST have `exp` no later than the
  smallest `exp` in the cap chain.
- The refreshed outer token MUST NOT widen authority beyond what
  the tip cap permits.
- The refreshed outer token's `cnf` MUST remain consistent with the
  tip cap's `cnf`.

An authorization server that supports refresh for cap-bearing tokens
MUST persist the `actor_caps` array in issuer-controlled storage
(token-state database, refresh-token state, or equivalent) so that
each refreshed access token can carry the chain forward unchanged.

When the tip cap's `exp` is reached, refresh MUST fail.  The
delegating actor is responsible for issuing a fresh child cap if
continued delegation is intended; the principal-authoritative party
is responsible for issuing a fresh root cap if the entire chain has
expired.

## 7. Consumer Processing {#consumer-processing}

For tokens carrying `actor_caps`, the recipient performs the
following steps.  Steps are performed in order; failure at any step
MUST cause the recipient to reject the token.

### 7.1 Outer-token validation

1. Validate the outer token per the core actor profile, including
   signature, `iss`, `exp`, `iat`, `aud`, `act` chain conformance,
   and sender-constraint proof against the outer token's top-level
   `cnf`.

### 7.2 Structural validation of `actor_caps`

2. Verify that `actor_caps` is present.  If absent, apply local
   policy or Protected Resource Metadata signals (see
   {{discovery}}) to decide whether the token is acceptable without
   cap-based provenance.
3. Verify that `actor_caps` is a non-empty JSON array of strings.
4. Verify that the array length equals the visible actor-chain
   depth of the outer token's `act` claim and does not exceed the
   recipient's configured maximum delegation depth.
5. Parse each cap JWT and verify that its `typ` header equals
   `waffle+jwt`.  Reject the token if any cap fails to parse.

### 7.3 Chain integrity {#consumer-chain-integrity}

6. Apply Waffles' chain-integrity verification per
   {{I-D.hingnikar-cecchetti-wimse-waffles}} §4.3, with the
   following step-by-step status under this profile:
   - Waffles §4.3 step 1 (split the `Authorization: DPoP <Stack>`
     header): REPLACED.  The Stack is read from the outer token's
     `actor_caps` claim.
   - Waffles §4.3 step 2 (split the Stack string on `&` to obtain
     ordered layer serializations): REPLACED.  The ordered layer
     serializations are the JSON-array elements of `actor_caps`.
     Depth limits from this profile and from the core actor profile
     apply per §4.
   - Waffles §4.3 step 3 (`wfl_parent` hash chain): INHERITED.  For
     each adjacent pair `(actor_caps[i], actor_caps[i+1])`, verify
     that `actor_caps[i].wfl_parent` equals the base64url-encoded
     SHA-256 of `actor_caps[i+1]`'s compact serialization.
   - Waffles §4.3 step 4 (per-layer signature, claims, and header
     constraints): INHERITED.  For each cap, resolve the signing
     key per Waffles' metadata rules and validate the JWS
     signature.  Verify Waffles' per-claim constraints, including
     monotonic `scope` reduction, `authorization_details`
     subset/refinement, identical `sub`, identical `aud`, and
     `iat`/`exp` adjacency rules between adjacent layers.  Verify
     `wfl_dep` (if present) is strictly decreasing across adjacent
     layers and is non-negative.
   - Profile addition: verify monotonic `resource` reduction per
     {{cap-jwt-format}} when any cap carries `resource`.
   - Waffles §4.3 step 5 (DPoP proof against `L_0.cnf.jkt` with
     `ath` over the Stack string): REPLACED.  The presenter binding
     is validated by this profile's outer-token sender-constraint
     proof in §7.1 step 1 and by the tip-to-outer `cnf` consistency
     check in §7.5 step 10.
   - Waffles §4.3 step 6 (local authorization policy applied across
     the attenuated claim set): INHERITED.  Applied as part of §7.5
     step 12 and §7.6 step 13.

### 7.4 Actor-profile bindings {#actor-profile-bindings}

7. Verify that the root cap's `iss` identifies a principal-
   authoritative party for the outer token's `sub` per
   {{principal-authoritative}}, and that the root cap's signing
   key is in the recipient's principal-authoritative trust set for
   that subject.
8. Verify the layer-to-`act` alignment defined in {{act-alignment}}:
   for each cap at index *i*, resolve the visible `act` object's
   actor identifier at depth *i*+1 to its authorized confirmation
   keys and verify that `actor_caps[i].cnf` names one of them.  For
   non-root caps, resolve `actor_caps[i+1].cnf` to an actor
   identifier and verify that it equals `actor_caps[i].iss`.
9. Verify that the cap chain's common `sub` equals the outer
   token's top-level `sub`.

### 7.5 Tip-to-outer binding {#outer-token-authority-binding}

10. Verify the tip cap's `cnf` is consistent with the outer token's
    top-level `cnf` according to the token's sender-constraining
    proof mechanism (DPoP {{RFC9449}}, mTLS {{RFC8705}}, or other
    permitted per {{cap-jwt-format}} §5.3).
11. Verify the outer token's `aud` equals the cap chain's common
    `aud`.
12. Verify the outer token's effective `scope` is a subset of the
    tip cap's `scope`, and that the outer token's effective
    `authorization_details` is a subset/refinement of the tip
    cap's.  When the outer token carries or implies effective
    `resource` indicators, verify that they are a subset of the tip
    cap's `resource`.

### 7.6 Extension processing

13. Apply any extension rules whose claims appear in any cap or in
    the outer token.  Because this profile is mutually exclusive
    with `actor_receipts`, `actor_proofs`, and `bounds`
    ({{coexistence}}), extension rules here include only:

    - Waffles updates that add or refine claim semantics on layers;
    - selective-disclosure variants applied to caps per §11 (out of
      scope for this revision but anticipated);
    - additional authority dimensions defined by future revisions
      of this profile, with subset-or-refinement semantics
      registered alongside the dimension.

    A recipient that does not recognize an extension claim treats
    it per JWT's unknown-claim-ignore rules unless an applicable
    specification says otherwise.

If any step fails, the recipient MUST reject the token under the
underlying protocol's error model.  A recipient that has rejected
a token under this profile MUST NOT use cap-derived authority in
any downstream authorization decision for that token.

## 8. Trust Model

### 8.1 Per-principal trust

The recipient's trust set is keyed by principal namespace.  For a
given `sub`, the recipient configures the set of keys authorized
to sign root caps for that subject's namespace.  Typically this is
the subject's identity provider's published JWKS or DID document.

A root cap signed by a key outside the configured principal-
authoritative set for the outer token's `sub` MUST be rejected.

### 8.2 Non-root cap signers

Non-root caps are signed by intermediate actors (delegators).
Their keys are resolved per Waffles' metadata rules
({{I-D.hingnikar-cecchetti-wimse-waffles}} §5).  This profile
additionally enforces, per {{act-alignment}}, that the resolved
signing entity matches the identity bound by the parent cap's
`cnf`.

Actor confirmation keys are resolved through the actor-key binding
source described in {{act-alignment}}.  That source binds visible
actor identifiers from the `act` chain to confirmation keys; it is
separate from the principal-authoritative root-cap trust set and
from ordinary OAuth authorization-server trust.

### 8.3 Outer-token authorization server trust {#outer-token-as-trust}

The outer token is signed by the authorization server that issued
it, and that signature is validated per the base actor profile in
{{consumer-processing}} step 1.  Recipients MUST configure trust for
outer-token issuers under their existing OAuth trust framework
(local trust set, federation metadata, or equivalent).

This trust requirement is independent of the cap chain.  An outer-
token issuer trusted to issue tokens to the current presenter is
not, by virtue of issuing the outer token, in the trust path of
any cap in the chain.  An outer-token issuer cannot fabricate caps
it did not sign and cannot widen authority beyond the tip cap.

A compromised outer-token issuer can issue outer tokens that wrap
valid cap chains but bind them to attacker-controlled presenters
(via top-level `cnf`); this is detected by the tip-to-outer
binding check in {{consumer-processing}} step 10 when the attacker
does not also control the tip-cap delegatee's key, since the tip
cap's `cnf` fixes the legitimate delegatee key.

### 8.4 Comparison to per-issuer trust

The receipts spec requires recipients to trust each receipt-issuing
authorization server in the chain (or to rely on transparency-log
trust).  This profile reduces the trust set to the principal-
authoritative key for each subject the recipient serves plus the
outer-token issuer for each delegation an outer token is issued by.
Authorization servers earlier in the chain (those that issued
inbound tokens that have since been re-exchanged) are not in the
recipient's trust set.

## 9. Discovery {#discovery}

### 9.1 Authorization Server Metadata

- `actor_caps_supported`: OPTIONAL.  Boolean indicating that the
  authorization server can issue outer tokens carrying
  `actor_caps`.
- `actor_caps_signing_alg_values_supported`: OPTIONAL.  Array of
  JWS algorithm identifiers the authorization server supports for
  caps in chains it validates and re-issues.  Must align with the
  algorithm set permitted by Waffles.

### 9.2 Protected Resource Metadata

- `actor_caps_required`: OPTIONAL.  Boolean.  When `true`, the
  resource server requires `actor_caps` on tokens it accepts for
  delegated authorization.
- `actor_caps_principal_authorities`: OPTIONAL.  Array of issuer
  identifiers (URIs) trusted to sign root caps for subjects under
  this resource server's policy.
- `actor_caps_sender_constraints_supported`: OPTIONAL.  Array
  containing one or more of `dpop`, `mtls`.  Indicates which
  sender-constraint mechanisms the resource server accepts under
  this profile (see {{cap-jwt-format}} §5.3).

### 9.3 Coexistence Discovery {#coexistence}

A token MUST NOT carry both `actor_caps` and any of
`actor_receipts`, `actor_proofs`, or `bounds`.  An authorization
server that supports both architectures advertises both via the
respective `_supported` metadata, but MUST emit at most one in any
given token.  Recipients MAY support both and select per token
based on which claim is present.

### 9.4 Introspection {#introspection}

An introspection response ({{RFC7662}}) for a token carrying
`actor_caps` SHOULD include `actor_caps` as a top-level member of
the response, with the same value the outer token carried inline.
Consumer processing in {{consumer-processing}} applies to the
introspection-returned chain identically to inline carriage.

An introspection server MUST NOT modify the cap chain.  An
introspection server that suppresses individual caps for privacy
or policy reasons MUST omit `actor_caps` entirely rather than
return a shortened chain; partial coverage is not defined in this
profile.

## 10. Security Considerations

This section enumerates threats this profile addresses or
acknowledges.  Underlying Waffles security considerations
({{I-D.hingnikar-cecchetti-wimse-waffles}} §7) apply.

### 10.1 Delegator key compromise

If a delegator's signing key is compromised, the attacker can mint
arbitrary sub-caps as that delegator.  The attacker's sub-caps are
constrained to the authority the delegator already held; the
attacker cannot widen.  Mitigation: rotate the compromised
delegator's keys and propagate the rotation through Waffles'
metadata-based key resolution.  Recipients MAY require short cap
`exp` to bound the exposure window.

### 10.2 Principal-authoritative key compromise {#principal-authoritative-key-compromise}

If a principal-authoritative key for `sub` is compromised, the
attacker can mint arbitrary root caps for that subject.  This is
the same exposure as compromise of an identity provider's signing
key in the base actor profile, and is mitigated by the same
mechanisms (key rotation, short cap `exp`, transparency monitoring
of root caps).

### 10.3 Truncation

A truncated chain (`actor_caps` shorter than the visible `act`
depth) is rejected at {{consumer-processing}} step 4.  An attacker
dropping an inner cap also drops the `wfl_parent` hash linkage;
the resulting gap fails Waffles' verification step in
{{consumer-chain-integrity}}.

### 10.4 Reordering

`wfl_parent` hash linkage prevents reordering: each non-root cap
fixes its parent's exact serialization.  Reordering inner caps
breaks the hash chain.

### 10.5 Transplantation

Caps are bound to delegatees via `cnf` and to outer-token positions
via layer-to-`act` alignment ({{act-alignment}}).  A cap minted for
one delegatee cannot be replayed for another delegatee without
matching `cnf`, and cannot be placed at a different depth without
matching the visible `act` object at that depth.

### 10.6 Revocation {#cap-revocation}

Each non-root cap inherits its parent's revocation state through
the chain.  A delegator that revokes a sub-delegation does so by
publishing revocation state keyed by cap `jti`, by requiring
introspection for cap-bearing tokens, or by using short cap
lifetimes.  Rotating or removing a key prevents future signatures
with that key from validating where key resolution depends on a
live key source, but it does not by itself revoke already-issued
caps whose signer key or delegatee key remains verifiable from
immutable chain material.

This profile does not standardize a revocation list format.  Two
deployment patterns are interoperable:

- **Short-`exp` revocation.**  Caps carry short `exp` (minutes to
  hours).  Revocation is implicit through expiration.  Continued
  delegation requires periodic re-signing by the delegator.
- **Active-revocation introspection.**  Recipients introspect outer
  tokens ({{introspection}}); the introspection server consults a
  per-delegator revocation list keyed by cap `jti` and returns the
  token as `active: false` when any cap in the chain is revoked.

Deployments combining the two patterns SHOULD document which
recipients use which mode so that delegator behavior matches
recipient expectations.

### 10.7 Cross-audience replay

Because Waffles requires `aud` identical across all layers, a cap
chain cannot be replayed to a different audience than the one the
root cap committed to.  This stronger guarantee is inherited from
Waffles' chain-wide `aud` requirement; the receipts spec's threat
model treats `aud` as a per-hop AS assertion only, without a
chain-wide invariant.  Recipients MUST verify the outer token's
`aud` equals the cap chain's common `aud` per
{{consumer-processing}} step 11.

### 10.8 cnf rebinding mid-chain {#cnf-rebinding-mid-chain}

The cap chain binds each delegatee key in the corresponding cap's
`cnf`.  An attacker who controls hop *i*'s delegatee key but not
the parent-cap signing key cannot rebind subsequent caps because
`wfl_parent` fixes the parent's exact serialization, including its
`cnf`.

When a delegatee legitimately rotates keys mid-flow (for example,
short-lived workload identity rotation), the rotation is reflected
by a fresh cap from the delegator binding the new key.  This
profile does not standardize a key-rotation proof short of
re-signing.

### 10.9 Token Exchange forgery

A request submitted to an authorization server under
{{token-exchange}} that carries an attacker-fabricated
`actor_token` is detected when the authorization server validates
the child cap's `wfl_parent` hash and `iss`-to-parent-`cnf`
linkage against the inbound `subject_token`'s tip cap.  An
authorization server implementing this profile MUST perform that
validation before prepending the new cap to `actor_caps`.

### 10.10 Out-of-scope adversaries

- **Compromised principal-authoritative key.**  Inherited from the
  identity provider's trust footprint.
- **Collusion across all delegators in a chain.**  As with
  receipts, full-chain collusion is indistinguishable from
  legitimate delegation at the protocol level.
- **Compromised current presenter cnf key.**  Defended by the
  underlying sender-constraint mechanism (DPoP or mTLS) and short
  token lifetime; not unique to this profile.

## 11. Privacy Considerations

A cap chain discloses the same delegation graph that the visible
`act` chain discloses (subject, intermediate actors, current
delegatee).  Unlike the `act` chain, however, each hop is
independently signed and tamper-evident; cap chains are therefore a
stronger fingerprinting surface than `act` alone.  An observer with
access to multiple outer tokens for the same subject can correlate
cap-chain prefixes (root cap `jti`, root cap `cnf`) across
deployments to track that subject's delegation behavior over time.

The privacy considerations of the core actor profile
{{I-D.mcguinness-oauth-actor-profile}} and of Waffles
{{I-D.hingnikar-cecchetti-wimse-waffles}} apply.  In addition:

- Issuers SHOULD treat cap chains as personal data when the
  subject's `sub` or any actor identifier is personally
  identifying.
- Recipients that log cap chains for audit purposes SHOULD log
  them with the same access controls applied to other delegated-
  identity artifacts.
- Delegators SHOULD avoid placing personally identifying material
  in `authorization_details` beyond what the underlying
  authorization-details `type` requires.
- An introspection server that exposes `actor_caps` in
  introspection responses extends the cap chain's reach to every
  party that can introspect a given token; introspection access
  policy SHOULD account for this.

Selective-disclosure considerations from Waffles §7.7 (combining
Waffles with SD-JWT) apply equally to this profile and are deferred
to future work.

## 12. Comparison to the Four-Companion Stack {#comparison}

### 12.1 What `actor_caps` subsumes

- **Receipts.**  Replaced.  A cap is signed by the delegator (the
  party with delegated authority), not by the authorization server
  that minted the current token.  Recipients trust principal-
  authoritative parties rather than per-issuer authorization
  servers.  The receipts spec's "compromised downstream issuer
  fabricating prior-hop provenance" adversary is defeated by
  Waffles' `wfl_parent` + signature mechanism: a compromised
  authorization server downstream cannot forge a delegator's
  signature.
- **Proofs.**  Replaced.  Actor non-repudiation is structural: the
  delegator for a non-root cap is the delegatee named by the
  parent cap, so signing a child cap is signing as the actor that
  already holds delegated authority.  The proofs sketch's
  `receipt_jti` alignment problem (the consistency report's H1
  finding) does not arise because there is no parallel artifact to
  align with.
- **Authority bounds.**  Largely replaced.  Attenuation is
  structural per the Waffles-derived rules in this profile: a child
  cap's `scope`, `resource`, and `authorization_details` are
  hash-chain-bound subsets/refinements of its parent's.  The bounds
  sketch's `reauthorized` carve-out (the consistency report's B2
  finding) does not arise because mid-chain widening is not
  supported.

  One bounds-sketch capability is not preserved: per-hop `aud`
  attenuation.  In this profile `aud` is fixed at the root cap and
  identical across the chain (a Waffles requirement), so a chain
  cannot narrow audience at an intermediate hop.  Deployments that
  rely on per-hop `aud` narrowing in bounds need to express that
  narrowing at root issuance instead, or remain on the receipts +
  bounds stack.

### 12.2 What `actor_caps` does not subsume

- **Transparency.**  Both transparency sketches (JWT-native and
  SCITT-aligned) remain applicable.  Adapting either to log caps
  is a one-line change.
- **AS-as-verifying-mediator.**  Receipts carry the additional
  semantics that an authorization server has *validated* the
  delegation at issuance time.  Caps do not carry that signal.
  Deployments that need a per-hop authorization-server validation
  trace SHOULD use receipts.

### 12.3 What `actor_caps` trades away

- **Operational footprint.**  Each delegating actor must hold a
  signing key.  In environments where actors are workloads or AI
  agents that come and go, key management for actors is heavier
  than in the receipts model where only authorization servers
  sign.
- **OAuth-AS centricity.**  The cap chain's trust path runs through
  delegators, not authorization servers.
- **Per-hop AS enrichment.**  An authorization server cannot add
  context that the delegator did not authorize.  The base
  profile's `sub_profile` enrichment by intermediate authorization
  servers has no analog here; entity classification must originate
  at the delegator or at the root.

### 12.4 When each architecture wins

- Use receipts + proofs + bounds when:
  - Authorization servers are the operational center of gravity
    and actors cannot reliably hold long-lived keys.
  - Per-hop AS-validation semantics are required by audit or
    policy.
  - Incremental adoption matters more than architectural
    minimality.
- Use `actor_caps` when:
  - Open-world cross-org delegation is the norm.
  - Actors (workloads, agents, tools) can hold ephemeral signing
    keys.
  - Architectural minimality and structural attenuation matter
    more than reuse of existing OAuth-AS machinery.

## 13. Principal-Authoritative Discovery {#principal-authoritative}

A recipient determines which keys are principal-authoritative for
a given `sub` through one of:

- A configured mapping of subject namespace prefixes to key sources
  (JWKS URIs or DID documents).
- The outer token's `iss` carrying authoritative subject namespace
  per local trust configuration (parallel to today's OpenID
  Connect trust models).
- A trust framework or federation metadata (analogous to OIDC
  Federation).

This profile does not standardize principal-authoritative
discovery; deployments use existing identity-trust mechanisms.

## 14. IANA Considerations {#iana}

This profile is derived from Waffles
({{I-D.hingnikar-cecchetti-wimse-waffles}}) and reuses Waffles'
IANA registrations.  In particular, the `waffle+jwt` media type
and the `wfl_parent`, `wfl_iss`, and `wfl_dep` JWS header
parameters are registered by Waffles; this profile does not
duplicate those registrations.

This profile registers the following additional artifacts.

### 14.1 JWT Claims Registry

- Claim Name: `actor_caps`
- Claim Description: Array of compact-serialized Waffles Layers
  representing the delegation chain attesting the outer token.
- Change Controller: the author of this sketch until/unless this
  document is adopted by the IETF.
- Specification Document: this document

### 14.2 OAuth Authorization Server Metadata Registry

- `actor_caps_supported`
- `actor_caps_signing_alg_values_supported`

### 14.3 OAuth Protected Resource Metadata Registry

- `actor_caps_required`
- `actor_caps_principal_authorities`
- `actor_caps_sender_constraints_supported`

### 14.4 OAuth Token Introspection Response Registry

- `actor_caps`: the array of compact-serialized cap JWTs returned
  alongside the introspected outer token, equivalent in semantics
  to the JWT claim of the same name.

### 14.5 OAuth Token Type Registry {#iana-token-types}

- `urn:ietf:params:oauth:token-type:actor-cap`
- Description: identifies the `actor_token_type` value used to
  carry a fresh delegator-signed child cap directly as a Token
  Exchange `actor_token` under this profile.

## 15. Worked Example {#worked-example}

This section is illustrative.  Signatures, key resolution details,
and unrelated claims are omitted for brevity.

A user `https://idp.enterprise.example/users/alice` authorizes a
travel agent `https://agents.example.com/travel` to act on her
behalf against `https://inventory.example`.  The travel agent then
sub-delegates to a booking tool `https://tools.example.com/booking`.

### 15.1 Root cap (Alice's identity provider signs)

JOSE header (excerpt):

~~~json
{
  "typ": "waffle+jwt",
  "alg": "ES256",
  "kid": "idp-key-2026",
  "wfl_dep": 3
}
~~~

Payload:

~~~json
{
  "iss": "https://idp.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "aud": "https://inventory.example",
  "resource": "https://inventory.example/trips",
  "scope": "trips:plan trips:book payments:authorize",
  "cnf": { "jkt": "agent-travel-jwk-thumbprint" },
  "iat": 1747900000,
  "exp": 1747986400,
  "jti": "cap-root-001"
}
~~~

This is the root cap (Waffles Root Waffle).  Its `iss` is Alice's
identity provider, configured as principal-authoritative for the
`https://idp.enterprise.example/users/*` namespace.  Its `cnf`
binds the travel agent's key.

### 15.2 Child cap (the travel agent signs)

JOSE header (excerpt):

~~~json
{
  "typ": "waffle+jwt",
  "alg": "ES256",
  "kid": "travel-agent-key-1",
  "wfl_parent": "<base64url(sha-256(root-cap-compact))>",
  "wfl_dep": 2
}
~~~

Payload:

~~~json
{
  "iss": "https://agents.example.com/travel",
  "sub": "https://idp.enterprise.example/users/alice",
  "aud": "https://inventory.example",
  "resource": "https://inventory.example/trips",
  "scope": "trips:book",
  "cnf": { "jkt": "tool-booking-jwk-thumbprint" },
  "iat": 1747900100,
  "exp": 1747903700,
  "jti": "cap-tool-007"
}
~~~

The child cap attenuates `scope` from three tokens to one,
preserves `sub` and `aud` per Waffles, preserves the authorized
`resource`, and binds the booking tool's key.  `wfl_parent` fixes
the root cap's serialization; `wfl_dep` is strictly less than the
parent's.

### 15.3 Outer token (the booking tool's AS issues)

~~~json
{
  "iss": "https://as.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "aud": "https://inventory.example",
  "resource": "https://inventory.example/trips",
  "scope": "trips:book",
  "act": {
    "sub": "https://tools.example.com/booking",
    "iss": "https://as.travel-provider.example",
    "act": {
      "sub": "https://agents.example.com/travel",
      "iss": "https://idp.enterprise.example"
    }
  },
  "cnf": { "jkt": "tool-booking-jwk-thumbprint" },
  "iat": 1747900200,
  "exp": 1747903700,
  "jti": "tok-outer-9000",
  "actor_caps": [
    "<child-cap-compact>",
    "<root-cap-compact>"
  ]
}
~~~

`actor_caps[0]` is the child cap (newest, covers the outermost
`act` at depth 1).  `actor_caps[1]` is the root cap (oldest, covers
the innermost `act` at depth 2).  The outer token's `aud` equals
the cap chain's common `aud`.  The outer token's `resource` is a
subset of the tip cap's `resource`.  The outer token's `scope`
("trips:book") is a subset of the tip cap's `scope` (also
"trips:book", in this example).

### 15.4 Verifier walk at the inventory service

The inventory service receives the outer OAuth access token,
presented by the booking tool with a DPoP proof matching the outer
token's `cnf.jkt`.  It applies {{consumer-processing}}:

1. Outer-token validation succeeds; DPoP proof verifies against
   `cnf`.
2. `actor_caps` present, length 2, equals visible `act` depth 2.
3. Both caps parse with `typ` = `waffle+jwt`.
4. Apply the Waffles-derived chain checks:
   - `actor_caps[0].wfl_parent` equals SHA-256 of root-cap-compact.
   - Both caps' signatures verify under their respective issuers'
     metadata-resolved keys.
   - `scope` and `resource` reduce monotonically; `sub` and `aud`
     are identical across layers; `iat`/`exp` adjacency holds;
     `wfl_dep` decreases.
   - No separate Waffles Stack-level DPoP `ath` check is required;
     the OAuth access-token proof was validated in step 1.
5. Root cap `iss` (`https://idp.enterprise.example`) is in the
   recipient's principal-authoritative trust set for Alice's
   namespace.
6. Layer-to-`act` alignment: actor-key metadata binds
   `actor_caps[0].cnf` to the actor at depth 1 (the booking tool)
   and `actor_caps[1].cnf` to the actor at depth 2 (the travel
   agent).  `actor_caps[0].iss` is the travel agent, which is the
   entity resolved from `actor_caps[1].cnf`.
7. Cap chain `sub` equals outer token `sub`.
8. Tip cap `cnf` matches outer token `cnf`; outer `aud` matches
   chain `aud`; outer `scope` and `resource` are subsets of the tip
   cap's values.
9. No extension dimensions.  Token accepted.

### 15.5 What an attacker cannot do

- **Forge upstream provenance.**  The booking tool cannot mint a
  fake root cap because it does not hold Alice's IdP signing key.
- **Widen authority.**  The booking tool cannot expand `scope`
  from "trips:book" to "payments:authorize"; widening would
  require signing a new chain.
- **Replay to a different audience.**  `aud` is identical across
  every layer; an outer token issued for
  `aud=https://inventory.example` cannot be accepted by any other
  audience under this profile's outer-token-binding check.
- **Reorder hops.**  Reversing the cap array breaks `wfl_parent`.

## 16. Open Design Questions

Many open questions about the per-layer format itself are inherited
from Waffles' open issues ({{I-D.hingnikar-cecchetti-wimse-waffles}}
Open Issues).  This section lists open questions specific to this
profile.

**Q1: Actor-key binding metadata (load-bearing).**
{{act-alignment}} defines the verification rule but leaves the
actor-key binding source to local policy (authorization-server
client metadata, workload identity metadata, DID documents,
federation metadata, or another configured registry).  This is the
most consequential under-specification in this profile: the
"delegator signs, not AS signs" trust model rests on recipients
being able to resolve a visible `act.sub` to a `cnf` key
deterministically.  Two implementations choosing different binding
sources will not interoperate.

Two leading candidate mechanisms:

- **Entity-statement-rooted.**  The principal-authoritative party
  publishes OpenID-Federation-style entity statements about each
  actor it has authorized; downstream `act.sub` identifiers
  resolve through that hierarchy.  Heavier on OIDC-Federation
  adoption but reuses an existing trust-fabric mechanism.
- **`act`-embedded `cnf`.**  Extend the core actor profile's `act`
  object to carry the actor's `cnf` directly, so resolution is
  reading the visible chain.  Closes the loop locally but pushes
  the actor-key binding requirement into the base profile.

A future revision of this profile SHOULD pick one as the default,
with the other (or a deployment-specific source) as a fallback,
before this profile is testable end-to-end.

**Q2: Bridge migration from receipts to caps.**  The mutual-
exclusion rule in {{coexistence}} excludes a per-hop migration
period.  Should a bridge format be defined (a cap that wraps a
receipt, or a receipt that references a cap) for deployments
mid-migration?

**Q3: Cap chain transparency.**  A transparency companion for
caps parallel to the receipts transparency sketches would address
principal-authoritative-key compromise.  Should this be defined as
part of this profile, as part of Waffles, or as a separate sketch?

**Q4: Bearer-mode variant.**  This profile requires `cnf` on every
cap (per Waffles).  This rules out bearer-style downstream actors
whose presenter identity is bound by the outer-token issuer rather
than by the delegator.  Should this profile define an explicit
bearer-mode variant where the tip cap's `cnf` is set by the
authorization server at issuance, or is this best left to Waffles
to address?

**Q5: Mid-chain re-authorization.**  {{re-authorization}} treats
re-authorization as supersession by a new root.  An alternative
formulation would treat it as a new cap signed by a principal-
authoritative key inserted mid-chain.  This alternative is not a
softer variant of the same model; it is a structurally different
model that introduces two principal-authoritative-signed caps in
one chain and breaks the invariant that every non-root cap's
signer is the entity named by the parent cap's `cnf`.  Worth
returning to once Waffles' open issues settle.

**Q6: cnf rotation proofs.**  Short-lived workload keys rotate
faster than typical cap chains.  {{cnf-rebinding-mid-chain}}
defers cnf rotation to re-signing.  Should this profile (or
Waffles) standardize a cnf-rotation proof so a delegator can
authorize a new delegatee key without re-issuing the full cap?

**Q7: Maximum root-cap lifetime.**  Cap `exp` constrains outer-
token `exp` through the chain.  Long-lived root caps make refresh
easier but expand the revocation surface.  Should this profile
recommend a maximum root-cap lifetime, or leave it to deployment?

**Q8: French Toast JWT, OVID, capabilities crosswalk.**  Waffles
explicitly notes structural compatibility with FT-JWT and OVID
({{I-D.hingnikar-cecchetti-wimse-waffles}} §6.3, §6.4).  Should
this profile define explicit crosswalks (typ substitution rules)
so an `actor_caps`-bearing token is interchangeable with FT-JWT or
OVID artifacts where the underlying semantics permit?

## 17. Net

This profile is one artifact replacing three companions: a Waffles
Stack carrying delegated authority through the OAuth Actor Profile
family.  Its bet is that signing the right thing by the right
party makes monotonicity, non-repudiation, and prior-hop
provenance fall out of a single signature chain.  Its costs are
operational (actors must sign) and architectural (authorization
servers leave the trust path).

By deriving from Waffles rather than reinventing the layer format,
this sketch avoids fragmenting the layered-JWT delegation space.
Waffles provides the per-layer structure and attenuation model; this
profile provides the OAuth actor-family integration (outer-token
wrapping, principal-authoritative trust root, `act`-chain
alignment, OAuth sender-constraint validation, Token Exchange, and
refresh).

## See Also

- [README](./README.md) for family context.
- {{I-D.hingnikar-cecchetti-wimse-waffles}}: the per-layer JWT
  format this profile applies.
- [draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md):
  the receipts companion this profile replaces in deployments that
  adopt `actor_caps`.
- [draft-mcguinness-oauth-actor-proofs](../draft-mcguinness-oauth-actor-proofs.md):
  the actor-signed proofs sketch this profile subsumes.
- [draft-mcguinness-oauth-actor-authority-bounds](../draft-mcguinness-oauth-actor-authority-bounds.md):
  the authority-monotonicity sketch this profile makes structural
  through Waffles.
- [draft-mcguinness-oauth-actor-receipts-transparency](./draft-mcguinness-oauth-actor-receipts-transparency.md)
  and
  [draft-mcguinness-oauth-actor-receipts-scitt](./draft-mcguinness-oauth-actor-receipts-scitt.md):
  the transparency sketches that remain applicable, with caps
  substituted for receipts in their leaf format.

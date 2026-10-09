---
title: "OAuth Actor Profile for Delegation"
abbrev: "OAuth Actor Profile"
category: std
docname: draft-mcguinness-oauth-actor-profile-latest
submissiontype: IETF
number:
ipr: "trust200902"
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - oauth
 - delegation
 - actor
 - token exchange
 - token
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "mcguinness/draft-mcguinness-oauth-actor-profile"
  latest: "https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-profile.html"

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC3986:
  RFC6749:
  RFC6750:
  RFC7009:
  RFC7519:
  RFC7521:
  RFC7523:
  RFC7800:
  RFC8705:
  RFC7662:
  RFC8414:
  RFC9728:
  RFC8693:
  RFC9068:
  RFC8707:
  RFC9449:
  I-D.ietf-oauth-transaction-tokens:
  I-D.ietf-wimse-workload-creds:
  I-D.ietf-wimse-wpt:
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
  I-D.ietf-oauth-identity-assertion-authz-grant:
    title: "Identity Assertion JWT Authorization Grant"
    author:
     -
        fullname: Aaron Parecki
        organization: Okta
     -
        fullname: Karl McGuinness
        organization: Independent
     -
        fullname: Brian Campbell
        organization: Ping Identity
    date: 2026-05-21
    seriesinfo:
      Internet-Draft: draft-ietf-oauth-identity-assertion-authz-grant-04
    target: https://www.ietf.org/archive/id/draft-ietf-oauth-identity-assertion-authz-grant-04.txt
  OpenID.Core:
    title: "OpenID Connect Core 1.0"
    author:
      org: OpenID Foundation
    date: 2014-11-08
    target: https://openid.net/specs/openid-connect-core-1_0.html

informative:
  RFC9700:
  RFC8792:
  I-D.parecki-oauth-jwt-dpop-grant:
    title: "JWT Authorization Grants with DPoP"
    author:
     -
        fullname: Aaron Parecki
        organization: Okta
    date: 2026-01-30
    seriesinfo:
      Internet-Draft: draft-parecki-oauth-jwt-dpop-grant-01
    target: https://datatracker.ietf.org/doc/html/draft-parecki-oauth-jwt-dpop-grant-01
  I-D.ietf-oauth-identity-chaining:
  OpenID.Federation:
    title: "OpenID Federation 1.0"
    author:
      org: OpenID Foundation
    date: 2024-05-01
    target: https://openid.net/specs/openid-federation-1_0.html

...

--- abstract

This document defines a common representation of delegated actors in OAuth JSON Web Token (JWT) assertion grants, JWT access tokens, and Transaction Tokens.  It profiles the `act` claim defined by OAuth 2.0 Token Exchange, requires issuer-scoped actor identifiers, and uses the `sub_profile` claim to classify actor entity types.  It specifies token processing, delegation-chain propagation, sender-constraint handling, and discovery metadata, so that issuers and resource servers in different trust domains interpret delegated actors consistently.

--- middle

# Introduction

Delegated requests can pass through several services and trust domains.  Each recipient needs to distinguish the subject whose authorization is exercised, the actor exercising it, and the OAuth client requesting the token: `sub` identifies the authorizing principal, `act.sub` the actor, and `client_id` the client registration.  This profile makes the actor explicit in the token rather than leaving it to be inferred from client registration, without redefining client identity or subject semantics.

OAuth 2.0 Token Exchange {{RFC8693}} defines the `act` claim for the current actor and prior actors, but leaves "the specifics of representing a composite token" to implementations ({{Section 1.1 of RFC8693}}).  It notes that the `iss` and `sub` claims together "might be necessary" to identify an actor ({{Section 4.1 of RFC8693}}), but it does not require an identifier context or classify actors.  Its delegation example derives the `act` claim from the subject of the `actor_token` ({{RFC8693, Appendix A.2.5}}), but it does not specify actor validation and derivation rules for each token type, how `act` propagates across JWT assertion grants, JWT access tokens, and Transaction Tokens, or how the actor relates to a sender-constrained presenter.  Without a common profile, deployments face four interoperability gaps:

*  **No standard entity classification.** The `sub` claim is overloaded across end-users, service accounts, AI agents, and workloads, with no classification that supports deterministic cross-domain policy.
*  **Inconsistent actor representation across token types.** Actor context, including actor key material, has no representation that survives transformation among JWT assertion grants, JWT access tokens, and Transaction Tokens.
*  **Implicit delegation via client identity.** A client registration alone might not identify the actor, particularly when one registration serves several agents or workloads, when requests pass through intermediaries, or when tokens cross trust domains.
*  **No discovery for actor-profile support.** Neither AS metadata {{RFC8414}} nor Protected Resource Metadata {{RFC9728}} defines parameters for advertising actor-profile support.

This document profiles `act` to close those gaps.  It defines:

*  Issuer-scoped actor identifiers and entity classification using `sub_profile`.
*  Rules for validating, extending, and preserving actor information when tokens are exchanged or reissued.
*  Presenter continuation and rebind rules for sender-constrained tokens, including upgrades from bearer tokens.
*  Resource server processing based on the (`sub`, outermost `act.sub`) pair, and metadata for advertising profile support.
*  Extension points for companion profiles that provide additional delegation evidence.

This profile applies to human, service, workload, and AI agent delegation.  The requirements of the underlying specifications, including {{RFC8693}}, {{RFC9068}}, {{RFC9449}}, and {{I-D.ietf-oauth-transaction-tokens}}, continue to apply unless this document states otherwise.  [Profile Scope](#profile-scope) describes the supported token paths and the boundary between representation and authorization policy.

## Illustrative Use Case

Alice authorizes an AI travel agent to book a trip.  The enterprise AS issues a credential with Alice as `sub` and the agent as `act`.  The agent presents that credential to a booking provider's AS for an access token, and the provider then issues a Transaction Token for an internal booking tool.  Alice remains the subject; the tool becomes the outermost actor, and the agent becomes an inner actor.  Each trust domain issues a new token under its own policy, and the outermost actor changes only when a new presenter is established.  [The cross-domain example](#appendix-cross-domain) shows the complete flow.

## Relationship to Related Work

*  **OAuth Token Exchange ({{RFC8693}})** defines the `act` claim and exchange mechanism profiled here.
*  **Identity Chaining ({{I-D.ietf-oauth-identity-chaining}})** propagates subject identity across domains and can be combined with this profile's actor representation.
*  **Identity Assertion JWT Authorization Grant (ID-JAG, {{I-D.ietf-oauth-identity-assertion-authz-grant}})** defines the issuance and consumption of JWT authorization grants.  It permits `actor_token` inputs but leaves their processing, and whether the issued grant carries the `act` claim, to future profiles or extensions; this document is one such profile, through its Token Exchange and JWT assertion-grant rules.
*  **OAuth Entity Profiles ({{I-D.mora-oauth-entity-profiles}})** defines the classification claims, metadata, and registry used by this profile.
*  **Transaction Tokens ({{I-D.ietf-oauth-transaction-tokens}})** defines the token and service model extended here with actor claims and processing rules.
*  **WIMSE Workload Identity ({{I-D.ietf-wimse-workload-creds}}, {{I-D.ietf-wimse-wpt}})** defines workload credentials and proofs used in [the cross-domain example](#appendix-cross-domain).  This profile also supports other presenter-authentication mechanisms.

# Conventions and Definitions {#conventions}

{::boilerplate bcp14-tagged}

This document uses the OAuth terminology defined in {{RFC6749}} and {{RFC8693}}, and the terms Transaction Token and Transaction Token Service (TTS) defined in {{I-D.ietf-oauth-transaction-tokens}}.  AS and RS denote authorization server and resource server, respectively.

This document uses the following terms:

Actor:
: The party actively making a request.  When delegation is present, the actor is distinct from the subject; the subject is the principal on whose behalf the actor is acting.

Subject:
: The principal whose authorization is being exercised.  In a delegated token, the subject is the original authorizing party (e.g., an end-user or an upstream service), not the party making the immediate network request.

Delegation:
: The act by which a principal authorizes another party (the actor) to exercise a subset of the principal's rights.

Cross-Domain Delegation:
: Delegation in which the subject and actor are governed by different trust domains or identifier namespaces.  Deployment policy determines whether a token represents cross-domain delegation, based on issuer context, actor identifiers, and applicable trust agreements.  The top-level `iss` claim alone is not always sufficient.

Actor Authorization at the Resource Server:
: An authorization policy evaluation that considers both the subject and the actor, and the relationship between them, as policy inputs.  Under this profile, the relevant actor is ordinarily the outermost actor.

Delegation Chain:
: The sequence of actors representing how authorization has passed from the subject principal (`sub`) to the first actor (innermost `act`) and through any intermediate parties to the immediate actor (outermost `act.sub`).  The chain is conveyed by the nested structure of the `act` claim.

Outermost Actor:
: The `act` object at the top level of the delegation chain (the one not nested inside any other `act` object).  The outermost actor identifies the current presenter of the token.

Local Policy:
: Rules or decisions, not defined by this document, that an AS, RS, or organization applies, such as delegation approval, scope reduction, identifier mapping, and entity-profile acceptance.

Identifier Reconciliation:
: Applying configured mapping rules to determine whether identifiers from different claims or namespaces refer to the same entity.  A recommendation to perform identifier reconciliation means the implementation SHOULD apply those rules.  String similarity or shared naming patterns do not establish equivalence.  If no applicable mapping exists or reconciliation fails, equivalence is not established: the identifiers MUST be treated as distinct, and an implementation MUST reject a request or token whose processing requires them to identify the same entity.

Examples are illustrative and can omit claims, parameters, or validation steps unrelated to this profile.

This document uses dot-path notation to refer to nested claim values.  For example, `act.sub` refers to the `sub` member of the `act` object, and `act.act.sub` refers to the `sub` member of the `act` object nested within the outer `act` object (the immediately prior actor in a depth-2 chain).


# Actor Profile for Delegation {#actor-profile}

## Overview

When an implementation uses this profile to represent an actor distinct from the subject, it MUST apply the requirements in this section.  The absence of an explicit inbound actor credential MUST NOT by itself be interpreted as making the OAuth client the delegated actor.

## Profile Invariants

| Claim | Meaning |
|-------|---------|
| Top-level `sub` | Subject whose authorization is exercised |
| Outermost (`act.iss`, `act.sub`) | Canonical identifier of the immediate actor |
| Inner `act` objects | Prior actors, ordered from most recent to earliest |
| `client_id`, `azp` | OAuth client identity |
| Top-level `cnf` | Current presenter's key or certificate binding |

Inner actors have the trust properties described in [Carry Prior-Actor Context](#carry-prior-actor-context).

When (`act.iss`, `act.sub`) identifies the same entity as the token's (`iss`, `sub`), consumers MUST NOT infer a delegation relationship.  Comparing `act.sub` with `sub` alone is insufficient; the identifier contexts are part of the comparison.

## Profile Scope {#profile-scope}

### Representation and Policy {#representation-and-policy}

This profile standardizes actor representation, propagation, validation, and discovery.  Delegation approval, trust frameworks, and identifier mappings remain deployment-specific.  Cross-domain deployments require agreements that cover permitted delegation relationships and identifier namespaces.

[Actor Authorization](#actor-authorization) describes when an RS enforces authorization of the (`sub`, outermost `act.sub`) pair; this document does not require it on every request.

### Token Format Scope

Conforming outputs are JWT assertion grants, JWT access tokens, and Transaction Tokens.  [Token Introspection](#token-introspection) defines optional equivalent response members for delegated opaque access tokens; this compatibility path does not make the opaque token itself conformant.

Opaque access tokens used as Token Exchange inputs are outside the interoperable scope of this document.  An AS MAY translate their introspection results into local inputs under deployment-specific rules.

### Supported Token Types and Request Semantics

| Role | Supported credentials |
|------|-----------------------|
| Token Exchange `subject_token` | JWT assertion grant, JWT access token, ID Token, refresh token, Transaction Token |
| Token Exchange `actor_token` | Workload identity credential, JWT client assertion, non-delegated JWT access token |
| TTS `subject_token` | JWT assertion grant, JWT access token, Transaction Token |
| Issued token | JWT assertion grant, JWT access token, Transaction Token |

Subject to endpoint policy and the underlying grant mechanism, implementations MAY transform supported inputs into outputs for which this document defines issuance rules.  They need not support every combination.  For each supported path, actor information MUST be validated and preserved according to the output rules in [JWT Assertion Grant Output](#jwt-assertion-grant-issuance), [JWT Access Token Output](#jwt-access-token-propagation), or [Transaction Token Output Rules](#transaction-token-output-rules).

The `may_act` claim is an optional delegation-authorization input, with the restrictions in [`may_act`](#may-act).  It neither establishes actor identity nor propagates to the output token.

[The service-to-service example](#appendix-service-to-service) and [the cross-domain example](#appendix-cross-domain) show same-domain service delegation and cross-domain delegation, respectively.

## Actor Object Structure {#actor-object-structure}

An actor object conforming to this profile is a JSON object that is the value of the `act` claim.  In addition to the `sub` claim required by {{RFC8693}}, a profile-conformant actor object MUST contain an `iss` claim and SHOULD contain a `sub_profile` claim when the issuer can authoritatively classify the actor's entity type.  An `act` object that omits the `iss` claim conforms to {{RFC8693}} but does not conform to this profile; [Migration and Adoption](#migration-and-adoption) specifies the handling of such objects.

~~~
act-object = {
  "sub"           : StringOrURI,        ; REQUIRED
  "iss"           : StringOrURI,        ; REQUIRED
  ? "sub_profile" : JSON String,        ; RECOMMENDED
  * StringOrURI => any                  ; extension claims
}
~~~

`sub`:
: REQUIRED.  The subject identifier of the actor, as defined in {{Section 4.1 of RFC8693}}.  It is a StringOrURI as defined in {{RFC7519}}.

`iss`:
: REQUIRED.  The issuer or namespace context for `act.sub`, expressed as a StringOrURI {{RFC7519}}.  Together, (`act.iss`, `act.sub`) form the canonical actor identifier.  For any actor identifier scheme, `act.iss` MUST identify the context used when assigning or asserting that identifier.

  Implementations MUST NOT interpret `act.iss` as the current token issuer, credential issuer, or hop-provenance marker.  These entities can coincide but have distinct roles.  HTTPS URLs and workload-identity URNs are examples of possible context identifiers.

  For example, a TTS at `https://tts.travel-provider.example` can issue a Transaction Token whose booking-tool actor has `act.iss` set to `https://as.travel-provider.example` because that AS issued the booking tool's workload credential.  The TTS signs the token; the AS supplies the namespace for the tool's identifier.

`sub_profile`:
: RECOMMENDED.  A space-delimited list of entity profile values classifying the actor identified by `act.sub`, as defined in {{Section 4.2 of I-D.mora-oauth-entity-profiles}}.  Values used within `act` objects MUST be registered with the "Actor Profile" usage location in the OAuth Entity Profiles registry ({{Section 14.1 of I-D.mora-oauth-entity-profiles}}) or be privately defined collision-resistant values.

  If the acting entity fits more than one profile, multiple values MAY be included as a space-delimited string (e.g., `"service ai_agent"`).  {{I-D.mora-oauth-entity-profiles}} defines interoperability requirements and implementation guidance for multi-value strings.

  When `sub_profile` is absent from an `act` object, implementations MUST NOT assume a specific entity type for the actor; resource servers that enforce entity-type-based access control MUST treat an absent `sub_profile` as an unclassified actor and SHOULD apply the more restrictive policy applicable to unknown entity types.

  The `sub_profile` claim MAY also appear as a top-level JWT claim outside any `act` object to classify the entity type of the token's `sub`; it applies exclusively to `sub` and does not affect `sub_profile` values within `act` objects.  Issuers SHOULD include a top-level `sub_profile` when they can authoritatively classify the subject entity type.

The top-level `cnf` claim ({{RFC7800}}) carries the current presenter's binding; see [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation).

The `client_profile` claim defined in {{I-D.mora-oauth-entity-profiles}} classifies the OAuth client and MUST NOT appear within an `act` object.  Client classification belongs at the top level of the token.  An AS or RS that encounters a `client_profile` member inside an `act` object MAY reject the token or ignore the offending member; it MUST NOT treat that member as a valid actor classification.

When an `act` object contains extension members beyond those defined in this document, issuers and consumers MUST ignore unrecognized members unless another specification or local policy defines their meaning.  An issuer that preserves a validated delegation chain copies unrecognized extension members in inherited `act` objects unchanged, as [Preserve Inbound Chain](#preserve-inbound-chain) requires.


## Delegation Chains {#delegation-chains}

Delegation chains MUST use nested `act` objects as specified in {{Section 4.1 of RFC8693}}.  The outermost object identifies the immediate actor; the innermost object identifies the first actor authorized by the subject.  The chain records prior actors under the conveying issuer's trust, without independently proving each hop.  This profile defines one linear chain per token; concurrent delegations use separate tokens.

This document uses the following terms:

*  A **hop** is a single `act` object in a delegation chain.  The number of hops in a chain equals the chain's delegation depth.
*  A **visible hop** is a hop that appears in the token's `act` chain as received by a recipient, after any filtering by an introspection server.
*  A **single-hop actor object** is an `act` object with no nested `act`; it represents delegation depth 1.
*  An **inbound delegation chain** is the complete `act` structure received in an inbound token, whether depth 1 or greater.
*  A **preserved delegation chain** is an inbound delegation chain that an issuer has validated and copied into a newly issued token without rewriting inherited actor entries.
*  A **new outermost actor** is the actor object created by the current issuer to represent the newly identified outermost actor for the token it is issuing.

The following example shows a delegation chain of depth 2:

~~~json
{
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
  "act": {
    "sub": "https://tools.travel-provider.example/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service",
    "act": {
      "sub": "https://agents.enterprise.example/travel-assistant",
      "iss": "https://as.enterprise.example",
      "sub_profile": "ai_agent"
    }
  }
}
~~~
In this example, Alice (`sub`) authorized the travel assistant (inner `act`), which delegated to the booking tool (outermost `act`).  The booking tool is the current presenter.

Delegation depth is the number of `act` objects in the chain, counting from the outermost.  Depth is counted on the resulting chain after any new outermost `act` is added, not on the inbound token.

Depth 1 is the minimum interoperable depth.  Implementations for cross-domain multi-hop use SHOULD support at least depth 4, and should document their maximum.  Same-domain deployments can use a shallower maximum when it is sufficient for their architecture.

Implementations MUST define and enforce a local maximum delegation depth.  Implementations that receive a token exceeding their configured local maximum MUST reject it as an invalid input under the applicable token-processing rules; AS and TTS errors follow [Error Responses](#actor-profile-error-responses).  When a request would add an actor beyond that limit, the AS MUST reject the request with the `invalid_request` error code; it MUST NOT silently truncate the chain.

A token represents delegation when the party exercising the token's authorization at runtime (the actor) is distinct from the token subject (`sub`) and the subject has authorized the actor to do so.  The following conditions establish this:

1.  A validated `actor_token` identifying a distinct actor was present in the exchange request that produced this token.
2.  An inbound `subject_token` from a trusted upstream issuer already carried an `act` chain, or the state of a refresh token presented as `subject_token` records one, establishing that delegation was present before the current exchange.
3.  The issuing AS has an independent delegation basis such as a pre-registered grant, an explicit consent record, a `may_act` claim in a validated upstream token (see [`may_act`](#may-act)), or a policy rule establishing that the current client or actor is acting as a distinct party on behalf of `sub` (see [JWT Access Tokens](#jwt-access-tokens) for the outside-Token-Exchange case).  In Token Exchange, such a basis applies only to an actor established as the next paragraph describes.

In Token Exchange, an independent delegation basis can authorize an actor established under this profile's processing rules but cannot by itself establish a new actor.  A new actor is established from a validated `actor_token` or, at an AS other than a TTS, from the authenticated client on the [`may_act`](#may-act) path without `actor_token`.  Existing delegation is preserved under presenter continuation ([Presenter Transition Model](#token-exchange-presenter-model)).  A Token Exchange that establishes no new actor and has no inbound `act` chain does not represent delegation.

When a token represents delegation, the `act` claim MUST be present and MUST conform to [Actor Object Structure](#actor-object-structure).  When none of these conditions holds, the token does not represent delegation and the `act` claim MUST be omitted.  The AS MUST NOT include the `act` claim solely because `sub` and the OAuth client identifier differ; the distinction between `sub` and `client_id` is expected and does not by itself constitute delegation under this profile.


## Delegation Chain Validation and Construction {#delegation-chain-algorithm}

An AS MUST apply this algorithm on paths that require or claim actor-profile conformance.  The invoking grant, Token Exchange, or TTS rules supply token-specific preconditions and delegation-authorization requirements.  Nonconforming actor objects, including those missing `iss`, MUST NOT enter this algorithm; [Migration and Adoption](#migration-and-adoption) defines their handling.

Interoperable processing under this profile is defined around `sub` and the outermost `act.sub`.  Inner actors are **prior-actor context**: preserved for audit and downstream use, and not inputs to access-control decisions, including scope determination, because prior actors identified by nested `act` objects are informational only ({{Section 4.1 of RFC8693}}).  Structural rules, such as the depth limit and the actor object requirements, still apply to every actor object.

### Validation Steps

The AS applies the validation steps in the following order:

1.  Validate the carrier token using [Validate Carrier Token](#validate-carrier-token).
2.  Validate the outermost actor using [Validate Outermost Actor](#validate-outermost-actor).
3.  Treat inner actors as prior-actor context using [Carry Prior-Actor Context](#carry-prior-actor-context).
4.  Enforce the configured maximum chain depth using [Enforce Depth Limit](#enforce-depth-limit).

#### Validate Carrier Token {#validate-carrier-token}

The AS MUST validate the token carrying the inbound delegation chain per the type-specific rules applicable to that token before extracting actor claims.

#### Validate Outermost Actor {#validate-outermost-actor}

For the outermost `act` object, the AS MUST:

1.  Verify that both `act.sub` and `act.iss` are present.  If either is absent, reject the input under [Error Responses](#actor-profile-error-responses).
2.  Verify that local policy trusts the token issuer to assert (`act.iss`, `act.sub`); otherwise, reject the input under [Error Responses](#actor-profile-error-responses).  [Trusting Actor Identifier Pairs](#act-iss-authority-guidance) gives examples of this deployment-specific trust decision.
3.  Evaluate delegation under local policy:

    *  When [extending the chain](#extend-chain-with-new-actor), the AS MUST confirm that the new actor is authorized to act for `sub`, for example through a grant, consent record, or policy rule.
    *  When [preserving a validated chain](#preserve-inbound-chain) from a trusted issuer, the AS SHOULD evaluate the preserved relationship.  Upstream evaluation suffices for baseline interoperability.
    *  If a required relationship is prohibited or cannot be confirmed, reject the input with the `actor_unauthorized` error code.

[Authorization Grant Processing](#jwt-assertion-grants-processing) adds requirements for JWT assertion grants.

#### Carry Prior-Actor Context {#carry-prior-actor-context}

For inner `act` objects, which are prior-actor context, the AS MAY rely on trust in the outer token issuer established by [Validate Carrier Token](#validate-carrier-token) rather than independently validating each hop.  The AS MUST NOT treat preserved prior-actor context as independently authenticated; an inner `act` entry carried in a token is endorsed only by the outer token issuer's signature, not by independent verification at each prior hop.

#### Enforce Depth Limit {#enforce-depth-limit}

Compute the depth of the resulting chain, including any new outermost actor added by [Extend Chain with New Actor](#extend-chain-with-new-actor).  If that depth exceeds the locally configured maximum ([Delegation Chains](#delegation-chains)), reject under [Error Responses](#actor-profile-error-responses), distinguishing an excessive inbound chain from an extension that would exceed the limit.

### Construction Steps

The AS selects exactly one construction step in the following order:

1.  If a new actor is identified for the issued token, use [Extend Chain with New Actor](#extend-chain-with-new-actor).
2.  Otherwise, if a validated inbound delegation chain is present, use [Preserve Inbound Chain](#preserve-inbound-chain).
3.  Otherwise, use [Omit `act`](#omit-act).

#### Extend Chain with New Actor {#extend-chain-with-new-actor}

When a new actor is identified, the AS creates a new outermost `act` object and nests any validated inbound chain beneath it.

~~~
AddOutermostActor(inbound_chain, new_actor):
  outermost.sub = new_actor.sub  // REQUIRED
  outermost.iss = new_actor.iss  // REQUIRED: identifier context
  // RECOMMENDED when authoritatively known:
  outermost.sub_profile = new_actor.sub_profile

  if inbound_chain is present:
    outermost.act = inbound_chain  // preserve the entire chain
    verify depth(outermost) <= local_max_depth

  return outermost
~~~

The AS MUST set the new actor's `act.iss` and MUST NOT change any inherited actor field.  It MUST preserve the entire inbound chain; [Enforce Depth Limit](#enforce-depth-limit) rejects a resulting chain that exceeds the local maximum.

If the new actor has the same (`act.iss`, `act.sub`) pair as the inbound outermost actor, the AS MAY instead apply [Preserve Inbound Chain](#preserve-inbound-chain) to avoid a duplicate entry.

This profile does not standardize detection of identifier reappearance deeper in the inbound chain (for example, the same actor appearing in both inner and outer positions of a longer chain).  An AS MAY apply local policy to such cases; the chain-construction algorithm itself neither requires nor prohibits cycle detection.

#### Preserve Inbound Chain {#preserve-inbound-chain}

When no new actor is identified and an inbound chain is present, the AS copies the validated inbound chain unchanged.

~~~
PreserveChain(inbound_chain):
  return inbound_chain  // copy without modification
~~~

The AS MUST copy the validated inbound chain exactly into the issued token.  The AS MUST NOT add, remove, or rewrite any field in any actor object of a preserved delegation chain.

#### Omit `act` {#omit-act}

When no delegation is present and no actor information should appear in the issued token, the AS omits `act`, as required by [Delegation Chains](#delegation-chains).  The AS MUST NOT silently drop an inbound `act` claim; if it cannot preserve or extend the chain, it MUST reject the request per [Error Responses](#actor-profile-error-responses).

## Sender Constraint and Proof-of-Possession Validation {#delegated-pop-validation}

The rules in this section apply to all supported token types, in addition to their type-specific proof-of-possession (PoP) requirements.  [Presenter Transition Model](#token-exchange-presenter-model) defines continuation and rebind for Token Exchange.  DPoP nonce handling follows {{Section 8 of RFC9449}} unchanged.

Per-actor confirmation members and prior-hop key provenance are outside the scope of this document; see [Companion Profiles and Extension Points](#companion-profile-extensibility).

### Top-Level `cnf` Governs the Current Presenter

The top-level `cnf` claim of any token identifies the key or certificate of the current presenter.  When delegation is present, that current presenter is the party identified by the outermost `act` claim: when DPoP ({{RFC9449}}) is used, the top-level `cnf.jkt` MUST identify that party's key; when mTLS ({{RFC8705}}) is used, the top-level `cnf.x5t#S256` MUST identify that party's certificate.  The AS or RS MUST validate proof of possession against the top-level `cnf`.

A confirmation member such as `act.cnf` is permitted as extension data under [Actor Object Structure](#actor-object-structure) and has no proof-of-possession semantics under this profile.  It remains unchanged in an inherited actor object under [Preserve Inbound Chain](#preserve-inbound-chain).  After further delegation, an inherited confirmation value can therefore differ from the current token's top-level `cnf` claim; this profile imposes no equality check between them.

### Token Exchange Continuation

Presenter continuation requires either a PoP-capable `subject_token` with a top-level `cnf` claim or a bearer `subject_token` presented by an authenticated requester that corresponds, under Identifier Reconciliation ([Conventions and Definitions](#conventions)), to the token's outermost (`act.iss`, `act.sub`) pair, or to its `sub` when the token carries no `act` claim.  For a PoP-capable `subject_token`, the requester MUST prove possession of its binding using the mechanism applicable to the token type and deployment.  A bearer continuation yields a bearer output.

### Token Exchange Rebind

Presenter rebind requires a validated `actor_token` whose top-level `sub` identifies the new presenter, as specified in [Actor Tokens](#actor-tokens), except on the [`may_act`](#may-act) path without an `actor_token`, where the authenticated client is the new presenter.

*  The issuer validates the credential per {{Section 2.1 of RFC8693}} and MUST validate any proof required by its profile or deployment, whether or not the output token is sender-constrained.
*  When the output token is sender-constrained, the issuer MUST validate proof of possession for the new presenter.  A bearer output does not waive validation of the credential or of any proof its profile requires.  A sender-constrained `subject_token` does not, by itself, require proof for its prior presenter during rebind.

Other actors that become presenters therefore need a direct credential: a workload identity credential, a JWT client assertion, or a non-delegated JWT access token.

### Bearer-to-PoP Upgrade

When the inbound `subject_token` is a bearer credential or an identity-only credential and the request supplies a validated `actor_token` establishing a new presenter, the issuer MAY issue a sender-constrained output token bound to that new presenter.  The absence of a top-level `cnf` claim in the `subject_token` imposes no continuity obligation in this case.

# JWT Assertion Grants {#jwt-assertion-grants}

## Structure {#jwt-assertion-grants-structure}

These requirements apply to JWT authorization grants under {{RFC7521}} and {{RFC7523}}, including ID-JAG {{I-D.ietf-oauth-identity-assertion-authz-grant}}.  The grant profile defines issuance and exchange; this document defines actor representation and delegation processing.

A JWT authorization grant MAY carry an `act` claim conforming to [Actor Object Structure](#actor-object-structure).  Actor claims in JWT client authentication assertions are outside the scope of this document.  The `act` claim represents explicit delegation, even when the issuer derives the actor from authenticated client context, as on the [`may_act`](#may-act) path.

The following claims are defined for a JWT assertion grant that carries actor-profile delegation.  Claims not listed here follow the requirements of {{RFC7521}} and {{RFC7523}}.

`iss` (REQUIRED):
: Identifies the assertion issuer.  MUST be authorized by local policy to assert the relationship between `sub` and `act.sub`.

`sub` (REQUIRED):
: Identifies the principal on whose behalf the grant is made.

`jti` (REQUIRED):
: Identifies the assertion for replay prevention ([Authorization Grant Processing](#jwt-assertion-grants-processing)).

`sub_profile` (RECOMMENDED):
: Classifies the entity type of `sub`.  MUST conform to {{Section 4.2 of I-D.mora-oauth-entity-profiles}}.

`act` (REQUIRED when delegation is asserted):
: The actor object identifying the entity exercising the subject's delegated rights.  MUST conform to the actor object structure defined in [Actor Profile for Delegation](#actor-profile).

`cnf` (REQUIRED when sender-constrained; otherwise OPTIONAL):
: When the JWT assertion grant is sender-constrained, the assertion MUST carry a top-level `cnf` claim identifying the binding: `cnf.jkt` per {{RFC9449}} when DPoP is used, or `cnf.x5t#S256` per {{RFC8705}} when mTLS is used.  When the assertion is not sender-constrained, a top-level `cnf` claim is OPTIONAL unless another profile or local policy requires it.

Before sending JWT assertion grants carrying actor-profile claims, a client needs to confirm that the AS supports the actor-determination model through deployment documentation, prior agreement, or discovery; for ID-JAG, that includes support for the actor-delegation extension model defined by this document.

The following example shows an AS-issued assertion grant, which is the recommended pattern.  The Enterprise IdP AS performed Token Exchange, authenticated the agent, which presented its client assertion as `actor_token`, confirmed the delegation relationship under local policy, and signed the assertion.  Here, `act.iss` equals the assertion's `iss` because the enterprise AS's issuer identifier is also the actor identifier context for the agent:

~~~json
{
  "iss": "https://as.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "aud": "https://as.travel-provider.example",
  "jti": "a1b2c3d4-...",
  "exp": 1711820400,
  "iat": 1711816800,
  "sub_profile": "user",
  "cnf": { "jkt": "NzbLsXh8uDCcd7MNwrnNZpX0ak8ACQ" },
  "act": {
    "sub": "https://agents.enterprise.example/travel-assistant",
    "iss": "https://as.enterprise.example",
    "sub_profile": "ai_agent"
  }
}
~~~

The top-level `sub_profile` classifies the assertion's `sub`; the `sub_profile` within `act` classifies the actor.  The top-level `cnf.jkt` binds the assertion to the agent's DPoP key.

In this example, the receiving AS trusts the enterprise AS both to issue the grant and to assert the actor identifier pair.

This document defines two issuer patterns:

*  an AS-issued delegated assertion, where the assertion's `iss` is a trusted AS and `act.sub` identifies the actor (recommended);
*  an assertion carrying a pre-existing nested `act` chain, where the current JWT `iss` is a trusted AS carrying forward prior actor assertions.

For actors registered in the issuing AS's own namespace, `act.iss` ([Actor Object Structure](#actor-object-structure)) is often the AS's own issuer URI.

A deployment MAY additionally accept a self-issued actor assertion when explicitly enabled by another specification or local policy, but that behavior is outside the interoperable scope of this document.  Implementations MUST reject self-issued assertion grants by default; see [Self-Issued Authorization Grants](#security-self-issued-grants) for the security controls any such deployment needs to establish independently.

## Authorization Grant Processing {#jwt-assertion-grants-processing}

When an AS receives a JWT assertion grant containing an `act` claim:

1.  The AS MUST validate the assertion per {{RFC7523}}, including its signature and its `iss`, `sub`, `aud`, `exp`, and `jti` claims.

    *  **Grants without an enforced grant-level sender constraint**: The AS MUST reject with `invalid_grant` an assertion whose validated (`iss`, `jti`) pair it has already accepted, for as long as the assertion remains acceptable, including any allowed clock skew.  Such a grant is therefore single-use under this profile, including an ID-JAG that {{Section 4.4.3 of I-D.ietf-oauth-identity-assertion-authz-grant}} would let a client re-submit.  A proof used only for client authentication or to bind the issued token is not a grant-level sender constraint, and neither is a top-level `cnf` that presenter rebind supersedes ([JWT Assertion Grant as subject_token](#jwt-assertion-grant-as-subject-token)).
    *  **Sender-constrained grants**: When the AS validates the request's proof of possession against the assertion's top-level `cnf` claim as specified in step 6, the AS SHOULD additionally apply replay prevention to the validated (`iss`, `jti`) pair as defense in depth.  Any permitted reuse requires validation of that binding on each redemption and remains subject to the single-use requirements in [Self-Issued Authorization Grants](#security-self-issued-grants) or the applicable grant profile.

2.  The AS MUST verify that the assertion's `iss` is trusted under local policy to assert delegation on behalf of the actor identified by `act.sub`.

3.  The AS MUST verify that the assertion's `iss` is trusted under local policy to assert the (`act.iss`, `act.sub`) actor identifier pair.

    *  If `act.iss` is absent: when policy or metadata requires profile conformance, reject with `invalid_grant`; otherwise, apply [Migration and Adoption](#migration-and-adoption).
    *  If the assertion's `iss` is not trusted to assert the actor identifier pair: reject with `invalid_grant`.

4.  The AS MUST evaluate whether the identified actor is authorized to exercise delegation on behalf of `sub`.  The required strength of that evaluation depends on how the outermost actor was introduced:

    *  **New actor**: When the request supplies an `actor_token` or self-issued assertion that introduces a new `act.sub` not carried by the inbound chain, the AS MUST confirm the delegation relationship under local policy (for example, a pre-registered grant, an explicit consent record, or a policy rule).
    *  **Preserved chain**: When the request preserves an existing chain from a validated, trusted upstream issuer, the issuer trust established in step 2 provides the baseline assurance; the AS SHOULD additionally evaluate under local policy but is not required to do so for baseline interoperability.

    In either case:

    *  If the delegation relationship is prohibited by AS policy or cannot be confirmed: reject with `actor_unauthorized`.

5.  If the inbound assertion's `act` object contains a nested `act` object (indicating that the asserted actor is itself a delegatee), the AS MUST handle the inner chain as follows:

    *  **Propagation decision**: The AS SHOULD propagate the inner chain by preserving its nested structure, provided the resulting chain depth does not exceed the limit in [Delegation Chains](#delegation-chains).  If the AS does not accept pre-chained assertions, it MUST reject the request.

    *  **Prior-actor context**: For inner `act` objects, [Carry Prior-Actor Context](#carry-prior-actor-context) applies.

6.  The AS MUST verify proof of possession according to the token-endpoint mechanism in use and the top-level `cnf` semantics in [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation).

    *  **DPoP**: When the inbound assertion grant is DPoP-bound, it MUST carry a top-level `cnf.jkt`; reject with `invalid_grant` if absent.  The AS MUST:
       *  Verify that the DPoP proof is valid per {{RFC9449}} with `htm="POST"` and `htu` equal to the AS token endpoint URI.
       *  Verify that the JWK SHA-256 thumbprint of the public key in the DPoP proof matches the assertion's `cnf.jkt` ({{Section 6.1 of RFC9449}}), as in the proof checks of {{Section 4.3 of RFC9449}}.
       *  Use the assertion's `cnf.jkt` as set by the upstream issuer; MUST NOT substitute a locally registered key.
       *  Reject with `invalid_grant` if the required proof is absent, as specified for ID-JAG in {{Section 9.8.1.2.2 of I-D.ietf-oauth-identity-assertion-authz-grant}}.  If a valid proof's key does not match the assertion's `cnf.jkt`, reject with `invalid_grant`.
       *  Reject with `invalid_dpop_proof` if a supplied proof is invalid, subject to the nonce challenge rules of {{Section 8 of RFC9449}}.

       > Note: The `ath` claim is not applicable at the token endpoint and MUST NOT be required.  See also {{I-D.parecki-oauth-jwt-dpop-grant}} for related work on DPoP-bound JWT grants.

    *  **mTLS**: When the inbound assertion grant is mTLS-bound, it MUST carry a top-level `cnf.x5t#S256`; reject with `invalid_grant` if absent.  The AS MUST:
       *  Validate the client certificate presented at the token endpoint against `cnf.x5t#S256`.
       *  Use the `cnf.x5t#S256` value set by the upstream issuer; MUST NOT substitute a locally registered certificate.
       *  Reject with `invalid_grant` if the presented certificate does not match `cnf.x5t#S256`.
    *  When this JWT assertion grant is later used as a `subject_token` in Token Exchange, presenter continuation and presenter rebind are determined by [Presenter Transition Model](#token-exchange-presenter-model) and [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation), not by the contents of nested `act` objects.  In presenter rebind, the AS does not perform this step's match against the grant's top-level `cnf` claim ([JWT Assertion Grant as subject_token](#jwt-assertion-grant-as-subject-token)).

7.  If the assertion or authenticated request context identifies an OAuth client separately from `act.sub`:

    *  The AS MAY use that client identity as an additional authorization input.
    *  The AS SHOULD NOT infer that the client is authorized to act on behalf of the subject solely because the client initiated the request.
    *  When local policy maps the client identity to an actor identifier expected to match `act.sub`, the AS SHOULD perform identifier reconciliation before issuing a token.  If reconciliation cannot be established, the AS treats the identifiers as distinct and rejects the request when issuance requires them to identify the same entity, as defined for Identifier Reconciliation in [Conventions and Definitions](#conventions).

8.  If the AS accepts the assertion, it MUST propagate the actor information into the issued token according to the rules for the output token type.  For JWT access tokens, see [JWT Access Token Output](#jwt-access-token-propagation).  For Transaction Tokens, see [Transaction Token Output Rules](#transaction-token-output-rules).  When the output is another JWT assertion grant profile, the resulting assertion MUST preserve the validated actor information subject to local policy and the chain-depth limit in [Delegation Chains](#delegation-chains).

# JWT Access Tokens {#jwt-access-tokens}

## Structure {#jwt-access-tokens-structure}

A delegated JWT access token is a JWT access token per {{RFC9068}} that carries an `act` claim conforming to the actor profile defined in [Actor Profile for Delegation](#actor-profile).  Claims not listed here follow {{RFC9068}} and any other applicable token profile.

The following claims are defined for a JWT access token that carries actor-profile delegation:

`iss` (REQUIRED):
: Identifies the access token issuer.

`sub` (REQUIRED):
: Identifies the principal on whose behalf the access token is issued.

`sub_profile` (RECOMMENDED):
: Classifies the entity type of `sub`.  MUST conform to {{Section 4.2 of I-D.mora-oauth-entity-profiles}}.

`act` (REQUIRED when the token represents delegation per [Delegation Chains](#delegation-chains)):
: The actor object identifying the entity exercising the subject's delegated rights.  MUST conform to the actor object structure defined in [Actor Profile for Delegation](#actor-profile).

`cnf` (REQUIRED when sender-constrained; otherwise OPTIONAL):
: Binds the access token to the current presenter when a sender-constraining mechanism such as DPoP or mTLS is used.

`client_id` (REQUIRED):
: Identifies the OAuth client that requested the token, per {{RFC9068}}.  It does not substitute for `act`; see [Client Identity and Delegation](#client-identity-delegation).

`azp` (OPTIONAL):
: An additional client identifier used by some deployments.  It does not substitute for `act`; see [Client Identity and Delegation](#client-identity-delegation).

If an issuer uses `azp` and `act.sub` to identify the same party, [Client Identity and Delegation](#client-identity-delegation) defines their reconciliation, along with the other common rules; [Migrating from Implicit to Explicit Delegation](#migration-implicit-explicit) describes rollout.

The following is an example of a JWT access token with actor-profile claims:

~~~json
{
  "iss": "https://as.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "client_id": "travel-assistant-client-id",
  "azp": "https://agents.enterprise.example/travel-assistant",
  "aud": "https://api.travel-provider.example",
  "jti": "xyz987",
  "exp": 1711820400,
  "iat": 1711816800,
  "scope": "travel:book",
  "sub_profile": "user",
  "cnf": {
    "jkt": "NzbLsXh8uDCcd7MNwrnNZpX0ak8ACQ"
  },
  "act": {
    "sub": "https://agents.enterprise.example/travel-assistant",
    "iss": "https://as.enterprise.example",
    "sub_profile": "ai_agent"
  }
}
~~~

The top-level `cnf.jkt` binds this token to the actor's DPoP key.  The client and actor identifiers illustrate different namespaces; any equivalence depends on trusted local mappings.

## Delegated Token Issuance {#delegated-token-issuance}

When an AS issues, outside Token Exchange, a JWT access token whose delegation rests on an independent delegation basis ([Delegation Chains](#delegation-chains)), it MUST establish that basis for the (`sub`, actor) relationship before including `act`.  Examples include a pre-registered delegation grant, an explicit consent record, or a policy rule covering the acting party or a class of acting parties.

A client registration MAY supply that basis only if it uniquely identifies one acting entity and the AS can derive the actor identifier from the registration alone.  A registration shared by several actors does not satisfy this condition.

For the authorization code grant, the AS MAY include the `act` claim when an independent delegation basis, such as authorization state, registration, consent, or local policy, establishes that the OAuth client is acting as a distinct actor for the resource owner.  The actor identity MUST derive from that delegation basis.  Without such a basis, the AS MUST NOT include `act`.

An AS that issues a refresh token together with a token carrying `act` MUST record that chain for the grant.  For the `refresh_token` grant ({{Section 6 of RFC6749}}), the AS MUST carry the `act` chain recorded for the grant into the refreshed token unchanged or reject the request; it MUST NOT issue a refreshed token without that chain.

This document defines no actor-selection or actor-proof parameter for the authorization code grant; actor determination there is deployment-specific.

# Token Exchange Processing {#token-exchange-processing}

This section defines input processing for Token Exchange {{RFC8693}} and the issuance of JWT assertion grants and JWT access tokens.  [Transaction Token Service Processing](#transaction-token-service) defines Transaction Token issuance.  [Error Responses](#actor-profile-error-responses) applies to all paths in this section.

This profile defines three JWT-based `actor_token` credential types.  JWT access tokens use `actor_token_type=urn:ietf:params:oauth:token-type:access_token` ([JWT Access Token as actor_token](#jwt-access-token-as-actor-token)).  Client assertions per {{RFC7523}} ([JWT Client Assertion](#jwt-client-assertion-as-actor-token)) and workload identity credentials ([Workload Credential Processing](#workload-identity-as-actor-token)) both use `actor_token_type=urn:ietf:params:oauth:token-type:jwt`.

For JWT `actor_token` inputs, the AS identifies the credential profile as follows:

*  A JWT also presented as `client_assertion` with a `client_assertion_type` of `urn:ietf:params:oauth:client-assertion-type:jwt-bearer` is a client assertion when its `sub` equals the authenticating client's `client_id`.
*  A JWT `actor_token` not presented as `client_assertion` is a client assertion when its `iss` and `sub` both equal the authenticated client's `client_id` ({{Section 5.2 of RFC7521}}).
*  If `sub` differs from `client_id`, the AS MUST NOT classify the JWT as a client assertion solely because it appears in `client_assertion`.  It MUST apply workload identity credential processing if that profile matches, or reject with `invalid_request`.
*  If exactly one supported actor-credential profile cannot be identified, the AS MUST reject with `invalid_request`.
*  If `client_assertion` and `actor_token` are different JWTs, the AS MUST process each independently.  These disambiguation rules apply only to `actor_token`.

## Presenter Transition Model {#token-exchange-presenter-model}

For PoP migration, this profile distinguishes two classes of `subject_token` input according to whether they carry presenter-continuity information; [Input Processing](#token-exchange-input-processing) classifies each input type.

Identity-only inputs (ID Tokens and refresh tokens) establish `sub` and MAY establish supporting subject state such as `sub_profile` or an authorization ceiling.  They do not establish presenter continuity.  An ID Token establishes no inbound `act` state; a refresh token establishes it only from an `act` chain recorded in its state ([Refresh Token](#refresh-tokens)).  Neither input by itself justifies adding a new actor.

Token-state inputs (JWT assertion grants, JWT access tokens, and Transaction Tokens) establish `sub` and MAY establish `sub_profile`, inbound `act` chain state, and current-presenter binding through top-level `cnf`.  They are the only `subject_token` inputs from which this document defines presenter continuation.

A Token Exchange is delegated when it establishes a new actor or carries an inbound `act` chain, including one recorded in the state of a refresh token presented as `subject_token` ([Delegation Chains](#delegation-chains)).  A delegated Token Exchange operates in exactly one of two presenter-transition modes:

*  **Presenter continuation**: neither a validated `actor_token` nor the [`may_act`](#may-act) path without `actor_token` establishes a new presenter.  The issued token retains the presenter of a token-state `subject_token`: the holder of its top-level `cnf` binding or, for a bearer `subject_token`, its authenticated outermost actor or subject, as [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation) requires.
*  **Presenter rebind**: a validated `actor_token`, or the authenticated client on the [`may_act`](#may-act) path without `actor_token`, establishes a new presenter for the issued token.  When the output token is sender-constrained, its top-level `cnf` is bound to that new presenter.

A delegated request that satisfies neither mode MUST be rejected with `invalid_request`.  A request that supplies an `actor_token`, or whose `subject_token` carries `act`, is processed as delegated; if that processing fails, the AS rejects the request and MUST NOT instead issue a token without `act`.

The AS processes a request that is not delegated under {{RFC8693}} and local policy, and a token issued for it omits `act`.  Local policy can reject such a request with `actor_unauthorized`, for example when the client or resource requires explicit delegation.

When a request that is not delegated presents a `subject_token` carrying a top-level `cnf` claim, the requester MUST prove possession of that binding, as in [Token Exchange Continuation](#token-exchange-continuation).

Outside the [`may_act`](#may-act) path without `actor_token`, presenter rebind requires a **direct presenter credential**: an `actor_token` whose top-level `sub` names the new presenter.  Other means of establishing a presenter are outside this profile.

Identity-only inputs cannot support continuation, and bearer inputs support only bearer continuation.  To preserve a delegation chain while changing presenters, deployments SHOULD present the delegated credential as `subject_token` and a separate direct credential as `actor_token`.

JWT assertion grants are not suitable for use as `actor_token`, because their `sub` identifies the subject of delegation rather than the acting party.  Requests that need to establish an agent, workload, or client as the actor SHOULD use one of the actor credential types defined in this section instead.

## Input Processing {#token-exchange-input-processing}

When a Token Exchange request ({{RFC8693}}) presents a `subject_token` or `actor_token` of a type defined in this section, the AS MUST apply the following steps, using the input type's row in the tables below and the rules in its subsection.  Steps 1 through 4 apply to each input; steps 5 and 6 apply to the issued token.

1.  The AS MUST validate the input per the specification in its table row.  If validation fails, the AS MUST reject the request with `invalid_request` ({{Section 2.2.2 of RFC8693}}), unless the input's subsection or [Error Responses](#actor-profile-error-responses) specifies another error.

2.  Where the table row gives an issuer-trust check, the AS MUST verify that the input's issuer is trusted under local policy as that row states.  If not, the AS MUST reject the request with `invalid_request`.

3.  The AS MUST apply [Presenter Transition Model](#token-exchange-presenter-model): for a delegated request, the continuation or rebind rules in [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation), including any type-specific proof rule in the input's subsection.  A token-state `subject_token` without a top-level presenter binding can still be used for presenter rebind.

4.  The AS establishes inbound state as follows:

    *  From a validated JWT access token or Transaction Token `subject_token`, the AS MUST extract `sub`, `sub_profile` (if present), and `act` (if present) as the inbound delegation state for [JWT Access Token Output](#jwt-access-token-propagation).  A validated JWT assertion grant establishes the same inbound state.
    *  An identity-only `subject_token` supplies no presenter continuity.  A refresh token whose state records an `act` chain supplies that chain as inbound delegation state ([Refresh Token](#refresh-tokens)); an ID Token supplies none.  The AS establishes a new actor from a validated `actor_token` or from the authenticated client on the [`may_act`](#may-act) path without `actor_token`; with neither a new actor nor inbound delegation state, the request is not delegated, and any token issued for it MUST omit `act`.
    *  For an `actor_token`, the AS MUST derive the outermost actor and handle any inbound chain as specified in [Actor Tokens](#actor-tokens).

5.  The AS can reduce scope under local policy.  The effective scope of the issued token MUST NOT exceed the scope ceiling, if any, in the `subject_token`'s table row.

6.  The AS MUST then apply the propagation rules in [JWT Access Token Output](#jwt-access-token-propagation) to determine the remaining claims in the issued token.

For `subject_token` inputs:

| Input | Validated per | Issuer trust check | Scope ceiling |
|---|---|---|---|
| JWT assertion grant | [Authorization Grant Processing](#jwt-assertion-grants-processing) | In validation | See subsection |
| JWT access token | {{RFC9068}}, `aud` relaxed | Trusted for its delegation chain | Its effective scope |
| Transaction Token | [Transaction Token specification](#txn-token-as-subject-token) | Trusted issuer | See subsection |
| ID Token | {{OpenID.Core}}, local policy | No separate check | None |
| Refresh token | Token store or trusted back-channel | No separate check | Its authorized scope |

For `actor_token` inputs, each of which establishes the outermost actor:

| Input | Validated per | Issuer trust check |
|---|---|---|
| JWT client assertion | {{RFC7523}} | See subsection |
| Workload identity credential | Its type specification | Trusted for the workload's identity |
| JWT access token | {{RFC9068}}, `aud` relaxed | Trusted for the acting party's identity |

## Subject Tokens

SAML assertions are outside the scope of this document.

### Token-State Subject Tokens

#### JWT Assertion Grant {#jwt-assertion-grant-as-subject-token}

A JWT assertion grant presented as `subject_token` establishes `sub` and, when present, `sub_profile`, inbound `act` chain state, and top-level `cnf`, and is processed under [Input Processing](#token-exchange-input-processing) with the following rules.  Validation applies the inbound validation rules of [Authorization Grant Processing](#jwt-assertion-grants-processing) to the grant; assertion-grant output construction from that section does not apply.  On this Token Exchange path, a failure that section rejects with the `invalid_grant` error code is rejected with the `invalid_request` error code instead ({{Section 2.2.2 of RFC8693}}).

In presenter continuation, step 6 of [Authorization Grant Processing](#jwt-assertion-grants-processing) applies.  In presenter rebind, the new presenter's binding supersedes the grant's: the AS does not perform step 6's match of the request's proof against the grant's top-level `cnf`, and it treats the grant as having no enforced grant-level sender constraint, so the (`iss`, `jti`) single-use rule in step 1 of [Authorization Grant Processing](#jwt-assertion-grants-processing) applies.  The AS still validates any proof that the new presenter's credential profile or the deployment requires, as the rebind rules specify.

The grant's scope ceiling is its `scope` claim when present (such as the ID-JAG `scope` claim of {{I-D.ietf-oauth-identity-assertion-authz-grant}}), otherwise the scope the AS would authorize if the grant were redeemed directly under [Authorization Grant Processing](#jwt-assertion-grants-processing).

#### JWT Access Token {#jwt-access-token-as-subject-token}

A JWT access token presented as `subject_token` (`subject_token_type=urn:ietf:params:oauth:token-type:access_token`) establishes `sub` and, when present, `sub_profile`, inbound `act` chain state, and top-level `cnf`, and is processed under [Input Processing](#token-exchange-input-processing) with the following rules.  Validation follows {{Section 4 of RFC9068}} for its `typ` header, signature, `iss`, and `exp`, and {{RFC7519}} for `nbf` when present; the token carries the `sub` and `jti` claims that {{Section 2.2 of RFC9068}} requires.  Because a JWT access token used as `subject_token` was issued for a resource server, its `aud` does not ordinarily include the Token Exchange AS's token endpoint; the AS MUST NOT reject the inbound token solely because its `aud` does not include the AS's token endpoint URI.

#### Transaction Token {#txn-token-as-subject-token}

A Transaction Token presented as `subject_token` (`subject_token_type=urn:ietf:params:oauth:token-type:txn_token`) establishes `sub` and, when present, `sub_profile`, inbound `act` chain state, and a top-level presenter binding.  The AS or TTS receiving it MUST apply steps 1 through 4 of [Input Processing](#token-exchange-input-processing) with the following rules; steps 5 and 6 apply only when the output is a JWT access token, and for a Transaction Token output, [Transaction Token Output Rules](#transaction-token-output-rules) apply instead.

Validation per {{I-D.ietf-oauth-transaction-tokens}} covers the signature, `aud`, `exp`, and issuer identity:

*  When the Transaction Token carries `act`, a top-level `iss` claim MUST be present, and the AS MUST validate it as the token issuer.  If `iss` is missing, the AS MUST reject the request with `invalid_request`.
*  When the Transaction Token carries neither `act` nor `iss`, the AS MUST determine the issuer through the Transaction Token trust-domain rules and local configuration.
*  If validation fails or the issuer cannot be established, the AS MUST reject with `invalid_request`.

In step 3, the AS uses the presenter-proof mechanism defined by the deployment profile; {{I-D.ietf-oauth-transaction-tokens}} defines none.

The `req_wl` claim identifies the workload that requested the Transaction Token from the TTS.  For a JWT access token output, the AS MAY use `req_wl` for audit or local policy decisions but MUST NOT carry it forward into the issued JWT access token.  The scope ceiling is the authority the Transaction Token represents; when the AS cannot determine that authority in the output's scope vocabulary, it MUST reject the request with `invalid_scope`, as {{Section 13.14 of I-D.ietf-oauth-transaction-tokens}} requires of a TTS.  Other Transaction Token field semantics remain defined by {{I-D.ietf-oauth-transaction-tokens}}, {{RFC8693}}, and local policy.

### Identity-Only Subject Tokens

#### OpenID Connect ID Token {#id-tokens}

##### Overview {#id-token-overview}

An OpenID Connect ID Token {{OpenID.Core}} presented as `subject_token` (`subject_token_type=urn:ietf:params:oauth:token-type:id_token`) establishes subject identity only.  It identifies an authenticated user in `sub` and the relying party in `aud` (and possibly `azp`); the acting party comes from `actor_token`, and `aud` and `azp` remain client identifiers.

##### Processing {#id-token-as-subject-token}

The AS MUST verify that the ID Token's `aud` contains a value that identifies the authenticated client under Identifier Reconciliation ([Conventions and Definitions](#conventions)), as {{Section 4.3.3 of I-D.ietf-oauth-identity-assertion-authz-grant}} requires for an ID-JAG.  It MUST reject the request with `invalid_request` when the client is not authenticated or no `aud` value identifies it.  These checks on `aud` and `azp` remain OpenID Connect and client-identity checks; they do not by themselves establish the delegated actor under this profile.

The AS MUST use the validated ID Token's `sub` as the subject identity for the issued token, subject to the same-subject preservation rule in [JWT Access Token Output](#jwt-access-token-propagation).  The AS SHOULD set `sub_profile` to `user` in the issued token if it can authoritatively classify the ID Token's `sub` as a human user identity and no conflicting subject classification is available under local policy.

Because an ID Token carries no OAuth scope ceiling, scope determination comes from the `actor_token` (if any), {{RFC8693}}, and local policy.

#### Refresh Token {#refresh-tokens}

##### Overview {#refresh-token-overview}

A refresh token presented as `subject_token` (`subject_token_type=urn:ietf:params:oauth:token-type:refresh_token`) establishes `sub`, an authorized scope, and any `act` chain recorded for its grant, which the AS obtains from trusted server state rather than by extracting actor claims from the token.  A refresh token MAY be used as `subject_token` when the AS can validate its state directly or through a trusted back-channel to its issuer.  It MUST NOT be treated as a portable cross-domain delegation artifact or used as `actor_token`.  Client binding, cross-client presentation, and cross-AS acceptance policies remain deployment-specific.

##### Processing {#refresh-token-as-subject-token}

The AS validates the refresh token through its token store or a trusted back-channel to its issuer; signature validation alone is insufficient, and an expired, revoked, or otherwise invalid token fails validation.  For a token issued by another AS, the AS MUST NOT accept it unless that back-channel provides the subject, client binding, authorized scope, revocation state, and any `act` chain recorded for the grant.

The AS MUST verify that the requesting client or authenticated presenter is authorized to use the refresh token under the refresh token's client-binding, sender-constraint, rotation, and cross-client presentation policy.  If the requester is not authorized to use the refresh token, the AS MUST reject the request with `invalid_request`.

The AS MUST extract the subject identity and authorized scope associated with the refresh token from its token store or other trusted refresh-token state.  The `sub` of the user associated with the refresh token becomes `sub` in the issued token.

Whether an AS issues refresh tokens for delegated JWT assertion grant requests, how it revokes them, and whether their later use requires re-presenting upstream delegation artifacts remain deployment-specific.  On Token Exchange, a refresh token `subject_token` establishes no new actor.  When the refresh token's state records an `act` chain, the AS MUST treat that chain as inbound `act` state: the request is delegated, an identity-only input cannot use presenter continuation, so the request requires presenter rebind ([Presenter Transition Model](#token-exchange-presenter-model)), and the AS MUST NOT issue a token without that chain.

## Actor Tokens {#actor-tokens}

The following rules apply to every `actor_token` type in this section:

1.  The credential MUST identify the acting party in its top-level `sub`.  If it carries `act`, the AS MUST reject with `invalid_request`.
2.  After validating the credential, the AS MUST use its `sub` as the new outermost `act.sub` and set `act.iss` to that identifier's namespace context: for a JWT client assertion, the issuer identifier of the AS at which the client is registered; for a workload identity credential, its `iss`, or its trust domain when `iss` is absent; for a JWT access token, its `iss`.
3.  If the `subject_token` carries a chain, the new actor takes precedence over its outermost actor.  Different identities are permitted for presenter rebind.  Local policy MAY require equivalence on paths that only confirm an existing actor; when such a restriction applies and no trusted mapping establishes equivalence, the AS MUST reject with `invalid_request`.

[Delegation Chain Validation and Construction](#delegation-chain-algorithm) governs nesting of the `subject_token` chain.

Deployments supporting sub-delegation SHOULD provision each potential presenter with a direct credential naming itself in `sub`.

### JWT Client Assertion {#jwt-client-assertion-as-actor-token}

#### Overview

A JWT client assertion per {{RFC7523}} presented as `actor_token` establishes an OAuth client's own identity as the acting party.  Per {{Section 5.2 of RFC7521}} and {{Section 3 of RFC7523}}, the assertion has `iss = sub = client_id` and is signed with the client's private key.  Two usage patterns arise:

*  A single client assertion is presented as both `client_assertion` and `actor_token`, making the authenticated client explicit in the issued token's `act` chain.
*  The client authenticates by another method (for example, `client_secret` or mTLS) and presents a separate client assertion naming itself as `actor_token`.

To establish a principal distinct from the OAuth `client_id` as the actor, the request MUST use a different actor credential type, such as a workload identity credential ([Workload Credential Processing](#workload-identity-as-actor-token)), whose `sub` names that distinct principal.

#### Processing

If validation of the client assertion per {{RFC7523}} fails and the `actor_token` is also used as `client_assertion` for client authentication in the same request, the AS MUST reject the request with `invalid_client`; if that validation fails otherwise, the AS MUST reject the request with `invalid_request`.

The AS MUST verify that the client assertion's `iss` is a client registered with the AS, that `sub` equals that client's `client_id`, and that local policy permits that client's assertion to be used as an actor credential.  If not, the AS MUST reject the request with `invalid_request`.

When the `actor_token` is the same JWT presented as `client_assertion` for client authentication in the same request, the AS MAY derive the actor identity from the already-authenticated client context rather than re-validating the `actor_token` separately, provided the result is an identical `act.sub` value.  Actor-profile-specific policy failures after successful client authentication follow [Error Responses](#actor-profile-error-responses), including `actor_unauthorized` for actor-authorization denials.

### Workload Identity Credential {#workload-identity}

#### Overview {#workload-identity-overview}

A workload identity credential is a JWT whose `sub` identifies a software workload, such as a service or agent; presented as `actor_token`, it establishes that workload as the acting party.  WIMSE credentials are defined in {{I-D.ietf-wimse-workload-creds}}.

The recommended pattern for agentic Token Exchange is:

*  `subject_token`: a JWT access token or JWT assertion grant carrying the user's `sub` and the delegation chain (`act`)
*  `actor_token`: a workload identity credential whose `sub` is the agent or service identity
*  Output: a JWT access token with the user as `sub` and the workload as the outermost `act.sub`

#### Processing {#workload-identity-as-actor-token}

For WIMSE workload identity credentials ({{I-D.ietf-wimse-workload-creds}}), validation follows the rules defined in that specification.  The issuer is identified by `iss` or, for a WIMSE credential without `iss` ({{Section 5.1 of I-D.ietf-wimse-workload-creds}}), by the trust anchors configured for the trust domain of its `sub` ({{Section 3 of I-D.ietf-wimse-workload-creds}}).

The AS MUST validate any proof the workload-credential profile requires, such as a WIMSE Workload Proof Token (WPT, {{I-D.ietf-wimse-wpt}}) per its specification, whether or not the output token is sender-constrained.  When the request uses this credential to establish a sender-constrained output token in presenter-rebind mode, the AS MUST also validate proof for the new presenter binding: the WPT, or a DPoP proof ({{RFC9449}}) over the token endpoint URI whose key is the confirmation key in the credential's `cnf` claim (`cnf.jwk` for a WIMSE credential).  If a required proof is absent or invalid, the AS MUST reject under [Error Responses](#actor-profile-error-responses), using the proof mechanism's error when specified and `invalid_request` otherwise.

### JWT Access Token {#jwt-access-token-as-actor-token}

#### Overview

A non-delegated JWT access token presented as `actor_token` establishes a service or workload as the acting party; its top-level `sub` identifies the acting party and satisfies the direct-presenter-credential requirement in [Presenter Transition Model](#token-exchange-presenter-model).

#### Processing

Validation per {{RFC9068}} applies the `aud` relaxation in [JWT Access Token as subject_token](#jwt-access-token-as-subject-token).  A JWT access token without top-level `cnf` is accepted as `actor_token` only when its `sub` corresponds to the authenticated client under Identifier Reconciliation ([Conventions and Definitions](#conventions)); otherwise, the AS MUST reject the request with `invalid_request`.  When the credential carries top-level `cnf`, the AS MUST validate proof for that binding per [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation), whether or not the output token is sender-constrained.

## `may_act` {#may-act}

The `may_act` claim ({{Section 4.4 of RFC8693}}) pre-authorizes a specific party to act on behalf of the subject in a subsequent Token Exchange.  This document defines limited use of `may_act` as a delegation-authorization input when actor identity is established by other means: when present in a validated `subject_token`, it MAY satisfy the delegation-authorization check in step 3 of [Validate Outermost Actor](#validate-outermost-actor) without a separately pre-registered grant.  The `may_act` claim is never itself the source of actor identity, and it MUST NOT be propagated into any output token.

Two preconditions apply regardless of how the Token Exchange request is structured:

1.  The `subject_token` issuer is trusted under local policy to assert `may_act` on behalf of the subject.
2.  The canonical `may_act` identifier matches the derived actor identity under Identifier Reconciliation ([Conventions and Definitions](#conventions)).

The canonical `may_act` identifier is (`may_act.iss`, `may_act.sub`) when `may_act` carries `iss`, and (`subject_token.iss`, `may_act.sub`) otherwise.  The AS MUST apply configured mapping rules and MUST NOT infer equivalence from naming similarity alone.

If `may_act.iss` is present but is not a valid StringOrURI, the AS MUST NOT use that `may_act` claim to authorize delegation and MUST NOT fall back to `subject_token.iss`.

Actor identity is established as follows:

*  **With `actor_token`**: the AS derives (`act.iss`, `act.sub`) under the credential's type-specific rules and reconciles that pair with the canonical `may_act` identifier.  The `may_act` claim MUST NOT override the derived actor.
*  **Without `actor_token`**: the requesting client MUST be a confidential client that has authenticated in the request; public clients MUST NOT use this path.  The AS reconciles the authenticated client with the canonical `may_act` identifier and sets `act.sub` to the client's canonical identifier.  The AS MUST set `act.iss` to the issuer or namespace context that locally registered the client, typically the AS's own issuer URI.  The authenticated client is then the new presenter in presenter-rebind mode ([Presenter Transition Model](#token-exchange-presenter-model)), and a sender-constrained output is bound to the key or certificate the client demonstrates in the request.

When an `actor_token` establishes the actor and `may_act` is absent or the conditions above are not met, the AS MUST satisfy the delegation-authorization check through another recognized basis (a pre-registered grant, a consent record, or an applicable policy rule).  Without an `actor_token`, such a basis establishes no actor ([Delegation Chains](#delegation-chains)).  The AS MUST NOT treat the presence of `may_act` alone as authorization for any actor other than the one whose canonical identity matches it.

## Output Token Rules

### JWT Assertion Grant Output {#jwt-assertion-grant-issuance}

When `requested_token_type` requests a JWT assertion grant, the output MUST satisfy [JWT Assertion Grant Structure](#jwt-assertion-grants-structure).  Supported type identifiers include `urn:ietf:params:oauth:token-type:jwt` and compatible profiles of that type, such as `urn:ietf:params:oauth:token-type:id-jag` {{I-D.ietf-oauth-identity-assertion-authz-grant}}.

The AS MUST:

*  Construct the chain per [JWT Access Token Output](#jwt-access-token-propagation).
*  Set `aud` from the request or deployment configuration: for an ID-JAG, to the Resource Authorization Server's issuer identifier ({{Section 3.1 of I-D.ietf-oauth-identity-assertion-authz-grant}}); otherwise, to a value identifying the downstream authorization server that {{Section 3 of RFC7523}} permits, such as its token endpoint URL.
*  Sign the assertion, per {{Section 3 of RFC7523}}.

Issuing such a grant is subject to AS configuration and, when the grant carries `act`, to [Validate Outermost Actor](#validate-outermost-actor).

### JWT Access Token Output {#jwt-access-token-propagation}

When the output is a JWT access token, the issued token MUST satisfy [JWT Access Token Structure](#jwt-access-tokens-structure).  After the applicable grant, subject-token, actor-token, or TTS input processing, the AS MUST apply the rules below.

For a sender-constrained output of a delegated Token Exchange request, the AS MUST set the top-level `cnf` claim according to [Presenter Transition Model](#token-exchange-presenter-model): retain the presenter's binding in continuation mode, or bind to the new presenter in rebind mode.  For any other request, the applicable grant or token specification and its proof mechanism govern that binding, such as {{Section 9.8.1.1 of I-D.ietf-oauth-identity-assertion-authz-grant}} for an ID-JAG.

If a Token Exchange request explicitly seeks a delegated output, for example by supplying an `actor_token` or by presenting a `subject_token` that already carries `act`, and the AS cannot validate the actor information, it MUST reject the request with `invalid_request`.  If the AS can validate the actor information but cannot establish or confirm the required delegation basis, or if local policy prohibits the relationship, it MUST reject the request with `actor_unauthorized`.  The AS MUST NOT issue a non-delegated token in place of the requested delegated output.

1.  The AS includes or omits `act` as required by [Delegation Chains](#delegation-chains), and does not silently drop inbound actor information ([Omit `act`](#omit-act)).

2.  The AS MUST preserve `sub` to refer to the same underlying subject as the inbound token.  If the AS uses a different subject-identifier namespace, it MAY change the `sub` value only to re-express that same subject in the new namespace under a trusted local mapping.  The AS MUST NOT replace `sub` with an identifier for a different subject.  [Subject Namespace Translation](#subject-namespace-translation) describes subject-namespace translation requirements and relying-party consequences.

3.  The AS MUST construct the `act` claim using the construction decision order in [Delegation Chain Validation and Construction](#delegation-chain-algorithm): extend with a new actor, preserve an existing chain, or omit `act`, in that order.  An actor derived from `actor_token` is asserted by the issuing AS; consumers MUST NOT infer that it was present in the `subject_token` or endorsed by its issuer.

4.  The AS MUST reject the request if actor validation fails or the resulting chain exceeds the depth limit, using [Error Responses](#actor-profile-error-responses).  It MUST NOT issue a partially preserved chain.

5.  Top-level `sub_profile` follows [Actor Object Structure](#actor-object-structure), which recommends it when the AS can authoritatively classify the token's `sub` entity type.  When the AS carries a trusted inbound top-level `sub_profile` into the issued token, it MUST preserve its unrecognized but syntactically valid values.

6.  The AS can reduce scope under local policy.  If this reduction, before any actor-based restriction, leaves no effective scope, it MUST reject with `invalid_scope`.

    If the AS also restricts scope using the (`sub`, `act.sub`) pair or `act.sub_profile`, it MUST return the final effective `scope` in the token response.  If this restriction leaves no scope, the AS MUST reject:

    *  with `actor_unauthorized` when the actor is categorically unauthorized for the remaining scope, for example because its entity type is prohibited;
    *  with `invalid_scope` for other causes, such as an actor scope ceiling that excludes the requested values.

7.  The AS MAY preserve inbound client identifiers per the output token profile or local policy.  Preserved values MUST retain their client-identity meaning; they do not represent delegation state ([Client Identity and Delegation](#client-identity-delegation)).  If preserving an optional identifier would create ambiguity about the delegated actor relationship, the AS SHOULD omit it.  JWT access tokens still require `client_id` per {{RFC9068}}.

8.  On a Token Exchange request, the AS MUST issue the token for every requested `audience` and `resource` or reject the request with `invalid_target` ({{Section 2.2.2 of RFC8693}}).  On other grants, resource indicators follow {{Section 2.2 of RFC8707}}.

# Transaction Token Service Processing {#transaction-token-service}

## Transaction Tokens {#transaction-tokens}

Transaction Tokens {{I-D.ietf-oauth-transaction-tokens}} are short-lived JWTs that capture the workload identity and request context for a series of related service calls within a single business transaction. A TTS, which is a specialized authorization server, issues them.

Transaction Token claims are defined in {{I-D.ietf-oauth-transaction-tokens}}.  This profile modifies or adds the following claims:

`iss` (OPTIONAL in {{I-D.ietf-oauth-transaction-tokens}}; REQUIRED by this profile when the token carries `act`):
: Identifies the Transaction Token issuer.  It MUST be present when the token carries `act`.  It MAY be omitted only when the token carries no `act` and all recipients know the issuer out of band.  In that case, recipients MUST identify the issuer using {{I-D.ietf-oauth-transaction-tokens}} and local configuration.

`req_wl`:
: This claim provides TTS-level workload context and is not a substitute for `act.sub`; see [Actor Claim in Transaction Tokens](#actor-claim-in-transaction-tokens).

`act` (REQUIRED when the token represents delegation per [Delegation Chains](#delegation-chains); omitted otherwise):
: Represents the current acting party and any prior delegation steps, and conforms to [Actor Object Structure](#actor-object-structure).  See [Actor Claim in Transaction Tokens](#actor-claim-in-transaction-tokens) for delegation semantics and the relationship between `act.sub` and `req_wl`.

### Actor Claim in Transaction Tokens {#actor-claim-in-transaction-tokens}

Under this profile, a Transaction Token represents delegation when a condition in [Delegation Chains](#delegation-chains) holds, that is, when an `actor_token` or an inbound `act` chain establishes that the workload acts for `sub`.  When no such condition holds, including when a workload acts under its own grant without any delegation basis, the token omits `act`.  The TTS MUST NOT infer delegation solely because `sub` and `req_wl` differ.

The `req_wl` claim identifies the workload that requested the token from the TTS.  The `act.sub` claim identifies the immediate acting party in the subject identifier namespace used by this profile.  The outermost `act.sub` is the authoritative actor identifier for authorization decisions under this document; `req_wl` is supporting workload context.

Claim semantics under this profile:

*  `sub`: identifies the original initiator.  A replacement Transaction Token keeps `sub` unchanged ({{Section 13.15 of I-D.ietf-oauth-transaction-tokens}}).
*  `act.sub` (outermost): identifies the immediate acting party.  When a TTS sets both `req_wl` and the new outermost `act.sub` in a single token issuance (presenter-rebind mode), it MUST ensure that they identify the same entity under local policy.  In presenter-continuation mode, `req_wl` identifies the requesting workload ({{Section 9.2 of I-D.ietf-oauth-transaction-tokens}}), which step 5 of [Transaction Token Output Rules](#transaction-token-output-rules) requires to correspond to the outermost actor.  A recipient that relies on both to identify the current presenter requires them to identify the same entity and therefore rejects the token when it cannot reconcile them ([Conventions and Definitions](#conventions)).
*  Inner `act` objects: identify prior presenters in the delegation path.  At each level, `act.sub_profile` classifies the entity type of that presenter.

The following example shows a Transaction Token after two hops:

~~~json
{
  "iss": "https://tts.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
  "scope": "inventory:check",
  "req_wl": "https://tools.travel-provider.example/booking-tool",
  "aud": "https://api.travel-provider.example",
  "txn": "550e8400-e29b-41d4-a716-446655440000",
  "exp": 1711816900,
  "iat": 1711816800,
  "tctx": {
    "action": "check-availability"
  },
  "rctx": {
    "req_ip": "203.0.113.42"
  },
  "cnf": {
    "jkt": "0ZcOCORZNYy9ZhHiZN..."
  },
  "act": {
    "sub": "https://tools.travel-provider.example/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service",
    "act": {
      "sub": "https://agents.enterprise.example/travel-assistant",
      "iss": "https://as.enterprise.example",
      "sub_profile": "ai_agent"
    }
  }
}
~~~
The booking tool is the current presenter, identified by `req_wl` and the outermost `act.sub` and bound to the top-level `cnf.jkt`.  The inner actor records the travel assistant's prior participation.

## Presenter Authentication and Transition

For a delegated request, the TTS applies the same two presenter-transition modes defined in [Presenter Transition Model](#token-exchange-presenter-model), but only for token-state `subject_token` inputs; that section also governs a request that is not delegated:

*  **Presenter continuation**: the authenticated requester is the same current presenter as the inbound token.  When the inbound token carries a top-level presenter binding, the TTS validates proof for that binding under the applicable deployment profile, because {{I-D.ietf-oauth-transaction-tokens}} defines no presenter-proof mechanism; a bearer inbound token yields a bearer Transaction Token, as in [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation).  In this mode, the TTS preserves the inbound `act` chain unchanged and MUST NOT add a new outermost `act`.
*  **Presenter rebind**: a validated `actor_token` that is a direct presenter credential establishes the current presenter for the issued Transaction Token.  In this mode, the TTS creates a new outermost `act` for that presenter and nests any inbound `act` chain beneath it.

Through presenter rebind with a validated `actor_token`, the TTS can upgrade a bearer input to a sender-constrained Transaction Token, as in [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation).

## Supported Subject Tokens

This profile defines TTS processing for whichever of the following inputs a TTS supports: JWT assertion grants, JWT access tokens, and Transaction Tokens.  This document does not define TTS processing of ID Tokens or refresh tokens.

For each accepted input, the TTS MUST apply the rules listed for it in the referenced sections:

| Input | Rules applied | Section |
|-------|---------------|---------|
| JWT assertion grant | Validation and presenter continuity | [Input Processing](#token-exchange-input-processing); [JWT Assertion Grant as subject_token](#jwt-assertion-grant-as-subject-token) |
| JWT access token | Validation and extraction | [Input Processing](#token-exchange-input-processing); [JWT Access Token as subject_token](#jwt-access-token-as-subject-token) |
| Transaction Token | Validation and extraction | [Input Processing](#token-exchange-input-processing); [Transaction Token as subject_token](#txn-token-as-subject-token) |

The resulting state (subject, classification, chain, and binding) is input to [Transaction Token Output Rules](#transaction-token-output-rules) instead of to JWT access token issuance.  For an inbound Transaction Token, the TTS maintains the call chain of requesting workloads as {{Section 13.15 of I-D.ietf-oauth-transaction-tokens}} requires, and the issued token's `req_wl` identifies the workload that requested it ({{Section 9.2 of I-D.ietf-oauth-transaction-tokens}}).  The `scope` claim of a Transaction Token can use a different vocabulary from that of the inbound token ({{Section 9.2 of I-D.ietf-oauth-transaction-tokens}}), so the literal scope-subset rules of [Input Processing](#token-exchange-input-processing) step 5 do not apply.  The TTS still ensures that the requested scope does not exceed the authority of the `subject_token` ({{Section 13.6 of I-D.ietf-oauth-transaction-tokens}}) and rejects the request when that authority cannot be determined ({{Section 13.14 of I-D.ietf-oauth-transaction-tokens}}).  A replacement Transaction Token cannot expand the scope of permitted actions ({{Section 13.15 of I-D.ietf-oauth-transaction-tokens}}).

## Transaction Token Output Rules {#transaction-token-output-rules}

The TTS applies [Delegation Chain Validation and Construction](#delegation-chain-algorithm) and [JWT Access Token Output](#jwt-access-token-propagation), except its JWT access token structure requirement and its step 6, with the Transaction Token adaptations below.  Transaction Token scope follows [Supported Subject Tokens](#supported-subject-tokens).

When a TTS receives a Token Exchange request to issue or refresh a Transaction Token from an inbound JWT assertion grant, JWT access token, or Transaction Token, it MUST apply the following rules; steps 3 through 6 apply only when the request is delegated ([Presenter Transition Model](#token-exchange-presenter-model)), including when an `actor_token` establishes delegation for an inbound token without `act`:

1.  The TTS preserves `sub` from the inbound token, as step 2 of [JWT Access Token Output](#jwt-access-token-propagation) requires.

    *  For an inbound JWT access token or JWT assertion grant, the TTS can re-express `sub` in a different identifier namespace only when a trusted local mapping establishes that both identifiers refer to the same underlying subject (for example, when crossing trust-domain boundaries in a federation scenario).  For an inbound Transaction Token, the replacement token keeps `sub` unchanged ({{Section 13.15 of I-D.ietf-oauth-transaction-tokens}}).

2.  The `req_wl` claim and any Transaction Token claims other than actor-profile claims remain governed by {{I-D.ietf-oauth-transaction-tokens}} and local policy.  Under this profile, `req_wl` is supporting workload context and MUST NOT be treated as a substitute for the outermost `act.sub`.

3.  The TTS applies [Enforce Depth Limit](#enforce-depth-limit) to the `act` chain that results from step 6.

4.  The TTS validates the inbound token and establishes issuer trust ([Validate Carrier Token](#validate-carrier-token)) before preserving or extending any `act` chain.  For the outermost `act` object in the inbound chain, the TTS applies [Validate Outermost Actor](#validate-outermost-actor), treating presenter rebind as extending the chain and presenter continuation as preserving it.

    For inner `act` objects in the inbound chain, [Carry Prior-Actor Context](#carry-prior-actor-context) applies.

5.  The TTS MUST determine whether the request is presenter continuation or presenter rebind:

    *  **Presenter continuation**: The TTS MUST authenticate the requester as the same current presenter as the inbound token.  When the inbound token carries `act`, the authenticated requester MUST correspond to the outermost (`act.iss`, `act.sub`) pair, or the TTS MUST reject the request with the `invalid_request` error code.  If local policy prohibits the preserved actor relationship, the TTS MUST reject the request with `actor_unauthorized`.
    *  **Presenter rebind**: The TTS MUST validate a direct presenter `actor_token` for the new presenter under [Actor Tokens](#actor-tokens).  Before creating a new outermost `act` object, the TTS MUST evaluate whether the newly authenticated presenter is authorized under local policy to act on behalf of `sub` for the requested transaction.  If the required actor relationship is prohibited by local policy, absent, or cannot be confirmed from the current inputs and policy, the TTS MUST reject the request with `actor_unauthorized`.

    If the current inputs satisfy neither the presenter-continuation nor the presenter-rebind requirements, the TTS MUST reject the request with the `invalid_request` error code.

6.  When the issued Transaction Token carries delegated actor information, it includes the top-level `iss` claim required by [Transaction Tokens](#transaction-tokens), identifying the TTS as its issuer, and the TTS MUST construct the `act` claim using [Delegation Chain Validation and Construction](#delegation-chain-algorithm).  In summary:

    *  in presenter-continuation mode, preserve the inbound chain unchanged ([Preserve Inbound Chain](#preserve-inbound-chain));
    *  in presenter-rebind mode, create a new outermost `act` object for the new presenter and nest any inbound chain beneath it ([Extend Chain with New Actor](#extend-chain-with-new-actor)).

7.  When the issued Transaction Token includes a top-level presenter-binding claim such as `cnf`, that binding applies to the current presenter.  Requester authentication follows {{Section 11.5 of I-D.ietf-oauth-transaction-tokens}}; the applicable deployment profile, not this document, defines the proof mechanism for that binding, because {{I-D.ietf-oauth-transaction-tokens}} defines none.

This document does not define TTS-specific `may_act` processing.  A deployment MAY use it as an authorization hint under local policy or another specification, but it MUST NOT replace credential validation, presenter authentication, or the rules in this document that govern whether a new outermost `act` is created.

# Resource Server Processing {#resource-server-processing}

## Actor Authorization {#actor-authorization}

When a token contains both a `sub` claim and an `act` claim, a resource server has two independent principals available for authorization policy:

*  **Subject principal** (`sub`): the party whose authorization is being exercised.  This principal typically has a relationship with the resource (e.g., an account, a role, or a permission).

*  **Actor principal** (`act.sub`): the party making the immediate request.  This principal can differ from the subject in organizational domain and trust level.  Wherever this document pairs `sub` with the outermost `act.sub` for authorization policy, the outermost actor is identified by its (`act.iss`, `act.sub`) pair ([Actor Object Structure](#actor-object-structure)).

For Transaction Tokens, the primary policy pair remains (`sub`, `act.sub`).  The `req_wl` claim provides workload context from the TTS and is not a substitute for `act.sub`.

Under this profile, actor authorization is conditional.  When an RS accepts a token as satisfying a delegated-access requirement, it MUST NOT ignore the `act` claim and authorize the request solely as if the token were non-delegated.  Whether or not it requires actor authorization, the RS SHOULD evaluate the (`sub`, outermost `act.sub`) pair according to local policy for authorization, audit, or trust decisions.  For security-sensitive delegated access, the RS SHOULD enforce authorization of the (`sub`, outermost `act.sub`) pair on every request.  Resource servers that receive delegated tokens should define and document their actor authorization policy.  The following steps describe one approach for resource servers that choose to enforce actor authorization policy:

1.  **Advertise delegated-token requirements**: An RS that wants to signal that delegated requests are expected to carry actor-profile information SHOULD set `actor_profile_required: true` ([Protected Resource Metadata](#protected-resource-metadata)).  An RS MAY still apply actor authorization without advertising it, but clients cannot rely on that behavior.

2.  **Evaluate subject authorization**: Determine whether `sub` has been granted the requested scope or permission, using the same mechanisms applied to non-delegated tokens.

3.  **Evaluate actor authorization**: Determine whether the (`sub`, outermost `act.sub`) pair is permitted for the requested operation.  This evaluation MAY be performed against:

    *  a registered delegation policy for the (subject, actor) pair,
    *  the actor's `sub_profile` (e.g., only AI agents from a trusted domain are permitted to act as delegatees),
    *  the token's `scope` claim.

    For Transaction Tokens, the RS SHOULD evaluate `req_wl` as supporting context.

4.  **Evaluate combined policy**: Apply resource-specific actor authorization policies (e.g., requiring both principals to have agreed to terms of service).

5.  If the RS requires actor authorization but cannot complete it, it MUST reject the request.

Nested actors are not inputs to actor authorization ({{Section 4.1 of RFC8693}}).

## JWT Access Token Processing {#jwt-access-token-rs-processing}

On a request path where delegated-token processing may apply, an RS MUST validate and process JWT access tokens according to its delegated-token policy.  A conforming `act` claim identifies a delegated token; the RS MUST NOT infer delegation from `client_id`, `azp`, or other client-identity claims alone.  [Protected Resource Metadata](#protected-resource-metadata) advertises delegated-token expectations; enforcement remains the responsibility of the RS.

When the resource server evaluates a JWT access token as a delegated token under local policy, it MUST:

1.  Validate the `typ` header, signature, `iss`, `aud`, and temporal claims per {{RFC9068}}, and reject a token whose visible `act` chain exceeds the configured maximum depth ([Delegation Chains](#delegation-chains)).  If the request path requires actor-profile conformance, including through `actor_profile_required: true`:

    *  A token evaluated as delegated MUST carry `act`, and every actor object in the visible `act` chain MUST include `iss`.  Otherwise, reject the token with HTTP 401 and `error="invalid_token"`.  This is structural validation, not independent authentication of historical actors.
    *  Non-delegated tokens need not carry `act`.

2.  If the token carries a top-level `cnf.jkt`, validate the accompanying DPoP proof per {{Section 7 of RFC9449}}.  If the token carries a top-level `cnf.x5t#S256`, validate the client certificate of the mutual-TLS connection against it per {{Section 3 of RFC8705}}.  If a DPoP proof is present but the token carries neither `cnf.jkt` nor `cnf.x5t#S256`, the RS MUST treat the token as a bearer token; the RS MUST NOT infer a confirmation binding from the DPoP proof key.

3.  Extract `sub` and the outermost `act.sub` as the two principals relevant for authorization policy.

4.  If the token carries `client_id`, `azp`, or both, treat them as client-identity inputs only.  The actor identifier is then `act.sub`, not `client_id` or `azp`.  When local policy expects both to identify the same acting party, the RS SHOULD perform identifier reconciliation; if reconciliation cannot be established, the RS treats them as distinct and rejects the request when its authorization decision requires them to identify the same party, as defined for Identifier Reconciliation in [Conventions and Definitions](#conventions).  See [Client Identity and Delegation](#client-identity-delegation).

5.  Apply actor authorization per [Actor Authorization](#actor-authorization) when required by local policy or when the token is accepted as satisfying a delegated-access requirement for the request path.

6.  The RS MAY traverse inner `act` objects for audit.  They are not inputs to access-control decisions ({{Section 4.1 of RFC8693}}).

7.  If any of the above steps fail, return an appropriate error response.  The HTTP authentication scheme used in the `WWW-Authenticate` challenge follows the token's binding mechanism: `Bearer` per {{Section 3.1 of RFC6750}} for bearer or mTLS-bound ({{RFC8705}}) tokens, or `DPoP` per {{Section 7.1 of RFC9449}} for DPoP-bound tokens.

    *  If signature, `iss`, `aud`, temporal, or depth validation fails: HTTP 401 with `error="invalid_token"`.
    *  If identifier reconciliation that the authorization decision requires fails (step 4): HTTP 403 with `error="actor_unauthorized"`.
    *  If DPoP proof validation for `cnf.jkt` fails: HTTP 401 per {{Section 7 of RFC9449}}.
    *  If the client certificate does not match `cnf.x5t#S256`: HTTP 401 with `error="invalid_token"`, per {{Section 3 of RFC8705}}.
    *  If actor authorization required by local policy fails for a structurally valid token: HTTP 403 with `error="actor_unauthorized"`, registered in [OAuth Error Registry](#iana-error-codes).  The RS MUST NOT use the `insufficient_scope` error code for this failure, because requesting broader scope does not resolve an actor-policy denial.
    *  The RS MUST NOT expose actor-specific rejection details outside the trust domain.

## Transaction Token Processing {#txn-token-rs-processing}

Upon receiving a Transaction Token on a request path where delegated-token processing may apply, a resource server MUST validate and process that token according to {{I-D.ietf-oauth-transaction-tokens}}, any applicable deployment profile, and the actor-profile rules in this document.

When the resource server evaluates a Transaction Token as a delegated token under local policy, it MUST:

1.  Validate the signature, audience, temporal claims, and issuer under {{I-D.ietf-oauth-transaction-tokens}} and the deployment profile:

    *  If the token carries `act`, the top-level `iss` claim MUST be present, and the RS MUST validate it as the token issuer.
    *  If the token carries neither `act` nor `iss`, the RS MUST determine the issuer through the Transaction Token trust-domain rules and local configuration.
    *  A token whose visible `act` chain exceeds the configured maximum depth ([Delegation Chains](#delegation-chains)) fails validation.
    *  If the request path requires actor-profile conformance, including through `actor_profile_required: true`, a token evaluated as delegated MUST carry `act`, and every actor object in the visible `act` chain MUST include `iss`.  If either is missing, reject the request.  This is structural validation, not independent authentication of historical actors.  Non-delegated tokens need not carry `act`.

2.  When the token carries a top-level presenter-binding claim such as `cnf`, validate the accompanying proof according to the applicable deployment profile, because {{I-D.ietf-oauth-transaction-tokens}} defines no presenter-proof mechanism.  The top-level presenter binding applies only to the current presenter.

3.  Extract `sub` and the outermost `act.sub` as the two principals relevant for authorization policy.  If `req_wl` is present, treat it as supporting workload context only.  The RS MUST NOT treat `req_wl` as a substitute for `act.sub`.  When local policy expects `req_wl` and the outermost `act.sub` to identify the same party, the RS SHOULD perform identifier reconciliation; if reconciliation cannot be established, the RS treats them as distinct and rejects the request when its authorization decision requires them to identify the same party, such as when it relies on both to identify the current presenter ([Actor Claim in Transaction Tokens](#actor-claim-in-transaction-tokens)).

4.  Apply actor authorization per [Actor Authorization](#actor-authorization) when required by local policy or when the token is accepted as satisfying a delegated-access requirement for the request path.

5.  Optionally traverse inner `act` objects to audit the full delegation chain.  They are not inputs to access-control decisions ({{Section 4.1 of RFC8693}}) and have the trust properties in [Carry Prior-Actor Context](#carry-prior-actor-context).

6.  If any of the above steps fail, the RS MUST reject the request through the deployment's Transaction Token handling, because {{I-D.ietf-oauth-transaction-tokens}} defines no error response.  Validation or presenter-proof failures are token-validation failures; failures of actor authorization required by local policy are authorization failures.  The RS MUST NOT include actor-specific rejection details in error responses exposed outside the trust domain.

## Token Introspection {#token-introspection}

When token introspection ({{RFC7662}}) is used for delegated tokens, an AS MUST expose actor-profile information needed for equivalent RS processing.  For an active delegated token whose authorization context includes actor-profile claims, the introspection response MUST include:

*  `active`: `true`, per {{Section 2.2 of RFC7662}}.
*  `sub`: REQUIRED.  The subject of the delegated token, as defined in {{RFC7662}}.
*  `act`: REQUIRED.  The actor object conforming to [Actor Object Structure](#actor-object-structure), including `act.sub`, `act.iss`, and any nested `act` chain, structured identically to the JWT form defined in this document.
*  `sub_profile`: REQUIRED when the token's authorization context includes a top-level `sub_profile`; otherwise SHOULD be included when the AS can authoritatively classify the subject entity type.
*  `scope`: REQUIRED.  The effective scope of the token.
*  `iss`: REQUIRED when the AS has a stable issuer identifier.
*  `chain_complete`: OPTIONAL.  When absent, the RS SHOULD treat the chain as complete unless local policy or deployment context indicates otherwise.

The AS MUST return actor claims from the token's authorization context, including the complete nested chain, except for the following privacy-filtering allowance.

If local privacy policy requires omitting inner actors, the AS MAY filter them but MUST include `"chain_complete": false`.

When `chain_complete` is `false`, the subject and the outermost actor remain available for authorization, and the RS records the incompleteness in any audit of the chain.

The RS MUST NOT treat a partial chain as complete delegation history.  Companion profiles with data aligned to `act` define their filtering behavior as required by [Companion Profiles and Extension Points](#companion-profile-extensibility).

When an AS supports delegated opaque access tokens through introspection, it MUST return the members listed above for active delegated tokens.  Support for this compatibility path MUST NOT be inferred solely from `actor_profile_required` metadata; see [Profile Scope](#profile-scope).

An introspecting RS MUST apply the same delegated-token processing as for equivalent locally validated JWT claims, including actor authorization when required by local policy.

If policy or token context indicates delegation, a missing `act` member is an inconsistency, and the RS MUST reject the token with HTTP 401 and `error="invalid_token"`; `actor_profile_required` alone does not indicate delegation.  Otherwise, the RS MAY treat an active response without `act` as non-delegated.

Introspection endpoints for delegated tokens SHOULD be advertised using the `introspection_endpoint` parameter in AS metadata ({{RFC8414}}).  When revocation is integrated, the introspection response for a revoked delegated token returns `"active": false` per {{Section 2.2 of RFC7662}} and MUST NOT include the `act` or `sub_profile` members.

Resource servers that cache introspection responses for delegated tokens should use short cache lifetimes consistent with revocation requirements.

# Error Responses {#actor-profile-error-responses}

When an AS or TTS rejects a request for reasons related to actor-profile processing, it MUST use the error mappings in this section and construct the response per {{Section 5.2 of RFC6749}}.

Input-validation errors depend on the request's grant type.  On Token Exchange requests, including TTS requests, an invalid or policy-unacceptable `subject_token` or `actor_token` uses the `invalid_request` error code ({{Section 2.2.2 of RFC8693}}).  On JWT bearer grant requests, an invalid assertion uses the `invalid_grant` error code ({{Section 3.1 of RFC7523}}).  This distinction also applies to missing required claims, invalid actor structure, and excessive inbound chain depth.

Client-authentication and proof-mechanism errors take precedence over these generic input-validation errors.  Failed client authentication uses the `invalid_client` error code per {{RFC6749}} or {{RFC7523}}.  A supplied invalid DPoP proof uses the `invalid_dpop_proof` error code; a nonce challenge uses the `use_dpop_nonce` error code, per {{RFC9449, Sections 5 and 8}}.  A missing required grant proof is an input-validation failure and uses the grant-type-specific error above.  Actor-authorization denials use the `actor_unauthorized` error code, an extension error permitted by {{Section 2.2.2 of RFC8693}}.

The following error codes apply to both AS and TTS endpoints:

| Error | Condition |
|-------|-----------|
| `invalid_request` | Malformed request, missing required request parameter, or an actor addition that would exceed the depth limit; on a Token Exchange request, also input-validation failures |
| `invalid_grant` | On a JWT bearer grant request: input-validation failures, including an invalid assertion, untrusted issuer, or failed grant binding |
| `invalid_scope` | No effective scope remains for reasons other than categorical actor denial |
| `actor_unauthorized` | Actor policy prohibits the request, rejects the actor type, or cannot confirm the required delegation relationship |

Missing required claims include `act.sub`, `act.iss`, the binding claim of a sender-constrained JWT assertion grant, and the top-level `iss` claim on a delegated Transaction Token.  TTS failures to preserve the subject or to trust inbound actor identifiers use the `invalid_request` error code.

The `error_description` parameter SHOULD be included and SHOULD describe which aspect of actor-profile processing failed, to the extent permitted by the server's security and privacy policy.

For a token-endpoint `actor_unauthorized` response, the client SHOULD check `entity_profiles_supported.actor` and any `error_description`.  A different actor credential or grant might resolve the failure; local-policy prohibitions might have no remediation.  After an RS returns this error, the client SHOULD obtain a token through a different actor credential or grant path.

The following is an example of an `actor_unauthorized` error response:

~~~json
{
  "error": "actor_unauthorized",
  "error_description": "Actor type not accepted for this scope"
}
~~~

# Metadata and Discovery {#metadata-and-discovery}

Authorization servers and resource servers advertise support for this profile using the following parameters:

| Parameter | Metadata | Capability |
|-----------|----------|------------|
| `authorization_grant_profiles_supported` | Authorization server | JWT authorization-grant profiles |
| `actor_profile_token_exchange` | Authorization server | Token Exchange input and output types |
| `entity_profiles_supported.actor` | Authorization server | Accepted actor entity types |
| `actor_profile_required` | Protected resource | Resource requirements for delegated requests |

{{I-D.ietf-oauth-identity-assertion-authz-grant}} defines `authorization_grant_profiles_supported`, and {{I-D.mora-oauth-entity-profiles}} defines `entity_profiles_supported.actor`.  This document defines the remaining two parameters in [Authorization Server Metadata](#authorization-server-metadata) and [Protected Resource Metadata](#protected-resource-metadata).

These signals do not guarantee acceptance of a particular request or every combination of advertised capabilities.  Additional constraints are communicated through deployment documentation or agreements.

## Authorization Server Metadata {#authorization-server-metadata}

This profile uses the following parameters in the AS metadata document ({{RFC8414}}):

`authorization_grant_profiles_supported`:
: OPTIONAL.  A JSON array defined by {{I-D.ietf-oauth-identity-assertion-authz-grant}}.  An AS that processes JWT authorization grants carrying actor-profile claims SHOULD include `urn:ietf:params:oauth:grant-profile:actor-profile`.  Including this value advertises the processing rules for all JWT authorization-grant paths defined in this document.  An AS advertising this value MUST also list `urn:ietf:params:oauth:grant-type:jwt-bearer` in `grant_types_supported`.

`actor_profile_token_exchange`:
: OPTIONAL.  A JSON object advertising coarse Token Exchange capabilities for requests in which actor-profile processing can apply.  When this parameter is absent, the AS makes no claim about Token Exchange support under this profile.  This document defines the following members:

  *  `subject_token_types_supported`: OPTIONAL.  A JSON array of token-type URI strings indicating the `subject_token_type` values the AS accepts for Token Exchange requests in which actor-profile processing can apply.  Values defined by this document are:

     -  `urn:ietf:params:oauth:token-type:jwt`: JWT assertion grants ([JWT Assertion Grants](#jwt-assertion-grants))
     -  `urn:ietf:params:oauth:token-type:access_token`: JWT access tokens ([JWT Access Tokens](#jwt-access-tokens))
     -  `urn:ietf:params:oauth:token-type:id_token`: OpenID Connect ID Tokens ([OpenID Connect ID Token](#id-tokens))
     -  `urn:ietf:params:oauth:token-type:refresh_token`: refresh tokens ([Refresh Token](#refresh-tokens))
     -  `urn:ietf:params:oauth:token-type:txn_token`: Transaction Tokens ([Transaction Tokens](#transaction-tokens))

  *  `actor_token_types_supported`: OPTIONAL.  A JSON array of token-type URI strings indicating the `actor_token_type` values the AS accepts for Token Exchange requests in which actor-profile processing can apply.  Values defined by this document are:

     -  `urn:ietf:params:oauth:token-type:jwt`: JWT client assertions ([JWT Client Assertion](#jwt-client-assertion-as-actor-token)) and workload identity credentials ([Workload Credential Processing](#workload-identity-as-actor-token)), which [Token Exchange Processing](#token-exchange-processing) distinguishes
     -  `urn:ietf:params:oauth:token-type:access_token`: JWT access tokens ([JWT Access Token as actor_token](#jwt-access-token-as-actor-token))

  *  `requested_token_types_supported`: OPTIONAL.  A JSON array of token-type URI strings indicating the `requested_token_type` values the AS accepts for Token Exchange requests in which actor-profile processing can apply.  Values defined by this document are:

     -  `urn:ietf:params:oauth:token-type:access_token`: JWT access tokens ([JWT Access Tokens](#jwt-access-tokens))
     -  `urn:ietf:params:oauth:token-type:jwt`: JWT assertion grants ([JWT Assertion Grants](#jwt-assertion-grants))
     -  `urn:ietf:params:oauth:token-type:id-jag`: ID-JAGs ({{I-D.ietf-oauth-identity-assertion-authz-grant}}), a JWT assertion grant profile ([JWT Assertion Grant Output](#jwt-assertion-grant-issuance))
     -  `urn:ietf:params:oauth:token-type:txn_token`: Transaction Tokens ([Transaction Tokens](#transaction-tokens))

Advertising a token type does not guarantee support for every input/output combination, resource, scope, binding mechanism, or JWT variant.

The following is an example of an AS metadata fragment:

~~~json
{
  "issuer": "https://as.enterprise.example",
  "token_endpoint": "https://as.enterprise.example/token",
  "grant_types_supported": [
    "urn:ietf:params:oauth:grant-type:token-exchange",
    "urn:ietf:params:oauth:grant-type:jwt-bearer"
  ],
  "dpop_signing_alg_values_supported": ["ES256", "RS256"],
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:actor-profile"
  ],
  "actor_profile_token_exchange": {
    "subject_token_types_supported": [
      "urn:ietf:params:oauth:token-type:id_token",
      "urn:ietf:params:oauth:token-type:jwt",
      "urn:ietf:params:oauth:token-type:access_token",
      "urn:ietf:params:oauth:token-type:txn_token"
    ],
    "actor_token_types_supported": [
      "urn:ietf:params:oauth:token-type:jwt",
      "urn:ietf:params:oauth:token-type:access_token"
    ],
    "requested_token_types_supported": [
      "urn:ietf:params:oauth:token-type:access_token",
      "urn:ietf:params:oauth:token-type:jwt",
      "urn:ietf:params:oauth:token-type:txn_token"
    ]
  },
  "entity_profiles_supported": {
    "client": ["service", "ai_agent"],
    "subject": ["user", "service", "ai_agent"],
    "actor":   ["user", "service", "ai_agent"]
  }
}
~~~

## Protected Resource Metadata {#protected-resource-metadata}

This document defines the following parameter for use in Protected Resource Metadata ({{RFC9728}}):

`actor_profile_required`:
: OPTIONAL.  A boolean indicating that delegated access to the resource requires actor information conforming to this profile.  When the value is `false` or the parameter is absent, the metadata makes no such claim.  Non-delegated requests need not carry the `act` claim.

  Clients SHOULD treat `true` as requiring a conforming token or an explicitly documented introspection path that provides equivalent claims for opaque tokens.

  An AS that knows, by configuration or from this metadata, that `actor_profile_required` is `true` for the resource MUST reject a request that would produce a nonconforming delegated token for the resource unless a supported introspection path provides equivalent actor information.  An RS enforcing this policy MUST reject a delegated request for which neither form of actor information is available.

  The parameter applies to the resource as a whole.  An RS with path-specific requirements MUST enforce them at the request layer.  It MAY advertise `true` as a conservative resource-wide signal; clients and deployment documentation SHOULD account for path-specific enforcement that metadata cannot fully express.

Clients discover the actor entity profile values that an authorization server for the resource accepts by consulting the `entity_profiles_supported.actor` array in the AS metadata of one of the authorization servers listed in the resource's `authorization_servers` array ({{RFC9728}}).  When `authorization_servers` lists multiple entries, the client SHOULD select the AS that issued or will issue the token being presented.

The following is an example of a Protected Resource Metadata fragment:

~~~json
{
  "resource": "https://api.travel-provider.example",
  "authorization_servers": [
    "https://as.travel-provider.example"
  ],
  "actor_profile_required": true
}
~~~

## Transaction Token Capability Signaling {#transaction-token-capability-signaling}

An AS or TTS advertises Transaction Token support for Token Exchange under this profile through `actor_profile_token_exchange.requested_token_types_supported`.  When an AS or TTS that can issue Transaction Tokens as delegated Token Exchange outputs under this profile publishes `actor_profile_token_exchange`, it MUST list `urn:ietf:params:oauth:token-type:txn_token` in `actor_profile_token_exchange.requested_token_types_supported`.  This document does not define a separate Transaction Token discovery parameter.

## Capability Signaling Usage

Clients consult the resource's Protected Resource Metadata ({{RFC9728}}) and the associated AS metadata ({{RFC8414}}) for the parameters listed in [Metadata and Discovery](#metadata-and-discovery).  When this profile is combined with Identity Chaining ({{I-D.ietf-oauth-identity-chaining}}), clients SHOULD additionally consult `identity_chaining_requested_token_types_supported`; the two parameter sets are independent.

The metadata defined in this document does not advertise authorization-code actor-selection mechanisms or per-scope or per-path actor type restrictions.  Deployments that need either capability rely on deployment documentation, bilateral agreement, or a companion profile.  When a delegated request carries `act.sub_profile`, its value SHOULD be drawn from `entity_profiles_supported.actor` when that metadata is available.

As an example of a client preflight failure, if the RS metadata advertises `"actor_profile_required": true` but the target AS metadata advertises `"entity_profiles_supported": { "actor": ["service"] }` and the client's acting entity profile is `ai_agent`, the client ordinarily stops before making the token request because the AS does not advertise support for the actor type that the client needs to represent.

# Companion Profiles and Extension Points {#companion-profile-extensibility}

This document defines delegated identity as represented in the current token; companion profiles can define supplementary behavior such as provenance, transparency, or deployment-specific audit material.

A companion profile that builds on this profile:

*  MAY define additional top-level JWT claims, OAuth metadata parameters, or introspection response members that apply only to a token that conforms to this profile or to an introspection response for a delegated opaque access token under [Token Introspection](#token-introspection);
*  MUST preserve the meanings of the token's top-level `sub`, the outermost `act.sub`, the (`act.iss`, `act.sub`) actor identifier pair, the nested `act` chain ordering, and the top-level `cnf` claim for the current presenter;
*  MUST NOT reinterpret `act.iss`, nested `act` objects, or the top-level `cnf` claim as independently trusted prior-hop provenance artifacts;
*  SHOULD define any supplementary provenance, receipt, or chain-wide state in separate top-level claims or equivalent companion mechanisms rather than by overloading members inside inherited `act` objects;
*  if it defines data that aligns to the visible `act` chain, MUST specify the alignment rules, the behavior when coverage is partial, and the behavior when introspection or privacy filtering suppresses part of the visible chain.

A companion profile can define authorization based on independently verifiable per-hop evidence that it carries in its own top-level claims.  Nested `act` objects remain informational for access control ({{Section 4.1 of RFC8693}}).

An implementation that conforms only to this core profile MUST ignore unrecognized companion-profile claims, metadata parameters, and introspection response members unless another specification or local policy defines their meaning.  A deployment that requires support for a companion profile expresses that requirement through the companion profile's own metadata, through out-of-band agreement, or through another explicit local-policy mechanism.

# Deployment Considerations

## Migration and Adoption {#migration-and-adoption}

### RFC 8693 Backwards Compatibility

An {{RFC8693}} actor object without the `iss` claim does not conform to this profile.  Implementations MUST treat it as nonconforming and MUST NOT infer semantics for absent claims.  When local policy or advertised metadata requires profile conformance for a token or assertion, recipients MUST reject such actor objects.

When an AS receives such an actor object:

*  If profile conformance is required by policy or metadata, the AS MUST reject the input under [Error Responses](#actor-profile-error-responses).
*  Otherwise, the AS MAY apply local rules for non-profile processing.  It MUST NOT add `iss` to an inherited actor, silently drop the inbound `act`, or carry the nonconforming chain into a profile-conforming output.  A request requiring that output MUST be rejected.

A deployment can migrate in three stages:

1.  Issuers emit `act.iss` for every actor in newly issued profile tokens, as [Actor Object Structure](#actor-object-structure) requires.  Existing consumers can ignore the additional claim.
2.  Consumers SHOULD begin validating the actor identifier context once issuers support it.
3.  Once all token issuers and consumers on a path have been updated, resources SHOULD enforce conformance through local policy and `actor_profile_required: true`.  ASes can also require conformance on updated inbound paths.

### Migrating from Implicit to Explicit Delegation {#migration-implicit-explicit}

Deployments that infer actors from `client_id`, `azp`, or request context can migrate incrementally:

*  Clients SHOULD prefer tokens with explicit actor claims when available.
*  Issuers SHOULD emit both legacy client identifiers and actor claims during the transition when feasible.
*  Without `act`, deployments MAY retain legacy client-based policy.
*  When both forms are present, deployments apply [Client Identity and Delegation](#client-identity-delegation) and record mismatches.

The legacy form carries only the `client_id` claim (and optionally the `azp` claim) to identify the acting party.  The following example shows the explicit form, which adds an `act` claim:

~~~json
{
  "iss": "https://as.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "client_id": "travel-assistant-client-id",
  "azp": "travel-assistant-client-id",
  "act": {
    "sub": "https://agents.enterprise.example/travel-assistant",
    "iss": "https://as.enterprise.example",
    "sub_profile": "ai_agent"
  },
  "scope": "booking:create"
}
~~~

In this example, `client_id` and `azp` remain auxiliary client-identity inputs, while `act.sub` carries the explicit actor identity.

The following example shows a mismatch: the OAuth client identifier and the explicit actor identifier name different parties unless trusted local mapping rules bind them.

~~~json
{
  "iss": "https://as.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "client_id": "travel-assistant-client-id",
  "act": {
    "sub": "https://agents.other-provider.example/concierge-bot",
    "iss": "https://as.other-provider.example",
    "sub_profile": "ai_agent"
  },
  "scope": "booking:create"
}
~~~

## Trusting Actor Identifier Pairs {#act-iss-authority-guidance}

[Validate Outermost Actor](#validate-outermost-actor) requires trust in the token issuer's authority to assert the (`act.iss`, `act.sub`) pair.  Deployment-specific mechanisms for establishing that trust include:

*  federation metadata or trust-framework configuration that authorizes the token issuer to assert actor identifiers in the `act.iss` context (for example, {{OpenID.Federation}});
*  pre-registration entries that explicitly authorize the token issuer to assert a specific (`act.iss`, `act.sub`) pair or identifiers of that form; and
*  bilateral or deployment-local policy rules that authorize the token issuer to carry the specific class of actor identifier used in `act.sub`.

For HTTPS identifiers, one possible local rule is URL namespace containment: an explicitly configured rule that compares scheme, host, port, and path boundaries.  Scheme and host comparisons follow {{Section 3.1 of RFC3986}} and {{Section 3.2.2 of RFC3986}}; paths are generally case-sensitive.  Subdomain relationships alone are often insufficient to establish trust without explicit configuration.

For example:

*  A token issued by `https://as.enterprise.example` with `act.iss = https://as.enterprise.example` and `act.sub = https://as.enterprise.example/agents/travel-assistant` would commonly satisfy a same-host local trust rule.
*  A token issued by `https://as.enterprise.example` with `act.iss = https://as.enterprise.example` and `act.sub = https://idp.enterprise.example/users/alice` would not ordinarily satisfy URL containment alone, because the host differs.

## Token Lifetime for Delegation Chains {#delegation-chain-token-lifetime}

Deployments SHOULD use shorter lifetimes for delegated tokens than for non-delegated tokens of equivalent scope.  JWT access tokens and assertion grants in multi-hop chains SHOULD last no longer than needed for the authorized task.  The lifetime of an upstream artifact also limits how long a downstream actor can request tokens without renewing that artifact.

Deployments with three or more actors SHOULD account for delayed revocation across hops.  Where revocation risk is significant, for example where a user can withdraw consent at any time, deployments SHOULD combine short lifetimes with introspection at sensitive resources rather than relying solely on the `exp` claim.  {{RFC9700}} provides general guidance.

# Security Considerations

This document defines no trust framework for actor identifier contexts, delegation approval, or subject-identifier translation; the security of those decisions depends on deployment policy and agreements ([Representation and Policy](#representation-and-policy)).

## Delegation Chain Integrity and Trust {#delegation-chain-integrity}

An attacker who can inject or forge `act` claims can impersonate an arbitrary actor and exercise a subject's permissions without authorization.  RS implementations validate the token signature before extracting actor claims, as the applicable token specification and [Resource Server Processing](#resource-server-processing) require, and MUST verify that the token issuer is trusted to convey the claims it carries.

Because inner `act` objects are set by upstream ASes and not re-signed at each hop, the integrity of the entire delegation chain depends on the signature of the token that carries it.  Implementations SHOULD use short token lifetimes, and an expired token is rejected per {{Section 4.1.4 of RFC7519}} regardless of chain depth.

Inner actor identities are not inputs to access-control decisions ({{Section 4.1 of RFC8693}}): they are endorsed only by the outer token issuer's signature ([Carry Prior-Actor Context](#carry-prior-actor-context)).  [Companion Profiles and Extension Points](#companion-profile-extensibility) describes how independently verifiable per-hop evidence can support authorization.

ASes performing Token Exchange MUST evaluate cross-domain delegation grants explicitly and SHOULD NOT grant cross-domain actors the same rights as same-domain actors absent an explicit trust decision that makes them equivalent.

## Self-Issued Authorization Grants {#security-self-issued-grants}

In a self-issued assertion grant, the acting entity is itself the grant's `iss` and directly asserts delegation to itself without any upstream AS having authenticated the actor or pre-validated the delegation relationship.  [JWT Assertion Grant Structure](#jwt-assertion-grants-structure) requires rejecting such grants by default; this section gives the controls a deployment needs when another specification or local policy enables them.

This section does not apply to client assertions used as `actor_token` ([JWT Client Assertion](#jwt-client-assertion-as-actor-token)), for which `iss = sub = client_id` is the conformant pattern.

Because no upstream AS vouches for the actor's identity or the delegation relationship, the receiving AS MUST NOT treat the self-asserted delegation claim alone as a sufficient authorization basis.  When a deployment enables self-issued authorization grants, the receiving AS MUST at minimum:

*  Validate the JWT signature using the key identified in the JWT header, obtained from a pre-registered or otherwise independently trusted source for the self-issuing party.
*  Verify the `exp`, `iat`, and `nbf` claims per {{RFC7519}}.
*  To prevent replay, reject an assertion whose validated (`iss`, `jti`) pair has already been accepted, for as long as the assertion remains acceptable, including any allowed clock skew.
*  Verify proof of possession per the token-endpoint mechanism in use (DPoP per {{RFC9449}} or mTLS per {{RFC8705}}), and against the assertion's top-level `cnf` when present and not superseded by presenter rebind, as in step 6 of [Authorization Grant Processing](#jwt-assertion-grants-processing) and [JWT Assertion Grant as subject_token](#jwt-assertion-grant-as-subject-token).
*  Apply the actor-profile validation and proof-of-possession requirements in [Authorization Grant Processing](#jwt-assertion-grants-processing).
*  Establish the delegation relationship from an independent authorization basis such as a pre-registered grant, explicit consent record, or equivalent deployment-specific artifact.

In the absence of these controls, an attacker can self-assert an arbitrary (`sub`, `act.sub`) pair and bypass actor-profile authorization enforcement.  Deployments MUST confine self-issued authorization grants to a single trust domain and MUST NOT propagate them across organizational boundaries.  Discovery of their acceptance is left to the enabling specification or local policy.

## Assertion Replay Prevention {#security-assertion-replay}

An attacker who replays a delegated assertion can obtain tokens that exercise the subject's authorization and can establish an unauthorized delegation chain.  Step 1 of [Authorization Grant Processing](#jwt-assertion-grants-processing) defines replay prevention; short assertion lifetimes bound how long replay records are retained.  Presenter rebind supersedes a sender-constrained grant's binding ([JWT Assertion Grant as subject_token](#jwt-assertion-grant-as-subject-token)), so a party that obtains such a grant but not its key can redeem it once, as a Token Exchange `subject_token`, with any actor credential that local policy authorizes to act for `sub`.  Actor authorization policy, mandatory single use of the grant, and short grant lifetimes remain the mitigations for this case.

## Token Substitution

An attacker who can present a token with a crafted `sub_profile` or delegation chain can attempt to escalate privileges.  ASes MUST validate inbound `sub_profile` values against the syntax requirements of this document, the applicable registry or deployment-specific allowed set where such checks are part of local policy, and the local policy applicable to the token they are issuing.  They MUST preserve unrecognized but syntactically valid values, as required by [Preserve Inbound Chain](#preserve-inbound-chain) for inherited actor objects and by step 5 of [JWT Access Token Output](#jwt-access-token-propagation) for a carried-forward top-level `sub_profile`, and they MUST reject values that are malformed or disallowed by local policy.

## Confused Deputy

A resource server that evaluates only `sub` when `act` is present is susceptible to a confused deputy attack: a malicious actor exploits a subject's pre-existing permissions without the subject's ongoing consent.  The mitigation is to authorize the (`sub`, outermost `act.sub`) pair, as [Actor Authorization](#actor-authorization) describes.

## Actor-Authorization Bypass

A resource server that requires actor authorization but does not enforce it on every request path where it accepts delegated access, including introspection-based paths ([Token Introspection](#token-introspection)), lets an attacker bypass that policy.  Deployments that signal delegated-token requirements with `actor_profile_required: true` SHOULD ensure that the documented request paths requiring delegated access align with their actual enforcement behavior so that clients do not overinterpret the signal.

## Client Identity and Delegation {#client-identity-delegation}

Client identity (`client_id`, `azp`, or authenticated client context) remains an auxiliary signal under this profile, and the outermost `act.sub`, when present, is the explicit delegated-actor signal.  The following rules apply:

*  When `act` is present, implementations MUST NOT substitute `client_id`, `azp`, or other client-identity signals for it as the delegated-actor signal.  A trusted local mapping can establish that a client identifier and `act.sub` identify the same entity without changing the meaning of either claim.
*  When a single `client_id` registration serves multiple distinct acting entities (for example, an agent orchestration platform executing requests on behalf of different agent instances), `client_id` alone does not identify the runtime actor.  Each such request SHOULD carry `act.sub` identifying the specific acting principal.
*  During token issuance, `client_id` and `azp` MUST NOT be rewritten to represent delegation state that belongs in `act`; see [JWT Access Token Output](#jwt-access-token-propagation) for propagation rules.
*  When both explicit (`act.sub`) and implicit (`client_id`, `azp`) signals are present and local policy expects them to identify the same party, implementations SHOULD perform identifier reconciliation; if it fails, the identifiers are treated as distinct, and an operation that requires them to identify the same party is rejected, as defined for Identifier Reconciliation in [Conventions and Definitions](#conventions).
*  When a protected resource or authorization path enforces explicit delegation under this profile, implementations MUST NOT downgrade to non-`act` processing solely because another token-acquisition path or legacy policy input remains available.

## `sub_profile` Trust

The token issuer asserts the `sub_profile` claim, so the claim is only as trustworthy as that issuer.  Resource servers MUST NOT trust `sub_profile` values in tokens issued by untrusted parties.  Resource server operators SHOULD configure a list of accepted entity-type profiles per trust domain.

## Subject Namespace Translation {#subject-namespace-translation}

An AS or TTS can translate `sub` when crossing identifier namespaces, but MUST NOT do so unless local policy establishes that both identifiers refer to the same subject.  Trust to perform that mapping is separate from trust to sign tokens and SHOULD be established explicitly.

A translated `sub` is authoritative only within the trust context of the issuer that performed the translation.  A recipient that relies only on the issued token MAY evaluate the translated `sub` as the subject in that issuer's namespace.  A recipient needing proof of equivalence to an upstream subject MUST obtain additional evidence, such as another specification, an identity-chain mechanism, or an explicit trust agreement.  If required evidence is unavailable, it SHOULD reject the token.

This profile provides neither portable subject-equivalence proofs nor a general mechanism for correlating subjects across domains.

## Presenter Binding

Without top-level presenter proof of possession, any party can replay a leaked token.  A sender-constrained delegated token binds the current presenter (the outermost actor), and the RS validates the presenter proof against the top-level `cnf` claim ([Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation)).  Clients should also use the `resource` parameter ({{RFC8707}}) when requesting delegated tokens, because audience restriction limits where a leaked token can be used.

This document does not define per-hop actor-key provenance within the delegation chain.  Deployments that need stronger assurance for prior-hop provenance MUST use an additional mechanism outside the scope of this document, such as signed hop receipts, transparency-log-based recording, or another future extension; they MUST NOT overload `act.iss` or redefine nested `act` semantics to carry that provenance.

## Delegation Depth Limits

Unbounded delegation chains increase the attack surface, complicate policy evaluation, and enable denial of service through chain parsing; the local maximum depth that [Delegation Chains](#delegation-chains) requires bounds them.

Extensions that attach per-hop signed material grow token size in proportion to chain depth.  Deployments that combine the reference chain depth with one or more such per-hop mechanisms SHOULD measure realistic token sizes against their transport limits (HTTP header limits are often 8 KB) and consider using token introspection ({{RFC7662}}) where inline carriage exceeds those limits.

## Actor Identity Rotation {#actor-identity-rotation}

Deployments SHOULD choose `act.sub` to be a durable, stable identifier independent of ephemeral key material.  In particular, deployments SHOULD NOT use a JWK thumbprint or other key-derived value as `act.sub`; doing so silently changes the actor identity on every key rotation, which can break delegation grants and policy bindings that reference the prior identifier.

When `act.sub` itself must change (for example, because an agent instance is replaced, a workload is renamed, or an actor identifier namespace migrates), existing delegation grants and issued tokens continue to reference the old identifier.  Deployments MUST explicitly re-establish delegation grants for the new identity; the old grants do not automatically transfer.  Tokens issued under the old identity remain valid until they expire.

## Delegation Revocation {#delegation-revocation}

Token revocation ({{RFC7009}}) does not by itself revoke a delegation relationship or propagate revocation to downstream tokens.  Without an additional revocation mechanism, those tokens can remain usable until expiration; see [Token Lifetime for Delegation Chains](#delegation-chain-token-lifetime).

An AS with authoritative knowledge that a delegation has been revoked SHOULD refuse new tokens for that (subject, actor) pair.  Revocation of a prior hop is enforced through the AS's own issuance records or independently verifiable evidence, not through nested `act` objects.  Implementations MUST NOT skip revocation checks because of chain depth.

Long-lived refresh behavior can delay revalidation of upstream delegation; refresh-token policy, delegation-state storage, and cross-hop revocation remain deployment-specific.

# Privacy Considerations {#privacy}

Delegation chains can reveal sensitive information about user behavior, enterprise topology, software suppliers, and internal tool composition.  Issuers therefore SHOULD disclose only the actor information needed by the relying party for authorization, audit, or policy enforcement.

Cross-domain deployments SHOULD prefer stable but non-reassigned identifiers and SHOULD consider pairwise identifiers for human subjects when a globally correlatable identifier is not required by the use case.

When the same logical entity can appear in different identifier namespaces, such as `azp`, `req_wl`, and `act.sub`, issuers and relying parties SHOULD use explicit issuer scoping and locally trusted mapping rules rather than string equality alone to determine whether those identifiers refer to the same entity.

Issuers SHOULD minimize disclosure of prior actors through audience and token-design decisions made before issuance.  Once an issuer preserves a delegation chain, [Preserve Inbound Chain](#preserve-inbound-chain) requires copying it intact.  If local privacy requirements would require omitting a chain element, the issuer rejects the request rather than truncating the chain.

A Transaction Token's `txn` value links service calls in the same transaction and can enable correlation across organizations.  Deployments SHOULD follow the privacy guidance in {{I-D.ietf-oauth-transaction-tokens}} when propagating it across trust domains.

The `act.sub_profile` claim reveals the actor's entity type, including whether it is an AI agent.  In some jurisdictions or deployment contexts, this disclosure can be legally significant or can reveal sensitive information about user behavior and tool composition.  Issuers SHOULD consider audience-specific disclosure constraints and SHOULD omit unnecessary entity classifications when constructing new actor objects.  Inherited actors remain subject to the preservation rules in [Preserve Inbound Chain](#preserve-inbound-chain).

The `req_wl` claim can reveal internal workload topology.  A TTS SHOULD disclose it only where needed for authorization, audit, or policy enforcement, and SHOULD avoid exposing internal workload identifiers across domains unless the deployment requires it.


# IANA Considerations

## OAuth URI Registration {#iana-oauth-uri}

This document requests IANA to register the following value in the "OAuth URI" registry:

*  URN: `urn:ietf:params:oauth:grant-profile:actor-profile`
*  Common Name: OAuth Actor Profile for Delegation
*  Change Controller: IETF
*  Reference: [Authorization Server Metadata](#authorization-server-metadata) of this document


## OAuth Authorization Server Metadata Registry

This document requests IANA to register the following value in the "OAuth Authorization Server Metadata" registry ({{RFC8414}}):

*  Metadata Name: `actor_profile_token_exchange`
*  Metadata Description: JSON object advertising coarse Token Exchange capabilities for requests in which actor-profile processing can apply
*  Change Controller: IETF
*  Reference: [Authorization Server Metadata](#authorization-server-metadata) of this document


## OAuth Protected Resource Metadata Registry

This document requests IANA to register the following value in the "OAuth Protected Resource Metadata" registry ({{RFC9728}}):

*  Metadata Name: `actor_profile_required`
*  Metadata Description: Boolean indicating whether the RS advertises that delegated requests for this resource are expected to provide actor-profile information conforming to this document's semantics
*  Change Controller: IETF
*  Reference: [Protected Resource Metadata](#protected-resource-metadata) of this document


## OAuth Error Registry {#iana-error-codes}

This document requests IANA to register the following value in the "OAuth Extensions Error Registry" ({{Section 11.4 of RFC6749}}):

*  Error Name: `actor_unauthorized`
*  Error Usage Location: token error response, resource access error response
*  Related Protocol Extension: OAuth Actor Profile for Delegation
*  Change Controller: IETF
*  Reference: [Error Responses](#actor-profile-error-responses) of this document


## OAuth Token Introspection Response Registry

This document requests IANA to register the following value in the "OAuth Token Introspection Response" registry ({{Section 3.1 of RFC7662}}):

*  Name: `chain_complete`
*  Description: Boolean indicating whether the `act` delegation chain in the introspection response is complete.  When `false`, one or more inner `act` chain entries have been omitted from the response for privacy reasons.  When absent, the chain SHOULD be treated as complete unless local policy or deployment context indicates otherwise.
*  Change Controller: IETF
*  Reference: [Token Introspection](#token-introspection) of this document


## JWT Claims Registry

This document does not request independent entries in the "JSON Web Token Claims" registry for the `act` object sub-claims (`iss`, `sub_profile`, and any extension claims) it defines or profiles.  These claims appear only within the JSON object value of the `act` claim, which {{RFC8693}} already registers in the "JSON Web Token Claims" registry.


## Transaction Token Type URI {#iana-token-types}

This document makes no independent requests to the "OAuth URI" registry for `urn:ietf:params:oauth:token-type:txn_token`.  That URI is defined and registered by {{I-D.ietf-oauth-transaction-tokens}}.  Its inclusion as a defined value for `actor_profile_token_exchange.requested_token_types_supported` in [Authorization Server Metadata](#authorization-server-metadata) is contingent on the progression of {{I-D.ietf-oauth-transaction-tokens}}.


## OAuth Entity Profiles Registry {#iana-entity-profiles}

This document makes no independent requests to the "OAuth Entity Profiles" registry.  It normatively depends on the "Actor Profile" usage location, the `actor` array in `entity_profiles_supported`, and the registration of `user`, `service`, and `ai_agent` with that usage location, all of which are defined and requested by {{I-D.mora-oauth-entity-profiles}}.  The IANA actions for those entries are contingent on the progression of {{I-D.mora-oauth-entity-profiles}}.


--- back

# Service-to-Service Delegation Example {#appendix-service-to-service}

This appendix gives an example of this profile in a same-domain service-to-service delegation flow that involves no AI agent.  A payroll batch processor acts on behalf of a human payroll administrator to call the Payroll API; the Payroll API then exchanges the inbound access token for an internal Transaction Token used to write an audit record.

## Scenario

| Party | Identifier |
|-------|------------|
| Payroll Administrator | `https://idp.example.com/users/pat` |
| Enterprise AS | `https://as.example.com` |
| Payroll Batch Processor | `https://services.example.com/payroll-batch` |
| Payroll API | `https://services.example.com/payroll-api` |
| Audit TTS | `https://tts.example.com` |
| Audit Service | `https://internal.example.com/audit` |

The batch processor is both the OAuth client and the acting service.  The `client_id` claim identifies the client registration, and `act.sub` explicitly identifies the acting service.

## Access Token

The Enterprise AS issues the following JWT access token for the Payroll API:

~~~json
{
  "iss": "https://as.example.com",
  "sub": "https://idp.example.com/users/pat",
  "sub_profile": "user",
  "client_id": "payroll-batch-client",
  "aud": "https://services.example.com/payroll-api",
  "scope": "payroll:run",
  "cnf": { "jkt": "SvcJKT-123" },
  "act": {
    "sub": "https://services.example.com/payroll-batch",
    "iss": "https://as.example.com",
    "sub_profile": "service"
  }
}
~~~

This token carries a single-hop actor object: the `act` claim is present but contains no nested `act` object.  The `sub` claim identifies the payroll administrator, while `act.sub` identifies the service exercising that administrator's authorization.

## Transaction Token

After processing the payroll request, the Payroll API exchanges the inbound access token at the Audit TTS to call the internal Audit Service, presenting its workload credential, issued by `https://as.example.com`, as `actor_token`.  The Payroll API is the requesting workload (`req_wl`).  In presenter-rebind mode, the TTS validates the inbound delegation chain and the workload credential, preserves the chain as an inner `act` object, and adds a new outermost actor for the Payroll API:

~~~json
{
  "iss": "https://tts.example.com",
  "sub": "https://idp.example.com/users/pat",
  "sub_profile": "user",
  "req_wl": "https://services.example.com/payroll-api",
  "aud": "https://internal.example.com/audit",
  "scope": "audit:create",
  "txn": "550e8400-e29b-41d4-a716-446655440099",
  "cnf": { "jkt": "ApiJKT-456" },
  "act": {
    "sub": "https://services.example.com/payroll-api",
    "iss": "https://as.example.com",
    "sub_profile": "service",
    "act": {
      "sub": "https://services.example.com/payroll-batch",
      "iss": "https://as.example.com",
      "sub_profile": "service"
    }
  }
}
~~~

The administrator remains the subject.  The Payroll API becomes the outermost actor, and the batch processor remains the inner actor.

# Cross-Domain AI Agent Flow: ID Token to Transaction Token {#appendix-cross-domain}

This appendix follows a delegated request across two trust domains.  Token validation and presenter proofs follow the underlying token specifications and the deployment profile.

All claim values, JWK thumbprints, and domain names are synthetic.  Long HTTP example lines use the backslash convention defined in {{RFC8792}}; other line breaks in form bodies are for readability only.

## Scenario and Parties

Alice authenticates at the Enterprise IdP AS.  Her Travel Assistant exchanges the ID Token for an ID-JAG, then presents that grant to the Travel Provider AS for an access token.  The agent calls the Booking Tool, which exchanges the access token for a Transaction Token to call the Inventory Service.

~~~
Enterprise domain                 Travel Provider domain
Alice
  | (1) authenticates
  v
Enterprise IdP AS -> ID Token
  | (2) Token Exchange
  v
Enterprise IdP AS -> ID-JAG
                       | (3) JWT bearer grant
                       +--------> Travel Provider AS -> Access Token
                                               |
                  Travel Assistant <-----------+
                       | (4) Access Token + DPoP
                       v
                  Booking Tool (RS) --(5) Token Exchange--> TTS
                       ^                                     |
                       +-------- Transaction Token ----------+
                       | (6) Transaction Token + WIMSE proof
                       v
                  Inventory Service (RS)
~~~

| Party | Identifier | Trust Domain |
|-------|------------|--------------|
| Alice | `https://idp.enterprise.example/users/alice` | Enterprise |
| Enterprise IdP AS | `https://as.enterprise.example` | Enterprise |
| Travel Assistant | `https://agents.enterprise.example/travel-assistant` | Enterprise |
| Travel Provider AS | `https://as.travel-provider.example` | Travel Provider |
| Travel Provider TTS | `https://tts.travel-provider.example` | Travel Provider |
| Booking Tool | `https://tools.travel-provider.example/booking-tool` | Travel Provider |
| Inventory Service | `https://internal.travel-provider.example/inventory` | Travel Provider |

The following table lists the presenter key bindings:

| Principal | JWK Thumbprint (`jkt`) |
|-----------|------------------------|
| Travel Assistant | `AgentJKT-NzbLsXh8uDCcd7MN` |
| Booking Tool | `ToolJKT-0ZcOCORZNYy9ZhHi` |


## Capability Discovery (Preflight)

Before initiating the flow, the agent consults the following Travel Provider AS metadata ([Metadata and Discovery](#metadata-and-discovery)) as an advisory compatibility check:

~~~json
{
  "issuer": "https://as.travel-provider.example",
  "grant_types_supported": [
    "urn:ietf:params:oauth:grant-type:jwt-bearer",
    "urn:ietf:params:oauth:grant-type:token-exchange"
  ],
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag",
    "urn:ietf:params:oauth:grant-profile:actor-profile"
  ],
  "actor_profile_token_exchange": {
    "subject_token_types_supported": [
      "urn:ietf:params:oauth:token-type:jwt"
    ],
    "actor_token_types_supported": [
      "urn:ietf:params:oauth:token-type:jwt"
    ],
    "requested_token_types_supported": [
      "urn:ietf:params:oauth:token-type:access_token",
      "urn:ietf:params:oauth:token-type:txn_token"
    ]
  },
  "entity_profiles_supported": {
    "subject": ["user", "ai_agent"],
    "actor":   ["user", "ai_agent", "service"]
  }
}
~~~

The agent confirms that its `sub_profile` (`ai_agent`) is in `entity_profiles_supported.actor`, that ID-JAG and actor-profile grants are advertised in `authorization_grant_profiles_supported`, and that the planned input/output token types appear in `actor_profile_token_exchange`.  These signals are only coarse compatibility indicators; the agent proceeds because the advertised capabilities cover its planned path.


## Step 1: User Authentication (ID Token)

Alice authenticates to the Enterprise IdP AS, which issues the following ID Token.  An ID Token implicitly identifies a user; it does not carry a `sub_profile` claim, and the AS establishes the entity type in subsequently issued tokens (Step 2):

~~~json
{
  "iss": "https://as.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "aud": "https://agents.enterprise.example/travel-assistant",
  "exp": 1743379200,
  "iat": 1743375600
}
~~~

## Step 2: Enterprise Token Exchange (ID Token to ID-JAG)

The agent presents Alice's ID Token as the `subject_token` in a Token Exchange request to the Enterprise IdP AS, requesting an ID-JAG ({{I-D.ietf-oauth-identity-assertion-authz-grant}}).  The agent's client assertion ({{RFC7523}}) serves as both the `client_assertion` (for client authentication) and the `actor_token` (for actor identity), as described in [JWT Client Assertion](#jwt-client-assertion-as-actor-token).  The Enterprise IdP AS authenticates the client, verifies that the ID Token audience matches that client, and uses local delegation policy to construct the issued ID-JAG.  The following is an example of the Token Exchange request:

~~~
NOTE: '\' line wrapping per RFC 8792

POST /token HTTP/1.1
Host: as.enterprise.example
Content-Type: application/x-www-form-urlencoded
DPoP: <AgentJKT-proof>

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token=<alice-id-token>
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3A\
  token-type%3Aid_token
&requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3A\
  token-type%3Aid-jag
&audience=https%3A%2F%2Fas.travel-provider.example
&resource=https%3A%2F%2Fapi.travel-provider.example
&scope=booking%3Acreate
&client_id=https%3A%2F%2Fagents.enterprise.example%2Ftravel-assistant
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A\
  client-assertion-type%3Ajwt-bearer
&client_assertion=<travel-assistant-client-assertion>
&actor_token=<travel-assistant-client-assertion>
&actor_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Ajwt
~~~

The Enterprise IdP AS validates the shared client assertion for both client authentication and actor identity, verifies the DPoP proof, and confirms Alice's delegation under local policy.  It applies scope reduction and binds the following issued ID-JAG to the demonstrated DPoP key:

~~~json
{
  "iss": "https://as.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
  "client_id": "https://agents.enterprise.example/travel-assistant",
  "azp": "https://agents.enterprise.example/travel-assistant",
  "aud": "https://as.travel-provider.example",
  "resource": "https://api.travel-provider.example",
  "jti": "ent-idj-20260401-001",
  "exp": 1743379200,
  "iat": 1743375600,
  "scope": "booking:create",
  "cnf": { "jkt": "AgentJKT-NzbLsXh8uDCcd7MN" },
  "act": {
    "sub": "https://agents.enterprise.example/travel-assistant",
    "iss": "https://as.enterprise.example",
    "sub_profile": "ai_agent"
  }
}
~~~

In this ID-JAG, the `client_id`, `azp`, and `act.sub` claims carry the same URI because the agent is both the client and the actor.  The top-level `cnf.jkt` member binds the grant to the agent's key.


## Step 3: Agent Exchanges ID-JAG for Access Token at Travel Provider AS

The agent presents the ID-JAG as a JWT bearer grant ({{RFC7523}}) to the Travel Provider AS, which processes it as an ID-JAG per {{I-D.ietf-oauth-identity-assertion-authz-grant}} with the actor-profile rules in [Authorization Grant Processing](#jwt-assertion-grants-processing):

~~~
POST /token HTTP/1.1
Host: as.travel-provider.example
Content-Type: application/x-www-form-urlencoded
DPoP: <AgentJKT-proof>

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
&assertion=<id-jag>
&scope=booking%3Acreate
~~~

The Travel Provider AS performs actor-profile processing per [Authorization Grant Processing](#jwt-assertion-grants-processing): it verifies the request's DPoP proof against the top-level `cnf.jkt` in the inbound ID-JAG and checks that `act.sub_profile` (`ai_agent`) is permitted as an actor for the requested scope under local policy.  It issues the following access token, which preserves the delegation chain:

~~~json
{
  "iss": "https://as.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
  "client_id": "https://agents.enterprise.example/travel-assistant",
  "azp": "https://agents.enterprise.example/travel-assistant",
  "aud": "https://api.travel-provider.example",
  "jti": "tp-at-20260401-001",
  "exp": 1743379200,
  "iat": 1743375600,
  "scope": "booking:create",
  "cnf": { "jkt": "AgentJKT-NzbLsXh8uDCcd7MN" },
  "act": {
    "sub": "https://agents.enterprise.example/travel-assistant",
    "iss": "https://as.enterprise.example",
    "sub_profile": "ai_agent"
  }
}
~~~

The Travel Provider AS preserves Alice's subject identifier and classification, the actor chain, and the agent's presenter binding.  Client identifiers remain separate from actor semantics.


## Step 4: Agent Calls Booking Tool API

The agent presents the access token with a DPoP proof:

~~~
POST /bookings HTTP/1.1
Host: api.travel-provider.example
Authorization: DPoP <tp-access-token>
DPoP: <AgentJKT-proof>
Content-Type: application/json

{"origin": "SFO", "destination": "NYC", "depart": "2026-04-15"}
~~~

The Booking Tool RS authorizes the (`sub`, outermost `act.sub`) pair ([Resource Server Processing](#resource-server-processing)): it evaluates Alice (`sub`, `sub_profile: user`) together with the Travel Assistant (`act.sub`, `sub_profile: ai_agent`) for the requested operation.  The `act.sub_profile` value is checked against `entity_profiles_supported.actor` per [Authorization Server Metadata](#authorization-server-metadata).


## Step 5: Booking Tool Exchanges Access Token for Transaction Token

The Booking Tool cannot reuse the received access token for internal calls because the token is sender-constrained to `AgentJKT`, a key that the Booking Tool does not possess.  Instead, it requests a Transaction Token from the TTS.  In this example, the TTS receives the Booking Tool's WIMSE Workload Identity Token (WIT), issued by the Travel Provider AS (`https://as.travel-provider.example`), as the Token Exchange `actor_token` and validates a Workload Proof Token (WPT).  The WIT identifies the Booking Tool and carries its confirmation key, while the WPT proves possession of that key and binds the request to the accompanying access token:

~~~
NOTE: '\' line wrapping per RFC 8792

POST /token HTTP/1.1
Host: tts.travel-provider.example
Content-Type: application/x-www-form-urlencoded
Workload-Proof-Token: <tool-wpt-with-wth-and-ath>

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token=<tp-access-token>
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3A\
  token-type%3Aaccess_token
&actor_token=<booking-tool-wit>
&actor_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Ajwt
&requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3A\
  token-type%3Atxn_token
&audience=https%3A%2F%2Ftravel-provider.example
&scope=inventory%3Acheck
&request_context=%7B%22req_ip%22%3A%22198.51.100.42%22%7D
~~~

The WIT is therefore the JWT `actor_token` defined by this profile, while the WPT provides the accompanying proof of possession required by the workload-credential profile.

The TTS applies actor-profile processing per [Transaction Token Output Rules](#transaction-token-output-rules): it preserves `sub` and `sub_profile` from the `subject_token`, sets `req_wl` to the authenticated Booking Tool, and creates a new outermost `act` object for the Booking Tool while nesting the `subject_token`'s existing `act` claim beneath it.  In this WIMSE-based deployment, the deployment profile defines the `cnf.jkt` binding, proven by the WPT, and the TTS binds the issued token to the Booking Tool's presenter key (`ToolJKT`), which the WIT's `cnf.jwk` carries:

~~~json
{
  "iss": "https://tts.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
  "scope": "inventory:check",
  "req_wl": "https://tools.travel-provider.example/booking-tool",
  "aud": "https://travel-provider.example",
  "jti": "txn-tok-20260401-001",
  "txn": "550e8400-e29b-41d4-a716-446655440001",
  "exp": 1743375750,
  "iat": 1743375650,
  "tctx": {
    "action": "check-availability",
    "origin": "SFO",
    "destination": "NYC",
    "depart": "2026-04-15"
  },
  "rctx": { "req_ip": "198.51.100.42" },
  "cnf": { "jkt": "ToolJKT-0ZcOCORZNYy9ZhHi" },
  "act": {
    "sub": "https://tools.travel-provider.example/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service",
    "act": {
      "sub": "https://agents.enterprise.example/travel-assistant",
      "iss": "https://as.enterprise.example",
      "sub_profile": "ai_agent"
    }
  }
}
~~~

The presenter binding changes at this step: `cnf.jkt` is now `ToolJKT` because the Booking Tool is the current presenter.


## Step 6: Booking Tool Calls Inventory Service

~~~
GET /inventory?origin=SFO&dest=NYC&depart=2026-04-15 HTTP/1.1
Host: internal.travel-provider.example
Txn-Token: <txn-token>
Workload-Identity-Token: <booking-tool-wit>
Workload-Proof-Token: <tool-wpt-with-wth-and-tth>
~~~

The Inventory Service validates the WIT and WPT and then authorizes Alice and the Booking Tool as the subject and the immediate actor, respectively.  It uses `req_wl` as supporting workload context.  The Travel Assistant remains prior-actor context and is not used for access control.


## Summary of Token Transformations

| Step | Token | `sub` | `req_wl` | `act.sub` (outermost) | `act.act.sub` (nested) | `cnf.jkt` |
|------|-------|-------|----------|-----------------------|------------------------|-----------|
| 1 | ID Token | Alice |  |  |  |  |
| 2 | ID-JAG | Alice |  | Travel Assistant |  | AgentJKT |
| 3 | Access Token | Alice |  | Travel Assistant |  | AgentJKT |
| 4 | (API call) | Alice |  | Travel Assistant |  | AgentJKT |
| 5 | Transaction Token | Alice | Booking Tool | Booking Tool | Travel Assistant | ToolJKT |
| 6 | (internal call) | Alice | Booking Tool | Booking Tool | Travel Assistant | ToolJKT |

Key observations:

*  In this example, `sub` (Alice) is unchanged across all trust domains and token transformations.
*  The presenter-binding key changes once, at Step 5, when the TTS rebinds the Transaction Token to the Booking Tool's key.
*  At Step 5 the TTS creates a new outermost `act` for the Booking Tool and nests the prior `act` chain beneath it.


# Acknowledgments
{:numbered="false"}

The author thanks the OAuth Working Group for the specifications on which this profile builds.

# Document History
{:numbered="false"}

[[ To be removed from the final specification ]]

-01

* Restructured and tightened the text: each rule has one home, dependencies are cited rather than restated, tables cover claim roles, token types, and error mappings, Token Exchange inputs share one processing algorithm, and Security Considerations point to the rules they rely on.  Removed the Conformance section, which restated those rules.
* Revised the Introduction to state what Token Exchange leaves open and the ID-JAG extension point this profile fills.
* Added hop and visible-hop terminology; only an introspection server filters the visible `act` chain.
* Required `client_id` in JWT access tokens, per {{RFC9068}}, and limited the JWT access token structure to JWT access token output.
* Reworked presenter transitions: bearer continuation by the authenticated outermost actor, rebind by a direct presenter credential or the `may_act` client, rejection of delegated requests that fit neither mode, a new Token Exchange actor only from an `actor_token` or the `may_act` path, issuance without `act` for requests that are not delegated, a refresh token's recorded `act` chain kept as inbound delegation state, and new-presenter proof only for sender-constrained output.
* Kept nested `act` informational for access control, per {{Section 4.1 of RFC8693}}: inner actors are prior-actor context only.
* Rebind now supersedes a sender-constrained assertion grant's binding and makes the grant single-use; grant replay protection depends on an enforced grant-level sender constraint and the (`iss`, `jti`) pair.
* Aligned error codes with {{Section 2.2.2 of RFC8693}} and {{Section 3.1 of RFC7523}}: `invalid_request` on Token Exchange, `invalid_grant` on JWT bearer grants, and `actor_unauthorized` for actor-policy denials, with explicit error precedence.
* Set an ID-JAG's `aud` to the Resource Authorization Server's issuer identifier, capped the scope issued from an assertion-grant or Transaction Token `subject_token`, required `jti` in assertion grants, and required an ID Token's `aud` to match the authenticated client.
* Aligned Transaction Token rules with {{I-D.ietf-oauth-transaction-tokens}}: `act` only for delegation, `sub` unchanged on replacement, scope within the `subject_token`'s authority, and presenter proof and rejection per the deployment profile.
* Used `may_act.iss` as the identifier context when present, and preserved inherited actor objects, their extension and confirmation members, and unrecognized `sub_profile` values unchanged.
* Resource servers validate mTLS-bound tokens, require `iss` on every visible actor object when conformance is required, identify the outermost actor by its (`act.iss`, `act.sub`) pair, and treat a missing `act` in introspection as an error only when delegation is indicated, and enforce the depth limit.
* Clarified actor-token classification and validation for standalone JWT client assertions, JWT access tokens, and WIMSE workload credentials, including each type's `act.iss` and a client check for a bearer JWT access token.
* Tightened the metadata triggers and added the ID-JAG token type to `requested_token_types_supported`.
* Removed BCP 14 keywords from guidance no other party can observe, made RFC 6749, RFC 7800, OpenID Connect Core, and ID-JAG normative, corrected citations, and aligned the examples on one travel scenario.

-00

* Initial version.

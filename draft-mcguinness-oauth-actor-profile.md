---
title: "OAuth Actor Profile for Delegation"
abbrev: "OAuth Actor Profile"
category: std
docname: draft-mcguinness-oauth-actor-profile-latest
submissiontype: IETF
number:
date: 2026-04-30
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
  RFC6750:
  RFC7009:
  RFC7519:
  RFC7521:
  RFC7523:
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
    target: https://www.ietf.org/archive/id/draft-mora-oauth-entity-profiles-01.txt

informative:
  RFC6749:
  RFC9700:
  RFC8792:
  I-D.parecki-oauth-jwt-dpop-grant:
    title: "JWT Authorization Grants with DPoP"
    author:
     -
        fullname: Aaron Parecki
        organization: Okta
    date: 2026-01-30
    target: https://datatracker.ietf.org/doc/html/draft-parecki-oauth-jwt-dpop-grant-01
  I-D.ietf-oauth-identity-chaining:
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
    date: 2026-04-22
    target: https://www.ietf.org/archive/id/draft-ietf-oauth-identity-assertion-authz-grant-03.txt
  OpenID.Core:
    title: "OpenID Connect Core 1.0"
    author:
      org: OpenID Foundation
    date: 2014-11-08
    target: https://openid.net/specs/openid-connect-core-1_0.html
  OpenID.Federation:
    title: "OpenID Federation 1.0"
    author:
      org: OpenID Foundation
    date: 2024-05-01
    target: https://openid.net/specs/openid-federation-1_0.html

...

--- abstract

This document defines a common representation of delegated actors in OAuth JSON Web Token (JWT) assertion grants, JWT access tokens, and Transaction Tokens.  It profiles the `act` claim defined by OAuth 2.0 Token Exchange, requires issuer-scoped actor identifiers, and uses `sub_profile` to classify actor entity types.  It specifies token processing, delegation-chain propagation, sender-constraint handling, and discovery metadata.  Delegation approval and trust policy remain deployment-specific.

--- middle

# Introduction

Delegated requests can pass through several services and trust domains.  Each recipient needs to distinguish the subject whose authorization is exercised, the actor exercising it, and the OAuth client requesting the token.  Without a common profile, deployments face four interoperability gaps:

*  **No standard entity classification.** `sub` is overloaded across end users, service accounts, AI agents, and workloads, with no classification that supports deterministic cross-domain policy.
*  **Inconsistent actor representation across token types.** Actor context, including actor key material, has no representation that survives transformation among JWT assertion grants, JWT access tokens, and Transaction Tokens.
*  **Implicit delegation via client identity.** A client registration alone may not identify the actor, particularly when one registration serves several agents or workloads, when requests pass through intermediaries, or when tokens cross trust domains.
*  **No discovery for actor-profile support.** Neither AS metadata {{RFC8414}} nor Protected Resource Metadata {{RFC9728}} defines parameters for advertising actor-profile support.

OAuth 2.0 Token Exchange {{RFC8693}} defines the `act` claim for representing actors and delegation chains.  This document profiles that claim across JWT assertion grants, JWT access tokens, and Transaction Tokens.  It defines:

*  Issuer-scoped actor identifiers and entity classification using `sub_profile`.
*  Rules for validating and preserving actor information across token transformations.
*  Presenter continuation and rebind rules for sender-constrained tokens, including upgrades from bearer tokens.
*  Resource server processing and metadata for advertising profile support.
*  Extension points for companion profiles that provide additional delegation evidence.

The profile applies to human, service, workload, and AI agent delegation.  The requirements of the underlying specifications, including {{RFC8693}}, {{RFC9068}}, {{RFC9449}}, and {{I-D.ietf-oauth-transaction-tokens}}, continue to apply unless stated otherwise.  [Profile Scope](#profile-scope) describes the supported token paths and the boundary between representation and authorization policy.

## Illustrative Use Case

Alice authorizes an AI travel agent to book a trip.  The enterprise AS issues a credential with Alice as `sub` and the agent as `act`.  The agent presents it to a booking provider's AS for an access token, and the provider then issues a Transaction Token for an internal booking tool.  Alice remains the subject; the tool becomes the outermost actor, and the agent becomes an inner actor.  [The cross-domain example](#appendix-cross-domain) shows the complete flow.

## Relationship to Related Work

*  **OAuth Token Exchange ({{RFC8693}})** defines the `act` claim and exchange mechanism profiled here.
*  **Identity Chaining ({{I-D.ietf-oauth-identity-chaining}})** propagates subject identity across domains and can be combined with this profile's actor representation.
*  **Identity Assertion JWT Authorization Grant (ID-JAG, {{I-D.ietf-oauth-identity-assertion-authz-grant}})** defines issuance and consumption of JWT authorization grants.  This document supplies actor-delegation processing through its Token Exchange and JWT assertion-grant rules.
*  **OAuth Entity Profiles ({{I-D.mora-oauth-entity-profiles}})** defines the classification claims, metadata, and registry used by this profile.
*  **Transaction Tokens ({{I-D.ietf-oauth-transaction-tokens}})** defines the token and service model extended here with actor claims and processing rules.
*  **WIMSE Workload Identity ({{I-D.ietf-wimse-workload-creds}}{{I-D.ietf-wimse-wpt}})** supplies workload credentials and proofs used in [the cross-domain example](#appendix-cross-domain).  This profile also supports other presenter-authentication mechanisms.

# Conventions and Definitions {#conventions}

{::boilerplate bcp14-tagged}

This document uses the OAuth terminology defined in {{RFC6749}} and {{RFC8693}}, and the terms Transaction Token and Transaction Token Service (TTS) defined in {{I-D.ietf-oauth-transaction-tokens}}.  AS and RS denote authorization server and resource server, respectively.

The following terms are used in this document:

Actor:
: The party that is actively making a request.  When delegation is present, the actor is distinct from the subject; the subject is the principal on whose behalf the actor is acting.

Subject:
: The principal whose authorization is being exercised.  In a delegated token, the subject is the original authorizing party (e.g., an end-user or an upstream service), not the party making the immediate network request.

Delegation:
: The act by which a principal authorizes another party (the actor) to exercise a subset of the principal's rights.

Cross-Domain Delegation:
: Delegation in which the subject and actor are governed by different trust domains or identifier namespaces.  Deployment policy determines whether a token represents cross-domain delegation, using issuer context, actor identifiers, and applicable trust agreements.  The top-level `iss` alone is not always sufficient.

Actor Authorization at the Resource Server:
: An authorization policy evaluation that considers both the subject and the actor, and the relationship between them, as policy inputs.  Under this profile, the relevant actor is ordinarily the outermost actor.

Delegation Chain:
: The sequence of actors representing how authorization has flowed from the subject principal (`sub`) to the first actor (innermost `act`) through any intermediate parties to the immediate actor (outermost `act.sub`).  The chain is conveyed structurally as the nested `act` claim.

Outermost Actor:
: The `act` object at the top level of the delegation chain (the one not nested inside any other `act` object).  When a delegation chain of depth greater than one is present, the outermost actor identifies the immediate bearer of the token.

Local Policy:
: Rules or decisions made by an AS, RS, or organization outside this specification, such as delegation approval, scope reduction, identifier mapping, and entity-profile acceptance.

Identifier Reconciliation:
: Applying configured mapping rules to determine whether identifiers from different claims or namespaces refer to the same entity.  A recommendation to perform identifier reconciliation means the implementation SHOULD apply those rules.  String similarity or shared naming patterns do not establish equivalence.  If no applicable mapping exists or reconciliation fails, equivalence is not established: the identifiers MUST be treated as distinct, and an implementation MUST reject a request or token whose processing requires them to identify the same entity.

Examples in this document are illustrative and focus on actor-profile-related claims and processing.  They may omit unrelated claims, parameters, or validation steps required by the underlying specifications for a complete deployment.

This document uses dot-path notation to refer to nested claim values.  For example, `act.sub` refers to the `sub` member of the `act` object, and `act.act.sub` refers to the `sub` member of the `act` object nested within the outer `act` object (the immediately prior actor in a depth-2 chain).


# Actor Profile for Delegation {#actor-profile}

## Overview

When an implementation uses this profile to represent an actor distinct from the subject, it MUST apply the requirements in this section.  The absence of an explicit inbound actor credential MUST NOT be interpreted as making the OAuth client the delegated actor.

## Profile Invariants

| Claim | Meaning |
|-------|---------|
| Top-level `sub` | Subject whose authorization is exercised |
| Outermost (`act.iss`, `act.sub`) | Canonical identifier of the immediate actor |
| Inner `act` objects | Prior actors, ordered from most recent to earliest |
| `client_id`, `azp` | OAuth client identity |
| Top-level `cnf` | Current presenter's key or certificate binding |

The actor identifier context (`act.iss`) is defined in [Actor Object Structure](#actor-object-structure).  Inner actors have the trust properties described in [Carry Prior-Actor Context](#carry-prior-actor-context).

When (`act.iss`, `act.sub`) identifies the same entity as the token's (`iss`, `sub`), consumers MUST NOT infer a delegation relationship.  Comparing `act.sub` with `sub` alone is insufficient; the identifier contexts also matter.

## Profile Scope {#profile-scope}

### Representation and Policy {#representation-and-policy}

This profile standardizes actor representation, propagation, validation, and discovery.  Delegation approval, trust frameworks, and identifier mappings remain deployment-specific.  Cross-domain deployments need agreements covering permitted delegation relationships and identifier namespaces.

This document does not require every deployment to enforce authorization of the (`sub`, outermost `act.sub`) pair on every request.  [Actor Authorization](#actor-authorization) describes when an RS applies that policy, including its recommendation to enforce it for security-sensitive delegated access.

### Token Format Scope

Conforming outputs are JWT assertion grants, JWT access tokens, and Transaction Tokens.  [Token Introspection](#token-introspection) defines optional equivalent response claims for delegated opaque access tokens; this compatibility path does not make the opaque token itself conformant.

Opaque access tokens as Token Exchange inputs are outside the interoperable scope.  An AS MAY translate their introspection results into local inputs under deployment-specific rules.

### Supported Token Types and Request Semantics

| Role | Supported credentials |
|------|-----------------------|
| Token Exchange `subject_token` | JWT assertion grant, JWT access token, ID token, refresh token, Transaction Token |
| Token Exchange `actor_token` | Workload identity credential, JWT client assertion, non-delegated JWT access token |
| TTS `subject_token` | JWT assertion grant, JWT access token, Transaction Token |
| Issued token | JWT assertion grant, JWT access token, Transaction Token |

Subject to endpoint policy and the underlying grant mechanism, implementations MAY transform supported inputs into outputs for which this document defines issuance rules.  They need not support every combination.  For each supported path, actor information MUST be validated and preserved according to the output rules in [JWT Assertion Grant Output](#jwt-assertion-grant-issuance), [JWT Access Token Output](#jwt-access-token-propagation), or [Transaction Token Output Rules](#transaction-token-output-rules).

The request parameters and semantics of {{RFC8693}} continue to apply.  The `may_act` claim is an optional delegation-authorization input, with the restrictions in [`may_act`](#may-act).  It neither establishes actor identity nor propagates to the output token.

For worked examples of same-domain service delegation and cross-domain delegation, see [the service-to-service example](#appendix-service-to-service) and [the cross-domain example](#appendix-cross-domain).

## Actor Object Structure {#actor-object-structure}

An actor object conforming to this profile is a JSON object that is the value of the `act` claim.  In addition to the `sub` claim required by {{RFC8693}}, a profile-conformant actor object MUST contain an `iss` claim and SHOULD contain a `sub_profile` claim when the issuer can authoritatively classify the actor's entity type.  An `act` object that omits `iss` conforms to {{RFC8693}} but does not conform to this profile; handling of such objects is specified in [Migration and Adoption](#migration-and-adoption).

~~~
act-object = {
  "sub"           : StringOrURI,        ; REQUIRED
  "iss"           : StringOrURI,        ; REQUIRED
  ? "sub_profile" : JSON String,        ; RECOMMENDED
  * StringOrURI => any                  ; extension claims
}
~~~

`sub`:
: REQUIRED.  The subject identifier of the actor, as defined in {{RFC8693, Section 4.1}}.  This value identifies the acting party.  It is a StringOrURI as defined in {{RFC7519}}.

`iss`:
: REQUIRED.  The issuer or namespace context for `act.sub`, expressed as a StringOrURI {{RFC7519}}.  Together, (`act.iss`, `act.sub`) form the canonical actor identifier.  For any actor identifier scheme, `act.iss` MUST identify the context used when assigning or asserting that identifier.

  Implementations MUST NOT interpret `act.iss` as the current token issuer, credential issuer, or hop-provenance marker.  These entities can coincide, but have distinct roles.  HTTPS URLs and workload-identity URNs are examples of possible context identifiers.

  For example, a TTS at `https://tts.travel-provider.example` can issue a token whose booking-tool actor has `act.iss` set to `https://as.travel-provider.example` when local policy uses that AS's identifier namespace for booking tool identifiers.  The TTS signs the token; the AS supplies the namespace for the tool's identifier.

`sub_profile`:
: RECOMMENDED.  A space-delimited list of entity profile values classifying the actor identified by `act.sub`, as defined in Section 4.2 of {{I-D.mora-oauth-entity-profiles}}.  Values used within `act` objects MUST be registered with the "Actor Profile" usage location in the OAuth Entity Profiles registry (Section 14.1 of {{I-D.mora-oauth-entity-profiles}}) or be privately defined collision-resistant values.

  If the acting entity fits more than one profile, multiple values MAY be included as a space-delimited string (e.g., `"service ai_agent"`).  Interoperability requirements and implementation guidance for multi-value strings are defined in {{I-D.mora-oauth-entity-profiles}}.

  When `sub_profile` is absent from an `act` object, implementations MUST NOT assume a specific entity type for the actor; resource servers that enforce entity-type-based access control MUST treat an absent `sub_profile` as an unclassified actor and SHOULD apply the more restrictive policy applicable to unknown entity types.

  The `sub_profile` claim MAY also appear as a top-level JWT claim outside any `act` object to classify the entity type of the token's `sub`; it applies exclusively to `sub` and does not affect `sub_profile` values within `act` objects.  Issuers SHOULD include a top-level `sub_profile` when they can authoritatively classify the subject entity type.

The current presenter's binding is carried in top-level `cnf`; see [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation).  Confirmation members inside `act` have no proof-of-possession semantics under this profile.  Per-actor key provenance requires another specification.

The `client_profile` claim defined in {{I-D.mora-oauth-entity-profiles}} classifies the OAuth client and MUST NOT appear within an `act` object.  Client classification belongs at the top level of the token.  An AS or RS that encounters a `client_profile` member inside an `act` node MAY reject the token or ignore the offending member; it MUST NOT treat it as a valid actor classification.

When an `act` object contains extension members beyond those defined in this document, issuers and consumers MUST ignore unrecognized members unless another specification or local policy defines their meaning.  An issuer that preserves a validated delegation chain copies unrecognized extension members in inherited `act` objects unchanged, as [Preserve Inbound Chain](#preserve-inbound-chain) requires.  However, [Companion Profiles and Extension Points](#companion-profile-extensibility) recommends that companion profiles needing independently verifiable provenance, per-hop receipts, or other chain-wide state use top-level extensions rather than inherited `act`-object extension members.


## Delegation Chains {#delegation-chains}

Delegation chains MUST use nested `act` objects as specified in {{RFC8693, Section 4.1}}.  The outermost object identifies the immediate actor; the innermost identifies the first actor authorized by the subject.  The chain records prior actors under the conveying issuer's trust, without independently proving each hop.  This profile defines one linear chain per token; concurrent delegations use separate tokens.

This document uses the following terminology consistently:

*  A **hop** is a single `act` object in a delegation chain.  The number of hops in a chain equals the chain's delegation depth.
*  A **visible hop** is a hop that appears in the token's `act` chain as received by a recipient, after any filtering by an introspection server or upstream issuer.
*  A **single-hop actor object** is an `act` object with no nested `act`; it represents delegation depth 1.
*  An **inbound delegation chain** is the complete `act` structure received in an inbound token, whether depth 1 or greater.
*  A **preserved delegation chain** is an inbound delegation chain that an issuer has validated and copied into a newly issued token without rewriting inherited actor entries.
*  A **new outermost actor** is the actor object created by the current issuer to represent the newly identified outermost actor for the token it is issuing.

~~~json
{
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
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
}
~~~
Alice (`sub`) authorized the travel assistant (inner `act`), which delegated to the booking tool (outermost `act`).  The booking tool is the current presenter.

Delegation depth is defined as the number of `act` objects in the chain, counting from the outermost.  A token with a single `act` object and no nested `act` within it has depth 1; each additional level of nesting adds 1.  Depth is counted on the resulting chain after any new outermost `act` is added, not on the inbound token.

Depth 1 is the minimum interoperable depth.  Implementations for cross-domain multi-hop use SHOULD support at least depth 4, and should document their maximum.  Depth-1 implementations are conformant but cannot support multi-hop chains.  Same-domain deployments can use a shallower maximum when sufficient for their architecture.

Implementations MUST define and enforce a local maximum delegation depth.  Implementations that receive a token exceeding their configured local maximum MUST reject it with `invalid_request`.  When a request would result in a chain exceeding that limit, the AS MUST reject with `invalid_request`; it MUST NOT silently truncate the chain.

A token represents delegation when the party exercising the token's authorization at runtime (the actor) is distinct from the token subject (`sub`) and the actor has been authorized by the subject to do so.  The conditions that establish this are:

1.  A validated `actor_token` identifying a distinct actor was present in the exchange request that produced this token.
2.  An inbound `subject_token` from a trusted upstream issuer already carried an `act` chain, establishing that delegation was present before the current exchange.
3.  The issuing AS has an independent delegation basis such as a pre-registered grant, explicit consent record, a `may_act` claim in a validated upstream token (see [`may_act`](#may-act)), or a policy rule establishing that the current client or actor is acting as a distinct party on behalf of `sub` (see [JWT Access Tokens](#jwt-access-tokens) for the outside-Token-Exchange case).

When a token represents delegation, the `act` claim MUST be present and MUST conform to [Actor Object Structure](#actor-object-structure).  When none of the above conditions holds, the token does not represent delegation and the `act` claim MUST be omitted.  The AS MUST NOT include `act` solely because `sub` and the OAuth client identifier differ; the distinction between `sub` and `client_id` is expected and does not by itself constitute delegation under this profile.


## Delegation Chain Validation and Construction {#delegation-chain-algorithm}

An AS MUST apply this algorithm on paths that require or claim actor-profile conformance.  The invoking grant, Token Exchange, or TTS rules supply token-specific preconditions and delegation-authorization requirements.  Nonconforming actor objects, including those missing `iss`, MUST NOT enter this algorithm; [Migration and Adoption](#migration-and-adoption) defines their handling.

### Terminology

*  **Depth**: the number of `act` objects in the resulting chain; see [Delegation Chains](#delegation-chains).
*  **Security-relevant entry**: an inner actor used by local policy for authorization, scope determination, or identity mapping during issuance.
*  **Prior-actor context**: an inner actor preserved for audit or downstream use without affecting the current issuance decision.

### Validation Steps

The AS applies the validation steps in the following order:

1.  Validate the carrier token using [Validate Carrier Token](#validate-carrier-token).
2.  Validate the outermost actor using [Validate Outermost Actor](#validate-outermost-actor).
3.  If inner actors are used as inputs to issuance decisions, validate those entries using [Validate Inner Actors Used for Decisions](#validate-inner-actors-used-for-decisions).
4.  For inner actors preserved only as prior-actor context, apply [Carry Prior-Actor Context](#carry-prior-actor-context).
5.  Enforce the configured maximum chain depth using [Enforce Depth Limit](#enforce-depth-limit).

#### Validate Carrier Token {#validate-carrier-token}

The AS MUST validate the token carrying the inbound delegation chain per the type-specific rules applicable to that token before extracting actor claims.

#### Validate Outermost Actor {#validate-outermost-actor}

For the outermost `act` object the AS MUST:

1.  Verify that both `act.sub` and `act.iss` are present.  If either is absent, reject with `invalid_request`.
2.  Verify that local policy trusts the token issuer to assert (`act.iss`, `act.sub`); otherwise, reject with `invalid_grant`.  This does not make `act.iss` the token issuer or independently authenticate prior hops.  [Trusting Actor Identifier Pairs](#act-iss-authority-guidance) gives examples of this deployment-specific trust decision.
3.  Evaluate delegation under local policy:

    *  When [extending the chain](#extend-chain-with-new-actor), the AS MUST confirm that the new actor is authorized to act for `sub`, for example through a grant, consent record, or policy rule.
    *  When [preserving a validated chain](#preserve-inbound-chain) from a trusted issuer, the AS SHOULD evaluate the preserved relationship.  Upstream evaluation suffices for baseline interoperability.
    *  If a required relationship is prohibited or cannot be confirmed, reject with `actor_unauthorized`.

[Authorization Grant Processing](#jwt-assertion-grants-processing) adds requirements for JWT assertion grants.

#### Validate Inner Actors Used for Decisions {#validate-inner-actors-used-for-decisions}

Interoperable processing under this profile is defined around `sub` and the outermost `act.sub`.  If local policy additionally uses an inner `act` object as an input to issuance decisions, the AS MUST validate that entry's `act.sub` and `act.iss` pair and MUST evaluate its delegation relationship, as in [Validate Outermost Actor](#validate-outermost-actor), before using it as a security input.  Failures use that step's errors: `invalid_request` for a missing `act.sub` or `act.iss`, `invalid_grant` when the token issuer is not trusted to assert the identifier pair, and `actor_unauthorized` when the delegation relationship is prohibited or cannot be confirmed.  Such use of inner actors is deployment-specific.

#### Carry Prior-Actor Context {#carry-prior-actor-context}

For inner `act` objects preserved solely as prior-actor context without being used for any issuance decision, the AS MAY rely on trust in the outer token issuer established by [Validate Carrier Token](#validate-carrier-token) rather than independently validating each hop.  The AS MUST NOT treat preserved prior-actor context as independently authenticated; an inner `act` entry carried in a token is endorsed only by the outer token issuer's signature, not by independent verification at each prior hop.

#### Enforce Depth Limit {#enforce-depth-limit}

Compute the depth of the resulting chain, including any new outermost actor added by [Extend Chain with New Actor](#extend-chain-with-new-actor).  If that depth exceeds the locally configured maximum ([Delegation Chains](#delegation-chains)), reject with `invalid_request`.

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

The AS MUST set the new actor's `act.iss` and MUST NOT change any inherited actor field.  It MUST preserve the entire inbound chain; [Enforce Depth Limit](#enforce-depth-limit) rejects a resulting chain that exceeds the local maximum.  With no inbound chain, the new chain has depth 1.

If the new actor has the same (`act.iss`, `act.sub`) pair as the inbound outermost actor, the AS MAY instead apply [Preserve Inbound Chain](#preserve-inbound-chain) to avoid a duplicate entry.

Detection of identifier reappearance deeper in the inbound chain (for example, the same actor appearing in both inner and outer positions of a longer chain) is not standardized by this profile.  An AS MAY apply local policy to such cases; the chain-construction algorithm itself neither requires nor prohibits cycle detection.

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

The rules in this section apply to all supported token types, in addition to their type-specific proof-of-possession (PoP) requirements.  [Presenter Transition Model](#token-exchange-presenter-model) defines continuation and rebind for Token Exchange.  DPoP nonce handling follows {{RFC9449, Section 8}} unchanged.

Per-actor confirmation members and prior-hop key provenance are outside this profile; see [Companion Profiles and Extension Points](#companion-profile-extensibility).

### Top-Level `cnf` Governs the Current Presenter

The top-level `cnf` claim of any token identifies the key or certificate of the current presenter.  When delegation is present, that current presenter is the party identified by the outermost `act` claim: when DPoP ({{RFC9449}}) is used, the top-level `cnf.jkt` MUST identify that party's key; when mTLS ({{RFC8705}}) is used, the top-level `cnf.x5t#S256` MUST identify that party's certificate.  The AS or RS MUST validate proof of possession against the top-level `cnf`.  Confirmation-style members that appear inside an `act` object due to another specification do not have standardized proof-of-possession semantics under this document.

### Token Exchange Continuation

Presenter continuation requires a PoP-capable `subject_token` with top-level `cnf`.  The requester MUST prove possession of that binding using the mechanism applicable to the token type and deployment.  This profile does not define continuation without that binding.

### Token Exchange Rebind

Presenter rebind requires a validated `actor_token` whose top-level `sub` identifies the new presenter, as specified in [Actor Tokens](#actor-tokens).

*  The issuer validates the credential per {{RFC8693, Section 2.1}} and MUST validate any proof required by its profile or deployment, whether or not the output token is sender-constrained.
*  When the output token is sender-constrained, the issuer MUST validate proof of possession for the new presenter.  A bearer output does not waive validation of the credential or of any proof its profile requires.  A sender-constrained `subject_token` does not, by itself, require proof for its prior presenter during rebind.

Actors that become presenters therefore need a direct credential: a workload credential, JWT client assertion, or non-delegated JWT access token.

### Bearer-to-PoP Upgrade

When the inbound `subject_token` is a bearer credential or an identity-only credential and the request supplies a validated `actor_token` establishing a new presenter, the issuer MAY issue a sender-constrained output token bound to that new presenter.  The absence of inbound top-level `cnf` creates no continuity obligation in this case.

# JWT Assertion Grants {#jwt-assertion-grants}

This section defines the actor-profile structure and authorization-grant processing rules for JWT assertion grants.  JWT access-token structure, Token Exchange processing, and Transaction Token Service processing are defined in later sections.

## Structure {#jwt-assertion-grants-structure}

These requirements apply to JWT authorization grants under {{RFC7521}} and {{RFC7523}}, including ID-JAG {{I-D.ietf-oauth-identity-assertion-authz-grant}}.  The grant profile defines issuance and exchange; this document defines actor representation and delegation processing.

A JWT authorization grant MAY carry an `act` claim conforming to [Actor Object Structure](#actor-object-structure).  Actor claims in JWT client authentication assertions are out of scope for this document.  Explicit delegation is represented by `act`, even when the issuer derives the actor from authenticated client context.

The following claims are defined for a JWT assertion grant that carries actor-profile delegation.  Claims not listed here follow the requirements of {{RFC7521}} and {{RFC7523}}.

`iss` (REQUIRED):
: Identifies the assertion issuer.  MUST be authorized by local policy to assert the relationship between `sub` and `act.sub`.

`sub` (REQUIRED):
: The principal on whose behalf the grant is being made.

`sub_profile` (RECOMMENDED):
: Classifies the entity type of `sub`.  MUST conform to the values defined in [Actor Profile for Delegation](#actor-profile).

`act` (REQUIRED when delegation is asserted):
: The actor object identifying the entity exercising the subject's delegated rights.  MUST conform to the actor object structure defined in [Actor Profile for Delegation](#actor-profile).

`cnf` (REQUIRED when sender-constrained; otherwise OPTIONAL):
: When the JWT assertion grant is sender-constrained, the assertion MUST carry a top-level `cnf` claim identifying the binding: `cnf.jkt` per {{RFC9449}} when DPoP is used, or `cnf.x5t#S256` per {{RFC8705}} when mTLS is used.  When the assertion is not sender-constrained, top-level `cnf` is OPTIONAL unless required by another profile or local policy.

When the assertion or request context also identifies an OAuth client via `client_id`, `azp`, or an authenticated client credential, that client identity does not substitute for `act.sub`, as required by [Client Identity and Delegation](#client-identity-delegation) (see also [Authorization Grant Processing](#jwt-assertion-grants-processing)).

Before sending JWT assertion grants carrying actor-profile claims, a client needs to confirm the AS's support for the actor-determination model through deployment documentation, prior agreement, or discovery; for ID-JAG, that includes support for the actor-delegation extension model defined by this document.

The following example shows an AS-issued assertion grant, which is the recommended pattern.  The Enterprise IdP AS performed Token Exchange, authenticated the agent as the OAuth client, established the delegation relationship under local policy, and signed the assertion.  `act.iss` equals the token `iss` here because the enterprise AS's issuer identifier is also the actor identifier context for the agent:

~~~json
{
  "iss": "https://as.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "aud": "https://as.resource-domain.example/token",
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

The top-level `sub_profile` classifies the JWT's `sub`; `sub_profile` within `act` classifies the actor.  The top-level `cnf.jkt` binds the assertion to the agent's DPoP key.

In this example, the receiving AS trusts the enterprise AS both to issue the grant and to assert the actor identifier pair.

This document defines two issuer patterns:

*  an AS-issued delegated assertion, where JWT `iss` is a trusted AS and `act.sub` identifies the actor (recommended);
*  an assertion carrying a pre-existing nested `act` chain, where the current JWT `iss` is a trusted AS carrying forward prior actor assertions.

The issuing AS sets a new actor's `act.iss` to the issuer or namespace context in which that actor's `act.sub` is interpreted, as required by [Actor Object Structure](#actor-object-structure); for actors registered in the AS's own namespace, this is often the AS's own issuer URI.

A deployment MAY additionally accept a self-issued actor assertion when explicitly enabled by another specification or local policy, but that behavior is outside the scope of this document.  Implementations MUST reject self-issued assertion grants by default; see [Self-Issued Authorization Grants](#security-self-issued-grants) for the security controls any such deployment needs to establish independently.

## Authorization Grant Processing {#jwt-assertion-grants-processing}

When an AS receives a JWT assertion grant containing an `act` claim:

1.  The AS MUST validate the assertion per {{RFC7523}}, including signature, `iss`, `sub`, `aud`, `exp`, and `jti`.

    *  **Non-sender-constrained grants**: When neither a DPoP proof ({{RFC9449}}) nor an mTLS client certificate ({{RFC8705}}) is required at the token endpoint, the AS MUST reject any assertion whose `jti` has already been accepted within the assertion's validity window.
    *  **Sender-constrained grants**: The AS SHOULD additionally apply `jti` replay prevention as defense-in-depth, consistent with {{RFC7523}}.

2.  The AS MUST verify that the JWT `iss` is trusted under local policy to assert delegation on behalf of the actor identified by `act.sub`.

    > Note: Under this document the JWT `iss` is expected to be a trusted AS.  Self-issued grants, where the acting entity is also the token issuer, are a deployment-specific extension outside the scope of this document; see [Self-Issued Authorization Grants](#security-self-issued-grants).

3.  The AS MUST verify that the JWT `iss` is trusted under local policy to assert the (`act.iss`, `act.sub`) actor identifier pair.

    *  If `act.iss` is absent: reject with `invalid_request` (structural violation).
    *  If the JWT `iss` is not trusted to assert the actor identifier pair: reject with `invalid_grant`.

    > Note: See [Validate Outermost Actor](#validate-outermost-actor) for the trust-validation framing and [Trusting Actor Identifier Pairs](#act-iss-authority-guidance) for non-normative examples.

4.  The AS MUST evaluate whether the identified actor is authorized to exercise delegation on behalf of `sub`.  The required strength of that evaluation depends on how the outermost actor was introduced:

    *  **New actor**: When the request supplies an `actor_token` or self-issued assertion that introduces a new `act.sub` not carried by the inbound chain, the AS MUST confirm the delegation relationship under local policy (for example, a pre-registered grant, explicit consent record, or policy rule).
    *  **Preserved chain**: When the request preserves an existing chain from a validated, trusted upstream issuer, the issuer trust established in step 2 provides the baseline assurance; the AS SHOULD additionally evaluate under local policy but is not required to do so for baseline interoperability.

    In either case:

    *  If the delegation relationship is prohibited by AS policy or cannot be confirmed: reject with `actor_unauthorized`.

5.  If the inbound assertion's `act` object contains a nested `act` claim (indicating that the asserted actor is itself a delegatee), the AS MUST handle the inner chain as follows:

    *  **Propagation decision**: The AS SHOULD propagate it by preserving the nested structure, provided the total resulting chain depth does not exceed the limit in [Delegation Chains](#delegation-chains).  If the AS does not accept pre-chained assertions, it MUST reject the request.

    *  **Entries used by the AS for issuance decisions**: Interoperable processing is defined around `sub` and the outermost `act.sub`.  If local policy additionally uses an inner `act` object for authorization, scope determination, or another issuance decision, [Validate Inner Actors Used for Decisions](#validate-inner-actors-used-for-decisions) applies before the AS uses that entry as a security input.  Such use of inner `act` objects is deployment-specific rather than part of the baseline interoperable behavior of this profile.

    *  **Preserved prior-actor context**: For inner `act` objects the AS preserves only as prior-actor context, apply [Carry Prior-Actor Context](#carry-prior-actor-context).  Downstream authorization interoperability is defined around `sub` and the outermost `act.sub`.

6.  The AS MUST verify proof of possession according to the token-endpoint mechanism in use and the top-level `cnf` semantics in [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation).

    *  **DPoP**: When the inbound assertion grant is DPoP-bound, it MUST carry a top-level `cnf.jkt`; reject with `invalid_request` if absent.  The AS MUST:
       *  Verify the DPoP proof is valid per {{RFC9449}} with `htm="POST"` and `htu` equal to the AS token endpoint URI.
       *  Verify that the JWK SHA-256 thumbprint of the public key in the DPoP proof matches the assertion's `cnf.jkt` ({{RFC9449, Section 6.1}}), as in the proof checks of {{RFC9449, Section 4.3}}.
       *  Use the assertion's `cnf.jkt` as set by the upstream issuer; MUST NOT substitute a locally registered key.
       *  Reject with `invalid_dpop_proof` or `invalid_grant` if the proof is absent or invalid.

       > Note: The `ath` claim is not applicable at the token endpoint and MUST NOT be required.  See also {{I-D.parecki-oauth-jwt-dpop-grant}} for related work on DPoP-bound JWT grants.

    *  **mTLS**: When the inbound assertion grant is mTLS-bound, it MUST carry a top-level `cnf.x5t#S256`; reject with `invalid_request` if absent.  The AS MUST:
       *  Validate the client certificate presented at the token endpoint against `cnf.x5t#S256`.
       *  Use the `cnf.x5t#S256` value set by the upstream issuer; MUST NOT substitute a locally registered certificate.
       *  Reject per {{RFC8705}} if the presented certificate does not match.
    *  When this JWT assertion grant is later used as a `subject_token` in Token Exchange, presenter continuation and presenter rebind are determined by [Presenter Transition Model](#token-exchange-presenter-model) and [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation), not by nested `act` contents.

7.  If the assertion or authenticated request context identifies an OAuth client separately from `act.sub`:

    *  The AS MAY use that client identity as an additional authorization input.
    *  The AS SHOULD NOT infer that the client is authorized to act on behalf of the subject solely because the client initiated the request.  Such inference is outside the interoperable behavior defined by this profile.
    *  When local policy maps the client identity to an actor identifier expected to match `act.sub`, the AS SHOULD perform identifier reconciliation before issuing a token.  If reconciliation cannot be established, the AS treats the identifiers as distinct and rejects the request when issuance requires them to identify the same entity, as defined for Identifier Reconciliation in [Conventions and Definitions](#conventions).

8.  If the AS accepts the assertion, it MUST propagate the actor information into the issued token according to the rules for the output token type being issued.  For JWT access tokens, see [JWT Access Token Output](#jwt-access-token-propagation).  For Transaction Tokens, see [Transaction Token Output Rules](#transaction-token-output-rules).  When the output is another JWT assertion grant profile, the resulting assertion MUST preserve the validated actor information subject to local policy and the chain-depth limit in [Delegation Chains](#delegation-chains).

9.  When constructing a new outermost `act` object using [Extend Chain with New Actor](#extend-chain-with-new-actor), the AS includes `sub_profile` in that object when it can authoritatively classify the actor's entity type, as [Actor Object Structure](#actor-object-structure) recommends.  The same section recommends a top-level `sub_profile` in the issued token when the AS can authoritatively classify `sub`.  Preserved inner `act` objects are immutable under [Preserve Inbound Chain](#preserve-inbound-chain).

# JWT Access Tokens {#jwt-access-tokens}

This section defines the actor-profile structure of delegated JWT access tokens used by this document.  Processing rules that lead to issuance of such tokens are defined in [Token Exchange Processing](#token-exchange-processing) and [Transaction Token Service Processing](#transaction-token-service).

## Structure {#jwt-access-tokens-structure}

A delegated JWT access token is a JWT access token per {{RFC9068}} that carries an `act` claim conforming to the actor profile defined in [Actor Profile for Delegation](#actor-profile).  Claims not listed here follow {{RFC9068}} and any other applicable token profile.

The following claims are defined for a JWT access token that carries actor-profile delegation:

`iss` (REQUIRED):
: Identifies the access token issuer.

`sub` (REQUIRED):
: Identifies the principal on whose behalf the access token is issued.

`sub_profile` (RECOMMENDED):
: Classifies the entity type of `sub`.  MUST conform to the values defined in [Actor Profile for Delegation](#actor-profile).

`act` (REQUIRED when the token represents delegation per [Delegation Chains](#delegation-chains)):
: The actor object identifying the entity exercising the subject's delegated rights.  MUST conform to the actor object structure defined in [Actor Profile for Delegation](#actor-profile).

`cnf` (REQUIRED when sender-constrained; otherwise OPTIONAL):
: Binds the access token to the current presenter when a sender-constraining mechanism such as DPoP or mTLS is used.

`client_id` (REQUIRED):
: Identifies the OAuth client that requested the token, per {{RFC9068}}.  It does not substitute for `act`; see [Client Identity and Delegation](#client-identity-delegation).

`azp` (OPTIONAL):
: An additional client identifier used by some deployments.  It does not substitute for `act`; see [Client Identity and Delegation](#client-identity-delegation).

If an issuer uses `azp` and `act.sub` for the same party, [Client Identity and Delegation](#client-identity-delegation) defines how they are reconciled, along with the other common rules; [Migrating from Implicit to Explicit Delegation](#migration-implicit-explicit) describes rollout.

The following example shows a JWT access token with actor profile claims:

~~~json
{
  "iss": "https://as.resource-domain.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "client_id": "travel-assistant-client-id",
  "azp": "https://agents.example.com/travel-assistant",
  "aud": "https://api.resource-domain.example",
  "jti": "xyz987",
  "exp": 1711820400,
  "iat": 1711816800,
  "scope": "travel:book",
  "sub_profile": "user",
  "cnf": {
    "jkt": "NzbLsXh8uDCcd7MNwrnNZpX0ak8ACQ"
  },
  "act": {
    "sub": "https://agents.example.com/travel-assistant",
    "iss": "https://as.enterprise.example",
    "sub_profile": "ai_agent"
  }
}
~~~

The top-level `cnf.jkt` binds this token to the actor's DPoP key.  The client and actor identifiers illustrate different namespaces; any equivalence depends on trusted local mappings.

## Delegated Token Issuance {#delegated-token-issuance}

When an AS issues a JWT access token outside Token Exchange whose delegation rests on an independent delegation basis ([Delegation Chains](#delegation-chains)), it MUST establish that basis for the (`sub`, actor) relationship before including `act`.  Examples include a pre-registered delegation grant, an explicit consent record, or a policy rule covering the acting party or a class of acting parties.

A client registration MAY supply that basis only if it uniquely identifies one acting entity and the AS can derive the actor identifier from the registration alone.  A registration shared by several actors does not satisfy this condition.

For the authorization code grant, the AS MAY include `act` when an independent delegation basis, such as authorization state, registration, consent, or local policy, establishes that the OAuth client is acting as a distinct actor for the resource owner.  The actor identity MUST derive from that delegation basis.  Without it, the AS MUST NOT include `act`.

This document defines no actor-selection or actor-proof parameter for the authorization code grant.  Actor determination on these paths is deployment-specific; this profile governs the issued token and its processing.

# Token Exchange Processing {#token-exchange-processing}

This section defines input processing for {{RFC8693}} Token Exchange and issuance of JWT assertion grants and JWT access tokens.  [Transaction Token Service Processing](#transaction-token-service) defines Transaction Token issuance.  [Error Responses](#actor-profile-error-responses) applies to all paths in this section.

This profile defines three JWT-based `actor_token` credential types.  JWT access tokens use `actor_token_type=urn:ietf:params:oauth:token-type:access_token` ([JWT Access Token as actor_token](#jwt-access-token-as-actor-token)).  RFC 7523 client assertions ([JWT Client Assertion](#jwt-client-assertion-as-actor-token)) and workload identity credentials ([Workload Credential Processing](#workload-identity-as-actor-token)) both use `actor_token_type=urn:ietf:params:oauth:token-type:jwt` and are distinguished as follows.

For JWT `actor_token` inputs, the AS identifies the credential profile as follows:

*  A JWT also presented as `client_assertion` with type `urn:ietf:params:oauth:client-assertion-type:jwt-bearer` is a client assertion when its `sub` equals the authenticating client's `client_id`.
*  If `sub` differs from `client_id`, the AS MUST NOT classify the JWT as a client assertion solely because it appears in `client_assertion`.  It MUST apply workload credential processing if that profile matches, or reject with `invalid_grant`.
*  If exactly one supported actor-credential profile cannot be identified, the AS MUST reject with `invalid_grant`.
*  If `client_assertion` and `actor_token` are different JWTs, the AS MUST process each independently.  These disambiguation rules apply only to `actor_token`.

## Presenter Transition Model {#token-exchange-presenter-model}

For PoP migration, this profile distinguishes two semantic classes of `subject_token` input based on whether they carry inbound `act` state and presenter-continuity information:

| Input type | Carries inbound `act` state | Carries top-level `cnf` | Presenter continuity |
|---|---|---|---|
| ID token | No | No | Not available |
| Refresh token | No | No | Not available |
| JWT assertion grant | Yes (if present) | Yes (if present) | Available |
| JWT access token | Yes (if present) | Yes (if present) | Available |
| Transaction Token | Yes (if present) | Yes (if present) | Available |

Identity-only inputs (ID tokens, refresh tokens) establish `sub` and MAY establish supporting subject state such as `sub_profile` or an authorization ceiling.  They do not establish inbound `act` state or presenter continuity, and do not by themselves justify carrying `act` into the issued token.

Token-state inputs (JWT assertion grants, JWT access tokens, Transaction Tokens) establish `sub` and MAY establish `sub_profile`, inbound `act` chain state, and current-presenter binding through top-level `cnf`.  They are the only `subject_token` inputs from which this document defines interoperable delegation-chain preservation and presenter continuation.

Token Exchange under this profile runs in exactly one of two presenter-transition modes:

*  **Presenter continuation**: no validated `actor_token` establishing a new presenter is supplied.  The issued token keeps the presenter of a PoP-capable token-state `subject_token`.
*  **Presenter rebind**: a validated `actor_token` establishes a new presenter for the issued token.  When the output token is sender-constrained, its top-level `cnf` is bound to that new presenter.

Presenter rebind requires a **direct presenter credential**: an `actor_token` whose top-level `sub` names the new presenter.  The request proves possession as required by that credential profile when establishing a sender-constrained output.  Other means of installing a presenter are deployment-specific.

Bearer and identity-only inputs cannot support continuation.  To upgrade them to sender-constrained tokens, present the existing credential as `subject_token` and a direct presenter credential as `actor_token`.  To preserve a delegation chain while changing presenters, deployments SHOULD likewise present the delegated credential as `subject_token` and a separate direct credential as `actor_token`.

JWT assertion grants are not suitable for use as `actor_token` in Token Exchange.  Their `sub` identifies the subject of delegation rather than the acting party.  Requests that need to establish an agent, workload, or client as the actor SHOULD use one of the actor credential types defined in this section instead.

## Subject Tokens

The following sections group inputs by the classes in [Presenter Transition Model](#token-exchange-presenter-model).  SAML assertions are outside this profile's scope.

### Token-State Subject Tokens

JWT assertion grants, JWT access tokens, and Transaction Tokens are token-state `subject_token` inputs.  For these inputs, this document defines the following common model:

*  the validated token establishes the inbound `sub`;
*  `sub_profile`, if present and trusted, becomes inbound supporting subject state;
*  `act`, if present, becomes inbound delegation-chain state for [JWT Access Token Output](#jwt-access-token-propagation) or [Transaction Token Output Rules](#transaction-token-output-rules);
*  top-level `cnf`, if present, makes the input eligible for presenter continuation under [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation);
*  if top-level `cnf` is absent, the token can still be used in presenter-rebind mode for bearer-to-PoP upgrade.

#### JWT Assertion Grant {#jwt-assertion-grant-as-subject-token}

When a Token Exchange request ({{RFC8693}}) presents a JWT assertion grant as the `subject_token`, the AS MUST apply the inbound validation rules of [Authorization Grant Processing](#jwt-assertion-grants-processing) to validate the inbound token.  Assertion-grant output construction from that section does not apply; propagation and scope reduction are governed by the rules below and by [JWT Access Token Output](#jwt-access-token-propagation).

Apply the continuation or rebind rules in [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation).

After any scope reduction under local policy, the AS MUST apply [JWT Access Token Output](#jwt-access-token-propagation).

#### JWT Access Token {#jwt-access-token-as-subject-token}

When a Token Exchange request ({{RFC8693}}) presents a JWT access token as the `subject_token` (`subject_token_type=urn:ietf:params:oauth:token-type:access_token`), the AS MUST apply the following steps.  Use of an opaque access token as the `subject_token` is outside the interoperable scope of this profile (see [Profile Scope](#profile-scope)).

1.  The AS MUST validate the inbound JWT access token per {{RFC9068}}: signature, `iss`, `sub`, `exp`, `nbf`, and `jti`.  Because a JWT access token used as `subject_token` was issued for a resource server, its `aud` will not ordinarily include the Token Exchange AS's token endpoint; the AS MUST NOT reject the inbound token solely because its `aud` does not include the AS's token endpoint URI.

2.  The AS MUST verify that the inbound token's `iss` is trusted under local policy to assert the delegation chain it carries.  If not, the AS MUST reject the request with `invalid_grant`.

3.  The AS MUST apply the continuation or rebind rules in [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation).  Without top-level `cnf`, the input can still be used for presenter rebind.

4.  The AS MUST extract `sub`, `sub_profile` (if present), and `act` (if present) from the validated token as the inbound delegation state for [JWT Access Token Output](#jwt-access-token-propagation).

5.  The AS can reduce scope under local policy.  The effective scope of the issued token MUST NOT exceed the inbound token's effective scope.

After completing these steps, the AS MUST apply the propagation rules in [JWT Access Token Output](#jwt-access-token-propagation).

#### Transaction Token {#txn-token-as-subject-token}

When a Token Exchange request ({{RFC8693}}) presents a Transaction Token as the `subject_token` (`subject_token_type=urn:ietf:params:oauth:token-type:txn_token`) to a regular AS (not a TTS), the AS MUST apply the following steps.

1.  The AS MUST validate the signature, `aud`, `exp`, `iat`, and issuer identity per {{I-D.ietf-oauth-transaction-tokens}}:

    *  With `act`, top-level `iss` MUST be present and the AS MUST validate it as the token issuer.  If `iss` is missing, the AS MUST reject the request with `invalid_request`.
    *  With neither `act` nor `iss`, the AS MUST determine the issuer through the Transaction Token trust-domain rules and local configuration.
    *  If validation fails or the issuer cannot be established, the AS MUST reject with `invalid_grant`.

2.  The AS MUST verify that the Transaction Token issuer identified in step 1 is trusted under local policy.  If not, the AS MUST reject the request with `invalid_grant`.

3.  The AS MUST apply [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation), using the mechanism defined by {{I-D.ietf-oauth-transaction-tokens}} and the deployment profile.  Without a top-level presenter binding, the token can still be used for presenter rebind.

4.  The AS MUST extract `sub`, `sub_profile` (if present), and `act` (if present) from the validated Transaction Token as the inbound delegation state for [JWT Access Token Output](#jwt-access-token-propagation).

5.  The `req_wl` claim identifies the workload that requested the Transaction Token from the TTS and provides informational context about the transaction origin.  The AS MAY use `req_wl` for audit or local policy decisions but MUST NOT carry it forward into the issued JWT access token.

6.  The AS MUST apply the propagation rules in [JWT Access Token Output](#jwt-access-token-propagation).  For a Transaction Token used as `subject_token`, this document defines only the actor-profile consequences of the inbound `sub`, `act`, `req_wl`, and presenter-binding state.  Transaction Token field semantics and any transaction-specific scope handling remain defined by {{I-D.ietf-oauth-transaction-tokens}}, {{RFC8693}}, and local policy.

### Identity-Only Subject Tokens

ID tokens and refresh tokens are identity-only `subject_token` inputs.  For these inputs, this document defines the following common model:

*  the validated input establishes `sub`;
*  `sub_profile`, if available from the validated input or trusted state, becomes supporting subject state;
*  the input does not establish inbound `act` chain state for propagation;
*  the input does not establish presenter continuity;
*  any `act` in the issued token therefore comes from a validated `actor_token` or another independent delegation basis under local policy;
*  any sender-constrained issued token is therefore issued in presenter-rebind mode, with the new presenter established in the current exchange.

#### OpenID Connect ID Token {#id-tokens}

##### Overview {#id-token-overview}

An OpenID Connect ID token {{OpenID.Core}} identifies an authenticated user in `sub` and its relying party in `aud` (and possibly `azp`).  Under this profile, it establishes subject identity only.  The acting party comes from `actor_token`; `aud` and `azp` remain client identifiers.

##### Processing {#id-token-as-subject-token}

When a Token Exchange request ({{RFC8693}}) presents an ID token as the `subject_token` (`subject_token_type=urn:ietf:params:oauth:token-type:id_token`), the AS MUST apply the following steps.

1.  The AS MUST validate the ID token per {{OpenID.Core}} and local policy before using it as actor-profile input.  Any checks on `aud` or `azp` remain OpenID Connect and client-identity checks; they do not by themselves establish the delegated actor under this profile.

2.  The AS MUST use the validated ID token's `sub` as the subject identity for the issued token, subject to the same-subject preservation rule in [JWT Access Token Output](#jwt-access-token-propagation).

3.  The AS SHOULD set `sub_profile` to `user` in the issued token if it can authoritatively classify the ID token's `sub` as a human user identity and no conflicting subject classification is available under local policy.

4.  The ID token is an identity-only `subject_token` for [Presenter Transition Model](#token-exchange-presenter-model).  It does not establish actor identity or presenter continuity.  If an `actor_token` is present, the AS processes it per its type-specific rules and derives `act.sub` from it as specified in [Actor Tokens](#actor-tokens).  If the issued token is sender-constrained, that `actor_token` also establishes the new presenter for presenter-rebind mode.  If no `actor_token` or independent delegation basis is present, the AS MUST NOT include `act` in the issued token.

5.  The AS MUST apply the propagation rules in [JWT Access Token Output](#jwt-access-token-propagation) to determine the remaining claims in the issued token.  Because an ID token carries no inbound `act` chain and no OAuth scope ceiling, delegation-chain construction and scope determination come from the `actor_token` (if any), {{RFC8693}}, and local policy rather than from the ID token itself.

#### Refresh Token {#refresh-tokens}

##### Overview {#refresh-token-overview}

A refresh token authorizes a client to obtain new access tokens.  For this profile, the AS obtains its subject, scope, and authorization state from trusted server state, rather than extracting actor claims from the token.

A refresh token MAY be used as `subject_token` when the AS can validate its state directly or through a trusted back-channel to its issuer.  It MUST NOT be treated as a portable cross-domain delegation artifact or used as `actor_token`.  The actor comes from a separate `actor_token` or an independent delegation basis, as required by step 4 of [Refresh Token Processing](#refresh-token-as-subject-token).

Client binding, cross-client presentation, and cross-AS acceptance policies remain deployment-specific.  Cross-AS presentation without trusted validation is outside this profile's scope.

##### Processing {#refresh-token-as-subject-token}

When a Token Exchange request ({{RFC8693}}) presents a refresh token as the `subject_token` (`subject_token_type=urn:ietf:params:oauth:token-type:refresh_token`), the AS MUST apply the following steps.

1.  The AS MUST validate the refresh token through its token store or a trusted back-channel to its issuer.  For a token issued by another AS, the AS MUST NOT accept it unless that back-channel provides the subject, client binding, authorized scope, and revocation state.  Signature validation alone is insufficient.  If the token is expired, revoked, or otherwise invalid, the AS MUST reject with `invalid_grant`.

2.  The AS MUST verify that the requesting client or authenticated presenter is authorized to use the refresh token under the refresh token's client-binding, sender-constraint, rotation, and cross-client presentation policy.  If the requester is not authorized to use the refresh token, the AS MUST reject the request with `invalid_grant`.

3.  The AS MUST extract the subject identity and authorized scope associated with the refresh token from its token store or other trusted refresh-token state.  The `sub` of the user associated with the refresh token becomes `sub` in the issued token.  Top-level `sub_profile` follows [Actor Object Structure](#actor-object-structure), which recommends it when the AS can authoritatively classify the subject entity type.

4.  The AS MUST establish the actor from `actor_token` or an independent delegation basis; otherwise, it MUST omit `act`.  If `actor_token` is present, the AS processes it under its type-specific rules and derives the outermost actor from it as specified in [Actor Tokens](#actor-tokens).  That credential also establishes the new presenter for a sender-constrained output.  The refresh token supplies no actor identity or presenter continuity.

5.  The effective scope of the issued token MUST be a subset of the scope authorized by the refresh token.  The AS can further reduce scope under local policy.

After completing these checks, the AS MUST apply the propagation rules in [JWT Access Token Output](#jwt-access-token-propagation) to determine the remaining claims in the issued token.

This document does not standardize whether an AS issues refresh tokens in response to delegated JWT assertion grant requests, how such refresh tokens are revoked, or whether later use of such refresh tokens requires re-presentation of upstream delegation artifacts.  Those decisions remain deployment-specific.

## Actor Tokens {#actor-tokens}

The following rules apply to every `actor_token` type in this section:

1.  The credential MUST identify the acting party in its top-level `sub`.  If it carries `act`, the AS MUST reject with `invalid_grant`.
2.  After validating the credential, the AS MUST use its `sub` as the new outermost `act.sub` and set `act.iss` to that identifier's issuer or namespace context.
3.  If `subject_token` carries a chain, the new actor takes precedence over its outermost actor.  Different identities are permitted for presenter rebind.  Local policy MAY require equivalence on paths that only confirm an existing actor; when such a restriction applies and no trusted mapping establishes equivalence, the AS MUST reject with `invalid_grant`.

[Delegation Chain Validation and Construction](#delegation-chain-algorithm) governs nesting of the `subject_token` chain.  Because `actor_token` cannot carry `act`, it contributes no prior chain to merge.

Deployments supporting sub-delegation SHOULD provision each potential presenter with a direct credential naming itself in `sub`.

### JWT Client Assertion {#jwt-client-assertion-as-actor-token}

#### Overview

A JWT client assertion per {{RFC7523}} may be presented as `actor_token` (`actor_token_type=urn:ietf:params:oauth:token-type:jwt`) to establish an OAuth client's own identity as the acting party.  Per {{RFC7523}}, the assertion has `iss = sub = client_id` and is signed with the client's private key.  Under [Presenter Transition Model](#token-exchange-presenter-model), it is a direct presenter credential.  Two usage patterns arise:

*  The same JWT is presented as both `client_assertion` and `actor_token` in a single request, making the authenticated client identity explicit in the issued token's `act` chain.
*  The client authenticates by another method (e.g., `client_secret`, mTLS) and presents a separate JWT client assertion as `actor_token` to name that same client as the actor.

To establish a principal distinct from the OAuth `client_id` as the actor, the request MUST use a different actor credential type, such as a workload identity credential ([Workload Credential Processing](#workload-identity-as-actor-token)), whose `sub` names that distinct principal.  A client assertion conforming to {{RFC7523}} cannot name a subordinate identity.

#### Processing

When a Token Exchange request includes an `actor_token` that is a JWT client assertion, the AS MUST apply the following steps.

1.  The AS MUST validate the `actor_token` per {{RFC7523}}.  If the same JWT is also used as `client_assertion` for client authentication in the same request and this shared validation fails, the AS MUST reject the request with `invalid_client`; otherwise, the AS MUST reject the request with `invalid_grant`.

2.  The AS MUST verify that the `actor_token`'s `iss` is a client registered with the AS, that `sub` equals that client's `client_id`, and that local policy permits that client's assertion to be used as an actor credential.  If not, the AS MUST reject the request with `invalid_grant`.

3.  The AS MUST derive the outermost actor and handle any inbound chain as specified in [Actor Tokens](#actor-tokens).

4.  When the `actor_token` is the same JWT presented as `client_assertion` for client authentication in the same request, the AS MAY derive the actor identity from the already-authenticated client context rather than re-validating the `actor_token` separately, provided the result is an identical `act.sub` value.  Actor-profile-specific policy failures that occur after successful client authentication are still `invalid_grant`, not `invalid_client`.

5.  When the request uses this client assertion to establish a sender-constrained output token in presenter-rebind mode, the AS MUST validate any proof required by the selected proof mechanism for the new presenter per [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation).


### Workload Identity Credential {#workload-identity}

#### Overview {#workload-identity-overview}

A workload identity credential is a JWT whose `sub` identifies a software workload, such as a service or agent.  It is a direct presenter credential under [Presenter Transition Model](#token-exchange-presenter-model).  WIMSE credentials are defined in {{I-D.ietf-wimse-workload-creds}}; [Token Exchange Processing](#token-exchange-processing) describes profile disambiguation.

The recommended pattern for agentic Token Exchange is:

*  `subject_token`: a JWT access token or JWT assertion grant carrying the user's `sub` and the delegation chain (`act`)
*  `actor_token`: a workload identity credential whose `sub` is the agent or service identity
*  Output: a JWT access token with the user as `sub` and the workload as the outermost `act.sub`

#### Processing {#workload-identity-as-actor-token}

When a Token Exchange request ({{RFC8693}}) includes an `actor_token` that is a workload identity credential, the AS MUST apply the following steps.

1.  The AS MUST validate the workload identity credential per its type specification.  For WIMSE workload identity credentials ({{I-D.ietf-wimse-workload-creds}}), validation follows the rules defined in that specification.  If validation fails, the AS MUST reject the request with `invalid_grant`.

2.  The AS MUST verify that the workload credential's issuer (`iss`) is trusted under local policy to assert the workload's identity.  If not, the AS MUST reject the request with `invalid_grant`.

3.  The AS MUST derive the outermost actor and handle any inbound chain as specified in [Actor Tokens](#actor-tokens).

4.  When the request uses this workload credential to establish a sender-constrained output token in presenter-rebind mode, the AS MUST validate the proof required by the workload-credential profile: a WIMSE Workload Proof Token (WPT, {{I-D.ietf-wimse-wpt}}) per its specification, or a DPoP proof ({{RFC9449}}) over the token endpoint URI when the credential carries `cnf.jkt`.  If the required proof is absent or invalid, the AS MUST reject the request with `invalid_grant`.


### JWT Access Token {#jwt-access-token-as-actor-token}

#### Overview

A non-delegated JWT access token may be presented as `actor_token` to establish a service or workload as the acting party; its top-level `sub` identifies the acting party and satisfies the direct-presenter-credential requirement in [Presenter Transition Model](#token-exchange-presenter-model).  A delegated JWT access token (one carrying `act`) does not satisfy that requirement; see [Actor Tokens](#actor-tokens) for the sub-delegation pattern.

#### Processing

When a Token Exchange request includes an `actor_token` that is a JWT access token (`actor_token_type=urn:ietf:params:oauth:token-type:access_token`), the AS MUST apply the following steps.  Use of an opaque access token as the `actor_token` is outside the interoperable scope of this profile (see [Profile Scope](#profile-scope)).

1.  The AS MUST validate the `actor_token` per {{RFC9068}}.  If validation fails, the AS MUST reject the request with `invalid_grant`.

2.  The AS MUST verify that the `actor_token`'s `iss` is trusted under local policy to assert the acting party's identity in the top-level `sub`.  If trust cannot be established, the AS MUST reject the request with `invalid_grant`.

3.  The AS MUST derive the outermost actor and handle any inbound chain as specified in [Actor Tokens](#actor-tokens), including rejecting an `actor_token` that carries `act`.

4.  When establishing a sender-constrained output in presenter-rebind mode and the credential carries top-level `cnf`, the AS MUST validate proof for that binding per [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation).

## `may_act` {#may-act}

The `may_act` claim ({{RFC8693, Section 4.4}}) pre-authorizes a specific party to act on behalf of the subject in a subsequent Token Exchange.  This document defines limited use of `may_act` as a delegation-authorization input when actor identity is established by other means: when present in a validated `subject_token`, it MAY satisfy the delegation-authorization check in step 3 of [Validate Outermost Actor](#validate-outermost-actor) without a separately pre-registered grant.  In no case is `may_act` itself the source of actor identity, and it MUST NOT be propagated into any output token.

Two pre-conditions apply regardless of how the Token Exchange request is structured:

1.  The `subject_token` issuer is trusted under local policy to assert `may_act` on behalf of the subject.
2.  The canonical `may_act` identifier matches the derived actor identity under Identifier Reconciliation ([Conventions and Definitions](#conventions)).

The canonical `may_act` identifier is (`may_act.iss`, `may_act.sub`) when `may_act` carries `iss`, and (`subject_token.iss`, `may_act.sub`) otherwise.  The AS MUST apply configured mapping rules and MUST NOT infer equivalence from naming similarity alone.

If `may_act.iss` is present but is not a valid StringOrURI, the AS MUST NOT use that `may_act` claim to authorize delegation and MUST NOT fall back to `subject_token.iss`.

Actor identity is established as follows:

*  **With `actor_token`**: derive (`act.iss`, `act.sub`) under the credential's type-specific rules and reconcile it with the canonical `may_act` identifier.  `may_act` MUST NOT override the derived actor.
*  **Without `actor_token`**: the requesting client MUST be a confidential client that has authenticated in the request; public clients MUST NOT use this path.  Reconcile the authenticated client with the canonical `may_act` identifier and set `act.sub` to the client's canonical identifier.  The AS MUST set `act.iss` to the issuer or namespace context that locally registered the client, typically the AS's own issuer URI.

The second path supports a token that pre-authorizes a particular client to present it without a separate actor credential.

When `may_act` is absent or the conditions above are not met, the AS MUST satisfy the delegation-authorization check through another recognized basis (pre-registered grant, consent record, or applicable policy rule).  The AS MUST NOT treat the mere presence of `may_act` as authorization for any actor other than the one whose canonical identity matches it.

## Output Token Rules

### JWT Assertion Grant Output {#jwt-assertion-grant-issuance}

When `requested_token_type` requests a JWT assertion grant, the output MUST satisfy [JWT Assertion Grant Structure](#jwt-assertion-grants-structure).  Supported type identifiers include `urn:ietf:params:oauth:token-type:jwt` and compatible profiles of that type, such as `urn:ietf:params:oauth:token-type:id-jag` {{I-D.ietf-oauth-identity-assertion-authz-grant}}.

The AS MUST:

*  Construct the chain per [JWT Access Token Output](#jwt-access-token-propagation).
*  Set `aud` to the downstream token endpoint, from `resource` or deployment configuration.
*  Sign the assertion, per {{RFC7523, Section 3}}.

Issuing such a grant is subject to AS configuration and to [Validate Outermost Actor](#validate-outermost-actor).

### JWT Access Token Output {#jwt-access-token-propagation}

The issued token MUST satisfy [JWT Access Token Structure](#jwt-access-tokens-structure).  After the applicable grant, subject-token, actor-token, or TTS input processing, the AS MUST apply the rules below.

For a sender-constrained output, the AS MUST set top-level `cnf` according to [Presenter Transition Model](#token-exchange-presenter-model): retain the presenter's binding in continuation mode, or bind to the validated `actor_token` presenter in rebind mode.  The latter also supports bearer-to-PoP upgrades.

If a Token Exchange request explicitly seeks a delegated output, for example by supplying an `actor_token` or by presenting a `subject_token` that already carries `act`, and the AS cannot validate the actor information, it MUST reject the request with `invalid_grant`.  If the AS can validate the actor information but cannot establish or confirm the required delegation basis, or if local policy prohibits the relationship, it MUST reject the request with `actor_unauthorized`.  The AS MUST NOT issue a non-delegated JWT access token in place of the requested delegated output.

1.  The AS includes or omits `act` as required by [Delegation Chains](#delegation-chains), and does not silently drop inbound actor information ([Omit `act`](#omit-act)).

2.  The AS MUST preserve `sub` to refer to the same underlying subject as the inbound token.  If the AS uses a different subject-identifier namespace, it MAY change the `sub` value only to re-express that same subject in the new namespace under a trusted local mapping.  The AS MUST NOT replace `sub` with an identifier for a different subject.  Subject-namespace translation requirements and relying-party consequences are described in [Subject Namespace Translation](#subject-namespace-translation).

3.  The AS MUST construct the `act` claim using the construction decision order in [Delegation Chain Validation and Construction](#delegation-chain-algorithm): extend with a new actor, preserve an existing chain, or omit `act`, in that order.  Inherited actors are not rewritten, as [Extend Chain with New Actor](#extend-chain-with-new-actor) and [Preserve Inbound Chain](#preserve-inbound-chain) require.  An actor derived from `actor_token` is asserted by the issuing AS; consumers MUST NOT infer that it was present in the `subject_token` or endorsed by its issuer.

4.  The AS MUST reject if actor validation fails or the resulting chain exceeds the depth limit.  It MUST use `invalid_request` for excessive depth or an inbound actor missing `act.sub` or `act.iss`, and `invalid_grant` for validation failures.  It MUST NOT issue a partially preserved chain.

5.  Top-level `sub_profile` follows [Actor Object Structure](#actor-object-structure), which recommends it when the AS can authoritatively classify the token's `sub` entity type.

6.  The AS can reduce scope under local policy.  If this reduction, before any actor-based restriction, leaves no effective scope, it MUST reject with `invalid_scope`.

    If the AS also restricts scope using the (`sub`, `act.sub`) pair or `act.sub_profile`, it MUST return the final effective `scope` in the token response.  If this restriction leaves no scope, the AS MUST reject:

    *  with `actor_unauthorized` when the actor is categorically unauthorized for the remaining scope, for example because its entity type is prohibited;
    *  with `invalid_scope` for other causes, such as an actor scope ceiling that excludes the requested values.

7.  The AS MAY preserve inbound client identifiers per the output token profile or local policy.  Preserved values MUST retain their client-identity meaning and MUST NOT represent delegation state.  If preserving an optional identifier would create ambiguity about the delegated actor relationship, the AS SHOULD omit it.  JWT access tokens still require `client_id` per {{RFC9068}}; see [Client Identity and Delegation](#client-identity-delegation).

8.  The AS MUST honor resource-indicator constraints ({{RFC8707}}) in delegated token requests.

# Transaction Token Service Processing {#transaction-token-service}

This section defines the actor-profile claim structure for Transaction Tokens and the rules a Transaction Token Service (TTS) applies when it validates supported `subject_token` inputs, authenticates the new presenter, and issues a delegated Transaction Token.

TTS error handling for requests processed in this section is defined in [Error Responses](#actor-profile-error-responses).

## Transaction Tokens {#transaction-tokens}

Transaction Tokens {{I-D.ietf-oauth-transaction-tokens}} are short-lived JWTs that capture the workload identity and request context for a series of related service calls within a single business transaction. They are issued by a Transaction Token Service (TTS), which is a specialized authorization server.

Transaction Token claims are defined in {{I-D.ietf-oauth-transaction-tokens}}.  This profile modifies or adds the following claims:

`iss` (OPTIONAL in {{I-D.ietf-oauth-transaction-tokens}}; REQUIRED by this profile when carrying `act`):
: Identifies the Transaction Token issuer.  It MUST be present with `act` and SHOULD be present when crossing trust domains.  It MAY be omitted only without `act`, within a single Trust Domain, and when all recipients know the issuer out of band.  In that case, recipients MUST identify the issuer using {{I-D.ietf-oauth-transaction-tokens}} and local configuration.

`req_wl`:
: This claim provides TTS-level workload context and is not a substitute for `act.sub`; see [Actor Claim in Transaction Tokens](#actor-claim-in-transaction-tokens).

`act` (REQUIRED when the token represents delegation per [Delegation Chains](#delegation-chains); omitted otherwise):
: Represents the current acting party and any prior delegation steps, conforming to [Actor Object Structure](#actor-object-structure).  See [Actor Claim in Transaction Tokens](#actor-claim-in-transaction-tokens) for delegation semantics and the relationship between `act.sub` and `req_wl`.

### Actor Claim in Transaction Tokens {#actor-claim-in-transaction-tokens}

For this profile, a Transaction Token represents delegation when a condition in [Delegation Chains](#delegation-chains) holds, typically because an `actor_token` or an inbound `act` chain establishes that the workload acts for `sub`.  It then carries `act`, as [Delegation Chains](#delegation-chains) requires, and top-level `iss`, as [Transaction Tokens](#transaction-tokens) requires.  When no such condition holds, including for a workload acting under its own grant without any delegation basis, `act` is omitted.  The TTS MUST NOT infer delegation solely because `sub` and `req_wl` differ.

`req_wl` identifies the workload that requested the token from the TTS.  `act.sub` identifies the immediate acting party in the subject identifier namespace used by this profile.  The authoritative actor identifier for authorization decisions under this document is the outermost `act.sub`; `req_wl` is supporting workload context.

Claim semantics under this profile:

*  `sub`: identifies the original initiator.  When a Transaction Token is exchanged for a replacement, the new token continues to refer to the same underlying subject, and the issuer can change `sub` only to re-express that subject in another identifier namespace under a trusted local mapping, as step 2 of [JWT Access Token Output](#jwt-access-token-propagation) requires.
*  `act.sub` (outermost): identifies the immediate acting party.  When a TTS sets both `req_wl` and the new outermost `act.sub` in a single token issuance (presenter-rebind mode), it MUST ensure they identify the same entity under local policy.  When a TTS preserves `req_wl` from an inbound token, the TTS SHOULD perform identifier reconciliation between `req_wl` and the outermost `act.sub`.  A recipient that relies on both to identify the current presenter requires them to identify the same entity, so it rejects the token when it cannot reconcile them ([Conventions and Definitions](#conventions)).
*  Inner `act` objects: identify prior presenters in the delegation path.  `act.sub_profile` at each level classifies the entity type of that presenter.

The following example shows a Transaction Token after two hops:

~~~json
{
  "iss": "https://tts.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
  "scope": "inventory:check",
  "req_wl": "https://tools.example.com/booking-tool",
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
    "sub": "https://tools.example.com/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service",
    "act": {
      "sub": "https://agents.example.com/travel-assistant",
      "iss": "https://as.enterprise.example",
      "sub_profile": "ai_agent"
    }
  }
}
~~~
The booking tool is the current presenter, identified by `req_wl` and outermost `act.sub` and bound to top-level `cnf.jkt`.  The inner actor records the travel assistant's prior participation.

## Presenter Authentication and Transition

The TTS applies the same two presenter-transition modes defined in [Presenter Transition Model](#token-exchange-presenter-model), but only for token-state `subject_token` inputs:

*  **Presenter continuation**: the authenticated requester is the same current presenter as the inbound token.  This mode is available only when the inbound token carries a top-level presenter binding and the TTS validates proof for that binding under {{I-D.ietf-oauth-transaction-tokens}} and the applicable deployment profile.  When the inbound token carries `act`, the authenticated requester corresponds to the outermost (`act.iss`, `act.sub`) pair, as step 5 of [Transaction Token Output Rules](#transaction-token-output-rules) requires.  In this mode the TTS preserves the inbound `act` chain unchanged and MUST NOT add a new outermost `act`.
*  **Presenter rebind**: a validated `actor_token` direct presenter credential establishes a different current presenter for the issued Transaction Token.  In this mode the TTS creates a new outermost `act` for that presenter and nests any inbound `act` chain beneath it.

A bearer input can be upgraded to a sender-constrained Transaction Token through presenter rebind with a validated `actor_token`, as in [Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation).

## Supported Subject Tokens

This profile defines TTS processing for JWT assertion grants, JWT access tokens, and Transaction Tokens, for whichever of these inputs a TTS supports.  ID tokens and refresh tokens are outside this TTS profile.

For each accepted input, the TTS MUST apply the rules listed for it in the referenced section:

| Input | Rules applied | Section |
|-------|---------------|---------|
| JWT assertion grant | Validation, presenter continuity, and scope reduction | [JWT Assertion Grant as subject_token](#jwt-assertion-grant-as-subject-token) |
| JWT access token | Validation and extraction | [JWT Access Token as subject_token](#jwt-access-token-as-subject-token) |
| Transaction Token | Validation and extraction | [Transaction Token as subject_token](#txn-token-as-subject-token) |

The resulting subject, classification, chain, and binding state feeds [Transaction Token Output Rules](#transaction-token-output-rules) instead of JWT access token issuance.  For a JWT assertion grant, this state also includes the effective scope ceiling; for an inbound Transaction Token, it also includes `req_wl`.

## Transaction Token Output Rules {#transaction-token-output-rules}

The TTS applies [Delegation Chain Validation and Construction](#delegation-chain-algorithm) and [JWT Access Token Output](#jwt-access-token-propagation), with the Transaction Token adaptations below.

When a TTS receives a token-exchange request to issue or refresh a Transaction Token from an inbound JWT assertion grant, JWT access token, or Transaction Token that carries actor-profile claims, it MUST apply the following rules:

1.  The TTS preserves `sub` from the inbound token as required by step 2 of [JWT Access Token Output](#jwt-access-token-propagation).

    *  The TTS can re-express `sub` in a different identifier namespace only when a trusted local mapping establishes that both identifiers refer to the same underlying subject (for example, when crossing trust-domain boundaries in a federation scenario).

    > Note: Subject-namespace translation requirements and relying-party consequences are described in [Subject Namespace Translation](#subject-namespace-translation).

2.  The `req_wl` field and any Transaction Token fields other than actor-profile claims remain governed by {{I-D.ietf-oauth-transaction-tokens}} and local policy.  Under this profile, `req_wl` is supporting workload context and MUST NOT be treated as a substitute for the outermost `act.sub`.

3.  The TTS applies [Enforce Depth Limit](#enforce-depth-limit) to the `act` chain that results from step 6.

4.  The TTS validates the inbound token and establishes issuer trust ([Validate Carrier Token](#validate-carrier-token)) before preserving or extending any `act` chain.  For the outermost `act` object in the inbound chain, the TTS applies [Validate Outermost Actor](#validate-outermost-actor), treating presenter rebind as extending the chain and presenter continuation as preserving it.

    For inner `act` objects in the inbound chain:

    *  **Security-relevant use**: If local policy uses an inner entry as an input to issuance decisions, such as access control or scope decisions, [Validate Inner Actors Used for Decisions](#validate-inner-actors-used-for-decisions) applies, including its error mapping.
    *  **Prior-actor context only**: If an inner entry is preserved solely for audit purposes without driving any security decision, apply [Carry Prior-Actor Context](#carry-prior-actor-context).

5.  The TTS MUST determine whether the request is presenter continuation or presenter rebind:

    *  **Presenter continuation**: The TTS MUST authenticate the requester as the same current presenter as the inbound token.  When the inbound token carries `act`, the authenticated requester MUST correspond to the outermost (`act.iss`, `act.sub`) pair or the TTS MUST reject the request with `invalid_grant`.  If the required actor relationship is prohibited by local policy, absent, or cannot be confirmed from the current inputs and policy, the TTS MUST reject the request with `actor_unauthorized`.
    *  **Presenter rebind**: The TTS MUST validate a direct presenter `actor_token` for the new presenter.  Before creating a new outermost `act` object, the TTS MUST evaluate whether the newly authenticated presenter is authorized under local policy to act on behalf of `sub` for the requested transaction.  If the required actor relationship is prohibited by local policy, absent, or cannot be confirmed from the current inputs and policy, the TTS MUST reject the request with `actor_unauthorized`.

    If the current inputs satisfy neither presenter-continuation nor presenter-rebind requirements, the TTS MUST reject the request with `invalid_grant`.

6.  When the issued Transaction Token carries delegated actor information, it includes the top-level `iss` claim required by [Transaction Tokens](#transaction-tokens), identifying the TTS as its issuer, and the TTS MUST construct the `act` claim using [Delegation Chain Validation and Construction](#delegation-chain-algorithm).  In summary:

    *  in presenter-continuation mode, preserve the inbound chain unchanged ([Preserve Inbound Chain](#preserve-inbound-chain));
    *  in presenter-rebind mode, create a new outermost `act` object for the new presenter and nest any inbound chain beneath it ([Extend Chain with New Actor](#extend-chain-with-new-actor)).

    For a new outermost actor, the TTS sets `act.sub` to the new presenter's identifier and `act.iss` to the issuer or namespace context for that identifier, as in [Extend Chain with New Actor](#extend-chain-with-new-actor), and includes `act.sub_profile` when it can authoritatively classify the actor, as [Actor Object Structure](#actor-object-structure) recommends.  Inherited `act` objects are not rewritten, as [Extend Chain with New Actor](#extend-chain-with-new-actor) and [Preserve Inbound Chain](#preserve-inbound-chain) require.

7.  When the issued Transaction Token includes a top-level presenter-binding claim such as `cnf`, that binding applies to the current presenter.  The underlying presenter-authentication and proof mechanism is defined by {{I-D.ietf-oauth-transaction-tokens}} and any applicable deployment profile, not by this document.

8.  Transaction Token fields other than actor-profile claims, including `scope`, `tctx`, and `rctx`, are defined by {{I-D.ietf-oauth-transaction-tokens}} and local policy.  This document does not standardize their issuance semantics.

TTS-specific `may_act` processing is outside this profile.  A deployment MAY use it as an authorization hint under local policy or another specification, but it MUST NOT replace credential validation, presenter authentication, or the rules in this document that govern whether a new outermost `act` is created.

# Resource Server Processing {#resource-server-processing}

This section defines RS processing for locally validated tokens and introspection responses.

## Actor Authorization {#actor-authorization}

When a token contains both `sub` and an `act` claim, a resource server has two independent principals available for authorization policy:

*  **Subject principal** (`sub`): the party whose authorization is being exercised.  This principal typically has a relationship with the resource (e.g., an account, a role, a permission).

*  **Actor principal** (`act.sub`): the party that is making the immediate request.  This principal may be in a different organizational domain and trust level from the subject.

For Transaction Tokens, the primary policy pair remains (`sub`, `act.sub`).  The `req_wl` claim provides workload context from the TTS and is not a replacement for `act.sub`.  Nested `act` objects provide prior-actor context for audit or other deployment-specific processing; this document does not standardize their authorization use.

Actor authorization is conditional under this profile.  When an RS accepts a token as satisfying a delegated-access requirement, it MUST NOT ignore the `act` claim and authorize the request solely as if the token were non-delegated.  Whether or not it requires actor authorization, the RS SHOULD evaluate the (`sub`, outermost `act.sub`) pair according to local policy for authorization, audit, or trust decisions, and enforcing that evaluation on every request is RECOMMENDED for security-sensitive delegated access.  When local policy requires actor authorization, enforcement is mandatory: step 5 below rejects a request for which it cannot be completed.  Resource servers that receive delegated tokens should define and document their actor authorization policy.  The following steps describe one approach for resource servers that choose to enforce actor authorization policy:

1.  **Advertise delegated-token requirements**: An RS that wants to signal that delegated requests are expected to carry actor-profile information SHOULD set `actor_profile_required: true` ([Protected Resource Metadata](#protected-resource-metadata)).  An RS MAY still apply actor authorization without advertising it, but clients cannot rely on that behavior.

2.  **Evaluate subject authorization**: Determine whether `sub` has been granted the requested scope or permission, using the same mechanisms applied to non-delegated tokens.

3.  **Evaluate actor authorization**: Determine whether the (`sub`, outermost `act.sub`) pair is permitted for the requested operation.  This evaluation MAY be performed against:

    *  a registered delegation policy for the (subject, actor) pair,
    *  the actor's `sub_profile` (e.g., only AI agents from a trusted domain are permitted to act as delegatees),
    *  the token's `scope` claim.

    For Transaction Tokens, the RS SHOULD evaluate `req_wl` as supporting context.  An RS that relies on both `req_wl` and `act.sub` to identify the current presenter requires them to identify the same entity and rejects the request if it cannot reconcile them, as [Actor Claim in Transaction Tokens](#actor-claim-in-transaction-tokens) specifies.

4.  **Evaluate combined policy**: Apply resource-specific actor authorization policies (e.g., requiring both principals to have agreed to terms of service).

5.  If the RS requires actor authorization but cannot complete it, it MUST reject the request.

Use of nested actors in authorization, including ordering and failure handling, is deployment-specific.  Clients cannot assume such use without a deployment agreement.

## JWT Access Token Processing {#jwt-access-token-rs-processing}

On a request path where delegated-token processing may apply, an RS MUST validate and process JWT access tokens according to its delegated-token policy.  A conforming `act` identifies a delegated token; the RS MUST NOT infer delegation from `client_id`, `azp`, or other client-identity claims alone.  [Protected Resource Metadata](#protected-resource-metadata) advertises expectations; enforcement remains the RS's responsibility.

When the resource server evaluates a JWT access token as a delegated token under local policy, it MUST:

1.  Validate the signature, `iss`, `aud`, and temporal claims per {{RFC9068}}.  If the request path requires actor-profile conformance, including through `actor_profile_required: true`:

    *  A token evaluated as delegated MUST carry `act`, and each actor object the RS relies on MUST include `iss`.  Otherwise, reject with HTTP 401 `invalid_token`.
    *  Non-delegated tokens need not carry `act`.

2.  If the token carries a top-level `cnf.jkt`, validate the accompanying DPoP proof per {{RFC9449, Section 7}}.  If a DPoP proof is present but the token does not carry `cnf.jkt`, the RS MUST treat the token as a bearer token; the RS MUST NOT infer a confirmation binding from the DPoP proof key.

3.  Extract the `sub` and the outermost `act.sub` as the two principals relevant for authorization policy.

4.  If the token carries `client_id`, `azp`, or both, treat those as client-identity inputs only.  The actor identifier is then `act.sub`, not `client_id` or `azp`.  When local policy expects both to identify the same acting party, the RS SHOULD perform identifier reconciliation; if reconciliation cannot be established, the RS treats them as distinct and rejects the request when its authorization decision requires them to identify the same party, as defined for Identifier Reconciliation in [Conventions and Definitions](#conventions).  See [Client Identity and Delegation](#client-identity-delegation).

5.  Apply actor authorization per [Actor Authorization](#actor-authorization) when required by local policy or when the token is accepted as satisfying a delegated-access requirement for the request path.  Resource servers that do not require actor authorization still evaluate the actor, for authorization, audit, or trust decisions, as [Actor Authorization](#actor-authorization) recommends.

6.  The RS MAY traverse inner `act` objects for audit, policy refinement, or trust decisions; such use is deployment-specific.  Inner `act` objects are prior-actor context as described in [Carry Prior-Actor Context](#carry-prior-actor-context), and interoperable authorization behavior is defined around `sub` and the outermost `act.sub`.

7.  If any of the above steps fail, return an appropriate error response.  The HTTP authentication scheme used in the `WWW-Authenticate` challenge follows the token's binding mechanism: `Bearer` per {{RFC6750, Section 3.1}} for bearer or mTLS-bound ({{RFC8705}}) tokens, or `DPoP` per {{RFC9449, Section 7.1}} for DPoP-bound tokens.

    *  If signature, `iss`, `aud`, or temporal validation fails: HTTP 401 with `error="invalid_token"`.
    *  If DPoP proof validation for `cnf.jkt` fails: HTTP 401 per {{RFC9449, Section 7}}.
    *  If actor authorization required by local policy fails for a structurally valid token: HTTP 403 with `error="actor_unauthorized"`, registered in [OAuth Error Registry](#iana-error-codes).  The RS MUST NOT use `insufficient_scope` for this failure, because requesting broader scope does not resolve an actor-policy denial.
    *  The RS MUST NOT expose actor-specific rejection details outside the trust domain.

## Transaction Token Processing {#txn-token-rs-processing}

Upon receiving a Transaction Token on a request path where delegated-token processing may apply, a resource server MUST validate and process that token according to {{I-D.ietf-oauth-transaction-tokens}}, any applicable deployment profile, and the actor-profile rules in this document.

When the resource server evaluates a Transaction Token as a delegated token under local policy, it MUST:

1.  Validate the signature, audience, temporal claims, and issuer under {{I-D.ietf-oauth-transaction-tokens}} and the deployment profile:

    *  With `act`, top-level `iss` MUST be present and the RS MUST validate it as the token issuer.
    *  With neither `act` nor `iss`, the RS MUST determine the issuer through the Transaction Token trust-domain rules and local configuration.
    *  If the request path requires actor-profile conformance, including through `actor_profile_required: true`, a token evaluated as delegated MUST carry `act`, and each actor object the RS relies on MUST include `iss`.  If either is missing, reject the request.  Non-delegated tokens need not carry `act`.

2.  When the token carries a top-level presenter-binding claim such as `cnf`, validate the accompanying proof according to {{I-D.ietf-oauth-transaction-tokens}} and the applicable deployment profile.  The top-level presenter binding applies to the current presenter only.

3.  Extract `sub` and the outermost `act.sub` as the two principals relevant for authorization policy.  If `req_wl` is present, treat it as supporting workload context only.  The RS MUST NOT treat `req_wl` as a substitute for `act.sub`.  When local policy expects `req_wl` and the outermost `act.sub` to identify the same party, the RS SHOULD perform identifier reconciliation; if reconciliation cannot be established, the RS treats them as distinct and rejects the request when its authorization decision requires them to identify the same party, such as when it relies on both to identify the current presenter ([Actor Claim in Transaction Tokens](#actor-claim-in-transaction-tokens)).

4.  Apply actor authorization per [Actor Authorization](#actor-authorization) when required by local policy or when the token is accepted as satisfying a delegated-access requirement for the request path.  Resource servers that do not require actor authorization still evaluate the actor, for authorization, audit, or trust decisions, as [Actor Authorization](#actor-authorization) recommends.

5.  Optionally traverse inner `act` objects to audit the full delegation chain.  If the RS relies on inner `act` objects for audit, policy refinement, or trust decisions, it MUST do so only under the prior-actor context rules in [Carry Prior-Actor Context](#carry-prior-actor-context).

6.  If any of the above steps fail, the RS MUST reject the request according to the Transaction Token mechanism in use and local deployment profile.  Validation or presenter-proof failures are token-validation failures; actor-policy failures required by local policy are authorization failures.  The RS MUST NOT include actor-specific rejection details in error responses exposed outside the trust domain.

## Token Introspection {#token-introspection}

When token introspection ({{RFC7662}}) is used for delegated tokens, an AS MUST expose actor-profile information needed for equivalent RS processing.  For an active delegated token whose authorization context includes actor-profile claims, the introspection response MUST include:

*  `active`: `true`, per {{RFC7662, Section 2.2}}.
*  `sub`: REQUIRED.  The subject of the delegated token, as defined in {{RFC7662}}.
*  `act`: REQUIRED.  The actor object conforming to [Actor Object Structure](#actor-object-structure), including `act.sub`, `act.iss`, and any nested `act` chain, structured identically to the JWT form defined in this document.
*  `sub_profile`: REQUIRED when the token's authorization context includes a top-level `sub_profile`; otherwise SHOULD be included when the AS can authoritatively classify the subject entity type.
*  `scope`: REQUIRED.  The effective scope of the token.
*  `iss`: REQUIRED when the AS has a stable issuer identifier.
*  `chain_complete`: OPTIONAL.  When absent, the RS SHOULD treat the chain as complete unless local policy or deployment context indicates otherwise.

The AS MUST return actor claims from the token's authorization context, including the complete nested chain, except for the following privacy-filtering allowance.

If local privacy policy requires omitting inner actors, the AS MAY filter them but MUST include `"chain_complete": false`.  When the AS knows the RS uses inner actors for security decisions, it SHOULD NOT filter the chain and SHOULD instead return it in full or reject introspection.

When `chain_complete` is `false`:

*  An RS using any inner actor for authorization, scope determination, or another security decision MUST reject the request.
*  An RS using inner actors only for audit or information MAY accept the response if it records the incompleteness.  The subject and outermost actor remain available for authorization.

The RS MUST NOT treat a partial chain as complete delegation history.  Companion profiles with data aligned to `act` define their filtering behavior as required by [Companion Profiles and Extension Points](#companion-profile-extensibility).

When an AS supports delegated opaque access tokens through introspection, it MUST return the fields listed above for active delegated tokens.  Support for this compatibility path MUST NOT be inferred solely from `actor_profile_required` metadata; see [Profile Scope](#profile-scope).

An introspecting RS MUST apply the same delegated-token processing as for equivalent locally validated JWT claims, including actor authorization when required by local policy.

If policy, protected resource metadata, or token context indicates delegation or requires actor-profile conformance, a missing `act` is an inconsistency and the RS MUST reject the token.  Otherwise, the RS MAY treat an active response without `act` as non-delegated.

Introspection endpoints for delegated tokens SHOULD be advertised via the `introspection_endpoint` parameter in AS metadata ({{RFC8414}}).  When revocation is integrated, the introspection response for a revoked delegated token returns `"active": false` per {{RFC7662, Section 2.2}} and MUST NOT include `act` or `sub_profile` claims.

Resource servers that cache introspection responses for delegated tokens should use short cache lifetimes consistent with revocation requirements.  An RS using inner actors for security decisions SHOULD NOT cache a response with `"chain_complete": false`.


# Error Responses {#actor-profile-error-responses}

When an AS or TTS rejects a request under this profile for reasons related to actor-profile processing, its error response follows {{RFC6749, Section 5.2}} and {{RFC8693, Section 2.2}}.  These error codes do not override `invalid_client` when a request fails client authentication per {{RFC6749}} or {{RFC7523}}.

The following errors apply to both AS and TTS endpoints:

| Error | Condition |
|-------|-----------|
| `invalid_request` | Invalid actor structure, missing required claim, or excessive chain depth |
| `invalid_grant` | Invalid credential, untrusted issuer, failed actor validation, or failed presenter proof |
| `invalid_scope` | No effective scope remains for reasons other than categorical actor denial |
| `actor_unauthorized` | Actor policy prohibits the request, rejects the actor type, or cannot confirm the required delegation relationship |

Missing required claims include `act.sub`, `act.iss`, the binding claim of a sender-constrained JWT assertion grant, and top-level `iss` on a delegated Transaction Token.  TTS failures to preserve the subject or to trust inbound actor identifiers also use `invalid_grant`.  Mechanism-specific proof errors, such as `invalid_dpop_proof`, follow the applicable processing section.

The `error_description` field SHOULD be included and SHOULD describe which aspect of actor-profile processing failed, to the extent permitted by the server's security and privacy policy.  Some `actor_unauthorized` failures are recoverable by using a different actor credential, actor type, or delegation grant; others are definitive local-policy prohibitions.

For a token-endpoint `actor_unauthorized` response, the client SHOULD check `entity_profiles_supported.actor` and any `error_description`.  A different actor credential or grant may resolve the failure; local-policy prohibitions may have no remediation.  After an RS returns this error, the client SHOULD obtain a token through a different actor credential or grant path.

Example:

~~~json
{
  "error": "actor_unauthorized",
  "error_description": "Actor type not accepted for this scope"
}
~~~

# Metadata and Discovery {#metadata-and-discovery}

Authorization servers and resource servers advertise support through these parameters:

| Parameter | Metadata | Capability |
|-----------|----------|------------|
| `authorization_grant_profiles_supported` | Authorization server | JWT authorization-grant profiles |
| `actor_profile_token_exchange` | Authorization server | Token Exchange input and output types |
| `entity_profiles_supported.actor` | Authorization server | Accepted actor entity types |
| `actor_profile_required` | Protected resource | Resource requirements for delegated requests |

{{I-D.ietf-oauth-identity-assertion-authz-grant}} defines `authorization_grant_profiles_supported`, and {{I-D.mora-oauth-entity-profiles}} defines `entity_profiles_supported.actor`.  This document defines the other two in [Authorization Server Metadata](#authorization-server-metadata) and [Protected Resource Metadata](#protected-resource-metadata).

These signals do not guarantee acceptance of a particular request or every combination of advertised capabilities.  Additional constraints require deployment documentation or agreements.  Companion profiles can define metadata under [Companion Profiles and Extension Points](#companion-profile-extensibility).

## Authorization Server Metadata {#authorization-server-metadata}

The following parameters are defined for use in the AS metadata document ({{RFC8414}}):

`authorization_grant_profiles_supported`:
: OPTIONAL.  A JSON array defined by {{I-D.ietf-oauth-identity-assertion-authz-grant}}.  An AS that processes JWT authorization grants carrying actor-profile claims SHOULD include `urn:ietf:params:oauth:grant-profile:actor-profile`.  This advertises the processing rules for all JWT authorization-grant paths defined here, without guaranteeing acceptance of a particular request.  An AS advertising this value MUST also list `urn:ietf:params:oauth:grant-type:jwt-bearer` in `grant_types_supported`.

`actor_profile_token_exchange`:
: OPTIONAL.  A JSON object advertising coarse Token Exchange capabilities for requests in which actor-profile processing can apply.  When absent, the AS makes no claim about Token Exchange support for this actor profile.  This parameter applies only to Token Exchange; it does not describe authorization code grants or JWT bearer authorization grants.  The object members defined by this document are:

  *  `subject_token_types_supported`: OPTIONAL.  A JSON array of token-type URI strings indicating the `subject_token_type` values the AS accepts for Token Exchange requests in which actor-profile processing can apply.  Values defined by this document are:

     -  `urn:ietf:params:oauth:token-type:jwt`: JWT assertion grants ([JWT Assertion Grants](#jwt-assertion-grants))
     -  `urn:ietf:params:oauth:token-type:access_token`: JWT access tokens ([JWT Access Tokens](#jwt-access-tokens))
     -  `urn:ietf:params:oauth:token-type:id_token`: OpenID Connect ID tokens ([OpenID Connect ID Token](#id-tokens))
     -  `urn:ietf:params:oauth:token-type:refresh_token`: refresh tokens ([Refresh Token](#refresh-tokens))
     -  `urn:ietf:params:oauth:token-type:txn_token`: Transaction Tokens ([Transaction Tokens](#transaction-tokens))

  *  `actor_token_types_supported`: OPTIONAL.  A JSON array of token-type URI strings indicating the `actor_token_type` values the AS accepts for Token Exchange requests in which actor-profile processing can apply.  Values defined by this document are:

     -  `urn:ietf:params:oauth:token-type:jwt`: JWT client assertions ([JWT Client Assertion](#jwt-client-assertion-as-actor-token)) and workload identity credentials ([Workload Credential Processing](#workload-identity-as-actor-token))
     -  `urn:ietf:params:oauth:token-type:access_token`: JWT access tokens ([JWT Access Token as actor_token](#jwt-access-token-as-actor-token))

  *  `requested_token_types_supported`: OPTIONAL.  A JSON array of token-type URI strings indicating the `requested_token_type` values the AS accepts for Token Exchange requests in which actor-profile processing can apply.  Values defined by this document are:

     -  `urn:ietf:params:oauth:token-type:access_token`: JWT access tokens ([JWT Access Tokens](#jwt-access-tokens))
     -  `urn:ietf:params:oauth:token-type:jwt`: JWT assertion grants ([JWT Assertion Grants](#jwt-assertion-grants))
     -  `urn:ietf:params:oauth:token-type:txn_token`: Transaction Tokens ([Transaction Tokens](#transaction-tokens))

Advertising a type does not guarantee every input/output combination, resource, scope, binding mechanism, or JWT variant.  In particular, the `jwt` type covers both client assertions and workload credentials; [Token Exchange Processing](#token-exchange-processing) defines disambiguation.

The entity profile types the AS accepts for actors are advertised via the `entity_profiles_supported.actor` array defined in {{I-D.mora-oauth-entity-profiles}}, not via a separate metadata parameter.  DPoP support is advertised via `dpop_signing_alg_values_supported` per {{RFC9449}}.

Example AS metadata fragment:

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

One new parameter is defined for use in Protected Resource Metadata ({{RFC9728}}):

`actor_profile_required`:
: OPTIONAL.  A boolean indicating that delegated access to the resource requires actor information conforming to this profile.  When `false` or absent, metadata makes no such claim.  Non-delegated requests need not carry `act`.

  Clients SHOULD treat `true` as requiring a conforming token or an explicitly documented introspection path that provides equivalent claims for opaque tokens.

  An AS that has processed this metadata with `actor_profile_required` set to `true` MUST reject a request that would produce a nonconforming delegated token for the resource unless a supported introspection path provides equivalent actor information.  An RS enforcing this policy MUST reject a delegated request for which neither form of actor information is available.

  The parameter applies to the resource as a whole.  An RS with path-specific requirements MUST enforce them at the request layer.  It MAY advertise `true` as a conservative resource-wide signal; clients and deployment documentation SHOULD account for path-specific enforcement that metadata cannot fully express.

Clients discover which actor entity profile values the RS's AS will accept by consulting `entity_profiles_supported.actor` in the AS metadata for one of the authorization servers listed in the resource's `authorization_servers` array ({{RFC9728}}).  When `authorization_servers` lists multiple entries, the client SHOULD select the AS that issued or will issue the token being presented.

Example Protected Resource Metadata fragment:

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

Transaction Token support under this profile for Token Exchange paths is advertised through `actor_profile_token_exchange.requested_token_types_supported`.  When an AS or TTS can issue Transaction Tokens as delegated Token Exchange outputs under this profile, it MUST list `urn:ietf:params:oauth:token-type:txn_token` in `actor_profile_token_exchange.requested_token_types_supported`.  This document does not define any separate Transaction Token discovery parameter.

## Capability Signaling Usage

Clients use Protected Resource Metadata ({{RFC9728}}) to determine whether a resource advertises actor-profile conformance (`actor_profile_required`), and the associated AS metadata ({{RFC8414}}) to assess JWT authorization-grant support (`authorization_grant_profiles_supported`), Token Exchange compatibility (`actor_profile_token_exchange`), and accepted actor entity profiles (`entity_profiles_supported.actor`).  When this profile is combined with Identity Chaining ({{I-D.ietf-oauth-identity-chaining}}), clients SHOULD additionally consult `identity_chaining_requested_token_types_supported`; the two parameter sets are independent.

The metadata in this document does not advertise authorization-code actor-selection mechanisms or per-scope/per-path actor type restrictions.  Deployments that need either capability rely on deployment documentation, bilateral agreement, or a companion profile.  When a delegated request carries `act.sub_profile`, its value SHOULD be drawn from `entity_profiles_supported.actor` when that metadata is available.

Example client preflight failure: if the RS metadata advertises `"actor_profile_required": true` but the target AS metadata advertises `"entity_profiles_supported": { "actor": ["service"] }` and the client's acting entity profile is `ai_agent`, the client would ordinarily stop before making the token request because the AS does not advertise support for the actor type the client would need to represent.

# Companion Profiles and Extension Points {#companion-profile-extensibility}

This document defines current-token delegated identity while leaving room for companion profiles to define supplementary behavior such as provenance, transparency, or deployment-specific audit material.

A companion profile layered on top of this one:

*  MAY define additional top-level JWT claims, OAuth metadata parameters, or introspection response parameters that apply only when a token already conforms to this profile;
*  MUST preserve the meanings of the token's top-level `sub`, the outermost `act.sub`, the (`act.iss`, `act.sub`) actor identifier pair, the nested `act` chain ordering, and the top-level `cnf` claim for the current presenter;
*  MUST NOT reinterpret `act.iss`, nested `act` objects, or the top-level `cnf` claim as independently trusted prior-hop provenance artifacts;
*  SHOULD define any supplementary provenance, receipt, or chain-wide state in separate top-level claims or equivalent companion mechanisms rather than by overloading members inside inherited `act` objects;
*  if it defines data that aligns to the visible `act` chain, MUST specify the alignment rules, the behavior when coverage is partial, and the behavior when introspection or privacy filtering suppresses part of the visible chain.

An implementation that conforms only to this core profile MUST ignore unrecognized companion-profile claims, metadata parameters, and introspection response parameters unless another specification or local policy defines their meaning.  A deployment that requires support for a companion profile expresses that requirement through the companion profile's own metadata, through out-of-band agreement, or through another explicit local-policy mechanism.

# Deployment Considerations

This section provides deployment and migration guidance for adopting the OAuth Actor Profile in systems that currently rely on implicit delegation or older `act` semantics.

## Migration and Adoption {#migration-and-adoption}

### RFC 8693 Backwards Compatibility

An {{RFC8693}} actor object without `iss` does not conform to this profile.  Implementations MUST treat it as nonconforming and MUST NOT infer semantics for absent claims.  When local policy or advertised metadata requires profile conformance for a token or assertion, recipients MUST reject such actor objects.

When an AS receives such an object:

*  If profile conformance is required by policy or metadata, the AS MUST reject with `invalid_request`.
*  Otherwise, the AS MAY apply local rules for non-profile processing.  It MUST NOT add `iss` to an inherited actor, silently drop the inbound `act`, or carry the nonconforming chain into a profile-conforming output.  A request requiring that output MUST be rejected.

A deployment can migrate in three stages:

1.  Issuers emit `act.iss` for every actor in newly issued profile tokens, as [Actor Object Structure](#actor-object-structure) requires.  Existing consumers can ignore the additional claim.
2.  Consumers SHOULD begin validating the actor identifier context once issuers support it.
3.  Once all token issuers and consumers on a path have been updated, resources SHOULD enforce conformance through local policy and `actor_profile_required: true`.  ASes can also require conformance on updated inbound paths.

Implementations that previously treated confirmation members inside `act` as active sender-constraining mechanisms should note that this document defines proof-of-possession only through the top-level `cnf` claim and the immediate presenter.  Deployments that relied on per-hop actor-key verification for multi-hop security properties will need a separate provenance mechanism or profile rather than the core actor profile defined here.

### Migrating from Implicit to Explicit Delegation {#migration-implicit-explicit}

Deployments that infer actors from `client_id`, `azp`, or request context can migrate incrementally:

*  Clients SHOULD prefer tokens with explicit actor claims when available.
*  Issuers SHOULD emit both legacy client identifiers and actor claims during transition when feasible.
*  Without `act`, deployments MAY retain legacy client-based policy.
*  When both forms are present, apply [Client Identity and Delegation](#client-identity-delegation) and record mismatches.
*  Once an RS requires explicit delegation on a path, it does not accept a token without `act` as a substitute for a delegated token merely because legacy client-based policy permits it, as [Client Identity and Delegation](#client-identity-delegation) requires.

Deployments that require explicit delegation from the outset can omit the transition.

The legacy form carries only `client_id` (and optionally `azp`) to identify the acting party.  The explicit form adds an `act` block:

~~~json
{
  "iss": "https://as.example.com",
  "sub": "https://idp.example.com/users/alice",
  "client_id": "travel-assistant-client-id",
  "azp": "travel-assistant-client-id",
  "act": {
    "sub": "https://agents.example.com/travel-assistant",
    "iss": "https://as.example.com",
    "sub_profile": "ai_agent"
  },
  "scope": "booking:create"
}
~~~

In this example, `client_id` and `azp` remain as auxiliary client-identity inputs while `act.sub` carries the explicit actor identity.  See [Client Identity and Delegation](#client-identity-delegation) for the normative treatment of these claims when both are present.

Mismatch example, where the client and actor identify different parties:

~~~json
{
  "iss": "https://as.example.com",
  "sub": "https://idp.example.com/users/alice",
  "client_id": "travel-assistant-client-id",
  "act": {
    "sub": "https://agents.other-provider.example/concierge-bot",
    "iss": "https://as.other-provider.example",
    "sub_profile": "ai_agent"
  },
  "scope": "booking:create"
}
~~~

This example illustrates the mismatch case covered by [Client Identity and Delegation](#client-identity-delegation): the OAuth client identifier and the explicit actor identifier name different parties unless trusted local mapping rules bind them.

## Trusting Actor Identifier Pairs {#act-iss-authority-guidance}

[Validate Outermost Actor](#validate-outermost-actor) requires trust in the token issuer's authority to assert (`act.iss`, `act.sub`).  Possible deployment-specific mechanisms include:

*  federation metadata or trust-framework configuration that authorizes the token issuer to assert actor identifiers in the `act.iss` context (for example, {{OpenID.Federation}});
*  pre-registration entries that explicitly authorize the token issuer to assert a specific (`act.iss`, `act.sub`) pair or identifiers of that form; and
*  bilateral or deployment-local policy rules that authorize the token issuer to carry the specific class of actor identifier used in `act.sub`.

For HTTPS identifiers, one possible local rule is URL namespace containment: an explicitly configured rule that compares scheme, host, port, and path boundaries.  Scheme and host comparisons follow {{RFC3986, Section 3.2.2}}; paths are generally case-sensitive.  Subdomain relationships alone are often insufficient to establish trust without explicit configuration.

Examples:

*  A token issued by `https://as.enterprise.example` with `act.iss = https://as.enterprise.example` and `act.sub = https://as.enterprise.example/agents/travel-assistant` would commonly satisfy a same-host local trust rule.
*  A token issued by `https://as.enterprise.example` with `act.iss = https://as.enterprise.example` and `act.sub = https://idp.enterprise.example/users/alice` would not ordinarily satisfy URL containment alone, because the host differs.
*  For non-HTTPS identifier schemes such as workload-identity URNs, deployments typically rely on registry, federation, or other explicit local trust configuration rather than URL-based rules.

## Token Lifetime for Delegation Chains {#delegation-chain-token-lifetime}

A delegation can be revoked while downstream tokens remain usable.  Expiration bounds that exposure when recipients do not consult current revocation state; see [Delegation Revocation](#delegation-revocation).

Deployments SHOULD use shorter lifetimes for delegated tokens than for non-delegated tokens of equivalent scope.  JWT access tokens and assertion grants in multi-hop chains SHOULD last no longer than needed for the authorized task.  An upstream artifact also limits how long a downstream actor can request tokens without renewing it.

Deployments with three or more actors SHOULD account for delayed revocation across hops.  Where revocation risk is significant, for example where a user can withdraw consent at any time, deployments SHOULD combine short lifetimes with introspection at sensitive resources rather than relying solely on `exp`.  {{RFC9700}} provides general guidance.

# Conformance {#conformance}

This section enumerates per-role requirements for claiming conformance to this profile.  Profile scope (representation versus policy, supported token formats, and supported request semantics) is defined in [Profile Scope](#profile-scope).  An implementation claiming conformance MUST satisfy the requirements listed for each role it performs.

## Issuing Authorization Server {#conformance-as}

An issuing AS that claims conformance to this profile MUST:

*  emit `act.iss` in every newly issued token carrying `act` ([Actor Object Structure](#actor-object-structure));
*  apply the chain validation and construction algorithm in [Delegation Chain Validation and Construction](#delegation-chain-algorithm);
*  define and enforce a local maximum delegation depth ([Delegation Chains](#delegation-chains));
*  preserve `sub` across token issuance, translating only under a trusted local mapping ([JWT Access Token Output](#jwt-access-token-propagation));
*  apply the presenter-transition model in [Presenter Transition Model](#token-exchange-presenter-model) when performing Token Exchange;
*  reject any `actor_token` carrying `act` ([Actor Tokens](#actor-tokens));
*  return the error codes specified in [Error Responses](#actor-profile-error-responses) for the listed failure conditions;
*  preserve `act` and top-level `sub_profile` in introspection responses for active delegated tokens, when introspection is supported ([Token Introspection](#token-introspection)).

## Transaction Token Service {#conformance-tts}

A TTS that claims conformance to this profile MUST:

*  satisfy the issuing-AS requirements above for Transaction Token output;
*  set top-level `iss` on every Transaction Token carrying `act` ([Transaction Tokens](#transaction-tokens));
*  apply the TTS presenter-authentication and output rules in [Transaction Token Output Rules](#transaction-token-output-rules);
*  treat `req_wl` as supporting workload context, not as a substitute for the outermost `act.sub` ([Actor Claim in Transaction Tokens](#actor-claim-in-transaction-tokens)).

## Resource Server {#conformance-rs}

An RS that claims conformance to this profile MUST:

*  validate `act` only when the outer token issuer is trusted to convey it ([Delegation Chain Integrity and Trust](#delegation-chain-integrity));
*  treat `client_id` and `azp` as client-identity inputs only, not as actor identifiers, when `act` is present ([Client Identity and Delegation](#client-identity-delegation));
*  evaluate proof of possession against the top-level `cnf` only ([Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation));
*  use the `WWW-Authenticate` challenge scheme appropriate to the token's binding mechanism ([JWT Access Token Processing](#jwt-access-token-rs-processing)).

An RS that enforces actor authorization additionally MUST evaluate the (`sub`, outermost `act.sub`) pair per [Actor Authorization](#actor-authorization).

## Client {#conformance-client}

A client that claims conformance to this profile SHOULD treat `actor_profile_required: true` as an indication that delegated access for the resource requires `act`-carrying tokens ([Protected Resource Metadata](#protected-resource-metadata)), and adjust its grant selection or token request accordingly.

# Security Considerations

As described in [Representation and Policy](#representation-and-policy), this document does not define a trust framework for proving that an actor identifier context is authoritative for an actor identifier, proving delegation approval, or validating subject-identifier translation.  Security for those decisions depends on deployment-specific policy and external agreements.

## Delegation Chain Integrity and Trust {#delegation-chain-integrity}

An attacker who can inject or forge `act` claims can impersonate an arbitrary actor and exercise a subject's permissions without authorization.  The primary mitigation is to accept `act` claims only in tokens whose issuer is trusted to assert the delegated actor relationship.  RS implementations validate the token signature before extracting actor claims, as the applicable token specification and [Resource Server Processing](#resource-server-processing) require, and MUST verify that the token issuer is trusted to convey the claims it carries.

Because inner `act` objects are set by upstream ASes and not re-signed at each hop, the integrity of the entire delegation chain rests on the outermost token's signature.  Implementations SHOULD use short token lifetimes, and an expired token is rejected per {{RFC7519, Section 4.1.4}} regardless of chain depth.

Inner `act` objects are prior-actor context under [Carry Prior-Actor Context](#carry-prior-actor-context).  Security policies that rely on inner actor identities for access control are deployment-specific and generally lower-assurance than policies based on `sub` and the outermost `act.sub`.

When a token crosses organizational boundaries, the receiving AS or RS needs to apply appropriate trust evaluation.  ASes performing Token Exchange MUST evaluate cross-domain delegation grants explicitly and SHOULD NOT grant cross-domain actors the same rights as same-domain actors absent an explicit trust decision that makes them equivalent.

## Self-Issued Authorization Grants {#security-self-issued-grants}

This section addresses self-issued JWT *authorization grants* ([JWT Assertion Grants](#jwt-assertion-grants)); it does not apply to RFC 7523 client assertions used as `actor_token` ([JWT Client Assertion](#jwt-client-assertion-as-actor-token)), where `iss = sub = client_id` is the conformant pattern defined by {{RFC7523}}.

In a self-issued assertion grant, the acting entity is itself the JWT `iss` and directly asserts delegation to itself without any upstream AS having authenticated the actor or pre-validated the delegation relationship.  Self-issued authorization grants are outside the interoperable scope of this document and are rejected by default, as required by [JWT Assertion Grant Structure](#jwt-assertion-grants-structure).  This section specifies the security controls that a deployment needs to establish independently when another specification or local policy explicitly enables self-issued authorization grant acceptance.

Because no upstream AS vouches for the actor's identity or the delegation relationship, the receiving AS MUST NOT treat the self-asserted delegation claim alone as a sufficient authorization basis.  When a deployment enables self-issued authorization grants, the receiving AS MUST at minimum:

*  Validate the JWT signature using the key identified in the JWT header, obtained from a pre-registered or otherwise independently trusted source for the self-issuing party.
*  Verify the `exp`, `iat`, and `nbf` claims per {{RFC7519}}.
*  Reject assertions whose `jti` has already been accepted within the assertion's validity window to prevent replay.
*  Verify proof of possession per the token-endpoint mechanism in use (DPoP per {{RFC9449}} or mTLS per {{RFC8705}}).
*  Apply the actor-profile validation and proof-of-possession requirements in [Authorization Grant Processing](#jwt-assertion-grants-processing).
*  Establish the delegation relationship from an independent authorization basis such as a pre-registered grant, explicit consent record, or equivalent deployment-specific artifact.

In the absence of these controls, an attacker can self-assert an arbitrary (`sub`, `act.sub`) pair and bypass actor-profile authorization enforcement.  Deployments MUST confine self-issued authorization grants to within a single trust domain and MUST NOT propagate them across organizational boundaries.  This document defines no discovery or negotiation mechanism for self-issued authorization grant acceptance; any such mechanism is the responsibility of the enabling specification or local policy.

## Assertion Replay Prevention {#security-assertion-replay}

Replaying a delegated assertion can obtain tokens exercising the subject's authorization and establish an unauthorized delegation chain.  For grants without sender constraint, deployments maintain a `jti` replay cache for each assertion's validity window, as required by [Authorization Grant Processing](#jwt-assertion-grants-processing).  Short assertion lifetimes bound cache retention.  With DPoP or mTLS, [Authorization Grant Processing](#jwt-assertion-grants-processing) still recommends `jti` replay prevention as an additional control.

## Token Substitution

An attacker who can present a token with a crafted `sub_profile` or delegation chain could attempt to escalate privileges.  ASes MUST validate inbound `sub_profile` values against the syntax requirements of this document, the applicable registry or deployment-specific allowed set where such checks are part of local policy, and the local policy applicable to the token they are issuing.  They MUST preserve unrecognized but syntactically valid values as required by [Actor Object Structure](#actor-object-structure), and they MUST reject values that are malformed or disallowed by local policy.

## Confused Deputy

A resource server that evaluates only the subject principal when an `act` claim is present is susceptible to a confused deputy attack: a malicious actor exploits a subject's pre-existing permissions without the subject's ongoing consent simply by presenting a token that names the subject in `sub`.  The mitigation is authorization of the (`sub`, outermost `act.sub`) pair before granting access.  [Actor Authorization](#actor-authorization) defines when resource servers apply that evaluation.

## Actor-Authorization Bypass

A resource server that accepts delegated tokens but fails to enforce the (`sub`, outermost `act.sub`) relationship required by its local policy allows an attacker to bypass that policy by exploiting gaps in enforcement logic.  Resource servers that require actor authorization need to apply that evaluation on every request path where delegated access is accepted, including introspection-based paths ([Token Introspection](#token-introspection)).  Deployments that signal delegated-token requirements with `actor_profile_required: true` SHOULD ensure that the documented request paths requiring delegated access are aligned with their actual enforcement behavior so that clients do not over-read the signal.

## Client Identity and Delegation {#client-identity-delegation}

Client identity, such as `client_id`, `azp`, or authenticated client context, is widely used in deployed systems as an authorization input.  Under this document, those values remain auxiliary client-identity signals, while the outermost `act.sub` is the explicit delegated-actor signal when present.  The following normative rules apply:

*  When `act` is present, implementations MUST NOT substitute `client_id`, `azp`, or other client-identity signals for it as the delegated-actor signal.  A trusted local mapping can establish that a client identifier and `act.sub` identify the same entity without changing the meaning of either claim.
*  When a single `client_id` registration fronts multiple distinct acting entities (for example, an agent orchestration platform executing requests on behalf of different agent instances), `client_id` alone does not identify the runtime actor.  Each such request SHOULD carry `act.sub` identifying the specific acting principal.
*  During token issuance, `client_id` and `azp` MUST NOT be rewritten to represent delegation state that belongs in `act`; see [JWT Access Token Output](#jwt-access-token-propagation) for propagation rules.
*  When both explicit (`act.sub`) and implicit (`client_id`, `azp`) signals are present and local policy expects them to identify the same party, implementations SHOULD perform identifier reconciliation; if it fails, the identifiers are treated as distinct, and an operation that requires them to identify the same party is rejected, as defined for Identifier Reconciliation in [Conventions and Definitions](#conventions).
*  When a protected resource or authorization path enforces explicit delegation under this profile, implementations MUST NOT downgrade to non-`act` processing solely because another token-acquisition path or legacy policy input remains available.

The detailed migration rules and transition patterns are defined in [Migrating from Implicit to Explicit Delegation](#migration-implicit-explicit).

## `sub_profile` Trust

The `sub_profile` claim is asserted by the token issuer and is only as trustworthy as that issuer.  Resource servers MUST NOT trust `sub_profile` values in tokens issued by untrusted parties.  Resource server operators SHOULD configure a list of accepted entity-type profiles per trust domain.

## Subject Namespace Translation {#subject-namespace-translation}

An AS or TTS can translate `sub` when crossing identifier namespaces, but MUST NOT do so unless local policy establishes that both identifiers refer to the same subject.  Trust to perform that mapping is separate from trust to sign tokens and SHOULD be established explicitly.

A translated `sub` is authoritative only within the trust context of the issuer that performed the translation.  A recipient that relies only on the issued token MAY evaluate the translated `sub` as the subject in that issuer's namespace.  A recipient needing proof of equivalence to an upstream subject MUST obtain additional evidence, such as another specification, an identity-chain mechanism, or an explicit trust agreement.  If required evidence is unavailable, it SHOULD reject the token.

This profile provides neither portable subject-equivalence proofs nor a general mechanism for correlating subjects across domains.

## Presenter Binding

Without top-level presenter proof of possession, a leaked token can be replayed by any party.  Clients should also use `resource` ({{RFC8707}}) when requesting delegated tokens, because audience restriction limits where a leaked token can be used.

*  When a token carries top-level `cnf`, the RS validates the presenter proof against it ([Sender Constraint and Proof-of-Possession Validation](#delegated-pop-validation)).  For example, JWT access tokens commonly use DPoP or mTLS, while Transaction Tokens can use the workload proof mechanism defined by their deployment profile.
*  A sender-constrained delegated token binds the current presenter, the outermost actor, which reduces delegation-token theft risk.

This document does not define per-hop actor-key provenance within the delegation chain.  Deployments that need stronger assurance for prior-hop provenance MUST use an additional mechanism outside the scope of this document, such as signed hop receipts, transparency-log-based recording, or another future extension; they MUST NOT overload `act.iss` or redefine nested `act` semantics to carry that provenance.  Companion profiles that supply such mechanisms are subject to [Companion Profiles and Extension Points](#companion-profile-extensibility).

## Delegation Depth Limits

Unbounded delegation chains increase attack surface and complicate policy evaluation.  Depth support and interoperability requirements are defined in [Delegation Chains](#delegation-chains).  Rejecting chains that exceed the configured local maximum, as that section requires, also prevents denial-of-service through chain parsing.

Token size grows in proportion to chain depth when extensions to this profile attach per-hop signed material to a delegated token.  Deployments that combine the reference chain depth with one or more such per-hop mechanisms SHOULD measure realistic token sizes against their transport limits (HTTP header limits are often 8 KB) and consider using token introspection ({{RFC7662}}) where inline carriage exceeds those limits.

## Actor Identity Rotation {#actor-identity-rotation}

The canonical actor identifier under this profile is the (`act.iss`, `act.sub`) pair.  Deployments SHOULD choose `act.sub` to be a durable, stable identifier independent of ephemeral key material.  In particular, deployments SHOULD NOT use a JWK thumbprint or other key-derived value as `act.sub`; doing so silently changes the actor identity on every key rotation, which can break delegation grants and policy bindings that reference the prior identifier.  Key rotation (replacing the DPoP key or mTLS certificate bound to an actor) does not require changing `act.sub` when the identifier is key-independent.

When `act.sub` itself must change (for example, because an agent instance is replaced, a workload is renamed, or an actor identifier namespace migrates), existing delegation grants and issued tokens continue to reference the old identifier.  Deployments MUST explicitly re-establish delegation grants for the new identity; the old grants do not automatically transfer.  Outstanding tokens issued under the old identity remain valid until their `exp` time; short token lifetimes bound the exposure window during the transition.

## Delegation Revocation {#delegation-revocation}

Token revocation {{RFC7009}} does not by itself revoke a delegation relationship or propagate revocation to downstream tokens.  Without an additional revocation mechanism, those tokens can remain usable until expiration; see [Token Lifetime for Delegation Chains](#delegation-chain-token-lifetime).

An AS with authoritative knowledge that a delegation has been revoked SHOULD refuse new tokens for that (subject, actor) pair.  Implementations MUST NOT skip revocation checks because of chain depth.

Long-lived refresh behavior can delay revalidation of upstream delegation.  Refresh-token policy, delegation-state storage, and cross-hop revocation remain deployment-specific.  Sensitive resources can use [Token Introspection](#token-introspection) for current token status.

# Privacy Considerations {#privacy}

Delegation chains can reveal sensitive information about user behavior, enterprise topology, software suppliers, and internal tool composition. Issuers therefore SHOULD disclose only the actor information needed by the relying party for authorization, audit, or policy enforcement.

Cross-domain deployments SHOULD prefer stable but non-reassigned identifiers and SHOULD consider pairwise identifiers for human subjects when a globally correlatable identifier is not required by the use case.

When the same logical entity can appear in different identifier namespaces, such as `azp`, `req_wl`, and `act.sub`, issuers and relying parties SHOULD use explicit issuer scoping and locally trusted mapping rules rather than string equality alone to determine whether those identifiers refer to the same entity.

Issuers SHOULD minimize disclosure of prior actors by audience and token-design decisions made before issuance.  Once an issuer preserves a delegation chain, [Preserve Inbound Chain](#preserve-inbound-chain) requires copying it intact.  If local privacy requirements would require omitting a chain element that would otherwise be security-relevant to the recipient's evaluation, the issuer rejects the request rather than truncating the chain.

A Transaction Token's `txn` value links service calls in the same transaction and can enable correlation across organizations.  Deployments SHOULD follow the privacy guidance in {{I-D.ietf-oauth-transaction-tokens}} when propagating it across trust domains.

`act.sub_profile` reveals the actor's entity type, including whether it is an AI agent.  In some jurisdictions or deployment contexts, this disclosure may be legally significant or may reveal sensitive information about user behavior and tool composition.  Issuers SHOULD consider audience-specific disclosure constraints and SHOULD omit unnecessary entity classifications when constructing new actor objects.  Inherited actors remain subject to the preservation rules in [Preserve Inbound Chain](#preserve-inbound-chain).

`req_wl` can reveal internal workload topology.  A TTS SHOULD disclose it only where needed for authorization, audit, or policy enforcement, and SHOULD avoid exposing internal workload identifiers across domains unless the deployment requires it.


# IANA Considerations

## OAuth URI Registration {#iana-oauth-uri}

This document requests IANA to register the following value in the "OAuth URI" registry:

*  URN: `urn:ietf:params:oauth:grant-profile:actor-profile`
*  Common Name: OAuth Actor Profile for Delegation
*  Change Controller: IETF
*  Reference: [Authorization Server Metadata](#authorization-server-metadata) of this document


## OAuth Authorization Server Metadata Registry

This document requests IANA to register the following values in the "OAuth Authorization Server Metadata" registry ({{RFC8414}}):

*  Metadata Name: `actor_profile_token_exchange`
*  Metadata Description: JSON object advertising coarse Token Exchange capabilities for requests in which actor-profile processing can apply
*  Change Controller: IETF
*  Reference: [Authorization Server Metadata](#authorization-server-metadata) of this document


## OAuth Protected Resource Metadata Registry

This document requests IANA to register the following values in the "OAuth Protected Resource Metadata" registry ({{RFC9728}}):

*  Metadata Name: `actor_profile_required`
*  Metadata Description: Boolean indicating whether the RS advertises that delegated requests for this resource are expected to provide actor-profile information conforming to this document's semantics
*  Change Controller: IETF
*  Reference: [Protected Resource Metadata](#protected-resource-metadata) of this document


## OAuth Error Registry {#iana-error-codes}

This document requests IANA to register the following value in the "OAuth Extensions Error Registry" ({{RFC6749, Section 11.4}}):

*  Error Name: `actor_unauthorized`
*  Error Usage Location: Token endpoint response, resource server response
*  Related Protocol Extension: OAuth Actor Profile for Delegation
*  Change Controller: IETF
*  Reference: [Error Responses](#actor-profile-error-responses) of this document


## OAuth Token Introspection Response Registry

This document requests IANA to register the following value in the "OAuth Token Introspection Response" registry ({{RFC7662, Section 3.3}}):

*  Claim Name: `chain_complete`
*  Claim Description: Boolean indicating whether the `act` delegation chain in the introspection response is complete.  When `false`, one or more inner `act` chain entries have been omitted from the response for privacy reasons.  When absent, the chain SHOULD be treated as complete unless local policy or deployment context indicates otherwise.
*  Change Controller: IETF
*  Reference: [Token Introspection](#token-introspection) of this document


## JWT Claims Registry

This document does not request independent JWT Claims Registry entries for the `act` object sub-claims (`iss`, `sub_profile`, and any extension claims) it defines or profiles.  These values appear only within the JSON object value of the `act` claim, which is already registered in the JWT Claims Registry by {{RFC8693}}.  Sub-object keys within a registered claim are scoped to that claim's JSON object and do not require separate top-level registry entries.


## OAuth Token Type Registry {#iana-token-types}

This document makes no independent requests to the "OAuth Token Type" registry for `urn:ietf:params:oauth:token-type:txn_token`.  That URI is defined and registered by {{I-D.ietf-oauth-transaction-tokens}}.  Its inclusion as a defined value for `actor_profile_token_exchange.requested_token_types_supported` in [Authorization Server Metadata](#authorization-server-metadata) is contingent on the progression of {{I-D.ietf-oauth-transaction-tokens}}.


## OAuth Entity Profiles Registry {#iana-entity-profiles}

This document makes no independent requests to the "OAuth Entity Profiles" registry.  It normatively depends on the "Actor Profile" usage location, the `actor` array in `entity_profiles_supported`, and the registration of `user`, `service`, and `ai_agent` with that usage location, all of which are defined and requested by {{I-D.mora-oauth-entity-profiles}}.  The IANA actions for those entries are contingent on the progression of {{I-D.mora-oauth-entity-profiles}}.


--- back

# Service-to-Service Delegation Example {#appendix-service-to-service}

This appendix gives a non-AI example of the actor profile in a same-domain service-to-service delegation flow.  A payroll batch processor acts on behalf of a human payroll administrator to call a payroll API; the payroll API then exchanges that access token for an internal Transaction Token used to write an audit record.

## Scenario

| Party | Identifier |
|-------|------------|
| Payroll Administrator | `https://idp.example.com/users/pat` |
| Enterprise AS | `https://as.example.com` |
| Payroll Batch Processor | `https://services.example.com/payroll-batch` |
| Payroll API | `https://services.example.com/payroll-api` |
| Audit TTS | `https://tts.example.com` |
| Audit Service | `https://internal.example.com/audit` |

The batch processor is an OAuth client and also the acting service.  The client registration remains identified by `client_id`; the acting service is represented explicitly in `act.sub`.

## Access Token

The Enterprise AS issues a JWT access token for the Payroll API:

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

This is a single-hop actor object: `act` is present, but it contains no nested `act`.  `sub` identifies the payroll administrator, while `act.sub` identifies the service exercising that administrator's authorization.

## Transaction Token

After processing the payroll request, the Payroll API exchanges the inbound access token at the Audit TTS to call the internal Audit Service.  The Payroll API is the requesting workload (`req_wl`).  The TTS validates the inbound delegation chain, preserves it as an inner `act`, and adds a new outermost actor for the Payroll API:

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

The administrator remains the subject.  The Payroll API becomes the outermost actor, and the batch processor remains as the inner actor.

# Cross-Domain AI Agent Flow: ID Token to Transaction Token {#appendix-cross-domain}

This appendix follows a delegated request across two trust domains.  Token validation and presenter proofs follow the underlying token specifications and deployment profile.

All claim values, JKT thumbprints, and domain names are synthetic.  Long HTTP example lines use the backslash convention in {{RFC8792}}; other line breaks in form bodies are for readability.

## Scenario and Parties

Alice authenticates at the Enterprise IdP AS.  Her Travel Assistant exchanges the ID token for an ID-JAG, then presents that grant to the Travel Provider AS for an access token.  The agent calls the Booking Tool, which exchanges the access token for a Transaction Token to call the Inventory Service.

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

Presenter key bindings:

| Principal | JWK Thumbprint (`jkt`) |
|-----------|------------------------|
| Travel Assistant | `AgentJKT-NzbLsXh8uDCcd7MN` |
| Booking Tool | `ToolJKT-0ZcOCORZNYy9ZhHi` |


## Capability Discovery (Preflight)

The agent consults the Travel Provider AS metadata ([Metadata and Discovery](#metadata-and-discovery)) as an advisory compatibility check before initiating the flow:

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

The agent confirms that its `sub_profile` (`ai_agent`) is in `entity_profiles_supported.actor`, that ID-JAG and actor-profile grants are advertised in `authorization_grant_profiles_supported`, and that the planned input/output token types appear in `actor_profile_token_exchange`.  These signals are coarse compatibility indicators only; the agent proceeds because the advertised capabilities cover its planned path.


## Step 1: User Authentication (ID Token)

Alice authenticates to the Enterprise IdP AS, which issues an ID Token.  An ID Token implicitly identifies a user; the entity type is not carried in a `sub_profile` claim and is established by the AS in subsequent issued tokens (Step 2):

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

The agent presents Alice's ID Token as `subject_token` in a Token Exchange request to the Enterprise IdP AS, requesting an ID-JAG ({{I-D.ietf-oauth-identity-assertion-authz-grant}}).  The agent's RFC 7523 client assertion serves as both `client_assertion` (for client authentication) and `actor_token` (for actor identity), per [JWT Client Assertion](#jwt-client-assertion-as-actor-token).  The Enterprise IdP AS authenticates the client, verifies the ID Token audience matches that client, and uses local delegation policy to construct the issued ID-JAG:

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
&audience=https%3A%2F%2Fas.travel-provider.example%2F
&resource=https%3A%2F%2Fas.travel-provider.example
&scope=booking%3Acreate
&client_id=https%3A%2F%2Fagents.enterprise.example%2Ftravel-assistant
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A\
  client-assertion-type%3Ajwt-bearer
&client_assertion=<travel-assistant-client-assertion>
&actor_token=<travel-assistant-client-assertion>
&actor_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Ajwt
~~~

The Enterprise IdP AS validates the shared JWT for client authentication and actor identity, verifies the DPoP proof, and confirms Alice's delegation under local policy.  It applies scope reduction and binds the issued ID-JAG to the demonstrated DPoP key:

~~~json
{
  "iss": "https://as.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
  "client_id": "https://agents.enterprise.example/travel-assistant",
  "azp": "https://agents.enterprise.example/travel-assistant",
  "aud": "https://as.travel-provider.example/token",
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

Here, `client_id`, `azp`, and `act.sub` use the same URI because the agent is both client and actor.  The top-level `cnf.jkt` binds the grant to the agent's key.


## Step 3: Agent Exchanges ID-JAG for Access Token at Travel Provider AS

The agent presents the ID-JAG as a JWT Bearer authorization grant ({{RFC7523}}) to the Travel Provider AS, which processes it as an ID-JAG per {{I-D.ietf-oauth-identity-assertion-authz-grant}} with the actor-profile rules in [Authorization Grant Processing](#jwt-assertion-grants-processing):

~~~
POST /token HTTP/1.1
Host: as.travel-provider.example
Content-Type: application/x-www-form-urlencoded
DPoP: <AgentJKT-proof>

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
&assertion=<id-jag>
&scope=booking%3Acreate
~~~

The Travel Provider AS performs actor-profile processing per [Authorization Grant Processing](#jwt-assertion-grants-processing): it verifies the request's DPoP proof against the top-level `cnf.jkt` in the inbound ID-JAG and checks that `act.sub_profile` (`ai_agent`) is permitted as an actor for the requested scope under local policy.  It issues an access token preserving the delegation chain:

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

The Booking Tool RS applies authorization of the (`sub`, outermost `act.sub`) pair ([Resource Server Processing](#resource-server-processing)): it evaluates Alice (`sub`, `sub_profile: user`) together with the Travel Assistant (`act.sub`, `sub_profile: ai_agent`) for the requested operation.  The `act.sub_profile` value is checked against `entity_profiles_supported.actor` per [Authorization Server Metadata](#authorization-server-metadata).


## Step 5: Booking Tool Exchanges Access Token for Transaction Token

The Booking Tool cannot reuse the received access token for internal calls: it is sender-constrained to `AgentJKT`, which the Booking Tool does not possess.  It requests a Transaction Token from the TTS.  In this example, the TTS receives the Booking Tool's WIMSE Workload Identity Token (WIT) as the Token Exchange `actor_token` and validates a Workload Proof Token (WPT).  The WIT identifies the Booking Tool and carries its confirmation key, while the WPT proves possession of that key and binds the request to the accompanying access token:

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
&rctx={"req_ip":"198.51.100.42"}
~~~

The WIT is therefore the JWT `actor_token` defined by this profile, while the WPT provides the accompanying proof of possession required by the workload-credential profile.

The TTS applies actor-profile processing per [Transaction Token Output Rules](#transaction-token-output-rules): it preserves `sub` and `sub_profile` from the `subject_token`, sets `req_wl` to the authenticated Booking Tool, and creates a new outermost `act` object for the Booking Tool while nesting the `subject_token`'s existing `act` claim beneath it.  In this WIMSE-based deployment, the underlying Transaction Token mechanism also binds the issued token to the Booking Tool's presenter key (`ToolJKT`) identified in the WIT confirmation claim:

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

The presenter binding rotates at this step: `cnf.jkt` is now `ToolJKT` because the Booking Tool is the current presenter.


## Step 6: Booking Tool Calls Inventory Service

~~~
GET /inventory?origin=SFO&dest=NYC&depart=2026-04-15 HTTP/1.1
Host: internal.travel-provider.example
Txn-Token: <txn-token>
Workload-Identity-Token: <booking-tool-wit>
Workload-Proof-Token: <tool-wpt-with-wth-and-tth>
~~~

The Inventory Service validates the WIT and WPT, then authorizes Alice and the Booking Tool as the subject and immediate actor.  It uses `req_wl` as supporting workload context.  The Travel Assistant remains prior-actor context and is not used for access control at this tier.


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
*  The presenter-binding key rotates once, at Step 5 when the TTS re-binds the Transaction Token to the Booking Tool's key.
*  At Step 5 the TTS creates a new outermost `act` for the Booking Tool and nests the prior `act` chain beneath it.


# Acknowledgments
{:numbered="false"}

The author thanks the OAuth Working Group for the specifications on which this profile builds.

# Document History
{:numbered="false"}

[[ To be removed from the final specification ]]

-01

* Consolidated and tightened the text throughout; claim roles, supported token types, and error mappings now use tables.
* Added hop and visible-hop terminology, and token-size guidance for extensions that attach per-hop signed material.
* JWT access tokens now require `client_id`, per {{RFC9068}}.
* Corrected citations: scope reduction and resource indicators are no longer attributed to {{RFC8693}}, and the DPoP key comparison cites Sections 4.3 and 6.1 of {{RFC9449}}.
* `may_act` now uses `may_act.iss` as the identifier context when present, and the `subject_token` issuer otherwise.
* Removed guidance on merging an `actor_token`'s own `act` chain, which conflicted with rejecting any `actor_token` that carries `act`.
* The `act.iss` rule for AS-issued grants and the privacy guidance on suppressing `act.sub_profile` now apply only to new actor objects, consistent with inherited-actor immutability.
* Aligned the error-response table with the processing rules.
* Redrew the cross-domain example diagram and added {{RFC8792}} line-wrapping headers to folded examples.
* Resolved conflicting requirements on inherited extension members, inner-actor validation, and `req_wl` reconciliation, and pointed Security and Privacy restatements at their normative rules.
* Removed BCP 14 keywords from operational guidance that no other party can observe.
* Consolidated duplicated requirements into single homes and cited dependencies instead of restating them.
* Resolved the remaining duplicate-rule conflicts: `act` in Transaction Tokens follows Delegation Chains, identifier reconciliation keeps both outcomes under explicit conditions, client identity must not substitute for `act`, inner-actor failures use the shared error mapping, and proof for a new presenter is required for sender-constrained output.

-00

* Initial version.

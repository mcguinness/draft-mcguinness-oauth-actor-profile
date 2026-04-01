--- 
title: "OAuth Actor Profile for Delegation" 
abbrev: "OAuth Actor Profile"
category: std
docname: draft-mcguinness-oauth-actor-profile-latest
submissiontype: IETF
number:
date: 2026-03-31
ipr: "trust200902"
area: Security
workgroup: Web Authorization Protocol
keyword:
 - oauth
 - delegation
 - actor
 - ai agents
 - token exchange
 - transaction tokens
 - jwt
 - cross-domain
venue:
  group: Web Authorization Protocol
  type: Working Group
  mail: oauth@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/oauth/
  github: mcguinness/draft-mcguinness-oauth-actor-profile
  latest: https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-profile.html

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: me@karlmcguinness.com

normative:
  RFC3986:
  RFC6750:
  RFC7009:
  RFC7519:
  RFC7517:
  RFC7521:
  RFC7523:
  RFC7662:
  RFC7800:
  RFC8126:
  RFC8414:
  RFC9728:
  RFC8693:
  RFC9068:
  RFC9449:
  I-D.ietf-oauth-transaction-tokens:
  I-D.mora-oauth-entity-profiles:
    title: "OAuth Entity Profiles"
    author:
     -
        fullname: Sreyantha Chary Mora
        organization: Microsoft
     -
        fullname: Pamela Dingle
        organization: Microsoft
    date: 2025-10-17
    target: https://www.ietf.org/archive/id/draft-mora-oauth-entity-profiles-00.txt

informative:
  RFC6749:
  RFC8705:
  RFC9700:
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
    date: 2026-03-02
    target: https://www.ietf.org/archive/id/draft-ietf-oauth-identity-assertion-authz-grant-02.txt
  I-D.ietf-oauth-security-topics:
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

OAuth 2.0 deployments increasingly involve multi-principal scenarios where an AI agent, automated workload, or service acts on behalf of a human user or another principal across organizational and trust-domain boundaries. Existing specifications provide relevant building blocks, including the `act` claim in Token Exchange (RFC 8693), JWT assertion grants (RFC 7523), JWT-formatted access tokens (RFC 9068), Demonstrating Proof of Possession (RFC 9449), and Transaction Tokens.  However, they do not define a common profile for representing delegation relationships across these token types, classifying actor entity types, or signaling support between authorization servers and resource servers.

This document defines the OAuth Actor Profile for Delegation: a common structure for the `act` claim that applies uniformly across JWT assertion grants, JWT access tokens, and Transaction Tokens.  It defines the `sub_profile` claim for machine-processable actor classification, specifies how proof-of-possession key binding applies to the current presenter and to prior actors in a delegation chain, and defines processing rules for authorization servers and resource servers.  The document also extends the existing OAuth Entity Profiles registry with an Actor Profile usage location, updates metadata for actor classification, and registers authorization server and protected resource metadata parameters for advertising and negotiating support for the profile in cross-domain deployments. The profile is intended to preserve actor semantics across supported token transformations, including exchanges among JWT assertion grants, JWT access tokens, and Transaction Tokens.


--- middle

# Introduction

The deployment of AI agents and automated workloads as first-class participants in OAuth-protected systems has exposed a long-standing gap: there is no single, interoperable way to express "this request was authorized to principal A, and that authorization is now being exercised by actor B on A's behalf."

## Problem Statement

The Token Exchange specification {{RFC8693}} introduced the `act` claim to represent the current actor, but its definition is tied to token exchange.  JWT assertion grants {{RFC7521}}{{RFC7523}} do not define a corresponding actor structure.  JWT-formatted access tokens {{RFC9068}} do not define how `act` is carried or validated.  Transaction Tokens {{I-D.ietf-oauth-transaction-tokens}} provide a workload-identity envelope but likewise do not define a common actor profile.

As a result, deployments often define proprietary conventions.  Resource servers cannot reliably determine which principal authorized a request, which actor is presenting it, or whether an actor chain should be trusted across domain boundaries.

## Solution Overview

This document addresses the gap by specifying:

*  A common actor profile structure that reuses `act` from {{RFC8693}} and adds `sub_profile` for entity-type classification and `cnf` for actor key binding.
*  How the actor profile is expressed and validated in each token type: JWT assertion grants ({{jwt-assertion-grants}}), JWT access tokens ({{jwt-access-tokens}}), and Transaction Tokens ({{transaction-tokens}}).
*  How actor-profile information is preserved when one supported token type is exchanged for another across JWT assertion grants, JWT access tokens, and Transaction Tokens.
*  Dual-principal authorization guidance ({{dual-principal-authorization}}), allowing resource servers to enforce policies over the (subject, actor) pair while accommodating deployments where delegation is informational only.
*  Extensions to OAuth Entity Profiles usage locations and metadata ({{entity-profile-extension}}) so existing entity classifications can be used consistently in actor position.
*  Discovery metadata and capability-negotiation procedures ({{discovery-capability-negotiation}}) for authorization servers that offer Token Exchange or Transaction Token issuance, and for resource servers that consume delegated tokens.

The primary motivating use case is cross-domain delegation involving AI agents and automated workloads, but the mechanisms are general-purpose and apply to other delegated-authorization scenarios.  This document is a profile and extension of existing OAuth building blocks; unless stated otherwise, the requirements of {{RFC8693}}, {{RFC9068}}, {{RFC9449}}, and {{I-D.ietf-oauth-transaction-tokens}} continue to apply.


## Illustrative Use Case

Alice authorizes an AI agent to book a business trip on her behalf. The agent calls an external booking tool operated in a separate organizational trust domain.  To do this:

1.  Alice authenticates at her enterprise identity provider authorization server (Enterprise IdP AS), and the agent obtains an ID Token establishing Alice's identity.  This authentication step is out of scope for this document; the OAuth-relevant processing begins when the Enterprise IdP AS receives the upstream credential.
2.  The agent exchanges the ID Token at the Enterprise IdP AS to obtain an Identity Assertion JWT Authorization Grant (ID-JAG): a JWT produced by token exchange that carries Alice's identity, the agent's actor profile, and an audience bound to the external tool AS's token endpoint.  When the agent later presents that JWT to another AS using the JWT bearer grant, the JWT functions as a profiled JWT assertion grant.
3.  The agent submits the ID-JAG as a JWT bearer authorization grant to the external tool's AS token endpoint.  The tool AS validates the enterprise chain and issues an access token for the external tool's API.
4.  The agent calls the external tool's API using that access token.
5.  The external tool exchanges the received access token at its Transaction Token Service (TTS) to obtain a Transaction Token for a backend internal service call, with the issued token bound to the tool's own DPoP key rather than the agent's.
6.  The tool calls the internal service using the Transaction Token.

At each step, Alice's identity is preserved as `sub`, the agent's identity is preserved in `act`, the current presenter holds a key-bound token, and the trust-domain boundary enforcer (the tool AS) re-issues the credential under local control.  No single specification currently achieves all of this uniformly across JWT assertion grants, access tokens, and Transaction Tokens.

This profile closes this gap.


# Conventions and Definitions {#conventions}

{::boilerplate bcp14-tagged}

The following terms are used in this document:

Actor:
: The party that is actively making a request.  When delegation is present, the actor is distinct from the subject; the subject is the principal on whose behalf the actor is acting.

Subject:
: The principal whose authorization is being exercised.  In a delegated token, the subject is the original authorizing party (e.g., an end-user or an upstream service), not the party making the immediate network request.

Delegation:
: The act by which a principal authorizes another party (the actor) to exercise a subset of the principal's rights.

Cross-Domain Delegation:
: A delegation scenario in which the top-level `sub` and `act.sub` are not issued within the same issuer namespace.  In this document, a token is considered cross-domain when the subject and actor do not share the same issuer, as determined by the top-level `iss` for `sub` and by the issuer value carried in the relevant actor object for the actor.

Dual-Principal Authorization:
: An authorization policy evaluation that considers both the subject and the actor as independent policy inputs.

Token Profile:
: A named, registered specification of how a particular set of claims MUST or SHOULD appear in one or more OAuth token types, together with the processing rules for issuers and consumers.

Actor Profile:
: The specific token profile defined in this document for representing delegation relationships via the `act` claim.

Transaction Token (Txn-Token):
: A token type defined in {{I-D.ietf-oauth-transaction-tokens}} that captures workload identity and request context for calls within a single transaction.

Authorization Server (AS):
: The server that issues tokens as defined in {{RFC6749}}.

Transaction Token Service (TTS):
: An authorization server that issues Transaction Tokens as defined in {{I-D.ietf-oauth-transaction-tokens}}.

Resource Server (RS):
: The server hosting protected resources as defined in {{RFC6749}}.


# Motivation and Interoperability Gaps

## Overloaded Subject Semantics

The `sub` claim of a JWT access token {{RFC9068}} is routinely overloaded to represent heterogeneous entity types: an end-user, a service account, a workload identifier, or an AI agent.  Resource servers must rely on out-of-band conventions or proprietary claim extensions to determine which type of entity `sub` refers to.  This ambiguity prevents deterministic cross-domain policy evaluation.

## Inconsistent Actor Representation

{{RFC8693}} defines the `act` claim for tokens obtained via token exchange but does not extend its semantics to JWT assertion grants or JWT access tokens issued by other means.  Deployments that do not use token exchange (for example, those using JWT client assertions for service-to-service calls) have no standard location for actor information.  Even when `act` is present, there is no standard sub-claim that identifies the entity type of the actor.

## Missing Proof-of-Possession Binding for Actors

Sender-constrained tokens {{RFC9449}}{{RFC8705}} bind the token to a proof-of-possession key identified in the top-level `cnf` claim.  When delegation is present, the presenter is the actor, not the original subject.  There is no standard mechanism to separately bind the actor's key within the `act` claim, leaving implementations to use the top-level `cnf` for both the token holder and the actor simultaneously, which is semantically ambiguous in chained delegation scenarios.

## No Cross-Token Profile Consistency

JWT assertion grants, JWT access tokens, and Transaction Tokens are specified in different documents with different claim conventions.  There is no common profile that spans all three and specifies how actor information flows from an assertion presented to an AS through to the access token issued by that AS and into any downstream Transaction Token.

## Absent Discovery and Capability Negotiation

Neither AS metadata {{RFC8414}} nor Protected Resource Metadata {{RFC9728}} define parameters for advertising actor-profile support.  Deployments therefore require bilateral out-of-band configuration, which is impractical for AI agents that dynamically discover and invoke tools across organizational boundaries.


# Actor Profile for Delegation {#actor-profile}

## Overview

This profile specifies an extended form of the `act` claim defined in {{RFC8693}}.  When an implementation elects to use this profile in a context where an actor is distinct from the subject, it MUST apply the profile as defined in this section.

The actor profile is applicable to:

*  JWT assertion grants ({{jwt-assertion-grants}})
*  JWT-formatted access tokens ({{jwt-access-tokens}})
*  Transaction Tokens ({{transaction-tokens}})

Subject to endpoint policy and the underlying token-exchange or grant mechanism, implementations MAY transform any supported input token type into any other supported output token type.  When they do so, actor profile information MUST be preserved and validated as specified in this document for the resulting token type.

## Scope of This Profile

This document standardizes the structure and processing of actor-profile claims when they are carried in supported token types.  In particular, it standardizes:

*  the actor object structure and its claim semantics,
*  how actor-profile information is preserved across supported token transformations,
*  how authorization servers and resource servers validate and consume those claims, and
*  the metadata used to advertise actor-profile support.

This document does not standardize:

*  which token transformations a given authorization server or TTS must support,
*  the local policy by which a subject authorizes a specific actor,
*  the trust framework by which one domain decides to trust another domain's issuer, or
*  the identifier-mapping logic by which different namespaces are determined to refer to the same logical entity.

Those decisions remain deployment- and policy-specific, except where a referenced specification defines them more precisely.


## Actor Object Structure

An actor object is a JSON object that is the value of the `act` claim. In addition to the `sub` claim required by {{RFC8693}}, an actor object MUST contain an `iss` claim, SHOULD contain a `sub_profile` claim, and MAY contain a `cnf` claim.

~~~
act-object = {
  "sub"         : StringOrURI,          ; REQUIRED
  "iss"         : StringOrURI,          ; REQUIRED
  "sub_profile" : string,               ; RECOMMENDED
  "cnf"         : cnf-object,           ; OPTIONAL
  * StringOrURI => any                  ; extension claims
}
~~~

`sub`:
: REQUIRED.  The subject identifier of the actor, as defined in {{RFC8693, Section 4.1}}.  This value identifies the acting party.  It is a StringOrURI as defined in {{RFC7519}}.

`iss`:
: REQUIRED.  The issuer that is authoritative for the actor identifier in `act.sub`.  This value scopes `act.sub` to an issuer namespace so that relying parties can distinguish otherwise-colliding subject identifiers across domains.  See "Cross-Domain Delegation" in {{conventions}}.  The value is a StringOrURI as defined in {{RFC7519}}.

`sub_profile`:
: RECOMMENDED.  A space-delimited list of entity profile values classifying the actor identified by `act.sub`, as defined in {{I-D.mora-oauth-entity-profiles, Section 4.2}}.  Values MUST be drawn from the OAuth Entity Profiles registry ({{I-D.mora-oauth-entity-profiles, Section 14.1}}) or be privately defined collision-resistant values.  Only values registered with the "Actor Profile" usage location (defined in {{entity-profile-extension}}) SHOULD be used within `act` objects.  If the acting entity fits more than one profile, multiple values MAY be included as a space-delimited string (e.g., `"service ai_agent"`).

`cnf`:
: OPTIONAL.  A confirmation claim as defined in {{!RFC7800}}.  When present, this value identifies the keying material the actor MUST use to demonstrate proof of possession when presenting tokens that contain this actor object.  This keying material is distinct from any top-level `cnf` claim, which constrains the current bearer of the overall token.  When a public key is conveyed in `act.cnf`, the `jkt` member is RECOMMENDED.


## Top-Level Subject Classification

The `sub_profile` claim MAY also appear as a top-level claim in a JWT (outside any `act` object) to classify the entity type of the token's `sub` claim.  When present at the top level:

*  The value MUST be a space-delimited entity profile string as defined in {{I-D.mora-oauth-entity-profiles, Section 3.3}}, drawn from the OAuth Entity Profiles registry or privately defined.
*  The claim applies exclusively to the top-level `sub`; it does not affect or override `sub_profile` values within `act` objects.
*  Entity-type profiles are not restricted to actor contexts; for example, `user` is valid at the top level of an access token whose `sub` is an end-user.

Issuers SHOULD include a top-level `sub_profile` when they can authoritatively classify the subject entity type.


## Delegation Chains {#delegation-chains}

Delegation chains MUST be represented by nesting `act` objects as specified in {{RFC8693, Section 4.1}}.  In a nested structure, the outermost `act` object identifies the immediate actor; inner `act` objects represent prior actors in the chain, with the innermost representing the original delegating party.

~~~json
{
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
  "act": {
    "sub": "https://tools.example.com/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service",
    "cnf": { "jkt": "0ZcOCORZNYy...9ZhHiZN" },
    "act": {
      "sub": "https://agents.example.com/travel-assistant",
      "iss": "https://as.enterprise.example",
      "sub_profile": "ai_agent",
      "cnf": { "jkt": "NzbLsXh8uDCcd7MNwrnNZpX0ak8A...CQ" }
    }
  }
}
~~~
In this example the booking tool (outermost `act`) is the immediate actor acting for the travel assistant (nested `act`), which is acting for Alice (`sub`).  Each actor has its own key binding via `cnf.jkt`.

Implementations MUST NOT represent delegation depth greater than eight levels.  Implementations that receive a token with more than eight levels of nested `act` MUST reject it.


## Sender Constraint and Key Binding {#sender-constraint}

When sender-constrained tokens are used with a delegated token, the top-level `cnf` claim identifies the key or certificate of the immediate presenter of that token.  When delegation is present, that immediate presenter is the outermost actor.

When DPoP {{RFC9449}} is used:

*  The top-level `cnf.jkt` of the token MUST identify the key of the immediate presenter—the actor identified by the outermost `act` claim.
*  The `cnf.jkt` within each `act` object identifies the key associated with that specific actor in the delegation chain.

This separation ensures that the resource server can verify the current presenter while preserving auditable bindings for prior actors in the delegation chain.

When mTLS ({{RFC8705}}) is used instead of DPoP, the top-level `cnf.x5t#S256` identifies the current presenter's certificate. Actor-level key binding in `act.cnf` SHOULD use a confirmation member appropriate to the mechanism in use; where a public key is conveyed, `jkt` is RECOMMENDED.

## Backwards Compatibility

This profile extends the base `act` claim semantics from {{RFC8693}} by requiring `iss` within each actor object and by defining `sub_profile` and optional actor-scoped confirmation semantics.  Deployments that receive an `act` object that conforms only to {{RFC8693}} but omits this profile's required members MUST treat that actor object as not conforming to this profile.

When a token or assertion is required by local policy or advertised metadata to conform to this profile, such non-conforming `act` objects MUST be rejected.  When profile conformance is not required, implementations MAY continue to process a base {{RFC8693}} `act` object according to local policy, but they MUST NOT infer profile-defined semantics for claims that are absent.


# JWT Assertion Grants {#jwt-assertion-grants}

## Structure {#jwt-assertion-grants-structure}

A JWT used as an authorization grant {{RFC7521}}{{RFC7523}} MAY include an `act` claim conforming to the actor profile defined in {{actor-profile}}.  Use of this claim in JWT client authentication assertions is out of scope for this document because such assertions have different issuer and subject semantics.  When included:

*  The `sub` of the JWT identifies the principal on whose behalf the grant is being made.
*  The `act` object identifies the entity presenting the assertion or on whose behalf the assertion issuer is speaking.
*  The `iss` of the JWT MUST identify the assertion issuer, and that issuer MUST be authorized by local policy to assert the relationship between `sub` and `act.sub`.

An Identity Assertion JWT Authorization Grant (ID-JAG) is a JWT that can be produced by token exchange and then presented using the JWT bearer grant.  When an ID-JAG is used as a JWT bearer assertion grant, this section applies to it.  In that usage, the ID-JAG carries delegation information using this profile.  The term and token type originate in {{I-D.ietf-oauth-identity-assertion-authz-grant}}.

~~~json
{
  "iss": "https://agents.example.com/travel-assistant",
  "sub": "https://idp.enterprise.example/users/alice",
  "aud": "https://as.resource-domain.example/token",
  "jti": "a1b2c3d4-...",
  "exp": 1711820400,
  "iat": 1711816800,
  "sub_profile": "user",
  "act": {
    "sub": "https://agents.example.com/travel-assistant",
    "iss": "https://as.enterprise.example",
    "sub_profile": "ai_agent",
    "cnf": { "jkt": "NzbLsXh8uDCcd7MNwrnNZpX0ak8ACQ" }
  }
}
~~~

The top-level `sub_profile` describes the entity type of the JWT's `sub`.  The `sub_profile` within `act` describes the entity type of the actor.


## Authorization Server Processing {#jwt-assertion-grants-processing}

When an AS receives a JWT assertion grant containing an `act` claim:

1.  The AS MUST validate the assertion per {{RFC7523}}.

2.  The AS MUST determine that the assertion issuer is trusted to assert the relationship between the JWT `sub` and `act.sub`.  If the issuer is not the same as `act.sub`, the AS MUST apply local policy to determine whether the issuer is authorized to speak for that actor.

3.  The AS MUST verify that the `act.sub` is authorized to act on behalf of the assertion's `sub`, using the AS's own policy (for example, a pre-registered delegation grant, a consent record, or a policy rule).

4.  If the inbound assertion's `act` object itself contains a nested `act` claim (indicating that the asserted actor is itself a delegatee), the AS MUST determine whether to propagate that inner chain into the issued token.  The AS SHOULD propagate the inner chain by preserving the nested structure in the issued token, provided that the total resulting chain depth does not exceed the limit in {{delegation-chains}}.  If the AS does not accept pre-chained assertions, it MUST reject the request.  An AS that propagates inner actor chains MUST independently validate that each inner `act.sub` was asserted by a trusted issuer before including it in the issued token.

5.  If the `act` object contains a `cnf` claim indicating a sender-constraining key, the AS MUST verify that the token request demonstrates possession of the corresponding key, using DPoP ({{RFC9449}}) or mutual TLS ({{RFC8705}}) consistent with token endpoint policy.  If possession cannot be verified, the AS MUST reject the request.

6.  If the AS accepts the assertion, it MUST propagate the actor information into the issued token according to the rules for the output token type being issued.  For JWT access tokens, see {{jwt-access-token-propagation}}.  For Transaction Tokens, see {{transaction-token-service-processing}}.  When the output is another JWT assertion grant profile, the resulting assertion MUST preserve the validated actor information subject to local policy and the chain-depth limit in {{delegation-chains}}.

7.  The AS MAY add additional `sub_profile` or `act` metadata to the issued token based on its own knowledge of the principals.


## Error Responses {#assertion-error-responses}

When an AS rejects a JWT assertion grant or Token Exchange request for reasons related to actor-profile validation, it MUST return an OAuth error response per {{RFC6749, Section 5.2}} and {{RFC8693, Section 2.2}}. The following error codes apply:

`invalid_request`:
: Use when the `act` claim structure is syntactically invalid, the delegation chain depth exceeds the limit in {{delegation-chains}}, or a required claim such as `act.sub` is absent.

`invalid_grant`:
: Use when the assertion or subject token cannot be validated, the issuer is not trusted to assert the delegation relationship, or the actor's possession of the key indicated by `act.cnf.jkt` cannot be confirmed.

`access_denied`:
: Use when AS policy does not permit the identified actor (`act.sub`) to act on behalf of the subject (`sub`), or when the requested scope exceeds what is permitted for this (subject, actor) pair.

The `error_description` field SHOULD be included and SHOULD describe which aspect of actor-profile validation failed, to the extent permitted by the AS's security and privacy policy.

Examples in this document are illustrative and focus on actor-profile-related claims and processing.  They can omit unrelated claims, parameters, or validation steps required by the underlying specifications for a complete deployment.


# JWT Access Tokens {#jwt-access-tokens}

## Structure

A JWT access token {{RFC9068}} MUST include an `act` claim conforming to the actor profile defined in {{actor-profile}} when any of the following conditions hold:

*  The token was issued following acceptance of a JWT assertion grant that itself contained an `act` claim per {{jwt-assertion-grants}};
*  The token was issued via Token Exchange ({{RFC8693}}) and an `actor_token` was presented; or
*  The AS has independent knowledge that the subject's rights are being exercised by a distinct acting party.

When none of these conditions hold, an `act` claim MUST NOT be added solely on the basis of the `azp` claim being present.

The `azp` claim ({{RFC9068, Section 2.2}}) identifies the OAuth 2.0 client that requested the token.  In deployments where the requesting client is also the acting party, `azp` and the outermost `act.sub` can refer to the same logical entity using different identifier formats.  This profile does not require that relationship for all JWT access tokens.  However, when an issuer uses both claims to represent the same acting party, it MUST ensure that they are semantically consistent.

The following example shows a JWT access token with actor profile claims.  Here the top-level bearer key (`NzbLsX...`) and the actor's key (`NzbLsX...`) are identical because in this single-hop case the actor is the bearer; in multi-hop chains each actor carries a distinct key, as shown in the detailed cross-domain example in {{appendix-cross-domain}}:

~~~json
{
  "iss": "https://as.resource-domain.example",
  "sub": "https://idp.enterprise.example/users/alice",
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
    "sub_profile": "ai_agent",
    "cnf": { "jkt": "NzbLsXh8uDCcd7MNwrnNZpX0ak8ACQ" }
  }
}
~~~

The top-level `cnf.jkt` identifies the key the immediate bearer (the travel assistant) MUST use to prove possession.  The `act.cnf.jkt` carries the same value in this single-actor scenario because the actor is the sole bearer; in a multi-hop chain these values will differ for each hop.


## Propagation from Assertion Grants and Token Exchange {#jwt-access-token-propagation}

When an AS issues a JWT access token following:

*  Acceptance of a JWT assertion grant that contains an `act` claim ({{jwt-assertion-grants}}), or
*  A Token Exchange request ({{RFC8693}}) that carries an `actor_token`,

the AS MUST include an `act` claim in the issued access token that faithfully represents the delegation relationship conveyed in the inbound request.  The AS MUST NOT silently drop actor information.

If the Token Exchange request itself contains a `subject_token` that already carries an `act` claim (i.e., the delegation chain already has depth > 1), the AS MUST nest the existing `act` structure within a new outermost `act` that represents the newly-identified actor, unless the AS has policy that limits chain depth.

If the inbound actor information cannot be validated or would exceed the maximum chain depth defined in {{delegation-chains}}, the AS MUST reject the request and MUST NOT issue a token that partially preserves the delegation chain.

The AS SHOULD add `sub_profile` to the issued token's top-level claims if it can authoritatively classify the token's `sub` entity type.

The AS MUST apply least-privilege scope reduction when issuing delegated access tokens:

*  The scope of the issued token MUST NOT exceed the scope of the subject's original authorization.
*  When the Token Exchange request specifies a `scope` parameter, the issued token MUST NOT carry scopes beyond those explicitly requested.
*  When no `scope` parameter is specified, the AS SHOULD restrict the issued scope to the minimum necessary for the stated purpose rather than inheriting the full scope of the subject token.
*  The AS SHOULD further restrict scope based on policy for the (subject, actor) pair or the actor's `sub_profile`.

The granted scope MUST be reflected in the issued token's `scope` claim so that resource servers and audit systems can verify that least-privilege constraints were applied.  Deployments involving AI agents or automated workloads SHOULD treat scope reduction as effectively mandatory unless a specific policy exception has been established for the (subject, actor) pair.

The same preservation requirements apply regardless of whether the inbound credential presented to the AS is a JWT assertion grant, a JWT access token, or a Transaction Token, provided that local policy and the underlying grant or token-exchange mechanism permit issuance of a JWT access token from that input.


## Resource Server Processing {#jwt-access-token-rs-processing}

Upon receiving a JWT access token that contains an `act` claim, a resource server MUST:

1.  Validate the token's signature, `iss`, `aud`, and temporal claims per {{RFC9068}}.

2.  If the token carries a top-level `cnf.jkt`, validate the accompanying DPoP proof per {{RFC9449}}.  The DPoP proof MUST be signed by the key identified in `cnf.jkt`, which is the key of the immediate bearer—the outermost actor when delegation is present.

3.  Extract the `sub` and the outermost `act.sub` as the two principals relevant for authorization policy.

4.  Apply dual-principal authorization per {{dual-principal-authorization}}.

5.  Optionally traverse inner `act` objects to audit the full delegation chain; inner actors are informational and SHOULD NOT be required to present proof of possession at the resource server.

6.  If the resource server relies on inner `act` objects for audit, policy refinement, or trust decisions, it MUST do so only after validating the outer token issuer and only when local policy trusts that issuer to carry forward the asserted actor chain.  The RS MUST NOT treat nested actor issuers as independently authenticated merely because their identifiers appear in inner `act.iss` values.

7.  If any of the above steps fail, return an appropriate error response per {{RFC6750, Section 3.1}}:

    *  If signature, `iss`, `aud`, or temporal validation fails: HTTP 401 with `WWW-Authenticate: Bearer error="invalid_token"`.
    *  If DPoP proof validation for `cnf.jkt` fails: HTTP 401 per {{RFC9449, Section 7}}.
    *  If the token is structurally valid but the actor fails dual-principal authorization: HTTP 403.  When the subject's scope was sufficient but the actor lacked authorization, the RS SHOULD include `error="insufficient_scope"` in the `WWW-Authenticate` challenge.  When the actor's `sub_profile` is not in the RS's accepted list, the RS MAY use `error="access_denied"`.
    *  The RS MUST NOT include actor-specific rejection details in error responses exposed to clients outside the trust domain.


## Token Introspection {#token-introspection}

When token introspection ({{RFC7662}}) is used, an AS that issues delegated tokens MUST include the `act` claim and top-level `sub_profile` claim in introspection responses for active tokens that carry those claims.  The AS MUST NOT omit actor profile claims from introspection responses, as their omission would misrepresent the delegation status of the token to the introspecting RS.

Resource servers using introspection for delegated tokens MUST apply the same dual-principal authorization logic described in {{dual-principal-authorization}} to the introspection response claims.  An RS that relies on introspection rather than local JWT validation MUST treat a missing `act` claim in the introspection response as indicating a non-delegated token and MUST NOT assume delegation when the claim is absent.

Introspection endpoints for delegated tokens SHOULD be advertised via the `introspection_endpoint` parameter in AS metadata ({{RFC8414}}). When revocation is integrated, the introspection response for a revoked delegated token MUST return `"active": false` and MUST NOT include `act` or `sub_profile` claims.


# Transaction Tokens {#transaction-tokens}

## Overview

Transaction Tokens {{I-D.ietf-oauth-transaction-tokens}} are short-lived JWTs that capture the workload identity and request context for a series of related service calls within a single business transaction. They are issued by a Transaction Token Service (TTS), which is a specialized authorization server.

A Transaction Token contains:

`sub`:
: The identity of the human user or non-personal entity that originated the transaction.

`scope`:
: The transaction authorization scope, as defined by the TTS for the specific transaction.

`req_wl`:
: The workload identifier of the workload that requested the Transaction Token from the TTS.

`txn`:
: A transaction identifier that links all tokens in the same business transaction.

`tctx`:
: Transaction context information that remains stable across the Call Chain.

`rctx`:
: Request context information (e.g., originating IP, user-agent).


## Actor Claim in Transaction Tokens

When a Transaction Token is used in a delegated scenario, the `act` claim conforming to this profile SHOULD be included to represent the current acting party and any prior delegation steps.  In Transaction Tokens, `req_wl` identifies the workload that requested the token from the TTS, while `act.sub` identifies the immediate acting party in the subject identifier namespace used by this profile.

The semantics are as follows:

*  `sub` identifies the original human or non-personal initiator of the transaction and MUST NOT change as the token propagates through the call chain.

*  `scope` captures the transaction authorization intent for this specific token instance, using the semantics defined by the TTS.

*  `req_wl` identifies the workload that requested this specific token instance from the TTS.

*  When an `act` claim is present, the outermost `act.sub` identifies the immediate acting party that presents the token downstream.  In deployments where `req_wl` contains a single workload identifier for that same entity, the issuer SHOULD ensure that `req_wl` and the outermost `act.sub` are semantically consistent.

*  When a downstream workload exchanges a Transaction Token to obtain a new Transaction Token, the new token retains the original `sub`, updates `req_wl` to the new requester, and preserves prior presenter history by creating a new outermost `act` for the new presenter and nesting any existing `act` chain beneath it.

*  Inner nested `act` objects, when present, identify prior presenters in the transaction's delegation path.

*  `act.sub_profile` classifies the entity type of the current presenter, while nested `act.sub_profile` values classify prior presenters.

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
    "cnf": { "jkt": "0ZcOCORZNYy9ZhHiZN..." },
    "act": {
      "sub": "https://agents.example.com/travel-assistant",
      "iss": "https://as.enterprise.example",
      "sub_profile": "ai_agent",
      "cnf": { "jkt": "NzbLsXh8uDCcd7MNwrnNZpX0ak8ACQ" }
    }
  }
}
~~~
In this example the booking tool is the current presenter.  It is identified by `req_wl` as the workload that requested the token and by the outermost `act.sub` in the actor profile's subject namespace, and it has its own top-level `cnf.jkt`.  The travel assistant appears as a nested `act` object, showing that it was the prior delegated actor between Alice and the booking tool.


## Transaction Token Service Processing {#transaction-token-service-processing}

When a TTS receives a token-exchange request to issue or refresh a Transaction Token, and the inbound `subject_token` or `actor_token` contains actor-profile claims:

1.  The TTS MUST propagate the `sub` from the inbound token unchanged.

2.  The TTS MUST set `req_wl` to the requesting workload's identity, as authenticated during the token-exchange request.

3.  When the issued Transaction Token represents delegated access, the TTS MUST create a new outermost `act` object identifying the new presenter.  If the inbound token already carries an `act` chain, the TTS MUST nest that chain beneath the new outermost `act` object so that prior presenters are preserved.

4.  The TTS SHOULD include `sub_profile` in any new `act` object it creates, based on its knowledge of the new presenter's entity type.

5.  The TTS MUST set the top-level `cnf.jkt` to the key of the new authorized presenter, as demonstrated by the accompanying DPoP proof or mTLS certificate.

6.  The TTS MUST set the Transaction Token `scope` according to the authorization semantics defined by {{I-D.ietf-oauth-transaction-tokens}} for the requested transaction, and it MAY include `tctx` or `rctx` as appropriate for downstream services.

These same preservation rules apply regardless of whether the inbound credential is a JWT assertion grant, a JWT access token, or a Transaction Token, provided that the TTS supports issuing a Transaction Token from that input.


# Dual-Principal Authorization {#dual-principal-authorization}

## Authorization Model

When a token contains both `sub` and an `act` claim, a resource server has two independent principals available for authorization policy:

*  **Subject principal** (`sub`): the party whose authorization is being exercised.  This principal typically has a relationship with the resource (e.g., an account, a role, a permission).

*  **Actor principal** (`act.sub`): the party that is making the immediate request.  This principal may be in a different organizational domain and trust level from the subject.

Dual-principal authorization evaluates both principals and the relationship between them (i.e., that the actor is authorized to act on behalf of the subject) before granting access.

For Transaction Tokens, the primary policy pair remains (`sub`, `act.sub`).  The `req_wl` claim provides workload context from the TTS, while nested `act` objects provide prior-actor context for audit, policy refinement, or chain validation.

## Resource Server Processing {#dual-principal-rs-processing}

Resource servers MUST implement dual-principal authorization when the Protected Resource Metadata for the resource includes `dual_principal_authorization_supported: true` ({{protected-resource-metadata}}). Resource servers that do not advertise this capability SHOULD implement dual-principal authorization; omitting it is permitted only when the resource is consumed exclusively by internal, same-domain services whose delegation relationship is validated by separate means outside this profile.  The following steps describe the RECOMMENDED evaluation:

1.  **Evaluate subject authorization**: Determine whether the subject (`sub`) has been granted the requested scope or permission, using the same mechanisms applied to non-delegated tokens.

2.  **Evaluate presenter or actor authorization**: Determine whether the current non-subject principal is permitted to exercise the subject's authorization.  This principal is the outermost `act.sub`.  For Transaction Tokens, the RS SHOULD evaluate `req_wl` as supporting workload context and MAY verify semantic consistency between `req_wl` and the outermost `act.sub` when the deployment uses a single workload identifier for the current presenter.  This MAY be evaluated against:

    *  a registered delegation policy for the (subject, actor) pair,
    *  the actor's `sub_profile` (e.g., only AI agents from a trusted domain are permitted to act as delegatees),
    *  scope restrictions applied at the AS when the delegated token was issued.

3.  **Evaluate combined policy**: Apply any resource-specific dual-principal policies.  For example, a resource server may require that both the subject and the actor have independently agreed to terms of service.

If the resource server cannot evaluate actor authorization, it SHOULD reject the request unless local policy explicitly permits subject-only evaluation for that resource and scope.  Resource servers that fall back to subject-only evaluation of a delegated token SHOULD log an audit record describing that decision.

## Policy Claims Summary

The following claims are available as dual-principal policy inputs:

| Claim | Principal | Description |
|-------|-----------|-------------|
| `sub` | Subject | Original authorized principal |
| `sub_profile` | Subject | Entity type of the subject |
| `act.sub` | Actor | Immediate acting principal |
| `act.sub_profile` | Actor | Entity type of the actor identified by `act.sub` |
| `act.cnf.jkt` | Actor | Proof-of-possession key of the actor identified by `act.sub` |
| `scope` | Both | Authorized scope |
| `req_wl` (Txn-Token) | Actor | Workload that requested the Transaction Token from the TTS; MAY be used as supporting context alongside the outermost `act.sub` |

For Transaction Tokens, `req_wl` carries TTS-level workload context while the outermost `act.sub` identifies the current actor for authorization.  Nested `act` objects describe prior actors in the delegation path.


# Extension to OAuth Entity Profiles {#entity-profile-extension}

## Overview

{{I-D.mora-oauth-entity-profiles}} defines the OAuth Entity Profiles mechanism, including the `sub_profile` and `client_profile` JWT claims, the `entity_profiles_supported` AS metadata parameter, and an IANA registry of entity profile values with associated usage locations ("Subject Profile", "Client Profile").

This document extends that specification in one way:

1.  It defines a new "Actor Profile" usage location, indicating that an entity profile value is valid within an `act` object's `sub_profile` claim.

It does not define a separate registry.  All entity profile values used by this specification are registered in the "OAuth Entity Profiles" registry established by {{I-D.mora-oauth-entity-profiles}}.


## Actor Profile Usage Location {#actor-usage-location}

The "OAuth Entity Profiles" registry includes a "Usage Location" field that indicates where a profile value may appear.  This document adds a new usage location:

**Actor Profile**:
: The entity profile value appears in the `sub_profile` claim within an `act` object, classifying the entity identified by `act.sub`.  This usage location is defined by this document.

The following existing values from {{I-D.mora-oauth-entity-profiles}} are extended with the "Actor Profile" usage location:

| Entity Profile | Actor Context Description |
|----------------|--------------------------|
| `user` | A human principal acting as a delegated actor (e.g., impersonation within a system) |
| `service` | An automated backend service, tool, or microservice acting as an actor |
| `ai_agent` | An AI agent acting autonomously or on behalf of another principal |

These are the initial entity profile values that this document extends for actor use.  This document does not imply that all existing entity-profile values are automatically valid within `act.sub_profile`; only values explicitly registered with the "Actor Profile" usage location are valid in that position.  Additional values MAY be added to the registry in the future through the normal registration process when their semantics are appropriate for acting entities.

When processing `act.sub_profile`, issuers and consumers MUST treat these values according to the entity profile semantics defined in {{I-D.mora-oauth-entity-profiles, Section 3}}.

## Extension to entity_profiles_supported {#entity-profiles-actor-array}

{{I-D.mora-oauth-entity-profiles, Section 5}} defines the `entity_profiles_supported` AS metadata parameter as a JSON object containing `client` and `subject` arrays.  This document extends that object with an `actor` array:

`actor`:
: OPTIONAL.  A JSON array of entity profile values (per {{I-D.mora-oauth-entity-profiles, Section 3.3}}) indicating which profiles the AS recognizes and will validate when they appear in `act.sub_profile` claims in inbound tokens and actor assertions.  Actors whose `sub_profile` is not listed in this array MAY be rejected by the AS.

An AS implementing both this profile and {{I-D.mora-oauth-entity-profiles}} SHOULD include an `actor` array in its `entity_profiles_supported` object.

Example:

~~~json
{
  "entity_profiles_supported": {
    "client": ["service", "ai_agent"],
    "subject": ["user", "service", "ai_agent"],
    "actor":   ["user", "service", "ai_agent"]
  }
}
~~~

# Discovery and Capability Negotiation {#discovery-capability-negotiation}

## Overview

This section defines a minimal set of new metadata parameters that, together with the `entity_profiles_supported` extension in {{entity-profile-extension}}, allow authorization servers and resource servers to advertise actor-profile support without out-of-band configuration.

The design principle is that **capability flags go in this spec's metadata; entity type enumeration goes in {{I-D.mora-oauth-entity-profiles}} metadata**.  Clients MUST use `entity_profiles_supported.actor` ({{entity-profiles-actor-array}}) to determine which actor entity profiles an AS or RS accepts, and MUST use `actor_profile_token_types_supported` ({{authorization-server-metadata}}) to determine for which token types actor-profile processing applies.


## Authorization Server Metadata {#authorization-server-metadata}

One new parameter is defined for use in the AS metadata document ({{RFC8414}}):

`actor_profile_token_types_supported`:
: OPTIONAL.  A JSON array of token-type URI strings indicating the token types for which the AS applies actor-profile processing as defined in this document, whether as inbound token types, outbound token types, or both.  When a token type appears in this array, the AS supports actor-profile processing for that token type in at least one role.  This parameter alone does not distinguish inbound support from outbound support; clients therefore MUST combine it with profile-specific conventions, endpoint documentation, or prior knowledge of the AS before assuming that a particular transformation is supported.  Defined values are:

  - `urn:ietf:params:oauth:token-type:access_token` — JWT access tokens ({{jwt-access-tokens}})
  - `urn:ietf:params:oauth:token-type:jwt` — JWT assertion grants ({{jwt-assertion-grants}})
  - `urn:ietf:params:oauth:token-type:txn_token` — Transaction Tokens ({{transaction-tokens}})

  When absent, the AS makes no claim about actor-profile support.

The entity profile types the AS accepts for actors are advertised via the `entity_profiles_supported.actor` array defined in {{entity-profiles-actor-array}}, not via a separate metadata parameter.  DPoP support is advertised via `dpop_signing_alg_values_supported` per {{RFC9449}}.

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
  "actor_profile_token_types_supported": [
    "urn:ietf:params:oauth:token-type:access_token",
    "urn:ietf:params:oauth:token-type:jwt",
    "urn:ietf:params:oauth:token-type:txn_token"
  ],
  "entity_profiles_supported": {
    "client": ["service", "ai_agent"],
    "subject": ["user", "service", "ai_agent"],
    "actor":   ["user", "service", "ai_agent"]
  }
}
~~~

## Protected Resource Metadata {#protected-resource-metadata}

Two new parameters are defined for use in Protected Resource Metadata ({{RFC9728}}):

`actor_profile_required`:
: OPTIONAL.  A boolean.  When `true`, the RS requires that inbound tokens carrying an `act` claim conform to the actor profile defined in this document.  The RS MUST reject tokens that carry `act` claims not conforming to this profile.  When `false` or absent, actor-profile conformance is not required.

`dual_principal_authorization_supported`:
: OPTIONAL.  A boolean.  When `true`, the RS advertises that it evaluates both subject and actor principals for authorization policy as described in {{dual-principal-authorization}}.  When `false` or absent, the RS makes no declaration about whether it performs such evaluation, and clients MUST NOT rely on dual-principal authorization behavior unless the value is `true`.

Clients discover which actor entity profile values the RS's AS will accept by consulting `entity_profiles_supported.actor` in the AS metadata for the AS listed in the resource's `authorization_servers`.

Example Protected Resource Metadata fragment:

~~~json
{
  "resource": "https://api.travel-provider.example",
  "authorization_servers": [
    "https://as.travel-provider.example"
  ],
  "actor_profile_required": true,
  "dual_principal_authorization_supported": true
}
~~~

## Capability Negotiation Flow

The following steps describe a RECOMMENDED capability-negotiation pattern a client agent can use as a preflight check to evaluate whether a cross-domain delegation is likely to be feasible:

1.  **Discover target resource metadata**: The client fetches Protected Resource Metadata ({{RFC9728}}) for the target API and reads `actor_profile_required` and `authorization_servers`.

2.  **Discover target AS metadata**: The client fetches AS metadata ({{RFC8414}}) for the listed AS and reads:

    *  `actor_profile_token_types_supported` to determine whether actor-profile processing is advertised for the relevant token type.
    *  `entity_profiles_supported.actor` to confirm the client's own entity profile (its `sub_profile` value) is in the accepted list.

3.  **Proceed or abort**: If `actor_profile_required` is `true` at the RS and the client's entity profile is not listed in `entity_profiles_supported.actor` at the AS, the client MUST NOT proceed with a delegation-based request and SHOULD surface the capability mismatch to the invoking system.  If the listed metadata is insufficient to determine whether the AS supports the needed inbound and outbound token roles for the intended transformation, the client SHOULD treat support as indeterminate and rely on deployment-specific knowledge or a trial request.

4.  **Construct and submit token exchange or assertion grant**: The client proceeds per {{jwt-assertion-grants}} for JWT assertion grants or per {{RFC8693}} for token-exchange requests.  The `actor_token` or assertion MUST include the client's `sub_profile` value from the accepted list.

5.  **RS validation**: The RS validates the resulting token according to the rules for that token type and applies dual-principal authorization per {{dual-principal-authorization}}.  For JWT access tokens, see {{jwt-access-token-rs-processing}}.  For Transaction Tokens, see {{I-D.ietf-oauth-transaction-tokens}} together with {{transaction-tokens}} of this document.


## Transaction Token Flows

Transaction Token support is advertised using the `transaction_token_supported` AS metadata parameter and the `transaction_token_required` Protected Resource Metadata parameter, both defined in {{I-D.ietf-oauth-transaction-tokens, Section 8}}. This document does not redefine those parameters.

When Transaction Tokens are used, the actor-profile claims MUST be propagated per {{transaction-token-service-processing}}, and the AS MUST list `urn:ietf:params:oauth:token-type:txn_token` in `actor_profile_token_types_supported`.


# Security Considerations

## Delegation Chain Integrity

The `act` claim MUST be set by a trusted AS that has validated the delegation relationship.  Resource servers MUST NOT accept tokens in which the `act` claim was demonstrably set by an untrusted party. Concretely, RS implementations MUST validate the token signature before extracting actor claims, and MUST verify that the issuer of the token is authoritative for the claims it contains.

Because inner `act` objects are set by upstream ASes and not re-signed at each hop, the integrity of the entire delegation chain rests on the outermost token's signature.  Implementations SHOULD use short token lifetimes and MUST reject tokens whose `exp` has passed, regardless of chain depth.

## Actor Identity Binding

Without actor proof of possession, a leaked token can be replayed by any party.  When an `act.cnf.jkt` is present:

*  The AS that set this value SHOULD have verified the actor's possession of the corresponding key before including it.
*  The RS SHOULD require a DPoP proof for the top-level `cnf.jkt` and SHOULD verify that the top-level confirmation information is consistent with the outermost actor's binding when the outermost `act` object also carries a `cnf` claim.

Deployments of AI agent systems SHOULD require actor key binding to prevent delegation token theft.

## Delegation Depth Limits

Unbounded delegation chains increase attack surface and complicate policy evaluation.  This document specifies a maximum depth of eight nested `act` objects ({{delegation-chains}}).  Implementations that encounter chains exceeding this limit MUST reject the token to prevent denial-of-service through chain parsing.

## Sub_profile Trust

The `sub_profile` claim is asserted by the token issuer and is only as trustworthy as that issuer.  Resource servers MUST NOT trust `sub_profile` values in tokens issued by untrusted parties.  Resource server operators SHOULD configure a list of accepted entity-type profiles per trust domain.

## Cross-Domain Trust

When a token crosses organizational boundaries, the receiving AS or RS MUST apply appropriate trust evaluation.  The `iss` claim of the outer token identifies the issuing AS; inner `act` objects may reference actors whose `sub` is from a different domain.  ASes performing token exchange MUST evaluate cross-domain delegation grants explicitly and MUST NOT implicitly grant cross-domain actors the same rights as same-domain actors.

## Delegation Revocation

Token revocation ({{RFC7009}}) applies to individual tokens but does not revoke an underlying delegation relationship or invalidate already-issued downstream tokens in a delegation chain.  When Alice revokes her delegation to an agent, access tokens already issued to downstream actors remain valid until their `exp` time.  Short token lifetimes are the primary mitigation.

Authorization servers SHOULD maintain a delegation grant registry mapping (subject, actor) pairs to active delegation relationships. When a delegation grant is revoked, the AS MUST refuse to issue new tokens for that (subject, actor) pair and SHOULD proactively revoke any outstanding refresh tokens associated with that delegation.

Resource servers in security-sensitive deployments SHOULD use token introspection ({{RFC7662}}, {{token-introspection}}) rather than local JWT validation so that revocation state is checked on each request. Implementations MUST NOT use delegation chain depth as a rationale for skipping revocation checks.

Operators deploying AI agent systems MUST provide end-users with a mechanism to enumerate and revoke active delegation grants.


## Confused Deputy and Dual-Principal Bypass

A resource server that evaluates only the subject principal when an `act` claim is present is susceptible to a confused deputy attack where a malicious actor exploits a subject's pre-existing permissions without the subject's ongoing consent.  Resource servers that advertise `dual_principal_authorization_supported: true` MUST implement dual-principal authorization per {{dual-principal-authorization}}.  Other resource servers SHOULD implement it as well.  New deployments MUST NOT intentionally omit dual-principal authorization except under the conditions stated in {{dual-principal-rs-processing}}.

## Token Substitution

An attacker who can present a token with a crafted `sub_profile` or actor chain could attempt to escalate privileges.  ASes MUST validate inbound `sub_profile` values against the registered values and their own policy before including them in issued tokens.


# Privacy Considerations {#privacy}

Delegation chains can reveal sensitive information about user behavior, enterprise topology, software suppliers, and internal tool composition. Issuers therefore SHOULD disclose only the actor information needed by the relying party for authorization, audit, or policy enforcement.

Cross-domain deployments SHOULD prefer stable but non-reassigned identifiers and SHOULD consider pairwise identifiers for human subjects when a globally correlatable identifier is not required by the use case.

When the same logical entity can appear in different identifier namespaces, such as `azp`, `req_wl`, and `act.sub`, issuers and relying parties SHOULD use explicit issuer scoping and locally trusted mapping rules rather than string equality alone to determine whether those identifiers refer to the same entity.

Actor chains SHOULD be truncated when prior actors are not relevant to the recipient's policy decision.  Where truncation would change the security semantics of the token, the issuer SHOULD reject the request rather than silently omit required chain elements.

The `txn` claim in Transaction Tokens ({{I-D.ietf-oauth-transaction-tokens}}) is a stable, globally unique identifier shared across all tokens in a single business transaction.  When Transaction Tokens cross organizational boundaries, `txn` enables cross-domain correlation of all service calls within a transaction by any party that observes multiple tokens.  Transaction Token Services SHOULD NOT propagate the same `txn` value into tokens presented to resource servers outside the originating trust domain unless that relying party is explicitly authorized to correlate the transaction.  Where cross-domain correlation is not required for the authorization decision, the TTS SHOULD derive a domain-scoped transaction identifier that cannot be linked back to the originating `txn` value.  Implementations MUST treat `txn` values as sensitive identifiers and MUST NOT include them in logs or audit records accessible to parties outside the delegation chain without the subject's consent.

The `req_wl` claim in Transaction Tokens can also expose sensitive information about internal workload topology and service composition.  Transaction Token Services SHOULD disclose `req_wl` only to relying parties that need that information for authorization, audit, or policy enforcement, and SHOULD avoid propagating internal-only workload identifiers across trust-domain boundaries unless such disclosure is explicitly required by the deployment.


# IANA Considerations

## OAuth Authorization Server Metadata Registry

This document requests IANA to register the following value in the "OAuth Authorization Server Metadata" registry ({{RFC8414}}):

*  Metadata Name: `actor_profile_token_types_supported`
*  Metadata Description: JSON array of token-type URIs for which the AS applies actor-profile processing per this document
*  Change Controller: IETF
*  Reference: {{authorization-server-metadata}} of this document


## OAuth Protected Resource Metadata Registry

This document requests IANA to register the following values in the "OAuth Protected Resource Metadata" registry ({{RFC9728}}):

*  Metadata Name: `actor_profile_required`
*  Metadata Description: Boolean indicating whether the RS requires actor-profile conformance for tokens carrying `act` claims
*  Change Controller: IETF
*  Reference: {{protected-resource-metadata}} of this document

*  Metadata Name: `dual_principal_authorization_supported`
*  Metadata Description: Boolean indicating whether the RS performs dual-principal authorization per this document
*  Change Controller: IETF
*  Reference: {{protected-resource-metadata}} of this document


## OAuth Entity Profiles Registry

This document requests the following updates to the "OAuth Entity Profiles" registry established by {{I-D.mora-oauth-entity-profiles}}.

### New Usage Location: Actor Profile

This document defines the "Actor Profile" usage location for the "OAuth Entity Profiles" registry.  A value registered with this usage location is valid as a `sub_profile` value within an `act` object in a JWT, classifying the entity identified by `act.sub`.

The Designated Experts for this registry SHOULD verify that any value submitted for "Actor Profile" usage describes a type of acting entity relevant to OAuth delegation scenarios.

### Updated Registry Entries

The following existing registry entries are updated to add the "Actor Profile" usage location:

*  Entity Profile Name: `user`
*  Added Usage Location: "Actor Profile"
*  Reference: {{actor-usage-location}} of this document

*  Entity Profile Name: `service`
*  Added Usage Location: "Actor Profile"
*  Reference: {{actor-usage-location}} of this document

*  Entity Profile Name: `ai_agent`
*  Added Usage Location: "Actor Profile"
*  Reference: {{actor-usage-location}} of this document

## Extension to entity_profiles_supported Metadata

Upon publication of {{I-D.mora-oauth-entity-profiles}} as an RFC, this document requests that the definition of the `entity_profiles_supported` AS metadata parameter ({{I-D.mora-oauth-entity-profiles, Section 5}}) be updated to allow an optional `actor` member alongside the existing `client` and `subject` members, as defined in {{entity-profiles-actor-array}} of this document.  Until that publication, implementations SHOULD treat the `actor` array extension as defined in {{entity-profiles-actor-array}} as a vendor-agreed extension to the base parameter.



--- back

# Cross-Domain AI Agent Flow: ID Token to Transaction Token {#appendix-cross-domain}

This appendix traces a single user request across two trust domains, highlighting the actor-profile claim structures and processing requirements specific to this specification.  Standard validation steps (JWT signature verification, DPoP proof binding, Token Exchange mechanics) are delegated to {{RFC7519}}, {{RFC9449}}, and {{RFC8693}} respectively.

All claim values, JKT thumbprints, and domain names are synthetic.

## Scenario and Parties

Alice's travel-assistant agent authenticates to the Enterprise IdP AS to obtain an ID Token.  The agent then performs a Token Exchange at the same AS to obtain the ID-JAG.  The ID-JAG is then presented to the Travel Provider AS using the JWT bearer grant, as described in {{jwt-assertion-grants}}.  The agent exchanges the ID-JAG for an access token at the Travel Provider AS and calls the Booking Tool API.  The Booking Tool exchanges the access token for a Transaction Token to call an internal inventory service.

~~~
Enterprise domain                 Travel Provider domain
────────────────────────────      ──────────────────────────────────────
Alice
  │ (1) authenticates
  ▼
Enterprise IdP AS ─► ID Token
  │ (2) Token Exchange (ID Token → ID-JAG)
  ▼
Enterprise IdP AS ─► ID-JAG
                       │ (3) JWT Bearer Grant (ID-JAG → AT)
                       └─────────────────► Travel Provider AS ─► AT
                                                                   │
                                            Travel Assistant ◄─────┘
                                                 │ (4) Access Token + DPoP
                                                 ▼
                                            Booking Tool API (RS)
                                                 │ (5) Token Exchange (AT → Transaction Token)
                                                 ▼
                                            Travel Provider TTS ─► Transaction Token
                                                 │ (6) Transaction Token + DPoP
                                                 ▼
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

DPoP key bindings:

| Principal | JWK Thumbprint (`jkt`) |
|-----------|------------------------|
| Travel Assistant | `AgentJKT-NzbLsXh8uDCcd7MN` |
| Booking Tool | `ToolJKT-0ZcOCORZNYy9ZhHi` |


## Discovery and Capability Negotiation

The agent checks the Travel Provider AS metadata ({{discovery-capability-negotiation}}) to confirm actor-profile support before initiating the flow:

~~~json
{
  "issuer": "https://as.travel-provider.example",
  "actor_profile_token_types_supported": [
    "urn:ietf:params:oauth:token-type:jwt",
    "urn:ietf:params:oauth:token-type:access_token",
    "urn:ietf:params:oauth:token-type:txn_token"
  ],
  "entity_profiles_supported": {
    "subject": ["user", "ai_agent"],
    "actor":   ["user", "ai_agent", "service"]
  }
}
~~~
The agent confirms its `sub_profile` (`ai_agent`) is listed in `entity_profiles_supported.actor` and that both `jwt` and `access_token` are covered by `actor_profile_token_types_supported`.  The Booking Tool RS metadata signals `"actor_profile_required": true` and `"dual_principal_authorization_supported": true`.


## Step 1: User Authentication — ID Token

Alice authenticates to the Enterprise IdP AS.  The profile-relevant claim is `sub_profile`, which propagates Alice's entity type into downstream tokens:

~~~json
{
  "iss": "https://as.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
  "aud": "travel-assistant-client-id",
  "exp": 1743379200,
  "iat": 1743375600
}
~~~

## Step 2: Enterprise Token Exchange — ID Token to ID-JAG

The agent presents Alice's ID Token and a self-signed actor JWT to the Enterprise IdP AS.  The actor JWT carries the agent's `sub_profile` and DPoP key binding per {{jwt-assertion-grants-structure}}:

~~~json
{
  "iss": "https://agents.enterprise.example/travel-assistant",
  "sub": "https://agents.enterprise.example/travel-assistant",
  "sub_profile": "ai_agent",
  "aud": "https://as.enterprise.example/token",
  "jti": "actor-jwt-20260401-001",
  "exp": 1743375660,
  "iat": 1743375600,
  "cnf": { "jkt": "AgentJKT-NzbLsXh8uDCcd7MN" }
}
~~~

~~~
POST /token HTTP/1.1
Host: as.enterprise.example
Content-Type: application/x-www-form-urlencoded
DPoP: <AgentJKT-proof>

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token=<alice-id-token>
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aid_token
&actor_token=<agent-actor-jwt>
&actor_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Ajwt
&requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aid-jag
&audience=https%3A%2F%2Fas.travel-provider.example%2F
&resource=https%3A%2F%2Fas.travel-provider.example
&scope=booking%3Acreate
~~~

The Enterprise IdP AS applies scope reduction and validates the actor information according to the output-token rules for the JWT it is issuing.  In this example, it verifies DPoP possession against `cnf.jkt` in the actor JWT ({{jwt-assertion-grants-processing}}) and issues the ID-JAG as a JWT output of token exchange with the actor chain established per {{actor-profile}}:

~~~json
{
  "iss": "https://as.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
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
    "sub_profile": "ai_agent",
    "cnf": { "jkt": "AgentJKT-NzbLsXh8uDCcd7MN" }
  }
}
~~~

The `act` object records the agent as the authorized actor.  Both `cnf.jkt` (top-level, for DPoP binding) and `act.cnf.jkt` (actor key reference) are set to `AgentJKT` because the agent is both the bearer and the acting principal at this stage.


## Step 3: Agent Exchanges ID-JAG for Access Token at Travel Provider AS

The agent presents the ID-JAG as a JWT Bearer authorization grant ({{RFC7523}}) to the Travel Provider AS.  In this usage, the ID-JAG functions as a profiled JWT assertion grant, so the processing rules in {{jwt-assertion-grants}} apply to it:

~~~
POST /token HTTP/1.1
Host: as.travel-provider.example
Content-Type: application/x-www-form-urlencoded
DPoP: <AgentJKT-proof>

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
&assertion=<id-jag>
&scope=booking%3Acreate
~~~

The Travel Provider AS performs actor-profile processing per {{jwt-assertion-grants-processing}}: it verifies the `act.cnf.jkt` DPoP binding and checks that `act.sub_profile` (`ai_agent`) is permitted as an actor for the requested scope.  It issues an access token preserving the actor chain:

~~~json
{
  "iss": "https://as.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
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
    "sub_profile": "ai_agent",
    "cnf": { "jkt": "AgentJKT-NzbLsXh8uDCcd7MN" }
  }
}
~~~

Alice's `sub` and `sub_profile` are preserved verbatim from the ID-JAG ({{jwt-access-token-propagation}}).  The Travel Provider AS does not translate or substitute the enterprise subject identifier.


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

The Booking Tool RS applies dual-principal authorization ({{dual-principal-authorization}}): it evaluates both Alice (`sub`, `sub_profile: user`) and the Travel Assistant (`act.sub`, `sub_profile: ai_agent`). Because the RS advertises `dual_principal_authorization_supported: true`, both principals MUST be independently authorized.  The `act.sub_profile` value is checked against `entity_profiles_supported.actor` per {{entity-profile-extension}}.


## Step 5: Booking Tool Exchanges Access Token for Transaction Token

The Booking Tool cannot reuse the received access token for internal calls: it is DPoP-bound to `AgentJKT`, which the Booking Tool does not possess.  It requests a Transaction Token from the TTS:

~~~
POST /token HTTP/1.1
Host: tts.travel-provider.example
Content-Type: application/x-www-form-urlencoded
Authorization: PrivateKeyJWT <booking-tool-assertion>
DPoP: <ToolJKT-proof>

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token=<tp-access-token>
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aaccess_token
&requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Atxn_token
&audience=https%3A%2F%2Finternal.travel-provider.example%2Finventory
&rctx={"req_ip":"198.51.100.42","req_time":1743375650}
~~~

The TTS applies actor-profile processing per {{transaction-token-service-processing}}: it preserves `sub` and `sub_profile` from the subject token, sets `req_wl` to the authenticated Booking Tool, creates a new outermost `act` object for the Booking Tool, nests the subject token's existing `act` claim beneath it, and re-binds `cnf.jkt` to the Booking Tool's DPoP key (`ToolJKT`):

~~~json
{
  "iss": "https://tts.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
  "scope": "inventory:check",
  "req_wl": "https://tools.travel-provider.example/booking-tool",
  "aud": "https://internal.travel-provider.example/inventory",
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
  "rctx": { "req_ip": "198.51.100.42", "req_time": 1743375650 },
  "cnf": { "jkt": "ToolJKT-0ZcOCORZNYy9ZhHi" },
  "act": {
    "sub": "https://tools.travel-provider.example/booking-tool",
    "iss": "https://as.travel-provider.example",
    "sub_profile": "service",
    "cnf": { "jkt": "ToolJKT-0ZcOCORZNYy9ZhHi" },
    "act": {
      "sub": "https://agents.enterprise.example/travel-assistant",
      "iss": "https://as.enterprise.example",
      "sub_profile": "ai_agent",
      "cnf": { "jkt": "AgentJKT-NzbLsXh8uDCcd7MN" }
    }
  }
}
~~~

The DPoP key rotates at this step: `cnf.jkt` is now `ToolJKT`, and the outermost `act.cnf.jkt` matches it because the Booking Tool is now the current actor.  The nested `act.act.cnf.jkt` retains the agent's original key, illustrating the multi-hop key-rotation property described in {{sender-constraint}}.


## Step 6: Booking Tool Calls Inventory Service

~~~
GET /inventory?origin=SFO&dest=NYC&depart=2026-04-15 HTTP/1.1
Host: internal.travel-provider.example
Txn-Token: <txn-token>
DPoP: <ToolJKT-proof>
~~~

The Inventory Service applies dual-principal authorization ({{dual-principal-authorization}}): Alice (`sub`) governs data access policy (e.g., travel tier); the Booking Tool (`act.sub`) is the authorized internal workload.  The `req_wl` claim provides consistent TTS workload context for the same service in this example.  The nested `act.act.sub` (Travel Assistant) is carried as prior delegation context and is not evaluated for access control at this internal tier, consistent with the guidance on inner actors in {{dual-principal-rs-processing}}.


## Summary of Token Transformations

| Step | Token | `sub` | `req_wl` | `act.sub` (outermost) | `act.act.sub` (nested) | `cnf.jkt` |
|------|-------|-------|----------|-----------------------|------------------------|-----------|
| 1 | ID Token | Alice | — | — | — | — |
| 2 | ID-JAG | Alice | — | Travel Assistant | — | AgentJKT |
| 3 | Access Token | Alice | — | Travel Assistant | — | AgentJKT |
| 4 | (API call) | Alice | — | Travel Assistant | — | AgentJKT |
| 5 | Transaction Token | Alice | Booking Tool | Booking Tool | Travel Assistant | ToolJKT |
| 6 | (internal call) | Alice | Booking Tool | Booking Tool | Travel Assistant | ToolJKT |

Key observations:

*  `sub` (Alice) is unchanged across all trust domains and token transformations.
*  The DPoP key rotates exactly once — at Step 5, when the TTS re-binds the Transaction Token to the Booking Tool's key.
*  At Step 5 the TTS creates a new outermost `act` for the Booking Tool and nests the prior `act` chain beneath it, so the Travel Assistant's identity and key are preserved in `act.act.sub` / `act.act.cnf.jkt`.


# Acknowledgments
{:numbered="false"}

The author thanks the OAuth Working Group for the foundational work on Token Exchange (RFC 8693), JWT-formatted access tokens (RFC 9068), DPoP (RFC 9449), and Transaction Tokens that this document builds upon. The motivating use cases for this work emerged from the deployment of AI agent systems that require cross-domain delegation with explicit, auditable delegation chains.

---
title: "OAuth Actor Profile for Delegation"
abbrev: "OAuth Actor Profile"
category: std
docname: draft-mcguinness-oauth-actor-profile-latest
submissiontype: IETF
number:
date: 2026-04-15
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
  RFC9493:
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
     -
        fullname: Karl McGuinness
        organization: Independent
    date: 2026-04-15
    target: https://www.ietf.org/archive/id/draft-mora-oauth-entity-profiles-01.txt

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
  I-D.ietf-wimse-workload-creds:
  I-D.ietf-wimse-wpt:
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

OAuth deployments increasingly involve multi-principal scenarios where an agent or workload acts on behalf of a human user across organizational and trust-domain boundaries.  Existing specifications provide relevant building blocks but do not define a common profile for representing the delegated actor relationship across token types, classifying actor entity types, or signaling support between authorization servers and resource servers.  The result is inconsistent actor representation and interoperability gaps that force deployments to rely on proprietary conventions.

This document defines the OAuth Actor Profile for Delegation.  The design center is delegation clarity: `client_id` identifies the OAuth client registration, `sub` identifies the authorizing principal, and `act.sub` identifies the actor exercising that authorization.  These are distinct concepts that this profile makes explicit in the token rather than leaving them to be inferred from client registration context.  The profile defines a common `act` claim structure with `sub_profile` for machine-processable entity classification and `cnf` for actor-associated key context, applies uniformly across JWT assertion grants, JWT access tokens, and Transaction Tokens, and specifies processing rules for authorization servers and resource servers.  The document also registers metadata parameters for advertising and negotiating actor-profile support in cross-domain deployments.  This document standardizes the token representation of delegation relationships and the processing rules for issuers and consumers; it does not standardize the upstream policies and mechanisms by which systems determine whether a given actor is authorized to act for a subject, which remain deployment-specific.


--- middle

# Introduction

OAuth deployments increasingly involve multi-principal scenarios where an agent or workload acts on behalf of a human user across organizational boundaries.  Existing specifications provide relevant building blocks — {{RFC8693}} introduced the `act` claim for token exchange — but do not define a consistent, interoperable way to represent delegated actor relationships across JWT assertion grants, JWT access tokens, and Transaction Tokens.  The resulting gaps are described in {{motivation-and-interoperability-gaps}}; in brief, deployments rely on proprietary conventions and resource servers cannot reliably determine which principal authorized a request, which actor is presenting it, or whether a delegation chain should be trusted across trust-domain boundaries.

The design center of this document is delegation clarity.  `client_id` identifies the OAuth client registration, `sub` identifies the authorizing principal, and `act.sub` identifies the actor exercising that authorization.  These are distinct concepts.  This profile makes the actor explicit in the token rather than leaving it to be inferred from client registration context, and it does so without redefining client identity or top-level subject semantics.

This document addresses that gap by specifying:

*  A common actor profile structure that reuses `act` from {{RFC8693}} and adds `sub_profile` for entity-type classification and `cnf` for actor-associated key context.
*  How the actor profile is expressed and validated in each token type: JWT assertion grants ({{jwt-assertion-grants}}), JWT access tokens ({{jwt-access-tokens}}), and Transaction Tokens ({{transaction-tokens}}).
*  How actor-profile information is preserved across supported token transformations.
*  Resource-server actor-authorization guidance ({{dual-principal-authorization}}), recommending that resource servers enforce policy over the (subject, actor) pair and define when they require actor evaluation.
*  Use of the OAuth Entity Profiles actor support ({{entity-profile-extension}}) defined in {{I-D.mora-oauth-entity-profiles}}, so existing entity classifications can be used consistently in actor position.
*  Discovery metadata and capability-negotiation procedures ({{discovery-capability-negotiation}}) for authorization servers that offer Token Exchange or Transaction Token issuance, and for resource servers that consume delegated tokens.

The mechanisms are general-purpose and apply beyond AI agent scenarios.  This document is a profile and extension of existing OAuth building blocks; unless stated otherwise, the requirements of {{RFC8693}}, {{RFC9068}}, {{RFC9449}}, and {{I-D.ietf-oauth-transaction-tokens}} continue to apply.


## Illustrative Use Case

Alice authorizes an AI agent to book a business trip on her behalf; the agent calls an external booking tool operated in a separate organizational trust domain.  The flow is:

1.  Alice authenticates at her enterprise identity provider authorization server (Enterprise IdP AS), and the agent obtains an ID Token establishing Alice's identity.  This authentication step is out of scope for this document; the processing described here begins when the Enterprise IdP AS receives the upstream credential.
2.  The agent exchanges the ID Token at the Enterprise IdP AS to obtain an Identity Assertion JWT Authorization Grant (ID-JAG): a JWT produced by token exchange that carries Alice's identity, the agent's actor profile, and an audience bound to the external tool AS's token endpoint.  When the agent later presents that JWT to another AS using the JWT bearer grant, the JWT functions as a profiled JWT assertion grant.
3.  The agent submits the ID-JAG as a JWT bearer authorization grant to the external tool's AS token endpoint.  The tool AS validates the enterprise chain and issues an access token for the external tool's API.
4.  The agent calls the external tool's API using that access token.
5.  The external tool exchanges the received access token at its Transaction Token Service (TTS) to obtain a Transaction Token for a backend internal service call, with the issued token rebound to the tool as the new presenter under the Transaction Token deployment's proof mechanism rather than the agent's.
6.  The tool calls the internal service using the Transaction Token.

At each step, Alice's identity is preserved as `sub`, the agent's identity is preserved in `act`, the current presenter holds a key-bound token, and the trust-domain boundary enforcer (the tool AS) re-issues the credential under local control.

## Relationship to Related Work

**OAuth Token Exchange ({{RFC8693}})**: This document profiles the `act` claim from {{RFC8693}}.  The actor object structure and processing rules supplement, and do not replace, the base Token Exchange requirements.

**Identity Chaining ({{I-D.ietf-oauth-identity-chaining}})**: Identity Chaining addresses cross-domain subject-identity propagation; this document addresses actor representation within those same tokens.  The two are complementary and designed to be used together.

**Identity Assertion JWT Authorization Grant ({{I-D.ietf-oauth-identity-assertion-authz-grant}})**: An ID-JAG carrying an `act` claim is processed per {{jwt-assertion-grants}} of this document.  See {{appendix-cross-domain}} for an end-to-end example.

**OAuth Entity Profiles ({{I-D.mora-oauth-entity-profiles}})**: Defines `sub_profile`, `client_profile`, `entity_profiles_supported`, and the entity profile registry.  This document consumes those mechanisms for actor classification and makes no independent registry requests.

**Transaction Tokens ({{I-D.ietf-oauth-transaction-tokens}})**: This document extends the Transaction Token claim model with actor-profile support and adds TTS processing rules.  Base Transaction Token requirements continue to apply.

**WIMSE Workload Identity ({{I-D.ietf-wimse-workload-creds}}{{I-D.ietf-wimse-wpt}})**: Defines workload credentials used to authenticate workloads at token endpoints.  This document is mechanism-agnostic; {{appendix-cross-domain}} illustrates a WIMSE-based TTS presenter binding.


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
: A delegation scenario in which `sub` and `act.sub` are governed by different issuer namespaces.  In this document, a token is considered cross-domain when `act.iss` (the authority for the actor's identifier namespace, as defined in {{actor-profile}}) differs from the token's top-level `iss` (the authority for the token itself).  Note that the token `iss` identifies the AS that issued the token, not necessarily the issuer of the subject's original credential; deployments where the subject's identity domain and the token-issuing AS domain differ should be aware that the token `iss` is used here as the practical proxy for the subject's issuer namespace.

Actor Authorization at the Resource Server:
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

Outermost Actor:
: The `act` object at the top level of the delegation chain — the one not nested inside any other `act` object.  When a delegation chain of depth greater than one is present, the outermost actor identifies the immediate bearer of the token.

Delegation Relationship:
: A trust relationship indicating that a subject has authorized a specific actor to exercise the subject's rights within defined scope limits.  A delegation relationship may be expressed by a pre-registered delegation grant, an explicit consent record, a policy rule naming both parties, or an inbound assertion grant that the receiving AS has validated.

Local Policy:
: Deployment-specific rules, configurations, or decisions made by an individual AS, RS, or organization that are not specified by this document.  Local policy MAY include delegation approval rules, scope-reduction algorithms, actor-identifier namespace mappings, and entity-profile acceptance criteria.  When this document references local policy, the specific decision logic is intentionally not standardized.

Semantic Consistency:
: Two identifiers or claims are semantically consistent when they refer to the same logical entity under trusted local mapping rules applicable to the context in which they appear.  When this document states that an AS or RS MUST verify semantic consistency between two identifiers, the AS or RS MUST apply its configured mapping rules to determine whether the identifiers are known to refer to the same entity; absent any applicable mapping rule, the identifiers MUST be treated as distinct.

Examples in this document are illustrative and focus on actor-profile-related claims and processing.  They may omit unrelated claims, parameters, or validation steps required by the underlying specifications for a complete deployment.


# Motivation and Interoperability Gaps

## Overloaded Subject Semantics

The `sub` claim of a JWT access token {{RFC9068}} is routinely overloaded to represent heterogeneous entity types: an end-user, a service account, a workload identifier, or an AI agent.  Resource servers must rely on out-of-band conventions or proprietary claim extensions to determine which type of entity `sub` refers to.  This ambiguity prevents deterministic cross-domain policy evaluation.  This gap is addressed by the `sub_profile` claim defined in {{actor-profile-claims-summary}} and the entity profile values defined in {{entity-profile-extension}}.

## Inconsistent Actor Representation

{{RFC8693}} defines the `act` claim for tokens obtained via token exchange but does not extend its semantics to JWT assertion grants or JWT access tokens issued by other means.  Deployments that do not use token exchange have no standard location for actor information.  Even when `act` is present, there is no standard sub-claim that identifies the entity type of the actor.

Many existing deployments rely on implicit delegation: the OAuth client identity (`client_id`, `azp`, or authenticated client context at the token endpoint) is treated as evidence of the acting party, and authorization policy infers delegation from the fact that the client obtained or presented the token.  That approach works within tightly controlled deployments but has three structural limitations that prevent it from scaling to multi-domain, multi-actor environments.

`client_id` conflates distinct identity concepts.
: The `client_id` identifies an OAuth client registration — the software component authorized to interact with the AS.  In practice it is repurposed as a stand-in for the acting party, conflating OAuth client identity (the registered software component), runtime actor identity (the concrete instance or operational principal executing the request), and delegated execution identity (the principal holding a delegation grant from the authorizing subject).  These identities collapse in simple deployments but diverge in shared-client deployments, backend-for-frontend patterns, multi-tier architectures, and agent-invocation scenarios.  When they diverge, the protocol has no explicit mechanism to express the difference, and authorization correctness, audit fidelity, and security analysis all depend on deployment-specific conventions that cannot be verified from the token alone.

A single `client_id` may front multiple distinct actors.
: An agent orchestration platform, a plugin host, or a multi-tenant automation service may operate as one registered client while executing requests on behalf of many different agents, tools, or workloads.  A resource server receiving two requests that share the same `client_id` cannot determine whether they originate from the same logical actor or from entirely different principals.  Authorization policy, rate limiting, audit logging, and anomaly detection all operate on the wrong granularity when the acting entity is not explicitly represented in the token.

Actor context does not survive token transformation.
: When a token is exchanged or minted by an intermediary AS, client context from the original request is not carried forward.  Each downstream consumer sees only what the issuing AS encoded — in the absence of a common actor profile, typically just the subject and the issuer.  Downstream services cannot apply actor-aware authorization policy, produce accurate audit records, or perform meaningful security analysis across a delegation chain.

This specification addresses these limitations through explicit actor modeling: `client_id` identifies the OAuth client registration (unchanged), `sub` identifies the authorizing principal, and `act.sub` identifies the actor exercising that authorization.  These three identities can refer to different parties and must not be conflated.  Authenticated client identity remains a valid authorization input, but it is auxiliary to, not a substitute for, explicit actor identity.  The normative rules are in {{client-identity-delegation}}; reconciliation and migration rules are in {{migration-implicit-explicit}}; the requirement that actor context survive token transformation is in {{jwt-access-token-propagation}}.

## Missing Actor-Associated Key Context

Sender-constrained tokens {{RFC9449}}{{RFC8705}} bind the token to a proof-of-possession key identified in the top-level `cnf` claim.  When delegation is present, the immediate presenter is the actor, not the original subject.  There is no standard location for recording key material associated with the acting party within the `act` claim itself.  Without such a location, implementations have no common way to carry actor-key context for audit, diagnostics, or optional local policy as delegation chains pass through multiple ASes.  This gap is addressed by the `cnf` claim in the actor object defined in {{actor-object-structure}} and the sender-constraint rules in {{sender-constraint}}.  Note that `act.cnf` is actor-associated key history for audit and optional local policy; the active proof-of-possession requirement for the current presenter is always carried by the top-level `cnf` claim.

## No Cross-Token Profile Consistency

JWT assertion grants, JWT access tokens, and Transaction Tokens are specified in different documents with different claim conventions.  There is no common profile that spans all three and specifies how actor information flows from an assertion presented to an AS through to the access token issued by that AS and into any downstream Transaction Token.  This gap is addressed by the common actor profile defined in {{actor-profile}}, with token-type-specific rules in {{jwt-assertion-grants}}, {{jwt-access-tokens}}, and {{transaction-tokens}}.

## Absent Discovery and Capability Negotiation

Neither AS metadata {{RFC8414}} nor Protected Resource Metadata {{RFC9728}} define parameters for advertising actor-profile support.  Deployments therefore require bilateral out-of-band configuration, which is impractical for AI agents that dynamically discover and invoke tools across organizational boundaries.  This gap is addressed by the metadata parameters defined in {{discovery-capability-negotiation}}.


# Actor Profile for Delegation {#actor-profile}

## Overview

This profile specifies an extended form of the `act` claim defined in {{RFC8693}}.  When an implementation elects to use this profile in a context where an actor is distinct from the subject, it MUST apply the profile as defined in this section.

The actor profile is applicable to:

*  JWT assertion grants ({{jwt-assertion-grants}})
*  JWT-formatted access tokens ({{jwt-access-tokens}})
*  Transaction Tokens ({{transaction-tokens}})

Subject to endpoint policy and the underlying token-exchange or grant mechanism, implementations MAY transform any supported input token type into any other supported output token type.  When they do so, actor profile information MUST be preserved and validated as specified in this document for the resulting token type.

For a complete worked example showing the actor profile in use across all three token types in a cross-domain flow, see {{appendix-cross-domain}}.

## Scope of This Profile

This document standardizes actor-profile claim structure, processing rules, and discovery metadata.  It does not standardize delegation approval policy, trust framework decisions, or identifier-mapping logic; those remain deployment-specific.  The absence of an explicit actor-carrying inbound credential MUST NOT be interpreted as meaning that the OAuth client automatically defines the delegated actor.

## Implementation Summary

The following table summarizes the minimum implementation obligations by role:

| Role | Inputs | Minimum checks | Outputs / behavior |
|------|--------|----------------|--------------------|
| Assertion-consuming AS | JWT assertion grant with `act` | Validate JWT; trust issuer; validate (sub, actor) delegation; enforce chain-depth limit; optionally verify `act.cnf` key | Preserve actor in issued token; see {{jwt-assertion-grants-processing}} |
| Token-exchange AS | `subject_token` or inbound token with `act` chain | Validate token; trust issuer; validate actor authorization; enforce scope reduction and chain-depth limit | Issue token with preserved or extended `act` chain; see {{jwt-access-token-propagation}} |
| Resource Server | Access token or Transaction Token with `act` | Validate token; validate top-level `cnf` binding; evaluate subject; evaluate actor when required by local policy | Apply delegated-token policy; advertise `actor_authorization_required`; see {{jwt-access-token-rs-processing}} |
| Transaction Token Service | Inbound token with subject and optional `act` chain | Validate token; authenticate workload; enforce chain-depth limit; bind new presenter key | Preserve `sub`; set `req_wl`; add new outermost `act`; see {{transaction-token-service-processing}} |
| Client or Agent | AS and RS metadata | Check `entity_profiles_supported.actor` and `actor_profile_token_types_supported` | Abort if capability mismatch; see {{discovery-capability-negotiation}} |


## Actor Object Structure

An actor object is a JSON object that is the value of the `act` claim. In addition to the `sub` claim required by {{RFC8693}}, an actor object MUST contain an `iss` claim, SHOULD contain a `sub_profile` claim, and MAY contain a `cnf` claim.

~~~
act-object = {
  "sub"           : StringOrURI,        ; REQUIRED
  "iss"           : StringOrURI,        ; REQUIRED
  ? "sub_profile" : JSON String,        ; RECOMMENDED
  ? "cnf"         : cnf-object,         ; OPTIONAL
  * StringOrURI => any                  ; extension claims
}
~~~

`sub`:
: REQUIRED.  The subject identifier of the actor, as defined in {{RFC8693, Section 4.1}}.  This value identifies the acting party.  It is a StringOrURI as defined in {{RFC7519}}.

`iss`:
: REQUIRED.  Identifies the namespace authority for the actor identifier carried in `act.sub`, playing the same role for `act.sub` that the JWT `iss` claim plays for the token `sub`: just as `iss` + `sub` form a globally unique principal identifier in a JWT (see {{RFC9493}}), `act.iss` + `act.sub` form a globally unique actor identifier within the delegation chain.  For URI, client, workload, or other deployment-specific identifiers, the value of `act.iss` MUST identify the authority that the deployment treats as authoritative for resolving or validating that actor identifier.  See "Cross-Domain Delegation" in {{conventions}}.  The value is a StringOrURI as defined in {{RFC7519}}.

  Unlike the credential issuer, which is an AS-internal concern resolved during assertion validation, `act.iss` identifies only the namespace authority and is not a credential-issuer claim.  The namespace authority and the credential issuer may be the same entity or different entities.  The value SHOULD be an HTTPS URL unless the deployment uses a well-known alternative identifier scheme (e.g., a URN for workload identities).

  For example, a TTS might issue a Transaction Token with top-level `iss` equal to `https://tts.travel-provider.example` while setting `act.iss` for the booking tool to `https://as.travel-provider.example`, if local policy treats that AS as authoritative for the booking tool identifier namespace.

`sub_profile`:
: RECOMMENDED.  A space-delimited list of entity profile values classifying the actor identified by `act.sub`, as defined in Section 4.2 of {{I-D.mora-oauth-entity-profiles}}.  Values used within `act` objects MUST be registered with the "Actor Profile" usage location in the OAuth Entity Profiles registry (Section 14.1 of {{I-D.mora-oauth-entity-profiles}}) or be privately defined collision-resistant values.  If the acting entity fits more than one profile, multiple values MAY be included as a space-delimited string (e.g., `"service ai_agent"`).  Policy evaluation rules for multi-value strings are defined in {{forward-compat-sub-profile}}.

`cnf`:
: OPTIONAL.  A confirmation claim as defined in {{RFC7800, Section 3.1}}.  When present, carries keying material associated with the actor identified by this `act` object.  It is distinct from the top-level `cnf` claim, which identifies the keying material the current token presenter MUST use to demonstrate proof of possession.  `act.cnf` does not create a presenter-binding obligation for the current token.  When a public key is conveyed, the `jkt` member is RECOMMENDED.

  For the **outermost** `act` object, `cnf` carries the actor-associated key of the immediate acting party.  Under local AS policy, when an inbound assertion's outermost `act` carries a `cnf.jkt` value, the AS MAY treat this as an optional sender-constraint check: it MAY verify that the inbound DPoP proof (or mTLS certificate) is consistent with that key.  An AS that applies this check MUST document it and MUST return `invalid_grant` when the check fails.  This check verifies that the proof matches the key the asserting party claimed; it does not independently verify that the actor possesses that key, and its security value is bounded by the trustworthiness of the assertion issuer.  It does not substitute for issuer-authority validation per {{jwt-assertion-grants-processing}} step 2.

  For **inner** (nested) `act` objects, `cnf` records the actor-associated key from an earlier hop as endorsed by the outer token issuer.  Inner `act.cnf` values MUST NOT drive proof-of-possession requirements at any verification layer.

The `client_profile` claim defined in {{I-D.mora-oauth-entity-profiles}} classifies the OAuth client and MUST NOT appear within an `act` object.  Client classification belongs at the top level of the token.  An AS or RS that encounters a `client_profile` member inside an `act` node MAY reject the token or ignore the offending member; it MUST NOT treat it as a valid actor classification.


## Multi-Value `sub_profile` Policy Evaluation {#forward-compat-sub-profile}

Value preservation and propagation for unrecognized `sub_profile` values are governed by {{I-D.mora-oauth-entity-profiles}}.  An AS or RS MUST NOT reject a token or assertion solely because a `sub_profile` value is unrecognized; unrecognized values MUST NOT be used to infer authorization semantics.

When local policy restricts the accepted set of `sub_profile` values for an actor, that set SHOULD be advertised via `entity_profiles_supported.actor` in AS metadata ({{authorization-server-metadata}}) so that clients can detect incompatibility before making a request.  Clients discover the accepted set for a given resource by consulting `entity_profiles_supported.actor` in the AS metadata for the AS listed in the resource's `authorization_servers` ({{RFC9728}}).

When `sub_profile` is absent from an `act` object, implementations MUST NOT assume a specific entity type for the actor.  Resource servers that enforce entity-type-based access control MUST treat an absent `sub_profile` as an unclassified actor and SHOULD apply the more restrictive policy applicable to unknown entity types.

When `sub_profile` contains multiple space-delimited values, the following rules apply to policy evaluation:

*  An entity MUST be treated as matching a policy rule if any of its `sub_profile` values satisfies that rule.  For example, an entity with `"service ai_agent"` matches both a policy rule for `service` and a policy rule for `ai_agent`.

*  When multiple values match different policy rules with conflicting outcomes, the more restrictive outcome MUST apply.  Implementations MUST NOT grant the union of all matched rules' privileges; they MUST apply least privilege across the matched set.

*  When local policy accepts only a specific set of `sub_profile` values (e.g., via `entity_profiles_supported.actor`), the entity is accepted if at least one of its values appears in the accepted set.  Values not in the accepted set MUST be ignored for that acceptance check; they do not cause rejection on their own.


## Delegation Chains {#delegation-chains}

Delegation chains MUST be represented by nesting `act` objects as specified in {{RFC8693, Section 4.1}}.  In a nested structure, the outermost `act` object identifies the immediate actor; inner `act` objects represent prior actors in the chain, with the innermost representing the original delegating party.  This structure records delegated-actor history within the trust model of the issuer that conveys it; it does not, by itself, provide independent cryptographic provenance for each prior hop.

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
In this example the booking tool (outermost `act`) is the current immediate actor.  The delegation chain originates with Alice (`sub`), who first authorized the travel assistant (nested `act`); the travel assistant then sub-delegated to the booking tool, which is now the immediate presenter.  Each actor carries associated key context via `cnf.jkt`, and the chain carries prior-actor information to the extent that the current token issuer is trusted to have validated and faithfully propagated it.

Delegation depth is defined as the number of `act` objects in the chain, counting from the outermost.  A token with a single `act` object and no nested `act` within it has depth 1.  Each additional level of nesting adds 1.  Depth is counted on the resulting chain after any new outermost `act` is added, not on the inbound token.  The default maximum delegation depth is five levels of nested `act` objects.  This covers realistic multi-tier architectures (e.g., user → orchestrator → agent → tool → internal service) while bounding parsing complexity and policy evaluation surface; real-world deployments rarely exceed three to four hops.  Implementations MUST NOT represent delegation depth greater than five levels.  Implementations that receive a token with more than five levels of nested `act` MUST reject it.  When a request would result in a chain that exceeds the limit, the AS MUST reject the request with `invalid_request`; it MUST NOT silently truncate the chain.


## Sender Constraint and Key Binding {#sender-constraint}

When sender-constrained tokens are used with a delegated token, the top-level `cnf` claim identifies the key or certificate of the immediate presenter of that token.  When delegation is present, that immediate presenter is the outermost actor.  Authorization servers and resource servers validate current proof of possession against the top-level `cnf`, not against nested `act.cnf` values.  `act.cnf` values, at any nesting depth, do not create an additional proof-of-possession obligation at the resource server unless another specification or local policy explicitly requires it.

When DPoP {{RFC9449}} is used:

*  The top-level `cnf.jkt` of the token MUST identify the key of the immediate presenter—the actor identified by the outermost `act` claim.
*  The `cnf.jkt` within each `act` object identifies keying material associated with that specific actor in the delegation chain.

This separation ensures that the resource server can verify the current presenter while carrying forward actor-associated key history for prior actors in the delegation chain within the trust model of the current issuer.  In single-hop cases, the top-level `cnf` and outermost `act.cnf` can carry the same value because the current presenter and current actor are the same party.  In multi-hop cases, the top-level `cnf` identifies the current presenter for the current token, while nested `act.cnf` values can preserve actor-key context for earlier hops.

When mTLS ({{RFC8705}}) is used instead of DPoP, the top-level `cnf.x5t#S256` identifies the current presenter's certificate. Actor-associated key history in `act.cnf` SHOULD use a confirmation member appropriate to the mechanism in use; where a public key is conveyed, `jkt` is RECOMMENDED.

## Backwards Compatibility

This profile extends the base `act` claim semantics from {{RFC8693}} by unconditionally requiring `iss` in every `act` object, and by defining `sub_profile` and optional actor-scoped confirmation semantics.  Deployments that receive an `act` object that conforms only to {{RFC8693}} but omits this profile's required members MUST treat that actor object as not conforming to this profile.

When a token or assertion is required by local policy or advertised metadata to conform to this profile, such non-conforming `act` objects MUST be rejected.  When profile conformance is not required, implementations MAY continue to process a base {{RFC8693}} `act` object according to local policy, but they MUST NOT infer profile-defined semantics for claims that are absent.

The RECOMMENDED migration path for deployments currently using RFC 8693 `act` objects is:

1.  Begin emitting `act.iss` in all newly issued tokens.  Existing consumers that do not recognize `act.iss` will ignore it.
2.  Update consumers to validate `act.iss` per {{actor-object-structure}} once issuers have deployed step 1.
3.  Once all token issuers and consumers on a given path have been updated, resource servers can enforce profile conformance by setting `actor_profile_required: true` in their Protected Resource Metadata.  ASes that wish to accept only profile-conformant inbound assertions can do so via local policy once issuers on inbound paths have deployed step 1.

This graduated approach avoids a flag-day cutover and allows incremental rollout across trust domains.

Implementations that previously treated `act.cnf` as an active sender-constraining mechanism at every chain depth should note that this document restricts optional PoP use of `act.cnf` to the outermost `act` object only, under explicit local AS policy.  For inner (non-outermost) `act` objects, `act.cnf` is actor-associated key context and MUST NOT drive proof-of-possession requirements.  The active PoP obligation always rests on the top-level `cnf` claim and the immediate presenter.  Deployments that relied on per-hop `act.cnf` key verification for multi-hop security properties MUST review their key-verification logic against the semantics defined in {{actor-object-structure}} and {{sender-constraint}}.


## Summary of Actor-Profile Claims {#actor-profile-claims-summary}

The `act` object structure defined in {{actor-object-structure}} applies uniformly to all three token types covered by this document.  The following tables summarize claim requirements.

The sub-claims of the `act` object have the same requirement level regardless of which token type carries the `act` claim:

| Claim | Requirement | Definition |
|-------|-------------|------------|
| `act.sub` | REQUIRED when an `act` claim is present | {{actor-object-structure}} |
| `act.iss` | REQUIRED | {{actor-object-structure}} |
| `act.sub_profile` | RECOMMENDED | {{actor-object-structure}} |
| `act.cnf` | OPTIONAL | {{actor-object-structure}} |

Requirements for top-level claims vary by token type:

| Claim | JWT Assertion Grant | JWT Access Token | Transaction Token |
|-------|--------------------|--------------------|-------------------|
| `act` | REQUIRED when delegation is asserted | REQUIRED when delegated | REQUIRED when delegated |
| `sub_profile` | RECOMMENDED | RECOMMENDED | propagated from inbound subject token |
| `cnf.jkt` | OPTIONAL; set by issuing AS when DPoP binding is applied | REQUIRED when DPoP is used ({{sender-constraint}}) | REQUIRED; set by TTS to bind the new presenter |
| `req_wl` | not applicable | not applicable | REQUIRED |

The `sub_profile` claim MAY also appear as a top-level JWT claim (outside any `act` object) to classify the entity type of the token's `sub`.  Its value MUST be a space-delimited entity profile string per {{I-D.mora-oauth-entity-profiles}} and applies exclusively to `sub`; it does not affect `sub_profile` values within `act` objects.  Issuers SHOULD include a top-level `sub_profile` when they can authoritatively classify the subject entity type.

Detailed semantics and processing rules for each token type are defined in {{jwt-assertion-grants}}, {{jwt-access-tokens}}, and {{transaction-tokens}} respectively.


# JWT Assertion Grants {#jwt-assertion-grants}

## Structure {#jwt-assertion-grants-structure}

A JWT used as an authorization grant {{RFC7521}}{{RFC7523}} MAY include an `act` claim conforming to the actor profile defined in {{actor-profile}}.  Use of this claim in JWT client authentication assertions is out of scope for this document because such assertions have different issuer and subject semantics.  However, implementers should note that some deployments rely on the authenticated OAuth client itself as implicit evidence of the acting party.  This specification does not prohibit that input, but when delegation is to be expressed explicitly and propagated across token transformations, the acting party is represented by `act` rather than inferred solely from client authentication.

The following claims are defined for a JWT assertion grant that carries actor-profile delegation.  Claims not listed here follow the requirements of {{RFC7521}} and {{RFC7523}}.

`iss` (REQUIRED):
: Identifies the assertion issuer.  MUST be authorized by local policy to assert the relationship between `sub` and `act.sub`.

`sub` (REQUIRED):
: The principal on whose behalf the grant is being made.

`aud` (REQUIRED):
: MUST identify the token endpoint of the receiving AS, per {{RFC7523, Section 3}}.

`exp` (REQUIRED):
: Expiration time per {{RFC7519}}.

`iat` (REQUIRED):
: Issued-at time per {{RFC7519}}.

`jti` (REQUIRED):
: A unique identifier for the assertion per {{RFC7523, Section 3}}, used to prevent replay.

`sub_profile` (RECOMMENDED):
: Classifies the entity type of `sub`.  MUST conform to the values defined in {{actor-profile}}.

`act` (REQUIRED when delegation is asserted):
: The actor object identifying the entity exercising the subject's delegated rights.  MUST conform to the actor object structure defined in {{actor-profile}}.  MUST include `act.sub` and `act.iss`.

When the assertion or request context also identifies an OAuth client via `client_id`, `azp`, or an authenticated client credential, that client identity MUST NOT be treated as a substitute for `act.sub`; see step 7 in {{jwt-assertion-grants-processing}}.

An Identity Assertion JWT Authorization Grant (ID-JAG) {{I-D.ietf-oauth-identity-assertion-authz-grant}} is a JWT produced by token exchange and presented via the JWT bearer grant.  When used as a JWT bearer assertion grant, this section applies.  Clients SHOULD use this path only when the AS's support for the actor-determination model has been confirmed via deployment documentation, prior agreement, or discovery.

The following example shows an AS-issued assertion grant, which is the recommended pattern.  The Enterprise IdP AS performed Token Exchange, authenticated the agent as the OAuth client, established the delegation relationship under local policy, and signed the assertion.  `act.iss` equals the token `iss` here because the enterprise AS is also the namespace authority for the agent's identifier:

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
    "sub_profile": "ai_agent",
    "cnf": { "jkt": "NzbLsXh8uDCcd7MNwrnNZpX0ak8ACQ" }
  }
}
~~~

The top-level `sub_profile` classifies the JWT's `sub`; `sub_profile` within `act` classifies the actor.  The top-level `cnf.jkt` binds the assertion to the agent's DPoP key; `act.cnf.jkt` records the same key as actor-associated key context.

The AS-issued grant is the more trustworthy pattern because the issuing AS independently authenticated the agent and validated the delegation relationship before signing.  The receiving AS needs to trust only the enterprise AS as an issuer.

For AS-issued grants, the issuing AS MUST set `act.iss` to the namespace it is authoritative for, typically its own issuer URI.

Allowed issuer patterns are:

*  an AS-issued delegated assertion where JWT `iss` is a trusted AS and `act.sub` identifies the actor (recommended),
*  an assertion carrying a pre-existing nested `act` chain where the current JWT `iss` is trusted to carry forward prior actor assertions, and
*  a self-issued actor assertion (see {{self-issued-grants}} below).

### Self-Issued Grants {#self-issued-grants}

In a self-issued assertion grant, the agent itself is `iss` and directly asserts Alice's delegation to itself in `act.sub`.  No upstream AS has authenticated the agent or pre-validated the delegation relationship.

~~~json
{
  "iss": "https://agents.enterprise.example/travel-assistant",
  "sub": "https://idp.enterprise.example/users/alice",
  "aud": "https://as.resource-domain.example/token",
  "jti": "a1b2c3d4-...",
  "exp": 1711820400,
  "iat": 1711816800,
  "sub_profile": "user",
  "act": {
    "sub": "https://agents.enterprise.example/travel-assistant",
    "iss": "https://as.enterprise.example",
    "sub_profile": "ai_agent",
    "cnf": { "jkt": "NzbLsXh8uDCcd7MNwrnNZpX0ak8ACQ" }
  }
}
~~~

Here `act.iss` names the enterprise AS as the namespace authority for the agent's identifier, which is registered in that AS's namespace rather than the agent's own identifier space.  The client MUST set `act.iss` to the issuer namespace authoritative for its own identifier, typically the AS governing the namespace in which `act.sub` is registered; the client MUST NOT set `act.iss` to its own identifier unless it is itself the authoritative namespace owner.  The client SHOULD include `act.cnf.jkt` set to the JWK thumbprint of the key it will use for proof of possession.

ASes MUST NOT accept self-issued assertion grants unless explicitly configured via local policy to do so; self-issued grant acceptance MUST be off by default.  When an AS does accept self-issued grants, it MUST apply heightened scrutiny to the claimed `sub` and delegation relationship.


## Authorization Server Processing {#jwt-assertion-grants-processing}

When an AS receives a JWT assertion grant containing an `act` claim:

1.  The AS MUST validate the assertion per {{RFC7523}}.

2.  The AS MUST determine that the assertion issuer is trusted to assert the relationship between the JWT `sub` and `act.sub`.  If the issuer does not identify the same logical entity as `act.sub` under local policy, the AS MUST apply local policy to determine whether the issuer is authorized to speak for that actor.

3.  The AS MUST verify that `act.iss` is authoritative for the identifier namespace of `act.sub`.  Whether a given `act.iss` is authoritative for the namespace containing `act.sub` is a local trust decision; this document defines no generic algorithm for proving namespace authority, as no such algorithm is universally applicable across identifier ecosystems.  Examples of mechanisms deployments use to make this determination include (but are not limited to):

    *  URL namespace containment for URI-shaped identifiers (e.g., `act.sub` uses the same scheme, host, and port as `act.iss`).
    *  Federation metadata that lists `act.iss` as authoritative for the namespace containing `act.sub` (e.g., {{OpenID.Federation}}).
    *  Pre-registration entries that explicitly authorize `act.iss` to assert identifiers of the form used in `act.sub`.
    *  Local policy rules that authorize `act.iss` to assert the specific class of identifier used in `act.sub`.

    If the AS cannot establish through its local trust determination that `act.iss` is authoritative for the namespace containing `act.sub`, the AS MUST reject the request with `invalid_grant`.

4.  The AS MUST verify that the `act.sub` is authorized to act on behalf of the assertion's `sub`, using the AS's own policy (for example, a pre-registered delegation grant, a consent record, or a policy rule).

5.  If the inbound assertion's `act` object contains a nested `act` claim (indicating that the asserted actor is itself a delegatee), the AS MUST handle the inner chain as follows:

    *  **Propagation decision**: The AS MUST determine whether to propagate the inner chain into the issued token.  The AS SHOULD propagate it by preserving the nested structure, provided the total resulting chain depth does not exceed the limit in {{delegation-chains}}.  If the AS does not accept pre-chained assertions, it MUST reject the request.

    *  **Validation of inner chain**: An AS that propagates an inner chain MUST independently validate each inner `act.sub` and `act.iss` before including that chain in the issued token, applying the same namespace-authority mechanisms as step 3 and the same delegation-authorization mechanisms as step 4 of this section to each inner `act` object.  If the AS cannot independently validate an inner chain, it MUST reject the request with `invalid_grant`.  An AS MUST NOT silently carry forward an unvalidated inner chain, and it MUST NOT issue a token that omits only the unvalidated portion of an asserted delegation chain; doing so introduces unverified or misleading actor claims into a token bearing its own signature.

    *  **Delegation relationships**: When the AS relies on the inner chain as a security-relevant delegation path (rather than informational audit context), it MUST also validate the delegation relationship at each hop in the nested chain according to local policy.

6.  The AS MUST verify proof of possession for the token request according to the token-endpoint proof mechanism in use (DPoP per {{RFC9449}} or mutual TLS per {{RFC8705}}).  When the outermost `act` object in the inbound assertion carries a `cnf.jkt` claim and local AS policy applies the optional sender-constraint check described in {{actor-object-structure}}, the AS MUST verify that the inbound DPoP proof (or mTLS certificate) is consistent with that key.  If that check fails, the AS MUST reject the request with `invalid_grant`.  Inner `act.cnf` values below the outermost level MUST NOT drive proof-of-possession requirements for the current request; see {{actor-object-structure}}.

    When DPoP is in use and the inbound assertion does not carry `act.cnf`, the AS MUST still bind the issued token's top-level `cnf.jkt` to the public key from the DPoP proof submitted at the token endpoint, per {{RFC9449}}.  When mTLS ({{RFC8705}}) is used, the same optional local-policy key-consistency check applies to the outermost `act.cnf`: if the outermost `act.cnf` carries a certificate hash member (e.g., `x5t#S256`), the AS MAY verify that the mTLS client certificate presented with the request matches that value.  Nested `act.cnf` values in inner chain objects MUST NOT drive mTLS certificate requirements for the current request.

7.  If the assertion or authenticated request context identifies an OAuth client separately from `act.sub`, the AS MAY use that client identity as an additional authorization input.  The AS MUST NOT infer that the client is authorized to act on behalf of the subject solely because the client initiated the request.  When the client identity is intended to identify the same acting party as `act.sub`, the AS MUST validate semantic consistency between the two identifiers before issuing a token.

8.  If the AS accepts the assertion, it MUST propagate the actor information into the issued token according to the rules for the output token type being issued.  For JWT access tokens, see {{jwt-access-token-propagation}}.  For Transaction Tokens, see {{transaction-token-service-processing}}.  When the output is another JWT assertion grant profile, the resulting assertion MUST preserve the validated actor information subject to local policy and the chain-depth limit in {{delegation-chains}}.

9.  The AS MAY add additional `sub_profile` or `act` metadata to the issued token based on its own knowledge of the principals.


## Error Responses {#assertion-error-responses}

When an AS rejects a JWT assertion grant or Token Exchange request for reasons related to actor-profile validation, it MUST return an OAuth error response per {{RFC6749, Section 5.2}} and {{RFC8693, Section 2.2}}. The following error codes apply:

`invalid_request`:
: Use when the `act` claim structure is syntactically invalid, the delegation chain depth exceeds the limit in {{delegation-chains}}, or a required claim (`act.sub` or `act.iss`) is absent.

`invalid_grant`:
: Use when the assertion or subject token cannot be validated, the issuer is not trusted to assert the delegation relationship, or the request's required proof-of-possession check cannot be confirmed.

`access_denied`:
: Use when AS policy does not permit the identified actor (`act.sub`) to act on behalf of the subject (`sub`), or when the requested scope exceeds what is permitted for this (subject, actor) pair.

The `error_description` field SHOULD be included and SHOULD describe which aspect of actor-profile validation failed, to the extent permitted by the AS's security and privacy policy.

Example rejection:

~~~json
{
  "error": "access_denied",
  "error_description": "actor_profile not permitted for requested scope"
}
~~~


## Refresh Tokens {#assertion-grant-refresh-tokens}

Authorization servers SHOULD NOT issue refresh tokens in response to delegated JWT assertion grant requests that cross a trust boundary.  For this purpose, a request crosses a trust boundary when the AS relies on delegation or identity assertions originating outside its own trust domain for the subject, the actor, or their relationship.  This is consistent with the guidance in {{I-D.ietf-oauth-identity-chaining}}.  In same-domain deployments, including authorization code flows or same-issuer token exchange where the issuer can continuously enforce local delegated-access policy, an AS MAY issue refresh tokens according to local policy.

The rationale is that a cross-domain delegated assertion grant is itself a short-lived, re-issuable artifact: the actor can obtain a new assertion grant and exchange it for a new access token at any time, provided the delegation relationship remains valid.  Issuing a long-lived refresh token in that context would allow an actor to continue obtaining access tokens without re-presenting a current assertion, bypassing re-validation of the delegation relationship at the trust boundary.  This creates a window where a revoked or expired delegation could still yield new access tokens until the refresh token itself expires or is revoked.

When a cross-domain delegated token is needed beyond the original access token lifetime, the preferred pattern is to re-present a new assertion grant, which ensures that:

*  The delegation relationship is re-validated at each token issuance.
*  The current presenter binding (`cnf`) is re-validated.
*  Revocation of the delegation relationship takes effect immediately at the next token request.

If an AS issues a refresh token in a delegated context, it MUST bind the refresh token to the specific (subject, actor) pair from the original grant and MUST NOT allow the refresh token to be used to obtain tokens for a different actor.  In cross-domain deployments, the AS SHOULD additionally ensure that refresh-token use does not bypass the revocation and trust-boundary checks that would otherwise occur when re-presenting the assertion grant.  When the delegation relationship between the subject and actor is revoked, the AS MUST ensure that the refresh token cannot be used to mint new delegated tokens after revocation is known to the AS.  The AS SHOULD revoke the refresh token immediately when it has authoritative knowledge of the revocation.  When immediate revocation is not observable (for example, in federated or cross-domain deployments where revocation signals may not be delivered in real time), the AS SHOULD require re-presentation of a current upstream assertion before issuing additional delegated tokens.


# JWT Access Tokens {#jwt-access-tokens}

## Structure

A JWT access token {{RFC9068}} MUST include an `act` claim conforming to the actor profile defined in {{actor-profile}} when any of the following conditions hold:

*  The token was issued following acceptance of a JWT assertion grant that itself contained an `act` claim per {{jwt-assertion-grants}};
*  The token was issued via Token Exchange ({{RFC8693}}) and the request carried explicit actor information, such as an `actor_token` or a `subject_token` that already carried an `act` chain; or
*  The AS has independent knowledge — established by a pre-registered delegation grant, an explicit consent record, or a policy rule that names both the subject and the acting party — that the subject's rights are being exercised by a distinct, identifiable acting party whose identifier and entity type the AS can assert authoritatively.  Claims about actor identity derived solely from client authentication, without an explicit delegation record naming both the subject and the acting party, do not satisfy this condition.

When the AS determines the actor from authenticated client context, local delegation policy, or other deployment-specific inputs rather than from an explicit actor-carrying artifact, that is an operational allowance rather than an interoperable actor-proof mechanism defined by this document.  The interoperability defined here applies to the issued token and its processing, not to the upstream method by which the AS determined the actor.

When none of these conditions hold, the AS MUST NOT include an `act` claim in the issued token.  When a client explicitly requests a delegated output — for example by supplying an `actor_token` parameter in a Token Exchange request — but none of the above conditions hold and no independent delegation basis can be established, the AS MUST reject the request with `invalid_grant`.

Some deployments also carry an `azp` claim as an auxiliary client-identity signal, often as an OpenID Connect carry-over used by vendors in practice.  It is referenced here for completeness, not because this document, {{RFC9068}}, or {{RFC8693}} makes it a required delegation input.  When an issuer uses both `azp` and `act.sub` to represent the same acting party, it MUST ensure they are semantically consistent.  Client identity claims (`client_id`, `azp`) identify the OAuth client, not the delegation relationship; they MUST NOT be treated as a substitute for `act`.  For migration and reconciliation rules, see {{migration-implicit-explicit}} and {{client-identity-delegation}}.

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
    "sub_profile": "ai_agent",
    "cnf": { "jkt": "NzbLsXh8uDCcd7MNwrnNZpX0ak8ACQ" }
  }
}
~~~

In this single-hop case the top-level bearer key and `act.cnf.jkt` are identical because the actor is the bearer.  In multi-hop chains each actor carries a distinct key; see {{appendix-cross-domain}}.


## Authorization Server Processing {#jwt-access-token-propagation}

When an AS issues a JWT access token following acceptance of a JWT assertion grant that contains an `act` claim ({{jwt-assertion-grants}}) or a Token Exchange request ({{RFC8693}}) that carries explicit actor information, the AS MUST apply the following rules.  These requirements apply regardless of whether the inbound credential is a JWT assertion grant, a JWT access token, or a Transaction Token.

1.  The AS MUST include an `act` claim in the issued access token that preserves or extends the delegation relationship conveyed in the inbound request.  The AS MAY add a new outermost `act` layer for a newly-identified actor but MUST NOT omit, modify, or contradict the validated inner chain.  The AS MUST NOT silently drop actor information.

2.  The AS MUST preserve `sub` to refer to the same underlying subject as the inbound token.  If the AS uses a different subject-identifier namespace, it MAY change the `sub` value only to re-express that same subject in the new namespace.  It MUST NOT replace `sub` with an identifier for a different subject.

3.  When the inbound token carries an `act` claim, the AS MUST nest the existing chain within a new outermost `act` for the newly-identified actor.  The new outermost `act` MUST include `act.sub` and `act.iss` for the newly-identified actor.  The namespace-authority obligation in this step applies only to the `act` object the AS creates for the current actor.  The AS MUST NOT rewrite `act.iss` or any other field in `act` objects inherited from the inbound subject token, regardless of inbound chain depth; those entries were authored by upstream ASes and their content is fixed at the point of authorship.

4.  If the inbound actor information cannot be validated, or nesting would exceed the chain-depth limit in {{delegation-chains}}, the AS MUST reject the request.  The AS MUST return `invalid_request` when the chain-depth limit would be exceeded and `invalid_grant` when actor information fails validation; see {{assertion-error-responses}}.  The AS MUST NOT issue a token that partially preserves the delegation chain.

5.  The AS SHOULD include `sub_profile` in the issued token's top-level claims if it can authoritatively classify the token's `sub` entity type.

6.  The AS MUST apply least-privilege scope reduction:

    *  The issued token's `scope` MUST NOT exceed the scope of the subject's original authorization.
    *  When the request specifies a `scope` parameter, the issued token MUST NOT carry scopes beyond those explicitly requested.
    *  When no `scope` parameter is specified, the AS SHOULD restrict scope to the minimum necessary for the stated purpose.
    *  The AS SHOULD further restrict scope based on policy for the (subject, actor) pair or the actor's `sub_profile`.
    *  The granted scope MUST be reflected in the issued token's `scope` claim.
    *  If applying all reduction steps yields an empty scope, the AS MUST reject the request with `invalid_scope`.

7.  If the inbound credential carries `client_id`, `azp`, or both, the AS MAY preserve those values per the output token profile or local policy.  If preserved:

    *  They MUST continue to identify the OAuth client and MUST NOT be rewritten to represent delegation state that belongs in `act`.
    *  If preserving them would create ambiguity about the delegated actor relationship, the AS SHOULD omit them.

    See {{client-identity-delegation}} for the normative rules governing client identity and actor identity.


## Resource Server Processing {#jwt-access-token-rs-processing}

Upon receiving a JWT access token that contains an `act` claim, a resource server MUST validate and process that token according to its local delegated-token policy.  The authorization-policy model for delegated tokens is defined in {{dual-principal-authorization}}.  A resource server that requires actor authorization for delegated tokens MUST advertise that requirement using `actor_authorization_required: true` ({{protected-resource-metadata}}).

When the resource server accepts delegated tokens, it MUST:

1.  Validate the token's signature, `iss`, `aud`, and temporal claims per {{RFC9068}}.  If the resource advertises `actor_profile_required: true` ({{protected-resource-metadata}}) and the token carries no `act` claim, or carries an `act` object missing `act.iss`, the RS MUST reject the request with HTTP 401 `invalid_token`.

2.  If the token carries a top-level `cnf.jkt`, validate the accompanying DPoP proof per {{RFC9449}}.  The DPoP proof MUST be signed by the key identified in `cnf.jkt`, which is the key of the immediate bearer—the outermost actor when delegation is present.  If the token carries `cnf.jkt` but no DPoP proof is present in the request, the RS MUST reject the request per {{RFC9449}} Section 7.  If a DPoP proof is present but the token does not carry `cnf.jkt`, the RS MUST treat the token as a bearer token; the RS MUST NOT infer a confirmation binding from the DPoP proof key.

3.  Extract the `sub` and the outermost `act.sub` as the two principals relevant for authorization policy.

4.  If the token carries `client_id`, `azp`, or both, treat those as client-identity inputs only.  The RS MUST NOT use `client_id` or `azp` as a substitute for `act.sub`.  When local policy expects both to identify the same acting party, the RS MUST verify semantic consistency; in the absence of such a policy determination, the RS MUST treat them as distinct identifiers.  See {{client-identity-delegation}}.

5.  Apply actor authorization per {{dual-principal-authorization}} when required by local policy.  Resource servers that do not require actor authorization SHOULD still evaluate the actor as part of authorization, audit, or trust decisions.

6.  Optionally traverse inner `act` objects to audit the full delegation chain; inner actors are informational and MUST NOT be required to present proof of possession at the resource server.

7.  If the resource server relies on inner `act` objects for audit, policy refinement, or trust decisions, it MUST do so only after validating the outer token issuer and only when local policy trusts that issuer to carry forward the asserted delegation chain.  The RS MUST NOT treat nested actor issuers as independently authenticated merely because their identifiers appear in inner `act.iss` values.

8.  If any of the above steps fail, return an appropriate error response per {{RFC6750, Section 3.1}}:

    *  If signature, `iss`, `aud`, or temporal validation fails: HTTP 401 with `WWW-Authenticate: Bearer error="invalid_token"`.
    *  If DPoP proof validation for `cnf.jkt` fails: HTTP 401 per {{RFC9449, Section 7}}.
    *  If the token is structurally valid but the actor fails an authorization evaluation required by local policy, including actor authorization when required: HTTP 403.  When the subject's scope was sufficient but the actor lacked authorization, the RS SHOULD include `error="insufficient_scope"` in the `WWW-Authenticate` challenge.  When the actor's `sub_profile` is not accepted under the actor-profile policy applicable to the resource, the RS SHOULD either return HTTP 403 without an OAuth error code or map the failure to `error="insufficient_scope"` when that accurately reflects the policy model.
    *  The RS MUST NOT include actor-specific rejection details in error responses exposed to clients outside the trust domain.


## Token Introspection {#token-introspection}

When token introspection ({{RFC7662}}) is used, an AS that issues delegated tokens MUST include the `act` claim and top-level `sub_profile` claim in introspection responses for active tokens that carry those claims.  The AS MUST NOT omit actor profile claims from introspection responses, as their omission would misrepresent the delegation status of the token to the introspecting RS.  When a delegated token carries a nested `act` chain (delegation depth greater than 1), the introspection response MUST include the complete nested `act` structure; partial omission of inner `act` objects is not permitted.  An introspecting RS MUST treat the introspection response as a faithful representation of the token's full delegation chain.

Resource servers using introspection for delegated tokens MUST apply the same delegated-token policy to the introspection response claims that they would apply to equivalent locally validated tokens, including actor authorization when required by local policy.  An RS that relies on introspection rather than local JWT validation MUST treat a missing `act` claim in the introspection response as an inconsistency whenever local policy, protected resource metadata, or other token context indicates that the token is delegated or that actor-profile conformance is required.  In such cases, the RS MUST reject the token.  Only when no such requirement or indication exists MAY the RS treat an active introspection response without `act` as representing a non-delegated token.

Introspection endpoints for delegated tokens SHOULD be advertised via the `introspection_endpoint` parameter in AS metadata ({{RFC8414}}). When revocation is integrated, the introspection response for a revoked delegated token MUST return `"active": false` and MUST NOT include `act` or `sub_profile` claims.


# Transaction Tokens {#transaction-tokens}

## Overview

Transaction Tokens {{I-D.ietf-oauth-transaction-tokens}} are short-lived JWTs that capture the workload identity and request context for a series of related service calls within a single business transaction. They are issued by a Transaction Token Service (TTS), which is a specialized authorization server.

> Note: The requirements in this section are written against the current version of {{I-D.ietf-oauth-transaction-tokens}}.  If that document changes its claim structure materially before publication, this section should be read in conjunction with the published version.

A Transaction Token contains the following claims.  Claims marked REQUIRED are defined as such by {{I-D.ietf-oauth-transaction-tokens}}; claims marked OPTIONAL may be omitted at the issuer's discretion.

`iss` (OPTIONAL):
: Issuer of the Transaction Token.  SHOULD be present when the Transaction Token crosses trust-domain boundaries, as recipients require `iss` to validate chain-of-trust per this document.  MAY be omitted when all tokens are scoped to a single Trust Domain and all recipients have out-of-band knowledge of the issuer.  When `iss` is omitted, recipients MUST rely on the trust-domain and issuer-identification rules of {{I-D.ietf-oauth-transaction-tokens}} and local deployment configuration rather than the generic outer-token `iss` processing rules in this document.

`aud` (REQUIRED):
: Identifies the Trust Domain in which the Transaction Token is valid.

`iat` (REQUIRED):
: Issued-at time per {{RFC7519}}.

`exp` (REQUIRED):
: Expiration time per {{RFC7519}}.

`sub` (REQUIRED):
: The identity of the human user or non-personal entity that originated the transaction.

`scope` (REQUIRED):
: The transaction authorization scope, as defined by the TTS for the specific transaction.

`txn` (REQUIRED):
: A transaction identifier that links all tokens in the same business transaction, as defined in {{I-D.ietf-oauth-transaction-tokens}}.

`req_wl` (REQUIRED):
: The workload identifier of the workload that requested the Transaction Token from the TTS.  When a Transaction Token is exchanged for a replacement, `req_wl` is updated to reflect the new requesting workload per {{I-D.ietf-oauth-transaction-tokens}}.  This claim provides TTS-level workload context and is not a substitute for `act.sub`; see {{actor-claim-in-transaction-tokens}}.

`tctx` (OPTIONAL):
: Transaction context information that remains stable across the call chain.

`rctx` (OPTIONAL):
: Request context information.  Defined sub-claims are `req_ip` (originating IP address) and `authn` (authentication method used).


## Actor Claim in Transaction Tokens {#actor-claim-in-transaction-tokens}

A Transaction Token is delegated for purposes of this document when it was issued for a business transaction initiated by a subject (`sub`) that is distinct from the workload currently processing the request, and when that workload is exercising the subject's authorization rather than acting under its own independent grant.  In a delegated Transaction Token, the `act` claim conforming to this profile MUST be included to represent the current acting party and any prior delegation steps, as specified in {{transaction-token-service-processing}}.  In non-delegated Transaction Tokens (those issued for a workload acting in its own right), `act` is OPTIONAL.

`req_wl` identifies the workload that requested the token from the TTS.  `act.sub` identifies the immediate acting party in the subject identifier namespace used by this profile.  The authoritative actor identifier for authorization decisions under this document is the outermost `act.sub`; `req_wl` is supporting workload context.

Claim semantics:

*  `sub`: identifies the original initiator.  When a Transaction Token is exchanged for a replacement, the new token MUST continue to refer to the same underlying subject.  The issuer MAY change `sub` only to re-express that same subject in a different identifier namespace.

*  `scope`: captures the transaction authorization intent for this token instance, per the TTS semantics.

*  `req_wl`: identifies the workload that requested this token instance from the TTS.

*  `act.sub` (outermost): identifies the immediate acting party.  When `req_wl` and the outermost `act.sub` identify the same entity, the issuer SHOULD ensure they are semantically consistent.  When a recipient relies on both and cannot reconcile them under local policy, the recipient MUST reject the token.

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

When a TTS receives a token-exchange request to issue or refresh a Transaction Token, and an inbound token contains actor-profile claims:

1.  The TTS MUST preserve `sub` to refer to the same underlying subject as the inbound token.  The TTS MAY change `sub` only to re-express that same subject in a different identifier namespace.  The TTS MUST NOT replace `sub` with an identifier for a different subject.

2.  The TTS MUST set `req_wl` to the authenticated requesting workload's identity per {{I-D.ietf-oauth-transaction-tokens}}.

3.  The TTS MUST verify that adding a new outermost `act` object would not cause the chain depth to exceed the limit in {{delegation-chains}}.  If it would, the TTS MUST reject the request with `invalid_request`.

4.  Before preserving or extending any inbound `act` chain, the TTS MUST validate the inbound actor information under local policy.  For the outermost inbound `act` object and for each nested inbound `act` object it preserves, the TTS MUST:

    *  validate the inbound token and trust the issuer that conveyed the chain;
    *  verify that each `act.iss` is authoritative for the identifier namespace of the corresponding `act.sub`, using local trust mechanisms equivalent in strength to those described in {{jwt-assertion-grants-processing}} step 3; and
    *  verify each delegation relationship it relies on as security-relevant context, using local policy equivalent in strength to {{jwt-assertion-grants-processing}} step 4.

    If the TTS cannot validate the inbound actor information it would preserve, it MUST reject the request with `invalid_grant`.  The TTS MUST NOT re-issue an inbound actor chain that it has not validated.

5.  The TTS MUST create a new outermost `act` object for the new presenter:

    *  The TTS MUST set `act.sub` to the new presenter's identifier.
    *  The TTS MUST set `act.iss` to the issuer namespace authoritative for the new presenter's identifier.  This obligation applies only to the `act` object the TTS creates for the current presenter.  The TTS MUST NOT rewrite `act.iss` or any other field in `act` objects inherited from the inbound token; those entries were authored by upstream ASes and their content is fixed at the point of authorship.
    *  If the inbound token already carries an `act` chain, the TTS MUST nest it beneath the new outermost `act`.
    *  The TTS SHOULD set `act.sub_profile` based on its knowledge of the new presenter's entity type.

6.  The TTS MUST bind the issued Transaction Token to the new presenter by setting the top-level `cnf` claim per the deployment's proof mechanism.  When the mechanism uses a public-key thumbprint, the TTS MUST set `cnf.jkt` to the new presenter's key.

    > Note: In WIMSE-based deployments, presenter binding is established via a Workload Identity Token (WIT) and Workload Proof Token (WPT) {{I-D.ietf-wimse-workload-creds}}{{I-D.ietf-wimse-wpt}}.

7.  The TTS MUST set `scope` per {{I-D.ietf-oauth-transaction-tokens}} and MAY include `tctx` or `rctx` as appropriate.  The `scope` of a Transaction Token captures the transaction-authorization intent for this token instance as defined by the TTS; it does not directly correspond to the OAuth access token scope that governs what the subject has authorized.  Accordingly, the least-privilege scope-reduction rule in {{jwt-access-token-propagation}} step 6 does not apply to Transaction Token issuance; the TTS determines the appropriate transaction scope according to its own policy and the semantics of {{I-D.ietf-oauth-transaction-tokens}}.

These same preservation rules apply regardless of whether the inbound credential is a JWT assertion grant, a JWT access token, or a Transaction Token, provided that the TTS supports issuing a Transaction Token from that input.


## Transaction Token Service Error Responses {#tts-error-responses}

When a TTS rejects a request for reasons related to actor-profile processing, it MUST return an OAuth error response per {{RFC6749, Section 5.2}} and {{RFC8693, Section 2.2}}.  The following error codes apply:

`invalid_request`:
: Use when the inbound token's `act` claim structure is syntactically invalid, the delegation chain depth would exceed the limit in {{delegation-chains}}, or a required actor-profile claim (`act.sub` or `act.iss`) is absent from an `act` object in the inbound token.

`invalid_grant`:
: Use when the inbound token fails validation, the `sub` identity cannot be preserved per step 1 of {{transaction-token-service-processing}}, or the inbound actor information that the TTS would preserve cannot be validated per step 4 of {{transaction-token-service-processing}}.

`access_denied`:
: Use when local TTS policy does not permit the requesting workload to act as an actor in the delegation chain for the identified subject, or when the `act.sub_profile` of an inbound actor is not accepted under local policy.

The `error_description` field SHOULD be included and SHOULD describe which aspect of actor-profile processing failed, to the extent permitted by the TTS's security and privacy policy.

Transaction Token support is advertised using the `transaction_token_supported` AS metadata parameter and the `transaction_token_required` Protected Resource Metadata parameter, both defined in {{I-D.ietf-oauth-transaction-tokens, Section 8}}.  This document does not redefine those parameters.  When an AS or TTS can issue Transaction Tokens as delegated outputs under this profile, it MUST list `urn:ietf:params:oauth:token-type:txn_token` in `actor_profile_token_types_supported`.


# Actor Authorization at the Resource Server {#dual-principal-authorization}

This section elaborates on the actor-authorization requirements referenced in step 5 of {{jwt-access-token-rs-processing}} and in {{actor-claim-in-transaction-tokens}}, and describes how resource servers SHOULD implement them.

## Authorization Model

When a token contains both `sub` and an `act` claim, a resource server has two independent principals available for authorization policy:

*  **Subject principal** (`sub`): the party whose authorization is being exercised.  This principal typically has a relationship with the resource (e.g., an account, a role, a permission).

*  **Actor principal** (`act.sub`): the party that is making the immediate request.  This principal may be in a different organizational domain and trust level from the subject.

Actor authorization at the resource server evaluates both principals and the relationship between them (i.e., that the actor is authorized to act on behalf of the subject) before granting access.  This specification recommends this subject-plus-actor evaluation model for delegated tokens, but does not require every RS to use it in every deployment.

For Transaction Tokens, the primary policy pair remains (`sub`, `act.sub`).  The `req_wl` claim provides workload context from the TTS and is not a replacement for `act.sub`.  Nested `act` objects provide prior-actor context for audit, policy refinement, or chain validation.

This specification defines subject-plus-actor evaluation as the interoperable baseline.  Deployments MAY apply multi-principal authorization under local policy by considering one or more nested `act` objects as additional authorization, trust, or risk inputs in addition to `sub` and the outermost `act.sub`.  This specification does not require or standardize such evaluation, and clients MUST NOT assume that nested actors will be used for authorization unless deployment-specific agreements say otherwise.

## Resource Server Processing {#dual-principal-rs-processing}

Actor authorization is OPTIONAL but RECOMMENDED for delegated tokens under this profile.  The following steps describe the RECOMMENDED evaluation:

1.  **Advertise requirements**: An RS that requires actor authorization MUST set `actor_authorization_required: true` ({{protected-resource-metadata}}).  An RS that does not advertise this value MAY still apply subject-plus-actor evaluation, but clients MUST NOT rely on that behavior.

2.  **Evaluate subject authorization**: Determine whether `sub` has been granted the requested scope or permission, using the same mechanisms applied to non-delegated tokens.

3.  **Evaluate actor authorization**: Determine whether the outermost `act.sub` is permitted to exercise the subject's authorization.  This evaluation MAY be performed against:

    *  a registered delegation policy for the (subject, actor) pair,
    *  the actor's `sub_profile` (e.g., only AI agents from a trusted domain are permitted to act as delegatees),
    *  the token's `scope` claim.

    For Transaction Tokens, the RS SHOULD evaluate `req_wl` as supporting context.  If the RS relies on both `req_wl` and `act.sub` to identify the current presenter and cannot reconcile them under local policy, it MUST reject the request.

4.  **Evaluate combined policy**: Apply resource-specific subject-plus-actor policies (e.g., requiring both principals to have agreed to terms of service).

5.  If the RS requires actor authorization but cannot complete it, it MUST reject the request.  An RS that does not require actor authorization SHOULD still log delegated actor information for audit purposes.

When a deployment applies multi-principal authorization under local policy, the outermost `act.sub` remains the baseline interoperable actor identifier, while nested `act` objects are additional local-policy inputs only.  Failure semantics, ordering, and weighting for those nested actors are deployment-specific.

## Policy Claims Summary

The following claims are available as subject-plus-actor policy inputs:

| Claim | Principal | Description |
|-------|-----------|-------------|
| `sub` | Subject | Original authorized principal |
| `sub_profile` | Subject | Entity type of the subject |
| `act.sub` | Actor | Immediate acting principal |
| `act.sub_profile` | Actor | Entity type of the actor identified by `act.sub` |
| `act.cnf.jkt` | Actor | Actor-associated key reference for the actor identified by `act.sub` |
| `scope` | Both | Authorized scope |
| `req_wl` (Txn-Token) | Actor | Workload that requested the Transaction Token from the TTS; supporting context alongside, but not a substitute for, the outermost `act.sub` |


# OAuth Entity Profile Usage {#entity-profile-extension}

{{I-D.mora-oauth-entity-profiles}} defines the OAuth Entity Profiles mechanism, including the `sub_profile` and `client_profile` JWT claims, the `entity_profiles_supported` AS metadata parameter with its `client`, `subject`, and `actor` arrays, and an IANA registry of entity profile values with associated usage locations, including the "Actor Profile" usage location.  It also explicitly defines `sub_profile` within `act` (Actor) nodes per {{RFC8693}} for classification of delegated actors in delegation chains.

This document uses the actor profile support defined in {{I-D.mora-oauth-entity-profiles}}.  The `sub_profile` claim within an `act` object classifies the entity identified by `act.sub`, using values registered with the "Actor Profile" usage location in the "OAuth Entity Profiles" registry.  The `entity_profiles_supported.actor` array in AS metadata advertises which actor entity profile values are accepted, as defined in {{I-D.mora-oauth-entity-profiles}}.

The values registered for actor use are defined in {{I-D.mora-oauth-entity-profiles}}.  Values within `act.sub_profile` MUST be either registered with the "Actor Profile" usage location in the "OAuth Entity Profiles" registry or privately defined collision-resistant values (see {{actor-object-structure}}).  When processing `act.sub_profile`, issuers and consumers MUST treat registered values according to the entity profile semantics defined in {{I-D.mora-oauth-entity-profiles}} and MUST NOT reject tokens solely because a value is unrecognized (see {{forward-compat-sub-profile}}).

This document makes no independent requests to the "OAuth Entity Profiles" registry; all actor-profile-related registry definitions are provided by {{I-D.mora-oauth-entity-profiles}}.

# Discovery and Capability Negotiation {#discovery-capability-negotiation}

## Overview

This section defines a minimal set of metadata parameters that, together with the `entity_profiles_supported.actor` array defined in {{I-D.mora-oauth-entity-profiles}}, allow authorization servers and resource servers to advertise actor-profile support without out-of-band configuration.  These parameters provide partial capability negotiation only; they do not provide a complete machine-readable description of every supported grant path or method by which the AS determines the acting party.  Capability negotiation is only a compatibility preflight and does not guarantee successful token issuance or resource access.

The design principle is that **capability flags go in this spec's metadata; entity type enumeration goes in {{I-D.mora-oauth-entity-profiles}} metadata**.  Clients MUST use `entity_profiles_supported.actor` per {{I-D.mora-oauth-entity-profiles}} to determine which actor entity profiles the associated AS accepts for interoperability under this profile, and MUST use `actor_profile_token_types_supported` ({{authorization-server-metadata}}) to determine which delegated output token types the AS can issue under this profile.


## Authorization Server Metadata {#authorization-server-metadata}

One new parameter is defined for use in the AS metadata document ({{RFC8414}}):

`actor_profile_token_types_supported`:
: OPTIONAL.  A JSON array of token-type URI strings indicating the delegated output token types that the AS can issue while applying actor-profile processing as defined in this document.  When a token type appears in this array, the AS can produce that token type as an output with actor-profile claims preserved or created according to this document.  This parameter does not by itself indicate every accepted input token type for a transformation; clients MUST combine it with `grant_types_supported`, deployment documentation, and the processing rules of the relevant token type before assuming that a particular exchange path is supported.  Defined values are:

  - `urn:ietf:params:oauth:token-type:access_token` — JWT access tokens ({{jwt-access-tokens}})
  - `urn:ietf:params:oauth:token-type:jwt` — JWT assertion grants ({{jwt-assertion-grants}})
  - `urn:ietf:params:oauth:token-type:txn_token` — Transaction Tokens ({{transaction-tokens}})

  When absent, the AS makes no claim about delegated output token types under this profile.

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
: OPTIONAL.  A boolean.  When `true`, the RS requires that tokens presented for this resource carry an `act` claim conforming to the actor profile defined in this document.  The RS MUST reject tokens that omit `act` or that carry `act` claims not conforming to this profile.  When `false` or absent, actor-profile conformance is not required.

`actor_authorization_required`:
: OPTIONAL.  A boolean.  When `true`, the RS advertises that it requires evaluation of the actor as an authorization input in addition to the subject, as described in {{dual-principal-authorization}}.  An RS that sets this value to `true` implicitly requires that presented tokens carry a conforming `act` claim; the RS MUST also treat `actor_profile_required` as `true` and MUST reject tokens that omit `act` or that carry non-conforming `act` claims.  When `false` or absent, the RS makes no declaration to clients about whether actor authorization is applied, and clients MUST NOT rely on that behavior unless the value is `true`.

Clients discover which actor entity profile values the RS's AS will accept by consulting `entity_profiles_supported.actor` in the AS metadata for the AS listed in the resource's `authorization_servers`.

Example Protected Resource Metadata fragment:

~~~json
{
  "resource": "https://api.travel-provider.example",
  "authorization_servers": [
    "https://as.travel-provider.example"
  ],
  "actor_profile_required": true,
  "actor_authorization_required": true
}
~~~

## Capability Negotiation Flow

The following steps describe a RECOMMENDED capability-negotiation pattern a client agent can use as a preflight check to evaluate whether a cross-domain delegation is likely to be feasible:

1.  **Discover target resource metadata**: The client fetches Protected Resource Metadata ({{RFC9728}}) for the target API and reads `actor_profile_required`, `actor_authorization_required`, and `authorization_servers`.

2.  **Discover target AS metadata**: The client fetches AS metadata ({{RFC8414}}) for the listed AS and reads:

    *  `actor_profile_token_types_supported` to determine whether the desired delegated output token type is advertised.
    *  `entity_profiles_supported.actor` to confirm the client's own entity profile (its `sub_profile` value) is in the accepted list.

3.  **Proceed or abort**: If `actor_profile_required` is `true` at the RS and the client's entity profile is not listed in `entity_profiles_supported.actor` at the AS, the client MUST NOT proceed with a delegation-based request and SHOULD surface the capability mismatch to the invoking system.  If the desired delegated output token type is not listed in `actor_profile_token_types_supported`, the client MUST treat that output as unsupported.  If `actor_authorization_required` is `true`, the client SHOULD expect the RS to require actor evaluation in addition to subject evaluation and SHOULD acquire a token whose actor information can satisfy that policy.  If the listed metadata is insufficient to determine whether the AS supports the needed input grant or exchange path, the client SHOULD treat support as indeterminate and rely on deployment-specific knowledge or a trial request.

    As a safe default, a client that intends to make a request on behalf of another principal SHOULD treat `actor_profile_required: true` as meaning that it needs an explicit `act`-carrying token for that resource.  Clients SHOULD prefer acquisition paths that explicitly carry actor information over paths that depend on AS-derived actor determination when both are available.

4.  **Construct and submit token exchange or assertion grant**: The client proceeds per {{jwt-assertion-grants}} for JWT assertion grants or per {{RFC8693}} for token-exchange requests.  When the delegated request explicitly carries actor-profile claims and `act.sub_profile` is included, its value MUST be drawn from the `entity_profiles_supported.actor` accepted list.  When actor information is derived by the AS from authenticated client context or other local policy, the client SHOULD verify — via deployment documentation, prior test requests, or AS metadata — that the acting entity corresponds to an accepted actor profile before relying on the AS to issue a delegated token under this profile.

5.  **RS validation**: The RS validates the resulting token according to the rules for that token type and applies local delegated-token policy.  If the RS requires actor authorization, it applies that policy per {{dual-principal-authorization}}.  For JWT access tokens, see {{jwt-access-token-rs-processing}}.  For Transaction Tokens, see {{I-D.ietf-oauth-transaction-tokens}} together with {{transaction-tokens}} of this document.

Example client preflight failure:

If the RS metadata advertises `"actor_profile_required": true`, but the target AS metadata advertises `"entity_profiles_supported": { "actor": ["service"] }` and the client's acting entity profile is `ai_agent`, the client MUST abort before making the token request because the AS does not advertise support for the actor type the client would need to represent.

# Deployment Considerations

This section provides guidance for deploying the OAuth Actor Profile in systems that currently rely on implicit delegation or client-identity-based actor inference.

## Migration from Implicit to Explicit Delegation {#migration-implicit-explicit}

The invariant of this document is that `client_id` identifies the OAuth client registration, `sub` identifies the authorizing principal, and `act.sub` is the authoritative actor identity signal when delegation is present.  Migration from implicit delegation is the process of making this distinction explicit in tokens where these roles were previously conflated through `client_id` or inferred from token-request context.

Deployments that currently rely on implicit delegation can migrate incrementally to this profile.  During migration, existing client-oriented inputs such as `client_id`, `azp`, and authenticated client context MAY remain in use, but the outermost `act.sub` becomes the authoritative explicit delegation signal whenever `act` is present.  The expected transition pattern is to emit both legacy client-oriented identifiers and explicit actor claims during rollout, measure and reconcile mismatches, and then require `act` where explicit delegation is needed.

The safe migration default is:

*  If `act` is present, use the outermost `act.sub` as the authoritative delegated-actor signal.
*  Clients that can obtain explicit-delegation tokens under this profile SHOULD prefer those tokens over relying on legacy implicit client-identity interpretation.
*  If `act` is absent, deployments MAY continue to apply legacy implicit client-based policy according to local policy, and existing client-based authorization logic MAY remain in place during migration.
*  If both explicit and implicit signals are present and local policy expects them to identify the same party under trusted local mapping rules, they MUST reconcile under those rules or the request MUST be rejected.
*  If both are present and no such equivalence rule exists, implementations SHOULD treat them as distinct identifiers with different semantics rather than infer equivalence.

During transition, issuers SHOULD emit both legacy client-oriented identifiers and explicit actor claims whenever doing so is feasible for the deployment, and they SHOULD log whether delegation was processed explicitly, implicitly, or inconsistently.

Legacy implicit form:

~~~json
{
  "iss": "https://as.example.com",
  "sub": "https://idp.example.com/users/alice",
  "client_id": "travel-assistant-client-id",
  "azp": "travel-assistant-client-id",
  "scope": "booking:create"
}
~~~

Explicit form:

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

`client_id` and `azp` remain as auxiliary client-identity inputs; `act.sub` is the authoritative delegation signal.  When local policy expects both to identify the same party under trusted local mapping rules, they MUST reconcile or the request MUST be rejected.

Mismatch example, where the client and actor identify different parties:

~~~json
{
  "iss": "https://as.example.com",
  "sub": "https://idp.example.com/users/alice",
  "client_id": "travel-assistant-client-id",
  "act": {
    "sub": "https://agents.example.com/concierge-bot",
    "iss": "https://agents.example.com",
    "sub_profile": "ai_agent"
  },
  "scope": "booking:create"
}
~~~

An AS or RS that expected the client and actor to identify the same party under trusted local mapping rules MUST reject this token unless those rules explicitly bind `travel-assistant-client-id` to `https://agents.example.com/concierge-bot`.  If no such mapping rule exists, the identifiers MUST be treated as distinct.

# Security Considerations

## Delegation Chain Integrity and Trust {#delegation-chain-integrity}

An attacker who can inject or forge `act` claims can impersonate an arbitrary actor and exercise a subject's permissions without authorization.  The primary mitigation is to accept `act` claims only in tokens whose issuer is trusted to assert the delegated actor relationship.  RS implementations MUST validate the token signature before extracting actor claims, and MUST verify that the token issuer is trusted to convey the claims it carries.

Because inner `act` objects are set by upstream ASes and not re-signed at each hop, the integrity of the entire delegation chain rests on the outermost token's signature.  Implementations SHOULD use short token lifetimes and MUST reject tokens whose `exp` has passed, regardless of chain depth.

The `act.iss` values in inner `act` objects are assertion-based — set by whoever constructed those objects at an earlier hop, not by the issuer of the current token.  Implementations MUST NOT treat an inner `act.iss` as independently authenticated merely because it appears in the token; the trust basis is the outer token issuer's endorsement.  Consequently: an RS that relies on inner `act.iss` for audit or policy MUST do so only when it trusts the outer issuer to have validated and faithfully propagated that chain; ASes that propagate inner chains MUST independently validate each inner `act.sub` and `act.iss` before endorsing them (see {{jwt-assertion-grants-processing}} step 5); and security policies relying on inner actor identities for access control SHOULD be treated as lower-assurance than policies based on the outermost `act.sub`.

When a token crosses organizational boundaries, the receiving AS or RS MUST apply appropriate trust evaluation.  ASes performing token exchange MUST evaluate cross-domain delegation grants explicitly and MUST NOT implicitly grant cross-domain actors the same rights as same-domain actors.

## Client Identity and Delegation {#client-identity-delegation}

Client identity, such as `client_id`, `azp`, or authenticated client context, is widely used in deployed systems as an authorization input.  Under this document, those values remain auxiliary client-identity signals, while the outermost `act.sub` is the explicit delegated-actor signal when present.  The following normative rules apply:

*  Implementations MUST NOT treat `client_id`, `azp`, or other client-identity signals as a substitute for `act` when `act` is present.
*  When a single `client_id` registration fronts multiple distinct acting entities (for example, an agent orchestration platform executing requests on behalf of different agent instances), each request MUST carry `act.sub` identifying the specific acting principal.  Using `client_id` alone to distinguish actors is insufficient and MUST NOT be relied upon.
*  During token issuance, `client_id` and `azp` MUST NOT be rewritten to represent delegation state that belongs in `act`; see {{jwt-access-token-propagation}} for propagation rules.
*  When both explicit (`act.sub`) and implicit (`client_id`, `azp`) signals are present and local policy expects them to identify the same party, they MUST reconcile under trusted local mapping rules or the request MUST be rejected.

The detailed migration rules and transition patterns are defined in {{migration-implicit-explicit}}.

## Actor Identity Binding

Without top-level presenter proof of possession, a leaked token can be replayed by any party.  When an `act.cnf.jkt` is present:

*  The AS that set this value SHOULD have verified the actor's possession of the corresponding key before including it.
*  The RS SHOULD require the presenter-proof mechanism appropriate to the token type and deployment for the top-level `cnf.jkt` or other top-level confirmation information.  For example, JWT access tokens commonly use DPoP or mTLS, while Transaction Tokens can use the workload proof mechanism defined by their deployment profile.
*  The RS MAY use `act.cnf` as audit, diagnostic, or policy input, but MUST NOT treat nested `act.cnf` values as independently verifiable presenter bindings for prior hops.

Deployments of AI agent systems SHOULD require sender-constrained tokens for the current presenter to prevent delegation token theft, and MAY carry `act.cnf` for actor-chain visibility.

Inner `act.cnf` values do not by themselves provide independently verifiable provenance for prior actor hops across trust boundaries.  An intermediate issuer that is trusted to issue a new token can also rewrite nested actor-chain content, including inner `act.cnf` values.  Accordingly, inner `act.cnf` is useful for carrying actor-associated key context from earlier hops, but it does not solve cross-domain integrity or non-repudiation for those hops.  Stronger assurance for prior-hop provenance would require an additional mechanism outside the scope of this document, such as signed hop receipts, transparency-log-based recording, or another future extension.

## Delegation Depth Limits

Unbounded delegation chains increase attack surface and complicate policy evaluation.  This document specifies a default maximum of five nested `act` objects ({{delegation-chains}}), which covers realistic multi-tier architectures while bounding parsing complexity.  Implementations that encounter chains exceeding this limit MUST reject the token to prevent denial-of-service through chain parsing.

## Sub_profile Trust

The `sub_profile` claim is asserted by the token issuer and is only as trustworthy as that issuer.  Resource servers MUST NOT trust `sub_profile` values in tokens issued by untrusted parties.  Resource server operators SHOULD configure a list of accepted entity-type profiles per trust domain.

## Delegation Revocation

Token revocation ({{RFC7009}}) applies to individual tokens but does not revoke an underlying delegation relationship or invalidate already-issued downstream tokens in a delegation chain.  When Alice revokes her delegation to an agent, access tokens already issued to downstream actors remain valid until their `exp` time.  Short token lifetimes are the primary mitigation; see {{I-D.ietf-oauth-security-topics}} for general access token lifetime guidance.

Authorization servers SHOULD maintain a delegation grant registry mapping (subject, actor) pairs to active delegation relationships. When a delegation grant is revoked, the AS MUST refuse to issue new tokens for that (subject, actor) pair.  Because refresh tokens SHOULD NOT be issued for cross-domain delegated assertion grants (see {{assertion-grant-refresh-tokens}}), revocation of the delegation relationship typically takes effect at the next assertion grant presentation.  If an AS has issued refresh tokens in a delegated context, it MUST proactively revoke them when the delegation relationship is revoked.

Resource servers in security-sensitive deployments SHOULD use token introspection ({{RFC7662}}, {{token-introspection}}) rather than local JWT validation so that revocation state is checked on each request. Implementations MUST NOT use delegation chain depth as a rationale for skipping revocation checks.

Operators deploying AI agent systems MUST provide end-users with a mechanism to enumerate and revoke active delegation grants.

## Confused Deputy

A resource server that evaluates only the subject principal when an `act` claim is present is susceptible to a confused deputy attack: a malicious actor exploits a subject's pre-existing permissions without the subject's ongoing consent simply by presenting a token that names the subject in `sub`.  The mitigation is actor authorization — evaluating both `sub` and `act.sub` before granting access.  Resource servers SHOULD implement actor authorization for delegated tokens under this document.

## Actor-Authorization Bypass

A resource server that advertises `actor_authorization_required: true` but fails to enforce actor evaluation on every code path allows an attacker to bypass that requirement by exploiting gaps in the enforcement logic.  Resource servers that require actor authorization MUST advertise that behavior using `actor_authorization_required: true` and MUST apply the evaluation on every request path that accepts delegated tokens, including introspection-based validation paths.  Deployments that choose not to require actor authorization SHOULD follow the narrowly scoped and explicitly documented informational-only cases described in {{dual-principal-authorization}}.

## Subject Namespace Translation

Several processing rules in this document permit an AS to change the `sub` value when re-expressing the same subject in a different identifier namespace (see {{jwt-access-token-propagation}} step 2 and {{transaction-token-service-processing}} step 1).  This creates a subject-substitution risk: a malicious or misconfigured AS could map `sub` to a different entity in the new namespace, silently replacing the authorized principal with a different one.

Mitigations:

*  Receiving ASes and RSes MUST NOT accept `sub` translations from ASes they do not trust to authoritatively map identifiers between the two namespaces.  Trust for namespace mapping is separate from trust for token signing and MUST be established explicitly in local policy.
*  When an AS translates `sub`, it SHOULD include both the original and translated identifiers or use a `sub_profile` value that lets the RS verify the namespace mapping, where local policy requires such verification.
*  RSes that enforce access control against a specific `sub` value SHOULD verify that the issuing AS is authoritative for the subject-identifier namespace used, and SHOULD NOT accept subject identifiers from namespaces for which the issuing AS is not authoritative.

## Token Substitution

An attacker who can present a token with a crafted `sub_profile` or actor chain could attempt to escalate privileges.  ASes MUST validate inbound `sub_profile` values against the syntax requirements of this document, the applicable registry or deployment-specific allowed set where such checks are part of local policy, and the local policy applicable to the token they are issuing.  They MUST preserve unrecognized but syntactically valid values as required by {{forward-compat-sub-profile}}, and they MUST reject values that are malformed or disallowed by local policy.


# Privacy Considerations {#privacy}

Delegation chains can reveal sensitive information about user behavior, enterprise topology, software suppliers, and internal tool composition. Issuers therefore SHOULD disclose only the actor information needed by the relying party for authorization, audit, or policy enforcement.

Cross-domain deployments SHOULD prefer stable but non-reassigned identifiers and SHOULD consider pairwise identifiers for human subjects when a globally correlatable identifier is not required by the use case.

When the same logical entity can appear in different identifier namespaces, such as `azp`, `req_wl`, and `act.sub`, issuers and relying parties SHOULD use explicit issuer scoping and locally trusted mapping rules rather than string equality alone to determine whether those identifiers refer to the same entity.

Issuers SHOULD minimize disclosure of prior actors by audience and token-design decisions made before issuance.  Once an issuer chooses to preserve a delegation chain in a token under this profile, it SHOULD preserve the validated chain intact for that token.  If local privacy requirements would require omitting a chain element that would otherwise be security-relevant to the recipient's evaluation, the issuer SHOULD reject the request rather than silently truncating the chain.

The `txn` claim in Transaction Tokens ({{I-D.ietf-oauth-transaction-tokens}}) is a stable, globally unique identifier shared across all tokens in a single business transaction.  When Transaction Tokens cross organizational boundaries, `txn` enables cross-domain correlation of all service calls within a transaction by any party that observes multiple tokens.  Transaction Token Services SHOULD NOT propagate the same `txn` value into tokens presented to resource servers outside the originating trust domain unless that relying party is explicitly authorized to correlate the transaction.  Where cross-domain correlation is not required for the authorization decision, the TTS SHOULD derive a domain-scoped transaction identifier that cannot be linked back to the originating `txn` value.  Implementations MUST treat `txn` values as sensitive identifiers and MUST NOT include them in logs or audit records accessible to parties outside the delegation chain without the subject's consent.

The `req_wl` claim in Transaction Tokens can also expose sensitive information about internal workload topology and service composition.  Transaction Token Services SHOULD disclose `req_wl` only to relying parties that need that information for authorization, audit, or policy enforcement, and SHOULD avoid propagating internal-only workload identifiers across trust-domain boundaries unless such disclosure is explicitly required by the deployment.


# IANA Considerations

## OAuth Authorization Server Metadata Registry

This document requests IANA to register the following value in the "OAuth Authorization Server Metadata" registry ({{RFC8414}}):

*  Metadata Name: `actor_profile_token_types_supported`
*  Metadata Description: JSON array of delegated output token-type URIs that the AS can issue while applying actor-profile processing per this document
*  Change Controller: IETF
*  Reference: {{authorization-server-metadata}} of this document


## OAuth Protected Resource Metadata Registry

This document requests IANA to register the following values in the "OAuth Protected Resource Metadata" registry ({{RFC9728}}):

*  Metadata Name: `actor_profile_required`
*  Metadata Description: Boolean indicating whether the RS requires delegated requests to use tokens carrying an `act` claim conforming to this profile
*  Change Controller: IETF
*  Reference: {{protected-resource-metadata}} of this document

*  Metadata Name: `actor_authorization_required`
*  Metadata Description: Boolean indicating whether the RS requires actor authorization in addition to subject authorization per this document
*  Change Controller: IETF
*  Reference: {{protected-resource-metadata}} of this document


## OAuth Entity Profiles Registry {#iana-entity-profiles}

This document makes no independent requests to the "OAuth Entity Profiles" registry.  It normatively depends on the "Actor Profile" usage location, the `actor` array in `entity_profiles_supported`, and the registration of `user`, `service`, and `ai_agent` with that usage location — all of which are defined and requested by {{I-D.mora-oauth-entity-profiles}}.  The IANA actions for those entries are contingent on the progression of {{I-D.mora-oauth-entity-profiles}}.




--- back

# Cross-Domain AI Agent Flow: ID Token to Transaction Token {#appendix-cross-domain}

This appendix traces a single user request across two trust domains, highlighting the actor-profile claim structures and processing requirements specific to this document.  Standard validation steps (JWT signature verification, sender-constrained access token proof, Transaction Token presenter proof, and Token Exchange mechanics) are delegated to the underlying token specifications and deployment profile.

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
                                                 │ (6) Transaction Token + WIMSE proof
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

Presenter key bindings:

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
The agent confirms that its `sub_profile` (`ai_agent`) is listed in `entity_profiles_supported.actor`, that both `jwt` and `access_token` are covered by `actor_profile_token_types_supported`, and that the Booking Tool RS requires explicit actor-profile support and actor authorization.


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

The agent presents Alice's ID Token to the Enterprise IdP AS using Token Exchange.  The Enterprise IdP AS authenticates the client as `travel-assistant-client-id`, verifies that the ID Token audience matches that client, and uses local delegation policy plus the authenticated client context to construct the actor-profile claims in the issued ID-JAG:

~~~
POST /token HTTP/1.1
Host: as.enterprise.example
Content-Type: application/x-www-form-urlencoded
DPoP: <AgentJKT-proof>

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token=<alice-id-token>
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aid_token
&requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aid-jag
&audience=https%3A%2F%2Fas.travel-provider.example%2F
&resource=https%3A%2F%2Fas.travel-provider.example
&scope=booking%3Acreate
&client_id=travel-assistant-client-id
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer
&client_assertion=<travel-assistant-client-assertion>
~~~

The Enterprise IdP AS applies scope reduction and validates the client-bound proof-of-possession according to RFC 9449.  In this example, it binds the issued ID-JAG to the key demonstrated in the DPoP proof, determines from local policy that the authenticated client is the delegated actor for Alice in this flow, and issues the ID-JAG as a JWT output of token exchange with the actor chain established per {{actor-profile}}:

~~~json
{
  "iss": "https://as.enterprise.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
  "client_id": "travel-assistant-client-id",
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

The `act` object records the agent as the authorized actor.  The `client_id` and `azp` values identify the OAuth client used in the exchange, while `act.sub` identifies the delegated actor.  In this example they all refer to the same party under trusted local mapping rules.  Both `cnf.jkt` (top-level, for DPoP binding) and `act.cnf.jkt` (actor-associated key reference) are set to `AgentJKT` because the agent is both the bearer and the acting principal at this stage.


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

The Travel Provider AS performs actor-profile processing per {{jwt-assertion-grants-processing}}: it verifies the request's DPoP proof against the current presenter binding and checks that `act.sub_profile` (`ai_agent`) is permitted as an actor for the requested scope.  It issues an access token preserving the actor chain:

~~~json
{
  "iss": "https://as.travel-provider.example",
  "sub": "https://idp.enterprise.example/users/alice",
  "sub_profile": "user",
  "client_id": "travel-assistant-client-id",
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

Alice's `sub` and `sub_profile` are preserved verbatim from the ID-JAG ({{jwt-access-token-propagation}}).  The Travel Provider AS does not translate or substitute the enterprise subject identifier.  The `client_id` and `azp` values continue to identify the OAuth client, but they do not replace `act.sub` as the authoritative delegated-actor identifier.


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

The Booking Tool RS applies actor authorization ({{dual-principal-authorization}}): it evaluates both Alice (`sub`, `sub_profile: user`) and the Travel Assistant (`act.sub`, `sub_profile: ai_agent`). Because the RS advertises `actor_authorization_required: true`, both principals MUST be independently authorized.  The `act.sub_profile` value is checked against `entity_profiles_supported.actor` per {{entity-profile-extension}}.


## Step 5: Booking Tool Exchanges Access Token for Transaction Token

The Booking Tool cannot reuse the received access token for internal calls: it is sender-constrained to `AgentJKT`, which the Booking Tool does not possess.  It requests a Transaction Token from the TTS.  In this example, the TTS authenticates the Booking Tool using a WIMSE Workload Identity Token (WIT) and a Workload Proof Token (WPT).  The WIT identifies the Booking Tool and carries its confirmation key, while the WPT proves possession of that key and binds the request to the accompanying access token:

~~~
POST /token HTTP/1.1
Host: tts.travel-provider.example
Content-Type: application/x-www-form-urlencoded
Workload-Identity-Token: <booking-tool-wit>
Workload-Proof-Token: <tool-wpt-with-wth-and-ath>

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token=<tp-access-token>
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aaccess_token
&requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Atxn_token
&audience=https%3A%2F%2Ftravel-provider.example
&scope=inventory%3Acheck
&rctx={"req_ip":"198.51.100.42"}
~~~

The TTS applies actor-profile processing per {{transaction-token-service-processing}}: it preserves `sub` and `sub_profile` from the subject token, sets `req_wl` to the authenticated Booking Tool, creates a new outermost `act` object for the Booking Tool, nests the subject token's existing `act` claim beneath it, and binds the issued Transaction Token to the Booking Tool's presenter key (`ToolJKT`) identified in the WIT confirmation claim:

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

The presenter binding rotates at this step: `cnf.jkt` is now `ToolJKT`, and the outermost `act.cnf.jkt` matches it because the Booking Tool is now the current actor.  The nested `act.act.cnf.jkt` retains the agent's original key, illustrating the multi-hop key-rotation property described in {{sender-constraint}}.


## Step 6: Booking Tool Calls Inventory Service

~~~
GET /inventory?origin=SFO&dest=NYC&depart=2026-04-15 HTTP/1.1
Host: internal.travel-provider.example
Txn-Token: <txn-token>
Workload-Identity-Token: <booking-tool-wit>
Workload-Proof-Token: <tool-wpt-with-wth-and-tth>
~~~

The Inventory Service validates the WIT and WPT according to the WIMSE specifications: the WPT proves possession of the key identified by the WIT, `wth` binds the proof to the presented WIT, and `tth` binds it to the presented Transaction Token.  The Inventory Service then applies actor authorization ({{dual-principal-authorization}}): Alice (`sub`) governs data access policy (e.g., travel tier); the Booking Tool (`act.sub`) is the authorized internal workload.  The `req_wl` claim provides consistent TTS workload context for the same service in this example.  The nested `act.act.sub` (Travel Assistant) is carried as prior delegation context and is not evaluated for access control at this internal tier, consistent with the guidance on inner actors in {{dual-principal-rs-processing}}.


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

*  In this example, `sub` (Alice) is unchanged across all trust domains and token transformations.
*  The presenter-binding key rotates once, at Step 5 when the TTS re-binds the Transaction Token to the Booking Tool's key.
*  At Step 5 the TTS creates a new outermost `act` for the Booking Tool and nests the prior `act` chain beneath it.


# Acknowledgments
{:numbered="false"}

The author thanks the OAuth Working Group for the foundational work on Token Exchange (RFC 8693), JWT-formatted access tokens (RFC 9068), DPoP (RFC 9449), and Transaction Tokens, upon which this document builds. The motivating use cases for this work emerged from the deployment of AI agent systems that require cross-domain delegation with explicit actor chains and carried-forward delegation state within the trust model of the issuing domains.

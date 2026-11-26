# Cross-Domain AI Agent Flow: ID Token to Transaction Token

This example follows a delegated request across two trust domains.  Token validation and presenter proofs follow the underlying token specifications and the deployment profile.

All claim values, JWK thumbprints, and domain names are synthetic.  Long HTTP example lines use the backslash convention defined in [RFC8792](https://www.rfc-editor.org/rfc/rfc8792.html); other line breaks in form bodies are for readability only.

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
                       | (3) jwt-dpop grant
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

Before initiating the flow, the agent consults the following Travel Provider AS metadata ([Metadata and Discovery](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-profile.html#metadata-and-discovery)) as an advisory compatibility check:

~~~json
{
  "issuer": "https://as.travel-provider.example",
  "grant_types_supported": [
    "urn:ietf:params:oauth:grant-type:jwt-bearer",
    "urn:ietf:params:oauth:grant-type:jwt-dpop",
    "urn:ietf:params:oauth:grant-type:token-exchange"
  ],
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:id-jag",
    "urn:ietf:params:oauth:grant-profile:actor-profile"
  ],
  "entity_profiles_supported": {
    "subject": ["user", "ai_agent"],
    "actor":   ["user", "ai_agent", "service"]
  }
}
~~~

The agent confirms that its `sub_profile` (`ai_agent`) is in `entity_profiles_supported.actor` and that the Travel Provider AS advertises ID-JAG and actor-profile grant processing in `authorization_grant_profiles_supported`, which covers the `jwt-dpop` grant in Step 3.  The Token Exchanges in Steps 2 and 5 happen at the Enterprise IdP AS and the TTS, each of which advertises its own capabilities in its own metadata.  These signals are only coarse compatibility indicators.


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

The agent presents Alice's ID Token as the `subject_token` in a Token Exchange request to the Enterprise IdP AS, requesting an ID-JAG ([draft-ietf-oauth-identity-assertion-authz-grant](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-identity-assertion-authz-grant)).  The agent's client assertion ([RFC7523](https://www.rfc-editor.org/rfc/rfc7523.html)) serves as both the `client_assertion` (for client authentication) and the `actor_token` (for actor identity), as described in [JWT Client Assertion](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-profile.html#jwt-client-assertion-as-actor-token).  The Enterprise IdP AS authenticates the client, verifies that the ID Token audience matches that client, and uses local delegation policy to construct the issued ID-JAG.  The following is an example of the Token Exchange request:

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

The agent presents the DPoP-bound ID-JAG with the `jwt-dpop` grant type ([draft-parecki-oauth-jwt-dpop-grant](https://datatracker.ietf.org/doc/html/draft-parecki-oauth-jwt-dpop-grant)) to the Travel Provider AS and authenticates as the client registered there with a separate client assertion, whose `aud` identifies the Travel Provider AS.  The Travel Provider AS processes the grant as an ID-JAG per [draft-ietf-oauth-identity-assertion-authz-grant](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-identity-assertion-authz-grant) with the actor-profile rules in [JWT Assertion Grants](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-profile.html#jwt-assertion-grants):

~~~
NOTE: '\' line wrapping per RFC 8792

POST /token HTTP/1.1
Host: as.travel-provider.example
Content-Type: application/x-www-form-urlencoded
DPoP: <AgentJKT-proof>

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-dpop
&assertion=<id-jag>
&scope=booking%3Acreate
&client_id=https%3A%2F%2Fagents.enterprise.example%2Ftravel-assistant
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A\
  client-assertion-type%3Ajwt-bearer
&client_assertion=<travel-assistant-client-assertion-for-provider>
~~~

The Travel Provider AS makes the two client checks that ID-JAG specifies ([Sections 4.4 and 4.4.1](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-identity-assertion-authz-grant#section-4.4)): it authenticates the client from its client assertion, and it verifies that the ID-JAG's `client_id` identifies that authenticated client.  These checks bind the OAuth client; the actor comes from the ID-JAG's `act` claim.  The Travel Provider AS then performs actor-profile processing per [JWT Assertion Grants](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-profile.html#jwt-assertion-grants): it verifies the request's DPoP proof against the top-level `cnf.jkt` in the inbound ID-JAG and checks that `act.sub_profile` (`ai_agent`) is permitted as an actor for the requested scope under local policy.  It issues the following access token, which preserves the delegation chain:

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

The Booking Tool RS authorizes the (`sub`, outermost `act.sub`) pair ([Resource Server Processing](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-profile.html#resource-server-processing)): it evaluates Alice (`sub`, `sub_profile: user`) together with the Travel Assistant (`act.sub`, `sub_profile: ai_agent`) for the requested operation.  The RS checks the `act.sub_profile` value against its own actor policy ([Actor Authorization](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-profile.html#actor-authorization)).


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

The TTS applies actor-profile processing per [TTS Request Processing](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/draft-mcguinness-oauth-actor-profile.html#transaction-token-output-rules): it preserves `sub` and `sub_profile` from the `subject_token`, sets `req_wl` to the authenticated Booking Tool, and creates a new outermost `act` object for the Booking Tool while nesting the `subject_token`'s existing `act` claim beneath it.  In this WIMSE-based deployment, the deployment profile defines the `cnf.jkt` binding, proven by the WPT, and the TTS binds the issued token to the Booking Tool's presenter key (`ToolJKT`), which the WIT's `cnf.jwk` carries:

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

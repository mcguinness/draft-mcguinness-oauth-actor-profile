<!-- regenerate: off (set to off if you edit this file) -->

# OAuth Actor Profile for Delegation

This is the working area for a family of individual Internet-Drafts that make OAuth delegation chains visible, evidenced, and bounded.  The target deployments are multi-hop service-to-service and AI-agent workflows, where a token may pass through several actors and trust domains between the original subject and the resource server that finally acts on it.

## The Document Family

The family is one base profile plus three optional companion profiles.  Each document answers a distinct question about a delegated token, with a distinct signer as its trust anchor:

| Document | Question it answers | Evidence signer |
|---|---|---|
| Actor Profile (base) | Who is acting, and through whom? | Outer token issuer |
| Actor Receipts | Which issuer added each hop, and when? | Each hop's authorization server |
| Actor Proofs | Did the actor itself participate, and toward what target? | Each hop's actor |
| Authority Bounds | Did authority ever widen across the chain? | Each hop's authorization server, plus re-authorization authorities |

**How they relate.**  The base profile defines the representation layer: a normalized RFC 8693 `act` chain that every companion aligns against, and that stands alone with no dependency on the companions.  The companions add evidence layers on top:

* **Receipts and proofs are parallel per-hop evidence arrays** (`actor_receipts`, `actor_proofs`) carried inside the token, aligned index-for-index with the visible `act` chain, hash-chained internally, byte-preserved, and verifiable by any recipient offline, with no retained-state or retrieval dependency on the issuing servers.  They attest the same hops from opposite sides of the trust relationship: the receipt says "this authorization server validated and added this hop"; the proof says "this actor signed its own participation and the target it authorized, before issuance."  When both are present, sibling references (`proof_jti`, `receipt_jti`) bind the two chains together.
* **Bounds rides inside receipts** as per-receipt claims recording the authority values (`scope`, `aud`, `resource`, `authorization_details`) in effect at each hop, with cross-hop monotonicity rules: authority may narrow but not widen, except at explicit, signed re-authorization events carried in a `bounds_events` array.  When proofs are also present, bounds cross-checks issued authority against actor-consented target bindings.

Each companion is independently adoptable, and the layers are matched to their threat models: receipts mitigate a compromised downstream issuer fabricating prior-hop provenance, proofs mitigate a compromised current issuer fabricating actor participation, and bounds mitigate an intermediate issuer silently expanding authority.  A token may carry any combination; deployments choose the evidence layers their trust requirements and infrastructure support, from receipts-only (no actor keys needed) to the full belt-and-suspenders stack.

## OAuth Actor Profile for Delegation

The base profile.  Normalizes how delegation is represented in tokens across OAuth grant types (Token Exchange, JWT assertion grants, actor tokens, Transaction Tokens): a conforming `act` chain with issuer-namespaced actor identifier pairs, actor classification via `sub_profile`, delegation-authorization processing rules for authorization servers and Transaction Token Services, resource-server validation and error semantics, and introspection behavior.  Published and self-contained; everything else in this repository builds on it.

* [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/#go.draft-mcguinness-oauth-actor-profile.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-actor-profile)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-mcguinness-oauth-actor-profile)
* [Compare Editor's Copy to Individual Draft](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/#go.draft-mcguinness-oauth-actor-profile.diff)

## OAuth Actor Receipts for Delegation Provenance

Companion profile for issuer-signed provenance.  Each authorization server that adds a visible actor hop signs a receipt JWT for that hop; receipts travel with the token in the `actor_receipts` array, linked by previous-receipt hashes and preserved byte-for-byte, so recipients validate every prior hop against the issuer that created it rather than trusting the current issuer's retelling.  Supports progressive deployment through partial coverage, opaque tokens through introspection delivery, and completeness attestation through `actor_receipts_complete`.

* [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/#go.draft-mcguinness-oauth-actor-receipts.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-actor-receipts)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-mcguinness-oauth-actor-receipts)
* [Compare Editor's Copy to Individual Draft](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/#go.draft-mcguinness-oauth-actor-receipts.diff)

## OAuth Actor-Signed Hop Proofs

Companion profile for actor-signed evidence.  The actor added at each hop signs a proof of its own participation and the target binding it authorizes (audience and resources, extensible to scopes), submitted with the token request via the `actor_proof` parameter and carried in the token's `actor_proofs` array.  Because the signature belongs to the actor rather than any issuer, a compromised authorization server cannot fabricate actor participation at proof-covered hops, and recipients verify that evidence directly.  Composes with receipts through sibling references for outer-token instance binding.

* [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/#go.draft-mcguinness-oauth-actor-proofs.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-actor-proofs)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-mcguinness-oauth-actor-proofs)
* [Compare Editor's Copy to Individual Draft](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/#go.draft-mcguinness-oauth-actor-proofs.diff)

## OAuth Actor Chain Authority Bounds

Companion profile for authority evidence.  Extends receipts with per-hop recording of the authority values in effect at each hop and cross-hop monotonicity verification: `scope`, `resource`, and `authorization_details` may narrow but not widen across the chain, audience is recorded with governance opt-in, and legitimate widening is an explicit, signed, anchored re-authorization event rather than a silent state change.  Gives resource servers and auditors offline-verifiable evidence for the question delegation chains most need answered: did anything expand along the way?

* [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/#go.draft-mcguinness-oauth-actor-authority-bounds.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-actor-authority-bounds)

## Companion Sketches

Early-stage architectural sketches for further companions (transparency logging, capability-token alternatives) live in [proposals/](proposals/README.md).

## Contributing

See the
[guidelines for contributions](https://github.com/mcguinness/draft-mcguinness-oauth-actor-profile/blob/main/CONTRIBUTING.md).

The contributing file also has tips on how to make contributions, if you
don't already know how to do that.

## Command Line Usage

Formatted text and HTML versions of the draft can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed.  See
[the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).

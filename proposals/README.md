# Companion-Profile Sketches

This directory holds early-stage architectural sketches for companion profiles
to [draft-mcguinness-oauth-actor-receipts](../draft-mcguinness-oauth-actor-receipts.md).
They are not published Internet-Drafts and not committed deliverables;
they document design exploration and serve as concrete artifacts for
working-group discussion.

## Current sketches

### Actor-signed hop proofs

- **[Actor-Signed Hop Proofs](./draft-mcguinness-oauth-actor-proofs.md)**:
  adds actor-side cryptographic non-repudiation alongside AS-signed receipts.
  At each hop, the actor being added signs a proof attesting their own
  participation and the target binding they authorize.  Where receipts
  mitigate a compromised *downstream* issuer, proofs additionally mitigate
  a compromised *current* outer-token issuer fabricating actor participation.
  Composes with receipts (a token MAY carry both); cross-references receipts
  by `jti`.

### Authority bounds

- **[Actor Chain Authority Bounds](./draft-mcguinness-oauth-actor-authority-bounds.md)**:
  records and verifies how authority changes at each hop, not just which
  actor hop was added.  Defines authority monotonicity (`scope`, `aud`,
  `resource`, `authorization_details` do not expand across hops) with an
  explicit re-authorization carve-out.  Where receipts attest past hop state,
  bounds add offline-verifiable evidence that authority was not widened
  across the visible receipt chain.  Also defines a weaker current-issuer
  attestation for deployments without receipt-carried bounds.  Motivated in
  part by the
  [PIC Model](https://github.com/pic-protocol/pic-spec), which makes the
  same property structural at the model layer.

### Transparency companions

Two architectural alternatives for adding transparency-log-based trust
verification to actor receipts.  Both address the per-issuer-trust scaling
problem: in federated and open-world delegation use cases, recipients
cannot reasonably enumerate every receipt issuer in advance.  Both shift
trust from N issuers to a small set of trusted log operators.

- **[Transparency, JWT-native variant](./draft-mcguinness-oauth-actor-receipts-transparency.md)**:
  defines inclusion proofs as JWS, with witness gossip deployment-specific.
  Self-contained but reinvents what SCITT is already building.

- **[Transparency, SCITT-aligned variant](./draft-mcguinness-oauth-actor-receipts-scitt.md)**:
  wraps each actor receipt JWS as a COSE_Sign1 Signed Statement, submits to
  a [SCITT Transparency Service](https://datatracker.ietf.org/doc/draft-ietf-scitt-architecture/),
  carries the SCITT Receipt alongside the actor receipt in the token.
  Reuses SCITT's architecture, witness model, and ecosystem at the cost of
  a JOSE/COSE format boundary.

The two transparency sketches address the same problem with different
ecosystem trade-offs.  A future companion-profile draft would commit to
one approach, likely after cross-working-group conversation between OAuth
and SCITT.

### Architectural alternative: capabilities

- **[Actor Capabilities](./draft-mcguinness-oauth-actor-capabilities.md)**:
  an architectural alternative to the receipts + proofs + bounds stack
  rather than a companion to it.  Each visible hop carries a single
  signed link from the *delegator at that hop* (the actor whose
  authority is being extended downward), not from the authorization
  server that minted the current token.  Attenuation is structural: a
  child link cannot reference authority not present in its parent.
  Collapses prior-hop provenance, actor non-repudiation, and authority
  monotonicity into one artifact at the cost of moving signing
  responsibility from authorization servers to actors.  A token uses
  either the receipts-style stack or the capabilities sketch, not
  both.  Offered as a sibling to the four companions so working-group
  discussion can examine whether the four-companion architecture is
  structurally required or reflects retrofit cost.

### Composability

The four sketches are independent companion profiles.  A token may carry
any combination: receipts only, proofs only, receipt-attested bounds,
issuer-attested bounds, transparency proofs, or any subset of these.
Recipients select trust modes based on which signals they require.  See the
receipts spec's Extensibility section for the underlying composability
mechanism.

Each sketch addresses a distinct adversary class:

- **Receipts**: compromised *downstream* issuer fabricating prior-hop
  provenance.
- **Proofs**: compromised *current* outer-token issuer fabricating actor
  participation.
- **Bounds**: intermediate issuer silently expanding scope, audience, or
  resources at a hop, when recipients validate receipt-attested bounds from
  trusted issuers.
- **Transparency**: trust scaling beyond per-issuer enumeration through
  append-only logs of attested artifacts.

A deployment that needs the strongest layered evidence uses all four
companions, while still relying on the trust anchors and local policy each
profile requires.  Most deployments will use a subset matched to their
threat model.

The capabilities sketch above is outside this composability framework
by design: it occupies the receipts-plus-proofs-plus-bounds layer with
a single artifact and is mutually exclusive with those three
companions at the token level.  The transparency sketches remain
applicable on top of either architecture; their leaf format is the
only thing that changes.

## Status

These sketches are personal exploration by the receipts spec author.  They
have not been reviewed, adopted, or endorsed by any working group.  Comments
welcome via repository issues or to public@karlmcguinness.com.

## Why sketches and not drafts

Each sketch leaves substantive design questions deliberately open
(architectural format choice, log governance, witness protocols, dependency
on SCITT WG progress) that should be resolved through working-group
discussion before drafting in earnest.  Publishing as sketches rather than
-00 drafts keeps the architectural conversation in the foreground without
prematurely committing to a specific format.

If a sketch graduates to a draft, it would be renamed and moved to the
repository root alongside the existing drafts.

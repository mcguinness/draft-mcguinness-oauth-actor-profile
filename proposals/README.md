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

### Composability

The three sketches are independent companion profiles.  A token may carry
any combination: receipts only, proofs only, transparency proofs only, or
all three.  Recipients select trust modes based on which signals they
require.  See the receipts spec's Extensibility section for the underlying
composability mechanism.

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

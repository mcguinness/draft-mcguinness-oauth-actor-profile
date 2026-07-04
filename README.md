<!-- regenerate: off (set to off if you edit this file) -->

# OAuth Actor Profile for Delegation

This is the working area for individual Internet-Drafts defining the OAuth Actor Profile for Delegation and its companion evidence profiles.

## OAuth Actor Profile for Delegation

The base profile: normalized `act` delegation chains across OAuth grant types, actor classification, and delegation-authorization processing.

* [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/#go.draft-mcguinness-oauth-actor-profile.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-actor-profile)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-mcguinness-oauth-actor-profile)
* [Compare Editor's Copy to Individual Draft](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/#go.draft-mcguinness-oauth-actor-profile.diff)

## OAuth Actor Receipts for Delegation Provenance

Companion profile: authorization-server-signed per-hop receipts carried in the token, hash-chained and recipient-verifiable offline.

* [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/#go.draft-mcguinness-oauth-actor-receipts.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-actor-receipts)

## OAuth Actor-Signed Hop Proofs

Companion profile: actor-signed per-hop participation proofs with an explicit target binding, carried in the token and validated by recipients.

* [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-oauth-actor-profile/#go.draft-mcguinness-oauth-actor-proofs.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-oauth-actor-proofs)

## OAuth Actor Chain Authority Bounds

Companion profile: per-hop authority recording with cross-hop monotonicity, so governed authority does not expand across a delegation chain except at explicit, signed re-authorization events.

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

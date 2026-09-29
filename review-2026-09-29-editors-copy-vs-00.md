# Editor's copy vs published -00: pre-publication review (2026-09-29)

Compared: draft-mcguinness-oauth-actor-profile-00, -receipts-00, -proofs-00 (datatracker) against main at c8f00c8; authority-bounds reviewed as a first submission.

Revised the same day after a second review. Its corrections are folded in below, and the items it withdrew are listed at the end.

## Summary

-01 improves on the published -00 in all three drafts. The consolidation preserves the main processing model; the intentional behavior changes and remaining issues are identified below. Resolve both "Blockers" (mechanical) and "Resolve before posting" (processing contradictions) before posting. Both matter more than the registry and editorial cleanup.

**Deadline.** The last submission time before IETF 127 (San Francisco) is 2026-11-02 23:59 UTC (datatracker submit page). Profile-00 expires 2026-11-01, so aim for November 1.

## How this was checked

- **Normative cuts:** every removed BCP 14 sentence (263 in the profile, 98 in receipts, 80 in proofs) was classified as moved elsewhere, cited to an RFC, intentionally dropped, softened, or lost. None was simply lost. The second review did not reproduce this audit.
- **Prose cuts:** 111 removed paragraphs with no close match were checked the same way.
- **Cold reads:** one working-group-style reviewer per draft, including bounds.
- **Direct checks:** the build, idnits, the datatracker, and the live IANA registries. Line numbers are confirmed against main unless marked "reviewer".

## What got better

- **Profile** (28.6k → 22.7k words, 464 → 361 keyword sentences):
  - Each rule now has one home, and claim roles and errors are tables.
  - Conflicting duplicates are resolved (the `actor_token` chain merge, inherited extension members, `req_wl` reconciliation).
  - `client_id` is REQUIRED, which {{RFC9068, Section 2.2}} does require.
  - Replay protection is now tied to a sender constraint on the grant itself.
  - The {{RFC8693}} and {{RFC9449}} citations are corrected.
  - The Introduction's claim about what ID-JAG leaves open matches Section 9.7 of ID-JAG -04.
- **Receipts:**
  - The new Receipt Instance Binding section replaces a 250-word bullet with four ordered cases.
  - There is now a testable floor for receipt `exp`.
  - Requirements no other party can check are lowercase, while the rejection rules stay MUSTs.
- **Proofs:** subject continuity now matches receipts, the RFC 6920 reference was dropped cleanly, and the IANA section is unchanged from -00.

## Blockers

1. **Profile output rules.** `draft-mcguinness-oauth-actor-profile.md:920` begins "The issued token MUST satisfy [JWT Access Token Structure]".
   - The Transaction Token output rules (L1041) and the ID token path (L771) reuse that whole section, so other output types inherit the entire access-token format, not just `client_id`. The appendix Transaction Tokens don't carry `client_id`.
   - This was introduced in -01.
   - Fix: prefix the sentence with "When the output is a JWT access token,". Then check each cross-reference so the shared actor-processing rules still apply while each output keeps its own format.
2. **Bounds references.** `ACTOR-RECEIPTS` and `ACTOR-PROOFS` point at the GitHub editor's copies; switch them to `I-D.mcguinness-oauth-actor-receipts` and `-proofs`.
3. **Bounds Section 10.2 (L360 to L362).**
   - The first bullet is an unconditional MUST NOT exceed the effective upper bound. The third bullet allows issuing without `authority_bounds_enforced` for the expanded dimension.
   - Dropping `authority_bounds_enforced` doesn't resolve this: consumer step 6 still rejects any value above the recorded bound.
   - Pick one outcome consistent with step 6: either fail the request, or drop the bounds evidence. Because bounds ride inside receipts, dropping the evidence may mean dropping the receipt chain.
4. **Hardcoded `date:` lines** (2026-04-30, 07-04, 07-03) make idnits report the date as in the past. Delete them or set them on the day you submit.

## Resolve before posting: processing contradictions

1. **Presenter modes (profile).**
   - L700: when a JWT assertion grant is the subject token, the text imports all of Authorization Grant Processing, including the step 6 check that the DPoP proof matches the assertion's `cnf.jkt`. During rebind (L434 to L439) a new presenter proves possession of its own key, so the two can require proofs of two different keys in one request.
   - Resolve this as an explicit security decision, not a blanket exemption. Rebind already keeps any proof the credential's own profile requires, and the replay rule (L516) makes a grant single-use whenever its grant-level binding is not enforced. Decide when grant-binding proof is still required during rebind, reject proof combinations the profile doesn't support, and keep mandatory single use whenever the grant-level binding is not enforced.
   - L898: `may_act` without an `actor_token` makes the authenticated client the new outermost actor, but doesn't say which of the two presenter modes applies. State it.
2. **Scope ceiling from an assertion grant (profile L704).**
   - The JWT access token path (L718) and the refresh path (L795) cap the issued scope; this path doesn't. L1037 (TTS) refers to an "effective scope ceiling" that this input path never establishes.
   - Define that ceiling, and what happens when the grant carries no `scope`. A subset test against an optional claim is not enough.
   - -00 had only a mis-cited "MUST apply scope reduction", so this is an old gap the cleanup exposed.
3. **Evidence lifetime on reissuance (receipts and proofs).**
   - -00 already let reissuance change the outer `exp` (receipts-00 L401). But -00 also required receipt issuers to set `exp` to "cover the expected maximum token lifetime of any token that will carry or inherit this receipt" (receipts-00 L306).
   - -01 turned that issuer-side rule into lowercase "needs to cover" (receipts L288) and made consumer expiry strict (L435). A reissued token can now outlive its receipts, and proofs has the same gap ("as long as their `exp` permits", proofs L746).
   - Require reissuers carrying receipts or proofs to stay within the evidence's remaining validity, or follow a defined drop or fail path. Keep the strict consumer check.
   - The refresh "unless…" clauses need the same repair in both companions: receipts L391 (which also conflicts with L369, and was already in -00) and proofs L409 (reviewer).
4. **Bounds RAR comparison.** The refinement rules omit `privileges`, which {{RFC9396, Section 2.2}} defines as a common field. Specify its comparison and what a missing member means.

## Other changes introduced in -01

- **Document History:**
  - The "Restored the Introduction's defining sentence…" entries (receipts L1182, proofs L1097) and "restore the design center" (profile L2102) describe edits within this cycle, not changes from -00. Rewrite the history against published -00.
  - Missing: Transaction Tokens now omit `act` when there is no delegation; introspection completeness went from SHOULD to MUST; expired older receipts are now strictly invalid.
  - Worth one line: the changes that break a -00 implementation (`client_id` required, the replay cache, `act` omitted in non-delegated Transaction Tokens, `may_act.iss` taking precedence).
- **Keyword policy.** "No other party can check it" is not a general reason to drop a BCP 14 keyword. Some remaining SHOULDs are observable and correctly keep it: L1396 (emit both identifiers) shows in the token, and L1386 (begin validating) shows as rejections.

## Cuts to reconsider

1. **Proofs: the separate-artifact rationale.**
   - -00 explained why receipts and proofs are two JWTs rather than one JWS with two signatures (JWS JSON Serialization). That explanation has no home now, and a JWS-savvy reviewer will ask.
   - Restore the reasons: different signers, adoption prerequisites, and threat models. Leave out -00's unsupported claim that JWS JSON Serialization is poorly supported in libraries.
2. **Profile: {{RFC7800}}** is no longer cited, even though `cnf` is used throughout. Cite it where `cnf` first appears (L201 or L268).
3. **Optional:**
   - Receipts L481 and proofs L512 narrowed "authorization decisions" to "authorization that requires subject continuity". That was intentional, but the new qualifier is never defined.
   - Profile L1263 (metadata) lists one token-type URN for both client assertions and workload credentials. Token Exchange Processing (L648) already explains this, so a cross-reference from the metadata entry is enough.
   - Receipts L290 lost why the receipt `exp` ceiling is coordinated: the originating issuer can't enumerate downstream issuers.

The other removals flagged along the way are still present: the ban on sending actor-specific error details outside the trust domain (L1142), the rule that an actor identical to the subject means no delegation (L205), and both empty-scope outcomes (L941). The rule that the response must report the final scope is covered by {{RFC8693, Section 2.2.1}}.

## Should-fix and clarifications

- **Error codes (all four drafts).**
  - Distinguish three cases. Invalid or policy-unacceptable Token Exchange inputs use `invalid_request`, which {{RFC8693, Section 2.2.2}} requires ("MUST be the invalid_request error code"). Invalid JWT bearer grants use `invalid_grant` per {{RFC7523, Section 3.1}}. Extension-specific denials use `actor_unauthorized`.
  - RFC 8693's "Other error codes may also be used, as appropriate" does not override its explicit `invalid_request` requirement.
  - Today the tables use `invalid_grant` on Token Exchange paths, and -01 says errors "follow {{RFC8693, Section 2.2}}".
- **`aud` in receipts and proofs.**
  - It is NOT RECOMMENDED and exempted from {{RFC8725, Section 3.9}}, but {{RFC7519, Section 4.1.3}} still rejects a JWT whose `aud` doesn't name the recipient.
  - Prefer prohibiting `aud` in these JWTs, or applying audience validation when it is present. Ignoring a present `aud` conflicts with RFC 7519.
- **Profile references.**
  - RFC 6749, OpenID.Core and ID-JAG are informative even though the text relies on them with MUSTs; move them to normative.
  - The ID-JAG reference is pinned to -03; -04 is current.
  - `draft-mora-oauth-entity-profiles-01` is a normative dependency that expires 2026-10-17; ask the author for a refresh.
- **Profile L1659:** use the {{RFC6749, Section 11.4.1}} names for where the error is used ("token error response", "resource access error response").
- **Receipts introspection filtering (L404, L516).**
  - Filtering only uncovered inner actors can keep the full array and its alignment. Filtering a covered actor requires omitting the evidence array.
  - State that interaction; keep both rules.
- **Proofs `target.resource`:** whether a resource must match exactly or by prefix is never defined (reviewer).
- **Bounds (first submission):**
  - Define the `receipt[i]` shorthand locally, pointing at receipts' ordering (index 0 is the newest hop).
  - Add a related-work note for draft-niyikiza (roadmap item 7).
  - Step 7 rejects any unrecognized dimension. Decide deliberately: fail closed, like `crit`, or stay compatible with dimensions added later.

## IANA change controllers (consistency, not a blocker)

The second review first argued for choosing per registry, based on IANA's exception for registries whose creating RFC requires the IESG. After seeing the registry evidence below, it agreed with "IETF" throughout, kept as consistency cleanup. The live registries show IANA doesn't treat these OAuth templates as that exception. In the three registries whose templates say "IESG", recent RFCs registered "IETF" and IANA accepted it:

| Registry (template says IESG) | Recent RFCs registering "IETF" |
|---|---|
| JWT Claims ({{RFC7519}}) | Every RFC since 9068: 9068, 9200, 9246, 9396, 9449, 9493, 9701, 9711, 9901. IESG entries stop at RFC 9027. |
| AS Metadata ({{RFC8414}}) | 9101, 9207, 9396, 9449, 9701, 9728. RFC 9126 is the last to use IESG. |
| Introspection Response ({{RFC7662}}) | 9200, 9396, 9470 |

IANA's examples for the exception are port numbers and XML namespaces. The recommendation is still "IETF" throughout: the profile is already right, and the companions' 36 "IESG" entries should change. Seven of those entries sit in registries whose own templates say "IETF" (six Protected Resource Metadata entries and one OAuth parameter entry), so they change under either approach.

## Publish order

Profile-01 first, then receipts-01 and proofs-01, then bounds-00 once its references are switched. The companions cite the profile only as a whole document, so renumbered sections can't break them. The profile names no companion draft.

## Withdrawn after the second review

- **Bounds author email:** already present (bounds L31).
- **Proofs introspection MAY vs MUST:** not a contradiction; the MUST narrows the general MAY for arrays known to be partial.
- **Profile client guidance ("as requiring" vs "indication"):** not a contradiction. The only difference is that the conformance bullet at L1503 omits the introspection alternative, which is optional to fix.
- **Bounds array order undefined:** overstated; bounds takes its terms from receipts, which defines the order.
- **Introspection filtering and reject-token vs reject-chain as contradictions:** both reframed above as clarifications or decisions.
- **Document History overstating the keyword removals:** withdrawn; several of the listed SHOULDs are observable.
- **Receipt lifetime on reissuance as introduced in -01:** reclassified as a pre-existing gap that -01 widened.
- **Receipts vs proofs disagreeing on malformed completeness:** they agree. Consumer step 4 in both (receipts L423, proofs L449) rejects the token when completeness is asserted and the count differs. Changing that in both would be a separate design decision.
- **Colliding anchors from repeated "Overview" and "Processing" headings:** the build generates unique anchors (`overview`, `overview-1`, `overview-2`, `processing`, `processing-1`). Explicit anchors would be optional maintenance.
- **Exempting the grant's `cnf.jkt` match during rebind:** replaced by an explicit decision, because a blanket exemption could drop enforcement of the grant's sender constraint.
- **Four items blocking posting:** the processing contradictions are also pre-posting work.

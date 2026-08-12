---
status: descriptive
verified: 8eced56
---

# Contract fixture provenance

Every file under `wire/` and `rest/` is copied **verbatim** from the iOS twin at
`apps/ios/Tests/CrossyProtocolTests/Fixtures/{wire,rest}/`, so all three twins
(TypeScript `packages/protocol`, Swift `CrossyProtocol`, Kotlin `:protocol`) pin against
the same normative bytes (the D04 hand-kept-twin pattern). Wire fixtures are the PROTOCOL.md §2-6 examples with
placeholders made concrete as `packages/protocol/src/codec.test.ts` makes them, except
the D32 vote events (checkVoteOpened / checkVoteCast / checkVoteClosed), not yet
fixtured on either twin (issue #345); REST fixtures follow §12's field lists.

Do not hand-edit these here. A change belongs on the iOS/TS side (reviewed against
PROTOCOL.md and the vectors) and is then re-copied across.

## Exceptions

`rest/me-response.json` landed here fresh in #228 and has no iOS twin yet, so the
verbatim rule does not cover it. It still follows §12's field list. The fix belongs on
the iOS side: add the matching fixture to `CrossyProtocolTests/Fixtures/rest/`, then
this file re-joins the copied-verbatim set. Until then, edit it against PROTOCOL.md
§12 directly.

The divergence also runs the other way: iOS carries `rest/share-link-response.json`
with no Kotlin twin (issue #345).

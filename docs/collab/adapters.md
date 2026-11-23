# Adapter plan: the PDF branch against the RFC-0020 contracts

`#6622` is closed (ADR-0021 §3): RFC-0020 puts live collaboration in shared libraries and
leaves each app a model adapter of its own PR. The branch `feature/realtime-collaboration`
stays in `copoka/pdfcraft` as the source of that adapter, not as an open PR. This page is the
inventory the port starts from — what each module becomes, what the adapter must not lose, the
state of the contracts, and the first increment. Measured at `d375f0595`.

## What the branch holds

| module | lines | what it is under RFC-0020 |
|---|---|---|
| `crates/collab-proto` | 1 215 | Envelope, ids, versions, limits and the PDF op vocabulary (`annotation.*`, `field.set_value`, `page.*`, `outline.*`, `dest.*`, `page.label.*`, `page.link.*`). The envelope goes to the session contract; the PDF vocabulary stays app-side as the adapter's change kinds. |
| `crates/collab-pdf` | 7 085 | The server half of the model: an op becomes engine `Edit`s through `Session::apply`; copy tokens for undo, the PDF epoch, page bytes capped at 160 KiB. Becomes the adapter's `apply`/`diff`. |
| `crates/collab-annot` | 2 619 | Typed wire form of the annotation shapes the room shares; wire names are asserted stable. Stays app-side; its whole-string text fields are the `Texts` gap. |
| `crates/collab-outline`, `crates/collab-link` | 1 113 | Typed wire forms of outline items, destinations, page labels and links. Stay app-side. |
| `crates/collab-session` | 1 314 | Protocol driver plus a native and a browser transport. The driver goes to the contract client; the transports stay behind the session seam. |
| `crates/collab-hub` | 662 | Transport seam between a socket and the room. App-side glue; the contract does not need it. |
| `crates/collab-client` | 742 | Optimistic queue, version/epoch tracking, resync, per-user undo. Replaced by the contract's client state machine. |
| `crates/collab-client-pdf` | 5 111 | Every replica turns `Applied.op` into the same engine edit. Becomes the adapter's client projection. |
| `crates/collab-ws` | 4 917 | WebSocket, JWT, revoke; 3 044 of it is the end-to-end suite. The transport stays a backend; the suite becomes conformance cases. |
| `crates/collab-server` | 1 977 | One writer, one monotonic version, dedupe window, journal, lease, resync tail, role checks. Replaced by the session contract. |
| `apps/pdfcraft-collab-server` | 5 623 | The daemon: control plane, per-room store (`CURRENT`, `journal.jsonl`, `CABINET`), cabinet commit and publication queue. App-specific backend; stays. |
| `crates/ui-egui/src/collab.rs` | 2 765 | The Collaboration window: participants, presence, role label, commit. Presence moves to the shared control; the rest is app policy. |
| engine and canvas hooks | ≈1 449 | `apply_remote`, edit interception, selection/epoch in canvas. App-side, stays. |

`wc -l` includes inline tests and, for `collab-ws` and the daemon, the test files. The
collab-attributable tree is ≈36 600 lines; only the generic ordering/journal/presence/undo
half moves, the PDF mapping and the cabinet backend stay.

## The six surfaces it must meet

RFC-0020's invite and chat surfaces are not in this plan: the cabinet mints invites and owns
chat, and the editor stays a bare client (RFC-0020 surfaces 2–3). The six below are the ones
the PDF adapter itself must meet.

**1. Session.** The contract owes ordered submission, one relay per document, join reconciled
with the persistent counter, relay-issued replica identities and opaque frames (RFC-0020
constraints 1–5, landing order 2). The branch has the one-relay rule (`DocKey` is the cabinet
`fileId`, never a path), a replayable journal with the version in `CURRENT`, dedupe by `opId`,
a 120 s writing lease and `resync` from `last_version` (`collab-server/src/lib.rs:603-614`).
It does **not** have opaque scope locks: structural ops are serialized by the lease plus a
`base_version == head` check (`:421-443`). And its frames are typed, with the server reading
the payload — the opposite of constraint 4. Both are contract questions, not local fixes.

**2. Presence.** Ephemeral latest-wins per peer and kind, without edit permission, never in the
document (RFC-0020 surface 1). The branch has presence that is never journaled
(`collab-server/src/lib.rs:619-620`), with a client-side 30 s TTL
(`ui-egui/src/collab.rs:30`), but its own selection model: a geometric text range, per-line
quads, a caret and a field caret (`collab-proto/src/lib.rs:660-780`). The badge control merged
in `#160` (`craft_ui::presence::{Peer, PresenceBar}`) is usable today and takes a display name
plus a free-form `at` string — enough for "page 7". What has no shared model is the selection
itself: the only node-id selection is in `#128`'s `PresenceMsg`, which is not the contract. A
PDF text selection is geometry, so this is a contract gap (below), not something to paper over
with a second presence wire.

**3. Texts.** `Texts` is the boundary; a mature CRDT sits behind it, and `craft-crdt` (`#345`)
is conditional on the differential oracle (RFC-0020 surface 4 and landing order 3). PdfCraft
has no page-text path in a session on purpose: page content edits are not in the op table and
`session.apply` callers bypass collaboration (STATUS §7). What *is* text and does merge today
is the comment body (`annotation.set_contents`) and a form-field value (`field.set_value`) —
both whole strings, with the server's order deciding a concurrent write. Those two properties
are the whole `Texts` surface here; so there is no `stale` text path to delete here.

**4. The PDF model.** The adapter maps the app's document into the contract's object model and
back. The branch already has this mapping in both directions (`collab-pdf`,
`collab-client-pdf`), applied to the real object graph through the engine, so unknown objects
survive. Two limits must survive the port: a structural op (insert, duplicate, replace, split,
combine) carries a whole page as bytes and bumps the epoch, so it is not a generic property
change and stays ordered and exclusive; and a page larger than `MAX_RESTORE_PAGE_BYTES`
(160 KiB) has no channel (RFC §16 blob gap) and must be refused, never truncated. Any edit the
projection cannot represent has to be refused at command entry, not diffed away.

**5. Commit and the cabinet.** The cabinet contract is ours (`INTEGRATION.md`): `commit` →
`committed {version, etag}`, a stale etag is `409`, the callback carries
`{fileId, bytes, sha256, version, expectedVersion}`, the idempotency key is
`fileId:version:sha256`, and the queue is rebuilt from `CURRENT` vs `CABINET` after a restart
(`apps/pdfcraft-collab-server/src/rooms.rs:590-618`). It is implemented and tested (four
integration files, 1 917 lines). Nothing in RFC-0020 changes it. The only shared piece is the
save answer — `craft_ui::cloud::Event::Saved` and `Fault::Conflict` at `#160` — which matters
only if the cabinet is later reached through `craft-cloud`; today the daemon's client is plain
`http://` with no redirects, chunking or TLS.

**6. Roles.** Server-enforced read-only, not only a UI gate. The branch takes the role from the
verified JWT, pins one role per person per room (`collab-server/src/lib.rs:365-397`), refuses a
viewer at command entry and in `submit`, and has four roles
(`Owner | Editor | Commenter | Viewer`, `collab-proto/src/lib.rs:226-248`). The contract's
`Access` is `Edit | View`. `Commenter` has no contract equivalent; the adapter should enforce
it in the schema (annotation-specific kinds only), not ask for a wire change.

## State of the contracts (2026-11-23)

- `craft-libs#160` (cloud contracts) **merged** 2026-11-17T18:36Z, merge commit `fd7232d7`
  (head `aa4647a7`, +9 166/31). `craft_cloud::Entry` carries `etag` and `writable`;
  `craft_ui::cloud::Event::Saved` answers a write with its entry or the `Fault` that refused
  it, and `Fault::Conflict` is the stale etag.
- The **live-session contracts PR** of RFC-0020 (session, presence, invite and conversation
  traits beside `craft-cloud`, with an in-process mock, degrade-to-absent tests and the text
  oracle) has **no open PR**. `#128` is the shared core in the same lineage, not that PR: head
  `9a6148e0`, open since 2026-10-14 (+4 440/27), with `DocSchema`/`HubAdapter`, a pure
  `server::Server` (accept with seq / refuse with fixes), `client::Client` and per-user undo,
  `presence::PresenceMsg`, RGA text in `value`, and no invite flows, no TTL scope locks and no
  `crdt.*` passthrough. Useful as the shape the contract will take; not a dependency yet.
- `craft-libs#345` (`craft-crdt`) open, head `cef9e963`. Under RFC-0020 it is an engine only if
  it passes the oracle; the `Texts` API is the dependency, not this crate.
- `craft-libs#716` (session messages in `craft-web-host`) open, head `7666e2b1`. The branch's
  web build already pins that head (`apps/pdfcraft-web/Cargo.toml:32`) while the rest of the
  workspace pins `e2dc681a3`; that is an unmerged upstream revision already in the tree. Do
  not add another, and move it to the merge commit when `#716` lands.
- `craft-libs#2108` (host statuses for the title bar) open, head `2882d72`.
- The workspace pin `e2dc681a3` is an ancestor of `main` and 2 285 commits behind it.
  `fd7232d7` is merged, so the adapter can pin it without the ADR-0016 fork exception.

## First increment: the PDF projection (≤1 500 lines)

One branch from `origin/main` `b192e36ff`, app side, no wire, no room, no unmerged pin. It
lands the half of the adapter the contract will call, and which the branch already proves:

| piece | lines |
|---|---|
| `PdfSchema`: kinds, parents and property bounds for page, annotation, field, outline item, dest, page label and link | 300 |
| `PdfView`: projection of a session's document into the merge-class view (page and annotation nodes, field nodes, outline nodes, doc entries) | 450 |
| `PdfChange`: the merge-class mapping both ways (`annotation.*`, `field.set_value`), wire names asserted stable | 450 |
| `apply` glue through `Session::apply`; no envelope, no ids, no ordering | 100 |
| tests: the five ported convergence, presence, undo, page and commit cases, plus a viewer refusal and an unsupported edit refused before the document changes | 200 |

It deliberately names no trait: with the contracts PR unpublished, a local `DocSchema`-shaped
trait would be another protocol copy — what rule 16 forbids. A view/change model with no envelope,
no ids and no ordering is the app's document mapping, which RFC-0020 keeps local. When the
contract lands, increment 2 is `impl DocSchema`/`impl HubAdapter` over this module plus a pin
bump (≤300 lines, no behaviour change).

The tests are ported from the branch's cases rather than invented:
`three_clients_converge_and_a_late_joiner_catches_up`, `two_clients_share_a_typed_highlight`,
`two_clients_undo_a_deleted_comment_and_a_deleted_page`,
`two_clients_insert_and_duplicate_a_page_and_converge`,
`a_commit_over_the_wire_produces_a_readable_pdf` (`crates/collab-ws/tests/e2e.rs`). The
structural-versus-merge case is the branch's own refused reply to a deleted parent
(`two_clients_reply_with_a_status_and_a_deleted_parent_refuses_a_reply`): a merge-class change
addressed to a scope a structural change removed is refused, never resurrected.

## Deferred, each with the condition to start

- **Merge of `origin/main` into the source branch.** Measured 2026-11-23: 823 behind, 174
  ahead, `git merge-tree` reports 17 conflicted files — `Cargo.toml`, `Cargo.lock`,
  `apps/pdfcraft-web/{Cargo.toml,src/main.rs}`, `crates/engine/src/commands.rs` and ten
  `ui-egui` files. Not merged: the branch is a closed PR and a source, and the port starts
  from `main`; reopening the merge only pays off if the branch is reopened as a PR.
- **Structural scope locks.** Start when the contracts PR opens, with the proposal below.
- **`Texts` for comment bodies and field values.** Start when the `Texts` API lands; then the
  whole-string properties become text properties and the last-writer-wins case disappears.
- **Web session messages and the title bar.** `#716` and `#2108`; the branch's web bridge and
  its pin are the reference.
- **Blob channel for pages over 160 KiB.** RFC §16 gap; refused today, and must stay refused.

## Two gaps to raise on the contracts PR (not local fixes)

1. **A geometric selection has no shared model.** `#160`'s badge control takes a free-form
   `at`, and `#128`'s `selection: Vec<NodeId>` fits an object canvas, not a text range over a
   PDF page. Options: an app-projection extension in the presence message, or a bounded
   per-kind extension keyed by the page node. The PDF adapter can live with either; it cannot
   live with a second presence wire.
2. **Scope locks versus the one-writer lease.** RFC-0020 asks for an opaque scope string per
   non-commuting command. The branch's lease is a scope of one (the page tree) with a TTL and
   refresh, and `base_version == head` is the conflict check. Propose to the contract that
   "lease + head on the page tree" is the submitted form of a page-tree lock, with the
   conformance cases the branch already has (a split addressed to a passed epoch is stale and
   retried; two combines roll back a stale one). If the contract wants real per-scope locks,
   the adapter adopts them; the branch must not keep its own kind.

## Measurement

`d375f0595`, 174 commits above the merge base `271e479c5`, 823 behind `origin/main`
`b192e36ff`; line counts are `wc -l` at that head. `#6622` and `#7657` are closed; no open
`copoka` PR remains in this repository.

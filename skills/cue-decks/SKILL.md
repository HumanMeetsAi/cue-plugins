---
name: cue-decks
description: Create, edit, and publish presentations and print-ready documents on cue through its hosted connector. Use when the user asks for a cue deck or document, revisions to an existing cue draft, build status, or publishing and sharing on cue.
version: 1.87.0
---

# Working with cue decks

cue decks are small React/TypeScript projects (flat file set: `index.ts`, `slides.tsx`,
`content.ts`, plus vendored pattern-library files or a `theme.tsx`). You edit the source; cue runs a
validation gate (typecheck, forbidden-API lint, browser-target dry-build) and builds a
sandboxed preview server-side. Submissions spend none of the deck owner's AI budget —
you are the author.

This skill is versioned via the `version` frontmatter field. Use this bundled copy.
Updates to these instructions are delivered through a new plugin package version.
Do not fetch or replace this skill with remotely served behavioral instructions.

## Tooling — pick in this order

1. **cue MCP tools** — use them if available in this session. The whole surface:
   reads `cue_list_decks`, `cue_shelf`, `cue_templates`, `cue_library`, `cue_job`,
   `cue_feedback`, `cue_readers`, `cue_talks`, `cue_proposals`; source `cue_submit`,
   `cue_create_deck`‡, `cue_list_files`‡, `cue_read_files`‡, `cue_grep`‡,
   `cue_write_files`‡, `cue_edit_file`‡, `cue_delete_files`‡, `cue_reseed_draft`‡;
   history `cue_history`, `cue_diff`, `cue_restore`, `cue_set_live`; assets
   `cue_request_upload`‡, `cue_list_assets`, `cue_delete_asset`; account fonts
   `cue_upload_font`, `cue_list_fonts`, `cue_delete_font`; talks `cue_talk_operators`,
   `cue_my_notes`;
   lifecycle `cue_resolve_note`, `cue_fork`, `cue_propose`, `cue_accept_proposal`,
   `cue_decline_proposal`, `cue_publish`, `cue_share`, `cue_unpublish`, `cue_delete_deck`,
   `cue_pin`; Calls `cue_call_create`, `cue_call_entries`, `cue_call_enter`,
   `cue_call_withdraw`, `cue_call_decide`; Programmes `cue_programme_create`,
   `cue_programme_slots`, `cue_programme_bind`, `cue_programme_approve_version`,
   `cue_programme_import` (and `cue_talks` without a deck for a speaker's upcoming slots).
   ‡ remote-connector-only (server-held draft workspace) — see "Two lanes" below;
   everything else is the identical tool on both lanes.

## First session on a new account

If `cue_list_decks` or `cue_shelf` comes back with an `onboarding` field, the account owns
no deck yet — the person has just connected you. Say that cue is connected, then offer the
first deck in the same breath: ask what it is about and who it is for (the one-round intake
below), or, if they would rather see something first, start from a template. The deck
appears on their cue home the moment it exists. Once the account has a deck the field is
gone; never mention onboarding again.

## Two lanes, one contract: local shim vs remote connector

Same account, same decks, same tools where they overlap, same gate. Two ways this skill
can be running — a local shim, with a filesystem, or a remote connector, with a
server-held draft workspace — and each lane has its own copy of it: the full skill at
`<CUE_URL>/cue-skill.md`, and, for remote sessions, the trimmed
`<CUE_URL>/cue-skill-remote.md` (same `version`, local-shim material removed).

- **Remote connector** — this session IS `POST <CUE_URL>/mcp` (claude.ai, ChatGPT — no
  local filesystem). Editing goes through a SERVER-HELD draft workspace, and `cue_submit`
  takes no `dir` (the working copy IS the submission). Auth is OAuth 2.1 via the host
  app's consent screen — no token to mint or paste.

The edit loop — locate, read narrowly, patch in place. Context spent on unchanged source
is context the deck does not get:

1. `cue_list_files` — what exists, plus the `rev` every write needs.
2. `cue_grep {deckId, pattern, files?, maxMatches?, context?, ignoreCase?}` — regex over
   the workspace, answering `file:line: text` lines. This is how you find the slide, the
   content key or the token — not by reading whole files.
3. `cue_read_files {deckId, files, lines?, rev?}` — RAW text blocks, one per file, each
   under a `=== name (bytes, hash)` header. `lines: [start, end]` is a 1-based inclusive
   range: read the 40 lines around the hit, not the 900-line file. `rev`
   (`draft` — the default — `live`, a version id, or a digest; owner-only) reads a past
   point without disturbing the draft.
4. `cue_edit_file {deckId, file, edits: [{old, new}], baseRev}` — exact-string replace,
   each `old` matching EXACTLY once, same `rev` contract as `cue_write_files`. This is
   the default way to change a file.
5. `cue_submit {deckId}`, then `cue_job` — same gate as every other lane.

`cue_write_files` is for NEW files and full rewrites only; `cue_delete_files` removes one.

The tool surface diverges ONLY where the filesystem does (the marked tools above); every
other tool — list, publish, share, feedback, fork, propose, assets — is the identical
tool on both lanes, same arguments, same gate.

STALENESS: the remote draft and a local shim directory are two independent working copies
of the SAME deck. Edit through the shim and submit, then keep going here without
re-seeding, and the next read/write reports `stale: true` — the deck's live source moved
on beneath the draft, and submitting anyway would replace edits you haven't seen.
`cue_reseed_draft` resolves it; it discards this workspace's unsubmitted edits, so read
what you still need first.

## The workflow

1. **Read before editing.** A submission REPLACES the deck's whole source (no merge). On
   a deck you did NOT author in this session, run `cue_diff` first — see *History, diff,
   restore* below. And on ANY existing deck, run `cue_feedback` first: the owner leaves
   change requests for you as notes in the studio (see *Notes* below) — they are part of
   the job even when the user's message doesn't repeat them.
   The draft workspace already holds the deck's source: `cue_list_files`, `cue_grep` to
   locate what you need to change, `cue_read_files` with `lines` to read just that part.
2. **Edit in the workspace** — `cue_edit_file` for a change inside an existing file,
   `cue_write_files` for a new file or a full rewrite. Constraints the gate enforces:
   - Flat filenames only; extensions `.ts .tsx .json .css .md`; `index.ts` required
     (default-exports the deck manifest); ≤ 80 files, ≤ 512 KB each, ≤ 8 MB total.
   - Imports must be browser-buildable, and the allowlist is exact: `react`,
     `motion/react`, `gsap` (plus its plugin subpaths — `gsap/ScrollTrigger`,
     `gsap/Flip`, `gsap/MotionPathPlugin`, `gsap/all`, …), `three` (BARE only — the
     `three/examples/jsm/*` addons are not available), and cue's `@/lib` primitives
     (`@/lib/types`, `@/lib/content`, `@/lib/primitives`, `@/lib/formula`,
     `@/lib/room`). No node builtins, no server-side packages, no other npm package —
     `katex` included: maths goes through `@/lib/formula` (see *Formulas*), never the
     library directly. There is no chart library on the list, by design: a results slide
     draws with the deck's own components (see *Asking the room*).
   - Forbidden: `eval()`, `process.env`, secrets, network calls to the platform origin.
   - Graphics are code, not files: the published artifact's CSP blocks EVERY external
     host, so `<img src="https://…">` renders as a broken image no matter where it is
     hosted. For vector/diagrammatic art — diagrams, icons, charts, logos-as-shapes —
     inline SVG/CSS with theme tokens stays the preferred build: it stays crisp,
     restyles with the theme, and can assemble step-by-step. For PHOTOS/SCREENSHOTS
     and VIDEO, use the asset lane:
     `cue_request_upload {deckId, filename, contentType}` returns a single-use PUT URL
     (10 min TTL) plus a ready-to-run `curl` line; with no shell of your own, pass
     `sourceUrl: "https://…"` instead and the server fetches that public image itself
     (no curl, no user round-trip). If the image is only on the USER's device, give them
     the `uploadUrl` — opened in a browser it is an upload page with a file picker — and
     confirm with `cue_list_assets` rather than handing them the `curl` line.
     The answer is `{assetId, ref, kind, ladder, width, height}` — put `ref` in deck
     source as a RELATIVE path exactly as returned, through the engine's primitives
     (`<Image asset="assets/<id>.jpg" … />`, `<Video asset="assets/<id>.mp4" … />` — sized,
     see below), never absolute or external. A photo drawn with a plain `<img>` loses the
     viewer's tier and cannot be cropped in Studio; the gate names each one as advice.
     UPLOAD BEFORE SUBMIT: the gate rejects source that references an asset the deck's
     store doesn't hold yet. Accepted types: png/jpeg/webp/gif, and video as mp4
     (H.264 video + AAC audio or silent, `moov` atom first — export with
     `-movflags +faststart`) or mov (transcoded to MP4 on upload; a video ref is always
     `.mp4`) — **no SVG** (vector art belongs inline, where it's animatable and
     themeable; an uploaded SVG could also carry script); an SVG the user gives you — a
     file, or a `.svg` from their cue library staged into `source-material/` — is
     inlined into the TSX the same way. Every upload is decoded and re-encoded
     server-side (EXIF/GPS stripped, so uploading identical bytes twice dedupes to the
     same ref), and gets a size LADDER on the media host (images: stage lossless,
     desktop, phone, lite, a tiny placeholder; video: stage ≤1080p, 720p, 480p and a
     poster). Every video, and any file over 8 MB, is processed in the background: the
     answer says `ladder: "pending"` and `cue_list_assets` flips it to `ready` (or
     `failed`). Videos may total 5/15/60 minutes per deck on Free/Pro/Event, and the processed ladder
     counts against the deck budget. Large media is never
     embedded in the artifact or sent to the compiler — it is served from the media
     host. The build maps every `assets/<id>` ref to its ladder in the artifact's
     `media.json` (also inlined into index.html). Render laddered media with the engine's
     primitives — `<Image asset="assets/<id>.jpg" fit="cover" style={{ position: "absolute", inset: 0 }} />`
     and `<Video asset="assets/<id>.mp4" playOnStep={1} muted style={{ width: 960, height: 540 }} />`
     from `@/lib/primitives` (see *Imported and event decks*): they pick the viewer's tier and load
     in slide priority. **Give them a size:** both have NO intrinsic height (the picture inside is
     absolutely positioned), so a bare `<Image asset fit="cover" />` in normal flow is 0 px tall and
     draws nothing — pass `style` with a width and height, or `position: "absolute", inset: 0`
     inside a positioned, sized parent. A plain `<img src="assets/<id>">` works only for a small image that is
     also embedded in the artifact; a video or a large image is media-only and a plain
     tag pointing at it renders nothing. A submit
     whose source names a video still laddering FAILS the gate with "ladder still
     running, retry" — wait for `ready`, then submit again. Optional: a
     `media-slides.json` source file `{ "<slideIndex>": ["<assetId>", …] }` (the
     16-hex id, no extension) tells the player which slide shows which asset so it can
     prefetch ahead of the presenter; ids must be ones your source references and
     indices must exist. Budgets are separate from the 8 MB source budget: Free: 16 MiB/image, 256 MiB/video, 512 MiB/deck, 1 GiB/account;
     Pro: 32 MiB/image, 1 GiB/video, 2 GiB/deck, 10 GiB/account;
     Event: 32 MiB/image, 2 GiB/video, 8 GiB/deck, 40 GiB/account.
     Operator media grants raise these independently of Pro. Large videos use resumable
     uploads in the local shim and browser page; keep the same upload link to resume (the upload grant's `maxBytes` is the real
     per-file cap; `cue_list_assets` reports usage; `cue_delete_asset` removes one — but
     drop its reference from source in the same round of edits, or the next submit fails
     the same dangling-ref check). A small `data:` URI remains fine for a tiny raster and
     still counts against the 8 MB source budget. If the user hands you an image that
     lives at a URL (a hosted photo, a shared-drive link, their website's hero image),
     do NOT paste the URL into source — bring the bytes in:
     `cue_request_upload {deckId, filename, contentType, sourceUrl}` has the server fetch
     it straight into the deck's asset store.
     Then reference the returned `ref`. Users can also import URLs into their
     library from the web UI (/library "Import from URL"), from where an authoring job
     pulls them like any other library file.
   - Fonts are ACCOUNT assets, never deck assets. An account on the event or team plan
     keeps licensed fonts (OTF/TTF/WOFF/WOFF2, 12 MB each) in its font library; a deck
     declares the faces it wants in its theme and every build of that deck under the
     owner's account ships a SUBSET of each (only the glyphs the deck's source uses) as a
     same-origin WOFF2 with `fonts.css`:
     `fonts: [{ family: "GT Haptik", weight: 500, style: "normal", fallback: { family: "Inter", sizeAdjust: 101 } }]`
     plus `fontsAllowLocal: false` (the default — `true` also emits `local()` for each
     declaration's `local: [...]` face names). Write the declaration as a LITERAL in
     `theme.tsx`'s `theme` object (the build reads only that file, as data; a variable or
     spread there fails the gate). A deck ships at most 24 MB of fonts. Use the faces in your tokens as a stack:
     `"--deck-font-display": '"GT Haptik", "GT Haptik Fallback", sans-serif'` — the
     `"<family> Fallback"` face carries the metric overrides. Resolution per face: the
     owner's library (family case-insensitive, same style, weight exact or ±100) → cue's
     licensed library → `local()` (only if allowed) → the declared `fallback`. The gate
     prints one `fonts` line per face (`library` / `cue` / `fallback` / `missing`) and the
     job result carries them as `fonts`; a face that resolves nowhere and declares no
     fallback FAILS the gate, a fallback is a warning. A deck forked to an account without
     the font renders its fallback there. Uploading requires `attest: true` — it records the
     user's statement "I hold a licence permitting use of this font in content served by
     cue on my behalf." — ask the USER first and only pass it on their say-so. Fonts
     whose licence flags (OS/2 fsType) forbid embedding are refused; nobody, the owner
     included, can download a font back.
     `cue_upload_font {attest: true, sourceUrl?}` has the server fetch a public font URL,
     or (no `sourceUrl`) returns a single-use `uploadUrl` to PUT the file to — or to hand
     the USER, who opens it in a browser and picks the file (the attestation is recorded
     when the URL is minted, with the calling agent's IP — not the browser's);
     `cue_list_fonts` shows the library (and `canUpload`), `cue_delete_font {fontId}`
     removes one (built versions keep their subset).
   - Fonts stay with the account; OUTLINES TRAVEL WITH THE DECK. A deck that ships a
     `text-layout.json` (the Keynote import generator writes it: per text layer the box, the
     lines with their boxes and baselines, font, size, tracking, align) gets vector glyph
     outlines of its source-language text at every build — `outlines/<slideId>.svg` (one
     `<symbol id="o-<layerId>">` per layer) and `outlines/index.json` (provenance per object:
     `source: library` when shaped from the real font, `fallback` when only the fallback face
     resolved). Line breaks come from the layout, never from the browser. The set is part of
     the deck: `cue_fork`, duplicate and an accepted `cue_propose` carry it, so an account
     WITHOUT the font still serves the exact text. A build never replaces a real-font outline
     with a fallback one: it keeps it, and marks it `stale` if the text changed (live text
     renders for stale objects until an account holding the font rebuilds). To upgrade
     fallback outlines, have the font's licensee fork, build and propose back. The job's
     `outlines` line reports shipped / library / fallback / stale / generated.
   - A talk's owner can name ROOM OPERATORS (`cue_talk_operators {talkId, add?, remove?}`,
     talkId = the `?room=` id): they open the presenter console, mint presenter room tokens,
     copy/revoke screen links and drive captions for that talk — never edit, publish, delete
     or re-share the deck.
   - Keep user-visible copy in `content.ts` as keyed LocalizedStrings referenced via
     `useContent()` — literal prose in slide files draws gate warnings.
   - JSX attribute strings are not JavaScript strings: `suffix="\u00d7"` renders the six
     characters literally. Write the real character (`suffix="×"`) or an expression
     (`suffix={"\u00d7"}`); the same goes for `\n` and every other escape.
3. **Submit with a message.** Pass a one-line `message` (CLI `--message`) saying what
   changed — it becomes a turn in the deck's session transcript the owner sees in the studio.
4. **Watch the job.** `cue_job {jobId, wait: true}`. On success the result has a `previewUrl`
   (sandboxed draft preview). On failure read `result.error` — it contains the full
   gate report (which file, which check); fix and resubmit. A failed gate leaves the
   deck unchanged.
5. **Publish only when asked.** `cue_publish {deckId}` makes an immutable shared version
   and returns `viewUrl` (the self-paced share page — give this to the user) and
   `liveUrl` (a presenter-led live session of the same version; append `?nav=0` to
   hide the deck's own navigation arrows for audience links). If the result carries
   `previewNotice` with `previewUrl: null`, the publish SUCCEEDED and only the preview link was
   not ready — do not publish again; `viewUrl`/`liveUrl` are good. Submitting alone only
   updates the draft preview. The deck's own navigation is a position bar and a counter:
   click or drag the bar, or press `g` (or a digit) and type the number the counter shows;
   `Home` / `End` go to the first and last slide. On a phone the share page steps on a tap at
   the deck's edges and leaves taps on links, buttons, form controls and media to the slide:
   make anything tappable a real `<a>` or `<button>`, or mark it `data-cue-interactive`.
   The share page, the studio Stage and the review page also list every slide (`o`, or the
   grid button) and keep the position in the address bar: `?slide=N` opens slide N (1-based,
   counting hidden slides), on `/embed/<id>?slide=N` too. A link from the share sheet carries
   no `?slide`, so it opens at the start.

   **What the free lane allows, so you can explain a 403.** Authoring is free on cue —
   writing decks with the user's own agent costs cue nothing — and the limits sit on
   DELIVERY. An account with no paid plan holds **five published decks live at once**
   (drafts are unlimited; `cue_unpublish` or deleting a deck frees a slot), **one share link
   per deck** (the deck's own `?k=` link — a second, named link is Pro), **ten named invitees
   per gated deck**, and **aggregate readership only** (`cue_readers` returns opens, unique
   viewers and languages with `aggregateOnly: true`, not who read what — see the
   `cue_readers` entry below before reporting it). Per-recipient tracked links, print-grade
   documents, deck translation and live captions/translation in the room are Pro. Each
   refusal is a 403 whose sentence names the number and what frees it — relay it as-is and
   offer the fix it names (e.g. unpublish an older deck) rather than retrying. Over MCP the
   tool result is that sentence with the status appended; over REST the body is
   `{ error, code: "needs_pro", capability }`, so an HTTP client can branch on `code` instead
   of pattern-matching the prose.

## History, diff, restore

A deck keeps an append-only **timeline**: every publish (a milestone readers actually saw)
and every accepted draft revision (a submit, a studio quick-edit, a co-editor's edit, an
accepted proposal, a restore), each with who made it, the prompt that produced it, and the
source digest it landed on. Going back is an *event* on that timeline, never an erasure.

**Look before you overwrite.** A submit replaces the deck's whole source. If you did not
author the current draft in this session — the user has been editing in the studio, a
co-editor pushed, a previous session left work unpublished — read the diff first.
`cue_diff {deckId}` defaults to live → draft (what is unpublished); `from`/`to` take any
pair of `draft` | `live` | `<versionId>` | `<digest>`, and `cue_history {deckId}` is who
changed what, with the prompt behind it.

`cue_diff` returns a summary plus `text`: the CONTENT cells that moved (per key *and*
language — a change in one language and not another is visible, not hidden), then the
per-file patches. A side marked `(source default)` is not an override at all — it is the
string `content.ts` ships, shown so an edit over untouched copy reads as the replacement it
is rather than as text appearing from nowhere. `cue_history` returns the log plus a one-line-per-entry rendering; each
entry's `digest` is what you pass back to `cue_diff` or `cue_restore` (and to a rev-aware read), and
each published entry's `id` is what `cue_set_live` takes. Unpublished work belongs to the
user: surface it ("your draft has unpublished changes to the intro — keep them?") rather
than quietly submitting over it.

**Read a past point.** `rev` is `draft` (default), `live`, a version id, or a digest —
for "what did the version we sent the board actually say?". It is a read; nothing moves
until you submit.
`cue_read_files {deckId, files, rev}` (owner-only) reads that point straight out of
history without disturbing the draft workspace.

**Restore** puts the DRAFT back: `cue_restore {deckId, discard: true}` drops unpublished
changes (draft ← live), `{digest}` or `{versionId}` puts it back to that point, and
`files: ["slides.tsx"]` narrows the move to those files (with
`restoreDeletedContentKeys: true` to bring their copy back).

It goes through the same validation gate a submit does, so it answers with a `jobId` —
poll it with `job` — and the promote appends a NEW history entry (`via: "restore"`). It
does not change what readers see. If the draft already matches, you get `alreadyClean` and
no job.

**Selective revert.** `cue_restore {files: ["slides.tsx"]}` (CLI `--file`) combines with any
one of the three selectors and moves ONLY those files back, keeping every other draft change
— the undo for "I deleted a slide AND rewrote the intro, and only want the slide back". A
file the target does not have is deleted from the draft. Partial reverts record no
`restored_from` digest, because the draft is then a mix of two revisions rather than a copy
of one. In the studio's Compare view the same thing is a button per changed file (Source
lens); single copy cells revert there too, through the overrides lane rather than the gate.

**Restoring a deleted slide's copy.** Add `restoreDeletedContentKeys: true` (CLI
`--restore-copy`) alongside `files` and the keys that point's `content.ts` has and the draft
no longer does are spliced back in too — otherwise a slide restored through `slides.tsx`
renders `⟦points.title⟧` where its copy used to be. Only the missing keys are added; every
other line of your `content.ts` is left exactly as you wrote it, and the answer lists what
came back in `contentKeysRestored`.

**Set live** is the reader-facing one: `cue_set_live {deckId, versionId}` re-points the
published artifact at an earlier version (and, by default, restores the draft to match, so
the deck does not immediately read as "published · edits"). It changes what everyone with
the link sees the moment it lands — do it when the user asks for it, not on your own
judgment.

**Version numbers (D361).** Every ORIGINAL publish gets an ordinal — v1, v2, … — shown on
`cue_history` entries (`ordinal`) and used by people when they say "the one I sent on
Tuesday". A set-live adds a history entry that reads "same as vN" (it carries `sameAs`, the
original's id, and never a new number). An entry's `createdAt` is the commit time. Say "v3",
not a version id, to the user.

**Viewing an older version by URL.** `/view/<deckId>?v=<number>` serves that published
version to the owner, a co-editor (`edit` grant), and anyone whose grant carries `history`
(the grant's meaning: "may browse this deck's published versions" — the list and `?v=`,
NOT revisions, diffs or restore). Everybody else simply gets what they would get without
`?v=`; there is never a "no such version" answer that leaks existence. The viewer shows a
Versions pill and drawer to those who may browse, and exports are hidden while browsing an
older version. Presenter surfaces (`/live`, room tokens) ignore `v`. The same list is
`GET /api/decks/<deckId>/versions` (session-based, same policy, `404` for anyone else):
`{ current, versions: [{ ordinal, id, createdAt, sameAs?, current }] }`, newest first.

## Access tiers: who can view a published deck

Every deck carries an access tier on its record, enforced on EVERY viewing surface (the
/view page, the raw artifact fetch, PDF, PPTX, thumbnail, and live-room viewers), not just
the share page. **New decks are born `gated`** — opening a deck up to anyone is an explicit
choice. `cue_publish` (and the `PATCH /api/decks/:id` lane) can set the tier via `access`
and `invitees`:

- **`gated`** (the default for a new deck): only the owner and explicitly named `invitees`
  can view. An empty/omitted list is owner-only; it never implies all signed-in accounts.
- **`public`**: anyone with the link, zero friction — pass `access: "public"` explicitly
  to open the deck up.
- **`link`**: an unguessable, rotatable share token presented as `?k=<token>` in the URL,
  no account needed. Only settable when cue enables secret links for the deck's
  owner; a `link` deck's `viewUrl`/`shareUrl` in the publish response
  carry the `?k=` credential, so hand the user THAT URL. A `link` deck can carry MANY
  named links — one per channel ("Conference QR", "Newsletter"), each with its own secret,
  optional expiry and its own revocation (see `cue_share` below).

Set the tier at publish: `cue_publish` with `access: "public"` to open it up, or
`access: "gated"` (plus `invitees` to share with named people) / `access: "link"`. Leave `access` off
to keep the deck's current tier (a fresh deck stays gated). The publish result reports the
effective `access` tier — **relay it to the user** so they know who can open the link.

## Lifecycle: share changes, unpublish, delete

- **`cue_share`** (CLI `share`) changes sharing WITHOUT republishing: set `access`,
  replace `invitees`, or `rotateLink` to revoke every previously shared `?k=` URL.
  On a `link` deck it also manages NAMED SHARE LINKS — `addShareLink: {label, expiresInDays?}`
  mints one more labelled `?k=` door and returns its URL **once** in `mintedShareLink`
  (the secret is stored hashed and can never be read back), `revokeShareLink: <id>` kills
  exactly that one while every other link keeps working, and `rotateShareLink: <id>`
  replaces one link's secret. `rotateLink` remains the wholesale panic button: it burns the
  deck key and every named link at once. Every result lists the live links in `shareLinks`
  (labels and ids, never secrets).
  It also mints PER-RECIPIENT tracked links — `addRecipient` (a `label`, optional
  `email`, optional `expiresInDays`) returns a `…/view/<deck>?r=<token>` URL tied to one
  named viewer that opens the deck regardless of tier and attributes their opens in
  `cue_readers`; `revokeRecipient` kills one such link. And it manages NAMED SHARES —
  `grants: [{email, pin?, edition?, history?, fork?, edit?, present?, hideNotes?,
  presentUntil?, notify?}]` and `revokeGrants: [email]` (see below), plus NAMED EDITIONS — `promote: {edition, version, notify?}` (see below) — and a
  top-level `notify` for people newly added via `invitees`. It returns the resulting tier,
  fresh `viewUrl`/`liveUrl`, the deck's named shares and editions, who was emailed this
  call, and its active tracked links — hand those to the user.
- **Identified readers: the door (Pro).** A `gated` deck can take readers its owner never
  named: `cue_share {access: "gated", door: "signin"}` (the two may ride one call; `door` on any
  other tier answers 400). The bare `viewUrl` can then be forwarded anywhere — whoever opens it
  sees the title, cover and author and one sentence saying the deck is for identified readers,
  signs in with their email (a 6-digit code; an existing cue user clicks Continue), and gets a
  VIEW-ONLY grant: the deck lands on their shelf and `cue_readers` names them with how far and how
  long they read. `door: "email"` is the softer variant: the reader only TYPES an address (no
  code, not verified) and gets a personal 30-day link — no account, no grant, nothing on any
  shelf — and `cue_readers` lists them under group `email` with `verified: false`; say
  "unverified" whenever you report them. The door overrides the reader's Do Not Track for that deck (the gate says so).
  Disposable addresses are refused. `door: "invite"` closes it again — nobody new gets in and
  everyone who already came through keeps access. `doorNotify` (default true) is the owner's
  "email me when someone new opens it" switch. The result reports `door`, `doorNotify` and
  `doorReaders` (how many hold the deck only through the door); door readers are NOT listed in
  `grants` or `invitees` (email-door links are not in the tracked-link list either; they are
  counted in `doorReaders`). `removeDoorReaders: true` removes EVERYONE who came through the door at
  once — signed-in door readers and every unverified email-door link (destructive — confirm with
  the user; people also invited directly or through a group keep that access; it does not close
  the door) and reports `doorReadersRemoved`/`doorReadersKept`/`doorRecipientsRemoved`;
  `revokeGrants: [email]` removes one person however they got in, door included. Pro: a free owner gets 403 `needs_pro` (`capability: "door"`); at a lapse
  the door takes no new readers while existing ones keep access. Tell the user the gate will show
  the deck's title and cover to anyone with the link, and that its share card unfurls with them.
- **Reading terms (Pro, gated decks).** `cue_share {terms: "…"}` sets the author's reading terms:
  plain text up to 2,000 characters (rendered as text, never HTML; 400 over the limit or on a
  non-gated deck — `access: "gated"` may ride the same call). Every identified non-owner reader —
  invited people, door readers, tracked links, Call judges — must tick "I accept <author>'s reading
  terms" before ANY surface serves the deck (view, PDF/PPTX, live, the apps); the owner never is.
  Changing the text asks everyone again (already-open readers are re-asked); `terms: null` (or
  `""`) clears them and never needs Pro, setting them does (403 `needs_pro`, `capability:
  "door"`). Off the gated tier the stored text is inert. The result reports `terms` and
  `termsRev`; `cue_readers` reports `termsRev` and each identity's `termsAccepted` (`rev`,
  `acceptedAt`, `current`, `verified` — false for an unverified email-door link — or null).
  Delegates accept too: presenters, co-editors and forkers get `terms-required` (source reads, submits and
  `cue_fork` included) until they accept on the deck's page. The terms are the author's — cue records acceptance, and
  readers can report terms as abuse. Draft them with the user; never invent legal text.
- **Live links: one permanent URL to print.** Every published deck has a permanent live link,
  `permanentLiveUrl` (`/live/<deckId>`, in every `cue_publish` and `cue_share` result): scanned
  while ANY talk of the deck runs, it lands in that room; scanned before or after, it shows a
  waiting page that joins by itself when the talk starts — so a QR on a poster, lanyard or
  handout works for every talk. Public and gated decks need nothing (the tier still decides who
  gets in). On a `link` deck a scan reaches the access gate unless the owner sets
  `cue_share {liveLink: true}` (link tier only; any other tier answers 400), which lets each live
  room's key ride the scan — tell the user that a printed code then admits anyone who scans it
  during a talk; `liveLink: false` turns it off. The result reports `liveLink`. An author with a
  live author page can also opt in (on `/author-page`) to `/by/<handle>/live`: whatever they are
  presenting right now, on any deck. The presenter console's Audience drawer shows both links.
- **Presenters: someone else delivers the deck.** A grant with `present: true` makes that
  person a **Presenter**: they start and run a live talk of the deck's published version from
  `/present/<deckId>` (or their shelf, or the cue phone app, which can also put it on a screen
  they scan — into their own talk, never yours) on their OWN account — their own plan, their own
  caption time — without ever seeing the source. Can fork (`fork: true`) and co-editor (`edit: true`)
  include presenting. The talk stays yours and its record says who presented. **Presenters read
  the deck's speaker notes** (from the published version, never your draft) unless the grant sets
  `hideNotes: true`; `presentUntil` (ISO date, ≤ 366 days out) ends the right on a date without
  ending their view. When you share as Presenter, TELL THE USER that this person will see the
  speaker notes (or that you hid them). `cue_share`'s `grants` result reports each person's
  `rung` (viewer / presenter / fork / coeditor) and `seesNotes`. A presenter cannot change who
  else has access, re-share, publish or edit. Revoking the share or lowering it below Presenter
  ENDS any talk that person is giving on the deck at once (the audience is not left in a
  presenter-less room); once `presentUntil` passes they lose the room within a few minutes. A
  presenter can read the record of the talks they presented (in the app, without attendee
  contact details); `cue_talks` stays owner-only.
- **Two layers of notes.** SPEAKER NOTES live in the deck source (`notes` per slide) and travel
  with the deck to everyone who presents it. PRIVATE NOTES ("my notes") are a per-person layer
  only the account you act for sees — never in the source, never shown to the owner, another
  presenter or the audience, untouched by fork and propose. `cue_my_notes {deckId, action:
  "read" | "write" | "clear", versionId?, slideIndex?, ref?, body?, all?}` works on a deck you own
  or may present. READ FIRST: it returns `versionId`, `slideCount` and each note's opaque `ref`;
  slides are 0-based in that PUBLISHED version, and a write or clear by `slideIndex` must send that
  `versionId` (after a republish it fails with `version_changed` — read again). Edit or clear an
  existing note by its `ref`. A write answers `{written: {slideIndex, ref, applied}, orphaned}` —
  read again for the full list. A note follows its slide across republishes; one whose slide was
  removed comes back under `orphaned` ("the deck changed since you wrote this note") — tell the
  user, never move it by position. Put a reminder
  that is only for the presenter in private notes, not in the speaker notes.
- **Standing audiences: `groups`.** An owner can keep named audiences ("CS101 Fall 26",
  "Sales EMEA") and share a private deck with a whole one instead of retyping emails:
  `cue_share {groups: ["CS101 Fall 26"]}` takes group **ids or exact names** and shares as
  **Viewer** (view + comment on `current`). To give the group more, pass an object:
  `{group: "Sales EMEA", present: true}` (Presenter), `{group, fork: true}` (Can fork), with
  `hideNotes` / `presentUntil` as on a person's grant — a group never carries co-editing, pins
  or editions. Re-naming an already-shared group changes only the fields you pass. Members
  lose what the group gives when they leave it. **Rights combine upward:** a person's right is
  the UNION of their direct share and every group that reaches them — a direct Viewer in a
  Presenter group can present; removing their direct share leaves them the group's rights;
  leaving the group lowers them to the direct share (and ends a talk they were presenting if
  they no longer may). Presenters through a group read the speaker notes unless `hideNotes` —
  TELL THE USER. `groups` is **additive**: naming one group never disturbs another, and —
  unlike `invitees` — a later `invitees` call cannot wipe a group's access.
  `revokeGroups: [id-or-name]` withdraws one group's share. Private (`gated`) and secret-link
  decks only: a public deck already admits anyone with the link, so a group share there is
  refused. Sharing with a group never sends mail. The response reports the deck's `groups`
  (name, memberCount, `rung`, `hideNotes`, `presentUntil`, sharedAt), and each grant reports its
  EFFECTIVE `rung` plus `directRung` (null when only groups reach them) and `via`
  (`[{group, rung}]`). A person's direct PIN wins over a group's `current`, so a pinned person in a Presenter group reports `presents: false` — unpin them to let them present. The owner creates and edits the groups themselves in the app at
  `/people` — no tool creates one.
- **Team groups.** If the owner is a JOINED member of a team (set up in the app at `/people`;
  teams are operator-granted), they may also share their OWN decks with that team's groups —
  including its built-in "Everyone" — by **group id only** (the `grp_…` id the user copies from
  `/people` or the share panel; names resolve personal groups only). Everything above applies
  unchanged (rungs, union, never co-editing). A team group the owner is not a joined member of is
  an "unknown group" (404). Shares of a deck with a team's groups end when its owner leaves or is
  removed from the team; members who leave lose what the team gave them at once. Team groups
  in the response carry `teamName`, and `via` entries carry `team`. No tool creates, joins or
  edits a team. (In the app: admins can demote or remove other admins, never the owner; an
  invite stays valid after the admin who sent it is demoted, until accepted, declined, withdrawn
  or expired.)
- **Named shares: `invitees` vs `grants`.** Both admit specific people to a gated deck
  regardless of its access tier, but they differ in shape. `invitees` (set via
  `cue_publish` or `cue_share`'s `invitees`) is a **wholesale replacement list** — simplest
  when "these are the only people" is the whole ask, and everyone on it always sees the
  `current` version. `grants` (`cue_share`'s `grants` array) are **additive and
  idempotent** — each call names one more person without disturbing anyone else's share —
  and each grant can optionally **pin** that person to one published version
  (`pin: 2` or `"v2"` — the number `cue_history` shows — or a version id from
  `cue_list_decks`; an unknown number fails like an unknown id), freezing what they see
  across every future republish. The share panel's equivalent is "This version (vN)";
  a shared email and shelf card then say "vN", or **bind** them to a named **edition** (`edition: "<name>"`, see below) so
  they follow that release channel instead; omit both and they follow `current` like an
  invitee would. `pin` and `edition` are mutually exclusive on one grant. Reach for `grants`
  whenever you're sharing with named people incrementally, or need to freeze a specific
  version for someone (a reviewer, a client sign-off copy, an investor update that
  shouldn't shift under them); reach for `invitees` when replacing the whole list is what
  you mean. `revokeGrants: [email]` withdraws one person's share without touching anyone
  else's. A grant can also carry **fork** (`fork: true`) or **edit** (`edit: true`) — see the
  fork and co-author bullets below. Without either, a grant is view (+ optional comment,
  `history` lets them BROWSE the deck's published versions — the Versions drawer and
  `/view/<id>?v=N` — and nothing more: not revisions, diffs or restore; a co-editor already
  has it. The share panel's toggle is "Can browse versions").
- **Named editions: per-audience release channels.** An edition is a named pointer to a
  published version (a slug like `"board"` or `"investors"`) that moves ONLY when you
  explicitly `promote: {edition, version, notify?}` it — never on a plain republish. Think
  of it as a release channel: grants bound to `current` follow every publish automatically,
  a `pin`-ned grant is frozen forever at one version, and a grant bound to an edition sees
  whatever that channel currently points at — you decide when that audience updates by
  promoting, independent of both ongoing publishes and any pinned copies. Use it when one
  audience should move together on your schedule (ship a new board deck only when you say
  so, cut the investor edition over deliberately after a call, keep a public-facing edition
  a version behind an internal one) rather than either following every publish or being
  frozen forever. Promote the edition first, then bind grants to it with
  `grants: [{email, edition: "<name>"}]` — a `promote` and the `grants` that bind to it can
  land in the same `cue_share` call (promote is applied first). `promote`'s own `notify:
  true` emails everyone currently bound to that edition when you move it, same opt-in model
  as a grant's `notify`. `cue_share`'s response reports the deck's `editions` (name,
  current version, when it was last promoted) alongside its `grants`.
- **Fork: let a recipient branch your deck.** Add `fork: true` to a grant
  (`grants: [{email, fork: true}]`) and that person may **fork** the deck — copy its source
  into a NEW deck they own outright and edit freely — with `cue_fork`. They fork the EXACT
  version they're served: a `pin`-ned grantee forks that frozen version's source, an
  edition-bound grantee forks the edition's current target, everyone else forks `current`.
  A fork carries source ONLY — never your sharing, editions, version history, or tracked
  links — and is untouched by later revoking the grant (revoke stops FUTURE forks, not decks
  already made). `cue_fork` requires that the caller can author on cue (any plan with
  authoring — a brand-new account gets a small free allowance) and holds a fork
  grant; it returns the new `deckId` + `jobId` (watch with `cue_job`, then `cue_submit` /
  `cue_publish` as usual). As the upstream owner you can see who forked your deck (forker +
  the version they branched from) — provenance is two-party, never public.
- **Edit: let a recipient co-author your deck.** Add `edit: true` to a grant
  (`grants: [{email, edit: true}]`) and that person may **co-author** YOUR deck: pull the
  version they're served and `cue_submit` revised source back to it — which
  replaces the deck's draft + rebuilds its preview, exactly as your own submit does. Unlike a
  fork (a new deck they own), an edit grant writes to YOUR deck: you stay the owner, only YOU
  publish (which mints the version), and the grantee gets no publish/share/delete/grant power.
  The person must be able to author on cue (the grant admits a co-author, it never expands their
  authoring rights). They edit the EXACT version they're served — a `pin`-ned grantee pulls
  that frozen source, never a newer one or your unpublished draft-ahead. The deck has ONE
  draft slot, so owner and co-author writes are last-write-wins (no locking) — coordinate.
  When you publish a co-author's draft, the new version is credited "edited by <their email>"
  in the version history. Revoking `edit` stops their next pull/submit immediately.
  From the RECIPIENT'S side: `cue_list_decks` lists only decks you own, so find decks shared
  with you via `cue_shelf` (`cue_shelf {canEdit: true}` narrows to the ones you may co-author)
  and pull the deckId from there.
- **Pins — put the next deck at the top.** `cue_pin {deckId, pinned}` floats a deck to the top
  of the owner's library on the web home AND in the cue phone app; `cue_list_decks` (ordered
  pinned first, then most recently updated) and `cue_shelf` both report `pinned`. Use it for
  the deck they need NEXT — the one you just built for tomorrow's talk — and unpin it after.
  The pin is PER PERSON: it does not move the deck for anyone it is shared with, and it changes
  nothing about the deck itself, not its content and not who can see it. Works on a deck you
  own and on one that is on your shelf.
- **Proposals — the PR loop.** Fork + gate = merge protection. When you own a **fork**
  (made with `cue_fork`), you can propose its changes back to the deck you forked from:
  publish your fork, then `cue_propose {deckId: <your fork>}`. The upstream target is derived
  from the fork's provenance — you can only propose to a deck you actually forked, and you
  inject source only from your own fork. The response includes `staleBase: true` when the
  upstream has published past the version you forked (accepting is verbatim, not a rebase, so
  a stale base warns the owner to review — rebasing your fork onto the newer upstream is a
  manual re-pull-and-edit, there is no auto-merge). As the **upstream owner**, list incoming
  proposals with `cue_proposals {deckId: <your deck>}` (each names the proposer, the fork, and
  `staleBase`); `cue_accept_proposal {deckId, proposalId, fromVersionId}` runs the proposer's
  source through YOUR gate and makes it your deck's next **draft** — it does NOT auto-publish
  (review, then `cue_publish` to mint the version, whose author is the proposer). Pass
  `fromVersionId` = the version you reviewed (the one `cue_proposals` returned): the accept is
  pinned to it, so if the proposer re-proposed a newer version since you listed you get a
  "changed since you reviewed it" error instead of silently accepting the new bytes — re-list
  and review. Accepting REPLACES your current draft (verbatim whole-source swap).
  `cue_decline_proposal {deckId, proposalId, reason?}` declines without touching your source.
  Only the upstream owner accepts/declines; only the proposer withdraws; there is one open
  proposal per fork (re-proposing revives it).
- **Opt-in email: `notify`.** Sharing never emails anyone by default — a grantee only gets
  a "someone shared a deck with you" message if you ask. Set `notify: true` on one
  `grants` item to email that person now, on `promote` to email everyone currently bound
  to the edition you just moved, or the top-level `notify: true` (with `invitees`) to
  email everyone NEWLY added to the invitees list this call; people already shared with are
  never re-notified. The email carries the deck's title, a cover thumbnail, your identity
  as sender, and a link to `/view/<deck>` that carries NO secret — the recipient signs in
  and their grant or invitee membership does the rest, so it's safe to forward. Sends are
  best-effort and rate-limited; a throttled or failed send never fails the share itself —
  `cue_share`'s response reports who actually got the email in `notified`. A recipient
  can come back as `notified: false, reason: "unavailable"` — the message could not be
  delivered to that address; the access grant is unaffected, so send them the link directly.
- **`cue_readers`** (CLI `readers`) is READ-ONLY read receipts for a published deck:
  totals (opens, unique viewers, average read-depth), a per-language split, a **`byVersion`**
  split (readers by the published version they last opened), per-recipient
  stats for every tracked link (opens + furthest slide), `identities[]` (each identified
  reader, with **`lastVersion`**: "v3", "same as v3" for a set-live entry, or `null` when
  not recorded — pre-D361 opens and Do-Not-Track browsers), and the most recent sessions.
  Use it to answer "did she open the version I sent?".
  Named identity shows for gated invitees and tracked-link recipients; `public`/`link`
  opens are counted but stay anonymous. Each identity carries `group` — `invited` (someone the
  owner named), `door` (signed in at the deck's door) or `email` (an unverified address) — and
  `verified` (false only for an unverified address); report door readers as such. `cue_share` sets up tracked links, `cue_readers`
  reads the results.
  **Read `aggregateOnly` and `detailUntil` before you interpret the payload.** When the
  response carries `aggregateOnly: true` (D216 — the deck owner is on the free lane, or the
  account is on hold), `totals`, `byLanguage` and `byVersion` are REAL and complete while `identities`,
  `byRecipient` and `recent` are deliberately EMPTY. The deck WAS read: report the totals ("42
  opens by 17 people, 12 of them in German") and say that seeing WHO opened what, per viewer and
  per slide, is Pro. Never read those empty arrays as "nobody opened it" — that is the one wrong
  sentence this field exists to prevent. `detailUntil` (an ISO instant) is the other, narrower
  case: the owner's author plan ENDED then, so the per-viewer rows are FROZEN at that moment
  rather than withheld — they are real but cover only sessions opened before it, while `totals`
  and `byLanguage` still count everything. Say so plainly ("18 opens, 6 of them since the plan
  ended — those are counted but not identified; Pro brings the names back") rather than
  presenting a stale list as the whole story. The two fields are mutually exclusive and both are
  absent entirely for every other plan, so an unchanged payload means unchanged meaning.
- **`cue_talks`** (CLI `talks`) is READ-ONLY, owner-only (D229): the talks a deck was
  presented in, not who read the deck but what the presenter actually SAID while presenting
  it. Without `talkId` it returns the list, newest first — date, duration, how many joined,
  how many caption lines, share status, recap status. With `talkId` it returns that talk's
  transcript, slide-keyed, in `lang` (default: the language the presenter spoke; pass one of
  the languages the talk was captioned in — its spoken language plus the languages the presenter
  chose to offer; a talk with none chosen is readable in its spoken language only — to read it in another). A language
  other than the spoken one is a cached machine translation: when the response carries
  `translated: true`, say so when you relay it. This is presenter speech only — audience
  microphones are never captured, attendees are never named, and nothing in the payload
  identifies a viewer. The use is revision, not readership: read what was actually said on
  slide 7 and fix the slide to match, or catch a claim the deck doesn't back up yet. `recap`
  is the presenter's AI-written recap in `lang` when one exists — headline, summary,
  slide-keyed chapters, takeaways, questions the presenter raised, quotes (`version: 1`; `qa`
  is reserved for a future extension and always empty) — and `recapLanguages` lists every
  language it has been written in. If you relay a recap, say it is AI-written and may be
  wrong. Writing one is a presenter action in the product (Pro) — this tool does not write
  one for you. With `talkId` the payload also carries **`responses`**: what the ROOM
  answered (see *Asking the room*) — one entry per ask and round with the question, the
  option labels, `n` and the aggregate, anonymous and never per person. The talk page
  downloads the same thing as CSV/JSON.
- **`cue_unpublish`** (CLI `unpublish`) takes a deck offline reversibly: every viewing
  surface refuses non-owner requests, but artifacts and version history are kept and
  `cue_publish` restores it. Prefer this over delete whenever the deck might return.
- **`cue_delete_deck`** (CLI `delete <id> --yes`) is PERMANENT — record, all published
  versions, PDFs/thumbnails, and studio source are gone with no undo. Always confirm
  with the user before deleting, and suggest unpublish if they only want it offline.

## Calls: a call for decks, entered with published decks (D362)

A **Call** is a call for decks — a hackathon, a pitch round, a class assignment: a brief, a
deadline in a stated IANA time zone, and an entry mode. People ENTER a Call with a published
deck of their own. **Entering is not submitting**: `cue_submit` stays the draft → validation-gate
step; a Call only ever sees published versions.

- **Organizer** — `cue_call_create {title, deadlineAt, tz, entryMode, allowList?, brief?,
  templateDeckId?, maxEntries?}` opens one. Calls are not part of any plan: cue enables them per
  account on request, so a `needs_grant` refusal means tell the user to write to the address in its `contact` field to have Calls turned on —
  do not retry, and do not suggest upgrading. `maxEntries` defaults to (and is capped at) the entries
  limit on that grant. `entryMode`: `link` (anyone with the link, or who types the slug on `/call`),
  `allow_list` (only the emails and `@domains` in `allowList`), `secret_key` (a `?k=` link — the
  result's `secretLink` is shown ONCE: relay it to the user then). Share the returned `link`.
  `cue_call_entries {call}` reads the table — every entry with its status (`active`,
  `withdrawn`, `replaced`), entrant emails, judges' opens and the roster; `format: "csv"` gives
  the same rows as CSV. Judges calling it get active entries without emails. Judges, the roster,
  closing and archiving are in the Call's page on cue.
- **Deciding (D370)** — an organizer records a decision on an active entry with `cue_call_decide
  {call, entryId | deckId, decision, note?, organizerNote?}`: `decision` is `pending`, `shortlisted`,
  `accepted`, `declined` or `waitlisted`. The ENTRANT sees the decision and `note` (at most 500
  characters) on the Call page; `organizerNote` (at most 2000) is private to the Call's organizers —
  never the entrant, never a judge. Back to `pending` clears the decision time; an empty string
  clears a note. A judge is refused `organizer_only`. Deciding sends no email and changes nothing
  else: before the deadline the entrant may still withdraw or replace (a replacement keeps the
  decision and both notes). The entrant sees `note` only once the decision is not `pending`. `cue_call_entries` rows carry `decision` for organizers; the CSV gains `decision`
  and `decided_at`. Confirm the decision with the user before recording it — the entrant sees it.
- **Participant** — publish the deck first (`cue_publish`), then `cue_call_enter {call: slug,
  deckId, team?, members?, key?}`. One entry per person per Call: entering again before the
  deadline replaces it. `cue_call_withdraw {call, deckId}` pulls it out until the deadline. While
  the Call is open an entered deck takes no publish slot and cannot be deleted (withdraw first);
  unpublishing stays allowed. Entering is free for everyone, on every plan.
- **The deadline is a cut, not a lock.** Publishing after it is allowed. Each entry's **counting
  version** is the deck's last original publish at or before the deadline; organizers and judges
  land on it, and later versions show to them marked "after deadline" (hidden if the organizer
  closes the Call). So the edit loop does not change near a deadline — just publish in time.

## Programmes: a conference run of show built on Calls (D370)

A **Programme** is an event's run of show: stages, slots (a talk title, a speaker, a time), roles
and holds. A Call attached to it is the speakers' **slides deadline**; an entry in that Call binds
to the speaker's slot, and the slot's version is what plays on the day.

- **Organizer agent** — `cue_programme_create {title, tz, brief?, startsOn?, endsOn?, venue?}`
  (Programmes are enabled per account: a `needs_grant` refusal with `capability: "programmes"`
  means tell the user to write to the `contact` address — do not retry). Then open the slides
  Call with `cue_call_create {…, programmeId}` — it needs the programme's grant, not a Calls
  grant; usually `entryMode: "allow_list"`. Load speakers with `cue_programme_import {programme,
  csv, callId?}`: a header `name,email,title,format,track` (+ optional `stage,starts_at,ends_at`,
  ISO times with an offset); each row upserts a planned slot keyed on (email, title) and adds the
  email to the Call's allow list, so re-importing a corrected sheet is safe. Report the result's
  `skipped` lines to the user. A result with `notice.code: "mail_capped"` saved the slots but held
  some speaker emails back (a daily limit on new addresses): tell the user they go out later.
  `cue_programme_slots {programme}` lists slots with `versions
  {counting, latest, approved, stage}` (stage = what plays: approved, else counting — or latest when
  the programme's `alwaysLatest` is on); `action: "upsert"` adds or edits slots. When an entrant's
  email differs from the slot's, bind by hand with `cue_programme_bind {slot, entry}`; approve what
  plays with `cue_programme_approve_version {slot, ordinal | "latest"}` — confirm with the user
  first (a stage member may approve "latest" on their own stage only; the slot's `approval` names
  who approved). Programme organizers are organizers of its Calls (`cue_call_entries`, `cue_call_decide`);
  committee members are its judges. **Recap term (D374)**: `cue_call_create {…, programmeId,
  recapTerm: true, recapTermNote?}` asks the speakers to agree, optionally, that the organizers may
  publish the recap of their talk; a speaker who agrees lets the organizers' share of that talk (after
  the event) open its RECAP — never the deck or the transcript — to the share's audience even when
  the deck is gated. Only ask for it when the user wants it. It can be turned off later, not turned on
  or reworded once the Call has entries; `cue_call_entries` rows carry `recapConsentAt` for
  organizers. Each slot also carries `readiness` (`booked`, `preloaded`,
  `warnings`: `terms_block_room` — the deck's reading terms keep the room invitees-only;
  `screen_not_owned` — nobody on the programme owns the stage's screen; `not_preloaded` — the
  screen does not hold the stage version within the hour): report red ones to the user. `remote`
  (`pairedAt`, `readyAt`, `devices`) says whether the speaker has paired a phone and said "ready" —
  read-only; pairing is the speaker's own tap on cue, never an agent's. **The day
  itself is not yours to run**: organizers and stage members start, hand back and close slots
  from the stage console and the phone remote — there is no tool for it; never try to start a
  talk for a programme. To print or post a stage's audience link, use the per-stage live page
  `<link>/live/<stageId>` (one permanent address per stage: it follows whatever talk is on that
  stage and shows the up-next card between talks; `GET /api/programmes/<slug>/stages/<stageId>/live`
  is its public read, once the programme is published; it opens the talk at the deck's own gate — the
  in-room QR with the room pass is printed by organizers from the stage console, never by you). Sharing a finished talk's recap is the
  organizers' own click on the programme page — no tool.
- **Speaker agent** — `cue_talks` with no `deckId` returns `{ upcoming }`: each slot with its
  programme, stage, times, the Call (`call.slug`, `call.deadlineAt`), the bound deck, the version
  ordinals, `phase` (`deliver` — enter the Call with a published deck; `delivered`; `cut` — the
  deadline passed; `day`) and `late`. To deliver: publish the deck, then `cue_call_enter {call:
  call.slug, deckId}`; its result carries the fixed `consent` line (entering means the version may
  be presented in the slot and followed by the room) — relay it to the user before entering. When
  the Call carries a recap term (the Call page shows it; the enter result's `recapTerm`), ASK the
  user and pass `recapConsent: true` only on their explicit yes (`false` declines — optional, the
  entry stands either way; omitted on a replace keeps the earlier answer). The speaker can give or
  withdraw it at any time on the Call page; never change it for them. A
  listed speaker on a free account may author and publish ONE deck for the slot (the first it
  submits, imports or copies from the Call's template after being listed) and may connect an
  agent (a token or OAuth connection) to do it; that connection authors the same one deck and
  nothing more. On it: submit, the draft file tools, asset uploads (`cue_request_upload`), publish,
  rename and resolving notes — not review links, editions, grants, talks, settings or documents. Any other deck or action gets
  the author-plan refusal — say so, do not retry. The speaker's web home shows the same slots as an "Upcoming" strip.

## Notes: the owner's change requests and reviewer feedback — address, then resolve

Two kinds of note reach you through the same tools:

- **Owner notes** (`kind: "owner"`) — the deck's owner writes them in the studio's
  "Notes for your agent" panel while looking at the slide: "What should change?". They
  usually carry an anchor — `slideIndex` (0-based) and often an `elementKey`, the content
  key they picked on the slide with Select element (e.g. `s3.title`; find it in
  `content.ts` or the slide source). Treat each one as an instruction from the user.
- **Reviewer notes** (`kind: "review"`) — per-slide or whole-deck notes left on a
  published deck through a review link (no account needed), with an optional `reaction`
  (works/unclear/cut).

Work them as a loop:

- **`cue_feedback`** (CLI `feedback <deckId> [--all]`) lists notes — UNRESOLVED only by
  default, `all: true` for the full history — each with `id`, `kind`, `slideIndex` (null =
  whole deck), `elementKey`, `reaction`, `body` and the author. Check it at the START of
  every edit on an existing deck.
- **Which language a note is about.** A reviewer reads in the language they choose, and every
  note carries **`lang`** — the deck language its author was looking at (null on older
  notes). Feedback about *wording, translation, typography or overflow* in that language →
  change only that language's strings in the content keys. Feedback about *substance,
  structure, visuals or slide order* → apply it across ALL languages, keeping every
  language's content complete (the multilingual rule below). When it is unclear which,
  apply it across languages and say so in the submit `message`.
- **Which version a note is about.** A reviewer's note carries **`versionId`**, the
  published version they reviewed; `stale: true` means the deck has been republished since
  (`latestVersionId`), so its `slideIndex` may point at a different slide now — find the
  slide in the current source by what the note describes before you edit it.
- Revise the deck source to address the notes, then `cue_submit` and — when the
  user wants it live — `cue_publish`: the same edit → gate → publish loop as any change.
  Say which notes the change addresses in the submit `message` — it is what the owner
  reads in the studio's Changes timeline.
- **`cue_resolve_note`** (CLI `resolve-note <noteId>`) marks a note resolved once you have
  addressed it, clearing it from the studio's open notes, the owner's inbox and
  `cue_feedback`'s default list. Resolve only the notes you actually fixed.

## Studio overrides: the owner's own edits survive your submits

The owner can rename the deck in the studio header and fix copy by double-clicking text on
the slide. Those edits are stored as *studio overrides* in `overrides.json`, which cue
manages: you cannot submit it (a submitted one is ignored), and cue RE-APPLIES it on every
submit. The rule is **most recent edit wins, field by field**: a field you leave unchanged in
source keeps the studio's value; a field you change in source (the title in `index.ts`, a
content key's text in one language) replaces it. Per-slide titles, speaker notes and hide/show
toggles from the studio follow their slide (matched by title, or by position when the slide count is unchanged);
one whose slide you removed is dropped, and the studio's slide order survives only while every
slide still maps. The owner can also re-crop any `<Image>` in the studio (select, drag to pan,
zoom, Reset): that crop (`imageCrops`) follows its slide too, and names the image by its `asset`
plus which copy of that asset it is on the slide (first, second, …) — so other images coming and
going (a build step) never move it, but adding or removing an earlier copy of the SAME asset in
source does. It replaces your `position` for that image only. Images whose `asset` is a data: URI
cannot be cropped.
A Restore never inherits studio edits — it is exactly that revision.
`cue_list_files` and `cue_submit` list the overridden fields as `studioOverrides` (the draft
workspace never holds `overrides.json`).
So if the owner renamed the deck, the deck's name is `studioOverrides.title`, not
necessarily the title in `index.ts` — don't "fix" one to match the other unless asked. The
job log (`cue_job`) says which overrides were re-applied and which your source superseded.

## Before building a NEW deck: one round of intake

Extract from the user's request first: audience & occasion, the ONE takeaway (or call
to action), tone/style, key proof points or numbers, language(s), and rough length. If
material pieces are missing, ask in a SINGLE round — 2–4 questions max, each offering a
sensible default — then build. If the host exposes a structured question tool (Claude
Code: `AskUserQuestion`; Codex: `request_user_input`; claude.ai: plain chat), use it for
the intake round — one call, every question at once, each option carrying its default.

For a thin request (no audience, no length, no language, no visual direction given),
the round always asks these four, each with a concrete default: **audience & occasion**;
**language(s)** (default: the language the user wrote in; offer a multilingual option —
one deck, every language, see below); **length** (default: the outline's beat count,
typically 8–14 slides for a talk); **visual identity** (offer 2–3 `cue_library`
identities matching the request's mood, plus "a look of its own" when none is a strong fit;
default the best fit).

When the occasion is a **workshop, a lecture, a class, a quiz or a live poll**, add one
more question to the same round: should the room ANSWER — and how many times? Two or
three asks across a session is the usual answer; the deck then declares them and the
slides draw the results (see *Asking the room*). Don't ask it for a keynote, a pitch or
anything mainly meant to be read.

Never ask what the request already answers, and never ask a second round: make the
remaining calls yourself. When the request is already complete, or the user says "just
build it," skip the round — print one line before building instead: "Defaults I'm
using: <audience> · <lang> · <~N slides> · <identity> · gated, not published" — so the
user can redirect early. Never re-ask after the draft; further changes go through
edits. If the user gives source material, ground the deck in it before asking anything.

## Creating a NEW deck: outline first, then the kit and the voice

Three roads lead to a new deck. Take the KIT by default: the platform serves a pattern
library — an IDENTITY (the deck's visual and kinetic voice), nine proven slide PATTERNS and
a DIAGRAM kit. The identity is the deck's voice (retint it if its palette does not fit,
step 5); the patterns and diagrams are components you use where they are the most impactful
slide for a beat and skip where they are not. Take a whole TEMPLATE when the user wants
exactly that look (below). And go FROM SCRATCH — your own `theme.tsx`, `slides.tsx`,
`content.ts`, `index.ts` on `@/lib/types` and `@/lib/primitives`, no library files — when no
identity fits the deck you want, or the user asks for a look of their own. Scratch is a
road, not a loophole: every convention above and every craft rule still applies, and it
costs the most tokens and carries the most layout risk, so take it on purpose. The goal is
the presentation, not library coverage.

1. **Outline the argument first** — before any visual decision, write the beats:
   one line of intent per beat ("land the one number", "show the transformation",
   "how it works, four stages"). The outline's length IS the deck's length.
2. **Choose the road — evaluate before you commit.** Read `cue_library`: each identity
   carries a `voice` paragraph, a `scheme` and a `swatch`. Read `cue_templates` for the
   eleven worked examples and their taglines. Judge
   each candidate against the audience, the subject and any brand in the brief, and
   decide by strength of fit, not by habit:
   - an identity is a STRONG fit (its structure, motion and register suit the deck; a
     palette mismatch alone does not disqualify — retint, step 5) → the kit. Pick ONE
     identity for the whole deck; audience/brand/mood decides, never variety;
   - the user wants exactly a template's look, or the closest template IS the deck → the
     template road (below);
   - nothing is a strong fit, or the user asked for a look of their own → from scratch.
   Say which road you took and why in the defaults line ("… · press, retinted to the
   brand · …" or "… · from scratch: no identity carries a children's picture register · …").
   Note each pattern's `bestFor`, `density`, `slots` and `steps` if you took the kit.
3. `cue_create_deck {title, identity}` — creates the deck AND vendors the same five
   library files into its draft workspace SERVER-SIDE: `contract.ts`, `patterns.tsx`,
   `diagrams.tsx`, `id-<identity>.tsx`, `identity.tsx`. They never enter your context:
   for the kit's API call `cue_library {identity}` — exported signatures of contract,
   patterns and diagrams plus that identity's token names and timings, ≈9 KB — and
   `cue_grep` a vendored file for one specific spot; never `cue_read_files` them whole.
   Never edit these — they ARE the kit; your deck is the files you write next.
4. **For every beat, decide by impact: pattern or bespoke.** A pattern when it is the
   best slide for the beat (gate-clean, in the voice, cheap). A slide you build yourself
   when it is not: an illustration in inline SVG, a big-picture slide with few words, a
   diagram that assembles step by step, an animated reveal on `motion/react`, the moment
   the audience will remember. Bespoke slides are expected — they are where a deck earns
   its character. A child's deck wants pictures; an engineer's deck wants the code or the
   matrix; a pitch wants the one chart that proves it. Bespoke still speaks the voice:
   theme tokens for colour, `identity.vocab` (`V.*`) for type roles, `identity.motion`
   (`M.*`) for entrances and stagger, copy read by key — never a second palette, an
   unrelated typeface, or literal prose in a slide.
5. **The palette and faces are a starting point, not a cage.** The identity's structure,
   motion and roles are what you keep; its colours and typefaces are tokens the deck may
   own. When the seeded palette does not fit the subject, the audience or a brand (a
   volcano in marker green), override the tokens in `index.ts` and keep everything else:
   `theme: { ...identity.theme, tokens: { ...identity.theme.tokens, "--deck-bg": …,
   "--deck-fg": …, "--deck-accent": …, "--deck-accent-2": …, "--deck-font-display": … } }`.
   All nine tokens (`bg fg muted surface accent accent-2 font-display font-body font-mono`)
   exist on every identity and every pattern, diagram and identity paints only through
   them, so a retint is safe; keep contrast legible, and set `fontsHref` (a Google Fonts
   URL) on the theme if the deck wants real faces — the build self-hosts them. Pick the
   base identity for how it moves and lays out; own the colour.
6. Author `content.ts` (every string of every slide — pattern slots under your key
   prefixes, your own keys for bespoke slides — in every language; name the subject on
   a slide), `slides.tsx` (patterns wired to keys with `steps: PATTERN_STEPS.X`; bespoke
   slides with their own `steps` and `useStep()`/`<Step>`; the signature slide with
   `transition: identity.motion.accent`), and `index.ts` (manifest with the identity's
   theme, retinted or not, and `transition: identity.motion.transition`).
7. `cue_submit {deckId}` — sends the workspace through the gate (the deck itself was
   already created in step 3 by `cue_create_deck`).

Composition rules: open with `Cover` (or a bespoke opener in the voice), close with
`Close`; give the deck one signature moment; vary density and rhythm — two dense grids
back-to-back is a smell, and so is a deck that never draws.

Whole-deck templates remain for two cases: the user wants exactly a template's look, or
you need a worked example. Read `cue_templates`, then take one with
`cue_create_deck {title, template}`.
A template is NOT a library composition and holds nothing to reuse: each one is
a bespoke, complete deck built on its own `theme.tsx` plus `@/lib/primitives`,
with no `patterns.tsx`, no identity, no `PATTERN_STEPS`. Read one for craft and
motion in situ — how a chart draws itself, how a build paces a point, how a
signature moment is staged — not for the pattern library. **A template's slide
count is scaffolding, not a target**: your deck has as many slides as the
outline has beats.

## Multilingual decks: one deck, every language

A deck is multilingual by construction — when the user wants a deck in N languages,
build ONE deck, never one deck per language:

- List every language in the manifest's `languages` (and pick `defaultLang`).
- Give EVERY content key — including the speaker-notes array — a value in every
  declared language.
- Viewers switch live from the language control on whatever surface they are reading —
  the viewer/live shell's own picker, or the deck's in-canvas one on a raw artifact URL.
  Nothing is resubmitted or republished per language. Projected surfaces (a cast screen,
  a stage display) deliberately show no control at all: their language is the screen's
  steward default plus the presenter's override, never the room's.

Splitting languages across separate decks breaks the model: N share links instead of
one, N copies to keep in sync on every edit, and live language switching goes dead.

Each person also has an ACCOUNT reading language (`/account`, or `GET`/`PATCH
/api/me/prefs` — session-only, so you read its effect rather than setting it): deck lists
and titles come back resolved in it (falling back to the deck's `defaultLang`, then `en`),
and a signed-in viewer's slides and captions start there. It never changes what a deck
declares, so it is no reason to build anything differently — declare every language the
user asked for and let each reader land in their own.

## Asking the room

A deck can put a question to the people in front of it, count every phone once, and draw
the answer as a slide. The author declares **asks** in the manifest, marks which slide
asks and which slide reveals, and reads the result with one hook. The platform does the
counting and the anonymity; the deck does the drawing.

Reach for an ask when the room is meant to participate: a **workshop** (open on what
people actually do, branch the session on the answer), a **lecture check-in** ("how clear
was that?" after a hard section), a **quiz**, a **conference poll**. Do NOT reach for one
when the deck will mostly be READ — a published deck open in someone's browser, an
emailed PDF and an export all show the slide's EMPTY state, because nothing there sends a
room. So an ask may never carry the argument: the reveal slide has to say something true
with no answers in it, and the ask makes it sharper when a room is there.

### Declaring the asks

`asks` sits on the manifest beside `languages` and `content`, keyed by a stable id. It is
pure data — the platform validates every answer against it without running deck code.

```ts
// index.ts
const asks: PresentationManifest["asks"] = {
  ship: {
    kind: "choice",
    question: { en: "How do you ship to production today?", de: "Wie deployt ihr heute?" },
    options: {
      main:    { en: "Push to main",   de: "Push auf main" },
      release: { en: "Release branch", de: "Release-Branch" },
      manual:  { en: "Manual",         de: "Manuell" },
    },
    reveal: "presenter",          // the default: the room sees it when the presenter reveals
    floor: 3,                     // …and only once three people have answered. Default: 0.
  },
  clear: {
    kind: "scale", min: 1, max: 5, reveal: "live",
    question: { en: "How clear was that?", de: "Wie klar war das?" },
    labels: { min: { en: "Lost me", de: "Abgehängt" }, max: { en: "Crystal clear", de: "Glasklar" } },
  },
};
```

**The five kinds**, and what one person sends:

| kind | one answer is | declares | the slide reads |
|---|---|---|---|
| `choice` | one option key | `options` (2–12), optional `input` | `counts`, `top` |
| `multi` | 1..`max` option keys | `options` (2–12), `max?` (default: all) | `counts`, `top` |
| `scale` | an integer in range | `min`, `max` (span 1–10), `labels?`, `input?` | `histogram`, `mean`, `median` |
| `number` | a number, snapped to `step` | `min` < `max`, `step?` (1), `unit?`, `bins?` (2–20, default 10) | `bins`, `mean`, `median` |
| `rank` | the full order of the options | `options` (2–8) | `scores` (Borda), `firsts`, `top` |

Every kind also takes `reveal?` and `floor?` — see *Reveal and the floor*.

Every bound, enforced at build: **≤ 24 asks** per deck; ask id `/^[A-Za-z][A-Za-z0-9_-]{0,31}$/`,
option key `/^[A-Za-z0-9][A-Za-z0-9_-]{0,31}$/`; `question` ≤ 300 characters per language,
option and unit labels ≤ 120; `question`, every option label and `labels.min`/`.max` are
LocalizedStrings that must carry a string for **every language in `manifest.languages`** —
a phone answering in German and a phone answering in Arabic land in the same bar, which is
the whole point; no control characters anywhere; and no id, option key or language tag may
be a name that already exists on `Object.prototype` (`toString`, `valueOf`,
`constructor`, `hasOwnProperty`, …) — those are refused, not sanitized. `reveal` is
`"presenter"` (default) or `"live"`. `floor` is a whole number 0–20 (default 0); anything
else — a fraction, a negative, `"3"` as a string — is refused at build. `input: "inline"`
says the SLIDE draws its own answer control instead of the shell's sheet, and is accepted on
`choice` and `scale` only.

**Which slide asks is data on the slide**, never an index on the ask:

```ts
{ Component: HowYouShip, navTitle: …, notes: …, asks: ["ship"] },      // arriving opens it
{ Component: WhatTheRoomSaid, navTitle: …, notes: …, reveals: ["ship"] }, // arriving reveals it
```

Both are `string[]` of ids the manifest declares (≤ 24, no duplicates). Arriving at a slide
that `asks` opens those asks; **leaving does not close them**, so a phone that was slow
still counts. Arriving at a slide that `reveals` closes and publishes them. Because the ids
travel on the slide, reordering slides in the visual editor carries them along.

### `useRoom()` — the hook

```tsx
import { useRoom } from "@/lib/room";

const room = useRoom();      // { mode, role, joined, ask, suggest }
const q = room.ask("ship");  // a pure read — safe during render, safe every render
```

`room.mode` is the whole trust model in one field:

- **`"off"`** — no room ever spoke to this deck: the studio before you press preview, a
  reader on the published link, a PDF or PPTX export, a shell older than this engine.
  Every ask is idle, every aggregate is `null`. This is why an export can never show
  invented numbers, and it is the state you must design for.
- **`"live"`** — a real room. An aggregate is present exactly when this surface is allowed
  to see it (see *Reveal and the floor*); missing means "not yet", and the deck must never
  compute one itself.
- **`"sample"`** — the studio preview and the presenter's rehearsal. The numbers are
  synthesized inside the engine from the ask's own options, deterministically from its id,
  so the chart you lay out is the same on every reload. Every such ask reports
  `sample: true` — say so on the slide.

`room.role` is `"presenter"` (the console), `"viewer"` (a phone) or `"output"` (a stage
display, a cast screen). `room.joined` is how many people are in the room, or `null`.

`room.ask(id)` returns one flat `AskState`, so a slide never switches on `kind` to read a
number:

- `spec` — the declaration, or `null` for an id nothing declares (guard with it and render
  nothing); `status` `"idle" | "open" | "closed"`, plus `open` / `closed` / `revealed`
  booleans; `round` (0 until first opened, then 1, 2, … per reopen); `n`, how many answered
  this round — **always known**, even before the aggregate arrives; `floor`, the number
  this ask declared (0 when it declared none) — also always known, including while the ask
  is idle, so a slide can write "{q.floor} answers, then the wall fills" from the real
  number instead of a hand-typed one.
- The aggregate, spread flat and `null` unless this kind produces it AND it is visible:
  `counts`, `top`, `mean`, `median`, `histogram`, `bins`, `scores`, `firsts`.
- `options` — the option keys in the AUTHOR's display order. **Enumerate with `q.options`,
  never `Object.keys(q.counts)`** — the stored manifest is JSON in a database column and
  does not keep key order. `label(key)` gives one option's label, and `question` the
  question, both in the language this copy of the deck is being read in.
- `labels` — a scale's two end captions, `{ min, max }`, and `unit` — a number ask's unit
  (`"%"`, `"min"`). Both resolved into the reader's language, both `null` when the spec
  declares none or the kind has none. Caption a gauge or an axis from THESE rather than
  from a content key repeating the same words: a copy in `content.ts` is a second version
  of a string the phones were shown, free to drift from it.
- `mine` — what THIS device answered this round (`null` on a stage, a console, or before
  the person answered). Rendering from it is local: the phone already knows its own answer,
  nothing is fetched and nobody else's is.
- `sample` — the numbers on screen are synthesized, not a room's.
- `respond(value)` — record this answer. A no-op unless the surface is a viewer and the ask
  is open, so a deck cannot answer on a stage or twice. You only need it for an
  `input: "inline"` ask; every other ask is answered in the shell's own sheet, which cue
  builds from the schema on web, iOS and Android.

`room.suggest({ slideIndex, stepIndex?, reason? })` is the deck's one influence over the
room, and it is only ever a suggestion: it lights a button in the presenter's console
(`reason` ≤ 120 characters, shown as plain text) and moves nothing. `null` withdraws it, it
is ignored off a presenter surface, and it de-duplicates, so calling it from an effect that
re-runs every frame costs one message. **Slides stay linear** — the deck may never navigate.

**Active content language (D283, requires the worker SDK update).** `useContent()` from the allowed
`@/lib/content` module exposes readonly `lang` alongside `c(key)`, `entry(key)` and `keys`. Use `lang` for
language-specific rendering policies; equal translated strings do not identify a language. `c()` keeps its
nullish fallback and preserves empty values. Do not import `@/lib/lang` into authored deck code. Existing
artifacts retain their baked SDK; use this property only after the worker release and artifact rebuild.

### A worked example

```tsx
// slides.tsx
"use client";
import * as React from "react";
import { useContent } from "@/lib/content";
import { useRoom } from "@/lib/room";

/** The asking slide: the question, and how many have answered. */
function HowYouShip() {
  const { c } = useContent();
  const room = useRoom();
  const q = room.ask("ship");
  if (!q.spec) return null;
  return (
    <>
      <h1 style={{ fontFamily: "var(--deck-font-display)" }}>{c("ship.title")}</h1>
      <p style={{ fontSize: 24, color: "var(--deck-muted)" }}>{q.question}</p>
      <p dir="ltr" style={{ fontSize: 56, color: "var(--deck-accent)", fontVariantNumeric: "tabular-nums" }}>
        {q.n}
      </p>
      <p style={{ color: "var(--deck-muted)" }}>{c("ship.answerOnYourPhone")}</p>
    </>
  );
}

/** The reveal slide: designed for the bars, and still a slide without them. */
function WhatTheRoomSaid() {
  const { c } = useContent();
  const room = useRoom();
  const q = room.ask("ship");
  const counts = q.counts;
  const total = counts ? Object.values(counts).reduce((a, b) => a + b, 0) : 0;

  // The deck may PROPOSE a jump; the presenter decides. De-duplicated, so an effect is fine.
  React.useEffect(() => {
    if (room.role !== "presenter") return;
    if (!q.closed || q.top !== "main") return;
    room.suggest({ slideIndex: 7, reason: c("ship.suggestReason") });
    return () => room.suggest(null);
  }, [room, c, q.closed, q.top]);

  if (!q.spec) return null;
  return (
    <>
      <h1 style={{ fontFamily: "var(--deck-font-display)" }}>{c("ship.resultsTitle")}</h1>
      {q.sample ? <span style={{ fontSize: 13, color: "var(--deck-muted)" }}>{c("ship.sample")}</span> : null}
      {q.n === 0 || !counts ? (
        // The EMPTY state — an export, a reader, a room where nobody has answered yet, and
        // (with a floor declared) a room that has not reached it. `q.n === 0` comes FIRST:
        // with no floor a revealed round hands you an aggregate of zeroes, not a null one.
        <p style={{ fontSize: 22, color: "var(--deck-muted)" }}>
          {q.n > 0 ? c("ship.fewAnswers") : c("ship.noRoom")}
        </p>
      ) : (
        <div style={{ display: "grid", gap: 12 }}>
          {q.options.map((key) => {
            const count = counts[key] ?? 0;
            return (
              <div key={key} style={{ display: "flex", alignItems: "center", gap: 14 }}>
                <span style={{ minWidth: 200, textAlign: "start" }}>{q.label(key)}</span>
                <span
                  aria-hidden="true"
                  style={{
                    height: 14, borderRadius: 7,
                    width: total ? `max(2%, ${((count / total) * 100).toFixed(1)}%)` : 0,
                    background: key === q.top ? "var(--deck-accent)" : "var(--deck-surface)",
                    transition: "width .4s ease",
                  }}
                />
                <span dir="ltr" style={{ fontVariantNumeric: "tabular-nums" }}>{count}</span>
              </div>
            );
          })}
        </div>
      )}
      {/* Per-viewer, rendered locally from this device's own answer — nothing is fetched. */}
      {typeof q.mine === "string" ? (
        <p style={{ color: "var(--deck-muted)" }}>{c("ship.youSaid")} {q.label(q.mine)}</p>
      ) : null}
    </>
  );
}
```

```ts
// index.ts — the slides that ask and reveal
export const slides: SlideSpec[] = [
  …,
  { Component: HowYouShip,     navTitle: navTitles[3], notes: notes[3], asks: ["ship"] },
  { Component: WhatTheRoomSaid, navTitle: navTitles[4], notes: notes[4], reveals: ["ship"] },
  …,
];
```

### Three shapes

- **Same slide.** One slide `asks` and `reveals` the same id: the question is on screen,
  then the presenter reveals and the slide flips to the chart. Cheapest; good for a poll
  you don't want to spend two beats on.
- **Ask, then reveal.** The shape above, and the default: an asking slide the presenter
  talks over while answers arrive, then a reveal slide designed for the aggregate — which
  is also where a quiz puts the explanation.
- **Ask early, reveal late.** Ask on slide 2, reveal on slide 20. An ask's state is ROOM
  state keyed by the id, not slide state, so any slide can read any ask. Reopening a closed
  ask starts round n+1 and keeps the earlier one: `room.ask("mood", { round: 1 })` against
  the current `room.ask("mood")` is a before-and-after on one slide. An unknown round reads
  as idle, so design that half the same way you design the empty state.

### Reveal and the floor

**The presenter reveals.** The console always sees the counts; the stage and the phones see
the question only until the presenter presses Reveal or the room reaches a slide that
`reveals` the ask. `reveal: "live"` shows the room the aggregate while the ask is still
open — use it for LOW-STAKES asks (a mood check, a warm-up), because an early answer visibly
pulls later ones, and keep it off anything you would call a result.

**`floor` is the anonymity promise, and by default there is none.** The room-facing rule is
`revealed && n >= floor`, and `floor` defaults to **0** — so unless you say otherwise, the
aggregate reaches the room the moment it is revealed, even if one person answered. That is
the right default for most decks: a five-person team, a demo, a room where half the phones
are flat all used to sit staring at a placeholder while the stage drew the result.

**Say what it costs.** A floor of 0 or 1 shows a single person's answer as the room's
answer. **Do not promise anonymity in the deck's copy unless you set a floor that backs it
up** — "nobody can be read out of a number" is a sentence the platform has to be keeping,
not one the slide can simply say. If your deck makes that promise out loud, declare the
number:

```ts
mood: {
  kind: "scale", min: 1, max: 5,
  question: { en: "How is the team holding up?" },
  floor: 3,   // nothing reaches the room until three people have answered
},
```

The `workshop` template is the worked example: all five of its asks declare `floor: 3`,
because its own copy says "the wall stays blank until three people have answered". Adapt
that template and you inherit the promise — keep the floors or rewrite the copy.

**Branch your empty state on `q.n`, not on a null aggregate.** With no floor, a revealed
round carries its aggregate from the first frame — and at `n === 0` that aggregate is all
zeros. `counts` is a real object of zeroes, `mean` is 0, the histogram is flat. It is honest
(`q.n` says 0 too) but it is not "nobody answered", and a slide that tests only `if (!counts)`
will draw a chart of nothing. Test `q.n === 0` first.

`floor` travels to every surface, so a slide can print it: `q.floor` is the real number,
0 when none is declared. A reveal slide with a floor therefore has three states to design,
not one: `n === 0` (nobody yet — or nobody at all, which is the export), `0 < n < floor`
(answers are arriving, no chart yet — here the aggregate really is absent), and the
aggregate. Say the honest thing in each; "3 answers needed, 2 so far" is a fine second
state, an empty chart frame is not. With no floor there are two, and the middle one
disappears.

Presenter surfaces are never floored: the console sees the counts as they arrive, whatever
`floor` says, and the talk record afterwards is complete for the deck's owner. The floor is
about what goes on a wall in front of a room, not about hiding a room's answers from the
person who ran it.

### Craft

- **A reveal slide is designed for the AGGREGATE, and must still read as a finished slide
  with none.** This is the rule this whole feature turns on. The studio, every reader of the
  published deck, and every PDF and PPTX export see the empty state — they never see sample
  data. Only the studio preview and the presenter's rehearsal synthesize numbers, and those
  are flagged `q.sample` for you to label.
- **Draw the results with the deck's own components and tokens.** There is no chart library
  on the import allowlist and you do not need one: bars are divs with a width, a scale
  histogram is a row of bars, a Borda ranking is an ordered list. Same type scale, same
  accent, same motion as the rest of the deck — a results slide that looks like a widget is
  the failure mode.
- **Enumerate options with `q.options`**, and label them with `q.label(key)`. Never
  `Object.keys(counts)`, never a hand-written array of keys beside the manifest's.
- **The deck may only SUGGEST a jump.** No deck code navigates the room; `room.suggest()`
  offers the presenter a button and stops there.
- **A per-viewer branch is local rendering from `q.mine`** — the quiz slide that tells one
  phone why its answer was wrong while its neighbour reads something else. Nothing about the
  person leaves the phone, and there is no room state involved.
- **Ask sparingly.** Two or three asks in a session is participation; one every slide is a
  survey. Each ask costs the room its attention and the presenter a beat.
- Write the speaker `notes` on an asking slide for what the presenter DOES: the ask opens on
  arrival, the console's ask panel holds Open / Close / Reveal / Reopen and the live counts,
  and a suggested jump shows up there as a button.

### Not there yet

Stage 1 is the five kinds above and nothing else. There is **no free-text, no image, no
point-on-a-picture and no reaction ask**; no moderation queue or show-on-screen; no
scoreboards or any other presenter-writable room state; no `room.history()` across talks;
no crosstabs between two asks; no roam window that lets phones navigate on their own. Don't
invent an API for any of them — a deck that imports something that isn't there fails the
gate, and a deck that fakes it lies to a room.

One operational note: a deck that declares `asks` needs the platform's current compiler.
Nothing for you to do — but if a submit comes back complaining about `asks` or about
`@/lib/room`, that deployment has not been updated yet, and the fix is the operator's, not
a change to your deck.

## What makes a compelling deck

(Craft, not mechanics — the gate can't check these, the audience will.)

- One idea per slide. The headline is an **assertion** ("Costs fell 90% in a decade"),
  not a topic label ("Cost trends"). Someone skimming only the headlines should get
  the whole argument.
- Shape a narrative arc: hook → tension → evidence → resolution → one clear ask or
  takeaway. A talk's outline usually sustains 8–14 beats — that band is a property
  of the argument, not of any template; when in doubt cut a beat rather than crowd
  a slide.
- Give the deck ONE signature moment — a big number, a striking chart, a staged
  reveal — the thing the audience will remember.
- Builds (`steps`) pace an argument, they don't decorate it: 2–4 steps on the few
  slides that earn one, none elsewhere.
- The slide carries what the audience must READ; the notes carry what the presenter
  SAYS. If a slide needs a paragraph, it is two slides.
- Write speaker notes as a TALK TRACK SOMEONE ELSE CAN DELIVER: anyone the owner shares the deck
  with as a Presenter (or Can fork / Co-editor) reads them in the console, live. Full sentences
  or crisp beats, the point of the slide, the transition — never private asides, never "remind
  me to…", nothing the owner would not hand a colleague.
- Numbers beat adjectives ("3× faster", not "blazing fast"), and any number that would
  raise an eyebrow names its source on the slide.
- Restraint reads as confidence: keep the template's type scale and palette, don't
  restyle mid-deck. Decks fail from clutter, not from plainness.

## Motion is the medium

This is why cue beats a static slide export — use it, with intent:

- **Diagrams are code, not images.** Build them from SVG/divs with theme tokens, so
  they stay crisp, restyle with the theme, and can assemble step-by-step. A pipeline
  that appears edge by edge as you speak explains more than a finished picture.
- **Charts draw themselves.** Animate SVG paths with `pathLength` 0→1, grow bars to
  their height, morph one shape into the next. Watching the line fall IS reading the
  data.
- **A hidden stroke must leave no dash boundary on the path.** Phones, cast screens
  and Safari render in WebKit, which draws a zero-length round-capped dash as a dot
  where Chromium draws nothing. A hand-rolled `stroke-dasharray` reveal on a
  `pathLength={1}` path therefore uses a gap longer than the path — `"1 2"`, offset
  1→0 — never `"1 1"`, whose next dash begins exactly at the path's end and shows up as
  fragments before the stroke animates (the 2026-10-13 守望 calligraphy lesson). A
  motion `pathLength: 0` start with round caps leaves the same dot at the path's start
  in every engine: keep the path at `opacity: 0` until its reveal begins, or use butt
  caps. Check anything stroke-animated on an iPhone, not only in Chrome.
- **Big numbers count up.** Gate a `<Counter to={N}/>` (`@/lib/primitives`) behind a
  step so the statistic lands as an event, not a caption.
- **Entrances in the deck's voice.** In a library deck use the identity's roles —
  `M.Reveal`/`M.RevealItem` for staggered groups, `M.StepIn` for step builds,
  `M.EnterText` for display lines; the platform `Reveal`/`RevealItem`/`Step`
  (`@/lib/primitives`) are voice-neutral fallbacks for decks without an identity.
  `Typewriter` only for terminal/console moods.
- **Tie motion to steps, not timers** — the presenter's arrow key is the clock. The
  only things moving without input are ambient loops (an animated theme `Backdrop`, a
  slow orbit), and they stay far quieter than the content.
- **Transitions:** one deck-wide `TransitionSpec` sets the rhythm; per-slide overrides
  mark act boundaries (a hard cut into the big reveal), not every slide.
- **Every animation earns its place** by revealing structure, directing the eye, or
  landing a number. Content motion: ~0.3–0.8s, ease-out. If it only decorates, cut it
  — one dead-still slide after heavy motion is itself a dramatic beat.

The eleven deck templates are worked examples of all of this: **signal** (editorial launch
keynote), **evidence** (dark numbers-forward readout), **atlas** (light research
briefing), **orbit** (deep-space telemetry briefing), **aurora** (atmospheric gradient
creative), **harbor** (engineering-blueprint history), **fab** (precision silicon
dossier), **meter** (phosphor-terminal readout), **pulse** (warm humanist demographics),
**mint** (engraved-currency ledger story), **workshop** (a hands-on working session that
asks the room). By technique: orbit/atlas/meter/mint for
self-drawing charts and counters, pulse for a morphing chart, harbor/fab for staged
diagrams, aurora for ambient backdrops, signal/evidence for staged reveals and
before/after builds, workshop for asks — every kind, both rounds, and every empty state
drawn (see *Asking the room*). Pull one and read `slides.tsx` before inventing your own idiom.
- **motion stays the default.** It is the right tool for UI state, entrances, layout
  and anything tied to a step. Reach for **gsap** when you need a real timeline —
  sequenced or overlapping choreography you scrub, reverse or restart as one unit — or
  a plugin motion has no answer for (`gsap/Flip` for layout morphs, `gsap/MotionPathPlugin`
  for path-following, `gsap/DrawSVGPlugin` for stroke draw-on). Drive the timeline from
  the step, don't autoplay it.
- **three is for genuine 3D**, never decoration: a rotating product, a spatial data
  landscape, a scene the argument actually needs. One hero moment per deck, not a
  background. Bundle cost is per deck (only what you import ships), and three adds
  roughly 150 KB gzipped to the deck that uses it — spend it once, deliberately.
- **Both still answer to the presenter.** gsap tweens and three render loops tie to
  steps, not wall-clock timers, and must be cleaned up on unmount (`tween.kill()`,
  `renderer.dispose()`, cancel the RAF) — a deck is navigated back and forth all night.

## Importing a legacy deck (PowerPoint, Keynote, PDF)

When the user already has the deck as a `.pptx`, `.key` or `.pdf`, **import first, then edit** — never
re-author it from a screenshot or a pasted outline. The import is deterministic (no model, no
generation spent, the same file gives the same deck): every slide becomes a normal cue deck with
live text, its pictures and video go to the deck's asset lane, and the project faces the same gate
as any submit. The new deck is born gated.

- The connector cannot send a local file: ask the user to import it in the studio (`/new` →
  **Import a deck**: two switches there start every video with its slide or import them muted, and a
  select sets the deck's language), then work on the deck it creates (`cue_list_decks` finds it).
  A new version of the same file goes INTO the deck from its page's import panel (**Import a new
  version…**): same link, edits, notes and translations kept; the result is a draft. Its "Keep my cue edits on
  slides the file also changed" checkbox is the studio's `onConflict: "keep-cue"`. **Run the import again**
  there re-imports the package cue kept (no file needed) with today's importer and the options chosen — the way
  to apply an importer fix or new options.
- **Editing an imported deck's source** (D418): keep each change INSIDE its slide's unit — the
  `// cue:slide-begin <key>` … `// cue:slide-end <key>` block, that slide's `s<key>.*` strings in the
  source-language `content-<lang>.ts`, and its row in `media-slides.json` (adding or removing a picture:
  set that row to the asset ids the block names); its translations in the other `content-<lang>.ts` travel
  with it. A later re-import of the file keeps those edits where the file did not change that slide (its
  content, master, theme or the import options), and lists the slides it did change in
  `report.reimport.replaced`. An edit outside the units (another file, the spec list, nav titles, presenter
  notes, a new slide) stops the re-import (`409 deck_edited_since_import`).
- The succeeded job's `result.import` is `{ deckId, slides, hidden, report }`, and `cue_job` leads
  it with `slidesToCheck` ("Slides to check: 3, 9–12") and `reviewSummary` (one line per finding
  code). **Read `report.review` first**: the import's self-check, `[{ slide, code, what }]` with
  1-based slides (`reviewMore` counts findings past its bound). Codes: `text-outside-frame` (margins
  wider than the box, the text draws outside it), `number-out-of-range`, `off-stage`,
  `invisible-paint` (a gradient outline or text came out transparent), `media-missing` (a picture,
  video or audio not placed — refused, failed or still processing — or a picture drawn as a labelled
  placeholder, counted in `report.media.placeholders`), `media-inert` (a video or audio that cannot
  start: no step, no start with its slide, no controls), `not-rendered` / `slide-degraded` (over 20% of a slide not drawn), `font-script` (CJK
  text no shipped face covers), `group-scale`. `report.coverage` is `{ slides, layers,
  layersRendered, chars, charsEmitted }` — how much of the source the deck carries. Then the fonts
  (`report.fonts[]`: `family`, `fallback`, `usedOnSlides`, `embedded`, `action`, `hint`), every
  element the extractor drew approximately or left out (`report.unsupported[]`: one line per kind,
  with its slides), warnings and `report.info` notes — every slide number in the report is 1-based.
  Tell the user what it says (the slides to check first) before you change anything; fix a listed
  slide with them rather than assuming the import is right.
- **Media still processing heals by itself** (D417): a video still processing when the import finished
  is counted in `report.media.pending` (its slides in `report.media.pendingSlides`, "still processing:
  will be placed automatically"); its slide shows a `sha256:<hex>` placeholder ref. When it is ready
  cue puts the real `assets/<id>.<ext>` ref on the slide in the DRAFT by itself (a revision
  `via: "media-heal"`; `report.media.healed` lists what was placed) — do not place it by hand and do
  not re-import for it; pull again before editing. Live never changes: a `cue_publish` while files are
  still processing answers `mediaPending` (`{ count, slides, message }`) — tell the user, and publish
  again once they are placed when the user wants live to show them.
- **Media formats the import takes:** pictures in PNG, JPEG, WebP, GIF, BMP, TIFF, HEIC (where the
  server can decode it), SVG (drawn as a picture when it is self-contained, else its PNG fallback) and
  EMF / WMF (the bitmap inside them; a vector-only one becomes a labelled placeholder box with a report
  line naming its slide — redraw it, or ask the user for the picture); video in MP4, MOV, M4V, AVI,
  WMV, MPEG, WebM and MKV (converted to MP4); audio in MP3, M4A, AAC and WAV, also a WAV or MP3
  inside an embedded package object, wrapped as an MP4 under the sound's icon picture.
- **Act on the report with the user.** A font with `action: "upload"` has no library face (the
  stand-in only matches its class): ask the user whether they have the font file AND a licence that
  allows web use.
  If so, ask them to upload it to their account's font library (event or team plan, the licence
  attestation), then `cue_submit` the draft unchanged so the build resolves the family from the
  library.
  `action: "accept"` is a metric-compatible stand-in (the layout holds; uploading is optional);
  `"none"` needs nothing. For charts, tables or other elements listed as approximated, offer to
  redraw them in cue (a chart as inline SVG or a primitive, a table as real rows) and do it only
  with the user's yes. Never silently re-author an imported deck.
- **The imported deck is a normal deck.** Content keys are `s<slide>.<layer>`, slides live in
  `slides-NN.tsx` modules, text is `<Text mode="live">`, positions are stage pixels on the
  source's stage (`stage` in the manifest).
  Edit it the usual way: `cue_grep` / `cue_read_files` to locate, `cue_edit_file` to change,
  `cue_submit` with the deckId.
  Prefer small edits to the imported source over re-authoring it.
- **Translations** are ordinary `content-<lang>.ts` modules keyed like `content.ts` — add a
  language by translating those keys and declaring it in `manifest.languages`.
- **"Rebuild it in cue's voice"** is a separate, explicit request: a new deck authored with the
  imported one as its source material — never a silent replacement of the import.

## Imported and event decks: stage, hidden slides, builds, media, exact text

Everything here is optional; a deck that uses none of it behaves exactly as before.

- **Stage size.** `stage: { w: 1920, h: 1080 }` on the manifest lays the slides out on that
  canvas (default 1280×720) and scales it to every screen; coordinates in the source are
  stage pixels. A declared stage is a pixel canvas: square corners, no shadow.
- **Hidden slides.** `hidden: true` on a SlideSpec keeps the slide in the deck (indices stay
  absolute) but next/prev, the rail and the counter skip it; the presenter console lists it
  under **Reserve** and one tap jumps there. `id: "s12"` gives a slide a stable id.
  The owner can also hide or show a slide in the studio's Slides tab (the eye on a row, or
  `H`) where the deployment's compiler supports it. That choice is a studio override keyed by
  the slide's SOURCE index, laid over your `hidden` declarations: the studio can show a slide
  you hid and hide one you did not. You cannot submit it; cue re-applies it on every submit.
  If you change a slide's `hidden` in source, your change wins for that slide. At least one
  slide must stay visible.
- **Hard cuts.** `transition: "none"` (deck-wide or per slide) switches slides on the next
  frame with no animation.
- **Timed builds.** `<Build order={0} trigger="afterPrevious" kind="dissolve"
  duration={400} delay={100}>…</Build>` wraps a layer (it fills its parent frame, so scale,
  move and wipe act about the layer). Triggers: `onClick` (starts a new step), `afterPrevious`,
  `withPrevious`; times are milliseconds; kinds `dissolve`, `appear`, `moveIn`, `wipe`,
  `keyboard`, `blur`, `blink`, `scale` (`options={{ from: 0.5, to: 1 }}`), `startMovie`
  (plays the `<Video>` it wraps); `direction` for moveIn/wipe/keyboard, `easing` any CSS
  easing, `out` for a build-out. More `options`: wipe `feather` (soft edge in px, 0 = hard),
  blur `blur` (start radius px), scale `origin` (`{ x, y }` px from the layer's top-left, or
  `{ x: 0.5, y: 0.5, units: "fraction" }`); blink — a smooth raised cosine (Keynote's measured
  curve) — `count` (default 3), or `period` (ms per blink; Keynote's is 361) to derive the count from
  the duration, `minOpacity` (the dip, default 0), `action: true` for a layer already on screen; keyboard on an
  exact `<Text>`: `perChar` (ms per glyph, or each glyph's time), `cursor` (a typing bar), and `caret: { keys }`
  — one time per keystroke, spaces and line breaks included (`null` = not typed) — for Keynote's measured cursor (blinks in, solid
  while typing, blinks out at the build's end; positions come from the generated outline, so it is for imported
  decks; unusable input falls back to the plain bar and shows as `keyboardCaret:*` in `buildFallbacks`); `delivery="byWord"` / `"byParagraph"` staggers a
  text layer's words / paragraphs (`options={{ stagger: 120 }}` ms apart). On LIVE text (every language but the
  source's, `mode="live"`, a patched layer) a keyboard types grapheme by grapheme (word by word for Indic, Thai and
  similar scripts; measured times kept by position, the cursor measured from the laid-out text) and words are the
  language's dictionary words; right-to-left text and layers over 20 000 units keep a stepped clip reveal
  (`textBuild:script` / `textBuild:units` in `buildFallbacks`); keyboard `count` is ignored on live text. Set the slide's `steps: buildSteps(builds)` (exported from
  `@/lib/primitives`) — one step per `onClick` chunk plus the entry step. Going back lands on
  end states; `prefers-reduced-motion` skips to them. `<Step>`/`useStep()` are unchanged —
  keep them for hand-authored step reveals.
- **Media.** `<Image asset fit position alt />` shows the inline placeholder, then the
  viewer's tier (lite/phone/desktop; screens can pin stage). A studio crop of that image (an
  `imageCrops` override, see Studio overrides) replaces `position` and may zoom it up to 4×. `<Video asset playOnStep loop
  muted playsAcrossSlides controls poster start end volume />`: `poster` (an image asset) replaces the
  media ladder's own poster frame; `start` / `end` (seconds) trim playback (a `loop` wraps at
  `end`); `playsAcrossSlides` keeps it playing through a slide
  change only when the next slide mounts the same asset; `controls` shows the native controls,
  and in a live room the presenter's play/pause/seek with them is mirrored on every follower;
  `volume` 0..1. Neither has an intrinsic height: size them (`style` width/height, or
  `position: "absolute", inset: 0` in a sized, positioned parent) or they draw nothing. Sound:
  `muted={false}` plays with sound only where the browser allows it; a play the autoplay policy
  refuses (a step that reached the deck without a click in it) is retried MUTED, the video shows
  a small speaker button that turns the sound on, and the engine's `deck:media-state` carries
  `mutedByPolicy: true`.
  Give the player `media-slides.json` so it prefetches ahead of the presenter.
- **Motion background.** `<MotionBackground stops={["#f75a46", "#3e39df"]} angle={90} scale={2}
  speed={50} still="assets/<id>.jpg" />` fills its parent with a gradient that repeats first → last →
  first stop and scrolls toward the first stop's edge (Keynote's moving backgrounds; the importer
  spreads the IR's `background.motion` into it). `angle`: degrees counter-clockwise on the slide
  stretched to a square (90: first stop at the bottom; 135: toward the top-left corner); `scale`: a
  cycle spans 5 / scale slide heights; `speed`: a cycle lasts 800000 / speed ms (16 s at 50), or set
  `period` (ms); `phase` (0..1) shifts it; `reverse` flips the drift. The clock runs on across slides
  (`clockMode="presentation"`); `clockMode="slide"` restarts it at `offset` ms on each entry. It costs
  one compositor animation; under reduced motion and in PDF / image renders it shows `still` (or its
  clock-0 frame), and a `deck:seek` sample draws the frame at `offset` + the seek's `clock` (in a
  render context only an explicit `clock` does; a seek without one keeps the still).
- **Exact text** (imported decks): `<Text obj="t12" k="s3.t12" mode="exact" box={[x, y, w, h]} />`
  renders the content key's live text and, in the source language, the generated glyph
  outline `o-<obj>` from `outlines/<slide>.svg` exactly over it. Other languages and edited
  copy show the live text. The import pipeline generates these; do not hand-write outlines.

## Formulas

Maths is typeset by KaTeX inside the deck, from `@/lib/formula` — decks only for now:
documents cannot export formulas to PDF yet.

```tsx
import { Formula, FormulaText } from "@/lib/formula";

<Formula tex="e^{i\pi} + 1 = 0" />              {/* inline, in the run of a sentence */}
<Formula tex="\int_0^\infty e^{-x^2}dx" display /> {/* its own centred block */}
<FormulaText text={c.c("law")} />                 {/* a content string with $…$ in it */}
```

- **`Formula`** takes `tex`, `display`, `className`, `style`, `macros`
  (`{"\\R": "\\mathbb{R}"}`) and `label` (an assistive-tech name when the spoken form
  should differ from the source). Formulas draw in `currentColor`, so they take the text
  colour of whatever contains them, and scale with `font-size`.
- **`FormulaText`** is for multi-language copy: put `$a^2 + b^2 = c^2$` (inline) or
  `$$…$$` (display) straight into a `content.ts` string and render the whole string with
  it. `\$` is a literal dollar; an unpaired delimiter renders as written.
- **Build a formula step by step** with `\htmlClass`: tag a term
  (`\htmlClass{focus}{mc^2}`), then key CSS off the step, exactly as with any other
  element — `.focus { opacity: .35 }` until `step >= 1`. `\htmlId`, `\htmlData` and
  `\htmlStyle` work too; `\href`, `\url` and `\includegraphics` are refused (the
  artifact CSP blocks external hosts anyway).
- **Supported commands:** https://katex.org/docs/support_table. A formula KaTeX cannot
  parse renders as its own source in red — it never fails the build — so check the
  preview rather than trusting a build that passed.
- **Cost:** the glyph fonts (~250 KB) ship only in a deck that actually imports
  `@/lib/formula`. A deck without formulas is unchanged.

## Codes in decks: present, join, save and live

A deck can draw the QR codes the screen around it already knows — the code that lets a visitor
**present** on an idle screen, the **join** code of the live room, the **save**-to-shelf code and
the deck's permanent **live** link — as part of its own design, from `@/lib/primitives`. The deck never makes a code: the hosting
surface hands it the URLs (a full snapshot, re-sent on every change), and `<Qr>` draws them as
inline SVG, so they work under the artifact CSP. For event screens: one idle deck per event, its
booth slide carrying the code.

```tsx
import { Qr, useLinks } from "@/lib/primitives";

<Qr kind="present" size={420} caption fallback={<Qr kind="join" size={420} caption />} />
<Qr kind="save" size={160} caption fallback={<p>{c("save.off")}</p>} />
<Qr kind="join" size={300} caption fallback={<Qr kind="live" size={300} caption />} />
const { present, join, save, live } = useLinks(); // each { url, caption?, expiresAt? } or undefined
```

| Host | `present` | `join` | `save` | `live` |
|---|---|---|---|---|
| Room screen, idle (its idle deck) | the take-over code, rotated every 30 s — only on an open screen whose idle deck belongs to the screen's owner | — | when the deck is collectable | always |
| Room screen bound to a talk, stage display, presenter console | — | when the room admits an audience (not a rehearsal) | when collectable | when the room admits an audience (not a rehearsal) |
| Audience phone (`/live`) | — | — | when the viewer could save it | always |
| Self-paced viewer (`/view`) | — | — | when collectable | always |
| Export (PDF/PPTX) | — | — | — | always (a published deck) |
| Embed | — | — | — | — |

- **Props:** `kind` (`"present" | "join" | "save" | "live"`), `size` (the code's edge in stage px,
  default 240), `caption`, `fallback`, `className`, `style`, `testId`.
- **Caption rule:** `caption` shows the HOST's wording under the code ("Scan to present on
  Lobby", "Scan to join", "Save to your shelf", "Scan to join the talk") in the deck's current language — don't write your
  own copy for it; put your own heading or call to action beside the code if you want one.
- **Reserve the space.** A code that is not available here, now renders `fallback` — nothing by
  default. Lay the slide out so it reads well both ways: give the code a fixed box, or a fallback
  that fills it. The first pattern above is the booth slide: "scan to present here" while the
  screen is idle, and the room's join code once a visitor has taken it over, with no logic of yours.
- **"Scan to join" on a slide: prefer `join`, fall back to `live`** (the third line above). During
  a talk the room's code draws — it admits on every tier, the room key included; outside a talk
  (`/view`, an idle screen) the deck's permanent code draws instead. This is the DEFAULT for any
  slide that invites the audience in. Use `kind="live"` alone only when the slide must look the
  same everywhere (it will be photographed or printed).
- **`present` is the owner's own deck's only.** It appears only on an OPEN screen whose idle
  deck belongs to the screen's owner; everywhere else the fallback renders. The reason is the
  rule every code follows: **anything a host pushes into a deck frame is disclosed to that
  deck's author** — a sandboxed frame can always navigate itself, so `connect-src 'none'` does not
  keep a value in — and the take-over code must never reach someone else's deck.
- **`live` is the permanent live link** (`/live/<deckId>`, the same `permanentLiveUrl` the
  publish result reports): scanned while any talk of the deck runs it lands in that room, otherwise
  on the waiting page. It never expires and never carries a key on any tier — the server decides
  admission (a `link` deck's scan meets the gate unless the owner set `cue_share {liveLink: true}`).
  It is the one code a PDF / PPTX export draws (in the export's language), so a printed handout
  carries it. It is absent in a rehearsal and in the embed — that gets no codes at all — so give
  it a `fallback` (or print `permanentLiveUrl` yourself) if the slide must read without it.
- **Expiry:** a `present` code that lapses with no fresh one greys out under "connecting…"
  rather than staying scannable-but-dead.
- **Where codes are absent:** the studio preview draws a placeholder tile ("present code" /
  "join code" / "save code" / "live code") — no real code exists while you author; a published viewer that
  has none renders `fallback`. An artifact built before this primitive shipped never receives
  codes — **only decks rebuilt (re-submitted) on the current builder have them**, so re-submit an
  older deck before relying on it.
- **Link-tier decks:** during a live talk the join and save codes carry a **room key** — a
  secret link minted for that one room, which admits only that room's audience and stops working
  the moment the talk ends (or the deck leaves the link tier, is unpublished, or its links are
  rotated). The room bounds the door, not copies: while it is live a key holder can download the
  deck (PDF/PPTX) like any link holder. Nothing to do in your deck; just don't print a live code
  as a lasting invitation. An
  idle screen has no room, so its save code for a link-tier deck carries no key and scanners meet
  the link gate.
- Corner QR cards the presenter toggles (join, collect) keep working independently; a deck that
  carries codes does not turn them off.
- The worked example is cue's own fixture deck (`tests/fixtures/qr-codes-deck` in the cue
  repository): exactly the booth slide above, a reserved save code and a `useLinks()` list of
  the URLs as text.

## Exporting a deck: PDF and PowerPoint

A published deck downloads as a **print-ready PDF** — `GET <CUE_URL>/api/decks/:id/pdf` —
every slide at its final step, one page each. Multilingual decks render one file per
declared language: the default language is the bare `/pdf`, the others carry `?lang=<code>`
(an unknown language 404s). The download honors the deck's access tier exactly like the
viewer — a `public` deck is zero-friction, a `gated` deck needs the owner or a named person's
signed-in account, and a `link` deck needs its `?k=<token>` share credential. A PDF may be
rendered on its first request: then the route answers `202 {"status":"preparing","pollAfterMs":…}`
(or `503` while a fresh publish is still rendering) — wait `pollAfterMs` and ask again until it
redirects to the file. Slides the published version marks hidden are left out of the PDF and of
the exact PowerPoint (where the deployment's renderer supports it); the owner or a co-author gets
every slide with `?includeHidden=1` (for anyone else the parameter is ignored), served through a
signed per-download URL.

Some deployments also offer an **exact-look PowerPoint export** — `GET <CUE_URL>/api/decks/:id/pptx`,
with the same `?lang=` behavior, the same access model and the same first-request answer
(`202 {"status":"preparing","lang":…,"pollAfterMs":…}` until the file lands, then the redirect).
It is a **faithful rendered snapshot**: one full-bleed image per slide (a high-quality JPEG,
2560 px wide) plus the speaker notes — NOT editable text, shapes, or charts (each slide is a
picture, not a re-layoutable object). Documents have no PPTX export.

Both exports render **on demand**, one language at a time, the first time a version's file is
requested — a cold render can take ~1–2 minutes (longer for a very large deck) — then the
artifact CDN serves every later download instantly.
PPTX export, like secret links, is enabled per deck owner, and an operator can pause it for everyone: where it isn't
enabled the `/pptx` route 404s, so treat it as available only where it responds.

## Documents: sent, not performed

cue also builds DOCUMENTS — a second artifact kind alongside decks. A deck is performed
(a stage, transitions, a slide/step clock); a document is SENT (paper, sections, print
geometry). Same envelope: the gate, versioned hosting, access tiers, share links, feedback
notes, and the content-string + multi-language model all carry over unchanged. Reach for a
document when the reader sets the pace and the piece may become a PDF — reports, investor
letters, proposals, statements of work, flyers, posters.

### Choosing document vs deck

- **Deck** when it is spoken to a room, paced by an arrow key, and lives on a screen.
- **Document** when it is read alone, paced by the reader, and may be printed or PDF'd.
  Prose in paragraphs, a figure that keeps its caption, a pricing table, a signature block,
  a poster headline — all document, not slide.

### The manifest

A document default-exports a `DocumentManifest` (from `@/lib/types`), not a
`PresentationManifest`. Same `title` / `languages` / `defaultLang` / `theme` / `content` as a
deck, plus:

- `paper`: `{ size: "A5"|"A4"|"A3"|"A2"|"Letter", orientation?, bleed?, marksMargin? }`.
- `mode`: `"flow"` or `"fixed"` (below).
- `sections` (flow) OR `pages` (fixed) — exactly one. Each entry is
  `{ Component, navTitle?, notes? }`; a flow section also carries `id` (its stable anchor)
  and optional `breakBefore: "page"`.
- `running?`: print header/footer strings + `pageNumbers` — manifest DATA (`mode: "static"`),
  rendered natively in the page margins, not React components.
- `toc?`: the reading-view rail; on by default once ≥ 3 sections carry a `navTitle`.

Theme tokens are `--doc-*` (`--doc-paper`, `--doc-ink`, `--doc-accent`, `--doc-font-display`,
…) applied to `.doc-root` — the paper-side analog of a deck's `--deck-*`.

### Flow vs fixed

- **flow** — reports and proposals: sections stack into one continuous, reflowing article
  (~65ch measure, adapts to a phone) and auto-paginate to exact paper size as a print-ready
  PDF. The default for anything text-driven.
- **fixed** — flyers and posters: each page is an exact paper-size canvas scaled to the
  viewport; overset (anything past the trim) is a BUG the gate blocks. For single-composition
  art, not paginated reading.

### Typographic craft (the documents analog of motion — the gate can't check these)

- **Hold the measure.** ~65ch is the readable line; the flow article already caps there.
  Don't fight it wider — a full-bleed wall of text is unread.
- **Real hierarchy, not bold-everything.** One display face, a clear step from heading to
  body, whitespace doing the separating. Restraint reads as confidence, same as in a deck.
- **Break well and let type breathe.** Body copy runs `hyphens: auto` + `text-wrap: pretty`;
  reserve `breakBefore: "page"` for the few sections that open a new part, not every one.
  Wrap charts (with captions) in `Figure` (`@/lib/primitives`) so a page break never slices
  them, and build data tables with `DocTable` so the header repeats across pages.
- **AutoFit for display type.** Poster and flyer headlines go through `<AutoFit>`
  (`@/lib/primitives`) so a longer localization can't overflow the sheet. It fits the largest
  size to the box — but a single unbreakable word is governed by its `max`, so set `max`
  width-safe for the LONGEST unbreakable word in every declared language, not for the
  average line.
- **Running content is for print.** Header/footer strings + page numbers live on
  `manifest.running` and render in the page margins; the reading view itself stays paper-first
  and near-chromeless.
- **Bleed only when printing.** Leave `bleed: 0` unless the piece is headed to a print shop;
  then set `bleed: 3` (mm) and keep every important mark inside the trim.
- **Graphics are code**, exactly as for decks: inline SVG/CSS, a `data:` URI at most. The
  artifact CSP blocks every external host — fonts included — so faces are self-hosted system
  or repo stacks (with a CJK fallback), never a remote font URL.

### Starting a document

The template library is decks only — nothing to pull and adapt here, so a document is
authored from scratch against the manifest and craft rules above.
`cue_create_deck {title, kind: "document"}` opens an empty server-held draft workspace to
write into.
`kind` is set once at creation and is ignored on every later submit, so pick it
deliberately — a deck cannot become a document afterwards. Everything downstream (gate,
versions, access tiers, share links, feedback, languages) is identical to a deck.

### The verified PDF (PDF/UA) — what "qualified" means

A published document's PDF — `GET <CUE_URL>/api/decks/:id/pdf` — is not a browser print. It is
produced by an isolated worker chain (renderer capture → packager → veraPDF verifier) and served
ONLY when it verified as PDF/UA-1 with a receipt; `GET <CUE_URL>/api/decks/:id/render-status`
reports `state: done` when it is ready, `processing` while the worker runs (~30 s), and
`document_pdf_unsupported_qualified_profile` / `document_pdf_verification_failed` when the
document is outside the qualified subset (the document still publishes and reads fine on
screen — only the verified download is unavailable). Documents that verify are called
**qualified**; `cue_templates` marks any template that can produce one with
`qualified: true` — and no shipped template is inside the subset today, so a qualified
document is authored from scratch against these rules. The rules the renderer enforces —
every rejection names one of them:

- **Fixed mode, one language, no bleed needed.** `mode: "fixed"`, `languages: ["en"]` (exactly
  one; Latin, Cyrillic or Greek script), the default screen profile. Flow documents cannot
  qualify (no `.doc-sheet`; the TOC rail is a `<button>`).
- **Each page component returns a fragment of allowed elements directly** (the engine wraps it
  in `.doc-sheet`): only `h1`–`h6`, `p`, `blockquote`, `figure` (one `img` with `alt`, png/jpeg/
  webp `data:` or same-origin), `ul`/`ol` (leaf-text `li`, `list-style: disc inside` /
  `decimal inside`), `table` (`border-collapse: separate; border-spacing: 0`, rectangular, `th`
  with `id` + `scope="col|row"`, `td` with `headers="<th ids>"`, cells leaf text), and empty
  `div.cue-safe-decoration` for bands/rules. No wrapper `div`, `section`, `header`, `span` at
  the top level; inline markup inside a block is `span`/`em`/`strong`/`code`/`a` only, unnested;
  a link is canonical public `https://…` or `#id`, and must fit on one visual line.
- **Opaque colours everywhere** (`#rrggbb` / `rgb()`; no `rgba`, `transparent`, `color-mix`
  with alpha, `opacity`). Backgrounds are a solid colour or a two-stop `linear-gradient` `to
  right` / `to bottom` — no radial or layered gradients. Borders solid, uniform, ≤ 32px,
  `border-radius: 0`. Paint (background/border) only on the sheet, on direct `h*`/`p`/
  `blockquote` children, on `div.cue-safe-decoration`, and on `th`/`td`.
- **None of:** transforms, filters, box/text shadows, outlines, `overflow` other than visible,
  `position`/`z-index`/`order` on anything but the sheet, negative margins, `letter-spacing`,
  `word-spacing`, `text-transform`, `white-space` other than normal, font-feature/variation
  settings, `::before`/`::after`/`::first-letter`/`::first-line` rules ANYWHERE in the
  document's CSS, `text-decoration` (links included: set `text-decoration: none`), hidden or
  `visibility: hidden` content, tabs/newlines/double spaces inside a text node, `<AutoFit>`
  (it is a `div`). A page may not END on a decoration — every `div.cue-safe-decoration` must be
  followed by semantic text on the same page.
- **Fonts and weight.** The PDF text is set in the pinned Noto Sans regardless of the screen
  face, and the theme's `--doc-font-body` / `--doc-font-display` stacks must map to it: end
  them in `sans-serif` (the engine's default body stack ends in `serif` and is rejected as
  "font outside the qualified Noto Sans mapping"); `font-weight` must resolve to 400 or 700,
  `font-style` normal or italic.
- **Geometry.** Everything must lie inside the sheet: inset with the children's margins (the
  page component cannot reach the sheet; a first-child top margin is safe since D143), keep
  `line-height ≥ 1.4`, leave room against the overset gate. Soft-wrapped paragraphs are fine.

No shipped template is inside the subset today, so there is nothing to start from: author the
fixed A4 sheet directly from the rulebook above — page components returning bare
`h*`/`p`/`ul`/`table` fragments, opaque colours only, a `sans-serif`-terminated font stack, no
pseudo-elements anywhere in the CSS.
When `/render-status` reports a failure, the delivery's diagnostics name the exact rule
(`unsupported safe document content: <rule>`); fix precisely that.

### Print honesty

The PDF pipeline outputs **sRGB** — no CMYK, ICC profiles, spot colors, or overprint. Call the
export a **"print-ready PDF"**, never "offset-proof" or "press-ready CMYK." It prints cleanly
on any office or short-run digital printer; a commercial offset run does its own color
conversion at the shop.

## Judgment calls

- If a submit 400s, the error names the exact rule violated — fix precisely that.
- `tsconfig.json` and `overrides.json` are platform-managed, not deck source: the server
  silently DROPS both on submit, so neither ever needs editing.

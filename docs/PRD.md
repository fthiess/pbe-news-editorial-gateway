# PBE News Editorial Gateway ("Gateway") — Product Requirements Document

**Status:** Draft 1, 2026-10-29. Requirements only; *how* each one is met belongs to the engineering design, which comes next. Decisions cited as `D<n>` are in [`DECISIONS.md`](DECISIONS.md), which is authoritative where this document and the log disagree.

---

## 1. Overview

PBE News is the twice-yearly online newsletter of the Phi Beta Epsilon alumni association, published on Ghost(Pro) at pbe400.org. Each issue carries 20–30 articles, most submitted by brothers by email, as Word, Google Docs or PDF files, often with photographs. Forrest, the publisher and editor, does all the editorial work by hand today: he uploads submissions as drafts, sets their metadata, canonicalizes every brother's name and links it to his profile in Book, writes alt text for every image, tracks open "TK" items, and checks each article before publication.

Forrest wants Claude, and Claude Duet in particular, to act as an **assistant editor**. That requires read and write access to the newsletter's content in Ghost. The access must be safe enough to point at the live site, and the credential must be somewhere a Claude session can never read it.

The **Gateway** provides that access. It is a small remote MCP server, run on Cloud Run, that Claude clients call to read articles and pages, look up brothers in Book, and edit drafts (D1, D5). It holds a dedicated Ghost staff credential (D4) that never leaves the server. It enforces the write policy in its own code (D7): **Claude can edit drafts freely, and can't change anything that isn't a draft.** Publishing stays a human act, and so does unpublishing, which is how Forrest hands a published article back to Claude.

The Gateway is plumbing. What the assistant editor *does* with these tools is a separate design (D3).

## 2. Background

### 2.1 Environments

- **Live:** pbe400.org on Ghost(Pro). Real member data. Mixpanel analytics are live. The Gateway touches it only after a deliberate cutover (§9).
- **Staging:** staging.pbe400.org, self-hosted Ghost on Forrest's GCP VM (gladstone). Its member list is fake (`@example.test`) and its bulk email is permanently disabled. It is the Gateway's development and test target.
- **Book** (book.pbe400.org, and Book staging) is the members-only brother directory. It is the system of record for brothers' names, class years and Constitution IDs, and it enforces field visibility by role in server-side projection.

### 2.2 How an issue is put together in Ghost

These are Forrest's conventions. The assistant editor applies them; the Gateway doesn't enforce them (D3), but it must be able to read and write every field they use.

- An article belongs to an issue through an **issue tag**, written year first (for example `#2026-spring`).
- Further tags place an article in a **section**.
- **Slugs** begin with the issue identifier (for example `spring-2026-from-the-editor`).
- Within a section, articles are ordered by **publication date and time**, which Forrest sets by hand to get the order he wants.
- **Bylines:** every author is a Ghost Contributor account, usually a placeholder Forrest creates under a plus-address of his own, because few authors log in to Ghost. Creating those accounts is user management and stays with Forrest. When an author has no account yet, Claude sets itself as the author until Forrest resolves it.
- **Email:** only the lead article, Forrest's "From the Editor", is ever sent by email. Readers reach every other article through its links.
- **TK** ("to come") marks open items. It is the literal string `TK` anywhere in an article, alone or inside a note such as `[TK add image]`. Ghost's editor flags every occurrence.

### 2.3 Ghost constraints

- The Ghost(Pro) plan allows three staff users. Forrest and the backup editor hold two. The third goes to Claude, as an **Editor** (D4). A Contributor would be no use: contributors see only their own drafts.
- Ghost 5 and later store content as **Lexical JSON**, with "cards" for images, galleries, bookmarks, callouts and similar. Converting HTML to Lexical is fine for creating an article (D10). Round-tripping an existing article through HTML would damage its cards, so **edits to existing content must preserve its structure** (FR-10).
- Ghost(Pro) can't be modified, which rules out plugins and custom fields. Editorial status lives outside Ghost (D3).

## 3. Goals and non-goals

### 3.1 Goals

1. **Save Forrest real editing time on the Fall 2026 issue.** The two largest time sinks, name canonicalization with Book profile links and image alt text, must work end to end through the Gateway before editing of that issue begins (D15).
2. **Make the safe thing structural.** No Claude client, however it is prompted, can publish, unpublish, schedule, delete, send email, or change anything that isn't a draft.
3. **Keep the credential out of every session.** The Ghost token is never visible to a model, a chat transcript, a tool result, a log line or the repository.
4. **One interface for every client.** The same operations serve Claude Code, claude.ai, Duet, Routines and, later, the Linter.
5. **Be frugal.** Low operational burden, near-zero cost at this volume, and tool outputs sized for a model's context.

### 3.2 Non-goals

- The assistant editor's behavior: intake judgment, TK sweeps, readiness checks, project management, editorial status (D3). §5 lists these only to test that the Gateway's interface is sufficient.
- Administration of any kind: members, tiers, newsletters, email sends, settings, themes, integrations, staff and contributor accounts, tags as objects (renaming or deleting them).
- Changing any article's status, and deleting anything (D7).
- Parsing source document formats such as Word or PDF (D10).
- Gmail or Google Drive access from the Gateway. Claude has Gmail through its own connector; the Gateway never does (D11).
- The Linter itself, including its rules and house style.

## 4. Users and clients

**Human editors.** Forrest today. The backup editor is expected soon, so the design must admit more than one human without change (D6). Each human signs in with their own Google account, on an allow-list.

**Claude clients** act for a human editor, all through the one Claude Ghost identity (D4, D6):

| Client | Phase | Typical use |
|---|---|---|
| Claude Code (desktop app) | A (MVP) | Interactive editing sessions with Forrest: names, alt text, other edits on request. |
| claude.ai (web) | B | The same, from a browser. |
| Claude Duet | B | Proactive assistant editor: intake from Forrest's email, noticing changes, proposing next steps. |
| Cloud Routines | B | Scheduled or unattended runs, such as an overnight TK list. |

**The PBE News Linter** is a later, programmatic client (D2). It is deterministic code that applies mechanical fixes to drafts. It needs the same read and edit operations, under a non-human identity.

## 5. Driving use cases

None of these is built *into* the Gateway, but each must be possible *through* it. They test the interface.

1. **Name canonicalization (MVP).** Claude reads a draft, finds each mention of a brother, asks the Gateway which brothers match ("James Smyth '84", say), decides which one is meant, and rewrites the mention in canonical form with a link to that brother's Book profile. This needs: read the article with precise addresses, Book lookup, and a structure-preserving "replace this text span with this linked text" edit.
2. **Alt text and captions (MVP).** Claude finds every image in a draft, including the feature image, looks at each one, and writes alt text (and a caption, where one is missing and the context supplies it). This needs: read images as pictures, and set alt text and captions.
3. **Edits on request.** Forrest asks for a change, Claude suggests it, and Forrest says "yes, please make that change". On a draft, Claude makes it. On a published article, Forrest unpublishes it first (D7). The MVP covers text-span replacement and metadata edits; general editing comes later.
4. **Intake (stretch).** Claude reads a submission in Forrest's email, extracts the article as HTML, and creates a draft with its metadata: issue and section tags, slug, author, ordering date, and `[TK image: …]` markers where photos belong (D10, D11).
5. **TK list.** Claude reads all of an issue's drafts and lists every TK. This needs only reads.
6. **Pre-publication readiness checks.** For example: the metadata is complete and matches the issue, there are no TKs, and every image has a caption and alt text. This needs only reads, but the reads must expose every field involved.
7. **Noticing change (stretch).** Duet asks what changed since it last looked, and decides on a next step, such as offering to lint a new draft (D9).

## 6. Scope and phasing

The schedule (§10) leaves about eight working days of build before the Fall 2026 issue's editing starts. The MVP is therefore cut to what the two time sinks need, and the riskiest component, OAuth for claude.ai connectors, comes after the deadline (D15).

**Phase A — MVP, for Fall 2026, Claude Code only:**
- Read posts and pages of every status, with all metadata (FR-1 to FR-4).
- Read images as resized pictures (FR-5).
- Draft edits: replace a text span, with or without a link (FR-10); set alt text and captions (FR-11); set metadata (FR-12).
- The write policy and collision check (FR-20 to FR-26).
- Book lookup (FR-30 to FR-32).
- Logging (FR-40 to FR-42).
- Authentication of allow-listed Google identities (FR-50).
- **Stretch:** create a draft from HTML (FR-13), the change feed (FR-35), author lookup (FR-6).

**Phase B — after the Fall 2026 issue is published:**
- OAuth for claude.ai custom connectors (FR-51), bringing Duet, web and Routines.
- Whatever Phase A stretch goals didn't make it.
- A non-human identity for the Linter (FR-52).

**Later:** general structure-preserving editing (moving, inserting and deleting blocks); image upload and placement (D11); webhooks (D9).

## 7. Functional requirements

Requirement IDs are stable; gaps in the numbering leave room for later additions.

### 7.1 Reading

- **FR-1.** List posts and pages, filterable by status, tag (including internal tags) and `updated_at` range. Each listing entry is a compact summary: id, title, slug, status, tags, authors, `published_at`, `updated_at`, and a count of TKs and images. Listings never include bodies.
- **FR-2.** Get one post or page with **every** metadata field Ghost exposes for it: title, slug, status, visibility, tags (with internal flag), authors (primary first), `published_at`, `updated_at`, feature image and its alt text and caption, custom excerpt, meta and social fields, canonical URL, code injection, template, and featured flag.
- **FR-3.** Get an article's body in a form a model reads efficiently and **can refer back to precisely**. Every block and every text run has a stable address, so a later edit can say exactly what it changes. Every image card's source, alt text and caption are included.
- **FR-4.** Reads work on posts and pages of every status. Reading never changes anything.
- **FR-5.** Fetch an image used in an article as a picture the calling model can see, resized so it costs a predictable, modest amount of context.
- **FR-6.** List the staff and contributor users who can be an article's author (name, slug, role), read-only.

### 7.2 Writing (drafts only)

- **FR-10.** **Replace a text span.** Within one addressed text block, replace a specified span with new text, optionally carrying a link. Formatting outside the span, and every card in the article, stays untouched. This is the name-canonicalization edit.
- **FR-11.** **Set image metadata.** Set the alt text and/or caption of an addressed image card, or of the feature image.
- **FR-12.** **Set article metadata.** Change any of: title, slug, tags (assigning a tag that doesn't exist yet creates it, which is how a new issue tag appears), authors (from FR-6's users), `published_at`, visibility, custom excerpt, meta and social fields, feature-image alt text and caption, and the featured flag. Code injection is excluded (FR-25).
- **FR-13.** *(Stretch.)* **Create a draft from HTML.** Create a new post or page, as a draft, from supplied HTML, using Ghost's own HTML conversion (D10), with any FR-12 metadata set at creation. If no author is given, or the author has no account, the author is the Claude user.
- **FR-14.** Every successful write returns the article's new `updated_at` and enough of the changed region to confirm the edit.

### 7.3 Write policy

These requirements are the Gateway's reason to exist. They are enforced by the Gateway itself, on every write, whatever the caller asks (D7, D8).

- **FR-20.** A write succeeds only if the target is a **draft** at the moment of writing. Published and scheduled articles are read-only, posts and pages alike.
- **FR-21.** No operation sets or changes an article's status. Nothing publishes, schedules, unpublishes or reverts to draft.
- **FR-22.** No operation deletes a post, page, tag, image, user or anything else.
- **FR-23.** No operation sends email or sets any newsletter or email parameter.
- **FR-24.** No operation reaches members, tiers, offers, newsletters, settings, themes, integrations, webhooks, staff or contributor accounts. The Gateway doesn't expose those parts of the Ghost API at all.
- **FR-25.** **No active content.** Content submitted by email is untrusted input, and a submission can contain instructions aimed at the model. So no write may add script, embedded active content, or raw HTML that could run on pbe400.org when the article is later published. Code-injection fields aren't writable, and HTML supplied for FR-13 is sanitized to safe markup before conversion. *(Open question Q6 asks Forrest to confirm this.)*
- **FR-26.** **Collisions.** Every write carries the `updated_at` that its change was based on. If the article changed since then, the write is rejected and reported, never retried blindly. This also closes the gap between FR-20's draft check and the write itself: a publish in between changes `updated_at`.

### 7.4 Book lookup

- **FR-30.** Given a name as it appears in prose (any mix of first, last, nickname or mug name, and class year), return the brothers who might match. Each result carries Book id, name fields, class year, the canonical name Book constructs, the profile URL, and whether the brother is deceased.
- **FR-31.** The Gateway sees exactly what Book's own rules let a brother-role viewer see, and nothing more: unlisted and de-brothered records never appear. Book enforces this. The Gateway doesn't re-implement it (D12).
- **FR-32.** The Gateway authenticates to Book with a Google-signed service identity. No Book credential is stored anywhere (D12). Which Book endpoint serves this is decided in Book ticket OFC-484.
- **FR-35.** *(Stretch.)* **Change feed.** Given a cursor, list the posts and pages changed since then (id, title, status, `updated_at`), mark which changes were the Gateway's own, and return a new cursor (D9).

### 7.5 Logging

- **FR-40.** Every operation, read or write and successful or not, is logged with: time, human principal, client, operation, target article, outcome, and for a refusal, which rule refused it (D13).
- **FR-41.** Logs never contain the Ghost token or any other secret. They don't contain whole article bodies, except under FR-42.
- **FR-42.** *(Open question Q1.)* Before each write, the article's prior state is saved, so a mistaken edit can be undone without restoring a site backup.

### 7.6 Authentication of callers

- **FR-50.** Every call is authenticated as an allow-listed human Google account. In Phase A, that's from Claude Code. Unauthenticated calls are refused.
- **FR-51.** *(Phase B.)* The Gateway works as a claude.ai custom connector, using OAuth with Google sign-in against the same allow-list. No authless mode exists.
- **FR-52.** *(Phase B.)* A non-human client (the Linter) can authenticate as its own service identity, logged as such.

## 8. Non-functional requirements

- **NFR-1. Credential isolation.** The Ghost staff token is readable only by the Gateway's runtime identity. Every Claude client works through tools that never return it.
- **NFR-2. Least privilege.** The Ghost identity is an Editor, not an administrator (D4). The GCP service identity holds only what the Gateway uses. Environments are separate projects (D5).
- **NFR-3. Fail closed.** If Ghost, Book or the configuration is unavailable or ambiguous, the Gateway refuses and says why. It never falls back to a weaker check.
- **NFR-4. Context frugality.** Tool outputs are compact by default and detailed on request. An article read costs proportionally to the article. Image reads have a bounded size (FR-5). A model working through a 30-article issue shouldn't exhaust its context on overhead.
- **NFR-5. Latency.** A cold start of a few seconds after an idle period is acceptable. Warm calls should feel immediate in an interactive session.
- **NFR-6. Cost.** At this volume (a few hundred calls a day at peak, near zero between issues), hosting should cost a few dollars a month at most.
- **NFR-7. Operability.** Infrastructure is reproducible from scripts in the repo. CI runs the full verification gate on every pull request, and merges deploy staging automatically. Production deploys only as a deliberate release on Forrest's word, following the dev-workflow skill's release process.
- **NFR-8. Testability.** The write policy (§7.3) is covered by automated tests that try every forbidden operation and expect refusal. Those tests are part of the CI gate.
- **NFR-9. Accessibility.** Any page a human sees, such as an OAuth sign-in or consent page in Phase B, meets WCAG 2.2 AA in light and dark themes.
- **NFR-10. Privacy.** Book data obtained by the Gateway is used only to answer the request, isn't stored beyond a short cache, and is never written to logs beyond the query and the result count.

## 9. Environments and cutover

The Gateway has a **staging** and a **production** deployment, each in its own GCP project. Staging pairs with ghost-staging and Book staging. Production pairs with pbe400.org and Book production. Profile links always point at the Book of the same environment.

**Staging content.** To test on realistic material, the published Fall 2025 and Spring 2026 articles and pages are copied from production into ghost-staging (Forrest's call). **Only posts and pages: never members, never member data.** Staging's fake member list and disabled bulk email stay exactly as they are. Whether images are copied or left pointing at production URLs is for the engineering design to settle.

**Claude's Ghost user.** On staging, a Claude Editor user is created freely. On production, it takes the third staff slot, and Forrest creates it.

**Cutover.** The Gateway touches pbe400.org only after it has been live-tested on staging and Forrest gives the word. Cutover follows the dev-workflow skill's `launch-and-cutover.md`, scaled to a read-mostly service: a written rollback (revoke the Claude user's token), and a first-hour watch of the logs for any refusal or error that wasn't expected.

## 10. Schedule

| Date | Milestone |
|---|---|
| 2026-10-29 | This PRD (draft 1). |
| 2026-10-30 | Engineering design, in a fresh session, if Forrest is available (Q3, Q4). |
| 2026-10-31 to 11-11 | Forrest traveling. Optional autonomous build of Phase 0 (Q4). |
| 2026-11-12 to 11-19 | Build Phase A against staging. Book's OFC-484, including a Book production release. |
| ~2026-11-20 | Gateway production cutover. |
| 2026-11-22 | Fall 2026 content deadline. Editing begins in earnest. |
| by 2026-12-10 | Fall 2026 published. Phase B begins after. |

## 11. Success criteria

For the Fall 2026 issue:
- Names are canonicalized and linked, and images given alt text, through the Gateway on most of the issue's articles, and Forrest judges it a real time saving.
- The logs show **zero** writes to any article that wasn't a draft, and zero successful status changes or deletes.
- The Ghost token appears in no transcript, tool result, log line or commit.
- No edit by Forrest or Claude is silently lost to a collision.

## 12. Verification items

Facts this design leans on that haven't been verified yet. Each is checked on staging early in the build, and a surprise comes back to Forrest.

- **V1.** Which Admin API endpoints a staff access token for an **Editor** can reach, and whether staff tokens are available on Ghost(Pro) for that role (D4).
- **V2.** Unpublishing and republishing an article keeps its `published_at`, and doesn't email the lead article a second time (D7).
- **V3.** Ghost's server-side HTML conversion matches the editor's paste quality on real past submissions (D10).
- **V4.** What Ghost's post revision history captures: body only, or metadata too. This feeds Q1.
- **V5.** ghost-staging runs the same major Ghost version as pbe400.org. If not, it must be upgraded before the tests mean anything.
- **V6.** Ghost(Pro)'s Admin API rate limits, against an issue-sized batch of reads and edits.
- **V7.** The `updated_at` collision check applies to pages as well as posts, and to every field the Gateway writes (D8).

## 13. Open questions

- **Q1. Pre-write snapshots (FR-42).** *Recommendation: yes.* Only the Gateway's own writes are snapshotted, never Forrest's edits in the Ghost editor. Compressed, an issue's worth (30 articles × about 20 writes) is a few megabytes. Store them in a Cloud Storage bucket that deletes them after about 90 days, with each log line pointing to its snapshot. Cost: well under a dollar a year. Forrest's call.
- **Q2. Book endpoint shape (FR-32).** Fill in Book's existing `GET /api/roster` and let the Gateway match candidates itself, or add a server-side search endpoint to Book? *Recommendation: the roster*, which is designed, its authentication built, and smaller. Decided at OFC-484's plan gate.
- **Q3. Adversarial review depth.** The design methodology calls for several independent review passes. Given the deadline: *recommendation* is one security-focused pass on the engineering design (credential, authentication, the write policy, FR-25) before building, with the other angles after 2026-12-10.
- **Q4. Using the travel window.** If the engineering design and coding plan are approved by 2026-10-30, autonomous sessions could build Phase 0 and the walking skeleton between 2026-10-31 and 11-11, stopping with PRs ready for Forrest on 11-12. It is ambitious, and it draws on extra usage. Forrest's call.
- **Q5. The canonical-name rule.** For names to be useful in the MVP, Claude needs Forrest's rule written down: the format, first mention versus later ones, and what gets linked. It belongs to the assistant editor, not the Gateway, but the MVP's value depends on it.
- **Q6. Active content (FR-25).** Confirm that Claude should never be able to add scripts, raw HTML cards or code injection, even to a draft. The cost: any article that needs an embed gets it from Forrest by hand.
- **Q7. Duet and connectors.** Confirm, before Phase B starts, that Duet can use a claude.ai custom connector, and what it does with tool-permission settings.

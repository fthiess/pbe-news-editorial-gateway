# PBE News Editorial Gateway ("Gateway")

A remote MCP server that gives Claude (Claude Code, Claude Duet, claude.ai, cloud Routines) and the planned PBE News Linter editorial access to the PBE News Ghost site: read posts and pages, create drafts, edit drafts. It holds the Ghost credential server-side and enforces the write policy in code. Sibling projects live in the parent `PBE News/` workspace (see its `CLAUDE.md`); Book (`pbe-address-book`) supplies the brother roster.

**Phase:** design. `docs/PRD.md` is drafted; the engineering design comes next, in its own session. No code yet.

## Development workflow

Sessions follow the committed **`dev-workflow` skill** (`.claude/skills/dev-workflow/`) — invoke it before any task that changes code or docs. The design stage follows its `design-methodology.md`. Discuss and plan first and wait for Forrest's explicit go; all work on a branch → PR. **There is no CI yet, so Gate 4's merge-on-green does not apply here: every merge waits for Forrest's OK** until CI enforces the full verification gate and branch protection requires it.

If another session may be active in this repo, work in a git worktree under `.claude/worktrees/`, never the main checkout.

## Documentation map

- `docs/PRD.md` — the product requirements: what the Gateway must do, for whom, in which phase.
- `DECISIONS.md` — the append-only decision log (`D*` design decisions, `N*` implementation notes). Append significant decisions as they land, update `DECISIONS-INDEX.md` in the same change, and propagate the change into the affected docs once, in place. **Consult the index first and read only the governing entries.**

Deferred work and bugs are Linear tickets: team Techgnosys (`OFC-*`), project **PBE-Gateway**. Book-side work goes in project PBE-Book.

## Standing invariants (do not violate)

- **This repo is PUBLIC.** No secrets, ever. The Ghost staff token lives only in GCP Secret Manager and is read only by the Gateway's own service identity — never in the tree, in a log line, in a tool result, or in a chat session. Grep the staged diff for secrets before every push.
- **A real brother's name is never written to this repo** — not the tree, not a commit message, not a PR description. Write "a brother", "the reporter", "the backup editor"; Forrest's own name is the exception. Names belong in Linear. No checker can enforce this, so it rests on this file.
- **The write policy is enforced in Gateway code, not by instructions to the model** (D7): the Gateway writes only to **drafts** (posts and pages); everything else — published, scheduled — is read-only. It never changes an article's status, never deletes, never sends email, and never touches members, tiers, newsletters, settings, themes or staff users. A tool that could do any of those must not exist.
- **Every write carries the `updated_at` it read** and is never blindly retried on a collision (D8).
- **Never test against the live pbe400.org.** Development targets ghost-staging (`staging.pbe400.org`) until the deliberate production cutover. Staging's member list stays fake: content imports from production carry **posts and pages only, never members**, and staging's `bulkEmail` stays unset.
- **Any user-facing surface** (for example an OAuth sign-in page) targets **WCAG 2.2 AA**.

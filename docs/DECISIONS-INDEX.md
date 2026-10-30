# PBE News Editorial Gateway — Decision Index

The subject → currently-authoritative-decisions map for [DECISIONS.md](DECISIONS.md). **Consult this file first and read only the entries it cites.** Every change that appends to the log updates the affected line(s) here in the same commit.

Chains run oldest → newest; **bold** marks the current word on a matter; ~~struck~~ IDs are wholly superseded but kept for their reasoning. `D*` = design decisions, `N*` = implementation notes.

## Architecture & hosting

- Mechanism: **D1** (purpose-built remote MCP server, custom connector, credential and guardrails server-side).
- Hosting: **D5** (Cloud Run + Secret Manager, own GCP project per environment).

## Scope, consumers & phasing

- Consumers: **D2** (Duet, Claude Code, claude.ai web, cloud Routines, the Linter).
- Scope: **D3** (editorial access, not admin, not the assistant editor).
- MVP & schedule: D15 (names + alt text define the MVP) → **D17** (on hold until Fall 2026 is published; Spring 2027 target; one MVP for all Claude clients, Phases A and B merged).
- Judgment vs. Linter: D14 → **D17** (alt text stays out of the Linter; who canonicalizes names for Spring 2027 is open, PRD Q8).

## Identity & authentication

- Ghost identity: **D4** (dedicated Claude Editor staff user, staff access token).
- Human callers: **D6** (Google sign-in allow-list; several humans, one Claude identity; human recorded per action).

## Write safety

- Policy: **D7** (drafts freely editable; everything else read-only; no status change, delete or email).
- Concurrency: **D8** (`updated_at` on every write; no blind retry; closes D7's check-then-write race).

## Content

- Intake: **D10** (caller supplies HTML; Ghost's own conversion).
- Images: **D11** (upload/placement deferred; read + alt/caption edit in the MVP; TK markers at intake).
- Change feed: **D9** (on request only; no webhooks in the MVP).

## Book

- Roster lookup: **D12** (service-to-service Google identity; Book's projection governs; endpoint shape open in OFC-484).

## Observability

- Logging: **D13** (every action logged; pre-write snapshots an open question).

## Repository

- **D16** (public repo `pbe-news-editorial-gateway`, MIT; "Gateway"; Linear project PBE-Gateway).

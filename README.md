# PBE News Editorial Gateway

The Gateway gives Claude — and other first-party tools such as the planned PBE News Linter — controlled editorial access to the [PBE News](https://pbe400.org), the online newsletter of the Phi Beta Epsilon alumni association, which is published on Ghost.

It is a small remote [MCP](https://modelcontextprotocol.io) server. Claude clients call its tools to read articles and pages, create drafts, and make edits to drafts. The Ghost credential and every safety rule live inside the Gateway, not in the calling session: the Gateway writes only to draft articles, never changes an article's publication state, and never deletes anything.

**Status:** design stage. The requirements are in [`docs/PRD.md`](docs/PRD.md); decisions and their reasoning are in [`DECISIONS.md`](DECISIONS.md) (start from [`DECISIONS-INDEX.md`](DECISIONS-INDEX.md)). No code yet.

Licensed under the MIT License (see [`LICENSE`](LICENSE)).

# xyz-spec — The xyz Specification

**The contract every xyz SDK implements, in every language.**

xyz turns one command definition into three interfaces — CLI, HTTP REST,
MCP tool server. This repository is the cross-language specification that
keeps implementations and the user experience identical, now that the
library ships in Go (xyz-go), Rust (xyz-rust), and — next — Node, Java and
beyond.

> 中文入口：[README.zh-CN.md](README.zh-CN.md)

## Documents

| Document | What it is |
|---|---|
| [spec.md](spec.md) | **The normative contract** (v0.4.2). Definition vocabulary, types, defaults, validation, the invocation pipeline, error taxonomy (incl. rich code/detail/status and the shared error body), rendering (incl. per-channel output functions and CLI `--format`), the three frontends (incl. HTTP application-identity/SDK-version headers and MCP result `_meta`), the root dispatcher, embedding surface, experience conventions, governance. RFC 2119 language. |
| [spec.zh-CN.md](spec.zh-CN.md) | Chinese mirror of spec.md (for readability; English is normative on conflict). |
| [conformance.md](conformance.md) | The conformance programme: Class A/B checklists, the 11-command showcase fixture with byte-exact golden outputs, and the required feature matrix. |
| [deviations.md](deviations.md) | The deviations register. Every SDK files here what differs from spec.md, classed as language-forced / SDK limitation / extension, statused open / resolved-in-spec / closed-by-implementation / retired, re-reviewed at every spec release. |

## SDK status

| SDK | Package | Specification target | Notes |
|---|---|---|---|
| [xyz-go](https://github.com/ejfkdev/xyz-go) | `github.com/ejfkdev/xyz-go` (v0.4.2) | **v0.4.2** (baseline anchor) | Go reference implementation |
| [xyz-rust](https://github.com/ejfkdev/xyz-rust) | crates.io `xyz-rust` 0.4.2 | v0.4.1 (spec v0.4.2 clauses pending) | Rust reference implementation |

### Compatibility matrix

Each SDK release targets exactly one spec version (the anchor is recorded in
[spec.md](spec.md) and in the SDK's own release note). Current anchors:

| spec | xyz-go | xyz-rust |
|---|---|---|
| v0.4.2 | v0.4.2 ✅ | pending (rust 0.4.2 → spec v0.4.1) |
| v0.4.1 | v0.4.1 | 0.4.2 ✅ |
| v0.4.0 | v0.4.0 | 0.4.0 |

The v0.4.2 surface — rich errors (§8.5/§8.6), CLI `--format` with the
full-name/conflict rule (§10.7), HTTP application-identity + SDK-version
headers (§11.6) and MCP result `_meta` (§12.8) — is shipped by xyz-go
v0.4.2; xyz-rust catches up in a follow-up release. Until then the clauses
that are MAY (output functions §9.5, headers §11.6, `_meta` §12.8, rich-error
layers §8.5) impose no obligation on xyz-rust 0.4.2, which remains conformant
against v0.4.1. Full history: see the git tags of each repository.

xyz-go files one open deviation (D-go-01, tagged unions); xyz-rust files
[deviations.md](deviations.md) (duration sign, rendering via serialised
values, transport differences of the official Rust MCP SDK, version
injection).

## How to read this repository

1. SDK implementers start at [spec.md](spec.md) §1–§9, which define everything
   user-facing; then §10–§13 for the frontends; §14–§15 for the embedding and
   experience surface.
2. Before calling a build conformant, run [conformance.md](conformance.md) —
   checklist, fixture with golden outputs, six-way feature matrix — and file
   the deviations register.
3. Proposing a change: open an issue/PR against this repository with the
   reference-implementation diff attached (spec §17.2). Spec bumps are
   semver: MAJOR changes normative behaviour, MINOR adds surface, PATCH
   clarifies.

## License

[MIT](LICENSE)

Release process: see [RELEASING.md](RELEASING.md) — every tag carries a full changelog (the annotated message *is* the release note).

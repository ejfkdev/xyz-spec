# Deviations Register

Every SDK MUST keep a register of behaviours that differ from
[spec.md](spec.md) v0.4.5, each entry referencing the spec section it
departs from, declaring the class — **language-forced** (the language cannot
express the spec as written), **SDK limitation** (not yet implemented), or
**extension** (adds surface beyond the spec, not a divergence) — and the
resolution. Entries are re-reviewed at every spec release (§17.3) and either
promoted into the spec or closed.

Format per entry:

```
### D-<sdk>-<nn> · spec §<section>
Class: language-forced | SDK limitation | extension
Status: open | resolved-in-spec | closed-by-implementation | retired
Detail: …
```

Status vocabulary:

- **open** — the divergence still exists; re-reviewed at each spec release.
- **resolved-in-spec** — a spec revision absorbed the behaviour as normative
  text (or an explicit MAY), so it is no longer a divergence. The entry is
  kept for provenance.
- **closed-by-implementation** — the SDK shipped the behaviour the spec
  already required (or an extension was withdrawn); no spec change was
  needed. Distinct from resolved-in-spec, which changes the spec.
- **retired** — the entry is void (the feature was removed, or the entry was
  filed in error). Kept only so the identifier is never reused.

---

## xyz-go v0.4.5 (baseline)

Prior baselines held no deviations. xyz-go v0.4.5 implements spec v0.4.5 —
including the §8.5/§8.6 rich errors, §10.7 `--format` (with the §10.7
full-name/conflict rule), §11.6 application-identity + SDK-version response
headers and §12.8 result `_meta` — except for one open deviation:

- **D-go-01 · spec §4.7 (tagged unions / oneOf):** Class SDK limitation,
  Status open. Go has no language-native enum-argument support; the semantics
  are settled by the xyz-rust v0.4.0 implementation and conformance A.52, so
  a Go implementation is planned against that fixture. Until then Go rejects
  tagged-union types at definition time by construction (there is no type to
  express them). §17.3 review at spec v0.4.1: the §4.7 tail (per-frontend
  skip policy) is a MAY and imposes no new obligation on a union-less SDK.
  §17.3 review at spec v0.4.5: unchanged — entry remains open until Go
  grows a union argument surface.

Resolved in v0.4.2 (absorbed into the spec, kept for provenance):

- **D-go-03 · spec §9.5 (per-channel output functions):** Class extension,
  Status resolved-in-spec. Per-channel output functions on
  `CliHints/HTTPHints/MCPHints` (`Output` fields) let a command adopt entirely
  channel-specific result renderings (rich CLI text/color, full HTTP response
  control, custom MCP textContent) while machine modes keep their normative
  behaviour. Precedence is pinned as machine/alternate format > Output >
  §12.7 envelope > default rendering; error paths never pass through Output.
  Originally registered as a Go extension at spec v0.4.1; spec v0.4.2 §9.5
  absorbs it as an *optional* (MAY) cross-language clause, so it is no longer
  a divergence — an SDK without output functions stays conforming, and the
  new §10.7 `--format` precedence is defined against it. Conformance: the
  §9.5 chain is exercised by A.54; the optional extension itself is not
  separately A-numbered.

Closed by implementation in v0.4.0 (kept for provenance):

- **D-go-02 · spec §12.7 (content-block results):** Class SDK limitation,
  Status closed-by-implementation. xyz-go v0.4.0 shipped the behaviour §12.7
  already required: a reserved-envelope detector (`block` package), CLI
  projection (inline text + temp-file paths for binary blocks), MCP `Content`
  verbatim + envelope `structuredContent`, and HTTP pass-through of the
  envelope body. No spec change was needed. Covered by
  `TestCLIBlockProjection`, `TestMCPBlockResult`,
  `TestHTTPBlockEnvelopePassThrough` and `block.TestDetect*`.

## xyz-rust 0.1.0

### D-rust-12 · spec §12.4a (MCP tool-name override)
**Class:** SDK limitation
**Status:** closed-by-implementation
**Detail:** Go v0.3.3 added `MCPHints.Name`; Rust carried it in xyz-rust
v0.4.2 (`MCPHints.name`, grammar-checked at registration — override-only
advertisement and call routing, locked by `mcp::mcp_test::mcp_name_override`).

### D-rust-01 · spec §4.3 (Duration)
**Class:** language-forced
**Status:** open
**Detail:** `std::time::Duration` is unsigned; negative durations
(`-1.5h`) are rejected with "negative duration not supported" at parse
time. Go's `time.Duration` holds signed values. The spec §4.3 duration row
defines wire strings but not sign semantics; resolution targeted for spec
0.2: either "durations MUST be non-negative" or Rust migrates to a signed
representation of its own wrapper type.

### D-rust-02 · spec §9.1 (map vs struct rendering)
**Class:** language-forced
**Status:** open
**Detail:** Rendering walks the serialised JSON value: struct and
`map<string,V>` are indistinguishable after serialisation (both render as
the KV block; declaration order preserved via serde_json
`preserve_order`). Go renders struct fields in declaration order and maps
sorted by key. Observable contract (KV pairs) is identical for structs;
map key order for truly dynamic maps is the language's serialisation order.
Spec §9.4 records the residue ruling; re-review at each release.

### D-rust-03 · spec §10.2 (usage-error classification)
**Class:** extension → resolved
**Status:** resolved-in-spec
**Detail:** Rust models usage errors (unknown flag etc.) as
`Kind::InvalidInput` so the exit code is 2; spec §10.5 already pins usage →
2 for both paths, making the internal modelling unobservable. Kept here for
transparency only.

### D-rust-04 · spec §11.4 (middleware primitives)
**Class:** language-forced
**Status:** open
**Detail:** gzip rides tower-http's compression layer: it honours
q-values (`gzip;q=0` → no compression), a superset of Go's raw
`contains("gzip")` check; and is size-gated to compress any size
(`SizeAbove(0)`) to match Go. Per-request timeout is a `TimeoutLayer`
(HTTP 408) rather than Go's server `WriteTimeout` (connection-level, no
status body). Spec §11.4 pins middleware order/behaviour, not the
mechanism; the timeout status-body difference is user-visible and stays
registered.

### D-rust-05 · spec §11.5 (server header timeout)
**Class:** SDK limitation
**Status:** open
**Detail:** Go configures a 10 s `ReadHeaderTimeout` baseline; the Rust
serve path does not set a separate header-read timeout (only the optional
per-request `TimeoutLayer`). Spec §13.3 documents the `--timeout` contract;
the header-timeout baseline is a Go detail the spec currently does not
require — re-review for spec 0.2.

### D-rust-06 · spec §12.3 (SSE transport absent)
**Class:** language-forced
**Status:** resolved-in-spec
**Detail:** The official Rust SDK removed legacy HTTP+SSE with the
2026-07-28 revision. `mcp sse` therefore fails fast with a clear error and
exit 2. Spec §12.3 codifies exactly this rule ("when a transport does not
exist… MUST fail fast"), so the behaviour is now normative, not a
deviation. Registered for provenance.

### D-rust-07 · spec §12.3 (streamable HTTP Host allowlist)
**Class:** extension
**Status:** resolved-in-spec
**Detail:** rmcp's streamable HTTP server defaults to loopback-only
`Host` validation (DNS-rebinding protection), and this SDK keeps the
default. Spec §12.3 records it as a SHOULD. Status resolved-in-spec.

### D-rust-08 · spec §13.8 (version injection)
**Class:** language-forced
**Status:** open
**Detail:** Rust has no `-ldflags -X`; version default is
`CARGO_PKG_VERSION` overridable at runtime via `set_version`. Spec §13.8
already permits per-language mechanisms. Registered for the record.

### D-rust-09 · spec §15.3 (core dependency policy)
**Class:** language-forced → resolved-in-spec
**Status:** resolved-in-spec
**Detail:** Rust std contains no JSON implementation; the core therefore
carries serde + serde_json (and chrono for timestamps) as the
"language-standard JSON" the spec §15.3 clause allows, with everything else
zero-dependency. Spec wording already absorbs this; registered for
provenance.

### D-rust-11b · spec §15.5 (transport-word catalog keys)
**Class:** language-forced → resolved-in-spec
**Status:** resolved-in-spec
**Detail:** The catalog keys listing the MCP transports
(`overview.mcp_mode`, `mcp.usage`, `mcp.err_unknown_transport`,
`mcp.err_sse_removed`) differ per SDK: Go lists `stdio|sse|http`, Rust
`stdio|http` (its official SDK removed SSE). Spec §15.5 explicitly permits
SDK-specific wording for these keys. Registered for provenance.

### D-rust-11 · spec §13.7 (per-request HTTP cancellation)
**Class:** SDK limitation
**Status:** open
**Detail:** Go cancels the handler context when the client disconnects
(`r.Context()`). The Rust HTTP handler receives the serve-level `Ctx`
only; a client disconnect does not cancel an already-running (synchronous)
handler. Spec §13.7 records the asymmetry as SHOULD; future Rust work may
switch to a per-request child context threaded through the shared
pipeline.

### D-rust-10 · spec §4.1 (serde rename fallback)
**Class:** extension
**Status:** open
**Detail:** Beyond the canonical `#[xyz(name = "...")]`, the Rust derive
also honours an existing `#[serde(rename)]` (and struct-level
`rename_all`) as the wire-name fallback, mirroring the language's own
serialisation convention. Spec §4.1 mentions the fallback; a future spec
release should decide whether the fallback is normative across languages
with established serialisation attributes (e.g. Java's Jackson
annotations).
### D-rust-13 · spec §4.7 (CLI union degradation)
**Class:** SDK limitation
**Status:** resolved-in-spec
**Detail:** xyz-rust v0.4.1 shipped union fields as a CLI registration
error; v0.4.2 degrades them to skipped fields (command stays usable).
Spec v0.4.1's §4.7 tail now permits per-frontend skipping explicitly,
recording the behavior as conformant rather than divergent. Closed.

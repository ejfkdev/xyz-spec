# Deviations Register

Every SDK MUST keep a register of behaviours that differ from
[spec.md](spec.md) v0.1.0, each entry referencing the spec section it
departs from, declaring the class — **language-forced** (the language cannot
express the spec as written), **SDK limitation** (not yet implemented), or
**extension** (adds surface beyond the spec, not a divergence) — and the
resolution. Entries are re-reviewed at every spec release (§17.3) and either
promoted into the spec or closed.

Format per entry:

```
### D-<sdk>-<nn> · spec §<section>
Class: language-forced | SDK limitation | extension
Status: open | resolved-in-spec
Detail: …
```

---

## xyz-go v0.1.0 (baseline)

No deviations. xyz-go is one of the two anchors the spec was written from;
where ambiguous, GO behaviour is normative.

## xyz-rust 0.1.0

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
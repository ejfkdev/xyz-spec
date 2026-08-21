# The xyz Specification

**Version 0.1.0** · Status: Baseline · Last updated: 2026-08-21

This document is the normative contract for every xyz SDK. A library may call
itself an *xyz SDK* only if it implements this specification. The key words
**MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and **MAY** are to be
interpreted as described in RFC 2119. Reference to section numbers refers to
this document.

Reference implementations (normative anchors for ambiguous cases):

| SDK | Path / package | Version baseline |
|---|---|---|
| xyz-go | `github.com/ejfkdev/xyz-go` (package name `xyz`) | v0.1.0 |
| xyz-rust | crates.io `xyz-rust` (lib `xyz-rust`; derive helpers in `xyz-rust-macros`) | 0.1.0 |

> Also available: [中文版](spec.zh-CN.md)

---

## 1. Purpose and scope

**1.1.** xyz turns one command definition into three live interfaces — a CLI,
an HTTP REST service, and an MCP tool server — through one shared pipeline.
A user who knows xyz in one language must recognise it immediately in
another: same definition vocabulary, same dispatch model, same errors, same
rendered output.

**1.2.** What this specification pins down (normative):

- the *definition contract* (§3–§6): names, field vocabulary, types, defaults,
  validation;
- the *invocation pipeline* (§7): one path from transport input to rendered
  output;
- the *error taxonomy* (§8): one classification driving all three channels;
- the *rendering contract* (§9): envelope-free output shapes;
- the *frontends* (§10 CLI, §11 HTTP, §12 MCP) and the *root dispatcher* (§13);
- the *embedding surface* (§14) and *experience conventions* (§15);
- the *conformance programme* (conformance.md) that every SDK must pass.

**1.3.** What this specification deliberately leaves to the language
(implementation-defined):

- idioms for errors, builders and configuration objects (see §15);
- the choice of JSON implementation and HTTP stack — with the constraint
  that MCP **MUST** use the *official* Model Context Protocol SDK for the
  language (§12.1);
- feature-trimming mechanisms, which map onto the language's own build
  system (§15.4).

---

## 2. The paradigm

**2.1.** *Define once.* A command is a name, an argument type, a handler, and
per-interface hints. The handler receives a fully decoded, defaulted and
validated argument value plus a cancellation context; it returns a result
value or an error.

**2.2.** *One pipeline.* Every frontend reduces its transport input to a
string-keyed map of values, calls the shared `Invoke`, and renders the result
or the classified error. There is exactly one implementation of decode,
defaults, validation and execution per definition — frontends never
re-derive it. The pipeline order is normative, see §7.

**2.3.** *Fail at registration.* Any definition error — bad name, bad field
vocabulary, unsupported validation rule, route conflict, reserved top-level
name — MUST surface when the command is registered (or earlier, e.g. at
compile time), never at the first invocation.

**2.4.** *The shell is uncuttable.* The mode words, `help`, `-v/--version` and
`completion` work in every configuration and every trimmed build (§13.6).

---

## 3. Command identity

**3.1.** A command name MUST match `^[A-Za-z0-9][A-Za-z0-9._-]{0,127}$`
(maximum 128 characters; first character alphanumeric; remainder
alphanumeric or one of `. _ -`). This is MCP tool-name compatible and also
bounds CLI subcommand segments.

**3.2.** A dot (`.`) separates CLI subcommand levels: `user.add` names the
subcommand path `user add`. Dots do not affect HTTP or MCP (the full name is
the tool name / not used for routing).

**3.3.** Summary is a one-line description. Description is the longer text.
MCP tool description is `summary` alone when `description` is empty,
`description` alone when `summary` is empty, and
`summary + "\n\n" + description` otherwise (§12.4).

**3.4.** The top-level segments of registered names MUST NOT collide with the
mode words (§13.1), or registration fails.

**3.5.** Registering a duplicate name MUST fail with an error that names the
conflict.

---

## 4. Argument definition

### 4.1 Vocabulary table

Fields are declared on a struct/class native to the language, annotated with
the following canonical vocabulary. The left column is the canonical concept;
the middle columns show the two reference spellings; the right column is the
semantics every SDK must reproduce.

| Concept | xyz-go (struct tag) | xyz-rust (attribute) | Normative semantics |
|---|---|---|---|
| Wire name | `json:"user_name"` | `#[xyz(name = "user_name")]` | Name on the wire (CLI flag, HTTP param/field, MCP schema key). Default = the language's native field name. Rust additionally honours a serde rename as fallback. |
| Exclude | `json:"-"` | `#[xyz(skip)]` | Field excluded from binding and generated schemas. It MAY still receive injected values (env, HTTP header) keyed by the language-internal field name (§4.4). MUST NOT be combined with `required`. |
| Description | `desc:"..."` | `#[xyz(desc = "...")]` | Human description; appears in CLI help and JSON Schema. |
| Global default | `default:"18"` | `#[xyz(default = "18")]` | Default applied by `Invoke` for absent values; parsed per field type at registration (parse failure = registration error). |
| Required | `required:"true"` | `#[xyz(required)]` | The key must be present (a present zero value does satisfy presence, §7.3). |
| Enum | `enum:"a,b"` | `#[xyz(enum = "a,b")]` | Allowed values, parsed per field type. Scalars only; non-scalar enum = registration error. |
| Validate | `validate:"min=2,email"` | `#[xyz(validate = "min=2,email")]` | Validation rule set, see §5. |
| Secret | `secret:"true"` | `#[xyz(secret)]` | Redact value in help text, logs and error echoes. |
| CLI bindings | `cli:"shorthand=a,positional,hidden,env=VAR,-"` | `#[xyz(cli = "shorthand=a,env=VAR")]` | §4.5. |
| HTTP location | `http:"query"` | `#[xyz(http = "query")]` | §4.6. |
| HTTP wire name | `httpName:"X-Key"` | `#[xyz(http_name = "X-Key")]` | Wire-name override for the HTTP transport (typically a header name). |

Languages without struct tags/attributes (e.g. TypeScript decorators, Java
annotations, Kotlin properties with named arguments) MUST provide a spelling
for the same concepts; the canonical concept names above SHOULD be preserved
as identifiers where the language allows it.

### 4.2 Merging rules

Command-level field hints (per-transport configuration maps keyed by wire
name *or* language-internal field name, case-insensitively for the latter)
merge **over** the tag/attribute values. A *zero-valued* hint field keeps the
tag value — so one may set the shorthand in the tag and only override the
default in the builder. Keys naming unknown or nested (dotted) fields MUST be
a registration error. Hint defaults are validated for convertibility at
registration.

### 4.3 Types

SDKs MUST support, as argument fields:

| Category | Go spelling | Rust spelling | Notes |
|---|---|---|---|
| String | `string` | `String` | |
| Boolean | `bool` | `bool` | |
| Signed integers | `int, int8…int64` | `i8…i64` | width-checked conversions |
| Unsigned integers | `uint, uint8…uint64` | `u8…u64` | negatives rejected |
| Floating point | `float32, float64` | `f32, f64` | |
| Timestamp | `time.Time` | `chrono::DateTime<Utc>` | RFC 3339 on the wire, JSON Schema `format: date-time` |
| Duration | `time.Duration` | `std::time::Duration` | Go-style duration strings (`300ms`, `1.5h`, `2h45m`, `µs`), JSON Schema `format: duration` |
| Bytes | `[]byte` | `Vec<u8>` | Wire: string or byte array; JSON Schema `string` |
| Slice | `[]T` | `Vec<T>` (T ≠ u8) | |
| Optional/pointer | `*T` | `Option<T>` | null ≈ absent; nullable schema = element schema |
| Nested struct | struct | struct | inlined into the schema (no `$defs`) |
| Named scalar | `type Port int` (automatic) | `#[derive(XyzField)]` newtype | transparently behaves as its underlying scalar |

**MUST NOT** be accepted as argument fields: maps, dynamic/interface values,
channels, functions, complex numbers, and recursive types (direct or
mutual). Reference implementations reject these at registration (or compile
time, which is earlier and therefore also conformant). Nesting depth SHOULD
be bounded defensively (reference implementations use a depth guard of 20).

**4.4. Lossless conversion** — The shared decoder accepts three source forms:
strings (CLI), JSON shapes (HTTP body), and arbitrary JSON (MCP). Conversions
between numeric forms MUST be lossless: a non-integral value (e.g. `3.7`)
MUST NOT silently become an integer; width overflow MUST error; negative to
unsigned MUST error. Booleans accept the standard true/false string forms
(`1,t,T,TRUE,true,True` and their false counterparts). Timestamps accept
RFC 3339 only.

### 4.5 CLI bindings (the `cli:` concept)

`shorthand=X` — one character; `positional` — consumes the next positional
argument; `hidden` — omitted from `--help` listings; `env=VAR` — fall back to
the environment variable when the flag is unset; `-` — invisible to the CLI
frontend. Validation: shorthand MUST be exactly one character; duplicated
shorthands within a command are a registration error; positionals marked
`required` MUST form a prefix of the positional list (a required positional
after an optional one is a registration error).

### 4.6 HTTP location (the `http:` concept)

One of `query` (the default when unset), `path`, `header`, `form`, `body`.
Unknown locations are a registration error. `http_name` overrides the wire
name; precedence is http_name > wire name > language field name.

---

## 5. Validation

**5.1.** The supported rule set is exactly:
`required, omitempty, min, max, len, gt, gte, lt, lte, oneof, email` —
a go-playground/validator-compatible subset. Rule syntax: comma-separated;
numeric rules take one numeric parameter (`min=2`); `oneof` takes
space-separated values (`oneof=a b`). An unsupported or malformed rule MUST
fail at registration.

**5.2.** Semantics:

- `required` — the *zero value* fails. Zero-value definitions, per type:
  string empty; numbers 0 (±0.0); bool false; slice/bytes empty; pointer
  nil/None; nested struct zero iff all fields zero.
- `omitempty` — zero value skips **all** rules of the field.
- `min`/`max`/`len` — strings and slices compare length; numeric types
  compare value. All comparisons happen in exact float64 arithmetic
  (integers up to 2^53 compare exactly; fractional thresholds such as
  `gt=1.5` compare as written, never truncated).
- `gt`/`gte`/`lt`/`lte` — numeric types only, float64 comparisons;
  anything else fails the rule.
- `oneof` — compares the value's display form against the listed forms.
- `email` — strings only, matching `^[^@\s]+@[^@\s]+\.[^@\s]+$`-equivalent.

**5.3.** Rule checking recurses into nested structs and into elements of
struct slices/optionals. The first failing rule produces the field error
`invalid value for field "<wire name>": <rule>` classified `invalid_input`
(§8).

**5.4.** Enum membership is enforced during decode (not validation); the
error message MUST include the acceptable values.

---

## 6. Default layering

**6.1.** Precedence for one field (CLI shown; the pattern is identical per
frontend):

```
explicit input > env fallback (CLI only) > interface default > global default > zero value
```

**6.2.** Each frontend injects its own interface-specific defaults into the
argument map *before* calling `Invoke`; `Invoke` then applies global
defaults for remaining absent keys. Explicit input always wins over both.

**6.3.** The MCP-specific default additionally replaces the `default` value
in the generated `inputSchema` (the schema is the MCP tool's contract).

**6.4.** The interface defaults of a frontend are exposed on the entry
(`CLIDefaults` / `HTTPDefaults` / `MCPDefaults` equivalents) so adapters
(§14) inject identical values.

---

## 7. The invocation pipeline

`Invoke(ctx, args-map) -> rendered-agnostic result value`

Normative order:

1. **Bind & decode** each declared field from the map (key = wire name;
   key = language field name for excluded/injection fields). Unknown keys are
   ignored.
2. **Convert** with §4.4 losslessness; fail = classified `invalid_input`
   error naming the field.
3. **Enum check** on converted value.
4. **Absent keys**: apply global default → else `required` error
   (`field "<name>" is required`) → else zero value.
5. **Validate** the completed value (§5).
6. **Run the language handler** with the cancellation context.
7. Return the result **as-is**; serialisation for the wire happens in the
   frontend (§9).

JSON `null` input is treated as absent at every level.

---

## 8. Error taxonomy

**8.1.** Kinds (stable identifiers):

| Kind | Meaning |
|---|---|
| `invalid_input` | malformed arguments, failed decode/validation |
| `unauthorized` | caller must authenticate |
| `forbidden` | authenticated but insufficient permission |
| `not_found` | target does not exist |
| `conflict` | operation collides with existing state |
| `canceled` | caller cancelled the operation |
| `unavailable` | dependency temporarily down |
| `internal` | fallback for unclassified failures |

**8.2.** Channel mappings (normative):

| Kind | HTTP status | CLI exit code | JSON-RPC / MCP |
|---|---|---|---|
| `invalid_input` | 400 | 2 | -32602 |
| `unauthorized` | 401 | 1 | -32010 |
| `forbidden` | 403 | 1 | -32011 |
| `not_found` | 404 | 1 | -32001 |
| `conflict` | 409 | 1 | -32009 |
| `canceled` | 499 (non-standard, documented) | 1 | -32012 |
| `unavailable` | 503 | 1 | -32603 |
| `internal` | 500 | 1 | -32603 |

**8.3.** Classification walks the language's error chain (cause/source):
the first coded classification found wins; an uncoded non-nil error is
`internal`. The SDK MUST let user handlers return their own conforming errors
(Go: the SDK exports `errs.New(kind, ...)`; Rust: `errs::new(kind, ...)`) and
MUST preserve the classification when the language wraps errors.

**8.4.** Errors on the HTTP and MCP channels carry the *most specific* cause
message (innermost known cause, not the wrapper).

---

## 9. Rendering (envelope-free)

**9.1.** Results are never wrapped in an envelope (no `{"data": ...}`). The
human channel (CLI) renders as follows:

| Result | CLI rendering |
|---|---|
| null / absent | nothing |
| string | the raw string (one line) |
| bool / numbers | the bare value |
| timestamp | RFC 3339 string |
| duration | canonical duration string (`300ms`, `1.5s`, `1h2m3.5s`; sub-second values use milliseconds) |
| slice of scalars | one element per line |
| struct | aligned `key  value` pairs (two-space gutter, keys = wire names) |
| slice of structs | aligned table: header row, dash row, data rows; every cell padded to its column width including the last |
| map | aligned key/value pairs |

Exact cell formatting (§9.4), including the integral float display (`3`, not
`3.0`) and trailing padding, is part of the contract — byte-identical CLIs
across SDKs is the goal. Each SDK SHOULD ship a golden-output test comparing
against the conformance fixtures (conformance.md).

**9.2.** JSON mode (CLI `--json`, HTTP responses, MCP `structuredContent`)
serialises the same result as a bare JSON value with two-space indentation
and a trailing newline when rendered as a document fragment (CLI/HTTP);
MCP structured content is the JSON value itself.

**9.3.** HTTP error bodies are compact (single-line)
`{"error":"<message>"}` + newline; HTTP result bodies are the pretty-printed
bare JSON + newline. Success responses use `Content-Type: application/json;
charset=utf-8`.

**9.4.** Field order on the wire follows declaration order (reference
implementations preserve declaration order; maps likewise render in the
language's serialisation order). Float display rounds through the shortest
representation the language's number formatting gives (`2.5`, `3`).
Aligned rendering measures column **widths in bytes** and pads **in
characters** (the reference implementations' hybrid; required to keep
tables byte-identical when CJK text appears in headers or cells).

Note (language-forced divergence, registered in deviations.md): languages
without structural reflection may not distinguish struct and map values
after serialisation; both render as key/value pairs, which is the observable
contract anyway.

---

## 10. CLI frontend

**10.1. Tree.** Dotted names become nested subcommands. Aliases are equal to
subcommand names for dispatch purposes but are not listed in the parent's
help. Registering a path conflicted by an existing leaf or node, or an alias
colliding with a sibling, MUST fail at registration.

**10.2. Flags.** Long flags use the wire name (`--name value`, `--name=value`);
shorthands via `-x value`, `-xvalue`, `-x=value`; booleans take no argument
and may take an explicit `=bool`; slice flags accumulate across repetitions;
`--` terminates flag parsing and makes the remainder positional. Unknown
flags are usage errors (see 10.5).

**10.3. Positionals.** Bound in declaration order; `required` positionals
form a prefix; the count must lie in `[min, max]` with a usage error stating
the expected and received counts (reference message:
`<path>: 位置参数数量不符（需要 a 到 b 个，收到 n 个）` — localised wording
allowed, structure mandatory).

**10.4. Built-ins.** `-h/--help` prints the help for the deepest matched node
(parents list children); `-v/--version` prints `<bin> version <version>`
and exits 0; `--json` switches result rendering to JSON; the `--` terminator
(§10.2) stops `-v`, `--version` and `--json` recognition — tokens past it
are positional data, never switches; `completion bash|zsh|fish` emits a
working completion
script for the binary name; unknown shells exit 2. Help layout order:
description → `Usage:` → optional `Aliases:` → `命令:`/`Flags:` → `Global
Flags:` / assistance lines, with inline hints `(default …)`, `(env …)`,
`(oneof a|b)` woven into flag descriptions.

**10.5. Exit codes.** Success 0. Command failures: the taxonomy's code
(§8.2). Usage errors (unknown flag, missing argument, positional count):
2. Registration errors reported at startup: 2. `validate`/decode failures are
`invalid_input` → 2.

**10.6. Output contract.** Command results go to stdout; errors and
diagnostics go to stderr. Diagnostics carry the `xyz[level]:` prefix
(log level via the global config, §13.5); the default level is `info`.

---

## 11. HTTP frontend

**11.1. Routes.** Commands with HTTP hints define `METHOD path` routes;
`{name}` is a single-segment path parameter. Conflicting method+path pairs
MUST be a registration error. Commands without HTTP hints are not routed.

**11.2. Binding.** Validated order for building the argument map:
interface defaults (base) → JSON body merge (methods other than GET/HEAD;
read cap SHOULD be 1 MiB) → per-field: path params (wire name), query
values (default location; slice fields collect all repeats, others take the
first), headers (name via §4.6), form fields (request body only). A body
that fails JSON parsing is a 400 strictly when the request *declares*
`Content-Type: application/json`; an undeclared body simply does not merge.
Fields
excluded by §4.1 receive header values keyed by the language field name.

**11.3. Built-in endpoints** (MUST exist on every HTTP frontend):

- `GET /healthz` → 200 with the exact body `{"status":"ok"}` + newline.
- `GET /openapi.json` → OpenAPI 3.0.3 document generated from the same
  `inputSchema`s; per-operation summary/parameters (path+query)/requestBody
  (POST/PUT/PATCH)/responses (200 with the output schema content, plus the
  taxonomy's 400/404/500 descriptions). `info.title`/`info.version` are
  reference-pinned to `example service`/`1` until the spec introduces a
  configurable identity for them.

**11.4. Middleware.** Served layers, outermost-first, in this fixed order:
CORS → Bearer → Gzip → router. CORS: allowlisted origins (or `*`); OPTIONS
preflights answered with 204 **before** authentication (browser preflights
carry no credentials), echoing `Access-Control-Allow-Origin` (+`Vary:
Origin`) for exact matches. Bearer: `Authorization: Bearer <tok>` must hit
the configured token set — otherwise 401 `{"error":"unauthorized"}` +
`WWW-Authenticate: Bearer`; empty token set = no auth. Gzip: compress
responses for clients sending `Accept-Encoding: gzip` regardless of response
size.

**11.5. Server**, from the global config: `addr` (default `:8080` for serve
and mcp-http), read/write/idle timeout (0 = header-timeout only), TLS when
cert+key are both given, graceful drain on cancellation (reference grant:
5 s).

---

## 12. MCP frontend

**12.1.** An SDK MUST sit on the **official Model Context Protocol SDK of
its language** (Go: `github.com/modelcontextprotocol/go-sdk`; Rust:
`rmcp`). No self-made protocol implementation is permitted.

**12.2. Protocol revisions.** The normative revision list, newest first:
`2026-07-28`, `2025-11-25`, `2025-06-18`, `2025-03-26`, `2024-11-05`.
An SDK supports the set its official SDK supports and advertises them in
preference order; `--versions` pins a subset (unknown/empty entries =
registration error).

**12.3. Transports.** `mcp stdio` (all revisions) and `mcp http` (streamable
HTTP). Transport availability follows the local official SDK: when a
transport does not exist there (reference example: the official Rust SDK
removed legacy HTTP+SSE with the 2026-07-28 revision), `mcp <transport>`
MUST fail fast with a clear error naming the transport and exit code 2 —
never silently degrade. Where an SDK does offer SSE (the Go SDK),
`mcp sse` serves revisions ≤ `2025-11-25`; streamable HTTP serving
`2026-07-28` requires stateless mode. Streamable HTTP defaults SHOULD
restrict `Host` to loopback unless configured wider (the Rust SDK does this
to prevent DNS rebinding).

**12.4. Tools.** One tool per registered command: name from §3, description
per §3.3, `inputSchema` from the pipeline schema (§9), `outputSchema` from
the result type when statically schematisable (absent otherwise).
Annotations map from the `MCPHints` annotation strings: `read` →
readOnlyHint true; `write` → readOnlyHint false; `destructive` →
destructiveHint true; `idempotent` → idempotentHint true; `openworld` →
openWorldHint true; `title:…` → title.

**12.5. Call result.** Success returns dual content — `textContent` rendered
by the *CLI renderer* (§9.1, trailing newline trimmed) **and**
`structuredContent` as the bare JSON value. Failure returns
`isError: true` whose text is the classified error's most specific message
(§8.4). MCP interface defaults only fill keys the caller did not supply
(§6.3).

**12.6. Server identity.** Name defaults to the binary basename; version to
`0.0.0`. Bearer/CORS configuration applies to the http transport (stdio is
local and MUST NOT be wrapped — emitting a warning note is the reference
behaviour).

---

## 13. Root dispatcher

**13.1. Mode words.** Default mode words: `serve` (HTTP), `mcp`, `help`.
They are reserved top-level names (§3.4), renamable through config, and
used verbatim by all library messages (no hard-coded words anywhere else).
Resolution rules: empty config field keeps the default; words must be plain
(no leading dash, no whitespace) and pairwise distinct; violations exit 2.

**13.2. Dispatch order** (fixed):

1. empty registry → silent no-op, exit 0;
2. `-v`/`--version` (before the `--` terminator) → `<bin> version <v>`,
   exit 0;
3. strip global `--xyz.*` built-ins (invalid values exit 2);
4. empty args / `help` / `--help` / `-h` → overview (mode list + command
   table; the table is omitted when CLI is disabled);
5. `serve` → HTTP mode; `mcp <transport>` → MCP mode; anything else → CLI
   mode.

**13.3. Built-in parameters.** Global namespace `--xyz.*` consumed anywhere
on the command line *before the `--` terminator* (tokens past `--` are
positional and are never stripped); inside `serve`/`mcp` mode the *mode word is the
namespace*, so bare names (`--addr`, `--bearer`, …) are equivalent. Renaming
a mode word migrates its namespace. Precedence: mode-local flag > global
flag / code config > library defaults. The table:

| Flag | Config field | Meaning |
|---|---|---|
| `--addr` | addr | default listen address for serve and mcp-http (`:8080`) |
| `--bearer=tok1,tok2` | bearer tokens | Bearer verification for serve REST and MCP http; empty = none; dedupe+append semantics |
| `--log-level=debug\|info\|warn\|error` | log level | stderr diagnostics level, default `info` |
| `--timeout=45s` | timeout | serve read/write/idle timeout; 0 = header timeout only |
| `--tls-cert/--tls-key` | cert file, key file | both set → TLS |
| `--cors=a,b` or `*` | cors origins | CORS allowlist |
| `--session-timeout=30m` | (mcp only) | idle-session expiry for streamable HTTP |

**13.4. Capability switches.** Runtime switches (no_cli / no_mcp / no_http)
disable a channel's runtime path only: mode words, `help`, `-v`, `completion`
keep working; entering a disabled mode prints a warning and exits 1; the
overview marks channels `（已禁用）`/`（本二进制未编译）`. Disabled
channels' config methods still compile (configuration is data, §2.3
corollary).

**13.5. Diagnostics.** All library diagnostics go to stderr with the
`xyz[level]:` prefix (levels debug/info/warn/error, default info, set via
code or `--xyz.log-level`). Command results and usage errors are never
subject to the log level.

**13.6. Trimmed builds.** Mechanisms that remove a channel from the binary
(Go build tags `nocli/nomcp/nohttp`; Rust cargo features `cli/http/mcp`)
MUST: keep the shell (§2.4) working; answer an invocation of a trimmed mode
with a clear "frontend not compiled" message, exit 1; note the trim in the
overview. The runtime capability switches (§13.4) remain available and
orthogonal.

**13.7. Cancellation.** The dispatcher owns one signal context
(SIGINT/SIGTERM) that reaches CLI/HTTP/MCP handlers; HTTP drains in-flight
requests before exit (reference grant 5 s); stdio MCP treats client
disconnect as normal exit 0. In HTTP, the *request's own* cancellation
(client disconnect) SHOULD additionally cancel the handler context (Go
does via `r.Context()`; Rust currently propagates only the serve-level
context — registered in deviations.md as D-rust-11).

**13.8. Version.** Version answer for `-v` is library-defined, overridable
per language's build mechanism (Go: ldflags; Rust: `set_version`).

---

## 14. Embedding surface

Every SDK MUST expose, independently of the one-line entry point:

1. **Explicit registry** — constructable registry; per-command
   `define(name, handler).register(&registry)`; sorted name/entry
   enumeration.
2. **Entry introspection** — the type-erased entry carries: name/summary/
   description, `input_schema` (and `output_schema` when derivable), the
   field tree (wire names, all bindings, all three default tiers), and
   `invoke(ctx, map)` callable by any adapter.
3. **CLI** — constructable app with injectable stdout/stderr and an
   execute-middleware hook (onion: outermost first, `next(args)` continues,
   short-circuit by not calling `next`).
4. **HTTP** — per-command handler (full binding + error mapping, mountable
   on the language's routers) and a whole-registry router (routes +
   `/healthz` + `/openapi.json`), plus reusable middleware pieces
   (Bearer/CORS/gzip).
5. **MCP** — a builder returning the official SDK's native server object so
   arbitrary SDK features may be added.
6. **Rendering** — the §9 renderer as a standalone function (JSON value in,
   rendered bytes out) shared by CLI and MCP.

Reference crossing points: Go's `registry.New` / `spec.Define(...)` /
`cli.NewWithOptions` + `App.Use` / `httpapi.HandlerFor` / `mcp.Server`;
Rust's `xyz_rust::Registry` / `spec::command::Command` /
`cli::App::new_with_options` + `use_mw` / `httpapi::handler_for` /
`mcp::handler::build`.

---

## 15. Experience conventions

**15.1. Distribution naming.** Repositories and package names use the
`<project>-<language>` scheme (`xyz-go`, `xyz-rust`; future `xyz-node`,
`xyz-java`, …). The imported symbol/package name SHOULD be the short form
(`xyz` in Go; crate `xyz-rust` in Rust) where the registry allows.

**15.2. Documentation.** Every SDK MUST ship a README in English (default)
and a Chinese mirror (`README.zh-CN.md`) cross-linked from the default, a
migration/adapters guide (`docs/adapters.md`) covering the three coexistence
levels (replace the frontend / mount per-command handlers / reuse pieces),
and a **deviations register** referencing deviations.md with section numbers
of this spec.

**15.3. Feature flags.** Channel trimming uses the language's native build
mechanism with channel names `cli`/`http`/`mcp` and the §13.6 invariants.
The core definition layer (vocabulary, schemas, errors, rendering) MUST NOT
require third-party dependencies beyond the language-standard JSON
implementation — the only sanctioned third-party tree is the official MCP
SDK (§12.1).

**15.4. Idiomatic entry points.** Each language offers its own constructor
flavour (Go: fluent `Define(...)...Run()` chain plus `Main/Run` functions;
Rust: `define(...)...run()` builder plus `main`/`run`/`run_config`) — the
normative content is the *set*: one-line main, versioned dispatch functions
returning exit codes, per-command registration, and config injection both
from code and from the command line.

---

## 16. Conformance

The conformance programme lives in [conformance.md](conformance.md):
a three-class checklist (MUST / SHOULD / MAY) plus golden-output scenarios
and the 11-command showcase fixture every SDK implements verbatim. An SDK
may claim conformance for a spec version after the checklist passes and its
deviations register is current.

## 17. Governance

**17.1.** The spec is versioned (semver): `v0.1.0` is the baseline
established from xyz-go v0.1.0 and xyz-rust 0.1.0. MAJOR bumps change
normative behaviour; MINOR adds normative surface; PATCH clarifies wording.
Each SDK states the spec version it targets.

**17.2.** Change process: proposal (issue/PR against this repository) with
the reference-implementation diff attached → agreement that every live SDK
can follow → spec bump → deviations registers updated.

**17.3.** Where two reference implementations disagree, this document
resolves it today by explicit ruling; residual language-forced divergences
live in [deviations.md](deviations.md) and MUST be re-reviewed at every
spec release.
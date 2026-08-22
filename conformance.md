# xyz Conformance Programme

Companion to [spec.md](spec.md) v0.1.1. An SDK claims conformance for a spec
version when:

1. every **Class A** item below passes (itemised tests or equivalent
   evidence), with the spec version recorded in the SDK repository;
2. the **showcase fixture** (§3) is implemented verbatim and its golden
   outputs (§3.3) match byte-for-byte where marked byte-exact;
3. the **feature matrix** (§4) passes for all six combinations;
4. a **deviations register** ([deviations.md](deviations.md) format) is
   current and every entry references a spec section.

---

## 1. Class A — MUST (normative, spec.md §2–§15)

### Identity & registration
- [ ] A.1 Command names enforced per spec §3.1 (grammar + 128-char cap).
- [ ] A.2 Dot separates CLI levels; full name is the MCP tool name (§3.2).
- [ ] A.3 Summary/description merge rule for MCP tool descriptions (§3.3).
- [ ] A.4 Reserved top-level mode words rejected at registration (§3.4).
- [ ] A.5 Duplicate names fail with a conflict-telling error (§3.5).
- [ ] A.6 Every definition error surfaces at registration, none at first
      invocation (§2.3).

### Field vocabulary & types
- [ ] A.7 Full §4.1 vocabulary implemented (wire name, exclude, desc,
      default, required, enum, validate, secret, CLI bindings, HTTP
      location, HTTP wire name).
- [ ] A.8 Hint merge semantics: non-zero hint overrides tag; filtered keys
      error at registration; defaults validated at registration (§4.2).
- [ ] A.9 All §4.3 required types decode from the three source forms
      (string / JSON shape / arbitrary JSON).
- [ ] A.10 Rejected types (maps, dynamic, channels, functions, recursion)
      fail at registration or earlier (§4.3).
- [ ] A.11 Lossless numeric conversion: non-integral→int, width overflow,
      negative→unsigned all error (§4.4).

### Validation & defaults
- [ ] A.12 Exact §5.1 rule set; unsupported rules fail at registration.
- [ ] A.13 Zero-value table (§5.2) drives required/omitempty correctly per
      type; nested recursion covers structs-in-slices and optionals (§5.3).
- [ ] A.14 Enum enforced at decode; error lists acceptable values (§5.4).
- [ ] A.15 Default precedence (§6.1) and the MCP schema-default
      replacement (§6.3) verified end-to-end.

### Pipeline & errors
- [ ] A.16 Pipeline order §7: bind → convert → enum → defaults → required →
      validate → handler; unknown keys ignored; JSON null = absent.
- [ ] A.17 Error taxonomy: all eight kinds; §8.2 channel mapping table
      (HTTP status / CLI exit / JSON-RPC code) exact.
- [ ] A.18 Classification walks the language error chain; uncoded ≠ internal
      (§8.3); handler classification survives language error wrapping.
- [ ] A.19 Channel error bodies carry the most specific cause message (§8.4).

### Rendering
- [ ] A.20 No envelope; §9.1 table exact for every row (bare scalars,
      KV pairs, tables with full padding, timestamps RFC 3339, canonical
      duration strings).
- [ ] A.21 CLI `--json` / HTTP results: two-space pretty JSON, trailing
      newline; HTTP errors compact `{"error":"…"}` + newline (§9.2/§9.3).

### CLI
- [ ] A.22 Tree, aliases and conflicts per §10.1 (aliases dispatch but are
      not listed).
- [ ] A.43 Default subcommand per §10.1: unmatched non-flag first
      argument forwards the entire argument list to the marked command;
      one default per parent (duplicate = registration error); empty
      argument lists, flags, explicit paths and aliases unaffected.
- [ ] A.23 Flag forms: `--name v`, `--name=v`, `-x v`, `-xv`, `-x=v`,
      boolean optional value, slice accumulation, `--` terminator (§10.2).
- [ ] A.24 Positional prefix rule enforced at registration (§10.3).
- [ ] A.25 `-h/--help`, `-v/--version`, `--json`, `completion
      bash|zsh|fish` semantics per §10.4; usage/flag failures exit 2
      (§10.5).
- [ ] A.26 Results to stdout, diagnostics to stderr with `xyz[level]:`
      prefix (§10.6).

### HTTP
- [ ] A.27 Route conflicts are registration errors; commands without hints
      are not routed (§11.1).
- [ ] A.28 Binding merge order and 1 MiB body cap; invalid JSON body =
      400 `{"error":"invalid JSON body"}` (§11.2).
- [ ] A.29 `/healthz` body is byte-exact `{"status":"ok"}` + `\n`;
      `/openapi.json` is a valid OpenAPI 3.0.3 doc from the same schemas
      (§11.3).
- [ ] A.30 Middleware order CORS → Bearer → Gzip and preflight-before-auth
      204 behavior (§11.4).

### MCP
- [ ] A.31 Uses the official MCP SDK of the language (§12.1).
- [ ] A.32 Revision list & `--versions` pinning (§12.2); missing transport
      rejects fast with a clear error, exit 2 (§12.3).
- [ ] A.33 Tool metadata: description merge, inputSchema, outputSchema,
      annotation mapping (§12.4).
- [ ] A.34 Dual content results (textContent = CLI renderer, stripped newline;
      structuredContent = bare JSON) and isError failures with specific
      message (§12.5).
- [ ] A.35 MCP defaults fill absent keys only; identity defaults
      (binary basename / `0.0.0`) (§12.5/§12.6).

### Dispatcher & platform
- [ ] A.36 Dispatch order §13.2 in full: empty registry no-op; `-v`
      before the `--` terminator only; built-in stripping stops at `--`;
      overview; mode routing.
- [ ] A.37 Built-in parameter table §13.3 (both `--xyz.*` and mode-local
      bare names; precedence correct).
- [ ] A.38 Capability switches: disabled mode warns and exits 1; shell
      survives; overview annotation (§13.4).
- [ ] A.39 Trimmed builds keep the shell; trimmed mode invocation prints a
      clear not-compiled message, exit 1 (§13.6); six-way feature matrix
      (§4 below) green.
- [ ] A.40 Signal-driven cancellation reaches handlers; HTTP drains
      in-flight requests (§13.7).
- [ ] A.41 Embedding surfaces §14 items 1–6 all present and documented.
- [ ] A.42 Bilingual READMEs cross-linked; adapters guide shipped (§15.2).

## 2. Class B — SHOULD

- [ ] B.1 Golden-output tests compare against §3.3 fixtures (byte-exact
      where marked).
- [ ] A.46 Command-level channel switches per §4.5a: CLI/HTTP/MCP skip bits
      remove the command from the marked channel only (no tree node / no
      route / no tool); overview keeps listing; CLI skip also drops
      aliases and completion words.
- [ ] A.47 Channel defaults per §6.1/§13.3: `--default k=v` (repeatable,
      comma-pairs, also `Config.ChannelDefaults`/`channel_defaults`) fills
      absent request/call keys only; values flow through the decode
      pipeline; invalid pairs are usage errors.
- [ ] A.48 Composable dispatch per §13.9: TryRun/try_run returns
      `(0, false)` silently for an unknown CLI top word (no command,
      alias, flag or default subcommand); all other paths identical to the
      main entry; CLI-skipped segments count as misses.
- [ ] A.49 Help flag type fidelity per §10.4: `-h` renders `string`/
      `integer`/`number`/`bool`/`duration`/`time`/`strings (repeatable)`
      by field type.
- [ ] A.45 Language (l10n) per §15.5: en + zh-CN bundled; identical canonical
      keys and English wording; --xyz.lang > Config > LANG/LC_ALL > en
      precedence; unknown values exit 2; per-language override table
      applies; unknown keys never panic.
- [ ] B.2 Core layers free of third-party dependencies beyond the
      language-standard JSON implementation (§15.3).
- [ ] B.3 Nested decode depth guard (reference value 20) (§4.3).
- [ ] B.4 Loopback-only Host default for streamable HTTP (§12.3).
- [ ] B.5 Library diagnostics never touch stdout; help texts follow the
      §10.4 layout including inline `(default …) (env …) (oneof …)` hints.

## 3. Showcase fixture

The canonical 11-command program every SDK ships verbatim (Go:
`cmd/example`; Rust: `examples/example`). Coverage checklist below is
normative; the program must be runnable with the exact commands of §3.3.

### 3.1 Commands and coverage

| Command | Covers |
|---|---|
| `user.add` | positional + shorthand + aliases + env-injected secret + enum + optional + slice + validate + CLI/HTTP/MCP three-tier defaults + annotations + usage line |
| `user.list` | `[]struct` → aligned table; HTTP GET route |
| `user.rm` | usage + alias + `not_found` business error + destructive annotation |
| `search.query` | global/CLI/MCP default layering (10/25/15) |
| `math.sum` / `math.div` | required scalars; bare int and float results; `invalid_input` on zero divisor |
| `time.now` | timestamp → RFC 3339 |
| `sys.sleep` | duration input; validation rejection beyond 5 s |
| `sys.port` | named scalar type |
| `file.hash` | bytes input; SHA-256 hex answer |
| `net.head` | HTTP header binding + `http_name` + env-only skip field + `unauthorized` |

### 3.2 Invocation matrix (runs on every SDK)

```
<app>                                   overview
<app> user add bob -a 25 -m fast --tags a,b
<app> user ua carol                     alias
<app> user rm alice                     ok; <app> user rm bob → not_found
<app> user list
<app> search query --query golang       CLI default k=25
<app> math sum --a 1 --b 2
<app> math div --a 10 --b 4             → 2.5; --b 0 → invalid_input, exit 2
<app> time now
<app> sys sleep --d 300ms
<app> sys port -p 9090
<app> file hash --data hello
NEXT_KEY=k <app> net head               env injection
<app> --json user list
<app> -v
<app> completion bash
<app> serve --addr :8080                curl matrix below
<app> mcp stdio                         MCP matrix below
```

### 3.3 Golden outputs

Output segments are **byte-exact** (including the final newline) for:

- `file hash --data hello` → `2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824`
- `math sum --a 1 --b 2` → `3`
- `math div --a 10 --b 4` → `2.5`
- `GET /healthz` body → `{"status":"ok"}` + LF

Multi-column output follows the §9.1 padding rule (every cell padded to its
column width, including the last column — trailing spaces are part of the
output and SHOULD be diff-tested). `user list` renders, when padding is
shown explicitly:

```
id  name   age  token_set
--  -----  ---  ---------
1   alice  18   false    
2   bob    25   false    
```

(rows as registered by the fixture; column widths derive from the visible
header/data cells, dashes repeat the width)

`user add bob -a 25 -m fast --tags a,b` → KV-aligned key/value block for
id/name/age/token_set.

`search query --query golang` → three lines `golang`, `...`, `top 25`.
(The `--query` spelling is significant: `--q` is an unknown-flag usage
error in both reference implementations.)

Errors (stderr):
- unknown flag → `unknown flag: --ghost`, exit 2;
- missing required → `field "b" is required`, exit 2;
- enum miss → message containing the acceptable values, exit 2;
- `user rm bob` → `user "bob" not found`, exit 1.

HTTP:
- `GET /healthz` → body `{"status":"ok"}` + LF, status 200 — byte-exact;
- `POST /users/alice -d '{"age":9}'` → pretty JSON object; then
  `GET /users` contains `"alice"`;
- `GET /openapi.json` → `"openapi":"3.0.3"`, paths contain
  `/users/{name}` with a `name` path parameter;
- Bearer off → 200; Bearer on without credentials → 401
  `{"error":"unauthorized"}` + `WWW-Authenticate: Bearer`;
- CORS preflight (allowlisted Origin, no credentials) → 204 before auth.

MCP (stdio, revision `2025-11-25`):
- `initialize` responds with negotiated `2025-11-25` and
  `serverInfo.version` `0.0.0`;
- `tools/list` → 11 tools sorted by name;
- `tools/call math.sum {"a":1,"b":2}` → textContent `3` + structuredContent
  `3`;
- `tools/call user.rm {"name":"bob"}` → `isError: true`, text
  `user "bob" not found`.

## 4. Feature matrix

Every build must pass tests in all nine channel combinations — the six that
remove a channel and the three aliases that keep it:

| Combination | Channels |
|---|---|
| default | CLI + HTTP + MCP |
| no mcp | CLI + HTTP |
| no cli | HTTP + MCP |
| no http | CLI + MCP |
| cli only | CLI |
| embedding only | — |

For each combination: unit/integration tests green, the trimmed-mode guard
(A.39) behaves as specified, and the overview annotates compiled/disabled
state correctly.

## 5. Evidence format

SDKs record conformance in their repository (e.g. `CONFORMANCE.md` or the
README section): spec version, checklist status (all items closed or a
pointer to the covering test), showcase run output archive, and the
deviations register link. New releases re-run §4 and re-attest the
checklist.
### 3.4 Extended golden scenarios (supplementary Class A evidence)

In addition to §3.3, SDK test suites SHOULD cover these rulings made after
the baseline (all verified byte-for-byte on both reference
implementations):

| Scenario | Expected behaviour |
|---|---|
| `user add -- -v` | `-v` is positional data: command output, not a version line |
| `user add -- --json` | `--json` is positional data |
| `--xyz.addr=:9` after `--` | not stripped from the argument list |
| `udf ./image.tar` with `extract` marked `default` | forwarded unchanged to `extract` (positional `./image.tar` receives it); `udf extract ./image.tar` unchanged; `udf -h` shows root help |
| validation `validate="gt=1.5"` with value 1.4 | fails with the rule; value 1.6 passes |
| aligned table with a CJK header/cell | byte-identical across SDKs (widths in bytes, padding in characters) |
| MCP `tools/call math.sum` | textContent is exactly `3` — no trailing newline |
| HTTP POST marked `application/json` with an unparseable body | 400 `{"error":"invalid JSON body"}`; the same body without a JSON declaration falls through to form binding |
| `SearchArgs` via `--query` (`--q` is an unknown flag) | `--query golang` returns `top 25` under CLI defaults |

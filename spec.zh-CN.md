# xyz 规范

**Version 0.1.0** · Status: Baseline · Last updated: 2026-08-21

English: [spec.md](spec.md)；冲突时以英文为准

本文档是每一个 xyz SDK 的规范性契约。一个库只有实现了本规范，才可自称
*xyz SDK*。关键词 **MUST**（必须）、**MUST NOT**（禁止）、**SHOULD**
（应当）、**SHOULD NOT**（不应）与 **MAY**（可以）按 RFC 2119 所述加以
解释。文中对章节编号的引用一律指本文档。

参考实现（歧义处的规范性锚点）：

| SDK | 路径 / 包 | 版本基线 |
|---|---|---|
| xyz-go | `github.com/ejfkdev/xyz-go`（包名 `xyz`） | v0.1.0 |
| xyz-rust | crates.io `xyz-rust`（库 `xyz-rust`；derive 辅助在 `xyz-rust-macros`） | 0.1.0 |

> 英文原版：[spec.md](spec.md)

---

## 1. 目的与范围

**1.1.** xyz 通过一条共享管线，把一次命令定义变成三种活的接口——CLI、
HTTP REST 服务与 MCP 工具服务器。熟悉一种语言里 xyz 的用户，必须能在
另一种语言里一眼认出它：相同的定义词汇、相同的派发模型、相同的错误、
相同的渲染输出。

**1.2.** 本规范钉死的内容（规范性）：

- *定义契约*（§3–§6）：名称、字段词汇表、类型、默认值、校验；
- *调用管线*（§7）：从传输层输入到渲染输出的唯一路径；
- *错误分类学*（§8）：驱动三个通道的唯一分类；
- *渲染契约*（§9）：无信封的输出形态；
- *前端*（§10 CLI、§11 HTTP、§12 MCP）与*根派发器*（§13）；
- *嵌入面*（§14）与*体验约定*（§15）；
- *一致性程序*（conformance.md）：每个 SDK 都必须通过。

**1.3.** 本规范有意留给各语言自行决定的内容（实现定义）：

- 错误、构建器与配置对象的惯用写法（见 §15）；
- JSON 实现与 HTTP 栈的选择——但有一条约束：MCP **MUST** 使用该语言
  *官方*的 Model Context Protocol SDK（§12.1）；
- 特性裁剪机制，映射到语言自身的构建系统上（§15.4）。

---

## 2. 范式

**2.1.** *一次定义。* 一条命令就是一个名称、一个参数类型、一个处理器，
以及若干按接口区分的提示。处理器收到一个已完全解码、已套用默认值并已
验证的参数值，外加一个取消上下文；它返回一个结果值或一个错误。

**2.2.** *唯一管线。* 每个前端把它的传输层输入归约为一个以字符串为键的
值 map，调用共享的 `Invoke`，再渲染结果或分类后的错误。每条定义恰有一
份解码、默认值、校验与执行的实现——前端绝不重新派生。管线顺序是规范性
的，见 §7。

**2.3.** *注册期即失败。* 任何定义错误——坏名称、坏字段词汇、不支持的
校验规则、路由冲突、保留的顶层名称——MUST 在命令注册时立即显现（或
更早，如编译期），绝不拖到首次调用。

**2.4.** *外壳不可裁剪。* 模式词、`help`、`-v/--version` 与 `completion`
在任何配置、任何裁剪后的构建里都可用（§13.6）。

---

## 3. 命令身份

**3.1.** 命令名 MUST 匹配 `^[A-Za-z0-9][A-Za-z0-9._-]{0,127}$`（最多 128
字符；首字符为字母数字；其余为字母数字或 `. _ -` 之一）。这与 MCP 工具
名兼容，同时也界定了 CLI 子命令段。

**3.2.** 点（`.`）分隔 CLI 子命令层级：`user.add` 命名子命令路径
`user add`。点不影响 HTTP 或 MCP（完整名即工具名 / 不用于路由）。

**3.3.** summary 是一行式描述。description 是较长的文本。MCP 工具描述在
`description` 为空时仅用 `summary`，在 `summary` 为空时仅用
`description`，否则为 `summary + "\n\n" + description`（§12.4）。

**3.4.** 已注册名称的顶层段 MUST NOT 与模式词（§13.1）冲突，否则注册
失败。

**3.5.** 注册重复名称 MUST 失败，并给出指明冲突的错误。

---

## 4. 参数定义

### 4.1 词汇表

字段声明在语言原生的 struct/class 上，用下列规范词汇作注解。左列是规范
概念；中间两列是两种参考拼写；右列是每个 SDK 都必须复现的语义。

| 概念 | xyz-go（struct tag） | xyz-rust（attribute） | 规范性语义 |
|---|---|---|---|
| 线上名 | `json:"user_name"` | `#[xyz(name = "user_name")]` | 线上名称（CLI flag、HTTP 参数/字段、MCP schema 键）。默认值 = 语言原生字段名。Rust 另以 serde rename 作为回退。 |
| 排除 | `json:"-"` | `#[xyz(skip)]` | 字段从绑定和生成的 schema 中排除。它 MAY 仍可按语言内部字段名接收注入值（env、HTTP header）（§4.4）。MUST NOT 与 `required` 组合。 |
| 描述 | `desc:"..."` | `#[xyz(desc = "...")]` | 人类可读描述；出现在 CLI help 与 JSON Schema 中。 |
| 全局默认值 | `default:"18"` | `#[xyz(default = "18")]` | `Invoke` 对缺失值应用的默认值；注册期按字段类型解析（解析失败 = 注册错误）。 |
| 必填 | `required:"true"` | `#[xyz(required)]` | 键必须存在（存在的零值同样满足在场要求，§7.3）。 |
| 枚举 | `enum:"a,b"` | `#[xyz(enum = "a,b")]` | 允许值，按字段类型解析。仅限标量；非标量枚举 = 注册错误。 |
| 校验 | `validate:"min=2,email"` | `#[xyz(validate = "min=2,email")]` | 校验规则集，见 §5。 |
| 机密 | `secret:"true"` | `#[xyz(secret)]` | 在帮助文本、日志与错误回显中对值脱敏。 |
| CLI 绑定 | `cli:"shorthand=a,positional,hidden,env=VAR,-"` | `#[xyz(cli = "shorthand=a,env=VAR")]` | §4.5。 |
| HTTP 位置 | `http:"query"` | `#[xyz(http = "query")]` | §4.6。 |
| HTTP 线上名 | `httpName:"X-Key"` | `#[xyz(http_name = "X-Key")]` | HTTP 传输的线上名覆盖（通常是 header 名）。 |

没有 struct tag/attribute 的语言（如 TypeScript decorator、Java 注解、
带具名参数的 Kotlin 属性）MUST 为同样的概念提供相应的拼写；语言允许时，
上述规范概念名 SHOULD 保留为标识符。

### 4.2 合并规则

命令级字段提示（按传输区分的配置 map，键为线上名*或*语言内部字段名——
后者不区分大小写）在 tag/attribute 值**之上**合并（覆盖）。*零值*的提示
字段保留 tag 值——因此可以在 tag 里设短名，只在构建器里覆盖默认值。键
指向未知字段或嵌套（带点的）字段 MUST 是注册错误。提示默认值在注册期
校验其可转换性。

### 4.3 类型

SDK MUST 支持以下参数字段类型：

| 类别 | Go 拼写 | Rust 拼写 | 备注 |
|---|---|---|---|
| 字符串 | `string` | `String` | |
| 布尔 | `bool` | `bool` | |
| 有符号整数 | `int, int8…int64` | `i8…i64` | 位宽检查的转换 |
| 无符号整数 | `uint, uint8…uint64` | `u8…u64` | 拒绝负数 |
| 浮点 | `float32, float64` | `f32, f64` | |
| 时间戳 | `time.Time` | `chrono::DateTime<Utc>` | 线上为 RFC 3339，JSON Schema `format: date-time` |
| 时长 | `time.Duration` | `std::time::Duration` | Go 风格时长字符串（`300ms`、`1.5h`、`2h45m`、`µs`），JSON Schema `format: duration` |
| 字节 | `[]byte` | `Vec<u8>` | 线上：字符串或字节数组；JSON Schema `string` |
| 切片 | `[]T` | `Vec<T>`（T ≠ u8） | |
| 可选/指针 | `*T` | `Option<T>` | null ≈ 缺失；nullable schema = 元素 schema |
| 嵌套结构体 | struct | struct | 内联进 schema（不用 `$defs`） |
| 具名标量 | `type Port int`（自动） | `#[derive(XyzField)]` newtype | 透明地表现为其底层标量 |

**MUST NOT** 被接受为参数字段：map、动态/interface 值、channel、函数、复数，
以及递归类型（直接或相互）。参考实现在注册期拒绝这些（或编译期——更早，
因此同样合规）。嵌套深度 SHOULD 作防御性限制（参考实现采用深度护栏 20）。

**4.4. 无损转换** —— 共享解码器接受三种来源形态：字符串（CLI）、JSON
形态（HTTP body）与任意 JSON（MCP）。数值形态之间的转换 MUST 无损：非
整数值（如 `3.7`）MUST NOT 无声息地变成整数；位宽溢出 MUST 报错；负数
转无符号 MUST 报错。布尔接受标准 true/false 字符串形态（
`1,t,T,TRUE,true,True` 及对应的 false 形态）。时间戳只接受 RFC 3339。

### 4.5 CLI 绑定（`cli:` 概念）

`shorthand=X` —— 单个字符；`positional` —— 消费下一个位置参数；
`hidden` —— 从 `--help` 列表中省略；`env=VAR` —— flag 未设置时回退到该
环境变量；`-` —— 对 CLI 前端不可见。校验：短名 MUST 恰好一个字符；同一
命令内重复的短名是注册错误；标记 `required` 的位置参数 MUST 构成位置参
数列表的前缀（必填位置参数出现在可选位置参数之后是注册错误）。

### 4.6 HTTP 位置（`http:` 概念）

`query`（未设置时的默认值）、`path`、`header`、`form`、`body` 之一。未知
位置是注册错误。`http_name` 覆盖线上名；优先级为 http_name > 线上名 >
语言字段名。

---

## 5. 校验

**5.1.** 支持的规则集恰好是：
`required, omitempty, min, max, len, gt, gte, lt, lte, oneof, email`——
go-playground/validator 兼容子集。规则语法：逗号分隔；数值规则带一个数
值参数（`min=2`）；`oneof` 带空格分隔的值（`oneof=a b`）。不支持或格式
错误的规则 MUST 在注册期失败。

**5.2.** 语义：

- `required` —— *零值*即失败。各类型的零值定义：字符串为空；数字 0
  （±0.0）；布尔 false；切片/字节为空；指针 nil/None；嵌套结构体当且仅
  当所有字段为零时为零。
- `omitempty` —— 零值跳过该字段的**所有**规则。
- `min`/`max`/`len` —— 字符串与切片比较长度；数值类型比较值。所有比较
  都在精确的 float64 空间进行（≤2^53 的整数无损；`gt=1.5` 之类的小数
  阈值按原样比较，绝不截断）。
- `gt`/`gte`/`lt`/`lte` —— 仅数值类型，float64 比较；其他类型一律不
  通过该规则。
- `oneof` —— 把值的显示形式与所列形式比较。
- `email` —— 仅字符串，匹配与 `^[^@\s]+@[^@\s]+\.[^@\s]+$` 等价的表达式。

**5.3.** 规则检查递归进入嵌套结构体，以及结构体切片/可选类型的元素。第
一个失败的规则产生字段错误
`invalid value for field "<wire name>": <rule>`，分类为 `invalid_input`
（§8）。

**5.4.** 枚举成员资格在解码期（而非校验期）强制；错误消息 MUST 包含可接
受的值。

---

## 6. 默认值分层

**6.1.** 单个字段的优先级（以 CLI 为例：各前端的模式一致）：

```
显式输入 > env 回退（仅 CLI）> 接口默认值 > 全局默认值 > 零值
```

**6.2.** 每个前端在调用 `Invoke` *之前*，把各自接口专属的默认值注入参数
map；随后 `Invoke` 对仍然缺失的键应用全局默认值。显式输入始终胜过这两
者。

**6.3.** 针对 MCP 的默认值还会替换生成的 `inputSchema` 中的 `default`
值（schema 是 MCP 工具的契约）。

**6.4.** 前端的接口默认值在入口上暴露（`CLIDefaults` / `HTTPDefaults` /
`MCPDefaults` 等价物），使适配器（§14）能注入完全一致的值。

---

## 7. 调用管线

`Invoke(ctx, args-map) -> rendered-agnostic result value`

规范性顺序：

1. **绑定与解码**：从 map 解码每个已声明字段（键 = 线上名；被排除/注入
   字段的键 = 语言字段名）。未知键被忽略。
2. **转换**：按 §4.4 的无损性转换；失败 = 分类为 `invalid_input` 的错误，
   并指明字段。
3. **枚举检查**：对转换后的值做枚举检查。
4. **缺失键**：应用全局默认值 → 否则 `required` 错误
   （`field "<name>" is required`）→ 否则零值。
5. **校验**：对补全后的值做校验（§5）。
6. **运行语言处理器**，携带取消上下文。
7. 把结果**原样**返回；面向线上的序列化发生在前端（§9）。

JSON `null` 输入在每一层都视为缺失。

---

## 8. 错误分类学

**8.1.** 种类（稳定标识符）：

| 种类 | 含义 |
|---|---|
| `invalid_input` | 参数格式错误、解码/校验失败 |
| `unauthorized` | 调用者必须认证 |
| `forbidden` | 已认证但权限不足 |
| `not_found` | 目标不存在 |
| `conflict` | 操作与现有状态冲突 |
| `canceled` | 调用者取消了操作 |
| `unavailable` | 依赖暂时不可用 |
| `internal` | 未分类失败的回退项 |

**8.2.** 通道映射（规范性）：

| 种类 | HTTP 状态码 | CLI 退出码 | JSON-RPC / MCP |
|---|---|---|---|
| `invalid_input` | 400 | 2 | -32602 |
| `unauthorized` | 401 | 1 | -32010 |
| `forbidden` | 403 | 1 | -32011 |
| `not_found` | 404 | 1 | -32001 |
| `conflict` | 409 | 1 | -32009 |
| `canceled` | 499（非标准，已在文档注明） | 1 | -32012 |
| `unavailable` | 503 | 1 | -32603 |
| `internal` | 500 | 1 | -32603 |

**8.3.** 分类沿语言的错误链（cause/source）行走：找到的第一个带码分类获
胜；未带码的非 nil 错误归为 `internal`。SDK MUST 允许用户处理器返回自有
且合规的错误（Go：SDK 导出 `errs.New(kind, ...)`；Rust：
`errs::new(kind, ...)`），并且在语言包装错误时 MUST 保留分类。

**8.4.** HTTP 与 MCP 通道上的错误携带*最具体*的 cause 消息（最内层的已
知原因，而非包装层）。

---

## 9. 渲染（无信封）

**9.1.** 结果绝不包信封（没有 `{"data": ...}`）。人工通道（CLI）按如下
方式渲染：

| 结果 | CLI 渲染 |
|---|---|
| null / 缺失 | 无输出 |
| 字符串 | 原始字符串（单行） |
| 布尔 / 数字 | 裸值 |
| 时间戳 | RFC 3339 字符串 |
| 时长 | 规范时长字符串（`300ms`、`1.5s`、`1h2m3.5s`；亚秒值以毫秒表示） |
| 标量切片 | 每行一个元素 |
| 结构体 | 对齐的 `key  value` 键值对（两空格间距，键 = 线上名） |
| 结构体切片 | 对齐表格：表头行、短横分隔行、数据行；每个单元格按所在列宽补全（含最后一列） |
| map | 对齐的键值对 |

精确的单元格格式（§9.4）——包括整数值浮点的显示方式（`3` 而非
`3.0`）与尾部补全——是契约的一部分，目标是各 SDK 的 CLI 逐字节一致。每
个 SDK SHOULD 附带 golden-output 测试，与一致性 fixtures
（conformance.md）比对。

**9.2.** JSON 模式（CLI `--json`、HTTP 响应、MCP `structuredContent`）把
同一结果序列化为裸 JSON 值：作为文档片段渲染时（CLI/HTTP）采用两空格缩
进并以换行结尾；MCP structured content 就是 JSON 值本身。

**9.3.** HTTP 错误体为紧凑（单行）的 `{"error":"<message>"}` + 换行；
HTTP 结果体为美化打印的裸 JSON + 换行。成功响应使用
`Content-Type: application/json; charset=utf-8`。

**9.4.** 线上的字段顺序遵循声明顺序（参考实现保持声明顺序；map 同样按
语言的序列化顺序渲染）。浮点显示采用语言数字格式化所能给出的最短表示
（`2.5`、`3`）。

注（语言造成的分歧，登记于 deviations.md）：没有结构反射的语言在序列化
后可能无法区分结构体值与 map 值；两者都渲染成键值对，而这本来就是可观
察的契约。

---

## 10. CLI 前端

**10.1. 命令树。** 带点的名称成为嵌套子命令。就派发而言，别名与子命令
名等同，但不列入父级的 help。注册的路径与既有叶子或节点冲突，或别名与
兄弟节点冲突，MUST 在注册期失败。

**10.2. Flags。** 长 flag 用线上名（`--name value`、`--name=value`）；短
名经 `-x value`、`-xvalue`、`-x=value`；布尔不带参数，也可带显式的
`=bool`；切片 flag 在多次出现时累积；`--` 终止 flag 解析，剩余部分成为位
置参数。未知 flag 是使用错误（见 10.5）。

**10.3. 位置参数。** 按声明顺序绑定；`required` 位置参数构成前缀；数量
必须落在 `[min, max]` 内，超出则为说明期望数量与实际数量的使用错误（参
考消息：`<path>: 位置参数数量不符（需要 a 到 b 个，收到 n 个）`——措辞允
许本地化，结构必须一致）。

**10.4. 内建。** `-h/--help` 打印最深匹配节点的帮助（父级列出子级）；
`-v/--version` 打印 `<bin> version <version>` 并退出 0；`--json` 把结果
渲染切换为 JSON；`--` 终止符（§10.2）同时终止 `-v`、`--version` 与
`--json` 的识别——其后的 token 一律是位置数据，不再是开关；
`completion bash|zsh|fish` 为二进制名生成可用的补全脚本；未知 shell 退出
2。帮助布局
顺序：description → `Usage:` → 可选 `Aliases:` → `命令:`/`Flags:` →
`Global Flags:` / 辅助行，内联提示 `(default …)`、`(env …)`、
`(oneof a|b)` 织入 flag 描述。

**10.5. 退出码。** 成功为 0。命令失败：分类学的码（§8.2）。使用错误（未
知 flag、缺少参数、位置参数数量不符）：2。启动时报告的注册错误：2。
`validate`/解码失败属于 `invalid_input` → 2。

**10.6. 输出契约。** 命令结果到 stdout；错误与诊断到 stderr。诊断携带
`xyz[level]:` 前缀（日志级别经全局配置设置，§13.5）；默认级别为 `info`。

---

## 11. HTTP 前端

**11.1. 路由。** 带 HTTP 提示的命令定义 `METHOD path` 路由；`{name}` 是
单段路径参数。method+path 冲突配对 MUST 是注册错误。无 HTTP 提示的命令
不参与路由。

**11.2. 绑定。** 构建参数 map 的校验顺序：接口默认值（基底）→ JSON body
合并（GET/HEAD 以外的方法；读取上限 SHOULD 为 1 MiB；body 不可解析 = 400
`{"error":"invalid JSON body"}`）→ 逐字段：路径参数（线上名）、query 值
（默认位置；切片字段收集全部重复值，其他字段取第一个）、headers（名称
按 §4.6）、form 字段（仅取请求体）。JSON 解析失败的请求体只有在该请求**声明**了
`Content-Type: application/json` 时才判 400；未声明的体只是不参与合并。被 §4.1 排除的字段按键为语言字段名
接收 header 值。

**11.3. 内建端点**（每个 HTTP 前端 MUST 存在）：

- `GET /healthz` → 200，body 恰为 `{"status":"ok"}` + 换行。
- `GET /openapi.json` → 由同一批 `inputSchema` 生成的 OpenAPI 3.0.3 文档；
  每个操作的 summary/parameters（path+query）/requestBody
  （POST/PUT/PATCH）/responses（200 携带输出 schema 内容，外加分类学中的
  400/404/500 描述）。

**11.4. 中间件。** 服务的层级，从最外层开始，按此固定顺序：CORS →
Bearer → Gzip → router。CORS：允许的 origin 列表（或 `*`）；OPTIONS 预检
在**认证之前**以 204 应答（浏览器预检不带凭据），精确匹配时回显
`Access-Control-Allow-Origin`（+`Vary: Origin`）。Bearer：
`Authorization: Bearer <tok>` 必须命中配置的 token 集——否则 401
`{"error":"unauthorized"}` + `WWW-Authenticate: Bearer`；token 集为空 = 无
认证。Gzip：只要客户端发送 `Accept-Encoding: gzip` 就压缩响应，无论响应
大小。

**11.5. 服务器**，配置取自全局配置：`addr`（serve 与 mcp-http 默认
`:8080`）、read/write/idle 超时（0 = 仅 header 超时）、cert+key 同时给出
则启用 TLS、取消时优雅排空（参考宽限：5 s）。

---

## 12. MCP 前端

**12.1.** SDK MUST 建立在**本语言的官方 Model Context Protocol SDK** 之
上（Go：`github.com/modelcontextprotocol/go-sdk`；Rust：`rmcp`）。不允许
自研协议实现。

**12.2. 协议修订。** 规范性修订列表，新在前：`2026-07-28`、
`2025-11-25`、`2025-06-18`、`2025-03-26`、`2024-11-05`。SDK 支持其官方
SDK 所支持的集合，并按偏好顺序宣告；`--versions` 固定一个子集（未知/空
条目 = 注册错误）。

**12.3. 传输。** `mcp stdio`（所有修订）与 `mcp http`（streamable
HTTP）。传输可用性跟随本地官方 SDK：当某传输在那里不存在时（参考例：官
方 Rust SDK 在 2026-07-28 修订中移除了旧式 HTTP+SSE），
`mcp <transport>` MUST 快速失败，以指明该传输的清晰错误并以退出码 2 退
出——绝不静默降级。SDK 若确实提供 SSE（Go SDK），`mcp sse` 服务 ≤
`2025-11-25` 的修订；以 streamable HTTP 服务 `2026-07-28` 需要
无状态模式。Streamable HTTP 默认 SHOULD 把 `Host` 限制在 loopback，除非
显式放宽配置（Rust SDK 即如此，以防 DNS rebinding）。

**12.4. 工具。** 每条注册命令一个工具：名称见 §3，描述依 §3.3，
`inputSchema` 来自管线 schema（§9），`outputSchema` 来自可静态模式化的结
果类型（否则缺省）。注解由 `MCPHints` 注解字符串映射而来：`read` →
readOnlyHint true；`write` → readOnlyHint false；`destructive` →
destructiveHint true；`idempotent` → idempotentHint true；`openworld` →
openWorldHint true；`title:…` → title。

**12.5. 调用结果。** 成功返回双重内容——由 *CLI 渲染器*渲染的
`textContent`（§9.1，修剪末尾换行）**以及**作为裸 JSON 值的
`structuredContent`。失败返回 `isError: true`，其文本是分类错误的最具体
消息（§8.4）。MCP 接口默认值只填充调用者未提供的键（§6.3）。

**12.6. 服务器身份。** 名称默认为二进制的 basename（不含路径的文件名）；
版本默认为 `0.0.0`。Bearer/CORS 配置适用于 http 传输（stdio 是本地通道，
MUST NOT 被包裹——发出警告注记是参考行为）。

---

## 13. 根派发器

**13.1. 模式词。** 默认模式词：`serve`（HTTP）、`mcp`、`help`。它们是保
留的顶层名称（§3.4），可经配置改名，并为所有库消息原样使用（除此之外
任何地方都不得硬编码这些词）。解析规则：配置字段为空则保留默认值；词必
须朴素（无前导短横、无空白）且两两互异；违反则退出 2。

**13.2. 派发顺序**（固定）：

1. 空注册表 → 静默无操作，退出 0；
2. `--` 终止符之前的 `-v`/`--version` → `<bin> version <v>`，退出 0；
3. 剥离全局 `--xyz.*` 内建参数（非法值退出 2）；
4. 空参数 / `help` / `--help` / `-h` → 概览（模式列表 + 命令表；CLI 被禁
   用时不显示命令表）；
5. `serve` → HTTP 模式；`mcp <transport>` → MCP 模式；其余 → CLI 模式。

**13.3. 内建参数。** 全局命名空间 `--xyz.*` 可在命令行任意位置（`--`
终止符之前）消费；在
`serve`/`mcp` 模式内，*模式词即命名空间*，因此裸名称（`--addr`、
`--bearer`、……）等价。重命名模式词即迁移其对应命名空间。优先级：模式
局部 flag > 全局 flag / 代码配置 > 库默认值。表如下：

| Flag | 配置字段 | 含义 |
|---|---|---|
| `--addr` | addr | serve 与 mcp-http 的默认监听地址（`:8080`） |
| `--bearer=tok1,tok2` | bearer tokens | serve REST 与 MCP http 的 Bearer 校验；空 = 无；去重+追加语义 |
| `--log-level=debug\|info\|warn\|error` | log level | stderr 诊断级别，默认 `info` |
| `--timeout=45s` | timeout | serve 的 read/write/idle 超时；0 = 仅 header 超时 |
| `--tls-cert/--tls-key` | cert 文件、key 文件 | 两者都设置 → TLS |
| `--cors=a,b` 或 `*` | cors origins | CORS 允许列表 |
| `--session-timeout=30m` | （仅 mcp） | streamable HTTP 的空闲会话过期时间 |

**13.4. 能力开关。** 运行时开关（no_cli / no_mcp / no_http）只禁用相应
通道的运行时路径：模式词、`help`、`-v`、`completion` 继续工作；进入被禁
用的模式打印警告并退出 1；概览把通道标记为
`（已禁用）`/`（本二进制未编译）`。被禁用通道的配置方法仍然可以编译（配
置即数据，§2.3 推论）。

**13.5. 诊断。** 所有库诊断到 stderr，前缀 `xyz[level]:`（级别
debug/info/warn/error，默认 info，经代码或 `--xyz.log-level` 设置）。命令
结果与使用错误绝不受日志级别约束。

**13.6. 裁剪构建。** 把某通道从二进制中移除的机制（Go build tag
`nocli/nomcp/nohttp`；Rust cargo feature `cli/http/mcp`）MUST：保持外壳
（§2.4）可用；对裁剪模式的调用给予清晰的 "frontend not compiled" 消息并
退出 1；在概览中注明裁剪。运行时能力开关（§13.4）保持可用且彼此正交。

**13.7. 取消。** 派发器拥有一个信号上下文（SIGINT/SIGTERM），送达
CLI/HTTP/MCP 处理器；HTTP 在退出前排空在途请求（参考宽限 5 s）；stdio
MCP 把客户端断开视为正常退出 0。

**13.8. 版本。** `-v` 的版本应答由库定义，可按语言的构建机制覆盖（Go：
ldflags；Rust：`set_version`）。

---

## 14. 嵌入面

每个 SDK MUST 在独立于一行式入口之外，暴露：

1. **显式注册表** —— 可构造的注册表；每命令
   `define(name, handler).register(&registry)`；有序的名称/条目枚举。
2. **入口自省** —— 类型擦除的入口携带：name/summary/description、
   `input_schema`（以及可派生时的 `output_schema`）、字段树（线上名、全
   部绑定、全部三层默认值），以及可被任何适配器调用的
   `invoke(ctx, map)`。
3. **CLI** —— 可构造的 app，注入 stdout/stderr 与执行中间件钩子（洋葱：
   最外层先执行，调用 `next(args)` 继续，不调用 `next` 即短路）。
4. **HTTP** —— 每命令处理器（完整绑定 + 错误映射，可挂到语言的路由器
   上）与整注册表路由器（路由 + `/healthz` + `/openapi.json`），外加可复
   用的中间件（Bearer/CORS/gzip）。
5. **MCP** —— 返回官方 SDK 原生服务器对象的构建器，使任意 SDK 特性可被
   添加。
6. **渲染** —— §9 渲染器作为独立函数（JSON 值进，渲染字节出），由 CLI
   与 MCP 共享。

参考对接点：Go 的 `registry.New` / `spec.Define(...)` /
`cli.NewWithOptions` + `App.Use` / `httpapi.HandlerFor` / `mcp.Server`；
Rust 的 `xyz_rust::Registry` / `spec::command::Command` /
`cli::App::new_with_options` + `use_mw` / `httpapi::handler_for` /
`mcp::handler::build`。

---

## 15. 体验约定

**15.1. 分发命名。** 仓库名与包名使用 `<project>-<language>` 方案
（`xyz-go`、`xyz-rust`；未来 `xyz-node`、`xyz-java`、……）。注册处允许时，
导入的符号/包名 SHOULD 采用短形式（Go 中为 `xyz`；Rust 中为 crate
`xyz-rust`）。

**15.2. 文档。** 每个 SDK MUST 附带英文 README（默认）与其中文镜像
（`README.zh-CN.md`，从默认版交叉链接）、迁移/适配器指南
（`docs/adapters.md`，覆盖三级共存：替换前端 / 挂载每命令处理器 / 复用部
件），以及一份**差异登记表**（引用 deviations.md 并附本规范章节号）。

**15.3. 特性开关。** 通道裁剪使用语言原生构建机制，通道名为
`cli`/`http`/`mcp`，并满足 §13.6 的不变量。核心定义层（词汇表、schema、
错误、渲染）MUST NOT 要求语言标准 JSON 实现之外的第三方依赖——唯一被认
可的第三方依赖树是官方 MCP SDK（§12.1）。

**15.4. 惯用入口。** 每种语言提供各自的构造器风格（Go：流式
`Define(...)...Run()` 链，外加 `Main/Run` 函数；Rust：
`define(...)...run()` 构建器，外加 `main`/`run`/`run_config`）——规范性内
容在于*集合*：一行式 main、返回退出码的带版本派发函数、逐命令注册，以
及既可从代码也可从命令行注入配置。

---

## 16. 一致性

一致性程序位于 [conformance.md](conformance.md)：一份三级检查单（MUST /
SHOULD / MAY）、golden-output 场景，以及每个 SDK 都逐字实现的 11 命令展
示 fixture。某 SDK 在检查单通过且其差异登记表为最新后，方可对某一规范版
本宣称一致性。

## 17. 治理

**17.1.** 规范按语义化版本（semver）版本化：`v0.1.0` 是由 xyz-go v0.1.0
与 xyz-rust 0.1.0 确立的基线。MAJOR 变更改变规范行为；MINOR 增加规范面；
PATCH 澄清措辞。每个 SDK 声明其目标规范版本。

**17.2.** 变更流程：提案（对本仓库提交 issue/PR），附参考实现的 diff →
就每个在用 SDK 都能跟进达成一致 → 提升规范版本 → 更新差异登记表。

**17.3.** 两个参考实现意见相左时，本文档以明示裁决即刻解决；剩余的语言
造成的分歧登记于 [deviations.md](deviations.md)，且 MUST 在每次规范发布
时复审。
# xyz 规范

**Version 0.4.5** · Status: Baseline · Last updated: 2026-10-06

English: [spec.md](spec.md)；冲突时以英文为准

本文档是每一个 xyz SDK 的规范性契约。一个库只有实现了本规范，才可自称
*xyz SDK*。关键词 **MUST**（必须）、**MUST NOT**（禁止）、**SHOULD**
（应当）、**SHOULD NOT**（不应）与 **MAY**（可以）按 RFC 2119 所述加以
解释。文中对章节编号的引用一律指本文档。

参考实现（歧义处的规范性锚点）：

| SDK | 路径 / 包 | 版本基线 |
|---|---|---|
| xyz-go | `github.com/ejfkdev/xyz-go`（包名 `xyz`） | v0.4.5（本规范） |
| xyz-rust | crates.io `xyz-rust`（库 `xyz-rust`；derive 辅助在 `xyz-rust-macros`） | 0.4.4 → spec v0.4.4（spec v0.4.5 条款待补） |

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
`user add`。点不影响 HTTP 或 MCP 路由：默认完整名即 MCP 工具名；可按命令
覆写（§12.4a）。

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

### 4.4 无损转换

共享解码器接受三种来源形态：字符串（CLI）、JSON
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

**4.5a. 命令级通道开关。** 三个 hints 各自 MAY 携带排除位——Go：
`CliHints{Skip}`、`HTTPHints{Skip}`、`MCPHints{Skip}`；Rust：
`CliHints { skip }`、`HTTPHints { skip }`、`MCPHints { skip }`。语义：
被标记的通道完全不消费该命令——CLI 不建子命令节点、别名不生效、不进
completion 词（比 `hidden` 更强——hidden 只藏帮助、仍可执行）；HTTP 不注册
路由、不出现在 `/openapi.json`；MCP 不成为工具。注册表与总览照常列出该命令
（注册是全局的，消费是分通道的）。

**守护命令。** 命令 MAY 标记 `Daemon`（Go：`CliHints{Daemon: true}`；Rust：
`CliHints { daemon: true }`）声明长驻生命周期：handler 阻塞到上下文取消，
届时 CLI 优雅退出 0。语义：标记隐含 CLI-only 消费（HTTP/MCP 视同 skip
排除）、CLI 不渲染返回值、handler 的分类错误照常映射退出码。

### 4.6 HTTP 位置（`http:` 概念）

`query`（未设置时的默认值）、`path`、`header`、`form`、`body` 之一。未知
位置是注册错误。`http_name` 覆盖线上名；优先级为 http_name > 线上名 >
语言字段名。

### 4.7 带标签联合（`oneOf`）

SDK MAY 把枚举类型的参数字段当作**邻接带标签联合**接受：序列化携带显式
判别键的枚举（serde 的 `#[serde(tag = "…")]` 形态，或语言原生等价物）映射
为各变体 schema 的 `inputSchema` `oneOf`。每个变体分支 MUST 注入判别属性，
并把变体名固化为唯一允许值（JSON-Schema `const`），客户端据以分派。无标签
或内部标签的枚举 MUST 在定义期拒绝——线上形态有歧义、无法往返。一个值在
**恰好一个**分支匹配时解码成功；零个或多个匹配都是 `invalid_input`。判别键
在三个前端都是一等参数选择（CLI flag、HTTP 字段、MCP 参数）。无法原生表达
联合参数的前端 MAY 在该通道跳过该字段（命令保持注册、其余字段可用），
而不是让注册失败；被跳过的字段在可选（`Option<…>`）时表现为缺席，
在标注 required 时表现为必填。

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
显式输入 > env 回退（仅 CLI）> 接口默认值 > 通道默认 > 全局默认值 > 零值
```

*通道默认*这一层在 serve/mcp 启动时经 `--default key=value` 注入
（可重复；逗号分隔对；亦即 Go `Config.ChannelDefaults` / Rust
`Config.channel_defaults`），只补缺席键——绝不覆盖显式输入与接口默认。
值走常规解码管线（数值通道默认按字段类型解析）。在 serve 与 mcp 模式，
任何未识别的 `--key value` / `--key=value` flag 同样透传为通道默认
（`gs serve --index ./wiki` 即 `--default index=./wiki` 的顺手写法）；
无值的悬空 `--key` 是用法错误。

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

**8.5. 富化错误上下文（可选层）。** 除 Kind 外，一个已分类错误 MAY 携带
三个可选层，纯语言原生错误（最简的 handler 错误——`errors.New`/
`fmt.Errorf`、`anyhow!`/`std::io::Error`——归类为 `internal`，仅以自身消息
渲染）一个都不需要提供：

| 层 | 类型 | 作用 |
|---|---|---|
| `code` | 自由字符串 | 领域标识符（`USER_NOT_FOUND`、`QUOTA_EXCEEDED`），原样送达调用方；它绝不影响传输映射（那是 Kind 的职责），让客户端无需解析消息即可按领域语义分支 |
| `detail` | 键值映射 | 结构化上下文，原样渲染进机器可读错误体 |
| `status` | 整数 | 覆盖本错误的 §8.2 HTTP 状态码（仅 HTTP 通道）；0/缺省 = 由 Kind 派生 |

SDK MUST 提供符合人体工学、可组合的方式来附加它们（Go：
`errs.NotFound("user %s", id).WithCode("USER_NOT_FOUND").WithDetail("user_id", id).WithStatus(410)`；
Rust：等价的 builder）。附加它们 MUST NOT 改变 §8.2 的 Kind 映射，显式
`status` 覆盖除外。

**8.6. 共享错误体。** 错误的机器可读形态是每个通道共用的同一个对象，使
HTTP 体、CLI 机器模式 stderr 与 MCP `_meta` 认同一个形状：

```json
{"error":"<message>","kind":"<kind>","code":"<code>","detail":{…}}
```

`error` 是最具体的 cause 消息（§8.4），恒在；`kind` 恒在；`code` 与
`detail` 仅在设置时出现。扁平的 `error` 字符串键逐字保留自 0.4.2 之前的契约
（无 code/detail 的错误体恰为 `{"error":"<message>"}`），故既有客户端照常
工作，更富的键纯属增补。逐通道交付：HTTP 以紧凑响应体写出（§9.3）；CLI
在机器格式（§10.7）下写到 **stderr**（stdout 绝不承载错误）；MCP 把人类
消息留在 `textContent`，把 `kind`/`code`/`detail` 附在结果 `_meta.xyz.error`
下（§12.8）。

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

**9.3.** HTTP 错误体为 §8.6 的共享错误对象，紧凑（单行）+ 换行——无
code/detail 的错误即 `{"error":"<message>"}`（与 0.4.2 之前的契约逐字节一
致），错误携带时再增补 `kind`/`code`/`detail` 键。HTTP 结果体为美化打印
的裸 JSON + 换行。成功响应使用 `Content-Type: application/json;
charset=utf-8`。

**9.4.** 线上的字段顺序遵循声明顺序（参考实现保持声明顺序；map 同样按
语言的序列化顺序渲染）。浮点显示采用语言数字格式化所能给出的最短表示
（`2.5`、`3`）。对齐渲染以**字节计算列宽、以字符填充**（两个参考实现的
混合语义——CJK 文本出现在表头或单元格时，表格仍需逐字节一致）。

注（语言造成的分歧，登记于 deviations.md）：没有结构反射的语言在序列化
后可能无法区分结构体值与 map 值；两者都渲染成键值对，而这本来就是可观
察的契约。

**9.5. 逐通道输出函数（可选）。** SDK MAY 允许一条命令用逐通道输出函数
覆盖它在某个或某些通道上的渲染（Go：`CliHints.Output` /
`HTTPHints.Output` / `MCPHints.Output`；缺省 = 本节的默认渲染）。这是
*可选*特性：不提供它的 SDK 仍然合规，未设输出函数的命令在各处渲染完全
一致。提供时，三通道优先级相同——

> 机器/替代格式标志（§10.7 `--format json|jsonl|markdown`，HTTP 恒为机器）
> **>** 逐通道输出函数 **>** §12.7 块信封投影 **>** 默认渲染

——且错误路径（§8）绝不经过输出函数：错误分类与 §8.6 错误体是框架专属。
CLI 函数全权负责人类可读渲染（富文本、彩色、分页）；HTTP 函数全权负责
状态码/响应头/响应体；MCP 函数全权负责 `textContent`，而
`structuredContent` 仍由框架生成。本条吸收偏差 D-go-03（见 deviations.md）。

---

## 10. CLI 前端

**10.1. 命令树。** 带点的名称成为嵌套子命令。就派发而言，别名与子命令
名等同，但不列入父级的 help。注册的路径与既有叶子或节点冲突，或别名与
兄弟节点冲突，MUST 在注册期失败。

**默认子命令。** 命令 MAY 被标记为其父节点的*默认子命令*（Go：
`CliHints{Default: true}`；Rust：`CliHints { default: true }`）。当首段
参数匹配不到任何已注册命令段、且不以 `-` 开头（从而 `-h`、`-v`、
`--json` 等永远不被吞掉）时，整串剩余参数不消费地转发给默认子命令：
`extract` 为默认时，`udf ./image.tar` 等价于 `udf extract ./image.tar`。
语义：

- 每个父节点至多一个默认子命令；第二个标记是注册期错误；
- 空参数列表不触发默认（总览/帮助停留在未匹配节点）；
- 显式路径、别名与下沉后的 `-h` 表现得与显式写出默认命令完全一致；
- 该特性仅作用于 CLI，且适用于树的任意层级（嵌套节点可各自声明默认）。

**10.2. Flags。** 长 flag 用线上名（`--name value`、`--name=value`）；短
名经 `-x value`、`-xvalue`、`-x=value`；布尔不带参数，也可带显式的
`=bool`；切片 flag 在多次出现时累积；`--` 终止 flag 解析，剩余部分成为位
置参数。未知 flag 是使用错误（见 10.5）。

**10.3. 位置参数。** 按声明顺序绑定；`required` 位置参数构成前缀；数量
必须落在 `[min, max]` 内，超出则为说明期望数量与实际数量的使用错误（参
考消息：`<path>: 位置参数数量不符（需要 a 到 b 个，收到 n 个）`——措辞允
许本地化，结构必须一致）。

**10.4. 内建。** `-h/--help` 打印最深匹配节点的帮助（父级列出子级）；
`-v/--version` 打印 `<app-name> version <app-version>` 并退出 0；`--json`
把结果渲染切换为 JSON；`--` 终止符（§10.2）同时终止 `-v`、`--version` 与
`--json` 的识别——其后的 token 一律是位置数据，不再是开关；
`completion bash|zsh|fish` 为二进制名生成可用的补全脚本；未知 shell 退出
2。`help` 模式还可带参数——`help <命令路径>` 打印该命令的详细帮助（与
`<路径> -h` 相同）、`help <模式>` 打印该模式的帮助——且每个服务模式对
`-h`/`--help` 以自身帮助应答而不起服务（派发见 §13.2）。补全词表必须包含
顶层命令词、shell 词（`completion`、`help`、`-h`、`--help`、`-v`、
`--version`）与模式词（语言允许时跟随配置）。帮助布局
顺序：description → `Usage:` → 可选 `Aliases:` → `命令:`/`Flags:` →
`Global Flags:` / 辅助行，内联提示 `(default …)`、`(env …)`、
`(oneof a|b)` 织入 flag 描述。`-h` 里的 flag 类型标注 MUST 反映字段
类型：`string`、`integer`、`number`、`bool`、`duration`、`time` 与
`strings (repeatable)`（切片走重复 flag；逗号分隔**不**切分）。

**自定义帮助块。** 叶子命令 MAY 携带两段原样文本块——`before`（插在 `-h`
输出的最前、description 之前）与 `after`（插在最末、`Global Flags` 之后）
——Go：`CliHints{Before, After}`；Rust：`CliHints { before, after }`。块
按作者原样输出（多行与缩进自控），末尾多余换行归一为一个；空块 MUST 是
零操作（字节级一致）。中间节点没有 hints，永不打印块。不规定任何命名块
种类——示例、版本行、仓库地址等一切内容都是用户自己的文本。

**10.5. 退出码。** 成功为 0。命令失败：分类学的码（§8.2）。使用错误（未
知 flag、缺少参数、位置参数数量不符）：2。启动时报告的注册错误：2。
`validate`/解码失败属于 `invalid_input` → 2。

**10.6. 输出契约。** 命令结果到 stdout；错误与诊断到 stderr。诊断携带
`xyz[level]:` 前缀（日志级别经全局配置设置，§13.5）；默认级别为 `info`。

**10.7. 输出格式（`--format`）。** 全局标志
`--format <auto|text|json|jsonl|markdown>` 选择结果渲染；`--json` 是
`--format json` 的向后兼容别名。默认为 `auto`（TTY 感知，见下）。各格式：

| `--format` | 渲染 |
|---|---|
| `auto`（默认） | 按 stdout 是否交互式终端解析——见下 |
| `text` | §9.1 人类可读渲染；走 §9.5 链（自定义输出 → 块投影 → 默认渲染） |
| `json` | 美化 JSON（§9.2），裸值，两空格缩进 |
| `jsonl` | JSON Lines：切片/数组结果每个元素一行**紧凑** JSON；其余结果整体一行紧凑 |
| `markdown` | 结果以 Markdown 呈现：结构体 → 两列 `\| Field \| Value \|` 表；结构体切片 → 以字段为列的表；标量切片 → `- 元素` 无序列表；map → `\| Key \| Value \|` 表（字符串键排序）；标量裸出；单元格转义 `\|`→`\\|`、换行→`<br>` |

**TTY 感知默认（`auto`）。** `auto` 在渲染时按结果输出目标是否为交互式
终端（TTY）解析：交互式 → *交互式格式*（参考默认 `text`，对齐的人类可读
渲染）；非交互式（管道、重定向、被别的程序调用）→ *管道格式*（参考默认
`jsonl`，每行一条紧凑记录——程序化消费最有用的形态）。两半都可配置（Go：
`Config.FormatInteractive` / `Config.FormatPiped`），故应用可保留 TTY 感知
而改用如 `text`/`markdown`。TTY 检测：输出目标为字符设备时判为交互式
（Go：`*os.File` 且 `Stat().Mode()` 含 `ModeCharDevice`）；任何非文件
输出（buffer、管道）判为非交互式。SDK SHOULD 允许嵌入代码强制该判定
（Go：`cli.Options.Interactive`），以便无需真实终端即可测试交互式路径。

**生效格式的优先级（由高到低）：**

1. 本次调用的裸 `--format`/`--json`（未被遮蔽时）；
2. `--xyz.format`（命令行全局，§13.3）；
3. 逐命令 hint（Go：`CliHints.Format`）；
4. 全局代码配置（Go：`Config.Format`）；
5. 内置默认 `auto`。

任一层本身可为 `auto`，随后按上述 TTY 解析；某层给出具体值即钉死格式。
命令行层（1–2）高于代码层（3–4）；代码层内，逐命令 hint（3）高于全局
配置（4）。

**与自定义输出的关系。** 显式的非 `text` 格式（json/jsonl/markdown）绕过
命令的自定义 CLI 输出函数（§9.5）——机器与替代格式压过逐命令样式，正如
`--json` 一样；只有 `text`（含 `auto` 解析为 `text`）走自定义输出链。非法
的 `--format` 取值，或 `--format` 缺参数，是用法错误（退出码 2）。在机器
格式（json/jsonl）下，命令错误以 §8.6 错误对象写到 **stderr**（json 美化、
jsonl 紧凑），取代纯文本行；stdout 绝不承载错误，退出码不变（§10.5）。

**全称与冲突规则。** 规范的、始终可用的形式是带命名空间的内置参数
`--xyz.format=<fmt>`（§13.3），在 `--` 终止符之前任意位置消费；因带命名
空间，它绝不与命令自己的 flag 冲突。裸 `--format`（及其 `--json` 别名）是
便捷简写：*仅当目标命令自身没有定义 `format`（相应地 `json`）flag 时*才被
识别为全局格式选择器。当命令确实定义了同名 flag，裸标志归命令所有（作为
普通 §10.2 flag 绑定到命令字段），全局格式只来自 `--xyz.format`。这是 xyz
内置参数的通则：全称 `--xyz.<name>` 永远可用，裸 `--<name>` 简写只在不
遮蔽用户自定义参数时才生效。

**10.7a. 格式与样式是两条独立的轴。** TTY 信号驱动两条正交、可独立配置
的轴，MUST NOT 合并成单个「交互式」开关：

- **格式轴**（编码——本节）：`auto|text|json|jsonl|markdown`；
- **样式轴**（呈现——彩色/富排版，*保留*）：将来的 `auto|always|never`
  彩色模式（Go：规划中的 `Config.Color`），其 `auto` 由*同一个* TTY 探测
  **以及** `NO_COLOR`/`TERM=dumb` 与强制覆盖共同解析。

二者共用同一底层 TTY 检测，但各自独立解析，因为其覆盖语义不同：真实终端里
的 `NO_COLOR` 只去色、*不*改变格式；管道到 `less -R` 可强制开彩色而格式仍
为管道默认；用户也可能在交互式终端里就要 `--format jsonl`。参考惯例：主流
CLI（gh、kubectl、docker）管道时*格式*仍是文本/表格、只有*彩色*不同——即
TTY 信号最显眼的作用在样式轴。SDK MAY 将来再实现样式轴；在此之前只有格式
轴是规范性的，而共用的 TTY 探测正是样式轴日后接入的缝。

---

## 11. HTTP 前端

**11.1. 路由。** 带 HTTP `path` 的命令即参与路由；`{name}` 是单段路径
参数。方法来自 HTTP 提示：方法为空则**同时注册 GET 与 POST**（默认——GET
绑定 query 参数、POST 绑定 JSON/表单体加 query，二者走同一处理器）；方法
有值则钉住一个或多个方法（单个方法或逗号分隔列表，如 `GET,POST,PUT`）。
method+path 冲突配对 MUST 是注册错误。无 HTTP path 的命令不参与路由。

**11.2. 绑定。** 构建参数 map 的校验顺序：接口默认值（基底）→ JSON body
合并（GET/HEAD 以外的方法；读取上限 SHOULD 为 1 MiB；body 不可解析 = 400
`{"error":"invalid JSON body"}`）→ 逐字段：路径参数（线上名）、query 值
（默认位置；切片字段收集全部重复值，其他字段取第一个）、headers（名称
按 §4.6）、form 字段（仅取请求体）。JSON 解析失败的请求体只有在该请求**声明**了
`Content-Type: application/json` 时才判 400；未声明的体只是不参与合并。被 §4.1 排除的字段按键为语言字段名
接收 header 值。

**11.3. 内建端点**（每个 HTTP 前端 MUST 存在）：

- `GET /healthz` → 200，body 恰为 `{"status":"ok"}` + 换行。
- `GET /openapi.json` → 由同一批 `inputSchema` 生成的 OpenAPI 3.0.3 文档。
  每个操作：`summary`（命令摘要）与 `description`（命令描述，存在时）；每个
  path/query/header 字段一条 `parameters`，携带其线上名、位置、`required`
  （路径参数恒为必填）、`description`（字段的 `desc`）与富 `schema`（类型加
  enum/default/format——与 MCP `inputSchema` 同一份逐字段 schema，而非裸
  类型）；POST/PUT/PATCH 带 `requestBody`（application/json，输入 schema）；
  `responses`（200 携带输出 schema 内容，外加分类学的 400/404/500 描述）。
  每个注册方法各产出一条操作（默认 GET+POST 命令同时产出 get 与 post）。
  `info.title`/`info.version` 为应用身份（与 `X-App-Name`/`X-App-Version`
  同源，§11.6）；应用未设置时参考回退为 `example service`/`1`。

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

**11.6. 服务器上下文响应头。** 每个 HTTP 响应 MAY 携带服务器上下文头，
让调用方无需读体即可获知*是哪个应用程序*服务了本次请求、它的版本、以及
是哪个命令处理了它。报告两个不同的版本：**应用程序**（用 xyz 构建的程序）
的版本与 **xyz 库自身**的版本。启用时（默认），前端写：

| 响应头 | 值 |
|---|---|
| `X-App-Name` | 应用程序名（配置覆盖优先，否则用二进制 basename） |
| `X-App-Version` | *应用程序*的版本（配置覆盖优先，否则用构建期注入的版本槽；默认 `dev`） |
| `X-XYZ-Version` | *xyz 库自身*的版本（是库，不是应用程序） |
| `X-XYZ-Command` | 服务该路由的命令的点分入口名 |
| `X-XYZ-Duration-Ms` | handler 调用耗时（整毫秒） |

外加配置里用户自定义的静态头（Go：`Config.ResponseHeaders`，CLI
`--xyz.header k=v`）。单个配置开关（Go：`Config.NoServerHeaders`，CLI
`--xyz.no-server-headers`）抑制这五个自动 `X-App-*` / `X-XYZ-*` 头；用户
自定义头是显式配置，无论如何都写。身份/静态头适用于*所有*路由，含
`/healthz`、`/openapi.json` 与挂载的 `/mcp`；命令/耗时头是逐路由的。这些
头是建议性上下文，MUST NOT 影响响应体或状态码。（HTTP 头名不区分大小写；
参考 Go 实现以 net/http 的规范大小写 `X-App-Version` / `X-Xyz-Version`
发出。）

**11.7. 逐请求语言（Accept-Language）。** HTTP 前端逐请求从
`Accept-Language` 头解析语言：q 值最高的受支持标签胜出（`zh*` → zh-CN、
`en*` → en）；头缺席或不含受支持标签时回退到进程默认语言（§15.5）。解析出
的语言携带在请求 context 中，于是 (a) 框架生成的响应消息（如非法 JSON 体的
400、§8.6 错误体里的框架文本）以该语言发出，(b) handler 可读取它并本地化
自己的输出。公开访问器暴露它（Go：`xyz.LanguageFromCtx(ctx)`；底层携带为
`langx.WithLang`/`FromCtx`）。这是逐请求的、独立于进程级 CLI 语言；CLI 与
MCP 通道用进程语言（§15.5）。参考注记：深层管线消息（解码/类型转换）在尚未
编目处 MAY 保持英文；但请求语言已在 context 中可供 handler 使用、并供逐步
本地化。

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

**12.4. 工具。** 每条注册命令一个工具。工具元数据在共享定义允许的范围内
尽量丰富，且每一部分都可逐命令覆盖（Go：`MCPHints`）、默认取自 tag——常见
情形零配置，而单条命令可只覆盖其中任意一项而不影响其余：

- `name` —— 依 §12.4a；
- `description` —— §3.3 的 summary + description 合并，可由
  `MCPHints.Description` 覆盖；
- `title` —— 人类友好的显示名（作为 SDK 的 `annotations.title` 承载），由
  `MCPHints.Title` 或 `title:…` 注解字符串设置；
- `inputSchema` —— 管线 schema（§9），携带每个字段的 `desc`/enum/default/
  format；字段描述可由 `MCPFieldHint.Description` 覆盖；
- `outputSchema` —— 来自可静态模式化的结果类型（否则缺省）；
- `annotations` —— 由 `MCPHints` 注解字符串映射：`read` → readOnlyHint
  true；`write` → readOnlyHint false；`destructive` → destructiveHint true；
  `idempotent` → idempotentHint true；`openworld` → openWorldHint true；
  `title:…` → title；
- `_meta` —— 来自 `MCPHints.Meta` 的任意逐工具元数据（键值映射，并入工具
  保留的 `_meta`）。

**12.4a. 工具名覆写。** 命令 MAY 通过 `MCPHints{name}`（Go：
`MCPHints.Name`；Rust：待移植，见 deviations）为其 MCP 工具名钉名。设置
后，覆写名是 `tools/list` 里唯一对外通告、`tools/call` 唯一接受的名字——
点分原名不再作为 MCP 工具名，但 CLI 与 HTTP 通道保持不变，各通道可携带
各自的命名。为空（默认）时按 §3.2 用完整名。覆写名须满足 §3.1 文法。

**12.5. 调用结果。** 成功返回双重内容——由 *CLI 渲染器*渲染的
`textContent`（§9.1，修剪末尾换行）**以及**作为裸 JSON 值的
`structuredContent`。失败返回 `isError: true`，其文本是分类错误的最具体
消息（§8.4）。MCP 接口默认值只填充调用者未提供的键（§6.3）。

**12.6. 服务器身份。** MCP `serverInfo` 标识的是*应用程序*（用 xyz 构建的
程序），而非 xyz 库：`serverInfo.name` 默认为二进制 basename、
`serverInfo.version` 默认为 `0.0.0`，两者都可由配置覆盖（Go：`Config.Name`
/ `Config.Version`，与 HTTP 上报告的 `X-App-Name` / `X-App-Version` 同源，
§11.6）。xyz 库自身的版本另行报告（`_meta.xyz.sdk_version`，§12.8；
`X-XYZ-Version`，§11.6）。Bearer/CORS 配置适用于 http 传输（stdio 是本地
通道，MUST NOT 被包裹——发出警告注记是参考行为）。

**12.7. 内容块结果。** 命令 MAY 返回结构化内容块（text / image / audio /
resource）以替代纯值。块以 MCP `Content` 原样传递；`structuredContent` 若
存在则持有 JSON 呈现（§12.5 的双内容形态）。块结果的 JSON 呈现是
**保留块信封**：仅含 `content` 一个键的对象，其数组项形如
`{"type": "text", "text": …}` 或
`{"type": "image", "mimeType": …, "data": …}`——二进制载荷是 base64 字符串，
绝不裸字节。三个前端就此投影：

- **CLI**：文本块内联输出；二进制块写入系统临时目录的文件，以文件路径替
  代内容输出——大载荷不进终端；
- **HTTP**：信封即响应体（自描述 JSON，base64 内联）；
- **MCP**：`Content` 原样携带块，`structuredContent` 持有信封——面向文本的
  客户端与结构化消费者同时可用。

SDK 渲染"唯一键为 `content` 且各项恰好符合上述形状"的对象时 MUST 按块信封
处理；保留形状正是块结果在 handler 与前端之间的类型擦除中存续的方式。

**12.8. 服务器上下文结果元数据。** 与 HTTP §11.6 头对应，每个工具调用
结果 MAY 在结果 `_meta`（MCP 保留的元数据对象）的 `xyz` 键下携带服务器
上下文：

```json
{"xyz":{"app_name":"…","app_version":"…","sdk_version":"…","command":"…",
        "duration_ms":12,"headers":{…},
        "error":{"kind":"…","code":"…","detail":{…}}}}
```

`app_name`/`app_version` 标识*应用程序*（用 xyz 构建的程序），
`sdk_version` 是 xyz 库自身的版本，`command`/`duration_ms` 描述本次调用；
这五项在每次调用（成功与错误）都在。`headers` 镜像用户自定义静态头（未
配置则缺席）；`error` 仅在 `isError` 结果上出现，携带 §8.6 的
`kind`/`code`/`detail`，让客户端无需解析 `textContent` 即可按领域语义
分支。抑制 HTTP 自动头（§11.6）的同一个配置开关也抑制 `_meta.xyz`（Go：
`Config.NoServerHeaders` / MCP `NoServerMeta`）；官方 SDK 自己的 `_meta`
条目（如 `io.modelcontextprotocol/serverInfo`）不受影响。应用身份还会流入
initialize 结果的 MCP `serverInfo`（§12.6）——`serverInfo.name` = 应用名、
`serverInfo.version` = 应用版本（替换 `0.0.0` 默认值）——于是看不到 HTTP
头的 stdio 客户端也能获知应用身份。

---

## 13. 根派发器

**13.1. 模式词。** 默认模式词：`serve`（HTTP REST + `/openapi.json` +
`/mcp` 端点）、`http`（仅 HTTP REST + `/openapi.json`——单独的 HTTP 接口，
不挂 `/mcp`）、`mcp`、`help`。它们可经配置改名，并为所有库消息原样使用
（除此之外任何地方都不得硬编码这些词）。解析规则：配置字段为空则保留默认
值；词必须朴素（无前导短横、无空白）且两两互异；违反则退出 2。

**命名空间形态与遮蔽。** 每个模式词 `W` 还有一个始终可用的命名空间形态
`xyz.W`（与 `--xyz.*` 参数命名空间一致），它*永不*显示在帮助里。顶层段等于
`W` 的用户命令会*遮蔽*裸词：此时 `W` 路由到用户命令，内建模式仅经 `xyz.W`
可达。遮蔽取代了旧的硬保留——注册名为如 `serve.*` 的命令不再是错误
（§3.4）。概览只在未遮蔽时列出某模式的裸词；被遮蔽的模式被省略（用户命令
改出现在命令表里），其 `xyz.W` 形态保持隐藏。CLI-Skip 的命令不参与遮蔽。

**13.2. 派发顺序**（固定）：

1. 空注册表 → 静默无操作，退出 0；
2. `--` 终止符之前的 `-v`/`--version` → `<app-name> version <app-version>`，
   退出 0；
3. 剥离全局 `--xyz.*` 内建参数（非法值退出 2）；
4. 空参数 / 根级 `--help` / `-h` → 概览（模式列表 + 命令表；CLI 被禁用时
   不显示命令表）。概览 MAY 携带两段原样配置块——`help_before` 在最前、
   `help_after` 在最后（命令表被省略时也打印；语义与 `-h` 块相同：原样/
   换行归一/空块零操作）；
5. 首 token 的模式匹配：`xyz.W` 恒选中内建模式；裸 `W` 仅在未遮蔽时选中
   （§13.1）。随后：
   - `help` 模式 → 无参数的 `help` 打印概览；`help <模式>` 打印该模式的
     帮助；`help <命令路径>`（点分 `user.add` 或空格 `user add`）打印该命令
     的详细帮助（与 `<命令路径> -h` 相同）；
   - `serve`/`http`/`mcp` 的参数里含 `-h`/`--help` 时打印该模式的帮助并
     退出 0，*不起服务*；
   - 否则 `serve` → HTTP 模式（REST + `/mcp`），`http` → HTTP 模式（仅
     REST），`mcp <transport>` → MCP 模式；
6. 其余 → CLI 模式。

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
| `--default k=v`（可重复） | 通道默认 | serve/mcp 启动默认，补缺失的请求/调用键（§6.1） |
| `--xyz.header k=v`（可重复） | 响应头 | 每个 HTTP 响应的静态上下文头，镜像进 MCP `_meta.xyz.headers`（§11.6/§12.8） |
| `--xyz.no-server-headers` | 无服务器头 | 抑制自动 `X-App-*`/`X-XYZ-*` 头与 MCP `_meta.xyz`（§11.6/§12.8）；用户 `--xyz.header` 值仍生效 |
| `--xyz.format=auto\|text\|json\|jsonl\|markdown` | 输出格式 | CLI 默认输出格式（§10.7）；`auto` 按 TTY 解析。命令行全局层——仅被裸 `--format`/`--json` 压过，压过逐命令 hint 与代码配置 |

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
MCP 把客户端断开视为正常退出 0。HTTP 的*请求自身*取消（客户端断开）
SHOULD 额外取消处理器上下文（Go 通过 `r.Context()` 做到；Rust 目前只
传播 serve 级上下文——登记为 deviations.md 的 D-rust-11）。

**13.9. 可组合派发。** MUST 暴露可组合入口——Go：`TryRun(reg, args)
(code int, handled bool)`（另有 `TryRunConfig`）；Rust：`try_run(reg,
args) -> (i32, bool)`（另有 `try_run_config`）。语义：§13.2 的完整派发
管线，唯 CLI 模式下首段未知（非命令段/别名、非 flag、无默认子命令）时
返回 `(0, false)` 且**静默**（不打印任何内容）——宿主可围绕 xyz 路由
自己的命令，无需第二套派发。其余（总览/版本/模式词/已知命令/CLI-skip
段的未命中）与非可组合入口完全一致。

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
7. **运行时环境上下文** —— 公开访问器，让 handler、自定义输出函数、中间件
   或宿主程序无需各自重复探测即可查询解析后的环境：UI 语言、主输出是否交互
   式（TTY）、是否去色。Go：`xyz.Language() string`、`xyz.Interactive()
   bool`、`xyz.NoColor() bool`、`xyz.Env() EnvContext{Language, Interactive,
   NoColor}`，以及逐请求的 `xyz.LanguageFromCtx(ctx)`（§11.7）。除 ctx
   访问器外均为进程/CLI 语境；TTY 探测是共享叶子（Go：`termx.Interactive`），
   格式轴（§10.7）与保留的样式轴（§10.7a）都查询它。

参考对接点：Go 的 `registry.New` / `spec.Define(...)` /
`cli.NewWithOptions` + `App.Use` / `httpapi.HandlerFor` / `mcp.Server` /
`xyz.Env`；Rust 的 `xyz_rust::Registry` / `spec::command::Command` /
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

**15.4. 惯用入口。** 每种语言提供自己的构造方式（Go：流式
`Define(...)...Run()` 链加 `Main/Run` 函数；Rust：`define(...)...run()`
构建器加 `main`/`run`/`run_config`）——规范内容是这个*集合*：一行式
main、返回退出码的可分派函数、逐命令注册、代码与命令行双重配置注入。

**15.5. 语言（l10n）。** 所有内置界面文本的语言（总览、帮助标签、用法
错误、派发警告、MCP 用法与诊断；§8 错误分类消息保持英文）按以下优先级
选择：

```
--xyz.lang flag > Config 语言（Go: Config.Lang，Rust: Config.lang）> LANG/LC_ALL 检测（小写 zh 前缀 → zh-CN；C/POSIX/缺失 → en）> en
```

- 库 MUST 内建 **en**（规范默认）与 **zh-CN** 两种语言；未知的
  `--xyz.lang` 值 MUST 在解析期报错（退出 2）。
- 每条内置字符串是一个**消息键**。规范键名与英文文案是规范性的——各
  SDK MUST 使用相同键名与英文措辞（golden 输出依赖此点）。传输词相关键
  （`overview.mcp_mode`、`mcp.usage`、`mcp.err_unknown_transport`、
  `mcp.err_sse_removed`）在本地官方 MCP SDK 传输集不同时可按 SDK 取值
  （登记于 deviations.md）。
- 用户内容（summary、description、帮助块）**永不**翻译。
- 用户通过按语言的覆盖表配置多语言内容（Go `Config.Translations`、Rust
  `Config.translations`：语言 → （键 → 文本））；覆盖优先于内置译文；
  未知键回退键名本身且 MUST NOT panic。
- 本版本（v0.2.0）的目录键集：

```
overview.usage_line   overview.cli_mode   overview.serve_mode   overview.mcp_mode
overview.builtins     overview.commands   overview.disabled     overview.not_compiled
help.usage            help.aliases        help.commands         help.flags
help.global_flags     help.commands_placeholder   help.flags_placeholder
help.json_flag        help.version_flag   help.help_flag
cli.err_positional_count
warn.mode_disabled    warn.no_cli         warn.bearer_stdio     stub.not_compiled
log.serve_listening   log.graceful        log.mcp_listening     log.cors_on
log.debug_dispatch
mcp.usage             mcp.err_missing_transport   mcp.err_unknown_transport
mcp.err_sse_removed   mcp.err_unknown_version      mcp.err_empty_version
mcp.err_transport_versions   mcp.err_usage_extra_arg
```

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
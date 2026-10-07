# xyz-spec — xyz 跨语言实现规范

**每个语言的 xyz SDK 都必须实现的契约。**

xyz 把一次命令定义变成三种接口——CLI、HTTP REST、MCP 工具服务器。本仓库
是跨语言规范：在库已有 Go（xyz-go）、Rust（xyz-rust）两个实现，并即将
支持 Node、Java 等语言的今天，用它来保证各语言实现的行为与用户体验
完全一致。

> English entry: [README.md](README.md)

## 文档

| 文档 | 内容 |
|---|---|
| [spec.md](spec.md) | **规范性契约**（v0.4.5）：定义词汇表、类型、默认值、校验、调用管线、错误分类学（含富化 code/detail/status 与共享错误体）、渲染（含逐通道输出函数与 CLI `--format`）、三个前端（含 HTTP 应用身份/SDK 版本响应头与 MCP 结果 `_meta`）、根派发器、嵌入面、体验约定、治理。RFC 2119 措辞。 |
| [spec.zh-CN.md](spec.zh-CN.md) | spec.md 的中文镜像（英文为准） |
| [conformance.md](conformance.md) | 一致性验收：A/B 两类检查单、11 命令展示程序与逐字节 golden 输出、必测的 feature 矩阵。 |
| [deviations.md](deviations.md) | 差异登记表：各 SDK 与规范不符之处（语言必然 / SDK 缺失 / 扩展），状态分 open / resolved-in-spec / closed-by-implementation / retired，每次规范发布时复审。 |

## SDK 状态

| SDK | 包 | 目标规范版本 | 备注 |
|---|---|---|---|
| [xyz-go](https://github.com/ejfkdev/xyz-go) | `github.com/ejfkdev/xyz-go`（v0.4.5） | **v0.4.5**（基线锚点） | Go 参考实现 |
| [xyz-rust](https://github.com/ejfkdev/xyz-rust) | crates.io `xyz-rust` 0.4.4 | v0.4.4（spec v0.4.5 条款待补） | Rust 参考实现 |

### 兼容矩阵

每个 SDK 发布恰好对齐一个规范版本（锚点记录于 [spec.md](spec.md) 与该 SDK
自己的发布说明）。当前锚点：

| 规范 | xyz-go | xyz-rust |
|---|---|---|
| v0.4.5 | v0.4.5 ✅ | 待补（0.4.4 → v0.4.4） |
| v0.4.4 | v0.4.4 | 0.4.4 ✅ |
| v0.4.2 | v0.4.2 | 0.4.2–0.4.3 ✅ |
| v0.4.1 | v0.4.1 | 0.4.2 |
| v0.4.0 | v0.4.0 | 0.4.0 |

（spec/xyz-go 跳过了 v0.4.3；xyz-rust 用 0.4.3 做一次 crates.io 构建修复。
xyz-rust 的 0.4.x 是该 crate 自己的版本号，不必等于它对齐的 spec 版本。）

v0.4.5 新增面——富化的 `openapi.json`（逐操作 description、带描述与富
schema 的 parameters、requestBody、应用身份 `info`）§11.3、仅有 path 的命令
默认注册 GET+POST §11.1、逐命令富化的 MCP 工具元数据（`MCPHints.Title/
Description/Meta`、`MCPFieldHint.Description`）§12.4——已由 xyz-go v0.4.5
交付；xyz-rust 后续跟进。v0.4.4 面（TTY 感知 `--format auto` §10.7、四模式
词 + `xyz.<词>` 命名空间与遮蔽 §13.1、`help <命令>`/模式 `-h` §10.4/§13.2、
逐请求 `Accept-Language` §11.7、运行时环境上下文 API §14 item 7）已由
xyz-go v0.4.4 与 xyz-rust 0.4.4 交付。v0.4.2 面（富化错误 §8.5/§8.6、CLI
`--format` §10.7、应用身份头 §11.6、MCP `_meta` §12.8）已由 xyz-go v0.4.2 与
xyz-rust 0.4.2–0.4.3 双双交付。MAY 条款（§9.5 输出函数、§11.6 响应头、
§12.8 `_meta`、§8.5 富化错误层）对不实现的 SDK 不构成义务。完整历史见各
仓库的 git tag。

xyz-go 登记一条开放差异（D-go-01，tagged unions）；xyz-rust 在
[deviations.md](deviations.md) 登记（Duration 符号、序列化后渲染、官方
Rust MCP SDK 的传输差异、版本注入等）。

## 阅读指引

1. SDK 实现者从 [spec.md](spec.md) §1–§9 入手（皆为面向用户的行为），
   再读 §10–§13（前端）、§14–§15（嵌入与体验面）。
2. 宣称一致性前：跑 [conformance.md](conformance.md) 的检查单、fixture
   与 golden 输出、六组合 feature 矩阵，并提交差异登记。
3. 变更提议：在本仓库开 issue/PR 并附参考实现 diff（spec §17.2）。规范
   版本语义化：MAJOR 改变规范行为，MINOR 增加规范面，PATCH 修正措辞。

## 许可证

[MIT](LICENSE)
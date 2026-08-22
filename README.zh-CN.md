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
| [spec.md](spec.md) | **规范性契约**（v0.1.2）：定义词汇表、类型、默认值、校验、调用管线、错误分类学、渲染、三个前端、根派发器、嵌入面、体验约定、治理。RFC 2119 措辞。 |
| [spec.zh-CN.md](spec.zh-CN.md) | spec.md 的中文镜像（英文为准） |
| [conformance.md](conformance.md) | 一致性验收：A/B 两类检查单、11 命令展示程序与逐字节 golden 输出、必测的 feature 矩阵。 |
| [deviations.md](deviations.md) | 差异登记表：各 SDK 与规范不符之处（语言必然 / SDK 缺失 / 扩展），每次规范发布时复审。 |

## SDK 状态

| SDK | 包 | 目标规范版本 | 备注 |
|---|---|---|---|
| [xyz-go](https://github.com/ejfkdev/xyz-go) | `github.com/ejfkdev/xyz-go`（v0.2.4+） | v0.1.2（基线锚点） | Go 参考实现 |
| [xyz-rust](https://github.com/ejfkdev/xyz-rust) | crates.io `xyz-rust` 0.1.2 | v0.1.2（基线锚点） | Rust 参考实现 |

两个锚点通过全部一致性验收；xyz-go 无差异登记，xyz-rust 在
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
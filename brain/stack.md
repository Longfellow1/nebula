---
slug: stack
title: Tech stack
role: tech-stack choices
updated: "2026-10-16T11:05:30"
---

# Tech stack

以下技术来自当前仓库，不是本轮新增选型。

| 范围 | 已观察 | 对 P0 的含义 |
|---|---|---|
| Workspace | Rust 2021；nebula-core / daemon / fuzzy / tui / CLI | parser 先留 daemon 私有 |
| 解析 | serde / serde_json | 复用依赖，不先添加通用 adapter 框架 |
| 后台与 hook | tokio / axum；loopback HTTP | parser 不进入 hook 热路径或持锁执行 |
| 持久化 | rusqlite / SQLite | 不新增 journal 或 migration |
| TUI | ratatui / crossterm | 本轮不改 Review UI |
| IPC | length-prefixed compact MessagePack；协议 50，最低兼容 49；帧上限 4 MiB | 内部事件契约不是 wire schema；未来扩展需版本门控 |
| 测试 | Rust 内联测试、tempfile | 未来用合成输入测试真实 parser，无需个人 transcripts |
| Brain | 全球 brain-page skill 的零依赖 Node CLI | 仅文档维护工具，不是 Nebula runtime 依赖 |

待定：来源 RFC3339 时间校验是否需要依赖，以及具体 budget 默认值。没有运行 Rust build/test，不声称工具链或当前整库测试已验证。见 [[issue-144-session-timeline]]。

---
slug: architecture
title: System architecture
role: system architecture
updated: "2026-10-16T11:05:30"
---

# System architecture

核实基线：`a00b76075913bb098021177295e0dab37d3521aa`（workspace 0.44.0）。

已观察：daemon 拥有 PTY、SQLite 与状态；ratatui TUI 经 Unix socket/MessagePack 通信；harness hook 经独立的 loopback HTTP 路径进入 daemon。领域层次为 Project -> Worktree -> Session。

Issue #144 的关键架构差异：Review History 目前只显示占位文字，没有获取历史数据；`orchestrator read` 读 PTY 尾部，通用 `nebula read` 才优先读记住的 Claude transcript。PTY 字节回放不能当作结构化事件时间线。

建议（尚未获维护者接受）：新增 daemon 私有 `session_timeline.rs`，显式输入 reader/session/source generation/limits，输出规范化事件与覆盖诊断。P0 不接 hook、后台 sweep、IPC、数据库或 UI。

可复用 `TranscriptRef` 与现有 JSONL 样本/完整换行游标模式；不能直接复用扁平化文本 renderer 或 64 KiB tail 来声称完整历史。详见 [[issue-144-session-timeline]] 和根目录 `ARCHITECTURE_GAP.md`。

---
slug: background
title: Project background
role: project background
updated: "2026-10-16T11:05:30"
---

# Project background

本 Brain 属于 `AgentSystemLabs/nebula` 的本地贡献分析 checkout，不是父目录 C_Agents 产品的 Brain。创建日期：2026-10-16（Asia/Taipei）。

Nebula 是面向 coding agents 的终端控制台；daemon 保持 agent PTY 存活，TUI 用项目、worktree 和 session 组织视图。当前 Brain 只覆盖 Issue #144 所需的架构勘察，并非整个仓库历史的完整总结。

当前目标：为 Session timeline / replay 提供维护者易评审的实现计划，先理解事实、定义语义与边界，不完整实现功能。Claude Code 优先；未来可支持 Codex/Pi，但当前不开发适配器。

任务背景、进度与约束见 [[issue-144-session-timeline]]。维护者交付草稿位于仓库根目录 `ARCHITECTURE_GAP.md`、`EVENT_SCHEMA.md`、`P0_PLAN.md`。

明确边界：不改核心协议、数据库迁移、Review UI，不发表 GitHub 评论或开 PR。用户安排 agy 整理工作树；Codex 当前只维护文档/Brain，不替其切分支、暂存或提交。

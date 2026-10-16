---
slug: mindmap
title: Feature mindmap
role: feature mindmap
updated: "2026-10-16T11:05:30"
---

# Feature mindmap

当前知识导航只围绕贡献准备，不扩写 Nebula 的完整功能图。

- 产品背景与原始需求：[[issue-144-session-timeline]]；上游 Issue #144。
- 实际架构与文档差异：`ARCHITECTURE_GAP.md`。
- 事件、身份、时间、工具关联、来源及脱敏：`EVENT_SCHEMA.md`。
- P0 范围、预计文件、测试矩阵、进度和维护者待决项：`P0_PLAN.md`。
- 协作边界：用户/agy 整理工作树；Codex 文档工作不切分支、不暂存、不提交。

核心风险：隐私、transcript schema drift、重复/片段、读取内存边界、volatile transcript discovery、混合版本 IPC。当前建议均为 draft，不代表 upstream 已接受。

---
slug: roadmap
title: Roadmap
role: milestones
updated: "2026-10-16T11:05:30"
---

# Roadmap

日期：2026-10-16（Asia/Taipei）。以下是本次贡献准备的阶段，不是维护者承诺的排期。

| 阶段 | 状态 | 验收 |
|---|---|---|
| R0 勘察与任务对齐 | 已完成 | README/架构/orchestrator/Review 已读，调用链交叉核实 |
| R1 背景沉淀与方案 | 已写入草稿 | Brain + 三份根目录文档；链接与范围检查结果写入项目页 timeline |
| R2 agy 整理工作树 | 用户另行安排，完成情况未验证 | Codex 不替其切分支、暂存、提交或清理父目录 |
| P0 parser/fixtures/tests | 未开始 | 先确认契约和边界，再在隔离本地分支实施，约 6 个 source/test 文件 |
| P1 discovery/IPC/History | 明确后续范围 | 维护者先确认 UI、分页、权限、协议门控 |
| Codex/Pi adapters | 延后 | 各自 schema、fixtures 和 identity mapping，不在 P0 实现 |

恢复工作：先读 [[issue-144-session-timeline]]，再看 `P0_PLAN.md` 的未决事项与测试矩阵。实施前重新检查 HEAD、dirty/staged/untracked 与用户/agy 的最新工作，不能把本次勘察基线当作实时现场。

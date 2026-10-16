---
slug: flow
title: Key flows
role: key flows
updated: "2026-10-16T11:05:30"
---

# Key flows

以下为已观察链路，基线 `a00b7607`；没有把建议的 parser 画成已接入。

```text
Claude hook -> authenticated loopback POST -> HookPayload -> HookDelivery
                                                        +-> status / prompts / cwd
                                                        +-> TranscriptRef -> title sync
remembered TranscriptRef -> nebula read -> flattened context text
live PTY -> orchestrator read -> cleaned OutputTail
ReviewOpened -> Review overlay -> static History placeholder
```

hook 不保留完整时间戳/tool-use correlation，也没有持久事件日志。它能帮助知道来源在哪里，无法独立还原历史。`PreToolUse` 只匹配 `AskUserQuestion`。

建议链路（未实现）：显式 Claude JSONL reader -> 有限 record framing -> 按块归一化 -> 事件、诊断和 coverage。按物理来源顺序输出，不用读取时刻补缺失时间，不把 tool-result-only 的 user envelope 当作人类 prompt。

相关语义与决策见 [[issue-144-session-timeline]]、`EVENT_SCHEMA.md`。

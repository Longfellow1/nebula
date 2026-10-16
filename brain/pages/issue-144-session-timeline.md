---
id: issue-144-session-timeline
title: "Issue #144：Session timeline / replay 贡献准备"
category: project
status: active
tags: [nebula, issue-144, timeline, planning]
created: "2026-10-16T11:05:30"
updated: "2026-10-16T12:58:52"
---

<!-- compiled_truth -->
## 目标与授权范围

用户希望分析 [AgentSystemLabs/nebula Issue #144](https://github.com/AgentSystemLabs/nebula/issues/144)，交付维护者易评审的 `ARCHITECTURE_GAP.md`、`EVENT_SCHEMA.md`、`P0_PLAN.md`，而不是完整实现 timeline/replay。

首轮明确要求先理解、同步状态和认知，并让 sub 帮忙抓取信息；已用 3 个只读 sub 分别调查 ingress、Review/History/IPC、contract/risks，并由主控交叉检查。后续用户要求建立 Brain、更新必要文档与进度、同步背景；工作树由用户另行安排 agy 整理。

限制：不改 core protocol、DB migrations 或 Review UI；不发表 GitHub 评论/开 PR；不访问个人真实 transcripts，不运行 harness。当前仅文档维护，不新增 parser、fixture 文件或 executable tests，不切分支/暂存/提交。

## 基线与资料

- checkout：`/Users/Harland/Go/C_agents/nebula`；父目录 C_Agents 是另一个项目，已有大量 unrelated dirty changes，禁止顺手清理。
- 勘察与本轮写入前 HEAD：`a00b76075913bb098021177295e0dab37d3521aa`，branch `main`，Nebula 工作树初始 clean，workspace 0.44.0。
- 已读 `README.md`、`ARCHITECTURE.md`、`docs/orchestrator.md`、现有 Review modal 与相关调用链；不是全仓历史审计。
- 源码定位与论证以根目录 `ARCHITECTURE_GAP.md` 为依据；后续 HEAD 变化时重新验证，不把源码行号当永久 API。

## 已核实事实

1. History 没有真实取数/渲染。`crates/nebula-tui/src/ui/overlay.rs:76` 渲染静态文字；`review_modal.rs:7` 仅持有 selection/layout；`event_loop/server_events.rs:489` 在 `ReviewOpened` 时只打开 modal。
2. `orchestrator read` -> daemon `orchestrator.rs:98` -> `session_context.rs:116` 读取 capped 256 KiB 的 cleaned PTY tail；通用 `nebula read` -> `session_context.rs:94` 优先读记住的 Claude transcript，然后 fallback PTY。`session_context.rs:100` 的注释与 orchestrator 的实际实现不一致。
3. Claude spawn 安装 hooks，携带 Nebula AgentId、loopback URL 和 bearer token；`hooks/mod.rs:361` 验证/解析 -> `HookDelivery` -> `lib.rs:189` drain -> status/prompts/transcript/title/cwd。只有 Claude route 产生 `TranscriptRef`。
4. hooks 没有保留完整来源时间、tool_use_id 或事件 journal；installer 的 PreToolUse 只匹配 AskUserQuestion。无法靠这些 lossy status signals 重建可靠 timeline。
5. `session_title.rs:54` 的 TranscriptRef 和 `registry.rs:191` 的 map 可供未来定位，但 map volatile、重启后可能要等新 hook。现有 resume existence helper 不是完整历史发现服务。
6. `session_context.rs:323` 的文本 renderer 丢失时间、UUID、工具关联和 block 边界；64 KiB tail 不能代表完整 replay；`session_model.rs:58` 有完整换行游标模式，但 unbounded read_to_end 不适合直接当新的内存保证。
7. IPC 当前/最低协议 50/49，帧 4 MiB；新增 variant 必须 peer gate，不能无条件广播给旧客户端。P0 内部类型不触碰该边界。

## 长期贡献目标（2026-10-16 补充）

用户明确希望通过本次贡献逐步成长为 Nebula 核心贡献者。评估 P0 时需要同时考虑维护者评审成本、可合并的实际价值、后续端到端交付与长期维护责任；不能仅以文件数少或 parser 测试通过作为成功标准。当前正在解释“收敛只读解析层”的理由与战略取舍，尚未获维护者接受或新增实施/对外沟通授权。

## 当前建议（不是维护者已接受决策）

新增 daemon 私有 `session_timeline.rs` + 一行 module declaration。显式 reader/session/source generation/limits 输入；events/diagnostics/coverage 输出。按来源物理顺序处理；timestamp 缺失则 null，不补读取时间；先按 block type 区分 tool call/result，避免 user envelope 误判。

最小事件：deterministic physical id、nullable source timestamp、kind、Nebula/harness session identity、tool_call_id、opaque source reference、redacted structural summary。来源 identity、provider UUID 与 call correlation 各司其职。distinct physical duplicates 先保留并报告，不擅自按 message.id 合并。默认不保留正文、命令、路径、参数、结果或 thinking。

不接 hook hot path、后台扫描、现有 text read APIs、数据库或 Review。未来 Codex/Pi 各自 adapter 映射到同一语义契约，当前不创建 adapter registry 或实现它们。

## 交付与真实进度

三份根目录方案文档已写入，内容包括 source evidence、风险、minimal schema、约 6 个 implementation/test 文件的估计、合成 fixture 设计、deterministic test matrix 和实际可用的未来验证命令。仅有文档内 synthetic JSONL example，没有 fixture 文件或 production parser。

本轮 Brain 通过 brain CLI 初始化/写入，`AGENTS.md` 和 `CLAUDE.md` 由 CLI 接线。文档与 Brain 的核验状态以本页后续 timeline evidence 为准；Rust build/test 未运行，不声称功能通过。

## 待维护者决定

- History 的产品语义：结构化 transcript timeline 和 bounded PTY output 的关系。
- 是否接受 structural-only summaries；可读 snippets/raw details 的隐私与访问策略。
- Claude 支持版本/grammar；时间校验工具；读取、单记录、事件、summary/correlation budgets。
- duplicate/fragment 合并标准、block-level continuation cursor 与未配对 tool results 的展示。
- source generation、重启后文件发现、subagent/sidechain/compaction coverage。
- 后续分页、detail authority 和 IPC mixed-version gates。

## 接手规则

先重新检查工作树/HEAD 和用户的 agy 整理结果；本 Brain 不是 agy 的执行回执。保持 Brain/support 文档与未来 upstream parser patch 可区分，是否提交 Brain 由用户/agy 决定。当前不应开始全量实现或未经授权写 GitHub。后续只有实现获授权时才创建/使用隔离本地分支，并运行 `P0_PLAN.md` 中的验证。


## Timeline

- time: 2026-10-16T11:05:30
  kind: decision
  summary: "Created this page: Issue #144：Session timeline / replay 贡献准备"
  source: "用户任务与 2026-10-16 只读源码勘察，基线 a00b76075913bb098021177295e0dab37d3521aa"
  affects: [issue-144-session-timeline]

- time: 2026-10-16T11:05:30
  kind: decision
  summary: "同步原始需求、已核实调用链、P0 建议、文档进度与 agy 工作树分工；明确未实施部分"
  source: "用户两轮任务与本地 a00b7607 源码勘察；ARCHITECTURE_GAP.md / EVENT_SCHEMA.md / P0_PLAN.md"
  affects: [issue-144-session-timeline]

- time: 2026-10-16T11:06:29
  kind: evidence
  summary: "文档核验完成：Brain lint-links 通过（1 project page + 6 root pages）；三文档相对链接、显式源码路径/行号范围、代码围栏及 3 行合成 JSONL 语法检查通过；scope audit 确认 14 个预期新增文档，无 tracked diff/staged/source changes，HEAD 与 origin/main 基线一致；未运行 Rust build/tests，未创建分支/提交/PR。"
  source: "2026-10-16 文档核验、brain CLI readback/lint-links 与 git scope/remote audit"
  affects: [issue-144-session-timeline]

- time: 2026-10-16T12:58:52
  kind: decision
  summary: "补充用户长期目标：成长为核心贡献者；P0 的评估需覆盖实际价值、可评审性与后续持续维护"
  source: "2026-10-16 用户询问收敛理由及其对成长为核心贡献者的帮助"
  affects: [issue-144-session-timeline]

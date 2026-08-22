# Codex 工作流程全景分析

> 分析对象：`codex-rs`（OpenAI Codex CLI 的 Rust 实现）
> 代码位置：`https://github.com/openai/codex`
> 分析日期：2026-08-22
> 主题：从用户键入需求开始，Codex 内部如何逐层处理，覆盖所有分支流程

---

## 目录

- [一、架构与分层](#一架构与分层)
- [二、阶段 1 — 按键与输入编辑](#二阶段-1--按键与输入编辑)
- [三、阶段 2 — 提交与 Op 构造](#三阶段-2--提交与-op-构造)
- [四、阶段 3 — Core 调度与任务互斥](#四阶段-3--core-调度与任务互斥)
- [五、阶段 4 — run_turn 前置流程](#五阶段-4--run_turn-前置流程)
- [六、阶段 5 — 采样循环](#六阶段-5--采样循环)
- [七、阶段 6 — 工具分发](#七阶段-6--工具分发)
- [八、阶段 7 — 审批与沙箱](#八阶段-7--审批与沙箱)
- [九、阶段 8 — apply_patch 与 diff 追踪](#九阶段-8--apply_patch-与-diff-追踪)
- [十、阶段 9 — 上下文管理与压缩](#十阶段-9--上下文管理与压缩)
- [十一、阶段 10 — MCP 集成](#十一阶段-10--mcp-集成)
- [十二、阶段 11 — 中断、错误、限流](#十二阶段-11--中断错误限流)
- [十三、阶段 12 — 持久化与恢复](#十三阶段-12--持久化与恢复)
- [十四、非主线流程速查](#十四非主线流程速查)
- [十五、规模参考](#十五规模参考)
- [十六、验证边界](#十六验证边界)

---

## 一、架构与分层

```
键盘事件
  ↓ crossterm
TUI (codex-rs/tui)      ChatComposer → InputResult → ChatWidget → AppCommand
  ↓ AppEvent::CodexOp
协议 (codex-rs/protocol) Op（提交队列 SQ） ⇄ Event/EventMsg（事件队列 EQ）
  ↓
Core (codex-rs/core)    submission_loop → spawn_task → run_turn 采样循环
  ↓                        ├─ ModelClient / ModelClientSession → Responses API
  ↓                        └─ ToolRouter → ToolRegistry → handlers
执行层                   exec-server（子进程+沙箱） / apply-patch / codex-mcp
```

**双队列模型是整个设计的地基**：任何客户端只能投递 `Op`，core 只能广播 `Event`。TUI、`app-server`（IDE / JSON-RPC）、`exec`（非交互 CLI）三种前端共享同一协议面，因此下文的 core 流程对三者完全一致。

整个 workspace 约 140 个 crate，统一 `codex-` 前缀。

---

## 二、阶段 1 — 按键与输入编辑

事件入口 `tui/src/app.rs:738-817` `handle_tui_event`，键事件下沉到 `bottom_pane/chat_composer.rs:1924` `handle_key_event`。此层已经开始分流：

- **粘贴突发检测**：`chat_composer.rs:3059-3096`，50ms 窗口内的连续输入判为粘贴，跳过逐字符的补全 / 高亮重算
- **斜杠命令**：`try_dispatch_bare_slash_command` (`:3246`)、`try_dispatch_slash_command_with_args` (`:3266`)；`slash_command.rs` 定义 40+ 命令，三个门控谓词 `supports_inline_args` (`:161`)、`available_in_side_conversation` (`:188`)、`available_during_task` (`:205`)，内置表 `built_in_slash_commands` (`:278`)
- **`@` 提及**：文件 / skill / plugin / app 四类，提交时才展开
- **`!cmd`**：shell 转义
- **图片**：粘贴或拖入成为独立输入项
- **历史导航**：上下键翻历史

编辑完成产出 `InputResult`（`chat_composer.rs:344`）—— 第一个真正的分支点：

| 变体 | 含义 |
| --- | --- |
| `Submitted` | 正常提交给模型 |
| `Queued` | 当前有活跃 turn，排队等待（steering） |
| `Command` / `CommandWithArgs` | 斜杠命令 |
| `ServiceTierCommand` | 切换服务档位 |
| `ParentOwnedInputBlocked` | 子会话中输入被父级接管 |
| `None` | 仅编辑，无副作用 |

---

## 三、阶段 2 — 提交与 Op 构造

`chatwidget/input_flow.rs:15-82` `handle_composer_input_result` 按上表分派。走 `Submitted` 时进入 `chatwidget/input_submission.rs:96` `submit_user_message_with_history_and_shell_escape_policy` —— 输入展开的核心：

- `:153` `ShellEscapePolicy::Allow` 且以 `!` 开头 → 走 `Op::RunUserShellCommand`，**完全绕过模型**
- `:191-229` skill 提及：`skill://` 前缀绑定 + `find_skill_mentions_with_tool_mentions` 文本提及，两路合并去重
- `:231-253` plugin 提及（`plugin://` 前缀）
- `:255-292` app 提及
- `:332-351` 组装 `AppCommand`

外层入口：`user_message_from_submission` (`:6`)、`submit_user_message_with_shell_escape_policy` (`:83`)。

随后 `app_command.rs:115-142` `AppCommand::user_turn(...)` 打包成一次完整轮次请求，经 `chatwidget.rs:1797-1831` `submit_op<T: Into<AppCommand>>` 发出 `AppEvent::CodexOp`（`app_event.rs:408`），落到 core 的 `Op`。

### Op 命令面

定义在 `protocol/src/protocol.rs:544`，标记为 `#[non_exhaustive]`。真正的用户输入入口是 **`Op::TurnInput`**（`:572`）：

```rust
TurnInput {
    request: Box<TurnInputRequest>,
    mode: TurnInputMode,
    reply: oneshot::Sender<CodexResult<TurnInputSubmission>>,
}
```

> ⚠️ **文档过时提醒**：仓库 `AGENTS.md` 的集成测试示例仍写 `Op::UserTurn { ... }`，该变体在当前枚举中**已不存在**。编写测试或对接协议时以 `protocol.rs:572` 的 `Op::TurnInput` 为准。

---

## 四、阶段 3 — Core 调度与任务互斥

`core/src/session/handlers.rs:515` `submission_loop` 是单消费者循环。**26 个 Op 分支已全部核实**（`:527-667`）：

| 行号 | Op | 用途 |
| --- | --- | --- |
| 527 | `Interrupt` | 中断当前 turn |
| 531 | `CleanBackgroundTerminals` | 清理后台终端 |
| 535-562 | `RealtimeConversationStart/Audio/Text/Speech/Close` | 实时语音会话 |
| 566 | `RealtimeConversationListVoices` | 枚举音色 |
| **570** | **`TurnInput`** | **主输入入口** |
| 579 | `RecoverTurn` | 恢复异常中断的轮次 |
| 588 | `ThreadSettings` | 修改会话设置 |
| 592 | `InterAgentCommunication` | 多智能体通信 |
| 603 | `ExecApproval` | 命令执行审批回执 |
| 611 | `PatchApproval` | 补丁审批回执 |
| 615 | `UserInputAnswer` | 工具反问的答复 |
| 619 | `RequestPermissionsResponse` | 权限申请答复 |
| 623 | `DynamicToolResponse` | 动态工具答复 |
| 627 | `RefreshMcpServers` | 重连 MCP |
| 631 | `ReloadUserConfig` | 热加载配置 |
| 635 | `Compact` | 显式压缩上下文 |
| 639 | `ThreadRollback` | 回滚 N 轮 |
| 643 | `SetThreadMemoryMode` | 记忆模式 |
| 647 | `RunUserShellCommand` | `!cmd` 本地执行 |
| 651 | `ResolveElicitation` | MCP elicitation 答复 |
| 662 | `Shutdown` | 关停会话 |
| 663 | `Review` | 进入评审模式 |
| 667 | `ApproveGuardianDeniedAction` | Guardian 拦截放行 |

通道关闭时执行 `shutdown_session_runtime()` + `emit_thread_stop_lifecycle()`。

### 任务派发

需要长时间运行的 Op 交给 `tasks/mod.rs:279` `spawn_task<T: SessionTask>`（trait 在 `:187`），四种实现：

- `tasks/regular.rs:92` — 普通对话轮次
- `tasks/compact.rs:86` — 上下文压缩
- `tasks/review.rs:276` — 代码评审模式
- `tasks/user_shell.rs:475` — `!cmd` 本地执行

**同一时刻只允许一个活跃 turn。** 抢占由 `abort_all_tasks(reason: TurnAbortReason)` (`:494`) 与 `abort_turn_if_active` (`:524`) 完成，各任务的 `SessionTask::abort` (`:218` / `:242` / `:273`) 负责自身清理。TUI 侧中断触发点 `chatwidget/interaction.rs:129-154` `interrupt_turn`，键位绑定 `keymap/bindings.rs:239`。

turn 进行中的新输入走 `InputResult::Queued` 进入 `input_queue`，在采样循环的安全点被吸收 —— 这就是 Codex 的 **steering** 机制。

---

## 五、阶段 4 — run_turn 前置流程

`core/src/session/turn.rs:153`：

```rust
pub(crate) async fn run_turn(
    sess: Arc<Session>,
    turn_context: Arc<TurnContext>,
    input: Vec<TurnInput>,
    prewarmed_client_session: Option<ModelClientSession>,
    cancellation_token: CancellationToken,
) -> CodexResult<Option<String>>
```

`turn.rs:140-152` 的文档注释精确定义了循环语义：模型要么返回函数调用（执行后把输出喂回下一次采样），要么只返回助手消息（记入历史，turn 结束）。

### 进入主循环前的严格顺序

1. `:161` `drain_async_hook_results(..., before_user_prompt = true)` — 消费异步 hook 结果
2. `:163` 取 `prewarmed_client_session`，否则 `model_client.new_session()`（`client.rs:503`）
3. `:169` `run_pre_sampling_compact` — 采样前压缩；`TurnAborted` / `ToolCollision` 特判，其他错误走 `emit_turn_error_lifecycle`
4. `:192` `turn_user_input(&input)` 规范化输入
5. `:193` `required_mcp_servers_for_input` — 推导本轮所需 MCP server（按需连接）
6. `:207` `capture_step_context_with_required_mcp_servers` — 冻结 `StepContext`
7. `:224` `tokio::join!(record_context_updates_and_set_reference_context_item, turn_diff_display_roots)`，后者受 `Feature::CwdRelativeTurnDiffs` 门控
8. `:250` `build_skills_and_plugins`
9. `:262` `run_pending_session_start_hooks`
10. `:266` `run_hooks_and_record_inputs(..., PersistContext::TurnStart)`
11. `:270` `merge_connector_selection`
12. `:272` `set_previous_turn_settings(PreviousTurnSettings { model, comp_hash, realtime_active })`
13. `:278` 逐条 `record_conversation_items` 写入注入项
14. `:283` `track_turn_resolved_config_analytics`
15. `:289` `TurnDiffTracker::with_environment_display_roots(...)`（`turn_diff_tracker.rs:84`）包进 `Arc<tokio::sync::Mutex<_>>`

### 三层状态快照

- **Session**（`session/session.rs`）— 会话级
- **TurnContext**（`session/turn_context.rs:144`）— 轮次级：cwd、模型、审批策略、沙箱策略
- **StepContext**（`session/step_context.rs`）— 单次采样级

分层的意义：轮次中途切模型或改策略，不会污染已经开始的采样步骤。

### 上下文片段注入

`core/src/context/` 下约 40 个 `ContextualUserFragment` 模块按需拼入 prompt：

`environment_context.rs`、`user_instructions.rs`、`permissions_instructions.rs`、`environments_instructions.rs`、`apps_instructions.rs`、`plugin_instructions.rs`、`personality_spec_instructions.rs`、`model_switch_instructions.rs`、`current_time_reminder.rs`、`token_budget_context.rs`、`turn_aborted.rs`、`image_resize_notice.rs`、`approved_command_prefix_saved.rs`、`network_rule_saved.rs`、`guardian_*`、`multi_agent_*`、`realtime_*`、`world_state/`。

---

## 六、阶段 5 — 采样循环

主循环 `turn.rs:301`。每次迭代：

1. `:305` 若 `can_drain_pending_input`，从 `input_queue.get_pending_input(&sess.active_turn)` 吸收排队输入（**steering 生效点**）
2. `:314` 运行 hooks
3. `:326` `rollout_budget::maybe_record_reminder`
4. `:334-339` take / capture `step_context`
5. `:1312` `build_prompt`
6. `:1448` `prepare_tool_recommendations` + `:1494` `built_tools` 决定本步可见工具集
7. `:1340` `run_sampling_request`，其内层 `:1368` 是重试循环；单次尝试在 `:2179` `try_run_sampling_request`
8. 消费流式 `ResponseEvent`
9. **有工具调用** → 执行 → 输出回灌 → 回到第 1 步；**仅文本** → turn 结束

### 请求构造

`client_common.rs:17-50`：

```rust
Prompt {
    input: Vec<ResponseItem>,
    tools: Arc<[ToolSpec]>,
    parallel_tool_calls: bool,
    base_instructions: BaseInstructions,
    output_schema: Option<Value>,
    output_schema_strict: bool,
}
```

- `ModelClient`（`client.rs:255`）会话级
- `ModelClientSession`（`client.rs:275`）轮次级，**懒建 WebSocket**，缓存 `x-codex-turn-state` 头
- `new_session()` `client.rs:503`，`build_responses_request()` `client.rs:867`

### 线路格式

只剩一种：`model-provider-info/src/lib.rs:64`

```rust
pub enum WireApi {
    /// The Responses API exposed by OpenAI at `/v1/responses`.
    #[default]
    Responses,
}
```

**Chat Completions 已彻底移除。**

请求体 `ResponsesApiRequest`（`codex-api/src/common.rs:252`）字段：`model, instructions, input, tools, tool_choice, parallel_tool_calls, reasoning, store, stream, stream_options, include, service_tier, prompt_cache_key, text, client_metadata`。

WebSocket 变体 `ResponseCreateWsRequest<'a>`（`codex-api/src/common.rs:302`）额外带 `previous_response_id`。

### 流式事件

`ResponseEvent`（`codex-api/src/common.rs:76`）：

`Created`、`SafetyBuffering`、`OutputItemAdded`、`OutputItemDone`、`ServerModel`、`ModelVerifications`、`TurnModerationMetadata`、`ServerReasoningIncluded`、`OutputTextDelta`、`ToolCallInputDelta`、`ReasoningSummaryPartAdded`、`ReasoningSummaryDelta`、`ReasoningSummaryDone`、`ReasoningContentDelta`、`RateLimits`、`ModelsEtag`、`Completed { response_id, token_usage, end_turn }`。

SSE 路径：`codex-api/src/sse/responses.rs:38-88` `spawn_response_stream`，逐事件解析 `:348-500` `process_responses_event()`。

turn 侧的流式渲染分支：`maybe_emit_pending_agent_message_start` (`:1723`)、`emit_streamed_assistant_text_delta` (`:1917`)、`flush_assistant_text_segments_for_item` (`:1952`) / `_all` (`:1964`)、`handle_plan_segments` (`:1854`)、`assign_missing_streamed_response_item_id` (`:2156`)、`drain_in_flight` (`:2130`)。

**Plan 模式**是独立分支：`maybe_complete_plan_item_from_message` (`:1983`)、`emit_agent_message_in_plan_mode` (`:2012`)、`emit_turn_item_in_plan_mode` (`:2056`)、`handle_assistant_item_done_in_plan_mode` (`:2077`)。

### 重试与超时默认值

- `DEFAULT_STREAM_IDLE_TIMEOUT_MS = 300_000`
- `DEFAULT_STREAM_MAX_RETRIES = 5`
- `DEFAULT_REQUEST_MAX_RETRIES = 4`
- `DEFAULT_WEBSOCKET_CONNECT_TIMEOUT_MS = 15_000`（超时后回落 HTTP）
- `MAX_ANALYTICS_TOOL_CALL_IDS_PER_RESPONSE = 256`（`turn.rs:2235`）

---

## 七、阶段 6 — 工具分发

`core/src/tools/router.rs`（290 行）。`build_tool_call` (`:148`) 把 `ResponseItem` 翻译成 `ToolCall`：

- `ResponseItem::FunctionCall { name, namespace, arguments, encrypted_function_args, call_id }` → `ToolName::new(namespace, name).with_default_namespace()` + `ToolPayload::Function { arguments }`
- `ResponseItem::ToolSearchCall` 且 `execution == "client"` → `ToolName::plain("tool_search")` + `ToolPayload::ToolSearch`
- `ResponseItem::CustomToolCall` → `ToolPayload::Custom { input }`
- 其他 → `Ok(None)`，走纯文本路径

`ToolCall::direct_source()` (`:40`) 对 `collaboration` 命名空间的 `spawn_agent` / `send_message` / `followup_task` 特判为 `ToolCallSource::DirectPlaintextMessage`。

`dispatch_tool_call_with_code_mode_result_inner` (`:250`) 构造：

```rust
ToolInvocation {
    session, turn, step_context, cancellation_token,
    tracker, call_id, tool_name, source, payload,
}   // :269
```

交给 `registry.dispatch_any_with_terminal_outcome(invocation, terminal_outcome_reached)` (`:281`)。

并行 / 运行时策略：`tool_supports_parallel` (`:137`)、`tool_runtime` (`:143`)、`deferred_tool_namespaces` (`:113`)；diff 采集挂接 `create_diff_consumer` (`:130`)。编排在 `tools/orchestrator.rs`（553 行），注册表 `tools/registry.rs`（823 行）。

### Handler 全清单（`core/src/tools/handlers/`）

| 类别 | 文件 |
| --- | --- |
| 执行 | `unified_exec.rs`、`shell_spec.rs`、`sleep.rs`、`wait_for_environment.rs` |
| 编辑 | `apply_patch.rs`（+ `apply_patch.lark` 语法、`apply_patch_spec.rs`） |
| MCP | `mcp.rs`、`mcp_resource.rs` |
| 多智能体 | `multi_agents.rs`、`multi_agents_v2.rs`、`send_user_message_async.rs` |
| 反问用户 | `request_permissions.rs`、`request_user_input.rs`、`request_plugin_install.rs` |
| 上下文自省 | `get_context_remaining.rs`、`new_context_window.rs`、`current_time.rs` |
| 发现 | `tool_search.rs`、`list_available_plugins_to_install.rs` |
| 其他 | `plan.rs`、`view_image.rs`、`dynamic.rs`、`extension_tools.rs`、`test_sync.rs` |

---

## 八、阶段 7 — 审批与沙箱

这是分支最密集的环节。三个正交维度共同决定一条命令能否执行。

### 维度 1：审批策略 `AskForApproval`（`protocol.rs:917-940`）

- `UnlessTrusted` (`:922`) — 仅信任列表内免批
- `OnRequest` (`:927`) — 模型主动请求时才询问
- `Granular` (`:942-957`) — 细粒度控制
- `Never` (`:939`) — 从不询问

### 维度 2：沙箱策略 `SandboxPolicy`（`protocol.rs:1003-1051`）

- `DangerFullAccess` (`:1006`)
- `ReadOnly` (`:1010`)
- `ExternalSandbox`
- `WorkspaceWrite` — 可写根 + 网络开关（`NetworkAccess::{Restricted, Enabled}`，`:989-990`）

### 维度 3：安全判定 `SafetyCheck`（`core/src/safety.rs:20`）

- `AutoApprove`
- `AskUser`
- `Reject { .. }`

补丁专用评估 `assess_patch_safety` (`safety.rs:26`)，配合 `is_write_patch_constrained_to_writable_paths` (`:121`) 校验越界写入，拒绝理由由 `patch_rejection_reason` (`:100`) 生成。

命令级判定在 `core/src/exec_policy.rs:697-816`：`dangerous_command_match_for_origin`、`dangerous_command_match_for_heuristics`、`render_decision_for_unmatched_command`（产出 Allow / Prompt / Forbidden），`profile_has_managed_filesystem_restrictions` (`:817`)。

### 执行参数

`core/src/exec.rs`：

- `ExecParams` (`:95`)
- `DEFAULT_EXEC_COMMAND_TIMEOUT_MS = 10_000` (`:61`)
- `ExecCapturePolicy { ShellTool, FullBuffer }` (`:111`)
- `ExecExpiration { Timeout, DefaultTimeout, Cancellation, TimeoutOrCancellation }` (`:145`)
- `ExecExpirationOutcome` (`:157`)
- 输出按 `EXEC_OUTPUT_MAX_BYTES` 截断

起进程：`exec-server/src/local_process.rs:732` `LocalProcess::start()`（`start_process()` `:273-370`，`prepare_exec_request()` `:345`），沙箱包装 `exec-server/src/process_sandbox.rs`。

### 平台沙箱实现

- **macOS**：`sandboxing/src/seatbelt.rs` — `/usr/bin/sandbox-exec` + `seatbelt_base_policy.sbpl`，参数由 `create_seatbelt_command_args` 生成
- **Linux**：`linux-sandbox/src/bwrap.rs:60/87`（bubblewrap，`BwrapNetworkMode::{FullAccess, Isolated, ProxyOnly}`）+ `linux-sandbox/src/landlock.rs:42/68/84`（Landlock LSM + seccomp）
- **Windows**：`core/src/windows_sandbox.rs:24-248` — 受限令牌

**关键分支**：沙箱内命令失败 → 判断是否由沙箱限制导致 → 若是，向用户申请提权重试；用户拒绝则把失败原因回灌模型，由模型换方案。

---

## 九、阶段 8 — apply_patch 与 diff 追踪

`apply-patch/src/lib.rs` 中调用链全部为 async：

```
apply_patch() :339
  → apply_patch_with_options() :361
    → apply_hunks() :401
      → apply_hunks_with_options() :423
```

数据结构：`ApplyPatchOptions` (`:74`)、`ApplyPatchArgs` (`:152`)、`ApplyPatchFileChange` (`:160`)、`MaybeApplyPatchVerified` (`:176`)、`ApplyPatchAction` (`:193`)、`AppliedPatchDelta` (`:247`)、`AppliedPatchChange` (`:288`)、`AppliedPatchFileChange` (`:294`)、`ApplyPatchFailure` (`:314`)、`AffectedPaths` (`:462`)；错误类型 `ApplyPatchError` (`:99`)、`IoError` (`:137`)；文件更新模式 `ApplyPatchFileUpdateMode` (`:64`) 由 `apply_patch_file_update_mode_from_env()` (`:91`) 解析。补丁语法由 `handlers/apply_patch.lark` 定义。

每轮累计变更由 `core/src/turn_diff_tracker.rs:49` `TurnDiffTracker` 维护：

- `with_environment_display_roots(...)` (`:84`)
- `track_delta(&mut self, environment_id: &str, delta: &AppliedPatchDelta)` (`:92`)
- `invalidate(&mut self)` (`:108`)
- `get_unified_diff(&self) -> Option<String>` (`:114`)
- `DIFF_TIMEOUT = 100ms` (`:17`，作用于 `:357`)，防止大仓库计算 diff 时卡顿

---

## 十、阶段 9 — 上下文管理与压缩

四种压缩时机：

1. **采样前压缩** — `run_pre_sampling_compact` (`turn.rs:1012`)
2. **模型切换内联压缩** — `maybe_run_previous_model_inline_compact` (`turn.rs:1080`)，用旧模型压缩再切换
3. **自动压缩** — `run_auto_compact` (`turn.rs:1178`)，token 逼近上限时触发
4. **显式压缩** — `/compact` → `Op::Compact` (`handlers.rs:635`) → `CompactTask`

辅助：`comp_hash_changed` (`turn.rs:1045`) 检测配置变更、`capture_current_model_fallback_step_context` (`:1055`) 模型回退快照。

预算模块：`session/token_budget.rs`、`session/context_window.rs`、`session/rollout_budget.rs`。模型可主动调用 `get_context_remaining` / `new_context_window` 工具自省剩余预算。

回滚：`Op::ThreadRollback { num_turns }` (`handlers.rs:639`) 丢弃最近 N 轮。

---

## 十一、阶段 10 — MCP 集成

`codex-mcp/src/connection_manager.rs`：`McpConnectionSet::new()` (`:193-291`)、`McpServerConnection` (`:79-87`)、集合 API (`:720-840`)。

要点：

- 工具名做**命名空间隔离**（`ToolName::new(namespace, name)`），避免不同 server 同名工具冲突
- **按需连接**：`required_mcp_servers_for_input` (`turn.rs:193`) 先算出本轮所需 server，只连这些
- 预热 / 刷新：`session/mcp_runtime.rs`、`session/mcp_prewarm.rs`、`session/mcp_refresh.rs`；`Op::RefreshMcpServers` (`handlers.rs:627`)
- **Elicitation**：MCP server 反向向用户提问 → 事件上抛 → 用户答复经 `Op::ResolveElicitation` (`handlers.rs:651`) 回注

---

## 十二、阶段 11 — 中断、错误、限流

- **用户中断**：`Op::Interrupt` (`handlers.rs:527`) → `abort_all_tasks` (`tasks/mod.rs:494`)；被中断的事实通过 `context/turn_aborted.rs` 片段告知模型，下一轮上下文中可见
- **网络 / 流错误**：`run_sampling_request` 内层重试 (`turn.rs:1368`)，退避策略在 `responses_retry.rs`
- **工具错误**：`FunctionCallError` 作为工具输出回灌模型，**不终止 turn**
- **限流**：`ResponseEvent::RateLimits` → `RateLimitSnapshot` → TUI 展示剩余额度
- **Token 统计**：`Completed { token_usage }` 累计到 `TokenUsage`，元数据解析在 `responses_metadata.rs`
- **Guardian**：拦截可疑动作后，用户可通过 `Op::ApproveGuardianDeniedAction` (`handlers.rs:667`) 放行

---

## 十三、阶段 12 — 持久化与恢复

`rollout` crate 把每轮的 `ResponseItem` 序列落盘，支撑三件事：

- **会话恢复**：重放 rollout 重建历史（`session/rollout_reconstruction.rs`）
- **轮次恢复**：`Op::RecoverTurn` (`handlers.rs:579`) 恢复异常打断的轮次
- **提示缓存**：`prompt_cache_key` + `store` 让服务端复用前缀；`ModelClientSession` 缓存 `x-codex-turn-state`（`client.rs:275`）

---

## 十四、非主线流程速查

| 用户动作 | 处理路径 |
| --- | --- |
| `!ls -la` | `input_submission.rs:153` → `Op::RunUserShellCommand` → `UserShellTask`，不经模型 |
| `/compact` | `Op::Compact` (`handlers.rs:635`) → `CompactTask` |
| `/review` | `Op::Review` (`handlers.rs:663`) → `ReviewTask`（`tasks/review.rs:276`） |
| 纯本地斜杠命令 | TUI 内消化，不发 `Op` |
| 语音会话 | `Op::RealtimeConversation*` (`handlers.rs:535-566`)，独立于文本采样循环 |
| 多智能体协作 | `Op::InterAgentCommunication` (`:592`) + `handlers/multi_agents*.rs` |
| 工具请求权限 | `handlers/request_permissions.rs` → 事件上抛 → `Op::RequestPermissionsResponse` (`:619`) |
| 工具反问用户 | `handlers/request_user_input.rs` → `Op::UserInputAnswer` (`:615`) |
| 安装插件 | `handlers/request_plugin_install.rs` → `Op::DynamicToolResponse` (`:623`) |
| 配置热更新 | `Op::ReloadUserConfig` (`:631`) |
| 回滚对话 | `Op::ThreadRollback` (`:639`) |
| 恢复中断轮次 | `Op::RecoverTurn` (`:579`) |

---

## 十五、规模参考

实测行数：

- `protocol/src/protocol.rs` — 6033
- `core/src/session/turn.rs` — 2791
- `core/src/client.rs` — 2556
- `core/src/tasks/mod.rs` — 978
- `core/src/tools/registry.rs` — 823
- `core/src/session/handlers.rs` — 763
- `core/src/tools/orchestrator.rs` — 553
- `core/src/tools/router.rs` — 289

整个 `codex-rs` workspace 约 140 个 crate，统一 `codex-` 前缀。

---

## 十六、验证边界

**已直接读源码核对**：`protocol.rs:540-699`、`session/turn.rs:140-339`、`tools/router.rs` 全文，以及 `handlers.rs` 全部 26 个 Op 分支、`tasks/mod.rs`、`safety.rs`、`client.rs`、`codex-api/common.rs`、`exec.rs`、`apply-patch/lib.rs`、`turn_diff_tracker.rs`、`model-provider-info/lib.rs`、`input_submission.rs` 的符号行号。

**本文修正的错误行号**（早期检索给出的值有偏差）：

- `build_responses_request` 实为 `client.rs:867`（非 588）
- `new_session` 在 `client.rs:503`
- `ResponseEvent` 在 `codex-api/common.rs:76`，`ResponsesApiRequest` 在 `:252`，`ResponseCreateWsRequest` 在 `:302`
- `DEFAULT_EXEC_COMMAND_TIMEOUT_MS` 在 `exec.rs:61`，不在 `ExecParams` 附近
- `WireApi` 在 `model-provider-info/lib.rs:64`
- `apply_patch` 系列全部是 `async fn`，且调用链中间还有一层 `apply_hunks` (`:401`)

**仍未逐行复核**：`exec-server/`、`sandboxing/`、`linux-sandbox/`、`codex-mcp/`、`tui/` 中的部分行号 —— 结构判断可靠，个别行号可能偏移几行。











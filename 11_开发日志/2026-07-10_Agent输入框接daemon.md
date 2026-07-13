# 2026-07-10 Agent 输入框接 daemon

## 本次修改背景

第八步已经让 daemon 可以通过 Codex CLI 生成 `h5-artifact.draft.json`。但二级项目页左侧 Agent 面板底部输入框仍然只是静态 UI：用户输入内容后不会发给 daemon，也不会触发 Codex / runner，更不会产生可追踪的 AgentEvent。

本轮目标是把这个输入框接入现有 daemon / Runner Adapter / SSE 链路，让它成为未来接 Codex CLI、Claude CLI 和原型修改 runner 的入口。

本轮不接真实生图，不重做 UI，不推翻 Home + Project Workspace 两级结构，不默认覆盖正式 `h5-artifact.json`。

## 本次目标

- 新增 `POST /api/agent/messages`。
- 前端左侧 Agent 输入框点击发送后调用 daemon。
- daemon 根据 intent 选择 runner。
- 第一版默认 intent 为 `probe_message`。
- Codex CLI 读取 `current/brief.md` 和用户输入 message。
- 生成 `current/agent-message-result.md`。
- 通过 SSE 回推用户消息、Codex 处理状态、文件写入和完成事件。
- 如果 daemon 不可用，前端明确显示 fallback 错误，不再静默无反应。

## 修改文件

- `10_产品代码包/packages/agent-contract/src/index.ts`
  - AgentEvent 新增 `user_message`。
  - FileKind 新增 `agent_note`。
  - 新增 `AgentMessageIntent` 和 `AgentMessageRequest`。
- `10_产品代码包/apps/daemon/src/server.ts`
  - 新增 `POST /api/agent/messages`。
  - 新增 message / intent 解析。
  - `startRunnerRun()` 支持把 message 和 intent 传入 RunnerContext。
- `10_产品代码包/apps/daemon/src/runners/types.ts`
  - RunnerContext 新增 `input` 字段。
- `10_产品代码包/apps/daemon/src/runners/codexCliRunner.ts`
  - `codex_cli_probe` 支持带 message 的 Agent 输入模式。
  - 新增 `agent-message-result.md` 输出。
  - 新增 `file_agent_message_result` produced file。
- `10_产品代码包/apps/daemon/src/projectStore.ts`
  - 支持读取 `file_agent_message_result`。
  - 新增 `agent-message-result.md` 到当前项目文件映射。
- `10_产品代码包/apps/web/src/api/daemonClient.ts`
  - 新增 `sendAgentMessage(message, intent)`。
- `10_产品代码包/apps/web/src/App.tsx`
  - 新增 Agent 输入发送状态。
  - 发送后启动 SSE，追加 daemon 推来的事件。
  - daemon 不可用时显示明确 fallback。
- `10_产品代码包/apps/web/src/components/ProjectShell.tsx`
  - 透传 Agent 输入发送 props。
- `10_产品代码包/apps/web/src/components/AgentRunPanel.tsx`
  - 底部输入框改为可提交表单。
  - 显示用户消息、sending 状态和错误提示。
- `10_产品代码包/apps/web/src/components/DesignFilesView.tsx`
  - 支持 `agent_note` 文件类型文案。
- `10_产品代码包/apps/web/src/styles.css`
  - 补充用户消息、输入禁用态和错误提示样式。
- `11_开发日志/00_当前开发状态.md`
  - 更新当前阶段状态。
- `11_开发日志/2026-07-10_Agent输入框接daemon.md`
  - 新增本日志。

## 新增 API

```http
POST /api/agent/messages
Content-Type: application/json

{
  "message": "把奖励模块往前放，整体更有活动氛围",
  "intent": "probe_message"
}
```

返回：

```json
{
  "runId": "mock-run-...",
  "runType": "codex_cli_probe",
  "projectId": "...",
  "status": "running"
}
```

当前 intent 映射：

- `probe_message` -> `codex_cli_probe`
- `artifact_draft` -> `codex_artifact_draft`
- `long_image_generation` -> `long_image_generation`

第一版前端默认使用 `probe_message`。

## AgentEvent 流

发送左侧输入后，daemon 会追加并通过 SSE 推送：

- `user_message`：用户输入内容。
- `run_started`：开始处理左侧 Agent 输入。
- `assistant_message`：开始调用 Codex CLI。
- `raw_log`：Codex CLI version / debug 信息。
- `file_written`：`agent-message-result.md`。
- `assistant_message`：Codex 生成的建议正文。
- `run_finished`：本轮 message run 完成。
- `error`：Codex 不存在、执行失败或输出失败时使用。

前端只消费标准化 AgentEvent，不直接解析 CLI 文本；raw log 仍只放在调试折叠区。

## 生成文件

新增文件：

```text
10_产品代码包/.h5-workbench/projects/current/agent-message-result.md
```

文件用途：

- 记录用户本次想改什么。
- 判断对 H5 原型的影响。
- 给出建议下一步。
- 判断是否需要生成 artifact draft。

FileManifest 新增项：

```text
fileId: file_agent_message_result
filename: agent-message-result.md
kind: agent_note
status: ready
relatedStage: prototype_review
openTarget: design-files
```

正式 `h5-artifact.json` 和 `prototype.html` 本轮不会被修改。

## 前端行为

用户在左侧输入：

```text
把奖励模块往前放，整体更有活动氛围
```

页面会出现：

- 用户消息立即显示在 Agent 输出区。
- 输入框清空，发送按钮进入处理中状态。
- Agent 输出显示开始调用 Codex CLI。
- Codex 返回建议后显示在 Agent 输出区。
- Produced Files 中出现 `agent-message-result.md`。
- 右侧「设计文件」tab 的文件数增加，并显示 `Markdown / Agent Note`。
- run finished 后输入框恢复可用。

daemon 不可用时：

- 页面不会崩溃。
- 左侧显示「daemon 未连接，当前无法发送给 Codex」。
- 这类 fallback 不写入本地项目目录。

## 本轮验证结果

- `pnpm typecheck` 通过。
- `pnpm --filter @h5-workbench/web build` 通过。
- `GET /api/health` 正常。
- `POST /api/agent/messages` 可启动 `codex_cli_probe` message run。
- SSE / events JSON 能看到：
  - `user_message`
  - `run_started`
  - `assistant_message`
  - `raw_log`
  - `file_written: agent-message-result.md`
  - `run_finished`
- 已生成：`10_产品代码包/.h5-workbench/projects/current/agent-message-result.md`。
- `GET /api/projects/current/files/file_agent_message_result` 能读取文件内容。
- `long_image_generation` 仍可启动并跑完。
- `codex_cli_probe` 仍可独立启动并生成 `codex-probe-result.md`。
- 浏览器验证通过：从 Home 进入项目页后，左侧输入框发送内容会回显用户消息、Codex 建议和 produced file。
- 前端 `http://127.0.0.1:5178/` 仍可访问。

## 尚未完成

- 左侧输入暂时默认走 `probe_message`，还没有前端 intent 选择。
- 还没有把 Agent 建议转成正式 artifact patch。
- 还没有「采用建议」或「生成草稿」的按钮链路。
- 还没有文件详情预览，因此 `agent-message-result.md` 只能在 Produced Files / 设计文件列表看到，不能在右侧直接展开正文。
- 还没有真实生图、真实导出或旧 Python runner 接入。
- Codex stderr 中仍可能出现 plugin/config warning，目前作为 raw log 保留。

## 下一步建议

1. 做右侧文件详情预览，优先支持 `agent-message-result.md`、`h5-artifact.draft.json`、`codex-probe-result.md`。
2. 给左侧 Agent 输出增加快捷操作，例如「生成 Artifact 草稿」「采用建议生成 patch」「进入原型预览」。
3. 设计 `artifact_patch` runType，让 Codex 只输出结构化 patch，人工确认后再修改正式 `h5-artifact.json`。
4. 把素材/KV 上传从前端 mock 迁移到 daemon 的 `assets/` 目录。
5. 继续保持 runner 小步接入，真实生图仍应等原型确认和文件协议稳定后再做。

## Guardian 结论

本轮没有跑偏。左侧输入框已成为「前端 Agent 面板 -> local daemon -> Codex CLI runner -> 文件/事件 -> 前端 SSE」链路的一部分，但仍保持产品化 Agent 面板形态，没有把前端变成终端，也没有让 Codex 直接修改正式 H5 原型或接真实生图。

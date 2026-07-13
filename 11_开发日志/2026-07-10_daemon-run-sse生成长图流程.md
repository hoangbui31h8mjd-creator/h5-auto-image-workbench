# 2026-07-10 daemon run + SSE 生成长图流程

## 本次修改背景

第三步已经完成 mock daemon 接口层，前端可以从 daemon 获取 current project、FileManifest、H5Artifact、RunState、ImageSlot 和 AgentEvent。

但「生成长图」按钮仍主要由前端本地 `setTimeout` 推进状态。这样不利于后续接 Codex CLI、Claude CLI 或旧 Python runner，因为真实执行应位于 daemon 层，前端只消费结构化事件和状态。

## 本次目标

- 不接真实 Codex CLI。
- 不接真实生图模型或旧 Python runner。
- 不重做 UI，不推翻一级 Home + 二级 Project Workspace。
- 新增 daemon run 启动接口。
- 让 daemon 维护 mock long image run 的 RunState 和 FileManifest。
- 让 SSE 推送 AgentEvent、RunState、FileManifest 和 project state。
- 前端点击「生成长图」后由 daemon run 驱动状态变化。
- daemon 不可用时保留前端本地 fallback，不让页面崩掉。

## 修改文件

- `10_产品代码包/packages/agent-contract/src/index.ts`：新增 `RunType`、`RunStartRequest`、`RunStartResponse`、`RunStreamEvent`。
- `10_产品代码包/apps/daemon/src/server.ts`：新增 long image mock run 调度器、`POST /api/runs/current/start`、SSE 状态广播、运行中 RunState/FileManifest 更新。
- `10_产品代码包/apps/web/src/api/daemonClient.ts`：新增 `startRun()`，扩展 `subscribeAgentEvents()` 支持 `run_state`、`file_manifest`、`project_state`。
- `10_产品代码包/apps/web/src/App.tsx`：将「生成长图」按钮改为优先调用 daemon run，并通过 SSE 更新 Agent 面板、长图流程、设计文件状态；本地 `setTimeout` 只作为 daemon 不可用时的 fallback。
- `11_开发日志/00_当前开发状态.md`：更新当前状态、run API、SSE 状态和下一步建议。

## 新增 run API

```text
POST /api/runs/current/start
```

请求体：

```json
{
  "runType": "long_image_generation"
}
```

返回：

```json
{
  "runId": "mock-run-xxx",
  "runType": "long_image_generation",
  "projectId": "daemon_project_xxx",
  "status": "running",
  "runState": {},
  "fileManifest": {}
}
```

## SSE 变化

```text
GET /api/runs/current/events/stream
```

现在会推送：

- `agent_event`：结构化 AgentEvent。
- `run_state`：当前 RunState。
- `file_manifest`：当前 FileManifest。
- `project_state`：完整 mockProjectState。
- `done`：本轮 run 结束。

说明：真实 CLI 接入后，daemon 仍应把 CLI 输出转换成结构化事件，前端不要直接解析 raw CLI 文本。

## 生成长图 mock run 流程

daemon 当前按以下顺序推进：

1. `prototype_review` -> `done`
2. `slice_planning` -> `running` -> `done`
3. `image_generation` -> `running` -> `done`
4. `stitching` -> `running` -> `done`
5. `quality_check` -> `running` -> `done`
6. `export` -> `running` -> `done`

文件状态同步：

- `slice-plan.json`：`running` -> `done`
- `long-image.png`：`running` -> `done`
- `critique.json`：`running` -> `done`
- `export.zip`：`running` -> `done`

项目状态同步：

- 启动 run 时：`artifact.status = generating`
- run 完成时：`artifact.status = generated`

## 前端接入变化

- 点击左侧 Agent 面板或右侧长图 tab 的「生成长图」，会调用 `startRun("long_image_generation")`。
- 启动成功后，前端订阅 SSE。
- 左侧 Agent 面板会随着 `agent_event` 追加执行说明、checklist、file_written、run_finished。
- 右侧「生成长图」tab 会随着 `run_state` 更新阶段状态和进度。
- 右侧「设计文件」tab 会随着 `file_manifest` 更新 `slice-plan.json`、`long-image.png`、`critique.json`、`export.zip`。
- daemon 不可用时，前端 fallback 到本地 mock，并在左侧 Agent 面板显示「daemon 未连接，当前降级为前端本地 mock」。

## 验收结果

### 命令验证

- `pnpm typecheck`：通过。
- `pnpm --filter @h5-workbench/web build`：通过。
- 前端 `http://127.0.0.1:5178/`：HTTP `200 OK`。
- daemon `http://127.0.0.1:7458/api/health`：返回 `ok: true`、`mode: mock`。

### API / SSE 验证

已验证 `POST /api/runs/current/start` 和 SSE 完整流程：

```json
{
  "startStatus": "running",
  "agentEvents": 39,
  "runStateEvents": 11,
  "fileManifestEvents": 11,
  "projectStateEvents": 11,
  "done": true,
  "finalRunStatus": "done",
  "artifactStatus": "generated"
}
```

最终阶段状态：

```json
{
  "intake": "done",
  "prototype_generation": "done",
  "prototype_review": "done",
  "slice_planning": "done",
  "image_generation": "done",
  "stitching": "done",
  "quality_check": "done",
  "export": "done"
}
```

最终文件状态：

```json
{
  "brief.md": "ready",
  "h5-artifact.json": "synced",
  "prototype.html": "draft",
  "critique.json": "done",
  "slice-plan.json": "done",
  "long-image.png": "done",
  "export.zip": "done"
}
```

### 浏览器验证

在 `http://127.0.0.1:5178/` 中验证：

- Home 输入需求后能进入 Project Workspace。
- 点击「生成长图」后，左侧 Agent 面板出现 daemon 接管说明和后续事件。
- 「生成长图」tab 的 6 个阶段最终全部为 `done`，进度为 100%。
- 切回「设计文件」tab 后，`slice-plan.json`、`long-image.png`、`export.zip` 均为 `done`。
- 关闭 daemon 后再次点击「生成长图」，页面不崩，并显示「daemon 未连接，当前降级为前端本地 mock」。
- 验证后已重新启动 daemon。

## 尚未解决

- 当前 daemon run 仍是内存 mock，尚未持久化 run history。
- 当前没有真实写入 `slice-plan.json`、`long-image.png`、`export.zip`。
- 当前没有接入真实 Codex CLI / Claude CLI。
- 当前没有接入旧 Python runner。
- 当前素材/KV 上传仍是前端 mock 状态，未接 daemon 文件系统。

## 下一步建议

1. 增加 daemon 项目目录持久化，把 `brief.md`、`h5-artifact.json`、`prototype.html`、`slice-plan.json` 等写入本地项目目录。
2. 把素材/KV 上传迁移到 daemon，形成真实文件路径和 `ImageSlot.assetPath`。
3. 设计旧 Python runner 适配层：先接切片计划和长图拼接，不搬旧项目整体。
4. 为 run history 增加本地保存和刷新恢复，避免 daemon 重启后丢失当前 run 状态。
5. 接真实 CLI 前继续保持：前端只消费结构化 `AgentEvent`、`RunState`、`FileManifest`。

## 给下一个 Codex 窗口的提示

先读：

- `00_新工作台主入口.md`
- `AGENTS.md`
- `11_开发日志/00_当前开发状态.md`
- `11_开发日志/2026-07-10_mock-daemon接口层.md`
- `11_开发日志/2026-07-10_daemon-run-sse生成长图流程.md`

然后继续：

- 不要推翻现有两级工作台结构。
- 不要重做 UI。
- 不要接真实 Codex CLI。
- 不要接真实生图 runner。
- 下一步优先做项目文件落盘和 run history 持久化。

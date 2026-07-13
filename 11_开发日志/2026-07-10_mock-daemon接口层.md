# 2026-07-10 mock daemon 接口层

## 本次修改背景

第二步「数据协议层」已经完成，前端已有统一 `mockProjectState` 可以驱动 Agent 面板、设计文件、原型预览、生成长图和素材/KV。

本轮进入第三步：建立本地 mock daemon/API 层，让前端未来可以从 daemon 获取项目状态和 AgentEvent，而不是永远直接读取前端本地 mock 数据。

## 本次目标

- 不接真实 Codex CLI。
- 不接真实生图服务或旧 Python runner。
- 不重做 UI，不推翻一级 Home + 二级 Project Workspace。
- 在 `apps/daemon` 提供当前 mock 项目状态 API。
- 在前端新增 daemon API client。
- 前端优先从 daemon 创建/读取项目，daemon 不可用时回退本地 mock。
- 增加 AgentEvent SSE mock 流，为后续 CLI 事件流接入预留通道。

## 修改文件

- `10_产品代码包/packages/agent-contract/src/index.ts`：新增泛型 `WorkbenchProjectState<Project, ImageSlot>`，统一描述前后端传输的项目状态整包。
- `10_产品代码包/apps/daemon/src/mockProjectState.ts`：新增 daemon 侧 mockProjectState 生成器，输出 `project`、`agentEvents`、`fileManifest`、`runState`、`imageSlots`。
- `10_产品代码包/apps/daemon/src/server.ts`：新增 current project API、分包 API、AgentEvent JSON API 和 SSE mock stream。
- `10_产品代码包/apps/web/src/api/daemonClient.ts`：新增前端 daemon client，封装 health、current project、files、artifact、run-state、assets、events 和 SSE。
- `10_产品代码包/apps/web/src/data/mockProjectState.ts`：改用共享 `WorkbenchProjectState` 类型。
- `10_产品代码包/apps/web/src/App.tsx`：Home 生成任务优先请求 mock daemon；失败时回退前端本地 mockProjectState；daemon 成功时订阅 SSE AgentEvent。
- `10_产品代码包/package.json`：新增 `pnpm dev`，可同时启动 web 和 daemon。
- `10_产品代码包/README.md`：补充本地开发命令和 mock daemon API 列表。
- `10_产品代码包/apps/daemon/README.md`：补充 daemon mock API 和启动方式。
- `11_开发日志/00_当前开发状态.md`：更新当前状态、运行方式、接口层状态和下一步建议。

## 新增接口

当前 daemon 地址：

```text
http://127.0.0.1:7458
```

已提供：

```text
GET  /api/health
GET  /api/projects/current
POST /api/projects/current
GET  /api/projects/current/files
GET  /api/projects/current/artifact
GET  /api/projects/current/run-state
GET  /api/projects/current/assets
GET  /api/runs/current/events
GET  /api/runs/current/events/stream
```

说明：

- `/api/projects/current` 返回统一 `mockProjectState`。
- `/api/projects/current/files` 返回 `FileManifest`。
- `/api/projects/current/artifact` 返回 `H5Artifact`。
- `/api/projects/current/run-state` 返回 `RunState`。
- `/api/projects/current/assets` 返回 `ImageSlot[]`。
- `/api/runs/current/events` 返回普通 JSON `AgentEvent[]`。
- `/api/runs/current/events/stream` 返回 SSE mock AgentEvent 流。

## 前端接入变化

- 打开页面仍默认进入 Home，不会直接进入编辑器。
- Home 点击「生成 H5 原型」后，会先调用 `POST /api/projects/current`。
- 如果 daemon 可用，前端用 daemon 返回的 `mockProjectState` 进入 Project Workspace。
- 如果 daemon 不可用，前端回退到原来的本地 `createMockProjectState`，页面不会打不开。
- daemon 可用时，左侧 Agent 面板通过 `/api/runs/current/events/stream` 逐步追加 mock AgentEvent。
- 当前 UI 结构没有重做，仍保持 Home + Project Workspace。

## 验收结果

### 命令验证

- `pnpm typecheck`：通过。
- `pnpm --filter @h5-workbench/web build`：通过。

### API 验证

已验证：

```json
{
  "healthOk": true,
  "healthMode": "mock",
  "createdProject": "游戏活动 H5",
  "currentProject": "游戏活动 H5",
  "files": 7,
  "artifactSections": 4,
  "runStages": 8,
  "assets": 6,
  "events": 17,
  "streamEventCount": 17,
  "streamDone": true
}
```

### 页面验证

- 前端地址 `http://127.0.0.1:5178/` 返回 `200 OK`。
- daemon 地址 `http://127.0.0.1:7458/api/health` 返回 `ok: true` 和 `mode: mock`。
- 当前没有接真实 CLI、真实生图服务或旧 runner。

## 尚未解决

- current mock project 目前主要存在 daemon 进程内存中，尚未做完整本地项目目录持久化。
- 前端长图生成按钮仍主要由前端 setTimeout 推进，尚未迁移到 daemon run API。
- 素材上传仍是前端 mock 状态，没有真实写入文件系统。
- `brief.md`、`h5-artifact.json`、`prototype.html`、`slice-plan.json`、`long-image.png`、`export.zip` 仍是 mock path，未真实落盘。
- 真实 Codex CLI / Claude CLI 未接入。
- 真实旧 Python runner 未接入。

## 下一步建议

1. 增加 daemon run API，例如 `POST /api/runs/current/generate-long-image`，由 daemon 推进 RunState 和 FileManifest，再通过 SSE 通知前端。
2. 增加项目目录持久化，让 current project 的 `brief.md`、`h5-artifact.json`、`prototype.html` 等先真实写入本地。
3. 将素材/KV mock 上传迁移到 daemon，形成真实 `ImageSlot.assetPath`。
4. 设计旧 Python runner 适配层，只接切片、分段生图、拼接、质检必要能力。
5. 接真实 CLI 前保持原则：前端只消费结构化 AgentEvent，不直接解析 raw CLI 文本。

## 给下一个 Codex 窗口的提示

先读：

- `00_新工作台主入口.md`
- `AGENTS.md`
- `11_开发日志/00_当前开发状态.md`
- `11_开发日志/2026-07-10_数据协议层梳理.md`
- `11_开发日志/2026-07-10_mock-daemon接口层.md`

然后继续：

- 不要推翻现有两级工作台结构。
- 不要重做 UI。
- 不要接真实 Codex CLI。
- 不要接真实生图 runner。
- 下一步优先把长图生成流程从前端 mock 推进迁到 daemon run API + SSE。

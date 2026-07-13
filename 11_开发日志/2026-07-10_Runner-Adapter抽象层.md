# 2026-07-10 Runner Adapter 抽象层

## 本次修改背景

第五步已经把当前项目文件和 run history 落盘到 `.h5-workbench/projects/current/`。下一步要为真实 Codex CLI、Claude CLI 或旧 Python runner 接入做准备，但现在不能直接调用真实 CLI，也不能把长图生成逻辑继续塞在 `server.ts` 里。

因此本轮目标是先抽出 daemon runner 层，让 server 只负责 API、状态应用、持久化和 SSE。

## 本次目标

- 不接真实 Codex CLI。
- 不接真实生图服务。
- 不重做 UI。
- 把当前 `long_image_generation` mock run 从 `server.ts` 迁移到统一 Runner 接口。
- 保持当前前端点击「生成长图」后的功能不变。
- 保持 SSE、run-state 落盘、events.jsonl 落盘、files.manifest.json 落盘不变。

## 修改文件

- `10_产品代码包/apps/daemon/src/runners/types.ts`：新增 `Runner`、`RunnerContext`、`RunnerEvent`、`RunnerEmit`、`RunnerHandle`、`RunnerRuntimeStatus`。
- `10_产品代码包/apps/daemon/src/runners/mockLongImageRunner.ts`：承接当前 mock 长图流程，分阶段产出 `AgentEvent`、`RunState`、`FileManifest`、项目状态和完成事件。
- `10_产品代码包/apps/daemon/src/runners/index.ts`：新增 runner 注册表和 `getRunner(runType)`。
- `10_产品代码包/apps/daemon/src/runners/codexCliRunner.ts`：新增 Codex CLI runner skeleton，只写未来接入说明，不执行命令。
- `10_产品代码包/apps/daemon/src/server.ts`：删除长图阶段编排细节，改为通过 `getRunner(runType)` 启动 runner，并统一应用 `RunnerEvent`。
- `11_开发日志/00_当前开发状态.md`：更新当前阶段状态。
- `11_开发日志/2026-07-10_Runner-Adapter抽象层.md`：新增本日志。

## 新增模块说明

- Runner 接口：用于把不同执行引擎统一成 daemon 内部可消费的事件源。
- RunnerContext：包含 `runId`、`runType`、`startedAt`、`projectDir`、当前项目状态和 abort signal。
- RunnerEvent：包括 `agent_event`、`run_state`、`file_manifest`、`project_status`、`state_snapshot`、`done`。
- mockLongImageRunner：当前唯一注册 runner，继续模拟确认原型、生成切片计划、分段生图、拼接长图、质量检查、导出长图。
- codexCliRunner skeleton：只说明未来 Codex CLI 应在 daemon 后面执行，并把输出归一化为结构化事件。

## 当前运行关系

```text
POST /api/runs/current/start
-> server.ts 读取 runType
-> getRunner(runType)
-> mockLongImageRunner.start(context, emit)
-> server.ts applyRunnerEvent
-> projectStore 写入 run-state / file manifest / events.jsonl
-> SSE 推送给前端
```

## 验收结果

- `pnpm typecheck` 通过。
- `pnpm --filter @h5-workbench/web build` 通过。
- `GET /api/health` 正常。
- `POST /api/runs/current/start` 正常启动 `long_image_generation`。
- SSE 正常推送 `agent_event`、`run_state`、`file_manifest`、`project_state` 和 `done`。
- mock run 完成后，`run-state.json` 中状态为 `done`。
- mock run 完成后，`slice-plan.json`、`long-image.png`、`export.zip` 的 FileManifest 状态为 `done`。
- `events.jsonl` 里能看到 `mockLongImageRunner` 写入的事件。
- daemon 重启后，`/api/projects/current` 能恢复最新 run 状态和事件。
- 前端 `http://127.0.0.1:5178/` 仍能访问，前端生成长图按钮仍使用同一个 daemon `startRun + SSE` 链路。

## 尚未解决

- Codex CLI 仍未接入。
- Claude CLI 仍未接入。
- 旧 Python runner 仍未接入。
- `codexCliRunner.ts` 当前只是 skeleton，不会 spawn 任何命令。
- 当前仍只有一个注册 runner：`mockLongImageRunner`。
- 未新增前端 UI，本轮只调整 daemon 内部执行层。

## 下一步建议

1. 先补素材/KV 上传到 daemon 的文件落盘，让 `assets/` 目录真正可用。
2. 给右侧「设计文件」增加文件详情预览，方便直接查看 runner 写出的 `brief.md`、`slice-plan.json`、`critique.json`。
3. 基于 Runner Adapter 写旧 Python runner 的 adapter skeleton，先定义输入输出，不直接跑真实生图。
4. 为 Codex CLI runner 设计事件归一化映射：raw CLI 文本只能进调试折叠区，前端继续消费 `AgentEvent`。
5. 多项目目录持久化应在 runner 接真实执行前单独规划，不要混进当前单项目 mock 流程。

## 给下一个 Codex 窗口的提示

先读：

- `00_新工作台主入口.md`
- `AGENTS.md`
- `11_开发日志/00_当前开发状态.md`
- `10_产品代码包/apps/daemon/src/runners/types.ts`
- `10_产品代码包/apps/daemon/src/runners/mockLongImageRunner.ts`

然后继续：

- 不要推翻现有 Home + Project Workspace。
- 不要把前端改成直接跑 CLI。
- 如果接 Codex CLI 或旧 runner，只能从 daemon runner 层接入，并继续产出结构化 `AgentEvent`、`RunState`、`FileManifest`。
- 不要复活旧 Dify / ComfyUI 作为当前产品入口。

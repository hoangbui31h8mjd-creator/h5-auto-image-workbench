# 2026-07-10 项目文件落盘与 run history 持久化

## 本轮目标

第五步目标是把 daemon 的当前 mock 项目状态从纯内存推进到真实本地项目目录，让后续 Codex CLI / Python runner 可以围绕同一套项目文件读写。

本轮不接真实 Codex CLI，不接真实生图服务，不重做 UI，不改变「一级 Home + 二级 Project Workspace」结构。

## 本次新增能力

- 新增 daemon 存储模块：`10_产品代码包/apps/daemon/src/projectStore.ts`。
- daemon 启动时会初始化或读取当前项目目录：`10_产品代码包/.h5-workbench/projects/current/`。
- `/api/projects/current` 现在返回从磁盘恢复后的当前项目状态，而不是只依赖进程内存初始值。
- `POST /api/projects/current` 会创建新的当前 mock 项目，并重写当前项目文件和初始事件。
- `POST /api/runs/current/start` 启动 mock 长图 run 后，会持续把事件、阶段状态和文件清单写入磁盘。
- 新增文件内容 API：`GET /api/projects/current/files/:fileId`。
- run history 使用 `events.jsonl` 保存，daemon 重启后可以恢复历史事件。
- 状态写入增加串行队列，避免高频 mock run 中事件和状态文件乱序写入。
- atomic JSON 写入的临时文件名改为 UUID，避免同一毫秒并发写入时临时文件名碰撞。

## 当前本地目录结构

当前运行数据目录：

```text
10_产品代码包/.h5-workbench/projects/current/
```

当前会写入：

```text
current/
  project.json
  brief.md
  h5-artifact.json
  prototype.html
  critique.json
  slice-plan.json
  files.manifest.json
  run-state.json
  events.jsonl
  assets/
  output/
    long-image.png
    export.zip
```

说明：`output/long-image.png` 和 `output/export.zip` 当前不是实际图片和 ZIP，而是 JSON placeholder 元数据。这样做是为了先固定文件关系和状态流，后续接入真实 runner 后再替换成真实产物。

## 新增和调整的 API

- `GET /api/health`：新增返回 `currentProjectDir`，方便确认 daemon 当前读写目录。
- `GET /api/projects/current`：返回从本地项目目录恢复的 `mockProjectState`。
- `POST /api/projects/current`：创建并落盘当前 mock 项目。
- `GET /api/projects/current/files/:fileId`：读取某个项目文件内容，返回 `file`、`content`、`contentType`。
- `POST /api/runs/current/start`：继续启动 `long_image_generation` mock run，同时写入 `events.jsonl`、`run-state.json`、`files.manifest.json` 和 output placeholder。

## 本轮验证结果

- `pnpm typecheck` 通过。
- `pnpm --filter @h5-workbench/web build` 通过。
- daemon `/api/health` 正常返回，并显示当前项目目录。
- daemon 第一次启动后已创建 `10_产品代码包/.h5-workbench/projects/current/`。
- `/api/projects/current` 能返回当前项目状态。
- `GET /api/projects/current/files/file_brief` 能返回 `brief.md` 内容。
- `GET /api/projects/current/files/file_slice_plan` 能返回包含 `slices` 的切片计划内容。
- `POST /api/runs/current/start` 能启动 mock 长图 run。
- SSE 能推送本轮 run 的 `agent_event`、`run_state`、`file_manifest`、`project_state` 和 `done`。
- mock run 结束后，`slice-plan.json`、`long-image.png`、`export.zip` 的文件状态为 `done`。
- `events.jsonl` 已追加事件；本轮验证后为 61 行。
- `run-state.json` 已更新为 `done`。
- `files.manifest.json` 已更新核心产物状态。
- `output/long-image.png` 和 `output/export.zip` 已写入 placeholder。
- 重启 daemon 后，`/api/projects/current` 能恢复上一轮 `generated` / `done` 状态和历史事件。
- 前端 `http://127.0.0.1:5178/` 仍能正常访问。

## 还没完成

- 未接真实 Codex CLI。
- 未接真实 Claude CLI。
- 未接旧 Python runner。
- 未生成真实 PNG / WebP / ZIP。
- 未做真实素材上传落盘；`assets/` 目录目前只是预留。
- 前端右侧「设计文件」还没有复杂的文件内容预览 UI，本轮只完成 daemon 文件内容 API。
- 当前持久化仍是单个 `projects/current/`，还不是多项目目录索引。

## 下一步建议

1. 先做素材/KV 的 daemon 上传接口，把 `ImageSlot.assetPath` 从 mock 字段变成真实 `assets/` 文件路径。
2. 给右侧「设计文件」增加轻量文件详情预览，优先支持 `brief.md`、`h5-artifact.json`、`prototype.html`、`critique.json`、`slice-plan.json`。
3. 梳理旧 Python runner 的最小适配点：读取 `h5-artifact.json` 和 `slice-plan.json`，输出真实长图和导出包。
4. 接真实 CLI 时保持当前原则：前端只消费结构化 `AgentEvent`、`RunState`、`FileManifest`，raw CLI 只进入调试折叠区。
5. 多项目持久化后再从 `projects/current/` 扩展为 `projects/<projectId>/`，不要提前做账号或团队协作。

## Guardian 结论

本轮没有跑偏。产品仍保持 Open Design 式两级工作台结构：Home 负责启动任务，Project Workspace 负责 Agent 执行、文件管理、原型预览、长图生成和素材管理。第五步只补齐 daemon 与本地项目目录之间的通信和持久化基础，没有把旧 Dify / ComfyUI 链路复活为当前主入口，也没有把前端改成普通后台或完整自由画布。

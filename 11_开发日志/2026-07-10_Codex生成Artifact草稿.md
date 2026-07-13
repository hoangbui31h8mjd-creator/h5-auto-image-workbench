# 2026-07-10 Codex 生成 Artifact 草稿

## 本次修改背景

第七步已经验证 daemon 可以通过 Runner Adapter 调用本机 Codex CLI，并生成一个很小的 `codex-probe-result.md`。第八步继续小步推进：让 Codex CLI 根据 `brief.md` 和 H5Artifact 类型说明生成一个结构化草稿，但不能覆盖正式 `h5-artifact.json`。

本轮不接真实生图，不生成真实长图，不重做 UI，不让 Codex 直接改正式协议文件。

## 本次目标

- 新增 `codex_artifact_draft` runType。
- daemon 通过 runner 调用 Codex CLI。
- Codex 输入为：
  - `current/brief.md`
  - `packages/h5-artifact/src/index.ts` 中的 H5Artifact 类型说明节选
- Codex 输出草稿：
  - `current/h5-artifact.draft.json`
- 如果 JSON 不合法：
  - 保留 `current/codex-artifact-draft.raw.md`
  - 写入 `error` event
- 不覆盖：
  - `current/h5-artifact.json`
  - `current/prototype.html`
- 保持 `codex_cli_probe` 和 `long_image_generation` 正常。

## 修改文件

- `10_产品代码包/packages/agent-contract/src/index.ts`：新增 `RunType = "codex_artifact_draft"`，新增 `FileKind = "artifact_draft"`。
- `10_产品代码包/apps/daemon/src/runners/codexCliRunner.ts`：扩展为双任务 runner，支持 `codex_cli_probe` 和 `codex_artifact_draft`。
- `10_产品代码包/apps/daemon/src/runners/index.ts`：注册 `codexArtifactDraftRunner`。
- `10_产品代码包/apps/daemon/src/server.ts`：允许 `POST /api/runs/current/start` 接收 `codex_artifact_draft`。
- `10_产品代码包/apps/daemon/src/projectStore.ts`：支持读取 `file_artifact_draft` 和失败 raw 文件 `file_codex_artifact_raw`。
- `10_产品代码包/apps/web/src/components/DesignFilesView.tsx`：补充 `artifact_draft` 文件类型显示文案。
- `11_开发日志/00_当前开发状态.md`：更新当前阶段状态。
- `11_开发日志/2026-07-10_Codex生成Artifact草稿.md`：新增本日志。

## 执行方式

```http
POST /api/runs/current/start
Content-Type: application/json

{
  "runType": "codex_artifact_draft"
}
```

runner 内部调用：

```text
codex exec
  --cd <current project dir>
  --sandbox read-only
  --skip-git-repo-check
  --ephemeral
  --color never
  --output-last-message <current project dir>/codex-artifact-draft.raw.md
  -
```

说明：

- Codex 的执行目录限制在 `10_产品代码包/.h5-workbench/projects/current/`。
- daemon 读取 H5Artifact 类型说明后放入 prompt，Codex 不直接访问源码目录。
- Codex raw 输出先写入 `codex-artifact-draft.raw.md`。
- daemon parse JSON 成功后，再写入格式化后的 `h5-artifact.draft.json`。
- daemon 不会覆盖正式 `h5-artifact.json`。

## JSON 校验

daemon 会检查：

- 顶层必须是 object。
- 必须包含 `id`、`artifactId`、`title`、`artifactType`、`status`、`language`、`h5Type`、`designSystem`、`pageSize`、`theme`、`sourceBrief`、`sections`、`imageSlots`、`assets`、`versions`、`createdAt`、`updatedAt`。
- `artifactType` 必须是 `h5_campaign`。
- `status` 必须是 `draft`。
- `sections` 数量必须在 4-6 个。
- `imageSlots` 数量必须在 3-6 个。
- 每个 section 必须包含 `id`、`sectionId`、`type`、`name`、`order`、`height`、`component`、`props`、`textFields`、`imageSlotIds`、`imageSlots`、`promptHint`。
- 每个 imageSlot 必须包含 `slotId`、`label`、`purpose`、`required`、`recommendedSize`、`acceptedTypes`、`status`、`usedBySections`、`promptHint`。

## 本轮验证结果

- `pnpm typecheck` 通过。
- `pnpm --filter @h5-workbench/web build` 通过。
- `GET /api/health` 正常。
- `POST /api/runs/current/start` with `codex_artifact_draft` 返回 `202`。
- SSE 能看到：
  - `run_started`
  - `assistant_message`
  - `file_written: h5-artifact.draft.json`
  - `artifact_updated`
  - `run_finished`
- 已生成：`10_产品代码包/.h5-workbench/projects/current/h5-artifact.draft.json`。
- `h5-artifact.draft.json` 是合法 JSON。
- 本轮实际结果：`title = Game Fiesta Premiada`，`sections = 5`，`imageSlots = 5`。
- `GET /api/projects/current/files/file_artifact_draft` 能读取草稿内容。
- FileManifest 已新增 `h5-artifact.draft.json`，kind 为 `artifact_draft`，status 为 `draft`。
- 正式 `h5-artifact.json` 未被覆盖。
- `codex_cli_probe` 仍可用。
- `long_image_generation` 仍可用。
- 前端 `http://127.0.0.1:5178/` 仍可访问。

## 尚未完成

- 还没有 Artifact draft 的前端详情预览。
- 还没有 draft 与正式 `h5-artifact.json` 的 diff。
- 还没有「采用草稿」按钮或确认流程。
- 还没有让 Codex 生成 prototype.html。
- 还没有接真实生图或旧 Python runner。
- Codex stderr 中仍会出现一些 plugin/config warning，目前作为 raw_log 保留。

## 下一步建议

1. 做「设计文件」文件详情预览，让用户能查看 `h5-artifact.draft.json`。
2. 做 Artifact draft diff / adopt 流程：人工确认后再覆盖正式 `h5-artifact.json`。
3. 增加 draft 校验错误的前端展示，避免只在 raw log 中查看失败原因。
4. 继续保持 Codex runner 小步推进，不要直接接完整自动生成和真实生图。
5. 旧 Python runner 接入应基于已确认的正式 `h5-artifact.json`，不要直接消费未确认 draft。

## Guardian 结论

本轮没有跑偏。Codex CLI 仍然只作为 daemon runner 后面的执行引擎；前端没有直接解析 CLI 文本，也没有变成终端 UI。产物是独立的 `h5-artifact.draft.json`，正式 `h5-artifact.json` 没有被覆盖，长图生成和旧 runner 入口没有被提前接入。

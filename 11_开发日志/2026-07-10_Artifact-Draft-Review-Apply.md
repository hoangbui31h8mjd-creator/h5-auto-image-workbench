# 2026-07-10 Artifact Draft Review & Apply

## 本次修改背景

第八步已经让 Codex CLI 能生成 `h5-artifact.draft.json`，第九步也让左侧 Agent 输入可以驱动 daemon / Codex runner。但当时草稿仍然只是一个文件产物：用户还不能在工作台里查看、确认、应用或放弃草稿。

本轮目标是补齐「草稿审核 -> 人工应用 -> 正式 Artifact 更新」的安全闭环。

本轮不让 Codex 直接覆盖正式 `h5-artifact.json`，不自动生成长图，不接真实生图服务，也不推翻现有 Home + Project Workspace 结构。

## 本次目标

- 新增 `POST /api/artifacts/draft/apply`。
- 新增 `POST /api/artifacts/draft/discard`。
- 前端在「设计文件」tab 中提供：
  - 查看草稿
  - 应用草稿
  - 放弃草稿
- 应用草稿前必须 parse JSON 并校验基础字段。
- 应用成功后：
  - 更新正式 `h5-artifact.json`
  - 备份上一版为 `h5-artifact.previous.json`
  - 更新 `prototype.html`
  - 更新 FileManifest
  - 写入 AgentEvent
  - 前端原型预览使用新 artifact
- 不自动启动长图生成。

## 修改文件

- `10_产品代码包/packages/agent-contract/src/index.ts`
  - `FileKind` 新增 `artifact_backup`。
  - `FileStatus` 新增 `discarded`。
- `10_产品代码包/apps/daemon/src/projectStore.ts`
  - 新增 `applyArtifactDraft()`。
  - 新增 `discardArtifactDraft()`。
  - 新增 `h5-artifact.previous.json` 文件映射。
  - 新增 H5Artifact draft 基础校验。
  - apply 时写入正式 artifact、previous 备份、prototype、manifest。
  - apply / discard 要求当前 FileManifest 中存在 active `file_artifact_draft`，避免误应用历史残留草稿。
- `10_产品代码包/apps/daemon/src/server.ts`
  - 新增 `POST /api/artifacts/draft/apply`。
  - 新增 `POST /api/artifacts/draft/discard`。
  - 成功时写入 `assistant_message`、`file_written`、`artifact_updated`。
  - 失败时写入 `error` event。
- `10_产品代码包/apps/web/src/api/daemonClient.ts`
  - 新增 `getProjectFile()`。
  - 新增 `applyArtifactDraft()`。
  - 新增 `discardArtifactDraft()`。
- `10_产品代码包/apps/web/src/App.tsx`
  - 新增 draft review 状态。
  - 新增查看、应用、放弃草稿处理逻辑。
  - apply 成功后用 daemon 返回的完整 state 刷新项目、文件、素材槽位、Agent 事件和原型预览。
- `10_产品代码包/apps/web/src/components/ProjectShell.tsx`
  - 透传 draft review 相关 props。
- `10_产品代码包/apps/web/src/components/ProjectWorkspace.tsx`
  - 透传 draft review 相关 props 到设计文件视图。
- `10_产品代码包/apps/web/src/components/DesignFilesView.tsx`
  - 新增 Artifact Draft Review 区。
  - 支持查看草稿 JSON 摘要和预览。
  - 支持应用 / 放弃草稿按钮。
  - 支持 `artifact_backup` 文件类型文案。
- `10_产品代码包/apps/web/src/styles.css`
  - 新增 draft review、draft preview、discarded 状态样式。
- `11_开发日志/00_当前开发状态.md`
  - 更新当前阶段状态。
- `11_开发日志/2026-07-10_Artifact-Draft-Review-Apply.md`
  - 新增本日志。

## 新增 API

### 应用草稿

```http
POST /api/artifacts/draft/apply
Content-Type: application/json

{}
```

成功返回：

```json
{
  "state": "...当前 mockProjectState...",
  "artifact": "...应用后的正式 H5Artifact...",
  "previousArtifactFile": "/api/projects/current/files/file_artifact_previous"
}
```

### 放弃草稿

```http
POST /api/artifacts/draft/discard
Content-Type: application/json

{}
```

成功返回：

```json
{
  "state": "...当前 mockProjectState..."
}
```

## 安全策略

- Codex 生成 `h5-artifact.draft.json` 后，不会自动覆盖正式 `h5-artifact.json`。
- apply / discard 必须由前端用户动作触发。
- apply 前 daemon 会校验：
  - 顶层 H5Artifact 必需字段。
  - `artifactType = "h5_campaign"`。
  - `sections` 非空。
  - 每个 section 必须有 `id`、`sectionId`、`type`、`name`、`order`、`component`、`props`、`textFields`、`imageSlotIds`、`imageSlots`、`promptHint`、`height`。
  - `imageSlots` 非空。
  - 每个 imageSlot 必须有 `slotId`、`label`、`purpose`、`required`、`recommendedSize`、`acceptedTypes`、`status`、`usedBySections`、`promptHint`。
- apply / discard 要求当前 FileManifest 中存在 active `file_artifact_draft`。
- 如果只有磁盘残留的旧 `h5-artifact.draft.json`，但当前项目 manifest 没有该文件项，则拒绝 apply，避免误应用历史草稿。
- apply 时会把草稿状态规范为 `edited`。
- apply 时会把上一版正式 artifact 写入 `h5-artifact.previous.json`。
- apply 不会自动启动 `long_image_generation`。

## 前端行为

当右侧「设计文件」tab 发现 `h5-artifact.draft.json`：

- 显示 Artifact Draft Review 区。
- 点击「查看草稿」：
  - 调用 `GET /api/projects/current/files/file_artifact_draft`。
  - 展示 title、sections 数、imageSlots 数和 JSON 预览。
- 点击「应用草稿」：
  - 调用 `POST /api/artifacts/draft/apply`。
  - 使用 daemon 返回的 state 刷新前端项目。
  - 切换到「原型预览」tab，显示新 artifact。
- 点击「放弃草稿」：
  - 调用 `POST /api/artifacts/draft/discard`。
  - 将草稿文件状态标记为 `discarded`。

## 本轮验证结果

- `pnpm typecheck` 通过。
- `pnpm --filter @h5-workbench/web build` 通过。
- `GET /api/health` 正常。
- `POST /api/runs/current/start` with `codex_artifact_draft` 可生成当前项目 draft。
- `FileManifest` 中出现 `file_artifact_draft: draft`。
- `POST /api/artifacts/draft/apply` 成功。
- apply 后：
  - `h5-artifact.json` title 为「游戏活动 H5」。
  - `h5-artifact.json` sections 为 5。
  - `h5-artifact.json` status 为 `edited`。
  - `h5-artifact.previous.json` 存在。
  - `h5-artifact.draft.json` 仍保留，manifest 状态为 `done`。
  - `prototype.html` 已按新 artifact 重写。
  - `GET /api/projects/current/artifact` 返回新 artifact。
  - AgentEvent 中包含 `artifact_updated`。
- 前端 `http://127.0.0.1:5178/` 仍可访问。

## 尚未完成

- 目前还不是完整 diff，只是草稿 JSON 摘要和预览。
- 设计文件列表还没有通用文件详情阅读器。
- 还没有「对比正式版 / 草稿版」的模块级差异视图。
- 还没有把左侧 Agent 建议直接转成 artifact patch。
- discard API 已实现，但本轮主要验收 apply；后续可在 UI 验收时再覆盖 discard 交互。
- 仍未接真实生图、旧 Python runner 或真实导出。

## 下一步建议

1. 做通用文件详情 Viewer，支持 markdown、json、html、placeholder output。
2. 做 Artifact draft diff：模块顺序、section 数、图片槽位、主题色、关键文案变化。
3. 做 `artifact_patch` runType，让 Codex 输出局部 patch，而不是每次完整 draft。
4. 给左侧 Agent 输入增加快捷 intent，例如「根据这条建议生成草稿」。
5. 等 artifact review/apply 稳定后，再接真实长图 runner。

## Guardian 结论

本轮没有跑偏。Codex 仍然只负责生成草稿，正式 `h5-artifact.json` 必须经过用户在工作台里的手动确认才会更新。更新后也只刷新原型预览和项目文件状态，不会自动进入生图阶段。

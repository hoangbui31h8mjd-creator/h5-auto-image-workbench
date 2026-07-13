# 2026-07-10 旧 Runner 原型 Review & Apply

## 本次修改背景

第十五步已经证明 daemon 可以真实调起旧项目 `run_local_prototype.py`，并在受控 dry-run 目录里产出 `prototype.dry-run.html`、`02_prototype.html`、`03_prototype_full.png`、`01_blueprint.json` 和 `07_quality_report.md`。

但旧 runner 产物不能自动覆盖当前工作台正式原型。本轮目标是把 `prototype.dry-run.html` 变成可人工审稿的候选原型：先查看，再由用户手动应用；不满意时可以放弃候选。

## 本次目标

- 给 daemon 增加旧 runner 候选原型 apply / discard API。
- apply 时备份当前正式 `prototype.html`，再应用 `prototype.dry-run.html`。
- discard 时只标记候选状态，不影响正式原型。
- 前端在「设计文件」tab 增加轻量 Review & Apply 区。
- 应用后正式原型预览能显示旧 runner HTML。
- 全程不修改正式 `h5-artifact.json`，不接真实生图，不自动启动长图生成。

## 修改文件

- `10_产品代码包/packages/agent-contract/src/index.ts`：补充 `prototype_previous` 文件类型，以及 `applied`、`legacy_applied` 文件状态。
- `10_产品代码包/apps/daemon/src/projectStore.ts`：新增 `applyPrototypeDryRun()`、`discardPrototypeDryRun()`，支持 `prototype.previous.html` 备份、候选应用、候选放弃、manifest 更新和文件内容读取。
- `10_产品代码包/apps/daemon/src/server.ts`：新增 `POST /api/prototypes/dry-run/apply` 和 `POST /api/prototypes/dry-run/discard`，并写入 AgentEvent。
- `10_产品代码包/apps/web/src/api/daemonClient.ts`：新增 `applyPrototypeDryRun()`、`discardPrototypeDryRun()`。
- `10_产品代码包/apps/web/src/App.tsx`：接入候选原型查看、应用、放弃逻辑；应用后重新拉取项目 state 和正式 `prototype.html`。
- `10_产品代码包/apps/web/src/components/ProjectShell.tsx`：透传候选原型 Review & Apply 状态与操作。
- `10_产品代码包/apps/web/src/components/ProjectWorkspace.tsx`：把候选原型操作传给设计文件区，并把正式 HTML 传给原型预览。
- `10_产品代码包/apps/web/src/components/DesignFilesView.tsx`：新增「旧 runner 候选原型」区域，支持查看候选原型、应用为正式原型、放弃候选原型。
- `10_产品代码包/apps/web/src/components/ArtifactViewer.tsx`：当正式 `prototype.html` 来自旧 runner 应用时，用 iframe 预览正式 HTML，并显示来源「旧 runner 已应用」。
- `10_产品代码包/apps/web/src/styles.css`：补充候选原型预览、正式 HTML iframe 预览、`applied` / `legacy_applied` 状态样式。
- `11_开发日志/00_当前开发状态.md`：更新当前阶段状态。
- `11_开发日志/2026-07-10_旧Runner原型Review-Apply.md`：新增本日志。

## 新增 API

### `POST /api/prototypes/dry-run/apply`

作用：

- 检查 `legacy-task/dry-run/prototype.dry-run.html` 是否存在。
- 备份当前正式 `prototype.html` 为 `prototype.previous.html`。
- 将 `prototype.dry-run.html` 应用为正式 `prototype.html`。
- 更新 FileManifest：
  - `file_prototype` -> status `legacy_applied`
  - `file_prototype_previous` -> status `ready`
  - `file_prototype_dry_run` -> apply 成功时 status `applied`
- 写入 AgentEvent：
  - `assistant_message`：说明已应用旧 runner 候选原型。
  - `file_written: prototype.previous.html`
  - `file_written: prototype.html`

安全边界：

- 不修改正式 `h5-artifact.json`。
- 不自动生成长图。
- 不运行 `run_local_h5.py`。

### `POST /api/prototypes/dry-run/discard`

作用：

- 将 FileManifest 中 `file_prototype_dry_run` 标记为 `discarded`。
- 写入 AgentEvent，说明候选原型已放弃。
- 不删除 dry-run 文件。
- 不影响正式 `prototype.html`。
- 不影响正式 `h5-artifact.json`。

## 当前文件状态

候选原型：

```text
10_产品代码包/.h5-workbench/projects/current/legacy-task/dry-run/prototype.dry-run.html
```

正式原型：

```text
10_产品代码包/.h5-workbench/projects/current/prototype.html
```

应用前正式原型备份：

```text
10_产品代码包/.h5-workbench/projects/current/prototype.previous.html
```

正式 Artifact：

```text
10_产品代码包/.h5-workbench/projects/current/h5-artifact.json
```

## 验收结果

- `pnpm typecheck`：通过。
- `pnpm --filter @h5-workbench/web build`：通过。
- `GET /api/health`：正常返回 daemon 状态。
- `GET /api/projects/current/files/file_prototype_dry_run`：可读取 `prototype.dry-run.html`。
- `POST /api/prototypes/dry-run/apply`：已成功执行。
- apply 后 `prototype.previous.html` 存在。
- apply 后正式 `prototype.html` 内容已变成旧 runner 候选原型。
- apply 后正式 `h5-artifact.json` hash 保持不变。
- apply 后 AgentEvent 写入 `assistant_message`、`file_written: prototype.previous.html`、`file_written: prototype.html`。
- `GET /api/projects/current/files/file_prototype`：可读取当前正式原型，status 为 `legacy_applied`。
- `GET /api/projects/current/files/file_prototype_previous`：可读取上一版正式原型备份。
- `POST /api/prototypes/dry-run/discard`：已验证可把候选原型标记为 `discarded`。
- discard 后正式 `prototype.html` 未回退、未修改，仍保持旧 runner 应用版本。
- 前端 `http://127.0.0.1:5178/` 可访问，HTTP `200 OK`。

## Hash 验证

本轮 apply / discard 后的关键 hash：

```text
prototype.html              02fd14a8855d64680f9857d6a548a01ff50aa6f518b4a62edb0799a5c41a7da7
prototype.dry-run.html      02fd14a8855d64680f9857d6a548a01ff50aa6f518b4a62edb0799a5c41a7da7
prototype.previous.html     855a1aecac54c45a4c915c3a3ba610853ca74aef11f5337aaa66b124b73670fd
h5-artifact.json            00af5829ea39929065b3f0e8787c358d49c9c9b0500c64c78537943d08a8c2b5
```

结论：

- 当前正式 `prototype.html` 已经来自旧 runner 候选原型。
- `prototype.previous.html` 保留了应用前的旧正式原型。
- 正式 `h5-artifact.json` 没有被修改。

## 当前 FileManifest 状态

- `file_prototype`：`prototype.html`，status `legacy_applied`。
- `file_prototype_previous`：`prototype.previous.html`，status `ready`。
- `file_prototype_dry_run`：本轮先验证 apply 时 status 为 `applied`，随后为了验证 discard API，最终标记为 `discarded`。

说明：discard 只表示当前候选项被放弃，不会回退已经应用的正式 `prototype.html`。当前正式原型仍是旧 runner 应用版本。

## 尚未解决

- 当前旧 runner 输出仍来自 dry-run 路径，不是稳定的正式 prototype generation adapter。
- 当前候选原型 Review 还没有复杂 diff，只能查看候选、应用、放弃。
- 当前 `h5-artifact.json` 不会从旧 runner HTML 反推更新，正式结构化协议仍保持原样。
- 当前 dry-run 使用占位 KV，不代表真实用户 KV 素材已接入。
- 仍未运行 `run_local_h5.py`，未接真实生图服务，未生成真实长图或真实 ZIP。

## 下一步建议

1. 把旧 runner dry-run 提升为更正式的 `legacy_prototype_generation` adapter，产出 `prototype.generated.html`，但继续走人工 Review & Apply。
2. 为候选原型增加审稿信息和 diff：对比 `prototype.previous.html`、当前 `prototype.html`、候选来源、生成时间、质量报告。
3. 把素材/KV 槽位真正落盘到 `assets/`，让旧 runner 使用真实 KV，而不是 dry-run 占位图。
4. 在原型生成 adapter 稳定后，再进入 `run_local_h5.py` / 长图真实链路接入。

## 给下一个 Codex 窗口的提示

先读：

- `00_新工作台主入口.md`
- `AGENTS.md`
- `11_开发日志/00_当前开发状态.md`
- `11_开发日志/2026-07-10_旧Runner-DryRun可行性验证.md`
- `11_开发日志/2026-07-10_旧Runner原型Review-Apply.md`

然后继续：

- 不要推翻「一级 Home + 二级 Project Workspace」结构。
- 不要把旧 runner 产物自动覆盖正式文件。
- 不要修改正式 `h5-artifact.json`，除非用户明确进入 Artifact apply / patch 流程。
- 下一步优先做正式 prototype generation adapter 和候选原型 diff / 审稿体验，再考虑真实长图生成。

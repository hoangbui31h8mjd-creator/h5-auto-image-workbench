# 2026-07-10 Home 接 Legacy 原型候选生成

## 本次修改背景

第十六步已经完成旧 runner 原型输出 Review & Apply：`prototype.dry-run.html` 可以人工查看、应用或放弃，应用前会备份 `prototype.previous.html`，且不会修改正式 `h5-artifact.json`。

但此前旧 runner 仍主要通过 dry-run / 手动 API 触发，不是从一级 Home 的正常产品流程触发。本轮目标是让 Home 的「生成 H5 原型」按钮真正启动旧 runner 原型候选生成。

## 本次目标

- 新增产品流程用 runType：`legacy_prototype_generation`。
- 新增 Home 专用 daemon API：`POST /api/projects/create-from-brief`。
- Home 输入需求后，daemon 创建/更新 current 项目，并启动旧 `run_local_prototype.py`。
- 旧 runner 输出写入 `legacy-task/candidate/`，生成 `prototype.candidate.html` 等候选文件。
- 前端进入二级 Project Workspace，左侧 Agent 面板展示生成过程。
- 右侧「设计文件」展示候选原型，并复用 Review & Apply。
- 不自动覆盖正式 `prototype.html`。
- 不修改正式 `h5-artifact.json`。
- 不接真实生图，不自动生成长图。

## 修改文件

- `10_产品代码包/packages/agent-contract/src/index.ts`：新增 `legacy_prototype_generation` runType；新增 `prototype_candidate`、`prototype_candidate_image`、`legacy_candidate_report`、`legacy_blueprint`、`legacy_quality_report` 文件类型；新增 `legacy_candidate`、`ready_for_review` 文件状态。
- `10_产品代码包/apps/daemon/src/runners/legacyPrototypeGenerationRunner.ts`：新增产品流程用旧 runner 候选原型生成 runner。
- `10_产品代码包/apps/daemon/src/runners/index.ts`：注册 `legacyPrototypeGenerationRunner`。
- `10_产品代码包/apps/daemon/src/projectStore.ts`：新增 candidate 文件读取、`applyPrototypeCandidate()`、`discardPrototypeCandidate()`，支持 `prototype.candidate.html` 应用为正式 `prototype.html`。
- `10_产品代码包/apps/daemon/src/server.ts`：新增 `POST /api/projects/create-from-brief`、`POST /api/prototypes/candidate/apply`、`POST /api/prototypes/candidate/discard`，并允许 `POST /api/runs/current/start` 启动 `legacy_prototype_generation`。
- `10_产品代码包/apps/web/src/api/daemonClient.ts`：新增 `createProjectFromBrief()`、`applyPrototypeCandidate()`、`discardPrototypeCandidate()`。
- `10_产品代码包/apps/web/src/App.tsx`：Home「生成 H5 原型」改为优先调用 `createProjectFromBrief()`；候选 Review 操作优先使用 `prototype.candidate.html`，兼容旧 `prototype.dry-run.html`。
- `10_产品代码包/apps/web/src/components/DesignFilesView.tsx`：候选原型 Review 区支持 `prototype.candidate.html`，并补充候选文件类型和状态展示。
- `10_产品代码包/apps/web/src/styles.css`：补充 `ready_for_review`、`legacy_candidate` 状态样式。
- `11_开发日志/00_当前开发状态.md`：更新当前阶段状态。
- `11_开发日志/2026-07-10_Home接Legacy原型候选生成.md`：新增本日志。

## 新增 daemon API

### `POST /api/projects/create-from-brief`

请求体示例：

```json
{
  "brief": "做一个游戏活动 H5，突出奖励、参与方式和底部 CTA，整体要有活动氛围。",
  "templateId": "game_campaign",
  "designSystem": "internal_default",
  "language": "pt-BR",
  "h5Type": "game_campaign"
}
```

行为：

- 创建/更新 current 项目。
- 写入 `project.json`、`brief.md`、初始化 `h5-artifact.json`。
- 启动 `legacy_prototype_generation`。
- 返回 `state`、`runId`、`runType`、`status`。
- 不直接覆盖正式 `prototype.html`。

### `POST /api/prototypes/candidate/apply`

行为：

- 检查 `legacy-task/candidate/prototype.candidate.html` 是否存在。
- 备份当前正式 `prototype.html` 为 `prototype.previous.html`。
- 将 `prototype.candidate.html` 应用为正式 `prototype.html`。
- 更新 FileManifest：
  - `file_prototype` -> `legacy_applied`
  - `file_prototype_candidate` -> `applied`
  - `file_prototype_previous` -> `ready`
- 不修改正式 `h5-artifact.json`。
- 不自动生成长图。

### `POST /api/prototypes/candidate/discard`

行为：

- 将 `prototype.candidate.html` 标记为 `discarded`。
- 不删除候选文件。
- 不影响正式 `prototype.html`。
- 不影响正式 `h5-artifact.json`。

## legacy_prototype_generation runner

实际旧 runner 入口：

```text
/Users/yiming/Documents/Codex/2026-06-17/h5-kv-h5-demo-dify-llm/h5_agent/runner/run_local_prototype.py
```

实际执行命令：

```bash
/Users/yiming/Documents/Codex/2026-06-17/h5-kv-h5-demo-dify-llm/.venv/bin/python \
  -m h5_agent.runner.run_local_prototype \
  --task-dir /Users/yiming/Desktop/H5自动生图工作台/10_产品代码包/.h5-workbench/projects/current/legacy-task/candidate/input \
  --output-root /Users/yiming/Desktop/H5自动生图工作台/10_产品代码包/.h5-workbench/projects/current/legacy-task/candidate/output \
  --job-id legacy-candidate-mock-run-1783648584577
```

cwd：

```text
/Users/yiming/Documents/Codex/2026-06-17/h5-kv-h5-demo-dify-llm
```

本轮仍只执行：

```text
run_local_prototype.py
```

本轮没有执行：

```text
run_local_h5.py
```

## 候选输出目录

所有候选输出都写在：

```text
10_产品代码包/.h5-workbench/projects/current/legacy-task/candidate/
```

本轮实际产物：

```text
command.json
stdout.log
stderr.log
exit-code.txt
legacy-candidate-report.md
prototype.candidate.html
prototype.candidate.png
legacy-blueprint.candidate.json
legacy-quality-report.md
input/content.md
input/special_requirements.md
input/run_config.json
input/kv.png
output/legacy-candidate-mock-run-1783648584577/01_blueprint.json
output/legacy-candidate-mock-run-1783648584577/02_prototype.html
output/legacy-candidate-mock-run-1783648584577/03_prototype_full.png
output/legacy-candidate-mock-run-1783648584577/07_quality_report.md
output/legacy-candidate-mock-run-1783648584577/run_summary.json
```

说明：`input/kv.png` 仍是候选生成占位素材，不是用户真实 KV。

## FileManifest 映射

本轮新增或更新：

```text
file_prototype_candidate
filename: prototype.candidate.html
kind: prototype_candidate
status: applied
```

```text
file_prototype_candidate_png
filename: prototype.candidate.png
kind: prototype_candidate_image
status: legacy_candidate
```

```text
file_legacy_blueprint_candidate
filename: legacy-blueprint.candidate.json
kind: legacy_blueprint
status: legacy_candidate
```

```text
file_legacy_quality_report
filename: legacy-quality-report.md
kind: legacy_quality_report
status: legacy_candidate
```

```text
file_legacy_candidate_report
filename: legacy-candidate-report.md
kind: legacy_candidate_report
status: ready_for_review
```

应用候选后：

```text
file_prototype
filename: prototype.html
status: legacy_applied
```

```text
file_prototype_previous
filename: prototype.previous.html
status: ready
```

## 前端交互变化

- Home「生成 H5 原型」现在优先调用 `createProjectFromBrief()`。
- 成功后直接进入 Project Workspace。
- 左侧 Agent 面板通过 SSE 展示：
  - 已接收 Home 需求
  - 准备调用旧 runner
  - raw_log 命令摘要
  - 写入候选文件
  - 候选原型生成完成，等待 Review & Apply
- 右侧「设计文件」tab 展示旧 runner 候选原型 Review 区。
- Review 区优先查看 `prototype.candidate.html`；如果没有产品候选，则兼容旧 `prototype.dry-run.html`。
- 应用候选后，原型预览 tab 显示正式 `prototype.html`，来源为「旧 runner 已应用」。

## 验收结果

- `pnpm typecheck`：通过。
- `pnpm --filter @h5-workbench/web build`：通过。
- 前端 `http://127.0.0.1:5178/`：HTTP `200 OK`。
- daemon `GET /api/health`：正常。
- `POST /api/projects/create-from-brief`：可启动，返回 `runType = legacy_prototype_generation`，`status = running`。
- 旧 runner 确实执行了 `run_local_prototype.py`。
- `legacy-task/candidate/` 下有输出。
- `prototype.candidate.html` 存在。
- FileManifest 出现候选原型和候选辅助产物。
- AgentEvent 记录了 `run_started`、`assistant_message`、`raw_log`、多个 `file_written` 和 `run_finished`。
- `GET /api/projects/current/files/file_prototype_candidate` 可读取候选 HTML。
- `POST /api/prototypes/candidate/apply` 可成功。
- apply 后 `prototype.previous.html` 存在。
- apply 后正式 `prototype.html` 更新为旧 runner 候选内容。
- apply 后正式 `h5-artifact.json` hash 保持不变。
- 未自动生成长图。

## Hash 验证

apply 前：

```text
prototype.html              855a1aecac54c45a4c915c3a3ba610853ca74aef11f5337aaa66b124b73670fd
prototype.candidate.html    02fd14a8855d64680f9857d6a548a01ff50aa6f518b4a62edb0799a5c41a7da7
h5-artifact.json            788cfabe890a0d05615cd3f251d75d212b7487601b04dc14023bd95f903c06d1
```

apply 后：

```text
prototype.html              02fd14a8855d64680f9857d6a548a01ff50aa6f518b4a62edb0799a5c41a7da7
prototype.previous.html     855a1aecac54c45a4c915c3a3ba610853ca74aef11f5337aaa66b124b73670fd
prototype.candidate.html    02fd14a8855d64680f9857d6a548a01ff50aa6f518b4a62edb0799a5c41a7da7
h5-artifact.json            788cfabe890a0d05615cd3f251d75d212b7487601b04dc14023bd95f903c06d1
```

结论：

- Home 已触发旧 runner 生成候选原型。
- 候选应用后正式 `prototype.html` 更新。
- `prototype.previous.html` 保留了应用前正式原型。
- 正式 `h5-artifact.json` 未被 candidate apply 修改。

## 尚未解决

- 当前 `legacy-blueprint.candidate.json` 还没有映射成 `h5-artifact.draft.json`。
- 当前候选 Review 还没有 diff，只能查看、应用、放弃。
- 当前候选生成仍使用占位 KV，没有接真实素材/KV 落盘。
- 当前没有运行 `run_local_h5.py`。
- 当前没有接真实生图服务。
- 当前没有生成真实 `long-image.png` 或真实 `export.zip`。

## 下一步建议

1. 做候选原型 diff / 审稿信息：对比 `prototype.previous.html`、当前 `prototype.html`、候选来源、生成时间、质量报告摘要。
2. 审计 `legacy-blueprint.candidate.json` 到 `h5-artifact.draft.json` 的可映射字段，仍走人工 Review & Apply。
3. 把素材/KV 槽位真正落盘到 `assets/`，让旧 runner 使用真实 KV。
4. 原型候选生成和素材落盘稳定后，再考虑 `run_local_h5.py` / 真实长图链路。

## 给下一个 Codex 窗口的提示

先读：

- `00_新工作台主入口.md`
- `AGENTS.md`
- `11_开发日志/00_当前开发状态.md`
- `11_开发日志/2026-07-10_旧Runner原型Review-Apply.md`
- `11_开发日志/2026-07-10_Home接Legacy原型候选生成.md`

然后继续：

- 不要推翻一级 Home + 二级 Project Workspace。
- 不要让旧 runner 自动覆盖正式文件。
- 不要把旧 runner blueprint 直接覆盖正式 `h5-artifact.json`。
- 下一步优先补候选 diff / blueprint 到 Artifact draft 的人工审核链路。

# 旧 Runner Adapter 设计与最小落地

日期：2026-07-10

## 本次修改背景

第十三步已经把当前工作台主链路稳定到：

```text
Home 输入需求
-> Project Workspace
-> 原型预览
-> Inspector 人工编辑
-> 确认原型
-> 长图 mock
-> 导出 mock
```

旧项目审计结论是：旧项目中有可复用的 Python 分层 pipeline、执行版 Skill 文档、组件 registry 和 H5 原型知识库，但没有可直接作为当前工作台入口运行的标准 Codex Skill。

因此本轮不接旧 pipeline，只先设计并落地当前工作台与旧 runner 之间的 Adapter 契约，避免未来直接把旧项目整体搬进来。

## 本次目标

- 定义当前工作台项目状态到旧 runner task_dir 的输入映射。
- 定义旧 runner 未来输出到当前 FileManifest / AgentEvent / RunState 的输出映射。
- 新增一个 `legacy_adapter_probe` runner，只验证 adapter 目录和 mock 输出。
- 不真实运行旧 Python pipeline。
- 不接真实生图服务。
- 不覆盖正式 `prototype.html` 和 `h5-artifact.json`。

## 修改文件

- `10_产品代码包/packages/agent-contract/src/index.ts`：新增 `legacy_adapter_probe` runType、`prototype_generated` 文件类型和 `adapter_mock` 文件状态。
- `10_产品代码包/apps/daemon/src/legacy-adapter/types.ts`：定义旧 runner adapter 的输入、配置、路径、输出清单和结果类型。
- `10_产品代码包/apps/daemon/src/legacy-adapter/legacyTaskMapper.ts`：实现 `prepareLegacyTask()`，把当前 projectState 写入 `legacy-task/` 输入目录。
- `10_产品代码包/apps/daemon/src/legacy-adapter/legacyOutputMapper.ts`：实现旧 runner 输出到当前 `FileManifest` 的 mock 映射。
- `10_产品代码包/apps/daemon/src/legacy-adapter/mockLegacyAdapter.ts`：实现 `runMockLegacyAdapter()`，写入 mock HTML 和输出清单。
- `10_产品代码包/apps/daemon/src/legacy-adapter/index.ts`：导出 adapter 模块。
- `10_产品代码包/apps/daemon/src/runners/legacyAdapterProbeRunner.ts`：新增 `legacy_adapter_probe` runner。
- `10_产品代码包/apps/daemon/src/runners/index.ts`：注册 `legacyAdapterProbeRunner`。
- `10_产品代码包/apps/daemon/src/runners/types.ts`：`file_manifest` RunnerEvent 增加 `writeProjectFiles?: boolean`，用于声明是否同步重写正式项目文件。
- `10_产品代码包/apps/daemon/src/server.ts`：允许启动 `legacy_adapter_probe`，并按 `writeProjectFiles` 控制 FileManifest 落盘行为。
- `10_产品代码包/apps/daemon/src/projectStore.ts`：支持读取 `file_prototype_generated`，映射到 `legacy-task/output/prototype.generated.html`。
- `10_产品代码包/apps/web/src/components/DesignFilesView.tsx`：设计文件区识别 `prototype_generated` 和 `adapter_mock`。
- `10_产品代码包/apps/web/src/styles.css`：补充 `adapter_mock` 状态样式。
- `11_开发日志/00_当前开发状态.md`：更新当前状态和下一步建议。

## legacy-task 目录结构

当前 adapter probe 会在当前项目目录下生成：

```text
10_产品代码包/.h5-workbench/projects/current/legacy-task/
  input/
    brief.md
    h5-artifact.json
    design-system.json
    image-slots.json
    assets-manifest.json
  config/
    task.json
  output/
    prototype.generated.html
    legacy-output-manifest.json
```

其中：

- `input/brief.md`：从当前工作台 `brief.md` 和 project metadata 派生。
- `input/h5-artifact.json`：当前正式 `h5-artifact.json` 的快照，用于未来旧 runner 读取。
- `input/design-system.json`：当前设计规范、语言、H5 类型和主题摘要。
- `input/image-slots.json`：当前 `artifact.imageSlots` 快照。
- `input/assets-manifest.json`：当前素材槽位到 assets 的占位映射。
- `config/task.json`：adapter 配置，当前 `mode = mock_prepare_only`，`willExecuteInThisRun = false`。
- `output/prototype.generated.html`：mock adapter 产物，不是旧 pipeline 真实输出。
- `output/legacy-output-manifest.json`：旧 runner 未来输出到当前工作台文件系统的映射清单。

## 当前工作台会传给旧 runner 的数据

第一版 adapter 输入覆盖：

- `brief.md`
- 正式 `h5-artifact.json`
- `language`
- `h5Type`
- `designSystem`
- `theme`
- `imageSlots`
- `assets` / 素材槽位清单
- `confirmedVersionId`
- 当前 projectId、project title、artifact status

这些数据足够未来旧 runner 判断：

- 当前 H5 要做什么。
- 当前有多少 section。
- 每个 section 的文本、props、图片槽位和高度。
- 哪些素材缺失或可用。
- 这次输入是否来自已确认版本。

## 旧 runner 未来输出如何映射回来

本轮只做 mock 映射，未来旧 runner 可能输出：

- `prototype.html`：映射为 `prototype.generated.html` 或在人工确认后覆盖正式 `prototype.html`。
- `h5-artifact.generated.json`：映射为 `h5-artifact.draft.json`，仍需人工 review / apply。
- `critique.json`：映射到当前 `file_critique`。
- `slice-plan.json`：映射到当前 `file_slice_plan`。
- `assets/`：映射到当前素材槽位和 assets manifest。
- `logs/`：映射到 AgentEvent / raw_log 或后续 run detail。

当前新增的 FileManifest 项：

```text
fileId: file_prototype_generated
filename: prototype.generated.html
kind: prototype_generated
status: adapter_mock
description: 旧 runner adapter mock 产物，不是旧 pipeline 真实输出。
openTarget: prototype
```

`GET /api/projects/current/files/file_prototype_generated` 已可读取该 mock HTML。

## 交互变化

本轮前端不重做 UI，也不新增明显主流程按钮。

可通过 daemon API 验证：

```bash
curl -X POST http://127.0.0.1:7458/api/runs/current/start \
  -H 'content-type: application/json' \
  --data '{"runType":"legacy_adapter_probe"}'
```

运行后左侧 AgentEvent / events.jsonl 会出现：

- `run_started`：准备旧 runner adapter。
- `assistant_message`：正在生成 legacy-task 输入目录。
- `file_written`：写入 `legacy-task/input/brief.md`。
- `file_written`：写入 `legacy-task/input/h5-artifact.json`。
- `file_written`：写入 `legacy-task/config/task.json`。
- `file_written`：写入 `prototype.generated.html`。
- `assistant_message`：明确说明本轮为 mock adapter，未执行旧 pipeline。
- `run_finished`。

## 本轮没有做什么

- 没有运行 `run_local_prototype.py`。
- 没有运行 `run_local_h5.py`。
- 没有调用 Dify / ComfyUI。
- 没有调用真实生图服务。
- 没有复制旧项目整体文件。
- 没有把旧 runner 输出写回旧项目目录。
- 没有覆盖正式 `prototype.html`。
- 没有覆盖正式 `h5-artifact.json`。
- 没有把 mock adapter 伪装成真实旧 pipeline 输出。

## 验收结果

- `pnpm typecheck` 通过。
- `pnpm --filter @h5-workbench/web build` 通过。
- `GET /api/health` 正常返回 `ok: true`。
- `POST /api/runs/current/start {"runType":"legacy_adapter_probe"}` 可启动。
- `legacy-task/input/brief.md` 存在。
- `legacy-task/input/h5-artifact.json` 存在。
- `legacy-task/input/design-system.json` 存在。
- `legacy-task/input/image-slots.json` 存在。
- `legacy-task/input/assets-manifest.json` 存在。
- `legacy-task/config/task.json` 存在。
- `legacy-task/output/prototype.generated.html` 存在。
- `legacy-task/output/legacy-output-manifest.json` 存在。
- FileManifest 已出现 `prototype.generated.html`，状态为 `adapter_mock`。
- `GET /api/projects/current/files/file_prototype_generated` 可读取，contentType 为 `text/html; charset=utf-8`。
- `events.jsonl` 明确包含「mock adapter」「没有执行 run_local_prototype.py、run_local_h5.py 或任何真实生图服务」。
- `legacy_adapter_probe` 运行前后，正式 `prototype.html` 的 SHA-256 保持不变。
- `legacy_adapter_probe` 运行前后，正式 `h5-artifact.json` 的 SHA-256 保持不变。
- 前端 `http://127.0.0.1:5178/` 返回 `200 OK`。

## 尚未解决

- 还没有做旧 runner dry-run。
- 还没有检查旧 Python runner 在当前机器的依赖环境。
- 还没有把旧 `run_local_prototype.py` 产物真实映射到当前工作台。
- 还没有把旧项目的高质量 HTML wireframe 生成能力接进 daemon。
- 还没有真实长图生成和真实 ZIP 导出。

## 下一步建议

1. 下一轮先做旧 runner dry-run 设计：只检查入口、参数、Python 环境和输出目录，不生成真实图片。
2. 先接 `run_local_prototype.py`，目标是生成高质量 `prototype.generated.html`，不要直接覆盖正式 `prototype.html`。
3. 等 `prototype.generated.html` 可稳定产生后，再做人工 Review & Apply，沿用当前 draft apply 的安全模式。
4. 暂缓接 `run_local_h5.py` 和真实生图，直到原型生成 adapter 稳定。

## 给下一个 Codex 窗口的提示

先读：

- `00_新工作台主入口.md`
- `AGENTS.md`
- `11_开发日志/00_当前开发状态.md`
- `11_开发日志/2026-07-10_旧项目原型生成能力审计.md`
- `11_开发日志/2026-07-10_旧Runner-Adapter设计与最小落地.md`

然后继续：

- 不要推翻现有「一级 Home + 二级 Project Workspace」。
- 不要真实运行旧 pipeline，除非任务明确进入 dry-run 或真实接入阶段。
- 先围绕 `apps/daemon/src/legacy-adapter/` 和 `apps/daemon/src/runners/legacyAdapterProbeRunner.ts` 扩展。
- 保持所有旧 runner 输出写入新项目 `.h5-workbench/projects/current/`，不要写回旧项目目录。

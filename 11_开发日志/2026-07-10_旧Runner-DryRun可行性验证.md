# 旧 Runner DryRun 可行性验证

日期：2026-07-10

## 本次修改背景

第十四步已经完成旧 runner Adapter 的协议和 mock 落地：

- `legacy-task/input/` 已能表达当前工作台输入。
- `legacy-task/config/task.json` 已能记录 adapter 配置。
- `prototype.generated.html` mock 产物已能进入 FileManifest。
- 但此前没有真实运行旧 Python pipeline。

本轮进入第十五步，只做旧 runner dry-run 可行性验证，不替换当前主流程，不覆盖正式文件，不接真实生图。

## 本次目标

本轮只回答三个问题：

1. 旧 runner 能不能被当前 daemon 调起来？
2. 旧 runner 需要哪些环境、依赖、输入文件？
3. 旧 runner 输出能不能被当前 adapter 识别并映射？

## 旧 runner 入口定位结果

实际找到的旧代码项目：

```text
/Users/yiming/Documents/Codex/2026-06-17/h5-kv-h5-demo-dify-llm
```

实际找到的旧资料项目：

```text
/Users/yiming/Desktop/h5自动生图项目文档（正在跑）
```

实际找到的 runner 入口：

```text
/Users/yiming/Documents/Codex/2026-06-17/h5-kv-h5-demo-dify-llm/h5_agent/runner/run_local_prototype.py
/Users/yiming/Documents/Codex/2026-06-17/h5-kv-h5-demo-dify-llm/h5_agent/runner/run_local_h5.py
```

本轮实际执行：

```text
run_local_prototype.py
```

本轮没有执行：

```text
run_local_h5.py
```

原因：`run_local_h5.py` 会进入分段生图和拼接链路，即使支持 `mock://local`，也不属于本轮“只验证旧 runner dry-run”的范围。

## 依赖与输入要求

旧代码项目依赖声明：

```text
/Users/yiming/Documents/Codex/2026-06-17/h5-kv-h5-demo-dify-llm/pyproject.toml
```

关键信息：

- `requires-python = >=3.11`
- 依赖包括：`fastapi`、`uvicorn`、`pillow`、`playwright`、`pydantic`、`requests`

本机实际执行 Python：

```text
/Users/yiming/Documents/Codex/2026-06-17/h5-kv-h5-demo-dify-llm/.venv/bin/python
```

版本：

```text
Python 3.14.5
```

旧 `run_local_prototype.py` 的输入要求：

- `content.md`：必需，活动内容。
- `special_requirements.md` 或 `requirements.md`：可选。
- `run_config.json`：可选，但可指定 `target_language` 和 `kv_path`。
- `kv.jpg` / `kv.png` / `kv.jpeg`：旧 `load_task_input()` 实际要求 KV 图，即使只跑 prototype。

本轮为了验证 runner 能否被 daemon 调起，在 `legacy-task/dry-run/input/kv.png` 写入了 dry-run 专用占位图。它不是正式素材，也不会进入当前工作台正式素材槽位。

## 修改文件

- `10_产品代码包/packages/agent-contract/src/index.ts`：新增 `legacy_runner_dry_run` runType；新增 `prototype_dry_run`、`legacy_dry_run_report` 文件类型；新增 `dry_run_success`、`dry_run_failed`、`dependency_missing`、`unsafe_to_run` 文件状态。
- `10_产品代码包/apps/daemon/src/runners/legacyRunnerDryRun.ts`：新增旧 runner dry-run runner。
- `10_产品代码包/apps/daemon/src/runners/index.ts`：注册 `legacyRunnerDryRun`。
- `10_产品代码包/apps/daemon/src/server.ts`：允许 `POST /api/runs/current/start` 启动 `legacy_runner_dry_run`。
- `10_产品代码包/apps/daemon/src/projectStore.ts`：支持读取 `file_legacy_dry_run_report` 和 `file_prototype_dry_run`。
- `10_产品代码包/apps/web/src/components/DesignFilesView.tsx`：识别 dry-run 文件类型和状态标签。
- `10_产品代码包/apps/web/src/styles.css`：补充 dry-run 状态样式。
- `11_开发日志/00_当前开发状态.md`：更新当前状态。

## dry-run 执行方式

通过现有 daemon run API 启动：

```bash
curl -X POST http://127.0.0.1:7458/api/runs/current/start \
  -H 'content-type: application/json' \
  --data '{"runType":"legacy_runner_dry_run"}'
```

daemon 实际执行命令：

```bash
/Users/yiming/Documents/Codex/2026-06-17/h5-kv-h5-demo-dify-llm/.venv/bin/python \
  -m h5_agent.runner.run_local_prototype \
  --task-dir /Users/yiming/Desktop/H5自动生图工作台/10_产品代码包/.h5-workbench/projects/current/legacy-task/dry-run/input \
  --output-root /Users/yiming/Desktop/H5自动生图工作台/10_产品代码包/.h5-workbench/projects/current/legacy-task/dry-run/output \
  --job-id legacy-dry-run-mock-run-1783643321347
```

cwd：

```text
/Users/yiming/Documents/Codex/2026-06-17/h5-kv-h5-demo-dify-llm
```

超时保护：

```text
120000ms
```

## dry-run 输出目录

本轮所有 dry-run 文件都写在：

```text
10_产品代码包/.h5-workbench/projects/current/legacy-task/dry-run/
```

固定日志文件：

```text
command.json
stdout.log
stderr.log
exit-code.txt
dry-run-report.md
prototype.dry-run.html
```

旧 runner 原始输出目录：

```text
legacy-task/dry-run/output/legacy-dry-run-mock-run-1783643321347/
```

旧 runner 本轮实际产物：

```text
01_blueprint.json
02_prototype.html
03_prototype_full.png
04_slices/
07_quality_report.md
run_summary.json
```

adapter 识别并复制：

```text
02_prototype.html
-> legacy-task/dry-run/prototype.dry-run.html
```

## FileManifest 映射

本轮新增：

```text
file_legacy_dry_run_report
filename: dry-run-report.md
kind: legacy_dry_run_report
status: dry_run_success
openTarget: design-files
```

以及：

```text
file_prototype_dry_run
filename: prototype.dry-run.html
kind: prototype_dry_run
status: dry_run_success
openTarget: prototype
```

文件读取 API：

```text
GET /api/projects/current/files/file_legacy_dry_run_report
GET /api/projects/current/files/file_prototype_dry_run
```

均已验证可读取。

## 执行结果

本轮 dry-run 真实执行成功。

- exitCode：`0`
- stdout：旧 runner 输出 `Local H5 Agent MVP finished`。
- stderr：空。
- 是否产出 prototype：是。
- 是否产出 artifact/blueprint：是。
- 是否产出切片：是。
- 是否产出质检报告：是。
- 是否使用占位 KV：是，仅 dry-run 使用。

这说明：

1. 当前 daemon 可以调起旧 `run_local_prototype.py`。
2. 当前机器上旧项目 `.venv`、Pillow、Playwright 等依赖在本次任务下可用。
3. 旧 runner 输出能被当前 adapter 识别，并可映射为 `prototype.dry-run.html`。

## 安全验证

本轮没有覆盖正式文件。

运行前后正式文件 SHA-256 保持一致：

```text
prototype.html:
855a1aecac54c45a4c915c3a3ba610853ca74aef11f5337aaa66b124b73670fd

h5-artifact.json:
00af5829ea39929065b3f0e8787c358d49c9c9b0500c64c78537943d08a8c2b5
```

本轮没有做：

- 没有运行 `run_local_h5.py`。
- 没有接真实生图服务。
- 没有覆盖正式 `prototype.html`。
- 没有覆盖正式 `h5-artifact.json`。
- 没有把 dry-run 产物自动应用到正式原型。
- 没有删除或修改旧项目文件。
- 没有大改 UI。

## 验收结果

- `pnpm typecheck` 通过。
- `pnpm --filter @h5-workbench/web build` 通过。
- `POST /api/runs/current/start {"runType":"legacy_runner_dry_run"}` 可启动。
- `legacy-task/dry-run/dry-run-report.md` 已写入。
- `stdout.log` 已写入。
- `stderr.log` 已写入。
- `exit-code.txt` 已写入，内容为 `0`。
- `legacy-task/dry-run/prototype.dry-run.html` 已写入。
- 旧 runner 原始产物在 `legacy-task/dry-run/output/legacy-dry-run-mock-run-1783643321347/`。
- FileManifest 可看到 `dry-run-report.md` 和 `prototype.dry-run.html`。
- `GET /api/projects/current/files/file_legacy_dry_run_report` 可读取。
- `GET /api/projects/current/files/file_prototype_dry_run` 可读取。
- 正式 `prototype.html` 未被覆盖。
- 正式 `h5-artifact.json` 未被覆盖。
- 前端 `http://127.0.0.1:5178/` 返回 `200 OK`。

## 当前限制

- dry-run 使用的是占位 KV，不代表真实用户 KV 素材已经接入。
- 旧 runner 生成的是它自己的 deterministic HTML wireframe，还没有和当前 `H5Artifact.sections` 做字段级对齐。
- 当前只是识别并复制 `02_prototype.html`，还没有实现 Review & Apply。
- 还没有把 `01_blueprint.json` 映射为 `h5-artifact.draft.json`。
- 还没有接入 `run_local_h5.py`、分段生图、真实长图和导出 ZIP。

## 下一步建议

1. 做 `run_local_prototype.py` 的真实 Prototype Adapter：把旧 runner 输出稳定映射成 `prototype.generated.html` 和候选 artifact/critique，而不是继续只保留 dry-run。
2. 增加 Prototype Generated Review & Apply：用户手动查看并确认后，才允许更新正式 `prototype.html`。
3. 做当前 H5Artifact 与旧 runner blueprint 的字段对齐审计，确认哪些字段可以互转，哪些只能作为生成中间层。
4. 先把素材/KV 文件落盘到 `assets/`，再让旧 runner 使用真实 KV，替换当前 dry-run 占位图。
5. 暂缓真实长图 runner；等 prototype adapter 稳定后，再考虑 `run_local_h5.py --gpt-image-endpoint mock://local`。

## 给下一个 Codex 窗口的提示

先读：

- `00_新工作台主入口.md`
- `AGENTS.md`
- `11_开发日志/00_当前开发状态.md`
- `11_开发日志/2026-07-10_旧Runner-Adapter设计与最小落地.md`
- `11_开发日志/2026-07-10_旧Runner-DryRun可行性验证.md`

然后继续：

- 不要推翻现有两级工作台结构。
- 不要把旧 runner 变成产品入口。
- 不要直接覆盖正式 `prototype.html`。
- 下一步应优先做 `prototype.generated.html` 的 Review & Apply，而不是直接接真实生图。

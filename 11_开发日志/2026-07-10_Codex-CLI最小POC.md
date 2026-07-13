# 2026-07-10 Codex CLI 最小 POC

## 本次修改背景

第六步已经抽出 Runner Adapter 层，daemon 可以通过 `runType` 选择不同 runner。第七步只验证一件事：daemon 是否能通过 runner 调用本机 Codex CLI，并把一个很小的结果写回当前项目目录。

本轮不做完整 H5 生成，不接真实生图，不让 Codex 修改 H5 Artifact，也不重做 UI。

## 本次目标

- 检查本机 `codex` 命令是否可用。
- 新增 `codex_cli_probe` runType。
- 让 daemon 通过 runner 调用 Codex CLI。
- 读取 `10_产品代码包/.h5-workbench/projects/current/brief.md`。
- 生成 `10_产品代码包/.h5-workbench/projects/current/codex-probe-result.md`。
- 通过 SSE 产出 `run_started`、`assistant_message`、`file_written`、`run_finished` 或 `error`。
- 保持 `long_image_generation` mock runner 不受影响。

## 本机 Codex CLI 检查

- `command -v codex`：`/opt/homebrew/bin/codex`
- `codex --version`：`codex-cli 0.143.0`
- `codex exec --help`：确认可使用非交互模式、`--cd`、`--sandbox`、`--skip-git-repo-check`、`--ephemeral`、`--output-last-message`。

## 修改文件

- `10_产品代码包/packages/agent-contract/src/index.ts`：新增 `RunType = "codex_cli_probe"`，新增 `FileKind = "codex_probe"`。
- `10_产品代码包/apps/daemon/src/runners/codexCliRunner.ts`：从 skeleton 改为最小 POC runner。
- `10_产品代码包/apps/daemon/src/runners/index.ts`：注册 `codexCliRunner`。
- `10_产品代码包/apps/daemon/src/server.ts`：允许 `POST /api/runs/current/start` 接收 `codex_cli_probe`。
- `10_产品代码包/apps/daemon/src/projectStore.ts`：支持读取 `file_codex_probe`，映射到 `codex-probe-result.md`。
- `11_开发日志/00_当前开发状态.md`：更新当前阶段状态。
- `11_开发日志/2026-07-10_Codex-CLI最小POC.md`：新增本日志。

## POC 执行方式

API：

```http
POST /api/runs/current/start
Content-Type: application/json

{
  "runType": "codex_cli_probe"
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
  --output-last-message <current project dir>/codex-probe-result.md
  -
```

说明：

- `cwd` 和 `--cd` 都限制在 `10_产品代码包/.h5-workbench/projects/current/`。
- prompt 中明确要求不要修改已有文件。
- `--sandbox read-only` 用于避免 Codex 修改项目文件。
- `codex-probe-result.md` 由 `--output-last-message` 写入。
- runner 只读取 `brief.md`，不会修改 `h5-artifact.json`、`prototype.html` 或生图产物。

## 本轮验证结果

- `pnpm typecheck` 通过。
- `pnpm --filter @h5-workbench/web build` 通过。
- `GET /api/health` 正常。
- `POST /api/runs/current/start` with `codex_cli_probe` 返回 `202`。
- SSE 能看到：
  - `run_started`
  - `assistant_message`
  - `raw_log: codex-cli 0.143.0`
  - `file_written: codex-probe-result.md`
  - `run_finished`
- 已生成：`10_产品代码包/.h5-workbench/projects/current/codex-probe-result.md`。
- `GET /api/projects/current/files/file_codex_probe` 能读取结果内容。
- `long_image_generation` mock runner 仍能完整跑完，`slice-plan.json`、`long-image.png`、`export.zip` 状态仍会变为 `done`。
- 前端 `http://127.0.0.1:5178/` 仍可访问。

## 当前结果文件内容示例

```md
# Codex Probe Result
- 一句话理解：这是一个面向葡语用户的游戏活动 H5，需要突出奖励、参与方式和底部转化 CTA，并营造强活动感。
- 建议 H5 类型：game_campaign
- 建议理由：需求核心是游戏活动传播与转化，重点信息包括奖励、参与步骤和 CTA，符合游戏活动型 H5 的结构。
```

## 已知情况

- 第一次验证时使用了 `--ask-for-approval never`，当前 Codex CLI 的 `exec` 子命令不支持该参数，已移除。
- Codex CLI 会在 stderr 中输出一些自身 plugin/config warning；runner 会把这些归入 `raw_log`，不会影响 POC 成功。
- 本轮没有新增明显前端入口，主要通过 daemon API 验证。
- `codex-probe-result.md` 没有加入右侧 7 个核心生产文件清单，避免改变当前 UI 信息架构；但可以通过 `file_codex_probe` API 读取。

## 尚未完成

- 未让 Codex CLI 生成 H5 Artifact。
- 未让 Codex CLI 生成 prototype.html。
- 未接真实生图服务。
- 未接旧 Python runner。
- 未做 Codex CLI 多轮交互、错误重试、进度事件细分。
- 未把 Codex probe 暴露到前端按钮。

## 下一步建议

1. 增加右侧「设计文件」的文件详情预览，让 `codex-probe-result.md` 可以被查看。
2. 把 Codex CLI runner 的下一步限制在“生成 prototype 建议 / 缺失信息补问”，不要直接生成完整 H5。
3. 增加 runner 级 timeout、取消和错误码归一化显示。
4. 把 Codex CLI stderr 的 warning 降噪，只保留必要 raw log。
5. 继续坚持：前端只消费结构化 `AgentEvent` / `RunState` / `FileManifest`，不要直接解析 raw CLI 文本。

## Guardian 结论

本轮没有跑偏。Codex CLI 被放在 daemon runner 层之后，只作为执行引擎做最小 POC；前端 UI、Home + Project Workspace 结构、长图 mock 流程和 H5 Artifact 主链路都没有被推翻。当前产物只是 `codex-probe-result.md`，没有让 Codex 修改 H5 协议或生成真实长图。

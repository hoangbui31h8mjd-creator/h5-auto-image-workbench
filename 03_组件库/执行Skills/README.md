# H5 Agent Skills

这里是本地 Codex H5 Agent 的可执行 Skill 入口。

## 当前文件

- `H5_Agent_组件Skill_执行版_v0.1.md`：组件选择、HTML 标记、模块边界和禁切标记。
- `H5_Agent_主题Skill_执行版_v0.1.md`：KV 使用模式、视觉风格契约、模块视觉计划。
- `H5_Agent_切图Skill_执行版_v0.1.md`：按模块边界规划 1-N 切片，并为 80px 参考带预留 1:3 高度预算。
- `H5_Agent_生图PromptSkill_执行版_v0.1.md`：把 KV、原型切片、主题契约和参考带语义转成每段 Image2 提示词。
- `component_registry_v0.1.json`：本地 Pipeline 可直接读取的组件注册表。
- `skill_manifest.json`：本目录的机器可读清单。

原始 Skill 已移出当前启用目录，归档在：

```text
h5_agent/knowledge/archive/original_skills/
```

它们只用于追溯来源，不作为当前 Pipeline 的执行契约。

补充图文规范来源在：

```text
h5_agent/knowledge/figma_sources/
```

其中 `parsed/figma_spec_components.json` 是从 Figma/HTML 组件规范解析出的机器可读组件规格，可作为 `component_registry_v0.1.json` 的补充依据。

## 使用原则

本地 Agent 运行时应优先读取执行版：

```text
组件 Skill 执行版
-> 组件注册表
-> 主题 Skill 执行版
-> 切图 Skill 执行版
-> 生图 Prompt Skill 执行版
```

原始 Skill 只作为归档资料，不直接作为 Pipeline 的执行契约。

## 在 Pipeline 中的位置

```text
knowledge.load_skill_bundle()
-> 读取执行版 Skill、组件注册表、Figma 解析规格
-> prototype 生成蓝图
-> prototype 生成 HTML
-> slicing 按切图 Skill、data-section / data-no-cut / data-cut-after 切片
-> generation 按生图 Prompt Skill 生成分段提示词
-> quality 按组件和主题规则质检
```

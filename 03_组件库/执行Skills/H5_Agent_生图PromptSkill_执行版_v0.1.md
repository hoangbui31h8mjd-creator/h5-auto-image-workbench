# H5 Agent 生图 Prompt Skill 执行版 v0.1

> 适用阶段：Codex 本地 H5 生成 Agent 的 Image2 生图层  
> 目标：把 KV、原型切片、主题契约和切图信息转成稳定、可执行、低歧义的分段生图提示词。

---

## 1. 使用边界

本 Skill 只负责“怎么写生图提示词”，不负责页面结构、组件规范、切点计算或最终图像拼接。

它应该读取：

- 组件 Skill 产出的 HTML/模块结构。
- 主题 Skill 产出的 KV 使用模式、KV 风格契约、模块视觉计划和负向视觉规则。
- 切图 Skill 产出的切片、切点和隐藏参考带策略。
- 用户的内容信息、默认目标语言 `pt-BR` 和可选特殊要求。

它不应该：

- 重新决定页面模块顺序。
- 改变原型切片比例。
- 把隐藏参考带写成画面可见标注。
- 把失败复盘式语言写进提示词。

---

## 2. 与主题 Skill 的强依赖

生图 Prompt Skill 不能单独发挥，必须把主题 Skill 的输出作为提示词主输入。

主题 Skill 应提供：

```json
{
  "kv_mode": "style_reference|hero_asset|locked_asset",
  "style_contract": {
    "main_colors": [],
    "accent_colors": [],
    "material": [],
    "edge": "",
    "lighting": "",
    "perspective": "",
    "texture": ""
  },
  "visual_motifs": [],
  "module_visual_plan": [
    {
      "module_id": "",
      "component_id": "",
      "visual_role": "",
      "background_theme": "",
      "main_elements": [],
      "component_treatment": "",
      "continuity": ""
    }
  ],
  "negative_visual_rules": []
}
```

Prompt Skill 必须逐段读取 `module_visual_plan`，根据当前原型切片包含的模块，把对应模块的视觉处理写入本段提示词。

不能出现：

```text
主题 Skill 写“暗色金属任务面板”，生图 Prompt 却写“白底简洁卡片”。
主题 Skill 写“KV 作为 locked_asset”，生图 Prompt 却要求重绘 KV。
主题 Skill 写“FAQ 背景克制”，生图 Prompt 却给 FAQ 加强光效和复杂背景。
```

---

## 3. 每段提示词必须包含

```text
1. 原型切片是布局主参考。
2. KV 是风格主参考。
3. 当前段相关的主题 Skill 模块视觉计划。
4. 保持模块顺序、相对位置、卡片数量、按钮位置和文字层级。
5. 灰色占位转换为主题化图标、道具、图片面板、奖励卡或纹理区域。
6. 可见文案使用目标语言；默认目标语言为巴西葡语 pt-BR，用户明确指定其他语言时才覆盖。
7. 不生成安全区、切口、接缝、尺寸数字、wireframe 标签等生产标记。
8. 第 2 段起，顶部参考带是上一段真实画面延续，不是 banner、标题或新模块。
```

---

## 4. 模块视觉计划合并规则

每段提示词应该包含当前段内模块的视觉计划摘要。

示例：

```text
Module visual plan for this segment:
- task_list: dark tactical panels, mission badges, subtle smoke; keep vertical task list and gold CTA buttons.
- reward_grid: reward props with amber rim light; keep 2-column card grid and item-name hierarchy.
- faq: restrained dark panel, high readability, minimal background elements.
```

合并规则：

- 只放当前段相关模块，不把全页所有模块都塞进每段。
- 每个模块最多 1-2 句，避免提示词过载。
- 模块视觉计划必须服从原型布局，不能为了风格改变卡片数量和位置。
- 文案密集模块优先可读性，视觉复杂度降级。
- 奖励、CTA、排行榜等重点模块可以提高视觉强度。

---

## 5. 参考带怎么写

提示词里不要写具体像素数字，例如不要写“80px”。

应该写：

```text
The top reference strip is the bottom continuation from the previous generated segment.
Treat it as locked visual context and continue downward from it.
Do not redesign it as a new header, banner, divider, or UI block.
Begin the current segment's own layout below that reference strip.
```

原因：

- 具体像素是机器参数，不是视觉内容。
- 写进提示词后，模型可能把“80px”“安全区”“接缝”等字样画出来。
- 参考带高度由切图/生图代码控制，Prompt 只解释它的语义。

---

## 6. 推荐提示词骨架

```text
Generate segment {index} of {total_segments} for a vertical mobile commercial H5 long image.

Input references:
- KV image: visual style reference only. Extract mood, palette, lighting, material language, composition energy, and campaign tone.
- prototype_slice_{index}: primary layout and information-architecture blueprint. Interpret gray boxes, placeholder labels, module names and rough copy as production instructions.
- If a top reference strip is present, it is the locked visual continuation from the previous generated segment, not a new UI module.

Production rules:
- Follow the prototype structure strictly: section order, relative positions, section height, card count, button placement, heading hierarchy and reading order.
- Convert low-fidelity placeholders into themed icons, item art, image panels, reward cards, ranking/list modules, rule panels, CTA components and footer modules while preserving size and placement.
- Render only final user-facing campaign content; exclude internal production annotations, measurements, debug labels and Chinese text.
- Use {target_language} for all visible user-facing copy.
- The output should look like a finished mobile campaign H5 segment.

Style contract from Theme Skill:
{style_contract}

Module visual plan from Theme Skill:
{module_visual_plan_for_this_segment}

Negative visual rules from Theme Skill:
{negative_visual_rules}

Segment continuity:
{continuity_instruction}

Additional user constraints:
{constraints}
```

---

## 7. 禁止写法

不要写：

- “不要再……”
- “之前生成错了……”
- “你刚才……”
- “修复上一次的问题……”
- “80px 安全带”
- “接缝区域”
- “切口”
- “灰色原型图”
- “wireframe labels should remain”

这些词容易让模型把调试语言画进成品，或者把原型当成最终视觉元素保留。

---

## 8. 正向施工语言

建议使用：

```text
Use the provided prototype slice as the primary layout reference.
Use the KV image as the visual style reference.
Preserve the section order, card count, button positions and text hierarchy.
Render low-fidelity placeholders as themed icons, reward props or image panels while preserving their size and placement.
Use the top reference strip only as locked visual continuity context.
```

---

## 9. 分段差异

第 1 段：

```text
Establish the campaign visual world and first-screen impact.
Make the bottom texture naturally extensible for the next segment.
```

中间段：

```text
Continue naturally from the top reference strip.
Follow the current prototype layout below the strip.
Make the bottom texture naturally extensible for the next segment.
```

最后一段：

```text
Continue naturally from the top reference strip.
Follow the current prototype layout below the strip.
Finish with final CTA, rules, footer or closing content.
```

---

## 10. 质检规则

生图提示词生成后必须检查：

- 是否明确原型切片是布局主参考。
- 是否明确 KV 是风格主参考，并遵守主题 Skill 的 `kv_mode`。
- 是否包含当前段相关的 `module_visual_plan`。
- 是否没有写出与 `style_contract` 矛盾的材质、光照、颜色或组件处理。
- 是否说明顶部参考带是视觉连续上下文。
- 是否没有写具体参考带像素数字。
- 是否没有出现“安全区、切口、接缝、wireframe 保留”等容易被画出来的词。
- 是否明确目标语言。
- 是否没有把 KV 主体强行复制进每一段。

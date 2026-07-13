# H5 Agent 主题 Skill 执行版 v0.1

> 来源基础：`h5-module-theme.md`  
> 适用阶段：Codex 本地 H5 生成 Agent  
> 目标：把 KV 风格分析和模块视觉主题规划改造成可执行规则

---

## 1. 使用边界

本 Skill 负责视觉风格理解和模块主题分配。

它不负责：

- 决定页面模块顺序。
- 生成 HTML 组件结构。
- 规划具体切点。
- 替代组件 Skill。
- 直接编写每段生图提示词。

它需要和组件 Skill 配合：

```text
组件 Skill 决定页面结构
主题 Skill 决定视觉风格和模块画面语言
生图 Prompt Skill 决定每段提示词怎么写
```

---

## 2. KV 使用模式

原始 Skill 里写了“KV 完整保留”。这个规则在当前项目中需要拆成三种模式，不能一刀切。

### 2.1 hero_asset 模式

产品包默认模式。

KV 作为 H5 第一屏头图素材。

适用：

- 用户提供头图或 KV，但没有额外说明。
- 用户明确说“这张 KV 基本就是头图”。
- 业务要求保留原活动主视觉。

规则：

- HTML 原型必须把 KV 嵌入首屏顶部。
- 生图阶段不得改动 KV 主体信息。
- 如果需要风格延展，只从 KV 底部之后开始。

### 2.2 style_reference 模式

KV 只作为风格参考，不直接嵌入 HTML 原型，也不直接粘进最终图。

适用：

- 用户只说“参考这张 KV 的风格”。
- KV 是风格锚点，不一定是最终头图。
- 用户明确要求不要把 KV 作为头图。

### 2.3 locked_asset 模式

KV 必须像素级保护。

适用：

- 品牌方要求 KV 完整保留。
- 图中有版权、logo、人物、固定文案。

规则：

- 不裁剪、不改色、不重绘、不加新元素。
- 只能在 KV 下方做自然延展。

---

## 3. KV 分析输出

主题 Skill 必须输出结构化风格契约，而不是只写“战斗风、科技风”。

```json
{
  "kv_mode": "style_reference|hero_asset|locked_asset",
  "style_contract": {
    "main_colors": ["#000000"],
    "accent_colors": ["#F4B23A"],
    "material": ["metal", "smoke", "dust"],
    "edge": "hard_cut|soft_glow|rough_texture",
    "lighting": "backlight|spotlight|flat",
    "perspective": "front|top|low_angle",
    "texture": "clean_flat|grunge|realistic_photo|3d_soft"
  },
  "visual_motifs": ["tank", "smoke", "gold_reward", "battlefield"],
  "negative_visual_rules": ["不要使用无关卡通元素", "不要出现白底成品区域"]
}
```

---

## 4. 风格观察三问

在写风格描述前，必须回答：

1. 线条粗细：均匀还是有变化？
2. 边缘状态：硬切、发光、毛边、磨损还是柔和？
3. 转角形态：直角、圆角、尖角、金属切角还是不规则？

禁止只写：

- 插画风
- 3D 风
- 写实风
- 高级感
- 科技感

必须写成可执行描述，例如：

```text
黑金战争风，暗部大面积压低，金色高光用于按钮和奖励，
边缘为硬切金属切角，背景有烟尘和战场逆光，文字层级粗重。
```

---

## 5. 模块主题分配

每个模块最多分配 3 个主要视觉元素，避免提示词过载。

输出格式：

```json
{
  "module_visual_plan": [
    {
      "module_id": "task_list_01",
      "component_id": "task_list",
      "visual_role": "任务推进区",
      "background_theme": "暗色战术面板",
      "main_elements": ["金属边框", "任务徽章", "微弱烟尘"],
      "component_treatment": "任务卡保持纵向列表，按钮高亮为金色",
      "continuity": "承接上一段暗色战场背景"
    }
  ]
}
```

原则：

- 文案密集模块：背景必须克制，优先保证信息可读。
- 奖励模块：可以增强光效和道具感。
- 任务列表：强调状态、进度、按钮。
- 排行榜：强调排名、徽章、竞技感。
- FAQ / 规则：背景最简，避免干扰阅读。
- CTA：强化对比，明确点击动作。

---

## 6. 生图 Prompt 交接

主题 Skill 只输出风格契约和模块视觉计划，不直接写最终分段提示词。

交给生图 Prompt Skill 的信息必须包括：

```json
{
  "kv_mode": "style_reference|hero_asset|locked_asset",
  "style_contract": {},
  "visual_motifs": [],
  "module_visual_plan": [],
  "negative_visual_rules": []
}
```

具体每段提示词由 `H5_Agent_生图PromptSkill_执行版_v0.1.md` 生成。

要求：

- `style_contract` 是所有分段生图提示词的视觉总约束。
- `module_visual_plan` 是每段模块视觉处理的来源。
- `kv_mode` 决定 KV 是只做风格参考，还是作为头图/锁定素材保护。
- `negative_visual_rules` 必须传给生图 Prompt Skill，防止模型生成无关元素。
- 生图 Prompt Skill 不得写出与主题 Skill 冲突的风格、材质、光照和组件处理。

---

## 7. 分段续接语义

第 2 段及之后：

- 顶部参考带来自上一段最终生成图的底部。
- 它只用于视觉连续，不是 banner、标题或新模块。
- 拼接时应与同一块参考带重叠，并做羽化过渡。

写给 Prompt Skill 的语义是：

```text
The top reference strip is the bottom continuation from the previous generated segment.
Treat it as locked visual context and continue downward from it.
Do not redesign it as a new header or UI block.
```

---

## 8. 海外业务语言规则

默认语言为巴西葡语 `pt-BR`。

只有用户明确指定其他语言时，才覆盖默认语言。例如用户说“用英语”“中文版本”“西语版”，则按用户指定语言执行。

要求：

- 原型中的可见业务文案使用目标语言。
- 生图提示词中明确目标语言。
- 不混入中文按钮、中文标题或中文生产标记。
- 如果模型文字不稳定，优先保留文字区域和层级，后续由 OCR/人工复核。

---

## 9. 质检规则

主题规划后检查：

- 是否明确 KV 使用模式。
- 是否给出主色、辅色、材质、边缘、光照、纹理。
- 是否每个模块都有视觉角色。
- 是否给每个模块不超过 3 个主要元素。
- 是否避免白底成品区。
- 是否避免与活动无关的视觉元素。

生图后检查：

- 风格是否和 KV 同源。
- 模块顺序是否和原型一致。
- 灰色占位是否被合理主题化。
- 是否出现错误语言。
- 是否出现明显拉伸、压缩、错位。
- 段间是否有割裂或重复模块。

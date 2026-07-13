# H5 Agent 组件 Skill 执行版 v0.1

> 来源基础：`jumu-components.md`  
> 适用阶段：Codex 本地 H5 生成 Agent  
> 目标：把积木组件规范从“设计说明”改造成“可执行的原型生成规则”

---

## 1. 使用边界

本 Skill 只负责页面结构和组件规范，不负责最终视觉美化。

它应该用于：

- 根据活动需求选择页面模块。
- 生成蓝图 JSON。
- 生成低保真 HTML wireframe。
- 给切图 Skill 提供模块边界、优先切点和禁止切断区域。
- 给质检提供组件级规则。

它不应该用于：

- 直接生成最终视觉图。
- 决定 Image2 的写实风格细节。
- 在首屏以外重复嵌入 KV 或把后续模块做成最终视觉图。

---

## 2. 当前尺寸策略

当前 H5 Agent 链路默认跟随积木/Figma 规范，以 `828px` 作为原型画布。

换算规则：

```text
scale = current_canvas_width / 828
```

常用尺寸映射：

| 积木/Figma 规范 | 当前默认 |
|---|---|
| 画布 828px | 画布 828px |
| 内容区 748px | 内容区 748px |
| 左右边距 40px | 左右边距 40px |
| 组件间距 24-32px | 24-32px |
| 排行榜行高 128px | 128px |
| 任务列表行高 168px | 168px |
| 进度条 8px | 8px |
| 按钮 56/64/80/88/96px | 56/64/80/88/96px |

如果用户明确要求其他画布宽度，必须全链路统一换算；不要局部混用不同画布。

---

## 3. 原型 HTML 生成硬规则

### 3.1 必须做

- 输出完整自包含 HTML。
- 主容器宽度读取当前组件注册表，默认 `828px`。
- 使用黑白灰 wireframe。
- 真实模块名、按钮文案、任务、奖励、规则等必须可读。
- 每个主要模块必须有 `data-section`。
- 不希望被切断的区域必须有 `data-no-cut="true"`。
- 适合切片的位置可以使用 `data-cut-after="preferred"`。

### 3.2 禁止做

- 禁止在可见内容中写“安全区、40px、80px、切口、接缝、续画、拼接”。
- 禁止为了凑切片数量拉伸空白。
- 禁止把原型做成最终视觉稿。
- 禁止直接调用外链图片、外链字体、脚本。
- 禁止在每段顶部或底部画灰条、白条、虚线安全带。

---

## 4. HTML 标记规范

每个模块建议结构：

```html
<section
  class="h5-section h5-section--task-list"
  data-section="task_list"
  data-component="task_list"
  data-no-cut="true"
>
  <h2>Missões Diárias</h2>
  ...
</section>
```

模块之间的自然留白必须使用切片器可识别的标记。当前本地 renderer 会优先识别 `.module-gap` 和 `data-module-break`：

```html
<div class="module-gap" data-module-break="task_list" data-cut-after="preferred"></div>
```

说明：

- `data-section` 用于识别页面模块。
- `data-component` 用于质检组件类型。
- `data-no-cut` 用于告诉切片器不要切断此模块。
- `data-cut-after` 用于提示切片器优先在此处之后裁切。
- `data-module-break` 用于让 renderer 把模块间留白提取为切点候选。

---

## 5. 组件选择规则

### 5.0 当前 MVP 可执行组件

当前 Python 本地 MVP 的组件注册表只稳定支持以下 `component_id`：

```text
hero_placeholder
activity_intro
how_to_join
task_list
reward_grid
leaderboard
video_grid
hashtag_section
lottery_cards
faq
final_cta
footer
```

新 Codex 在没有扩展 `component_registry_v0.1.json` 和 `local_generator.py` renderer 前，不要输出注册表之外的 `component_id`。如果业务需要团队挑战、版本亮点、我的排名、规则说明等细分模块，先映射到以上可执行组件，或先补注册表和 renderer。

### 5.1 游戏版本 / 任务挑战类活动

推荐模块顺序：

```text
hero_placeholder
activity_intro
how_to_join
task_list
reward_grid
leaderboard
faq
final_cta
footer
```

必选组件：

- 活动介绍
- 任务列表
- 奖励卡
- CTA

可选组件：

- 排行榜
- 进度条
- FAQ
- 版本亮点
- 团队挑战

### 5.2 排名 / 榜单类活动

推荐模块顺序：

```text
hero_placeholder
activity_intro
leaderboard
reward_grid
faq
final_cta
```

必选组件：

- Tab 导航
- 排行榜
- 我的位置
- 奖励说明

### 5.3 投稿 / UGC 活动

推荐模块顺序：

```text
hero_placeholder
activity_intro
hashtag_section
video_grid
reward_grid
faq
final_cta
```

必选组件：

- Hashtag 拍摄
- 视频卡片
- 投票 / 点赞入口
- CTA

### 5.4 抽奖 / 翻卡活动

推荐模块顺序：

```text
hero_placeholder
activity_intro
lottery_cards
reward_grid
faq
final_cta
```

必选组件：

- 翻卡
- 剩余次数
- 奖品卡
- 规则说明

---

## 6. 组件执行表

| 组件 | component_id | 适用场景 | 禁止切断 | 推荐切点 |
|---|---|---|---|---|
| 头图区占位 | `hero_placeholder` | 首屏主视觉或 KV 预留 | 是 | 模块后 |
| 活动介绍 | `activity_intro` | 所有活动 | 是 | 模块后 |
| 参与方式 | `how_to_join` | 任务、挑战、报名 | 是 | 模块后 |
| 任务列表 | `task_list` | 任务挑战、签到、积分 | 是 | 每组任务后 |
| 奖励卡 / 奖励网格 | `reward_grid` | 奖励展示 | 是 | 模块后 |
| 排行榜 | `leaderboard` | 排名活动 | 是 | 榜单组后 |
| 视频卡片 | `video_grid` | 投稿、UGC、内容推荐 | 是 | 卡片组后 |
| Hashtag 拍摄 | `hashtag_section` | 投稿活动 | 是 | 模块后 |
| 抽奖翻卡 | `lottery_cards` | 抽奖活动 | 是 | 模块后 |
| FAQ | `faq` | 规则解释 | 否，单条不可切 | FAQ 条目之间 |
| 最终 CTA | `final_cta` | 收口转化 | 是 | 模块后 |
| 尾标 | `footer` | 版权、品牌尾部 | 是 | 页面结束 |

---

## 7. 蓝图 JSON 输出要求

生成 HTML 前，必须先生成蓝图。

蓝图至少包含：

```json
{
  "page": {
    "width": 828,
    "content_width": 748,
    "side_margin": 40,
    "target_language": "pt-BR",
    "height_strategy": "content_driven"
  },
  "modules": [
    {
      "id": "task_list_01",
      "component_id": "task_list",
      "title": "Missões Diárias",
      "purpose": "展示每日任务和奖励",
      "required_fields": ["task_title", "reward", "progress", "action"],
      "no_cut": true,
      "cut_after": "preferred"
    }
  ],
  "copy_map": {},
  "cut_hints": [],
  "quality_rules": []
}
```

---

## 8. 切图协作规则

组件 Skill 不直接决定切片数量和最终切点；这些由 `H5_Agent_切图Skill_执行版_v0.1.md` 负责。

组件 Skill 只需要在 HTML 原型里提供清晰的机器标记：

```html
<section data-section="task_list" data-component="task_list" data-no-cut="true">
  ...
</section>

<div class="module-gap" data-module-break="task_list" data-cut-after="preferred"></div>
```

协作原则：

```text
组件 Skill：把页面做成清晰模块，并标出哪里不能切、哪里适合切。
切图 Skill：根据这些标记、页面高度和模型比例限制，决定 1-N 个切点。
```

组件 Skill 不写固定切片数，不把 80px 参考带画到页面上，也不在可见内容里出现切口说明。

---

## 9. 质检规则

HTML 生成后必须检查：

- 是否存在完整 `<!doctype html>`。
- 是否主容器宽度等于当前组件注册表画布宽度，默认 `828px`。
- 是否每个主要模块都有 `data-section`。
- 是否任务、奖励、CTA、规则等核心模块缺失。
- 是否出现禁用可见词：安全区、40px、80px、切口、接缝、续画、拼接。
- 是否使用外链资源。
- 是否模块高度异常或出现大段无意义空白。

裁切后必须检查：

- 每张切片高度是否在模型限制内。
- 是否切断 `data-no-cut="true"` 模块。
- 是否按 `data-cut-after` 或模块边界裁切。
- 是否切片数量合理。

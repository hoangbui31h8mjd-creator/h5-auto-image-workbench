# H5 Agent 本地 MVP 质检报告

- 状态：PASS
- 页面尺寸：828 x 4947
- 模型比例上限：1:3，对应高度 2484px
- 参考带高度：80px
- 原型切片高度预算：2404px
- 切片数量：3
- 切点：[2321, 3965]
- 检查通过：11/11

## 检查项

- PASS：HTML 完整
- PASS：主容器宽度 828px
- PASS：包含 data-section
- PASS：包含 data-component
- PASS：包含模块切片标记
- PASS：没有可见生产术语
  - 详情：`{'found': []}`
- PASS：核心组件齐全
  - 详情：`{'missing': []}`
- PASS：切片高度不超过原型切片上限
  - 详情：`{'too_tall_slice_indexes': []}`
- PASS：第 2 段起加参考带后仍不超过 1:3
  - 详情：`{'too_tall_after_reference_band': []}`
- PASS：切片数量为 1-N
- PASS：切点来自模块边界候选

## 切片明细

- 第 1 段：y=0-2321，尺寸 828x2321，文件 `001.png`
- 第 2 段：y=2321-3965，尺寸 828x1644，文件 `002.png`
- 第 3 段：y=3965-4947，尺寸 828x982，文件 `003.png`

## 下一步建议

- 如果 HTML 结构和切片结果 OK，再接 Image2 API 做分段生图。
- 如果切片切到了主体内容，优先调整 HTML 中的 `module-gap` 和 `data-no-cut`。
- 接 Image2 时，第 2 段及之后要在顶部追加 80px 参考带，并保持总高度不超过 1:3。
- 如果原型内容不符合业务，优先调整组件 Skill 和蓝图生成规则。

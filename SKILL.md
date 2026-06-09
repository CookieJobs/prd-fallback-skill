---
name: prd-fallback
description: 读取飞书 PRD 文档，识别缺失的兜底逻辑和边界条件，在原文档中插入带短哈希 ID 的高亮建议块，每处提供 2~3 个方案供 PM 选择。使用方式：提供飞书文档 URL，说「帮我审查这份 PRD 的兜底逻辑」或「/prd-fallback <URL>」。
---

# PRD 兜底逻辑审查

## 前置准备（每次执行前必读）

在执行任何文档操作前，使用 Read 工具读取以下文件：

1. `~/.claude/skills/lark-shared/SKILL.md` — 认证和全局参数
2. `~/.claude/skills/lark-doc/references/lark-doc-fetch.md` — fetch 参数选择
3. `~/.claude/skills/lark-doc/references/lark-doc-xml.md` — callout XML 语法（插入阶段）
4. `~/.claude/skills/lark-doc/references/lark-doc-update.md` — block_insert_after 和 block_replace 用法（插入和替换阶段）
5. `~/.claude/skills/prd-review/references/analysis-prompt.md` — 边界条件识别规则
6. `~/.claude/skills/prd-review/references/xml-templates.md` — callout XML 模板

## 路径 A：初次审查 PRD

### 第 1 步：获取文档内容

```bash
lark-cli docs +fetch --api-version v2 --doc "<URL>" --scope all
```

若文档超过 8000 字，改为按章节分段获取，每段独立分析，最后汇总去重。判断方法：fetch 结果字数 > 8000 字时自动分段。去重标准：同一 `location_text`（或高度相似的文本）且同一 `category` 的条目视为重复，保留描述更详细的一条。

### 第 2 步：AI 分析边界条件

按 `references/analysis-prompt.md` 中的规则分析全文。输出严格符合其中定义的 JSON 数组格式。内部步骤，不向用户展示 JSON。

### 第 3 步：终端展示预览摘要

**展示所有分析结果，不得因"重要性"或"已有部分描述"等理由在预览前裁剪条目数量。过滤权在 PM，不在 AI——跳过逻辑仅在第 4 步由 PM 通过 `skip` 指令触发。**

向 PM 展示如下格式的预览（使用实际分析结果）：

```
📋 PRD 审查完成，发现 N 处需要补充兜底逻辑：

 #  位置                    类型          场景摘要
──────────────────────────────────────────────────────
 1  [section_label]        [🔴/🟡/💡 类型]  [trigger 前30字]
 2  ...

输入要跳过的编号（如：skip 3,7），或直接按回车全部插入：
```

颜色 emoji 映射：`red` → 🔴，`yellow` → 🟡，`green` → 💡

### 第 4 步：处理 PM 的确认输入

- PM 直接按回车 / 输入"确认" / "yes" / "ok"：对全部条目执行插入
- PM 输入 `skip 3,7`：从列表中去除编号 3 和 7，对剩余条目执行插入
- PM 输入具体说明（如"第3条不需要"）：根据语义判断后，复述解析结果「我理解为跳过第 X 条，其余全部插入，确认吗？」，PM 确认后再执行

### 第 5 步：生成 callout 块

对每个确认的条目：

1. **生成唯一 ID**：生成一个 4 字符小写十六进制字符串（字符范围：0-9, a-f）。先搜索文档已有内容，确认 `FB-` 后的 4 字符没有与已有 ID 重复。格式：`FB-xxxx`（如 `FB-a3f7`）。

2. **选择模板颜色**：按 `references/xml-templates.md` 的颜色决策表选择 yellow / red / green。callout 颜色须与预览摘要（第 3 步）中该条目的 emoji 颜色一致（🔴 = red，🟡 = yellow，💡 = green）。

3. **填充内容**：将 JSON 中的 `trigger`、`option_a`、`option_b`、`option_c`（如有）填入对应模板，替换 `FB-XXXX` 占位符为生成的 ID。

4. 方案描述要求：详细、用户感知视角，包含界面文案、状态变化、后续可操作路径。如 JSON 中的方案描述已经足够详细，直接使用；如过于简短，适当扩写。

### 第 6 步：获取目标 block ID 并批量插入

```bash
# 1. 用 JSON 中的 location_text 字段定位目标 block（每个条目单独 fetch）
lark-cli docs +fetch --api-version v2 --doc "<URL>" --detail with-ids --scope keyword --keyword "<JSON 中的 location_text>"
# 若关键词未命中或返回多个块，改用 --scope all 获取全文 block 列表后逐一人工比对定位
```

对每个目标段落（**从文档末尾往前处理，避免插入后位置偏移**）：

```bash
lark-cli docs +update --api-version v2 --doc "<URL>" \
  --command block_insert_after \
  --block-id "<目标段落的 block ID>" \
  --content '<callout emoji="🔍" background-color="light-yellow" border-color="yellow">
  <p><b>兜底建议</b> · 选定方案后删除其余方案行　<code>FB-xxxx</code></p>
  <p>⚠️ <b>触发场景：</b>...</p>
  <p><b>方案 A</b>　...</p>
  <p><b>方案 B</b>　...</p>
</callout>'
```

### 第 7 步：完成提示

```
✅ 已插入 N 个兜底建议块。

文档链接：<URL>

PM 操作指引：
- 🔴 红色块 = 高风险，必须处理；🟡 黄色块 = 建议处理；💡 绿色块 = 体验优化，可选
- 在每个高亮块中选择一个方案，删除其余方案行
- 对绿色（💡）建议块，可直接删除整块（体验建议，非必须）
- 若某个建议需要修改，复制块 ID（如 FB-a3f7）后在对话中告知我
```

---

## 路径 B：精修某个 callout 块

当 PM 提供一个 FB-xxxx ID 并说明修改方向时（无需特殊命令，自然对话触发）：

> **前提：** 若 PM 未在当前会话中提供文档链接，先询问「请提供 PRD 的飞书文档链接」，获得 URL 后再继续执行以下步骤。

### 第 1 步：定位目标块

```bash
lark-cli docs +fetch --api-version v2 --doc "<URL>" --detail with-ids --scope keyword --keyword "FB-xxxx"
```

读取包含该 ID 的 callout 块的当前内容和 block ID。

### 第 2 步：重写内容

根据 PM 的反馈方向重新生成方案，保持 `FB-xxxx` ID 不变，只更新文字内容。

### 第 3 步：替换块

```bash
lark-cli docs +update --api-version v2 --doc "<URL>" \
  --command block_replace \
  --block-id "<callout 的 block ID>" \
  --content '<callout ...>新内容</callout>'
```

### 边界情况

- ID 不存在：提示「未找到 FB-xxxx，请确认 ID 是否正确，或提供文档链接」
- PM 只想讨论方案，不写回文档：直接对话，询问确认后再写入
- 文档已被大幅修改导致 fetch 失败：要求 PM 重新提供文档链接

---

## 注意事项

- 插入顺序：**始终从文档末尾往前插入**，避免 block ID 偏移导致后续插入位置错误
- 不重复插入：每次执行前，先检查文档中已有的 `FB-` 模式，已有兜底建议的位置跳过
- callout 内只能使用 `<p>` 块，不能使用 `<h1>` `<ul>` 等其他块类型
- 批量插入中若某次 `block_insert_after` 返回错误，记录该条目编号，继续执行其余插入；全部完成后统一向 PM 报告「第 X 条插入失败（block ID: xxx），请手动处理」，不静默丢弃

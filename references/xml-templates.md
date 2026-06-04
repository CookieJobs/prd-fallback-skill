# Callout XML Templates

## Color Decision Table

| Color | When to use |
|-------|-------------|
| `light-yellow` / border `yellow` | Network errors, system errors, input validation, empty states, loading states |
| `light-red` / border `red` | Data loss risk, oversell, payment anomalies, permission bypass — anything that could cause irreversible harm |
| `light-green` / border `green` | UX improvement suggestions (nice-to-have, not required) |

## Yellow Template (most common)

```xml
<callout emoji="🔍" background-color="light-yellow" border-color="yellow">
  <p><b>兜底建议</b> · 选定方案后删除其余方案行　<code>FB-XXXX</code></p>
  <p>⚠️ <b>触发场景：</b>[一句话描述触发条件，含操作动作和失败原因]</p>
  <p><b>方案 A</b>　[详细方案——用户界面感知的完整交互描述，含提示文案、页面状态、后续可操作路径]</p>
  <p><b>方案 B</b>　[详细方案]</p>
  <p><b>方案 C</b>　[详细方案，如只有2个合理方案则省略此行]</p>
</callout>
```

## Red Template (high-risk)

```xml
<callout emoji="🔍" background-color="light-red" border-color="red">
  <p><b>兜底建议 ⚠️ 高风险</b> · 选定方案后删除其余方案行　<code>FB-XXXX</code></p>
  <p>🚨 <b>触发场景：</b>[触发条件，强调风险后果]</p>
  <p><b>方案 A</b>　[详细方案——用户界面感知的完整交互描述，含提示文案、页面状态、后续可操作路径]</p>
  <p><b>方案 B</b>　[详细方案——用户界面感知的完整交互描述，含提示文案、页面状态、后续可操作路径]</p>
  <p><b>方案 C</b>　[可选]</p>
</callout>
```

## Green Template (UX suggestion)

```xml
<callout emoji="💡" background-color="light-green" border-color="green">
  <p><b>体验建议</b> · 非必须，可按需采纳　<code>FB-XXXX</code></p>
  <p>💡 <b>场景：</b>[简要场景描述，说明为什么这是体验改进机会]</p>
  <p><b>方案 A</b>　[方案描述]</p>
  <p><b>方案 B</b>　[方案描述]</p>
</callout>
```

## Notes

- Replace `FB-XXXX` with the generated 4-char lowercase hex ID (characters: 0-9, a-f), e.g. `FB-a3f7`
- Replace all `[placeholder]` text with actual content
- `<code>` tag makes the ID visually distinct (monospace font in Feishu)
- The `<p>` tags inside callout are the only supported block type — do NOT use headings or lists inside callout
- Supported inline formatting inside `<p>`: `<b>`, `<code>`, emojis — do not use block elements like `<h1>`, `<ul>`, `<li>`

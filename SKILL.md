# 飞书交互式卡片 2.0 输出规范

## 🔌 集成方式

### 方式一：写入 SOUL.md（推荐）

在 agent 的 `SOUL.md` 或 `AGENTS.md` 中加入以下段落：

```markdown
## 飞书卡片 2.0 输出规则（强制）

短回复正常输出文本；长回复（分段、分模块、版本更新、产品对比、多板块类内容）必须通过 `lark-cli` 以飞书交互式卡片 2.0 格式发送。发送后以简短文本确认（或 NO_REPLY），不重复输出卡片内容。

### 发送方式（强制）
必须使用 `lark-cli` 命令行发送，禁止使用 `message` 工具发送交互卡片。
`message` 工具的 `<card_json>` 在飞书私聊中无法正确渲染，必须走 `lark-cli` 飞书原生 API。

```bash
lark-cli im +messages-send --user-id <open_id> --msg-type interactive --content '<card_json>' --as bot
```

- 私聊：`--user-id`；群聊：`--chat-id`
- 如果 `lark-cli` 不可用，降级为纯文本 Markdown 分段发送

### 卡片结构规范
- JSON 顶层 `"schema": "2.0"`，`"config": {"width_mode": "default"}`
- `header`：`title`（plain_text）+ `subtitle`（更新时间/来源）+ `template: "blue"`（不加 icon）
- `body`：顶部总述 markdown → 多个 `collapsible_panel` 折叠面板，`"expanded": false` 默认收起
- `collapsible_panel.header` 结构：`{"title": {"tag": "plain_text", "content": "板块标题"}, "background_color": "颜色-50"}`（不是直接字符串）
- 不同分类板块使用不同浅色背景区分：`blue-50` / `green-50` / `orange-50` / `grey-50` 循环交替
- 折叠面板内部使用 `markdown` 组件，`lark_md` 标准语法
- JSON 太长时写入临时文件：`lark-cli im +messages-send --user-id ... --msg-type interactive --content "$(cat /tmp/card.json)" --as bot`

### 常见坑
- `header.title` 是对象不是字符串 → `{"tag":"plain_text","content":"标题"}`
- `message` 工具发卡片 → 飞书私聊 JSON 原样显示 → 必须用 lark-cli
- 单卡片表格过多 → 飞书 11310/230099 错误 → 拆分多张卡片

### 延伸技巧
- 卡片内最后一个面板放选购建议 & 延伸话题，`expanded: false` 默认收起，替代文字追问
- Emoji 开头板块标题（🔥📋💡🎣）让卡片更生动
```

### 方式二：作为独立 Skill 文件引用

将本 skill 放到 `skills/` 目录，在 SOUL.md 中引用：

```markdown
## 飞书卡片输出

见 skills/feishu-interactive-card/SKILL.md
```

Agent 框架会自动加载并遵循其中规则。

---

## 适合卡片的场景

- 多板块内容（产品介绍、参数对比、选购建议等）
- 版本更新说明
- 产品线梳理
- 需要折叠面板组织的大量信息

**不适合卡片**：简单问答、闲聊、单轮问候 → 纯文本即可。

## 完整 JSON 模板

```json
{
  "schema": "2.0",
  "config": { "width_mode": "default" },
  "header": {
    "title": {"tag": "plain_text", "content": "标题（≤20字）"},
    "subtitle": {"tag": "plain_text", "content": "副标题/更新时间"},
    "template": "blue"
  },
  "body": {
    "direction": "vertical",
    "padding": "12px 12px 20px 12px",
    "elements": [
      {"tag": "markdown", "content": "总述文字（lark_md格式）"},
      {
        "tag": "collapsible_panel",
        "expanded": false,
        "header": {
          "title": {"tag": "plain_text", "content": "🔥 板块一"},
          "background_color": "blue-50"
        },
        "elements": [
          {"tag": "markdown", "content": "板块正文（lark_md格式）"}
        ]
      },
      {
        "tag": "collapsible_panel",
        "expanded": false,
        "header": {
          "title": {"tag": "plain_text", "content": "📋 板块二"},
          "background_color": "green-50"
        },
        "elements": [
          {"tag": "markdown", "content": "板块正文（lark_md格式）"}
        ]
      },
      {
        "tag": "collapsible_panel",
        "expanded": false,
        "header": {
          "title": {"tag": "plain_text", "content": "💡 选购建议 & 延伸"},
          "background_color": "grey-50"
        },
        "elements": [
          {"tag": "markdown", "content": "选购建议 + 延伸话题推荐"}
        ]
      }
    ]
  }
}
```

## 颜色轮换参考

| 板块分类（示例） | 背景色 |
|-----------------|--------|
| 水滴轮 / 核心功能 | `blue-50` |
| 纺车轮 / 参数规格 | `green-50` |
| 鱼竿 / 使用建议 | `orange-50` |
| 选购建议 / 延伸推荐 | `grey-50` |

具体场景自定即可，核心是**不同板块不同色**，视觉层次清晰。

## 发送流程

1. 组装 card JSON
2. 用 `--dry-run` 校验（推荐）
3. 通过 `lark-cli` 发送
4. 发送成功后简短确认或 `NO_REPLY`

**失败重试**：最多 3 次，超过降级为纯文本。

## 常见坑速查

| 问题 | 原因 | 解决 |
|------|------|------|
| JSON 原样显示 | 用了 `message` 工具 | 必须用 `lark-cli` |
| 卡片渲染空白 | `header.title` 写成字符串 | `{"tag":"plain_text","content":"标题"}` |
| 面板默认展开 | `expanded` 没设为 `false` | 所有面板设 `"expanded": false` |
| 超长 JSON 被截断 | shell 参数长度限制 | 写入文件，用 `$(cat tmp.json)` 传参 |
| 表格过多 11310/230099 | 单卡片表格数超限 | 拆多张卡片或多条消息 |

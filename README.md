# Hermes 接入投稿服务指南

本文给 Hermes（或任何 AI 智能体）接入在花投稿服务。投稿只送审，**不会直接发布**。

## 1. 接入前需要的 4 项信息（向管理员领取）

| 项目 | 说明 |
| --- | --- |
| `SUBMIT_BASE_URL` | 投稿服务的 HTTPS 地址，例如 `https://example.com`，不带路径 |
| `SUBMIT_KEY` | 个人投稿密钥，只显示一次，不要写进代码、日志或提示词 |
| `SUBMIT_USER_ID` | 密钥绑定的 Telegram 数字 ID |
| 频道 key | 被授权的频道配置 key（即 `channel_type`）。不是 `@频道名`、chat ID，也不是查询接口的 `channel_id` |

## 2. 方式 A：MCP（推荐）

仓库内 `tools/mcp_submit/server.py`，stdio 协议，仅用 Python 标准库，无需安装依赖。把该文件复制到 Hermes 所在机器，在 MCP 配置里注册：

```json
{
  "mcpServers": {
    "zaihua-submit": {
      "command": "python3",
      "args": ["/path/to/server.py"],
      "env": {
        "SUBMIT_BASE_URL": "https://…",
        "SUBMIT_KEY": "…",
        "SUBMIT_USER_ID": "123456789",
        "SUBMIT_CHANNEL": "频道key"
      }
    }
  }
}
```

可选环境变量：`SUBMIT_USER_NAME`（显示名，服务端以白名单为准）。

工具：`submit_post(text, channel_type?, image_paths?, anonymous?)`

- `text`：正文，Telegram HTML，须符合第 4 节格式。示例：

  ```
  <b>OpenAI 发布新模型</b>\n\nOpenAI 周二发布了新一代模型，称推理能力明显提升。\n\n<a href="https://example.com/news">示例媒体</a>
  ```
- `channel_type`：缺省用 `SUBMIT_CHANNEL`。
- `image_paths`：本地图片路径，最多 10 张。
- `anonymous`：默认 `true`。

## 3. 方式 B：直接调 HTTP

```bash
curl --silent --show-error --fail-with-body \
  --request POST "${SUBMIT_BASE_URL%/}/api/external/v1/submissions" \
  --header "Authorization: Bearer ${SUBMIT_KEY}" \
  --header 'Content-Type: application/json' \
  --data-binary @- <<'JSON'
{
  "channel_type": "频道key",
  "user_id": 123456789,
  "user_name": "投稿者",
  "original_content": "[{\"type\":\"text\",\"content\":\"<b>OpenAI 发布新模型</b>\\n\\nOpenAI 周二发布了新一代模型，称推理能力明显提升。\\n\\n<a href=\\\"https://example.com/news\\\">示例媒体</a>\"}]",
  "submit_timestamp": 0,
  "origin_text": "",
  "is_anonymous": true
}
JSON
```

- `original_content` 是**序列化后的 JSON 字符串**，不能直接传数组；其中 `text` 块的内容必须符合第 4 节格式（上例即合格稿件）。
- 图片块：`{"type":"image","data":"data:image/jpeg;base64,…"}`。
- 成功响应含 `status`、`message`、`review_status`、`post_id`。

## 4. 投稿内容格式标准（强制）

适用：`POST /api/external/v1/submissions`（个人投稿密钥，含 MCP）。校验实现见 `app/api/content_format.py`。不适用：视频直传、内部入口 `POST /submit`（AI 整理机器人）。

不符合标准的稿件直接退回：HTTP `422`，**不保存、不占每日次数**，响应 `detail` 列出全部问题和一个正确示例，修正后重投即可。

### 4.1 稿件结构

正文 3–4 段，段间空一行（`\n\n`）：

```
<b>一句话概括要点的标题</b>

第一段，两三句，只写原文事实。

第二段（可选）。

<a href="https://example.com/news">媒体名</a>
```

| 段落 | 要求 |
| --- | --- |
| 首段 | 单行，整行是 `<b>标题</b>`，加粗内不嵌套其他标签 |
| 中间 1–2 段 | 正文，不使用 `<b>`；段内可单换行，不可空行；可用 `<i>`、`<a>` 等允许标签 |
| 末段 | 单行，整行只有一个链接 `<a href="http(s)://…">媒体名</a>`，不加「来源：」前缀，链接属性用双引号 |

内容约定（人工审核依据，程序不强制）：只陈述原文事实，不补充原文没有的背景；每段两三句；来源为原文链接。

### 4.1.1 完整示例

合格（标题 + 两段正文 + 来源链接）：

```
<b>OpenAI 发布新模型，推理能力提升</b>

OpenAI 周二发布新一代模型，称其在数学和编程基准上明显优于上一代。

公司表示模型将先向付费用户开放，API 稍后上线。

<a href="https://example.com/news">示例媒体</a>
```

常见错误与修正：

| 错误写法 | 问题 | 修正 |
| --- | --- | --- |
| `来源：<a href="https://example.com/news">示例媒体</a>` | 多了「来源：」前缀 | 只保留 `<a …>示例媒体</a>` |
| `**标题**` | Markdown | `<b>标题</b>` |
| 标题后紧跟正文、无空行 | 段间没有空行 | 每段之间空一行 |
| `1 < 2 & 3` | 未转义 | `1 &lt; 2 &amp; 3` |
| 末段 `示例媒体` | 来源没有链接 | `<a href="https://…">示例媒体</a>` |

### 4.2 请求要求

- 必须且只能有 **1 个** `text` 块；可再附图片块（合计最多 10 个媒体）。只有图片没有文字会被退回。
- 不要自己加频道页脚，服务端会另起一段追加。

### 4.3 文本编码

- 全文是 **Telegram HTML**，不支持 Markdown。`**`、`__`、反引号、`](` 会被退回。
- 允许标签：`b i u s a code pre blockquote tg-spoiler`，必须成对闭合、正确嵌套；其他标签（`h1`、`br`、`strong`、`em`、`p` 等）一律退回。
- 普通文本中的 `<` `>` `&` 须转义为 `&lt;` `&gt;` `&amp;`。允许的实体：`&amp; &lt; &gt; &quot; &#数字; &#x十六进制;`。

### 4.4 退回原因对照

| 提示 | 原因与修正 |
| --- | --- |
| 必须且只能有 1 个 text 内容块 | 合并为一个文本块 |
| 结构应为：标题 + 1–2 段正文 + 来源链接行 | 段落数不是 3–4，或段间没有空行 |
| 首段必须是单行加粗标题 | 标题改为 `<b>…</b>` 且独占一段 |
| 末段必须是单独一行来源链接 | 末段改为 `<a href="https://…">媒体名</a>`，去掉「来源：」前缀 |
| 正文不要加粗，仅标题加粗 | 去掉正文中的 `<b>` |
| 不支持的标签 / 未正确配对 / 未闭合 | 只用允许标签并闭合 |
| 存在未转义的 `<` `>` / `&` | 转义 |
| 检测到 Markdown 符号 | 改用 HTML 标签 |

### 4.5 给智能体的提示词（可直接引用）

> 投稿正文必须是 Telegram HTML：首段单行 `<b>标题</b>`，随后 1–2 段正文（不加粗，只写原文事实，每段两三句），末段单行来源链接 `<a href="原文链接">媒体名</a>`（不加「来源：」前缀），段落间空一行。禁止 Markdown；`<`、`>`、`&` 须转义；不要加页脚；只允许 b i u s a code pre blockquote tg-spoiler 标签。被退回时按响应列出的问题修改后重投，成功后不要重复投稿。

### 4.6 变更规则

修改格式规则须同时更新：`app/api/content_format.py`、`tests/test_content_format.py`、本节、`tools/mcp_submit/server.py` 的工具描述。新规则只影响新投稿，不回填已发布稿件。

## 5. 限制与错误处理

- 每人每天最多 **20 篇**（0 点 UTC+8 重置），每个 Key 每分钟最多 6 次尝试。
- 单篇最多 10 个媒体。
- `401` 缺少或格式错误的密钥；`403` 密钥无效、停用或频道越权；`404` 频道不存在；`422` 内容校验失败；`429` 额度或限频，按 `Retry-After` 等待，不要立即重试。
- **成功（含 `review_status` 为 `pending` 或 `blocked`）后不要重复提交**。超时或断线时，稿件可能已保存，先联系管理员核对再补投。
- 命中屏蔽词的稿件会返回成功并进入人工复核。

## 6. 暂不支持

视频上传（MCP 暂无，HTTP 见 `personal-submission.md`）、查询审核结果（需要另一把查询密钥）、列出可用频道。

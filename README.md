# 给 AI 的指令：安装投稿 MCP 并学会投稿

> **用法**：把本文整篇发给你的 AI（任何支持 MCP 的 AI 智能体），再补一句「按这个做，需要的信息我稍后给你」。AI 会自己安装、配置、验证。
> 你只需要向管理员领取 4 项信息，并在 AI 问你时提供。

---

以下内容是写给 AI 的。

## 你的任务

为用户安装一个「投稿」MCP 工具 `submit_post`，并在今后用户说「投稿」时用它向频道投稿。投稿只送审，不会直接发布。**用户不接触 HTML：你只提供纯文本字段，工具自动生成格式；投稿前必须先给用户看预览，用户确认后才提交。**

## 第 1 步：向用户索要 4 项信息（缺哪项问哪项）

| 变量 | 含义 |
| --- | --- |
| `SUBMIT_BASE_URL` | 投稿服务 HTTPS 地址，不带路径 |
| `SUBMIT_KEY` | 个人投稿密钥（保密） |
| `SUBMIT_USER_ID` | 密钥绑定的 Telegram 数字 ID |
| `SUBMIT_CHANNEL` | 频道配置 key（不是 `@频道名`，也不是 chat ID） |

密钥只能写进 MCP 配置的 `env`：不要回显、不要写进日志、提示词、笔记或任何仓库。

## 第 2 步：保存脚本

确认机器上有 `python3`（仅用标准库，无需安装依赖）。把下面整段代码原样保存为一个固定位置的文件（如 `~/mcp/zaihua-submit/server.py`），记下绝对路径：

````python
#!/usr/bin/env python3
"""zaihuabot 个人投稿 MCP 服务（stdio，仅标准库）。

环境变量：
  SUBMIT_BASE_URL  投稿服务 HTTPS 地址（必填）
  SUBMIT_KEY       个人投稿密钥（必填，不会出现在工具参数或日志中）
  SUBMIT_USER_ID   密钥绑定的 Telegram 数字 ID（必填）
  SUBMIT_USER_NAME 显示名（可选，服务端以白名单为准）
  SUBMIT_CHANNEL   默认频道 key（可选）
"""
from __future__ import annotations

import base64
import html
import json
import mimetypes
import os
import re
import sys
import urllib.error
import urllib.request

PROTOCOL = "2024-11-05"

FIELDS = {
    "title": {"type": "string", "description": "标题，一句话概括要点（纯文本，单行）"},
    "paragraphs": {
        "type": "array",
        "items": {"type": "string"},
        "minItems": 1,
        "maxItems": 2,
        "description": "正文 1–2 段（纯文本，不加粗，段内不要空行）",
    },
    "source_name": {"type": "string", "description": "来源媒体名（纯文本）"},
    "source_url": {"type": "string", "description": "原文链接，http(s)://"},
}
REQUIRED = ["title", "paragraphs", "source_name", "source_url"]

TOOLS = [
    {
        "name": "preview_post",
        "description": (
            "生成投稿预览，不会提交、不联网。只需给出纯文本字段：标题、1–2 段正文、来源名称和链接；"
            "HTML 由工具生成并转义。返回渲染后的预览，请原样展示给用户并询问是否确认投稿。"
            "字段不合规会直接报错并说明原因。"
        ),
        "inputSchema": {"type": "object", "properties": FIELDS, "required": REQUIRED},
    },
    {
        "name": "submit_post",
        "description": (
            "向频道投稿（只送审，不会直接发布，投稿后无法撤回）。字段同 preview_post。"
            "必须先用 preview_post 向用户展示预览并得到明确确认，再带 confirmed=true 调用。"
            "正文只写原文事实，每段两三句。成功后不要重复投稿。"
        ),
        "inputSchema": {
            "type": "object",
            "properties": {
                **FIELDS,
                "confirmed": {
                    "type": "boolean",
                    "description": "用户已看过预览并明确确认投稿时才填 true",
                },
                "channel_type": {
                    "type": "string",
                    "description": "频道配置 key；缺省用 SUBMIT_CHANNEL",
                },
                "image_paths": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "本地图片路径，最多 10 张",
                },
                "anonymous": {"type": "boolean", "description": "是否匿名，默认 true"},
            },
            "required": REQUIRED + ["confirmed"],
        },
    },
]


def _env(name: str) -> str:
    value = os.environ.get(name, "").strip()
    if not value:
        raise RuntimeError(f"缺少环境变量 {name}")
    return value


_MD_MARKS = ("**", "__", "`", "](")


def _fields(args: dict) -> tuple[str, list[str], str, str]:
    def clean(name: str) -> str:
        v = args.get(name)
        if not isinstance(v, str) or not v.strip():
            raise ValueError(f"{name} 不能为空")
        v = v.strip()
        if "\n" in v or "\r" in v:
            raise ValueError(f"{name} 必须是单行")
        return v

    title, name, url = clean("title"), clean("source_name"), clean("source_url")
    paras = args.get("paragraphs")
    if (not isinstance(paras, list) or not 1 <= len(paras) <= 2
            or not all(isinstance(x, str) and x.strip() for x in paras)):
        raise ValueError("paragraphs 须为 1–2 段非空文本")
    paras = [x.strip() for x in paras]
    if any("\n\n" in x.replace("\r", "") or re.search(r"\n\s*\n", x) for x in paras):
        raise ValueError("每段内不能有空行，请拆成 paragraphs 的不同元素（最多 2 段）")
    if not re.match(r"https?://[^\s\"<>]+$", url):
        raise ValueError("source_url 必须是 http(s):// 开头且不含空格的链接")
    for label, text in [("title", title), ("source_name", name), *[("paragraphs", x) for x in paras]]:
        for mark in _MD_MARKS:
            if mark in text:
                raise ValueError(f"{label} 含 Markdown 符号 {mark!r}，请只写纯文本")
    return title, paras, name, url


def compose_text(args: dict) -> str:
    title, paras, name, url = _fields(args)
    esc = lambda t: html.escape(t, quote=False)  # noqa: E731
    parts = [f"<b>{esc(title)}</b>"] + [esc(x) for x in paras]
    parts.append(f'<a href="{html.escape(url, quote=True)}">{esc(name)}</a>')
    return "\n\n".join(parts)


def preview_post(args: dict) -> str:
    title, paras, name, url = _fields(args)
    compose_text(args)
    body = "\n\n".join([f"**{title}**", *paras, f"[{name}]({url})"])
    return (
        "字段检查通过。以下是预览（原样展示给用户，不要改写），"
        "请询问「确认投稿吗？」，用户明确确认后再调用 submit_post（confirmed=true）：\n\n"
        "-----\n" + body + "\n-----"
    )


def build_payload(args: dict) -> dict:
    text = compose_text(args)
    channel = (args.get("channel_type") or os.environ.get("SUBMIT_CHANNEL", "")).strip()
    if not channel:
        raise ValueError("需要 channel_type（或设置 SUBMIT_CHANNEL）")
    images = args.get("image_paths") or []
    if not isinstance(images, list):
        raise ValueError("image_paths 须为路径数组")
    if len(images) > 10:
        raise ValueError("单篇最多 10 张图片")
    items = [{"type": "text", "content": text}]
    for path in images:
        if not os.path.isfile(path):
            raise ValueError(f"图片文件不存在：{path}")
        mime = mimetypes.guess_type(path)[0] or ""
        if not mime.startswith("image/"):
            raise ValueError(f"不是图片文件：{path}")
        with open(path, "rb") as fh:
            data = base64.b64encode(fh.read()).decode()
        items.append({"type": "image", "data": f"data:{mime};base64,{data}"})
    return {
        "channel_type": channel,
        "user_id": int(_env("SUBMIT_USER_ID")),
        "user_name": os.environ.get("SUBMIT_USER_NAME", "投稿者"),
        "original_content": json.dumps(items, ensure_ascii=False),
        "submit_timestamp": 0,
        "origin_text": "",
        "is_anonymous": args.get("anonymous", True) is not False,
    }


def _summarize(raw: str) -> str:
    try:
        d = json.loads(raw)
    except ValueError:
        return raw
    if not isinstance(d, dict):
        return raw
    review = {"pending": "待人工审核", "blocked": "命中屏蔽词，进入人工复核"}.get(
        d.get("review_status"), d.get("review_status"))
    return (f"投稿成功（{d.get('status')}）：已送审，不要重复投稿。"
            f"\npost_id: {d.get('post_id')}\n审核状态: {review}\n{d.get('message') or ''}").strip()


def submit_post(args: dict) -> str:
    if args.get("confirmed") is not True:
        raise ValueError("未确认：请先用 preview_post 向用户展示预览，用户明确确认后再带 confirmed=true 调用")
    payload = build_payload(args)
    base = _env("SUBMIT_BASE_URL").rstrip("/")
    req = urllib.request.Request(
        base + "/api/external/v1/submissions",
        data=json.dumps(payload).encode(),
        method="POST",
        headers={
            "Authorization": "Bearer " + _env("SUBMIT_KEY"),
            "Content-Type": "application/json",
        },
    )
    try:
        with urllib.request.urlopen(req, timeout=60) as resp:
            return _summarize(resp.read().decode())
    except urllib.error.HTTPError as exc:
        retry = exc.headers.get("Retry-After")
        body = exc.read().decode(errors="replace")
        extra = f"（Retry-After: {retry}s）" if retry else ""
        raise RuntimeError(f"HTTP {exc.code}{extra}: {body}") from None
    except OSError as exc:
        raise RuntimeError(
            f"网络错误，稿件可能已保存，请先联系管理员核对再决定是否补投：{exc}"
        ) from None


def handle(msg: dict) -> dict | None:
    if not isinstance(msg, dict):
        return {"jsonrpc": "2.0", "id": None,
                "error": {"code": -32600, "message": "Invalid Request"}}
    method, mid = msg.get("method"), msg.get("id")
    if mid is None:  # notification
        return None
    if method == "initialize":
        result = {
            "protocolVersion": (msg.get("params") or {}).get("protocolVersion") or PROTOCOL,
            "capabilities": {"tools": {}},
            "serverInfo": {"name": "zaihuabot-submit", "version": "0.2.0"},
        }
    elif method == "tools/list":
        result = {"tools": TOOLS}
    elif method == "tools/call":
        params = msg.get("params") or {}
        try:
            fn = {"preview_post": preview_post, "submit_post": submit_post}.get(params.get("name"))
            if fn is None:
                raise ValueError("未知工具")
            out, err = fn(params.get("arguments") or {}), False
        except Exception as exc:  # report as tool error, not protocol error
            out, err = str(exc), True
        result = {"content": [{"type": "text", "text": out}], "isError": err}
    elif method == "ping":
        result = {}
    else:
        return {"jsonrpc": "2.0", "id": mid,
                "error": {"code": -32601, "message": "Method not found"}}
    return {"jsonrpc": "2.0", "id": mid, "result": result}


def main() -> None:
    for line in sys.stdin:
        line = line.strip()
        if not line:
            continue
        try:
            reply = handle(json.loads(line))
        except json.JSONDecodeError:
            reply = {"jsonrpc": "2.0", "id": None,
                     "error": {"code": -32700, "message": "Parse error"}}
        if reply is not None:
            sys.stdout.write(json.dumps(reply, ensure_ascii=False) + "\n")
            sys.stdout.flush()


if __name__ == "__main__":
    main()
````

## 第 3 步：注册 MCP

按你自身的 MCP 配置方式，注册一个 stdio 服务，名称 `zaihua-submit`：

```json
{
  "mcpServers": {
    "zaihua-submit": {
      "command": "python3",
      "args": ["<server.py 的绝对路径>"],
      "env": {
        "SUBMIT_BASE_URL": "<第1步>",
        "SUBMIT_KEY": "<第1步>",
        "SUBMIT_USER_ID": "<第1步>",
        "SUBMIT_CHANNEL": "<第1步>"
      }
    }
  }
}
```

可选 `SUBMIT_USER_NAME`（显示名）。若你的配置格式不同，按你的格式等价写入。然后重载或重启，使配置生效。

## 第 4 步：验证（不要用真实稿件试）

1. 确认工具列表里出现 `preview_post` 和 `submit_post`。没有 → 检查配置 JSON、脚本路径、`python3`，修好再试。
2. 告诉用户「已装好」。不要为了测试发起真实投稿。

## 第 5 步：投稿流程（长期有效）

用户说「投稿」时：读懂用户回复的那条消息或给出的链接，按下面流程做。

**1. 整理成字段（纯文本，不写任何 HTML 或 Markdown）：**

- `title`：标题，一句话概括要点。
- `paragraphs`：正文 1–2 段，每段两三句，只陈述原文事实，不补充原文没有的背景，不加粗。
- `source_name`：来源媒体名。
- `source_url`：原文链接（http/https）。
- 外语原文：标题和正文用简体中文表述，`source_url` 保留原文地址。
- 可选：`channel_type`（缺省用 `SUBMIT_CHANNEL`）、`image_paths`（本地图片路径，最多 10 张）、`anonymous`（默认 `true`）。

**2. 预览与确认。** 用渲染后的样子展示，不要展示 HTML 标签：

> **标题**
>
> 第一段……
>
> 第二段……
>
> [来源媒体名](原文链接)

并问「确认投稿吗？」。用户要改就改完再预览；用户确认后才调用 `submit_post`。投稿后无法撤回，不得未经确认直接提交。

**3. 先调用 `preview_post`**（不联网、不提交），传入上面的字段。它会检查字段并返回预览，把预览原样展示给用户并问「确认投稿吗？」。字段不合规会报错，按原因改完再试。

**4. 用户明确确认后，调用 `submit_post`**，传入同样的字段并带 `confirmed=true`。未确认时工具会拒绝提交。工具会自动生成 HTML 并转义特殊字符。

## 第 6 步：处理结果

| 返回 | 你该怎么做 |
| --- | --- |
| 成功（`status=success`） | 告诉用户已送审及 `post_id`。**不要重复投稿。** |
| `422` | 格式不合格，不保存、不扣次数。按返回的问题修改字段后重新预览、确认、再投 |
| `401` / `403` / `404` | 密钥、频道 key 有问题。告诉用户检查对应项，不要反复重试 |
| `429` | 超额（每人每天 20 篇）或过频（每密钥每分钟 6 次）。按 `Retry-After` 等待，不要立即重试 |
| 超时/断线 | 稿件可能已保存。先请用户联系管理员核对，不要自动补投 |

视频投稿暂不支持。

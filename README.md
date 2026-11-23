# 给 AI 的指令：安装投稿 MCP 并学会投稿

> **用法**：把本文整篇发给你的 AI（Hermes 等），再补一句「按这个做，需要的信息我稍后给你」。AI 会自己安装、配置、验证。
> 你只需要向管理员领取 4 项信息，并在 AI 问你时提供。

---

以下内容是写给 AI 的。

## 你的任务

为用户安装一个「投稿」MCP 工具 `submit_post`，并在今后用户说「投稿」时用它向频道投稿。投稿只送审，不会直接发布。

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
import json
import mimetypes
import os
import sys
import urllib.error
import urllib.request

PROTOCOL = "2024-11-05"

TOOLS = [
    {
        "name": "submit_post",
        "description": (
            "向频道投稿（只送审，不会直接发布）。text 必须严格符合格式，否则 422 退回（不扣次数）："
            "首段单行 <b>标题</b>，空一行，1–2 段正文（不加粗，只写原文事实），空一行，"
            "末段单行来源链接 <a href=\"原文链接\">媒体名</a>（不加“来源：”前缀），共 3–4 段。text 按 Telegram HTML 解析："
            "可用 <b>、<i>、<a href=\"https://…\">名字</a>；不支持 Markdown，"
            "** 与 [名](链接) 会原样显示；普通文本里的 < > & 须先转义为 &lt; &gt; &amp;。"
            "频道页脚由服务端另起一段追加，无需自己加。成功后不要重复投稿。"
        ),
        "inputSchema": {
            "type": "object",
            "properties": {
                "text": {"type": "string", "description": "正文（Telegram HTML）"},
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
            "required": ["text"],
        },
    }
]


def _env(name: str) -> str:
    value = os.environ.get(name, "").strip()
    if not value:
        raise RuntimeError(f"缺少环境变量 {name}")
    return value


def build_payload(args: dict) -> dict:
    text = args.get("text")
    if not isinstance(text, str) or not text.strip():
        raise ValueError("text 不能为空")
    channel = (args.get("channel_type") or os.environ.get("SUBMIT_CHANNEL", "")).strip()
    if not channel:
        raise ValueError("需要 channel_type（或设置 SUBMIT_CHANNEL）")
    images = args.get("image_paths") or []
    if len(images) > 10:
        raise ValueError("单篇最多 10 张图片")
    items = [{"type": "text", "content": text}]
    for path in images:
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


def submit_post(args: dict) -> str:
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
            return resp.read().decode()
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
    method, mid = msg.get("method"), msg.get("id")
    if mid is None:  # notification
        return None
    if method == "initialize":
        result = {
            "protocolVersion": PROTOCOL,
            "capabilities": {"tools": {}},
            "serverInfo": {"name": "zaihuabot-submit", "version": "0.1.0"},
        }
    elif method == "tools/list":
        result = {"tools": TOOLS}
    elif method == "tools/call":
        params = msg.get("params") or {}
        try:
            if params.get("name") != "submit_post":
                raise ValueError("未知工具")
            out, err = submit_post(params.get("arguments") or {}), False
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

1. 确认工具列表里出现 `submit_post`。没有 → 检查配置 JSON、脚本路径、`python3`，修好再试。
2. 直接运行 `python3 <server.py路径>` 并向 stdin 发送 `{"jsonrpc":"2.0","id":1,"method":"tools/list"}` 一行，应返回含 `submit_post` 的结果。
3. 告诉用户「已装好」。不要为了测试发起真实投稿。

## 第 5 步：记住投稿规则（长期有效）

用户说「投稿」时：把用户回复的那条消息，或用户给出的链接，读懂后整理成稿件，调用 `submit_post`。

稿件必须是 Telegram HTML，3–4 段，段间空一行：

```
<b>一句话概括要点的标题</b>

第一段，两三句，只写原文事实。

第二段（可选）。

<a href="原文链接">媒体名</a>
```

- 首段：单行 `<b>标题</b>`。
- 中间 1–2 段：正文，**不加粗**，每段两三句，只陈述原文事实，不补充原文没有的背景。
- 末段：单行来源链接，**不加「来源：」前缀**。
- 禁止 Markdown（`**`、`__`、反引号、`[名](链接)`）。
- 普通文本里的 `<` `>` `&` 须转义为 `&lt;` `&gt;` `&amp;`。
- 只允许标签：`b i u s a code pre blockquote tg-spoiler`。
- 不要自己加频道页脚，服务端会追加。
- 外语原文：正文用简体中文表述，来源链接保留原文地址。
- 参数：`text` 必填；`channel_type` 缺省用 `SUBMIT_CHANNEL`；`image_paths` 本地图片路径，最多 10 张；`anonymous` 默认 `true`。

## 第 6 步：处理结果

| 返回 | 你该怎么做 |
| --- | --- |
| 成功（`status=success`） | 告诉用户已送审及 `post_id`。**不要重复投稿。** |
| `422` | 格式不合格，不保存、不扣次数。按返回的问题列表修改后重投 |
| `401` / `403` / `404` | 密钥、频道 key 有问题。告诉用户检查对应项，不要反复重试 |
| `429` | 超额（每人每天 20 篇）或过频（每密钥每分钟 6 次）。按 `Retry-After` 等待，不要立即重试 |
| 超时/断线 | 稿件可能已保存。先请用户联系管理员核对，不要自动补投 |

视频投稿暂不支持。

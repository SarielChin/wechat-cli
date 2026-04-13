---
name: wechat-cli
version: 1.0.0
description: |
  微信聊天记录查询工具：查看最近会话、读取聊天记录、全局搜索消息、查询联系人和群成员、统计聊天数据、导出聊天记录、查看收藏、检查未读消息和新消息增量。当用户需要查微信聊天记录、搜索微信消息、查看微信联系人、导出微信对话、统计聊天活跃度、查看未读消息时使用。即使用户只是说"帮我找一下微信里xxx说的那条消息"、"看看群里最近聊了什么"、"导出和某人的聊天记录"，也应该触发。
metadata:
  requires:
    bins: ["wechat-cli"]
  cliHelp: "wechat-cli --help"
---

# wechat-cli 命令参考

wechat-cli 是一个**只读**的本地工具，从本机微信的加密 SQLite 数据库解密查询数据，不会联网、不会修改任何数据。

## 安全与执行规则（必须遵守）

- 默认仅执行只读查询命令（`sessions/history/search/contacts/members/stats/favorites/unread/new-messages`）。
- 任何涉及 `init` / `init --force` / `sudo` / `codesign` 的操作，必须先明确告知风险并获得用户确认后再执行。
- 严禁输出或分享 `~/.wechat-cli/all_keys.json` 内容；返回结果需最小化披露，避免泄露密钥、wxid、手机号、文件绝对路径等敏感信息。
- 若用户请求导出或大批量检索，优先缩小时间范围与数量（`--start-time/--end-time/--limit`）。

## 初始化

首次使用前必须运行 `init`，从微信进程内存中提取数据库加密密钥（微信必须处于**登录状态且正在运行**）：

```bash
sudo wechat-cli init              # macOS/Linux 通常需要 sudo
wechat-cli init                   # Windows（管理员终端）
wechat-cli init --db-dir <path>   # 手动指定数据库目录
wechat-cli init --force           # 强制重新提取密钥
```

初始化后产生 `~/.wechat-cli/config.json`（数据库路径）和 `~/.wechat-cli/all_keys.json`（加密密钥，**不要输出或分享**）。

## 全局参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--config <path>` | 自动 | 配置文件路径，也可通过 `WECHAT_CLI_CONFIG` 环境变量指定 |

说明：`--format` 不是全局参数，而是子命令参数。多数查询命令支持 `json|text`，`export` 仅支持 `markdown|txt`。

## 命令速查表

| 命令 | 用途 | 典型场景 |
|------|------|---------|
| `init` | 初始化密钥 | "首次配置 wechat-cli" |
| `sessions` | 最近会话列表 | "看看最近和谁聊过" |
| `history` | 聊天记录 | "看看和Alice的聊天记录" |
| `search` | 搜索消息 | "搜一下谁发过这个链接" |
| `contacts` | 联系人查询 | "查一下某人的微信号" |
| `members` | 群成员列表 | "看看群里有谁" |
| `stats` | 聊天统计 | "这个群谁最活跃" |
| `export` | 导出记录 | "导出这段对话" |
| `favorites` | 收藏内容 | "看看我收藏了什么文章" |
| `unread` | 未读消息 | "有没有未读消息" |
| `new-messages` | 增量新消息 | "上次看过之后有什么新消息" |

---

## sessions — 最近会话

列出最近的聊天会话，按最后消息时间倒序排列。

```bash
wechat-cli sessions
wechat-cli sessions --limit 10
wechat-cli sessions --format text
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--limit <n>` | `20` | 返回数量上限 |
| `--format json\|text` | `json` | 输出格式 |

**JSON 输出字段**：`chat`（显示名）、`username`（wxid）、`is_group`、`unread`、`last_message`、`msg_type`、`sender`（群聊时）、`timestamp`、`time`

---

## history — 聊天记录

获取指定聊天的消息历史，支持分页、时间范围和消息类型过滤。

```bash
wechat-cli history "Alice"
wechat-cli history "Alice" --limit 100 --offset 50
wechat-cli history "工作群" --start-time "2026-04-01" --end-time "2026-04-03"
wechat-cli history "Alice" --type link
wechat-cli history "Alice" --media    # 解析本地媒体文件路径
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `<chat_name>` | *必填* | 联系人名称、备注、昵称或 wxid |
| `--limit <n>` | `50` | 返回数量上限 |
| `--offset <n>` | `0` | 分页偏移量 |
| `--start-time <date>` | — | 起始时间（`YYYY-MM-DD` 或 `YYYY-MM-DD HH:MM:SS`） |
| `--end-time <date>` | — | 结束时间 |
| `--type <type>` | — | 消息类型过滤（见下方类型表） |
| `--media` | `false` | 解析并返回图片/视频/语音/文件的本地磁盘路径 |
| `--format json\|text` | `json` | 输出格式 |

---

## search — 搜索消息

全文关键词搜索，支持单聊、多聊或全局搜索。

```bash
wechat-cli search "会议纪要"
wechat-cli search "hello" --chat "Alice"
wechat-cli search "报告" --chat "团队A" --chat "团队B"   # 多聊天搜索
wechat-cli search "合同" --type file --start-time "2026-01-01"
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `<keyword>` | *必填* | 搜索关键词（SQL LIKE 模糊匹配） |
| `--chat <name>` | — | 限定聊天范围（可多次指定） |
| `--start-time <date>` | — | 起始时间 |
| `--end-time <date>` | — | 结束时间 |
| `--limit <n>` | `20` | 返回数量上限（硬上限 500） |
| `--offset <n>` | `0` | 分页偏移量 |
| `--type <type>` | — | 消息类型过滤 |
| `--format json\|text` | `json` | 输出格式 |

**搜索范围逻辑**：
- 不指定 `--chat` → 全局搜索所有消息
- 指定 1 个 `--chat` → 单聊搜索
- 指定多个 `--chat` → 多聊搜索，结果按时间合并排序

---

## contacts — 联系人查询

搜索联系人或查看某个联系人的详细信息。

```bash
wechat-cli contacts --query "张"
wechat-cli contacts --detail "Alice"
wechat-cli contacts --detail "wxid_xxx"
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--query <text>` | `""` | 搜索关键词（匹配昵称、备注、wxid，不区分大小写） |
| `--detail <name>` | — | 查看指定联系人的完整信息 |
| `--limit <n>` | `50` | 列表模式返回数量上限 |
| `--format json\|text` | `json` | 输出格式 |

**详情模式额外字段**：`alias`（微信号）、`description`（个性签名）、`avatar`（头像 URL）、`is_group`、`is_subscription`（公众号）

---

## members — 群成员

列出群聊的所有成员。

```bash
wechat-cli members "工作群"
wechat-cli members "xxx@chatroom"
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `<group_name>` | *必填* | 群名称、备注或 `@chatroom` ID |
| `--format json\|text` | `json` | 输出格式 |

**JSON 输出**：`group`、`username`（@chatroom ID）、`member_count`、`owner`、`members`（包含 `username`、`nick_name`、`remark`、`display_name`，群主排第一）

---

## stats — 聊天统计

聚合指定聊天的统计数据（消息总量、类型分布、活跃发送者、24小时活跃度）。

```bash
wechat-cli stats "工作群"
wechat-cli stats "Alice" --start-time "2026-04-01" --end-time "2026-04-07"
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `<chat_name>` | *必填* | 聊天名称或 wxid |
| `--start-time <date>` | — | 起始时间 |
| `--end-time <date>` | — | 结束时间 |
| `--format json\|text` | `json` | 输出格式 |

**JSON 输出**：`total`（总消息数）、`type_breakdown`（类型分布）、`top_senders`（前10活跃发送者）、`hourly`（24小时活跃度分布）

---

## export — 导出记录

将聊天记录导出为 Markdown 或纯文本。

```bash
wechat-cli export "Alice" --format markdown
wechat-cli export "Alice" --format txt --output chat.txt
wechat-cli export "工作群" --start-time "2026-04-01" --limit 1000
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `<chat_name>` | *必填* | 聊天名称或 wxid |
| `--format markdown\|txt` | `markdown` | 导出格式（注意：不支持 json） |
| `--output <path>` | stdout | 输出文件路径（不指定则输出到终端） |
| `--start-time <date>` | — | 起始时间 |
| `--end-time <date>` | — | 结束时间 |
| `--limit <n>` | `500` | 导出数量上限 |

---

## favorites — 收藏内容

列出微信收藏，支持按类型和关键词过滤。

```bash
wechat-cli favorites
wechat-cli favorites --type article
wechat-cli favorites --query "机器学习" --limit 5
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--limit <n>` | `20` | 返回数量上限 |
| `--type` | — | 类型过滤：`text`（文本）、`image`（图片）、`article`（文章）、`card`（名片）、`video`（视频号） |
| `--query <text>` | — | 关键词搜索 |
| `--format json\|text` | `json` | 输出格式 |

---

## unread — 未读消息

列出所有有未读消息的会话。

```bash
wechat-cli unread
wechat-cli unread --limit 10 --format text
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--limit <n>` | `50` | 返回数量上限 |
| `--format json\|text` | `json` | 输出格式 |

---

## new-messages — 增量新消息

有状态的增量轮询，返回上次调用以来的新消息。

```bash
wechat-cli new-messages
wechat-cli new-messages --format text
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--format json\|text` | `json` | 输出格式 |

**行为说明**：
- **首次调用**（无状态文件）：返回当前未读会话并保存时间戳，JSON 包含 `"first_call": true`
- **后续调用**：对比时间戳差异，返回有新消息的会话
- **重置**：删除 `~/.wechat-cli/last_check.json` 即可重置状态

---

## 消息类型过滤

`history` 和 `search` 命令的 `--type` 参数支持以下值：

| 值 | 说明 |
|----|------|
| `text` | 文本消息 |
| `image` | 图片 |
| `voice` | 语音 |
| `video` | 视频 |
| `sticker` | 表情包 |
| `location` | 位置分享 |
| `link` | 链接/小程序/文章 |
| `file` | 文件 |
| `call` | 音视频通话 |
| `system` | 系统消息（入群/退群等） |

## 联系人名称匹配规则

所有接受 `<chat_name>` 或 `--chat` 的命令都使用以下匹配优先级：
1. 精确匹配（备注 > 昵称 > wxid）
2. 不区分大小写的精确匹配
3. 不区分大小写的子串匹配

也可以直接使用 wxid（如 `wxid_xxx`）或群 ID（如 `xxx@chatroom`）。

## 常见问题

- **初始化失败** → 确认微信正在运行且已登录；macOS/Linux 用 `sudo`；Windows 用管理员终端
- **密钥过期** → 微信重新登录后需 `wechat-cli init --force`
- **联系人找不到** → 尝试备注名、昵称或 wxid；支持模糊匹配
- **数据过时** → 删除 `/tmp/wechat_cli_cache/` 强制刷新缓存

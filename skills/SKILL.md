---
name: wechat-cli
version: 1.2.0
description: |
  微信本地聊天数据查询工具。只要用户想查看这台机器上的微信聊天记录、搜索消息、查联系人或群成员、看最近会话、未读消息、收藏、聊天统计、导出对话，或者提到“微信里”“群里谁说过”“把这段聊天导出来”“最近谁联系我了”“这个群谁最活跃”，都应该优先使用这个技能；即使用户没有明确说 wechat-cli，也应触发。
metadata:
  requires:
    bins: ["wechat-cli"]
  cliHelp: "wechat-cli --help"
---

# wechat-cli 技能

wechat-cli 是一个**只读**的本地查询工具，用来读取这台机器上的微信数据库并返回结构化结果。它不会联网，也不会修改聊天数据。

## 何时使用

当用户想从**本地微信数据**里获取信息时，使用这个技能。典型触发方式：

- “帮我查微信里谁发过这句话”
- “看看和某人的聊天记录”
- “最近谁联系我了”
- “这个群里都有谁”
- “导出这段聊天”
- “找一下这个微信号/联系人”
- “看看未读消息”
- “上次看完之后又来了哪些新消息”
- “我收藏过那篇文章吗”
- “这个群谁最活跃”

如果用户只是泛泛提到“微信里”“群里”“聊天记录”“联系人”“未读”“收藏”“导出聊天”“最近消息”，也应该优先想到这个技能。

## 执行原则

1. 先判断用户要的是：最近会话、历史消息、全文搜索、联系人、群成员、统计、导出、收藏、未读，还是增量新消息。
2. 默认优先用**最小范围**查询：限制聊天对象、时间范围和 `--limit`，避免一次性拉太多数据。
3. 默认优先用 `--format json` 做结构化查询；只有用户明确要纯文本展示或导出文件时，再用 `text` 或 `export`。
4. 如果聊天对象不明确，先用 `contacts --query` 辅助确认名字，再继续查 `history`、`search`、`members`、`stats`、`export`。
5. 如果想找“谁说过某句话/某个关键词”，优先 `search`；如果已经知道聊天对象且要连续上下文，优先 `history`。
6. 输出时做最小披露：总结重点，不泄露密钥、完整 wxid、手机号、绝对路径等敏感信息，除非用户明确需要。

## 安全与确认规则

- 默认只执行只读查询命令：`sessions`、`history`、`search`、`contacts`、`members`、`stats`、`export`、`favorites`、`unread`、`new-messages`。
- `init` / `init --force` / `sudo` / `codesign` 都不是默认动作。只有用户明确要初始化或查询因未初始化而无法继续时，才先解释风险再征求确认。
- 严禁输出或分享 `~/.wechat-cli/all_keys.json` 内容。
- 若查询结果包含本地媒体路径、wxid、手机号、头像 URL、签名等信息，默认只返回解决当前问题所需的最少字段。
- 若用户要求导出或大批量检索，优先缩小 `--start-time`、`--end-time`、`--limit`，避免一次性暴露过多数据。

## 任务路由

| 用户意图 | 首选命令 | 备注 |
|---|---|---|
| 看最近和谁聊过 | `sessions` | 最近会话列表 |
| 看某人/某群最近消息 | `history` | 支持分页、时间范围、类型过滤 |
| 搜某句话/某个关键词 | `search` | 可全局，也可限制到一个或多个聊天 |
| 查联系人/微信号 | `contacts` | `--query` 搜索，`--detail` 看详情 |
| 看群成员 | `members` | 只适用于群聊 |
| 看谁最活跃/类型分布 | `stats` | 适合群聊和单聊统计 |
| 导出一段对话 | `export` | 输出 `markdown` 或 `txt` |
| 看微信收藏 | `favorites` | 支持类型和关键词过滤 |
| 看当前未读 | `unread` | 所有未读会话 |
| 看上次检查后新增的消息 | `new-messages` | 有状态增量查询 |

## 推荐工作流

### 1. 定位对象

- 联系人/群名不确定：先 `contacts --query`
- 用户直接给了 wxid 或 `@chatroom`：可直接用于后续命令

### 2. 选择查询方式

- 要“完整上下文”用 `history`
- 要“关键词命中”用 `search`
- 要“列表概览”用 `sessions` / `unread` / `favorites`
- 要“结构统计”用 `stats`
- 要“交付文件”用 `export`

### 3. 缩小范围

优先补这些参数：

- `--chat`
- `--start-time`
- `--end-time`
- `--limit`
- `--offset`
- `--type`

### 4. 返回结果

- 默认先给结论，再附少量关键证据
- 长结果先总结，再按需展开
- 用户若要原始输出，再返回完整 JSON / text / 导出文件

## 初始化

首次使用前通常需要运行 `init`，从微信进程内存中提取数据库加密密钥。只有在用户明确同意后，才执行这一步。微信必须处于**登录状态且正在运行**。

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

说明：`--format` 不是全局参数，而是子命令参数。多数查询命令支持 `json|text`（默认 `json`），`history` 另外支持 `jsonl`，`export` 仅支持 `markdown|txt`。

## 时间与排序

- `--start-time` / `--end-time` 支持 `YYYY-MM-DD`、`YYYY-MM-DD HH:MM`、`YYYY-MM-DD HH:MM:SS`。
- 只写日期的 `--end-time` 会自动算到当天 23:59:59，所以 `--start-time 2026-04-01 --end-time 2026-04-01` 就是 4 月 1 日整天。
- `history`、`search`、`export` 先取**最新**的符合条件的消息（`--offset` 从最新往前跳），再把这一页按**时间先后**排列，最旧的在前。所以“最新一条”在结果的**最后**。
- `sessions`、`unread` 按最后消息时间倒序，最新的在前。

## 命令速查表

| 命令 | 用途 | 典型场景 |
|------|------|---------|
| `init` | 初始化密钥 | “首次配置 wechat-cli” |
| `sessions` | 最近会话列表 | “看看最近和谁聊过” |
| `history` | 聊天记录 | “看看和 Alice 的聊天记录” |
| `search` | 搜索消息 | “搜一下谁发过这个链接” |
| `contacts` | 联系人查询 | “查一下某人的微信号” |
| `members` | 群成员列表 | “看看群里有谁” |
| `stats` | 聊天统计 | “这个群谁最活跃” |
| `export` | 导出记录 | “导出这段对话” |
| `favorites` | 收藏内容 | “看看我收藏了什么文章” |
| `unread` | 未读消息 | “有没有未读消息” |
| `new-messages` | 增量新消息 | “上次看过之后有什么新消息” |

## 自然语言到命令的常见映射

- “帮我找一下微信里谁说过‘预算冻结’” -> `search "预算冻结"`
- “看看 Alice 最近聊了什么” -> `history "Alice" --limit 50`
- “导出我和 Bob 昨天的聊天” -> `export "Bob" --start-time "YYYY-MM-DD" --end-time "YYYY-MM-DD" --format markdown`（只写日期的 `--end-time` 会涵盖当天整天）
- “看看产品群都有谁” -> `members "产品群"`
- “这个群谁最活跃” -> `stats "群名"`
- “我最近有谁没回” / “有没有未读” -> `unread`
- “上次看完之后又来了什么消息” -> `new-messages`

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

**JSON 输出**：数组，每项字段为 `chat`（显示名）、`username`（wxid）、`is_group`、`unread`、`last_message`、`msg_type`、`sender`（群聊时）、`timestamp`、`time`

适合先做概览，再决定是否继续 `history` 或 `search`。

## history — 聊天记录

获取指定聊天的消息历史，支持分页、时间范围和消息类型过滤。

```bash
wechat-cli history "Alice"
wechat-cli history "Alice" --limit 100 --offset 50
wechat-cli history "工作群" --start-time "2026-04-01" --end-time "2026-04-03"
wechat-cli history "Alice" --type link
wechat-cli history "Alice" --media
wechat-cli history "Alice" --format jsonl
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `<chat_name>` | *必填* | 联系人名称、备注、昵称、wxid 或 `@chatroom` ID |
| `--limit <n>` | `50` | 返回数量上限 |
| `--offset <n>` | `0` | 分页偏移量 |
| `--start-time <date>` | — | 起始时间（`YYYY-MM-DD`、`YYYY-MM-DD HH:MM` 或 `YYYY-MM-DD HH:MM:SS`） |
| `--end-time <date>` | — | 结束时间 |
| `--type <type>` | — | 消息类型过滤（见下方类型表） |
| `--media` | `false` | 解析并返回图片/视频/语音/文件的本地磁盘路径 |
| `--format json\|jsonl\|text` | `json` | 输出格式 |

**输出格式**：
- `json`：`{chat, username, is_group, count, offset, limit, start_time, end_time, type, messages, failures}`。`messages` 里每条是**排好版的字符串**（时间、发送者、内容），不是对象。
- `jsonl`：每行一个 JSON 对象，便于流式消费或按字段处理：`{local_id, create_time, msg_type, local_type, is_self, sender_id, sender_display, is_group, chat_username, chat_display, text}`。查询失败的信息以 `failure: …` 输出到 stderr。

需要自己按发送者、类型或时间过滤、统计时，优先用 `jsonl`。

适合用户已经知道聊天对象，并且需要连续上下文时使用。

## search — 搜索消息

全文关键词搜索，支持单聊、多聊或全局搜索。

```bash
wechat-cli search "会议纪要"
wechat-cli search "hello" --chat "Alice"
wechat-cli search "报告" --chat "团队A" --chat "团队B"
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
- 不指定 `--chat` -> 全局搜索所有消息
- 指定 1 个 `--chat` -> 单聊搜索
- 指定多个 `--chat` -> 多聊搜索，结果按时间合并排序

**JSON 输出**：`{scope, keyword, count, offset, limit, start_time, end_time, type, results, failures}`。`results` 里每条是排好版的字符串：`"[YYYY-MM-DD HH:MM] [聊天名] 发送者: 内容"`。

适合“谁说过这句话”“这个关键词在哪些群出现过”。

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

在名字不确定、备注和昵称可能不一致时，先用它做目标确认。

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

**JSON 输出**：`group`、`username`（`@chatroom` ID）、`member_count`、`owner`、`members`（包含 `username`、`nick_name`、`remark`、`display_name`，群主排第一）

如果目标不是群聊，这个命令会报错。

## stats — 聊天统计

聚合指定聊天的统计数据（消息总量、类型分布、活跃发送者、24 小时活跃度）。

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

**JSON 输出**：`chat`、`username`、`is_group`、`total`（总消息数）、`type_breakdown`（类型分布）、`top_senders`（前 10 活跃发送者）、`hourly`（24 小时活跃度分布）

适合“谁最活跃”“这个群主要发什么类型消息”这类问题。

## export — 导出记录

将聊天记录导出为 Markdown 或纯文本。

```bash
wechat-cli export "Alice" --format markdown
wechat-cli export "Alice" --format txt --output chat.txt
wechat-cli export "工作群" --start-time "2026-04-01" --limit 1000
wechat-cli export "Alice" --media --output chat.md
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `<chat_name>` | *必填* | 聊天名称或 wxid |
| `--format markdown\|txt` | `markdown` | 导出格式（注意：不支持 json） |
| `--output <path>` | stdout | 输出文件路径（不指定则输出到终端） |
| `--start-time <date>` | — | 起始时间 |
| `--end-time <date>` | — | 结束时间 |
| `--limit <n>` | `500` | 导出数量上限 |
| `--media` | `false` | 解析并写入图片/视频/语音/文件的本地磁盘路径 |

当用户明确要文件产物或长文本留存时使用；若只是问答，不要优先导出。

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

**JSON 输出**：`{count, favorites}`，每项字段为 `id`、`type`、`time`、`summary`、`from`、`source_chat`。

适合找“我收藏过什么”“收藏里有没有某篇文章/某个关键词”。

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

**JSON 输出**：与 `sessions` 相同的数组，只包含有未读消息的会话。

适合快速确认当前待处理会话。

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
- **首次调用**（无状态文件）：返回当前未读会话并保存时间戳。JSON：`{first_call: true, unread_count, messages}`
- **后续调用**：对比时间戳差异，返回有新消息的会话。JSON：`{first_call: false, new_count, messages}`
- **重置**：删除 `~/.wechat-cli/last_check.json` 即可重置状态

每次调用都会把检查点往后移，所以不要为了“看一眼”而调用；只是想看看时用 `unread` 或 `sessions`。

适合“从我上次检查之后发生了什么”这类请求。

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
1. 精确匹配已知的名称或 ID（wxid 如 `wxid_xxx`、群 ID 如 `xxx@chatroom` 会直接使用）
2. 对显示名做不区分大小写的精确匹配
3. 对显示名做不区分大小写的子串匹配

太短的子串可能匹配到错误的聊天。请核对结果里的 `chat` / `username` 字段；需要精确时直接传 wxid 或 `@chatroom` ID。

## 错误与退出码

错误信息以中文输出到 stderr。

| 退出码 | 含义 | 处理方式 |
|---|---|---|
| `0` | 成功 | — |
| `1` | 找不到聊天对象、该聊天没有消息记录，或尚未初始化 | stderr 出现 `请运行: wechat-cli init` 时见“初始化”；否则先用 `contacts --query`，或改用备注、昵称、wxid、`@chatroom` ID |
| `2` | 参数错误（时间格式、分页） | 修正参数 |
| `3` | 数据库无法解密或打开 | 密钥可能已失效，说明风险后征求确认再 `init --force` |

例外：`contacts --detail` 找不到联系人时会在 stderr 输出 `找不到联系人`，但退出码仍是 `0`，需检查 stdout 是否为空。

## 常见处理策略

- **找不到联系人/群** -> 先用 `contacts --query`，再尝试备注、昵称、wxid 或 `@chatroom`
- **只知道关键词，不知道聊天对象** -> 先 `search`，再对命中的聊天补 `history`
- **结果太多** -> 缩小时间范围、聊天范围或 `--limit`
- **想要完整留档** -> 用 `export`
- **想要增量查看** -> 用 `new-messages`
- **未初始化或密钥失效** -> 解释 `init` / `init --force` 风险并征求确认
- **Windows 上中文乱码** -> 运行前设置 `PYTHONUTF8=1`（或 `PYTHONIOENCODING=utf-8`），管道输出时尤其需要

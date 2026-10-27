---
name: wechat-cli
version: 1.2.0
description: |
  Read-only queries over the WeChat data stored on this machine. Use this skill whenever the user wants to look at their WeChat chat history, search messages, look up contacts or group members, see recent or unread conversations, browse favorites, get chat statistics, or export a conversation — e.g. "who in the group said X", "what did Alice send me last week", "export this chat", "who has messaged me recently", "who is most active in this group". Trigger it even if the user does not mention wechat-cli by name.
metadata:
  requires:
    bins: ["wechat-cli"]
  cliHelp: "wechat-cli --help"
---

# wechat-cli

`wechat-cli` is a **read-only**, fully local tool. It decrypts the WeChat databases on this machine on the fly and returns structured results. It never goes online and never modifies chat data.

## When to use

Use it whenever the answer lives in the user's **local WeChat data**:

- "Find who in WeChat said 'budget freeze'"
- "Show my chat with Alice"
- "Who has contacted me recently?" / "Any unread messages?"
- "Who is in the product group?" / "Who is most active there?"
- "Export yesterday's chat with Bob"
- "Did I save that article to favorites?"
- "What came in since I last checked?"

## Ground rules

1. **Read-only by default.** Run only query commands: `sessions`, `history`, `search`, `contacts`, `members`, `stats`, `export`, `favorites`, `unread`, `new-messages`.
2. **`init` is never automatic.** Run `init` / `init --force` only when the user explicitly asks to set up, or a query fails because the tool is not initialized — and then explain what it does and get confirmation first.
3. **Never print or share `~/.wechat-cli/all_keys.json`.** It holds the database encryption keys.
4. **Minimal disclosure.** Results can include wxids, phone numbers, avatar URLs, signatures and local media paths. Return only the fields needed to answer the question unless the user asks for more.
5. **Query the smallest scope that answers the question.** Narrow by chat, time range and `--limit` before pulling large result sets.
6. **Lead with the answer.** Summarize first, then quote a few key messages as evidence. Return raw JSON or full output only when the user wants it.

## Picking a command

| User intent | Command | Notes |
|---|---|---|
| Who have I chatted with recently | `sessions` | Recent conversations, newest first |
| Read a specific chat | `history` | Paging, time range, type filter |
| Find a phrase / keyword | `search` | Global, one chat, or several chats |
| Look up a contact or WeChat ID | `contacts` | `--query` to search, `--detail` for one contact |
| List group members | `members` | Group chats only |
| Most active people / message mix | `stats` | Works for groups and 1:1 chats |
| Save a conversation to a file | `export` | `markdown` or `txt` |
| Browse WeChat favorites | `favorites` | Filter by type or keyword |
| What is unread right now | `unread` | All conversations with unread messages |
| What is new since last check | `new-messages` | Stateful, incremental |

### Typical workflow

1. **Identify the chat.** If the name is ambiguous (remark vs. nickname, similar group names), run `contacts --query` first. A wxid (`wxid_…`) or group ID (`…@chatroom`) can be used directly.
2. **Choose the query.** Continuous context → `history`. Keyword hits → `search`. Overview → `sessions` / `unread` / `favorites`. Numbers → `stats`. A file → `export`.
3. **Narrow it.** Add `--chat`, `--start-time`, `--end-time`, `--limit`, `--offset`, `--type` as needed.
4. **Don't know the chat but know the words?** `search` globally, then `history` on the chat that matched for context.

### Natural language → command

| Request | Command |
|---|---|
| "Who said 'budget freeze'?" | `wechat-cli search "budget freeze"` |
| "What has Alice been saying lately?" | `wechat-cli history "Alice" --limit 50` |
| "Export my chat with Bob from yesterday" | `wechat-cli export "Bob" --start-time "YYYY-MM-DD" --end-time "YYYY-MM-DD"` |
| "Who is in the product group?" | `wechat-cli members "Product Group"` |
| "Who is most active in this group?" | `wechat-cli stats "Group Name"` |
| "Anything unread?" | `wechat-cli unread` |
| "What's new since I last checked?" | `wechat-cli new-messages` |

## Global options

| Option | Default | Description |
|---|---|---|
| `--config <path>` | `~/.wechat-cli/config.json` | Config file path; can also be set with the `WECHAT_CLI_CONFIG` environment variable |
| `--version` | — | Print the version |

`--format` is a per-command option, not global. Most commands accept `json|text` (default `json`); `history` also accepts `jsonl`; `export` accepts `markdown|txt` only.

## Commands

Times for `--start-time` / `--end-time` use `YYYY-MM-DD`, `YYYY-MM-DD HH:MM` or `YYYY-MM-DD HH:MM:SS`. A date-only `--end-time` covers the whole day (it ends at 23:59:59), so `--start-time 2026-04-01 --end-time 2026-04-01` is all of April 1.

`history`, `search` and `export` pick the **newest** matching messages (`--offset` skips back from the newest), then list that page in **chronological order**, oldest first. `sessions` and `unread` are listed newest first.

### sessions — recent conversations

```bash
wechat-cli sessions
wechat-cli sessions --limit 10
wechat-cli sessions --format text
```

| Option | Default | Description |
|---|---|---|
| `--limit <n>` | `20` | Max conversations |
| `--format json\|text` | `json` | Output format |

JSON: an array of `{chat, username, is_group, unread, last_message, msg_type, sender, timestamp, time}`.

### history — messages in one chat

```bash
wechat-cli history "Alice"
wechat-cli history "Alice" --limit 100 --offset 50
wechat-cli history "Work Group" --start-time "2026-04-01" --end-time "2026-04-03"
wechat-cli history "Alice" --type link
wechat-cli history "Alice" --media
wechat-cli history "Alice" --format jsonl
```

| Option | Default | Description |
|---|---|---|
| `<chat_name>` | *required* | Remark, nickname, display name, wxid or `@chatroom` ID |
| `--limit <n>` | `50` | Max messages |
| `--offset <n>` | `0` | Paging offset |
| `--start-time <time>` | — | Start of range |
| `--end-time <time>` | — | End of range |
| `--type <type>` | — | Message type filter (see below) |
| `--media` | off | Resolve local file paths for images, videos, voice and files |
| `--format json\|jsonl\|text` | `json` | Output format |

- **`json`:** `{chat, username, is_group, count, offset, limit, start_time, end_time, type, messages, failures}`. Each entry in `messages` is a **pre-formatted string** (timestamp, sender, text), not an object.
- **`jsonl`:** one JSON object per line, for streaming or field-level processing: `{local_id, create_time, msg_type, local_type, is_self, sender_id, sender_display, is_group, chat_username, chat_display, text}`. Query failures go to stderr as `failure: …` lines.

Prefer `jsonl` when you need to filter or count by sender, type or time yourself.

### search — keyword search

```bash
wechat-cli search "meeting notes"
wechat-cli search "hello" --chat "Alice"
wechat-cli search "report" --chat "Team A" --chat "Team B"
wechat-cli search "contract" --type file --start-time "2026-01-01"
```

| Option | Default | Description |
|---|---|---|
| `<keyword>` | *required* | Substring match (SQL `LIKE`) |
| `--chat <name>` | — | Limit to a chat; repeat for several chats |
| `--start-time <time>` | — | Start of range |
| `--end-time <time>` | — | End of range |
| `--limit <n>` | `20` | Max results (hard cap 500) |
| `--offset <n>` | `0` | Paging offset |
| `--type <type>` | — | Message type filter |
| `--format json\|text` | `json` | Output format |

Scope: no `--chat` searches every chat; one `--chat` searches that chat; several are searched together and merged by time.

JSON: `{scope, keyword, count, offset, limit, start_time, end_time, type, results, failures}`; each entry in `results` is a formatted string `"[YYYY-MM-DD HH:MM] [chat] sender: text"`.

### contacts — look up contacts

```bash
wechat-cli contacts --query "Zhang"
wechat-cli contacts --detail "Alice"
wechat-cli contacts --detail "wxid_xxx"
```

| Option | Default | Description |
|---|---|---|
| `--query <text>` | `""` | Case-insensitive match on nickname, remark or wxid |
| `--detail <name>` | — | Full record for one contact |
| `--limit <n>` | `50` | Max results in list mode |
| `--format json\|text` | `json` | Output format |

Detail mode adds `alias` (WeChat ID), `description` (signature), `avatar` (URL), `is_group`, `is_subscription` (official account). Use it to confirm the target before other queries when names are ambiguous.

### members — group members

```bash
wechat-cli members "Work Group"
wechat-cli members "xxx@chatroom"
```

| Option | Default | Description |
|---|---|---|
| `<group_name>` | *required* | Group name, remark or `@chatroom` ID |
| `--format json\|text` | `json` | Output format |

JSON: `{group, username, member_count, owner, members}`; each member has `username`, `nick_name`, `remark`, `display_name`, with the owner listed first. Fails if the target is not a group.

### stats — chat statistics

```bash
wechat-cli stats "Work Group"
wechat-cli stats "Alice" --start-time "2026-04-01" --end-time "2026-04-07"
```

| Option | Default | Description |
|---|---|---|
| `<chat_name>` | *required* | Chat name or wxid |
| `--start-time <time>` | — | Start of range |
| `--end-time <time>` | — | End of range |
| `--format json\|text` | `json` | Output format |

JSON: `{chat, username, is_group, total, type_breakdown, top_senders, hourly}` — `top_senders` is the 10 most active senders, `hourly` is a 24-hour activity distribution.

### export — save a conversation

```bash
wechat-cli export "Alice" --format markdown
wechat-cli export "Alice" --format txt --output chat.txt
wechat-cli export "Work Group" --start-time "2026-04-01" --limit 1000
wechat-cli export "Alice" --media --output chat.md
```

| Option | Default | Description |
|---|---|---|
| `<chat_name>` | *required* | Chat name or wxid |
| `--format markdown\|txt` | `markdown` | Export format (no JSON) |
| `--output <path>` | stdout | Write to a file instead of stdout |
| `--start-time <time>` | — | Start of range |
| `--end-time <time>` | — | End of range |
| `--limit <n>` | `500` | Max messages |
| `--media` | off | Resolve local file paths for images, videos, voice and files |

Use it when the user wants a file or a long-lived copy. For a question about the chat, use `history` or `search` instead.

### favorites — WeChat favorites

```bash
wechat-cli favorites
wechat-cli favorites --type article
wechat-cli favorites --query "machine learning" --limit 5
```

| Option | Default | Description |
|---|---|---|
| `--limit <n>` | `20` | Max items |
| `--type <type>` | — | `text`, `image`, `article`, `card` (contact card), `video` (Channels video) |
| `--query <text>` | — | Keyword search |
| `--format json\|text` | `json` | Output format |

JSON: `{count, favorites}`; each item has `id`, `type`, `time`, `summary`, `from`, `source_chat`.

### unread — unread conversations

```bash
wechat-cli unread
wechat-cli unread --limit 10 --format text
```

| Option | Default | Description |
|---|---|---|
| `--limit <n>` | `50` | Max conversations |
| `--format json\|text` | `json` | Output format |

JSON: same shape as `sessions`, filtered to conversations with unread messages.

### new-messages — incremental updates

```bash
wechat-cli new-messages
wechat-cli new-messages --format text
```

| Option | Default | Description |
|---|---|---|
| `--format json\|text` | `json` | Output format |

- **First call** (no state file): returns the current unread conversations and records a checkpoint. JSON: `{first_call: true, unread_count, messages}`.
- **Later calls:** return only conversations with messages newer than the checkpoint. JSON: `{first_call: false, new_count, messages}`.
- **Reset:** delete `~/.wechat-cli/last_check.json`.

Each call moves the checkpoint forward, so don't call it just to peek — use `unread` or `sessions` for that.

## Message types

`--type` on `history` and `search`:

| Value | Meaning |
|---|---|
| `text` | Text |
| `image` | Image |
| `voice` | Voice message |
| `video` | Video |
| `sticker` | Sticker / emoji |
| `location` | Shared location |
| `link` | Link, mini program, article |
| `file` | File |
| `call` | Voice / video call |
| `system` | System message (joined, left, etc.) |

## Chat name matching

Every `<chat_name>` and `--chat` argument is resolved in this order:

1. Exact match on a known name or ID (a wxid or `@chatroom` ID is used directly)
2. Case-insensitive exact match on the display name
3. Case-insensitive substring match on the display name

A short substring can match the wrong chat. Check the `chat` / `username` fields in the result, and pass the wxid or `@chatroom` ID when it matters.

## Errors and exit codes

Error messages are printed to stderr in Chinese.

| Exit code | Meaning | What to do |
|---|---|---|
| `0` | Success | — |
| `1` | Chat not found, no messages for it, or not initialized | If stderr says `请运行: wechat-cli init` ("please run wechat-cli init"), see Initialization. Otherwise try `contacts --query`, a remark, nickname, wxid or `@chatroom` ID |
| `2` | Invalid argument (time format, paging) | Fix the argument |
| `3` | Database could not be decrypted or opened | Keys may be stale — offer `init --force` (with confirmation) |

`contacts --detail` is the exception: when the contact is not found it prints `找不到联系人` ("contact not found") to stderr but still exits `0`, so check for empty stdout.

## Initialization

`init` extracts the database keys from the running WeChat process's memory. WeChat must be **running and logged in**. Run it only with the user's explicit consent.

```bash
wechat-cli init                   # Windows: run from an Administrator terminal
sudo wechat-cli init              # macOS / Linux usually need sudo
wechat-cli init --db-dir <path>   # Specify the WeChat data directory if auto-detection fails
wechat-cli init --force           # Re-extract keys (e.g. after a WeChat update or re-login)
```

It writes `~/.wechat-cli/config.json` (database location) and `~/.wechat-cli/all_keys.json` (encryption keys — **never print or share**).

## Tips

- **Too many results:** narrow the time range, add `--chat`, or lower `--limit`, then page with `--offset`.
- **Garbled Chinese on Windows:** set `PYTHONUTF8=1` (or `PYTHONIOENCODING=utf-8`) before running, especially when piping output.
- **"The latest message"** is the **last** entry of a `history` or `search` page, not the first.

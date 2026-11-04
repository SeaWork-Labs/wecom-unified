# chat groups list API

按时间范围分页获取有消息的会话列表。当前**只返回群聊会话**。

## 命令

```bash
wecom-cli chat groups list --json '<JSON 参数>'
```

## 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `begin_time` | string | ✅ | 查询开始时间，格式 `YYYY-MM-DD HH:MM:SS`，仅支持最近 7 天 |
| `end_time` | string | ✅ | 查询结束时间，格式 `YYYY-MM-DD HH:MM:SS`，需晚于 `begin_time` |
| `cursor` | string | ❌ | 翻页游标，首次不传，后续传上次响应的 `next_cursor` |

## 请求示例

```bash
wecom-cli chat groups list --json '{"begin_time": "2026-03-11 00:00:00", "end_time": "2026-03-17 23:59:59"}'
```

分页请求：

```bash
wecom-cli chat groups list --json '{"begin_time": "2026-03-11 00:00:00", "end_time": "2026-03-17 23:59:59", "cursor": "NEXT_CURSOR"}'
```

## 返回字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `chats` | array | 会话列表 |
| `chats[].chat_id` | string | 群会话 ID，可直接作为 `chat messages list` 的 `chat_id` 使用 |
| `chats[].chat_name` | string | 会话名称，下游未提供时为空 |
| `chats[].last_msg_time` | string | 该会话最后一条消息时间，格式 `YYYY-MM-DD HH:MM:SS` |
| `chats[].msg_count` | integer | 该会话在查询时间范围内的消息条数 |
| `chats_count` | integer | `chats` 数组元素数量 |
| `has_more` | boolean | 是否还有更多数据 |
| `next_cursor` | string | 下一页游标，为空表示没有更多 |

## 响应示例

```json
{
    "chats": [
        {
            "chat_id": "CHAT_ID",
            "chat_name": "项目讨论群",
            "last_msg_time": "2026-03-17 15:30:45",
            "msg_count": 128
        }
    ],
    "chats_count": 1,
    "has_more": true,
    "next_cursor": "NEXT_CURSOR"
}
```

## 注意

- 只返回**群聊**会话。单聊没有列表接口，要查单聊记录必须先知道对方 `userid`（用 `contact users search` 获取），再直接调 `chat messages list`
- 时间窗口最长 7 天，超出会被后端拒绝

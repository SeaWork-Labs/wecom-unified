# chat messages list API

按时间范围分页拉取指定会话的消息列表。

## 命令

```bash
wecom-cli chat messages list --json '<JSON 参数>'
```

## 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `chat_id` | string | ✅ | 会话 ID：单聊传对方成员的 `userid`，群聊传群会话 ID |
| `begin_time` | string | ✅ | 拉取开始时间，格式 `YYYY-MM-DD HH:MM:SS`，仅支持最近 7 天 |
| `end_time` | string | ✅ | 拉取结束时间，格式 `YYYY-MM-DD HH:MM:SS`，需晚于 `begin_time` |
| `cursor` | string | ❌ | 翻页游标，首次不传，后续传上次响应的 `next_cursor` |

`chat_id` 的类型由框架自行判断，不再需要 `chat_type` 参数。

## 请求示例

群聊：

```bash
wecom-cli chat messages list --json '{"chat_id": "wrxxxxxxxx", "begin_time": "2026-03-17 09:00:00", "end_time": "2026-03-17 18:00:00"}'
```

单聊（`userid` 来自 `contact users search`）：

```bash
wecom-cli chat messages list --json '{"chat_id": "zhangsan", "begin_time": "2026-03-17 09:00:00", "end_time": "2026-03-17 18:00:00"}'
```

## 返回字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `messages` | array | 消息列表，**按消息时间正序排列** |
| `messages[].msg_type` | string | `text` / `image` / `file` / `voice` / `video` / `mixed` |
| `messages[].send_time` | string | 发送时间，格式 `YYYY-MM-DD HH:MM:SS` |
| `messages[].userid` | string | 发送者 userid |
| `messages[].user_name` | string | 发送者名称（接口已附带，通常不需要再查通讯录） |
| `messages[].text.content` | string | 文本内容，`msg_type=text` 时返回 |
| `messages[].image.media_id` | string | 图片媒体 ID，`msg_type=image` 时返回 |
| `messages[].file.media_id` / `.file_name` | string | 文件媒体 ID 与文件名，`msg_type=file` 时返回 |
| `messages[].voice.media_id` | string | 语音媒体 ID，`msg_type=voice` 时返回 |
| `messages[].video.media_id` | string | 视频媒体 ID，`msg_type=video` 时返回 |
| `messages[].mixed.items` | array | 图文混排内容项，`msg_type=mixed` 时返回 |
| `messages_count` | integer | `messages` 数组元素数量 |
| `has_more` | boolean | 是否还有更多数据 |
| `next_cursor` | string | 下一页游标，为空表示没有更多 |

> ⚠️ `user_name` 通常已经由接口带回，**优先直接使用**，不要为了展示姓名再调一次通讯录。只有 `user_name` 为空时才回落到 `contact users search`。

## 响应示例

```json
{
    "messages": [
        {
            "msg_type": "text",
            "send_time": "2026-03-17 09:12:33",
            "userid": "zhangsan",
            "user_name": "张三",
            "text": {"content": "今天的评审推迟到下午三点"}
        },
        {
            "msg_type": "image",
            "send_time": "2026-03-17 09:15:02",
            "userid": "lisi",
            "user_name": "李四",
            "image": {"media_id": "MEDIAID_xxxxxx"}
        }
    ],
    "messages_count": 2,
    "has_more": false,
    "next_cursor": ""
}
```

## 注意

- 时间窗口最长 7 天
- 结果按时间正序，展示时通常按此顺序即可
- 非文本消息只有 `media_id`，内容要用 `message files get` 取

# message send API

向单聊或群聊发送文本消息。

## 命令

```bash
wecom-cli message send --json '<JSON 参数>'
```

## 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `chat_id` | string | ✅ | 会话 ID：单聊传对方成员的 `userid`，群聊传群会话 ID |
| `msg_type` | string | ✅ | 消息类型，目前仅支持 `text` |
| `text` | object | 条件必填 | 文本消息内容，`msg_type=text` 时必填 |
| `text.content` | string | ✅ | 消息内容，最长 2048 字符 |

> ⚠️ 旧版的 `chat_type` + `chatid` + `msgtype` 三件套已经废弃，现在只有 `chat_id` + `msg_type`。会话类型由框架根据 `chat_id` 自行判断。

## 请求示例

单聊（`userid` 来自 `contact users search`）：

```bash
wecom-cli message send --json '{"chat_id": "zhangsan", "msg_type": "text", "text": {"content": "hello world"}}'
```

群聊（群 ID 来自 `chat groups list`）：

```bash
wecom-cli message send --json '{"chat_id": "wrxxxxxxxx", "msg_type": "text", "text": {"content": "大家好"}}'
```

## 返回字段

成功与否由框架外壳的 `errcode` / `errmsg` 表达，无业务字段。

```json
{
    "errcode": 0,
    "errmsg": "ok"
}
```

## 注意

- `text.content` 最长 2048 字符，超限时不要静默截断，先请用户缩短或拆分
- 发送成功后只说明发送目标与消息类型，不要编造消息 ID
- 需要发送图片 / 文件 / 语音 / 视频时用 `message aibot send`（见 [wecom-msg.md](wecom-msg.md)）

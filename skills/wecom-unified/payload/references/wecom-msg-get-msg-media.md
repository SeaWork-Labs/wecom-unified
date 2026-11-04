# message files get API

根据消息媒体 ID 获取图片 / 文件 / 语音 / 视频的实际内容。CLI 会把文件落盘，并返回本地路径。

## 命令

```bash
wecom-cli message files get --json '<JSON 参数>'
```

## 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `media_id` | string | ✅ | 消息媒体 ID，取自 `chat messages list` 返回的 `image` / `file` / `voice` / `video` 里的 `media_id` |

## 请求示例

```bash
wecom-cli message files get --json '{"media_id": "MEDIAID_xxxxxx"}'
```

指定落盘目录（默认是当前工作目录）：

```bash
wecom-cli message files get --json '{"media_id": "MEDIAID_xxxxxx"}' --output-dir ./downloads
```

## 返回字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `media_item.media_id` | string | 媒体 ID，与入参一致 |
| `media_item.media_type` | string | `image` / `file` / `voice` / `video` |
| `media_item.file_name` | string | 文件名（含扩展名） |
| `media_item.file_path` | string | 落盘后的本地绝对路径 |
| `media_item.content` | string | 内容不长时直接返回；CLI 模式下通常走 `file_path` |

## 响应示例

```json
{
    "media_item": {
        "media_id": "MEDIAID_xxxxxx",
        "media_type": "image",
        "file_name": "screenshot.png",
        "file_path": "/absolute/path/to/screenshot.png"
    }
}
```

## 注意

- 文件默认落在**当前工作目录**，可用 `--output-dir` 或 `--output` 改
- 下载后检查 `file_path` 指向的文件是否存在、大小是否大于 0；为空或缺失时如实告知用户下载异常
- 这些是从聊天记录里下载的历史附件，只告知本地路径即可，**不要**用媒体指令把它们重新发回给用户

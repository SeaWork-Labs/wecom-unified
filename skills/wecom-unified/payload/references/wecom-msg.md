# 企业微信消息技能

> 公共概念与规则请参考 [wecom-shared.md](wecom-shared.md)

通过 `wecom-cli chat ...` 与 `wecom-cli message ...` 与企业微信消息系统交互。

---

## 接口列表

### chat groups list — 获取群会话列表

```bash
wecom-cli chat groups list --json '{"begin_time": "2026-03-11 00:00:00", "end_time": "2026-03-17 23:59:59"}'
```

按时间范围查询有消息的**群聊**会话列表，支持分页。参见 [API 详情](wecom-msg-get-msg-chat-list.md)。

### chat messages list — 拉取会话消息

```bash
wecom-cli chat messages list --json '{"chat_id": "zhangsan", "begin_time": "2026-03-17 09:00:00", "end_time": "2026-03-17 18:00:00"}'
```

按会话 ID 和时间范围拉取消息记录，支持分页。返回 text / image / file / voice / video / mixed 六种类型，仅支持 7 天内。参见 [API 详情](wecom-msg-get-message.md)。

### message files get — 获取消息媒体内容

```bash
wecom-cli message files get --json '{"media_id": "MEDIAID_xxxxxx"}'
```

根据媒体 ID 下载文件到本地，返回 `file_path`、`file_name`、`media_type`。用于获取图片、文件、语音、视频的实际内容。参见 [API 详情](wecom-msg-get-msg-media.md)。

### message send — 发送文本消息

```bash
wecom-cli message send --json '{"chat_id": "zhangsan", "msg_type": "text", "text": {"content": "hello world"}}'
```

向单聊或群聊发送文本消息。参见 [API 详情](wecom-msg-send-message.md)。

### message aibot sessions list / message aibot send — 机器人会话与富媒体发送

```bash
wecom-cli message aibot sessions list
wecom-cli message aibot send --json '{"chat_id": "CHAT_ID", "msg_type": "markdown", "markdown": {"content": "**周报**"}}'
```

`aibot send` 支持 markdown / image / file / voice / video 五种类型，媒体文件需先用 `media upload` 换取 `media_id`。

---

## 核心规则

### 时间范围规则
- **格式**：所有时间参数使用 `YYYY-MM-DD HH:MM:SS`
- **默认范围**：用户未指定时，默认最近 7 天
- **限制**：消息类接口只允许最近 7 天，超出会被后端拒绝
- **相对时间支持**：支持"昨天"、"最近三天"等自动推算

### chat_id 查找规则

**群聊**：当用户给出群名时：
1. 调用 `chat groups list`（时间范围与目标查询一致）
2. 在 `chats` 中按 `chat_name` 匹配
3. 匹配策略：
   - 唯一命中：直接用其 `chat_id`
   - 多个候选：按返回顺序展示聊天名和最后消息时间，让用户选择
   - 无命中：告知用户未找到

**单聊**：`chat groups list` 只返回群聊，单聊必须走通讯录：
1. 调用 `contact users search --json '{"keywords": ["张三"]}'`（参见 [wecom-contact.md](wecom-contact.md)）
2. 确认唯一命中的 `userid`
3. 把该 `userid` 直接作为 `chat_id`

> ⚠️ 不要用猜测或用户口述的 ID 作为 `chat_id`。用户提供的 ID 只能当作匹配线索，最终值必须来自接口返回。

### user_name 优先

`chat messages list` 已经在每条消息上返回 `user_name`，**直接用它展示发送者姓名**。只有 `user_name` 为空时才回落到通讯录查询。

### 强制交互步骤（不可跳过）

涉及非文本消息下载时，以下步骤必须逐一执行，不得合并或省略：

1. **必须主动告知文件位置**：下载完成后立即以汇总形式展示存放目录和每个文件的完整路径
2. **必须询问是否删除**：告知位置后立即询问用户是否需要清理临时文件

---

## 典型工作流

### 查看群聊列表
**用户query示例**：
- "看看我最近一周有哪些群消息"
- "这几天哪些群活跃"

**执行流程**：
1. 确定时间范围（用户指定或默认最近 7 天）
2. 调用 `chat groups list`
3. 展示群名、最后消息时间、消息数量
4. 若 `has_more` 为 `true`，告知用户还有更多可继续查看

### 查看聊天记录
**用户query示例**：
- "帮我看看和张三最近的聊天记录"
- "看看项目群里最近的消息"

**执行流程**：
1. 确定时间范围
2. 按 **chat_id 查找规则** 确定目标 `chat_id`
3. 调用 `chat messages list` 拉取消息
4. 展示消息时直接使用 `user_name`：
   - 文本：`姓名 [时间]: 内容`
   - 图片：`姓名 [时间]: [图片]`
   - 文件：`姓名 [时间]: [文件] 文件名`
   - 语音：`姓名 [时间]: [语音]`
   - 视频：`姓名 [时间]: [视频]`
5. **非文本消息处理**：统计非文本消息数量和类型，主动询问是否下载
   - 用户确认后，逐个调用 `message files get` 拿到 `file_path`
   - 检查文件是否存在、大小是否大于 0；文件名缺后缀时按 `media_type` 补齐
   - ⚠️ **不要用媒体指令把下载的文件重新发回给用户**
6. ⚠️ **必须主动告知文件位置**：以汇总形式展示目录和每个文件的完整路径
7. ⚠️ **必须询问是否删除**：随后立即询问是否需要清理临时文件
8. 若 `next_cursor` 不为空，告知用户还有更多消息可继续查看

### 发送消息
**用户query示例**：
- "帮我给张三发一条消息：明天会议改到下午3点"
- "在项目群里发一条消息：今天下午3点开会"

**执行流程**：
1. 按 **chat_id 查找规则** 确定目标 `chat_id`
2. **发送前确认**：向用户确认发送对象和内容（如："即将向 张三 发送：'明天会议改到下午3点'，确认发送吗？"）
3. 调用 `message send`，`msg_type` 固定为 `text`
4. 展示发送结果

### 查看消息并回复
**用户query示例**：
- "看看张三给我发了什么，然后帮我回复收到"

**执行流程**：
1. 先执行"查看聊天记录"流程（复用已确定的 `chat_id`）
2. 展示消息后，执行"发送消息"流程（需确认后再发送）

---

## 错误处理
- **时间范围超限**：告知用户 7 天限制并调整为有效范围
- **会话未找到**：明确告知用户未找到对应会话
- **`unrecognized subcommand`**：CLI 版本过旧，先升级到 `>= 1.2.1`
- **API 错误**：展示 `errcode` 与 `errmsg`，必要时重试
- **网络问题**：HTTP 错误时主动重试最多 3 次

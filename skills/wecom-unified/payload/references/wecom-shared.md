# 公共概念与规则

本文档包含所有业务域共享的公共内容：CLI 安装与版本要求、凭证配置、通用调用格式、返回格式、错误处理、人员查询方法和时间格式规范。

---

## CLI 安装与版本要求

- **包名**：`@wecom/cli`
- **最低版本**：`1.2.1`（本技能的命令语法从 1.2.x 起生效，0.1.x 的旧语法已不再受支持）
- **安装/升级命令**：

```bash
npm install -g @wecom/cli
```

- **检查安装与版本**：

```bash
wecom-cli --version
```

输出形如 `wecom-cli 1.3.4 (wecom 2026-09-23T11:50:48Z f9b2815)`。取第一个空格后的版本号，低于 `1.2.1` 时先升级再继续。

> ⚠️ 如果本机装的是 0.1.x，所有命令都会报 `unrecognized subcommand`。看到这个错误一律先升级，不要试图改写命令去迁就旧版。

---

## 凭证配置

### 检查凭证状态

```bash
wecom-cli auth show --status
```

- 输出 `authorized` → 已配置，可以继续
- 输出 `unauthorized` → 未配置，执行初始化
- 不带 `--status` 时输出人类可读的 `Status` 与 `Bot ID`

### 配置凭证

```bash
wecom-cli auth init --noninteractive
```

> 该命令会输出授权链接和二维码，并阻塞等待用户用企业微信扫码。授权成功后命令自动退出，仅需执行一次。
> 在非交互环境（CI、管道、脚本）中必须带 `--noninteractive`；交互终端下可以省略，由用户选择扫码或手动输入 Bot ID 与 Secret。

初始化完成后重新执行 `wecom-cli auth show --status`，只有输出 `authorized` 才继续业务命令。

---

## 通用调用格式

所有业务命令遵循统一格式：

```bash
wecom-cli <服务> [资源...] <方法> --json '<JSON 入参>'
```

- **服务**：`contact`、`chat`、`message`、`doc`、`smartpage`、`smartsheet`、`sheet`、`calendar`、`meeting`、`todo`、`mail`、`disk`、`media`、`identity`、`slide`
- **资源**：部分方法带嵌套资源路径，例如 `message aibot sessions list`、`calendar schedules create`。资源层级会出现在命令里，**不要**写成 `calendar create_schedule` 这种旧形式
- **方法**：路径的最后一段（`create`、`list`、`get` 等）
- **入参**：用 `--json '<JSON>'` 传完整请求体；也可以用 schema 生成的命名参数（如 `--subject 周例会`），两者可组合

**示例**：

```bash
# 发送文本消息
wecom-cli message send --json '{"chat_id": "zhangsan", "msg_type": "text", "text": {"content": "hello"}}'

# 查询某时间范围内的群会话
wecom-cli chat groups list --json '{"begin_time": "2026-03-11 00:00:00", "end_time": "2026-03-17 23:59:59"}'
```

**通用执行 flag**：

| Flag | 说明 |
|------|------|
| `--dry-run` | 只在本地校验并打印将要发送的请求，不实际调用 |
| `--page-count <n>` | 启用游标式自动分页，最多拉取 n 页，输出 NDJSON |
| `--output <file>` | 将响应体写入文件 |
| `--output-dir <dir>` | 指定下载文件的落盘目录（默认当前目录） |
| `--schema` / `--doc` | 输出该方法的 schema / 文档 |

**查询帮助**：

```bash
wecom-cli --help                    # 列出所有服务
wecom-cli doc --help                # 列出 doc 服务下的方法
wecom-cli doc create --help         # 查看单个方法的参数
```

---

## 通用返回格式

业务成功时，CLI 把该方法的响应 JSON 直接输出到 stdout。部分接口在外层带 `errcode` / `errmsg`：

| 字段 | 类型 | 说明 |
|------|------|------|
| `errcode` | integer | 返回码，`0` 表示成功，非 `0` 表示失败 |
| `errmsg` | string | 错误信息，成功时通常为 `"ok"` |

CLI 自身失败时，stdout 输出结构化错误，并以非零退出码结束：

```json
{
    "error": {
        "type": "AuthError",
        "code": 893201,
        "message": "..."
    }
}
```

---

## 通用错误处理与重试策略

1. **`errcode` 非 0**：把 `errcode` 和 `errmsg` 展示给用户
2. **`error` 对象**：按 `type` 判断。`AuthError` → 引导用户执行 `wecom-cli auth init`；`UsageError` / 退出码 `2` → 说明参数写错，对照 `--help` 修正参数后重试，不要原样重试
3. **`unrecognized subcommand`**：CLI 版本过旧，升级到 `>= 1.2.1`
4. **可重试错误**：HTTP 或网络问题，主动重试，最多 3 次
5. **不可重试错误**：参数错误、权限不足，直接告知用户

---

## 人员查询（userid ↔ 姓名转换）

多个业务域（消息、日程、会议、待办等）在涉及人员操作时，需要把姓名换成 `userid`，或反向展示。

### 按关键词搜索成员

```bash
wecom-cli contact users search --json '{"keywords": ["张三"]}'
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `keywords` | string[] | 是 | 搜索关键词，最多 10 个，多个关键词之间是 OR 关系；可匹配姓名、拼音、英文名、别名 |
| `search_mode` | string | 否 | 传 `"list"` 返回全量命中列表；不传时返回最相关的少量结果 |

返回：

```json
{
    "users": [
        {
            "userid": "zhangsan",
            "name": "张三",
            "alias": "Sam",
            "email": "zhangsan@example.com",
            "position": "负责人",
            "departments": ["产品部"],
            "matched_keywords": ["张三"]
        }
    ],
    "users_count": 1,
    "hint": ""
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `users` | array | 命中的用户列表 |
| `users[].userid` | string | 用户唯一 ID |
| `users[].name` | string | 中文姓名 |
| `users[].alias` | string | 英文名 / 别名，可能为空 |
| `users[].email` | string | 邮箱，可能为空 |
| `users[].position` | string | 职务，可能为空 |
| `users[].departments` | string[] | 部门路径，主部门靠前 |
| `users_count` | integer | `users` 数组元素数量 |
| `hint` | string | 命中结果被截断时的补充说明，可能为空 |

### 姓名 → userid（用于创建/修改操作）

1. 用目标姓名作为 `keywords` 调用 `contact users search`
2. **唯一命中**：直接使用其 `userid`
3. **多个命中**：把候选人的姓名、部门、职务展示给用户选择，不要自行决定
4. **无命中**：告知用户未找到，改用别名、拼音或邮箱再搜一次

> ⚠️ 禁止根据姓名自行猜测或拼造 `userid`，必须通过通讯录接口获取。

### userid → 姓名（用于展示操作）

接口返回里的 `userid`、`creator_id`、`follower_id` 等内部标识直接展示对用户没有意义。展示前先换算：

1. 收集结果里出现的所有 ID
2. 用其中的 key 作为 `keywords` 批量查询（一次最多 10 个关键词）
3. 建立 `userid` → `name` 的映射
4. 展示时替换为姓名；查不到的标注「未知用户」

> 只在需要时才查：纯查询类结果如果已经带可读名称（如 `chat_name`、`subject`），不必额外换算。

---

## 时间格式规范

不同业务域的时间格式略有差异，请注意区分：

| 业务域 | 入参格式 | 说明 |
|--------|----------|------|
| 消息 / 会话（`chat`、`message`） | `YYYY-MM-DD HH:MM:SS` | 只能查询最近 7 天 |
| 日程（`calendar`） | `YYYY-MM-DD HH:MM:SS` | 全天日程只给日期 |
| 会议（`meeting`） | `YYYY-MM-DD HH:MM:SS` | 返回同样格式 |
| 待办（`todo`） | `YYYY-MM-DD HH:MM:SS` | 截止时间用 `deadline` 对象表达 |
| 文档（`doc`、`smartsheet`、`smartpage`） | 无时间参数 | — |

### 相对时间支持

用户说"今天"、"明天"、"昨天"、"最近三天"、"下周一"等相对时间时，根据当前日期自动推算为具体日期时间。注意消息类接口只允许最近 7 天，超出范围时先告知用户并收窄区间。

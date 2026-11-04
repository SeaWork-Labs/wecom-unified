---
name: wecom-unified
description: 企业微信 CLI 全能套件，覆盖通讯录、消息、文档（含文档/智能表格/智能文档（原名智能主页）三种品类）、日程、会议、待办 6 大业务域。支持按姓名/别名查找联系人、收发消息（文本/图片/文件/语音/视频）、以 Markdown 格式创建/读取/编辑文档、管理智能表格结构与记录、创建智能文档并读取智能文档内容为 Markdown、创建/修改/取消日程并查询闲忙、预约/管理会议、创建/跟踪/分派待办任务。即使用户未明确提到'企业微信'，只要涉及消息/日程/待办等场景也应触发本技能
version: 1.0.5
allowed-tools: Bash, Read
---

# 企业微信套件 (WeCom Unified)

企业微信 CLI (`wecom-cli`) 全能套件，通过命令行工具与企业微信系统交互，覆盖 6 大业务域：通讯录、消息、文档（含文档/智能表格/智能文档 3 种品类）、日程、会议、待办。

> ⚠️ **本技能的命令语法对应 `@wecom/cli` 1.2.x 及以上。** 0.1.x 的老语法（`msg get_msg_chat_list`、`doc create_doc`、`schedule get_schedule_list_by_range`、`meeting create_meeting`、`todo get_todo_list`、`auth show --auth-status`、裸 `init` 等）在本版本里全部作废。装上 0.1.x 时会报 `error: unrecognized subcommand` —— 先升级 CLI，不要改写命令去迁就旧版。

## ⚠️ 前置检查 — 使用任何命令前必须执行

### Step 1: 检查 CLI 是否安装、版本是否够新

```bash
wecom-cli --version
```

输出形如 `wecom-cli 1.3.4 (wecom 2026-09-23T11:50:48Z f9b2815)`。取第一个空格后的版本号，**低于 `1.2.1` 必须先升级**：

```bash
npm install -g @wecom/cli
```

命令不存在时同样执行上面的安装。

### Step 2: 检查凭证是否配置

```bash
wecom-cli auth show --status
```

- 输出 `authorized` → 已配置，可以继续使用
- 输出 `unauthorized` → 未配置，执行 Step 3

> 注意是 `--status`，不是 `--auth-status`；后者在 1.x 上会报 `error: unexpected argument '--auth-status' found`。

### Step 3: 配置凭证（仅未授权时执行）

```bash
wecom-cli auth init --noninteractive
```

> 该命令会输出一个授权链接和二维码，并阻塞等待用户扫码完成验证。授权成功后命令会自动退出，仅需执行一次。
> 初始化完成后重新执行 `wecom-cli auth show --status`，只有 `authorized` 才继续业务命令。

---

## 业务域概览

### 👤 通讯录 (contact)

按关键词搜索成员、匹配姓名/拼音/英文名/别名、查询 userid。

→ 详见 [references/wecom-contact.md](references/wecom-contact.md)

### 💬 消息 (chat / message)

群会话列表、消息记录拉取（文本/图片/文件/语音/视频）、媒体文件下载、文本与富媒体消息发送。

→ 详见 [references/wecom-msg.md](references/wecom-msg.md)

### 📄 文档、智能表格 & 智能文档 (doc / smartsheet / smartpage)

文档创建/读取/编辑（Markdown 格式），智能表格子表、字段、记录管理，智能文档创建与内容读取。支持通过 docid 或 URL 定位文档，自动识别品类并路由到对应服务。

→ 详见 [references/wecom-doc.md](references/wecom-doc.md)

### 📅 日程 (calendar)

查询日程列表与详情、创建/修改/取消日程、查询多成员闲忙与共同空闲时段。

→ 详见 [references/wecom-schedule.md](references/wecom-schedule.md)

### 🎥 会议 (meeting)

创建/查询/修改/取消预约会议、搜索会议、拉会议转写原文、查会议室。

→ 详见 [references/wecom-meeting.md](references/wecom-meeting.md)

### ✅ 待办 (todo)

查询待办列表与详情、创建/更新/删除待办、变更处理状态、分派任务。

→ 详见 [references/wecom-todo.md](references/wecom-todo.md)

---

## 公共概念与规则

所有业务域共享的通用调用格式、返回格式、错误处理、通讯录查询方法和时间格式规范。

→ 详见 [references/wecom-shared.md](references/wecom-shared.md)

---

## 快速示例

### 按姓名查通讯录

```bash
wecom-cli contact users search --json '{"keywords": ["张三"]}'
```

### 查看群会话列表

```bash
wecom-cli chat groups list --json '{"begin_time": "2026-04-08 00:00:00", "end_time": "2026-04-15 23:59:59"}'
```

### 发送文本消息

```bash
wecom-cli message send --json '{"chat_id": "zhangsan", "msg_type": "text", "text": {"content": "hello"}}'
```

### 创建文档

```bash
wecom-cli doc create --json '{"doc_name": "项目周报", "doc_type": "doc"}'
```

### 读取文档内容（Markdown 格式）

```bash
wecom-cli doc contents get --json '{"docid": "DOCID", "content_type": "markdown"}'
```

### 创建智能文档

```bash
wecom-cli smartpage create --json '{"name": "项目概览"}'
wecom-cli smartpage import --json '{"name": "项目概览", "file_path": "/path/to/overview.md"}'
```

### 读取智能文档内容

```bash
wecom-cli smartpage pages get --json '{"docid": "DOCID", "content_type": "markdown"}'
```

### 查询今天的日程

```bash
wecom-cli calendar schedules list --json '{"begin_time": "2026-04-15 00:00:00", "end_time": "2026-04-15 23:59:59"}'
```

### 创建预约会议

```bash
wecom-cli meeting create --json '{"subject": "周例会", "begin_time": "2026-04-16 15:00:00", "end_time": "2026-04-16 16:00:00"}'
```

### 查看待办列表

```bash
wecom-cli todo list --json '{"status_filter": ["proceed"], "limit": 20}'
```

### 创建待办

```bash
wecom-cli todo create --json '{"items": [{"title": "完成Q2规划文档", "deadline": {"type": "datetime", "value": "2026-04-20 09:00:00"}}]}'
```

---

## 工具说明

本 skill 在 `allowed-tools` 中声明的工具用途：

- **Bash**: 执行 `wecom-cli` 命令行工具，覆盖通讯录、消息、文档、日程、会议、待办等 6 大业务域的所有业务调用，包括前置检查（`wecom-cli --version`、`wecom-cli auth show --status`）和初始化（`wecom-cli auth init --noninteractive`）。不确定某个方法的参数时，用 `wecom-cli <服务> <方法> --help` 或 `--schema` 现查，不要猜字段名。
- **Read**: 按需读取 `references/` 下的业务详细文档（如 `wecom-doc.md`、`wecom-msg.md` 等），查阅具体命令的参数格式、字段字典、返回结构和示例；以及读取本地待发送/待上传的文件（如发送消息时的图片/文件路径、`smartpage import` 引用的 Markdown 源文件），以及 `doc contents get` 在内容超过 48KB 时返回的 `file_path`。

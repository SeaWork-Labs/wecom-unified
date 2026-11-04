# 企业微信待办事项管理

> 公共概念与规则请参考 [wecom-shared.md](wecom-shared.md)

通过 `wecom-cli todo <方法> --json '<JSON 入参>'` 与企业微信待办系统交互。

## 接口列表

| 方法 | 用途 |
|------|------|
| `todo list` | 查询待办列表（按创建时间/截止时间/状态/关键词过滤） |
| `todo get` | 按 ID 批量取待办详情（1~20 条） |
| `todo create` | 创建待办（支持批量，单次最多 20 条） |
| `todo update` | 修改待办标题、描述、分派人、截止时间 |
| `todo delete` | 删除待办（非创建人调用则语义为「退出待办」） |
| `todo finish` | 把当前用户在该待办中的部分标记为完成 |

> 旧版的 `get_todo_list` / `get_todo_detail` / `create_todo` / `update_todo` / `delete_todo` / `change_todo_user_status` 全部不存在了，一律改用上面的新方法。「变更用户状态」也随之并入 `todo finish`。

## 核心概念

### `deadline` 对象规范

待办的截止时间统一用 `deadline` 对象表达。

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `type` | string | ✅ | `date`（只到日期）/ `datetime`（精确到时分秒） |
| `value` | string | ✅ | `type=date` → `YYYY-MM-DD`；`type=datetime` → `YYYY-MM-DD HH:mm:ss` |

```json
{ "type": "date",     "value": "2026-05-08" }
{ "type": "datetime", "value": "2026-05-08 09:00:00" }
```

- 清空已设置的截止时间：`"deadline": {}`；不传该字段则保持原值
- 未设置截止时间的待办，返回里不出现 `deadline` 或为 `null`

**`remind_at_deadline`（提醒时机）必须与 `deadline` 同传**，脱离 `deadline` 单独传无效：

| 取值 | 语义 |
|------|------|
| `true` | 在截止时刻提醒（仅 `deadline.type=datetime` 可传） |
| `false` / 不传 | 按后台默认提前时间提醒 —— **不是关闭提醒** |

> 入参层面**没有「关闭提醒」这一档**。是否真正提醒由后台判断，以返回的 `extra_info` 为准。

### 状态取值

| 字段 | 取值 |
|------|------|
| `status`（整条待办） | `proceed`（进行中）/ `finished`（已完成） |
| `user_status`（当前用户在待办中的状态） | `accept` / `reject` / `finished` / `removed` / `notshow` |
| `status_filter`（查询入参） | 只接受 `finished` / `proceed`，可多选 |

### ID 与人员

- `todo_id` 前缀为 `td`，必须来自 `todo list` / `todo get` / `todo create` 的返回，禁止推测构造；**禁止展示给用户**
- `userid` 前缀为 `wo`，姓名用 `wecom-cli contact users search` 查（参见 [wecom-contact.md](wecom-contact.md)）
- 返回里的 `creator.user_name` / `followers[].user_name` 已附带人名，格式 `英文名(中文名)`，**展示时直接用，不必再查通讯录**

---

## 1. todo list — 查询待办列表

```bash
wecom-cli todo list --json '{"create_begin_time": "2026-05-01 00:00:00", "create_end_time": "2026-05-09 23:59:59", "status_filter": ["proceed"], "limit": 20}'
```

`list` 返回的就是完整字段（标题、描述、分派人、创建人、截止时间），**不需要再逐条查详情**。

### 入参（平铺，不套 `items`）

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `create_begin_time` | string | ❌ | 创建时间起，`YYYY-MM-DD HH:mm:ss` |
| `create_end_time` | string | ❌ | 创建时间止 |
| `deadline_begin_time` | string | ❌ | 截止时间起 |
| `deadline_end_time` | string | ❌ | 截止时间止 |
| `status_filter` | string[] | ❌ | `finished` / `proceed`，可多选；**不传时默认只返回进行中** |
| `keywords` | string[] | ❌ | 对待办标题/描述做字面命中过滤 |
| `limit` | integer | ❌ | 单次条数，默认 10，最大 20 |
| `cursor` | string | ❌ | 分页游标，首次不传 |

命令行翻页参数写在 `--json` 之外：`--page-count N` 自动最多拉 N 页。

### `keywords` 语义

- 数组元素之间是 **OR**
- 单个元素内空格分隔的词是 **AND**

`["项目 评审", "报销"]` ≡ `("项目" AND "评审") OR "报销"`。

### 返回

| 字段 | 类型 | 说明 |
|------|------|------|
| `items` | array | 待办列表 |
| `next_cursor` | string | 下一页游标 |
| `has_more` | boolean | 是否还有更多 |

`items[]`：`{todo_id, title, description, status, user_status, creator{userid,user_name}, followers[]{userid,user_name,user_status,update_time}, deadline, extra_info, source, create_time, update_time}`

### 使用要点

- **按状态查必须显式传 `status_filter`**：不传只有进行中。查"已完成的"传 `["finished"]`，查"所有"传 `["finished","proceed"]`。漏传会把"其实有"误判成"没有"
- **不能查已删除**：`status_filter` 不接受 `deleted`，用户要查已删除待办时直接说明不支持
- **时间范围默认归到创建时间**：用户说"5 月 1 号到 9 号"没点明创建还是截止时，用 `create_*`；只有出现"截止/到期/ddl"等字样才切到 `deadline_*`
- **统计/计数/"有哪些"要翻完所有页**：`--page-count` 取足够大直到 `has_more=false`；结果过大被转存到文件时要读完整再统计
- **定位待办用哪个状态**：修改通常查进行中即可；完成、删除/退出时传 `["finished","proceed"]`，避免把已完成误判为未找到

---

## 2. todo get — 批量取详情

```bash
wecom-cli todo get --json '{"items": [{"todo_id": "td_xxx"}, {"todo_id": "td_yyy"}]}'
```

- `items` 必填，每项 `{todo_id}`，单次最多 20 个，超出分批
- 返回 `items[]` 与入参一一对应：`{success, todo_id, title, description, status, user_status, creator, followers, deadline, extra_info, source, create_time, update_time, errmsg}`
- 已有 `todo_id` 想确认最新状态时用它

---

## 3. todo create — 创建待办

```bash
wecom-cli todo create --json '{"items": [{"title": "准备周会材料", "description": "本周三上午周会需要的销售数据 PPT", "follower_ids": ["wo_xxx"], "deadline": {"type": "datetime", "value": "2026-05-13 09:00:00"}, "remind_at_deadline": true}]}'
```

### 入参

外层 `{items: []}`，单次最多 20 条。

`items[]`：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `title` | string | ✅ | 短标题，长度 ≥ 1 |
| `description` | string | ❌ | 详细描述（标题之外的补充说明） |
| `follower_ids` | string[] | ❌ | 分派人 `userid` 列表（前缀 `wo`），最多 50 人 |
| `deadline` | object | ❌ | 截止时间，见上文规范 |
| `remind_at_deadline` | boolean | ❌ | 提醒时机，须与 `deadline` 同传 |

> 注意创建用 `follower_ids`（扁平数组），更新用 `followers`（对象数组），**两者形状不同**，别混用。

### 返回

`items[]`：`{success, todo_id, title, followers[]{userid,user_name}, extra_info, errmsg}`

### 字段推断规则

- **`title`**：从用户原话提炼，优先"动宾"结构。只有当用户完全没讲事情本身（"帮我记个待办"）时才追问
- **`description`**：只在有标题装不下的额外信息时才传。**禁止写成与 `title` 相同或复述**；没额外信息就不传
- **`follower_ids`**：用户说"分派给我"/"我和某某一起"时，要把当前用户自己的 `userid` 也放进去（后台不会自动把创建者算作分派人）；只给自己建且无其他人时不用放
- **`deadline` / `remind_at_deadline`**：
  - 只说截止/到期时间 → 只填 `deadline`
  - 明确要"某时刻提醒" → 该时刻作 `deadline.type=datetime`，并传 `remind_at_deadline=true`
  - 只给日期 → `type=date`，不传 `remind_at_deadline=true`
  - "xx 截止并提前 yy 提醒" → `deadline` 永远填 **xx**，不要填提前后的时刻；当前入参不能直接设"提前 yy"，创建后按 `extra_info` 判断是否刚好满足
  - 完全没提时间 → 两个都不传，也不追问

### 回显

成功后回显 **标题、参与人、截止时间**（不存在的项缺省，不要硬写"无"）：

> 已创建待办「准备周会材料」，参与人：张三、李四，截止时间：2026-05-13 09:00:00。

参与人取 `followers[].user_name`，只用顿号拼接人名，不出现 `userid`。截止时间取**本次入参**的 `deadline.value`（返回体不回传它）。批量创建逐条回显。

### 提醒说明

传了 `remind_at_deadline=true` 或用户提到提醒诉求时，必须在回显后附提醒说明：

- 用户要求"提前 X 提醒"→ 核对 `extra_info` 是不是截止时间提前 X 的时刻；匹配则说明已满足，否则固定话术：`目前不支持直接创建您需要的提醒时间，已为您设置截止时间为 XX，请到企业微信待办功能中手动修改提醒时间。`
- 用户要求"截止时提醒"→ 只有 `datetime` 才应传 `true`；`extra_info` 不等于 `deadline.value` 或缺失时，仍引导用户到企业微信待办里改
- 有 `extra_info`（非"提前 X"场景）→ 引用其中的时刻
- 无 `extra_info` → 说明返回未确认提醒时间，引导用户去待办应用检查
- **不要另建定时任务模拟待办提醒**，会重复提醒

仅带 `deadline`、未要求提醒的普通待办，不需要额外说明。

---

## 4. todo update — 修改待办

```bash
wecom-cli todo update --json '{"items": [{"todo_id": "td_xxx", "title": "调整后的周会材料", "deadline": {"type": "datetime", "value": "2026-05-13 09:00:00"}, "remind_at_deadline": true}]}'
```

### 入参

外层 `{items: []}`，单次最多 20 条。`items[]` 只传需修改的字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `todo_id` | string | ✅ 待办 ID |
| `title` | string | 新标题 |
| `description` | string | 新描述 |
| `followers` | object[] | **全量替换**后的分派人，每项只接受 `{userid}`，最多 50 人 |
| `deadline` | object | 新截止时间；传 `{}` 表示清空；不传则保持 |
| `remind_at_deadline` | boolean | 须与 `deadline` 同传 |

- 入参 `followers` 子对象**只接收 `userid`**；从 `list`/`get` 拿到的 `user_name` / `user_status` / `update_time` 要全部剥掉
- **`followers` 是全量替换不是增量**：只加/减部分人时，先取现有名单，本地合并或删减，再整体提交
- 说"把我也加进去"时同样要带上当前用户自己的 `userid`
- 用户要"取消提醒/别提醒了"→ 直接告知当前不支持关闭待办提醒；若坚持完全不提醒，唯一办法是连截止时间一起清空（`deadline: {}`），须先确认再操作

### 使用规则

- 上下文没有 `todo_id` 时，先用 `todo list` 定位
- **避免冗余更新**：如果用户只是复述已记录的内容（标题已相同），这是确认不是修改，直接回复"这条已经记好了"；尤其不要把 `description` 更新成与 `title` 相同
- 提醒说明的要求与 `create` 完全一致

---

## 5. todo delete — 删除 / 退出待办

```bash
wecom-cli todo delete --json '{"items": [{"todo_id": "td_xxx"}, {"todo_id": "td_yyy"}]}'
```

语义取决于当前用户身份：

| 当前用户 | 语义 |
|----------|------|
| 是创建人 | 删除整条待办，其他参与人也不再看到 |
| 不是创建人 | 当前用户**退出待办 / 从自己的列表移除**，不影响其他人 |

- **非创建人也可以调用**，不要因为 `creator.userid` 不是当前用户就拒绝，也不要回复"创建人之外无权删除"
- 调用前核对创建人只是为了理解语义和做幂等，不是权限门
- 用户说某待办"已完成"时默认是完成操作，**不等于删除**；只有明确说删除才调本接口
- 定位时 `status_filter` 传 `["finished","proceed"]`，否则可能找不到
- 返回 `items[]`：`{success, todo_id, title, errmsg}`
- 删除前应向用户确认

---

## 6. todo finish — 完成待办

```bash
wecom-cli todo finish --json '{"items": [{"todo_id": "td_xxx", "finished_all": false}]}'
```

把**当前用户**在该待办中的部分标记完成。

| 字段 | 类型 | 说明 |
|------|------|------|
| `todo_id` | string | ✅ 待办 ID |
| `finished_all` | boolean | 创建人可设为 `true` 全部完成；默认 `false` 仅完成自己的部分 |

### 何时传 `finished_all`

- 用户明确"仅我完成"（"我这边搞完了"、"我那块先标了"）→ **显式**传 `false`，让后端跳过 `ask_finish_all` 兜底
- 用户明确"全部完成"（"完成了"、"这条结掉"），或本会话此前对同一 `todo_id` 已调过一次 `finished_all=false`、用户又说要完成 → 传 `true`
- 表达不明确（只说"把那条待办完成了"）→ **不传**，让后端返回 `ask_finish_all` 再问

### `ask_finish_all` 处理

返回里出现 `ask_finish_all` 说明当前用户是创建人，且第一次调用已把本人部分标记完成。**必须**用自然语言确认完成范围，列出「仅我完成」「已完全完成」两个选项，文字中包含待办标题和参与人中文名（顿号拼接）：

```
待办「<待办标题>」中您的部分已完成。参与人：<参与人中文名>。请选择完成范围：仅我完成，还是已完全完成？
```

- 选「仅我完成」→ 不再调接口，告知已标记完成
- 选「已完全完成」→ 用同一 `todo_id` 再调一次，传 `finished_all: true`

### 幂等

定位时若发现整条 `status=finished` 或当前用户 `user_status=finished`，说明已完成，直接告知"这条待办已完成"，不要再调 `finish`。只有用户明确要求"完成后删除"时才继续走删除流程。

返回 `items[]`：`{success, todo_id, title, ask_finish_all, errmsg}`

---

## 注意事项

1. **`todo_id` 来源**：必须来自接口返回，禁止推测构造；匹配到多个相似待办时展示候选让用户确认；**禁止展示给用户**
2. **人员查询**：`userid` 必须通过 `wecom-cli contact users search` 获取，禁止按姓名猜测；同名时展示候选让用户选
3. **人名优先用返回里的 `user_name`**：`list` / `get` / `create` 都已附带，展示前不必再查通讯录
4. **时间格式**：`YYYY-MM-DD HH:mm:ss`；用户说"明天""下周一"时按当前日期推算
5. **展示格式**：查列表时按 `status` 分组，每条含标题、创建人（是自己则省略）、参与人、截止时间；分组内按 `deadline.value` 升序、无截止时间排最后、同一时间按 `update_time` 倒序
   - 若 `list` 是为了定位 `todo_id` 而内部调用，**不要**把列表展示给用户
6. **分页**：`has_more` 为 `true` 时必须告诉用户"还有更多待办未显示，是否继续查看"
7. **错误处理**：返回 `success=false` 时展示 `errmsg`；`extra_info` 非空时如实转告
8. **破坏性操作确认**：删除待办、清空截止时间前先确认

完整流程示例见 [wecom-todo-workflows.md](wecom-todo-workflows.md)。

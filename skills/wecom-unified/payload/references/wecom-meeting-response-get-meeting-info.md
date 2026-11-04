# meeting get 返回参数

`wecom-cli meeting get --json '{"meeting_ids": [{"meeting_id": "mtXXXX"}]}'` 的完整返回结构。

> ⚠️ 旧版返回的是**单个会议**对象：`title`、`meeting_start_datetime`、`meeting_duration`、`status`（1~5 数字）、`meeting_type`（0/1/2/3/5/6 数字）、`attendees.member[]`、`settings{}`、`sub_meetings[]`、`sub_repeat_list[]`。**这些全部不存在了**。现在返回的是 `meetings[]` 数组，字段名和枚举值都换了。

## 顶层

```json
{
  "meetings": [ /* ... */ ],
  "meetings_count": 1
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `meetings` | object[] | 会议详情列表，最多 10 条（一次请求的上限） |
| `meetings_count` | integer | 上面数组的元素个数 |

## meetings[]

```json
{
  "meeting_id": "mtXXXX",
  "sub_meeting_id": "",
  "subject": "产品需求评审",
  "begin_time": "2026-11-02 14:00:00",
  "end_time": "2026-11-02 15:00:00",
  "meeting_status": "init",
  "location": "3 楼会议室",
  "meeting_room": "3F-大会议室",
  "description": "评审 Q4 需求文档",
  "timezone": {"timezone_id": "Asia/Shanghai", "timezone_offset": 28800},
  "attendees": [
    {
      "userid": "woAAA",
      "name": "张三",
      "is_external": false,
      "is_attended": false,
      "duration": 0
    }
  ],
  "repeat_rule": null,
  "notes": [],
  "note_url": "",
  "record_url": "",
  "has_note_permission": false,
  "current_user_enter_time": "",
  "current_user_quit_time": "",
  "is_except_meet": false
}
```

### 基础字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `meeting_id` | string | 会议 ID，`mt` 前缀长字符串，**不是** 9 位会议号 |
| `sub_meeting_id` | string | 子会议 ID，周期会议改/查单场时才有 |
| `subject` | string | 会议主题 |
| `begin_time` / `end_time` | string | `YYYY-MM-DD HH:MM:SS` |
| `meeting_status` | string | `init` 未开始 / `started` 进行中 / `end` 已结束（终止态，不回退） |
| `location` | string | 会议地点（文本） |
| `meeting_room` | string | 会议室名称，没订会议室为空 |
| `description` | string | 备注 |
| `timezone` | object | `{timezone_id, timezone_offset}` |

### 参会人 attendees[]

| 字段 | 类型 | 说明 |
|------|------|------|
| `userid` | string | 内部成员 `wo` 前缀、外部联系人 `wm` 前缀；**不要展示** |
| `name` | string | 展示名，格式如 `zhangsan(张三)`，**原样展示** |
| `is_external` | boolean | 是否外部联系人（含跨企业成员） |
| `is_attended` | boolean | 是否已入会 |
| `duration` | integer | 参会时长（秒） |

`name` 为空时用 `userid` 走 `contact users search` 反查，仍查不到再告知用户该参会人信息暂不可得，**禁止直接展示 userid**。

### 周期规则 repeat_rule

非周期会议为 `null`/空。有值时：

| 字段 | 说明 |
|------|------|
| `repeat_type` | `daily` / `weekday` / `weekly` / `biweekly` / `monthly` |
| `repeat_days` | 重复天数 |
| `repeat_step` | 每 N（天/周/月）一次，与 `repeat_type` 配合 |
| `until_type` | `by_date` / `by_times` |
| `until_date` | 按日期结束时的结束时刻 |
| `until_times` | 按次结束时的总场次 |
| `first_begin_time` / `first_end_time` | 首场时间 |
| `version` | 规则版本，默认 0 |

> 注意：**读**出来的 `repeat_type` 可达 5 种（含 `biweekly`），但**建**会议时 `recurrence.repeat_type` 只接受 4 种（`daily` / `weekly` / `monthly` / `work_day`）。读到 `biweekly` 的会议不能原样拿去重建。

另外还有 `repeat_rule_list`（多段周期规则 + 例外场次），返回条件与 `repeat_rule` 相同，普通展示用不到。

### 纪要 notes[]

| 字段 | 说明 |
|------|------|
| `note_content` | 智能纪要文字内容 |
| `todo_content` | 纪要里提取的待办 |

每个媒体房间一条，最多 10 条。`has_note_permission == false` 时**不要展示** `notes` 内容。

### 链接

| 字段 | 说明 |
|------|------|
| `note_url` | 智能纪要链接 |
| `record_url` | 录制 / 回放链接 |

**默认不输出。** 只有用户明确问「纪要链接 / 录制 / 回放」时才给，且必须写成 markdown 链接 `[会议主题](url)`——`[]` 里放 `subject`，禁止裸贴 URL、禁止用「点击查看」这类固定文案。

### 其它

| 字段 | 说明 |
|------|------|
| `current_user_enter_time` | 当前用户最早入会时间；**仅会议结束后有值**，未入会为空 |
| `current_user_quit_time` | 当前用户最晚离会时间；同上 |
| `is_except_meet` | 是否周期会议中被单独修改的那一场 |

## 旧字段对照

| 旧字段（0.1.x） | 现在 |
|------------------|------|
| `title` | `subject` |
| `meeting_start_datetime` + `meeting_duration` | `begin_time` + `end_time` |
| `status`：1 待开始 / 2 会议中 / 3 已结束 / 4 已取消 / 5 已过期 | `meeting_status`：`init` / `started` / `end`（**没有「已取消」和「已过期」两种状态**） |
| `meeting_type`：0 一次性 / 1 周期 / 2 微信专属 / 3 Rooms / 5 个人会议号 / 6 网络研讨会 | ❌ 无对应字段；周期判断看 `repeat_rule` 是否为空 |
| `meeting_code` / `meeting_link` | ❌ `meeting get` **不再返回**，只有 `meeting create` 返回一次 |
| `attendees.member[].status`：1 已参与 / 2 未参与 | `attendees[].is_attended` 布尔 |
| `settings{}` | ❌ 整体删除 |
| `sub_meetings[]` / `sub_repeat_list[]` / `current_sub_meetingid` | 用 `sub_meeting_id` 单指，或 `repeat_rule_list` |
| `has_more_sub_meeting` / `remain_sub_meetings` | ❌ 无对应字段 |

## 用法要点

- 一次最多传 10 个 `meeting_id`，超了分批
- `meeting_ids` 是**对象数组** `[{"meeting_id": "..."}]`；`sub_meeting_id` 只在周期会议指定单场时传，不传不返回子会议信息
- 拿链接只有创建那一刻（`meeting create` 返回 `meeting_link` / `meeting_code`）；事后 `get` 拿不到，只能拿 `note_url` / `record_url`

# calendar schedules create API

创建新日程，支持设置标题、时间、地点、参与者、提醒和重复规则。

## 命令

```bash
wecom-cli calendar schedules create --json '<JSON 参数>'
```

> 注意：请求体是**平的**，旧版那个 `{"schedule": {...}}` 包装层已经废弃。

## 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `subject` | string | ✅ | 日程标题 |
| `begin_time` | string | ✅ | 开始时间，`YYYY-MM-DD HH:MM:SS` |
| `end_time` | string | ✅ | 结束时间，`YYYY-MM-DD HH:MM:SS` |
| `description` | string | ❌ | 日程备注 |
| `location` | string | ❌ | 地点 |
| `is_all_day` | boolean | ❌ | 是否全天，默认 `false` |
| `attendees` | array | ❌ | 参与者列表，每项含 `userid` |
| `mark_optional_attendees` | string[] | ❌ | 标记为可选的参与人 `userid` |
| `allow_self_join` | boolean | ❌ | 是否允许主动加入，默认 `true` |
| `meeting_room_id` | string | ❌ | 会议室 ID，传入即预定会议室 |
| `reminders` | object | ❌ | 提醒设置（见 [reminders 字段参考](wecom-schedule-ref-reminders.md)） |
| `repeat_rule` | object | ❌ | 周期规则；`is_repeat=false` 即非周期日程 |
| `timezone` | object | ❌ | `{"timezone_id": "Asia/Shanghai"}` |

## 请求示例

```bash
wecom-cli calendar schedules create --json '{
  "subject": "需求评审",
  "begin_time": "2026-03-12 14:00:00",
  "end_time": "2026-03-12 15:00:00",
  "attendees": [{"userid": "USER_ID"}],
  "location": "3 楼会议室",
  "reminders": {"is_remind": true, "reminder_time": [-900]}
}'
```

## 返回字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `schedule_id` | string | 创建成功的日程 ID |

## 响应示例

```json
{
    "schedule_id": "SCHEDULE_ID"
}
```

## 注意

- 加会议室要传 `meeting_room_id`，先用 `meeting rooms search` 查（见 [wecom-meeting.md](wecom-meeting.md)）
- 创建成功后把 `schedule_id` 留在上下文里，后续改/取消都要用它；但**不要**把它展示给用户

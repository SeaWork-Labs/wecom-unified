# calendar schedules update API

修改已有日程，Patch 语义：只需传入需要修改的字段，未传字段保持不变。

## 命令

```bash
wecom-cli calendar schedules update --json '<JSON 参数>'
```

## 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `schedule_id` | string | ✅ | 目标日程 ID |
| `subject` | string | ❌ | 新标题 |
| `begin_time` | string | ❌ | 新开始时间 |
| `end_time` | string | ❌ | 新结束时间 |
| `description` | string | ❌ | 新备注，**传空字符串表示清空** |
| `location` | string | ❌ | 新地点，**传空字符串表示清空** |
| `is_all_day` | boolean | ❌ | 是否全天 |
| `add_attendees` | array | ❌ | 新增的参与人，每项含 `userid` |
| `remove_attendees` | array | ❌ | 移除的参与人，每项含 `userid` |
| `meeting_room_id` | string | ❌ | 会议室 ID |
| `allow_self_join` | boolean | ❌ | 是否允许主动加入 |
| `repeat_mod` | object | ❌ | 周期编辑规则，周期日程必填 |

`repeat_mod` 结构：

```json
{"repeat_edit_type": "this_only", "repeat_event_time": "YYYY-MM-DD HH:MM:SS"}
```

`repeat_edit_type` 取值：`this_only`（仅此场次）、`this_and_future`（此场次及将来）、`all`（所有）。

## 请求示例

改标题与时间：

```bash
wecom-cli calendar schedules update --json '{"schedule_id": "SCHEDULE_ID", "subject": "更新后的标题", "begin_time": "2026-03-13 15:00:00", "end_time": "2026-03-13 16:00:00"}'
```

加人：

```bash
wecom-cli calendar schedules update --json '{"schedule_id": "SCHEDULE_ID", "add_attendees": [{"userid": "USER_ID"}]}'
```

移除人：

```bash
wecom-cli calendar schedules update --json '{"schedule_id": "SCHEDULE_ID", "remove_attendees": [{"userid": "USER_ID"}]}'
```

## 返回字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `detail` | object | 更新后的日程详情 |

## 注意

- 加/移除参与人**没有独立方法**，就是本接口的 `add_attendees` / `remove_attendees`
- 修改周期日程的时间时必须带 `repeat_mod`，否则会报错

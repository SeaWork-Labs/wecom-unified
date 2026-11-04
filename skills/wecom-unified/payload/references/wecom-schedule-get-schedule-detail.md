# calendar schedules get API

通过日程 ID 批量获取日程详细信息。

## 命令

```bash
wecom-cli calendar schedules get --json '<JSON 参数>'
```

## 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `schedule_ids` | array | ✅ | 日程 ID 列表，1~50 个 |

## 请求示例

```bash
wecom-cli calendar schedules get --json '{"schedule_ids": ["SCHEDULE_ID_1", "SCHEDULE_ID_2"]}'
```

## 返回字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `schedule_list` | array | 日程详情列表 |
| `schedule_list_count` | integer | 数组元素数量 |

### schedule_list[] 字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `schedule_id` | string | 日程唯一 ID |
| `subject` | string | 日程标题 |
| `description` | string | 日程备注 |
| `begin_time` | string | 开始时间，`YYYY-MM-DD HH:MM:SS` |
| `end_time` | string | 结束时间，`YYYY-MM-DD HH:MM:SS` |
| `location` | string | 地点 |
| `is_all_day` | boolean | 是否全天 |
| `attendees` | array | 参与者列表 |
| `attendees[].userid` | string | 参与者 userid |
| `attendees[].name` | string | 参与者姓名（接口已附带） |
| `calendar_name` | string | 所属日历本名称 |
| `creator_name` | string | 创建者姓名 |
| `is_share_cal` | boolean | 是否为共享日历（非本人创建） |
| `meeting` | object | 关联的腾讯会议：`meeting_id` / `meeting_code` / `meeting_link` |
| `meeting_room` | object | 会议室：`meeting_room_id` / `meeting_room_name` |
| `reminders` | object | 提醒设置 |
| `repeat_rule` | object | 周期规则 |
| `timezone` | object | 时区 |

> ✅ `begin_time` / `end_time` 已经是可读字符串；`attendees[].name` 也已附带，展示时不必再查通讯录。

## 响应示例

```json
{
    "schedule_list": [
        {
            "schedule_id": "SCHEDULE_ID",
            "subject": "需求评审",
            "begin_time": "2026-03-12 14:00:00",
            "end_time": "2026-03-12 15:00:00",
            "location": "3 楼会议室",
            "is_all_day": false,
            "attendees": [{"userid": "USER_ID", "name": "张三"}],
            "reminders": {"is_remind": true, "reminder_time": [-900]}
        }
    ],
    "schedule_list_count": 1
}
```

## 注意

- 查询列表用 `calendar schedules list` 就够了：它返回的就是完整日程详情，不需要再按 ID 逐条查
- 只有手上已经有一批 `schedule_id`（例如用户从别处带过来的）时，才用本接口

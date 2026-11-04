# calendar schedules free list API

查询多名成员在指定时间范围内的共同空闲时段。

## 命令

```bash
wecom-cli calendar schedules free list --json '<JSON 参数>'
```

> 注意：旧版的 `schedule check_availability` + `check_user_list` 已不存在。现在是 `calendar schedules free list`，用户列表用 `userids`，返回的是**推荐空闲时段**而不是忙碌列表。

## 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `userids` | string[] | ✅ | 成员 userid 列表，1 个及以上（单人合法） |
| `begin_time` | string | ✅ | 查询窗口起，`YYYY-MM-DD HH:MM:SS` |
| `end_time` | string | ✅ | 查询窗口止，必须晚于 `begin_time` 且晚于当前时间 |
| `min_duration_minutes` | integer | ❌ | 最短时长（分钟），默认 30 |
| `limit` | integer | ❌ | 返回时段上限，默认 10 |
| `strategy` | string | ❌ | 默认 `max_attendees` |

## 请求示例

```bash
wecom-cli calendar schedules free list --json '{"userids": ["USER_ID_1", "USER_ID_2"], "begin_time": "2026-03-12 09:00:00", "end_time": "2026-03-12 18:00:00", "min_duration_minutes": 60}'
```

## 返回字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `slots` | array | 推荐时段列表 |
| `slots[].begin_time` | string | 时段起 |
| `slots[].end_time` | string | 时段止 |
| `slots[].available_count` | integer | 该时段空闲的成员数 |
| `slots[].available_users` | array | 空闲成员，每项含 `userid` / `name` |
| `slots[].busy_users` | array | 忙碌成员，每项含 `userid` / `name` |
| `slots_count` | integer | 时段数量 |
| `total_count` | integer | 入参 userid 总数 |
| `strategy` | string | 实际生效的策略 |
| `extra_info` | string | 降级提示文案，非空时如实转告用户 |

## 响应示例

```json
{
    "slots": [
        {
            "begin_time": "2026-03-12 14:00:00",
            "end_time": "2026-03-12 16:00:00",
            "available_count": 2,
            "available_users": [
                {"userid": "USER_ID_1", "name": "张三"},
                {"userid": "USER_ID_2", "name": "李四"}
            ],
            "busy_users": []
        }
    ],
    "slots_count": 1,
    "total_count": 2,
    "strategy": "max_attendees",
    "extra_info": ""
}
```

## 注意

- 展示时段时用 `name`，不要展示 `userid`
- `available_count < total_count` 的时段要注明谁冲突，别直接当成"都有空"
- `extra_info` 非空时一并转告用户（通常是降级说明）

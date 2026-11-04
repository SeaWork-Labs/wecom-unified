# reminders 字段参考

提醒设置对象，用于 `calendar schedules create` 和 `calendar schedules update` 接口。

## 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `is_remind` | boolean | 是否提醒 |
| `reminder_time` | integer[] | 提醒时间点，**负数秒**，表示提前多少秒提醒 |

常用 `reminder_time` 取值：

| 含义 | 值 |
|------|-----|
| 会议开始时 | `0` |
| 提前 5 分钟 | `-300` |
| 提前 15 分钟 | `-900` |
| 提前 1 小时 | `-3600` |
| 提前 1 天 | `-86400` |

可多项同时传，实现多次提醒。

> 旧版的 `remind_before_event_secs`（正数秒）+ `remind_time_diffs` + 整数 `timezone` 这套结构已经废弃，现在只有 `is_remind` + `reminder_time`。

## 使用示例

### 基本提醒（提前 1 小时）

```json
{
    "is_remind": true,
    "reminder_time": [-3600]
}
```

### 多次提醒（提前 1 天 + 提前 15 分钟）

```json
{
    "is_remind": true,
    "reminder_time": [-86400, -900]
}
```

### 不提醒

```json
{
    "is_remind": false
}
```

## 注意

- 只在用户明确表达提醒意图时传本字段；用户没提就不要自行补默认提醒
- 时区不在 `reminders` 里，改用日程顶层的 `timezone: {"timezone_id": "Asia/Shanghai"}`

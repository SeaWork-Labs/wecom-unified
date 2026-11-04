# 创建会议 — 提醒 / 响铃（已不支持）

> ⚠️ **旧版的会议提醒设置整体不存在了。** `settings.remind_scope`（提醒范围：1 不提醒 / 2 仅主持人 / 3 全体 / 4 指定人）和 `settings.ring_users`（响铃用户）这两个字段在 `@wecom/cli` 1.3.x 的 `meeting.create` schema 里没有任何对应项——`meeting create` 的完整入参只有 `subject`、`begin_time`、`end_time`、`attendees`、`mark_optional_attendees`、`location`、`meeting_room_id`、`description`、`timezone`、`cal_id`、`recurrence`。

## 用户提「只提醒主持人 / 只让某几个人响铃」时怎么办

不要伪造参数，也不要写一个看起来像的字段名——那样只会被 schema 拒绝或静默忽略。正确做法是明确告知：

> 通过接口创建会议时不支持设置参会提醒范围或指定响铃人，需要你在企业微信客户端的会议详情里修改提醒设置。

## 能用的是什么

会议本身没有提醒参数，但**日程**有 `reminders` 字段。如果用户要的其实是「到点提醒我」，可以改用日程：

```bash
wecom-cli calendar schedules create --json '{
  "subject": "紧急故障复盘",
  "begin_time": "2026-11-03 20:00:00",
  "end_time": "2026-11-03 21:00:00",
  "attendees": [{"userid": "woAAA"}, {"userid": "woBBB"}],
  "reminders": {"is_remind": true, "reminder_time": [-900]}
}'
```

`reminder_time` 是**负数秒**，表示「开始前多少秒提醒」；`-900` 就是提前 15 分钟。传多个值就是多个提醒点：

```json
"reminders": {"is_remind": true, "reminder_time": [-300, -1800]}
```

关闭提醒用 `{"is_remind": false}`。详见 [wecom-schedule-ref-reminders.md](wecom-schedule-ref-reminders.md)。

> 注意：这是**日程**的提醒，不是会议的。会议仍然会在创建时顺带生成一条关联日程，但那条日程的提醒设置接口改不到（`meeting create` 不返回它的 `schedule_id`）。所以「会议 + 自定义提醒」这个组合目前只能二分：要么建会议（无自定义提醒），要么建日程（有提醒、无入会链接）。

## 旧示例（不可用，仅供识别用户描述）

旧文档里的这两种说法都属于**已删除能力**：

- 「只提醒主持人，其他人不要响铃」→ `settings.remind_scope = 2`
- 「只响铃提醒张三和李四，其他人不提醒」→ `settings.remind_scope = 4` + `settings.ring_users.userid = ["zhangsan","lisi"]`

听到这类诉求时，按上面的话术告知不支持，不要尝试构造等价参数。

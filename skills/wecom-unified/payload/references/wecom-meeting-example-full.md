# 创建会议 — 全参数示例

> ⚠️ **旧版的 `settings` 对象在 1.3.x 上整体不存在**。`password`、`enable_waiting_room`、`allow_enter_before_host`、`enable_enter_mute`、`allow_external_user`、`enable_screen_watermark`、`remind_scope`、`ring_users` 这八个字段在全部 168 个接口 schema 里都查不到，传了不会生效（会被 schema 校验拦掉或忽略）。这些安全设置目前只能到企业微信客户端里改。

## 场景：把一个会议的所有可用字段都用上

**用户意图**：「下周一上午 9 点开 Q4 战略会，4 小时，叫上张三李四王五，地点总部大会议室，备注写清楚」

```bash
wecom-cli meeting create --json '{
  "subject": "Q4 季度战略规划会",
  "begin_time": "2026-11-09 09:00:00",
  "end_time": "2026-11-09 13:00:00",
  "attendees": [
    {"userid": "woAAA"},
    {"userid": "woBBB"},
    {"userid": "woCCC"}
  ],
  "location": "总部大会议室",
  "description": "请各部门负责人提前准备汇报材料",
  "timezone": {"timezone_id": "Asia/Shanghai", "timezone_offset": 28800}
}'
```

## 字段对照：可用 / 已删

| 旧字段（0.1.x，已失效） | 现在怎么写 |
|--------------------------|-----------|
| `title` | `subject` |
| `meeting_start_datetime`（`YYYY-MM-DD HH:mm`） | `begin_time`（`YYYY-MM-DD HH:MM:SS`） |
| `meeting_duration`（秒） | 换算成 `end_time` = `begin_time` + 时长 |
| `invitees: {"userid": ["a","b"]}` | `attendees: [{"userid":"a"},{"userid":"b"}]` |
| `settings.password` | ❌ 无对应字段 |
| `settings.enable_waiting_room` | ❌ 无对应字段 |
| `settings.allow_enter_before_host` | ❌ 无对应字段 |
| `settings.enable_enter_mute` | ❌ 无对应字段 |
| `settings.allow_external_user` | ❌ 无对应字段 |
| `settings.enable_screen_watermark` | ❌ 无对应字段 |
| `settings.remind_scope` / `settings.ring_users` | ❌ 无对应字段（见 [wecom-meeting-example-reminder.md](wecom-meeting-example-reminder.md)） |

## 新增的可用字段

| 字段 | 用法 |
|------|------|
| `mark_optional_attendees` | 把部分参会人标成「可选」：`["woAAA"]`（平铺字符串数组，且必须同时出现在 `attendees` 里） |
| `meeting_room_id` | 订会议室。必须先 `meeting rooms search` 查到真实 ID |
| `recurrence` | 周期规则，见 [wecom-meeting.md](wecom-meeting.md#周期规则-recurrence) |
| `cal_id` | 指定所属日历本 |

## 带可选参会人的例子

```bash
wecom-cli meeting create --json '{
  "subject": "Q4 战略会",
  "begin_time": "2026-11-09 09:00:00",
  "end_time": "2026-11-09 13:00:00",
  "attendees": [{"userid": "woAAA"}, {"userid": "woBBB"}, {"userid": "woCCC"}],
  "mark_optional_attendees": ["woCCC"]
}'
```

## 展示

创建成功后只输出主题、时间、参会人，不输出会议号、入会链接、`meeting_id`、会议室 ID：

```
主题：Q4 季度战略规划会
时间：2026-11-09 09:00:00 - 13:00:00
参会人：张三、李四、王五
```

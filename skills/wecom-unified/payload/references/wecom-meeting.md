# 企业微信会议

> 公共概念与规则请参考 [wecom-shared.md](wecom-shared.md)

会议是**独立服务** `meeting`，与 `calendar`（日程）并列。创建会议会同时生成一条关联日程；取消会议会连带取消那条日程（不要再对同一场调 `calendar schedules cancel`）。

## 接口列表

| 方法 | 用途 |
|------|------|
| `meeting create` | 创建会议（可带周期规则、可订会议室） |
| `meeting list` | 按时间范围拉会议列表 |
| `meeting get` | 按 ID 批量取详情（含参会人、状态、纪要） |
| `meeting search` | 按关键词搜会议 |
| `meeting update` | 改主题 / 时间 / 参会人 / 地点 / 会议室 |
| `meeting cancel` | 取消会议 |
| `meeting original get` | 拉会议转写原文 |
| `meeting rooms buildings list` | 查企业办公大楼 |
| `meeting rooms search` | 查会议室 |

调用格式：

```bash
wecom-cli meeting <方法> --json '<JSON 入参>'
```

> ⚠️ 旧版的 `meeting create_meeting` / `list_user_meetings` / `get_meeting_info` / `cancel_meeting` / `set_invite_meeting_members` 在 1.3.x 上**全部不存在**，会报 `error: unrecognized subcommand`。字段也整体换了：`title` → `subject`、`meeting_start_datetime` + `meeting_duration` → `begin_time` + `end_time`、`invitees` → `attendees`、`meetingid` → `meeting_id`。

---

## meeting create — 创建会议

```bash
wecom-cli meeting create --json '{
  "subject": "产品需求评审",
  "begin_time": "2026-11-02 14:00:00",
  "end_time": "2026-11-02 15:00:00",
  "attendees": [{"userid": "woXXXX"}],
  "description": "评审 Q4 需求文档"
}'
```

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `subject` | string | ✅ | 会议标题，1~255 字节 |
| `begin_time` | string | ✅ | `YYYY-MM-DD HH:MM:SS`，必须晚于当前时间 |
| `end_time` | string | ✅ | `YYYY-MM-DD HH:MM:SS`，必须晚于 `begin_time` |
| `attendees` | object[] | ❌ | `[{"userid": "woXXX"}]`，最多 300 人 |
| `mark_optional_attendees` | string[] | ❌ | 可选参会人的 userid 平铺数组 `["woXXX"]`；必须在 `attendees` 里同时存在 |
| `location` | string | ❌ | 地点文本，最长 128 字节 |
| `meeting_room_id` | string | ❌ | 会议室 ID，传入即触发「建会议 + 占会议室」 |
| `description` | string | ❌ | 备注，最长 500 字节 |
| `timezone` | object | ❌ | `{"timezone_id": "Asia/Shanghai", "timezone_offset": 28800}`；不传默认东八区 |
| `cal_id` | string | ❌ | 所属日历本 ID |
| `recurrence` | object | ❌ | 周期规则，见下；不传即单次会议 |

### 返回

```json
{
  "meeting_id": "mtXXXX",
  "meeting_code": "123456789",
  "meeting_link": "https://meeting.tencent.com/dm/xxxx"
}
```

> `meeting_id` 是长字符串（`mt` 前缀），**不是** 9 位会议号。后续 `get` / `update` / `cancel` 都要用它。

### 周期规则 recurrence

```json
{
  "is_repeat": true,
  "repeat_type": "weekly",
  "repeat_day_of_week": ["MO", "WE"],
  "repeat_time": 10
}
```

| 字段 | 说明 |
|------|------|
| `is_repeat` | `true` 才建周期会议；不传或 `false` 建单次 |
| `repeat_type` | 只能是 `daily` / `weekly` / `monthly` / `work_day` |
| `repeat_day_of_week` | `weekly` 用，`["MO","TU","WE","TH","FR","SA","SU"]`，最多 7 个 |
| `repeat_day_of_month` | `monthly` 用，`1~31`，最多 31 个 |
| `repeat_interval` | 每 N 天/周/月一次，默认 1；**需要同时传 `is_custom: true` 才生效** |
| `repeat_until` | 按日期结束（`YYYY-MM-DD HH:MM:SS`） |
| `repeat_time` | 按场次结束，含首场，1~200 |
| `is_custom` | 是否用自定义间隔 |

`repeat_until` 与 `repeat_time` 二选一，至少给一个——**不支持永不结束的周期会议**。

> **不支持的周期类型**：`monthly_on_the_nth_day`（每月第 N 个星期几）和 `yearly`。底层 CGI 会无条件拒绝，传了必然失败——用户要这种周期时直接说明不支持，引导在企业微信客户端建。

### 会议室

用户说「订会议室 / 在 1605 开 / 找间会议室」时：

1. `meeting rooms buildings list` 确认楼栋（用户没提楼可跳过，后端用当前楼兜底）
2. `meeting rooms search` 查真实会议室（带 `begin_time` / `end_time`，可加 `room_keyword`、`floor_name`、`min_capacity`）
3. 从 `target[]`（用户点名了具体会议室）或 `recommendations[]`（没点名）里选 `status` 可用的 `room.meeting_room_id`
4. 把 `meeting_room_id` 传进 `create`，**不要再把会议室名写进 `location`**——会议室名由后端关联返回

**禁止**把会议室名只写进 `location`（那样不会真正占用会议室），**禁止**凭记忆编造 `meeting_room_id`。多个候选必须让用户选，不能静默替用户挑。

### 展示规则

创建成功后只输出**主题、时间、参会人**三部分；不要输出 `meeting_id`、会议室 ID、会议号、入会链接，也不要加寒暄。参会人取 `attendees[].name` 原样展示，不暴露 userid。

---

## meeting list — 按时间范围拉列表

```bash
wecom-cli meeting list --json '{"begin_time": "2026-11-01 00:00:00", "end_time": "2026-11-07 23:59:59", "limit": 100}'
```

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `begin_time` | string | ❌ | `YYYY-MM-DD HH:MM:SS`；与 `end_time` 要么都给要么都不给 |
| `end_time` | string | ❌ | 同上 |
| `cursor` | string | ❌ | 上一页的 `next_cursor` |
| `limit` | integer | ❌ | 每页条数，默认 20，上限 100 |

> 两个时间都不传时默认**当前时间起 30 天**。

### 返回

返回的是**两个列表**，结构各自相同：

| 字段 | 说明 |
|------|------|
| `created_meetings[]` | 当前用户创建的会议 |
| `attended_meetings[]` | 当前用户参加但非创建的会议（多一个 `creator_name`） |
| `created_meetings_count` / `attended_meetings_count` | 各自条数 |
| `next_cursor` | `has_more=true` 时有效 |
| `has_more` | 是否还有下一页 |

数组元素：`meeting_id`、`sub_meeting_id`、`subject`、`begin_time`、`end_time`、`attendee_count`、`meeting_room`、`location`、`is_repeat_meeting`、`timezone`、`creator_name`、`attendees[]`。

> `has_more == true` 就必须带 `next_cursor` 继续翻，否则会漏数据。

---

## meeting get — 批量取详情

```bash
wecom-cli meeting get --json '{"meeting_ids": [{"meeting_id": "mtXXXX"}]}'
```

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `meeting_ids` | object[] | 二选一 | **必须是对象数组**，每项 `{"meeting_id": "...", "sub_meeting_id": "..."}`；1~10 个 |
| `urls` | string[] | 二选一 | 直接传会议 URL，最多 10 个 |

`meeting_ids` **不能简化成字符串数组**——`["id1","id2"]` 会校验失败。超过 10 个要分批（10 + 10 + 5…）。

### 返回

完整字段清单见 [wecom-meeting-response-get-meeting-info.md](wecom-meeting-response-get-meeting-info.md)。核心字段：

| 字段 | 说明 |
|------|------|
| `meetings[].subject` | 会议主题 |
| `meetings[].begin_time` / `end_time` | 时间 |
| `meetings[].meeting_status` | `init` 未开始 / `started` 进行中 / `end` 已结束 |
| `meetings[].attendees[]` | 参会人，含 `userid` / `name` / `is_external` / `is_attended` / `duration` |
| `meetings[].location` / `meeting_room` | 地点 / 会议室名 |
| `meetings[].repeat_rule` | 周期规则，非周期为空 |
| `meetings[].notes[]` | 智能纪要 `note_content` + 待办 `todo_content` |
| `meetings[].note_url` / `record_url` | 纪要 / 录制链接 |

- `notes` 每场最多 10 条（每个媒体房间一条）；`has_note_permission == false` 时不展示纪要内容
- `note_url` / `record_url` **只在用户明确要链接时输出**，且必须写成 markdown 链接 `[会议主题](url)`，不要裸贴 URL
- 参会人展示取 `name`，不暴露 `userid`

---

## meeting update — 修改会议

```bash
wecom-cli meeting update --json '{"meeting_id": "mtXXXX", "subject": "评审（改期）", "begin_time": "2026-11-03 15:00:00", "end_time": "2026-11-03 16:00:00"}'
```

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `meeting_id` | string | ✅ | 目标会议 |
| `subject` | string | ❌ | 新标题，1~128 字节 |
| `begin_time` / `end_time` | string | ❌ | 新时间；要改就两个都给，且 `end_time` 必须晚于 `begin_time` |
| `add_attendees` | object[] | ❌ | `[{"userid": "woXXX"}]`，最多 100 |
| `remove_attendees` | object[] | ❌ | 同上 |
| `location` | string | ❌ | 新地点，最长 256 字节；传空字符串表示清空 |
| `description` | string | ❌ | 新备注，最长 5000 字节；传空字符串表示清空 |
| `meeting_room_id` | string | ❌ | 改订会议室 |
| `repeat_edit_type` | string | 周期会议必传 | `this_only` / `this_and_future` / `all` |
| `sub_meeting_id` | string | 周期会议必传 | 要改的那一场 |

- 只传要改的字段，未传的保持原值
- 参会人是**增量**（`add_attendees` / `remove_attendees`），不是全量覆盖——这点和旧版 `set_invite_meeting_members` 正好相反
- 周期会议改单场时：`repeat_edit_type: "this_only"` + `sub_meeting_id`
- 换会议室同样要先 `meeting rooms search` 拿到可用的 `meeting_room_id`，不能凭记忆传

### 返回

返回更新后的 `subject`、`begin_time`、`end_time`、`location`、`description`、`attendees[]`、`attendees_count`。

---

## meeting cancel — 取消会议

```bash
wecom-cli meeting cancel --json '{"meeting_id": "mtXXXX"}'
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `meeting_id` | string | ✅ | 目标会议 |
| `sub_meeting_id` | string | ❌ | 周期会议取消单场时传 |

- 成功返回空对象 `{}`，这是正常结果
- **不按「是否本人创建」提前拦截**——直接调，权限错误由接口返回
- 取消会议会连带取消关联日程，**不要再对同一场调 `calendar schedules cancel`**
- `meeting_status` 为 `started` / `end` 时不要取消，直接告知用户

---

## meeting search — 按关键词搜

```bash
wecom-cli meeting search --json '{"keywords": ["项目评审"], "limit": 20}'
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `keywords` | string[] | ✅ | 最多 10 个；**数组元素之间是 OR，单个元素内空格分隔是 AND** |
| `begin_time` / `end_time` | string | ❌ | 搜索时间范围 |
| `cursor` | string | ❌ | 上一页的 `next_cursor` |
| `limit` | integer | ❌ | 固定传 20（上限就是 20） |
| `sessionid` | string | ❌ | 限定某个群聊/单聊会话的召回 |

返回 `meetings[]`（`meeting_id`、`sub_meeting_id`、`subject`、`begin_time`、`end_time`、`attendee_count`、`meeting_room`、`location`）+ `next_cursor` + `has_more`。

**选择 search 还是 list**：有关键词就用 `search`（不追问时间）；只有时间或泛泛浏览用 `list`。

---

## meeting original get — 会议转写原文

```bash
wecom-cli meeting original get --json '{"meeting_id": "mtXXXX", "limit": 100}'
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `meeting_id` | string | 二选一 | 会议 ID |
| `url` | string | 二选一 | 会议 URL |
| `sub_meeting_id` | string | ❌ | 周期会议某一场 |
| `media_index` | integer | ❌ | 第几段转写，从 0 开始；**不传返回全部段**。只有用户明确说「第 N 段」才传 `N-1` |
| `cursor` | string | ❌ | 分页游标 |
| `limit` | integer | ❌ | 每页条数，默认 100，上限 500 |

返回 `original_data`（逐行 `序号 时间 说话人: 内容`）+ `next_cursor` + `has_more` + `media_index`。

> **转写原文 ≠ 智能纪要**：要「纪要 / 要点 / 待办」用 `meeting get` 的 `notes[]`；要「原话 / 逐字记录 / 转写」才用本方法。`has_more` 为 true 时带 `next_cursor` 翻到底再拼接。

---

## meeting rooms buildings list — 企业大楼

```bash
wecom-cli meeting rooms buildings list --json '{}'
```

返回 `buildings[]`：`name`、`city`、`is_current`。

## meeting rooms search — 查会议室

```bash
wecom-cli meeting rooms search --json '{
  "begin_time": "2026-11-02 14:00:00",
  "end_time": "2026-11-02 15:00:00",
  "room_keyword": "1605",
  "min_capacity": 6
}'
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `begin_time` | string | ✅ | 查询时段开始 |
| `end_time` | string | ✅ | 查询时段结束 |
| `building_name` / `building_city` | string | ❌ | 限定楼栋 / 城市 |
| `floor_name` | string | ❌ | 限定楼层 |
| `room_keyword` | string | ❌ | 会议室名称关键词 |
| `min_capacity` | integer | ❌ | 最小容量 |
| `expand_to_other_buildings` | boolean | ❌ | 允许跨楼推荐（同城）；只有用户明确说可以跨楼才传 |
| `cursor` / `limit` | | ❌ | 分页 |

### 返回

| 字段 | 说明 |
|------|------|
| `target[]` | 按 `room_keyword` 精确/模糊命中的会议室，每项 `{room: {meeting_room_id, name, floor, capacity}, status}` |
| `recommendations[]` | 其余可推荐会议室（同结构，不带 status） |
| `inferred_building` | 后端推断的楼栋 `{name, city, source}` |
| `next_cursor` / `has_more` | 分页 |

> `meeting_rooms` / `meeting_rooms_count` 字段已废弃，不要读。

---

## 常见工作流

### 1. 创建会议（最简）

用户：「明天下午两点约个产品对齐会，一小时」

```bash
wecom-cli meeting create --json '{"subject": "产品对齐", "begin_time": "2026-11-02 14:00:00", "end_time": "2026-11-02 15:00:00"}'
```

用户没给时长时默认 1 小时（`begin_time` + 1h），不要追问。

### 2. 带参会人

1. 姓名 → userid：`wecom-cli contact users search --json '{"keywords": ["张三","李四"]}'`（见 [wecom-contact.md](wecom-contact.md)）
2. 多候选必须让用户选，不能自动挑
3. 组装 `attendees: [{"userid": "..."}]`

```bash
wecom-cli meeting create --json '{"subject": "需求评审", "begin_time": "2026-11-02 10:00:00", "end_time": "2026-11-02 11:00:00", "attendees": [{"userid": "woAAA"}, {"userid": "woBBB"}]}'
```

### 3. 查本周会议

```bash
wecom-cli meeting list --json '{"begin_time": "2026-11-02 00:00:00", "end_time": "2026-11-08 23:59:59", "limit": 100}'
```

把 `created_meetings` 和 `attended_meetings` 合并、按开始时间升序；对要展示的前 10 条 `meeting get` 补参会人姓名；超过 10 条说「还有 N 条」。

**列表为空时不要直接说「没有会议」**——企微里「会」还可能是日程，改用 `calendar schedules list` 用同一时间范围查一遍，命中就一并呈现并说明「这是一条日程，未关联在线会议链接」。

### 4. 改期

```bash
wecom-cli meeting get --json '{"meeting_ids": [{"meeting_id": "mtXXXX"}]}'      # 先确认状态与是否周期
wecom-cli meeting update --json '{"meeting_id": "mtXXXX", "begin_time": "2026-11-04 15:00:00", "end_time": "2026-11-04 16:00:00"}'
```

改时间前用 `calendar schedules free list` 查忙闲（见 [wecom-schedule-check-availability.md](wecom-schedule-check-availability.md)）；有人占线时让用户在「坚持这个时间 / 换一个时间」二选一，不要自行改期。

### 5. 取消

```bash
wecom-cli meeting get --json '{"meeting_ids": [{"meeting_id": "mtXXXX"}]}'
wecom-cli meeting cancel --json '{"meeting_id": "mtXXXX"}'
```

### 6. 拿纪要 / 待办

```bash
wecom-cli meeting get --json '{"meeting_ids": [{"meeting_id": "mtXXXX"}]}'
```

`has_note_permission == true` 且 `notes[].note_content` 有内容就直接用；为空或没权限才切到 `meeting original get` 拉原文再总结。

---

## 注意事项

- **时间格式**统一 `YYYY-MM-DD HH:MM:SS`
- **`meeting_id` 是长字符串**，不是 9 位会议号，两者不能混用
- **`meeting get` 的 `meeting_ids` 必须是对象数组**，上限 10 个
- **`meeting list` 的两个时间参数要么都给要么都不给**
- **参会人先解析成 userid**，不要把姓名直接当 userid 传；多候选必须让用户确认
- **会议室必须查了再订**，只传 `meeting_room_id`，不把会议室名塞进 `location`
- **不暴露内部 ID**：`meeting_id`、`meeting_room_id` 都不进用户可见正文
- **链接按需给、按格式给**：`note_url` / `record_url` 只在用户要链接时输出，且用 `[主题](url)`
- **没有的能力**：会议密码、等候室、入会静音、屏幕水印、外部用户开关、响铃提醒范围——这些 `settings` 字段在 1.3.x 里**全部不存在**（详见 [wecom-meeting-example-full.md](wecom-meeting-example-full.md)）

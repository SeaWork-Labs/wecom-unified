# 企业微信日程管理

> 公共概念与规则请参考 [wecom-shared.md](wecom-shared.md)

通过 `wecom-cli calendar schedules <方法> --json '<JSON 入参>'` 与企业微信日程系统交互。

## 注意事项

- 日程列表查询默认覆盖**当前时间起 30 天**，可用 `begin_time` / `end_time` 收窄；时间格式 `YYYY-MM-DD HH:MM:SS`
- 涉及参与者 `userid` 时，需先用通讯录查询获取（参见 [wecom-contact.md](wecom-contact.md)）；存在同名时展示候选让用户选择（禁止暴露 userid）
- 创建/修改/取消前，先确认目标日程和参与者信息
- 非 0 的 `errcode` 要展示错误信息
- 返回的 `begin_time` / `end_time` 已经是 `YYYY-MM-DD HH:MM:SS` 字符串，**不再是 Unix 时间戳**，不需要再转换

---

## 接口列表

### calendar schedules list — 查询日程列表

```bash
wecom-cli calendar schedules list --json '{"begin_time": "2026-04-15 00:00:00", "end_time": "2026-04-15 23:59:59"}'
```

返回 `schedule_list`，每项都带完整详情（标题、时间、地点、参与人、提醒、周期规则），**不需要再按 ID 查一次详情**。`begin_time` 不传默认当前时间，`end_time` 不传默认开始时间 +30 天。

### calendar schedules get — 按 ID 获取日程详情

```bash
wecom-cli calendar schedules get --json '{"schedule_ids": ["SCHEDULE_ID"]}'
```

按 ID 批量取详情，1~50 个。参见 [API 详情](wecom-schedule-get-schedule-detail.md)。

### calendar schedules create — 创建日程

```bash
wecom-cli calendar schedules create --json '{"subject": "需求评审", "begin_time": "2026-03-12 14:00:00", "end_time": "2026-03-12 15:00:00", "attendees": [{"userid": "USER_ID"}], "location": "3 楼会议室", "reminders": {"is_remind": true, "reminder_time": [-900]}}'
```

参见 [API 详情](wecom-schedule-create-schedule.md) | [reminders 字段](wecom-schedule-ref-reminders.md)。

### calendar schedules update — 修改日程

只需传入需修改的字段，未传字段保持不变；`location` / `description` 传空字符串表示清空。

```bash
wecom-cli calendar schedules update --json '{"schedule_id": "SCHEDULE_ID", "subject": "更新后的标题"}'
```

参见 [API 详情](wecom-schedule-update-schedule.md)。

### calendar schedules cancel — 取消日程

```bash
wecom-cli calendar schedules cancel --json '{"schedule_id": "SCHEDULE_ID"}'
```

周期日程需额外传 `repeat_mod`：`{"repeat_edit_type": "this_only" | "this_and_future" | "all", "repeat_event_time": "目标场次时间"}`。

### 管理参与人

统一走 `calendar schedules update`，用 `add_attendees` / `remove_attendees`：

- 添加参与人：
```bash
wecom-cli calendar schedules update --json '{"schedule_id": "SCHEDULE_ID", "add_attendees": [{"userid": "USER_ID"}]}'
```
- 移除参与人：
```bash
wecom-cli calendar schedules update --json '{"schedule_id": "SCHEDULE_ID", "remove_attendees": [{"userid": "USER_ID"}]}'
```

> ⚠️ 旧版的 `add_schedule_attendees` / `del_schedule_attendees` 两个独立方法已不存在，一律用上面的 patch 形式。

### calendar schedules free list — 查询共同空闲

```bash
wecom-cli calendar schedules free list --json '{"userids": ["USER_ID_1", "USER_ID_2"], "begin_time": "2026-03-12 09:00:00", "end_time": "2026-03-12 18:00:00"}'
```

返回推荐的共同空闲时段。支持 1 个及以上成员（单人也合法，退化为该人的空闲时段查询），可选 `min_duration_minutes`（默认 30）与 `strategy`（默认 `max_attendees`）。参见 [API 详情](wecom-schedule-check-availability.md)。

---

## 典型工作流

### 查询日程

**经典 query 示例：**
- "我今天有哪些日程？"
- "帮我看看这周三下午有没有会议"
- "明天的日程安排是什么？"
- "查一下最近有没有关于项目评审的日程"
- "我下周一到周五的日程都有哪些？"

**流程：**
1. 根据用户意图计算时间范围（如"今天"→当日 00:00:00 至 23:59:59，"这周"→本周一至周日）
2. 调用 `calendar schedules list` 拉取范围内的日程
3. 若用户提到关键词（如"项目评审"），在 `subject` / `description` 中匹配筛选；未找到则逐步扩大范围
4. 展示日程列表时包含标题、时间、地点、参与者等关键信息

### 创建日程

**经典 query 示例：**
- "帮我创建一个明天下午 2 点到 3 点的会议，标题叫需求评审"
- "安排一个周五全天的团建活动"
- "创建日程：后天上午 10 点和张三、李四开产品方案讨论会，地点在 3 楼会议室"
- "帮我建个日程，下周一 14:00-15:00，提前 15 分钟提醒"
- "约一个明天上午的日程，邀请王伟参加"

**流程：**
1. 解析用户意图，提取时间、标题、地点、参与人、提醒设置等信息
2. 若涉及参与人，先通过通讯录查询 userid（参见 [wecom-contact.md](wecom-contact.md)）；存在同名时展示候选让用户选择
3. 若用户未指定提醒，默认 `reminders: {"is_remind": true, "reminder_time": [-900]}`（提前 15 分钟）
4. 若用户说"全天"，设置 `is_all_day: true`，并把时间设为当天 00:00:00 至 23:59:59
5. 向用户确认日程信息（标题、时间、地点、参与人等）后调用 `calendar schedules create`

### 修改日程

**经典 query 示例：**
- "把明天的需求评审改到后天下午 3 点"
- "帮我修改下今天下午的会议标题，改成技术方案评审"
- "我今天 14 点的日程地点改成线上腾讯会议"
- "把周五的团建活动推迟一个小时"
- "帮我给明天的周会加个描述：讨论 Q2 规划"

**流程：**
1. 先通过查询工作流定位目标日程（根据用户提到的时间、标题等关键词匹配）
2. 若匹配到多个日程，展示候选列表让用户确认
3. 向用户确认要修改的字段和目标值
4. 调用 `calendar schedules update`，只传入需修改的字段；周期日程需带 `repeat_mod`

### 取消日程

**经典 query 示例：**
- "取消明天下午的需求评审"
- "帮我把周五的团建日程删掉"
- "我不想开今天 15 点的会了，帮我取消"

**流程：**
1. 先通过查询工作流定位目标日程
2. 向用户确认取消的日程信息（标题、时间等），避免误操作
3. 确认后调用 `calendar schedules cancel`

### 管理参与人

**经典 query 示例：**
- "把张三加到明天的需求评审会议里"
- "帮我把李四从周五的日程里移除"
- "明天下午的会议再邀请一下王伟和赵敏"
- "把我后天那个技术分享的参与人里去掉刘强"

**流程：**
1. 通过通讯录获取目标人员 userid（参见 [wecom-contact.md](wecom-contact.md)）；存在同名时展示候选让用户选择
2. 通过查询工作流定位目标日程
3. 调用 `calendar schedules update`，用 `add_attendees` 或 `remove_attendees` 完成添加/移除

### 查询闲忙并安排会议

**经典 query 示例：**
- "帮我看看张三和李四明天下午有没有空"
- "查一下我和王伟这周的空闲时间，想约个会"
- "我想跟产品组的小明、小红开个会，看看大家什么时候有空"
- "找一个明天下午大家都有空的时段，安排一个 1 小时的会议"

**流程：**
1. 通过通讯录获取相关人员 userid
2. 调用 `calendar schedules free list` 查询共同空闲时段
3. 把返回的 `slots` 整理成可读时段推荐给用户
4. 用户确认时段后，调用 `calendar schedules create` 创建日程并添加参与人

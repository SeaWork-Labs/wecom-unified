# 创建会议 — 安全设置（已不支持）

> ⚠️ **旧版的会议安全设置整体不存在了。** 会议密码、等候室、允许成员在主持人之前入会、入会静音、禁止外部用户、屏幕水印——这些字段在 `@wecom/cli` 1.3.x 的 `meeting.create` schema 里一个都没有，全部无法通过接口设置。

## 用户提「设密码 / 开等候室 / 禁止外部人入会」时怎么办

明确告知不支持，并引导到客户端：

> 通过接口创建会议时不支持设置会议密码、等候室、入会静音或外部用户限制，需要你在企业微信客户端的会议详情里配置。

**不要伪造字段。** 旧文档里的 `settings` 对象整体作废：

```jsonc
// ❌ 这段在 1.3.x 上会被 schema 拒绝或静默忽略，什么都不会发生
"settings": {
  "password": "1234",
  "enable_waiting_room": true,
  "allow_enter_before_host": false,
  "allow_external_user": false,
  "enable_enter_mute": 1,
  "enable_screen_watermark": true
}
```

## 现在能做的、与安全/范围相关的事

| 需求 | 现在怎么做 |
|------|-----------|
| 控制谁能进会 | 只能在 `attendees` 里点名邀请；`attendees` 支持企业外部联系人（`wm` 前缀的 userid），但**没有**「禁止外部人」的开关 |
| 订一个封闭的物理会议室 | `meeting rooms search` → `meeting_room_id`，见 [wecom-meeting.md](wecom-meeting.md#会议室) |
| 限制会议规模 | `attendees` 上限 300（客户端/企业版本可能更低） |
| 备注里写注意事项 | `description`，最长 500 字节 |

```bash
wecom-cli meeting create --json '{
  "subject": "客户汇报会议",
  "begin_time": "2026-11-03 14:00:00",
  "end_time": "2026-11-03 15:30:00",
  "attendees": [{"userid": "woAAA"}, {"userid": "woBBB"}],
  "description": "内部会议，请勿转发入会链接"
}'
```

`description` 只是备注文本，**不是**访问控制——不要跟用户暗示它能阻止他人入会。

## 旧示例（不可用，仅供识别用户描述）

旧文档里的这个场景属于已删除能力：

- 「设置密码 1234，开启等候室，不允许外部人员入会」→ `settings.password` + `settings.enable_waiting_room` + `settings.allow_external_user`

听到这类诉求时按上面的话术告知，不要尝试构造等价参数，也不要编造字段名。

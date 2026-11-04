# 通讯录成员查询

> 公共概念与规则请参考 [wecom-shared.md](wecom-shared.md)

按关键词搜索企业微信通讯录中的成员，拿到 `userid` 用于消息、日程、会议、待办等需要指定人员的场景。

## 操作

### 1. 按关键词搜索成员

```bash
wecom-cli contact users search --json '{"keywords": ["张三"]}'
```

**参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `keywords` | string[] | 是 | 搜索关键词，最多 10 个，关键词之间是 OR 关系 |
| `search_mode` | string | 否 | 传 `"list"` 返回全量命中列表；不传时返回最相关的少量结果 |

关键词可匹配：姓名、姓名拼音、英文名、别名。

**返回：**

```json
{
    "users": [
        {
            "userid": "zhangsan",
            "name": "张三",
            "alias": "Sam",
            "email": "zhangsan@example.com",
            "position": "负责人",
            "departments": ["产品部"],
            "matched_keywords": ["张三"]
        }
    ],
    "users_count": 1,
    "hint": ""
}
```

**返回字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `users` | array | 命中的用户列表 |
| `users[].userid` | string | 用户唯一 ID |
| `users[].name` | string | 中文姓名 |
| `users[].alias` | string | 英文名 / 别名，可能为空 |
| `users[].email` | string | 邮箱，可能为空 |
| `users[].position` | string | 职务，可能为空（注意是"职务"不是"职位"） |
| `users[].departments` | string[] | 部门路径列表，主部门靠前 |
| `users[].matched_keywords` | string[] | 该成员命中的关键词 |
| `users_count` | integer | `users` 数组元素数量 |
| `hint` | string | 命中结果被截断时的补充说明，可能为空 |

---

### 2. 匹配策略

- **精确匹配**：`name` 或 `alias` 与关键词完全一致 → 直接使用
- **唯一命中**：只有一个候选 → 直接使用，向用户展示结果
- **多个命中**：把候选人的姓名、部门、职务展示给用户选择，不得自行决定
- **无命中**：告知用户未找到，改用别名、拼音或邮箱再搜一次

---

## 注意事项

- `search_mode` 只有三种情形需要传 `"list"`：用户明确要「完整名单」「所有叫张三的人」「一共有几个李四」这类清点、穷举意图。其余情况不传
- `alias` 和 `position` 可能为空字符串，搜索与展示时需做空值判断
- 多个关键词之间是 **OR**，单次最多 10 个关键词
- 若结果里 `hint` 非空，说明有命中被截断，展示时如实告知用户
- ⚠️ 禁止根据姓名自行猜测或拼造 `userid`

---

## 典型工作流

### 工作流 1：查询人员信息

用户问："帮我查一下 Sam 是谁？"

1. 调用：

```bash
wecom-cli contact users search --json '{"keywords": ["Sam"]}'
```

2. 若找到唯一匹配，展示：

```
📇 找到成员：
- 姓名：张三
- 别名：Sam
- 部门：产品部
- 职务：负责人
```

3. 若找到多个匹配，展示候选列表请用户确认：

```
🔍 找到多个匹配成员，请确认您要查询的是哪位：

1. 张三（别名：Sam，部门：产品部）
2. 张三丰（别名：Sam2，部门：研发部）

请问您要查询的是哪一位？
```

### 工作流 2：为其他功能提供 userid 转换

用户问："帮我发消息给张三"

1. 调用 `wecom-cli contact users search --json '{"keywords": ["张三"]}'`
2. 确认唯一命中后取其 `userid`
3. 把 `userid` 作为 `chat_id` 传给 `message send`

### 工作流 3：批量查询多个人员

用户问："帮我查一下张三和李四分别是谁？"

1. 一次查询传入多个关键词：

```bash
wecom-cli contact users search --json '{"keywords": ["张三", "李四"]}'
```

2. 按 `matched_keywords` 区分各自命中的人
3. 汇总后一并展示

---

## 快速参考

| 接口 | 用途 | 输入 | 返回 |
|------|------|------|------|
| `contact users search` | 按关键词搜索通讯录成员 | `keywords`、可选 `search_mode` | `users[]`（含 userid、name、alias、departments） |

| 场景 | 策略 |
|------|------|
| 精确匹配（name 或 alias 完全一致） | 直接使用，无需用户确认 |
| 唯一命中 | 直接使用，向用户展示结果 |
| 多个命中 | 展示候选列表，请用户选择 |
| 无命中 | 告知用户未找到，建议换关键词 |

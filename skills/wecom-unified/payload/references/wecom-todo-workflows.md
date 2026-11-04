# 典型工作流

> 字段与命令的完整说明见 [wecom-todo.md](wecom-todo.md)

### 工作流 1：查看待办列表

**用户意图**："看看我最近的待办" / "我有哪些待办事项？"

**步骤：**

1. **拉列表**（`list` 已返回标题、描述、分派人、创建人，**不需要再逐条查详情**）：
```bash
wecom-cli todo list --json '{}'
```

2. **按状态分组展示**。`creator.user_name` / `followers[].user_name` 已经是人名（格式 `英文名(中文名)`），直接用，不要展示 `userid`；创建人是自己时省略该项。

```markdown
## 进行中（2 条）

1. 完成 Q2 规划文档
   - 参与人：张三、李四
   - 截止时间：2026-06-01 09:00:00

2. 提交周报
   - 截止时间：2026-03-17 18:00:00

## 已完成（1 条）

1. 代码评审
```

3. **检查分页**：`has_more` 为 `true` 时必须告诉用户"以上是部分待办，还有更多未显示，需要我继续查看吗？"

---

### 工作流 2：按时间范围查询待办

**用户意图**："这个月创建的待办有哪些？"

```bash
wecom-cli todo list --json '{"create_begin_time": "2026-03-01 00:00:00", "create_end_time": "2026-03-31 23:59:59"}'
```

- 用户没说"创建"还是"截止"时，默认按**创建时间**
- 用户说"已完成的"要显式传 `"status_filter": ["finished"]`；说"所有"传 `["finished","proceed"]`

---

### 工作流 3：创建待办并分派给同事

**用户意图**："帮我创建一个待办，让张三下周一前完成需求文档"

1. **查 userid**：
```bash
wecom-cli contact users search --json '{"keywords": ["张三"]}'
```
同名多人时展示候选让用户选，禁止暴露 `userid`。

2. **创建并分派**：
```bash
wecom-cli todo create --json '{"items": [{"title": "完成需求文档", "follower_ids": ["wo_zhangsan"], "deadline": {"type": "date", "value": "2026-03-24"}}]}'
```

3. **回显结果**（标题、参与人、截止时间三项）：
> 已创建待办「完成需求文档」，参与人：张三，截止时间：2026-03-24。

---

### 工作流 4：完成待办

现在只有一个方法：`todo finish`。

**场景 A：只完成自己那部分**

用户说"我这边搞完了"、"我那块先标了"→ 显式传 `finished_all: false`：

```bash
wecom-cli todo finish --json '{"items": [{"todo_id": "td_xxx", "finished_all": false}]}'
```

**场景 B：整条待办结掉**

用户说"完成了"、"这条结掉"→ 传 `finished_all: true`：

```bash
wecom-cli todo finish --json '{"items": [{"todo_id": "td_xxx", "finished_all": true}]}'
```

**表达不明确时**（"把那条待办完成了"）→ 不传 `finished_all`，让后端返回 `ask_finish_all`，再按 [wecom-todo.md](wecom-todo.md) 里的流程问用户。

**定位待办**时 `status_filter` 要传 `["finished","proceed"]`，避免把已完成的误判为未找到；若发现已经完成，直接告知，不要重复调用。

---

### 工作流 5：修改待办内容或截止时间

**用户意图**："把那个需求文档的待办截止时间改到下周五"

1. **定位目标**：
```bash
wecom-cli todo list --json '{"status_filter": ["finished", "proceed"], "keywords": ["需求文档"]}'
```
匹配到多条时列出候选让用户确认。

2. **更新**（`followers` 是全量替换，改人前先取现有名单再合并）：
```bash
wecom-cli todo update --json '{"items": [{"todo_id": "td_xxx", "deadline": {"type": "date", "value": "2026-03-28"}}]}'
```

3. **回显**：标题 + 参与人 + 新的截止时间。

---

### 工作流 6：删除待办

**用户意图**："删掉'代码评审'那个待办"

1. **定位目标**（必须同时查已完成，否则可能找不到）：
```bash
wecom-cli todo list --json '{"status_filter": ["finished", "proceed"], "keywords": ["代码评审"]}'
```

2. **向用户确认后删除**：
```bash
wecom-cli todo delete --json '{"items": [{"todo_id": "td_xxx"}]}'
```

> - 确认措辞示例："确认删除待办'代码评审'吗？删除后不可恢复。"
> - 若当前用户不是创建人，同一接口的语义是**退出待办**，回复时说"已从你的待办中移除"，不要说"创建人之外无权删除"

---

### 工作流 7：分页获取大量待办

两种翻页方式，二选一：

**方式一：命令行自动翻页**（推荐，写一次拉多页）
```bash
wecom-cli todo list --json '{"limit": 20}' --page-count 5
```

**方式二：手动带 cursor**
```bash
# 首次
wecom-cli todo list --json '{"limit": 20}'
# 返回 has_more=true, next_cursor="CURSOR_1"
wecom-cli todo list --json '{"limit": 20, "cursor": "CURSOR_1"}'
# 返回 has_more=false，拉取完毕
```

**分页规则：**

- `has_more` 为 `true` 时，把 `next_cursor` 作为下次的 `cursor` 传入
- `has_more` 为 `false` 时停止
- 翻页过程中其余过滤参数保持不变
- **统计、计数、"共有多少"这类需求必须翻完所有页再下结论**；结果过大被转存到文件时要读完整
- **如果选择不继续翻页，必须告诉用户还有更多待办未显示，并询问是否需要继续查看**

# 企业微信文档、智能表格与智能文档管理

> 公共概念与规则请参考 [wecom-shared.md](wecom-shared.md)

企业微信文档分四个可用品类，接口按品类分开：

| 品类 | 服务 | URL | 读 | 写 |
|------|------|-----|----|----|
| 文档（doc） | `doc` | `https://doc.weixin.qq.com/doc/*` | `doc contents get` | `doc contents overwrite` / `doc contents append` |
| 智能表格（smartsheet） | `smartsheet` | `https://doc.weixin.qq.com/smartsheet/*` | `smartsheet records list` / `records query` | `smartsheet records add/update/delete` |
| 智能文档（smartpage，原名智能主页） | `smartpage` | `https://doc.weixin.qq.com/smartpage/*` | `smartpage pages get` | `smartpage pages append/overwrite/update` |
| 在线表格（sheet） | `sheet` | `https://doc.weixin.qq.com/sheet/*` | 本参考未覆盖 | 本参考未覆盖 |

> ⚠️ **触发规则**：只有用户明确说「智能文档 / 智能主页」或给出 `/smartpage/` 链接时才走 `smartpage` 服务。泛化的"创建文档 / 写个文档"用 `doc` 服务。

## 调用方式

```bash
wecom-cli doc <方法> --json '<JSON 参数>'
wecom-cli smartpage <方法> --json '<JSON 参数>'
wecom-cli smartsheet <方法> --json '<JSON 参数>'
```

> 旧版的 `doc create_doc` / `doc get_doc_content` / `doc edit_doc_content` / `doc smartsheet_*` / `doc +smartpage_create` 一律不存在了。文档操作留在 `doc` 服务，智能表格整体搬到 `smartsheet` 服务，智能文档整体搬到 `smartpage` 服务；那个 `+` 前缀的特殊写法也随之作废。

## URL 品类识别

- 路径以 `/doc/*` 开头 → 文档 → `doc contents get`
- 路径以 `/smartsheet/*` 开头 → 智能表格 → `smartsheet sheets list` 再往下取数
- 路径以 `/smartpage/*` 开头 → 智能文档 → `smartpage pages get`
- 路径以 `/sheet/*` 开头 → 在线表格 → 用 `sheet` 服务

`docid` 也可用 URL 代替（多数接口的 `docid` 字段支持直接传 URL）。

---

## 一、文档（doc）

### 创建

```bash
wecom-cli doc create --json '{"doc_name": "项目周报", "doc_type": "doc"}'
```

`doc_type` 是字符串枚举：`doc` / `sheet` / `smartsheet` / `smartpage` / `ppt` / `mind` / `flowchart`（**不再是 3 / 10 这类数字**）。返回 `{docid, url, doc_name}`。

可选初始内容：`content`（纯文本，`doc_type=doc` 时有效）或 `file_path`（本地文件路径），二选一；`content_type` 传 `text`（doc）或 `markdown`（smartpage）。

参见 [API 详情](wecom-doc-create-doc.md)。

### 读取

```bash
wecom-cli doc contents get --json '{"docid": "DOCID", "content_type": "markdown"}'
```

返回 `{url, name, content, file_path, version}`：

- 内容较短 → 直接读 `content`
- 内容较大 → `content` 为空、改用 `file_path` 指向的本地文件，用 Read 工具读该文件
- `content_type`：`text` / `markdown`（默认）/ `ooxml`

> 已经**没有异步轮询**了：不再有 `task_id` / `task_done`，一次调用直接拿结果。

参见 [API 详情](wecom-doc-get-doc-content.md)。

### 覆写 / 追加

```bash
wecom-cli doc contents overwrite --json '{"docid": "DOCID", "content_type": "markdown", "content": "# 标题\n\n正文"}'
wecom-cli doc contents append --json '{"docid": "DOCID", "content": "追加的一行"}'
```

- `overwrite` 是全量替换，`content_type` 传 `text` 或 `markdown`；`content` 与 `file_path` 二选一
- 要清空文档不能传空串，传 `" "`（一个空格）
- `append` 只在末尾追加纯文本（`content` 必填），不改动已有内容

**写语义裁定**：用户说"写入 / 写进 / 记录 / 补充 / 加进去"→ 用 `append`；只有明确说"覆盖 / 重写 / 替换 / 清空重写"才用 `overwrite`。

参见 [API 详情](wecom-doc-edit-doc-content.md)。

### 导入本地文件

```bash
wecom-cli doc import --json '{"doc_type": "doc", "file_name": "周报.docx", "file_path": "/abs/path/周报.docx"}'
```

返回 `{docid, url, task_id, task_status}`；`task_status` 为 `processing` 时带上 `task_id` 再调一次本接口查进度。

### 查找文档

```bash
wecom-cli doc search --json '{"keywords": ["项目周报"]}'
```

`keywords` 必填（字面关键词数组）。可选 `doc_types`、`creator_userids`、`created_after` / `created_before`、`search_scope`（`title` / `title_content` / `content`）、`sort_by`（`best_match` / `create_time` / `modify_time`）、`limit`（默认 10，最大 100）、`cursor`。返回 `{docs, docs_count, has_more, next_cursor}`。

### 重命名 / 加协作成员

```bash
wecom-cli doc names update --json '{"docid": "DOCID", "new_name": "新标题"}'
wecom-cli doc members update --json '{"docid": "DOCID", "add_member_list": [{"userid": "USER_ID", "auth": 1}]}'
```

---

## 二、智能文档（smartpage）

### 创建（空文档 / 导入 Markdown）

```bash
# 新建空文档
wecom-cli smartpage create --json '{"name": "项目概览"}'

# 用本地 markdown 生成智能文档
wecom-cli smartpage import --json '{"name": "项目概览", "file_path": "/path/to/overview.md"}'
```

两者都返回 `{docid, url}`。`import` 的 `file_path` 必填、且必须是本地已存在的 markdown 文件。

> 旧版的 `doc +smartpage_create`（带 `+` 前缀、`pages[]` 多子页面）没有了。要建多页面结构：先 `smartpage create` 拿 `docid`，再用 `smartpage pages update` 逐页建。

参见 [API 详情](wecom-doc-smartpage-create.md)。

### 读取

```bash
wecom-cli smartpage pages get --json '{"docid": "DOCID", "content_type": "markdown"}'
```

返回 `{doc_title, pages[], pages_count, rev, creator_name, creator_userid, create_time}`：

| 字段 | 说明 |
|------|------|
| `pages[].page_id` | 页面 ID，后续编辑必须用它 |
| `pages[].page_title` | 页面标题 |
| `pages[].parent_id` | 父页面 ID；为空表示根页面 |
| `pages[].content_file_inner` | 内容 ≤48KB 时直接返回正文 |
| `pages[].file_path` | 内容 >48KB 时返回本地文件路径，用 Read 读 |
| `rev` | 文档版本，编辑时用于乐观锁 |

- `content_type`：`markdown`（默认）/ `text` / `block`（需要 block ID 才能编辑时用）
- 传 `page_id` 只返回该页；不传返回整棵页面树（只有标题、层级、page_id，不含正文）

### 编辑

```bash
# 追加内容到某页
wecom-cli smartpage pages append --json '{"docid": "DOCID", "page_id": "PAGE_ID", "content_type": "markdown", "file_path": "/tmp/new.md"}'

# 覆盖某页
wecom-cli smartpage pages overwrite --json '{"docid": "DOCID", "page_id": "PAGE_ID", "content_type": "markdown", "file_path": "/tmp/full.md"}'

# 建页 / 删页 / 改名 / 移动
wecom-cli smartpage pages update --json '{"docid": "DOCID", "method": "insertAfter", "target_page_id": "PAGE_ID", "create_page": {"page_title": "新页面", "content_type": "markdown"}}'
```

`smartpage pages update` 的 `method`：`insertBefore` / `insertAfter` / `delete` / `overwrite` / `move`；配套 `create_page` / `delete_page` / `rename_page` / `move_page` / `update_page_layout`，四者互斥。

### 上传附件

```bash
wecom-cli smartpage images upload --json '{"docid": "DOCID", "file_path": "/path/a.png"}'
wecom-cli smartpage files upload --json '{"docid": "DOCID", "file_path": "/path/b.pdf"}'
```

图片用 `images`，PDF / Office / 压缩包用 `files`；`file_path` 与 `media_id` 二选一，返回 `{url}`。

### 导出

> ⚠️ **导出接口已经没有了**。`smartpage_export_task` / `smartpage_get_export_result` 在 1.3.x 上不存在，导出能力被 `smartpage pages get` 取代：直接拿 `content_type=markdown` 读正文即可，不需要再提交任务、也不需要轮询 `task_id`。

参见 [API 详情](wecom-doc-smartpage-export.md)。

---

## 三、智能表格结构

智能表格搬到独立的 `smartsheet` 服务：

```bash
wecom-cli smartsheet create --json '{"name": "任务跟踪表"}'
wecom-cli smartsheet sheets list --json '{"docid": "DOCID"}'
wecom-cli smartsheet sheets add --json '{"docid": "DOCID", "sheet_title": "新子表"}'
wecom-cli smartsheet sheets update --json '{"docid": "DOCID", "sheet_id": "SHEET_ID", "new_sheet_title": "改名后"}'
wecom-cli smartsheet sheets delete --json '{"docid": "DOCID", "sheet_id": "SHEET_ID"}'
```

`sheets list` 返回 `{name, url, sheets[], sheets_count}`，`sheets[]` = `{sheet_id, title, type, field_count, record_count, chart_count, fields[]}`；`type` 为 `smartsheet`（子表）或 `dashboard`（仪表盘）。

字段：

```bash
wecom-cli smartsheet fields list --json '{"docid": "DOCID", "sheet_title": "任务跟踪表"}'
wecom-cli smartsheet fields add --json '{"docid": "DOCID", "sheet_title": "任务跟踪表", "fields": [{"field_title": "任务名称", "field_type": "text"}]}'
wecom-cli smartsheet fields update --json '{"docid": "DOCID", "sheet_title": "任务跟踪表", "fields": [{"field_id": "FIELD_ID", "new_field_title": "新标题"}]}'
wecom-cli smartsheet fields delete --json '{"docid": "DOCID", "sheet_title": "任务跟踪表", "fields": [{"field_id": "FIELD_ID"}]}'
```

> ⚠️ `field_type` 现在是**短枚举**（`text` / `number` / `single_select` …），不再是 `FIELD_TYPE_TEXT` 这种常量。完整对照见 [字段类型参考](wecom-doc-smartsheet-field-types.md)。
> 改标题用 `new_field_title`，不是把 `field_title` 改成新值。字段类型不可修改，只能改名。

创建新子表时也可以直接带 `fields` 一次建好结构。

---

## 四、智能表格数据

```bash
wecom-cli smartsheet records list --json '{"docid": "DOCID", "sheet_title": "任务跟踪表", "limit": 100}'
wecom-cli smartsheet records add --json '{"docid": "DOCID", "sheet_title": "任务跟踪表", "records": [{"values": {"任务名称": "完成需求文档", "优先级": [{"text": "高"}]}}]}'
wecom-cli smartsheet records update --json '{"docid": "DOCID", "sheet_title": "任务跟踪表", "records": [{"record_id": "REC_ID", "values": {"任务名称": "更新后"}}]}'
wecom-cli smartsheet records delete --json '{"docid": "DOCID", "sheet_title": "任务跟踪表", "records": [{"record_id": "REC_ID"}]}'
```

- `sheet_id` 与 `sheet_title` 二选一，指定要操作哪个子表
- `records list` 支持 `limit`（最大 1000）、`cursor` / `start`、`view_id`、`filter_spec`、`sort`、`field_titles` / `field_ids` 投影；返回 `{records, records_count, has_more, next_cursor, total}`
- 值格式见 [单元格值格式参考](wecom-doc-smartsheet-cell-value-formats.md)

### SQL 查询（大数据量用这个）

```bash
wecom-cli smartsheet records query --docid 'DOCID' --sql 'SELECT `任务名称`, `负责人` FROM `任务跟踪表` WHERE `优先级` = "高" LIMIT 100'
```

- 这是唯一用 `--docid` 顶层参数的方法；`--sql` 可重复传入多条
- 只允许 `SELECT`；有筛选条件必须写进 SQL，**不要拉全量再本地筛**
- 日期时间字段在 SQL 里是 Excel 序列号，不是 Unix 时间戳，用 `DATE_FORMAT` 格式化
- 返回 `values string[]`，每个元素解析后取 `rows`

参见 [API 详情](wecom-doc-smartsheet-get-records.md)。

---

## 典型工作流

### 读一个文档

1. 看 URL / docid 判断品类
2. `doc` → `doc contents get`；`smartsheet` → `smartsheet sheets list` → `records list`；`smartpage` → `smartpage pages get`
3. 返回 `file_path` 而不是 `content` 时，用 Read 工具读该文件

### 新建文档

1. `doc create --json '{"doc_name":"...", "doc_type":"doc"}'`，保存 `docid`
2. 写内容 → `doc contents overwrite`（初写）或 `doc contents append`（续写）
3. 展示给用户时用 `url`，**不要展示 docid**

### 写智能表格

1. `smartsheet sheets list` 确认子表存在（失败时先查这里，不要直接重试）
2. `smartsheet fields list` 了解列类型与选项
3. 涉及成员字段：优先传 `{"userName": "张三"}`，报错再用 `contact users search` 取 userid 传 `{"userId": "..."}`
4. `records add` 写入
5. 写完 `records list` 回读核对结果

### 改智能表格

1. `records list` 拿 `record_id`
2. `records update` / `records delete`
3. 回读验证

> **成员（USER）字段**必须传 `userId` 或 `userName`，不能只写一个裸字符串姓名。userid 通过 `wecom-cli contact users search` 查（参见 [wecom-contact.md](wecom-contact.md)）。

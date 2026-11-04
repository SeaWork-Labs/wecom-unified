# 智能表格读记录：records query / records list

> ⚠️ 旧版的 `wecom-cli doc smartsheet_get_records '{"docid":..., "sheet_id":...}'` 在 1.3.x 上**不存在**。智能表格已经是独立服务 `smartsheet`，读记录有两个方法：
> - `smartsheet records list` — 按子表读原始行（分页 / 排序 / 过滤）
> - `smartsheet records query` — 用 SQL 读（统计、JOIN、聚合、TopN）
>
> 优先用 `query`；被权限拒绝（`errcode=538005`）时才降级到 `list`。

## 一、smartsheet records query（SQL）

```bash
wecom-cli smartsheet records query --docid '<docid>' --sql 'SELECT RECORD_ID, `任务名称`, `状态` FROM `任务列表` LIMIT 100'
```

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `--docid` | string | ✅ | 文档 ID 或文档 URL。**这条命令的 docid 是顶层 shell 参数**，不放进 `--json` |
| `--sql` | string[] | ✅ | 只读 `SELECT`；可重复传多个 `--sql`，一个元素对应一条语句 |

> 注意 `records query` 与其余 smartsheet 方法不同：其余方法用 `--json '{"docid": ...}'`，只有 `query` 把 `docid` 放在 `--docid` 上。写成 `--json` 里的 `docid` 会报缺少参数。

### 返回

```json
{
  "errcode": 0,
  "values": ["{\"rows\":[{\"文本\":\"内容\",\"单选\":\"选项A\",\"RECORD_ID\":\"r2gG1i\"}]}"]
}
```

| 字段 | 说明 |
|------|------|
| `values` | 与请求中的 SQL 一一对应；每个元素是**字符串**，解析后读里面的 `rows` |
| `rows[]` | 每行是 `{字段名: 值}`，key 是字段**标题**（不是字段 ID） |

### SQL 规则

- 只允许 `SELECT`；禁止写入、删除、建表
- `FROM` 只能用真实子表名，反引号包裹：``FROM `任务列表` ``
- 列引用必须是字段**标题**（反引号包裹），唯一例外是特殊列 `RECORD_ID`——需要后续 `records update` / `delete` 定位行时，显式把它写进 `SELECT`
- 日期时间字段在 SQL 里是 **Excel 序列号**（不是 Unix 毫秒），要可读日期用 `DATE_FORMAT`
- 支持 `JOIN` / `GROUP BY` / `HAVING` / `COUNT(DISTINCT)` / `SUM` / `AVG` / `MIN` / `MAX` / `DATE_FORMAT` / `NOW()` / `CASE WHEN` / `IN` / `EXISTS` / `LIKE`
- 不支持 `FULL JOIN`、窗口函数、`COALESCE` / `IFNULL`、`UNION` / CTE / `PIVOT`、子查询、`CAST`、`STDDEV`、`GROUP_CONCAT` / `ARRAY_AGG`
- **超过 1000 行的求和/计数不要在推理里口算**，让 SQL 算，或提示用户加公式字段

### 人员字段

`user` 类型在 `rows` 里是对象数组 `[{"corp_name":"...","id":"...","name":"张三"}]`。筛选优先用 `LIKE`，不要用 `=`：

```sql
SELECT `负责人` FROM `任务列表` WHERE `负责人` LIKE '%张三%'
```

展示取 `name`，不要暴露 `id`。

## 二、smartsheet records list（原始行）

`records query` 返回 `errcode=538005 / errmsg="没有该智能表的全部权限，请降级使用 wecom-cli smartsheet records list"` 时，改用本方法。

```bash
wecom-cli smartsheet records list --json '{"docid": "DOCID", "sheet_title": "任务列表", "limit": 100}'
```

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `docid` | string | ✅ | 文档 ID 或 URL（**放在 JSON 里**） |
| `sheet_title` | string | 推荐 | 子表名称；也可用 `sheet_id` |
| `field_titles` | string[] | ❌ | 只返回这些列，不传返回全部 |
| `sort` | object[] | ❌ | `[{"field_title": "截止日期", "desc": true}]` |
| `filter_spec` | object | ❌ | 过滤条件，见下 |
| `limit` | integer | ❌ | 每页条数，最大 1000；且需满足 `limit × 返回列数 < 10000` |
| `cursor` | string | ❌ | 上一页的 `next_cursor` |
| `start` | integer | ❌ | 偏移量，默认 0；与 `cursor` 同时设置时 `cursor` 优先 |
| `type` | string | ❌ | 取什么：`fields` / `charts` / `views` / `records` |

> `type` 不传时按普通记录读取处理。要一次连字段定义一起拿，传 `type: "fields"`；要视图传 `views`。

### filter_spec

```json
{
  "conjunction": "and",
  "conditions": [
    {
      "field_title": "状态",
      "field_type": "single_select",
      "operator": "is",
      "string_value": {"value": ["进行中"]}
    }
  ]
}
```

- `field_title` / `field_type` / `operator` 三项必填
- 值字段按类型二选一对应：文本/单选/多选 → `string_value`；数字/货币/百分比/进度 → `number_value`；复选框 → `bool_value`；成员/创建人/编辑人 → `user_value`；日期/创建时间/编辑时间 → `date_time_value`
- 单选/多选可以直接传选项文本，不必先查 `options[].id`
- 空的 `filter_spec` 或空 `conditions` 会被拒

### 返回

```json
{
  "errcode": 0,
  "total": 3,
  "has_more": false,
  "next_cursor": "",
  "records": [
    {
      "record_id": "QizwnX",
      "create_time": "1775025981849",
      "update_time": "1775035035706",
      "creator_name": "张三",
      "updater_name": "张三",
      "values": {
        "任务名称": "完成项目需求文档",
        "状态": [{"id": "oTBIKO", "style": 7, "text": "待开始"}]
      }
    }
  ],
  "records_count": 1
}
```

| 字段 | 说明 |
|------|------|
| `records[].record_id` | 行 ID，`records update` / `delete` 用它定位 |
| `records[].values` | `{字段名: 值}`，值形态按字段类型（见 [单元格值格式](wecom-doc-smartsheet-cell-value-formats.md)） |
| `next_cursor` | `has_more=true` 时非空，下次作为 `cursor` 传入 |
| `warning` | 可能带的告警提示，不要当错误 |

> 只取某一批列时配 `field_titles`，否则 `limit × 列数` 容易超 10000。

## 三、取子表与字段

读记录前通常要先知道子表名和字段名：

```bash
wecom-cli smartsheet sheets list --json '{"docid": "DOCID"}'
wecom-cli smartsheet fields list --json '{"docid": "DOCID", "sheet_title": "任务列表", "limit": 100}'
```

- `sheets[]` 里 `type` 为 `smartsheet`（子表）或 `dashboard`（仪表盘）；子表带 `field_count` / `record_count`
- `sheets[].fields` 只是轻量的列预览（`field_title` + `field_type`），**不能替代 `fields list`**：写记录、判选项 ID、看字段属性时都要单独调 `fields list`

## 四、读不到时的处理

| 现象 | 处理 |
|------|------|
| `errcode=538005` | 降级到 `records list` |
| 子表访问失败 | **不要直接重试**，先 `sheets list` 确认子表存在；存在仍失败按权限问题告知用户 |
| 返回了临时文件路径 | 优先回 SQL/接口层补 `WHERE` / `LIMIT` / 字段投影，不要整文件读进上下文 |
| `errcode == 0` 但没有 `records` | 成功但结果为空，告知用户当前条件下无数据 |

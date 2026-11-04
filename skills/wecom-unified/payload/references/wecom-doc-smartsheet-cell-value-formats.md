# 单元格值格式参考

`value` 是一个 key-value 映射，**key 是字段标题**（也可以用 `key_type` 指定用字段 ID），value 的格式由字段类型决定。

- `smartsheet records add/update/delete`：`key_type` 不传时默认用字段标题
- 想用字段 ID 作 key 时传 `key_type: "field_id"`

> `key_type` 的合法值有四个：`field_title`、`field_id`，以及旧版的 `CELL_VALUE_KEY_TYPE_FIELD_TITLE`、`CELL_VALUE_KEY_TYPE_FIELD_ID`——两个旧常量**仍然被接受**（等于前两者），但新写一律用短枚举。

## 各字段类型的值格式

| `field_type` | value 格式 | 示例 |
|--------------|-----------|------|
| `text` | 字符串 | `"文本内容"` |
| `number` | 数值 | `123.45` |
| `currency` | 数值 | `99.99` |
| `percentage` | 数值（0~1） | `0.85`（显示 85%） |
| `progress` | 数值（0~100） | `75.5` |
| `checkbox` | 布尔 | `true` |
| `date_time` | 字符串，严格 `YYYY-MM-DD HH:mm:ss` | `"2026-01-15 14:30:00"` |
| `phone_number` | 字符串 | `"13800138000"` |
| `email` | 字符串 | `"a@b.com"` |
| `barcode` | 字符串 | `"6901234567890"` |
| `user` | `CellUserValue` 数组 | `[{"userName": "张三"}]` |
| `url` | `CellUrlValue` 数组 | `[{"text": "官网", "link": "https://example.com"}]` |
| `image` | `CellImageValue` 数组 | `[{"id": "xxx", "title": "图", "imageUrl": "https://..."}]` |
| `attachment` | `CellAttachmentValue` 数组 | `[{"id": "xxx", "title": "a.pdf", "fileUrl": "https://..."}]` |
| `select` | `Option` 数组 | `[{"id": "选项ID", "text": "选项A"}]` |
| `single_select` | `Option` 数组 | `[{"id": "选项ID", "text": "高"}]` |
| `location` | `CellLocationValue` 数组 | `[{"id": "...", "source_type": 1, "title": "北京", "latitude": "39.9", "longitude": "116.3", "address": "..."}]` |
| `reference` | `CellReferenceValue` 数组 | `[{"record_id": "rec_xxx"}]` |
| `wwgroup` | `CellGroupValue` 数组 | `[{"chat_id": "<chat_id>"}]` |
| `autonumber` | 只读 | 系统生成，不可写 |

> ⚠️ 旧版要求文本字段写成 `[{"type": "text", "text": "内容"}]` 的数组形式；现在**文本直接传字符串**。单选/多选仍需数组。

## 成员字段（`user`）

```json
"负责人": [{"userName": "张三"}]
```

- **写入优先传 `userName`**，系统会自动匹配同名人
- 传 `userName` 报错（重名、查无此人）时，改用 `contact users search` 查 `userid` 后传 `{"userId": "..."}`
- 读取时两项都会返回：`[{"userId": "...", "userName": "张三"}]`

```bash
wecom-cli contact users search --json '{"keywords": ["张三"]}'
```

## 选项字段（`select` / `single_select`）

已存在的选项用 `id` 匹配（`id` 从 `smartsheet fields list` 的 `property_select.options` 取得），新增选项不填 `id`：

```json
"优先级": [{"id": "opt_123", "text": "高"}],
"标签": [{"text": "紧急"}, {"text": "重要"}]
```

## 图片 / 附件

必须先上传到该文档空间，再把返回的 `url` 写进单元格：

```bash
wecom-cli smartsheet images upload --json '{"docid": "DOCID", "file_path": "/path/a.png"}'
wecom-cli smartsheet files upload --json '{"docid": "DOCID", "file_path": "/path/b.pdf"}'
```

图片返回 `https://w...qpic.cn/...`，非图片文件返回分享链接。

## 完整示例

```bash
wecom-cli smartsheet records add --json '{
  "docid": "DOCID",
  "sheet_title": "任务跟踪表",
  "records": [{
    "values": {
      "任务名称": "完成需求文档",
      "优先级": [{"text": "高"}],
      "截止日期": "2026-03-20 18:00:00",
      "完成进度": 30,
      "已完成": false,
      "负责人": [{"userName": "张三"}]
    }
  }]
}'
```

## 读回来的是另一种形状

`records query`（SQL）返回的行里，人员字段是对象数组，相关筛选优先用 `LIKE`，不要用 `=` 精确匹配。展示时取 `name` 字段。

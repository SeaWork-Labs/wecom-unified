# 智能表格字段类型参考

## 支持的字段类型

`smartsheet.fields.add/update` 的 `field_type` 是**短枚举字符串**，不是 `FIELD_TYPE_*` 常量。

| `field_type` | 说明 | 对应属性字段 |
|--------------|------|-------------|
| `text` | 文本 | — |
| `number` | 数字 | `property_number` |
| `checkbox` | 复选框 | `property_checkbox` |
| `date_time` | 日期时间 | `property_date_time` |
| `image` | 图片 | — |
| `attachment` | 文件 | `property_attachment` |
| `user` | 成员 | `property_user` |
| `url` | 超链接 | `property_url` |
| `select` | 多选 | `property_select` |
| `single_select` | 单选 | `property_single_select` |
| `progress` | 进度 | `property_progress` |
| `phone_number` | 电话 | — |
| `email` | 邮箱 | — |
| `location` | 地理位置 | `property_location` |
| `currency` | 货币 | `property_currency` |
| `percentage` | 百分数 | `property_percentage` |
| `barcode` | 条码 | `property_barcode` |
| `autonumber` | 自动编号 | `property_auto_number` |
| `reference` | 关联 | `property_reference` |
| `two_way_link_records` | 双向关联 | `property_two_way_link_records` |
| `lookup` | 查找引用 | `property_lookup` |
| `formula` | 公式 | `property_formula` |
| `wwgroup` | 群 | `property_ww_group` |
| `created_user` / `modified_user` | 创建人 / 最后编辑人 | 系统字段 |
| `created_time` / `modified_time` | 创建时间 / 最后编辑时间 | `property_created_time` / `property_modified_time` |

> **旧常量到新短枚举的换算**：去掉 `FIELD_TYPE_` 前缀，其余转小写即可 —— `FIELD_TYPE_TEXT` → `text`，`FIELD_TYPE_DATE_TIME` → `date_time`，`FIELD_TYPE_SINGLE_SELECT` → `single_select`，`FIELD_TYPE_TWOWAYLINKRECORDS` → `two_way_link_records`。

> **ai 类字段暂不支持通过接口创建**，遇到这类需求让用户手动建。

## 添加字段

```bash
wecom-cli smartsheet fields add --json '{
  "docid": "DOCID",
  "sheet_title": "任务跟踪表",
  "fields": [
    {"field_title": "任务名称", "field_type": "text"},
    {"field_title": "优先级", "field_type": "single_select"},
    {"field_title": "截止日期", "field_type": "date_time"},
    {"field_title": "完成进度", "field_type": "progress"}
  ]
}'
```

需要选项或属性时带上对应的 `property_*` 对象：

```json
{
  "field_title": "优先级",
  "field_type": "single_select",
  "property_single_select": {
    "is_quick_add": true,
    "options": [{"text": "高"}, {"text": "中"}, {"text": "低"}]
  }
}
```

## 更新字段

```bash
wecom-cli smartsheet fields update --json '{"docid": "DOCID", "sheet_title": "任务跟踪表", "fields": [{"field_id": "FIELD_ID", "new_field_title": "新标题"}]}'
```

- **只能改名，不能改类型**
- 新标题用 `new_field_title`，不是把 `field_title` 改成新值
- `fields` 单次最多 150 项

## 删除字段

```bash
wecom-cli smartsheet fields delete --json '{"docid": "DOCID", "sheet_title": "任务跟踪表", "fields": [{"field_id": "FIELD_ID"}]}'
```

**不可逆**，`field_id` 通过 `smartsheet fields list` 获取。

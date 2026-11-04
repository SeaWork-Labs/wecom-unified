# create 文档 API

新建文档、表格、智能表格、智能文档等品类，可选写入初始内容。

## 命令

```bash
wecom-cli doc create --json '<JSON 参数>'
```

> 旧版的 `doc create_doc`（带 `doc_type: 3 / 10` 数字）已不存在。现在是 `doc create`，`doc_type` 是字符串枚举。

## 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `doc_name` | string | ✅ | 文档名，不能包含 `/ \ : * ? " < > |` |
| `doc_type` | string | ❌ | 文档类型，见下方枚举 |
| `content` | string | ❌ | 纯文本初始内容（`doc_type=doc`）；与 `doc_requests` / `file_path` 三选一 |
| `content_type` | string | ❌ | 内容格式：`doc` 传 `text`，`smartpage` 传 `markdown` |
| `file_path` | string | ❌ | 初始内容文件路径，与 `content` 二选一 |
| `doc_requests` | array | ❌ | document 节点编辑写入（`doc_type=doc` 时有效），与 `content` 二选一 |
| `fields` | array | ❌ | 初始化字段（`doc_type=smartsheet` 时有效），不传用默认字段 |
| `grid_data` | object | ❌ | 默认子表的表格数据（`doc_type=sheet` 时有效） |
| `sheet_title` | string | ❌ | 默认子表名称 |

### `doc_type` 枚举

| 值 | 品类 |
|----|------|
| `doc` | 文档 |
| `sheet` | 在线表格 |
| `smartsheet` | 智能表格 |
| `smartpage` | 智能文档（原智能主页） |
| `ppt` | 演示文稿 |
| `mind` | 思维导图 |
| `flowchart` | 流程图 |

## 请求示例

```bash
# 建空文档
wecom-cli doc create --json '{"doc_name": "项目周报", "doc_type": "doc"}'

# 建文档并写入初始文本
wecom-cli doc create --json '{"doc_name": "项目周报", "doc_type": "doc", "content": "本周进展：", "content_type": "text"}'

# 建智能表格
wecom-cli doc create --json '{"doc_name": "任务跟踪表", "doc_type": "smartsheet"}'
```

## 返回字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `docid` | string | 新建文档 ID，**后续所有操作都要用，务必留在上下文** |
| `url` | string | 文档访问链接，展示给用户用这个 |
| `doc_name` | string | 文档名 |

```json
{
    "docid": "DOCID",
    "url": "https://doc.weixin.qq.com/doc/xxx",
    "doc_name": "项目周报"
}
```

## 注意

- `doc_type=smartsheet` 时文档自带一个默认子表，不需要再 `smartsheet sheets add`
- `docid` 只在创建时返回；**不要展示给用户**，展示用 `url`
- 要建智能文档且希望一次带入多页 markdown，先用 `smartpage import`，不要在这里绕

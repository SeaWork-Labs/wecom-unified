# contents get API

读取 doc 文档正文，支持 text / markdown / ooxml 三种返回格式。

## 命令

```bash
wecom-cli doc contents get --json '<JSON 参数>'
```

> 旧版的 `doc get_doc_content`（带 `type: 2`、`task_id` 异步轮询）已不存在。现在是 `doc contents get`，`content_type` 是字符串枚举，**一次调用直接返回结果，没有轮询**。

## 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `docid` | string | ✅ | 文档 ID，也可直接传文档 URL |
| `content_type` | string | ❌ | `text` / `markdown`（默认）/ `ooxml` |

## 请求示例

```bash
wecom-cli doc contents get --json '{"docid": "DOCID", "content_type": "markdown"}'
wecom-cli doc contents get --json '{"docid": "https://doc.weixin.qq.com/doc/xxx", "content_type": "markdown"}'
```

## 返回字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `content` | string | 内容不长时直接返回正文 |
| `file_path` | string | 内容较大时落盘的本地文件路径；此时 `content` 为空 |
| `name` | string | 文档标题 |
| `url` | string | 文档链接 |
| `version` | integer | 文档版本号 |
| `document` | object | `content_type=ooxml` 时返回的文档对象 |
| `errcode` / `errmsg` | — | 失败时返回 |

```json
{
    "name": "项目周报",
    "url": "https://doc.weixin.qq.com/doc/xxx",
    "content": "# 项目周报\n\n本周进展……",
    "file_path": "",
    "version": 12
}
```

## 注意

- **拿到 `file_path` 时必须用 Read 工具读该文件**，不要向用户展示路径本身
- 篇幅很长时同样走 `file_path`，不要假设 `content` 一定有值
- `content_type` 传错会拿到不同结构（`ooxml` 返回 `document` 对象而不是文本）

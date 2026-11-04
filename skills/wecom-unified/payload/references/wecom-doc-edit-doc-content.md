# contents overwrite / append API

修改 doc 文档正文：`overwrite` 全量替换，`append` 末尾追加。

## 命令

```bash
wecom-cli doc contents overwrite --json '<JSON 参数>'
wecom-cli doc contents append --json '<JSON 参数>'
```

> 旧版的 `doc edit_doc_content`（`content_type: 1`）已不存在。覆写是 `doc contents overwrite`，追加是 `doc contents append`。

## contents overwrite

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `docid` | string | ✅ | 文档 ID，也可直接传 URL |
| `content` | string | 二选一 | 完整的新内容 |
| `file_path` | string | 二选一 | 本地文件路径，`content` 与 `file_path` 必须有一个 |
| `content_type` | string | ❌ | `text`（默认）/ `markdown` |

### 示例

```bash
wecom-cli doc contents overwrite --json '{"docid": "DOCID", "content_type": "markdown", "content": "# 标题\n\n正文内容"}'
wecom-cli doc contents overwrite --json '{"docid": "DOCID", "content_type": "markdown", "file_path": "/tmp/report.md"}'
```

### 返回

成功返回空对象 `{}`。

## contents append

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `docid` | string | ✅ | 文档 ID，也可直接传 URL |
| `content` | string | ✅ | 要追加的纯文本 |

### 示例

```bash
wecom-cli doc contents append --json '{"docid": "DOCID", "content": "补充：本周完成联调。"}'
```

### 返回

成功返回空对象 `{}`。

## 注意

- **`overwrite` 会替换全部内容**，调用前应先用 `doc contents get` 了解现状，并向用户确认
- **清空文档不能传空字符串**：`content` 传 `null` / `""` / 不传都会被拒；要清空请传 `" "`（一个空格）
- **默认用 append**：用户说"写入 / 写进 / 记录 / 补充 / 加进去"都是追加语义；只有明确说"覆盖 / 重写 / 替换 / 清空重写"才用 `overwrite`
- `append` 只支持纯文本，不接受 `file_path`
- 返回空对象是正常成功结果，不代表失败

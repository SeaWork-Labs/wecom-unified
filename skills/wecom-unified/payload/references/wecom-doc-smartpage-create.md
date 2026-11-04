# smartpage create / import API

创建智能文档（原名智能主页）。

## 命令

```bash
wecom-cli smartpage create --json '<JSON 参数>'
wecom-cli smartpage import --json '<JSON 参数>'
```

> 旧版的 `doc +smartpage_create`（带 `+` 前缀、`pages[]` 多子页面数组、`page_filepath`）已不存在。现在是独立服务 `smartpage`，且拆成 `create`（空文档）和 `import`（带 markdown）两个方法；那个 `+` 前缀的特殊写法彻底作废。

## smartpage create

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | ✅ | 智能文档标题 |

### 示例

```bash
wecom-cli smartpage create --json '{"name": "项目概览"}'
```

## smartpage import

把本地 markdown 文件导入成一个智能文档。

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | ✅ | 智能文档标题 |
| `file_path` | string | ✅ | 本地 markdown 文件路径 |

### 示例

```bash
wecom-cli smartpage import --json '{"name": "项目概览", "file_path": "/tmp/overview.md"}'
```

## 返回字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `docid` | string | 新建的文档 ID，后续编辑都要用 |
| `url` | string | 文档访问链接，展示给用户用这个 |
| `doc_name` | string | 文档名 |

```json
{
    "docid": "a1_xxxx",
    "url": "https://doc.weixin.qq.com/smartpage/a1_xxxx",
    "doc_name": "项目概览"
}
```

## 注意

- `import` 的 `file_path` 必须是**本地已存在**的 markdown 文件，且是绝对路径
- **没有多子页面一次创建的能力了**：先 `smartpage create` 拿 `docid`，再用 `smartpage pages update` 逐个建页（`create_page` 动作），或用 `smartpage pages append` 往页面写内容
- docid 通常以 `a1` / `b1` 开头，见到这种前缀就可以判定是智能文档而不是 doc 文档
- **不要展示 docid**，展示 `url`

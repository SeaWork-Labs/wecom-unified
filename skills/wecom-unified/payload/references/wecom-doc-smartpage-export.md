# 智能文档内容读取（原导出接口）

> ⚠️ **`smartpage_export_task` 与 `smartpage_get_export_result` 在 1.3.x 上已经不存在。**
> 异步「提交导出任务 → 轮询 task_id → 拿 content」这套流程整个被删除。现在直接读页面即可，一次调用拿到内容，不需要任务 ID，也不需要轮询。

## 现用命令

```bash
wecom-cli smartpage pages get --json '{"docid": "DOCID", "content_type": "markdown"}'
```

### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `docid` | string | 与 `url` 二选一 | 智能文档 ID |
| `url` | string | 与 `docid` 二选一 | 文档 URL |
| `content_type` | string | ❌ | `markdown`（默认）/ `text` / `block` |
| `page_id` | string | ❌ | 只读该页；不传返回整棵页面树（不含正文） |

### 返回

| 字段 | 说明 |
|------|------|
| `doc_title` | 文档标题 |
| `pages` | 页面数组（扁平列表，用 `parent_id` 表达层级） |
| `pages[].page_id` | 页面 ID，编辑时必用 |
| `pages[].page_title` | 页面标题 |
| `pages[].parent_id` | 父页面 ID；为空表示根页面 |
| `pages[].content_file_inner` | 内容 ≤48KB 时直接返回的正文 |
| `pages[].file_path` | 内容 >48KB 时的本地文件路径，用 Read 读 |
| `rev` | 文档版本，编辑时用于乐观锁 |

## 换算对照

| 旧写法（已失效） | 现在怎么写 |
|------------------|-----------|
| `doc smartpage_export_task '{"docid":..., "content_type":1}'` → 拿 `task_id` | 不需要这一步 |
| `doc smartpage_get_export_result '{"task_id":"..."}'` → 拿 `content` | `smartpage pages get --json '{"docid":"...", "content_type":"markdown"}'` |
| `content_type: 1`（整数） | `content_type: "markdown"`（字符串） |

## 注意

- 不传 `page_id` 时只返回页面结构（标题、层级、page_id），**不含正文**；要正文必须带 `page_id` 或逐页取
- 返回 `file_path` 时用 Read 工具读文件，不要展示路径
- `content_type=block` 只在需要 block ID 做精细编辑时用；普通读取用 `markdown`
- 正文里的图片会以 `![](<CDN URL>)` 形式出现；用户要求基于文档内容作答时，把这些图片下载到本地再交给图片识别能力处理，不要凭想象描述图片内容

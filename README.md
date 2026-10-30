# QHX-BNU.github.io

个人主页 + 知识库博客（GitHub Pages 静态站，无框架、无构建工具）。

## 目录结构

- `index.html` — 主页（含「写作」预览区：最近更新 + 正在探索的问题）
- `blog.html` — 博客列表页（分类筛选 + 分层标签 + 线索 + 搜索 + 时间归档）
- `post.html` — 文章阅读页（`?p=slug`），含阅读时间、所属线索、相关文章
- `posts/posts.json` — 文章目录清单
- `posts/threads.json` — 线索定义（把同一主题的系列文章串成一条脉络）
- `posts/topics.json` — 标签分组（研究领域 / 技术方向 / 具体问题）
- `posts/<slug>.md` — 每篇文章一个 Markdown 文件

## 内容组织原则

**按内容的目的分类，按研究主题打标签，用线索串起演化。**

- **分类**回答"这篇文章是什么性质"：随想 / 学习笔记 / 研究思路 / 工程实践 / 阅读笔记 / 复盘总结。按目的归类，不按主题归类
- **标签**回答"这篇文章讨论什么主题"。`topics.json` 把标签分成三层：研究领域（Agent、RL、LLM、RSI、AI4Edu）、技术方向（Harness、训练、推理、评测、数据、系统设计）、具体问题（Credit Assignment、Reward Model、Memory、Routing、Verification）。没列进分组的标签会自动归到「其他」组，不影响使用
- **线索（thread）**回答"这几篇文章在追同一件事"。跨分类的同主题文章挂同一个 thread，就会在文章页互相串起来，形成 随想 → 学习笔记 → 研究思路 → 工程实践 → 复盘 的脉络

## 发布一篇新文章

1. 新建 `posts/<slug>.md`（`<slug>` 用英文短横线命名，如 `my-first-post.md`），正文用 Markdown 写
2. 打开 `posts/posts.json`，在数组**开头**加一条：

   ```json
   {
     "slug": "my-first-post",
     "title": "文章标题",
     "date": "2026-10-30",
     "category": "ideas",
     "status": "published",
     "thread": "agent-trace-eval",
     "tags": ["Agent", "Evaluation"],
     "summary": "一句话摘要，显示在博客列表里"
   }
   ```

3. 提交并推送到 `main`，GitHub Pages 会自动发布（约 1 分钟内生效）

### 字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| `slug` | ✓ | 文件名（不含 `.md`），也是 URL 参数 |
| `title` | ✓ | 文章标题 |
| `date` | ✓ | `YYYY-MM-DD`，列表和归档按它排序 |
| `category` | ✓ | 六选一：`thoughts`（随想）/ `notes`（学习笔记）/ `ideas`（研究思路）/ `engineering`（工程实践）/ `reading`（阅读笔记）/ `retro`（复盘总结） |
| `tags` | | 主题标签数组，点击标签可筛选同主题文章 |
| `summary` | | 列表页摘要 |
| `thread` | | 线索 id，对应 `threads.json` 里的条目；同线索文章会在文章页互相显示 |
| `status` | | 可选：`published`（默认）/ `draft`（草稿）/ `organizing`（整理中）。非已发布状态会显示小徽标 |
| `featured` | | 可选：首页精选标记，留给后续扩展 |

## 加一条线索

在 `posts/threads.json` 里加一条：

```json
{
  "id": "agent-trace-eval",
  "title": "Agent Trace 与评测",
  "desc": "从单条 Trace 里的噪声，到跨天聚类暴露出来的系统性缺陷。",
  "tags": ["Agent", "Trace", "Evaluation"]
}
```

然后在相关文章的 `posts.json` 条目里填 `"thread": "agent-trace-eval"`。文章页会自动出现「线索」区块，博客列表也会多出这条线索的筛选按钮。

## 维护标签分组

`posts/topics.json` 里每个组有一个 `tags` 数组。新标签想归到「研究领域」，就加进第一组的 `tags`；不加也能用，会显示在「其他」组里——所以标签可以随手加，分组可以慢慢整理。

## 页面参数

- `blog.html?cat=ideas` — 直接打开某个分类
- `blog.html?tag=Agent` — 按标签筛选
- `blog.html?thread=agent-trace-eval` — 按线索筛选
- `blog.html?q=Trace` — 预填搜索
- `blog.html?view=archive` — 直接进入时间归档
- `post.html?p=<slug>` — 打开某篇文章
- 任何页面加 `?theme=paper` 可指定配色（midnight / aurora / paper / terminal）

## 评论（giscus · GitHub Discussions）

文章页评论由 [giscus](https://giscus.app) 提供：读者用 GitHub 账号登录后直接发表，支持回复和表情回应，评论即时显示。

- 评论存储在本仓库的 **Discussions** 里，分类为 `Announcements`，每篇文章对应一个讨论（term = 文章 slug）
- 站点配色切换时，评论框主题会自动跟随（paper → light，其余 → dark）
- 管理评论：直接在仓库 Discussions 页面删除/锁定讨论；GitHub 应用「giscus」需保持对本仓库的访问权限

相关配置在 `post.html` 的 `GISCUS` 常量中（repo、repo-id、category、category-id）。

## Markdown 支持范围

标题（`#`～`###`）、加粗、斜体、行内代码、围栏代码块、有序/无序列表、引用、链接、图片、分割线。

## 数学公式（KaTeX）

块级公式用 `math` 围栏，不要写成 `text`，否则上下标会按原始字符显示：

````markdown
```math
\hat A_t^{\mathrm{GAE}(\gamma,\lambda)}
=\sum_{l=0}^{\infty}(\gamma\lambda)^l\delta_{t+l}^V
```
````

支持 `aligned`、`\frac`、`\sum`、`\mathbb`、`\operatorname` 等常见 LaTeX 写法。公式由 KaTeX 在文章页渲染；如果 CDN 加载失败，会回退显示原始 TeX，便于排查而不是静默丢失。


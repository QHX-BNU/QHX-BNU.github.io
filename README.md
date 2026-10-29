# QHX-BNU.github.io

个人主页（GitHub Pages 静态站，无框架、无构建工具）。

## 目录结构

- `index.html` — 主页
- `blog.html` — 博客列表页（分类筛选 + 标签筛选 + 时间归档）
- `post.html` — 文章阅读页（读取 `?p=slug` 参数渲染对应文章）
- `posts/posts.json` — 文章目录清单
- `posts/<slug>.md` — 每篇文章一个 Markdown 文件

## 内容组织原则

**按内容的目的分类，按研究主题打标签。**

- **分类**回答"这篇文章是什么性质"：随想 / 学习笔记 / 研究思路 / 工程实践 / 阅读笔记 / 复盘总结
- **标签**回答"这篇文章讨论什么主题"：Agent、RL、LLM、Harness、Evaluation、AI4Edu 等，写文章的过程中逐步增加

## 如何发布一篇新文章

1. 新建 `posts/<slug>.md`（`<slug>` 用英文短横线命名，如 `my-first-post.md`），正文用 Markdown 写
2. 打开 `posts/posts.json`，在数组**开头**加一条：

   ```json
   {
     "slug": "my-first-post",
     "title": "文章标题",
     "date": "2026-10-30",
     "category": "ideas",
     "status": "published",
     "tags": ["Agent", "Evaluation"],
     "summary": "一句话摘要，显示在博客列表里"
   }
   ```

3. 提交并推送到 `main`，GitHub Pages 会自动发布（约 1 分钟内生效）

### 字段说明

| 字段 | 说明 |
|------|------|
| `slug` | 文件名（不含 `.md`），也是 URL 参数 |
| `category` | 六选一：`thoughts`（随想）/ `notes`（学习笔记）/ `ideas`（研究思路）/ `engineering`（工程实践）/ `reading`（阅读笔记）/ `retro`（复盘总结） |
| `status` | 可选：`published`（已发布，默认）/ `draft`（草稿）/ `organizing`（整理中）。非已发布状态会显示小徽标 |
| `tags` | 主题标签数组，点击标签可筛选同主题文章 |
| `summary` | 列表页摘要 |

## 分类与标签的约定

- 一篇关于 Agent 的内容，可以是随想（一个疑问）、学习笔记（一个推导）、也可以是研究思路（一个待验证的假设）——按**目的**归类，不按主题归类
- 想法可以演化：随想 → 学习/阅读笔记 → 研究思路 → 工程实践 → 复盘总结，相关文章在正文里用 Markdown 链接互相引用

## Markdown 支持范围

标题（`#`～`###`）、加粗、斜体、行内代码、围栏代码块、有序/无序列表、引用、链接、图片、分割线。

# QHX-BNU.github.io

个人主页（GitHub Pages 静态站，无框架、无构建工具）。

## 目录结构

- `index.html` — 主页
- `blog.html` — 博客列表页
- `post.html` — 文章阅读页（读取 `?p=slug` 参数渲染对应文章）
- `posts/posts.json` — 文章目录清单
- `posts/<slug>.md` — 每篇文章一个 Markdown 文件

## 如何发布一篇新文章

1. 新建 `posts/<slug>.md`（`<slug>` 用英文短横线命名，如 `my-first-post.md`），正文用 Markdown 写
2. 打开 `posts/posts.json`，在数组里加一条：

   ```json
   {
     "slug": "my-first-post",
     "title": "文章标题",
     "date": "2026-10-30",
     "tags": ["随笔"],
     "summary": "一句话摘要，显示在博客列表里"
   }
   ```

3. 提交并推送到 `main`，GitHub Pages 会自动发布（约 1 分钟内生效）

## Markdown 支持范围

标题（`#`～`###`）、加粗、斜体、行内代码、围栏代码块、有序/无序列表、引用、链接、图片、分割线。

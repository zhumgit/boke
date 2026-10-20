# boke

Hugo 静态博客源站，通过 GitHub Pages 发布。

## 约定

- 文章目录：`content/posts/<slug>.md`
- frontmatter 只保留四个字段：`title` / `date` / `tags` / `author`
- 发布流程：新文章 → 提交 PR → 合并到 `main` → 自动部署

## 本地预览

```bash
hugo server -D
```

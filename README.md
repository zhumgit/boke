# boke

Hugo 静态博客源站，通过 GitHub Pages 发布 → https://zhumgit.github.io/boke/

## 约定

- 文章目录：`content/posts/<slug>.md`
- frontmatter 只保留四个字段：`title` / `date` / `tags` / `author`
- 发布流程：新文章 → 提交 PR → 合并到 `main` → 自动构建并部署

## 目录结构

```
content/posts/          文章（Markdown）
archetypes/posts.md     `hugo new posts/xxx.md` 的 frontmatter 模板
hugo.toml               站点配置（主题、菜单、markup）
.github/workflows/
  deploy.yml            合并到 main 后构建 + 部署 Pages
  pr-check.yml          PR 上只构建不部署
```

## 写一篇新文章

```bash
git switch -c post/my-post
hugo new posts/my-post.md   # 用 archetypes/posts.md 生成 frontmatter
# 写完
git add content/posts/my-post.md
git commit -m "post: my-post"
git push -u origin post/my-post
gh pr create --base main --fill
```

## 本地预览

主题不随仓库提交（见 `.gitignore`），本地按 **CI 用的同一个 commit** 拉一份（全量 clone 很慢，只取那一个 commit 约 1MB）：

```bash
mkdir -p themes/PaperMod && cd themes/PaperMod
git init -q . && git remote add origin https://github.com/adityatelange/hugo-PaperMod
git fetch -q --depth 1 origin 969bebdc5600a1639158b5ef4eff9cf6cbb511ac
git checkout -q FETCH_HEAD
cd ../..
hugo server -D
```

Hugo 版本用 extended 版，CI 固定为 0.168.0，主题固定为 PaperMod commit `969bebdc`（v8.0 tag 与 Hugo 0.158+ 不兼容）。

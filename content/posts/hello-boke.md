---
title: "Hello, boke"
date: 2026-10-20T16:16:24+08:00
tags: ["公告", "hugo"]
author: "zhumgit"
---

第一篇测试文章，用来验证 Hugo + GitHub Pages 的构建链路。

## 为什么要验证

只有当站点能真的构建出 HTML，才算这条流水线可用。

```bash
hugo --minify
```

- 文章放在 `content/posts/`
- frontmatter 只有四个字段
- 合并到 `main` 后自动部署

---
title: "如何发布一篇文章"
date: 2026-10-06T09:05:00+08:00
---

在 `content/posts/` 下新建一个 Markdown 文件，推送到 GitHub 即自动发布，全程约一分钟。

## 1. 新建文章

```markdown
---
title: "文章标题"
date: 2026-10-06T12:00:00+08:00
---

正文用 Markdown 书写。
```

文件名就是链接的一部分：`my-first-post.md` 会发布到
`https://ursaaaa.github.io/posts/my-first-post/`。

## 2. 本地预览（可选）

```bash
hugo server -D
```

打开 http://localhost:1313 即可实时预览，改动即时生效。

## 3. 发布

```bash
git add .
git commit -m "post: 文章标题"
git push
```

推送后 GitHub Actions 会自动构建并部署，约一分钟后刷新页面即可看到新文章。

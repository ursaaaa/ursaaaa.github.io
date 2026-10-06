# Ursaaaa · 技术笔记

基于 [Hugo](https://gohugo.io/) 的个人博客，部署在 GitHub Pages：
<https://ursaaaa.github.io/>

## 写新文章

1. 在 `content/posts/` 新建 Markdown 文件（参考已有文章的 front matter）
2. `git push`，GitHub Actions 会自动构建发布（约 1 分钟）

## 本地预览

```bash
hugo server -D
```

## 结构

- `content/posts/` 文章
- `layouts/` 自定义极简主题（无第三方依赖）
- `static/css/style.css` 样式（含深色模式适配）

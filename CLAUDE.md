# Blog 项目

## 项目概述

基于 Hexo + Fluid 主题的个人博客，部署在 Edge One。

## 部署流程

**写文章 → commit → push → Edge One 自动构建**

```bash
# 1. 在 source/_posts/ 下创建 .md 文件
# 2. 提交并推送
git add source/_posts/你的文章.md
git commit -m "新增文章：xxx"
git push origin source
```

- Edge One 监听 `source` 分支，push 后自动构建部署
- **不需要** `hexo generate` 和 `hexo deploy`（那个是推 master 分支的，Edge One 不读）
- `master` 分支是 hexo deploy 的产物，本项目不使用

## 文章规范

- 文件位置：`source/_posts/`
- Markdown 格式，YAML frontmatter 必填字段：title, date, tags, categories, description
- 图片使用七牛云 CDN：`https://static.xiangdangnian.net.cn/blog/xxx.png`

## 图片配图（doc-image skill）

使用 `/doc-image` 命令自动配图。流程：AI 生成 → TinyPNG 压缩 → 七牛云上传 → CDN URL 插入文章。

## 技术栈

- Hexo + Fluid 主题
- 部署：Edge One（监听 source 分支）
- 图片 CDN：七牛云（`static.xiangdangnian.net.cn`）
- 域名：`blog.xiangdangnian.net.cn`

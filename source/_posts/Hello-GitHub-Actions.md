---
---
title: Hello GitHub Actions：我的云端自动化部署测试
date: 2026-03-28 16:30:00
tags: [CI/CD, 博客折腾, 测试]
categories: 日常记录
---

## 🚀 自动化流水线测试点火

就像把写好的 C 代码交给交叉编译器一样，这是我第一次尝试用 **GitHub Actions** 来接管博客的编译和部署工作。

如果大家能看到这篇文章，说明我的云端 CI/CD 流水线已经跑通了！

### 测试清单：
- [x] 本地 Markdown 源码推送到 `source` 分支
- [x] GitHub 机器人自动拉取 Ubuntu 环境和 Node.js
- [x] 云端执行 `hexo generate` 生成静态网页
- [x] 自动将 HTML 文件推送到 `main` 分支并发布

> 以后写博客，终于可以像提交代码一样优雅了：`git commit` 之后，剩下的全交给云端服务器。
---

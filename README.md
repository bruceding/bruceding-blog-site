# BruceDing's Blog

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Hugo](https://img.shields.io/badge/Hugo-0.139%2B-ff4088)](https://gohugo.io)
[![PaperMod](https://img.shields.io/badge/Theme-PaperMod-blue)](https://github.com/adityatelange/hugo-PaperMod)

> 个人技术博客，记录学习与思考。访问地址：[bruceding.me](https://bruceding.me)

## 📖 简介

这是我的个人技术博客，基于 [Hugo](https://gohugo.io) 静态网站生成器构建，使用 [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 主题。

博客内容主要涵盖：
- **云原生技术** - Kubernetes、容器、微服务
- **WebAssembly** - WASM 技术与实践
- **MCP 协议** - Model Context Protocol 相关
- **Linux 系统** - IO 栈分析、性能调优
- **开发工具** - Hugo、Ollama、OpenClaw 等

## 🛠️ 技术栈

| 技术 | 说明 |
|------|------|
| [Hugo](https://gohugo.io) | 静态网站生成器 |
| [PaperMod](https://github.com/adityatelange/hugo-PaperMod) | Hugo 主题 |
| Markdown | 博客内容格式 |
| GitHub Pages / Cloudflare Pages | 部署平台（可选） |

## 📁 项目结构

```
bruceding-blog-site/
├── archetypes/          # 内容模板
├── assets/              # 静态资源（CSS、JS 等）
├── content/             # 博客内容
│   └── posts/           # 博客文章
│       └── images/      # 文章图片
├── themes/              # 主题目录
│   └── PaperMod/        # PaperMod 主题（git submodule）
├── hugo.toml            # Hugo 配置文件
├── .github/             # GitHub Actions 工作流
└── LICENSE              # MIT 许可证
```

## 🚀 本地开发

### 前置要求

- [Hugo Extended](https://gohugo.io/installation/) v0.139.0 或更高版本
- Git

### 安装与运行

```bash
# 克隆仓库（包含子模块）
git clone --recursive https://github.com/bruceding/bruceding-blog-site.git
cd bruceding-blog-site

# 如果忘记 --recursive，可以单独拉取子模块
git submodule update --init --recursive

# 启动开发服务器
hugo server -D

# 浏览器访问 http://localhost:1313
```

### 常用命令

```bash
# 创建新文章
hugo new posts/my-new-post.md

# 构建静态文件
hugo --minify

# 构建结果在 public/ 目录
```

## 📝 写作

### 创建新文章

```bash
hugo new posts/文章标题.md
```

### 文章 Front Matter

```yaml
---
title: "文章标题"
date: 2026-03-20T14:00:00+08:00
draft: false
tags: ["tag1", "tag2"]
categories: ["分类"]
---
```

## 🌐 部署

### GitHub Pages

项目已配置 `.github/workflows/`，推送到 main 分支会自动构建部署。

### 手动部署

```bash
# 构建
hugo --minify

# public/ 目录即为静态文件，可部署到任意静态托管平台
```

## 🔗 链接

- **博客地址**: [https://bruceding.me](https://bruceding.me)
- **GitHub**: [https://github.com/bruceding](https://github.com/bruceding)
- **Twitter/X**: [https://x.com/bruce_ding](https://x.com/bruce_ding)

## 📄 License

[MIT License](LICENSE) © 2025 bruceding

---

欢迎 Star ⭐ 和 Fork 🍴

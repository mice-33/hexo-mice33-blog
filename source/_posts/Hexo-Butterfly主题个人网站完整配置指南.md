---
title: Hexo Butterfly主题个人网站完整配置指南
date: 2025-08-01 05:53:28
updated: 2025-08-01 05:53:28
tags:
  - Hexo
  - Butterfly
  - 博客搭建
  - 网站配置
  - 教程
categories:
  - 技术教程
  - 博客搭建
author: mice33
description: 从零开始配置Hexo Butterfly主题，包含基础配置、评论系统、页面创建等完整教程
keywords: Hexo, Butterfly主题, 博客配置, Waline评论, 网站搭建
top: true
cover: /img/hexo-butterfly-config.jpg
---

# 🦋 Hexo Butterfly主题个人网站完整配置指南

本文详细记录了我搭建个人博客网站的完整过程，包含主题配置、评论系统、页面创建等所有步骤。

<!-- more -->

## 📚 参考资料

- [Hexo Butterfly主题相关配置 - 知乎](https://zhuanlan.zhihu.com/p/492207978)
- [Hexo-Next 主题博客个性化配置超详细](https://blog.csdn.net/as480133937/article/details/100138838)
- [Hexo-Butterfly主题配置](https://www.cnblogs.com/ncphoton/p/16950595.html)

## 1️⃣ 基础配置

### 1.1 准备工作

#### 🔧 配置文件管理

为了减少升级主题后带来的不便，建议使用以下方法：

在 Hexo 根目录创建 `_config.butterfly.yml` 文件，并把主题目录的 `_config.yml` 内容复制进去。

```text
├── _config.yml                  # Hexo 主配置文件
├── _config.butterfly.yml        # Butterfly 主题配置文件
└── themes/butterfly/_config.yml # 主题原始配置（保留）
```

> ⚠️ **注意**：
> - 不要删除主题目录的 `_config.yml`
> - 优先级：`_config.butterfly.yml` > `themes/butterfly/_config.yml`
> - 以后只需在 `_config.butterfly.yml` 中配置

### 1.2 代码仓库管理

#### 📂 双仓库策略

建议将代码仓库和网站仓库分开管理：

```bash
# 博客源码仓库（私有）
https://github.com/用户名/hexo-blog-source

# 网站部署仓库（公开）
https://github.com/用户名/用户名.github.io
```

#### 🔗 本地连接代码仓库

```bash
# 查看当前远程仓库
git remote -v

# 连接源码仓库
git remote add origin https://github.com/用户名/hexo-blog-source.git

# 推送代码
git push -u origin main
```

### 1.3 网站更新流程

```bash
# 清除缓存
hexo clean

# 重新生成静态文件
hexo generate

# 部署到网页仓库
hexo deploy
```

## 2️⃣ 评论系统配置

### 2.1 Waline 评论系统部署

#### 🚀 部署步骤

**① 一键部署到 Vercel**

1. 访问 [Waline 官方部署链接](https://github.com/walinejs/waline/tree/main/example)
2. 点击 "Deploy with Vercel"
3. 连接 GitHub 账号
4. 设置项目名称（如：`my-waline-comments`）
5. 直接 Deploy
6. 获取部署地址（如：`https://my-waline-comments.vercel.app`）

#### 🗄️ 数据库配置（推荐）

**为什么需要配置数据库？**

| 不配置数据库 | 配置 LeanCloud |
|-------------|---------------|
| ✅ 零配置，5分钟搞定 | ✅ 数据永久保存 |
| ✅ 适合测试 | ✅ 支持邮件通知 |
| ❌ 重部署丢失数据 | ✅ 可备份导出 |
| ❌ 功能受限 | ✅ 后台管理 |

**LeanCloud 配置步骤：**

1. **注册 LeanCloud**
   - 访问 [LeanCloud 控制台](https://console.leancloud.app/)
   - 注册免费账号
   - 创建应用（选择开发版）

2. **获取密钥**
   ```text
   应用设置 → 应用凭证

   需要的信息：
   - AppID: xxxx-gzGzoHsz
   - AppKey: xxxx-bjGzoHsz
   - MasterKey: xxxx,master
   ```

3. **配置 Vercel 环境变量**
   ```text
   Settings → Environment Variables

   添加三个变量：
   LEAN_ID = 你的AppID
   LEAN_KEY = 你的AppKey
   LEAN_MASTER_KEY = 你的MasterKey
   ```

4. **重新部署项目**

#### ⚙️ 主题配置

```yaml
# _config.butterfly.yml
comments:
  use: waline
  text: true
  lazyload: false
  count: true
  card_post_count: true

waline:
  serverURL: https://comments.mice33.top/
  bg:
  pageview: true
  option:
    meta: ['nick', 'mail', 'link']
    requiredMeta: ['nick']
    lang: 'zh-CN'
    locale:
      placeholder: '欢迎留言讨论~'
    avatar: 'monsterid'
    avatarCDN: 'https://www.gravatar.com/avatar/'
    wordLimit: 0
    pageSize: 10
```

## 3️⃣ 页面内容配置

### 3.1 菜单配置

```yaml
# _config.butterfly.yml
menu:
  主页: / || fas fa-home
  博文 || fa fa-graduation-cap:
    分类: /categories/ || fa fa-archive
    标签: /tags/ || fa fa-tags
    归档: /archives/ || fa fa-folder-open
  生活 || fas fa-list:
    分享: /shuoshuo/ || fa fa-comments-o
    相册: /photos/ || fa fa-camera-retro
    音乐: /music/ || fa fa-music
    影视: /movies/ || fas fa-video
  留言板: /comment/ || fa fa-paper-plane
  关于笔者: /about/ || fas fa-heart
```

### 3.2 页面创建

#### 📄 基础页面创建命令

```bash
# 创建关于页面
hexo new page about

# 创建留言板
hexo new page comment

# 创建分类页面
hexo new page categories

# 创建标签页面
hexo new page tags

# 创建生活页面
hexo new page shuoshuo
hexo new page photos
hexo new page music
hexo new page movies
```

#### 📝 页面内容示例

**关于页面 (`source/about/index.md`)：**

````markdown
---
title: 关于我
date: 2025-08-01 05:53:28
type: about
comments: true
---

# 👋 你好，我是 Mice33

## 🎯 关于这个博客
这里是我的个人技术博客，主要分享：
- 生信技术心得
- 生物学习笔记
- 生活感悟/评论
- 有趣的故事

## 📚 技能栈
### 🧬 生物信息学
- **序列分析**: BLAST, ClustalW, MEGA
- **基因组学**: BWA, GATK, SAMtools
- **转录组学**: DESeq2, edgeR, GSEA

### 💻 编程语言
- **R语言**: 数据分析、可视化
- **Python**: 生信脚本、机器学习
- **Shell/Bash**: Linux环境数据处理

## 📫 联系我
- **邮箱**: mice33@example.com
- **GitHub**: https://github.com/mice33
- **ORCID**: 0000-0000-0000-0000

欢迎在下方留言交流 👇
````

## 4️⃣ 如何创建博文

### 4.1 创建新文章

#### 🖊️ 基本命令

```bash
# 创建新文章
hexo new "文章标题"

# 创建草稿（不会发布）
hexo new draft "文章标题"

# 指定文章类型
hexo new post "文章标题"
```

#### 📍 实际示例

```bash
# 创建生信相关文章
hexo new "生信入门：BLAST基本使用方法"
hexo new "Python处理FASTA文件的技巧"
hexo new "R语言数据可视化入门"
```

### 4.2 文章模板

创建后的文章位于：`source/_posts/文章标题.md`

**完整的文章模板：**

````markdown
---
title: 生信入门：BLAST基本使用方法
date: 2025-08-01 05:53:28
updated: 2025-08-01 05:53:28
tags:
  - 生物信息学
  - BLAST
  - 序列比对
  - 入门教程
categories:
  - 生信技术
  - 工具使用
author: mice33
description: 详细介绍BLAST的基本概念和使用方法，适合生信初学者
keywords: BLAST, 生物信息学, 序列比对, 生信入门
top: false
cover: https://example.com/blast-cover.jpg
---

# 🧬 生信入门：BLAST基本使用方法

## 📖 什么是BLAST？

BLAST（Basic Local Alignment Search Tool）是生物信息学中最常用的序列比对工具...

<!-- more -->

## 🎯 BLAST的主要类型

### 1. BLASTn
- 用于核酸序列对核酸数据库的比对
- 适用于基因序列、转录本序列等

### 2. BLASTp
- 用于蛋白质序列对蛋白质数据库的比对
- 适用于蛋白质功能注释

## 💻 实际操作示例

```bash
# 使用命令行BLAST
blastn -query input.fasta -db nt -out result.txt -outfmt 6
```
````

### 4.3 Front-matter 字段说明

| 字段 | 说明 | 示例 |
|------|------|------|
| `title` | 文章标题 | `"BLAST使用教程"` |
| `date` | 发布日期 | `2025-08-01 05:53:28` |
| `updated` | 更新日期 | `2025-08-01 05:53:28` |
| `tags` | 标签（数组） | `["生信", "教程"]` |
| `categories` | 分类（数组） | `["技术", "生信"]` |
| `description` | 文章描述 | SEO 优化用 |
| `cover` | 封面图片 | 文章头图 |
| `top` | 是否置顶 | `true/false` |

### 4.4 发布流程

```bash
# 1. 创建文章
hexo new "你的文章标题"

# 2. 编辑内容
notepad source/_posts/你的文章标题.md

# 3. 预览效果
hexo server

# 4. 发布文章
hexo clean
hexo generate
hexo deploy
```

---

**这份配置指南帮到你了吗？如果有任何问题，欢迎在评论区讨论！** 💬

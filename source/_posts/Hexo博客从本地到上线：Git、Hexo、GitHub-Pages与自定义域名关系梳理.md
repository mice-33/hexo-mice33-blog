---
title: Hexo博客从本地到上线：Git、Hexo、GitHub Pages与自定义域名关系梳理
date: 2026-03-21 22:30:00
updated: 2026-03-21 22:30:00
tags:
  - Hexo
  - Git
  - GitHub Pages
  - SSH
  - Node.js
categories:
  - 技术教程
  - 博客搭建
description: 结合我自己搭建博客时遇到的问题，系统梳理 Hexo 博客源码、Git 提交、Hexo 生成、GitHub Pages 部署、自定义域名与本地预览之间的关系。
cover:
---

# 🧩 Hexo博客从本地到上线：Git、Hexo、GitHub Pages与自定义域名关系梳理

最近在重新折腾个人博客的时候，考虑了一下如下问题：

- 为什么 GitHub 上的内容变了，但网站没有立刻变？
- 为什么 `mice-33.github.io` 和 `mice33.top` 都能打开，但其实还是同一个站点？
- 为什么本地能运行，VS Code 终端里却不行？
- 为什么中国内地访问 GitHub Pages 有时不太稳定？
- 为什么有的教程直接用 `hexo`，而我换了电脑后得用 `npx hexo`？

这篇文章不再只讲“怎么操作”，而是把**整个博客运行链路背后的知识**系统梳理一遍。搞懂这些之后，后面无论是更新博客、部署网站，还是迁移托管平台，都会清楚很多。

---

## 1️⃣ 你到底在维护什么

很多人刚开始搭博客时，会下意识觉得自己在维护“一个网页”。

其实不完全对。

你真正维护的是一个 **Hexo 博客工程**，它至少包含两类内容。

### 1.1 博客源码

也就是你平时真正编辑的那部分文件，比如：

- `source/` 里的文章、页面
- `_config.yml`
- `_config.butterfly.yml`
- `package.json`
- 主题目录里的模板、样式和配置

这些内容是给 **Hexo** 用的，不是浏览器直接访问的网页。

### 1.2 生成后的静态文件

Hexo 会把源码“编译”成浏览器能直接打开的静态页面，比如：

- `index.html`
- CSS
- JavaScript
- 图片资源
- 每篇文章对应的 HTML 页面

这些内容通常会被生成到：

```bash
public/

⚠️ 真正上线给别人访问的，是这份生成结果，而不是源码本身。

所以要特别记住一句话：

GitHub 仓库变了，不一定等于网站变了。
因为你可能只是上传了源码，还没有重新生成并部署网页。

2️⃣ Git 和 GitHub 到底在做什么
2.1 Git 是什么

Git 是 版本管理工具。

它负责记录：

你改了哪些文件
哪次改动是一个完整版本
如何回退到旧版本
如何把本地版本同步到远程仓库
2.2 GitHub 是什么

GitHub 是 托管 Git 仓库的平台。

它负责帮你保存远程仓库内容，也就是把你本地的 Git 项目放到网上。

2.3 最常见的三个 Git 操作
git add .
git commit -m "更新博客内容"
git push origin main

它们分别表示：

git add：把改动加入暂存区
git commit：在本地生成一个版本快照
git push：把本地提交推送到远程仓库

整个过程可以理解成：

改文件 → add → commit → push

3️⃣ 为什么 GitHub 变了，网站却没变

这是 Hexo 初学者最容易困惑的一点。

原因是：源码同步 和 网站发布 其实是两件事。

3.1 源码同步

比如你执行：

git add .
git commit -m "更新博客源码"
git push origin main

这一步做的事情是：

✅ 把博客源码同步到 GitHub 仓库

但它不一定会自动更新网站页面。

3.2 网站发布

如果你想让别人真正看到更新后的网站，Hexo 还需要执行：

npx hexo clean
npx hexo g
npx hexo d

或者简写成：

npx hexo d -g

这里分别表示：

clean：清理旧缓存和旧生成结果
g = generate：重新生成静态站点
d = deploy：把生成结果发布到托管平台

所以更准确地说：

git push 推的是源码
hexo deploy 推的是网站

这两个概念一定不要混。

4️⃣ SSH、HTTPS、remote 是什么关系

Git 把本地仓库同步到 GitHub 时，常见有两种连接方式。

4.1 HTTPS 方式

像这样：

https://github.com/用户名/仓库名.git

优点：

好理解
一开始最常见

缺点：

容易遇到认证、网络重置、登录问题
某些网络环境下不太稳定
4.2 SSH 方式

像这样：

git@github.com:mice-33/hexo-mice33-blog.git

优点：

配好后比较稳定
推送时不用反复走 HTTPS 认证
4.3 remote 是什么

remote 可以理解成：

这个本地 Git 仓库对应的远程地址

查看当前远程仓库：

git remote -v

把远程仓库从 HTTPS 改成 SSH：

git remote set-url origin git@github.com:mice-33/hexo-mice33-blog.git
4.4 公钥和私钥

使用 SSH 时，会涉及一对密钥：

私钥：保存在你电脑里，不能泄露
公钥：添加到 GitHub，让 GitHub 识别你

GitHub 通过这套机制确认：

你是不是这个仓库对应账号的合法拥有者

5️⃣ Node、npm、npx、Hexo 分别是什么

这几个名字经常一起出现，但作用不一样。

5.1 Node.js

Node.js 是 JavaScript 的运行环境。

很多前端工具、构建工具、博客生成器都依赖它，Hexo 也是其中之一。

5.2 npm

npm 是 Node.js 自带的包管理工具。

它负责安装项目依赖，比如：

npm install

含义是：

按照 package.json 安装这个项目需要的所有依赖

5.3 Hexo

Hexo 是一个 静态博客生成器。

它的核心工作有两步：

读取你的博客源码
生成静态网页文件

常见命令如下：

hexo clean
hexo g
hexo s
hexo d

分别表示：

clean：清理缓存
g：生成静态站点
s：本地预览
d：部署到远程平台
5.4 npx 是什么

npx 可以理解成：

临时运行项目里安装的命令工具

比如当前项目里已经有 Hexo，但你的系统里没有全局 hexo 命令，那么就可以这样运行：

npx hexo g

这表示：

用当前项目里的 Hexo 来执行生成操作

5.5 为什么有的教程不用 npx

因为很多教程默认作者已经全局安装过 Hexo CLI，所以可以直接写：

hexo clean
hexo generate
hexo deploy

而我这里更适合用：

npx hexo clean
npx hexo g
npx hexo d

两种写法本质上是在做同一件事，只是调用方式不同：

hexo ...：依赖系统全局环境
npx hexo ...：依赖当前项目环境

对于 Windows + PowerShell 场景，npx 往往更稳一些。

6️⃣ 本地预览和正式发布不是一回事

这点非常重要。

6.1 本地预览
npx hexo s

执行后，通常会得到一个地址：

http://localhost:4000

这表示网站只是运行在你自己的电脑上。

特点是：

默认只有本机能访问
电脑关机后网站就不存在了
它只是预览，不是正式上线
6.2 正式发布
npx hexo d

这一步表示：

把生成好的静态网站发布到托管平台

比如：

GitHub Pages
Cloudflare Pages
Vercel
OSS

这时别人访问的是托管平台上的网站，不再依赖你的电脑一直开机。

所以可以得出一个结论：

内网穿透只对“本地运行的网站”有意义
对 GitHub Pages 这种已经托管好的站点没意义

7️⃣ GitHub Pages 是什么

GitHub Pages 可以理解为：

GitHub 自带的静态网站托管服务

它可以把仓库中的静态文件发布成网页。

常见有两种类型：

7.1 用户站点

仓库名通常是：

username.github.io

比如我的默认站点地址就是：

https://mice-33.github.io
7.2 项目站点

某个普通仓库也可以开启 Pages，从某个分支或目录发布静态网站。

8️⃣ 默认域名和自定义域名是什么关系

我现在这个博客有两个入口：

https://mice-33.github.io
https://mice33.top/

它们的关系其实是这样的。

8.1 mice-33.github.io

这是 GitHub Pages 提供的默认域名。

8.2 mice33.top

这是我绑定的自定义域名。

但一定要注意：

自定义域名只是换了一个访问入口
不等于换了网站托管服务器

如果 mice33.top 最终通过 DNS 解析到的还是 GitHub Pages，
那么它本质上仍然是 GitHub Pages 上的同一个网站。

所以：

默认域名能访问
自定义域名也能访问

并不代表这是两个不同网站。
通常只是同一站点的两个入口。

9️⃣ 为什么中国内地访问可能不稳定

如果博客当前的核心链路是：

本地生成
部署到 GitHub Pages
域名最终解析到 GitHub Pages

那么出现下面这种情况时：

海外/新加坡网络可以访问
中国内地网络不太稳定

问题通常不在 Hexo，也不在 Git，而在这条链路本身：

中国内地到 GitHub Pages 的访问线路不稳定

也就是说，这不是“Hexo 没部署成功”，而是“访问托管平台的网络体验不稳定”。

因此：

换自定义域名 不一定能解决问题
内网穿透 也解决不了 GitHub Pages 的访问链路问题
真要改善内地访问体验，通常需要考虑更适合的托管平台或 CDN

不过对于个人博客来说，如果目前主要是自己使用和偶尔分享，那么继续用 GitHub Pages 其实已经足够轻量、方便、免费。

🔟 VS Code 终端和本机终端为什么会不一样


理论上，VS Code 的终端也是你电脑上的终端，但它和你平时单独打开的 PowerShell / Git Bash / cmd 还是可能有差异。

常见差别包括：

10.1 终端类型不同

比如：

本机外部终端是 Git Bash
VS Code 里开的是 PowerShell

那同一条命令的表现就可能完全不同。

10.2 PATH 环境变量不同

有时你在系统里已经安装了 Node、Git、Hexo，
但 VS Code 打开得比较早，没有重新读取最新环境变量。

于是就会出现：

外部终端能运行
VS Code 终端找不到命令
10.3 权限不同

有些命令在管理员 PowerShell 里能执行，
但在普通权限终端里不一定能执行。

10.4 SSH agent 会话不同

就算你已经配好了 SSH，
也不代表每一个终端窗口都会自动拿到同样的 SSH 会话。

所以我后来采取的方式是：

✅ 写博客、改文件、看差异：用 VS Code
✅ Git 提交、Hexo 生成、Hexo 部署：优先用本机已经验证可用的终端

这样最省心。

1️⃣1️⃣ 我现在实际采用的更新流程

结合前面这些知识，我现在更推荐的日常更新流程是：

11.1 提交源码到 GitHub
git add .
git commit -m "更新博客内容"
git push origin main
11.2 重新生成并部署网站
npx hexo d -g

如果要拆开写，也可以：

npx hexo clean
npx hexo g
npx hexo d

这样一来：

GitHub 上的源码会更新
GitHub Pages 上的网站也会更新
1️⃣2️⃣ 把整个知识链串成一句话

最后，用一句话把整条链路总结一下：

Hexo 博客源码通过 Git 做版本管理，用 GitHub 存远程仓库，通过 SSH key 做安全认证，在本地终端里用 Node / npm / npx / Hexo 生成静态网站，再把生成结果部署到 GitHub Pages，最后通过默认域名或自定义域名访问。
如果中国内地访问不稳，那属于托管平台和网络链路问题，不是 Hexo 或 Git 本身的问题。

✅ 小结

如果你也和我一样，一开始总把下面这些概念混在一起：

源码和网站
本地预览和正式发布
GitHub 仓库和 GitHub Pages
默认域名和自定义域名
hexo 和 npx hexo

那这篇文章最想说明的就是：

只要把“源码、生成、部署、托管、访问”这五层关系理清楚，整个博客系统就会一下子明朗很多。

后面不管是继续用 GitHub Pages，还是以后迁移到其他托管平台，思路都会更清晰。

📌 我自己的最简工作流

最后附上我当前自己在用的一套最简命令：

git add .
git commit -m "更新博客"
git push origin main
npx hexo d -g

如果你只是想先看本地效果：

npx hexo s

如果你只想备份源码，不更新网站：

git add .
git commit -m "更新源码"
git push origin main

欢迎留言交流，如果后面我把博客迁移到其他托管平台，也会继续记录完整过程。
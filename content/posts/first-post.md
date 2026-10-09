---
title: "把博客挂上云：为什么选 Hugo + Cloudflare Pages"
date: 2026-10-09T12:00:00+08:00
draft: false
tags: ["Hugo", "部署", "静态站"]
categories: ["工程实践"]
description: "零成本、零运维地把中文技术博客上线的一次完整选型与踩坑记录。"
---

写博客这件事，难点从来不在写作，而在**上线之后还要有人照看它**。所以我给自己定的目标是：不花钱、不维护服务器、写完 `git push` 就自动发布。

## 为什么不租免费服务器

免费 VM（各家云厂商的试用实例、Oracle Cloud 的 Always Free 之类）看起来更"自由"，但代价是把运维全揽回自己身上：系统补丁、Nginx 配置、HTTPS 证书续期、被扫描爆破的 SSH 端口，还有实例闲置后被回收的风险。博客的内容是 Markdown，输出是一堆静态 HTML，用不到任何服务端算力。

静态托管则只需要回答两个问题：文件放哪，谁来构建。

| 方案 | 免费额度 | 大陆访问 | 备注 |
| --- | --- | --- | --- |
| Cloudflare Pages | 带宽不限、构建次数够用 | 相对最稳 | 全球 CDN，本文的落地方案 |
| GitHub Pages | 软限流量 | 时好时坏 | 零注册成本，域名固定 |
| Vercel / Netlify | 每月约 100G 流量 | 一般需要自有域名改善 | 更适合带 SSR 的项目 |
| Render 免费档 | 会休眠 | 差 | 冷启动十几秒，不适合给人看 |

## 为什么是 Hugo

在 Astro、Next.js、Hugo 之间选 Hugo，是因为它把「文章」当第一类公民：一个 `content/posts/*.md` 目录就是全部数据源，构建是毫秒级的单二进制调用，产物是纯静态文件，不依赖运行时。

代价是主题生态多用 SCSS，于是**必须安装 Hugo Extended**。标准版 Hugo 缺少 SCSS 编译器，构建 PaperMod 时会直接失败，而报错信息并不会提醒你缺的是什么：

```bash
# 确认自己装的是 extended 版
hugo version
# hugo v0.167.0-xxxx+extended windows/amd64   ← 必须带 +extended
```

## 站内搜索依赖一个容易被忽略的配置

PaperMod 的搜索不是服务端检索，而是构建期生成一份 JSON 索引，再由浏览器里的 Fuse.js 做模糊匹配。所以站点配置里必须让首页额外输出 JSON：

```toml
[outputs]
  home = ["HTML", "RSS", "JSON"]
```

少了 `"JSON"`，`/search/` 页面照样能打开，输入关键词却永远零结果 —— 而且不会有任何报错。这类"静默失效"的配置项，是我把它写进第一篇博客的原因。

## 代码块配色跟随深浅色

Chroma 默认把颜色以 `style="color:#..."` 内联进每个 token，于是深色模式下代码块会刺眼。关掉它，改由主题的 CSS class 着色：

```toml
[markup.highlight]
  noClasses = false
```

## 部署流水线

GitHub Actions 里做三件事：按固定版本装 Hugo、构建、用 `wrangler` 把 `public/` 推到 Cloudflare Pages。把 Hugo 版本钉死很重要 —— 本地和 CI 用了不同小版本时，产物的目录结构、slug 甚至 RSS 都可能漂移，线上就会看到本地复现不了的差异。

```yaml
- run: hugo --gc --minify
- uses: cloudflare/wrangler-action@v3
  with:
    packageManager: pnpm
    command: pages deploy public --project-name=dream-blog
```

## 上线后的自查清单

1. `curl -I https://<你的站点>/` 返回 200；
2. 首页文章列表、文章页目录、归档页时间分组、关于页都正常；
3. 搜索一个只在正文出现的词，能命中；
4. 切到深色模式，代码块配色跟随变化；
5. **改一个字再 push 一次**，确认自动重新部署 —— 首次上线成功只说明搭对了，第二次才说明流水线真的在跑。

## 还没做的

评论（Giscus）和访问统计（Cloudflare Web Analytics）都各有前置条件：前者要先在 giscus.app 上给仓库装 GitHub App 并拿到几个 ID，后者要等站点上线后从 Cloudflare 后台取 beacon token。它们都能很干净地接进来，但值得等站点先稳定跑一段时间。

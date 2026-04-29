---
title: 从零搭建个人博客：Astro + Fuwari + GitHub Pages
published: 2026-04-29
description: "记录一下用 Astro 和 Fuwari 主题搭建个人博客，并部署到 GitHub Pages 的完整过程。"
tags: ["Astro", "博客", "GitHub Pages", "前端"]
category: 技术
draft: false
---

一直想搭个自己的博客，记录生活和折腾技术的笔记。研究了一圈，最终选了 Astro + Fuwari 主题，部署在 GitHub Pages 上。整个过程比想象中简单，记录一下。

## 为什么选 Astro

对比了几个方案：

- **Hugo**：Go 写的，速度极快，但模板语法不太习惯
- **Hexo**：中文社区活跃，但性能一般
- **Next.js**：太重了，个人博客用不着那么多功能
- **Astro**：新锐选手，默认零 JS，构建快，主题生态好

Astro 的理念是「内容优先」，页面默认不发送任何 JavaScript，需要交互的组件按需加载。对博客这种以阅读为主的场景非常合适。

## Fuwari 主题

在 Astro 主题市场里挑了 Fuwari，主要看中：

- 卡片式布局，颜值在线
- 自带深色/浅色模式
- 支持全文搜索
- 文章分类 + 标签
- RSS 订阅
- 目录导航（TOC）

开箱即用，不用自己写太多样式。

## 搭建过程

### 1. 克隆模板

```bash
git clone --depth 1 https://github.com/saicaca/fuwari.git my-blog
cd my-blog
```

用 `--depth 1` 浅克隆，速度快很多。

### 2. 安装依赖

```bash
pnpm install
```

Fuwari 用 pnpm 管理依赖，确保你装了 pnpm。没有的话：

```bash
npm install -g pnpm
```

### 3. 配置博客

编辑 `src/config.ts`，主要改这几项：

```typescript
export const siteConfig: SiteConfig = {
  title: "随风的博客",
  subtitle: "记录生活与技术",
  lang: "zh_CN",       // 中文
  themeColor: {
    hue: 200,           // 蓝色系，0-360 自由调整
  },
};

export const profileConfig: ProfileConfig = {
  avatar: "assets/images/avatar.png",   // 替换成自己的头像
  name: "随风",
  bio: "记录生活，探索技术，保持好奇。",
  links: [
    {
      name: "GitHub",
      icon: "fa6-brands:github",
      url: "https://github.com/你的用户名",
    },
  ],
};
```

### 4. 本地预览

```bash
pnpm dev
```

打开 http://localhost:4321 就能看到效果。

## 部署到 GitHub Pages

### 创建仓库

在 GitHub 上创建一个仓库，比如 `my-blog`。

### 配置 Astro

编辑 `astro.config.mjs`，设置站点地址和 base path：

```javascript
export default defineConfig({
  site: "https://sucli.github.io",
  base: "/my-blog",        // 仓库名
  trailingSlash: "always",
});
```

### 创建 GitHub Actions 工作流

在 `.github/workflows/deploy.yml` 创建部署配置：

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: "pnpm"

      - name: Install dependencies
        run: pnpm install

      - name: Build Astro
        run: pnpm build

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: dist

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 开启 GitHub Pages

1. 进入仓库 Settings → Pages
2. Source 选择 **GitHub Actions**
3. 保存

### 推送代码

```bash
git add -A
git commit -m "初始化博客"
git push -u origin main
```

推送后 GitHub Actions 会自动构建和部署。等 2-3 分钟就能访问了。

## 写新文章

在 `src/content/posts/` 下创建文件夹，新建 `index.md`：

```markdown
---
title: 文章标题
published: 2026-05-01
description: "一句话描述"
tags: ["标签1", "标签2"]
category: 分类名
draft: false
---

正文内容，支持 Markdown...
```

写完 push 到 main 分支，博客会自动更新。

## 总结

整个搭建过程大概半小时，主要花在选方案和配置上。Astro + Fuwari 的组合对个人博客来说很合适：

- 构建快，写完秒部署
- Markdown 写作体验好
- 颜值高，不用折腾样式
- GitHub Pages 免费，够用

博客地址：https://sucli.github.io/my-blog/

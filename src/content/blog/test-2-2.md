---
title: 从零到一：使用 Astro 主题 "Frosti" 搭建并部署 GitHub Pages 个人博客笔记
description: 如题，这片博文记录的是我在GitHub上fork来EveSunMaple的博客作为我的GitHubPages的过程
pubDate: 2025-5-17 # 发表日期，注意格式，可以参考其他文章或config里的date_format
image: /dist/view.png # 可选，文章封面图路径，图片放public/image/下
categories:
  - tech
tags:
  - Astro
  - GitHub Pages
# badge: Pin # 如果想置顶，可以加上
draft: false # false 表示发布，true 表示草稿
---

## 一、 前期准备与目标

- **目标：** Fork 一个 GitHub 上的 Astro 博客主题 "Frosti"，将其个性化配置，并成功部署到 GitHub Pages，最终使用 [用户名].github.io 作为博客主域名。
- **技术栈/工具：**
  - GitHub 账号
  - Git (通过 GitHub Desktop 操作)
  - Node.js
  - pnpm (项目推荐的包管理器)
  - Astro (静态站点生成器)
  - Frosti (博客主题)
  - GitHub Actions (用于 CI/CD 自动部署)
  - VS Code (或其他代码编辑器)

## 二、 Fork 项目与本地环境搭建

1. **Fork 项目：**

   - 访问 "Frosti" 项目的 GitHub 主页 (https://github.com/EveSunMaple/Frosti)。
   - 点击右上角的 "Fork" 按钮，将项目 Fork 到自己的 GitHub 账号下。
   - **初始仓库名：** 暂定为 Frosti (后续会修改为 [用户名].github.io)。

2. **Clone 项目到本地：**

   - 使用 GitHub Desktop 将 Fork 后的 Frosti 仓库 Clone 到本地电脑。

3. **安装 pnpm：**

   - 根据项目 README 推荐，全局安装 pnpm：

     ```
     npm i -g pnpm
     ```

     content_copydownload

     Use code [with caution](https://support.google.com/legal/answer/13505487).Bash

   - 验证安装：pnpm -v (记录下版本号，如 v10.x.x，后续 Actions 配置可能需要)。

4. **安装项目依赖：**

   - 在本地项目根目录下，通过终端运行：

     ```
     pnpm install
     ```

     content_copydownload

     Use code [with caution](https://support.google.com/legal/answer/13505487).Bash

   - **遇到的问题1：pnpm approve-builds 提示**

     - **现象：** pnpm install 后出现警告，提示 esbuild, sharp, swup 等依赖的构建脚本被忽略。
     - **原因：** pnpm 出于安全考虑，默认不执行依赖的构建脚本。
     - **解决：** 运行 pnpm approve-builds，在交互式命令行中选中 esbuild, sharp, swup，按回车确认，允许它们执行构建脚本。

5. **首次构建项目 (为 Pagefind 搜索索引做准备)：**

   - 根据 README 指示，在启动开发服务器前需要先构建一次：

     ```
     pnpm run build
     ```

     content_copydownload

     Use code [with caution](https://support.google.com/legal/answer/13505487).Bash

   - **遇到的问题2：构建脚本中 cp 命令在 Windows 下不兼容**

     - **现象：** pnpm run build 过程中，Astro 构建成功，Pagefind 索引生成成功，但最后一步报错 "'cp' 不是内部或外部命令..."。

     - **原因：** package.json 的 build 脚本中包含了 cp -r dist/pagefind public/ 命令，该命令在 Windows 下无效。

     - **分析：** 此 cp 命令的目的是将构建产物 dist/pagefind 复制回源码目录 public/，这在逻辑上是不必要的，且可能导致循环复制和源码目录污染。

     - **解决：**

       1. 安装 shx 作为开发依赖，以提供跨平台的 Unix 命令支持：pnpm add -D shx。

       2. 修改 package.json 中的 build 脚本，将 cp 替换为 shx cp (虽然解决了兼容性，但逻辑问题仍在)。

       3. **最终优化：** 移除该 shx cp -r dist/pagefind public/ 操作，因为 pagefind --site dist 已将索引生成到正确的 dist/pagefind/ 目录，无需复制回 public。最终 build 脚本简化为：

          ```
          "build": "astro check && astro build && pagefind --site dist",
          ```

          content_copydownload

          Use code [with caution](https://support.google.com/legal/answer/13505487).Json

6. **启动本地开发服务器：**

   ```
   pnpm run dev
   ```

   content_copydownload

   Use code [with caution](https://support.google.com/legal/answer/13505487).Bash

   - 访问 http://localhost:4321 (Astro 默认端口) 预览博客初始效果。

## 三、 个性化配置博客

1. **核心配置文件 frosti.config.yaml (位于项目根目录)：**
   - **site 部分：**
     - tab: 浏览器标签页文字。
     - title: 网站主标题。
     - description: 网站描述 (SEO)。
     - language: 网站语言 (如 en 改为 zh 中文)。
     - favicon: 网站图标路径。
   - **user 部分：**
     - name: 用户名/昵称。
     - site: **重要！** 用户个人网站链接。**这里需要配置为 GitHub Pages 的根 URL，例如 https://[你的用户名].github.io。** 这个值会被 src/config.ts 中的 USER_SITE 变量引用，并最终用于 astro.config.ts 的 site 选项。
     - avatar: 头像图片路径 (如 /my-avatar.png，图片需放置在 public/ 目录下)。
   - **menu 部分：** 配置导航栏链接、文字、图标。
   - **social 部分：** 配置社交媒体链接。
   - 其他如 theme, date_format 可根据喜好调整。
2. **多语言文本 src/i18n/translations.yaml：**
   - 如果 frosti.config.yaml 中 site.language 设置为 zh，则需要编辑此文件中 zh: 下的标签，将其翻译成中文。
3. **替换图片资源：**
   - 头像：放置在 public/ 目录下，并在 frosti.config.yaml 的 user.avatar 中正确引用。
   - 网站图标：替换 public/favicon.ico。
   - 文章封面图等其他图片：放置在 public/image/ 或其他自定义路径，并在文章 Frontmatter 中引用。
4. **Astro 配置文件 astro.config.ts (或 .mjs)：**
   - **site 选项：** 已通过 import { USER_SITE } from "./src/config.ts"; 从 frosti.config.yaml (经由 src/config.ts) 间接获取，确保其值为 https://[你的用户名].github.io。
   - **base 选项：** **关键！** 用于指定网站部署在域名的哪个子路径下。
     - **初始配置 (仓库名为 Frosti)：** base: '/Frosti' (注意大小写与仓库名一致)。
5. **撰写博客文章：**
   - 文章 Markdown (.md 或 .mdx) 文件存放于 src/content/blog/ 目录。
   - 每篇文章头部包含 Frontmatter，定义标题、描述、发布日期、分类、标签等元信息。

## 四、 配置 GitHub Actions 实现自动部署 (初次尝试)

1. **创建 Workflow 文件：**
   - 在项目根目录下创建 .github/workflows/deploy.yml 文件。
   - 粘贴提供的标准 Astro 部署到 GitHub Pages 的 Actions Workflow 代码。关键步骤包括：
     - 触发条件 (如 on: push: branches: [ main ])。
     - 权限设置 (permissions)。
     - 检出代码 (actions/checkout)。
     - 设置 pnpm (pnpm/action-setup) 和 Node.js (actions/setup-node)。
     - 安装依赖 (pnpm install --frozen-lockfile)。
     - 构建项目 (pnpm run build)。
     - 配置 Pages (actions/configure-pages)。
     - 上传构建产物 (actions/upload-pages-artifact)。
     - 部署到 Pages (actions/deploy-pages)。
   
2. **提交代码到 GitHub：**
   - 使用 GitHub Desktop 将所有本地修改 (包括配置文件、新添加的 workflow 文件等) 提交 (Commit) 并推送 (Push) 到远程 Frosti 仓库的 main 分支。
   
3. **遇到的问题3：Actions 中 pnpm install 报错 ERR_PNPM_INVALID_WORKSPACE_CONFIGURATION**
   - **现象：** "Install dependencies" 步骤失败，提示 packages field missing or empty。
   - **原因：** 项目中存在 pnpm-workspace.yaml 文件，但其内容 (onlyBuiltDependencies: ...) 不符合 pnpm workspace 对 packages 字段的要求。
   - **解决：**
     1. 将 pnpm-workspace.yaml 中的 onlyBuiltDependencies 配置移至 package.json 文件内的 pnpm 字段下。
     2. 删除项目根目录下的 pnpm-workspace.yaml 文件。
     3. 提交并推送这些更改。
   
4. **遇到的问题4：Actions 中 pnpm install 报错 ERR_PNPM_NO_LOCKFILE**
   - **现象：** "Install dependencies" 步骤失败，提示 Cannot install with "frozen-lockfile" because pnpm-lock.yaml is absent 或 Ignoring not compatible lockfile。
   - **原因分析：**
     - pnpm-lock.yaml 文件未被正确提交到远程仓库。
     - CI 环境中的 pnpm 版本与本地生成 lockfile 的 pnpm 版本不兼容。
   - **解决：**
     1. **确认 .gitignore 文件没有忽略 pnpm-lock.yaml (检查结果：未忽略，正常)。**
     2. **在本地重新运行 pnpm install 确保 pnpm-lock.yaml 是最新的，并提交推送到远程仓库。**
     3. **统一 CI 环境与本地的 pnpm 版本。** 查看本地 pnpm -v (例如为 v10.x.x)，修改 .github/workflows/deploy.yml 中 pnpm/action-setup 的 version 参数为匹配的主版本 (如 version: 10)。
     4. 提交并推送 deploy.yml 的修改。
   
5. **遇到的问题5：Actions 中 "Setup Pages" 步骤报错 Get Pages site failed**
   - **现象：** actions/configure-pages@v4 步骤失败，提示 Please verify that the repository has Pages enabled and configured to build using GitHub Actions。
   - **原因：** GitHub 仓库的 "Settings" -> "Pages" 中尚未将部署源 (Source) 配置为 "GitHub Actions"。
   - **解决：**
     1. 进入 Frosti 仓库的 "Settings" -> "Pages"。
     2. 在 "Build and deployment" 部分，将 "Source" 修改为 "GitHub Actions"。
     3. 手动重新运行失败的 Actions workflow。
   
6. **部署成功，但 CSS 样式丢失：**
   - **现象：** 访问 https://[用户名].github.io/Frosti/，页面内容显示但无样式。
   - **原因：** astro.config.ts 中的 base: '/Frosti' 配置可能存在大小写问题或未正确生效。
   - **排查：**
     1. 浏览器查看网页源代码，检查 CSS 文件的 href 路径是否包含了正确的 /Frosti/ 前缀，并且大小写与仓库名 Frosti 一致。
     2. 确认 astro.config.ts 中 base: '/Frosti' (首字母大写F) 配置无误。
   - **解决：** 确保 base 配置的大小写与仓库名完全匹配后，重新提交并等待 Actions 部署。
   
   ## 五、 首次部署成功与样式问题修复 (回顾与确认)
   
   经过一系列的配置调整和问题排查 (详见上半部分)，博客最终成功部署到了 https://[你的用户名].github.io/Frosti/。
   
   - **关键配置回顾：**
     - frosti.config.yaml 中 user.site 设置为 https://[你的用户名].github.io。
     - astro.config.ts 中 site 来源于 USER_SITE (间接来自 frosti.config.yaml)，base 设置为 '/Frosti' (注意大小写与仓库名 Frosti 一致)。
     - GitHub Actions workflow (.github/workflows/deploy.yml) 配置正确，pnpm 版本与本地基本一致。
     - GitHub 仓库 "Settings" -> "Pages" 的 "Source" 设置为 "GitHub Actions"。
   - **CSS 样式丢失问题的最终解决：** 确认 astro.config.ts 中的 base: '/Frosti' 的大小写与仓库名 Frosti 完全匹配，确保了 CSS 等静态资源路径的正确性。
   
   此时，博客在子路径下已可正常访问，所有功能 (包括 Pagefind 搜索) 均按预期工作。
   
   ## 六、 迁移博客到根域名 [用户名].github.io
   
   为了使博客 URL 更简洁、更具代表性 (作为个人主站点)，决定将其从 /[仓库名]/ 子路径迁移到根域名 https://[用户名].github.io/。
   
   1. **在 GitHub 上修改仓库名称：**
   
      - 进入原 Frosti 仓库的 GitHub 页面。
      - 导航到 "Settings" -> "General"。
      - 将 "Repository name" 从 Frosti 修改为 [你的用户名].github.io (例如 hqslsz.github.io)。
      - 确认重命名。
   
   2. **修改 Astro 配置文件 (astro.config.ts) 以适应根路径部署：**
   
      - 在本地项目中，打开 astro.config.ts (或 .mjs) 文件。
      - **核心修改：** 将 base 选项的值从原来的 '/Frosti' 修改为 '/'。或者，由于 Astro 默认的 base 即为 '/'，也可以直接删除 base 这一行配置。
      - site 选项的值 (来源于 frosti.config.yaml 中的 user.site) 应保持为 https://[你的用户名].github.io，无需改动。
   
      修改后的 astro.config.ts 示例 (关键部分)：
   
      ```
      // astro.config.ts
      export default defineConfig({
        site: 'https://[你的用户名].github.io', // 保持不变
        base: '/',                             //  <--- 修改为根路径
        // ... 其他配置 ...
      });
      ```
   
      content_copydownload
   
      Use code [with caution](https://support.google.com/legal/answer/13505487).TypeScript
   
   3. **更新本地 Git 仓库的远程跟踪地址 (可选，GitHub Desktop 通常会自动处理)：**
   
      - 如果使用命令行，可以运行 git remote set-url origin https://github.com/[你的用户名]/[你的用户名].github.io.git。
   
   4. **提交并推送配置更改：**
   
      - 使用 GitHub Desktop，将对 astro.config.ts 的修改提交。
      - 提交信息可为：Update base path for root domain deployment 或 Configure for [用户名].github.io deployment。
      - 将提交推送到远程仓库 (此时远程仓库名已是 [你的用户名].github.io)。
   
   5. **GitHub Actions 自动重新部署：**
   
      - 代码推送到新的仓库名后，GitHub Actions 会自动使用相同的 workflow (.github/workflows/deploy.yml) 进行构建和部署。
      - 监控新仓库 ([你的用户名].github.io) "Actions" 标签页的 workflow 运行状态。
   
   6. **验证 GitHub Pages 设置 (通常无需更改)：**
   
      - 进入新仓库 ([你的用户名].github.io) 的 "Settings" -> "Pages"。
      - "Source" 应依然是 "GitHub Actions"。
      - 由于仓库名符合 [用户名].github.io 格式，GitHub Pages 会自动将其识别为用户/组织站点，并从根路径提供服务。
   
   ## 七、 访问新的博客主页与最终确认
   
   1. **等待 GitHub Actions 成功完成部署。** (可能需要几分钟)
   2. **新的博客访问地址为：https://[你的用户名].github.io/**
   3. **重要：** 清除浏览器缓存并强制刷新页面 (Ctrl+Shift+R 或 Cmd+Shift+R)，或使用隐身/无痕模式访问，以确保看到的是最新部署的内容和样式。
   4. **全面测试：**
      - 检查首页、文章列表页、文章详情页、分类/标签页、关于页面等是否都正常显示且样式正确。
      - 测试站内搜索功能 (Pagefind) 是否可用。
      - 测试亮色/暗色模式切换。
      - 检查所有内部链接和外部链接是否按预期工作。
      - 检查 RSS feed 是否能正确生成和访问。
   
   至此，个人博客已成功迁移到 [你的用户名].github.io 根域名下，并可稳定访问。
   
   ## 八、 总结与后续
   
   - **遇到的主要挑战及学习点：**
     - 理解 pnpm 的依赖管理和构建脚本审批机制。
     - 处理 package.json 脚本中的跨平台兼容性问题 (如 cp 命令)。
     - 正确配置 Astro 的 site 和 base 选项以适应不同的部署路径 (子目录 vs. 根域名)。
     - 熟悉 GitHub Actions 的基本配置和调试，特别是与 GitHub Pages 部署相关的步骤。
     - 排查 pnpm lockfile 在 CI 环境中的问题 (版本兼容性、文件存在性)。
     - 理解 GitHub Pages 的工作原理，特别是用户/组织站点与项目站点的区别。
   - **后续可做的事情：**
     - 持续撰写和发布博客内容。
     - 进一步定制主题样式和功能。
     - 学习和集成更多 Astro integrations (如评论系统、分析工具等)。
     - 考虑绑定自定义域名。
     - 备份重要配置和文章内容。
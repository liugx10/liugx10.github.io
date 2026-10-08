# Jekyll 使用指南

本文说明本站（https://liugx10.github.io）的构建方式、目录作用、本地预览方法，以及日常写作与发布流程。

> 本文档不会发布到网站上，已在 `_config.yml` 的 `exclude` 中排除。

---

## 一、网站是怎么跑起来的

- 这是一个 **GitHub Pages 用户站点**，仓库名 `liugx10.github.io` 必须与 GitHub 用户名一致，因此不需要配置 baseurl。
- 向 `main` 分支推送后，GitHub 会自动用 Jekyll 构建并把结果发布到 `https://liugx10.github.io`。**推送即发布**，没有单独的部署步骤。
- 线上构建环境由 GitHub Pages 固定，不能自己指定版本：

  | 组件 | 线上版本 |
  |---|---|
  | Jekyll | 3.10.x |
  | github-pages gem | v232 |
  | Minimal Mistakes | 4.28.1（由 `_config.yml` 里 `remote_theme` 的 `@` 后缀指定，GitHub 在每次构建时拉取） |

  （用 `bundle exec github-pages versions` 可以查本地对应的版本，官方列表在 https://pages.github.com/versions/）

因为版本被锁定，有两件事在线上做不到：

1. **非白名单插件不能用**。`_config.yml` 的 `plugins:` 里出现非白名单插件会导致线上构建失败。当前用到的 `jekyll-paginate`、`jekyll-sitemap`、`jekyll-gist`、`jekyll-feed`、`jekyll-include-cache` 都在白名单内。
2. **主题不能随便升级**。Minimal Mistakes 的 `gemspec` 要求 `jekyll >= 3.7, < 5.0`，升级主题前要确认它仍然兼容 Jekyll 3.10。

如果确实需要这些功能，要改用 GitHub Actions 自行构建（添加 `.github/workflows/` 工作流），而不是依赖 Pages 的默认构建。

### 主题是怎么装上来的

Minimal Mistakes 不在 GitHub Pages 的主题白名单里，用 `theme:` 直接指定会构建失败。正确做法是**远程主题**：

```yaml
remote_theme: "mmistakes/minimal-mistakes@4.28.1"
```

`remote_theme` 依赖的 `jekyll-remote-theme` 插件在白名单内，所以这套方案完全跑在 Pages 的默认构建上，不需要 Actions。

代价是**主题里的东西只有四个目录能用**——Jekyll 3.10 只从主题挂载 `_layouts`、`_includes`、`_sass`、`assets`，其余一律读不到。两个具体后果：

1. **主题的 `_config.yml` 不会被继承。** 主题需要的配置（`plugins`、`defaults`、`kramdown`、`permalink` 等）都得自己写全，不能指望从主题那边读到默认值。
2. **主题的 `_data/` 读不到。** 主题自带一份中英对照的 `_data/ui-text.yml`，但只有放在**你的仓库里**才会生效（`Theme#data_path` 是 Jekyll 4.x 才加的功能，线上是 3.10，没有）。所以仓库里有一份 `_data/ui-text.yml`，装的是主题界面文案的简体中文版；没有它，`locale: "zh-CN"` 会静默回退成英文，界面上就是 `Recent Posts`、`Next` 这些词，而且**不会有任何报错**。

这两条是从 minima 迁过来时最容易踩的坑。

---

## 二、目录结构与各文件夹作用

### 当前已有的文件

| 路径 | 作用 |
|---|---|
| `_config.yml` | 站点全局配置：标题、描述、主题、插件、文章默认值、排除文件等。**改完必须重启本地服务才生效。** |
| `index.html` | 首页。使用主题的 `layout: home`，会自动列出 `_posts/` 里的文章；front matter 里的 `author_profile: true` 让左侧显示作者资料卡（见第六节）。 |
| `_posts/` | 博客文章，一篇一个文件。 |
| `_data/ui-text.yml` | 主题界面文案的简体中文版（`最新文章`、`下一页`、`目录` 等 47 个键）。**必须存在**，见第一节末尾。想把界面上某个词换掉，改这里。 |
| `_includes/social-share.html` | 文章页底部的分享按钮（微博 / QQ 空间 / 微信、钉钉、飞书二维码 / 复制链接），覆盖主题自带的 X / Facebook / LinkedIn / Bluesky 版本。详见第六节「改分享按钮」。 |
| `assets/images/og-default.png` | 分享卡片的默认缩略图（1200×630），由 `_config.yml` 的 `og_image` 指向。目前是**没有文字的占位图**，换图直接替换文件即可。 |
| `README.md` | 仓库说明，已排除，不发布。 |
| `CLAUDE.md`、`JEKYLL_GUIDE.md` | 开发文档，已排除，不发布。 |

> 首页文件名**必须是 `.html`**。分页插件 `jekyll-paginate` 只把根目录下的 `index.html` 当作分页模板，改名成 `index.md` 后分页会被静默跳过（日志里只有一行 warning，网站上看不出来），超过 `paginate` 数量的文章就再也翻不到了。

### Jekyll 约定目录（目前只建了 `_data/`、`_includes/`、`assets/`，其余需要时自己新建）

Jekyll 靠**目录名**识别用途，新建后自动生效，通常不用改配置：

| 路径 | 作用 |
|---|---|
| `_layouts/` | 页面模板。Minimal Mistakes 提供 `default`、`home`、`single`、`archive`、`splash` 等（**没有 `post`**）；在这里新建**同名文件**即可覆盖主题的版本。 |
| `_includes/` | 可复用的 HTML 片段（页头、页脚、`head` 等），同样按文件名覆盖。**已创建**，目前只放了 `social-share.html`。 |
| `_sass/` | SCSS 片段。覆盖 `_sass/minimal-mistakes/_variables.scss` 可整体调配色和字体。 |
| `assets/` | 样式表和图片。放图片最常用的目录。**已创建**，目前有 `images/og-default.png`（分享卡片缩略图）和 `images/avatar-default.png`（侧栏头像），两张都是占位图。 |
| `_pages/` | 独立页面（关于、分类归档等）。已在 `_config.yml` 的 `include` 里声明，放进去就会被构建。 |
| `_data/` | YAML 数据文件，已存在（`ui-text.yml`）。再加 `_data/navigation.yml` 可以给页头加导航菜单。 |
| `_drafts/` | 草稿。文件名**不需要日期**，只有加 `--drafts` 参数预览时才可见，不会发布。 |
| `_site/` | **构建产物**，已加入 `.gitignore`。不要手工编辑，也不要提交。 |

覆盖主题文件的规则：只要路径和文件名与主题里的一致，就以你的为准。例如新建 `_layouts/single.html` 就会替换主题的文章页模板。

想直接看主题里的某个文件长什么样、照着改，可以临时把主题下载下来：

```bash
curl -L https://github.com/mmistakes/minimal-mistakes/archive/refs/tags/4.28.1.tar.gz | tar xz
```

---

## 三、本地环境搭建（只需做一次）

本机是 Alibaba Cloud Linux（dnf），安装步骤：

```bash
sudo dnf install -y ruby ruby-devel gcc gcc-c++ make
ruby -v            # 建议 3.x；github-pages 不支持 Ruby 4.0+
gem install bundler
```

国内下载 gem 很慢，建议换成阿里云镜像：

```bash
gem sources --add https://mirrors.aliyun.com/rubygems/ --remove https://rubygems.org/
```

然后在仓库根目录新建 `Gemfile`：

```ruby
source "https://rubygems.org"

# 锁定与线上一致的版本，本地预览才和网站一致
gem "github-pages", group: :jekyll_plugins

# Minimal Mistakes 必需，缺了会报 Unknown tag 'include_cached'
gem "jekyll-include-cache", group: :jekyll_plugins

# Ruby 3.x 不再自带 webrick，本地预览需要它
gem "webrick", "~> 1.8"
```

安装依赖：

```bash
bundle config mirror.https://rubygems.org https://mirrors.aliyun.com/rubygems/
bundle install
```

> 注意：只写 `gem "jekyll"` 是跑不起来的。`remote_theme` 要求 `jekyll-include-cache` 插件在场，而它的版本又要和线上对齐，所以用 `github-pages` 一个 gem 全带齐。
>
> `Gemfile.lock` 已加入 `.gitignore`（各机器平台不同，不提交更省事）。`Gemfile` 本身没有加入 `.gitignore`，但这个仓库当前没有提交它——远程主题不需要 Gemfile 就能在线上构建，本地预览才需要。

---

## 四、常用命令

```bash
bundle exec jekyll serve              # 启动本地服务，默认 http://127.0.0.1:4000
bundle exec jekyll serve --livereload # 改文件后自动刷新浏览器
bundle exec jekyll serve --drafts     # 连同 _drafts/ 里的草稿一起预览
bundle exec jekyll serve --host 0.0.0.0 --port 4000   # 需要从别的机器访问时

bundle exec jekyll build              # 只构建，输出到 _site/
bundle exec jekyll clean              # 清掉 _site/ 和缓存（构建结果诡异时先试这个）
bundle exec github-pages versions     # 查看线上锁定的各组件版本
```

改 `_config.yml`、加主题、改 `_layouts/` 后需要**重启服务**（Ctrl+C 再启动）；只改文章内容 `--livereload` 会自动刷新。

---

## 五、写一篇文章

1. 在 `_posts/` 下新建文件，命名必须是 `YYYY-MM-DD-英文短标题.md`，例如 `2026-09-23-my-first-post.md`。
2. 开头写 front matter：

```markdown
---
layout: single
title: "文章标题"
date: 2026-09-23
categories: 随笔
---

正文用 Markdown 写。
```

常用字段：

| 字段 | 说明 |
|---|---|
| `layout` | 文章写 `single`。**主题没有 `post` 这个 layout**，写错会构建告警并按无模板渲染。`_config.yml` 的 `defaults` 已经设好，其实可以整行省略。 |
| `title` | 标题，含空格或中文标点时用引号包起来。 |
| `date` | 写成 `YYYY-MM-DD` 即可。**必须与文件名一致**。 |
| `categories` | 分类，用空格分隔多个。注意它会出现在网址里（见下）。 |
| `tags` | 标签，只做展示。不配置归档页的话，文章顶部不会渲染标签。 |
| `excerpt` | 手动指定列表页显示的摘要。 |
| `toc` | 写 `true` 会在文章右侧生成目录。 |
| `toc_sticky` | 写 `true` 让目录跟随滚动。 |

几个容易踩的坑：

- **日期不能是未来时间。** 比如今天是 9 月 23 日，写成 `date: 2026-09-25` 的文章不会被构建，网站上完全看不到。线上构建按 **UTC** 计时，靠 `_config.yml` 里的 `timezone: "Asia/Shanghai"` 校正——**这行必须留着**，否则北京时间 0:00–8:00 推送的「当天」文章会被当成未来时间，页面上看不到、也不会有任何报错。
- **分类和文件名都会影响网址。** `_config.yml` 里配置的是 `permalink: /:categories/:title/`，所以 `_posts/2026-09-23-my-first-post.md` 配合 `categories: 随笔` 最终网址是 `/随笔/my-first-post/`（中文在浏览器里会显示成转义形式）。**修改已有文章的分类或改文件名，等于换了网址，原来的链接会 404。**
- **文章日期显示的格式**由 `_config.yml` 的 `date_format` 控制，当前是 `%Y 年 %-m 月 %-d 日`。
- 首页不需要手动维护列表，`index.html` 的 `layout: home` 会自动列出所有文章（每页 `paginate` 篇）。
- 文章里插入图片：把图片放进 `assets/`（自己新建），然后写 `![说明](/assets/图片名.png)`，路径以 `/` 开头是相对于网站根目录。

---

## 六、主题与自定义

当前主题是 **Minimal Mistakes 4.28.1**，以远程主题方式安装（见第一节末尾）。主题文件本身不在这个仓库里，改主题行为只能靠本仓库的 `_config.yml` 和「同名文件覆盖」两种方式。

### 换皮肤

改 `_config.yml` 这一行即可，改完重启服务：

```yaml
minimal_mistakes_skin: "default"
```

可选值：`default`、`air`、`aqua`、`contrast`、`dark`、`dirt`、`neon`、`mint`、`plum`、`sunrise`、`catppuccin_latte`、`catppuccin_mocha`。

### 加自定义样式

主题的样式表在 `assets/css/main.scss`。新建仓库里**同名同路径**的文件就会整个替换掉主题那份，所以必须把原来的两行 `@import` 抄上，再在后面追加自己的规则：

```scss
---
---

@import "minimal-mistakes/skins/{{ site.minimal_mistakes_skin | default: 'default' }}"; // 皮肤
@import "minimal-mistakes"; // 主题

// 在这里写自己的样式，会覆盖主题默认值
.site-title {
  font-size: 28px;
}
```

只是想把配色、字体整体调一调的话，更好的做法是覆盖变量：把主题的 `_sass/minimal-mistakes/_variables.scss` 原样复制到仓库的同名路径，改里面的 `$doc-font-size`、`$serif`、`$sans-serif` 等。

### 覆盖模板

想改页脚、页头，把主题里对应的文件复制到仓库再改。主题在 GitHub 上按目录浏览即可（`_layouts/`、`_includes/`），路径和文件名一致就会覆盖主题的版本。

注意 `_layouts/` 里**没有 `post.html`**，文章用的是 `_layouts/single.html`。

复制到仓库**同名路径**后即可自由修改。改动越少越不容易在主题更新后出问题。

### 改分享按钮

文章页底部的分享按钮来自 `_includes/social-share.html`。仓库里放着的那份**覆盖**了主题自带的版本（主题那份是 X / Facebook / LinkedIn，国内用不上）。按钮显示与否由 `_config.yml` 里 `_posts` 默认值中的 `share: true` 控制；首页用的是 `layout: home`，本身没有分享块。

加平台就是往 `<ul>` 里加一个 `<li>`，六个现有项分三类：

| 平台 | 实现方式 |
|---|---|
| 微博 | 跳到 `service.weibo.com/share/share.php`，网址和标题作为查询参数带上 |
| QQ 空间 | 跳到 `sns.qzone.qq.com/cgi-bin/qzshare/cgi_qzshare_onekey`，同上 |
| 微信 | 没有网页分享接口（腾讯早已关闭），只能弹二维码让读者扫 |
| 钉钉 | 同上，另外在弹层里给一条官方「统一跳转协议」深链 `dingtalk://dingtalkclient/page/link?url=…` |
| 飞书 | 同上，另外给一条官方 AppLink `https://applink.feishu.cn/client/web_url/open?mode=window&url=…` |
| 复制链接 | 十几行内联脚本，优先用 `navigator.clipboard`，本地 http 预览时退回 `execCommand` |

三个二维码内容相同（都是本页地址），只是提示文字不同；都由第三方接口 `api.qrserver.com` 现画现给，换服务商只改文件顶部 `qr_api` 那一行。钉钉和飞书那两条深链是**兜底**：桌面装了对应客户端才能唤起（钉钉还会开在 PC 侧边栏），没装的话点了没有任何反应，所以主入口始终是二维码。

分享出去的**卡片**（标题、描述、缩略图）不归这个文件管——那是主题的 `seo.html` 从 front matter 和 `_config.yml` 生成的：缩略图取 `_config.yml` 的 `og_image`（当前指向 `assets/images/og-default.png`，一张没有文字的占位图），单篇文章可以用 front matter 里的 `og_image` 覆盖；描述取 `description`，没写则用正文首段（`excerpt`）。**不设 `og_image` 的话，分享到微信/钉钉/飞书的卡片就没有缩略图。**

两个刻意的决定：**样式和脚本都内联在这个文件里**，不依赖 `_includes/head/custom.html` 之类的挂载点——本仓库没有本地构建，挂载点没生效的话样式会静默丢失且不报错；配色按 default 皮肤写死成浅色，将来换成 `dark` 等深色皮肤时要同步改 `.mm-share__popover` 的 `background` / `border` / `color`。

整份改动只有一个文件，**删掉它即还原成主题自带的分享按钮**。

### 侧栏的「作者资料卡」

文章页和首页左侧那一栏（主题的 `.sidebar`：实测宽 200px、超宽屏 300px，`opacity: .75` 悬停变不透明，滚动时吸顶）里装什么，由 `_config.yml` 的 `author` 块决定：

```yaml
author:
  name: "Liu Guoxuan"
  avatar: "/assets/images/avatar-default.png"
  bio: "记录技术与生活"
  links:
    - label: "GitHub"
      icon: "fab fa-fw fa-github"
      url: "https://github.com/liugx10"
    - label: "个人主页"
      icon: "fas fa-fw fa-home"
      url: "https://liugx10.github.io"
    - label: "邮箱"
      icon: "fas fa-fw fa-envelope"
      url: "mailto:liugx17@tsinghua.org.cn"
```

| 字段 | 说明 |
|---|---|
| `name` | 显示在侧栏，链到网站首页。 |
| `avatar` | 被裁成圆形、最大 **110px**（CSS 里 `.author__avatar img` 的 `max-width`，还带 5px 内边距和 1px 边框）。所以图要用**正方形、边长至少 220px**，否则糊。换头像直接替换同名文件。 |
| `bio` | 简介。 |
| `links` | 社交链接列表，`icon` 用 Font Awesome 类名（主题的 FA 是 CDN 上的 `@latest`，类名请先确认当前版本存在，否则页面上是空白方块）。邮箱这类写 `url: "mailto:地址"` 即可。**注意**：写进这里的每一项都会以明文出现在页面 HTML 里、会被爬虫采集——现在卡片只在首页渲染，所以只暴露在主页；哪天把 `author_profile` 加回 `defaults`，这些信息就会跟着出现在每一篇文章页上。 |

**谁在哪儿显示：只有首页。** 打开它的是 `index.html` front matter 里的 `author_profile: true`；`defaults` 里**故意不设**这一项，所以文章页和独立页面都没有左栏。因为卡片只在这一处渲染，写进 `links` 的内容（包括邮箱）也就只出现在主页。

**一个已知副作用（属正常，不是 bug）**：主题 CSS 是**无条件**给左栏留 200px 的（`.page` 是 `float: inline-end; width: calc(100% - 200px); padding-inline-end: 200px`），所以文章页没有卡片时，**正文宽度不变、左右各空 200px**，观感是正文变成居中的窄栏。**别用 `classes: wide` 去修**——它只把 `padding-inline-end` 归零，结果反而左右不对称（主题里 `:has()`、`without-sidebar` 之类的机制一个都没有，没有现成的"无左栏"形态）。真想改就用自定义样式覆盖 `.page` 的宽度。

### 加导航菜单

新建 `_data/navigation.yml`，顶层键必须是 `main`（这是 Minimal Mistakes 的格式，和 minima 不一样）：

```yaml
main:
  - title: "首页"
    url: /
  - title: "关于"
    url: /about/
```

### 加"关于"页面

新建 `_pages/about.md`（`_pages/` 已在 `_config.yml` 的 `include` 里声明）：

```markdown
---
layout: single
title: 关于
permalink: /about/
---

这里写自我介绍。
```

### 开启站内搜索

主题自带 lunr 搜索，不需要额外插件（`assets/js/lunr/` 里的索引文件和库都在主题内），三步：

1. `_config.yml` 里加 `search: true`。
2. 新建 `_pages/search.md`：

   ```markdown
   ---
   layout: search
   title: 搜索
   permalink: /search/
   ---
   ```

3. 重启服务，页头右上角会出现搜索按钮。

> **中文站点慎用：** lunr 的分词是按空格切的，中文正文会被整段当成一个词，搜索基本搜不出结果。要用的话得另外引入中文分词（主题不提供），或者换 Algolia（`search_provider: algolia`，需要额外注册和配置）。

---

## 七、发布流程

```bash
git add .
git commit -m "新增文章：xxx"
git push origin main
```

推送后 1～2 分钟内网站更新。如果几分钟后还是旧内容：

1. 打开仓库的 **Actions** 页或 **Settings → Pages**，看最近一次构建是否失败——构建失败时 GitHub 会发邮件，但推送命令本身不会报错。
2. 最常见的失败原因是 `_config.yml` 的 `plugins:` 里加了非白名单插件、`minimal_mistakes_skin` 写了不存在的皮肤名，或 front matter 的 YAML 格式错误（例如 `title:` 里有未转义的冒号）。
3. 本地 `bundle exec jekyll build` 能提前发现大部分问题（如文章日期写成未来时间、Markdown 语法错误）。

提交信息建议写清楚做了什么，本仓库历史上也在用 PR 合并到 `main` 的方式，两种都可以。

---

## 八、常见问题

| 现象 | 原因与处理 |
|---|---|
| 本地能看到文章，线上没有 | 文章 `date` 是未来时间（线上按 UTC 算，靠 `timezone: Asia/Shanghai` 校正，见第五节）；或文件没推送上去（`git status` 确认）。 |
| 改了 `_config.yml` 没变化 | 配置改动需要重启本地服务。 |
| 新文章不出现在首页 | 文件名日期格式不对（必须是 `YYYY-MM-DD-`）。 |
| 构建报 `Unknown tag 'include_cached'` | `_config.yml` 的 `plugins:` 里少了 `jekyll-include-cache`。 |
| 文章页完全不显示日期 | `_config.yml` 的 post `defaults` 里少了 `show_date: true`——Minimal Mistakes 默认不显示日期。 |
| 文章页显示「少于 1 分钟阅读」 | 阅读时长按空格分词统计，中文只能算成个位数。用 `read_time: false` 关掉（当前配置已关）。 |
| 文章排版全乱 / 主题完全没生效 | `_config.yml` 里同时写了 `theme:` 和 `remote_theme:`，两者只能留一个；或 `remote_theme` 的版本号写错。 |
| 首页翻不到第 2 页 | 首页文件名不是 `index.html`。`jekyll-paginate` 只认根目录下的 `index.html`。 |
| 页面样式完全丢失 | 自己建的 `assets/css/main.scss` 少了开头的空 front matter，或漏抄了那两行 `@import`。 |
| 本地和线上显示不一致 | 本地用了 Jekyll 4.x；改用 `github-pages` gem 才能对齐线上的 Jekyll 3.10 + Minimal Mistakes 4.28.1。 |
| 中文分类的网址很怪 | 这是 `permalink: /:categories/:title/` 的结果。把 `categories` 改成英文，或改 `permalink` 去掉分类段（注意会让现有链接 404）。 |
| 本地 `bundle install` 报 Ruby 版本错误 | 需要 Ruby 3.x；Ruby 4.0 目前不被 github-pages 支持。 |

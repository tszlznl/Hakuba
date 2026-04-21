# 模块与目录职责

## 1. 顶层目录

| 路径 | 职责 |
| --- | --- |
| `src/` | 站点源码（SvelteKit 路由、组件、业务逻辑、类型定义） |
| `.scripts/` | 构建期数据生成与构建后处理脚本 |
| `static/` | 静态资源（favicon 等） |
| `.github/workflows/` | CI 工作流（lint、CodeQL、自动合并等） |
| `svelte.config.js` / `vite.config.ts` | 框架构建配置（adapter-static、mdsvex、Vite 插件等） |

## 2. `src/` 目录

### 2.1 路由层：`src/routes/`

SvelteKit 路由目录直接决定站点 URL 结构与渲染方式。

- **列表页（首页/分页/标签页统一入口）**：[[...index=pageMatcher].svelte](file:///workspace/src/routes/%5B...index=pageMatcher%5D.svelte)
  - 解析参数：`src/params/pageMatcher.ts`（SvelteKit ParamMatcher）
  - 加载数据：`fetchPosts` / `fetchLabels`
  - 渲染组件：`AboutSection / LabelsSection / PostsSection / Pagination`
- **文章详情页**：[[post]@withoutHeader.svelte](file:///workspace/src/routes/post/%5Bpost%5D@withoutHeader.svelte)
  - 路由参数：`params.post`（可为 `metadata.path` 或 discussion number 字符串）
  - 加载数据：`fetchPost`，并用 `readableDate` 格式化时间
- **独立页面**：[[page].svelte](file:///workspace/src/routes/%5Bpage%5D.svelte)
  - 路由参数：`params.page`（匹配 `metadata.path` 或 `title.toLowerCase()`）
  - 加载数据：`fetchPage`
- **Atom feed**：[atom.xml.ts](file:///workspace/src/routes/atom.xml.ts)
  - 依赖：`feed` 包 + `fetchPosts`；缺少 `DOMAIN` 时 302 到 `/`
- **Markdown 布局**：[__layout-md.svelte](file:///workspace/src/routes/__layout-md.svelte)
  - mdsvex `layout` 配置指向该文件（见 [svelte.config.js](file:///workspace/svelte.config.js#L21-L33)）

### 2.2 组件层：`src/lib/components/`

以“页面骨架组件 + 内容组件 + 小型交互组件”为主，供路由页面组合。

- **文章容器**：[Article.svelte](file:///workspace/src/lib/components/Article.svelte)
- **分页**：[Pagination.svelte](file:///workspace/src/lib/components/Pagination.svelte)
- **评论（Giscus）**：[Giscus.svelte](file:///workspace/src/lib/components/Giscus.svelte)
- **页面元信息**：[PageMeta.svelte](file:///workspace/src/lib/components/PageMeta.svelte)
- **页脚与导航**：[Footer.svelte](file:///workspace/src/lib/components/Footer.svelte) / [Nav.svelte](file:///workspace/src/lib/components/Nav.svelte)

### 2.3 业务工具：`src/lib/helper/`

聚焦“从 Markdown 模块加载并组织数据”与“路由参数解析”。

- posts 聚合/分页/标签：
  - [fetchPosts.ts](file:///workspace/src/lib/helper/fetchPosts.ts)
  - `fetchPosts({ offset, limit, label })`：核心列表聚合与过滤
  - `fetchLabels()`：基于 posts 统计标签频次
  - `fetchPost(path)`：按 `metadata.path` 或 `number` 查找
- pages 聚合/排序：
  - [fetchPage.ts](file:///workspace/src/lib/helper/fetchPage.ts)
  - `fetchPages()`：过滤 `__error` 并按 `priority` 排序
  - `fetchPage(path)`：按 `metadata.path` 或 `title.toLowerCase()` 查找
- 路由参数匹配：
  - [matcher.ts](file:///workspace/src/lib/helper/matcher.ts) 提供 `generateMatcher()`
  - [pageMatcher.ts](file:///workspace/src/params/pageMatcher.ts) 组合出 `labelsMatcher/indexMatcher`

### 2.4 类型与常量

- 类型：
  - [BasePageType](file:///workspace/src/lib/types/base.ts)
  - [Post](file:///workspace/src/lib/types/post.ts)
  - [Page](file:///workspace/src/lib/types/page.ts)
- 运行时常量（来自 `VITE_*`）：
  - [constants.ts](file:///workspace/src/lib/constants.ts)

## 3. `.scripts/`（构建期脚本）

### 3.1 `.scripts/pre/`：构建前生成内容

入口：[.scripts/pre/index.ts](file:///workspace/.scripts/pre/index.ts)

- `fetcher.ts`：GitHub GraphQL 拉取 viewer 与 discussions
- `filter.ts`：按 Discussions Category 分类 + 解析配置 discussion
- `converter.ts`：合并/生成 front matter（YAML）
- `writer.ts`：写入 Markdown 源文件与 `.env.local`

### 3.2 `.scripts/post/`：构建后生成 sitemap

- 入口：[.scripts/post/index.ts](file:///workspace/.scripts/post/index.ts)
- 行为：若 `.env.local` 中 `VITE_DOMAIN` 存在则调用 `svelte-sitemap` 生成 sitemap


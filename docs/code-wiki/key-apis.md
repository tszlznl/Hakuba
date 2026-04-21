# 关键类型 / 函数 / 入口说明

## 1. 构建期（Data Generation）

### 1.1 `.scripts/pre/index.ts`：生成数据总入口

代码：[index.ts](file:///workspace/.scripts/pre/index.ts#L1-L48)

职责：

- 读取 `.env` / 环境变量（`dotenv.config()`）
- `fetchUser()` 获取 GitHub viewer 信息（login/url/bio）
- `fetchAllDiscussions()` 拉取目标仓库 discussions 列表
- `findConfig()` 从指定 category 的 `title=index` discussion 中解析 dotenv 文本为 config 对象
- 合并 config：优先级 `config[key]` → `process.env[key]` → 默认值（部分来自 viewer）
- `convertFrontMatter()` 把 discussions 元字段合入 Markdown front matter
- `filterPage/filterPost()` 按分类拆分 page/post
- `writePosts/writePages/writeEnv()` 落盘

输入（关键环境变量）：

- `GITHUB_TOKEN`：必需（GraphQL API bearer token）
- `REPOSITORY`：必需（仓库名，不含 owner）
- `CONFIG_CATEGORY/POST_CATEGORY/PAGE_CATEGORY`：可选（默认 `Config/Post/Page`）

输出（构建产物中的“源内容”）：

- `src/routes/post/_source/*.md`
- `src/routes/_page/*.md`
- `.env.local`（所有键写为 `VITE_*`）

### 1.2 `fetcher.ts`：GitHub GraphQL 拉取

代码：[fetcher.ts](file:///workspace/.scripts/pre/fetcher.ts#L7-L103)

- `fetchData<T>(query: string): Promise<T>`
  - 统一 GraphQL POST 调用
  - 若缺失 `GITHUB_TOKEN`，直接抛错（避免静默失败）
- `fetchUser()`
  - 查询 `viewer { login url bio }`
- `fetchDiscussions(owner, after?)`
  - 查询 `repository(owner, name: process.env.REPOSITORY).discussions(first: 100, after?, orderBy...)`
- `fetchAllDiscussions(user)`
  - 通过 `pageInfo.hasNextPage/endCursor` 循环分页拉取

### 1.3 `converter.ts`：Front matter 合成

代码：[converter.ts](file:///workspace/.scripts/pre/converter.ts#L4-L30)

- `splitMdx(mdx: string): [md: string] | [md: string, frontMatter: any]`
  - 使用 `--- ... ---` 分隔并 `YAML.parse`
- `convertFrontMatter(list)`
  - 把 `{ number, title, publishedAt, lastEditedAt, url, labels }` 与原 front matter 合并
  - 生成新的 `---\nYAML.stringify(frontMatter)---` 前缀并拼回 body

### 1.4 `filter.ts`：分类与配置

代码：[filter.ts](file:///workspace/.scripts/pre/filter.ts#L5-L29)

- `findConfig(list)`
  - 查找 `category.name === CONFIG_CATEGORY && title === 'index'` 的 discussion body
  - `dotenv.parse()` 解析为 key-value
- `filterPage(list)` / `filterPost(list)`
  - 根据 `PAGE_CATEGORY` / `POST_CATEGORY` 过滤

### 1.5 `writer.ts`：落盘

代码：[writer.ts](file:///workspace/.scripts/pre/writer.ts#L5-L33)

- `writePosts(list)`
  - 写入 `./src/routes/post/_source/{number}.md`
- `writePages(list)`
  - 写入 `./src/routes/_page/{title}.md`
- `writeEnv(config)`
  - 写入 `./.env.local`，键统一加 `VITE_` 前缀

## 2. 运行期（Site Runtime）

### 2.1 `fetchPosts.ts`：文章聚合与查询

代码：[fetchPosts.ts](file:///workspace/src/lib/helper/fetchPosts.ts#L9-L76)

- `fetchPosts({ offset?, limit?, label? })`
  - 通过 `import.meta.glob('../../routes/post/_source/*.md')` 枚举所有 markdown 模块
  - 并行 `await page()` 获取 `{ metadata, default: component }`
  - 按 `metadata.published` 倒序排序
  - `label` 过滤：`metadata.labels[].name` 包含目标标签
  - `offset/limit` 支持分页截取
  - 返回 `{ list, count }`
- `fetchLabels()`
  - 基于 `fetchPosts()` 的结果统计标签频次，返回 `Array<[label, count]>`
- `fetchPost(path)`
  - 按 `metadata.path === path` 或 `metadata.number` 等于 path 字符串匹配

### 2.2 `fetchPage.ts`：页面聚合与查询

代码：[fetchPage.ts](file:///workspace/src/lib/helper/fetchPage.ts#L9-L30)

- `fetchPages()`
  - 通过 `import.meta.glob('../../routes/_page/*.md')` 枚举页面 markdown 模块
  - 过滤 `metadata.title === '__error'`
  - 按 `metadata.priority` 倒序排序（用于 Footer/Nav 的展示优先级）
- `fetchPage(path)`
  - `path === metadata.path` 或 `path === title.toLowerCase()` 匹配

### 2.3 路由参数匹配：`generateMatcher` + `pageMatcher`

- `generateMatcher(paramPattern)`：[matcher.ts](file:///workspace/src/lib/helper/matcher.ts#L1-L11)
  - 基于 `path-to-regexp` 的 `match()` 生成“字符串路径 → params”解析器
- `listMatcher(param)`：[pageMatcher.ts](file:///workspace/src/params/pageMatcher.ts#L4-L7)
  - 支持两种形式：
    - `label/:label{/page/:page(\d+)}?`
    - `{page/:page(\d+)}?`
- `match(param)`：[pageMatcher.ts](file:///workspace/src/params/pageMatcher.ts#L8-L13)
  - 用于 SvelteKit `[...index=pageMatcher]` 路由校验：禁止 `page <= 1`

### 2.4 关键路由 `load()` 入口

- 列表页：[[...index=pageMatcher].svelte](file:///workspace/src/routes/%5B...index=pageMatcher%5D.svelte#L1-L46)
  - 从 `params.index` 得到 `{ label, page }`
  - 并行加载：`fetchPosts({ offset, limit, label })` 与 `fetchLabels()`
- 文章页：[[post]@withoutHeader.svelte](file:///workspace/src/routes/post/%5Bpost%5D@withoutHeader.svelte#L1-L37)
  - `fetchPost(params.post)`，不存在则 404
  - 用 `readableDate()` 格式化 `published/updated`
- 页面：[[page].svelte](file:///workspace/src/routes/%5Bpage%5D.svelte#L1-L18)
  - `fetchPage(params.page)`，不存在则 404

### 2.5 服务端 Hook：页面语言注入

代码：[hooks.server.ts](file:///workspace/src/hooks.server.ts#L6-L34)

- 通过 route id 分支决定是否读取 `metadata.lang`
  - `post/[post]@withoutHeader` → `fetchPost(params.post)?.metadata.lang`
  - `[page]` → `fetchPage(params.page)?.metadata.lang`
- 在 `resolve(..., { transformPageChunk })` 阶段替换 `app.html` 中的 `%lang%`

## 3. 核心类型（内容元数据）

- [BasePageType](file:///workspace/src/lib/types/base.ts#L1-L14)
  - “GitHub 元信息 + 自定义扩展”的统一基类接口
- [Post](file:///workspace/src/lib/types/post.ts#L1-L6)
  - `labels?: { name; color }[]`
  - `timezone?: string`
- [Page](file:///workspace/src/lib/types/page.ts#L1-L5)
  - `priority?: number`

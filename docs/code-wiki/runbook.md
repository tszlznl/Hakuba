# 运行与部署手册（Runbook）

## 1. 前置条件

- Node.js `>=22`（见 [package.json](file:///workspace/package.json#L4-L6)）
- 已启用 GitHub Discussions 的目标仓库
- 一个可访问 GraphQL API 的 Token（`GITHUB_TOKEN`）
  - 公共仓库通常需要 `public_repo` scope
  - 私有仓库通常需要 `repo` scope

## 2. 必需环境变量

| 变量 | 作用 | 用在何处 |
| --- | --- | --- |
| `GITHUB_TOKEN` | GraphQL 拉取 discussions | [fetcher.ts](file:///workspace/.scripts/pre/fetcher.ts#L7-L18) |
| `REPOSITORY` | 目标仓库名（不含 owner） | [fetcher.ts](file:///workspace/.scripts/pre/fetcher.ts#L57-L61) |

建议将它们放在本地 `.env`（不提交到仓库）。

## 3. 可选配置（生成到 `VITE_*`）

构建前脚本会把配置写到 `.env.local`（[writeEnv](file:///workspace/.scripts/pre/writer.ts#L28-L33)），前端通过 [constants.ts](file:///workspace/src/lib/constants.ts) 读取。

常用键（不区分来源：可来自环境变量或“配置 discussion”）：

- `CONFIG_CATEGORY`：配置 category（默认 `Config`）
- `POST_CATEGORY`：文章 category（默认 `Post`）
- `PAGE_CATEGORY`：页面 category（默认 `Page`）
- `PAGE_SIZE`：列表分页大小
- `BLOG_NAME` / `BIO` / `EMAIL` / `TWITTER` / `DESCRIPTION` / `KEYWORDS`
- `DOMAIN`：站点域名；影响 Atom feed 与 sitemap
- `LANGUAGE`：默认语言（用于 HTML `lang`）
- `COMMENT`：默认是否启用评论
- `TIMEZONE`：日期格式化时区

配置优先级见 [.scripts/pre/index.ts](file:///workspace/.scripts/pre/index.ts#L18-L35)：

1. 配置 discussion（`findConfig()` 解析结果）
2. 环境变量 `process.env.*`
3. 内置默认值（部分来自 GitHub viewer）

## 4. GitHub Discussions 约定

### 4.1 分类（Categories）

默认使用三个分类（可用环境变量覆盖）：

- `Config`：仅使用 `title=index` 的 discussion 作为配置文本（dotenv 格式）
- `Post`：文章 discussions
- `Page`：页面 discussions

分类筛选逻辑见 [filter.ts](file:///workspace/.scripts/pre/filter.ts#L5-L29)。

### 4.2 内容落盘路径

由构建前脚本生成：

- posts：`src/routes/post/_source/[discussion number].md`
- pages：`src/routes/_page/[title].md`

写入逻辑见 [writer.ts](file:///workspace/.scripts/pre/writer.ts#L5-L26)。

## 5. 本地开发

```bash
npm ci
npm run generateData
npm run dev
```

说明：

- `generateData` 需要能访问 GitHub API，否则会在 `fetcher.ts` 抛错
- `dev` 运行时读取的是本地生成的 markdown 源文件

## 6. 构建与预览

```bash
npm run build
npm run preview
```

- `build` 会依次执行：
  - `.scripts/pre/index.ts`（生成 markdown + `.env.local`）
  - `vite build`（SvelteKit 静态构建，输出到 `build/`）
  - `.scripts/post/index.ts`（若设置了 `DOMAIN` 则生成 sitemap）
  - 见 [package.json scripts](file:///workspace/package.json#L7-L18)

## 7. 部署

该项目输出目录为 `build/`，适合部署到静态站点托管：

- Vercel / Netlify / Cloudflare Pages 等

建议：

- 在托管平台设置环境变量 `GITHUB_TOKEN`、`REPOSITORY`（以及可选配置）
- 触发构建时执行 `npm run build`
- 如需“内容更新自动触发部署”，可使用 GitHub Discussions 的 webhook + 平台的 deploy hook（参考仓库 [README](file:///workspace/README.md#L73-L99)）

## 8. 常见问题定位

- 构建前脚本报 `GitHub Token 未设置`：检查 `GITHUB_TOKEN`
- `fetchDiscussions` 取不到仓库：检查 `REPOSITORY` 是否正确、token 权限是否足够
- Atom feed 不生效：检查是否设置 `DOMAIN`（缺失会 302 回 `/`，见 [atom.xml.ts](file:///workspace/src/routes/atom.xml.ts#L6-L10)）
- sitemap 不生成：检查 `.env.local` 里是否存在 `VITE_DOMAIN`（见 [.scripts/post/index.ts](file:///workspace/.scripts/post/index.ts#L5-L12)）


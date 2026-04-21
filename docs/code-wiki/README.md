# Code Wiki（Hakuba / svelte-blog）

本 Wiki 面向仓库维护者与二次开发者，聚焦“代码怎么组织、数据怎么流、如何运行/部署、关键模块与接口”。

## 1. 项目定位

- **类型**：基于 **SvelteKit + TypeScript** 的静态博客/站点
- **数据源**：GitHub Discussions（GraphQL API 拉取 discussions 内容）
- **核心机制**：构建前脚本把 discussions 转成本地 Markdown 源文件（被 mdsvex 编译为 Svelte 组件）；运行时通过 `import.meta.glob` 聚合这些 Markdown 模块并渲染路由页面

## 2. 快速索引

- [整体架构与数据流](./architecture.md)
- [模块与目录职责](./modules.md)
- [关键类型/函数/入口说明](./key-apis.md)
- [依赖关系（包/模块）](./dependencies.md)
- [运行与部署手册（Runbook）](./runbook.md)

## 3. 关键入口一览

- **构建前数据生成**：`npm run generateData` → [.scripts/pre/index.ts](file:///workspace/.scripts/pre/index.ts)
- **完整构建**：`npm run build`（生成数据 → `vite build` → sitemap）→ [package.json](file:///workspace/package.json)
- **站点路由入口**：
  - 列表页/分页/标签：[[...index=pageMatcher].svelte](file:///workspace/src/routes/%5B...index=pageMatcher%5D.svelte)
  - 文章详情：[[post]@withoutHeader.svelte](file:///workspace/src/routes/post/%5Bpost%5D@withoutHeader.svelte)
  - 独立页面：[[page].svelte](file:///workspace/src/routes/%5Bpage%5D.svelte)
  - Atom Feed：[atom.xml.ts](file:///workspace/src/routes/atom.xml.ts)

# Pinia 入门 · 案例分析

> 四个真实事件，看 Pinia 是怎么从一个实验项目变成 Vue 官方默认方案的，以及「选错写法」在真实项目里会付出什么代价。

## Case 1：Pinia 如何取代 Vuex 成为官方推荐

### 背景

2019 年，Vue 3 的组合式 API 还在 RFC 阶段。Vuex 4 虽然能跑在 Vue 3 上，但它的架构是从 Vue 2 时代延续下来的：`mutations` 必须同步、`actions` 必须提交 mutation、TypeScript 类型推断几乎不可用。Vue 核心团队成员 Eduardo San Martin Morote 当时的判断是：如果继续在 Vuex 4 上打补丁，组合式 API 的优势会被这层老架构吃掉。

### 事件经过

2019 年 11 月，Pinia 以 `pinia` 的名字出现在 GitHub 上，最初的设计目标就三条：去掉 mutations、完整支持 TypeScript 类型推断、支持组合式 API。2021 年 8 月，Pinia 的 API 稳定；2022 年 2 月 7 日，Vue 官方博客发布 [Vue 3 as the New Default](https://blog.vuejs.org/posts/vue-3-as-the-new-default.html)，同一天 Vue 官方文档把 Pinia 列为推荐状态管理方案，Vuex 状态从「推荐」改为「维护模式」。2022 年 3 月，Pinia v2.0.0 发布，作为与 Vue 3 配套的稳定版本。

这个决定在社区里争议了一段时间，主要反对意见是「又一个新东西要学」。官方的回应是：`create-vue` 脚手架里 Pinia 是默认勾选项，且 Vuex 到 Pinia 的迁移路径被写进了官方文档，不需要重写业务逻辑。

### 与本章的关系

本章第一节说「Pinia 不是 Vuex 的升级版，而是一次重新设计」，依据就是这段历史：`mutations` 的消失不是简化，而是因为组合式 API 让「同步修改入口」这个约束失去了必要性。理解了这一点，才不会在新项目里照着老教程补一层 mutations。

### 参考资料

- [Vue 3 as the New Default（Vue 官方博客）](https://blog.vuejs.org/posts/vue-3-as-the-new-default.html)
- [Pinia v2.0.0 Release](https://github.com/vuejs/pinia/releases/tag/v2.0.0)
- [Pinia 官方文档：与 Vuex 对比](https://pinia.vuejs.org/zh/cookbook/migration-vuex.html)

## Case 2：create-vue 默认脚手架带来的写法统一

### 背景

2020 年前后，Vue 3 项目的创建方式有三种：`vue-cli`、`vite` 手动配置、以及社区模板。三种方式生成的目录结构、别名配置、环境变量前缀都不一样（`VUE_APP_` 与 `VITE_` 的差异就来自这里）。团队换一个模板，`main.js` 就要重写一遍。

### 事件经过

2022 年，Vue 官方把 `create-vue` 定为推荐脚手架，它基于 Vite，交互式的选项里包含 Router、Pinia、TypeScript、Vitest、ESLint 等。选择带 Pinia 时，生成的项目里 `src/stores/` 目录、`main.js` 里的 `app.use(createPinia())`、以及一个 `counter.js` 示例 store 都已经写好。到 2023 年，`vue-cli` 进入维护模式，官方文档不再推荐新项目使用。

这个变化的直接后果是：网上大量 2021 年之前的 Vuex 教程与 2023 年之后的 Pinia 教程混在一起，初学者照着老教程在新脚手架里写 `store/index.js`，会发现 `main.js` 里根本没有 `store` 的引用。这是搜索引擎时代特有的问题。

### 与本章的关系

本章第二节强调「注册顺序必须在 mount 之前」，这个顺序正是 `create-vue` 生成代码里的样子。看一遍脚手架生成的真实 `main.js`，比自己推演更容易记住。同时也解释了为什么本章的示例 store 都放在 `stores/` 目录：这是脚手架的默认约定，跟 `index.html` 无关，但团队统一后能省掉很多找文件的时间。

### 参考资料

- [create-vue 官方仓库](https://github.com/vuejs/create-vue)
- [Vue 官方文档：快速上手](https://cn.vuejs.org/guide/quick-start.html)
- [Vite 官方文档：环境变量与模式](https://cn.vite.dev/guide/env-and-mode.html)

## Case 3：小兔鲜用 category store 把三次请求压成一次

### 背景

小兔鲜前台的顶部导航需要一份完整的分类数据（一级分类、二级分类、对应图标）。按章节顺序开发时，`Layout` 页要渲染导航，`一级分类页` 要渲染侧边分类，`二级分类页` 要渲染面包屑，三个页面各自写了「进入页面就请求分类接口」。结果是用户在首页点两下，同一个接口被调了三次，每次要等 300 毫秒左右，首屏肉眼可见地慢。

### 事件经过

改造的切入点是把请求从组件搬进 `category` store。store 里维护一个 `list` 和一段 `fetchList()`，其中加了一个前置判断：`if (list.value.length) return list.value`。三个组件改成只调用 `store.fetchList()`，不再自己发请求。第一个进入的页面触发真实请求并把结果存进 store，后两个页面命中缓存，几乎不耗时。

紧接着出现了两个新问题。其一是不同页面对数据形状要求不同：导航要一级分类加图标，分类页要两级树。解法是不在 store 里存两个版本，而是存原始列表，用 `getters` 派生 `topLevel`。其二是刷新页面后 store 清空，第一个页面照样要请求一次，这属于预期行为，不需要持久化——分类数据变化不频繁但也不该被长期缓存。

### 与本章的关系

这个案例同时用到了本章的三个点：`action` 里放「怎么取、取完存哪」（第六节）、`getters` 里放「派生视图」（第五节）、「只在 store 里请求一次」（第六节的职责边界）。也说明了一个判断标准：**能被多个页面共享的请求，才值得放进 store**。只在一个页面用的请求放进 store 反而是负担，因为你还得额外管理它的清理时机。

### 参考资料

- [Pinia 官方文档：在 store 之外使用 store](https://pinia.vuejs.org/zh/cookbook/composables.html)
- [Vue Router 官方文档：路由懒加载](https://router.vuejs.org/zh/guide/advanced/lazy-loading.html)
- [MDN：HTTP 缓存](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching)

## Case 4：`$reset` 不适用于组合式 store 引发的社区讨论

### 背景

Pinia 有两种 store 写法，官方文档在 API 章节里写明：`$reset()` 只在选项式 store 上可用。但这条信息藏在 API 文档深处，而大量教程先教组合式写法、后提 `$reset`，导致很多人在退出登录时写下 `store.$reset()`，运行时直接抛 `store.$reset is not a function`。

### 事件经过

在 Pinia 的 GitHub 仓库中，关于「setup store 是否应该支持 $reset」的讨论从 v2 发布后一直存在。维护方的立场是：组合式 store 里的 state 是一组独立的 `ref`，框架没有办法知道哪些 `ref` 属于 state、哪些只是局部变量，因此无法自动重置。折中方案是要求开发者在组合式 store 里手动 `return` 一个 `reset` 函数。文档里给出的示例就是把每个 `ref` 重新赋初值。

实践中还发现了第二层问题：即使手动实现了 `reset`，如果 store 接了 `pinia-plugin-persistedstate`，本地存储里仍然是旧值，刷新后数据会「复活」。正确顺序是先清本地存储再重置内存，或者干脆在插件的配置里为这个 store 关掉持久化。

### 与本章的关系

本章第四节与第八节第四条坑说的就是这件事。这个案例的价值在于它展示了「框架能力的边界」：Pinia 的自动化建立在它能识别 state 的基础上，而组合式写法打破了这种可识别性。理解这一点，就不会在遇到类似问题时反复怀疑是自己写错了。

### 参考资料

- [Pinia 官方文档：Option Stores](https://pinia.vuejs.org/zh/core-concepts/#option-stores)
- [Pinia 官方文档：Setup Stores](https://pinia.vuejs.org/zh/core-concepts/#setup-stores)
- [pinia-plugin-persistedstate 文档](https://prazdevs.github.io/pinia-plugin-persistedstate/zh/)

# Layout 页 · 案例分析

> 四个真实的技术背景，解释「外壳为什么这样设计」，以及重复请求在真实项目里的代价。

## Case 1：嵌套路由成为前台标配的历史

### 背景

Vue Router 3（Vue 2 时代）已经支持 `children`，但当时的主流写法是每个页面各自包含头部与底部组件。原因是 Vue 2 的组件通信靠 `$emit` 与 `provide/inject`，把页面塞进外壳后，页面要访问外壳的数据会绕很多层。所以很多团队选择「页面自治」，用复制结构换取耦合度低。

### 事件经过

Vue 3 与 Vue Router 4 一起发布后，组合式 API 让跨层共享数据变得便宜：`provide/inject` 可以直接传 `ref`，Pinia 也能在任何组件里 `useXxxStore()`。这消除了「页面进外壳就难以取数据」的顾虑，嵌套路由开始成为前台的标准结构。Vue Router 4 的官方文档也把「嵌套路由」放在基础章节而不是进阶章节，说明它已经是默认写法。

同期出现的一个配套变化是命名视图的价值下降。Vue Router 3 时代常用多个具名 `router-view` 做布局组合，Vue 3 之后大多数项目改用「一个外壳组件 + 一个默认出口」，因为外壳本身已经是组件，不需要靠命名视图拼接。

### 与本章的关系

本章第一节的 Layout 结构就是这个演进的产物。理解「为什么老项目不这么做」，才能在接手旧代码时看懂它的目录为什么是反的，而不是一上手就重构。

### 参考资料

- [Vue Router 官方文档：嵌套路由](https://router.vuejs.org/zh/guide/essentials/nested-routes.html)
- [Vue Router 官方文档：命名视图](https://router.vuejs.org/zh/guide/essentials/named-views.html)
- [Vue 官方文档：依赖注入](https://cn.vuejs.org/guide/components/provide-inject.html)

## Case 2：从雪碧图到 iconfont 再到 SVG

### 背景

2000 年代末，网页图标的通行做法是雪碧图（CSS Sprite）：把几十个小图拼成一张大图，用 `background-position` 取其中一块。它能减少 HTTP 请求数——在 HTTP/1.1 时代这是最重要的性能优化之一。

### 事件经过

2012 年前后，阿里巴巴的 iconfont 平台上线，把图标做成字体字形。它的优势直接击中了雪碧图的痛点：换颜色不用重新切图（改 `color`）、缩放不失真（矢量）、一个文件装几十个图标。到 2015 年，iconfont 在国内前端项目里基本成为默认方案。

2018 年之后，SVG 组件的方案开始流行，原因是设计工具（Sketch、Figma）能直接导出 SVG，且 SVG 支持多色、渐变、动画。但它的代价是每个图标都要 `import`，且着色要用 `fill` 而不是 `color`，团队规范不统一时容易出现「一半图标能换色一半不能」。

到了 HTTP/2 普及之后，雪碧图的「减少请求数」优势基本消失，它退出了主流选择。今天的选择基本是：单色、数量多、要频繁换色 → iconfont；多色、需要动画、数量少 → SVG 组件。

### 与本章的关系

本章第三节选 iconfont，依据就是「导航图标是单色且要跟随主题变色」这个具体条件。案例的价值在于给出边界：如果哪天产品要求图标带渐变，就不该硬用 iconfont 再想办法，而应该换 SVG。

### 参考资料

- [iconfont 官网](https://www.iconfont.cn/)
- [MDN：CSS Sprite 与 background-position](https://developer.mozilla.org/zh-CN/docs/Web/CSS/background-position)
- [MDN：SVG 与 fill](https://developer.mozilla.org/zh-CN/docs/Web/SVG/Attribute/fill)

## Case 3：小兔鲜把三次分类请求合并成一次

### 背景

小兔鲜首屏由 Layout 与 Home 组成。Layout 的导航需要分类数据，Home 的分类楼层也需要，用户点进分类页后侧边栏还需要。三次请求指向同一个接口，每次耗时 200 到 400 毫秒。

### 事件经过

第一轮改造把请求搬进 `category` store，加了「已有数据就返回」的判断。效果是：从首页进分类页时不再重复请求。但首屏的并发请求数没有下降——因为 Layout 与 Home 是同时挂载的，两者都在 `list` 还是空的时候调了 `fetchList`。

第二轮改造缓存了「进行中的 Promise」：第一次调用把 Promise 存进模块级变量，后续调用直接返回同一个 Promise。这样首屏的三次调用共享一次请求，接口调用数从 3 降到 1。同时加了 `loading` 状态用于骨架屏，以及一个独立的 `refresh()` 用于强制刷新。

第三轮遇到的问题是「刷新了但列表没变」：`refresh()` 清空了 `list` 却没有清空 `pending`，导致在请求进行中调用 `refresh` 会拿到旧的 Promise。修法是在 `refresh` 里把 `pending` 也置空，并在 Promise 完成后校验版本号。

### 与本章的关系

本章第六节把「结果缓存」与「请求去重」分开讲，就是因为这两轮改造解决的是不同问题：前者省掉重复的网络往返，后者解决并发。只做前者，首屏依然是三次请求。

### 参考资料

- [MDN：Promise](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)
- [VueUse：useFetch 的去重实现思路](https://vueuse.org/core/useFetch/)
- [Pinia 官方文档：Setup Stores](https://pinia.vuejs.org/zh/core-concepts/#setup-stores)

## Case 4：effectScope 解决了「组合式函数怎么统一清理」

### 背景

Vue 3.0 的组合式 API 让逻辑可以抽成函数复用，但带来一个新问题：如果一个组合式函数内部创建了多个 `watch`、`computed`，调用方想「把它们一起停掉」时没有办法。`watch` 返回的停止函数是零散的，调用方得逐个保存、逐个调用，漏一个就是泄漏。

### 事件经过

Vue 3.2（2021 年 8 月）引入了 `effectScope` API：它把一组响应式副作用收集到一个作用域里，调用 `scope.stop()` 一次性停掉全部。这个 API 的直接动机就是给组合式库作者用的——VueUse 的 `useScroll`、`useIntersectionObserver` 都在内部依赖它。

同一时期 Vue 也把 `effectScope` 用在了 Pinia 上：`createPinia()` 创建的实例持有一个 scope，所有 store 的 `computed` 与 `watch` 都注册在里面，`app.unmount()` 时一起停止。这就是第 2 章说的「总电闸」。

对普通业务开发者的影响是：**只要用 VueUse 或自己写 composable 时用了 `onScopeDispose`，就不需要手动清理**。这也解释了为什么「`onUnmounted` 必须在 setup 同步执行期间调用」——它需要一个当前组件实例或作用域来挂载。

### 与本章的关系

本章第七节的 `useSticky`、`useCountDown` 都建立在「组合式函数能自动清理」这个前提上。如果不知道有 `effectScope`，很容易写出「自己 `new` 一个监听器然后忘了拆」的代码；知道了之后，就会优先选择「有自动清理」的封装方式。

### 参考资料

- [Vue 官方文档：effectScope](https://cn.vuejs.org/api/reactivity-advanced.html#effectscope)
- [Vue 官方文档：onScopeDispose](https://cn.vuejs.org/api/reactivity-advanced.html#onscopedispose)
- [VueUse：useScroll 源码](https://github.com/vueuse/vueuse/blob/main/packages/core/useScroll/index.ts)

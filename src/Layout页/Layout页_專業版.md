# Layout 页

> Layout 是全站唯一的「外壳」：它决定每个页面共享什么、页面自己能改什么。外壳设计错了，改一次导航要动十个文件。

## 一、Layout 的职责：一个壳子管住全站

电商前台有一个共同特征：几乎所有页面都长着同一个头、同一个尾。顶部有一条导航带、下面是搜索与购物车、再下面是内容、最下面是帮助条款。如果每个页面各自复制一遍这套结构，改动一次就要改十个文件，而且很容易出现「购物车页的头部比首页矮两像素」这种说不清的问题。

Layout 就是把「所有页面共享的部分」抽出来做成一个组件，内容区留一个出口（`router-view`），由路由决定往出口里塞哪个页面。这是嵌套路由的直接应用：`/` 这一层的组件是 Layout，它的 `children` 才是首页、分类页、详情页、购物车。

```js
{
  path: '/',
  component: () => import('@/views/Layout/index.vue'),
  children: [
    { path: '', component: () => import('@/views/Home/index.vue') },
    { path: 'category/:id', component: () => import('@/views/Category/index.vue') },
    { path: 'detail/:id', component: () => import('@/views/Detail/index.vue') },
    { path: 'cart', component: () => import('@/views/Cart/index.vue') },
  ],
}
```

哪些页面不进 Layout？登录页、支付页、会员中心这类需要整页排版的。判断标准很实用：**这个页面需不需要顶部导航**。需要就放进 `children`，不需要就单独一条顶层路由。小兔鲜的支付页是独立路由，因为付款时不该给用户到处乱点的入口；会员中心也是独立路由，因为它有自己的左侧菜单。

Layout 里要放几块？至少三块：`LayoutNav`（顶部一级导航）、`LayoutHeader`（搜索、购物车、二级导航）、`LayoutFooter`（帮助条款）。再加一个 `LayoutFixed` 做吸顶效果。这个拆分不是为了好看，而是因为它们的更新频率不同：导航数据来自 store、头部要读购物车角标、底部基本静态、吸顶头有自己的显隐逻辑。混在一个文件里，改任何一个都要重读两百行。

```html
<template>
  <LayoutFixed />
  <LayoutNav />
  <LayoutHeader />
  <div class="main"><RouterView /></div>
  <LayoutFooter />
</template>
```

注意这里的 `<RouterView />` 没有 name，表示渲染默认视图。如果某个 Layout 需要同时渲染多个视图（比如带侧栏的后台），才用 `name` 指定；前台不需要，加了反而多一层心智负担。

## 二、组件结构快速搭建：目录与插槽

小兔鲜的 Layout 目录长这样：`src/views/Layout/` 下放 `index.vue` 与 `components/`，`components/` 里是 `LayoutNav.vue`、`LayoutHeader.vue`、`LayoutFixed.vue`、`LayoutFooter.vue`。这个结构在所有页面级目录里保持一致，找文件时有肌肉记忆。

为什么 `components/` 放在 `views/Layout/` 里面，而不是 `src/components/`？因为这几个组件只服务于 Layout，没有任何其他页面会用到。放在 `src/components/` 会让那个目录变成杂物间，时间一长没人敢删。判断标准是：**只被一个页面用的，跟着页面走；被两个以上页面用的，进全局组件库**。

Layout 里需要插槽吗？严格说不需要，因为内容区已经由 `router-view` 承担。但有一种场景会用：如果某个页面需要往头部塞一个「品牌专区」的横幅，而其他页面不需要，这时可以在 LayoutHeader 里留一个具名插槽，由页面通过 `Teleport` 或路由 `meta` 控制。

```html
<!-- LayoutHeader.vue 里留一个可选区域 -->
<slot name="banner"></slot>
```

```html
<!-- 页面里按需填充，不填就不渲染 -->
<Teleport to="#layout-banner" v-if="isBrandPage">
  <BrandBanner />
</Teleport>
```

多数项目用不到这一步。写 Layout 的第一原则是「宁可少留口子」，每多一个插槽，就多一处「这个页面为什么和其他页面不一样」的排查成本。

布局用 Flex 还是 Grid？前台这种「头 + 内容 + 尾」的纵向结构，用 Flex 的 `column` 最直接，配合 `min-height: 100vh` 让短内容页也能把底部推到底。

```css
.layout {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}
.main { flex: 1; }
```

`flex: 1` 是关键：没有它，内容少的页面（比如空购物车）会让底部浮在屏幕中间，下半屏留白。这是 Layout 最常见的视觉 bug，几乎每个项目都会踩一次。

## 三、字体图标：为什么用 iconfont 而不是 svg 组件

导航上有几十个小图标：首页、分类、购物车、用户。可选方案有三种：一张雪碧图、每个图标一个 SVG 组件、字体图标（iconfont）。小兔鲜选第三种。

字体图标的本质是把图形写成字形，像文字一样用 `font-family` 渲染。好处有三：颜色和大小用 CSS 直接控制（`color`、`font-size`），不需要准备多套切图；多色图标用不了，但导航图标本来就单色；一个字体文件装几十个图标，体积远小于几十个 SVG。

```css
@font-face {
  font-family: 'iconfont';
  src: url('@/assets/fonts/iconfont.woff2') format('woff2');
}
.iconfont {
  font-family: 'iconfont';
  font-style: normal;
}
```

```html
<i class="iconfont icon-home"></i>
```

三个使用要点。第一，`font-style: normal` 不能省，`<i>` 标签默认斜体，图标会跟着歪。第二，字形码要与字体文件对应，从 iconfont 平台下载时它会给你一份 `iconfont.css`，里面是 `content: "\e600"` 这类码位，直接抄，不要自己猜。第三，图标与文字对齐经常差一两像素，用 `vertical-align: middle` 或包一层 Flex 修正。

```css
.nav-item {
  display: flex;
  align-items: center;
  gap: 4px;
}
```

为什么不选 SVG 组件？因为 SVG 组件在每处都要 `import`，而且改颜色要通过 `fill` 而不是 `color`，团队里容易用得不一致。为什么不选雪碧图？因为换颜色要重新导出切图，改一次设计要重新走一遍流程。字体图标在「单色、数量多、要换色」这三个条件下是最优解；如果图标需要多色或渐变，那就该用 SVG。

## 四、一级导航渲染：分类数据的消费方

顶部导航的内容来自分类接口，不是写死的。渲染流程是三步：拿到数据、选出一级分类、渲染成链接。

```js
const categoryStore = useCategoryStore()
const { list } = storeToRefs(categoryStore)
onMounted(() => categoryStore.fetchList())
```

```html
<RouterLink
  v-for="item in list.slice(0, 9)"
  :key="item.id"
  :to="`/category/${item.id}`"
>
  {{ item.name }}
</RouterLink>
```

`slice(0, 9)` 是有意的：导航条宽度有限，分类可能有二十个，全渲染会换行或溢出。截断之后剩下的从「全部分类」入口进。这个细节看起来是样式问题，实际是产品决策——导航条上放几个，应该在需求阶段定，而不是让 CSS 溢出隐藏悄悄砍掉。

`RouterLink` 自带一个能力：当前路由匹配时会给链接加上 `router-link-active` 类。导航的高亮就靠它，不需要自己监听路由变化。

```css
.router-link-active { color: #ff6b00; font-weight: 600; }
```

需要注意精确匹配与包含匹配的区别。`/category/101` 访问时，指向 `/category/101` 的链接是精确匹配，指向 `/category` 的也会被算作「包含」。如果一级导航同时有「分类」和「分类/101」两个入口，高亮会同时命中两个。解法是给外层加 `exact-active-class` 或者用 `:class` 自己判断。

```html
<RouterLink :to="`/category/${item.id}`" active-class="nav-active">
```

数据来自 store 而不是组件内请求，这是本章第六节要展开的关键：Layout 的导航、分类页的侧栏、二级页的面包屑都需要这份数据，三处各自请求就是三次网络往返。

## 五、吸顶交互：滚动监听与状态切换

用户往下滚时，导航条要固定在顶部；滚回顶部，它会以过渡动画落回原位。小兔鲜的做法是：准备一个独立的 `LayoutFixed` 组件，只在滚动超过阈值时显示。

判断逻辑用 VueUse 的 `useScroll`：

```js
import { useScroll } from '@vueuse/core'
const { y } = useScroll(window)
const show = computed(() => y.value > 78)
```

```html
<div class="fixed-nav" :class="{ show }">
  <!-- 与 LayoutHeader 相同的结构 -->
</div>
```

```css
.fixed-nav {
  position: sticky;
  top: -80px;
  transition: top .3s;
  z-index: 100;
}
.fixed-nav.show { top: 0; }
```

这里用的是 `sticky` 加 `top` 过渡，而不是 `position: fixed` 加显示/隐藏。两种做法都能用，区别在于：`fixed` 会把元素从文档流里摘出来，需要额外占位，否则下面的内容会跳一下；`sticky` 保留占位，过渡更平滑。小兔鲜选后者。

三个实现细节。第一，阈值 78 是头部高度，写死容易出问题，最好用 `ref` 量出来或者用一个常量统一定义。第二，滚动事件必须做节流，`useScroll` 内部已经处理，但如果你手写 `addEventListener('scroll')`，记得加 `requestAnimationFrame` 或节流，否则每秒触发几十次重排。第三，`position: fixed` 在某些祖先元素有 `transform` 或 `filter` 时会失效或错位——如果哪天你把 Layout 包进一个带动画的容器，吸顶突然不工作了，先查这个。

为什么吸顶头要复制一份结构，而不是复用 `LayoutHeader`？因为两者样式不同：吸顶头更矮、只保留一级导航与搜索。复用一个组件再传 `mode` 属性也能做，但会让组件内部充满条件判断，两处改动互相影响。复制一份结构、抽取公共数据源，是这种场景更省心的选择。

```js
// composables/useSticky.js —— 抽出来两边共用
export function useSticky(threshold = 78) {
  const { y } = useScroll(window)
  return { show: computed(() => y.value > threshold) }
}
```

## 六、Pinia 优化重复请求：把请求搬到 store

导航要分类数据、分类页的侧栏要、面包屑要。如果三处都写 `onMounted(() => getCategoryAPI())`，用户从首页点到分类页，同一个接口被调两次，每次都要等几百毫秒。

优化分两步。第一步把请求搬进 `category` store，组件只负责调用。

```js
// stores/category.js
export const useCategoryStore = defineStore('category', () => {
  const list = ref([])
  const loading = ref(false)

  async function fetchList() {
    if (list.value.length) return list.value
    if (loading.value) return list.value
    loading.value = true
    try {
      const res = await getCategoryAPI()
      list.value = res.result
      return list.value
    } finally {
      loading.value = false
    }
  }
  return { list, loading, fetchList }
})
```

第二步解决「三个页面同时挂载」的并发问题。首页与 Layout 几乎同时渲染，两者都调 `fetchList`，第一个请求还没回来时 `list` 还是空的，第二个调用会再发一次。代码里的 `if (loading.value) return` 只能拦住第二次，拦不住「同时」——因为在 `await` 之前两个调用都可能已经通过检查。

正确的做法是缓存「正在进行的请求」，而不是只缓存结果。

```js
let pending = null
async function fetchList() {
  if (list.value.length) return list.value
  if (!pending) {
    pending = getCategoryAPI()
      .then((res) => { list.value = res.result; return list.value })
      .finally(() => { pending = null })
  }
  return pending
}
```

这样并发的三次调用共享同一个 Promise，实际只发一次请求。这个模式叫「请求去重」，在真实项目里比「结果缓存」更常用，因为用户经常在数据还没回来时就连点两下。

需要提醒的是：这个 store 的实例作用域是整个应用。如果某个页面需要「强制刷新分类」，就不能靠再调 `fetchList`，而要提供单独的 `refresh()` 清空 `list` 再取。把「读」和「刷新」分开，能避免「想刷新却命中缓存」的困惑。

## 七、行为拆分：把滚动逻辑抽成 composable

第 2 章说过 store 负责「怎么取数据」，页面负责「什么时候要」。这里多一类：**跨组件复用的行为**，它既不该进 store（没有全局状态），也不该复制到每个组件（会漂移）。这类东西放 `src/composables/`。

`useSticky` 是典型例子。它内部持有 `y` 的监听、提供 `show` 的计算结果、在组件卸载时自动清理（`useScroll` 内部用 `effectScope` 处理）。任何组件想要「滚动超过阈值就显示」，一行 `const { show } = useSticky(78)` 就够。

```js
// 页面里
const { show } = useSticky(78)
```

判断一个逻辑该放哪，用三个问题：它有没有跨页面共享的状态？有就进 store。它是不是纯计算、无副作用？是就进 `utils/`。它是不是「一段有生命周期的行为」（监听、定时器、订阅）？是就进 `composables/`。

第三类最容易被写散。小兔鲜里的候选还有：`useCountDown`（支付页倒计时）、`useIntersectionObserver`（图片懒加载）、`useSku`（详情页规格选择）。它们的共同点都是「装的时候要想好什么时候拆」，如果写在组件里，一旦漏了 `onUnmounted`，就是内存泄漏。

```js
// composables/useCountDown.js
export function useCountDown(seconds = 60) {
  const left = ref(seconds)
  let timer = null
  function start() {
    stop()
    timer = setInterval(() => {
      left.value--
      if (left.value <= 0) stop()
    }, 1000)
  }
  function stop() {
    if (timer) { clearInterval(timer); timer = null }
  }
  onUnmounted(stop)
  return { left, start, stop }
}
```

注意 `onUnmounted(stop)` 写在 composable 内部。它能在组件卸载时生效，是因为 `useCountDown` 是在 `setup` 同步执行期间被调用的，此时 `onUnmounted` 能拿到当前组件实例。如果把它放在 `setTimeout` 里延迟调用，就绑不上，定时器永远不会被清理。

## 八、工程实践与常见坑

**第一，忘了 `flex: 1`。** 现象是内容少的页面底部浮在屏幕中间，下方大片留白。原因是内容区没有撑满。处理方式：`.main { flex: 1 }` 配合容器 `min-height: 100vh`。

**第二，`<i>` 图标是斜的。** 现象是字体图标整体倾斜。原因是 `<i>` 默认 `font-style: italic`。处理方式：给图标类加 `font-style: normal`。

**第三，吸顶不生效。** 三种可能：祖先元素有 `transform`/`filter`/`perspective` 导致 `fixed` 错位；滚动容器不是 window 而是某个 div；`z-index` 被其他层盖住。排查顺序按这个来，能省很多时间。

**第四，导航高亮同时亮两个。** 现象是访问 `/category/101` 时「分类」和「分类/101」都高亮。原因是 `router-link-active` 是包含匹配。处理方式：用 `exact-active-class` 或自定义 `:class`。

**第五，三个页面各请求一次分类。** 现象是首屏并发数异常、接口被调三次。根因是请求写在各自组件里，且没有处理并发去重。处理方式：搬进 store 并缓存「进行中的 Promise」。

**第六，滚动监听没清理。** 现象是切换页面后内存占用持续上涨、控制台报「在已卸载组件上调用」。原因是手写了 `addEventListener` 但没在 `onUnmounted` 里 `removeEventListener`。处理方式：优先用 `useScroll` 这类封装好的 composable，手写时一定成对出现。

## 延伸阅读

- [Vue Router 官方文档：嵌套路由](https://router.vuejs.org/zh/guide/essentials/nested-routes.html) —— Layout 加 `children` 的结构就来自这里，含命名视图的用法。
- [VueUse 文档：useScroll](https://vueuse.org/core/useScroll/) —— 吸顶与滚动相关逻辑的标准实现，注意它的清理机制。
- [Vue 官方文档：组合式函数](https://cn.vuejs.org/guide/reusability/composables.html) —— 第七节「什么该抽成 composable」的官方依据。
- [MDN：position sticky](https://developer.mozilla.org/zh-CN/docs/Web/CSS/position#sticky) —— 解释 sticky 与 fixed 的本质区别，以及为什么 `top` 必须在滚动方向上设置。
- [第 6 章 一级分类页](../一级分类页/一级分类页_專業版.md) —— 本章的 category store 在分类页第一次承担更重的职责。

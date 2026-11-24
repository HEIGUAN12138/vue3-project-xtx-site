# Home 页

> 首页是唯一一个「必须同时面对所有组件技术」的页面：并发请求、轮播、组件封装、全局指令、懒加载、列表复用。它也是最能体现工程判断的一页。

## 一、首页的信息架构：为什么是「楼层」

电商首页不是一张长图，而是一列**楼层**：轮播、一级分类、新鲜好物、人气推荐、商品列表。每个楼层独立成块，各自有标题、有数据源、有"查看更多"。这个结构不是设计偏好，而是运营需求的直接结果——运营要能增删楼层、调整顺序、按活动替换某一块。

楼层化带来一个技术决策：**一个楼层一个接口，还是一个聚合接口**。前者请求数多，但单块失败只影响那一块，可以展示"该楼层暂无内容"；后者请求数少，但一个字段出错整页空白，而且运营加一个楼层就要后端改接口。

小兔鲜选了前者。首页三个请求（banner、新鲜好物、人气推荐）互相独立，用 `Promise.all` 并发发出，首屏时间取决于最慢的那个。如果串行发，总时间就是三者相加，在弱网下差别非常明显。

```js
onMounted(async () => {
  await Promise.all([
    bannerStore.fetchBanner(),
    goodsStore.fetchNew(),
    goodsStore.fetchHot(),
  ])
})
```

注意 `Promise.all` 的语义：任何一个 reject 都会让整体 reject。如果希望「banner 挂了但商品列表照常显示」，就要用 `Promise.allSettled`，或者每个请求自己 `catch`。

```js
await Promise.allSettled([a(), b(), c()])   // 不因单个失败而中断
```

首页还有一个容易被忽略的判断：**哪些请求该在 Layout 里发**。分类数据在 Layout 发（导航需要），首页不再重复请求；banner 只在首页出现，所以留在首页。判断标准是「这个数据有几个页面要用」，不是「它出现在哪个位置」。

## 二、静态结构与分类区：先把骨架立起来

写页面的顺序建议是「先静态、再数据、后交互」。先把 HTML 结构与样式写完，用一个假数组渲染，确认布局在各种数据量下都不塌，再接真实接口。反过来先接接口，一旦样式有问题，你会分不清是数据形状不对还是布局不对。

首页的骨架大致是这样：

```html
<div class="home">
  <div class="banner"><Carousel /></div>
  <div class="category">...</div>
  <HomePanel title="新鲜好物" sub="新鲜出炉 品质靠谱">...</HomePanel>
  <HomePanel title="人气推荐" sub="人气爆款 不容错过">...</HomePanel>
  <HomeProduct title="生鲜" /> ...
</div>
```

顶部导航占位要留出来。因为 Layout 的头部是吸顶的，首页第一屏如果从 `top: 0` 开始排，会被导航盖住一截。常见做法是给内容区加 `padding-top`，或者在 Layout 里给内容区一个 `margin-top`。这个值要和头部高度一致，所以最好定义成 CSS 变量而不是散落的数字。

```css
:root { --header-height: 78px; }
.home { padding-top: var(--header-height); }
```

分类区通常是一个横向滚动的图标列表，每个图标下面是分类名。数据来自 Layout 已经取过的分类 store，这里只做渲染，不发请求。如果分类数量很多，横向滚动比换行更合适——换行会挤压下面的楼层。

```html
<div class="category-list">
  <RouterLink v-for="c in list" :key="c.id" :to="`/category/${c.id}`" class="cat-item">
    <img :src="c.picture" :alt="c.name" />
    <span>{{ c.name }}</span>
  </RouterLink>
</div>
```

```css
.category-list { display: flex; gap: 16px; overflow-x: auto; }
.cat-item { flex: none; width: 64px; text-align: center; }
```

`flex: none` 是关键：不加它，flex 子项会被压缩，图标变形。`overflow-x: auto` 让超出部分可以横向滑动，而不是把容器撑破。

## 三、Banner 轮播图：组件库组件的受控封装

小兔鲜用 Element Plus 的 `el-carousel`。用组件库的组件有两个层次：直接写标签（能跑就行），或者包一层自己的组件（可以统一行为）。Banner 值得包一层，因为它的数据形状、点击跳转、图片尺寸在三处都要一致。

```html
<el-carousel height="500px" :interval="3000" arrow="never">
  <el-carousel-item v-for="item in bannerList" :key="item.id">
    <RouterLink :to="item.target">
      <img :src="item.imgUrl" :alt="item.title" />
    </RouterLink>
  </el-carousel-item>
</el-carousel>
```

三个细节值得说。第一，`height` 必须显式给，`el-carousel` 不会根据内容自动撑高，不给就会变成一条缝。第二，图片要用 `object-fit: cover`，否则不同尺寸的运营图会被拉变形。

```css
.el-carousel__item img { width: 100%; height: 100%; object-fit: cover; display: block; }
```

`display: block` 也不能省：`img` 默认是行内元素，下方会有几像素的基线间隙，表现为图片和容器底部之间一条白边。

第三，轮播的自动播放要用 `interval` 而不是自己写定时器。自己写会踩两个坑：切换页面没清理（第 4 章的坑），以及和组件内部的定时器打架（用户手动切换后自己那个还在按原节奏跳）。

如果不想用组件库的轮播，也可以手写，但至少要处理四件事：自动播放、鼠标悬停暂停、指示点同步、无缝衔接。这四件事每一件都有边界情况，用现成的更省时间。

## 四、面板组件封装：一个组件吃掉四处重复

首页有四个楼层，结构一模一样：标题、副标题、「查看更多」链接、内容区。内容区不同：有的是四个商品卡片，有的是六个，有的带价格。这就是典型该封装的场景。

封装的关键是决定「用什么方式把不同部分传进去」。三种选择：props 传数据、插槽传结构、混合。这里用混合最合适：标题用 props，内容区用默认插槽。

```html
<!-- HomePanel.vue -->
<template>
  <div class="panel">
    <div class="panel-head">
      <h3>{{ title }}</h3>
      <span class="sub">{{ sub }}</span>
      <RouterLink v-if="more" :to="more" class="more">查看全部 →</RouterLink>
    </div>
    <div class="panel-body"><slot /></div>
  </div>
</template>

<script setup>
defineProps({
  title: String,
  sub: String,
  more: String,
})
</script>
```

```html
<!-- 使用 -->
<HomePanel title="新鲜好物" sub="新鲜出炉 品质靠谱" more="/category/101">
  <ul class="goods-list">
    <li v-for="g in newList" :key="g.id"><GoodsItem :item="g" /></li>
  </ul>
</HomePanel>
```

为什么不用 `props` 传一个数组然后组件内部渲染？因为四个楼层的内容结构不同：有的显示价格、有的只显示名字、有的带标签。用插槽把「怎么渲染」交给调用方，组件只管「框子」，职责最清晰。

判断该不该封装的三个信号：同样的结构出现三次以上；改动一次要改多处；新来的人会问「这两个为什么不一样」。三个都满足才封装——只出现两次的，先复制，等到第三次再抽，避免过早抽象。

## 五、新鲜好物与人气推荐：同一套取数模式

这两个楼层的实现几乎相同：封装接口 → 组件里调用 → 拿到数组 → 循环渲染。差别只在接口路径和卡片显示字段。

```js
// apis/home.js
export const getNewGoodsAPI = () => request.get('/home/new', { params: { limit: 4 } })
export const getHotGoodsAPI = () => request.get('/home/hot', { params: { limit: 4 } })
```

```js
// 组件里
const newList = ref([])
onMounted(async () => {
  const res = await getNewGoodsAPI()
  newList.value = res.result
})
```

这段代码看起来平淡，但它体现了「三个层次各管一件事」：`apis` 管地址与参数，组件管「什么时候要」，模板管「长什么样」。只要三层不混，后面加一个楼层就是复制二十行。

要提醒的是 `limit` 参数的归属。如果写在前端，运营想改显示数量就得发版；如果后端按楼层配置返回，前端只负责渲染。小兔鲜写在 `apis` 层，属于折中——改数量要发版，但至少不在模板里散落数字。

如果楼层多了，会出现「每个楼层都写一遍 `ref` + `onMounted` + `catch`」的重复。这时可以抽一个 `useFloor` 组合式函数，但要注意别把「接口不同」也抽进去，否则参数会膨胀成配置文件。

```js
function useFloor(fetcher) {
  const list = ref([])
  const loading = ref(false)
  onMounted(async () => {
    loading.value = true
    try { list.value = (await fetcher()).result } finally { loading.value = false }
  })
  return { list, loading }
}
```

## 六、图片懒加载指令：从「全部加载」到「进入视口才加载」

首页图片多，如果全部在首屏加载，会抢占带宽导致轮播图（用户第一眼要看的）反而是最后才出来的。懒加载的目标很明确：**只加载用户当前能看到的图**。

实现方式有两种：监听 `scroll` 自己算位置，或者用 `IntersectionObserver`。后者是浏览器原生 API，性能更好且不用手写节流，现代浏览器全面支持。

先写一个独立的指令逻辑：

```js
// directives/lazy.js
import { useIntersectionObserver } from '@vueuse/core'

export const lazy = {
  mounted(el, binding) {
    const { stop } = useIntersectionObserver(el, ([entry]) => {
      if (!entry.isIntersecting) return
      el.src = binding.value
      stop()                                  // 加载一次就够了
    })
    el._stopLazy = stop                        // 卸载时兜底清理
  },
  unmounted(el) {
    if (el._stopLazy) el._stopLazy()
  },
}
```

关键点有三个。第一，`stop()` 必须在首次进入视口后调用，否则滚动过程中会反复触发赋值。第二，要在 `unmounted` 里兜底清理，防止图片一直没进视口就离开页面时留下监听。第三，`binding.value` 是图片地址，用 `v-lazy="imgUrl"` 传入。

注册成全局指令：

```js
// main.js
app.directive('lazy', lazy)
```

```html
<img v-lazy="item.picture" alt="" />
```

为什么用指令而不是组件？因为要懒加载的是原生 `<img>`，包一层组件会破坏原有的样式与布局（组件需要额外一层包裹，`img` 的 `width: 100%` 参照就变了）。指令能直接操作元素属性，侵入最小。

一个常见误区：给 `<img>` 写 `src` 占位图再替换。占位图本身也要加载，反而多一次请求。更好的做法是先不设 `src`，或者用一张极小的 base64 占位。

## 七、GoodsItem 组件封装：卡片的三处复用

商品卡片在首页、分类页、搜索结果三处出现，样式一致。它的字段：图片、标题、价格、销量/标签。

```html
<template>
  <RouterLink :to="`/detail/${item.id}`" class="goods-item">
    <img v-lazy="item.picture" :alt="item.name" />
    <p class="name">{{ item.name }}</p>
    <p class="price">¥{{ item.price }}</p>
  </RouterLink>
</template>

<script setup>
defineProps({ item: { type: Object, required: true } })
</script>
```

注意价格显示。`item.price` 可能是 `199` 也可能是 `199.00`，直接渲染会出现「¥199」和「¥199.00」不统一。应该在组件内做一次格式化，而不是指望后端。

```js
const priceText = computed(() => Number(props.item.price).toFixed(2))
```

组件放在 `src/components/` 下，因为三个页面都要用。这和 Layout 内部组件的判断标准正好相反：Layout 的子组件只有 Layout 用，跟随页面走；GoodsItem 跨页面复用，进全局。

一个设计问题：卡片要不要内置「加入购物车」按钮？如果内置，首页和分类页都会出现，但用户可能在首页误点；如果不内置，三个页面各写一遍。小兔鲜的选择是不内置，只在详情页与购物车出现加购入口——这是把「什么时候允许加购」当成产品决策，而不是组件能力。

## 八、工程实践与常见坑

**第一，首屏被导航盖住。** 现象是轮播图顶部被吸顶导航遮住一截。原因是内容从 `top: 0` 开始排。处理方式：内容区加 `padding-top: var(--header-height)`，变量与头部高度同源。

**第二，轮播图显示成一条缝。** `el-carousel` 不给 `height` 就不会撑开。处理方式：显式设置高度，并给内部 `img` 加 `object-fit: cover` 与 `display: block`。

**第三，图片下方一条白边。** `img` 是行内元素，有基线间隙。处理方式：`display: block`。

**第四，懒加载图片反复赋值。** 现象是滚动时控制台刷日志、图片闪烁。原因是没在首次进入视口后 `stop()`。

**第五，`Promise.all` 导致整页空白。** 一个楼层接口挂了，`Promise.all` 直接 reject，三个楼层都没渲染。处理方式：`Promise.allSettled`，或每个请求单独 `catch` 做局部降级。

**第六，楼层组件里写死 `limit`。** 现象是运营要改显示数量就得发版。处理方式：参数放 `apis` 层或由后端配置，组件只渲染。

## 延伸阅读

- [Element Plus 文档：Carousel 轮播图](https://element-plus.org/zh-CN/component/carousel.html) —— 高度、间隔、指示器的完整参数，第三节的对照版。
- [MDN：Intersection Observer API](https://developer.mozilla.org/zh-CN/docs/Web/API/Intersection_Observer_API) —— 懒加载的底层原理，理解它才能判断什么时候该 `stop()`。
- [VueUse 文档：useIntersectionObserver](https://vueuse.org/core/useIntersectionObserver/) —— 本章指令内部用的就是它，注意它返回的 `stop` 与自动清理机制。
- [Vue 官方文档：自定义指令](https://cn.vuejs.org/guide/reusability/custom-directives.html) —— 指令的生命周期钩子，第六节的直接依据。
- [第 7 章 二级分类页](../二级分类页/二级分类页_專業版.md) —— 本章的懒加载与列表封装，在二级分类页会进化成「无限滚动」。

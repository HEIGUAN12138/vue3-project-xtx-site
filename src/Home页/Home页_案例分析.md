# Home 页 · 案例分析

> 四个真实背景：懒加载 API 的由来、首页楼层化对性能指标的影响、指令式封装的选择，以及过度抽象的代价。

## Case 1：IntersectionObserver 解决了什么

### 背景

在 `IntersectionObserver` 出现之前，判断「元素是否进入视口」只能靠监听 `scroll` 事件并调用 `getBoundingClientRect()`。问题有两个：一是 `scroll` 每秒触发几十次，每次调用 `getBoundingClientRect()` 都会触发布局计算（强制同步布局），页面很快就卡；二是滚动的容器可能不是 `window`，嵌套滚动、iframe、`overflow: auto` 的 div 都要单独处理。

### 事件经过

2016 年前后，W3C 与浏览器厂商开始推进 `IntersectionObserver` API，Chrome 51（2016 年 5 月）率先实现，Safari 与 Firefox 在 2017 到 2019 年陆续跟进。它的思路是把「元素与视口的交叉状态」交给浏览器内部的合成线程计算，只在交叉状态发生变化时回调，主线程不需要参与逐帧计算。

这直接改变了懒加载与无限滚动的实现方式。有了它之后，「图片懒加载」从「必须做节流 + 必须自己算位置」变成了「注册一个观察器 + 在回调里判断 `isIntersecting`」。VueUse 的 `useIntersectionObserver` 就是对它的封装，额外处理了 SSR、旧浏览器降级与自动清理。

### 与本章的关系

本章第六节的懒加载指令直接建立在 `IntersectionObserver` 之上，`stop()` 对应的是观察到的 `unobserve`。理解「回调只在状态变化时触发」这一点，才能明白为什么「首次进入后必须 stop」——不 stop 的话，每次交叉状态变化（包括滚出去又滚回来）都会重新赋值。

### 参考资料

- [MDN：Intersection Observer API](https://developer.mozilla.org/zh-CN/docs/Web/API/Intersection_Observer_API)
- [VueUse：useIntersectionObserver](https://vueuse.org/core/useIntersectionObserver/)
- [web.dev：使用 Intersection Observer 实现懒加载](https://web.dev/articles/lazy-loading-observers)

## Case 2：首页图片直接影响 LCP

### 背景

2020 年，Google 把 Core Web Vitals 作为搜索排名信号之一，其中 LCP（Largest Contentful Paint，最大内容绘制）衡量「主内容什么时候可见」。电商首页的 LCP 元素通常是轮播图的第一张或首屏最大的商品图，也就是说：**首屏大图的加载时间几乎决定了这个指标**。

### 事件经过

在此之前，性能优化的重点常在「JS 体积」与「首屏渲染时间」。Core Web Vitals 把关注点拉到了「用户看到大图的时间」，于是懒加载从「可选优化」变成「必须做」：如果首屏之外的图不做懒加载，它们会和首屏大图抢带宽，在慢速网络下把 LCP 拖后两到三秒。

同时出现的还有两个配套做法：给首屏大图加 `fetchpriority="high"` 提示浏览器优先下载；给首屏之外的图加 `loading="lazy"`（原生懒加载，Chrome 76 起支持）。原生属性并不能完全替代 JS 方案——它无法控制「加载后的后续行为」，也无法在加载完成后做动画，所以在需要精细控制时仍然要写指令。

### 与本章的关系

本章第六节说「全部加载会把轮播图挤到最后一个」，依据就是 LCP 的机制：带宽是共享的，先发的请求不一定先完成，而首屏之外的图片会消耗本该留给首屏大图的带宽。这也是为什么懒加载不是「优化项」而是「首屏正确性的一部分」。

### 参考资料

- [web.dev：Core Web Vitals](https://web.dev/articles/vitals)
- [MDN：img 的 loading 属性](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/img#attr-loading)
- [web.dev：优化 LCP](https://web.dev/articles/optimize-lcp)

## Case 3：为什么用指令而不是组件做懒加载

### 背景

Vue 的自定义指令在 Vue 2 时代主要用于「操作 DOM 的底层行为」，比如聚焦、拖拽、点击外部关闭。Vue 3 之后指令的写法调整为 `mounted`/`updated`/`unmounted`，与组合式 API 的生命周期命名对齐。

### 事件经过

做懒加载有两条路。第一条是包一个 `<LazyImage>` 组件，内部渲染 `<img>` 并处理观察逻辑。第二条是写一个 `v-lazy` 指令，直接改元素属性。

组件方案的问题在于多了一层 DOM。原本 `<img class="goods-img">` 上的 `width: 100%`、`object-fit: cover`、`aspect-ratio` 全部失效，因为选择器命中的父元素而不是 `img` 本身。要么把样式透传下去（`v-bind="$attrs"`），要么让调用方改成给内部图片写样式——两种都会让使用者困惑。

指令方案没有这个问题：元素还是那个元素，只是多了一个属性绑定。代价是指令无法返回模板结构，也无法接收插槽，但对「只是换个 src」这个需求完全够用。

社区里的实践也印证了这一点：VueUse 官方推荐用 `v-lazy` 这类指令做图片懒加载，而 `<LazyImage>` 更多出现在需要「占位图 + 淡入动画 + 错误兜底」的完整图片组件里。

### 与本章的关系

本章第六节选指令，判断依据是「我们只需要换 src」。如果需求升级为「加载失败显示默认图 + 骨架占位 + 淡入」，那就应该改回组件，因为那时要控制的是「渲染结构」而不只是「一个属性」。技术选型跟着需求变，而不是跟着流行变。

### 参考资料

- [Vue 官方文档：自定义指令](https://cn.vuejs.org/guide/reusability/custom-directives.html)
- [Vue 官方文档：透传 Attributes](https://cn.vuejs.org/guide/components/attrs.html)
- [MDN：object-fit](https://developer.mozilla.org/zh-CN/docs/Web/CSS/object-fit)

## Case 4：过度抽象让首页组件变得难以修改

### 背景

2002 年，Joel Spolsky 写过一篇《抽象泄漏定律》：所有非平凡的抽象在某种程度上都是有漏洞的。这个判断在组件封装上同样成立——抽象能减少重复，但会把「差异」藏起来，等到差异变多时，抽象的代价会反超收益。

### 事件经过

一个常见的演化路径是这样：首页四个楼层结构相似，于是有人抽了一个 `HomePanel`，用 `props` 传 `type` 区分四种布局。第一周很顺，代码少了两百行。第二个月，运营要求「新鲜好物」显示倒计时，「人气推荐」显示标签，商品列表要横向滚动。于是 `HomePanel` 里出现了 `v-if="type === 'new'"`、`v-if="type === 'hot'"`，`props` 从 3 个涨到 9 个。

到第三个月，改「人气推荐」的样式导致「商品列表」也跟着变形，因为两者共用了一段条件分支。最终团队把它拆回四个独立组件，公共部分只保留标题栏。

这个路径的可复用结论是：**用插槽传结构、用 props 传数据，比用 props 传「类型」更抗变化**。因为插槽把「结构差异」留在调用方，组件本身不需要知道有几种形态。

### 与本章的关系

本章第四节选择「标题用 props、内容用插槽」，正是为了避免这个演化陷阱。判断标准也写在了那一节：如果两个组件的差异小于 20%，合并；差异会持续增长，就分开。

### 参考资料

- [Joel on Software：The Law of Leaky Abstractions](https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/)
- [Vue 官方文档：插槽](https://cn.vuejs.org/guide/components/slots.html)
- [Vue 官方文档：组件设计原则](https://cn.vuejs.org/guide/components/props.html)

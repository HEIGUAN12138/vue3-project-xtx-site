# Home 页 · 进一步学习

> 这一章的关键词是「并发取数」「封装边界」与「视觉细节」，清单围绕这三件事组织。

## 官方文档

- [Element Plus 文档：Carousel 轮播图](https://element-plus.org/zh-CN/component/carousel.html) —— `height`、`interval`、`arrow`、`indicator-position` 的完整参数，第三节的对照版。难度：入门
- [Element Plus 文档：Layout 布局](https://element-plus.org/zh-CN/component/layout.html) —— 楼层内部的栅格与间距，比手写 flex 更省事。难度：入门
- [MDN：Intersection Observer API](https://developer.mozilla.org/zh-CN/docs/Web/API/Intersection_Observer_API) —— 懒加载的底层能力，读完才能解释「为什么回调只在状态变化时触发」。难度：进阶
- [VueUse 文档：useIntersectionObserver](https://vueuse.org/core/useIntersectionObserver/) —— 本章指令内部用的就是它，注意 `stop` 与 `isSupported` 的处理。难度：进阶
- [Vue 官方文档：自定义指令](https://cn.vuejs.org/guide/reusability/custom-directives.html) —— 指令的生命周期钩子与参数形状，第六节的直接依据。难度：入门
- [Vue 官方文档：插槽](https://cn.vuejs.org/guide/components/slots.html) —— 默认插槽、具名插槽、作用域插槽的区别，第四节的判断基础。难度：入门
- [MDN：object-fit](https://developer.mozilla.org/zh-CN/docs/Web/CSS/object-fit) —— `cover` 与 `contain` 的实际效果差异，配 `object-position` 一起看。难度：入门
- [MDN：Promise.allSettled](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise/allSettled) —— 与 `Promise.all` 的行为对照表，第一节的依据。难度：入门

## 书籍

- 《CSS 世界》（张鑫旭） —— 行内元素的基线、`vertical-align`、替换元素的高度计算，解释了「图片下方那条白边」从哪来。难度：进阶
- 《Vue.js 设计与实现》（霍春阳） —— 组件渲染与插槽编译那一部分，能说明「为什么插槽比 props 更灵活」。难度：高级
- 《高性能网站建设指南》（Steve Souders） —— 虽然年代久，但「减少请求、控制优先级」的判断至今有效，是理解懒加载动机的经典读物。难度：进阶

## 课程

- [web.dev：Learn Performance](https://web.dev/learn/performance) —— 免费课程，LCP、CLS 的定义与优化手段，解释「为什么首屏大图决定指标」。难度：进阶
- [Vue 官方互动教程](https://cn.vuejs.org/tutorial/) —— 组件、插槽、自定义指令三节，是本章第二、四、六节的基础。难度：入门
- [小兔鲜电商项目实战（本笔记的原稿来源）](../Vue3入门/Vue3入门_專業版.md) —— 首页的完整搭建过程在原始课件里有逐步说明，包括静态页与接口对接顺序。难度：进阶

## 视频与播客

- [Vue.js 官方 YouTube 频道](https://www.youtube.com/@vuejs) —— 有关于组合式函数与组件设计的分享。难度：进阶
- [Chrome for Developers：Lazy loading 讲解](https://www.youtube.com/@ChromeDevs) —— 讲原生 `loading="lazy"` 与 IntersectionObserver 的取舍，配 Case 2 一起看。难度：进阶
- [Syntax.fm](https://syntax.fm/) —— 关于「什么时候该抽组件」的讨论，对第四节的三信号判断有帮助。难度：入门

## 怎么用这份清单

如果只是想把首页做出来，看 Element Plus 的 Carousel 与 Layout 两页就够，配合本章第二节的骨架代码能直接跑。如果首页加载慢、图片一条条冒出来，去读 `IntersectionObserver` 与 web.dev 的 LCP 部分，那是唯一能把「为什么先加载的反而后显示」讲清楚的地方。如果想彻底搞懂「什么该封成组件、什么该写指令」，把 Vue 的插槽与自定义指令两章连着读一遍，再回头看第四节与第六节的取舍，判断就会稳。

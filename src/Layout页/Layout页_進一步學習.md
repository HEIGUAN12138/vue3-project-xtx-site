# Layout 页 · 进一步学习

> 这一章的关键词是「复用」与「清理」，清单围绕这两件事组织。

## 官方文档

- [Vue Router 官方文档：嵌套路由](https://router.vuejs.org/zh/guide/essentials/nested-routes.html) —— Layout 加 `children` 的官方写法，含默认子路由。难度：入门
- [Vue Router 官方文档：链接激活类](https://router.vuejs.org/zh/api/interfaces/RouterLinkProps.html) —— `active-class` 与 `exact-active-class` 的区别，第四节高亮问题的依据。难度：入门
- [Vue 官方文档：组合式函数](https://cn.vuejs.org/guide/reusability/composables.html) —— 什么时候该抽函数、约定怎么命名，第七节直接依据。难度：进阶
- [Vue 官方文档：effectScope](https://cn.vuejs.org/api/reactivity-advanced.html#effectscope) —— 解释「为什么组合式函数能自动清理」，是理解副作用生命周期的关键。难度：高级
- [VueUse 文档：useScroll](https://vueuse.org/core/useScroll/) —— 吸顶、滚动进度的标准实现，注意它的参数与清理方式。难度：入门
- [MDN：position](https://developer.mozilla.org/zh-CN/docs/Web/CSS/position) —— `sticky` 与 `fixed` 的差异，以及包含块的概念。难度：入门

## 书籍

- 《CSS 世界》（张鑫旭） —— 讲清了 `position`、包含块、层叠上下文这些 Layout 页最容易出问题的部分。难度：进阶
- 《Vue.js 设计与实现》（霍春阳） —— 副作用与作用域那一章能解释 `effectScope` 的设计动机。难度：高级

## 课程

- [Vue 官方互动教程](https://cn.vuejs.org/tutorial/) —— 组件与插槽部分，是本章第二节的基础。难度：入门
- [Vue 官方文档：组件基础](https://cn.vuejs.org/guide/essentials/component-basics.html) —— 插槽、props、事件三节读一遍，再回来看 Layout 的结构会更清楚。难度：入门
- [小兔鲜电商项目实战（本笔记的原稿来源）](../Vue3入门/Vue3入门_專業版.md) —— Layout 页的完整搭建过程在原始课件里有逐步说明。难度：进阶

## 视频与播客

- [Vue.js 官方 YouTube 频道](https://www.youtube.com/@vuejs) —— 有组件设计与复用的分享。难度：进阶
- [CSS 布局讲解（B 站）](https://search.bilibili.com/all?keyword=CSS%20sticky%20%E5%B8%83%E5%B1%80) —— 中文讲解，专门讲 sticky 的失效场景，比文档更直观。难度：入门
- [Syntax.fm](https://syntax.fm/) —— 关于「抽象与复用的边界」的讨论，对判断「该不该合并组件」有帮助。难度：进阶

## 怎么用这份清单

如果只是想做出吸顶效果，看 VueUse 的 `useScroll` 文档就够。如果想彻底搞懂为什么吸顶会突然失效，去读 MDN 的 `position` 与「包含块」条目，那是唯一能把「代码没改却坏了」解释清楚的地方。如果想把「哪些逻辑该抽成 composable」变成直觉，读 Vue 官方的组合式函数章节并对照本章第七节的三问法，练三五次就稳了。

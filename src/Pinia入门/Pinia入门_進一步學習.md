# Pinia 入门 · 进一步学习

> 按「先官方文档、再书、再课程、最后视频」的顺序用，每一项都标明难度。

## 官方文档

- [Pinia 官方文档（中文）](https://pinia.vuejs.org/zh/) —— 全站最权威的参考，核心概念章节与本章一一对应。难度：入门
- [Pinia 官方文档：Cookbook](https://pinia.vuejs.org/zh/cookbook/) —— 收录了跨 store 调用、组合式函数复用、测试隔离等实战写法，是查漏补缺的第一站。难度：进阶
- [Pinia 官方文档：迁移指南](https://pinia.vuejs.org/zh/cookbook/migration-vuex.html) —— 如果手里有 Vuex 项目要迁，按这篇逐项对照改，不要凭记忆。难度：进阶
- [Vue 官方文档：状态管理](https://cn.vuejs.org/guide/scaling-up/state-management.html) —— 讲清了「什么时候不需要 Pinia」，是克制使用全局状态的理论依据。难度：入门
- [Vue 官方文档：组合式 API 常见问答](https://cn.vuejs.org/guide/extras/composition-api-faq.html) —— `ref` 与 `reactive` 的取舍、解构丢响应性的原理，在读完本章第七节后看效果最好。难度：进阶

## 书籍

- 《Vue.js 设计与实现》（霍春阳） —— 第 4 章讲响应式系统，从 `Proxy` 到依赖收集的实现细节，能解释清「为什么解构会丢响应性」这类看起来玄学的问题。难度：高级
- 《Vue.js 3 企业级应用开发实战》 —— 有完整的后台管理项目，Pinia 部分覆盖了权限 store 与持久化，适合想看「团队里怎么组织 store」的读者。难度：进阶

## 课程

- [Vue 官方互动教程](https://cn.vuejs.org/tutorial/) —— 在浏览器里直接写代码，不需要本地环境，适合快速把组合式 API 的手感找回来。难度：入门
- [Vue Mastery: Pinia 课程](https://www.vuemastery.com/courses/) —— 有专门讲状态管理的系列，讲解节奏偏慢，适合边看边敲。难度：入门
- [小兔鲜电商项目实战（本笔记的原稿来源）](../Vue3入门/Vue3入门_專業版.md) —— 从项目起步一路做到支付页，Pinia 在 Layout、购物车、登录三处反复出现。难度：进阶

## 视频与播客

- [Vue.js 官方 YouTube 频道](https://www.youtube.com/@vuejs) —— Vue Conf 的分享录像，能看到核心团队对状态管理的设计讨论。难度：进阶
- [Vue 3 组合式 API 系列讲解（B 站）](https://search.bilibili.com/all?keyword=Vue3%20%E7%BB%84%E5%90%88%E5%BC%8F%20API) —— 中文讲解，适合在通勤时听一遍建立印象，具体写法仍以官方文档为准。难度：入门
- [Syntax.fm：前端状态管理讨论](https://syntax.fm/) —— 不只讲 Vue，能从 React 的 Redux、Zustand 对比出「状态管理库到底该负责什么」。难度：进阶

## 怎么用这份清单

如果只想把项目跑起来，看「官方文档」前两条即可。如果准备在团队里推广 Pinia，补上 Cookbook 和迁移指南，重点是「什么时候不该用」。如果想把「解构丢响应性」这类问题彻底弄清，去读《Vue.js 设计与实现》的响应式章节——那是唯一能让你从「记住结论」变成「推出结论」的路径。

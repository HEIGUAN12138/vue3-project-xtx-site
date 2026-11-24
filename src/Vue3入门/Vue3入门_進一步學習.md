# Vue3入门：进一步学习资源

> 这一章学完，多数人的短板不是「不知道 API」，而是「不知道为什么要这样设计」。下面的资源按「补设计动机 → 补语言基础 → 补工程实践」的顺序排列。

---

## 官方文档

**[Vue 3 官方文档（中文）](https://cn.vuejs.org/guide/)**
- 全章内容的权威来源，尤其是「深入响应式系统」一节，能把 `ref`/`reactive` 的边界讲透。
- 难度：入门到进阶

**[Vue 3 API 参考](https://cn.vuejs.org/api/)**
- 查具体 API 的签名与边界条件，比教程更准确；写代码时应该常备。
- 难度：入门

**[Vue 官方互动教程](https://cn.vuejs.org/tutorial/)**
- 浏览器里直接改代码看效果，适合把本章第八节的六个坑亲手复现一遍。
- 难度：入门

**[MDN：Proxy](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)**
- 响应式的底层机制就是它，看懂 `get`/`set` 陷阱，就明白为什么解构会丢响应性。
- 难度：进阶

---

## 书籍

**[《Vue.js 设计与实现》— 霍春阳](https://www.ituring.com.cn/book/2953)**
- 讲的是「框架怎么被造出来」，包括响应式系统的依赖收集与调度器。读完再回看本章第三节，会有完全不同的理解。
- 难度：高级

**[《JavaScript 高级程序设计（第 4 版）》— Matt Frisbie](https://www.ituring.com.cn/book/2472)**
- 补语言地基：`Proxy`、`this`、闭包、事件循环。本章的很多坑本质是 JS 坑，不是 Vue 坑。
- 难度：进阶

---

## 课程

**[Vue Mastery](https://www.vuemastery.com/)**
- 官方合作的英文课程站，Composition API 系列讲得系统，配有可运行示例。
- 难度：入门到进阶

**[Vue School](https://vueschool.io/)**
- 偏工程实践，含 Pinia、路由、测试等后续章节会用到的内容。
- 难度：入门到进阶

---

## 视频与播客

**[Vue.js Nation（会议录像）](https://vuejsnation.com/)**
- 每年一届的 Vue 社区大会，能看到真实团队怎么组织大型项目。
- 难度：进阶

**[Anthony Fu 的 YouTube 频道](https://www.youtube.com/@antfu)**
- Vue 与 Vite 核心团队成员的频道，常讲工具链与工程化细节。
- 难度：进阶

---

## 学习建议

**第一步，补设计动机。** 本章的 API 都能查文档，难的是「为什么是这套 API」。花一个下午读《Vue.js 设计与实现》的前四章，理解「依赖收集 → 触发更新」这条主线，再回头看 `ref` 与 `reactive` 的差别，就不再需要背。

**第二步，补语言基础。** 第八节列的六个坑里，至少四个本质是 JavaScript 问题：解构、引用、闭包、事件循环。如果 `Proxy` 和 `Object.defineProperty` 的区别说不清，建议先补 MDN 的 Proxy 与 Reflect，再回来看第三节。

**第三步，用输出验证。** 把本章的六个坑各写一个最小复现（每个不超过 30 行），放在一个页面里逐个触发。能稳定复现、能解释原因、能给出改法，才算真的学会。之后进入下一章 Pinia 时，你会不断回到这一章的状态设计上。

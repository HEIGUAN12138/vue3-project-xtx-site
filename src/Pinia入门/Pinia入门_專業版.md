# Pinia 入门

> 本章专为「小兔鲜」这类多页电商前台而写：当 token、购物车、分类数据需要跨页面存活时，组件本地的 `ref` 就不够用了。

## 一、为什么是 Pinia：Vuex 之后的官方选择

在组合式 API 出现之前，Vue 官方推荐的状态管理方案是 Vuex。Vuex 的核心设计建立在选项式 API 之上：一个 store 由 `state`、`mutations`、`actions`、`getters` 四块拼成，其中 `mutations` 是唯一允许同步修改状态的地方，异步逻辑必须放进 `actions` 再由它提交 `mutations`。这套约束在 2016 年是合理的，它让「谁改了状态」有唯一的入口，方便调试与时间旅行。但它也带来了明显的样板代码：想改一个数字，要写一个 mutation、写一个 action、在组件里 dispatch，三层跳转才落一次赋值。

Pinia（Pinia，Vue 官方状态管理库）在 2019 年底由 Eduardo San Martin Morote 发起，2022 年初成为 Vue 官方推荐的默认方案，Vue 官方文档也从那时起把 Vuex 标注为维护模式。它取消了 `mutations`，一个 store 就是一个组合式函数：`state` 是 `ref`，`getters` 是 `computed`，`actions` 是普通函数。你在组件里怎么写逻辑，在 store 里就怎么写，没有第二套心智模型。

这个变化对电商项目特别重要。以购物车为例，本地购物车的加购逻辑包含「同一 SKU 累加、否则新增」「同时更新勾选状态」「金额汇总」三件事，在 Vuex 里必须拆成 mutation 加 action，在 Pinia 里就是一个 `addCart` 函数，写完就能读。代码量大约减少三成，更重要的是减少了一半的「这段逻辑该放哪」的犹豫。

```js
// Vuex 的写法：改一个数字要三处配合
mutations: { SET_COUNT(state, n) { state.count = n } },
actions: { setCount({ commit }, n) { commit('SET_COUNT', n) } }
// 组件里
this.$store.dispatch('setCount', 10)
```

```js
// Pinia 的写法：一个函数搞定
actions: { setCount(n) { this.count = n } }
// 组件里
store.setCount(10)
```

需要说清的一点是：Pinia 不是 Vuex 的升级版，而是一次重新设计。Vuex 里 `state` 是响应式对象、`mutations` 是同步入口，这两条在 Pinia 里被合并为「直接改 state」。代价是失去了 mutation 提供的那条「所有修改都从这里过」的审计链，所以 Pinia 用了另一个机制补上：DevTools 里每一次 state 变化都会被记录，并标注是哪个 action 引起的。审计能力没有丢，只是从代码约束变成了工具约束。

## 二、安装与注册：app.use(pinia) 之后发生了什么

安装只有一条命令，`npm i pinia`。注册则要求在创建应用之后、挂载之前调用 `createPinia()` 并传给 `app.use()`。这个顺序不能反：如果先 `app.mount('#app')` 再装 Pinia，组件里第一次调用 `useUserStore()` 时会拿不到 pinia 实例，报「getActivePinia was called with no active Pinia」。

```js
// main.js
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

const app = createApp(App)
app.use(createPinia())   // 必须在 mount 之前
app.mount('#app')
```

`createPinia()` 做了一件容易被忽略的事：它创建了一个独立的响应式容器，并把 `effectScope` 挂在这个容器上。所有 store 的 `computed`、`watch` 都注册在这个 scope 里。这意味着当你调用 `app.unmount()` 时，所有 store 的副作用会一起被停掉，不会留下泄漏的侦听器。这也是为什么在测试里可以「每次用一个新的 pinia 实例」，测试之间互不污染。

在 SSR（Server-Side Rendering）场景下，这个设计更关键：每个请求要有一个独立的 pinia 实例，否则不同用户的购物车会串在一起。Nuxt 3 里 `usePinia()` 帮你处理了这件事，手写 SSR 时要自己维护。

```js
// 服务端每个请求一份实例，避免用户数据串号
export default async function (ctx) {
  const pinia = createPinia()
  ctx.app.use(pinia)
}
```

第三个容易踩的点是 store 的定义位置。`defineStore` 可以在任意模块里调用，但它只是「登记」一个 store 工厂，真正创建实例发生在组件里第一次 `useXxxStore()` 时。所以 store 文件被 `import` 并不会立刻产生状态，这让循环依赖的风险比 Vuex 低很多。

## 三、定义 store：defineStore 的两种写法

`defineStore` 接收两个参数：一个是 store 的唯一 id，一个是配置对象或组合式函数。id 必须全局唯一，DevTools 里靠它区分不同的 store，如果两个 store 用了同一个 id，后者会直接复用前者的状态，表现为「改了 A 结果 B 也变了」。

选项式写法与 Vuex 的 `state` 很像，适合从旧项目迁过来的团队。

```js
// stores/user.js
import { defineStore } from 'pinia'

export const useUserStore = defineStore('user', {
  state: () => ({
    token: '',
    userInfo: null,
  }),
  getters: {
    isLogin: (state) => !!state.token,
  },
  actions: {
    setToken(t) { this.token = t },
    clear() { this.token = ''; this.userInfo = null },
  },
})
```

注意 `state` 必须是一个函数，不能是对象字面量。原因是 store 可能被多次创建（比如 SSR 或测试），如果写成共享对象，两次创建就会指向同一份数据，这是从 Vuex 迁移时最常见的错误。

组合式写法是 Pinia 更有优势的地方：它就是一个 `setup` 函数，`ref` 当 state，`computed` 当 getters，普通函数当 actions。

```js
// stores/category.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { getCategoryAPI } from '@/apis/category'

export const useCategoryStore = defineStore('category', () => {
  const list = ref([])
  const loading = ref(false)
  const topLevel = computed(() => list.value.filter(c => c.id % 2 === 0))

  async function fetchList() {
    if (loading.value) return
    loading.value = true
    try {
      const res = await getCategoryAPI()
      list.value = res.result
    } finally {
      loading.value = false
    }
  }

  return { list, loading, topLevel, fetchList }
})
```

两种写法可以混用，跨 store 调用时也一样。判断标准是：如果这个 store 需要复用已有的组合式函数（比如 `useIntersectionObserver`、`useCountDown`），用组合式；如果团队习惯 Vuex 的骨架，用选项式。同一个项目里两种都有完全正常，小兔鲜的 `user` 用选项式、`category` 用组合式，验证了这一点。

## 四、state 的读写：直接改、$patch 与 $reset

选项式 store 里读 state 有两种路径：在 store 内部用 `this.xxx`，在组件里用 `store.xxx`。两者都是响应式的，因为 Pinia 把整个 store 包在 `reactive` 里。要注意的是不能解构，原因在第七节展开。

修改 state 有三种方式，选哪种取决于改动的规模。

第一种是直接赋值，适合单字段改动。

```js
const userStore = useUserStore()
userStore.token = res.token          // 组件里直接改
```

第二种是 `$patch`，适合一次改多个字段。传对象时是浅合并，传函数时可以写更复杂的逻辑。

```js
// 对象形式：浅合并
userStore.$patch({ token: res.token, userInfo: res.userInfo })
// 函数形式：可以对数组做 push
userStore.$patch((state) => {
  state.userInfo = res.userInfo
  state.history.push(Date.now())
})
```

第三种是 `$reset()`，只在选项式 store 里有效。它把 state 恢复成 `state()` 函数返回的初始值。这里有个陷阱：组合式 store 没有 `$reset`，调用会报错。组合式的重置要自己写一个函数，把每个 `ref` 重新赋值。

```js
// 组合式 store 自己实现重置
function reset() {
  list.value = []
  loading.value = false
}
```

小兔鲜里 `$reset` 的典型用武之地是退出登录：`userStore.$reset()` 一次清掉 token 与用户信息，比手写两行赋值更不容易漏字段。但要注意 `$reset` 不会清理你手动加在 store 外部的状态，也不会动 pinia 插件的持久化存储，如果接了 `pinia-plugin-persistedstate`，重置后本地存储里还是旧值，需要额外调用一次清理。

## 五、getters：派生状态与缓存

getters 的本质是 `computed`，因此有两个特性：一是结果被缓存，依赖不变时重复读取不会重复计算；二是不允许有副作用，不能在里面发请求或改 state。把这两条记住，getters 的边界就清晰了。

```js
getters: {
  isLogin: (state) => !!state.token,
  // 需要用到其他 getter 时，第二个参数传 this
  loginText() { return this.isLogin ? '已登录' : '未登录' },
  // 需要传参时，返回一个函数（注意：此时缓存失效）
  findById: (state) => (id) => state.list.find(i => i.id === id),
}
```

第三行的写法值得单独说。getter 返回函数后，每次调用都是一个新函数体，`computed` 的缓存就退化了，实际上是「用 getter 的语法写普通函数」。在购物车里按 id 查条目是可以接受的，因为查找本身是 O(n) 的轻量操作；但如果这个函数里还有聚合计算，就应该改成 action 或在外层算好。

购物车的金额汇总是 getters 最经典的用法。它要同时满足三个条件：只统计勾选项、单价乘数量、加上运费。

```js
getters: {
  checkedItems: (state) => state.list.filter(i => i.selected),
  totalCount: (state) => state.list.reduce((s, i) => s + (i.selected ? i.count : 0), 0),
  totalPrice() {
    const goods = this.checkedItems.reduce((s, i) => s + i.price * i.count, 0)
    return goods + (goods > 0 ? this.freight : 0)
  },
}
```

这段代码的价值不在省了几行，而在于「金额只有一个真源」。购物车页、订单页、支付页读同一个 `totalPrice`，就不会出现详情页显示 199、支付页显示 189 这种对不上的情况。凡是把金额计算复制到多个组件里的项目，最后都要花时间排查这类差价。

## 六、action：同步、异步与职责边界

action 在 Pinia 里就是普通函数，可以是同步的也可以是异步的，没有 `dispatch` 这层包装。组件里 `store.fetchList()` 直接调用，和调一个普通方法完全一样。

```js
actions: {
  async fetchProfile() {
    const res = await getProfileAPI()
    this.userInfo = res.result
    return res.result
  },
}
```

这样写法带来两个便利。第一，异步 action 的返回值可以直接用，组件里 `const info = await store.fetchProfile()` 就能拿到数据，不用再去 state 里读一遍。第二，多个 store 可以互相调用，Pinia 会在 `this` 上注入所有已创建的 store，不需要 import。

```js
// cart store 里调用 user store，不需要 import
import { useUserStore } from './user'

actions: {
  async checkout() {
    const userStore = useUserStore()
    if (!userStore.isLogin) {
      return { needLogin: true }
    }
    // ... 结算逻辑
  },
}
```

这里要注意调用时机：`useUserStore()` 必须写在 action 内部，不能写在模块顶层。写在顶层会在模块加载时就执行，此时 pinia 还没注册，会直接抛错。这条规则在四个 store 互相引用的项目里尤其重要，写错了报错信息很难定位。

关于职责边界，一个实用的判断是：**action 负责「让状态变得可以从服务端拿到」，组件负责「什么时候需要它」**。例如 `categoryStore.fetchList()` 只关心怎么取数据、取完之后存哪；至于「页面进入时要取」，写在组件的 `onMounted` 或路由守卫里。这样同一个 action 可以被首页、分类页、详情页复用，而不用在每个页面重复写请求逻辑。小兔鲜的 Layout 页之所以能用 Pinia 优化重复请求，正是因为这个划分：`fetchList` 里加了「已有数据就直接返回」的判断，三个页面调用同一个 action，实际只发了一次请求。

```js
async function fetchList() {
  if (list.value.length) return list.value   // 命中缓存，直接返回
  const res = await getCategoryAPI()
  list.value = res.result
  return list.value
}
```

## 七、storeToRefs：为什么不能直接解构

把 store 解构到组件里是最容易出错的地方。`const { token } = userStore` 拿到的只是一个字符串快照，之后 store 里的 token 变了，组件里的 `token` 不会变，模板也不会重新渲染。原因是 Pinia 的 store 是一个 `reactive` 对象，而解构会读取属性值，读取动作发生在 setup 阶段，之后就断开了。

正确的做法有两条。读取 state 与 getters 时用 `storeToRefs`，它会把每个属性包成 `ref` 并保持与 store 的连接；读取 actions 时直接解构，因为 action 是函数，不需要响应式。

```js
import { storeToRefs } from 'pinia'

const userStore = useUserStore()
const { token, userInfo, isLogin } = storeToRefs(userStore)  // state + getters
const { setToken, clear } = userStore                        // actions
```

```html
<template>
  <div v-if="isLogin">{{ userInfo.nickname }}</div>
</template>
```

为什么 `storeToRefs` 能保持连接？它对每个属性调用 `toRef(store, key)`，得到的 ref 拥有指向源对象的 getter/setter，所以写 `token.value = 'x'` 会真的改到 store。相比之下 `toRefs` 会连 action 一起转成 ref，而 action 被包成 ref 没有意义，还会在解构后调用时报错，这就是官方单独提供 `storeToRefs` 的原因。

还有一个更隐蔽的坑：如果只用 store 里的一两个字段，也可以不解构，直接在模板里写 `userStore.token`，这样天然是响应式的。但要注意模板里的 `userStore` 必须是响应式可追踪的引用，也就是不要在 setup 里先把它转成普通对象。

小兔鲜的购物车列表会用到这个特性：组件里解构 `list` 与 `totalPrice`，模板里直接循环；而 `addCart`、`removeCart` 直接解构。这是全项目最标准的写法，建议照着抄。

## 八、工程实践与常见坑

**第一，state 写成对象字面量而不是函数。** 现象是两个组件看到同一份数据、互不影响的需求失效。原因是共享引用。处理方式：选项式 store 的 `state` 一律写 `state: () => ({...})`。

**第二，在模块顶层调用 `useXxxStore()`。** 现象是刷新页面时报 "getActivePinia was called with no active Pinia"。原因是模块加载早于 `app.use(pinia)`。处理方式：store 之间的调用一律写在 action 或 setup 内部。

**第三，解构 store 后丢失响应性。** 现象是改数据但视图不更新。原因是 `const { x } = store` 取了快照。处理方式：state 与 getters 用 `storeToRefs`，actions 直接解构。

**第四，组合式 store 调用 `$reset()`。** 现象是控制台报 `$reset is not a function`。原因是 `$reset` 只对选项式 store 生效。处理方式：组合式自己写 `reset()`，逐个把 `ref` 赋回初值。

**第五，异步 action 没有并发保护。** 现象是快速切换分类标签时，后发的请求先返回，列表显示错乱。原因是多个请求共享同一个 `list` 变量，谁最后返回谁覆盖。处理方式：加 `loading` 标记提前返回，或用请求序号校验「返回的是不是我这次发的」。

```js
let seq = 0
async function fetchByCategory(id) {
  const cur = ++seq
  const res = await getGoodsAPI(id)
  if (cur !== seq) return          // 已有更新的请求，丢弃本次结果
  list.value = res.result
}
```

**第六，把接口错误处理留给组件。** 现象是每个页面都要写一遍 `try/catch` 加弹窗。原因是 action 只负责发请求不负责兜底。处理方式：在 axios 响应拦截器里统一处理业务错误码，action 只处理「成功之后怎么存」，真正的异常再往外抛给组件做局部降级。

## 延伸阅读

- [Pinia 官方文档：开始使用](https://pinia.vuejs.org/zh/getting-started.html) —— 安装、注册、第一个 store 的完整流程，先跑通官方示例再看本章会省很多时间。
- [Pinia 官方文档：组合式写法](https://pinia.vuejs.org/zh/cookbook/composing-stores.html) —— 讲清了跨 store 调用与循环依赖的处理，是选项式迁移到组合式的必读章节。
- [Vue 官方文档：状态管理](https://cn.vuejs.org/guide/scaling-up/state-management.html) —— 先讲「什么时候不需要 Pinia」，能避免把组件本地状态无脑搬进 store。
- [Pinia 官方文档：测试](https://pinia.vuejs.org/zh/cookbook/testing.html) —— 演示如何用 `createTestingPinia` 隔离 store，写单元测试时直接抄。
- [第 10 章 购物车](../购物车/购物车_專業版.md) —— 本章的 getters 与 action 在真实业务里怎么落地，全选、汇总、本地与接口购物车的切换都在那一章展开。

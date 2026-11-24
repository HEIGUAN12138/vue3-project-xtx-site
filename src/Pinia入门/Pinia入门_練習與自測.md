# Pinia 入门 · 练习与自测

> 题量：是非 10 / 选择 10 / 配对 8 / 情境 3 / 代码阅读 3 / 延伸思考 2。答案紧跟各区块。

## 一、是非题（10 题）

1. Pinia 里仍然需要写 `mutations` 才能修改 state。
2. `createPinia()` 必须在 `app.mount()` 之前调用。
3. `defineStore` 的第一个参数是 store 的唯一 id，重复时后者会复用前者的状态。
4. 选项式 store 的 `state` 可以直接写成对象字面量，不必是函数。
5. 组合式 store 自带 `$reset()` 方法。
6. `getters` 的结果会被缓存，依赖不变时不重复计算。
7. `const { token } = userStore` 之后，store 里的 token 变化会自动同步到组件。
8. `storeToRefs` 会把 actions 也转成 ref。
9. 在 store 的 action 里可以调用另一个 store，`useXxxStore()` 应写在 action 内部。
10. `$reset()` 会一并清理 `pinia-plugin-persistedstate` 写入的本地存储。

### 答案解析

1. **错**。Pinia 取消了 mutations，直接改 state 即可，DevTools 负责记录。
2. **对**。否则首个 `useXxxStore()` 会报 `getActivePinia was called with no active Pinia`。
3. **对**。id 是 store 的身份标识，重复会导致状态串用。
4. **错**。必须写成 `state: () => ({...})`，否则多次创建会共享同一份数据。
5. **错**。`$reset` 只对选项式 store 有效，组合式需手写 reset 函数。
6. **对**。getters 本质是 computed；但返回函数的 getter 会失去缓存。
7. **错**。解构取到的是值快照，失去与 store 的连接。
8. **错**。`storeToRefs` 只处理 state 与 getters，actions 直接解构即可。
9. **对**。写在模块顶层会在 pinia 注册前执行，必然报错。
10. **错**。`$reset` 只清内存，持久化插件的数据需要单独清理。

## 二、选择题（10 题）

1. Pinia 取消 mutations 的主要原因是？
   A. 为了减少代码体积　B. 组合式 API 让「同步修改入口」不再必要　C. Vuex 的 mutations 有 bug　D. 便于支持 TypeScript 的装饰器

2. 下面哪个写法能保证响应性？
   A. `const { list } = cartStore`　B. `const { list } = storeToRefs(cartStore)`　C. `const list = cartStore.list`　D. `const list = toRefs(cartStore.list)`

3. 组合式 store 里想要「重置」，正确做法是？
   A. 调 `$reset()`　B. 重新 `createPinia()`　C. 自己写函数把每个 ref 赋回初值　D. 调 `$dispose()`

4. `$patch` 传入对象时，执行的是？
   A. 深合并　B. 浅合并　C. 完全替换 state　D. 逐字段调用 action

5. 下面关于 getters 的说法，错误的是？
   A. 依赖变化时重新计算　B. 可以在里面发请求　C. 可以用 `this` 访问其他 getter　D. 返回函数时缓存失效

6. 商品详情页加购、导航角标、购物车列表同时更新，根本原因是？
   A. 用了事件总线　B. 三处读同一份 store 响应式数据　C. 用了 `watch` 手动同步　D. 组件通过 props 层层传递

7. `fetchList()` 里加 `if (list.value.length) return` 的作用是？
   A. 防止并发请求覆盖　B. 命中缓存，避免重复请求　C. 做错误重试　D. 触发 getters 重算

8. 下面哪个场景**不适合**放进 Pinia？
   A. 登录 token　B. 购物车条目　C. 某个弹窗是否打开的临时状态　D. 全局分类树

9. 异步 action 中判断「本次结果是否已过期」，常用手段是？
   A. 用 `setTimeout` 延迟赋值　B. 用递增序号比对　C. 用 `nextTick`　D. 用 `$patch`

10. 关于 `pinia` 的 `effectScope`，正确的说法是？
    A. 用于限制 store 数量　B. 让 store 的副作用在应用卸载时统一停止　C. 用于持久化　D. 用于 SSR 时的序列化

### 答案解析

1. **B**。mutations 的约束是为「选项式 API 下的可追踪性」服务的，组合式 API 提供了别的追踪方式。A 不是设计动机，C 不成立，D 与装饰器无关。
2. **B**。`storeToRefs` 返回与 store 相连的 ref；A、C 是值快照，D 写法错误（`toRefs` 接收响应式对象而非单值）。
3. **C**。组合式 store 无 `$reset`；`$dispose` 是销毁 store 而非重置。
4. **B**。对象形式是浅合并，嵌套对象需整体替换或用函数形式。
5. **B**。getters 不能有副作用，禁止发请求。
6. **B**。同一个响应式数据被多处读取，不是通知机制。
7. **B**。这是最朴素的缓存；并发覆盖要靠序号校验（见第 9 题）。
8. **C**。只在单个组件内、生命周期很短的 UI 状态应留在组件里，放进 store 反而增加清理负担。
9. **B**。记录本次请求序号，返回时比对，不一致就丢弃。
10. **B**。`createPinia()` 创建的 scope 统一管理 store 内的 computed 与 watch 副作用。

## 三、名词配对（8 组）

把左侧概念与右侧描述配对。

| 左 | 右 |
|---|---|
| 1. `defineStore` | A. 让解构后的 state 保持响应式 |
| 2. `storeToRefs` | B. 一次修改多个 state 字段 |
| 3. `$patch` | C. 登记一个 store 工厂 |
| 4. `$reset` | D. 派生状态，带缓存 |
| 5. `getters` | E. 恢复选项式 store 的初始 state |
| 6. `actions` | F. 唯一标识一个 store |
| 7. store id | G. 同步或异步的业务函数 |
| 8. `createPinia()` | H. 创建并安装响应式容器 |

### 答案

1-C　2-A　3-B　4-E　5-D　6-G　7-F　8-H

## 四、情境分析（3 题）

### 情境 1：改了数据但页面不刷新

某同学在组件里写了：

```js
const cartStore = useCartStore()
const { list, totalPrice } = cartStore
```

然后在另一个组件里调 `cartStore.addCart(item)`，发现列表和总价都不更新，但 DevTools 里 state 确实变了。

**问**：根因是什么？怎么改？

**答**：`const { list, totalPrice } = cartStore` 取到的是值快照，脱离了响应式追踪，所以 DevTools 里 state 变了但组件拿的是旧值。改成 `const { list, totalPrice } = storeToRefs(cartStore)`，或者模板里直接写 `cartStore.list`。

### 情境 2：进入页面就报错

某同学把跨 store 调用写在了文件顶层：

```js
import { useUserStore } from './user'
const userStore = useUserStore()      // 顶层

export const useCartStore = defineStore('cart', () => {
  // ...
})
```

刷新页面时报 `getActivePinia was called with no active Pinia`。

**问**：为什么？如何改？

**答**：模块顶层代码在 `app.use(createPinia())` 执行之前就被加载了，此时还没有激活的 pinia 实例。把 `useUserStore()` 挪进 action 或 setup 函数内部，让它延迟到真正调用时执行。

### 情境 3：分类切换后列表错乱

快速点「手机 → 电脑 → 家电」，最后停在家电，但列表显示的是手机的商品。

**问**：这是什么问题？给出两种解法。

**答**：请求竞态。手机的请求最慢、最后返回，覆盖了家电的结果。解法一：用递增序号校验，返回时序号不等于最新序号就丢弃。解法二：用 `AbortController` 取消上一个请求。前者不依赖网络层，后者更彻底但需要接口封装支持。

## 五、代码阅读（3 题）

### 题 1

```js
export const useUserStore = defineStore('user', {
  state: () => ({ token: '', userInfo: null }),
  getters: { isLogin: (state) => !!state.token },
})
```

**问**：这里 `state` 为什么必须是函数？如果把箭头函数写成普通函数 `state() { return {...} }` 行不行？

**答**：必须是函数，是为了每次创建 store 时拿到一份新对象，避免多个实例共享同一份数据。写成 `state() { return {...} }` 也可以，它同样是函数；但不能写成 `state: { token: '' }`。

### 题 2

```js
export const useCartStore = defineStore('cart', () => {
  const list = ref([])
  const total = computed(() => list.value.reduce((s, i) => s + i.price * i.count, 0))
  return { list, total }
})
```

**问**：在组件里 `const { list, total } = useCartStore()` 会有什么问题？`useCartStore().$reset()` 呢？

**答**：第一个问题同情境 1，失去响应性，需要 `storeToRefs`。第二个会报 `$reset is not a function`，因为这是组合式 store，需要自己写 reset 函数把 `list.value` 置空。

### 题 3

```js
async function fetchList() {
  if (list.value.length) return list.value
  loading.value = true
  try {
    const res = await getCategoryAPI()
    list.value = res.result
    return list.value
  } finally {
    loading.value = false
  }
}
```

**问**：这段代码解决了什么问题？`finally` 里放 `loading.value = false` 相比写在 `try` 末尾好在哪？

**答**：`if (list.value.length) return` 让多个页面共享一次请求，命中缓存后不再发请求。`finally` 保证无论请求成功、失败还是抛异常，`loading` 都会被复位，避免异常时按钮永久转圈。

## 六、延伸思考（2 题）

1. 有人说「有了 Pinia 就不需要 props 了」。你同意吗？请从「数据由谁拥有」的角度论述，并指出一个用了 Pinia 反而更糟的场景。

   **思路**：props 表达的是「父组件拥有的数据交给子组件用」，方向明确、边界清晰；Pinia 表达的是「全局共享」。一个只在父组件和它三个子组件之间流转的数据，用 Pinia 会让「谁负责初始化、谁负责清理」变得模糊。典型反例：一个只在一个页面上出现的筛选面板状态，放进 store 后，离开页面还得手动清空，否则下次进来是上次的残留。

2. 购物车的 `totalPrice` 用 `getters` 算，而不是在 `addCart` 里同步维护一个 `totalPrice` 变量。请说明为什么后者在长期维护中一定会出问题，并举出一个「看起来只是少算一块钱」的实际后果。

   **思路**：同步维护意味着任何能改变「勾选状态、数量、单价、优惠」的入口都必须记得更新它。入口越多，漏一处就越容易。实际后果：购物车显示 199，提交订单时服务端按真实金额算 198，用户看到「金额变化」而怀疑平台，或者反过来——前端显示 189 实际扣 199，直接变成客诉与退款。

## 自测标准

- 10 题是非全对，且能说出每个「错」的题错在哪：合格
- 情境分析三题能独立说清根因，不只是背结论：良好
- 能不看资料写出一个组合式 store，包含 state、getter、异步 action、reset：优秀

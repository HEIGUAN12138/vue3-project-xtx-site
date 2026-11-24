# Layout 页 · 练习与自测

> 题量：是非 10 / 选择 10 / 配对 8 / 情境 3 / 代码阅读 3 / 延伸思考 2。答案紧跟各区块。

## 一、是非题（10 题）

1. Layout 应该是顶层路由，所有页面都是它的 `children`。
2. 只被一个页面使用的组件，应放在 `src/components/`。
3. `<i class="iconfont">` 不需要处理字体会显示正常。
4. 字体图标可以通过 CSS 的 `color` 改变颜色。
5. `router-link-active` 是精确匹配，不会误伤父路径。
6. `position: sticky` 会保留元素在文档流中的占位。
7. 祖先元素带 `transform` 时，`position: fixed` 可能失效或错位。
8. store 里只缓存请求结果，就能避免「同时挂载」造成的重复请求。
9. composable 内部的 `onUnmounted` 只有在 setup 同步执行期间调用才生效。
10. 吸顶头应该复用 `LayoutHeader` 组件并传 `mode` 属性，避免重复代码。

### 答案解析

1. **错**。登录页、支付页、会员中心通常不套 Layout，它们是独立顶层路由。
2. **错**。只被一个页面用的组件应放在该页面的 `components/` 下。
3. **错**。`<i>` 默认 `font-style: italic`，图标会倾斜，需加 `font-style: normal`。
4. **对**。字体图标本质是文字，颜色由 `color` 控制。
5. **错**。`router-link-active` 是包含匹配，父路径也会被激活。
6. **对**。这是 sticky 与 fixed 的核心区别。
7. **对**。`transform`/`filter`/`perspective` 会创建新的包含块，导致 fixed 相对它定位。
8. **错**。并发时两个调用都在结果返回前通过检查，仍会发两次请求，需要缓存「进行中的 Promise」。
9. **对**。`onUnmounted` 依赖当前组件实例或 effectScope。
10. **错**。两者结构差异大，硬复用一个组件会塞满条件判断；复制结构、共用数据更清晰。

## 二、选择题（10 题）

1. 判断某组件该放全局还是跟随页面，依据是？
   A. 组件大小　B. 被几个页面使用　C. 是否有 props　D. 是否需要样式

2. 内容少的页面底部浮在中间，最可能缺的是？
   A. `min-height: 100vh` 或内容区 `flex: 1`　B. `position: relative`　C. `overflow: hidden`　D. `z-index`

3. `<i>` 图标显示为斜体，应加？
   A. `text-decoration: none`　B. `font-style: normal`　C. `transform: none`　D. `font-weight: 400`

4. `router-link-active` 误伤父路径时，应改用？
   A. `active-class`　B. `exact-active-class`　C. `replace`　D. `v-if`

5. 吸顶实现选 `sticky` 而非 `fixed`，主要收益是？
   A. 性能更好　B. 保留文档流占位，不跳动　C. 兼容性更好　D. 支持动画

6. 首屏 Layout 与 Home 同时请求分类，避免重复请求的关键是？
   A. 加 `loading` 标记　B. 缓存「进行中的 Promise」　C. 用 `watch` 监听　D. 用 `debounce`

7. 下面哪类逻辑最该放 `composables/`？
   A. 金额格式化　B. 滚动监听与清理　C. 全局 token　D. 路由表

8. 滚动监听建议用 VueUse 的 `useScroll`，主要因为它？
   A. 支持移动端　B. 内部处理了节流与清理　C. 体积小　D. 支持 SSR

9. 导航条只显示前 9 个分类，这个决定属于？
   A. 纯样式问题　B. 产品/信息架构决策　C. 性能优化　D. 兼容性处理

10. 需要强制刷新分类数据时，正确的做法是？
    A. 再调一次 `fetchList`　B. 提供独立的 `refresh()` 清空缓存　C. 刷新页面　D. 用 `$reset`

### 答案解析

1. **B**。复用范围决定归属。
2. **A**。容器撑满视口、内容区 `flex: 1` 才能把底部推到底。
3. **B**。斜体来自 `font-style`。
4. **B**。`exact-active-class` 只在精确匹配时激活。
5. **B**。sticky 不脱离文档流。
6. **B**。并发时 `loading` 检查会同时通过，必须共享同一个 Promise。
7. **B**。它是有生命周期、需要清理的行为。
8. **B**。`useScroll` 内部基于 effectScope 自动清理并处理了性能问题。
9. **B**。放几个属于信息架构，不该由 CSS 溢出隐藏决定。
10. **B**。`fetchList` 会被「已有数据」挡住，必须走独立刷新入口。

## 三、名词配对（8 组）

| 左 | 右 |
|---|---|
| 1. `children` | A. 嵌套路由的子页面声明 |
| 2. `RouterView` | B. 子页面的渲染出口 |
| 3. `iconfont` | C. 把图标做成字形的方案 |
| 4. `useScroll` | D. 返回滚动位置的组合式函数 |
| 5. `position: sticky` | E. 到边界即贴住，保留占位 |
| 6. `exact-active-class` | F. 精确匹配时激活的类名 |
| 7. 进行中的 Promise | G. 并发请求去重的载体 |
| 8. `onUnmounted` | H. 组件卸载时执行清理 |

### 答案

1-A　2-B　3-C　4-D　5-E　6-F　7-G　8-H

## 四、情境分析（3 题）

### 情境 1：导航项高亮错了

访问 `/category/101` 时，「全部分类」（指向 `/category`）和「手机」（指向 `/category/101`）两个导航项同时高亮。

**问**：原因与修法。

**答**：`router-link-active` 是包含匹配，`/category/101` 以 `/category` 开头，所以父链接也被激活。修法：给链接加 `exact-active-class` 并用它写样式，或者用 `:class` 手动判断 `route.path === to`。

### 情境 2：吸顶条突然不工作

某天把 Layout 外面包了一层带动画的容器（`transform: translateZ(0)` 用于开启 GPU 加速），吸顶条再也不出现了，代码一个字没改。

**问**：为什么？

**答**：`transform` 会为后代元素创建新的包含块，`position: fixed` 会相对这个祖先定位而不是视口，导致「固定」失效。`position: sticky` 受同样影响。修法是把这个容器移到吸顶元素之外，或改用其他方式做性能优化。

### 情境 3：首屏分类接口被调三次

在 `LayoutNav`、`LayoutHeader`、`Home` 三处各写了 `onMounted(() => getCategoryAPI())`，Network 面板显示同一接口三个请求并发发出。

**问**：如何改造成一次？

**答**：三步。统一搬到 `category` store 的 `fetchList`；在 store 里缓存「进行中的 Promise」（模块级变量 `pending`），并发的调用共用它；三处组件改为只调 `store.fetchList()`。另外提供独立的 `refresh()` 供强制刷新，避免被缓存挡住。

## 五、代码阅读（3 题）

### 题 1

```js
const { y } = useScroll(window)
const show = computed(() => y.value > 78)
```

**问**：`78` 这个数字代表什么？直接写死有什么风险？

**答**：代表头部高度，滚动超过它就让吸顶条出现。风险是头部高度调整（改字号、加活动条、响应式）时这里不同步，表现为吸顶条出现得太早或太晚。应抽成常量或通过测量得到。

### 题 2

```js
async function fetchList() {
  if (list.value.length) return list.value
  if (loading.value) return list.value
  loading.value = true
  const res = await getCategoryAPI()
  list.value = res.result
  loading.value = false
}
```

**问**：这段代码能防止并发重复请求吗？为什么？

**答**：不能完全防止。两个调用如果在第一个 `await` 之前先后执行，第一个已经把 `loading` 置为 `true` 吗？能——但问题在于第二个调用返回的是 `list.value`（此时为空数组），拿不到数据，而不是共享请求结果。真正的去重要缓存 Promise 本身，让后来者 `await` 同一个请求。

### 题 3

```js
export function useCountDown(seconds) {
  const left = ref(seconds)
  let timer = null
  function start() {
    timer = setInterval(() => { left.value-- }, 1000)
  }
  onUnmounted(() => clearInterval(timer))
  return { left, start }
}
```

**问**：这段代码有两个问题，分别是什么？

**答**：一是重复 `start` 会创建多个定时器而旧的没清，计数会加速；二是 `timer` 为 `null` 时 `clearInterval(null)` 虽然无害，但组件卸载后若仍在 `start` 中创建定时器就清不到了。修法：`start` 前先 `stop()`，并把 `stop` 也返回给调用方。

## 六、延伸思考（2 题）

1. 「吸顶头复制了一份 LayoutHeader 的结构」看起来违反 DRY。请从「耦合 vs 重复」的角度说明这个选择的合理性，并指出什么情况下应该反过来合并。

   **思路**：两份结构共享数据源（store）、样式与结构差异大（高度、可见元素），合并会引入大量条件分支，改一处影响两处，属于「错误的抽象」。当两份差异小于 20%（比如只差一个 padding）时，才应该合并成一个组件加 `variant` 属性。

2. 请求去重缓存的是 Promise，如果那个 Promise 最终 reject 了，会发生什么？应该如何处理才不让「一次失败永久失败」？

   **思路**：必须在 `finally` 里清空 `pending`，否则后续调用会拿到已经 reject 的同一个 Promise，表现为「接口第一次超时，之后永远失败直到刷新页面」。同时要注意 unhandled rejection——如果缓存的 Promise 没有消费者，会触发全局警告，可在创建时挂一个空 `catch` 兜底。

## 自测标准

- 能说清 Layout 该包含什么、哪些页面不该套 Layout：合格
- 能独立实现吸顶效果并解释 sticky 与 fixed 的差别：良好
- 能写出带并发去重的 store 请求，并正确处理失败与强制刷新：优秀

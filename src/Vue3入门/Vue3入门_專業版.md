# Vue3入门：从选项式到组合式的思维切换

## 一、从选项式到组合式：为什么要换写法

Vue 2 的组件用「选项」组织代码：数据放 `data`，方法放 `methods`，派生值放 `computed`，监听放 `watch`。这种组织方式在组件只有两三个功能时很清晰，但一个真实的商品详情组件通常同时管理「轮播图状态、SKU 选择、库存校验、加购请求、埋点上报」。选项式写法会把这些功能的代码切碎，分别塞进五个不同的选项里——改一个功能要在文件里上下跳五次，这被称为「逻辑关注点分散」。

Vue 3 的组合式 API（Composition API）把组织维度从「代码类型」换成「功能」。同一个功能的响应式数据、计算、监听、生命周期钩子写在连续的几行里，需要复用时整段搬走即可。这不是风格偏好，而是复用成本的问题：选项式时代复用逻辑必须依赖 mixin 或作用域插槽，前者来源不明、后者嵌套地狱；组合式 API 里复用只是一个普通函数。

代价也要说清楚：组合式 API 把「什么时候执行」暴露给了开发者。选项式里 `data` 一定在 `created` 之前，而 `setup` 里的代码顺序就是执行顺序，写错了不会有人帮你兜底。理解执行时机，是这一章后面所有内容的前提。

```js
// 选项式：一个「购物车数量」功能被拆到三处
export default {
  data() { return { count: 0 } },
  computed: { double() { return this.count * 2 } },
  methods: { add() { this.count++ } }
}
```

```js
// 组合式：同一个功能写在一起，且可以直接抽走
import { ref, computed } from 'vue'
const count = ref(0)
const double = computed(() => count.value * 2)
const add = () => count.value++
```

## 二、setup 与执行时机：组件第一次运行的地方

`setup` 是组合式 API 的唯一入口。它在组件实例创建之前执行，此时 `this` 是 `undefined`，所以不能访问 `this.$refs`、`this.$router`，也不能在里面写 `await` 阻塞渲染（除非配合 `<Suspense>`）。它接收两个参数：`props`（响应式且只读）和 `context`（包含 `attrs`、`slots`、`emit`）。返回值是一个对象，其中的绑定会被暴露给模板。

实际项目里几乎不用手写 `setup()` 函数，而是用 `<script setup>` 语法糖。编译器会把顶层声明的变量、函数、import 的组件自动暴露给模板，省掉 return。这不是运行时特性，而是编译期改写：`<script setup>` 的内容会被内联进组件的 `setup()`，因此它必须是同步的顶层代码。

一个容易被忽略的细节：`setup` 在 `beforeCreate` 之前执行，此时 DOM 还不存在。任何需要读取元素尺寸、初始化第三方图表、绑定 window 事件的代码，都必须放进 `onMounted`。在 `setup` 顶层直接写 `document.querySelector('.banner')` 会稳定拿到 `null`。

```vue
<script setup>
import { ref, onMounted } from 'vue'
const el = ref(null)
const width = ref(0)

onMounted(() => {
  // 此时 DOM 已挂载，el.value 才有值
  width.value = el.value.getBoundingClientRect().width
})
</script>

<template>
  <div ref="el">宽度：{{ width }}px</div>
</template>
```

## 三、ref 与 reactive：两种响应式容器的分工

Vue 3 用 `Proxy` 拦截对象的读写：读的时候记录「谁用了这个属性」（依赖收集），写的时候通知这些依赖重新执行（触发更新）。这是响应式的全部机制，`ref` 和 `reactive` 只是两层不同的包装。

`reactive` 直接返回对象的代理，适合管理一组相关字段，例如 `reactive({ list: [], loading: false, page: 1 })`。它的限制很硬：只能代理对象，不能代理数字和字符串；解构会丢失响应性，因为解构拿到的是当时的值而不是代理；重新赋值整个对象也会断开代理。

`ref` 解决的就是这些限制。它把值塞进一个带 `get value` / `set value` 的对象里，任何类型都能包装，通过 `.value` 读写。在模板中 Vue 会自动解包顶层 ref，所以模板里写 `count` 而不是 `count.value`；但在 `<script setup>` 里必须写 `.value`，这是新手最高频的报错来源之一。`ref` 传入对象时，内部会用 `reactive` 处理，所以 `ref({a:1}).value.a` 同样是响应式的。

选择标准很简单：需要在模板里整体替换（如 `list = newList`）就用 `ref`；只是一组不会整体替换的字段就用 `reactive`。Vue 官方文档的倾向是「优先用 `ref`」，因为它的心智负担更小。

```js
import { ref, reactive } from 'vue'

const count = ref(0)
count.value++            // 正确
// count++               // 错误：改的是包装对象本身

const form = reactive({ name: '', age: 18 })
form.name = 'Tom'        // 正确，触发更新

const { name } = form    // 错误：name 是普通字符串，后续更新不会重渲染
```

## 四、模板语法与指令：从数据到 DOM 的映射

模板不是 HTML 字符串，而是被编译成渲染函数。`{{ }}` 插值会被编译成 `toDisplayString(state)`，`v-bind` 编译成属性赋值，`v-on` 编译成事件绑定。理解这一点，才能理解为什么模板里不能写 `new Date()` 这类每次渲染都会重新求值的表达式——它们会在每次更新时重新执行。

条件渲染有两个指令，语义完全不同。`v-if` 是真正的挂载与卸载：条件为假时组件被销毁，内部状态丢失，切换时有创建成本；`v-show` 只是切换 `display: none`，组件始终存在，切换成本低但首屏要渲染。判断标准是「切换频率」：频繁切换用 `v-show`，条件稳定但分支重（如权限区块）用 `v-if`。

列表渲染必须给 `key`，且 `key` 要稳定唯一。用数组索引当 `key`，在插入或删除中间项时会让 Vue 复用错节点，表现为「输入框的值跟着行跑」。`key` 的作用是告诉 diff 算法「这个节点还是原来那个」，没有它 Vue 只能按位置猜。

`v-model` 是「属性 + 事件」的语法糖：在原生元素上展开为 `value` 与 `input`，在组件上展开为 `modelValue` 与 `update:modelValue`。所以自定义组件支持 `v-model` 并不需要特殊 API，只要声明这两个东西。

```vue
<template>
  <!-- 频繁切换：用 v-show -->
  <div v-show="tab === 'desc'">商品描述</div>

  <!-- 列表：key 用业务 id，不要用 index -->
  <li v-for="item in list" :key="item.id">{{ item.name }}</li>

  <!-- 组件上的 v-model 等价于 :modelValue + @update:modelValue -->
  <MyInput v-model="keyword" />
</template>
```

## 五、computed 与 watch：派生状态和副作用的分界

`computed` 用于「由其他状态算出来的值」。它内部维护一个 dirty 标记：依赖没变时直接返回缓存，不会重复计算。这正是它和「在模板里写函数调用」的区别——后者每次渲染都会执行。`computed` 默认是只读的，需要写回时用 `get`/`set`，但写回场景很少，多数情况说明状态设计有问题。

`watch` 用于「状态变化后做别的事」，例如请求接口、写 localStorage、操作 DOM。它的默认行为是浅监听：监听对象时只关心引用是否变化，内部字段变了不会触发，需要加 `deep: true`（代价是全量遍历，大对象慎用）。`immediate: true` 让回调在建立监听时立刻执行一次，适合「初始化时也要跑一遍」的场景。

`watchEffect` 是自动收集依赖的版本：回调里用到谁就监听谁，且立即执行一次。它写起来短，但依赖关系不显式，回调一复杂就难以判断「到底谁触发了它」。项目里建议：能明确列出依赖就用 `watch`，只有依赖动态变化时才用 `watchEffect`。

三者的分工可以这样记：**算出来的用 computed，要做事的用 watch，要缓存的绝不用 watch 去手写。**

```js
import { ref, computed, watch } from 'vue'

const price = ref(100)
const count = ref(2)
const total = computed(() => price.value * count.value)   // 派生值，有缓存

watch(count, (next, prev) => {
  if (next > 99) count.value = 99        // 副作用：限购
})

watch(() => form, () => save(), { deep: true })  // 深监听，注意性能
```

## 六、组件通信：props、emit 与插槽

Vue 的数据流是单向的：父传子用 `props`，子传父用 `emit`。`props` 在子组件里是只读的，直接修改会在开发环境报警告，因为那会让「数据从哪来」变得不可追踪。需要基于 `props` 变化做处理时，用 `watch` 或 `computed`，不要赋回 `props`。

`defineProps` 和 `defineEmits` 是编译宏，不需要 import。`defineProps` 支持运行时声明（对象）和类型声明（泛型），后者能获得完整的类型提示。`withDefaults` 用于给可选属性设默认值，注意对象/数组的默认值必须用工厂函数，否则所有实例会共享同一个引用。

插槽解决的是「结构由父组件决定、位置由子组件决定」的问题。默认插槽用于内容，具名插槽用于多个区域，作用域插槽把子组件内部数据暴露给父组件的模板。电商项目里，商品卡片、弹窗、表格列几乎都用作用域插槽实现可定制。

跨层级通信（祖孙、兄弟）不要用 `provide/inject` 传可变状态，也不要用事件总线——前者不响应（除非传 ref），后者在 Vue 3 已移除。正确做法是把共享状态放进 Pinia，这也是下一章的主题。

```vue
<!-- 子组件 -->
<script setup>
const props = defineProps({ title: String, count: { type: Number, default: 0 } })
const emit = defineEmits(['update:count'])
</script>

<template>
  <button @click="emit('update:count', props.count + 1)">
    {{ title }}：{{ count }}
  </button>
</template>
```

## 七、生命周期与渲染时机：什么时候该做副作用

Vue 3 的生命周期钩子以 `onXxx` 形式导入，只能在 `setup` 同步执行阶段注册。挂在 `setTimeout` 或 `await` 之后注册会失效，因为那时组件实例的上下文已经丢失。

常用钩子的实际用途：`onMounted` 处理需要 DOM 的初始化（图表、地图、滚动容器）；`onUpdated` 在组件更新后执行，但它会在**任何**数据变化后触发，容易写成死循环，优先用 `watch` 替代；`onUnmounted` 清理定时器、事件监听、第三方实例，这是内存泄漏的主要来源；`onActivated` / `onDeactivated` 配合 `KeepAlive` 使用，缓存页面时用它刷新数据。

`nextTick` 解决的是「改完数据想立刻读 DOM」的问题。Vue 的更新是异步批处理的：同一轮事件循环里改十次数据只渲染一次，所以改完数据立刻读 DOM 拿到的还是旧值。`await nextTick()` 之后 DOM 才是最新的。典型场景是「展开列表后把滚动条定位到某一项」。

```js
import { onMounted, onUnmounted, nextTick } from 'vue'

let timer
onMounted(() => {
  timer = setInterval(poll, 3000)
})

onUnmounted(() => {
  clearInterval(timer)          // 不清理会在组件销毁后继续请求
})

async function scrollToLast() {
  list.value.push(newItem)
  await nextTick()              // 等 DOM 更新完
  container.value.scrollTop = container.value.scrollHeight
}
```

## 八、工程实践与常见坑

**现象**：页面数据变了，但视图不更新，控制台没有任何报错。
**原因**：解构了 `reactive` 对象，或把 `ref` 当普通变量赋值。解构拿到的是值快照，赋值改的是局部变量，依赖收集根本不知道你改过东西。
**处理**：保持 `.value` 写法，或改用 `toRefs` 后再解构；需要整体替换时用 `ref`。

**现象**：`v-for` 里的输入框，删除一行后下面一行的内容串到了上面。
**原因**：`key` 用了数组索引。删除中间项后索引整体前移，diff 认为「索引 1 还是索引 1」，于是复用了错误的 DOM。
**处理**：`key` 用后端返回的唯一 id。

**现象**：`v-if` 和 `v-for` 写在同一个元素上，报「属性未定义」。
**原因**：Vue 3 中 `v-if` 的优先级高于 `v-for`，条件表达式执行时循环变量还不存在。
**处理**：在外层套一个 `<template v-for>`，把 `v-if` 移到内层，或用 `computed` 先过滤列表。

**现象**：在 `setup` 里给元素绑定第三方库，报 `Cannot read properties of null`。
**原因**：`setup` 执行时 DOM 尚未挂载。
**处理**：放进 `onMounted`；如果元素在 `v-if` 分支里，要等条件为真后 `nextTick` 再初始化。

**现象**：路由参数变了，页面数据不刷新。
**原因**：同组件复用，`setup` 只执行一次，路由参数变化不会重新执行。
**处理**：`watch(() => route.params.id, load, { immediate: true })`。

**现象**：页面切走后定时器还在跑，内存和请求都涨。
**原因**：只写了 `onMounted` 没写 `onUnmounted`。
**处理**：所有 `setInterval`、`addEventListener`、第三方实例都在 `onUnmounted` 里清理。

---

## 相关主题延伸阅读

- [Pinia入门（专业版）](../Pinia入门/Pinia入门_專業版.md) — 本章只讲了组件内状态，跨页面共享的状态必须交给 Pinia，读完本章紧接着读它
- [项目起步（专业版）](../项目起步/项目起步_專業版.md) — 想把手写的代码放进真实工程，需要先知道 create-vue 的目录约定与路径别名
- [商品详情（专业版）](../商品详情/商品详情_專業版.md) — 本章的 ref 与 computed 会在 SKU 选择里被真正用起来，那里有最典型的派生状态设计
- [购物车（专业版）](../购物车/购物车_專業版.md) — 购物车的全选、半选状态是本章「computed 与 watch 分界」的实战检验
- [支付页（专业版）](../支付页/支付页_專業版.md) — 支付倒计时与轮询是 watch 与生命周期清理的集中考场

# 第 2 章 Vue 3 核心与 Element Plus

> 本章共 5 课时（第 7–11 课），目标是：**掌握组合式 API 与 `<script setup>` 的写法，理解组件通信与插槽，能说清生命周期与 `keep-alive` 的关系，会注册与使用全局组件和指令，并能用 Element Plus 完成带校验的表单与表格页面。**
> 对应大纲：`docs/课程大纲.md` 第 2 章。
> 前置要求：完成第 0、1 章；能在本地运行 `yarn dev`。

---

## 本章学习地图

```
第 7 课 组合式 API 与 <script setup>
   └─► 第 8 课 组件通信与插槽
          └─► 第 9 课 生命周期与 keep-alive
                 └─► 第 10 课 全局注册、自定义指令与插件
                        └─► 第 11 课 Element Plus 实战与主题
```

本章完成后，你应当能够：

1. 用 `ref`、`reactive`、`computed`、`watch` 写出一个带状态的组件，并解释它们的区别。
2. 设计父子组件之间的 `props`、`emit`、`v-model` 通信，并使用具名插槽与作用域插槽。
3. 说明 `onMounted`、`nextTick` 的使用时机，并解释 `keep-alive` 如何依赖组件 `name`。
4. 说明全局注册与局部引入各自的取舍，并编写一个自定义指令。
5. 用 Element Plus 的表单、表格、弹窗完成一个带校验的增删改页面，并理解项目的暗黑模式与尺寸配置。

---

## 第 7 课：组合式 API 与 `<script setup>`

### 7.1 学习目标

- 区分选项式 API 与组合式 API 的组织方式。
- 熟练使用 `ref`、`reactive`、`toRefs`、`computed`、`watch`、`watchEffect`。
- 理解 `<script setup>` 的语法糖：顶层变量自动暴露给模板，`defineProps`、`defineEmits` 的用法。
- 能读懂 `src/views/system/user/index.vue` 中的状态组织方式。

### 7.2 课前准备

- 阅读 Vue 官方文档的"响应式基础"与"组合式 API"章节（<https://cn.vuejs.org>）。
- 打开 `src/views/system/user/index.vue`，定位 `<script setup>` 部分（约第 188 行之后）。

### 7.3 知识讲解

#### （一）两种 API 风格

选项式（Options API）把代码按"选项"分类：

```js
export default {
  data() { return { count: 0 } },
  methods: { add() { this.count++ } },
  mounted() { /* ... */ }
}
```

组合式（Composition API）把代码按"逻辑"组织：同一功能的状态、方法、生命周期写在一起，便于复用与拆分。

本项目的业务页面统一采用 **组合式 API + `<script setup>`**。这是 Vue 3 当前推荐的写法，也是阅读本仓库代码的主要前提。

#### （二）`<script setup>` 的基本形态

```vue
<template>
  <el-button @click="add">{{ count }}</el-button>
</template>

<script setup>
import { ref } from 'vue'      // 本项目中 ref 也会被自动导入，显式写出便于理解
const count = ref(0)
function add() { count.value++ }
</script>
```

要点：

- 顶层声明的变量与函数 **自动暴露给模板**，无需 `return`。
- 组件无需 `export default` 与 `components` 注册，子组件导入即可直接使用（如 `import TreePanel from '@/components/TreePanel'`）。
- 编译后的代码在 `setup` 函数内执行，每个组件实例都有独立的状态。

#### （三）响应式基础：`ref` 与 `reactive`

| API | 适用场景 | 访问方式 | 注意事项 |
| --- | --- | --- | --- |
| `ref(值)` | 基本类型，或需要整体替换的对象 | 脚本中 `.value`，模板中自动解包 | 忘记 `.value` 是最常见的错误 |
| `reactive(对象)` | 一组相关字段 | 直接访问属性 | 解构会丢失响应式；替换整个对象会断开引用 |
| `toRefs(reactiveObj)` | 把 reactive 的每个属性转为 ref | 解构后仍保持响应式 | 解构后脚本中仍需 `.value` |

在用户列表页中，我们可以看到这样的组织方式（节选）：

```js
const userList = ref([])          // 列表数据：数组用 ref
const open = ref(false)           // 弹窗开关：布尔值用 ref
const dateRange = ref([])

const data = reactive({           // 一组相关的表单与查询条件用 reactive
  form: {},
  queryParams: { pageNum: 1, pageSize: 10, userName: undefined /* ... */ },
  rules: { userName: [{ required: true, message: '用户名称不能为空', trigger: 'blur' }] /* ... */ }
})

const { queryParams, form, rules } = toRefs(data)   // 解构后仍为响应式
```

> **设计思路**：把相关的查询条件、表单、校验规则放在一个 `reactive` 对象中，可以在语义上归为一组；配合 `toRefs` 解构后，模板与脚本都能方便地访问。

#### （四）`computed`：派生状态

```js
const pageCount = computed(() => Math.ceil(total.value / queryParams.value.pageSize))
```

- `computed` 是**带缓存的派生值**：依赖不变时不会重复计算。
- 它是只读的（默认）；如需可写，可传入 `{ get, set }`。
- 在 `Pagination` 组件中，我们会看到 `computed` 的读写形式（第 8 课）。

#### （五）`watch` 与 `watchEffect`

```js
// watch：明确指定监听源，可拿到新旧值
watch(() => props.modelValue, (v) => { /* ... */ })

// watchEffect：自动收集函数内用到的响应式依赖，立即执行一次
watchEffect(() => { addIframe() })
```

项目中的实例：

- `src/layout/components/AppMain.vue` 同时使用了 `onMounted` 与 `watchEffect` 来调用 `addIframe()`。
- `src/components/Editor/index.vue` 使用 `watch(() => props.modelValue, ...)` 同步外部传入的富文本内容。

**选择原则**：需要旧值、需要精确控制监听源时用 `watch`；只关心"用到了什么就跟着变"时用 `watchEffect`。

#### （六）`defineProps` 与 `defineEmits`

```js
const props = defineProps({
  total: { required: true, type: Number },
  page:  { type: Number, default: 1 },
  limit: { type: Number, default: 20 }
})
const emit = defineEmits()        // 本项目 Pagination 中未声明事件名，直接 emit 字符串
```

`Pagination` 组件（`src/components/Pagination/index.vue`）中可以看到完整示例，我们在第 8 课再详细展开。

### 7.4 代码阅读任务

1. 在 `src/views/system/user/index.vue` 中，列出所有 `ref(` 与 `reactive(` 的声明，并分类：哪些是基本类型，哪些是对象。
2. 找出一个 `computed` 与一个 `watch` 的使用位置（可在 `src` 中搜索 `computed(` 与 `watch(`）。
3. 对比 `src/components/Pagination/index.vue` 中 `currentPage` 的 `computed` 写法，解释它为什么要写成 `get` / `set` 形式。

### 7.5 动手实践（课后任务）

**任务 A：写一个计数器组件**
- 在 `src/components/Demo/Counter.vue` 中用 `<script setup>` 实现计数器，要求：
  - 使用 `ref` 保存计数；
  - 用 `computed` 计算"是否为偶数"；
  - 用 `watch` 在计数超过 10 时打印一条提示。
- 在某个页面中引用它，验证效果。完成后删除实验文件。

**任务 B：`reactive` 的解构陷阱**
1. 写一段代码：`const state = reactive({ n: 1 }); const { n } = state; n++`。
2. 观察 `state.n` 是否变化，解释原因。
3. 改为 `const { n } = toRefs(state)`，再次验证，并记录结论。

**任务 C：替换整个 reactive 对象的影响**
- 在笔记中推演：如果执行 `data = reactive({...})` 重新赋值（而不是修改属性），`toRefs` 解构出的变量会怎样？为什么项目中的写法是修改属性而不是替换对象？

### 7.6 课堂自测

1. 组合式 API 相比选项式 API 的主要优势是什么？
2. `ref` 与 `reactive` 在模板中访问时有什么区别？
3. 为什么解构 `reactive` 对象会丢失响应式？如何解决？
4. `computed` 与 `watch` 的核心区别是什么？
5. `<script setup>` 中，为什么不需要写 `return`？

<details>
<summary>参考答案（先自己思考再查看）</summary>

1. 按逻辑组织代码，同一功能的状态与方法集中在一起；逻辑更易复用（如组合式函数）、拆分与测试。
2. 脚本中 `ref` 需要 `.value`，模板中会自动解包，直接写名字即可；`reactive` 的属性在脚本与模板中都直接访问。
3. 解构时取出的是普通属性值（原始值或对象引用），与原对象的响应式连接已断开。用 `toRefs` 或 `toRef` 把属性转为 ref 即可保持连接。
4. `computed` 用于派生值，有缓存，重点是"计算结果"；`watch` 用于执行副作用，重点是"数据变化时做某事"。
5. 因为 `<script setup>` 编译器会自动把顶层绑定暴露给模板，相当于隐式地 `return` 了所有顶层变量与函数。

</details>

---

## 第 8 课：组件通信与插槽

### 8.1 学习目标

- 掌握 `props` 单向数据流与 `emit` 事件的使用。
- 理解 `v-model` 在 Vue 3 中的约定（`modelValue` / `update:modelValue`），以及 `v-model:xxx` 的多绑定。
- 能编写具名插槽与作用域插槽。
- 了解 `provide` / `inject` 的用途，并能说出本项目中替代它的做法。

### 8.2 课前准备

- 阅读 `src/components/Pagination/index.vue` 与 `src/components/Editor/index.vue`。
- 阅读 `src/components/DictTag/index.vue`，了解其 `props` 的定义方式。

### 8.3 知识讲解

#### （一）`props`：自上而下的数据传递

父组件通过属性把数据传给子组件，子组件通过 `defineProps` 声明接收：

```vue
<!-- 父组件 -->
<pagination :total="total" v-model:page="queryParams.pageNum" v-model:limit="queryParams.pageSize" @pagination="getList" />
```

```js
// Pagination 子组件
const props = defineProps({
  total: { required: true, type: Number },
  page:  { type: Number, default: 1 },
  limit: { type: Number, default: 20 }
})
```

**单向数据流原则**：子组件 **不应直接修改 props**。需要修改时，应通过事件通知父组件，由父组件修改自己的状态。

#### （二）`emit`：自下而上的事件

```js
const emit = defineEmits()
emit('update:page', 2)                               // 更新 page
emit('pagination', { page: 2, limit: 10 })          // 通知父组件分页变化
```

`Pagination` 组件中的关键逻辑：

```js
const currentPage = computed({
  get() { return props.page },                       // 读：来自父组件
  set(val) { emit('update:page', val) }              // 写：通知父组件更新
})
```

这是 **"computed 的 get/set + emit"** 的经典写法：组件内部使用 `currentPage` 就像普通变量，但实际的状态仍由父组件掌控。

#### （三）`v-model`：双向绑定的语法糖

Vue 3 中 `v-model` 的约定是：

- 默认：`v-model="x"` 等价于 `:modelValue="x"` + `@update:modelValue="x = $event"`。
- 具名：`v-model:page="p"` 等价于 `:page="p"` + `@update:page="p = $event"`。

项目中的实际应用：

| 组件 | 用法 | 说明 |
| --- | --- | --- |
| `Pagination` | `v-model:page`、`v-model:limit` | 同时绑定两个属性 |
| `Editor` | 通过 `modelValue` 接收，`emit('update:modelValue', content)` 回传 | 富文本内容双向同步 |
| `FileUpload` / `ImageUpload` | 同样使用 `modelValue` 约定 | 上传结果双向同步 |

`Editor` 中的回传代码：

```vue
<!-- 编辑器内容变化时，通知父组件更新 modelValue -->
@textChange="(e) => $emit('update:modelValue', content)"
```

#### （四）事件与 `$emit`

- 模板中可直接使用 `$emit`，效果与 `emit()` 相同。
- 项目中的 `pagination` 事件是一个约定俗成的命名：父组件监听 `@pagination="getList"`，即可在分页变化时重新查询。

#### （五）插槽（Slots）

插槽让父组件向子组件"注入"模板内容。

**默认插槽**：子组件 `<slot />`，父组件直接写内容。

**具名插槽**：

```vue
<!-- 子组件 -->
<header><slot name="header" /></header>
<main><slot /></main>

<!-- 父组件 -->
<template #header><h2>标题</h2></template>
<p>默认内容</p>
```

**作用域插槽**：子组件可以把数据传给插槽，父组件决定如何渲染。这在表格列自定义渲染中最常见：

```vue
<el-table-column label="状态">
  <template #default="scope">
    <dict-tag :options="sys_normal_disable" :value="scope.row.status" />
  </template>
</el-table-column>
```

这里 `scope.row` 来自 `el-table` 的作用域插槽数据。它是 Element Plus 的约定，业务代码通过它读取每一行的数据。

#### （六）`DictTag`：一个典型的展示型组件

`src/components/DictTag/index.vue` 的 `props` 包括 `options`（字典数组）、`value`（当前值）、`separator`（多值分隔符）。它内部用 `computed` 把 `value` 统一转为数组，再根据 `options` 找出对应的标签与样式。

这种"只负责展示、不关心数据从哪来"的组件，是可复用性的典型体现。

#### （七）`provide` / `inject`

- `provide` 在祖先组件中提供数据，`inject` 在后代组件中注入，中间层组件无需逐层传递。
- 适合跨多层的全局性数据，如主题、表单上下文。
- **本项目中较少使用** `provide/inject`（我们检索 `src` 时没有找到业务使用）。跨组件的共享状态主要通过 **Pinia store**（第 3 章）与 **插件**（如 `useDict`、`$modal`）实现。

> 选择建议：父子层级少用 `props` / `emit`；跨多层或全局共享用 Pinia；只在一个子树内共享、又不想逐层传递，才考虑 `provide/inject`。

### 8.4 代码阅读任务

1. 在 `src/views/system/user/index.vue` 中找到 `<pagination>` 的使用，列出它绑定的所有属性与事件。
2. 阅读 `src/components/Editor/index.vue`，画出"父组件 → Editor → 父组件"的数据流图。
3. 在 `src` 中搜索 `<template #`，找出三个不同的插槽使用位置，并说明各自属于哪种插槽。

### 8.5 动手实践（课后任务）

**任务 A：实现一个 `v-model` 组件**
1. 写一个 `StatusSwitch.vue`，接收 `modelValue`（布尔值），点击时 `emit('update:modelValue', !value)`。
2. 在父组件中用 `v-model` 绑定，验证双向同步。
3. 尝试在子组件中直接修改 `props.modelValue`，观察控制台的警告信息，并解释原因（单向数据流）。

**任务 B：具名插槽卡片**
- 实现一个 `InfoCard.vue`，包含 `header`、默认插槽与 `footer` 三个插槽。
- 在某个页面中使用，并验证各插槽位置。

**任务 C：作用域插槽**
- 在一个列表页的表格中，用作用域插槽把 `status` 列渲染为带颜色的文字（可用 `el-tag`）。

**任务 D：阅读分析**
- 对比 `Pagination` 的 `pagination` 事件与直接 `update:page` 事件，分析为什么项目要同时提供两种通知方式。

### 8.6 课堂自测

1. 为什么子组件不应直接修改 `props`？正确的做法是什么？
2. `v-model:page="x"` 等价于什么写法？
3. 作用域插槽与普通插槽的区别是什么？它在 `el-table` 中的典型应用是什么？
4. `provide/inject` 适合什么场景？本项目为什么较少使用它？
5. `Pagination` 中 `currentPage` 的 `computed` 为什么要写 `get` 和 `set`？

<details>
<summary>参考答案（先自己思考再查看）</summary>

1. 为了保持单向数据流，让状态的修改位置可预测。正确做法是子组件 `emit` 事件，由父组件修改自己的数据。
2. 等价于 `:page="x"` 加上 `@update:page="x = $event"`。
3. 普通插槽只传递模板内容；作用域插槽还能把子组件的数据回传给插槽内容。`el-table` 的 `#default="scope"` 就是典型应用，用来读取每一行的 `row`。
4. 适合跨多层的数据或子树内的共享上下文。本项目跨组件共享主要通过 Pinia 与插件完成，层级关系也比较清晰，所以较少使用。
5. `get` 把父组件的值作为读取来源，`set` 把写入转为 `emit`，使组件内部可以像普通变量一样读写，而状态仍由父组件掌握。

</details>

---

## 第 9 课：生命周期与 keep-alive 缓存

### 9.1 学习目标

- 说出 `onMounted`、`onUnmounted`、`onActivated`、`onDeactivated` 等生命周期钩子的典型用途。
- 理解 `nextTick` 的作用：等待 DOM 更新后再执行操作。
- 理解 `<keep-alive>` 的缓存机制，以及为什么它依赖组件 `name`。
- 理解项目中 `meta.noCache`、`tagsView` 与缓存列表之间的联动。

### 9.2 课前准备

- 阅读 `src/layout/components/AppMain.vue`。
- 阅读 `src/store/modules/tagsView.js` 中与 `cachedViews` 相关的方法（`addCachedView`、`delCachedView`）。

### 9.3 知识讲解

#### （一）常用生命周期钩子

| 钩子 | 触发时机 | 典型用途 |
| --- | --- | --- |
| `onMounted` | 组件挂载到 DOM 后 | 发起数据请求、初始化第三方库（如 ECharts） |
| `onUpdated` | 响应式数据变化导致 DOM 更新后 | 较少使用，需注意避免死循环 |
| `onUnmounted` | 组件卸载后 | 清理定时器、事件监听、订阅 |
| `onActivated` | 被 `keep-alive` 缓存的组件重新显示时 | 缓存页面中刷新数据 |
| `onDeactivated` | 被 `keep-alive` 缓存的组件隐藏时 | 暂停轮询等耗资源操作 |

项目中的典型用法（用户列表页）：

```js
onMounted(() => {
  getDeptTree()   // 加载部门树
  getList()       // 加载用户列表
})
```

#### （二）`nextTick`：等待 DOM 更新

Vue 的数据更新是**异步批量**进行的。修改数据后，DOM 不会立即更新。如果需要操作更新后的 DOM，要用 `nextTick`：

```js
// 项目中 settings store 的写法
toggleTheme() {
  this.isDark = !this.isDark
  toggleDark()
  nextTick(() => {          // 等 DOM 与 class 更新完成后再处理主题样式
    handleThemeStyle(this.theme)
  })
}
```

#### （三）`<keep-alive>`：组件缓存

`AppMain.vue` 中的核心代码：

```vue
<router-view v-slot="{ Component, route }">
  <transition name="fade-transform" mode="out-in">
    <keep-alive :include="tagsViewStore.cachedViews">
      <component v-if="!route.meta.link" :is="Component" :key="route.path" />
    </keep-alive>
  </transition>
</router-view>
```

机制解释：

- `<keep-alive>` 会缓存被切换掉的组件实例，而不是销毁它们。再次进入时，恢复实例的状态（如表单输入、滚动位置）。
- `:include="tagsViewStore.cachedViews"`：只有列表中的组件名会被缓存。`cachedViews` 由标签页的 store 维护。
- `:key="route.path"`：同一组件不同路径视为不同实例，避免共用状态。
- `v-if="!route.meta.link"`：外链页面不走 `<component>` 渲染。

#### （四）缓存依赖组件 `name`

`keep-alive` 的 `include` 匹配的是**组件名**（`name`），而不是路由名或文件名。

- 标准 `<script setup>` 没有 `name` 选项。
- 因此项目通过 `unplugin-vue-setup-extend-plus` 插件（第 5 课）支持 `<script setup name="Cache">` 写法。
- 若组件没有可用的 `name`，它就不会被 `include` 匹配，也就无法被缓存。

> **思考**：`src/views/monitor/cache/index.vue` 声明了 `name="Cache"`，所以当 `Cache` 被加入 `cachedViews` 时，它会被缓存。

#### （五）缓存列表如何变化

`src/store/modules/tagsView.js` 维护了 `cachedViews`，在打开页面、关闭页面时同步增删。关键动作：

- `addCachedView(view)`：打开页面时，若该页面 `meta.noCache` 不为 true，则把组件名加入缓存。
- `delCachedView(view)`：关闭标签页时，移除对应缓存。
- `delOthersCachedViews`、`delAllCachedViews`：批量关闭时，清理相应缓存。

`meta.noCache: true` 的路由（如某些表单编辑页）不会被缓存，每次进入都是新实例。

#### （六）`onActivated` 与"缓存页面的数据刷新"

被缓存的页面切回时，`onMounted` **不会再次触发**，因为组件实例没有被销毁。如果需要每次进入都刷新数据，应使用 `onActivated`：

```js
onActivated(() => {
  getList()   // 每次从其它页面切回时刷新
})
```

### 9.4 代码阅读任务

1. 阅读 `AppMain.vue`，说明 `:include` 绑定的数据从哪里来，列出它的变化时机。
2. 在 `src` 中搜索 `onActivated`，判断是否有使用。如果没有，思考：如果列表页被缓存，用户切回后看到的数据可能是旧的，应如何解决？
3. 阅读 `tagsView.js` 中 `addCachedView` 的判断条件，记录 `noCache` 的处理方式。

### 9.5 动手实践（课后任务）

**任务 A：观察生命周期**
1. 在某个列表页中，分别在 `onMounted`、`onUnmounted`、`onActivated`、`onDeactivated` 中打印日志（可用 `console.log`）。
2. 操作：进入页面 → 切到其它标签 → 切回 → 关闭标签。
3. 记录每个阶段触发了哪些钩子，并解释原因。实验后删除日志。

**任务 B：缓存与 name 的关系**
1. 找一个 `<script setup>` 组件（如 `src/views/index.vue`），删除 `name` 属性（备份后操作）。
2. 打开它，切换到其它标签，观察它是否还能保留状态（可在页面中放一个输入框验证）。
3. 恢复配置，写下结论。

**任务 C：`nextTick` 实验**
- 写一个组件：修改某个数据后，立即读取一个 DOM 元素的文本，记录结果；再用 `nextTick` 包裹读取，对比两者差异。

### 9.6 课堂自测

1. `onMounted` 与 `onActivated` 的触发时机有什么不同？分别适合做什么？
2. 为什么修改数据后不能立即读取 DOM？`nextTick` 解决什么问题？
3. `<keep-alive>` 缓存的是什么？它为什么需要组件名？
4. `:key="route.path"` 的作用是什么？
5. `meta.noCache` 设置为 true 会产生什么效果？

<details>
<summary>参考答案（先自己思考再查看）</summary>

1. `onMounted` 在组件首次挂载后执行一次；`onActivated` 在被缓存的组件每次重新显示时执行。前者适合首次加载数据，后者适合切回时刷新数据。
2. Vue 批量异步更新 DOM，修改数据后 DOM 尚未更新。`nextTick` 把回调推迟到 DOM 更新完成之后。
3. 缓存的是组件实例（保留其状态与 DOM）。`include` 通过组件名匹配，所以需要 `name`。
4. 让同一组件在不同路径下生成不同实例，避免路由切换时复用错误的状态。
5. 该页面不会被 `keep-alive` 缓存，每次进入都会重新创建实例，状态不会保留。

</details>

---

## 第 10 课：全局注册、自定义指令与插件

### 10.1 学习目标

- 理解 `main.js` 中全局方法（`globalProperties`）与全局组件（`app.component`）的注册方式。
- 理解全局注册的收益与代价，并能判断哪些组件适合全局注册。
- 掌握自定义指令的生命周期钩子，能阅读并编写指令。
- 理解 `app.use(plugin)` 插件机制的作用。

### 10.2 课前准备

- 打开 `src/main.js`，找到"全局方法挂载""全局组件挂载"两段代码。
- 打开 `src/directive/index.js`、`src/directive/permission/hasPermi.js`、`src/directive/common/copyText.js`。

### 10.3 知识讲解

#### （一）全局方法：`app.config.globalProperties`

```js
app.config.globalProperties.parseTime = parseTime
app.config.globalProperties.resetForm = resetForm
app.config.globalProperties.handleTree = handleTree
app.config.globalProperties.useDict = useDict
app.config.globalProperties.download = download
```

注册后，在模板中可直接调用（如 `parseTime(scope.row.createTime)`）。在 `<script setup>` 中则通过 `getCurrentInstance()` 获取：

```js
const { proxy } = getCurrentInstance()
proxy.resetForm("queryRef")
proxy.addDateRange(queryParams.value, dateRange.value)
```

这正是我们在用户页中看到的写法。

> **注意**：`globalProperties` 不是响应式的，也没有类型提示。过度使用会让代码"依赖来源不明"。本项目中大部分工具函数已通过自动导入（第 5 课）直接 import 使用，`globalProperties` 主要保留给历史写法和模板场景。

#### （二）全局组件：`app.component`

```js
app.component('DictTag', DictTag)
app.component('Pagination', Pagination)
app.component('RightToolbar', RightToolbar)
app.component('Editor', Editor)
// ...
```

注册后，任何模板中都可以直接写 `<pagination>`、`<dict-tag>`，无需导入。

#### （三）全局注册的取舍

| 维度 | 全局注册 | 局部引入（`import`） |
| --- | --- | --- |
| 使用便利 | 高，模板里直接用 | 需要在每个文件导入 |
| 依赖可见性 | 低，看模板时不知道组件来源 | 高，导入语句即依赖说明 |
| 打包体积 | 只要注册了，即使不用也会打包 | 按需引入，未使用不打包 |
| 适用场景 | 高频、通用、几乎每个页面都用 | 低频、特定页面使用 |

**建议**：`Pagination`、`DictTag`、`RightToolbar` 这类高频通用组件适合全局；`TreePanel`、`IconSelect` 这类只在少数页面使用的组件，更适合局部引入。本项目两种方式都有，你可以在阅读中对照判断。

#### （四）自定义指令：`src/directive`

指令的结构由生命周期钩子组成，常用的有：

| 钩子 | 时机 |
| --- | --- |
| `created` / `beforeMount` | 元素创建后，挂载前 |
| `mounted` | 元素插入 DOM 后 |
| `updated` | 所在组件更新后 |
| `unmounted` | 元素被移除时 |

项目中的 `v-hasPermi` 使用的是 `mounted` 钩子：

```js
export default {
  mounted(el, binding, vnode) {
    const { value } = binding                     // 指令的值，如 ['system:user:add']
    const permissions = useUserStore().permissions
    // 若无权限，直接从 DOM 中移除该元素
    if (!hasPermissions) el.parentNode && el.parentNode.removeChild(el)
  }
}
```

**参数说明**：
- `el`：指令绑定的 DOM 元素；
- `binding`：包含 `value`（绑定值）、`arg`（参数）、`modifiers`（修饰符）等；
- `vnode`：虚拟节点（较少使用）。

`copyText` 指令则使用了 `beforeMount`，并为元素绑定 `click` 事件：

```js
beforeMount(el, { value, arg }) {
  el.$copyValue = value
  el.addEventListener('click', handler)
  el.$destroyCopy = () => el.removeEventListener('click', handler)
}
```

这里有一个重要细节：`copyText` 注册了点击监听，但在 `unmounted` 时并没有调用 `$destroyCopy`。这是阅读时可以思考的点，我们在第 39 课讨论"内存与事件清理"时会回到它。

#### （五）指令的注册入口：`src/directive/index.js`

```js
export default function directive(app) {
  app.directive('hasRole', hasRole)
  app.directive('hasPermi', hasPermi)
  app.directive('copyText', copyText)
}
```

这是一种"集中注册"的模式：`main.js` 只需调用 `directive(app)`，所有指令的增删都集中在一个文件中。

#### （六）插件：`app.use(plugin)`

插件是一个带 `install(app, options)` 方法的对象或函数，用于一次性扩展应用。`src/plugins/index.js` 中的插件通常会：

- 挂载全局属性（如 `app.config.globalProperties.$modal`、`$tab`、`$auth`、`$cache`）；
- 注册全局指令或组件；
- 提供统一的 API 封装。

项目中的 `$modal`（消息弹窗）、`$tab`（标签页操作）就是通过插件注入的（第 30 课、第 19 课会深入）。

### 10.4 代码阅读任务

1. 在 `src/main.js` 中，列出所有 `app.config.globalProperties` 的属性，并判断哪些已经被自动导入替代。
2. 在 `src` 中搜索 `v-hasRole` 与 `v-hasPermi`，各找出一个使用位置，并判断它们控制的是按钮还是菜单。
3. 阅读 `src/directive/permission/hasRole.js`，对比它与 `hasPermi` 的实现差异。

### 10.5 动手实践（课后任务）

**任务 A：实现 `v-focus` 指令**
1. 在 `src/directive/common/focus.js` 中实现：元素挂载后自动聚焦（`mounted(el) { el.focus() }`，注意 input 元素需要找到内部的 `input`）。
2. 在 `src/directive/index.js` 中注册。
3. 在某个搜索框上使用 `v-focus`，验证页面加载后自动聚焦。
4. 完成后可保留（如果你愿意作为学习成果），或删除实验代码。

**任务 B：全局 vs 局部对比**
- 选择 `TreePanel` 组件，写出它在某个页面中的两种引入方式（全局注册与局部导入），并列出各自的优缺点。

**任务 C：指令的卸载**
- 修改 `copyText` 的副本，在 `unmounted` 钩子中调用 `el.$destroyCopy()`，并解释它为什么能防止事件泄漏。（只在练习副本中修改，不要改动原文件。）

### 10.6 课堂自测

1. `globalProperties` 与 `app.component` 的主要区别是什么？
2. 全局注册的主要代价是什么？
3. 自定义指令的 `mounted` 与 `beforeMount` 有什么区别？
4. `v-hasPermi` 为什么要用 `mounted` 而不是 `created`？（提示：思考 DOM 的存在时机。）
5. 插件与普通的工具函数相比，有什么优势？

<details>
<summary>参考答案（先自己思考再查看）</summary>

1. `globalProperties` 挂载的是可通过实例访问的属性或方法，在模板与 `proxy` 中使用；`app.component` 注册的是可在模板中直接使用的组件标签。
2. 依赖来源不明显、可能增加打包体积（即使未使用也会打包）、命名冲突风险。
3. `beforeMount` 在元素挂载前触发，元素尚未插入 DOM；`mounted` 在元素插入 DOM 之后触发。`hasPermi` 需要操作父节点（`el.parentNode`）来移除元素，所以要在插入 DOM 后执行。
4. `created` 时元素还未挂载到文档中，`parentNode` 可能为空；`mounted` 时元素已插入 DOM，才能安全地移除它。
5. 插件可以统一地扩展应用（全局属性、组件、指令、依赖注入），在 `app.use` 时一次性完成，并且通常会在应用级别共享状态或配置。

</details>

---

## 第 11 课：Element Plus 实战与主题

### 11.1 学习目标

- 使用 `el-form` 完成表单的数据绑定与校验。
- 使用 `el-table` 与 `el-dialog` 完成列表与弹窗交互。
- 理解全局尺寸、语言包与暗黑模式的配置位置。
- 能完成一个带必填、长度、手机号格式校验的表单弹窗。

### 11.2 课前准备

- 阅读 Element Plus 文档（<https://element-plus.org/zh-CN/>）中的 Form、Table、Dialog 章节。
- 打开 `src/views/system/user/index.vue` 中的 `<el-dialog>` 与 `<el-form>` 部分。

### 11.3 知识讲解

#### （一）表单：`el-form` 的三件套

```vue
<el-form :model="form" :rules="rules" ref="userRef" label-width="80px">
  <el-form-item label="用户名称" prop="userName">
    <el-input v-model="form.userName" placeholder="请输入用户名称" />
  </el-form-item>
</el-form>
```

| 属性 | 作用 |
| --- | --- |
| `:model` | 表单数据对象，`v-model` 绑定的字段都在其中 |
| `:rules` | 校验规则，键名与 `form-item` 的 `prop` 一致 |
| `ref` | 获取表单实例，用于手动调用 `validate()` |
| `prop` | 标记该表单项对应 `model` 中的哪个字段，并关联规则 |

用户页中的校验规则（节选）：

```js
rules: {
  userName: [
    { required: true, message: '用户名称不能为空', trigger: 'blur' },
    { min: 2, max: 20, message: '用户名称长度必须介于 2 和 20 之间', trigger: 'blur' }
  ],
  email: [{ type: 'email', message: '请输入正确的邮箱地址', trigger: ['blur', 'change'] }],
  phonenumber: [{ pattern: /^1[3|4|5|6|7|8|9][0-9]\d{8}$/, message: '请输入正确的手机号码', trigger: 'blur' }]
}
```

规则要点：
- `required`：必填；`min`/`max`：长度范围；`type: 'email'`：内置邮箱格式；`pattern`：自定义正则。
- `trigger`：触发时机，`blur`（失焦）、`change`（值变化）。

**提交前校验**：

```js
function submitForm() {
  proxy.$refs['userRef'].validate(valid => {
    if (valid) { /* 调用接口 */ }
  })
}
```

`validate` 回调中的 `valid` 为 `true` 才应继续提交。

#### （二）动态校验规则：`usePasswordRule`

用户密码的校验规则不是静态写死的，而是通过 `usePasswordRule()` 生成，并通过 `form-item` 的局部 `:rules` 覆盖：

```vue
<el-form-item v-if="form.userId == undefined" label="用户密码" prop="password" :rules="pwdValidator">
```

这说明：**`form-item` 上的 `:rules` 可以覆盖表单级的规则**，适合规则需要根据运行时配置变化的场景（第 29 课会详细讲）。

#### （三）表格：`el-table` 与列配置

```vue
<el-table v-loading="loading" :data="userList" @selection-change="handleSelectionChange">
  <el-table-column type="selection" width="50" align="center" />
  <el-table-column label="用户编号" align="center" prop="userId" />
  <el-table-column label="状态" align="center">
    <template #default="scope">
      <dict-tag :options="sys_normal_disable" :value="scope.row.status" />
    </template>
  </el-table-column>
</el-table>
```

- `:data`：表格数据；`prop`：直接显示该字段；
- `type="selection"`：多选列，触发 `selection-change` 事件，项目用它维护 `ids`、`single`、`multiple`；
- 需要自定义渲染时使用作用域插槽（第 8 课）。

#### （四）弹窗：`el-dialog`

```vue
<el-dialog :title="title" v-model="open" width="500px" append-to-body>
  <!-- 表单内容 -->
  <template #footer>
    <el-button type="primary" @click="submitForm">确 定</el-button>
    <el-button @click="cancel">取 消</el-button>
  </template>
</el-dialog>
```

- `v-model="open"` 控制显示与隐藏；
- `append-to-body` 将弹窗挂到 `body` 下，避免被父容器样式影响；
- `#footer` 插槽放置按钮。

#### （五）全局配置：尺寸与语言

`src/main.js` 中：

```js
import locale from 'element-plus/es/locale/lang/zh-cn'

app.use(ElementPlus, {
  locale: locale,                                  // 中文语言包：分页、日期等文案
  size: Cookies.get('size') || 'default'           // 组件尺寸，来自 Cookie
})
```

- `locale`：统一切换组件内部文案（如分页的"条/页"、日期选择器的"今天"）。
- `size`：全局组件尺寸，默认 `default`，可通过 Cookie 记录用户偏好。这也是右上角"尺寸"设置的数据来源（`SizeSelect` 组件）。

#### （六）暗黑模式

- 入口：`src/main.js` 导入 `element-plus/theme-chalk/dark/css-vars.css`，它定义了 Element Plus 的暗色 CSS 变量。
- 切换：`src/store/modules/settings.js` 使用 `useDark()`（来自 `@vueuse/core`）。它会在 `<html>` 上添加或移除 `dark` 类。
- 设置持久化：`localStorage` 的 `layout-setting` 记录了布局设置（见 `settings.js` 中的读取逻辑）。
- 效果：当 `html.dark` 存在时，Element Plus 的 CSS 变量切换为暗色值，组件随之变色。

> **原理**：Element Plus 的样式大量使用 CSS 变量，因此切换主题不需要重新渲染组件，只需要切换变量取值。这是现代组件库做暗黑模式的常用方式。

### 11.4 操作步骤（综合练习：带校验的表单弹窗）

以"公告分类"为例（不涉及后端，纯前端练习）：

1. 在 `src/views/demo/category.vue` 中创建页面。
2. 使用 `reactive` 定义 `form`（字段：`name`、`code`、`phone`）与 `rules`：
   - `name`：必填，长度 2–10；
   - `code`：必填，只允许字母与数字（用 `pattern`）；
   - `phone`：手机号格式（复用用户页的正则）。
3. 放置一个 `el-button`，点击后打开 `el-dialog`。
4. 弹窗中放置 `el-form`，绑定 `form` 与 `rules`，提交时调用 `validate`。
5. 校验通过后，把数据 `console.log` 并关闭弹窗；失败时保留弹窗。
6. 使用 `reset` 方法（`proxy.resetForm`）在关闭时清空表单。

### 11.5 动手实践（课后任务）

**任务 A：完成上述综合练习**，并截图记录"校验失败提示"与"校验通过后的数据"。

**任务 B：暗黑模式观察**
1. 在页面右上角切换暗黑模式（如果你的页面包含设置入口）。
2. 在 `<html>` 元素上观察 `class` 的变化。
3. 切换后刷新页面，观察模式是否保留，并解释原因（提示：`useDark` 与 `localStorage`）。

**任务 C：尺寸对比**
- 修改 Cookie 中的 `size` 值为 `small`，刷新页面，观察按钮与表单的尺寸变化。实验后清除 Cookie。

**任务 D：表格多选**
- 在用户列表中多选三条记录，观察 `ids` 与 `multiple` 的变化（可在代码中临时打印）。实验后删除打印语句。

### 11.6 课堂自测

1. `el-form` 中 `model`、`rules`、`prop` 三者如何关联？
2. `validate` 方法的回调参数是什么？为什么必须判断它？
3. `form-item` 上的 `:rules` 与表单级 `rules` 有什么关系？什么场景下需要用前者？
4. 全局 `size` 与 `locale` 分别影响什么？
5. 暗黑模式为什么可以通过切换 CSS 变量实现，而不需要重新渲染组件？

<details>
<summary>参考答案（先自己思考再查看）</summary>

1. `model` 是数据来源，`rules` 是校验规则集合，`prop` 指定表单项对应的字段名，并使该字段关联到 `rules` 中同名键的规则。
2. 回调参数 `valid` 是布尔值，表示整体校验是否通过。必须判断它，否则校验失败也会执行提交。
3. `form-item` 上的规则优先级更高，会覆盖表单级同名字段的规则。适合规则需要按运行时条件动态生成的场景，例如密码规则依赖后台配置。
4. `locale` 影响组件内置的文案与格式（分页、日期等）；`size` 影响所有组件的默认尺寸。
5. Element Plus 的样式使用 CSS 变量，切换 `dark` 类时只改变变量值，浏览器重新计算样式即可，组件的 JS 逻辑与 DOM 结构保持不变。

</details>

---

## 本章小结

| 课时 | 核心问题 | 关键结论 |
| --- | --- | --- |
| 第 7 课 | 状态如何组织？ | 组合式 API + `<script setup>`；`ref` 与 `reactive` 各司其职；`toRefs` 保持解构响应式 |
| 第 8 课 | 组件如何通信？ | `props` 向下、`emit` 向上、`v-model` 是语法糖；插槽实现内容分发；跨层共享优先用 Pinia |
| 第 9 课 | 页面何时加载、何时缓存？ | `onMounted` 首次加载，`onActivated` 切回刷新；`keep-alive` 依赖组件 `name` |
| 第 10 课 | 通用能力如何注入？ | `globalProperties`、全局组件、指令、插件各有场景；取舍在于可见性与体积 |
| 第 11 课 | 页面 UI 如何搭建？ | `el-form` 三件套 + `rules`；表格作用域插槽；全局尺寸、语言与暗黑模式集中配置 |

**本章的一个核心观念**：Vue 3 组件的本质是"状态 + 视图 + 通信接口"。`props` 与 `emit` 定义了组件的边界，`ref`/`reactive` 管理内部状态，插槽与指令则扩展了视图的灵活性。掌握这三者之间的关系，就能读懂本项目绝大多数页面。

**下一章预告**：第 3 章《状态管理：Pinia》，我们将看到 `user`、`app`、`settings`、`dict`、`tagsView` 等 store 如何把跨页面的状态集中管理起来。

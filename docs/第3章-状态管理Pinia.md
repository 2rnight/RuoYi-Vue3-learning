# 第 3 章 状态管理：Pinia

> 本章共 3 课时（第 12–14 课），目标是：**理解 Pinia 的 store 定义方式，读懂项目中的核心 store（用户、应用、设置、字典、锁屏、标签页），并能说清每类状态的存储位置与生命周期。**
> 对应大纲：`docs/课程大纲.md` 第 3 章。
> 前置要求：完成第 2 章（尤其是 `ref`、`reactive`、`computed` 的用法）。

---

## 本章学习地图

```
第 12 课 Pinia 基础（定义、使用、持久化思路）
   └─► 第 13 课 核心 store 解读（user / app / settings / dict / lock）
          └─► 第 14 课 标签页 store（tagsView）与持久化
```

本章完成后，你应当能够：

1. 用选项式或组合式写法定义一个 store，并在组件与路由守卫中正确使用它。
2. 说明 `state`、`getters`、`actions` 各自的职责，以及它们与组件的关系。
3. 画出登录后用户信息、权限、字典、设置等状态的来源与去向。
4. 区分"内存状态""localStorage""sessionStorage""Cookie"四种存储位置，并说出每个 store 用的是哪一种。
5. 解释标签页 store 的三类数据（`visitedViews`、`cachedViews`、`iframeViews`）各自的用途，以及它们如何与 `keep-alive` 协作。

---

## 第 12 课：Pinia 基础

### 12.1 学习目标

- 理解 Pinia 解决的问题，以及它与 Vuex 的关系。
- 掌握 `defineStore` 的两种写法，并能说明它们的异同。
- 理解 `state`、`getters`、`actions` 的职责边界。
- 理解项目中 `src/store/index.js` 的作用，以及 store 在组件与守卫中的调用时机。

### 12.2 课前准备

- 阅读 `src/store/index.js`、`src/store/modules/user.js` 的前 40 行。
- 阅读 Pinia 官方文档（<https://pinia.vuejs.org/zh/>）的"定义 Store"章节。

### 12.3 知识讲解

#### （一）为什么需要全局状态管理

组件之间的数据传递（第 8 课）适合父子关系。当出现以下情况时，就需要全局状态：

- **跨页面共享**：用户信息、权限、字典数据，几乎每个页面都会用到；
- **跨层级共享**：侧边栏的开合状态，需要被布局、导航、菜单多个组件同时读取；
- **需要持久化或缓存**：标签页列表、锁屏状态，即使刷新页面也要保留。

Pinia 是 Vue 官方推荐的状态管理库，相比 Vuex 更简洁：没有 `mutations`，直接用 `actions` 修改状态，并且天然支持 TypeScript 与组合式 API。

#### （二）全局 Pinia 实例：`src/store/index.js`

```js
const store = createPinia()
export default store
```

在 `src/main.js` 中通过 `app.use(store)` 安装。整个应用只有一个 Pinia 实例，所有 `defineStore` 定义的 store 都挂载在它上面。

#### （三）两种 `defineStore` 写法

**选项式写法**（本项目 `user.js`、`app.js` 等采用）：

```js
const useAppStore = defineStore('app', {
  state: () => ({
    sidebar: { opened: true, withoutAnimation: false, hide: false },
    device: 'desktop',
    size: 'default'
  }),
  getters: {},                 // 本例未使用
  actions: {
    toggleSideBar() { this.sidebar.opened = !this.sidebar.opened }
  }
})
```

**组合式写法**（与 `setup()` 函数类似）：

```js
const useCounterStore = defineStore('counter', () => {
  const count = ref(0)                                  // 相当于 state
  const double = computed(() => count.value * 2)        // 相当于 getters
  function increment() { count.value++ }                // 相当于 actions
  return { count, double, increment }
})
```

| 对比项 | 选项式 | 组合式 |
| --- | --- | --- |
| 风格 | 与 Options API 相似，结构清晰 | 与 `<script setup>` 相似，逻辑聚合 |
| 状态定义 | `state: () => ({...})` | `ref` / `reactive` |
| 派生值 | `getters` | `computed` |
| 方法 | `actions`，通过 `this` 访问 | 普通函数，直接访问变量 |
| 本项目 | 普遍使用 | 未见使用（可作为对比练习） |

> **注意**：选项式 `actions` 中使用 `this` 时，**不要使用箭头函数**，否则 `this` 指向错误。

#### （四）`state`、`getters`、`actions` 的职责

| 部分 | 职责 | 能否异步 | 典型示例 |
| --- | --- | --- | --- |
| `state` | 保存数据 | — | `user.roles`、`dict.dict` |
| `getters` | 基于 state 的派生计算（有缓存） | 否 | 本项目使用较少，可作为练习 |
| `actions` | 修改状态、执行业务逻辑、发起异步请求 | 是 | `user.login()`、`user.getInfo()` |

**原则**：组件应该通过 `actions` 修改状态，而不是直接到处改 `store.xxx = ...`。这样修改入口集中，便于调试与追踪。

#### （五）在组件中使用 store

```js
import useUserStore from '@/store/modules/user'

const userStore = useUserStore()        // 在 setup 中调用
console.log(userStore.name)             // 读取 state
userStore.logOut()                      // 调用 action
```

注意以下几点：

1. `useXxxStore()` 必须在 `setup` 或守卫等"有 Pinia 实例上下文"的地方调用；
2. 调用后得到的是响应式对象，可以直接在模板中使用；
3. 如果想解构 store 而保持响应式，需要用 `storeToRefs`：

```js
import { storeToRefs } from 'pinia'
const { name, roles } = storeToRefs(useUserStore())   // 保持响应式
```

直接解构 `const { name } = useUserStore()` 会丢失响应式，这与第 7 课讲过的 `reactive` 解构问题是同一个原理。

#### （六）在路由守卫中使用 store

本项目的 `src/permission.js` 中大量使用 store：

```js
router.beforeEach(async (to, from) => {
  if (getToken()) {
    const isLock = useLockStore().isLock
    if (useUserStore().roles.length === 0) {
      await useUserStore().getInfo()
      const accessRoutes = await usePermissionStore().generateRoutes()
      // ...
    }
  }
})
```

这里的关键点：**守卫在路由组件加载之前执行，它不在任何组件的 `setup` 中**。但 Pinia 允许在守卫里直接调用 `useXxxStore()`，因为 Pinia 实例已经在 `main.js` 中安装完成（守卫在 `app.use(router)` 之后才会真正运行）。

#### （七）持久化思路

Pinia 默认只把状态保存在内存中，**刷新页面就会丢失**。项目中的几种持久化方式：

| 方式 | 生命周期 | 本项目用例 |
| --- | --- | --- |
| 内存（无持久化） | 页面刷新即清空 | `dict` store、`user` 的部分状态 |
| `localStorage` | 长期保留，直到手动清除 | `layout-setting`、锁屏状态、标签页持久化 |
| `sessionStorage` | 标签页关闭即清空 | 防重复提交缓存（`cache.session`）、`pwrChrtype` |
| Cookie | 可设置过期时间，会随请求发送 | `sidebarStatus`、`size`、Token（`utils/auth.js`） |

项目中提供了 `src/plugins/cache.js` 封装，它统一了 `sessionStorage` 与 `localStorage` 的读写，并支持 JSON：

```js
cache.session.setJSON('sessionObj', requestObj)   // 写入 JSON 到 sessionStorage
cache.local.getJSON('tags-view-visited')          // 从 localStorage 读取 JSON
```

> **思考**：为什么有的状态放 `localStorage`，有的放 Cookie？Token 通常需要随请求自动携带，因此放在 Cookie 或请求头中更合适；布局偏好则只在前端使用，`localStorage` 更方便。

### 12.4 代码阅读任务

1. 在 `src/store/modules` 下列出所有 store 文件，分别记录它们的 store ID（`defineStore` 的第一个参数）。
2. 找出一个调用 `storeToRefs` 的地方（可在 `src` 中搜索）；如果没有，说明你会在什么场景下使用它。
3. 阅读 `src/permission.js`，列出守卫中调用了哪几个 store，并说明每个 store 的作用。

### 12.5 动手实践（课后任务）

**任务 A：写一个计数 store**
1. 在 `src/store/modules/demo.js` 中用选项式写法定义 `demo` store：`state` 有 `count`，`getters` 有 `double`，`actions` 有 `increment` 和 `reset`。
2. 在某个页面中使用它，并用按钮验证。
3. 尝试在组件中直接写 `store.count = 100`，观察是否生效（它在 Pinia 中是允许的，但不推荐）。讨论为什么仍建议用 action。
4. 实验后删除文件。

**任务 B：组合式写法对比**
- 把任务 A 的 store 改写为组合式写法，比较两者的代码行数与可读性，写下你的偏好与理由。

**任务 C：持久化实验**
1. 在 `demo` store 的 `count` 变化时，同时写入 `localStorage`。
2. 刷新页面，观察值是否保留。
3. 思考：如果不手动写入 `localStorage`，刷新后值为什么会丢失？

### 12.6 课堂自测

1. Pinia 与 Vuex 的主要区别是什么？
2. 选项式 `defineStore` 与组合式 `defineStore` 中，`state` / `getters` / `actions` 分别对应什么？
3. 为什么推荐通过 `actions` 修改状态？
4. 直接解构 store 为什么会丢失响应式？如何解决？
5. 路由守卫中为什么可以直接调用 `useUserStore()`？

<details>
<summary>参考答案（先自己思考再查看）</summary>

1. Pinia 去掉了 `mutations`，直接通过 `actions` 修改状态；API 更简洁，并且对 TypeScript 与组合式 API 支持更好。
2. 选项式：`state` 对应数据，`getters` 对应派生计算，`actions` 对应方法。组合式：`ref`/`reactive` 对应数据，`computed` 对应派生计算，普通函数对应方法。
3. 修改入口集中，便于追踪、调试和复用，也便于在 action 中加入校验、日志或异步逻辑。
4. 解构取出的是普通值，与 store 的响应式连接断开。应使用 `storeToRefs` 取出响应式的 state 与 getter。
5. 因为 Pinia 实例已在 `main.js` 中通过 `app.use(store)` 安装，且守卫在应用挂载后才真正执行，此时 `useXxxStore()` 能正确获取实例。

</details>

---

## 第 13 课：核心 store 解读

### 13.1 学习目标

- 说清楚 `user`、`app`、`settings`、`dict`、`lock` 五个 store 的职责。
- 理解登录后用户信息与权限的获取流程，并能画出数据流图。
- 理解不同 store 的存储位置与生命周期差异。
- 能发现代码中的潜在问题，并提出改进建议。

### 13.2 课前准备

- 打开 `src/store/modules/` 下的 `user.js`、`app.js`、`settings.js`、`dict.js`、`lock.js`。
- 在笔记中准备一张表格，列出五个 store 的 ID、state 字段、存储位置。

### 13.3 知识讲解

#### （一）`user`：用户身份与权限

`src/store/modules/user.js` 的 state：

```js
state: () => ({
  token: getToken(),     // 从 Cookie 读取（utils/auth.js）
  id: '',
  name: '',
  nickName: '',
  avatar: '',
  roles: [],
  permissions: []
})
```

主要 actions：

| action | 作用 | 关键步骤 |
| --- | --- | --- |
| `login(userInfo)` | 登录 | 调用登录接口 → `setToken` 保存 Token → 解锁屏幕 |
| `getInfo()` | 获取用户信息与权限 | 请求 `/getInfo` → 写入 `roles`、`permissions`、头像等 → 检查初始密码与过期密码 |
| `logOut()` | 退出登录 | 调用退出接口 → 清空 Token、角色、权限 → `removeToken` |

**`getInfo` 的几个细节**（值得仔细阅读）：

1. **头像处理**：若头像不是完整 URL（`isHttp` 为假），会拼接 `VITE_APP_BASE_API`；为空则使用默认头像。
2. **角色与权限**：当后端返回的 `roles` 非空，写入 `roles` 与 `permissions`；否则只设置默认角色 `ROLE_DEFAULT`。
3. **密码安全提示**：如果 `isDefaultModifyPwd` 为真，提示用户修改初始密码；如果 `isPasswordExpired` 为真，提示密码过期。两者都会跳转到个人中心的重置密码页。
4. **密码复杂度类型**：`res.pwdChrtype` 被写入 `sessionStorage`（键 `pwrChrtype`），供 `passwordRule.js` 生成校验规则（第 29 课）。

这里有一个值得思考的写法：`getInfo` 内部使用 `new Promise` 包裹一个已经返回 Promise 的 `getInfo()` 调用。这是可以简化的地方，在第 39 课的代码质量讨论中会再次提到。

**`logOut` 的执行顺序**：先请求后端退出，成功后才清空本地状态。如果请求失败，本地状态不会被清空，这是一种偏保守的策略。

#### （二）`app`：应用级 UI 状态

```js
state: () => ({
  sidebar: {
    opened: Cookies.get('sidebarStatus') ? !!+Cookies.get('sidebarStatus') : true,
    withoutAnimation: false,
    hide: false
  },
  device: 'desktop',
  size: Cookies.get('size') || 'default'
})
```

| 字段 | 含义 | 存储位置 |
| --- | --- | --- |
| `sidebar.opened` | 侧边栏是否展开 | Cookie `sidebarStatus`（1/0） |
| `sidebar.withoutAnimation` | 切换时是否关闭动画 | 仅内存 |
| `sidebar.hide` | 是否隐藏侧边栏（如某些页面全屏） | 仅内存 |
| `device` | 设备类型（desktop / mobile） | 仅内存 |
| `size` | 组件全局尺寸 | Cookie `size` |

actions：

- `toggleSideBar(withoutAnimation)`：切换展开状态，并同步写入 Cookie。若 `sidebar.hide` 为真则直接返回。
- `closeSideBar({ withoutAnimation })`：强制收起，常用于移动端点击菜单后自动收起。
- `toggleDevice(device)`：记录设备类型，由布局组件根据窗口宽度调用。
- `setSize(size)`：修改组件尺寸并写入 Cookie（与第 11 课的 `size` 配置对应）。
- `toggleSideBarHide(status)`：控制侧边栏的显示/隐藏能力。

**设计要点**：侧边栏状态存在 Cookie 中，所以刷新页面后保持展开或收起，这是一个"用户偏好持久化"的典型例子。

#### （三）`settings`：布局与主题设置

`src/store/modules/settings.js` 是设置类 store 中最复杂的一个。它的初始化逻辑体现了"**默认值 + 用户覆盖**"的模式：

```js
const { sideTheme, showSettings, navType, tagsView, /* ... */ } = defaultSettings   // 来自 src/settings.js
const storageSetting = JSON.parse(localStorage.getItem('layout-setting')) || ''

state: () => ({
  theme: storageSetting.theme || '#409EFF',
  navType: storageSetting.navType === undefined ? navType : storageSetting.navType,
  tagsView: storageSetting.tagsView === undefined ? tagsView : storageSetting.tagsView,
  // ...
  isDark: isDark.value
})
```

优先级：**用户保存的设置（localStorage）> `src/settings.js` 的默认值**。

- 注意判断方式：布尔值与数字用 `=== undefined` 判断，而不是 `||`。原因是 `false` 与 `0` 是合法的用户设置，`||` 会把它们误判为"未设置"。这是一个非常值得记住的细节。
- `changeSetting({ key, value })`：通用修改入口，只允许修改 state 中已存在的键（`hasOwnProperty` 检查）。
- `setTitle(title)`：设置页面标题，并调用 `useDynamicTitle()`（第 19 课会讲）。
- `toggleTheme()`：通过 `@vueuse/core` 的 `useDark` 切换暗黑模式，并用 `nextTick` 调用 `handleThemeStyle`（第 25 课会讲）。

> **与 `Settings` 组件的关系**：`src/layout/components/Settings/index.vue` 中的面板会调用 `changeSetting`，并把结果写入 `localStorage` 的 `layout-setting` 键。清除设置时则调用 `localStorage.removeItem`。

#### （四）`dict`：字典缓存

```js
state: () => ({ dict: new Array() })    // 结构：[{ key: 'sys_user_sex', value: [...] }, ...]
```

actions：

| action | 作用 |
| --- | --- |
| `getDict(key)` | 按 key 查找已缓存的字典数组，找不到返回 `undefined` |
| `setDict(key, value)` | 追加一条字典缓存 |
| `removeDict(key)` | 删除某个字典的缓存 |
| `cleanDict()` | 清空全部字典缓存 |
| `initDict()` | 预留的初始化方法，当前为空实现 |

结合 `src/utils/dict.js` 中的 `useDict`，完整流程是：

```
页面调用 useDict('sys_user_sex')
   │
   ├─ dictStore.getDict('sys_user_sex')  ← 命中缓存？直接返回
   │
   └─ 未命中 → getDicts('sys_user_sex') 请求接口
              → 转换为 { label, value, elTagType, elTagClass }
              → dictStore.setDict(...) 写入缓存
```

**关键点**：`dict` store 只存在于内存中，**刷新页面后缓存会清空**，下次使用时会重新请求。这保证了字典数据不会长期过期，代价是每次刷新都会多一次请求。

#### （五）`lock`：锁屏状态

```js
state: () => ({
  isLock: JSON.parse(localStorage.getItem('screen-lock') || 'false'),
  lockPath: localStorage.getItem('screen-lock-path') || '/index'
})
```

actions：

- `lockScreen(currentPath)`：记录当前路径，并把锁定状态写入 `localStorage`；
- `unlockScreen()`：清除锁定状态，并把 `lockPath` 重置为 `/index`。

锁屏状态被持久化到 `localStorage`，所以刷新页面后仍然处于锁屏状态。`permission.js` 的守卫会检查 `isLock`：锁定时强制跳转到 `/lock` 页面（第 16 课会详细讲解）。

> 注意：`unlockScreen` 在 `user.login` 成功后也会被调用。也就是说，重新登录会自动解除锁屏。

#### （六）五个 store 的对比总表

| store ID | 文件 | 主要内容 | 存储位置 | 刷新后是否保留 |
| --- | --- | --- | --- | --- |
| `user` | `user.js` | Token、用户信息、角色、权限 | Token 在 Cookie，其余在内存 | Token 保留，其余需重新获取 |
| `app` | `app.js` | 侧边栏、设备、尺寸 | Cookie（侧边栏、尺寸） | 保留 |
| `settings` | `settings.js` | 主题、布局、标签页开关 | `localStorage`（`layout-setting`） | 保留 |
| `dict` | `dict.js` | 字典缓存 | 内存 | 不保留 |
| `lock` | `lock.js` | 锁屏状态与路径 | `localStorage` | 保留 |

### 13.4 代码阅读任务

1. 在 `src/store/modules/user.js` 中，找出 `getInfo` 中所有条件分支，并画出流程图。
2. 对比 `app.js` 中的 `sidebar.opened` 初始化与 `settings.js` 中的 `theme` 初始化，列出两者在"读取默认值"上的异同。
3. 在 `src` 中搜索 `useDict(`，统计至少 5 个页面的使用，并判断它们是否都能命中缓存。

### 13.5 动手实践（课后任务）

**任务 A：绘制状态来源图**
- 以"登录后进入首页"为主线，画出 `user`、`permission`、`dict`、`settings`、`lock` 在各阶段的状态变化（可以用表格：阶段 × store）。

**任务 B：验证持久化差异**
1. 修改侧边栏收起状态，刷新页面，观察是否保留（对应 Cookie `sidebarStatus`）。
2. 修改一个布局设置（如开启/关闭标签页），刷新页面，观察 `localStorage` 中 `layout-setting` 的变化。
3. 在 DevTools 的 Application 面板中，记录两者的位置。

**任务 C：验证字典缓存**
1. 打开"用户管理"页面，在 Network 面板中记录 `dict/data/type` 相关请求。
2. 切换到其它页面再切回，观察是否重复请求。
3. 刷新页面后再次进入，观察请求是否重新发出。
4. 解释三次观察的结果。

**任务 D：代码审阅（重点练习）**
阅读 `src/store/modules/dict.js` 的 `getDict`：

```js
if (_key == null && _key == "") {
  return null
}
```

- 思考：条件中的 `&&` 应该是什么？如果改为 `||`，会有什么效果？
- 再看 `setDict`：它直接 `push`，没有检查同名 key 是否已存在。如果同一个字典被两次 `setDict`，会出现什么情况？
- 把你的分析写成一段"代码审阅意见"（不要修改仓库中的原文件，仅在笔记中记录）。

> 提示：`getDict` 中的判断逻辑在 `_key` 为 `null` 时并不会返回，而是会继续往下执行循环；`setDict` 不去重，因此同一 key 可能出现多条记录。这两点都是有益的审阅练习。

### 13.6 课堂自测

1. `user` store 中 `token` 的初始值来自哪里？为什么这样设计？
2. `logOut` 为什么先请求后端，再清空本地状态？
3. `app.sidebar.opened` 的初始值如何决定？
4. `settings` 的初始化为什么用 `=== undefined` 而不用 `||`？
5. `dict` store 的缓存为什么不持久化？这样做的利弊是什么？
6. `lock` store 为什么需要持久化？

<details>
<summary>参考答案（先自己思考再查看）</summary>

1. 来自 `getToken()`，即从 Cookie 中读取（`utils/auth.js`）。这样刷新后 Token 仍然有效，无需重新登录。
2. 只有后端确认退出成功后，才清空本地状态，避免后端会话仍然有效而前端显示已退出的不一致。代价是请求失败时本地状态保持不变。
3. 优先读取 Cookie `sidebarStatus`：若存在，则 `!!+值` 转为布尔；若不存在，默认展开（`true`）。
4. `false` 与 `0` 是合法的用户设置，`||` 会把它们当作"未设置"而错误地使用默认值。
5. 不持久化可以保证字典数据不会长期陈旧，代价是刷新后会多一次请求。如果字典变化频繁或请求很慢，可以考虑持久化并设置过期策略。
6. 锁屏是一种安全状态，用户刷新页面后仍应处于锁屏，否则可能绕过锁屏保护。

</details>

---

## 第 14 课：标签页 store（tagsView）与持久化

### 14.1 学习目标

- 理解 `visitedViews`、`cachedViews`、`iframeViews` 三类数据的职责。
- 掌握常用操作：添加、删除、批量关闭、固定（affix）标签。
- 理解标签页持久化（`tagsViewPersist`）的实现与边界。
- 能说明标签页与 `keep-alive` 缓存之间的协作关系。

### 14.2 课前准备

- 打开 `src/store/modules/tagsView.js`，通读 `addView`、`delView`、`loadPersistedViews`。
- 打开 `src/layout/components/TagsView/index.vue`，观察它如何调用 store。

### 14.3 知识讲解

#### （一）三类数据的职责

| 数据 | 类型 | 含义 | 与谁相关 |
| --- | --- | --- | --- |
| `visitedViews` | 对象数组 | 页面标签栏上显示的所有标签（每个标签包含 path、title、meta 等） | 标签栏 UI |
| `cachedViews` | 字符串数组（组件名） | 需要被 `keep-alive` 缓存的组件名 | `AppMain.vue` 中的 `<keep-alive :include>` |
| `iframeViews` | 对象数组 | 以 iframe 方式打开的外链页面 | `IframeToggle` 组件 |

三者的关系可以用一句话概括：**同一个页面会同时出现在 `visitedViews`（显示标签）和 `cachedViews`（缓存组件）中，除非它设置了 `meta.noCache`。**

```
用户打开 /system/user
   │
   ├─► visitedViews.push({ path: '/system/user', title: '用户管理', ... })   → 标签栏出现"用户管理"
   └─► cachedViews.push('User')                                           → keep-alive 缓存 User 组件
```

#### （二）核心 actions

| action | 作用 | 对应场景 |
| --- | --- | --- |
| `addView(view)` | 同时添加到 visited 与 cached | 打开新页面 |
| `addVisitedView(view)` | 只添加标签（去重） | 恢复持久化标签 |
| `addCachedView(view)` | 只添加缓存（去重，遵守 `noCache`） | 打开新页面 |
| `addAffixView(view)` | 添加固定标签，插入到最前面 | 首页等固定标签 |
| `addIframeView(view)` | 添加 iframe 页面（去重） | 打开外链 |
| `delView(view)` | 删除某个页面的标签与缓存 | 关闭标签 |
| `delOthersViews(view)` | 关闭除当前页以外的所有标签 | 右键菜单"关闭其它" |
| `delAllViews(view)` | 关闭全部标签（保留固定标签） | 右键菜单"关闭全部" |
| `delLeftTags` / `delRightTags` | 关闭左侧或右侧标签 | 右键菜单 |
| `updateVisitedView(view)` | 更新某个标签的标题等信息 | 标题变化 |

这些方法的命名很有规律：`add*` 添加，`del*` 删除，`*Views` 与 `*CachedViews` 分别对应标签与缓存。阅读时可以按这个规律快速定位。

**`addCachedView` 的关键判断**（来自代码）：

```js
addCachedView(view) {
  if (this.cachedViews.includes(view.name)) return   // 已缓存则跳过
  if (!view.meta.noCache) {
    this.cachedViews.push(view.name)                 // 只有非 noCache 页面才缓存
  }
}
```

这里以 `view.name` 作为缓存标识，与第 9 课的组件 `name` 对应。路由配置中的 `name` 字段必须与组件的 `name` 保持一致，否则缓存无法生效。

#### （三）标签的持久化

持久化由设置项 `tagsViewPersist` 控制。核心代码：

```js
const PERSIST_KEY = 'tags-view-visited'

function saveVisitedViews(views) {
  if (!isPersistEnabled()) return                    // 未开启则不保存
  const toSave = views.filter(v => !(v.meta && v.meta.affix))   // 固定标签不保存（由路由自动添加）
    .map(v => ({ path, fullPath, name, title, query, ... }))
  cache.local.setJSON(PERSIST_KEY, toSave)           // 写入 localStorage
}

function loadVisitedViews() {
  return cache.local.getJSON(PERSIST_KEY) || []
}
```

- **保存时机**：每次 `addVisitedView` 成功添加后调用 `saveVisitedViews`。
- **恢复时机**：标签栏组件（`TagsView/index.vue`）在初始化时调用 `loadPersistedViews()`，它遍历持久化的数据并逐个调用 `addVisitedView`。
- **固定标签不持久化**：因为固定标签由路由的 `meta.affix` 决定，每次都会自动添加，无需保存。

#### （四）一个需要注意的边界

从代码可以看出，`loadPersistedViews` 只调用了 `addVisitedView`，**没有调用 `addCachedView`**。这意味着：

- 刷新后，标签栏会恢复，但对应页面的组件**并未加入缓存列表**；
- 当用户点击某个恢复的标签时，页面会重新创建组件，此时会通过路由的正常流程调用 `addView`，缓存才会建立。

这是一个值得讨论的设计取舍：恢复标签不意味着恢复组件状态。阅读时可以思考：如果希望刷新后的标签也能保留表单输入，应该如何修改？

#### （五）标签页与 keep-alive 的协作

```
TagsView 组件（标签栏 UI）
   │  点击标签 → router.push
   ▼
路由切换 → 守卫 → 页面组件
   │
   ▼
AppMain.vue
   <keep-alive :include="tagsViewStore.cachedViews">
     <component :is="Component" :key="route.path" />
   </keep-alive>
```

- `visitedViews` 决定**用户看到哪些标签**；
- `cachedViews` 决定**哪些组件保留状态**；
- 关闭标签时（`delView`），两者同时被清理，组件实例随之被销毁。

#### （六）`plugins/tab.js`：对 store 的封装

`src/plugins/tab.js` 提供了 `$tab` 工具（第 19 课会详细讲）。它把 `router.push` 与 `tagsViewStore` 的操作封装成统一的 API，例如 `openPage`、`closePage`、`refreshPage`。页面中通常调用 `$tab` 而不是直接操作 store，这样更不容易遗漏"导航 + 标签"的组合逻辑。

### 14.4 代码阅读任务

1. 在 `tagsView.js` 中，列出所有会同时修改 `visitedViews` 与 `cachedViews` 的 action，并说明它们为什么要同时修改。
2. 阅读 `saveVisitedViews` 中的 `map` 字段，列出被保存的字段，并思考为什么不保存 `component` 等不可序列化的内容。
3. 在 `src/layout/components/TagsView/index.vue` 中找到 `loadPersistedViews` 的调用位置，记录它在什么生命周期中执行。

### 14.5 动手实践（课后任务）

**任务 A：观察标签与缓存的同步**
1. 依次打开 3 个页面（如首页、用户管理、角色管理）。
2. 在 DevTools 中用 Vue DevTools 查看 `tags-view` store 的 `visitedViews` 与 `cachedViews`。
3. 关闭其中一个标签，观察两个数组的变化。
4. 记录结论：哪些变化是同步的？

**任务 B：验证 `noCache`**
1. 找一个路由配置，临时在 `meta` 中添加 `noCache: true`（例如某个编辑页）。
2. 打开它，观察 `cachedViews` 中是否出现该组件名。
3. 实验后恢复配置。

**任务 C：验证标签持久化**
1. 在设置中开启标签页持久化（`tagsViewPersist`）。打开若干页面后刷新。
2. 观察标签栏是否恢复，并在 `localStorage` 的 `tags-view-visited` 中查看内容。
3. 在 `cachedViews` 中检查恢复的标签是否也在缓存中（预期：不在）。
4. 记录你的观察，并解释原因。

**任务 D：批量关闭**
- 对比 `delOthersViews` 与 `delAllViews` 的代码，列出它们对固定标签（affix）的处理差异。

### 14.6 课堂自测

1. `visitedViews`、`cachedViews`、`iframeViews` 各自的职责是什么？
2. 为什么 `addCachedView` 以 `view.name` 作为标识？这与组件的 `name` 有什么关系？
3. `meta.noCache` 为 true 的页面，会在哪个环节被排除？会产生什么效果？
4. 标签持久化保存了哪些内容？为什么固定标签不需要保存？
5. 刷新后恢复的标签，其组件状态能恢复吗？为什么？

<details>
<summary>参考答案（先自己思考再查看）</summary>

1. `visitedViews` 决定标签栏显示哪些标签；`cachedViews` 决定哪些组件被 `keep-alive` 缓存；`iframeViews` 记录以 iframe 方式打开的外链页面。
2. `keep-alive` 的 `include` 通过组件名匹配，因此缓存标识使用 `name`。路由配置的 `name` 应与组件的 `name` 一致，否则缓存不生效。
3. 在 `addCachedView` 中，`noCache` 为 true 时不会加入 `cachedViews`。效果是该页面每次进入都是新实例，状态不保留。
4. 保存 `path`、`fullPath`、`name`、`title`、`query` 等可序列化的字段。固定标签由路由 `meta.affix` 决定，每次启动都会自动添加，保存反而会造成重复。
5. 不能。`loadPersistedViews` 只恢复标签（`visitedViews`），没有恢复缓存（`cachedViews`），组件需要重新创建，表单输入等状态会丢失。

</details>

---

## 本章小结

| 课时 | 核心问题 | 关键结论 |
| --- | --- | --- |
| 第 12 课 | Pinia 怎么用？ | 选项式与组合式两种写法；`state`/`getters`/`actions` 分工；`storeToRefs` 保持解构响应式 |
| 第 13 课 | 核心状态有哪些？ | `user` 管身份与权限，`app` 管 UI 偏好，`settings` 管布局主题，`dict` 管字典缓存，`lock` 管锁屏；存储位置各不相同 |
| 第 14 课 | 标签页如何管理？ | `visitedViews` 管标签，`cachedViews` 管缓存，二者协作；持久化只恢复标签不恢复组件状态 |

**本章的一个核心观念**：store 中的每一个字段都应当回答三个问题——**它从哪里来、存在哪里、什么时候失效**。搞清楚这三点，就能判断一个状态是否应该持久化、何时需要刷新，也能快速定位"数据不对"之类的问题。

**下一章预告**：第 4 章《路由与权限》，我们将从 Vue Router 的配置开始，逐步进入守卫、动态路由生成和按钮权限控制。

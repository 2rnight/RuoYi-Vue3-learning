# 第 1 章 工程化基础：Vite

> 本章共 4 课时（第 3–6 课），目标是：**看懂项目"从 HTML 到页面"的启动链路，掌握环境变量与 Vite 配置，理解插件体系，并能完成一次生产构建与部署相关的检查。**
> 对应大纲：`docs/课程大纲.md` 第 1 章。
> 前置要求：已完成第 0 章，前端依赖已安装，后端可在 `localhost:8080` 运行（阅读和构建不强依赖后端）。

---

## 本章学习地图

```
第 3 课 启动流程与环境变量 ─► 第 4 课 Vite 配置详解 ─► 第 5 课 Vite 插件体系 ─► 第 6 课 构建脚本、部署与兼容
（页面怎么跑起来）             （开发与构建怎么配）       （能力从哪里来）             （产物怎么交付）
```

本章完成后，你应当能够：

1. 画出 `index.html` → `src/main.js` → 应用挂载的启动流程图。
2. 说明 `.env.*` 文件的加载规则，知道 `VITE_` 前缀的含义，并能正确读取环境变量。
3. 解释 `vite.config.js` 中别名、代理、构建输出、CSS 处理等配置的作用。
4. 说出项目中每个 Vite 插件解决的问题，并能验证其效果。
5. 执行生产构建，读懂 `dist/` 产物结构，并理解 `base`、压缩和部署代理的关系。

---

## 第 3 课：应用启动流程与环境变量

### 3.1 学习目标

- 描述从浏览器请求 `index.html` 到页面渲染完成的完整过程。
- 理解 `src/main.js` 中各行代码的执行顺序与意义。
- 掌握 `.env` 文件的命名、加载优先级与 `VITE_` 前缀规则。
- 能在业务代码中正确使用 `import.meta.env`。

### 3.2 课前准备

- 打开以下文件：`index.html`、`src/main.js`、`.env.development`、`.env.production`、`.env.staging`。
- 在本地执行一次 `yarn dev`，确保页面可以打开（登录页即可）。

### 3.3 知识讲解

#### （一）入口 HTML：`index.html`

`index.html` 是单页应用（SPA）的唯一 HTML 文件，它的关键内容有三处：

```html
<title>%VITE_APP_TITLE%</title>                          <!-- ① 标题占位符 -->
<!--[if lt IE 11]><script>window.location.href='/html/ie.html';</script><![endif]-->  <!-- ② 旧版 IE 跳转 -->
<div id="app">                                            <!-- ③ 挂载点 -->
  <div id="loader-wrapper">...正在加载系统资源...</div>  <!--    加载动画，应用挂载后被替换 -->
</div>
<script type="module" src="/src/main.js"></script>        <!-- ④ 应用入口脚本 -->
```

要点：

- **`%VITE_APP_TITLE%`**：Vite 会在构建或启动时，把它替换为对应环境变量的值。我们在构建产物中验证过，最终标题是「若依管理系统」。
- **`<script type="module">`**：声明为 ES 模块，浏览器会按模块规则加载，这是 Vite 能"按需编译"的基础。
- **`#app`**：Vue 应用挂载的容器。挂载前显示的 `#loader-wrapper` 加载动画，会在应用挂载后被 Vue 接管的 DOM 替换掉。

#### （二）入口脚本：`src/main.js` 的执行顺序

`src/main.js` 的代码可以分为 6 个阶段。阅读时请按阶段理解，而不是逐行死记：

| 阶段 | 代码（节选） | 作用 |
| --- | --- | --- |
| ① 基础依赖 | `import { createApp } from 'vue'`、`Cookies`、`ElementPlus`、`zh-cn` 语言包 | 引入框架与 UI 库 |
| ② 全局样式 | `import '@/assets/styles/index.scss'` | 注入全局样式 |
| ③ 核心模块 | `App`、`store`、`router`、`directive` | 引入根组件、状态、路由、指令 |
| ④ 插件与图标 | `plugins`、`virtual:svg-icons-register`、`SvgIcon`、`elementIcons` | 注册插件与 SVG 图标 |
| ⑤ 路由守卫 | `import './permission'` | 导入即注册 `beforeEach` 守卫 |
| ⑥ 创建并挂载 | `createApp(App)` → 注册全局属性与组件 → `app.use(...)` → `app.mount('#app')` | 真正启动应用 |

这里有两个值得注意的细节：

1. **`import './permission'` 的副作用**：`permission.js` 中没有导出任何东西，它只是在被导入时执行 `router.beforeEach(...)`。所以"导入即生效"，这是一种常见的副作用模块写法。
2. **`app.use` 的顺序**：`app.use(router)` 与 `app.use(store)` 需要在 `app.mount` 之前完成。挂载后再注册插件，会导致插件内的组件或指令无法在首屏生效。

用一张图概括：

```
index.html
   └─► src/main.js
         ├─ import 各模块（执行副作用：permission.js 注册守卫）
         ├─ createApp(App)
         ├─ app.config.globalProperties.xxx = ...   （全局方法）
         ├─ app.component('DictTag', ...)           （全局组件）
         ├─ app.use(router) / app.use(store) / app.use(plugins)
         ├─ app.use(ElementPlus, { locale, size })
         └─ app.mount('#app')  ──► 渲染 App.vue ──► 路由匹配 ──► 布局与页面
```

#### （三）环境变量：`.env` 文件体系

Vite 会根据当前 **模式（mode）** 加载对应的 `.env` 文件。

| 命令 | 模式（mode） | 加载的文件（按优先级从高到低） |
| --- | --- | --- |
| `yarn dev` | `development` | `.env.development.local` > `.env.development` > `.env.local` > `.env` |
| `yarn build:prod` | `production` | `.env.production.local` > `.env.production` > `.env.local` > `.env` |
| `yarn build:stage` | `staging` | `.env.staging.local` > `.env.staging` > `.env.local` > `.env` |

> 注意：我们的仓库中没有 `.env`，但 `.gitignore` 中包含 `*.local`，所以 `.env.local` 之类的文件**不会被提交**，适合放个人本地配置。

项目中三个环境文件的核心差异：

| 变量 | `.env.development` | `.env.staging` | `.env.production` |
| --- | --- | --- | --- |
| `VITE_APP_TITLE` | 若依管理系统 | 若依管理系统 | 若依管理系统 |
| `VITE_APP_ENV` | `'development'` | `'staging'` | `'production'` |
| `VITE_APP_BASE_API` | `/dev-api` | `/stage-api` | `/prod-api` |
| `VITE_BUILD_COMPRESS` | 未配置 | `gzip` | `gzip` |

#### （四）`VITE_` 前缀规则与 `import.meta.env`

- **只有以 `VITE_` 开头的变量才会暴露给前端代码**。例如 `DB_PASSWORD=xxx` 即使写在 `.env` 中，也不会进入浏览器代码。这是一道重要的安全边界。
- 在业务代码中通过 `import.meta.env` 读取：

```js
// src/settings.js
title: import.meta.env.VITE_APP_TITLE

// src/utils/request.js
baseURL: import.meta.env.VITE_APP_BASE_API
```

- 这些值在**构建时被静态替换**。也就是说，`production` 构建的 JS 文件里，`/prod-api` 已经被写死在代码中，运行时修改服务器上的 `.env` 文件是不会生效的。

> **验证方法**：我们在生产构建产物中搜索到了 `/prod-api` 字符串，在测试环境构建产物中搜索到了 `/stage-api`，证明了替换发生在构建期。

#### （五）内置变量

Vite 还提供几个内置变量，项目中也可能用到：

| 变量 | 含义 |
| --- | --- |
| `import.meta.env.MODE` | 当前模式，如 `development` |
| `import.meta.env.DEV` | 是否为开发环境（布尔值） |
| `import.meta.env.PROD` | 是否为生产环境（布尔值） |
| `import.meta.env.BASE_URL` | 配置的 `base` 路径 |

### 3.4 代码阅读任务

1. 在 `index.html` 中找出 `%VITE_APP_TITLE%` 占位符，并在 `src/settings.js` 中找到它的读取位置。
2. 打开 `src/main.js`，按第 3.3 节的六个阶段，用不同颜色在纸上标注每一段代码。
3. 在 `src` 中搜索 `import.meta.env`，列出所有使用的变量，并填写下表：

| 变量 | 使用文件 | 用途 |
| --- | --- | --- |
| | | |

### 3.5 动手实践（课后任务）

**任务 A：启动流程图**
- 绘制一张从"浏览器请求 `/`"到"页面可见"的时序图，参与者包括：浏览器、`index.html`、`main.js`、`permission.js`、Vue 应用、路由、后端。

**任务 B：自定义环境变量**
1. 在 `.env.development` 末尾添加：`VITE_APP_DEMO_TEXT = 第一章实验`。
2. 在 `src/views/index.vue` 中临时加入 `console.log(import.meta.env.VITE_APP_DEMO_TEXT)`。
3. 执行 `yarn dev`，打开首页控制台查看输出。
4. 把 `VITE_APP_DEMO_TEXT` 改为 `VITE_DEMO_TEXT` 的反例（去掉前缀），再次观察 `undefined`，理解前缀规则。
5. 实验结束后**删除实验代码与变量**，恢复仓库。

**任务 C：验证构建期替换**
1. 执行 `yarn build:prod`。
2. 在 `dist/static/js/` 中搜索 `/prod-api`，记录命中的文件名。
3. 再执行 `yarn build:stage`，搜索 `/stage-api`，对比两次结果。
4. 思考：如果部署后想把接口前缀从 `/prod-api` 改成 `/api`，需要做什么？（答案见第 6 课。）

### 3.6 课堂自测

1. `yarn build:stage` 会加载哪些 `.env` 文件？它们的优先级如何？
2. 为什么 `DB_PASSWORD` 这样的变量即使写进 `.env` 也不会暴露给浏览器？
3. `app.use(router)` 为什么必须在 `app.mount('#app')` 之前执行？
4. `import './permission'` 没有任何导出，它是如何生效的？
5. 环境变量为什么在构建后就"固定"了？这对部署流程有什么启示？

<details>
<summary>参考答案（先自己思考再查看）</summary>

1. 加载 `.env.staging.local`、`.env.staging`、`.env.local`、`.env`，优先级从高到低，靠前的覆盖靠后的同名变量。
2. 因为 Vite 只把 `VITE_` 前缀的变量注入到前端代码中，其它变量只在 Node 侧（如 `vite.config.js`）可用，不会打包进浏览器产物。
3. 挂载后插件才注册，已经渲染的组件无法使用路由或 store 等能力，首屏可能报错或行为异常。`app.mount` 的语义就是"开始渲染"，之前应完成所有配置。
4. 模块被导入时会立即执行其顶层代码，因此 `router.beforeEach(...)` 在导入时就被注册。这是一种"副作用导入"。
5. 因为 `import.meta.env.X` 在构建时被替换为字面量。部署时要选对构建模式，修改配置需要重新构建，而不是只改服务器上的文件。

</details>

---

## 第 4 课：Vite 配置详解

### 4.1 学习目标

- 读懂 `vite.config.js` 的整体结构：`defineConfig`、`loadEnv`、插件、别名、服务器、构建、CSS。
- 理解路径别名 `@`、`~` 的作用，并能在代码中正确使用。
- 理解开发代理的工作原理，能独立修改代理目标。
- 理解构建输出的文件命名规则与 `sourcemap` 的开发/构建差异。

### 4.2 课前准备

- 打开 `vite.config.js`，通读一遍，标出你不认识的配置项。
- 参考 Vite 官方文档的 `server.proxy` 与 `build` 章节（中文文档：<https://cn.vitejs.dev>）。

### 4.3 知识讲解

#### （一）配置文件的整体结构

```js
export default defineConfig(({ mode, command }) => {
  const env = loadEnv(mode, process.cwd())   // ① 读取当前模式的环境变量
  const { VITE_APP_ENV } = env
  return {
    base: '/',                               // ② 部署基础路径
    plugins: createVitePlugins(env, command === 'build'), // ③ 插件
    resolve: { alias: {...}, extensions: [...] },         // ④ 模块解析
    build: { sourcemap, outDir, assetsDir, chunkSizeWarningLimit, rollupOptions },  // ⑤ 构建
    server: { port: 80, host: true, open: true, proxy: {...} },                    // ⑥ 开发服务器
    css: { postcss: {...} }                                                        // ⑦ 样式处理
  }
})
```

关键点：

- **函数式配置**：`defineConfig` 接收一个函数，函数参数中的 `mode`（模式）与 `command`（`serve` 或 `build`）让配置可以根据场景变化。这正是 `sourcemap`、插件等配置能区分开发与构建的原因。
- **`loadEnv(mode, process.cwd())`**：在配置文件中读取环境变量。注意，这里读取的是 Node 侧环境，与前端代码中的 `import.meta.env` 是两条不同的路径。

#### （二）`base`：部署基础路径

```js
base: VITE_APP_ENV === 'production' ? '/' : '/',
```

- `base` 决定了打包产物中资源的 URL 前缀。例如 `base: '/admin/'` 时，资源会变成 `/admin/static/js/xxx.js`。
- 项目中三元表达式的两个分支都是 `/`，所以当前**没有任何效果**。这是一个很好的阅读练习：它说明作者预留了按环境切换子路径的入口。
- 如果将来部署到子路径，需要同时修改 `base` 和 Web 服务器的路由配置。

#### （三）`resolve`：路径别名与扩展名

```js
resolve: {
  alias: {
    '~': path.resolve(__dirname, './'),        // 项目根目录
    '@': path.resolve(__dirname, './src')      // src 目录
  },
  extensions: ['.mjs', '.js', '.ts', '.jsx', '.tsx', '.json', '.vue']
}
```

- `@/utils/request` 等价于 `./src/utils/request`，是项目中最常用的导入写法。
- `~` 指向项目根目录，可用于引用根目录下的文件。
- `extensions` 决定导入时可以省略哪些后缀。例如 `import App from './App'` 能找到 `App.vue`，正是因为 `.vue` 在列表中。

> **常见坑**：别名必须在 `vite.config.js` 中配置，但**如果你使用 IDE 的智能提示或 ESLint，还需要在 `jsconfig.json` 中配置 `paths`**，否则编辑器可能报"找不到模块"。本仓库未包含这些配置文件，这是可以思考的问题。

#### （四）`server`：开发服务器与代理

```js
server: {
  port: 80,        // 端口
  host: true,      // 监听 0.0.0.0，局域网其它设备也能访问
  open: true,      // 启动后自动打开浏览器
  proxy: {
    '/dev-api': {
      target: 'http://localhost:8080',
      changeOrigin: true,
      rewrite: (p) => p.replace(/^\/dev-api/, '')
    },
    '^/v3/api-docs/(.*)': {
      target: 'http://localhost:8080',
      changeOrigin: true
    }
  }
}
```

代理工作原理（以登录接口为例）：

```
浏览器  ──► http://localhost/dev-api/login
             │  （同源，无跨域）
             ▼
Vite 开发服务器（:80）匹配到 /dev-api 规则
             │  rewrite：去掉 /dev-api 前缀
             ▼
后端  ◄──  http://localhost:8080/login
```

- `changeOrigin: true`：让转发请求的 `Host` 头变为目标地址，许多后端框架依赖它做虚拟主机判断。
- 第二条规则 `^/v3/api-docs/(.*)` 用于代理 SpringDoc 生成的接口文档 JSON，没有 `rewrite`，路径原样转发。

> **思考**：`host: true` 与 `port: 80` 组合时，如果 80 端口被占用，启动会失败。Linux/macOS 下绑定 1024 以下端口通常需要管理员权限，这也是很多人首次运行会遇到的问题。

#### （五）`build`：构建输出

```js
build: {
  sourcemap: command === 'build' ? false : 'inline',
  outDir: 'dist',
  assetsDir: 'assets',
  chunkSizeWarningLimit: 2000,
  rollupOptions: {
    output: {
      chunkFileNames: 'static/js/[name]-[hash].js',
      entryFileNames: 'static/js/[name]-[hash].js',
      assetFileNames: 'static/[ext]/[name]-[hash].[ext]'
    }
  }
}
```

| 配置 | 含义 | 项目取值的原因 |
| --- | --- | --- |
| `sourcemap` | 是否生成 Source Map | 开发时 `inline` 便于调试；生产构建关闭，避免暴露源码 |
| `outDir` | 输出目录 | 默认 `dist`，与 `.gitignore` 对应，产物不入库 |
| `chunkSizeWarningLimit` | 单个 chunk 体积告警阈值（KB） | 设为 2000，因为若依的大依赖（如 Element Plus、ECharts）会超过默认的 500 |
| `chunkFileNames` / `entryFileNames` | JS 文件命名规则 | 统一放到 `static/js/`，文件名带 `[hash]`，便于长期缓存 |
| `assetFileNames` | 静态资源命名规则 | 按扩展名分目录，如 `static/css/`、`static/png/` |

**`[hash]` 的意义**：文件名由内容计算得出。内容不变，文件名不变，浏览器可以长期缓存；内容变化，文件名随之变化，避免用户读到旧文件。

我们在第 6 课会看到实际的产物文件名。

#### （六）`css.postcss`：自定义 PostCSS 插件

```js
css: {
  postcss: {
    plugins: [{
      postcssPlugin: 'internal:charset-removal',
      AtRule: {
        charset: (atRule) => { if (atRule.name === 'charset') atRule.remove() }
      }
    }]
  }
}
```

- PostCSS 是处理 CSS 的工具链。这里编写了一个**自定义插件**，删除 CSS 文件中的 `@charset` 规则。
- 为什么要删？当多个 CSS 文件合并时，中间出现的 `@charset` 声明可能导致样式解析异常（它只在文件开头合法）。这是典型的"工程兼容性修补"。

> 阅读建议：这段代码展示了 PostCSS 插件的最小结构，即一个带 `postcssPlugin` 名字的对象，并通过 `AtRule`、`Rule`、`Declaration` 等访问器处理节点。

### 4.4 代码阅读任务

1. 找出 `vite.config.js` 中所有 **依赖 `command` 或 `mode` 的配置**，并说明各自在开发与构建时的取值。
2. 画出一次 `/dev-api/getInfo` 请求从浏览器到后端的完整路径（包括 URL 变化）。
3. 找出 `baseUrl` 常量被使用的位置，说明为什么要把它抽成常量。

### 4.5 动手实践（课后任务）

**任务 A：别名实验**
1. 在任意页面中新增一行 `import { parseTime } from '~/src/utils/ruoyi'`，观察是否可以正常工作。
2. 对比 `@/utils/ruoyi` 写法，说明两者的区别与各自适用场景。
3. 恢复代码。

**任务 B：代理实验**
1. 把 `vite.config.js` 中 `/dev-api` 的 `rewrite` 整行注释掉，重启 `yarn dev`。
2. 打开登录页，观察 Network 面板中 `login` 请求的 URL 与状态码变化，并解释原因（后端会收到带 `/dev-api` 前缀的路径）。
3. 恢复配置，并记录你的观察结论。

**任务 C：构建输出实验**
1. 执行 `yarn build:prod`，在 `dist/static/js/` 中找出入口文件，记录其文件名。
2. 修改 `chunkFileNames` 为 `static/js/[name].js`（去掉 hash），重新构建，比较文件名的变化。
3. 思考：去掉 hash 后，为什么部署后用户可能读到旧版本？
4. 恢复配置并重新构建。

**任务 D：`base` 实验（仅阅读与推演，不必真正部署）**
1. 假设把 `base` 改为 `/admin/`，推演以下问题的答案：
   - `dist/index.html` 中 `<script src="...">` 的路径会变成什么？
   - 访问 `http://域名/admin/` 时，Web 服务器需要做什么配置？
2. 将推演结果写入笔记，不要修改仓库。

### 4.6 课堂自测

1. `defineConfig` 接收函数而不是对象的好处是什么？
2. `@` 别名在哪里配置？如果编辑器报"找不到模块"，应该检查什么？
3. 代理中 `changeOrigin` 的作用是什么？
4. 为什么生产构建要关闭 `sourcemap`？
5. `[hash]` 文件名如何同时兼顾缓存与更新？

<details>
<summary>参考答案（先自己思考再查看）</summary>

1. 配置可以根据 `mode` 和 `command` 动态生成，例如开发时开启 inline sourcemap、构建时关闭，而不需要维护多份配置文件。
2. 在 `vite.config.js` 的 `resolve.alias` 中配置。编辑器需要在 `jsconfig.json` 或 `tsconfig.json` 中配置 `paths`，才能识别别名。
3. 把请求头 `Host` 改写为目标服务器的地址，避免后端因域名不匹配而拒绝或错误路由。
4. Source Map 会把压缩后的代码映射回源码，生产环境开启会暴露源码结构和业务逻辑，并增加体积。
5. 文件名由内容哈希决定：内容不变则文件名不变，可长期缓存；内容变化则文件名变化，浏览器必然请求新文件。

</details>

---

## 第 5 课：Vite 插件体系

### 5.1 学习目标

- 理解 `vite/plugins/index.js` 如何组装插件，以及"开发/构建"差异化插件的思路。
- 说出 4 个自定义插件（自动导入、setup 扩展、SVG 图标、压缩）各自解决的问题。
- 能通过实际修改验证插件的效果。

### 5.2 课前准备

- 打开 `vite/plugins/` 目录，依次阅读 `index.js` 与 5 个插件文件。
- 了解 Vite 插件的基本概念：插件是一个返回配置对象的函数，可以在构建的不同阶段介入。

### 5.3 知识讲解

#### （一）插件的组装：`vite/plugins/index.js`

```js
export default function createVitePlugins(viteEnv, isBuild = false) {
  const vitePlugins = [vue()]                       // 1. Vue 单文件组件支持（必需）
  vitePlugins.push(createAutoImport())              // 2. API 自动导入
  vitePlugins.push(createSetupExtend())             // 3. <script setup name="..."> 扩展
  vitePlugins.push(createSvgIcon(isBuild))          // 4. SVG 图标（构建时启用 SVGO 优化）
  isBuild && vitePlugins.push(...createCompression(viteEnv))  // 5. 仅构建时：压缩
  return vitePlugins
}
```

这种组织方式的优点：

- **职责清晰**：每个插件封装在独立文件中，`vite.config.js` 只需一行调用。
- **差异化配置**：通过 `isBuild` 参数决定哪些插件只在构建时生效。例如压缩只在构建时需要，开发时跳过可以加快启动。
- **可替换性**：想移除某个能力，只需注释一行。

#### （二）`@vitejs/plugin-vue`：Vue 单文件组件

`vue()` 负责把 `.vue` 文件（模板、脚本、样式三合一）编译为浏览器可执行的 JavaScript。它是本项目的**必需插件**，没有它，所有页面都无法加载。

#### （三）自动导入：`vite/plugins/auto-import.js`

```js
autoImport({
  imports: [
    'vue',
    'vue-router',
    'pinia',
    { '@/utils/dict': ['useDict'], '@/utils/ruoyi': ['selectDictLabel'] }
  ],
  dts: false
})
```

作用：**无需手动 `import` 就能在代码中使用 `ref`、`computed`、`onMounted`、`useRouter`、`defineStore` 等 API**。

我们在仓库中可以验证：

- `src/views/system/user/index.vue` 中使用了 `ref`，但文件顶部的 `import` 语句中并没有从 `vue` 导入它。
- `src/views/monitor/job/index.vue` 中直接调用了 `useDict('sys_job_group')`，没有 `import`。

**两个 `{ '@/utils/dict': [...] }` 条目**是自定义的自动导入：项目中高频使用的 `useDict` 与 `selectDictLabel` 也被纳入了自动导入。

> `dts: false` 表示不生成 `auto-imports.d.ts` 类型声明文件。代价是编辑器对自动导入的符号可能缺少类型提示；如果你使用 TypeScript 版本，就需要把它打开。

**取舍思考**：
- 优点：减少样板代码，页面更简洁。
- 缺点：读代码时不容易一眼看出某个函数的来源；新手可能不知道 `ref` 从哪来。
- 建议：阅读时用编辑器的"跳转到定义"辅助理解。

#### （四）`setup-extend`：给 `<script setup>` 加上 `name`

```js
import setupExtend from 'unplugin-vue-setup-extend-plus/vite'
export default function createSetupExtend() {
  return setupExtend({})
}
```

在项目中可以看到这样的写法：

```vue
<script setup name="Index">        <!-- src/views/index.vue -->
<script setup name="Cache">        <!-- src/views/monitor/cache/index.vue -->
<script setup name="CacheList">    <!-- src/views/monitor/cache/list.vue -->
```

- 标准的 `<script setup>` 没有 `name` 选项，但 `<keep-alive>` 缓存需要组件名来匹配 `include`。
- 这个插件让 `<script setup>` 也能声明 `name`，从而实现按组件名的缓存控制。
- 与路由 `meta.noCache` 等配置配合，可以精细控制页面是否缓存（第 9 课会深入）。

#### （五）`vite-plugin-svg-icons`：SVG 图标系统

```js
createSvgIconsPlugin({
  iconDirs: [path.resolve(process.cwd(), 'src/assets/icons/svg')],
  symbolId: 'icon-[dir]-[name]',
  svgoOptions: isBuild
})
```

工作机制：

1. 扫描 `src/assets/icons/svg/` 下所有 SVG 文件（如 `user.svg`）。
2. 在构建时把它们合并进一个 SVG 雪碧图（`<symbol>` 集合）。
3. 在 `src/main.js` 中通过 `import 'virtual:svg-icons-register'` 注册。这是一个**虚拟模块**，由插件在内存中生成，不对应真实文件。
4. 业务中用 `<svg-icon icon-class="user" />` 引用。`SvgIcon` 组件内部会生成 `#icon-${iconClass}` 形式的引用（见 `src/components/SvgIcon/index.vue`）。对于直接位于 `src/assets/icons/svg/` 下的 `user.svg`，构建产物中的符号 ID 为 `icon-user`（我们在构建产物中验证过 `id="icon-user"`）。

参数说明：

- `symbolId: 'icon-[dir]-[name]'`：定义符号 ID 的生成规则。
- `svgoOptions: isBuild`：构建时开启 SVGO 优化（压缩 SVG 代码），开发时关闭以提升启动速度与调试便利。

> 图标组件 `SvgIcon` 位于 `src/components/SvgIcon/index.vue`，与 `svgicon.js` 配合使用，`svgicon.js` 还负责注册 Element Plus 图标。

#### （六）`vite-plugin-compression`：预压缩

```js
export default function createCompression(env) {
  const { VITE_BUILD_COMPRESS } = env
  // 当 VITE_BUILD_COMPRESS 包含 gzip → 生成 .gz 文件
  // 当包含 brotli → 生成 .br 文件
}
```

- **预压缩**：在构建时就生成 `.gz` 或 `.br` 文件，而不是在服务器运行时压缩。Web 服务器（如 Nginx）可直接返回压缩文件，节省 CPU。
- 开关由环境变量 `VITE_BUILD_COMPRESS` 控制：`.env.staging` 与 `.env.production` 中设为 `gzip`，`.env.development` 中未配置，所以开发构建不压缩。
- `deleteOriginFile: false` 保留原始文件，因为并非所有客户端都支持压缩。

> 这与第 6 课的部署内容直接相关：Nginx 需要配置 `gzip_static on;` 才会优先返回 `.gz` 文件。

### 5.4 代码阅读任务

1. 列出 `createVitePlugins` 中所有插件，并标注哪些只在构建时启用。
2. 在 `src` 中搜索 `virtual:svg-icons-register` 与 `icon-class=`，各找出一个使用位置。
3. 在 `src/views/monitor/job/index.vue` 中，找出一个未被显式导入但被使用的 API，并在 `vite/plugins/auto-import.js` 中确认它来源于自动导入列表。

### 5.5 动手实践（课后任务）

**任务 A：验证自动导入**
1. 在 `src/views/index.vue` 中新建一个 `ref`，不要手动导入。
2. 运行 `yarn dev`，确认页面正常。
3. 将 `auto-import.js` 中 `'vue'` 一项临时注释，再次运行，观察报错信息（预期：`ref is not defined`）。
4. 恢复配置，删除实验代码。

**任务 B：验证 SVG 图标**
1. 在 `src/assets/icons/svg/` 中复制一个已有图标，例如 `user.svg`，命名为 `demo-user.svg`。
2. 在某个页面中用 `<svg-icon icon-class="demo-user" />` 显示它。
3. 观察页面中是否显示图标；在开发者工具中检查 DOM，观察 `<use>` 元素引用的符号 ID。
4. 实验结束后删除 `demo-user.svg` 和页面中的实验代码。

**任务 C：验证压缩插件开关**
1. 执行 `yarn build:prod`，记录 `dist/` 中 `.gz` 文件的数量。
2. 修改 `.env.production` 中的 `VITE_BUILD_COMPRESS` 为空（或删除该行），重新构建，统计 `.gz` 数量。
3. 恢复配置，写下结论：压缩开关是如何影响产物的？

**任务 D：阅读 `setup-extend`**
- 找到一个 `<script setup>` 组件，尝试去掉 `name` 属性后，观察 `keep-alive` 缓存行为的变化（可在标签页切换中观察）。实验后恢复。

### 5.6 课堂自测

1. 为什么 `createCompression` 只在构建时调用，而不在开发时调用？
2. `virtual:svg-icons-register` 是一个真实文件吗？它是如何工作的？
3. 自动导入带来的主要收益与代价各是什么？
4. `<script setup>` 本身为什么不能直接给组件命名？`setup-extend` 解决了什么问题？
5. 为什么预压缩（生成 `.gz`）比运行时压缩更高效？

<details>
<summary>参考答案（先自己思考再查看）</summary>

1. 压缩是为了减小生产环境的传输体积；开发时频繁启动，压缩会拖慢启动速度，而且开发环境传输的是本地资源，压缩意义不大。
2. 不是真实文件，而是插件在内存中生成的虚拟模块。`main.js` 导入它时，插件会返回包含所有 SVG 符号的注册代码。
3. 收益是代码简洁、减少样板导入；代价是来源不直观、缺少类型声明（本项目 `dts: false`）。
4. `<script setup>` 是编译宏，它的语义里没有 `name` 选项，因此无法在其中直接声明。`setup-extend` 允许在该语法中声明 `name`，供 `keep-alive` 等功能使用。
5. 压缩在构建时完成一次，服务器直接返回已压缩文件，避免每次请求都消耗 CPU；同时可以选择最高压缩级别，因为耗时不影响请求响应。

</details>

---

## 第 6 课：构建脚本、部署与兼容

### 6.1 学习目标

- 区分 `build:prod`、`build:stage` 与 `preview` 的用途，并能完成一次生产构建。
- 读懂 `dist/` 产物的目录结构与文件命名。
- 理解 `base` 与 Web 服务器配置对部署的影响。
- 了解 Windows 下的一键脚本与 IE 兼容提示页。

### 6.2 课前准备

- 确保 `node_modules` 已安装（第 2 课内容）。
- 准备好一个用于预览的终端窗口。

### 6.3 知识讲解

#### （一）`package.json` 的构建脚本

| 脚本 | 执行的命令 | 模式 | 读取的环境文件 | 典型用途 |
| --- | --- | --- | --- | --- |
| `yarn dev` | `vite` | development | `.env.development` | 本地开发 |
| `yarn build:prod` | `vite build` | production | `.env.production` | 正式环境打包 |
| `yarn build:stage` | `vite build --mode staging` | staging | `.env.staging` | 测试/预发布打包 |
| `yarn preview` | `vite preview` | 与上次构建一致 | 无需再读取 | 本地检查 `dist/` |

> 注意：`vite build` 默认模式是 `production`，所以 `build:prod` 没有显式指定 `--mode`。

#### （二）构建的过程（概念）

```
源码（.vue / .js / .scss / .svg）
   │
   ├─ 插件转换：Vue 编译、SCSS 编译、SVG 雪碧图、自动导入注入
   ├─ 模块打包：按路由与动态 import 切分 chunk（懒加载）
   ├─ 压缩与混淆：生产环境压缩 JS/CSS
   ├─ 哈希命名：文件名带内容 hash
   └─ 预压缩：生成 .gz（由 VITE_BUILD_COMPRESS 控制）
   ▼
dist/  （index.html + static/js + static/css + static/png ...）
```

你会在 `src/permission.js` 与 `src/store/modules/permission.js` 中看到 `import()` 的动态导入。它们是**代码分割**的来源：每个页面可以被打成独立的 chunk，按需加载。

#### （三）实际构建数据（本仓库的真实记录）

我们在编写本课程时，在沙箱环境中对仓库执行了一次生产构建，记录如下（不同机器结果会有差异）：

| 指标 | 结果 |
| --- | --- |
| Vite 版本 | 6.4.1 |
| 构建命令 | `vite build`（生产模式） |
| 构建耗时 | 约 20 秒 |
| `dist/` 总大小 | 约 7.0 MB |
| `dist/static/js/` 大小 | 约 5.4 MB，约 260 个文件 |
| 最大单个 JS 产物 | 约 1.7 MB（gzip 后约 553 KB） |
| `.gz` 压缩文件数量 | 约 110 个 |
| 构建日志告警 | 无阻断性错误 |

> **思考题**：为什么最大的 chunk 有 1.7 MB？它大概率包含哪些大依赖？（提示：Element Plus、ECharts、富文本编辑器、代码生成器相关的 `utils/generator` 等。）这正是第 39 课"性能优化"要解决的问题。

#### （四）`dist/` 产物结构

```
dist/
├── index.html            ← 入口 HTML（标题已替换，无 %VITE_APP_TITLE% 占位符）
├── index.html.gz         ← 预压缩文件
├── favicon.ico
└── static/
    ├── js/               ← 所有 JS chunk，文件名 = 名称-哈希.js
    │   ├── index-xxxx.js           （入口或业务 chunk）
    │   ├── login-xxxx.js           （登录页 chunk）
    │   └── ...
    ├── css/              ← 样式文件
    ├── png/ jpg/ gif/    ← 图片资源（按扩展名分目录）
    └── ...
```

阅读产物时的几个要点：

- `index.html` 中的 `<script type="module" src="/static/js/index-xxxx.js">` 是入口，浏览器从它开始加载。
- 文件名中的哈希值会随内容变化而变化。
- 每个 `.js` 通常都有对应的 `.js.gz`（若开启了 gzip）。

#### （五）预览构建产物：`yarn preview`

```bash
yarn build:prod
yarn preview
```

- `vite preview` 会启动一个静态服务器，服务 `dist/` 中的内容。
- **注意**：它并不会代理 `/prod-api` 到后端。所以如果后端没有配合，登录请求会失败。预览的主要目的是检查"静态资源是否能正确加载、路由刷新是否正常"。

#### （六）部署与 `base`、代理

生产部署的典型结构：

```
用户浏览器
   │  https://域名/
   ▼
Nginx（Web 服务器）
   ├─ /           → 返回 dist/ 静态文件（配合 try_files 支持 history 路由刷新）
   ├─ /prod-api/  → 反向代理到后端 http://后端地址/（去掉 /prod-api 前缀）
   └─ 开启 gzip_static on; 优先返回 .gz 文件
```

这里有三个关键点：

1. **`/prod-api` 由 Nginx 代理**：Vite 的 `proxy` 只在开发时有效，生产环境的前缀转发完全依赖 Web 服务器。这与第 0 课的结论一致。
2. **history 路由需要回退**：项目使用 `createWebHistory`，URL 如 `/system/user` 在服务器端并不存在对应文件。Nginx 需要 `try_files $uri $uri/ /index.html;`，否则刷新页面会 404。
3. **`base` 与子路径部署**：如果部署在 `https://域名/admin/`，需要把 `vite.config.js` 的 `base` 设为 `/admin/`，同时 Nginx 的 `location` 与路由回退规则也要相应调整。

> 常见问题：生产环境刷新页面 404，多数是第 2 点没有配置。这也是部署时最高频的问题之一。

#### （七）Windows 一键脚本（`bin/`）

| 文件 | 核心命令 | 作用 |
| --- | --- | --- |
| `bin/package.bat` | `yarn --registry=https://registry.npmmirror.com` | 安装依赖，生成 `node_modules` |
| `bin/run-web.bat` | `yarn dev` | 启动开发服务器 |
| `bin/build.bat` | `yarn build:prod` | 生产打包，输出 `dist/` |

- 脚本通过 `cd %~dp0` 切换到脚本所在目录，再 `cd ..` 回到项目根目录，因此可以在任意位置双击运行。
- `pause` 让窗口在执行结束后保持打开，便于查看输出。
- **编码提示**：这些 `.bat` 文件是 **GBK 编码**，所以用 UTF-8 方式查看时中文会乱码。这是 Windows 批处理的常见情况，编辑时请保持原编码，否则可能导致脚本中的中文输出异常。

#### （八）IE 兼容：`html/ie.html` 与条件注释

`index.html` 中包含：

```html
<!--[if lt IE 11]><script>window.location.href='/html/ie.html';</script><![endif]-->
```

- 这是 **条件注释**（IE 专有语法），只对 IE 10 及更低版本生效，会把用户跳转到 `html/ie.html`。
- `html/ie.html` 是一个静态提示页，告诉用户"请升级您的浏览器"。
- 这说明项目的目标运行环境是现代浏览器。Vue 3、Vite 产物本身也不支持 IE。
- 对于 IE 11 与其它旧浏览器，项目并没有做 polyfill 或降级处理，这是需要了解的边界。

### 6.4 操作步骤

1. 在仓库根目录执行生产构建：

   ```bash
   yarn build:prod
   ```

2. 构建完成后，记录以下信息：
   - 构建耗时（终端输出或计时）；
   - `dist/` 总大小（如 `du -sh dist`，Windows 可用文件属性查看）；
   - `dist/static/js/` 中最大的文件名与大小。

3. 执行预览：

   ```bash
   yarn preview
   ```

   在浏览器中打开终端输出的地址（通常是 `http://localhost:4173`），观察页面是否能加载出登录页样式与图片。

4. 尝试直接访问一个不存在的前端路由，例如 `/system/user`。如果页面出现 404 或空白，记录下来，并思考它与 Nginx `try_files` 的关系。

### 6.5 动手实践（课后任务）

**任务 A：构建与体积分析**
1. 执行 `yarn build:prod`，把 `dist/static/js/` 中体积最大的 5 个文件记录到表格中。
2. 思考：每个文件大概对应哪些依赖？（可通过文件内容中的字符串，如 `echarts`、`quill` 等关键词辅助判断。）

**任务 B：对比 staging 与 production**
1. 分别执行 `yarn build:prod` 与 `yarn build:stage`。
2. 每次构建后搜索 `dist/static/js` 中的 `/prod-api` 或 `/stage-api`，记录结果。
3. 总结：两次构建最终的差别仅在哪里？（提示：文件名哈希、接口前缀、压缩产物是否存在。）

**任务 C：压缩开关对比**
- 参考第 5 课任务 C 的结果，把本次对比写入笔记。

**任务 D：部署方案设计（纸上练习）**
1. 假设你要把本项目部署到 `https://example.com/admin/`，后端地址为 `https://api.example.com`。
2. 写出以下三项配置的草稿：
   - `vite.config.js` 中的 `base`；
   - `.env.production` 中的 `VITE_APP_BASE_API`（你认为应该是什么？是否需要改？）；
   - Nginx 中的 `location` 与 `try_files` 规则。
3. 将草稿与本课的讲解对照，找出你考虑不周的地方。

**任务 E：清理**
- 构建产物 `dist/` 已被 `.gitignore` 忽略，实验结束后可以删除。确认 `git status` 干净。

### 6.6 课堂自测

1. `yarn build:stage` 与 `yarn build:prod` 的唯一区别是什么？它们分别读取哪个环境文件？
2. 为什么 `yarn preview` 下登录请求可能失败？
3. 生产环境中 `/prod-api` 是谁负责转发到后端的？为什么开发环境的代理不够用？
4. 用户直接访问 `/system/user` 出现 404，原因是什么？如何解决？
5. 如果把项目部署在子路径 `/admin/` 下，需要同时修改哪些地方？
6. 为什么 `html/ie.html` 只能是一个提示页，而不是让旧浏览器正常运行？

<details>
<summary>参考答案（先自己思考再查看）</summary>

1. 两者都执行生产构建，但 `build:stage` 指定了 `--mode staging`，读取 `.env.staging`；`build:prod` 使用默认的 production 模式，读取 `.env.production`。
2. 因为 `vite preview` 只提供静态文件服务，不会代理 `/prod-api` 前缀到后端，所以接口请求会 404 或连接失败。
3. 由生产环境的 Web 服务器（如 Nginx）负责反向代理。Vite 的 `server.proxy` 只存在于开发服务器进程中，部署后不存在该进程。
4. 这是前端路由（history 模式）的特点：服务器上没有 `/system/user` 对应的文件。解决办法是在 Nginx 中配置 `try_files $uri $uri/ /index.html;`，让所有未命中的路径回退到 `index.html`，再由前端路由接管。
5. 需要修改 `vite.config.js` 的 `base`，并同步修改 Nginx 的 `location` 路径、路由回退规则，以及后端代理前缀（如有必要）。
6. Vue 3 与 Vite 构建产物依赖现代 JavaScript 语法与 API，IE 不支持，即使加载也无法运行。提示页是更诚实、更低成本的选择，说明了项目的支持范围。

</details>

---

## 本章小结

| 课时 | 核心问题 | 关键结论 |
| --- | --- | --- |
| 第 3 课 | 页面如何启动？配置如何注入？ | `index.html` → `main.js` → 挂载；`VITE_` 变量在构建期替换 |
| 第 4 课 | Vite 如何被配置？ | 函数式配置 + 别名 + 开发代理 + 构建命名 + PostCSS |
| 第 5 课 | 插件能力从哪来？ | 插件按职责封装，差异化启用；自动导入、SVG、压缩等各司其职 |
| 第 6 课 | 产物如何交付？ | `build:prod`/`build:stage` 生成 `dist/`；部署依赖 Nginx 代理与路由回退 |

**本章的一个核心观念**：开发环境的代理、构建时的环境变量替换、生产环境的 Web 服务器，这三者分别负责"开发能跑""产物正确""线上可用"。只要分清楚它们各自的职责，大多数部署问题都能快速定位。

**下一章预告**：第 2 章《Vue 3 核心与 Element Plus》，我们将从 `<script setup>` 与响应式开始，逐步进入组件、插槽与表单的实际写法。

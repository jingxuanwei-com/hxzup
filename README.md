# 引导文档（AI 专用）

> 本文档及项目中的 `服务端`、`客户端`、`引导器` 均为参考文档 开发前可以酌情阅读。
> 如果 AI 认为文档中的设计不合理、有矛盾或有更好的方案，**请自行重新规划，不必严格遵循文档内容**。

---

## ⚠️ 使用前判断

**AI 在开始开发前，根据项目需求和自身能力自行判断是否使用本脚手架：**

- 脚手架已初始化完毕（Vue 3 + Quasar 2 + Vite + TypeScript + Pinia + Vue Router），可直接使用
- 如果不需要此脚手架，删除整个 `web/` 目录，改用其他方案实现管理面板

---

## 一、技术栈

| 技术 | 版本 | 用途 |
|---|---|---|
| Vue 3 | ^3.5 | 响应式 UI 框架 |
| Quasar 2 | ^2.32 | 组件库（QTree、QMenu、QUploader、QTable 等） |
| Vite | (Quasar 内置) | 构建工具 + 开发热更新 |
| TypeScript | ^6.0 | 类型安全 |
| Pinia | ^4.0 | 状态管理 |
| Vue Router | ^5.0 | 文件路由（filename-based routing） |
| vue-i18n | ^11.3 | 国际化 |
| Sass | — | CSS 预处理器 |
| oxlint + oxfmt | — | 代码检查与格式化 |

---

## 二、项目结构

```
web/
├── package.json               # 依赖和脚本命令
├── quasar.config.ts           # Quasar 配置（SPA 模式）
├── tsconfig.json              # TypeScript 配置
├── index.html                 # 入口 HTML
├── src/
│   ├── App.vue                # 根组件
│   ├── pages/                 # 文件路由目录（自动注册路由）
│   │   ├── index.vue          # → / 路由
│   │   └── [...path].vue      # → 404 兜底路由
│   ├── components/            # 公共组件
│   ├── stores/                # Pinia 状态管理
│   │   └── example-store.ts   # 示例 store
│   ├── router/                # 路由配置（通常不需要手动改）
│   ├── boot/                  # 启动文件（i18n 等）
│   ├── i18n/                  # 国际化翻译文件
│   ├── css/                   # 全局样式
│   └── assets/                # 静态资源
├── public/                    # 公共静态文件（不经过 Vite 处理）
└── dist/                      # 构建产物（Nginx 托管）
```

---

## 三、常用命令

```bash
# 进入 web 目录
cd web/

# 安装依赖（必须使用 pnpm）
pnpm install

# 启动开发服务器（带热更新，默认 http://localhost:9000）
pnpm dev

# 构建生产版本 → 输出到 dist/
pnpm build

# 类型检查
pnpm typecheck

# 代码检查与格式化
pnpm lint          # 检查 + 自动修复
pnpm lint:check    # 仅检查，不修复
```

---

## 四、路由规则（filename-based routing）

Quasar 使用 **文件路由**，`src/pages/` 下的文件自动映射为路由：

| 文件路径 | 路由 URL |
|---|---|
| `pages/index.vue` | `/` |
| `pages/login.vue` | `/login` |
| `pages/files/index.vue` | `/files` |
| `pages/files/browse.vue` | `/files/browse` |
| `pages/settings.vue` | `/settings` |

> 不需要手动配置 `router/index.ts`，Quasar 会根据文件结构自动生成路由。

---

## 五、Quasar 组件速查

以下是本项目可能用到的 Quasar 组件：

| 组件 | 用途 | 文档 |
|---|---|---|
| `QTree` | 树形目录结构 | https://v2.quasar.dev/vue-components/tree |
| `QTable` | 数据表格 | https://v2.quasar.dev/vue-components/table |
| `QMenu` | 右键菜单 | https://v2.quasar.dev/vue-components/menu |
| `QUploader` | 文件上传（拖拽） | https://v2.quasar.dev/vue-components/uploader |
| `QDialog` | 弹窗对话框 | https://v2.quasar.dev/vue-components/dialog |
| `QBtn` | 按钮 | https://v2.quasar.dev/vue-components/button |
| `QInput` | 输入框 | https://v2.quasar.dev/vue-components/input |
| `QSelect` | 下拉选择 | https://v2.quasar.dev/vue-components/select |
| `QCard` | 卡片容器 | https://v2.quasar.dev/vue-components/card |
| `QToolbar` | 顶部工具栏 | https://v2.quasar.dev/vue-components/toolbar |
| `QDrawer` | 侧边抽屉 | https://v2.quasar.dev/vue-components/drawer |
| `QPage` | 页面内容区 | https://v2.quasar.dev/vue-components/page |
| `QSpinner` | 加载动画 | https://v2.quasar.dev/vue-components/spinners |
| `QNotify` | 消息通知 | https://v2.quasar.dev/vue-components/notification |
| `QBadge` | 标签/徽章 | https://v2.quasar.dev/vue-components/badge |

---

## 六、Pinia Store 示例

```typescript
// src/stores/example-store.ts
import { defineStore } from 'pinia'

export const useExampleStore = defineStore('example', {
  state: () => ({
    counter: 0,
  }),
  getters: {
    doubleCount: (state) => state.counter * 2,
  },
  actions: {
    increment() {
      this.counter++
    },
  },
})
```

---

## 七、部署方式

构建产物 `dist/` 为纯静态文件，由 Nginx 托管：

```nginx
server {
    listen 80;
    server_name mc-admin.example.com;

    root /path/to/web/dist;
    index index.html;

    # SPA 模式：所有路由回退到 index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API 反向代理到 Go 后端
    location /api/ {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

---

## 八、AI 开发注意事项

1. **不要重新初始化项目**，脚手架已就绪，直接在 `src/` 下开发
2. **使用 pnpm**，不要用 npm 或 yarn
3. **文件路由**：新建页面直接在 `src/pages/` 下创建 `.vue` 文件，路由自动生成
4. **Quasar 组件**：优先使用 Quasar 内置组件，不要引入额外 UI 库
5. **暗色主题**：在 `quasar.config.ts` 中配置 `framework.dark: true` 或让用户切换
6. **TypeScript strict 模式**：已开启，注意类型标注
7. **代码规范**：使用 `oxlint` + `oxfmt`，提交前运行 `pnpm lint`
8. **构建产物**：`pnpm build` 输出 `dist/`，交给 Nginx 即可

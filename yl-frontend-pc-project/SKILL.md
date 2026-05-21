---
name: yl-frontend-pc-project
description: 引领 PC 前端项目开发规范。当修改或新增 PC 端页面、组件、hooks、启动配置时使用。
metadata:
  version: "2026.5.21"
---

# 引领 PC 前端项目

## 技术栈

Vue 3 + Vite 5 + Element Plus + vxe-table + Pinia + ECharts + TypeScript

## 约束

- **不可修改后端代码**。需要改后端时先说明理由征求同意。
- 前端使用 pnpm，Git 提交用 commitizen (`pnpm commit`)
- 拉取后端代码注意分支，拿不准先问
- **模板引用使用 `useTemplateRef()`**：Vue 3.5+ 项目获取模板 DOM/组件引用时，使用 `useTemplateRef('refName')` 而非 `ref()`。`useTemplateRef()` 编译时与模板 `ref` 属性绑定，类型更安全。**使用时需先确认项目 Vue 版本 ≥ 3.5**，低于此版本继续用 `ref()`。
  ```ts
  // Vue ≥ 3.5
  const dataTableRef = useTemplateRef('dataTableRef')
  // Vue < 3.5
  const dataTableRef = ref()
  ```
- **批量操作后清除表格勾选**：DataTable 组件已暴露 `clearCheckbox()` 方法。页面批量操作（如批量送审）成功后，必须调用 `dataTableRef.value?.clearCheckbox()` 清除勾选，避免 vxe-grid 的 `reserve: true` 导致已变更状态的行仍被保留在勾选集合中，再次操作时报错。
  ```ts
  // 批量送审成功
  http.post('/xxx/submit', ids).then((res) => {
    ElMessage.success(res.message)
    dataTableRef.value?.clearCheckbox()  // 清除勾选
    handleTableRefresh()
  })
  ```

## 项目目录

所有项目在 `D:\resources\code\xayl\` 下。每个项目通常包含前端 PC、前端 APP（可选）、后端、数据库脚本。

## 启动项与打包配置

### 端口约定

| 模式 | 端口范围 | 说明 |
|------|----------|------|
| `local` | 30xx | 连本地后端电脑 |
| `onlineTest` | 90xx | 连测试环境 |

### 命令常用程度

| 等级 | 命令 | 说明 |
|------|------|------|
| **常用** | `dev:local`、`dev:onlineTest` | 日常开发 |
| **常用** | `build-prod` | 打生产包 |
| **常用** | `deploy` | 部署到测试环境 |
| **常用** | `preview` | 预览构建结果 |
| 很少用 | `dev`、`build` | 默认命令，配置不全且老旧 |

### .env 文件对应

| 文件 | 触发命令 | 用途 | 状态 |
|------|----------|------|------|
| `.env.development` | `dev` | 默认开发 | 很少用，配置老旧 |
| `.env.dev-local` | `dev:local` | 本地开发（连本地后端） | **常用** |
| `.env.dev-onlineTest` | `dev:onlineTest` | 连接测试环境 | **常用** |
| `.env.production` | `build` | 默认打包 | 很少用，配置老旧 |
| `.env.prod` | `build-prod`（`--mode prod`） | 生产环境打包 | **常用** |

### 配置文件关键点

- **Vite**：`vite.config.ts` 通过 `loadEnv(mode, ...)` 读取 `VITE_APP_*` 变量，设置 `server.port` 和 `server.proxy`
- **Vue CLI**：`vue.config.js` 通过 `process.env.VUE_APP_*` 读取变量，设置 `devServer.port` 和 `devServer.proxy`
- 代理目标、端口全部通过环境变量注入，不在配置文件中硬编码
- 详细模板见 `references/startup-config.md`

## 参考文档

开发时按需查阅：

| 文档 | 内容 |
|------|------|
| `references/architecture.md` | 核心架构原则：配置驱动 UI、Hook 组合式开发、JSX 渲染、枚举双轨制 |
| `references/components.md` | 核心组件 API：DataTable、Form、DialogForm、CurdDialog、EditTable、SelectDialog 等 |
| `references/hooks.md` | 核心 Hooks 用法：useTable、useCurd、useFormConfig、useEnum、useDoc 等 |
| `references/page-template.md` | 页面开发模板：列表页脚手架、审批页附加层、文件命名约定 |
| `references/dual-system.md` | 双系统共存：common 与 common-xxx 的目录结构、对照表、注册顺序 |
| `references/edittable-scroll.md` | EditTable 横向滚动条跳回问题的原因与修复 |
| `references/selectdialog-groupkey.md` | SelectDialog 按字段分组勾选（groupKey）的用法与实现 |
| `references/textarea-display.md` | 多行文本原样显示（white-space: pre-wrap）的使用场景 |
| `references/startup-config.md` | 启动项配置模板：scripts、.env 文件、vite.config.ts / vue.config.js |

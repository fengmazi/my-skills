---
name: yl-frontend-pc-project
description: 引领 PC 前端项目开发规范。当修改或新增 PC 端页面、组件、hooks 时使用。
metadata:
  version: "2026.5.15"
---

# 引领 PC 前端项目

## 技术栈

Vue 3 + Vite 5 + Element Plus + vxe-table + Pinia + ECharts + TypeScript

## 约束

- **不可修改后端代码**。需要改后端时先说明理由征求同意。
- 前端使用 pnpm，Git 提交用 commitizen (`pnpm commit`)
- 拉取后端代码注意分支，拿不准先问

## 项目目录

所有项目在 `D:\resources\code\xayl\` 下。每个项目通常包含前端 PC、前端 APP（可选）、后端、数据库脚本。

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

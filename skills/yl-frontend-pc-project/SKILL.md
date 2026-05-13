---
name: yl-frontend-pc-project
description: 银铃 PC 前端项目开发规范。当修改或新增 PC 端页面、组件、hooks 时使用。
metadata:
  version: "2026.5.13"
---

# 银铃 PC 前端项目

## 技术栈

Vue 3 + Vite 5 + Element Plus + vxe-table + Pinia + ECharts + TypeScript

## 约束

- **不可修改后端代码**。需要改后端时先说明理由征求同意。
- 前端使用 pnpm，Git 提交用 commitizen (`pnpm commit`)
- 拉取后端代码注意分支，拿不准先问

## 项目目录

所有项目在 `D:\resources\code\xayl\` 下。每个项目通常包含：
- 前端 PC 项目
- 前端 APP 项目（可选）
- 后端项目
- 数据库脚本文件

## 常见修改模式

详见 [common-modification-patterns](references/common-modification-patterns.md)

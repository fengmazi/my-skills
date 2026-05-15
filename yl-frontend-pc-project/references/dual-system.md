# 双系统共存（组件体系说明）

项目可能存在两套组件/hooks 体系共存的情况：一套基础 `common` 体系，一套带后缀的迁移/定制体系（后缀可能是 `nnw`、`bt`、`jb` 等，因项目而异）。

## 判断方式

看 `src/components/` 目录下有哪些子目录：
- 只有 `common/` → 单系统
- 有 `common/` + `common-xxx/` → 双系统共存，参考既有页面风格选择

## 对照表（以 `xxx` 代表后缀）

| 类别 | common 基础体系 | xxx 定制体系 |
|------|---------------|-------------|
| DataTable | `@/components/common/DataTable.vue` | `@/components/common-xxx/DataTable.vue` |
| Hooks 目录 | `@/hooks/` | `@/hooks-xxx/` |
| Utils 目录 | `@/utils/` | `@/utils-xxx/` |
| Enums 目录 | `@/enums/index` | `@/enums-xxx/index` |

## 全局注册顺序

`main.ts` 中先注册 xxx 版本，后注册 common 版本，保证 xxx 同名组件覆盖：

```ts
app.use(xxxComponents)
app.use(commonComponents)
```

## 注意事项

DataTable 的事件名和筛选配置可能因版本不同而有差异，开发时参考已有页面写法保持一致。

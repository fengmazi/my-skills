# 常见修改模式速查

新增业务页面时最常遇到的操作模式，按场景分类。

---

## 组件体系说明（常见双系统共存模式）

项目可能存在两套组件/hooks 体系共存的情况：一套基础 `common` 体系，一套带后缀的迁移/定制体系（后缀可能是 `nnw`、`bt`、`jb` 等，因项目而异）。

开发者通过看 `src/components/` 目录下有哪些子目录即可判断：
- 只有 `common/` → 单系统
- 有 `common/` + `common-xxx/` → 双系统共存，参考既有页面风格选择

两套体系对照表（以 `xxx` 代表后缀）：

| 类别 | common 基础体系 | xxx 定制体系 |
|------|---------------|-------------|
| DataTable | `@/components/common/DataTable.vue` | `@/components/common-xxx/DataTable.vue` |
| Hooks 目录 | `@/hooks/` | `@/hooks-xxx/` |
| Utils 目录 | `@/utils/` | `@/utils-xxx/` |
| Enums 目录 | `@/enums/index` | `@/enums-xxx/index` |

DataTable 的事件名和筛选配置也可能不同（因项目而异），开发时参考已有页面写法保持一致。

---

## SelectDialog 的 groupKey 特性

当需要在选择弹窗中按某个字段自动分组勾选时（如勾选某行后自动选中同入库单的所有行），在 SelectDialog 上设置 `groupKey`：

```vue
<SelectDialog
  v-model="showDialog"
  url="/api/queryDetail"
  rowId="rowKey"
  :columns="columns"
  :multiple="true"
  groupKey="orderId"
  @change="handleSelected"
/>
```

效果：勾选某行时，所有 `orderId` 相同的行自动选中，不同 `orderId` 的行自动取消。
实现原理：SelectDialog 内部 `checkboxChange` 事件中根据 `groupKey` 字段值批量操作复选框。

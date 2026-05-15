# 核心架构原则

## 1. 配置驱动 UI

表单和表格不由模板硬编码，而是用配置数组声明。

### 表单：`FormItem[][]` 声明

```ts
// useFormConfig.ts 中定义
export const formConfig = ref<FormItem[][]>([
  [
    { label: '单号', prop: 'billNo', type: 'input', placeholder: '请输入' },
    { label: '日期', prop: 'date', type: 'datePicker', valueFormat: 'YYYY-MM-DD' },
  ],
  [
    { label: '供应商', prop: 'supplierId', type: 'select', options: supplierOptions },
  ],
])
```

模板侧只做渲染：

```vue
<Form :config="formConfig" :model="formData" />
```

### 表格：`Column[]` 声明

```ts
// useColumn.ts 中定义
export const columns: Column[] = [
  { type: 'checkbox', width: 50 },
  { field: 'billNo', title: '单号', width: 160 },
  { field: 'supplierName', title: '供应商', width: 200 },
  { field: 'amount', title: '金额', width: 120, align: 'right', type: 'money' },
  { title: '操作', width: 180, slots: { default: 'action' }, fixed: 'right' },
]
```

---

## 2. Hook 组合式开发

页面逻辑通过 hooks 编排，setup 中按固定顺序组合：

```
useTable → useFormConfig → useCurd → useDetail / useOperate
```

每个 hook 职责单一：

| Hook | 输入 | 输出 |
|------|------|------|
| `useTable` | 查询 API + 查询条件 | `tableData`, `loading`, `pagination`, `onSearch` |
| `useFormConfig` | 枚举/档案数据 | `formConfig` 响应式配置 |
| `useCurd` | CRUD API + 刷新回调 | `handleAdd`, `handleEdit`, `handleDelete` |
| `useDetail` | 详情 API | `detailVisible`, `detailData`, `openDetail` |
| `useOperate` | 审批 API | `handleApprove`, `handleReject`, `handleRevoke` |

页面 setup 典型结构：

```vue
<script setup lang="ts">
// 1. 表格
const { tableData, loading, pagination, onSearch } = useTable(url, queryParams)

// 2. 枚举/档案
const { enums } = useEnum('groupNo')
const { docMap } = useDoc('docType')

// 3. 表单配置
const { formConfig } = useFormConfig(enums, docMap)

// 4. 列配置
const columns = useColumn(enums)

// 5. CRUD
const { handleAdd, handleEdit, handleDelete } = useCurd(url, onSearch)

// 6. 详情
const { detailVisible, detailData, openDetail } = useDetail()
</script>
```

---

## 3. JSX 渲染复杂区域

模板保持简洁，toolbar 按钮、操作列、格式化单元格用 TSX：

```tsx
// 操作列
{
  title: '操作',
  width: 180,
  slots: { default: 'action' },
}
```

```vue
<!-- 模板中 -->
<template #action="{ row }">
  <el-button link type="primary" @click="handleEdit(row)">修改</el-button>
  <el-button link type="danger" @click="handleDelete(row)">删除</el-button>
</template>
```

复杂格式化用 `formatter`：

```ts
{
  field: 'status',
  title: '状态',
  formatter: ({ row }) => <el-tag type={statusMap[row.status]}>{statusMap[row.status]}</el-tag>,
}
```

---

## 4. 枚举双轨制

| 类型 | 存放位置 | 使用方式 | 适用场景 |
|------|---------|---------|---------|
| 静态枚举 | `src/enums/index.ts` | 直接 import | 前端固定选项（性别、对齐方式） |
| 动态枚举 | 服务端维护 | `useEnum(groupNo)` | 业务枚举（单据状态、审批结果） |

```ts
// 静态枚举
import { GENDER_OPTIONS } from '@/enums'

// 动态枚举
const { enums } = useEnum('PO_STATUS')
// enums.value → [{ label: '待审核', value: '0' }, { label: '已审核', value: '1' }]
```

---

## 5. 迁移隔离（双系统共存）

迁入代码放 `-xxx` 后缀目录，全局注册先 xxx 后 common 保证覆盖。详见 `references/dual-system.md`。

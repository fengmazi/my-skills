# 页面开发模板

## 列表页标准模板

新增业务页面时按以下结构搭建：

```
src/views/{module}/
├── index.vue          # 页面模板
├── useTable.ts        # 表格数据
├── useFormConfig.ts   # 搜索表单配置
├── useColumn.ts       # 表格列配置
├── useCurd.ts         # CRUD 操作（可选）
└── types.ts           # 类型定义
```

### index.vue

```vue
<template>
  <div class="page-container">
    <!-- 搜索区域 -->
    <Form :config="formConfig" :model="queryParams" @search="onSearch" />

    <!-- 表格区域 -->
    <DataTable
      :columns="columns"
      :data="tableData"
      :loading="loading"
      :pagination="pagination"
      @page-change="onPageChange"
    >
      <template #action="{ row }">
        <el-button link type="primary" @click="handleEdit(row)">修改</el-button>
        <el-button link type="danger" @click="handleDelete(row)">删除</el-button>
      </template>
    </DataTable>

    <!-- CRUD 弹窗 -->
    <CurdDialog
      v-model="curdState.visible"
      :mode="curdState.mode"
      :config="formConfig"
      :data="curdState.data"
      @submit="handleCurdSubmit"
    />
  </div>
</template>

<script setup lang="ts">
import { useTable } from './useTable'
import { useFormConfig } from './useFormConfig'
import { useColumn } from './useColumn'
import { useCurd } from './useCurd'

const { tableData, loading, pagination, onSearch, onPageChange } = useTable()
const { formConfig, queryParams } = useFormConfig()
const { columns } = useColumn()
const { curdState, handleEdit, handleDelete, handleCurdSubmit } = useCurd(onSearch)
</script>
```

### useTable.ts

```ts
import { ref } from 'vue'
import { getList } from '@/api/module'

export function useTable() {
  const queryParams = ref({ billNo: '', date: '' })

  const {
    tableData,
    loading,
    pagination,
    onSearch,
    onPageChange,
  } = useTable('/api/module/list', queryParams)

  return { tableData, loading, pagination, queryParams, onSearch, onPageChange }
}
```

### useFormConfig.ts

```ts
import { computed } from 'vue'
import type { FormItem } from '@/types'

export function useFormConfig() {
  const queryParams = ref({ billNo: '', date: '' })

  const formConfig = computed<FormItem[][]>(() => [
    [
      { label: '单号', prop: 'billNo', type: 'input' },
      { label: '日期', prop: 'date', type: 'datePicker', valueFormat: 'YYYY-MM-DD' },
    ],
  ])

  return { formConfig, queryParams }
}
```

### useColumn.ts

```ts
import type { Column } from '@/types'

export function useColumn() {
  const columns: Column[] = [
    { type: 'checkbox', width: 50 },
    { field: 'billNo', title: '单号', width: 160 },
    { field: 'amount', title: '金额', width: 120, align: 'right', type: 'money' },
    { title: '操作', width: 180, slots: { default: 'action' }, fixed: 'right' },
  ]

  return { columns }
}
```

### useCurd.ts

```ts
export function useCurd(onRefresh: () => void) {
  return useCurd('/api/module', onRefresh)
}
```

---

## 审批页面附加层

审批类页面在列表页基础上额外引入：

```ts
// 详情
const { detailVisible, detailData, openDetail } = useDetail()
// 审批操作
const { handleApprove, handleReject, handleRevoke } = useOperate('/api/module', onSearch)
```

操作列增加审批按钮：

```vue
<template #action="{ row }">
  <el-button link type="primary" @click="openDetail(row)">详情</el-button>
  <el-button v-if="row.status === 'PENDING'" link type="success" @click="handleApprove(row)">
    审核
  </el-button>
</template>
```

---

## 文件命名约定

| 文件 | 导出方式 | 命名 |
|------|---------|------|
| `useTable.ts` | `export function` | `use{Module}Table`（如重名冲突时） |
| `useFormConfig.ts` | `export function` | `use{Module}FormConfig` |
| `useColumn.ts` | `export function` | `use{Module}Column` |
| `useCurd.ts` | `export function` | `use{Module}Curd` |
| `types.ts` | `export interface` | PascalCase |

---

## filterParam 筛选配置类型

表格列的 `filterParam` 支持四种筛选类型：

| type | 渲染控件 | 示例 |
|------|---------|------|
| `String` | 文本输入 | `filterParam: { type: String }` |
| `Number` | 数字输入 | `filterParam: { type: Number }` |
| `Date` | 日期选择 | `filterParam: { type: Date }` |
| `Array` | 下拉选择 | `filterParam: { type: Array, options: enums.statusOption }` |

### 常用枚举选项

- `enums.statusOption` — 启用/停用
- `enums.booleanOption` — 是/否
- `enums.approveStatusOption` — 审批状态
- `enums.orderStatusOption` — 订单状态

### 动态选项

```ts
const column = getColumn('fieldName')
column.filterParam!.options = res.options
```

# 核心 Hooks

## useTable

分页表格数据管理。封装了请求、分页、排序、筛选状态。

```ts
const {
  tableData,     // Ref<T[]>         表格数据
  loading,       // Ref<boolean>     加载状态
  pagination,    // Reactive         分页 { currentPage, pageSize, total }
  onSearch,      // () => void       触发查询（会重置到第一页）
  onRefresh,     // () => void       保持当前页刷新
  onPageChange,  // (page) => void   翻页
  onSortChange,  // (sort) => void   排序变更
} = useTable('/api/list', queryParams)
```

**参数**：
- `url: string` — 查询 API 路径
- `queryParams: Ref<Record<string, any>>` — 查询条件（响应式）
- `options?: { immediate?: boolean, pageSize?: number }` — 可选配置

**典型用法**：

```ts
const queryParams = ref({ billNo: '', date: '' })
const { tableData, loading, pagination, onSearch } = useTable('/api/purchase/list', queryParams)
```

---

## useCurd

新增/修改/删除操作管理。

```ts
const {
  curdState,       // Reactive<{ visible, mode, data }>
  handleAdd,       // () => void                    打开新增弹窗
  handleEdit,      // (row: T) => void              打开编辑弹窗
  handleDelete,    // (row: T) => Promise<void>     删除确认
  handleSubmit,    // (data: T) => Promise<void>    提交表单
} = useCurd('/api/entity', onSearch)
```

**参数**：
- `baseUrl: string` — CRUD API 基础路径
- `onRefresh: () => void` — 操作成功后的刷新回调
- `options?: { beforeSubmit?, afterSubmit? }` — 生命周期钩子

---

## useFormConfig

表单配置生成，依赖枚举和档案数据。

```ts
const { formConfig } = useFormConfig(enums, docMap)
```

返回的 `formConfig` 自动根据枚举/档案生成 select 选项。

---

## useColumn

表格列配置。

```ts
const columns = useColumn(enums)
```

返回的 `columns` 中，枚举类型字段自动关联选项映射，用于 `formatter` 显示 label。

---

## useEnum

服务端动态枚举获取。

```ts
const { enums, loading } = useEnum('PURCHASE_STATUS')
// enums.value → [{ label: '待审核', value: 'PENDING' }, ...]
```

**参数**：`groupNo: string` — 枚举分组号

**缓存**：多次调用同一 `groupNo` 会复用缓存，不会重复请求。

---

## 合计行（useTable 的 sum 与 footerMethod）

`useTable` 内置了默认的 `footerMethod`，但不推荐直接使用。**合计行必须在本页面覆盖重写**，不应依赖 useTable 的默认实现。

### 原因

- 默认 footerMethod 统一 `toFixed(2)`，无法适配不同精度需求（如金额需 4 位 + 千分位 `formatMoney`）
- 默认 "合计" 固定在第 0 列，不能跟随是否有选择列动态调整位置

### 规范

1. **合计优先在页面覆盖重写**：从 `useTable` 解构 `sum`，在页面内写 `footerMethod`，不要使用 useTable 默认导出的 `footerMethod`

2. **"合计"位置规则**：
   - 列表**有选择列（checkbox）** → "合计"放在第 **2** 列（index 1，即选择列后面一列）
   - 列表**无选择列**（全是展示列） → "合计"放在第 **1** 列（index 0）

3. **格式化与列定义保持一致**：footerMethod 中各字段的格式化方式应与该列 `formatter` 一致

### 示例

```tsx
// 从 useTable 解构 sum
const { tableData, total, pageNo, pageSize, loading, queryList, sum, handleTableQuery } =
  useTable('/materialOutbound/query')

// 有选择列的 detailColumns — "合计"在 index 1
const { columns: detailColumns } = useColumn([
  { type: 'checkbox', width: 50 },          // index 0
  { field: 'approveStatus', title: '审批状态' }, // index 1 ← "合计"放这里
  { field: 'amount', title: '数量', ... },
  { field: 'taxMoney', title: '金额', ... },
])

const footerMethod: VxeTablePropTypes.FooterMethod = ({ columns }: { columns: any[] }) => {
  return [
    columns.map((col: any, index: number) => {
      if (index === 1) return '合计'  // 有选择列，放第二列
      if (['amount'].includes(col.field) && sum.value[col.field] != null) {
        return Number(sum.value[col.field]).toFixed(4)
      }
      if (['taxMoney'].includes(col.field) && sum.value[col.field] != null) {
        return formatMoney(sum.value[col.field], 4)
      }
      return ''
    }),
  ]
}
```

```tsx
// 无选择列的 detailColumns — "合计"在 index 0
const { columns: detailColumns } = useColumn([
  { field: 'approveStatus', title: '审批状态' },  // index 0 ← "合计"放这里
  { field: 'amount', title: '数量', ... },
])

const footerMethod: VxeTablePropTypes.FooterMethod = ({ columns }: { columns: any[] }) => {
  return [
    columns.map((col: any, index: number) => {
      if (index === 0) return '合计'  // 无选择列，放第一列
      ...
    }),
  ]
}
```

### 模板配置

```html
<DataTable
  :show-footer="listType === 'detail'"
  :footer-method="listType === 'detail' ? footerMethod : undefined"
  ...
/>
```

---

## useDoc

档案数据获取（如供应商、物料、仓库等基础档案）。

```ts
const { docMap, docList, loading } = useDoc('SUPPLIER')
// docMap.value → { '1': '供应商A', '2': '供应商B' }
// docList.value → [{ id: '1', name: '供应商A' }, ...]
```

---

## useDetail

审批详情查看。

```ts
const {
  detailVisible,   // Ref<boolean>
  detailData,      // Ref<T | null>
  openDetail,      // (row: T) => void
  closeDetail,     // () => void
} = useDetail()
```

---

## useOperate

审批操作（审核、弃审、撤销）。

```ts
const {
  handleApprove,   // (row: T) => Promise<void>
  handleReject,    // (row: T, reason: string) => Promise<void>
  handleRevoke,    // (row: T) => Promise<void>
} = useOperate('/api/purchase', onRefresh)
```

触发审批后自动弹窗确认、调用 API、刷新列表。

---

## useImport

Excel 导入管理。

```ts
const {
  importVisible,    // Ref<boolean>
  importUrl,        // string
  handleImport,     // () => void    打开导入弹窗
  handleUpload,     // (file) => void 处理上传
} = useImport('/api/purchase/import', onSearch)
```

---

## usePrint

单据打印。

```ts
const { handlePrint } = usePrint()
// handlePrint(billId, billType) — 打开打印预览
```

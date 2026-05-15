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

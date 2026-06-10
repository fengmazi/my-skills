# API 请求模式

## 分页查询参数格式

项目后端统一使用以下请求参数格式：

```ts
{
  current: 1,           // 当前页
  pageSize: 20,         // 每页条数
  querys: [             // 查询条件
    {
      property: 'name',
      operator: 'CONTAINS',
      value: '测试'
    }
  ],
  sorts: [              // 排序
    {
      property: 'createDate',
      sort: 'DESC'
    }
  ]
}
```

### 操作符表

| 操作符 | 说明 |
|-----|-----|
| EQUAL | 等于 |
| NOT_EQUAL | 不等于 |
| CONTAINS | 包含 |
| NOT_CONTAINS | 不包含 |
| GT | 大于 |
| GTE | 大于等于 |
| LT | 小于 |
| LTE | 小于等于 |
| BETWEEN | 范围 |
| NOT_BETWEEN | 不在范围 |
| IN | 在集合中 |
| NOT_IN | 不在集合中 |
| NULL | 为空 |
| NOT_NULL | 不为空 |
| STARTS_WITH | 开头匹配 |
| ENDS_WITH | 结尾匹配 |

## 常用操作模式

### CRUD 标准写法

```ts
import http from '@/plugins/axios'
import { ElMessage, ElMessageBox } from 'element-plus'

// 新增
http.post('/api/add', formData).then(() => {
  ElMessage.success('新增成功')
  handleTableRefresh()
})

// 修改
http.post('/api/update', formData).then(() => {
  ElMessage.success('修改成功')
  handleTableRefresh()
})

// 删除（带确认）
ElMessageBox.confirm('确定删除吗？', '提示', {
  confirmButtonText: '确定',
  cancelButtonText: '取消',
  type: 'warning'
}).then(() => {
  http.delete('/api/delete/id').then(() => {
    ElMessage.success('删除成功')
    handleTableRefresh()
  })
})

// async/await 写法
const handleSubmit = async () => {
  try {
    loading.value = true
    await http.post('/api/add', formData)
    ElMessage.success('操作成功')
    handleTableRefresh()
  } catch (error) {
    console.error('操作失败:', error)
  } finally {
    loading.value = false
  }
}
```

### 审批操作

```ts
// 提交审批
http.post('/api/submit', {
  businessId: row.id,
  taskTypeCode: '1001001'
}).then(() => {
  ElMessage.success('提交成功')
  handleTableRefresh()
})

// 审批通过/驳回
http.post('/api/check', {
  businessId: row.id,
  taskTypeCode: '1001001',
  approveStatus: 'pass',   // pass: 通过, reject: 驳回
  opinion: '同意'
}).then(() => {
  ElMessage.success('审批成功')
  handleTableRefresh()
})

// 撤销
http.post('/api/revocation', {
  businessId: row.id,
  taskTypeCode: '1001001'
}).then(() => {
  ElMessage.success('撤销成功')
  handleTableRefresh()
})
```

### 批量操作

项目中有两种批量操作模式，按场景选用：

---

#### 模式一：从已加载数据中勾选（常规批量）

适用于操作列已有批量按钮、用户从当前表格勾选行后直接提交的场景。

```ts
http.post('/api/batch', {
  ids: selectedRows.map(r => r.id)
}).then(() => {
  ElMessage.success('提交成功')
  dataTableRef.value?.clearCheckbox()  // 清除勾选
  handleTableRefresh()
})
```

---

#### 模式二：进入批量模式 → 自动请求符合条件的全部数据（筛选后批量）

**适用场景**：用户点击"批量审批"等按钮后，需要自动筛选出符合批量操作条件的数据（如所有 `approveStatus === 'submit'` 的记录），而不是让用户手动筛选表格后再勾选。

**核心思路**：点击批量按钮时，向 `queryList` 注入过滤条件并重新请求表格数据，使表格仅展示符合批量条件的数据；用户勾选后确认提交；取消或完成后移除过滤条件恢复原始数据。

**参考实现**：`vehicle-admin/src/views/archive/vehicle.vue`（批量审批）

##### 1. 状态管理

```ts
// 批量模式开关
const batchApproveVisible = ref(false)

// 批量确认弹窗开关（如需填写审批意见等额外信息）
const batchApproveDialogVisible = ref(false)
const batchApproveForm = reactive({
  approveStatus: 'pass' as 'pass' | 'reject',
  comment: '',
})

// 表格选中数据
let tableSelected: any[] = []
```

##### 2. 进入批量模式（工具栏按钮）

```tsx
const toolbarBtns = () => [
  batchApproveVisible.value ? (
    <>
      <el-button type="success" size="small" onClick={() => handleBatchApproveBtn()}>
        确认批量审批
      </el-button>
      <el-button size="small" onClick={() => {
        // 取消：移除过滤条件，恢复原始数据
        batchApproveVisible.value = false
        queryList.value = [...queryList.value.filter(
          (q: any) => q.property !== 'approveStatus'
        )]
        handleQueryPage(pageNo.value, pageSize.value, queryList.value, [])
      }}>
        取消
      </el-button>
    </>
  ) : (
    <el-button
      type="success"
      size="small"
      onClick={() => {
        // 进入批量模式：注入过滤条件，重新请求只含符合条件的数据
        batchApproveVisible.value = true
        queryList.value = [...queryList.value.filter(
          (q: any) => q.property !== 'approveStatus'
        )]
        queryList.value.push({
          property: 'approveStatus',
          value: 'submit',
          operator: 'EQUAL',
        })
        handleQueryPage(pageNo.value, pageSize.value, queryList.value, [])
      }}
    >
      批量审批
    </el-button>
  ),
]
```

**关键点**：
- 进入批量模式前先移除旧的同名字段过滤条件，避免重复
- 注入过滤条件后调用 `handleQueryPage`，表格自动刷新为只含符合批量条件的数据
- 取消时同样移除过滤条件并重新请求，恢复原始表格

##### 3. 确认提交

```ts
// 按钮点击 → 打开确认弹窗
const handleBatchApproveBtn = () => {
  if (!tableSelected.length) return ElMessage.warning('请至少选择一条记录')
  batchApproveDialogVisible.value = true
}

// 弹窗确认 → 调接口
const handleBatchApprove = () => {
  if (!tableSelected.length) return ElMessage.warning('请至少选择一条记录')
  http.post('/docVehicle/check', {
    ids: tableSelected.map((item: any) => item.vehicleId),
    approveStatus: batchApproveForm.approveStatus,
    comment: batchApproveForm.comment,
  }).then((res: any) => {
    ElMessage.success(res.message || '批量审批成功')
    // 清理状态
    batchApproveDialogVisible.value = false
    batchApproveVisible.value = false
    batchApproveForm.approveStatus = 'pass'
    batchApproveForm.comment = ''
    tableSelected = []
    // 移除过滤条件，恢复原始数据
    queryList.value = [...queryList.value.filter(
      (q: any) => q.property !== 'approveStatus'
    )]
    handleQueryPage(pageNo.value, pageSize.value, queryList.value, [])
  })
}
```

##### 4. DataTable 配置

```vue
<DataTable
  rowId="vehicleId"
  :columns="columns"
  :tableData="tableData"
  :toolbarBtns="toolbarBtns"
  :checkboxConfig="{ checkMethod }"
  @checkboxAll="checkboxChange"
  @checkboxChange="checkboxChange"
/>
```

**流程总结**：

```
点击"批量审批" → 注入过滤条件 → 表格刷新（只显示符合条件的）
    ↓
用户勾选行 → 点击"确认批量审批" → 弹窗 → 调接口
    ↓
成功 → 清除过滤条件 → 恢复原始表格
失败/取消 → 同样清除过滤条件 → 恢复原始表格
```

### 文件下载

```ts
import { downloadFile } from '@/utils'

// POST 下载
downloadFile('/api/export', '文件名.xlsx', 'post', { params: {} })
// GET 下载
downloadFile('/api/download/file', '文件名.pdf', 'get')
```

### 关联数据 / 下拉选项查询

```ts
// 查询关联明细
http.post('/api/detail/query', { masterId: row.id }).then(res => {
  detailData.value = res.data
})

// 查询下拉选项（映射为 { label, value } 格式）
http.post('/api/options').then(res => {
  getConfig('field').options = res.data.map(item => ({
    label: item.name,
    value: item.id
  }))
})
```

## useTable 请求流程

`useTable` 内部封装了分页请求，将 `pageNo`、`pageSize`、`queryList`、`sortList` 组装为后端要求的分页参数格式。页面只需调用 `handleTableRefresh()` 或 `handleQueryPage()` 即可触发请求。

**注意**：不同项目（`-bt` vs `-nnw` 后缀）的 `useTable` 实现可能有差异，以当前项目实际代码为准。

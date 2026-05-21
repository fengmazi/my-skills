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

```ts
http.post('/api/batch', {
  ids: selectedRows.map(r => r.id)
}).then(() => {
  ElMessage.success('提交成功')
  handleTableRefresh()
})
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

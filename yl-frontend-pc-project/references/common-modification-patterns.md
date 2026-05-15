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

## EditTable 横向滚动条跳回问题

EditTable 的 deep watcher 中**不要调用 `reloadData()`**，否则子表列过多出现横向滚动条时，在靠后的列输入会导致滚动条跳回开头：

```ts
// ❌ 错误：会导致 scrollLeft 复位
watch(
  () => tableData,
  () => {
    emit('update:modelValue', tableData.value)
    xTable.value && xTable.value.reloadData(tableData.value)
  },
  { deep: true }
)

// ✅ 正确：注释掉 reloadData 并加警告说明，保留代码备后续参考
watch(
  () => tableData,
  () => {
    emit('update:modelValue', tableData.value)
    // 注意：reloadData 会导致横向滚动条跳回开头，不要启用
    // xTable.value && xTable.value.reloadData(tableData.value)
  },
  { deep: true }
)
```

> 原因：`reloadData()` 会完全重建 grid body，将 `scrollLeft` 复位为 0。旧版 `common-nnw/EditTable.vue` 早已注释掉此行。

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

---

## 多行文本原样显示（textarea 内容保留格式）

当展示多行文本框（textarea）输入的内容时（如审批意见、初步结论等），后端存的是带 `\n` 的原始文本。要让"填什么样，显示什么样"，使用以下 CSS：

```css
.summary-content {
  white-space: pre-wrap;    /* 保留换行和空格，超出自动换行 */
  word-wrap: break-word;    /* 长单词/URL 在边界断行 */
  overflow-wrap: break-word;
}
```

核心属性是 `white-space: pre-wrap`：它保留了 `<pre>` 的空白语义（换行、连续空格不折叠），但同时允许超出容器宽度时自动换行（区别于 `pre` 会在溢出时出现横向滚动条）。

> 使用场景：打印模板、详情弹窗、审批意见展示等需要原样呈现用户输入格式的地方。

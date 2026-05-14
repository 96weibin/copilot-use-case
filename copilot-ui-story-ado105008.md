# Copilot 辅助实现 UI Story：ADO 105008 Diagnostics Tab 更新

> 演示如何通过 Copilot + Figma Design Inspector + ADO MCP，完整实现一个带有设计稿的 UI 需求，包括 ag-Grid 新增列、Badge 样式、i18n、以及过程中的 bug 修复。

---

## 背景

ADO 105008 是一个 UI Story，要求对 Aura 的 Case Validation → Diagnostics 标签页做如下改动：

1. 将第一个分组的标题从 **"Warnings"** 改为 **"Warnings, errors and assumptions"**
2. 新增一列 **"Type"**，显示带颜色 Badge 的诊断类型（Warning / Error / Assumption）
3. 将原来的 "Warning" 列改名为 **"Message"**

---

## 使用的 Copilot 能力

| 能力 | 用途 |
|---|---|
| `ui-story-delivery` skill | 读取 ADO Story + 找 Figma 链接 + 制定实现计划 |
| `figma-design-inspector` skill | 打开 Figma 截图，提取 Badge 颜色和布局 |
| ADO MCP (`ado-wit_get_work_item`) | 读取 Story 详情、AC、关联 Work Items |
| Chrome DevTools MCP | 操作浏览器打开 Figma 并截图 |

---

## 步骤演示

### 1. 触发 `ui-story-delivery` skill

输入：
```
帮我实现 ADO 105008
```

Copilot 自动：
- 调用 ADO MCP 读取 Story 内容和 Acceptance Criteria
- 从 AC 中找到 Figma 链接
- 搜索项目 KB (`.github/kb/`) 了解代码结构
- 调用 `figma-design-inspector` 打开 Figma 截图

### 2. Figma Design Inspector 提取设计细节

Figma 登录后，Copilot 截图并分析：

| Badge 类型 | 背景色 | 文字色 |
|---|---|---|
| Error | `#d42b2b` | `#ffffff` |
| Warning | `#f5c518` | `#1a1a1a` |
| Assumption | `#1565d8` | `#ffffff` |

> 💡 Figma 中颜色直接从截图 + a11y tree 读取，无需手动取色。

### 3. Copilot 制定计划并等待确认

Copilot 生成实现计划，列出所有要改的文件：

- `locales/en/aura.json` — 新增 i18n keys
- `auraResources.ts` — 同步 key 访问器
- `solver-log.html` — 更新 section 标题绑定
- `solver-log.ts` — 新增 Type 列 + 加载 ERROR/ASSUMPTION severity
- `solver-log.less` — 添加 Badge 样式

用户在确认阶段提出一个调整：
> "Warning column 应该改成 Message"

Copilot 更新计划后开始实现。

### 4. 核心实现：ag-Grid Type 列

```typescript
// solver-log.ts — Type 列定义
{
    headerName: Resources.casesAnalysis.type,
    field: 'severity',
    pinned: 'left',
    width: 130,
    cellRenderer: (params) => {
        if (params.node.group) return '';
        const label = SolverLogViewModel.getDiagnosticTypeLabel(params.value);
        const variant = SolverLogViewModel.getDiagnosticTypeVariant(params.value);
        return `<span class="diagnostic-badge diagnostic-badge--${variant}">${label}</span>`;
    },
    valueFormatter: (params) => {
        if (params.node.group) return '';
        return SolverLogViewModel.getDiagnosticTypeLabel(params.value);
    }
}
```

> ⚠️ **重要**：`valueFormatter` 需要单独处理 group row（返回空字符串），否则 Excel 导出时 group 行会显示 "Warning"。  
> `cellRenderer` 仅用于 UI 渲染，不影响导出。

### 5. Badge 样式

```less
// solver-log.less
.diagnostic-badge {
    display: inline-block;
    padding: 2px 8px;
    border-radius: 4px;
    font-size: 12px;
    font-weight: 500;

    &--error      { background: #d42b2b; color: #ffffff; }
    &--warning    { background: #f5c518; color: #1a1a1a; }
    &--assumption { background: #1565d8; color: #ffffff; }
}
```

---

## 遇到的 Bug 及修复

### Bug 1：group row 显示 "Warning"

**现象**：Type 列在 ag-Grid group row 里显示了 "Warning" 文字，而其他列的 group row 是空的。

**原因**：`cellRenderer` 没有对 `params.node.group` 做判断，ag-Grid 在 group row 上也调用了 renderer。

**修复**：
```typescript
cellRenderer: (params) => {
    if (params.node.group) return '';  // ← 加这一行
    // ...
}
```

### Bug 2：Badge 颜色与 Figma 不一致

**现象**：实现后发现 Warning/Error/Assumption 的颜色与 Figma 不符。

**解决**：重新用 `figma-design-inspector` 聚焦到对应 Frame（按 `2` 键 zoom to selection，再 `+` 放大），直接从 a11y tree 读出精确颜色值。

---

## 涉及的文件

| 文件 | 变更内容 |
|---|---|
| `locales/en/aura.json` | 新增 `warningsErrorsAssumptions`, `type`, `message`, `diagnosticType.*` 等 key |
| `aura-workspace/resources/auraResources.ts` | 同步新 key 的强类型访问器 |
| `solver-log.html` | section 标题 i18n key 改为 `warningsErrorsAssumptions` |
| `solver-log.ts` | 新增 Type 列、加载3种 severity、重命名 Message 列、更新导出文件名 |
| `solver-log.less` | 新增 `.diagnostic-badge` 及3种颜色变体 |

---

## 经验总结

1. **Figma → Copilot 全程自动**：从 ADO 读链接 → 打开 Figma → 截图分析，无需手动取色或量尺寸。

2. **ag-Grid cellRenderer + valueFormatter 职责分离**：
   - `cellRenderer` → 纯 UI 显示（HTML）
   - `valueFormatter` → Excel/CSV 导出文本
   - 两者可以同时存在，但都需要对 group row 做保护

3. **计划确认环节很重要**：Copilot 制定计划后等待确认，用户在这里调整了列名（Description→Message），避免了返工。

4. **KB 驱动**：Copilot 在实现前读了 `.github/kb/projects/aura.md`，了解了 `auraResources.ts` 需要和 `aura.json` 同步更新的约定，没有漏改。

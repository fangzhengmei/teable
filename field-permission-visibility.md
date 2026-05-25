# 字段级权限与列可见性同步机制分析

## 一、核心概念与数据模型

### 1.1 字段权限模型 (Field Permission Model)

字段权限的核心载体是 `enabledFieldIds` —— 一个**白名单数组**，表示当前用户有权限访问的字段 ID 集合。该值由 `RecordPermissionService` 扩展点提供。

```typescript
// record-permission.service.ts:4-14
export type IWrapViewQuery = {
  keepPrimaryKey?: boolean;  // 是否强制保留主键字段
  viewId?: string;
};

export type IRecordReadQuerySource = {
  tableName: string;
  cteName: string;
  cteSql: string;
  enabledFieldIds?: string[];  // 字段权限白名单
};
```

`RecordPermissionService` 是一个**基础服务类**，实际的权限计算逻辑由业务扩展实现：
- `wrapView()` - 返回权限包装后的查询构建器和 `enabledFieldIds`
- `getReadQuerySource()` - 返回读取查询源信息

---

## 二、两条读取链路的完整流程

### 2.1 链路一：仅依赖字段白名单的查询处理（v1 链路）

**入口**：`record.service.ts: getRecords()` (line 1033)

#### 完整流程

```
┌──────────────────────────────────────────────────────────────────┐
│ 阶段 1: 查询记录 ID（仅用于排序、分页、过滤）                       │
└──────────────────────────────────────────────────────────────────┘
getRecords()
    ↓
getDocIdsByQuery()
    ↓
prepareQuery()
    ├─ recordPermissionService.wrapView()  → 获取 enabledFieldIds
    │     (record.service.ts:729-736)
    ├─ sanitizeFilterByEnabledFields()     → 移除过滤条件中无权限字段
    │     (record.service.ts:741)
    └─ getNecessaryFieldMap()              → 构建字段映射（考虑权限）
          (record.service.ts:519-543)
    ↓
buildFilterSortQuery()
    ├─ projectionIds = fieldMap.values ∩ enabledFieldIds
    │     (record.service.ts:837-841)
    └─ SQL SELECT 仅包含 projectionIds 字段
    ↓
返回记录 ID 列表 (queryResult.ids)

┌──────────────────────────────────────────────────────────────────┐
│ 阶段 2: 计算返回字段投影 (projection)                              │
└──────────────────────────────────────────────────────────────────┘
projection =
    ├─ ① 用户显式指定 query.projection → 使用用户指定
    │     (record.service.ts:1056-1057)
    └─ ② 否则调用 getViewProjection()
          ├─ 检测 columnMeta 中有 visible 属性 → visible 模式（白名单）
          ├─ 检测 columnMeta 中有 hidden 属性 → hidden 模式（黑名单）
          └─ 都没有 → 返回 undefined（不限制）
          (record.service.ts:978-1031)

┌──────────────────────────────────────────────────────────────────┐
│ 阶段 3: 获取实际字段数据                                          │
└──────────────────────────────────────────────────────────────────┘
getSnapshotBulkWithPermission(recordIds, projection)
    ├─ recordPermissionService.wrapView(keepPrimaryKey: true)
    │     → 再次获取 enabledFieldIds
    │     (record.service.ts:1907-1913)
    └─ finalProjection =
          ├─ ① 如果有 projection（来自用户或视图）→ 直接使用
          │     ⚠️  此时 enabledFieldIds 被忽略！
          │     (record.service.ts:1915-1917)
          └─ ② 否则 convertEnabledFieldIdsToProjection(enabledFieldIds)
    ↓
getSnapshotBulkInner() → SQL SELECT 仅包含 finalProjection 字段
```

#### 关键代码位置

| 步骤 | 文件 | 行号 |
|------|------|------|
| 入口 | `record.service.ts` | 1033 |
| 查询 ID | `record.service.ts` | 1038-1054 |
| 计算投影 | `record.service.ts` | 1056-1058 |
| 获取数据 | `record.service.ts` | 1060-1067 |
| getViewProjection | `record.service.ts` | 978-1031 |
| getSnapshotBulkWithPermission | `record.service.ts` | 1898-1926 |

> **⚠️ 重要发现**：v1 链路中，如果指定了 `viewId` 或用户提供了 `projection`，则 `enabledFieldIds` 在**返回字段时被完全忽略**。只有当两者都没有时，才会使用权限白名单过滤返回字段。

---

### 2.2 链路二：携带视图与权限信息的读取上下文（v2 链路）

**入口**：`record-open-api-v2.service.ts: getRecords()` (line 208)

#### 完整流程

```
┌──────────────────────────────────────────────────────────────────┐
│ 阶段 1: 构建权限上下文                                            │
└──────────────────────────────────────────────────────────────────┘
getRecords()
    ↓
createV2ReadContext()
    └─ recordPermissionService.getReadQuerySource()
       → enabledFieldIds 注入到 context.recordReadQuerySource
       (record-open-api-v2.service.ts:497-518)

┌──────────────────────────────────────────────────────────────────┐
│ 阶段 2: 预过滤排序和分组字段                                      │
└──────────────────────────────────────────────────────────────────┘
sanitizeReadableSortAndGroup(query, enabledFieldIds)
    ├─ orderBy = orderBy.filter(item => enabledFieldIdSet.has(item.fieldId))
    └─ groupBy = groupBy.filter(item => enabledFieldIdSet.has(item.fieldId))
    (record-open-api-v2.service.ts:520-539)

┌──────────────────────────────────────────────────────────────────┐
│ 阶段 3: 计算返回字段投影 (snapshotProjection)                     │
└──────────────────────────────────────────────────────────────────┘
resolveSnapshotProjection()
    ├─ ① 用户显式指定 query.projection → 使用用户指定
    │     (record-open-api-v2.service.ts:421-426)
    ├─ ② 否则如果有 enabledFieldIds
    │     ├─ fieldKeyType === Id → toProjectionMap(enabledFieldIds)
    │     └─ 其他 → 查询字段元数据转换键类型
    │     ⚠️  此时视图可见性被忽略！
    │     (record-open-api-v2.service.ts:428-446)
    └─ ③ 否则（没有 enabledFieldIds）
          └─ getFieldsByQuery(viewId, filterHidden: true)
             → 只返回视图可见字段
             (record-open-api-v2.service.ts:448-470)

┌──────────────────────────────────────────────────────────────────┐
│ 阶段 4: 查询记录 ID（v2 领域层处理）                               │
└──────────────────────────────────────────────────────────────────┘
executeListRecordsEndpoint() → ListTableRecordsHandler.handle()
    ├─ getEnabledFieldIdSet(context) → 从 context 获取权限
    ├─ sanitizeFilterByEnabledFieldIds() → 过滤查询条件
    │     (ListTableRecordsHandler.ts:229-264)
    ├─ resolveSortValues() → 过滤排序字段
    │     (ListTableRecordsHandler.ts:286-327)
    ├─ 计算 searchVisibleFieldIds = 
    │     query.viewId
    │       ? filterFieldIdsByEnabledFieldIds(
    │           getOrderedVisibleFieldIds(viewId),  // 视图可见字段
    │           enabledFieldIds                    // 权限字段
    │         )
    │       : filterFieldIdsByEnabledFieldIds(table.fieldIds(), enabledFieldIds)
    │     (ListTableRecordsHandler.ts:482-492)
    └─ tableRecordQueryRepository.find() → 返回记录 ID 列表

┌──────────────────────────────────────────────────────────────────┐
│ 阶段 5: 获取实际字段数据                                          │
└──────────────────────────────────────────────────────────────────┘
getSnapshotBulkWithPermission(recordIds, snapshotProjection)
    └─ （与 v1 相同，但此时 snapshotProjection 已按 v2 策略计算）
```

#### 关键代码位置

| 步骤 | 文件 | 行号 |
|------|------|------|
| 入口 | `record-open-api-v2.service.ts` | 208 |
| 构建上下文 | `record-open-api-v2.service.ts` | 221, 497-518 |
| 过滤排序分组 | `record-open-api-v2.service.ts` | 227-230, 520-539 |
| 计算投影 | `record-open-api-v2.service.ts` | 233-238, 415-470 |
| 查询记录 ID | `record-open-api-v2.service.ts` | 255-282 |
| 获取数据 | `record-open-api-v2.service.ts` | 290-297 |
| v2 Handler | `ListTableRecordsHandler.ts` | 410-539 |

> **⚠️ 重要发现**：v2 链路中，如果有 `enabledFieldIds`，返回字段**完全基于权限白名单，不考虑视图可见性**。只有当没有权限白名单时，才会使用视图可见性过滤返回字段。
>
> 但**搜索字段**是个例外：`searchVisibleFieldIds` 始终是 `视图可见字段 ∩ 权限字段` 的交集。

---

## 三、不同视图在展示侧的可见性分支

### 3.1 核心判断逻辑

视图可见性有**三套独立的判断逻辑**，分别用于不同场景：

#### 场景 A：API 层返回字段列表（考虑强制显示字段）
**判断函数**：`isNotHiddenField()` (is-not-hidden-field.ts:9-45)

这是**唯一考虑强制显示字段**的判断逻辑，用于 `GET /tables/{tableId}/fields` 接口。

```typescript
export const isNotHiddenField = (fieldId: string, view) => {
  const { type: viewType, columnMeta, options } = view;

  // Kanban 视图：stackFieldId、coverFieldId 强制显示
  if (viewType === ViewType.Kanban) {
    const { stackFieldId, coverFieldId } = options as IKanbanViewOptions;
    return (
      [stackFieldId, coverFieldId].includes(fieldId) ||
      columnMeta[fieldId]?.visible !== false
    );
  }

  // Gallery 视图：coverFieldId 强制显示
  if (viewType === ViewType.Gallery) {
    const { coverFieldId } = options as IGalleryViewOptions;
    return fieldId === coverFieldId || columnMeta[fieldId]?.visible !== false;
  }

  // Calendar 视图：日期字段、标题字段、颜色配置字段强制显示
  if (viewType === ViewType.Calendar) {
    const { startDateFieldId, endDateFieldId, titleFieldId, colorConfig } = options as ICalendarViewOptions;
    return (
      (colorConfig?.type === ColorConfigType.Field && colorConfig.fieldId === fieldId) ||
      [startDateFieldId, endDateFieldId, titleFieldId].includes(fieldId) ||
      columnMeta[fieldId]?.visible !== false
    );
  }

  // Form 视图：必须显式标记 visible: true
  if (viewType === ViewType.Form) {
    return Boolean(columnMeta[fieldId]?.visible);
  }

  // Grid 等其他视图：默认可见，hidden: true 才隐藏
  return !columnMeta[fieldId]?.hidden;
};
```

#### 场景 B：v1 链路返回数据投影（不考虑强制显示字段）
**判断函数**：`getViewProjection()` (record.service.ts:978-1031)

根据 `columnMeta` 中实际存在的属性自动判断模式：

```typescript
const useVisible = Object.values(columnMeta).some(column => 'visible' in column);
const useHidden = Object.values(columnMeta).some(column => 'hidden' in column);

if (useVisible) {
  if (column.visible) acc[fieldKey] = true;  // 白名单模式
} else if (useHidden) {
  if (!column.hidden) acc[fieldKey] = true;  // 黑名单模式
}
```

#### 场景 C：v2 链路搜索字段可见性（不考虑强制显示字段）
**判断函数**：`isFieldVisible()` (getOrderedVisibleFieldIds.ts:14-21)

根据视图类型硬编码判断：

```typescript
function isFieldVisible(meta, viewType) {
  // Form, Kanban, Gallery, Calendar, Plugin → 使用 visible 属性
  if (['form', 'kanban', 'gallery', 'calendar', 'plugin'].includes(viewType)) {
    return meta?.visible === true;  // 白名单模式
  }
  // Grid → 使用 hidden 属性
  return meta?.hidden !== true;     // 黑名单模式
}
```

---

### 3.2 各视图可见性规则汇总

| 视图类型 | 可见性模式 | 默认值 | 强制显示字段 | 使用场景 |
|----------|-----------|--------|-------------|----------|
| **Grid** | `hidden` 黑名单 | 可见 | 无 | 所有场景 |
| **Form** | `visible` 白名单 | 隐藏 | 无 | 所有场景 |
| **Kanban** | `visible` 白名单* | 可见** | `stackFieldId`（分组字段）<br>`coverFieldId`（封面字段） | 仅 API 字段列表 |
| **Gallery** | `visible` 白名单* | 可见** | `coverFieldId`（封面字段） | 仅 API 字段列表 |
| **Calendar** | `visible` 白名单* | 可见** | `startDateFieldId`（开始日期）<br>`endDateFieldId`（结束日期）<br>`titleFieldId`（标题）<br>`colorConfig.fieldId`（颜色字段） | 仅 API 字段列表 |
| **Plugin** | `visible` 白名单 | 隐藏 | 无 | 所有场景 |

> * 注：Kanban/Gallery/Calendar 在 v1 `getViewProjection()` 和 v2 `isFieldVisible()` 中使用 `visible` 白名单模式，但 `isNotHiddenField()` 中使用 `visible !== false`（默认可见）。这是不一致的。
>
> ** 注：仅 `isNotHiddenField()` 中默认可见，其他判断逻辑中默认隐藏。

---

### 3.3 强制显示字段规则说明

强制显示字段**仅在 API 层返回字段列表**时生效（`isNotHiddenField()`），在**查询过滤和数据返回**时不生效。

这意味着：
- 如果某个强制显示字段在 `columnMeta` 中被标记为隐藏
  - ✓ `GET /fields` 接口仍然会返回该字段（因为强制显示）
  - ✗ `GET /records` 接口返回的数据中**不包含**该字段
  - ✗ 搜索时**不会**在该字段中搜索
  - ✗ 不能用该字段作为过滤条件

> **设计不一致**：强制显示字段规则仅应用于字段元数据 API，未同步到记录查询和数据返回。

---

## 四、查询过滤与展示侧可见性的对齐机制

### 4.1 各层级过滤对比表

| 过滤层级 | v1 链路 | v2 链路 | 对齐情况 |
|---------|---------|---------|---------|
| **查询条件过滤** | ✓ `sanitizeFilterByEnabledFields()`<br>按 `enabledFieldIds` 过滤 | ✓ `sanitizeFilterByEnabledFieldIds()`<br>按 `enabledFieldIds` 过滤 | ✓ 两者对齐，都基于权限 |
| **排序字段过滤** | ✓ SQL `projectionIds`<br>按 `enabledFieldIds` 过滤 | ✓ `resolveSortValues()`<br>按 `enabledFieldIds` 过滤 | ✓ 两者对齐，都基于权限 |
| **搜索字段过滤** | ✓ `getSearchFields()`<br>先过滤视图隐藏字段<br>再按 `enabledFieldIds` 过滤 | ✓ `searchVisibleFieldIds`<br>`视图可见字段 ∩ 权限字段` | ✓ 两者对齐，都是交集 |
| **返回字段过滤** | ✗ 有 viewId/projection 时<br>**忽略 `enabledFieldIds`**<br>仅使用视图可见性 | ✗ 有 `enabledFieldIds` 时<br>**忽略视图可见性**<br>仅使用权限白名单 | ✗ 两者策略相反，不对齐 |
| **强制显示字段** | ✗ 不考虑 | ✗ 不考虑 | ✗ 都不考虑，仅字段列表 API 考虑 |

---

### 4.2 返回字段过滤策略对比

```
v1 链路返回字段策略：

用户指定 projection?
    ├─ 是 → 使用用户 projection
    └─ 否 → 指定了 viewId?
              ├─ 是 → 使用 getViewProjection() → 视图可见性
              └─ 否 → 使用 enabledFieldIds → 权限白名单

v2 链路返回字段策略：

用户指定 projection?
    ├─ 是 → 使用用户 projection
    └─ 否 → 有 enabledFieldIds?
              ├─ 是 → 使用 enabledFieldIds → 权限白名单
              └─ 否 → 使用视图可见字段
```

**结论**：v1 和 v2 的返回字段策略**完全相反**：
- v1：视图优先，权限兜底
- v2：权限优先，视图兜底

---

### 4.3 搜索字段的特殊处理（交集策略）

搜索字段在两条链路中都使用**交集策略**，这是唯一一致的地方：

```typescript
// v2 ListTableRecordsHandler.ts:482-492
const searchVisibleFieldIds =
  query.viewId && !query.ignoreViewQuery
    ? filterFieldIdsByEnabledFieldIds(
        yield* table.getOrderedVisibleFieldIds(query.viewId),  // 视图可见
        enabledFieldIds                                      // 权限允许
      )
    : filterFieldIdsByEnabledFieldIds(table.fieldIds(), enabledFieldIds);
```

```typescript
// v1 record.service.ts:2088-2103
if (viewColumnMeta) {
  Object.entries(viewColumnMeta).forEach(([key, value]) => {
    if (get(value, ['hidden'])) {
      delete fieldInstanceMap[key];  // 先移除视图隐藏
    }
  });
}

if (projection?.length) {
  Object.keys(fieldInstanceMap).forEach((fieldId) => {
    if (!projection.includes(fieldId)) {
      delete fieldInstanceMap[fieldId];  // 再按权限过滤
    }
  });
}
```

---

## 五、核心代码位置索引

| 功能 | 文件路径 | 关键行 |
|------|----------|--------|
| **权限模型** | | |
| RecordPermissionService 定义 | `apps/nestjs-backend/src/features/record/record-permission.service.ts` | 1-35 |
| enabledFieldIds 注入 v2 上下文 | `apps/nestjs-backend/src/features/record/open-api/record-open-api-v2.service.ts` | 497-518 |
| **v1 链路** | | |
| v1 getRecords 入口 | `apps/nestjs-backend/src/features/record/record.service.ts` | 1033-1073 |
| v1 计算 projection | `apps/nestjs-backend/src/features/record/record.service.ts` | 1056-1058 |
| getViewProjection | `apps/nestjs-backend/src/features/record/record.service.ts` | 978-1031 |
| getSnapshotBulkWithPermission | `apps/nestjs-backend/src/features/record/record.service.ts` | 1898-1926 |
| v1 搜索字段过滤 | `apps/nestjs-backend/src/features/record/record.service.ts` | 2088-2103 |
| **v2 链路** | | |
| v2 getRecords 入口 | `apps/nestjs-backend/src/features/record/open-api/record-open-api-v2.service.ts` | 208-310 |
| resolveSnapshotProjection | `apps/nestjs-backend/src/features/record/open-api/record-open-api-v2.service.ts` | 415-470 |
| sanitizeReadableSortAndGroup | `apps/nestjs-backend/src/features/record/open-api/record-open-api-v2.service.ts` | 520-539 |
| ListTableRecordsHandler | `packages/v2/core/src/queries/ListTableRecordsHandler.ts` | 410-539 |
| 权限过滤查询条件 | `packages/v2/core/src/queries/ListTableRecordsHandler.ts` | 229-264 |
| 权限过滤排序字段 | `packages/v2/core/src/queries/ListTableRecordsHandler.ts` | 286-327 |
| 搜索字段交集计算 | `packages/v2/core/src/queries/ListTableRecordsHandler.ts` | 482-492 |
| **视图可见性** | | |
| isNotHiddenField（强制显示） | `apps/nestjs-backend/src/utils/is-not-hidden-field.ts` | 9-45 |
| isFieldVisible（v2 搜索） | `packages/v2/core/src/domain/table/methods/getOrderedVisibleFieldIds.ts` | 14-21 |
| getOrderedVisibleFieldIds | `packages/v2/core/src/domain/table/methods/getOrderedVisibleFieldIds.ts` | 34-104 |
| filterFieldsByView | `apps/nestjs-backend/src/features/field/fields-utils.ts` | 48-70 |
| ViewColumnMeta 定义 | `packages/v2/core/src/domain/table/views/ViewColumnMeta.ts` | 12-36 |

---

## 六、设计特点与不一致性总结

### 6.1 设计优点

1. **权限多层防御**：查询条件、排序、搜索在两条链路中都基于权限过滤，避免了"先查询后过滤"的性能和安全问题。

2. **搜索字段交集策略**：搜索字段同时考虑视图可见性和权限，符合用户预期——用户不会在隐藏字段中搜索。

3. **强制显示字段**：Kanban/Gallery/Calendar 等视图保证功能必需字段始终在字段列表中可见，避免视图配置损坏。

### 6.2 已知不一致性

| 问题 | 影响 | 建议 |
|------|------|------|
| **v1/v2 返回字段策略相反** | 相同权限下，v1 和 v2 API 返回不同的字段集合 | 统一为交集策略：`视图可见 ∩ 权限允许` |
| **强制显示字段仅应用于字段列表 API** | 强制显示字段在数据查询时可能不返回，导致前端展示异常 | 将强制显示字段规则同步到返回字段投影计算 |
| **三套可见性判断逻辑** | 不同场景可能得出不同的可见性结论 | 统一为单一判断函数，在所有场景复用 |
| **v1 有 viewId 时忽略权限** | 可能导致权限泄露——无权限字段通过视图可见性返回 | v1 也应采用交集策略 |
| **v2 有 enabledFieldIds 时忽略视图** | 用户隐藏的字段仍然返回，不符合用户预期 | v2 也应采用交集策略 |

### 6.3 理想的同步机制

建议统一为以下策略：

```
最终返回字段 = (用户指定投影) ∪ (视图可见字段 ∩ 权限允许字段 ∪ 强制显示字段)
查询条件字段 ∈ 权限允许字段
排序字段 ∈ 权限允许字段
搜索字段 ∈ (视图可见字段 ∩ 权限允许字段)
```

这样可以确保：
1. 权限始终是硬约束，不会被绕过
2. 视图可见性是用户偏好，与权限取交集
3. 强制显示字段作为补充，确保视图功能正常
4. 所有层级对齐，避免不一致

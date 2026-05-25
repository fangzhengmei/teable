# 字段级权限与列可见性同步机制分析

## 声明与边界

本文档基于代码事实进行分析，将内容分为三类：
- ✅ **已确认事实**：代码中明确实现的逻辑，可直接引用代码行验证
- 🔍 **推断结论**：基于代码逻辑可以确定的行为，但未在单一位置明确写出
- ❓ **待验证假设**：依赖于 `RecordPermissionService` 具体实现的行为，需结合业务代码确认
- ⚠️ **风险判断**：基于代码逻辑分析出的潜在问题，需结合实际场景评估

---

## 一、核心概念与数据模型

### 1.1 字段权限模型 (Field Permission Model)

✅ **已确认事实**：
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

✅ **已确认事实**：
`RecordPermissionService` 是一个**基础服务类**，实际的权限计算逻辑由业务扩展实现：
- `wrapView()` - 返回权限包装后的查询构建器和 `enabledFieldIds`
- `getReadQuerySource()` - 返回读取查询源信息

---

### 1.2 RecordPermissionService 的默认实现与装配关系

✅ **已确认事实**：
当前仓库中 `RecordPermissionService` 的**默认实现**（`apps/nestjs-backend/src/features/record/record-permission.service.ts:16-35`）是空实现：

```typescript
@Injectable()
export class RecordPermissionService {
  async getReadQuerySource(
    _tableId: string,
    _query?: IWrapViewQuery
  ): Promise<IRecordReadQuerySource | undefined> {
    return undefined;  // 默认返回 undefined
  }

  async wrapView(
    _tableId: string,
    builder: Knex.QueryBuilder,
    _query?: IWrapViewQuery
  ): Promise<{ viewCte?: string; builder: Knex.QueryBuilder; enabledFieldIds?: string[] }> {
    return {
      viewCte: undefined,
      builder,  // 直接返回原 builder，不做任何修改
    };  // 默认不返回 enabledFieldIds
  }
}
```

✅ **已确认事实**：
**装配关系**（按 app/module）：
1. **定义**：`RecordPermissionService` 在 `record.service.ts:132` 中被注入
2. **注册**：在 `record.module.ts:20` 中注册为 provider
3. **导出**：在 `record.module.ts:22` 中导出，供外部模块覆盖
4. **覆盖入口**：`global.module.ts:93` 注释明确说明该服务用于被覆盖：
   ```typescript
   // for overriding the default TablePermissionService, FieldPermissionService,
   // RecordPermissionService, and ViewPermissionService
   ```

✅ **已确认事实**：
默认运行态下（不扩展权限服务）：
- `enabledFieldIds` 始终为 `undefined`
- `wrapView()` 直接返回原 query builder，不做任何修改
- `getReadQuerySource()` 返回 `undefined`
- 所有权限过滤逻辑被旁路，字段可见性完全由视图配置控制

❓ **待验证假设**：
- `enabledFieldIds` 是否包含主键字段？
- `keepPrimaryKey: true` 是否会强制在 `enabledFieldIds` 中添加主键？
- 什么情况下 `enabledFieldIds` 为 `undefined`？
- CTE（Common Table Expression）是否会在 SQL 层面进一步限制字段访问？
- 业务扩展实现中，`wrapView()` 是否会传入 `viewId` 用于计算 `enabledFieldIds`？

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

✅ **已确认事实**：
v1 链路中，如果指定了 `viewId` 或用户提供了 `projection`，则 `enabledFieldIds` 在**返回字段时被完全忽略**。只有当两者都没有时，才会使用权限白名单过滤返回字段。

---

### 2.3 v1 搜索链路中 ignoreViewQuery 的两段路径处理差异

✅ **已确认事实**：
v1 搜索链路分为两段，`ignoreViewQuery` 在两段中的处理**不一致**：

#### 段 A：主查询搜索（buildFilterSortQuery 中）
**位置**：`record.service.ts:895-906`

```typescript
if (search && search[2] && fieldMap) {
  const searchFields = await this.getSearchFields(
    fieldMap,
    search,
    query?.viewId,       // ⚠️  直接传递 viewId，未考虑 ignoreViewQuery
    enabledFieldIds
  );
  // ... 搜索条件构建
}
```

**行为**：无论 `ignoreViewQuery` 是否为 `true`，都直接传递 `query.viewId` 给 `getSearchFields()`。`getSearchFields()` 会根据 `viewId` 读取视图的 `columnMeta` 并过滤 `hidden` 字段。

#### 段 B：搜索命中索引（getSearchHitIndex 中）
**位置**：`record.service.ts:2170-2199`

```typescript
private async getSearchHitIndex(tableId, query, builder, enabledFieldIds) {
  const { search, viewId, projection, ignoreViewQuery } = query;
  // ...
  const searchFields = await this.getSearchFields(
    fieldInstanceMap,
    search,
    ignoreViewQuery ? undefined : viewId,  // ✅ 正确处理 ignoreViewQuery
    projection
  );
}
```

**行为**：正确处理 `ignoreViewQuery`，当 `ignoreViewQuery: true` 时传递 `undefined` 作为 `viewId`，跳过视图过滤。

#### 调用链对比

```
主查询搜索（段 A）：
getRecords() → getDocIdsByQuery() → buildFilterSortQuery()
  → getSearchFields(fieldMap, search, query.viewId, enabledFieldIds)
    ⚠️  未考虑 ignoreViewQuery，始终使用 viewId

搜索命中索引（段 B）：
getDocIdsByQuery() → getSearchHitIndex()
  → getSearchFields(..., ignoreViewQuery ? undefined : viewId, ...)
    ✅ 正确处理 ignoreViewQuery
```

⚠️ **风险判断**：
当 `ignoreViewQuery: true` 时，两段搜索逻辑不一致：
- 段 A：仍然按视图隐藏字段过滤搜索范围
- 段 B：不按视图隐藏字段过滤搜索范围

这可能导致：
1. 主查询搜索不到的内容，在搜索命中索引中可以搜到
2. 或反之，搜索结果不一致

---

### 2.4 链路二：携带视图与权限信息的读取上下文（v2 链路）

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
             → 使用 isNotHiddenField 判断（考虑强制显示字段）
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

✅ **已确认事实**：
v2 链路中，如果有 `enabledFieldIds`，返回字段**完全基于权限白名单，不考虑视图可见性**。只有当没有权限白名单时，才会使用视图可见性过滤返回字段。

✅ **已确认事实**：
v2 链路中，如果没有 `enabledFieldIds`，返回字段投影会通过 `getFieldsByQuery(viewId, filterHidden: true)` 调用 `isNotHiddenField()`，**考虑强制显示字段规则**。

---

## 三、搜索字段过滤的交集与非交集分析

### 3.1 v1 链路搜索字段过滤

✅ **已确认事实**：
v1 搜索字段过滤在两处调用：
1. 主查询中：`buildFilterSortQuery()` → `getSearchFields(fieldMap, search, viewId, enabledFieldIds)` (line 896-901)
2. 搜索命中索引中：`getSearchHitIndex()` → `getSearchFields(fieldInstanceMap, search, viewId, projection)` (line 2194-2199)

**核心逻辑**：
```typescript
// record.service.ts:2071-2168
async getSearchFields(originFieldInstanceMap, search?, viewId?, projection?) {
  const fieldInstanceMap = { ...originFieldInstanceMap };
  
  // 步骤 1: 按视图隐藏字段过滤（只检查 hidden，不检查 visible）
  if (viewId) {
    const viewColumnMeta = ...;
    if (viewColumnMeta) {
      Object.entries(viewColumnMeta).forEach(([key, value]) => {
        if (get(value, ['hidden'])) {
          delete fieldInstanceMap[key];  // ⚠️ 只检查 hidden 属性
        }
      });
    }
  }
  
  // 步骤 2: 按权限投影过滤
  if (projection?.length) {
    Object.keys(fieldInstanceMap).forEach((fieldId) => {
      if (!projection.includes(fieldId)) {
        delete fieldInstanceMap[fieldId];
      }
    });
  }
  
  // 步骤 3: 再次过滤（防御性编程）
  return Object.values(fieldInstanceMap)
    .filter((field) => {
      if (!viewColumnMeta) return true;
      return !viewColumnMeta?.[field.id]?.hidden;  // ⚠️ 再次只检查 hidden
    })
    .filter((field) => {
      if (!projection) return true;
      return projection.includes(field.id);
    })
    .filter((field) => {
      // 过滤不可搜索的字段类型
      if (field.type === FieldType.Button) return false;
      if (field.cellValueType === CellValueType.Boolean) return false;
      if (isSearchAllFields && field.cellValueType === CellValueType.DateTime) return false;
      // ...
    });
}
```

---

### 3.2 v2 链路搜索字段过滤

✅ **已确认事实**：
v2 搜索字段过滤在 `ListTableRecordsHandler.handle()` 中计算：
```typescript
// ListTableRecordsHandler.ts:482-492
const searchVisibleFieldIds =
  query.viewId && !query.ignoreViewQuery
    ? filterFieldIdsByEnabledFieldIds(
        yield* table.getOrderedVisibleFieldIds(query.viewId),  // 视图可见字段
        enabledFieldIds                                      // 权限字段
      )
    : filterFieldIdsByEnabledFieldIds(table.fieldIds(), enabledFieldIds);
```

✅ **已确认事实**：
`getOrderedVisibleFieldIds()` 内部使用 `isFieldVisible()` 判断：
```typescript
// getOrderedVisibleFieldIds.ts:14-21
function isFieldVisible(meta, viewType) {
  // Form, Kanban, Gallery, Calendar, Plugin → 使用 visible 属性
  if (['form', 'kanban', 'gallery', 'calendar', 'plugin'].includes(viewType)) {
    return meta?.visible === true;  // ✅ 白名单模式，必须显式标记 visible
  }
  // Grid → 使用 hidden 属性
  return meta?.hidden !== true;     // ✅ 黑名单模式，默认可见
}
```

---

### 3.3 交集与非交集对比

| 过滤维度 | v1 链路 | v2 链路 | 交集/非交集 |
|---------|---------|---------|------------|
| **权限过滤** | ✓ `projection` 参数即 `enabledFieldIds`，用于过滤 | ✓ `filterFieldIdsByEnabledFieldIds()` 取交集 | ✅ 交集：都基于 `enabledFieldIds` 过滤 |
| **视图 `hidden` 过滤** | ✓ 检查 `columnMeta[fieldId].hidden` | ✓ `isFieldVisible()` 中 Grid 视图检查 `hidden` | ✅ 交集：都检查 `hidden` |
| **视图 `visible` 过滤** | ❌ 完全不检查 `visible` 属性 | ✓ `isFieldVisible()` 中非 Grid 视图检查 `visible` | ❌ 非交集：v1 遗漏 |
| **字段类型过滤** | ✓ 过滤 Button、Boolean、DateTime 等不可搜索字段 | ❌ 代码中未见字段类型过滤（可能在更底层） | ❌ 非交集：v2 缺失（或位置不同） |
| **强制显示字段** | ❌ 不考虑 | ❌ 不考虑 | ✅ 交集：都不考虑 |

⚠️ **风险判断**：
在 Form/Kanban/Gallery/Calendar 等使用 `visible` 白名单模式的视图中，v1 的搜索字段过滤**只检查 `hidden` 不检查 `visible`**。如果某个字段在 `columnMeta` 中没有 `hidden: true` 但也没有 `visible: true`（符合这些视图的默认隐藏策略），v1 搜索时**仍然会在该字段中搜索**，可能导致信息泄露。

🔍 **推断结论**：
v1 的 `getSearchFields()` 设计初衷可能只针对 Grid 视图，没有考虑使用 `visible` 白名单的视图类型。

---

## 四、不同视图在展示侧的可见性分支

### 4.1 三套独立的可见性判断逻辑

✅ **已确认事实**：
视图可见性有**三套独立的判断逻辑**，分别用于不同场景：

| 判断函数 | 使用场景 | 强制显示字段 | 可见性模式判断 |
|---------|---------|-------------|---------------|
| `isNotHiddenField()` | API 字段列表<br>v2 无权限时的返回投影 | ✓ 考虑 | 根据视图类型硬编码 |
| `getViewProjection()` | v1 返回数据投影 | ✗ 不考虑 | 根据 `columnMeta` 实际存在的属性自动检测 |
| `isFieldVisible()` | v2 搜索字段可见性<br>v2 有序可见字段 | ✗ 不考虑 | 根据视图类型硬编码 |

---

### 4.2 场景 A：`isNotHiddenField()`（考虑强制显示字段）

✅ **已确认事实**：
这是**唯一考虑强制显示字段**的判断逻辑，用于：
1. `GET /tables/{tableId}/fields` 接口（`field.service.ts:878`）
2. v2 链路中无 `enabledFieldIds` 时的返回投影计算

```typescript
// is-not-hidden-field.ts:9-45
export const isNotHiddenField = (fieldId: string, view) => {
  const { type: viewType, columnMeta, options } = view;

  // Kanban 视图：stackFieldId、coverFieldId 强制显示
  if (viewType === ViewType.Kanban) {
    const { stackFieldId, coverFieldId } = options as IKanbanViewOptions;
    return (
      [stackFieldId, coverFieldId].includes(fieldId) ||
      columnMeta[fieldId]?.visible !== false  // ⚠️ 默认可见
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

---

### 4.3 场景 B：`getViewProjection()`（不考虑强制显示字段）

✅ **已确认事实**：
用于 v1 链路返回数据投影，**不考虑强制显示字段**：
```typescript
// record.service.ts:978-1031
private async getViewProjection(tableId, query) {
  const columnMeta = ...;
  
  // 根据 columnMeta 中实际存在的属性自动判断模式
  const useVisible = Object.values(columnMeta).some(column => 'visible' in column);
  const useHidden = Object.values(columnMeta).some(column => 'hidden' in column);

  const projection = Object.entries(columnMeta).reduce((acc, [fieldId, column]) => {
    if (useVisible) {
      if ('visible' in column && column.visible) {  // 必须有 visible 且为 true
        acc[fieldKey] = true;
      }
    } else if (useHidden) {
      if (!('hidden' in column) || !column.hidden) {  // 没有 hidden 或为 false
        acc[fieldKey] = true;
      }
    } else {
      acc[fieldKey] = true;
    }
  }, {});
}
```

🔍 **推断结论**：
`getViewProjection()` 的 `useVisible` 判断存在歧义：只要有一个字段有 `visible` 属性，就对所有字段使用 `visible` 模式。但 `isNotHiddenField()` 中 Kanban/Gallery/Calendar 视图使用 `visible !== false`（默认可见），而 `getViewProjection()` 使用 `visible === true`（必须显式标记）。这会导致不一致。

---

### 4.4 场景 C：`isFieldVisible()`（不考虑强制显示字段）

✅ **已确认事实**：
用于 v2 链路搜索字段和有序可见字段，**不考虑强制显示字段**：
```typescript
// getOrderedVisibleFieldIds.ts:14-21
function isFieldVisible(meta, viewType) {
  if (['form', 'kanban', 'gallery', 'calendar', 'plugin'].includes(viewType)) {
    return meta?.visible === true;  // ✅ 必须显式标记 visible
  }
  return meta?.hidden !== true;     // ✅ 默认可见
}
```

---

### 4.5 各视图可见性规则汇总

| 视图类型 | 可见性模式 | `isNotHiddenField()` | `getViewProjection()` | `isFieldVisible()` |
|----------|-----------|---------------------|----------------------|--------------------|
| **Grid** | `hidden` 黑名单 | 默认可见 | 默认可见 | 默认可见 |
| **Form** | `visible` 白名单 | 必须 `visible: true` | 必须 `visible: true`* | 必须 `visible: true` |
| **Kanban** | `visible` 白名单* | `visible !== false`（默认可见）<br>+ 强制显示字段 | 必须 `visible: true`* | 必须 `visible: true` |
| **Gallery** | `visible` 白名单* | `visible !== false`（默认可见）<br>+ 强制显示字段 | 必须 `visible: true`* | 必须 `visible: true` |
| **Calendar** | `visible` 白名单* | `visible !== false`（默认可见）<br>+ 强制显示字段 | 必须 `visible: true`* | 必须 `visible: true` |
| **Plugin** | `visible` 白名单 | 无专门处理（默认可见） | 必须 `visible: true`* | 必须 `visible: true` |

> * 注：`getViewProjection()` 根据 `columnMeta` 实际属性自动判断模式，不是根据视图类型硬编码。

⚠️ **不一致性**：
`isNotHiddenField()` 中 Kanban/Gallery/Calendar 使用 `visible !== false`（默认可见），但其他两个判断使用 `visible === true`（默认隐藏）。这意味着在字段列表 API 中这些视图的字段默认可见，但在数据查询和搜索时默认隐藏。

---

### 4.6 强制显示字段规则对读取投影路径的影响

✅ **已确认事实**：
强制显示字段**仅在以下场景**影响读取投影路径：
1. **v2 链路 + 无 `enabledFieldIds`**：`resolveSnapshotProjection()` → `getFieldsByQuery(viewId, filterHidden: true)` → `isNotHiddenField()`

✅ **已确认事实**：
强制显示字段**在以下场景不生效**：
1. v1 链路所有场景
2. v2 链路有 `enabledFieldIds` 的场景
3. 搜索字段过滤（两条链路都不考虑）
4. 查询条件过滤（两条链路都不考虑）
5. 排序字段过滤（两条链路都不考虑）

🔍 **推断结论**：
当 v2 链路中没有 `enabledFieldIds`（即未启用字段级权限）时，返回字段会考虑强制显示字段。这是为了确保视图功能正常（例如 Kanban 视图必须能看到分组字段）。

⚠️ **风险判断**：
如果启用了字段级权限（有 `enabledFieldIds`），强制显示字段规则**完全失效**。如果 Kanban 视图的 `stackFieldId` 不在 `enabledFieldIds` 中，v2 链路返回的数据不包含该字段，但字段列表 API 仍然返回该字段，可能导致前端渲染异常。

---

## 五、显式 projection、权限过滤、视图可见性的兜底关系证据链

### 5.1 v1 链路兜底关系证据链

✅ **已确认事实**：
v1 链路返回字段投影的兜底关系（按优先级从高到低）：

```
优先级 1: 用户显式指定 query.projection
    ↓ 证据链：
    record.service.ts:1056-1058
    └─ const projection = query.projection
           ? this.convertProjection(query.projection)
           : await this.getViewProjection(tableId, query);

优先级 2: 视图可见性（viewId 存在时）
    ↓ 证据链：
    record.service.ts:978-1031 (getViewProjection)
    └─ 根据 columnMeta 中的 visible/hidden 属性计算返回字段

优先级 3: 权限白名单（兜底）
    ↓ 证据链：
    record.service.ts:1915-1917 (getSnapshotBulkWithPermission)
    └─ finalProjection = projection ?? convertEnabledFieldIdsToProjection(...)
```

**完整决策树**：
```
query.projection 存在吗?
    ├─ 是 → 使用 query.projection （最高优先级，直接使用，不与任何其他过滤取交集）
    │    证据：record.service.ts:1056-1057
    └─ 否 → query.viewId 存在吗?
              ├─ 是 → 使用 getViewProjection() （次高优先级，直接使用，不与权限取交集）
              │    证据：record.service.ts:1058, 1915-1917
              └─ 否 → 使用 enabledFieldIds （兜底，只有当两者都不存在时才使用权限）
                   证据：record.service.ts:1915-1917
```

⚠️ **关键发现**：v1 链路中，`projection` 和 `viewId` 的处理是**互斥且直接使用**，不与 `enabledFieldIds` 取交集。这意味着：
- 如果用户指定了 `projection`，即使其中包含无权限字段，也会被返回
- 如果指定了 `viewId`，即使视图可见字段包含无权限字段，也会被返回
- 只有当两者都不存在时，权限白名单才生效

---

### 5.2 v2 链路兜底关系证据链

✅ **已确认事实**：
v2 链路返回字段投影的兜底关系（按优先级从高到低）：

```
优先级 1: 用户显式指定 query.projection
    ↓ 证据链：
    record-open-api-v2.service.ts:421-426
    └─ const explicitProjection = this.toProjectionMap(query.projection)
       if (explicitProjection) return explicitProjection;

优先级 2: 权限白名单（enabledFieldIds 存在时）
    ↓ 证据链：
    record-open-api-v2.service.ts:428-446
    └─ if (enabledFieldIds?.length) {
           if (fieldKeyType === Id) return toProjectionMap(enabledFieldIds)
           else 按键类型转换后返回
       }

优先级 3: 视图可见性 + 强制显示字段（兜底）
    ↓ 证据链：
    record-open-api-v2.service.ts:448-470
    └─ if (ignoreViewQuery || !viewId) return undefined
       else getFieldsByQuery(viewId, filterHidden: true)
           → isNotHiddenField() → 考虑强制显示字段
```

**完整决策树**：
```
query.projection 存在吗?
    ├─ 是 → 使用 query.projection （最高优先级，直接使用）
    │    证据：record-open-api-v2.service.ts:421-426
    └─ 否 → enabledFieldIds 存在且非空?
              ├─ 是 → 使用 enabledFieldIds （次高优先级，直接使用，不与视图取交集）
              │    证据：record-open-api-v2.service.ts:428-446
              └─ 否 → viewId 存在且 !ignoreViewQuery?
                        ├─ 是 → 使用 isNotHiddenField()（考虑强制显示字段）
                        │    证据：record-open-api-v2.service.ts:448-470
                        └─ 否 → 不限制（返回所有字段）
```

⚠️ **关键发现**：v2 链路中，`enabledFieldIds` 的优先级高于视图可见性，且同样**不与视图取交集**。这意味着：
- 如果启用了字段级权限，即使用户在视图中隐藏了某些字段，这些字段仍然会被返回
- 只有当权限白名单不存在时，视图可见性才生效

---

### 5.3 两条链路兜底关系对比

| 优先级 | v1 链路 | v2 链路 | 证据位置 |
|--------|---------|---------|---------|
| 1 | 用户显式 projection | 用户显式 projection | v1: record.service.ts:1056-1057<br>v2: record-open-api-v2.service.ts:421-426 |
| 2 | 视图可见性 | 权限白名单 | v1: record.service.ts:1058<br>v2: record-open-api-v2.service.ts:428-446 |
| 3 | 权限白名单（兜底） | 视图可见性 + 强制显示字段（兜底） | v1: record.service.ts:1915-1917<br>v2: record-open-api-v2.service.ts:448-470 |
| 是否取交集 | ❌ 都不取交集，互斥使用 | ❌ 都不取交集，互斥使用 | v1: record.service.ts:1915-1917<br>v2: record-open-api-v2.service.ts:421-470 |

🔍 **推断结论**：
两条链路都遵循**单一来源原则**而非**交集原则**：每次只从一个来源获取返回字段列表，而不是对多个来源取交集。这是导致 v1 和 v2 行为不一致的根本原因。

⚠️ **风险判断**：
两条链路都存在"单一来源"带来的安全风险：
- v1：视图可见字段可能绕过权限检查
- v2：权限白名单可能绕过用户的视图隐藏偏好

理想情况下，返回字段应该是 `(用户 projection) ∪ (视图可见字段 ∩ 权限白名单 ∪ 强制显示字段)`。

---

## 六、查询过滤与展示侧可见性的对齐机制

### 6.1 各层级过滤对比表

| 过滤层级 | v1 链路 | v2 链路 | 对齐情况 |
|---------|---------|---------|---------|
| **查询条件过滤** | ✓ `sanitizeFilterByEnabledFields()`<br>按 `enabledFieldIds` 过滤 | ✓ `sanitizeFilterByEnabledFieldIds()`<br>按 `enabledFieldIds` 过滤 | ✅ 对齐 |
| **排序字段过滤** | ✓ SQL `projectionIds`<br>按 `enabledFieldIds` 过滤 | ✓ `resolveSortValues()`<br>按 `enabledFieldIds` 过滤 | ✅ 对齐 |
| **搜索字段过滤** | ✓ 先过滤 `hidden`<br>再按 `enabledFieldIds` 过滤<br>⚠️ 不检查 `visible` | ✓ `视图可见字段 ∩ 权限字段`<br>✅ 同时检查 `hidden` 和 `visible` | ⚠️ 部分对齐：v1 遗漏 `visible` 检查 |
| **返回字段过滤** | ✗ 有 viewId/projection 时<br>**忽略 `enabledFieldIds`**<br>仅使用视图可见性 | ✗ 有 `enabledFieldIds` 时<br>**忽略视图可见性**<br>仅使用权限白名单 | ❌ 策略相反，不对齐 |
| **强制显示字段** | ✗ 不考虑 | △ 仅无 `enabledFieldIds` 时考虑 | ❌ 都不在查询侧考虑 |

---

### 6.2 返回字段过滤策略对比

```
v1 链路返回字段策略（补充兜底关系后）：

用户指定 projection?
    ├─ 是 → 使用用户 projection
    └─ 否 → 指定了 viewId?
              ├─ 是 → 使用 getViewProjection() → 视图可见性
              └─ 否 → 使用 enabledFieldIds → 权限白名单（兜底）

v2 链路返回字段策略（补充兜底关系后）：

用户指定 projection?
    ├─ 是 → 使用用户 projection
    └─ 否 → 有 enabledFieldIds?
              ├─ 是 → 使用 enabledFieldIds → 权限白名单
              │     （忽略视图可见性和强制显示字段）
              └─ 否 → getFieldsByQuery(viewId, filterHidden: true)
                    → isNotHiddenField() → 视图可见性 + 强制显示字段（兜底）
```

✅ **已确认事实**：
v1 和 v2 的返回字段策略**完全相反**：
- v1：视图优先，权限兜底
- v2：权限优先，视图兜底

---

### 6.3 搜索字段的特殊处理（交集策略）

✅ **已确认事实**：
搜索字段在两条链路中都使用**交集策略**，但实现方式不同：

**v2 实现（正确的交集）**：
```typescript
// ListTableRecordsHandler.ts:482-492
const searchVisibleFieldIds =
  query.viewId && !query.ignoreViewQuery
    ? filterFieldIdsByEnabledFieldIds(
        yield* table.getOrderedVisibleFieldIds(query.viewId),  // 视图可见
        enabledFieldIds                                      // 权限允许
      )
    : filterFieldIdsByEnabledFieldIds(table.fieldIds(), enabledFieldIds);
```

**v1 实现（有缺陷的交集）**：
```typescript
// record.service.ts:2097-2111
if (viewColumnMeta) {
  Object.entries(viewColumnMeta).forEach(([key, value]) => {
    if (get(value, ['hidden'])) {
      delete fieldInstanceMap[key];  // ⚠️ 只移除 hidden，不移除没有 visible 的字段
    }
  });
}

if (projection?.length) {
  Object.keys(fieldInstanceMap).forEach((fieldId) => {
    if (!projection.includes(fieldId)) {
      delete fieldInstanceMap[fieldId];  // 按权限过滤
    }
  });
}
```

---

## 七、核心代码位置索引

| 功能 | 文件路径 | 关键行 |
|------|----------|--------|
| **权限模型与装配** | | |
| RecordPermissionService 定义与默认实现 | `apps/nestjs-backend/src/features/record/record-permission.service.ts` | 1-35 |
| RecordPermissionService 注册与导出 | `apps/nestjs-backend/src/features/record/record.module.ts` | 20, 22 |
| 覆盖入口注释 | `apps/nestjs-backend/src/global/global.module.ts` | 93 |
| enabledFieldIds 注入 v2 上下文 | `apps/nestjs-backend/src/features/record/open-api/record-open-api-v2.service.ts` | 497-518 |
| **v1 链路** | | |
| v1 getRecords 入口 | `apps/nestjs-backend/src/features/record/record.service.ts` | 1033-1073 |
| v1 projection 决策 | `apps/nestjs-backend/src/features/record/record.service.ts` | 1056-1058 |
| getViewProjection | `apps/nestjs-backend/src/features/record/record.service.ts` | 978-1031 |
| getSnapshotBulkWithPermission | `apps/nestjs-backend/src/features/record/record.service.ts` | 1898-1926 |
| 主查询搜索（段 A） | `apps/nestjs-backend/src/features/record/record.service.ts` | 895-906 |
| 搜索命中索引（段 B） | `apps/nestjs-backend/src/features/record/record.service.ts` | 2170-2199 |
| getSearchFields | `apps/nestjs-backend/src/features/record/record.service.ts` | 2071-2168 |
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
| getFieldsByQuery | `apps/nestjs-backend/src/features/field/field.service.ts` | 826-887 |
| filterFieldsByView | `apps/nestjs-backend/src/features/field/fields-utils.ts` | 48-70 |
| ViewColumnMeta 定义 | `packages/v2/core/src/domain/table/views/ViewColumnMeta.ts` | 12-36 |

---

## 八、设计特点与不一致性总结

### 8.1 设计优点

✅ **已确认事实**：
1. **权限多层防御**：查询条件、排序在两条链路中都基于权限过滤，避免了"先查询后过滤"的性能和安全问题。
2. **搜索字段交集策略（v2）**：v2 正确实现了搜索字段的交集策略，符合用户预期。
3. **强制显示字段**：Kanban/Gallery/Calendar 等视图保证功能必需字段在字段列表中可见，避免视图配置损坏。

---

### 8.2 已知不一致性

| 问题 | 已确认事实 | 风险等级 |
|------|-----------|----------|
| **v1/v2 返回字段策略相反** | v1 视图优先，v2 权限优先 | 中 |
| **v1 搜索字段不检查 `visible`** | `getSearchFields()` 只检查 `hidden`，不检查 `visible` | 高 |
| **v1 两段搜索 ignoreViewQuery 处理不一致** | 主查询搜索不考虑 ignoreViewQuery，搜索命中索引考虑 | 中 |
| **三套可见性判断逻辑不一致** | Kanban/Gallery/Calendar 在不同判断逻辑中默认可见性不同 | 中 |
| **强制显示字段仅部分场景生效** | 仅 v2 无权限时生效，其他场景失效 | 中 |
| **v1 有 viewId 时忽略权限** | `getViewProjection()` 的结果直接使用，不与权限取交集 | 高 |
| **v2 有 enabledFieldIds 时忽略视图** | 返回字段完全基于权限，不考虑用户隐藏偏好 | 低 |

---

### 8.3 理想的同步机制建议

🔍 **推断结论**：
建议统一为以下策略：

```
最终返回字段 = 
    (用户显式指定 projection) 
    ∪ 
    ((视图可见字段 ∩ 权限允许字段) ∪ 强制显示字段)

查询条件字段 ∈ 权限允许字段
排序字段 ∈ 权限允许字段
搜索字段 ∈ (视图可见字段 ∩ 权限允许字段)
```

这样可以确保：
1. 权限始终是硬约束，不会被绕过
2. 视图可见性是用户偏好，与权限取交集
3. 强制显示字段作为补充，确保视图功能正常
4. 所有层级对齐，避免不一致

---

## 九、风险判断与确定性结论的边界

### 9.1 可以得出的确定性结论

✅ **已确认事实**：
1. `RecordPermissionService` 默认是空实现，当前仓库默认运行态无字段级权限控制
2. `RecordPermissionService` 在 `record.module.ts:20` 注册，`global.module.ts:93` 明确标注为可覆盖
3. v1 有 viewId 时返回字段忽略 `enabledFieldIds`，使用视图可见性
4. v2 有 `enabledFieldIds` 时返回字段忽略视图可见性，使用权限白名单
5. v1 两段搜索链路对 `ignoreViewQuery` 处理不一致
6. v1 搜索字段只检查 `hidden`，不检查 `visible`
7. 两套链路都遵循"单一来源原则"，不做交集运算
8. 三套可见性判断逻辑独立且存在不一致
9. 强制显示字段仅在 v2 无权限时影响返回投影
10. 显式 projection 在两条链路中都具有最高优先级

---

### 9.2 需要结合权限实现验证的假设

❓ **待验证假设**：
1. `enabledFieldIds` 的具体生成逻辑：是否包含主键？是否考虑视图？
2. `keepPrimaryKey: true` 的实际作用：是否强制添加主键到 `enabledFieldIds`？
3. CTE 的具体实现：是否在 SQL 层面有额外的字段过滤？
4. 什么场景下 `enabledFieldIds` 为 `undefined`？
5. 实际业务中是否会同时使用 `visible` 和 `hidden` 属性？
6. 业务扩展实现中，`wrapView()` 是否会传入 `viewId` 用于计算 `enabledFieldIds`？

---

### 9.3 风险判断的边界

⚠️ **风险判断基于以下前提**：
1. 假设 `enabledFieldIds` 确实是权限白名单（即只包含用户有权限的字段）
2. 假设字段级权限是启用的（即 `enabledFieldIds` 不是 `undefined` 也不是全量字段）
3. 假设 `columnMeta` 中同时使用 `visible` 和 `hidden` 属性不会出现在实际业务中
4. 假设视图配置是正确的（即 Form 视图不会出现 `hidden` 属性）
5. 假设用户显式指定的 `projection` 已经过前端或业务层校验

如果以上前提不成立，风险等级可能需要调整。

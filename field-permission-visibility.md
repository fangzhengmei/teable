# 字段级权限与列可见性同步机制分析

## 一、核心概念与数据模型

### 1.1 字段权限模型 (Field Permission Model)

字段权限的核心载体是 `enabledFieldIds` —— 一个**白名单数组**，表示当前用户有权限访问的字段 ID 集合。

#### 数据结构定义

```typescript
// ListTableRecordsHandler.ts:215-221
type IRecordReadQuerySource = {
  enabledFieldIds?: ReadonlyArray<string>;  // 字段权限白名单
  tableName?: string;
  cteName?: string;
  cteSql?: string;
};

type IExecutionContextWithRecordReadQuerySource = IExecutionContext & {
  recordReadQuerySource?: IRecordReadQuerySource;  // 注入到执行上下文
};
```

#### 权限注入链路

权限信息通过 `RecordPermissionService` 计算并注入到执行上下文：

```typescript
// record-open-api-v2.service.ts:497-518
private async createV2ReadContext(tableId: string, query: ...) {
  const context = await this.v2ContextFactory.createContext();
  const readSource = await this.recordPermissionService.getReadQuerySource(tableId, {
    viewId: query.viewId,
    keepPrimaryKey: Boolean(query.filterLinkCellSelected),
  });
  return {
    ...context,
    recordReadQuerySource: {
      tableName: readSource.tableName,
      cteName: readSource.cteName,
      cteSql: readSource.cteSql,
      enabledFieldIds: readSource.enabledFieldIds,  // 注入权限
    },
  } as IExecutionContext;
}
```

> **关键点**：`enabledFieldIds` 由 `RecordPermissionService` 的 `wrapView()` 和 `getReadQuerySource()` 方法提供，这是权限系统的扩展点。

---

### 1.2 视图列裁剪模型 (View Column Trimming)

视图列可见性通过 `ViewColumnMeta` 数据结构控制，每个视图为每个字段维护一份元数据。

#### ViewColumnMeta 数据结构

```typescript
// ViewColumnMeta.ts:12-20
export type ViewColumnMetaEntry = {
  order?: number | null;       // 列排序
  visible?: boolean;           // 是否可见（用于 form/kanban/gallery 等）
  hidden?: boolean;            // 是否隐藏（用于 grid 视图）
  width?: number;              // 列宽
  required?: boolean;          // 是否必填
  statisticFunc?: string | null;  // 统计函数
  [key: string]: unknown;
};

export type ViewColumnMetaValue = Record<string, ViewColumnMetaEntry>;
```

#### 视图类型差异

不同视图类型使用不同的可见性判断逻辑：

```typescript
// getOrderedVisibleFieldIds.ts:14-21
function isFieldVisible(meta: ViewColumnMetaEntry | undefined, viewType: string): boolean {
  // Form, Kanban, Gallery, Calendar, Plugin 视图使用 visible 属性
  if (['form', 'kanban', 'gallery', 'calendar', 'plugin'].includes(viewType)) {
    return meta?.visible === true;  // 白名单模式：必须显式标记可见
  }
  // Grid 视图使用 hidden 属性（默认可见）
  return meta?.hidden !== true;     // 黑名单模式：默认可见，显式标记隐藏才不可见
}
```

#### 视图默认列可见性初始化

新建视图时，`ViewColumnMeta.forView()` 方法会根据视图类型设置默认可见性：

```typescript
// ViewColumnMeta.ts:71-107
static forView(params: { viewType: ViewType; fields: ...; primaryFieldId: FieldId }) {
  // Form 视图：通过 FieldFormVisibilityVisitor 判断字段是否可在表单中显示
  if (viewType === 'form') {
    const visitor = new FieldFormVisibilityVisitor();
    for (const field of params.fields) {
      const visibleResult = field.accept(visitor);
      if (visibleResult.isOk() && visibleResult.value) {
        columnMeta[key] = { ...previous, visible: true };
      }
    }
  }

  // Kanban/Gallery/Calendar 视图：仅主键字段默认可见
  if (['kanban', 'gallery', 'calendar'].includes(viewType)) {
    const key = params.primaryFieldId.toString();
    columnMeta[key] = { ...previous, visible: true };
  }
}
```

---

## 二、查询侧的三层过滤机制

### 2.1 第一层：查询条件过滤 (Filter Sanitization)

在构建查询条件时，会移除所有引用无权限字段的过滤条件。

```typescript
// ListTableRecordsHandler.ts:229-264
const sanitizeFilterByEnabledFieldIds = (
  filter: RecordFilter | undefined,
  enabledFieldIds: ReadonlySet<string> | undefined
): RecordFilter | undefined => {
  const sanitizeNode = (node: RecordFilterNode): RecordFilterNode | undefined => {
    if (isRecordFilterCondition(node)) {
      return enabledFieldIds.has(node.fieldId) ? node : undefined;  // 无权限字段直接移除
    }
    // 递归处理过滤组和 NOT 节点...
  };
  return sanitizeNode(filter);
};
```

**应用位置**：`ListTableRecordsHandler.handle()` 第 459-462 行，在合并视图默认过滤和用户查询过滤后应用。

---

### 2.2 第二层：排序字段过滤 (Sort Sanitization)

排序时会跳过无权限的字段。

```typescript
// ListTableRecordsHandler.ts:286-327
const resolveSortValues = (
  table: Table,
  sort: ReadonlyArray<RecordSortValue> | undefined,
  fieldKeyType: FieldKeyType,
  enabledFieldIds?: ReadonlySet<string>
) => {
  for (const item of sort ?? []) {
    // ... 解析 fieldId ...
    if (enabledFieldIds && !enabledFieldIds.has(normalizedFieldId)) {
      continue;  // 无权限字段跳过
    }
    resolvedSort.push({ fieldId: normalizedFieldId, order: item.order });
  }
};
```

---

### 2.3 第三层：搜索字段过滤 (Search Field Filtering)

搜索时只在有权限且视图可见的字段中进行。

```typescript
// ListTableRecordsHandler.ts:482-492
const searchVisibleFieldIds =
  query.viewId && !query.ignoreViewQuery
    ? filterFieldIdsByEnabledFieldIds(
        yield* table.getOrderedVisibleFieldIds(query.viewId),  // 先取视图可见字段
        enabledFieldIds                                      // 再与权限取交集
      )
    : filterFieldIdsByEnabledFieldIds(table.fieldIds(), enabledFieldIds);

const visibleRowSearch = resolveVisibleRowSearch(
  RecordSearch.fromOptionalTuple(query.search),
  searchVisibleFieldIds
);
```

```typescript
// ListTableRecordsHandler.ts:359-368
const filterFieldIdsByEnabledFieldIds = (
  fieldIds: ReadonlyArray<FieldId>,
  enabledFieldIds: ReadonlySet<string> | undefined
): ReadonlyArray<FieldId> => {
  if (enabledFieldIds == null) return fieldIds;
  return fieldIds.filter((fieldId) => enabledFieldIds.has(fieldId.toString()));
};
```

> **关键点**：`searchVisibleFieldIds` 是 **视图可见字段 ∩ 权限允许字段** 的交集。

---

## 三、API 返回侧的字段裁剪

### 3.1 SQL 查询层面的列裁剪

在构建 SQL 查询时，通过 `projectionFieldIds` 只选择允许的字段。

```typescript
// record.service.ts:837-841
const projectionIds = fieldMap
  ? Array.from(new Set(Object.values(fieldMap).map((f) => f.id))).filter(
      (id) => !enabledFieldIds || enabledFieldIds.includes(id)  // 权限过滤
    )
  : [];
```

### 3.2 搜索字段的双重过滤

`getSearchFields()` 方法同时考虑视图隐藏和权限限制：

```typescript
// record.service.ts:2088-2103
if (viewId) {
  const { columnMeta: viewColumnRawMeta } = await this.prismaService.view.findUnique(...);
  viewColumnMeta = viewColumnRawMeta ? JSON.parse(viewColumnRawMeta) : null;

  if (viewColumnMeta) {
    Object.entries(viewColumnMeta).forEach(([key, value]) => {
      if (get(value, ['hidden'])) {
        delete fieldInstanceMap[key];  // 先移除视图隐藏字段
      }
    });
  }
}

if (projection?.length) {
  Object.keys(fieldInstanceMap).forEach((fieldId) => {
    if (!projection.includes(fieldId)) {
      delete fieldInstanceMap[fieldId];  // 再按权限投影过滤
    }
  });
}
```

---

## 四、三者关系与同步机制

### 4.1 整体数据流

```
权限服务 (RecordPermissionService)
    ↓ wrapView() / getReadQuerySource()
enabledFieldIds (权限白名单)
    ↓ 注入到 IExecutionContext.recordReadQuerySource
查询处理器 (ListTableRecordsHandler)
    ├─→ 步骤1: sanitizeFilterByEnabledFieldIds()  过滤查询条件
    ├─→ 步骤2: resolveSortValues()               过滤排序字段
    ├─→ 步骤3: getOrderedVisibleFieldIds()        获取视图可见字段
    │     └─→ isFieldVisible()                   根据视图类型判断可见性
    ├─→ 步骤4: filterFieldIdsByEnabledFieldIds()  视图可见字段 ∩ 权限字段
    └─→ 步骤5: resolveVisibleRowSearch()          搜索字段过滤
    ↓
数据库查询 (PostgresTableRecordQueryRepository)
    ├─→ queryBuilder.select(projectionFieldIds)  SQL 列裁剪
    └─→ buildRecordSearchWhereClause()           搜索条件构建
    ↓
API 响应
    └─→ FieldKeyResolverService.transformResponseKeys()  字段键转换（无过滤）
```

### 4.2 同步的核心原则

| 机制 | 控制维度 | 生效时机 | 数据来源 |
|------|----------|----------|----------|
| 字段权限 | `enabledFieldIds` 白名单 | 查询构建全阶段 | `RecordPermissionService` |
| 视图列可见性 | `ViewColumnMeta.hidden/visible` | 获取可见字段列表时 | 视图配置（用户可调整） |
| API 返回过滤 | `projectionFieldIds` 裁剪 | SQL 查询和结果序列化 | 前两者的交集 |

### 4.3 关键同步点

#### 同步点 1：查询条件的双重校验

查询条件中的字段必须同时满足：
1. 在 `enabledFieldIds` 权限白名单中
2. 字段存在且有效（由 `resolveFilterFieldKeys` 校验）

#### 同步点 2：搜索字段的交集计算

```
最终搜索字段 = (视图可见字段) ∩ (权限允许字段) ∩ (用户指定搜索字段)
```

代码实现：
```typescript
// 先取视图可见字段，再与权限取交集
filterFieldIdsByEnabledFieldIds(
  yield* table.getOrderedVisibleFieldIds(query.viewId),  // 视图层
  enabledFieldIds                                      // 权限层
)
```

#### 同步点 3：新增字段时的视图同步

新增字段时，`cloneViewsWithField()` 方法会同步更新所有视图的 `columnMeta`：

```typescript
// Table.ts:1230-1301
private cloneViewsWithField(fields: ReadonlyArray<Field>, newField: Field, options?) {
  // Grid 视图：如果已有显式隐藏配置，则新增字段默认为隐藏
  if (view.type().toString() === 'grid' && hasExplicitHiddenVisibilityConfig) {
    nextEntry = { ...defaultEntry, hidden: true };
  }
  // 其他视图：使用默认可见性
}
```

---

## 五、核心代码位置索引

| 功能 | 文件路径 | 关键行 |
|------|----------|--------|
| 字段权限上下文定义 | `packages/v2/core/src/queries/ListTableRecordsHandler.ts` | 215-227 |
| 查询条件权限过滤 | `packages/v2/core/src/queries/ListTableRecordsHandler.ts` | 229-264 |
| 排序字段权限过滤 | `packages/v2/core/src/queries/ListTableRecordsHandler.ts` | 286-327 |
| 可见字段交集计算 | `packages/v2/core/src/queries/ListTableRecordsHandler.ts` | 359-368, 482-492 |
| 视图可见性判断 | `packages/v2/core/src/domain/table/methods/getOrderedVisibleFieldIds.ts` | 14-21 |
| 获取有序可见字段ID | `packages/v2/core/src/domain/table/methods/getOrderedVisibleFieldIds.ts` | 34-103 |
| ViewColumnMeta 定义 | `packages/v2/core/src/domain/table/views/ViewColumnMeta.ts` | 12-36 |
| 视图默认可见性初始化 | `packages/v2/core/src/domain/table/views/ViewColumnMeta.ts` | 71-107 |
| 权限注入上下文 | `apps/nestjs-backend/src/features/record/open-api/record-open-api-v2.service.ts` | 497-518 |
| 搜索字段双重过滤 | `apps/nestjs-backend/src/features/record/record.service.ts` | 2071-2168 |
| 字段同步过滤计划 | `packages/v2/core/src/domain/table/fields/filter-sync.ts` | 54-126 |

---

## 六、设计特点总结

### 6.1 权限与视图解耦设计

- **字段权限** 是**系统级**控制，由权限服务计算，用户无法修改
- **视图可见性** 是**用户级**配置，用户可以在视图中隐藏/显示字段
- 两者通过**取交集**的方式协同，最终可见字段是两者的叠加限制

### 6.2 多层防御的安全设计

权限控制不是在最后一步过滤返回结果，而是在**查询构建的每个阶段**都进行校验：
1. 过滤条件中不能引用无权限字段
2. 排序不能使用无权限字段
3. 搜索不能在无权限字段中进行
4. SQL 查询只选择有权限的列

这种设计避免了"先查询后过滤"带来的性能问题和安全隐患。

### 6.3 视图类型的差异化处理

不同视图类型使用不同的可见性默认值，符合各视图的使用场景：
- **Grid 视图**：默认全部可见，用户按需隐藏（适合数据浏览）
- **Form 视图**：默认全部隐藏，用户按需添加（适合数据录入）
- **Kanban/Gallery 视图**：仅主键默认可见（适合卡片展示）

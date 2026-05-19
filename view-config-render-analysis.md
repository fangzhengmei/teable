# 视图配置从后端定义到前端渲染的完整路径分析

## 概述

本文档分析 Teable 项目中视图配置（View Configuration）从后端定义、持久化、接口暴露到前端组件消费的完整传递路径。重点关注三组核心状态：

- **字段顺序（columnMeta）**：控制列的显示顺序、宽度、可见性等
- **过滤排序（filter + sort）**：控制数据的筛选条件和排序规则
- **分组聚合（group）**：控制数据的分组展示方式

---

## 一、核心数据模型定义（@teable/core）

所有视图配置的类型定义集中在 `packages/core/src/models/view/` 目录下。

### 1.1 字段顺序 - columnMeta

**定义位置**：`packages/core/src/models/view/column-meta.schema.ts`

```typescript
// 基础列元数据
export const columnSchemaBase = z.object({
  order: z.number()  // 排序用的浮点数，列按此值排序
});

// 网格视图列元数据（继承基础）
export const gridColumnSchema = columnSchemaBase.extend({
  width: z.number().optional(),           // 列宽
  hidden: z.boolean().optional(),         // 是否隐藏
  statisticFunc: z.enum(StatisticsFunc).nullable().optional()  // 统计函数
});

// 按视图类型区分的列元数据
// - kanbanColumnSchema: 看板视图
// - galleryColumnSchema: 画廊视图
// - calendarColumnSchema: 日历视图
// - formColumnSchema: 表单视图（含 required 字段）
// - pluginColumnSchema: 插件视图

// 最终结构：Record<fieldId, columnSchema>
export const columnMetaSchema = z.record(z.string().startsWith(IdPrefix.Field), columnSchema);
```

**关键特性**：
- 以 `fieldId` 为 key 的 Record 结构
- `order` 是浮点数，支持灵活的重排序（通过插入两个现有值之间实现）
- 不同视图类型有不同的列元数据字段

### 1.2 排序 - sort

**定义位置**：`packages/core/src/models/view/sort/sort.ts`

```typescript
export const sortItemSchema = z.object({
  fieldId: z.string(),    // 字段ID
  order: orderSchema,     // 排序方向：asc / desc
});

export const sortSchema = z.object({
  sortObjs: sortItemSchema.array(),  // 排序对象数组（支持多字段排序）
  manualSort: z.boolean().optional() // 是否手动排序
}).nullable();

// 合并视图默认排序与查询排序
export function mergeWithDefaultSort(
  defaultViewSort?: string | null,
  querySort?: ISortItem[]
): ISortItem[] {
  // ... 实现见下文
}
```

### 1.3 过滤 - filter

**定义位置**：`packages/core/src/models/view/filter/filter.ts`

```typescript
// 支持嵌套的过滤条件
export type IFilterSet = {
  conjunction: IConjunction;  // 逻辑连接符：and / or
  filterSet: (IFilterItem | IFilterSet)[];  // 过滤项或嵌套过滤集
};

export const filterSchema = nestedFilterItemSchema.nullable();

// 合并视图默认过滤与查询过滤
export function mergeWithDefaultFilter(
  defaultViewFilter?: string | null,
  queryFilter?: IFilter
): IFilter | undefined {
  // ... 实现见下文
}
```

**关键特性**：
- 支持嵌套的过滤条件树
- 支持 `and` / `or` 逻辑连接符
- 提供 `mergeWithDefaultFilter` 函数用于合并默认过滤和查询过滤

### 1.4 分组 - group

**定义位置**：`packages/core/src/models/view/group/group.ts`

```typescript
export const groupItemSchema = z.object({
  fieldId: z.string(),    // 分组字段ID
  order: orderSchema,     // 分组排序方向
});

export const groupSchema = groupItemSchema.array().nullable();

// 解析并限制分组数量（最多3层）
export function parseGroup(queryGroup?: IGroup): IGroup | undefined {
  if (queryGroup == null) return;
  const parsedGroup = groupSchema.safeParse(queryGroup);
  return parsedGroup.success ? parsedGroup.data?.slice(0, 3) : undefined;
}
```

### 1.5 视图整体结构

**定义位置**：`packages/core/src/models/view/view.schema.ts`

```typescript
export const viewVoSchema = z.object({
  id: z.string().startsWith(IdPrefix.View),
  name: z.string(),
  type: z.enum(ViewType),    // Grid / Kanban / Gallery / Calendar / Form / Plugin
  description: z.string().optional(),
  order: z.number().optional(),
  options: viewOptionsSchema.optional(),  // 视图特定选项（含 frozenFieldId）
  sort: sortSchema.optional(),
  filter: filterSchema.optional(),
  group: groupSchema.optional(),
  columnMeta: columnMetaSchema,
  // ... 其他元数据字段
});

// 需要JSON序列化存储的字段
export const VIEW_JSON_KEYS = ['options', 'sort', 'filter', 'group', 'shareMeta', 'columnMeta'];
```

### 1.6 网格视图选项（含 frozen 配置）

**定义位置**：`packages/core/src/models/view/derivate/grid-view-option.schema.ts`

```typescript
export const gridViewOptionSchema = z.object({
  rowHeight: z.enum(RowHeightLevel).optional(),
  fieldNameDisplayLines: z.number().min(1).max(3).optional(),
  frozenColumnCount: z.number().min(0).optional(),  // 已废弃
  frozenFieldId: z.string().optional(),  // 冻结到该字段的右侧
}).strict();

export type IGridViewOptions = z.infer<typeof gridViewOptionSchema>;
```

---

## 二、后端持久化层（NestJS Backend）

### 2.1 视图服务 - ViewService

**位置**：`apps/nestjs-backend/src/features/view/view.service.ts`

#### 2.1.1 创建视图

```typescript
async createDbView(tableId: string, viewRo: IViewRo) {
  // ...
  const data: Prisma.ViewCreateInput = {
    id: viewId,
    name,
    type,
    options: options ? JSON.stringify(options) : undefined,  // JSON序列化
    sort: sort ? JSON.stringify(sort) : undefined,          // JSON序列化
    filter: filter ? JSON.stringify(filter) : undefined,    // JSON序列化
    group: group ? JSON.stringify(group) : undefined,       // JSON序列化
    columnMeta: mergedColumnMeta ? JSON.stringify(mergedColumnMeta) : JSON.stringify({}),
    // ...
  };
  return await prisma.view.create({ data });
}
```

**关键点**：
- `options`、`sort`、`filter`、`group`、`columnMeta` 均以 JSON 字符串形式存储在数据库中
- 创建视图时自动生成默认的 `columnMeta`（按字段顺序分配 order）

#### 2.1.2 columnMeta 变更联动 frozenFieldId

**位置**：`apps/nestjs-backend/src/features/view/view.service.ts:484-491`

```typescript
if (type === ViewType.Grid) {
  const originOptions = options ? JSON.parse(options) : {};
  const newOptions = adjustFrozenField(
    originOptions,
    originColumnMeta,
    updateView.columnMeta as IGridColumnMeta
  );

  if (newOptions) {
    values.options = JSON.stringify(newOptions);
  }
}
```

**adjustFrozenField 函数**：
**位置**：`apps/nestjs-backend/src/features/view/utils/derive-frozen-fields.ts`

```typescript
export function adjustFrozenField(
  originOptions: IGridViewOptions,
  originColumnMeta: IGridColumnMeta,
  columnMetaUpdate: IGridColumnMeta
): IGridViewOptions | null {
  const frozenFieldId = originOptions?.frozenFieldId;

  if (!frozenFieldId) return null;
  if (!Object.prototype.hasOwnProperty.call(columnMetaUpdate, frozenFieldId)) return null;

  const frozenColumnUpdate: IGridColumn | undefined = frozenFieldId
    ? columnMetaUpdate[frozenFieldId]
    : undefined;
  const originOrders = Object.keys(originColumnMeta).sort(
    (a, b) => originColumnMeta[a].order - originColumnMeta[b].order
  );

  // 场景1: frozen 字段被删除，将 frozenFieldId 移到前一个字段
  if (frozenColumnUpdate == null) {
    const index = originOrders.indexOf(frozenFieldId);
    const newFrozenId = index > 0 ? originOrders[index - 1] : undefined;
    return {
      ...originOptions,
      frozenFieldId: newFrozenId,
    };
  }

  const oldOrder = originColumnMeta[frozenFieldId]?.order;
  const newOrder = frozenColumnUpdate.order;

  if (oldOrder == null || newOrder == null || newOrder === oldOrder) return null;

  // 场景2: frozen 字段顺序变化，将 frozenFieldId 移到前一个字段
  const oldIndex = originOrders.indexOf(frozenFieldId);
  const prevNeighborId = oldIndex > 0 ? originOrders[oldIndex - 1] : undefined;

  const nextOptions: IGridViewOptions = { ...(originOptions as IGridViewOptions) };
  if (prevNeighborId) {
    nextOptions.frozenFieldId = prevNeighborId;
  } else {
    delete (nextOptions as Record<string, unknown>).frozenFieldId;
  }
  return nextOptions;
}
```

**frozenFieldId 联动逻辑**：
1. 当 frozen 字段被删除时，自动将 frozenFieldId 调整为前一个字段
2. 当 frozen 字段的 order 变化（被拖动）时，自动将 frozenFieldId 调整为前一个字段
3. 如果没有前一个字段，则清除 frozenFieldId

#### 2.1.3 更新视图属性

```typescript
async updateViewSort(tableId: string, viewId: string, sort: ISort) {
  const updateInput: Prisma.ViewUpdateInput = {
    sort: JSON.stringify(sort),  // 序列化后存储
    lastModifiedBy: userId,
    lastModifiedTime: new Date(),
  };
  
  // 创建 OT 操作记录
  const ops = [
    ViewOpBuilder.editor.setViewProperty.build({
      key: 'sort',
      newValue: sort,
      oldValue: viewRaw?.sort ? JSON.parse(viewRaw.sort) : null,
    }),
  ];
  
  // 更新数据库
  await prisma.view.update({ where: { id: viewId }, data: updateInput });
  
  // 保存操作日志（用于实时协作）
  await batchService.saveRawOps(tableId, RawOpType.Edit, IdPrefix.View, [/* ... */]);
}
```

### 2.2 视图工厂 - 从数据库读取

**位置**：`apps/nestjs-backend/src/features/view/model/factory.ts`

```typescript
export function createViewVoByRaw(viewRaw: View): IViewVo {
  return {
    id: viewRaw.id,
    name: viewRaw.name,
    type: viewRaw.type as ViewType,
    options: JSON.parse(viewRaw.options as string) || undefined,  // JSON反序列化
    filter: JSON.parse(viewRaw.filter as string) || undefined,    // JSON反序列化
    sort: JSON.parse(viewRaw.sort as string) || undefined,        // JSON反序列化
    group: JSON.parse(viewRaw.group as string) || undefined,      // JSON反序列化
    columnMeta: JSON.parse(viewRaw.columnMeta as string) || undefined,  // JSON反序列化
    // ...
  };
}
```

---

## 三、后端查询层 - 视图配置转化为SQL

### 3.1 记录查询准备 - prepareQuery

**位置**：`apps/nestjs-backend/src/features/record/record.service.ts:708-765`

这是后端处理 viewId 查询时合并默认配置、权限裁剪的核心入口。

```typescript
async prepareQuery(
  tableId: string,
  query: Pick<IGetRecordsRo, 'viewId' | 'orderBy' | 'groupBy' | 'filter' | 'search' | 'filterLinkCellSelected' | 'ignoreViewQuery'>
) {
  const viewId = query.ignoreViewQuery ? undefined : query.viewId;
  const {
    orderBy: extraOrderBy,
    groupBy: extraGroupBy,
    filter: extraFilter,
    search: originSearch,
  } = query;

  const dbTableName = await this.getDbTableName(tableId);
  
  // 步骤1: 权限服务包装视图，获取 enabledFieldIds（权限裁剪依据）
  const { viewCte, builder, enabledFieldIds } = await this.recordPermissionService.wrapView(
    tableId,
    this.knex.queryBuilder(),
    { viewId: query.viewId, keepPrimaryKey: Boolean(query.filterLinkCellSelected) }
  );

  // 步骤2: 获取视图配置（filter/sort/group）
  const view = await this.getTinyView(tableId, viewId);

  // 步骤3: 合并默认 filter 与查询 filter
  const mergedFilter = mergeWithDefaultFilter(view?.filter, extraFilter);
  
  // 步骤4: 按权限裁剪过滤条件
  const filter = await this.sanitizeFilterByEnabledFields(tableId, mergedFilter, enabledFieldIds);
  
  // 步骤5: 合并默认 sort 与查询 sort
  const orderBy = mergeWithDefaultSort(view?.sort, extraOrderBy);
  
  // 步骤6: 解析并限制分组数量（最多3层）
  const groupBy = parseGroup(extraGroupBy);
  
  // ...
}
```

### 3.2 mergeWithDefaultFilter - 合并过滤条件

**位置**：`packages/core/src/models/view/filter/filter.ts:56-79`

```typescript
export function mergeWithDefaultFilter(
  defaultViewFilter?: string | null,
  queryFilter?: IFilter
): IFilter | undefined {
  if (!defaultViewFilter && !queryFilter) {
    return undefined;
  }

  const parseFilter = filterStringSchema.safeParse(defaultViewFilter);
  const viewFilter = parseFilter.success ? parseFilter.data : undefined;

  let mergeFilter = viewFilter;
  if (queryFilter) {
    if (viewFilter) {
      // 用 and 连接视图默认过滤和查询过滤
      mergeFilter = {
        filterSet: [{ filterSet: [viewFilter, queryFilter], conjunction: 'and' }],
        conjunction: 'and',
      };
    } else {
      mergeFilter = queryFilter;
    }
  }
  return mergeFilter;
}
```

**合并逻辑**：
- 视图默认 filter 和查询 filter 通过 `and` 逻辑连接
- 确保视图级别的过滤条件始终生效，同时叠加查询级别的过滤

### 3.3 mergeWithDefaultSort - 合并排序条件

**位置**：`packages/core/src/models/view/sort/sort.ts:44-73`

```typescript
export function mergeWithDefaultSort(
  defaultViewSort?: string | null,
  querySort?: ISortItem[]
): ISortItem[] {
  if (!defaultViewSort && !querySort) {
    return [];
  }

  const parseSort = sortStringSchema.safeParse(defaultViewSort);
  const viewSort = parseSort.success ? parseSort.data : undefined;

  // 手动排序模式且无查询排序时，返回空数组（使用物理顺序）
  if (viewSort?.manualSort && !querySort?.length) {
    return [];
  }

  const mergeSort = viewSort?.sortObjs || [];

  if (querySort?.length) {
    // 合并同字段ID的排序项，查询排序优先
    const map = new Map(querySort.map((sortItem) => [sortItem.fieldId, sortItem]));
    mergeSort.forEach((sortItem) => {
      !map.has(sortItem.fieldId) && map.set(sortItem.fieldId, sortItem);
    });
    return Array.from(map.values());
  }

  return mergeSort;
}
```

**合并逻辑**：
- 查询排序优先于视图默认排序
- 同字段的排序项，查询排序覆盖视图排序
- 手动排序（manualSort）模式下，无查询排序时使用物理顺序

### 3.4 parseGroup - 解析并限制分组

**位置**：`packages/core/src/models/view/group/group.ts:37-42`

```typescript
export function parseGroup(queryGroup?: IGroup): IGroup | undefined {
  if (queryGroup == null) return;
  const parsedGroup = groupSchema.safeParse(queryGroup);
  return parsedGroup.success ? parsedGroup.data?.slice(0, 3) : undefined;
}
```

**分组上限生效位置**：
- ✅ **核心层（@teable/core）**：`slice(0, 3)` 确保最多只取前3个分组字段
- ✅ **前端 UI 层**：`Group.tsx:48` 中 `limit={3}` 限制用户最多添加3层分组
- ❌ 后端查询时没有额外限制（依赖 parseGroup 的限制）

### 3.5 sanitizeFilterByEnabledFields - 权限裁剪过滤条件

**位置**：`apps/nestjs-backend/src/features/record/record.service.ts:546-609`

```typescript
private async sanitizeFilterByEnabledFields(
  tableId: string,
  filter: IFilter | undefined,
  enabledFieldIds?: string[]
): Promise<IFilter | undefined> {
  if (!filter || !enabledFieldIds?.length) {
    return filter;
  }

  // 构建字段ID映射（支持 id/name/dbFieldName 三种查询方式）
  const fields = await this.dataLoaderService.field.load(tableId);
  const keyToId = new Map<string, string>();
  for (const field of fields) {
    keyToId.set(field.id, field.id);
    keyToId.set(field.name, field.id);
    keyToId.set(field.dbFieldName, field.id);
  }
  const allowed = new Set(enabledFieldIds);

  // 递归遍历过滤条件树，移除无权访问字段的过滤
  const sanitize = (target: IFilter): IFilter | null => {
    if (!target) return null;

    const isFilterGroup = (value: unknown): value is IFilter =>
      !!value && typeof value === 'object' && 'filterSet' in value;
    const isFilterLeaf = (value: unknown): value is IFilterItem =>
      !!value && typeof value === 'object' && 'fieldId' in value;

    const sanitizedSet: NonNullable<IFilter>['filterSet'] = [];
    for (const item of target.filterSet) {
      if (isFilterGroup(item)) {
        const nested = sanitize(item);
        if (nested) sanitizedSet.push(nested);
        continue;
      }
      if (!isFilterLeaf(item)) continue;

      // 检查字段是否在允许列表中
      const candidateId = keyToId.get(item.fieldId) ?? item.fieldId;
      if (!allowed.has(candidateId)) continue;
      
      sanitizedSet.push({ ...item, fieldId: candidateId });
    }

    return sanitizedSet.length === 0 ? null : { ...target, filterSet: sanitizedSet };
  };

  const sanitized = sanitize(filter);
  return sanitized ?? undefined;
}
```

**权限裁剪逻辑**：
1. 获取当前用户有权限访问的字段列表（enabledFieldIds）
2. 递归遍历过滤条件树
3. 移除所有引用无权访问字段的过滤条件
4. 如果某个过滤组（filterSet）内的所有条件都被移除，则移除该组
5. 支持通过字段 id/name/dbFieldName 三种方式匹配字段

---

## 四、前端状态管理层（@teable/sdk）

### 4.1 视图上下文 - ViewProvider

**位置**：`packages/sdk/src/context/view/ViewProvider.tsx`

```typescript
export const ViewProvider: FC<IViewProviderProps> = ({ children, fallback, serverData }) => {
  const { tableId } = useContext(AnchorContext);
  const { instances: views } = useInstances({
    collection: `${IdPrefix.View}_${tableId}`,
    factory: createViewInstance,  // 创建视图模型实例
    initData: serverData,
    queryParams: {},
  });

  const value = useMemo(() => ({ views }), [views]);
  return <ViewContext.Provider value={value}>{children}</ViewContext.Provider>;
};
```

### 4.2 视图模型 - View 类

**位置**：`packages/sdk/src/model/view/view.ts`

```typescript
export abstract class View extends ViewCore {
  protected doc!: Doc<IViewVo>;  // ShareDB 文档实例
  tableId!: string;

  // updateOption 是抽象方法，由各视图类型具体实现
  abstract updateOption(
    option: object
  ): Promise<AxiosResponse<void, any>> | void;

  // 其他更新方法调用后端 API
  async updateFilter(filter: IFilter) {
    return await requestWrap(updateViewFilter)(this.tableId, this.id, { filter });
  }

  async updateSort(sort: ISort) {
    return await requestWrap(updateViewSort)(this.tableId, this.id, { sort });
  }

  async updateGroup(group: IGroup) {
    return await requestWrap(updateViewGroup)(this.tableId, this.id, { group });
  }

  async updateColumnMeta(columnMetaRo: IColumnMetaRo) {
    return await requestWrap(updateViewColumnMeta)(this.tableId, this.id, columnMetaRo);
  }
}
```

> **注意**：`updateOption` 是抽象方法，没有基类实现。具体实现分布在各视图类型中（GridView、KanbanView、GalleryView 等），详见第十章。

### 4.3 useView Hook

**位置**：`packages/sdk/src/hooks/use-view.ts`

```typescript
export function useView(viewId?: string) {
  const { viewId: activeViewId } = useContext(AnchorContext);
  const viewCtx = useContext(ViewContext);
  return viewCtx?.views.find((view) => view.id === (viewId ?? activeViewId));
}
```

### 4.4 useFields Hook - 字段顺序排序的核心

**位置**：`packages/sdk/src/hooks/use-fields.ts`

这是字段顺序按 `columnMeta[field.id].order` 排序的真正位置。

```typescript
export function useFields(options: { withHidden?: boolean; withDenied?: boolean } = {}) {
  const { withHidden, withDenied } = options;
  const { fields: originFields } = useContext(FieldContext);

  const view = useView();
  const { type: viewType, columnMeta } = view ?? {};

  return useMemo(() => {
    // 🔴 核心：按 columnMeta[field.id].order 排序字段
    const sortedFields = sortBy(originFields, (field) => columnMeta?.[field.id]?.order ?? Infinity);

    if ((withHidden && withDenied) || viewType == null) {
      return sortedFields;
    }

    // 过滤隐藏字段和无权限字段
    return sortedFields.filter(({ id, canReadFieldRecord }) => {
      const isHidden = () => {
        if (withHidden) return true;
        // 不同视图类型的可见性逻辑不同
        if (viewType === ViewType.Kanban || viewType === ViewType.Gallery || viewType === ViewType.Calendar) {
          return columnMeta?.[id]?.visible === undefined ? true : columnMeta?.[id]?.visible;
        }
        if (viewType === ViewType.Form) {
          return columnMeta?.[id]?.visible;
        }
        return !columnMeta?.[id]?.hidden;
      };
      const hasPermission = () => {
        if (withDenied) return true;
        return canReadFieldRecord;
      };
      return isHidden() && hasPermission();
    });
  }, [originFields, withHidden, viewType, JSON.stringify(columnMeta)]);
}
```

**字段顺序排序逻辑**：
- ✅ **真正的排序位置**：`use-fields.ts:15` 使用 `sortBy(originFields, (field) => columnMeta?.[field.id]?.order ?? Infinity)`
- 没有 columnMeta 的字段会被排到最后（order 为 Infinity）
- 排序后还会根据视图类型过滤隐藏字段和无权限字段

---

## 五、前端组件消费层

### 5.1 字段顺序消费 - useGridColumns

**位置**：`packages/sdk/src/components/grid-enhancements/hooks/use-grid-columns.tsx`

```typescript
export function useGridColumns(hasMenu?: boolean, hiddenFieldIds?: string[], highlightedFieldId?: string | null) {
  const view = useView() as GridView | undefined;
  const originFields = useFields();  // 🔴 这里已经是按 order 排序后的字段
  
  // 从视图中获取三组状态
  const sort = view?.sort;
  const group = view?.group;
  const filter = view?.filter;
  
  // 提取排序字段ID集合（用于UI高亮）
  const sortFieldIds = useMemo(() => {
    if (!isAutoSort) return;
    return sort.sortObjs.reduce((prev, item) => {
      prev.add(item.fieldId);
      return prev;
    }, new Set<string>());
  }, [sort, isAutoSort]);
  
  // 提取分组字段ID集合（用于UI高亮）
  const groupFieldIds = useMemo(() => {
    if (!group?.length) return;
    return group.reduce((prev, item) => {
      prev.add(item.fieldId);
      return prev;
    }, new Set<string>());
  }, [group]);
  
  // 提取过滤字段ID集合（用于UI高亮）
  const filterFieldIds = useMemo(() => {
    if (filter == null) return;
    return getFilterFieldIds(filter?.filterSet, keyBy(totalFields, 'id'));
  }, [filter, totalFields]);
  
  // 生成列配置
  const generateColumns = useCallback(({ fields, view, ... }) => {
    return fields
      .map((field, i) => {
        const columnMeta = view?.columnMeta[field.id] ?? null;
        const width = columnMeta?.width || GRID_DEFAULT.columnWidth;
        const customTheme = getColumnThemeByField({ /* 根据sort/group/filter高亮 */ });
        
        return {
          id: field.id,
          name: field.name,
          width,
          customTheme,
          // ...
        };
      })
      .filter(Boolean)
      .filter((field) => !view?.columnMeta?.[field?.id]?.hidden);  // 过滤隐藏列
  }, [t]);
  
  return { columns: generateColumns({ /* ... */ }), cellValue2GridDisplay };
}
```

**columnMeta 消费流程**：
1. 调用 `useFields()` 获取已按 order 排序的字段
2. 从 `view.columnMeta` 中获取每个字段的元数据
3. 使用 `width` 设置列宽
4. 使用 `hidden` 过滤隐藏列
5. 根据 `sortFieldIds`、`groupFieldIds`、`filterFieldIds` 高亮对应列

### 5.2 排序组件 - Sort

**位置**：`packages/sdk/src/components/sort/Sort.tsx`

```typescript
function Sort(props: ISortProps) {
  const { children, onChange, sorts: outerSorts } = props;
  const view = useView();
  const [innerSorts, setInnerSorts] = useState(outerSorts);
  
  // 防抖更新
  useDebounce(() => {
    if (isEqual(innerSorts, outerSorts)) return;
    !innerSorts?.manualSort && onChange(innerSorts);
  }, 50, [innerSorts]);
  
  // 手动排序（触发后端重新排序记录）
  const manualSort = async () => {
    if (innerSorts?.sortObjs?.length) {
      const viewRo: IManualSortRo = { sortObjs: innerSorts.sortObjs };
      view && mutateAsync({ view, viewRo });
    }
  };
  
  return (
    <SortBase
      sorts={innerSorts}
      onChange={onChangeInner}
      manualSortOnClick={manualSort}
    >
      {children?.(text, isActive)}
    </SortBase>
  );
}
```

### 5.3 过滤组件 - ViewFilter

**位置**：`packages/sdk/src/components/filter/view-filter/ViewFilter.tsx`

```typescript
export const ViewFilter = (props: IViewFilterProps) => {
  const { filters, children, onChange } = props;
  const [filter, setFilter] = useState(filters);
  
  // 本地编辑版本追踪（防止竞态条件）
  const localEditVersionRef = useRef(0);
  const lastSyncedVersionRef = useRef(0);
  
  // 防抖更新（300ms）
  useDebounce(() => {
    if (!isEqual(filter, filters)) {
      const currentVersion = localEditVersionRef.current;
      onChange(filter);
      lastSyncedVersionRef.current = currentVersion;
    }
  }, 300, [filter]);
  
  return (
    <FilterValidationContext.Provider value={validationErrors}>
      <Popover>
        <BaseViewFilter
          fields={fields}
          value={filter}
          onChange={onChangeHandler}
        />
      </Popover>
    </FilterValidationContext.Provider>
  );
};
```

### 5.4 分组组件 - Group

**位置**：`packages/sdk/src/components/group/Group.tsx`

```typescript
export const Group = (props: IGroupProps) => {
  const { children, onChange, group } = props;
  
  const onChangeInner = (group?: IGroup | null) => {
    onChange?.(group?.length ? group : null);
  };
  
  return (
    <Popover>
      <SortContent
        limit={3}  // 🔴 UI 层限制最多3层分组
        sortValues={group ?? undefined}
        onChange={onChangeInner}
      />
    </Popover>
  );
};
```

### 5.5 分组集合渲染 - useGridGroupCollection

**位置**：`packages/sdk/src/components/grid-enhancements/hooks/use-grid-group-collection.ts`

```typescript
export const useGridGroupCollection = () => {
  const view = useView();
  const group = view?.group;
  const fields = useFields({ withHidden: true, withDenied: true });
  
  // 根据 group 配置获取分组字段
  const groupFields = useMemo(() => {
    if (!group?.length) return [];
    return group
      .map(({ fieldId }) => fields.find((f) => f.id === fieldId))
      .filter(Boolean) as IFieldInstance[];
  }, [fields, group]);
  
  // 生成分组列和单元格渲染函数
  return useMemo(() => ({
    groupColumns: generateGroupColumns(groupFields),
    getGroupCell: generateGroupCellFn(groupFields),
  }), [generateGroupCellFn, groupFields]);
};
```

### 5.6 冻结字段 - FieldMenu

**位置**：`apps/nextjs-app/src/features/app/blocks/view/grid/components/FieldMenu.tsx`

```typescript
const freezeField = async () => {
  const fieldId = fieldIds[0];
  if (!fieldId) return;
  await view?.updateOption({ frozenFieldId: fieldId });
};
```

---

## 六、完整传递路径总结

### 6.1 数据流总览（准确版）

```
前端组件（Filter/Sort/Group/Grid）
    ↓ [packages/sdk/src/model/view/*.view.ts]
调用具体视图类型的 updateFilter/updateSort/updateGroup/updateColumnMeta/updateOption*
    ↓ [packages/sdk/src/utils/requestWrap.ts]
HTTP 请求到后端 API
    ↓ [apps/nestjs-backend/src/features/view/open-api/view-open-api.controller.ts]
ZodValidationPipe 校验 + Permissions('view|update') 鉴权
    ↓ [apps/nestjs-backend/src/features/view/open-api/view-open-api.service.ts:385-446]
setViewProperty / patchViewOptions / updateViewColumnMeta：
  1. findFirstOrThrow({ tableId, id, deletedTime: null })
  2. filter/sort/group validate 分支（仅 setViewProperty）
  3. VIEW_JSON_KEYS 解析 oldValue
  4. ViewOpBuilder 构建 OT 操作
  5. updateViewByOps 事务包裹
  6. 发送 OPERATION_VIEW_UPDATE 事件
    ↓ [apps/nestjs-backend/src/features/view/view.service.ts:439-533]
batchUpdateViewByOps：
  1. 解析 opsMap
  2. 更新数据库
  3. columnMeta 更新时 adjustFrozenField 联动 frozenFieldId
  4. 🔴 batchService.saveRawOps() 持久化 rawOps
    ↓
ShareDB 广播变更到所有在线客户端
    ↓ [packages/sdk/src/context/use-instances/useInstances.ts]
useInstances：
  - op batch 事件触发
  - dispatch({ type: 'update', doc })
  - instanceReducer 重新创建 View 实例
    ↓
useView Hook 获取最新视图数据
    ↓ [packages/sdk/src/hooks/use-fields.ts]
useFields Hook 按 columnMeta.order 重新排序字段
    ↓
消费组件（useGridColumns/Sort/ViewFilter/Group）重新渲染
```

> *注：`updateOption` 是抽象方法，由各视图类型（GridView/KanbanView/GalleryView 等）具体实现，详见第十章。

### 6.2 三组状态的具体路径

#### 字段顺序（columnMeta）

```
数据库存储（JSON字符串）
    ↓
createViewVoByRaw 反序列化 → IColumnMeta
    ↓
ViewProvider → useInstances → View 实例
    ↓
useView() 获取 view.columnMeta
    ↓
🔴 useFields() 中按 columnMeta[field.id].order 排序字段
    ↓
useGridColumns 中：
  - 应用 width 设置列宽
  - 过滤 hidden=true 的列
  - 为列生成 customTheme（如果在 sort/group/filter 中）
```

#### 过滤（filter）

```
数据库存储（JSON字符串）
    ↓
createViewVoByRaw 反序列化 → IFilter
    ↓
ViewProvider → useInstances → View 实例
    ↓
useView() 获取 view.filter
    ↓
① ViewFilter 组件渲染过滤条件
② useGridColumns 中提取 filterFieldIds 用于列高亮
③ 🔴 后端 prepareQuery 中：
   - mergeWithDefaultFilter 合并视图默认 filter 与查询 filter
   - sanitizeFilterByEnabledFields 按权限裁剪过滤条件
   - record-query-builder 中应用到 SQL WHERE 子句
```

#### 排序（sort）

```
数据库存储（JSON字符串）
    ↓
createViewVoByRaw 反序列化 → ISort
    ↓
ViewProvider → useInstances → View 实例
    ↓
useView() 获取 view.sort
    ↓
① Sort 组件渲染排序条件
② useGridColumns 中提取 sortFieldIds 用于列高亮
③ 🔴 后端 prepareQuery 中：
   - mergeWithDefaultSort 合并视图默认 sort 与查询 sort
   - record-query-builder 中应用到 SQL ORDER BY 子句
④ manualSort 时触发记录物理重排
```

#### 分组（group）

```
数据库存储（JSON字符串）
    ↓
createViewVoByRaw 反序列化 → IGroup
    ↓
ViewProvider → useInstances → View 实例
    ↓
useView() 获取 view.group
    ↓
① Group 组件渲染分组条件（UI limit=3）
② useGridColumns 中提取 groupFieldIds 用于列高亮
③ 🔴 后端 prepareQuery 中：
   - parseGroup 解析并 slice(0, 3) 限制最多3层
   - useGridGroupCollection 中生成分组列和渲染函数
```

#### frozenFieldId 联动

```
用户更新 columnMeta（拖动列/删除列）
    ↓ [packages/sdk/src/model/view/view.ts:53-54]
前端 View.updateColumnMeta 发送 API
    ↓ [apps/nestjs-backend/src/features/view/open-api/view-open-api.controller.ts:189-203]
后端 updateViewColumnMeta API
    ↓ [apps/nestjs-backend/src/features/view/open-api/view-open-api.service.ts:240-285]
ViewOpenApiService.updateViewColumnMeta：
  - 构建多个 updateViewColumnMeta op
  - 调用 updateViewByOps
    ↓ [apps/nestjs-backend/src/features/view/view.service.ts:439-533]
🔴 view.service.batchUpdateViewByOps 中：
   - 检查 columnMeta 更新
   - 检查视图类型是否为 Grid
   - 调用 adjustFrozenField：
     * 如果 frozen 字段被删除 → 移到前一个字段
     * 如果 frozen 字段 order 变化 → 移到前一个字段
     * 如果没有前一个字段 → 清除 frozenFieldId
   - 如果 options 变化，追加 setViewProperty op
    ↓
更新 options.frozenFieldId
    ↓
batchService.saveRawOps 持久化所有 ops
    ↓
ShareDB 实时同步到前端
    ↓
Grid 组件重新计算 freezeColumnCount
```

---

## 七、四个关键问题的核实结论

### 7.1 分组上限真正在哪一层生效？

**答案**：**两层限制，核心层是最终保障**

| 层级 | 文件位置 | 限制方式 | 说明 |
|------|---------|---------|------|
| 核心层（@teable/core） | `packages/core/src/models/view/group/group.ts:41` | `slice(0, 3)` | ✅ 最终保障，解析分组时只取前3个 |
| 前端 UI 层 | `packages/sdk/src/components/group/Group.tsx:48` | `limit={3}` | ✅ UI 交互限制，用户最多添加3层 |
| 后端查询层 | - | - | ❌ 无额外限制，依赖 parseGroup |

### 7.2 字段顺序究竟由哪个 hook 按 order 排序？

**答案**：**`useFields` Hook**

- **文件位置**：`packages/sdk/src/hooks/use-fields.ts:15`
- **核心代码**：`sortBy(originFields, (field) => columnMeta?.[field.id]?.order ?? Infinity)`
- **说明**：所有消费字段的组件（包括 `useGridColumns`）都是先调用 `useFields()` 获取已排序的字段列表

### 7.3 viewId 查询在后端如何合并默认 filter sort group 并做权限裁剪？

**答案**：**在 `prepareQuery` 方法中分四步处理**

1. **获取权限上下文**：`recordPermissionService.wrapView()` 返回 `enabledFieldIds`
2. **合并 filter**：`mergeWithDefaultFilter(view?.filter, extraFilter)` 用 `and` 连接视图默认过滤和查询过滤
3. **权限裁剪**：`sanitizeFilterByEnabledFields()` 递归遍历过滤条件树，移除无权访问字段的过滤条件
4. **合并 sort**：`mergeWithDefaultSort(view?.sort, extraOrderBy)` 查询排序优先，同字段覆盖
5. **解析 group**：`parseGroup(extraGroupBy)` 解析并 `slice(0, 3)` 限制

### 7.4 columnMeta 变更如何联动 frozen fields options 与实时同步？

**答案**：**后端 batchUpdateViewByOps 中自动联动，ShareDB 实时同步**

1. **触发时机**：`apps/nestjs-backend/src/features/view/view.service.ts:476-501` 在 `batchUpdateViewByOps` 中检查 `updateViewKeySet.has('columnMeta')` 和视图类型是否为 Grid
2. **联动逻辑**：`adjustFrozenField()` 函数处理两种场景：
   - frozen 字段被删除 → 移到前一个字段
   - frozen 字段 order 变化（拖动）→ 移到前一个字段
3. **op 追加**：如果 options 变化，自动追加 `setViewProperty` op 到 opsMap
4. **实时同步**：所有 ops 通过 `batchService.saveRawOps` 持久化 → ShareDB 广播 → 前端 `useInstances` 自动更新

---

## 八、相关文件索引

| 层级 | 文件路径 | 说明 |
|------|---------|------|
| 核心模型 | `packages/core/src/models/view/view.ts` | 视图抽象基类 |
| 核心模型 | `packages/core/src/models/view/view.schema.ts` | 视图 VO Schema |
| 核心模型 | `packages/core/src/models/view/column-meta.schema.ts` | 列元数据 Schema |
| 核心模型 | `packages/core/src/models/view/sort/sort.ts` | 排序 Schema + mergeWithDefaultSort |
| 核心模型 | `packages/core/src/models/view/filter/filter.ts` | 过滤 Schema + mergeWithDefaultFilter |
| 核心模型 | `packages/core/src/models/view/group/group.ts` | 分组 Schema + parseGroup（含 slice(0,3)） |
| 核心模型 | `packages/core/src/models/view/derivate/grid-view-option.schema.ts` | 网格视图选项（frozenFieldId） |
| 后端服务 | `apps/nestjs-backend/src/features/view/view.service.ts` | 视图 CRUD 服务 + adjustFrozenField 调用 |
| 后端服务 | `apps/nestjs-backend/src/features/view/utils/derive-frozen-fields.ts` | adjustFrozenField 实现 |
| 后端服务 | `apps/nestjs-backend/src/features/view/model/factory.ts` | 视图工厂（DB→VO转换） |
| 后端查询 | `apps/nestjs-backend/src/features/record/record.service.ts` | prepareQuery + sanitizeFilterByEnabledFields |
| 后端查询 | `apps/nestjs-backend/src/features/record/query-builder/record-query-builder.service.ts` | 记录查询构建器 |
| 前端状态 | `packages/sdk/src/context/view/ViewProvider.tsx` | 视图上下文提供者 |
| 前端模型 | `packages/sdk/src/model/view/view.ts` | 前端视图模型 |
| 前端 Hook | `packages/sdk/src/hooks/use-view.ts` | 视图 Hook |
| 前端 Hook | `packages/sdk/src/hooks/use-fields.ts` | 🔴 字段排序 Hook |
| 前端组件 | `packages/sdk/src/components/grid-enhancements/hooks/use-grid-columns.tsx` | 网格列生成 Hook |
| 前端组件 | `packages/sdk/src/components/sort/Sort.tsx` | 排序组件 |
| 前端组件 | `packages/sdk/src/components/filter/view-filter/ViewFilter.tsx` | 过滤组件 |
| 前端组件 | `packages/sdk/src/components/group/Group.tsx` | 🔴 分组组件（UI limit=3） |
| 前端组件 | `packages/sdk/src/components/grid-enhancements/hooks/use-grid-group-collection.ts` | 分组集合 Hook |
| 后端控制器 | `apps/nestjs-backend/src/features/view/open-api/view-open-api.controller.ts` | 视图配置更新 API 路由 |
| 后端服务 | `apps/nestjs-backend/src/features/view/open-api/view-open-api.service.ts` | setViewProperty + updateViewByOps 事务包裹 |
| 后端服务 | `apps/nestjs-backend/src/features/view/view.service.ts` | 🔴 batchUpdateViewByOps（数据库更新 + rawOps 持久化） |
| 后端服务 | `apps/nestjs-backend/src/features/view/utils/derive-frozen-fields.ts` | adjustFrozenField 实现 |
| 前端模型 | `packages/sdk/src/model/view/grid.view.ts` | GridView 的 updateOption 实现 |
| 前端模型 | `packages/sdk/src/model/view/kanban.view.ts` | KanbanView 的 updateOption 实现 |
| 前端模型 | `packages/sdk/src/model/view/gallery.view.ts` | GalleryView 的 updateOption 实现 |
| 前端模型 | `packages/sdk/src/model/view/form.view.ts` | FormView 的 updateOption 实现 |
| 前端模型 | `packages/sdk/src/model/view/calendar.view.ts` | CalendarView 的 updateOption 实现 |
| 前端模型 | `packages/sdk/src/model/view/plugin.view.ts` | PluginView 的 updateOption 实现（抛错） |
| 前端模型 | `packages/sdk/src/model/view/factory.ts` | 视图实例创建工厂 |
| 前端核心 | `packages/sdk/src/context/use-instances/useInstances.ts` | 🔴 实时同步核心 Hook |
| 前端核心 | `packages/sdk/src/context/use-instances/opListener.ts` | OpListenersManager 实现 |
| 前端核心 | `packages/sdk/src/context/use-instances/reducer.ts` | instanceReducer 状态更新 |

---

## 九、view 配置更新接口的完整暴露链路

### 9.1 接口路由一览

**位置**：`apps/nestjs-backend/src/features/view/open-api/view-open-api.controller.ts`

所有视图配置更新接口都在 `ViewOpenApiController` 中，使用 `@Controller('api/table/:tableId/view')` 路由前缀。

| HTTP方法 | 路由路径 | 权限 | 功能 |
|---------|---------|------|------|
| `@Put` | `/:viewId/name` | `view|update` | 更新视图名称 |
| `@Put` | `/:viewId/description` | `view|update` | 更新视图描述 |
| `@Put` | `/:viewId/locked` | `view|update` | 更新视图锁定状态 |
| `@Put` | `/:viewId/share-meta` | `view|update` | 更新视图分享元数据 |
| `@Put` | `/:viewId/column-meta` | `view|update` | 更新列元数据（字段顺序/宽度/可见性） |
| `@Put` | `/:viewId/filter` | `view|update` | 更新过滤条件 |
| `@Put` | `/:viewId/sort` | `view|update` | 更新排序规则 |
| `@Put` | `/:viewId/group` | `view|update` | 更新分组规则 |
| `@Patch` | `/:viewId/options` | `view|update` | 部分更新视图选项（含frozenFieldId） |
| `@Put` | `/:viewId/order` | `view|update` | 更新视图排序 |
| `@Put` | `/:viewId/manual-sort` | `view|update` | 手动排序 |
| `@Put` | `/:viewId/record-order` | `view|update` | 更新记录物理顺序 |

### 9.2 Zod 校验层

每个接口都使用 `@Body(new ZodValidationPipe(schema))` 进行请求体校验：

```typescript
// 示例：更新过滤条件
@Permissions('view|update')
@Put('/:viewId/filter')
async updateViewFilter(
  @Param('tableId') tableId: string,
  @Param('viewId') viewId: string,
  @Body(new ZodValidationPipe(filterRoSchema)) updateViewFilterRo: IFilterRo,
  @Headers('x-window-id') windowId?: string
): Promise<void> {
  return await this.viewOpenApiService.setViewProperty(
    tableId,
    viewId,
    'filter',
    updateViewFilterRo.filter,
    windowId
  );
}
```

### 9.3 setViewProperty 真实实现

**位置**：`apps/nestjs-backend/src/features/view/open-api/view-open-api.service.ts:385-446`

```typescript
async setViewProperty(
  tableId: string,
  viewId: string,
  key: IViewPropertyKeys,
  newValue: unknown,
  windowId?: string
) {
  // 步骤1: 查询当前视图，条件：tableId + viewId + deletedTime: null
  const curView = await this.prismaService.view
    .findFirstOrThrow({
      select: { [key]: true },
      where: { tableId, id: viewId, deletedTime: null },
    })
    .catch(() => {
      throw new CustomHttpException(
        `View not found with id: ${viewId} and tableId: ${tableId}`,
        HttpErrorCode.NOT_FOUND,
        {
          localization: {
          i18nKey: 'httpErrors.view.notFound',
        },
      });
    });

  // 步骤2: filter/sort/group 三段 validate 分支
  if (key === 'filter') {
    await this.validateFilter(tableId, newValue as IFilter);
  }

  if (key === 'sort') {
    await this.validateSort(tableId, newValue as ISort);
  }

  if (key === 'group') {
    await this.validateGroup(tableId, newValue as IGroup);
  }

  // 步骤3: VIEW_JSON_KEYS 解析 oldValue
  // VIEW_JSON_KEYS = ['options', 'sort', 'filter', 'group', 'shareMeta', 'columnMeta']
  const oldValue =
    curView[key] != null && VIEW_JSON_KEYS.includes(key)
      ? JSON.parse(curView[key])
      : curView[key];

  // 步骤4: 构建 OT 操作
  const ops = ViewOpBuilder.editor.setViewProperty.build({
    key,
    newValue,
    oldValue,
  });

  // 步骤5: 应用到数据库
  await this.updateViewByOps(tableId, viewId, [ops]);

  // 步骤6: 发送 byKey 事件负载（如果有 windowId）
  if (windowId) {
    this.eventEmitterService.emitAsync(Events.OPERATION_VIEW_UPDATE, {
      tableId,
      windowId,
      viewId,
      userId: this.cls.get('user.id'),
      byKey: {
        key,
        newValue,
        oldValue,
      },
    });
  }
}
```

**validateFilter 逻辑**：
- 提取过滤条件中的字段ID
- 检查是否包含 Button 类型字段（不支持过滤
- 验证过滤条件兼容性

**validateSort 逻辑**：
- 提取排序条件中的字段ID
- 检查是否包含 Button 类型字段（不支持排序）

**validateGroup 逻辑**：
- 提取分组条件中的字段ID
- 检查是否包含 Button 类型字段（不支持分组）

### 9.4 updateViewByOps 职责边界

**open-api 层的 updateViewByOps**：
**位置**：`apps/nestjs-backend/src/features/view/open-api/view-open-api.service.ts:448-452`

```typescript
async updateViewByOps(tableId: string, viewId: string, ops: IOtOperation[]) {
  return await this.prismaService.$tx(async () => {
    return await this.viewService.updateViewByOps(tableId, viewId, ops);
  });
}
```

**职责**：
- ✅ 仅负责开启数据库事务
- ✅ 转发调用 `viewService.updateViewByOps`
- ❌ **不负责 rawOps 持久化**

**view.service 层的 updateViewByOps**：
**位置**：`apps/nestjs-backend/src/features/view/view.service.ts:435-437`

```typescript
async updateViewByOps(tableId: string, viewId: string, ops: IOtOperation[]) {
  await this.batchUpdateViewByOps(tableId, { [viewId]: ops });
}
```

**batchUpdateViewByOps - rawOps 持久化的真正位置**：
**位置**：`apps/nestjs-backend/src/features/view/view.service.ts:439-533`

```typescript
async batchUpdateViewByOps(tableId: string, opsMap: { [viewId: string]: IOtOperation[] }) {
  // 1. 解析 opsMap，获取需要更新的视图和属性
  const { updateViewMap, updateViewKeySet } = this.getBatchUpdateViewContext(opsMap);
  
  // 2. 查询当前视图数据（含 version, columnMeta, options, type）
  const viewRaws = await this.prismaService.txClient().view.findMany({...});
  
  // 3. 构建更新数据
  const data = viewRaws.map((view) => {
    // columnMeta 更新时联动 frozenFieldId
    if (updateView.columnMeta && type === ViewType.Grid) {
      const newOptions = adjustFrozenField(...);
      if (newOptions) {
        values.options = JSON.stringify(newOptions);
        // 追加 options 更新 op
        opsMap[viewId] = [...(opsMap[viewId] ?? []), newOptionsOp];
      }
    }
    return { id: viewId, values };
  });
  
  // 4. 更新数据库
  if (data.length === 1) {
    await this.prismaService.txClient().view.update({...});
  } else if (data.length > 1) {
    await this.batchUpdateDB(data);
  }
  
  // 🔴 5. rawOps 持久化（发生在 view.service，不是 open-api）
  const opDataList = viewRaws.map((view) => ({
    docId: view.id,
    version: view.version,
    data: opsMap[view.id],
  }));
  
  this.batchService.saveRawOps(tableId, RawOpType.Edit, IdPrefix.View, opDataList);
}
```

**职责边界总结**：

| 层级 | 职责 | rawOps 持久化 |
|------|------|--------------|
| open-api.controller | 路由、权限校验、Zod 校验 | ❌ |
| open-api.service | setViewProperty（validate + 构建 op + 事务包裹）） | ❌ |
| view.service | batchUpdateViewByOps（解析 op、更新数据库、持久化 rawOps | ✅ |

### 9.5 配置更新到实时同步的端到端路径（准确文件与函数名）

```
前端组件（Grid/Sort/Filter/Group）
    ↓ [packages/sdk/src/model/view/*.view.ts]
调用具体视图类型的 updateOption/updateFilter/updateSort/updateGroup/updateColumnMeta
    ↓ [packages/sdk/src/utils/requestWrap.ts]
requestWrap 发送 HTTP PUT/PATCH 请求
    ↓ [apps/nestjs-backend/src/features/view/open-api/view-open-api.controller.ts]
ViewOpenApiController 接收请求：
  - @Permissions('view|update') 权限校验
  - ZodValidationPipe 参数校验
  - 调用 viewOpenApiService.setViewProperty
    ↓ [apps/nestjs-backend/src/features/view/open-api/view-open-api.service.ts:385-446]
ViewOpenApiService.setViewProperty：
  1. prisma.view.findFirstOrThrow({ where: { tableId, id: viewId, deletedTime: null } })
  2. filter/sort/group 三段 validate 分支
  3. VIEW_JSON_KEYS 判断是否 JSON.parse(oldValue)
  4. ViewOpBuilder.editor.setViewProperty.build(...) 构建 OT 操作
  5. 调用 this.updateViewByOps(tableId, viewId, [ops])
  6. 发送 Events.OPERATION_VIEW_UPDATE 事件（byKey 负载）
    ↓ [apps/nestjs-backend/src/features/view/open-api/view-open-api.service.ts:448-452]
ViewOpenApiService.updateViewByOps：
  - prismaService.$tx() 开启事务
  - 调用 viewService.updateViewByOps
    ↓ [apps/nestjs-backend/src/features/view/view.service.ts:435-437]
ViewService.updateViewByOps：
  - 调用 this.batchUpdateViewByOps(tableId, { [viewId]: ops })
    ↓ [apps/nestjs-backend/src/features/view/view.service.ts:439-533]
ViewService.batchUpdateViewByOps：
  1. getBatchUpdateViewContext(opsMap) 解析操作
  2. prisma.txClient().view.findMany() 查询当前视图
  3. 构建更新数据（columnMeta 更新时调用 adjustFrozenField 联动 options
  4. prisma.txClient().view.update() / batchUpdateDB() 更新数据库
  5. 🔴 batchService.saveRawOps() 持久化 rawOps
    ↓ [ShareDB 服务]
ShareDB 监听到 rawOps 变化，广播 op 消息
    ↓ [packages/sdk/src/hooks/use-connection.ts]
前端 ShareDB 连接收到 op 消息
    ↓ [sharedb/lib/client]
ShareDB Doc 对象自动应用 op 到本地数据
    ↓ [packages/sdk/src/context/use-instances/opListener.ts:15]
Doc 触发 'op batch' 事件
    ↓ [packages/sdk/src/context/use-instances/useInstances.ts:608-613]
OpListenersManager 中注册的 handler 被调用：
  - dispatch({ type: 'update', doc })
    ↓ [packages/sdk/src/context/use-instances/reducer.ts:28-41]
instanceReducer 处理 'update' action：
  - factory(action.doc.data, action.doc) 重新创建实例
    ↓ [React]
React 检测到 instances 变化，触发重新渲染
    ↓ [packages/sdk/src/context/view/ViewProvider.tsx]
ViewProvider 中 views 数组更新
    ↓ [packages/sdk/src/hooks/use-view.ts]
useView() 返回最新视图数据
    ↓ [packages/sdk/src/hooks/use-fields.ts]
useFields() 按新的 columnMeta.order 重新排序字段
    ↓ [下游组件]
useGridColumns / Sort / ViewFilter / Group 等组件重新渲染
```

> **注**：更新 `options` 时不走 `setViewProperty`，而是通过单独的 `patchViewOptions` 方法，最终同样调用 `updateViewByOps`。更新 `columnMeta` 时通过 `updateViewColumnMeta` 方法，构建多个 `updateViewColumnMeta` op 而不是 `setViewProperty` op。

---

## 十、updateOption 抽象方法与各视图类型的真实实现

### 10.1 View 基类为 abstract 的事实

**位置**：`packages/sdk/src/model/view/view.ts:32-39`

```typescript
export abstract class View extends ViewCore {
  protected doc!: Doc<IViewVo>;
  tableId!: string;

  abstract updateOption(
    option: object
  ): Promise<AxiosResponse<void, any>> | void;
}
```

**关键事实**：
- ✅ `View` 类声明为 `abstract class`
- ✅ `updateOption` 方法声明为 `abstract`，没有具体实现
- ❌ 之前文档中写的 `async updateOption(options: Partial<IViewOptions>)` 是错误的，基类中没有这个实现

### 10.2 各视图类型的真实实现分布

所有具体视图类型使用 `ts-mixer` 的 `Mixin` 实现多继承，各自实现 `updateOption` 方法：

#### GridView - 网格视图
**位置**：`packages/sdk/src/model/view/grid.view.ts`

```typescript
export class GridView extends Mixin(GridViewCore, View) {
  async updateOption(options: Partial<GridView['options']>) {
    return await requestWrap(updateViewOptions)(this.tableId, this.id, { options });
  }
}
```

#### KanbanView - 看板视图
**位置**：`packages/sdk/src/model/view/kanban.view.ts`

```typescript
export class KanbanView extends Mixin(KanbanViewCore, View) {
  async updateOption({
    stackFieldId, coverFieldId, isCoverFit, isFieldNameHidden, isEmptyStackHidden,
  }: KanbanView['options']) {
    return await requestWrap(updateViewOptions)(this.tableId, this.id, {
      options: { stackFieldId, coverFieldId, isCoverFit, isFieldNameHidden, isEmptyStackHidden },
    });
  }
}
```

#### GalleryView - 画廊视图
**位置**：`packages/sdk/src/model/view/gallery.view.ts`

```typescript
export class GalleryView extends Mixin(GalleryViewCore, View) {
  async updateOption({ coverFieldId, isCoverFit, isFieldNameHidden }: GalleryView['options']) {
    return await requestWrap(updateViewOptions)(this.tableId, this.id, {
      options: { coverFieldId, isCoverFit, isFieldNameHidden },
    });
  }
}
```

#### FormView - 表单视图
**位置**：`packages/sdk/src/model/view/form.view.ts`

```typescript
export class FormView extends Mixin(FormViewCore, View) {
  async updateOption({ coverUrl, logoUrl, submitLabel }: FormView['options']) {
    return await requestWrap(updateViewOptions)(this.tableId, this.id, {
      options: { coverUrl, logoUrl, submitLabel },
    });
  }
}
```

#### CalendarView - 日历视图
**位置**：`packages/sdk/src/model/view/calendar.view.ts`

```typescript
export class CalendarView extends Mixin(CalendarViewCore, View) {
  async updateOption({ titleFieldId, startFieldId, endFieldId, isAllDayFieldId, recurrenceFieldId, colorFieldId }: CalendarView['options']) {
    return await requestWrap(updateViewOptions)(this.tableId, this.id, {
      options: { titleFieldId, startFieldId, endFieldId, isAllDayFieldId, recurrenceFieldId, colorFieldId },
    });
  }
}
```

#### PluginView - 插件视图
**位置**：`packages/sdk/src/model/view/plugin.view.ts`

```typescript
export class PluginView extends Mixin(PluginViewCore, View) {
  async updateOption(_options: unknown): Promise<AxiosResponse<void, unknown>> {
    throw new Error('Plugin view does not support update option');
  }
}
```

### 10.3 视图实例创建工厂

**位置**：`packages/sdk/src/model/view/factory.ts`

```typescript
export function createViewInstance(vo: IViewVo, doc?: Doc<IViewVo>): View {
  switch (vo.type) {
    case ViewType.Grid:
      return new GridView(vo, doc);
    case ViewType.Kanban:
      return new KanbanView(vo, doc);
    case ViewType.Gallery:
      return new GalleryView(vo, doc);
    case ViewType.Calendar:
      return new CalendarView(vo, doc);
    case ViewType.Form:
      return new FormView(vo, doc);
    case ViewType.Plugin:
      return new PluginView(vo, doc);
    default:
      throw new Error(`Unknown view type: ${vo.type}`);
  }
}
```

---

## 十一、实时同步机制深度解析：useInstances 内部工作原理

### 11.1 整体架构

**位置**：`packages/sdk/src/context/use-instances/useInstances.ts`

`useInstances` 是整个前端实时同步的核心 Hook，负责：
- 创建 ShareDB 订阅查询
- 监听文档变更
- 管理实例状态
- 处理 Schema 刷新

### 11.2 createSubscribeQuery - 创建订阅查询

#### 查询缓存机制

```typescript
// 全局缓存，跨 Hook 实例去重相同订阅查询
type CachedQuery = { query: Query; refCount: number };
const subscribeQueryCache = new Map<string, CachedQuery>();

const acquireQuery = <T>(
  collection: string,
  connection: ReturnType<typeof useConnection>['connection'],
  queryParams: unknown,
  refreshToken = 0
) => {
  const key = makeQueryKey(collection, queryParams, refreshToken);
  const cached = subscribeQueryCache.get(key);
  if (cached) {
    cached.refCount += 1;
    return { key, query: cached.query };
  }
  // 创建新的 ShareDB 订阅查询
  const query = connection!.createSubscribeQuery<T>(collection, queryParams);
  subscribeQueryCache.set(key, { query, refCount: 1 });
  return { key, query };
};
```

**查询键生成**：
- 对 queryParams 递归排序键，保证相同参数生成相同 key
- Set 转为排序后的数组
- Map 转为排序后的 entries

#### 查询生命周期

```
useInstances Effect 触发
    ↓
makeQueryKey(collection, queryParams, schemaRefreshToken)
    ↓
acquireQuery()：
  - 查询缓存命中 → refCount++
  - 未命中 → connection.createSubscribeQuery()
    ↓
query.on('ready') → dispatch({ type: 'ready', results, extra })
    ↓
query.on('insert') / 'remove' / 'move' → dispatch 对应 action
    ↓
instanceReducer 更新 instances 数组
```

### 11.3 presence receive - 监听 Schema 变更广播

**位置**：`packages/sdk/src/context/use-instances/useInstances.ts:532-595`

#### Presence 订阅

```typescript
useEffect(() => {
  if (!connection || !schemaRefreshCollectionTableId || !isRecordCollection(collection)) {
    return;
  }

  const presence: Presence = connection.getPresence(
    getActionTriggerChannel(schemaRefreshCollectionTableId)
  );

  if (!presence.subscribed) {
    presence.subscribe((error) => { /* ... */ });
  }

  const receiveListener = (_id: string, batch: unknown) => {
    // 处理批量操作
  };

  presence.addListener('receive', receiveListener);

  return () => {
    presence.removeListener('receive', receiveListener);
    if (presence.listenerCount('receive') === 0) {
      presence.unsubscribe();
      presence.destroy();
    }
  };
}, [/* ... */]);
```

#### receive 事件处理分支

```typescript
const receiveListener = (_id: string, batch: unknown) => {
  // 分支1: 删除记录（skipRealtime=true)
  const deletedRecordIds = getProjectedDeleteRecordIds(schemaRefreshCollectionTableId, batch);
  if (deletedRecordIds?.length) {
    removeProjectedRecordsByIds(deletedRecordIds);
    return;
  }

  // 分支2: 批量修改/新增记录（skipRealtime=true)
  if (
    shouldRefreshAfterProjectedMutation(..., 'setRecord') ||
    shouldRefreshAfterProjectedMutation(..., 'addRecord')
  ) {
    setSchemaRefreshToken((current) => current + 1);
    return;
  }

  // 分支3: Schema 相关字段变更
  if (!isSchemaRefreshAction(schemaRefreshCollectionTableId, batch)) {
    return;
  }

  // 分支4: 字段 Schema 变更，尝试增量刷新
  const fieldIds = getSchemaRefreshRecordFieldIds(schemaRefreshCollectionTableId, batch);
  if (fieldIds?.length) {
    void refreshProjectedRecordFields(fieldIds).then((handled) => {
      if (!handled) {
        setSchemaRefreshToken((current) => current + 1);
      }
    });
    return;
  }

  // 分支5: 其他情况，全量刷新
  setSchemaRefreshToken((current) => current + 1);
};
```

#### Schema 刷新触发字段

```typescript
const schemaRefreshFieldProperties = new Set([
  'type',
  'options',
  'expression',
  'lookupOptions',
  'rollupConfig',
  'linkConfig',
  'linkRelationship',
]);
```

### 11.4 schemaRefreshToken - Schema 刷新令牌

**位置**：`packages/sdk/src/context/use-instances/useInstances.ts:412`

```typescript
const [schemaRefreshToken, setSchemaRefreshToken] = useState(0);
```

**作用**：
- `schemaRefreshToken` 是一个递增的数字状态，用于触发重新创建 ShareDB 查询
- 当 Schema 发生变更时（如字段类型、选项、表达式等变化
- 通过 `setSchemaRefreshToken((current) => current + 1` 递增令牌
- 令牌变化会触发 `useEffect` 重新执行 `acquireQuery`
- 新的查询会携带新的数据，确保前端数据与后端 Schema 一致

**触发场景**：
1. 字段类型变更
2. 字段选项变更
3. 查找/汇总/链接字段配置变更
4. 链接关系变更
5. 批量记录修改/新增（skipRealtime=true)
6. 批量记录删除（skipRealtime=true)
7. 其他需要全量刷新的场景

### 11.5 op batch - 文档操作批量更新

#### OpListenersManager - 操作监听管理器

**位置**：`packages/sdk/src/context/use-instances/opListener.ts`

```typescript
export class OpListenersManager<T> {
  private opListeners: Map<string, () => void> = new Map();

  add(doc: Doc<T>, handler: (op: unknown[]) => void) {
    if (this.opListeners.has(doc.id)) {
      return;
    }
    doc.on('op batch', handler);
    this.opListeners.set(doc.id, () => {
      doc.removeListener('op batch', handler);
      doc.listenerCount('op batch') === 0 && doc.destroy();
    });
  }

  remove(doc: Doc<T>) {
    const cleanupFunction = this.opListeners.get(doc.id);
    cleanupFunction && cleanupFunction();
    this.opListeners.delete(doc.id);
  }

  clear() {
    this.opListeners.forEach((cleanupFunction) => cleanupFunction());
    this.opListeners.clear();
  }
}
```

#### op batch 事件监听

**位置**：`packages/sdk/src/context/use-instances/useInstances.ts:608-613`

```typescript
const handleReady = useCallback((query: Query<T>) => {
  dispatch({ type: 'ready', results: query.results, extra: query.extra });
  query.results.forEach((doc) => {
    opListeners.current.add(doc, (op) => {
      console.log(`${query.collection} on op:', op, doc);
      dispatch({ type: 'update', doc });
    });
  });
}, []);
```

#### instanceReducer - 状态更新

**位置**：`packages/sdk/src/context/use-instances/reducer.ts:22-102`

```typescript
export function instanceReducer<T, R extends { id: string }>(
  state: IInstanceState<R>,
  action: IInstanceAction<T>,
  factory: (data: T, doc?: Doc<T>) => R
): IInstanceState<R> {
  switch (action.type) {
    case 'update': {
      if (!hasDocData(action.doc)) {
        return state;
      }
      return {
        ...state,
        instances: state.instances.map((instance) => {
          if (instance.id === action.doc.id) {
            return factory(action.doc.data, action.doc);
          }
          return instance;
        }),
      };
    }
    // ... 其他 action 类型
  }
}
```

### 11.6 完整实时同步流程图

```
后端保存 rawOps 到数据库
    ↓
ShareDB 服务监听到 rawOps 变化
    ↓
ShareDB 广播 op 消息到所有订阅客户端
    ↓
前端 ShareDB 客户端收到 op 消息
    ↓
ShareDB Doc 对象自动应用 op 到本地数据
    ↓
Doc 触发 'op batch' 事件
    ↓
OpListenersManager 中注册的 handler 被调用
    ↓
dispatch({ type: 'update', doc })
    ↓
instanceReducer 重新创建实例（factory(doc.data, doc)
    ↓
React 重新渲染
```

### 11.7 与视图配置相关的 Schema 刷新流程

```
用户修改视图配置（filter/sort/group/columnMeta/options)
    ↓
HTTP 请求到后端
    ↓
后端更新数据库 + 保存 rawOps
    ↓
ShareDB 广播 op 到所有客户端
    ↓
前端 ShareDB 客户端收到 op
    ↓
View Doc 数据更新
    ↓
op batch 事件触发
    ↓
useInstances dispatch update action
    ↓
instanceReducer 重新创建 View 实例
    ↓
ViewProvider 中 views 数组更新
    ↓
useView() 返回新的视图数据
    ↓
useFields/useGridColumns 等重新计算
    ↓
Grid/Sort/Filter/Group 组件重新渲染
```

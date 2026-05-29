# 看板（Kanban）与画廊（Gallery）视图的状态转换

本文档梳理看板和画廊视图中**字段拖动**、**分组重排**和**持久化保存**三者之间的关联关系。

---

## 1. 数据模型基础

### 1.1 视图核心结构（ViewCore）

所有视图共享 `ViewCore` 基类（`packages/core/src/models/view/view.ts`），包含以下关键属性：

| 属性 | 类型 | 说明 |
|---|---|---|
| `options` | 视图特有 | 每种视图独立的配置项（如看板的 stackFieldId） |
| `columnMeta` | `Record<fieldId, ColumnSchema>` | 每个字段在视图中的元信息，包含 `order`、`visible`/`hidden` 等 |
| `sort` | `ISort` | 视图排序规则 |
| `filter` | `IFilter` | 视图过滤规则 |
| `group` | `IGroup` | 视图分组规则 |

### 1.2 ColumnMeta 差异

不同视图类型的 `columnMeta` 字段结构不同（`packages/core/src/models/view/column-meta.schema.ts`）：

- **Grid** → `{ order, width, hidden, statisticFunc }`
- **Kanban** → `{ order, visible }`
- **Gallery** → `{ order, visible }`
- **Form** → `{ order, visible, required }`

`order` 是浮点数，用于字段在各视图中的排列顺序。

### 1.3 视图 Options 差异

- **Kanban**（`kanban-view-option.schema.ts`）：
  - `stackFieldId`：分组字段 ID，决定看板按哪个字段分列
  - `coverFieldId`、`isCoverFit`、`isFieldNameHidden`、`isEmptyStackHidden`

- **Gallery**（`gallery-view-option.schema.ts`）：
  - `coverFieldId`、`isCoverFit`、`isFieldNameHidden`

---

## 2. 看板视图（Kanban）的拖动与重排

### 2.1 整体架构

```
KanbanView (入口)
  └─ GroupPointProvider (提供 groupPoints 数据)
       └─ KanbanProvider (组装 context: stackField, stackCollection, permission 等)
            └─ KanbanViewBase
                 └─ KanbanContainer (DragDropContext 容器)
                      ├─ Droppable (type="column", 水平方向, 列拖动)
                      │    └─ KanbanStackContainer (Draggable 列)
                      │         └─ KanbanStack (Droppable 卡片列)
                      │              └─ KanbanCard (Draggable 卡片)
                      └─ KanbanStackCreator
```

拖拽库：`@hello-pangea/dnd`，两层 Droppable 嵌套实现列拖动 + 卡片拖动。

### 2.2 三种拖动场景（KanbanContainer.onDragEnd）

`KanbanContainer`（`apps/nextjs-app/src/features/app/blocks/view/kanban/components/KanbanContainer.tsx`）的 `onDragEnd` 根据 `source.droppableId` 区分三种场景：

#### 场景 A：列重排（Stack Reorder）

**触发条件**：`sourceStackId === viewId`（droppableId 等于 viewId 表示是列级别的拖动）

**流程**：
1. 调用 `reorder(stackIds, sourceIndex, targetIndex)` 本地重排 stackIds 数组
2. 仅当 `isSingleSelectField`（单选字段且非 lookup）且位置真正变化时才持久化
3. 将 stackIds 映射回 `stackField.options.choices`，得到新 choices 数组
4. 调用 `stackField.convert({ type, options: { ...options, choices: newChoices } })` 持久化

**关键点**：列的排列顺序**实质上就是单选字段的 choices 顺序**。拖列 = 改字段选项顺序。这通过 `convertField` API 实现，最终触发字段结构变更并通过 OT 同步到所有客户端。

#### 场景 B：同列卡片重排（Card Reorder within Same Stack）

**触发条件**：`sourceStackId === targetStackId` 且不等于 viewId

**流程**：
1. 调用 `updateRecordOrders` API，传入 `{ anchorId, position, recordIds }`
2. 本地调用 `reorder(cards, sourceIndex, targetIndex)` 乐观更新 `cardMap`

**持久化**：`PUT /table/{tableId}/view/{viewId}/record-order`，后端通过 `updateMultipleOrders` 算法计算浮点 order 值写入数据库索引字段。

#### 场景 C：跨列卡片移动（Card Move between Stacks）

**触发条件**：`sourceStackId !== targetStackId`

**流程**：
1. 找到目标 stack，用 `getCellValueByStack(stack)` 算出目标字段值
2. 调用 `updateRecord` API，同时更新：
   - 字段值：`{ [fieldId]: fieldValue }`（改变记录所属的列）
   - 排序位置：`{ viewId, anchorId, position }`（确定在目标列中的位置）
3. 本地调用 `moveTo()` 乐观更新两个 stack 的 cardMap

**持久化**：`PUT /table/{tableId}/record/{recordId}`，后端同时处理字段值更新和记录位置更新。

### 2.3 看板分组数据来源（GroupPoint）

看板的列数据不直接来自视图属性，而是通过 `GroupPointProvider` 动态计算：

```
GroupPointProvider
  └─ useGroupPoint() → GroupPointContext
       └─ KanbanProvider → 组装 stackCollection
```

`GroupPointProvider`（`packages/sdk/src/context/aggregation/GroupPointProvider.tsx`）：
- 对看板视图，取 `options.stackFieldId` 构造 `groupBy` 查询
- 调用 `getGroupPoints` API 获取分组聚合数据
- 每条 Header 类型的 groupPoint 对应一个 stack（列），Row 类型给出计数
- 监听 table 和 view 事件自动刷新

`KanbanProvider` 中的 `stackCollection` 计算逻辑：
- 遍历 groupPoints，提取 Header + Row 组成 `{ id, count, data }`
- 对 `SingleSelect` 字段：用 `options.choices` 顺序排列，未出现的 choice 也保留（count=0）
- 对 `User` 字段：用协作者列表排列
- 其他字段：按 groupPoints 原始顺序
- 始终在开头插入 `UNCATEGORIZED_STACK_ID`（未分类列）
- 若 `isEmptyStackHidden` 为 true，过滤掉 count=0 的列

### 2.4 看板列折叠的本地持久化

列折叠状态（collapsed）通过 zustand + persist 存储在 localStorage：

```
useKanbanStackCollapsedStore (zustand + persist middleware)
  key: LocalStorageKeys.ViewKanbanCollapsedStack
  数据结构: Record<localId, stackId[]>
```

这是**纯客户端状态**，不同步到服务端。

---

## 3. 画廊视图（Gallery）的拖动与重排

### 3.1 整体架构

```
GalleryView (入口)
  └─ RowCountProvider (提供总行数)
       └─ GalleryProvider (组装 context: permission, displayFields, coverField 等)
            └─ GalleryViewBase
                 └─ DndContext (@dnd-kit/core)
                      └─ SortableContext (rectSortingStrategy)
                           └─ SortableItem (useSortable)
                                └─ Card
```

拖拽库：`@dnd-kit/core` + `@dnd-kit/sortable`，单层平铺卡片排序。

### 3.2 卡片拖动（GalleryViewBase.handleDragEnd）

画廊视图**只有一种拖动场景**：卡片重排序。

**流程**：
1. 从 `recordIds` 中找到 `activeId` 和 `overId` 的索引
2. 调用 `updateRecordOrder(oldIndex, newIndex)` 本地乐观更新：
   - `recordIds`：通过 `arrayMove` 重排
   - `loadedRecordMap`：按索引移动记录对象
3. 调用 `updateRecord` API 持久化，请求体包含：
   - `record.fields: {}`（空，不修改字段值）
   - `record.order: { viewId, anchorId, position }`（指定新位置）

**持久化**：`PUT /table/{tableId}/record/{recordId}`，后端同时处理位置更新。

### 3.3 画廊与看板的拖动差异

| 维度 | 看板 | 画廊 |
|---|---|---|
| 拖拽库 | @hello-pangea/dnd | @dnd-kit |
| 拖动层级 | 两层（列 + 卡片） | 单层（仅卡片） |
| 卡片重排 API | `updateRecordOrders`（批量） | `updateRecord`（单条 + order） |
| 列重排 | 有（改字段 choices） | 无 |
| 跨列移动 | 有（改字段值 + order） | 无 |
| 乐观更新 | cardMap (按 stack 分组) | recordIds + loadedRecordMap (扁平) |

---

## 4. 持久化保存机制

### 4.1 三条持久化路径

```
┌─────────────────────────────────────────────────────────────┐
│                      前端拖动操作                            │
│                                                              │
│  看板列重排 ──→ stackField.convert() ──→ convertField API    │
│  看板同列卡片 ──→ updateRecordOrders() ──→ record-order API  │
│  看板跨列移动 ──→ updateRecord() ──────→ record API          │
│  画廊卡片排序 ──→ updateRecord() ──────→ record API          │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                      后端处理                                │
│                                                              │
│  convertField → 字段结构变更 → OT ops → ShareDB 广播         │
│  record-order → updateMultipleOrders → 浮点 order 写入 DB    │
│  record + order → 字段值更新 + 位置更新 → OT ops → ShareDB   │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    实时同步 (ShareDB/OT)                      │
│                                                              │
│  ViewOpBuilder.editor.setViewProperty / updateViewColumnMeta │
│  → 生成 OT operation → ShareDB 推送到所有连接的客户端        │
│  → 前端 instanceReducer 接收 update 事件 → 重建视图实例      │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 浮点 Order 算法

所有排序（视图列排序、记录排序）都使用**浮点数 order**机制：

- 每条记录/视图有一个 `order: number` 字段
- 移动到 anchor 前/后时，新 order = `(anchor.order + next.order) / 2`
- 批量移动时，gap = `|anchor - next| / (length + 1)`，每项 order = `base + gap * index`
- 当 gap 小于 `Number.EPSILON * 2` 时，触发 shuffle（全量重排，重新分配整数 order）

该算法在 `apps/nestjs-backend/src/utils/update-order.ts` 中实现。

### 4.3 OT 协同编辑

视图状态的变更通过 OT（Operational Transformation）实现多端同步：

1. **前端发起**：调用 REST API（如 `updateViewOptions`、`updateRecordOrders`）
2. **后端处理**：生成 `ViewOpBuilder` 操作（如 `setViewProperty`、`updateViewColumnMeta`）
3. **写入 ShareDB**：通过 `viewService.updateViewByOps()` 提交 OT 操作
4. **广播**：ShareDB 将操作推送到所有连接的客户端
5. **前端接收**：`instanceReducer` 处理 `update` 事件，用 `factory` 重建视图实例

### 4.4 SDK 模型层 API 映射

| SDK 方法 | REST API | 用途 |
|---|---|---|
| `view.updateOption()` | `updateViewOptions` | 更新视图 options（如 stackFieldId） |
| `view.updateColumnMeta()` | `updateViewColumnMeta` | 更新字段在视图中的元信息（order、visible 等） |
| `view.updateOrder()` | `updateViewOrder` | 更新视图本身在视图列表中的排序 |
| `view.manualSort()` | `manualSortView` | 手动排序触发全量排序 |
| `stackField.convert()` | `convertField` | 转换字段类型/选项（看板列重排实质调用） |
| `updateRecordOrders()` | `updateRecordOrders` | 批量更新记录排序 |
| `updateRecord()` | `updateRecord` | 更新单条记录（含字段值 + order） |

---

## 5. 三者之间的关联总结

```
字段拖动 (Column Drag)
  │
  │  看板: 改变 columnMeta.order（字段排列）→ updateViewColumnMeta
  │  画廊: 同上（但画廊卡片不直接暴露字段拖动 UI）
  │
  ▼
分组重排 (Group/Stack Reorder)
  │
  │  看板: 列拖动 → 改变 SingleSelect choices 顺序 → convertField
  │        卡片跨列 → 改变记录字段值 + record.order → updateRecord
  │        卡片同列 → 仅改变 record.order → updateRecordOrders
  │  画廊: 卡片排序 → 改变 record.order → updateRecord
  │
  ▼
持久化保存 (Persistence)
  │
  ├─→ REST API 调用（字段/记录/视图变更）
  ├─→ 后端生成 OT 操作
  ├─→ ShareDB 写入 + 广播
  └─→ 前端 reducer 更新本地状态
```

### 核心关联

1. **字段拖动 ↔ 分组重排**：看板的列本质是字段的分组值（groupPoint）。当拖动列改变顺序时，对于 SingleSelect 字段，实际上改变了字段的 `options.choices` 顺序，这会通过 `convertField` 触发字段结构变更，进而导致 groupPoints 重新计算，列顺序随之更新。

2. **分组重排 ↔ 持久化**：
   - 记录排序 → 浮点 order 值写入数据库索引字段
   - 列排序（SingleSelect choices）→ 字段结构持久化
   - 跨列移动 → 记录字段值 + 位置双写

3. **字段拖动 ↔ 持久化**：
   - `columnMeta.order` 变更 → `updateViewColumnMeta` API → OT ops → ShareDB
   - 字段可见性（visible/hidden）→ 同上

### 乐观更新策略

两种视图都采用**乐观更新**：先本地修改状态（cardMap / recordIds / loadedRecordMap），同时发起 API 请求。后端处理完成后，通过 ShareDB OT 推送最终状态，前端 instanceReducer 会用服务端数据覆盖本地。这保证了拖动操作的即时响应性。

---

## 6. 关键文件索引

| 文件 | 职责 |
|---|---|
| `packages/core/src/models/view/view.ts` | ViewCore 基类定义 |
| `packages/core/src/models/view/column-meta.schema.ts` | 各视图类型的 ColumnMeta schema |
| `packages/core/src/models/view/derivate/kanban-view-option.schema.ts` | 看板 Options schema |
| `packages/core/src/models/view/derivate/gallery-view-option.schema.ts` | 画廊 Options schema |
| `packages/sdk/src/model/view/view.ts` | SDK View 基类（API 调用封装） |
| `packages/sdk/src/model/view/kanban.view.ts` | SDK KanbanView（updateOption） |
| `packages/sdk/src/model/view/gallery.view.ts` | SDK GalleryView（updateOption） |
| `packages/sdk/src/hooks/use-record-operations.ts` | 记录操作 hooks（updateRecord, updateRecordOrders 等） |
| `packages/sdk/src/context/aggregation/GroupPointProvider.tsx` | 分组聚合数据 Provider |
| `apps/nextjs-app/.../kanban/components/KanbanContainer.tsx` | 看板拖动核心逻辑（onDragEnd） |
| `apps/nextjs-app/.../kanban/utils/drag.ts` | reorder / moveTo 工具函数 |
| `apps/nextjs-app/.../kanban/utils/filter.ts` | 按 stack 构造 filter 的工具 |
| `apps/nextjs-app/.../kanban/utils/card.ts` | getCellValueByStack 等卡片工具 |
| `apps/nextjs-app/.../kanban/context/KanbanProvider.tsx` | 看板 Context 组装（stackCollection 计算） |
| `apps/nextjs-app/.../kanban/store/useKanbanStackCollapsed.ts` | 列折叠本地持久化（zustand + persist） |
| `apps/nextjs-app/.../gallery/GalleryViewBase.tsx` | 画廊拖动核心逻辑（handleDragEnd） |
| `apps/nextjs-app/.../gallery/hooks/useCacheRecords.ts` | 画廊虚拟滚动记录缓存 + 乐观更新 |
| `apps/nextjs-app/.../gallery/context/GalleryProvider.tsx` | 画廊 Context 组装 |
| `apps/nestjs-backend/.../view-open-api.service.ts` | 后端视图操作服务 |
| `apps/nestjs-backend/src/utils/update-order.ts` | 浮点 order 算法（updateOrder / updateMultipleOrders） |
| `packages/openapi/src/view/update-order.ts` | 视图排序 API 定义 |
| `packages/openapi/src/view/update-record-order.ts` | 记录排序 API 定义 |

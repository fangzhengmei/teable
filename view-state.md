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

## 5. v1/v2 API 路径区分

### 5.1 双轨架构

Teable 采用 v1/v2 双轨架构，通过金丝雀发布（Canary Release）控制切换：

**当前实现**（`apps/nestjs-backend/src/features/view/open-api/view-open-api.controller.ts:284-307`）：

```typescript
@Put('/:viewId/record-order')
@UseV2Feature('reorderRecords')
@UseGuards(V2FeatureGuard)
@UseInterceptors(V2IndicatorInterceptor)
async updateRecordOrders(...) {
  if (this.cls.get('useV2')) {
    await this.viewOpenApiV2Service.updateRecordOrders(...);
    return;
  }
  // v1 实现
  return await this.viewOpenApiService.updateRecordOrders(...);
}
```

### 5.2 金丝雀决策优先级

`CanaryService.shouldUseV2WithReason()`（`apps/nestjs-backend/src/features/canary/canary.service.ts:166-201`）按以下优先级判断：

| 优先级 | 判断条件 | 说明 |
|---|---|---|
| 1 | `FORCE_V2_ALL=true` 环境变量 | 全局强制 v2，最高优先级 |
| 2 | `ENABLE_CANARY_FEATURE !== 'true'` | 金丝雀功能未启用 → 使用 v1 |
| 3 | `config.forceV2All === true` | 数据库配置强制 v2 |
| 4 | `x-canary` 请求头 | Header 覆盖（true/false） |
| 5 | `base.v2Enabled === true` | 新建 base 默认为 v2-first |
| 6 | 空间在金丝雀列表中 | `config.spaceIds.includes(spaceId)` |

### 5.3 V2FeatureGuard 特殊处理

`V2FeatureGuard.isUnsupportedV2Payload()`（`apps/nestjs-backend/src/features/canary/guards/v2-feature.guard.ts:85-100`）有一个**关键例外**：

> V2 尚不支持仅用于重新排序记录的 updateRecord 调用。

即当 `updateRecord` 请求满足以下条件时，**强制降级到 v1**：
- `body.order` 存在
- `body.record.fields` 为空对象

**影响**：画廊卡片排序（只传 order 不传 fields）会走 v1 路径；看板跨列移动（同时传 fields 和 order）可以走 v2 路径。

### 5.4 已启用 v2 的视图相关功能

目前仅 `reorderRecords`（批量记录排序）标记了 `@UseV2Feature('reorderRecords')`：
- **走 v2**：看板同列卡片重排（调用 `updateRecordOrders` API）
- **走 v1**：画廊卡片排序、看板跨列移动

---

## 6. 空列隐藏与列重排的交互影响

### 6.1 空列隐藏的实现机制

`isEmptyStackHidden` 在 `KanbanProvider.stackCollection` 计算逻辑中生效：

```typescript
// KanbanProvider.tsx:192-197
if (isEmptyStackHidden) {
  return stackList.filter(({ count }) => count > 0);
}
```

**作用时机**：在 stackCollection 最终返回前过滤。

### 6.2 与列重排的完整交互链路

```
stackField.options.choices = [A, B, C] (A有数据, B空, C有数据)
              │
              ▼
  KanbanProvider 计算 stackCollection
              │
              ├─ isEmptyStackHidden = false
              │    stackCollection = [未分类, A, B, C]
              │    stackIds = [未分类, A_id, B_id, C_id]
              │    stackMap = {A_id: A, B_id: B, C_id: C}
              │
              └─ isEmptyStackHidden = true
                   stackCollection = [未分类, A, C]  (B被过滤)
                   stackIds = [未分类, A_id, C_id]  (B_id不存在)
                   stackMap = {A_id: A, C_id: C}    (B_id不存在)
```

### 6.3 列重排时的 choices 映射逻辑

```typescript
// KanbanContainer.tsx:67-80
const { choices } = stackField.options;                    // [A, B, C]
const choiceMap = keyBy(choices, 'name');                 // {A_name: A, B_name: B, C_name: C}
const newChoices = newStackIds
  .map((choiceId) => {
    if (choiceId === UNCATEGORIZED_STACK_ID) return;      // 未分类列跳过
    const stack = stackMap[choiceId];                     // 从 stackMap 查找
    if (stack == null) return;                            // ⚠️ 空列被过滤后 stack == null
    return choiceMap[stack.data as string];               // stack.data 是 choice.name
  })
  .filter((choice): choice is NonNullable<typeof choice> => Boolean(choice));
```

### 6.4 完整推导：空列隐藏时拖动列的真实行为

**前提条件**：
- choices = [A, B, C]（A、C 有数据，B 为空）
- `isEmptyStackHidden = true`

**拖动前**：
- 可见列：[未分类, A, C]
- `stackIds = [未分类_id, A_id, C_id]`
- `stackMap = {A_id: A_stack, C_id: C_stack}`（B_id 不在其中）

**拖动操作**：将 C 拖到 A 前面

**重排后的 newStackIds**：
- `newStackIds = [未分类_id, C_id, A_id]`

**构造 newChoices 的过程**：
1. 遍历 `newStackIds`：
   - 未分类_id → return（跳过）
   - C_id → `stackMap[C_id]` → C_stack → `choiceMap[C_stack.data]` → C
   - A_id → `stackMap[A_id]` → A_stack → `choiceMap[A_stack.data]` → A

2. **B 完全没有出现在 newStackIds 中！**

3. 最终 `newChoices = [C, A]`

**最终结论**：
- ❌ **B 被从 choices 数组中彻底删除了**
- ❌ 不是"顺序被打乱"，而是"被移除"
- ⚠️ 这是一个严重的 bug：空列隐藏时拖动列会导致被隐藏的空列从字段选项中**永久丢失**

**验证方法**：
1. 创建 SingleSelect 字段，添加 3 个选项
2. 只给第 1、3 选项创建记录，保持第 2 个选项为空
3. 开启"隐藏空列"
4. 拖动第 3 列到第 1 列前面
5. 关闭"隐藏空列"
6. 检查字段选项：原第 2 个选项已被删除

### 6.5 显示/隐藏空列触发的重渲染

`isEmptyStackHidden` 变更时：
1. `stackCollection` 重新计算
2. `useEffect` 触发，`setStackIds` 同步新的完整/过滤后的列表
3. 列顺序重置为字段 choices 的原始顺序

### 6.6 未分类列的特殊性

`UNCATEGORIZED_STACK_ID` 列：
- 始终插入在 stackList 开头
- 但在列重排映射时会被 `if (choiceId === UNCATEGORIZED_STACK_ID) return` 跳过
- 因此它不参与字段 choices 顺序的持久化
- 它始终显示在最左边，不受列拖动影响

### 6.7 建议的修复方案

在 `KanbanContainer.tsx` 的列重排逻辑中，构造 `newChoices` 时应：
1. 遍历**原始完整的 choices 数组**，而非 `newStackIds`
2. 根据 `newStackIds` 调整可见列的顺序
3. 将被隐藏的空列附加到数组末尾（或保持其原始相对位置）

---

## 7. record 更新接口的完整核对

### 7.1 画廊卡片排序的 API 调用链

**前端调用**（`GalleryViewBase.tsx:105-117`）：
```typescript
updateRecord({
  tableId,
  recordId: activeId as string,
  recordRo: {
    fieldKeyType: FieldKeyType.Id,
    record: { fields: {} },          // 空对象，不修改字段
    order: {
      viewId,
      anchorId: overId as string,
      position: actualOldIndex > actualNewIndex ? 'before' : 'after',
    },
  },
});
```

**后端处理链**：
1. `RecordOpenApiController.updateRecord()`（`record-open-api.controller.ts:141-160`）
   - 检查 V2FeatureGuard，由于 `fields` 为空且有 `order`，**强制走 v1**
2. `RecordOpenApiService.updateRecord()`（`record-open-api.service.ts:172-207`）
   - 转换为 `updateRecords` 调用：
     ```typescript
     await this.updateRecords(
       tableId,
       {
         ...updateRecordRo,
         records: [{ id: recordId, fields: updateRecordRo.record.fields }],
       },
       windowId,
       isAiInternal
     );
     ```
3. `RecordUpdateService.updateRecords()`（`record-modify/record-update.service.ts:61-147`）
   - 在事务中处理 `order`：
     ```typescript
     if (order != null) {
       const { viewId, anchorId, position } = order as IRecordInsertOrderRo;
       await this.viewOpenApiService.updateRecordOrders(table, viewId, {
         anchorId,
         position,
         recordIds: records.map((r) => r.id),
       });
     }
     ```
   - 然后处理字段值更新 + 计算字段联动

**结论**：接口调用链是一致的。画廊的 `updateRecord` 会被转换为 `updateRecords`，在同一个事务中处理 order 更新。

### 7.2 看板跨列移动的 API 调用链

**前端调用**（`KanbanContainer.tsx:122-155`）：
```typescript
const recordRo: IUpdateRecordRo = {
  fieldKeyType: FieldKeyType.Id,
  record: {
    fields: {
      [fieldId]: fieldValue,    // 有字段值更新
    },
  },
};
if (targetCardId == null) {
  recordRo.order = { viewId, anchorId: lastTargetCardId, position: 'after' };
} else {
  recordRo.order = { viewId, anchorId: targetCardId, position: 'before' };
}
updateRecord({ tableId, recordId: sourceCardId, recordRo });
```

**后端处理**：
- 由于 `fields` 不为空，V2FeatureGuard 不会强制降级
- 可以走 v2（如果金丝雀配置启用）
- 处理逻辑与画廊类似，在同一个事务中处理字段值 + order

### 7.3 看板同列卡片重排的 API 调用链

**前端调用**（`KanbanContainer.tsx:92-100`）：
```typescript
updateRecordOrders({
  tableId,
  viewId,
  order: {
    anchorId: cards[targetIndex].id,
    position: targetIndex > sourceIndex ? 'after' : 'before',
    recordIds: [cards[sourceIndex].id],
  },
});
```

**后端处理**：
- 走 `PUT /table/{tableId}/view/{viewId}/record-order` 接口
- 有 `@UseV2Feature('reorderRecords')` 标记
- 可以走 v2（如果金丝雀配置启用）

### 7.4 API 方法与实际代码的一致性核对

| 场景 | 前端调用 | 后端实际处理 | 一致性 |
|---|---|---|---|
| 画廊卡片排序 | `updateRecord(fields={}, order)` | `updateRecord` → `updateRecords` → `updateRecordOrders` | ✅ 一致 |
| 看板跨列移动 | `updateRecord(fields={...}, order)` | `updateRecord` → `updateRecords` → `updateRecordOrders` | ✅ 一致 |
| 看板同列卡片 | `updateRecordOrders` | 独立 API，直接处理 | ✅ 一致 |

**不一致之处**：
- 画廊调用 `updateRecord` 传入 `fields={}`，这是一个"空更新"，但后端仍然会走完整的记录更新流程（包括系统字段更新、计算字段联动等）
- 从性能角度看，画廊应该直接调用 `updateRecordOrders` API，与看板同列卡片重排保持一致
- 但当前实现是**正确的**，只是有优化空间

---

## 8. v1/v2 路径与行为的关联说明

### 8.1 各场景的 v1/v2 路径判定

| 场景 | API | Feature 标记 | V2Guard 检查 | 实际路径 |
|---|---|---|---|---|
| 画廊卡片排序 | `updateRecord` | `updateRecord` | `fields={}` 且有 `order` → **强制 v1** | v1 |
| 看板跨列移动 | `updateRecord` | `updateRecord` | `fields` 非空 → 正常判定 | 可 v1 或 v2 |
| 看板同列卡片 | `updateRecordOrders` | `reorderRecords` | 正常判定 | 可 v1 或 v2 |
| 看板列重排 | `convertField` | 无 | 无标记 → **v1 仅** | v1 |

### 8.2 空列隐藏 bug 与 v1/v2 的关联

空列隐藏时拖动列导致 choices 丢失的 bug：
- **与 v1/v2 无关**：因为列重排调用的是 `convertField` API，没有 `@UseV2Feature` 标记
- 始终走 v1 路径
- bug 存在于前端逻辑，不是后端版本差异导致的

### 8.3 记录排序与 v1/v2 的行为差异

v1 实现（`ViewOpenApiService.updateRecordOrders`）：
- 在同一个事务中先更新 order，再更新记录字段
- 顺序执行：updateRecordOrders → validateFieldsAndTypecast → systemFieldOps → computedOrchestrator
- 有 `orderIndexesBefore` 用于事件通知

v2 实现（`ViewOpenApiV2Service.updateRecordOrders`）：
- 通过 `executeReorderRecordsEndpoint` 调用 DDD 命令总线
- 具体实现位于 `packages/v2/` 目录
- 错误处理通过 domain error code 映射

**注意**：由于画廊卡片排序强制走 v1，即使空间在金丝雀列表中，画廊也始终用 v1 实现。

---

## 9. 字段拖动的关键边界条件

### 9.1 看板列拖动（Stack Draggable）

**权限条件**（`KanbanProvider.tsx:130`、`KanbanStackContainer.tsx:42`）：

```typescript
stackDraggable: Boolean(fieldPermission['field|update'])
```

**禁用条件**（`KanbanStackContainer.tsx:42`、`KanbanContainer.tsx:33,61`）：

| 条件 | 说明 |
|---|---|
| `!isSingleSelectField` | 非 SingleSelect 或 lookup 字段 |
| `disabled` prop | 父组件传入的禁用标志 |
| `editMode` | 列标题编辑中 |
| `isUncategorized` | 未分类列（`UNCATEGORIZED_STACK_ID`） |
| `sourceIndex === targetIndex` | 位置未变 |

**字段类型限制**（`KanbanContainer.tsx:33`）：
```typescript
const isSingleSelectField = fieldType === FieldType.SingleSelect && !isLookup;
```

> 只有单选字段且非 lookup 字段才能拖动列。User、多选等其他字段类型的看板**不支持列重排**。

### 9.2 看板卡片拖动（Card Draggable）

**权限条件**（`KanbanProvider.tsx:134-136`）：

```typescript
cardDraggable: Boolean(
  permission['record|update'] && permission['view|update'] && stackFieldRecordEditable
)
```

需要同时满足：
1. 记录更新权限（`record|update`）
2. 视图更新权限（`view|update`）
3. 栈字段记录可编辑（`stackFieldRecordEditable`）

**禁用条件**（`KanbanStack.tsx:148`）：

| 条件 | 说明 |
|---|---|
| `!cardDraggable` | 权限不足 |
| `isComputed` | 计算字段（公式、自动编号等） |

### 9.3 画廊卡片拖动（Card Draggable）

**权限条件**（`GalleryProvider.tsx:64`）：

```typescript
cardDraggable: Boolean(permission['record|update'] && permission['view|update'])
```

比看板少了 `stackFieldRecordEditable` 检查，因为画廊没有分组字段概念。

**禁用方式**（`GalleryViewBase.tsx:131`）：
```typescript
<SortableContext items={recordIds} strategy={rectSortingStrategy} disabled={!cardDraggable}>
```

### 9.4 字段在列中显示顺序的拖动（ColumnMeta Order）

虽然看板和画廊视图不直接暴露字段顺序拖动 UI，但 `columnMeta.order` 字段仍然存在，用于控制卡片内字段的显示顺序。

**Grid 视图中的实现参考**（`packages/sdk/src/components/grid-enhancements/hooks/use-grid-column-order.ts`）：
```typescript
view.updateColumnMeta(
  operationFields.map((field, index) => ({
    fieldId: field.id,
    columnMeta: { order: newOrders[index] },
  }))
);
```

---

## 10. 分组重排的关键边界条件

### 10.1 看板 stackCollection 计算的完整边界

`KanbanProvider.tsx:141-228` 完整逻辑：

| 条件 | 行为 |
|---|---|
| `groupPoints == null` | 返回 undefined，不渲染 |
| `stackField == null` | 返回 undefined，不渲染 |
| `type === FieldType.Attachment` | 返回 undefined（禁用字段类型） |
| `!stackFieldRecordEditable` | 只返回 `[UNCATEGORIZED_STACK_DATA]` |
| `value == null`（groupPoint 中） | 跳过该分组 |
| `isEmptyStackHidden` | 过滤 `count > 0` 的列 |

### 10.2 SingleSelect 字段的特殊处理

**列顺序来源**：
- 列顺序 = `stackField.options.choices` 的顺序（`KanbanProvider.tsx:182-198`）
- 不是 groupPoints 的返回顺序
- 未出现在 groupPoints 中的 choice 也会显示（count=0）

**列重排的持久化**（`KanbanContainer.tsx:67-80`）：

```typescript
const newChoices = newStackIds
  .map((choiceId) => {
    if (choiceId === UNCATEGORIZED_STACK_ID) return;
    const stack = stackMap[choiceId];
    if (stack == null) return;
    return choiceMap[stack.data as string];
  })
  .filter((choice): choice is NonNullable<typeof choice> => Boolean(choice));
```

**关键点**：
1. 未分类列被跳过，不写入 choices
2. `stack.data` 是 choice 的 name（字符串），不是 choice 的 id
3. 通过 `choiceMap[stack.data as string]` 反查完整 choice 对象
4. 过滤掉 undefined（未找到的 choice）

### 10.3 User 字段的特殊处理

**列顺序来源**：
- 列顺序 = 协作者列表的顺序（`KanbanProvider.tsx:200-220`）
- 不是 groupPoints 的返回顺序
- 未出现在 groupPoints 中的用户也显示（count=0）

**列重排限制**：
- User 字段 `isSingleSelectField=false`（因为 `fieldType !== FieldType.SingleSelect`）
- 因此 User 字段的看板**不支持列拖动重排**

### 10.4 其他字段类型的特殊处理

**列顺序来源**：
- 列顺序 = groupPoints 返回的顺序（`KanbanProvider.tsx:156-179, 222-227`）
- 只包含有数据的分组（value 不为 null）
- 不显示空列（除非 groupPoints 返回了 count=0 的分组）

**列重排限制**：
- 非 SingleSelect 字段 `isSingleSelectField=false`
- 因此**不支持列拖动重排**

---

## 11. 持久化链路的关键边界

### 11.1 记录排序持久化（v1）

`ViewOpenApiService.updateRecordOrders()`（`apps/nestjs-backend/src/features/view/open-api/view-open-api.service.ts:744-797`）：

**完整流程**：
1. 获取 `orderIndexesBefore`（用于事件通知）
2. `getOrCreateViewIndexField()` - 获取或创建视图索引字段
3. `updateRecordOrdersInner()` - 计算并写入新 order 值
   - 查询 anchor 记录的当前 order
   - `updateMultipleOrders()` 计算浮点 order 数组
   - 循环执行 UPDATE SQL 写入每条记录
4. 更新视图 `lastModifiedTime`（OT 操作）
5. 发送 `OPERATION_RECORDS_ORDER_UPDATE` 事件

**关键边界**：
- 索引字段是**每个视图独立**的（`__order_{viewId}`）
- 写入在 `dataPrismaService.$tx` 事务中
- 视图 `lastModifiedTime` 更新走 OT 链路

### 11.2 记录排序持久化（v2）

`ViewOpenApiV2Service.updateRecordOrders()`（`apps/nestjs-backend/src/features/view/open-api/view-open-api-v2.service.ts:36-65`）：

```typescript
const v2Input = {
  tableId,
  recordIds: updateRecordOrdersRo.recordIds,
  order: {
    viewId,
    anchorId: updateRecordOrdersRo.anchorId,
    position: updateRecordOrdersRo.position,
  },
};
const result = await executeReorderRecordsEndpoint(context, v2Input, commandBus);
```

**与 v1 的差异**：
- 使用 v2 DDD 命令总线架构
- 错误处理通过 domain error code 映射
- 具体实现位于 `packages/v2/` 目录

### 11.3 手动排序（Manual Sort）

`ViewOpenApiService.manualSort()`（`apps/nestjs-backend/src/features/view/open-api/view-open-api.service.ts:147-184`）：

**触发场景**：用户点击"手动排序"按钮，按当前 sortObjs 全量重排所有记录。

**流程**：
1. `getOrCreateViewIndexField()` - 获取或创建视图索引字段
2. 构造 SQL：`ROW_NUMBER() OVER (ORDER BY ${orderRawSql})`
3. 执行 `UPDATE ... FROM ...` 批量更新所有记录的 order 字段
4. 更新视图 sort（标记 `manualSort: true`）

**关键边界**：
- 事务超时使用 `bigTransactionTimeout` 配置
- 直接 SQL 更新，不走浮点 order 算法
- 会重置所有间隙，相当于一次 shuffle

### 11.4 浮点 Order 算法的边界

`updateMultipleOrders()`（`apps/nestjs-backend/src/utils/update-order.ts:68-102`）：

**Shuffle 触发条件**（第 89-94 行）：
```typescript
if (gap < Number.EPSILON * 2) {
  await shuffle(parentId);
  // recursive call
  await updateMultipleOrders(params);
  return;
}
```

**Shuffle 的含义**：
- 全量重排所有记录的 order 为连续整数
- 释放浮点精度压力
- 是一个相对重的操作（全表扫描 + 全量更新）

### 11.5 跨列移动的双写原子性

看板跨列移动同时写：
1. 字段值（改变所属列）
2. order 值（改变目标列中的位置）

**实现**：通过单个 `updateRecord` API 请求，后端在一个事务中处理。

**画廊卡片排序**：
- 只写 order 值
- `fields` 为空对象
- **注意**：由于 `V2FeatureGuard.isUnsupportedV2Payload()` 的限制，这种纯排序请求**强制走 v1**

### 11.6 OT 操作与事件的双轨

视图变更同时走两条链路：

```
API 请求
    │
    ├─→ 数据库写入（数据一致性）
    │
    ├─→ ShareDB OT 操作（实时同步）
    │    ViewOpBuilder → updateViewByOps → ShareDB 广播
    │
    └─→ 业务事件通知（审计/撤销）
         Events.OPERATION_* → eventEmitter
```

**关键文件**：
- OT：`ViewOpBuilder`（`packages/core`）
- 事件：`Events.OPERATION_VIEW_UPDATE`、`Events.OPERATION_RECORDS_ORDER_UPDATE`

---

## 12. 三者之间的关联总结

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
  ├─→ v1/v2 路由选择（Canary + FeatureGuard）
  ├─→ REST API 调用（字段/记录/视图变更）
  ├─→ 浮点 order 算法 → DB 索引字段写入
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

## 13. 关键发现与建议

### 13.1 已发现的 Bug

**Bug 1：空列隐藏时拖动列导致 choices 丢失**
- 位置：`KanbanContainer.tsx:67-80`
- 现象：开启"隐藏空列"后拖动列，被隐藏的空列会从 `stackField.options.choices` 中被永久删除
- 原因：构造 `newChoices` 时只遍历可见列的 `newStackIds`，空列不在其中
- 修复建议：遍历原始 choices，根据 `newStackIds` 调整顺序而非过滤

### 13.2 可优化点

**优化点 1：画廊卡片排序应直接调用 updateRecordOrders**
- 当前：画廊调用 `updateRecord(fields={}, order)` → 走完整的记录更新流程
- 建议：直接调用 `updateRecordOrders` API，与看板同列卡片保持一致
- 好处：减少不必要的字段验证、计算字段联动等开销

**优化点 2：画廊排序应支持 v2**
- 当前：由于 V2FeatureGuard 的限制，画廊的纯排序请求强制走 v1
- 建议：修改 V2FeatureGuard 或画廊调用方式，让画廊也能走 v2 路径

### 13.3 v1/v2 迁移状态

| 功能 | v1 支持 | v2 支持 | 备注 |
|---|---|---|---|
| 看板列重排 | ✅ | ❌ | 无 @UseV2Feature 标记 |
| 看板同列卡片 | ✅ | ✅ | 有标记 |
| 看板跨列移动 | ✅ | ✅ | 有标记（需 fields 非空） |
| 画廊卡片排序 | ✅ | ❌ | 强制走 v1 |

---

## 14. 关键文件索引

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
| `apps/nestjs-backend/.../view-open-api.service.ts` | 后端视图操作服务（v1） |
| `apps/nestjs-backend/.../view-open-api-v2.service.ts` | 后端视图操作服务（v2） |
| `apps/nestjs-backend/.../record-open-api.controller.ts` | 记录 API Controller |
| `apps/nestjs-backend/.../record-modify/record-update.service.ts` | 记录更新服务（处理 order） |
| `apps/nestjs-backend/.../canary/canary.service.ts` | 金丝雀发布决策服务 |
| `apps/nestjs-backend/.../canary/guards/v2-feature.guard.ts` | v1/v2 路由 Guard |
| `apps/nestjs-backend/src/utils/update-order.ts` | 浮点 order 算法（updateOrder / updateMultipleOrders） |
| `packages/openapi/src/record/update.ts` | 记录更新 API 定义 |
| `packages/openapi/src/view/update-order.ts` | 视图排序 API 定义 |
| `packages/openapi/src/view/update-record-order.ts` | 记录排序 API 定义 |

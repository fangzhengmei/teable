# Teable Trash & Restore 机制全解析

## 重点摘要

本文档围绕 **三个核心要点** 深入解析 Teable 的 Trash & Restore 机制：

| 要点 | 核心机制 | 关键代码位置 |
|------|----------|--------------|
| **V1/V2 分流** | `CanaryService` 根据 Base 的 `v2Enabled` 标志 + 灰度配置决策路径 | `trash.controller.ts:prepareRestoreTableCanary()` → `canaryService.shouldUseV2ForBaseWithReason()` |
| **deletedTime 级联恢复** | 删除时 table/field/view 共享同一 `deletedTime` 时间戳，恢复时精确匹配批量清除 | `table-open-api.service.ts:restoreTable()` → `updateMany({ deletedTime })` |
| **垃圾清理** | 恢复后在同一事务中删除 `record_trash`（快照内容）和 `table_trash`（操作索引） | `trash.service.ts:874` / `RestoreRecordsHandler.ts` `cleanupTrashRecordIds` |

---

## 1. 整体架构概览

Teable 的 Trash/Restore 系统采用**软删除 + 快照持久化**的双重策略，分为两个层级：

| 层级 | 数据库 | 主要表 | 管理对象 |
|------|--------|--------|----------|
| **Space/Base/Table 级** | 主库（meta DB） | `trash` | Space、Base、Table 的删除/恢复 |
| **Table 内资源级** | 数据库（data DB） | `table_trash` + `record_trash` | View、Field、Record 的删除/恢复 |

关键代码分布：

```
apps/nestjs-backend/src/features/trash/
├── trash.controller.ts              # REST API 入口
├── trash.service.ts                 # 核心业务逻辑（V1 + 路由分发）
├── v2-table-trash.service.ts        # V2 领域事件投影（CQRS）
├── v2-record-trash.service.ts       # V2 记录快照持久化
├── v2-trash-record-name.ts          # V2 记录显示名解析
└── listener/
    └── table-trash.listener.ts      # V1 事件监听：记录/字段/视图删除快照

packages/v2/core/src/
├── commands/
│   ├── RestoreTableCommand.ts       # 恢复表格命令定义
│   ├── RestoreTableHandler.ts       # 恢复表格命令处理器
│   ├── RestoreRecordsCommand.ts     # 恢复记录命令定义
│   ├── RestoreRecordsHandler.ts     # 恢复记录命令处理器
│   ├── RestoreRecordsStreamCommand.ts   # 流式恢复记录命令
│   └── RestoreRecordsStreamHandler.ts   # 流式恢复记录处理器
├── domain/table/
│   ├── Table.ts                     # markTrashed() / markRestored()
│   └── events/
│       ├── TableTrashed.ts          # 表格软删除领域事件
│       └── TableRestored.ts         # 表格恢复领域事件
└── ports/
    ├── TableRepository.ts           # restore() 端口定义
    └── TableRecordRepository.ts     # insertMany() 带 restore 选项
```

---

## 2. 软删除标记机制

### 2.1 deletedTime 字段 — 统一的软删除标记

Teable 没有使用布尔型 `isDeleted` 标记，而是在所有可删除实体上统一使用 `deletedTime: DateTime?` 字段：

| 模型 | deletedTime 语义 |
|------|------------------|
| `Space` | `null` = 活跃，非空 = 已软删除 |
| `Base` | 同上 |
| `TableMeta` | 同上，额外有索引 `@@index([baseId, deletedTime])` |
| `Field` | 同上，`@@index([tableId, deletedTime])` |
| `View` | 同上 |
| `Record`（数据表） | **物理删除**，不设软删除标记，依赖快照恢复 |

**设计要点**：
- `deletedTime` 记录精确的删除时刻，而非布尔值，便于后续按时间筛选和清理
- TableMeta 还有 `permanentDeletedTime` 字段，用于区分软删除与永久删除
- 删除时同步将 `provisionState` 设为 `deleting`，恢复时设回 `ready`

### 2.2 Trash 表 — 删除索引

```prisma
model Trash {
  id           String   @id @default(cuid())
  resourceType String   @map("resource_type")   // "space" | "base" | "table"
  resourceId   String   @map("resource_id")      // 被删资源的 ID
  parentId     String?  @map("parent_id")         // 父资源 ID（Base→SpaceId, Table→BaseId）
  deletedTime  DateTime @default(now()) @map("deleted_time")
  deletedBy    String   @map("deleted_by")
  @@unique([resourceType, resourceId])
}
```

- `Trash` 表是**顶层资源的删除索引**，不存快照内容
- `parentId` 记录层级关系，用于恢复时的**父级存在性校验**
- 唯一约束 `(resourceType, resourceId)` 防止重复删除条目

### 2.3 TableTrash + RecordTrash — 表内资源的快照存储

```prisma
model TableTrash {
  id           String   @id @default(cuid())
  tableId      String   @map("table_id")
  resourceType String   @map("resource_type")   // "view" | "field" | "record"
  snapshot     String   @map("snapshot")          // JSON 快照
  createdTime  DateTime @default(now()) @map("created_time")
  createdBy    String   @map("created_by")
}

model RecordTrash {
  id          String   @id @default(cuid())
  tableId     String   @map("table_id")
  recordId    String   @map("record_id")
  snapshot    String   @map("snapshot")            // 完整记录快照 JSON
  createdTime DateTime @default(now()) @map("created_time")
  createdBy   String   @map("created_by")
}
```

**快照内容差异**：
- **View**: snapshot = `JSON.stringify([viewId])`，仅存 ID
- **Field**: snapshot = `JSON.stringify({ fields: IFieldVo[], records?: IRecord[] })`，存字段定义 + 关联记录数据
- **Record**: `table_trash.snapshot` 存记录 ID 列表，`record_trash.snapshot` 存完整记录 JSON

---

## 3. Restore 主流程与依赖顺序重建

### 3.1 API 入口与 V1/V2 分流（CanaryService 灰度控制）

```
POST /api/trash/restore/:trashId
  └─ TrashController.restoreTrash()
       ├─ prepareRestoreTableCanary()  → 判断是否走 V2
       │    └─ canaryService.shouldUseV2ForBaseWithReason(base, 'restoreTable')
       ├─ V2 路径 → trashService.restoreTrashV2(trashId)
       │    └─ assertParentNotTrashed()
       │    └─ restoreTableV2() → tableOpenApiV2Service.restoreTable()
       │         └─ RestoreTableCommand → RestoreTableHandler
       └─ V1 路径 → trashService.restoreTrash(trashId)
            ├─ trashId 以 "opr" 开头 → restoreTableResource()（表内资源）
            └─ 否则 → restoreResource()（Space/Base/Table）
```

**⚠️ V1/V2 分流的关键逻辑完全在 CanaryService**：
- 决策入口：`getRestoreTableV2Decision(trashId)` → `canaryService.shouldUseV2ForBaseWithReason(base, 'restoreTable')`
- 决策依据：Base 的 `v2Enabled` 标志 + 灰度配置（按百分比、按用户名单等）
- 结果通过 `X-Teable-V2` 响应头返回给前端

决策完整流程：
```typescript
// trash.service.ts:674
async getRestoreTableV2Decision(trashId: string) {
  const trash = await prisma.trash.findUnique({ where: { id: trashId } });
  const baseId = trash.parentId;
  const base = await prisma.base.findUnique({ where: { id: baseId, deletedTime: null } });
  return this.canaryService.shouldUseV2ForBaseWithReason(base, 'restoreTable');
}
```

### 3.2 Space/Base/Table 恢复流程

#### Space 恢复

```typescript
// trash.service.ts:569
private async restoreSpace(spaceId: string) {
  // 1. 权限校验
  await this.permissionService.validPermissions(spaceId, ['space|create'], ...);
  // 2. 直接清空 deletedTime
  await this.prismaService.txClient().space.update({
    where: { id: spaceId },
    data: { deletedTime: null },
  });
}
```

#### Base 恢复（含冲突检测）

```typescript
// trash.service.ts:579
private async restoreBase(baseId: string) {
  // 1. 权限校验
  // 2. 查找 Base 所属 Space
  const base = await prisma.base.findUniqueOrThrow({ ... });
  // 3. 冲突检测：检查父级 Space 是否也在垃圾箱中
  const trashedSpace = await prisma.trash.findFirst({
    where: { resourceId: base.spaceId, resourceType: TrashType.Space },
  });
  if (trashedSpace != null) {
    throw new CustomHttpException(
      'Unable to restore this base because its parent space is also trashed',
      HttpErrorCode.VALIDATION_ERROR,
      { localization: { i18nKey: 'httpErrors.trash.parentSpaceTrashed' } }
    );
  }
  // 4. 清空 deletedTime
  await prisma.base.update({ where: { id: baseId }, data: { deletedTime: null } });
  // 5. 清除性能缓存
  this.performanceCacheService.del(generateBaseNodeListCacheKey(baseId));
}
```

#### Table 恢复（V1）

```typescript
// table-open-api.service.ts:529
async restoreTable(baseId: string, tableId: string) {
  const { deletedTime } = await prisma.trash.findFirstOrThrow({
    where: { resourceId: tableId, resourceType: ResourceType.Table },
  });
  // 1. 恢复 Table 本身（清 deletedTime + provisionState → ready）
  await this.tableService.restoreTable(baseId, tableId);
  // 2. 恢复该 Table 下所有字段（按同一 deletedTime 匹配）
  await prisma.field.updateMany({
    where: { tableId, deletedTime },
    data: { deletedTime: null },
  });
  // 3. 恢复该 Table 下所有视图（按同一 deletedTime 匹配）
  await prisma.view.updateMany({
    where: { tableId, deletedTime },
    data: { deletedTime: null },
  });
}
```

**⚠️ 关键设计（V1 核心机制）：deletedTime 原子对齐恢复**
V1 表格恢复时，字段和视图的恢复依靠 `deletedTime` 精确匹配实现级联恢复：
1. **删除时**：`table/field/view` 在同一事务中被赋予**同一个精确的 deletedTime 时间戳**
2. **恢复时**：通过 `where: { tableId, deletedTime }` 批量清除该时间戳
3. **保证**：只有在"这次删除"中一起被删除的字段和视图才会一起恢复，保证原子性

**为什么不用批量清 `deletedTime: { not: null }`？**
因为表格可能多次删除-恢复，不同批次删除的字段有不同的 deletedTime，精确匹配避免恢复错误批次。

#### Table 恢复（V2 — CQRS）

```typescript
// RestoreTableHandler.ts
async handle(context, command) {
  // 1. 从 repository 查找已软删除的 Table 聚合
  const table = yield* await this.tableQueryService.getDeletedByIdInBase(
    context, command.baseId, command.tableId
  );
  // 2. 在事务中持久化恢复（meta scope）
  yield* await this.unitOfWork.withTransaction(
    context,
    (transactionContext) => this.tableRepository.restore(transactionContext, table),
    { scope: 'meta' }
  );
  // 3. 领域事件：markRestored()
  yield* table.markRestored();
  const events = table.pullDomainEvents();
  // 4. 发布领域事件（触发投影清理 trash 表）
  yield* await this.eventBus.publishMany(context, events);
}
```

**V2 领域事件驱动**：
- `TableTrashed` 事件 → `V2TableTrashedProjection` 写入 `trash` 表
- `TableRestored` 事件 → `V2TableRestoredProjection` 从 `trash` 表删除条目
- `RecordsDeleted` 事件 → `V2RecordsDeletedTableTrashProjection` 持久化记录快照

### 3.3 表内资源恢复流程

表内资源恢复走 `restoreTableResource(trashId)`，按 `resourceType` 分发：

#### View 恢复

```typescript
// trash.service.ts:799
case TableTrashType.View: {
  await this.viewService.restoreView(tableId, snapshot[0]);
  // → view.update({ where: { id: viewId }, data: { deletedTime: null } })
  break;
}
```

#### Field 恢复（含依赖排序重建）

```typescript
// trash.service.ts:802
case TableTrashType.Field: {
  const { fields, records } = snapshot as ICreateFieldsOperation['result'];
  // 1. 重新创建字段（内含依赖排序）
  await this.fieldOpenApiService.createFields(tableId, fields);
  // 2. 如果快照中包含关联记录数据，更新已有记录
  if (records) {
    const existingSnapshots = await this.recordService.getSnapshotBulk(...);
    const existingIdSet = new Set(existingSnapshots.map((s) => s.data.id));
    const filteredRecords = records.filter((r) => existingIdSet.has(r.id));
    if (filteredRecords.length) {
      await this.recordOpenApiService.updateRecords(tableId, {
        fieldKeyType: FieldKeyType.Id,
        records: filteredRecords,
      });
    }
  }
  break;
}
```

**字段恢复的本质是"重新创建"**，而非简单清空 `deletedTime`。这是因为字段删除后其数据库列可能已被清理，恢复需要重新创建物理列。

**依赖排序重建** — `sortCreateFieldsByDependencies()`：

```typescript
// field-open-api.service.ts:1075
private sortCreateFieldsByDependencies<T>(tableId: string, fields: T[]): T[] {
  // 1. 构建依赖图：通过 getFieldReferenceIds() 解析每个字段的引用依赖
  //    如 Lookup 字段依赖 Link 字段，Rollup 依赖 Lookup
  // 2. 拓扑排序（Kahn 算法）：
  //    - 计算每个字段的入度（依赖数）
  //    - 入度为 0 的字段先创建
  //    - 创建后更新依赖字段的入度
  // 3. 检测循环依赖：若排序后数量 < 原始数量，说明存在环，回退到原始顺序
  // 4. 同层级按原始索引排序，保持用户定义的字段顺序
}
```

**引用恢复** — `restoreReference()`：
字段恢复后，关联字段的引用关系也需要恢复：

```typescript
async restoreReference(references: string[]) {
  // 查找引用字段（仅未删除的）
  const fieldRaws = await this.prismaService.txClient().field.findMany({
    where: { id: { in: references }, deletedTime: null },
  });
  // 对每个引用字段进行配置校验和错误标记
  for (const refFieldRaw of fieldRaws) {
    const refField = createFieldInstanceByRaw(refFieldRaw);
    await this.checkAndUpdateError(refFieldRaw.tableId, refField);
  }
}
```

#### Record 恢复（含快照匹配）

```typescript
// trash.service.ts:822
case TableTrashType.Record: {
  const recordIds = snapshot as string[];
  // 1. 查找所有匹配的 record_trash 行
  const recordTrashRows = await this.dataPrismaService.recordTrash.findMany({
    where: { tableId, recordId: { in: recordIds } },
    orderBy: [{ recordId: 'asc' }, { createdTime: 'desc' }, { id: 'desc' }],
  });

  // 2. 快照匹配：一条记录可能被多次删除-恢复-再删除
  //    需要找到"属于本次 trash 操作"的快照
  const latestSnapshotsByRecordId = recordTrashRows.reduce((acc, row) => {
    // 只取 createdTime <= 当前 trash 条目的 createdTime 的最近快照
    if (row.createdTime <= createdTime && !acc.has(row.recordId)) {
      acc.set(row.recordId, row);
    }
    return acc;
  }, new Map());

  // 3. V1 路径：multipleCreateRecords + 清理 trash 数据
  // 4. V2 路径：RestoreRecordsCommand → RestoreRecordsHandler
  if (await this.shouldRestoreRecordsWithV2(tableId)) {
    await this.restoreRecordsV2(tableId, records);
    return;
  }
  await this.recordOpenApiService.multipleCreateRecords(tableId, {
    fieldKeyType: FieldKeyType.Id,
    records,
    typecast: true,
  }, true);
  
  // ⚠️ 清理 record_trash 和 table_trash（关键：在同一事务中）
  await this.dataPrismaService.$tx(async (prisma) => {
    // 1. 删除所有匹配的 record_trash 快照条目
    await prisma.recordTrash.deleteMany({ where: { id: { in: matchedRecordTrashRowIds } } });
    // 2. 删除 table_trash 中的操作索引条目
    await prisma.tableTrash.delete({ where: { id: trashId } });
  }, {
    timeout: this.thresholdConfig.bigTransactionTimeout,  // 大事务超时配置
  });
}
```

**⚠️ 垃圾清理机制（record_trash + table_trash）**
1. **双表联动**：`table_trash` 是操作索引表（存记录 ID 列表），`record_trash` 是快照内容表（存完整记录 JSON）
2. **原子删除**：恢复成功后，两表必须在同一事务中一起删除
3. **精确匹配**：只删除本次恢复对应的 record_trash 行（通过 `createdTime <= trashItem.createdTime` 匹配的行）
4. **大事务保护**：配置 `bigTransactionTimeout` 避免超大数据量恢复超时

**快照时间匹配的核心问题**：同一 `recordId` 可能在 `record_trash` 中有多行（多次删除），恢复时必须取 `createdTime <= 当前trash条目.createdTime` 的最新快照，避免恢复到错误的版本。

### 3.4 V2 记录恢复

```typescript
// RestoreRecordsHandler.ts
async handle(context, command) {
  const table = (await this.tableQueryService.getById(context, command.tableId)).value;
  const batchSize = resolveRestoreRecordsBatchSize(command.records.length);

  for (const batch of this.restoreRecordBatches(command.records, batchSize)) {
    // 1. 构建 TableRecord 领域对象
    const records = this.buildTableRecords(table, batch);
    // 2. 构建恢复元数据（version, orders, autoNumber, createdTime 等）
    const restoreRecordsById = this.buildRestoreRecordsById(batch);
    // 3. 在事务中持久化
    await this.unitOfWork.withTransaction(context, (txContext) =>
      tableRecordRepository.insertMany(txContext, table, records, {
        restoreRecordsById,              // 系统列恢复值
        cleanupTrashRecordIds: batch.map(r => r.recordId),  // ⚠️ V2：Repository 内部执行垃圾清理
      })
    );
    // 4. 发布 RecordsBatchCreated 事件
    const batchEvents = this.buildBatchCreatedEvents(table, batch);
    await this.eventBus.publishMany(context, batchEvents);
  }
}
```

**V2 垃圾清理设计差异**：
- V1：在 trash.service 显示调用 `recordTrash.deleteMany` + `tableTrash.delete`
- V2：通过 `cleanupTrashRecordIds` 参数传递给 `tableRecordRepository.insertMany()`，由 Repository 层在内部事务中清理

**流式恢复** — `RestoreRecordsStreamHandler`：
对于大批量记录，支持流式处理：
- 接收 `AsyncIterable<RestoreRecordInput>` 作为输入
- 按 `batchSize`（1~5000）分批处理
- 每批通过 `insertManyStream` 持久化
- 通过 AsyncGenerator 产出 `progress / done / error` 事件
- 支持延迟计算更新（`deferComputedUpdates`）和跳过计算（`skipComputedUpdates`）

---

## 4. 冲突检测与拦截

### 4.1 父级链路冲突检测 — `assertParentNotTrashed()`

这是最核心的冲突拦截机制，使用递归 CTE 检查整个父级链路：

```typescript
// trash.service.ts:614
private async assertParentNotTrashed(parentId: string | null) {
  if (!parentId) return;

  // 递归 CTE：沿 trash 表的 parent_id 链路向上查找
  const query = this.knex
    .withRecursive('parent_chain', (qb) => {
      // 基础条件：检查直接父级是否在 trash 中
      qb.select('resource_id', 'parent_id')
        .from('trash')
        .where('resource_id', parentId)
        .unionAll((qb) => {
          // 递归条件：沿 parent_id 继续向上查找
          qb.select('t.resource_id', 't.parent_id')
            .from('trash as t')
            .join('parent_chain as pc', 't.resource_id', 'pc.parent_id')
            .whereNotNull('pc.parent_id');
        });
    })
    .select('resource_id')
    .from('parent_chain')
    .limit(1)
    .toQuery();

  const result = await this.prismaService.$queryRawUnsafe(query);
  if (result.length > 0) {
    throw new CustomHttpException(
      'Unable to restore this resource because its parent is also in trash',
      HttpErrorCode.VALIDATION_ERROR,
      { localization: { i18nKey: 'httpErrors.trash.parentBaseTrashed' } }
    );
  }
}
```

**拦截场景举例**：
- 恢复 Base → 检查所属 Space 是否在 trash 中
- 恢复 Table → 检查所属 Base 是否在 trash 中
- 链式场景：恢复 Table → 所属 Base 在 trash → 所属 Space 也在 trash

**为什么需要递归**：trash 表中 `parentId` 只记录直接父级，但父级自身可能也有父级在 trash 中，需要递归遍历整条链路。

### 4.2 Base 恢复时的 Space 级冲突检测

```typescript
// trash.service.ts:588
private async restoreBase(baseId: string) {
  const base = await prisma.base.findUniqueOrThrow({ ... });
  const trashedSpace = await prisma.trash.findFirst({
    where: { resourceId: base.spaceId, resourceType: TrashType.Space },
  });
  if (trashedSpace != null) {
    throw new CustomHttpException(
      'Unable to restore this base because its parent space is also trashed',
      ...
    );
  }
}
```

这是非递归的快捷路径，直接查 trash 表看 Space 是否在垃圾箱中。

### 4.3 Table 恢复时的 deletedTime 校验

```typescript
// table-open-api.service.ts:532
async restoreTable(baseId: string, tableId: string) {
  const { deletedTime } = await prisma.trash.findFirstOrThrow({
    where: { resourceId: tableId, resourceType: ResourceType.Table },
  });
  if (!deletedTime) {
    throw new CustomHttpException(
      'Unable to restore this table because it is not in the trash',
      ...
    );
  }
}
```

防止对不在垃圾箱中的表格执行恢复操作。

### 4.4 Field 恢复时的配置有效性校验

字段恢复走"重新创建"路径，创建后通过 `isFieldConfigurationValid()` 检查：

```typescript
private async isFieldConfigurationValid(tableId, field) {
  // Lookup 字段：校验引用的 Link 字段是否存在且有效
  if (field.lookupOptions && !field.isConditionalLookup) {
    const lookupValid = await this.validateLookupField(field);
    if (!lookupValid) return false;
    // Rollup 字段：额外校验聚合函数
    if (field.type === FieldType.Rollup) {
      return await this.validateRollupAggregation(field);
    }
  }
  // Conditional Lookup：校验条件引用链
  if (field.isConditionalLookup) {
    return await this.validateConditionalLookup(tableId, field);
  }
  // Conditional Rollup：校验条件聚合
  if (field.type === FieldType.ConditionalRollup) {
    return await this.validateConditionalRollupAggregation(tableId, field);
  }
  return true;
}
```

校验失败的字段会被标记为 `hasError: true`，不会阻止恢复操作，但会在 UI 上显示错误状态。

### 4.5 Record 恢复时的快照版本冲突

```typescript
// trash.service.ts:846
// 一条记录可能被多次删除-恢复，需要正确匹配快照
const latestSnapshotsByRecordId = recordTrashRows.reduce((acc, row) => {
  // 只取 createdTime <= 当前 trash 操作时间的快照
  if (row.createdTime <= createdTime && !acc.has(row.recordId)) {
    acc.set(row.recordId, row);
  }
  return acc;
}, new Map());
```

这解决了同一 `recordId` 在 `record_trash` 中有多条记录时的版本选择问题。

---

## 5. 关联对象恢复处理

### 5.1 删除时的级联软删除

表格删除时，所有子资源一起软删除：

```typescript
// table-open-api.service.ts:500
async deleteTable(baseId, tableId) {
  // 1. 先解关联 Link 字段
  await this.detachLink(tableId);
  // 2. 事务中标记所有资源
  const deletedTime = new Date();
  await this.tableService.deleteTable(baseId, tableId, deletedTime);     // table_meta
  await prisma.field.updateMany({ where: { tableId, deletedTime: null }, data: { deletedTime } });
  await prisma.view.updateMany({ where: { tableId, deletedTime: null }, data: { deletedTime } });
}
```

### 5.2 恢复时的级联恢复

V1 Table 恢复时，按同一 `deletedTime` 批量恢复所有字段和视图：

```typescript
await prisma.field.updateMany({ where: { tableId, deletedTime }, data: { deletedTime: null } });
await prisma.view.updateMany({ where: { tableId, deletedTime }, data: { deletedTime: null } });
```

### 5.3 Field 恢复时的关联数据处理

字段恢复时，快照中可能包含关联的记录数据：

```typescript
case TableTrashType.Field: {
  const { fields, records } = snapshot;
  await this.fieldOpenApiService.createFields(tableId, fields);
  if (records) {
    // 只更新当前仍存在的记录
    const existingSnapshots = await this.recordService.getSnapshotBulk(tableId, records.map(r => r.id));
    const existingIdSet = new Set(existingSnapshots.map(s => s.data.id));
    const filteredRecords = records.filter(r => existingIdSet.has(r.id));
    if (filteredRecords.length) {
      await this.recordOpenApiService.updateRecords(tableId, {
        fieldKeyType: FieldKeyType.Id,
        records: filteredRecords,
      });
    }
  }
}
```

**重要细节**：只更新仍然存在的记录，避免对已删除的记录执行无效更新。

### 5.4 Link 字段解关联 — `detachLink()`

表格删除前会先解关联 Link 字段：

```typescript
// table-open-api.service.ts 中 deleteTable 和 permanentDeleteTables 都会调用
await this.detachLink(tableId);
```

这确保删除表格时，其他表格中引用该表格的 Link 字段不会留下悬空引用。

### 5.5 V2 记录恢复的系统列保留

V2 恢复记录时，不仅恢复字段数据，还恢复系统列：

```typescript
// RestoreRecordsHandler.ts
private buildRestoreRecordsById(batch) {
  return new Map(batch.map(record => [
    record.recordId,
    {
      ...(record.version !== undefined ? { version: record.version } : {}),
      ...(record.orders ? { orders: record.orders } : {}),
      ...(record.autoNumber !== undefined ? { autoNumber: record.autoNumber } : {}),
      ...(record.createdTime ? { createdTime: record.createdTime } : {}),
      ...(record.createdBy ? { createdBy: record.createdBy } : {}),
      ...(record.lastModifiedTime ? { lastModifiedTime: record.lastModifiedTime } : {}),
      ...(record.lastModifiedBy ? { lastModifiedBy: record.lastModifiedBy } : {}),
      ...(record.extraColumnValues ? { extraColumnValues: record.extraColumnValues } : {}),
    },
  ]));
}
```

这确保恢复后的记录保留原始的创建时间、版本号、行序等元数据。

### 5.6 永久删除与垃圾箱清理

```typescript
// trash.service.ts:1060
private async resetTableTrashItems(tableId) {
  // 收集所有已删除的 view/field/record ID
  // 1. 物理删除 view、field、taskReference、ops
  // 2. 清理 record_trash、table_trash
}

// table-open-api.service.ts:362
async permanentDeleteTables(baseId, tableIds) {
  // 1. detachLink
  // 2. dropTables — 物理删除数据表
  // 3. cleanTaskRelatedData — 清理任务引用
  // 4. cleanTablesRelatedData — 清理所有关联数据（field, view, ops, trash, recordTrash 等）
}
```

---

## 6. 删除→恢复完整数据流

### 6.1 Table 删除数据流（V1）

```
用户操作 → deleteTable(baseId, tableId)
  ├─ detachLink(tableId)            // 解除 Link 关联
  ├─ tableService.deleteTable()     // table_meta.deletedTime = now, provisionState = deleting
  ├─ field.updateMany()             // 所有字段 deletedTime = now
  ├─ view.updateMany()              // 所有视图 deletedTime = now
  └─ 事件 → trash 表写入记录       // (resourceType=Table, resourceId=tableId, parentId=baseId)
```

### 6.2 Table 删除数据流（V2 CQRS）

```
用户操作 → DeleteTableCommand → DeleteTableHandler
  ├─ tableQueryService.getByIdInBase()       // 查找 Table 聚合
  ├─ table.markTrashed()                     // 领域事件: TableTrashed
  ├─ unitOfWork.withTransaction()            // 持久化: tableRepository.delete()
  └─ eventBus.publishMany()                  // 发布事件
       ├─ V2TableTrashedProjection           // → 写入 trash 表
       └─ (其他投影处理器)

记录删除 → RecordsDeleted 事件
  ├─ V2RecordsDeletedTableTrashProjection    // → 持久化快照到 table_trash + record_trash
  └─ V2RecordsDeletedAttachmentProjection    // → 清理 attachments_table
```

### 6.3 Table 恢复数据流（V1）

```
POST /api/trash/restore/:trashId
  └─ trashService.restoreTrash(trashId)
       ├─ prisma.trash.findUniqueOrThrow()       // 查找 trash 记录
       ├─ assertParentNotTrashed(parentId)         // 递归 CTE 检查父级链
       ├─ restoreResource({resourceType, resourceId})
       │    └─ restoreTable(tableId)
       │         ├─ permissionService.validPermissions()
       │         └─ tableOpenApiService.restoreTable(baseId, tableId)
       │              ├─ tableService.restoreTable()    // deletedTime=null, provisionState=ready
       │              ├─ field.updateMany()             // 同一 deletedTime 的字段恢复
       │              └─ view.updateMany()              // 同一 deletedTime 的视图恢复
       └─ prisma.trash.deleteMany({id: trashId})   // 清理 trash 索引
```

### 6.4 Table 恢复数据流（V2）

```
POST /api/trash/restore/:trashId
  └─ trashService.restoreTrashV2(trashId)
       ├─ getRestoreTableV2Decision()                // V2 分流判断
       ├─ assertParentNotTrashed(baseId)              // 递归 CTE 父级检查
       └─ restoreTableV2(baseId, tableId)
            └─ tableOpenApiV2Service.restoreTable()
                 └─ RestoreTableCommand → RestoreTableHandler
                      ├─ getDeletedByIdInBase()       // 查找已软删除的 Table
                      ├─ unitOfWork.withTransaction()  // tableRepository.restore()
                      ├─ table.markRestored()          // 领域事件: TableRestored
                      └─ eventBus.publishMany()        // 发布事件
                           └─ V2TableRestoredProjection  // → 从 trash 表删除条目
```

---

## 7. 核心设计模式总结

| 模式 | 实现 | 场景 |
|------|------|------|
| **软删除标记** | `deletedTime: DateTime?` | Space/Base/Table/Field/View |
| **删除索引** | `Trash` 表 + `parentId` 层级 | 快速查询已删除资源 + 父级校验 |
| **快照持久化** | `TableTrash` + `RecordTrash` | View/Field/Record 的完整数据保存 |
| **⚠️ deletedTime 时间戳对齐** | 删除时 table/field/view 共享精确 `deletedTime` | **V1 批量级联恢复的匹配依据** |
| **快照版本匹配** | `createdTime <= trashItem.createdTime` | 同一记录多次删除-恢复的版本选择 |
| **拓扑排序** | Kahn 算法 + 字段依赖图 | Field 恢复时的创建顺序 |
| **递归 CTE** | `assertParentNotTrashed()` | 父级链路完整性校验 |
| **⚠️ CanaryService 灰度分流** | `shouldUseV2ForBaseWithReason()` | **V1/V2 架构迁移路径决策** |
| **CQRS 投影** | V2 领域事件 → 投影处理器 | Trash 表的读写分离 |
| **流式批量** | `RestoreRecordsStreamHandler` | 大批量记录恢复的渐进式处理 |
| **引用恢复** | `restoreReference()` + `isFieldConfigurationValid()` | Link/Lookup/Rollup 字段恢复后的校验 |
| **⚠️ 原子垃圾清理** | 事务中同时删除 `record_trash` + `table_trash` | **恢复成功后清理快照** |

# Teable Trash & Restore 机制全解析（完整代码路径追踪）

## 重点摘要

本文档从**完整代码链路**深度解析 Teable 的 Trash & Restore 机制，核心围绕两个设计原则：

| 原则 | 适用对象 | 实现机制 | 关键代码位置 |
|------|----------|----------|--------------|
| **统一软删除标记** | Space/Base/Table/Field/View | 所有可删除实体共用 `deletedTime: DateTime?` 字段，`null`=活跃，非空=已删除 | `packages/db-main-prisma/prisma/template.prisma` 中各模型 |
| **物理删除 + 快照恢复** | Record（数据表记录） | 不设软删除标记，直接从数据表物理删除，恢复依赖 `table_trash` + `record_trash` 快照 | `record.service.ts:batchDeleteRecords` → `DELETE FROM table` |

---

## 1. 架构分层与代码路径总览

### 1.1 层级架构

```
┌──────────────────────────────────────────────────────────────┐
│                    REST API 层 (Controller)                   │
│  trash.controller.ts → POST /api/trash/restore/:trashId       │
├──────────────────────────────────────────────────────────────┤
│              Service 层 (业务逻辑 + 路径分发)                  │
│  ├─ trash.service.ts              (V1 主逻辑 + V1/V2 分流)    │
│  ├─ v2-table-trash.service.ts      (V2 领域事件投影)          │
│  ├─ v2-record-trash.service.ts     (V2 记录快照持久化)        │
│  ├─ table-open-api.service.ts      (V1 Table 删除/恢复)       │
│  ├─ table-open-api-v2.service.ts   (V2 Table 删除/恢复)       │
│  ├─ field-open-api.service.ts      (字段恢复 + 依赖排序)      │
│  ├─ record-delete.service.ts       (V1 记录删除)              │
│  ├─ record-open-api.service.ts     (V1 记录恢复)              │
│  └─ view.service.ts                (视图恢复)                 │
├──────────────────────────────────────────────────────────────┤
│              Domain 层 (V2 核心)                              │
│  ├─ Table.ts → markTrashed() / markRestored()                 │
│  ├─ 领域事件: TableTrashed / TableRestored / RecordsDeleted   │
│  └─ 命令处理器: RestoreTableHandler / RestoreRecordsHandler   │
├──────────────────────────────────────────────────────────────┤
│              Repository 层 (数据访问)                         │
│  ├─ TableRepository.restore()                                 │
│  ├─ TableRecordRepository.insertMany({ cleanupTrashRecordIds })│
│  └─ Prisma 元数据操作                                          │
└──────────────────────────────────────────────────────────────┘
```

### 1.2 代码调用链全景

```
删除路径:
  Table 删除: deleteTable(baseId, tableId)
    → detachLink()
    → deletedTime = new Date() 「同一时间戳」
    → tableService.deleteTable(..., deletedTime)  → table_meta.deletedTime
    → prisma.field.updateMany({ deletedTime })     → field.deletedTime
    → prisma.view.updateMany({ deletedTime })      → view.deletedTime
    → (写入 trash 表索引)

  Record 删除 (V1): deleteRecords(tableId, recordIds)
    → getRecordsById() 「保存快照」
    → batchDeleteRecords() 「物理删除: DELETE FROM table」
    → emit OPERATION_RECORDS_DELETE 事件
        → TableTrashListener.recordDeleteListener()
            → tableTrash.create  { snapshot: JSON.stringify(recordIds) }
            → recordTrash.createMany { snapshot: JSON.stringify(fullRecord) }

  Record 删除 (V2): DeleteRecordsCommand → DeleteRecordsHandler
    → tableRecordRepository.deleteMany() 「物理删除」
    → RecordsDeleted 事件
        → V2RecordsDeletedTableTrashProjection.persistDeletedRecords()
            → tableTrash.insertInto
            → recordTrash.insertInto （分批次，每批 5000 条）

恢复路径:
  Table 恢复 (V1): restoreTable(baseId, tableId)
    → const { deletedTime } = trash.findFirst() 「获取删除时的时间戳」
    → tableService.restoreTable()                  → deletedTime = null
    → prisma.field.updateMany({ tableId, deletedTime }, { deletedTime: null })
    → prisma.view.updateMany({ tableId, deletedTime }, { deletedTime: null })

  Record 恢复 (V1): restoreTableResource(trashId)
    → recordTrash.findMany() 「按 createdTime <= trash.createdTime 匹配快照」
    → multipleCreateRecords() 「重新插入: INSERT INTO table」
    → $tx: recordTrash.deleteMany() + tableTrash.delete() 「原子清理快照」

  Record 恢复 (V2): RestoreRecordsCommand → RestoreRecordsHandler
    → buildTableRecords() 「构建领域对象」
    → tableRecordRepository.insertMany({ cleanupTrashRecordIds }) 「Repository 内部清理」
```

---

## 2. 统一软删除标记机制：`deletedTime: DateTime?`

### 2.1 Prisma Schema 定义

**所有可删除实体采用完全相同的软删除模式**，在 `packages/db-main-prisma/prisma/template.prisma` 中统一定义：

```prisma
model Space {
  deletedTime      DateTime? @map("deleted_time")   // ← 软删除标记
}

model Base {
  deletedTime      DateTime?   @map("deleted_time") // ← 软删除标记
}

model TableMeta {
  deletedTime       DateTime?           @map("deleted_time")  // ← 软删除标记
  @@index([baseId, deletedTime])                         // 查询索引
}

model Field {
  deletedTime         DateTime? @map("deleted_time")  // ← 软删除标记
  @@index([tableId, deletedTime])                    // 查询索引
}

model View {
  deletedTime         DateTime? @map("deleted_time")  // ← 软删除标记
}
```

### 2.2 语义统一

| deletedTime 值 | 语义 | 查询条件 |
|----------------|------|----------|
| `null` | 活跃（正常） | `where: { deletedTime: null }` |
| 非空 Date | 已软删除 | `where: { deletedTime: { not: null } }` |

### 2.3 删除时的原子性标记（V1 Table 删除）

**核心设计**：删除时 Table/Field/View 共享**同一个精确到毫秒的 `deletedTime` 时间戳**，保证级联恢复的原子性。

完整代码路径：

```typescript
// apps/nestjs-backend/src/features/table/open-api/table-open-api.service.ts:500
async deleteTable(baseId: string, tableId: string) {
  await this.detachLink(tableId);  // 先解除 Link 字段关联

  return await this.prismaService.$tx(async (prisma) => {
    // ⚠️ 关键：只生成一个时间戳，全表共享
    const deletedTime = new Date();

    // 1. Table 标记删除
    await this.tableService.deleteTable(baseId, tableId, deletedTime);
    // → table_meta.deletedTime = deletedTime
    // → provisionState = 'deleting'

    // 2. 所有 Field 标记删除（同一时间戳）
    await prisma.field.updateMany({
      where: { tableId, deletedTime: null },  // 只更新当前活跃的
      data: { deletedTime },                  // 精确时间戳
    });

    // 3. 所有 View 标记删除（同一时间戳）
    await prisma.view.updateMany({
      where: { tableId, deletedTime: null },  // 只更新当前活跃的
      data: { deletedTime },                  // 精确时间戳
    });
  });
}
```

`tableService.deleteTable` 内部实现：

```typescript
// apps/nestjs-backend/src/features/table/table.service.ts:287
async deleteTable(baseId: string, tableId: string, deletedTime: Date) {
  const result = await this.prismaService.txClient().tableMeta.findFirst({
    where: { id: tableId, baseId, deletedTime: null },  // 确保当前是活跃的
  });

  await this.prismaService.txClient().tableMeta.update({
    where: { id: tableId, baseId },
    data: {
      version: version + 1,
      deletedTime,          // 使用传入的时间戳
      lastModifiedBy: userId,
      provisionState: ProvisionState.deleting,
    },
  });

  // 记录操作历史
  await this.batchService.saveRawOps(baseId, RawOpType.Del, IdPrefix.Table, [
    { docId: tableId, version },
  ]);
}
```

### 2.4 恢复时的精确匹配（V1 Table 恢复）

恢复时通过**精确匹配同一 `deletedTime`** 实现级联恢复：

```typescript
// apps/nestjs-backend/src/features/table/open-api/table-open-api.service.ts:529
async restoreTable(baseId: string, tableId: string) {
  return await this.prismaService.$tx(async (prisma) => {
    // 1. 从 trash 表获取删除时的精确时间戳
    const { deletedTime } = await prisma.trash.findFirstOrThrow({
      where: { resourceId: tableId, resourceType: ResourceType.Table },
    });

    if (!deletedTime) {
      throw new CustomHttpException(
        'Unable to restore this table because it is not in the trash',
        HttpErrorCode.VALIDATION_ERROR,
      );
    }

    // 2. 恢复 Table 本身
    await this.tableService.restoreTable(baseId, tableId);
    // → deletedTime = null
    // → provisionState = 'ready'

    // 3. ⚠️ 只恢复「这次删除」的字段（精确时间戳匹配）
    await prisma.field.updateMany({
      where: { tableId, deletedTime },  // 精确匹配删除时的时间戳
      data: { deletedTime: null },      // 清空标记
    });

    // 4. ⚠️ 只恢复「这次删除」的视图（精确时间戳匹配）
    await prisma.view.updateMany({
      where: { tableId, deletedTime },  // 精确匹配删除时的时间戳
      data: { deletedTime: null },      // 清空标记
    });
  });
}
```

**设计合理性分析**：
- 表格可能多次删除→恢复→再删除，每次删除有不同的 `deletedTime`
- 恢复时精确匹配时间戳，确保只恢复"这次删除"的字段和视图
- 如果使用 `where: { deletedTime: { not: null } }` 会错误地恢复所有历史删除批次

---

## 3. Record 物理删除 + 快照恢复

### 3.1 设计原则：Record 不走软删除

与 Space/Base/Table/Field/View 不同，**数据表中的 Record 记录不使用软删除**：
- 数据表可能有百万/千万级记录，软删除会导致数据膨胀
- 查询时需要额外过滤 `WHERE deleted_time IS NULL`，影响性能
- 恢复时需要重建索引和关联，快照恢复更可靠

### 3.2 V1 记录删除完整路径

```typescript
// apps/nestjs-backend/src/features/record/record-modify/record-delete.service.ts:30
async deleteRecords(tableId: string, recordIds: string[], windowId?: string) {
  const table = await this.tableDomainQueryService.getTableDomainById(tableId);

  const { records: recordsForEvent, orders } = await this.prismaService.$tx(async () => {
    // 1. 先查询完整记录，保存快照（用于事件和恢复）
    const recordsForEvent = await this.recordService.getRecordsById(
      tableId, recordIds, false, false
    );

    // 2. 处理 Link 字段关联更新
    const cellContextsByTableId = await this.linkService.getDeleteRecordUpdateContext(
      tableId, recordsForEvent.records
    );

    // 3. 获取记录在视图中的行序（恢复时用）
    const orders = windowId
      ? await this.recordService.getRecordIndexes(table, recordIds)
      : undefined;

    // 4. ⚠️ 物理删除记录（DELETE FROM table WHERE __id IN (...)）
    await this.computedOrchestrator.computeCellChangesForRecordsMulti(
      sources, async () => {
        await this.recordService.batchDeleteRecords(tableId, recordIds);
      }
    );

    return { records: recordsForEvent, orders };
  });

  // 5. 发布删除事件，触发 TableTrashListener 写入快照
  this.eventEmitterService.emitAsync(Events.OPERATION_RECORDS_DELETE, {
    operationId: generateOperationId(),
    windowId,
    tableId,
    userId: this.cls.get('user.id'),
    records: recordsForEvent.records.map((record, index) => ({
      ...record,
      order: orders?.[index],
    })),
  });
}
```

物理删除核心实现：

```typescript
// apps/nestjs-backend/src/features/record/record.service.ts:1123
async batchDeleteRecords(tableId: string, recordIds: string[]) {
  const dbTableName = await this.getDbTableName(tableId);

  // 1. 先查版本号（用于乐观锁）
  const nativeQuery = this.knex(dbTableName)
    .select('__id as id', '__version as version')
    .whereIn('__id', recordIds)
    .toQuery();
  const recordRaw = await this.dataPrismaService
    .txClient()
    .$queryRawUnsafe<{ id: string; version: number }[]>(nativeQuery);

  // 2. 记录操作历史
  const dataList = recordIds.map((recordId) => ({
    docId: recordId,
    version: recordRawMap[recordId].version,
  }));
  await this.batchService.saveRawOps(tableId, RawOpType.Del, IdPrefix.Record, dataList);

  // 3. ⚠️ 物理删除（真正的 DELETE SQL）
  await this.batchDel(tableId, recordIds);
}
```

### 3.3 快照持久化（V1 监听器）

删除事件触发后，`TableTrashListener` 异步写入快照：

```typescript
// apps/nestjs-backend/src/features/trash/listener/table-trash.listener.ts:19
@OnEvent(Events.OPERATION_RECORDS_DELETE)
async recordDeleteListener(payload: IDeleteRecordsPayload) {
  const { operationId, userId, tableId, records } = payload;
  const recordIds = records.map((record) => record.id);
  const createdTime = new Date();

  // ⚠️ 同一事务写入 table_trash 和 record_trash
  await this.dataPrismaService.$tx(async (prisma) => {
    // 1. 写入 table_trash 索引表
    await prisma.tableTrash.create({
      data: {
        id: operationId,              // 操作 ID，用于定位
        tableId,
        createdBy: userId,
        resourceType: ResourceType.Record,
        snapshot: JSON.stringify(recordIds),  // 仅存记录 ID 列表
        createdTime,
      },
    });

    // 2. 写入 record_trash 快照表（批量，每批 5000 条）
    const batchSize = 5000;
    for (let i = 0; i < records.length; i += batchSize) {
      const batch = records.slice(i, i + batchSize);
      await prisma.recordTrash.createMany({
        data: batch.map((record) => ({
          id: generateRecordTrashId(),  // 每条快照独立 ID
          tableId,
          recordId: record.id,          // 原记录 ID
          snapshot: JSON.stringify(record),  // ⚠️ 完整记录 JSON 快照
          createdBy: userId,
          createdTime,                  // ⚠️ 同一 createdTime
        })),
      });
    }
  });
}
```

### 3.4 V2 记录删除路径

V2 通过领域事件驱动，快照由投影处理器持久化：

```typescript
// packages/v2/core/src/commands/DeleteRecordsHandler.ts:66
async handle(context, command) {
  // ...
  // 1. 物理删除
  const deleteResult = yield* await this.unitOfWork.withTransaction(
    context,
    async (transactionContext) => {
      return await handler.tableRecordRepository.deleteMany(
        transactionContext, table, scopedDeleteSpec
      );
    }
  );

  // 2. 发布领域事件
  const events: IDomainEvent[] = [
    RecordsDeleted.create({
      tableId: table.id(),
      baseId: table.baseId(),
      recordIds: deletedRecordIds,
      recordSnapshots,  // ⚠️ 携带完整快照
      orchestration: { ... },
    }),
  ];
  yield* await handler.eventBus.publishMany(context, events);
}
```

V2 投影处理器写入快照：

```typescript
// apps/nestjs-backend/src/features/trash/v2-table-trash.service.ts:48
@ProjectionHandler(RecordsDeleted)
export class V2RecordsDeletedTableTrashProjection implements IEventHandler<RecordsDeleted> {
  async handle(context: IExecutionContext, event: RecordsDeleted) {
    const records = event.recordSnapshots.map((snapshot) => {
      const record: IDeleteRecordsPayload['records'][number] = {
        id: snapshot.id,
        fields: snapshot.fields as IRecord['fields'],
        version: snapshot.version,
        autoNumber: snapshot.autoNumber,
        createdTime: snapshot.createdTime,
        createdBy: snapshot.createdBy,
        lastModifiedTime: snapshot.lastModifiedTime,
        lastModifiedBy: snapshot.lastModifiedBy,
        order: snapshot.orders,
      };
      if (snapshot.displayName) record.name = snapshot.displayName;
      return record;
    });

    await this.v2RecordTrashService.persistDeletedRecords(
      {
        operationId: generateOperationId(),
        windowId: context.windowId,
        tableId: event.tableId.toString(),
        userId: context.actorId.toString(),
        records,
      },
      context
    );
  }
}
```

### 3.5 V1 记录恢复 + 垃圾清理

恢复时先匹配快照版本，再重新插入，最后原子清理快照：

```typescript
// apps/nestjs-backend/src/features/trash/trash.service.ts:822
case TableTrashType.Record: {
  const recordIds = snapshot as string[];

  // 1. 查找所有可能的快照（同一 recordId 可能有多条历史快照）
  const recordTrashRows = await this.dataPrismaService.recordTrash.findMany({
    where: { tableId, recordId: { in: recordIds } },
    orderBy: [{ recordId: 'asc' }, { createdTime: 'desc' }, { id: 'desc' }],
  });

  // 2. ⚠️ 快照版本匹配：只取 createdTime <= 当前 trash 操作时间的最近快照
  const latestSnapshotsByRecordId = recordTrashRows.reduce<
    Map<string, IRecordTrashSnapshotRow>
  >((acc, row) => {
    // 同一条记录可能多次删除-恢复-再删除
    // 只有 createdTime <= 本次 trashItem.createdTime 且未被匹配过的才取
    if (row.createdTime <= createdTime && !acc.has(row.recordId)) {
      acc.set(row.recordId, row);
    }
    return acc;
  }, new Map());

  // 3. 提取匹配的快照
  const matchedRecordTrashRows = recordIds
    .map((recordId) => latestSnapshotsByRecordId.get(recordId))
    .filter((row): row is IRecordTrashSnapshotRow => row != null);
  const records = matchedRecordTrashRows.map(({ snapshot }) => JSON.parse(snapshot));

  // 4. V1 路径：物理插入
  await this.recordOpenApiService.multipleCreateRecords(
    tableId,
    {
      fieldKeyType: FieldKeyType.Id,
      records,
      typecast: true,
    },
    true
  );

  // 5. ⚠️ 原子清理快照（同一事务）
  await this.dataPrismaService.$tx(
    async (prisma) => {
      // 删除已恢复的 record_trash 快照条目
      await prisma.recordTrash.deleteMany({
        where: { id: { in: matchedRecordTrashRows.map(({ id }) => id) },
      });
      // 删除 table_trash 操作索引
      await prisma.tableTrash.delete({
        where: { id: trashId },
      });
    },
    {
      timeout: this.thresholdConfig.bigTransactionTimeout,  // 大事务保护
    }
  );
}
```

### 3.6 V2 记录恢复 + 垃圾清理

V2 路径中，垃圾清理由 Repository 层内部处理：

```typescript
// packages/v2/core/src/commands/RestoreRecordsHandler.ts:56
async handle(context, command) {
  const table = (await this.tableQueryService.getById(context, command.tableId)).value;
  const batchSize = resolveRestoreRecordsBatchSize(command.records.length);

  for (const batch of this.restoreRecordBatches(command.records, batchSize)) {
    // 1. 构建 TableRecord 领域对象
    const records = this.buildTableRecords(table, batch);

    // 2. 构建恢复元数据（系统列保留）
    const restoreRecordsById = this.buildRestoreRecordsById(batch);

    // 3. 持久化 + 清理（Repository 内部执行）
    const persistedResult = await this.unitOfWork.withTransaction(
      context,
      async (transactionContext) => {
        return tableRecordRepository.insertMany(
          transactionContext,
          table,
          records,
          {
            restoreRecordsById,
            // ⚠️ 传递需要清理的记录 ID，Repository 内部删除快照
            cleanupTrashRecordIds: batch.map((record) => record.recordId),
          }
        );
      }
    );
  }
}
```

### 3.7 V1 vs V2 垃圾清理设计差异

| 维度 | V1 | V2 |
|------|----|----|
| **清理位置** | `trash.service` 显式调用 delete | `TableRecordRepository.insertMany()` 内部 |
| **参数传递** | 显式匹配后得到的 `matchedRecordTrashRowIds` | `cleanupTrashRecordIds: recordId[]` |
| **事务控制** | `trash.service` 中开启事务 | Repository 层内部事务 |
| **清理内容** | `record_trash.deleteMany` + `table_trash.delete` | 只清理 `record_trash`（`table_trash` 由上层清理？） |

---

## 4. 关联对象恢复处理

### 4.1 View 恢复

View 恢复最简单，直接清空 `deletedTime`：

```typescript
// apps/nestjs-backend/src/features/view/view.service.ts:237
async restoreView(tableId: string, viewId: string) {
  await this.prismaService.$tx(async () => {
    // 清空软删除标记
    await this.prismaService.txClient().view.update({
      where: { id: viewId },
      data: { deletedTime: null },
    });
    // 更新修改时间
    const ops = ViewOpBuilder.editor.setViewProperty.build({
      key: 'lastModifiedTime',
      newValue: new Date().toISOString(),
    });
    await this.updateViewByOps(tableId, viewId, [ops]);
  });
}
```

### 4.2 Field 恢复（含依赖排序重建）

Field 恢复走"重新创建"路径（非清空 `deletedTime`），因为字段删除后物理列可能已被清理：

```typescript
// apps/nestjs-backend/src/features/trash/trash.service.ts:802
case TableTrashType.Field: {
  const { fields, records } = snapshot as ICreateFieldsOperation['result'];

  // 1. 重新创建字段（内部会做依赖排序）
  await this.fieldOpenApiService.createFields(tableId, fields);

  // 2. 如果快照中有记录数据，更新仍存在的记录
  if (records) {
    const existingSnapshots = await this.recordService.getSnapshotBulk(
      tableId,
      records.map((r) => r.id)
    );
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

**依赖排序重建** - 字段创建前会进行拓扑排序：

```typescript
// apps/nestjs-backend/src/features/field/open-api/field-open-api.service.ts:1075
private sortCreateFieldsByDependencies<T>(tableId: string, fields: T[]): T[] {
  // 1. 解析每个字段的依赖
  //    - Lookup 字段依赖 Link 字段
  //    - Rollup 字段依赖 Lookup 字段
  //    - Formula 字段依赖其他字段
  const depsByFieldId = new Map<string, string[]>();
  for (const field of fields) {
    const instance = createFieldInstanceByVo(fieldVo);
    const deps = this.fieldSupplementService
      .getFieldReferenceIds(instance)
      .filter(id => idSet.has(id) && id !== field.id);
    depsByFieldId.set(field.id, deps);
  }

  // 2. Kahn 拓扑排序
  const indegree = new Map<string, number>();
  const outgoing = new Map<string, string[]>();
  // ... 构建图 ...

  const ready: string[] = [];
  while (ready.length) {
    const current = ready.shift()!;
    orderedIds.push(current);
    for (const next of outgoing.get(current) ?? []) {
      const nextDegree = (indegree.get(next) ?? 0) - 1;
      indegree.set(next, nextDegree);
      if (nextDegree === 0) ready.push(next);
    }
  }

  // 3. 环检测：如果排序后数量 < 原始数量，说明有循环依赖
  if (orderedIds.length !== fields.length) {
    this.logger.warn(`detected a dependency cycle; falling back to input order`);
    return fields;  // 回退到原始顺序
  }
}
```

### 4.3 Link 字段解关联

表格删除前先解关联 Link 字段，避免悬空引用：

```typescript
// table-open-api.service.ts 删除和永久删除前都会调用
await this.detachLink(tableId);
```

字段恢复后触发引用恢复和校验：

```typescript
// field-open-api.service.ts:1214
if (referencesToRestore.size) {
  await this.restoreReference(Array.from(referencesToRestore));
}
```

---

## 5. 冲突检测与拦截

### 5.1 父级链路完整性校验（递归 CTE）

恢复前检查整个父级链路是否都不在垃圾箱中：

```typescript
// apps/nestjs-backend/src/features/trash/trash.service.ts:614
private async assertParentNotTrashed(parentId: string | null) {
  if (!parentId) return;

  // ⚠️ 递归 CTE：沿 trash.parent_id 向上遍历整条链路
  const query = this.knex
    .withRecursive('parent_chain', (qb) => {
      // 基础条件：直接父级
      qb.select('resource_id', 'parent_id')
        .from('trash')
        .where('resource_id', parentId)
        .unionAll((qb) => {
          // 递归条件：父级的父级
          qb.select('t.resource_id', 't.parent_id')
            .from('trash as t')
            .join('parent_chain as pc', 't.resource_id', 'pc.parent_id')
            .whereNotNull('pc.parent_id');
        });
    })
    .select('resource_id')
    .from('parent_chain')
    .limit(1)  // 找到一个就足够
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

### 5.2 Base 恢复时的 Space 校验

```typescript
// trash.service.ts:588
private async restoreBase(baseId: string) {
  const base = await prisma.base.findUniqueOrThrow({ ... });
  // 检查所属 Space 是否也在垃圾箱
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

### 5.3 Field 恢复时的配置有效性校验

字段恢复后校验 Link/Lookup/Rollup 引用链是否完整：

```typescript
private async isFieldConfigurationValid(tableId, field) {
  // Lookup 字段：校验引用的 Link 字段是否存在
  if (field.lookupOptions && !field.isConditionalLookup) {
    const lookupValid = await this.validateLookupField(field);
    if (!lookupValid) return false;
    // Rollup 额外校验聚合函数
    if (field.type === FieldType.Rollup) {
      return await this.validateRollupAggregation(field);
    }
  }
  // Conditional Lookup / Conditional Rollup 校验
  if (field.isConditionalLookup) {
    return await this.validateConditionalLookup(tableId, field);
  }
  return true;
}
```

校验失败的字段标记 `hasError: true`，UI 显示错误状态，但不阻止恢复。

---

## 6. V1/V2 分流机制

### 6.1 分流决策点

```typescript
// apps/nestjs-backend/src/features/trash/trash.controller.ts:45
@Post('restore/:trashId')
async restoreTrash(
  @Param('trashId') trashId: string,
  @Res({ passthrough: true }) response: Response
): Promise<void> {
  // ⚠️ 先做 V2 决策
  await this.prepareRestoreTableCanary(trashId, response);

  if (this.cls.get('useV2')) {
    return await this.trashService.restoreTrashV2(trashId);  // V2 路径
  }
  return await this.trashService.restoreTrash(trashId);     // V1 路径
}
```

### 6.2 决策逻辑

```typescript
// apps/nestjs-backend/src/features/trash/trash.controller.ts:72
protected async prepareRestoreTableCanary(trashId: string, response: Response) {
  const decision = await this.trashService.getRestoreTableV2Decision(trashId);
  if (!decision) return;

  this.cls.set('useV2', decision.useV2);
  this.cls.set('v2Feature', TrashController.restoreTableV2Feature);
  this.cls.set('v2Reason', decision.reason);

  // 响应头返回决策结果
  response.setHeader(X_TEABLE_V2_HEADER, decision.useV2 ? 'true' : 'false');
  response.setHeader(X_TEABLE_V2_FEATURE_HEADER, TrashController.restoreTableV2Feature);
  response.setHeader(X_TEABLE_V2_REASON_HEADER, decision.reason);
}
```

### 6.3 决策核心：`CanaryService`

```typescript
// apps/nestjs-backend/src/features/trash/trash.service.ts:674
async getRestoreTableV2Decision(trashId: string) {
  const trash = await this.prismaService.txClient().trash.findUnique({
    where: { id: trashId },
    select: { resourceId: true, resourceType: true, parentId: true },
  });

  const baseId = trash.parentId;
  const base = await this.prismaService.txClient().base.findUnique({
    where: { id: baseId, deletedTime: null },
    select: { spaceId: true, v2Enabled: true },
  });

  // ⚠️ 决策交给 CanaryService
  const decision = await this.canaryService.shouldUseV2ForBaseWithReason(
    base,
    'restoreTable'  // 功能维度
  );
  return { ...decision, baseId, tableId: trash.resourceId };
}
```

**决策依据**：
1. Base 的 `v2Enabled` 标志（数据库配置）
2. 灰度配置（按百分比、按用户名单、按环境等）
3. 功能维度（`restoreTable`）单独的灰度策略

---

## 7. Trash 表的完整生命周期

### 7.1 Trash 表写入（V1 Table 删除）

```typescript
// V2 领域事件投影：TableTrashed 事件触发
@ProjectionHandler(TableTrashed)
export class V2TableTrashedProjection implements IEventHandler<TableTrashed> {
  async handle(context, event) {
    const db = container.resolve<Kysely<IAttachmentsTableDb>>(v2MetaDbTokens.db);
    const table = await db
      .selectFrom('table_meta')
      .where('id', '=', event.tableId.toString())
      .select(['base_id', 'deleted_time'])
      .executeTakeFirst();

    // 先删除可能存在的旧条目（唯一键冲突保护）
    await db
      .deleteFrom('trash')
      .where('resource_id', '=', event.tableId.toString())
      .where('resource_type', '=', ResourceType.Table)
      .execute();

    // 写入新条目
    await db
      .insertInto('trash')
      .values({
        id: nanoid(),
        resource_id: event.tableId.toString(),
        resource_type: ResourceType.Table,
        parent_id: table.base_id,
        deleted_time: table.deleted_time,
        deleted_by: context.actorId.toString(),
      })
      .execute();
  }
}
```

### 7.2 Trash 表清理（V2 Table 恢复）

```typescript
@ProjectionHandler(TableRestored)
export class V2TableRestoredProjection implements IEventHandler<TableRestored> {
  async handle(_context, event) {
    const db = container.resolve<Kysely<IAttachmentsTableDb>>(v2MetaDbTokens.db);
    // TableRestored 事件触发清理 trash 表条目
    await db
      .deleteFrom('trash')
      .where('resource_id', '=', event.tableId.toString())
      .where('resource_type', '=', ResourceType.Table)
      .execute();
  }
}
```

### 7.3 永久删除流程

```typescript
// trash.service.ts:362
async permanentDeleteTables(baseId: string, tableIds: string[]) {
  // 1. 解除 Link 关联
  await this.detachLink(tableId);
  // 2. DROP TABLE 物理删除数据表
  await this.dropTables(tableIds);
  // 3. 清理任务关联数据
  await this.cleanTaskRelatedData(tableIds);
  // 4. 清理所有元数据（field/view/ops/trash/tableTrash/recordTrash）
  await this.cleanTablesRelatedData(baseId, tableIds);
}
```

---

## 8. 核心设计模式总结

| 模式 | 实现 | 场景 |
|------|------|------|
| **统一软删除标记** | `deletedTime: DateTime?` | Space/Base/Table/Field/View（元数据级） |
| **⚠️ 物理删除 + 快照恢复** | 直接 `DELETE` + `table_trash`/`record_trash` 双表快照 | Record（数据级，避免膨胀） |
| **deletedTime 时间戳对齐** | 删除时 table/field/view 共享精确 `deletedTime` | V1 批量级联恢复的匹配依据 |
| **⚠️ 快照版本匹配** | `createdTime <= trashItem.createdTime` + 最新匹配 | 同一记录多次删除-恢复的版本选择 |
| **双表联动快照** | `table_trash`（索引）+ `record_trash`（内容） | 记录删除快照的高效存储与查询 |
| **原子垃圾清理** | 事务中同时删除 `record_trash` + `table_trash` | 恢复成功后清理快照，避免残留 |
| **拓扑排序** | Kahn 算法 + 字段依赖图（Lookup→Link，Rollup→Lookup） | Field 恢复时的创建顺序 |
| **递归 CTE** | `assertParentNotTrashed()` 沿 `parent_id` 向上遍历 | 父级链路完整性校验 |
| **⚠️ CanaryService 灰度分流** | `shouldUseV2ForBaseWithReason()` | V1/V2 架构迁移路径决策 |
| **CQRS 投影** | 领域事件 → 投影处理器异步读写分离 | Trash 表写入与清理 |
| **流式批量** | `RestoreRecordsStreamHandler` 异步生成器 | 大批量记录恢复的渐进式处理 |
| **引用恢复与校验** | `restoreReference()` + `isFieldConfigurationValid()` | Link/Lookup/Rollup 字段恢复后的完整性校验 |

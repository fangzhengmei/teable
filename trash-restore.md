# Teable Trash & Restore — 代码路径全追踪

## 设计双原则

Teable 对"可删除实体"分两类，走**完全不同的删除/恢复路径**：

| 原则 | 适用实体 | 标记方式 | 恢复方式 |
|------|---------|----------|---------|
| **统一软删除** `deletedTime: DateTime?` | Space、Base、TableMeta、Field、View | `null` = 活跃，非空 = 已删除 | 清空 `deletedTime` 即可恢复 |
| **物理删除 + 快照恢复** | Record（数据表行） | 不设标记，直接 `DELETE FROM` | 从 `table_trash` + `record_trash` 快照重新 `INSERT` |

---

## 一、统一软删除标记：`deletedTime: DateTime?`

### 1.1 Schema 定义（`packages/db-main-prisma/prisma/template.prisma`）

五个模型共用同一字段：

```prisma
model Space {                          // :22
  deletedTime  DateTime? @map("deleted_time")
}

model Base {                           // :56
  deletedTime  DateTime?   @map("deleted_time")
}

model TableMeta {                      // :82
  deletedTime  DateTime?   @map("deleted_time")
  @@index([baseId, deletedTime])       // 恢复时按 baseId + deletedTime 精确匹配
}

model Field {                          // :156
  deletedTime  DateTime? @map("deleted_time")
  @@index([tableId, deletedTime])      // 恢复时按 tableId + deletedTime 精确匹配
}

model View {                           // :286
  deletedTime  DateTime? @map("deleted_time")
  @@index([tableId, deletedTime])
}
```

语义完全统一：

- `deletedTime IS NULL` → 活跃（查询时加 `where: { deletedTime: null }`）
- `deletedTime IS NOT NULL` → 已软删除
- TableMeta 额外有 `permanentDeletedTime` 区分软删除与永久删除

### 1.2 删除：同一 `deletedTime` 时间戳原子写入

当删除一个 Table 时，Table / Field / View 在**同一事务**中被赋予**同一个 `new Date()` 时间戳**：

```
table-open-api.service.ts :500-527  deleteTable()

  await this.detachLink(tableId);           // 先解 Link 关联

  this.prismaService.$tx(async (prisma) => {
    const deletedTime = new Date();         // ← 唯一时间戳

    await this.tableService.deleteTable(baseId, tableId, deletedTime);
    //   → table_meta: { deletedTime, provisionState: 'deleting' }

    await prisma.field.updateMany({
      where: { tableId, deletedTime: null },  // 只更新当前活跃字段
      data:  { deletedTime },                 // 同一时间戳
    });

    await prisma.view.updateMany({
      where: { tableId, deletedTime: null },
      data:  { deletedTime },
    });
  });
```

`tableService.deleteTable` 内部（`table.service.ts :287-320`）：

```
  tableMeta.findFirst({ where: { id, baseId, deletedTime: null } })  // 确认活跃
  tableMeta.update({
    data: { version + 1, deletedTime, provisionState: 'deleting', lastModifiedBy }
  })
  saveRawOps(baseId, RawOpType.Del, IdPrefix.Table, [{ docId: tableId, version }])
  //   ↑ 关键：saveRawOps 只写操作日志，不直接写 trash 表
```

#### 1.2.1 Trash 索引表的写入链路（V1 异步事件驱动）

**⚠️ 关键修正：V1 路径中 `trash` 表不是在 `deleteTable` 中直接写入**。

完整链路经过三层中转：

```
tableService.deleteTable()
  → saveRawOps(baseId, RawOpType.Del, IdPrefix.Table, ...)
    → 写入 raw_ops 表（操作日志）
    → share-db.service.ts :80  bindAfterTransaction()
        → 事务提交后触发 eventEmitterService.ops2Event(ops)
          → event-emitter.service.ts :85  ops2Event()
            → rawOpType=Del + docId 前缀=IdPrefix.Table
              → 映射为 Events.TABLE_DELETE
              → 发射 TableDeleteEvent
                → trash.listener.ts :20  @OnEvent(Events.TABLE_DELETE, { async: true })
                    → TrashListener.onEvent()
                      → 从 DB 查出 deletedTime（因为事件是异步的，此时事务已提交）
                      → prisma.trash.create({
                          resourceId: tableId,
                          resourceType: ResourceType.Table,
                          parentId: table.baseId,
                          deletedTime: table.deletedTime,
                          deletedBy: user.id,
                        })
```

**关键细节**：
1. `TrashListener` 监听 5 种事件：`SPACE_DELETE` / `BASE_DELETE` / `TABLE_DELETE` / `APP_DELETE` / `WORKFLOW_DELETE`（`trash.listener.ts :18-22`）
2. 事件是 `{ async: true }` 异步监听，**在事务提交之后执行**——因此能查到 `deletedTime`
3. 如果 `payload.permanent === true`（永久删除），监听器直接 `return`，不写 `trash` 索引
4. 监听器从 DB 重新查出 `deletedTime`（而非从事件 payload 传递），保证时间戳准确
5. `parentId` 根据资源类型不同取值：Space 无父级，Base 取 `spaceId`，Table/App/Workflow 取 `baseId`

### 1.3 恢复：按同一 `deletedTime` 精确匹配批量恢复

恢复时从 `trash` 表取出删除时的精确时间戳，只恢复"那一次删除"的子资源：

```
table-open-api.service.ts :529-564  restoreTable()

  this.prismaService.$tx(async (prisma) => {
    // 从 trash 索引表取回删除时间戳
    const { deletedTime } = await prisma.trash.findFirstOrThrow({
      where: { resourceId: tableId, resourceType: ResourceType.Table },
    });

    if (!deletedTime) throw 'not in trash';

    await this.tableService.restoreTable(baseId, tableId);
    //   → table_meta: { deletedTime: null, provisionState: 'ready' }

    // 只恢复「那一次删除」的字段——精确匹配 deletedTime
    await prisma.field.updateMany({
      where: { tableId, deletedTime },        // 精确匹配！
      data:  { deletedTime: null },
    });

    await prisma.view.updateMany({
      where: { tableId, deletedTime },
      data:  { deletedTime: null },
    });
  });
```

`tableService.restoreTable` 内部（`table.service.ts :322-344`）：

```
  tableMeta.findFirst({ where: { id, baseId, deletedTime: { not: null } } })
  tableMeta.update({
    data: { version + 1, deletedTime: null, provisionState: 'ready', lastModifiedBy }
  })
```

**为什么不用 `deletedTime: { not: null }` 全量恢复？**
表格可能经历多次删除→恢复→再删除，每批有不同的 `deletedTime`。精确匹配确保只恢复"这一批次"被删除的字段和视图。

### 1.4 Space / Base 的软删除恢复

Space 恢复最直接：

```
trash.service.ts :569-577  restoreSpace()

  permissionService.validPermissions(spaceId, ['space|create'])
  space.update({ where: { id: spaceId }, data: { deletedTime: null } })
```

Base 恢复多一步**父级 Space 冲突检测**：

```
trash.service.ts :579-612  restoreBase()

  permissionService.validPermissions(baseId, ['base|create'])

  const base = await prisma.base.findUniqueOrThrow({ ... })
  const trashedSpace = await prisma.trash.findFirst({
    where: { resourceId: base.spaceId, resourceType: TrashType.Space }
  })
  if (trashedSpace != null) {
    throw 'Unable to restore this base because its parent space is also trashed'
  }

  prisma.base.update({ where: { id: baseId }, data: { deletedTime: null } })
  performanceCacheService.del(cacheKey)     // 清缓存
```

### 1.5 View 的软删除恢复

View 恢复也是直接清空 `deletedTime`：

```
view.service.ts :237-251  restoreView()

  this.prismaService.$tx(async () => {
    await this.prismaService.txClient().view.update({
      where: { id: viewId },
      data:  { deletedTime: null },
    })
    // 更新 lastModifiedTime
    await this.updateViewByOps(tableId, viewId, [ops])
  })
```

---

## 二、物理删除 + 快照恢复：Record

### 2.1 为什么 Record 不走软删除

数据表可能有百万/千万级行记录。如果每条记录加 `deletedTime` 软删除标记：
- 数据膨胀严重（大量 `deletedTime IS NOT NULL` 的行长期占用空间）
- 查询需要额外 `WHERE deleted_time IS NULL`，影响全表扫描性能
- 索引膨胀

因此 Record 采用**物理删除** + **快照持久化**恢复的方案。

### 2.2 快照存储结构（`packages/db-data-prisma/prisma/schema.prisma`）

两张表各司其职：

```prisma
// 操作索引表：每次删除操作一条记录
model TableTrash {                            // :127
  id           String   @id @default(cuid())
  tableId      String   @map("table_id")
  resourceType String   @map("resource_type")   // "view" | "field" | "record"
  snapshot     String   @map("snapshot")          // JSON 快照
  createdTime  DateTime @default(now()) @map("created_time")
  createdBy    String   @map("created_by")
  @@index([tableId])
  @@map("table_trash")
}

// 记录快照内容表：每条被删记录一条
model RecordTrash {                           // :139
  id          String   @id @default(cuid())
  tableId     String   @map("table_id")
  recordId    String   @map("record_id")         // 原记录 ID
  snapshot    String   @map("snapshot")           // 完整记录 JSON
  createdTime DateTime @default(now()) @map("created_time")
  createdBy   String   @map("created_by")
  @@index([tableId, recordId])
  @@map("record_trash")
}
```

**快照内容差异**：

| resourceType | table_trash.snapshot | record_trash.snapshot |
|---|---|---|
| Record | `JSON.stringify(recordIds[])` — 仅 ID 列表 | `JSON.stringify(fullRecord)` — 完整记录含 fields、version、orders 等 |
| Field | `JSON.stringify({ fields, records })` — 字段定义 + 关联记录 | — |
| View | `JSON.stringify([viewId])` — 仅 ID | — |

### 2.3 V1 记录删除完整链路

```
record-delete.service.ts :30-88  deleteRecords()

  ① 查快照：recordService.getRecordsById(tableId, recordIds)
  ② 处理 Link 关联：linkService.getDeleteRecordUpdateContext()
  ③ 获取行序（恢复用）：recordService.getRecordIndexes()
  ④ 物理删除：recordService.batchDeleteRecords(tableId, recordIds)
     → record.service.ts :1123-1158
       1. SELECT __id, __version WHERE __id IN (recordIds)   // 乐观锁
       2. saveRawOps(RawOpType.Del)                          // 写操作日志
       3. batchDel(tableId, recordIds)                       // DELETE FROM table
  ⑤ 异步发事件：emitAsync(Events.OPERATION_RECORDS_DELETE, {
       operationId: generateOperationId(),
       records: [...with orders],
     })
```

事件监听器写入快照：

```
table-trash.listener.ts :19-60  @OnEvent(OPERATION_RECORDS_DELETE)

  const createdTime = new Date();                // 统一时间戳

  dataPrismaService.$tx(async (prisma) => {
    // 1. 写 table_trash 索引（一条）
    prisma.tableTrash.create({
      id: operationId,
      resourceType: ResourceType.Record,
      snapshot: JSON.stringify(recordIds),       // 仅 ID 列表
      createdTime,
    })

    // 2. 写 record_trash 快照（批量，每批 5000）
    for (batch of records.slice(i, i+5000)) {
      prisma.recordTrash.createMany({
        data: batch.map(record => ({
          recordId: record.id,
          snapshot: JSON.stringify(record),       // 完整记录
          createdTime,                            // 同一时间戳
        }))
      })
    }
  })
```

### 2.4 V2 记录删除链路

V2 走领域事件 + 投影：

```
DeleteRecordsHandler.ts :66-212  handle()

  ① tableRecordRepository.deleteMany()          // 物理删除
  ② 构建 RecordsDeleted 领域事件（携带 recordSnapshots）
  ③ eventBus.publishMany()
```

投影处理器写入快照：

```
v2-table-trash.service.ts :48-114  V2RecordsDeletedTableTrashProjection

  event.recordSnapshots → 转换为 IDeleteRecordsPayload 格式
  → v2RecordTrashService.persistDeletedRecords()

v2-record-trash.service.ts :53-106  persistDeletedRecords()

  db.transaction().execute(async (trx) => {
    trx.insertInto('table_trash').values({       // 一条索引
      id: operationId,
      snapshot: JSON.stringify(recordIds),
      created_time: createdTime,
    })

    for (batch) {
      trx.insertInto('record_trash').values(     // 批量快照
        batch.map(record => ({
          record_id: record.id,
          snapshot: JSON.stringify(record),
          created_time: createdTime,             // 同一时间戳
        }))
      )
    }
  })
```

### 2.5 V1 记录恢复 + 垃圾清理

```
trash.service.ts :822-904  restoreTableResource() → case Record

  ① 取出 trash 条目的 snapshot（记录 ID 列表）和 createdTime
  ② 从 record_trash 查所有可能快照：
     recordTrash.findMany({
       where: { tableId, recordId: { in: recordIds } },
       orderBy: [{ recordId: 'asc' }, { createdTime: 'desc' }, { id: 'desc' }],
     })
  ③ 快照版本匹配（核心算法）：
     recordTrashRows.reduce((acc, row) => {
       // 只取 createdTime <= 当前 trash 操作时间 且未被匹配过的
       if (row.createdTime <= createdTime && !acc.has(row.recordId)) {
         acc.set(row.recordId, row);
       }
       return acc;
     }, new Map())
  ④ V1 路径 → multipleCreateRecords() 重新 INSERT
  ⑤ 原子清理快照：
     dataPrismaService.$tx(async (prisma) => {
       prisma.recordTrash.deleteMany({ where: { id: { in: matchedIds } } })
       prisma.tableTrash.delete({ where: { id: trashId } })
     }, { timeout: bigTransactionTimeout })
```

**为什么需要快照版本匹配？**
同一 `recordId` 可能经历：删除 → 恢复 → 再删除。`record_trash` 中会存在同一 `recordId` 的多条快照。恢复时必须取 `createdTime <= 当前trash条目.createdTime` 的最新一条，避免恢复到错误版本。

### 2.6 V2 记录恢复 + 垃圾清理

```
RestoreRecordsHandler.ts :55-112  handle()

  for (batch of restoreRecordBatches(records, batchSize)) {
    const records = buildTableRecords(table, batch)
    const restoreRecordsById = buildRestoreRecordsById(batch)  // 系统列元数据

    unitOfWork.withTransaction(context, async (txCtx) => {
      tableRecordRepository.insertMany(txCtx, table, records, {
        restoreRecordsById,                                    // 恢复原始 version/orders/autoNumber 等
        cleanupTrashRecordIds: batch.map(r => r.recordId),     // ← Repository 内部清理快照
      })
    })

    eventBus.publishMany(context, batchEvents)  // RecordsBatchCreated
  }
```

`buildRestoreRecordsById` 保留系统列（`RestoreRecordsHandler.ts :156-172`）：

```
  new Map(batch.map(record => [
    record.recordId,
    {
      version, orders, autoNumber,
      createdTime, createdBy,
      lastModifiedTime, lastModifiedBy,
      extraColumnValues,
    }
  ]))
```

#### 2.6.1 V2 `cleanupTrashRecordIds` 的实际实现（Repository 层）

**⚠️ 关键修正：V2 清理只删除 `record_trash`，不删除 `table_trash`。**

`PostgresTableRecordRepository.insertMany()` 内部（`adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts :1476`）：

```
  if (options?.cleanupTrashRecordIds?.length) {
    await cleanupRestoredRecordTrash(
      db,
      table.id().toString(),
      options.cleanupTrashRecordIds    // recordId[]
    );
  }
```

`cleanupRestoredRecordTrash` 函数实现（`:192-206`）：

```
const cleanupRestoredRecordTrash = async (
  db: Kysely<DynamicDB> | Transaction<DynamicDB>,
  tableId: string,
  recordIds: ReadonlyArray<string>
): Promise<void> => {
  if (recordIds.length === 0) return;

  const restoredRecordIds = Array.from(new Set(recordIds));  // 去重
  await db
    .deleteFrom('record_trash')              // ← 只删 record_trash
    .where('table_id', '=', tableId)
    .where('record_id', 'in', restoredRecordIds)  // ← 按 record_id 匹配，非按快照行 ID
    .execute();
};
```

**V2 清理 vs V1 清理的关键差异**：
- V1 按**快照行 ID**（`record_trash.id`）精确删除匹配的快照条目
- V2 按**原记录 ID**（`record_trash.record_id`）删除该记录在 `record_trash` 中的所有快照
- V2 不删 `table_trash`——`table_trash` 的清理由上层调用方（`trash.service`）在恢复流程结束后处理

### 2.7 V1 vs V2 垃圾清理对比

| | V1 | V2 |
|---|---|---|
| **清理位置** | `trash.service` 显式调用 | `PostgresTableRecordRepository.insertMany()` 内部 |
| **参数** | 匹配后的 `matchedRecordTrashRowIds`（快照行 ID） | `cleanupTrashRecordIds: recordId[]`（原记录 ID） |
| **匹配方式** | `record_trash.id IN (...)` — 按快照行 ID 精确删除 | `record_trash.record_id IN (...)` — 按原记录 ID 删除所有快照 |
| **事务** | `trash.service` 开启 `dataPrismaService.$tx` | Repository 层 `unitOfWork.withTransaction` |
| **清理内容** | `record_trash` + `table_trash` 在同一事务中一起删 | **仅** `record_trash`；`table_trash` 由上层清 |
| **大事务保护** | `bigTransactionTimeout` | 由 UnitOfWork 管理 |

---

## 三、表内资源恢复（View / Field / Record）

入口在 `trash.service.ts :759-904  restoreTableResource(trashId)`，按 `resourceType` 分发：

```
if (trashId.startsWith('opr'))  →  restoreTableResource()    // 表内资源
else                            →  restoreResource()         // Space/Base/Table
```

### 3.1 View 恢复

```
case TableTrashType.View:
  viewService.restoreView(tableId, snapshot[0])
  → view.update({ where: { id: viewId }, data: { deletedTime: null } })
  → 更新 lastModifiedTime
```

恢复完删 table_trash 条目：

```
dataPrismaService.tableTrash.delete({ where: { id: trashId } })
```

### 3.2 Field 恢复（重新创建 + 依赖排序）

Field 恢复走"重新创建"而非清 `deletedTime`——因为字段删除后其数据列可能已被物理清理。

```
case TableTrashType.Field:
  const { fields, records } = snapshot

  // 1. 重新创建字段（内部做拓扑排序）
  fieldOpenApiService.createFields(tableId, fields)

  // 2. 更新仍存在的记录中的关联字段数据
  if (records) {
    const existingSnapshots = recordService.getSnapshotBulk(tableId, records.map(r => r.id))
    const existingIdSet = new Set(existingSnapshots.map(s => s.data.id))
    const filteredRecords = records.filter(r => existingIdSet.has(r.id))
    if (filteredRecords.length) {
      recordOpenApiService.updateRecords(tableId, { fieldKeyType: Id, records: filteredRecords })
    }
  }
```

**依赖排序**（`field-open-api.service.ts` `sortCreateFieldsByDependencies`）：
- 解析每个字段的引用依赖（Lookup→Link，Rollup→Lookup）
- Kahn 拓扑排序确定创建顺序
- 检测循环依赖，有环则回退原始顺序

**引用恢复**（`field-open-api.service.ts` `restoreReference`）：
- 恢复后查找引用字段
- `isFieldConfigurationValid()` 校验 Link/Lookup/Rollup 引用链完整性
- 失败标记 `hasError: true`，不阻止恢复

恢复完删 table_trash 条目。

### 3.3 Record 恢复

见上方 §2.5（V1）和 §2.6（V2）。

---

## 四、顶层恢复入口与 V1/V2 分流

### 4.1 Controller 入口

```
trash.controller.ts :45-56

  @Post('restore/:trashId')
  async restoreTrash(trashId, response) {
    await this.prepareRestoreTableCanary(trashId, response)  // V2 决策
    if (this.cls.get('useV2')) {
      return trashService.restoreTrashV2(trashId)
    }
    return trashService.restoreTrash(trashId)
  }
```

### 4.2 V2 决策：`CanaryService`

```
trash.controller.ts :72-85  prepareRestoreTableCanary()

  const decision = trashService.getRestoreTableV2Decision(trashId)
  if (!decision) {
    return;                              // ← 不设 cls.useV2，走 V1 路径
  }
  cls.set('useV2', decision.useV2)
  response.setHeader('X-Teable-V2', decision.useV2)
  response.setHeader('X-Teable-V2-Reason', decision.reason)
```

```
trash.service.ts :674-714  getRestoreTableV2Decision()

  if (trashId.startsWith('opr')) return undefined     // 表内资源不走 V2 Table 路径

  const trash = prisma.trash.findUnique({ id: trashId })
  if (!trash || trash.resourceType !== Table) return undefined  // 只有 Table 走分流

  const baseId = trash.parentId
  if (!baseId) return { useV2: false, reason: 'disabled', baseId: '', tableId }  // 无父级

  const base = prisma.base.findUnique({
    where: { id: baseId, deletedTime: null },
    select: { spaceId: true, v2Enabled: true }
  })
  if (!base?.spaceId) return { useV2: false, reason: 'disabled', baseId, tableId }  // Base 无 space

  const decision = canaryService.shouldUseV2ForBaseWithReason(base, 'restoreTable')
  return { ...decision, baseId, tableId }
```

#### 4.2.1 V2 分支边界条件与影响链

**⚠️ 关键修正：`getRestoreTableV2Decision` 返回 `undefined` 与返回 `{ useV2: false }` 是完全不同的路径。**

| 决策结果 | 触发条件 | `cls.useV2` | 实际走的路径 | 影响 |
|----------|----------|-------------|-------------|------|
| `undefined` | `trashId` 以 `opr` 开头（表内资源）| 不设（保持原值） | V1 `restoreTrash()` → `restoreTableResource()` | 正确：表内资源恢复无 V2 路径 |
| `undefined` | `trash` 记录不存在，或 `resourceType !== Table` | 不设 | V1 `restoreTrash()` → `restoreResource()` | 正确：Space/Base 走 V1 |
| `{ useV2: false }` | `baseId` 为空，或 Base 无 `spaceId` | `false` | V1 `restoreTrash()` | 数据异常场景，安全回退 |
| `{ useV2: false }` | CanaryService 灰度决策为 V1 | `false` | V1 `restoreTrash()` | 正常灰度回退 |
| `{ useV2: true }` | CanaryService 灰度决策为 V2 | `true` | V2 `restoreTrashV2()` | 正常 V2 路径 |

**`restoreTrashV2` 内部的二次决策**（`trash.service.ts :716-728`）：

```
  async restoreTrashV2(trashId: string) {
    const decision = await this.getRestoreTableV2Decision(trashId);
    if (!decision) {
      // ⚠️ 如果 controller 层判断 useV2=true 但 service 层二次查询返回 undefined
      // （理论上不应发生，但可能是并发删除导致 trash 记录消失）
      throw new CustomHttpException(
        `The trash ${trashId} not found`, HttpErrorCode.NOT_FOUND
      );
    }
    await this.assertParentNotTrashed(decision.baseId);
    await this.restoreTableV2(decision.baseId, decision.tableId);
  }
```

**潜在风险**：如果 `cls.useV2` 在之前的请求中被设为 `true`（CLS 是请求级隔离的，但需确认），且当前请求的 `getRestoreTableV2Decision` 返回 `undefined`，则 `cls.useV2` 不会被显式设为 `false`。但由于 NestJS CLS 是请求级作用域，每个请求开始时 `useV2` 默认为 `undefined`（falsy），因此不会误入 V2 路径。

**决策依据**：Base 的 `v2Enabled` 字段 + CanaryService 灰度配置（百分比/用户名单/环境）。

### 4.3 V1 恢复路径

```
trash.service.ts :972-1007  restoreTrash()

  if (trashId.startsWith('opr')) → restoreTableResource()  // 表内资源

  prismaService.$tx(async (prisma) => {
    const trash = prisma.trash.findUniqueOrThrow({ id: trashId })
    assertParentNotTrashed(trash.parentId)    // 递归 CTE 检查父级链
    restoreResource({ resourceType, resourceId })
    prisma.trash.deleteMany({ id: trashId })  // 清理 trash 索引
  })
```

### 4.4 V2 恢复路径

```
trash.service.ts :716-728  restoreTrashV2()

  const decision = getRestoreTableV2Decision(trashId)
  assertParentNotTrashed(decision.baseId)
  restoreTableV2(decision.baseId, decision.tableId)
    → tableOpenApiV2Service.restoreTable()
      → RestoreTableCommand → RestoreTableHandler
        → tableQueryService.getDeletedByIdInBase()    // 查找已删除的 Table 聚合
        → unitOfWork.withTransaction(tableRepository.restore)
        → table.markRestored()                         // 领域事件 TableRestored
        → eventBus.publishMany()                       // 投影清理 trash 索引
```

V2 Table 恢复的投影清理：

```
v2-table-trash.service.ts :216-234  V2TableRestoredProjection

  db.deleteFrom('trash')
    .where('resource_id', '=', event.tableId)
    .where('resource_type', '=', ResourceType.Table)
    .execute()
```

---

## 五、冲突检测：`assertParentNotTrashed`

核心拦截——恢复前用**递归 CTE**检查整个父级链路：

```
trash.service.ts :614-651

  knex.withRecursive('parent_chain', (qb) => {
    // 基础：检查直接父级是否在 trash
    qb.select('resource_id', 'parent_id')
      .from('trash')
      .where('resource_id', parentId)
    // 递归：沿 parent_id 继续向上
    .unionAll((qb) => {
      qb.select('t.resource_id', 't.parent_id')
        .from('trash as t')
        .join('parent_chain as pc', 't.resource_id', 'pc.parent_id')
        .whereNotNull('pc.parent_id')
    })
  })
  .select('resource_id').from('parent_chain').limit(1)

  if (result.length > 0) throw 'parent is also in trash'
```

**拦截场景**：
- 恢复 Table → 所属 Base 在 trash → 拦截
- 恢复 Base → 所属 Space 在 trash → 拦截
- 恢复 Table → Base 不在 trash 但 Base 的 Space 在 trash → 递归发现 → 拦截

Base 恢复还有一个非递归快捷路径（`trash.service.ts :588-602`）直接查 trash 表检查 Space。

---

## 六、Trash 索引表生命周期

`Trash` 表（主库）是顶层资源的删除索引：

```prisma
model Trash {
  id           String   @id @default(cuid())
  resourceType String   @map("resource_type")    // "space" | "base" | "table"
  resourceId   String   @map("resource_id")
  parentId     String?  @map("parent_id")          // 父资源 ID
  deletedTime  DateTime @default(now()) @map("deleted_time")
  deletedBy    String   @map("deleted_by")
  @@unique([resourceType, resourceId])
}
```

| 事件 | 写入 | 清理 |
|------|------|------|
| Space 删除（V1） | `TrashListener` 监听 `SPACE_DELETE` 事件异步写入 | `restoreTrash()` 中 `prisma.trash.deleteMany({ id: trashId })` |
| Base 删除（V1） | `TrashListener` 监听 `BASE_DELETE` 事件异步写入 | `restoreTrash()` 中 `prisma.trash.deleteMany({ id: trashId })` |
| Table 删除（V1） | `saveRawOps` → `ops2Event` → `TrashListener` 监听 `TABLE_DELETE` 异步写入 | `restoreTrash()` 中 `prisma.trash.deleteMany({ id: trashId })` |
| App 删除（V1） | `TrashListener` 监听 `APP_DELETE` 事件异步写入 | `restoreTrash()` 中 `prisma.trash.deleteMany({ id: trashId })` |
| Workflow 删除（V1） | `TrashListener` 监听 `WORKFLOW_DELETE` 事件异步写入 | `restoreTrash()` 中 `prisma.trash.deleteMany({ id: trashId })` |
| Table 删除（V2） | `V2TableTrashedProjection` 写入 | `V2TableRestoredProjection` 删除 |
| 永久删除 | 不写（`TrashListener` 检查 `permanent` 标志直接 return） | `spaceService.permanentDeleteSpace` / `baseService.permanentDeleteBase` / `tableOpenApiService.permanentDeleteTables` 内清理 |

---

## 七、永久删除与垃圾清理

### 7.1 永久删除入口

```
trash.service.ts :1131-1203  delete()

  prisma.trash.findUniqueOrThrow({ id: trashId })
  deleteResource(trash)
    case Space → spaceService.permanentDeleteSpace()
    case Base  → baseService.permanentDeleteBase()
    case Table → tableOpenApiService.permanentDeleteTables(baseId, [resourceId])
```

### 7.2 表内资源永久清理

```
trash.service.ts :1060-1129  resetTableTrashItems()

  // 收集所有 table_trash 中的 view/field/record ID
  tableTrash.findMany({ where: { tableId } })

  // 主库事务：物理删除 view、field、taskReference、ops
  prisma.$tx(async () => {
    view.deleteMany({ id: { in: deletedViewIds } })
    field.deleteMany({ id: { in: deletedFieldIds } })
    taskReference.deleteMany(...)
    ops.deleteMany(...)
  })

  // 数据库事务：清理快照表
  dataPrismaService.$tx(async () => {
    recordTrash.deleteMany({ tableId })
    tableTrash.deleteMany({ tableId })
  })
```

### 7.3 Table 永久删除

```
table-open-api.service.ts  permanentDeleteTables()

  1. detachLink()                   // 解除 Link 关联
  2. dropTables()                   // DROP TABLE 物理删除数据表
  3. cleanTaskRelatedData()         // 清理任务引用
  4. cleanTablesRelatedData()       // 清理 field/view/ops/trash/recordTrash
```

---

## 八、完整调用链速查

```
┌─ 删除 ──────────────────────────────────────────────────────────┐
│                                                                  │
│  Table 删除 (V1)                                                 │
│    deleteTable() → detachLink → $tx {                            │
│      deletedTime = new Date()                                    │
│      tableService.deleteTable(baseId, tableId, deletedTime)      │
│      field.updateMany({ deletedTime })                           │
│      view.updateMany({ deletedTime })                            │
│    } → saveRawOps → shareDb.bindAfterTransaction                │
│      → ops2Event → TABLE_DELETE                                  │
│        → TrashListener (async) → prisma.trash.create()           │
│                                                                  │
│  Space/Base/App/Workflow 删除 (V1)                               │
│    → emit SPACE_DELETE/BASE_DELETE/APP_DELETE/WORKFLOW_DELETE    │
│      → TrashListener (async) → prisma.trash.create()             │
│                                                                  │
│  Record 删除 (V1)                                                │
│    deleteRecords() → $tx {                                       │
│      getRecordsById()              // 保存快照                   │
│      linkService.getDeleteRecordUpdateContext()                  │
│      batchDeleteRecords()          // DELETE FROM               │
│    } → emit OPERATION_RECORDS_DELETE                             │
│      → TableTrashListener                                        │
│        tableTrash.create { snapshot: recordIds }                 │
│        recordTrash.createMany { snapshot: fullRecord }           │
│                                                                  │
│  Record 删除 (V2)                                                │
│    DeleteRecordsHandler → deleteMany() → emit RecordsDeleted     │
│      → V2RecordsDeletedTableTrashProjection                      │
│        → V2RecordTrashService.persistDeletedRecords()            │
│          tableTrash.insertInto + recordTrash.insertInto           │
│                                                                  │
├─ 恢复 ──────────────────────────────────────────────────────────┤
│                                                                  │
│  POST /api/trash/restore/:trashId                                │
│    → prepareRestoreTableCanary()                                 │
│      → getRestoreTableV2Decision(trashId)                        │
│        → undefined: cls.useV2 不设 → V1                         │
│        → { useV2: false }: cls.useV2 = false → V1               │
│        → { useV2: true }:  cls.useV2 = true  → V2               │
│      → canaryService.shouldUseV2ForBaseWithReason(base,          │
│          'restoreTable')                                         │
│                                                                  │
│  Table 恢复 (V1)                                                 │
│    restoreTrash() → $tx {                                        │
│      assertParentNotTrashed(parentId)  // 递归 CTE               │
│      restoreResource() → restoreTable()                          │
│        trash.findFirst → { deletedTime }                         │
│        tableService.restoreTable()  → deletedTime=null           │
│        field.updateMany({ deletedTime }) → deletedTime=null      │
│        view.updateMany({ deletedTime })  → deletedTime=null      │
│      trash.deleteMany({ id: trashId })                           │
│    }                                                             │
│                                                                  │
│  Table 恢复 (V2)                                                 │
│    restoreTrashV2()                                              │
│      getRestoreTableV2Decision() — 二次验证                      │
│        → undefined → throw NOT_FOUND                             │
│      assertParentNotTrashed(baseId)                              │
│      restoreTableV2() → RestoreTableHandler                      │
│        getDeletedByIdInBase → tableRepository.restore()          │
│        table.markRestored() → emit TableRestored                 │
│          → V2TableRestoredProjection → trash.deleteFrom()        │
│                                                                  │
│  Record 恢复 (V1)                                                │
│    restoreTableResource() → case Record                          │
│      recordTrash.findMany()                                      │
│      快照版本匹配 (createdTime <= trash.createdTime)             │
│      multipleCreateRecords()       // INSERT INTO                │
│      $tx { recordTrash.deleteMany(id IN matched)                 │
│           + tableTrash.delete(id = trashId) }                    │
│                                                                  │
│  Record 恢复 (V2)                                                │
│    restoreRecordsV2() → RestoreRecordsCommand                    │
│      → RestoreRecordsHandler                                     │
│        insertMany({ restoreRecordsById,                          │
│                     cleanupTrashRecordIds })                     │
│          → PostgresTableRecordRepository:                        │
│            INSERT INTO table + cleanupRestoredRecordTrash()      │
│            → record_trash.deleteFrom()                           │
│              .where('record_id', 'in', ids)                      │
│              // ⚠️ 只删 record_trash，不删 table_trash          │
│        emit RecordsBatchCreated                                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

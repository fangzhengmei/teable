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
  saveRawOps(baseId, RawOpType.Del, ...)     // 写操作日志
```

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

### 2.7 V1 vs V2 垃圾清理对比

| | V1 | V2 |
|---|---|---|
| **清理位置** | `trash.service` 显式调用 | `TableRecordRepository.insertMany()` 内部 |
| **参数** | 匹配后的 `matchedRecordTrashRowIds` | `cleanupTrashRecordIds: recordId[]` |
| **事务** | `trash.service` 开启 `dataPrismaService.$tx` | Repository 层 `unitOfWork.withTransaction` |
| **清理内容** | `record_trash` + `table_trash` 一起删 | 仅 `record_trash`（`table_trash` 由上层清） |
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
  cls.set('useV2', decision.useV2)
  response.setHeader('X-Teable-V2', decision.useV2)
  response.setHeader('X-Teable-V2-Reason', decision.reason)
```

```
trash.service.ts :674-714  getRestoreTableV2Decision()

  if (trashId.startsWith('opr')) return undefined     // 表内资源不走 V2 Table 路径

  const trash = prisma.trash.findUnique({ id: trashId })
  if (trash.resourceType !== Table) return undefined  // 只有 Table 走分流

  const base = prisma.base.findUnique({
    where: { id: baseId, deletedTime: null },
    select: { spaceId: true, v2Enabled: true }
  })

  const decision = canaryService.shouldUseV2ForBaseWithReason(base, 'restoreTable')
  return { ...decision, baseId, tableId }
```

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
| Table 删除（V1） | 由 `tableService.deleteTable` 后续逻辑写入 | `restoreTrash()` 中 `prisma.trash.deleteMany({ id: trashId })` |
| Table 删除（V2） | `V2TableTrashedProjection` 写入 | `V2TableRestoredProjection` 删除 |
| 永久删除 | — | `spaceService.permanentDeleteSpace` / `baseService.permanentDeleteBase` / `tableOpenApiService.permanentDeleteTables` 内清理 |

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
│    }                                                             │
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
│      $tx { recordTrash.deleteMany + tableTrash.delete }          │
│                                                                  │
│  Record 恢复 (V2)                                                │
│    restoreRecordsV2() → RestoreRecordsCommand                    │
│      → RestoreRecordsHandler                                     │
│        insertMany({ restoreRecordsById, cleanupTrashRecordIds }) │
│        emit RecordsBatchCreated                                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

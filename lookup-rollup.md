# 查找与汇总字段的值传播链路分析

## 一、核心架构概览

Teable 的查找(Lookup)与汇总(Rollup)字段的值传播采用**分层架构**，从记录变更触发到最终派生字段重算，经过以下核心服务协作：

```
记录更新 → 计算编排器(ComputedOrchestrator) → 依赖收集器(DependencyCollector)
                                                          ↓
                                         链接级联解析器(LinkCascadeResolver)
                                                          ↓
                                         计算评估器(ComputedEvaluator) → SQL更新
                                                          ↓
                                         ShareDB实时发布 → 前端缓存失效
```

**关键文件位置：**
- 字段模型: `packages/core/src/models/field/derivate/`
- 计算编排: `apps/nestjs-backend/src/features/record/computed/services/`
- 引用图: `apps/nestjs-backend/src/features/calculation/reference.service.ts`

---

## 二、核心字段模型

### 2.1 汇总字段 (RollupFieldCore)

**文件**: `packages/core/src/models/field/derivate/rollup.field.ts:20-79`

```typescript
export class RollupFieldCore extends FormulaAbstractCore {
  type!: FieldType.Rollup;
  declare options: IRollupFieldOptions;  // 包含 expression (sum/count/avg 等)
  declare lookupOptions: ILookupOptionsVo;  // 跨表引用配置

  // 返回外键表ID，用于跨表依赖追踪
  getForeignTableId(): string | undefined {
    return this.lookupOptions?.foreignTableId;
  }
}
```

**设计要点**:
- 继承自 `FormulaAbstractCore`，复用公式表达式解析能力
- `lookupOptions` 定义跨表引用的三要素：`linkFieldId`(链接字段)、`foreignTableId`(外表)、`lookupFieldId`(目标字段)
- `expression` 定义聚合函数，如 `sum({values})`、`count({values})`

### 2.2 链接字段 (LinkFieldCore)

**文件**: `packages/core/src/models/field/derivate/link.field.ts:34-205`

链接字段是跨表引用的**桥梁**，提供两个关键方法用于发现依赖：

```typescript
// 发现所有引用此链接字段的 Lookup 字段
getLookupFields(tableDomain: TableDomain): FieldCore[] {
  return tableDomain.filterFields(
    (field) =>
      !!field.isLookup &&
      !!field.lookupOptions &&
      'linkFieldId' in field.lookupOptions &&
      field.lookupOptions.linkFieldId === this.id
  );
}

// 发现所有引用此链接字段的 Rollup 字段
getRollupFields(tableDomain: TableDomain): FieldCore[] {
  return tableDomain.filterFields(
    (field) =>
      field.type === FieldType.Rollup &&
      !!field.lookupOptions &&
      'linkFieldId' in field.lookupOptions &&
      field.lookupOptions.linkFieldId === this.id
  );
}
```

### 2.3 查找选项模式 (Lookup Options)

**文件**: `packages/core/src/models/field/lookup-options-base.schema.ts:7-235`

支持两种查找模式：

| 模式 | 判别特征 | 适用场景 |
|------|---------|---------|
| 链接查找 (Link Lookup) | 含 `linkFieldId` | 通过现有链接字段引用 |
| 条件查找 (Conditional Lookup) | 含 `filter` 不含 `linkFieldId` | 基于过滤条件跨表查询 |

---

## 三、值传播触发入口

### 3.1 记录更新触发

**文件**: `apps/nestjs-backend/src/features/record/record-modify/record-update.service.ts:86-147`

```typescript
const ctxs = await this.prismaService.$tx(async () => {
  // ... 字段类型转换、系统字段处理 ...
  
  // 核心：包装基础更新，在同一事务内完成派生字段重算
  await this.computedOrchestrator.computeCellChangesForRecords(
    tableId,
    ctxs,  // ICellContext[]: 包含 recordId, fieldId, oldValue, newValue
    async (tables) => {
      // 1. 处理链接字段衍生更新（对称链接、外键更新）
      const linkDerivate = await this.linkService.planDerivateByLink(...);
      // 2. 执行基础记录更新
      await this.batchService.updateRecords(composedOpsMap, ...);
    }
  );
});
```

**触发场景**:
- 记录创建 (`RecordCreateService`)
- 记录更新 (`RecordUpdateService`)
- 记录删除 (`RecordDeleteService`)
- 字段定义变更（增/删/改）

---

## 四、依赖收集与链路穿透

### 4.1 依赖收集器 (ComputedDependencyCollectorService)

**文件**: `apps/nestjs-backend/src/features/record/computed/services/computed-dependency-collector.service.ts:74-1792`

#### 4.1.1 字段依赖图遍历 (SQL CTE 递归)

```typescript
private async collectDependentFieldsByTable(
  startFieldIds: string[],
  excludeFieldIds?: string[]
): Promise<Record<string, Set<string>>> {
  // WITH RECURSIVE dep_graph AS (
  //   SELECT from_field_id, to_field_id FROM reference WHERE from_field_id IN (...)
  //   UNION
  //   SELECT r.from_field_id, r.to_field_id FROM reference r
  //   JOIN dep_graph d ON r.from_field_id = d.to_field_id
  // )
  // SELECT DISTINCT to_field_id, table_id FROM dep_graph
  // JOIN field ON field.id = dep_graph.to_field_id
}
```

**reference 表结构**:
- `from_field_id`: 被依赖的字段（如链接字段、原始字段）
- `to_field_id`: 依赖字段（如 lookup、rollup、formula）

#### 4.1.2 链接字段穿透 (Lookup Options 查询)

SQL CTE 可能遗漏历史数据，因此采用**双重保障**机制：

```typescript
// 通过 lookup_options->>'linkFieldId' 直接查询引用该链接字段的所有派生字段
private async findLookupsByLinkIds(linkFieldIds: string[]): Promise<Record<string, Set<string>>> {
  const accessor = this.buildLookupOptionsAccessor('linkFieldId');
  // WHERE lookup_options::json->>'linkFieldId' IN (linkFieldIds)
}
```

**穿透路径**:
```
源表A.字段X更新 → 表A.链接字段L → 表B.lookup字段 (lookupOptions.linkFieldId = L.id)
                                               ↓
                                       表B.rollup字段 (依赖lookup)
                                               ↓
                                       表C.跨表引用...
```

#### 4.1.3 邻接图构建

```typescript
private getAdjacencyMaps(tableDomains: ReadonlyMap<string, TableDomain>, projection?: ...) {
  // 链接邻接图: foreignTableId → Set<hostTableId>
  const linkAdj: Record<string, Set<string>> = {};
  // 条件汇总邻接图: foreignTableId → IConditionalRollupAdjacencyEdge[]
  const conditionalRollupAdj: Record<string, IConditionalRollupAdjacencyEdge[]> = {};
  
  for (const [tableId, tableDomain] of tableDomains) {
    for (const field of tableDomain.fieldList) {
      if (field.type === FieldType.Link && !field.isLookup) {
        // 链接字段：建立外键表 → 主表的边
        linkAdj[foreignTableId].add(tableId);
      } else if (field.type === FieldType.ConditionalRollup || field.isConditionalLookup) {
        // 条件汇总/查找：基于过滤条件动态关联
        conditionalRollupAdj[foreignTableId].push({ tableId, fieldId, filter });
      }
    }
  }
}
```

### 4.2 链接级联解析器 (LinkCascadeResolver)

**文件**: `apps/nestjs-backend/src/features/record/computed/services/link-cascade-resolver.ts:38-227`

#### 4.2.1 BFS 遍历算法

```typescript
async resolve(params: IResolveLinkCascadeParams): Promise<Array<{ tableId: string; recordId: string }>> {
  // 队列元素: { tableId, ids?: Set<string>, all: boolean }
  const queue: Array<{ tableId: string; ids?: Set<string>; all: boolean }> = [];
  
  // 种子初始化
  for (const seed of explicitSeeds) {
    visited.set(seed.tableId, new Set(seed.recordIds));
    queue.push({ tableId: seed.tableId, ids: new Set(seed.recordIds), all: false });
  }
  
  // BFS 遍历
  while (queue.length) {
    const { tableId, ids, all } = queue.shift()!;
    const edgesFromTable = edgeBySrc.get(tableId);
    
    for (const edge of edgesFromTable) {
      // 通过 junction table 查询关联记录
      const rows = all
        ? await this.fetchEdgeTargetsFromAll(edge)      // 全表扫描
        : await this.fetchEdgeTargetsBatched(edge, frontierIds);  // 批量查询
      
      // 去重后加入下一轮
      for (const row of rows) {
        if (!dstSet.has(rid)) {
          dstSet.add(rid);
          added.add(rid);
        }
      }
    }
    
    // 新发现的记录继续传播
    for (const [dstTable, newIds] of additionsByTable) {
      queue.push({ tableId: dstTable, ids: newIds, all: false });
    }
  }
}
```

#### 4.2.2 Junction Table 查询

```sql
-- 多对多关系查询示例
SELECT "__self_id"::text as record_id
FROM "junction_table"
WHERE "__foreign_id" IN (recordIds)
  AND "__foreign_id" IS NOT NULL
  AND "__self_id" IS NOT NULL
```

**优化策略**:
- 分批处理（每批 500 条）避免 IN 子句过长
- `ALL_RECORDS` 标记避免全表 ID 物化
- 去重机制防止循环依赖导致的无限传播

---

## 五、派生字段重算流程

### 5.1 计算编排器 (ComputedOrchestratorService)

**文件**: `apps/nestjs-backend/src/features/record/computed/services/computed-orchestrator.service.ts:23-528`

```typescript
async computeCellChangesForRecords(...) {
  // 1. 收集影响：受影响的字段 + 受影响的记录
  const { impact, tableDomains } = await this.collector.collect(tableId, cellContexts, exclude);
  
  // 2. 行级锁防止并发更新冲突
  await this.lockImpactedRecords(filtered, impact, tableDomains);
  
  // 3. 执行基础更新（用户发起的变更）
  await update(tableDomains);
  
  // 4. 评估并发布派生字段变更
  const total = await this.evaluator.evaluate(impact, {
    excludeFieldIds,
    tableDomains,
  });
}
```

### 5.2 计算评估器 (ComputedEvaluatorService)

**文件**: `apps/nestjs-backend/src/features/record/computed/services/computed-evaluator.service.ts:28-200`

#### 5.2.1 拓扑分层

```typescript
private async buildFieldLayers(entries: ...) {
  // 加载字段依赖边
  const edges = await this.loadFieldDependencyEdges(uniqueFieldIds);
  
  // Kahn 算法拓扑排序，确保依赖字段先计算
  const levels = this.topoSortFieldLevels(uniqueFieldIds, edges);
  
  // 结果按层级分组：
  // Layer 0: 不依赖其他派生字段的字段
  // Layer 1: 仅依赖 Layer 0 的字段
  // Layer 2: 依赖 Layer 0/1 的字段
  // ...
}
```

#### 5.2.2 分页执行策略

```typescript
// 策略选择
private readonly paginationStrategies: IRecordPaginationStrategy[] = [
  new RecordIdBatchStrategy(),      // 已知记录ID，每批10000条
  new AutoNumberCursorStrategy(),   // 全表扫描，基于自增游标
];

// 执行每层更新
for (const layer of layers) {
  for (const [tableId, layerFieldIds] of layer) {
    // 构建查询：包含所有 lookup/rollup 子查询
    const { qb, alias } = await this.recordQueryBuilder.createRecordQueryBuilder(...);
    
    // 分页执行 UPDATE
    await strategy.run(paginationContext, async (rows) => {
      const evaluatedRows = this.buildEvaluatedRows(rows, fieldInstances);
      // 发布 ShareDB ops
      totalOps += this.publishBatch(tableId, impactedFieldIds, ..., evaluatedRows);
    });
  }
}
```

### 5.3 SQL 更新执行 (RecordComputedUpdateService)

**文件**: `apps/nestjs-backend/src/features/record/computed/services/record-computed-update.service.ts:18-243`

```typescript
async updateFromSelect(tableId: string, qb: Knex.QueryBuilder, fields: IFieldInstance[], ...) {
  // UPDATE table SET (col1, col2, ..., __version) = 
  //   (SELECT src.col1, src.col2, ..., __version + 1 FROM (subquery) src WHERE src.__id = table.__id)
  // WHERE __id IN (SELECT __id FROM src)
  // RETURNING __id, __version, ...
  
  const sql = this.dbProvider.updateFromSelectSql({
    dbTableName,
    idFieldName: '__id',
    subQuery: qb,  // 包含所有派生字段计算的子查询
    dbFieldNames: columnNames,
    returningDbFieldNames: returningWithAutoNumber,
    restrictRecordIds,
  });
}
```

**原子性保障**:
- 单个 UPDATE 语句完成所有派生字段更新
- `__version` 自增实现乐观锁
- 行级锁 (`SELECT FOR UPDATE`) 防止死锁

---

## 六、缓存失效机制

### 6.1 版本号机制

```sql
-- 每次更新自动递增版本号
__version = __version + 1
```

**文件**: `apps/nestjs-backend/src/features/record/computed/services/record-computed-update.service.ts:164-176`

### 6.2 ShareDB 实时发布

**文件**: `apps/nestjs-backend/src/features/record/computed/services/computed-evaluator.service.ts:130-140`

```typescript
await strategy.run(paginationContext, async (rows) => {
  const evaluatedRows = this.buildEvaluatedRows(rows, fieldInstances);
  totalOps += this.publishBatch(
    tableId,
    impactedFieldIds,
    validFieldIdSet,
    excludeFieldIds,
    evaluatedRows
  );
});
```

**发布内容**:
- OpType: `setRecord`
- 包含新的 cellValue 和 __version
- 通过 WebSocket 推送到前端
- 前端据此更新本地缓存

### 6.3 失效传播链

```
源记录更新 → 版本号+1 → ShareDB Op发布
                         ↓
                 前端缓存匹配版本号
                         ↓
                 失效本地缓存
                         ↓
                 触发重新查询/渲染
```

---

## 七、条件汇总/条件查找的特殊处理

### 7.1 条件汇总的受影响记录判定

**文件**: `apps/nestjs-backend/src/features/record/computed/services/computed-dependency-collector.service.ts:745-1058`

```typescript
private async getConditionalRollupImpactedRecordIds(
  edge: IConditionalRollupAdjacencyEdge,
  foreignRecordIds: string[],
  changeContextMap?: Map<string, ICellContext[]>,
  ctx?: ICollectorExecutionContext
): Promise<string[] | typeof ALL_RECORDS> {
  // 对变更前后的值都应用过滤器，取并集
  // 1. 使用原始值查询哪些主机记录受影响
  // 2. 使用更新后的值查询哪些主机记录受影响
  // 3. 合并结果集
  
  // 如果过滤器引用了主机表字段，则需要 EXISTS 子查询
  const existsSubquery = this.dataKnex
    .select(this.dataKnex.raw('1'))
    .from(foreignFrom())
    .join(VALUES 子查询)
    .where(filter);  // 应用条件汇总的过滤器
  
  const queryBuilder = this.dataKnex
    .select(`"__host"."__id" as id`)
    .from(`${hostTableName} as __host`)
    .whereExists(existsSubquery);
}
```

---

## 八、关键设计决策

### 8.1 双重依赖追踪

| 机制 | 优点 | 缺点 |
|------|-----|-----|
| reference 表 SQL CTE | 高效，支持任意深度递归 | 依赖数据完整性，历史数据可能缺失 |
| lookup_options 查询 | 准确，直接匹配字段配置 | 仅支持一层 lookup，需要递归补充 |

### 8.2 ALL_RECORDS 标记

避免全表记录ID物化，当变更影响范围超过阈值（10000条）时，直接标记为全表重算。

### 8.3 拓扑分层执行

确保依赖字段按正确顺序计算，避免读取到过期值。

### 8.4 基于 SQL 的批量更新

相比逐行更新，`UPDATE FROM SELECT` 性能提升显著，且保证原子性。

---

## 九、代码优化建议

### 潜在问题 1: 重复的 JSON 类型检查

**文件**: `computed-dependency-collector.service.ts:804-814`

```typescript
// 重复检查两次，可合并为一次
if (
  Array.from(foreignFieldMap.values()).some((field) => field.dbFieldType === DbFieldType.Json)
) {
  return ALL_RECORDS;
}

if (
  Array.from(foreignFieldMap.values()).some((field) => field.dbFieldType === DbFieldType.Json)
) {
  return ALL_RECORDS;
}
```

**优化**: 删除重复检查。

### 潜在问题 2: 排序字段过滤的可复用性

`buildSortFieldAccessor` 和 `applySortFieldFilter` 方法可提取为通用工具函数，供条件汇总和条件查找共享。

---

## 十、总结

Teable 的查找与汇总字段的值传播链路是一个**精心设计的分布式计算管道**，其核心优势在于：

1. **完整性**: 通过 reference 表 + lookup_options 双重机制确保依赖不遗漏
2. **性能**: BFS 级联 + 批量 SQL 更新 + 分页策略，兼顾效率与内存
3. **正确性**: 拓扑排序 + 版本号乐观锁 + 行级锁，保证并发安全
4. **实时性**: ShareDB 实时推送，确保前端缓存及时失效

该架构成功解决了多维表格中跨表引用、派生字段重算、缓存一致性等核心难题，为复杂数据模型提供了可靠的计算基础设施。

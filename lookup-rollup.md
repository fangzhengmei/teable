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

### 3.2 计算编排器的双入口模式

**文件**: `apps/nestjs-backend/src/features/record/computed/services/computed-orchestrator.service.ts:44-359`

| 入口方法 | 触发时机 | 调用的收集器方法 |
|---------|---------|----------------|
| `computeCellChangesForRecords` | 记录数据变更 | `collector.collect()` |
| `computeCellChangesForFields` | 字段定义变更 | `collector.collectForFieldChanges()` |

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

## 五、ALL_RECORDS 触发条件深度分析

### 5.1 ALL_RECORDS 的独立定义

**注意**：`ALL_RECORDS` 在两个文件中独立定义，值均为 `Symbol('ALL_RECORDS')`：

| 文件 | 行号 | 用途 |
|------|------|------|
| `link-cascade-resolver.ts` | 32 | 链接级联解析器内部使用 |
| `computed-dependency-collector.service.ts` | 71 | 依赖收集器内部使用 |

### 5.2 条件汇总 ALL_RECORDS 八大触发条件

**文件**: `apps/nestjs-backend/src/features/record/computed/services/computed-dependency-collector.service.ts:745-814`

`getConditionalRollupImpactedRecordIds` 方法中有 **8 个明确的触发点**，按检查顺序排列：

```typescript
private async getConditionalRollupImpactedRecordIds(
  edge: IConditionalRollupAdjacencyEdge,
  foreignRecordIds: string[],
  changeContextMap?: Map<string, ICellContext[]>,
  ctx?: ICollectorExecutionContext
): Promise<string[] | typeof ALL_RECORDS> {
```

| 触发条件 | 代码位置 | 判定逻辑 |
|---------|---------|---------|
| **① 样本量超限** | 755-757 | `uniqueForeignIds.length > MAX_CONDITIONAL_ROLLUP_SAMPLE` (阈值 = 10,000) |
| **② 无过滤器** | 762-765 | `!filter` 条件汇总未配置过滤条件 |
| **③ 无主机字段引用** | 767-770 | `!hostFieldRefs.length` 过滤器不引用主机表任何字段 |
| **④ 无外键字段引用** | 772-774 | `foreignFieldIds.size === 0` 过滤器不引用外表任何字段 |
| **⑤ 跨表引用** | 776-778 | 过滤器引用了非主机表的字段 (`ref.tableId && ref.tableId !== edge.tableId`) |
| **⑥ 主机字段加载失败** | 780-784 | `hostFieldMap.size !== uniqueHostFieldIds.length` |
| **⑦ 外键字段加载失败** | 786-793 | `foreignFieldMap.size !== foreignFieldIds.size` |
| **⑧ JSON 类型字段** | 804-814 | 外表过滤字段的 `dbFieldType === DbFieldType.Json` |

> **注意**：代码中第 ⑧ 项存在**重复检查**（804-808 行和 810-814 行），属于可优化的冗余代码。

### 5.3 条件汇总 ALL_RECORDS 传播逻辑

**文件**: `computed-dependency-collector.service.ts:1322-1370`

当 `getConditionalRollupImpactedRecordIds` 返回 `ALL_RECORDS` 时：

```typescript
if (matched === ALL_RECORDS) {
  const updated = this.markAllSeed(tablesWithAllRecords, edge.tableId);
  if (updated) {
    targetGroup.preferAutoNumberPaging = true;  // 启用游标分页
    dirty = true;                              // 标记需要重新计算
    enqueueConditional(edge.tableId);          // 加入处理队列
    enqueueLinkDependents(edge.tableId);       // 传播到链接依赖
  }
}
```

### 5.4 字段变更场景的 ALL_RECORDS

**文件**: `computed-dependency-collector.service.ts:1247-1251`

在 `collectForFieldChanges`（字段定义变更）场景中：

```typescript
// 字段变更影响该表的 ALL 记录
const tablesWithAllRecords = new Set<string>(originTableIds);
```

此时**源表的所有记录**都被标记为需要重算，因为字段定义变更可能影响每一行。

---

## 六、对称链接预播种机制

### 6.1 对称链接字段解析

**文件**: `computed-dependency-collector.service.ts:582-612`

```typescript
private async resolveRelatedLinkFieldIds(
  fieldIds: string[],
  fieldToTableMap?: Map<string, string>,
  ctx?: ICollectorExecutionContext
): Promise<string[]> {
  // 遍历变更字段，找出其中的链接字段
  for (const [tableId, ids] of groupedByTable) {
    const tableDomain = await this.getTableDomain(tableId, ctx);
    for (const id of ids) {
      const field = tableDomain.getField(id);
      if (!field || field.type !== FieldType.Link || field.isLookup) continue;
      
      result.add(field.id);  // 加入链接字段自身
      
      // 关键：解析对称链接字段ID
      const opts = this.parseOptionsLoose<{ symmetricFieldId?: string }>(field.options);
      if (opts?.symmetricFieldId) result.add(opts.symmetricFieldId);
    }
  }
  return Array.from(result);
}
```

### 6.2 对称链接记录预播种

**文件**: `computed-dependency-collector.service.ts:1561-1629`

```typescript
// 找出变更中的链接字段
const linkFields = currentTableDomain.fieldList.filter(
  (field) => changedFieldIdSet.has(field.id) && field.type === FieldType.Link && !field.isLookup
);

// 预播种容器：按外表分组的记录ID集合
const plannedForeignRecordIds: Record<string, Set<string>> = {};

for (const lf of linkFields) {
  const optsLoose = this.parseOptionsLoose<ILinkOptionsWithSymmetric>(lf.options);
  const foreignTableId = optsLoose?.foreignTableId;
  const symmetricFieldId = optsLoose?.symmetricFieldId;

  if (foreignTableId && symmetricFieldId) {
    // ① 将对称字段加入受影响字段集合
    (impact[foreignTableId] ||= {
      fieldIds: new Set<string>(),
      recordIds: new Set<string>(),
    }).fieldIds.add(symmetricFieldId);

    // ② 从 oldValue 和 newValue 双端提取记录ID，覆盖添加和删除场景
    const targetIds = new Set<string>();
    for (const ctx of ctxs) {
      if (ctx.fieldId !== lf.id) continue;
      const toIds = (v: unknown) => {
        if (!v) return [] as string[];
        const arr = Array.isArray(v) ? v : [v];
        return arr
          .map((x) => (x && typeof x === 'object' ? (x as { id?: string }).id : undefined))
          .filter((id): id is string => !!id);
      };
      toIds(ctx.oldValue).forEach((id) => targetIds.add(id));  // 移除的链接
      toIds(ctx.newValue).forEach((id) => targetIds.add(id));  // 新增的链接
    }
    
    // ③ 存入预播种容器
    if (targetIds.size) {
      const set = (plannedForeignRecordIds[foreignTableId] ||= new Set<string>());
      targetIds.forEach((id) => set.add(id));
    }
  }
}
```

### 6.3 预播种注入链接级联

**文件**: `computed-dependency-collector.service.ts:1624-1629`

```typescript
const explicitSeeds = new Map<string, Set<string>>();
explicitSeeds.set(tableId, new Set(changedRecordIds));  // 源表变更记录

// 注入对称链接预播种的记录ID
for (const [tid, ids] of Object.entries(plannedForeignRecordIds)) {
  if (!ids.size) continue;
  explicitSeeds.set(tid, new Set(ids));  // 外表预播种记录
}
```

**设计意图**：
- 对称链接的两端记录都需要更新，通过预播种确保 BFS 一开始就包含两端
- 从 `oldValue` 和 `newValue` 双端提取，确保链接**添加**和**删除**都能触发重算

---

## 七、条件边迭代收敛机制

### 7.1 迭代收敛完整流程

**文件**: `computed-dependency-collector.service.ts:1257-1395`（字段变更场景）和 `1637-1778`（记录变更场景）

两个场景的迭代算法**完全一致**，以下以记录变更场景为例：

```
┌─────────────────────────────────────────────────────────────────────┐
│  初始状态                                                           │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  1. 初始 computeLinkClosure 计算链接边传播                    │  │
│  │  2. findRecordSetGrowth({}, recordSets) 检测初始增长           │  │
│  │  3. 将有增长的表加入队列 queue                                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                            ↓                                        │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  while (queue.length)                                        │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │  src = queue.shift()                                   │  │  │
│  │  │  处理该表的所有条件边 referenceEdges                   │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  │                            ↓                                  │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │  对每条条件边：                                         │  │  │
│  │  │  • ALL_RECORDS: markAllSeed + dirty = true              │  │  │
│  │  │  • 记录ID集合: addExplicitSeed + dirty = true           │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  │                            ↓                                  │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │  if (dirty)                                             │  │  │
│  │  │  • 重新 computeLinkClosure 计算链接传播                 │  │  │
│  │  │  • findRecordSetGrowth 检测新增记录                     │  │  │
│  │  │  • 将新增长的表重新入队                                 │  │  │
│  │  │  • recordSets = nextRecordSets                          │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                            ↓                                        │
│  收敛：queue 为空，没有新的记录需要处理                             │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 记录集增长检测 (findRecordSetGrowth)

**文件**: `computed-dependency-collector.service.ts:332-375`

```typescript
private findRecordSetGrowth(
  previous: Record<string, Set<string> | typeof ALL_RECORDS | undefined>,
  next: Record<string, Set<string> | typeof ALL_RECORDS>
): string[] {
  for (const tableId of tableIds) {
    const prevSet = previous[tableId];
    const nextSet = next[tableId];
    
    // 新增表 → 有增长
    if (!prevSet) { changed.push(tableId); continue; }
    
    // 从部分记录升级为全表 → 有增长
    if (prevSet !== ALL_RECORDS && nextSet === ALL_RECORDS) {
      changed.push(tableId); continue;
    }
    
    // 从全表降级为部分记录 → 不应该发生，忽略
    if (prevSet === ALL_RECORDS && nextSet !== ALL_RECORDS) {
      continue;
    }
    
    // 记录集合有新增 → 有增长
    if (prevSet instanceof Set && nextSet instanceof Set) {
      if (nextSet.size > prevSet.size) { changed.push(tableId); continue; }
      for (const id of nextSet) {
        if (!prevSet.has(id)) { hasNew = true; break; }
      }
      if (hasNew) changed.push(tableId);
    }
  }
  return changed;
}
```

### 7.3 去重入队机制

**文件**: `computed-dependency-collector.service.ts:1267-1278`

```typescript
const queued = new Set<string>();  // 防重集合

const enqueueConditional = (tableId: string) => {
  if (!tableId || queued.has(tableId)) return;  // 已在队列中，跳过
  queued.add(tableId);
  queue.push(tableId);
};

const enqueueLinkDependents = (tableId: string) => {
  const targets = linkAdj[tableId];
  if (!targets) return;
  targets.forEach((tid) => enqueueConditional(tid));  // 传播到链接依赖表
};
```

### 7.4 收敛判定

迭代收敛发生在：
1. `queue.length === 0` —— 没有新的表需要处理
2. 每次迭代中 `dirty === false` —— 没有新的记录ID被发现

**收敛保证**：
- 每次迭代只可能增加记录ID（单调增长）
- 记录ID总数有限（不会超过表的总行数）
- 因此迭代必定在有限步骤内收敛

---

## 八、派生字段重算流程

### 8.1 计算编排器 (ComputedOrchestratorService)

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

### 8.2 计算评估器 (ComputedEvaluatorService)

**文件**: `apps/nestjs-backend/src/features/record/computed/services/computed-evaluator.service.ts:28-200`

#### 8.2.1 拓扑分层

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

#### 8.2.2 分页执行策略

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

### 8.3 SQL 更新执行 (RecordComputedUpdateService)

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

## 九、缓存失效机制

### 9.1 版本号机制

```sql
-- 每次更新自动递增版本号
__version = __version + 1
```

**文件**: `apps/nestjs-backend/src/features/record/computed/services/record-computed-update.service.ts:164-176`

### 9.2 ShareDB 实时发布

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

### 9.3 失效传播链

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

## 十、关键设计决策

### 10.1 双重依赖追踪

| 机制 | 优点 | 缺点 |
|------|-----|-----|
| reference 表 SQL CTE | 高效，支持任意深度递归 | 依赖数据完整性，历史数据可能缺失 |
| lookup_options 查询 | 准确，直接匹配字段配置 | 仅支持一层 lookup，需要递归补充 |

### 10.2 ALL_RECORDS 标记

避免全表记录ID物化，当条件汇总样本量超过阈值（**10,000 条**，由 `MAX_CONDITIONAL_ROLLUP_SAMPLE` 定义）或其他 7 种条件触发时，直接标记为全表重算。

### 10.3 对称链接双端预播种

从 `oldValue` 和 `newValue` 两端提取记录ID，确保链接的**添加**和**删除**都能正确触发对称链接端的重算。

### 10.4 条件边迭代收敛

通过 `dirty` 标志 + `findRecordSetGrowth` 检测 + 重新 `computeLinkClosure` 的循环，确保条件汇总的跨表影响被完整传播，直到没有新记录发现为止。

### 10.5 拓扑分层执行

确保依赖字段按正确顺序计算，避免读取到过期值。

### 10.6 基于 SQL 的批量更新

相比逐行更新，`UPDATE FROM SELECT` 性能提升显著，且保证原子性。

---

## 十一、代码优化建议

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

### 潜在问题 3: ALL_RECORDS 符号重复定义

`ALL_RECORDS` 在 `link-cascade-resolver.ts:32` 和 `computed-dependency-collector.service.ts:71` 分别定义，可考虑提取为共享常量。

---

## 十二、总结

Teable 的查找与汇总字段的值传播链路是一个**精心设计的分布式计算管道**，其核心优势在于：

1. **完整性**: 通过 reference 表 + lookup_options 双重机制确保依赖不遗漏
2. **性能**: BFS 级联 + 批量 SQL 更新 + 分页策略，兼顾效率与内存
3. **正确性**: 拓扑排序 + 版本号乐观锁 + 行级锁，保证并发安全
4. **实时性**: ShareDB 实时推送，确保前端缓存及时失效
5. **健壮性**: 8 种 ALL_RECORDS 兜底条件 + 对称链接预播种 + 条件边迭代收敛，确保极端场景下的正确性

**三大核心机制的协同作用**：
- **ALL_RECORDS 触发**：在样本量过大、过滤器复杂或字段类型特殊时，优雅降级为全表重算，避免内存溢出和 SQL 错误
- **对称链接预播种**：通过 oldValue/newValue 双端提取，确保链接关系变更的两端都能被正确处理
- **条件边迭代收敛**：通过单调增长的记录集和队列去重机制，确保条件汇总的跨表影响被完整传播

该架构成功解决了多维表格中跨表引用、派生字段重算、缓存一致性等核心难题，为复杂数据模型提供了可靠的计算基础设施。

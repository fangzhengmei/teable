# 公式字段依赖图构建、脏值检测与重算机制分析

## 1. 整体架构概述

Teable v2 的计算字段（公式/查找/汇总/链接/条件查找/条件汇总）更新采用混合模式（同步 + 异步 Outbox）。整个流程分为三个核心阶段：

1. **依赖图构建** - `FieldDependencyGraph` 从数据库加载字段元数据和引用关系，构建有向无环图（DAG）
2. **更新规划** - `ComputedUpdatePlanner` 基于依赖图和变更上下文，生成有序的更新计划和脏值传播路径
3. **执行重算** - `ComputedFieldUpdater` 准备脏值状态、传播脏记录、按计划执行 SQL 更新

## 2. 依赖图构建（FieldDependencyGraph）

### 2.1 核心数据结构

**FieldDependencyGraphData** ([FieldDependencyGraph.ts:59-62](packages/v2/adapter-table-repository-postgres/src/record/computed/FieldDependencyGraph.ts#L59-L62)):
```typescript
type FieldDependencyGraphData = {
  fieldsById: Map<string, FieldMeta>;      // 字段元数据
  edges: ReadonlyArray<FieldDependencyEdge>;  // 依赖边
};
```

**FieldDependencyEdge** ([types.ts:67-77](packages/v2/field-dependency-core/src/types.ts#L67-L77)):
```typescript
interface FieldDependencyEdge {
  fromFieldId: string;      // 依赖源字段
  toFieldId: string;        // 依赖目标字段
  fromTableId: string;      // 源表ID
  toTableId: string;        // 目标表ID
  kind: 'same_record' | 'cross_record';  // 依赖类型
  linkFieldId?: string;     // 跨记录依赖时的链接字段
  semantic?: FieldDependencyEdgeSemantic; // 语义提示
}
```

### 2.2 两种加载模式

#### 全量加载模式（loadFull）
`FieldDependencyGraph.loadFull()` ([FieldDependencyGraph.ts:124-318](packages/v2/adapter-table-repository-postgres/src/record/computed/FieldDependencyGraph.ts#L124-L318))

加载整个 Base 的所有计算字段，用于 Schema 验证等场景：
1. 加载所有计算字段元数据（公式/查找/汇总/链接/条件查找/条件汇总）
2. 加载引用表（`reference`）中的公式引用边
3. 为每个计算字段派生出依赖边
4. 合并引用边和派生边

#### 增量加载模式（loadIncremental）
`FieldDependencyGraph.loadIncremental()` ([FieldDependencyGraph.ts:620-820](packages/v2/adapter-table-repository-postgres/src/record/computed/FieldDependencyGraph.ts#L620-L820))

只加载与种子字段相关的依赖，用于记录变更场景：
1. **`findAffectedFieldIds()`** - 使用迭代 BFS 找出所有受影响的字段 ID
   - 使用 `MAX_ITERATIONS = 1000` 和 `MAX_VISITED = 50000` 防止 runaway BFS
   - 批量处理（每批 100 条）避免过大的 IN 子句
   - 使用 UNION 组合多个来源的依赖查询（引用表、查找链接字段、查找源字段等）

2. 加载受影响字段的元数据
3. 构建依赖边

### 2.3 边的类型与语义

**边的类型（kind）**:
- `same_record` - 同一条记录内的依赖，无需链接遍历
- `cross_record` - 跨记录依赖，需要通过链接字段或条件匹配

**语义类型（semantic）** ([types.ts:52-61](packages/v2/field-dependency-core/src/types.ts#L52-L61)):

| 语义类型 | 说明 | 边类型 |
|---------|------|--------|
| `formula_ref` | 公式字段直接引用其他字段 | same_record / cross_record |
| `lookup_link` | 查找/汇总依赖其链接字段 | same_record |
| `lookup_source` | 查找字段依赖源表字段 | cross_record |
| `lookup_filter` | 查找过滤条件引用的字段 | same_record / cross_record |
| `rollup_source` | 汇总字段依赖源表字段 | cross_record |
| `rollup_filter` | 汇总过滤条件引用的字段 | same_record / cross_record |
| `link_title` | 链接字段标题依赖源表主键字段 | cross_record |
| `conditional_rollup_source` | 条件汇总依赖 | cross_record |
| `conditional_lookup_source` | 条件查找依赖 | cross_record |

### 2.4 派生边的构建逻辑

**Lookup/Rollup 字段** ([FieldDependencyGraph.ts:161-225](packages/v2/adapter-table-repository-postgres/src/record/computed/FieldDependencyGraph.ts#L161-L225)):
```
lookupFieldId → lookupField (cross_record, via linkFieldId)
linkFieldId → lookupField (same_record)
filterFieldId → lookupField (same_record / cross_record)
```

**Link 字段** ([FieldDependencyGraph.ts:228-250](packages/v2/adapter-table-repository-postgres/src/record/computed/FieldDependencyGraph.ts#L228-L250)):
```
foreignTable.lookupFieldId → linkField (cross_record, link_title)
```

**条件查找/条件汇总** ([FieldDependencyGraph.ts:253-311](packages/v2/adapter-table-repository-postgres/src/record/computed/FieldDependencyGraph.ts#L253-L311)):
```
foreignTable.lookupFieldId → conditionalField (cross_record)
conditionFieldId → conditionalField (same_record / cross_record)
```

### 2.5 边的合并策略

`mergeEdges()` ([FieldDependencyGraph.ts:1260-1278](packages/v2/adapter-table-repository-postgres/src/record/computed/FieldDependencyGraph.ts#L1260-L1278)):
- 派生边优先于引用边（更精确的语义）
- 使用 `fromFieldId|toFieldId|kind|linkFieldId` 作为去重键
- 避免重复边导致的入度计算错误

## 3. 脏值检测与失效机制

### 3.1 字段级失效 vs 记录级失效

#### 字段级失效（Field-level Invalidation）

在 `ComputedUpdatePlanner.planStage()` 中确定哪些字段需要重算：

1. **值变更传播** ([ComputedUpdatePlanner.ts:418-427](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L418-L427)):
   - 过滤 `isEdgeRelevantForValue()` - 除 `lookup_link` 外的所有边
   - 用于普通字段值变更导致的依赖字段失效

2. **链接关系变更传播** ([ComputedUpdatePlanner.ts:428-483](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L428-L483)):
   - 过滤 `isEdgeRelevantForLink()` - 仅 `same_record` 且 `semantic === 'lookup_link'`
   - 处理对称链接字段的级联更新
   - 链接关系变更后，级联到值依赖字段

3. **特殊字段处理**:
   - INSERT 时包含无依赖的公式字段（`context-free formulas`）
   - INSERT 时包含所有条件字段（`conditionalRollup`/`conditionalLookup`）
   - 公式表达式直接引用种子字段的强制包含

#### 记录级失效（Record-level Invalidation）

在 `ComputedFieldUpdater.prepareDirtyState()` 中确定哪些记录需要重算：

1. **脏记录表** ([ComputedFieldUpdater.ts:68-70](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldUpdater.ts#L68-L70)):
   ```sql
   CREATE TEMPORARY TABLE tmp_computed_dirty (
     table_id text NOT NULL,
     record_id text NOT NULL,
     PRIMARY KEY (table_id, record_id)
   ) ON COMMIT DROP
   ```

2. **三种脏值传播模式** ([ComputedUpdatePlanner.ts:90](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L90)):

| 传播模式 | 适用场景 | 性能影响 |
|---------|---------|---------|
| `linkTraversal` | 常规查找/汇总/链接，通过链接关系遍历 | 最优，仅传播关联记录 |
| `conditionalFiltered` | 条件字段且过滤条件未变更 | 较好，精确匹配过滤条件 |
| `allTargetRecords` | 过滤条件变更、DELETE、运行时降级 | 保守，标记目标表所有记录 |

### 3.2 脏值传播实现

`propagateDirtyRecords()` ([ComputedFieldUpdater.ts:1857-2016](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldUpdater.ts#L1857-L2016)):

```
迭代传播（多轮）:
  ├─ 构建每条边的 SELECT 查询
  ├─ 合并相同的传播路径（UNION ALL）
  └─ 执行 INSERT INTO tmp_computed_dirty ... SELECT ...
     └─ 如果本轮没有新插入记录，终止传播
```

**链接遍历模式** (`linkTraversal`) - `buildDirtySelectQuery()` ([ComputedFieldUpdater.ts:2028-2133](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldUpdater.ts#L2028-L2133)):
- `manyOne` / `oneOne`: 通过外键直接 JOIN
- `oneMany`: 通过自键或关联表遍历
- `manyMany`: 通过关联表（junction table）遍历

**条件过滤模式** (`conditionalFiltered`) - `buildPropagationSelect()` ([ComputedFieldUpdater.ts:2195-2334](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldUpdater.ts#L2195-L2334)):
- 将 `filterDto` 转换为 SQL WHERE 条件
- 支持 `includeBeforeImage` 处理记录离开过滤集合的场景
- 使用 `jsonb_populate_record` 重构变更前的记录状态

**全表标记模式** (`allTargetRecords`) - `buildGatedAllTargetSelect()` ([ComputedFieldUpdater.ts:2135-2155](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldUpdater.ts#L2135-L2155)):
```sql
SELECT target_table_id, t.__id
FROM target_table t
INNER JOIN (
  SELECT table_id FROM tmp_computed_dirty 
  WHERE table_id = source_table_id LIMIT 1
) dg ON TRUE
```

### 3.3 全表标记的触发条件

`allTargetRecordsReason` ([ComputedUpdatePlanner.ts:92-105](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L92-L105)):

| 原因 | 说明 |
|-----|------|
| `conditional_filter_field_changed` | 条件字段的过滤条件字段变更 |
| `conditional_missing_filter` | 条件字段缺少过滤配置 |
| `conditional_delete` | DELETE 操作，源记录已不存在 |
| `conditional_filter_fields_not_in_source` | 过滤字段不在源表 |
| `filtered_lookup_delete_requires_source_record` | 带过滤的查找在 DELETE 时无法精确遍历 |
| `symmetric_no_seed_records` | 对称链接无种子记录 |
| `conditional_runtime_invalid_filter` | 运行时过滤条件无效 |
| `conditional_runtime_empty_filter` | 运行时过滤条件为空 |
| `conditional_runtime_invalid_condition_spec` | 运行时条件规范无效 |
| `conditional_runtime_missing_condition_spec` | 运行时缺少条件规范 |

## 4. 重算触发与执行流程

### 4.1 拓扑排序与更新计划

`topoSort()` ([ComputedUpdatePlanner.ts:1231-1274](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L1231-L1274)):

使用 Kahn 算法进行拓扑排序：
1. 计算每个字段的入度
2. 从入度为 0 的节点开始处理
3. 处理完成后减少相邻节点的入度
4. 记录每个字段的层级（level）

生成 `UpdateStep` 按层级分组：
```typescript
type UpdateStep = {
  tableId: TableId;
  fieldIds: ReadonlyArray<FieldId>;
  level: number;  // 依赖层级，0 为最上游
};
```

### 4.2 同表批量优化

`buildSameTableBatches()` ([ComputedUpdatePlanner.ts:1745-1817](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L1745-L1817)):

将同表、仅含 `same_record` 依赖的连续步骤合并为批量：
- 条件：步骤间只有 `same_record` 依赖，无 `cross_record` 依赖
- 优化：使用 CTE 链一次性计算所有公式字段
- 避免：公式展开导致的 volatile 函数重复求值

### 4.3 执行流程

`ComputedFieldUpdater.execute()` ([ComputedFieldUpdater.ts:308-501](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldUpdater.ts#L308-L501)):

```
execute(plan, context):
  ├─ 1. prepareDirtyState()
  │   ├─ 创建临时脏记录表
  │   ├─ 种子脏记录（变更的记录）
  │   ├─ 传播脏记录（按传播边迭代）
  │   └─ 收集脏记录统计
  │
  └─ 2. executePreparedSteps()
      └─ 按 level 顺序执行每个 UpdateStep
          ├─ 构建 SELECT 查询（计算新值）
          ├─ 构建 UPDATE ... FROM 查询
          └─ 执行更新（可选收集变更数据）
```

### 4.4 增量传播与链式更新

`collectDirtySeedGroups()` ([ComputedFieldUpdater.ts:1503-1589](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldUpdater.ts#L1503-L1589)):

- 阈值 `SEED_ALL_THRESHOLD = 5000`：超过该阈值的表采用全表种子
- 低于阈值的表传递具体的记录 ID
- 用于多阶段更新（同步 + 异步）之间的脏记录传递

## 5. 环依赖防护机制

### 5.1 多层防护架构

#### 第一层：Schema 层检测

`detectCircularDependency()` ([detectCircularDependency.ts:22-115](packages/v2/adapter-table-repository-postgres/src/schema/helpers/detectCircularDependency.ts#L22-L115)):

在字段创建/更新时检测环依赖：
```typescript
function detectCircularDependency(edges: FieldDependencyEdge[]): Result<void, DomainError> {
  // 1. 构建邻接表和入度表
  // 2. Kahn 算法拓扑排序
  // 3. 如果排序结果长度 != 节点总数，存在环
  // 4. DFS 查找具体的环路径
  // 5. 返回包含环路径的错误信息
}
```

#### 第二层：规划层检测

`ComputedUpdatePlanner.planStage()` ([ComputedUpdatePlanner.ts:634-762](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L634-L762)):

```
topoSort() 后检测:
  if (ordered.length !== computedFieldIds.size):
    ├─ 找出未排序的字段
    ├─ findCycle() - DFS 查找环路径
    ├─ findCycleParticipantFieldIds() - Tarjan 强连通分量算法
    └─ 根据 cyclePolicy 决定:
        ├─ 'error' - 抛出冲突错误
        └─ 'skip' - 跳过环中的字段，继续执行其他字段
```

**强连通分量算法** (`findCycleParticipantFieldIds()`) ([ComputedUpdatePlanner.ts:1276-1350](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L1276-L1350)):
- 使用 Tarjan 算法找出所有强连通分量
- 大小 > 1 的分量即为环参与者
- 也检测自环（self-loop）

#### 第三层：BFS 迭代防护

`FieldDependencyGraph.findAffectedFieldIds()` ([FieldDependencyGraph.ts:827-1030](packages/v2/adapter-table-repository-postgres/src/record/computed/FieldDependencyGraph.ts#L827-L1030)):

```typescript
const MAX_ITERATIONS = 1000;  // 防止无限迭代
const MAX_VISITED = 50000;     // 防止内存溢出
```

### 5.2 环处理策略

**`cyclePolicy: 'error'`** (默认):
- 检测到环立即抛出 `domainError.conflict()`
- 包含详细的环路径信息，例如：
  ```
  fieldA(formula) → fieldB(lookup) → fieldA(formula)
  ```

**`cyclePolicy: 'skip'`**:
- 识别环中的所有参与字段
- 将这些字段从更新计划中移除
- 记录 `cycleInfo` 包含：
  - 未排序字段列表
  - 检测到的环路径
  - 示例字段信息
  - 警告消息

### 5.3 防止传播死循环

`propagateDirtyRecords()` 中的终止条件 ([ComputedFieldUpdater.ts:2001-2003](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldUpdater.ts#L2001-L2003)):
```typescript
if (insertedRowCount === 0) {
  break;  // 无新脏记录，终止传播
}
```

此外，`collectDirectAffectedFieldIds()` 中的 `includeSeedsAlways` 参数 ([ComputedUpdatePlanner.ts:1023-1084](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L1023-L1084)):
- INSERT: `true` - 包含所有种子字段
- UPDATE/DELETE: `false` - 只包含被其他种子依赖的种子字段，防止无限循环

## 6. 关键类与文件索引

| 类/文件 | 职责 |
|--------|------|
| `FieldDependencyGraph.ts` | 依赖图加载与构建 |
| `ComputedUpdatePlanner.ts` | 更新规划、拓扑排序、环检测、传播边构建 |
| `ComputedFieldUpdater.ts` | 脏值准备、传播、SQL 执行 |
| `detectCircularDependency.ts` | Schema 层环依赖检测 |
| `field-dependency-core/types.ts` | 核心类型定义 |
| `ComputedUpdateWorker.ts` | 异步更新 Worker |
| `HybridWithOutboxStrategy.ts` | 混合模式（同步+异步）策略 |

## 7. 代码优化建议

### 7.1 环检测性能优化

当前 `findCycleParticipantFieldIds()` 使用 Tarjan 算法，而 `findCycle()` 使用独立的 DFS。可以统一使用一次 Tarjan 算法同时完成环检测和路径重建。

### 7.2 增量加载的缓存优化

`findAffectedFieldIds()` 每次都执行复杂的 UNION 查询，可以考虑：
- 为常用查询模式添加物化视图
- 应用层缓存字段依赖关系（注意失效策略）

### 7.3 脏值传播的批处理优化

当前 `propagateDirtyRecords()` 按 `maxPasses` 迭代，可以考虑：
- 基于边的层级（level）一次性传播，避免多轮迭代
- 对 `allTargetRecords` 模式的边优先处理，减少后续迭代

---

## 8. 重算触发链路：完整调用路径

### 8.1 记录写入入口层

#### Command Handler 入口

**CreateRecordHandler** ([CreateRecordHandler.ts:37](packages/v2/core/src/commands/CreateRecordHandler.ts#L37)):
```
用户创建记录请求 → CreateRecordCommand → CreateRecordHandler.handle()
  ↓
PostgresTableRecordRepository.insert()
```

**UpdateRecordHandler** ([UpdateRecordHandler.ts:94](packages/v2/core/src/commands/UpdateRecordHandler.ts#L94)):
```
用户更新记录请求 → UpdateRecordCommand → UpdateRecordHandler.handle()
  ↓
PostgresTableRecordRepository.update() / updateMany()
```

**DeleteRecordsHandler** ([DeleteRecordsHandler.ts:45](packages/v2/core/src/commands/DeleteRecordsHandler.ts#L45)):
```
用户删除记录请求 → DeleteRecordsCommand → DeleteRecordsHandler.handle()
  ↓
PostgresTableRecordRepository.deleteMany()
```

#### Repository 写入方法层级

`PostgresTableRecordRepository` ([PostgresTableRecordRepository.ts:870](packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts#L870)) 提供多个写入方法，每个方法最后调用 `runComputedUpdate*` 系列函数：

| 写入方法 | 对应计算更新方法 | changeType |
|---------|-----------------|------------|
| `insert()` | `runComputedUpdate()` | `'insert'` |
| `insertMany()` | `runComputedUpdateMany()` | `'insert'` |
| `update()` | `runComputedUpdateById()` | `'update'` |
| `updateMany()` | `runComputedUpdateManyByIds()` | `'update'` |
| `deleteMany()` | `runComputedDeleteUpdateMany()` | `'delete'` |

### 8.2 种子字段展开与执行策略选择

#### 种子字段展开

`expandComputedSeedFieldIds()` ([PostgresTableRecordRepository.ts:3399-3438](packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts#L3399-L3438)):

在调用 planner 之前，先展开种子字段集合：
1. 收集用户显式变更的字段 ID
2. 遍历表中所有计算字段，检查它们的依赖是否包含变更字段
3. 对公式字段额外检查表达式引用的字段 ID
4. 将这些依赖于变更字段的计算字段也加入种子集合

**目的**：确保即使计算字段本身未被显式修改，只要其依赖变更了，也能被正确识别为需要重算。

#### 执行策略选择

`runComputedUpdate()` 中的策略判断 ([PostgresTableRecordRepository.ts:2971-2973](packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts#L2971-L2973)):
```typescript
const shouldExecuteInline =
  this.computedUpdateStrategy.mode === 'sync' ||
  (this.computedUpdateStrategy.mode === 'hybrid' && changeType === 'insert');
```

| 策略 | mode | INSERT | UPDATE/DELETE |
|-----|------|--------|---------------|
| SyncInTransactionStrategy | `'sync'` | 同步执行 | 同步执行 |
| HybridWithOutboxStrategy | `'hybrid'` | 同步执行 | 异步 Outbox |
| AsyncWithRetryStrategy | `'async'` | 异步 Outbox | 异步 Outbox |

默认策略：`HybridWithOutboxStrategy` ([register.ts:186](packages/v2/adapter-table-repository-postgres/src/record/di/register.ts#L186))

### 8.3 同步路径（Sync Path）

#### SyncInTransactionStrategy.execute() ([SyncInTransactionStrategy.ts:34-131](packages/v2/adapter-table-repository-postgres/src/record/computed/strategies/SyncInTransactionStrategy.ts#L34-L131))

```
同步执行循环:
  while (currentPlan.steps.length > 0):
    ├─ acquireLocks() - 获取表级 advisory lock
    ├─ updater.execute() - 执行完整的 dirtyState + executePreparedSteps
    ├─ collectDirtySeedGroups() - 收集本轮更新产生的新脏记录
    ├─ planNextStage() - 用新的种子字段和记录规划下一轮
    └─ 如果没有更多步骤或无新字段，终止循环
```

**关键特性**：
- 多阶段循环直到没有更多需要更新的字段
- 使用 `updatedFieldIds` 集合防止重复更新同一字段
- 每次阶段的变更字段作为下一轮的种子

### 8.4 混合路径（Hybrid Path）

#### HybridWithOutboxStrategy.execute() ([HybridWithOutboxStrategy.ts:162-421](packages/v2/adapter-table-repository-postgres/src/record/computed/strategies/HybridWithOutboxStrategy.ts#L162-L421))

```
混合策略执行:
  while (currentPlan.steps.length > 0):
    ├─ updater.prepareDirtyState() - 只准备脏值，不执行更新
    ├─ splitStepsByPolicy() - 按策略分割 syncSteps 和 asyncSteps
    │
    ├─ [Sync Phase]
    │   ├─ acquireLocks()
    │   ├─ updater.executePreparedSteps(syncSteps) - 只执行同步步骤
    │   ├─ publish RecordsBatchUpdated 事件
    │   └─ collectDirtySeedGroups()
    │
    └─ [Async Phase - 如果有 asyncSteps]
        ├─ buildOutboxTaskInput() - 构建异步任务
        ├─ outbox.enqueueOrMerge() - 写入 Outbox 表
        ├─ scheduleDispatch() - （可选）触发即时调度
        └─ 返回同步部分结果，异步部分由 Worker 处理
```

#### 同步/异步分割策略 `splitStepsByPolicy()` ([HybridWithOutboxStrategy.ts:544-613](packages/v2/adapter-table-repository-postgres/src/record/computed/strategies/HybridWithOutboxStrategy.ts#L544-L613))

三种分割策略由 `syncPolicy` 配置：

| syncPolicy | 说明 | 适用场景 |
|-----------|------|---------|
| `'none'` | 所有步骤都异步 | 非生产环境，高吞吐 |
| `'seedTableOnly'` | 仅种子表的步骤同步 | 生产默认，延迟敏感 |
| `'threshold'` | 根据脏记录数和层级阈值决定 | 平衡性能与延迟 |

**threshold 策略的判断逻辑**：
```
同步层级上限 = max(种子表最大层级, syncMaxLevelHardCap)
从层级 0 开始往上累计:
  累计脏记录数 += 该层级所有表的脏记录数
  如果 单表最大脏记录 > syncMaxDirtyPerTable → 停止
  如果 累计脏记录 > syncMaxTotalDirty → 停止
  否则 syncMaxLevel = 当前层级
```

默认配置 ([HybridWithOutboxStrategy.ts:94-104](packages/v2/adapter-table-repository-postgres/src/record/computed/strategies/HybridWithOutboxStrategy.ts#L94-L104)):
```typescript
{
  syncPolicy: 'seedTableOnly',
  syncMaxDirtyPerTable: 2000,
  syncMaxTotalDirty: 5000,
  syncMaxLevelHardCap: 1,
}
```

### 8.5 异步路径（Async Path / Outbox Worker）

#### Outbox 任务类型

| 任务类型 | 触发时机 | 包含内容 |
|---------|---------|---------|
| **Seed Task** | hybrid 模式下 UPDATE/DELETE | 最小触发信息：种子表、种子记录、变更字段、changeType |
| **Computed Task** | 同步阶段分割出的异步部分 | 完整计划：steps、edges、脏记录统计、已完成进度 |
| **Backfill Task** | 字段创建/修改后的回填 | 表 ID、字段 ID 列表 |

#### Seed Task 入队流程

`buildSeedTaskInput()` + `outbox.enqueueSeedTask()` ([PostgresTableRecordRepository.ts:3039-3072](packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts#L3039-L3072)):
- 只存储最小触发信息，不包含完整计划
- 计划计算延迟到 Worker 执行时进行
- 支持任务合并（相同 `planHash` 的任务合并）

#### ComputedUpdateWorker 执行流程

`ComputedUpdateWorker.runOnce()` ([ComputedUpdateWorker.ts:403-484](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L403-L484)):

```
Worker 单次执行:
  ├─ outbox.claimBatch() - 声明所有权，加锁
  │   └─ 每条任务设置 lockedBy、lockedAt、expiresAt
  ├─ ClaimedTaskLeaseManager.start() - 启动心跳定时器
  │   └─ 定时调用 renewLease() 续期
  └─ 遍历每条任务:
      ├─ leaseManager.ensureTaskActive() - 检查租期
      ├─ processClaimedTask() - 处理任务
      │   ├─ processSeedTask() - Seed 任务 → 计划 + 执行
      │   ├─ processComputedTask() - Computed 任务 → 直接执行
      │   └─ processFieldBackfillTask() - Backfill 任务
      └─ leaseManager.releaseTask() - 释放租期
```

**三种任务处理流程**：

1. **Seed Task 处理** ([ComputedUpdateWorker.ts:1066-1308](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L1066-L1308)):
   ```
   deserializeSeedPayload() → 加载 Table → planner.plan() → 生成完整计划
     → execute() 执行 → collectDirtySeedGroups() → planNextStage()
     → 如果有后续步骤，enqueueOrMerge() 新 Computed Task
     → markDone()
   ```

2. **Computed Task 处理** ([ComputedUpdateWorker.ts:576-764](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L576-L764)):
   ```
   deserializeComputedUpdatePlan() → acquireLocks(wait=false)
     → execute() 执行 → publish events → collectDirtySeedGroups()
     → planNextStage() → 如果有后续步骤且 stageDepth < 50，enqueue 下一阶段
     → markDone()
   ```

3. **Backfill Task 处理** ([ComputedUpdateWorker.ts:924-1059](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L924-L1059)):
   ```
   解析 fieldIds → 加载 Table → backfillService.executeSyncMany()
     → markDone()
   ```

#### 租期管理与故障转移

`ClaimedTaskLeaseManager` ([ComputedUpdateWorker.ts:252-368](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L252-L368)):
- 心跳间隔：`heartbeatIntervalMs`（配置项）
- 任务锁定后，Worker 必须在租期到期前续期
- 如果 Worker 崩溃，任务租期到期后自动释放
- 其他 Worker 可以认领超时任务

#### 大任务拆分

`splitComputedTaskForSeedRecordLimit()` / `splitSeedTaskForSeedRecordLimit()` ([ComputedUpdateWorker.ts:145-218](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L145-L218)):
- 阈值：`maxSeedRecordsPerTask`（Outbox 配置项）
- 超过阈值的任务拆分为多个 chunk
- 每个 chunk 处理一部分种子记录
- 拆分后原任务标记为 done，chunk 任务入队

#### 防级联深度限制

`MAX_STAGE_DEPTH = 50` ([ComputedUpdateWorker.ts:67](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L67)):
- 每个 follow-up task 的 `stageDepth` 递增
- 达到 50 后停止创建后续任务
- 防止链式更新无限循环

#### 任务调度模式

三种 `dispatchMode` ([HybridWithOutboxStrategy.ts:61](packages/v2/adapter-table-repository-postgres/src/record/computed/strategies/HybridWithOutboxStrategy.ts#L61)):

| dispatchMode | 说明 | 延迟 | 可靠性 |
|-------------|------|------|-------|
| `'external'` | 仅外部 Worker 轮询 | 高（轮询间隔） | 最高 |
| `'push'` | 入队后即时调度，delay ≥ 50ms | 低 | 中（崩溃可能丢失） |
| `'hybrid'` | push + external 兜底 | 低 | 最高 |

### 8.6 完整调用链路图

```
Command Handler (Create/Update/Delete)
        │
        ▼
PostgresTableRecordRepository.[writeMethod]()
        │
        ├─ 执行记录写入（INSERT/UPDATE/DELETE）
        ├─ 收集变更字段、链接关系、extraSeedRecords
        │
        ▼
runComputedUpdate*()
        │
        ├─ expandComputedSeedFieldIds()  ← 展开种子字段
        │
        ├─ shouldExecuteInline 判断
        │   ├─ true → 同步路径
        │   │   │
        │   │   ├─ planner.planStage()  ← 生成完整计划
        │   │   ├─ strategy.execute(updater, plan, context)
        │   │   │   ├─ SyncInTransactionStrategy: 循环执行所有阶段
        │   │   │   └─ HybridWithOutboxStrategy:
        │   │   │       ├─ prepareDirtyState()
        │   │   │       ├─ splitStepsByPolicy() → syncSteps + asyncSteps
        │   │   │       ├─ executePreparedSteps(syncSteps)
        │   │   │       └─ [有 asyncSteps] → outbox.enqueueOrMerge()
        │   │   │
        │   │   └─ publish RecordsBatchUpdated 事件
        │   │
        │   └─ false → 异步路径（Seed Task）
        │       │
        │       ├─ buildSeedTaskInput()
        │       ├─ outbox.enqueueSeedTask()
        │       └─ scheduleDispatch() [hybrid/push 模式]
        │
        ▼
Outbox 表（computed_update_outbox）
        │
        ▼
ComputedUpdateWorker.runOnce()
        │
        ├─ claimBatch() - 声明任务
        ├─ processSeedTask() / processComputedTask()
        │   ├─ unitOfWork.withTransaction()
        │   ├─ acquireLocks()
        │   ├─ execute() / planner.plan() + execute()
        │   ├─ collectDirtySeedGroups()
        │   ├─ planNextStage() → enqueueOrMerge() 后续阶段
        │   └─ markDone()
        │
        └─ 发布 RecordsBatchUpdated 事件
```

---

## 9. 操作类型到失效与传播模式的映射

### 9.1 UpdateImpactHint 体系

**UpdateImpactHint** ([ComputedUpdatePlanner.ts:65-68](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L65-L68)):
```typescript
type UpdateImpactHint = {
  valueFieldIds: ReadonlyArray<FieldId>;  // 值变更的字段
  linkFieldIds: ReadonlyArray<FieldId>;   // 链接关系变更的字段
};
```

**normalizeImpactHint()** ([PostgresTableRecordRepository.ts:3440-3463](packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts#L3440-L3463)):
- `linkFieldIds` 中的字段同时加入 `valueFieldIds`
- 链接字段变更同时传播值语义和链接关系语义

### 9.2 操作类型映射总表

| 操作类型 | changeType | 字段级失效逻辑 | 记录级传播模式 | 特殊处理 |
|---------|------------|---------------|---------------|---------|
| **INSERT** | `'insert'` | 包含表所有非链接字段 + 所有条件字段 + 无依赖公式 | 同 UPDATE | 所有字段视为已变更（隐式 null 值也需要计算） |
| **UPDATE（值）** | `'update'` | 值变更传播（isEdgeRelevantForValue） | 条件字段用 conditionalFiltered/ allTargetRecords；非条件字段用 linkTraversal | 过滤字段变更触发 allTargetRecords |
| **UPDATE（链接）** | `'update'` | 链接关系传播（isEdgeRelevantForLink） → 级联到值依赖 | 同 UPDATE（值） | 对称链接级联更新 |
| **DELETE** | `'delete'` | 表所有字段作为种子；额外包含条件字段的源字段 | 条件字段用 allTargetRecords；非条件字段 DELETE 时用 allTargetRecords | beforeImage 用于条件字段；extraSeedRecords 包含关联记录 |

### 9.3 INSERT 操作详解

#### 字段级失效特殊处理

`planStage()` 中 INSERT 的特殊逻辑 ([ComputedUpdatePlanner.ts:297-300](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L297-L300)):
```typescript
const planningSeedFieldIds =
  context.changeType === 'insert' && context.table
    ? context.table.getFields().map((field) => field.id())  // 所有字段！
    : impactSeedFieldIds;
```

**原因**：公式 `{textField} + ''` 在 textField 未提供（值为 null）时也需要计算。

`collectDirectAffectedFieldIds()` 中 INSERT 的特殊处理 ([ComputedUpdatePlanner.ts:1058-1073](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L1058-L1073)):
- `includeSeedsAlways = true` - 包含所有种子字段
- 包含无入边的计算字段（context-free formulas）
- 包含所有条件字段（conditionalLookup/conditionalRollup）
- 强制包含公式表达式直接引用种子字段的字段

#### 记录级传播
- 同 UPDATE 模式（条件字段根据过滤状态决定 conditionalFiltered / allTargetRecords）
- 非条件字段使用 linkTraversal

### 9.4 UPDATE 操作详解

#### 值变更传播

`isEdgeRelevantForValue()` ([ComputedUpdatePlanner.ts:389-410](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L389-L410)):
- 排除 `semantic === 'lookup_link'` 的边（链接字段本身的变更走链接传播路径）
- 其余所有边都参与值传播

#### 链接关系变更传播

`isEdgeRelevantForLink()` ([ComputedUpdatePlanner.ts:411-427](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L411-L427)):
- 仅 `kind === 'same_record'` 且 `semantic === 'lookup_link'`
- 即：查找/汇总字段依赖其链接字段的场景

**链接变更级联逻辑** ([ComputedUpdatePlanner.ts:428-483](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L428-L483)):
```
链接字段变更 → 先级联到 lookup_link 边的目标（lookup/rollup 字段）
               → 再将这些 lookup/rollup 字段作为值变更种子继续传播
```

#### 传播模式判定（非条件字段）

非条件字段的 lookup/rollup 传播模式判定 ([ComputedUpdatePlanner.ts:1648-1710](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L1648-L1710)):
- 有 `linkFieldId` → `linkTraversal`
- DELETE 且是 filtered lookup → `allTargetRecords`（原因：`filtered_lookup_delete_requires_source_record`）
- UPDATE 即使过滤字段变更也保持 `linkTraversal`（原因：链接关系本身是边界）

#### 传播模式判定（条件字段）

条件字段（conditionalLookup/conditionalRollup）的传播模式判定 ([ComputedUpdatePlanner.ts:1540-1623](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L1540-L1623)):

| 场景 | 传播模式 | 原因 |
|-----|---------|------|
| 无 filterDto | `allTargetRecords` | `conditional_missing_filter` |
| 过滤字段变更 + 有 beforeImage | `conditionalFiltered` (includeBeforeImage=true) | 精确匹配新旧状态 |
| 过滤字段变更 + 无 beforeImage | `allTargetRecords` | `conditional_filter_field_changed` |
| DELETE + 有 beforeImage | `conditionalFiltered` (includeBeforeImage=true) | 精确匹配 |
| DELETE + 无 beforeImage | `allTargetRecords` | `conditional_delete` |
| 过滤字段不在源表 | `allTargetRecords` | `conditional_filter_fields_not_in_source` |
| 仅查找值字段变更 | `conditionalFiltered` | 精确匹配过滤条件 |

### 9.5 DELETE 操作详解

#### 字段级失效特殊处理

`planStage()` 中 DELETE 的特殊逻辑 ([ComputedUpdatePlanner.ts:326-346](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L326-L346)):
- 遍历所有从种子表出发的 `cross_record` 边
- 对于 `conditional_rollup_source` / `conditional_lookup_source` 语义的边
- 将源字段加入 `valueSeedFieldIds`（确保条件字段能感知源记录删除）

#### extraSeedRecords 收集

`deleteMany()` 中关联记录收集 ([PostgresTableRecordRepository.ts:2703-2748](packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts#L2703-L2748)):
1. 收集** outgoing 链接**：当前记录通过链接字段指向的其他表记录
2. 收集** incoming 链接**：其他表通过链接字段指向当前记录的记录

这些关联记录作为 `extraSeedRecords` 参与后续脏值传播。

#### 记录级传播特殊处理

DELETE 的传播边过滤 ([ComputedUpdatePlanner.ts:1531-1538](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L1531-L1538)):
- 跳过从种子表到 `extraSeedTableIds` 中表的边
- 原因：这些表的记录已经作为 extraSeedRecords 加入种子，避免重复传播

### 9.6 传播边去重与合并

`propagationEdgeKey()` ([ComputedUpdatePlanner.ts:1473-1488](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L1473-L1488)):

去重键组成：
```
[propagationMode, fromTableId, toTableId, isSelfRefresh, linkFieldId, filterConditionKey]
```

**合并效果**：
- 多条目标字段共享相同传播路径的边会被合并
- `propagationTargetFieldIds` 字段记录所有目标字段
- `allTargetRecordsReasons` 合并所有原因
- `order` 取最小值

### 9.7 运行时传播模式降级

`ComputedFieldUpdater.propagateDirtyRecords()` 中的降级逻辑：
- 条件字段解析 filterDto 失败 → `conditional_runtime_invalid_filter`
- 条件字段 filterDto 为空 → `conditional_runtime_empty_filter`
- 条件字段 conditionSpec 无效 → `conditional_runtime_invalid_condition_spec`
- 条件字段缺少 conditionSpec → `conditional_runtime_missing_condition_spec`

降级后传播模式变为 `allTargetRecords`，确保正确性但牺牲性能。

---

## 10. 补充类与文件索引

| 类/文件 | 职责 |
|--------|------|
| `PostgresTableRecordRepository.ts` | 记录写入入口、种子展开、策略选择 |
| `SyncInTransactionStrategy.ts` | 同步策略实现 |
| `HybridWithOutboxStrategy.ts` | 混合策略（同步+异步分割） |
| `AsyncWithRetryStrategy.ts` | 异步策略（参考） |
| `ComputedUpdateWorker.ts` | Outbox Worker 执行逻辑、租期管理、任务处理 |
| `PostgresComputedUpdateOutbox.ts` | Outbox 表操作（enqueue/claim/renew/markDone） |
| `IComputedUpdateOutbox.ts` | Outbox 接口与任务类型定义 |
| `record/di/register.ts` | 依赖注入注册，策略选择 |

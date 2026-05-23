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

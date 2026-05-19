# 记录列表查询：过滤、排序与虚拟列计算的三段式协调机制分析

## 核心结论（第二次修正）

**重要修正**：
1. **所有用户查询路径都显式传递 `mode: 'stored'`**，列表查询和单条查询都是如此
2. **但仓储层存在默认分流逻辑**：当未显式传递 mode 时，含 link 字段或条件字段的表会自动走 computed 模式
3. **Computed 模式几乎仅用于更新路径**，唯一例外是 DevTools 调试接口可手动指定 mode
4. **虚拟列的值来源于预计算回填**，查询时直接读取存储列，无任何结果后处理计算

---

## 架构总览

记录列表查询与虚拟列计算采用"读写分离 + 双模式查询"的架构设计：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              写入路径（计算字段更新）                          │
│                                                                             │
│  记录变更 ────►  依赖图分析 ────►  Computed模式动态计算 ────►  UPDATE写回存储列 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                                         │
                                                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              读取路径（查询记录）                              │
│                                                                             │
│  查询请求 ────►  条件解析 ────►  调用仓储 ────►  { mode: 'stored' } ────►  返回结果 │
│             （过滤/排序）        │                                         ▲
│                                   │                                         │
│                                   └────────────────── 读取预计算回填的值 ────┘
│                                                                             │
│  【理论路径】：不传递 mode → 仓储判断表结构 → 含 link/conditional 字段走 computed │
│  【实际情况】：所有调用点都显式传递 mode: 'stored'，默认分流逻辑几乎不被触发        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 仓储层默认 mode 分流机制详解

### 2.1 resolveQueryMode 逻辑

**位置**：`packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordQueryRepository.ts:854-871`

```typescript
const resolveQueryMode = (
  table: Table,
  mode: TableRecordQueryMode | undefined
): TableRecordQueryMode => {
  if (mode) return mode;  // 1. 显式传递则优先使用
  
  // 2. 有 link 字段 → computed
  const needsComputedLinks = table
    .getFields()
    .some((field) => field.type().equals(FieldType.link()));
  if (needsComputedLinks) return 'computed';
  
  // 3. 有 conditionalRollup/conditionalLookup → computed
  const hasConditionalFields = table
    .getFields()
    .some(
      (field) =>
        field.type().equals(FieldType.conditionalRollup()) ||
        field.type().equals(FieldType.conditionalLookup())
    );
  return hasConditionalFields ? 'computed' : 'stored';  // 4. 否则 stored
};
```

**接口注释与实际行为不一致**：
- `TableRecordQueryRepository.ts:35` 注释说 `'computed'` 是默认
- 但所有实际调用点都显式传递 `mode: 'stored'`，该默认逻辑几乎不被触发

### 2.2 所有查询入口的 mode 传递情况

| 调用点 | 文件位置 | mode 传递 |
|--------|---------|-----------|
| 列表查询 API | `ListTableRecordsHandler:507-509` | 显式 `mode: 'stored'` |
| 单条查询 API | `GetRecordByIdHandler:65-69` | 显式 `mode: 'stored'` |
| 链接宿主记录查询 | `ListTableRecordsHandler:674-678` | 显式 `mode: 'stored'` |
| 更新记录前查询 | `UpdateRecordHandler:162-166` | 显式 `mode: 'stored'` |
| 重排序前查询 | `ReorderRecordsHandler:99-103` | 显式 `mode: 'stored'` |
| 粘贴操作查询 | `PasteHandler:366-370` | 显式 `mode: 'stored'` |
| 粘贴流式查询 | `PasteHandler:438-443` | 显式 `mode: 'stored'` |
| 粘贴外键查询 | `PasteHandler:2005-2009` | 显式 `mode: 'stored'` |
| DevTools 查询 | `DebugDataLive:251-253` | 默认为 `stored`，可指定 |
| DevTools 单条查询 | `DebugDataLive:295-297` | 默认为 `stored`，可指定 |
| 计算字段更新 | `ComputedFieldUpdater:1033-1045` | 显式创建 `ComputedTableRecordQueryBuilder` |

### 2.3 两种查询模式的使用场景总结

| 模式 | 使用场景 | 调用方 |
|------|---------|--------|
| `stored` | 所有用户查询路径（列表、单条、更新前读取等） | 所有 HTTP API Handler、Command Handler |
| `computed` | 计算字段更新、字段类型转换、全表回填 | `ComputedFieldUpdater`（仅内部使用） |
| `computed` | 调试诊断 | DevTools（可手动指定，默认 stored） |

---

## 第一段：接口解析层 - 过滤与排序条件解析

### 1.1 输入参数定义

**位置**：`packages/v2/core/src/queries/ListTableRecordsQuery.ts`

核心输入 Schema `listTableRecordsInputSchema` 定义：

| 参数 | 类型 | 说明 |
|------|------|------|
| `tableId` | `string` | 表格 ID（必填） |
| `filter` | `RecordFilter` | 过滤条件（AST 结构，支持 AND/OR/NOT） |
| `sort` | `Array<{fieldId, order}>` | 排序字段与方向 |
| `groupBy` | `string[]` | 分组字段 |
| `search` | `RecordSearchInput` | 搜索关键词与字段范围 |
| `viewId` | `string` | 视图 ID（用于应用视图默认过滤/排序） |
| `ignoreViewQuery` | `boolean` | 忽略视图默认查询 |
| `limit` / `offset` | `number` | 分页参数 |
| `fieldKeyType` | `'id' \| 'name'` | 字段标识方式 |

### 1.2 过滤条件解析流程

**位置**：`packages/v2/core/src/queries/ListTableRecordsHandler.ts`

#### 1.2.1 字段键解析 (`resolveFilterFieldKeys`)
```
前端传入（fieldId 可能是字段名）
    ↓
递归遍历 Filter AST 树
    ↓
FieldKeyResolverService.resolveFieldKey()
    ↓
所有条件节点的 fieldId 转换为 FieldId 格式
```

#### 1.2.2 当前用户标签替换 (`replaceCurrentUserTagInFilter`)
- 过滤值中 `"Me"` 自动替换为当前用户 ID
- 仅对 `user` / `createdBy` / `lastModifiedBy` 类型字段生效

#### 1.2.3 视图默认值合并
- 视图级过滤与查询级过滤通过 `AND` 连接
- 视图级排序与查询级排序合并，查询级优先

#### 1.2.4 过滤 AST → 领域规约 (`RecordFilterMapper.buildRecordConditionSpec`)
```
RecordFilter（条件树）
    ↓
递归遍历：
  - 条件节点 → 调用 field.spec().create({operator, value})
  - 分组节点 → AND/OR 组合规约
  - NOT 节点 → notSpec(spec) 包装
    ↓
ISpecification<TableRecord, ITableRecordConditionSpecVisitor>
```

### 1.3 排序条件解析

**位置**：`packages/v2/core/src/commands/shared/orderBy.ts`

#### 1.3.1 排序解析流程
```
前端 sort: [{fieldId, order}]
    ↓
resolveSortValues() → 字段名 → FieldId 转换 + 权限校验
    ↓
mergeSortWithViewDefaults() → 与视图默认排序合并
    ↓
resolveOrderBy() → 转换为 TableRecordOrderBy 格式
    ↓
mergeOrderBy(groupBy, sort, viewId) → 分组排序合并 + 稳定排序兜底
    ↓
最终 orderBy: Array<FieldOrderBy | SystemColumnOrderBy>
```

#### 1.3.2 稳定排序兜底策略
- 无显式排序时：`__row_{viewId}` 视图行顺序 → `__auto_number` 兜底
- 有显式排序时：始终追加 `__auto_number asc` 作为 tie-breaker

---

## 第二段：SQL 构造层 - 查询构建（双模式对比）

### 3.1 Stored 模式（所有查询路径使用）

**位置**：`packages/v2/adapter-table-repository-postgres/src/record/query-builder/stored/StoredTableRecordQueryBuilder.ts`

#### 3.1.1 核心流程
```typescript
build():
  1. 基础列选择：__id, __version, __auto_number 等系统列
  2. 字段列：StoredFieldSelectVisitor 生成 SELECT t.field_col AS field_col
     ├─ 普通字段：直接 SELECT 列
     ├─ 计算字段（link/lookup/rollup/formula）：直接 SELECT 预存储的列值
     └─ 无任何动态计算！
  3. WHERE 条件：TableRecordConditionWhereVisitor 将 ISpecification 转为 SQL
     └─ 过滤条件直接作用于存储列
  4. ORDER BY 处理：
     - 普通字段：直接按列排序，NULL 排序对齐 v1 语义
     - 用户/链接字段：按 title 列排序
     - 单选/多选字段：按选项顺序排序
     └─ 排序直接作用于存储列
  5. LIMIT/OFFSET 分页
```

#### 3.1.2 NULL 排序对齐策略
- ASC 时：`ORDER BY col IS NULL DESC, col ASC` → NULL 排在最前
- DESC 时：`ORDER BY col IS NULL ASC, col DESC` → NULL 排在最后
- 与 PostgreSQL 默认行为相反，保持与 v1 兼容

### 3.2 Computed 模式（仅更新路径使用）

**位置**：`packages/v2/adapter-table-repository-postgres/src/record/query-builder/computed/ComputedTableRecordQueryBuilder.ts`

#### 3.2.1 Lateral Join 上下文管理
```typescript
createLateralContext():
  // 按 linkFieldId + filterCondition 哈希去重
  // 多个 lookup 字段引用同一 link 字段时共享同一 LATERAL JOIN
  laterals = Map<"linkFieldId|foreignTableId|filterHash", {
    alias: "lat_fldxxx_hash",
    columns: Array<{outputAlias, columnType}>,
    condition?: FieldCondition
  }>
```

#### 3.2.2 各类计算字段的 SQL 生成

**链接字段 (LinkField)**：
```sql
INNER JOIN LATERAL (
  SELECT jsonb_agg(jsonb_build_object('id', f.__id, 'title', ...) ORDER BY ...) AS lat_col
  FROM foreign_table f
  WHERE f.__id = ANY(t.link_field_col)
) lat_fldxxx ON true
```

**查找字段 (LookupField)**：
```sql
INNER JOIN LATERAL (
  SELECT jsonb_agg(f.lookup_field_col ORDER BY ...) AS lat_col
  FROM foreign_table f
  WHERE f.__id = ANY(t.link_field_col)
) lat_fldxxx ON true
```

**汇总字段 (RollupField)**：
```sql
INNER JOIN LATERAL (
  SELECT SUM(f.value_col) AS lat_col
  FROM foreign_table f
  WHERE f.__id = ANY(t.link_field_col)
) lat_fldxxx ON true
```

### 3.3 两种模式过滤/排序的区别

| 特性 | Stored 模式 | Computed 模式 |
|------|------------|--------------|
| 过滤条件作用对象 | 直接作用于存储列 | 可作用于动态计算列（LATERAL JOIN 结果） |
| 排序条件作用对象 | 直接作用于存储列 | 可作用于动态计算列 |
| 索引支持 | 存储列可建索引，高效 | 动态计算列无法建索引，较慢 |
| 过滤计算字段 | 完全支持（读存储列） | 完全支持（读动态计算列） |
| 排序计算字段 | 完全支持（读存储列） | 完全支持（读动态计算列） |
| NULL 排序策略 | 对齐 v1，NULL 在前 | 对齐 v1，NULL 在前 |

---

## 第三段：虚拟列计算 - 预计算回填机制

### 4.1 虚拟列的存储方式

所有计算字段（link/lookup/rollup/formula/conditionalLookup/conditionalRollup）都有对应的**物理存储列**：

| 计算字段类型 | 存储列类型 | 值来源 |
|-------------|-----------|--------|
| `link` | `jsonb` | 预存储的关联记录 ID + title |
| `lookup` | 根据目标字段类型动态 | 预存储的查找结果 |
| `rollup` | `numeric` / `jsonb` | 预存储的汇总结果 |
| `formula` | 根据公式返回类型动态 | 预存储的公式计算结果 |
| `conditionalLookup` | `jsonb` | 预存储的条件查找结果 |
| `conditionalRollup` | `numeric` / `jsonb` | 预存储的条件汇总结果 |

### 4.2 预计算回填触发时机

**位置**：`packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts`

```
记录变更（INSERT/UPDATE/DELETE）
    ↓
字段变更检测：changedFieldIds
    ↓
ComputedUpdatePlanner.planStage() → 构建依赖图
    ↓
更新策略选择：
  ├─ 同步模式（SyncInTransactionStrategy）：低复杂度计算，事务内立即执行
  └─ 异步模式（HybridWithOutboxStrategy）：高复杂度计算，写入 outbox 表后台执行
    ↓
ComputedFieldUpdater.execute() → 执行更新计划
```

### 4.3 计算字段更新执行流程

**位置**：`packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldUpdater.ts`

#### 4.3.1 执行步骤
```typescript
execute(plan, context):
  1. 优化同表批量：相同表的公式链合并为一步
  2. 准备 dirty 状态：标记需要更新的记录
  3. 按拓扑排序执行每一步：
     for step in plan.steps (按 level 排序):
       a. 构建 ComputedTableRecordQueryBuilder（computed 模式！）
       b. 动态计算该步骤所有计算字段的值
       c. UpdateFromSelectBuilder 生成 UPDATE...FROM SQL
       d. 执行 UPDATE，将计算结果写回存储列
```

#### 4.3.2 Computed 模式动态计算
**位置**：`ComputedFieldUpdater.executeStep():1033-1045`

```typescript
const builder = new ComputedTableRecordQueryBuilder(db, {
  typeValidationStrategy: this.typeValidationStrategy,
  forceLookupArrayOutput: true,
})
  .from(table)
  .select(fieldIds)  // 需要计算的字段
  .withDirtyFilter({ tableId: step.tableId.toString() });

await builder.prepare({ context, tableRepository });
const selectQuery = yield* builder.build();  // 生成带 LATERAL JOIN 的 SELECT
```

#### 4.3.3 UPDATE...FROM 写回存储列
**位置**：`packages/v2/adapter-table-repository-postgres/src/record/computed/UpdateFromSelectBuilder.ts`

生成的 SQL 结构：
```sql
UPDATE target_table as u
SET 
  computed_field_1 = c.computed_field_1,
  computed_field_2 = c.computed_field_2,
  __version = __version + 1
FROM (
  -- Computed 模式动态计算的 SELECT 语句
  SELECT 
    t.__id,
    lat_fld1.lat_col AS computed_field_1,
    lat_fld2.lat_col AS computed_field_2
  FROM target_table t
  INNER JOIN LATERAL (...) lat_fld1 ON true
  INNER JOIN LATERAL (...) lat_fld2 ON true
  WHERE t.__id IN (SELECT record_id FROM tmp_computed_dirty)
) c
WHERE u.__id = c.__id
AND (
  -- 仅更新实际发生变化的行
  u.computed_field_1 IS DISTINCT FROM c.computed_field_1 OR
  u.computed_field_2 IS DISTINCT FROM c.computed_field_2
)
RETURNING u.__id, u.__version - 1 AS __old_version, ...
```

---

## 三段式衔接关系详解

### 5.1 过滤与虚拟列的衔接

```
前端过滤条件（可能包含计算字段）
    ↓
RecordFilterMapper → ISpecification
    ↓
TableRecordConditionWhereVisitor → SQL WHERE 子句
    ↓
直接作用于计算字段的存储列（Stored 模式）
    ↓
过滤生效的前提：该计算字段的预计算回填已完成
```

**关键特性**：
- 过滤条件可以引用计算字段（link/lookup/rollup/formula）
- 过滤逻辑直接在存储列上执行，无需额外 JOIN
- 如果计算字段更新滞后，过滤结果可能基于旧值（最终一致性）

### 5.2 排序与虚拟列的衔接

```
前端排序条件（可能包含计算字段）
    ↓
resolveOrderBy() → TableRecordOrderBy
    ↓
StoredTableRecordQueryBuilder.buildOrderBy() → SQL ORDER BY 子句
    ↓
直接作用于计算字段的存储列
    ↓
排序生效的前提：该计算字段的预计算回填已完成
```

**排序优化**：
- 计算字段的存储列可以建索引，支持高效排序
- 用户/链接字段按 title 排序：存储列中已包含 title 信息

### 5.3 虚拟列计算与查询的完整时序

```
时间轴：
T0: 用户创建表，添加 link 字段 + lookup 字段
    └─ 此时无数据，存储列为 NULL

T1: 用户插入记录 A（link 字段指向记录 B）
    ├─ INSERT 语句写入存储列（link 字段值为 B 的 ID）
    ├─ 触发计算字段更新计划
    ├─ ComputedFieldUpdater 异步执行：
    │   └─ 动态计算 lookup 字段值 → UPDATE 写回存储列
    └─ lookup 存储列从 NULL → 实际值

T2: 用户查询记录列表
    ├─ 显式传递 mode: 'stored'
    ├─ Stored 模式 SELECT 所有列（包括 lookup 的存储列）
    ├─ 直接返回存储列中的预计算值
    └─ 无需任何动态计算

T3: 用户修改记录 B 的值
    ├─ UPDATE 语句更新 B
    ├─ 触发反向依赖更新：所有引用 B 的记录需要更新
    ├─ ComputedFieldUpdater 异步执行，更新所有相关记录的 lookup 列
    └─ 期间查询可能短暂看到旧值（最终一致性窗口）
```

### 5.4 两种查询模式的完整链路对比

| 阶段 | Stored 模式（查询路径） | Computed 模式（更新路径） |
|------|-------------------------|---------------------------|
| **触发** | 用户 HTTP 查询请求 | 记录变更后内部更新 |
| **调用方** | ListTableRecordsHandler / GetRecordByIdHandler | ComputedFieldUpdater |
| **mode 传递** | 显式 `mode: 'stored'` | 直接创建 ComputedTableRecordQueryBuilder |
| **字段值来源** | 直接读取存储列（预计算结果） | LATERAL JOIN 动态计算 |
| **过滤** | WHERE 作用于存储列 | WHERE 可作用于动态计算列 |
| **排序** | ORDER BY 作用于存储列 | ORDER BY 可作用于动态计算列 |
| **性能** | 最快，可利用索引 | 较慢，多表 JOIN |
| **数据新鲜度** | 最终一致（可能延迟） | 实时最新 |
| **结果后处理** | 无，直接映射返回 | 无，计算结果用于 UPDATE |

---

## 核心设计决策

### 1. 为什么所有查询都显式用 Stored 模式？
- **性能**：列表查询可能返回大量记录，每次都用 LATERAL JOIN 动态计算成本过高
- **索引支持**：存储列可以建索引，支持高效的过滤和排序
- **一致性**：预计算回填保证最终一致性，用户可接受短暂延迟
- **简单性**：查询路径无需处理复杂的计算逻辑

### 2. 为什么仓储层还保留默认分流到 Computed 的逻辑？
- **历史演进**：早期设计可能计划让查询路径也使用 computed 模式
- **灵活性**：保留能力，未来如有实时性要求高的场景可切换
- **内部使用**：计算字段更新流程内部依赖该逻辑判断何时需要 computed 模式
- **但实际上**：所有外部调用都显式指定 mode，该默认逻辑仅为防御性设计

### 3. 为什么计算字段要预存储？
- 计算字段通常依赖其他表数据，每次查询都 JOIN 成本过高
- 预存储后列表查询可直接索引，支持高效过滤和排序
- 异步更新机制将计算成本分摊到写入路径

### 4. 为什么不做结果后处理（JavaScript 计算）？
- **性能**：大量记录时 JavaScript 计算是性能瓶颈
- **排序/过滤**：如果值在 JavaScript 层计算，数据库无法对其排序和过滤
- **一致性**：预存储机制保证所有查询看到相同的值
- **复杂度**：需要在应用层维护计算逻辑，与数据库层重复

### 5. 为什么用 UPDATE...FROM 而不是逐行更新？
- **批量高效**：一条 SQL 更新所有脏记录，避免 N+1 查询
- **原子性**：单条 SQL 保证原子性，无需复杂事务
- **优化空间**：PostgreSQL 优化器可以优化 UPDATE...FROM 的执行计划

---

## 关键代码索引

| 功能 | 文件位置 |
|------|---------|
| 查询输入 Schema | `packages/v2/core/src/queries/ListTableRecordsQuery.ts` |
| 查询处理核心逻辑（强制 stored 模式） | `packages/v2/core/src/queries/ListTableRecordsHandler.ts:507-509` |
| 单条查询（强制 stored 模式） | `packages/v2/core/src/queries/GetRecordByIdHandler.ts:65-69` |
| 过滤 AST → 规约 | `packages/v2/core/src/queries/RecordFilterMapper.ts` |
| 排序合并逻辑 | `packages/v2/core/src/commands/shared/orderBy.ts` |
| 仓储默认 mode 分流逻辑 | `packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordQueryRepository.ts:854-871` |
| 存储模式查询构建器 | `packages/v2/adapter-table-repository-postgres/src/record/query-builder/stored/StoredTableRecordQueryBuilder.ts` |
| 计算模式查询构建器（仅用于更新） | `packages/v2/adapter-table-repository-postgres/src/record/query-builder/computed/ComputedTableRecordQueryBuilder.ts` |
| 记录查询仓储 | `packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordQueryRepository.ts` |
| 计算字段更新器核心 | `packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldUpdater.ts` |
| 计算字段更新架构说明 | `packages/v2/adapter-table-repository-postgres/src/record/computed/ARCHITECTURE.md` |
| UPDATE...FROM 构建器 | `packages/v2/adapter-table-repository-postgres/src/record/computed/UpdateFromSelectBuilder.ts` |
| 字段输出列映射 | `packages/v2/adapter-table-repository-postgres/src/record/query-builder/FieldOutputColumnVisitor.ts` |

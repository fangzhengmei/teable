# 记录列表查询：过滤、排序与虚拟列计算的三段式协调机制分析

## 概述

记录列表查询采用清晰的三段式架构：

1. **接口解析层**：将前端传入的查询参数（过滤、排序、分页等）解析为领域对象
2. **SQL 构造层**：根据领域对象构建高效的 SQL 查询（支持存储字段与计算字段两种模式）
3. **结果后处理层**：将数据库返回的原始行映射为可读模型，计算字段值通过预存储或动态 SQL 计算获得

---

## 第一段：接口解析层 - 查询条件解析

### 1.1 输入参数定义

**位置**：`packages/v2/core/src/queries/ListTableRecordsQuery.ts`

核心输入 Schema `listTableRecordsInputSchema` 定义了所有合法查询参数：

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

**关键特性**：
- `parseJsonInput` 中间件：自动解析 JSON 字符串参数（支持前端通过 query string 传递复杂对象）
- Zod `superRefine` 校验：`filterLinkCellSelected` 与 `filterLinkCellCandidate` 互斥校验
- 默认页大小 100，最大 1000

### 1.2 过滤条件解析与转换

**位置**：`packages/v2/core/src/queries/ListTableRecordsHandler.ts` + `RecordFilterMapper.ts`

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

#### 1.2.3 视图默认值合并 (`mergeFilterWithViewDefaults`)
- 视图级过滤与查询级过滤通过 `AND` 连接
- `mergeSortWithViewDefaults` 采用 Map 去重合并，查询级排序优先于视图级

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

## 第二段：SQL 构造层 - 查询构建

### 2.1 查询构建器管理器

**位置**：`packages/v2/adapter-table-repository-postgres/src/record/query-builder/TableRecordQueryBuilderManager.ts`

#### 2.1.1 模式选择策略
```typescript
// 决策逻辑 PostgresTableRecordQueryRepository:854-871
resolveQueryMode(table, mode):
  if (mode) return mode
  if (table.hasLinkFields) return 'computed'
  if (table.hasConditionalFields) return 'computed'
  return 'stored'
```

**重要提示**：列表查询在 `ListTableRecordsHandler.handle():509` 硬编码使用 `mode: 'stored'`，因为列表场景依赖预存储值以保证性能。

#### 2.1.2 两种构建器对比

| 特性 | Stored 模式 | Computed 模式 |
|------|------------|--------------|
| 读取方式 | 直接读取列值 | 通过 LATERAL JOIN 动态计算 |
| 性能 | 最快，无额外 JOIN | 较慢，多表关联 |
| 适用场景 | 列表查询、批量操作 | 单条记录、计算字段更新 |
| 准备阶段 | 无操作 | 预加载外键关联表 |
| 典型 SQL | `SELECT t.col FROM table t` | `SELECT lat_xxx.col FROM table t INNER JOIN LATERAL (...) lat_xxx ON true` |

### 2.2 Stored 模式构建器

**位置**：`packages/v2/adapter-table-repository-postgres/src/record/query-builder/stored/StoredTableRecordQueryBuilder.ts`

#### 2.2.1 核心流程
```typescript
build():
  1. 基础列选择：__id, __version, __auto_number 等系统列
  2. 字段列：StoredFieldSelectVisitor 生成 SELECT t.field_col AS field_col
  3. WHERE 条件：TableRecordConditionWhereVisitor 将 ISpecification 转为 SQL
  4. ORDER BY 处理：
     - 普通字段：直接按列排序，NULL 排序对齐 v1 语义
     - 用户/链接字段：applyUserLikeOrderBy() → 按 title 排序
     - 单选/多选字段：applySelectChoiceOrderBy() → 按选项顺序排序
  5. LIMIT/OFFSET 分页
```

#### 2.2.2 NULL 排序对齐策略
- ASC 时：`ORDER BY col IS NULL DESC, col ASC` → NULL 排在最前
- DESC 时：`ORDER BY col IS NULL ASC, col DESC` → NULL 排在最后
- 与 PostgreSQL 默认行为相反，保持与 v1 兼容

### 2.3 Computed 模式构建器

**位置**：`packages/v2/adapter-table-repository-postgres/src/record/query-builder/computed/ComputedTableRecordQueryBuilder.ts`

#### 2.3.1 Lateral Join 上下文管理
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

#### 2.3.2 各类计算字段的 SQL 生成

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
- 嵌套 JSONB 数组处理：递归 CTE 扁平化 + 去重

**汇总字段 (RollupField)**：
```sql
INNER JOIN LATERAL (
  SELECT SUM(f.value_col) AS lat_col
  FROM foreign_table f
  WHERE f.__id = ANY(t.link_field_col)
) lat_fldxxx ON true
```

**条件汇总/查找 (ConditionalRollup/ConditionalLookup)**：
- 独立 LATERAL JOIN，通过过滤条件选择关联记录
- 支持 Fast Path：简单 `is` / `isAnyOf` 条件下使用非关联 JOIN 优化

### 2.4 仓储执行层

**位置**：`packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordQueryRepository.ts`

#### 2.4.1 完整执行流程
```typescript
find(context, table, spec, options):
  1. 创建查询构建器（含 prepare 阶段）
  2. 应用投影（projectionFieldIds）
  3. 应用排序（orderBy → 处理视图行顺序列不存在的降级）
  4. 应用分页（limit/offset）
  5. 应用过滤规约（spec → where 条件）
  6. 应用搜索条件（buildRecordSearchWhereClause）
  7. 应用显式 recordIdsOrder（CASE WHEN 自定义排序）
  8. 并行执行：
     - 数据查询：带 LIMIT/OFFSET
     - 计数查询：COUNT(*) 不带 LIMIT
  9. 结果映射：mapRowsToReadModels()
```

---

## 第三段：结果后处理层 - 虚拟列计算

### 3.1 结果集行映射

**位置**：`PostgresTableRecordQueryRepository.mapRowsToReadModels()` (L638-722)

#### 3.1.1 系统列提取
```typescript
{
  id: row.__id,                    // 记录 ID
  version: row.__version,          // 版本号（用于乐观锁/实时同步）
  autoNumber: row.__auto_number,   // 自增序号
  createdTime: row.__created_time, // 创建时间
  createdBy: row.__created_by,     // 创建人
  lastModifiedTime: row.__last_modified_time,
  lastModifiedBy: row.__last_modified_by,
  orders: { [viewId]: orderValue } // 各视图中的行顺序值
}
```

#### 3.1.2 字段值映射
```
FieldOutputColumnVisitor.collect()
    ↓
生成 fieldId → columnAlias 映射
    ↓
遍历所有字段列：
  fields[fieldId] = row[columnAlias]
  如果是 user 类字段 → normalizeStoredUserAvatarUrls()
```

#### 3.1.3 用户头像 URL 规范化
- 检测旧版头像路径 `/api/attachments/read/public/avatar/`
- 替换为通过 `buildUserAvatarUrl(id)` 生成的新版路径
- 递归处理数组和嵌套对象

### 3.2 计算字段的"预计算"机制

计算字段（link/lookup/rollup/formula/conditionalRollup/conditionalLookup）的值不是在查询后通过 JavaScript 计算的，而是采用"预存储 + 异步更新"策略：

#### 3.2.1 混合更新架构
**位置**：`packages/v2/adapter-table-repository-postgres/src/record/computed/ARCHITECTURE.md`

```
记录变更（insert/update/delete）
    ↓
ComputedUpdatePlanner 构建依赖图
    ↓
按拓扑排序生成更新步骤
    ↓
执行策略选择：
  ├─ 同步模式（SyncInTransactionStrategy）：
  │   低复杂度计算，事务内立即执行
  │   UpdateFromSelectBuilder 生成 UPDATE...FROM SQL
  │
  └─ 异步模式（HybridWithOutboxStrategy）：
      高复杂度计算，写入 outbox 表
      ComputedUpdateWorker 后台轮询执行
```

#### 3.2.2 批量更新 SQL 生成
`UpdateFromSelectBuilder` 生成高效的批量更新：
```sql
UPDATE target_table t
SET computed_col = sub.result
FROM (
  SELECT 
    t.__id, 
    AGG(f.value) as result
  FROM target_table t
  INNER JOIN LATERAL (...) lat ON true
  WHERE t.__id IN (SELECT record_id FROM tmp_dirty)
) sub
WHERE t.__id = sub.__id
```

### 3.3 两种计算模式的适用场景

| 场景 | 模式 | 说明 |
|------|------|------|
| 列表查询（GET /records） | stored | 读取预存储列值，性能最优 |
| 单条查询（GET /records/:id） | computed | 动态计算，保证实时性 |
| 记录变更后 | computed → stored | 动态计算后写回存储列 |
| 计算字段回填 | computed | 全表重新计算 |

---

## 三段式对接关键接口

### 第一段 → 第二段：`ListTableRecordsHandler` → `PostgresTableRecordQueryRepository`

**传递参数**：
```typescript
{
  spec: ISpecification<TableRecord, ITableRecordConditionSpecVisitor>,  // 过滤规约
  pagination: OffsetPagination,                                          // 分页
  orderBy: ReadonlyArray<TableRecordOrderBy>,                           // 排序
  search: RecordQuerySearch,                                            // 搜索
  mode: 'stored' | 'computed',                                          // 查询模式
  recordIdsOrder: ReadonlyArray<RecordId>                               // 显式排序
}
```

**位置**：`ListTableRecordsHandler.handle():498-511`

### 第二段 → 第三段：`PostgresTableRecordQueryRepository` → 调用方

**返回结果**：
```typescript
{
  records: ReadonlyArray<TableRecordReadModel>,  // 映射后的记录数组
  total: number                                  // 总记录数
}
```

**位置**：`PostgresTableRecordQueryRepository.find():258`

---

## 核心设计决策

### 1. 为什么列表查询强制用 stored 模式？
- 列表查询通常返回大量记录，computed 模式的多 LATERAL JOIN 会导致 O(n) 次子查询
- 预存储值通过异步更新保证最终一致性，用户可接受短暂延迟
- 单条记录查询使用 computed 模式保证操作时的实时性

### 2. 为什么计算字段要预存储？
- 计算字段通常依赖其他表数据，每次查询都 JOIN 成本过高
- 预存储后列表查询可直接索引，支持高效过滤和排序
- 异步更新机制将计算成本分摊到写入路径

### 3. 为什么过滤条件要转换成 Specification 模式？
- Specification 是领域层的纯对象，与持久化技术无关
- 通过 Visitor 模式可轻松支持多种数据库（目前 PostgreSQL，未来可扩展）
- 规约对象可被组合、缓存、序列化，便于复杂查询构建

### 4. 为什么排序需要特殊处理 NULL 顺序？
- PostgreSQL 默认 `ASC NULLS LAST, DESC NULLS FIRST`
- 为与 v1 版本兼容，显式指定 `ASC NULLS FIRST, DESC NULLS LAST`
- 用户/链接/选择字段需要自定义排序逻辑（按 title、按选项顺序）

---

## 关键代码索引

| 功能 | 文件位置 |
|------|---------|
| 查询输入 Schema | `packages/v2/core/src/queries/ListTableRecordsQuery.ts` |
| 查询处理核心逻辑 | `packages/v2/core/src/queries/ListTableRecordsHandler.ts` |
| 过滤 AST → 规约 | `packages/v2/core/src/queries/RecordFilterMapper.ts` |
| 排序合并逻辑 | `packages/v2/core/src/commands/shared/orderBy.ts` |
| 存储模式查询构建器 | `packages/v2/adapter-table-repository-postgres/src/record/query-builder/stored/StoredTableRecordQueryBuilder.ts` |
| 计算模式查询构建器 | `packages/v2/adapter-table-repository-postgres/src/record/query-builder/computed/ComputedTableRecordQueryBuilder.ts` |
| 查询构建器管理器 | `packages/v2/adapter-table-repository-postgres/src/record/query-builder/TableRecordQueryBuilderManager.ts` |
| 记录查询仓储 | `packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordQueryRepository.ts` |
| 计算字段更新架构说明 | `packages/v2/adapter-table-repository-postgres/src/record/computed/ARCHITECTURE.md` |
| 字段输出列映射 | `packages/v2/adapter-table-repository-postgres/src/record/query-builder/FieldOutputColumnVisitor.ts` |

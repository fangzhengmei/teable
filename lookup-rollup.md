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
                                         saveRawOps 存入 CLS → 事务提交发布 → 前端缓存失效
```

**关键文件位置：**
- 字段模型: `packages/core/src/models/field/derivate/`
- 计算编排: `apps/nestjs-backend/src/features/record/computed/services/`
- 引用图: `apps/nestjs-backend/src/features/calculation/reference.service.ts`
- BatchService: `apps/nestjs-backend/src/features/calculation/batch.service.ts`

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

### 2.3 链接关系类型 (LinkRelationship)

**文件**: `packages/v2/field-dependency-core/src/types.ts:82`

```typescript
export type LinkRelationship = 'oneMany' | 'manyOne' | 'oneOne' | 'manyMany';
```

**重要修正**：链接字段支持**四种关系类型**，不只是 `manyMany`。不同关系类型的存储模型有本质区别，需要明确表述边界：

| 关系类型 | 说明 | 存储模型 | fkHostTableName 指向 | selfKeyName | foreignKeyName |
|---------|------|---------|-------------------|-------------|---------------|
| `manyMany` | 多对多 | **独立 junction table** | junction table 限定名 | 对称字段ID | 当前字段ID |
| `manyOne` | 多对一 | **当前表加外键列** | 当前表 dbTableName | `__id` | 外键列名 |
| `oneOne` | 一对一 | **当前表加外键列** | 当前表 dbTableName | `__id` | 外键列名 |
| `oneMany` (非单向) | 一对多（双向） | **外表加外键列** | 外表 foreignTableName | 外键列名 | `__id` |
| `oneMany` (单向) | 一对多（单向） | **独立 junction table** | junction table 限定名 | 对称字段ID | 当前字段ID |

**关键结论**：
- ✅ `manyMany` 和 单向 `oneMany`：使用独立的 junction table
- ❌ `manyOne`、`oneOne`、双向 `oneMany`：**不使用** junction table，而是在现有表上加外键列

**链接选项配置**（`packages/v2/field-dependency-core/src/types.ts:101-109`）：
```typescript
export interface ParsedLinkOptions {
  foreignTableId: string;
  lookupFieldId: string;
  isOneWay?: boolean;
  symmetricFieldId?: string;
  /** FK host table name (format: baseDbName.tableDbName) */
  fkHostTableName?: string;
  /** Link relationship type */
  relationship?: LinkRelationship;
}
```

### 2.4 查找选项模式 (Lookup Options)

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

#### 4.2.1 ILinkEdge 动态列名接口

**重要校正**：链接级联查询的列名**不是固定字段**，而是通过 `ILinkEdge` 接口动态拼装。其中 `fkTableName` 来源于链接字段配置的 `fkHostTableName`。

```typescript
// link-cascade-resolver.ts:8-14
export interface ILinkEdge {
  foreignTableId: string;
  hostTableId: string;
  fkTableName: string;        // junction table 的限定名（格式：baseDbName.tableDbName）
  selfKeyName: string;        // 动态主键列名（如 "fldxxxx"）
  foreignKeyName: string;     // 动态外键列名（如 "fldyyyy"）
}
```

**边的动态构建**（`computed-dependency-collector.service.ts:256-266`）：
```typescript
const opts = this.parseLinkOptions(field.options);
edges.push({
  foreignTableId: opts.foreignTableId,
  hostTableId: tableId,
  fkTableName: opts.fkHostTableName,    // 从链接字段配置读取，格式：baseDbName.tableDbName
  selfKeyName: opts.selfKeyName,        // 从链接字段配置读取
  foreignKeyName: opts.foreignKeyName,  // 从链接字段配置读取
});
```

**fkHostTableName 与存储模型的关系**（表述边界修正）：
- `fkHostTableName` 是**外键所在表的限定名**，格式为 `baseDbName.tableDbName`，**不一定是 junction table**
- `formatQualifiedName` 方法将其按 `.` 分割，分别用双引号包裹，形成真正的 SQL 引用：`"baseDbName"."tableDbName"`
- 仅当 `manyMany` 或 单向 `oneMany` 时，`fkHostTableName` 才指向独立的 junction table
- 对于 `manyOne`、`oneOne`、双向 `oneMany`，`fkHostTableName` 指向**现有业务表**，通过外键列存储链接关系
- 链接级联查询时，`fetchEdgeTargets` 通过动态的 `selfKeyName` 和 `foreignKeyName` 正确访问不同存储模型

**限定名拼装**（`link-cascade-resolver.ts:221-226`）：
```typescript
private formatQualifiedName(qualified: string): string {
  return qualified
    .split('.')
    .map((part) => this.quoteIdentifier(part))  // '"' + part.replace(/"/g, '""') + '"'
    .join('.');
}
```

#### 4.2.2 BFS 遍历算法（动态列名拼装）

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
      // 通过 junction table 查询关联记录（动态列名拼装）
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

#### 4.2.3 Junction Table 查询（动态列名）

**文件**: `link-cascade-resolver.ts:175-215`

```typescript
private async fetchEdgeTargets(
  edge: ILinkEdge,
  srcIds: string[]
): Promise<Array<{ record_id?: string }>> {
  const placeholders = srcIds.map((_, i) => `$${i + 1}`).join(', ');
  // 将 baseDbName.tableDbName 转为 "baseDbName"."tableDbName"
  const fkTableRef = this.formatQualifiedName(edge.fkTableName);
  // 动态列名：edge.foreignKeyName 和 edge.selfKeyName 来自链接字段配置
  const srcCol = this.quoteIdentifier(edge.foreignKeyName);
  const dstCol = this.quoteIdentifier(edge.selfKeyName);
  
  const sql = `select ${dstCol}::text as record_id
from ${fkTableRef}
where ${srcCol} in (${placeholders})
  and ${srcCol} is not null
  and ${dstCol} is not null`;
  
  return await this.dataPrismaService
    .txClient()
    .$queryRawUnsafe<Array<{ record_id?: string }>>(sql, ...srcIds);
}
```

**全表扫描版本**（`fetchEdgeTargetsFromAll`）：
```typescript
private async fetchEdgeTargetsFromAll(edge: ILinkEdge): Promise<Array<{ record_id?: string }>> {
  const fkTableRef = this.formatQualifiedName(edge.fkTableName);
  // 同样使用动态列名
  const srcCol = this.quoteIdentifier(edge.foreignKeyName);
  const dstCol = this.quoteIdentifier(edge.selfKeyName);
  
  const sql = `select distinct ${dstCol}::text as record_id
from ${fkTableRef}
where ${srcCol} is not null
  and ${dstCol} is not null`;
  // ...
}
```

**设计意图**：
- 每个链接字段都有独立的 junction table 和独立的列名配置
- 动态拼装避免硬编码，支持多租户、多数据库实例
- 通过 `selfKeyName`/`foreignKeyName` 映射到实际的数据库列（通常为 `fld` 前缀的字段ID）

**优化策略**:
- 分批处理（每批 500 条，`IN_CHUNK = 500`）避免 IN 子句过长
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

### 5.2 条件汇总 ALL_RECORDS 完整触发条件（去重后共 10 个）

**文件**: `apps/nestjs-backend/src/features/record/computed/services/computed-dependency-collector.service.ts:745-981`

`getConditionalRollupImpactedRecordIds` 方法中共有 **10 个独立的触发点**（去掉了重复的 JSON 类型检查），按检查顺序排列：

#### 5.2.1 前半段：前置检查（7 个触发点）

```typescript
private async getConditionalRollupImpactedRecordIds(
  edge: IConditionalRollupAdjacencyEdge,
  foreignRecordIds: string[],
  changeContextMap?: Map<string, ICellContext[]>,
  ctx?: ICollectorExecutionContext
): Promise<string[] | typeof ALL_RECORDS> {
```

| # | 触发条件 | 代码位置 | 判定逻辑 |
|---|---------|---------|---------|
| ① | 样本量超限 | 755-757 | `uniqueForeignIds.length > MAX_CONDITIONAL_ROLLUP_SAMPLE` (阈值 = 10,000) |
| ② | 无过滤器 | 762-765 | `!filter` 条件汇总未配置过滤条件 |
| ③ | 无主机字段引用 | 767-770 | `!hostFieldRefs.length` 过滤器不引用主机表任何字段 |
| ④ | 无外键字段引用 | 772-774 | `foreignFieldIds.size === 0` 过滤器不引用外表任何字段 |
| ⑤ | 跨表引用 | 776-778 | 过滤器引用了非主机表的字段 (`ref.tableId && ref.tableId !== edge.tableId`) |
| ⑥ | 主机字段加载失败 | 780-784 | `hostFieldMap.size !== uniqueHostFieldIds.length` |
| ⑦ | 外键字段加载失败 | 786-793 | `foreignFieldMap.size !== foreignFieldIds.size` |
| ⑧ | JSON 类型字段 | 795-800 | 外表过滤字段的 `dbFieldType === DbFieldType.Json` |

> **注意**：代码中原先第 ⑧ 项存在**重复检查**（795-800 行和 802-806 行），去重后计为 1 个触发条件。

#### 5.2.2 后半段：变更前后双端过滤（新增 3 个触发点）

当 `changeContextMap` 存在（记录变更场景）时，需要对变更前后的值都应用过滤器，取并集确保完整性。此阶段新增 **3 个 ALL_RECORDS 触发点**：

| # | 触发条件 | 代码位置 | 判定逻辑 |
|---|---------|---------|---------|
| ⑨ | DB列名缺失 | 897-899 | `foreignDbFieldNamesOrdered.length !== foreignFieldIds.size`，无法获取字段的数据库列名 |
| ⑩ | 字段值缺失 | 949-963 | 构造 `updatedRows` 时，字段不存在或 `dbFieldName` 不在 base 对象中 |
| ⑪ | 矩阵值未定义 | 979-981 | `valuesMatrix.some((row) => row.some((value) => typeof value === 'undefined'))`，值矩阵包含 undefined |

**触发点 ⑨ 详细分析**（889-899 行）：
```typescript
const foreignDbFieldNamesOrdered = Array.from(
  new Set(
    Array.from(foreignFieldIds)
      .map((fid) => foreignFieldMap.get(fid)?.dbFieldName)
      .filter((name): name is string => !!name)
  )
);

if (foreignDbFieldNamesOrdered.length !== foreignFieldIds.size) {
  return ALL_RECORDS;  // 某些字段没有 dbFieldName，无法构建查询
}
```

**触发点 ⑩ 详细分析**（949-963 行）：
```typescript
let missing = false;
for (const fieldId of foreignFieldIds) {
  const field = foreignFieldMap.get(fieldId);
  if (!field) { missing = true; break; }              // 字段不存在
  if (!(field.dbFieldName in base)) { missing = true; break; }  // 值缺失
}
if (missing) {
  return ALL_RECORDS;
}
```

**触发点 ⑪ 详细分析**（979-981 行）：
```typescript
if (valuesMatrix.some((row) => row.some((value) => typeof value === 'undefined'))) {
  return ALL_RECORDS;  // 值矩阵包含 undefined，无法安全绑定参数
}
```

#### 5.2.3 双端过滤的完整流程（记录变更场景）

当有 `changeContextMap` 时，执行两次 EXISTS 查询取并集：

```
1. 先查当前值：用数据库中的当前值构建 EXISTS 子查询 → 结果集 ids
2. 再查变更后的值：
   a. 查询变更记录的当前数据库值 → baseRows
   b. 用 newValue 替换变更字段的值 → updatedRows
   c. 将 updatedRows 转为 VALUES 子查询（CAST 到正确类型）
   d. 用变更后的值构建 EXISTS 子查询 → postRows
3. 合并结果：ids ∪ postRows → 返回 Array.from(ids)
```

**变更后值的类型转换**（986-1008 行）：
```typescript
const resolveColumnType = (column: string): string => {
  if (column === '__id') return 'text';
  const field = foreignFieldByDbName.get(column);
  switch (field?.dbFieldType) {
    case DbFieldType.Integer: return 'integer';
    case DbFieldType.Real: return 'double precision';
    case DbFieldType.Boolean: return 'boolean';
    case DbFieldType.DateTime: return 'timestamp';
    case DbFieldType.Blob: return 'bytea';
    case DbFieldType.Json: return 'jsonb';
    case DbFieldType.Text:
    default: return 'text';
  }
};
```

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

### 7.1 两条迭代路径对比

计算编排器通过两个不同入口调用两种收集方法，虽然迭代算法结构相似，但在多个关键维度存在差异：

| 对比维度 | 记录变更 (`collect`) | 字段变更 (`collectForFieldChanges`) |
|---------|-------------------|----------------------------------|
| **调用场景** | 记录数据增删改 | 字段定义增删改 |
| **初始种子** | `explicitSeeds` 包含变更记录ID + 对称链接预播种ID | `explicitSeeds` 为空 |
| **初始 ALL_RECORDS** | `tablesWithAllRecords` 初始为空 | `tablesWithAllRecords` 初始化为 `originTableIds` |
| **上下文传递** | 有 `changeContextMap`（oldValue/newValue），用于变更前后双端过滤 | 无 `changeContextMap`，仅使用当前值过滤 |
| **条件汇总查询** | 先查当前值 → 再查变更后的值 → 取并集 | 仅查当前值一次 |
| **分页策略** | `preferAutoNumberPaging` 仅在 `ALL_RECORDS` 时设置 | 源表默认 `preferAutoNumberPaging = true` |
| **历史兼容** | 无 `fallbackLookupIds` 处理 | 有 `fallbackLookupIds`，兼容 `lookupOptions.linkFieldId` 之前的历史数据 |
| **自动编号字段** | 有 `autoNumberFieldIds` 处理 | 无 |
| **plannedForeignRecordIds** | 有，用于对称链接预播种 | 无 |

### 7.2 字段变更路径的历史兼容处理

**文件**: `computed-dependency-collector.service.ts:1203-1232`

字段变更场景额外处理了历史兼容性问题：

```typescript
const fallbackLookupIds = new Set<string>();
if (relatedLinkIds.length) {
  const byTable = await this.findLookupsByLinkIds(relatedLinkIds);
  for (const [tid, fset] of Object.entries(byTable)) {
    const group = (impact[tid] ||= {
      fieldIds: new Set<string>(),
      recordIds: new Set<string>(),
    });
    fset.forEach((fid) => {
      if (!group.fieldIds.has(fid)) {
        group.fieldIds.add(fid);
        fallbackLookupIds.add(fid);
      }
    });
  }
}

if (fallbackLookupIds.size) {
  // Legacy compatibility: pre-link reference rows created before lookupOptions.linkFieldId
  // existed do not include the link→lookup edge. We need to synthesize those missing
  // dependencies so downstream lookups/formulas still recompute.
  const extraDeps = await this.collectDependentFieldsByTable(Array.from(fallbackLookupIds));
  // ... 补充额外的依赖字段
}
```

### 7.3 记录变更路径的 autoNumber 处理

**文件**: `computed-dependency-collector.service.ts:1608-1609`

```typescript
const autoNumberFieldIds = this.getAutoNumberFieldIds(entryDomain, excludeFieldIds);
this.addContextFreeFormulasToImpact(impact, tableId, autoNumberFieldIds);
```

**设计意图**：自动编号字段不依赖其他字段，每次记录创建时都会变化，需要单独加入受影响集合。

### 7.4 迭代收敛完整流程

**文件**: `computed-dependency-collector.service.ts:1257-1395`（字段变更场景）和 `1637-1778`（记录变更场景）

两个场景的**迭代算法结构完全一致**，以下以记录变更场景为例：

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

### 7.5 记录集增长检测 (findRecordSetGrowth)

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

### 7.6 去重入队机制

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

### 7.7 收敛判定

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

### 8.3 publishBatch 与 saveRawOps 调用链（真实字段结构）

**重要修正**：`publishBatch` 不直接发布 ops，而是通过 `batchService.saveRawOps` 存入 CLS 上下文，在事务提交时才真正发布。

**完整调用链与数据结构**：
```
computed-evaluator.service.ts:publishBatch()
    │
    ├─ 输入: evaluatedRows = [{ recordId, version, prevVersion, fields }]
    │
    ├─ 为每个字段构建 JSON0 Op: { p: ['fields', fid], oi: newValue, od: oldValue }
    │
    ├─ 组装 opDataList: [{ docId, version: prevVersion ?? version, data: ops[] }]
    │
    ↓
batch.service.ts:saveRawOps(collectionId, RawOpType.Edit, IdPrefix.Record, dataList)
    │
    ├─ collection = `${docType}_${collectionId}`  // "rec_tbl000..."
    │
    ├─ 构建 rawOpMap: {
    │     [collection]: {
    │       [docId]: {
    │         src: 'xxx',      // CLS 请求ID
    │         seq: 1,
    │         m: { ts: 123 }, // 时间戳
    │         op: IOtOperation[],  // JSON0 操作数组
    │         v: version     // 文档版本号
    │       }
    │     }
    │   }
    │
    ↓
存入 cls.get('tx.rawOpMaps') → 事务提交时从 CLS 取出批量发布
```

#### 8.3.1 publishBatch 完整实现（真实字段结构）

**文件**: `computed-evaluator.service.ts:337-388`

```typescript
private publishBatch(
  tableId: string,
  impactedFieldIds: Set<string>,
  validFieldIds: Set<string>,
  excludeFieldIds: Set<string>,
  evaluatedRows: Array<{
    recordId: string;    // 记录ID
    version: number;     // 更新后的新版本号（__version + 1）
    prevVersion?: number;// 更新前的旧版本号（__version）
    fields: Record<string, unknown>;  // 字段ID -> 新值
  }>
): number {
  if (!evaluatedRows.length) return 0;

  const targetFieldIds = Array.from(impactedFieldIds).filter(
    (fid) => validFieldIds.has(fid) && !excludeFieldIds.has(fid)
  );
  if (!targetFieldIds.length) return 0;

  const opDataList = evaluatedRows
    .map(({ recordId, version, prevVersion, fields }) => {
      const ops = targetFieldIds
        .map((fid) => {
          const hasValue = Object.prototype.hasOwnProperty.call(fields, fid);
          const newCellValue = hasValue ? fields[fid] : null;
          // 构建 JSON0 OT 操作，oldCellValue 固定为 null
          return RecordOpBuilder.editor.setRecord.build({
            fieldId: fid,
            newCellValue,
            oldCellValue: null,
          });
        })
        .filter(Boolean) as IOtOperation[];

      if (!ops.length) return null;

      // 版本号优先级：prevVersion（更新前） > version（更新后）
      const opVersion = prevVersion ?? version;

      return { 
        docId: recordId,    // 文档ID = 记录ID
        version: opVersion, // ShareDB 操作的基线版本
        data: ops,          // JSON0 操作数组
        count: ops.length   // 操作计数
      } as const;
    })
    .filter(Boolean) as { docId: string; version: number; data: IOtOperation[]; count: number }[];

  if (!opDataList.length) return 0;

  // 调用 saveRawOps 存入 CLS 上下文
  this.batchService.saveRawOps(
    tableId,
    RawOpType.Edit,
    IdPrefix.Record,
    opDataList.map(({ docId, version, data }) => ({ docId, version, data }))
  );

  return opDataList.reduce((sum, current) => sum + current.count, 0);
}
```

#### 8.3.2 SetRecordBuilder 构建 JSON0 Op（真实三种形式）

**文件**: `packages/core/src/op-builder/record/set-record.ts:5-43`

**Op 上下文接口**：
```typescript
export interface ISetRecordOpContext {
  name: OpName.SetRecord;
  fieldId: string;
  newCellValue: unknown;
  oldCellValue: unknown;
}
```

**JSON0 Op 真实构建逻辑**（三种形式）：
```typescript
build(params: { fieldId: string; newCellValue: unknown; oldCellValue: unknown }): IOtOperation {
  const { fieldId } = params;
  let { newCellValue, oldCellValue } = params;
  newCellValue = newCellValue ?? null;
  oldCellValue = oldCellValue ?? null;

  // 1. 新值为 null 或空数组 → 删除键（od + oi: null）
  if (newCellValue == null || (Array.isArray(newCellValue) && newCellValue.length === 0)) {
    return {
      p: ['fields', fieldId],  // path: ["fields", "fldxxx"]
      od: oldCellValue,        // old delete: 被删除的旧值
      oi: null,                // old insert: 标记为删除
    };
  }

  // 2. 旧值为 null → 插入键（仅 oi）
  if (oldCellValue == null) {
    return {
      p: ['fields', fieldId],
      oi: newCellValue,        // old insert: 插入的新值
    };
  }

  // 3. 新旧值都存在 → 替换（od + oi）
  return {
    p: ['fields', fieldId],
    od: oldCellValue,        // old delete: 被替换的旧值
    oi: newCellValue,        // old insert: 新值
  };
}
```

#### 8.3.3 saveRawOps 完整实现（真实字段结构）

**文件**: `batch.service.ts:458-519`

```typescript
@Timing()
saveRawOps(
  collectionId: string,    // 表ID
  opType: RawOpType,       // Create | Edit | Del
  docType: IdPrefix,       // Record = "rec"
  dataList: { docId: string; version: number; data?: unknown }[]
) {
  // collection 命名: "rec_tbl000000000000000000"
  const collection = `${docType}_${collectionId}`;
  const rawOpMap: IRawOpMap = { [collection]: {} };

  // 基础元数据
  const baseRaw = {
    src: this.cls.getId() || 'unknown',  // 请求来源ID
    seq: 1,                              // 序列号
    m: {
      ts: Date.now(),                    // 时间戳
    },
  };

  this.logger.verbose(`saveOp: ${baseRaw.src}-${collection}`);

  dataList.forEach(({ docId, version, data }) => {
    let rawOp: IRawOp;
    if (opType === RawOpType.Create) {
      rawOp = {
        ...baseRaw,
        create: {
          type: 'json0',
          data,
        },
        v: version,    // 版本号
      };
    } else if (opType === RawOpType.Del) {
      rawOp = {
        ...baseRaw,
        del: true,
        v: version,
      };
    } else if (opType === RawOpType.Edit) {
      rawOp = {
        ...baseRaw,
        op: data as IOtOperation[],  // JSON0 OT 操作数组
        v: version,                  // 基线版本号
      };
    } else {
      throw new CustomHttpException(...);
    }
    rawOpMap[collection][docId] = rawOp;
  });

  // 存入 CLS 上下文，事务提交时发布
  const prevMap = this.cls.get('tx.rawOpMaps') || [];
  prevMap.push(rawOpMap);
  this.cls.set('tx.rawOpMaps', prevMap);
  return rawOpMap;
}
```

#### 8.3.4 IOtOperation 完整接口

**文件**: `packages/core/src/models/op.ts:5-15`

```typescript
// ot-type from https://github.com/ottypes/json0
export type IOTPath = (string | number)[];

export interface IOtOperation {
  p: IOTPath;           // path: (string | number)[]
  na?: number;          // number add
  li?: any;             // list insert
  ld?: any;             // list delete
  lm?: number;          // list move
  oi?: any;             // object insert
  od?: any;             // object delete
  si?: string;          // string insert
  sd?: string;          // string delete
  t?: string;           // type
  o?: any;              // object (额外字段)
}
```

### 8.4 SQL 更新执行 (RecordComputedUpdateService)

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

### 9.1 代码可证实事实（逐条校验结果）

以下内容均已在源码中定位，每条事实标注精确的文件和行号：

---

#### 9.1.1 版本号自增机制

✅ **校验通过**：有两处独立实现，均可在源码中定位

**证据 1 - SQL UPDATE 语句中的版本自增**：
**精确位置**: `apps/nestjs-backend/src/db-provider/postgres.provider.ts:474-475`

```typescript
// bump version on target table; qualify to avoid ambiguity with FROM subquery columns
updateColumns['__version'] = this.knex.raw('?? + 1', [`${dbTableName}.__version`]);
```

**证据 2 - BatchService 批量更新中的版本自增**：
**精确位置**: `apps/nestjs-backend/src/features/calculation/batch.service.ts:440-442`

```typescript
return {
  id: recordId,
  values: {
    ...Object.entries(updateParam).reduce<{ [dbFieldName: string]: unknown }>(...),
    __version: version + 1,  // 版本号自增
  },
};
```

**证据 3 - RETURNING 子句返回旧版本号**：
**精确位置**: `apps/nestjs-backend/src/db-provider/postgres.provider.ts:482-484`

```typescript
// also return previous version for ShareDB op version alignment
const returningAll = [
  ...qualifiedReturning,
  this.knex.raw('?? - 1 as __prev_version', [`${dbTableName}.__version`]),
];
```

**校验结论**：每次数据库更新时 `__version` 都会自增，有三处独立代码证据。

---

#### 9.1.2 publishBatch 调用 saveRawOps

✅ **校验通过**：调用链完整可追踪

**精确位置**: `apps/nestjs-backend/src/features/record/computed/services/computed-evaluator.service.ts:379-385`

```typescript
this.batchService.saveRawOps(
  tableId,
  RawOpType.Edit,
  IdPrefix.Record,
  opDataList.map(({ docId, version, data }) => ({ docId, version, data }))
);
```

**调用上下文**（`computed-evaluator.service.ts:132-140`）：
```typescript
await strategy.run(paginationContext, async (rows) => {
  if (!rows.length) return;
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

**校验结论**：`publishBatch` 确实调用 `saveRawOps`，而非直接发布。

---

#### 9.1.3 saveRawOps 存入 CLS 上下文

✅ **校验通过**：有明确的 CLS 存取代码

**精确位置**: `apps/nestjs-backend/src/features/calculation/batch.service.ts:514-518`

```typescript
// 存入 CLS 上下文，事务提交时发布
const prevMap = this.cls.get('tx.rawOpMaps') || [];
prevMap.push(rawOpMap);
this.cls.set('tx.rawOpMaps', prevMap);
return rawOpMap;
```

**CLS 键名**: `tx.rawOpMaps`（数组类型，每个元素是一个 `IRawOpMap`）

**校验结论**：ops 被存入 CLS 的 `tx.rawOpMaps` 键下，事务提交时才真正发布。

---

#### 9.1.4 Op 结构为 JSON0 格式

✅ **校验通过**：有接口定义和构建代码双重证据

**接口定义精确位置**: `packages/core/src/models/op.ts:1-17`

```typescript
// ot-type from https://github.com/ottypes/json0
export type IOTPath = (string | number)[];

export interface IOtOperation {
  p: IOTPath;           // path: (string | number)[]
  na?: number;          // number add
  li?: any;             // list insert
  ld?: any;             // list delete
  lm?: number;          // list move
  oi?: any;             // object insert
  od?: any;             // object delete
  si?: string;          // string insert
  sd?: string;          // string delete
  t?: string;           // type
  o?: any;              // object (额外字段)
}
```

**Op 构建精确位置**: `packages/core/src/op-builder/record/set-record.ts:15-43`

```typescript
build(params: { fieldId: string; newCellValue: unknown; oldCellValue: unknown }): IOtOperation {
  // 1. 新值为 null 或空数组 → 删除键
  if (newCellValue == null || (Array.isArray(newCellValue) && newCellValue.length === 0)) {
    return {
      p: ['fields', fieldId],  // path: ["fields", "fldxxx"]
      od: oldCellValue,        // old delete: 被删除的旧值
      oi: null,                // old insert: 标记为删除
    };
  }
  // 2. 旧值为 null → 插入键
  if (oldCellValue == null) {
    return {
      p: ['fields', fieldId],
      oi: newCellValue,
    };
  }
  // 3. 新旧值都存在 → 替换
  return {
    p: ['fields', fieldId],
    od: oldCellValue,
    oi: newCellValue,
  };
}
```

**校验结论**：Op 确实是标准 JSON0 格式，包含 `p`（path）、`oi`（object insert）、`od`（object delete）等字段。

---

#### 9.1.5 evaluatedRows 包含 version 和 prevVersion

✅ **校验通过**：有构建逻辑和使用逻辑双重证据

**构建逻辑精确位置**: `apps/nestjs-backend/src/features/record/computed/services/computed-evaluator.service.ts:304-335`

```typescript
private buildEvaluatedRows(
  rows: Array<IComputedRowResult>,
  fieldInstances: IFieldInstance[]
): Array<{
  recordId: string;
  version: number;
  prevVersion?: number;
  fields: Record<string, unknown>;
}> {
  return rows.map((row) => {
    const recordId = row.__id;
    const version = row.__version as number;           // 更新后的新版本号
    const prevVersion = row.__prev_version as number | undefined;  // 更新前的旧版本号
    // ... 构建 fields 映射 ...
    return { recordId, version, prevVersion, fields: fieldsMap };
  });
}
```

**使用逻辑精确位置**: `apps/nestjs-backend/src/features/record/computed/services/computed-evaluator.service.ts:354-376`

```typescript
const opDataList = evaluatedRows
  .map(({ recordId, version, prevVersion, fields }) => {
    // ... 构建 ops ...
    const opVersion = prevVersion ?? version;  // 优先使用旧版本号作为基线
    return { docId: recordId, version: opVersion, data: ops, count: ops.length };
  })
```

**校验结论**：`evaluatedRows` 确实包含 `version`（新）和 `prevVersion`（旧），优先使用 `prevVersion` 作为 op 的基线版本号。

---

#### 9.1.6 ISetRecordOpContext 字段结构

✅ **校验通过**：接口定义完整

**精确位置**: `packages/core/src/op-builder/record/set-record.ts:5-11`

```typescript
export interface ISetRecordOpContext {
  name: OpName.SetRecord;
  fieldId: string;
  newCellValue: unknown;
  oldCellValue: unknown;
}
```

**detect 方法精确位置**: `packages/core/src/op-builder/record/set-record.ts:45-59`

```typescript
detect(op: IOtOperation): ISetRecordOpContext | null {
  const { p, oi, od } = op;
  const result = pathMatcher<{ fieldId: string }>(p, ['fields', ':fieldId']);
  if (!result) return null;
  return {
    name: this.name,
    fieldId: result.fieldId,
    newCellValue: oi,
    oldCellValue: od,
  };
}
```

**校验结论**：接口包含 `name`、`fieldId`、`newCellValue`、`oldCellValue` 四个字段，`detect` 方法验证了字段对应关系。

---

#### 9.1.7 __prev_version 的来源

✅ **校验通过**：SQL 层面计算旧版本号

**精确位置**: `apps/nestjs-backend/src/db-provider/postgres.provider.ts:479-484`

```typescript
// also return previous version for ShareDB op version alignment
const returningAll = [
  ...qualifiedReturning,
  // Unqualified reference to target table column to avoid FROM-clause issues
  this.knex.raw('?? - 1 as __prev_version', [`${dbTableName}.__version`]),
];
```

**SQL 语义**：`__version - 1 as __prev_version`，即通过新版本号减 1 得到旧版本号。

**校验结论**：`__prev_version` 是在 SQL RETURNING 子句中通过 `__version - 1` 计算得出的。

---

#### 9.1.8 版本号在 RETURNING 子句中的返回

✅ **校验通过**：新版本号和旧版本号同时返回

**精确位置**: `apps/nestjs-backend/src/db-provider/postgres.provider.ts:477-484`

```typescript
const returningCols = [idFieldName, '__version', ...(returningDbFieldNames || dbFieldNames)];
const qualifiedReturning = returningCols.map((c) => this.knex.ref(`${dbTableName}.${c}`));
// also return previous version for ShareDB op version alignment
const returningAll = [
  ...qualifiedReturning,
  this.knex.raw('?? - 1 as __prev_version', [`${dbTableName}.__version`]),
];
```

**返回字段顺序**：
1. `idFieldName` (`__id`)
2. `__version`（更新后的新版本号）
3. 其他业务字段
4. `__prev_version`（更新前的旧版本号，通过计算得出）

**校验结论**：UPDATE 语句的 RETURNING 子句同时返回新旧版本号，供后续 ShareDB op 使用。

---

### 9.2 推断（合理推测）

以下内容基于代码架构和实现逻辑的合理推测，未找到直接代码证据：

#### 9.2.1 前端缓存失效机制

**推测**：前端接收到 ShareDB op 后，通过 `__version` 匹配失效本地缓存。

**推理依据**：
- Op 中携带了最新的 `__version`（来自数据库自增后的值）
- 前端本地缓存通常按版本号管理，当收到更高版本号时，旧版本缓存失效
- 这是 ShareDB/OT 系统的标准缓存失效模式

#### 9.2.2 缓存失效传播链

**推测**：完整的缓存失效链为：
```
源记录更新 → 版本号+1 → SQL UPDATE → saveRawOps 存入 CLS → 事务提交发布 → WebSocket推送 → 前端接收Op → 版本号匹配 → 失效本地缓存 → 触发重新查询/渲染
```

**推理依据**：
- 后端通过 WebSocket 推送 ops 是实时协作系统的标准做法
- 前端需要根据推送的变更更新本地状态，避免重新加载整个表格
- 版本号机制可以确保不会应用过期的更新

---

## 十、关键设计决策

### 10.1 双重依赖追踪

| 机制 | 优点 | 缺点 |
|------|-----|-----|
| reference 表 SQL CTE | 高效，支持任意深度递归 | 依赖数据完整性，历史数据可能缺失 |
| lookup_options 查询 | 准确，直接匹配字段配置 | 仅支持一层 lookup，需要递归补充 |

### 10.2 ALL_RECORDS 标记

避免全表记录ID物化，当条件汇总样本量超过阈值（**10,000 条**，由 `MAX_CONDITIONAL_ROLLUP_SAMPLE` 定义）或其他 9 种条件触发时，直接标记为全表重算。

### 10.3 对称链接双端预播种

从 `oldValue` 和 `newValue` 两端提取记录ID，确保链接的**添加**和**删除**都能正确触发对称链接端的重算。

### 10.4 条件边迭代收敛

通过 `dirty` 标志 + `findRecordSetGrowth` 检测 + 重新 `computeLinkClosure` 的循环，确保条件汇总的跨表影响被完整传播，直到没有新记录发现为止。

### 10.5 拓扑分层执行

确保依赖字段按正确顺序计算，避免读取到过期值。

### 10.6 基于 SQL 的批量更新

相比逐行更新，`UPDATE FROM SELECT` 性能提升显著，且保证原子性。

### 10.7 动态列名拼装

链接级联查询的列名通过 `ILinkEdge` 接口的 `selfKeyName`、`foreignKeyName`、`fkTableName` 动态读取自链接字段配置：
- `fkHostTableName` 是 `baseDbName.tableDbName` 格式的限定名，**不一定是 junction table**
- 通过 `formatQualifiedName` 转为 `"baseDbName"."tableDbName"` 的 SQL 引用
- `selfKeyName` 和 `foreignKeyName` 根据关系类型动态变化：`__id` 或外键列名（`__fk_fldxxx`）
- 仅 `manyMany` 和 单向 `oneMany` 使用独立 junction table，其他关系类型复用现有业务表
- 避免硬编码，支持多租户和灵活的数据库架构

### 10.8 CLS 延迟发布

通过 `saveRawOps` 将 ops 存入 CLS 上下文，在事务提交时才真正发布，确保：
- 原子性：事务提交失败时不会发布部分变更
- 性能：批量发布减少推送次数
- 一致性：确保数据库状态和发布的 ops 一致

---

## 十一、代码优化建议

### 潜在问题 1: 重复的 JSON 类型检查

**文件**: `computed-dependency-collector.service.ts:795-806`

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

### 潜在问题 2: ALL_RECORDS 符号重复定义

`ALL_RECORDS` 在 `link-cascade-resolver.ts:32` 和 `computed-dependency-collector.service.ts:71` 分别定义，可考虑提取为共享常量。

### 潜在问题 3: computeLinkClosure 参数不一致

`computeLinkClosure` 方法在两个调用场景中参数不完全一致：
- 字段变更场景：无 `tableDomains` 参数（内部 fallback 加载）
- 记录变更场景：传入 `tableDomains` 参数

可考虑统一参数传递，避免重复加载 TableDomain。

---

## 十二、总结

Teable 的查找与汇总字段的值传播链路是一个**精心设计的分布式计算管道**，其核心优势在于：

1. **完整性**: 通过 reference 表 + lookup_options 双重机制确保依赖不遗漏
2. **性能**: BFS 级联 + 批量 SQL 更新 + 分页策略，兼顾效率与内存
3. **正确性**: 拓扑排序 + 版本号乐观锁 + 行级锁，保证并发安全
4. **实时性**: CLS 延迟发布 + ShareDB 推送，确保前端缓存及时失效
5. **健壮性**: 10 种 ALL_RECORDS 兜底条件（去重后） + 对称链接预播种 + 条件边迭代收敛，确保极端场景下的正确性

**三大核心机制的协同作用**：
- **ALL_RECORDS 触发**：在样本量过大、过滤器复杂、字段类型特殊或变更值不完整时，优雅降级为全表重算，避免内存溢出和 SQL 错误
- **对称链接预播种**：通过 oldValue/newValue 双端提取，确保链接关系变更的两端都能被正确处理
- **条件边迭代收敛**：通过单调增长的记录集和队列去重机制，确保条件汇总的跨表影响被完整传播

**动态列名设计**：
链接级联查询的列名通过 `ILinkEdge` 接口动态拼装，`fkHostTableName` 是 `baseDbName.tableDbName` 格式的限定名，通过 `formatQualifiedName` 转为 SQL 引用。**表述边界**：仅 `manyMany` 和 单向 `oneMany` 使用独立 junction table，`manyOne`、`oneOne`、双向 `oneMany` 复用现有业务表，通过 `selfKeyName` 和 `foreignKeyName` 动态映射 `__id` 或外键列名，避免硬编码，支持多租户架构。

**Op 发布链路**：
`publishBatch` 不直接推送，而是调用 `batchService.saveRawOps` 将 ops 存入 CLS 上下文的 `tx.rawOpMaps` 键下，在事务提交时才真正发布。Op 结构为标准 JSON0 格式（`{ p, oi, od }`），携带版本号确保一致性。

**两条路径的差异化设计**：
- **记录变更**：侧重精确性，通过 `changeContextMap` 实现变更前后双端过滤，最小化重算范围
- **字段变更**：侧重完整性，初始全表标记 + 历史兼容处理，确保字段定义变更不遗漏任何记录

该架构成功解决了多维表格中跨表引用、派生字段重算、缓存一致性等核心难题，为复杂数据模型提供了可靠的计算基础设施。

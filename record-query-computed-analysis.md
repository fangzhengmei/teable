# 记录列表查询：过滤、排序与虚拟列计算的三段式协调机制分析

## 核心结论（第四次修订）

**重要修订**：
1. 明确区分两类仓储：`ITableRepository`（表元数据，无 mode） vs `ITableRecordQueryRepository`（记录查询，有 mode）
2. Schema 检查器查询的是表元数据，**不涉及记录查询，不存在 mode 概念**
3. DevTools 有两个真正的记录查询入口，**支持 mode 配置，默认 stored**
4. 统一真实用户访问链路判定：仅 `contract-http-implementation` 暴露的 HTTP 接口
5. computed 在生产链路永不出现，仅在调试链路手动指定或计算字段更新流程中出现

---

## 仓储接口与 mode 概念澄清

### 2.1 两类仓储接口的本质区别

| 接口 | 职责 | 是否涉及记录查询 | mode 概念 |
|------|------|-----------------|-----------|
| `ITableRepository` | 表元数据 CRUD（表结构、字段定义、视图定义等） | ❌ 否 | ❌ 不存在 |
| `ITableRecordQueryRepository` | 表中记录数据的查询（列表、单条、流式） | ✅ 是 | ✅ 有 stored/computed 选项 |

**关键代码证据**：
- `ITableRepository` 定义：`TableRepository.ts:59` - 操作 `Table` 领域对象
- `ITableRecordQueryRepository` 定义：`TableRecordQueryRepository.ts:163` - 操作 `TableRecordReadModel` 记录数据

### 2.2 mode 配置仅适用于记录查询

**mode 参数的定义位置**：`ITableRecordQueryRepository` 的 `find()` / `findOne()` / `findStream()` 方法的 options 中。

**结论**：只有调用 `ITableRecordQueryRepository` 的方法时，mode 配置才有意义。

---

## 读取入口分类与 mode 来源（修订版）

### 3.1 分类总览

| 类别 | 典型场景 | 涉及记录查询 | mode 来源 | 触发 computed | 真实用户访问 |
|------|---------|-------------|-----------|--------------|-------------|
| **线上业务接口** | 列表查询 API、单条查询 API | ✅ 是 | 显式 `stored` | ❌ 不会 | ✅ 是 |
| **内部写入前读取** | 更新前查询、重排序前查询等 | ✅ 是 | 显式 `stored` | ❌ 不会 | ❌ 否（内部流程） |
| **DevTools 记录查询** | DevTools listRecords/getRecord | ✅ 是 | 默认 `stored`，可手动指定 | ⚠️ 仅手动指定时 | ❌ 否（开发环境） |
| **Schema 检查器** | checkTable/checkField | ❌ 否（仅表元数据查询） | N/A | ❌ 不涉及 | ❌ 否（开发工具） |

### 3.2 第一类：线上业务接口（真实用户访问链路）

**判定标准**：通过 `contract-http-implementation` 暴露的 HTTP 接口，可被外部用户访问。

| 接口 | HTTP 路由 | 调用仓储 | mode 传递位置 | mode 值 |
|------|----------|---------|---------------|---------|
| 列表查询 | `GET /tables/:tableId/records` | `ITableRecordQueryRepository.find()` | `ListTableRecordsHandler:507-509` | 显式 `stored` |
| 单条查询 | `GET /tables/:tableId/records/:recordId` | `ITableRecordQueryRepository.findOne()` | `GetRecordByIdHandler:65-69` | 显式 `stored` |

**链路示例（列表查询）**：
```
用户浏览器 / API 调用
    ↓
HTTP 服务器 → contract-http-implementation: listTableRecords.ts:15
    ↓
ListTableRecordsQuery.create()  // 参数校验与解析
    ↓
queryBus.execute()
    ↓
ListTableRecordsHandler.handle():507
    { mode: 'stored' }  // 显式指定
    ↓
ITableRecordQueryRepository.find()
    ↓
StoredTableRecordQueryBuilder  // 读取存储列
    ↓
返回结果给用户
```

### 3.3 第二类：内部写入前读取（非用户直接访问）

**判定标准**：写入操作执行前需要先读取当前数据的内部流程，不直接暴露给用户。

| 场景 | 调用仓储 | mode 传递位置 | mode 值 |
|------|---------|---------------|---------|
| 更新记录前读取 | `ITableRecordQueryRepository.findOne()` | `UpdateRecordHandler:162-166` | 显式 `stored` |
| 重排序前读取 | `ITableRecordQueryRepository.find()` | `ReorderRecordsHandler:99-103` | 显式 `stored` |
| 粘贴前计数查询 | `ITableRecordQueryRepository.find()` | `PasteHandler:366-370` | 显式 `stored` |
| 粘贴流式读取 | `ITableRecordQueryRepository.findStream()` | `PasteHandler:438-443` | 显式 `stored` |
| 粘贴外键查询 | `ITableRecordQueryRepository.find()` | `PasteHandler:2005-2009` | 显式 `stored` |
| 批量更新前读取 | `ITableRecordQueryRepository.find()` | `RecordBulkUpdateService:938-943` | 显式 `stored` |

**链路示例（更新记录前读取）**：
```
用户提交更新请求 → HTTP 接口
    ↓
UpdateRecordHandler.handle():162
    const currentRecord = tableRecordQueryRepository.findOne(
      context, table, command.recordId,
      { mode: 'stored', includeOrders: true }  // 显式指定
    )
    ↓
读取旧值用于：乐观锁校验、变更历史、事件通知
    ↓
执行更新操作
    ↓
触发计算字段异步更新
```

### 3.4 第三类：DevTools 记录查询（开发环境）

**判定标准**：通过 `devtools` 包提供的调试接口，仅开发环境可用。

**代码位置**：`packages/v2/devtools/src/layers/DebugDataLive.ts`

| 方法 | 调用仓储 | mode 配置 | 默认 mode | 代码位置 |
|------|---------|-----------|-----------|---------|
| `listRecords()` | `ITableRecordQueryRepository.find()` | `options?.mode ?? 'stored'` | `stored` | `DebugDataLive:251-254` |
| `getRecord()` | `ITableRecordQueryRepository.findOne()` | `mode ?? 'stored'` | `stored` | `DebugDataLive:295-297` |

**完整代码证据**：
```typescript
// DebugDataLive:251-254
const queryResult = await recordQueryRepo.find(context, table, undefined, {
  mode: options?.mode ?? 'stored',  // 可选，默认 stored
  pagination,
});

// DebugDataLive:295-297
const result = await recordQueryRepo.findOne(context, table, recordIdResult.value, {
  mode: mode ?? 'stored',  // 可选，默认 stored
});
```

**说明**：
- DevTools 是唯一允许手动指定 `mode: 'computed'` 的记录查询入口
- 仅用于开发调试，生产环境不暴露
- 默认值仍为 `stored`

### 3.5 Schema 检查器（不涉及记录查询）

**代码位置**：`packages/v2/devtools/src/layers/SchemaCheckerLive.ts`

| 方法 | 调用仓储 | 说明 | 是否涉及记录查询 |
|------|---------|------|-----------------|
| `checkTable()` | `ITableRepository.findOne()` | 查询表元数据，检查表结构是否正确 | ❌ 否（仅表定义） |
| `checkField()` | `ITableRepository.findOne()` | 查询表元数据，检查字段结构是否正确 | ❌ 否（仅表定义） |

**代码证据**：
```typescript
// SchemaCheckerLive:47
const tableResult = await tableRepo.findOne(context, tableSpec);
// tableRepo 是 ITableRepository，查询 Table 领域对象（表结构定义）
// 不涉及记录数据查询，不存在 mode 概念
```

**结论**：Schema 检查器完全不涉及记录查询，与 mode 配置无关。

### 3.6 仓储默认分流逻辑（理论路径，实际不触发）

**位置**：`PostgresTableRecordQueryRepository.ts:854-871`

```typescript
const resolveQueryMode = (table, mode) => {
  if (mode) return mode;                              // 显式传递优先
  if (table.hasLinkFields) return 'computed';           // 有 link 字段 → computed
  if (table.hasConditionalFields) return 'computed';    // 有条件字段 → computed
  return 'stored';                                      // 否则 stored
};
```

**实际触发情况分析**：

| 调用方 | 是否传递 mode | 实际使用模式 | 是否触发默认分流 |
|--------|-------------|-------------|-----------------|
| 线上业务接口 | ✅ 显式 `stored` | `stored` | ❌ 否 |
| 内部写入前读取 | ✅ 显式 `stored` | `stored` | ❌ 否 |
| DevTools（默认） | ✅ 默认 `stored` | `stored` | ❌ 否 |
| DevTools（指定 computed） | ✅ 显式 `computed` | `computed` | ❌ 否 |
| 计算字段更新流程 | ❌ 不调用此函数 | 直接创建 Computed 构建器 | ❌ 否 |

**结论**：`resolveQueryMode` 默认分流逻辑在**真实链路中永不被触发**，仅为防御性设计。

---

## computed 模式出现条件汇总

### 4.1 生产链路（永不出现 computed）

**生产环境下，computed 模式不会出现在任何查询路径中**：

| 场景 | 模式 | 原因 |
|------|------|------|
| 用户列表查询 | `stored` | 显式指定 |
| 用户单条查询 | `stored` | 显式指定 |
| 更新操作前读取 | `stored` | 显式指定 |
| 重排序操作前读取 | `stored` | 显式指定 |
| 粘贴操作前读取 | `stored` | 显式指定 |

### 4.2 调试链路（可手动触发 computed）

| 场景 | 模式 | 触发条件 |
|------|------|---------|
| DevTools 列表查询 | 默认 `stored` | 手动指定 `mode: 'computed'` 时触发 |
| DevTools 单条查询 | 默认 `stored` | 手动指定 `mode: 'computed'` 时触发 |

### 4.3 内部更新链路（computed 用于计算）

| 场景 | 模式 | 说明 |
|------|------|------|
| 计算字段更新 | `computed` | `ComputedFieldUpdater.executeStep()` 直接创建 Computed 构建器 |
| 字段类型转换 | `computed` | 内部使用，不经过仓储接口 |
| 全表回填 | `computed` | 内部使用，不经过仓储接口 |

---

## 三段式衔接关系详解

### 5.1 第一段：接口解析层 - 过滤与排序条件解析

**四类入口的差异**：

| 特性 | 线上业务接口 | 内部写入前读取 | DevTools 记录查询 | Schema 检查器 |
|------|-------------|---------------|-----------------|--------------|
| 涉及记录查询 | ✅ 是 | ✅ 是 | ✅ 是 | ❌ 否 |
| 过滤条件解析 | ✅ 完整支持（AST → Specification） | ✅ 支持（通常仅用 ID 查询） | ✅ 支持（简单条件） | N/A |
| 排序条件解析 | ✅ 完整支持（视图排序 + 查询排序合并） | ❌ 通常无排序 | ✅ 支持（简单排序） | N/A |
| 搜索条件解析 | ✅ 支持 | ❌ 无 | ✅ 支持 | N/A |
| 视图默认值合并 | ✅ 支持 | ❌ 无视图上下文 | ✅ 支持 | N/A |
| 用户标签替换 ("Me") | ✅ 支持 | ❌ 内部流程无前端输入 | ❌ 无 | N/A |
| mode 配置 | 显式 `stored` | 显式 `stored` | 默认 `stored`，可指定 | N/A |

**线上业务接口完整解析流程**：
```
前端传入 filter/sort/search
    ↓
ListTableRecordsQuery.create()  // Zod 校验
    ↓
ListTableRecordsHandler.handle():
  1. resolveFilterFieldKeys()    // 字段名 → FieldId
  2. replaceCurrentUserTagInFilter()  // "Me" → 用户ID
  3. mergeFilterWithViewDefaults()    // 与视图过滤 AND 合并
  4. RecordFilterMapper.buildRecordConditionSpec()  // AST → ISpecification
  5. resolveSortValues()        // 字段名 → FieldId
  6. mergeSortWithViewDefaults()    // 与视图排序合并
  7. resolveOrderBy()           // → TableRecordOrderBy
  8. mergeOrderBy()             // + 稳定排序兜底
    ↓
传递给仓储层：
{
  spec: ISpecification,
  orderBy: TableRecordOrderBy[],
  pagination: OffsetPagination,
  search: RecordQuerySearch,
  mode: 'stored'  // 显式指定
}
```

### 5.2 第二段：SQL 构造层 - 查询构建

**两类记录查询模式的区别**（Schema 检查器不涉及，不列入）：

| 特性 | Stored 模式（所有真实链路 + DevTools 默认） | Computed 模式（仅 DevTools 手动指定） |
|------|--------------------------------------------|---------------------------------------|
| 字段值来源 | 直接读取存储列 | LATERAL JOIN 动态计算 |
| 过滤作用对象 | 存储列 | 动态计算列 |
| 排序作用对象 | 存储列 | 动态计算列 |
| 索引支持 | 存储列可建索引，高效 | 动态列无法建索引 |
| 过滤计算字段 | 完全支持 | 完全支持 |
| 排序计算字段 | 完全支持 | 完全支持 |
| NULL 排序策略 | 对齐 v1，NULL 在前 | 对齐 v1，NULL 在前 |
| 性能 | 最快 | 较慢（多表 JOIN） |
| 数据新鲜度 | 最终一致 | 实时最新 |
| 适用场景 | 生产环境所有查询 | 调试对比 stored/computed 差异 |

**Stored 模式构建流程**（所有真实链路）：
```
StoredTableRecordQueryBuilder.build():
  1. SELECT 系统列：__id, __version, __auto_number, __created_time 等
  2. SELECT 字段列：
     - 普通字段：直接 SELECT t.field_col AS field_col
     - 计算字段：直接 SELECT t.computed_col AS computed_col
     └─ 无任何动态计算！
  3. WHERE：TableRecordConditionWhereVisitor 将 ISpecification 转为 SQL
     └─ 过滤条件直接作用于存储列
  4. ORDER BY：
     - 普通字段：ORDER BY t.field_col
     - 计算字段：ORDER BY t.computed_col
     - 用户/链接字段：按 title 列排序
     - 单选/多选字段：按选项顺序排序
     └─ 排序直接作用于存储列
  5. LIMIT/OFFSET 分页
```

**Computed 模式构建流程**（仅 DevTools 调试用）：
```
ComputedTableRecordQueryBuilder.build():
  1. SELECT 系统列
  2. 为每个计算字段创建 LATERAL JOIN：
     - link 字段：INNER JOIN LATERAL (...) lat_fldxxx ON true
     - lookup 字段：INNER JOIN LATERAL (...) lat_fldxxx ON true
     - rollup 字段：INNER JOIN LATERAL (...) lat_fldxxx ON true
     - formula 字段：表达式计算
  3. SELECT 动态计算列：lat_fldxxx.lat_col AS computed_col
  4. WHERE：作用于动态计算列（可能性能较差）
  5. ORDER BY：作用于动态计算列（可能性能较差）
  6. LIMIT/OFFSET 分页
```

### 5.3 第三段：虚拟列计算 - 预计算回填机制

**虚拟列计算完全独立于查询路径**，仅在写入路径触发：

| 特性 | 说明 |
|------|------|
| 触发时机 | 记录 INSERT/UPDATE/DELETE 后 |
| 计算模式 | Computed 模式（LATERAL JOIN 动态计算） |
| 计算位置 | `ComputedFieldUpdater.executeStep()` |
| 结果写回 | 通过 `UPDATE...FROM` 写回存储列 |
| 与查询的关系 | 查询直接读取预计算结果，无感知 |
| 一致性模型 | 最终一致（异步更新） |

**完整时序**：
```
T0: 用户创建表，添加 link 字段 + lookup 字段
    └─ 此时无数据，存储列为 NULL

T1: 用户插入记录 A（link 字段指向记录 B）
    ├─ INSERT 语句写入存储列（link 字段值为 B 的 ID）
    ├─ 触发计算字段更新计划（ComputedUpdatePlanner）
    ├─ ComputedFieldUpdater 异步执行：
    │   └─ 动态计算 lookup 字段值 → UPDATE 写回存储列
    └─ lookup 存储列从 NULL → 实际值

T2: 用户查询记录列表（HTTP API）
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

---

## 过滤、排序、虚拟列计算三者的衔接关系

### 6.1 衔接总览

```
写入路径：记录变更 → 依赖图分析 → Computed模式动态计算 → UPDATE写回存储列
                                                                  │
                                                                  ▼
查询路径（所有记录查询入口）：
  接口解析层：过滤/排序条件 → ISpecification + OrderBy
                    │
                    ▼
  SQL 构造层（Stored模式 / Computed模式）：
    WHERE spec(列) + ORDER BY 列
                    │
                    ▼
  结果层：直接返回列值（无后处理）
```

### 6.2 过滤与虚拟列的衔接

```
前端过滤条件（可引用计算字段）
    ↓
RecordFilterMapper → ISpecification
    ↓
TableRecordConditionWhereVisitor → SQL WHERE 子句
    ↓
作用于计算字段的列（存储列或动态计算列）
    ↓
Stored 模式：过滤生效的前提是预计算回填已完成
Computed 模式：实时计算，无需等待回填
```

**关键特性**：
- 过滤条件可以引用计算字段（link/lookup/rollup/formula）
- Stored 模式：过滤逻辑直接在存储列上执行，无需额外 JOIN
- Computed 模式：过滤逻辑作用于动态计算列，可能性能较差
- 如果计算字段更新滞后，Stored 模式过滤结果可能基于旧值（最终一致性）

### 6.3 排序与虚拟列的衔接

```
前端排序条件（可引用计算字段）
    ↓
resolveOrderBy() → TableRecordOrderBy
    ↓
QueryBuilder.buildOrderBy() → SQL ORDER BY 子句
    ↓
作用于计算字段的列（存储列或动态计算列）
    ↓
Stored 模式：排序生效的前提是预计算回填已完成
Computed 模式：实时计算，无需等待回填
```

**排序优化**：
- Stored 模式：计算字段的存储列可以建索引，支持高效排序
- Computed 模式：动态计算列无法建索引，排序性能较差
- 用户/链接字段按 title 排序：Stored 模式存储列中已包含 title 信息

### 6.4 虚拟列计算与查询的解耦设计

```
┌─────────────────────┐     独立异步     ┌─────────────────────┐
│   写入路径（更新）   │ ───────────────► │   计算字段回填       │
│  (同步/异步执行)     │                  │  (Computed 模式)      │
└─────────────────────┘                  └───────────┬─────────┘
                                                     │
                                                     ▼
                                               存储列更新
                                                     │
┌─────────────────────┐                              │
│   查询路径（读取）   │ ◄────────────────────────────┘
│  (默认 Stored 模式)   │
│  线上接口 / 内部读取 │
│  DevTools            │
└─────────────────────┘
```

**设计意图**：
- 读写分离：将复杂计算从读路径转移到写路径
- 性能优先：读路径尽可能简单高效
- 最终一致：接受短暂延迟，保证系统整体吞吐量

---

## 四类入口的完整对比

| 对比维度 | 线上业务接口 | 内部写入前读取 | DevTools 记录查询 | Schema 检查器 |
|---------|-------------|---------------|-----------------|--------------|
| **真实用户访问** | ✅ 是 | ❌ 否（内部流程） | ❌ 否（开发环境） | ❌ 否（开发工具） |
| **涉及记录查询** | ✅ 是 | ✅ 是 | ✅ 是 | ❌ 否（仅表元数据） |
| **mode 来源** | 显式 `stored` | 显式 `stored` | 默认 `stored`，可手动指定 | N/A |
| **触发 computed** | ❌ 永不 | ❌ 永不 | ⚠️ 仅手动指定时 | ❌ 不涉及 |
| **过滤条件** | 复杂 AST（用户输入） | 简单（仅 ID 查询） | 简单条件 | N/A |
| **排序条件** | 复杂（视图 + 查询合并） | 无（按 ID 查询） | 简单排序 | N/A |
| **字段投影** | 按需选择（前端指定） | 全字段（用于变更检测） | 全字段 | N/A |
| **分页** | 支持 | 不支持（批量 ID 查询） | 支持 | N/A |
| **搜索** | 支持 | 不支持 | 支持 | N/A |
| **权限校验** | 完整（行级/列级） | 跳过（已在写入入口校验） | 完整（开发权限） | N/A |
| **性能要求** | 最高（面向用户） | 高（影响写入延迟） | 低（调试用） | 低（工具用） |
| **数据新鲜度** | 最终一致 | 最终一致（但刚写入可能更新中） | 可选择实时（computed 模式） | N/A |

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

### 5. 为什么 DevTools 允许手动指定 computed 模式？
- **调试需求**：开发人员需要对比 stored 和 computed 结果的差异
- **问题定位**：当计算字段值不一致时，可以直接查询实时值
- **非生产环境**：DevTools 仅在开发环境可用，不影响生产性能

### 6. 为什么 Schema 检查器不涉及记录查询？
- Schema 检查器的职责是验证表结构定义与数据库物理结构的一致性
- 它不需要读取表中的记录数据，只需要表的元数据定义
- 因此它使用 `ITableRepository` 而不是 `ITableRecordQueryRepository`

---

## 关键代码索引

| 功能 | 文件位置 |
|------|---------|
| 列表查询 Handler（显式 stored） | `packages/v2/core/src/queries/ListTableRecordsHandler.ts:507-509` |
| 单条查询 Handler（显式 stored） | `packages/v2/core/src/queries/GetRecordByIdHandler.ts:65-69` |
| 更新前读取（显式 stored） | `packages/v2/core/src/commands/UpdateRecordHandler.ts:162-166` |
| 重排序前读取（显式 stored） | `packages/v2/core/src/commands/ReorderRecordsHandler.ts:99-103` |
| 粘贴前查询（显式 stored） | `packages/v2/core/src/commands/PasteHandler.ts:366-370` |
| DevTools 列表查询（可选 mode） | `packages/v2/devtools/src/layers/DebugDataLive.ts:251-254` |
| DevTools 单条查询（可选 mode） | `packages/v2/devtools/src/layers/DebugDataLive.ts:295-297` |
| Schema 检查器（仅表元数据查询） | `packages/v2/devtools/src/layers/SchemaCheckerLive.ts:47` |
| 仓储默认 mode 分流逻辑 | `packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordQueryRepository.ts:854-871` |
| ITableRepository 接口（表元数据） | `packages/v2/core/src/ports/TableRepository.ts:59` |
| ITableRecordQueryRepository 接口（记录查询） | `packages/v2/core/src/ports/TableRecordQueryRepository.ts:163` |
| 存储模式查询构建器 | `packages/v2/adapter-table-repository-postgres/src/record/query-builder/stored/StoredTableRecordQueryBuilder.ts` |
| 计算模式查询构建器 | `packages/v2/adapter-table-repository-postgres/src/record/query-builder/computed/ComputedTableRecordQueryBuilder.ts` |
| 计算字段更新器 | `packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldUpdater.ts` |
| 计算字段更新架构说明 | `packages/v2/adapter-table-repository-postgres/src/record/computed/ARCHITECTURE.md` |
| 过滤 AST → 规约 | `packages/v2/core/src/queries/RecordFilterMapper.ts` |
| 排序合并逻辑 | `packages/v2/core/src/commands/shared/orderBy.ts` |

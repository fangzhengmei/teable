# Adapter 与 Repository 分层抽象梳理

## 一、分层架构概览

本项目采用**六边形架构（端口与适配器模式），数据访问层分为两层抽象：

```
┌─────────────────────────────────────────────────────────────┐
│                  Application Layer                       │
│  (commands/handlers, application services)         │
└─────────────────────────────┬───────────────────────────────┘
                       │ 依赖（面向接口）
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Repository 层（Ports - 端口）                │
│  定义抽象接口，描述"做什么"，不关心"怎么做"            │
└─────────────────────────────┬───────────────────────────────┘
                       │ 实现（面向具体技术）
                       ▼
┌─────────────────────────────────────────────────────────────┐
│               Adapter 层（Adapters - 适配器）            │
│  具体技术实现，针对特定数据库/驱动                       │
└─────────────────────────────────────────────────────────────┘
```

## 二、Repository 层（端口层）

### 2.1 位置与职责

**位置**: `packages/v2/core/src/ports/`

**核心职责**：
- 定义数据访问的抽象契约（接口）
- 描述"做什么（what）"，而非"怎么做（how）"
- 属于领域层，不依赖任何具体数据库技术
- 为应用层提供稳定的持久化抽象

### 2.2 核心 Repository 接口

| 接口 | 文件 | 职责 |
|------|------|------|
| `IBaseRepository` | `ports/BaseRepository.ts` | Base 基础数据的增删查 |
| `ITableRepository` | `ports/TableRepository.ts` | Table 元数据的增删改查（table_meta, field, view 等系统表） |
| `ITableRecordRepository` | `ports/TableRecordRepository.ts` | 表记录的 DML 操作（INSERT/UPDATE/DELETE） |
| `ITableRecordQueryRepository` | `ports/TableRecordQueryRepository.ts` | 表记录的查询操作 |
| `ITableSchemaRepository` | `ports/TableSchemaRepository.ts` | 物理表结构的 DDL 操作（CREATE TABLE, ALTER TABLE 等） |
| `ISchemaOperationRepository` | `ports/SchemaOperationRepository.ts` | Schema 变更操作记录 |
| `IUnitOfWork` | `ports/UnitOfWork.ts` | 事务管理抽象 |

### 2.3 Repository 接口设计特点

**使用 Specification 模式**：

```typescript
// ports/TableRepository.ts:59-99
export interface ITableRepository {
  insert(context: IExecutionContext, table: Table): Promise<Result<Table, DomainError>>;
  
  findOne(
    context: IExecutionContext,
    spec: ISpecification<Table, ITableSpecVisitor>,  // 查询规约
    options?: Pick<TableFindOptions, 'state'>
  ): Promise<Result<Table, DomainError>>;
  
  updateOne(
    context: IExecutionContext,
    table: Table,
    mutateSpec: ISpecification<Table, ITableSpecVisitor>  // 变更规约
  ): Promise<Result<TableUpdatePersistResult | void, DomainError>>;
}
```

**关键设计**：
- 所有方法接收 `IExecutionContext` 作为第一个参数，用于传递事务、追踪、actor 信息
- 返回类型统一使用 `Result<T, DomainError>` 进行错误处理
- 查询和更新通过 `ISpecification` 规约对象描述，而非直接暴露 SQL
- 规约对象通过 **Visitor 模式** 由 adapter 层解释执行

### 2.4 依赖方向

**Repository 层不依赖 Adapter 层**：
- 应用层 → 依赖 Repository 接口（ports）
- Repository 接口 → 仅依赖领域模型（domain）
- 无任何数据库特定的导入

```typescript
// 正确：Repository 接口仅依赖领域模型
import type { Table } from '../domain/table/Table';
import type { ISpecification } from '../domain/shared/specification/ISpecification';
```

### 2.5 内存实现（用于测试）

**位置**: `packages/v2/core/src/ports/memory/`

```typescript
// ports/memory/MemoryTableRepository.ts:15-99
export class MemoryTableRepository implements ITableRepository {
  private readonly savedTables: Table[] = [];
  
  async insert(_: IExecutionContext, table: Table): Promise<Result<Table, DomainError>> {
    // 内存数组操作，无任何数据库依赖
  }
  
  async findOne(
    _: IExecutionContext,
    spec: ISpecification<Table, ITableSpecVisitor>
  ): Promise<Result<Table, DomainError>> {
    const found = this.savedTables.find((t) => spec.isSatisfiedBy(t));
    // 直接使用规约的 isSatisfiedBy 方法进行内存过滤
  }
}
```

**用途**：
- 单元测试无需启动真实数据库
- 快速开发环境快速迭代
- 验证规约逻辑的正确性

## 三、Adapter 层（适配器层）

### 3.1 位置与职责

**位置**: `packages/v2/adapter-*/`

**核心职责**：
- 实现 Repository 接口，针对具体数据库技术
- 负责将领域规约转换为具体的数据库操作
- 管理数据库连接、事务、SQL 构建
- 处理数据库特定的错误和异常

### 3.2 Adapter 包划分

项目采用**细粒度分包**，每个 adapter 包负责特定的适配职责：

| Adapter 包 | 职责 | 关键文件 |
|----------|------|----------|
| `adapter-repository-postgres` | Table 元数据持久化（系统表操作 | `PostgresTableRepository.ts` |
| `adapter-table-repository-postgres` | 业务表记录持久化 + DDL | `PostgresTableRecordRepository.ts`, `PostgresTableSchemaRepository.ts` |
| `adapter-db-postgres-pg` | Postgres 驱动（node-postgres） | `createDb.ts`, `unitOfWork.ts` |
| `adapter-db-postgres-pglite` | PGlite 驱动（浏览器端） | `createDb.ts` |
| `adapter-db-postgres-postgresjs` | Postgres.js 驱动 | `createDb.ts` |
| `adapter-db-postgres-shared` | 共享的 PG 适配逻辑 | `unitOfWork.ts` |

### 3.3 Adapter 实现示例

#### 3.3.1 元数据 Repository 适配器

```typescript
// adapter-repository-postgres/src/repositories/PostgresTableRepository.ts:96-103
@injectable()
export class PostgresTableRepository implements core.ITableRepository {
  constructor(
    @inject(v2PostgresStateTokens.db)
    private readonly db: Kysely<V1TeableDatabase>,  // 注入具体数据库连接
    @inject(v2PostgresStateTokens.tableMapper)
    private readonly tableMapper: core.ITableMapper
  ) {}
```

**实现特点**：
- 实现 `ITableRepository` 接口
- 依赖注入 Kysely 查询构建器
- 负责 `table_meta`, `field`, `view` 等系统表的 CRUD

#### 3.3.2 记录 Repository 适配器

```typescript
// adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts
@injectable()
export class PostgresTableRecordRepository implements core.ITableRecordRepository {
  // 实现记录的 INSERT/UPDATE/DELETE
  // 包含复杂的 SQL 构建、计算字段更新、链接关系维护
}
```

#### 3.3.3 Schema Repository 适配器

```typescript
// adapter-table-repository-postgres/src/schema/repositories/PostgresTableSchemaRepository.ts
@injectable()
export class PostgresTableSchemaRepository implements core.ITableSchemaRepository {
  // 实现物理表的 CREATE/ALTER/DROP 等 DDL 操作
  // 处理字段类型转换、索引管理、undo capture 等
}
```

### 3.4 跨数据库适配点

#### 3.4.1 数据库驱动适配层

通过 `adapter-db-postgres-*` 系列包实现不同驱动的适配：

```typescript
// adapter-db-postgres-shared/src/unitOfWork.ts:64-81
export const getPostgresTransaction = <DB>(
  context?: IExecutionContext,
  scope: UnitOfWorkScope = 'data'
): Transaction<DB> | null => {
  const transaction = getUnitOfWorkTransaction(context, scope);
  if (transaction instanceof PostgresUnitOfWorkTransaction) {
    return transaction.db as Transaction<DB>;
  }
  return null;
};

export const resolvePostgresDbOrTx = <DB>(
  db: Kysely<DB>,
  context?: IExecutionContext,
  scope: UnitOfWorkScope = 'data'
): Kysely<DB> | Transaction<DB> => {
  return getPostgresTransaction<DB>(context, scope) ?? db;
};
```

**适配点**：
- 不同驱动（pg, pglite, postgresjs）都返回 Kysely 接口
- UnitOfWork 实现统一管理事务
- `scope` 概念支持 meta/data 双数据库场景

#### 3.4.2 Specification → SQL 的 Visitor 转换

Adapter 层通过 Visitor 模式将领域规约转换为 SQL：

```typescript
// adapter-repository-postgres/src/repositories/PostgresTableRepository.ts:579-590
async findOne(
  context: core.IExecutionContext,
  spec: core.ISpecification<core.Table, core.ITableSpecVisitor>,
  options?: Pick<core.TableFindOptions, 'state'>
): Promise<Result<core.Table, DomainError>> {
  const visitor = new TableWhereVisitor(options?.state);
  const acceptResult = spec.accept(visitor);  // 规约接受访问者
  if (acceptResult.isErr()) return err(acceptResult.error);
  
  const whereResult = visitor.where();  // 访问者生成 SQL
```

**核心适配点：

1. **Where 条件生成 Visitor
   - `TableWhereVisitor` - 表元数据查询条件生成
   - `TableRecordConditionWhereVisitor` - 记录查询条件生成
   - `TableMetaUpdateVisitor` - 元数据更新 SQL 生成

2. **字段值 SQL 生成
   - `FieldInsertValueVisitor` - 字段值转 SQL 字面量
   - `FieldSqlLiteralVisitor` - 字段类型转换为 SQL 表达式

3. **Schema 变更 Visitor
   - `TableSchemaUpdateVisitor` - Schema 变更 SQL 生成
   - `PostgresTableSchemaFieldCreateVisitor` - 字段创建 DDL 生成

#### 3.4.3 PostgreSQL 版本适配

```typescript
// adapter-table-repository-postgres/src/di/register.ts:102-112
export async function createTypeValidationStrategy<DB>(
  db: Kysely<DB>
): Promise<IPgTypeValidationStrategy> {
  const hasPgInputValid = await hasPgInputIsValid(db);
  if (hasPgInputValid) {
    return new Pg16TypeValidationStrategy();  // PG 16+ 原生支持
  }
  return new PgLegacyTypeValidationStrategy();  // PG < 16 使用 polyfill
}
```

## 四、依赖注入与注册

### 4.1 DI Token 定义

```typescript
// core/src/ports/tokens.ts:1-68
export const v2CoreTokens = {
  baseRepository: Symbol('v2.core.baseRepository'),
  tableRepository: Symbol('v2.core.tableRepository'),
  tableRecordRepository: Symbol('v2.core.tableRecordRepository'),
  tableSchemaRepository: Symbol('v2.core.tableSchemaRepository'),
  // ... 更多 token
};
```

### 4.2 Adapter 注册

元数据 Repository 注册：

```typescript
// adapter-repository-postgres/src/di/register.ts:45-53
c.register(v2CoreTokens.tableRepository, PostgresTableRepository, {
  lifecycle: Lifecycle.Singleton,
});
c.register(v2CoreTokens.baseRepository, PostgresBaseRepository, {
  lifecycle: Lifecycle.Singleton,
});
```

表记录 Repository 注册：

```typescript
// adapter-table-repository-postgres/src/di/register.ts:273-283
c.register(v2CoreTokens.tableRecordQueryRepository, PostgresTableRecordQueryRepository, {
  lifecycle: Lifecycle.Singleton,
});
c.register(v2CoreTokens.tableRecordRepository, PostgresTableRecordRepository, {
  lifecycle: Lifecycle.Singleton,
});
c.register(v2CoreTokens.tableSchemaRepository, PostgresTableSchemaRepository, {
  lifecycle: Lifecycle.Singleton,
});
```

## 五、两层协作模式

### 5.1 完整调用链路

```
CreateTableHandler (command handler)
         │
         ▼
  1. 构建 Table 领域对象
         │
         ▼
  2. 调用 ITableRepository.insert(table)
         │ （通过 DI 注入的 PostgresTableRepository
         ▼
  3. PostgresTableRepository.insert()
         │
         ├─► TableMapper.toDTO(table) 
         │    领域对象 → 转换为持久化 DTO
         │
         ├─► TableFieldPersistenceBuilder
         │    构建字段元数据
         │
         └─► 执行 SQL INSERT 系统表 (table_meta, field, view)
         │
         ▼
  4. 调用 ITableSchemaRepository.insert(table)
         │
         └─► 执行 CREATE TABLE DDL
         │
         ▼
  5. 返回 Result<Table, DomainError>
```

### 5.2 事务管理协作

```typescript
// 应用层通过 UnitOfWork 管理跨 Repository 事务
const result = await unitOfWork.withTransaction(context, async (txContext) => {
  // 同一个 txContext 传递给多个 Repository 调用
  await tableRepository.insert(txContext, table);
  await tableSchemaRepository.insert(txContext, table);
  await tableRecordRepository.insertMany(txContext, table, records);
  return ok(undefined);
});
```

### 5.3 双 Scope 事务

系统支持 meta/data 分离的双数据库架构：

```typescript
// adapter-db-postgres-shared/src/unitOfWork.ts:96-101
private usesSinglePhysicalDatabase(): boolean {
  return (
    this.metaDb === this.dataDb ||
    this.metaConfig.pg.connectionString === this.dataConfig.pg.connectionString
  );
}
```

**Scope 分类**：
- `'meta'` - 元数据操作（系统表）
- `'data'` - 业务数据操作（用户表）

当 meta 和 data 是同一物理数据库时，事务自动复用连接。

## 六、职责对比总结

| 维度 | Repository 层（端口） | Adapter 层（适配器） |
|------|-------------------|------------------|
| **位置** | `core/src/ports/` | `adapter-*/` |
| **职责** | 定义"做什么"的契约 | 实现"怎么做"的细节 |
| **依赖方向** | 被依赖，不依赖外部 | 依赖 Repository 接口和具体数据库技术 |
| **技术绑定** | 无，纯 TypeScript 接口 | 绑定 Kysely、PostgreSQL 驱动 |
| **可替换性** | 稳定，不易变 | 可替换为不同数据库实现 |
| **测试友好** | 提供内存实现 | 需要测试 |

## 八、三条核心调用链深入分析

### 8.1 表元数据仓储调用链（ITableRepository）

**场景**：查询或更新表元数据（table_meta, field, view 等系统表）

#### 8.1.1 查询调用链：findOne

```
应用层（Command Handler）
         │
         ▼  构建 Specification
    TableByIdSpec(tableId)
         │
         ▼  调用 repository
  ITableRepository.findOne(context, spec)
         │
         ▼  ┌────────────────────────────────────────────┐
            │  Adapter 层：PostgresTableRepository       │
            │                                            │
            │  1. 创建 TableWhereVisitor(state)         │
            │  2. spec.accept(visitor)                  │
            │     → visitor.visitTableById(spec)        │
            │     → 生成 where 条件：id = tableId       │
            │  3. visitor.where() → 获取 where 函数      │
            │  4. resolvePostgresDbOrTx(db, ctx, 'meta')│
            │  5. 构建 Kysely SELECT 查询                │
            │     - LATERAL JOIN 聚合 fields/views       │
            │     - 应用 where 条件                       │
            │  6. executeTakeFirst() → 执行 SQL         │
            │  7. mapTableRow() → DTO → 领域对象         │
            └────────────────────────────────────────────┘
         │
         ▼  返回 Result<Table, DomainError>
```

**关键代码细节**：

```typescript
// adapter-repository-postgres/src/repositories/PostgresTableRepository.ts:579-617
async findOne(
  context: core.IExecutionContext,
  spec: core.ISpecification<core.Table, core.ITableSpecVisitor>,
  options?: Pick<core.TableFindOptions, 'state'>
): Promise<Result<core.Table, DomainError>> {
  // Step 1: 创建 Visitor
  const visitor = new TableWhereVisitor(options?.state);
  
  // Step 2: Specification 接受 Visitor（双分派）
  const acceptResult = spec.accept(visitor);
  if (acceptResult.isErr()) return err(acceptResult.error);

  // Step 3: Visitor 生成 SQL Where 条件
  const whereResult = visitor.where();
  if (whereResult.isErr()) return err(whereResult.error);
  const whereFactory = whereResult.value;

  // Step 4: 解析事务上下文（支持 meta scope）
  const db = resolvePostgresDbOrTx(this.db, context, 'meta');

  // Step 5: 构建复杂查询（LATERAL JOIN 聚合 fields/views）
  const baseQuery = db
    .selectFrom('table_meta')
    .leftJoinLateral(fieldsLateral, (join) => join.onTrue())
    .leftJoinLateral(viewsLateral, (join) => join.onTrue())
    .select(['table_meta.id', 'table_meta.name', 'fields.fields', 'views.views'])
    .where((eb) => whereFactory(eb));  // 应用 Visitor 生成的条件

  // Step 6: 执行并映射结果
  const tableRow = await baseQuery.executeTakeFirst();
  const tableResult = this.mapTableRow(tableRow);
}
```

**TableWhereVisitor 核心实现**：

```typescript
// adapter-repository-postgres/src/repositories/visitors/TableWhereVisitor.ts:176-186
visitTableById(spec: TableByIdSpec): Result<ITableMetaWhere, DomainError> {
  const cond: ITableMetaWhere = (eb) => eb.eb('id', '=', spec.tableId().toString());
  this.mergeSpecInfo({ specName: 'TableByIdSpec', tableId: spec.tableId().toString() });
  return this.addCond(cond).map(() => cond);
}

// 继承自 AbstractSpecFilterVisitor，支持 AND/OR/NOT 组合
and(left: ITableMetaWhere, right: ITableMetaWhere): ITableMetaWhere {
  return (eb) => eb.and([left(eb), right(eb)]);
}
```

#### 8.1.2 更新调用链：updateOne

```
应用层（Command Handler）
         │
         ▼  构建变更 Specification
    TableAddFieldSpec(field)
         │
         ▼  调用 repository
  ITableRepository.updateOne(context, table, mutateSpec)
         │
         ▼  ┌────────────────────────────────────────────┐
            │  Adapter 层：PostgresTableRepository       │
            │                                            │
            │  1. 构建基础 where 条件（id, base_id）     │
            │  2. 创建 TableMetaUpdateVisitor            │
            │  3. mutateSpec.accept(visitor)             │
            │     → visitor.visitTableAddField(spec)     │
            │     → 生成 INSERT INTO field 语句          │
            │     → 支持 ON CONFLICT 软恢复              │
            │  4. visitor.where() → 获取 SQL 语句数组    │
            │  5. executeCompiledQueries() → 批量执行     │
            │  6. 加载更新后的 field/view 版本号         │
            │  7. 返回 TableUpdatePersistResult         │
            └────────────────────────────────────────────┘
         │
         ▼  返回版本变更结果
```

**TableMetaUpdateVisitor 核心实现**：

```typescript
// adapter-repository-postgres/src/repositories/visitors/TableMetaUpdateVisitor.ts:144-155
visitTableAddField(
  spec: TableAddFieldSpec
): Result<ReadonlyArray<TableUpdateBuilder>, DomainError> {
  const fieldRowResult = this.fieldRowBuilder.buildRowForField(spec.field());
  if (fieldRowResult.isErr()) return err(fieldRowResult.error);

  const statements: ReadonlyArray<TableUpdateBuilder> = [
    this.buildInsertOrReviveFieldStatement(fieldRowResult.value),
  ];

  return this.addCond(statements).map(() => statements);
}

// 软删除恢复逻辑：先 INSERT，冲突时 UPDATE 并清除 deleted_time
private buildInsertOrReviveFieldStatement(fieldRow: TableFieldRow): TableUpdateBuilder {
  return this.params.db
    .insertInto('field')
    .values(fieldRow)
    .onConflict((oc) =>
      oc.column('id').doUpdateSet({
        name: fieldRow.name,
        deleted_time: null,  // 清除删除标记
        version: sql<number>`coalesce(field.version, 0) + 1`,
      })
    );
}
```

### 8.2 记录仓储调用链（ITableRecordRepository + ITableRecordQueryRepository）

**场景**：业务表记录的 CRUD 操作

#### 8.2.1 查询调用链：find

```
应用层（Query Handler）
         │
         ▼  构建记录查询 Specification
    RecordConditionSpecBuilder
      .where(fieldId, 'equals', value)
      .build()
         │
         ▼  调用 repository
  ITableRecordQueryRepository.find(context, table, spec, options)
         │
         ▼  ┌────────────────────────────────────────────┐
            │  Adapter 层：PostgresTableRecordQueryRepo  │
            │                                            │
            │  1. queryBuilderManager.createBuilder()   │
            │  2. queryBuilder.select(projectionFields) │
            │  3. 如有 spec：                            │
            │     → 创建 TableRecordConditionWhereVisitor│
            │     → spec.accept(visitor)                 │
            │     → 生成记录查询 WHERE 条件              │
            │  4. 处理 orderBy、pagination               │
            │  5. queryBuilder.build() → 生成完整 SQL    │
            │  6. execute() → 执行查询                   │
            │  7. 映射结果为 TableRecordReadModel        │
            └────────────────────────────────────────────┘
         │
         ▼  返回查询结果
```

**TableRecordConditionWhereVisitor 核心实现**：

```typescript
// adapter-table-repository-postgres/src/record/visitors/TableRecordConditionWhereVisitor.ts
// 处理各种字段类型的查询条件
visitSingleLineTextCondition(spec: SingleLineTextConditionSpec): Result<RecordConditionWhere, DomainError> {
  const { fieldId, operator, value } = spec;
  const dbFieldName = resolveDbFieldName(fieldId);
  
  return match(operator)
    .with('equals', () => sql`${sql.ref(dbFieldName)} = ${value}`)
    .with('contains', () => sql`${sql.ref(dbFieldName)} LIKE ${`%${value}%`}`)
    .with('startsWith', () => sql`${sql.ref(dbFieldName)} LIKE ${`${value}%`}`)
    .otherwise(() => err(unsupportedOperator));
}

// User/Link 字段特殊处理：JSONB 路径查询
visitUserCondition(spec: UserConditionSpec): Result<RecordConditionWhere, DomainError> {
  const columnRef = sql`${sql.ref(dbFieldName)}`;
  const idArray = buildUserLinkIdArray(columnRef, isMultiple);
  return sql`${idArray} @> ARRAY[${userId}]::text[]`;
}
```

#### 8.2.2 插入调用链：insert

```
应用层（Command Handler）
         │
         ▼  构建 TableRecord
    TableRecord.create(fields)
         │
         ▼  调用 repository
  ITableRecordRepository.insert(context, table, record, options)
         │
         ▼  ┌────────────────────────────────────────────┐
            │  Adapter 层：PostgresTableRecordRepository │
            │                                            │
            │  1. RecordInsertBuilder.create()           │
            │  2. 遍历所有字段：                          │
            │     → FieldInsertValueVisitor              │
            │     → field.accept(visitor)                │
            │     → 生成 columnValues + queryExecutors   │
            │  3. 处理计算字段、链接字段特殊逻辑          │
            │  4. 构建主 INSERT SQL 语句                 │
            │  5. 执行主插入                              │
            │  6. 执行额外 queryExecutors（链接表等）    │
            │  7. 触发计算字段更新（ComputedUpdater）    │
            │  8. 返回 RecordMutationResult              │
            └────────────────────────────────────────────┘
         │
         ▼  返回插入结果
```

**FieldInsertValueVisitor 核心实现**：

```typescript
// adapter-table-repository-postgres/src/record/visitors/FieldInsertValueVisitor.ts:76-100
export class FieldInsertValueVisitor implements IFieldVisitor<FieldInsertResult> {
  private simpleValueFrom(value: unknown): Result<FieldInsertResult, DomainError> {
    return ok({
      columnValues: { [this.ctx.dbFieldName]: value ?? null },
      queryExecutors: [],
    });
  }

  private jsonValueFrom(value: unknown): Result<FieldInsertResult, DomainError> {
    // JSONB 列必须使用 JSON.stringify（pg 驱动要求）
    const serialized = value === null || value === undefined ? null : JSON.stringify(value);
    return ok({
      columnValues: { [this.ctx.dbFieldName]: serialized },
      queryExecutors: [],
    });
  }

  visitLinkField(field: LinkField): Result<FieldInsertResult, DomainError> {
    // 链接字段特殊处理：生成关联表插入、FK 更新等额外语句
    const linkStatements = buildLinkInsertStatements(field, this.rawValue);
    return ok({
      columnValues: { [this.ctx.dbFieldName]: serializedValue },
      queryExecutors: linkStatements.executors,
    });
  }
}
```

### 8.3 结构仓储调用链（ITableSchemaRepository）

**场景**：物理表结构的 DDL 操作（CREATE TABLE, ALTER TABLE 等）

#### 8.3.1 创建表调用链：insert

```
应用层（CreateTableCommand Handler）
         │
         ▼  已创建 Table 领域对象（含字段）
         │
         ▼  调用 repository
  ITableSchemaRepository.insert(context, table)
         │
         ▼  ┌────────────────────────────────────────────┐
            │  Adapter 层：PostgresTableSchemaRepository │
            │                                            │
            │  1. PostgresTableSchemaFieldCreateVisitor  │
            │    .forTableCreation(builderRef)           │
            │  2. 遍历所有字段：                          │
            │     → field.accept(visitor)                │
            │     → 调用字段 Schema Rules                │
            │     → 生成 ADD COLUMN 定义                  │
            │  3. 构建 CREATE TABLE 语句                 │
            │  4. 创建搜索索引（GIN trigram）             │
            │  5. 创建系统列（__id, __version, 等）       │
            │  6. 执行 DDL 语句                          │
            │  7. 初始化 undo capture 基础设施           │
            │  8. 填充链接字段标题等初始数据              │
            └────────────────────────────────────────────┘
         │
         ▼  返回 Result<void, DomainError>
```

#### 8.3.2 字段变更调用链：updateOne

```
应用层（UpdateFieldCommand Handler）
         │
         ▼  构建变更 Specification
    TableUpdateFieldTypeSpec(oldField, newField)
         │
         ▼  调用 repository
  ITableSchemaRepository.updateOne(context, table, mutateSpec)
         │
         ▼  ┌────────────────────────────────────────────┐
            │  Adapter 层：PostgresTableSchemaRepository │
            │                                            │
            │  1. 创建 TableSchemaUpdateVisitor          │
            │  2. mutateSpec.accept(visitor)             │
            │     → visitor.visitTableUpdateFieldType()  │
            │     → 检测依赖变化（DependencyChangeDetec- │
            │       torVisitor）                         │
            │     → 收集值变更（FieldValueChangeCollec-  │
            │       torVisitor）                         │
            │     → 字段类型转换（FieldTypeConversion    │
            │       Visitor）                            │
            │     → 生成 ALTER TABLE ALTER COLUMN 语句   │
            │     → 处理索引重建                          │
            │  3. visitor.where() → 获取 DDL 语句数组    │
            │  4. 执行 DDL 语句（带 undo capture）       │
            │  5. 回填计算字段值                          │
            │  6. 级联更新依赖字段                        │
            └────────────────────────────────────────────┘
         │
         ▼  返回变更结果
```

**TableSchemaUpdateVisitor 核心实现**：

```typescript
// adapter-table-repository-postgres/src/schema/visitors/TableSchemaUpdateVisitor.ts:104-110
export class TableSchemaUpdateVisitor
  extends AbstractSpecFilterVisitor<ReadonlyArray<TableSchemaStatementBuilder>>
  implements ITableSpecVisitor<ReadonlyArray<TableSchemaStatementBuilder>>
{
  // 处理字段类型转换
  visitTableUpdateFieldType(
    spec: TableUpdateFieldTypeSpec
  ): Result<ReadonlyArray<TableSchemaStatementBuilder>, DomainError> {
    // 1. 检测依赖变化
    const depDetector = new DependencyChangeDetectorVisitor();
    spec.accept(depDetector);
    
    // 2. 收集值变更
    const valueCollector = new FieldValueChangeCollectorVisitor();
    spec.accept(valueCollector);
    
    // 3. 生成类型转换 SQL
    const conversionParams = buildConversionParams(spec.oldField(), spec.newField());
    const statements = generateFieldConversionStatements(conversionParams);
    
    // 4. 处理索引重建（删除旧索引，创建新索引）
    const indexStatements = rebuildSearchIndexIfNeeded(spec);
    
    return ok([...statements, ...indexStatements]);
  }
}

// adapter-table-repository-postgres/src/schema/visitors/PostgresTableSchemaFieldCreateVisitor.ts
// 字段创建 DDL 生成
visitNumberField(field: NumberField): Result<ReadonlyArray<TableSchemaStatementBuilder>, DomainError> {
  const dbFieldName = field.dbFieldName().value;
  const columnDef = this.builderRef.builder
    .addColumn(dbFieldName, 'numeric', (col) => {
      if (field.notNull().toBoolean()) col = col.notNull();
      if (field.unique().toBoolean()) col = col.unique();
      return col;
    });
  
  return ok([{ compile: () => columnDef.compile() }]);
}
```

### 8.4 三条调用链对比

| 维度 | 元数据仓储 | 记录仓储 | 结构仓储 |
|------|----------|--------|--------|
| **Scope** | `'meta'` | `'data'` | `'data'` |
| **操作类型** | 系统表 DML | 业务表 DML | 业务表 DDL |
| **核心 Visitor** | `TableWhereVisitor` <br> `TableMetaUpdateVisitor` | `TableRecordConditionWhereVisitor` <br> `FieldInsertValueVisitor` <br> `CellValueMutateVisitor` | `TableSchemaUpdateVisitor` <br> `PostgresTableSchemaFieldCreateVisitor` <br> `FieldTypeConversionVisitor` |
| **事务范围** | 与其他元数据操作共享 | 支持跨记录批量 | 支持 undo capture |
| **版本管理** | field/view version 递增 | record __version 乐观锁 | schema operation 历史记录 |
| **错误处理** | 数据库错误包装为 DomainError | 死锁自动重试（最多3次） | 类型验证失败提前拦截 |

## 九、三种 PostgreSQL 驱动的注册与事务差异

### 9.1 驱动包概览

| 驱动包 | 底层库 | 适用场景 | 连接方式 |
|-------|--------|--------|--------|
| `adapter-db-postgres-pg` | `pg` (node-postgres) | 服务端生产环境 | TCP 连接池 |
| `adapter-db-postgres-pglite` | `@electric-sql/pglite` | 浏览器端、本地开发 | 内存/本地文件 |
| `adapter-db-postgres-postgresjs` | `postgres` (Postgres.js) | 高性能场景 | TCP 连接 |

### 9.2 驱动创建差异

#### 9.2.1 pg 驱动（node-postgres）

```typescript
// adapter-db-postgres-pg/src/createDb.ts:22-44
const createPgDb = async <DB>(config: IV2PostgresDbConfig): Promise<Kysely<DB>> => {
  const connectionString = config.pg.connectionString;
  
  // 特殊处理：确保 OpenTelemetry 能正确注入
  // 使用 __non_webpack_require__ 绕过 webpack 打包，确保 require-in-the-middle 能工作
  const pg = await loadPg();
  const Pool = pg.Pool ?? (hasPgDefault(pg) ? pg.default.Pool : undefined);
  
  const poolOptions = resolvePoolOptions(config);
  const pool = new Pool(poolOptions);
  pool.on('error', handlePgPoolError);

  return new Kysely<DB>({
    dialect: new PostgresDialect({ pool }),
  });
};
```

**特点**：
- 支持连接池配置（max, idleTimeoutMillis, connectionTimeoutMillis）
- 支持 DSN 中的 `connection_limit` 参数
- 自动忽略 admin shutdown 错误（57P01, 57P02）
- Webpack 环境特殊处理，确保 OTel 链路追踪正常工作

#### 9.2.2 pglite 驱动

```typescript
// adapter-db-postgres-pglite/src/createDb.ts:5-14
const createPostgresPgliteDb = async <DB>(config: IV2PostgresDbConfig): Promise<Kysely<DB>> => {
  // connectionString 被解释为数据目录，"memory://" 表示内存数据库
  const dataDir = config.pg.connectionString ?? 'memory://';

  const { dialect } = await KyselyPGlite.create(dataDir);

  return new Kysely<DB>({ dialect });
};
```

**特点**：
- `connectionString` 用作数据目录路径
- 支持 `memory://` 内存数据库
- 无需 TCP 连接，浏览器端可用
- 启动速度快，适合测试

#### 9.2.3 postgresjs 驱动

```typescript
// adapter-db-postgres-postgresjs/src/createDb.ts:6-21
const createPostgresJsDb = <DB>(config: IV2PostgresDbConfig): Kysely<DB> => {
  const connectionString = config.pg.connectionString;
  
  const sql = postgres(connectionString, {
    onnotice: () => undefined,  // 静默处理 NOTICE 消息
  });

  return new Kysely<DB>({
    dialect: new PostgresJSDialect({ postgres: sql }),
  });
};
```

**特点**：
- 高性能、轻量级驱动
- 支持流式查询
- 类型安全更好
- 无连接池概念（内部管理）

### 9.3 DI 注册差异

#### 9.3.1 pg 驱动注册

```typescript
// adapter-db-postgres-pg/src/di/register.ts:9-57
const registerDb = async (
  c: DependencyContainer,
  rawConfig: Partial<IV2PostgresDbConfig>,
  target: 'all' | 'meta' | 'data'  // 支持分别注册 meta 和 data 数据库
): Promise<DependencyContainer> => {
  const config = v2PostgresDbConfigSchema.parse(rawConfig);
  const db = await createV2PostgresDb(config);

  if (target === 'all' || target === 'meta') {
    c.registerInstance(v2MetaDbTokens.db, db);
    c.registerInstance(v2MetaDbTokens.config, config);
  }
  if (target === 'all' || target === 'data') {
    c.registerInstance(v2DataDbTokens.db, db);
    c.registerInstance(v2DataDbTokens.config, config);
  }
  // ...
};

// 支持三种注册方式
export const registerV2PostgresDb = async (...) => registerDb(c, rawConfig, 'all');
export const registerV2PostgresMetaDb = async (...) => registerDb(c, rawConfig, 'meta');
export const registerV2PostgresDataDb = async (...) => registerDb(c, rawConfig, 'data');
```

**特点**：
- 支持 meta/data 分离注册
- 支持双数据库架构（不同的 connectionString）
- 提供细粒度的注册函数

#### 9.3.2 pglite 和 postgresjs 驱动注册

```typescript
// adapter-db-postgres-pglite/src/di/register.ts:13-33
export const registerV2PostgresPgliteDb = async (
  c: DependencyContainer = container,
  rawConfig: Partial<IV2PostgresDbConfig> = {}
): Promise<DependencyContainer> => {
  const config = v2PostgresDbConfigSchema.parse(rawConfig);
  const db = await createV2PostgresPgliteDb(config);

  // pglite 总是注册到所有 token（meta 和 data 使用同一实例）
  c.registerInstance(v2MetaDbTokens.db, db);
  c.registerInstance(v2MetaDbTokens.config, config);
  c.registerInstance(v2DataDbTokens.db, db);
  c.registerInstance(v2DataDbTokens.config, config);
  c.registerInstance(v2PostgresDbTokens.db, db);
  c.registerInstance(v2PostgresDbTokens.config, config);

  return c;
};
```

**特点**：
- pglite 和 postgresjs 总是使用单一数据库实例
- meta 和 data scope 共享同一连接
- 不支持双数据库架构

### 9.4 事务上下文切换差异

#### 9.4.1 共享的 UnitOfWork 实现

三种驱动共享相同的 UnitOfWork 实现（来自 `adapter-db-postgres-shared`）：

```typescript
// adapter-db-postgres-shared/src/unitOfWork.ts:64-81
export const getPostgresTransaction = <DB>(
  context?: IExecutionContext,
  scope: UnitOfWorkScope = 'data'
): Transaction<DB> | null => {
  const transaction = getUnitOfWorkTransaction(context, scope);
  if (transaction instanceof PostgresUnitOfWorkTransaction) {
    return transaction.db as Transaction<DB>;
  }
  return null;
};

export const resolvePostgresDbOrTx = <DB>(
  db: Kysely<DB>,
  context?: IExecutionContext,
  scope: UnitOfWorkScope = 'data'
): Kysely<DB> | Transaction<DB> => {
  return getPostgresTransaction<DB>(context, scope) ?? db;
};
```

#### 9.4.2 单数据库 vs 双数据库事务

| 场景 | pg 驱动（双数据库） | pg 驱动（单数据库） | pglite/postgresjs |
|------|-------------------|-------------------|-----------------|
| meta scope 事务 | 使用 metaDb 连接 | 复用同一连接 | 复用同一连接 |
| data scope 事务 | 使用 dataDb 连接 | 复用同一连接 | 复用同一连接 |
| 跨 scope 事务 | 检测到同一连接字符串时自动复用 | 天然复用 | 天然复用 |
| 分布式事务 | 不支持（需要分别管理） | 单事务 | 单事务 |

**事务复用逻辑**：

```typescript
// adapter-db-postgres-shared/src/unitOfWork.ts:96-126
private usesSinglePhysicalDatabase(): boolean {
  return (
    this.metaDb === this.dataDb ||
    this.metaConfig.pg.connectionString === this.dataConfig.pg.connectionString
  );
}

private reuseSiblingScopeTransaction(
  context: IExecutionContext,
  scope: UnitOfWorkScope
): IExecutionContext | null {
  if (!this.usesSinglePhysicalDatabase()) {
    return null;  // 双数据库时不复用
  }

  // 单数据库时，meta 和 data scope 共享同一事务
  const siblingScope: UnitOfWorkScope = scope === 'meta' ? 'data' : 'meta';
  const siblingTransaction = getUnitOfWorkTransaction(context, siblingScope);
  if (siblingTransaction) {
    return {
      ...context,
      transactions: {
        ...context.transactions,
        [scope]: siblingTransaction,  // 指向同一事务对象
      },
    };
  }
  return null;
}
```

#### 9.4.3 事务执行流程

```typescript
// adapter-db-postgres-shared/src/unitOfWork.ts:128-191
async withTransaction<T>(
  context: IExecutionContext,
  work: UnitOfWorkOperation<T>,
  options?: IUnitOfWorkOptions
): Promise<Result<T, DomainError>> {
  const scope = options?.scope ?? 'data';
  const existingTransaction = getUnitOfWorkTransaction(context, scope);

  // 1. 已有事务：直接复用（嵌套事务支持）
  if (existingTransaction) {
    if (existingTransaction instanceof PostgresUnitOfWorkTransaction) {
      return work(activateUnitOfWorkScope(context, scope));
    }
    return err(domainError.validation({ message: 'Unsupported transaction context' }));
  }

  // 2. 检查是否可以复用兄弟 scope 的事务
  const sharedTransactionContext = this.reuseSiblingScopeTransaction(context, scope);
  if (sharedTransactionContext) {
    return work(sharedTransactionContext);
  }

  // 3. 新建事务（最多重试 3 次死锁）
  const db = scope === 'meta' ? this.metaDb : this.dataDb;
  while (true) {
    try {
      const transactionResult = await db.transaction().execute(async (trx) => {
        const transaction = new PostgresUnitOfWorkTransaction(trx, scope);
        const transactionContext = bindUnitOfWorkTransaction(context, transaction);

        const workResult = await work(transactionContext);
        if (workResult.isErr()) {
          throw new UnitOfWorkAbort(workResult.error);  // 抛出自定义异常触发回滚
        }

        return { workResult, transaction };
      });
      await transactionResult.transaction.runAfterCommitHandlers();
      return transactionResult.workResult;
    } catch (error) {
      if (error instanceof UnitOfWorkAbort) {
        // 死锁/序列化失败自动重试（指数退避 + 抖动）
        if (attempt < maxRetries && isRetryableTransactionAbort(error.error)) {
          const delayMs = backoffMs(attempt);  // 5ms, 10ms, 20ms + 随机抖动
          attempt += 1;
          await sleep(delayMs);
          continue;
        }
        await transaction?.runAfterRollbackHandlers();
        return err(error.error);
      }
    }
  }
}
```

### 9.5 驱动功能对比表

| 功能 | pg (node-postgres) | pglite | postgresjs |
|-----|-------------------|--------|-----------|
| **连接池** | ✅ 完全支持 | ❌ 单连接 | ❌ 内部管理 |
| **OTel 链路追踪** | ✅ 特殊处理支持 | ❌ | ❌ |
| **浏览器端运行** | ❌ | ✅ | ❌ |
| **内存数据库** | ❌ | ✅ | ❌ |
| **双数据库架构** | ✅ | ❌（总是单实例） | ❌（总是单实例） |
| **死锁自动重试** | ✅（共享实现） | ✅（共享实现） | ✅（共享实现） |
| **事务 scope 复用** | ✅（同连接字符串时） | ✅（天然） | ✅（天然） |
| **afterCommit 钩子** | ✅ | ✅ | ✅ |
| **undo capture** | ✅ | ✅ | ✅ |
| **连接错误处理** | ✅ 完整 | ✅ 简化 | ✅ 简化 |
| **生产环境推荐** | ✅ 首选 | ❌ 仅开发/测试 | ⚠️ 性能敏感场景 |

## 十、关键设计模式

1. **依赖倒置原则（DIP）**：高层模块依赖抽象，不依赖具体
2. **端口与适配器模式**：清晰分离内部与外部
3. **规约模式（Specification）**：查询和更新逻辑的领域化描述
4. **访问者模式（Visitor）**：Specification → SQL 的双分派转换
5. **工作单元模式（Unit of Work）**：事务管理与跨 Repository 协作
6. **依赖注入（DI）**：运行时绑定实现，支持驱动热插拔
7. **组合模式**：Specification 的 AND/OR/NOT 组合
8. **模板方法模式**：AbstractSpecFilterVisitor 提供组合逻辑骨架

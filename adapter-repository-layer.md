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

### Specification → Visitor → SQL 通用转换模式（可复核）

所有三条仓储调用链都遵循相同的双分派转换模式：

```
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 1: 领域层构建 Specification（纯领域逻辑，无 SQL）               │
├─────────────────────────────────────────────────────────────────────┤
│  应用层 / Command Handler                                            │
│    → 调用 XxxSpec.create(...) 构建规约对象                            │
│      例：TableByIdSpec.create(tableId)                                │
│      例：SingleLineTextConditionSpec.create(field, 'equals', value)   │
│                                                                     │
│  Specification 内部实现：                                             │
│    interface ISpecification<T, TVisitor> {                           │
│      accept(visitor: TVisitor): Result<TResult, DomainError>;       │
│    }                                                                 │
│                                                                     │
│    class TableByIdSpec implements ISpecification<Table, ITableSpecVisitor> {│
│      accept(visitor: ITableSpecVisitor<TResult>): Result<TResult, DomainError> {│
│        return visitor.visitTableById(this);  // 【双分派关键】       │
│      }                                                               │
│    }                                                                 │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 2: Adapter 层创建 Visitor（数据库特定实现）                     │
├─────────────────────────────────────────────────────────────────────┤
│  PostgresXxxRepository 方法内部：                                     │
│    → 创建具体 Visitor 实例                                            │
│      例：const visitor = new TableWhereVisitor(state);              │
│      例：const visitor = new TableRecordConditionWhereVisitor();     │
│                                                                     │
│  Visitor 接口定义（每个领域有独立的 Visitor 接口）：                  │
│    interface ITableSpecVisitor<TResult> {                           │
│      visitTableById(spec: TableByIdSpec): Result<TResult, DomainError>;│
│      visitTableByBaseId(spec: TableByBaseIdSpec): Result<TResult, DomainError>;│
│      visitTableByName(spec: TableByNameSpec): Result<TResult, DomainError>;│
│      // ... 20+ 个方法，每个 Specification 对应一个 visit 方法       │
│    }                                                                 │
│                                                                     │
│    interface ITableRecordConditionSpecVisitor<TResult> {            │
│      visitRecordById(spec: RecordByIdSpec): ...;                    │
│      visitSingleLineTextIs(spec: SingleLineTextConditionSpec): ...; │
│      visitSingleLineTextContains(spec: ...): ...;                   │
│      // ... 130+ 个方法，每个字段类型 × 操作符组合                   │
│    }                                                                 │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 3: 双分派：spec.accept(visitor)                                │
├─────────────────────────────────────────────────────────────────────┤
│  调用 spec.accept(visitor)                                           │
│    → 进入 Specification 的 accept 方法                               │
│    → 调用 visitor.visitXxx(this) 【反向调用，双分派完成】             │
│    → 进入具体 Visitor 的 visitXxx 方法                               │
│                                                                     │
│  可复核断点：                                                         │
│    1. acceptResult = spec.accept(visitor)                           │
│    2. 检查 acceptResult 是否为 Err（规约不支持此 Visitor）            │
│    3. 检查 visitor 内部状态是否已累积 SQL 片段                       │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 4: Visitor 生成 SQL 片段 / 语句                               │
├─────────────────────────────────────────────────────────────────────┤
│  三种 Visitor 输出类型：                                              │
│                                                                     │
│  ▶ 查询类 Visitor（TableWhereVisitor, TableRecordConditionWhereVisitor）│
│    输出：(eb: ExpressionBuilder) => SqlBool                          │
│    用途：作为 Kysely .where() 参数                                   │
│                                                                     │
│  ▶ 元数据更新类 Visitor（TableMetaUpdateVisitor）                    │
│    输出：ReadonlyArray<TableUpdateBuilder>                            │
│    用途：编译后批量执行（INSERT/UPDATE field/view 系统表）           │
│                                                                     │
│  ▶ DDL 类 Visitor（TableSchemaUpdateVisitor）                        │
│    输出：ReadonlyArray<TableSchemaStatementBuilder>                   │
│    用途：编译后执行（CREATE/ALTER TABLE 等）                         │
│                                                                     │
│  组合逻辑（AbstractSpecFilterVisitor 提供）：                          │
│    and(left, right) => (eb) => eb.and([left(eb), right(eb)])        │
│    or(left, right)  => (eb) => eb.or([left(eb), right(eb)])         │
│    not(inner)       => (eb) => eb.not(inner(eb))                    │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 5: 构建完整 SQL 并执行                                         │
├─────────────────────────────────────────────────────────────────────┤
│  调用 visitor.where() 获取最终输出                                   │
│    → whereResult = visitor.where()                                   │
│    → sqlFragment = whereResult.value                                 │
│                                                                     │
│  三条仓储的执行方式：                                                 │
│                                                                     │
│  ▶ 元数据仓储（PostgresTableRepository）                            │
│    查询：db.selectFrom('table_meta').where((eb) => sqlFragment(eb)) │
│          .executeTakeFirst()                                        │
│    更新：executeCompiledQueries(db, statements, { method: 'updateOne' }) │
│          → 循环编译执行每条语句                                       │
│                                                                     │
│  ▶ 记录仓储（PostgresTableRecordRepository）                        │
│    查询：queryBuilder.select(...).where(sqlExpr).build().execute()  │
│    插入：主 INSERT + executeStatements(db, additionalStatements)    │
│          → 内联执行链接关系维护语句                                   │
│                                                                     │
│  ▶ 结构仓储（PostgresTableSchemaRepository）                        │
│    DDL：executeScopedTableSchemaStatements(context, db, statements) │
│          → 按 scope 分组，meta scope 切换到 metaDb                   │
│          → 每条语句 stmt.compile().execute()                         │
│                                                                     │
│  执行并获取结果：                                                     │
│    查询：executeTakeFirst() → 领域对象                               │
│    DML：execute() → 影响行数 / 返回值                                │
│    DDL：无返回值，失败抛出异常                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.1 表元数据仓储调用链（ITableRepository）

**场景**：查询或更新表元数据（table_meta, field, view 等系统表）

#### 8.1.1 查询调用链：findOne

```
应用层（Command Handler）
         │
         ▼  【步骤 1】构建 Specification（领域层）
    TableByIdSpec.create(tableId: TableId)
         │
         ▼  【步骤 2】调用 repository 接口
  ITableRepository.findOne(context: IExecutionContext, spec)
         │
         ▼  ┌──────────────────────────────────────────────────────────────┐
            │  Adapter 层：PostgresTableRepository                            │
            │                                                                 │
            │  【步骤 3】创建 TableWhereVisitor                               │
            │     new TableWhereVisitor(state?: 'active' | 'deleted' |      │
            │                                       'both')                   │
            │                                                                 │
            │  【步骤 4】双分派：Specification 接受 Visitor                   │
            │     spec.accept(visitor)                                       │
            │       ↓ TableByIdSpec.accept() 实现：                           │
            │       accept(visitor: ITableSpecVisitor<TResult>):             │
            │         Result<TResult, DomainError> {                         │
            │         return visitor.visitTableById(this);  // 调用对应方法   │
            │       }                                                         │
            │                                                                 │
            │  【步骤 5】Visitor.visitTableById() 生成条件                    │
            │     → cond = (eb) => eb.eb('id', '=', spec.tableId())          │
            │     → this.addCond(cond)                                       │
            │                                                                 │
            │  【步骤 6】获取组合后的 where 工厂函数                           │
            │     visitor.where() → Result<(eb: ExpressionBuilder) =>        │
            │                               SqlBool, DomainError>            │
            │                                                                 │
            │  【步骤 7】解析事务上下文（支持 'meta' scope）                   │
            │     resolvePostgresDbOrTx(this.db, context, 'meta')            │
            │       → getPostgresTransaction(context, 'meta') ?? this.db     │
            │                                                                 │
            │  【步骤 8】构建复杂 SELECT 查询                                 │
            │     db                                                          │
            │       .selectFrom('table_meta')                                 │
            │       .leftJoinLateral(fieldsLateral)  // 聚合 field 记录       │
            │       .leftJoinLateral(viewsLateral)    // 聚合 view 记录       │
            │       .select(['id', 'name', 'fields.fields', 'views.views'])  │
            │       .where((eb) => whereFactory(eb))  // 应用条件             │
            │       .$if(state === 'deleted', (qb) =>                         │
            │         qb.where(sql`deleted_time is not null`)                │
            │       )                                                         │
            │                                                                 │
            │  【步骤 9】执行 SQL                                              │
            │     baseQuery.executeTakeFirst() → tableRow                     │
            │                                                                 │
            │  【步骤 10】映射回领域对象                                       │
            │     this.tableMapper.toDomain(tableRow) → Table                │
            └──────────────────────────────────────────────────────────────┘
         │
         ▼  返回 Result<Table, DomainError>
```

**TableWhereVisitor 核心实现（可复核）**：

```typescript
// adapter-repository-postgres/src/repositories/visitors/TableWhereVisitor.ts:169-190
// 继承自 AbstractSpecFilterVisitor，支持 AND/OR/NOT 组合
export class TableWhereVisitor
  extends core.AbstractSpecFilterVisitor<ITableMetaWhere>
  implements core.ITableSpecVisitor<ITableMetaWhere>
{
  // 按 ITableSpecVisitor 接口方法逐项实现：
  visitTableById(spec: core.TableByIdSpec): Result<ITableMetaWhere, DomainError> {
    const cond: ITableMetaWhere = (eb) => eb.eb('id', '=', spec.tableId().toString());
    this.mergeSpecInfo({ specName: 'TableByIdSpec', tableId: spec.tableId().toString() });
    return this.addCond(cond).map(() => cond);
  }

  visitTableByBaseId(spec: core.TableByBaseIdSpec): Result<ITableMetaWhere, DomainError> {
    const cond: ITableMetaWhere = (eb) => eb.eb('base_id', '=', spec.baseId().toString());
    return this.addCond(cond).map(() => cond);
  }

  visitTableByName(spec: core.TableByNameSpec): Result<ITableMetaWhere, DomainError> {
    const cond: ITableMetaWhere = (eb) => eb.eb('name', '=', spec.name().toString());
    return this.addCond(cond).map(() => cond);
  }

  visitTableBySpaceId(spec: core.TableBySpaceIdSpec): Result<ITableMetaWhere, DomainError> {
    const cond: ITableMetaWhere = (eb) => eb.eb('space_id', '=', spec.spaceId().toString());
    return this.addCond(cond).map(() => cond);
  }

  // ... 20+ 个 visit 方法对应不同的 Specification

  // 父类提供的组合逻辑：
  and(left: ITableMetaWhere, right: ITableMetaWhere): ITableMetaWhere {
    return (eb) => eb.and([left(eb), right(eb)]);
  }
  or(left: ITableMetaWhere, right: ITableMetaWhere): ITableMetaWhere {
    return (eb) => eb.or([left(eb), right(eb)]);
  }
  not(inner: ITableMetaWhere): ITableMetaWhere {
    return (eb) => eb.not(inner(eb));
  }
}
```

**PostgresTableRepository.findOne 核心逻辑（可复核）**：

```typescript
// adapter-repository-postgres/src/repositories/PostgresTableRepository.ts:579-617
async findOne(
  context: core.IExecutionContext,
  spec: core.ISpecification<core.Table, core.ITableSpecVisitor>,
  options?: Pick<core.TableFindOptions, 'state'>
): Promise<Result<core.Table, DomainError>> {
  return safeTry<core.Table, DomainError>(async function* (this: PostgresTableRepository) {
    // 步骤 3: 创建 Visitor
    const visitor = new TableWhereVisitor(options?.state);

    // 步骤 4-5: 双分派 + 生成条件
    const acceptResult = spec.accept(visitor);
    if (acceptResult.isErr()) return err(acceptResult.error);

    // 步骤 6: 获取 where 工厂
    const whereResult = visitor.where();
    if (whereResult.isErr()) return err(whereResult.error);
    const whereFactory = whereResult.value;

    // 步骤 7: 解析事务上下文
    const db = resolvePostgresDbOrTx(this.db, context, 'meta');

    // 步骤 8: 构建查询
    const baseQuery = db
      .selectFrom('table_meta')
      .leftJoinLateral(fieldsLateral, (join) => join.onTrue())
      .leftJoinLateral(viewsLateral, (join) => join.onTrue())
      .select(['table_meta.id', 'table_meta.name', 'fields.fields', 'views.views'])
      .where((eb) => whereFactory(eb));

    // 步骤 9: 执行
    const tableRow = await baseQuery.executeTakeFirst();

    // 步骤 10: 映射
    const tableResult = this.tableMapper.toDomain(tableRow);
    if (tableResult.isErr()) return err(tableResult.error);

    return ok(tableResult.value);
  }.bind(this));
}
```

#### 8.1.2 更新调用链：updateOne

```
应用层（Command Handler）
         │
         ▼  【步骤 1】构建变更 Specification
    TableAddFieldSpec.create(field: Field)
         │
         ▼  【步骤 2】调用 repository 接口
  ITableRepository.updateOne(context, table, mutateSpec)
         │
         ▼  ┌──────────────────────────────────────────────────────────────┐
            │  Adapter 层：PostgresTableRepository                            │
            │                                                                 │
            │  【步骤 3】验证表状态：从 DB 加载当前版本（乐观锁校验）          │
            │     const existing = yield* this.findOneOrFail(context,       │
            │       TableByIdSpec.create(table.id()), options);             │
            │                                                                 │
            │  【步骤 4】创建 TableMetaUpdateVisitor                         │
            │     const visitor = new TableMetaUpdateVisitor({              │
            │       db, table, existingVersion, now, actorId, ...           │
            │     });                                                        │
            │                                                                 │
            │  【步骤 5】双分派：mutateSpec.accept(visitor)                  │
            │     ↓ TableAddFieldSpec.accept() 实现：                        │
            │     accept(visitor) { return visitor.visitTableAddField(this); }│
            │                                                                 │
            │  【步骤 6】Visitor.visitTableAddField() 生成 SQL 语句          │
            │     → this.fieldRowBuilder.buildRowForField(field)            │
            │     → buildInsertOrReviveFieldStatement(fieldRow)             │
            │       · INSERT INTO field VALUES (...)                        │
            │       · ON CONFLICT (id) DO UPDATE SET                        │
            │         deleted_time = NULL, version = version + 1            │
            │     → statements = [insertOrReviveStmt]                       │
            │     → this.addCond(statements)                                │
            │                                                                 │
            │  【步骤 7】获取所有语句：visitor.where()                       │
            │     → 返回 ReadonlyArray<TableUpdateBuilder>                  │
            │                                                                 │
            │  【步骤 8】编译并执行所有语句                                  │
            │     const results = yield* executeCompiledQueries(            │
            │       db, statements, { method: 'updateOne' }                 │
            │     );                                                         │
            │                                                                 │
            │  【步骤 9】收集版本变更信息                                    │
            │     → 从 RETURNING 子句提取新的 field/view version            │
            │     → 构建 FieldPersistResult[]                               │
            │                                                                 │
            │  【步骤 10】返回 TableUpdatePersistResult                      │
            │     { fieldVersions: [...], viewVersions: [...] }             │
            └──────────────────────────────────────────────────────────────┘
         │
         ▼  返回版本变更结果
```

**TableMetaUpdateVisitor 核心实现（可复核）**：

```typescript
// adapter-repository-postgres/src/repositories/visitors/TableMetaUpdateVisitor.ts:138-165
// 按 ITableSpecVisitor 接口方法逐项实现：
visitTableAddField(
  spec: core.TableAddFieldSpec
): Result<ReadonlyArray<TableUpdateBuilder>, DomainError> {
  const fieldRowResult = this.fieldRowBuilder.buildRowForField(spec.field());
  if (fieldRowResult.isErr()) return err(fieldRowResult.error);

  const statements: ReadonlyArray<TableUpdateBuilder> = [
    this.buildInsertOrReviveFieldStatement(fieldRowResult.value),
  ];

  this.mergeSpecInfo({ specName: 'TableAddFieldSpec' });
  return this.addCond(statements).map(() => statements);
}

// 软删除恢复逻辑（可复核）
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

visitTableUpdateField(
  spec: core.TableUpdateFieldSpec
): Result<ReadonlyArray<TableUpdateBuilder>, DomainError> {
  const statements: ReadonlyArray<TableUpdateBuilder> = [
    this.params.db
      .updateTable('field')
      .set({ name: spec.field().name().toString() })
      .where('id', '=', spec.field().id().toString())
      .where('version', '=', this.fieldRowBuilder.fieldVersion(spec.field())),
  ];
  return this.addCond(statements).map(() => statements);
}

visitTableDeleteField(
  spec: core.TableDeleteFieldSpec
): Result<ReadonlyArray<TableUpdateBuilder>, DomainError> {
  const statements: ReadonlyArray<TableUpdateBuilder> = [
    this.params.db
      .updateTable('field')
      .set({ deleted_time: this.params.now })  // 软删除
      .where('id', '=', spec.fieldId().toString())
      .where('version', '=', this.fieldRowBuilder.fieldVersion(spec.field())),
  ];
  return this.addCond(statements).map(() => statements);
}

// ... 20+ 个 visit 方法处理不同的变更操作
```

**PostgresTableRepository.updateOne 核心逻辑（可复核）**：

```typescript
// adapter-repository-postgres/src/repositories/PostgresTableRepository.ts:823-905
async updateOne(
  context: core.IExecutionContext,
  table: core.Table,
  mutateSpec: core.ISpecification<core.Table, core.ITableSpecVisitor>,
  options?: core.TableFindOptions
): Promise<Result<core.TableUpdatePersistResult | void, DomainError>> {
  return safeTry<core.TableUpdatePersistResult | void, DomainError>(
    async function* (this: PostgresTableRepository) {
      // 步骤 3: 验证表状态（乐观锁）
      const existingResult = yield* await this.findOneOrFail(
        context,
        core.TableByIdSpec.create(table.id()),
        options
      );

      // 步骤 4: 创建 Visitor
      const db = resolvePostgresDbOrTx(this.db, context, 'meta');
      const visitor = new TableMetaUpdateVisitor({
        db,
        table,
        existingVersion: existingResult.version(),
        now: new Date().toISOString(),
        actorId: context.actorId.toString(),
        fieldRowBuilder: this.fieldRowBuilder,
      });

      // 步骤 5-6: 双分派 + 生成 SQL 语句
      const acceptResult = mutateSpec.accept(visitor);
      if (acceptResult.isErr()) return err(acceptResult.error);

      // 步骤 7: 获取语句数组
      const whereResult = visitor.where();
      if (whereResult.isErr()) return err(whereResult.error);
      const statements = whereResult.value;

      // 步骤 8: 执行所有语句
      const results = yield* await executeCompiledQueries(db, statements, {
        method: 'updateOne',
        tableId: table.id().toString(),
      });

      // 步骤 9-10: 收集版本变更并返回
      return ok({
        fieldVersions: extractFieldVersions(results),
        viewVersions: extractViewVersions(results),
      });
    }.bind(this)
  );
}
```

### 8.2 记录仓储调用链（ITableRecordRepository + ITableRecordQueryRepository）

**场景**：业务表记录的 CRUD 操作

#### 8.2.1 查询调用链：find

```
应用层（Query Handler）
         │
         ▼  【步骤 1】构建记录查询 Specification
    SingleLineTextConditionSpec.create(
      field: textField,
      operator: 'equals',
      value: RecordConditionLiteralValue.create('foo')
    )
         │
         ▼  【步骤 2】调用 repository
  ITableRecordQueryRepository.find(context, table, spec, options)
         │
         ▼  ┌──────────────────────────────────────────────────────────────┐
            │  Adapter 层：PostgresTableRecordQueryRepository                │
            │                                                               │
            │  【步骤 3】queryBuilderManager.createBuilder(table, options)  │
            │  【步骤 4】queryBuilder.select(projectionFields)              │
            │  【步骤 5】如有 spec：                                         │
            │     → 创建 TableRecordConditionWhereVisitor()                │
            │     → spec.accept(visitor)         [双分派关键节点]            │
            │       ↓ 调用对应 visit 方法                                   │
            │       visitSingleLineTextIs(spec)                             │
            │         → this.applyIs(field, value)                         │
            │           → 生成 SQL: "fld_xxx" = 'foo'                      │
            │     → visitor.where() → 获取 sql 表达式                       │
            │     → queryBuilder.where(sqlExpr)                            │
            │  【步骤 6】处理 orderBy、pagination                            │
            │  【步骤 7】queryBuilder.build() → 生成完整 Kysely AST         │
            │  【步骤 8】execute() → 编译并执行 SQL                          │
            │  【步骤 9】映射结果为 TableRecordReadModel                     │
            └──────────────────────────────────────────────────────────────┘
         │
         ▼  返回查询结果
```

**TableRecordConditionWhereVisitor 核心实现（按真实方法名逐项对齐）**：

```typescript
// core/src/domain/table/records/specs/ITableRecordConditionSpecVisitor.ts:26-263
// 每个操作符对应独立的 visit 方法，共 130+ 个方法
export interface ITableRecordConditionSpecVisitor<TResult = unknown> {
  // 基础查询
  visitRecordById(spec: RecordByIdSpec): Result<TResult, DomainError>;
  visitRecordByIds(spec: RecordByIdsSpec): Result<TResult, DomainError>;
  visitIncomingLinkSelected(spec: IncomingLinkSelectedSpec): Result<TResult, DomainError>;
  visitIncomingLinkCandidate(spec: IncomingLinkCandidateSpec): Result<TResult, DomainError>;

  // SingleLineText（6 个操作符）
  visitSingleLineTextIs(spec: SingleLineTextConditionSpec): Result<TResult, DomainError>;
  visitSingleLineTextIsNot(spec: SingleLineTextConditionSpec): Result<TResult, DomainError>;
  visitSingleLineTextContains(spec: SingleLineTextConditionSpec): Result<TResult, DomainError>;
  visitSingleLineTextDoesNotContain(spec: SingleLineTextConditionSpec): Result<TResult, DomainError>;
  visitSingleLineTextIsEmpty(spec: SingleLineTextConditionSpec): Result<TResult, DomainError>;
  visitSingleLineTextIsNotEmpty(spec: SingleLineTextConditionSpec): Result<TResult, DomainError>;

  // Number（8 个操作符）
  visitNumberIs(spec: NumberConditionSpec): Result<TResult, DomainError>;
  visitNumberIsNot(spec: NumberConditionSpec): Result<TResult, DomainError>;
  visitNumberIsGreater(spec: NumberConditionSpec): Result<TResult, DomainError>;
  visitNumberIsGreaterEqual(spec: NumberConditionSpec): Result<TResult, DomainError>;
  visitNumberIsLess(spec: NumberConditionSpec): Result<TResult, DomainError>;
  visitNumberIsLessEqual(spec: NumberConditionSpec): Result<TResult, DomainError>;
  visitNumberIsEmpty(spec: NumberConditionSpec): Result<TResult, DomainError>;
  visitNumberIsNotEmpty(spec: NumberConditionSpec): Result<TResult, DomainError>;

  // User（12 个操作符）
  visitUserIs(spec: UserConditionSpec): Result<TResult, DomainError>;
  visitUserIsNot(spec: UserConditionSpec): Result<TResult, DomainError>;
  visitUserIsAnyOf(spec: UserConditionSpec): Result<TResult, DomainError>;
  visitUserIsNoneOf(spec: UserConditionSpec): Result<TResult, DomainError>;
  visitUserHasAnyOf(spec: UserConditionSpec): Result<TResult, DomainError>;
  visitUserHasAllOf(spec: UserConditionSpec): Result<TResult, DomainError>;
  visitUserIsExactly(spec: UserConditionSpec): Result<TResult, DomainError>;
  visitUserIsNotExactly(spec: UserConditionSpec): Result<TResult, DomainError>;
  visitUserHasNoneOf(spec: UserConditionSpec): Result<TResult, DomainError>;
  visitUserIsEmpty(spec: UserConditionSpec): Result<TResult, DomainError>;
  visitUserIsNotEmpty(spec: UserConditionSpec): Result<TResult, DomainError>;
  visitUserContains(spec: UserConditionSpec): Result<TResult, DomainError>;

  // Link（16 个操作符）
  visitLinkIs(spec: LinkConditionSpec): Result<TResult, DomainError>;
  visitLinkIsNot(spec: LinkConditionSpec): Result<TResult, DomainError>;
  visitLinkIsAnyOf(spec: LinkConditionSpec): Result<TResult, DomainError>;
  visitLinkIsNoneOf(spec: LinkConditionSpec): Result<TResult, DomainError>;
  visitLinkHasAnyOf(spec: LinkConditionSpec): Result<TResult, DomainError>;
  visitLinkHasAllOf(spec: LinkConditionSpec): Result<TResult, DomainError>;
  visitLinkIsExactly(spec: LinkConditionSpec): Result<TResult, DomainError>;
  visitLinkIsNotExactly(spec: LinkConditionSpec): Result<TResult, DomainError>;
  visitLinkHasNoneOf(spec: LinkConditionSpec): Result<TResult, DomainError>;
  visitLinkContains(spec: LinkConditionSpec): Result<TResult, DomainError>;
  visitLinkDoesNotContain(spec: LinkConditionSpec): Result<TResult, DomainError>;
  visitLinkIsEmpty(spec: LinkConditionSpec): Result<TResult, DomainError>;
  visitLinkIsNotEmpty(spec: LinkConditionSpec): Result<TResult, DomainError>;

  // ... LongText, Button, Rating, Checkbox, Date, SingleSelect, MultipleSelect,
  //     Attachment, Formula, Rollup, ConditionalRollup, ConditionalLookup
}
```

```typescript
// adapter-table-repository-postgres/src/record/visitors/TableRecordConditionWhereVisitor.ts:1440-1444
// 真实实现：每个 visit 方法委托给对应的 apply* 方法
visitSingleLineTextIs(
  spec: core.SingleLineTextConditionSpec
): Result<RecordConditionWhere, DomainError> {
  return this.applyIs(spec.field(), spec.value());
}

// adapter-table-repository-postgres/src/record/visitors/TableRecordConditionWhereVisitor.ts:1764-1766
visitUserIs(spec: core.UserConditionSpec): Result<RecordConditionWhere, DomainError> {
  return this.applyIs(spec.field(), spec.value());
}

// adapter-table-repository-postgres/src/record/visitors/TableRecordConditionWhereVisitor.ts:1808-1810
visitLinkIs(spec: core.LinkConditionSpec): Result<RecordConditionWhere, DomainError> {
  return this.applyIs(spec.field(), spec.value());
}
```

**applyIs 核心逻辑（可复核）**：

```typescript
// adapter-table-repository-postgres/src/record/visitors/TableRecordConditionWhereVisitor.ts:695-826
const buildIsCondition = (
  field: core.Field,
  value: core.RecordConditionValue | undefined,
  tableAlias?: string,
  hostTableAlias?: string
): Result<RecordConditionWhere, DomainError> => {
  return safeTry<RecordConditionWhere, DomainError>(function* () {
    const column = yield* resolveColumn(field, tableAlias);     // 步骤 A: 解析列名
    const columnRef = sql.ref(column);                           // 步骤 B: 构造列引用
    const isMultiple = yield* fieldIsMultiple(field);            // 步骤 C: 判断是否多值

    // 分支 1: 日期值 → BETWEEN 范围查询
    if (core.isRecordConditionDateValue(value)) {
      const range = yield* resolveDateRange(value, resolveDateFormatting(field));
      return ok(sql`${columnRef} between ${range.start} and ${range.end}`);
    }

    // 分支 2: User/Link 字段 → JSONB 路径提取 + id 匹配
    const isUserOrLinkLike = fieldIsUserOrLink(field);
    const operand = yield* resolvePrimitiveOperand(value, tableAlias, hostTableAlias);
    if (isUserOrLinkLike && operand.kind === 'literal') {
      return ok(
        sql`jsonb_extract_path_text(to_jsonb(${columnRef}), 'id') = ${String(operand.value)}`
      );
    }

    // 分支 3: 多值字段 → jsonb_array_elements_text + EXISTS
    if (isMultiple && operand.kind === 'literal') {
      const normalizedArray = normalizeToJsonArray(columnRef);
      return ok(sql`EXISTS (
        SELECT 1 FROM jsonb_array_elements_text(${normalizedArray}) AS elem
        WHERE elem = ${primitiveLiteralToText(operand.value)}
      )`);
    }

    // 分支 4: 普通字段 → 直接相等比较
    return ok(sql`${columnRef} = ${operand.value}`);
  });
};
```

#### 8.2.2 插入调用链：insert

```
应用层（Command Handler）
         │
         ▼  【步骤 1】构建 TableRecord
    TableRecord.create(fieldIdValueMap)
         │
         ▼  【步骤 2】调用 repository
  ITableRecordRepository.insert(context, table, record, options)
         │
         ▼  ┌──────────────────────────────────────────────────────────────────┐
            │  Adapter 层：PostgresTableRecordRepository                          │
            │                                                                   │
            │  【步骤 3】构建执行上下文                                          │
            │     → resolvePostgresDbOrTx(db, context)     [事务解析]           │
            │     → resolveActorIdentity(actorLookupDb, actorId)                │
            │     → getViewOrderInfo(db, tableName, views)                      │
            │                                                                   │
            │  【步骤 4】RecordInsertBuilder.buildInsertData()                  │
            │     ┌────────────────────────────────────────────────────────┐    │
            │     │ 遍历所有字段（table.getFields()）                       │    │
            │     │                                                        │    │
            │     │ 对于每个字段：                                          │    │
            │     │   → field.dbFieldName() → 数据库列名                    │    │
            │     │   → 创建 FieldInsertValueVisitor(rawValue, ctx)        │    │
            │     │   → field.accept(insertVisitor)     [双分派]            │    │
            │     │     · visitSingleLineTextField() → simpleValueFrom()    │    │
            │     │     · visitUserField() → jsonValueFrom()                │    │
            │     │     · visitLinkField() → 返回 {columnValues,            │    │
            │     │                         queryExecutors}                  │    │
            │     │                                                        │    │
            │     │   ⚠️  关键：queryExecutors 仅返回，不执行                │    │
            │     │                                                        │    │
            │     │   【链接字段特殊路径】                                   │    │
            │     │   if (field.type === 'link') {                          │    │
            │     │     → columnValues 并入主 INSERT 值                     │    │
            │     │     → 忽略 visitor 返回的 queryExecutors                │    │
            │     │     → 调用 builder.buildLinkFieldSqls(field, value)     │    │
            │     │       · 解析 linkItems                                  │    │
            │     │       · 根据关系类型生成 SQL：                            │    │
            │     │         manyMany → junction 表 DELETE + INSERT          │    │
            │     │         manyOne  → 主表 FK 列赋值                        │    │
            │     │         oneMany  → 外表 UPDATE SET FK                   │    │
            │     │         oneOne   → 对称链接 UPDATE 外表 FK              │    │
            │     │       · 生成 CompiledSqlStatement 数组                  │    │
            │     │     → additionalStatements.push(...statements)          │    │
            │     │   }                                                     │    │
            │     └────────────────────────────────────────────────────────┘    │
            │                                                                   │
            │  【步骤 5】构建主 INSERT 语句                                      │
            │     → 合并系统列（__id, __version, __created_time, 等）          │
            │     → 合并视图排序列（__row_viewId）                              │
            │     → 构建：INSERT INTO table VALUES (...)                       │
            │     → 如有 changedFields：RETURNING 子句                         │
            │                                                                   │
            │  【步骤 6】执行主插入                                              │
            │     → db.insertInto(tableName).values(valuesWithViewOrder)       │
            │     → .executeTakeFirst() / .execute()                          │
            │                                                                   │
            │  【步骤 7】内联执行 additionalStatements                          │
            │     → RecordInsertBuilder.executeStatements(db, statements)      │
            │     → 循环执行每条：db.executeQuery(stmt.compiled)               │
            │       · junction 表 INSERT（manyMany 链接）                      │
            │       · 外表 UPDATE SET FK（oneMany 链接）                       │
            │       · 附件索引表 INSERT                                        │
            │                                                                   │
            │  【步骤 8】计算字段更新                                            │
            │     → runComputedUpdate(context, table, record, 'insert')       │
            │     → 收集快照                                                    │
            │  【步骤 9】返回 RecordMutationResult                              │
            └──────────────────────────────────────────────────────────────────┘
         │
         ▼  返回插入结果
```

**FieldInsertValueVisitor 核心实现（按真实方法名逐项对齐）**：

```typescript
// adapter-table-repository-postgres/src/record/visitors/FieldInsertValueVisitor.ts:76-212
export class FieldInsertValueVisitor implements IFieldVisitor<FieldInsertResult> {
  private simpleValueFrom(value: unknown): Result<FieldInsertResult, DomainError> {
    return ok({
      columnValues: { [this.ctx.dbFieldName]: value ?? null },
      queryExecutors: [],  // 简单字段无额外执行器
    });
  }

  private jsonValueFrom(value: unknown): Result<FieldInsertResult, DomainError> {
    // JSONB 列：必须 JSON.stringify（pg 驱动要求）
    const serialized = value === null || value === undefined ? null : JSON.stringify(value);
    return ok({
      columnValues: { [this.ctx.dbFieldName]: serialized },
      queryExecutors: [],
    });
  }

  private computedField(): Result<FieldInsertResult, DomainError> {
    return ok({ columnValues: {}, queryExecutors: [] });
  }

  // 按 IFieldVisitor 接口的方法名逐项对齐：
  visitSingleLineTextField(_field: SingleLineTextField): Result<FieldInsertResult, DomainError> {
    return this.simpleValueFrom(this.rawValue);
  }
  visitLongTextField(_field: LongTextField): Result<FieldInsertResult, DomainError> {
    return this.simpleValueFrom(this.rawValue);
  }
  visitNumberField(_field: NumberField): Result<FieldInsertResult, DomainError> {
    return this.simpleValueFrom(this.rawValue);
  }
  visitRatingField(_field: RatingField): Result<FieldInsertResult, DomainError> {
    return this.simpleValueFrom(this.rawValue);
  }
  visitCheckboxField(_field: CheckboxField): Result<FieldInsertResult, DomainError> {
    return this.simpleValueFrom(this.rawValue);
  }
  visitDateField(_field: DateField): Result<FieldInsertResult, DomainError> {
    return this.simpleValueFrom(this.rawValue);
  }
  visitSingleSelectField(field: SingleSelectField): Result<FieldInsertResult, DomainError> {
    return this.simpleValueFrom(this.mapSingleSelectValue(field, this.rawValue));
  }
  visitMultipleSelectField(field: MultipleSelectField): Result<FieldInsertResult, DomainError> {
    return this.jsonValueFrom(this.mapMultipleSelectValue(field, this.rawValue));
  }
  visitAttachmentField(_field: AttachmentField): Result<FieldInsertResult, DomainError> {
    return this.jsonValueFrom(this.rawValue);
  }
  visitUserField(_field: UserField): Result<FieldInsertResult, DomainError> {
    return this.jsonValueFrom(this.rawValue);
  }
  visitFormulaField(_field: FormulaField): Result<FieldInsertResult, DomainError> {
    return this.computedField();
  }
  visitRollupField(_field: RollupField): Result<FieldInsertResult, DomainError> {
    return this.computedField();
  }
  visitLookupField(_field: LookupField): Result<FieldInsertResult, DomainError> {
    return this.computedField();
  }
  visitConditionalLookupField(_field: ConditionalLookupField): Result<FieldInsertResult, DomainError> {
    return this.computedField();
  }
  visitConditionalRollupField(_field: ConditionalRollupField): Result<FieldInsertResult, DomainError> {
    return this.computedField();
  }
  visitCreatedTimeField(_field: CreatedTimeField): Result<FieldInsertResult, DomainError> {
    return this.computedField();
  }
  visitLastModifiedTimeField(_field: LastModifiedTimeField): Result<FieldInsertResult, DomainError> {
    return this.computedField();
  }
  visitCreatedByField(_field: CreatedByField): Result<FieldInsertResult, DomainError> {
    return this.computedField();
  }
  visitLastModifiedByField(_field: LastModifiedByField): Result<FieldInsertResult, DomainError> {
    return this.computedField();
  }
  visitAutoNumberField(_field: AutoNumberField): Result<FieldInsertResult, DomainError> {
    return this.computedField();
  }
  visitButtonField(_field: ButtonField): Result<FieldInsertResult, DomainError> {
    return this.computedField();
  }
```

**链接字段 visitLinkField 实现（可复核）**：

```typescript
// adapter-table-repository-postgres/src/record/visitors/FieldInsertValueVisitor.ts:213-349
visitLinkField(field: LinkField): Result<FieldInsertResult, DomainError> {
  return safeTry<FieldInsertResult, DomainError>(
    function* (this: FieldInsertValueVisitor) {
      const columnValues: Record<string, unknown> = {};
      const queryExecutors: QueryExecutor[] = [];  // ⚠️  返回但不执行

      // 【步骤 A】处理 JSONB 列值
      const storedValue = field.isMultipleValue()
        ? this.rawValue
        : Array.isArray(this.rawValue)
          ? this.rawValue[0] ?? null
          : this.rawValue;
      columnValues[this.ctx.dbFieldName] =
        storedValue === null || storedValue === undefined ? null : JSON.stringify(storedValue);

      if (this.rawValue === null || this.rawValue === undefined) {
        return ok({ columnValues, queryExecutors });
      }

      // 【步骤 B】解析 link items
      const linkItems = Array.isArray(this.rawValue)
        ? (this.rawValue as Array<{ id: string; title?: string }>)
        : [this.rawValue as { id: string; title?: string }];

      // 【步骤 C】根据关系类型生成 queryExecutors
      const relationship = field.relationship().toString();

      // CASE 1: manyMany / oneMany 单向 → junction 表
      if (relationship === 'manyMany' || (relationship === 'oneMany' && field.isOneWay())) {
        const fkHostTableName = yield* field.fkHostTableName();
        const tableName = yield* fkHostTableName.split({ defaultSchema: 'public' });
        const selfKeyName = yield* field.selfKeyNameString();
        const foreignKeyName = yield* field.foreignKeyNameString();

        for (let i = 0; i < linkItems.length; i++) {
          const insertValues = {
            [selfKeyName]: this.ctx.recordId,
            [foreignKeyName]: linkItems[i].id,
            ...(orderColumnName ? { [orderColumnName]: i + 1 } : {}),
          };

          // 策略：先 DELETE 再 INSERT（比 ON CONFLICT 更稳健）
          queryExecutors.push(async (db) => {        // ⚠️  推入数组但不执行
            await db
              .deleteFrom(tableName)
              .where(selfKeyName, '=', this.ctx.recordId)
              .where(foreignKeyName, '=', linkItems[i].id)
              .execute();
            await db.insertInto(tableName).values(insertValues).execute();
          });
        }
      }
      // CASE 2: manyOne / oneOne → 主表 FK 列或对称链接外表 UPDATE
      else if (relationship === 'manyOne' || relationship === 'oneOne') {
        const foreignKeyName = yield* field.foreignKeyNameString();
        if (foreignKeyName === '__id') {
          // 对称链接：UPDATE 外表 FK
          queryExecutors.push((db) =>                 // ⚠️  推入数组但不执行
            db
              .updateTable(foreignTableName)
              .set({ [selfKeyName]: this.ctx.recordId })
              .where('__id', '=', linkItems[0].id)
              .execute()
          );
        } else {
          // 普通 FK：直接在 columnValues 中赋值
          columnValues[foreignKeyName] = linkItems[0].id;
        }
      }
      // CASE 3: oneMany 双向 → UPDATE 外表 FK
      else if (relationship === 'oneMany') {
        for (const linkItem of linkItems) {
          queryExecutors.push((db) =>                 // ⚠️  推入数组但不执行
            db
              .updateTable(foreignTableName)
              .set({ [selfKeyName]: this.ctx.recordId })
              .where('__id', '=', linkItem.id)
              .execute()
          );
        }
      }

      return ok({ columnValues, queryExecutors });  // ⚠️  queryExecutors 返回给调用方
    }.bind(this)
  );
}
```

**⚠️  关键事实校准**：
- `FieldInsertValueVisitor.visitLinkField()` 返回的 `queryExecutors` 在生产代码中**不被执行**
- 实际执行路径：`RecordInsertBuilder.buildInsertData()` 检测到链接字段时，调用 `buildLinkFieldSqls()` 生成 `additionalStatements`
- `additionalStatements` 通过 `RecordInsertBuilder.executeStatements()` 内联执行
- `queryExecutors` 仅用于单元测试（参见 `FieldInsertValueVisitor.spec.ts`）

### 8.3 结构仓储调用链（ITableSchemaRepository）

**场景**：物理表结构的 DDL 操作（CREATE TABLE, ALTER TABLE 等）

#### 8.3.1 创建表调用链：insert

```
应用层（CreateTableCommand Handler）
         │
         ▼  【步骤 1】已创建 Table 领域对象（含所有字段）
         │
         ▼  【步骤 2】调用 repository 接口
  ITableSchemaRepository.insert(context: IExecutionContext, table: Table)
         │
         ▼  ┌──────────────────────────────────────────────────────────────┐
            │  Adapter 层：PostgresTableSchemaRepository                       │
            │                                                                 │
            │  【步骤 3】解析表名                                               │
            │     const dbTableName = yield* table.dbTableName();              │
            │     const tableName = yield* dbTableName.value();                │
            │                                                                 │
            │  【步骤 4】创建表结构 DDL 构建器                                 │
            │     const schemaBuilder = this.db.schema.createTable(tableName)  │
            │       .addColumn(RECORD_ID_COLUMN, 'uuid', (col) =>             │
            │         col.primaryKey()                                        │
            │       )                                                         │
            │       .addColumn(VERSION_COLUMN, 'integer', (col) =>            │
            │         col.defaultTo(1).notNull()                              │
            │       )                                                         │
            │       .addColumn(CREATED_TIME_COLUMN, 'timestamptz')            │
            │       .addColumn(LAST_MODIFIED_TIME_COLUMN, 'timestamptz')      │
            │       .addColumn(CREATED_BY_COLUMN, 'text')                     │
            │       .addColumn(LAST_MODIFIED_BY_COLUMN, 'text');              │
            │                                                                 │
            │  【步骤 5】创建字段创建 Visitor                                   │
            │     const visitor =                                             │
            │       PostgresTableSchemaFieldCreateVisitor.forTableCreation(   │
            │         { builder: schemaBuilder, db: this.db }                 │
            │       );                                                         │
            │                                                                 │
            │  【步骤 6】遍历所有字段，双分派生成列定义                          │
            │     for (const field of table.getFields()) {                    │
            │       if (field.computed().toBoolean()) continue;               │
            │       const result = field.accept(visitor);                      │
            │       // 调用对应 visit 方法：                                    │
            │       // visitSingleLineTextField() → ADD COLUMN text           │
            │       // visitNumberField() → ADD COLUMN numeric                │
            │       // visitDateField() → ADD COLUMN timestamptz              │
            │       // visitUserField() → ADD COLUMN jsonb                     │
            │       // visitLinkField() → ADD COLUMN jsonb + foreign_key       │
            │     }                                                           │
            │                                                                 │
            │  【步骤 7】执行 CREATE TABLE DDL                                  │
            │     await schemaBuilder.execute();                               │
            │                                                                 │
            │  【步骤 8】创建搜索索引（可选）                                   │
            │     for (const field of table.getFields()) {                    │
            │       if (needsSearchIndex(field)) {                             │
            │         await db.schema                                         │
            │           .createIndex(...)                                     │
            │           .using('gin')                                         │
            │           .expression(sql`(${col} gin_trgm_ops)`)              │
            │           .execute();                                            │
            │       }                                                         │
            │     }                                                           │
            │                                                                 │
            │  【步骤 9】初始化 undo capture 基础设施                           │
            │     await installUndoCaptureTriggers(db, tableName);            │
            │     await createRecordMutationSnapshotTable(db, tableName);     │
            │                                                                 │
            │  【步骤 10】填充链接字段标题等初始数据                             │
            │     await populateLinkFieldTitles(db, table);                   │
            └──────────────────────────────────────────────────────────────┘
         │
         ▼  返回 Result<void, DomainError>
```

#### 8.3.2 字段变更调用链：update

```
应用层（UpdateFieldCommand Handler）
         │
         ▼  【步骤 1】构建变更 Specification
    TableUpdateFieldTypeSpec.create(oldField: Field, newField: Field)
         │
         ▼  【步骤 2】调用 repository 接口（⚠️  接口方法名是 update，不是 updateOne）
  ITableSchemaRepository.update(context, table, mutateSpec)
         │
         ▼  ┌──────────────────────────────────────────────────────────────┐
            │  Adapter 层：PostgresTableSchemaRepository                       │
            │                                                                 │
            │  【步骤 3】ensureDbFieldNames(table.getFields())                │
            │     确保所有字段都有 dbFieldName，否则重新注入                    │
            │                                                                 │
            │  【步骤 4】解析表名和事务上下文                                   │
            │     const { schema, tableName } = yield* table.dbTableName()    │
            │       .andThen((name) => name.split({ defaultSchema: null }));  │
            │     const db = resolvePostgresDbOrTx(this.db, context);         │
            │                                                                 │
            │  【步骤 5】创建 TableSchemaUpdateVisitor                         │
            │     const visitor = new TableSchemaUpdateVisitor({              │
            │       db, schema, tableName, tableId: table.id(), table         │
            │     });                                                         │
            │                                                                 │
            │  【步骤 6】双分派：mutateSpec.accept(visitor)                    │
            │     ↓ TableUpdateFieldTypeSpec.accept() 实现：                  │
            │     accept(visitor) {                                           │
            │       return visitor.visitTableUpdateFieldType(this);           │
            │     }                                                           │
            │                                                                 │
            │  【步骤 7】Visitor.visitTableUpdateFieldType() 生成 DDL 语句     │
            │                                                                 │
            │     【分支 7a】非类型转换（仅引用变化）                           │
            │     if (!spec.isTypeConversion()) {                             │
            │       // 仅 regenerate 引用表记录（如 ConditionalRollup 配置变） │
            │       statements = yield* visitor.regenerateFieldReferences(...)│
            │       return this.addCond(statements).map(() => statements);    │
            │     }                                                           │
            │                                                                 │
            │     【分支 7b】类型转换（完整流程）                               │
            │     → dbFieldName = visitor.resolveDbFieldNameText(oldField)    │
            │     → conversionParams = { db, schema, tableName, tableId, ...}│
            │     → conversionStatements = yield* generateFieldConversionSta │
            │       tements(conversionParams, oldField, newField)             │
            │       · 内部通过 FieldTypeConversionVisitor 处理各种场景        │
            │       · link→link, link→text, link→select, scalar→link, 等      │
            │     → referenceStatements = visitor.regenerateFieldReferences(…)│
            │     → dropSearchIdx = visitor.dropSearchIndexStatement(...)    │
            │     → createSearchIdx = visitor.createSearchIndexStatement(...) │
            │                                                                 │
            │     【语句顺序 7c】                                              │
            │     statements = [                                              │
            │       dropSearchIdx,                                            │
            │       ...conversionStatements,                                  │
            │       ...referenceStatements,                                   │
            │       ...(createSearchIdx ? [createSearchIdx] : []),            │
            │     ];                                                          │
            │     this.addCond(statements);                                   │
            │                                                                 │
            │  【步骤 8】获取所有语句：visitor.where()                         │
            │     → 返回 ReadonlyArray<TableSchemaStatementBuilder>           │
            │                                                                 │
            │  【步骤 9】执行 DDL 语句（按 scope 分批执行）                     │
            │     await repository.executeScopedTableSchemaStatements(        │
            │       context, db, statements, { tracer, attributes }          │
            │     )                                                           │
            │     · 内部按 statement.scope 分组                                 │
            │     · 'meta' scope 使用 resolveMetaDb(context)                  │
            │     · 'data' scope 使用当前 db                                  │
            │     · 调用 executeTableSchemaStatements 执行                    │
            │                                                                 │
            │  【步骤 10】循环依赖检测                                          │
            │     const depDetector = new DependencyChangeDetectorVisitor();  │
            │     yield* mutateSpec.accept(depDetector);                     │
            │     if (depDetector.needsCheck()) {                             │
            │       const graphResult = yield* repository.fieldDependencyGraph│
            │         .load(table.baseId(), context, ...);                   │
            │       const cycleCheckResult = detectCircularDependency(edges); │
            │     }                                                           │
            │                                                                 │
            │  【步骤 11】收集值变更（用于级联更新）                             │
            │     const valueChanges = yield* repository.collectFieldValueChan│
            │       ges(mutateSpec);                                          │
            │     · 通过 FieldValueChangeCollectorVisitor 收集                 │
            │     · 返回 { selfBackfillFieldIds, valueChangedFieldIds,        │
            │                deferredBackfillFieldIds,                         │
            │                hasDbStorageTypeChange }                          │
            │                                                                 │
            │  【步骤 12】新增字段 backfill（如有）                              │
            │     const backfillVisitor = new TableAddFieldCollectorVisitor();│
            │     yield* mutateSpec.accept(backfillVisitor);                  │
            │     if (fields.length > 0) {                                    │
            │       yield* repository.computedFieldBackfillService.backfillMan│
            │         y(context, { table, fields, skipDistinctFilter: true });│
            │     }                                                           │
            │                                                                 │
            │  【步骤 13】级联更新依赖计算字段                                  │
            │     if (valueChanges.selfBackfillFieldIds.length > 0 ||         │
            │         valueChanges.valueChangedFieldIds.length > 0) {         │
            │       yield* repository.cascadeService.cascade(context, {       │
            │         table, selfBackfillFieldIds, valueChangedFieldIds, ...  │
            │       });                                                       │
            │     }                                                           │
            │                                                                 │
            │  【步骤 14】刷新内存表 + 后置动作 + 延迟回填                      │
            │     const nextTable = yield* repository.refreshInMemoryTableAfte│
            │       rUpdate(context, table, valueChanges.valueChangedFieldIds);│
            │     yield* repository.recordPostPersistActionTriggers(...);     │
            │     yield* repository.scheduleDeferredBackfillAfterUpdate(...); │
            │                                                                 │
            │  【步骤 15】返回更新后的 Table                                    │
            │     return ok(nextTable);                                       │
            └──────────────────────────────────────────────────────────────┘
         │
         ▼  返回 Result<Table, DomainError>
```

**PostgresTableSchemaFieldCreateVisitor 核心实现（可复核，按真实方法名对齐）**：

```typescript
// adapter-table-repository-postgres/src/schema/visitors/PostgresTableSchemaFieldCreateVisitor.ts
// 按 IFieldVisitor 接口的方法名逐项对齐：
export class PostgresTableSchemaFieldCreateVisitor
  implements core.IFieldVisitor<ReadonlyArray<TableSchemaStatementBuilder>>
{
  visitSingleLineTextField(
    field: core.SingleLineTextField
  ): Result<ReadonlyArray<TableSchemaStatementBuilder>, DomainError> {
    const dbFieldName = yield* field.dbFieldName().value();
    const columnDef = this.builderRef.builder
      .addColumn(dbFieldName, 'text')
      .$if(field.notNull().toBoolean(), (col) => col.notNull());
    return ok([{ compile: () => columnDef.compile() }]);
  }

  visitNumberField(
    field: core.NumberField
  ): Result<ReadonlyArray<TableSchemaStatementBuilder>, DomainError> {
    const dbFieldName = yield* field.dbFieldName().value();
    const columnDef = this.builderRef.builder
      .addColumn(dbFieldName, 'numeric')
      .$if(field.notNull().toBoolean(), (col) => col.notNull())
      .$if(field.unique().toBoolean(), (col) => col.unique());
    return ok([{ compile: () => columnDef.compile() }]);
  }

  visitDateField(
    field: core.DateField
  ): Result<ReadonlyArray<TableSchemaStatementBuilder>, DomainError> {
    const dbFieldName = yield* field.dbFieldName().value();
    const columnDef = this.builderRef.builder
      .addColumn(dbFieldName, 'timestamptz');
    return ok([{ compile: () => columnDef.compile() }]);
  }

  visitCheckboxField(
    field: core.CheckboxField
  ): Result<ReadonlyArray<TableSchemaStatementBuilder>, DomainError> {
    const dbFieldName = yield* field.dbFieldName().value();
    const columnDef = this.builderRef.builder
      .addColumn(dbFieldName, 'boolean')
      .defaultTo(false);
    return ok([{ compile: () => columnDef.compile() }]);
  }

  visitUserField(
    field: core.UserField
  ): Result<ReadonlyArray<TableSchemaStatementBuilder>, DomainError> {
    const dbFieldName = yield* field.dbFieldName().value();
    const columnDef = this.builderRef.builder
      .addColumn(dbFieldName, 'jsonb');  // JSONB 存储 {id, name, email}
    return ok([{ compile: () => columnDef.compile() }]);
  }

  visitLinkField(
    field: core.LinkField
  ): Result<ReadonlyArray<TableSchemaStatementBuilder>, DomainError> {
    const dbFieldName = yield* field.dbFieldName().value();
    const statements: TableSchemaStatementBuilder[] = [];

    // JSONB 列存储链接关系
    statements.push({
      compile: () =>
        this.builderRef.builder
          .addColumn(dbFieldName, 'jsonb')
          .compile(),
    });

    // manyOne/oneOne: 添加 FK 列
    if (isFkInMainTable(field)) {
      const fkColumnName = yield* field.foreignKeyNameString();
      statements.push({
        compile: () =>
          this.builderRef.builder
            .addColumn(fkColumnName, 'uuid')
            .addForeignKeyConstraint(
              `fk_${fkColumnName}`,
              [fkColumnName],
              yield* field.foreignTableName().value(),
              ['__id']
            )
            .compile(),
      });
    }

    return ok(statements);
  }

  // visitLongTextField, visitSingleSelectField, visitMultipleSelectField,
  // visitAttachmentField, visitFormulaField, visitRollupField, visitLookupField,
  // visitRatingField, visitButtonField, visitCreatedTimeField, ...
}
```

**TableSchemaUpdateVisitor 核心实现（可复核，按真实代码对齐）**：

```typescript
// adapter-table-repository-postgres/src/schema/visitors/TableSchemaUpdateVisitor.ts:632-694
// 按 ITableSpecVisitor 接口方法逐项实现：
export class TableSchemaUpdateVisitor
  extends core.AbstractSpecFilterVisitor<ReadonlyArray<TableSchemaStatementBuilder>>
  implements core.ITableSpecVisitor<ReadonlyArray<TableSchemaStatementBuilder>>
{
  visitTableAddField(
    spec: core.TableAddFieldSpec
  ): Result<ReadonlyArray<TableSchemaStatementBuilder>, DomainError> {
    // 生成 ADD COLUMN 语句
    const visitor = PostgresTableSchemaFieldCreateVisitor.forAlterTable({
      builder: this.params.db.schema.alterTable(this.params.tableName),
      db: this.params.db,
    });
    const fieldStatements = yield* spec.field().accept(visitor);
    return this.addCond(fieldStatements).map(() => fieldStatements);
  }

  // ⚠️  真实实现：visitTableUpdateFieldType 不直接使用 depDetector/valueCollector
  // 而是调用 generateFieldConversionStatements 函数处理各种转换场景
  visitTableUpdateFieldType(
    spec: core.TableUpdateFieldTypeSpec
  ): Result<ReadonlyArray<TableSchemaStatementBuilder>, DomainError> {
    const visitor = this;
    const addCond = this.addCond.bind(this);

    return safeTry<ReadonlyArray<TableSchemaStatementBuilder>, DomainError>(function* () {
      // 分支 1: 非类型转换（仅引用变化，如 ConditionalRollup 配置变更）
      if (!spec.isTypeConversion()) {
        const statements = yield* visitor.regenerateFieldReferences(
          spec.oldField(),
          spec.newField()
        );
        yield* addCond(statements);
        return ok(statements);
      }

      // 分支 2: 类型转换（完整流程）
      const oldField = spec.oldField();
      const newField = spec.newField();

      // 步骤 A: 解析 dbFieldName（从 oldField 获取，因为还没变更）
      const dbFieldNameResult = visitor.resolveDbFieldNameText(oldField);
      if (dbFieldNameResult.isErr()) return err(dbFieldNameResult.error);
      const dbFieldName = dbFieldNameResult.value;

      // 步骤 B: 调用 generateFieldConversionStatements（在 FieldTypeConversionVisitor.ts 中）
      const conversionParams: FieldConversionParams = {
        db: visitor.params.db,
        schema: visitor.params.schema,
        tableName: visitor.params.tableName,
        tableId: visitor.params.tableId,
        dbFieldName,
        fieldId: newField.id().toString(),
      };
      const conversionStatements = yield* generateFieldConversionStatements(
        conversionParams,
        oldField,
        newField
      );

      // 步骤 C: regenerate 引用表记录
      const referenceStatements = yield* visitor.regenerateFieldReferences(
        oldField,
        newField
      );

      // 步骤 D: 搜索索引管理（先删后建）
      const fieldId = newField.id().toString();
      const dropSearchIdx = visitor.dropSearchIndexStatement(fieldId, dbFieldName);
      const createSearchIdx = visitor.createSearchIndexStatement(newField, dbFieldName);

      // 步骤 E: 按顺序组装语句
      const statements = [
        dropSearchIdx,               // 先删除旧索引
        ...conversionStatements,     // 类型转换语句
        ...referenceStatements,      // 引用表更新
        ...(createSearchIdx ? [createSearchIdx] : []),  // 后创建新索引
      ];
      yield* addCond(statements);
      return ok(statements);
    });
  }

  visitTableDeleteField(
    spec: core.TableDeleteFieldSpec
  ): Result<ReadonlyArray<TableSchemaStatementBuilder>, DomainError> {
    const dbFieldName = yield* spec.field().dbFieldName().value();
    const statements: TableSchemaStatementBuilder[] = [
      {
        compile: () =>
          this.params.db.schema
            .alterTable(this.params.tableName)
            .dropColumn(dbFieldName)
            .compile(),
      },
    ];

    // 同时删除可能存在的 FK 列（manyOne/oneOne）
    if (isFkInMainTable(spec.field())) {
      const fkColumnName = yield* spec.field().foreignKeyNameString();
      statements.push({
        compile: () =>
          this.params.db.schema
            .alterTable(this.params.tableName)
            .dropColumn(fkColumnName)
            .compile(),
      });
    }

    return this.addCond(statements).map(() => statements);
  }

  // ... 其他 visit 方法：visitTableUpdateFieldDbFieldName, visitTableUpdateFieldConstraints,
  //     visitTableUpdateFieldHasError, visitUpdateSingleLineTextShowAs, 等 30+ 方法
}
```

**generateFieldConversionStatements 真实实现位置**：

```typescript
// adapter-table-repository-postgres/src/schema/visitors/FieldTypeConversionVisitor.ts:2938-2988
export function generateFieldConversionStatements(
  params: FieldConversionParams,
  oldField: Field,
  newField: Field
): Result<ReadonlyArray<TableSchemaStatementBuilder>, DomainError> {
  return safeTry<ReadonlyArray<TableSchemaStatementBuilder>, DomainError>(function* () {
    const isNewLink = newField.type().toString() === 'link';
    const isOldLink = oldField.type().toString() === 'link';

    // 场景 1: link → link（外键表变更时需要数据迁移）
    if (isNewLink && isOldLink) {
      const oldLinkField = oldField as LinkField;
      const newLinkField = newField as LinkField;
      const foreignChanged = !oldLinkField.foreignTableId().equals(newLinkField.foreignTableId());
      if (foreignChanged) {
        return yield* buildLinkToLinkForeignTableMigrationStatements(
          params, oldLinkField, newLinkField
        );
      }
    }

    // 场景 2: link → text（singleLineText / longText）
    if (isOldLink && !isNewLink) {
      const newType = newField.type().toString();
      if (newType === 'singleLineText' || newType === 'longText') {
        return yield* buildLinkToTextMigrationStatements(params, oldField as LinkField, newField);
      }
      if (newType === 'singleSelect' || newType === 'multipleSelect') {
        return yield* buildLinkToSelectMigrationStatements(
          params, oldField as LinkField, newField as SingleSelectField | MultipleSelectField
        );
      }
    }

    // 场景 3: scalar → link（保留源值，通过临时列迁移）
    if (isNewLink && !isOldLink) {
      const newLinkField = newField as LinkField;
      const oldType = oldField.type().toString();
      const isScalarSource = ['singleLineText', 'longText', 'singleSelect'].includes(oldType);
      if (isScalarSource) {
        // 复杂流程：重命名旧列为临时列 → 创建 link 新列 → 按值 lookup 迁移 → 删除临时列
        return yield* buildScalarToLinkMigrationStatements(params, oldField, newLinkField);
      }
    }

    // 场景 4: 常规类型转换（text → numeric, date → text 等）
    return yield* buildStandardTypeConversionStatements(params, oldField, newField);
  });
}
```

### 8.4 三条调用链对比

| 维度 | 元数据仓储（ITableRepository） | 记录仓储（ITableRecordRepository） | 结构仓储（ITableSchemaRepository） |
|------|----------|--------|--------|
| **接口方法** | `insert`, `findOne`, `find`, `updateOne`, `delete` | `insert`, `insertMany`, `update`, `delete`, `find` | `insert`, `update`, `delete` |
| **Scope** | `'meta'` | `'data'` | `'data'`（支持 meta scope 语句） |
| **操作类型** | 系统表 DML（table_meta, field, view） | 业务表 DML（记录增删改查） | 业务表 DDL（CREATE/ALTER/DROP TABLE） |
| **核心 Visitor** | `TableWhereVisitor` <br> `TableMetaUpdateVisitor` | `TableRecordConditionWhereVisitor` <br> `FieldInsertValueVisitor` | `TableSchemaUpdateVisitor` <br> `PostgresTableSchemaFieldCreateVisitor` <br> `FieldTypeConversionVisitor` |
| **核心外部函数** | `executeCompiledQueries` | `RecordInsertBuilder.buildInsertData` <br> `RecordInsertBuilder.executeStatements` | `generateFieldConversionStatements` <br> `executeScopedTableSchemaStatements` |
| **双分派节点** | `spec.accept(visitor)` → `visitor.visitXxx(spec)` | `field.accept(visitor)` → `visitor.visitXxxField(field)` | `mutateSpec.accept(visitor)` → `visitor.visitTableUpdateXxx(spec)` |
| **事务范围** | 与其他元数据操作共享 | 支持跨记录批量 | 按 scope 分批执行，支持 meta/data 切换 |
| **版本管理** | field/view version 递增（乐观锁） | record __version 乐观锁 | 通过 mutation_snapshot 捕获 undo |
| **错误处理** | 数据库错误包装为 DomainError | 基础设施错误自动重试（最多3次）<br>基于 `infrastructure` 标签 + 错误消息匹配 | Unique/NotNull 违规捕获为领域验证错误 |
| **返回类型** | `Result<TableUpdatePersistResult \| void>` | `Result<RecordMutationResult>` | `Result<Table>` |

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

### 9.3 DI 注册差异（可复核）

#### 9.3.1 pg 驱动注册（node-postgres）

```typescript
// adapter-db-postgres-pg/src/di/register.ts:9-57
// 【核心注册函数】支持 meta/data 分别注册
const registerDb = async (
  c: DependencyContainer,
  rawConfig: Partial<IV2PostgresDbConfig>,
  target: 'all' | 'meta' | 'data'  // 注册目标：全部 / 仅元数据 / 仅业务数据
): Promise<DependencyContainer> => {
  // 步骤 1: 配置校验（Zod Schema - 使用 safeParse 而非 parse）
  const parsed = v2PostgresDbConfigSchema.safeParse(rawConfig);
  if (!parsed.success) {
    throw new Error('Invalid v2 postgres db config');
  }
  const config = parsed.data;

  // 步骤 2: 创建数据库连接（node-postgres Pool）
  const db = await createV2PostgresDb(config);

  // 步骤 3: 根据 target 注册到对应的 DI token
  if (target === 'all' || target === 'meta') {
    c.registerInstance(v2MetaDbTokens.db, db);
    c.registerInstance(v2MetaDbTokens.config, config);
  }
  if (target === 'all' || target === 'data') {
    c.registerInstance(v2DataDbTokens.db, db);
    c.registerInstance(v2DataDbTokens.config, config);
  }
  if (target === 'all') {
    c.registerInstance(v2PostgresDbTokens.db, db);
    c.registerInstance(v2PostgresDbTokens.config, config);
  }

  return c;
};

// 【导出的三个注册入口】
// 1. 注册到所有 token（单数据库场景）
export const registerV2PostgresDb = async (
  c: DependencyContainer = container,
  rawConfig: Partial<IV2PostgresDbConfig> = {}
): Promise<DependencyContainer> => registerDb(c, rawConfig, 'all');

// 2. 仅注册 meta 数据库（双数据库场景，先调用）
export const registerV2PostgresMetaDb = async (
  c: DependencyContainer = container,
  rawConfig: Partial<IV2PostgresDbConfig> = {}
): Promise<DependencyContainer> => registerDb(c, rawConfig, 'meta');

// 3. 仅注册 data 数据库（双数据库场景，后调用）
export const registerV2PostgresDataDb = async (
  c: DependencyContainer = container,
  rawConfig: Partial<IV2PostgresDbConfig> = {}
): Promise<DependencyContainer> => registerDb(c, rawConfig, 'data');
```

**pg 驱动注册流程（可复核）**：
```
双数据库场景注册序列：
  1. registerV2PostgresMetaDb(c, metaConfig)
     → 创建 Pool1（连接 meta 数据库）
     → 注册到 v2MetaDbTokens.db, v2MetaDbTokens.config
     
  2. registerV2PostgresDataDb(c, dataConfig)
     → 创建 Pool2（连接 data 数据库）
     → 注册到 v2DataDbTokens.db, v2DataDbTokens.config
     
  3. UnitOfWork 初始化时检测：
     metaConfig.connectionString !== dataConfig.connectionString
     → usesSinglePhysicalDatabase() = false
     → meta 和 data scope 使用独立事务
```

#### 9.3.2 pglite 驱动注册

```typescript
// adapter-db-postgres-pglite/src/di/register.ts:13-33
export const registerV2PostgresPgliteDb = async (
  c: DependencyContainer = container,
  rawConfig: Partial<IV2PostgresDbConfig> = {}
): Promise<DependencyContainer> => {
  // 步骤 1: 配置校验（使用 safeParse 而非 parse）
  const parsed = v2PostgresDbConfigSchema.safeParse(rawConfig);
  if (!parsed.success) {
    throw new Error('Invalid v2 postgres db config');
  }
  const config = parsed.data;

  // 步骤 2: 创建 PGlite 数据库（内存或文件）
  // connectionString 被解释为数据目录：
  //   "memory://" → 内存数据库
  //   "./data/my.db" → 本地文件数据库
  const db = await createV2PostgresPgliteDb(config);

  // 步骤 3: 总是注册到所有 token（单实例）
  c.registerInstance(v2MetaDbTokens.db, db);
  c.registerInstance(v2MetaDbTokens.config, config);
  c.registerInstance(v2DataDbTokens.db, db);
  c.registerInstance(v2DataDbTokens.config, config);
  c.registerInstance(v2PostgresDbTokens.db, db);
  c.registerInstance(v2PostgresDbTokens.config, config);

  return c;
};
```

#### 9.3.3 postgresjs 驱动注册

```typescript
// adapter-db-postgres-postgresjs/src/di/register.ts:13-30
export const registerV2PostgresJsDb = async (
  c: DependencyContainer = container,
  rawConfig: Partial<IV2PostgresDbConfig> = {}
): Promise<DependencyContainer> => {
  // 步骤 1: 配置校验（使用 safeParse 而非 parse）
  const parsed = v2PostgresDbConfigSchema.safeParse(rawConfig);
  if (!parsed.success) {
    throw new Error('Invalid v2 postgres db config');
  }
  const config = parsed.data;

  // 步骤 2: 创建 Postgres.js 连接
  const db = createV2PostgresJsDb(config);

  // 步骤 3: 总是注册到所有 token（单实例）
  c.registerInstance(v2MetaDbTokens.db, db);
  c.registerInstance(v2MetaDbTokens.config, config);
  c.registerInstance(v2DataDbTokens.db, db);
  c.registerInstance(v2DataDbTokens.config, config);
  c.registerInstance(v2PostgresDbTokens.db, db);
  c.registerInstance(v2PostgresDbTokens.config, config);

  return c;
};
```

**pglite / postgresjs 注册特点（可复核）**：
- ✅ 总是使用单一数据库实例
- ✅ meta 和 data scope 共享同一连接
- ❌ 不支持双数据库架构
- ✅ `usesSinglePhysicalDatabase()` 恒返回 `true`
- ✅ 兄弟 scope 事务天然复用

### 9.4 事务上下文切换差异

#### 9.4.1 共享的 UnitOfWork 实现（可复核）

三种驱动共享相同的 UnitOfWork 实现（来自 `adapter-db-postgres-shared`），这是跨驱动适配的核心：

```typescript
// adapter-db-postgres-shared/src/unitOfWork.ts:64-81
// 【关键适配函数 1】获取当前 scope 的事务
export const getPostgresTransaction = <DB>(
  context?: IExecutionContext,
  scope: UnitOfWorkScope = 'data'
): Transaction<DB> | null => {
  // 步骤 1: 从 context 中获取指定 scope 的事务对象
  const transaction = getUnitOfWorkTransaction(context, scope);
  
  // 步骤 2: 类型检查 - 只接受 PostgresUnitOfWorkTransaction
  if (transaction instanceof PostgresUnitOfWorkTransaction) {
    return transaction.db as Transaction<DB>;  // 返回 Kysely Transaction 对象
  }
  return null;  // 没有事务或类型不匹配
};

// 【关键适配函数 2】解析应该使用 db 还是 tx
export const resolvePostgresDbOrTx = <DB>(
  db: Kysely<DB>,
  context?: IExecutionContext,
  scope: UnitOfWorkScope = 'data'
): Kysely<DB> | Transaction<DB> => {
  // 策略：有事务用事务，无事务用 db
  return getPostgresTransaction<DB>(context, scope) ?? db;
};
```

**UnitOfWork 事务上下文切换流程（可复核，与源码完全一致）**：

```
应用层调用 unitOfWork.withTransaction(context, work, { scope: 'data' })
         │
         ▼  【步骤 1】检查是否已有同 scope 事务
    existingTx = getUnitOfWorkTransaction(context, scope)
    if (existingTx) → 直接复用，执行 work(activateUnitOfWorkScope(context, scope))
         │
         ▼  【步骤 2】检查是否可以复用兄弟 scope 事务
    if (usesSinglePhysicalDatabase()) {
        // meta 和 data 是同一物理数据库
        siblingTx = getUnitOfWorkTransaction(context, siblingScope)
        if (siblingTx) {
            // 创建新的 context，让当前 scope 指向兄弟 scope 的事务对象
            sharedContext = {
                ...context,
                transaction: siblingTx,        // ⚠️  同步顶层 transaction 字段
                transactions: {
                    ...(context.transactions ?? {}),
                    ...(siblingTx.scope ? { [siblingTx.scope]: siblingTx } : {}),
                    [scope]: siblingTx  // 指向同一事务对象
                }
            }
            → 执行 work(sharedContext)
        }
    }
         │
         ▼  【步骤 3】新建事务
    db = (scope === 'meta') ? this.metaDb : this.dataDb
    const maxRetries = 3;  // ⚠️  定义在方法内部
    let attempt = 0;
    while (true) {
        let transaction: PostgresUnitOfWorkTransaction<DB> | undefined;
        try {
            const transactionResult = await db.transaction().execute(async (trx) => {
                // 创建 PostgresUnitOfWorkTransaction 包装
                transaction = new PostgresUnitOfWorkTransaction(trx, scope)
                transactionContext = bindUnitOfWorkTransaction(context, transaction)
                
                // 执行业务逻辑
                workResult = await work(transactionContext)
                if (workResult.isErr()) {
                    // 抛出自定义异常触发回滚
                    throw new UnitOfWorkAbort(workResult.error)
                }
                return { workResult, transaction }
            })
            
            // 事务提交后执行 afterCommit 钩子
            await transactionResult.transaction.runAfterCommitHandlers()
            return transactionResult.workResult
        } catch (error) {
            // 【步骤 4】死锁/序列化失败自动重试
            if (error instanceof UnitOfWorkAbort) {
                // ⚠️  注意：传入的是 error.error（DomainError），不是 error
                if (attempt < maxRetries && isRetryableTransactionAbort(error.error)) {
                    const delayMs = backoffMs(attempt);  // 5/10/20ms + [0,9]ms 抖动
                    attempt += 1;
                    await sleep(delayMs);
                    continue;  // 重试
                }
                // 回滚后执行 afterRollback 钩子
                await transaction?.runAfterRollbackHandlers();
                return err(error.error);
            }
            // 非 UnitOfWorkAbort 的异常（如 Kysely 内部错误）
            await transaction?.runAfterRollbackHandlers();
            return err(domainError.unexpected({
                message: `Unexpected unit of work error: ${describeError(error)}`
            }));
        }
    }
```

**重试策略（可复核，与源码完全一致）**：

```typescript
// adapter-db-postgres-shared/src/unitOfWork.ts:148-150
// ⚠️  maxRetries 定义在 withTransaction 方法内部，不是外部常量
const maxRetries = 3;
let attempt = 0;

// adapter-db-postgres-shared/src/unitOfWork.ts:204-212
// ⚠️  重试判定：检查 DomainError 的 'infrastructure' 标签 + 错误消息匹配
const isRetryableTransactionAbort = (error: DomainError): boolean => {
  // 步骤 1: 必须包含 'infrastructure' 标签
  if (!error.tags.includes('infrastructure')) return false;

  // 步骤 2: 错误消息必须包含以下关键词之一（大小写不敏感）
  const message = error.message.toLowerCase();
  return (
    message.includes('deadlock detected') ||          // 死锁
    message.includes('could not serialize access') ||  // 无法序列化访问
    message.includes('serialization failure')          // 序列化失败
  );
};

// adapter-db-postgres-shared/src/unitOfWork.ts:198-202
// ⚠️  backoff 抖动范围是 [0, 9]ms，不是 [0, 5]ms
const backoffMs = (attempt: number): number => {
  // 指数退避：attempt 0 → 5ms, attempt 1 → 10ms, attempt 2 → 20ms
  const base = 5 * 2 ** attempt;
  // 随机抖动：Math.floor(Math.random() * 10) → [0, 9]ms
  const jitter = Math.floor(Math.random() * 10);
  return base + jitter;
};
```

#### 9.4.2 单数据库 vs 双数据库事务

| 场景 | pg 驱动（双数据库） | pg 驱动（单数据库） | pglite/postgresjs |
|------|-------------------|-------------------|-----------------|
| meta scope 事务 | 使用 metaDb 连接 | 复用同一连接 | 复用同一连接 |
| data scope 事务 | 使用 dataDb 连接 | 复用同一连接 | 复用同一连接 |
| 跨 scope 事务 | 检测到同一连接字符串时自动复用 | 天然复用 | 天然复用 |
| 分布式事务 | 不支持（需要分别管理） | 单事务 | 单事务 |

**事务复用逻辑（可复核，与源码完全一致）**：

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
  if (!siblingTransaction) {
    return null;
  }

  // ⚠️  真实返回结构包含顶层 transaction 字段和 transactions 中的双 scope 绑定
  return {
    ...context,
    transaction: siblingTransaction,  // 同步顶层 transaction 字段
    transactions: {
      ...(context.transactions ?? {}),
      ...(siblingTransaction.scope
        ? { [siblingTransaction.scope]: siblingTransaction }
        : {}),
      [scope]: siblingTransaction,  // 指向同一事务对象
    },
  };
}
```

#### 9.4.3 事务执行流程（可复核，与源码完全一致）

```typescript
// adapter-db-postgres-shared/src/unitOfWork.ts:128-192
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

  // 3. 新建事务（最多重试 3 次可重试基础设施错误）
  // ⚠️  maxRetries 定义在方法内部，紧挨着 while 循环
  const db = scope === 'meta' ? this.metaDb : this.dataDb;
  const maxRetries = 3;
  let attempt = 0;

  // Retry only for top-level transactions, and only for retryable infra failures.
  // Nested transactions must not retry because they share an outer transaction scope.
  // Keep delays tiny because this is often used in request/response paths.
  // eslint-disable-next-line no-constant-condition
  while (true) {
    let transaction: PostgresUnitOfWorkTransaction<DB> | undefined;
    try {
      const transactionResult = await db.transaction().execute(async (trx) => {
        transaction = new PostgresUnitOfWorkTransaction(trx, scope);
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
      // ⚠️  区分 UnitOfWorkAbort（业务异常）和其他异常
      if (error instanceof UnitOfWorkAbort) {
        // ⚠️  重试条件：attempt < maxRetries 且是可重试的基础设施错误
        if (attempt < maxRetries && isRetryableTransactionAbort(error.error)) {
          const delayMs = backoffMs(attempt);  // 5ms, 10ms, 20ms + [0,9]ms 抖动
          attempt += 1;
          await sleep(delayMs);
          continue;
        }
        // ⚠️  非重试场景：先执行回滚钩子，再返回错误
        await transaction?.runAfterRollbackHandlers();
        return err(error.error);
      }
      // ⚠️  非 UnitOfWorkAbort 的异常（如 Kysely 内部错误）
      await transaction?.runAfterRollbackHandlers();
      return err(
        domainError.unexpected({
          message: `Unexpected unit of work error: ${describeError(error)}`,
        })
      );
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
| **基础设施错误自动重试** | ✅（共享实现）<br>基于 DomainError `infrastructure` 标签 + 错误消息匹配<br>（死锁/序列化失败最多重试 3 次） | ✅（共享实现） | ✅（共享实现） |
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

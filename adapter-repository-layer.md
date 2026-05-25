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

## 七、关键设计模式

1. **依赖倒置原则（DIP）**：高层模块依赖抽象，不依赖具体
2. **端口与适配器模式**：清晰分离内部与外部
3. **规约模式（Specification）**：查询和更新
4. **访问者模式（Visitor）**：
5. **工作单元模式（Unit of Work）**：事务管理
6. **依赖注入（DI）**：运行时绑定实现

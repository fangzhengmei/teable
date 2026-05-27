# View Filter 与 SQL 构造代码链路分析

## 概述

Teable 的 View Filter 系统采用 **Specification 模式 + Visitor 模式** 的双层架构，实现了从前端过滤条件到数据库 SQL 查询的完整转换链路。整个流程分为三个核心阶段：

1. **条件树解析** - 将前端 JSON 过滤条件解析为领域对象 Specification
2. **字段映射** - 将逻辑字段 ID 映射为数据库列名
3. **SQL 拼装** - 通过 Visitor 模式将 Specification 转换为可执行的 SQL WHERE 子句

---

## 第一阶段：条件树解析 (Condition Tree Parsing)

### 1.1 过滤条件 DTO 定义

**文件**: `packages/v2/core/src/queries/RecordFilterDto.ts`

前端传入的过滤条件是一个递归的树形结构，支持三种节点类型：

```typescript
type RecordFilterNode = 
  | RecordFilterCondition   // 叶子节点：具体条件
  | RecordFilterGroup       // 组合节点：AND/OR 组
  | RecordFilterNot;        // 否定节点：NOT 操作

// 叶子节点示例
{
  fieldId: "fld_xxx",
  operator: "is" | "contains" | "isGreater" | ...,
  value: string | number | boolean | Array | DateValue | FieldReference
}

// 组合节点示例
{
  conjunction: "and" | "or",
  items: [RecordFilterNode, RecordFilterNode, ...]
}

// 否定节点示例
{
  not: RecordFilterNode
}
```

**关键 Schema**:
- `recordFilterConditionSchema` - 叶子条件验证
- `recordFilterGroupSchema` - 组合条件验证（递归定义）
- `recordFilterNotSchema` - 否定条件验证

### 1.2 DTO 到 Specification 的转换

**文件**: `packages/v2/core/src/queries/RecordFilterMapper.ts`

核心函数 `buildRecordConditionSpec` 通过递归遍历将 DTO 树转换为 Specification 树：

```typescript
// 递归构建 Specification
const buildSpecFromNode = (table: Table, node: RecordFilterNode) => {
  if (isRecordFilterCondition(node)) {
    // 叶子节点：创建字段特定的 ConditionSpec
    return resolveField(table, node.fieldId)
      .andThen(field => buildConditionValue(table, node.value))
      .andThen(value => field.spec().create({ operator: node.operator, value }));
  }
  
  if (isRecordFilterNot(node)) {
    // 否定节点：包装 NotSpec
    return buildSpecFromNode(table, node.not)
      .andThen(spec => notSpec(spec));
  }
  
  if (isRecordFilterGroup(node)) {
    // 组合节点：使用 SpecBuilder 组装
    const builder = RecordConditionSpecBuilder.create(node.conjunction);
    for (const item of node.items) {
      builder.addConditionSpec(buildSpecFromNode(table, item).value);
    }
    return builder.build();
  }
};
```

### 1.3 Specification 构建器

**文件**: `packages/v2/core/src/domain/table/records/specs/RecordConditionSpecBuilder.ts`

`RecordConditionSpecBuilder` 继承自 `SpecBuilder`，负责将多个条件组合：

```
SpecBuilder (抽象基类)
  ├── specs: 存储子 Specification 数组
  ├── mode: 'and' | 'or'
  ├── addSpec() / addNotSpec() / addGroup()
  └── build(): 将 specs 组合为 AndSpec 或 OrSpec

RecordConditionSpecBuilder (具体实现)
  ├── addCondition({ field, operator, value })
  ├── addConditionSpec(spec)
  ├── recordId(recordId)
  └── andGroup() / orGroup() / not()
```

---

## 第二阶段：字段映射 (Field Mapping)

### 2.1 字段 ID 到数据库列名的映射

**文件**: `packages/v2/adapter-table-repository-postgres/src/record/query-builder/FieldOutputColumnVisitor.ts`

`FieldOutputColumnVisitor` 实现 `IFieldVisitor` 接口，负责收集字段到列的映射：

```typescript
type FieldOutputColumn = {
  fieldId: FieldId;
  columnAlias: string;  // 数据库列名
  valueKind?: 'user';   // 标记用户类型字段
};

class FieldOutputColumnVisitor {
  collect(table: Table, projection?: FieldId[]): FieldOutputColumn[] {
    for (const field of table.getFields()) {
      field.accept(this);  // 访问者模式分派
    }
  }
  
  // 每个字段类型的处理（实际都调用 getColumnAlias）
  visitSingleLineTextField(field): FieldOutputColumn {
    return this.addColumn(field);
  }
  
  private getColumnAlias(field: Field): string {
    // 从 Field.dbFieldName() 获取真实数据库列名
    return field.dbFieldName().value();
  }
}
```

### 2.2 列名解析辅助函数

**文件**: `packages/v2/adapter-table-repository-postgres/src/record/visitors/TableRecordConditionWhereVisitor.ts:269-282`

```typescript
const resolveColumn = (field: core.Field, tableAlias?: string): Result<string, DomainError> => {
  return safeTry<string, DomainError>(function* () {
    const dbFieldName = yield* field.dbFieldName();
    const column = yield* dbFieldName.value();
    // 可选：添加表别名前缀，用于 JOIN 查询
    return ok(tableAlias ? `${tableAlias}.${column}` : column);
  });
};
```

---

## 第三阶段：SQL 拼装 (SQL Construction)

### 3.1 Visitor 模式的核心接口

**文件**: `packages/v2/core/src/domain/table/records/specs/ITableRecordConditionSpecVisitor.ts`

定义了超过 200 个 visit 方法，每个字段类型 + 每个操作符组合一个方法：

```typescript
interface ITableRecordConditionSpecVisitor<TResult = unknown> {
  // 基础记录条件
  visitRecordById(spec: RecordByIdSpec): Result<TResult, DomainError>;
  visitRecordByIds(spec: RecordByIdsSpec): Result<TResult, DomainError>;
  
  // 单行文本字段条件
  visitSingleLineTextIs(spec: SingleLineTextConditionSpec): Result<TResult, DomainError>;
  visitSingleLineTextIsNot(spec: SingleLineTextConditionSpec): Result<TResult, DomainError>;
  visitSingleLineTextContains(spec: SingleLineTextConditionSpec): Result<TResult, DomainError>;
  visitSingleLineTextDoesNotContain(...): Result<TResult, DomainError>;
  visitSingleLineTextIsEmpty(...): Result<TResult, DomainError>;
  visitSingleLineTextIsNotEmpty(...): Result<TResult, DomainError>;
  
  // 数字字段条件 (6 个操作符)
  visitNumberIs / visitNumberIsNot / visitNumberIsGreater
  visitNumberIsGreaterEqual / visitNumberIsLess / visitNumberIsLessEqual
  
  // 日期字段条件 (8 个操作符)
  visitDateIs / visitDateIsNot / visitDateIsWithIn
  visitDateIsBefore / visitDateIsAfter
  visitDateIsOnOrBefore / visitDateIsOnOrAfter
  
  // ... 其他字段类型 (多选、用户、链接、公式、汇总等)
  // 总计约 200+ 个 visit 方法
}
```

### 3.2 SQL WHERE 条件访问者实现

**文件**: `packages/v2/adapter-table-repository-postgres/src/record/visitors/TableRecordConditionWhereVisitor.ts`

`TableRecordConditionWhereVisitor` 是最核心的 SQL 生成器，实现了上述接口的所有方法，将每个 Specification 转换为 `kysely` 的 SQL 表达式。

#### 核心结构：

```typescript
class TableRecordConditionWhereVisitor
  extends AbstractSpecFilterVisitor<RecordConditionWhere>
  implements ITableRecordConditionSpecVisitor<RecordConditionWhere> {
  
  constructor(options?: { tableAlias?: string; hostTableAlias?: string }) {
    super();
  }
  
  // 逻辑组合
  and(left: SqlExpr, right: SqlExpr): SqlExpr {
    return sql`(${left}) and (${right})`;
  }
  or(left: SqlExpr, right: SqlExpr): SqlExpr {
    return sql`(${left}) or (${right})`;
  }
  not(inner: SqlExpr): SqlExpr {
    return sql`not (${inner})`;
  }
  
  // 每个 visitXxx 方法调用对应的辅助构建函数
  visitSingleLineTextIs(spec: SingleLineTextConditionSpec) {
    return this.applyIs(spec.field(), spec.value());
  }
}
```

#### 关键构建函数：

**1. `buildIsCondition` - 等值比较** (`TableRecordConditionWhereVisitor.ts:695-826`)

处理 `is` 操作符，针对不同字段类型生成不同 SQL：

```typescript
// 简单字符串
sql`${columnRef} = ${literalValue}`

// 用户/链接类型（JSON 存储）
sql`jsonb_extract_path_text(to_jsonb(${columnRef}), 'id') = ${rightLiteral}`

// 多值字段（数组）
sql`EXISTS (
  SELECT 1 FROM jsonb_array_elements_text(${normalizedArray}) AS elem
  WHERE elem = ${value}
)`

// 字段引用比较（字段 A = 字段 B）
// 需要类型路由：userOrLinkIds / linkTitle / date / json / generic
```

**2. `buildContainsCondition` - 包含匹配** (`TableRecordConditionWhereVisitor.ts:968-1019`)

```typescript
// 简单字符串 ILIKE
sql`${columnRef} ilike '%${escapedValue}%' escape '\\'`

// JSON/数组类型
sql`jsonb_path_exists(${target}, '$[*] ? (@ like_regex "${escapedValue}" flag "i")'::jsonpath)`
```

**3. `buildNumericComparisonCondition` - 数值比较** (`TableRecordConditionWhereVisitor.ts:1021-1078`)

```typescript
// 简单数值
sql`${columnRef} > ${value}`

// 数组内元素比较
sql`EXISTS (
  SELECT 1 FROM jsonb_array_elements_text(${normalizedArray}) AS elem
  WHERE NULLIF(REGEXP_REPLACE(elem, '[^0-9.+-]', '', 'g'), '')::double precision > ${value}
)`
```

**4. `buildDateComparisonCondition` - 日期比较** (`TableRecordConditionWhereVisitor.ts:1080-1182`)

```typescript
// 处理时区转换
const compareAsDateOnly = shouldCompareAsDateOnly(field);
const leftExpr = compareAsDateOnly 
  ? sql`(${columnRef} AT TIME ZONE ${timeZone})::date`
  : columnRef;
sql`${leftExpr} ${operator} ${right}`
```

**5. `buildListCondition` - 列表操作（any/none/all/exact）** (`TableRecordConditionWhereVisitor.ts:1211-1327`)

```typescript
// IN 查询
sql`${columnRef} in (${valueList})`

// JSON 数组操作符
sql`${jsonbColumn} ?| ${textArray}`    // any
sql`${jsonbColumn} ?& ${textArray}`    // all
sql`${jsonbColumn} @> ${jsonbArray}`   // contains
```

**6. `resolveDateRange` - 日期范围解析** (`TableRecordConditionWhereVisitor.ts:473-639`)

支持多种日期模式：
- `today` / `tomorrow` / `yesterday`
- `oneWeekAgo` / `oneMonthAgo`
- `daysAgo` / `daysFromNow` (需要 numberOfDays)
- `exactDate` / `exactFormatDate`
- `currentWeek` / `currentMonth` / `currentYear`
- `lastWeek` / `nextWeekPeriod` 等

### 3.3 WHERE 子句构建入口

**文件**: `packages/v2/adapter-table-repository-postgres/src/record/repository/buildRecordWhereClause.ts`

```typescript
const buildRecordWhereClause = (
  spec: ISpecification<TableRecord, ITableRecordConditionSpecVisitor>,
  options?: { tableAlias?: string }
): Result<Expression<SqlBool> | null, DomainError> => {
  const visitor = new TableRecordConditionWhereVisitor(options);
  const acceptResult = spec.accept(visitor);  // 触发访问者模式
  return visitor.where();  // 获取最终 SQL 表达式
};
```

### 3.4 搜索条件构建

**文件**: `packages/v2/adapter-table-repository-postgres/src/record/repository/RecordSearchWhereBuilder.ts`

全局搜索（跨字段搜索）的 SQL 构建：

```typescript
buildRecordSearchWhereClause(table, recordSearch, { tableAlias: 't' })
  // 为每个字段生成搜索条件，然后 OR 组合
  // 不同字段类型有不同的搜索策略：
  // - 结构化字段 (user/link/attachment): 匹配 title
  // - 数字: ROUND 后字符串匹配
  // - 日期: 解析为日期范围查询
  // - 长文本: 去除换行符后匹配
  // - 多值字段: 聚合数组元素后匹配
```

---

## 完整调用链

### 查询记录的完整流程

```
ListTableRecordsHandler.execute()
  │
  ├─ 解析 ViewQueryDefaults (View 级默认过滤)
  │   └─ ViewQueryDefaults.filter()
  │
  ├─ 解析查询级过滤条件
  │   └─ resolveFilterFieldKeys() → 字段键解析
  │
  ├─ 合并过滤条件
  │   └─ ViewQueryDefaults.merge()
  │
  ├─ 构建 Specification
  │   └─ buildRecordConditionSpec(table, filter)
  │       └─ RecordFilterMapper.ts (递归 DTO → Spec)
  │
  ├─ 调用 Repository
  │   └─ PostgresTableRecordQueryRepository.find()
  │       │
  │       ├─ 创建 QueryBuilder
  │       │   └─ queryBuilderManager.createBuilder()
  │       │
  │       ├─ 应用过滤
  │       │   └─ queryBuilder.where(spec)
  │       │
  │       ├─ 构建 SQL
  │       │   └─ queryBuilder.build()
  │       │       │
  │       │       └─ buildRecordWhereClause(spec, { tableAlias: 't' })
  │       │           └─ TableRecordConditionWhereVisitor
  │       │               └─ spec.accept(visitor)  →  SQL Expr
  │       │
  │       └─ 执行查询
  │           └─ kysely.executeQuery()
  │
  └─ 返回结果
      └─ ListTableRecordsResult.create()
```

---

## 关键设计模式

### 1. Specification 模式

**目的**: 将业务规则（过滤条件）封装为可组合的对象

```
ISpecification
  ├─ isSatisfiedBy(t)  // 内存校验
  ├─ mutate(t)         // 变更操作
  └─ accept(visitor)   // 访问者接受

AndSpec / OrSpec / NotSpec  (组合子)
SingleLineTextConditionSpec / NumberConditionSpec / ...  (叶子)
```

### 2. Visitor 模式

**目的**: 分离算法（SQL 生成）与数据结构（Specification 树）

```
ISpecification.accept(visitor)
  → 分派到具体的 visitXxx 方法

ITableRecordConditionSpecVisitor
  ├─ TableRecordConditionWhereVisitor  (生成 SQL WHERE)
  ├─ NoopRecordConditionSpecVisitor    (空操作，用于测试)
  └─ ... (未来可扩展: 内存校验、MongoDB 查询生成等)
```

### 3. Builder 模式

**目的**: 提供流畅的 API 构建复杂条件树

```
RecordConditionSpecBuilder.create('and')
  .addCondition({ field, operator: 'is', value })
  .orGroup(b => b.addCondition(...).addCondition(...))
  .not(b => b.addCondition(...))
  .build()
```

---

## 注意事项

### 1. NULL 值处理

- `isEmpty` 条件需要同时匹配 `IS NULL` 和空值（空字符串/空数组）
- `isNot` 使用 `IS DISTINCT FROM` 而非 `!=` 以正确处理 NULL
- 否定条件（NOT IN、NOT LIKE）需要 `COALESCE` 确保 NULL 行通过

### 2. JSON/数组字段

- PostgreSQL `jsonb` 类型需要特殊处理
- 使用 `jsonb_array_elements_text` 展开数组进行元素级查询
- `jsonb_path_exists` 用于正则匹配查询

### 3. 字段引用比较

支持 `字段 A = 字段 B` 的比较方式，需要：
- 类型兼容性检查
- 结构化类型特殊路由（用户/链接按 ID 比较）
- 日期时区统一转换

### 4. 性能考虑

- 复杂条件可能生成深层嵌套子查询
- 多值字段过滤使用 `EXISTS` 而非数组操作符
- 日期范围查询优先使用 BETWEEN 以利用索引

---

## 相关文件速查

| 模块 | 文件路径 |
|------|---------|
| DTO 定义 | `packages/v2/core/src/queries/RecordFilterDto.ts` |
| DTO → Spec 映射 | `packages/v2/core/src/queries/RecordFilterMapper.ts` |
| Spec 构建器 | `packages/v2/core/src/domain/table/records/specs/RecordConditionSpecBuilder.ts` |
| 访问者接口 | `packages/v2/core/src/domain/table/records/specs/ITableRecordConditionSpecVisitor.ts` |
| SQL WHERE 生成 | `packages/v2/adapter-table-repository-postgres/src/record/visitors/TableRecordConditionWhereVisitor.ts` |
| 字段列映射 | `packages/v2/adapter-table-repository-postgres/src/record/query-builder/FieldOutputColumnVisitor.ts` |
| 查询仓库 | `packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordQueryRepository.ts` |
| 列表查询处理器 | `packages/v2/core/src/queries/ListTableRecordsHandler.ts` |
| View 查询默认值 | `packages/v2/core/src/domain/table/views/ViewQueryDefaults.ts` |
| 字段条件值对象 | `packages/v2/core/src/domain/table/fields/types/FieldCondition.ts` |

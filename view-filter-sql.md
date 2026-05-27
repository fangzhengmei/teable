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

---

## 附录：完整样例追踪

以下通过一个**真实的复合过滤条件**，逐步追踪从前端 JSON 到最终 SQL 的完整转换过程。

---

### A.1 样例场景

假设我们有一个任务管理表，字段配置如下：

| 字段名 | 字段类型 | 字段 ID | 数据库列名 |
|--------|---------|---------|-----------|
| Name | 单行文本 | `fld_name` | `col_name` |
| Status | 单选 | `fld_status` | `col_status` |
| Priority | 单选 | `fld_priority` | `col_priority` |
| Due Date | 日期 | `fld_due` | `col_due_date` |
| Assignee | 用户(单选) | `fld_assignee` | `col_assignee` |
| Score | 数字 | `fld_score` | `col_score` |

**业务需求**：查询 **高优先级且未完成** 的任务，满足以下任一条件：
- 名称包含 "bug" **并且** 负责人是 "张三"
- **或者** 截止日期在本周内 **并且** 分数 > 80

---

### A.2 第一阶段：JSON 过滤条件 (前端输入)

**前端传入的完整 Filter JSON**：

```json
{
  "conjunction": "and",
  "items": [
    {
      "fieldId": "fld_priority",
      "operator": "is",
      "value": "High"
    },
    {
      "fieldId": "fld_status",
      "operator": "isNot",
      "value": "Done"
    },
    {
      "conjunction": "or",
      "items": [
        {
          "conjunction": "and",
          "items": [
            {
              "fieldId": "fld_name",
              "operator": "contains",
              "value": "bug"
            },
            {
              "fieldId": "fld_assignee",
              "operator": "is",
              "value": { "type": "user", "id": "usr_zhangsan", "title": "张三" }
            }
          ]
        },
        {
          "conjunction": "and",
          "items": [
            {
              "fieldId": "fld_due",
              "operator": "isWithin",
              "value": { "mode": "currentWeek", "timeZone": "Asia/Shanghai" }
            },
            {
              "fieldId": "fld_score",
              "operator": "isGreater",
              "value": 80
            }
          ]
        }
      ]
    }
  ]
}
```

**条件树结构**：

```
AND (根节点)
  ├─ Priority = "High"           (叶子 1)
  ├─ Status ≠ "Done"             (叶子 2)
  └─ OR
      ├─ AND (分支 A)
      │   ├─ Name contains "bug"    (叶子 3)
      │   └─ Assignee = "张三"      (叶子 4)
      └─ AND (分支 B)
          ├─ Due Date is 本周内     (叶子 5)
          └─ Score > 80             (叶子 6)
```

---

### A.3 第二阶段：JSON → Specification 递归转换

**文件**: `packages/v2/core/src/queries/RecordFilterMapper.ts`

转换过程通过 `buildRecordConditionSpec` 函数递归执行：

#### 步骤 1: 解析根节点 AND 组

```typescript
// 入口: buildRecordConditionSpec(table, rootFilter)
const builder = RecordConditionSpecBuilder.create('and');

// 遍历 items:
// item[0]: Priority = "High"
builder.addConditionSpec(
  SingleSelectConditionSpec.create(priorityField, 'is', LiteralValue("High"))
);

// item[1]: Status ≠ "Done"
builder.addConditionSpec(
  SingleSelectConditionSpec.create(statusField, 'isNot', LiteralValue("Done"))
);

// item[2]: OR 组 → 递归构建子 Specification
const orGroupBuilder = RecordConditionSpecBuilder.create('or');
// ... 递归处理 OR 组的 items
builder.addConditionSpec(orGroupResult.value);

// 最终返回: AndSpec(AndSpec(Spec1, Spec2), OrSpec(SpecA, SpecB))
```

#### 步骤 2: 递归处理 OR 组内的 AND 分支

```typescript
// 分支 A: (Name contains "bug" AND Assignee = "张三")
const branchA = RecordConditionSpecBuilder.create('and');
branchA.addConditionSpec(
  SingleLineTextConditionSpec.create(nameField, 'contains', LiteralValue("bug"))
);
branchA.addConditionSpec(
  UserConditionSpec.create(assigneeField, 'is', UserValue({ id: "usr_zhangsan" }))
);

// 分支 B: (Due Date is 本周内 AND Score > 80)
const branchB = RecordConditionSpecBuilder.create('and');
branchB.addConditionSpec(
  DateConditionSpec.create(dueField, 'isWithin', DateValue({ mode: "currentWeek" }))
);
branchB.addConditionSpec(
  NumberConditionSpec.create(scoreField, 'isGreater', LiteralValue(80))
);

// 组合为 OrSpec
orGroupBuilder.addConditionSpec(branchA.build().value);
orGroupBuilder.addConditionSpec(branchB.build().value);
```

#### 步骤 3: 最终 Specification 树结构

```
AndSpec (根)
  ├─ left: AndSpec
  │     ├─ left: SingleSelectConditionSpec (Priority = "High")
  │     └─ right: SingleSelectConditionSpec (Status ≠ "Done")
  └─ right: OrSpec
        ├─ left: AndSpec (分支 A)
        │     ├─ left: SingleLineTextConditionSpec (Name contains "bug")
        │     └─ right: UserConditionSpec (Assignee = "张三")
        └─ right: AndSpec (分支 B)
              ├─ left: DateConditionSpec (Due Date is 本周内)
              └─ right: NumberConditionSpec (Score > 80)
```

---

### A.4 第三阶段：字段 ID → 数据库列名映射

**文件**: `packages/v2/adapter-table-repository-postgres/src/record/query-builder/FieldOutputColumnVisitor.ts`

在 SQL 生成之前，通过访问者模式收集所有涉及字段的列名：

```typescript
// 伪代码：字段到列的映射表
const fieldToColumnMap = {
  "fld_name":      "col_name",      // 单行文本 → TEXT
  "fld_status":    "col_status",    // 单选 → TEXT
  "fld_priority":  "col_priority",  // 单选 → TEXT
  "fld_due":       "col_due_date",  // 日期 → TIMESTAMPTZ
  "fld_assignee":  "col_assignee",  // 用户 → JSONB
  "fld_score":     "col_score",     // 数字 → DOUBLE PRECISION
};
```

**关键点**：
- 每个字段对象通过 `field.dbFieldName()` 获取存储的列名
- 表别名通过参数传入（通常为 `t`）
- 最终列引用格式：`"t"."col_name"`

---

### A.5 第四阶段：Specification → Kysely SQL 表达式

**文件**: `packages/v2/adapter-table-repository-postgres/src/record/visitors/TableRecordConditionWhereVisitor.ts`

#### 访问者模式的执行流程

```typescript
const visitor = new TableRecordConditionWhereVisitor({ tableAlias: 't' });
spec.accept(visitor);  // 触发递归访问
const sqlExpr = visitor.where().value;
```

**AndSpec.accept() 的实现** (`packages/v2/core/src/domain/shared/specification/AndSpec.ts:30-36`):
```typescript
accept(v: V): Result<void, DomainError> {
  return v
    .visit(this)              // 1. 通知访问者进入 AndSpec
    .andThen(() => this.left.accept(v))   // 2. 递归访问左子树
    .andThen(() => this.right.accept(v))  // 3. 递归访问右子树
    .map(() => undefined);
}
```

**OrSpec.accept() 的特殊实现** (`packages/v2/core/src/domain/shared/specification/OrSpec.ts:33-52`):
```typescript
accept(v: V): Result<void, DomainError> {
  if (isSpecFilterVisitor(v)) {
    const leftVisitor = v.clone();   // 克隆访问者处理左分支
    const rightVisitor = v.clone();  // 克隆访问者处理右分支

    return v.visit(this)
      .andThen(() => this.left.accept(leftVisitor))
      .andThen(() => this.right.accept(rightVisitor))
      .andThen(() => leftVisitor.where())
      .andThen(leftCond => 
        rightVisitor.where().map(rightCond => v.or(leftCond, rightCond))
      )
      .andThen(cond => v.addCond(cond));
  }
  // ...
}
```

> **关键洞察**：OrSpec 需要**克隆两个独立的访问者**来分别处理左右分支，因为每个分支会累积自己的条件状态。而 AndSpec 可以复用同一个访问者，条件会被顺序累积。

---

### A.6 各个叶子节点的 SQL 生成详解

#### 叶子 1: Priority = "High"

```typescript
// visitSingleSelectIs(spec)
// → buildIsCondition(field, value)

// 最终 SQL:
"t"."col_priority" = $1
// 参数: ["High"]
```

**代码路径**：`TableRecordConditionWhereVisitor.ts:2200` → `visitSingleSelectIs()` → `applyIs()` → `buildIsCondition()`

#### 叶子 2: Status ≠ "Done"

```typescript
// visitSingleSelectIsNot(spec)
// → buildIsNotCondition(field, value)

// 注意: 使用 IS DISTINCT FROM 以正确处理 NULL
// 最终 SQL:
"t"."col_status" is distinct from $1
// 参数: ["Done"]
```

**代码路径**：`TableRecordConditionWhereVisitor.ts:2206` → `visitSingleSelectIsNot()` → `buildIsNotCondition()`

#### 叶子 3: Name contains "bug"

```typescript
// visitSingleLineTextContains(spec)
// → buildContainsCondition(field, value)

// 注意: 使用 ILIKE 进行大小写不敏感匹配
// 最终 SQL:
"t"."col_name" ilike $1 escape '\'
// 参数: ["%bug%"]
```

**代码路径**：`TableRecordConditionWhereVisitor.ts:2052` → `visitSingleLineTextContains()` → `buildContainsCondition()`

#### 叶子 4: Assignee = "张三"

```typescript
// visitUserIs(spec)
// → buildIsCondition(field, value, type: 'userOrLinkIds')

// 用户字段存储为 JSONB: { id: "usr_xxx", title: "xxx" }
// 需要从 JSON 中提取 id 进行比较
// 最终 SQL:
jsonb_extract_path_text(to_jsonb("t"."col_assignee"), 'id') = $1
// 参数: ["usr_zhangsan"]
```

**代码路径**：`TableRecordConditionWhereVisitor.ts:2333` → `visitUserIs()` → `applyIs()` → `buildIsCondition(type: 'userOrLinkIds')`

#### 叶子 5: Due Date is 本周内

```typescript
// visitDateIsWithIn(spec)
// → resolveDateRange(mode, timeZone)
// → buildDateBetweenCondition(field, start, end)

// 假设今天是 2026-05-28 (周四)
// 上海时区的本周: 2026-05-26 00:00:00 ~ 2026-06-01 23:59:59.999
// 最终 SQL:
(("t"."col_due_date" AT TIME ZONE $1)::date >= $2 AND ("t"."col_due_date" AT TIME ZONE $1)::date < $3)
// 参数: ["Asia/Shanghai", "2026-05-26", "2026-06-02"]
```

**代码路径**：`TableRecordConditionWhereVisitor.ts:2132` → `visitDateIsWithIn()` → `resolveDateRange()` → `buildDateBetweenCondition()`

#### 叶子 6: Score > 80

```typescript
// visitNumberIsGreater(spec)
// → buildNumericComparisonCondition(field, value, '>')

// 最终 SQL:
"t"."col_score" > $1
// 参数: [80]
```

**代码路径**：`TableRecordConditionWhereVisitor.ts:2260` → `visitNumberIsGreater()` → `buildNumericComparisonCondition()`

---

### A.7 组合条件的 SQL 拼装

#### 分支 A 的 AND 组合

```sql
-- (Name contains "bug") AND (Assignee = "张三")
(("t"."col_name" ilike $1 escape '\')) and (jsonb_extract_path_text(to_jsonb("t"."col_assignee"), 'id') = $2)
-- 参数: ["%bug%", "usr_zhangsan"]
```

#### 分支 B 的 AND 组合

```sql
-- (Due Date is 本周内) AND (Score > 80)
(((("t"."col_due_date" AT TIME ZONE $1)::date >= $2 AND ("t"."col_due_date" AT TIME ZONE $1)::date < $3)) and ("t"."col_score" > $4))
-- 参数: ["Asia/Shanghai", "2026-05-26", "2026-06-02", 80]
```

#### OR 组合（分支 A ∨ 分支 B）

```sql
-- 分支 A OR 分支 B
((分支 A 的 SQL) or (分支 B 的 SQL))
```

#### 根节点 AND 组合

```sql
-- (Priority = "High") AND (Status ≠ "Done") AND (OR 组合)
((("t"."col_priority" = $1) and ("t"."col_status" is distinct from $2)) and ((分支 A) or (分支 B)))
```

---

### A.8 最终生成的完整 SQL

```sql
SELECT *
FROM "bse_xxxxxxxxxxxxxxx"."tbl_xxxxxxxxxxxxxxx" AS "t"
WHERE (
  (
    ("t"."col_priority" = $1) 
    AND 
    ("t"."col_status" is distinct from $2)
  ) 
  AND 
  (
    (
      ("t"."col_name" ilike $3 escape '\') 
      AND 
      (jsonb_extract_path_text(to_jsonb("t"."col_assignee"), 'id') = $4)
    ) 
    OR 
    (
      (
        (("t"."col_due_date" AT TIME ZONE $5)::date >= $6 
         AND 
         ("t"."col_due_date" AT TIME ZONE $5)::date < $7)
      ) 
      AND 
      ("t"."col_score" > $8)
    )
  )
)
```

**参数绑定数组**：
```typescript
[
  "High",                     // $1: Priority
  "Done",                     // $2: Status
  "%bug%",                    // $3: Name contains
  "usr_zhangsan",             // $4: Assignee id
  "Asia/Shanghai",            // $5: 时区
  "2026-05-26",               // $6: 本周开始
  "2026-06-02",               // $7: 本周结束(+1天)
  80                          // $8: Score
]
```

---

### A.9 数据库表结构说明

#### 动态表创建机制

Teable **不使用 Prisma 定义业务数据表**，而是通过 Kysely 动态创建和管理表：

**Schema 命名规则**：
- Schema 名 = baseId: `bse + 16位字符` (如 `bse_aaaaaaaaaaaaaaaa`)
- 表名 = tableId: `tbl + 16位字符` (如 `tbl_tttttttttttttttt`)

**文件**: `packages/v2/adapter-table-repository-postgres/src/schema/visitors/PostgresTableSchemaFieldCreateVisitor.ts`

```typescript
// 动态 CREATE TABLE 示例
CREATE TABLE "bse_xxxxxxxxxxxxxxx"."tbl_xxxxxxxxxxxxxxx" (
  "__id" TEXT PRIMARY KEY,          // 记录 ID
  "__created_time" TIMESTAMPTZ,     // 创建时间(系统)
  "__last_modified_time" TIMESTAMPTZ, // 最后修改时间(系统)
  "__auto_serial" SERIAL,           // 自增序号
  
  -- 用户定义字段 (动态添加)
  "col_name" TEXT,                  // 单行文本
  "col_status" TEXT,                // 单选
  "col_priority" TEXT,              // 单选
  "col_due_date" TIMESTAMPTZ,       // 日期
  "col_assignee" JSONB,             // 用户(JSON 存储)
  "col_score" DOUBLE PRECISION      // 数字
);
```

**字段类型映射**：

| Teable 字段类型 | PostgreSQL 列类型 |
|---------------|------------------|
| SingleLineText | TEXT |
| LongText | TEXT |
| Number | DOUBLE PRECISION |
| SingleSelect | TEXT |
| MultipleSelect | JSONB (TEXT[] 编码) |
| Checkbox | BOOLEAN |
| Date | TIMESTAMPTZ |
| User (单选) | JSONB `{ id, title, email }` |
| User (多选) | JSONB `[{ id, title }, ...]` |
| Attachment | JSONB |
| Link (多对一) | TEXT (外键) |
| Link (多对多) | 通过中间 junction 表 |

#### Prisma 的作用

Prisma 仅用于**系统元数据表**的管理（`packages/db-data-prisma/prisma/schema.prisma`）：
- `ComputedUpdateOutbox` - 计算字段更新队列
- `RecordHistory` - 记录历史
- `TableTrash` / `RecordTrash` - 回收站
- 等等...

**业务数据存储完全绕过 Prisma**，直接使用 Kysely 进行动态 SQL 构建和执行。

---

### A.10 关键设计决策总结

| 决策点 | 设计选择 | 原因 |
|--------|---------|------|
| 条件表示 | Specification 模式 | 类型安全、可组合、可扩展新操作符 |
| SQL 生成 | Visitor 模式 | 分离领域逻辑与数据库实现，支持多种数据库 |
| OR 分支处理 | 克隆独立访问者 | 每个 OR 分支需要独立的条件累积状态 |
| NULL 处理 | IS DISTINCT FROM / COALESCE | 符合 SQL 三值逻辑，确保 NULL 行被正确筛选 |
| 日期查询 | 时区转换 + DATE 截断 | 支持用户时区的日期精确匹配 |
| 用户/链接字段 | JSONB 存储 | 灵活的结构化数据存储，支持按 id/title 查询 |
| 业务表管理 | 动态 Kysely 建表 | 支持用户自定义字段，无需迁移流程 |
| 系统表管理 | Prisma | 元数据结构稳定，享受 Prisma 生态 |

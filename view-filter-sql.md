# View Filter 与 SQL 构造代码链路分析

## 概述

Teable 的 View Filter 系统采用 **Specification 模式 + Visitor 模式** 的双层架构，实现了从前端过滤条件到数据库 SQL 查询的完整转换链路。

在 `ListTableRecordsHandler.handle()` 中，一条 filter JSON 要依次经过 **5 道预处理** 才进入 Specification 构建，最终由 `TableRecordConditionWhereVisitor` 输出 Kysely SQL 表达式。完整管线如下：

```
前端 Filter JSON
  │
  ├─ 1. resolveFilterFieldKeys()      字段键名→字段ID 统一
  ├─ 2. replaceCurrentUserTagInFilter() "Me"→实际用户ID 替换
  ├─ 3. sanitizeRecordFilter()        校验并剔除无效节点
  ├─ 4. buildRecordConditionSpec()    递归构建 Specification 树
  └─ 5. TableRecordConditionWhereVisitor  Specification→SQL WHERE
```

---

## 第零阶段：前置预处理管线

### 0.1 resolveFilterFieldKeys — 字段键名统一

**文件**: `packages/v2/core/src/queries/ListTableRecordsHandler.ts:69-159`

前端 API 可通过 `fieldKeyType` 参数指定字段标识方式：

```typescript
// packages/v2/core/src/domain/table/fields/FieldKeyType.ts
export const FieldKeyType = {
  Id: 'id',           // 字段ID (默认)
  Name: 'name',       // 字段名
  DbFieldName: 'dbFieldName',  // 数据库列名
} as const;
```

当 `fieldKeyType !== 'id'` 时，`resolveFilterFieldKeys` 递归遍历 filter 树，把所有 `fieldId` 从 name/dbFieldName 反查为真正的字段 ID。**字段引用值**（`value.type === 'field'`）中的 `fieldId` 也一并解析。

```typescript
// ListTableRecordsHandler.ts:81-159
function resolveFilterNodeFieldKeys(
  table: Table,
  node: RecordFilterNode,
  fieldKeyType: FieldKeyType
): Result<RecordFilterNode, DomainError> {
  if (fieldKeyType === FieldKeyType.Id) {
    return ok(node);   // 已经是ID，直接通过
  }

  if (isRecordFilterCondition(node)) {
    // 解析 condition 的 fieldId
    const fieldIdResult = FieldKeyResolverService.resolveFieldKey(
      table, node.fieldId, fieldKeyType
    );
    // 同时解析 value 中的字段引用
    if (isRecordFilterFieldReferenceValue(node.value)) {
      const valueFieldIdResult = FieldKeyResolverService.resolveFieldKey(
        table, node.value.fieldId, fieldKeyType
      );
      // ...
    }
  }
  // group / not 递归处理...
}
```

**调用时机** (`ListTableRecordsHandler.ts:414-416`):
```typescript
const resolvedFilter = query.filter
  ? yield* resolveFilterFieldKeys(table, query.filter, query.fieldKeyType)
  : undefined;
```

### 0.2 replaceCurrentUserTagInFilter — "Me" 占位符替换

**文件**: `packages/v2/core/src/queries/ListTableRecordsHandler.ts:169-213`

用户类型字段（user / createdBy / lastModifiedBy）的过滤值中可以使用字符串 `"Me"` 代表当前登录用户。此函数递归遍历 filter 树，将所有用户字段条件值中的 `"Me"` 替换为当前用户的 `actorId`。

```typescript
const currentUserFilterValue = 'Me';  // ListTableRecordsHandler.ts:45

function replaceCurrentUserTagInFilter(
  table: Table,
  filter: RecordFilter | null | undefined,
  actorId: string
): RecordFilter | null | undefined {
  // ...
  const replaceNode = (node: RecordFilterNode): RecordFilterNode => {
    if (isRecordFilterCondition(node)) {
      const fieldResult = table.getField(
        (field) => field.id().toString() === node.fieldId
      );
      if (fieldResult.isErr() || !isUserLikeFieldType(fieldResult.value.type())) {
        return node;  // 非用户字段，跳过
      }
      // 替换 value 中所有 "Me"
      const replaceValue = (value: RecordFilterValue): RecordFilterValue => {
        if (Array.isArray(value)) {
          return value.map((item) => (item === currentUserFilterValue ? actorId : item));
        }
        return value === currentUserFilterValue ? actorId : value;
      };
      return { ...node, value: replaceValue(node.value) };
    }
    // group / not 递归...
  };
  return replaceNode(filter);
}
```

**调用时机** (`ListTableRecordsHandler.ts:417-421`):
```typescript
const actorResolvedFilter = replaceCurrentUserTagInFilter(
  table, resolvedFilter, context.actorId.toString()
);
```

View 默认 filter 也经过同样的替换 (`ListTableRecordsHandler.ts:453-457`):
```typescript
const defaultFilter = replaceCurrentUserTagInFilter(
  table, effectiveQueryDefaults?.filter(), context.actorId.toString()
);
```

### 0.3 sanitizeRecordFilter — 校验并剔除无效节点

**文件**: `packages/v2/core/src/queries/RecordFilterMapper.ts:161-170`

View 默认 filter 可能引用已被删除的字段。`sanitizeRecordFilter` 对 filter 树做"尽力校验"：逐节点尝试验证，**无法通过验证的节点被静默剔除**（返回 null），而非报错中断。

```typescript
// RecordFilterMapper.ts:104-151
const sanitizeNode = (
  table: Table,
  node: RecordFilterNode
): Result<RecordFilterNode | null, DomainError> => {
  if (isRecordFilterCondition(node)) {
    const fieldResult = resolveField(table, node.fieldId);
    if (fieldResult.isErr()) return ok(null);       // 字段已删除→剔除

    const valueResult = buildConditionValue(table, node.value);
    if (valueResult.isErr()) return ok(null);        // 值无效→剔除

    const specResult = fieldResult.value.spec().create({
      operator: node.operator, value: valueResult.value
    });
    if (specResult.isErr()) return ok(null);         // 操作符不兼容→剔除

    return ok(node);
  }

  if (isRecordFilterGroup(node)) {
    const items: RecordFilterNode[] = [];
    for (const item of node.items) {
      const sanitized = sanitizeNode(table, item);
      if (sanitized.isErr()) return err(sanitized.error);
      if (sanitized.value) items.push(sanitized.value);  // 仅保留有效节点
    }
    if (items.length === 0) return ok(null);             // 组为空→整体剔除
    return ok({ conjunction: node.conjunction, items });
  }
  // not 递归...
};

export const sanitizeRecordFilter = (
  table: Table,
  filter: RecordFilter | null | undefined
): Result<RecordFilter | null | undefined, DomainError> => {
  if (filter === undefined || filter === null) return ok(filter);
  return sanitizeNode(table, filter).map((sanitized) => sanitized ?? null);
};
```

**调用时机** (`ListTableRecordsHandler.ts:458`):
```typescript
const sanitizedDefaultFilter = yield* sanitizeRecordFilter(table, defaultFilter);
```

**注意**：查询级 filter（`actorResolvedFilter`）不经过 sanitize，因为它是用户主动传入的，无效字段应直接报错而非静默剔除。

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

// 叶子节点
{
  fieldId: "fld_xxx",
  operator: "is" | "contains" | "isGreater" | "isWithIn" | ...,
  value: string | number | boolean | Array | DateValue | FieldReference | null
}

// 组合节点
{
  conjunction: "and" | "or",
  items: [RecordFilterNode, RecordFilterNode, ...]
}

// 否定节点
{
  not: RecordFilterNode
}
```

**关键 Schema**:
- `recordFilterConditionSchema` - 叶子条件验证
- `recordFilterGroupSchema` - 组合条件验证（递归定义）
- `recordFilterNotSchema` - 否定条件验证

### 1.2 完整操作符枚举

**文件**: `packages/v2/core/src/domain/table/records/specs/RecordConditionOperators.ts`

```typescript
export const recordConditionOperatorSchema = z.enum([
  'is', 'isNot',
  'contains', 'doesNotContain',
  'isEmpty', 'isNotEmpty',
  'isGreater', 'isGreaterEqual', 'isLess', 'isLessEqual',
  'isAnyOf', 'isNoneOf',
  'hasAnyOf', 'hasAllOf',
  'isNotExactly', 'hasNoneOf', 'isExactly',
  'isWithIn',              // ← 注意大写 W
  'isBefore', 'isAfter',
  'isOnOrBefore', 'isOnOrAfter',
]);
```

> **纠正**：日期范围操作符为 **`isWithIn`**（大写 W），而非 `isWithin`。这在 `RecordConditionOperators.ts:26` 和 `DateConditionSpec.ts:28` 中均有定义。

日期模式枚举（`RecordConditionOperators.ts:143-172`）：
```typescript
export const recordConditionDateModeSchema = z.enum([
  'today', 'tomorrow', 'yesterday',
  'currentWeek', 'currentMonth', 'currentYear',
  'lastWeek', 'lastMonth', 'lastYear',
  'nextWeekPeriod', 'nextMonthPeriod', 'nextYearPeriod',
  'oneWeekAgo', 'oneWeekFromNow',
  'oneMonthAgo', 'oneMonthFromNow',
  'daysAgo', 'daysFromNow',
  'exactDate', 'exactFormatDate',
  'pastWeek', 'pastMonth', 'pastYear',
  'nextWeek', 'nextMonth', 'nextYear',
  'pastNumberOfDays', 'nextNumberOfDays',
]);
```

### 1.3 DTO 到 Specification 的转换

**文件**: `packages/v2/core/src/queries/RecordFilterMapper.ts`

核心函数 `buildRecordConditionSpec` 通过递归遍历将 DTO 树转换为 Specification 树：

```typescript
// RecordFilterMapper.ts:153-159
export const buildRecordConditionSpec = (
  table: Table,
  filter: RecordFilter
): Result<ISpecification<TableRecord, ITableRecordConditionSpecVisitor>, DomainError> => {
  if (!filter) return err(domainError.validation({ message: 'Filter is empty' }));
  return buildSpecFromNode(table, filter);
};

const buildSpecFromNode = (table: Table, node: RecordFilterNode) => {
  if (isRecordFilterCondition(node)) {
    return resolveField(table, node.fieldId).andThen((field) =>
      buildConditionValue(table, node.value).andThen((value) =>
        field.spec().create({ operator: node.operator, value })
      )
    );
  }
  if (isRecordFilterNot(node)) {
    return buildSpecFromNode(table, node.not).andThen((spec) => notSpec(spec));
  }
  if (isRecordFilterGroup(node)) {
    const mode = node.conjunction === 'and' ? 'and' : 'or';
    const builder = RecordConditionSpecBuilder.create(mode);
    for (const item of node.items) {
      const childResult = buildSpecFromNode(table, item);
      if (childResult.isErr()) return err(childResult.error);
      builder.addConditionSpec(childResult.value);
    }
    return builder.build();
  }
  return err(domainError.validation({ message: 'Invalid record filter node' }));
};
```

**`buildConditionValue` 的值类型路由** (`RecordFilterMapper.ts:39-72`):

```typescript
const buildConditionValue = (table: Table, rawValue: RecordFilterValue) => {
  if (rawValue === null) return ok(undefined);                       // 空值操作符
  if (isRecordFilterFieldReferenceValue(rawValue)) {                 // 字段引用
    return FieldId.create(rawValue.fieldId).andThen((fieldId) =>
      table.getField((c) => c.id().equals(fieldId)).andThen((field) => {
        if (rawValue.tableId) { /* 校验跨表引用 */ }
        return RecordConditionFieldReferenceValue.create(field);
      })
    );
  }
  if (isRecordFilterDateValue(rawValue)) {                           // 日期值
    return RecordConditionDateValue.create(rawValue);
  }
  if (Array.isArray(rawValue)) {                                     // 列表值
    return RecordConditionLiteralListValue.create(rawValue);
  }
  return RecordConditionLiteralValue.create(rawValue);               // 标量值
};
```

### 1.4 Specification 构建器

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
      field.accept(this);
    }
  }

  private getColumnAlias(field: Field): string {
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
    return ok(tableAlias ? `${tableAlias}.${column}` : column);
  });
};
```

### 2.3 hostTableAlias 与字段引用路由

**文件**: `packages/v2/adapter-table-repository-postgres/src/record/visitors/TableRecordConditionWhereVisitor.ts`

当 filter 值是**字段引用**（`{ type: 'field', fieldId: '...' }`）而非字面量时，需要确定引用字段属于哪张表。`hostTableAlias` 正是为这个场景设计的。

#### 构造参数

```typescript
// TableRecordConditionWhereVisitor.ts:207-218
interface TableRecordConditionWhereVisitorOptions {
  tableAlias?: string;
  /**
   * Optional host table alias for field reference values (when isSymbol is true).
   * When conditional lookups use field references, the referenced field is from
   * the host table (e.g. 't'), while the filter field is from the foreign table
   * (tableAlias, e.g. 'f').
   * This generates SQL like: "f"."Status" = "t"."StatusFilter"
   */
  hostTableAlias?: string;
}
```

#### resolvePrimitiveOperand — 决定引用列归属

```typescript
// TableRecordConditionWhereVisitor.ts:284-301
const resolvePrimitiveOperand = (
  value: core.RecordConditionValue,
  tableAlias?: string,
  hostTableAlias?: string
): Result<PrimitiveOperand, DomainError> => {
  if (core.isRecordConditionLiteralValue(value)) {
    return ok({ kind: 'literal', value: value.toValue() });
  }
  if (core.isRecordConditionFieldReferenceValue(value)) {
    return safeTry<PrimitiveOperand, DomainError>(function* () {
      // 字段引用使用 hostTableAlias（如有），否则回退到 tableAlias
      const alias = hostTableAlias ?? tableAlias;
      const column = yield* resolveColumn(value.field(), alias);
      return ok({ kind: 'field', column });
    });
  }
  return err(core.domainError.unexpected({ message: '...' }));
};
```

#### classifyFieldReferenceComparison — 字段引用的类型路由

**文件**: `TableRecordConditionWhereVisitor.ts:432-471`

当两个字段相互比较时（如 `字段A = 字段B`），需要根据字段类型决定比较策略。`hasHostTableAlias` 参数会影响路由结果：

```typescript
type FieldReferenceComparisonRoute =
  | { kind: 'userOrLinkIds' }      // 用户/链接按 ID 比较
  | { kind: 'linkTitle' }          // 链接按标题比较
  | { kind: 'date'; compareAsDateOnly: boolean }  // 日期比较
  | { kind: 'json' }               // JSON 序列化比较
  | { kind: 'generic' }            // 通用相等比较
  | { kind: 'incompatible' };      // 类型不兼容→ 1=0

const classifyFieldReferenceComparison = (
  field: core.Field,
  referenceField: core.Field,
  hasHostTableAlias: boolean        // ← 是否有 hostTableAlias
): Result<FieldReferenceComparisonRoute, DomainError> => {
  // 用户/链接类字段
  if (leftIsUserOrLinkLike) {
    if (rightIsUserOrLinkLike) return ok({ kind: 'userOrLinkIds' });
    if (fieldIsLink(field)) return ok({ kind: 'linkTitle' });
    // 有 hostTableAlias 且右侧不是用户/链接 → 不兼容
    return ok(hasHostTableAlias ? { kind: 'incompatible' } : { kind: 'generic' });
  }

  // 日期字段
  if (compareAsDateOnly) return ok({ kind: 'date', compareAsDateOnly });

  // JSON 字段
  if (fieldIsJson(field) || fieldIsJson(referenceField)) return ok({ kind: 'json' });

  // 无 hostTableAlias → 通用比较
  if (!hasHostTableAlias) return ok({ kind: 'generic' });

  // 有 hostTableAlias → 检查两侧类型是否兼容
  const leftKind = yield* resolveFieldReferenceComparisonKind(field);
  const rightKind = yield* resolveFieldReferenceComparisonKind(referenceField);
  return ok(leftKind === rightKind ? { kind: 'generic' } : { kind: 'incompatible' });
};
```

**路由结果对应的 SQL 生成** (`buildIsCondition`, `TableRecordConditionWhereVisitor.ts:695-826`):

| 路由 | SQL 策略 |
|------|---------|
| `userOrLinkIds` | `jsonb_extract_path_text(to_jsonb(left), 'id') = jsonb_extract_path_text(to_jsonb(right), 'id')` |
| `linkTitle` | `buildLinkTitleMatchCondition(left, right)` |
| `date` | `(left AT TIME ZONE tz)::date = (right AT TIME ZONE tz)::date` |
| `json` | `to_jsonb(left) = to_jsonb(right)` |
| `generic` | `left = right` |
| `incompatible` | `1 = 0` (恒假) |

---

## 第三阶段：SQL 拼装 (SQL Construction)

### 3.1 Visitor 模式的核心接口

**文件**: `packages/v2/core/src/domain/table/records/specs/ITableRecordConditionSpecVisitor.ts`

定义了超过 200 个 visit 方法，每个字段类型 × 每个操作符组合一个方法。

### 3.2 SQL WHERE 条件访问者实现

**文件**: `packages/v2/adapter-table-repository-postgres/src/record/visitors/TableRecordConditionWhereVisitor.ts`

`TableRecordConditionWhereVisitor` 是最核心的 SQL 生成器。

#### 核心结构：

```typescript
class TableRecordConditionWhereVisitor
  extends AbstractSpecFilterVisitor<RecordConditionWhere>
  implements ITableRecordConditionSpecVisitor<RecordConditionWhere> {

  constructor(options?: TableRecordConditionWhereVisitorOptions) {
    super();
    this.tableAlias = options?.tableAlias;
    this.hostTableAlias = options?.hostTableAlias;
  }

  clone(): this {
    return new TableRecordConditionWhereVisitor({
      tableAlias: this.tableAlias,
      hostTableAlias: this.hostTableAlias,
    }) as this;
  }

  and(left, right) { return sql`(${left}) and (${right})`; }
  or(left, right)  { return sql`(${left}) or (${right})`; }
  not(inner)       { return sql`not (${inner})`; }
}
```

#### 关键构建函数：

**1. `buildIsCondition`** (`TableRecordConditionWhereVisitor.ts:695-826`)

处理 `is` 操作符，根据字段类型和值类型分路：

```
buildIsCondition(field, value, tableAlias, hostTableAlias)
  │
  ├─ value 是日期值 → resolveDateRange → BETWEEN
  ├─ value 是字段引用 → classifyFieldReferenceComparison → 路由
  │   ├─ userOrLinkIds → jsonb_extract_path_text 比较
  │   ├─ linkTitle     → buildLinkTitleMatchCondition
  │   ├─ date          → 时区转换后比较
  │   ├─ json          → to_jsonb 比较
  │   ├─ generic       → 直接 =
  │   └─ incompatible  → 1 = 0
  ├─ 字段是 user/link (字面量) → jsonb_extract_path_text(..., 'id') =
  ├─ 字段是 json/array → EXISTS + jsonb_array_elements_text
  └─ 普通字段 → column = literal
```

**2. `buildContainsCondition`** (`TableRecordConditionWhereVisitor.ts:968-1019`)

**3. `buildNumericComparisonCondition`** (`TableRecordConditionWhereVisitor.ts:1021-1078`)

**4. `buildDateComparisonCondition`** (`TableRecordConditionWhereVisitor.ts:1080-1182`)

**5. `buildListCondition`** (`TableRecordConditionWhereVisitor.ts:1211-1327`)

**6. `resolveDateRange`** (`TableRecordConditionWhereVisitor.ts:473-639`)

#### 日期过滤专项深度解析

日期操作符分为**两条独立代码路径**，调用链完全分离：

```
日期操作符分派
  ├─ isWithIn → visitDateIsWithIn → applyIsWithin → buildIsWithinCondition
  │
  └─ isBefore / isAfter / isOnOrBefore / isOnOrAfter
          → visitDateIsBefore / visitDateIsAfter
          → applyDateComparison
          → buildDateComparisonCondition
```

##### 路径 A：isWithIn → buildIsWithinCondition

**文件**: `TableRecordConditionWhereVisitor.ts:1184-1209`

```typescript
const buildIsWithinCondition = (
  field: core.Field,
  value: core.RecordConditionValue | undefined,
  tableAlias?: string
): Result<RecordConditionWhere, DomainError> => {
  return safeTry<RecordConditionWhere, DomainError>(function* () {
    const column = yield* resolveColumn(field, tableAlias);
    const dateValue = yield* resolveDateValue(value);
    const range = yield* resolveDateRange(dateValue, resolveDateFormatting(field));
    const columnRef = sql.ref(column);
    const isMultiple = isArrayLikeOutputField(field, yield* fieldIsMultiple(field));

    if (isMultiple || fieldIsJson(field)) {
      // 多值字段: EXISTS + jsonb_array_elements_text
      const normalizedArray = normalizeToJsonArray(columnRef);
      return ok(sql`EXISTS (
        SELECT 1 FROM jsonb_array_elements_text(${normalizedArray}) AS elem
        WHERE NULLIF(elem, 'null')::timestamptz BETWEEN ${range.start} AND ${range.end}
      )`);
    }

    // 单值字段: 直接 BETWEEN
    return ok(sql`${columnRef} between ${range.start} and ${range.end}`);
  });
};
```

**关键特征**：
- **不支持字段引用**：`value` 必须是 `RecordConditionDateValue`（字面量日期模式），不能是字段引用
- **总是用 BETWEEN**：无论单值/多值，都生成 `BETWEEN start AND end` 形式
- **参数位**：2 个参数位（`range.start`, `range.end`）

##### 路径 B：isBefore/isAfter 等 → buildDateComparisonCondition

**文件**: `TableRecordConditionWhereVisitor.ts:1080-1182`

```typescript
const buildDateComparisonCondition = (
  field: core.Field,
  value: core.RecordConditionValue | undefined,
  operator: ComparisonOperator,  // '<' | '<=' | '>' | '>='
  tableAlias?: string,
  hostTableAlias?: string
): Result<RecordConditionWhere, DomainError> => {
  return safeTry<RecordConditionWhere, DomainError>(function* () {
    const column = yield* resolveColumn(field, tableAlias);
    const columnRef = sql.ref(column);
    const isMultiple = isArrayLikeOutputField(field, yield* fieldIsMultiple(field));

    // 分支 1: 值是字段引用（如 日期1 < 日期2）
    if (core.isRecordConditionFieldReferenceValue(value)) {
      const rightColumn = yield* resolveColumn(value.field(), hostTableAlias ?? tableAlias);
      const right = sql.ref(rightColumn);
      // ... 生成 col OP right 的比较
    }

    // 分支 2: 值是字面量日期
    const dateValue = yield* resolveDateValue(value);
    const range = yield* resolveDateRange(dateValue, resolveDateFormatting(field));

    // ══════════════════════════════════════════════
    // 关键：边界选择逻辑（第 1148 行）
    // ══════════════════════════════════════════════
    const boundary = operator === '>' || operator === '<=' ? range.end : range.start;
    //            ┌──────────┬──────────┬──────────┐
    //            │  '>'     │  '>='    │  '<'     │  '<='
    // ┌──────────┼──────────┼──────────┼──────────┼──────────
    // │ boundary │  end     │  start   │  start   │  end
    // └──────────┴──────────┴──────────┴──────────┴──────────

    const right = sql`${boundary}`;

    if (isMultiple || fieldIsJson(field)) {
      // 多值字段: EXISTS + 元素级比较
      return ok(sql`EXISTS (
        SELECT 1 FROM jsonb_array_elements_text(${normalizedArray}) AS elem
        WHERE NULLIF(elem, 'null')::timestamptz ${sql.raw(operator)} ${right}
      )`);
    }

    // 单值字段: 直接比较
    if (operator === '>') return ok(sql`${columnRef} > ${right}`);
    if (operator === '>=') return ok(sql`${columnRef} >= ${right}`);
    if (operator === '<') return ok(sql`${columnRef} < ${right}`);
    return ok(sql`${columnRef} <= ${right}`);
  });
};
```

**关键特征**：
- **支持字段引用**：`value` 可以是字段引用（`{ type: 'field', fieldId: '...' }`）
- **边界选择逻辑**（第 1148 行）：
  | 操作符 | 比较方向 | 使用的边界 | 语义（以 today 为例） |
  |--------|---------|-----------|---------------------|
  | `>` (isAfter) | `col > ?` | `range.end` | 今天结束之后 → 明天及以后 |
  | `>=` (isOnOrAfter) | `col >= ?` | `range.start` | 今天开始或之后 → 今天及以后 |
  | `<` (isBefore) | `col < ?` | `range.start` | 今天开始之前 → 昨天及以前 |
  | `<=` (isOnOrBefore) | `col <= ?` | `range.end` | 今天结束或之前 → 今天及以前 |
- **参数位**：1 个参数位（仅 `boundary`）

##### resolveDateRange 的统一输出

**文件**: `TableRecordConditionWhereVisitor.ts:473-639`

无论哪种操作符路径，日期范围解析都统一通过 `resolveDateRange`，返回 ISO 字符串：

```typescript
const resolveDateRange = (
  value: core.RecordConditionDateValue,
  formatting?: core.DateTimeFormatting
): Result<{ start: string; end: string }, DomainError> => {
  // ... 27 种 mode 的分支逻辑
  return ok({
    start: range[0].toISOString(),  // 始终是 ISO 格式
    end: range[1].toISOString()     // 始终是 ISO 格式
  });
};
```

**各 date mode 的范围计算规则**（基准日 2026-05-28 周四, timeZone=UTC, weekStart=1）：

| mode | 范围计算 | range.start | range.end |
|------|---------|-------------|-----------|
| `today` | `[startOf('day'), endOf('day')]` | `2026-05-28T00:00:00.000Z` | `2026-05-28T23:59:59.999Z` |
| `tomorrow` / `yesterday` | 当天 | - | - |
| `currentWeek` | `[cursorDate.startOf('week').startOf('day'), cursorDate.endOf('week').endOf('day')]` | `2026-05-25T00:00:00.000Z` (周一) | `2026-05-31T23:59:59.999Z` (周日) |
| `currentMonth` | 当月 | `2026-05-01T00:00:00.000Z` | `2026-05-31T23:59:59.999Z` |
| `currentYear` | 当年 | `2026-01-01T00:00:00.000Z` | `2026-12-31T23:59:59.999Z` |
| `lastWeek` | cursorDate - 1 week, 然后 startOf/endOf('week') | `2026-05-18T00:00:00.000Z` (周一) | `2026-05-24T23:59:59.999Z` (周日) |
| `nextWeekPeriod` | cursorDate + 1 week, 然后 startOf/endOf('week') | `2026-06-01T00:00:00.000Z` (周一) | `2026-06-07T23:59:59.999Z` (周日) |
| `oneWeekAgo` | 7 天前的那一天 | `2026-05-21T00:00:00.000Z` | `2026-05-21T23:59:59.999Z` |
| `daysAgo(3)` | 3 天前的那一天 | `2026-05-25T00:00:00.000Z` | `2026-05-25T23:59:59.999Z` |
| `pastWeek` | 过去 7 天（含今天） | `2026-05-21T00:00:00.000Z` | `2026-05-28T23:59:59.999Z` |
| `nextWeek` | 未来 7 天（含今天） | `2026-05-28T00:00:00.000Z` | `2026-06-03T23:59:59.999Z` |
| `exactDate('2026-05-28')` | 指定日期 | `2026-05-28T00:00:00.000Z` | `2026-05-28T23:59:59.999Z` |

> **注意**：
> - 周起始是周一（第 567-569 行明确设置 `weekStart: 1`），这会影响所有周相关的计算。
> - `endOf('day')` 产生 `23:59:59.999`（毫秒精度），不是 `23:59:59`。
> - `pastWeek` / `nextWeek` 与 `lastWeek` / `nextWeekPeriod` 不同：前者是基于天的 7 天滑动窗口（含今天），后者是基于周的日历周。

##### 两条路径的参数位对应关系表

| 操作符 | 函数 | range.start 位置 | range.end 位置 | 总参数位 |
|--------|------|-----------------|---------------|---------|
| `isWithIn` | `buildIsWithinCondition` | $1 | $2 | **2** |
| `isBefore` (`<`) | `buildDateComparisonCondition` | $1 (boundary=start) | - | **1** |
| `isAfter` (`>`) | `buildDateComparisonCondition` | - | $1 (boundary=end) | **1** |
| `isOnOrBefore` (`<=`) | `buildDateComparisonCondition` | - | $1 (boundary=end) | **1** |
| `isOnOrAfter` (`>=`) | `buildDateComparisonCondition` | $1 (boundary=start) | - | **1** |

### 3.3 WHERE 子句构建入口

**文件**: `packages/v2/adapter-table-repository-postgres/src/record/repository/buildRecordWhereClause.ts`

```typescript
export const buildRecordWhereClause = (
  spec: ISpecification<TableRecord, ITableRecordConditionSpecVisitor>,
  options?: TableRecordConditionWhereVisitorOptions  // 含 tableAlias + hostTableAlias
): Result<Expression<SqlBool> | null, DomainError> => {
  const visitor = new TableRecordConditionWhereVisitor(options);
  const acceptResult = spec.accept(visitor);
  if (acceptResult.isErr()) return err(acceptResult.error);
  const whereResult = visitor.where();
  if (whereResult.isErr()) {
    if (whereResult.error.message === 'Empty where condition') return ok(null);
    return err(whereResult.error);
  }
  return ok(whereResult.value as unknown as Expression<SqlBool>);
};
```

---

## 完整调用链

### ListTableRecordsHandler 中的真实调用路径

```
ListTableRecordsHandler.handle(context, query)
  │
  ├─ 1. 加载 Table
  │     └─ tableRepository.findOne(context, TableByIdSpec)
  │
  ├─ 2. resolveFilterFieldKeys(table, query.filter, query.fieldKeyType)
  │     └─ 将 name/dbFieldName 键名反查为字段 ID
  │
  ├─ 3. replaceCurrentUserTagInFilter(table, resolvedFilter, actorId)
  │     └─ 将 "Me" 替换为当前用户 ID
  │
  ├─ 4. 加载 View 默认过滤
  │     ├─ effectiveView.queryDefaults()
  │     ├─ replaceCurrentUserTagInFilter(table, defaultFilter, actorId)
  │     └─ sanitizeRecordFilter(table, defaultFilter)
  │         └─ 剔除引用已删除字段的节点
  │
  ├─ 5. mergeFilterWithViewDefaults(sanitizedDefault, actorResolvedFilter)
  │     └─ sanitizeFilterByEnabledFieldIds(merged, enabledFieldIds)
  │
  ├─ 6. buildQueryPlan(context, table, query, effectiveFilter, ...)
  │     └─ buildRecordConditionSpec(table, resolvedFilter)
  │         └─ RecordFilterMapper.ts 递归 DTO → Specification
  │
  ├─ 7. tableRecordQueryRepository.find(context, table, spec, ...)
  │     └─ buildRecordWhereClause(spec, { tableAlias: 't' })
  │         └─ TableRecordConditionWhereVisitor
  │             └─ spec.accept(visitor) → SQL Expr
  │
  └─ 8. 返回结果 + 字段键名反向转换
      └─ FieldKeyResolverService.transformResponseKeys(table, fields, fieldKeyType)
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

**AndSpec.accept()** — 复用同一个访问者，顺序累积条件：
```typescript
accept(v: V): Result<void, DomainError> {
  return v.visit(this)
    .andThen(() => this.left.accept(v))
    .andThen(() => this.right.accept(v))
    .map(() => undefined);
}
```

**OrSpec.accept()** — 克隆两个独立访问者，分别累积后合并：
```typescript
accept(v: V): Result<void, DomainError> {
  if (isSpecFilterVisitor(v)) {
    const leftVisitor = v.clone();
    const rightVisitor = v.clone();
    return v.visit(this)
      .andThen(() => this.left.accept(leftVisitor))
      .andThen(() => this.right.accept(rightVisitor))
      .andThen(() => leftVisitor.where())
      .andThen(leftCond =>
        rightVisitor.where().map(rightCond => v.or(leftCond, rightCond))
      )
      .andThen(cond => v.addCond(cond));
  }
}
```

**NotSpec.accept()** — 克隆一个访问者处理内部条件，然后取反：
```typescript
accept(v: V): Result<void, DomainError> {
  if (isSpecFilterVisitor(v)) {
    const innerVisitor = v.clone();
    return v.visit(this)
      .andThen(() => this.inner.accept(innerVisitor))
      .andThen(() => innerVisitor.where())
      .map((innerCond) => v.not(innerCond))
      .andThen((cond) => v.addCond(cond));
  }
}
```

### 3. Builder 模式

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
- 通过 `hostTableAlias` 确定引用字段的表归属
- 通过 `classifyFieldReferenceComparison` 路由到正确的比较策略
- 跨表且类型不兼容时降级为 `1 = 0`

### 4. 性能考虑

- 复杂条件可能生成深层嵌套子查询
- 多值字段过滤使用 `EXISTS` 而非数组操作符
- 日期范围查询优先使用 BETWEEN 以利用索引

---

## 数据库表结构说明

### 动态表命名机制

Teable **不使用 Prisma 定义业务数据表**，而是通过 Kysely 动态创建和管理表。

**`DbTableName` 的真实规则** (`packages/v2/core/src/domain/table/DbTableName.ts`):

`DbTableName` 是一个 `RehydratedValueObject`，其值由建表时决定，**没有固定的 bse/tbl 前缀模式**。格式为 `schema.tableName`（含点号时按 `.` 分割）或不含 schema 的纯表名。

```typescript
// DbTableName.ts
export class DbTableName extends RehydratedValueObject {
  split(options?: { defaultSchema?: string | null }): Result<{
    schema: string | null;
    tableName: string;
  }> {
    return this.value().map((raw) => {
      const dotIndex = raw.indexOf('.');
      if (dotIndex === -1) {
        return { schema: options?.defaultSchema ?? null, tableName: raw };
      }
      return { schema: raw.slice(0, dotIndex), tableName: raw.slice(dotIndex + 1) };
    });
  }
}
```

建表 API 允许用户指定 `dbTableName` (`packages/v2/core/src/schemas/table/createTable.schema.ts:15`):
```typescript
export const createTableInputSchema = z.object({
  baseId: z.string(),
  tableId: z.string().optional(),
  name: z.string(),
  dbTableName: z.string().optional(),  // ← 用户可自定义
  // ...
});
```

**实际使用中**，`dbTableName` 通常被赋值为 `{baseId}.{tableId}` 格式（如 `bse_aaaaaaaaaaaaaaaa.tbl_tttttttttttttttt`），但**这不是由代码强制约束的**，不能假设一定遵循此模式。

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

**业务数据存储完全绕过 Prisma**，直接使用 Kysely 进行动态 SQL 构建和执行。

---

## 相关文件速查

| 模块 | 文件路径 |
|------|---------|
| DTO 定义 | `packages/v2/core/src/queries/RecordFilterDto.ts` |
| 操作符枚举 | `packages/v2/core/src/domain/table/records/specs/RecordConditionOperators.ts` |
| 字段键类型 | `packages/v2/core/src/domain/table/fields/FieldKeyType.ts` |
| 字段键解析服务 | `packages/v2/core/src/application/services/FieldKeyResolverService.ts` |
| DTO → Spec 映射 | `packages/v2/core/src/queries/RecordFilterMapper.ts` |
| Spec 构建器 | `packages/v2/core/src/domain/table/records/specs/RecordConditionSpecBuilder.ts` |
| 访问者接口 | `packages/v2/core/src/domain/table/records/specs/ITableRecordConditionSpecVisitor.ts` |
| SQL WHERE 生成 | `packages/v2/adapter-table-repository-postgres/src/record/visitors/TableRecordConditionWhereVisitor.ts` |
| 字段列映射 | `packages/v2/adapter-table-repository-postgres/src/record/query-builder/FieldOutputColumnVisitor.ts` |
| 查询仓库 | `packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordQueryRepository.ts` |
| 列表查询处理器 | `packages/v2/core/src/queries/ListTableRecordsHandler.ts` |
| 表名值对象 | `packages/v2/core/src/domain/table/DbTableName.ts` |
| 建表 Schema | `packages/v2/core/src/schemas/table/createTable.schema.ts` |

---

## 附录：最小可复现样例

以下给出一个可直接通过 API 提交的最小 filter JSON，追踪到最终 SQL 的完整过程。

### B.1 样例场景

一张任务表，包含 3 个字段：

| 字段名 | 字段类型 | 字段 ID | 数据库列名 (dbFieldName) |
|--------|---------|---------|--------------------------|
| Name | 单行文本 | `fldName001` | `col_name` |
| Status | 单选 | `fldStatus01` | `col_status` |
| Due Date | 日期(仅日期,UTC) | `fldDueDate1` | `col_due_date` |

业务需求：**Status 为 "Open" 且 Due Date 在本周内**

### B.2 前端提交的 filter JSON

使用 `fieldKeyType: "name"` 提交（字段键名为字段名，非 ID）：

```json
{
  "conjunction": "and",
  "items": [
    {
      "fieldId": "Status",
      "operator": "is",
      "value": "Open"
    },
    {
      "fieldId": "Due Date",
      "operator": "isWithIn",
      "value": {
        "mode": "currentWeek",
        "timeZone": "UTC"
      }
    }
  ]
}
```

### B.3 第零阶段：预处理管线执行

#### 步骤 1: resolveFilterFieldKeys

`fieldKeyType = "name"`，需要将 `"Status"` → `"fldStatus01"`、`"Due Date"` → `"fldDueDate1"`：

```json
{
  "conjunction": "and",
  "items": [
    {
      "fieldId": "fldStatus01",
      "operator": "is",
      "value": "Open"
    },
    {
      "fieldId": "fldDueDate1",
      "operator": "isWithIn",
      "value": {
        "mode": "currentWeek",
        "timeZone": "UTC"
      }
    }
  ]
}
```

#### 步骤 2: replaceCurrentUserTagInFilter

无用户字段条件，无 `"Me"` 需要替换，直接通过。

#### 步骤 3: sanitizeRecordFilter

查询级 filter 不经过 sanitize（仅 View 默认 filter 经过）。

### B.4 第一阶段：buildRecordConditionSpec

递归解析为 Specification 树：

```
AndSpec (根)
  ├─ left: SingleSelectConditionSpec (Status = "Open")
  └─ right: DateConditionSpec (Due Date isWithIn currentWeek)
```

### B.5 第二阶段：字段映射

```
"fldStatus01" → resolveColumn → "col_status" (加别名 → "t.col_status")
"fldDueDate1" → resolveColumn → "col_due_date" (加别名 → "t.col_due_date")
```

### B.6 第三阶段：SQL 生成

#### 叶子 1: Status = "Open"

```
visitSingleSelectIs → buildIsCondition → 普通字段 + 字面量

SQL: "t"."col_status" = $1
参数: ["Open"]
```

#### 叶子 2: Due Date isWithIn currentWeek

```
visitDateIsWithIn → applyIsWithin → buildIsWithinCondition
  → resolveDateValue(value)
    → resolveDateRange({ mode: "currentWeek", timeZone: "UTC" })

基准日 2026-05-28 (周四, UTC):
  dateUtil = new DateUtil("UTC")
  cursorDate = dateUtil.date()  // 2026-05-28Txx:xx:xxZ (当前 UTC 时间)
  weekStart = cursorDate.startOf('week').startOf('day')  // 周一
  weekEnd   = cursorDate.endOf('week').endOf('day')       // 周日

  → return {
      start: "2026-05-25T00:00:00.000Z",   // 周一 00:00
      end:   "2026-05-31T23:59:59.999Z"     // 周日 23:59:59.999
    }

⚠️ 注意：buildIsWithinCondition 中**没有 AT TIME ZONE 转换**
  时区转换完全在 resolveDateRange 内部完成
  range.start / range.end 已经是对应时区的 ISO 字符串

SQL: "t"."col_due_date" between $1 and $2
参数: ["2026-05-25T00:00:00.000Z", "2026-05-31T23:59:59.999Z"]
```

### B.7 最终 SQL

```sql
SELECT *
FROM "bse_xxx"."tbl_yyy" AS "t"
WHERE (
  ("t"."col_status" = $1)
  AND
  ("t"."col_due_date" between $2 and $3)
)
```

**参数绑定**：
```typescript
["Open", "2026-05-25T00:00:00.000Z", "2026-05-31T23:59:59.999Z"]
```

### B.8 补充：含字段引用和 hostTableAlias 的场景

当 Conditional Lookup 字段的 filter 使用字段引用值时，`hostTableAlias` 被传入。例如：在链接表的查询中，条件为"链接表的 Status 等于主表的 StatusFilter"：

```
构造: TableRecordConditionWhereVisitor({
  tableAlias: 'f',      // 链接表（外键表）
  hostTableAlias: 't'   // 主表
})

filter: {
  fieldId: "fld_link_status",
  operator: "is",
  value: { type: "field", fieldId: "fld_main_status_filter" }
}

字段引用路由:
  resolvePrimitiveOperand(value, 'f', 't')
    → hostTableAlias ?? tableAlias = 't'
    → 引用列 = resolveColumn(fld_main_status_filter, 't') = "t.col_status_filter"

  classifyFieldReferenceComparison(link_status, main_status_filter, hasHostTableAlias=true)
    → 假设都是 singleSelect → kind: 'generic'

SQL: "f"."col_status" = "t"."col_status_filter"
```

如果左侧是 user 字段而右侧不是 user/link 类型，且 `hasHostTableAlias=true`：
```
classifyFieldReferenceComparison → kind: 'incompatible'
SQL: 1 = 0   (恒假，保护数据安全)
```

### B.9 补充：含 "Me" 占位符的场景

```json
{
  "fieldId": "fld_assignee",
  "operator": "is",
  "value": "Me"
}
```

经过 `replaceCurrentUserTagInFilter` 后：
```json
{
  "fieldId": "fld_assignee",
  "operator": "is",
  "value": "usr_abc123"
}
```

最终 SQL（用户字段，单选）：
```sql
jsonb_extract_path_text(to_jsonb("t"."col_assignee"), 'id') = $1
-- 参数: ["usr_abc123"]
```

### B.10 isWithIn 与 isBefore 的 SQL 对照

**固定基准日**：2026-05-28 (周四), timeZone=UTC, weekStart=1

**resolveDateRange 统一输出**（所有操作符共享）：

| mode | range.start | range.end |
|------|-------------|-----------|
| `today` | `2026-05-28T00:00:00.000Z` | `2026-05-28T23:59:59.999Z` |
| `currentWeek` | `2026-05-25T00:00:00.000Z` (周一) | `2026-05-31T23:59:59.999Z` (周日) |

---

#### 对照 1: isWithIn today

```json
{ "fieldId": "fldDueDate1", "operator": "isWithIn", "value": { "mode": "today", "timeZone": "UTC" } }
```

**调用路径**：`visitDateIsWithIn → applyIsWithin → buildIsWithinCondition`

**SQL**：
```sql
"t"."col_due_date" between $1 and $2
```
**参数**：`["2026-05-28T00:00:00.000Z", "2026-05-28T23:59:59.999Z"]`

**匹配范围**：`2026-05-28 00:00:00.000` ≤ col ≤ `2026-05-28 23:59:59.999` → **今天全天**

---

#### 对照 2: isBefore today

```json
{ "fieldId": "fldDueDate1", "operator": "isBefore", "value": { "mode": "today", "timeZone": "UTC" } }
```

**调用路径**：`visitDateIsBefore → applyDateComparison('<') → buildDateComparisonCondition`

**边界选择**：`operator === '<' → boundary = range.start = 2026-05-28T00:00:00.000Z`

**SQL**：
```sql
"t"."col_due_date" < $1
```
**参数**：`["2026-05-28T00:00:00.000Z"]`

**匹配范围**：col < `2026-05-28 00:00:00.000` → **昨天及以前**

---

#### 对照 3: isOnOrBefore today

**边界选择**：`operator === '<=' → boundary = range.end = 2026-05-28T23:59:59.999Z`

**SQL**：
```sql
"t"."col_due_date" <= $1
```
**参数**：`["2026-05-28T23:59:59.999Z"]`

**匹配范围**：col ≤ `2026-05-28 23:59:59.999` → **今天及以前**

---

#### 对照 4: isAfter today

**边界选择**：`operator === '>' → boundary = range.end = 2026-05-28T23:59:59.999Z`

**SQL**：
```sql
"t"."col_due_date" > $1
```
**参数**：`["2026-05-28T23:59:59.999Z"]`

**匹配范围**：col > `2026-05-28 23:59:59.999` → **明天及以后**

---

#### 对照 5: isOnOrAfter today

**边界选择**：`operator === '>=' → boundary = range.start = 2026-05-28T00:00:00.000Z`

**SQL**：
```sql
"t"."col_due_date" >= $1
```
**参数**：`["2026-05-28T00:00:00.000Z"]`

**匹配范围**：col ≥ `2026-05-28 00:00:00.000` → **今天及以后**

---

#### 对照总结表（mode = "today"）

| 操作符 | 函数 | 边界选择 | SQL 形式 | 参数值 | 匹配范围 |
|--------|------|---------|---------|--------|---------|
| `isWithIn` | `buildIsWithinCondition` | start + end | `BETWEEN $1 AND $2` | `[00:00:00.000, 23:59:59.999]` | **今天全天** |
| `isBefore` (`<`) | `buildDateComparisonCondition` | start | `< $1` | `00:00:00.000` | **昨天及以前** |
| `isOnOrBefore` (`<=`) | `buildDateComparisonCondition` | end | `<= $1` | `23:59:59.999` | **今天及以前** |
| `isAfter` (`>`) | `buildDateComparisonCondition` | end | `> $1` | `23:59:59.999` | **明天及以后** |
| `isOnOrAfter` (`>=`) | `buildDateComparisonCondition` | start | `>= $1` | `00:00:00.000` | **今天及以后** |

#### 对照总结表（mode = "currentWeek"）

| 操作符 | 函数 | 边界选择 | SQL 形式 | 参数值 | 匹配范围 |
|--------|------|---------|---------|--------|---------|
| `isWithIn` | `buildIsWithinCondition` | start + end | `BETWEEN $1 AND $2` | `[05-25T00:00, 05-31T23:59:59.999]` | **本周(周一~周日)** |
| `isBefore` (`<`) | `buildDateComparisonCondition` | start | `< $1` | `05-25T00:00:00.000` (周一) | **上周末及以前** |
| `isOnOrBefore` (`<=`) | `buildDateComparisonCondition` | end | `<= $1` | `05-31T23:59:59.999` (周日) | **本周及以前** |
| `isAfter` (`>`) | `buildDateComparisonCondition` | end | `> $1` | `05-31T23:59:59.999` (周日) | **下周一及以后** |
| `isOnOrAfter` (`>=`) | `buildDateComparisonCondition` | start | `>= $1` | `05-25T00:00:00.000` (周一) | **本周及以后** |

> **核心差异**：`isWithIn` 始终用 `BETWEEN start AND end`（闭区间，2 个参数位），`isBefore/After` 用单边界比较运算符（1 个参数位）。边界选择的关键在于 `operator === '>' || '<=' ? end : start`，即「严格大于/小于等于」取 end，「严格小于/大于等于」取 start——这保证了 `isAfter(today)` 不含今天、`isOnOrAfter(today)` 含今天的语义正确性。

---

## 关键设计决策总结

| 决策点 | 设计选择 | 原因 |
|--------|---------|------|
| 字段键解析 | resolveFilterFieldKeys 统一入口 | 支持三种键名格式（id/name/dbFieldName），尽早统一为 ID |
| 当前用户替换 | replaceCurrentUserTagInFilter | "Me" 占位符在查询时才知实际用户，需延迟替换 |
| 无效节点容错 | sanitizeRecordFilter | View 默认 filter 可能引用已删字段，静默剔除而非报错 |
| 条件表示 | Specification 模式 | 类型安全、可组合、可扩展新操作符 |
| SQL 生成 | Visitor 模式 | 分离领域逻辑与数据库实现，支持多种数据库 |
| OR 分支处理 | 克隆独立访问者 | 每个 OR 分支需要独立的条件累积状态 |
| 字段引用路由 | classifyFieldReferenceComparison | 不同字段类型间比较策略不同，hostTableAlias 影响兼容性判定 |
| NULL 处理 | IS DISTINCT FROM / COALESCE | 符合 SQL 三值逻辑，确保 NULL 行被正确筛选 |
| **日期范围操作符** | **两条独立代码路径** | `isWithIn` 走 `buildIsWithinCondition`（BETWEEN，2 参数位）；`isBefore/After` 走 `buildDateComparisonCondition`（单边界，1 参数位） |
| 日期边界选择 | `operator === '>' || '<=' ? end : start` | 统一用 `resolveDateRange` 输出 range，根据操作符选择一个边界；`isWithIn` 同时用两个边界 |
| 周起始 | 周一为起始（`weekStart: 1`） | 符合国内/国际通用周定义，影响所有周相关的日期计算 |
| 时区处理 | 在 JS 层计算（`DateUtil`） | 不在 SQL 层做 AT TIME ZONE 转换，`resolveDateRange` 返回的 ISO 字符串已考虑时区 |
| 表名机制 | DbTableName 可自定义 | 不强制 bse/tbl 前缀，支持用户指定任意 schema.table |
| 业务表管理 | 动态 Kysely 建表 | 支持用户自定义字段，无需迁移流程 |
| 系统表管理 | Prisma | 元数据结构稳定，享受 Prisma 生态 |

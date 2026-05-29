# 导入解析器字段类型自动推断逻辑

## 概述

导入解析器在导入 CSV/Excel 文件时，会自动推断每列的数据类型。整个推断流程分为三个核心环节：**数据采样** → **类型猜测** → **冲突回退**。

核心代码位于 `apps/nestjs-backend/src/features/import/open-api/import.class.ts`。

---

## 一、数据采样 (Sampling)

### 采样策略

- **采样行数**：默认采样前 500 行（`CsvImporter.CHECK_LINES = 500`）

- **采样位置**：
  - CSV: 通过 PapaParse 的 `preview` 参数限制
  - Excel: 读取全部 sheet 的 `!data` 数据

```typescript
// CSV 采样实现 (import.class.ts:458-469
Papa.parse(stream, {
  download: false,
  dynamicTyping: true,
  preview: CsvImporter.CHECK_LINES,  // 500 行
  // ...
});
```

### 采样数据处理

采样数据通过 `zip(...cols)` 转置后，按列进行类型推断。

---

## 二、类型猜测 (Type Guessing)

### 支持的类型及优先级

类型检测**顺序至关重要**，按**从窄到宽：

| 优先级 | 字段类型 | 说明 |
|---------|---------|------|
| 1 | Checkbox | 最严格 |
| 2 | Number | 数字 |
| 3 | Date | 日期 |
| 4 | LongText | 长文本（含换行）|
| 5 | SingleLineText | 单行文本（默认） |

```typescript
// import.class.ts:209-215
public static readonly SUPPORTEDTYPE: IValidateTypes[] = [
  FieldType.Checkbox,
  FieldType.Number,
  FieldType.Date,
  FieldType.LongText,
  FieldType.SingleLineText,
];
```

### 类型验证 Schema

各类型使用 Zod Schema 验证：

#### 1. Checkbox 验证规则
- 布尔值或字符串 "true"/"false"（不区分大小写）

```typescript
[FieldType.Checkbox]: z.union([z.string(), z.boolean()]).refine(
  (value: unknown) => {
    if (typeof value === 'boolean') return true;
    if (typeof value === 'string') {
      return value.toLowerCase() === 'false' || value.toLowerCase() === 'true';
    }
    return false;
  }
)
```

#### 2. Number 验证规则
- 可被 `Number()` 转换为有效数字

```typescript
[FieldType.Number]: z.any().refine(
  (value) => !isNaN(Number(value))
)
```

#### 3. Date 验证规则
- **白名单正则匹配日期格式（避免误判如 "CC-38716" 这样的字符串
- 年份在合理范围 (1-9999
- 支持格式：
  - `YYYY-MM-DD`
  - `YYYY-MM-DD HH:mm:ss`
  - ISO 8601 格式
  - `DD-MM-YYYY` / `MM-DD-YYYY`
  - `YYYY/MM/DD`
  - `MM/DD/YYYY`

```typescript
const dateFormatPatterns: RegExp[] = [
  /^\d{4}-\d{2}-\d{2}$/,
  /^\d{4}-\d{2}-\d{2}\s+\d{1,2}:\d{2}(?::\d{2})?(?:\.\d{1,3})?$/,
  // ... 更多格式
];

function isValidDateForImport(value: unknown): boolean {
  // 1. 白名单正则匹配
  // 2. Date.parse() 验证
  // 3. 年份范围校验
}
```

#### 4. LongText 验证规则
- 包含换行符 `\n`

```typescript
[FieldType.LongText]: z.string().refine(
  (value) => z.string().safeParse(value) && /\n/.test(value)
)
```

#### 5. SingleLineText 验证规则
- 任意字符串（兜底类型）

---

## 三、冲突回退 (Conflict Fallback)

### 核心算法

**逐步收敛算法**：遍历每列值，**逐步过滤**不符合的类型，直到只剩一个类型。

```typescript
// import.class.ts:307-333
let validatingFieldTypes = [...supportTypes];  // 初始：所有类型
for (let i = 0; i < column.length; i++) {
  if (validatingFieldTypes.length <= 1) {
    break;  // 只剩一个类型，提前终止
  }

  // 跳过空值和表头行
  if (column[i] === '' || column[i] == null || i === 0) {
    continue;
  }

  // LongText 特殊处理：一旦匹配立即终止
  if (validateZodSchemaMap[FieldType.LongText].safeParse(column[i]).success) {
    validatingFieldTypes = [FieldType.LongText];
    break;
  }

  // 过滤：只保留验证通过的类型
  const matchTypes = validatingFieldTypes.filter((type) => {
    const schema = validateZodSchemaMap[type];
    return schema.safeParse(column[i]).success;
  });

  validatingFieldTypes = matchTypes;
}
```

### 回退规则

1. **空列回退**：空列默认回退到 `SingleLineText

```typescript
// import.class.ts:336-338
validatingFieldTypes = !isColumnEmpty
  ? validatingFieldTypes
  : [Importer.DEFAULT_COLUMN_TYPE];  // SingleLineText
```

2. **多值冲突回退**：所有值都无法匹配任何类型时，回退到 `SingleLineText

3. **LongText 快速路径**：一旦发现换行符，立即判定为 LongText，不再继续检测其他类型

### 最终类型确定

取 `validatingFieldTypes[0]` 作为该列的最终类型。

```typescript
// import.class.ts:345
type: validatingFieldTypes[0] || Importer.DEFAULT_COLUMN_TYPE,
```

---

## 四、完整流程示意图

```
┌─────────────────┐
│   数据采样      │
│  (前500行)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  初始化候选类型  │
│ [Checkbox,      │
│  Number,         │
│  Date,           │
│  LongText,       │
│  SingleLineText]   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  逐行验证      │
│  逐个值检测       │
├─────────────────┤
│  ✓ LongText? ──→ 是 → 立即终止
│         │           │
│         ▼ 否        │
│  过滤不匹配类型     │
│         │           │
│         ▼           │
│  只剩1种类型? ──→ 是 → 终止
│         │ 否        │
│         ▼           │
│  下一行 ←──────────┘
│         │
│         ▼ 所有值处理完
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  确定最终类型    │
│  (候选[0]或默认  │
└─────────────────┘
```

---

## 五、关键设计决策

### 1. **顺序优先策略**

类型按严格程度排序，确保最具体的类型优先被匹配。

### 2. **保守推断**

只有当**所有**采样值都符合某类型才会被选中。

### 3. **采样性能优化**

- 提前终止：只剩 类型时立即停止
- LongText 快速路径：换行符立即判定
- 采样限制：500行平衡性能

### 4. **边界情况处理**

| 场景 | 处理方式 |
|------|----------|
| 空列 | SingleLineText |
| 混合类型 | 取最宽泛的匹配类型 |
| 全为空值 | SingleLineText |
| 表头行 | 跳过(i===0) |
| LongText 含换行 | LongText |

---

## 六、Excel 特殊说明

Excel 导入的类型推断与 CSV 基本相同，但 ExcelImporter 有自己的类型顺序（LongText 在最后）。

```typescript
// ExcelImporter 类型顺序
[FieldType.Checkbox,
FieldType.Number,
FieldType.Date,
FieldType.SingleLineText,
FieldType.LongText,
```

---

## 代码位置

| 模块 | 文件路径 | 核心函数 |
|-------|----------|-----------|
| 导入基类 | `import.class.ts | `genColumns()` |
| CSV 导入 | `import.class.ts` | `CsvImporter.parse()` |
| Excel 导入 | `import.class.ts` | `ExcelImporter.parse()` |
| 日期验证 | `import.class.ts` | `isValidDateForImport()` |
| 类型 Schema | `import.class.ts` | `validateZodSchemaMap` |

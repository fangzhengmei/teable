# 导入解析器字段类型自动推断逻辑

## 概述

导入解析器在导入 CSV/Excel 文件时，会自动推断每列的数据类型。整个推断流程分为三个核心环节：**数据采样** → **类型猜测** → **冲突回退**。

核心代码位于 `apps/nestjs-backend/src/features/import/open-api/import.class.ts`。

---

## 一、数据采样 (Sampling)

### 1.1 CSV 与 Excel 采样范围差异

| 维度 | CSV (CsvImporter) | Excel (ExcelImporter) |
|------|------------------|----------------------|
| **采样行数** | 前 500 行 (固定) | **全部行** (无限制) |
| **动态类型转换** | `dynamicTyping: true` | `v.w ?? v.v` (优先格式化文本) |
| **实现方式** | PapaParse `preview` 参数 | XLSX `!data` 全量读取 |
| **Sheet 支持** | 单 Sheet | 多 Sheet (遍历 workbook.Sheets) |

#### CSV 采样实现

```typescript
// import.class.ts:455-469
// 类型推断时调用 parse() 无参数 → 走 preview 500 行分支
return new Promise((resolve, reject) => {
  Papa.parse(stream, {
    download: false,
    dynamicTyping: true,      // 自动转换数字、布尔值
    preview: CsvImporter.CHECK_LINES,  // = 500
    complete: (result) => {
      resolve({
        [CsvImporter.DEFAULT_SHEETKEY]: result.data,
      });
    },
  });
});
```

**关键点**：
- `dynamicTyping: true` 让 PapaParse 自动将 "123" 转为 `number`，"true" 转为 `boolean`
- 这意味着 CSV 的类型推断**基于转换后的值**，而非原始字符串

#### Excel 采样实现

```typescript
// import.class.ts:518-539
const asyncRs = async (stream: NodeJS.ReadableStream): Promise<IParseResult> =>
  new Promise((res, rej) => {
    const buffers: Uint8Array[] = [];
    stream.on('data', function (data) {
      buffers.push(data);
    });
    stream.on('end', function () {
      const buf = Buffer.concat(buffers);
      const workbook = XLSX.read(buf, { dense: true });  // 全量读取
      const result: IParseResult = {};
      Object.keys(workbook.Sheets).forEach((name) => {
        result[name] = workbook.Sheets[name]['!data']?.map((item) =>
          item.map((v) => v.w ?? v.v)  // 优先格式化文本，否则原始值
        ) as unknown[][];
      });
      res(result);
    });
  });
```

**关键点**：
- 无预览限制，**读取整个文件**到大文件性能问题
- `v.w ?? v.v`：优先使用单元格的**格式化文本** (w)，否则用原始值 (v)
- 这意味着 Excel 的类型推断**基于文本表示**，而非单元格的实际类型

### 1.2 采样数据处理

采样数据通过 lodash 的 `zip(...cols)` 进行矩阵转置，从「行优先」转为「列优先」，便于按列进行类型推断：

```typescript
// import.class.ts:301-302
for (const [sheetName, cols] of Object.entries(parseResult)) {
  const zipColumnInfo = zip(...cols);  // 转置：rows → columns
  // ...
}
```

---

## 二、类型猜测 (Type Guessing)

### 2.1 候选类型顺序何时生效？

**候选类型顺序在两处生效**：

#### 生效点 1：初始化候选类型池

```typescript
// import.class.ts:307
let validatingFieldTypes = [...supportTypes];  // 按顺序初始化
```

此时 `validatingFieldTypes` 包含所有支持的类型，**顺序决定了最终优先级**。

#### 生效点 2：最终类型选择

```typescript
// import.class.ts:345
type: validatingFieldTypes[0] || Importer.DEFAULT_COLUMN_TYPE,
```

当有多个类型都通过所有值的验证时，**取数组第一个**（即优先级最高的）。

### 2.2 支持的类型及优先级

#### CSV (Importer 基类) 类型顺序

```typescript
// import.class.ts:209-215
public static readonly SUPPORTEDTYPE: IValidateTypes[] = [
  FieldType.Checkbox,      // 优先级 1 (最严格)
  FieldType.Number,        // 优先级 2
  FieldType.Date,          // 优先级 3
  FieldType.LongText,      // 优先级 4
  FieldType.SingleLineText,// 优先级 5 (最宽泛)
];
```

#### Excel (ExcelImporter) 类型顺序

**关键差异**：ExcelImporter 覆盖了类型顺序，LongText 优先级最低！

```typescript
// import.class.ts:494-500
public static readonly SUPPORTEDTYPE: IValidateTypes[] = [
  FieldType.Checkbox,      // 1
  FieldType.Number,        // 2
  FieldType.Date,          // 3
  FieldType.SingleLineText,// 4 (↑ 上移了)
  FieldType.LongText,      // 5 (↓ 移到最后)
];
```

**为什么 Excel 的 LongText 在最后？**
- Excel 单元格通常不会显式包含换行符
- 避免误判：长文本优先识别为 SingleLineText

### 2.3 类型验证 Schema

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
- **白名单正则匹配**日期格式（避免误判如 "CC-38716" 这样的字符串）
- 年份在合理范围 (1-9999)
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

### 3.1 核心算法：逐步收敛

```typescript
// import.class.ts:306-333
let isColumnEmpty = true;
let validatingFieldTypes = [...supportTypes];  // 初始：所有类型

for (let i = 0; i < column.length; i++) {
  // 优化点：只剩 1 种类型，提前终止循环
  if (validatingFieldTypes.length <= 1) {
    break;
  }

  // 跳过：空值、null、表头行 (i===0)
  if (column[i] === '' || column[i] == null || i === 0) {
    continue;
  }

  // 标记：列非空
  isColumnEmpty = false;

  // ========== LongText 快速路径 ==========
  if (validateZodSchemaMap[FieldType.LongText].safeParse(column[i]).success) {
    validatingFieldTypes = [FieldType.LongText];
    break;  // 立即终止，不再检测其他类型
  }

  // ========== 常规过滤 ==========
  const matchTypes = validatingFieldTypes.filter((type) => {
    const schema = validateZodSchemaMap[type];
    return schema.safeParse(column[i]).success;
  });

  validatingFieldTypes = matchTypes;
}
```

### 3.2 回退条件详解

#### 回退条件 1：LongText 快速路径

**触发时机**：遍历过程中**任意一个值**匹配 LongText

**后果**：
- `validatingFieldTypes` 被直接设为 `[FieldType.LongText]`
- `break` 跳出循环，**不再检测后续值**
- 最终类型一定是 LongText

```typescript
// 只要有一个值包含换行符 → 整列判定为 LongText
if (validateZodSchemaMap[FieldType.LongText].safeParse(column[i]).success) {
  validatingFieldTypes = [FieldType.LongText];
  break;
}
```

**设计意图**：
- 换行符是 LongText 的强特征
- 一旦发现，无需继续检测，提升性能

#### 回退条件 2：空列默认类型

**触发时机**：遍历结束后 `isColumnEmpty === true`

**后果**：
- `validatingFieldTypes` 被替换为 `[Importer.DEFAULT_COLUMN_TYPE]`
- 即 `[FieldType.SingleLineText]`

```typescript
// import.class.ts:336-338
validatingFieldTypes = !isColumnEmpty
  ? validatingFieldTypes
  : [Importer.DEFAULT_COLUMN_TYPE];  // SingleLineText
```

**如何判断空列？**
```typescript
// 初始为 true
let isColumnEmpty = true;

// 遇到非空、非表头值时设为 false
if (column[i] === '' || column[i] == null || i === 0) {
  continue;  // 跳过，不修改 isColumnEmpty
}
isColumnEmpty = false;  // 只要有一个非空值，就不是空列
```

**注意**：表头行 (i===0) 即使有值，也不会影响空列判断！

#### 回退条件 3：所有类型都不匹配

**触发时机**：`validatingFieldTypes` 变成空数组

**后果**：
- `validatingFieldTypes[0]` 为 `undefined`
- 触发 `||` 运算符，回退到 `Importer.DEFAULT_COLUMN_TYPE`

```typescript
// import.class.ts:345
type: validatingFieldTypes[0] || Importer.DEFAULT_COLUMN_TYPE,
```

#### 回退条件 4：多类型共存，取优先级最高的

**触发时机**：遍历结束后 `validatingFieldTypes.length > 1`

**后果**：取数组第一个元素（候选类型顺序决定）

```typescript
// 例如：某列所有值既是 Number 也是 SingleLineText
// validatingFieldTypes = [Number, SingleLineText]
// 最终取 Number（优先级更高）
type: validatingFieldTypes[0]  // Number
```

### 3.3 回退决策树

```
                              ┌─────────────────────┐
                              │  开始遍历列值       │
                              └──────────┬──────────┘
                                         │
                                         ▼
                        ┌─────────────────────────────────┐
                        │  遇到值匹配 LongText?           │
                        └──────────┬─────────┬───────────┘
                                   │ 是      │ 否
                                   ▼         ▼
                        ┌──────────────┐  ┌──────────────────┐
                        │  设为 LongText│  │  过滤不匹配类型  │
                        │  立即终止     │  └─────────┬────────┘
                        └──────┬───────┘            │
                               │                    ▼
                               │          ┌───────────────────────┐
                               │          │  只剩 ≤1 种类型?       │
                               │          └──────────┬────────────┘
                               │                     │ 是        否
                               │                     ▼          │
                               │          ┌──────────────┐       │
                               │          │  提前终止     │       │
                               │          └──────┬───────┘       │
                               │                 │               │
                               └─────────────────┼───────────────┘
                                                 │
                                                 ▼
                              ┌─────────────────────────────────┐
                              │  遍历结束，检查 isColumnEmpty?    │
                              └──────────┬─────────┬───────────┘
                                         │ 是      │ 否
                                         ▼         ▼
                              ┌──────────────┐  ┌──────────────────┐
                              │  SingleLineText│  │  取 validating[0]│
                              └──────┬───────┘  └─────────┬────────┘
                                     │                     │
                                     └──────────┬──────────┘
                                                │
                                                ▼
                                      ┌──────────────────┐
                                      │  最终类型确定     │
                                      └──────────────────┘
```

---

## 四、完整流程示意图

```
┌─────────────────┐
│   数据采样      │
│  CSV: 前500行   │
│  Excel: 全部行  │
└────────┬────────┘
         │
         ▼
┌───────────────────────────────┐
│  初始化候选类型池              │
│  CSV: [Checkbox, Number, Date,│
│        LongText, SingleLineText]│
│  Excel: [Checkbox, Number, Date,│
│          SingleLineText, LongText]│
└────────┬───────────────────────┘
         │
         ▼
┌─────────────────┐
│  逐行验证      │
│  逐个值检测       │
├─────────────────┤
│  ✓ LongText? ──→ 是 → 立即终止 → LongText
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
┌───────────────────────────────┐
│  isColumnEmpty?               │
│  ├─ 是 → SingleLineText       │
│  └─ 否 → 取 validatingFieldTypes[0] │
└───────────────────────────────┘
```

---

## 五、关键设计决策

### 5.1 顺序优先策略

**何时生效？**
1. 初始化候选类型池时
2. 多类型共存时取第一个

**为什么按这个顺序？**
- Checkbox → Number → Date → LongText → SingleLineText
- 从最具体到最宽泛
- 确保 "123" 被识别为 Number 而非 String

### 5.2 保守推断原则

**只有当所有采样值都符合某类型才会被选中。**

例如：
- 100 个值中有 99 个是 Number，1 个是 String
- Number 会被过滤掉（因为有 1 个不匹配）
- 最终可能只剩 SingleLineText

### 5.3 采样性能优化

1. **提前终止**：只剩 1 种类型时立即停止遍历
2. **LongText 快速路径**：发现换行符立即判定并终止
3. **CSV 采样限制**：500 行平衡了准确性与性能

### 5.4 边界情况处理表

| 场景 | 处理方式 | 代码位置 |
|------|----------|----------|
| 空列（所有值为空） | SingleLineText | L336-338 |
| 混合类型（部分值不匹配） | 取仍匹配的最宽泛类型 | L327-332 |
| 所有值都不匹配任何类型 | SingleLineText | L345 |
| 表头行（i===0） | 跳过检测 | L314 |
| 任意值含换行符 | LongText（立即终止） | L322-325 |
| 候选类型池空 | SingleLineText | L345 |

---

## 六、CSV vs Excel 类型推断差异总结

| 维度 | CSV | Excel |
|------|-----|-------|
| **采样行数** | 前 500 行 | 全部行 |
| **动态类型** | PapaParse 自动转换 | 基于文本表示 |
| **LongText 优先级** | 第 4 位（在 SingleLineText 前） | 第 5 位（在最后） |
| **类型推断依据** | 转换后的值类型 | 单元格文本 |
| **Sheet 处理** | 单 Sheet | 多 Sheet 独立推断 |

### 实际影响示例

**示例 1：含换行符的单元格**
- CSV：优先识别为 LongText（第 4 位）
- Excel：优先识别为 SingleLineText（第 5 位，被 SingleLineText 抢占）

**示例 2：大文件（10000 行）**
- CSV：只看前 500 行，可能误判（后 9500 行有特殊值）
- Excel：全部行都看，更准确但更慢

---

## 七、代码位置速查表

| 模块 | 文件路径 | 核心函数/常量 |
|-------|----------|-----------|
| 类型推断主逻辑 | `import.class.ts` | `genColumns()` L295-360 |
| CSV 采样（类型推断用） | `import.class.ts` | `CsvImporter.parse()` 无参数分支 L454-470 |
| Excel 全量采样 | `import.class.ts` | `ExcelImporter.parse()` L510-569 |
| CSV 类型顺序 | `import.class.ts` | `Importer.SUPPORTEDTYPE` L209-215 |
| Excel 类型顺序 | `import.class.ts` | `ExcelImporter.SUPPORTEDTYPE` L494-500 |
| 日期验证函数 | `import.class.ts` | `isValidDateForImport()` L52-74 |
| 类型验证 Schema | `import.class.ts` | `validateZodSchemaMap` L76-105 |
| 空列回退逻辑 | `import.class.ts` | L336-338 |
| LongText 快速路径 | `import.class.ts` | L322-325 |

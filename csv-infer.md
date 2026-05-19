# CSV 字段类型自动推断逻辑分析

## 概述

CSV 上传后的字段类型自动推断是一个三阶段流程：**数据采样 → 类型猜测 → 列定义生成**。整个逻辑主要在 `import.class.ts` 中实现，通过 `genColumns()` 方法统一调度。

---

## 一、完整流程串联图

```
用户上传 CSV
    ↓
[API 层] GET /import/analyze
    ↓
ImportOpenApiService.analyze()
    ↓
importerFactory() → CsvImporter / ExcelImporter
    ↓
Importer.genColumns()  ←───────────┐
    │                              │
    ├─→ parse() 采样前 500 行       │
    │                              │
    ├─→ 逐列类型推断               │
    │   ├─ 初始化候选类型列表       │
    │   ├─ 遍历采样单元格           │
    │   ├─ Zod Schema 验证过滤      │
    │   └─ 确定最终类型             │
    │                              │
    └─→ 生成列定义（name + type） ──┘
    ↓
返回给前端供用户确认/修改
    ↓
用户确认后导入数据
```

---

## 二、第一阶段：数据采样

### 采样策略

**文件位置**：`import.class.ts:454-470`

```typescript
Papa.parse(stream, {
  download: false,
  dynamicTyping: true,  // 自动识别基本类型
  preview: CsvImporter.CHECK_LINES,  // 500 行
  // ...
});
```

| 配置项 | 值 | 说明 |
|--------|----|------|
| `preview` | 500 | 仅采样前 500 行用于类型推断，避免大文件性能问题 |
| `dynamicTyping` | `true` | PapaParse 自动将字符串转换为基本类型（数字、布尔等） |

### dynamicTyping 的双重作用

PapaParse 的 `dynamicTyping: true` 在采样阶段执行**第一层类型识别**：

| 输入字符串 | PapaParse 转换后类型 | 说明 |
|-----------|---------------------|------|
| `"123"` | `number` (123) | 整数自动转数字 |
| `"123.45"` | `number` (123.45) | 浮点数自动转数字 |
| `"1.2e3"` | `number` (1200) | 科学记数法自动转数字 |
| `"true"` | `boolean` (true) | 布尔值自动转换 |
| `"false"` | `boolean` (false) | 布尔值自动转换 |
| `"2024-01-01"` | `string` ("2024-01-01") | **日期不自动转换**，保持字符串 |
| `"null"` | `null` | 特殊值转 null |
| `"undefined"` | `undefined` | 特殊值转 undefined |
| `"hello"` | `string` ("hello") | 普通文本保持字符串 |

### 采样特点

1. **固定采样量**：无论文件多大，只分析前 500 行数据
2. **内存高效**：流式解析，采样完成即停止
3. **局限性**：如果前 500 行数据不具有代表性（如后面行出现不同类型），会导致推断错误
4. **分层识别**：PapaParse 做第一层基础类型转换，后续 Zod Schema 做第二层精细化验证

---

## 三、第二阶段：类型猜测（核心算法）

**文件位置**：`import.class.ts:295-360`

### 3.1 候选类型优先级

类型推断采用**优先级过滤法**，按以下顺序尝试匹配，优先级从高到低：

```typescript
// import.class.ts:209-215
public static readonly SUPPORTEDTYPE: IValidateTypes[] = [
  FieldType.Checkbox,    // 最高优先级
  FieldType.Number,
  FieldType.Date,
  FieldType.LongText,
  FieldType.SingleLineText,  // 最低优先级（兜底）
];
```

**优先级设计意图**：
- 严格类型优先（Checkbox > Number > Date）
- 模糊类型靠后（LongText > SingleLineText）
- SingleLineText 作为最终兜底

### 3.2 Zod Schema 验证规则详解

每种类型对应一个 Zod 验证 Schema，这是**第二层类型识别**：

**文件位置**：`import.class.ts:76-105`

#### 3.2.1 Checkbox 类型

```typescript
[FieldType.Checkbox]: z.union([z.string(), z.boolean()]).refine(
  (value: unknown) => {
    if (typeof value === 'boolean') return true;
    if (typeof value === 'string') {
      const lowered = value.toLowerCase();
      return lowered === 'true' || lowered === 'false';
    }
    return false;
  },
  { message: 'Invalid checkbox value' }
)
```

**识别范围**：
- 原生布尔值：`true`, `false`
- 字符串：`"true"`, `"false"`, `"TRUE"`, `"FALSE"`（不区分大小写）
- **注意**：不识别 `"yes"`, `"no"`, `"1"`, `"0"` 等常见布尔表示

#### 3.2.2 Number 类型（数值识别边界）

```typescript
[FieldType.Number]: z.any().refine(
  (value) => !isNaN(Number(value)),
  { message: 'Invalid number' }
)
```

**核心逻辑**：`Number(value)` 转换后不是 `NaN` 即视为有效数字

**能识别的格式**：

| 输入 | `Number(value)` 结果 | 是否识别为 Number |
|------|---------------------|-------------------|
| `"123"` | 123 | ✅ 是 |
| `"123.45"` | 123.45 | ✅ 是 |
| `"1.2e3"` | 1200 | ✅ 是（科学记数法） |
| `"1.2E+5"` | 120000 | ✅ 是（科学记数法大写 E） |
| `"1.2e-3"` | 0.0012 | ✅ 是（负指数） |
| `"0.1"` | 0.1 | ✅ 是 |
| `".1"` | 0.1 | ✅ 是（省略前导零） |
| `"1."` | 1 | ✅ 是（省略小数部分） |
| `"-123"` | -123 | ✅ 是（负数） |
| `"+123"` | 123 | ✅ 是（正号） |
| `"Infinity"` | Infinity | ✅ 是（特殊值） |
| `"-Infinity"` | -Infinity | ✅ 是（特殊值） |

**不能识别的格式**：

| 输入 | `Number(value)` 结果 | 是否识别为 Number |
|------|---------------------|-------------------|
| `"1,234"` | NaN | ❌ 否（千分位逗号） |
| `"1 234"` | NaN | ❌ 否（空格分隔） |
| `"123元"` | NaN | ❌ 否（含单位） |
| `"￥123"` | NaN | ❌ 否（含货币符号） |
| `"123.45.67"` | NaN | ❌ 否（多个小数点） |
| `"--123"` | NaN | ❌ 否（多个负号） |
| `"12-34"` | NaN | ❌ 否（中间有横杠） |
| `"N/A"` | NaN | ❌ 否（空值标记） |
| `""` | 0 | ✅ 是（但空值会被提前跳过） |
| `" "` | 0 | ✅ 是（但空值会被提前跳过） |

**特殊边界**：
- `Number("")` 返回 `0`，但空字符串在类型推断前会被跳过（`column[i] === ''`）
- `Number(" ")` 返回 `0`，同样会被跳过
- `Number(null)` 返回 `0`，但 `null` 会被跳过（`column[i] == null`）

#### 3.2.3 Date 类型（日期白名单机制）

```typescript
[FieldType.Date]: z.any().refine(isValidDateForImport, { message: 'Invalid date' })
```

**两层验证机制**：
1. **格式白名单**：先用正则匹配 7 种支持的日期格式
2. **有效性校验**：再用 `new Date()` 解析并验证年份范围

**文件位置**：`import.class.ts:38-74`

```typescript
const dateFormatPatterns: RegExp[] = [
  /^\d{4}-\d{2}-\d{2}$/,                                    // 1. YYYY-MM-DD (ISO 日期)
  /^\d{4}-\d{2}-\d{2}\s+\d{1,2}:\d{2}(?::\d{2})?(?:\.\d{1,3})?$/,  // 2. YYYY-MM-DD HH:mm:ss
  /^\d{4}-\d{2}-\d{2}T\d{1,2}:\d{2}(?::\d{2})?(?:\.\d{1,3})?(?:Z|[+-]\d{2}:?\d{2})?$/,  // 3. ISO 8601
  /^\d{1,2}-\d{1,2}-\d{4}$/,                                // 4. DD-MM-YYYY or MM-DD-YYYY
  /^\d{4}\/\d{1,2}\/\d{1,2}$/,                              // 5. YYYY/MM/DD
  /^\d{1,2}\/\d{1,2}\/\d{4}$/,                              // 6. MM/DD/YYYY (美式)
  /^\d{1,2}\/\d{1,2}\/\d{4}\s+\d{1,2}:\d{2}(?::\d{2})?$/,  // 7. MM/DD/YYYY HH:mm:ss (美式+时间)
];
```

**白名单格式详解**：

| 格式 | 示例 | 说明 |
|------|------|------|
| YYYY-MM-DD | `"2024-01-15"` | ISO 标准日期 |
| YYYY-MM-DD HH:mm | `"2024-01-15 14:30"` | 日期+分钟 |
| YYYY-MM-DD HH:mm:ss | `"2024-01-15 14:30:45"` | 日期+秒 |
| YYYY-MM-DD HH:mm:ss.SSS | `"2024-01-15 14:30:45.123"` | 日期+毫秒 |
| ISO 8601 | `"2024-01-15T14:30:45Z"` | 带时区标记 |
| ISO 8601 | `"2024-01-15T14:30:45+08:00"` | 带时区偏移 |
| DD-MM-YYYY | `"15-01-2024"` | 日-月-年 |
| MM-DD-YYYY | `"01-15-2024"` | 月-日-年（无法区分 DD-MM 和 MM-DD） |
| YYYY/MM/DD | `"2024/01/15"` | 斜杠分隔 |
| MM/DD/YYYY | `"01/15/2024"` | 美式日期 |
| MM/DD/YYYY HH:mm:ss | `"01/15/2024 14:30:45"` | 美式日期+时间 |

**日期有效性校验**：

```typescript
function isValidDateForImport(value: unknown): boolean {
  // 1. 空值直接返回 false
  if (value === '' || value == null) return false;

  // 2. 数字类型（Excel 序列号）
  if (typeof value === 'number') {
    if (!Number.isFinite(value)) return false;
    const d = new Date(value);
    if (d.toString() === 'Invalid Date') return false;
    const year = d.getFullYear();
    return year >= 1 && year <= 9999;  // 合理年份范围
  }

  // 3. 字符串类型
  if (typeof value !== 'string') return false;
  const str = value.trim();
  if (!str) return false;
  
  // 关键：先匹配白名单正则，再解析
  if (!dateFormatPatterns.some((p) => p.test(str))) return false;
  
  const d = new Date(value);
  if (d.toString() === 'Invalid Date') return false;
  const year = d.getFullYear();
  return year >= 1 && year <= 9999;
}
```

**设计意图**：
- 避免 JavaScript `Date` 构造函数的宽松解析（如 `"CC-38716"` 被解析为公元 38716 年）
- 只识别明确的、无歧义的日期格式
- 年份限制在 1-9999 之间，排除异常值

#### 3.2.4 LongText 类型

```typescript
[FieldType.LongText]: z.string().refine(
  (value) => z.string().safeParse(value) && /\n/.test(value),
  { message: 'Invalid long text' }
)
```

**识别规则**：
- 必须是字符串
- 必须包含换行符 `\n`
- **短路规则**：只要有一个单元格匹配 LongText，整列立即判定为 LongText，不再继续检查其他单元格

#### 3.2.5 SingleLineText 类型

```typescript
[FieldType.SingleLineText]: z.string()
```

**兜底类型**：
- 只要是字符串就匹配
- 所有类型都不匹配时，最终 fallback 到 SingleLineText

### 3.3 类型推断算法（混合类型收敛过程）

**文件位置**：`import.class.ts:305-333`

```typescript
// 伪代码描述
for each column:
  candidateTypes = [Checkbox, Number, Date, LongText, SingleLineText]
  isColumnEmpty = true
  
  for each cell in column (skip first row = header):
    if candidateTypes.length <= 1:
      break  // 只剩一个候选，无需继续
    
    if cell is empty:
      continue  // 空值不影响类型判断
    
    isColumnEmpty = false
    
    // 特殊规则：LongText 一旦匹配立即确定
    if cell matches LongText:
      candidateTypes = [LongText]
      break
    
    // 过滤：只保留当前单元格能通过验证的类型
    candidateTypes = candidateTypes.filter(type => 
      validateZodSchemaMap[type].safeParse(cell).success
    )
  
  // 空列默认为 SingleLineText
  if isColumnEmpty:
    candidateTypes = [SingleLineText]
  
  columnType = candidateTypes[0]
```

### 3.4 混合类型列收敛行为详解

#### 3.4.1 收敛原理

类型收敛是一个**逐步排除**的过程：
- 初始候选集包含所有 5 种类型
- 每遇到一个单元格，就过滤掉无法通过该单元格验证的类型
- 候选集只剩 1 个类型时，立即停止遍历（提前终止优化）
- 最终取候选集中优先级最高的类型

#### 3.4.2 典型收敛场景示例

**场景 1：纯数字列**
```
单元格值: "100", "200", "300", "400"
步骤:
  初始: [Checkbox, Number, Date, LongText, SingleLineText]
  "100": 过滤 Checkbox → [Number, Date, LongText, SingleLineText]
  "200": 过滤 Date → [Number, LongText, SingleLineText]
  "300": 过滤 LongText → [Number, SingleLineText]
  "400": 候选集仍为 [Number, SingleLineText]
结果: Number（优先级更高）
```

**场景 2：数字+文本混合列**
```
单元格值: "100", "200", "N/A", "400"
步骤:
  初始: [Checkbox, Number, Date, LongText, SingleLineText]
  "100": → [Number, Date, LongText, SingleLineText]
  "200": → [Number, LongText, SingleLineText]
  "N/A": 过滤 Number → [Date, LongText, SingleLineText]
         过滤 Date → [LongText, SingleLineText]
         过滤 LongText → [SingleLineText]
         提前终止！
结果: SingleLineText
```

**场景 3：日期+数字混合列**
```
单元格值: "2024-01-01", "2024-01-02", "12345", "2024-01-04"
步骤:
  初始: [Checkbox, Number, Date, LongText, SingleLineText]
  "2024-01-01": 过滤 Checkbox → [Number, Date, LongText, SingleLineText]
                Date 通过，Number 通过（Number("2024-01-01") = NaN ❌）
                → [Date, LongText, SingleLineText]
  "2024-01-02": → [Date, LongText, SingleLineText]
  "12345": 过滤 Date → [LongText, SingleLineText]
           过滤 LongText → [SingleLineText]
           提前终止！
结果: SingleLineText
```

**场景 4：含 LongText 的列**
```
单元格值: "hello", "world\nline2", "test"
步骤:
  初始: [Checkbox, Number, Date, LongText, SingleLineText]
  "hello": → [Date, LongText, SingleLineText]
  "world\nline2": 匹配 LongText → 立即设置 [LongText]，break！
结果: LongText（短路规则）
```

**场景 5：大部分数字，个别文本**
```
单元格值: "1", "2", "3", ..., "498", "499", "500", "invalid"
（前 499 个是数字，第 500 个是文本）
步骤:
  前 499 个数字: 候选集收敛到 [Number, SingleLineText]
  第 500 个 "invalid": 过滤 Number → [SingleLineText]
结果: SingleLineText
```

**场景 6：空值不影响收敛**
```
单元格值: "100", "", "200", null, "300"
步骤:
  "100": → [Number, Date, LongText, SingleLineText]
  "": 跳过
  "200": → [Number, LongText, SingleLineText]
  null: 跳过
  "300": → [Number, SingleLineText]
结果: Number（空值被跳过，不影响类型判断）
```

#### 3.4.3 收敛过程中的关键规则

| 规则 | 行为 | 影响 |
|------|------|------|
| **提前终止** | 候选集 ≤ 1 时停止遍历 | 性能优化，但可能漏掉后续异常值 |
| **空值忽略** | 空单元格不参与过滤 | 空值不影响类型判断 |
| **LongText 短路** | 匹配 LongText 立即终止 | LongText 优先级最高，一旦出现整列锁定 |
| **优先级兜底** | 候选集取[0] | 优先级高的类型在前面，保证严格类型优先 |

#### 3.4.4 提前终止的局限性

```
CSV 内容:
  行1: 100
  行2: 200
  ...
  行100: 10000  ← 候选集收敛到 [Number, SingleLineText]，提前终止！
  行101: "文本" ← 不会被检查到！
  ...
  行500: "数据"

推断结果: Number（错误）
实际情况: 混合类型，应为 SingleLineText
```

**风险**：前 100 行纯数字导致提前终止，后面的文本值不会被检查到。

### 3.5 关键规则总结

1. **提前终止**：候选类型只剩 1 个时立即停止遍历该列剩余单元格
2. **空值忽略**：空单元格不参与类型判断
3. **首行跳过**：第一行作为表头，不参与类型推断
4. **LongText 短路**：只要有一个单元格包含换行符，整列立即判定为 LongText
5. **空列兜底**：整列为空时默认 SingleLineText
6. **分层识别**：PapaParse dynamicTyping 做第一层，Zod Schema 做第二层

---

## 四、第三阶段：列定义生成

### 4.1 列名生成规则

**文件位置**：`import.class.ts:340-342`

```typescript
const name = getUniqName(
  toString(column?.[0]).trim() || `Field ${index}`, 
  existNames
);
```

规则：
1. 使用第一行（表头行）作为列名
2. 表头为空时使用 `Field {index}` 命名
3. 自动去重（添加后缀确保唯一性）

### 4.2 输出结构

```typescript
// IAnalyzeVo 结构
{
  worksheets: {
    [sheetName]: {
      name: string,           // 工作表名称
      columns: Array<{        // 列定义数组
        type: FieldType,      // 推断出的字段类型
        name: string          // 列名
      }>
    }
  }
}
```

---

## 五、数据导入时的类型转换（Typecast）

类型推断仅用于生成列定义建议，**实际数据导入时会进行第二次类型转换**。

### 5.1 V1 架构导入流程

**文件位置**：`import-csv.processor.ts:230-234`

```typescript
await createFn(
  table.id,
  { fieldKeyType: FieldKeyType.Id, typecast: true, records: cleanRecords },
  false
);
```

**`typecast: true` 的作用**：
- 导入时自动将字符串值转换为目标字段类型
- 如目标字段是 Number，字符串 `"123"` 会被转换为数字 `123`
- 转换失败会触发错误，进入二分查找容错流程

### 5.2 二分查找容错机制

**文件位置**：`import-csv.processor.ts:323-397`

当批量插入失败时，采用二分查找定位失败记录：
1. 先尝试整批插入
2. 失败则分成两半分别尝试
3. 递归直到定位到单个失败记录
4. 记录错误并继续导入其他记录

**性能特征**：N 条记录 K 条失败，需要 O(N/B + K*log(B)) 次 INSERT

### 5.3 V2 架构导入流程

**文件位置**：`ImportRecordsHandler.ts:478-493`

```typescript
private rowToFieldValues(
  row: ReadonlyArray<unknown>,
  sourceColumnMap: SourceColumnMap
): ReadonlyMap<string, unknown> {
  const fieldValues = new Map<string, unknown>();
  for (const [fieldId, columnIndex] of Object.entries(sourceColumnMap)) {
    if (columnIndex === null || columnIndex >= row.length) continue;
    const value = row[columnIndex];
    if (value === null || value === undefined) continue;
    if (typeof value === 'string' && value.length === 0) continue;
    fieldValues.set(fieldId, value);  // 直接使用原始值，不做转换
  }
  return fieldValues;
}
```

**V2 特点**：
- 导入时不做类型转换，直接使用原始值
- 后续在领域层通过 RecordMutationSpec 进行类型验证和转换
- 支持 link/user/attachment 等复杂字段的批量解析

---

## 六、V1 与 V2 架构对比

### V1 架构（完整推断流程）

```
前端上传 → /import/analyze → 后端推断类型 → 返回列定义 → 
前端展示供用户修改 → 用户确认 → 创建表 → 导入数据（typecast: true）
```

**特点**：
- 完整的类型推断在后端 `import.class.ts:genColumns()`
- 用户可在前端修改推断结果
- 类型推断与数据导入分离
- 导入时使用 `typecast: true` 自动转换

### V2 架构（简化版）

**文件位置**：`ImportCsvHandler.ts:305-333`

```typescript
private buildTableFromHeaders(...) {
  // 所有列默认为 SingleLineText 类型
  const fieldBuilder = builder.field().singleLineText().withName(...);
}
```

**特点**：
- 所有列默认创建为 `SingleLineText`
- 无自动类型推断
- 用户需后续手动修改字段类型
- 设计目标：简化流程，避免类型推断错误导致数据丢失

---

## 七、常见问题与边界情况

### 7.1 为什么数字列被推断为文本？

**可能原因**：
1. **前 500 行中存在非数字值**（如 "N/A"、"-"、"--"）
2. **数字包含千分位逗号**（如 "1,234" → `Number("1,234") = NaN`）
3. **数字包含货币符号**（如 "￥123"、"$123"）
4. **数字包含单位**（如 "123kg"、"123px"）
5. **提前终止导致漏检**：前 N 行都是数字，后面出现文本但已停止遍历
6. **PapaParse dynamicTyping 阶段未识别**：某些特殊数字格式可能被保留为字符串

### 7.2 为什么日期列被推断为文本？

**可能原因**：
1. **日期格式不在 7 种白名单内**（如 "2024年1月15日"、"Jan 15, 2024"）
2. **年份超出 1-9999 范围**
3. **前 500 行日期格式不一致**（部分在白名单内，部分不在）
4. **日期包含额外字符**（如 "2024-01-15 (周一)"）
5. **日期为中文格式**（不支持）

### 7.3 为什么所有列都是 SingleLineText？

**可能原因**：
1. 使用了 V2 版本的导入 API（默认不做类型推断）
2. 采样的前 500 行全为空
3. 每列都有至少一个无法识别为 Number/Date/Checkbox/LongText 的值

### 7.4 为什么明明是数字却导入失败？

**可能原因**：
1. 类型推断正确（Number），但某些行实际包含非数字值
2. 导入时 typecast 转换失败（如 "1,234" 无法转为 Number）
3. 数字超出数据库字段范围

### 7.5 科学记数法会被正确识别吗？

**是的**：
- PapaParse `dynamicTyping: true` 阶段会自动转换 `"1.2e3"` → `1200`（number 类型）
- Zod Number 验证 `!isNaN(Number(1200))` 通过
- 最终推断为 Number 类型

**注意**：Excel 中的大数字可能自动显示为科学记数法，CSV 导出时可能变成 `"1.2E+10"` 字符串

### 7.6 类型推断错误怎么办？

**V1 流程**：前端展示推断结果时，用户可手动修改每列类型

**V2 流程**：导入后在字段设置中修改字段类型，系统会自动转换数据

---

## 八、关键代码位置索引

| 功能模块 | 文件路径 | 行号 |
|----------|----------|------|
| 类型推断入口 | `import.class.ts` | 295-360 |
| 数据采样（CSV） | `import.class.ts` | 454-470 |
| Zod 验证 Schema | `import.class.ts` | 76-105 |
| 日期格式白名单 | `import.class.ts` | 38-46 |
| 日期有效性校验 | `import.class.ts` | 52-74 |
| 候选类型优先级 | `import.class.ts` | 209-215 |
| Number 验证规则 | `import.class.ts` | 93-98 |
| LongText 短路逻辑 | `import.class.ts` | 322-325 |
| API 分析接口 | `import-open-api.service.ts` | 107-116 |
| V2 简化导入 | `ImportCsvHandler.ts` | 305-333 |
| V2 字段值映射 | `ImportRecordsHandler.ts` | 478-493 |
| 二分查找容错 | `import-csv.processor.ts` | 323-397 |
| 类型定义 | `packages/openapi/src/import/types.ts` | 1-41 |

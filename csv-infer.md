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

### 采样特点

1. **固定采样量**：无论文件多大，只分析前 500 行数据
2. **内存高效**：流式解析，采样完成即停止
3. **局限性**：如果前 500 行数据不具有代表性（如后面行出现不同类型），会导致推断错误

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

### 3.2 Zod Schema 验证规则

每种类型对应一个 Zod 验证 Schema：

**文件位置**：`import.class.ts:76-105`

| 类型 | 验证规则 |
|------|----------|
| **Checkbox** | 布尔值，或字符串 "true"/"false"（不区分大小写） |
| **Number** | `!isNaN(Number(value))` - 能被解析为数字即可 |
| **Date** | 匹配 8 种日期格式之一，且年份在 1-9999 之间 |
| **LongText** | 包含换行符 `\n` 的字符串 |
| **SingleLineText** | 任意字符串（兜底） |

### 3.3 日期格式白名单

为避免 JavaScript 日期解析的宽松性导致误判，使用正则白名单：

**文件位置**：`import.class.ts:38-46`

```typescript
const dateFormatPatterns: RegExp[] = [
  /^\d{4}-\d{2}-\d{2}$/,           // YYYY-MM-DD
  /^\d{4}-\d{2}-\d{2}\s+\d{1,2}:\d{2}(?::\d{2})?(?:\.\d{1,3})?$/,  // 带时间
  /^\d{4}-\d{2}-\d{2}T\d{1,2}:\d{2}(?::\d{2})?(?:\.\d{1,3})?(?:Z|[+-]\d{2}:?\d{2})?$/,  // ISO 8601
  /^\d{1,2}-\d{1,2}-\d{4}$/,       // DD-MM-YYYY or MM-DD-YYYY
  /^\d{4}\/\d{1,2}\/\d{1,2}$/,     // YYYY/MM/DD
  /^\d{1,2}\/\d{1,2}\/\d{4}$/,     // MM/DD/YYYY
  /^\d{1,2}\/\d{1,2}\/\d{4}\s+\d{1,2}:\d{2}(?::\d{2})?$/,  // 美式日期+时间
];
```

### 3.4 类型推断算法

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

### 3.5 关键规则说明

1. **提前终止**：候选类型只剩 1 个时立即停止遍历该列剩余单元格
2. **空值忽略**：空单元格不参与类型判断
3. **首行跳过**：第一行作为表头，不参与类型推断
4. **LongText 短路**：只要有一个单元格包含换行符，整列立即判定为 LongText
5. **空列兜底**：整列为空时默认 SingleLineText

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

## 五、V1 与 V2 架构对比

### V1 架构（完整推断流程）

```
前端上传 → /import/analyze → 后端推断类型 → 返回列定义 → 
前端展示供用户修改 → 用户确认 → 创建表 → 导入数据
```

**特点**：
- 完整的类型推断在后端 `import.class.ts:genColumns()`
- 用户可在前端修改推断结果
- 类型推断与数据导入分离

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

## 六、常见问题与边界情况

### 6.1 为什么数字列被推断为文本？

**可能原因**：
1. 前 500 行中存在非数字值（如 "N/A"、"-"）
2. 数字包含千分位逗号（如 "1,234"）
3. 科学计数法格式不被识别

### 6.2 为什么日期列被推断为文本？

**可能原因**：
1. 日期格式不在 8 种白名单内
2. 年份超出 1-9999 范围
3. 前 500 行日期格式不一致

### 6.3 为什么所有列都是 SingleLineText？

**可能原因**：
1. 使用了 V2 版本的导入 API（默认不做类型推断）
2. 采样的前 500 行全为空

### 6.4 类型推断错误怎么办？

**V1 流程**：前端展示推断结果时，用户可手动修改每列类型

**V2 流程**：导入后在字段设置中修改字段类型，系统会自动转换数据

---

## 七、关键代码位置索引

| 功能模块 | 文件路径 | 行号 |
|----------|----------|------|
| 类型推断入口 | `import.class.ts` | 295-360 |
| 数据采样（CSV） | `import.class.ts` | 454-470 |
| Zod 验证 Schema | `import.class.ts` | 76-105 |
| 日期格式白名单 | `import.class.ts` | 38-46 |
| 日期有效性校验 | `import.class.ts` | 52-74 |
| 候选类型优先级 | `import.class.ts` | 209-215 |
| API 分析接口 | `import-open-api.service.ts` | 107-116 |
| V2 简化导入 | `ImportCsvHandler.ts` | 305-333 |
| 类型定义 | `packages/openapi/src/import/types.ts` | 1-41 |

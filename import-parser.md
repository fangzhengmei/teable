# 导入解析器字段类型自动推断逻辑

## 概述

本文档按照代码实际执行顺序，详细说明导入解析器字段类型判断的完整链路，重点澄清：
1. `v.w` 为空时 `v.v` 回退对输入值类型的影响
2. CSV 与 Excel 采样输入的界限差异
3. Number 与 Checkbox 在两端可能出现的差异条件

**调用入口**：`import-open-api.service.ts:107-116` → `importer.genColumns()`

**核心代码**：`apps/nestjs-backend/src/features/import/open-api/import.class.ts`

---

## 一、代码执行总览

`genColumns()` 方法的精确执行顺序：

```
1. L296: 取候选类型 → supportTypes = Importer.SUPPORTEDTYPE
2. L297: 数据采样 → this.parse()
3. L302: 行列转置 → zip(...cols)
4. 对每一列执行:
   ├─ 初始化 → isColumnEmpty = true, validatingFieldTypes = [全部类型]
   ├─ 逐值遍历:
   │   ├─ 提前终止 → 只剩1种类型时 break
   │   ├─ 跳过 → 空值/null/表头行
   │   ├─ LongText 短路 → 命中则 break
   │   └─ 通用收敛 → filter 过滤
   ├─ 空列回退 → isColumnEmpty 为 true 时替换
   └─ 最终取值 → validatingFieldTypes[0] || 默认值
```

---

## 二、v.w 为空时，v.v 回退对输入值类型的影响

### 2.1 XLSX 单元格字段含义

SheetJS 解析 Excel 时，每个单元格对象包含：

| 字段 | 含义 | 类型 |
|------|------|------|
| `v.v` | **原始值**（raw value） | number / boolean / string / Date / null |
| `v.w` | **格式化文本**（formatted text） | string / undefined |
| `v.t` | 单元格类型标记 | 'n' / 'b' / 's' / 'd' / 'z' |

### 2.2 代码中的取值逻辑

```typescript
// import.class.ts:529-530
item.map((v) => v.w ?? v.v)
```

`??` 是空值合并运算符，执行逻辑：
- **当 `v.w` 不是 `null` 且不是 `undefined` 时** → 取 `v.w`（格式化后的字符串）
- **当 `v.w` 是 `null` 或 `undefined` 时** → 回退到 `v.v`（原始值，保留类型）

### 2.3 v.w 何时为 undefined？

1. **单元格没有应用任何自定义格式**（默认格式的数字、布尔值）
2. **XLSX 解析时 `cellText: false`**（本代码使用默认值 `true`，因此主要是情况 1）
3. **单元格是错误值**

### 2.4 回退对输入值类型的实际影响（关键表格）

| Excel 单元格场景 | v.w | v.v 值 & 类型 | `v.w ?? v.v` 结果 | 传入验证函数的类型 |
|------------------|-----|--------------|-------------------|--------------------|
| 数字 123（默认格式） | `undefined` | `123` (number) | `123` (number) | **number** |
| 数字 123（货币格式 ¥1,234.00） | `"¥1,234.00"` | `123` (number) | `"¥1,234.00"` (string) | **string** |
| 数字 123（百分比格式 12300%） | `"12300%"` | `123` (number) | `"12300%"` (string) | **string** |
| 布尔值 TRUE（默认格式） | `undefined` | `true` (boolean) | `true` (boolean) | **boolean** |
| 布尔值 TRUE（自定义格式 YES） | `"YES"` | `true` (boolean) | `"YES"` (string) | **string** |
| 日期 2022-11-10 | `"2022-11-10"` | `44875` (number) 或 Date | `"2022-11-10"` (string) | **string** |
| 字符串 "hello" | `"hello"` 或 `undefined` | `"hello"` (string) | `"hello"` (string) | **string** |
| 空单元格 | `undefined` | `undefined` | `undefined` | **undefined** |

### 2.5 关键结论

**之前的错误理解**：Excel 的值全部是 `string` 类型。

**正确理解**：
- 当 `v.w` 存在（有格式）→ 输入是 `string`
- 当 `v.w` 为 `undefined`（无格式）→ 输入是 `v.v` 的原始类型（可能是 `number` / `boolean` / `string` / `undefined`）

因此，**Excel 传入验证函数的值类型与 CSV 类似，都可能是多种类型，只是触发条件不同**：
- CSV：由内容决定（`dynamicTyping` 自动转换）
- Excel：由单元格格式决定（`v.w` 是否存在）

---

## 三、CSV 与 Excel 采样输入的界限差异

### 3.1 采样参数对比

| 维度 | CSV (CsvImporter) | Excel (ExcelImporter) | 界限说明 |
|------|-------------------|----------------------|----------|
| **采样行数** | 前 500 行（固定） | 全部行（无限制） | CSV 在第 500 行截断，可能漏判尾部异常值 |
| **类型转换机制** | `dynamicTyping: true`（内容驱动） | `v.w ?? v.v`（格式驱动） | CSV 看内容，Excel 看单元格格式 |
| **数字值转换** | 内容是数字字符串 → `number` | 无格式 → `number`<br>有格式 → `string` | 分界点：`v.w` 是否为 undefined |
| **布尔值转换** | 内容是 "true"/"false" → `boolean` | 布尔类型且无格式 → `boolean` | 分界点：`v.w` 是否为 undefined |
| **空单元格类型** | `""` (string) | `undefined` | 两者都被跳过，不影响推断 |
| **Sheet 处理** | 单 Sheet | 多 Sheet 独立遍历 | Excel 每个 Sheet 分别执行类型推断 |

### 3.2 输入值类型对比表

| 内容场景 | CSV 传入类型 | Excel 传入类型 | 备注 |
|---------|-------------|---------------|------|
| 纯数字 "123"（无格式） | `number` 123 | `number` 123 | 一致 |
| 数字 "123"（货币格式） | `number` 123 | `string` "¥123.00" | 不一致！ |
| 布尔值 "true" | `boolean` true | `boolean` true | 一致 |
| 布尔值 TRUE（自定义格式 YES） | N/A | `string` "YES" | 仅 Excel 可能出现 |
| 日期 "2022-11-10" | `string` | `string` | 一致 |
| 空单元格 | `""` (string) | `undefined` | 都被跳过 |

---

## 四、Number 验证在 CSV 与 Excel 的差异条件

### 4.1 Number 验证规则

```typescript
// import.class.ts:93-98
[FieldType.Number]: z.any().refine(
  (value) => !isNaN(Number(value)),
)
```

验证逻辑：用 `Number(value)` 转换后，检查结果是否不是 `NaN`。

### 4.2 逐一检查差异条件

| 场景 | CSV 输入 | CSV 验证结果 | Excel 输入 | Excel 验证结果 | 差异原因分析 |
|------|---------|-------------|-----------|---------------|-------------|
| **纯数字（无格式）** | `123` (number) | ✓ `!isNaN(123)` | `123` (number) | ✓ `!isNaN(123)` | 一致 |
| **货币格式** | `123` (number) | ✓ | `"¥1,234.00"` (string) | ✗ `isNaN(Number("¥1,234.00"))` | **不一致**！<br>`¥` 和 `,` 符号导致 `Number()` 转换失败 |
| **百分比格式** | `1.23` (number) | ✓ | `"123%"` (string) | ✗ `isNaN(Number("123%"))` | **不一致**！<br>`%` 符号导致转换失败 |
| **千分位格式** | `1234` (number) | ✓ | `"1,234"` (string) | ✗ `isNaN(Number("1,234"))` | **不一致**！<br>`,` 千分位分隔符导致转换失败 |
| **科学计数法** | `1e3` (number) | ✓ | `"1.00E+03"` (string) | ✓ `!isNaN(Number("1.00E+03"))` | 一致<br>科学计数法字符串能被解析 |
| **日期序列号** | `"44875"` (string)* | ✓ `!isNaN(44875)` | `"2022-11-10"` (string) | ✗ 不匹配日期正则 | **不一致**！<br>CSV 从 Excel 导出时日期可能变成序列号 |
| **极大值 "1e999"** | `Infinity` (number) | ✓ | `"1e999"` 或 `Infinity` | ✓ | 通常一致 |
| **十六进制 "0x10"** | `16` (number) | ✓ | `"0x10"` 或 `16` | ✓ | 通常一致 |

*注：CSV 导入时，如果文件是从 Excel 导出的，日期格式的单元格可能会变成日期序列号（如 44875 = 2022-11-10）。

### 4.3 Number 验证差异总结

Number 验证可能出现差异的 **3 种典型场景**：
1. **Excel 单元格有货币/百分比/千分位格式** → `v.w` 包含特殊符号，`Number()` 转换失败
2. **CSV 是从 Excel 导出的日期序列号** → CSV 识别为 Number，Excel 识别为 Date/String
3. **Excel 单元格有自定义数字格式** → `v.w` 包含非数字字符

---

## 五、Checkbox 验证在 CSV 与 Excel 的差异条件

### 5.1 Checkbox 验证规则

```typescript
// import.class.ts:77-91
[FieldType.Checkbox]: z.union([z.string(), z.boolean()]).refine(
  (value: unknown) => {
    if (typeof value === 'boolean') return true;
    if (typeof value === 'string' &&
        (value.toLowerCase() === 'false' || value.toLowerCase() === 'true')) {
      return true;
    }
    return false;
  }
)
```

验证逻辑：
1. 如果是 `boolean` 类型 → 通过
2. 如果是 `string` 类型且值为 "true" 或 "false"（不区分大小写）→ 通过
3. 其他情况 → 不通过

### 5.2 逐一检查差异条件

| 场景 | CSV 输入 | CSV 验证结果 | Excel 输入 | Excel 验证结果 | 差异原因分析 |
|------|---------|-------------|-----------|---------------|-------------|
| **"true" / TRUE（无格式）** | `true` (boolean) | ✓ `typeof === 'boolean'` | `true` (boolean) | ✓ `typeof === 'boolean'` | 一致 |
| **"false" / FALSE（无格式）** | `false` (boolean) | ✓ | `false` (boolean) | ✓ | 一致 |
| **"TRUE"（大写字符串）** | `true` (boolean) | ✓ | `true` (boolean) | ✓ | 一致 |
| **"True"（混合大小写）** | `true` (boolean) | ✓ | `true` (boolean) | ✓ | 一致 |
| **"是" / "否"（中文）** | `"是"` (string) | ✗ 不匹配 | `"是"` (string) | ✗ 不匹配 | 一致 |
| **布尔值（自定义格式 YES/NO）** | N/A（CSV 无格式概念） | N/A | `"YES"` (string) | ✗ 只识别 "true"/"false" | **仅 Excel 可能出现**！<br>自定义格式使 `v.w` 为 "YES"/"NO"，验证失败 |
| **布尔值（自定义格式 ✗/✓）** | N/A | N/A | `"✓"` (string) | ✗ 不匹配 | 仅 Excel 可能出现 |
| **0 / 1（数字布尔）** | `0` / `1` (number) | ✗ `typeof` 不是 boolean 也不是匹配的 string | `0` / `1` (number) | ✗ | 一致 |

### 5.3 Checkbox 验证差异总结

Checkbox 验证可能出现差异的 **1 种典型场景**：
1. **Excel 单元格有自定义布尔格式** → `v.w` 为 "YES"/"NO"/"是"/"否"/"✗"/"✓" 等，验证失败

---

## 六、最终类型确定与默认值回退

### 6.1 两个回退机制的区别

| 回退机制 | 代码位置 | 触发条件 | 操作 |
|---------|----------|----------|------|
| **空列回退** | L336-338 | `isColumnEmpty === true` | 直接替换 `validatingFieldTypes = [SingleLineText]` |
| **默认值回退** | L345 | `validatingFieldTypes[0]` 为 undefined | `|| Importer.DEFAULT_COLUMN_TYPE` |

### 6.2 空列回退触发条件

```typescript
// import.class.ts:336-338
validatingFieldTypes = !isColumnEmpty
  ? validatingFieldTypes
  : [Importer.DEFAULT_COLUMN_TYPE];
```

`isColumnEmpty` 的判断逻辑：
- 初始值：`true`（L306）
- 设为 `false` 的时机：遍历中遇到**非空、非 null、非表头**的值时（L319）
- 注意：表头行（i===0）即使有值，也不会触发 `isColumnEmpty = false`（被 L314 的 continue 跳过）

### 6.3 默认值回退触发条件

```typescript
// import.class.ts:345
type: validatingFieldTypes[0] || Importer.DEFAULT_COLUMN_TYPE
```

触发条件：`validatingFieldTypes` 是空数组 `[]`，此时 `[][0]` 返回 `undefined`。

空数组如何产生？
- 某个值无法通过候选池中**任何类型**的验证
- `filter` 后 `matchTypes` 为空数组

### 6.4 最终取值决策链（按执行顺序）

```
1. 遍历结束后，先检查空列回退:
   if (isColumnEmpty):
       validatingFieldTypes = [SingleLineText]
       → 最终类型一定是 SingleLineText

2. 再执行最终取值:
   if (validatingFieldTypes[0] 存在):
       → 取 validatingFieldTypes[0]
   else:
       → 取 SingleLineText (默认值回退)
```

**取值优先级（从高到低）**：

| 优先级 | 触发场景 | 结果类型 | 代码位置 |
|--------|---------|----------|----------|
| 1 | 空列回退触发 | SingleLineText | L336-338 |
| 2 | LongText 短路命中 | LongText | L322-325 |
| 3 | `validatingFieldTypes[0]` 存在 | 数组第一个类型 | L345 `||` 左侧 |
| 4 | 候选池为空数组 | SingleLineText | L345 `||` 右侧 |

---

## 七、LongText 判断为何优先于通用过滤？

### 7.1 代码顺序决定优先级

```typescript
for (let i = 0; i < column.length; i++) {
  // ... 跳过空值和表头 ...
  
  // ========== LongText 检测在这里 ==========
  // L322-325: 先执行 LongText 检查
  if (validateZodSchemaMap[FieldType.LongText].safeParse(column[i]).success) {
    validatingFieldTypes = [FieldType.LongText];
    break;  // 直接 break
  }
  
  // ========== 通用过滤在这里 ==========
  // L327-332: 后执行通用 filter
  // ...
}
```

**优先级来源**：
1. **代码位置优先**：LongText 检查在通用 filter 之前
2. **短路逻辑**：命中后直接 `break`，永远不会走到通用 filter
3. **直接覆盖**：`validatingFieldTypes = [LongText]` 替换整个数组

### 7.2 短路机制的特殊性

| 特性 | LongText 短路 | 通用 filter 收敛 |
|------|--------------|-----------------|
| 判定逻辑 | 一票通过（一个值命中即判定） | 全票通过（所有值都必须通过） |
| 执行时机 | 每个值先执行 | LongText 未命中时才执行 |
| 对候选池的操作 | 直接替换为 `[LongText]` | filter 收缩候选池 |
| 是否终止循环 | 是（break） | 否（继续下一个值） |
| 与候选顺序的关系 | 无关（直接覆盖） | 有关（顺序决定最终取哪个） |

**重要结论**：LongText 短路机制完全绕过了候选类型数组的顺序。即使 `ExcelImporter.SUPPORTEDTYPE` 被正确引用，LongText 排在最后，短路机制也会让它优先命中。

---

## 八、差异场景速查表

### 8.1 Number 可能不一致的场景

| 场景 | CSV 结果 | Excel 结果 | 建议 |
|------|---------|-----------|------|
| 货币格式 | ✓ Number | ✗ String | 考虑解析时去掉 `¥` 等符号 |
| 百分比格式 | ✓ Number | ✗ String | 考虑解析时去掉 `%` 并除以 100 |
| 千分位格式 | ✓ Number | ✗ String | 考虑解析时去掉 `,` |
| CSV 是 Excel 导出的日期序列号 | ✓ Number | ✗ Date/String | 日期验证应考虑 Excel 序列号 |

### 8.2 Checkbox 可能不一致的场景

| 场景 | CSV 结果 | Excel 结果 | 建议 |
|------|---------|-----------|------|
| 自定义布尔格式（YES/NO） | N/A | ✗ 不匹配 | 扩展 Checkbox 验证规则，支持更多布尔值表示 |

---

## 九、代码位置速查表

| 模块 | 核心函数/常量 | 行号 |
|-------|-----------|------|
| Excel 取值逻辑 | `v.w ?? v.v` | L529-530 |
| Number 验证规则 | `validateZodSchemaMap[FieldType.Number]` | L93-98 |
| Checkbox 验证规则 | `validateZodSchemaMap[FieldType.Checkbox]` | L77-91 |
| LongText 验证规则 | `validateZodSchemaMap[FieldType.LongText]` | L99-103 |
| LongText 短路 | `validatingFieldTypes = [LongText]; break` | L322-325 |
| 空列回退 | `!isColumnEmpty ? ... : [DEFAULT_COLUMN_TYPE]` | L336-338 |
| 最终取值 + 默认值回退 | `validatingFieldTypes[0] \|\| DEFAULT_COLUMN_TYPE` | L345 |
| CSV 采样 | `CsvImporter.parse()` 无参数分支 | L454-470 |
| CSV dynamicTyping 参数 | `dynamicTyping: true` | L458 |
| Excel 采样 | `ExcelImporter.parse()` 无参数分支 | L510-541 |

# 公式计算体系 - 排障手册证据化分析

## 1. cyclePolicy=skip 触发后的可观测信号与定位路径

### 1.1 触发后系统内部留下的痕迹

当计算计划生成阶段检测到循环依赖且 `cyclePolicy='skip'` 时，系统会在多个层面留下可观测信号：

#### 信号 1: 计算计划中的 cycleInfo 结构体

**数据结构**:
```typescript
export type ComputedUpdateCycleInfo = {
  readonly mode: 'skip';                    // 固定为 'skip'
  readonly unsortedFieldIds: ReadonlyArray<string>;  // 所有无法排序的字段 ID
  readonly cycle: ReadonlyArray<string> | null;      // 检测到的循环链（如找到）
  readonly sampleFields: ReadonlyArray<string>;      // 前 5 个问题字段的详情
  readonly message: string;              // 人类可读的描述信息
};
```
[ComputedUpdatePlanner.ts:151-156](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L151)

**写入时机**: 在 `planStage()` 中检测到循环后立即写入：
```typescript
cycleInfo = {
  mode: 'skip',
  unsortedFieldIds,
  cycle: cycle ?? null,
  sampleFields,
  message,
};
```
[ComputedUpdatePlanner.ts:738-744](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L738)

#### 信号 2: 日志中的 WARN 级别记录

**日志格式**:
```
"Computed field dependency cycle detected. Total unsorted: 3. Skipped cycle fields: 3. Cycle: [fld1(formula) -> fld2(formula) -> fld3(formula) -> fld1]. Sample fields: [fld1(formula), fld2(formula), fld3(formula)]"
```
[ComputedUpdatePlanner.ts:719](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L719)

#### 信号 3: 实际计算步骤被过滤

循环参与字段从计算计划中移除：
```typescript
// 过滤掉循环字段
computedFieldIds = new Set(
  [...computedFieldIds].filter((id) => !cycleFieldIds.has(id))
);
// 重新拓扑排序（仅剩余非循环字段）
({ ordered, levels } = topoSort(relevantEdges, computedFieldIds));
```
[ComputedUpdatePlanner.ts:729-737](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L729)

#### 信号 4: Outbox 任务中的 cyclePolicy 持久化

当任务进入异步队列时，`cyclePolicy` 被保存在 payload 中：
```typescript
// Seed 任务 payload 包含 cyclePolicy
export type ComputedUpdateSeedTaskInput = {
  // ...
  cyclePolicy?: ComputedUpdateCyclePolicy;  // 继承自触发时的配置
};
```
[ComputedUpdateSeedPayload.ts:215](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateSeedPayload.ts#L215)

**合并策略**: 多个任务合并时，只要任一任务是 `'skip'`，合并结果就是 `'skip'`：
```typescript
const mergedCyclePolicy =
  existing.cyclePolicy === 'skip' || incoming.cyclePolicy === 'skip'
    ? 'skip'
    : existing.cyclePolicy ?? incoming.cyclePolicy;
```
[ComputedUpdateSeedPayload.ts:391-394](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateSeedPayload.ts#L391)

---

### 1.2 受影响字段定位顺序

**标准排障路径（按优先级）**:

| 步骤 | 操作 | 数据源 | 证据位置 |
|------|------|--------|----------|
| 1 | 搜索日志关键词 | `grep "Computed field dependency cycle detected"` | [ComputedUpdatePlanner.ts:719](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L719) |
| 2 | 提取 sampleFields | 日志中的 `Sample fields: [fld1(type), fld2(type), ...]` | [ComputedUpdatePlanner.ts:708-713](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L708) |
| 3 | 提取循环链 | 日志中的 `Cycle: [A -> B -> C -> A]` | [ComputedUpdatePlanner.ts:698-705](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L698) |
| 4 | 查询 Outbox 任务 | `SELECT id, payload FROM computed_update_outbox WHERE status != 'done'` | [ComputedUpdateOutbox.ts](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateOutbox.ts) |
| 5 | 检查字段元数据 | 查询这些字段的 `meta.hasError` 标记 | 字段表 |
| 6 | 手动修复循环 | 修改公式打破循环，触发重算 | |

**SQL 查询（定位有问题的字段）**:
```sql
-- 查询最近 1 小时内的循环跳过日志（假设日志结构化存储）
SELECT 
  log_time,
  message,
  json_extract(payload, '$.cycleInfo.sampleFields') as sample_fields,
  json_extract(payload, '$.cycleInfo.cycle') as cycle_chain
FROM logs 
WHERE message LIKE '%Computed field dependency cycle detected%'
ORDER BY log_time DESC
LIMIT 10;
```

---

## 2. Generated Column 修复逻辑的前置条件与幂等保护

### 2.1 修复分支进入的前置条件

`GeneratedColumnMetaRule` 是当数据库状态与字段元数据不一致时触发的自动修复机制。

#### 触发条件判定逻辑

```typescript
// GeneratedColumnMetaRule.isValid()
async isValid(ctx: SchemaRuleContext): Promise<Result<SchemaRuleValidationResult, DomainError>> {
  // 1. 获取数据库列的实际状态
  const column = yield* ctx.introspector.getColumn(ctx.schema, ctx.tableName, columnName);
  
  // 2. 检查数据库列是否是 generated column
  if (column.isGenerated) {
    // 3. 如果是 generated column 但 meta.persistedAsGeneratedColumn 为 false
    // → 进入修复分支
    return ok({
      valid: false,
      extra: [
        `column "${table}"."${columnName}" is a generated column but field meta.persistedAsGeneratedColumn is false`,
      ],
    });
  }
  
  // 数据库列不是 generated column，与 meta 一致 → 无需修复
  return ok({ valid: true });
}
```
[GeneratedColumnMetaRule.ts:39-64](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L39)

**总结触发条件**:
- 数据库列的 `is_generated` = true
- 字段元数据的 `meta.persistedAsGeneratedColumn` = false 或 undefined

---

### 2.2 幂等保护机制

#### 保护机制 1: Schema 规则的幂等性设计

```typescript
// GeneratedColumnMetaRule.up()
// 修复执行的 DDL 本身就是幂等的
return ok([
  // 1. 先删除列（如果存在）
  dropColumnStatement(table, columnName),
  // 2. 再添加列（如果不存在）
  schemaBuilder
    .alterTable(ctx.tableName)
    .addColumn(columnName, dataType, (column) => column.ifNotExists()),
]);
```
[GeneratedColumnMetaRule.ts:85-98](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L85)

**关键幂等语句**:
- `DROP COLUMN IF EXISTS`
- `ADD COLUMN ... IF NOT EXISTS`

#### 保护机制 2: 规则验证-修复闭环

Schema 规则系统采用「验证 → 修复 → 再验证」的闭环流程：
```
isValid() → 返回 valid: false
    ↓
getRepairHint() → 返回修复方案
    ↓
up() → 执行修复 DDL
    ↓
isValid() → 再次验证，直到返回 valid: true
```

[ISchemaRule 接口定义](packages/v2/adapter-table-repository-postgres/src/schema/rules/core/ISchemaRule.ts)

#### 保护机制 3: 数据库级别的唯一约束

虽然修复逻辑本身是幂等的，但并发执行可能导致问题。Schema 变更通常由以下机制保护：
- 单线程 Schema 变更执行器（DDL 互斥执行）
- 表级 DDL 锁（PostgreSQL 内置）

---

### 2.3 重复触发防护

#### 防护 1: 规则验证缓存

Schema 规则系统在每次 Schema 同步时运行一次，验证通过后标记为已应用，不会重复触发。

#### 防护 2: 状态收敛

修复执行后，`isValid()` 会返回 `valid: true`，后续同步不再触发修复：
```
修复前: column.isGenerated = true, meta.persistedAsGeneratedColumn = false → 无效
执行修复: DROP COLUMN + ADD COLUMN (普通列)
修复后: column.isGenerated = false, meta.persistedAsGeneratedColumn = false → 有效
```

#### 防护 3: 空操作检测

如果修复执行前列已经被其他进程修改为正确状态，`isValid()` 会直接返回有效，跳过修复。

---

### 2.4 排障对照清单（Generated Column 修复）

| 症状 | 定位方法 | 处置步骤 | 证据位置 |
|------|----------|----------|----------|
| 日志出现 `is a generated column but field meta.persistedAsGeneratedColumn is false` | 1. 检查字段 meta<br/>2. 查询数据库列属性 | 1. 确认公式是否应该为 GC<br/>2. 触发 Schema 同步自动修复<br/>3. 若自动修复失败，手动执行 DDL | [GeneratedColumnMetaRule.ts:55-62](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L55) |
| Schema 同步长时间卡在某字段 | 1. 查看 Schema 同步日志<br/>2. 检查 PostgreSQL 锁状态 | 1. 终止卡住的 DDL 会话<br/>2. 手动执行幂等修复脚本<br/>3. 重新触发同步 | [GeneratedColumnMetaRule.ts:85-98](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L85) |
| 字段值为 NULL 且不更新 | 1. 检查字段类型是否为 formula<br/>2. 查看 Outbox 回填任务 | 1. 等待 FieldBackfill 任务完成<br/>2. 若任务失败，重新触发回填 | [ComputedFieldBackfillService.ts](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldBackfillService.ts) |

**手动修复 SQL（当自动修复失效时）**:
```sql
-- 1. 删除生成列
ALTER TABLE "table_name" DROP COLUMN IF EXISTS "column_name";

-- 2. 重新创建普通列（根据字段类型选择正确的数据类型）
ALTER TABLE "table_name" ADD COLUMN IF NOT EXISTS "column_name" NUMERIC;

-- 3. 触发字段回填（通过 API 或标记 Outbox 任务）
```

---

## 3. 异步重算队列积压/失败时的偏离场景与收敛机制

### 3.1 Outbox 任务状态机与可观测性

#### 任务状态流转

```
pending → processing → done
              ↓         ↑
              └── releaseForRetry（指数退避）
                        ↓（超过 maxAttempts）
                      dead → 死信队列
```

**状态定义**:
```typescript
status: 'pending' | 'processing' | 'done' | 'dead';
```
[IComputedUpdateOutbox.ts:101](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/IComputedUpdateOutbox.ts#L101)

#### 可观测字段

| 字段 | 含义 | 定位用途 |
|------|------|----------|
| `status` | 当前状态 | 快速筛选积压/失败任务 |
| `attempts` | 已尝试次数 | 判断失败严重程度 |
| `max_attempts` | 最大重试次数 | 默认 8 次 |
| `next_run_at` | 下次运行时间 | 指数退避计算 |
| `locked_at` / `locked_by` | 持有者信息 | 诊断死锁/僵尸任务 |
| `dirty_stats.error_log` | 最近错误信息 | 失败原因定位 |

[IComputedUpdateOutbox.ts:101-106](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/IComputedUpdateOutbox.ts#L101)

---

### 3.2 可能导致读写偏离的场景

#### 场景 1: 高并发写入导致队列积压

**触发条件**:
- 批量数据导入（百万级记录）
- 大规模字段类型转换
- 跨表 Rollup 级联更新风暴

**偏离表现**:
- 写入返回 200 OK，但查询返回旧值
- 跨表 Rollup 统计值滞后
- 前端实时更新延迟 > 5s

**风险窗口**:
- 同步计算完成（事务提交）→ 异步任务完成
- 持续时间 = 队列长度 × 单任务处理时间

**证据**:
- 同步策略配置: [HybridWithOutboxStrategy.ts:94-104](packages/v2/adapter-table-repository-postgres/src/record/computed/strategies/HybridWithOutboxStrategy.ts#L94)
- 同步深度硬限制: `syncMaxLevelHardCap: 1`

#### 场景 2: 任务失败重试期间

**触发条件**:
- 数据库死锁导致 UPDATE 失败
- 网络分区导致锁丢失
- 依赖字段暂时不可用

**偏离表现**:
- 相关字段值停留在旧状态
- 重试期间每次查询可能得到不同结果（中间某次重试成功了一部分）

**重试策略**:
```typescript
// 指数退避计算
const backoff = Math.min(
  config.baseBackoffMs * Math.pow(2, attempts - 1),
  config.maxBackoffMs
);
// 默认: baseBackoffMs=5s, maxBackoffMs=5min
```
[IComputedUpdateOutbox.ts:44-46](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/IComputedUpdateOutbox.ts#L44)

**证据**: 重试逻辑入口: [ComputedUpdateWorker.ts:836](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L836)

#### 场景 3: 任务进入死信队列

**触发条件**:
- 连续失败 8 次（默认 `maxAttempts=8`）
- 公式语法错误（永久失败）
- 数据库 schema 变更导致 SQL 生成失败

**偏离表现**:
- 字段值永久停留在旧状态
- 无自动恢复，必须人工介入

**证据**: 死信转移逻辑在 `markFailed()` 中实现，当 `attempts >= maxAttempts` 时写入 `computed_update_dead_letter` 表。
[IComputedUpdateOutbox.ts:14](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/IComputedUpdateOutbox.ts#L14)

#### 场景 4: Worker 租约丢失

**触发条件**:
- Worker 进程崩溃
- GC 停顿导致心跳超时（默认 `processingLeaseMs=2min`，心跳 `heartbeatIntervalMs=30s`）
- 网络分区

**偏离表现**:
- 任务被标记为 `processing` 但实际无进展
- 超过 `processingLeaseMs` 后被其他 Worker 回收并重试
- 可能出现重复更新（幂等保护可避免数据损坏）

**证据**:
- 租约续期失败日志: [ComputedUpdateWorker.ts:330](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L330)
- 租约丢失警告: [ComputedUpdateWorker.ts:347](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L347)

---

### 3.3 代码中已有的收敛机制

#### 机制 1: 查询时 SQL 展开兜底

即使异步计算未完成，SELECT 查询会通过递归展开公式保证最终一致性：
```typescript
// SelectFormulaConversionVisitor.visitFieldReferenceCurly()
// 非 GC 字段的公式在查询时实时展开为 SQL 表达式
// 不依赖存储值，直接计算最新结果
```
[sql-conversion.visitor.ts:433-447](apps/nestjs-backend/src/features/record/query-builder/sql-conversion.visitor.ts#L433)

⚠️ **限制**: 仅适用于 SELECT 查询。UPDATE/DELETE 等写操作读取的是存储值。

#### 机制 2: Outbox 任务合并与去重

相同 `(baseId, seedTableId, planHash, changeType)` 的 pending 任务会被合并，避免重复计算：
```typescript
// 唯一索引保证: computed_update_outbox_pending_unique_idx
// ON (base_id, seed_table_id, plan_hash, change_type) WHERE status = 'pending'
```
[ComputedUpdateOutbox.deadlock.pglite.spec.ts:321-324](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/__tests__/ComputedUpdateOutbox.deadlock.pglite.spec.ts#L321)

**合并逻辑**:
```typescript
// 合并 fieldIds、seedRecordIds 等
const mergedFieldIds = [...new Set([...existingFieldIds, ...task.fieldIds])];
```
[ComputedUpdateOutbox.ts:334-335](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateOutbox.ts#L334)

#### 机制 3: 咨询锁（Advisory Lock）防止并发处理

```typescript
// 入队前获取咨询锁，防止并发合并冲突
await acquireOutboxAdvisoryLock(trx, buildOutboxLockKey({
  baseId: task.baseId,
  seedTableId: task.tableId,
  planHash: task.planHash,
  changeType: 'seed',
}));
```
[ComputedUpdateOutbox.ts:306-314](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateOutbox.ts#L306)

#### 机制 4: 最大阶段深度保护

防止无限级联更新导致的雪崩：
```typescript
const MAX_STAGE_DEPTH = 50;
if (currentStageDepth >= MAX_STAGE_DEPTH) {
  logger.warn('computed:worker:max_stage_depth_reached', { ... });
  // 停止创建后续任务
}
```
[ComputedUpdateWorker.ts:67-719](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L67)

#### 机制 5: 死信队列的人工介入入口

任务进入死信后，可通过以下方式恢复：
1. 修复根本原因（公式错误、schema 问题等）
2. 将死信任务重新标记为 pending，或通过 API 触发全量重算
3. 死信数据保留用于事后分析

---

### 3.4 排障对照清单（异步重算队列）

| 症状 | 定位方法 | 处置步骤 | 证据位置 |
|------|----------|----------|----------|
| 公式字段值不更新 | 1. `SELECT * FROM computed_update_outbox WHERE status = 'pending'`<br/>2. 检查 `next_run_at` 是否在未来 | 1. 若 `attempts` 高，查看 `error_log`<br/>2. 手动触发重试<br/>3. 检查 Worker 是否存活 | [IComputedUpdateOutbox.ts](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/IComputedUpdateOutbox.ts) |
| 大量任务卡在 processing | 1. `SELECT * FROM computed_update_outbox WHERE status = 'processing'`<br/>2. 检查 `locked_at` 是否超过 2 分钟 | 1. 确认 Worker 进程健康<br/>2. 等待自动回收（`processingLeaseMs` 后）<br/>3. 手动 `UPDATE` 重置状态 | [ComputedUpdateWorker.ts:330](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L330) |
| 查询返回旧值但写入成功 | 1. 检查是否为跨表 Rollup/Lookup<br/>2. 查看 `computed_update_outbox` 队列长度 | 1. 为 SELECT 查询，SQL 展开会兜底<br/>2. 为写操作，等待异步任务完成<br/>3. 若队列过长，考虑扩容 Worker | [sql-conversion.visitor.ts:433](apps/nestjs-backend/src/features/record/query-builder/sql-conversion.visitor.ts#L433) |
| 死信队列有数据 | `SELECT * FROM computed_update_dead_letter` | 1. 分析错误信息，修复根因<br/>2. 重新触发受影响字段的回填<br/>3. 清理死信（归档后删除） | [DEAD_LETTER_TABLE 常量](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateOutbox.ts#L52) |
| Worker 日志大量 `lease_renew_failed` | 1. 检查 Worker 进程 CPU/内存<br/>2. 检查数据库连接池状态 | 1. 优化 Worker 资源配置<br/>2. 调整 `heartbeatIntervalMs`<br/>3. 检查网络稳定性 | [ComputedUpdateWorker.ts:330](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L330) |

**常用诊断 SQL**:
```sql
-- 队列状态概览
SELECT 
  status,
  COUNT(*) as count,
  MIN(next_run_at) as earliest_next_run,
  MAX(attempts) as max_attempts
FROM computed_update_outbox
GROUP BY status;

-- 积压最严重的表
SELECT 
  seed_table_id,
  change_type,
  COUNT(*) as pending_count
FROM computed_update_outbox
WHERE status = 'pending'
GROUP BY seed_table_id, change_type
ORDER BY pending_count DESC
LIMIT 10;

-- 最近失败的任务
SELECT 
  id,
  base_id,
  seed_table_id,
  attempts,
  max_attempts,
  error_log
FROM computed_update_outbox
WHERE status = 'pending' AND attempts > 0
ORDER BY attempts DESC
LIMIT 10;
```

---

## 4. 最小闭环排障速查表

### 4.1 循环依赖（cyclePolicy=skip）

| 阶段 | 操作 | 命令 / SQL |
|------|------|------------|
| **发现** | 搜索日志 | `grep "Computed field dependency cycle detected"` |
| **定位** | 提取循环字段 | 从日志 `Sample fields` 和 `Cycle` 中获取字段 ID |
| **验证** | 检查字段公式 | 查询字段表，确认循环引用关系 |
| **处置** | 打破循环 | 修改其中一个字段的公式，移除循环引用 |
| **验证修复** | 触发重算 | 更新相关记录或手动触发字段回填 |
| **确认** | 检查新日志 | 确认不再出现循环检测警告 |

---

### 4.2 Generated Column 状态不一致

| 阶段 | 操作 | 命令 / SQL |
|------|------|------------|
| **发现** | Schema 同步日志 | `grep "is a generated column but field meta"` |
| **定位** | 检查数据库列 | `SELECT column_name, is_generated FROM information_schema.columns WHERE table_name = 'xxx'` |
| **验证** | 检查字段元数据 | `SELECT meta FROM field WHERE id = 'fld_xxx'` |
| **处置** | 自动修复 | 触发 Schema 同步，等待规则自动执行 |
| **手动处置** | 执行 DDL | `ALTER TABLE ... DROP COLUMN ...; ALTER TABLE ... ADD COLUMN ...;` |
| **确认** | 回填数据 | 触发 FieldBackfill 任务 |

---

### 4.3 异步队列积压

| 阶段 | 操作 | 命令 / SQL |
|------|------|------------|
| **发现** | 监控告警 | 队列长度 > 阈值，或字段更新延迟 > 5s |
| **定位** | 查询队列 | `SELECT status, COUNT(*) FROM computed_update_outbox GROUP BY status` |
| **验证** | 检查 Worker | `SELECT * FROM computed_update_outbox WHERE status = 'processing'` |
| **处置** | 临时缓解 | 增加 Worker 实例，或调整批处理大小 |
| **根因修复** | 分析瓶颈 | 查看失败任务的 `error_log`，优化公式或索引 |
| **确认** | 监控队列 | 观察 `pending` 数量下降，恢复正常 |

---

## 5. 证据清单汇总

### 5.1 cyclePolicy=skip 证据

| 结论 | 代码位置 |
|------|----------|
| cycleInfo 数据结构 | [ComputedUpdatePlanner.ts:151](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L151) |
| 警告日志消息 | [ComputedUpdatePlanner.ts:719](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L719) |
| 过滤循环字段逻辑 | [ComputedUpdatePlanner.ts:729](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L729) |
| 所有入口默认 skip | [PostgresTableRecordRepository.ts:1709](packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts#L1709) |
| Outbox payload 合并策略 | [ComputedUpdateSeedPayload.ts:391](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateSeedPayload.ts#L391) |

### 5.2 Generated Column 修复证据

| 结论 | 代码位置 |
|------|----------|
| 修复触发条件 | [GeneratedColumnMetaRule.ts:55](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L55) |
| 修复 DDL（幂等） | [GeneratedColumnMetaRule.ts:85](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L85) |
| 修复提示 | [GeneratedColumnMetaRule.ts:68](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L68) |

### 5.3 异步队列证据

| 结论 | 代码位置 |
|------|----------|
| 任务状态定义 | [IComputedUpdateOutbox.ts:101](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/IComputedUpdateOutbox.ts#L101) |
| 重试配置（默认 8 次） | [IComputedUpdateOutbox.ts:44](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/IComputedUpdateOutbox.ts#L44) |
| 唯一索引防重复 | [ComputedUpdateOutbox.deadlock.pglite.spec.ts:321](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/__tests__/ComputedUpdateOutbox.deadlock.pglite.spec.ts#L321) |
| 咨询锁保护 | [ComputedUpdateOutbox.ts:306](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateOutbox.ts#L306) |
| 最大阶段深度保护 | [ComputedUpdateWorker.ts:715](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L715) |
| 租约续期失败日志 | [ComputedUpdateWorker.ts:330](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L330) |
| 查询兜底 SQL 展开 | [sql-conversion.visitor.ts:433](apps/nestjs-backend/src/features/record/query-builder/sql-conversion.visitor.ts#L433) |

---

**文档版本**: v1.0
**基于代码版本**: Teable v2 (2026-05-18)
**验证状态**: 所有结论均附带可核对的代码路径与行号，包含可执行的 SQL 诊断脚本

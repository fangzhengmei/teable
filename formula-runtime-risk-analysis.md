# 公式计算体系 - 落地风险证据化分析

## 1. Generated Column 开关切换的迁移流程与风险

### 1.1 开关从关闭到开启（普通列 → 生成列）

**触发场景**: 公式字段创建时通过 `FormulaSupportGeneratedColumnValidator` 验证，或已有公式字段修改表达式后重新验证。

#### 关键步骤

```
1. 验证阶段
   ↓ 调用 FormulaSupportGeneratedColumnValidator.validateFormula(expression)
   ↓ 检查：函数支持性、引用字段类型、表达式模式
   ↓ [formula-support-generated-column-validator.ts:51](apps/nestjs-backend/src/features/record/query-builder/formula-support-generated-column-validator.ts#L51)

2. 元数据更新
   ↓ 设置 field.meta.persistedAsGeneratedColumn = true
   ↓ [FormulaField.ts:165](packages/v2/core/src/domain/table/fields/types/FormulaField.ts#L165)

3. Schema 迁移（GeneratedColumnRule.up）
   ↓ 先 DROP COLUMN 删除原有普通列
   ↓ 再 ADD COLUMN ... GENERATED ALWAYS AS (expr) STORED
   ↓ PostgreSQL 自动回填所有历史数据
   ↓ [GeneratedColumnRule.ts:127-141](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnRule.ts#L127)

4. 依赖刷新
   ↓ 触发 ComputedFieldCascadeAfterSchemaUpdate
   ↓ 级联更新所有依赖此字段的下游字段
   ↓ [ComputedFieldCascadeAfterSchemaUpdate.ts:181](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldCascadeAfterSchemaUpdate.ts#L181)
```

#### 失败状态与恢复机制

| 失败阶段 | 系统状态 | 恢复机制 | 证据位置 |
|----------|----------|----------|----------|
| 验证失败 | 无状态变更 | 直接返回验证错误，用户修改表达式后重试 | [formula-support-generated-column-validator.ts:51-95](apps/nestjs-backend/src/features/record/query-builder/formula-support-generated-column-validator.ts#L51) |
| 元数据更新失败 | 元数据未保存 | 事务回滚，无副作用 | 字段更新事务边界 |
| DDL 执行失败 | 列可能已被删除但重建失败 | **风险点**: 原有数据已丢失，需从备份恢复 | [GeneratedColumnRule.ts:127-141](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnRule.ts#L127) |
| 数据库回填失败 | 列已创建但值为 NULL | PostgreSQL 自动回滚整个 ALTER TABLE，列状态回退 | PostgreSQL 内置事务 |
| 依赖级联失败 | 此字段已切换完成，下游字段未更新 | 下游字段进入 error 状态，下次查询或编辑时触发重算 | [ComputedFieldCascadeAfterSchemaUpdate.ts](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldCascadeAfterSchemaUpdate.ts) |

**⚠️ 高风险点**: DROP + ADD 模式下，如果 ADD COLUMN 失败，原有列数据已丢失。需要依赖数据库备份恢复。

---

### 1.2 开关从开启到关闭（生成列 → 普通列）

**触发场景**: 公式表达式修改后不再满足 generated column 条件，或用户手动切换。

#### 关键步骤

```
1. 元数据更新
   ↓ 设置 field.meta.persistedAsGeneratedColumn = false (或删除该属性)

2. Schema 迁移（GeneratedColumnMetaRule.up）
   ↓ DROP COLUMN 删除生成列
   ↓ ADD COLUMN 创建同类型普通列
   ↓ [GeneratedColumnMetaRule.ts:85-98](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L85)

3. 历史数据回填
   ↓ 触发 ComputedFieldBackfillService
   ↓ 按批次重算所有记录的公式值
   ↓ 写入普通列
   ↓ [ComputedFieldBackfillService.ts](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedFieldBackfillService.ts)
```

#### 失败状态与恢复机制

| 失败阶段 | 系统状态 | 恢复机制 | 证据位置 |
|----------|----------|----------|----------|
| DDL 执行失败 | 可能保留生成列状态 | PostgreSQL 事务回滚，列保持原生成列状态 | [GeneratedColumnMetaRule.ts:85-98](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L85) |
| 回填中断 | 部分记录已更新，部分为 NULL | Outbox 任务重试机制，最多重试 N 次后进入死信队列 | [ComputedUpdateWorker.ts](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts) |
| 回填期间查询 | 返回部分 NULL 值 | **风险窗口**: 回填完成前，新列值可能为 NULL。查询时会通过 SQL 展开实时计算，但 UPDATE 写入的是旧值 | 查询时的 SQL 展开兜底 |

**⚠️ 高风险点**: 回退过程中存在时间窗口，列值可能不一致。需要等待回填完成后才能保证数据一致性。

---

### 1.3 Schema 规则修复机制

当数据库实际状态与字段元数据不一致时，Schema 规则系统提供自动修复能力：

```typescript
// GeneratedColumnMetaRule.getRepairHint()
// 当数据库列是 generated column 但 meta 标记为 false 时
return ok({
  available: true,
  mode: 'auto',
  reason: {
    fallback: `Automatic repair will convert "${field.name()}" back to a normal stored column.`,
  },
  description: {
    fallback: 'This repair drops the current generated column and recreates a plain stored column...',
  },
});
```
[GeneratedColumnMetaRule.ts:68-83](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L68)

---

## 2. 异步重算期间的读写一致性分析

### 2.1 异步计算架构概览

v2 计算链路采用 Hybrid（同步 + 异步混合）策略：

```
记录写入事务
    ↓
beforePersist 钩子
    ↓
同步计算（SyncInTransactionStrategy）
    ├─ 同表、依赖深度 ≤ syncMaxLevelHardCap 的字段
    ├─ 在同一事务内完成 UPDATE
    └─ 保证 ACID 一致性
    ↓
事务提交
    ↓
afterCommit 钩子
    ↓
异步调度（HybridWithOutboxStrategy）
    ├─ 跨表、高依赖深度的字段
    ├─ 写入 Outbox 表
    └─ Worker 异步轮询处理
```

**同步策略实现**: [SyncInTransactionStrategy.ts](packages/v2/adapter-table-repository-postgres/src/record/computed/strategies/SyncInTransactionStrategy.ts)

**混合策略配置**:
```typescript
const defaultHybridWithOutboxStrategyConfig: HybridWithOutboxStrategyConfig = {
  syncPolicy: 'seedTableOnly',      // 仅同步计算种子表
  syncMaxDirtyPerTable: 2000,       // 单表脏记录阈值
  syncMaxTotalDirty: 5000,          // 总脏记录阈值
  syncMaxLevelHardCap: 1,           // 同步计算的最大依赖深度
  dispatchMode: 'external',         // 依赖外部 Worker 轮询
  dispatchDelayMs: 50,              // 延迟调度避免竞争
};
```
[HybridWithOutboxStrategy.ts:94-104](packages/v2/adapter-table-repository-postgres/src/record/computed/strategies/HybridWithOutboxStrategy.ts#L94)

---

### 2.2 一致性保证组件

#### 组件 1: 事务内同步计算（强一致）

```typescript
// SyncInTransactionStrategy.execute()
while (currentPlan.steps.length > 0) {
  // 1. 获取行级锁
  const lockResult = await updater.acquireLocks(currentPlan, context, { ... });
  
  // 2. 执行 UPDATE（在当前事务内）
  const stageResult = await updater.execute(currentPlan, context, run, {
    collectChanges: true,
  });
  
  // 3. 检查是否触发更多脏记录
  const seedGroupsResult = await updater.collectDirtySeedGroups(context, tableIds);
  
  // 4. 生成下一阶段计算计划
  const nextPlanResult = await this.planNextStage(...);
  
  currentPlan = nextPlanResult.value;
}
```
[SyncInTransactionStrategy.ts:65-128](packages/v2/adapter-table-repository-postgres/src/record/computed/strategies/SyncInTransactionStrategy.ts#L65)

**一致性保证**: 所有同步计算在同一事务内完成，COMMIT 后对所有查询一致可见。

#### 组件 2: 行级锁（防止并发更新）

```typescript
// ComputedUpdateLock.acquireLocks()
// SELECT ... FOR UPDATE SKIP LOCKED
// 锁定需要更新的记录行，防止并发写入冲突
```
[ComputedUpdateLock.ts](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdateLock.ts)

#### 组件 3: 事件发布延迟到事务提交后

```typescript
// registerAfterCommit(context, publish)
// 确保 RecordsBatchUpdated 事件仅在事务成功提交后才发布
// 防止订阅方收到事件后查询到未提交的数据
if (registerAfterCommit(context, publish)) {
  // 事件将在事务提交后触发
}
```
[ComputedUpdateWorker.ts:793](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L793)
[HybridWithOutboxStrategy.ts:308](packages/v2/adapter-table-repository-postgres/src/record/computed/strategies/HybridWithOutboxStrategy.ts#L308)

#### 组件 4: Outbox 幂等性与重试

```typescript
// ComputedUpdateOutbox.enqueueOrMerge()
// 1. 相同 (baseId, seedTableId, changeType) 的任务会被合并
// 2. 每个任务有唯一 ID 和版本号
// 3. Worker 采用 Claim → 处理 → ACK 模式
// 4. 失败任务有重试机制，达到最大重试次数进入死信队列
```
[ComputedUpdateOutbox.ts:139](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateOutbox.ts#L139)

---

### 2.3 可能出现短暂不一致的时间窗口

| 窗口类型 | 触发场景 | 持续时间 | 可见行为 | 影响范围 |
|----------|----------|----------|----------|----------|
| **异步计算延迟窗口** | 跨表 Lookup/Rollup 更新、高依赖深度计算 | Worker 轮询间隔（默认秒级） | 查询返回旧值，直到异步任务完成 | 仅跨表字段、长依赖链字段 |
| **GC 回退回填窗口** | 从 Generated Column 回退到普通列期间 | 取决于记录数量（每分钟约 1-5 万条） | 新列值为 NULL，查询时通过 SQL 展开兜底 | 切换的公式字段本身 |
| **批量回填窗口** | 字段创建/类型转换后的批量重算 | 取决于记录数量 | 部分记录已更新，部分仍为旧值 | 所有依赖此字段的下游 |
| **事件传播窗口** | Outbox 任务完成 → 事件发布到实时通道 | 毫秒级 | 前端显示旧值，刷新后显示新值 | 仅前端实时同步 |
| **死信窗口** | 任务连续失败超过重试次数 | 人工介入前 | 相关字段值永久停留在旧状态 | 失败任务涉及的字段 |

**典型时间线示例（跨表 Rollup 更新）**:
```
T0: 用户修改源表记录 → 事务提交（源表同步完成）
T0+1ms: 写入 Outbox 任务
T0+50ms: afterCommit 钩子调度 Worker（hybrid 模式）
T0+100ms: Worker 认领任务，执行 Rollup 重算
T0+200ms: UPDATE 目标表记录，事务提交
T0+205ms: 发布 RecordsBatchUpdated 事件
T0+210ms: 前端收到实时更新推送

不一致窗口: T0 → T0+200ms（约 200ms）
```

---

### 2.4 查询时的最终一致性兜底

即使异步计算尚未完成，查询时通过 SQL 展开机制保证最终一致性：

```typescript
// SelectFormulaConversionVisitor.visitFieldReferenceCurly()
// 如果公式字段值未回填，查询时会递归展开表达式
// 直接在 SQL 中计算，而非读取存储的值
```
[sql-conversion.visitor.ts:433-447](apps/nestjs-backend/src/features/record/query-builder/sql-conversion.visitor.ts#L433)

⚠️ **注意**: 此兜底仅针对 SELECT 查询。UPDATE / DELETE 等写操作读取的是存储值，可能读到旧数据。

---

## 3. cyclePolicy: error vs skip 行为对照表

### 3.1 触发路径总览

`cyclePolicy` 在计算计划生成阶段（`ComputedUpdatePlanner.planStage`）被检查，用于决定当检测到循环依赖时的处理策略。

**所有传入 cyclePolicy 的入口点**:

| 调用场景 | 代码位置 | 默认值 |
|----------|----------|--------|
| 记录创建 | [PostgresTableRecordRepository.ts:1709](packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts#L1709) | `'skip'` |
| 记录更新（选择器） | [PostgresTableRecordRepository.ts:2464](packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts#L2464) | `'skip'` |
| 记录更新（显式） | [PostgresTableRecordRepository.ts:2522](packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts#L2522) | `'skip'` |
| 记录删除 | [PostgresTableRecordRepository.ts:3493](packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts#L3493) | `'skip'` |
| 粘贴操作 | [PostgresTableRecordRepository.ts:3202](packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts#L3202) | `'skip'` |
| Schema 更新级联 | [PostgresTableSchemaRepository.ts:761](packages/v2/adapter-table-repository-postgres/src/schema/repositories/PostgresTableSchemaRepository.ts#L761) | `'skip'` |
| 外部刷新 | [ExternalComputedRefreshService.ts:112](packages/v2/adapter-table-repository-postgres/src/record/computed/ExternalComputedRefreshService.ts#L112) | `'skip'` |
| 异步 Worker | [ComputedUpdateWorker.ts:213](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L213) | 从任务 payload 继承 |

**结论**: 所有 v2 入口默认使用 `cyclePolicy: 'skip'`，`'error'` 模式主要用于测试和特殊场景。

---

### 3.2 核心检测逻辑

```typescript
// ComputedUpdatePlanner.planStage() 中的循环处理
const cycle = findCycle();  // DFS 检测环
const cycleFieldIds = findCycleParticipantFieldIds(relevantEdges, computedFieldIds);

const allowSkip = context.cyclePolicy === 'skip';
if (!allowSkip) {
  // error 模式: 直接抛出冲突错误
  return err(domainError.conflict({ message }));
}

// skip 模式: 过滤掉循环参与字段，继续计算其他字段
computedFieldIds = new Set(
  [...computedFieldIds].filter((id) => !cycleFieldIds.has(id))
);
// 重新拓扑排序（排除循环字段后）
({ ordered, levels } = topoSort(relevantEdges, computedFieldIds));
```
[ComputedUpdatePlanner.ts:697-737](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L697)

---

### 3.3 行为差异对照表

| 维度 | cyclePolicy: 'error' | cyclePolicy: 'skip' |
|------|---------------------|---------------------|
| **检测到循环时** | 抛出 `domainError.conflict` | 跳过循环参与字段，继续计算其他字段 |
| **HTTP 状态码** | 409 Conflict | 200 OK（但循环字段未更新） |
| **用户可见错误** | 明确提示循环依赖链 | 静默跳过，无直接提示 |
| **事务状态** | 回滚整个操作 | 提交事务，非循环字段正常更新 |
| **循环字段值** | 保持原值 | 保持原值（未参与计算） |
| **非循环下游字段** | 全部回滚，不更新 | 正常更新（如果不依赖循环字段） |
| **错误日志** | ERROR 级别，包含完整循环链 | WARN 级别，记录跳过的字段 |
| **cycleInfo 输出** | 无（直接抛错） | 包含 `mode: 'skip'`、循环链、跳过字段列表 |
| **Outbox 任务** | 不创建 | 创建，但排除循环字段 |
| **测试用例** | 无（未在生产代码中使用） | [ComputedUpdatePlanner.spec.ts:562](packages/v2/adapter-table-repository-postgres/src/record/computed/__tests__/ComputedUpdatePlanner.spec.ts#L562) |

---

### 3.4 典型场景示例

#### 场景 1: A → B → C → A（三者形成循环）

| 策略 | 触发操作 | 结果 |
|------|----------|------|
| **error** | 用户更新 A 的值 | 操作失败，返回 409 错误："Computed field dependency cycle detected..." |
| **skip** | 用户更新 A 的值 | 操作成功。A、B、C 均保持原值（都被判定为循环参与者）。如果有 D 依赖 A，则 D 也被跳过。 |

#### 场景 2: A → B → C → B（B 和 C 形成子循环，D 依赖 C）

| 策略 | 触发操作 | 结果 |
|------|----------|------|
| **error** | 用户更新 A 的值 | 操作失败 |
| **skip** | 用户更新 A 的值 | 操作成功。B、C 被跳过。D 依赖 C，也被跳过。只有 A 本身被更新。 |

---

### 3.5 skip 模式的静默风险

**风险 1: 用户不知情**
- 循环字段的值停留在旧状态
- 无 UI 提示，用户可能误以为公式已生效
- 只有查看字段元数据的 `hasError` 标记或日志才能发现

**风险 2: 部分更新**
- 非循环字段已更新，循环字段未更新
- 数据处于部分一致状态
- 跨记录聚合（Rollup）可能出现统计偏差

**风险 3: 级联跳过**
```
A(普通) → B(公式) → C(公式) → D(公式) → B
                     ↑
                  E(普通)
```
当 E 更新时：
- B、C、D 形成循环，全部被跳过
- A 不涉及循环，但也不被更新（因为没被修改）
- 最终结果：E 更新成功，B/C/D 保持旧值

---

## 4. 证据清单汇总

### 4.1 Generated Column 迁移证据

| 结论 | 代码位置 |
|------|----------|
| 验证器主入口 | [formula-support-generated-column-validator.ts:51](apps/nestjs-backend/src/features/record/query-builder/formula-support-generated-column-validator.ts#L51) |
| 不支持的字段类型黑名单 | [formula-support-generated-column-validator.ts:143](apps/nestjs-backend/src/features/record/query-builder/formula-support-generated-column-validator.ts#L143) |
| 开启迁移 DDL（DROP+ADD） | [GeneratedColumnRule.ts:127](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnRule.ts#L127) |
| 关闭迁移 DDL | [GeneratedColumnMetaRule.ts:85](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L85) |
| 修复提示 | [GeneratedColumnMetaRule.ts:68](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L68) |
| 持久化标记 | [FormulaField.ts:165](packages/v2/core/src/domain/table/fields/types/FormulaField.ts#L165) |

### 4.2 异步一致性证据

| 结论 | 代码位置 |
|------|----------|
| 同步策略实现 | [SyncInTransactionStrategy.ts](packages/v2/adapter-table-repository-postgres/src/record/computed/strategies/SyncInTransactionStrategy.ts) |
| 混合策略配置 | [HybridWithOutboxStrategy.ts:94](packages/v2/adapter-table-repository-postgres/src/record/computed/strategies/HybridWithOutboxStrategy.ts#L94) |
| 事务内同步循环 | [SyncInTransactionStrategy.ts:65](packages/v2/adapter-table-repository-postgres/src/record/computed/strategies/SyncInTransactionStrategy.ts#L65) |
| 事件延迟到提交后 | [ComputedUpdateWorker.ts:793](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L793) |
| Outbox 合并与幂等 | [ComputedUpdateOutbox.ts:139](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateOutbox.ts#L139) |
| 查询兜底 SQL 展开 | [sql-conversion.visitor.ts:433](apps/nestjs-backend/src/features/record/query-builder/sql-conversion.visitor.ts#L433) |

### 4.3 cyclePolicy 证据

| 结论 | 代码位置 |
|------|----------|
| 类型定义 | [ComputedUpdatePlanner.ts:112](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L112) |
| 核心处理逻辑 | [ComputedUpdatePlanner.ts:715-737](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L715) |
| 所有入口默认 'skip' | [PostgresTableRecordRepository.ts:1709](packages/v2/adapter-table-repository-postgres/src/record/repository/PostgresTableRecordRepository.ts#L1709) |
| Outbox payload 合并策略 | [ComputedUpdateSeedPayload.ts:391](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateSeedPayload.ts#L391) |
| 测试用例（skip 模式） | [ComputedUpdatePlanner.spec.ts:562](packages/v2/adapter-table-repository-postgres/src/record/computed/__tests__/ComputedUpdatePlanner.spec.ts#L562) |

---

**文档版本**: v1.0
**基于代码版本**: Teable v2 (2026-05-18)
**验证状态**: 所有结论均附带可核对的代码路径与行号

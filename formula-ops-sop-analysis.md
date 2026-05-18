# 公式计算体系值班 SOP（证据化分析版）

> **适用范围**: 值班/On-call 场景下公式计算相关故障的快速定位与处置。所有结论均附代码证据，单一症状只给一条主路径。

---

## 1. 核心阈值常量速查表

值班时最常查询的硬编码阈值，全部来自代码常量定义：

| 阈值项 | 值 | 代码位置 | 说明 |
|--------|----|----------|------|
| 最大重试次数 | 8 | [IComputedUpdateOutbox.ts:44](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/IComputedUpdateOutbox.ts#L44) | 超过即进死信队列 |
| 重试基础退避 | 5000ms | [IComputedUpdateOutbox.ts:45](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/IComputedUpdateOutbox.ts#L45) | 指数退避基数 |
| 重试最大退避 | 300000ms (5min) | [IComputedUpdateOutbox.ts:46](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/IComputedUpdateOutbox.ts#L46) | 退避封顶值 |
| 处理租约时长 | 120000ms (2min) | [IComputedUpdateOutbox.ts:47](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/IComputedUpdateOutbox.ts#L47) | 超时自动回收 |
| 级联深度上限 | 50 | [ComputedUpdateWorker.ts:67](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L67) | 防级联风暴 |
| 内联种子上限 | 5000 | [IComputedUpdateOutbox.ts:43](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/IComputedUpdateOutbox.ts#L43) | 超过则溢写至 seed 表 |

---

## 2. cyclePolicy=skip 最短排障闭环

### 2.1 触发信号链（按观测顺序）

```
用户写入 → Planner 拓扑排序失败 → 检测到循环 → cycleInfo 生成 → WARN 日志 →
跳过循环字段 → 重新拓扑排序 → Outbox 任务持久化（含 cyclePolicy）
```

### 2.2 最短定位路径（3 步）

| 步骤 | 动作 | 观测点 | 代码位置 |
|------|------|--------|----------|
| 1 | **查日志** | 搜索 `Computed field dependency cycle detected` | [ComputedUpdatePlanner.ts:719](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L719) |
| 2 | **提字段** | 从日志提取 `sampleFields`（前 5 个受影响字段）和 `cycle` 链 | [ComputedUpdatePlanner.ts:708](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L708) |
| 3 | **确范围** | 从 `unsortedFieldIds` 确认全部跳过字段（Tarjan 强连通分量） | [ComputedUpdatePlanner.ts:1276](packages/v2/adapter-table-repository-postgres/src/record/computed/ComputedUpdatePlanner.ts#L1276) |

**日志关键字段解析**:
```
Total unsorted: N    → 拓扑排序失败的字段总数
Skipped cycle fields: M  → 实际跳过的循环参与字段数（M ≤ N）
Cycle: [fld1 → fld2 → ...]  → 循环路径（DFS 找到的第一条）
Sample fields: [fld1(type), fld2(type), ...]  → 字段类型辅助识别
```

### 2.3 处置动作（单一路径）

**症状**: 公式字段值不更新，日志有 cycle skip WARN

1. **立即定位**: 从日志提取循环字段 ID 列表
2. **根因修复**: 通知业务方修改公式，打破循环依赖
3. **触发回填**: 修改字段保存时自动触发重算（无需手动操作）
4. **验证**: 观察新写入无 cycle skip 日志，查询字段值正确

> **安全提示**: 循环字段被跳过时，非循环字段仍正常计算。无需立即回滚，先收集字段信息再处置。

---

## 3. Generated Column 修复分支判定矩阵

### 3.1 触发条件矩阵

| 场景 | DB 列 is_generated | 字段 meta.persistedAsGeneratedColumn | 进入分支 | 代码位置 |
|------|-------------------|--------------------------------------|----------|----------|
| **自动修复** | true | false | 修复分支（DROP + ADD） | [GeneratedColumnMetaRule.ts:55](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L55) |
| **正常流程** | false | false | 无需修复 | - |
| **正常流程** | true | true | 无需修复 | - |
| **不一致** | false | true | 另一规则（GeneratedColumnRule）处理 | - |

### 3.2 三类情形判定与处置

#### 情形 A: 自动修复（主路径）

**判定**: `isValid()` 返回 `{ valid: false }` 且 extra 提示 "is a generated column but field meta...false"
[GeneratedColumnMetaRule.ts:52-58](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L52-L58)

**处置**: 系统自动执行，无需人工干预
- DDL: `DROP COLUMN IF EXISTS` + `ADD COLUMN ... IF NOT EXISTS`
- 幂等保护: DDL 本身幂等，重复执行无副作用
- 验证闭环: 修复后重新 `isValid()` 检查，直到通过

#### 情形 B: 幂等重入

**判定**: 同一字段多次触发修复规则（并发或重试场景）

**防护点**:
1. DDL 语句使用 `IF EXISTS` / `IF NOT EXISTS`
   [GeneratedColumnMetaRule.ts:85](packages/v2/adapter-table-repository-postgres/src/schema/rules/field/GeneratedColumnMetaRule.ts#L85)
2. 修复后状态收敛: 一旦 `isValid()` 返回 true，不再触发

**处置**: 观察即可，无需干预

#### 情形 C: 并发竞争

**判定**: 多个请求同时修改同一字段的 GC 状态

**防护点**:
1. 字段修改接口的数据库事务隔离
2. Schema 规则执行在同一事务内
3. 先验证后修复，原子性保证

**处置**: 后提交的请求会因字段状态已变而自动走正确分支，无需人工干预

---

## 4. 异步队列积压与失败重试分流规则

### 4.1 风险优先级排序（值班处理顺序）

```
P0: 死信队列有新增任务 (status=dead)
P1: processing 任务租约超时 (lockedAt < NOW - 2min)
P2: pending 任务持续增长 (队列长度 > 阈值且不下降)
P3: 单表 pending 任务集中 (>1000 条)
P4: 任务重试中 (1 ≤ attempts < 8)
```

### 4.2 各优先级处置动作

#### P0: 死信队列（必须立即处理）

**判定**: `computed_update_dead_letter` 表有新增记录
[ComputedUpdateOutbox.ts:987](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateOutbox.ts#L987)

**信号**: 日志 `computed:outbox:dead_letter`
[ComputedUpdateOutbox.ts:992](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateOutbox.ts#L992)

**处置**:
1. 查询死信表: `SELECT * FROM computed_update_dead_letter ORDER BY created_at DESC LIMIT 10`
2. 分析 `error` 字段确定根因
3. 修复后，将任务从死信表移回 outbox 表（重置 attempts=0, status=pending）
4. 监控队列消费情况

#### P1: 租约超时（可能卡住）

**判定**: `locked_at < NOW() - INTERVAL '2 minutes' AND status = 'processing'`
[IComputedUpdateOutbox.ts:47](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/IComputedUpdateOutbox.ts#L47)

**信号**: Worker 崩溃、GC 停顿、长事务

**处置**:
1. 系统自动回收: `reclaimStaleTasks()` 每 30s 执行一次
2. 若持续超时，检查 Worker 存活与资源占用
3. 必要时重启 Worker 实例

#### P2: 队列积压增长（流量高峰）

**判定**: `pending` 任务数持续 5 分钟增长不下降

**信号**: 监控指标 `outbox_pending_total` 上升

**处置**:
1. 临时止损: 观察查询是否正常（SQL 展开兜底已生效）
   [sql-conversion.visitor.ts:433](apps/nestjs-backend/src/features/record/query-builder/sql-conversion.visitor.ts#L433)
2. 根因分析: 检查是否批量导入、级联风暴
3. 扩容: 增加 Worker 实例数
4. 限流: 如为批量导入导致，可暂停导入任务

#### P3: 单表任务集中

**判定**: 单表 pending 任务 > 1000 条

**信号**: 某表频繁字段修改或批量写入

**处置**:
1. 检查该表是否有复杂公式级联
2. 观察级联深度: 若 `stageDepth` 接近 50，可能存在级联风暴
   [ComputedUpdateWorker.ts:714](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L714)
3. 考虑拆分表或简化公式

#### P4: 重试中（正常现象）

**判定**: `1 ≤ attempts < 8` 且 `nextRunAt` 在合理时间范围内

**信号**: 瞬时网络抖动、死锁冲突

**处置**: 观察即可，指数退避自动重试
- 重试间隔: `min(5s * 2^(attempts-1), 5min)`
  [ComputedUpdateOutbox.ts:996](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateOutbox.ts#L996)

### 4.3 查询结果偏离收敛机制

当队列积压时，查询结果可能与写入预期短时偏离。代码中已有 5 种收敛机制：

| 机制 | 触发场景 | 代码位置 |
|------|----------|----------|
| SQL 展开兜底 | 任何 pending 状态时查询 | [sql-conversion.visitor.ts:433](apps/nestjs-backend/src/features/record/query-builder/sql-conversion.visitor.ts#L433) |
| 任务合并去重 | 同表同字段多次写入 | 唯一索引 `idx_outbox_seed_table_field_unique` |
| 咨询锁防并发 | 多 Worker 抢任务 | [ComputedUpdateOutbox.ts:306](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateOutbox.ts#L306) |
| 级联深度限制 | 级联风暴防护 | [ComputedUpdateWorker.ts:714](packages/v2/adapter-table-repository-postgres/src/record/computed/worker/ComputedUpdateWorker.ts#L714) |
| 死信队列隔离 | 持续失败任务 | [ComputedUpdateOutbox.ts:987](packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateOutbox.ts#L987) |

> **关键认知**: 队列积压 ≠ 数据不一致。查询时 SQL 展开兜底保证最终一致性，只是查询性能下降。

---

## 5. 值班清单：信号 → 判定 → 动作 → 验收

同一症状只给一条主路径，避免决策分叉。

### 5.1 循环依赖类

| # | 信号（观测到什么） | 判定（是什么问题） | 动作（怎么做） | 验收（怎么确认好） |
|---|-------------------|-------------------|----------------|-------------------|
| 1 | 日志有 `Computed field dependency cycle detected` | cyclePolicy=skip 触发，循环字段跳过计算 | 1. 提取 sampleFields 和 cycle 链<br>2. 通知业务方修改公式打破循环<br>3. 字段保存自动触发重算 | 新写入无 cycle skip 日志<br>查询字段值正确 |
| 2 | 日志有 `computed field dependency cycle` 且返回 409 | cyclePolicy=error 触发，写入被拒绝 | 1. 告知用户公式存在循环<br>2. 指导修改公式 | 用户成功保存字段 |

### 5.2 Generated Column 类

| # | 信号（观测到什么） | 判定（是什么问题） | 动作（怎么做） | 验收（怎么确认好） |
|---|-------------------|-------------------|----------------|-------------------|
| 3 | 字段修改日志有 `GeneratedColumnMetaRule` 修复记录 | DB 列与元数据不一致，自动修复 | 观察即可，无需干预 | 修复后 `isValid()` 返回 true<br>字段值正常计算 |
| 4 | 字段保存失败，错误提示 generated column 相关 | 自动修复失败（DDL 执行错误） | 1. 查询具体 DDL 错误<br>2. 检查数据库权限与磁盘空间<br>3. 手动执行修复 DDL | 字段可正常保存 |

### 5.3 异步队列类

| # | 信号（观测到什么） | 判定（是什么问题） | 动作（怎么做） | 验收（怎么确认好） |
|---|-------------------|-------------------|----------------|-------------------|
| 5 | 死信表有新增记录<br>日志 `computed:outbox:dead_letter` | 任务连续失败 8 次，进入死信 | 1. 查询死信表 error 字段<br>2. 修复根因（数据/公式/资源）<br>3. 重置任务回 outbox | 任务重新消费成功<br>死信表无新增 |
| 6 | processing 任务 locked_at > 2min | Worker 租约超时，可能卡住 | 1. 等待自动回收（30s 轮询）<br>2. 检查 Worker 存活 | 任务被重新认领处理<br>processing 任务无超时 |
| 7 | pending 队列持续增长 > 5min | 消费速度 < 生产速度，队列积压 | 1. 确认查询正常（SQL 展开兜底）<br>2. 检查是否批量导入<br>3. 增加 Worker 实例 | 队列长度开始下降<br>最终 pending ≈ 0 |
| 8 | 日志 `computed:worker:max_stage_depth_reached` | 级联深度达到 50 级上限 | 1. 检查是否存在复杂级联公式<br>2. 考虑拆分或简化公式 | 新任务 stageDepth < 50<br>无此 WARN 日志 |

---

## 6. 常用诊断 SQL（值班一键执行）

```sql
-- 6.1 队列状态概览
SELECT status, COUNT(*) as count 
FROM computed_update_outbox 
GROUP BY status;

-- 6.2 积压最严重的 TOP 10 表
SELECT seed_table_id, change_type, COUNT(*) as pending_count 
FROM computed_update_outbox 
WHERE status = 'pending'
GROUP BY seed_table_id, change_type 
ORDER BY pending_count DESC 
LIMIT 10;

-- 6.3 最近失败的任务（含错误信息）
SELECT id, base_id, seed_table_id, attempts, last_error, next_run_at
FROM computed_update_outbox 
WHERE attempts > 0 
ORDER BY attempts DESC, updated_at DESC
LIMIT 20;

-- 6.4 租约超时的 processing 任务
SELECT id, seed_table_id, locked_at, locked_by,
       NOW() - locked_at as locked_duration
FROM computed_update_outbox 
WHERE status = 'processing' 
  AND locked_at < NOW() - INTERVAL '2 minutes';

-- 6.5 死信队列最近 10 条
SELECT id, seed_table_id, error, created_at 
FROM computed_update_dead_letter 
ORDER BY created_at DESC 
LIMIT 10;

-- 6.6 级联深度异常的任务
SELECT id, seed_table_id, stage_depth, attempts
FROM computed_update_outbox 
WHERE stage_depth >= 40  -- 接近 50 上限
ORDER BY stage_depth DESC 
LIMIT 20;
```

---

## 7. 值班 Escalation 路径

```
发现告警 → 匹配值班清单 → 执行处置动作 → 验证修复 →
  ├─ 成功 → 关闭事件，记录处理笔记
  └─ 失败（30 分钟内未恢复） → 升级给公式计算模块负责人
```

**升级触发条件**:
- P0 死信队列 1 小时内无法消化
- P1 租约超时持续发生且 Worker 重启无效
- P2 队列积压超过 10000 条且持续增长
- 业务方反馈大面积公式字段不更新

---

## 附录: 关键日志模式速查

| 日志关键字 | 级别 | 含义 | 处置优先级 |
|-----------|------|------|-----------|
| `Computed field dependency cycle detected` | WARN | 循环依赖触发 skip | P3 |
| `computed field dependency cycle` + 409 | ERROR | 循环依赖触发 error | P2 |
| `computed:outbox:dead_letter` | WARN | 任务进入死信队列 | P0 |
| `computed:worker:max_stage_depth_reached` | WARN | 级联深度达上限 | P2 |
| `GeneratedColumnMetaRule` repair | INFO | GC 自动修复执行 | P4 |
| `computed:outbox:reclaim` | INFO | 租约超时任务回收 | P4 |

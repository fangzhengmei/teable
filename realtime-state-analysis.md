# Teable 实时协同状态更新机制深度分析

本文档梳理 Teable v2 架构中多人编辑同一表格时的实时事件订阅、状态合并及冲突解决机制。

---

## 1. 整体架构概览

Teable 的实时协同系统采用 **分层架构** + **端口-适配器**（Ports & Adapters）模式，核心思想是将"业务领域变更"与"实时传输通道"解耦：

```
┌──────────────────────────────────────────────────────────────┐
│                        前端 (Browser)                         │
│  ┌─────────────────┐    ┌──────────────────────────────┐     │
│  │  React Hooks     │    │  BroadcastChannelRealtimeEngine │  │
│  │  (use-table等)  │◄──►│  (同浏览器标签间同步)            │  │
│  └────────┬────────┘    └──────────────────────────────┘     │
│           │ WebSocket                                       │
└───────────┼──────────────────────────────────────────────────┘
            │
┌───────────┼──────────────────────────────────────────────────┐
│        后端 (Node.js / NestJS)                                │
│           │                                                   │
│  ┌────────▼──────────────────────────────────────────────┐    │
│  │  ShareDbService (NestJS)                              │    │
│  │  ┌─────────────────────────────────────────────────┐ │    │
│  │  │  ShareDB Server + Redis Pub/Sub                  │ │    │
│  │  │  (多实例间实时同步)                                │ │    │
│  │  └─────────────────────────────────────────────────┘ │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  V2 领域层 (DDD)                                      │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │    │
│  │  │  CommandBus  │  │  EventBus    │  │  Projection│ │    │
│  │  │  (命令处理)  │  │  (事件分发)  │  │  (状态转换)│ │    │
│  │  └──────────────┘  └──────────────┘  └────────────┘ │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  V2 实时引擎 (Adapter)                                │    │
│  │  ┌────────────────────┐  ┌────────────────────────┐  │    │
│  │  │ ShareDbRealtimeEngine│  │ BroadcastChannel Engine│  │    │
│  │  │ (服务端→ShareDB)     │  │ (浏览器端→BC)         │  │    │
│  │  └────────────────────┘  └────────────────────────┘  │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. 核心接口定义 (Ports 层)

### 2.1 `IRealtimeEngine` — 实时引擎抽象

文件：`packages/v2/core/src/ports/RealtimeEngine.ts`

```typescript
export interface IRealtimeEngine {
  ensure(context, docId, initial): Promise<Result<void, DomainError>>;
  applyChange(context, docId, change, options?): Promise<Result<void, DomainError>>;
  delete(context, docId): Promise<Result<void, DomainError>>;
}
```

三个核心方法构成了实时协同的全部语义：

| 方法 | 语义 | 典型场景 |
|------|------|---------|
| `ensure` | 确保文档存在并设置初始快照 | 表创建、字段创建、记录创建 |
| `applyChange` | 对文档应用增量变更 | 字段更新、记录更新、记录重排序 |
| `delete` | 删除文档 | 字段删除、记录删除 |

### 2.2 `RealtimeDocId` — 文档标识

文件：`packages/v2/core/src/ports/RealtimeDocId.ts`

采用 `collection/docId` 格式，用于在实时通道中唯一定位文档：

- `tbl_{baseId}/{tableId}` — 表级文档
- `fld_{tableId}/{fieldId}` — 字段级文档
- `rec_{tableId}/{recordId}` — 记录级文档
- `viw_{tableId}/{viewId}` — 视图级文档

### 2.3 `RealtimeChange` — 增量变更类型

文件：`packages/v2/core/src/ports/RealtimeChange.ts`

三种变更类型覆盖了表格协同编辑的所有场景：

```typescript
type RealtimeChange =
  | { type: 'set';    path: RealtimePath; value: unknown; oldValue?: unknown }  // 路径替换
  | { type: 'insert'; path: RealtimePath; index: number; value: unknown }         // 数组插入
  | { type: 'delete'; path: RealtimePath; index: number; count: number };         // 数组删除
```

`RealtimePath` 是 `string | number` 的只读数组，表示从文档根到目标属性的路径（类似 JSON Pointer）。

---

## 3. 事件订阅与分发机制

### 3.1 `AsyncMemoryEventBus` — 异步事件总线

文件：`packages/v2/core/src/ports/memory/AsyncMemoryEventBus.ts`

这是整个事件系统的核心调度器，负责：

1. **事件入队**：所有领域事件通过 `publish`/`publishMany` 进入队列，获得递增序号
2. **异步调度**：使用 `setImmediate`/`setTimeout(0)`/`queueMicrotask` 调度事件的处理，避免阻塞命令执行
3. **处理器分发**：根据事件类型查找已注册的处理器，并按序调用

#### 关键实现细节

**事件排队（FIFO 有序处理）：**

```typescript
private enqueue(context, events): number {
  let targetSeq = this.processedSeq;
  for (const event of events) {
    const seq = this.nextSeq;
    this.nextSeq += 1;
    targetSeq = seq;
    this.queue.push({ context, event, seq });
  }
  if (!this.draining) {
    this.draining = true;
    this.scheduleDrain();
  }
  return targetSeq;
}
```

事件按 FIFO 顺序处理，保证因果一致性。

**投影并发分组策略：**

```typescript
const flushProjectionGroup = async () => {
  if (shouldDispatchProjectionGroupConcurrently || currentGroup.length === 1) {
    await Promise.all(
      currentGroup.map(handlerToken => this.dispatchToHandler(context, event, handlerToken))
    );
  } else {
    for (const handlerToken of currentGroup) {
      await this.dispatchToHandler(context, event, handlerToken);
    }
  }
};
```

- 标记为 `@ProjectionHandler`（role=`projection`）的处理器会被分组
- 连续的投影处理器并发执行（`Promise.all`）
- 非投影处理器创建**顺序边界**：先等待前面的投影组完成，再执行自身，之后的投影组开启新的并发组
- **大批量事件降级**：`RecordsBatchUpdated` 超过阈值（默认 1000）时，投影组退化为串行执行，避免内存压力

**等待特定事件处理完成：**

```typescript
private waitUntilProcessed(targetSeq: number): Promise<void> {
  if (targetSeq <= this.processedSeq) return Promise.resolve();
  return new Promise(resolve => {
    this.waiters.push({ targetSeq, resolve });
  });
}
```

对于字段创建、更新、删除等关键操作，发布者可以选择 `await` 直到事件处理完成，确保实时通道已同步。

### 3.2 事件处理器注册

文件：`packages/v2/core/src/ports/EventHandler.ts`

使用装饰器模式注册事件处理器：

```typescript
@EventHandler(RecordCreated)  // 或 @ProjectionHandler(RecordCreated)
class RecordCreatedRealtimeProjection implements IEventHandler<RecordCreated> {
  async handle(context, event): Promise<Result<void, DomainError>> { ... }
}
```

`ProjectionHandler` 是 `EventHandler` 的别名，自动标记 `role: 'projection'`。

### 3.3 需等待的事件类型

```typescript
private shouldAwait(events): boolean {
  const awaitableEventNames = new Set([
    'FieldCreated', 'FieldUpdated', 'FieldDeleted',
    'FieldDuplicated', 'FieldOptionsAdded', 'ViewColumnMetaUpdated',
  ]);
  return events.every(event => awaitableEventNames.has(event.name.toString()));
}
```

字段结构变更（创建/更新/删除）需要等待实时同步完成，因为后续的记录操作依赖字段元数据。

---

## 4. 状态合并机制 — Projection 层

Projection 是连接 **领域事件** 和 **实时通道** 的桥梁，负责将领域事件转换为 `RealtimeChange` 并提交给 `IRealtimeEngine`。

### 4.1 Projection 分类与职责

| Projection | 监听事件 | 操作 | 文档类型 |
|-----------|---------|------|---------|
| `TableCreatedRealtimeProjection` | `TableCreated` | `ensure` | `tbl_` |
| `FieldCreatedRealtimeProjection` | `FieldCreated` | `ensure` | `fld_` |
| `FieldUpdatedRealtimeProjection` | `FieldUpdated` | `applyChange` (set) | `fld_` |
| `FieldDeletedRealtimeProjection` | `FieldDeleted` | `delete` | `fld_` |
| `FieldOptionsAddedRealtimeProjection` | `FieldOptionsAdded` | `applyChange` (set) | `fld_` |
| `ViewColumnMetaUpdatedRealtimeProjection` | `ViewColumnMetaUpdated` | `applyChange` (set) | `viw_` |
| `RecordCreatedRealtimeProjection` | `RecordCreated` | `ensure` | `rec_` |
| `RecordUpdatedRealtimeProjection` | `RecordUpdated` | `applyChange` (set) | `rec_` |
| `RecordsBatchCreatedRealtimeProjection` | `RecordsBatchCreated` | `ensure` (批量) | `rec_` |
| `RecordsBatchUpdatedRealtimeProjection` | `RecordsBatchUpdated` | `applyChange` (set, 批量) | `rec_` |
| `RecordsDeletedRealtimeProjection` | `RecordsDeleted` | `delete` (批量) | `rec_` |
| `RecordReorderedRealtimeProjection` | `RecordReordered` | `applyChange` (set) | `rec_` |

### 4.2 记录创建 Projection 示例

文件：`packages/v2/core/src/application/projections/RecordCreatedRealtimeProjection.ts`

```typescript
@ProjectionHandler(RecordCreated)
class RecordCreatedRealtimeProjection {
  async handle(context, event) {
    // 1. 构造文档ID
    const docId = RealtimeDocId.fromParts(`rec_${event.tableId}`, event.recordId);
    // 2. 转换字段值为扁平映射
    const fields: Record<string, unknown> = {};
    for (const fieldValue of event.fieldValues) {
      fields[fieldValue.fieldId] = fieldValue.value;
    }
    // 3. 发布到实时引擎
    realtimeEngine.ensure(context, docId, { id, fields });
  }
}
```

### 4.3 记录更新 Projection — 增量变更

文件：`packages/v2/core/src/application/projections/RecordUpdatedRealtimeProjection.ts`

关键设计决策：更新使用 `applyChange`（增量操作）而不是 `ensure`（全量快照）：

```typescript
for (const change of event.changes) {
  realtimeEngine.applyChange(context, docId, {
    type: 'set',
    path: ['fields', change.fieldId],
    value: newValue,
    ...(oldValue === undefined ? {} : { oldValue }),
  }, { version: event.oldVersion });
}
```

- **增量 vs 全量**：更新只推送变更字段，避免覆盖客户端已有的其他字段数据
- **版本号传递**：`oldVersion` 传递给 ShareDB，用于 OT 操作排序
- **oldValue 传递**：提供旧值用于冲突检测和历史回溯

### 4.4 字段更新 Projection — 路径深度更新

文件：`packages/v2/core/src/application/projections/FieldUpdatedRealtimeProjection.ts`

字段更新比记录更新更复杂，因为字段属性嵌套在文档中（如 `options.formatting.date`）：

```typescript
const buildFieldRealtimeChanges = (fieldDto, event) => {
  const fieldChanges: RealtimeChange[] = [];
  for (const property of event.updatedProperties) {
    const path = event.realtimePathFor(property);  // 如 ['options', 'formatting', 'date']
    fieldChanges.push({
      type: 'set',
      path: [...path],
      value: nextValue,
      oldValue: ...,
    });
  }
  // 同时处理形状刷新（如选项变更导致数据迁移）
  fieldChanges.push(...buildFieldShapeRefreshChanges(fieldDto, event, seenPaths));
  return fieldChanges;
};
```

### 4.5 批量操作优化

文件：`packages/v2/core/src/application/projections/BatchRecordRefreshPolicy.ts`

大批量记录操作的特殊处理策略：

```typescript
const DEFAULT_LARGE_RECORD_BATCH_REFRESH_THRESHOLD = 1000;

export const shouldSkipRealtimeBatchMutation = (size): boolean => {
  return size >= DEFAULT_LARGE_RECORD_BATCH_REFRESH_THRESHOLD;
};
```

当批量操作超过 1000 条记录时：
- 跳过逐条实时推送
- 改为触发**表级刷新**，由前端整体重新拉取数据
- 避免 Redis 压力雪崩和内存膨胀

### 4.6 批量操作的并发控制

文件：`packages/v2/core/src/application/projections/runRealtimeTasks.ts`

批量记录操作使用任务数组并行执行，但每个记录的变更按序累积：

```typescript
for (const update of event.updates) {
  const previous = tasksByRecord.get(update.recordId);
  const next = async () => {
    if (previous) {
      const previousResult = await previous();
      if (previousResult.isErr()) return previousResult;
    }
    // ... 应用变更
  };
  tasksByRecord.set(update.recordId, next);
}
// 所有记录并行执行
for (const result of await runRealtimeTasks(Array.from(tasksByRecord.values()))) {
  result._unsafeUnwrap();
}
```

---

## 5. 冲突解决机制

### 5.1 ShareDB OT 冲突解决

ShareDB 基于 **操作转换（Operational Transformation）**，使用 `json0` 操作类型。

#### 操作转换原理

当两个客户端同时修改同一文档时：

1. 操作携带版本号 `v`，表示基于文档的哪个版本
2. 服务器按版本号排序操作
3. 如果操作基于旧版本，ShareDB 自动将操作**转换**（transform）到最新版本
4. 转换后的操作应用到文档，产生一致的最终状态

#### json0 操作映射

`ShareDbRealtimeEngine` 将 `RealtimeChange` 映射到 json0：

```typescript
private toJson0Op(change: RealtimeChange): unknown[] {
  const path = [...change.path];
  switch (change.type) {
    case 'set':
      // set 替换 → json0 对象替换 { p: path, oi: newValue, od: oldValue }
      if (oldValue !== undefined) {
        return [{ p: path, oi: change.value, od: change.oldValue }];
      }
      return [{ p: path, oi: change.value }];
    case 'insert':
      // 数组插入 → json0 列表插入 { p: path.concat(index), li: value }
      return [{ p: [...path, change.index], li: change.value }];
    case 'delete':
      // 数组删除 → json0 列表删除（从后往前，保持索引有效）
      return ops.map(i => ({ p: [...path, change.index + i], ld: true }));
  }
}
```

### 5.2 操作来源过滤 — 防止无限循环

文件：`apps/nestjs-backend/src/share-db/share-db.service.ts`

V2 投影产生的操作带有特殊来源标识，ShareDB submit 中间件识别后跳过处理：

```typescript
private onSubmit = (context, next) => {
  const submitSource = context.options?.source;
  if (submitSource === '@@v2-projection') return next();  // V2 投影操作，跳过

  const opSource = context.op.src;
  if (opSource.startsWith('@@v2-projection:')) return next();  // 带 requestId 的投影操作

  // 客户端操作才进行验证和处理
  if (!hasClientStream(context.agent)) return next();
  // ... 验证文档类型（只允许记录操作）
  next();
};
```

这确保了：
- 领域事件 → Projection → ShareDB 的单向流动
- ShareDB 不会将 Projection 操作当作客户端操作重新处理

### 5.3 乐观并发控制

文件：`packages/v2/adapter-table-repository-postgres/src/record/computed/outbox/ComputedUpdateOutbox.ts`

Postgres 层使用**咨询锁（Advisory Lock）**+**版本号**实现乐观并发控制：

```typescript
// 每个记录的计算字段更新使用 advisory lock 防止冲突
pg_try_advisory_xact_lock(...)
```

计算字段（公式、查找、汇总）的更新通过 Outbox 模式异步处理，避免与实时操作竞争。

### 5.4 版本号递增

每个领域事件携带 `oldVersion`，ShareDB 使用它来排序操作：

```typescript
realtimeEngine.applyChange(context, docId, change, { version: event.oldVersion });
```

文档版本单调递增（+1 per operation），客户端订阅时可检测到版本跳变。

### 5.5 冲突场景分析

| 场景 | 解决机制 | 代码位置 |
|------|---------|---------|
| 两人同时修改同一记录的不同字段 | ShareDB OT 自动转换 | `ShareDbRealtimeEngine.toJson0Op()` |
| 两人同时修改同一记录的同一字段 | 后提交者的操作被转换到最新版本，基于前一次结果重新计算 | ShareDB 内置 |
| 字段删除后记录更新 | Projection 顺序保证：FieldDeleted 先于 RecordUpdated | `AsyncMemoryEventBus` FIFO |
| 记录删除后更新 | ShareDB 文档删除后操作失败，前端重新拉取 | ShareDB 内置 |
| 批量导入覆盖单条编辑 | 批量操作跳过实时推送（>1000条），前端刷新 | `shouldSkipRealtimeBatchMutation` |
| 计算字段与手动修改竞争 | 计算字段使用 Outbox 异步更新 + 咨询锁 | `ComputedUpdateOutbox` |

---

## 6. 实时引擎适配器实现

### 6.1 ShareDB 引擎 (服务端)

文件：`packages/v2/adapter-realtime-sharedb/src/ShareDbRealtimeEngine.ts`

```
领域事件 → Projection → RealtimeChange → json0 op → ShareDB → WebSocket → 客户端
```

**发布流程：**

```typescript
async applyChange(context, docId, change, options?) {
  // 1. 解析 docId → collection + documentId
  const { collection, docId: documentId } = RealtimeDocIdValue.parse(docId);
  // 2. 转换为 json0 操作
  const json0Op = changes.flatMap(item => this.toJson0Op(item));
  // 3. 构造 ShareDB 操作
  const op = { create: undefined, del: undefined, op: json0Op, src: ..., v: version, c: collection, d: documentId };
  // 4. 发布到通道
  publisher.publish([collection, `${collection}.${documentId}`], op);
}
```

### 6.2 ShareDB 后端发布器

文件：`packages/v2/adapter-realtime-sharedb/src/ShareDbBackendPublisher.ts`

直接操作 ShareDB 后端连接：

```typescript
async publish(channels, op) {
  const connection = backend.connect();
  const doc = connection.get(collection, docId);
  if (op.create) {
    doc.fetch(err => {
      if (doc.type) return;  // 已存在，跳过
      doc.create(op.create.data, op.create.type, { source: '@@v2-projection' }, done);
    });
  } else if (op.del) {
    doc.fetch(err => {
      if (!doc.type) { /* 先创建空文档再删除 */ }
      doc.del({ source: '@@v2-projection' }, done);
    });
  } else if (op.op) {
    doc.fetch(err => doc.submitOp(op.op, { source: '@@v2-projection' }, done));
  }
}
```

### 6.3 BroadcastChannel 引擎 (浏览器端)

文件：`packages/v2/adapter-realtime-broadcastchannel/src/BroadcastChannelRealtimeHub.ts`

用于**同一浏览器多标签页间**的实时同步：

```
标签页A 修改 → BroadcastChannel.postMessage → 标签页B/C message 事件 → 更新本地快照
```

**核心特性：**

- 维护内存中的文档快照（`Map<docKey, DocState>`）
- 支持按文档订阅（`subscribeDoc`）和按集合订阅（`subscribeCollection`）
- 变更直接在本地快照上应用（`applyRealtimeChange`），无需服务器
- 使用 `structuredClone` 深拷贝快照，避免引用污染

```typescript
const applyRealtimeChange = (snapshot, change) => {
  switch (change.type) {
    case 'set':
      return updateAtPath(snapshot, change.path, () => change.value);
    case 'insert':
      return updateAtPath(snapshot, change.path, target => {
        const list = Array.isArray(target) ? [...target] : [];
        list.splice(change.index, 0, change.value);
        return list;
      });
    case 'delete':
      return updateAtPath(snapshot, change.path, target => {
        const list = Array.isArray(target) ? [...target] : [];
        list.splice(change.index, change.count);
        return list;
      });
  }
};
```

### 6.4 Noop 引擎 (测试/降级)

文件：`packages/v2/core/src/ports/defaults/NoopRealtimeEngine.ts`

空实现，所有方法直接返回 `ok(undefined)`。用于：
- 单元测试（不依赖真实实时通道）
- 浏览器端未注册实时引擎时的降级

---

## 7. V1 后端集成（NestJS + ShareDB）

文件：`apps/nestjs-backend/src/share-db/share-db.service.ts`

V1 后端基于 NestJS 构建，使用 ShareDB + Redis Pub/Sub 实现多实例实时同步：

### 7.1 ShareDB 服务初始化

```typescript
class ShareDbService extends ShareDBClass {
  constructor(shareDbAdapter, ...) {
    super({ presence: true, db: shareDbAdapter, maxSubmitRetries: 3 });
    // Redis Pub/Sub 实现多实例间同步
    if (provider === 'redis') {
      this.pubsub = new RedisPubSub({ redisURI });
    }
    // 认证中间件
    authMiddleware(this, sessionHandleService);
    // Submit 中间件（来源过滤）
    this.use('submit', this.onSubmit);
  }
}
```

### 7.2 事务后发布原始操作

```typescript
this.prismaService.bindAfterTransaction(async () => {
  const rawOpMaps = this.cls.get('tx.rawOpMaps');
  if (ops.length) {
    await this.updateTableMetaByRawOpMap(rawOpMaps);
    await this.publishOpsMap(rawOpMaps);
    this.eventEmitterService.ops2Event(ops);  // 转换为领域事件
  }
});
```

### 7.3 原始操作到领域事件的转换

文件：`apps/nestjs-backend/src/event-emitter/event-emitter.service.ts`

```typescript
ops2Event(rawOpMaps) {
  for (const rawOpMap of rawOpMaps) {
    for (const collection in rawOpMap) {
      for (const docId in data) {
        const rawOp = data[docId];
        const eventName = this.eventNameMapping[rawOp.type][docType];
        // 如 Events.TABLE_RECORD_UPDATE
        this.emit(eventName, transformedEvent);
      }
    }
  }
}
```

V1 的原始操作经过 `EventEmitterService` 转换为领域事件后，被 V2 的 Projection 消费，形成闭环：

```
客户端操作 → ShareDB → 原始操作 → EventEmitterService → 领域事件 → V2 Projection → RealtimeEngine
```

---

## 8. 数据流全景

### 8.1 完整数据流

```
用户A修改单元格
    │
    ▼
CommandBus (CreateRecordCommand / UpdateRecordCommand)
    │  领域模型生成事件
    ▼
领域事件 (RecordUpdated / RecordCreated)
    │
    ▼
AsyncMemoryEventBus.publish()
    │  事件入队，FIFO 调度
    ▼
Projection 处理 (RecordUpdatedRealtimeProjection)
    │  转换为 RealtimeChange
    ▼
IRealtimeEngine.applyChange(change, { version })
    │
    ├──► ShareDbRealtimeEngine → toJson0Op() → ShareDB → WebSocket → 用户B/C
    │
    └──► BroadcastChannelRealtimeEngine → applyRealtimeChange() → postMessage → 同浏览器其他标签页
```

### 8.2 批量操作数据流

```
粘贴/导入100条记录
    │
    ▼
CommandBus (PasteRecordsCommand)
    │
    ▼
领域事件 (RecordsBatchCreated)
    │  100 < 1000 → 不跳过实时
    ▼
RecordsBatchCreatedRealtimeProjection
    │  tasks = [ensure(record1), ensure(record2), ...]
    │  runRealtimeTasks(tasks) → Promise.all
    ▼
批量发布到实时通道
    │
    ▼
客户端接收100条文档创建通知
```

### 8.3 大批量跳过实时

```
导入10000条记录
    │
    ▼
领域事件 (RecordsBatchCreated, totalRecordCount=10000)
    │
    ▼
shouldSkipRealtimeBatchMutation(10000) → true
    │
    ▼
Projection 跳过实时推送
    │
    ▼
客户端通过轮询/刷新获取数据
```

---

## 9. 依赖注入与组装

### 9.1 ShareDB 适配器注册

文件：`packages/v2/adapter-realtime-sharedb/src/di/register.ts`

```typescript
export const registerV2ShareDbRealtime = (container, config) => {
  // 注册发布器
  container.registerInstance(v2ShareDbTokens.publisher, config.publisher);
  // 注册实时引擎
  container.register(v2CoreTokens.realtimeEngine, ShareDbRealtimeEngine, Singleton);
  // 注册所有 Projection
  container.register(TableCreatedRealtimeProjection, ..., Singleton);
  container.register(FieldCreatedRealtimeProjection, ..., Singleton);
  // ... 更多 Projection
};
```

### 9.2 浏览器端容器组装

文件：`packages/v2/container-browser/src/index.ts`

```typescript
export const registerV2BrowserPgliteDependencies = async (container, options) => {
  // 事件总线
  container.registerInstance(v2CoreTokens.eventBus, new AsyncMemoryEventBus(container, { ... }));
  // 默认 Noop 实时引擎（除非被覆盖）
  if (!container.isRegistered(v2CoreTokens.realtimeEngine)) {
    container.register(v2CoreTokens.realtimeEngine, NoopRealtimeEngine, Singleton);
  }
  // 核心服务
  registerV2CoreServices(container, { lifecycle: Singleton });
};
```

### 9.3 服务端容器组装

服务端通过 `registerV2ShareDbRealtime` 注册 ShareDB 引擎后，覆盖默认的 Noop 引擎。

---

## 10. 关键设计决策总结

| 决策 | 理由 | 代码位置 |
|------|------|---------|
| 使用 Ports & Adapters 模式 | 实时传输通道可替换：ShareDB ↔ BroadcastChannel ↔ Noop | `IRealtimeEngine` |
| 增量更新（applyChange）而非全量快照（ensure） | 避免覆盖客户端其他字段数据；减少传输量 | `RecordUpdatedRealtimeProjection` |
| 事件 FIFO 有序处理 | 保证因果一致性：字段创建先于记录创建 | `AsyncMemoryEventBus` |
| 投影并发分组 | 最大化吞吐量，同时保证非投影处理器的顺序 | `AsyncMemoryEventBus.dispatch()` |
| 大批量跳过实时 | 避免 Redis 压力雪崩和内存膨胀 | `shouldSkipRealtimeBatchMutation` |
| 操作来源过滤 | 防止 V2 Projection → ShareDB → 领域事件 的无限循环 | `ShareDbService.onSubmit()` |
| RealtimeDocId 使用 collection/docId 格式 | 兼容 ShareDB 的 collection + docId 模型 | `RealtimeDocId.fromParts()` |
| BroadcastChannel 维护内存快照 | 浏览器端无需服务器即可实现标签间同步 | `BroadcastChannelRealtimeHub` |

---

## 11. 文件索引

| 文件 | 职责 |
|------|------|
| `packages/v2/core/src/ports/RealtimeEngine.ts` | 实时引擎接口定义 |
| `packages/v2/core/src/ports/RealtimeChange.ts` | 增量变更类型定义 |
| `packages/v2/core/src/ports/RealtimeDocId.ts` | 文档标识值对象 |
| `packages/v2/core/src/ports/memory/AsyncMemoryEventBus.ts` | 异步事件总线 |
| `packages/v2/core/src/ports/EventHandler.ts` | 事件处理器装饰器 |
| `packages/v2/core/src/application/projections/RealtimeProjection.ts` | 投影接口别名 |
| `packages/v2/core/src/application/projections/RecordCreatedRealtimeProjection.ts` | 记录创建投影 |
| `packages/v2/core/src/application/projections/RecordUpdatedRealtimeProjection.ts` | 记录更新投影 |
| `packages/v2/core/src/application/projections/RecordsBatchUpdatedRealtimeProjection.ts` | 批量更新投影 |
| `packages/v2/core/src/application/projections/RecordsDeletedRealtimeProjection.ts` | 批量删除投影 |
| `packages/v2/core/src/application/projections/RecordReorderedRealtimeProjection.ts` | 记录重排序投影 |
| `packages/v2/core/src/application/projections/TableCreatedRealtimeProjection.ts` | 表创建投影 |
| `packages/v2/core/src/application/projections/FieldUpdatedRealtimeProjection.ts` | 字段更新投影 |
| `packages/v2/core/src/application/projections/BatchRecordRefreshPolicy.ts` | 批量刷新策略 |
| `packages/v2/adapter-realtime-sharedb/src/ShareDbRealtimeEngine.ts` | ShareDB 引擎实现 |
| `packages/v2/adapter-realtime-sharedb/src/ShareDbBackendPublisher.ts` | ShareDB 后端发布器 |
| `packages/v2/adapter-realtime-sharedb/src/di/register.ts` | ShareDB DI 注册 |
| `packages/v2/adapter-realtime-broadcastchannel/src/BroadcastChannelRealtimeHub.ts` | BroadcastChannel 枢纽 |
| `packages/v2/adapter-realtime-broadcastchannel/src/BroadcastChannelRealtimeEngine.ts` | BroadcastChannel 引擎 |
| `packages/v2/core/src/ports/defaults/NoopRealtimeEngine.ts` | 空实现引擎 |
| `apps/nestjs-backend/src/share-db/share-db.service.ts` | V1 ShareDB 服务 |
| `apps/nestjs-backend/src/event-emitter/event-emitter.service.ts` | V1 事件发射器 |
| `packages/v2/e2e/src/realtimeShareDb.e2e.spec.ts` | 端到端测试（最佳学习资源） |
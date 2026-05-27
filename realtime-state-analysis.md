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

## 10. 前端实时订阅入口与状态更新路径

### 10.1 连接建立与管理

#### 10.1.1 WebSocket 连接初始化

文件：`packages/sdk/src/context/app/useConnection.tsx`

前端使用 **SockJS** + **ShareDB Connection** 建立实时连接，支持自动重连：

```typescript
export const useConnection = (path?: string) => {
  const [connected, setConnected] = useState(false);
  const [connection, setConnection] = useState<Connection>();
  const [socket, setSocket] = useState<ReconnectingSockJS | null>(null);

  useEffect(() => {
    const newSocket = new ReconnectingSockJS(path || getWsPath());
    setSocket(newSocket);
    return () => newSocket.destroy();
  }, [path]);

  useConnectionAutoManage(socket, undefined, {
    inactiveTimeout: 10 * 60 * 1000,  // 页面不可见10分钟后关闭连接
    reconnectDelay: 2000,              // 页面恢复后2秒重连
  });

  useEffect(() => {
    const connection = new Connection(socket as Socket);
    setConnection(connection);

    const onConnected = () => {
      setConnected(true);
      pingInterval = setInterval(() => connection.ping(), 1000 * 10);
    };

    connection.on('connected', onConnected);
    connection.on('disconnected', onDisconnected);
    connection.on('error', shareDbErrorHandler);
  }, [path, socket]);
};
```

**连接生命周期：**
1. 组件挂载时创建 `ReconnectingSockJS` 实例
2. 建立 ShareDB Connection 并监听连接状态
3. 每 10 秒发送 ping 保活
4. 页面不可见 10 分钟后自动关闭连接
5. 页面恢复可见时自动重连

#### 10.1.2 自动重连策略

文件：`packages/sdk/src/utils/reconnectingSockJS.ts`

采用**指数退避**的重连策略：

```typescript
class ReconnectingSockJS {
  private calculateDelay(): number {
    return Math.min(
      this.reconnectInterval * Math.pow(this.reconnectDecay, this.reconnectAttempts),
      this.maxReconnectInterval
    );
  }
}
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `reconnectInterval` | 1000ms | 初始重连间隔 |
| `reconnectDecay` | 1.5 | 间隔衰减系数 |
| `maxReconnectInterval` | 30000ms | 最大重连间隔 |

#### 10.1.3 页面可见性管理

文件：`packages/sdk/src/context/app/useConnectionAutoManage.ts`

```
页面可见 → 检查连接状态 → 断开则启动重连定时器（2s后重连）
页面不可见 → 启动关闭定时器（10min后关闭连接）
```

### 10.2 订阅入口 — `useInstances` Hook

文件：`packages/sdk/src/context/use-instances/useInstances.ts`

这是前端实时订阅的核心入口，管理：
- ShareDB Query 的创建与销毁
- 查询结果缓存与去重
- 操作监听与状态更新

#### 10.2.1 查询缓存与去重

```typescript
// 全局查询缓存，跨 Hook 实例复用相同的订阅查询
const subscribeQueryCache = new Map<string, CachedQuery>();

const makeQueryKey = (collection: string, queryParams: unknown, refreshToken = 0) =>
  `${collection}|${JSON.stringify(normalizeForKey(queryParams))}|refresh:${refreshToken}`;

const acquireQuery = <T>(collection, connection, queryParams, refreshToken = 0) => {
  const key = makeQueryKey(collection, queryParams, refreshToken);
  const cached = subscribeQueryCache.get(key);
  if (cached) {
    cached.refCount += 1;
    return { key, query: cached.query };
  }
  const query = connection!.createSubscribeQuery<T>(collection, queryParams);
  subscribeQueryCache.set(key, { query, refCount: 1 });
  return { key, query };
};
```

**缓存策略关键点：**
1. `queryParams` 递归排序后序列化，保证语义相同的查询生成相同的 key
2. 引用计数（`refCount`）管理共享查询的生命周期
3. 最后一个使用者释放时销毁查询和相关文档

#### 10.2.2 订阅流程

```
useInstances 挂载
    │
    ├─► 计算 queryKey
    │
    ├─► acquireQuery() — 从缓存获取或创建新查询
    │
    ├─► 监听 Query 事件
    │   ├─► ready   — 初始数据加载完成
    │   ├─► insert  — 文档插入
    │   ├─► remove  — 文档删除
    │   ├─► move    — 文档移动
    │   └─► extra   — 额外元数据
    │
    └─► 监听单个文档 'op batch' 事件
        └─► 文档更新时触发 re-render
```

### 10.3 状态更新路径

#### 10.3.1 Query 事件 → Reducer → React 状态

文件：`packages/sdk/src/context/use-instances/reducer.ts`

```typescript
type IInstanceAction<T> =
  | { type: 'update'; doc: Doc<T> }
  | { type: 'ready'; results: Doc<T>[]; extra: unknown }
  | { type: 'insert'; docs: Doc<T>[]; index: number }
  | { type: 'remove'; docs: Doc<T>[]; index: number }
  | { type: 'removeByIds'; ids: string[] }
  | { type: 'move'; docs: Doc<T>[]; from: number; to: number }
  | { type: 'clear' }
  | { type: 'extra'; extra: unknown };
```

**状态更新流程：**

```
ShareDB Query 事件 (insert/remove/move)
        │
        ▼
useInstances 事件处理器
        │
        ▼
dispatch({ type, ...payload })
        │
        ▼
instanceReducer — 不可变更新 instances 数组
        │
        ▼
useReducer 返回新状态
        │
        ▼
React 触发 re-render
        │
        ▼
factory(doc.data, doc) — 创建模型实例
        │
        ▼
组件使用最新数据
```

#### 10.3.2 单文档 'op batch' 事件监听

文件：`packages/sdk/src/context/use-instances/opListener.ts`

```typescript
class OpListenersManager<T> {
  private opListeners: Map<string, () => void> = new Map();

  add(doc: Doc<T>, handler: (op: unknown[]) => void) {
    if (this.opListeners.has(doc.id)) return;
    doc.on('op batch', handler);
    this.opListeners.set(doc.id, () => {
      doc.removeListener('op batch', handler);
      doc.listenerCount('op batch') === 0 && doc.destroy();
    });
  }
}
```

**关键点：**
- 每个文档只注册一个监听器，避免重复监听
- 文档销毁前检查监听计数，确保没有其他使用者
- 监听器清理时自动销毁无监听的文档

#### 10.3.3 本地乐观更新

文件：`packages/sdk/src/model/record/record.ts:98-111`

用户编辑时先**本地提交**，再发送 API 请求：

```typescript
private onCommitLocal(fieldId: string, cellValue: unknown, undo?: boolean) {
  const oldCellValue = this.fields[fieldId];
  const operation = RecordOpBuilder.editor.setRecord.build({
    fieldId,
    newCellValue: cellValue,
    oldCellValue,
  });

  // 1. 更新本地 ShareDB 文档数据
  this.doc.data.fields[fieldId] = cellValue;

  // 2. 触发 'op batch' 事件，通知订阅者更新
  this.doc.emit('op batch', [operation], false);

  // 3. 调整本地版本号（模拟操作已提交）
  if (this.doc.version) {
    undo ? this.doc.version-- : this.doc.version++;
  }

  // 4. 更新模型实例字段
  this.fields[fieldId] = cellValue;
}
```

**乐观更新流程：**

```
用户编辑单元格
        │
        ▼
onCommitLocal() — 立即更新本地状态（UI 瞬时响应）
        │
        ├─► doc.data.fields[fieldId] = newValue
        ├─► emit('op batch', [op], false)  — 触发 UI 更新
        └─► doc.version++  — 保持版本号一致
        │
        ▼
API 请求发送到后端
        │
        ├─► 成功：后端返回最新数据（含计算字段），updateComputedField 同步
        │
        └─► 失败：onCommitLocal(oldValue, true) — 回滚本地状态，显示错误
```

#### 10.3.4 计算字段同步

文件：`packages/sdk/src/model/record/record.ts:113-129`

```typescript
private updateComputedField = async (fieldIds: string[], record: IRecord) => {
  const changeCellFieldIds = fieldIds.filter((fieldId) => {
    // 跳过 undefined 值 — 计算字段尚未更新（V2 异步）
    if (record.fields[fieldId] === undefined) return false;
    return !isEqual(this.fields[fieldId], record.fields[fieldId]);
  });

  if (!changeCellFieldIds.length) return;

  changeCellFieldIds.forEach((fieldId) => {
    this.doc.data.fields[fieldId] = record.fields[fieldId];
  });

  this.doc.emit('op batch', [], false);  // 触发更新，不传具体 op
};
```

### 10.4 单记录订阅 — `useRecord` Hook

文件：`packages/sdk/src/hooks/use-record.ts`

用于单个记录的细粒度订阅：

```typescript
export const useRecord = (recordId: string | undefined, initData?: IRecord) => {
  useEffect(() => {
    if (!connection || !recordId) return;

    const doc = connection.get(`${IdPrefix.Record}_${tableId}`, recordId);

    // 1. 拉取当前快照
    doc.fetch((err) => {
      if (!err) setInstance(createRecordInstance(doc.data, doc));
    });

    // 2. 订阅后续更新
    doc.subscribe(() => {
      doc.on('op batch', () => {
        setInstance(createRecordInstance(doc.data, doc));
      });
    });

    return () => {
      doc.removeListener('op batch', listeners);
      doc.listenerCount('op batch') === 0 && doc.unsubscribe();
      doc.listenerCount('op batch') === 0 && doc.destroy();
    };
  }, [connection, recordId, tableId]);
};
```

---

## 11. 乱序事件与重连回放处理

### 11.1 ShareDB 内置的乱序处理机制

ShareDB 客户端自动处理乱序和重复操作：

#### 11.1.1 版本号驱动的操作排序

每个 ShareDB 文档维护单调递增的 `version`：

```
本地文档版本: v=5
收到远程操作1: v=6  → 立即应用
收到远程操作2: v=5  → 已应用，跳过（去重）
收到远程操作3: v=8  → 版本跳变，等待中间操作或触发 fetch
```

#### 11.1.2 待确认操作队列

本地提交的操作进入**待确认队列**，收到服务器确认后移除：

```
用户提交本地操作 op1 (v=5)
    │
    ▼
pendingOps = [op1]
    │
    ▼
发送到服务器
    │
    ├─► 成功：服务器返回 ack → 移除 pendingOps[0]
    │
    └─► 失败或超时：重新提交或回滚
```

### 11.2 重连回放机制

#### 11.2.1 ShareDB 自动重连与状态恢复

当 WebSocket 断开重连后，ShareDB 自动：

1. **重新建立连接**：`Connection` 自动重新握手
2. **重新订阅查询**：Query 自动重新执行 `subscribe`
3. **快照拉取与差异合并**：
   - 发送当前版本号 `v` 到服务器
   - 服务器返回快照或增量 ops
   - 客户端应用差异，恢复到最新状态

#### 11.2.2 页面恢复时的连接管理

文件：`packages/sdk/src/context/app/useConnectionAutoManage.ts`

```typescript
useEffect(() => {
  if (visible && !isConnected(connection)) {
    reconnectTimerRef.current = setTimeout(() => {
      reconnect ? reconnect() : currentConnection.reconnect();
    }, reconnectDelay);
  }
  if (!visible && isConnected(connection)) {
    inactiveTimerRef.current = setTimeout(() => {
      currentConnection.close();
    }, inactiveTimeout);
  }
}, [visible, ...]);
```

**状态转换：**

```
页面可见
    │
    ▼
检查连接状态
    │
    ├─► 已连接 → 保持
    │
    └─► 未连接 → 2秒后启动重连
        │
        ▼
ReconnectingSockJS 指数退避重连
        │
        ▼
连接成功
        │
        ▼
ShareDB Query 自动重新订阅
        │
        ▼
查询结果 ready → dispatch ready 事件
        │
        ▼
UI 恢复到最新状态
```

### 11.3 大批量操作的刷新策略

文件：`packages/sdk/src/context/use-instances/useInstances.ts:79-87`

**schemaRefreshToken** 机制处理跳过实时推送的大批量操作：

```typescript
const [schemaRefreshToken, setSchemaRefreshToken] = useState(0);

// 监听 Presence 通道中的 schema refresh 通知
const receiveListener = (_id: string, batch: unknown) => {
  // 1. 处理计算字段刷新
  const fieldIds = getSchemaRefreshRecordFieldIds(tableId, batch);
  if (fieldIds?.length) {
    refreshProjectedRecordFields(fieldIds).then((handled) => {
      if (!handled) setSchemaRefreshToken(t => t + 1);  // 增量失败，全量刷新
    });
    return;
  }

  // 2. 处理批量删除
  const deletedRecordIds = getProjectedDeleteRecordIds(tableId, batch);
  if (deletedRecordIds?.length) {
    removeProjectedRecordsByIds(deletedRecordIds);
    return;
  }

  // 3. 其他情况全量刷新
  setSchemaRefreshToken(t => t + 1);
};
```

#### 11.3.1 计算字段增量刷新

文件：`packages/sdk/src/context/use-instances/useInstances.ts:428-503`

```typescript
const refreshProjectedRecordFields = async (fieldIds: string[]) => {
  // 1. API 拉取指定字段的最新值
  const { data } = await getRecords(tableId, {
    ...queryParams,
    projection: fieldIds,  // 只拉取需要刷新的字段
  });

  // 2. 验证记录集合未变化（顺序和 ID 一致）
  if (currentDocIds.length !== fetchedRecordIds.length) return false;
  if (currentDocIds.some((id, i) => id !== fetchedRecordIds[i])) return false;

  // 3. 增量更新文档数据
  currentDocs.forEach((doc) => {
    fieldIds.forEach((fieldId) => {
      if (!isEqual(doc.data.fields?.[fieldId], nextValue)) {
        doc.data.fields![fieldId] = nextValue;
        changed = true;
      }
    });
    if (changed) notifyProjectedRecordDocUpdate(doc, dispatch);
  });

  return true;
};
```

#### 11.3.2 刷新失败降级

增量刷新失败时（如记录集合已变化），通过递增 `schemaRefreshToken` 触发查询重建：

```
schemaRefreshToken 变化
    │
    ▼
useEffect 依赖检测到变化
    │
    ▼
releaseQuery(previousKey) — 释放旧查询
    │
    ▼
acquireQuery() — 创建新查询，参数相同但 refreshToken 不同
    │
    ▼
新查询从服务器拉取完整快照
    │
    ▼
dispatch ready 事件，UI 全量更新
```

### 11.4 Presence 通道的批量操作通知

文件：`packages/sdk/src/context/use-instances/useInstances.ts:532-595`

通过 ShareDB Presence 通道传递批量操作通知：

```typescript
const presence: Presence = connection.getPresence(
  getActionTriggerChannel(schemaRefreshCollectionTableId)
);

presence.subscribe();
presence.addListener('receive', receiveListener);
```

**通知类型：**
1. `setField` — 字段更新，触发计算字段刷新
2. `setRecord` — 记录批量更新，触发刷新
3. `addRecord` — 记录批量创建，触发刷新
4. `deleteRecord` — 记录批量删除，增量移除

---

## 12. 前端冲突解决机制

### 12.1 乐观更新与服务器最终一致

#### 12.1.1 本地优先策略

用户编辑采用**本地优先**的乐观更新：
1. UI 立即响应（无需等待服务器）
2. 后台异步发送 API 请求
3. 成功时用服务器返回的权威数据修正本地
4. 失败时回滚本地状态并提示用户

#### 12.1.2 冲突场景：同时编辑同一单元格

```
用户A编辑字段 fld1 → 本地更新值为 "A" → API 发送中...
用户B编辑字段 fld1 → 本地更新值为 "B" → API 发送成功 → ShareDB 广播 op(v=6)

用户A收到远程 op(v=6)
    │
    ├─► 本地 pendingOp 基于 v=5
    │
    ├─► ShareDB OT 转换：将 pendingOp 转换为基于 v=6
    │
    └─► 最终值取决于：
        ├─► 后提交者覆盖先提交者（默认 json0 行为）
        └─► 或 API 层进行值比较（业务逻辑决定）
```

### 12.2 操作构建器与路径匹配

文件：`packages/core/src/op-builder/record/set-record.ts`

`IOtOperation` 是 ShareDB json0 类型的标准格式：

```typescript
interface IOtOperation {
  p: (string | number)[];  // JSON 路径
  oi?: unknown;             // object insert
  od?: unknown;             // object delete
  li?: unknown;             // list insert
  ld?: unknown;             // list delete
  na?: number;              // number add
  si?: string;              // string insert
  sd?: string;              // string delete
}
```

#### 12.2.1 SetRecord 操作构建

```typescript
class SetRecordBuilder {
  build({ fieldId, newCellValue, oldCellValue }): IOtOperation {
    // null/空数组 → 删除键
    if (newCellValue == null || (Array.isArray(newCellValue) && newCellValue.length === 0)) {
      return { p: ['fields', fieldId], od: oldCellValue, oi: null };
    }

    // 旧值为空 → 插入键
    if (oldCellValue == null) {
      return { p: ['fields', fieldId], oi: newCellValue };
    }

    // 替换键
    return { p: ['fields', fieldId], od: oldCellValue, oi: newCellValue };
  }
}
```

#### 12.2.2 操作检测与路径匹配

文件：`packages/core/src/op-builder/common.ts`

```typescript
function pathMatcher<T>(path: (string | number)[], matchList: string[]): T | null {
  if (path.length !== matchList.length) return null;

  const res: Record<string, string | number> = {};
  for (let i = 0; i < matchList.length; i++) {
    if (matchList[i].startsWith(':')) {          // :fieldId → 捕获参数
      res[matchList[i].slice(1)] = path[i];
      continue;
    }
    if (matchList[i] === '*') continue;           // * → 跳过匹配
    if (path[i] !== matchList[i]) return null;    // 精确匹配
  }
  return res as T;
}

// 示例：匹配 ['fields', 'fld123']
pathMatcher(op.p, ['fields', ':fieldId']);
// → { fieldId: 'fld123' }
```

### 12.3 OT 转换原理（ShareDB 内置）

json0 类型的操作转换遵循以下规则：

#### 12.3.1 并发插入同一对象的不同键

```
op1: { p: ['fields', 'a'], oi: 1 }
op2: { p: ['fields', 'b'], oi: 2 }

转换结果：两个操作都保留，最终 { a: 1, b: 2 }
```

#### 12.3.2 并发插入同一对象的相同键

```
op1: { p: ['fields', 'a'], oi: 1 }  // 先到达服务器
op2: { p: ['fields', 'a'], oi: 2 }  // 后到达

转换结果：后提交者获胜，最终值为 2
```

#### 12.3.3 删除与修改冲突

```
op1: { p: ['fields', 'a'], od: 1 }        // 删除 a
op2: { p: ['fields', 'a'], od: 1, oi: 2 } // 修改 a 为 2

转换结果：删除优先，a 不存在
```

### 12.4 字段类型特殊处理

#### 12.4.1 附件字段 — 服务器补充签名 URL

文件：`packages/sdk/src/model/record/record.ts:154-158`

```typescript
// 附件字段需要同步（服务器补充 presignedUrl）
if (this.fieldMap[fieldId]?.type === FieldType.Attachment) {
  fieldsToSync.add(fieldId);
}
```

#### 12.4.2 计算字段 — 异步更新

- **公式、查找、汇总字段**：服务器异步计算
- **本地不覆盖**：`updateComputedField` 中跳过 `undefined` 值
- **通过实时通道更新**：计算完成后通过 ShareDB op 推送

### 12.5 冲突解决矩阵

| 冲突场景 | 解决策略 | 代码位置 |
|---------|---------|---------|
| 两人编辑同一记录不同字段 | ShareDB OT 自动合并，互不影响 | `SetRecordBuilder.build()` |
| 两人编辑同一记录同一字段 | 后提交者覆盖先提交者（服务器时序） | ShareDB json0 内置 |
| 本地编辑与远程计算字段更新 | 本地手动编辑优先，计算字段通过 op 更新 | `updateComputedField` |
| 字段类型变更后记录编辑 | 实时推送字段更新通知，前端刷新 schema | `schemaRefreshToken` |
| 记录删除后编辑 | ShareDB 操作失败，前端重新拉取 | ShareDB 内置 |
| 重连后状态不一致 | Query 重新订阅，拉取最新快照 | ShareDB Query 内置 |

---

## 13. 关键设计决策总结

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
| **查询缓存与引用计数** | 避免重复订阅，减少服务器压力 | `useInstances.acquireQuery()` |
| **本地乐观更新** | UI 瞬时响应，提升用户体验 | `Record.onCommitLocal()` |
| **指数退避重连** | 网络波动时自动恢复，避免服务器压力 | `ReconnectingSockJS` |
| **页面可见性管理** | 后台页面自动释放连接，节省资源 | `useConnectionAutoManage` |
| **增量刷新优先，失败降级全量** | 平衡实时性与性能 | `refreshProjectedRecordFields()` |
| **Presence 通道传递批量通知** | 大批量操作跳过实时推送但通知前端刷新 | `useInstances.receiveListener` |
| **版本号驱动的 OT 转换** | 乱序操作自动排序和去重 | ShareDB 内置 |

---

## 14. 文件索引

### 14.1 V2 核心架构

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

### 14.2 实时引擎适配器

| 文件 | 职责 |
|------|------|
| `packages/v2/adapter-realtime-sharedb/src/ShareDbRealtimeEngine.ts` | ShareDB 引擎实现 |
| `packages/v2/adapter-realtime-sharedb/src/ShareDbBackendPublisher.ts` | ShareDB 后端发布器 |
| `packages/v2/adapter-realtime-sharedb/src/di/register.ts` | ShareDB DI 注册 |
| `packages/v2/adapter-realtime-broadcastchannel/src/BroadcastChannelRealtimeHub.ts` | BroadcastChannel 枢纽 |
| `packages/v2/adapter-realtime-broadcastchannel/src/BroadcastChannelRealtimeEngine.ts` | BroadcastChannel 引擎 |
| `packages/v2/core/src/ports/defaults/NoopRealtimeEngine.ts` | 空实现引擎 |

### 14.3 V1 后端集成

| 文件 | 职责 |
|------|------|
| `apps/nestjs-backend/src/share-db/share-db.service.ts` | V1 ShareDB 服务 |
| `apps/nestjs-backend/src/event-emitter/event-emitter.service.ts` | V1 事件发射器 |

### 14.4 前端 SDK

| 文件 | 职责 |
|------|------|
| `packages/sdk/src/context/app/useConnection.tsx` | ShareDB 连接管理 Hook |
| `packages/sdk/src/context/app/useConnectionAutoManage.ts` | 页面可见性驱动的连接管理 |
| `packages/sdk/src/utils/reconnectingSockJS.ts` | 自动重连的 SockJS 封装 |
| `packages/sdk/src/context/use-instances/useInstances.ts` | **核心**：批量订阅与状态更新 Hook |
| `packages/sdk/src/context/use-instances/reducer.ts` | 实例状态 Reducer |
| `packages/sdk/src/context/use-instances/opListener.ts` | 单文档操作监听器管理 |
| `packages/sdk/src/hooks/use-record.ts` | 单记录订阅 Hook |
| `packages/sdk/src/model/record/record.ts` | 记录模型（含乐观更新） |
| `packages/sdk/src/model/record/factory.ts` | 记录实例工厂 |
| `packages/sdk/src/context/app/ConnectionContext.tsx` | 连接 Context |
| `packages/sdk/src/hooks/use-connection.ts` | 连接 Hook |

### 14.5 OT 操作构建器

| 文件 | 职责 |
|------|------|
| `packages/core/src/models/op.ts` | OT 操作类型定义 |
| `packages/core/src/op-builder/op-builder.abstract.ts` | 操作构建器抽象基类 |
| `packages/core/src/op-builder/record/record-op-builder.ts` | 记录操作构建器 |
| `packages/core/src/op-builder/record/set-record.ts` | SetRecord 操作构建 |
| `packages/core/src/op-builder/common.ts` | 路径匹配工具函数 |

### 14.6 测试资源

| 文件 | 职责 |
|------|------|
| `packages/v2/e2e/src/realtimeShareDb.e2e.spec.ts` | 端到端测试（最佳学习资源） |
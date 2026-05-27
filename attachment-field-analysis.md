# 附件上传与字段绑定全链路分析

本文档基于对 Teable 代码库的静态分析，梳理附件（Attachment）从上传到与表格行/列绑定的完整链路，涵盖上传通道、字段关联、权限校验三大核心机制。

---

## 一、整体架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                        前端 (nextjs-app)                        │
│  uploadAttachment() ──► POST /api/table/{tableId}/record/{...}  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     NestJS 后端 Controller                      │
│  RecordOpenApiController.uploadAttachment()                     │
│    ├─ 校验字段类型 & 记录存在                                    │
│    ├─ AttachmentsService.uploadFile() / uploadFromUrl()          │
│    │    ├─ signature() ──► 生成预签名 URL + token               │
│    │    └─ uploadStreamToStorage() ──► 存入对象存储              │
│    ├─ notify() ──► 写入 attachments 元数据表                    │
│    └─ RecordOpenApiService.updateRecord() ──► 写入行绑定          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
             ┌──────────────┴──────────────┐
             ▼                              ▼
┌──────────────────────────┐  ┌──────────────────────────────────┐
│  对象存储 (S3/MinIO/Local)│  │  PostgreSQL: 元数据表             │
│  - bucket/path/file       │  │  - attachments (文件元数据)       │
│                           │  │  - attachments_table (行-列绑定) │
└──────────────────────────┘  └──────────────────────────────────┘
```

---

## 二、上传通道

### 2.1 上传入口

系统存在 **多条上传通道**，由 `UploadType` 枚举区分（`packages/openapi/src/attachment/signature.ts:6-21`）：

| UploadType | 用途 | 存储 bucket |
|---|---|---|
| `Table` (1) | 表格附件字段上传 | 表格专用 |
| `Avatar` (2) | 用户头像 | 用户专用 |
| `Form` (3) | 表单附件 | 表单专用 |
| `OAuth` (4) | OAuth 相关文件 | OAuth 专用 |
| `Import` (5) | 数据导入文件 | 导入专用 |
| `Plugin` (6) | 插件资源 | 插件专用 |
| `Comment` (7) | 评论附件 | 评论专用 |
| `Logo` (8) | 空间/Base Logo | Logo 专用 |
| `ExportBase` (9) | 导出文件 | 导出专用 |
| `Template` (10) | 模板文件 | 模板专用 |
| 其他 | 内部用途 | 对应专用 |

### 2.2 核心上传控制器

`apps/nestjs-backend/src/features/attachments/attachments.controller.ts` 提供四个端点：

| 端点 | 方法 | 鉴权 | 说明 |
|---|---|---|---|
| `/api/attachments/upload/:token` | `PUT`/`POST` | token 校验（缓存） | 客户端直传文件到对象存储 |
| `/api/attachments/read/:path(*)` | `GET` | token 或路径校验 | 读取附件（带条件缓存） |
| `/api/attachments/signature` | `POST` | AuthGuard + DynamicAuthGuard | 申请预签名 URL |
| `/api/attachments/notify/:token` | `POST` | AuthGuard + DynamicAuthGuard | 通知上传完成并写入元数据 |

### 2.3 上传流程（三步握手）

```
客户端                                  服务端
  │                                       │
  │── POST /attachments/signature ───────►│
  │   { type: Table, contentLength,       │
  │     contentType, hash }               │
  │                                       │── storageAdapter.presigned()
  │                                       │── cache.set(signature:token, path+bucket)
  │◄──── { url, token, requestHeaders } ──│
  │                                       │
  │── PUT {url} (直传到对象存储) ────────►│
  │   (携带 token + requestHeaders)       │
  │                                       │
  │── POST /attachments/notify/:token ───►│
  │   { filename }                        │
  │                                       │── cache.get(signature:token)
  │                                       │── storageAdapter.getObjectMeta()
  │                                       │── prisma.attachments.create()
  │                                       │── attachmentsCropQueueProcessor.add()
  │◄──── { token, size, mimetype, url,    │
  │       presignedUrl }                  │
```

### 2.4 存储适配器

`apps/nestjs-backend/src/features/attachments/plugins/storage.ts` 注入 `StorageAdapter`，提供多种后端实现：

- **Local** (`local.ts`)：本地文件系统
- **S3** (`s3.ts`)：AWS S3
- **MinIO** (`minio.ts`)：MinIO 兼容服务
- **Aliyun** (`aliyun.ts`)：阿里云 OSS

适配器统一接口 `presigned()` / `getObjectMeta()` / `getPreviewUrl()` / `uploadFile()` / `cropImage()`，上传时调用 `presigned()` 获取预签名 URL，客户端直传对象存储。

### 2.5 签名与 Token 机制

`attachments.service.ts:131-151` — `signature()` 方法：

1. 生成存储路径和 token
2. 将 `{ path, bucket, hash }` 缓存到 `attachment:signature:{token}`
3. 返回预签名 URL 给客户端

Token 是一次性的临时凭证，有效期由 `storageConfig.tokenExpireIn` 控制。

### 2.6 上传完成通知

`attachments.service.ts:173-237` — `notify()` 方法：

1. 校验 token 缓存（防止伪造）
2. 调用 `storageAdapter.getObjectMeta()` 获取文件真实元数据（hash, size, mimetype, width, height）
3. 写入 `attachments` 数据库表
4. 提交图片裁剪任务到队列（`attachmentsCropQueueProcessor`）
5. 生成预览 URL 并返回

### 2.7 直接上传（服务端中转）

除了客户端直传外，还有两种服务端中转方式：

- `uploadFile()` (`attachments.service.ts:248-287`)：接收 Multer 文件流，签名后上传到存储
- `uploadFromUrl()` (`attachments.service.ts:289-327`)：从 URL 下载文件再上传到存储，带 SSRF 防护（`getSsrfSafeAgents()`）

---

## 三、字段关联与行绑定

### 3.1 数据模型设计（真实 Schema）

基于 `packages/db-main-prisma/prisma/postgres/schema.prisma:428-463` 的真实定义：

#### 3.1.1 Attachments 表（文件元数据）

```prisma
model Attachments {
  id             String    @id @default(cuid())    // 主键，cuid 格式
  token          String    @unique                  // 上传 token，唯一索引
  hash           String                              // 文件哈希
  size           BigInt                              // 文件大小
  mimetype       String                              // MIME 类型
  path           String                              // 对象存储路径
  width          Int?                                 // 图片宽度
  height         Int?                                 // 图片高度
  deletedTime    DateTime? @map("deleted_time")     // 软删除时间
  createdTime    DateTime  @default(now()) @map("created_time")
  createdBy      String    @map("created_by")       // 上传者 user ID
  lastModifiedBy String?   @map("last_modified_by")
  thumbnailPath  String?   @map("thumbnail_path")   // 缩略图路径 (JSON: {sm, lg})

  @@map("attachments")
}
```

**主键与约束**：
- 主键：`id`（cuid 格式，非 token）
- 唯一约束：`token` 上有唯一索引
- 软删除：通过 `deletedTime` 字段实现，无外键级联删除

#### 3.1.2 AttachmentsTable 表（行-列绑定）

```prisma
model AttachmentsTable {
  id               String    @id @default(cuid())    // 主键，cuid 格式
  attachmentId     String    @map("attachment_id")   // 附件 ID (actxxx 格式)
  name             String                             // 文件名
  token            String                             // 附件 token (关联 attachments.token)
  tableId          String    @map("table_id")        // 表 ID
  recordId         String    @map("record_id")       // 行 (记录) ID
  fieldId          String    @map("field_id")        // 列 (字段) ID
  createdTime      DateTime  @default(now()) @map("created_time")
  createdBy        String    @map("created_by")
  lastModifiedBy   String?   @map("last_modified_by")
  lastModifiedTime DateTime? @updatedAt @map("last_modified_time")

  @@index([tableId, recordId])
  @@index([tableId, fieldId])
  @@index([attachmentId])
  @@map("attachments_table")
}
```

**主键与约束**：
- 主键：`id`（cuid 格式，前缀 `attt`）
- 索引：`(tableId, recordId)`、`(tableId, fieldId)`、`(attachmentId)`
- **无外键约束**：`token` 字段逻辑关联 `attachments.token`，但无数据库级外键
- **无软删除字段**：物理删除

**删除行为**：
- 删除记录/字段/表时，通过应用层代码调用 `deleteRecords()` / `deleteFields()` / `deleteTable()` 物理删除 `attachments_table` 中的绑定行
- `attachments` 表的元数据不会被级联删除（可能产生孤立文件）

### 3.2 附件字段类型

`packages/v2/core/src/domain/table/fields/types/AttachmentField.ts` — `AttachmentField` 继承自 `Field`，类型为 `FieldType.attachment()`，无额外配置选项。

### 3.3 附件值规格 (Spec)

`packages/v2/core/src/domain/table/records/specs/values/SetAttachmentValueSpec.ts` 定义了附件单元格的值结构：

```typescript
interface AttachmentItem {
  id: string;           // 附件 ID (actxxx 格式)
  name: string;         // 文件名
  path: string;         // 存储路径
  token: string;        // 附件 token (关联 attachments.token)
  size: number;         // 文件大小
  mimetype: string;     // MIME 类型
  presignedUrl?: string;  // 预签名访问 URL
  width?: number;
  height?: number;
  smThumbnailUrl?: string;  // 小缩略图 URL
  lgThumbnailUrl?: string;  // 大缩略图 URL
}
```

单元格值为 `AttachmentItem[]` 数组，支持同一单元格存储多个附件。

### 3.4 绑定写入的两条路径（真实模型）

#### 3.4.1 v1 路径：事件驱动的异步绑定（ShareDB 实时协作）

**触发链路**：

```
ShareDB 操作提交
    │
    ▼
event-emitter.service.ts: ops2Event()
    │  ├─ 解析 RawOp (Create/Edit/Del)
    │  ├─ 通过 eventNameMapping 映射到事件
    │  └─ emitAsync(Events.TABLE_RECORD_*)
    │
    ▼
attachment.listener.ts: 事件监听器（async: true）
    ├─ @OnEvent(Events.TABLE_RECORD_CREATE) → createRecords()
    ├─ @OnEvent(Events.TABLE_RECORD_UPDATE) → updateRecords()
    └─ @OnEvent(Events.TABLE_RECORD_DELETE) → deleteRecords()
```

**关键代码**：

`apps/nestjs-backend/src/event-emitter/event-emitter.service.ts:85-101` — `ops2Event()`：

1. 从 `rawOpMaps` 收集事件（`collectEventsFromRawOpMap()`）
2. 按 `tableId + eventName` 分组聚合
3. 通过 RxJS 流式处理并触发 `handleEventResult()` → `emitAsync()`

`apps/nestjs-backend/src/event-emitter/listeners/attachment.listener.ts:16-63` — 四个监听点：

```typescript
@OnEvent(Events.TABLE_RECORD_CREATE, { async: true })
  → attachmentsTableService.createRecords()

@OnEvent(Events.TABLE_RECORD_UPDATE, { async: true })
  → attachmentsTableService.updateRecords()

@OnEvent(Events.TABLE_RECORD_DELETE, { async: true })
  → attachmentsTableService.deleteRecords()

@OnEvent(Events.TABLE_FIELD_DELETE, { async: true })
  → attachmentsTableService.deleteFields()
```

**核心服务**：`apps/nestjs-backend/src/features/attachments/attachments-table.service.ts`

- `createRecords()`：创建记录时，扫描所有 `Attachment` 类型字段，将附件项批量写入 `attachments_table`
- `updateRecords()`：更新记录时，对比 oldValue/newValue，计算需要删除和新增的绑定（通过 `tableId-fieldId-recordId-attachmentId` 四元组做 diff）
- `deleteRecords()`：按 recordId 删除绑定
- `deleteFields()`：按 fieldId 删除绑定
- `deleteTable()`：按 tableId 删除所有绑定

**重要特性**：
- `{ async: true }`：事件异步处理，不阻塞主流程
- 事件合并：`combineEvents()` 将同 tableId 同类型的事件批量处理
- 最终一致性：绑定写入与记录写入异步，可能存在短暂延迟

#### 3.4.2 v2 路径：同步的绑定落库（领域模型 + Kysely）

**触发边界**：

v2 路径的绑定写入是**同步**的，发生在 SQL 事务内，由 `CellValueMutateVisitor` 和 `RecordInsertBuilder` 在构建 SQL 时直接生成。

**调用链**：

```
TableRecordRepository.createMany() / update()
    │
    ▼
RecordInsertBuilder / RecordUpdateBuilder
    │
    ├─ 构建主表 INSERT/UPDATE 语句
    │
    ▼  (插入时)
RecordInsertBuilder.ts:384-395
    │  遍历字段，遇到 Attachment 类型时
    │  调用 buildAttachmentTableInsertQuery()
    │  生成附加的 INSERT 语句
    │
    ▼  (更新时)
CellValueMutateVisitor.visitSetAttachmentValueSpec()
    │  CellValueMutateVisitor.ts:496-523
    │  调用 buildAttachmentTableReplaceQueries()
    │  生成 DELETE + INSERT 语句
    │
    ▼
attachmentTableMutations.ts: buildAttachmentTableReplaceQueries()
    ├─ DELETE FROM attachments_table
    │     WHERE table_id=? AND record_id=? AND field_id=?
    └─ INSERT INTO attachments_table (...) VALUES (...)
```

**关键代码**：

`packages/v2/adapter-table-repository-postgres/src/record/attachments/attachmentTableMutations.ts:74-94` — `buildAttachmentTableReplaceQueries()`：

```typescript
export const buildAttachmentTableReplaceQueries = (db, params) => {
  // 1. 先删除该单元格的所有旧绑定
  const deleteQuery = db
    .deleteFrom('attachments_table')
    .where('table_id', '=', params.tableId)
    .where('record_id', '=', params.recordId)
    .where('field_id', '=', params.fieldId)
    .compile();

  // 2. 再插入新绑定（如果有值）
  const insertQuery = buildAttachmentTableInsertQuery(db, params);

  // 3. 返回查询数组，将在同一事务中执行
  return insertQuery ? [deleteQuery, insertQuery] : [deleteQuery];
};
```

`packages/v2/adapter-table-repository-postgres/src/record/visitors/CellValueMutateVisitor.ts:496-523` — `visitSetAttachmentValueSpec()`：

```typescript
visitSetAttachmentValueSpec(spec: SetAttachmentValueSpec) {
  // ... 验证字段存在且类型正确 ...

  // 将附件绑定 SQL 添加到 additionalStatements
  this.additionalStatements.push(
    ...buildAttachmentTableReplaceQueries(this.db, {
      actorId: this.ctx.actorId,
      tableId: this.table.id().toString(),
      recordId: this.ctx.recordId,
      fieldId: spec.fieldId.toString(),
      value: spec.value.toValue(),
    })
  );

  return ok(undefined);
}
```

**v2 触发边界总结**：

| 触发场景 | 触发点 | 执行方式 |
|---|---|---|
| 创建记录 | `RecordInsertBuilder.build()` 中检测到 Attachment 字段 | 同步，事务内 |
| 更新记录 | `CellValueMutateVisitor.visitSetAttachmentValueSpec()` | 同步，事务内 |
| 删除记录 | 主表记录删除后无自动清理，需显式调用 | N/A |
| 删除字段 | 无自动清理 | N/A |
| 删除表 | 无自动清理 | N/A |

**v1 vs v2 对比**：

| 维度 | v1 (事件驱动) | v2 (同步 SQL) |
|---|---|---|
| 执行时机 | 异步，事件监听 | 同步，SQL 构建时 |
| 一致性 | 最终一致 | 强一致（事务内） |
| 触发方式 | ShareDB op → 事件 → 监听器 | 领域模型 → SQL 构建 |
| 删除处理 | 支持记录/字段/表删除的级联清理 | 仅支持创建/更新时的绑定 |
| 批量处理 | 支持事件合并批量 | 单条记录处理 |

### 3.5 附件值装饰（URL 签名）

当记录返回给客户端时，需要为每个附件项生成预签名 URL。此过程在 `AttachmentsStorageService` 和 `V2AttachmentUrlSignerService` 中完成：

`apps/nestjs-backend/src/features/v2/v2-attachment-url-signer.service.ts:38-61` — `signItems()`：

1. 查询附件元数据（`attachmentLookupService.listAttachmentsByTokens()`）
2. 并发调用 `signOne()` 为每个附件生成预签名 URL（并发限制 4）
3. 处理缩略图 URL（图片有 sm/lg 两级缩略图）
4. 缓存结果到 `attachment:preview:{token}`

`apps/nestjs-backend/src/features/v2/v2-record-changed-value-decorator.service.ts` — 桥接 v2-core 的 `AttachmentValueDecoratorService`，在记录变更时自动为附件字段添加预签名 URL。

### 3.6 ShareDB 实时修复

`apps/nestjs-backend/src/share-db/repair-attachment-op/repair-attachment-op.service.ts`：

当 ShareDB 操作到达时，附件项可能缺少 `presignedUrl` 或元数据，修复服务：

1. 检测操作中的附件 token（`getCollectionsAttachmentToken()`）
2. 批量查询 `attachments` 表获取元数据（`getAttachmentMetaTokenMap()`）
3. 批量查询缓存中的预览 URL（`getCachePreviewUrlTokenMap()`）
4. 为缺失 URL 的附件生成预签名 URL
5. 重命名时清除缓存（`cache.del(attachment:preview:{token})`）
6. 合并元数据到操作中（`mergeAttachmentMeta()`）

---

## 四、权限校验

### 4.1 上传端点权限

`attachments.controller.ts` 中：

- `/upload/:token` 和 `/read/:path`：标记为 `@Public()`，无需登录，但通过 token 或路径校验确保合法性
- `/signature` 和 `/notify/:token`：使用 `@UseGuards(AuthGuard, DynamicAuthGuardFactory)`

### 4.2 DynamicAuthGuardFactory

`apps/nestjs-backend/src/features/attachments/guard/auth.guard.ts`：

```typescript
canActivate(context: ExecutionContext) {
  const shareId = context.switchToHttp().getRequest().headers['tea-share-id'];
  if (shareId) {
    this.cls.set('shareViewId', shareId);
    return this.shareAuthGuard.validate(context, shareId);
  }
  return this.authGuard.validate(context);
}
```

- 有 `Tea-Share-Id` 头 → 走分享视图鉴权（`ShareAuthGuard`）
- 无 → 走标准登录鉴权（`AuthGuard`）

### 4.3 记录级权限（uploadAttachment）

`apps/nestjs-backend/src/features/record/open-api/record-open-api.controller.ts:162-175`：

```typescript
@Permissions('record|update')
@Post(':recordId/:fieldId/uploadAttachment')
async uploadAttachment(...) {
  return await this.recordOpenApiService.uploadAttachment(...);
}
```

- 装饰器 `@Permissions('record|update')` 要求用户有该表的 `record|update` 权限
- `getValidateAttachmentRecord()` 验证：
  - 字段存在且类型为 `Attachment`
  - 字段不是计算字段（`isComputed` 为 false）
  - 记录存在

### 4.4 Token 安全机制

- **签名 token**：`attachment:signature:{token}` 缓存存储路径和 bucket，上传时校验
- **预览 token**：`attachment:preview:{token}` 缓存预签名 URL，减少重复签名
- **上传 token**：`attachment:upload:{token}` 缓存上传后的文件元数据（mimetype, hash, size）

所有 token 均有过期时间，防止重放攻击。

### 4.5 分享视图访问

当通过分享链接访问时，`DynamicAuthGuardFactory` 会：
1. 从请求头提取 `tea-share-id`
2. 调用 `ShareAuthGuard.validate()` 校验分享视图的有效性
3. 通过后在 CLS 中设置 `shareViewId`

### 4.6 SSRF 防护

`apps/nestjs-backend/src/features/attachments/attachments.service.ts:446-451` 中 `uploadFileContent()` 使用 `getSsrfSafeAgents()` 进行 HTTP 请求，防止从 URL 上传时的 SSRF 攻击。

### 4.7 文件大小限制

`signature()` 方法检查 `contentLength > MAX_FILE_SIZE`（`thresholdConfig.maxAttachmentUploadSize`），`uploadFile()` 使用 `thresholdConfig.maxOpenapiAttachmentUploadSize`，双重限制防止超大文件。

---

## 五、端到端调用链（典型场景）

### 场景：在表格的附件列中上传一个文件

```
1. 前端调用 getSignature({ type: Table, contentLength, contentType })
   └─ POST /api/attachments/signature
      ├─ AuthGuard + DynamicAuthGuardFactory 鉴权
      └─ AttachmentsService.signature()
         ├─ storageAdapter.presigned(bucket, dir, params)
         ├─ cache.set(attachment:signature:token, {path, bucket, hash})
         └─ return { url, token, requestHeaders }

2. 前端直传文件到对象存储
   └─ PUT {url} (携带 token + requestHeaders)

3. 前端调用 notify(token, filename)
   └─ POST /api/attachments/notify/:token
      ├─ AuthGuard + DynamicAuthGuardFactory 鉴权
      └─ AttachmentsService.notify()
         ├─ cache.get(attachment:signature:token)
         ├─ storageAdapter.getObjectMeta(bucket, path, token)
         ├─ prisma.attachments.create()  ← 写入元数据表
         ├─ attachmentsCropQueueProcessor.add()  ← 图片裁剪
         └─ return { token, size, mimetype, url, presignedUrl }

4. 前端构造 AttachmentItem，调用更新记录 API
   └─ 更新记录字段值
      ├─ v1 路径:
      │   └─ ShareDB op
      │       └─ ops2Event() → Events.TABLE_RECORD_UPDATE
      │           └─ AttachmentListener.recordUpdateListener()
      │               └─ attachmentsTableService.updateRecords()  ← 异步写入绑定
      └─ v2 路径:
          └─ TableRecordRepository.update()
              └─ CellValueMutateVisitor.visitSetAttachmentValueSpec()
                  └─ buildAttachmentTableReplaceQueries()
                      ├─ DELETE FROM attachments_table
                      └─ INSERT INTO attachments_table  ← 同步写入绑定（事务内）
```

---

## 六、核心文件索引

| 文件 | 作用 |
|---|---|
| `packages/db-main-prisma/prisma/postgres/schema.prisma` | 数据库真实 Schema 定义 |
| `apps/nestjs-backend/src/features/attachments/attachments.controller.ts` | 附件 HTTP 端点 |
| `apps/nestjs-backend/src/features/attachments/attachments.service.ts` | 核心上传/签名/通知逻辑 |
| `apps/nestjs-backend/src/features/attachments/attachments-table.service.ts` | v1 事件驱动的绑定写入 |
| `apps/nestjs-backend/src/features/attachments/attachments-storage.service.ts` | 预签名 URL 生成与缓存 |
| `apps/nestjs-backend/src/features/attachments/guard/auth.guard.ts` | 动态鉴权（登录/分享） |
| `apps/nestjs-backend/src/event-emitter/event-emitter.service.ts` | ShareDB op → 事件转换 |
| `apps/nestjs-backend/src/event-emitter/listeners/attachment.listener.ts` | 事件监听器，触发绑定写入 |
| `apps/nestjs-backend/src/share-db/repair-attachment-op/repair-attachment-op.service.ts` | ShareDB 操作附件修复 |
| `packages/v2/adapter-table-repository-postgres/src/record/attachments/attachmentTableMutations.ts` | v2 绑定 SQL 构建 |
| `packages/v2/adapter-table-repository-postgres/src/record/visitors/CellValueMutateVisitor.ts` | v2 更新时的绑定触发 |
| `packages/v2/adapter-table-repository-postgres/src/record/query-builder/insert/RecordInsertBuilder.ts` | v2 插入时的绑定触发 |
| `packages/v2/core/src/domain/table/fields/types/AttachmentField.ts` | v2 附件字段领域模型 |
| `packages/v2/core/src/domain/table/records/specs/values/SetAttachmentValueSpec.ts` | 附件值规格定义 |
| `packages/v2/core/src/domain/table/fields/visitors/FieldToSpecVisitor.ts` | 原始值→规格的转换 |
| `packages/v2/core/src/domain/table/fields/visitors/SetFieldValueSpecFactoryVisitor.ts` | 已验证值→规格的创建 |
| `packages/v2/core/src/ports/AttachmentLookupService.ts` | 附件查询端口 |
| `packages/v2/core/src/ports/AttachmentUrlSignerService.ts` | URL 签名端口 |
| `packages/openapi/src/attachment/signature.ts` | 签名 API 定义与 UploadType |
| `packages/openapi/src/record/upload-attachment.ts` | 记录上传 API 定义 |
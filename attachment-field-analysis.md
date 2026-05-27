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

系统存在 **三条上传通道**，由 `UploadType` 枚举区分（`packages/openapi/src/attachment/signature.ts:6-21`）：

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

### 3.1 数据模型设计

附件关联存储在两个层面：

#### 3.1.1 attachments 表（文件元数据）

```sql
CREATE TABLE attachments (
  token         VARCHAR PRIMARY KEY,   -- 上传时生成的唯一 token
  path          VARCHAR NOT NULL,      -- 对象存储路径
  hash          VARCHAR,               -- 文件哈希
  size          BIGINT,                -- 文件大小
  mimetype      VARCHAR,               -- MIME 类型
  width         INT,                   -- 图片宽度
  height        INT,                   -- 图片高度
  thumbnail_path VARCHAR,              -- 缩略图路径 (JSON: {sm, lg})
  created_by    VARCHAR,               -- 上传者 user ID
  created_time  TIMESTAMPTZ,
  deleted_time  TIMESTAMPTZ            -- 软删除
);
```

#### 3.1.2 attachments_table 表（行-列绑定）

```sql
CREATE TABLE attachments_table (
  id            BIGSERIAL PRIMARY KEY,
  table_id      VARCHAR NOT NULL,      -- 表 ID
  record_id     VARCHAR NOT NULL,      -- 行 (记录) ID
  field_id      VARCHAR NOT NULL,      -- 列 (字段) ID
  attachment_id VARCHAR NOT NULL,      -- 附件 ID (actxxx 格式)
  token         VARCHAR NOT NULL,      -- 附件 token (关联 attachments 表)
  name          VARCHAR,               -- 文件名
  created_by    VARCHAR,
  created_time  TIMESTAMPTZ,
  deleted_time  TIMESTAMPTZ
);
```

**关键点**：`attachments` 表以 `token` 为主键存储文件元数据，`attachments_table` 通过 `table_id + record_id + field_id + attachment_id` 四元组将文件与具体单元格绑定。

### 3.2 附件字段类型

`packages/v2/core/src/domain/table/fields/types/AttachmentField.ts` — `AttachmentField` 继承自 `Field`，类型为 `FieldType.attachment()`，无额外配置选项。

### 3.3 附件值规格 (Spec)

`packages/v2/core/src/domain/table/records/specs/values/SetAttachmentValueSpec.ts` 定义了附件单元格的值结构：

```typescript
interface AttachmentItem {
  id: string;           // 附件 ID (actxxx 格式)
  name: string;         // 文件名
  path: string;         // 存储路径
  token: string;        // 附件 token (关联 attachments 表)
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

### 3.4 绑定写入流程

#### 3.4.1 v1 路径（ShareDB 实时协作）

`apps/nestjs-backend/src/features/attachments/attachments-table.service.ts`：

- `createRecords()`：创建记录时，扫描所有 `Attachment` 类型字段，将附件项批量写入 `attachments_table`
- `updateRecords()`：更新记录时，对比 oldValue/newValue，计算需要删除和新增的绑定
- `deleteRecords()` / `deleteFields()` / `deleteTable()`：级联清理绑定关系

```typescript
// attachments-table.service.ts:12-19
createUniqueKey(tableId, fieldId, recordId, attachmentId) {
  return `${tableId}-${fieldId}-${recordId}-${attachmentId}`;
}
```

使用四元组生成唯一键，用于 diff 计算。

#### 3.4.2 v2 路径（领域模型）

`packages/v2/core/src/domain/table/fields/visitors/FieldToSpecVisitor.ts:470-495`：

```typescript
visitAttachmentField(field: AttachmentField): Result<ICellValueSpec, DomainError> {
  // 1. null → SetAttachmentValueSpec(null)
  // 2. 解析 AttachmentItem[] 数组格式
  // 3. typecast 模式下支持 "actxxx,actyyy" 字符串格式
  // 4. 返回 SetAttachmentValueSpec(fieldId, CellValue<AttachmentItem[]>)
}
```

`packages/v2/core/src/domain/table/fields/visitors/SetFieldValueSpecFactoryVisitor.ts:127-132`：

```typescript
visitAttachmentField(field: AttachmentField): Result<ICellValueSpec, DomainError> {
  const cellValue = CellValue.fromValidated<AttachmentItem[]>(
    this.value as AttachmentItem[] | null
  );
  return ok(new SetAttachmentValueSpec(field.id(), cellValue));
}
```

`SetAttachmentValueSpec.mutate()` 最终调用 `TableRecord.setFieldValue()` 将值写入领域模型的记录中。

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

4. 前端调用 uploadAttachment(tableId, recordId, fieldId, file)
   └─ POST /api/table/{tableId}/record/{recordId}/{fieldId}/uploadAttachment
      ├─ @Permissions('record|update') 校验
      └─ RecordOpenApiService.uploadAttachment()
         ├─ getValidateAttachmentRecord() 校验字段和记录
         ├─ attachmentsService.uploadFile() 或 uploadFromUrl()
         │  ├─ signature() 获得上传凭证
         │  └─ uploadStreamToStorage() 上传到存储
         ├─ 构造 { fieldId: [oldAttachments..., newAttachmentItem] }
         └─ updateRecord() 写入行数据
            ├─ v1: ShareDB op → repairAttachmentOp → attachmentsTable.createMany()
            └─ v2: SetAttachmentValueSpec → TableRecord.setFieldValue() → attachments_table
```

---

## 六、核心文件索引

| 文件 | 作用 |
|---|---|
| `apps/nestjs-backend/src/features/attachments/attachments.controller.ts` | 附件 HTTP 端点 |
| `apps/nestjs-backend/src/features/attachments/attachments.service.ts` | 核心上传/签名/通知逻辑 |
| `apps/nestjs-backend/src/features/attachments/attachments-table.service.ts` | 附件-行-列绑定写入 |
| `apps/nestjs-backend/src/features/attachments/attachments-storage.service.ts` | 预签名 URL 生成与缓存 |
| `apps/nestjs-backend/src/features/attachments/guard/auth.guard.ts` | 动态鉴权（登录/分享） |
| `apps/nestjs-backend/src/features/attachments/plugins/adapter.ts` | 存储适配器抽象 |
| `apps/nestjs-backend/src/features/attachments/plugins/local.ts` | 本地存储实现 |
| `apps/nestjs-backend/src/features/attachments/plugins/s3.ts` | S3 存储实现 |
| `apps/nestjs-backend/src/features/record/open-api/record-open-api.controller.ts` | 记录级上传端点 |
| `apps/nestjs-backend/src/features/record/open-api/record-open-api.service.ts` | 记录级上传服务 |
| `apps/nestjs-backend/src/share-db/repair-attachment-op/repair-attachment-op.service.ts` | ShareDB 操作附件修复 |
| `packages/v2/core/src/domain/table/fields/types/AttachmentField.ts` | v2 附件字段领域模型 |
| `packages/v2/core/src/domain/table/records/specs/values/SetAttachmentValueSpec.ts` | 附件值规格定义 |
| `packages/v2/core/src/domain/table/fields/visitors/FieldToSpecVisitor.ts` | 原始值→规格的转换 |
| `packages/v2/core/src/domain/table/fields/visitors/SetFieldValueSpecFactoryVisitor.ts` | 已验证值→规格的创建 |
| `packages/v2/core/src/ports/AttachmentLookupService.ts` | 附件查询端口 |
| `packages/v2/core/src/ports/AttachmentUrlSignerService.ts` | URL 签名端口 |
| `packages/openapi/src/attachment/signature.ts` | 签名 API 定义与 UploadType |
| `packages/openapi/src/record/upload-attachment.ts` | 记录上传 API 定义 |
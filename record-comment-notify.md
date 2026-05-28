# Teable Record Comment 与通知分发代码走向

## 总览

评论系统的代码分为三个核心阶段：**评论写入** → **@提及收件人解析** → **实时推送 & 通知分发**。后端入口集中在 `CommentOpenApiService`，实时推送基于 ShareDB Presence，通知分发由 `NotificationService` 统一处理。

---

## 一、评论写入

### 1.1 前端发起

用户在 `CommentEditor` 组件中编辑评论内容，按 Enter 提交：

```
CommentEditor (packages/sdk/src/components/comment/comment-editor/CommentEditor.tsx)
  ├── Plate 富文本编辑器，支持 MentionPlugin（@提及）、ImagePlugin（图片）
  ├── 编辑器值由 EditorTransform.editorValue2CommentValue() 转为 ICommentContent
  └── submit() 调用 createCommentFn / updateCommentFn
```

- **创建评论**：`createComment(tableId, recordId, { quoteId, content })`
- **更新评论**：`updateComment(tableId, recordId, commentId, { content })`

`createComment` 定义在 `packages/openapi/src/comment/create.ts`，发送 `POST /api/comment/{tableId}/{recordId}/create`。

### 1.2 后端 Controller

```
CommentOpenApiController (apps/nestjs-backend/src/features/comment/comment-open-api.controller.ts:92)
  @Post('/:recordId/create')
  @Permissions('record|comment')
  → commentOpenApiService.createComment(tableId, recordId, createCommentRo)
```

### 1.3 后端 Service — createComment

```
CommentOpenApiService.createComment() (apps/nestjs-backend/src/features/comment/comment-open-api.service.ts:355)

步骤：
1. generateCommentId() 生成评论 ID
2. filterCommentContent(content)
   - 剥离 img 节点的 url 字段（只保留 path）
   - 剥离 mention 节点的 name/avatar 字段（只保留 value 即 userId）
3. prismaService.comment.create() 写入 DB
   - id, tableId, recordId, content (JSON.stringify), createdBy (当前用户), quoteId
4. sendCommentNotify(tableId, recordId, id, { content, quoteId })  ← 通知分发
5. sendCommentPatch(tableId, recordId, CommentPatchType.CreateComment, result)  ← 实时推送
6. sendTableCommentPatch(tableId, recordId, CommentPatchType.CreateComment)  ← 表级计数推送
```

### 1.4 filterCommentContent — 入库前清洗

```
CommentOpenApiService.filterCommentContent() (comment-open-api.service.ts:330)

遍历 ICommentContent 数组：
- CommentNodeType.Img → 移除 url，只保留 path
- CommentNodeType.Paragraph → 遍历 children：
  - CommentNodeType.Mention → 移除 name/avatar，只保留 value (userId)
- 其他节点原样保留
```

**原因**：`name`、`avatar`、`url` 等属于展示态信息，入库时只保留引用 ID，读取时再动态填充。

---

## 二、@提及收件人解析

### 2.1 评论内容结构

评论内容类型定义在 `packages/openapi/src/comment/types.ts`：

```
ICommentContent = Array<IParagraphCommentContent | IImageCommentContent>

IParagraphCommentContent = {
  type: 'p',
  children: Array<ITextNode | IMentionNode | ILinkNode>
}

IMentionNode = {
  type: 'mention',
  value: string,       // userId
  name?: string,       // 读取时填充
  avatar?: string,     // 读取时填充
}
```

### 2.2 读取时填充用户信息

```
CommentOpenApiService.collectionsContext() (comment-open-api.service.ts:47)

遍历 content：
- 遇到 CommentNodeType.Img → 收集 path
- 遇到 CommentNodeType.Paragraph → 遍历 children
  - 遇到 CommentNodeType.Mention → 收集 child.value (userId)

返回 { imagePaths, mentionUserIds }
```

```
CommentOpenApiService.additionalContentContext() (comment-open-api.service.ts:131)

用 imagePathMap 和 mentionUserMap 回填：
- Img 节点：补上 url
- Mention 节点：补上 name、avatar
```

### 2.3 通知时的提及用户解析

```
CommentOpenApiService.sendCommentNotify() (comment-open-api.service.ts:603)

收件人来源有三个：
① 被引用评论的作者 → quoteId 查 comment.createdBy
② @提及的用户 → getMentionUserByContent(content)
③ 订阅了该记录评论的用户 → commentSubscription 表

最终去重并排除发送者自身：
subscribeUsersIds = uniq([...subscribeUsers, ...relativeUsers]).filter(id !== fromUserId)
```

```
CommentOpenApiService.getMentionUserByContent() (comment-open-api.service.ts:701)

解析流程：
1. JSON.parse(commentContentRaw) → ICommentContent
2. filter(type === 'p') 筛出段落节点
3. flatMap(children) 展开所有子节点
4. filter(type === 'mention') 筛出提及节点
5. map(mentionNode.value) 提取 userId 数组
```

---

## 三、实时推送（ShareDB Presence）

评论的实时推送不通过 ShareDB 的 OT 文档操作，而是通过 **Presence 机制**（轻量级的广播通道）。

### 3.1 Channel 命名规则

```
packages/core/src/models/channel.ts

评论频道：  getCommentChannel(tableId, recordId) → `__record_comment_${tableId}_${recordId}`
表级频道：  getTableCommentChannel(tableId)       → `__table_comment_${tableId}`
通知频道：  getUserNotificationChannel(userId)     → `__notification_user_${userId}`
```

### 3.2 后端推送 — sendCommentPatch

```
CommentOpenApiService.sendCommentPatch() (comment-open-api.service.ts:724)

1. createCommentPresence(tableId, recordId)
   - 获取 channel = getCommentChannel(tableId, recordId)
   - shareDbService.connect().getPresence(channel)
   - presence.create(channel) 创建 localPresence

2. 根据 CommentPatchType 决定数据：
   - CreateComment / UpdateComment / CreateReaction / DeleteReaction
     → getCommentDetail(commentId) 查完整评论
   - DeleteComment → { id: commentId }

3. localPresence.submit({ type, data })
   → 通过 ShareDB Presence 广播到频道 `__record_comment_${tableId}_${recordId}`
```

### 3.3 后端推送 — sendTableCommentPatch

```
CommentOpenApiService.sendTableCommentPatch() (comment-open-api.service.ts:763)

频道：`__table_comment_${tableId}`
数据：{ type, data: { recordId } }

用途：通知表格级别的评论计数变更
```

### 3.4 前端监听 — 评论列表实时更新

```
useCommentPatchListener() (packages/sdk/src/components/comment/comment-list/useCommentPatchListener.ts)

1. connection.getPresence(getCommentChannel(tableId, recordId))
2. presence.subscribe()
3. presence.on('receive', () => {
     const remoteData = remotePresences[presenceKey]
     cb?.(remoteData)
   })
```

在 `CommentList` 组件中回调处理：

```
CommentList (packages/sdk/src/components/comment/comment-list/CommentList.tsx:127)

commentListener(remoteData):
  CreateComment → 追加到列表，若是自己发的滚到底部，否则微滚
  DeleteComment → 从列表移除
  UpdateComment / CreateReaction / DeleteReaction → 替换对应项
```

### 3.5 前端监听 — 评论计数实时更新

```
useCommentCountMap() (packages/sdk/src/hooks/use-comment-count-map.ts)

频道：getTableCommentChannel(tableId) → `__table_comment_${tableId}`
监听 receive 事件：
  CreateComment → count++ 或新增 entry
  DeleteComment → count--，到 0 则移除 entry
```

---

## 四、通知分发

### 4.1 sendCommentNotify — 收件人汇总

```
CommentOpenApiService.sendCommentNotify() (comment-open-api.service.ts:603)

流程：
1. 获取 fromUser 信息 (ClsStore)
2. 收集 relativeUsers：
   a. quoteId → 查被引用评论的 createdBy
   b. getMentionUserByContent(content) → 提取 @提及的 userId
3. 查 baseId、tableName、primaryFieldId、recordName
4. 查 commentSubscription 表获取订阅用户
5. 合并去重，排除自己
6. 构造 i18n 消息：
   i18nKey: 'common.email.templates.notify.recordComment.message'
   context: { fromUserName, recordName, tableName, baseName }
7. 遍历 subscribeUsersIds，逐个调用 notificationService.sendCommentNotify()
```

### 4.2 NotificationService.sendCommentNotify

```
NotificationService.sendCommentNotify() (apps/nestjs-backend/src/features/notification/notification.service.ts:452)

1. 查 toUser，不存在则返回
2. type = NotificationTypeEnum.Comment
3. 构造 urlMeta = { baseId, tableId, recordId, commentId }
4. generateNotifyPath(Comment, urlMeta)
   → `/base/${baseId}/table/${tableId}?recordId=${recordId}&commentId=${commentId}`
5. 调用 sendCommonNotify({ path, fromUserId, toUserId, message, emailConfig }, Comment)
```

### 4.3 NotificationService.sendCommonNotify — 统一通知入口

```
NotificationService.sendCommonNotify() (notification.service.ts:305)

对每个收件人执行：
1. generateNotificationId() 生成通知 ID
2. 查 toUser 信息
3. 构造 Prisma.NotificationCreateInput 写入 DB
   - id, fromUserId, toUserId, type, urlPath, message, messageI18n, severity
4. createNotify(data) → prismaService.notification.create()
5. 查 unreadCount
6. 生成 notifyIcon (userId/userName/userAvatarUrl)
7. sendNotifyBySocket(toUserId, socketNotification) → 实时 WebSocket 推送
8. 如果 toUser.notifyMeta.email 开启：
   → mailSenderService.commonEmailOptions() 构造邮件
   → mailSenderService.sendMail() 发送邮件
```

### 4.4 WebSocket 推送 — sendNotifyBySocket

```
NotificationService.sendNotifyBySocket() (notification.service.ts:707)

1. channel = getUserNotificationChannel(toUserId) → `__notification_user_${userId}`
2. shareDbService.connect().getPresence(channel)
3. presence.create(notification.id) 创建 localPresence
4. localPresence.submit(data) 广播

data 结构 (INotificationBuffer)：
{
  notification: {
    id, message, messageI18n,
    notifyIcon: { userId, userName, userAvatarUrl },
    notifyType: 'comment',
    url, severity: 'info',
    isRead: false, createdTime
  },
  unreadCount: number
}
```

### 4.5 前端监听 — NotificationProvider

```
NotificationProvider (packages/sdk/src/context/notification/NotificationProvider.tsx)

1. channel = getUserNotificationChannel(user.id) → `__notification_user_${userId}`
2. connection.getPresence(channel)
3. presence.subscribe()
4. presence.on('receive', (id, res: INotificationBuffer) => {
     setNotification(res)
   })

通过 NotificationContext 暴露给全局，各组件通过 useNotification() 消费。
```

---

## 五、完整调用链路图

```
[前端] CommentEditor.submit()
  → createComment(tableId, recordId, { quoteId, content })
  → POST /api/comment/{tableId}/{recordId}/create

[后端] CommentOpenApiController.createComment()
  → CommentOpenApiService.createComment()
    ├── generateCommentId()
    ├── filterCommentContent(content)        // 清洗冗余字段
    ├── prismaService.comment.create()       // 写 DB
    │
    ├── sendCommentNotify()                  // ── 通知分发 ──
    │   ├── getMentionUserByContent()        // 解析 @提及 userId
    │   ├── quoteId → comment.createdBy      // 被引用评论作者
    │   ├── commentSubscription 查询         // 订阅用户
    │   ├── 合并去重排除自己
    │   └── for each userId:
    │       → NotificationService.sendCommentNotify()
    │         ├── generateNotifyPath()       // /base/xxx/table/xxx?recordId=xxx&commentId=xxx
    │         └── sendCommonNotify()
    │             ├── prismaService.notification.create()  // 通知写 DB
    │             ├── sendNotifyBySocket()                 // ShareDB Presence 推送
    │             │   channel: __notification_user_{userId}
    │             │   data: { notification, unreadCount }
    │             └── mailSenderService.sendMail()         // 邮件通知（可选）
    │
    ├── sendCommentPatch()                   // ── 评论实时推送 ──
    │   ├── channel: __record_comment_{tableId}_{recordId}
    │   ├── getCommentDetail() 查完整评论
    │   └── localPresence.submit({ type: CreateComment, data })
    │
    └── sendTableCommentPatch()              // ── 表级计数推送 ──
        ├── channel: __table_comment_{tableId}
        └── localPresence.submit({ type: CreateComment, data: { recordId } })

[前端实时监听]
  ├── useCommentPatchListener()
  │   channel: __record_comment_{tableId}_{recordId}
  │   → CommentList.commentListener() 更新评论列表
  │
  ├── useCommentCountMap()
  │   channel: __table_comment_{tableId}
  │   → 更新各记录的评论计数
  │
  └── NotificationProvider
      channel: __notification_user_{userId}
      → setNotification() 全局通知状态更新
```

---

## 六、订阅关系数据模型

### 6.1 表结构

`comment_subscription` 表由迁移 `20240919032636_add_comment` 创建：

```sql
CREATE TABLE "comment_subscription" (
    "table_id"   TEXT NOT NULL,
    "record_id"  TEXT NOT NULL,
    "created_by" TEXT NOT NULL,
    "created_time" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);

UNIQUE INDEX ("table_id", "record_id")   -- 联合唯一约束
INDEX ("table_id", "record_id")          -- 查询索引
```

对应的 Prisma 模型（`schema.prisma:757`）：

```
model CommentSubscription {
  id          String   @id @default(cuid())
  tableId     String   @map("table_id")
  recordId    String   @map("record_id")
  createdBy   String   @map("created_by")
  createdTime DateTime @default(now()) @map("created_time")

  @@unique([tableId, recordId])
  @@index([tableId, recordId])
  @@map("comment_subscription")
}
```

### 6.2 订阅的唯一约束语义与重复订阅行为

`@@unique([tableId, recordId])` 意味着 **同一 (tableId, recordId) 组合只能有一条订阅记录**。`createdBy` 不在唯一约束中，因此同一记录只允许存在一条订阅行。

**重复订阅时的实际行为：报错，而非覆盖。**

代码证据——`subscribeComment` 使用的是 `prismaService.commentSubscription.create()`，而非 `upsert`：

```typescript
// comment-open-api.service.ts:531
async subscribeComment(tableId: string, recordId: string) {
  await this.prismaService.commentSubscription.create({  // ← create，不是 upsert
    data: {
      tableId,
      recordId,
      createdBy: this.cls.get('user.id'),
    },
  });
}
```

当同一 `(tableId, recordId)` 已存在记录时，Prisma 会抛出 `P2002` 唯一约束冲突错误，转化为 HTTP 500 返回给前端。**不会覆盖之前的订阅者**。

**前端防护机制**：

```typescript
// CommentHeader.tsx:55-61
const subscribeHandler = () => {
  if (!subscribeStatus) {          // ← 检查是否已有订阅
    createSubscribe({ tableId: tableId!, recordId: recordId! });
  } else {
    deleteSubscribeFn({ tableId: tableId!, recordId: recordId! });
  }
};

const subscribeComment = () => {
  if (!subscribeStatus) {          // ← 同样的检查
    createSubscribe({ tableId: tableId!, recordId: recordId! });
  }
};
```

前端先通过 `getCommentSubscribe` API 查询当前订阅状态。如果 `subscribeStatus` 非空，`subscribeComment` 会直接跳过，不会尝试重复创建。

**但这里存在一个跨用户问题**：`getSubscribeDetail` 返回的是 `(tableId, recordId)` 对应的**唯一一条订阅记录**，无论 `createdBy` 是谁。这意味着：

1. 用户 A 订阅了记录 X → `comment_subscription` 中存在 `(tableX, recordX, createdBy: userA)`
2. 用户 B 打开记录 X 的评论面板 → `getSubscribeDetail` 返回 `{ tableId: 'tableX', recordId: 'recordX', createdBy: 'userA' }` （非 null）
3. 用户 B 看到 `subscribeStatus` 非空 → UI 显示"通知全部"（已订阅状态）
4. 但实际上**只有 userA 会收到订阅通知**，userB 的 `subscribeStatus` 是 userA 的订阅

同时，`unsubscribeComment` 按 `(tableId, recordId)` 删除，**不校验 createdBy**：

```typescript
// comment-open-api.service.ts:541
async unsubscribeComment(tableId: string, recordId: string) {
  await this.prismaService.commentSubscription.delete({
    where: {
      tableId_recordId: { tableId, recordId },  // ← 无 createdBy 条件
    },
  });
}
```

所以用户 B 点击"取消订阅"时，实际删除的是用户 A 的订阅。

**结论**：订阅模型是一个**记录级开关**（per-record toggle），而非用户-记录级关系。同一记录只能由一个人"持有"订阅，其他人看到的是这个人的订阅状态。这在多人协作场景下会导致误判——用户以为自己订阅了，实际通知只会发给记录中存储的那个 `createdBy`。

### 6.3 订阅 CRUD

| 操作 | API | Controller 方法 | 权限 |
|------|-----|----------------|------|
| 查询订阅 | `GET /api/comment/{tableId}/{recordId}/subscribe` | `getSubscribeDetail` | `record|read` |
| 创建订阅 | `POST /api/comment/{tableId}/{recordId}/subscribe` | `subscribeComment` | `record|read` |
| 取消订阅 | `DELETE /api/comment/{tableId}/{recordId}/subscribe` | `unsubscribeComment` | `record|read` |

注意：订阅/取消订阅只需要 `record|read` 权限，而发评论需要 `record|comment` 权限。这意味着**有读权限就能订阅通知，但不一定能发评论**。

### 6.4 订阅与通知的关系

在 `sendCommentNotify` 中（`comment-open-api.service.ts:669`），订阅用户的查询逻辑为：

```typescript
const notifyUsers = await this.prismaService.commentSubscription.findMany({
  where: { tableId, recordId },
  select: { createdBy: true },
});

const subscribeUsersIds = Array.from(
  new Set([...notifyUsers.map(({ createdBy }) => createdBy), ...relativeUsers])
).filter((userId) => userId !== fromUserId);
```

由于唯一约束 `(tableId, recordId)`，`findMany` 实际最多返回一条记录。因此订阅用户的贡献**最多是一个人**。而 `relativeUsers`（@提及 + 被引用评论作者）可以包含多人，最终合并后才是一个完整的收件人列表。

### 6.5 comment_subscription 结构演变历史

`comment_subscription` 表经历了三次结构变更，从无主键表逐步演化为有主键的标准表。

**阶段 1：初始创建（20240919032636_add_comment）

```sql
-- 20240919032636_add_comment/migration.sql:17-32
CREATE TABLE "comment_subscription" (
    "table_id" TEXT NOT NULL,        -- 无 id 列！
    "record_id" TEXT NOT NULL,
    "created_by" TEXT NOT NULL,
    "created_time" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX "comment_subscription_table_id_record_id_idx" ON "comment_subscription"("table_id", "record_id");
CREATE UNIQUE INDEX "comment_subscription_table_id_record_id_key" ON "comment_subscription"("table_id", "record_id");

-- ⚠️ 没有 PRIMARY KEY 约束！
```

**关键特征**：
- 没有 `id` 列
- 没有 `PRIMARY KEY` 约束
- 只有 `(table_id, record_id)` 作为唯一索引（但不是主键）

这是一张**无主键表**，不符合关系型数据库的最佳实践。Prisma 要求所有模型必须有主键，所以当时的 Prisma schema 可能使用了 `@@unique([tableId, recordId]` 作为复合主键，或者通过其他方式绕过。

**阶段 2：补主键（20250509062715_require_primary_key）

```sql
-- 20250509062715_require_primary_key/migration.sql:2-4
ALTER TABLE "comment_subscription" ADD COLUMN "id" TEXT DEFAULT substring(md5(random()::text), 1, 25),
ADD CONSTRAINT "comment_subscription_pkey" PRIMARY KEY ("id");
```

**关键变更**：
- 新增 `id` 列，默认值为 `substring(md5(random()::text), 1, 25)`（PostgreSQL 原生 SQL 级别的随机字符串）
- 设为 `PRIMARY KEY` 约束

为什么用 `md5(random()::text)` 而不是 `cuid()`：
- 这是数据库级别的默认值，不是应用级别的
- 对已有数据自动填充 ID

**阶段 3：移除默认值（20250922111648_add_indexes）

```sql
-- 20250922111648_add_indexes/migration.sql:2
ALTER TABLE "comment_subscription" ALTER COLUMN "id" DROP DEFAULT;
```

**关键变更**：
- 移除 `id` 列的默认值

**当前 Prisma schema（schema.prisma:757）

```
model CommentSubscription {
  id          String   @id @default(cuid())   -- Prisma 应用级别
  tableId     String   @map("table_id")
  recordId    String   @map("record_id")
  createdBy   String   @map("created_by")
  createdTime DateTime @default(now()) @map("created_time")

  @@unique([tableId, recordId])
  @@index([tableId, recordId])
  @@map("comment_subscription")
```

**当前线上结构的判断方法

| 判断维度 | 检查方法 |
|--------|---------|
| **阶段 1（无主键） | 查 `information_schema.columns` 中 `comment_subscription` 没有 `id` 列 |
| **阶段 2（有默认值的主键） | `id` 列存在且有 `DEFAULT` 不为空 |
| **阶段 3（无默认值的主键） | `id` 列存在且 `column_default` 为 null |

**SQL 判断脚本：

```sql
-- 检查 id 列是否存在
SELECT column_name, column_default, is_nullable
FROM information_schema.columns
WHERE table_name = 'comment_subscription'
  AND column_name = 'id';

-- 检查主键约束
SELECT constraint_name, constraint_type
FROM information_schema.table_constraints
WHERE table_name = 'comment_subscription'
  AND constraint_type = 'PRIMARY KEY';
```

---

## 七、权限边界

### 7.1 Controller 层权限注解汇总

| 端点 | 方法 | @Permissions | 实际含义 |
|------|------|-------------|---------|
| `GET /:recordId/count` | getRecordCommentCount | `view\|read` | 查看视图即可读计数 |
| `GET /count` | getTableCommentCount | `view\|read` | 查看视图即可读计数 |
| `GET /:recordId/attachment/:path` | getAttachmentPresignedUrl | `record\|read` | 读记录即可获取附件 |
| `GET /:recordId/subscribe` | getSubscribeDetail | `record\|read` | 读记录即可查订阅 |
| `POST /:recordId/subscribe` | subscribeComment | `record\|read` | 读记录即可订阅 |
| `DELETE /:recordId/subscribe` | unsubscribeComment | `record\|read` | 读记录即可取消订阅 |
| `GET /:recordId/list` | getCommentList | `record\|read` | 读记录即可看评论列表 |
| `POST /:recordId/create` | createComment | `record\|comment` | 需要评论权限 |
| `GET /:recordId/:commentId` | getCommentDetail | `record\|read` | 读记录即可看评论详情 |
| `PATCH /:recordId/:commentId` | updateComment | `record\|comment` | 需要评论权限 |
| `DELETE /:recordId/:commentId` | deleteComment | `record\|read` ⚠️ | 删除仅需读权限 |
| `DELETE /:recordId/:commentId/reaction` | deleteCommentReaction | `record\|comment` | 需要评论权限 |
| `PATCH /:recordId/:commentId/reaction` | updateCommentReaction | `record\|comment` | 需要评论权限 |

### 7.2 权限边界问题

**⚠️ deleteComment 仅需 `record|read` 权限**

Controller 第 121-129 行：

```typescript
@Delete('/:recordId/:commentId')
@Permissions('record|read')   // ← 只检查了 record|read
async deleteComment(...) { ... }
```

而 `createComment`、`updateComment`、Reaction 操作都要求 `record|comment`。删除评论只要求 `record|read` 看起来是一个**权限遗漏**——拥有只读权限的用户理论上可以删除评论。

不过 Service 层有二次保护：`deleteComment` 通过 `createdBy: this.cls.get('user.id')` 限定只能删除自己的评论。所以实际效果是：**有 `record|read` 权限的用户只能删除自己发的评论，不能删除别人的**。但这与 `record|comment` 的语义仍不一致——拥有 `record|comment` 权限的管理员反而无法通过此接口删除他人的评论。

### 7.3 Service 层的二次校验

| 操作 | Service 层额外校验 |
|------|------------------|
| `updateComment` | `where: { id: commentId, createdBy: this.cls.get('user.id') }` → 只能改自己的 |
| `deleteComment` | `where: { id: commentId, createdBy: this.cls.get('user.id') }` → 只能删自己的 |
| `createCommentReaction` | 无 createdBy 限制（任何人可对任何评论表态） |
| `deleteCommentReaction` | 通过 userId 过滤 reaction → 只能删自己的表情 |

### 7.4 角色与 `record|comment` 权限矩阵

```
packages/core/src/auth/role/constant.ts

角色            record|comment
─────────────────────────────
Owner           true
Creator         true
Editor          true
Commenter       true      ← 唯一与 read 的区别
Viewer          false

packages/core/src/auth/role/share.ts
shareViewPermissions:
  record|comment: false    ← 分享链接无评论权限

packages/core/src/auth/role/template.ts
TemplateRolePermission:
  record|comment: false    ← 模板无评论权限
```

**关键边界**：
- `Commenter` 角色是专门为评论设计的：它有 `record|read` + `record|comment`，但无 `record|update`/`record|delete`
- 分享链接和模板用户都无法评论，也无法订阅通知（因为订阅 API 的 `@AllowAnonymous` 可能允许，但 `record|read` 在分享场景下由 share 权限控制）
- 订阅通知只需要 `record|read`，意味着 Viewer 也能订阅但无法评论

### 7.5 Controller 级 @AllowAnonymous

整个 `CommentOpenApiController` 标注了 `@AllowAnonymous()`（第 25 行），这表示评论接口**允许匿名访问**。但具体权限仍由 `@Permissions` 守卫在 `PermissionGuard` 中校验。`@AllowAnonymous` 的效果是：当用户未登录时，走分享链接/模板的权限降级逻辑（见 `permission.guard.ts:410` 的 `permissionCheckWithPublicFallback`），而非直接放行。

---

## 八、updateComment 与 createComment 的数据清洗一致性

### 8.1 createComment 的清洗流程

```
createComment (comment-open-api.service.ts:355)
1. filterCommentContent(createCommentRo.content)  ← 清洗
2. prismaService.comment.create({ content: JSON.stringify(content) })
3. sendCommentNotify(...)
4. sendCommentPatch(...)
5. sendTableCommentPatch(...)
```

### 8.2 updateComment 的清洗流程

```
updateComment (comment-open-api.service.ts:384)
1. prismaService.comment.update({
     where: { id: commentId, createdBy: this.cls.get('user.id') },
     data: {
       content: JSON.stringify(updateCommentRo.content),  ← ⚠️ 未清洗！
       lastModifiedTime: new Date().toISOString(),
     }
   })
2. sendCommentPatch(...)
3. sendCommentNotify(...)
```

### 8.3 不一致分析

**`updateComment` 没有调用 `filterCommentContent`**，直接 `JSON.stringify(updateCommentRo.content)` 入库。

这意味着：

| 场景 | createComment | updateComment |
|------|:---:|:---:|
| Img 节点剥离 `url` | ✅ 已清洗 | ❌ 未清洗，`url` 可能入库 |
| Mention 节点剥离 `name`/`avatar` | ✅ 已清洗 | ❌ 未清洗，`name`/`avatar` 可能入库 |

**影响**：

1. **数据冗余**：updateComment 写入的 content 可能包含 `name`/`avatar`/`url` 等展示态字段，而 createComment 写入的不会。同一个 `comment` 表的 `content` 列中，不同评论的 JSON 结构可能不一致。

2. **读取时回填覆盖**：`additionalContentContext` 在回填时，会用最新的 `mentionUserMap` 和 `imagePathMap` 覆盖 `name`/`avatar`/`url`，所以**读取结果不受影响**——冗余字段会被正确覆盖。但如果用户改了名/换了头像，updateComment 写入的旧 `name`/`avatar` 在被覆盖前是过时的。

3. **`sendCommentNotify` 中的 `getMentionUserByContent` 不受影响**：该方法只从 content 中提取 `type === 'mention'` 节点的 `value` (userId)，不关心 `name`/`avatar` 是否存在。

4. **实际风险较低**：前端 `updateComment` 传入的 content 来自 `EditorTransform.editorValue2CommentValue(value)`，与 `createComment` 相同的转换逻辑。但这个转换是否已经剥离了冗余字段取决于前端 Plate 编辑器的序列化行为——从前端传来的 content 本身可能就不包含 `url`（图片路径）和 `name`/`avatar`（mention 只有 `value`），所以实际可能不会触发问题。但这是一个**应然与实然的差异**——后端应统一清洗。

**建议**：在 `updateComment` 中也调用 `filterCommentContent(updateCommentRo.content)` 后再入库，保持数据一致性。

### 8.4 通知触发的差异

| 操作 | 触发 sendCommentNotify | 触发 sendCommentPatch | 触发 sendTableCommentPatch |
|------|:---:|:---:|:---:|
| createComment | ✅ await | ✅ 同步 | ✅ 同步 |
| updateComment | ✅ await | ✅ 同步 | ❌ 不触发 |
| deleteComment | ❌ 不触发 | ✅ 同步 | ✅ 同步 |
| createCommentReaction | ✅ await | ✅ 同步 | ❌ 不触发 |
| deleteCommentReaction | ❌ 不触发 | ✅ 同步 | ❌ 不触发 |

注意：
- `updateComment` 触发通知但**不触发表级计数推送**，这是合理的（更新不改变计数）
- `deleteComment` 不触发通知，删除后不通知其他用户（靠 Presence 实时推送列表更新）
- `deleteCommentReaction` 不触发通知，emoji 取消无需通知

---

## 九、通知分发的可靠性与异步语义

### 9.1 await 语义分析

`createComment` 中的调用顺序（`comment-open-api.service.ts:355`）：

```typescript
// 1. 写 DB —— 必须等
const result = await this.prismaService.comment.create({ ... });

// 2. 通知分发 —— await 但只等查询阶段
await this.sendCommentNotify(tableId, recordId, id, { ... });

// 3. 评论实时推送 —— 不等（fire-and-forget）
this.sendCommentPatch(tableId, recordId, CommentPatchType.CreateComment, result);

// 4. 表级计数推送 —— 不等（fire-and-forget）
this.sendTableCommentPatch(tableId, recordId, CommentPatchType.CreateComment);
```

| 步骤 | await | 实际等待范围 |
|------|:---:|------|
| `comment.create` | ✅ | DB 写入完成，失败则整体报错 |
| `sendCommentNotify` | ✅ | **仅等待内部查询阶段**，不等待通知 DB 写入、推送或邮件 |
| `sendCommentPatch` | ❌ | 完全 fire-and-forget |
| `sendTableCommentPatch` | ❌ | 完全 fire-and-forget |

**关键纠正**：虽然 `createComment` 中对 `sendCommentNotify` 使用了 `await`，但这并不意味着等待通知全部完成。详见 9.2 的逐行追踪。

### 9.2 sendCommentNotify 内部的完整 await 链——逐行追踪

```typescript
// comment-open-api.service.ts:603
private async sendCommentNotify(
  tableId: string, recordId: string, commentId: string,
  notifyVo: { quoteId: string | null; content: string | null }
) {
  // ── 阶段 A：查询阶段（被外层 await 等待）──

  // A1. 查被引用评论作者
  if (quoteId) {
    const { createdBy: quoteCommentCreator } =
      (await this.prismaService.comment.findUnique({ ... })) || {};  // ← await ✅
    quoteCommentCreator && relativeUsers.push(quoteCommentCreator);
  }

  // A2. 提取 @提及用户（纯同步计算）
  const mentionUsers = this.getMentionUserByContent(content);        // ← 同步

  // A3. 查 tableMeta
  const { baseId, name: tableName } =
    (await this.prismaService.tableMeta.findFirst({ ... })) || {};   // ← await ✅

  // A4. 查 primary field
  const { id: fieldId } =
    (await this.prismaService.field.findFirst({ ... })) || {};      // ← await ✅

  if (!baseId || !fieldId) { return; }                              // 提前退出

  // A5. 查 baseName
  const { name: baseName } = await this.prismaService.base
    .findUniqueOrThrow({ ... });                                    // ← await ✅

  // A6. 查 recordName
  const recordName = await this.recordService
    .getCellValue(tableId, recordId, fieldId);                      // ← await ✅

  // A7. 查 commentSubscription
  const notifyUsers = await this.prismaService.commentSubscription
    .findMany({ where: { tableId, recordId }, ... });               // ← await ✅

  // A8. 合并去重
  const subscribeUsersIds = Array.from(new Set([...])).filter(...); // ← 同步

  // A9. 构造 i18n 消息
  const message: ILocalization<I18nPath> = { i18nKey: '...', context: {...} };  // ← 同步

  // ── 阶段 B：通知分发阶段（不被外层 await 等待）──

  // B1. forEach 遍历收件人 —— 同步循环，异步操作未被 await
  subscribeUsersIds.forEach((userId) => {
    this.notificationService.sendCommentNotify({                     // ← 无 await ❌
      baseId, tableId, recordId, commentId,
      toUserId: userId, message, fromUserId,
    });
    // 返回的 Promise 被丢弃，无人 await 或 catch
  });
  // forEach 循环本身同步完成 → sendCommentNotify 的 async 函数体结束
  // → 外层的 await resolve → createComment 继续执行后续代码
}
```

**分界线**：A1-A9 是查询阶段，B1 是分发阶段。`createComment` 中的 `await this.sendCommentNotify(...)` **只等待阶段 A 完成**。阶段 B 的所有异步操作（通知 DB 写入、WebSocket 推送、邮件发送）全部在后台执行，不被等待。

### 9.3 NotificationService.sendCommentNotify 内部的二次无 await

即使 `forEach` 中对 `notificationService.sendCommentNotify` 加了 `await`，仍然不会等待通知完成，因为它**内部也没有 await `sendCommonNotify`**：

```typescript
// notification.service.ts:452
async sendCommentNotify(params: {...}) {
  const { toUserId, tableId, message, baseId, commentId, recordId, fromUserId } = params;
  const toUser = await this.userService.getUserById(toUserId);      // ← await ✅
  if (!toUser) { return; }

  const type = NotificationTypeEnum.Comment;
  const urlMeta = notificationUrlSchema.parse({...});
  const notifyPath = this.generateNotifyPath(type, urlMeta);

  this.sendCommonNotify(                                             // ← 无 await ❌
    { path: notifyPath, fromUserId, toUserId, message, ... },
    type
  );
  // sendCommonNotify 返回的 Promise 被丢弃
}
```

**双重无 await**：`forEach` 无 await → `sendCommentNotify` 内部也无 await `sendCommonNotify` → 即使外层修了 forEach，仍然等不到通知完成。

### 9.4 sendCommonNotify 内部的可靠性

```typescript
// notification.service.ts:305
async sendCommonNotify(...) {
  const notifyData = await this.createNotify(data);                  // ① 写 DB — await ✅
  const unreadCount = (await this.unreadCount(...)).unreadCount;     // ② 查未读数 — await ✅
  // ... 构造 socketNotification（同步）...
  this.sendNotifyBySocket(toUser.id, socketNotification);            // ③ WebSocket 推送 — 无 await ❌
  if (emailConfig && toUser.notifyMeta && toUser.notifyMeta.email) {
    const emailOptions = await this.mailSenderService               // ④ 构造邮件模板 — await ✅
      .commonEmailOptions({...});
    this.mailSenderService.sendMail({...}, {...});                   // ⑤ 发送邮件 — 无 await ❌
  }
}
```

| 步骤 | await | 失败时行为 | 是否影响接口返回 |
|------|:---:|---------|:---:|
| ① `notification.create` | ✅ | DB 写入失败 → Promise reject | ❌ 不影响 |
| ② `unreadCount` | ✅ | 查询失败 → Promise reject | ❌ 不影响 |
| ③ `sendNotifyBySocket` | ❌ | 推送失败 → resolve（不 reject），仅记日志 | ❌ 不影响 |
| ④ `commonEmailOptions` | ✅ | 构造失败 → Promise reject | ❌ 不影响 |
| ⑤ `mailSenderService.sendMail` | ❌ | 邮件失败 → catch 后 log error，返回 false | ❌ 不影响 |

**为什么全部不影响接口返回？** 因为从 `createComment` 到 `sendCommonNotify` 之间有**两层无 await**：`forEach` 无 await + `sendCommentNotify` 内部无 await `sendCommonNotify`。所有从 `sendCommonNotify` 抛出的异常都会变成**unhandled Promise rejection**，不会被 NestJS 的 `GlobalExceptionFilter` 捕获（它只处理请求上下文内的异常），也不会传播到 `createComment`。

**WebSocket 推送失败的具体行为**：

```typescript
// notification.service.ts:707
private async sendNotifyBySocket(toUserId: string, data: INotificationBuffer) {
  const channel = getUserNotificationChannel(toUserId);
  const presence = this.shareDbService.connect().getPresence(channel);
  const localPresence = presence.create(data.notification.id);

  return new Promise((resolve) => {
    localPresence.submit(data, (error) => {
      error && this.logger.error(error);  // 只记日志
      resolve(data);                       // 无论成功失败都 resolve（永不 reject）
    });
  });
}
```

即使 ShareDB Presence submit 失败，Promise 也会 resolve。通知已经写入了 DB，用户下次刷新页面仍可从通知列表 API 获取。

**邮件发送失败的具体行为**：

```typescript
// mail-sender.service.ts:248
return sender.catch((reason) => {
  if (reason) {
    console.error(reason);
    this.logger.error(`Mail sending failed: ${reason.message}`, reason.stack);
  }
  return false;   // 吞掉错误，返回 false
});
```

邮件发送是**最大努力交付**：失败只记日志，不重试，不回滚通知记录，不向调用方抛出异常。

### 9.5 完整异步调用链路图

```
createComment()                                              ← API 入口
  ├── await prismaService.comment.create()                   ← 等待 ✅ ── 阶段1: 主操作
  ├── await sendCommentNotify()                              ← 等待 ✅ ── 阶段2: 查询
  │     ├── await comment.findUnique()                       ← 等待 ✅    (查被引用作者)
  │     ├── getMentionUserByContent()                        ← 同步       (提取@提及)
  │     ├── await tableMeta.findFirst()                      ← 等待 ✅    (查baseId)
  │     ├── await field.findFirst()                          ← 等待 ✅    (查primaryField)
  │     ├── await base.findUniqueOrThrow()                   ← 等待 ✅    (查baseName)
  │     ├── await recordService.getCellValue()               ← 等待 ✅    (查recordName)
  │     ├── await commentSubscription.findMany()             ← 等待 ✅    (查订阅者)
  │     ├── 合并去重                                         ← 同步
  │     └── forEach(userId =>                                ← 同步循环 ── 分界线
  │           notificationService.sendCommentNotify()        ← 无await ❌ 🔥 fire-and-forget
  │             ├── await userService.getUserById()          ← await ✅   但Promise被丢弃
  │             └── sendCommonNotify()                       ← 无await ❌ 🔥 二次fire-and-forget
  │                   ├── await createNotify()               ← await ✅   但Promise被丢弃
  │                   ├── await unreadCount()                ← await ✅   但Promise被丢弃
  │                   ├── sendNotifyBySocket()               ← 无await ❌  🔥
  │                   │     └── localPresence.submit()       ← 永远resolve
  │                   └── mailSenderService.sendMail()       ← 无await ❌  🔥
  │                         └── sender.catch(() => false)    ← 吞掉错误
  │         )                                                ← forEach 结束
  │     ← sendCommentNotify 的 Promise resolve（只等了查询阶段）
  ├── sendCommentPatch()                                     ← 无await ❌ 🔥 ── 阶段3: 实时推送
  └── sendTableCommentPatch()                                ← 无await ❌ 🔥 ── 阶段4: 计数推送

返回评论数据给前端
```

**结论**：`createComment` 的 `await sendCommentNotify` 实际只等待了收件人查询阶段。所有通知写入、推送、邮件均为 fire-and-forget。这些操作的失败**不会导致 API 返回错误**，而是变成 unhandled Promise rejection。

### 9.6 失败场景汇总

| 失败点 | 时机 | 是否影响 API 返回 | 影响 | 可恢复性 |
|--------|------|:---:|------|---------|
| 评论 DB 写入失败 | 阶段1 | ✅ 影响 | API 返回错误，评论不存在 | 用户可重试 |
| sendCommentNotify 查询阶段失败 | 阶段2 | ✅ 影响 | API 返回错误，但评论已写入 DB ⚠️ | 评论已存在，通知未发 |
| 通知 DB 写入失败 (createNotify) | 阶段2 之后的 FAF | ❌ 不影响 | 该收件人无通知记录 | 不可自动恢复 |
| WebSocket 评论推送失败 | 阶段3 FAF | ❌ 不影响 | 其他在线用户看不到实时更新 | 刷新页面可恢复 |
| WebSocket 通知推送失败 | 阶段2 之后的 FAF | ❌ 不影响 | 在线用户收不到实时通知弹窗 | 刷新页面后从通知列表可恢复 |
| 邮件发送失败 | 阶段2 之后的 FAF | ❌ 不影响 | 收件人收不到邮件 | 不可自动恢复，无重试机制 |

**⚠️ 注意**：阶段 2（查询阶段）的失败**会影响 API 返回**，因为 `createComment` 中 `await sendCommentNotify` 会等待这些查询。如果查询阶段抛出异常（如 `base.findUniqueOrThrow` 找不到记录），评论已经写入了 DB，但 API 返回 500——**用户看到的是创建失败，但评论实际上已经存在**。

### 9.7 可靠性评估

1. **评论写入**：强一致，失败即回滚，用户感知明确。

2. **sendCommentNotify 查询阶段**：与评论写入非原子。查询失败时评论已入库但 API 报错，用户可能重试导致重复评论。

3. **通知持久化**：完全 fire-and-forget。`sendCommonNotify` 中的 `createNotify` 虽然用了 `await`，但因为外层两层无 await，其 Promise 被丢弃。通知 DB 写入失败不会影响任何人——不会报错、不会重试、不会补偿。但由于同一评论多次调用会生成不同 `notificationId`，不具幂等性。

4. **实时推送**：尽力交付（best-effort）。ShareDB Presence 没有持久化保证，断线期间的消息不会重放。但评论和通知已写入 DB，刷新页面即可恢复。

5. **邮件通知**：尽力交付，无重试。SMTP 发送失败后仅记日志，不会重发。

6. **幂等性修正**：之前的结论"自然幂等"是错误的。`createComment` 每次调用都生成新的 `commentId`，**完全不具备幂等性**。代码证据：

   ```typescript
   // comment-open-api.service.ts:355
   async createComment(tableId: string, recordId: string, createCommentRo: ICreateCommentRo) {
     const id = generateCommentId();  // ← 函数入口第一行就生成 ID
     // ...
     await this.prismaService.comment.create({ data: { id, ... } });
   }
   ```

   `generateCommentId()` 定义（`id-generator.ts:118`）：
   ```typescript
   export function generateCommentId() {
     return IdPrefix.Comment + getRandomString(16);  // 随机字符串，每次调用不同
   }
   ```

   **后果**：如果用户因为 API 返回 500（查询阶段失败）而重试发送评论，每次重试都会在 DB 中插入一条新评论——**产生重复评论**。这是一个真实的 bug，不是理论风险。

   相比之下，`sendCommentNotify` 中对每个 `userId` 调用 `sendCommonNotify` 也生成新的 `notificationId`，如果被多次调用（如网络超时后重试），确实会导致**同一评论对同一用户产生多条通知**。

### 9.8 unhandledRejection → uncaughtException 的进程级错误传播路径

那些被丢弃的 Promise（`sendCommentNotify` 的 forEach、`sendCommonNotify`、`sendCommentPatch`、`sendTableCommentPatch`）在失败时会经历以下传播路径：

```
[1] Promise 被拒绝且无 .catch() 处理
    │
    ▼
[2] 触发 Node.js unhandledRejection 事件        (bootstrap.ts:79)
    process.on('unhandledRejection', (reason, promise) => {
      logger.error(`Unhandled Rejection at: ${promise}, reason: ${reason}`);
      throw reason;  // ← 将异步拒绝转化为同步抛出
    })
    │
    ▼
[3] throw reason 抛出同步异常（未被 try-catch 捕获）
    │
    ▼
[4] 触发 Node.js uncaughtException 事件         (bootstrap.ts:84)
    process.on('uncaughtException', (error) => {
      logger.error(error);  // ← 只记日志，不退出进程
    })
    │
    ▼
[5] 进程继续运行（状态可能已损坏）
```

**GlobalExceptionFilter 的边界**：

`GlobalExceptionFilter`（`apps/nestjs-backend/src/filter/global-exception.filter.ts`）只捕获**NestJS 请求处理管道内**的异常，即从 Controller 入口到返回响应过程中抛出的异常。它无法捕获：

- `unhandledRejection`（Promise 被丢弃，不在请求上下文内）
- `uncaughtException`（同步异常，不在请求上下文内）
- 定时器回调、事件监听器中的异常

**代码证据 — bootstrap.ts:79-86**：

```typescript
process.on('unhandledRejection', (reason: string, promise: Promise<unknown>) => {
  logger.error(`Unhandled Rejection at: ${promise}, reason: ${reason}`);
  throw reason;  // ← 关键：主动 throw，将异步错误升级为同步错误
});

process.on('uncaughtException', (error) => {
  logger.error(error);  // ← 只记日志
});
```

**重要细节**：`unhandledRejection` 监听器中调用了 `throw reason`，这会将异步的 Promise 拒绝转化为同步的未捕获异常，进而触发 `uncaughtException`。如果没有这个 `throw`，Promise 拒绝只会被记录然后被吞掉。

**进程稳定性风险**：

按照 Node.js 官方文档，`uncaughtException` 发生后继续运行进程是不安全的，因为应用状态可能已损坏。但 Teable 的 `uncaughtException` 监听器只记日志、不退出进程——如果通知 DB 写入失败导致了某些资源泄漏或状态不一致，进程会带病继续运行。

### 9.9 createComment 查询阶段失败时的精确顺序与用户可见性

当 `sendCommentNotify` 的查询阶段（阶段 2）抛出异常时，执行顺序和对用户的影响需要精确分析。

**正常路径的代码结构**（`comment-open-api.service.ts:355`）：

```typescript
async createComment(tableId: string, recordId: string, createCommentRo: ICreateCommentRo) {
  const id = generateCommentId();
  const content = await this.filterCommentContent(createCommentRo.content);

  // 步骤 1：写评论到 DB —— await
  const result = await this.prismaService.comment.create({
    data: { id, tableId, recordId, content: JSON.stringify(content), ... }
  });  // ← 成功后评论已持久化，事务已提交

  // 步骤 2：通知查询阶段 —— await
  await this.sendCommentNotify(tableId, recordId, id, {
    content: result.content, quoteId: result.quoteId,
  });  // ← 如果这里抛异常，函数立即终止

  // 步骤 3：实时推送 —— 无 await
  this.sendCommentPatch(tableId, recordId, CommentPatchType.CreateComment, result);
  this.sendTableCommentPatch(tableId, recordId, CommentPatchType.CreateComment);

  // 步骤 4：返回结果
  return { ...result, content: result.content ? JSON.parse(result.content) : null };
}
```

**失败时的精确执行顺序**：

| 阶段 | 执行状态 | 操作 | 结果 |
|------|---------|------|------|
| 步骤 1 | ✅ 已执行 | `prismaService.comment.create()` | 评论写入 DB，事务已提交 |
| 步骤 2 | ✅ 已执行（失败） | `sendCommentNotify` 查询阶段 | 抛异常（如 `base.findUniqueOrThrow` 找不到 base） |
| 步骤 3 | ❌ 未执行 | `sendCommentPatch`、`sendTableCommentPatch` | 实时推送完全没发 |
| 步骤 4 | ❌ 未执行 | `return` | 没有返回评论数据 |

**异常传播**：
- `await sendCommentNotify()` 抛异常 → `createComment` 函数立即终止
- 异常向上冒泡到 NestJS 调用栈 → `GlobalExceptionFilter` 捕获
- `GlobalExceptionFilter.catch()` → 返回 HTTP 500 JSON 响应

**对用户可见性的影响**：

| 视角 | 现象 | 后果 |
|------|------|------|
| **发送者（前端）** | API 返回 500，UI 显示"发送失败" | 用户以为评论没发出去，可能重试导致**重复评论** |
| **发送者（刷新后）** | 评论出现在列表中 | 用户困惑——刚才明明显示失败 |
| **其他在线用户** | 没有实时推送，评论计数不变 | 不知道有新评论，也看不到评论 |
| **其他在线用户（刷新后）** | 评论出现，计数更新 | 正常看到，但延迟了 |
| **通知收件人** | 没有任何通知 | 完全不知道有人评论了 |

**特殊场景：提前 return 而非抛异常**

`sendCommentNotify` 中存在提前返回逻辑（`comment-open-api.service.ts:654`）：

```typescript
if (!baseId || !fieldId) {
  return;  // ← 正常 return，不抛异常
}
```

如果 `tableMeta.findFirst()` 或 `field.findFirst()` 返回 null（但不抛异常），`sendCommentNotify` 会正常 return：

| 阶段 | 执行状态 | 结果 |
|------|---------|------|
| 步骤 1 | ✅ | 评论已写入 DB |
| 步骤 2 | ✅ | 查询阶段提前 return，无异常 |
| 步骤 3 | ✅ | 实时推送正常发送 |
| 步骤 4 | ✅ | API 返回 200，评论数据正常返回 |
| **通知** | ❌ | 完全没发（forEach 循环被跳过） |

这种情况是**静默失败**：用户看到评论发送成功，其他用户也能看到实时更新，**但没有任何人收到通知**——这是最隐蔽的 bug。

**updateComment 也有同样的问题**：

```typescript
// comment-open-api.service.ts:401-405
this.sendCommentPatch(tableId, recordId, CommentPatchType.UpdateComment, result);
await this.sendCommentNotify(tableId, recordId, commentId, {
  quoteId: result.quoteId,
  content: result.content,
});
```

注意 `updateComment` 的顺序是**先发实时推送，再 await 查询阶段**。如果查询阶段失败：
- `sendCommentPatch` 已执行 → 其他在线用户能看到实时更新
- API 返回 500 → 用户以为更新失败
- 通知完全没发

这与 `createComment` 的顺序不同（先查询后推送），说明评论的更新和创建路径**没有统一的失败策略**。

---

## 十、关键文件索引

| 职责 | 文件路径 |
|------|----------|
| 前端评论编辑器 | `packages/sdk/src/components/comment/comment-editor/CommentEditor.tsx` |
| 前端评论列表 + 实时监听 | `packages/sdk/src/components/comment/comment-list/CommentList.tsx` |
| 前端评论 Presence 监听 hook | `packages/sdk/src/components/comment/comment-list/useCommentPatchListener.ts` |
| 前端订阅管理 | `packages/sdk/src/components/comment/CommentHeader.tsx` |
| 前端评论计数 hook | `packages/sdk/src/hooks/use-comment-count-map.ts` |
| 前端通知 Provider | `packages/sdk/src/context/notification/NotificationProvider.tsx` |
| 后端评论 Controller | `apps/nestjs-backend/src/features/comment/comment-open-api.controller.ts` |
| 后端评论 Service | `apps/nestjs-backend/src/features/comment/comment-open-api.service.ts` |
| 后端通知 Service | `apps/nestjs-backend/src/features/notification/notification.service.ts` |
| 后端权限 Guard | `apps/nestjs-backend/src/features/auth/guard/permission.guard.ts` |
| 全局异常过滤器 | `apps/nestjs-backend/src/filter/global-exception.filter.ts` |
| 邮件发送 Service | `apps/nestjs-backend/src/features/mail-sender/mail-sender.service.ts` |
| 评论内容类型定义 | `packages/openapi/src/comment/types.ts` |
| Channel 命名 | `packages/core/src/models/channel.ts` |
| 通知枚举 & Schema | `packages/core/src/models/notification/notification.enum.ts` |
| 通知 Schema | `packages/core/src/models/notification/notification.schema.ts` |
| 角色权限常量 | `packages/core/src/auth/role/constant.ts` |
| 分享链接权限 | `packages/core/src/auth/role/share.ts` |
| 模板权限 | `packages/core/src/auth/role/template.ts` |
| 权限 Action 定义 | `packages/core/src/auth/actions.ts` |
| 创建评论 OpenAPI | `packages/openapi/src/comment/create.ts` |
| 评论迁移 SQL | `packages/db-main-prisma/prisma/postgres/migrations/20240919032636_add_comment/migration.sql` |
| CommentSubscription Prisma 模型 | `packages/db-main-prisma/prisma/postgres/schema.prisma:757` |
| CommentSubscription 补主键迁移 | `packages/db-main-prisma/prisma/postgres/migrations/20250509062715_require_primary_key/migration.sql` |
| CommentSubscription 移除默认值迁移 | `packages/db-main-prisma/prisma/postgres/migrations/20250922111648_add_indexes/migration.sql` |
| ID 生成器 | `packages/core/src/utils/id-generator.ts` |

---

## 十一、设计要点

1. **入库清洗**：`filterCommentContent` 在写入 DB 前剥离 `name`/`avatar`/`url` 等展示态字段，只存引用 ID；读取时通过 `additionalContentContext` 动态回填，保证用户改名/换头像后评论展示始终最新。

2. **三路推送分离**：
   - `sendCommentPatch` → 单条记录的评论实时更新频道
   - `sendTableCommentPatch` → 整表的评论计数变更频道
   - `sendNotifyBySocket` → 用户个人通知频道
   三者互不干扰，前端按需订阅。

3. **收件人合并去重**：@提及用户、被引用评论作者、记录订阅者三路合并后去重，且始终排除评论发送者自身，避免自己给自己发通知。

4. **Presence 而非 OT**：评论的实时同步不走 ShareDB 的文档 OT 协作，而是用 Presence 做轻量广播。评论列表和计数都由前端本地状态管理，Presence 事件仅作为"补丁"增量更新本地数据，不走快照/回滚机制。

5. **通知双通道**：每条通知同时走 WebSocket（即时）和 Email（用户开启 notifyMeta.email 时异步），邮件模板复用 i18n 体系。

6. **订阅模型是记录级开关**：`comment_subscription` 表的 `(tableId, recordId)` 唯一约束使同一记录只能有一行订阅，`createdBy` 不在唯一约束中。重复订阅会触发 Prisma P2002 错误（前端通过 `subscribeStatus` 检查防止自重复订阅，但无法区分是否是自己的订阅）。`getSubscribeDetail` 返回的是记录上唯一的订阅行，无论 `createdBy` 是谁——其他用户看到"已订阅"实际是别人的订阅。`unsubscribeComment` 也不校验 `createdBy`，可以删除他人的订阅。

7. **deleteComment 权限不对称**：删除评论只需 `record|read` 而非 `record|comment`，虽然 Service 层通过 `createdBy` 限制只能删自己的评论，但权限注解与语义不一致。管理员也无法通过此接口删除他人的不当评论。

8. **updateComment 数据清洗缺失**：`updateComment` 未调用 `filterCommentContent`，可能导致冗余展示态字段入库。虽然读取时回填覆盖了此问题，但数据一致性应从写入端保证。

9. **通知分发是完全 fire-and-forget**：评论 DB 写入是强一致的，但通知分发从 `sendCommentNotify` 的 `forEach` 开始就是 fire-and-forget——`forEach` 无 await、`NotificationService.sendCommentNotify` 内部也无 await `sendCommonNotify`，形成双重无 await。所有通知 DB 写入、WebSocket 推送、邮件发送的失败都不会影响 API 返回，而是变成 unhandled Promise rejection。唯一能影响 API 返回的是 `sendCommentNotify` 的查询阶段（查收件人、查表名等），但此时评论已入库，查询失败会导致 API 返回 500 而评论已存在（用户重试可能产生重复评论）。

10. **unhandledRejection → uncaughtException 的升级链**：进程级错误传播路径设计特殊。`unhandledRejection` 监听器中不仅记日志，还会 `throw reason`，将异步 Promise 拒绝主动升级为同步未捕获异常，进而触发 `uncaughtException`。`uncaughtException` 监听器只记日志、不退出进程，违反 Node.js 最佳实践——异常发生后进程状态可能已损坏，但仍带病继续运行。

11. **创建与更新的失败策略不一致**：`createComment` 顺序是「写评论 → 查询通知 → 实时推送」，查询失败时实时推送不会发送，其他用户看不到更新；而 `updateComment` 顺序是「写评论 → 实时推送 → 查询通知」，查询失败时实时推送已经发了，其他用户能看到更新但通知不会发送。两种操作没有统一的失败处理策略，会导致用户在不同场景下看到不一致的现象。

12. **createComment 完全不具备幂等性**：函数入口第一行就调用 `generateCommentId()` 生成随机 ID，每次调用都会产生不同的 ID。这意味着如果查询阶段失败导致 API 返回 500，用户重试时会在 DB 中插入多条不同的评论——**产生重复评论是真实的 bug，不是理论风险**。

13. **comment_subscription 的三次结构演变**：从 2024 年 9 月的无主键表（只有 `table_id`/`record_id`/`created_by`/`created_time`）→ 2025 年 5 月补 `id` 主键（PostgreSQL 原生 `md5(random()::text)` 默认值）→ 2025 年 9 月移除默认值。当前 Prisma schema 使用 `@id @default(cuid())` 应用级别生成 ID。线上结构可通过 `information_schema.columns` 查询 `id` 列的 `column_default` 来判断处于哪个阶段。

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

### 6.2 订阅的唯一约束语义

`@@unique([tableId, recordId])` 意味着 **同一 (tableId, recordId) 组合只能有一条订阅记录**。这隐含了一个重要设计决策：

- 一条记录只允许**一个人**订阅（`createdBy` 记录谁订阅了这条记录的评论）
- 这**不是**一个"多用户可以同时订阅同一记录"的设计——否则唯一约束应该是 `(tableId, recordId, createdBy)`

实际上从 Prisma schema 来看，`createdBy` 不在唯一约束中，因此同一记录只能存在一条订阅行。这意味着**同一记录的评论订阅只能由最后订阅的人独占**，之前的订阅者会被覆盖。这看起来像是一个设计上的特殊点：订阅可能是"记录级开关"而非"用户-记录级关系"。

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

`createComment` 中的调用顺序：

```typescript
// 1. 写 DB —— 必须等
const result = await this.prismaService.comment.create({ ... });

// 2. 通知分发 —— 必须等
await this.sendCommentNotify(tableId, recordId, id, { ... });

// 3. 评论实时推送 —— 不等（fire-and-forget）
this.sendCommentPatch(tableId, recordId, CommentPatchType.CreateComment, result);

// 4. 表级计数推送 —— 不等（fire-and-forget）
this.sendTableCommentPatch(tableId, recordId, CommentPatchType.CreateComment);
```

| 步骤 | await | 含义 |
|------|:---:|------|
| `comment.create` | ✅ | DB 写入是主操作，必须等，失败则整体报错 |
| `sendCommentNotify` | ✅ | 通知写入 DB + 推送 + 发邮件全部完成后才返回 |
| `sendCommentPatch` | ❌ | 实时推送 fire-and-forget，失败不影响主流程 |
| `sendTableCommentPatch` | ❌ | 表级计数推送 fire-and-forget |

**`sendCommentNotify` 是 await 的**，意味着：如果通知写入 DB 失败、ShareDB 推送失败或邮件发送失败，`createComment` 整体会抛出异常——**评论创建成功但通知失败时，API 会返回 500**，但评论已经写入了 DB。

### 9.2 sendCommentNotify 内部的 await 链

```typescript
private async sendCommentNotify(...) {
  // 1. 查 quoteId 的 createdBy           ← await
  // 2. getMentionUserByContent(content)   ← 同步
  // 3. 查 tableMeta                       ← await
  // 4. 查 primary field                   ← await
  // 5. 查 baseName                        ← await
  // 6. 查 recordName                      ← await
  // 7. 查 commentSubscription             ← await
  // 8. 构造消息
  // 9. for each userId:
  //      this.notificationService.sendCommentNotify(...)  ← 同步调用（无 await）
}
```

关键点：步骤 9 中对 `subscribeUsersIds` 的遍历调用 `notificationService.sendCommentNotify` **没有 await**——这是一个**同步循环中发起多个异步操作**的模式。由于 `sendCommentNotify` 返回 `Promise<void>` 但在循环中未被 await，这些通知操作会**并发执行**，而 `sendCommentNotify` 外部的 await 只等到了循环本身（同步代码）完成，不等每个通知操作完成。

**含义**：`createComment` 中的 `await this.sendCommentNotify(...)` 实际上只等待了查询阶段（步骤 1-7），并没有等待通知真正写入 DB 或推送完成。通知操作是**半 fire-and-forget** 的。

### 9.3 sendCommonNotify 内部的可靠性

```typescript
async sendCommonNotify(...) {
  const notifyData = await this.createNotify(data);       // ① 写 DB — await
  const unreadCount = (await this.unreadCount(...));       // ② 查未读数 — await
  this.sendNotifyBySocket(toUserId, socketNotification);   // ③ WebSocket 推送 — 无 await
  if (emailEnabled) {
    this.mailSenderService.sendMail(...)                   // ④ 邮件 — 无 await
  }
}
```

| 步骤 | await | 失败影响 |
|------|:---:|---------|
| ① `notification.create` | ✅ | DB 写入失败 → `sendCommonNotify` 抛异常 → 但外层没有 await 它 |
| ② `unreadCount` | ✅ | 查询失败 → 同上 |
| ③ `sendNotifyBySocket` | ❌ | 推送失败 → 仅 console.error |
| ④ `mailSenderService.sendMail` | ❌ | 邮件失败 → catch 后 log error，返回 false |

**WebSocket 推送失败**：

```typescript
private async sendNotifyBySocket(toUserId: string, data: INotificationBuffer) {
  return new Promise((resolve) => {
    localPresence.submit(data, (error) => {
      error && this.logger.error(error);  // 只记日志
      resolve(data);                       // 无论成功失败都 resolve
    });
  });
}
```

即使 ShareDB Presence submit 失败，Promise 也会 resolve（不会 reject）。通知已经写入了 DB，用户下次刷新页面仍可从通知列表 API 获取。

**邮件发送失败**：

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

邮件发送是**最大努力交付**：失败只记日志，不重试，不回滚通知记录。

### 9.4 失败场景汇总

| 失败点 | 时机 | 影响 | 可恢复性 |
|--------|------|------|---------|
| 评论 DB 写入失败 | 主操作 | 评论不存在，API 返回错误 | 用户可重试 |
| 通知 DB 写入失败 | `sendCommonNotify` 内 | 该收件人无通知记录，但评论已创建 | 不可自动恢复，需手动补偿 |
| ShareDB 评论推送失败 | `sendCommentPatch` | 其他在线用户看不到实时更新 | 刷新页面可恢复（从 API 拉取） |
| ShareDB 通知推送失败 | `sendNotifyBySocket` | 在线用户收不到实时通知弹窗 | 刷新页面后从通知列表可恢复 |
| 邮件发送失败 | `mailSenderService.sendMail` | 收件人收不到邮件 | 不可自动恢复，无重试机制 |
| `sendCommentNotify` 的循环无 await | 通知写入阶段 | 如果第一个通知写入失败抛异常，后续通知不会发出 | 部分收件人丢失通知 |

### 9.5 可靠性评估

1. **评论写入**：强一致，失败即回滚，用户感知明确。

2. **通知持久化**：最终一致但非原子——评论写入成功后，通知写入可能部分失败。由于 `sendCommonNotify` 的调用在循环中无 await，某个收件人的通知失败不会阻断其余收件人，但也无法保证全部送达。

3. **实时推送**：尽力交付（best-effort）。ShareDB Presence 没有持久化保证，断线期间的消息不会重放。但评论和通知已写入 DB，刷新页面即可恢复。

4. **邮件通知**：尽力交付，无重试。SMTP 发送失败后仅记日志，不会重发。

5. **幂等性**：`createComment` 每次调用都生成新的 `commentId`，所以天然幂等。但 `sendCommentNotify` 中对每个 `userId` 调用 `sendCommonNotify` 也生成新的 `notificationId`，如果被多次调用（如网络超时后重试），可能导致**同一评论对同一用户产生多条通知**。

---

## 十、关键文件索引

| 职责 | 文件路径 |
|------|----------|
| 前端评论编辑器 | `packages/sdk/src/components/comment/comment-editor/CommentEditor.tsx` |
| 前端评论列表 + 实时监听 | `packages/sdk/src/components/comment/comment-list/CommentList.tsx` |
| 前端评论 Presence 监听 hook | `packages/sdk/src/components/comment/comment-list/useCommentPatchListener.ts` |
| 前端评论计数 hook | `packages/sdk/src/hooks/use-comment-count-map.ts` |
| 前端通知 Provider | `packages/sdk/src/context/notification/NotificationProvider.tsx` |
| 后端评论 Controller | `apps/nestjs-backend/src/features/comment/comment-open-api.controller.ts` |
| 后端评论 Service | `apps/nestjs-backend/src/features/comment/comment-open-api.service.ts` |
| 后端通知 Service | `apps/nestjs-backend/src/features/notification/notification.service.ts` |
| 后端权限 Guard | `apps/nestjs-backend/src/features/auth/guard/permission.guard.ts` |
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

6. **订阅模型的单用户约束**：`comment_subscription` 表的 `(tableId, recordId)` 唯一约束导致同一记录只能有一个订阅者，这是一个值得注意的设计限制——多人协作场景下，只有最后一个订阅者能收到订阅通知，其他协作者必须依赖 @提及 或被引用才能收到通知。

7. **deleteComment 权限不对称**：删除评论只需 `record|read` 而非 `record|comment`，虽然 Service 层通过 `createdBy` 限制只能删自己的评论，但权限注解与语义不一致。管理员也无法通过此接口删除他人的不当评论。

8. **updateComment 数据清洗缺失**：`updateComment` 未调用 `filterCommentContent`，可能导致冗余展示态字段入库。虽然读取时回填覆盖了此问题，但数据一致性应从写入端保证。

9. **通知可靠性为尽力交付**：评论 DB 写入是强一致的，但通知分发（DB 持久化 + WebSocket 推送 + 邮件）均为尽力交付，无重试和补偿机制。WebSocket 推送失败后刷新可恢复（通知已入 DB），但邮件发送失败不可恢复。`sendCommentNotify` 内部的循环调用无 await，存在部分通知丢失的窗口。

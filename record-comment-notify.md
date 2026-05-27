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

## 六、关键文件索引

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
| 评论内容类型定义 | `packages/openapi/src/comment/types.ts` |
| Channel 命名 | `packages/core/src/models/channel.ts` |
| 通知枚举 & Schema | `packages/core/src/models/notification/notification.enum.ts` |
| 通知 Schema | `packages/core/src/models/notification/notification.schema.ts` |
| 创建评论 OpenAPI | `packages/openapi/src/comment/create.ts` |

---

## 七、设计要点

1. **入库清洗**：`filterCommentContent` 在写入 DB 前剥离 `name`/`avatar`/`url` 等展示态字段，只存引用 ID；读取时通过 `additionalContentContext` 动态回填，保证用户改名/换头像后评论展示始终最新。

2. **三路推送分离**：
   - `sendCommentPatch` → 单条记录的评论实时更新频道
   - `sendTableCommentPatch` → 整表的评论计数变更频道
   - `sendNotifyBySocket` → 用户个人通知频道
   三者互不干扰，前端按需订阅。

3. **收件人合并去重**：@提及用户、被引用评论作者、记录订阅者三路合并后去重，且始终排除评论发送者自身，避免自己给自己发通知。

4. **Presence 而非 OT**：评论的实时同步不走 ShareDB 的文档 OT 协作，而是用 Presence 做轻量广播。评论列表和计数都由前端本地状态管理，Presence 事件仅作为"补丁"增量更新本地数据，不走快照/回滚机制。

5. **通知双通道**：每条通知同时走 WebSocket（即时）和 Email（用户开启 notifyMeta.email 时异步），邮件模板复用 i18n 体系。

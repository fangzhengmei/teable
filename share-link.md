# 共享链接访问边界实现分析

## 概述

Teable 实现了两种共享链接机制：
1. **视图共享 (View Share)** - 共享单个视图，路径前缀 `/api/share/:shareId/view/*`
2. **基库共享 (Base Share)** - 共享整个基库或基库中的节点，路径前缀 `/api/share/:shareId/base/*`

两种机制在 token 签发、权限收敛和匿名访问衔接上遵循相似的设计模式。

---

## 一、共享启用与 shareId 生成

### 1.1 视图共享启用

**代码路径**: `apps/nestjs-backend/src/features/view/open-api/view-open-api.service.ts:837-888`

```typescript
async enableShare(tableId: string, viewId: string) {
  const newShareId = generateShareId();
  const enableShareOp = ViewOpBuilder.editor.setViewProperty.build({
    key: 'enableShare',
    newValue: true,
  });
  const setShareIdOp = ViewOpBuilder.editor.setViewProperty.build({
    key: 'shareId',
    newValue: newShareId,
  });
  // ...
}
```

**关键逻辑**:
- 调用 `generateShareId()` 生成唯一的 shareId（`shr` 前缀）
- 通过 OT 操作设置 `enableShare: true` 和 `shareId`
- 自动初始化默认的 `shareMeta`（如表单视图的提交权限）

### 1.2 基库共享创建

**代码路径**: `apps/nestjs-backend/src/features/base-share/base-share.service.ts:78-113`

```typescript
async createBaseShare(baseId: string, data: ICreateBaseShareRo) {
  const share = await this.prismaService.baseShare.create({
    data: {
      baseId,
      shareId: generateShareId(),
      nodeId: data.nodeId ?? null,
      createdBy: this.cls.get('user.id'),
    },
  });
  // ...
}
```

**关键逻辑**:
- 支持整个基库共享（`nodeId: null`）或节点级共享（`nodeId: bno...`）
- 可配置 `allowEdit`、`allowSave`、`allowCopy`、`password` 等权限标志
- `allowEdit` 和 `allowSave` 互斥

---

## 二、Token 签发机制

### 2.1 视图共享 Token 签发

**代码路径**: 
- `apps/nestjs-backend/src/features/share/share-auth.service.ts:67-69`
- `apps/nestjs-backend/src/features/share/share.controller.ts:79-91`

```typescript
// 签发 Token
async authToken(jwtShareInfo: IJwtShareInfo) {
  return await this.jwtService.signAsync(jwtShareInfo);
}

// 认证接口
@Post('/:shareId/view/auth')
async auth(@Request() req: any, @Res({ passthrough: true }) res: Response) {
  const shareId = req.shareId;
  const password = req.password;
  const token = await this.shareAuthService.authToken({ shareId, password });
  res.cookie(shareId, token, {
    httpOnly: true,
    maxAge: 1000 * 60 * 60 * 24 * 7, // 7天
  });
  return { token };
}
```

**Token 结构**:
```typescript
interface IJwtShareInfo {
  shareId: string;
  password: string;
}
```

**关键逻辑**:
- Token 存储在 HttpOnly Cookie 中，Cookie 名称为 shareId
- 有效期 7 天
- 包含 `shareId` 和明文 `password`（注意：密码明文存储在 JWT 中）

### 2.2 基库共享 Token 签发

**代码路径**: `apps/nestjs-backend/src/features/base-share/base-share-auth.service.ts:61-63`

与视图共享完全相同的机制，使用相同的 JWT 结构和 Cookie 存储方式。

### 2.3 Token 验证策略

**代码路径**: `apps/nestjs-backend/src/features/share/strategies/jwt.strategy.ts:26-39`

```typescript
public static fromAuthCookieAsToken(req: Request): string | null {
  const shareId = req.params.shareId || (req.headers['tea-share-id'] as string);
  const cookieObj = cookie.parse(req.headers.cookie ?? '');
  return cookieObj?.[shareId] ?? null;
}

async validate(payload: IJwtShareInfo) {
  const { shareId, password } = payload;
  const authShareId = await this.shareAuthService.authShareView(shareId, password);
  if (!authShareId) {
    throw new UnauthorizedException();
  }
  return authShareId;
}
```

**关键逻辑**:
- 从 Cookie 中提取与 shareId 同名的 Token
- 验证时对比 JWT 中的密码与数据库中存储的密码
- 管理员修改密码后，旧 Token 自动失效

---

## 三、权限收敛逻辑

### 3.1 视图共享权限矩阵

**代码路径**: `packages/core/src/auth/role/share.ts:6-22`

```typescript
export const shareViewPermissions: Record<ShareViewAction, boolean> = {
  'view|create': false,
  'view|delete': false,
  'view|read': true,
  'view|update': false,
  'view|share': false,
  'field|create': false,
  'field|delete': false,
  'field|read': true,
  'field|update': false,
  'record|create': false,
  'record|comment': false,
  'record|delete': false,
  'record|read': true,
  'record|update': false,
  'record|copy': false,
};
```

**权限特点**:
- 默认仅开放 `read` 权限（view、field、record）
- 可通过 `shareMeta` 配置扩展权限：
  - `allowCopy: true` → 允许 `record|copy`
  - `submit.allow: true` → 允许表单提交
  - `includeRecords: true` → 允许查询记录
  - `includeHiddenField: true` → 允许访问隐藏字段

### 3.2 基库共享权限收敛

**代码路径**: `apps/nestjs-backend/src/features/auth/permission.service.ts:602-652`

```typescript
const SHARE_EXCLUDED_PERMISSIONS = new Set<Action>([
  'view|share',
  'space|invite_email',
  'base|invite_email',
  'user|email_read',
  'user|integrations',
]);

async getBaseSharePermissions(shareId: string, resourceId: string) {
  // ... 节点归属验证 ...
  
  if (baseShare.allowEdit && !this.isAnonymous()) {
    return getPermissions(Role.Editor).filter((p) => !SHARE_EXCLUDED_PERMISSIONS.has(p));
  }
  
  const permissions = [...TemplatePermissions];
  if (baseShare.allowCopy) {
    permissions.push('record|copy');
  }
  return permissions;
}
```

**权限分层**:
1. **无密码 + 匿名用户** → `TemplatePermissions`（只读）
2. **无密码 + 登录用户** → `TemplatePermissions`（只读）
3. **有密码 + 匿名用户** → 需密码验证，`TemplatePermissions`
4. **有密码 + 登录用户** → 需密码验证，`TemplatePermissions`
5. **`allowEdit: true` + 登录用户** → `Editor` 权限（排除敏感操作）

### 3.3 节点级共享的边界控制

**代码路径**: `apps/nestjs-backend/src/features/auth/permission.service.ts:658-715`

```typescript
private async checkResourceBelongsToShare(
  resourceId: string,
  baseId: string,
  nodeId: string
): Promise<boolean> {
  const prefix = resourceId.substring(0, 3);
  switch (prefix) {
    case IdPrefix.Base:
      return resourceId === baseId;
    case IdPrefix.Table:
      return this.checkTableBelongsToShare(resourceId, baseId, nodeId);
    case IdPrefix.View:
      return this.checkViewBelongsToShare(resourceId, baseId, nodeId);
    case IdPrefix.Field:
      return this.checkFieldBelongsToShare(resourceId, baseId, nodeId);
    case IdPrefix.App:
      return this.checkAppBelongsToShare(resourceId, baseId, nodeId);
    default:
      return false;
  }
}
```

**边界控制逻辑**:
- 收集共享节点及其所有后代节点 ID
- 验证访问的资源是否属于允许的节点子树
- **特殊处理**：关联字段的外部表自动获得访问权限（`isTableLinkedFromSharedNode`）

---

## 四、匿名访问衔接

### 4.1 匿名用户定义

**代码路径**: `packages/core/src/auth/anonymous.ts:1-10`

```typescript
export const ANONYMOUS_USER_ID = 'anonymous';

export const isAnonymous = (userId: string) => userId === ANONYMOUS_USER_ID;

export const ANONYMOUS_USER = {
  id: ANONYMOUS_USER_ID,
  name: 'Anonymous',
  email: 'anonymous@system.teable.ai',
};
```

### 4.2 匿名认证策略

**代码路径**: `apps/nestjs-backend/src/features/auth/strategies/anonymous/anonymous.strategy.ts:14-18`

```typescript
async validate() {
  this.cls.set('user', ANONYMOUS_USER);
  return ANONYMOUS_USER;
}
```

### 4.3 视图共享 Guard 中的匿名衔接

**代码路径**: `apps/nestjs-backend/src/features/share/guard/auth.guard.ts:27-70`

```typescript
async validate(context: ExecutionContext, shareId: string) {
  const req = context.switchToHttp().getRequest();
  const shareInfo = await this.shareAuthService.getShareViewInfo(shareId);
  
  try {
    req.shareInfo = shareInfo;
    
    // 表单提交需要登录
    const isShareSubmit = this.reflector.getAllAndOverride<boolean>(IS_SHARE_SUBMIT_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    const submit = shareInfo.shareMeta?.submit;
    if (isShareSubmit && submit?.allow && submit?.requireLogin) {
      return this.authGuard.validate(context); // 走正常登录验证
    }
    
    // 设置匿名用户
    this.cls.set('user', {
      id: ANONYMOUS_USER_ID,
      name: ANONYMOUS_USER_ID,
      email: '',
    });
    
    // 有密码则走 JWT 验证
    if (shareInfo.view?.shareMeta?.password) {
      return (await super.canActivate(context)) as boolean;
    }
    return true;
  } catch (err) {
    throw new CustomHttpException('Unauthorized', HttpErrorCode.UNAUTHORIZED_SHARE);
  }
}
```

### 4.4 基库共享 Guard 中的匿名衔接

**代码路径**: `apps/nestjs-backend/src/features/base-share/guard/base-share-auth.guard.ts:20-52`

```typescript
async validate(context: ExecutionContext, shareId: string) {
  const req = context.switchToHttp().getRequest();
  
  try {
    const shareInfo = await this.baseShareAuthService.getBaseShareInfo(shareId);
    req.baseShareInfo = shareInfo;
    
    // 保留已登录用户身份（用于复制操作）
    const currentUserId = this.cls.get('user.id');
    if (!currentUserId) {
      this.cls.set('user', {
        id: ANONYMOUS_USER_ID,
        name: ANONYMOUS_USER_ID,
        email: '',
      });
    }
    
    // 有密码则走 JWT 验证
    const hasPassword = await this.baseShareAuthService.hasPassword(shareId);
    if (hasPassword) {
      return (await super.canActivate(context)) as boolean;
    }
    return true;
  } catch (err) {
    // ...
  }
}
```

**关键区别**:
- 基库共享 Guard 会优先保留已登录用户的身份（用于 `allowEdit` 场景）
- 视图共享 Guard 直接设置匿名用户，除非是需要登录的表单提交

---

## 五、完整访问流程

### 5.1 视图共享访问流程

```
用户访问 /share/[shareId]/view
        ↓
SSR getServerSideProps 调用 /api/share/[shareId]/view
        ↓
ShareAuthGuard.canActivate()
        ↓
├─ 1. getShareViewInfo(shareId) → 验证 shareId 有效性
│    └─ 查询 view 表，检查 enableShare=true, deletedTime=null
        ↓
├─ 2. 检查是否为需要登录的表单提交
│    ├─ 是 → authGuard.validate() → 正常登录流程
│    └─ 否 → 设置 CLS user = ANONYMOUS_USER
        ↓
├─ 3. 检查是否设置了密码
│    ├─ 是 → JwtStrategy 验证 Cookie 中的 Token
│    │    └─ 对比 JWT 中的 password 与数据库密码
│    └─ 否 → 直接放行
        ↓
└─ 4. 执行业务逻辑（查询记录、聚合等）
     └─ 所有操作受 shareViewPermissions 限制
```

### 5.2 基库共享访问流程

```
用户访问 /share/[shareId]/base
        ↓
BaseShareAuthGuard.canActivate()
        ↓
├─ 1. getBaseShareInfo(shareId) → 验证 shareId 有效性
│    └─ 查询 base_share 表，检查 enabled=true
        ↓
├─ 2. 保留或设置用户身份
│    ├─ 已登录 → 保留用户身份（用于 allowEdit）
│    └─ 未登录 → 设置 CLS user = ANONYMOUS_USER
        ↓
├─ 3. 检查是否设置了密码
│    ├─ 是 → JwtStrategy 验证 Cookie 中的 Token
│    └─ 否 → 直接放行
        ↓
└─ 4. getBaseSharePermissions() → 权限收敛
     ├─ 检查资源是否属于共享节点子树
     ├─ allowEdit && 已登录 → Editor 权限（排除敏感操作）
     └─ 其他 → TemplatePermissions + allowCopy
```

---

## 六、安全边界关键点

### 6.1 密码验证
- 密码明文存储在数据库（`view.shareMeta.password`、`base_share.password`）
- JWT Token 中包含明文密码
- 密码变更后旧 Token 自动失效（验证时实时对比）

### 6.2 权限排除
```typescript
const SHARE_EXCLUDED_PERMISSIONS = new Set<Action>([
  'view|share',           // 禁止二次共享
  'space|invite_email',   // 禁止邀请用户
  'base|invite_email',    // 禁止邀请用户
  'user|email_read',      // 禁止读取邮箱
  'user|integrations',    // 禁止访问集成
]);
```

### 6.3 节点边界
- 节点级共享仅允许访问共享节点及其后代
- 关联字段的外部表作为例外自动放行
- 使用 CLS 缓存节点树避免重复查询

### 6.4  Cookie 安全
- HttpOnly Cookie，防止 XSS 窃取
- Cookie 名称为 shareId，支持同一浏览器多标签访问不同共享

---

## 八、三种越权场景错误处理

### 8.1 错误码定义

**代码路径**: `packages/core/src/errors/http/http-response.types.ts:15-66`

```typescript
export enum HttpErrorCode {
  VALIDATION_ERROR = 'validation_error',       // 400 - 参数验证错误
  UNAUTHORIZED = 'unauthorized',           // 401 - 未授权
  UNAUTHORIZED_SHARE = 'unauthorized_share',   // 401 - 共享链接未授权
  RESTRICTED_RESOURCE = 'restricted_resource', // 403 - 资源访问受限
  NOT_FOUND = 'not_found',                    // 404 - 资源不存在
}
```

---

### 8.2 场景一：shareId 无效

**拦截位置与错误码**：

| 场景 | 文件路径 | 错误码 | HTTP 状态 |
|-----|---------|---------|-----------|
| 视图共享 - shareId 不存在/未启用/已删除 | `share-auth.service.ts:76` | `VALIDATION_ERROR` | 400 |
| 基库共享 - shareId 不存在/未启用 | `base-share-auth.service.ts:71` | `NOT_FOUND` | 404 |
| 基库共享 - 更新/删除时不存在 | `base-share.service.ts:167,208,231,253` | `NOT_FOUND` | 404 |
| 基库共享 Guard - NOT_FOUND 特殊处理 | `base-share-auth.guard.ts:46-51` | `NOT_FOUND` | 404 |
| **Header 路径 - shareId 格式错误** | `permission.service.ts:995-996` | 跳过共享检查（返回 undefined） | - |
| **Header 路径 - shareId 不存在/未启用** | `permission.service.ts:604-608` | `RESTRICTED_RESOURCE` | 403 |

**视图共享拦截逻辑** (`share-auth.service.ts:70-79`)：
```typescript
const view = await this.prismaService.view.findFirst({
  where: { shareId, enableShare: true, deletedTime: null },
});
if (!view) {
  throw new CustomHttpException('Share view not found', HttpErrorCode.VALIDATION_ERROR, {
    localization: { i18nKey: 'httpErrors.shareAuth.shareViewNotFound' },
  });
}
```

**基库共享拦截逻辑** (`base-share-auth.service.ts:65-74`)：
```typescript
const share = await this.prismaService.baseShare.findFirst({
  where: { shareId, enabled: true },
});
if (!share || !share.enabled) {
  throw new CustomHttpException('Base share not found', HttpErrorCode.NOT_FOUND, {
    localization: { i18nKey: 'httpErrors.baseShare.notFound' },
  });
}
```

**关键区别**：
- 视图共享使用 `VALIDATION_ERROR` (400)，基库共享使用 `NOT_FOUND` (404)
- 基库共享 Guard 会特殊处理 `NOT_FOUND` 错误，直接抛出而不转换为 `UNAUTHORIZED_SHARE`

---

### 8.3 场景二：密码校验失败

**拦截位置与错误码**：

| 场景 | 文件路径 | 错误码 | HTTP 状态 |
|-----|---------|---------|-----------|
| 登录提交密码 - 视图 | `share-auth-local.guard.ts:19` | `VALIDATION_ERROR` | 400 |
| 登录提交密码 - 基库 | `base-share-auth-local.guard.ts:19` | `VALIDATION_ERROR` | 400 |
| JWT Token 验证失败 | `jwt.strategy.ts:126` | `UNAUTHORIZED` (转换为 `UNAUTHORIZED_SHARE`) | 401 |
| Cookie Token 缺失/无效 - PermissionGuard | `permission.guard.ts:180,184` | `UNAUTHORIZED_SHARE` | 401 |
| 视图共享 Guard 捕获异常 | `share-auth.guard.ts:68` | `UNAUTHORIZED_SHARE` | 401 |
| 基库共享 Guard 捕获异常 | `base-share-auth.guard.ts:50` | `UNAUTHORIZED_SHARE` | 401 |
| 视图共享 - 未启用密码却调用 auth 接口 | `share-auth.service.ts:54-62` | `VALIDATION_ERROR` | 400 |
| 基库共享 - 未启用密码却调用 auth 接口 | `base-share-auth.service.ts:47-56` | `VALIDATION_ERROR` | 400 |

**登录时密码校验** (`share-auth-local.guard.ts:11-26`)：
```typescript
async canActivate(context: ExecutionContext) {
  const req = context.switchToHttp().getRequest();
  const shareId = req.params.shareId;
  const password = req.body.password;
  const authShareId = await this.shareAuthService.authShareView(shareId, password);
  if (!authShareId) {
    throw new CustomHttpException('Incorrect password.', HttpErrorCode.VALIDATION_ERROR, {
      localization: { i18nKey: 'httpErrors.share.incorrectPassword' },
    });
  }
  return true;
}
```

**JWT Token 验证** (`jwt.strategy.ts:122-129`)：
```typescript
async validate(payload: IJwtShareInfo) {
  const { shareId, password } = payload;
  const authShareId = await this.shareAuthService.authShareView(shareId, password);
  if (!authShareId) {
    throw new UnauthorizedException();
  }
  return authShareId;
}
```

**PermissionGuard 二次验证** (`permission.guard.ts:171-186`)：
```typescript
private async ensureBaseShareAuth(context: ExecutionContext, shareId: string) {
  const requirePassword = await this.permissionService.baseShareRequiresPassword(shareId);
  if (!requirePassword) return;
  const cookies = cookie.parse(req.headers.cookie ?? '');
  const token = cookies[shareId];
  if (!token) {
    throw new CustomHttpException('Unauthorized', HttpErrorCode.UNAUTHORIZED_SHARE);
  }
  const valid = await this.permissionService.validateBaseSharePasswordToken(shareId, token);
  if (!valid) {
    throw new CustomHttpException('Unauthorized', HttpErrorCode.UNAUTHORIZED_SHARE);
  }
}
```

**密码 Token 验证逻辑** (`permission.service.ts:581-599`)：
```typescript
async validateBaseSharePasswordToken(shareId: string, token: string) {
  const payload = await this.jwtService.verifyAsync<{ shareId: string; password: string }>(token);
  if (payload.shareId !== shareId) return false;
  const baseShare = await this.prismaService.baseShare.findFirst({
    where: { shareId, enabled: true },
    select: { password: true },
  });
  if (!baseShare?.password) return false;
  return payload.password === baseShare.password;
}
```

---

### 8.4 场景三：共享节点越权

**拦截位置与错误码**：

| 场景 | 文件路径 | 错误码 | HTTP 状态 |
|-----|---------|---------|-----------|
| 访问资源不属于共享节点 | `permission.service.ts:629-632` | `RESTRICTED_RESOURCE` | 403 |
| 权限不足（操作超出共享权限范围） | `permission.service.ts:971-979` | `RESTRICTED_RESOURCE` | 403 |
| PermissionGuard 资源 ID 不存在 | `permission.guard.ts:138` | `RESTRICTED_RESOURCE` | 403 |
| 复制权限不足（allowSave=false） | `base-share-open.controller.ts:166` | `RESTRICTED_RESOURCE` | 403 |
| 视图 Socket 越权访问其他视图 | `share-socket.service.ts:46` | `RESTRICTED_RESOURCE` | 403 |
| 视图 Socket 字段越权 | `share-socket.service.ts:93` | `RESTRICTED_RESOURCE` | 403 |
| 视图 Socket 记录越权 | `share-socket.service.ts:151,163` | `RESTRICTED_RESOURCE` | 403 |

**节点归属检查** (`permission.service.ts:620-633`)：
```typescript
const resourceBelongsToShare = await this.checkResourceBelongsToShare(
  resourceId, baseId, nodeId
);
if (!resourceBelongsToShare) {
  this.logger.warn(
    `[BaseShare] Resource ${resourceId} is not accessible via share ${shareId}`
  );
  throw new CustomHttpException(
    `Resource ${resourceId} is not accessible via share ${shareId}`,
    HttpErrorCode.RESTRICTED_RESOURCE
  );
}
```

**操作权限验证** (`permission.service.ts:966-980`)：
```typescript
async validBaseSharePermissions(shareId: string, resourceId: string, permissions: Action[]) {
  const sharePermissions = await this.getBaseSharePermissions(shareId, resourceId);
  if (permissions.every((permission) => sharePermissions.includes(permission))) {
    return sharePermissions;
  }
  throw new CustomHttpException(
    `Base share access denied, not allowed to operate ${permissions.join(', ')} on ${resourceId}`,
    HttpErrorCode.RESTRICTED_RESOURCE,
    { localization: { i18nKey: notAllowedOperationI18nKey } }
  );
}
```

**视图 Socket 越权检查** (`share-socket.service.ts:40-49`)：
```typescript
if (ids.length > 1 || ids[0] !== view.id) {
  throw new CustomHttpException(
    'View permission not allowed: read',
    HttpErrorCode.RESTRICTED_RESOURCE,
    { localization: { i18nKey: 'httpErrors.shareSocket.viewPermissionNotAllowed' } }
  );
}
```

---

### 8.5 场景四：Base Share Header 链路完整追踪

**触发条件**：请求头携带 `X-Tea-Base-Share: shrxxx`，但 shareId 无效。

**完整拦截链路**：

```
请求进入 PermissionGuard.canActivate()
        ↓
permissionCheckWithPublicFallback()
        ↓
1. getBaseShareHeader(req) → 提取 X-Tea-Base-Share header
   代码路径: auth/utils.ts:30-34
        ↓
2. permissionCheckWithPublicFallback() 步骤 2
   if (baseShareHeader) → 进入 tryBaseSharePermissionCheck()
   代码路径: permission.guard.ts:430-433
        ↓
3. tryBaseSharePermissionCheck()
   代码路径: permission.guard.ts:292-320
   ├─ 3.1 getBaseShareIdByHeader(baseShareHeader)
   │    代码路径: permission.service.ts:994-999
   │    ├─ if (!shareHeader || !shareHeader.startsWith('shr'))
   │    └─ return null → 跳过共享检查，走正常权限流程
   ├─ 3.2 检查 @Permissions() 装饰器
   │    └─ 无权限装饰器 → return undefined → 跳过
   ├─ 3.3 检查资源 ID
   │    └─ 无资源 ID 或 space 级别 → return undefined → 跳过
   └─ 3.4 调用 baseSharePermissionCheck(context, shareId)
        ↓
4. baseSharePermissionCheck()
   代码路径: permission.guard.ts:132-169
   ├─ 4.1 ensureBaseShareAuth(context, shareId)
   │    代码路径: permission.guard.ts:171-186
   │    ├─ baseShareRequiresPassword(shareId)
   │    │  代码路径: permission.service.ts:573-579
   │    │  └─ 查询 base_share 表，shareId 不存在则返回 false
   │    └─ 无密码 → 跳过验证，继续执行
   ├─ 4.2 获取 resourceId
   ├─ 4.3 validBaseSharePermissions(shareId, resourceId, permissions)
   │    代码路径: permission.service.ts:966-980
   │    └─ 调用 getBaseSharePermissions(shareId, resourceId)
   │        ↓
5. getBaseSharePermissions()
   代码路径: permission.service.ts:602-652
   ├─ 5.1 getBaseShareInfo(shareId)
   │    代码路径: permission.service.ts:563-571
   │    ├─ 查询 base_share 表: where: { shareId, enabled: true }
   │    └─ shareId 不存在/未启用 → return null
   ├─ 5.2 if (!baseShare) → 抛出异常
   │    代码路径: permission.service.ts:604-608
   │    └─ throw new CustomHttpException(
   │         `Base share ${shareId} is not found`,
   │         HttpErrorCode.RESTRICTED_RESOURCE  // 403
   │       )
   └─ 后续节点归属检查（不会执行到，因为已抛出异常）
```

**各阶段拦截详情**：

**阶段 1：Header 提取** (`auth/utils.ts:30-34`)
```typescript
export const getBaseShareHeader = (request: Request): string | undefined => {
  const baseShareHeader =
    request.headers[BASE_SHARE_ID_HEADER.toLowerCase()] || request.headers[BASE_SHARE_ID_HEADER];
  return typeof baseShareHeader === 'string' ? baseShareHeader : undefined;
};
```
- 不区分大小写匹配 header 名称
- Header 值必须是字符串，否则返回 `undefined`
- 返回 `undefined` 则跳过整个共享检查流程

**阶段 2：shareId 格式验证** (`permission.service.ts:994-999`)
```typescript
getBaseShareIdByHeader(shareHeader: string): string | null {
  if (!shareHeader || !shareHeader.startsWith('shr')) {
    return null;
  }
  return shareHeader;
}
```
- 检查 header 值是否以 `shr` 开头
- 格式错误返回 `null` → `tryBaseSharePermissionCheck` 返回 `undefined` → 跳过共享检查
- **无错误码**，静默降级为正常权限检查

**阶段 3：shareId 有效性验证** (`permission.service.ts:563-571`)
```typescript
async getBaseShareInfo(shareId: string) {
  const baseShare = await this.prismaService.baseShare.findFirst({
    where: { shareId, enabled: true },
  });
  if (!baseShare) {
    return null;
  }
  return baseShare;
}
```
- 查询条件：`shareId` 匹配 AND `enabled: true`
- 注意：**不检查删除时间**（base_share 表无 deletedTime 字段）
- shareId 不存在或已禁用 → 返回 `null`

**阶段 4：最终拦截** (`permission.service.ts:602-608`)
```typescript
async getBaseSharePermissions(shareId: string, resourceId: string) {
  const baseShare = await this.getBaseShareInfo(shareId);
  if (!baseShare) {
    throw new CustomHttpException(
      `Base share ${shareId} is not found`,
      HttpErrorCode.RESTRICTED_RESOURCE  // 403
    );
  }
  // ...
}
```
- **错误码**：`RESTRICTED_RESOURCE` (403)
- **关键差异**：通过 Header 路径访问时，shareId 无效返回 403，而非 /api/share/* 路径的 404
- 表明系统"不承认"该共享链接存在，而非"资源不存在"

---

### 8.6 未启用密码却调用 auth 接口的真实实现

**真实实现位置**：

| 共享类型 | 实现文件 | 方法名 | 关键行 |
|---------|---------|--------|-------|
| 视图共享 | `apps/nestjs-backend/src/features/share/share-auth.service.ts` | `authShareView()` | 43-65 |
| 基库共享 | `apps/nestjs-backend/src/features/base-share/base-share-auth.service.ts` | `authBaseShare()` | 36-59 |

**视图共享实现** (`share-auth.service.ts:43-65`)：
```typescript
async authShareView(shareId: string, pass: string): Promise<string | null> {
  const view = await this.prismaService.view.findFirst({
    where: { shareId, enableShare: true, deletedTime: null },
    select: { shareId: true, shareMeta: true },
  });
  if (!view) {
    return null;  // shareId 无效，返回 null → LocalGuard 抛出密码错误
  }
  const shareMeta = view.shareMeta ? JSON.parse(view.shareMeta) : undefined;
  const password = shareMeta?.password;
  if (!password) {
    throw new CustomHttpException(
      'Password restriction is not enabled',
      HttpErrorCode.VALIDATION_ERROR,  // 400
      { localization: { i18nKey: 'httpErrors.shareAuth.passwordRestrictionNotEnabled' } }
    );
  }
  return pass === password ? shareId : null;
}
```

**基库共享实现** (`base-share-auth.service.ts:36-59`)：
```typescript
async authBaseShare(shareId: string, pass: string): Promise<string | null> {
  const share = await this.prismaService.baseShare.findUnique({
    where: { shareId },
    select: { shareId: true, password: true, enabled: true },
  });
  if (!share || !share.enabled) {
    return null;  // shareId 无效，返回 null → LocalGuard 抛出密码错误
  }
  const password = share.password;
  if (!password) {
    throw new CustomHttpException(
      'Password restriction is not enabled',
      HttpErrorCode.VALIDATION_ERROR,  // 400
      { localization: { i18nKey: 'httpErrors.shareAuth.passwordRestrictionNotEnabled' } }
    );
  }
  return pass === password ? shareId : null;
}
```

**执行流程**：
1. 用户调用 `POST /api/share/:shareId/view/auth` 或 `/base/auth`
2. `ShareAuthLocalGuard.canActivate()` 调用 `authShareView()` 或 `authBaseShare()`
3. 先验证 shareId 是否有效：
   - 无效 → 返回 `null` → LocalGuard 抛出 `Incorrect password.` (VALIDATION_ERROR, 400)
   - 有效 → 继续检查密码是否启用
4. 密码未启用 → 抛出 `Password restriction is not enabled` (VALIDATION_ERROR, 400)
5. 密码已启用 → 对比密码，正确返回 shareId，错误返回 null

**之前的错误映射修正**：
- ❌ 原文档：`share-auth.service.ts:56` 对应基库共享
- ✅ 修正后：
  - 视图共享：`share-auth.service.ts:54-62`（`authShareView()` 中 `if (!password)` 分支）
  - 基库共享：`base-share-auth.service.ts:47-56`（`authBaseShare()` 中 `if (!password)` 分支）

---

## 十一、PermissionGuard 与基础共享权限的关联

### 11.1 关联架构

PermissionGuard 是全局权限守卫，它通过两条路径与共享权限关联：

```
                    +-------------------+
                    |  请求进入      |
                    +---------+---------+
                              |
                              v
                    +---------+---------+
                    | PermissionGuard     |
                    | canActivate()   |
                    +---------+---------+
                              |
            +-----------------+-----------------+
            |                                   |
            v                                   v
+-----------+-----------+           +-----------+-----------+
| 路径匹配 /api/share/*  |           | 其他路径（通过 Header） |
| BaseShareAuthGuard       |           | X-Tea-Base-Share     |
| + ShareAuthGuard       |           | getBaseShareHeader()   |
+-----------+-----------+           +-----------+-----------+
            |                                   |
            v                                   v
| 认证层（匿名/密码验证）               | tryBaseSharePermissionCheck() |
            |                                   |
            v                                   v
| PermissionGuard.baseSharePermissionCheck() |
                        |
                        v
              +-----------+-----------+
              | validBaseSharePermissions() |
              | checkResourceBelongsToShare()   |
              | getBaseSharePermissions()  |
              +-------------------------------+
```

### 11.2 共享路径的关联

**代码路径**: `base-share-open.controller.ts:150-154`

```typescript
@HttpCode(200)
@UseGuards(BaseShareAuthGuard, PermissionGuard)
@Permissions('base|create')
@ResourceMeta('spaceId', 'body')
@Post('/:shareId/base/copy')
async copyBaseShare(...) { ... }
```

**执行顺序**：
1. **BaseShareAuthGuard** (`base-share-auth.guard.ts:20-52`)：
   - 验证 shareId 有效性
   - 设置匿名用户或保留登录用户身份
   - 有密码则走 JWT 验证
   - 捕获异常并转换错误码

2. **PermissionGuard** (`permission.guard.ts:132-169`)：
   ```typescript
   protected async baseSharePermissionCheck(context: ExecutionContext, shareId: string) {
     await this.ensureBaseShareAuth(context, shareId);  // 二次密码验证
     const resourceId = ...;
     const permissions = this.reflector.get(PERMISSIONS_KEY, ...);
     const ownPermissions = await this.permissionService.validBaseSharePermissions(
       shareId, resourceId, permissions
     );
     this.cls.set('permissions', ownPermissions);
     return true;
   }
   ```

### 11.3 非共享路径的关联

**通过 Header 关联**: `permission.guard.ts:410-447`

```typescript
protected async permissionCheckWithPublicFallback(...) {
  // 1. RESOURCE-level: exclusively use resource-specific auth
  if (allowAnonymousType === AllowAnonymousType.RESOURCE) {
    const result = await this.resolveResourcePermission(context, baseShareHeader, templateHeader);
    if (result !== undefined) return result;
  }

  // 2. Share link — permissions are bounded by the link
  if (baseShareHeader) {
    const result = await this.tryBaseSharePermissionCheck(context, baseShareHeader);
    if (result !== undefined) return result;
  }
  // ...
}
```

**Header 提取** (`auth/utils.ts:30-34`)：
```typescript
export const getBaseShareHeader = (request: Request): string | undefined => {
  const baseShareHeader =
    request.headers[BASE_SHARE_ID_HEADER.toLowerCase()] || request.headers[BASE_SHARE_ID_HEADER];
  return typeof baseShareHeader === 'string' ? baseShareHeader : undefined;
};
```

**ShareId 解析** (`permission.service.ts:994-999`)：
```typescript
getBaseShareIdByHeader(shareHeader: string): string | null {
  if (!shareHeader || !shareHeader.startsWith('shr')) {
    return null;
  }
  return shareHeader;
}
```

### 11.4 关键关联点

1. **双重密码验证**：
   - BaseShareAuthGuard 验证一次
   - PermissionGuard.ensureBaseShareAuth 二次验证
   - 防止绕过 Guard 直接访问接口

2. **CLS 权限传递**：
   - `this.cls.set('permissions', ownPermissions)`
   - 下游服务通过 CLS 获取共享权限

3. **共享身份标记**：
   - `this.cls.set('baseShare', { baseId, nodeId })`
   - 标记当前处于共享访问上下文

4. **权限天花板**：
   - 共享权限是所有用户的权限上限
   - 即使是管理员，通过共享链接访问也受共享权限限制
   - 注释说明：`Share link check — when share header is present, share permissions are the ceiling for ALL users`

### 11.5 跳过共享检查的条件

**代码路径**: `permission.guard.ts:304-318`

```typescript
// Skip share path for endpoints without @Permissions
const permissions = this.reflector.get(PERMISSIONS_KEY, ...);
if (!permissions?.length) {
  return undefined;
}
// Skip share check when the target resource is outside the share scope
const resourceId = this.getResourceId(context) || this.defaultResourceId(context);
if (!resourceId || resourceId.startsWith(IdPrefix.Space)) {
  return undefined;
}
```

**跳过场景**：
1. 接口没有 `@Permissions()` 装饰器（如 `/user/me`）
2. 资源是 space 级别的接口
3. 资源 ID 不存在

---

## 十二、核心代码模块索引

| 功能模块 | 文件路径 | 关键行 |
|---------|---------|-------|
| 视图共享启用 | `apps/nestjs-backend/src/features/view/open-api/view-open-api.service.ts` | 837-888 |
| 基库共享创建 | `apps/nestjs-backend/src/features/base-share/base-share.service.ts` | 78-113 |
| 视图 Token 签发 | `apps/nestjs-backend/src/features/share/share-auth.service.ts` | 67-69 |
| 基库 Token 签发 | `apps/nestjs-backend/src/features/base-share/base-share-auth.service.ts` | 61-63 |
| JWT 验证策略 | `apps/nestjs-backend/src/features/share/strategies/jwt.strategy.ts` | 26-39 |
| 视图共享 Guard | `apps/nestjs-backend/src/features/share/guard/auth.guard.ts` | 27-70 |
| 基库共享 Guard | `apps/nestjs-backend/src/features/base-share/guard/base-share-auth.guard.ts` | 20-52 |
| 视图权限矩阵 | `packages/core/src/auth/role/share.ts` | 6-22 |
| 基库权限收敛 | `apps/nestjs-backend/src/features/auth/permission.service.ts` | 602-652 |
| 节点边界检查 | `apps/nestjs-backend/src/features/auth/permission.service.ts` | 658-715 |
| 匿名用户定义 | `packages/core/src/auth/anonymous.ts` | 1-10 |
| 匿名认证策略 | `apps/nestjs-backend/src/features/auth/strategies/anonymous/anonymous.strategy.ts` | 14-18 |
| 错误码定义 | `packages/core/src/errors/http/http-response.types.ts` | 15-66 |
| 视图密码登录 Guard | `apps/nestjs-backend/src/features/share/guard/share-auth-local.guard.ts` | 11-26 |
| 基库密码登录 Guard | `apps/nestjs-backend/src/features/base-share/guard/base-share-auth-local.guard.ts` | 11-26 |
| PermissionGuard | `apps/nestjs-backend/src/features/auth/guard/permission.guard.ts` | 1-491 |
| 共享权限验证 | `apps/nestjs-backend/src/features/auth/permission.service.ts` | 966-980 |
| 密码 Token 验证 | `apps/nestjs-backend/src/features/auth/permission.service.ts` | 581-599 |
| Header 工具函数 | `apps/nestjs-backend/src/features/auth/utils.ts` | 24-34 |
| shareId 解析 | `apps/nestjs-backend/src/features/auth/permission.service.ts` | 994-999 |
| 视图 Socket 越权检查 | `apps/nestjs-backend/src/features/share/share-socket.service.ts` | 40-49,90-96,148-163 |
| 复制权限检查 | `apps/nestjs-backend/src/features/base-share/base-share-open.controller.ts` | 163-172 |
| Header shareId 格式验证 | `apps/nestjs-backend/src/features/auth/permission.service.ts` | 994-999 |
| Header shareId 有效性验证 | `apps/nestjs-backend/src/features/auth/permission.service.ts` | 563-571 |
| Header 路径 shareId 无效拦截 | `apps/nestjs-backend/src/features/auth/permission.service.ts` | 602-608 |
| 视图共享密码验证 | `apps/nestjs-backend/src/features/share/share-auth.service.ts` | 43-65 |
| 基库共享密码验证 | `apps/nestjs-backend/src/features/base-share/base-share-auth.service.ts` | 36-59 |
| tryBaseSharePermissionCheck | `apps/nestjs-backend/src/features/auth/guard/permission.guard.ts` | 292-320 |
| ensureBaseShareAuth | `apps/nestjs-backend/src/features/auth/guard/permission.guard.ts` | 171-186 |

---

## 十三、permissionCheckWithPublicFallback 分支详解

### 13.1 方法入口与前置提取

**代码路径**: `permission.guard.ts:410-447`

```typescript
protected async permissionCheckWithPublicFallback(
  context: ExecutionContext,
  permissionCheck: () => Promise<boolean>
) {
  const req = context.switchToHttp().getRequest();
  const templateHeader = getTemplateHeader(req);       // X-Tea-Template
  const baseShareHeader = getBaseShareHeader(req);      // X-Tea-Base-Share
  const allowAnonymousType = this.reflector.getAllAndOverride<AllowAnonymousType | undefined>(
    IS_ALLOW_ANONYMOUS,
    [context.getHandler(), context.getClass()]
  );

  // 步骤 1: RESOURCE 级别 → 排他性资源鉴权
  // 步骤 2: Share link → 共享权限天花板
  // 步骤 3: Anonymous → 匿名用户处理
  // 步骤 4: Authenticated → 常规检查 + PUBLIC 兜底
}
```

**AllowAnonymousType 枚举** (`allow-anonymous.decorator.ts:3-7`)：
```typescript
export enum AllowAnonymousType {
  RESOURCE = 'resource',  // 排他性资源鉴权（共享/模板）
  USER = 'user',          // 仅需登录即可访问
  PUBLIC = 'public',      // 允许匿名 + PUBLIC 兜底
}
```

---

### 13.2 四步决策流程

```
permissionCheckWithPublicFallback()
        │
        ├─ 步骤 1: allowAnonymousType === RESOURCE ?
        │    └─ resolveResourcePermission(baseShareHeader, templateHeader)
        │         ├─ baseShareHeader 存在 → tryBaseSharePermissionCheck()
        │         │    ├─ shareId 有效 + 校验通过 → return true（共享权限）
        │         │    ├─ shareId 格式错 → return undefined → 尝试模板
        │         │    └─ shareId 校验失败 → 抛 RESTRICTED_RESOURCE ← 不兜底
        │         ├─ templateHeader 存在 → templatePermissionCheck()
        │         │    ├─ 校验通过 → return true（模板权限）
        │         │    └─ 校验失败 → 抛异常 ← 不兜底
        │         └─ 两者都无 → return undefined → 降级到步骤 2
        │
        ├─ 步骤 2: baseShareHeader 存在 ?
        │    └─ tryBaseSharePermissionCheck()
        │         ├─ return true → 共享权限放行
        │         └─ return undefined → 降级到步骤 3
        │
        ├─ 步骤 3: isAnonymous() ?
        │    └─ resolveAnonymousPermission()
        │         ├─ allowAnonymousType === PUBLIC → templatePermissionCheck()
        │         ├─ allowAnonymousType === USER → return true
        │         └─ 其他 → throw UnauthorizedException (401)
        │
        └─ 步骤 4: 登录用户 → permissionCheck()
             ├─ 通过 → return true（常规权限）
             └─ 失败 → allowAnonymousType === PUBLIC ?
                  ├─ 是 → resolvePublicFallback()
                  │    ├─ tryBaseShareFallback() → 吞掉异常
                  │    │    ├─ 通过 → return true（共享权限）
                  │    │    └─ 失败 → return undefined → 继续兜底
                  │    ├─ templatePermissionCheck()
                  │    │    ├─ 通过 → return true（模板权限）
                  │    │    └─ 失败 → throw originalError（原始错误）
                  │    └─ 都无 header → templatePermissionCheck() → 同上
                  └─ 否 → throw error（原始错误直接抛出）
```

---

### 13.3 场景一：Malformed Share Header

**定义**：请求头携带 `X-Tea-Base-Share: <非法值>`（不以 `shr` 开头）。

**代码追踪**：

```
步骤 1 (RESOURCE):
  resolveResourcePermission() → tryBaseSharePermissionCheck()
    getBaseShareIdByHeader("非法值")
      → 不以 'shr' 开头 → return null
    → return undefined
  无 templateHeader → return undefined
  → 降级到步骤 2

步骤 2:
  baseShareHeader 存在 → tryBaseSharePermissionCheck()
    getBaseShareIdByHeader("非法值") → return null
    → return undefined
  → 降级到步骤 3

步骤 3 (匿名用户):
  → resolveAnonymousPermission()
  → 取决于 allowAnonymousType

步骤 4 (登录用户):
  → permissionCheck() → 常规权限检查
```

**结论**：malformed share header **静默降级**，不影响后续判断。请求最终归属于：
- 匿名用户 → 模板权限（PUBLIC）/ 放行（USER）/ 401（无装饰器）
- 登录用户 → **常规权限**（完全忽略共享 header）

**关键**：`tryBaseSharePermissionCheck` 和 `tryBaseShareFallback` 都不会因 malformed header 抛出异常。

---

### 13.4 场景二：BaseShare 校验失败

**定义**：shareId 格式正确（以 `shr` 开头），但数据库中不存在或 `enabled=false`。

**代码追踪（步骤 1 - RESOURCE 路径）**：

```
步骤 1 (RESOURCE):
  resolveResourcePermission() → tryBaseSharePermissionCheck()
    getBaseShareIdByHeader("shrInvalid") → return "shrInvalid"
    @Permissions 装饰器存在, resourceId 存在
    → baseSharePermissionCheck(context, "shrInvalid")
      → ensureBaseShareAuth() → baseShareRequiresPassword("shrInvalid")
        → 数据库查不到 → return false → 跳过密码验证
      → validBaseSharePermissions("shrInvalid", resourceId, permissions)
        → getBaseSharePermissions("shrInvalid", resourceId)
          → getBaseShareInfo("shrInvalid") → return null
          → throw RESTRICTED_RESOURCE (403)  ← 直接抛出，不兜底
```

**步骤 1 中校验失败会直接抛异常，不会降级到步骤 2-4。**

**代码追踪（步骤 2 - 非 RESOURCE 路径）**：

```
步骤 2:
  baseShareHeader 存在 → tryBaseSharePermissionCheck()
    → baseSharePermissionCheck() → 抛 RESTRICTED_RESOURCE (403)
    → 异常向上传播，不返回 undefined
```

**步骤 2 中校验失败也直接抛异常。**

**代码追踪（步骤 4 - PUBLIC 兜底路径）**：

```
步骤 4 (登录用户, 常规权限检查失败, allowAnonymousType === PUBLIC):
  → resolvePublicFallback()
    → tryBaseShareFallback()
      → baseSharePermissionCheck() → 抛 RESTRICTED_RESOURCE
      → catch 捕获 → return undefined  ← 吞掉异常！
    → baseShareResult === undefined → 继续兜底
    → templatePermissionCheck()
      → 校验通过 → return true（模板权限）
      → 校验失败 → throw originalError（常规权限的原始错误）
```

**关键差异**：`tryBaseShareFallback` 会吞掉 baseShare 校验失败的异常，降级为模板权限检查。

**结论**：

| 上下文 | 校验失败后的归属 | 最终错误码 |
|-------|---------------|-----------|
| 步骤 1 (RESOURCE) | **不降级**，直接抛出 | `RESTRICTED_RESOURCE` (403) |
| 步骤 2 (非 RESOURCE) | **不降级**，直接抛出 | `RESTRICTED_RESOURCE` (403) |
| 步骤 4 PUBLIC 兜底 | 降级为**模板权限** | 模板失败则抛常规权限的原始错误 |

---

### 13.5 场景三：PUBLIC 兜底

**定义**：接口装饰 `@AllowAnonymous(AllowAnonymousType.PUBLIC)`，登录用户常规权限检查失败。

**代码追踪**：

```
步骤 4:
  permissionCheck() → 失败 → 抛异常
  allowAnonymousType === PUBLIC → resolvePublicFallback(context, baseShareHeader, error)

  resolvePublicFallback():
  ├─ tryBaseShareFallback()
  │    ├─ 无 baseShareHeader → return undefined
  │    ├─ shareId 格式错 → return undefined
  │    ├─ baseSharePermissionCheck() 通过 → return true（共享权限）
  │    └─ baseSharePermissionCheck() 失败 → catch → return undefined
  │
  ├─ baseShareResult !== undefined → return baseShareResult
  │
  └─ baseShareResult === undefined → templatePermissionCheck()
       ├─ 通过 → return true（模板权限）
       └─ 失败 → throw originalError（常规权限的原始错误，不是模板错误）
```

**tryBaseShareFallback 与 tryBaseSharePermissionCheck 的核心区别**：

| 维度 | tryBaseSharePermissionCheck | tryBaseShareFallback |
|-----|---------------------------|---------------------|
| 调用位置 | 步骤 1、步骤 2 | 步骤 4 的 resolvePublicFallback |
| 异常处理 | **不捕获**，直接向上传播 | **吞掉异常**，return undefined |
| 返回 undefined 含义 | 格式错/条件不满足，跳过共享检查 | 格式错/校验失败，降级到模板 |
| 校验失败时 | 请求被拒绝 (403) | 请求降级到模板权限 |
| 设计意图 | 共享权限作为天花板，不可绕过 | 共享权限作为兜底尝试，失败不应阻断 |

**结论**：PUBLIC 兜底时的权限归属顺序：
1. 有 baseShareHeader 且校验通过 → **共享权限**
2. 有 baseShareHeader 但校验失败 → **模板权限**（异常被吞掉）
3. 无 baseShareHeader → **模板权限**
4. 模板权限也失败 → **抛常规权限的原始错误**（非模板错误）

---

### 13.6 完整分支归属与错误码速查

**行号参考**: `permission.guard.ts:410-447`

| 条件组合 | 归属权限 | 放行条件 | 失败错误码 |
|---------|---------|---------|-----------|
| RESOURCE + valid shareHeader + 校验通过 | 共享权限 | baseSharePermissionCheck 通过 | `RESTRICTED_RESOURCE` (403) |
| RESOURCE + valid shareHeader + 校验失败 | **无降级** | 不放行 | `RESTRICTED_RESOURCE` (403) |
| RESOURCE + malformed shareHeader + valid templateHeader | 模板权限 | templatePermissionCheck 通过 | 匿名→`UNAUTHORIZED`(401) / 登录→`RESTRICTED_RESOURCE`(403) |
| RESOURCE + 无 header | 降级步骤 2-4 | 取决于后续步骤 | 取决于后续步骤 |
| 非RESOURCE + valid shareHeader + 校验通过 | 共享权限 | tryBaseSharePermissionCheck 通过 | `RESTRICTED_RESOURCE` (403) |
| 非RESOURCE + valid shareHeader + 校验失败 | **无降级** | 不放行 | `RESTRICTED_RESOURCE` (403) |
| 非RESOURCE + malformed shareHeader | 降级步骤 3/4 | 取决于后续步骤 | 取决于后续步骤 |
| 匿名 + PUBLIC | 模板权限 | templatePermissionCheck 通过 | `UNAUTHORIZED`(401) 或 `RESTRICTED_RESOURCE`(403) |
| 匿名 + USER | 无需权限 | return true | 不失败 |
| 匿名 + 无装饰器 | **拒绝** | 不放行 | `UNAUTHORIZED` (401) |
| 登录 + 常规权限通过 | 常规权限 | permissionCheck 通过 | `RESTRICTED_RESOURCE` (403) |
| 登录 + 常规失败 + PUBLIC + shareHeader通过 | 共享权限 | tryBaseShareFallback 通过 | `RESTRICTED_RESOURCE` (403) |
| 登录 + 常规失败 + PUBLIC + shareHeader失败 | 模板权限 | templatePermissionCheck 通过 | 抛常规权限原始错误 |
| 登录 + 常规失败 + PUBLIC + 模板失败 | **拒绝** | 不放行 | 常规权限原始错误 |
| 登录 + 常规失败 + 非PUBLIC | **拒绝** | 不放行 | 常规权限原始错误 |

---

### 13.7 关键设计洞察

1. **共享权限的两面性**：
   - 步骤 1/2 中：共享权限是**天花板**（ceiling），校验失败直接拒绝，不降级
   - 步骤 4 兜底中：共享权限是**垫脚石**（fallback），校验失败被吞掉，降级到模板

2. **malformed header 的隐含风险**：
   - `getBaseShareIdByHeader` 对非 `shr` 前缀的值返回 `null`
   - 所有共享检查被静默跳过，请求走常规权限
   - **不会报错**，但也不会走共享路径——攻击者无法通过伪造 header 绕过常规权限

3. **resolvePublicFallback 的错误保留**：
   - 模板权限检查失败时，抛出的不是模板错误，而是**常规权限的原始错误**
   - 这意味着用户看到的是"你没有操作权限"而非"模板权限不足"，避免暴露内部权限结构

4. **RESOURCE 级别的排他性**：
   - `allowAnonymousType === RESOURCE` 时，只认资源级鉴权（共享/模板）
   - 即使有有效的 baseShareHeader，如果后续 templateHeader 也无效，会降级到步骤 2
   - 步骤 2 中相同的 shareHeader 会重新走 `tryBaseSharePermissionCheck`，不会丢失

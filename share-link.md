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

## 七、核心代码模块索引

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

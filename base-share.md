# 数据底座对外分享权限边界分析

## 一、整体架构概览

数据底座的分享体系由三层权限边界协同工作：

```
┌─────────────────────────────────────────────────────────┐
│                    空间访问判定层                        │
│  SpaceService + PermissionService + CollaboratorModel  │
│  判定用户是否具备空间/基表的访问权限                      │
└────────────────────────────┬────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────┐
│                    分享令牌发放层                        │
│  ShareAuthService + BaseShareAuthService + JwtStrategy │
│  发放带权限边界的访问令牌，支持密码保护                   │
└────────────────────────────┬────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────┐
│                   字段可见范围控制层                      │
│  ShareService + isNotHiddenField + FieldService        │
│  控制分享视图中字段的可见性，多级过滤确保数据安全          │
└─────────────────────────────────────────────────────────┘
```

---

## 二、空间访问判定逻辑

### 2.1 核心判定流程

空间访问判定是权限体系的第一道防线，代码位于 `src/features/space/space.service.ts:108-123`。

```typescript
// space.service.ts:108-123
async getSpaceById(spaceId: string) {
  const space = await this.prismaService.space.findFirst({...});
  if (!space) {
    throw new CustomHttpException('Space not found', ...);
  }
  const role = await this.permissionService.getRoleBySpaceId(spaceId);
  if (!role) {
    throw new CustomHttpException('You have no permission to access this space', ...);
  }
  return { ...space, role };
}
```

### 2.2 权限继承模型

`PermissionService`（`src/features/auth/permission.service.ts`）实现了三级权限继承：

| 层级 | 判定方法 | 权限来源 |
|------|---------|---------|
| 空间级 | `getPermissionBySpaceId()` | 空间协作者角色 |
| 基表级 | `getPermissionByBaseId()` | 基表协作者角色 + 空间角色权限的并集 |
| 表级 | `getPermissionByTableId()` | 通过 tableId 向上追溯到 baseId，复用基表权限 |

关键代码（`permission.service.ts:354-389`）：
```typescript
private async getPermissionByBaseId(baseId: string) {
  const role = await this.getRoleByBaseId(baseId);
  const spaceRole = await this.getRoleBySpaceId(
    (await this.getUpperIdByBaseId(baseId)).spaceId
  );
  const basePermissions = role ? getPermissions(role) : [];
  const spacePermissions = spaceRole ? getPermissions(spaceRole) : [];
  return union(basePermissions, spacePermissions);  // 取并集
}
```

### 2.3 AccessToken 附加校验

当使用 PAT（个人访问令牌）时，会在用户权限基础上叠加令牌的资源限制：
```typescript
// permission.service.ts:416-435
async getPermissions(resourceId: string, accessTokenId?: string) {
  const userPermissions = await this.getPermissionsByResourceId(resourceId);
  if (accessTokenId) {
    const accessTokenPermission = await this.getPermissionsByAccessToken(resourceId, accessTokenId);
    return intersection(userPermissions, accessTokenPermission);  // 取交集
  }
  return userPermissions;
}
```

令牌限制包括：
- `spaceIds`：可访问的空间白名单
- `baseIds`：可访问的基表白名单
- `hasFullAccess`：是否拥有完全访问权
- `scopes`：可执行的操作范围

### 2.4 基表分享的空间边界校验

`base-share` 模式下，`PermissionService.getBaseSharePermissions()`（`permission.service.ts:602-652`）会按以下顺序执行严格的权限控制：

#### 2.4.1 资源归属校验与硬拒绝机制

节点级分享（`nodeId != null`）时，首先执行资源归属校验，**校验失败直接抛出异常（硬拒绝），不存在权限收敛**：

```typescript
// permission.service.ts:617-634
if (nodeId) {
  // Node-level share: verify the resource belongs to the shared node subtree
  const resourceBelongsToShare = await this.checkResourceBelongsToShare(
    resourceId,
    baseId,
    nodeId
  );

  if (!resourceBelongsToShare) {
    this.logger.warn(
      `[BaseShare] Resource ${resourceId} is not accessible via share ${shareId}`
    );
    // 硬拒绝：直接抛出异常，不返回任何权限，请求终止
    throw new CustomHttpException(
      `Resource ${resourceId} is not accessible via share ${shareId}`,
      HttpErrorCode.RESTRICTED_RESOURCE
    );
  }
}
// When nodeId is null (whole-base share), all resources in the base are accessible
```

**资源归属校验的完整判定链**：
```
resourceId
    │
    ▼
checkResourceBelongsToShare() → 根据 ID 前缀分发
    │
    ├─ Base (bso) → resourceId === baseId
    │
    ├─ Table (tbl) → checkTableBelongsToShare()
    │   ├─ 校验 table.baseId === 分享 baseId
    │   ├─ 校验 table 是否在分享节点子树内 (isTableAllowedByNodeId)
    │   └─ 校验失败时尝试链接字段穿透 (isTableLinkedFromSharedNode)
    │
    ├─ View (viw) → checkViewBelongsToShare() → 追溯到所属 table 后复用 table 校验逻辑
    │
    ├─ Field (fld) → checkFieldBelongsToShare() → 追溯到所属 table 后复用 table 校验逻辑
    │
    └─ App (app) → checkAppBelongsToShare() → 校验 app 节点是否在分享子树内
```

#### 2.4.2 关联表穿透机制

当表不在分享节点子树内时，会尝试链接字段穿透（`isTableLinkedFromSharedNode`）：
1. 收集分享节点子树内的所有表
2. 查询这些表中的所有链接字段
3. 如果任何链接字段的 `foreignTableId` 指向目标表，则允许访问
4. 确保链接功能正常工作，即使关联表不在分享范围内

#### 2.4.3 权限降级（归属校验通过后）

资源归属校验通过后，根据分享配置和用户身份进行权限降级（这是真正的"权限收敛"）：

| 权限模式 | 触发条件 | 权限内容 |
|---------|---------|---------|
| 可编辑权限 | `allowEdit=true` **且** 用户已登录（非匿名） | `getPermissions(Role.Editor)` - `SHARE_EXCLUDED_PERMISSIONS` |
| 匿名只读权限 | 上述条件不满足 | `TemplatePermissions` + `allowCopy ? ['record|copy'] : []` |

---

### 2.5 匿名只读与可编辑权限的详细触发条件

#### 2.5.1 可编辑权限触发条件（**必须同时满足**）

```typescript
// permission.service.ts:640-644
if (baseShare.allowEdit && !this.isAnonymous()) {
  return getPermissions(Role.Editor).filter((p) => !SHARE_EXCLUDED_PERMISSIONS.has(p));
}
```

**条件 1：`baseShare.allowEdit === true`**
- 分享创建时通过 `BaseShareService.updateBaseShare()` 设置
- 仅支持表（Table）和文件夹（Folder）节点类型
- `allowEdit` 与 `allowSave` 互斥：设置 `allowEdit=true` 会强制 `allowSave=false`

**条件 2：`!this.isAnonymous()`（用户已登录）**
- 判定逻辑：`isAnonymous(this.cls.get('user.id'))`
- 匿名用户 ID 为 `ANONYMOUS_USER_ID`（值为 `"anon_anonymous"`）
- 即使 `allowEdit=true`，匿名用户也**无法**获得编辑权限

**可编辑权限的范围限制（排除敏感操作）**：
`SHARE_EXCLUDED_PERMISSIONS` 集合中定义了即使是可编辑模式也禁止的操作：
```typescript
// permission.service.ts:35-41
const SHARE_EXCLUDED_PERMISSIONS = new Set<Action>([
  'view|share',        // 禁止分享视图
  'space|invite_email',// 禁止空间邮件邀请
  'base|invite_email', // 禁止基表邮件邀请
  'user|email_read',   // 禁止读取用户邮箱
  'user|integrations', // 禁止读取用户集成配置
]);
```

#### 2.5.2 匿名只读权限触发条件（**满足任一即可**）

```typescript
// permission.service.ts:646-651
const permissions = [...TemplatePermissions];
if (baseShare.allowCopy) {
  permissions.push('record|copy');
}
return permissions;
```

**触发场景 1：`baseShare.allowEdit !== true`**
- 分享未开启可编辑功能
- 或者 `allowEdit` 为 `null`/`false`

**触发场景 2：用户为匿名用户**
- `this.isAnonymous() === true`
- 即使 `allowEdit=true`，匿名用户也只能获得只读权限

**匿名只读权限的组成**：
- 基础权限：`TemplatePermissions`（完整的只读权限集，定义于 `packages/core/src/auth/role/template.ts`）
- 可选追加：`baseShare.allowCopy === true` 时追加 `record|copy` 权限

#### 2.5.3 权限判定的优先级与边界

```
请求携带 X-Tea-Base-Share: shrxxx Header
    │
    ▼
PermissionGuard.tryBaseSharePermissionCheck() 被触发
    │
    ▼
permissionService.validBaseSharePermissions(shareId, resourceId, permissions)
    │
    ├─ getBaseSharePermissions(shareId, resourceId)
    │   ├─ 校验 share 是否存在且 enabled
    │   ├─ nodeId 存在时校验资源归属（失败 → 硬拒绝）
    │   └─ 归属校验通过后 → 权限降级判定
    │       ├─ allowEdit && 已登录 → Editor - 排除权限
    │       └─ 其他情况 → TemplatePermissions + 可选 record|copy
    │
    └─ 校验所需 permissions 是否在返回的权限集合内
        ├─ 全部满足 → 权限写入 cls.permissions，请求继续
        └─ 任一不满足 → 抛出 RESTRICTED_RESOURCE 异常
```

> **关键边界**：分享链接的权限是"天花板"，即使已登录用户原本拥有更高权限（如 Owner），通过分享链接访问时也只能获得分享配置的权限级别。这在 `PermissionGuard.permissionCheckWithPublicFallback()` 中通过优先检查 share header 实现。

---

### 2.6 接口边界异常处理 - 跳过分享权限检查的条件

`PermissionGuard.tryBaseSharePermissionCheck()`（`permission.guard.ts:292-320`）在以下三种场景下会**直接跳过分享权限检查**，返回 `undefined` 让请求走正常用户权限路径：

```typescript
// permission.guard.ts:292-320
private async tryBaseSharePermissionCheck(
  context: ExecutionContext,
  baseShareHeader: string | undefined
): Promise<boolean | undefined> {
  // ... 省略 header 解析 ...
  
  // 跳过条件1：接口缺少 @Permissions 装饰器标记
  const permissions = this.reflector.getAllAndOverride<Action[] | undefined>(PERMISSIONS_KEY, ...);
  if (!permissions?.length) {
    return undefined;
  }
  
  // 跳过条件2：资源ID为空或属于Space级别
  const resourceId = this.getResourceId(context) || this.defaultResourceId(context);
  if (!resourceId || resourceId.startsWith(IdPrefix.Space)) {
    return undefined;
  }
  
  return await this.baseSharePermissionCheck(context, shareId);
}
```

#### 2.6.1 跳过场景1：缺少 @Permissions 装饰器

**典型接口**：
- `GET /api/user/me` - 获取当前用户信息（仅需登录）
- `POST /api/auth/login` - 登录接口（@Public）
- 其他通用用户级接口，仅标记 `@AllowAnonymous(USER)` 而无具体权限要求

**代码注释说明**：
> "Skip share path for endpoints without @Permissions (e.g. /user/me),
> otherwise baseSharePermissionCheck throws ForbiddenException."

#### 2.6.2 跳过场景2：资源属于 Space 级别（`spc` 前缀）

**典型接口**：
- `GET /api/space` - 获取空间列表
- `POST /api/space` - 创建空间
- `GET /api/space/:spaceId` - 获取空间详情
- `PUT /api/space/:spaceId` - 更新空间信息
- `POST /api/share/:shareId/base/copy` - 复制分享（`@ResourceMeta('spaceId', 'body')` 标记资源为 `spaceId`）

**代码注释说明**：
> "Skip share check when the target resource is outside the share scope.
> e.g. space-level endpoints (GET /space, POST /share/:id/base/copy with spaceId in body)
> should use the user's own permissions, not the share's."

**ID 前缀规则**（`packages/core/src/utils/id-generator.ts:3-23`）：
| 前缀 | 资源类型 | 是否跳过分享检查 |
|------|---------|----------------|
| `spc` | Space | 是 |
| `bse` | Base | 否 |
| `tbl` | Table | 否 |
| `viw` | View | 否 |
| `fld` | Field | 否 |
| `rec` | Record | 否 |

#### 2.6.3 跳过场景3：Header 中无有效分享标识

- 未携带 `X-Tea-Base-Share` Header
- Header 值无法解析出有效的 `shr` 前缀 shareId

---

### 2.7 view-share 与 base-share 两条鉴权链路的本质区别

在分析 Guard 职责之前，必须先明确系统中**两条独立的分享鉴权链路**，它们在"是否受分享上限约束"的判定上标准完全不同：

| 维度 | view-share 链路 | base-share 链路 |
|------|---------------|----------------|
| 控制器 | `share.controller.ts` | `base-share-open.controller.ts` |
| 路由前缀 | `api/share` | `api/share` |
| 类级别装饰器 | `@Controller('api/share')` + `@Public()` | `@Controller('api/share')`（无类级别 `@Public()`） |
| 核心守卫 | 仅 `ShareAuthGuard` | `BaseShareAuthGuard` ± `PermissionGuard` |
| `@Permissions` | ❌ 无 | ✅ 部分接口有 |
| `@ResourceMeta` | ❌ 无 | ✅ 部分接口有 |
| 分享标识传递 | URL 参数 `:shareId` | URL 参数 + `X-Tea-Base-Share` Header |
| 是否经过 PermissionGuard | ❌ 从不经过 | ✅ 部分接口经过 |
| "受分享上限约束"概念 | ❌ 不存在 | ✅ 仅存在于使用 `PermissionGuard` 的接口 |

**代码证据**：
```typescript
// share.controller.ts:70-72 —— 类级别 @Public()，所有接口都是公开的
@Controller('api/share')
@Public()
export class ShareController {
  // 所有接口仅使用 ShareAuthGuard，无 PermissionGuard
  @UseGuards(ShareAuthGuard)
  @Get('/:shareId/view/records')
  async getViewRecords() { ... }
}

// base-share-open.controller.ts:27-28 —— 无类级别 @Public()
@Controller('api/share')
export class BaseShareOpenController {
  // 部分接口仅认证
  @Public()
  @UseGuards(BaseShareAuthGuard)
  @Get('/:shareId/base')
  async getBaseShare() { ... }
  
  // 部分接口认证+权限检查
  @UseGuards(BaseShareAuthGuard, PermissionGuard)
  @Permissions('base|create')
  @ResourceMeta('spaceId', 'body')
  @Post('/:shareId/base/copy')
  async copyBaseShare() { ... }
}
```

> **关键结论**："受分享上限约束"的判断逻辑**仅存在于 `PermissionGuard.tryBaseSharePermissionCheck()`**，因此只有同时满足以下条件的接口才会触发该判断：
> 1. 属于 **base-share 链路**
> 2. 使用了 **`PermissionGuard`**
> 3. 携带 **`X-Tea-Base-Share` Header**
> 
> view-share 链路由于从不经过 PermissionGuard，因此不存在"受/不受分享上限约束"的概念，其权限控制完全在 `ShareAuthGuard` 和 `ShareService` 内部完成。

---

### 2.8 Guard 职责分工与执行顺序

#### 2.8.1 BaseShareAuthGuard vs PermissionGuard 职责划分

| Guard | 核心职责 | 执行阶段 | 判定结果 |
|-------|---------|---------|---------|
| **BaseShareAuthGuard** | **认证门禁**：验证分享链接的有效性，处理密码保护，注入用户身份 | 第一道防线 | 分享不存在/禁用 → NOT_FOUND<br>需要密码未认证 → UNAUTHORIZED_SHARE<br>认证通过 → 挂载 `req.baseShareInfo` |
| **PermissionGuard** | **权限判定**：根据分享配置动态计算权限，校验操作是否被允许 | 第二道防线 | 权限不足 → RESTRICTED_RESOURCE<br>权限满足 → 写入 `cls.permissions` |

**BaseShareAuthGuard 认证流程**（`base-share-auth.guard.ts:20-58`）：
```typescript
async validate(context: ExecutionContext, shareId: string) {
  // 1. 获取分享信息（校验存在性 + enabled）
  const shareInfo = await this.baseShareAuthService.getBaseShareInfo(shareId);
  req.baseShareInfo = shareInfo;
  
  // 2. 注入用户身份（保留已登录用户）
  const currentUserId = this.cls.get('user.id');
  if (!currentUserId) {
    this.cls.set('user', { id: ANONYMOUS_USER_ID, ... });
  }
  
  // 3. 密码保护检查
  const hasPassword = await this.baseShareAuthService.hasPassword(shareId);
  if (hasPassword) {
    return (await super.canActivate(context)) as boolean;  // JWT 策略验证
  }
  return true;
}
```

**BaseShareAuthGuard 的两种使用模式**：

| 模式 | 装饰器配置 | 适用场景 | 权限来源 |
|------|-----------|---------|---------|
| 仅认证 | `@UseGuards(BaseShareAuthGuard)`<br>`@AllowAnonymous()` | 查询分享元数据（`GET /:shareId/base`） | 不做权限检查，仅认证分享有效性 |
| 认证+权限 | `@UseGuards(BaseShareAuthGuard, PermissionGuard)`<br>`@Permissions(...)`<br>`@ResourceMeta(...)` | 复制分享、查询分享内资源 | 先认证，再通过 PermissionGuard 做分享权限校验 |

#### 2.8.2 完整执行顺序

`PermissionGuard.canActivate()` → `permissionCheckWithPublicFallback()` 的完整判定链：

```
请求到达 PermissionGuard
    │
    ▼
┌─────────────────────────────────────────────┐
│  1. @Public() 装饰器检查                      │
│  → 标记为 public → 直接放行，不做任何校验       │
└───────────────────┬───────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│  2. @DisabledPermission() 装饰器检查           │
│  → 标记为 disabled → 直接放行                  │
└───────────────────┬───────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│  3. permissionCheckWithPublicFallback()      │
│  ┌────────────────────────────────────────┐  │
│  │ 3.1 RESOURCE 模式检查                   │  │
│  │ @AllowAnonymous(RESOURCE) → 优先使用      │  │
│  │ base share / template 权限                │  │
│  └─────────────┬────────────────────────────┘  │
│                │                               │
│                ▼                               │
│  ┌────────────────────────────────────────┐  │
│  │ 3.2 分享 Header 检查（分享上限约束）     │  │
│  │ 携带 X-Tea-Base-Share →                 │  │
│  │ tryBaseSharePermissionCheck()           │  │
│  │ → 跳过条件检查（@Permissions、Space级）  │  │
│  │ → baseSharePermissionCheck()            │  │
│  └─────────────┬────────────────────────────┘  │
│                │                               │
│                ▼                               │
│  ┌────────────────────────────────────────┐  │
│  │ 3.3 匿名用户处理                        │  │
│  │ @AllowAnonymous(PUBLIC) → template 检查 │  │
│  │ @AllowAnonymous(USER) → 直接放行        │  │
│  │ 其他 → 抛出 UnauthorizedException       │  │
│  └─────────────┬────────────────────────────┘  │
│                │                               │
│                ▼                               │
│  ┌────────────────────────────────────────┐  │
│  │ 3.4 已登录用户正常权限检查               │  │
│  │ permissionCheck() →                     │  │
│  │ 校验用户在该资源上的实际权限              │  │
│  └─────────────┬────────────────────────────┘  │
│                │                               │
│                ├─ 成功 → 放行                  │
│                └─ 失败 → PUBLIC fallback       │
│                   ┌─────────────────────────┐  │
│                   │ 3.5 Public fallback    │  │
│                   │ @AllowAnonymous(PUBLIC) │  │
│                   │ → tryBaseShareFallback  │  │
│                   │ → templatePermissionCheck ││
│                   └─────────────────────────┘  │
└─────────────────────────────────────────────┘
```

**执行优先级**：
1. `@Public()` > 所有检查
2. 资源级认证（`AllowAnonymousType.RESOURCE`）> 分享链接权限 > 匿名处理 > 正常用户权限
3. 分享链接权限是"天花板"，已登录用户的权限也不能超过分享配置

---

### 2.9 受分享上限约束 / 不受约束的接口判定标准

> **前置说明**：本节内容**仅适用于 base-share 链路**。view-share 链路由于从不经过 `PermissionGuard`，不存在"受/不受分享上限约束"的概念。

#### 2.9.1 判定标准

| 维度 | 受分享上限约束 | 不受分享上限约束 |
|------|--------------|----------------|
| **链路类型** | ✅ base-share 链路 | ❌ view-share 链路，或非分享链路 |
| **使用 PermissionGuard** | ✅ 是 | ❌ 否（仅使用 `BaseShareAuthGuard`） |
| **@Permissions** | ✅ 有标记（非空数组） | ❌ 无标记或空数组 |
| **资源类型** | ✅ Base/Table/View/Field/Record 级 | ❌ Space 级（`spc` 前缀） |
| **Header** | ✅ 携带 `X-Tea-Base-Share` | ❌ 不携带，或解析不出 shareId |

**判定逻辑表达式**：
```
受约束 = (链路类型 = base-share)
        AND (使用 PermissionGuard)
        AND (X-Tea-Base-Share 存在且可解析)
        AND (@Permissions 非空)
        AND (resourceId 非空且不以 spc 开头)
```

#### 2.9.2 base-share 链路三个接口的实际情况

`base-share-open.controller.ts` 中共有 3 个接口，它们的约束情况如下：

| 接口 | 装饰器配置 | 是否受约束 | 原因 |
|------|-----------|-----------|------|
| `POST /:shareId/base/auth` | `@Public()`<br>`@UseGuards(BaseShareAuthLocalGuard)` | ❌ 不受约束 | 无 `PermissionGuard`，无 `@Permissions` |
| `GET /:shareId/base` | `@Public()`<br>`@UseGuards(BaseShareAuthGuard)`<br>`@AllowAnonymous()` | ❌ 不受约束 | 无 `PermissionGuard`，无 `@Permissions` |
| `POST /:shareId/base/copy` | `@UseGuards(BaseShareAuthGuard, PermissionGuard)`<br>`@Permissions('base|create')`<br>`@ResourceMeta('spaceId', 'body')` | ❌ 不受约束 | 有 `PermissionGuard` 和 `@Permissions`，但资源是 Space 级 → 触发跳过条件2 |

> **重要发现**：在当前代码中，`base-share-open.controller.ts` 的三个接口**实际上都不受分享上限约束**！
> - 前两个接口没有 `PermissionGuard`
> - 第三个接口虽然有 `PermissionGuard`，但 `@ResourceMeta('spaceId', 'body')` 导致 resourceId 是 Space 级（`spc` 前缀），触发跳过条件2

**代码证据 - 跳过条件2触发时的日志**：
```typescript
// permission.guard.ts:313-318
// e.g. space-level endpoints (GET /space, POST /share/:id/base/copy with spaceId in body)
// should use the user's own permissions, not the share's.
const resourceId = this.getResourceId(context) || this.defaultResourceId(context);
if (!resourceId || resourceId.startsWith(IdPrefix.Space)) {
  return undefined;  // 跳过分享权限检查
}
```

#### 2.9.3 不受约束接口示例

**示例1：获取分享元数据（base-share 链路）**
```typescript
// base-share-open.controller.ts:56-80
@Public()
@UseGuards(BaseShareAuthGuard)  // 仅 BaseShareAuthGuard，无 PermissionGuard
@AllowAnonymous()
@Get('/:shareId/base')
async getBaseShare() { ... }
```
- ❌ 无 `PermissionGuard` 参与，无 `@Permissions` 检查
- **结论**：不受分享上限约束，仅做分享有效性认证

**示例2：复制分享到用户空间（base-share 链路）**
```typescript
// base-share-open.controller.ts:150-220
@UseGuards(BaseShareAuthGuard, PermissionGuard)
@Permissions('base|create')
@ResourceMeta('spaceId', 'body')  // ⚠️ 资源是 Space 级
@Post('/:shareId/base/copy')
async copyBaseShare() { ... }
```
- ✅ 有 `PermissionGuard` 和 `@Permissions('base|create')`
- ❌ 资源是 Space（`spc` 前缀）→ 触发跳过条件2
- **结论**：不受分享上限约束，走用户自身权限检查（即使携带分享 Header）

**示例3：获取当前用户信息（非分享链路）**
```typescript
// 无 @Permissions 装饰器
@Get('/user/me')
async getMe() { ... }
```
- ❌ 无 `@Permissions` → 触发跳过条件1
- **结论**：不受分享上限约束，仅需登录

**示例4：获取空间列表（非分享链路）**
```typescript
@Permissions('space|read')
@Get('/space')
async getSpaceList() { ... }
```
- ✅ 有 `@Permissions`
- ❌ 资源是 Space（`spc` 前缀）→ 触发跳过条件2
- **结论**：不受分享上限约束，即使携带分享 Header 也走用户空间权限

**示例5：查询视图记录（view-share 链路）**
```typescript
// share.controller.ts:126-137
@ShareLinkView()
@UseGuards(ShareAuthGuard)  // 仅 ShareAuthGuard，无 PermissionGuard
@AllowAnonymous()
@Get('/:shareId/view/records')
async getViewRecords() { ... }
```
- ❌ 属于 view-share 链路，不经过 PermissionGuard
- **结论**：不存在"受分享上限约束"的概念，权限控制在 ShareService 内部完成

#### 2.9.4 理论上的受约束场景（代码中暂无实际接口）

虽然当前代码中没有实际的"受分享上限约束"接口，但如果未来添加如下配置的接口，则会触发约束：

```typescript
// 假设的接口（当前代码中不存在）
@UseGuards(BaseShareAuthGuard, PermissionGuard)
@Permissions('base|read')
@ResourceMeta('baseId', 'params')  // ✅ 资源是 Base 级（bse 前缀）
@Get('/:shareId/base/info')
async getBaseInfo() { ... }
```

此时判定过程：
1. ✅ 属于 base-share 链路
2. ✅ 使用 `PermissionGuard`
3. ✅ 有 `@Permissions('base|read')`
4. ✅ 资源是 Base 级（`bse` 前缀，非 `spc`）
5. ✅ 携带 `X-Tea-Base-Share` Header
6. **结论**：受分享上限约束，匿名用户权限降级为 TemplatePermissions

---

## 三、分享令牌发放机制

### 3.1 两种分享模式

系统支持两种独立的分享体系：

| 模式 | 视图级分享 (view-share) | 基表级分享 (base-share) |
|------|------------------------|------------------------|
| 粒度 | 单个视图 | 整个基表或基表内节点 |
| 核心服务 | `ShareAuthService` | `BaseShareAuthService` |
| 守卫 | `ShareAuthGuard` | `BaseShareAuthGuard` |
| 令牌策略 | `SHARE_JWT_STRATEGY` | `BASE_SHARE_JWT_STRATEGY` |
| shareId 前缀 | `shr` (view) | `shr` (base) |

### 3.2 视图级分享令牌发放流程

**口令校验与令牌签发的真实职责划分**：

口令校验与令牌签发是**分离**的两个步骤，由不同组件负责：

| 组件 | 职责 | 代码位置 |
|------|------|---------|
| `ShareAuthLocalGuard` | 仅校验密码正确性，不签发 Token | `share/guard/share-auth-local.guard.ts:11-26` |
| `ShareController.auth()` | 调用 authToken 签发 JWT，**负责写入 Cookie** | `share/share.controller.ts:79-91` |

**第一步：密码认证（Guard 仅做校验）**

`ShareAuthLocalGuard` 只校验密码，校验通过后将 `shareId` 和 `password` 挂载到 `req` 对象上：
```typescript
// share/guard/share-auth-local.guard.ts:11-26
async canActivate(context: ExecutionContext) {
  const req = context.switchToHttp().getRequest();
  const shareId = req.params.shareId;
  const password = req.body.password;
  // 仅做密码校验
  const authShareId = await this.shareAuthService.authShareView(shareId, password);
  // 校验通过后将结果挂载到 req，供后续 Controller 使用
  req.shareId = authShareId;
  req.password = password;
  if (!authShareId) {
    throw new CustomHttpException('Incorrect password.', ...);
  }
  return true;
}
```

**第二步：签发 Token 与写入 Cookie（Controller 负责）**

`ShareController.auth()` 方法是真正签发 Token 并写入 Cookie 的地方：
```typescript
// share/share.controller.ts:79-91
@HttpCode(200)
@UseGuards(ShareAuthLocalGuard)
@Post('/:shareId/view/auth')
async auth(@Request() req: any, @Res({ passthrough: true }) res: Response) {
  const shareId = req.shareId;        // 从 Guard 挂载的结果中获取
  const password = req.password;      // 从 Guard 挂载的结果中获取
  const token = await this.shareAuthService.authToken({ shareId, password });
  // Controller 负责写入 Cookie
  res.cookie(shareId, token, {
    httpOnly: true,
    maxAge: 1000 * 60 * 60 * 24 * 7,  // 7 天有效期
  });
  return { token };
}
```

**第三步：令牌验证与用户注入**

`ShareAuthGuard`（`share/guard/auth.guard.ts:27-70`）是核心守卫：
```typescript
async validate(context: ExecutionContext, shareId: string) {
  // 1. 获取分享视图信息（含 shareMeta）
  const shareInfo = await this.shareAuthService.getShareViewInfo(shareId);
  
  // 2. 表单提交且要求登录时，走正常用户认证
  if (isShareSubmit && submit?.allow && submit?.requireLogin) {
    return this.authGuard.validate(context);
  }
  
  // 3. 其他情况注入匿名用户
  this.cls.set('user', { id: ANONYMOUS_USER_ID, name: ANONYMOUS_USER_ID, email: '' });
  
  // 4. 有密码时走 JWT 策略验证
  if (shareInfo.view?.shareMeta?.password) {
    return super.canActivate(context);
  }
  return true;
}
```

**第三步：JWT 策略校验**

`JwtStrategy`（`share/strategies/jwt.strategy.ts:32-39`）从 Cookie 中提取令牌并验证：
```typescript
async validate(payload: IJwtShareInfo) {
  const { shareId, password } = payload;
  const authShareId = await this.shareAuthService.authShareView(shareId, password);
  if (!authShareId) throw new UnauthorizedException();
  return authShareId;
}
```

### 3.3 基表级分享令牌流程

基表级分享的口令校验与 Cookie 写入职责划分与视图级完全一致：

| 组件 | 职责 | 代码位置 |
|------|------|---------|
| `BaseShareAuthLocalGuard` | 仅校验密码正确性 | `base-share/guard/base-share-auth-local.guard.ts:11-26` |
| `BaseShareOpenController.auth()` | 签发 JWT 并写入 Cookie | `base-share/base-share-open.controller.ts:36-54` |

基表级 Cookie 写入的完整代码：
```typescript
// base-share-open.controller.ts:36-54
@HttpCode(200)
@Public()
@UseGuards(BaseShareAuthLocalGuard)
@Post('/:shareId/base/auth')
async auth(
  @Request() req: Express.Request & { shareId: string; password: string },
  @Res({ passthrough: true }) res: Response
): Promise<IBaseShareAuthVo> {
  const shareId = req.shareId;
  const password = req.password;
  const token = await this.baseShareAuthService.authToken({ shareId, password });
  // Controller 负责写入 Cookie，生产环境额外增加 secure 和 sameSite 配置
  res.cookie(shareId, token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 1000 * 60 * 60 * 24 * 7, // 7 days
  });
  return { token };
}
```

`BaseShareAuthGuard`（`base-share/guard/base-share-auth.guard.ts:20-52`）与视图级类似，但：
- 保留已登录用户身份（用于复制操作溯源）
- 权限通过 `PermissionService.getBaseSharePermissions()` 动态计算

### 3.4 分享元数据 (IShareViewMeta)

分享的权限边界通过 `IShareViewMeta` 定义（`packages/core/src/models/view/view.schema.ts:18-29`）：
```typescript
const shareViewMetaSchema = z.object({
  allowCopy: z.boolean().optional(),           // 允许复制
  includeHiddenField: z.boolean().optional(),  // 包含隐藏字段
  password: z.string().min(3).optional(),      // 访问密码
  includeRecords: z.boolean().optional(),      // 包含记录数据
  submit: z.object({                           // 表单提交配置
    allow: z.boolean().optional(),
    requireLogin: z.boolean().optional(),
  }).optional(),
});
```

---

## 四、字段可见范围控制

字段可见性是最精细的权限边界，采用**多级过滤**机制确保数据安全。

### 4.1 三级过滤机制

```
查询请求
    │
    ▼
┌─────────────────────┐
│  第一级：视图隐藏字段过滤  │
│  filterHidden 参数控制      │
│  isNotHiddenField() 判定  │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────┐
│  第二级：visibleFieldIds 过滤 │
│  链接字段场景下的白名单      │
│  始终保留主键字段 (isPrimary) │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────┐
│  第三级：projection 投影  │
│  记录查询时的字段投影      │
│  仅返回过滤后的字段        │
└─────────────────────┘
```

### 4.2 第一级：视图隐藏字段过滤

`isNotHiddenField` 工具函数（`utils/is-not-hidden-field.ts:9-44`）根据视图类型判断字段可见性：

| 视图类型 | 判定逻辑 |
|---------|---------|
| Kanban | 堆叠字段、封面字段 + columnMeta.visible !== false |
| Gallery | 封面字段 + columnMeta.visible !== false |
| Calendar | 颜色字段、起止日期、标题字段 + columnMeta.visible !== false |
| Form | 仅 columnMeta.visible === true 的字段 |
| 其他 (Grid 等) | columnMeta.hidden !== true |

代码调用点在 `FieldService.getFieldsByQuery()`（`field/field.service.ts:877-879`）：
```typescript
if (query?.filterHidden) {
  result = result.filter((field) => isNotHiddenField(field.id, view));
}
```

### 4.3 第二级：visibleFieldIds 白名单过滤

`ShareService` 中两次应用此过滤（`share.service.ts:77-83` 和 `share.service.ts:231-237`）：
```typescript
// 获取视图字段
const fields = await this.fieldService.getFieldsByQuery(tableId, {
  viewId,
  filterHidden: Boolean(filterByViewId) || !shareMeta?.includeHiddenField,
});
const filteredFields = visibleFieldIds?.length
  ? fields.filter((f) => visibleFieldIds?.includes(f.id) || f.isPrimary)
  : fields;
```

**关键点**：
- `filterHidden` 受 `includeHiddenField` 和 `filterByViewId` 共同控制
- `visibleFieldIds` 来自链接字段的配置，用于限制链接弹窗中的可见字段
- 主键字段 (`isPrimary`) 始终保留，确保记录可识别

### 4.4 第三级：查询投影限制

在获取记录时，将过滤后的字段作为 projection 传入（`share.service.ts:96-97`）：
```typescript
projection: filteredFields.map((f) => f.id),
```

### 4.5 聚合计数中的字段校验

在 `getViewAggregations()` 中会预检查字段是否隐藏（`share.service.ts:169-171`）：
```typescript
if (shareInfo.view) {
  this.preCheckFieldHidden(shareInfo.view as IViewVo, key);
}
```

`preCheckFieldHidden` 方法（`share.service.ts:295-308`）：
```typescript
private preCheckFieldHidden(view: IViewVo, fieldId: string) {
  if (!view.shareMeta?.includeHiddenField && !isNotHiddenField(fieldId, view)) {
    throw new CustomHttpException('field is hidden, not allowed', ...);
  }
}
```

### 4.6 链接字段的特殊处理

`getViewLinkRecords()`（`share.service.ts:310-353`）对链接字段做额外校验：
1. 先调用 `preCheckFieldHidden()` 确保链接字段本身可见
2. 校验字段类型必须是 `FieldType.Link`
3. 对返回的关联记录仅返回 `id` 和 `title`（lookupFieldId 对应的值），不暴露其他字段

---

## 五、三者协作方式详解

### 5.1 完整请求处理链路

以 `GET /api/share/:shareId/view/records` 为例：

```
1. 请求到达 ShareController
   └─ @UseGuards(ShareAuthGuard)
      │
      ▼
2. ShareAuthGuard.validate()
   ├─ 调用 ShareAuthService.getShareViewInfo(shareId)
   │  └─ 校验 view.enableShare === true 且 deletedTime === null
   ├─ 解析 shareMeta，判定是否需要密码/登录
   ├─ 设置匿名用户或触发登录认证
   └─ 将 shareInfo 挂载到 req.shareInfo
      │
      ▼
3. ShareService.getViewRecords(shareInfo)
   ├─ 第一级过滤：调用 fieldService.getFieldsByQuery()
   │  ├─ filterHidden = !shareMeta.includeHiddenField
   │  └─ isNotHiddenField() 逐个判定
   ├─ 第二级过滤：visibleFieldIds 白名单过滤
   └─ 第三级过滤：将 filteredFields 作为 projection 调用 recordService.getRecords()
      │
      ▼
4. PermissionService 后台校验
   └─ 对敏感操作（如 buttonClick）调用 validPermissions()
      校验当前用户（匿名或登录）具备所需权限
```

### 5.2 关键协作点

#### 空间判定 → 令牌发放
- 空间权限校验不直接参与分享链路，但**基表分享**时通过 `PermissionService.getBaseSharePermissions()` 校验资源归属
- 分享创建时（`BaseShareService.createBaseShare()`），创建者必须具备基表管理权限

#### 令牌发放 → 字段控制
- `ShareAuthGuard` 将解析后的 `shareInfo`（含 `shareMeta`、`linkOptions`）传递给 `ShareService`
- `shareMeta.includeHiddenField` 直接决定 `filterHidden` 参数值
- `linkOptions.visibleFieldIds` 作为第二级过滤的白名单

#### 空间判定 → 字段控制
- 普通登录用户访问时，`PermissionService.getPermissions()` 校验表级读权限
- 分享用户访问时，空间判定被绕过，直接通过令牌+字段过滤实现权限控制

### 5.3 安全边界汇总

| 边界层级 | 保护机制 | 绕过风险点 |
|---------|---------|-----------|
| 空间层 | 协作者角色 + AccessToken 限制 | 分享链接绕过空间权限 |
| 令牌层 | JWT 签名 + httpOnly Cookie + 密码校验 | 密码泄露、Cookie 窃取 |
| 字段层 | 三级过滤 + preCheck + 链接字段脱敏 | includeHiddenField=true 时暴露隐藏字段 |

### 5.4 特殊权限组合场景

**场景 1：表单视图分享 + 允许提交 + 要求登录**
```
ShareAuthGuard 检测到 isShareSubmit + submit.requireLogin = true
→ 调用 authGuard.validate() 走正常登录流程
→ 登录用户身份保留
→ 提交时通过 formSubmit() 写入数据，不触发匿名限制
```

**场景 2：链接字段分享（link-view）**
```
shareId 以 'fld' 开头 → ShareAuthGuard 识别为链接视图
→ 调用 getLinkViewInfo() 校验链接字段权限
→ linkOptions 携带 visibleFieldIds 限制可见范围
→ 仅返回 lookupFieldId 对应的值作为 title
```

**场景 3：基表分享 + allowEdit = true + 匿名用户**
```
匿名用户访问 allowEdit=true 的分享链接
→ getBaseSharePermissions() 检测到 isAnonymous() = true
→ 权限收敛为 TemplatePermissions（只读）
→ 即使 allowEdit=true，匿名用户也无法编辑
→ 只能查看，allowCopy=true 时可复制记录
```

**场景 4：基表分享 + allowEdit = true + 已登录用户**
```
已登录用户访问 allowEdit=true 的分享链接
→ getBaseSharePermissions() 检测到 allowEdit=true 且 !isAnonymous()
→ 返回 Editor 权限，但排除 SHARE_EXCLUDED_PERMISSIONS
→ 可编辑记录，但无法分享视图、邀请协作者、读取用户邮箱
→ 用户原本的 Owner 权限被分享链接权限覆盖（收敛）
```

**场景 5：节点级分享 + 访问节点外资源**
```
用户访问节点级分享链接，但请求节点外的表数据
→ checkResourceBelongsToShare() 返回 false
→ 硬拒绝：抛出 RESTRICTED_RESOURCE 异常
→ 不存在权限降级，请求直接终止
→ 即使该用户是基表 Owner，通过分享链接也无法访问节点外资源
```

**场景 6：复制分享接口（POST /:shareId/base/copy）的权限判定**
```
已登录用户携带 X-Tea-Base-Share Header 请求复制分享
→ BaseShareAuthGuard 认证通过，挂载 req.baseShareInfo
→ PermissionGuard.tryBaseSharePermissionCheck() 被触发
→ 解析 @ResourceMeta('spaceId', 'body') → resourceId 是 spc 前缀
→ 触发跳过条件2（resourceId.startsWith(IdPrefix.Space)）
→ 返回 undefined，跳过分享权限检查
→ 执行 permissionCheck() 校验用户在该 spaceId 上的 base|create 权限
→ 权限不足 → 抛出异常；权限足够 → 继续执行复制逻辑
→ 💡 关键点：即使携带分享 Header，也走用户自身权限，不受分享上限约束
```

---

## 六、核心代码文件索引

| 模块 | 文件路径 | 核心职责 |
|------|---------|---------|
| 空间权限 | `src/features/space/space.service.ts` | 空间列表查询、权限校验 |
| 权限服务 | `src/features/auth/permission.service.ts` | 三级权限判定、基表分享权限 |
| 视图分享认证 | `src/features/share/share-auth.service.ts` | 视图分享信息获取、密码校验 |
| 基表分享认证 | `src/features/base-share/base-share-auth.service.ts` | 基表分享信息获取、密码校验 |
| 视图分享守卫 | `src/features/share/guard/auth.guard.ts` | 分享请求认证主入口 |
| 基表分享守卫 | `src/features/base-share/guard/base-share-auth.guard.ts` | 基表分享认证入口 |
| 分享业务逻辑 | `src/features/share/share.service.ts` | 字段过滤、记录查询、表单提交 |
| 基表分享管理 | `src/features/base-share/base-share.service.ts` | 分享 CRUD、刷新 shareId |
| 字段可见性 | `src/utils/is-not-hidden-field.ts` | 视图字段隐藏判定 |
| 字段服务 | `src/features/field/field.service.ts` | `getFieldsByQuery()` 隐藏字段过滤 |
| 访问令牌 | `src/features/access-token/access-token.service.ts` | PAT 令牌创建、验证 |
| 元数据定义 | `packages/core/src/models/view/view.schema.ts` | `IShareViewMeta` 类型定义 |
| JWT 策略 | `src/features/share/strategies/jwt.strategy.ts` | 视图分享 JWT 验证 |
| JWT 策略 | `src/features/base-share/strategies/jwt.strategy.ts` | 基表分享 JWT 验证 |
| 权限守卫 | `src/features/auth/guard/permission.guard.ts` | 基表分享权限校验入口、跳过逻辑、fallback 机制 |
| 基表分享管理 | `src/features/base-share/base-share.controller.ts` | 基表分享 CRUD 管理接口 |
| 模板角色定义 | `packages/core/src/auth/role/template.ts` | TemplatePermissions 权限集定义 |
| 允许匿名装饰器 | `src/features/auth/decorators/allow-anonymous.decorator.ts` | AllowAnonymousType 枚举定义 |
| ID 前缀定义 | `packages/core/src/utils/id-generator.ts` | IdPrefix 枚举（Space/Base/Table 等前缀） |
| 本地认证守卫 | `src/features/share/guard/share-auth-local.guard.ts` | 视图分享密码校验 |
| 本地认证守卫 | `src/features/base-share/guard/base-share-auth-local.guard.ts` | 基表分享密码校验 |
| 视图分享控制器 | `src/features/share/share.controller.ts` | 视图分享完整路由（类级别 `@Public()`）、Cookie 写入、所有 view-share 接口入口 |
| 基表分享 OpenAPI | `src/features/base-share/base-share-open.controller.ts` | 基表分享公开接口（认证、查询、复制），三种不同的守卫组合模式 |

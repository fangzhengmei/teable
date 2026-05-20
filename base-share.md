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

`base-share` 模式下，`PermissionService.getBaseSharePermissions()`（`permission.service.ts:602-652`）会校验：

1. **资源归属校验**：通过 `checkResourceBelongsToShare()` 递归校验资源是否属于分享节点子树
2. **关联表穿透**：`isTableLinkedFromSharedNode()` 允许链接字段引用的外部表被访问（即使不在分享节点内）
3. **权限降级**：
   - `allowEdit=true` 且已登录：授予 Editor 权限，但排除 `SHARE_EXCLUDED_PERMISSIONS`（分享、邀请、读邮箱等敏感操作）
   - 其他情况：授予 TemplatePermissions（只读），`allowCopy=true` 时追加 `record|copy`

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

**第一步：密码认证（如有）**

`ShareAuthLocalGuard`（`share/guard/share-auth-local.guard.ts:11-26`）处理密码校验：
```typescript
async canActivate(context: ExecutionContext) {
  const req = context.switchToHttp().getRequest();
  const shareId = req.params.shareId;
  const password = req.body.password;
  const authShareId = await this.shareAuthService.authShareView(shareId, password);
  // 校验成功后签发 JWT
  const token = await this.shareAuthService.authToken({ shareId, password });
  res.cookie(shareId, token, { httpOnly: true, maxAge: 7 * 24h });
}
```

**第二步：令牌验证与用户注入**

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

**场景 3：基表分享 + allowEdit = true**
```
用户已登录 → getBaseSharePermissions() 返回 Editor 权限
→ 排除 SHARE_EXCLUDED_PERMISSIONS（禁止分享、邀请等）
→ 可编辑记录，但无法修改分享配置或邀请协作者
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

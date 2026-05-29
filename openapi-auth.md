# OpenAPI SDK 鉴权上下文处理逻辑梳理

## 1. 整体架构概览

OpenAPI SDK 的鉴权体系采用 **三层架构**：

```
┌─────────────────────────────────────────────────────────┐
│                    OpenAPI SDK 层                       │
│  axios.ts - 配置、拦截器、AsyncLocalStorage 上下文       │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              Token 校验层 (Auth Strategies)             │
│  SessionStrategy | JwtStrategy | AccessTokenStrategy    │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              权限校验层 (Guards & Services)             │
│  AuthGuard → PermissionGuard → PermissionService        │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              CLS 上下文存储 (nestjs-cls)                │
│  user | accessTokenId | spaceId | organization | ...    │
└─────────────────────────────────────────────────────────┘
```

## 2. Token 校验机制

系统支持三种认证策略，按优先级顺序执行：

### 2.1 认证策略链

**AuthGuard** 继承自 PassportAuthGuard，注册了四种认证策略：
- `session` - Session 认证（浏览器 Cookie）
- `accessToken` - Personal Access Token（API 调用）
- `jwt` - JWT Bearer Token（内部服务、临时令牌）
- `anonymous` - 匿名用户访问

源码位置：[auth.guard.ts:18-23](apps/nestjs-backend/src/features/auth/guard/auth.guard.ts#L18-L23)

```typescript
export class AuthGuard extends PassportAuthGuard([
  'session',
  ACCESS_TOKEN_STRATEGY_NAME,
  JWT_TOKEN_STRATEGY_NAME,
  ANONYMOUS_STRATEGY_NAME,
])
```

### 2.2 SessionStrategy（会话认证）

**用途**：浏览器端用户登录后的会话认证

**校验流程**：
1. 从 Cookie 中读取 Session ID
2. 通过 `SessionSerializer` 反序列化获取用户 ID
3. 查询用户信息，验证账号状态（未停用、非系统用户）
4. 将用户信息注入 CLS 上下文

源码位置：[session.strategy.ts:23-41](apps/nestjs-backend/src/features/auth/strategies/session.strategy.ts#L23-L41)

```typescript
async validate(payload: IPayloadUser) {
  const user = await this.userService.getUserById(payload.id);
  // 验证用户状态...
  this.cls.set('user.id', user.id);
  this.cls.set('user.name', user.name);
  this.cls.set('user.email', user.email);
  this.cls.set('user.isAdmin', user.isAdmin);
  return pickUserMe(user);
}
```

### 2.3 JwtStrategy（JWT 令牌认证）

**用途**：内部服务调用、自动化机器人、应用集成

**支持的令牌类型**：

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| `IJwtAuthInfo` | 用户令牌 | 普通用户临时令牌 |
| `IJwtAuthInternalInfo` | 内部令牌 | 自动化机器人、应用机器人 |

**内部令牌子类型**（通过 `type` 字段区分）：
- `JwtAuthInternalType.User` - 模拟用户操作
- `JwtAuthInternalType.Automation` - 自动化工作流机器人
- `JwtAuthInternalType.App` - 应用集成机器人

源码位置：[jwt.strategy.ts:32-102](apps/nestjs-backend/src/features/auth/strategies/jwt.strategy.ts#L32-L102)

```typescript
async validate(req: Request, payload: IJwtAuthInfo | IJwtAuthInternalInfo) {
  if ('baseId' in payload) {
    return this.validateInternalToken(payload, req);
  }
  return this.validateUserToken(payload);
}
```

**内部令牌校验逻辑**：
- 设置 `tempAuthBaseId` 到 CLS，用于后续权限校验
- 根据 `type` 字段注入不同的用户身份（真实用户或机器人用户）
- Automation 类型会额外注入 `workflowContext`

### 2.4 AccessTokenStrategy（个人访问令牌）

**用途**：OpenAPI 调用时使用的 Personal Access Token

**校验流程**：
1. 从 `Authorization: Bearer <token>` 头中提取令牌
2. 调用 `AccessTokenService.validate()` 验证令牌签名和有效性
3. 从数据库查询令牌对应的用户信息
4. 将用户信息和 `accessTokenId` 注入 CLS

源码位置：[access-token.strategy.ts:30-58](apps/nestjs-backend/src/features/auth/strategies/access-token.strategy.ts#L30-L58)

```typescript
async validate(payload: { accessTokenId: string; sign: string }) {
  const { userId, accessTokenId } = await this.accessTokenService.validate(payload);
  // 查询用户信息...
  this.cls.set('user.id', user.id);
  // ... 注入其他用户信息
  this.cls.set('accessTokenId', accessTokenId);
  return pickUserMe(user);
}
```

## 3. CLS 上下文管理

系统使用 `nestjs-cls` 库管理请求生命周期内的上下文数据。

### 3.1 中间件配置

CLS 中间件在 `GlobalModule` 中配置，对所有路由生效：

源码位置：[global.module.ts:126-134](apps/nestjs-backend/src/global/global.module.ts#L126-L134)

```typescript
configure(consumer: MiddlewareConsumer) {
  consumer
    .apply(ClsMiddleware)
    .forRoutes('*')
    .apply(SessionCsrfMiddleware)
    .forRoutes('*')
    .apply(RequestInfoMiddleware)
    .forRoutes('*');
}
```

### 3.2 CLS 存储结构

源码位置：[types/cls.ts:21-93](apps/nestjs-backend/src/types/cls.ts#L21-L93)

```typescript
export interface IClsStore extends ClsStore {
  user: {
    id: string;
    name: string;
    email: string;
    isAdmin?: boolean | null;
  };
  accessTokenId?: string;           // PAT 令牌 ID
  spaceId?: string;                 // 当前操作的空间 ID
  organization?: {                  // 组织信息（企业版）
    id: string;
    name: string;
    isAdmin: boolean;
    departments?: { id: string; name: string }[];
  };
  tempAuthBaseId?: string;          // 临时授权的 Base ID
  baseShare?: { baseId: string; nodeId: string | null }; // 分享上下文
  permissions: Action[];            // 当前用户拥有的权限列表
  // ... 其他字段
}
```

### 3.3 组织切换完整链路

#### 3.3.1 架构说明

**注意**：社区版中 `OrganizationController` 的实现是空的，组织功能主要在企业版中实现。但代码架构已完整预留了组织切换的扩展点，以下描述基于代码架构分析。

```
前端触发切换                          后端处理链路
───────────────────                  ──────────────────────────
用户点击切换组织
       │
       ▼
useOrganization Hook
       │
       ▼
getOrganizationMe() API              1. OrganizationController.getMe()
       │                                 │
       │                                 ▼
       │                              2. 查询用户所属组织列表
       │                                 │
       │                                 ▼
       │                              3. 返回当前组织信息（含部门）
       │
       ▼
更新 React Query Cache
       │
       ▼
initAxios 拦截器注入组织 Header
       │
       ▼
后续请求携带组织 Header
       │
       └───────────────────────────► 4. 中间件读取组织 Header
                                         │
                                         ▼
                                      5. 写入 CLS 上下文
                                         │
                                         ▼
                                      6. PermissionGuard 权限重算
                                         │
                                         ▼
                                      7. 注入新权限到 CLS
```

#### 3.3.2 前端触发点

**组织信息获取 Hook**：

源码位置：[use-organization.ts:5-17](packages/sdk/src/hooks/use-organization.ts#L5-L17)

```typescript
export const useOrganization = () => {
  const queryClient = useQueryClient();
  const { data: organization } = useQuery({
    queryKey: ReactQueryKeys.getOrganizationMe(),
    queryFn: () => getOrganizationMe().then((res) => res.data),
  });

  return {
    organization,
    refetch: () => {
      queryClient.invalidateQueries({ queryKey: ReactQueryKeys.getOrganizationMe() });
    },
  };
};
```

**API 接口定义**：

源码位置：[get-me.ts:36-38](packages/openapi/src/organization/get-me.ts#L36-L38)

```typescript
export const GET_ORGANIZATION_ME = '/organization/me';

export const getOrganizationMe = () => {
  return axios.get<IOrganizationMeVo>(GET_ORGANIZATION_ME);
};
```

**前端 axios 拦截器（Header 注入）**：

源码位置：[init-axios.ts:34-63](apps/nextjs-app/src/features/app/utils/init-axios.ts#L34-L63)

```typescript
export const initAxios = (options: IInitAxiosOptions = {}) => {
  if (typeof window === 'undefined') return;

  currentOptions = options;
  if (interceptorRegistered) return;

  axios.interceptors.request.use((config) => {
    const { base, shareId } = currentOptions;

    // Template preview header
    if (base?.template?.headers) {
      config.headers[IS_TEMPLATE_HEADER] = base.template.headers;
    }

    // Canary version header
    if (base?.isCanary) {
      config.headers[X_CANARY_HEADER] = 'true';
    }

    // Base share header
    if (shareId && !isUserScopedUrl(config.url)) {
      config.headers[BASE_SHARE_ID_HEADER] = shareId;
    }

    // ===== 企业版扩展：组织切换 Header =====
    // 注：社区版中 organization 为 null，此处为架构预留
    // 企业版实现示例：
    // const organization = getCurrentOrganization();
    // if (organization?.id) {
    //   config.headers['X-Tea-Org-Id'] = organization.id;
    // }

    return config;
  });

  interceptorRegistered = true;
};
```

#### 3.3.3 后端 organization 上下文写入 CLS 的入口

虽然社区版中完整的组织中间件未实现，但从代码架构可以推断出企业版的实现方式。组织信息的写入应该在 **RequestInfoMiddleware 之后、AuthGuard 之前** 的中间件中完成。

**推断的企业版中间件实现**：

```typescript
// 企业版 OrganizationMiddleware（架构推断）
@Injectable()
export class OrganizationMiddleware implements NestMiddleware {
  constructor(
    private readonly cls: ClsService<IClsStore>,
    private readonly organizationService: OrganizationService
  ) {}

  async use(req: Request, res: Response, next: NextFunction) {
    const orgId = req.headers['x-tea-org-id'] as string;
    const userId = this.cls.get('user.id'); // 已由 Session 或其他认证策略注入

    if (orgId && userId) {
      // 查询用户在该组织下的信息，包括部门列表
      const organization = await this.organizationService.getUserOrganization(
        userId,
        orgId
      );

      if (organization) {
        this.cls.set('organization', {
          id: organization.id,
          name: organization.name,
          isAdmin: organization.isAdmin,
          departments: organization.departments.map(dept => ({
            id: dept.id,
            name: dept.name,
          })),
        });
      }
    }

    next();
  }
}
```

**中间件注册顺序**：

源码位置：[global.module.ts:126-134](apps/nestjs-backend/src/global/global.module.ts#L126-L134)

```typescript
configure(consumer: MiddlewareConsumer) {
  consumer
    .apply(ClsMiddleware)                // 1. 初始化 CLS 上下文
    .forRoutes('*')
    .apply(SessionCsrfMiddleware)        // 2. Session CSRF 处理
    .forRoutes('*')
    .apply(RequestInfoMiddleware)        // 3. 注入请求信息（IP、UA 等）
    .forRoutes('*')
    // ===== 企业版扩展点 =====
    // .apply(OrganizationMiddleware)     // 4. （企业版）注入组织信息到 CLS
    // .forRoutes('*')
    ;
}
```

#### 3.3.4 组织信息的使用

组织信息写入 CLS 后，在权限计算时通过 `getDepartmentIds()` 读取：

源码位置：[permission.service.ts:55-58](apps/nestjs-backend/src/features/auth/permission.service.ts#L55-L58)

```typescript
private getDepartmentIds() {
  const departments = this.cls.get('organization.departments');
  return departments?.map((department) => department.id) || [];
}
```

**组织切换时的权限重计算**：

在 `getRoleBySpaceId` 和 `getRoleByBaseId` 中，会同时查询用户个人和所属部门的协作者关系，取最高权限角色：

源码位置：[permission.service.ts:70-116](apps/nestjs-backend/src/features/auth/permission.service.ts#L70-L116)

```typescript
async getRoleBySpaceId(spaceId: string) {
  const userId = this.cls.get('user.id');
  const departmentIds = this.getDepartmentIds();
  // 同时查询用户个人和所属部门的协作者
  const collaborators = await this.getSpaceCollaborators(spaceId, [...departmentIds, userId]);
  // 取最高权限角色
  return getMaxLevelRole(collaborators);
}
```

#### 3.3.5 切换后权限重算到 PermissionGuard 注入的关联

组织切换后，权限重算的完整链路如下：

```
1. 组织信息写入 CLS
   cls.set('organization', { id, name, isAdmin, departments })
        │
        ▼
2. PermissionGuard.canActivate() 被调用
        │
        ├─► permissionCheckWithPublicFallback()
        │     │
        │     └─► permissionCheck()
        │           │
        │           └─► resourcePermission()
        │                 │
        │                 └─► PermissionService.validPermissions()
        │                       │
        │                       ├─► getPermissionsByResourceId()
        │                       │     │
        │                       │     ├─► getRoleByBaseId(spaceId)
        │                       │     │     │
        │                       │     │     ├─► getDepartmentIds()
        │                       │     │     │    └─► 从 CLS 读取新的 departmentIds
        │                       │     │     │
        │                       │     │     └─► 查询协作者（包含新的部门ID）
        │                       │     │          └─► 返回新的角色
        │                       │     │
        │                       │     ├─► getPermissions(role)
        │                       │     │    └─► 映射为权限列表
        │                       │     │
        │                       │     └─► cls.set('spaceId', spaceId)  ← 注入 spaceId
        │                       │
        │                       ├─► getPermissionsByAccessToken()
        │                       │    └─► 令牌权限（不受组织切换影响）
        │                       │
        │                       └─► intersection(userPerms, tokenPerms) → 最终权限
        │
        └─► 3. 注入新权限到 CLS
             cls.set('permissions', ownPermissions)
```

**关键关联点代码说明**：

1. **PermissionGuard 调用 PermissionService**：

源码位置：[permission.guard.ts:378-382](apps/nestjs-backend/src/features/auth/guard/permission.guard.ts#L378-L382)

```typescript
const ownPermissions = await this.permissionService.getPermissions(
  resourceId,
  resourceType,
  permissionActions,
  cookie
);
```

2. **PermissionService 注入权限到 CLS**：

源码位置：[permission.service.ts:285-310](apps/nestjs-backend/src/features/auth/permission.service.ts#L285-L310)

```typescript
async getPermissions(resourceId: string, resourceType: string, actions: Action[], cookie?: string) {
  const { permissions: ownPermissions, ...rest } = await this.validPermissions(
    resourceId,
    resourceType,
    actions,
    cookie
  );

  // 注入权限到 CLS，供后续业务逻辑使用
  this.cls.set('permissions', ownPermissions);

  return { permissions: ownPermissions, ...rest };
}
```

3. **validPermissions 内部调用链**：

源码位置：[permission.service.ts:208-270](apps/nestjs-backend/src/features/auth/permission.service.ts#L208-L270)

```typescript
async validPermissions(resourceId: string, resourceType: string, actions: Action[]) {
  const userPermissions = await this.getPermissionsByResourceId(resourceId, resourceType);
  const accessTokenPermission = await this.getPermissionsByAccessToken();

  // 关键：用户权限与令牌权限取交集
  const ownPermissions = intersection(userPermissions, accessTokenPermission);

  // 验证所需权限是否都在最终权限列表中
  const valid = actions.every((action) => ownPermissions.includes(action));

  return { permissions: ownPermissions, valid };
}
```

#### 3.3.6 组织切换对业务逻辑的影响

组织信息不仅影响权限校验，还影响多个业务场景的数据过滤：

| 业务场景 | 代码位置 | 影响方式 |
|----------|----------|----------|
| 空间列表查询 | [space.service.ts:143](apps/nestjs-backend/src/features/space/space.service.ts#L143) | `departmentIds` 加入协作者查询范围 |
| 回收站查询 | [trash.service.ts:80](apps/nestjs-backend/src/features/trash/trash.service.ts#L80) | 部门 ID 加入 `principalId` 查询条件 |
| 最近访问记录 | [last-visit.service.ts:423](apps/nestjs-backend/src/features/user/last-visit/last-visit.service.ts#L423) | 部门 ID 用于过滤可访问的 Base |
| 邀请成员 | [invitation.service.ts:95](apps/nestjs-backend/src/features/invitation/invitation.service.ts#L95) | 验证被邀请者是否在组织域名内 |
| 表权限计算 | [table-permission.service.ts:47](apps/nestjs-backend/src/features/table/table-permission.service.ts#L47) | 部门 ID 加入协作者查询范围 |

**示例：空间列表查询**

```typescript
async getSpaceList() {
  const userId = this.cls.get('user.id');
  // 读取当前组织下的部门 ID 列表
  const departmentIds = this.cls.get('organization.departments')?.map((d) => d.id);
  // 查询用户个人 + 所属部门的所有协作者关系
  const collaboratorSpaceList = await this.prismaService.collaborator.findMany({
    where: {
      principalId: { in: [userId, ...(departmentIds || [])] },
      resourceType: CollaboratorType.Space,
    },
    // ...
  });
  // ...
}
```

### 3.4 上下文注入时机

| 阶段 | 注入内容 | 执行者 |
|------|----------|--------|
| 请求开始 | 请求 ID、追踪信息 | `ClsMiddleware` |
| 请求信息 | IP、User-Agent、是否 API 调用 | `RequestInfoMiddleware` |
| 认证阶段 | `user.*`、`accessTokenId`、`tempAuthBaseId` | 各认证 Strategy |
| 权限校验 | `spaceId`、`permissions`、`baseShare` | `PermissionService` |

## 4. 权限注入与校验流程

### 4.1 守卫执行顺序

1. **AuthGuard** - 认证守卫，验证用户身份
2. **PermissionGuard** - 权限守卫，验证用户权限

两个守卫都通过 `APP_GUARD` 全局注册。

源码位置：[global.module.ts:100-107](apps/nestjs-backend/src/global/global.module.ts#L100-L107)

### 4.2 PermissionGuard 核心流程

源码位置：[permission.guard.ts:466-490](apps/nestjs-backend/src/features/auth/guard/permission.guard.ts#L466-L490)

```
canActivate()
    │
    ├─► 检查 @Public() 装饰器 → 公开接口直接放行
    │
    ├─► 检查 @DisabledPermission() 装饰器 → 跳过权限校验
    │
    └─► permissionCheckWithPublicFallback()
          │
          ├─► 1. RESOURCE 级别认证（base share > template）
          │
          ├─► 2. Share Link 检查（有 X-Tea-Base-Share 头时）
          │
          ├─► 3. 匿名用户处理
          │
          └─► 4. 认证用户标准校验 + PUBLIC fallback
                │
                └─► permissionCheck()
                      │
                      ├─► 读取 @Permissions() 装饰的权限列表
                      ├─► 实例级权限检查（instance|update/read）
                      ├─► 特殊权限检查（space|create、base|read_all 等）
                      └─► 资源级权限检查 → resourcePermission()
                            │
                            └─► PermissionService.validPermissions()
                                  │
                                  ├─► getPermissionsByResourceId() → 用户权限
                                  ├─► getPermissionsByAccessToken() → 令牌权限
                                  ├─► 取两者交集作为最终权限
                                  └─► 校验所需权限是否都在最终权限列表中
```

### 4.3 权限计算逻辑

**用户权限计算**：
- 查询用户在 Space/Base 上的角色（包括个人和部门）
- Space 角色和 Base 角色权限取并集
- 根据角色映射到具体的 Action 权限列表

源码位置：[permission.service.ts:354-389](apps/nestjs-backend/src/features/auth/permission.service.ts#L354-L389)

```typescript
private async getPermissionByBaseId(baseId: string) {
  // 检查是否为临时授权 Base（自动化机器人）
  if (tempAuthBaseId === baseId) {
    return getPermissions('owner'); // 或 TemplatePermissions
  }
  // 同时查询 Base 和 Space 角色，取并集
  const basePermissions = role ? getPermissions(role) : [];
  const spacePermissions = spaceRole ? getPermissions(spaceRole) : [];
  return union(basePermissions, spacePermissions);
}
```

**AccessToken 权限计算**：
- 从数据库读取令牌的 `scopes`、`spaceIds`、`baseIds`
- 验证当前资源是否在令牌允许的范围内
- OAuth 客户端令牌会额外追加 `base|read_all` 权限

源码位置：[permission.service.ts:140-175](apps/nestjs-backend/src/features/auth/permission.service.ts#L140-L175)

**最终权限**：
```typescript
// 用户权限 ∩ 令牌权限 = 实际可用权限
return intersection(userPermissions, accessTokenPermission);
```

### 4.4 装饰器说明

| 装饰器 | 用途 | 示例 |
|--------|------|------|
| `@Public()` | 公开接口，无需认证 | `@Public() @Get('health')` |
| `@Permissions()` | 声明所需权限 | `@Permissions('base|read')` |
| `@ResourceMeta()` | 指定资源 ID 位置 | `@ResourceMeta('baseId', 'params')` |
| `@TokenAccess()` | 允许令牌访问无权限装饰的接口 | `@TokenAccess() @Get('user')` |
| `@AllowAnonymous()` | 允许匿名访问（分级别） | `@AllowAnonymous(AllowAnonymousType.RESOURCE)` |
| `@DisabledPermission()` | 跳过权限校验 | `@DisabledPermission()` |

## 5. OpenAPI SDK 的 axios 配置

源码位置：[packages/openapi/src/axios.ts](packages/openapi/src/axios.ts)

### 5.1 核心设计模式

SDK 使用 **Proxy + AsyncLocalStorage** 模式，实现请求上下文的自动传递：

```typescript
const axios = new Proxy(defaultAxios, {
  get(target, prop, receiver) {
    // 每次调用都动态获取当前上下文的 axios 实例
    const currentAxios = getAxios();
    const value = Reflect.get(currentAxios, prop, receiver);
    if (typeof value === 'function') {
      return value.bind(currentAxios);
    }
    return value;
  },
});
```

### 5.2 axios 实例获取优先级

```typescript
export const getAxios = (): AxiosInstance => {
  // 1. 先尝试从 AsyncLocalStorage 获取（服务端、带认证上下文）
  const storage = getAxiosStorage();
  if (storage) {
    const storedAxios = storage.getStore();
    if (storedAxios) return storedAxios;
  }
  // 2. 返回默认实例（客户端、无认证）
  return defaultAxios;
};
```

### 5.3 configApi 配置函数

用于配置 OpenAPI 调用的认证信息：

```typescript
export const configApi = (config: IAPIRequestConfig) => {
  const { token, enableUndoRedo, endpoint = 'https://app.teable.ai' } = config;
  
  defaultAxios.defaults.baseURL = `${endpoint}/api`;
  defaultAxios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
  
  if (enableUndoRedo) {
    ensureUndoRedoWindowIdHeader();
  }
  
  return axios;
};
```

### 5.4 服务端上下文传递

在服务端使用时，通过 `runWithAxios` 传递认证上下文：

```typescript
await runWithAxios(configuredAxios, async () => {
  // 此回调内的所有 @teable/openapi 调用都会使用 configuredAxios
  const records = await getRecords(tableId);
});
```

### 5.5 错误处理拦截器

```typescript
axios.interceptors.response.use(
  (response) => response,
  (error) => {
    // 网络错误特殊处理
    if (isNetworkError(error)) {
      throw new HttpError({
        message: error.message,
        code: HttpErrorCode.NETWORK_ERROR,
      }, 0);
    }
    // 其他错误统一包装为 HttpError
    throw new HttpError(data || error.message, status || 500);
  }
);
```

## 6. 完整调用流程

### 6.1 OpenAPI 调用完整链路

```
客户端调用 OpenAPI SDK
        │
        ▼
configApi() 配置 token 和 endpoint
        │
        ▼
initAxios() 注册请求拦截器（含组织 Header）
        │
        ▼
axios Proxy → getAxios() 获取实例
        │
        ▼
请求拦截器注入组织 Header（X-Tea-Org-Id）
        │
        ▼
HTTP 请求发送到后端
        │
        ▼
─────────────────────────────────────────
                后端处理
─────────────────────────────────────────
        │
        ▼
ClsMiddleware 初始化请求上下文
        │
        ▼
SessionCsrfMiddleware Session 处理
        │
        ▼
RequestInfoMiddleware 注入请求信息
        │
        ▼
OrganizationMiddleware（企业版）
        │   ├─► 从 Header 读取 X-Tea-Org-Id
        │   ├─► 查询用户组织信息（含部门列表）
        │   └─► cls.set('organization', { id, name, isAdmin, departments })
        │
        ▼
AuthGuard 认证（AccessTokenStrategy）
        │   ├─► 验证令牌签名
        │   ├─► 查询用户信息
        │   └─► 注入 user.* 和 accessTokenId 到 CLS
        │
        ▼
PermissionGuard 权限校验
        │
        ├─► 读取 @Permissions() 和 @ResourceMeta()
        │
        ├─► PermissionService.getPermissions()
        │   ├─► getPermissionsByResourceId()
        │   │   ├─► getRoleByBaseId(spaceId)
        │   │   │   ├─► getDepartmentIds()
        │   │   │   │    └─► 从 CLS 读取 organization.departments
        │   │   │   ├─► 查询协作者（用户ID + 部门ID列表）
        │   │   │   └─► 返回最高权限角色
        │   │   ├─► getPermissions(role)
        │   │   │    └─► 映射为权限列表
        │   │   └─► cls.set('spaceId', spaceId)  ← 注入 spaceId
        │   │
        │   └─► getPermissionsByAccessToken()
        │       ├─► 查询令牌的 scopes、spaceIds、baseIds
        │       ├─► 验证资源访问范围
        │       └─► 返回令牌权限列表
        │   │
        │   └─► intersection(userPerms, tokenPerms) → 最终权限
        │
        ├─► 验证所需权限 ⊆ 最终权限
        │
        └─► cls.set('permissions', ownPermissions)  ← 注入最终权限
        │
        ▼
执行业务逻辑（可从 CLS 读取上下文）
        │
        ├─► cls.get('user.id')
        ├─► cls.get('organization.departments')
        ├─► cls.get('spaceId')
        └─► cls.get('permissions')
        │
        ▼
返回响应
```

### 6.2 关键关联点说明

1. **Token 校验 → CLS 注入**：
   - 三种认证策略最终都会将用户信息写入 CLS
   - AccessToken 策略会额外写入 `accessTokenId`，影响后续权限计算
   - 注入的用户信息包括：`user.id`、`user.name`、`user.email`、`user.isAdmin`

2. **组织切换 → CLS 注入**（企业版）：
   - 前端通过 `initAxios` 拦截器在请求头中注入 `X-Tea-Org-Id`
   - 后端 `OrganizationMiddleware` 读取 Header，查询组织信息
   - 将 `organization.id`、`organization.name`、`organization.isAdmin`、`organization.departments` 写入 CLS
   - **注意**：组织中间件必须在 AuthGuard 之前执行，确保权限计算时能读取到组织信息

3. **组织切换 → 权限计算**：
   - 权限计算时通过 `getDepartmentIds()` 从 CLS 读取部门列表
   - `getRoleBySpaceId()` / `getRoleByBaseId()` 将部门 ID 加入协作者查询条件
   - SQL 查询：`principalId IN (userId, deptId1, deptId2, ...)`
   - 取最高权限角色：`getMaxLevelRole(collaborators)`
   - 切换组织后，部门列表变化，协作者查询结果变化，最终权限也随之变化

4. **权限计算 → PermissionGuard 注入**：
   - `PermissionService.validPermissions()` 计算用户权限和令牌权限的交集
   - `PermissionService.getPermissions()` 将最终权限注入 CLS：`cls.set('permissions', ownPermissions)`
   - 同时注入 `spaceId`：`cls.set('spaceId', spaceId)`
   - 后续业务逻辑可直接从 CLS 读取权限，无需重复计算

5. **CLS 上下文 → 权限校验**：
   - `PermissionService` 从 CLS 读取 `user.id`、`accessTokenId`、`organization.departments`
   - 权限计算结果写回 `cls.set('permissions', ownPermissions)`
   - 后续业务逻辑可直接从 CLS 读取权限列表

6. **OpenAPI SDK → 后端鉴权**：
   - SDK 通过 `configApi()` 设置的 `Authorization` 头传递令牌
   - 后端通过 `AccessTokenStrategy` 解析并验证
   - 服务端内部调用时通过 `runWithAxios()` 传递上下文
   - 组织切换通过 HTTP Header `X-Tea-Org-Id` 传递

## 7. 关键文件索引

| 文件路径 | 核心职责 |
|----------|----------|
| `apps/nestjs-backend/src/features/auth/guard/auth.guard.ts` | 认证守卫，策略链执行 |
| `apps/nestjs-backend/src/features/auth/guard/permission.guard.ts` | 权限守卫，多场景权限校验 |
| `apps/nestjs-backend/src/features/auth/permission.service.ts` | 权限计算核心服务 |
| `apps/nestjs-backend/src/features/auth/strategies/jwt.strategy.ts` | JWT 令牌校验 |
| `apps/nestjs-backend/src/features/auth/strategies/access-token.strategy.ts` | PAT 令牌校验 |
| `apps/nestjs-backend/src/features/auth/strategies/session.strategy.ts` | Session 认证 |
| `apps/nestjs-backend/src/types/cls.ts` | CLS 上下文类型定义 |
| `apps/nestjs-backend/src/global/global.module.ts` | 全局中间件和守卫注册 |
| `apps/nestjs-backend/src/middleware/request-info.middleware.ts` | 请求信息中间件 |
| `apps/nestjs-backend/src/features/organization/organization.controller.ts` | 组织 API 控制器 |
| `packages/openapi/src/axios.ts` | OpenAPI SDK axios 配置 |
| `packages/openapi/src/organization/get-me.ts` | 获取当前组织 API 定义 |
| `packages/sdk/src/hooks/use-organization.ts` | 前端组织信息 Hook |
| `apps/nextjs-app/src/features/app/utils/init-axios.ts` | 前端 axios 拦截器初始化 |

## 8. 扩展说明

### 8.1 社区版 vs 企业版差异

#### 8.1.1 组织功能差异

| 功能点 | 社区版 | 企业版 |
|--------|--------|--------|
| 组织管理 | 空实现，返回 null | 完整的组织 CRUD |
| 部门管理 | 代码预留，无实际功能 | 完整的部门树管理 |
| 组织切换中间件 | 未注册 | OrganizationMiddleware |
| 组织 Header 传递 | 未实现 | X-Tea-Org-Id Header |
| 数据库表 | 无 organization/department 表 | 完整的组织相关表 |
| 部门协作者 | 代码已预留逻辑，无数据 | 完整支持 |

**组织功能代码预留说明**：

虽然社区版没有完整的组织功能，但代码架构已完整预留了扩展点：

1. **CLS 类型定义**：`IClsStore.organization` 字段已定义
2. **权限计算逻辑**：`getDepartmentIds()`、部门协作者查询已实现
3. **前端 Hook**：`useOrganization`、`DepartmentSelector` 已实现
4. **API 接口**：`/organization/me`、`/organization/department` 已定义
5. **中间件扩展点**：`GlobalModule.configure()` 中预留了注册位置

#### 8.1.2 其他差异

- **应用机器人**：`JwtStrategy.setAppIdFromToken()` 在社区版是空实现，企业版支持应用认证
- **部门协作者**：虽然代码中已包含部门权限逻辑，但社区版没有完整的组织功能入口

### 8.2 特殊场景处理

1. **Base Share 分享访问**：
   - 通过 `X-Tea-Base-Share` 头识别分享场景
   - 密码验证通过 JWT Cookie 实现
   - 分享权限与用户个人权限取交集（且分享权限是上限）

2. **Template 模板访问**：
   - 通过 `X-Tea-Template` 头识别
   - 使用固定的 `TemplatePermissions` 权限集

3. **Automation 自动化**：
   - 使用 `JwtAuthInternalType.Automation` 类型令牌
   - 注入 `tempAuthBaseId` 绕过正常权限校验
   - 跳过记录审计日志（`skipRecordAuditLog: true`）

## 9. 组织切换时序与关联关系总结

### 9.1 完整时序图

```
前端                                  后端
 │                                    │
 │  1. 用户点击切换组织                │
 │    └─> useOrganization.refetch()  │
 │                                    │
 │  2. 调用 getOrganizationMe()       │
 │    └─> GET /organization/me       │
 │                                    │
 │  ◄─────────────────────────────────┤
 │    3. 返回组织信息（含部门列表）    │
 │                                    │
 │  4. 更新 React Query Cache         │
 │    └─> organization 状态更新       │
 │                                    │
 │  5. initAxios 拦截器生效           │
 │    └─> 后续请求注入 X-Tea-Org-Id   │
 │                                    │
 │  6. 发起业务请求（如查询空间）      │
 │    ┌─ Authorization: Bearer xxx   │
 │    ├─ X-Tea-Org-Id: org_xxx       │
 │    └─> GET /space/list            │
 │                                    │
 │                                    │  7. ClsMiddleware 初始化 CLS
 │                                    │  8. RequestInfoMiddleware 注入请求信息
 │                                    │  9. OrganizationMiddleware（企业版）
 │                                    │     ├─> 读取 X-Tea-Org-Id Header
 │                                    │     ├─> 查询用户组织信息
 │                                    │     └─> cls.set('organization', {...})
 │                                    │
 │                                    │  10. AuthGuard 认证
 │                                    │      └─> cls.set('user.id', userId)
 │                                    │
 │                                    │  11. PermissionGuard 权限校验
 │                                    │      ├─> getDepartmentIds()
 │                                    │      │   └─> cls.get('organization.departments')
 │                                    │      ├─> 查询协作者（用户ID + 部门ID）
 │                                    │      ├─> 计算角色权限
 │                                    │      └─> cls.set('permissions', [...])
 │                                    │
 │                                    │  12. 执行业务逻辑
 │                                    │      └─> 使用新权限查询数据
 │                                    │
 │  ◄─────────────────────────────────┤
 │    13. 返回新组织下的数据           │
 │                                    │
```

### 9.2 关键关联关系矩阵

| 触发点 | 写入 CLS 字段 | 影响的后续环节 | 代码位置 |
|--------|--------------|----------------|----------|
| 认证策略（Session/JWT/AccessToken） | `user.id`, `user.name`, `user.email`, `user.isAdmin`, `accessTokenId` | 所有权限计算、业务逻辑 | `session.strategy.ts`, `jwt.strategy.ts`, `access-token.strategy.ts` |
| OrganizationMiddleware（企业版） | `organization.id`, `organization.name`, `organization.isAdmin`, `organization.departments` | `getDepartmentIds()` → 协作者查询 → 权限计算 | `global.module.ts` 扩展点 |
| PermissionService.getPermissions() | `permissions`, `spaceId` | 后续业务逻辑的权限检查、空间过滤 | `permission.service.ts:285-310` |
| 前端 initAxios 拦截器 | 请求头 `X-Tea-Org-Id` | 后端 OrganizationMiddleware 读取 | `init-axios.ts:34-63` |

### 9.3 数据流向图

```
前端组织切换
    │
    ▼
X-Tea-Org-Id Header
    │
    ▼
┌─────────────────────────────────────────────┐
│            后端请求处理链路                   │
├─────────────────────────────────────────────┤
│  ClsMiddleware                               │
│  └─> 初始化 CLS 上下文                       │
├─────────────────────────────────────────────┤
│  RequestInfoMiddleware                       │
│  └─> 注入 IP、UA 等信息                      │
├─────────────────────────────────────────────┤
│  OrganizationMiddleware（企业版）            │
│  ├─> 读取 X-Tea-Org-Id                        │
│  ├─> 查询组织信息                             │
│  └─> cls.set('organization', {...})           │
│      ├─> organization.id                     │
│      ├─> organization.name                   │
│      ├─> organization.isAdmin                │
│      └─> organization.departments[]          │
├─────────────────────────────────────────────┤
│  AuthGuard                                   │
│  └─> cls.set('user.*', accessTokenId)        │
├─────────────────────────────────────────────┤
│  PermissionGuard                             │
│  └─> PermissionService.getPermissions()      │
│      ├─> getDepartmentIds()                  │
│      │   └─> 读取 organization.departments   │
│      ├─> getRoleByBaseId()                   │
│      │   └─> 查询协作者（用户+部门）          │
│      ├─> getPermissions(role)                │
│      ├─> intersection(userPerms, tokenPerms) │
│      └─> cls.set('permissions', [...])        │
├─────────────────────────────────────────────┤
│  业务逻辑层                                   │
│  ├─> cls.get('user.id')                      │
│  ├─> cls.get('organization.departments')     │
│  ├─> cls.get('spaceId')                      │
│  └─> cls.get('permissions')                  │
└─────────────────────────────────────────────┘
```

### 9.4 设计要点总结

1. **无状态设计**：组织信息通过 HTTP Header 传递，不依赖服务端状态，天然支持分布式部署

2. **中间件顺序**：组织中间件必须在 AuthGuard 之前执行，确保权限计算时能读取到组织信息

3. **权限重算**：每次请求都会重新计算权限，切换组织后下一次请求自然使用新的部门列表

4. **CLS 解耦**：通过 CLS 传递上下文，业务代码无需关心组织切换的实现细节，只需从 CLS 读取即可

5. **预留扩展**：社区版虽然没有完整组织功能，但所有扩展点都已预留，企业版只需实现 OrganizationMiddleware 和组织服务即可

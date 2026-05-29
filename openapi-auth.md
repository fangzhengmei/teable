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

### 3.3 组织切换链路分析

#### 3.3.1 架构说明

**重要提示**：本节内容分为「✅ 仓库现有实现」和「🔮 企业版推断」两部分，请仔细区分。

```
现有实现（社区版已存在）          企业版推断（预留扩展点）
──────────────────────────          ──────────────────────────
✅ CLS 类型定义                      🔮 OrganizationMiddleware
✅ getDepartmentIds() 方法           🔮 X-Tea-Org-Id Header 传递
✅ 部门协作者查询逻辑                 🔮 组织管理 API
✅ 前端 useOrganization Hook          🔮 部门管理功能
✅ initAxios 拦截器框架               🔮 组织数据库表
✅ /organization/me API 定义
```

#### 3.3.2 ✅ 仓库现有实现

**1. CLS 类型定义**

源码位置：[types/cls.ts:68-76](apps/nestjs-backend/src/types/cls.ts#L68-L76)

```typescript
organization?: {
  id: string;
  name: string;
  isAdmin: boolean;
  departments?: {
    id: string;
    name: string;
  }[];
};
```

**2. 权限计算中部门 ID 读取**

源码位置：[permission.service.ts:55-58](apps/nestjs-backend/src/features/auth/permission.service.ts#L55-L58)

```typescript
private getDepartmentIds() {
  const departments = this.cls.get('organization.departments');
  return departments?.map((department) => department.id) || [];
}
```

**3. 协作者查询（含部门）**

源码位置：[permission.service.ts:70-105](apps/nestjs-backend/src/features/auth/permission.service.ts#L70-L105)

```typescript
async getRoleBySpaceId(spaceId: string) {
  const userId = this.cls.get('user.id');
  const departmentIds = this.getDepartmentIds();
  // 同时查询用户个人和所属部门的协作者
  const collaborators = await this.getSpaceCollaborators(
    spaceId,
    [...departmentIds, userId]
  );
  return getMaxLevelRole(collaborators);
}
```

**4. 前端组织信息 Hook**

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

**5. API 接口定义**

源码位置：[get-me.ts:36-38](packages/openapi/src/organization/get-me.ts#L36-L38)

```typescript
export const GET_ORGANIZATION_ME = '/organization/me';

export const getOrganizationMe = () => {
  return axios.get<IOrganizationMeVo>(GET_ORGANIZATION_ME);
};
```

**6. 前端 axios 拦截器框架**

源码位置：[init-axios.ts:34-63](apps/nextjs-app/src/features/app/utils/init-axios.ts#L34-L63)

```typescript
export const initAxios = (options: IInitAxiosOptions = {}) => {
  if (typeof window === 'undefined') return;

  currentOptions = options;
  if (interceptorRegistered) return;

  axios.interceptors.request.use((config) => {
    const { base, shareId } = currentOptions;

    // 现有：Template preview header
    if (base?.template?.headers) {
      config.headers[IS_TEMPLATE_HEADER] = base.template.headers;
    }

    // 现有：Canary version header
    if (base?.isCanary) {
      config.headers[X_CANARY_HEADER] = 'true';
    }

    // 现有：Base share header
    if (shareId && !isUserScopedUrl(config.url)) {
      config.headers[BASE_SHARE_ID_HEADER] = shareId;
    }

    // 预留扩展点：组织切换 Header
    // const organization = getCurrentOrganization();
    // if (organization?.id) {
    //   config.headers['X-Tea-Org-Id'] = organization.id;
    // }

    return config;
  });

  interceptorRegistered = true;
};
```

**7. 受组织影响的业务场景（现有代码）**

| 业务场景 | 代码位置 | 实现状态 |
|----------|----------|----------|
| 空间列表查询 | [space.service.ts:143](apps/nestjs-backend/src/features/space/space.service.ts#L143) | ✅ 已实现（读取 departmentIds） |
| 回收站查询 | [trash.service.ts:80](apps/nestjs-backend/src/features/trash/trash.service.ts#L80) | ✅ 已实现（读取 departmentIds） |
| 最近访问记录 | [last-visit.service.ts:423](apps/nestjs-backend/src/features/user/last-visit/last-visit.service.ts#L423) | ✅ 已实现（读取 departmentIds） |
| 邀请成员 | [invitation.service.ts:95](apps/nestjs-backend/src/features/invitation/invitation.service.ts#L95) | ✅ 已实现（读取 departmentIds） |
| 表权限计算 | [table-permission.service.ts:47](apps/nestjs-backend/src/features/table/table-permission.service.ts#L47) | ✅ 已实现（读取 departmentIds） |

#### 3.3.3 🔮 企业版推断（预留扩展点）

以下内容为基于代码架构的推断，**社区版中未实现**。

**1. passport.initialize() 与 SessionStrategy 的职责边界**

⚠️ **关键校正**：之前的文档错误地将 `req.user` 的来源归为 `passport.initialize()`。代码证实两者职责完全不同：

| 组件 | 执行阶段 | 职责 | 是否设置 req.user |
|------|----------|------|-------------------|
| `passport.initialize()` | 中间件阶段 | 在 `req` 上挂载 `_passport` 属性，初始化 Passport 实例 | ❌ **不设置** |
| `SessionStrategy.authenticate()` | Guard 阶段（AuthGuard 内） | 从 `req.session.passport.user` 反序列化用户到 `req.user` | ✅ **设置** |

**代码证据**：

`passport.initialize()` 注册为中间件：
源码位置：[session.module.ts:19](apps/nestjs-backend/src/features/auth/session/session.module.ts#L19)
```typescript
consumer
  .apply(this.sessionHandleService.sessionMiddleware, passport.initialize())
  .forRoutes('/api/*');
```

`SessionStrategy.authenticate()` 设置 `req.user` 的逻辑：
源码位置：[session.passport.ts:30-44](apps/nestjs-backend/src/features/auth/strategies/session.passport.ts#L30-L44)
```typescript
authenticate(req: any, options?: { pauseStream?: boolean }): void {
  const user: any = req.session?.[_key]?.user;  // 从 session.passport.user 读取
  if (user) {
    _deserializeUser(user, req, function (err, user) {
      // ...
      const property = req._userProperty || 'user';
      req[property] = user;  // ← 这里才设置 req.user ✅ 代码已证实
      success(user);
    });
  } else {
    fail('No user');
  }
}
```

`SessionSerializer.deserializeUser()` 返回 `{ id: user.id }`：
源码位置：[session.serializer.ts:16-18](apps/nestjs-backend/src/features/auth/session/session.serializer.ts#L16-L18)
```typescript
async deserializeUser(payload: any, done: Function) {
  done(null, payload);  // 直接返回 { id: userId }，不做额外查询
}
```

**结论**：
- ✅ **代码已证实**：`req.user` 由 `SessionStrategy.authenticate()` 设置，不是 `passport.initialize()` 设置
- ✅ **代码已证实**：`SessionStrategy` 在 AuthGuard 内执行，属于 Guard 阶段，不在中间件阶段
- ✅ **代码已证实**：`passport.initialize()` 不反序列化用户，只初始化 Passport 上下文
- 🔮 **仅推断**：OrganizationMiddleware 在中间件阶段能从 `req.user` 获取用户 ID（因为 `req.user` 实际上也要到 Guard 阶段才设置）

**2. 中间件阶段的用户上下文边界分析**

⚠️ **核心问题**：中间件阶段**无法获取用户身份**，无论哪种认证方式。

```
中间件阶段（AuthGuard 执行前）可用的上下文：

┌────────────────────────────────────────────────────────────────┐
│ Session 认证                                                    │
├────────────────────────────────────────────────────────────────┤
│ ✅ req.session         ← session 中间件已解析                   │
│ ✅ req.sessionID       ← 可用                                  │
│ ✅ req.session.passport ← 包含 { user: { id: userId } }       │
│ ❌ req.user            ← 尚未设置（SessionStrategy 未执行）     │
│ ❌ cls.user.id         ← 尚未设置（AuthGuard 未执行）           │
│                                                                │
│ 💡 可行方案：从 req.session.passport.user.id 读取用户 ID       │
│    代码支撑：SessionHandleService.getUserId() 方法             │
│    源码：session-handle.service.ts:43-55                       │
│    该方法通过 sessionStore 直接读取 session.passport.user.id   │
├────────────────────────────────────────────────────────────────┤
│ AccessToken 认证                                               │
├────────────────────────────────────────────────────────────────┤
│ ✅ req.headers.authorization ← 可用，包含 Bearer <token>       │
│ ❌ req.user            ← 尚未设置                              │
│ ❌ cls.user.id         ← 尚未设置                              │
│                                                                │
│ 💡 可行方案：从 Authorization Header 提取 token 并验证         │
│    但这需要在中间件中复制 AccessTokenService 的验证逻辑        │
│    这会与 AuthGuard 内的 AccessTokenStrategy 产生重复          │
├────────────────────────────────────────────────────────────────┤
│ JWT 认证                                                       │
├────────────────────────────────────────────────────────────────┤
│ ✅ req.headers.authorization ← 可用，包含 Bearer <jwt>         │
│ ❌ req.user            ← 尚未设置                              │
│ ❌ cls.user.id         ← 尚未设置                              │
│                                                                │
│ 💡 可行方案：从 Authorization Header 提取 JWT 并解码           │
│    但 JWT payload 不直接包含 userId（可能含 baseId 等）        │
│    需要完整解析才能确定用户身份                                 │
├────────────────────────────────────────────────────────────────┤
│ 无认证（匿名/公开接口）                                         │
├────────────────────────────────────────────────────────────────┤
│ ❌ 无任何用户身份信息                                           │
│ ✅ X-Tea-Org-Id Header 仍可存在                               │
│ 💡 可行方案：仅注入 orgId，不查询用户关联的组织                 │
└────────────────────────────────────────────────────────────────┘
```

**3. OrganizationMiddleware 推断实现（校正版）**

基于上述分析，OrganizationMiddleware 获取用户 ID 的可行方案因认证方式而异：

```typescript
// 企业版 OrganizationMiddleware（架构推断，校正版）
@Injectable()
export class OrganizationMiddleware implements NestMiddleware {
  constructor(
    private readonly cls: ClsService<IClsStore>,
    private readonly organizationService: OrganizationService,
    private readonly sessionHandleService: SessionHandleService
  ) {}

  async use(req: Request, res: Response, next: NextFunction) {
    const orgId = req.headers['x-tea-org-id'] as string;
    if (!orgId) { next(); return; }

    // 方案 A：Session 认证 — 从 req.session.passport.user.id 读取
    // ✅ 代码已证实：session.passport.user 在中间件阶段可用
    // 代码支撑：SessionHandleService.getUserId(sessionId) 方法
    const userId = req.session?.passport?.user?.id;

    // 方案 B：AccessToken/JWT 认证 — 从 Authorization Header 提取
    // 🔮 仅推断：需要在中间件中提前解析 token
    // ⚠️ 这会与 AuthGuard 内的 Strategy 产生逻辑重复
    // const userId = await this.extractUserIdFromRequest(req);

    if (userId) {
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

**4. 中间件与 Guard 的执行时序分析（校正版）**

```
NestJS 请求处理管线（代码已证实的执行顺序）：

1. 中间件阶段
   ├── session middleware           ← ✅ 解析 Session Cookie，填充 req.session
   ├── passport.initialize()        ← ✅ 在 req 上挂载 _passport（不设置 req.user）
   ├── ClsMiddleware                ← ✅ 初始化 CLS 上下文
   ├── SessionCsrfMiddleware        ← ✅ CSRF 处理
   ├── RequestInfoMiddleware        ← ✅ 注入请求信息
   └── 🔮 OrganizationMiddleware    ← 推断：需在此阶段注入组织信息
       │
       │  中间件阶段可用的用户信息来源：
       │  ┌─ Session 认证 → ✅ req.session.passport.user.id（代码已证实）
       │  ├─ AccessToken  → 🔮 req.headers.authorization（需提前解析 token）
       │  ├─ JWT          → 🔮 req.headers.authorization（需提前解析 JWT）
       │  └─ 匿名         → ❌ 无用户信息
       │
       │  ⚠️ req.user 此时不可用（任何认证方式下都不行）
       │  ⚠️ CLS 中 user.id 不可用（AuthGuard 未执行）

2. Guard 阶段
   ├── AuthGuard                    ← ✅ 执行 Strategy 链
   │   │
   │   ├── SessionStrategy          ← ✅ 代码已证实：设置 req.user + cls.set('user.*')
   │   │   │  1. 从 req.session.passport.user 读取
   │   │   │  2. SessionSerializer.deserializeUser() 返回 { id }
   │   │   │  3. SessionStrategy.validate() 查询用户
   │   │   │  4. 设置 req[property] = user  →  req.user ✅
   │   │   │  5. cls.set('user.id', user.id) ✅
   │   │
   │   ├── AccessTokenStrategy      ← ✅ 代码已证实：设置 req.user + cls.set('user.*', 'accessTokenId')
   │   │   │  1. 从 Authorization Header 提取 Bearer token
   │   │   │  2. AccessTokenService.validate() 验证令牌
   │   │   │  3. UserService.getUserById() 查询用户
   │   │   │  4. Passport 内部设置 req.user ✅
   │   │   │  5. cls.set('user.id', user.id) ✅
   │   │
   │   └── JwtStrategy              ← ✅ 代码已证实：设置 req.user + cls.set('user.*', 'tempAuthBaseId')
   │       │  1. 从 Authorization Header 提取 JWT
   │       │  2. 验证 JWT 签名
   │       │  3. 查询用户
   │       │  4. Passport 内部设置 req.user ✅
   │       │  5. cls.set('user.id', user.id) ✅
   │
   └── PermissionGuard              ← ✅ 执行权限校验，注入 permissions 到 CLS
```

**5. 中间件注册顺序推断**

源码位置：[global.module.ts:126-134](apps/nestjs-backend/src/global/global.module.ts#L126-L134)

```typescript
configure(consumer: MiddlewareConsumer) {
  consumer
    .apply(ClsMiddleware)                // 1. ✅ 现有：初始化 CLS 上下文
    .forRoutes('*')
    .apply(SessionCsrfMiddleware)        // 2. ✅ 现有：Session CSRF 处理
    .forRoutes('*')
    .apply(RequestInfoMiddleware)        // 3. ✅ 现有：注入请求信息
    .forRoutes('*')
    // ===== 🔮 企业版扩展点 =====
    // .apply(OrganizationMiddleware)     // 4. 推断：注入组织信息到 CLS
    // .forRoutes('*')
    ;
}
```

**4. 组织切换触发点推断**

⚠️ **代码事实核查**：仓库中 `useOrganization` 的 `refetch` 方法未被任何代码调用为"组织切换触发点"。`useOrganization()` 的所有实际使用场景仅是**读取** `organization` 数据：

| 使用位置 | 用途 | 是否调用 refetch |
|----------|------|-------------------|
| [DepartmentSelector.tsx:56](packages/sdk/src/components/member-selector/DepartmentSelector.tsx#L56) | 读取 organization 判断是否有组织 | ❌ 未调用 |
| [DepartmentList.tsx:56](packages/sdk/src/components/member-selector/DepartmentList.tsx#L56) | 读取 organization 判断是否有组织 | ❌ 未调用 |
| [AccessTokenForm.tsx:50](apps/nextjs-app/src/features/app/blocks/setting/access-token/form/AccessTokenForm.tsx#L50) | 读取 organization 判断是否显示空间范围 | ❌ 未调用 |

因此，**将 `useOrganization.refetch` 视为组织切换触发点仅属推断，无直接代码支持**。企业版中组织切换的 UI 触发点和交互流程在社区版代码中不可见。

**5. 组织切换完整链路推断**

```
前端触发（企业版）                     后端处理（企业版）
───────────────────                  ──────────────────────────
🔮 用户通过组织切换 UI 操作
（具体 UI 组件在社区版中不存在）
       │
       ▼
🔮 更新当前组织状态
（实现方式不可见，可能是 Context/Store）
       │
       ▼
🔮 initAxios 拦截器注入 X-Tea-Org-Id
       │
       ▼
后续请求携带组织 Header
       │
       └───────────────────────────► OrganizationMiddleware
                                         │
                                         ▼
                                      读取 X-Tea-Org-Id
                                         │
                                         ▼
                                      从 req.user 获取 userId（⚠️ 非 CLS）
                                         │
                                         ▼
                                      查询组织信息（含部门）
                                         │
                                         ▼
                                      cls.set('organization', {...})
                                         │
                                         ▼
                                      AuthGuard 认证
                                         │
                                         ▼
                                      PermissionGuard 权限校验
                                         │
                                         ▼
                                      返回新权限
```

#### 3.3.4 ✅ 现有实现中的权限计算链路

即使在社区版中，部门权限计算逻辑也已完整实现（当 `organization.departments` 有数据时生效）：

```
1. organization.departments 已注入 CLS
   （注：社区版中此值为 undefined，企业版由 OrganizationMiddleware 注入）
        │
        ▼
2. PermissionService.getDepartmentIds()
   └─► cls.get('organization.departments')
        │
        ▼
3. PermissionService.getRoleBySpaceId(spaceId)
   ├─► userId = cls.get('user.id')
   ├─► departmentIds = getDepartmentIds()
   └─► 查询协作者：principalId IN ([...departmentIds, userId])
        │
        ▼
4. getMaxLevelRole(collaborators)
   └─► 返回最高权限角色
        │
        ▼
5. getPermissions(role)
   └─► 映射为权限列表
        │
        ▼
6. PermissionGuard.resourcePermission()
   └─► this.cls.set('permissions', ownPermissions)
```

**关键代码说明**：

源码位置：[permission.service.ts:70-105](apps/nestjs-backend/src/features/auth/permission.service.ts#L70-L105)

```typescript
async getRoleBySpaceId(spaceId: string, includeInactiveResource?: boolean) {
  const userId = this.cls.get('user.id');
  const departmentIds = this.getDepartmentIds();
  // 同时查询用户个人和所属部门的协作者
  const collaborators = await this.getSpaceCollaborators(
    spaceId,
    [...departmentIds, userId]
  );
  // ...
  return getMaxLevelRole(collaborators);
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

### 4.3 PermissionGuard 与 PermissionService 的职责分工

| 组件 | 职责 | CLS 操作 |
|------|------|----------|
| **PermissionGuard** | NestJS 守卫，拦截请求，控制流程 | **写入** `permissions` 到 CLS |
| **PermissionService** | 纯业务逻辑，计算权限 | **读取** `organization.departments`、`user.id` 等；**写入** `spaceId` |

#### 4.3.1 PermissionGuard 的核心职责

PermissionGuard 是 NestJS 的守卫，负责**控制权限校验流程**，并在权限校验通过后将权限注入 CLS。

**主要职责**：
1. 读取装饰器信息（@Public、@Permissions、@ResourceMeta 等）
2. 处理多场景权限校验流程（普通用户、分享、模板、匿名用户）
3. 协调调用 PermissionService 的方法进行实际的权限计算
4. **将最终权限注入 CLS**：`this.cls.set('permissions', ownPermissions)`

**CLS 写入点**（三处）：

1. **模板场景**：
源码位置：[permission.guard.ts:124-129](apps/nestjs-backend/src/features/auth/guard/permission.guard.ts#L124-L129)
```typescript
const ownPermissions = await this.permissionService.validTemplatePermissions(
  resourceId,
  permissions
);
this.cls.set('permissions', ownPermissions);
```

2. **分享场景**：
源码位置：[permission.guard.ts:153-168](apps/nestjs-backend/src/features/auth/guard/permission.guard.ts#L153-L168)
```typescript
const ownPermissions = await this.permissionService.validBaseSharePermissions(
  shareId,
  resourceId,
  permissions
);
this.cls.set('permissions', ownPermissions);
```

3. **常规资源场景**：
源码位置：[permission.guard.ts:188-208](apps/nestjs-backend/src/features/auth/guard/permission.guard.ts#L188-L208)
```typescript
private async resourcePermission(resourceId: string | undefined, permissions: Action[]) {
  const accessTokenId = this.cls.get('accessTokenId');
  const ownPermissions = await this.permissionService.validPermissions(
    resourceId,
    permissions,
    accessTokenId
  );
  this.cls.set('permissions', ownPermissions);  // PermissionGuard 负责注入 permissions
  return true;
}
```

#### 4.3.2 PermissionService 的核心职责

PermissionService 是**纯业务逻辑服务**，负责实际的权限计算。它从 CLS 读取上下文信息进行计算，但不负责将最终的 `permissions` 注入 CLS。

**主要职责**：
1. 从 CLS 读取上下文：`user.id`、`organization.departments`、`accessTokenId`、`tempAuthBaseId`
2. 查询用户角色（个人 + 部门）
3. 计算用户权限与令牌权限的交集
4. **注入 `spaceId` 到 CLS**（在权限计算过程中）

**CLS 读取点**：
源码位置：[permission.service.ts:55-58](apps/nestjs-backend/src/features/auth/permission.service.ts#L55-L58)
```typescript
private getDepartmentIds() {
  const departments = this.cls.get('organization.departments');  // 读取组织信息
  return departments?.map((department) => department.id) || [];
}
```

**CLS 写入点（spaceId）**：

1. Space 资源：
源码位置：[permission.service.ts:337-352](apps/nestjs-backend/src/features/auth/permission.service.ts#L337-L352)
```typescript
private async getPermissionBySpaceId(spaceId: string) {
  const role = await this.getRoleBySpaceId(spaceId);
  // ...
  this.cls.set('spaceId', spaceId);  // PermissionService 负责注入 spaceId
  return getPermissions(role);
}
```

2. Base 资源（通过 getUpperIdByBaseId）：
源码位置：[permission.service.ts:203-226](apps/nestjs-backend/src/features/auth/permission.service.ts#L203-L226)
```typescript
async getUpperIdByBaseId(baseId: string) {
  const base = await this.prismaService.base.findFirst({...});
  const spaceId = base?.spaceId;
  this.cls.set('spaceId', spaceId);  // 注入 spaceId
  return { spaceId };
}
```

3. Table 资源（通过 getUpperIdByTableId）：
源码位置：[permission.service.ts:177-201](apps/nestjs-backend/src/features/auth/permission.service.ts#L177-L201)
```typescript
async getUpperIdByTableId(tableId: string) {
  const table = await this.prismaService.txClient().tableMeta.findFirst({...});
  const spaceId = table?.base?.spaceId;
  this.cls.set('spaceId', spaceId);  // 注入 spaceId
  return { baseId, spaceId };
}
```

#### 4.3.3 准确的调用链

```
PermissionGuard.canActivate()
        │
        ▼
PermissionGuard.permissionCheck()
        │
        ├─► 特殊权限检查（space|create、base|read_all 等）
        │
        └─► PermissionGuard.resourcePermission()
              │
              ├─► PermissionService.validPermissions()
              │     │
              │     ├─► PermissionService.getPermissions()
              │     │     ├─► getPermissionsByResourceId()
              │     │     │     ├─► getRoleByBaseId()
              │     │     │     │   ├─► getDepartmentIds()  ← 读取 organization.departments
              │     │     │     │   └─► 查询协作者（用户ID + 部门ID）
              │     │     │     ├─► getPermissions(role)
              │     │     │     └─► this.cls.set('spaceId', spaceId)  ← 注入 spaceId
              │     │     │
              │     │     └─► getPermissionsByAccessToken()
              │     │
              │     └─► intersection(userPerms, tokenPerms) → 返回 ownPermissions
              │
              └─► this.cls.set('permissions', ownPermissions)  ← PermissionGuard 注入 permissions
```

### 4.4 权限计算逻辑

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

### 4.5 装饰器说明

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
initAxios() 注册请求拦截器
        │
        ▼
axios Proxy → getAxios() 获取实例
        │
        ▼
请求拦截器注入 Header（Base Share、Template 等）
        │
        ▼
HTTP 请求发送到后端
        │
        ▼
─────────────────────────────────────────
          后端处理（✅ 代码已证实 + 🔮 推断）
─────────────────────────────────────────
        │
        ▼
✅ session middleware（解析 Cookie → req.session）
        │
        ▼
✅ passport.initialize()（挂载 _passport，不设置 req.user）
        │
        ▼
✅ ClsMiddleware 初始化请求上下文
        │
        ▼
✅ SessionCsrfMiddleware Session 处理
        │
        ▼
✅ RequestInfoMiddleware 注入请求信息
        │
        ▼
🔮 OrganizationMiddleware（企业版推断）
        │   ├─► 读取 X-Tea-Org-Id Header
        │   ├─► ✅ 从 req.session.passport.user.id 获取 userId（Session 认证）
        │   │   ⚠️ 不能从 req.user 或 CLS 读取（两者此时均不可用）
        │   │   ⚠️ AccessToken/JWT 场景需提前解析 token（仅推断）
        │   ├─► 🔮 查询用户组织信息（含部门列表）
        │   └─► cls.set('organization', { id, name, isAdmin, departments })
        │
        ▼
✅ AuthGuard 认证（Strategy 链）
        │   ├─► SessionStrategy: cls.set('user.*')
        │   ├─► AccessTokenStrategy: cls.set('user.*', 'accessTokenId')
        │   └─► JwtStrategy: cls.set('user.*', 'tempAuthBaseId')
        │   此时 user.id 才写入 CLS ✅
        │
        ▼
✅ PermissionGuard 权限校验
        │
        ├─► 读取 @Permissions() 和 @ResourceMeta()
        │
        ├─► ✅ PermissionGuard.resourcePermission()
        │   │
        │   ├─► ✅ PermissionService.validPermissions()
        │   │     │
        │   │     ├─► ✅ PermissionService.getPermissions()
        │   │     │     ├─► getPermissionsByResourceId()
        │   │     │     │   ├─► getRoleByBaseId()
        │   │     │     │   │   ├─► getDepartmentIds()
        │   │     │     │   │   │   └─► cls.get('organization.departments')
        │   │     │     │   │   └─► 查询协作者（用户ID + 部门ID）
        │   │     │     │   ├─► getPermissions(role)
        │   │     │     │   └─► cls.set('spaceId', spaceId)  ← PermissionService 注入
        │   │     │     │
        │   │     │     └─► getPermissionsByAccessToken()
        │   │     │
        │   │     └─► intersection(userPerms, tokenPerms) → 返回 ownPermissions
        │   │
        │   └─► ✅ cls.set('permissions', ownPermissions)  ← PermissionGuard 注入
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

1. **Token 校验 → CLS 注入**（✅ 代码已证实）：
   - 三种认证策略最终都会将用户信息写入 CLS
   - AccessToken 策略会额外写入 `accessTokenId`，影响后续权限计算
   - 注入的用户信息包括：`user.id`、`user.name`、`user.email`、`user.isAdmin`

2. **中间件阶段 user.id 和 req.user 均不可用**（✅ 代码已证实）：
   - NestJS 中间件在 Guard 之前执行
   - `user.id` 由 AuthGuard 内的 Strategy 注入 CLS，中间件执行时 CLS 中尚无 `user.id`
   - `req.user` 由 `SessionStrategy.authenticate()` 设置（[session.passport.ts:43-44](apps/nestjs-backend/src/features/auth/strategies/session.passport.ts#L43-L44)），也在 Guard 阶段执行
   - `passport.initialize()` 仅挂载 `_passport` 属性，**不**设置 `req.user`
   - Session 认证下，中间件阶段可从 `req.session.passport.user.id` 获取用户 ID（代码支撑：[session-handle.service.ts:43-55](apps/nestjs-backend/src/features/auth/session/session-handle.service.ts#L43-L55)）
   - AccessToken/JWT 场景下，中间件阶段仅有 `req.headers.authorization`，需提前解析

3. **组织切换 → CLS 注入**（🔮 仅推断）：
   - 前端组织切换的 UI 触发点在社区版代码中不可见
   - `useOrganization.refetch` 不是组织切换触发点（仅属推断，无直接代码支持）
   - `X-Tea-Org-Id` Header 传递方式仅属推断
   - OrganizationMiddleware 实现方式仅属推断

4. **组织切换 → 权限计算**（✅ 代码已证实）：
   - 权限计算时通过 `getDepartmentIds()` 从 CLS 读取部门列表
   - `getRoleBySpaceId()` / `getRoleByBaseId()` 将部门 ID 加入协作者查询条件
   - SQL 查询：`principalId IN (userId, deptId1, deptId2, ...)`
   - 取最高权限角色：`getMaxLevelRole(collaborators)`

5. **PermissionService → CLS 注入**（✅ 代码已证实）：
   - PermissionService 负责注入 `spaceId` 到 CLS
   - 注入时机：在 `getPermissionBySpaceId()`、`getUpperIdByBaseId()`、`getUpperIdByTableId()` 中
   - PermissionService **不负责**注入 `permissions` 到 CLS

6. **PermissionGuard → CLS 注入**（✅ 代码已证实）：
   - PermissionGuard 负责注入 `permissions` 到 CLS
   - 注入时机：在 `resourcePermission()`、`templatePermissionCheck()`、`baseSharePermissionCheck()` 中
   - 调用 `PermissionService.validPermissions()` 获取权限后，由 Guard 写入 CLS

7. **CLS 上下文 → 权限校验**（✅ 代码已证实）：
   - `PermissionService` 从 CLS 读取 `user.id`、`accessTokenId`、`organization.departments`
   - 权限计算结果由 `PermissionGuard` 写回 `cls.set('permissions', ownPermissions)`
   - 后续业务逻辑可直接从 CLS 读取权限列表

8. **OpenAPI SDK → 后端鉴权**（✅ 代码已证实 + 🔮 推断）：
   - ✅ SDK 通过 `configApi()` 设置的 `Authorization` 头传递令牌
   - ✅ 后端通过 `AccessTokenStrategy` 解析并验证
   - ✅ 服务端内部调用时通过 `runWithAxios()` 传递上下文
   - 🔮 组织切换通过 HTTP Header `X-Tea-Org-Id` 传递（仅推断）

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

| 功能点 | 社区版（✅ 现有） | 企业版（🔮 推断） |
|--------|-------------------|-------------------|
| CLS 类型定义 | ✅ `organization?: {...}` | ✅ 相同 |
| `getDepartmentIds()` 方法 | ✅ 已实现 | ✅ 相同 |
| 部门协作者查询逻辑 | ✅ 已实现 | ✅ 相同 |
| 前端 `useOrganization` Hook | ✅ 已实现 | ✅ 相同 |
| `/organization/me` API 定义 | ✅ 已实现 | ✅ 相同 |
| `initAxios` 拦截器框架 | ✅ 已实现 | ✅ 相同 |
| 空间列表查询（含部门） | ✅ 已实现 | ✅ 相同 |
| 回收站查询（含部门） | ✅ 已实现 | ✅ 相同 |
| 最近访问记录（含部门） | ✅ 已实现 | ✅ 相同 |
| OrganizationMiddleware | ❌ 未注册 | 🔮 已实现 |
| `X-Tea-Org-Id` Header 传递 | ❌ 未实现 | 🔮 已实现 |
| 组织管理 API（CRUD） | ❌ 空实现 | 🔮 已实现 |
| 部门管理功能 | ❌ 未实现 | 🔮 已实现 |
| 组织数据库表 | ❌ 不存在 | 🔮 已实现 |

#### 8.1.2 PermissionGuard 与 PermissionService 职责（✅ 现有实现）

| 组件 | CLS 读取 | CLS 写入 |
|------|----------|----------|
| **PermissionGuard** | `accessTokenId` | `permissions` |
| **PermissionService** | `user.id`, `organization.departments`, `accessTokenId`, `tempAuthBaseId` | `spaceId`, `template`, `baseShare`, `baseShareNodeCache` |

**调用链（✅ 现有实现）**：
```
PermissionGuard.resourcePermission()
        │
        ├─► PermissionService.validPermissions()
        │     ├─► PermissionService.getPermissions()
        │     │   ├─► getPermissionsByResourceId()
        │     │   └─► getPermissionsByAccessToken()
        │     └─► 返回 ownPermissions
        │
        └─► this.cls.set('permissions', ownPermissions)  ← Guard 写入
```

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

### 9.1 ✅ 代码已证实的时序图（社区版）

```
前端                                  后端
 │                                    │
 │  1. ✅ 用户登录（Session/JWT）      │
 │                                    │
 │  ◄─────────────────────────────────┤
 │    2. 返回用户信息                  │
 │                                    │
 │  3. ✅ initAxios 注册拦截器         │
 │    └─> Base Share、Template Header │
 │                                    │
 │  4. ✅ 发起业务请求                 │
 │    ┌─ Authorization: Bearer xxx   │
 │    └─> GET /space/list            │
 │                                    │
 │                                    │  5. ✅ session middleware（解析 Cookie → req.session）
│                                    │  6. ✅ passport.initialize()（挂载 _passport，不设置 req.user）
│                                    │  7. ✅ ClsMiddleware 初始化 CLS
│                                    │  8. ✅ RequestInfoMiddleware 注入请求信息
│                                    │  9. ✅ AuthGuard 认证
│                                    │      ├─> SessionStrategy: 设置 req.user + cls.set('user.*')
│                                    │      └─> 此时 req.user 和 cls.user.id 才可用 ✅
 │                                    │
 │                                    │  9. ✅ PermissionGuard 权限校验
 │                                    │      ├─> PermissionGuard.resourcePermission()
 │                                    │      │   ├─> PermissionService.validPermissions()
 │                                    │      │   │   ├─> getDepartmentIds()
 │                                    │      │   │   │   └─> cls.get('organization.departments')
 │                                    │      │   │   │      └─> ✅ 返回 []（社区版为 undefined）
 │                                    │      │   │   ├─> ✅ 查询协作者（仅用户ID）
 │                                    │      │   │   ├─> getPermissions(role)
 │                                    │      │   │   └─> 返回 ownPermissions
 │                                    │      │   └─> ✅ cls.set('permissions', ownPermissions)
 │                                    │      └─> 返回 true
 │                                    │
 │                                    │  10. ✅ 执行业务逻辑
 │                                    │      └─> 使用权限查询数据
 │                                    │
 │  ◄─────────────────────────────────┤
 │    11. 返回数据                     │
 │                                    │
```

### 9.2 🔮 仅推断的企业版时序图

⚠️ 以下时序图中的组织切换触发点、X-Tea-Org-Id Header、OrganizationMiddleware 均为推断，无直接代码支持。

```
前端                                  后端
 │                                    │
 │  1. 🔮 用户通过组织切换 UI 操作     │
 │    （具体 UI 组件在社区版中不存在）  │
 │                                    │
 │  2. 🔮 更新当前组织状态             │
 │    （实现方式不可见）               │
 │                                    │
 │  3. 🔮 initAxios 拦截器注入        │
 │    └─> X-Tea-Org-Id Header        │
 │                                    │
 │  4. 🔮 发起业务请求                │
 │    ┌─ Authorization: Bearer xxx   │
 │    ├─ X-Tea-Org-Id: org_xxx       │
 │    └─> GET /space/list            │
 │                                    │
 │                                    │  5. ✅ session middleware（解析 Cookie → req.session）
│                                    │  6. ✅ passport.initialize()（挂载 _passport，不设置 req.user）
│                                    │  7. ✅ ClsMiddleware 初始化 CLS
│                                    │  8. ✅ RequestInfoMiddleware 注入请求信息
│                                    │  9. 🔮 OrganizationMiddleware
│                                    │      ├─> 读取 X-Tea-Org-Id
│                                    │      ├─> ✅ Session 认证：从 req.session.passport.user.id 获取 userId
│                                    │      │   ⚠️ req.user 此时不可用（SessionStrategy 未执行）
│                                    │      │   ⚠️ CLS 中 user.id 不可用（AuthGuard 未执行）
│                                    │      ├─> 🔮 AccessToken/JWT：从 Authorization Header 提取并解析
│                                    │      ├─> 🔮 查询组织信息（含部门）
│                                    │      └─> cls.set('organization', {...})
│                                    │
│                                    │  10. ✅ AuthGuard 认证
│                                    │       ├─> SessionStrategy: 设置 req.user + cls.set('user.*')
│                                    │       └─> 此时 req.user 和 cls.user.id 才可用 ✅
 │                                    │
 │                                    │  10. ✅ PermissionGuard 权限校验
 │                                    │       ├─> PermissionGuard.resourcePermission()
 │                                    │       │   ├─> PermissionService.validPermissions()
 │                                    │       │   │   ├─> getDepartmentIds()
 │                                    │       │   │   │   └─> cls.get('organization.departments')
 │                                    │       │   │   │      └─> 🔮 返回 [dept1, dept2, ...]
 │                                    │       │   │   ├─> 🔮 查询协作者（用户ID + 部门ID）
 │                                    │       │   │   ├─> getPermissions(role)
 │                                    │       │   │   └─> 返回 ownPermissions
 │                                    │       │   └─> ✅ cls.set('permissions', ownPermissions)
 │                                    │       └─> 返回 true
 │                                    │
 │                                    │  11. ✅ 执行业务逻辑
 │                                    │       └─> 使用新权限查询数据
 │                                    │
 │  ◄─────────────────────────────────┤
 │    12. 返回新组织下的数据           │
 │                                    │
```

### 9.3 ✅ 关键关联关系矩阵（现有实现）

| 触发点 | 读取 CLS 字段 | 写入 CLS 字段 | 代码位置 | 实现状态 |
|--------|--------------|--------------|----------|----------|
| SessionStrategy.validate() | - | `user.id`, `user.name`, `user.email`, `user.isAdmin` | [session.strategy.ts:23-41](apps/nestjs-backend/src/features/auth/strategies/session.strategy.ts#L23-L41) | ✅ |
| AccessTokenStrategy.validate() | - | `user.id`, `user.name`, `user.email`, `user.isAdmin`, `accessTokenId` | [access-token.strategy.ts:30-58](apps/nestjs-backend/src/features/auth/strategies/access-token.strategy.ts#L30-L58) | ✅ |
| JwtStrategy.validate() | - | `user.id`, `user.name`, `user.email`, `user.isAdmin`, `tempAuthBaseId` | [jwt.strategy.ts:32-102](apps/nestjs-backend/src/features/auth/strategies/jwt.strategy.ts#L32-L102) | ✅ |
| PermissionService.getDepartmentIds() | `organization.departments` | - | [permission.service.ts:55-58](apps/nestjs-backend/src/features/auth/permission.service.ts#L55-L58) | ✅ |
| PermissionService.getPermissionBySpaceId() | `user.id` | `spaceId` | [permission.service.ts:337-352](apps/nestjs-backend/src/features/auth/permission.service.ts#L337-L352) | ✅ |
| PermissionService.getUpperIdByBaseId() | - | `spaceId` | [permission.service.ts:203-226](apps/nestjs-backend/src/features/auth/permission.service.ts#L203-L226) | ✅ |
| PermissionGuard.resourcePermission() | `accessTokenId` | `permissions` | [permission.guard.ts:188-208](apps/nestjs-backend/src/features/auth/guard/permission.guard.ts#L188-L208) | ✅ |
| PermissionGuard.templatePermissionCheck() | - | `permissions` | [permission.guard.ts:90-130](apps/nestjs-backend/src/features/auth/guard/permission.guard.ts#L90-L130) | ✅ |
| PermissionGuard.baseSharePermissionCheck() | `user.id` | `permissions`, `user` | [permission.guard.ts:132-169](apps/nestjs-backend/src/features/auth/guard/permission.guard.ts#L132-L169) | ✅ |

### 9.4 🔮 仅推断的企业版关联关系

| 触发点 | 读取 | 写入 | 实现状态 | 备注 |
|--------|------|------|----------|------|
| 前端组织切换 UI | - | 当前组织状态 | 🔮 仅推断 | 社区版无此 UI 组件，`useOrganization.refetch` 未被用作切换触发点 |
| 前端 initAxios 拦截器 | 当前组织状态 | 请求头 `X-Tea-Org-Id` | 🔮 仅推断 | Header 名称和注入方式均未在代码中找到 |
| OrganizationMiddleware.use() | 请求头 `X-Tea-Org-Id`，`req.session.passport.user.id` | `organization.id`, `organization.name`, `organization.isAdmin`, `organization.departments` | 🔮 仅推断 | ⚠️ Session 认证下可从 `req.session.passport.user.id` 获取 userId；⚠️ `req.user` 在中间件阶段不可用；⚠️ AccessToken/JWT 需提前解析 token（无代码先例） |

### 9.5 ✅ 现有实现数据流向图

```
前端请求
    │
    ▼
Authorization Header
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
│  AuthGuard                                   │
│  └─> cls.set('user.*', accessTokenId)        │
├─────────────────────────────────────────────┤
│  PermissionGuard                             │
│  └─> PermissionService.validPermissions()    │
│      ├─> getDepartmentIds()                  │
│      │   └─> 读取 organization.departments   │
│      │       └─> 返回 []（社区版）            │
│      ├─> getRoleByBaseId()                   │
│      │   └─> 查询协作者（仅用户ID）           │
│      ├─> intersection(userPerms, tokenPerms) │
│      └─> 返回 ownPermissions                  │
│  └─> cls.set('permissions', ownPermissions)  │
├─────────────────────────────────────────────┤
│  业务逻辑层                                   │
│  ├─> cls.get('user.id')                      │
│  ├─> cls.get('organization.departments')     │
│  ├─> cls.get('spaceId')                      │
│  └─> cls.get('permissions')                  │
└─────────────────────────────────────────────┘
```

### 9.6 ✅ 代码已证实 / 🔮 仅推断 的关键结论

| 结论 | 标记 | 依据 |
|------|------|------|
| NestJS 中间件在 Guard 之前执行 | ✅ 代码已证实 | NestJS 框架机制，[global.module.ts:126-134](apps/nestjs-backend/src/global/global.module.ts#L126-L134) |
| `user.id` 由 AuthGuard 内的 Strategy 注入 CLS | ✅ 代码已证实 | [session.strategy.ts:36](apps/nestjs-backend/src/features/auth/strategies/session.strategy.ts#L36), [access-token.strategy.ts:52](apps/nestjs-backend/src/features/auth/strategies/access-token.strategy.ts#L52), [jwt.strategy.ts:57](apps/nestjs-backend/src/features/auth/strategies/jwt.strategy.ts#L57) |
| 中间件执行时 CLS 中尚无 `user.id` | ✅ 代码已证实 | 中间件在 Guard 之前执行，`user.id` 由 Guard 内的 Strategy 注入 |
| `passport.initialize()` 不设置 `req.user`，只挂载 `_passport` | ✅ 代码已证实 | [session.module.ts:19](apps/nestjs-backend/src/features/auth/session/session.module.ts#L19)，passport 源码确认 |
| `req.user` 由 `SessionStrategy.authenticate()` 设置 | ✅ 代码已证实 | [session.passport.ts:43-44](apps/nestjs-backend/src/features/auth/strategies/session.passport.ts#L43-L44)：`req[property] = user` |
| `SessionStrategy` 在 AuthGuard 内执行，属 Guard 阶段 | ✅ 代码已证实 | [session.strategy.ts:14](apps/nestjs-backend/src/features/auth/strategies/session.strategy.ts#L14)：`PassportStrategy(PassportSessionStrategy)` |
| 中间件阶段 `req.user` 不可用（任何认证方式） | ✅ 代码已证实 | `req.user` 由 Strategy 设置，Strategy 在 Guard 阶段执行 |
| Session 认证下，中间件阶段 `req.session.passport.user.id` 可用 | ✅ 代码已证实 | session 中间件已解析，[session-handle.service.ts:52](apps/nestjs-backend/src/features/auth/session/session-handle.service.ts#L52)：`session.passport.user.id` |
| AccessToken/JWT 场景下中间件阶段仅有 `req.headers.authorization` | ✅ 代码已证实 | 无其他用户信息来源，token 解析在 Guard 阶段的 Strategy 中进行 |
| OrganizationMiddleware 可从 `req.user` 获取 userId | 🔮 仅推断 | `req.user` 在中间件阶段不可用，此前推断有误 |
| OrganizationMiddleware 从 `req.session.passport.user.id` 获取 userId | 🔮 仅推断 | 虽然数据在中间件阶段存在，但此用法在仓库中无先例 |
| `useOrganization.refetch` 是组织切换触发点 | 🔮 仅推断 | 仓库中无任何代码调用 `refetch` 作为切换触发点，所有使用仅读取 `organization` |
| `X-Tea-Org-Id` Header 传递方式 | 🔮 仅推断 | Header 名称和注入方式均未在代码中找到 |
| OrganizationMiddleware 实现 | 🔮 仅推断 | 社区版无此中间件，企业版实现不可见 |
| 前端组织切换 UI 组件 | 🔮 仅推断 | 社区版无此 UI 组件 |

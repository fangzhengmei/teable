# OpenAPI 文档与 SDK 自动生成机制分析

## 概述

Teable 项目正在经历从 v1 到 v2 的架构演进。两个版本都采用了**以 Zod Schema 为单一数据源**的 API 定义与客户端生成机制，但在具体实现路径、责任归属和技术选型上有显著差异。本文档深入分析两个版本的接口契约、文档生成、客户端封装的接力方式，以及运行时切换机制。

## 版本架构总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              V1 架构                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  @teable/openapi (packages/openapi)                                     │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Zod Schema + RouteConfig + Axios 函数 (手动编写)                  │  │
│  │  路径风格: /table/{tableId}/record/{recordId} (RESTful)            │  │
│  └───────────────────────────────────┬───────────────────────────────┘  │
│                                      │                                  │
│              ┌───────────────────────┴───────────────────────┐          │
│              ▼                                               ▼          │
│  NestJS Controller (@UseV2Feature)                @teable/sdk           │
│  - ZodValidationPipe 验证                              - React Hooks    │
│  - 手动路由定义                                        - React Query    │
│  - Service 调用 v1 或 v2 实现                               │          │
│                                                             ▼          │
│                                                      UI 组件调用        │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                              V2 架构                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  @teable/v2-contract-http (packages/v2/contract-http)                   │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  @orpc/contract 定义 (oc.route + Zod Schema)                      │  │
│  │  路径风格: /tables/createRecord (动作式 RPC)                       │  │
│  │  统一响应格式: { ok: true, data: T } | { ok: false, error: E }    │  │
│  └───────────────────────────────────┬───────────────────────────────┘  │
│                                      │                                  │
│              ┌───────────────────────┴───────────────────────┐          │
│              ▼                                               ▼          │
│  @teable/v2-contract-http-*                         @teable/v2-contract-http-client │
│  - express / fastify / hono 适配器                    - @orpc/client 自动生成     │
│  - 自动路由绑定                                          - 类型安全客户端           │
│  - executeXxxEndpoint 处理器                                                │
│              │                                                               │
│              ▼                                                               ▼
│  @teable/v2-core (命令总线 + 领域模型)                          前端/外部调用方        │
└─────────────────────────────────────────────────────────────────────────┘
```

## 第一部分：V1 实现机制详解

### 1.1 接口契约定义

**包**: `@teable/openapi` (packages/openapi)

#### 核心技术栈
- **Zod**: TypeScript 优先的 schema 验证库
- **@asteasolutions/zod-to-openapi**: Zod Schema → OpenAPI 3.0 转换
- **Axios**: HTTP 客户端（手动封装）

#### 目录结构
```
packages/openapi/src/
├── zod.ts                      # 扩展 OpenAPI 支持的 Zod
├── utils.ts                    # 路由注册与 URL 构建
├── generate.schema.ts          # OpenAPI 文档生成器
├── axios.ts                    # Axios 实例配置
├── types.ts                    # 通用类型
└── [feature]/                  # 按功能模块划分
    ├── index.ts
    ├── get.ts                  # 每个操作一个文件
    ├── create.ts
    ├── update.ts
    └── ...
```

#### API 定义四要素（以 getRecord 为例）

[packages/openapi/src/record/get.ts](packages/openapi/src/record/get.ts)

```typescript
// 1. Zod Schema 定义（请求参数）
export const getRecordQuerySchema = z.object({
  projection: z.union([z.string(), z.string().array()])
    .transform((val) => typeof val === 'string' ? [val] : val)
    .optional()
    .meta({
      type: 'array',
      items: { type: 'string' },
      description: '字段投影...'
    }),
  fieldKeyType: fieldKeyTypeRoSchema,
});

export type IGetRecordQuery = z.infer<typeof getRecordQuerySchema>;

// 2. URL 常量 (RESTful 风格)
export const GET_RECORD_URL = '/table/{tableId}/record/{recordId}';

// 3. RouteConfig 注册（用于 OpenAPI 文档生成）
export const GetRecordRoute: RouteConfig = registerRoute({
  method: 'get',
  path: GET_RECORD_URL,
  summary: 'Get record',
  description: 'Retrieve a single record...',
  request: {
    params: z.object({
      tableId: z.string(),
      recordId: z.string(),
    }),
    query: getRecordQuerySchema,
  },
  responses: {
    200: {
      description: 'Success',
      content: {
        'application/json': {
          schema: recordSchema,
        },
      },
    },
  },
  tags: ['record'],
});

// 4. Axios 客户端函数（手动编写）
export async function getRecord(
  tableId: string,
  recordId: string,
  query?: IGetRecordQuery
): Promise<AxiosResponse<IRecord>> {
  return axios.get<IRecord>(
    urlBuilder(GET_RECORD_URL, { tableId, recordId }), 
    { params: query }
  );
}
```

**责任归属**:
- ✅ Schema 定义: `@teable/openapi`
- ✅ 路由元数据: `@teable/openapi`
- ✅ 客户端函数: `@teable/openapi` (手动编写)
- ❌ 服务端路由: 不负责，由 NestJS Controller 手动定义

### 1.2 文档生成

[packages/openapi/src/generate.schema.ts](packages/openapi/src/generate.schema.ts)

```typescript
import { OpenAPIRegistry, OpenApiGeneratorV3 } from '@asteasolutions/zod-to-openapi';
import { getRoutes } from './utils';

export async function getOpenApiDocumentation(config: {
  origin?: string;
  snippet?: boolean;
}): Promise<OpenAPIObject> {
  const registry = new OpenAPIRegistry();
  const routeObjList: RouteConfig[] = getRoutes();

  // 注册所有路由到 OpenAPI 注册表
  for (const routeObj of routeObjList) {
    const bearerAuth = registry.registerComponent('securitySchemes', 'bearerAuth', {
      type: 'http',
      scheme: 'bearer',
    });
    registry.registerPath({ ...routeObj, security: [{ [bearerAuth.name]: [] }] });
  }

  // 生成 OpenAPI 3.0 文档
  const generator = new OpenApiGeneratorV3(registry.definitions);
  return generator.generateDocument({
    openapi: '3.0.0',
    info: {
      version: '1.0.0',
      title: 'Teable App',
      description: 'Manage Data as easy as drink a cup of tea',
    },
    servers: [{ url: origin + '/api' }],
  });
}
```

**后端接入点**: [apps/nestjs-backend/src/swagger.ts](apps/nestjs-backend/src/swagger.ts)

```typescript
export async function setupSwagger(app: INestApplication, publicOrigin: string, enabledSnippet: boolean) {
  const openApiDocumentation = await getOpenApiDocumentation({
    origin: publicOrigin,
    snippet: enabledSnippet,
  });

  // 写入 JSON 文件
  fs.writeFileSync(path.join(__dirname, '/openapi.json'), JSON.stringify(openApiDocumentation));
  
  // Swagger UI: /docs
  SwaggerModule.setup('/docs', app, openApiDocumentation as OpenAPIObject);
  
  // Redoc UI: /redocs
  await RedocModule.setup('/redocs', app, openApiDocumentation as OpenAPIObject, redocOptions);
}
```

### 1.3 客户端类型/调用封装

#### Axios 实例配置

[packages/openapi/src/axios.ts](packages/openapi/src/axios.ts)

```typescript
export const createAxios = () => {
  const axios = axiosInstance.create({
    baseURL: '/api',
  });

  axios.interceptors.response.use(
    (response) => response,
    (error) => {
      const { data, status } = error?.response || {};
      throw new HttpError(data || error?.message || 'no response from server', status || 500);
    }
  );
  return axios;
};

// 支持服务端 AsyncLocalStorage 的代理 axios
const axios = new Proxy(defaultAxios, {
  get(target, prop, receiver) {
    const currentAxios = getAxios(); // 从 AsyncLocalStorage 获取或使用默认
    const value = Reflect.get(currentAxios, prop, receiver);
    return typeof value === 'function' ? value.bind(currentAxios) : value;
  },
}) as AxiosInstance;
```

#### SDK 层封装

[packages/sdk/src/hooks/use-records-query.ts](packages/sdk/src/hooks/use-records-query.ts)

```typescript
import { useQuery } from '@tanstack/react-query';
import type { IGetRecordsRo, IRecordsVo } from '@teable/openapi';
import { getRecords } from '@teable/openapi';
import { createRecordInstance } from '../model';

export const useRecordsQuery = (query?: IGetRecordsRo, enabled = true) => {
  const tableId = useTableId();
  const viewId = useViewId();
  
  const queryParams = useMemo(() => ({
    viewId,
    fieldKeyType: FieldKeyType.Id,
    ...query,
  }), [query, viewId]);

  const { data, isLoading } = useQuery({
    queryKey: ReactQueryKeys.linkEditorRecords(tableId!, queryParams),
    queryFn: () => getRecords(tableId!, queryParams).then(({ data }) => data),
    enabled: Boolean(tableId && enabled),
  });

  const records = (data?.records ?? []).map((record: IRecord) => {
    const instance = createRecordInstance(record);
    instance.getCellValue = (fieldId: string) => record.fields[fieldId];
    return instance;
  });

  return { records, extra: data?.extra, isLoading };
};
```

**责任归属**:
- ✅ HTTP 客户端: `@teable/openapi` (Axios)
- ✅ React Query 集成: `@teable/sdk` (手动封装)
- ✅ 领域模型包装: `@teable/sdk` (手动编写)

### 1.4 V1 后端接入

[apps/nestjs-backend/src/features/record/open-api/record-open-api.controller.ts](apps/nestjs-backend/src/features/record/open-api/record-open-api.controller.ts)

```typescript
import { getRecordQuerySchema } from '@teable/openapi';
import type { IGetRecordQuery, IRecord } from '@teable/openapi';
import { ZodValidationPipe } from '../../../zod.validation.pipe';

@Controller('api/table/:tableId/record')
export class RecordOpenApiController {
  constructor(
    private readonly recordService: RecordService,
    private readonly recordOpenApiService: RecordOpenApiService,
    private readonly recordOpenApiV2Service: RecordOpenApiV2Service  // v2 服务
  ) {}

  @UseV2Feature('getRecords')
  @Permissions('record|read')
  @Get()
  async getRecords(
    @Param('tableId') tableId: string,
    @Query(new ZodValidationPipe(getRecordsRoSchema), TqlPipe, FieldKeyPipe) query: IGetRecordsRo
  ): Promise<IRecordsVo> {
    // 运行时决定使用 v1 还是 v2 实现
    if (this.cls.get('useV2')) {
      return this.recordOpenApiV2Service.getRecords(tableId, query);
    }
    return await this.recordService.getRecords(tableId, query, true);
  }
}
```

## 第二部分：V2 实现机制详解

### 2.1 接口契约定义

**包**: `@teable/v2-contract-http` (packages/v2/contract-http)

#### 核心技术栈
- **@orpc/contract**: 类型安全的 RPC 契约定义框架
- **Zod**: Schema 验证（与 v1 共用）
- **neverthrow**: Result 类型处理（`Ok<T> | Err<E>`）

#### 目录结构
```
packages/v2/contract-http/src/
├── contract.ts                 # 总契约定义
├── index.ts                    # 导出
├── shared/
│   ├── http.ts                 # 统一响应格式
│   ├── domainEvent.ts          # 领域事件 DTO
│   └── container.ts            # DI 容器接口
├── base/
│   ├── createBase.ts
│   └── listBases.ts
└── table/
    ├── createTable.ts
    ├── createRecord.ts
    ├── updateRecord.ts
    ├── deleteRecords.ts
    ├── dto.ts
    ├── recordDto.ts
    └── ...
```

#### API 定义方式（以 createRecord 为例）

**1. 输入 Schema 定义** (来自 `@teable/v2-core`)

[packages/v2/core/src/commands/CreateRecordCommand.ts](packages/v2/core/src/commands/CreateRecordCommand.ts)

```typescript
import { z } from 'zod';

export const createRecordInputSchema = z.object({
  tableId: z.string(),
  fields: z.record(z.string(), z.unknown()),
  fieldKeyType: z.nativeEnum(FieldKeyType).optional(),
  typecast: z.boolean().optional(),
  order: z.object({
    viewId: z.string().optional(),
    anchorId: z.string().optional(),
    position: z.enum(['before', 'after']).optional(),
  }).optional(),
});

export type ICreateRecordCommandInput = z.infer<typeof createRecordInputSchema>;
```

**2. 响应 DTO 定义**

[packages/v2/contract-http/src/table/createRecord.ts](packages/v2/contract-http/src/table/createRecord.ts)

```typescript
import { z } from 'zod';
import { ok } from 'neverthrow';
import { apiOkResponseDtoSchema, type IApiOkResponseDto } from '../shared/http';
import { tableRecordDtoSchema, type ITableRecordDto } from './recordDto';

export interface ICreateRecordResponseDataDto {
  record: ITableRecordDto;
  events: Array<IDomainEventDto>;
}

export const createRecordResponseDataSchema = z.object({
  record: tableRecordDtoSchema,
  events: z.array(domainEventDtoSchema),
});

export const createRecordOkResponseSchema = apiOkResponseDtoSchema(createRecordResponseDataSchema);

// Domain → DTO 转换
export const mapCreateRecordResultToDto = (
  result: CreateRecordResult
): Result<ICreateRecordResponseDataDto, DomainError> => {
  return ok({
    record: {
      id: result.record.id().toString(),
      fields: Object.fromEntries(
        result.record.fields().entries().map(([fieldId, value]) => 
          [fieldId.toString(), value.toValue()]
        )
      ),
    },
    events: result.events.map(mapDomainEventToDto),
  });
};
```

**3. 契约注册**

[packages/v2/contract-http/src/contract.ts](packages/v2/contract-http/src/contract.ts)

```typescript
import { oc } from '@orpc/contract';
import { createRecordInputSchema } from '@teable/v2-core';
import { createRecordOkResponseSchema } from './table/createRecord';

const TABLES_CREATE_RECORD_PATH = '/tables/createRecord';

export const v2Contract = {
  tables: {
    createRecord: oc
      .route({
        method: 'POST',
        path: TABLES_CREATE_RECORD_PATH,
        successStatus: 201,
        summary: 'Create record',
        tags: ['tables'],
      })
      .input(createRecordInputSchema)
      .output(createRecordOkResponseSchema),
    // ... 更多路由
  },
} as const satisfies AnyContractRouter;
```

**统一响应格式**:

[packages/v2/contract-http/src/shared/http.ts](packages/v2/contract-http/src/shared/http.ts)

```typescript
export interface IApiOkResponseDto<T> {
  ok: true;
  data: T;
}

export interface IApiErrorResponseDto {
  ok: false;
  error: {
    code: string;
    message: string;
    tags: ReadonlyArray<string>;
    details?: Readonly<Record<string, unknown>>;
  };
}

export type IApiResponseDto<T> = IApiOkResponseDto<T> | IApiErrorResponseDto;
```

**责任归属**:
- ✅ Schema 定义: `@teable/v2-core` (领域层)
- ✅ 契约定义: `@teable/v2-contract-http`
- ✅ DTO 转换: `@teable/v2-contract-http`
- ✅ 服务端路由: 由适配器自动生成
- ✅ 客户端类型: 由 `@orpc/client` 自动推导

### 2.2 文档生成

[packages/v2/contract-http-openapi/src/generate.ts](packages/v2/contract-http-openapi/src/generate.ts)

```typescript
import { OpenAPIGenerator } from '@orpc/openapi';
import { v2Contract } from '@teable/v2-contract-http';

export const generateV2OpenApiDocument = async (options: IV2OpenApiGenerateOptions = {}) => {
  const openAPIGenerator = new OpenAPIGenerator();

  return openAPIGenerator.generate(v2Contract, {
    info: {
      title: options.title ?? 'Teable v2 API',
      version: options.version ?? '0.0.0',
    },
    servers: options.servers,
    customErrorResponseBodySchema: () => ({
      type: 'object',
      properties: {
        ok: { const: false },
        error: { type: 'string' },
      },
      required: ['ok', 'error'],
    }),
  });
};
```

**后端接入点**: [apps/nestjs-backend/src/features/v2/v2-openapi.controller.ts](apps/nestjs-backend/src/features/v2/v2-openapi.controller.ts)

```typescript
@Controller('api/v2')
export class V2OpenApiController {
  @Get('openapi.json')
  async openapi(@Req() req: Request) {
    return generateV2OpenApiDocument({
      servers: [{ url: `${serverUrl}/api/v2` }],
    });
  }

  @Get('docs')
  docs(@Res() res: Response) {
    // 使用 Scalar API Reference UI
    return buildScalarHtml('/api/v2/openapi.json', nonce);
  }
}
```

**V2 文档访问路径**:
- OpenAPI JSON: `/api/v2/openapi.json`
- Scalar UI: `/api/v2/docs`

### 2.3 客户端类型/调用封装

**包**: `@teable/v2-contract-http-client` (packages/v2/contract-http-client)

[packages/v2/contract-http-client/src/index.ts](packages/v2/contract-http-client/src/index.ts)

```typescript
import { createORPCClient, createORPCErrorFromJson, isORPCErrorJson } from '@orpc/client';
import { OpenAPILink } from '@orpc/openapi-client/fetch';
import { v2Contract, apiErrorResponseDtoSchema } from '@teable/v2-contract-http';

export const createV2HttpClient = (
  options: IV2HttpClientOptions
): ContractRouterClient<typeof v2Contract> => {
  const link = new OpenAPILink(v2Contract, {
    url: options.baseUrl,
    headers: options.headers,
    fetch: options.fetch,
    customErrorResponseBodyDecoder: (body, response) => {
      // 自定义错误解码
      if (isORPCErrorJson(body)) {
        return createORPCErrorFromJson(body);
      }
      const parsedError = apiErrorResponseDtoSchema.safeParse(body);
      if (parsedError.success) {
        return createORPCErrorFromJson({
          defined: false,
          code: inferErrorCode(response.status),
          status: response.status,
          message: parsedError.data.error.message,
          data: {
            domainErrorCode: parsedError.data.error.code,
            tags: parsedError.data.error.tags,
            details: parsedError.data.error.details,
          },
        });
      }
      return null;
    },
  });

  return createORPCClient(link) as ContractRouterClient<typeof v2Contract>;
};
```

**使用示例**:

```typescript
import { createV2HttpClient } from '@teable/v2-contract-http-client';

const client = createV2HttpClient({
  baseUrl: 'https://app.teable.ai/api/v2',
  headers: { Authorization: 'Bearer <token>' },
});

// 完全类型安全的调用
const result = await client.tables.createRecord({
  tableId: 'tblxxx',
  fields: {
    name: 'Hello',
    age: 25,
  },
  fieldKeyType: 'name',
});

// result 类型: { ok: true; data: { record: ...; events: ... } }
// 或抛出 ORPCError
```

**责任归属**:
- ✅ 客户端生成: `@orpc/client` (自动)
- ✅ 类型安全: 由 `v2Contract` 类型推导
- ✅ 错误处理: `@teable/v2-contract-http-client` (自定义解码)

### 2.4 V2 服务端实现

**架构分层**:

```
@teable/v2-contract-http (契约)
        ↓
@teable/v2-contract-http-implementation (端点处理器)
        ↓
@teable/v2-contract-http-express / fastify / hono (HTTP 适配器)
        ↓
@teable/v2-core (命令总线 + 领域模型)
```

**端点处理器示例**:

[packages/v2/contract-http-implementation/src/handlers/tables/createRecord.ts](packages/v2/contract-http-implementation/src/handlers/tables/createRecord.ts)

```typescript
import type { IExecutionContext, ICommandBus } from '@teable/v2-core';
import { CreateRecordCommand } from '@teable/v2-core';
import { mapCreateRecordResultToDto } from '@teable/v2-contract-http';

export const executeCreateRecordEndpoint = async (
  context: IExecutionContext,
  input: ICreateRecordCommandInput,
  commandBus: ICommandBus
): Promise<ICreateRecordEndpointResult> => {
  const commandResult = CreateRecordCommand.create(input);
  
  if (commandResult.isErr()) {
    return {
      status: mapDomainErrorToHttpStatus(commandResult.error),
      body: {
        ok: false,
        error: mapDomainErrorToHttpError(commandResult.error),
      },
    };
  }

  const result = await commandBus.execute<CreateRecordCommand, CreateRecordResult>(
    context,
    commandResult.value
  );

  if (result.isErr()) {
    return {
      status: mapDomainErrorToHttpStatus(result.error),
      body: {
        ok: false,
        error: mapDomainErrorToHttpError(result.error),
      },
    };
  }

  const dto = mapCreateRecordResultToDto(result.value);

  return {
    status: 201,
    body: { ok: true, data: dto.value },
  };
};
```

**NestJS 集成**:

[apps/nestjs-backend/src/features/v2/v2.controller.ts](apps/nestjs-backend/src/features/v2/v2.controller.ts)

```typescript
import { Controller } from '@nestjs/common';
import { Implement, implement, ORPCError } from '@orpc/nest';
import { v2Contract } from '@teable/v2-contract-http';
import { executeCreateRecordEndpoint } from '@teable/v2-contract-http-implementation/handlers';

@Controller('api/v2')
export class V2Controller {
  @Implement(v2Contract.tables)
  tables() {
    return {
      createRecord: implement(v2Contract.tables.createRecord).handler(async ({ input }) => {
        const container = await this.v2Container.getContainer();
        const commandBus = container.resolve<ICommandBus>(v2CoreTokens.commandBus);
        const context = await this.v2ContextFactory.createContext();

        const result = await executeCreateRecordEndpoint(context, input, commandBus);

        if (result.status === 201) return result.body;
        throwOrpcErrorByStatus(result.status, result.body.error);
      }),
      // ... 更多端点
    };
  }
}
```

## 第三部分：V1 与 V2 责任归属对比

| 职责 | V1 | V2 |
|------|----|----|
| **Schema 定义** | `@teable/openapi` (与 HTTP 绑定) | `@teable/v2-core` (纯领域，与传输无关) |
| **接口契约** | `@teable/openapi` RouteConfig | `@teable/v2-contract-http` @orpc/contract |
| **文档生成** | `@teable/openapi` + `@asteasolutions/zod-to-openapi` | `@teable/v2-contract-http-openapi` + `@orpc/openapi` |
| **服务端路由** | NestJS Controller 手动定义 | `@orpc/nest` 自动绑定 + 适配器 |
| **服务端验证** | ZodValidationPipe 手动应用 | 由 @orpc 自动处理 |
| **客户端函数** | `@teable/openapi` 手动编写 Axios 函数 | `@orpc/client` 自动生成类型安全客户端 |
| **响应格式** | 自由格式 (直接返回数据) | 统一 `{ ok: true, data: T }` / `{ ok: false, error: E }` |
| **路径风格** | RESTful `/table/{tableId}/record/{recordId}` | RPC 动作式 `/tables/createRecord` |
| **错误处理** | HttpException 抛异常 | neverthrow Result + ORPCError |

## 第四部分：运行时 V2 切换机制与文档对齐

### 4.1 Canary 发布系统

**核心组件**:

[apps/nestjs-backend/src/features/canary/canary.service.ts](apps/nestjs-backend/src/features/canary/canary.service.ts)

```typescript
@Injectable()
export class CanaryService {
  async shouldUseV2WithReason(spaceId: string, feature: V2Feature): Promise<IV2Decision> {
    // 优先级从高到低:
    // 1. FORCE_V2_ALL 环境变量
    if (this.isForceV2AllEnabled()) {
      return { useV2: true, reason: 'env_force_v2_all' };
    }

    // 2. 全局 canary 开关
    if (!this.isCanaryFeatureEnabled()) {
      return { useV2: false, reason: 'disabled' };
    }

    // 3. 数据库配置 forceV2All
    const config = await this.getCanaryConfig();
    if (config?.forceV2All) {
      return { useV2: true, reason: 'config_force_v2_all' };
    }

    // 4. 请求头 x-canary 覆盖
    const headerOverride = this.getHeaderCanaryOverride();
    if (headerOverride !== undefined) {
      return { useV2: headerOverride, reason: 'header_override' };
    }

    // 5. 空间 ID 在 canary 列表中
    if (config?.spaceIds?.includes(spaceId)) {
      return { useV2: true, reason: 'space_feature' };
    }

    return { useV2: false, reason: 'feature_not_enabled' };
  }
}
```

### 4.2 V2FeatureGuard 工作流程

[apps/nestjs-backend/src/features/canary/guards/v2-feature.guard.ts](apps/nestjs-backend/src/features/canary/guards/v2-feature.guard.ts)

```typescript
@Injectable()
export class V2FeatureGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const req = context.switchToHttp().getRequest();
    
    // 1. 从装饰器获取 feature 名称
    const feature = this.reflector.getAllAndOverride<V2Feature>(USE_V2_FEATURE_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!feature) {
      this.cls.set('useV2', false);
      return true;
    }

    // 2. 解析 base 上下文 (spaceId, v2Enabled 标记)
    const base = await this.getBaseV2DecisionContext(context);
    
    // 3. 决策是否使用 v2
    const decision = await this.canaryService.shouldUseV2ForBaseWithReason(base, feature);
    
    // 4. 存储到 CLS 供 Controller 使用
    this.cls.set('useV2', decision.useV2);
    this.cls.set('v2Feature', feature);
    this.cls.set('v2Reason', decision.reason);

    return true;
  }
}
```

### 4.3 Controller 中的分支逻辑

[apps/nestjs-backend/src/features/record/open-api/record-open-api.controller.ts](apps/nestjs-backend/src/features/record/open-api/record-open-api.controller.ts)

```typescript
@UseGuards(V2FeatureGuard)
@UseInterceptors(V2IndicatorInterceptor)
@Controller('api/table/:tableId/record')
export class RecordOpenApiController {
  @UseV2Feature('createRecord')
  @Permissions('record|create')
  @Post()
  async createRecords(
    @Param('tableId') tableId: string,
    @Body(new ZodValidationPipe(createRecordsRoSchema)) createRecordsRo: ICreateRecordsRo,
  ): Promise<ICreateRecordsVo> {
    // 运行时分支
    if (this.cls.get('useV2')) {
      return this.recordOpenApiV2Service.createRecords(tableId, createRecordsRo);
    }
    return this.recordOpenApiService.multipleCreateRecords(tableId, createRecordsRo);
  }
}
```

### 4.4 文档与实际行为的对齐情况

#### ✅ 对齐的部分

1. **V1 文档与 V1 行为**: 完全对齐
   - 文档从 `@teable/openapi` 生成
   - 后端 Controller 路径与 RouteConfig 定义一致
   - 请求/响应格式匹配

2. **V2 独立文档与 V2 行为**: 完全对齐
   - `/api/v2/openapi.json` 从 `v2Contract` 生成
   - 路径 `/api/v2/tables/createRecord` 与契约定义一致
   - 统一响应格式 `{ ok: true, data: T }`

#### ⚠️ 不完全对齐的部分

1. **V1 文档中的 V2 行为**: 文档显示 V1 格式，但实际可能返回 V2 数据
   - 当 `useV2=true` 时，`GET /api/table/{tableId}/record` 实际调用 V2 服务
   - 但 Swagger 文档 (`/docs`) 仍然显示 V1 的响应 schema
   - **注意**: V2Service 会进行数据格式转换以保持 V1 兼容性

2. **V1 路径与 V2 路径并存**:
   - V1: `GET /api/table/{tableId}/record/{recordId}`
   - V2: `POST /api/v2/tables/getRecord` (body: { tableId, recordId })
   - 两者功能相同但调用方式不同

3. **v2Indicator 响应头**:
   - `V2IndicatorInterceptor` 会在响应头中添加 `x-teable-v2: true` 当使用 V2 实现时
   - 可用于调试和监控

### 4.5 V2 服务中的格式适配

[apps/nestjs-backend/src/features/record/open-api/record-open-api-v2.service.ts](apps/nestjs-backend/src/features/record/open-api/record-open-api-v2.service.ts)

```typescript
@Injectable()
export class RecordOpenApiV2Service {
  async createRecords(
    tableId: string,
    createRecordsRo: ICreateRecordsRo,  // V1 输入格式
  ): Promise<ICreateRecordsVo> {       // V1 输出格式
    const container = await this.v2ContainerService.getContainer();
    const commandBus = container.resolve<ICommandBus>(v2CoreTokens.commandBus);
    const context = await this.v2ContextFactory.createContext();

    // 调用 V2 端点处理器
    const result = await executeCreateRecordsEndpoint(
      context,
      {
        tableId,
        records: createRecordsRo.records,
        typecast: createRecordsRo.typecast ?? false,
        fieldKeyType: createRecordsRo.fieldKeyType,
        order: createRecordsRo.order,
      },
      commandBus
    );

    if (result.status === 201 && result.body.ok) {
      // V2 → V1 格式转换
      return {
        records: result.body.data.records as IRecord[],
      };
    }

    this.throwV2Error(result.body.error, result.status);
  }
}
```

## 第五部分：完整数据流对比

### V1 数据流

```
1. API 定义 (开发时)
   @teable/openapi/src/record/get.ts
   ├─ Zod Schema: getRecordQuerySchema
   ├─ URL: GET_RECORD_URL = '/table/{tableId}/record/{recordId}'
   ├─ RouteConfig: GetRecordRoute (注册到全局数组)
   └─ Axios 函数: getRecord()

2. 文档生成 (启动时)
   setupSwagger()
   └─ getOpenApiDocumentation()
      ├─ getRoutes() → 所有已注册的 RouteConfig
      └─ @asteasolutions/zod-to-openapi → openapi.json
         └─ 挂载到 /docs, /redocs

3. 请求处理 (运行时)
   HTTP GET /api/table/tblxxx/record/recxxx
   ├─ NestJS 路由匹配 → RecordOpenApiController.getRecord()
   ├─ ZodValidationPipe(getRecordQuerySchema) 验证 query
   ├─ V2FeatureGuard 决策 useV2
   ├─ 分支: useV2 ? v2Service.getRecord() : v1Service.getRecord()
   └─ 返回 IRecord (V1 格式)

4. 前端调用
   @teable/sdk useRecordsQuery()
   └─ @teable/openapi getRecords()
      └─ axios.get('/api/table/tblxxx/record/recxxx')
```

### V2 数据流

```
1. API 定义 (开发时)
   @teable/v2-core/src/commands/CreateRecordCommand.ts
   └─ Zod Schema: createRecordInputSchema
   
   @teable/v2-contract-http/src/contract.ts
   └─ oc.route({ path: '/tables/createRecord', method: 'POST' })
         .input(createRecordInputSchema)
         .output(createRecordOkResponseSchema)

2. 文档生成 (启动时)
   V2OpenApiController.openapi()
   └─ generateV2OpenApiDocument()
      ├─ @orpc/openapi OpenAPIGenerator
      └─ v2Contract → openapi.json
         └─ 挂载到 /api/v2/docs (Scalar UI)

3. 请求处理 (运行时)
   HTTP POST /api/v2/tables/createRecord
   ├─ @orpc/nest 自动路由匹配
   ├─ @orpc 自动验证输入 schema
   ├─ V2Controller.tables.createRecord handler
   ├─ executeCreateRecordEndpoint(context, input, commandBus)
   │  ├─ CreateRecordCommand.create(input) → 领域验证
   │  ├─ commandBus.execute() → 领域处理
   │  └─ mapCreateRecordResultToDto() → DTO 转换
   └─ 返回 { ok: true, data: { record, events } }

4. 前端调用
   @teable/v2-contract-http-client createV2HttpClient()
   └─ client.tables.createRecord({ tableId, fields })
      └─ @orpc/client 自动序列化请求 + 解析响应
```

## 第六部分：设计优势与演进方向

### V1 优势
- ✅ 简单直接，易于理解
- ✅ 手动编写的 Axios 函数灵活性高
- ✅ RESTful 路径符合传统 API 设计习惯

### V1 问题
- ❌ 契约与传输层绑定，无法复用
- ❌ 客户端函数需手动编写，易出错
- ❌ 响应格式不统一，错误处理不一致
- ❌ 服务端路由需手动定义，与契约可能不一致

### V2 优势
- ✅ 契约与实现分离，可支持多传输协议 (HTTP, WebSocket, 等)
- ✅ 客户端自动生成，零样板代码
- ✅ 统一响应格式，错误处理标准化
- ✅ 服务端路由自动绑定，确保与契约一致
- ✅ 领域模型与 HTTP 传输解耦，更清晰的架构分层

### V2 当前限制
- ⚠️ 部分 V1 API 尚未迁移到 V2
- ⚠️ 双轨运行增加了维护成本
- ⚠️ 需要格式转换以保持向后兼容

## 总结

Teable 的 OpenAPI 与 SDK 对齐机制经历了从 **v1 手动契约** 到 **v2 契约驱动** 的演进：

| 维度 | v1 方式 | v2 方式 |
|------|---------|---------|
| **契约定义** | 手动 RouteConfig + Axios 函数 | @orpc/contract 声明式 |
| **文档生成** | zod-to-openapi | @orpc/openapi |
| **服务端** | NestJS Controller 手动绑定 | @orpc/nest 自动实现 |
| **客户端** | 手动编写 Axios 函数 | @orpc/client 自动生成 |
| **类型安全** | 依赖人工维护 | 编译时自动推导 |
| **错误处理** | HttpException | neverthrow + ORPCError |

**文档对齐现状**:
- V1 文档与 V1 行为完全对齐
- V2 文档与 V2 行为完全对齐
- 运行时 V2 切换通过 V2FeatureGuard + CLS 实现
- V1 API 路径在使用 V2 实现时会进行格式转换以保持向后兼容
- 通过 `x-teable-v2` 响应头标识实际使用的实现版本

这种渐进式演进策略确保了系统在重构过程中的稳定性，同时为最终全面迁移到 v2 架构奠定了基础。

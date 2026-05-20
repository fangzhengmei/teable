# OpenAPI 文档与 SDK 自动生成机制分析

## 概述

Teable 项目正在经历从 v1 到 v2 的架构演进。两个版本都采用了**以 Zod Schema 为单一数据源**的 API 定义与客户端生成机制，但在具体实现路径、责任归属和技术选型上有显著差异。本文档深入分析两个版本的接口契约、文档生成、客户端封装的接力方式，以及运行时切换机制，并修正关键实现细节。

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
│  │  路径风格: /tables/listRecords (动作式 RPC)                         │  │
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

#### API 定义四要素（以 getRecords 为例）

[packages/openapi/src/record/get-list.ts](packages/openapi/src/record/get-list.ts)

```typescript
// 1. Zod Schema 定义（请求参数）
export const getRecordsRoSchema = getRecordQuerySchema.extend(contentQueryBaseSchema.shape).extend({
  take: z.string().or(z.number()).transform(Number)
    .pipe(z.number().min(1).max(1000)).default(100).optional(),
  skip: z.string().or(z.number()).transform(Number)
    .pipe(z.number().min(0)).default(0).optional(),
});

export type IGetRecordsRo = z.infer<typeof getRecordsRoSchema>;

// 2. URL 常量 (RESTful 风格)
export const GET_RECORDS_URL = '/table/{tableId}/record';

// 3. RouteConfig 注册（用于 OpenAPI 文档生成）
export const GetRecordsRoute: RouteConfig = registerRoute({
  method: 'get',
  path: GET_RECORDS_URL,
  summary: 'List records',
  description: 'Retrieve a list of records...',
  request: {
    params: z.object({ tableId: z.string() }),
    query: getRecordsRoSchema,
  },
  responses: {
    200: {
      description: 'List of records',
      content: {
        'application/json': { schema: recordsVoSchema },
      },
    },
  },
  tags: ['record'],
  // ❌ 注意：这里没有指定 security，但生成时会被强制添加
});

// 4. Axios 客户端函数（手动编写）
export async function getRecords(
  tableId: string,
  query?: IGetRecordsRo
): Promise<AxiosResponse<IRecordsVo>> {
  // 手动序列化复杂参数
  const serializedQuery = {
    ...query,
    filter: query?.filter ? JSON.stringify(query.filter) : undefined,
    orderBy: query?.orderBy ? JSON.stringify(query.orderBy) : undefined,
  };

  return axios.get<IRecordsVo>(
    urlBuilder(GET_RECORDS_URL, { tableId }), 
    { params: serializedQuery }
  );
}
```

**责任归属**:
- ✅ Schema 定义: `@teable/openapi`
- ✅ 路由元数据: `@teable/openapi`
- ✅ 客户端函数: `@teable/openapi` (手动编写)
- ❌ 服务端路由: 不负责，由 NestJS Controller 手动定义
- ❌ 鉴权声明: RouteConfig 中不定义，由生成器统一添加

### 1.2 文档生成

[packages/openapi/src/generate.schema.ts](packages/openapi/src/generate.schema.ts)

```typescript
function registerRoutes(filters?: { tags?: string[]; paths?: string[]; methods?: string[] }) {
  const registry = new OpenAPIRegistry();
  const routeObjList: RouteConfig[] = getRoutes();

  // 过滤路由...

  for (const routeObj of filteredRoutes) {
    const bearerAuth = registry.registerComponent('securitySchemes', 'bearerAuth', {
      type: 'http',
      scheme: 'bearer',
    });

    // ⚠️  关键：所有路由被强制添加鉴权声明
    // 无论 RouteConfig 中是否定义 security，都会被覆盖
    registry.registerPath({ 
      ...routeObj, 
      security: [{ [bearerAuth.name]: [] }]  // 强制添加
    });
  }
  return registry;
}

export async function getOpenApiDocumentation(config: {...}) {
  const registry = registerRoutes({ tags, paths, methods });
  const generator = new OpenApiGeneratorV3(registry.definitions);
  
  return generator.generateDocument({
    openapi: '3.0.0',
    info: {
      version: '1.0.0',
      title: 'Teable App',
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

#### ⚠️  文档与运行时行为的不一致

**1. 鉴权声明不一致**：
- **文档中**：所有路由都被强制标记为需要 `bearerAuth` 鉴权
- **运行时**：部分 Controller 使用 `@AllowAnonymous()` 装饰器，实际不需要鉴权

```typescript
// apps/nestjs-backend/src/features/record/open-api/record-open-api.controller.ts
@AllowAnonymous()  // 实际允许匿名访问
@Controller('api/table/:tableId/record')
export class RecordOpenApiController {
  // 但文档中该接口仍被标记为需要 Authorization header
}
```

**受影响的接口包括**：
- 共享视图接口 (`/share/{shareId}/view/records`)
- 记录相关接口 (`/table/{tableId}/record`)
- 其他使用 `@AllowAnonymous()` 或 `@Public()` 的接口

**2. 状态码不一致**：
- **文档中**：RouteConfig 定义的响应状态码（如 200, 201）
- **运行时**：实际可能返回 400, 401, 403, 404, 500 等错误状态码，但文档中未完整声明

### 1.3 路由收集的副作用与覆盖风险

[packages/openapi/src/utils.ts](packages/openapi/src/utils.ts)

```typescript
const routes: RouteConfig[] = [];

export const registerRoute = (route: RouteConfig) => {
  // ⚠️  关键：没有去重检查，直接 push
  routes.push(route);
  return route;
};

export const getRoutes = () => routes;
```

#### 覆盖风险分析

**1. 模块加载顺序决定路由顺序**

路由数组的填充依赖于模块的 `import` 顺序。在 `packages/openapi/src/index.ts` 中：

```typescript
export * from './zod';
export * from './axios';
export * from './generate.schema';
export * from './record';     // 先加载 record 模块
export * from './field';      // 后加载 field 模块
export * from './view';       // 后加载 view 模块
// ... 更多模块
```

每个被 `export *` 的模块会执行其顶级代码，包括 `registerRoute()` 调用。

**2. 相同 path + method 的后注册者覆盖**

当生成 OpenAPI 文档时：

```typescript
// 假设 routes 数组中有两条相同 path + method 的路由
// [
//   { path: '/table/{tableId}/record', method: 'get', ... },  // 先注册
//   { path: '/table/{tableId}/record', method: 'get', ... },  // 后注册
// ]

// 在 generateDocument 时，OpenAPI 的 paths 是对象
// paths['/table/{tableId}/record']['get'] = 后注册的路由定义
// 先注册的被覆盖！
```

**3. 实际风险场景**

目前代码中没有发现重复路径，但以下情况可能触发：
- 不同模块意外定义了相同的 URL 路径
- 重构时重命名文件但忘记删除旧文件
- 条件加载导致某些模块被重复导入

**4. 缓解措施**

- 模块导出顺序固定（在 `index.ts` 中明确列出）
- 代码审查时检查是否有重复的 URL 常量定义
- 单元测试验证路由数量和路径唯一性

### 1.4 客户端类型/调用封装

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

### 1.5 V1 后端接入

[apps/nestjs-backend/src/features/record/open-api/record-open-api.controller.ts](apps/nestjs-backend/src/features/record/open-api/record-open-api.controller.ts)

```typescript
@UseGuards(V2FeatureGuard)
@UseInterceptors(V2IndicatorInterceptor)
@AllowAnonymous()  // 实际允许匿名访问
@Controller('api/table/:tableId/record')
export class RecordOpenApiController {
  @UseV2Feature('getRecords')
  @Permissions('record|read')
  @Get()
  async getRecords(
    @Param('tableId') tableId: string,
    @Query(new ZodValidationPipe(getRecordsRoSchema), TqlPipe, FieldKeyPipe) query: IGetRecordsRo
  ): Promise<IRecordsVo> {
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
    ├── getRecordById.ts        // 单条记录查询
    ├── listTableRecords.ts     // 列表查询
    ├── dto.ts
    ├── recordDto.ts
    └── ...
```

#### API 定义方式（以读取记录为例）

**1. 单条记录查询 (getRecord)**

**输入 Schema 定义** (来自 `@teable/v2-core`):

[packages/v2/core/src/queries/GetRecordByIdQuery.ts](packages/v2/core/src/queries/GetRecordByIdQuery.ts)

```typescript
export const getRecordByIdInputSchema = z.object({
  tableId: z.string(),
  recordId: z.string(),
});

export type IGetRecordByIdQueryInput = z.input<typeof getRecordByIdInputSchema>;

export class GetRecordByIdQuery {
  static create(raw: unknown): Result<GetRecordByIdQuery, DomainError> {
    const parsed = getRecordByIdInputSchema.safeParse(raw);
    if (!parsed.success)
      return err(domainError.validation({ message: 'Invalid GetRecordByIdQuery input' }));

    return TableId.create(parsed.data.tableId).andThen((tableId) =>
      RecordId.create(parsed.data.recordId).map(
        (recordId) => new GetRecordByIdQuery(tableId, recordId)
      )
    );
  }
}
```

**响应 DTO 定义**:

[packages/v2/contract-http/src/table/getRecordById.ts](packages/v2/contract-http/src/table/getRecordById.ts)

```typescript
export interface IGetRecordByIdResponseDataDto {
  record: ITableRecordDto;
}

export const getRecordByIdResponseDataSchema = z.object({
  record: tableRecordDtoSchema,
});

export const getRecordByIdOkResponseSchema = apiOkResponseDtoSchema(
  getRecordByIdResponseDataSchema
);

export const mapGetRecordByIdResultToDto = (
  result: GetRecordByIdResult
): Result<IGetRecordByIdResponseDataDto, DomainError> => {
  return mapTableRecordToDto(result.record).map((record) => ({ record }));
};
```

**2. 记录列表查询 (listRecords)**

**输入 Schema 定义**:

[packages/v2/core/src/queries/ListTableRecordsQuery.ts](packages/v2/core/src/queries/ListTableRecordsQuery.ts)

```typescript
export const listTableRecordsInputSchema = z
  .object({
    tableId: z.string(),
    filter: parseJsonInput(recordFilterSchema).optional(),
    sort: parseJsonInput(z.array(recordSortSchema)).optional(),
    groupBy: parseJsonInput(recordGroupBySchema).optional(),
    search: parseJsonInput(recordSearchInputSchema).optional(),
    viewId: z.string().min(1).optional(),
    ignoreViewQuery: z.coerce.boolean().optional(),
    limit: z.coerce.number().int().positive().max(MAX_RECORDS_LIMIT).optional(),
    offset: z.coerce.number().int().nonnegative().optional(),
    fieldKeyType: fieldKeyTypeSchema,
  })
  .superRefine((value, ctx) => {
    if (value.filterLinkCellSelected && value.filterLinkCellCandidate) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'filterLinkCellSelected and filterLinkCellCandidate can not be set at the same time',
        path: ['filterLinkCellSelected'],
      });
    }
  });

export type IListTableRecordsQueryInput = z.input<typeof listTableRecordsInputSchema>;
```

**响应 DTO 定义**:

[packages/v2/contract-http/src/table/listTableRecords.ts](packages/v2/contract-http/src/table/listTableRecords.ts)

```typescript
export interface IListTableRecordsPaginationDto {
  total: number;
  offset: number;
  limit: number;
  hasMore: boolean;
}

export interface IListTableRecordsResponseDataDto {
  records: ITableRecordDto[];
  pagination: IListTableRecordsPaginationDto;
}

export const listTableRecordsOkResponseSchema = apiOkResponseDtoSchema(
  z.object({
    records: z.array(tableRecordDtoSchema),
    pagination: z.object({
      total: z.number().int().nonnegative(),
      offset: z.number().int().nonnegative(),
      limit: z.number().int().positive(),
      hasMore: z.boolean(),
    }),
  })
);

export const mapListTableRecordsResultToDto = (
  result: ListTableRecordsResult
): Result<IListTableRecordsResponseDataDto, DomainError> => {
  return sequenceResults(result.records.map(mapTableRecordToDto)).map((records) => ({
    records: [...records],
    pagination: {
      total: result.total,
      offset: result.offset,
      limit: result.limit,
      hasMore: result.offset + records.length < result.total,
    },
  }));
};
```

**3. 契约注册**:

[packages/v2/contract-http/src/contract.ts](packages/v2/contract-http/src/contract.ts)

```typescript
import { oc } from '@orpc/contract';
import { getRecordByIdInputSchema, listTableRecordsInputSchema } from '@teable/v2-core';
import { getRecordByIdOkResponseSchema, listTableRecordsOkResponseSchema } from './table';

const TABLES_GET_RECORD_PATH = '/tables/getRecord';
const TABLES_LIST_RECORDS_PATH = '/tables/listRecords';

export const v2Contract = {
  tables: {
    // ✅ 单条记录查询：GET 方法，参数通过 query string 传递
    getRecord: oc
      .route({
        method: 'GET',
        path: TABLES_GET_RECORD_PATH,
        successStatus: 200,
        summary: 'Get record by id',
        tags: ['tables'],
      })
      .input(getRecordByIdInputSchema)  // { tableId, recordId }
      .output(getRecordByIdOkResponseSchema),

    // ✅ 记录列表查询：GET 方法，参数通过 query string 传递
    listRecords: oc
      .route({
        method: 'GET',
        path: TABLES_LIST_RECORDS_PATH,
        successStatus: 200,
        summary: 'List table records',
        tags: ['tables'],
      })
      .input(listTableRecordsInputSchema)  // { tableId, filter?, sort?, ... }
      .output(listTableRecordsOkResponseSchema),

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
- ✅ Schema 定义: `@teable/v2-core` (领域层，与传输无关)
- ✅ 契约定义: `@teable/v2-contract-http`
- ✅ DTO 转换: `@teable/v2-contract-http`
- ✅ 服务端路由: 由 @orpc/nest 自动绑定
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

**调用示例**：

```typescript
import { createV2HttpClient } from '@teable/v2-contract-http-client';

const client = createV2HttpClient({
  baseUrl: 'https://app.teable.ai/api/v2',
  headers: { Authorization: 'Bearer <token>' },
});

// ✅ 单条记录查询：GET /tables/getRecord?tableId=tblxxx&recordId=recxxx
const recordResult = await client.tables.getRecord({
  tableId: 'tblxxx',
  recordId: 'recxxx',
});
// 返回: { ok: true; data: { record: { id, fields } } }

// ✅ 记录列表查询：GET /tables/listRecords?tableId=tblxxx&limit=100&offset=0
const listResult = await client.tables.listRecords({
  tableId: 'tblxxx',
  limit: 100,
  offset: 0,
  fieldKeyType: 'name',
  // 复杂参数会被自动序列化为 JSON 字符串
  filter: {
    conditions: [
      { fieldId: 'fldxxx', operator: 'is', value: 'value' },
    ],
  },
  sort: [{ fieldId: 'fldxxx', order: 'asc' }],
});
// 返回: { ok: true; data: { records: [...], pagination: {...} } }
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

**端点处理器示例** (listRecords):

[packages/v2/contract-http-implementation/src/handlers/tables/listTableRecords.ts](packages/v2/contract-http-implementation/src/handlers/tables/listTableRecords.ts)

```typescript
export const executeListTableRecordsEndpoint = async (
  context: IExecutionContext,
  rawInput: unknown,
  queryBus: IQueryBus
): Promise<IListTableRecordsEndpointResult> => {
  // 1. 原始输入 → 领域对象
  const queryResult = ListTableRecordsQuery.create(rawInput);
  if (queryResult.isErr()) {
    return {
      status: mapDomainErrorToHttpStatus(queryResult.error),
      body: { ok: false, error: mapDomainErrorToHttpError(queryResult.error) },
    };
  }

  // 2. 执行查询
  const result = await queryBus.execute<ListTableRecordsQuery, ListTableRecordsResult>(
    context,
    queryResult.value
  );
  if (result.isErr()) {
    return {
      status: mapDomainErrorToHttpStatus(result.error),
      body: { ok: false, error: mapDomainErrorToHttpError(result.error) },
    };
  }

  // 3. 领域结果 → DTO
  const mapped = mapListTableRecordsResultToDto(result.value);
  if (mapped.isErr()) {
    return {
      status: mapDomainErrorToHttpStatus(mapped.error),
      body: { ok: false, error: mapDomainErrorToHttpError(mapped.error) },
    };
  }

  return {
    status: 200,
    body: { ok: true, data: mapped.value },
  };
};
```

**NestJS 集成**:

[apps/nestjs-backend/src/features/v2/v2.controller.ts](apps/nestjs-backend/src/features/v2/v2.controller.ts)

```typescript
@Controller('api/v2')
export class V2Controller {
  @Implement(v2Contract.tables)
  tables() {
    return {
      getRecord: implement(v2Contract.tables.getRecord).handler(async ({ input }) => {
        const container = await this.v2Container.getContainer();
        const queryBus = container.resolve<IQueryBus>(v2CoreTokens.queryBus);
        const context = await this.v2ContextFactory.createContext();

        const result = await executeGetRecordByIdEndpoint(context, input, queryBus);
        if (result.status === 200) return result.body;
        throwOrpcErrorByStatus(result.status, result.body.error);
      }),
      listRecords: implement(v2Contract.tables.listRecords).handler(async ({ input }) => {
        const container = await this.v2Container.getContainer();
        const queryBus = container.resolve<IQueryBus>(v2CoreTokens.queryBus);
        const context = await this.v2ContextFactory.createContext();

        const result = await executeListTableRecordsEndpoint(context, input, queryBus);
        if (result.status === 200) return result.body;
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
| **鉴权声明** | 生成器统一强制添加 bearerAuth | 由 @orpc 契约定义 |
| **服务端路由** | NestJS Controller 手动定义 | `@orpc/nest` 自动绑定 |
| **服务端验证** | ZodValidationPipe 手动应用 | 由 @orpc 自动处理 |
| **客户端函数** | `@teable/openapi` 手动编写 Axios 函数 | `@orpc/client` 自动生成类型安全客户端 |
| **响应格式** | 自由格式 (直接返回数据) | 统一 `{ ok: true, data: T }` / `{ ok: false, error: E }` |
| **路径风格** | RESTful `/table/{tableId}/record/{recordId}` | RPC 动作式 `/tables/listRecords` |
| **错误处理** | HttpException 抛异常 | neverthrow Result + ORPCError |

## 第四部分：文档-类型-调用 链路边界关系

### 三条独立链路

```
┌─────────────────────────────────────────────────────────────────────┐
│                    文档生成链路 (OpenAPI JSON)                       │
├─────────────────────────────────────────────────────────────────────┤
│  Zod Schema                                                          │
│      ↓ .meta() / .describe()                                        │
│  RouteConfig (path, method, summary, tags, request, responses)       │
│      ↓ registerRoute()                                               │
│  全局 routes[] 数组                                                  │
│      ↓ getOpenApiDocumentation()                                    │
│  OpenAPIRegistry                                                    │
│      ↓ OpenApiGeneratorV3                                           │
│  OpenAPI 3.0 JSON                                                   │
│      ↓                                                               │
│  Swagger UI / Redoc                                                 │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    类型生成链路 (TypeScript)                         │
├─────────────────────────────────────────────────────────────────────┤
│  Zod Schema                                                          │
│      ↓ z.infer<typeof schema>                                       │
│  TypeScript 类型 (IGetRecordsRo, IRecordsVo, 等)                    │
│      ↓                                                               │
│  导入到 Controller / Service / 前端组件                              │
│      ↓                                                               │
│  编译时类型检查                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    调用封装链路 (HTTP Client)                        │
├─────────────────────────────────────────────────────────────────────┤
│  V1: 手动编写                                                        │
│    Axios 实例 → getRecords(tableId, query) → { data }              │
│        ↓                                                             │
│    @teable/sdk → useRecordsQuery() → React Query → UI               │
│                                                                     │
│  V2: 自动生成                                                        │
│    v2Contract → @orpc/client → createV2HttpClient()                │
│        ↓                                                             │
│    client.tables.listRecords(input) → 类型安全调用                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 边界关系详解

#### 1. 文档生成链路 vs 类型生成链路

**共同点**：共享同一个 Zod Schema

**不同点**：
- 文档生成需要额外的元数据：`.meta({ type: 'string', description: '...' })`
- 类型生成只需要 Schema 的结构，不需要元数据
- 文档生成是运行时行为（启动时执行）
- 类型生成是编译时行为（TypeScript 编译）

**边界**：
- 如果 Schema 缺少 `.meta()`，文档生成可能失败或显示不正确的类型
- 但类型生成不受影响

#### 2. 文档生成链路 vs 调用封装链路

**V1 中完全独立**：
- 文档生成：从 `routes[]` 数组读取 RouteConfig
- 调用封装：手动编写的 Axios 函数
- **没有自动同步机制**：RouteConfig 更新后，Axios 函数需要手动同步

**V2 中通过契约关联**：
- 文档生成：从 `v2Contract` 读取
- 调用封装：`@orpc/client` 也从 `v2Contract` 生成
- **自动同步**：契约修改后，文档和客户端同时更新

#### 3. 类型生成链路 vs 调用封装链路

**V1 中手动关联**：
- 类型：`z.infer<typeof getRecordsRoSchema>` → `IGetRecordsRo`
- 调用：`getRecords(tableId: string, query?: IGetRecordsRo)`
- 开发者需要手动确保函数签名与类型匹配

**V2 中自动关联**：
- 类型：由 `v2Contract` 推导
- 调用：`client.tables.listRecords(input)` 自动获得类型
- 编译时自动检查一致性

## 第五部分：运行时 V2 切换机制与文档对齐

### 5.1 Canary 发布系统

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

### 5.2 V2FeatureGuard 工作流程

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

### 5.3 Controller 中的分支逻辑

[apps/nestjs-backend/src/features/record/open-api/record-open-api.controller.ts](apps/nestjs-backend/src/features/record/open-api/record-open-api.controller.ts)

```typescript
@UseGuards(V2FeatureGuard)
@UseInterceptors(V2IndicatorInterceptor)
@AllowAnonymous()
@Controller('api/table/:tableId/record')
export class RecordOpenApiController {
  @UseV2Feature('getRecords')
  @Permissions('record|read')
  @Get()
  async getRecords(
    @Param('tableId') tableId: string,
    @Query(new ZodValidationPipe(getRecordsRoSchema), TqlPipe, FieldKeyPipe) query: IGetRecordsRo
  ): Promise<IRecordsVo> {
    // 运行时分支
    if (this.cls.get('useV2')) {
      return this.recordOpenApiV2Service.getRecords(tableId, query);
    }
    return await this.recordService.getRecords(tableId, query, true);
  }
}
```

### 5.4 文档与实际行为的对齐情况

#### ✅ 对齐的部分

1. **V1 文档与 V1 行为**: 完全对齐（除鉴权声明外）
   - 文档从 `@teable/openapi` 生成
   - 后端 Controller 路径与 RouteConfig 定义一致
   - 请求/响应格式匹配

2. **V2 独立文档与 V2 行为**: 完全对齐
   - `/api/v2/openapi.json` 从 `v2Contract` 生成
   - 路径 `/api/v2/tables/listRecords` 与契约定义一致
   - 统一响应格式 `{ ok: true, data: T }`

#### ⚠️ 不完全对齐的部分

1. **V1 文档中的鉴权声明过度覆盖**:
   - 文档中所有路由都被标记为需要 Bearer Token
   - 但实际 `@AllowAnonymous()` 接口不需要鉴权
   - 影响：Swagger UI 测试时需要填写不必要的 Authorization header

2. **V1 文档中的 V2 行为**:
   - 当 `useV2=true` 时，`GET /api/table/{tableId}/record` 实际调用 V2 服务
   - 但 Swagger 文档 (`/docs`) 仍然显示 V1 的响应 schema
   - **注意**: V2Service 会进行数据格式转换以保持 V1 兼容性

3. **V1 路径与 V2 路径并存**:
   - V1: `GET /api/table/{tableId}/record/{recordId}`
   - V2: `GET /api/v2/tables/getRecord?tableId=xxx&recordId=xxx`
   - 两者功能相同但调用方式不同（URL 路径 vs Query 参数）

4. **v2Indicator 响应头**:
   - `V2IndicatorInterceptor` 会在响应头中添加 `x-teable-v2: true` 当使用 V2 实现时
   - 可用于调试和监控

### 5.5 V2 服务中的格式适配

[apps/nestjs-backend/src/features/record/open-api/record-open-api-v2.service.ts](apps/nestjs-backend/src/features/record/open-api/record-open-api-v2.service.ts)

```typescript
@Injectable()
export class RecordOpenApiV2Service {
  async getRecords(
    tableId: string,
    query: IGetRecordsRo,  // V1 输入格式
  ): Promise<IRecordsVo> {   // V1 输出格式
    const container = await this.v2ContainerService.getContainer();
    const queryBus = container.resolve<IQueryBus>(v2CoreTokens.queryBus);
    const context = await this.createV2ReadContext(tableId, query);

    // 调用 V2 端点处理器
    const pageResult = await this.executeListRecordsEndpoint(
      {
        tableId,
        fieldKeyType: FieldKeyType.Id,
        limit: query.take,
        offset: query.skip,
        filter: normalizedFilter,
        sort: normalizedSort,
        groupBy: normalizedGroupBy,
        search: query.search,
        viewId: query.viewId,
      },
      context,
      queryBus
    );

    // V2 → V1 格式转换
    // V2: { records: [...], pagination: { total, offset, limit, hasMore } }
    // V1: { records: [...], extra?: { groupPoints, ... } }
    const records = await this.recordService.getSnapshotBulkWithPermission(...);
    
    return queryExtra
      ? { records: normalizedRecords, extra: queryExtra }
      : { records: normalizedRecords };
  }
}
```

## 第六部分：完整数据流对比

### V1 数据流

```
1. API 定义 (开发时)
   @teable/openapi/src/record/get-list.ts
   ├─ Zod Schema: getRecordsRoSchema
   ├─ URL: GET_RECORDS_URL = '/table/{tableId}/record'
   ├─ RouteConfig: GetRecordsRoute (注册到全局数组)
   └─ Axios 函数: getRecords(tableId, query)

2. 类型生成 (编译时)
   z.infer<typeof getRecordsRoSchema> → IGetRecordsRo
   z.infer<typeof recordsVoSchema> → IRecordsVo

3. 文档生成 (启动时)
   setupSwagger()
   └─ getOpenApiDocumentation()
      ├─ getRoutes() → 所有已注册的 RouteConfig
      ├─ 强制添加 security: bearerAuth 到所有路由
      └─ @asteasolutions/zod-to-openapi → openapi.json
         └─ 挂载到 /docs, /redocs

4. 请求处理 (运行时)
   HTTP GET /api/table/tblxxx/record?limit=100
   ├─ NestJS 路由匹配 → RecordOpenApiController.getRecords()
   ├─ ZodValidationPipe(getRecordsRoSchema) 验证 query
   ├─ V2FeatureGuard 决策 useV2
   ├─ 分支: useV2 ? v2Service.getRecords() : v1Service.getRecords()
   └─ 返回 IRecordsVo (V1 格式)

5. 前端调用
   @teable/sdk useRecordsQuery()
   └─ @teable/openapi getRecords(tableId, query)
      └─ axios.get('/api/table/tblxxx/record?limit=100')
```

### V2 数据流

```
1. API 定义 (开发时)
   @teable/v2-core/src/queries/ListTableRecordsQuery.ts
   └─ Zod Schema: listTableRecordsInputSchema
   
   @teable/v2-contract-http/src/contract.ts
   └─ oc.route({ path: '/tables/listRecords', method: 'GET' })
         .input(listTableRecordsInputSchema)
         .output(listTableRecordsOkResponseSchema)

2. 类型生成 (编译时)
   @orpc/contract 自动推导输入输出类型
   无需手动 z.infer

3. 文档生成 (启动时)
   V2OpenApiController.openapi()
   └─ generateV2OpenApiDocument()
      ├─ @orpc/openapi OpenAPIGenerator
      └─ v2Contract → openapi.json
         └─ 挂载到 /api/v2/docs (Scalar UI)

4. 请求处理 (运行时)
   HTTP GET /api/v2/tables/listRecords?tableId=tblxxx&limit=100
   ├─ @orpc/nest 自动路由匹配
   ├─ @orpc 自动验证输入 schema
   ├─ V2Controller.tables.listRecords handler
   ├─ executeListTableRecordsEndpoint(context, input, queryBus)
   │  ├─ ListTableRecordsQuery.create(input) → 领域验证
   │  ├─ queryBus.execute() → 领域处理
   │  └─ mapListTableRecordsResultToDto() → DTO 转换
   └─ 返回 { ok: true, data: { records, pagination } }

5. 前端调用
   @teable/v2-contract-http-client createV2HttpClient()
   └─ client.tables.listRecords({ tableId, limit: 100 })
      └─ @orpc/client 自动序列化请求 + 解析响应
```

## 第七部分：设计优势与演进方向

### V1 优势
- ✅ 简单直接，易于理解
- ✅ 手动编写的 Axios 函数灵活性高
- ✅ RESTful 路径符合传统 API 设计习惯

### V1 问题
- ❌ 契约与传输层绑定，无法复用
- ❌ 客户端函数需手动编写，易出错且与文档可能不一致
- ❌ 响应格式不统一，错误处理不一致
- ❌ 服务端路由需手动定义，与契约可能不一致
- ❌ 鉴权声明在文档中过度覆盖，与实际行为不符
- ❌ 路由收集无去重，存在模块加载顺序依赖和覆盖风险

### V2 优势
- ✅ 契约与实现分离，可支持多传输协议 (HTTP, WebSocket, 等)
- ✅ 客户端自动生成，零样板代码，与文档自动同步
- ✅ 统一响应格式，错误处理标准化
- ✅ 服务端路由自动绑定，确保与契约一致
- ✅ 领域模型与 HTTP 传输解耦，更清晰的架构分层
- ✅ Schema 定义在纯领域层，可被复用

### V2 当前限制
- ⚠️ 部分 V1 API 尚未迁移到 V2
- ⚠️ 双轨运行增加了维护成本（V2Service 需要格式转换）
- ⚠️ V1 路径和 V2 路径并存，增加了理解成本

## 总结

Teable 的 OpenAPI 与 SDK 对齐机制经历了从 **v1 手动契约** 到 **v2 契约驱动** 的演进：

| 维度 | v1 方式 | v2 方式 |
|------|---------|---------|
| **契约定义** | 手动 RouteConfig + Axios 函数 | @orpc/contract 声明式 |
| **文档生成** | zod-to-openapi（强制鉴权） | @orpc/openapi |
| **服务端** | NestJS Controller 手动绑定 | @orpc/nest 自动实现 |
| **客户端** | 手动编写 Axios 函数 | @orpc/client 自动生成 |
| **类型安全** | 依赖人工维护 | 编译时自动推导 |
| **错误处理** | HttpException | neverthrow + ORPCError |

**文档对齐现状**:
- V1 文档与 V1 行为基本对齐，但鉴权声明存在过度覆盖
- V2 文档与 V2 行为完全对齐
- 运行时 V2 切换通过 V2FeatureGuard + CLS 实现
- V1 API 路径在使用 V2 实现时会进行格式转换以保持向后兼容
- 通过 `x-teable-v2` 响应头标识实际使用的实现版本

**三条独立链路的边界**:
1. **文档生成链路**: Zod Schema → RouteConfig → OpenAPI JSON，依赖 `.meta()` 元数据
2. **类型生成链路**: Zod Schema → z.infer → TypeScript 类型，纯编译时行为
3. **调用封装链路**: V1 手动编写 / V2 自动生成，与文档链路在 V2 中通过契约自动同步

这种渐进式演进策略确保了系统在重构过程中的稳定性，同时为最终全面迁移到 v2 架构奠定了基础。

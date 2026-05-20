# OpenAPI 文档与 SDK 自动生成机制分析

## 概述

Teable 项目采用了一套**以 Zod Schema 为单一数据源**的 API 定义与客户端生成机制，实现了从接口元数据定义 → OpenAPI 文档生成 → 前端 SDK 封装的完整接力。这套机制确保了前后端类型一致，避免了重复定义和手动维护的一致性问题。

## 三层架构与接力关系

```
┌─────────────────────────────────────────────────────────────┐
│  packages/openapi (@teable/openapi)                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  1. Zod Schema 定义 (请求/响应类型)                    │  │
│  │  2. RouteConfig 注册 (路径、方法、描述、标签)           │  │
│  │  3. Axios 客户端函数封装                               │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────┬───────────────────────────────────────┘
                       │
     ┌─────────────────┴─────────────────┐
     ▼                                   ▼
┌──────────────────────────┐    ┌──────────────────────────┐
│ apps/nestjs-backend      │    │ packages/sdk             │
│  - ZodValidationPipe     │    │  - React Hooks 封装      │
│  - Controller 路由       │    │  - React Query 集成      │
│  - Service 业务逻辑      │    │  - Model 实例包装        │
└──────────────────────────┘    └──────────────────────────┘
```

## 第一层：@teable/openapi - API 元数据定义中心

### 核心技术栈

- **Zod**: TypeScript 优先的 schema 验证库
- **@asteasolutions/zod-to-openapi**: 将 Zod Schema 转换为 OpenAPI 3.0 规范
- **Axios**: HTTP 客户端

### 目录结构

```
packages/openapi/src/
├── zod.ts                      # 扩展了 OpenAPI 支持的 Zod
├── utils.ts                    # 路由注册与 URL 构建工具
├── generate.schema.ts          # OpenAPI 文档生成器
├── axios.ts                    # Axios 实例配置与导出
├── types.ts                    # 通用类型定义
└── [feature]/                  # 按功能模块划分的 API 定义
    ├── index.ts
    ├── get.ts
    ├── create.ts
    ├── update.ts
    └── ...
```

### 关键实现细节

#### 1. Zod 扩展 OpenAPI 支持

[packages/openapi/src/zod.ts](packages/openapi/src/zod.ts)

```typescript
import { extendZodWithOpenApi } from '@asteasolutions/zod-to-openapi';
import { z } from 'zod';

extendZodWithOpenApi(z);
export { z };
```

通过 `extendZodWithOpenApi` 扩展 Zod，使其支持 `.openapi()`、`.meta()`、`.describe()` 等方法，用于添加 OpenAPI 元数据。

#### 2. 路由注册机制

[packages/openapi/src/utils.ts](packages/openapi/src/utils.ts)

```typescript
import type { RouteConfig } from '@asteasolutions/zod-to-openapi';

const routes: RouteConfig[] = [];

export const registerRoute = (route: RouteConfig) => {
  routes.push(route);
  return route;
};

export const getRoutes = () => routes;
```

使用一个全局数组收集所有路由定义，在生成 OpenAPI 文档时统一处理。

#### 3. API 定义示例（以 getRecord 为例）

[packages/openapi/src/record/get.ts](packages/openapi/src/record/get.ts)

每个 API 定义包含四个关键部分：

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

// 2. URL 常量
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

// 4. Axios 客户端函数
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

#### 4. OpenAPI 文档生成

[packages/openapi/src/generate.schema.ts](packages/openapi/src/generate.schema.ts)

```typescript
import { OpenAPIRegistry, OpenApiGeneratorV3 } from '@asteasolutions/zod-to-openapi';
import { getRoutes } from './utils';

export async function getOpenApiDocumentation(config: {
  origin?: string;
  snippet?: boolean;
  tags?: string[];
  paths?: string[];
  methods?: string[];
}): Promise<OpenAPIObject> {
  const registry = new OpenAPIRegistry();
  const routeObjList: RouteConfig[] = getRoutes();

  // 注册所有路由
  for (const routeObj of filteredRoutes) {
    const bearerAuth = registry.registerComponent('securitySchemes', 'bearerAuth', {
      type: 'http',
      scheme: 'bearer',
    });
    registry.registerPath({ ...routeObj, security: [{ [bearerAuth.name]: [] }] });
  }

  // 生成 OpenAPI 3.0 文档
  const generator = new OpenApiGeneratorV3(registry.definitions);
  const document = generator.generateDocument({
    openapi: '3.0.0',
    info: {
      version: '1.0.0',
      title: 'Teable App',
      description: 'Manage Data as easy as drink a cup of tea',
    },
    servers: [{ url: origin + '/api' }],
  });

  // 可选：生成代码示例
  if (snippet) {
    await generateCodeSamples(document);
  }

  return document;
}
```

#### 5. Axios 实例配置

[packages/openapi/src/axios.ts](packages/openapi/src/axios.ts)

Axios 实例支持：
- 默认 baseURL: `/api`
- 全局响应拦截器（错误处理）
- 服务端 AsyncLocalStorage 支持（请求隔离）
- `configApi()` 配置函数（token、endpoint、undo/redo）

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

## 第二层：NestJS 后端 - 使用 Schema 进行验证

### ZodValidationPipe

[apps/nestjs-backend/src/zod.validation.pipe.ts](apps/nestjs-backend/src/zod.validation.pipe.ts)

自定义 NestJS Pipe，使用 Zod Schema 对请求参数进行运行时验证：

```typescript
@Injectable()
export class ZodValidationPipe implements PipeTransform {
  constructor(private readonly schema: unknown) {}

  public transform(value: unknown, _metadata: ArgumentMetadata): unknown {
    const result = (this.schema as z.Schema).safeParse(value);
    if (!result.success) {
      throw new BadRequestException(fromZodError(result.error).message);
    }
    return result.data;
  }
}
```

### Controller 中的使用

[apps/nestjs-backend/src/features/record/open-api/record-open-api.controller.ts](apps/nestjs-backend/src/features/record/open-api/record-open-api.controller.ts)

```typescript
import { getRecordQuerySchema } from '@teable/openapi';
import type { IGetRecordQuery, IRecord } from '@teable/openapi';
import { ZodValidationPipe } from '../../../zod.validation.pipe';

@Controller('api/table/:tableId/record')
export class RecordOpenApiController {
  @Permissions('record|read')
  @Get(':recordId')
  async getRecord(
    @Param('tableId') tableId: string,
    @Param('recordId') recordId: string,
    @Query(new ZodValidationPipe(getRecordQuerySchema)) query: IGetRecordQuery
  ): Promise<IRecord> {
    return await this.recordService.getRecord(tableId, recordId, query, true, true);
  }
}
```

**关键点**：
- 从 `@teable/openapi` 导入 Zod Schema 和类型
- 使用 `ZodValidationPipe(schema)` 对请求参数进行验证
- 类型标注使用 `z.infer<typeof schema>` 生成的类型
- 路由路径与 OpenAPI 定义保持一致

### Swagger 文档接入

[apps/nestjs-backend/src/swagger.ts](apps/nestjs-backend/src/swagger.ts)

```typescript
import { getOpenApiDocumentation } from '@teable/openapi';
import { SwaggerModule } from '@nestjs/swagger';
import { RedocModule } from 'nestjs-redoc';

export async function setupSwagger(app: INestApplication, publicOrigin: string, enabledSnippet: boolean) {
  const openApiDocumentation = await getOpenApiDocumentation({
    origin: publicOrigin,
    snippet: enabledSnippet,
  });

  // 写入 JSON 文件
  const jsonString = JSON.stringify(openApiDocumentation);
  fs.writeFileSync(path.join(__dirname, '/openapi.json'), jsonString);
  
  // Swagger UI
  SwaggerModule.setup('/docs', app, openApiDocumentation as OpenAPIObject);
  
  // Redoc UI
  await RedocModule.setup('/redocs', app, openApiDocumentation as OpenAPIObject, redocOptions);
}
```

## 第三层：@teable/sdk - 前端 SDK 封装

SDK 直接使用 `@teable/openapi` 导出的类型和 API 函数，在此基础上提供更高级的抽象。

### React Query 封装

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

### Mutation 操作封装

[packages/sdk/src/hooks/use-record-operations.ts](packages/sdk/src/hooks/use-record-operations.ts)

```typescript
import { useMutation } from '@tanstack/react-query';
import type { ICreateRecordsRo, IUpdateRecordRo } from '@teable/openapi';
import {
  createRecords as createRecordsApi,
  updateRecord as updateRecordApi,
} from '@teable/openapi';

export const useRecordOperations = () => {
  const { mutateAsync: createRecords } = useMutation({
    mutationFn: ({ tableId, recordsRo }: { tableId: string; recordsRo: ICreateRecordsRo }) =>
      createRecordsApi(tableId, recordsRo),
  });

  const { mutateAsync: updateRecord } = useMutation({
    mutationFn: ({ tableId, recordId, recordRo }: {
      tableId: string;
      recordId: string;
      recordRo: IUpdateRecordRo;
    }) => updateRecordApi(tableId, recordId, recordRo),
  });

  return { createRecords, updateRecord };
};
```

### Model 层包装

SDK 还提供了领域模型包装，将原始 API 响应转换为具有行为的对象：

```typescript
// packages/sdk/src/model/record/record.ts
export class Record {
  id: string;
  fields: Record<string, unknown>;
  createdTime: string;
  lastModifiedTime: string;

  constructor(data: IRecord) {
    this.id = data.id;
    this.fields = data.fields;
    this.createdTime = data.createdTime;
    this.lastModifiedTime = data.lastModifiedTime;
  }

  getCellValue(fieldId: string) {
    return this.fields[fieldId];
  }

  // 更多业务方法...
}

export const createRecordInstance = (data: IRecord): Record => {
  return new Record(data);
};
```

## 完整数据流与接力过程

### 1. 开发阶段：API 定义

1. 开发者在 `packages/openapi/src/[feature]/` 下定义新 API
2. 编写 Zod Schema（请求/响应）
3. 注册 RouteConfig（添加 summary、description、tags 等元数据）
4. 导出对应的 axios 调用函数
5. 导出类型定义

### 2. 后端集成

1. Controller 从 `@teable/openapi` 导入 schema 和类型
2. 使用 `ZodValidationPipe(schema)` 进行参数验证
3. 实现对应的 Service 业务逻辑
4. 路由路径必须与 OpenAPI 定义一致

### 3. 文档生成

1. 后端启动时调用 `getOpenApiDocumentation()`
2. 遍历所有已注册的 RouteConfig
3. 使用 `@asteasolutions/zod-to-openapi` 转换为 OpenAPI 3.0 JSON
4. 挂载到 `/docs` (Swagger UI) 和 `/redocs` (Redoc)

### 4. 前端使用

1. SDK 从 `@teable/openapi` 导入 API 函数和类型
2. 使用 React Query 包装为 hooks
3. 提供 Model 层进行领域对象包装
4. UI 组件直接使用 SDK hooks

## 设计优势

### 1. 单一数据源 (Single Source of Truth)

所有 API 元数据（schema、路径、描述、类型）都在 `@teable/openapi` 中定义一次，后端、文档、前端共享使用。

### 2. 类型安全贯穿全链路

```
Zod Schema → TypeScript类型 → 后端验证 → 前端SDK → UI组件
```

类型定义一次，全程自动推导，避免手动类型转换和重复定义。

### 3. 文档与代码自动同步

OpenAPI 文档直接从代码生成，确保文档始终与实际 API 一致。修改代码时文档自动更新。

### 4. 渐进式验证

- **编译时**: TypeScript 类型检查
- **运行时（后端）**: Zod Schema 验证
- **运行时（前端）**: Axios 响应类型注解

### 5. 易于维护

新增 API 只需在 `@teable/openapi` 中定义一次，后端和前端自动获得类型安全的调用方式。

## 潜在改进点

1. **自动生成 SDK**: 目前 SDK 仍需手动封装 hooks，可考虑基于 OpenAPI 文档自动生成
2. **版本管理**: 缺少 API 版本控制机制
3. **Mock 数据**: 可基于 Zod Schema 自动生成 mock 数据用于测试
4. **变更检测**: 可添加 CI 检查 API 变更是否破坏向后兼容性

## 总结

Teable 的 OpenAPI + SDK 机制是一套设计精良的全栈类型安全方案。通过以 Zod Schema 为核心，配合 zod-to-openapi、自定义 ValidationPipe 和分层封装，实现了 API 定义、文档生成、后端验证、前端调用的完美协同。这种架构大幅减少了前后端联调成本，确保了系统的一致性和可维护性。

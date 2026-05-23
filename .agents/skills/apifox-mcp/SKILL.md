---
name: apifox-mcp
description: '调用 Apifox MCP 处理 projectId 查询项目结构信息、根据 projectId 和接口 URL 查找接口、查询项目内数据模型、创建或更新 HTTP 接口。适用于 Apifox 项目检索、接口定位、数据模型检索、文档维护、接口新增。关键词: apifox, mcp, projectId, url, path, endpoint, schema, data model, create http endpoint.'
---

# Apifox MCP Skill

## 适用场景

在以下场景使用本 skill：

- 已知 projectId，需要读取某个 Apifox 项目的完整结构信息
- 已知 projectId 和接口 URL，需要定位对应 HTTP 接口
- 已知 projectId，需要查询项目中的数据模型(schema)
- 需要在指定项目下创建新的 HTTP 接口
- 需要补充接口文档、修正接口路径、更新接口状态
- 需要让接口字段正确引用项目内已有的数据模型

## 调用原则

- 先确认 projectId，再做接口级操作
- 不要编造 folderId、responsibleId、serverId、httpApiId 这类结构化 ID
- 创建或更新接口前，优先先读取项目结构，确保目录 ID 和上下文正确
- `mcp_apifox_getProjectSummary` 中的 `schemaFolderCount` 只能说明“schema 目录数量”，不能据此断言项目里没有数据模型
- 查询项目内数据模型时，优先通过 OpenAPI 端点查询 `api-schemas` 列表和详情，不要只看项目结构摘要
- 接口字段引用数据模型时，`$ref` 必须使用 schema ID，例如 `#/definitions/277244771`，不要写成 `#/definitions/resultStatus` 这类模型名引用
- 配置 Mock 时，优先使用字段级 `x-apifox-mock`；只有字段级动态值不够用时，再考虑 Mock 期望或 mockScript
- 枚举模型引用字段通常不需要额外配置 Mock；例如统一返回中的 `code` 应直接引用 `resultStatus`，不要再叠加字段级 mock
- 当工具没有“按 URL 直接搜索接口”的固定能力时，要明确告知限制，不要假装能一步命中

## 工作流 1：通过 projectId 查询特定项目的所有信息

这里的“所有信息”优先指项目的完整结构元数据，包括分支、模块、接口目录、Schema 目录、测试分组等结构信息。

### 步骤

1. 如果用户没有给出 projectId，先调用 `mcp_apifox_listAccessibleProjects` 获取可访问项目列表。
2. 已知 projectId 后，调用 `mcp_apifox_getProjectSummary`。
3. 如果用户指定 branchId，则把 branchId 一并传入；否则默认主分支。
4. 输出时优先整理以下内容：
  - 项目基础信息
  - 分支信息
  - 模块/目录结构
  - HTTP 接口目录结构
  - Schema 目录结构
  - 测试分组/测试结构
5. 如果用户后续需要项目内具体数据模型列表或某个模型详情，不要停在目录层，继续走“工作流 4”。
6. 如果用户后续需要某个具体接口的完整定义，再继续调用 `mcp_apifox_getHttpEndpoint` 获取接口详情。

### 示例

```json
{
  "tool": "mcp_apifox_getProjectSummary",
  "arguments": {
    "projectId": 123456
  }
}
```

带分支示例：

```json
{
  "tool": "mcp_apifox_getProjectSummary",
  "arguments": {
    "projectId": 123456,
    "branchId": 7890
  }
}
```

### 输出建议

- 如果用户说“查项目所有信息”，优先返回结构化摘要，不要原样倾倒整个响应
- 如果用户继续追问某个目录、数据模型或接口，再下钻到对应实体详情

## 工作流 2：配合 projectId，通过接口 URL 地址找到对应接口

### 先说明能力边界

当前固定暴露的 Apifox MCP 工具中，有：

- `mcp_apifox_getProjectSummary`
- `mcp_apifox_getHttpEndpoint`
- `mcp_apifox_createHttpEndpoint`
- `mcp_apifox_updateHttpEndpoint`

但没有一个已经固定暴露的“仅凭 projectId + URL 直接搜索并返回接口 ID”的专用工具。

因此，推荐使用下面的两段式流程。

### 推荐流程

1. 先调用 `mcp_apifox_getProjectSummary`，确认项目、分支、目录上下文正确。
2. 如果当前会话额外启用了 Apifox 的项目内容搜索或实体检索类工具，用 `projectId + url/path` 先搜候选接口。
3. 拿到候选 `httpApiId` 后，再调用 `mcp_apifox_getHttpEndpoint` 获取详情。
4. 对比接口的 `method` 和 `path`，确认是否与用户给出的 URL 完全匹配。
5. 如果当前会话没有启用项目内容搜索类工具，要明确告诉用户：当前只能在“已知接口 ID”时精确获取详情，不能保证仅凭 URL 一步命中。

### 已知接口 ID 时的详情查询示例

```json
{
  "tool": "mcp_apifox_getHttpEndpoint",
  "arguments": {
    "pathParams": {
      "projectId": 123456,
      "httpApiId": 10001
    },
    "headers": {
      "X-Project-Id": "123456"
    }
  }
}
```

### skill 执行时的建议话术

- 如果只有 URL 没有接口 ID：先说明“当前固定工具集中没有按 URL 直查接口的专用工具，我会先读取项目结构，再尝试使用当前会话可用的检索类 Apifox 工具定位候选接口。”
- 如果已经拿到候选接口：再调用详情工具做二次确认，而不是直接断言命中

## 工作流 3：Apifox MCP 创建新接口

固定工具里已经暴露了 `mcp_apifox_createHttpEndpoint`，可以直接创建 HTTP 接口。

### 最小可用创建方式

创建接口至少建议提供：

- `headers.X-Project-Id`
- `body.method`
- `body.path`

如果想让接口落到正确目录，应该额外提供 `body.folderId`。`folderId` 应先通过项目结构信息确认，不要编造。

### 最小示例

```json
{
  "tool": "mcp_apifox_createHttpEndpoint",
  "arguments": {
    "headers": {
      "X-Project-Id": "123456"
    },
    "body": {
      "name": "获取用户详情",
      "method": "GET",
      "path": "/api/users/{id}",
      "type": "http"
    }
  }
}
```

### 推荐完整示例

```json
{
  "tool": "mcp_apifox_createHttpEndpoint",
  "arguments": {
    "headers": {
      "X-Project-Id": "123456"
    },
    "body": {
      "name": "创建订单",
      "method": "POST",
      "path": "/api/orders",
      "folderId": 2001,
      "status": "designing",
      "description": "创建订单接口",
      "type": "http",
      "tags": "order,create"
    }
  }
}
```

### 创建前的推荐步骤

1. 先调用 `mcp_apifox_getProjectSummary` 获取目录结构
2. 确认接口应放入哪个目录，拿到 `folderId`
3. 确认路径和方法是否已存在，避免重复创建
4. 调用 `mcp_apifox_createHttpEndpoint`
5. 如需补充响应、请求体、鉴权、示例，再按需补字段，或后续调用 `mcp_apifox_updateHttpEndpoint`
6. 如果响应或请求字段需要引用项目内已有数据模型，先走“工作流 4”查到 schema ID，再写 `$ref: "#/definitions/{schemaId}"`

### 创建或补全响应时的引用规则

- 需要引用数据模型时，先查 schema ID，再写 `$ref`
- `code` 这类字段如果要引用 `resultStatus`，正确格式是 `"$ref": "#/definitions/277244771"`
- 不要写成 `"$ref": "#/definitions/resultStatus"`，这会在 Apifox 中变成坏引用
- `responses` 字段本身是 JSON 字符串数组，但每个响应项里的 `jsonSchema` 应写成 JSON 对象结构，而不是再包一层字符串

### 统一返回模型

当项目约定统一返回结构时，优先复用同一个顶层包裹模型，不要每个接口各写一套风格。

当前已确认的统一返回规范如下：

- `code`：引用 `resultStatus` 枚举模型
- `msg`：`string`
- `data`：泛型数据位，根据接口真实业务变化，可能是 `string`、`object`、`integer`、`boolean`、`array`，也可能是更深层嵌套对象

### 统一返回模型的约束

- 顶层结构固定为 `code`、`msg`、`data`
- `code` 始终写成对 `resultStatus` 的 `$ref`
- `msg` 默认为字符串，不要改成对象或数组
- `data` 的类型由当前接口语义决定，不要因为项目有统一返回包装，就把所有接口都强行写成 `string`
- 当 `data` 是对象或嵌套结构时，应在 `data` 下继续细化 `properties`
- 当 `data` 是数组时，应把 `data.type` 写成 `array`，并补 `items`

### 统一返回模型示例

#### data 为 string

```json
{
  "type": "object",
  "properties": {
    "code": {
      "$ref": "#/definitions/277244771"
    },
    "msg": {
      "type": "string"
    },
    "data": {
      "type": "string"
    }
  },
  "required": ["code", "msg", "data"]
}
```

#### data 为 object

```json
{
  "type": "object",
  "properties": {
    "code": {
      "$ref": "#/definitions/277244771"
    },
    "msg": {
      "type": "string"
    },
    "data": {
      "type": "object",
      "properties": {
        "id": {
          "type": "integer"
        },
        "name": {
          "type": "string"
        }
      }
    }
  },
  "required": ["code", "msg", "data"]
}
```

#### data 为 integer 或 boolean

```json
{
  "type": "object",
  "properties": {
    "code": {
      "$ref": "#/definitions/277244771"
    },
    "msg": {
      "type": "string"
    },
    "data": {
      "type": "integer"
    }
  },
  "required": ["code", "msg", "data"]
}
```

```json
{
  "type": "object",
  "properties": {
    "code": {
      "$ref": "#/definitions/277244771"
    },
    "msg": {
      "type": "string"
    },
    "data": {
      "type": "boolean"
    }
  },
  "required": ["code", "msg", "data"]
}
```

#### data 为数组或嵌套对象

```json
{
  "type": "object",
  "properties": {
    "code": {
      "$ref": "#/definitions/277244771"
    },
    "msg": {
      "type": "string"
    },
    "data": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {
            "type": "integer"
          },
          "profile": {
            "type": "object",
            "properties": {
              "nickname": {
                "type": "string"
              }
            }
          }
        }
      }
    }
  },
  "required": ["code", "msg", "data"]
}
```

### 响应引用模型示例

```json
{
  "tool": "mcp_apifox_updateHttpEndpoint",
  "arguments": {
    "pathParams": {
      "http_api_id": 10001
    },
    "headers": {
      "X-Project-Id": "123456"
    },
    "body": {
      "responses": "[{\"name\":\"成功\",\"code\":\"200\",\"contentType\":\"json\",\"jsonSchema\":{\"type\":\"object\",\"properties\":{\"code\":{\"$ref\":\"#/definitions/277244771\"},\"msg\":{\"type\":\"string\"},\"data\":{\"type\":\"string\"}},\"required\":[\"code\",\"msg\",\"data\"]}}]"
    }
  }
}
```

## 工作流 4：查询项目内数据模型(schema)

当用户提到“数据模型”“schema”“resultStatus”“字段要引用已有模型”这类诉求时，不要只看 `getProjectSummary`，应直接查询项目的数据模型列表。

### 推荐流程

1. 先确认 `projectId`，必要时先调用 `mcp_apifox_getProjectSummary` 校验项目上下文。
2. 调用 `mcp_apifox_listOpenApiEndpoints`，用 `schema` 或 `api-schemas` 作为关键词，确认当前会话已暴露的数据模型查询端点。
3. 使用 `GET /api/v1/api-schemas` 对应的工具获取项目数据模型列表。
4. 在返回结果中按 `name` 匹配目标模型，拿到 `schemaId`。
5. 如需确认模型的真实 `jsonSchema`、枚举值或默认值，再调用 `GET /api/v1/api-schemas/{schema_id}` 获取详情。
6. 如果后续要让接口字段引用该模型，写成 `$ref: "#/definitions/{schemaId}"`。

### 当前会话中已验证可用的模型查询端点

- `mcp_apifox_listOpenApiEndpoints`：用于发现 OpenAPI 端点
- `GET /api/v1/api-schemas`，工具 ID `9bbb5b98b1a6a909`：获取项目数据模型列表
- `GET /api/v1/api-schemas/{schema_id}`，工具 ID `9b7a91d2f33fc687`：获取指定数据模型详情

### 查询数据模型列表示例

```json
{
  "tool": "mcp_apifox_executeOpenApi",
  "arguments": {
    "id": "9bbb5b98b1a6a909",
    "pathParams": {},
    "queryParams": {},
    "headers": {
      "X-Project-Id": "123456"
    }
  }
}
```

### 查询单个数据模型详情示例

```json
{
  "tool": "mcp_apifox_executeOpenApi",
  "arguments": {
    "id": "9b7a91d2f33fc687",
    "pathParams": {
      "schema_id": "277244771"
    },
    "queryParams": {},
    "headers": {
      "X-Project-Id": "123456"
    }
  }
}
```

### 输出建议

- 输出时至少说明模型名称、schema ID、jsonSchema 类型、关键枚举值或默认值
- 如果 `schemaFolderCount` 为 0，不要直接说“项目没有数据模型”，要继续查询 `api-schemas`
- 如果用户要引用某个模型，回复中要同时给出模型名和 schema ID，避免后续写错 `$ref`

## 更新已有接口

如果已经知道接口 ID，可以用 `mcp_apifox_updateHttpEndpoint` 更新名称、方法、路径、状态、描述、参数和响应定义。

### 示例

```json
{
  "tool": "mcp_apifox_updateHttpEndpoint",
  "arguments": {
    "pathParams": {
      "http_api_id": 10001
    },
    "headers": {
      "X-Project-Id": "123456"
    },
    "body": {
      "name": "更新后的接口名称",
      "path": "/api/users/{id}",
      "status": "developing",
      "description": "补充后的接口说明"
    }
  }
}
```

### 更新接口时的模型引用规则

1. 先查数据模型，拿到 schema ID。
2. 更新 `responses` 或 `requestBody` 时，优先保留现有字段结构，只替换需要引用模型的那一项。
3. 使用 `"$ref": "#/definitions/{schemaId}"`，不要使用模型名。
4. 如果 Apifox 页面报 `Reference not found`，优先检查引用是否写成了模型名，或 schema ID 是否来自当前项目。

## 工作流 5：配置字段级 Mock

当用户要求“给接口配 Mock”“给某几个字段生成动态值”“图片返回随机在线地址”时，优先使用字段级 `x-apifox-mock`，不要默认上 mockScript。

### 推荐流程

1. 先读取接口详情，确认当前 `responses.jsonSchema` 的真实结构。
2. 只给需要 Mock 的字段补 `x-apifox-mock`，不要改动无关字段。
3. 如果字段本身已经引用枚举或数据模型，先判断是否真的需要 Mock。
4. 更新后立即重新读取接口详情，确认 `x-apifox-mock` 已落到对应字段。

### 字段级 Mock 规则

- 优先使用简单值或 Faker 动态值，例如 `success`、`{{$person.firstName}}`
- 图片地址优先使用 Faker 的在线图片 URL，而不是固定假地址
- 如果字段是枚举模型引用，例如 `code -> resultStatus`，默认不额外写 `x-apifox-mock`
- `msg` 这类稳定字段可以直接写固定值，如 `success`
- 只有字段级 Mock 无法满足时，再考虑 Mock 期望或 mockScript

### 常用写法

- 固定字符串：`success`
- 姓名：`{{$person.firstName}}`
- 邮箱：`{{$internet.email}}`
- 在线图片：`{{$image.urlPicsumPhotos(width=640,height=480)}}`

### 统一返回模型下的 Mock 建议

- `code`：不写字段级 Mock，保留对 `resultStatus` 的引用
- `msg`：通常可写固定值 `success`
- `data`：按真实业务类型补 Mock

如果 `data` 是图片 URL，推荐这样写：

```json
{
  "type": "object",
  "properties": {
    "code": {
      "$ref": "#/definitions/277244771"
    },
    "msg": {
      "type": "string",
      "x-apifox-mock": "success"
    },
    "data": {
      "type": "string",
      "x-apifox-mock": "{{$image.urlPicsumPhotos(width=640,height=480)}}"
    }
  },
  "required": ["code", "msg", "data"]
}
```

### 不推荐的做法

- 给 `code` 这种枚举引用字段再叠加随机数 Mock
- 图片 URL 写死成单个固定值，导致每次 Mock 都一样
- 在只需字段级动态值时直接上 mockScript，增加排查成本

## 推荐执行顺序

### 查询项目

1. `mcp_apifox_listAccessibleProjects`（可选）
2. `mcp_apifox_getProjectSummary`

### 按 URL 找接口

1. `mcp_apifox_getProjectSummary`
2. 当前会话如有项目内容搜索类 Apifox 工具，则先做搜索
3. `mcp_apifox_getHttpEndpoint` 对候选接口逐个确认

### 创建接口

1. `mcp_apifox_getProjectSummary`
2. 确认 `folderId`
3. 如果涉及数据模型引用，先走“工作流 4”拿到 `schemaId`
4. `mcp_apifox_createHttpEndpoint`
5. 必要时 `mcp_apifox_updateHttpEndpoint`

### 查询数据模型

1. `mcp_apifox_listOpenApiEndpoints` 发现 `api-schemas` 相关端点
2. `mcp_apifox_executeOpenApi` 调用 `GET /api/v1/api-schemas`
3. 必要时 `mcp_apifox_executeOpenApi` 调用 `GET /api/v1/api-schemas/{schema_id}`

### 配置字段级 Mock

1. `mcp_apifox_getHttpEndpoint` 读取当前接口详情
2. `mcp_apifox_updateHttpEndpoint` 只更新目标响应字段的 `x-apifox-mock`
3. `mcp_apifox_getHttpEndpoint` 回读确认

## 输出要求

- 对外回答时，明确区分“项目结构信息”和“单个接口详情”
- 对外回答时，明确区分“schema 目录结构”和“数据模型列表/详情”
- 对外回答时，如果接口采用统一返回模型，要明确说明 `code`、`msg`、`data` 的固定约束和 `data` 的实际类型
- 如果工具能力不足以通过 URL 直接定位接口，要直接说明限制
- 如果缺少 projectId、folderId、httpApiId、schemaId 等关键 ID，先补齐上下文，不要猜
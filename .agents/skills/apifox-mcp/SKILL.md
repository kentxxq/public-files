---
name: apifox-mcp
description: '调用 Apifox MCP 处理 projectId 查询项目结构信息、根据 projectId 和接口 URL 查找接口、创建或更新 HTTP 接口。适用于 Apifox 项目检索、接口定位、文档维护、接口新增。关键词: apifox, mcp, projectId, url, path, endpoint, create http endpoint.'
---

# Apifox MCP Skill

## 适用场景

在以下场景使用本 skill：

- 已知 projectId，需要读取某个 Apifox 项目的完整结构信息
- 已知 projectId 和接口 URL，需要定位对应 HTTP 接口
- 需要在指定项目下创建新的 HTTP 接口
- 需要补充接口文档、修正接口路径、更新接口状态

## 调用原则

- 先确认 projectId，再做接口级操作
- 不要编造 folderId、responsibleId、serverId、httpApiId 这类结构化 ID
- 创建或更新接口前，优先先读取项目结构，确保目录 ID 和上下文正确
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
   - Schema 或数据模型目录
   - 测试分组/测试结构
5. 如果用户后续需要某个具体接口的完整定义，再继续调用 `mcp_apifox_getHttpEndpoint` 获取接口详情。

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
- 如果用户继续追问某个目录或接口，再下钻到对应实体详情

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
3. `mcp_apifox_createHttpEndpoint`
4. 必要时 `mcp_apifox_updateHttpEndpoint`

## 输出要求

- 对外回答时，明确区分“项目结构信息”和“单个接口详情”
- 如果工具能力不足以通过 URL 直接定位接口，要直接说明限制
- 如果缺少 projectId、folderId、httpApiId 等关键 ID，先补齐上下文，不要猜
---
slug: MCP-Protocol
title: MCP协议
description: MCP 协议全解读：Legacy（2025-11-25）与 Modern（2026-07-28）的架构、功能与 Java/Python 代码案例
published: 2026-11-27
updated: 2026-11-27
tags:
  - MCP
  - AI
  - Agent
category: MCP
draft: true
---

# 概述

[模型上下文协议](https://modelcontextprotocol.io/)（Model Context Protocol，简称 MCP）是一项开放协议，用于把 LLM 应用与外部数据源、工具和能力无缝集成。

MCP 采用 **JSON-RPC 2.0** 作为统一的消息格式，参与方分为三类：

| 角色 | 说明 |
| --- | --- |
| **Host（宿主）** | 发起连接的 LLM 应用，如 IDE、聊天客户端、Agent 运行时 |
| **Client（客户端）** | 宿主内部的连接器，与某个 Server 维持一对一通信 |
| **Server（服务端）** | 提供上下文与能力的服务，如工具网关、数据库、文件系统 |

协议在服务端侧定义了三个核心原语：**Resources（资源）**、**Prompts（提示词模板）**、**Tools（工具）**；在客户端侧定义了 **Elicitation（向用户征求输入）**，Legacy 时代还包含 **Sampling（请求客户端代为调用 LLM）** 与 **Roots（提供文件系统根目录）**。此外还有进度、取消、错误上报等通用工具。

传输层支持两种方式：**stdio**（本地子进程，按行传输 JSON-RPC）与 **Streamable HTTP**（远程部署，单端点 `POST`，响应可为单条 JSON 或按需开启的 SSE 流）。

以 **2026-07-28** 修订版为界，协议被官方划分为两个时代：**Legacy（传统）** 与 **Modern（现代）**。前者是**有状态的会话模型**（`initialize` 握手 + 会话 ID），后者是**无状态的请求-响应模型**（每个请求自带版本、身份与能力）。本文按“概述 → 版本区分 → 架构对比 → Legacy → Modern”的顺序展开，代码案例只使用 **Java** 与 **Python**。

# 版本区分

## 修订版本时间线

| 修订版本 | 时代 | 关键变化 |
| --- | --- | --- |
| 2024-11-05 | Legacy | 首个正式版本；JSON-RPC 2.0；stdio 与 HTTP+SSE 双端点传输 |
| 2025-03-26 | Legacy | 引入 **Streamable HTTP** 取代 HTTP+SSE；OAuth 2.1；工具注解；音频内容；补全能力 |
| 2025-06-18 | Legacy | 移除 JSON-RPC 批量请求；后续 HTTP 请求必须携带 `MCP-Protocol-Version`；结构化工具输出；引入 Elicitation |
| 2025-11-25 | Legacy | **Legacy 时代最后一版**；OIDC Discovery、图标、URL 模式 Elicitation、Sampling 支持工具调用、实验性 Tasks；默认 JSON Schema 2020-12 |
| 2026-07-28 | Modern | **Modern 时代第一版**；无状态核心、移除握手与会话、MRTR、Header 路由、结果缓存、扩展框架；Sampling/Roots/Logging 进入弃用窗口 |

## 简要区分

| 时代 | 定义特征 | 涵盖版本 |
| --- | --- | --- |
| **Legacy（传统）** | 通过 `initialize` **握手**来建立会话，依赖会话状态 | **2025-11-25 及更早**的所有版本 |
| **Modern（现代）** | 无握手，每个请求**自带**版本、身份和能力等元数据 | **2026-07-28 及以后**的版本 |
| **Dual-era（双时代）** | 同时支持 Legacy 和 Modern 两种范式的实现 | SDK 层面的兼容模式（如 Python SDK v2、Java SDK 2.x） |

## 详细架构对比

| 对比维度 | Legacy (2025-11-25 及更早) | Modern (2026-07-28) | 核心影响 |
| --- | --- | --- | --- |
| **连接模型** | `initialize` **握手**建立会话 | **取消握手**，每个请求独立自描述 | 协议行为更像标准无状态 HTTP 服务 |
| **状态管理** | 依赖 `Mcp-Session-Id` 请求头绑定特定服务器实例 | **移除协议级会话**和 `Mcp-Session-Id`，跨调用状态通过显式**业务句柄**（如 `jobId`）传递 | 支持水平扩展、普通轮询负载均衡，适配 Serverless/边缘环境 |
| **能力协商** | 在 `initialize` 响应中**一次性协商** | 每个请求通过 `_meta` 字段**自带**能力声明，辅以 `server/discover` 方法供客户端查询 | 协商粒度更细，更灵活 |
| **Server→Client 交互** | 可通过双向流**主动回调**（如 Sampling, Roots） | **禁止**服务器主动发起请求，改为 **MRTR（多轮往返请求）** 模式，服务器在响应中返回 `input_required`，由客户端重试 | 不再依赖持久双向连接 |
| **传输与连接** | 支持 **SSE** 长连接、断线续传（`Last-Event-ID`） | **移除** SSE 续传，默认使用无状态 **Streamable HTTP**，用 `subscriptions/listen` 替代旧式长连接 GET 端点 | 连接模型简化，更易于网关和代理处理 |
| **网关治理** | 网关需**解析 JSON-RPC 请求体**才能路由、鉴权 | HTTP Header 新增 **`Mcp-Method`**、**`Mcp-Name`**，网关可直接基于 Header 进行**路由、限流、鉴权** | 治理更高效，无需解析请求体 |
| **缓存机制** | 无标准化缓存 | 列表接口返回 **`ttlMs`**、**`cacheScope`** 字段，支持客户端缓存与确定性排序 | 减少延迟与请求量，优化 Prompt Cache |
| **长任务处理** | 无统一标准，依赖长连接或自定义实现（2025-11-25 有实验性 Tasks） | 通过 **Tasks 扩展**（`taskId`、轮询、恢复）处理持续性的后端工作任务 | 更适合 CI、仿真等长耗时场景 |
| **可观测性** | 使用 MCP 自有 **Logging** 通道 | Logging 进入弃用窗口，**标准化 OpenTelemetry** 上下文传播 | 更容易融入企业现有监控体系 |
| **扩展机制** | 新功能倾向于直接加入核心协议 | 引入 **Extension Framework**，核心保持稳定，能力（如 MCP Apps, Tasks）按需扩展 | 核心协议更精简，生态扩展性更强 |
| **功能弃用** | 包含 **Sampling、Roots、Logging** 等特性 | 上述特性进入 **至少 12 个月**的弃用窗口，新实现不应再支持 | 引导生态向新范式迁移 |
| **JSON Schema** | `draft-07`（2025-11-25 起默认 2020-12） | `2020-12`，并允许其全部关键字 | 支持更丰富的模式定义能力 |
| **结果信封** | 直接返回结果对象，无统一类型字段 | 所有结果必带 **`resultType`**（`complete` / `input_required`） | 客户端可统一识别“完成”与“待补充输入” |
| **错误码** | 资源不存在使用 `-32002`，其余实现自定义 | `-32002` 改为 `-32602`；新增 `-32020`/`-32021`/`-32022` 规范错误码 | 错误语义标准化，便于网关与 SDK 处理 |
| **方法集合** | 包含 `ping`、`logging/setLevel`、`resources/subscribe` | 移除上述方法，统一为 `subscriptions/listen` | 方法面收敛，订阅模型统一 |

## 如何判断对方属于哪个时代

官方在 [Versioning 页面](https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning) 给出了标准判定方式：

- **stdio**：先发送 `server/discover` 探测，返回可识别的 Modern 错误（如 `UnsupportedProtocolVersionError`）说明是 Modern；其他错误或超时说明是 Legacy，回退到 `initialize`。
- **Streamable HTTP**：直接发送一个 Modern 请求；若返回 `400` 且响应体是可识别的 Modern JSON-RPC 错误，则是 Modern；否则视为 Legacy，回退握手（必要时再回退到已弃用的 HTTP+SSE）。
- **Dual-era 服务端**：请求携带 Modern `_meta` 则按无状态语义处理；收到 `initialize` 则按 Legacy 语义处理，同一端点可同时服务两种客户端。

时代属性属于“服务端”，而不是单个请求，客户端应缓存判定结果。

# Legacy（2025-11-25）：协议结构、功能与代码案例

## 协议结构

### 分层模型

| 层 | 内容 |
| --- | --- |
| 传输层 | stdio、Streamable HTTP（会话式）、已弃用的 HTTP+SSE |
| 消息层 | JSON-RPC 2.0 请求 / 响应 / 通知 |
| 生命周期层 | `initialize` → `notifications/initialized` → 正常操作 → 关闭 |
| 能力层 | 客户端能力（roots/sampling/elicitation/tasks）与服务端能力（tools/resources/prompts/logging/completions/tasks） |

### 生命周期：initialize 握手

初始化阶段 **必须** 是客户端与服务端的第一次交互，用于协商协议版本、交换能力与身份信息。

客户端请求：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "roots": { "listChanged": true },
      "sampling": {},
      "elicitation": { "form": {}, "url": {} }
    },
    "clientInfo": { "name": "demo-client", "version": "1.0.0" }
  }
}
```

服务端响应（返回自身支持的最新版本与能力）：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "tools": { "listChanged": true },
      "resources": { "subscribe": true, "listChanged": true },
      "prompts": { "listChanged": true },
      "logging": {}
    },
    "serverInfo": { "name": "demo-server", "version": "1.0.0" },
    "instructions": "可选的服务端使用说明"
  }
}
```

握手完成后，客户端发送就绪通知：

```json
{ "jsonrpc": "2.0", "method": "notifications/initialized" }
```

要点：

- **版本协商**：客户端发送自己支持的版本；服务端支持则原样返回，不支持则返回自己支持的最新版本；客户端再不支持则应断开。
- **能力协商**：只有双方声明并协商一致的能力才能使用。
- **HTTP 会话**：服务端可在 `initialize` 响应中下发 `Mcp-Session-Id`，客户端后续请求必须携带；后续请求还需携带 `MCP-Protocol-Version` 头（2025-06-18 起）。
- **关闭**：没有专门的关闭消息，stdio 关闭输入流 / 终止进程，HTTP 关闭连接。

### 传输层（2025-11-25）

| 传输 | 特征 |
| --- | --- |
| stdio | 本地子进程；stdin/stdout 按行传输 JSON-RPC；日志写 stderr |
| Streamable HTTP | 单端点 `POST`；响应为单条 JSON 或 SSE 流；可选 `GET` 长流用于服务端推送；`Mcp-Session-Id` 绑定会话；`Last-Event-ID` 支持断线续传 |
| HTTP+SSE | 已弃用：`GET /sse` + `POST /messages` 双端点 |

### 方法清单（2025-11-25）

| 类别 | 方法 | 方向 |
| --- | --- | --- |
| 生命周期 | `initialize`、`notifications/initialized` | Client → Server |
| 工具 | `tools/list`、`tools/call` | Client → Server |
| 资源 | `resources/list`、`resources/templates/list`、`resources/read`、`resources/subscribe`、`resources/unsubscribe` | Client → Server |
| 提示词 | `prompts/list`、`prompts/get` | Client → Server |
| 补全 | `completion/complete` | Client → Server |
| 日志 | `logging/setLevel` | Client → Server |
| 健康检查 | `ping` | 双向 |
| 采样 | `sampling/createMessage` | **Server → Client** |
| 引导 | `elicitation/create` | **Server → Client** |
| 根目录 | `roots/list`、`notifications/roots/list_changed` | **Server → Client** |
| 通知 | `notifications/tools/list_changed`、`notifications/resources/list_changed`、`notifications/prompts/list_changed`、`notifications/resources/updated`、`notifications/progress`、`notifications/message`、`notifications/cancelled` | 双向 |
| 任务（实验） | `tasks/get`、`tasks/result`、`tasks/cancel`、`tasks/list` | Client → Server |

## 功能特性

| 能力 | 归属 | 说明 |
| --- | --- | --- |
| Tools | Server | 模型可调用的函数；`tools/list` 发现，`tools/call` 执行 |
| Resources | Server | 可读上下文数据；支持模板、订阅单个资源的变更 |
| Prompts | Server | 模板化消息与工作流 |
| Completions | Server | 参数补全建议 |
| Sampling | Client | 服务端请求客户端代为调用 LLM（2025-11-25 起支持 `tools`/`toolChoice`） |
| Elicitation | Client | 服务端向用户征求输入；2025-11-25 支持 **form** 与 **URL** 两种模式 |
| Roots | Client | 服务端查询客户端可访问的文件系统根目录 |
| Logging | Server | `logging/setLevel` 设置日志级别，`notifications/message` 推送日志 |
| Tasks | Both | 实验性长任务：创建任务、轮询、取结果、取消、列举 |
| 授权 | Client/Server | OAuth 2.1 + Resource Indicators；2025-11-25 增加 OIDC Discovery 与 Client ID Metadata Documents |
| 元数据 | Server | 2025-11-25 起工具/资源/提示词可携带 `icons` 等展示信息 |

## 代码案例

以下示例基于 **Spring AI + MCP Java SDK 1.x** 与 **官方 Python SDK 1.x**。Java 服务端端点为 `http://localhost:8080/mcp`，Python 服务端端点为 `http://localhost:8000/mcp`。

### Java 服务端（Spring AI）

引入 Streamable HTTP 的 MCP Server starter：

```xml
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.ai</groupId>
			<artifactId>spring-ai-bom</artifactId>
			<version>1.0.3</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>

<dependencies>
	<dependency>
		<groupId>org.springframework.ai</groupId>
		<artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
	</dependency>
</dependencies>
```

指定 Streamable HTTP 协议：

```yaml
spring:
  ai:
    mcp:
      server:
        name: legacy-order-server
        version: 1.0.0
        protocol: STREAMABLE   # STDIO / SSE / STREAMABLE / STATELESS
```

暴露工具（普通 Spring Bean，方法即 MCP Tool）：

```java
import org.springframework.ai.tool.ToolCallbackProvider;
import org.springframework.ai.tool.method.MethodToolCallbackProvider;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

@SpringBootApplication
public class LegacyServerApplication {

	@Component
	static class OrderTools {

		@Tool(description = "查询订单状态")
		public String queryOrder(@ToolParam(description = "订单号") String orderId) {
			return "order " + orderId + " -> SHIPPED";
		}
	}

	@Bean
	ToolCallbackProvider mcpTools(OrderTools orderTools) {
		return MethodToolCallbackProvider.builder().toolObjects(orderTools).build();
	}

	public static void main(String[] args) {
		SpringApplication.run(LegacyServerApplication.class, args);
	}
}
```

### Java 客户端（MCP Java SDK 1.x）

```java
import io.modelcontextprotocol.client.McpClient;
import io.modelcontextprotocol.client.McpSyncClient;
import io.modelcontextprotocol.client.transport.HttpClientStreamableHttpTransport;
import io.modelcontextprotocol.spec.McpSchema;

import java.util.Map;

public class LegacyClientDemo {

	public static void main(String[] args) {
		var transport = HttpClientStreamableHttpTransport.builder("http://localhost:8080")
				.endpoint("/mcp")
				.build();

		McpSyncClient client = McpClient.sync(transport)
				.clientInfo(new McpSchema.Implementation("demo-client", "1.0.0"))
				.build();

		// 显式握手：协商版本与能力，并接收服务端下发的 Mcp-Session-Id
		var init = client.initialize();
		System.out.println("协议版本: " + init.protocolVersion());

		var tools = client.listTools();
		System.out.println("可用工具: " + tools.tools());

		var result = client.callTool(
				new McpSchema.CallToolRequest("queryOrder", Map.of("orderId", "A1001")));
		System.out.println(result.content());

		client.closeGracefully();
	}
}
```

### Python 服务端（FastMCP）

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("legacy-order-server")

@mcp.tool()
def query_order(order_id: str) -> str:
    """查询订单状态"""
    return f"order {order_id} -> SHIPPED"

@mcp.resource("order://{order_id}")
def order_detail(order_id: str) -> str:
    """把订单详情暴露为资源"""
    return f'{{"orderId": "{order_id}", "status": "SHIPPED"}}'

@mcp.prompt()
def refund_prompt(order_id: str) -> str:
    """退款处理提示词模板"""
    return f"请检查订单 {order_id} 是否符合退款条件，并给出结论。"

if __name__ == "__main__":
    # 会话式 Streamable HTTP，默认端点为 /mcp
    mcp.run(transport="streamable-http")
```

### Python 客户端（ClientSession）

```python
import asyncio

from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

async def main() -> None:
    async with streamablehttp_client("http://localhost:8000/mcp") as (read, write, _):
        async with ClientSession(read, write) as session:
            init = await session.initialize()          # 显式握手
            print(init.protocolVersion)                # "2025-11-25"

            tools = await session.list_tools()
            print(tools)

            result = await session.call_tool("query_order", {"order_id": "A1001"})
            print(result.content)

asyncio.run(main())
```

### 服务端 → 客户端：Sampling 与 Elicitation

Legacy 允许服务端在处理某个请求的过程中，**反向发起**一条 JSON-RPC 请求（采样或征求用户输入），这条请求走同一条双向流（stdio 或 Streamable HTTP 的 SSE 流）。

服务端发起采样：

```json
{
  "jsonrpc": "2.0",
  "id": "req-42",
  "method": "sampling/createMessage",
  "params": {
    "messages": [
      { "role": "user", "content": { "type": "text", "text": "总结这条订单记录" } }
    ],
    "maxTokens": 256
  }
}
```

服务端发起 Elicitation：

```json
{
  "jsonrpc": "2.0",
  "id": "req-43",
  "method": "elicitation/create",
  "params": {
    "mode": "form",
    "message": "确认是否取消订单 A1001？",
    "requestedSchema": {
      "type": "object",
      "properties": { "approved": { "type": "boolean" } },
      "required": ["approved"]
    }
  }
}
```

Python 客户端注册 Elicitation 回调：

```python
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client
from mcp.shared.context import RequestContext
from mcp.types import ElicitRequestParams, ElicitResult

async def on_elicit(
    context: RequestContext, params: ElicitRequestParams
) -> ElicitResult:
    # 实际项目里在这里弹出 UI 让用户确认
    return ElicitResult(action="accept", content={"approved": True})

async def main() -> None:
    async with streamablehttp_client("http://localhost:8000/mcp") as (read, write, _):
        async with ClientSession(read, write, elicitation_callback=on_elicit) as session:
            await session.initialize()
            await session.call_tool("cancel_order", {"order_id": "A1001"})

asyncio.run(main())
```

> 说明：Tasks 在 2025-11-25 中仍是**实验性**能力，需在能力协商中显式声明 `tasks`，不属于稳定接口。

# Modern（2026-07-28）：协议结构、功能与代码案例

## 协议结构

### 无状态请求模型

Modern 移除了 `initialize` 握手与协议级会话。**每一个请求都必须自带协议版本、客户端身份与客户端能力**，服务端仅凭当前请求即可处理，不依赖任何先前请求或连接状态。

请求中的 `_meta` 保留字段：

| `_meta` 键 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `io.modelcontextprotocol/protocolVersion` | string | 是 | 当前请求使用的协议版本，如 `"2026-07-28"` |
| `io.modelcontextprotocol/clientInfo` | object | 建议 | 客户端名称与版本，用于展示、日志与排障 |
| `io.modelcontextprotocol/clientCapabilities` | object | 是 | 该请求相关的客户端能力（如 `elicitation`、`extensions`） |
| `io.modelcontextprotocol/logLevel` | string | 否 | 本请求的日志级别；未提供时服务端不得发送 `notifications/message` |
| `progressToken` | string/number | 否 | 为本请求开启进度通知 |
| `traceparent` / `tracestate` / `baggage` | string | 否 | OpenTelemetry 上下文传播 |

一个完整的 Streamable HTTP 请求：

```http
POST /mcp HTTP/1.1
Content-Type: application/json
Accept: application/json, text/event-stream
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: query_order

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "query_order",
    "arguments": { "orderId": "A1001" },
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientInfo": { "name": "demo-client", "version": "1.0.0" },
      "io.modelcontextprotocol/clientCapabilities": {}
    }
  }
}
```

### 结果信封：resultType

所有结果都必须携带 `resultType`：

- `"complete"`：请求正常完成，结果即最终内容。
- `"input_required"`：请求尚未完成，服务端还需要客户端补充输入（见 MRTR）。
- 客户端对缺少 `resultType` 的旧版服务端结果，**必须**按 `"complete"` 处理。

### 方法清单（2026-07-28）

| 类别 | 方法 | 说明 |
| --- | --- | --- |
| 发现 | `server/discover` | **服务端必须实现**；返回 `supportedVersions`、`capabilities`、`instructions` 与服务端身份 |
| 工具 | `tools/list`、`tools/call` | 列表结果需带缓存字段，工具顺序需确定 |
| 资源 | `resources/list`、`resources/templates/list`、`resources/read` | 不再有 `resources/subscribe` |
| 提示词 | `prompts/list`、`prompts/get` | |
| 补全 | `completion/complete` | 由 prompts/resources feature 各自处理 |
| 订阅 | `subscriptions/listen` | 取代旧 GET 长流与资源订阅，单一长生命周期请求 |
| 通知 | `notifications/subscriptions/acknowledged`、`notifications/tools/list_changed`、`notifications/prompts/list_changed`、`notifications/resources/list_changed`、`notifications/resources/updated`、`notifications/progress`、`notifications/message` | 请求级通知只走该请求的响应流 |
| 已移除 | `ping`、`logging/setLevel`、`resources/subscribe`、`resources/unsubscribe`、`notifications/roots/list_changed` | 新实现不应再使用 |
| MRTR 载体 | `elicitation/create`、`sampling/createMessage`、`roots/list` | 不再是独立请求，只能出现在 `InputRequiredResult` 的 `inputRequests` 中 |

版本不匹配时，服务端返回 `UnsupportedProtocolVersionError`（`-32022`），并在 `data.supported` 中列出支持的版本，客户端选择共同版本后重试。

### 传输与 Header

| 机制 | 说明 |
| --- | --- |
| 单端点 `POST` | 每个请求/通知都是独立的 HTTP POST；**移除 GET 端点**（GET 返回 `405`） |
| 响应形态 | 单条 `application/json`，或按需开启的 `text/event-stream`（SSE） |
| 路由 Header | 所有请求必须带 `Mcp-Method`；`tools/call`、`prompts/get`、`resources/read` 必须带 `Mcp-Name` |
| 版本 Header | 必须带 `MCP-Protocol-Version`，且与请求 `_meta` 中的版本一致 |
| 头部校验 | 缺失或不匹配返回 `-32020`（Header Mismatch）与 HTTP `400` |
| 自定义 Header | 工具参数可用 `x-mcp-header` 标注，客户端镜像为 `Mcp-Param-{Name}` |
| 通知回执 | 仅含通知的 POST 返回 `202 Accepted`，无响应体 |
| 状态码 | `400` 解析/参数/Header/能力/版本错误；`404` 未知方法（`-32601`）；`403` 非法 `Origin` |
| SSE | **不再支持** `Last-Event-ID` 续传；关闭 SSE 响应流即视为取消请求 |
| 订阅 | 变更通知通过 `subscriptions/listen` 的响应流推送，通知带 `io.modelcontextprotocol/subscriptionId` |
| 安全 | 必须校验 `Origin`；本地绑定 `127.0.0.1`；HTTP 场景遵循 OAuth 2.1 |

错误码分区：

| 错误码 | 名称 | 说明 |
| --- | --- | --- |
| `-32020` | `HeaderMismatch` | Header 与请求体不一致 |
| `-32021` | `MissingRequiredClientCapability` | 缺少请求所需客户端能力 |
| `-32022` | `UnsupportedProtocolVersion` | 不支持的协议版本 |
| `-32602` | `InvalidParams` | 资源不存在也改用此码（旧码 `-32002` 仅保留兼容接收） |

### 扩展机制

核心协议保持稳定，能力通过 `capabilities.extensions` 声明与协商：

| 扩展标识 | 能力 |
| --- | --- |
| `io.modelcontextprotocol/tasks` | 长任务：`tasks/get` 轮询、`tasks/update` 客户端补充输入、持久化任务句柄 |
| `io.modelcontextprotocol/ui` | MCP Apps：在对话内渲染图表、表单、视频等交互 UI |
| Skills over MCP | 结构化技能说明，供 Agent 工作流发现与消费 |

扩展是**可选**能力，双方都必须声明；一方不支持时应回退到核心行为或返回相应错误。

## 功能特性

| 能力 | 归属 | Modern 下的形态 |
| --- | --- | --- |
| Tools / Resources / Prompts | Server | 与 Legacy 语义一致，但列表/读取结果必须带 `ttlMs`、`cacheScope`，且顺序确定 |
| Elicitation | Client | 保留能力，但通过 **MRTR** 在 `tools/call`、`prompts/get`、`resources/read` 的响应中征求输入 |
| Sampling | Client | **已弃用**（12 个月弃用窗口），新实现不应接入；建议直接集成 LLM Provider API |
| Roots | Client | **已弃用**；建议通过工具参数、资源 URI 或服务端配置传递目录 |
| Logging | Server | **已弃用**；建议 stdio 写 stderr，或使用 OpenTelemetry |
| 变更通知 | Server | 统一为 `subscriptions/listen`，支持列表变更与指定资源更新 |
| 缓存 | Server | `ttlMs` 新鲜度提示 + `cacheScope`（`public`/`private`） |
| 可观测性 | Both | `_meta` 传播 W3C Trace Context（`traceparent` 等），无缝接入 OTel |
| 长任务 | Both | 由 Tasks 扩展承载，核心协议不再内置任务方法 |
| 授权 | Client/Server | 强化 OAuth：`iss`（RFC 9207）校验、凭证绑定签发方、Client ID Metadata Documents |

缓存结果示例：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "resultType": "complete",
    "tools": [
      { "name": "query_order", "description": "查询订单状态", "inputSchema": { "type": "object" } }
    ],
    "ttlMs": 60000,
    "cacheScope": "public"
  }
}
```

## 代码案例

以下示例基于 **MCP Java SDK 2.2.0** 的 `io.modelcontextprotocol.modern` 包与 **官方 Python SDK v2**。Java 服务端端点为 `http://localhost:8080/mcp`，Python 服务端端点为 `http://localhost:8000/mcp`。

### Java 服务端（MCP Java SDK 2.x modern）

```xml
<dependency>
	<groupId>io.modelcontextprotocol.sdk</groupId>
	<artifactId>mcp</artifactId>
	<version>2.2.0</version>
</dependency>
```

Modern Server 没有会话，也没有 `initialize`：`McpServer` 不可变且无状态，`build()` 时收集已注册 Feature 的能力，并直接对外提供 `server/discover`。

```java
import java.util.List;
import java.util.Map;

import io.modelcontextprotocol.modern.McpSchema.CallToolResult;
import io.modelcontextprotocol.modern.McpSchema.Implementation;
import io.modelcontextprotocol.modern.McpSchema.TextContent;
import io.modelcontextprotocol.modern.McpSchema.Tool;
import io.modelcontextprotocol.modern.server.McpRequestContext;
import io.modelcontextprotocol.modern.server.McpServer;
import io.modelcontextprotocol.modern.server.McpSyncResponse;
import io.modelcontextprotocol.modern.server.feature.McpSyncTool;
import io.modelcontextprotocol.modern.server.feature.McpSyncToolRepository;
import io.modelcontextprotocol.modern.server.feature.ToolsFeature;
import io.modelcontextprotocol.modern.server.feature.ToolsPage;
import io.modelcontextprotocol.modern.server.transport.HttpServletMcpTransport;

class OrderTools implements McpSyncToolRepository {

	private final McpSyncTool queryOrder = McpSyncTool.of(
			Tool.builder("query_order", Map.of(
							"type", "object",
							"properties", Map.of("orderId", Map.of("type", "string")),
							"required", List.of("orderId")))
					.description("查询订单状态")
					.build(),
			(ctx, request) -> McpSyncResponse.result(CallToolResult.builder()
					.addContent(TextContent.builder(
							"order " + request.arguments().get("orderId") + " -> SHIPPED").build())
					.build()));

	@Override
	public ToolsPage list(McpRequestContext ctx, String cursor) {
		// 顺序必须确定，不能随连接变化
		return ToolsPage.of(List.of(this.queryOrder.definition()));
	}

	@Override
	public McpSyncTool find(McpRequestContext ctx, String name) {
		return "query_order".equals(name) ? this.queryOrder : null;
	}
}

McpServer server = McpServer.builder()
		.serverInfo(Implementation.builder("order-server", "1.0.0").build())
		.feature(ToolsFeature.ofSync(new OrderTools()))
		.build();

HttpServletMcpTransport transport = HttpServletMcpTransport.builder(server)
		.endpoint("/mcp")
		.build();

// 将 transport 注册为 Servlet 并映射到 /mcp，同时开启 async 支持；
// subscriptions/listen 的响应流需要容器支持异步请求。
```

资源、提示词分别用 `ResourcesFeature`、`PromptsFeature` 注册；只注册需要的 Feature，`server/discover` 会自动只声明对应能力。`server.closeGracefully()` 用于在关闭时以 `complete` 结果结束所有打开的订阅流。

### Java 客户端（modern）

```java
import java.util.Map;

import io.modelcontextprotocol.modern.McpSchema.Implementation;
import io.modelcontextprotocol.modern.client.McpClient;
import io.modelcontextprotocol.modern.client.McpSyncClient;
import io.modelcontextprotocol.modern.client.transport.HttpClientMcpTransport;

public class ModernClientDemo {

	public static void main(String[] args) {
		HttpClientMcpTransport transport =
				HttpClientMcpTransport.builder("http://localhost:8080/mcp").build();

		McpSyncClient client = McpClient.sync(transport)
				.clientInfo(Implementation.builder("demo-client", "1.0.0").build())
				.build();

		// 无需 initialize：首个请求会自带 _meta 与 Mcp-* 头
		var discovered = client.discover();   // 等价于 server/discover
		System.out.println("支持版本: " + discovered.supportedVersions());

		for (var tool : client.listTools().list().tools()) {
			System.out.println("工具: " + tool.name());
		}

		var result = client.tool("query_order")
				.arguments(Map.of("orderId", "A1001"))
				.call();
		System.out.println(result.content());
	}
}
```

客户端是**不可变且线程安全**的，构建时不产生 I/O；一个客户端代表一个身份，多用户场景应每个用户一个客户端，多个客户端可共享同一 transport。版本协商、`_meta` 注入、`Mcp-Method`/`Mcp-Name` 头、输入请求的多轮重试都由 SDK 自动完成。

### Python 服务端（MCP SDK v2）

```python
from mcp.server import MCPServer

mcp = MCPServer("modern-order-server")

@mcp.tool()
def query_order(order_id: str) -> str:
    """查询订单状态"""
    return f"order {order_id} -> SHIPPED"

@mcp.resource("order://{order_id}")
def order_detail(order_id: str) -> str:
    """把订单详情暴露为资源"""
    return f'{{"orderId": "{order_id}", "status": "SHIPPED"}}'

if __name__ == "__main__":
    # 无状态 Streamable HTTP：每个 POST 独立，无会话
    mcp.run(transport="streamable-http", stateless_http=True, json_response=True)
```

Python SDK v2 的同一个应用可同时服务 Legacy 与 Modern 客户端：收到 `initialize` 走旧语义，收到带 `_meta` 的请求走无状态语义，无需额外开关。

### Python 客户端（v2 Client）

```python
import anyio

from mcp import Client

async def main() -> None:
    # 传入 URL 即使用 Streamable HTTP；传 MCPServer 对象则直接在内存中调用（适合测试）
    async with Client("http://localhost:8000/mcp") as client:
        print("协议版本:", client.protocol_version)      # "2026-07-28"
        print("服务端信息:", client.server_info)
        print("服务端能力:", client.server_capabilities)

        tools = await client.list_tools()
        print(tools)

        result = await client.call_tool("query_order", {"order_id": "A1001"})
        print(result.structured_content, result.content)   # v2 字段名为 snake_case

if __name__ == "__main__":
    anyio.run(main)
```

### MRTR：多轮往返请求

当服务端需要用户补充输入时，它不再反向调用客户端，而是把“待补充输入”放进结果里返回。客户端收集答案后，**用新的 JSON-RPC id 重试原请求**，并在 `params` 中带回 `inputResponses` 与原始 `requestState`。

第一轮响应：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "input_required",
    "inputRequests": {
      "confirm": {
        "method": "elicitation/create",
        "params": {
          "mode": "form",
          "message": "确认取消订单 A1001？",
          "requestedSchema": {
            "type": "object",
            "properties": { "approved": { "type": "boolean" } },
            "required": ["approved"]
          }
        }
      }
    },
    "requestState": "opaque-server-state"
  }
}
```

客户端重试：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "cancel_order",
    "arguments": { "orderId": "A1001" },
    "inputResponses": {
      "confirm": { "action": "accept", "content": { "approved": true } }
    },
    "requestState": "opaque-server-state",
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": { "elicitation": {} }
    }
  }
}
```

`requestState` 对客户端完全不透明，客户端必须原样回传；服务端必须把它当作**不可信输入**，做完整性保护（HMAC/AEAD）并校验主体、过期时间与原始请求，防止篡改与重放。

Python SDK v2 用 `Resolve` 把多轮往返封装成“依赖注入”，工具函数体只写一遍：

```python
from typing import Annotated

from pydantic import BaseModel

from mcp.server import MCPServer
from mcp.server.mcpserver import AcceptedElicitation, Elicit, ElicitationResult, Resolve

mcp = MCPServer("order-server")

class Confirm(BaseModel):
    approved: bool

async def ask_confirm() -> Elicit[Confirm]:
    """解析器：向用户提问"""
    return Elicit("确认取消该订单？", Confirm)

@mcp.tool()
async def cancel_order(
    order_id: str,
    confirm: Annotated[ElicitationResult[Confirm], Resolve(ask_confirm)],
) -> str:
    """取消订单，需要用户确认"""
    if isinstance(confirm, AcceptedElicitation) and confirm.data.approved:
        return f"已取消订单 {order_id}"
    return "用户未确认，订单保留"
```

`confirm` 参数不会出现在工具的 `inputSchema` 中，模型看不到也无法伪造；Legacy 连接下 SDK 发送实时 elicitation 请求，Modern 连接下自动改为 `input_required` 多轮往返。

### 订阅变更：subscriptions/listen

客户端通过一个长生命周期的 `subscriptions/listen` 请求订阅所需的变更类型，服务端先返回 `notifications/subscriptions/acknowledged`，之后所有通知都带 `io.modelcontextprotocol/subscriptionId`：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "subscriptions/listen",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientInfo": { "name": "demo-client", "version": "1.0.0" },
      "io.modelcontextprotocol/clientCapabilities": {}
    },
    "notifications": {
      "toolsListChanged": true,
      "resourceSubscriptions": ["order://A1001"]
    }
  }
}
```

`notifications` 是过滤器：`toolsListChanged`、`promptsListChanged`、`resourcesListChanged`、`resourceSubscriptions`（资源 URI 数组），未声明即不订阅。`notifications/progress` 与 `notifications/message` 等**请求级**通知不会走这条流，只走各自请求的响应流。

# 两个时代的互操作

官方给出的兼容矩阵：

| 客户端 | 服务端 | 结果 |
| --- | --- | --- |
| Modern | Modern | 正常；`server/discover` 可选，版本不匹配时按 `UnsupportedProtocolVersionError` 换版本重试 |
| Modern | Legacy | 失败；stdio 上建议先探测 `server/discover`，尽早给出可操作的错误 |
| Dual-era | Modern | 正常，保持 Modern |
| Dual-era | Legacy | 正常；stdio 探测失败后回退 `initialize`，HTTP 上 `4xx` 且无可识别 Modern 错误体时回退 |
| Legacy | Modern | 失败；Modern 服务端会拒绝缺少 `_meta` 与 Header 的请求 |
| Legacy | Dual-era | 正常；服务端按 `initialize` 走 Legacy 语义 |

落地到 SDK：

- **Java SDK 2.x**：Legacy 在 `io.modelcontextprotocol.server` / `io.modelcontextprotocol.client`，Modern 在 `io.modelcontextprotocol.modern.server` / `io.modelcontextprotocol.modern.client`，两套类型互不共享；同一个应用可各自接入。
- **Python SDK v2**：同一个 `MCPServer` 应用、同一个端点，同时服务两种时代，客户端 `Client` 进入时自动完成探测与协商。

# 迁移要点

1. **删除握手与会话**：不再调用 `initialize`，不再维护 `Mcp-Session-Id`；所有请求携带 `_meta`，服务端不得依赖连接状态。
2. **状态改为显式句柄**：跨调用状态由工具生成业务句柄（如 `jobId`、`orderId`）并由模型在后续调用中回传，而不是藏在传输会话里。
3. **重写反向调用**：`ctx.elicit()` / `sampling` / `roots` 在 Modern 连接下不再可用；改用 MRTR（`resultType: "input_required"` + `inputRequests` + `requestState`）。
4. **统一订阅**：用 `subscriptions/listen` 替代 GET 长流与 `resources/subscribe`，并处理 `notifications/subscriptions/acknowledged`。
5. **补齐缓存字段**：`tools/list`、`prompts/list`、`resources/list`、`resources/read`、`resources/templates/list` 返回 `ttlMs`、`cacheScope`，并保证确定性排序。
6. **适配 HTTP 治理**：客户端发送 `Mcp-Method`、`Mcp-Name`、`MCP-Protocol-Version`；网关直接基于 Header 路由、限流、鉴权。
7. **处理新错误码**：`-32020`/`-32021`/`-32022`；资源不存在改用 `-32602`，同时兼容接收旧服务端的 `-32002`。
8. **停止新增弃用能力**：Sampling、Roots、Logging 不再接入；日志写 stderr 或接 OpenTelemetry。
9. **Schema 与扩展**：默认 JSON Schema 2020-12；新能力优先通过扩展（Tasks、MCP Apps、Skills）实现，而不是改核心协议。

# 参考

- [MCP 规范 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28)
- [MCP 规范 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25)
- [2026-07-28 变更日志](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- [版本与兼容性](https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning)
- [Multi Round-Trip Requests](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr)
- [server/discover](https://modelcontextprotocol.io/specification/2026-07-28/server/discover)
- [subscriptions/listen](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/subscriptions)
- [Streamable HTTP（2026-07-28）](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)
- [MCP Java SDK（Modern Server）](https://java.sdk.modelcontextprotocol.io/2.2.0-SNAPSHOT/server-modern/)
- [MCP Java SDK（Modern Client）](https://java.sdk.modelcontextprotocol.io/2.2.0-SNAPSHOT/client-modern/)
- [Python SDK v2](https://py.sdk.modelcontextprotocol.io/v2/whats-new/)
- [Spring AI MCP Server Boot Starter](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-server-boot-starter-docs.html)

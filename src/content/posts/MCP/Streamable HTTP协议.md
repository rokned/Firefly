---
slug: Streamable-HTTP
title: Streamable HTTP
description: Streamable HTTP协议
published: 2026-11-26
updated: 2026-11-26
tags:
  - MCP
  - AI
  - Agent
category: MCP
draft: false
---

# 概述

Streamable HTTP 是 MCP 的远程传输方式，于 **2025-03-26** 版规范中引入，用来取代早期的 **HTTP+SSE** 传输（2024-11-05 版）。它把原先“两条连接、两个端点”的模型，收敛为**单一 MCP 端点**上的普通 HTTP 请求：客户端通过 `POST` 发送 JSON-RPC 消息，服务端根据请求性质返回**一次性 JSON**，或返回**一条按需开启的 SSE 流**。

一句话概括：**用最普通的 HTTP 语义，实现“能流式、可恢复、可无状态”的 MCP 通信。**

# 出现原因

## 旧传输模型：HTTP+SSE

2024-11-05 版规范定义的远程传输由两条独立连接组成：

| 连接 | 方法 | 用途 |
| --- | --- | --- |
| `GET /sse` | 长连接 SSE | 服务端 → 客户端的**唯一**下行通道 |
| `POST /messages` | 普通 POST | 客户端 → 服务端的上行请求 |

客户端先建立 SSE 长连接，服务端通过 `endpoint` 事件告诉客户端后续消息的投递地址；此后所有响应、通知、服务端发起的能力（如 Sampling）都只能从这条 SSE 流里回来。

## 旧模型的痛点

1. **双连接强耦合**：任意一条连接断开，会话即失效；客户端无法只保持其中一条。
2. **长连接对基础设施不友好**：企业代理、CDN、WAF、负载均衡普遍对空闲连接设置几十秒到几分钟的超时，SSE 流被切断后无法恢复；服务端还必须为每个客户端常驻一条连接，连接数与内存随客户端数量线性增长。
3. **无法无状态 / Serverless 部署**：长期存活的 GET 流要求进程常驻，与 Serverless、按请求计费、快速弹性伸缩的部署方式冲突；水平扩展还需要会话粘性（sticky session）。
4. **不支持断线续传**：规范没有事件 ID 与重放机制，网络抖动后只能整段重建会话，期间错过的通知直接丢失。
5. **鉴权能力受限**：浏览器原生 `EventSource` 不能携带自定义请求头，token 只能塞进 URL 查询参数，容易泄露到日志、代理缓存与浏览器历史中。
   ```js
   // EventSource 没有 headers 参数，Authorization 头加不进去
const es = new EventSource("https://mcp.example.com/sse?token=abc123");
   ```
6. **上行响应无法直接流式**：客户端请求只能拿到普通 JSON 响应，进度、日志、采样等一切流式内容都被迫挤回那条唯一的 SSE 通道，通道成为单点与瓶颈。

   `/messages` 并不是规范固定的路径，而是服务端通过 `endpoint` 事件下发的动态地址，承担**客户端 → 服务端**方向的全部消息，并不止一种功能：

   - JSON-RPC **请求**：`tools/call`、`tools/list`、`resources/read` 等；
   - JSON-RPC **通知**：`notifications/initialized`、取消通知等；
   - JSON-RPC **响应**：对服务端主动发起的请求（如 Sampling）的答复。

   但它作为 HTTP 接口只有一种响应语义：**“已受理”的回执**（实现上通常返回 `202 Accepted`，无响应体）。JSON-RPC 的业务结果不从这里返回。
   
   旧模型下一次工具调用的完整链路：
   ```mermaid
   sequenceDiagram
       participant C as 客户端
       participant S as 服务端
       C->>S: GET /sse（建立唯一下行长连接）
       S-->>C: event: endpoint（/messages?sessionId=...）
       C->>S: POST /messages（tools/call 请求）
       S-->>C: 202 Accepted（仅表示已受理）
       S-->>C: SSE: progress 30%（经 GET /sse）
       S-->>C: SSE: result（经 GET /sse）
   ```

   所谓“POST 响应本身没法流式返回”，意思是：这个 POST 在 t=0s 就以 `202` 结束了，耗时 30 秒的工具执行过程中产生的进度、日志和最终结果，只能绕回所有请求共用的 `GET /sse` 流；客户端还得自己按 JSON-RPC `id` 把消息匹配回对应请求。

| 时间    | 旧模型（HTTP+SSE）                         | Streamable HTTP                             |
| ----- | ------------------------------------- | ------------------------------------------- |
| t=0s  | POST `tools/call`，立即收到 `202 Accepted` | POST `tools/call`，响应头返回 `text/event-stream` |
| t=1s  | 进度 `30%` 只能从 `GET /sse` 流推回           | 进度 `30%` 直接写在此次 POST 的响应流里                  |
| t=10s | 进度 `60%`，依旧走公共 SSE 流                  | 进度 `60%`，同一条响应流                             |
| t=30s | 最终 `result` 仍从公共 SSE 流返回              | 最终 `result` 在同一响应流返回，随后关闭                   |

   Streamable HTTP 把流式能力还给了每个请求自己的响应：
   ```mermaid
   sequenceDiagram
       participant C as 客户端
       participant S as 服务端
       C->>S: POST /mcp（tools/call 请求）
       S-->>C: 200 OK Content-Type: text/event-stream
       S-->>C: SSE: progress 30%（属于本次请求）
       S-->>C: SSE: progress 60%
       S-->>C: SSE: result
       S-->>C: 流关闭
   ```

7. **部署与排障复杂**：两个端点、一套额外事件协议，网关需要分别放行，排查问题时要在两条连接之间来回对照。

## 设计目标

Streamable HTTP 用一次传输层重写解决了上述问题，核心目标：

- **单端点**：所有 JSON-RPC 消息都发往同一个 URL（如 `/mcp`），网关只需配置一条路由。
- **按需流式**：普通请求返回 JSON，需要流式时在同一响应内切换为 SSE，不必维护常驻流。
- **可选会话**：会话可开可关，支持无状态部署。
- **可恢复**：通过事件 ID 与 `Last-Event-ID` 重放断线期间的消息。
- **兼容标准 HTTP 基础设施**：普通请求/响应语义，天然适配代理、负载均衡、鉴权与可观测体系。

# 核心机制

| 机制    | 说明                                                                                                  |
| ----- | --------------------------------------------------------------------------------------------------- |
| 单端点   | 客户端 `POST` 到 MCP 端点；请求头必须同时接受 `application/json` 与 `text/event-stream`。                             |
| 响应形态  | 服务端可直接返回 `application/json` 的单条 JSON-RPC 响应，或返回 `text/event-stream` 的 SSE 流，流中每条事件承载一条 JSON-RPC 消息。 |
| 通知与回执 | 只包含通知/响应（不含请求）的 POST，服务端返回 `202 Accepted`，无响应体。                                                     |
| GET 流 | 客户端可选地通过 `GET` 建立服务端主动推送通道；服务端不支持时返回 `405 Method Not Allowed`，不影响主流程。                               |
| 会话    | 服务端在 `initialize` 响应中下发 `Mcp-Session-Id`，客户端后续请求必须携带；返回 `404` 表示会话过期，需重新初始化；`DELETE` 用于显式终止会话。      |
| 断点续传  | SSE 事件带 `id`，客户端重连时携带 `Last-Event-ID`，服务端重放该 ID 之后的消息（依赖服务端实现 EventStore）。                          |
| 协议版本  | 初始化后的请求需携带 `MCP-Protocol-Version` 头（2025-06-18 版起），便于服务端识别与降级。                                      |
| 安全    | 服务端应校验 `Origin` 防 DNS 重绑定攻击；本地部署绑定 `127.0.0.1`；远程部署使用 OAuth 2.1 等标准鉴权。                              |

> 说明：上表描述的是 **Legacy 时代**（2025-11-25 及更早版本）的 Streamable HTTP 机制。**2026-07-28（Modern）** 已移除协议级会话与 `Mcp-Session-Id`、SSE 断点续传及长期 GET 推送：默认采用无状态 Streamable HTTP，跨调用状态通过显式业务句柄（如 `jobId`）传递，服务端推送改由 `subscriptions/listen` 承担。下文示例基于仍广泛部署的 Legacy 实现（SDK 会自动处理会话头）。

# 与 SSE、WebSocket 对比

> 说明：此处的“SSE”指旧版 **HTTP+SSE 传输方案**，而不是 SSE 技术本身。Streamable HTTP 内部依然用 SSE 格式承载流式响应，被淘汰的是“双连接 + 常驻 SSE”的用法。

## 三方对比

| 维度      | HTTP+SSE（旧传输）         | Streamable HTTP          | WebSocket                   |
| ------- | --------------------- | ------------------------ | --------------------------- |
| 连接数量    | 2 条（SSE GET + POST）   | 单端点，请求独立，流按需开启           | 1 条长连接                      |
| 通信方向    | SSE 单向下行 + POST 单向上行  | 请求-响应为主，响应内可按需流式         | 全双工双向                       |
| 协议层级    | 普通 HTTP               | 普通 HTTP                  | HTTP `Upgrade` 后的独立协议       |
| 消息格式    | JSON-RPC over SSE     | JSON-RPC over JSON / SSE | 任意，MCP 需自行约定                |
| 断线续传    | 不支持                   | `Last-Event-ID` 重放       | 无标准机制，需自行实现                 |
| 无状态部署   | 不支持                   | 支持（会话可选）                 | 不支持                         |
| 代理 / 网关 | 长连接易被空闲超时切断           | 普通 HTTP 请求，天然兼容          | 需支持 `Upgrade`，部分 WAF/代理直接阻断 |
| 鉴权      | `EventSource` 无法带自定义头 | 标准 Header / OAuth 2.1    | 握手时传 Header，浏览器端受限          |
| 服务端主动推送 | 必须常驻长连接               | 可选 GET 流                 | 天然支持                        |
| 典型定位    | 已被规范淘汰                | 远程 MCP 默认传输              | 实时协同、游戏、IM 等通用场景            |

## 为什么不用 WebSocket

- **语义不匹配**：MCP 是请求-响应模型，绝大多数调用是“一问一答”，不需要全双工。WebSocket 只提供裸消息通道，请求 ID 关联、超时、错误处理都要自行封装，等于在协议之上再发明一套 RPC。
- **基础设施成本高**：WebSocket 依赖 `Upgrade` 握手与长连接，负载均衡需要会话粘性，Serverless 与边缘计算基本无法承载；企业内网的代理、WAF 也常直接拦截。
- **工程能力缺失**：没有标准化的鉴权、续传、事件重放、状态码语义，断线恢复与可观测性都靠自定义。
- **Streamable HTTP 已覆盖需求**：流式通知、日志、进度完全可以在一次 POST 的响应里用 SSE 表达，用不到全双工。

## Streamable HTTP 与 SSE 的关系

**不是取代 SSE 技术，而是取代 SSE 传输方案。** SSE 被“降级”为一种可选的响应表现形式：需要流式时，服务端在 POST 响应中返回 `text/event-stream` 并持续推送事件；不需要时就是一条普通 JSON 响应。这样既保留了 SSE 的低门槛（浏览器 `fetch` 即可消费），又摆脱了它作为独立长连接通道带来的部署负担。

# 代码使用案例

以下示例基于 **Spring AI / MCP Java SDK** 与官方 **Python SDK**；Java 服务端端点为 `http://localhost:8080/mcp`，Python 服务端端点为 `http://localhost:8000/mcp`。

## 1. Java 服务端（Spring Boot + Spring AI）

引入 Streamable HTTP 的 MCP Server starter（以 Spring AI 1.0.x 为例，版本以实际最新为准）：

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

在 `application.yml` 中指定 Streamable HTTP 协议：

```yaml
spring:
  ai:
    mcp:
      server:
        name: demo-server
        version: 1.0.0
        protocol: STREAMABLE   # STDIO / SSE / STREAMABLE / STATELESS
```

暴露工具（普通的 Spring Bean，方法即 MCP Tool）：

```java
@Component
public class CalculatorTools {

	@Tool(description = "计算两个数字之和")
	public int add(int a, int b) {
		return a + b;
	}
}
```

注册工具回调：

```java
@SpringBootApplication
public class McpServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(McpServerApplication.class, args);
	}

	@Bean
	ToolCallbackProvider mcpTools(CalculatorTools calculatorTools) {
		return MethodToolCallbackProvider.builder()
				.toolObjects(calculatorTools)
				.build();
	}
}
```

启动后 MCP 端点为 `POST /mcp`；若把 `protocol` 改为 `STATELESS`，则得到 Serverless 友好的无状态模式（不生成 `Mcp-Session-Id`）。

## 2. Java 客户端（MCP Java SDK）

```java
import io.modelcontextprotocol.client.McpClient;
import io.modelcontextprotocol.client.transport.HttpClientStreamableHttpTransport;
import io.modelcontextprotocol.spec.McpSchema;

import java.util.Map;

public class McpClientDemo {

	public static void main(String[] args) {
		// 单一端点，默认路径即为 /mcp
		var transport = HttpClientStreamableHttpTransport.builder("http://localhost:8080")
				.endpoint("/mcp")
				.build();

		var client = McpClient.sync(transport)
				.clientInfo(new McpSchema.Implementation("demo-client", "1.0.0"))
				.build();

		// initialize 握手、Mcp-Session-Id 维护、SSE 响应解析均由 SDK 自动完成
		client.initialize();

		var tools = client.listTools();
		System.out.println("可用工具: " + tools.tools());

		var result = client.callTool(
				new McpSchema.CallToolRequest("add", Map.of("a", 1, "b", 2)));
		System.out.println(result.content());

		client.closeGracefully();
	}
}
```

鉴权仍按标准 HTTP 方式处理：在 transport 构造时自定义请求头（如 `Authorization: Bearer <token>`）即可。

## 3. Python 服务端（FastMCP）

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo-server")

@mcp.tool()
def add(a: int, b: int) -> int:
    """计算两个整数之和"""
    return a + b

if __name__ == "__main__":
    # Streamable HTTP 传输，默认端点为 /mcp
    mcp.run(transport="streamable-http")
```

## 4. Python 客户端

```python
import asyncio
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

async def main() -> None:
    async with streamablehttp_client("http://localhost:8000/mcp") as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = await session.list_tools()
            print(tools)

asyncio.run(main())
```

# 版本演进

| 规范版本               | 与 Streamable HTTP 相关的变化                                                                        |
| ------------------ | ---------------------------------------------------------------------------------------------- |
| 2024-11-05         | 远程传输为 HTTP+SSE（GET 长连 + POST 双端点）                                                              |
| 2025-03-26         | 引入 Streamable HTTP，取代 HTTP+SSE，成为推荐的远程传输                                                       |
| 2025-06-18         | 移除 JSON-RPC 批量请求；要求后续请求携带 `MCP-Protocol-Version` 头；强化 `Origin` 校验                              |
| 2025-11-25         | Legacy 时代最后一个版本，Streamable HTTP 仍是远程传输主体，保留会话与 SSE 续传                                          |
| 2026-07-28（Modern） | 协议级会话与 `Mcp-Session-Id` 移除，SSE 续传移除，默认无状态 Streamable HTTP，以 `subscriptions/listen` 取代长期 GET 推送 |

# 注意事项

1. **会话过期处理**：客户端收到 `404` 时不能重试原请求，必须重新 `initialize` 并换用新的 `Mcp-Session-Id`。
2. **不要依赖 GET 长连接**：GET 流是可选能力，服务端返回 `405` 属正常行为，客户端应能降级为主链路工作。
3. **Origin 校验必做**：远程 MCP 服务应校验 `Origin` 白名单，防止 DNS 重绑定攻击；本地服务绑定 `127.0.0.1`。
4. **鉴权放 Header**：不要在 URL 查询参数中传递 token；远程部署遵循 OAuth 2.1 授权流程。
5. **按需选择模式**：有状态模式适合需要采样、通知重放的长会话；无状态模式适合 Serverless、边缘部署与只读工具查询。
6. **续传依赖 EventStore**：`Last-Event-ID` 重放需要服务端保存事件历史，未实现时断线即丢消息。

# 参考

- [MCP 传输规范（2025-06-18）](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports)
- [MCP 传输规范（2025-03-26）](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports)
- [MCP Java SDK](https://github.com/modelcontextprotocol/java-sdk)
- [Spring AI MCP Server Boot Starter](https://docs.spring.io/spring-ai/reference/api/mcp/mcp-server-boot-starter-docs.html)
- [Python SDK](https://github.com/modelcontextprotocol/python-sdk)

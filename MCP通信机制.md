# **MCP通信机制**

## **1. MCP 是什么**

**MCP（Model Context Protocol）是一种应用层的标准化交互协议，用于规范 AI 应用与外部工具、资源等能力之间如何交互。**

核心价值：过去不同 AI 应用对接外部工具往往需要分别适配；MCP 提供统一标准，只要双方实现 MCP，就可以按照统一方式进行能力发现和调用。

典型结构：

```text
AI Application / Agent
        │
    MCP Client
        │
     MCP 协议
        │
    MCP Server
        │
  外部工具 / 数据
```

需要区分：

- **MCP**：规定“交互什么、怎么交互”。
- **stdio / HTTP**：负责 MCP 消息“怎么传过去”。

因此 MCP 不是 TCP、HTTP 这种底层网络协议，而是建立在传输机制之上的应用层协议。

------

## **2. MCP Transport**

MCP Client 与 MCP Server 需要具体的传输方式来交换 MCP 消息，常见/历史上主要有：

1. stdio
2. HTTP + SSE（旧远程传输方案）
3. Streamable HTTP（新的远程传输方案）

### **stdio**

stdio = **Standard Input / Output（标准输入 / 标准输出）**。

它不是网络协议，而是利用操作系统进程的 stdin / stdout 进行通信。

例如 Java：

```java
System.in;   // stdin
System.out;  // stdout
```

MCP 中通常是两个不同进程：

```text
MCP Client
    │
    │ stdin / stdout
    ↓
MCP Server 子进程
```

适合 **Client 和 MCP Server 都运行在同一台机器** 的场景，例如 Desktop Agent、IDE 中调用本地 Git、文件系统等 MCP Server。

优点是简单、不需要启动 HTTP Server、不需要监听端口。

stdio 通常用于本机不同进程之间的 MCP 通信，不能直接用于跨机器网络通信。

远程 MCP Server 通常使用 HTTP 类 Transport。

------

# **3. SSE**

## **3.1 为什么需要 SSE**

普通 HTTP 最常见的是 Request → Response：

```text
Client ── Request ──→ Server
Client ←─ Response ── Server
                    ↓
                  完成
```

如果 Server 只需要返回一个完整结果，这已经足够。

但有些交互需要 Server **持续发送多条消息**：

```text
Client ── Request ──→ Server

Client ←── 消息1 ──── Server
Client ←── 消息2 ──── Server
Client ←── 消息3 ──── Server
                    ...
```

SSE（Server-Sent Events）就是 HTTP 上的一种服务器事件流机制。

------

## **3.2 SSE 的原理**

普通 HTTP Response：

```http
Content-Type: application/json

{"result":"success"}
```

完整 Body 返回后，本次 Response 结束。

SSE：

```http
Content-Type: text/event-stream

data: {"progress":10}

data: {"progress":50}

data: {"progress":100}
```

区别在于：

**SSE 的 HTTP Response 不立即结束，Server 可以持续向这个 Response 写入事件。**

因此：

```text
普通 HTTP：
Request → 完整 Response → 结束

SSE：
Request → Response 开始 → event1 → event2 → event3 → ... → 最终结束
```

SSE 本身主要解决：

**Server → Client 的持续推送。**

因此旧 MCP 的 HTTP+SSE 可以粗略理解为：

```text
Client ── HTTP POST ──→ Server
Client ←─── SSE ─────── Server
```

------

# **4. HTTP Keep-Alive 与 SSE 长连接**

两者都经常被称为“长连接”，但不是一回事。

### **HTTP Keep-Alive**

```text
TCP连接建立

Request1 → Response1  （本次请求完成）
Request2 → Response2  （本次请求完成）
Request3 → Response3  （本次请求完成）

TCP连接继续保留
```

目的：

**复用底层连接，减少重复建立 TCP/TLS 连接的成本。**

### **SSE**

```text
Request
   ↓
Response 开始
   ↓
event1
   ↓
event2
   ↓
event3
   ↓
...
```

**一个 HTTP Request / Response 本身长期没有结束。**

所以一句话区分：

**HTTP Keep-Alive：请求已经结束，但底层连接先别断，下次继续用。**

> 

**SSE：这一次请求本身就还没有结束，Server 还在持续写 Response。**

------

# **5. 旧 HTTP + SSE 的问题**

SSE 本身并不是“性能差”，问题主要在于旧 MCP **强依赖长期 SSE 通道**，会增加远程服务的架构复杂度。

主要包括：

- Client → Server 和 Server → Client 通信方式分离，模型较复杂；
- 需要管理长期 SSE Connection；
- 需要考虑断线、重连、超时；
- 对负载均衡、故障切换、无状态服务和横向扩容不够友好。

例如大量 Client 长期连接：

```text
Client1 ───── SSE ───── Server
Client2 ───── SSE ───── Server
Client3 ───── SSE ───── Server
...
```

这也是后来 Streamable HTTP 要重点改进的问题。

------

# **6. Streamable HTTP**

Streamable HTTP 的核心不是“抛弃 SSE”，而是：

**统一 HTTP 通信模型，普通请求使用普通 HTTP Response，需要 Streaming 时再使用 SSE。**

Client 通过统一 Endpoint，例如：

```text
POST /mcp
```

发送 MCP 消息。

Server 根据实际情况决定响应方式：

```text
                 POST /mcp
                     │
                 MCP Server
                  /      \
                 /        \
          普通 Response    SSE Stream
              ↓               ↓
      application/json   text/event-stream
              ↓               ↓
          一次返回完成      持续返回消息
```

因此，一个简单的 `tools/list` 请求可以直接：

```text
POST /mcp
    ↓
tools/list
    ↓
JSON Response
    ↓
结束
```

如果某次交互需要持续返回消息，则可以使用 SSE Stream。

必要时还可以通过 GET 建立 SSE 流，用于 Server → Client 的消息推送。

------

## **7. Streamable HTTP 相比旧 HTTP+SSE 的核心优势**

### **① 通信模型更统一**

统一 HTTP Endpoint，不再强制维护旧模式下固定的 SSE 通道。

### **② SSE 按需使用**

普通请求：

```text
POST → JSON → 结束
```

需要 Streaming：

```text
POST → SSE → event1 → event2 → ... → 结束
```

**不是所有 MCP 请求都必须建立 SSE。**

### **③ 更容易支持无状态服务和横向扩容**

无状态情况下：

```text
             Load Balancer
            /      |      \
           ↓       ↓       ↓
       Server A Server B Server C
```

不同请求可以由不同 Server 处理，更适合负载均衡和集群扩容。

Streamable HTTP 的重要价值不是简单的“比 SSE 快”，而是降低对长期 SSE 会话的强依赖，使远程 MCP 服务更容易扩展和部署。

------

# **8. Content-Type：application/json 与 text/event-stream**

HTTP 本身并不要求 Body 必须是 JSON。

`Content-Type` 描述的是：

**当前 HTTP Body 使用什么数据格式。**

普通 JSON Response：

```http
Content-Type: application/json

{"result":"success"}
```

适合一次返回完整结果。

SSE：

```http
Content-Type: text/event-stream

data: {"progress":10}

data: {"progress":50}
```

适合持续返回多个事件。

因此没有必要所有请求一开始都使用 `text/event-stream`。

如果一次就能返回：

```text
tools/list → 一个完整工具列表
```

使用普通 JSON 更简单。

如果需要：

```text
消息1
消息2
消息3
...
```

才使用 event stream。

另外需要区分三个 Header 概念：

- **请求 Content-Type**：我发送给你的 Body 是什么格式。
- **Accept**：我希望/能够接受什么格式的 Response。
- **响应 Content-Type**：Server 最终返回的 Body 是什么格式。

Streamable HTTP 中，请求本身可以携带 MCP/JSON-RPC 消息，而 Server 根据交互需要返回普通 JSON 或 `text/event-stream`。

------

# **9. 一个容易产生的误区**

错误理解：

```text
获取工具信息 → 普通 HTTP

调用工具 → SSE / Streamable HTTP
```

实际上 Transport 服务于整个 **MCP Client ↔ MCP Server** 通信，而不仅仅是 Tool Call。

例如：

```text
initialize
tools/list
tools/call
resources/...
prompts/...
notifications
```

这些都属于 MCP 消息。

区别只在于：

**某一次 MCP 交互是否需要流式传输。**

例如 `tools/list` 通常一次返回即可，而某些长时间交互可能需要流式消息。

**10. 最终关系图**

```text
                    MCP
                     │
              应用层交互协议
                     │
              MCP Transport
                     │
        ┌────────────┴────────────┐
        │                         │
      stdio                Streamable HTTP
        │                         │
    本机进程通信             远程网络通信
                                  │
                         ┌────────┴────────┐
                         │                 │
                   普通 HTTP Response    SSE Stream
                         │                 │
                    一次返回完整结果     持续返回事件
```

核心记忆：

**MCP 规定“怎么交互”，Transport 解决“消息怎么传”。stdio 适合本机进程通信；远程通信基于 HTTP。SSE 本质是一个长期不结束、Server 持续写入事件的 HTTP Response；Streamable HTTP 并没有淘汰 SSE，而是把普通 HTTP Response 与 SSE Streaming 统一起来——能一次返回就一次返回，需要流式时才使用 SSE，从而简化通信模型并更好地支持无状态服务和横向扩容。**



# **10.SessionId 与有状态 / 无状态服务**

### **1. 什么是有状态与无状态**

这里的“状态”主要指 **MCP Server 是否需要保存跨多次请求的会话状态**。

**无状态（Stateless）**：Server 处理当前请求时，不依赖某一台 Server 本地保存的前序请求状态。

```text
请求1 → Server A → 处理 → 返回
请求2 → Server B → 处理 → 返回
请求3 → Server C → 处理 → 返回
```

每次请求都能够独立处理，因此非常适合：

- 负载均衡
- 水平扩容
- Server 故障切换

例如：

```text
             Load Balancer
            /      |      \
           ↓       ↓       ↓
       Server A Server B Server C
```

请求不需要固定发送到某一台 Server。

------

**有状态（Stateful）**：Server 需要保存 Client 跨请求的会话信息，后续请求依赖之前保存的状态。

例如：

```text
请求1 → Server A

Server A 保存：
Session abc123
    ├── Client信息
    ├── 会话上下文
    └── 其他状态
```

后续请求：

```text
Client
  │
  │ SessionId: abc123
  ↓
Server A
  │
  ↓
找到 abc123 对应的状态
  ↓
继续处理
```

如果状态只保存在 Server A 内存，那么请求被负载均衡到 Server B：

```text
Client → LB → Server B

SessionId: abc123
        ↓
Server B 找不到 Server A 内存中的状态
```

因此有状态服务做集群时通常还需要考虑 Sticky Session、共享 Session Store 等方案。

------

### **2. SessionId 是什么**

`Mcp-Session-Id` 可以理解为 **MCP 会话的唯一标识符**。

例如初始化后 Server 创建：

```text
MCP Session
     ↓
abc123
```

后续 Client：

```http
Mcp-Session-Id: abc123
```

Server 就知道：

当前请求属于 `abc123` 这个 MCP Session。

概念上类似 Java Web 中的 `JSESSIONID`。

需要注意：

**SessionId 只是“会话编号”，本身并不保存会话状态。**

真正的状态可能存在 Server 内存、Redis、数据库或其他共享系统中。

------

### **3. SessionId 不等于有状态**

不能简单理解为：

```text
有 SessionId = 一定是有状态服务
```

更准确地说：

- **SessionId**：解决“这个请求属于哪个 Session”。
- **Stateful / Stateless**：描述 Server 是否依赖跨请求保存的会话状态。

Streamable HTTP 允许 MCP Server 根据需要选择是否维护 Session。

```text
             Streamable HTTP
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
      Stateless            Stateful
          │                   │
   请求独立处理         维护跨请求状态
                              │
                      可使用 Mcp-Session-Id
```

因此 **Streamable HTTP 并不是必须使用 SessionId**。如果某个 MCP Server 报 `Missing session ID`，说明该 Server 的实现选择了需要 Session 的工作方式。

------

### **4. 为什么 Streamable HTTP 强调无状态**

无状态和 SessionId 都不是 Streamable HTTP 发明的，它们本身是非常常见的 Web 服务设计。

旧 MCP HTTP+SSE 的问题在于通信模型围绕长期 SSE 通道：

```text
Client ── POST ─────→ Server
Client ←══ SSE ══════ Server
```

Server 需要维护 Client、POST 请求与 SSE Connection 之间的关联，因此不利于实现标准的无状态 HTTP 服务。

Streamable HTTP 将模型简化为：

```text
Client ── POST /mcp ──→ Server
Client ←── Response ─── Server
```

Response 根据需要可以是：

```text
普通 JSON Response
```

或者：

```text
SSE Stream
```

因此不需要 Session 的 MCP Server 可以像普通 Web 服务一样：

```text
请求 → 处理 → 返回 → 结束
```

不同请求可以由不同 Server 处理，更容易进行负载均衡和水平扩容。

**Streamable HTTP 并没有发明“无状态”，而是重新设计 MCP 的 HTTP Transport，使 MCP Server 不再强依赖长期 SSE 会话，从而可以自然地采用无状态架构。**

------

### **5. SessionId、连接、Tool Task 不要混淆**

这几个概念属于不同层面：

```text
Mcp-Session-Id
    ↓
当前请求属于哪个 MCP 会话


HTTP / SSE Connection
    ↓
当前网络通信连接


Tool Task
    ↓
真正执行工具工作的任务


Stateful / Stateless
    ↓
Server 是否依赖跨请求保存的状态
```

例如一个 Tool 需要执行 20 秒，第 10 秒 SSE Connection 意外断开：

**只能确定当前 HTTP/SSE Connection 已经中断。**

不能仅根据 SessionId 推断：

- Tool 一定停止；
- Tool 一定继续；
- 换一台 Server 就能从第 10 秒继续；
- Server 一定能够恢复之前没有收到的数据。

这些属于 Tool 生命周期管理和断线恢复（Resumability）机制，不是 SessionId 或 Stateless 本身解决的问题。

------

### **核心记忆**

**SessionId = “这是哪个 MCP 会话”。**

> 

**Stateful = Server 处理后续请求需要依赖之前保存的会话状态。**

> 

**Stateless = 每次请求可以独立处理，不依赖某台 Server 私有保存的前序会话状态，因此更容易负载均衡和水平扩容。**

> 

**Streamable HTTP 并没有发明 SessionId 或无状态，而是通过重新设计 MCP 的 HTTP 通信模型，使 Server 可以选择简单的无状态模式，也可以在确有需要时维护 Session。**
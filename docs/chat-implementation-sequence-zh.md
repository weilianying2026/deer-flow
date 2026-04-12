# DeerFlow Chat 实现说明与时序图

## 目标

本文整理 DeerFlow Chat 在前后端的关键实现位置，以及从用户发送消息到流式返回的完整时序。

适用场景：

- 快速理解聊天功能主链路
- 排查流式返回、停止按钮、新会话建链问题
- 新同学入门代码导航

## 关键代码位置

### 前端

- 聊天页面入口：frontend/src/app/workspace/chats/[thread_id]/page.tsx
- 流式会话与发送逻辑：frontend/src/core/threads/hooks.ts
- API Client 封装：frontend/src/core/api/index.ts
- API Client 具体实现：frontend/src/core/api/api-client.ts

### 后端 Gateway

- 路由挂载入口：backend/app/gateway/app.py
- 线程运行路由：backend/app/gateway/routers/thread_runs.py
- 无预建线程运行路由：backend/app/gateway/routers/runs.py
- 运行服务层：backend/app/gateway/services.py

### 后端 Runtime 与 Agent

- Run Worker：backend/packages/harness/deerflow/runtime/runs/worker.py
- Lead Agent 构建：backend/packages/harness/deerflow/agents/lead_agent/agent.py

## 总览：聊天主链路

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户
    participant P as Chat 页面
    participant H as useThreadStream
    participant C as LangGraph SDK Client
    participant G as Gateway thread_runs
    participant S as Gateway services.start_run
    participant RM as RunManager
    participant W as runtime/runs/worker
    participant A as make_lead_agent
    participant B as StreamBridge/SSE

    U->>P: 在输入框提交消息
    P->>H: sendMessage(threadId, message)
    H->>H: 处理 optimistic UI 与附件上传
    H->>C: thread.submit(input, context, stream options)

    alt 已有 thread
        C->>G: POST /api/threads/{thread_id}/runs/stream
    else 新 thread
        C->>G: POST /api/runs/stream 或创建后进入 thread_runs
    end

    G->>S: start_run(body, thread_id)
    S->>RM: 创建 RunRecord 并启动任务
    RM->>W: 异步执行 run worker
    W->>A: make_lead_agent(config)
    A-->>W: 返回已装配模型/工具/中间件的 agent

    loop 执行期间
        W->>W: agent.astream(...)
        W->>B: 发布运行事件与状态增量
        B-->>G: sse_consumer 读取事件
        G-->>C: SSE 帧持续返回
        C-->>H: 更新 messages / values / tool 事件
        H-->>P: 渲染消息列表、标题、任务状态
    end

    W->>B: 发布完成事件
    B-->>G: run 完成
    G-->>C: SSE 结束
    C-->>H: onFinish(state)
    H-->>P: 刷新 threads 列表与最终 UI
```

## 分支一：停止按钮 Stop 流程

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户
    participant P as Chat 页面
    participant H as useThreadStream
    participant C as LangGraph SDK Client
    participant G as Gateway thread_runs
    participant RM as RunManager
    participant W as runtime/runs/worker
    participant B as StreamBridge/SSE

    U->>P: 点击 Stop
    P->>H: thread.stop()
    H->>C: 发送停止请求(joinStream stop flow)
    C->>G: POST /api/threads/{thread_id}/runs/{run_id}/stream?action=interrupt

    G->>RM: cancel(run_id, action=interrupt)
    RM-->>W: 标记取消(中断)

    loop worker 检查取消点
        W->>W: 在 astream 迭代边界检测取消
    end

    alt wait=1
        G->>RM: 等待任务结束
        RM-->>G: 任务结束
        G-->>C: 204 No Content
    else wait=0
        G-->>C: 继续返回剩余 SSE 缓冲事件
    end

    W->>B: 发布 cancelled/finished 事件
    B-->>G: sse_consumer 消费尾部事件
    G-->>C: SSE 结束
    C-->>H: 停止流并落盘状态
    H-->>P: UI 解除 loading，保留已产出内容
```

## 分支二：新会话创建流程

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户
    participant P as Chat 页面(thread_id=new)
    participant H as useThreadStream
    participant C as LangGraph SDK Client
    participant G1 as Gateway runs(无预建线程)
    participant S as services.start_run
    participant T as Thread Store/Checkpointer
    participant G2 as Gateway thread_runs

    U->>P: 首次发送消息
    P->>H: sendMessage(tempThreadId, message)
    H->>C: thread.submit(...)

    alt SDK 走无预建线程流
        C->>G1: POST /api/runs/stream
        G1->>S: start_run(body, generated_thread_id)
    else SDK 先建线程后流式运行
        C->>G2: POST /api/threads/{thread_id}/runs/stream
        G2->>S: start_run(body, thread_id)
    end

    S->>T: upsert 线程元数据(确保可在 /threads/search 看到)
    S-->>C: 通过 SSE Header 返回 Content-Location
    C-->>H: onCreated(meta.thread_id)
    H->>P: onStart(createdThreadId)
    P->>P: history.replaceState(/workspace/chats/{createdThreadId})
    P-->>U: URL 切换到真实 thread_id，消息继续流式显示
```

## 前端到后端的参数要点

- 前端 sendMessage 会把模式信息写入 context：thinking_enabled、is_plan_mode、subagent_enabled、reasoning_effort。
- thread.submit 使用 streamSubgraphs 与 streamResumable，支持更完整的流式事件和断线恢复。
- 后端通过 Content-Location 返回 run 资源路径，供 SDK 识别与续流。

## 排查建议

- 页面无流式更新：先看 thread_runs 路由是否收到请求，再看 services 的 SSE consumer。
- Stop 不生效：检查 /runs/{run_id}/stream?action=interrupt 是否命中，以及 RunManager.cancel 返回状态。
- 新会话 URL 不切换：检查 onCreated 回调是否触发与 history.replaceState 是否执行。

## 一句话总结

DeerFlow Chat 是一个基于 LangGraph SDK + Gateway SSE 的流式对话链路：前端 useThreadStream 发起 run，后端 Run Worker 驱动 agent.astream，事件经 StreamBridge 回传，最终在前端增量渲染为聊天界面。
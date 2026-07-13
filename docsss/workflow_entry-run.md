入口已经清楚了。现在继续追 AppGenerateService.generate 里怎么根据 ADVANCED_CHAT 切到工作流执行器，以及流式事件怎么
  回前端。

• Explored
  └ Read app_generate_service.py, app_generator.py, helper.py
    Search WorkflowAppRunner|AdvancedChat.*Runner|workflow_runner|run draft workflow|DEBUGGER in core

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 我再把异步任务和 AdvancedChatAppRunner 补齐，这样能看到“请求返回 SSE 后，实际是谁在跑图”。

• Explored
  └ Read app_runner.py, app_generator.py, workflow_execute_task.py
    Search fetch\(|post\(|event-source|text/event-stream|workflow_run_id|node_started|node_finished|workflow_finished
    in
           workflow-app

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 任务文件路径我搜错了一层目录，补一下 Celery task 和前端 ssePost 监听逻辑。

• Explored
  └ Read use-workflow-run.ts
    Search workflow_based_app_execution_task|class AppExecutionParams in api
    Search ssePost\( in web

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 前端这边确认了：它不是普通 fetch，而是 ssePost，POST 建立后直接按 SSE 事件流消费。接下来我把 ssePost 的事件解析和后
  端任务执行体连起来。

• Explored
  └ Read base.ts, workflow_execute_task.py, app_runner.py

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 后端执行体已经串起来了。我再看前端 handleStream 把哪些后端事件翻成画布状态，回答会更完整。

• Explored
  └ Search function handleStream|const handleStream|workflow_started|node_started|node_finished|workflow_finished|
           workflow_paused|human_input_required in base.ts
    Read base.ts

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 再补最后一段：API 线程怎么订阅任务发布出来的 Redis 事件。这个是 SSE 能持续收到节点状态的关键。

• Explored
  └ Search def retrieve_events|get_response_topic|convert_to_event_stream in message_based_app_generator.py
    Read message_based_app_generator.py

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 还差一个点：DEBUGGER 模式下拿的是草稿工作流还是已发布版本。我把 _get_workflow 看一下，避免这里说错。

• Explored
  └ Read app_generate_service.py

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 这条链路是“前端建立 SSE 连接，后端异步起任务，任务执行工作流，再把节点事件通过 Redis 推回这个 SSE”。

  执行链路

  1. 前端点击运行后，handleRun 会在高级对话模式下把 URL 组装成
     web/app/components/workflow-app/hooks/use-workflow-run.ts:221

  /apps/${appDetail.id}/advanced-chat/workflows/draft/run

  然后不是普通 fetch，而是直接调用 ssePost(...) 发起 POST，并持续读取返回的 text/event-stream。入口在：
  web/app/components/workflow-app/hooks/use-workflow-run.ts:844
  web/service/base.ts:184
  handleStream 会把后端发回来的 workflow_started、node_started、node_finished、workflow_finished 等事件分发给前端状态
  更新逻辑。

  2. 后端 controller 收到请求后进入：
     api/controllers/console/app/workflow.py:313

  它做的事很薄：

  - 校验请求体 AdvancedChatWorkflowRunPayload
  - 取当前用户
  - 调 AppGenerateService.generate(..., invoke_from=InvokeFrom.DEBUGGER, streaming=True)
  - 再用 helper.compact_generate_response(...) 把生成器包装成 SSE 响应
    对应 SSE 包装在：
    api/libs/helper.py:289

  3. 真正的分发逻辑在 AppGenerateService.generate：
     api/services/app_generate_service.py:73

  对 AppMode.ADVANCED_CHAT，它会：

  - 调 _get_workflow(app_model, InvokeFrom.DEBUGGER) 取工作流
  - 因为是 DEBUGGER，这里拿的是草稿工作流，不是发布版
    见： api/services/app_generate_service.py:350
  - 组装 AppExecutionParams
  - 返回一个“订阅事件流”的 generator
  - 在 SSE 真正开始订阅时，触发 Celery 任务 workflow_based_app_execution_task.delay(payload_json)

  这里有个关键设计：API 线程本身不直接跑工作流，它先订阅事件，再把执行任务丢给 Celery。

  4. SSE 订阅的是基于 workflow_run_id 的 Redis topic：
     api/core/app/apps/message_based_app_generator.py:302

  - get_response_topic(app_mode, workflow_run_id)
  - retrieve_events(...)
  - stream_topic_events(...)

  所以 API 返回给前端的 SSE，本质是在消费这个 topic 上的事件。

  5. Celery worker 真正执行工作流的入口在：
     api/tasks/app_generate/workflow_execute_task.py:256

  workflow_based_app_execution_task 会：

  - 反序列化 AppExecutionParams
  - 创建 _AppRunner
  - _AppRunner.run() 再调用 AdvancedChatAppGenerator().generate(...)
    对应： api/tasks/app_generate/workflow_execute_task.py:149

  6. AdvancedChatAppGenerator.generate(...) 会先准备执行上下文：
     api/core/app/apps/advanced_chat/app_generator.py:91

  主要做这些事：

  - 校验 query
  - 取/建 conversation
  - 解析 files
  - 构造 AdvancedChatAppGenerateEntity
  - 创建 workflow execution / node execution repository
  - 创建 AdvancedChatAppRunner
  - 最后 runner.run()

  7. 真正“跑图”的地方在：
     api/core/app/apps/advanced_chat/app_runner.py:88

  AdvancedChatAppRunner.run() 会：

  - 构造系统变量 SystemVariable
  - 初始化 VariablePool
  - 用草稿工作流的 graph_dict 初始化 graph
  - 创建 WorkflowEntry
  - 挂上持久化层 WorkflowPersistenceLayer
  - 调 workflow_entry.run()
  - 遍历 generator，把每个运行事件交给 _handle_event(...)

  这里的 workflow_entry.run() 就是工作流图执行引擎真正开始跑节点的地方。

  8. Worker 侧产生的事件会被发布到 Redis topic：
     api/tasks/app_generate/workflow_execute_task.py:239

  _publish_streaming_response(...) 会把 generator 里出来的每个 event json.dumps 后 topic.publish(...)。

  9. API 线程已经在 retrieve_events(...) 上订阅这个 topic，所以这些事件会被立刻转成 SSE，前端 handleStream(...) 再把
     它们映射到：

  - workflow_started
  - node_started
  - node_finished
  - iteration_*
  - loop_*
  - text_chunk
  - workflow_finished
  - workflow_paused
  - human_input_required

  前端对应解析点在：
  web/service/base.ts:292

  一句话总结

  前端发起 /advanced-chat/workflows/draft/run 后，并不是 HTTP 请求线程直接执行工作流；它先建立 SSE 订阅，后端再把“执
  行草稿工作流”的任务扔给 Celery，Celery 里的 AdvancedChatAppRunner -> WorkflowEntry.run() 真正跑图，运行事件通过
  Redis 回流到原来的 SSE 连接，前端据此实时更新节点状态。

  如果你要，我可以继续把 workflow_entry.run() 往下拆到“单个节点是怎么调度执行的”。

─ Worked for 1m 49s ─────────────────────────────────────────────────────────────────────────────────────────────────


› 好的


• 我继续往下拆 WorkflowEntry.run() 到节点调度层，重点看图引擎如何推进节点、产出事件。

• Explored
  └ Read workflow_entry.py
    Search class GraphEngine|def run\(|yield .*node_started|yield .*workflow_started|yield .*workflow_finished|
           yield .*node_finished in dify_graph
    Read graph_engine.py, worker.py, base.py

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 已经看到引擎骨架了。我再补一层 Dispatcher/EventHandler，这样就能解释“节点跑完后，后继节点为什么会进入 ready
  queue”。

• Explored
  └ Read event_management.py, orchestration.py, node.py

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 节点本身的执行已经清楚了：Worker 从 ready queue 取 node，node.run() 先发 NodeRunStartedEvent，再发成功/失败/流式事
  件。现在补“谁把后继节点推进 ready queue”。

• Explored
  └ Search class Dispatcher|class EventHandler|handle.*NodeRunSucceededEvent|ready_queue|enqueue|mark.*ready in
           graph_engine
    List graph_engine
    Read dispatcher.py, event_handlers.py, graph_state_manager.py

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 再往下一层，就是 WorkflowEntry.run() 把执行交给 GraphEngine.run()：

  api/core/workflow/workflow_entry.py:116

  这里的核心不是业务逻辑，而是初始化一个图执行引擎 GraphEngine，然后 yield from graph_engine.run()。

  图引擎怎么跑

  GraphEngine.run() 在这里：

  api/dify_graph/graph_engine/graph_engine.py:217

  它做的事可以概括成 4 步：

  1. 初始化 layer，并先产出一个 GraphRunStartedEvent
  2. 启动 worker pool 和 dispatcher
  3. 持续从 EventManager.emit_events() 往外吐事件
  4. 根据最终状态吐出 GraphRunSucceededEvent / GraphRunPausedEvent / GraphRunFailedEvent

  所以你前端看到的 workflow_started / workflow_finished，本质上就是这层 graph event 再往上转换后的结果。

  节点是怎么执行的

  真正执行节点的是 worker 线程：

  api/dify_graph/graph_engine/worker.py:90

  worker 会循环做这件事：

  - 从 ready_queue 取一个 node_id
  - 找到对应 node
  - 调 node.run()
  - 把 node 运行过程中产生的 event 丢进 event_queue

  而每个 node 的公共执行入口在：

  api/dify_graph/nodes/base/node.py:309

  node.run() 的固定模式是：

  1. 先产出 NodeRunStartedEvent
  2. 调子类实现的 _run()
  3. _run() 要么直接返回 NodeRunResult
  4. 要么边跑边产出流式事件，比如 chunk / iteration / loop / agent log
  5. 最后产出 NodeRunSucceededEvent 或 NodeRunFailedEvent

  所以节点事件不是前端层硬编码出来的，是每个 node 在 graph engine 里自然跑出来的。

  后继节点为什么会继续跑

  这一步由 dispatcher + event handler 完成。

  dispatcher 在：

  api/dify_graph/graph_engine/orchestration/dispatcher.py:28

  它会不断从 event_queue 取事件，然后交给 EventHandler.dispatch(...)。

  真正决定“下一个节点是谁”的逻辑在：

  api/dify_graph/graph_engine/event_management/event_handlers.py:129

  当收到 NodeRunSucceededEvent 时，它会：

  - 记录当前节点已完成
  - 把节点输出写进 variable_pool
  - 调 edge_processor.process_node_success(...) 算出哪些下游节点 ready
  - 把这些 ready 节点重新 enqueue 到 ready_queue
  - worker 再去取这些节点执行

  入队操作在：

  api/dify_graph/graph_engine/graph_state_manager.py:42

  也就是：

  self._graph.nodes[node_id].state = NodeState.TAKEN
  self._ready_queue.put(node_id)

  这就是整个工作流“一个节点跑完，触发下一个节点”的核心闭环。

  和前端事件的对应关系

  整条链路可以压缩成：

  1. 前端 ssePost(...) 发请求
  2. controller 调 AppGenerateService.generate(...)
  3. Celery 里 workflow_based_app_execution_task
  4. AdvancedChatAppGenerator -> AdvancedChatAppRunner -> WorkflowEntry.run()
  5. GraphEngine.run() 启 worker / dispatcher
  6. worker 执行 node，产出 node events
  7. dispatcher 处理 node events，推进后继节点
  8. 事件通过 Redis topic 发布
  9. API 侧 SSE 订阅这个 topic
  10. 前端 handleStream(...) 把事件更新到画布和运行面板

  如果你要继续深挖，我下一步可以直接拆某一种节点，比如 LLM node 或 Answer node，看单个节点 _run() 里具体怎么调用模型
  和写回输出。
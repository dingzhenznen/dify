• 这个接口的后端定义在：

  api/controllers/console/app/workflow.py:313

  对应路由是：

  @console_ns.route("/apps/<uuid:app_id>/advanced-chat/workflows/draft/run")
  class AdvancedChatDraftWorkflowRunApi(Resource):

  所以你看到的完整请求路径：

  /console/api/apps/94622e65-a83b-43f8-9ee8-9834895173ab/advanced-chat/workflows/draft/run

  其中 94622e65-a83b-43f8-9ee8-9834895173ab 就是 <uuid:app_id>。

  前端调用位置在：

  web/app/components/workflow-app/hooks/use-workflow-run.ts:221

  这里会在 appDetail.mode === AppModeEnum.ADVANCED_CHAT 时组装：

  url = `/apps/${appDetail.id}/advanced-chat/workflows/draft/run`

  如果你要继续往下追这个接口实际执行逻辑，我可以继续帮你把 controller -> service -> 具体运行入口 一路定位出来。

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────


› 前端发起请求后工作流是如何执行的


• 我沿着这条链路往下追：前端如何发请求、后端 controller 进哪个 service、再到工作流调度和流式返回。

• Explored
  └ Read workflow.py, use-workflow-run.ts
    Search class AppGenerateService|def generate\( in api
    Search compact_generate_response|def compact_generate_response in api

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 入口已经清楚了。现在继续追 AppGenerateService.generate 里怎么根据 ADVANCED_CHAT 切到工作流执行器，以及流式事件怎么
  回前端。

• Explored
  └ Read app_generate_service.py, app_generator.py, helper.py
    Search WorkflowAppRunner|AdvancedChat.*Runner|workflow_runner|run draft workflow|DEBUGGER in core

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 我再把异步任务和 AdvancedChatAppRunner 补齐，这样能看到“请求返回 SSE 后，实际是谁在跑图”。

    in
           workflow-app

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    Search workflow_based_app_execution_task|class AppExecutionParams in api
    Search ssePost\( in web

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
• Explored
  └ Read base.ts, workflow_execute_task.py, app_runner.py

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
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
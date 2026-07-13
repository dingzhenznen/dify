'/Users/dingzhenzhen/works/study/dify/api/core/workflow/workflow_entry.py'
  _trace_node_run 时对单个node 进行追踪，对于整个工作流时如何追踪的


• 我先沿着 workflow_entry.py 里的节点追踪往上看，定位“整条工作流”的 trace 是在哪一层挂
  进去的，以及 workflow run 和 node run 的 span/trace 分工。


───────────────────────────────────────────────────────────────────────────────────────

• 我已经确认 ObservabilityLayer 只负责 node span。现在继续看 workflow_entry.py 里你提到
  的 _trace_node_run，以及整个 workflow span 是在哪个外层建立的。
 

  └ Read __init__.py, workflow_entry.py, persistence.py, ops_trace_manager.py
    Search TraceTaskName\.WORKFLOW|WORKFLOW_TRACE|WORKFLOW_RUN|WORKFLOW_APP_RUNNER|
    workflow
           trace in ops
    Read persistence.py, ops_trace_manager.py, trace_entity.py

───────────────────────────────────────────────────────────────────────────────────────

• 节点级 span 和工作流级 trace 是两套机制叠加的。我再把 WORKFLOW_TRACE 的生成逻辑打开，
  看整条工作流 trace 最终是从 workflow_run_id 聚合哪些数据。

• Explored

• Explored
  └ Read ops_trace_manager.py, app_runner.py, app_generator.py, base.py,
         workflow_app_runner_handler.py, generate_handler.py

───────────────────────────────────────────────────────────────────────────────────────

• 不是靠 _traced_node_run() 追整个工作流的。

  _traced_node_run() 在 api/core/workflow/workflow_entry.py:620 这段，本质上只是把“单个
  node 的 node.run()”包一层 ObservabilityLayer，产出 node 级 span。它更像单节点执行时的
  兜底 tracing。

  整条工作流的追踪分成 3 层：

  1. 工作流级 OTel span
  外层 span 是通过装饰器挂上的，不在 workflow_entry.py 里。

  - api/services/app_generate_service.py:76 上有 @trace_span(AppGenerateHandler)
  - api/core/app/apps/workflow/app_runner.py:72 上有
    @trace_span(WorkflowAppRunnerHandler)

  对应 handler 在：

  - api/extensions/otel/decorators/handlers/generate_handler.py:13
  - api/extensions/otel/decorators/handlers/workflow_app_runner_handler.py:13

  这里会创建“整个一次 workflow run”的外层 span，带上 app_id、tenant_id、workflow_id、
  user_id、streaming 这些属性。

  2. 节点级 OTel span
  节点 span 是 GraphEngine 上挂的 ObservabilityLayer 负责的。

  挂载点在 api/core/workflow/workflow_entry.py:248：

  - self.graph_engine.layer(ObservabilityLayer())

  实现看 api/core/app/workflow/layers/observability.py:43：

  - on_node_run_start() 创建 node span
  - on_node_run_end() 结束 node span
  - span 的 parent context 取 context_api.get_current()

  所以如果外层 WorkflowAppRunner.run span 已经在当前上下文里，node span 会自然挂到它下
  面，形成：
  workflow run span -> node span -> node 内部 HTTP/DB/LLM 自动埋点 span

  3. 工作流级“业务 trace”
  这不是 OTel span，而是 Dify 自己的 workflow trace 产物。

  关键在 api/core/app/workflow/layers/persistence.py:80：

  - GraphRunStartedEvent 时创建 WorkflowExecution
  - GraphRunSucceeded/Failed/Paused/... 时更新执行状态
  - 完成后调用 _enqueue_trace_task(...)

  代码在 api/core/app/workflow/layers/persistence.py:418：

  - 构造 TraceTask(TraceTaskName.WORKFLOW_TRACE, workflow_execution=execution, ...)
  - 然后交给 TraceQueueManager

  这个 layer 是在 api/core/app/apps/workflow/app_runner.py:186 挂到 graph_engine 上的。

  之后 TraceQueueManager 会执行 WORKFLOW_TRACE，走到 api/core/ops/
  ops_trace_manager.py:770 的 workflow_trace(...)，再根据 workflow_run_id 聚合整次运行
  的数据：

  - run 基本信息
  - inputs / outputs
  - status
  - elapsed_time
  - token 统计
  - conversation/message 关联
  - external trace / parent trace context

  具体聚合逻辑在 api/core/ops/ops_trace_manager.py:820

  结论
  - OTel 层面：靠 @trace_span(...) 的外层 workflow span + ObservabilityLayer 的 node 子
    span
  - Dify 业务 trace 层面：靠 WorkflowPersistenceLayer 监听整个 GraphEngine 的 graph 事
    件，落 WorkflowExecution，再 enqueue WORKFLOW_TRACE

  所以整条工作流不是通过 _traced_node_run() 追的；那个只覆盖单 node。整条 workflow 的主
  入口是 WorkflowAppRunner.run() 上的 span，主数据源是 WorkflowPersistenceLayer 的
  graph-level event 监听。

  如果你要，我可以下一步把这三层关系画成一张简化调用图。

─ Worked for 2m 11s ───────────────────────────────────────────────────────────────────


› 好的


• 可以，简化成这张图：

  HTTP Request
    -> AppGenerateService.generate()                [工作流外层 span #1]
      -> WorkflowAppGenerator.generate()
        -> WorkflowAppRunner.run()                 [工作流外层 span #2]
          -> WorkflowEntry.run()
            -> GraphEngine.run()
              -> WorkflowPersistenceLayer.on_event(...)
                 - GraphRunStarted -> 创建 WorkflowExecution
                 - NodeRunStarted/Succeeded/... -> 落节点执行记录
                 - GraphRunSucceeded/Failed/... -> 更新 WorkflowExecution
                 - 完成后 _enqueue_trace_task(WORKFLOW_TRACE)

              -> ObservabilityLayer.on_node_run_start/end(...)
                 - 为每个 node 创建/结束 node span

              -> Node.run()
                 -> 节点内部 HTTP / DB / LLM 调用
                 -> 自动挂到当前 node span 下

  对应职责：

  - AppGenerateService.generate / WorkflowAppRunner.run
    负责“整次 workflow run”的外层 OTel span
  - ObservabilityLayer
    负责“每个节点”的 OTel span
    文件：api/core/app/workflow/layers/observability.py:43
  - WorkflowPersistenceLayer
    负责“整次 workflow run”的业务追踪数据
    文件：api/core/app/workflow/layers/persistence.py:80
  - TraceQueueManager -> WORKFLOW_TRACE
    负责把 workflow_run_id 聚合成最终 trace 信息
    文件：api/core/ops/ops_trace_manager.py:820

  你可以把它理解成：

  - _traced_node_run() = 单节点 tracing 工具
  - ObservabilityLayer = 图运行时的节点 tracing
  - WorkflowPersistenceLayer + WORKFLOW_TRACE = 整个工作流运行的业务 trace
  - @trace_span(...) = 整个调用链最外层 OTel span

  如果你愿意，我可以继续把“一个 node span 是怎么挂到 workflow span 下面的 context 传递
  链”也拆开给你看
第5步 是如何启动 worker 的


• 我把第 5 步展开到 GraphEngine._start_execution() 和 WorkerPool.start()，直接看 worker 是怎么创建、启动、扩缩容的。

• Explored
  └ Read graph_engine.py, worker_pool.py, execution_coordinator.py

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• 第 5 步就是在 GraphEngine._start_execution() 里直接启动 WorkerPool：

  api/dify_graph/graph_engine/graph_engine.py:303

  self._worker_pool.start()

  这一步本身还没开始跑节点，它只是先把 worker 线程池拉起来。随后才会：

  - 注册 response node
  - 把 root node 或 resume 的节点塞进 ready_queue
  - 启动 dispatcher

  也就是说顺序是：

  1. worker_pool.start()
  2. enqueue root node
  3. dispatcher.start()

  worker_pool.start() 具体做了什么

  实现在：

  api/dify_graph/graph_engine/worker_management/worker_pool.py:56

  它做三件事：

  1. 标记池子进入 running 状态
  2. 根据图里节点数计算初始 worker 数量
  3. 循环调用 _create_worker() 创建并启动线程

  初始 worker 数量规则是：

  - 节点数 < 10：min_workers
  - 节点数 < 50：min(min_workers + 1, max_workers)
  - 否则：min(min_workers + 2, max_workers)

  这些配置来自 GraphEngineConfig，也就是前面 WorkflowEntry 初始化 GraphEngine 时传进去的配置。

  单个 worker 是怎么被启动的

  _create_worker() 在这里：

  api/dify_graph/graph_engine/worker_management/worker_pool.py:104

  它会：

  - 生成一个 worker_id
  - new 一个 Worker(...)
  - 调 worker.start()
  - 把这个 worker 放进 self._workers

  这里的 worker.start() 是 Python threading.Thread.start()，所以每个 worker 本质上就是一个后台线程。

  worker 线程启动后在干什么

  线程入口在：

  api/dify_graph/graph_engine/worker.py:78

  它启动后会一直循环：

  - 从 ready_queue.get(timeout=0.1) 取 node_id
  - 找到 graph.nodes[node_id]
  - 执行 node.run()
  - 把 node 产生的 event 放进 event_queue

  所以 worker 线程是先启动并阻塞等待任务，等 GraphEngine._start_execution() 后面把 root node enqueue 进去之后，它们才
  真正开始干活。

  谁负责扩缩容

  不是 start() 一次性定死。dispatcher 循环里会定期调用：

  api/dify_graph/graph_engine/orchestration/execution_coordinator.py:37

  self._worker_pool.check_and_scale()

  而 check_and_scale() 会根据：

  - ready_queue 深度
  - 当前 worker 数
  - idle worker 数
  - scale_up_threshold
  - scale_down_idle_time

  决定是否新增或回收 worker。

  一句话说，第 5 步不是“启动一个执行器对象”，而是“先启动一组后台线程，这些线程盯着 ready_queue，等 root node 被入队后
  立即开始执行”。

  如果你要，我可以继续把“root node 是在哪里被识别出来并放进 ready_queue 的”再拆一层。
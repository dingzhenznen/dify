不是 worker.run() 自己“找下一个节点”。

  worker 只负责这件事：

  1. 从 _ready_queue 取一个 node_id
  2. 执行这个 node
  3. 把 node 执行过程中产生的 event 放进 _event_queue

  代码就在这里：

  api/dify_graph/graph_engine/worker.py:90

  node_id = self._ready_queue.get(timeout=0.1)
  node = self._graph.nodes[node_id]
  self._execute_node(node)

  所以“下一个节点是谁”不是 worker 算的，而是另一条链路算出来后，再放回 _ready_queue，worker 下一轮循环再取。

  完整回路

  1. worker 执行当前节点
     当前节点的 node.run() 会产出 NodeRunStartedEvent、NodeRunSucceededEvent、NodeRunFailedEvent 等事件。
     见： api/dify_graph/nodes/base/node.py:309
  2. worker 把这些事件塞进 _event_queue
     见： api/dify_graph/graph_engine/worker.py:126

  for event in node_events:
      self._event_queue.put(event)

  3. dispatcher 从 _event_queue 取事件并分发
     见： api/dify_graph/graph_engine/orchestration/dispatcher.py:78

  event = self._event_queue.get(timeout=0.1)
  self._event_handler.dispatch(event)

  4. EventHandler 在节点成功时，调用 EdgeProcessor 计算下游哪些节点 ready
     见： api/dify_graph/graph_engine/event_management/event_handlers.py:129

  ready_nodes, edge_streaming_events = self._edge_processor.process_node_success(event.node_id)

  如果是分支节点，则走：

  ready_nodes, edge_streaming_events = self._edge_processor.handle_branch_completion(
      event.node_id, event.node_run_result.edge_source_handle
  )

  5. EdgeProcessor 遍历当前节点的 outgoing edges，标记 edge 为 taken/skipped，并判断 edge 的下游节点是否 ready
     见： api/dify_graph/graph_engine/graph_traversal/edge_processor.py:58

  对于普通节点：

  outgoing_edges = self._graph.get_outgoing_edges(node_id)
  for edge in outgoing_edges:
      nodes, events = self._process_taken_edge(edge)

  而 _process_taken_edge() 里会做两件关键事：

  - mark_edge_taken(edge.id)
  - is_node_ready(edge.head)

  见： api/dify_graph/graph_engine/graph_traversal/edge_processor.py:111

  self._state_manager.mark_edge_taken(edge.id)

  if self._state_manager.is_node_ready(edge.head):
      ready_nodes.append(edge.head)

  6. EventHandler 把 ready 的下游节点重新塞回 _ready_queue
     见： api/dify_graph/graph_engine/event_management/event_handlers.py:153

  for node_id in ready_nodes:
      self._state_manager.enqueue_node(node_id)
      self._state_manager.start_execution(node_id)

  而 enqueue_node() 真正做的是：

  api/dify_graph/graph_engine/graph_state_manager.py:42

  self._graph.nodes[node_id].state = NodeState.TAKEN
  self._ready_queue.put(node_id)

  7. worker 下一轮循环再从 _ready_queue 里取这个新节点执行
     也就是又回到：

  node_id = self._ready_queue.get(timeout=0.1)

  一句话总结

  worker 并不会在当前节点结束后直接“跳到下一个节点”。
  真正的机制是：

  worker 执行当前节点 -> 把结果事件放进 event_queue -> dispatcher/EventHandler 根据边关系算出 ready 的下游节点 -> 把
  这些节点重新放进 ready_queue -> 任意空闲 worker 再取出来继续执行。

  如果你要，我可以继续画一个“单节点完成后的时序图”，把 worker / dispatcher / edge_processor / ready_queue 四者关系画
  成非常直观的流程。

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────


› 好的


• 可以，单节点完成后的时序基本就是这样：

  Worker
    |
    | 1. 从 ready_queue 取出当前 node_id
    v
  执行 node.run()
    |
    | 2. 产出事件：started / chunk / succeeded / failed
    v
  event_queue.put(event)
    |
    v
  Dispatcher
    |
    | 3. 从 event_queue 取事件
    v
  EventHandler.dispatch(event)
    |
    | 4. 如果是 NodeRunSucceededEvent
    |    - 保存当前节点 outputs
    |    - 调 EdgeProcessor 处理 outgoing edges
    v
  EdgeProcessor
    |
    | 5. 遍历当前节点的下游边
    |    - mark edge taken/skipped
    |    - 判断下游节点是否 ready
    v
  返回 ready_nodes
    |
    v
  EventHandler
    |
    | 6. 把 ready_nodes 入队
    v
  ready_queue.put(next_node_id)
    |
    v
  任意空闲 Worker
    |
    | 7. 再次从 ready_queue 取节点执行
    v
  下一个 node.run()

  对应代码点：

  - worker 执行节点
    api/dify_graph/graph_engine/worker.py:100
  - worker 把事件塞进 event_queue
    api/dify_graph/graph_engine/worker.py:126
  - dispatcher 取事件并分发
    api/dify_graph/graph_engine/orchestration/dispatcher.py:78
  - 成功事件触发下游节点计算
    api/dify_graph/graph_engine/event_management/event_handlers.py:129
  - edge processor 判断哪些下游节点 ready
    api/dify_graph/graph_engine/graph_traversal/edge_processor.py:58
  - ready 节点重新入队
    api/dify_graph/graph_engine/event_management/event_handlers.py:153
    api/dify_graph/graph_engine/graph_state_manager.py:42

  关键点就一句：

  下一个节点不是当前 worker 直接算出来的，而是当前节点完成后，由 dispatcher/event handler 根据图的边关系算出 ready 节
  点，再放回 ready_queue，供后续 worker 消费。
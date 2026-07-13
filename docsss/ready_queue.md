ready_queue

woker线程 从 ready_queue 获取任务

结束后 把事件放入队列 ，EventHandler dispatch
然后调用 EdgeProcessor 来获取下一个 node节点 然后放入队列
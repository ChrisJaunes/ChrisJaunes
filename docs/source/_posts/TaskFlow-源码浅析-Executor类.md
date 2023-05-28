---
title: TaskFlow-源码浅析-Executor类
date: 2023-05-25 20:19:15
categories: 
- [TaskFlow]
tags:
- TaskFlow 
excerpt: 本文浅析了TaskFlow中的Executor
---

类Executor是负责执行taskflow的类。

## 构造函数

构造函数中创建了一个大小为N的线程池 _threads 和 大小为N的Worker队列 _workers，并在 _spawn 中初始化

首先看线程池的部分，可以看到在 _spawn 中有一个for循环，初始化了使用 lambda表达式 去初始化了 _threads 中的每一个线程，所有线程在初始化后将开始运行

其中 Worker& w, std::mutex& mutex, std::condition_variable& cond, size_t& n 都是引用，为了保证线程中正确使用，对应的实参使用了std::ref

```cpp
inline void Executor::_spawn(size_t N) {
    // ...
    for(size_t id=0; id<N; ++id) {
        // ...
        _threads[id] = std::thread([this] (
            Worker& w, std::mutex& mutex, std::condition_variable& cond, size_t& n
        ) -> void {
            // ...
        }, std::ref(_workers[id]), std::ref(mutex), std::ref(cond), std::ref(n));
    }
    // ...
}
```

在该lambda开始部分，worker的_threads指针指向自身线程，接着设置线程ID到worker ID的映射。

为了防止多线程读写_wids，mutex是主线程和_threads中所有线程共享的，对其上锁可以保证对_wids读写不会发生冲突。

lock是块级变量，在std::scoped_lock lock(mutex)中 构造函数调用mutex.lock上锁，在块结束时析构函数调用mutex.unlock释放锁

在主线程执行完for以后，将使用mutex构造unique_lock变量，使用condition_variable的wait函数堵塞主线程，wait被调用和会释放所在线程持有的 mutex 锁， 所以这里不会造成死锁

_threads里的各个线程会对启动的线程计数，需要的N个线程都启动后，使用cond.notify_one唤醒因cond.wait堵塞的主线程，构造函数完成

```cpp
inline void Executor::_spawn(size_t N) {
    std::mutex mutex;
    std::condition_variable cond;
    size_t n=0;
    for(size_t id=0; id<N; ++id) {
        // ...
        _threads[id] = std::thread([this] (
            Worker& w, std::mutex& mutex, std::condition_variable& cond, size_t& n
        ) -> void {
            w._thread = &_threads[w._id];
            {
                std::scoped_lock lock(mutex);
                _wids[std::this_thread::get_id()] = w._id;
                if(n++; n == num_workers()) {
                cond.notify_one();
                }
            }
            // ...
        }, std::ref(_workers[id]), std::ref(mutex), std::ref(cond), std::ref(n));
    }
    std::unique_lock<std::mutex> lock(mutex);
    cond.wait(lock, [&](){ return n==N; });
}
```
{% spoiler "Executor 初始化 实现相关源码" %}
[executor.hpp tf:Executor::Executor 实现](https://github.com/taskflow/taskflow/blob/master/taskflow/core/executor.hpp#L1107-L1124)
[executor.hpp tf:Executor::_spawn 实现](https://github.com/taskflow/taskflow/blob/master/taskflow/core/executor.hpp#L1170-L1241)
```cpp
class Executor {
    std::vector<std::thread> _threads;
    std::vector<Worker> _workers
}
// Constructor
inline Executor::Executor(size_t N, std::shared_ptr<WorkerInterface> wix) :
  _MAX_STEALS {((N+1) << 1)},
  _threads    {N},
  _workers    {N},
  _notifier   {N},
  _worker_interface {std::move(wix)} {

  if(N == 0) {
    TF_THROW("no cpu workers to execute taskflows");
  }

  _spawn(N);

  // instantite the default observer if requested
  if(has_env(TF_ENABLE_PROFILER)) {
    TFProfManager::get()._manage(make_observer<TFProfObserver>());
  }
}
// Procedure: _spawn
inline void Executor::_spawn(size_t N) {

  std::mutex mutex;
  std::condition_variable cond;
  size_t n=0;

  for(size_t id=0; id<N; ++id) {

    _workers[id]._id = id;
    _workers[id]._vtm = id;
    _workers[id]._executor = this;
    _workers[id]._waiter = &_notifier._waiters[id];

    _threads[id] = std::thread([this] (
      Worker& w, std::mutex& mutex, std::condition_variable& cond, size_t& n
    ) -> void {
      
      // assign the thread
      w._thread = &_threads[w._id];

      // enables the mapping
      {
        std::scoped_lock lock(mutex);
        _wids[std::this_thread::get_id()] = w._id;
        if(n++; n == num_workers()) {
          cond.notify_one();
        }
      }

      Node* t = nullptr;
      
      // before entering the scheduler (work-stealing loop), 
      // call the user-specified prologue function
      if(_worker_interface) {
        _worker_interface->scheduler_prologue(w);
      }
      
      // must use 1 as condition instead of !done because
      // the previous worker may stop while the following workers
      // are still preparing for entering the scheduling loop
      std::exception_ptr ptr{nullptr};
      try {
        while(1) {

          // execute the tasks.
          _exploit_task(w, t);

          // wait for tasks
          if(_wait_for_task(w, t) == false) {
            break;
          }
        }
      } 
      catch(...) {
        ptr = std::current_exception();
      }
      
      // call the user-specified epilogue function
      if(_worker_interface) {
        _worker_interface->scheduler_epilogue(w, ptr);
      }

    }, std::ref(_workers[id]), std::ref(mutex), std::ref(cond), std::ref(n));
    
  }

  std::unique_lock<std::mutex> lock(mutex);
  cond.wait(lock, [&](){ return n==N; });
}
```
{% endspoiler %}

## 消费者

thread 接受了一个lambda作为运行函数，处理了worker和threads中的映射后，剩下部分是一个死循环，不停的消费 worker 的 wsq 中的内容

有趣的是，这里是先执行_exploit_task，再执行_wait_for_task，而不是相反的操作。

这个死循环只有当 _wait_for_task 的返回值为false时才会结束；而 _wait_for_task 只有 _done 才会返回false；而 _done 只用在 Excutor被析构的时候才会置为true

```cpp
inline void Executor::_spawn(size_t N) {
    //...
    _threads[id] = std::thread([this] (
        Worker& w, std::mutex& mutex, std::condition_variable& cond, size_t& n
    ) -> void {
        // ...
        while(1) {
          // execute the tasks.
          _exploit_task(w, t);
          // wait for tasks
          if(_wait_for_task(w, t) == false) {
            break;
          }
        }
    }, std::ref(_workers[id]), std::ref(mutex), std::ref(cond), std::ref(n));
    // ...
}
```

### _exploit_task
其中, _exploit_task 对获取的任务(实际是Node*), 交由 _invoke 处理，然后从worker的TaskQueue中获取下一个

_invoke 在执行完 Node 后，会把新的可以执行 Node 放入当前 worker 的 _wsq

```cpp
// Procedure: _exploit_task
inline void Executor::_exploit_task(Worker& w, Node*& t) {
  while(t) {
    _invoke(w, t);
    t = w._wsq.pop();
  }
}
```

### _wait_for_task
在工作队列为空后，_wait_for_task 将使用 _explore_task 获取一个Node

当 _explore_task 成功获取到 Node 时会尝试唤醒另一个线程避免饿死

tips：线程饿死（Thread Starvation）是指某些线程在系统中无法获得所需的资源而无限期地等待的情况。这些资源可以是CPU时间、内存、锁、文件句柄等。线程饿死可能会导致系统性能下降和资源浪费。

源码这里的描述也很灵性：The last thief who successfully stole a task will wake up another thief worker to avoid starvation.

在没有获取到 Node 的时候，遍历所有的worker，检查他们的 _wsq 然后找到一个受害者， 这里的命名很有趣，需要去偷取任务的WorkerID被命名成了vtm(victim), 也就是受害者的意思，实属生动形象了

这里2PC guard是两阶段提交，一种常见的安全机制，用于确保只有所有参与方同意执行事务时才会执行。

{% spoiler "Executor _wait_for_task 实现相关源码" %}
```cpp
// Function: _wait_for_task
inline bool Executor::_wait_for_task(Worker& worker, Node*& t) {

  explore_task:

  _explore_task(worker, t);
  
  // The last thief who successfully stole a task will wake up
  // another thief worker to avoid starvation.
  if(t) {
    _notifier.notify(false);
    return true;
  }

  // ---- 2PC guard ----
  _notifier.prepare_wait(worker._waiter);

  if(!_wsq.empty()) {
    _notifier.cancel_wait(worker._waiter);
    worker._vtm = worker._id;
    goto explore_task;
  }
  
  if(_done) {
    _notifier.cancel_wait(worker._waiter);
    _notifier.notify(true);
    return false;
  }
  
  // We need to use index-based scanning to avoid data race
  // with _spawn which may initialize a worker at the same time.
  for(size_t vtm=0; vtm<_workers.size(); vtm++) {
    if(!_workers[vtm]._wsq.empty()) {
      _notifier.cancel_wait(worker._waiter);
      worker._vtm = vtm;
      goto explore_task;
    }
  }
  
  // Now I really need to relinguish my self to others
  _notifier.commit_wait(worker._waiter);

  goto explore_task;
}
```
{% endspoiler %}

#### _explore_task
过程如下:
1. 获取新的任务，指去一个Worker中偷取(steal)一个任务
2. 如果偷到了就直接返回了
3. 如果没偷到且偷得过于频繁(偷取次数超过_MAX_STEALS)就歇歇(堵塞当前线程)
4. 如果没偷到且歇歇的次数过于频繁(num_yields>100)就直接返回了
5. 其余情况再找一个受害者

流程图如下:
{% plantuml %}
@startuml
(*) --> "初始化"

if "随机到本线程id" then
  -->[true] "从Executor中的_wsq获取" 
  -->===B2===
else
  -->[false] "从对应线程的_wsq获取" 
  -->===B2===
endif

if "" then
  -->[成功获得任务] (*)
else
  -->[未能成功获得任务]if "" then
    -->[超过最大尝试偷取次数] "堵塞本线程"
    if "" then
        -->[超过堵塞次数] (*)
    else
        -->[没有超过堵塞次数] "随机线程id"
    endif
  else
    -->[没有超过最大尝试偷取次数] "随机线程id"
  endif
endif
"随机线程id" --> "初始化"
@enduml
{% endplantuml %}

源代码如下:
{% spoiler "Executor _explore_task 实现相关源码" %}
[github](https://github.com/taskflow/taskflow/blob/master/taskflow/core/executor.hpp#L1288-L1317)
```cpp
inline void Executor::_explore_task(Worker& w, Node*& t) {
  size_t num_steals = 0;
  size_t num_yields = 0;
  std::uniform_int_distribution<size_t> rdvtm(0, _workers.size()-1);
  // Here, we write do-while to make the worker steal at once
  // from the assigned victim.
  do {
    t = (w._id == w._vtm) ? _wsq.steal() : _workers[w._vtm]._wsq.steal();
    if(t) {
      break;
    }
    if(num_steals++ > _MAX_STEALS) {
      std::this_thread::yield();
      if(num_yields++ > 100) {
        break;
      }
    }
    w._vtm = rdvtm(w._rdgen);
  } while(!_done);
}
```
{% endspoiler %}

## 生产者

run 方法最终会调用 run_until(Taskflow& f, P&& p, C&& c)
{% spoiler "Executor run 实现相关源码" %}
```cpp
inline tf::Future<void> Executor::run(Taskflow& f) {
  return run_n(f, 1, [](){});
}
template <typename C>
tf::Future<void> Executor::run(Taskflow& f, C&& c) {
  return run_n(f, 1, std::forward<C>(c));
}
template <typename C>
tf::Future<void> Executor::run_n(Taskflow& f, size_t repeat, C&& c) {
  return run_until(
    f, [repeat]() mutable { return repeat-- == 0; }, std::forward<C>(c)
  );
}
template <typename P, typename C>
tf::Future<void> Executor::run_until(Taskflow& f, P&& p, C&& c) {
  _increment_topology();
  // Need to check the empty under the lock since dynamic task may
  // define detached blocks that modify the taskflow at the same time
  bool empty;
  {
    std::lock_guard<std::mutex> lock(f._mutex);
    empty = f.empty();
  }

  // No need to create a real topology but returns an dummy future
  if(empty || p()) {
    c();
    std::promise<void> promise;
    promise.set_value();
    _decrement_topology_and_notify();
    return tf::Future<void>(promise.get_future(), std::monostate{});
  }
  // create a topology for this run
  auto t = std::make_shared<Topology>(f, std::forward<P>(p), std::forward<C>(c));
  // need to create future before the topology got torn down quickly
  tf::Future<void> future(t->_promise.get_future(), t);
  // modifying topology needs to be protected under the lock
  {
    std::lock_guard<std::mutex> lock(f._mutex);
    f._topologies.push(t);
    if(f._topologies.size() == 1) {
      _set_up_topology(_this_worker(), t.get());
    }
  }
  return future;
}
```
{% endspoiler %}

_set_up_topology 获取了入度为0的Node，将这些 Node 组装成了一个 Node 的 list

最终调用了 _schedule 将这些Node放到了对应worker的_wsq中或Executor的_wsq，而消费者线程会不停的尝试从 _wsq 中获取 Node 并执行Node对应函数

{% spoiler "Executor _set_up_topology 实现相关源码" %}
```cpp
inline void Executor::_set_up_topology(Worker* worker, Topology* tpg) {
  // ---- under taskflow lock ----
  tpg->_sources.clear();
  tpg->_taskflow._graph._clear_detached();

  // scan each node in the graph and build up the links
  for(auto node : tpg->_taskflow._graph._nodes) {

    node->_topology = tpg;
    node->_parent = nullptr;
    node->_state.store(0, std::memory_order_relaxed);

    if(node->num_dependents() == 0) {
      tpg->_sources.push_back(node);
    }

    node->_set_up_join_counter();
  }

  tpg->_join_counter = tpg->_sources.size();

  if(worker) {
    _schedule(*worker, tpg->_sources);
  }
  else {
    _schedule(tpg->_sources);
  }
}
```
{% endspoiler %}

以下是调度方法的不同重载形式，将一个(Node* node)或多个(SmallVector<Node*>& nodes)放到对应worker的_wsq中，如果没有对应的worker就放入Executor的_wsq

{% spoiler "Executor _schedule 实现相关源码" %}
```cpp
// Procedure: _schedule
inline void Executor::_schedule(Worker& worker, Node* node) {
  
  // We need to fetch p before the release such that the read 
  // operation is synchronized properly with other thread to
  // void data race.
  auto p = node->_priority;

  node->_state.fetch_or(Node::READY, std::memory_order_release);

  // caller is a worker to this pool - starting at v3.5 we do not use
  // any complicated notification mechanism as the experimental result
  // has shown no significant advantage.
  if(worker._executor == this) {
    worker._wsq.push(node, p);
    _notifier.notify(false);
    return;
  }

  {
    std::lock_guard<std::mutex> lock(_wsq_mutex);
    _wsq.push(node, p);
  }

  _notifier.notify(false);
}

// Procedure: _schedule
inline void Executor::_schedule(Node* node) {
  
  // We need to fetch p before the release such that the read 
  // operation is synchronized properly with other thread to
  // void data race.
  auto p = node->_priority;

  node->_state.fetch_or(Node::READY, std::memory_order_release);

  {
    std::lock_guard<std::mutex> lock(_wsq_mutex);
    _wsq.push(node, p);
  }

  _notifier.notify(false);
}

// Procedure: _schedule
inline void Executor::_schedule(Worker& worker, const SmallVector<Node*>& nodes) {

  // We need to cacth the node count to avoid accessing the nodes
  // vector while the parent topology is removed!
  const auto num_nodes = nodes.size();

  if(num_nodes == 0) {
    return;
  }

  // caller is a worker to this pool - starting at v3.5 we do not use
  // any complicated notification mechanism as the experimental result
  // has shown no significant advantage.
  if(worker._executor == this) {
    for(size_t i=0; i<num_nodes; ++i) {
      // We need to fetch p before the release such that the read 
      // operation is synchronized properly with other thread to
      // void data race.
      auto p = nodes[i]->_priority;
      nodes[i]->_state.fetch_or(Node::READY, std::memory_order_release);
      worker._wsq.push(nodes[i], p);
      _notifier.notify(false);
    }
    return;
  }

  {
    std::lock_guard<std::mutex> lock(_wsq_mutex);
    for(size_t k=0; k<num_nodes; ++k) {
      auto p = nodes[k]->_priority;
      nodes[k]->_state.fetch_or(Node::READY, std::memory_order_release);
      _wsq.push(nodes[k], p);
    }
  }

  _notifier.notify_n(num_nodes);
}

// Procedure: _schedule
inline void Executor::_schedule(const SmallVector<Node*>& nodes) {

  // parent topology may be removed!
  const auto num_nodes = nodes.size();

  if(num_nodes == 0) {
    return;
  }

  // We need to fetch p before the release such that the read 
  // operation is synchronized properly with other thread to
  // void data race.
  {
    std::lock_guard<std::mutex> lock(_wsq_mutex);
    for(size_t k=0; k<num_nodes; ++k) {
      auto p = nodes[k]->_priority;
      nodes[k]->_state.fetch_or(Node::READY, std::memory_order_release);
      _wsq.push(nodes[k], p);
    }
  }

  _notifier.notify_n(num_nodes);
}
```
{% endspoiler %}

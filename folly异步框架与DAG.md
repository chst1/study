---
title: folly中异步框架future与DAG
date: 2023-01-08 16:23:18
tags:
    folly
categories:
    DAG
mathjax:
    true
description: 深入学习了一下folly的异步框架future和基于该异步框架实现的DAG调度，只是说是叹为观止，十分优雅。
---

# DAG设计

## 需求

DAG将执行流程构建一个单向无环图，其中每个点表示一个逻辑执行单元，有向边表示逻辑单元的依赖关系，其中入节点依赖出节点的执行结果。对于个人节点来说，所有执行其的节点（对该节点来说是入节点）执行完成后立即执行该节点。

对于无依赖关系可以同步执行的节点来说，使用线程池来统一并发执行，通过图编排尽可能的保证执行的高效性。

同时由于存再依赖关系，所已需要考虑某个节点其执行错误时，依赖其结果的所有节点处理逻辑，folly中使用的方式是当任意一个节点失败时，依赖其结果的节点都停止执行。

## 设计

对于所有依赖节点执行完成才能执行当前节点的实现首先想到的是屏障或者条件变量，但是folly都未使用，而是使用了更为优雅的方式：智能指针（原子变量），通过依赖的节点每个持有一个智能指针的拷贝，当依赖节点执行完成时会释放该智能指针，当所以智能指针都被释放时将执行析构函数，在析构函数中将执行当前节点（并非立即执行，而是放到线程池中）。

对于节点并发执行来说，每个节点可指定使用的线程池，当某个节点未指定使用的线程池将使用其依赖节点的线程池（依赖的最后执行完成的那个节点）。

在执行开始和完成都有两个单独节点，其中开始节点是第一层节点的依赖，由于设置触发，对于最后一个节点则是依赖所有的最后一层节点，用于监控所有节点执行完成。

# 使用样例

待补充，可看folly源码中test部分。

# 源码代码解析

这里对folly的DAG源码进行详细阅读。要学习DAG，首先要学习folly的future，其类似std提供的future，但是更加灵活。future的基类是core，我们先来了解一下core相关内容。

## core

core对应于文件`folly/futures/detail/Core.h`其包含了如下内容`State`,`Spin Lock`，`DeferredExecutor`，`KeepAliveOrDeferred`，`InterruptHandler`，`CoreBase`，`Core`。

其中core的核心是一个FSM即有限状态机。其状态流转如下图：

```
///   +----------------------------------------------------------------+
///   |                       ---> OnlyResult -----                    |
///   |                     /                       \                  |
///   |                  (setResult())             (setCallback())     |
///   |                   /                           \                |
///   |   Start --------->                              ------> Done   |
///   |     \             \                           /                |
///   |      \           (setCallback())           (setResult())       |
///   |       \             \                       /                  |
///   |        \              ---> OnlyCallback ---                    |
///   |         \           or OnlyCallbackAllowInline                 |
///   |          \                                  \                  |
///   |      (setProxy())                          (setProxy())        |
///   |            \                                  \                |
///   |             \                                   ------> Empty  |
///   |              \                                /                |
///   |               \                            (setCallback())     |
///   |                \                            /                  |
///   |                  --------> Proxy ----------                    |
///   +----------------------------------------------------------------+
```

本文介绍中暂不考虑`setProxy()`这条支路。

这里需要对一些名称进行一个定义：

1. result：即结果，一个Try类型数据，包含一个T类型数据或者一个exception。对于链式控制来说（使用then）上一个的结果会作为下一个的输出，当设置当前的result即表示上一个执行单元完成处理，对于当前执行单元来说，执行完成的结果会被设置到下一个执行单元，以此来触发下一个执行单元的执行，以此达到链式的控制及数据传递。
2. callback：回调函数。当设置了result时会执行函数并为下一个执行单元设置result。
3. executor：执行器。执行callback的结构，也会作为参数传递到callback中，可以暂时理解为线程池。
4. consumer thread：消费线程，这里的消费线程时指消费整个future执行结果的线程（而不是消费任务的线程），其提供`callback`。
5. producer thread：生产者线程，这里指生成结果的线程（而不是生成任务的线程）。
6. `interrupt`：exception_wrapper，表示异常结果。及try机制。
7. interrupt handler：出现异常时的执行函数。

core持有三组数据，每组数据都是并发控制的：

1. producer-to-consumer(生产者到消费者)的信息流：这组数据包含：`result`，`callback`，`executor`以及运行`callback`的·优先级。控制并发的方式是使用上图的FSM，其中有State表示状态，状态是原子的进行变更的。
2. consumer-to-producer(消费者到生产者)的请求干预：包含`interrupt-handler`和`interrupt`。通过`Spin Lock`控制并发。
3. 生命周期控制：包含了两个引用计数，均为原子变量。

这里为了区分生产者和消费者，并且为了方便使用，有两个结构共同维护该FSM（共用同一个core），分别是`future`和`promise`。其中`future`一般由消费者线程持有，消费者可以注册`callback`。`promise`一般由生产者线程持有，其接受上一层的`result`，执行`callback`，设置下一层的`result`。消费者持有的一般是下一层的`future`，当消费者判断持有的`future`已经设置了`result`则表示设置的`callback`已经执行完成。对于多层级执行链来说，处于中间的`future`和`promise`对用户来说均是不可见的，我们只能获取到最起点的`future`和`promise`，以及最后一个`future`。对于中间的`future`来说，其执行完成链路的构建就会被析构掉（由于不会被任何变量持有，生命周期结束），对于`promise`来说，会被生产者线程持有（准确的来说，其实不是被生产者线程持有，因为此时`callback`未被添加到生产者线程池中，其实际是被每个的上一层的`callback`持有），当最初的`promise`设置了`resul`t后，就会按照执行链路执行对应的`callback`。当每个`callback`执行完成时，会先设置下一个`promise`的`result`（此时就会触发下一层`callabck`执行），之后会执行自身`callback`的析构，此时就会析构`callback`持有的下一层的`promise`（在下一层开始执行`callback`时，会自己保证在执行完成`callback`前自生不被析构，因此不用担心此时`promise`被析构，同样在其执行完成`callback`再执行析构），通过这种方式达到链式执行效果。

上面介绍的可能相对较为复杂，很多概念还未提到，暂时可以先有一个大致的概念，不用现在就完全搞明白，后面会详细介绍。

为了实现上述功能，需要有很多基础的class，下面对这些class进行简单介绍。

### State

State相对较为简单，一个枚举类型来表示状态：

```c++
enum class State : uint8_t {
  Start = 1 << 0,
  OnlyResult = 1 << 1,
  OnlyCallback = 1 << 2,
  OnlyCallbackAllowInline = 1 << 3,
  Proxy = 1 << 4,
  Done = 1 << 5,
  Empty = 1 << 6,
};
constexpr State operator&(State a, State b) {
  return State(uint8_t(a) & uint8_t(b));
}
constexpr State operator|(State a, State b) {
  return State(uint8_t(a) | uint8_t(b));
}
constexpr State operator^(State a, State b) {
  return State(uint8_t(a) ^ uint8_t(b));
}
constexpr State operator~(State a) {
  return State(~uint8_t(a));
}
```

这里不过多介绍。

### SpinLock

`SpinLock`就是一个自己实现的自旋锁，这里可以将其当做一个简单的锁就好了：

```c++
/// SpinLock is and must stay a 1-byte object because of how Core is laid out.
struct SpinLock : private MicroSpinLock {
  SpinLock() : MicroSpinLock{0} {}

  using MicroSpinLock::lock;
  using MicroSpinLock::unlock;
};
static_assert(sizeof(SpinLock) == 1, "missized");
```

### DeferredExecutor

`DeferredExecutor`延迟执行器。其主要功能是持有一个函数，但不立即执行，等待时机成熟（达到某种条件时才执行）。

该class使用场景似乎很特殊，目前没有看到应用场景，读者可暂时忽略该部分（或者有大佬知道怎么用，辛苦不吝赐教）。

其定义如下：

```c++
class DeferredExecutor final {
 public:
  // addFrom will:
  //  * run func inline if there is a stored executor and completingKA matches
  //    the stored executor
  //  * enqueue func into the stored executor if one exists
  //  * store func until an executor is set otherwise
  /*
  如果已经存再executor，并且executor与completingKA一致，则立即执行函数
  如果已经存再executor，但不一致，则将函数添加到executor
  否则只存储func，直到设置了一个executor
  */
  void addFrom(
      Executor::KeepAlive<>&& completingKA,
      Executor::KeepAlive<>::KeepAliveFunc func);

  Executor* getExecutor() const;

  /*
  如果func为空，则只设置executor，否则将func添加进入executor，并设置executor
  递归的对所有nestedExecutors_中元素执行相同操作。
  */
  void setExecutor(folly::Executor::KeepAlive<> executor);

  void setNestedExecutors(std::vector<DeferredWrapper> executors);

  void detach();

  DeferredWrapper copy();

  static DeferredWrapper create();

 private:
  friend class UniqueDeleter;

  DeferredExecutor();

  void acquire();
  void release();

  enum class State { EMPTY, HAS_FUNCTION, HAS_EXECUTOR, DETACHED };

  std::atomic<State> state_{State::EMPTY};
  Executor::KeepAlive<>::KeepAliveFunc func_;
  folly::Executor::KeepAlive<> executor_;
  std::unique_ptr<std::vector<DeferredWrapper>> nestedExecutors_;
  std::atomic<ssize_t> keepAliveCount_{1};
};

class UniqueDeleter {
 public:
  void operator()(DeferredExecutor* ptr);
};

using DeferredWrapper = std::unique_ptr<DeferredExecutor, UniqueDeleter>;
```

其中`Executor::KeepAlive`就是folly自己实现的对`Executor`封装的安全指针：

```c++
class ExecutorKeepAliveBase {
 public:
  //  A dummy keep-alive is a keep-alive to an executor which does not support
  //  the keep-alive mechanism.
  static constexpr uintptr_t kDummyFlag = uintptr_t(1) << 0;

  //  An alias keep-alive is a keep-alive to an executor to which there is
  //  known to be another keep-alive whose lifetime surrounds the lifetime of
  //  the alias.
  static constexpr uintptr_t kAliasFlag = uintptr_t(1) << 1;

  static constexpr uintptr_t kFlagMask = kDummyFlag | kAliasFlag;
  static constexpr uintptr_t kExecutorMask = ~kFlagMask;
};

template <typename ExecutorT = Executor>
  class KeepAlive : private detail::ExecutorKeepAliveBase;
```

`KeepAlive`利用指针最后两位一定是0的特性，使用最好两位来做标识。`kDummyFlag`表示假的`keep-alive`，`kAliasFlag`标识当前的`Executor`是一个别名·。

`KeepAlive`封装的`ExecutorT`，正常都需要继承`Executor`，其提供了三个接口：`keepAliveAcquire`和`keepAliveRelease`，在两个接口用于提供引用计数计算的，默认返回值为false，表示不支持引用计数，此时的`KeepAlive`是一个假的支持`keep-alive`，此时执行copy操作时，直接返回`ExecutorT`的地址（加上kDummyFlag），如果继承`Executor`的类实现了上述两个接口，则`copy`返回指针（加上kAliasFlag）的同时，会增加引用计数。另一个接口时线程池都需要有的接口`add`，即向其中添加task任务，默认的函数时立即执行函数。



### KeepAliveOrDeferred

`KeepAliveOrDeferred`是一个折叠类型，包含一个`Executor::KeepAlive`或者`DeferredWrapper`。

```c++
/**
 * Wrapper type that represents either a KeepAlive or a DeferredExecutor.
 * Acts as if a type-safe tagged union of the two using knowledge that the two
 * can safely be distinguished.
 */
class KeepAliveOrDeferred {
 private:
  using KA = Executor::KeepAlive<>;
  using DW = DeferredWrapper;

 public:
  KeepAliveOrDeferred() noexcept;
  /* implicit */ KeepAliveOrDeferred(KA ka) noexcept;
  /* implicit */ KeepAliveOrDeferred(DW deferred) noexcept;
  KeepAliveOrDeferred(KeepAliveOrDeferred&& other) noexcept;

  ~KeepAliveOrDeferred();

  KeepAliveOrDeferred& operator=(KeepAliveOrDeferred&& other) noexcept;

  DeferredExecutor* getDeferredExecutor() const noexcept;

  Executor* getKeepAliveExecutor() const noexcept;

  KA stealKeepAlive() && noexcept;

  DW stealDeferred() && noexcept;

  bool isDeferred() const noexcept;

  bool isKeepAlive() const noexcept;

  KeepAliveOrDeferred copy() const;

  explicit operator bool() const noexcept;

 private:
  friend class DeferredExecutor;
  // 枚举被折叠类型
  enum class State { Deferred, KeepAlive } state_;
  union {
    DW deferred_;
    KA keepAlive_;
  };
};

// 判断类型
inline bool KeepAliveOrDeferred::isDeferred() const noexcept {
  return state_ == State::Deferred;
}

inline bool KeepAliveOrDeferred::isKeepAlive() const noexcept {
  return state_ == State::KeepAlive;
}
```

这里我们主要使用`Executor::KeepAlive<>`类型，可不考虑`DeferredExecutor`。

由于`KeepAlive`存再隐式构造函数：

```c++
/* implicit */ KeepAlive(ExecutorT* executor) {
  *this = getKeepAliveToken(executor);
}

template <typename ExecutorT>
static KeepAlive<ExecutorT> getKeepAliveToken(ExecutorT* executor) {
  static_assert(
    std::is_base_of<Executor, ExecutorT>::value,
    "getKeepAliveToken only works for folly::Executor implementations.");
  if (!executor) {
    return {};
  }
  folly::Executor* executorPtr = executor;
  if (executorPtr->keepAliveAcquire()) {
    return makeKeepAlive<ExecutorT>(executor);
  }
  return makeKeepAliveDummy<ExecutorT>(executor);
}
```

因此对于使用的时候，如果一个函数参数要求是`KeepAlive`，但是传递的是一个`Executor`的指针时，默认都会转成`KeepAlive`。

而`KeepAliveOrDeferred`中又存再

```c++
/* implicit */ KeepAliveOrDeferred(KA ka) noexcept;
```

以`KeepAlive`作为参数的的隐式构造函数，因此当使用`Executor`的指针作为参数时，往往使用的是`KeepAliveOrDeferred`中的`KeepAlive`模式。这也是我前文所述主要使用的是`KeepAlive`的原因，后面可以看到在`feature`中参数均为`KeepAlive`，使用时经常会传递线程池指针，这种情况下，都是使用的上述内容。

### InterruptHandler&InterruptHandlerImpl

这两个类提供了异常处理机制，都比较简单，这里不做详细介绍。主要包括三部分，引用计数，异常处理函数，异常类（folly::exception_wrapper），其中folly::exception_wrapper相对复杂，这里不展开介绍。

```
class InterruptHandler {
 public:
  virtual ~InterruptHandler();

  virtual void handle(const folly::exception_wrapper& ew) const = 0;

  void acquire();
  void release();

 private:
  std::atomic<ssize_t> refCount_{1};
};

template <class F>
class InterruptHandlerImpl final : public InterruptHandler {
 public:
  template <typename R>
  explicit InterruptHandlerImpl(R&& f) noexcept(
      noexcept(F(static_cast<R&&>(f))))
      : f_(static_cast<R&&>(f)) {}

  void handle(const folly::exception_wrapper& ew) const override { f_(ew); }

 private:
  F f_;
};
```



### CoreBase

`CoreBase`是核心类，其维护了之前介绍了数据流转和FSM。

```c++
class CoreBase {
 protected:
  // context用于传递请求相关信息，主要是架构使用，用户可不关心，future中会详细介绍
  using Context = std::shared_ptr<RequestContext>;
  // Callback函数包含线程池和异常，线程池主要用于链式传递
  using Callback = folly::Function<void(
      CoreBase&, Executor::KeepAlive<>&&, exception_wrapper* ew)>;

 public:
  // CoreBase不可拷贝和移动
  // not copyable
  CoreBase(CoreBase const&) = delete;
  CoreBase& operator=(CoreBase const&) = delete;

  // not movable (see comment in the implementation of Future::then)
  CoreBase(CoreBase&&) noexcept = delete;
  CoreBase& operator=(CoreBase&&) = delete;

  /// May call from any thread
  // 判断是否设置了callback，消费者和生产者线程均可能调用
  bool hasCallback() const noexcept {
    constexpr auto allowed = State::OnlyCallback |
        State::OnlyCallbackAllowInline | State::Done | State::Empty;
    auto const state = state_.load(std::memory_order_acquire);
    return State() != (state & allowed);
  }

  /// May call from any thread
  ///
  /// True if state is OnlyResult or Done.
  ///
  /// Identical to `this->ready()`
  // 判断是否已经有result了，所有线程均有可能调用，和this->ready()一致。
  // 设置了result和done均返回成功
  bool hasResult() const noexcept;

  /// May call from any thread
  ///
  /// True if state is OnlyResult or Done.
  ///
  /// Identical to `this->hasResult()`
  bool ready() const noexcept { return hasResult(); }

  // // 消费者线程调用，如果引用计数为0，执行析构函数
  /// Called by a destructing Future (in the consumer thread, by definition).
  /// Calls `delete this` if there are no more references to `this`
  /// (including if `detachPromise()` is called previously or concurrently).
  void detachFuture() noexcept { detachOne(); }

  // 生产者线程调用，如果引用计数为0，执行析构函数
  /// Called by a destructing Promise (in the producer thread, by definition).
  /// Calls `delete this` if there are no more references to `this`
  /// (including if `detachFuture()` is called previously or concurrently).
  void detachPromise() noexcept {
    DCHECK(hasResult());
    detachOne();
  }
  
  // 消费者线程调用，调用要么在设置一个callback之前，要么在callback已经执行完成后。
  // 不能和任意可能导致callback被执行的操作并发
  // 这里也是上面说的，参数是`KeepAliveOrDeferred`，目前看到的基本都是KeepAlive
  /// Call only from consumer thread, either before attaching a callback or
  /// after the callback has already been invoked, but not concurrently with
  /// anything which might trigger invocation of the callback.
  void setExecutor(KeepAliveOrDeferred&& x) {
    DCHECK(
        state_ != State::OnlyCallback &&
        state_ != State::OnlyCallbackAllowInline);
    executor_ = std::move(x);
  }
  
  // 获取线程池
  Executor* getExecutor() const;

  DeferredExecutor* getDeferredExecutor() const;

  DeferredWrapper stealDeferredExecutor();

  /// 消费者抛出一个异常，如果已经有异常处理函数了，则立即执行异常处理函数
  /// Call only from consumer thread
  ///
  /// Eventual effect is to pass `e` to the Promise's interrupt handler, either
  /// synchronously within this call or asynchronously within
  /// `setInterruptHandler()`, depending on which happens first (a coin-toss if
  /// the two calls are racing).
  ///
  /// Has no effect if it was called previously.
  /// Has no effect if State is OnlyResult or Done.
  void raise(exception_wrapper e);

  // 新的corebase，使用旧的interrupt handler来初始化自己的interrupt handler
  /// Copy the interrupt handler from another core. This should be done only
  /// when initializing a new core:
  ///
  /// - interruptHandler_ must be nullptr
  /// - interruptLock_ is not acquired.
  void initCopyInterruptHandlerFrom(const CoreBase& other);
   
  // 生产者线程调用，设置一个异常处理函数，如果有异常了，则立即执行异常处理函数
  /// Call only from producer thread
  ///
  /// May invoke `fn()` (passing the interrupt) synchronously within this call
  /// (if `raise()` preceded or perhaps if `raise()` is called concurrently).
  ///
  /// Has no effect if State is OnlyResult or Done.
  ///
  /// Note: `fn()` must not touch resources that are destroyed immediately after
  ///   `setResult()` is called. Reason: it is possible for `fn()` to get called
  ///   asynchronously (in the consumer thread) after the producer thread calls
  ///   `setResult()`.
  template <typename F>
  void setInterruptHandler(F&& fn) {
    using handler_type = InterruptHandlerImpl<std::decay_t<F>>;
    if (hasResult()) {
      return;
    }
    handler_type* handler = nullptr;
    auto interrupt = interrupt_.load(std::memory_order_acquire);
    switch (interrupt & InterruptMask) {
      case InterruptInitial: { // store the handler
        assert(!interrupt);
        handler = new handler_type(static_cast<F&&>(fn));
        auto exchanged = folly::atomic_compare_exchange_strong_explicit(
            &interrupt_,
            &interrupt,
            reinterpret_cast<uintptr_t>(handler) | InterruptHasHandler,
            std::memory_order_release,
            std::memory_order_acquire);
        if (exchanged) {
          return;
        }
        // lost the race!
        if (interrupt & InterruptHasHandler) {
          terminate_with<std::logic_error>("set-interrupt-handler race");
        }
        assert(interrupt & InterruptHasObject);
        FOLLY_FALLTHROUGH;
      }
      case InterruptHasObject: { // invoke over the stored object
        auto exchanged = interrupt_.compare_exchange_strong(
            interrupt, InterruptTerminal, std::memory_order_relaxed);
        if (!exchanged) {
          terminate_with<std::logic_error>("set-interrupt-handler race");
        }
        auto pointer = interrupt & ~InterruptMask;
        auto object = reinterpret_cast<exception_wrapper*>(pointer);
        if (handler) {
          handler->handle(*object);
          delete handler;
        } else {
          // mimic constructing and invoking a handler: 1 copy; non-const invoke
          auto fn_ = static_cast<F&&>(fn);
          fn_(as_const(*object));
        }
        delete object;
        return;
      }
      case InterruptHasHandler: // fail all calls after the first
        terminate_with<std::logic_error>("set-interrupt-handler duplicate");
      case InterruptTerminal: // fail all calls after the first
        terminate_with<std::logic_error>("set-interrupt-handler after done");
    }
  }

 protected:
  CoreBase(State state, unsigned char attached);

  virtual ~CoreBase();

  // Helper class that stores a pointer to the `Core` object and calls
  // `derefCallback` and `detachOne` in the destructor.
  class CoreAndCallbackReference;

  // interrupt_ is an atomic acyclic finite state machine with guarded state
  // which takes the form of either a pointer to a copy of the object passed to
  // raise or a pointer to a copy of the handler passed to setInterruptHandler
  //
  // the object and the handler values are both at least pointer-aligned so they
  // leave the bottom 2 bits free on all supported platforms; these bits are
  // stolen for the state machine
  enum : uintptr_t {
    InterruptMask = 0x3u,
  };
  enum InterruptState : uintptr_t {
    InterruptInitial = 0x0u,
    InterruptHasHandler = 0x1u,
    InterruptHasObject = 0x2u,
    InterruptTerminal = 0x3u,
  };

  /*
  如果状态为start，则变更到OnlyCallback | OnlyCallbackAllowInline
  如果状态为OnlyResult，则立即执行doCallback，变更到Done
  其中allowInline控制是否立即执行callback，及切换状态为OnlyCallbackAllowInline还是OnlyCallback
  */
  void setCallback_(
      Callback&& callback,
      std::shared_ptr<folly::RequestContext>&& context,
      futures::detail::InlineContinuation allowInline);

  /*
  如果是OnlyCallback || OnlyCallbackAllowInline，则变更为done 立即执行doCallback
  否则仅变更为OnlyResult
  */
  void setResult_(Executor::KeepAlive<>&& completingKA);
  void setProxy_(CoreBase* proxy);
  /*
  将executor_替换为一个新的空的executor_（bool判断为false）
  如果此前未设置executor_，则立即执行callback_，传参this、completingKA、nullptr
  否则：
      如果状态不为OnlyCallbackAllowInline，则completingKA赋值为新创建的一个KeepAlive
      如果当前executor_为DeferredExecutor，使用completingKA和封装的callback_执行addFrom
      如果当前executor_为KeepAliveExecutor，同样判断executor_和completingKA是否一致，如果一致，则立即执行封装的callback_，
    否则将封装的callback_添加进入当前的executor_
  */
  void doCallback(Executor::KeepAlive<>&& completingKA, State priorState);
  void proxyCallback(State priorState);
  // 执行core的析构，将attached_ -1， 如果将attached_ == 0时，析构this
  void detachOne() noexcept;
  
  /* 
  执行context_&callback_的析构，callbackReferences_ -1， 
  如果callbackReferences_==0，执行context_&callback_析构函数
  */
  void derefCallback() noexcept;

  union {
    Callback callback_;
  };
  // 状态
  std::atomic<State> state_;
  // 自生引用计数
  std::atomic<unsigned char> attached_;
  // callback引用计数
  std::atomic<unsigned char> callbackReferences_{0};
  KeepAliveOrDeferred executor_;
  union {
    Context context_;
  };
  std::atomic<uintptr_t> interrupt_{}; // see InterruptMask, InterruptState
  CoreBase* proxy_;
};
```

`setExecutor`使用参数是`KeepAliveOrDeferred`，也就是在上面介绍的，实际往往都是`keepalive`。

大部分情况下使用`corebase`时，会先提前设置一个`Executor`和设置`callback`，当设置`Result`时往往是链式请求中的上一个`future`，这时会将上一个执行的线程池传递过来，如果之前已经设置的线程池和传递的线程池不一致，则使用旧版的线程池（优先用户设置的，传过来的是架构透传的，后面讲到`future`的then会详细介绍）。

对于执行方式有两种类型，一个是立即执行，另一种是放到线程池中等待调度执行。当`setResult_`（如果已经设置了`callback`，将调用`doCallback`）时，之前未设置线程池，则认为是在原线程池中立即执行，所以直接执行`callback`。当此前已经设置了线程池时，则可以通过`state`是`OnlyCallbackAllowInline`还是`OnlyCallback`，对于前者来说，表示立即执行`callback`，对后者来说，则表示将`callback`添加到线程池中，等待线程池调度。

<a id="RequestContext">RequestContext</a>：其中`context_`是线程维度数据，其实现为一个线程维度单例（使用static thread local），用于传递一些线程数据。一个比较常见使用场景是，对于一个线程执行的任务，向该结构内增加数据，当出问题，确定是哪个请求导致的。

有了上面介绍，我们来详细看一下`doCallback`函数实现<a id="doCallback">doCallback</a>"：

```c++
// May be called at most once.
void CoreBase::doCallback(
    Executor::KeepAlive<>&& completingKA, State priorState) {
  DCHECK(state_ == State::Done);

  auto executor = std::exchange(executor_, KeepAliveOrDeferred{});

  // Customise inline behaviour
  // If addCompletingKA is non-null, then we are allowing inline execution
  auto doAdd = [](Executor::KeepAlive<>&& addCompletingKA,
                  KeepAliveOrDeferred&& currentExecutor,
                  auto&& keepAliveFunc) mutable {
    if (auto deferredExecutorPtr = currentExecutor.getDeferredExecutor()) {
      deferredExecutorPtr->addFrom(
          std::move(addCompletingKA), std::move(keepAliveFunc));
    } else {
      // If executors match call inline
      auto currentKeepAlive = std::move(currentExecutor).stealKeepAlive();
      if (addCompletingKA.get() == currentKeepAlive.get()) {
        keepAliveFunc(std::move(currentKeepAlive));
      } else {
        std::move(currentKeepAlive).add(std::move(keepAliveFunc));
      }
    }
  };

  if (executor) {
    // If we are not allowing inline, clear the completing KA to disallow
    if (!(priorState == State::OnlyCallbackAllowInline)) {
      completingKA = Executor::KeepAlive<>{};
    }
    exception_wrapper ew;
    // We need to reset `callback_` after it was executed (which can happen
    // through the executor or, if `Executor::add` throws, below). The
    // executor might discard the function without executing it (now or
    // later), in which case `callback_` also needs to be reset.
    // The `Core` has to be kept alive throughout that time, too. Hence we
    // increment `attached_` and `callbackReferences_` by two, and construct
    // exactly two `CoreAndCallbackReference` objects, which call
    // `derefCallback` and `detachOne` in their destructor. One will guard
    // this scope, the other one will guard the lambda passed to the executor.
    attached_.fetch_add(2, std::memory_order_relaxed);
    callbackReferences_.fetch_add(2, std::memory_order_relaxed);
    CoreAndCallbackReference guard_local_scope(this);
    CoreAndCallbackReference guard_lambda(this);
    try {
      doAdd(
          std::move(completingKA),
          std::move(executor),
          [core_ref =
               std::move(guard_lambda)](Executor::KeepAlive<>&& ka) mutable {
            auto cr = std::move(core_ref);
            CoreBase* const core = cr.getCore();
            RequestContextScopeGuard rctx(std::move(core->context_));
            core->callback_(*core, std::move(ka), nullptr);
          });
    } catch (...) {
      ew = exception_wrapper(std::current_exception());
    }
    if (ew) {
      RequestContextScopeGuard rctx(std::move(context_));
      callback_(*this, Executor::KeepAlive<>{}, &ew);
    }
  } else {
    attached_.fetch_add(1, std::memory_order_relaxed);
    SCOPE_EXIT {
      context_.~Context();
      callback_.~Callback();
      detachOne();
    };
    RequestContextScopeGuard rctx(std::move(context_));
    callback_(*this, std::move(completingKA), nullptr);
  }
}
```

传参中`completingKA`有两种情况：

1. 如果是先设置的result，再设置的callback，则是新建的一个新的。
2. 如果是先设置的callback，再设置的result，则大部分情况是上一个task任务执行时所使用的线程池。

其中doAdd是实际执行callback的函数，参数中`addCompletingKA`是新传入的线程池，`currentExecutor`是旧版线程池，`keepAliveFunc`是又进过一层封装的callback。

```c++
	auto doAdd = [](Executor::KeepAlive<>&& addCompletingKA,
                  KeepAliveOrDeferred&& currentExecutor,
                  auto&& keepAliveFunc) mutable {
    if (auto deferredExecutorPtr = currentExecutor.getDeferredExecutor()) {
      deferredExecutorPtr->addFrom(
          std::move(addCompletingKA), std::move(keepAliveFunc));
    } else {
      // If executors match call inline
      auto currentKeepAlive = std::move(currentExecutor).stealKeepAlive();
      if (addCompletingKA.get() == currentKeepAlive.get()) {
        keepAliveFunc(std::move(currentKeepAlive));
      } else {
        std::move(currentKeepAlive).add(std::move(keepAliveFunc));
      }
    }
  };
```

首先判断原线程池是否为defer线程池，这里一般不是，暂时不考虑。

对于均为keepalive线程池来说，判断新旧线程池是否一致，如果一致，则立即执行callback，否则将callback添加到线程池中。

如果之前存再线程池，执行逻辑如下：

```c++
		if (!(priorState == State::OnlyCallbackAllowInline)) {
      completingKA = Executor::KeepAlive<>{};
    }
    exception_wrapper ew;
    // We need to reset `callback_` after it was executed (which can happen
    // through the executor or, if `Executor::add` throws, below). The
    // executor might discard the function without executing it (now or
    // later), in which case `callback_` also needs to be reset.
    // The `Core` has to be kept alive throughout that time, too. Hence we
    // increment `attached_` and `callbackReferences_` by two, and construct
    // exactly two `CoreAndCallbackReference` objects, which call
    // `derefCallback` and `detachOne` in their destructor. One will guard
    // this scope, the other one will guard the lambda passed to the executor.
    attached_.fetch_add(2, std::memory_order_relaxed);
    callbackReferences_.fetch_add(2, std::memory_order_relaxed);
    CoreAndCallbackReference guard_local_scope(this);
    CoreAndCallbackReference guard_lambda(this);
    try {
      doAdd(
          std::move(completingKA),
          std::move(executor),
          [core_ref =
               std::move(guard_lambda)](Executor::KeepAlive<>&& ka) mutable {
            auto cr = std::move(core_ref);
            CoreBase* const core = cr.getCore();
            RequestContextScopeGuard rctx(std::move(core->context_));
            core->callback_(*core, std::move(ka), nullptr);
          });
    } catch (...) {
      ew = exception_wrapper(std::current_exception());
    }
    if (ew) {
      RequestContextScopeGuard rctx(std::move(context_));
      callback_(*this, Executor::KeepAlive<>{}, &ew);
    }
```

<a id="执行方式">执行方式</a>：如果不允许立即执行函数（即非OnlyCallbackAllowInline），则让新线程池变成一个空的线程池，这样保证不会和之前线程池一致，在doAdd时就不会立即执行callback了。

在介绍下面代码之前，先看一下`CoreAndCallbackReference`实现：

```c++
class CoreBase::CoreAndCallbackReference {
 public:
  explicit CoreAndCallbackReference(CoreBase* core) noexcept : core_(core) {}

  ~CoreAndCallbackReference() noexcept { detach(); }

  CoreAndCallbackReference(CoreAndCallbackReference const& o) = delete;
  CoreAndCallbackReference& operator=(CoreAndCallbackReference const& o) =
      delete;
  CoreAndCallbackReference& operator=(CoreAndCallbackReference&&) = delete;

  CoreAndCallbackReference(CoreAndCallbackReference&& o) noexcept
      : core_(std::exchange(o.core_, nullptr)) {}

  CoreBase* getCore() const noexcept { return core_; }

 private:
  void detach() noexcept {
    if (core_) {
      core_->derefCallback();
      core_->detachOne();
    }
  }

  CoreBase* core_{nullptr};
};
```

其实现相当简单，就是存储core，并且负责管理coar自身生命周期以及callback和context生命周期。利用变量作用域结束后调用析构函数保证执行core的`derefCallback`和`detachOne()`。

下面再来看接下来的四行代码：

```c++
		// We need to reset `callback_` after it was executed (which can happen
    // through the executor or, if `Executor::add` throws, below). The
    // executor might discard the function without executing it (now or
    // later), in which case `callback_` also needs to be reset.
    // The `Core` has to be kept alive throughout that time, too. Hence we
    // increment `attached_` and `callbackReferences_` by two, and construct
    // exactly two `CoreAndCallbackReference` objects, which call
    // `derefCallback` and `detachOne` in their destructor. One will guard
    // this scope, the other one will guard the lambda passed to the executor.
    attached_.fetch_add(2, std::memory_order_relaxed);
    callbackReferences_.fetch_add(2, std::memory_order_relaxed);
    CoreAndCallbackReference guard_local_scope(this);
    CoreAndCallbackReference guard_lambda(this);
```

其中注释给了详细介绍。我们要在`callback_`执行结束时reset（这里应该是指要执行析构函数），`doAdd`执行后有可能会丢弃`callback_`，即使这种情况下，我们依然需要reset callback，因此需要有两个`CoreAndCallbackReference`来维护`callback_`的生命周期。而执行`keepAliveFunc`需要core，因此core也不能在执行doAdd（有可能只是将keepAliveFunc加到了线程池中而未执行）完成后就被析构，因此对core的生命周期监控也需要两个`CoreAndCallbackReference`，因此这里将core和callback的引用计数都先加2，使用两个`CoreAndCallbackReference`来对core和callback进行生命周期监控。

这里解释了此前说的，在执行`callback`执行时会自己维护FSM，保证自己不被析构（在执行callback时，future和promise可能都已经被析构了，当callback执行完成后core被析构）。

<a id="异常处理">异常处理</a>：如果执行doAdd抛出异常，则使用该异常执行callback，因此callback应该有对异常处理的能力。目前异常处理不需要通过函数中判断参数是否存再异常，这一步由future架构实现了，目前实现异常处理可以通过`thenError`来实现，这部分在future中将详细介绍。

当此前未设置线程池时，其处理相对简单，先注册执行完成的析构方法，用于析构`callback_`，而后立即执行callback。

这里关于`core`的生命周期还有两个重要函数：

```c++
	/// Called by a destructing Future (in the consumer thread, by definition).
  /// Calls `delete this` if there are no more references to `this`
  /// (including if `detachPromise()` is called previously or concurrently).
  void detachFuture() noexcept { detachOne(); }

  /// Called by a destructing Promise (in the producer thread, by definition).
  /// Calls `delete this` if there are no more references to `this`
  /// (including if `detachFuture()` is called previously or concurrently).
  void detachPromise() noexcept {
    DCHECK(hasResult());
    detachOne();
  }
  
  void CoreBase::detachOne() noexcept {
    auto a = attached_.fetch_sub(1, std::memory_order_acq_rel);
    assert(a >= 1);
    if (a == 1) {
      delete this;
    }
  }
```

上面说过，维护FSM有两个结构，future和promise，`detachFuture`和`detachPromise`分别对应于两个结构的析构函数。由于`future`析构时，可能什么也未操作（只设置了callback），因此可以直接尝试执行析构即可。而析构`promise`时，正常来说已经设置了`result`(上一层的promise设置的这一层的result)，因此这里先做了一个检查，如果未设置，打日志。

后面可以看到，正常创建一个空的core时，其引用计数就是2，表示会有两个结构持有，分别执行析构函数，最终来析构core。

### ResultHolder

`ResultHolder`用处存储`result`。其定义如下：

```c++
template <typename T>
class ResultHolder {
 protected:
  ResultHolder() {}
  ~ResultHolder() {}
  // Using a separate base class allows us to control the placement of result_,
  // making sure that it's in the same cache line as the vtable pointer and the
  // callback_ (assuming it's small enough).
  union {
    Try<T> result_;
  };
};
```

其中`Try`存储一个`T`类型原始，或者一个`expection`或者未空，其中有枚举值标记其类型。

### core

core继承了corebase和resultholder，组成了FSM的原始。其定义如下：

```c++
template <typename T>
class Core final : private ResultHolder<T>, public CoreBase {
  static_assert(
      !std::is_void<T>::value,
      "void futures are not supported. Use Unit instead.");

 public:
  using Result = Try<T>;

  /// State will be Start
  // 创建空的core，状态为start，core自生计数为2
  static Core* make() { return new Core(); }

  /// State will be OnlyResult
  /// Result held will be move-constructed from `t`
  // 创建含有result的core，状态为OnlyResult，core自生计数为1
  static Core* make(Try<T>&& t) { return new Core(std::move(t)); }

  /// State will be OnlyResult
  /// Result held will be the `T` constructed from forwarded `args`
  // 创建含有result的core，状态为OnlyResult，core自生计数为1
  template <typename... Args>
  static Core<T>* make(in_place_t, Args&&... args) {
    return new Core<T>(in_place, static_cast<Args&&>(args)...);
  }

  /// Call only from consumer thread (since the consumer thread can modify the
  ///   referenced Try object; see non-const overloads of `future.result()`,
  ///   etc., and certain Future-provided callbacks which move-out the result).
  ///
  /// Unconditionally returns a reference to the result.
  ///
  /// State dependent preconditions:
  ///
  /// - Start, OnlyCallback or OnlyCallbackAllowInline: Never safe - do not
  /// call. (Access in those states
  ///   would be undefined behavior since the producer thread can, in those
  ///   states, asynchronously set the referenced Try object.)
  /// - OnlyResult: Always safe. (Though the consumer thread should not use the
  ///   returned reference after it attaches a callback unless it knows that
  ///   the callback does not move-out the referenced result.)
  /// - Done: Safe but sometimes unusable. (Always returns a valid reference,
  ///   but the referenced result may or may not have been modified, including
  ///   possibly moved-out, depending on what the callback did; some but not
  ///   all callbacks modify (possibly move-out) the result.)
  // 消费者线程调用，获取结果
  Try<T>& getTry() {
    DCHECK(hasResult());
    auto core = this;
    while (core->state_.load(std::memory_order_relaxed) == State::Proxy) {
      core = static_cast<Core*>(core->proxy_);
    }
    return core->result_;
  }
  Try<T> const& getTry() const {
    DCHECK(hasResult());
    auto core = this;
    while (core->state_.load(std::memory_order_relaxed) == State::Proxy) {
      core = static_cast<Core const*>(core->proxy_);
    }
    return core->result_;
  }

  /// Call only from consumer thread.
  /// Call only once - else undefined behavior.
  ///
  /// See FSM graph for allowed transitions.
  ///
  /// If it transitions to Done, synchronously initiates a call to the callback,
  /// and might also synchronously execute that callback (e.g., if there is no
  /// executor or if the executor is inline).
  template <class F>
  void setCallback(
      F&& func,
      std::shared_ptr<folly::RequestContext>&& context,
      futures::detail::InlineContinuation allowInline) {
    Callback callback = [func = static_cast<F&&>(func)](
                            CoreBase& coreBase,
                            Executor::KeepAlive<>&& ka,
                            exception_wrapper* ew) mutable {
      auto& core = static_cast<Core&>(coreBase);
      if (ew != nullptr) {
        core.result_ = Try<T>{std::move(*ew)};
      }
      func(std::move(ka), std::move(core.result_));
    };

    setCallback_(std::move(callback), std::move(context), allowInline);
  }

  /// Call only from producer thread.
  /// Call only once - else undefined behavior.
  ///
  /// See FSM graph for allowed transitions.
  ///
  /// If it transitions to Done, synchronously initiates a call to the callback,
  /// and might also synchronously execute that callback (e.g., if there is no
  /// executor or if the executor is inline).
  // 生产者线程调用，设置结果
  void setResult(Try<T>&& t) {
    setResult(Executor::KeepAlive<>{}, std::move(t));
  }

  /// Call only from producer thread.
  /// Call only once - else undefined behavior.
  ///
  /// See FSM graph for allowed transitions.
  ///
  /// If it transitions to Done, synchronously initiates a call to the callback,
  /// and might also synchronously execute that callback (e.g., if there is no
  /// executor, if the executor is inline or if completingKA represents the
  /// same executor as does executor_).
  // 生产者线程调用，设置结果，其中completingKA用于控制链式执行使用的线程池
  void setResult(Executor::KeepAlive<>&& completingKA, Try<T>&& t) {
    ::new (&this->result_) Result(std::move(t));
    setResult_(std::move(completingKA));
  }

  /// Call only from producer thread.
  /// Call only once - else undefined behavior.
  ///
  /// See FSM graph for allowed transitions.
  ///
  /// This can not be called concurrently with setResult().
  void setProxy(Core* proxy) {
    // NOTE: We could just expose this from the base, but that accepts any
    // CoreBase, while we want to enforce the same Core<T> in the interface.
    setProxy_(proxy);
  }

 private:
  Core() : CoreBase(State::Start, 2) {}

  explicit Core(Try<T>&& t) : CoreBase(State::OnlyResult, 1) {
    new (&this->result_) Result(std::move(t));
  }

  template <typename... Args>
  explicit Core(in_place_t, Args&&... args) noexcept(
      std::is_nothrow_constructible<T, Args&&...>::value)
      : CoreBase(State::OnlyResult, 1) {
    new (&this->result_) Result(in_place, static_cast<Args&&>(args)...);
  }
  
  // 析构函数只负责处理result_，并不析构其他内容，包括callback和自生的引用计数
  ~Core() override {
    DCHECK(attached_ == 0);
    auto state = state_.load(std::memory_order_relaxed);
    switch (state) {
      case State::OnlyResult:
        FOLLY_FALLTHROUGH;

      case State::Done:
        this->result_.~Result();
        break;

      case State::Proxy:
        proxy_->detachFuture();
        break;

      case State::Empty:
        break;

      case State::Start:
      case State::OnlyCallback:
      case State::OnlyCallbackAllowInline:
      default:
        terminate_with<std::logic_error>("~Core unexpected state");
    }
  }
};
```

`core`相较`corebase`增加内容不多，主要看一下其`setCallback`，其主要对`fun`进行了一层封装，将对异常的处理进行了封装：

```c++
	template <class F>
  void setCallback(
      F&& func,
      std::shared_ptr<folly::RequestContext>&& context,
      futures::detail::InlineContinuation allowInline) {
    Callback callback = [func = static_cast<F&&>(func)](
                            CoreBase& coreBase,
                            Executor::KeepAlive<>&& ka,
                            exception_wrapper* ew) mutable {
      auto& core = static_cast<Core&>(coreBase);
      if (ew != nullptr) {
        core.result_ = Try<T>{std::move(*ew)};
      }
      func(std::move(ka), std::move(core.result_));
    };

    setCallback_(std::move(callback), std::move(context), allowInline);
  }
```

当传递的异常不为空时，表示出现了异常，这时将会把异常添加到result中，此前说过[异常处理](#异常处理)由folly架构完成，这里是其中实现的一部分。更多的部分则在future中介绍。这里可以回头看一下`doCallback`在遇到异常时的处理，会将异常作为参数传递到callback中执行。

### 小结

至此基本介绍完成了core的基本信息，对folly的异步框架的FSM进行了深入的了解，同时对future的链式执行有了初步认识。接下来会更加细致的介绍folly链式执行的实现原理及DAG的实现。

## future&promise

由于future&promise通过了大量接口，全部介绍比较啰嗦（很多很简单的接口），这里只介绍一些核心接口。

如上文所说folly&promise沟通维护一个core结构，共同维护FSM。promise一般会由生产者线程持有，负载生产result，future有消费者线程持有（用户线程），负责添加callback并且控制执行流程，以及最终结果获取。

promise持有core（成员变量），并且可以通过promise获取到future。

### 正常异步执行流程为

异步执行流程如下

1. 创建一个promise。
2. 从promise中获取一个future，并将future给消费者线程。
3. 用户向future中添加callback（同时可设置executor）。添加callback后会新建一个promise，并通过新的promise获取一个新的future。其中promise自生被生产者线程持有（上一层的callback持有，future的then将详细介绍）。future被返回给消费者线程。
4. 如果返回的future也设置了callback，即链式设置callback，则继续执行第三步，直到不再添加callback为止。
5. 用户设置最最初的promise的result，开启链式执行（这一步也可能是在第一个执行，这时创建的一般是future）。
6. 链式执行中，当上一层执行了callback后，设置当前层promise的result，执行当前层级的callback，设置下一层的result。析构当前的promise。
7. 如果没有下一层了，则用户通过future获取到结构。如果依然有下一层，则当前层的future已经被析构（构建完成多次执行链就已经被析构了），重复执行第七步。
8. 通过最后一个返回的future，调用阻塞方法，等待所有callback执行完成（或出错）。

对于一个简单代码实例如下：

```c++
#include <folly/futures/Future.h>
#include <folly/executors/ThreadedExecutor.h>
using namespace folly;
using namespace std;

void foo(int x) {
  // do something with x
  cout << "foo(" << x << ")" << endl;
}

void foo1(int x) {
  // do something with x
  cout << "foo1(" << x << ")" << endl;
}

void foo2(int x) {
  // do something with x
  cout << "foo2(" << x << ")" << endl;
}
// ...
  folly::ThreadedExecutor executor;
  cout << "making Promise" << endl;
  Promise<int> p;
  Future<int> f = p.getSemiFuture().via(&executor);
  auto f2 = move(f)
    					.thenValue(foo).via(&executor)
    					.thenValue(foo1).via(&executor)
    					.thenValue(foo2).via(&executor)
    
  cout << "Future chain made" << endl;

// ... now perhaps in another event callback

  cout << "fulfilling Promise" << endl;
  p.setValue(42);
  move(f2).get();
  cout << "Promise fulfilled" << endl;
```

下面我们就按照上面的步骤，进行介绍。

### 创建promise

一般创建promise使用默认构造函数，其定义如下：

```c++
template <class T>
Promise<T>::Promise() : retrieved_(false), core_(Core::make()) {}
```

其中retrieved由于记录是否已经从当前的promise中获取到了future，该参数用于析构时判断。core_是core的引用。其中使用`make()`创建的core将有两个引用计数。

由于retrieved_用于析构，我们先来看一下Promise的析构函数：

```c++
template <class T>
Promise<T>::~Promise() {
  detach();
}

template <class T>
void Promise<T>::detach() {
  if (core_) {
    if (!retrieved_) {
      core_->detachFuture();
    }
    futures::detail::coreDetachPromiseMaybeWithResult(*core_);
    core_ = nullptr;
  }
}

template <typename T>
void coreDetachPromiseMaybeWithResult(Core<T>& core) {
  if (!core.hasResult()) {
    core.setResult(Try<T>(exception_wrapper(BrokenPromise(pretty_name<T>()))));
  }
  core.detachPromise();
}
```

这里，首先会判断是否存再core，之后，如果retrieved_未false，及还未从promise获取到future，则会执行对future的析构，否则仅执行promise析构（之间有一个处理异常的处理）。

这里主要作用是保证core被完整析构，如果还未创建future在，则future的引用计数也有promise释放，如果已经创建了future，则future的引用计数则有future自生释放。

这里我们再看一下future的析构函数：

```c++
template <class T>
FutureBase<T>::~FutureBase() {
  detach();
}

template <class T>
void FutureBase<T>::detach() {
  if (core_) {
    core_->detachFuture();
    core_ = nullptr;
  }
}
```

future继承自FutureBase。这里可以看到，future析构时会释放掉core的future的引用计数。

上面说的启动方式，是callback等待result，还有一种方式是result等待callback。这是创建方式一般是：

```c++
future::makeFuture(T);
future::makeFeature();
```

其中`makeFeature()`等价调用`makeFeature(Unit{})`，`Unit`可以理解为void：

```c++
/// In functional programming, the degenerate case is often called "unit". In
/// C++, "void" is often the best analogue. However, because of the syntactic
/// special-casing required for void, it is frequently a liability for template
/// metaprogramming. So, instead of writing specializations to handle cases like
/// SomeContainer<void>, a library author may instead rule that out and simply
/// have library users use SomeContainer<Unit>. Contained values may be ignored.
/// Much easier.
///
/// "void" is the type that admits of no values at all. It is not possible to
/// construct a value of this type.
/// "unit" is the type that admits of precisely one unique value. It is
/// possible to construct a value of this type, but it is always the same value
/// every time, so it is uninteresting.
struct Unit {
  constexpr bool operator==(const Unit& /*other*/) const { return true; }
  constexpr bool operator!=(const Unit& /*other*/) const { return false; }
};
```

主要用于对void的封装。

其中`makeFuture(t)`实现为：

```c++
template <class T>
Future<T> makeFuture(Try<T> t) {
  return Future<T>(Future<T>::Core::make(std::move(t)));
}
```

这里由于已经设置了result，因此不需要promise，因此引用计数是1。

### 从promise中获取future

promise可以获取future，有两种future，一个是`SemiFuture`，另一个是`future`。其中二者区别主要是在是否设置了`executor`。`SemiFuture`未设置`executor`，`SemiFuture`通过via设置`executor`后就变成了future。设置`executor`含义是指定callback在那个线程池中执行。

对应的两个方法为：

```
template <class T>
SemiFuture<T> Promise<T>::getSemiFuture() {
  if (retrieved_) {
    throw_exception<FutureAlreadyRetrieved>();
  }
  retrieved_ = true;
  return SemiFuture<T>(&getCore());
}

template <class T>
Future<T> Promise<T>::getFuture() {
  // An InlineExecutor approximates the old behaviour of continuations
  // running inine on setting the value of the promise.
  return getSemiFuture().via(&InlineExecutor::instance());
}
```

首先会设置`retrieved_`为true，标识已经生成了future，避免重复析构。其中如果返回future，则会使用folly提供的默认executor。一般我们返回`SemiFuture`，由用户自定义`executor`。

### 添加callback

添加`callback`主要有四个方法，分别是`thenValue`，`thenTry`，`thenValueInline`，`thenTryInline`。其中`Value`和`try`主要区别是`callback`参数不同，value是普通类似，try是对value的封装，同时有可能是还有异常。`Inline`和非`inline`区别时设置callback时是允许立即执行，还是放到线程池中执行，可参考[执行方式](#执行方式)。

其中四个函数实现如下：

```c++
template <class T>
template <typename F>
Future<typename futures::detail::tryCallableResult<T, F>::value_type>
Future<T>::thenTry(F&& func) && {
  auto lambdaFunc = [f = static_cast<F&&>(func)](
                        folly::Executor::KeepAlive<>&&,
                        folly::Try<T>&& t) mutable {
    return static_cast<F&&>(f)(std::move(t));
  };
  using R = futures::detail::tryExecutorCallableResult<T, decltype(lambdaFunc)>;
  return this->thenImplementation(
      std::move(lambdaFunc), R{}, futures::detail::InlineContinuation::forbid);
}

template <class T>
template <typename F>
Future<typename futures::detail::tryCallableResult<T, F>::value_type>
Future<T>::thenTryInline(F&& func) && {
  auto lambdaFunc = [f = static_cast<F&&>(func)](
                        folly::Executor::KeepAlive<>&&,
                        folly::Try<T>&& t) mutable {
    return static_cast<F&&>(f)(std::move(t));
  };
  using R = futures::detail::tryExecutorCallableResult<T, decltype(lambdaFunc)>;
  return this->thenImplementation(
      std::move(lambdaFunc), R{}, futures::detail::InlineContinuation::permit);
}

template <class T>
template <typename F>
Future<typename futures::detail::valueCallableResult<T, F>::value_type>
Future<T>::thenValue(F&& func) && {
  auto lambdaFunc = [f = static_cast<F&&>(func)](
                        Executor::KeepAlive<>&&, folly::Try<T>&& t) mutable {
    return futures::detail::wrapInvoke(std::move(t), static_cast<F&&>(f));
  };
  using R = futures::detail::tryExecutorCallableResult<T, decltype(lambdaFunc)>;
  return this->thenImplementation(
      std::move(lambdaFunc), R{}, futures::detail::InlineContinuation::forbid);
}

template <class T>
template <typename F>
Future<typename futures::detail::valueCallableResult<T, F>::value_type>
Future<T>::thenValueInline(F&& func) && {
  auto lambdaFunc = [f = static_cast<F&&>(func)](
                        Executor::KeepAlive<>&&, folly::Try<T>&& t) mutable {
    return futures::detail::wrapInvoke(std::move(t), static_cast<F&&>(f));
  };
  using R = futures::detail::tryExecutorCallableResult<T, decltype(lambdaFunc)>;
  return this->thenImplementation(
      std::move(lambdaFunc), R{}, futures::detail::InlineContinuation::permit);
}
```

首先可以看到是否有`inline`，仅在调用`thenImplementation`时最后一个参数有`diff`。

对应`thenTry`来说，其封装的`lambdaFunc`直接执行`callback`，而对于`thenValue`来说，则是使用`futures::detail::wrapInvoke`又进行了一层封装。这是因为，在future框架代码执行过程中，数据交换都是通过`try`进行传递的，如果`callback`参数是try，则可以直接使用，如果参数是value，则需要从try中提取出value（如果有的话）。这也导致了异常处理的差异，对于使用`thenTry`来说，需要在处理函数内部判断是否有异常，并对异常进行处理，如果使用`thenValue`，则在出现异常时，不会调用对应的callback，当用户设置了`thenError`时，根据`thenError`匹配的错误类型进行执行对应的callback（这部分后面会详细介绍）。

我们看一下`futures::detail::wrapInvoke`的实现:

```c++
template <typename T, typename F>
auto wrapInvoke(folly::Try<T>&& t, F&& f) {
  auto fn = [&]() {
    return static_cast<F&&>(f)(
        t.template get<
            false,
            typename futures::detail::valueCallableResult<T, F>::FirstArg>());
  };
  using FnResult = decltype(fn());
  using Wrapper = InvokeResultWrapper<FnResult>;
  if (t.hasException()) {
    return Wrapper::wrapException(std::move(t).exception());
  }
  return Wrapper::wrapResult(fn);
}
```

`fn`执行callback函数，参数为try中的value。

首先判断try中是是否有异常，如果有直接返回异常，如果没有，则返回`fn`执行结果。其中返回类型是callback的返回结果类型。可以大致看一下`InvokeResultWrapper`实现：

```c++
template <typename T>
struct InvokeResultWrapperBase {
  template <typename F>
  static T wrapResult(F fn) {
    return T(fn());
  }
  static T wrapException(exception_wrapper&& e) { return T(std::move(e)); }
};
```

对应返回类型为`void`，及callback未设置返回的来说，利用模板的偏特化，返回的类型是`Try<Unit>`。

```c++
template <>
struct InvokeResultWrapper<void> : InvokeResultWrapperBase<Try<Unit>> {
  template <typename F>
  static Try<Unit> wrapResult(F fn) {
    fn();
    return Try<Unit>(unit);
  }
};
```

在上述四个`then`方法中构建的`lambdaFunc`函数中，传递了参数`Executor::KeepAlive<>&&`，但是并未使用，这是由于除了上述方法，还有三个方法：

```
template <typename T>
template <typename R, typename Caller, typename... Args>
Future<typename isFuture<R>::Inner> Future<T>::then(
    R (Caller::*func)(Args...), Caller* instance) && {
  using FirstArg =
      remove_cvref_t<typename futures::detail::ArgType<Args...>::FirstArg>;

  return std::move(*this).thenTry([instance, func](Try<T>&& t) {
    return (instance->*func)(t.template get<isTry<FirstArg>::value, Args>()...);
  });
}

template <class T>
template <typename F>
Future<typename futures::detail::tryExecutorCallableResult<T, F>::value_type>
Future<T>::thenExTry(F&& func) && {
  auto lambdaFunc = [f = static_cast<F&&>(func)](
                        Executor::KeepAlive<>&& ka, folly::Try<T>&& t) mutable {
    // Enforce that executor cannot be null
    DCHECK(ka);
    return static_cast<F&&>(f)(std::move(ka), std::move(t));
  };
  using R = futures::detail::tryExecutorCallableResult<T, decltype(lambdaFunc)>;
  return this->thenImplementation(
      std::move(lambdaFunc), R{}, futures::detail::InlineContinuation::forbid);
}

template <class T>
template <typename F>
Future<typename futures::detail::tryExecutorCallableResult<T, F>::value_type>
Future<T>::thenExTryInline(F&& func) && {
  auto lambdaFunc = [f = static_cast<F&&>(func)](
                        Executor::KeepAlive<>&& ka, folly::Try<T>&& t) mutable {
    // Enforce that executor cannot be null
    DCHECK(ka);
    return static_cast<F&&>(f)(std::move(ka), std::move(t));
  };
  using R = futures::detail::tryExecutorCallableResult<T, decltype(lambdaFunc)>;
  return this->thenImplementation(
      std::move(lambdaFunc), R{}, futures::detail::InlineContinuation::permit);
}
```

其中`thenExTry`和`thenExTryInline`传递的`callback`会将`Executor::KeepAlive<>&&`作为参数传递，这三个方法均不常使用，这里不做过多介绍。

其中`R`是作为`callback`的辅助数据。这里不展开介绍。

对于`thenImplementation`来说，根据`callback`返回值类型不同，有两个重载：

1. 当返回普通类型时，执行callback返回结果时，直接设置下一层`promise`的result。
2. 当返回类型为`future`时，执行完callback，替换下一层的`promise`的core到`callback`返回的future的core，同时将下一层的future中设置的callback传递给这一层返回的future，这样相当于在执行时，对返回的future进行了替换，替换为callback返回的future。

下面针对上面两个类型的callback详细介绍。

#### `callback`返回try

`callback`返回try类型的逻辑相对简单一些，代码如下：

```c++
// Variant: returns a value
// e.g. f.then([](Try<T>&& t){ return t.value(); });
template <class T>
template <typename F, typename R>
typename std::enable_if<!R::ReturnsFuture::value, typename R::Return>::type
FutureBase<T>::thenImplementation(
    F&& func, R, futures::detail::InlineContinuation allowInline) {
  static_assert(R::Arg::ArgsSize::value == 2, "Then must take two arguments");
  typedef typename R::ReturnsFuture::Inner B;

  Promise<B> p;
  // 继承上一层的异常处理
  p.core_->initCopyInterruptHandlerFrom(this->getCore());

  // grab the Future now before we lose our handle on the Promise
  auto sf = p.getSemiFuture();
  // 继承上一层的执行器
  sf.setExecutor(folly::Executor::KeepAlive<>{this->getExecutor()});
  auto f = Future<B>(sf.core_);
  // 复制触发析构，置空
  sf.core_ = nullptr;

  this->setCallback_(
      [state = futures::detail::makeCoreCallbackState(
           std::move(p), static_cast<F&&>(func))](
          Executor::KeepAlive<>&& ka, Try<T>&& t) mutable {
        if (!R::Arg::isTry() && t.hasException()) {
          state.setException(std::move(ka), std::move(t.exception()));
        } else {
          auto propagateKA = ka.copy();
          state.setTry(std::move(propagateKA), makeTryWith([&] {
                         return detail_msvc_15_7_workaround::invoke(
                             R{}, state, std::move(ka), std::move(t));
                       }));
        }
      },
      allowInline);
  return f;
}
```

首先会新建一个`Promise`，作为下一层的FSM，其拷贝本层的异常处理函数。从新建的`promise`获取到一个future作为返回结果。其默认使用的执行器与本层的执行器一致。

`setCallback_`对传递的`callback`又进行了一次封装。其中`futures::detail::makeCoreCallbackState`创建的结构为`CoreCallbackState`，其用于控制callback执行，其实现如下：

```c++
//  Guarantees that the stored functor is destructed before the stored promise
//  may be fulfilled. Assumes the stored functor to be noexcept-destructible.
template <typename T, typename F>
class CoreCallbackState {
  using DF = std::decay_t<F>;

 public:
  CoreCallbackState(Promise<T>&& promise, F&& func) noexcept(
      noexcept(DF(std::declval<F&&>())))
      : func_(static_cast<F&&>(func)), promise_(std::move(promise)) {
    assert(before_barrier());
  }

  CoreCallbackState(CoreCallbackState&& that) noexcept(
      noexcept(DF(std::declval<F&&>()))) {
    if (that.before_barrier()) {
      new (&func_) DF(static_cast<F&&>(that.func_));
      promise_ = that.stealPromise();
    }
  }

  CoreCallbackState& operator=(CoreCallbackState&&) = delete;

  ~CoreCallbackState() {
    if (before_barrier()) {
      stealPromise();
    }
  }

  template <typename... Args>
  auto invoke(Args&&... args) noexcept(
      noexcept(std::declval<F&&>()(std::declval<Args&&>()...))) {
    assert(before_barrier());
    return static_cast<F&&>(func_)(static_cast<Args&&>(args)...);
  }

  template <typename... Args>
  auto tryInvoke(Args&&... args) noexcept {
    return makeTryWith([&] { return invoke(static_cast<Args&&>(args)...); });
  }

  void setTry(Executor::KeepAlive<>&& keepAlive, Try<T>&& t) {
    stealPromise().setTry(std::move(keepAlive), std::move(t));
  }

  void setException(Executor::KeepAlive<>&& keepAlive, exception_wrapper&& ew) {
    setTry(std::move(keepAlive), Try<T>(std::move(ew)));
  }

  Promise<T> stealPromise() noexcept {
    assert(before_barrier());
    func_.~DF();
    return std::move(promise_);
  }

 private:
  bool before_barrier() const noexcept { return !promise_.isFulfilled(); }

  union {
    DF func_;
  };
  Promise<T> promise_{Promise<T>::makeEmpty()};
};
```

其持有一次`func`和一个`promise`，其中`func`是本层的`callback`，而`promise`是下一层的`promise`。

`callback`函数使用move方法拷贝到`CoreCallbackState`中，因此其负责管理`callback`的生命周期。调用`stealPromise()`方法就会析构`callback`。

`Invoke`方法用于实际执行`callback`，`setTry`和`setException`负责将`callback`结果传递到`promise`（下一层的FMS）。当设置的`promise`时，`promise`的`isFulfilled()`方法将会返回ture，通过该方式来控制析构`CoreCallbackState`不会double free `callback`。

下面来分析一下封装的`callback`：

```c++
			[state = futures::detail::makeCoreCallbackState(
           std::move(p), static_cast<F&&>(func))](
          Executor::KeepAlive<>&& ka, Try<T>&& t) mutable {
        if (!R::Arg::isTry() && t.hasException()) {
          state.setException(std::move(ka), std::move(t.exception()));
        } else {
          auto propagateKA = ka.copy();
          state.setTry(std::move(propagateKA), makeTryWith([&] {
                         return detail_msvc_15_7_workaround::invoke(
                             R{}, state, std::move(ka), std::move(t));
                       }));
        }
      }
```

`callback`参数为`Executor::KeepAlive<>`和`Try<T>`，其中前者用于传递执行器，后者是上一层的`promise`执行返回的结果。

<a id="异常传递">异常传递</a>首先判断上一层结果是否方式异常，如果存再异常，则不在执行本层的`callback`，设置下一层结果为异常，这样如果下一层依然是这样的处理逻辑，则会一直往下一层设置结果为异常，直到设置到异常处理的`callback`，或者到最有一层，返回给用于。

如果未出现异常，则执行本层的`callback`：

```c++
template <typename R, typename State, typename T, IfArgsSizeIs<R, 2> = 0>
decltype(auto) invoke(R, State& state, Executor::KeepAlive<>&& ka, Try<T>&& t) {
  using Arg1 = typename R::Arg::ArgList::Tail::FirstArg;
  return state.invoke(
      std::move(ka), std::move(t).template get<R::Arg::isTry(), Arg1>());
}
```

并将结果赋值到下一层的`promise`，这里设置的都是`Try`类型数据，但是由于上一层的封装中(`thenTry`,`thenValue`)会对类型进行一次转换，保证传递的参数是下一层·`callback`需要的参数·。

如果本层`callback`执行结果出现异常，则下一层如果是`then`类型的处理，则就会像上面介绍的一样进行[异常传递](#异常传递)。如果下一层就是异常处理函数(`thenError`)则立即就执行异常处理了（相对于跳过的异常传递的传递部分）。

如果本层`callback`执行结果正常，则会在设置下一层`promise`时，立即开始执行下一层的`callback`，之后依次链式执行。

这里再来看一下`setCallback_`内容，其对传递的`callback`又进行了一层封装(无线套娃了属于是):

```c++
template <class T>
template <class F>
void FutureBase<T>::setCallback_(
    F&& func, futures::detail::InlineContinuation allowInline) {
  throwIfContinued();
  getCore().setCallback(
      static_cast<F&&>(func), RequestContext::saveContext(), allowInline);
}
```

这里封装较为简单，仅仅只是增加了一个`RequestContext`，这部分可以结合[RequestContext](#RequestContext)阅读。

其中`RequestContext`是一个folly提供的线程维度全局单例(`thread local`)。其作用就是在线程之间传递数据。实现原理较为简单。当请求到来时，我们可以在请求处理的主线程中设置`RequestContext`，向其中添加数据。当我们在主线程之外使用别的线程池的时候，folly架构通过`RequestContext::saveContext()`方法获取到主线程的`RequestContext`数据。将该数据作为参数传递到异步线程要执行的函数中，这样在异步线程执行函数之前，通过`RequestContextScopeGuard`类，来将主线程的`RequestContext`数据拷贝到当前线程中，同时将自己的`RequestContext`存储起来，在`RequestContextScopeGuard`析构时，使用存储的`RequestContext`恢复自生的线程数据。以此导致数据屏蔽用户进行传递的作用。可以参考[doCallback](#doCallback)中`doAdd`传递的`func`，其在执行`callback`时会先创建`RequestContextScopeGuard`，并将这里传递的`RequestContext::saveContext()`作为参数。

该数据的一个典型使用场景是日志打点，对于某个大型服务来说，经常出现`coredump`，但是这些数据往往是由于实验导致的（毕竟如果上线就有core就上不去了），这时我们可能需要有一个添加方式来确定是哪个实验导致的问题，这个时候就可以使用`RequestContext`。我们在请求构建时，将实验参数写到`RequestContext`中，这样就会在请求使用到的所有线程中都可以获取到该数据，当出现core时，使用core的信号处理还是将实验参数打印出来。这样就不需要在每次使用异步线程的时候手动传递该值了。

#### `callback`返回future

对于`callback`返回`future`的场景来说，较为复杂，其实现逻辑就是，在构建时返回的future将会被运行时返回的future隐式替换，这样原本等待返回的future被设置result变成了等待执行时返回的future被设置result，其实现通过此前未介绍的`Proxy`。下面来看一下具体代码：

```c++
// Variant: returns a Future
// e.g. f.then([](T&& t){ return makeFuture<T>(t); });
template <class T>
template <typename F, typename R>
typename std::enable_if<R::ReturnsFuture::value, typename R::Return>::type
FutureBase<T>::thenImplementation(
    F&& func, R, futures::detail::InlineContinuation allowInline) {
  static_assert(R::Arg::ArgsSize::value == 2, "Then must take two arguments");
  typedef typename R::ReturnsFuture::Inner B;

  Promise<B> p;
  p.core_->initCopyInterruptHandlerFrom(this->getCore());

  // grab the Future now before we lose our handle on the Promise
  auto sf = p.getSemiFuture();
  auto e = getKeepAliveToken(this->getExecutor());
  sf.setExecutor(std::move(e));
  auto f = Future<B>(sf.core_);
  sf.core_ = nullptr;

  this->setCallback_(
      [state = futures::detail::makeCoreCallbackState(
           std::move(p), static_cast<F&&>(func))](
          Executor::KeepAlive<>&& ka, Try<T>&& t) mutable {
        if (!R::Arg::isTry() && t.hasException()) {
          state.setException(std::move(ka), std::move(t.exception()));
        } else {
          // Ensure that if function returned a SemiFuture we correctly chain
          // potential deferral.
          auto tf2 = detail_msvc_15_7_workaround::tryInvoke(
              R{}, state, ka.copy(), std::move(t));
          if (tf2.hasException()) {
            state.setException(std::move(ka), std::move(tf2.exception()));
          } else {
            auto statePromise = state.stealPromise();
            auto tf3 = chainExecutor(std::move(ka), *std::move(tf2));
            std::exchange(statePromise.core_, nullptr)
                ->setProxy(std::exchange(tf3.core_, nullptr));
          }
        }
      },
      allowInline);

  return f;
}
```

对应异常处理之前的部分和原来没有区别。当上一层执行未出异常时，首先执行本层的`callback`。如果本层`callback`出现异常，则进行与上一层出现异常一样的处理。未出异常时，获取到原本的下一层`promise`，将下一层的`promise`的core设置`Proxy`为返回的future的core。同时为了避免重复析构，将二者的core都设置为空值。

这里我们详细看一下`setProxy`执行逻辑：

```c++
void CoreBase::setProxy_(CoreBase* proxy) {
  DCHECK(!hasResult());

  proxy_ = proxy;

  auto state = state_.load(std::memory_order_acquire);
  switch (state) {
    case State::Start:
      if (folly::atomic_compare_exchange_strong_explicit(
              &state_,
              &state,
              State::Proxy,
              std::memory_order_release,
              std::memory_order_acquire)) {
        break;
      }
      assume(
          state == State::OnlyCallback ||
          state == State::OnlyCallbackAllowInline);
      FOLLY_FALLTHROUGH;

    case State::OnlyCallback:
    case State::OnlyCallbackAllowInline:
      proxyCallback(state);
      break;
    case State::OnlyResult:
    case State::Proxy:
    case State::Done:
    case State::Empty:
    default:
      terminate_with<std::logic_error>("setCallback unexpected state");
  }

  detachOne();
}
```

对应原本返回的future后面有设置的链式处理来说，执行的是`proxyCallback`处理函数，其逻辑如下：

```
void CoreBase::proxyCallback(State priorState) {
  // If the state of the core being proxied had a callback that allows inline
  // execution, maintain this information in the proxy
  futures::detail::InlineContinuation allowInline =
      (priorState == State::OnlyCallbackAllowInline
           ? futures::detail::InlineContinuation::permit
           : futures::detail::InlineContinuation::forbid);
  state_.store(State::Empty, std::memory_order_relaxed);
  proxy_->setExecutor(std::move(executor_));
  proxy_->setCallback_(std::move(callback_), std::move(context_), allowInline);
  proxy_->detachFuture();
  context_.~Context();
  callback_.~Callback();
}
```

可以看到，其逻辑就是把当前core的所有信息都迁移到`proxy_`中，包括要执行的`callback_`。这样，对于原本需要等待原`promise`设置`result`才能执行的`callback`来说，变成了依赖`proxy_`对应的`promise`被设置`result`，这样就完成了运行时对原`future`依赖的替换(妙啊!!!)。

这时，其实原本的`promise`就没有作用了，使用的就是替换后的future对应的`promise`。

### `callback`设置线程池

对于每个`callback`都可以设置对应的执行线程池，由于每个`callback`都有一个future持有，因此设置future使用的线程池即可。通过`via`方法设置，一般设置执行的线程池是在`SemiFuture`设置，较为简单：

```c++
template <class T>
Future<T> SemiFuture<T>::via(Executor::KeepAlive<> executor) && {
  folly::async_tracing::logSemiFutureVia(this->getExecutor(), executor.get());

  if (!executor) {
    throw_exception<FutureNoExecutor>();
  }
  // 不考虑defer执行
  if (auto deferredExecutor = this->getDeferredExecutor()) {
    deferredExecutor->setExecutor(executor.copy());
  }

  auto newFuture = Future<T>(this->core_);
  this->core_ = nullptr;
  newFuture.setExecutor(std::move(executor));

  return newFuture;
}
```



### 等待执行结束

对应异步来说，最终用户拿到一个`future`，我们需要在合适的时候等待异步的结束，调用`get`方法获取执行结果：

```c
template <class T>
T SemiFuture<T>::get() && {
  return std::move(*this).getTry().value();
}

template <class T>
Try<T> SemiFuture<T>::getTry() && {
  wait();
  auto future = folly::Future<T>(this->core_);
  this->core_ = nullptr;
  return std::move(std::move(future).result());
}

template <class T>
Future<T>& Future<T>::wait() & {
  futures::detail::waitImpl(*this);
  return *this;
}

template <class FutureType, typename T = typename FutureType::value_type>
void waitImpl(FutureType& f) {
  if (std::is_base_of<Future<T>, FutureType>::value) {
    f = std::move(f).via(&InlineExecutor::instance());
  }
  // short-circuit if there's nothing to do
  if (f.isReady()) {
    return;
  }

  Promise<T> promise;
  auto ret = convertFuture(promise.getSemiFuture(), f);
  FutureBatonType baton;
  f.setCallback_([&baton, promise = std::move(promise)](
                     Executor::KeepAlive<>&&, Try<T>&& t) mutable {
    promise.setTry(std::move(t));
    baton.post();
  });
  f = std::move(ret);
  baton.wait();
  assert(f.isReady());
}
```

可以看到，等待的核心逻辑在`waitImpl`中。其核心是增加一层调用链，使用条件变量来等待执行完成。

在`waitImpl`中，增加一个`promise`，`baton`可以理解为一个条件变量。新增的调用链设置的callback只有两个作用，将上一层的结果写到新的future里面，作为最终返回给用户的结构，执行`baton.post()`方法，让在等待条件变量的线程被唤醒。设置完成`callback`后，线程就调用等待条件变量成立的环节，一直扥到最后的`callback`被执行完成，唤醒该线程，从而返回结果。

### 异常处理

上面已经介绍了不少关于异常处理的内容，但是一直未介绍如果设置异常处理函数。首先关于c++异常，可以参考该博客[C++异常](https://www.cnblogs.com/QG-whz/p/5136883.html)。

设置异常处理使用`thenError`方法，其逻辑如下：

```c++
template <class T>
template <class ExceptionType, class F>
typename std::enable_if<
    !isFutureOrSemiFuture<invoke_result_t<F, ExceptionType>>::value,
    Future<T>>::type
Future<T>::thenError(tag_t<ExceptionType>, F&& func) && {
  Promise<T> p;
  p.core_->initCopyInterruptHandlerFrom(this->getCore());
  auto sf = p.getSemiFuture();
  auto* ePtr = this->getExecutor();
  auto e = folly::getKeepAliveToken(ePtr ? *ePtr : InlineExecutor::instance());

  this->setCallback_([state = futures::detail::makeCoreCallbackState(
                          std::move(p), static_cast<F&&>(func))](
                         Executor::KeepAlive<>&& ka, Try<T>&& t) mutable {
    if (auto ex = t.template tryGetExceptionObject<
                  std::remove_reference_t<ExceptionType>>()) {
      state.setTry(std::move(ka), makeTryWith([&] {
                     return state.invoke(std::move(*ex));
                   }));
    } else {
      state.setTry(std::move(ka), std::move(t));
    }
  });

  return std::move(sf).via(std::move(e));
}

 	template <class E>
  E* tryGetExceptionObject() {
    return hasException() ? e_.get_exception<E>() : nullptr;
  }
```

异常处理的返回值也区分`try`类型和`future`类型，一般`future`类似使用的较少，这里仅展示`try`类型。

可以看到`thenError`整体实现与`thenTry`,`thenValue`区别不大，同样是增加一层调用链，区别只在设置的`callback`上。

`thenError`一般有两个参数，第一个指示错误类型，当出错是进行类型匹配，只执行匹配到的那个，第二个则是设置的`callback`。

`callback`处理逻辑为，先判断上一层执行的结构是否是参数中的异常类型（当未出现异常或者出现异常类型不匹配，均返回false），如果匹配，则执行`callback`为下一层设置`result`，否则直接将结果传递到下一层，开始下一层的逻辑。结合[异常传递](#异常传递)更易理解。

## DAG

上面介绍了folly的异步框架实现，下面我们来看如何基于异步框架来实现DAG。首先介绍两个异步框架使用的额外方法`collect`函数和`SharedPromise`类。

### `collect`

`collect`方法参数是一个`future`的list，返回是一个`SemiFuture`，其`result`所有`future`的`result`构成的元组。其实现的功能是，新创建一个`future`，以参数中的所有`future`的结果作为其输入，即新建的`future`仅在参数中所有`future`获取到结果才执行自身的`callback`。这是DAG中重要的一环，即某个算子依赖多个算子，在多个算子执行完成时才能够执行，该功能即由`collect`来实现。

下面来看一下其具体实现：

```c++
template <typename... Fs>
SemiFuture<std::tuple<typename remove_cvref_t<Fs>::value_type...>> collect(
    Fs&&... fs) {
  using Result = std::tuple<typename remove_cvref_t<Fs>::value_type...>;
  struct Context {
    ~Context() {
      if (!threw.load(std::memory_order_relaxed)) {
        // if any of the input futures were off the end of a weakRef(), the
        // logic added in setCallback_ will not execute as an executor
        // weakRef() drops all callbacks added silently without executing them
        auto brokenPromise = false;
        folly::for_each(results, [&](auto& result) {
          if (!result.hasValue() && !result.hasException()) {
            brokenPromise = true;
          }
        });

        if (brokenPromise) {
          p.setException(BrokenPromise{pretty_name<Result>()});
        } else {
          p.setValue(unwrapTryTuple(std::move(results)));
        }
      }
    }
    Promise<Result> p;
    std::tuple<Try<typename remove_cvref_t<Fs>::value_type>...> results;
    std::atomic<bool> threw{false};
  };
  
  // 该步骤时获取执行器中所有defer执行器，这里不考虑该逻辑
  std::vector<futures::detail::DeferredWrapper> executors;
  futures::detail::stealDeferredExecutorsVariadic(executors, fs...);

  auto ctx = std::make_shared<Context>();
  futures::detail::foreach(
      [&](auto i, auto&& f) {
        f.setCallback_([i, ctx](Executor::KeepAlive<>&&, auto&& t) {
          if (t.hasException()) {
            if (!ctx->threw.exchange(true, std::memory_order_relaxed)) {
              ctx->p.setException(std::move(t.exception()));
            }
          } else if (!ctx->threw.load(std::memory_order_relaxed)) {
            std::get<i.value>(ctx->results) = std::move(t);
          }
        });
      },
      static_cast<Fs&&>(fs)...);

  auto future = ctx->p.getSemiFuture();
  // 不考虑该逻辑，不考虑defer执行器
  if (!executors.empty()) {
    auto work = [](Try<typename decltype(future)::value_type>&& t) {
      return std::move(t).value();
    };
    future = std::move(future).defer(work);
    const auto& deferredExecutor = futures::detail::getDeferredExecutor(future);
    deferredExecutor->setNestedExecutors(std::move(executors));
  }
  return future;
}
```

通过看代码，我们可以看到，其实现就是利用`shared_ptr`的特性，即只在引用计数为0时，才执行析构函数。这样，我们让参数中所有`folly`均通过`callback`持有`shared_ptr`，让所有`future`执行`callback`结束时，就会执行析构函数，自动将`shared_ptr`的引用计数减一，直到所有的`future`均执行完成`callback`，就会最终析构`shared_ptr`，这时在析构函数中对新建的`promise`设置`result`，这样新建的`promise`就依赖所有参数中`future`执行结果，保证在所有`future`的`callback`执行·完成才执行。

其中`Context`就充当该`shared_ptr`的数据。对每个传递的`future`设置`callback`为将结果写到`Context`中对应的位置（或者抛出异常）。

在参数中所有`future`的`callback`执行完成后，就获取到了所有`folly`的结果，执行`Context`的析构函数，判断是否有异常产生，如果没有异常产生，则设置下一层的`promise`，完成计算。对应返回的`future`来说，如果后面还有别的链式执行逻辑，则会在这里被设置`result`后继续执行，如果没有，则用户直接获取到结果。

### `SharedPromise`类

上面介绍的`collect`解决了一个算子依赖多个算子的情况，但还有另一个情况，就是多个算了依赖了同一个算子。这部分则是通过`SharedPromise`类来实现，其实现逻辑也相对简单，就是其会持有一个`promise`的list，当某个算子要依赖该算子时，就会将list增加一个`promise`，这样依赖其的算子就会获得一个`future`，当该算子执行完成后，会对该list中所有`promise`设置`result`，这样持有该算子`future`的·所有算子都可以继续执行了。

我们来看其具体实现，仅看核心数据和接口：

```c++
template <class T>
class SharedPromise {
 public:
 	SemiFuture<T> getSemiFuture() const;
  void setTry(Try<T>&& t);
  mutable Mutex mutex_;
  mutable Defaulted<size_t> size_;
  Defaulted<Try<T>> try_;
  mutable std::vector<Promise<T>> promises_;
  std::function<void(exception_wrapper const&)> interruptHandler_;
};

template <class T>
SemiFuture<T> SharedPromise<T>::getSemiFuture() const {
  std::lock_guard<std::mutex> g(mutex_);
  size_.value++;
  if (hasResult()) {
    return makeFuture<T>(Try<T>(try_.value));
  } else {
    promises_.emplace_back();
    if (interruptHandler_) {
      promises_.back().setInterruptHandler(interruptHandler_);
    }
    return promises_.back().getSemiFuture();
  }
}

template <class T>
void SharedPromise<T>::setTry(Try<T>&& t) {
  std::vector<Promise<T>> promises;
  
  // 不能重复设置result
  {
    std::lock_guard<std::mutex> g(mutex_);
    if (hasResult()) {
      throw_exception<PromiseAlreadySatisfied>();
    }
    try_.value = std::move(t);
    promises.swap(promises_);
  }

  for (auto& p : promises) {
    p.setTry(Try<T>(try_.value));
  }
}
```

在使用中，每个算子持有一个`SharedPromise`，当某个算子依赖自身时，通过`getSemiFuture()`方法获取一个`SemiFuture`。当算子执行完成后，通过`setTry`向所有生成的`promise`设置结果，这样所有依赖该算子的算子，都可以开始执行其`callback`。

介绍完了上面的两个依赖，我们来看一下实际DAG的实现，folly实现了一个简单的DAG class `FutureDAG`。

### `FutureDAG`

`FutureDAG`使用future&promise异步框架和上面介绍的两个工具实现了通用的DAG。

```c++
class FutureDAG : public std::enable_shared_from_this<FutureDAG> {
 public:
  static std::shared_ptr<FutureDAG> create(
      Executor::KeepAlive<> defaultExecutor) {
    return std::shared_ptr<FutureDAG>(
        new FutureDAG(std::move(defaultExecutor)));
  }

  typedef size_t Handle;
  typedef std::function<Future<Unit>()> FutureFunc;

  Handle add(FutureFunc func, Executor::KeepAlive<> executor) {
    nodes.emplace_back(std::move(func), executor);
    return nodes.size() - 1;
  }

  void remove(Handle a) {
    if (a >= nodes.size()) {
      return;
    }

    if (nodes[a].hasDependents) {
      for (auto& node : nodes) {
        auto& deps = node.dependencies;
        deps.erase(
            std::remove(std::begin(deps), std::end(deps), a), std::end(deps));
        for (Handle& handle : deps) {
          if (handle > a) {
            handle--;
          }
        }
      }
    }

    nodes.erase(nodes.begin() + a);
  }

  void reset() {
    // Delete all but source node, and reset dependency properties
    Handle source_node = 0;
    std::unordered_set<Handle> memo;
    for (auto& node : nodes) {
      for (Handle handle : node.dependencies) {
        memo.insert(handle);
      }
    }
    for (Handle handle = 0; handle < nodes.size(); handle++) {
      if (memo.find(handle) == memo.end()) {
        source_node = handle;
      }
    }

    nodes.erase(nodes.begin(), nodes.begin() + source_node);
    nodes.erase(nodes.begin() + 1, nodes.end());
    nodes[0].hasDependents = false;
    nodes[0].dependencies.clear();
  }

  void dependency(Handle a, Handle b) {
    nodes[b].dependencies.push_back(a);
    nodes[a].hasDependents = true;
  }

  void clean_state(Handle source, Handle sink) {
    for (auto handle : nodes[sink].dependencies) {
      nodes[handle].hasDependents = false;
    }
    nodes[0].hasDependents = false;
    remove(source);
    remove(sink);
  }

  Future<Unit> go() {
    if (hasCycle()) {
      return makeFuture<Unit>(std::runtime_error("Cycle in FutureDAG graph"));
    }
    std::vector<Handle> rootNodes;
    std::vector<Handle> leafNodes;
    for (Handle handle = 0; handle < nodes.size(); handle++) {
      if (nodes[handle].dependencies.empty()) {
        rootNodes.push_back(handle);
      }
      if (!nodes[handle].hasDependents) {
        leafNodes.push_back(handle);
      }
    }

    auto sinkHandle = add([] { return Future<Unit>(); }, defaultExecutor_);
    for (auto handle : leafNodes) {
      dependency(handle, sinkHandle);
    }

    auto sourceHandle = add(nullptr, defaultExecutor_);
    for (auto handle : rootNodes) {
      dependency(sourceHandle, handle);
    }

    for (Handle handle = 0; handle < nodes.size() - 1; handle++) {
      std::vector<Future<Unit>> dependencies;
      for (auto depHandle : nodes[handle].dependencies) {
        dependencies.push_back(nodes[depHandle].promise.getFuture());
      }

      collect(dependencies)
          .via(nodes[handle].executor)
          .thenValue([this, handle](std::vector<Unit>&&) {
            nodes[handle].func().then([this, handle](Try<Unit>&& t) {
              nodes[handle].promise.setTry(std::move(t));
            });
          })
          .thenError([this, handle](exception_wrapper ew) {
            nodes[handle].promise.setException(std::move(ew));
          });
    }

    nodes[sourceHandle].promise.setValue();
    return nodes[sinkHandle].promise.getFuture().thenValue(
        [that = shared_from_this(), sourceHandle, sinkHandle](Unit) {
          that->clean_state(sourceHandle, sinkHandle);
        });
  }

 private:
  FutureDAG(Executor::KeepAlive<> defaultExecutor)
      : defaultExecutor_{std::move(defaultExecutor)} {}

  bool hasCycle() {
    // Perform a modified topological sort to detect cycles
    std::vector<std::vector<Handle>> dependencies;
    for (auto& node : nodes) {
      dependencies.push_back(node.dependencies);
    }

    std::vector<size_t> dependents(nodes.size());
    for (auto& dependencyEdges : dependencies) {
      for (auto handle : dependencyEdges) {
        dependents[handle]++;
      }
    }

    std::vector<Handle> handles;
    for (Handle handle = 0; handle < nodes.size(); handle++) {
      if (!nodes[handle].hasDependents) {
        handles.push_back(handle);
      }
    }

    while (!handles.empty()) {
      auto handle = handles.back();
      handles.pop_back();
      while (!dependencies[handle].empty()) {
        auto dependency = dependencies[handle].back();
        dependencies[handle].pop_back();
        if (--dependents[dependency] == 0) {
          handles.push_back(dependency);
        }
      }
    }

    for (auto& dependencyEdges : dependencies) {
      if (!dependencyEdges.empty()) {
        return true;
      }
    }

    return false;
  }

  struct Node {
    Node(FutureFunc&& funcArg, Executor::KeepAlive<> executorArg)
        : func(std::move(funcArg)), executor(std::move(executorArg)) {}

    FutureFunc func{nullptr};
    Executor::KeepAlive<> executor;
    SharedPromise<Unit> promise;
    std::vector<Handle> dependencies;
    bool hasDependents{false};
    bool visited{false};
  };

  std::vector<Node> nodes;
  Executor::KeepAlive<> defaultExecutor_;
};
```

`Node`是一个算子，其持有要执行的函数，要执行该任务的线程池，一个`SharedPromise`，`dependencies`节点自身依赖的节点列表，`hasDependents`表示是否有节点依赖自身，`visited`应该是旧版本判断是否有环的，目前没有用。

`FutureDAG`使用逻辑是，创建一个空的`FutureDAG`实例，使用`add`向其中增加节点，同时返回节点对应下标，之后使用`dependency`构建节点间依赖关系，其中参数含义是：b依赖a。构建完成后调用`go`来执行DAG，在`go`中，首先判断节点间依赖是否成环，如果成环则不开执行，返回有异常的`future`，否则利用`collect`和`SharedPromise`构建执行的依赖关系。

`hasCycle`函数判断是否成环，其逻辑较为简单。首先计算每个节点被依赖的次数存到`dependents`中，将不被别的节点依赖的节点放到`handles`中，从`handles`中取出一个节点，将该节点依赖的`depends`清空，并将所有该节点依赖的节点的计数减一，如果减一后结果为0，表示不再有别的节点依赖这个节点了，这时将该节点加到`handles`中，之后一直从`handles`中取数据，直到`handles`为空，此时判断是否还有未清空的`depends`，如果有则表示有环。

对有无环的DAG执行来说，首先区分叶节点和根节点，根节点是不依赖别的节点结果可以立即执行的节点，叶节点是哪些没有节点依赖该节点的节点。对应叶节点，增加一个`sink`节点，让该节点依赖所有的叶节点，作为执行结束标识。对应根节点，增加一个`source`节点，让所有根节点都依赖该节点，作为DAG启动标识。

遍历每个节点，通过其依赖的节点的`promise`获取到一个`future`list，通过`collect`该list构建一个新的`future`，设置一个`callback`，则该`collback`中执行节点对应的函数，并设置节点自生`SharedPromise`的`result`。同时设置异常处理函数。

之后设置`source`节点的`result`开始DAG执行，返回`sink`节点新增的一个调用链，用于清理`source`和`sink`节点。

对应返回的`future`，用户调用`std::move(f).get()`方法等待执行完成即可。



# 参考

## C++高级方法

### std::enable_if

enable_if 的定义类似于下面的代码：（只有 Cond = true 时定义了 type）

```c++
template<bool Cond, class T = void> struct enable_if {};
template<class T> struct enable_if<true, T> { typedef T type; };
```

这样的话，`enable_if<true, T>::type` 即为 `T`，而 `enable_if<false, T>::type` 会引发编译错误（在 SFINAE 下，即不将包含这一 enable_if 的函数 / 类作为候选）。

### 类模板的偏特化

https://sg-first.gitbooks.io/cpp-template-tutorial/content/jie_te_hua_yu_pian_te_hua.html

### std::decay_t

去除变量的所有引用属性，获取其原始class




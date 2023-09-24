---
title: folly协程Task学习
date: 2023-09-24 17:59:18
tags:
    folly
categories:
    coro
mathjax:
    true
description: 听说Facebook目前已经将内部的future替换成协程coro了，恰巧工作中有相关协程使用的讨论，于是看一下folly中基于C++20协程封装的框架。
---

<center/><font size = 8>C++协程之folly coro</font></center>

之前，介绍了[folly的异步框架](https://www.yinkuiwang.cn/2023/01/08/folly%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E4%B8%8EDAG/)，其调度中实际执行任务时由线程池完成。听说Facebook目前已经将内部的`future`替换成协程`coro`了，恰巧工作中有相关协程使用的讨论，于是看一下folly中基于C++20协程封装的框架。这里不介绍C++20协程的基本使用方式，想要了解可以看下面两个文档。

[C++20协程官方文档](https://en.cppreference.com/w/cpp/language/coroutines)

[协程原理及C++20使用](https://lewissbaker.github.io/)

# 基础背景

协程相较于线程，其性能更优，由开发人员自己实现任务切换，而不用操作系统进行切换，避免了操作系统线程切换的开销。同时，协程提供了新的开发范式，协程可以看做一个天然的动态DAG调度框架，当某段处理逻辑计算依赖某个数据时，我们可以通过协程切换，先去获取到对应数据，等拿到对应数据后在切换回原逻辑继续执行。对于在执行过程中才能判断是否要执行的数据，对于静态图来说，其支持较为困难，可能需要在图中增加动态的disable逻辑，而使用协程，其天然支持动态决策，灵活性更高。

例如如下一个简单逻辑：

```
def Function():
	a = FunctionA();
	template result;
	if(a) {
		result = FunctionB();
	} else {
		result = FunctionC()
	}
	// 对result进一步处理
	return result;
```

对于如上逻辑，其含义是，Function的执行依赖了三个函数`FunctionA`，`FunctionB`,`FunctionC`。其中对于B,C函数来说，其执行依赖于A的结果，对于静态构图，可能的形式为：

```
            FunctionA
                /\
               /  \
              /    \
      FunctionB   FunctionC
      				\    /
      				 \  /
      			 Function
```

在`FunctionB`和`FunctionC`中根据`FunctionA`的值进行判断，来决定是否执行。

其实现方式较为繁琐，需要将一个节点拆分成为多个节点。

使用协程泽不需要如此繁琐，其实现如上面的伪代码基本一致，可能变成如下形式：

```
def Function():
	co_await a = FunctionA();
	template result;
	if(a) {
		co_await result = FunctionB();
	} else {
		co_await result = FunctionC()
	}
	// 对result进一步处理
	return result;
```

当我们需要某个数据时，使用协程切换，将执行逻辑切换到对应的数据获取方法上即可，当取回数据后，再回到原函数中继续进行处理。处理协程函数也是放到一个大的线程池中处理。这样，我们将要获取的数据全都直接丢到线程池中，由协程调度来自动寻找其依赖的函数，自动丢到线程池中，这样就只需要一个线程池，一个协程调度，就完美的实现了一个动态dag。

这里还存在一些问题，会在后续讲解中逐步回答：

1. 如果一个节点被多个算子依赖，如何避免被重复计算。
2. 一个节点依赖多个数据，如果每一次执行到要使用的位置在切换协程获取，那会导致每个字段获取串行执行，如何让依赖尽可能并发执行（这个其实和动态图有一定的冲突，但往往是一个强需求）。

# Task使用

`folly`实现的coro核心是Task类，folly官方文档上有十分详细的介绍，这里只贴出来一个使用样例：

```c++
#include "folly/experimental/coro/BlockingWait.h"
#include <coroutine>
#include <mutex>
#include <string>
#include <folly/executors/GlobalExecutor.h>
#include <iostream>
#include <folly/init/Init.h>
#include<folly/experimental/coro/Task.h>
#include <folly/experimental/coro/Collect.h>
#include <sys/select.h>
#include <time.h>
#include <folly/experimental/coro/SharedMutex.h>
#include "folly/executors/CPUThreadPoolExecutor.h"
using namespace folly;
using namespace folly::coro;

int64_t get_us_time() {
    struct timeval t;
    gettimeofday(&t, NULL);
    return (int64_t)(t.tv_sec * 1000000 + t.tv_usec);
}

static folly::CPUThreadPoolExecutor& get_executor1() {
    static folly::CPUThreadPoolExecutor executor(1);
    return executor;
}

static folly::CPUThreadPoolExecutor& get_executor2() {
    static folly::CPUThreadPoolExecutor executor(1);
    return executor;
}

SharedMutexFair coro_lock;

std::map<std::string, int> global_value = {
    {"a", 0},
    {"b", 0},
    {"c", 0},
    {"d", 0}
};
int global = 0;

Task<void> a() {
    global_value["a"] = ++global;
    std::cout<<"a process time is "<<get_us_time()<<std::endl;
    co_return;
}

Task<void> b() {
    sleep(2);
    global_value["b"] = ++global;
    std::cout<<"b process time is "<<get_us_time()<<std::endl;
    co_return;
}

Task<void> c() {
    global_value["c"] = ++global;
    std::cout<<"c process time is "<<get_us_time()<<std::endl;
    co_return;
}

Task<void> d() {
    global_value["d"] = ++global;
    std::cout<<"d process time is "<<get_us_time()<<std::endl;
    co_return;
}

Task<void> getA() {
    // Lock lock(mutex);
    auto lock = co_await coro_lock.co_scoped_lock();
    std::cout<<"process get A"<<std::endl;
    co_return;
}

Task<void> getB() {
    // Lock lock(mutex);
    auto lock = co_await coro_lock.co_scoped_lock();
    co_await getA().scheduleOn(getGlobalCPUExecutor());
    std::cout<<"process get B"<<std::endl;
    co_return;
}


Task<void> getB_unlock() {
    co_await getA().scheduleOn(getGlobalCPUExecutor());
    std::cout<<"process get B unlock"<<std::endl;
    co_return;
}

Task<bool> sycn() {
    std::vector<Task<void>> sum;
    std::cout << "Coroutine started on thread: " << std::this_thread::get_id() << '\n';
    sum.push_back(a());
    sum.push_back(b());
    sum.push_back(c());
    sum.push_back(d());
    // co_await folly::coro::collectAll(sum);
    try {
        co_await folly::coro::collectAllRange(std::move(sum));
    } catch (...) {
        std::cout<<"catch error"<<std::endl;
    }
    std::cout << "Coroutine ended on thread: " << std::this_thread::get_id() << '\n';
    co_return true;
}

Task<bool> sycn_v2() {
    std::vector<Task<void>> sum;
    std::cout << "Coroutine started on thread: " << std::this_thread::get_id() << '\n';
    sum.push_back(a());
    sum.push_back(b());
    sum.push_back(c());
    sum.push_back(d());
    // co_await folly::coro::collectAll(sum);
    co_await folly::coro::collectAllRange(std::move(sum)).scheduleOn(&get_executor1());
    std::cout << "Coroutine ended on thread: " << std::this_thread::get_id() << '\n';
    co_return true;
}

Task<bool> asycn() {
    std::vector<TaskWithExecutor<void>> sum;
    std::cout << "Coroutine started on thread: " << std::this_thread::get_id() << '\n';
    sum.push_back(a().scheduleOn(getGlobalCPUExecutor()));
    sum.push_back(b().scheduleOn(getGlobalCPUExecutor()));
    sum.push_back(c().scheduleOn(getGlobalCPUExecutor()));
    sum.push_back(d().scheduleOn(getGlobalCPUExecutor()));
    co_await folly::coro::collectAllRange(std::move(sum));
    std::cout << "Coroutine ended on thread: " << std::this_thread::get_id() << '\n';
    co_return true;
}



int main(int argc, char *argv[])
{
    folly::init(&argc, &argv);
    // std::cout<<"process1"<<std::endl;
    // auto exe = getCPUExecutor();
    LOG(INFO) << "info test";
    auto task1 = sycn();
    folly::coro::blockingWait(std::move(task1).scheduleOn(&get_executor2()));
    std::cout<<"a is "<<global_value["a"]<<" b is "<<global_value["b"]<<" c is "<<global_value["c"]<<" d is "<<global_value["d"]<<std::endl;
    std::cout<<"global is "<<global<<std::endl;

    auto task3 = sycn_v2();
    folly::coro::blockingWait(std::move(task3).scheduleOn(&get_executor2()));
    std::cout<<"a is "<<global_value["a"]<<" b is "<<global_value["b"]<<" c is "<<global_value["c"]<<" d is "<<global_value["d"]<<std::endl;
    std::cout<<"global is "<<global<<std::endl;

    auto task2 = asycn();
    folly::coro::blockingWait(std::move(task2).scheduleOn(&get_executor2()));
    std::cout<<"a is "<<global_value["a"]<<" b is "<<global_value["b"]<<" c is "<<global_value["c"]<<" d is "<<global_value["d"]<<std::endl;
    std::cout<<"global is "<<global<<std::endl;

    folly::coro::blockingWait(getB_unlock().scheduleOn(&get_executor2()));

    folly::coro::blockingWait(getB().scheduleOn(&get_executor2()));
    google::ShutdownGoogleLogging();
    return 0;
}
// g++ -L/opt/lib -I/opt/include test_folly_coro.cpp -std=c++20 -lfolly -lglog -lgflags -lpthread -ldl -ldouble-conversion -lfmt -levent -lboost_context

```

执行结果：

```bash
$ ./a.out 
Coroutine started on thread: 139646669821504
a process time is 1695172727482615
b process time is 1695172729482921
c process time is 1695172729482970
d process time is 1695172729482983
Coroutine ended on thread: 139646669821504
a is 1 b is 2 c is 3 d is 4
global is 4
Coroutine started on thread: 139646669821504
a process time is 1695172729484680
b process time is 1695172731484800
c process time is 1695172731484854
d process time is 1695172731484864
Coroutine ended on thread: 139646669821504
a is 5 b is 6 c is 7 d is 8
global is 8
Coroutine started on thread: 139646669821504
a process time is 1695172731486864
c process time is 1695172731487174
d process time is 1695172731487422
b process time is 1695172733486971
Coroutine ended on thread: 139646669821504
a is 9 b is 12 c is 10 d is 11
global is 12
process get A
process get B unlock
^C
$
```

可以看到调用`sycn`时，其输出严格有序，调用`asycn`时，其输出就是无序的了。这里说明了一个问题是，调用`collectAllRange`时，如果task本身是同步方法，则其会被串行调用，如果其本身是异步方法，则调用就会异步执行。

同时可以看到调用异步方法`asycn`时，在`co_await`前后执行执行在相同的线程池，虽然我们设置了`co_await`等待的`task`在另外的线程池执行。这是因为`Task`的`promise_type`的`await_transform`方法调用了`co_viaIfAsync`，保证协程始终在指定线程池中执行。当`await_suspend`返回`void`或者`false`时，会立即返回给协程函数的调用者。同时协程处于`suspend`状态。按照如此逻辑，上面实例代码，在执行`async`函数内部逻辑时（即`async`协程第一次被`resume`，即被`co_await`时），在调用`co_await`方法前，逻辑都执行在主线程中，当调用`co_await`时，直接返回到主流程中开始执行下面的语句了，而`async`协程被挂起，被`co_awiat`的协程被分配到线程池中执行，在这些协程执行结束后，重新唤醒`async`协程，由于`co_viaIfAsync`方法封装了一层协程，保证被唤醒的`async`协程依然在原线程池中执行。

这里想要说明的一点是，线程池执行协程函数时，如果被`suspend`而未拉起其他协程的协程（当`await_suspend`返回`coroutine handle `时，会立即执行`coroutine handle`对应的协程，原协程被挂起，至于执行完成新的协程后的处理逻辑，则由新的协程处理函数决定，可以选择恢复原协程，也可以选择再拉起一个协程，或者什么都不干。相当于使用新的协程上下文替换原协程的上下文，新协程执行逻辑和原协程无关，执行完成也不存在要返回到某个原协程的什么位置的概念），并不会占用线程池，因为<font color = red>被suspend的协程函数，会立即返回到调用处（不是返回到调用`co_await`的地方，而是协程函数的入口位置）</font>，在task中，一般是回到`resumeCoroutineWithNewAsyncStackRoot`函数中的`h.resume();`，这样线程就会认为执行完成了该task，会继续从线程池的task任务池中取其他的task。而未完成的协程调用什么时候继续呢，会在线程池调用的某个方法中调用被暂停协程的`resume`方法时被继续执行。这里也说明一个问题，应该尽可能的避免协程被suspend而不拉起新的协程，因为当出现这种情况时，线程池会需要从task队列中查找新的任务，这势必会造成额外的开销，相当于协程的链式调度切换失效了。不论怎么样，当线程池中执行的都是协程函数时，可以大大减少线程池数量，理论上来说，线程池数量和cpu核数绑定即可。

这里还有一个对协程锁的验证，可以看到，程序最终会卡住，这时因为执行`getB()`函数时，我们获取了锁，在没有释放的前提下协程被切换到了执行`getA()`函数，这里再次尝试获取锁，这就造成了死锁。这里说明了，协程锁不能解决由于协程切换造成的死锁问题，使用协程锁，更需要考虑死锁问题，要保证协程切换时锁要被正常释放。

对于这个例子，可以仅仅简单运行一下，当前不必深究，当完整了解coro实现后可以回过来来再看一下。

`Task`包含了大量的基础类，这里我们进行逐一介绍。

# TaskPromiseBase

`TaskPromiseBase`是Task的`promise_type`的基类，其决定了返回Task协程函数的实际执行逻辑。因此先介绍其具体实现。

`TaskPromiseBase`类包含如下成员：

```C++
class TaskPromiseBase {
 private:
	ExtendedCoroutineHandle continuation_;
  folly::AsyncStackFrame asyncFrame_;
  folly::Executor::KeepAlive<> executor_;
  folly::CancellationToken cancelToken_;
  bool hasCancelTokenOverride_ = false;
  bool ownsAsyncFrame_ = true;

 protected:
  enum class BypassExceptionThrowing : uint8_t {
    INACTIVE,
    ACTIVE,
    REQUESTED,
  } bypassExceptionThrowing_{BypassExceptionThrowing::INACTIVE};
}
```

## ExtendedCoroutineHandle

该类是`coroutine_handle<void>`的拓展版本，其定义如下：

```c++
class ExtendedCoroutineHandle {
 public:
  template <typename Promise>
  /*implicit*/ ExtendedCoroutineHandle(
      coroutine_handle<Promise> handle) noexcept
      : basic_(handle), extended_(fromBasic(handle)) {}

  /*implicit*/ ExtendedCoroutineHandle(coroutine_handle<> handle) noexcept
      : basic_(handle) {}

  /*implicit*/ ExtendedCoroutineHandle(ExtendedCoroutinePromise* ptr) noexcept
      : basic_(ptr->getHandle()), extended_(ptr) {}

  ExtendedCoroutineHandle() noexcept = default;

  void resume() { basic_.resume(); }

  void destroy() { basic_.destroy(); }

  coroutine_handle<> getHandle() const noexcept { return basic_; }

  ExtendedCoroutinePromise* getPromise() const noexcept { return extended_; }

  std::pair<ExtendedCoroutineHandle, AsyncStackFrame*> getErrorHandle(
      exception_wrapper& ex) {
    if (extended_) {
      return extended_->getErrorHandle(ex);
    }
    return {basic_, nullptr};
  }

  explicit operator bool() const noexcept { return !!basic_; }

 private:
  template <typename Promise>
  static auto fromBasic(coroutine_handle<Promise> handle) noexcept {
    if constexpr (std::is_convertible_v<Promise*, ExtendedCoroutinePromise*>) {
      return static_cast<ExtendedCoroutinePromise*>(&handle.promise());
    } else {
      return nullptr;
    }
  }

  coroutine_handle<> basic_;
  ExtendedCoroutinePromise* extended_{nullptr};
};
```

其实际存储的是调用`co_await`的协程的`coroutine_handle`。用于执行完成当前协程函数后，唤醒原来被切换出去的协程函数。举例来说：

```
Task<T> func {
	...
	co_await ()->Task<T>{
		....
	}();
	...
}
```

在这个协程函数中，当执行到co_await时，当前协程函数`func`会被暂停，执行调度到co_await对应的`lambda`函数中去，但是当对于的`lambda`执行完成后，如何回到原来的函数呢，这个工作是由Task自己完成的，不需要用户指定，其实现的核心就是这里的`continuation_`（ExtendedCoroutineHandle类），lambda函数对应的Task中的`promise_type`会持有`func`协程的`coroutine_handle`。当lambda执行完成后，调用`coroutine_handle`的resume即唤醒`func`协程。

其中`ExtendedCoroutinePromise`是扩展的promise，这里其仅仅充当接口（因此是纯虚类），其定义如下：

```c++
class ExtendedCoroutinePromise {
 public:
  virtual coroutine_handle<> getHandle() = 0;
  // Types may provide a more efficient resumption path when they know they will
  // be receiving an error result from the awaitee.
  // If they do, they might also update the active stack frame.
  virtual std::pair<ExtendedCoroutineHandle, AsyncStackFrame*> getErrorHandle(
      exception_wrapper&) = 0;

 protected:
  ~ExtendedCoroutinePromise() = default;
};
```

## AsyncStackFrame

异步栈帧，其定义如下：

```c++
// An async stack frame contains information about a particular
// invocation of an asynchronous operation.
//
// For example, asynchronous operations implemented using coroutines
// would have each coroutine-frame contain an instance of AsyncStackFrame
// to record async-stack trace information for that coroutine invocation.
struct AsyncStackFrame {
 public:
  AsyncStackFrame() = default;

  // The parent frame is the frame of the async operation that is logically
  // the caller of this frame.
  AsyncStackFrame* getParentFrame() noexcept;
  const AsyncStackFrame* getParentFrame() const noexcept;
  void setParentFrame(AsyncStackFrame& frame) noexcept;

  // Get access to the current stack-root.
  //
  // This is only valid for either the root or leaf AsyncStackFrame
  // in a chain of frames.
  //
  // In the case of an active leaf-frame it is used as a cache to
  // avoid accessing the thread-local when pushing/popping frames.
  // In the case of the root frame (which has a null parent frame)
  // it points to an AsyncStackRoot that contains information about
  // the normal-stack caller.
  AsyncStackRoot* getStackRoot() noexcept;

  // The return address is generallty the address of the code in the
  // caller that will be executed when the operation owning the current
  // frame completes.
  void setReturnAddress(void* p = FOLLY_ASYNC_STACK_RETURN_ADDRESS()) noexcept;
  void* getReturnAddress() const noexcept;

 private:
  friend AsyncStackRoot;

  friend AsyncStackFrame& getDetachedRootAsyncStackFrame() noexcept;
  friend void activateAsyncStackFrame(
      folly::AsyncStackRoot&, folly::AsyncStackFrame&) noexcept;
  friend void deactivateAsyncStackFrame(folly::AsyncStackFrame&) noexcept;
  friend void pushAsyncStackFrameCallerCallee(
      folly::AsyncStackFrame&, folly::AsyncStackFrame&) noexcept;
  friend void checkAsyncStackFrameIsActive(
      const folly::AsyncStackFrame&) noexcept;
  friend void popAsyncStackFrameCallee(folly::AsyncStackFrame&) noexcept;

  // Pointer to the async caller's stack-frame info.
  //
  // This forms a linked-list of frames that make up a stack.
  // The list is terminated by a null pointer which indicates
  // the top of the async stack - either because the operation
  // is detached or because the next frame is a thread that is
  // blocked waiting for the async stack to complete.
  AsyncStackFrame* parentFrame = nullptr;

  // Instruction pointer of the caller of this frame.
  // This will typically be either the address of the continuation
  // of this asynchronous operation, or the address of the code
  // that launched this asynchronous operation. May be null
  // if the address is not known.
  //
  // Typically initialised with the result of a call to
  // FOLLY_ASYNC_STACK_RETURN_ADDRESS().
  void* instructionPointer = nullptr;

  // Pointer to the stack-root for the current thread.
  // Cache this in each async-stack frame so we don't have to
  // read from a thread-local to get the pointer.
  //
  // This pointer is only valid for the top-most stack frame.
  // When a frame is pushed or popped it should be copied to
  // the next frame, etc.
  //
  // The exception is for the bottom-most frame (ie. where
  // parentFrame == null). In this case, if stackRoot is non-null
  // then it points to a root that is currently blocked on some
  // thread waiting for the async work to complete. In this case
  // you can find the information about the stack-frame for that
  // thread in the AsyncStackRoot and can use it to continue
  // walking the stack-frames.
  AsyncStackRoot* stackRoot = nullptr;
};


inline AsyncStackFrame* AsyncStackFrame::getParentFrame() noexcept {
  return parentFrame;
}

inline const AsyncStackFrame* AsyncStackFrame::getParentFrame() const noexcept {
  return parentFrame;
}

inline void AsyncStackFrame::setParentFrame(AsyncStackFrame& frame) noexcept {
  parentFrame = &frame;
}

inline AsyncStackRoot* AsyncStackFrame::getStackRoot() noexcept {
  return stackRoot;
}

inline void AsyncStackFrame::setReturnAddress(void* p) noexcept {
  instructionPointer = p;
}

inline void* AsyncStackFrame::getReturnAddress() const noexcept {
  return instructionPointer;
}

inline void AsyncStackRoot::setTopFrame(AsyncStackFrame& frame) noexcept {
  assert(this->topFrame.load(std::memory_order_relaxed) == nullptr);
  assert(frame.stackRoot == nullptr);
  frame.stackRoot = this;
  this->topFrame.store(&frame, std::memory_order_release);
}

inline AsyncStackFrame* AsyncStackRoot::getTopFrame() const noexcept {
  return topFrame.load(std::memory_order_relaxed);
}

inline void AsyncStackRoot::setStackFrameContext(
    void* framePtr, void* ip) noexcept {
  stackFramePtr = framePtr;
  returnAddress = ip;
}

inline void* AsyncStackRoot::getStackFramePointer() const noexcept {
  return stackFramePtr;
}

inline void* AsyncStackRoot::getReturnAddress() const noexcept {
  return returnAddress;
}

inline const AsyncStackRoot* AsyncStackRoot::getNextRoot() const noexcept {
  return nextRoot;
}

inline void AsyncStackRoot::setNextRoot(AsyncStackRoot* next) noexcept {
  nextRoot = next;
}
```

其中`parentFrame`代表该异步操作的调用者的栈帧，通过`parentFrame`将调用栈串连起来，调用链通过一个空指针终止，对于`paremtFrame`为空指针去情况，要么表示该栈帧是被分离的状态（销毁），要么表示下一帧是阻塞等待异步堆栈完成的线程（栈顶？）。

`instructionPointer`表示这个栈帧调用者的指令指针。这通常是此异步操作的延续地址，或启动此异步操作的代码的地址。 如果地址未知，则可能为空。该变量的赋值通常使用`FOLLY_ASYNC_STACK_RETURN_ADDRESS()`方法。

其定义如下：

```
#define FOLLY_ASYNC_STACK_RETURN_ADDRESS() __builtin_return_address(0)
```

其中`__builtin_return_address`可以看[__builtin_return_address](https://gcc.gnu.org/onlinedocs/gcc/Return-Address.html)。

简单来说`__builtin_return_address`是编译器内建函数，作用是用于获取当前函数或者调用函数的返回地址，当参数是0时，表示的是当前函数的返回地址，参数为1时表示的是调用该函数的函数返回地址。

`stackRoot`是指向当前线程栈根的指针（stack root）。通过这里cache该变量，我们就不需要通过读取一个线程纬度的数据来获取该指针了。该指针只对最顶层的栈帧有效，当一个栈帧被入栈或者出栈时，该值需要被进行拷贝到对应的栈帧上。一个例外是最底层的栈帧，如果最底层的栈帧中该值不为空，则表示指向当前被阻塞在等待一个异步线程完成的根上（指向阻塞当前线程的异步线程的根上）。在这种情况下，您可以在`AsyncStackRoot`中找到有关该线程的堆栈帧的信息，并可以使用它来继续遍历堆栈帧。

## AsyncStackRoot

`AsyncStackRoot`包含如下内容

```c
struct AsyncStackRoot{
// Pointer to the currently-active AsyncStackFrame for this event
  // loop or callback invocation. May be null if this event loop is
  // not currently executing any async operations.
  //
  // This is atomic primarily to enforce visibility of writes to the
  // AsyncStackFrame that occur before the topFrame in other processes,
  // such as profilers/debuggers that may be running concurrently
  // with the current thread.
  std::atomic<AsyncStackFrame*> topFrame{nullptr};

  // Pointer to the next event loop context lower on the current
  // thread's stack.
  // This is nullptr if this is not a nested call to an event loop.
  AsyncStackRoot* nextRoot = nullptr;

  // Pointer to the stack-frame and return-address of the function
  // call that registered this AsyncStackRoot on the current thread.
  // This is generally the stack-frame responsible for executing async
  // callbacks (typically an event-loop).
  // Anything prior to this frame on the stack in the current thread
  // is potentially unrelated to the call-chain of the current async-stack.
  //
  // Typically initialised with FOLLY_ASYNC_STACK_FRAME_POINTER() or
  // setStackFrameContext().
  void* stackFramePtr = nullptr;

  // Typically initialise with FOLLY_ASYNC_STACK_RETURN_ADDRESS() or
  // setStackFrameContext().
  void* returnAddress = nullptr;
}
```

`topFrame`指向事件循环或者回调调用中当前正在执行的栈帧。

`nextRoot`指向当前线程堆栈上下一个事件循环上下文的指针。

`stackFramePtr`指向在当前线程上注册此 `AsyncStackRoot` 的函数调用的堆栈帧和返回地址的指针。这通常是负责执行异步回调（通常是事件循环）的堆栈框架。初始化该值的典型方法为`FOLLY_ASYNC_STACK_FRAME_POINTER()`或者`setStackFrameContext()`。

`returnAddress`，通过`FOLLY_ASYNC_STACK_RETURN_ADDRESS()`或者`setStackFrameContext()`初始化。其中`FOLLY_ASYNC_STACK_RETURN_ADDRESS`方法为：

```c
#define FOLLY_ASYNC_STACK_RETURN_ADDRESS() __builtin_return_address(0)
```

这同样是编译器内建方法，作用是获取调研函数的返回地址。

## 线程栈与异步栈

`AsyncStackRoot`和`AsyncStackRoot`将普通线程栈和异步栈串连起来，其结构大致如下：

```
//      Current Thread Stack
//      ====================
// +------------------------------------+ <--- current top of stack
// | Normal Stack Frame                 |
// | - stack-base-pointer  ---.         |
// | - return-address         |         |          Thread Local Storage
// |                          |         |          ====================
// +--------------------------V---------+
// |         ...                        |     +-------------------------+
// |                          :         |     | - currentStackRoot  -.  |
// |                          :         |     |                      |  |
// +--------------------------V---------+     +----------------------|--+
// | Normal Stack Frame                 |                            |
// | - stack-base-pointer  ---.         |                            |
// | - return-address         |      .-------------------------------`
// |                          |      |  |
// +--------------------------V------|--+
// | Active Async Operation          |  |
// | (Callback or Coroutine)         |  |            Heap Allocated
// | - stack-base-pointer  ---.      |  |            ==============
// | - return-address         |      |  |
// | - pointer to async state | --------------> +-------------------------+
// |   (e.g. coro frame or    |      |  |       | Coroutine Frame         |
// |    future core)          |      |  |       | +---------------------+ |
// |                          |      |  |       | | Promise             | |
// +--------------------------V------|--+       | | +-----------------+ | |
// |   Event  / Callback             |  |   .------>| AsyncStackFrame | | |
// |   Loop     Callsite             |  |   |   | | | - parentFrame  --------.
// | - stack-base-pointer  ---.      |  |   |   | | | - instructionPtr| | |  |
// | - return-address         |      |  |   |   | | | - stackRoot -.  | | |  |
// |                          |      |  |   |   | | +--------------|--+ | |  |
// |  +--------------------+  |      |  |   |   | | ...            |    | |  |
// |  | AsyncStackRoot     |<--------`  |   |   | +----------------|----+ |  |
// |  | - topFrame   -----------------------`   | ...              |      |  |
// |  | - stackFramePtr -. |<---------------,   +------------------|------+  |
// |  | - nextRoot --.   | |  |         |   |                      |         |
// |  +--------------|---|-+  |         |   '----------------------`         |
// +-----------------|---V----V---------+       +-------------------------+  |
// |         ...     |                  |       | Coroutine Frame         |  |
// |                 |        :         |       |                         |  |
// |                 |        :         |       |  +-------------------+  |  |
// +-----------------|--------V---------+       |  | AsyncStackFrame   |<----`
// | Async Operation |                  |       |  | - parentFrame   --------.
// | (Callback/Coro) |                  |       |  | - instructionPtr  |  |  |
// |                 |        :         |       |  | - stackRoot       |  |  |
// |                 |        :         |       |  +-------------------+  |  |
// +-----------------|--------V---------+       +-------------------------+  |
// |  Event Loop /   |                  |                                    :
// |  Callback Call  |                  |                                    :
// | - frame-pointer | -------.         |                                    V
// | - return-address|        |         |
// |                 |        |         |      Another chain of potentially
// |  +--------------V-----+  |         |      unrelated AsyncStackFrame
// |  | AsyncStackRoot     |  |         |       +---------------------+
// |  | - topFrame  ---------------- - - - - >  | AsyncStackFrame     |
// |  | - stackFramePtr -. |  |         |       | - parentFrame -.    |
// |  | - nextRoot -.    | |  |         |       +----------------|----+
// |  +-------------|----|-+  |         |                        :
// |                |    |    |         |                        V
// +----------------|----V----V---------+
// |         ...    :                   |
// |                V                   |
// |                                    |
// +------------------------------------+
//
```

`AsyncStackFrame`和`AsyncStackRoot`用来串连协程调用的堆栈，使得其与线程调用栈类似。每个协程存在一个`AsyncStackFrame`，协程之间通过`AsyncStackFrame.parentFrame`串连起来。每个线程存在一个`currentThreadAsyncStackRoot`，其存储一个`AsyncStackRoot`。`AsyncStackRoot`的`topFrame`执行当前正在执行的协程栈帧。维护这些调用关系是方便进行debug。

对这些字段的维护设计如下函数：

### `pushAsyncStackFrameCallerCallee`

该函数在一个协程调用另一个协程时执行，构建调用者和被调者的关系，并且维护`stackRoot`指针（指向线程的`AsyncStackRoot`）。

其实现如下：

```c++
/*
callerFrame: 调用者栈帧
calleeFrame: 被调者栈帧
*/
inline void pushAsyncStackFrameCallerCallee(
    folly::AsyncStackFrame& callerFrame,
    folly::AsyncStackFrame& calleeFrame) noexcept {
  checkAsyncStackFrameIsActive(callerFrame);
  // 栈顶栈帧持有指向AsyncStackRoot的指针
  calleeFrame.stackRoot = callerFrame.stackRoot;
  // 被调者parentFrame指向调用者，构建调用链
  calleeFrame.parentFrame = &callerFrame;
  // 设置当前线程执行的协程栈
  calleeFrame.stackRoot->topFrame.store(
      &calleeFrame, std::memory_order_release);

  // Clearing out non-top-frame's stackRoot is not strictly necessary
  // but it may help with debugging.
  callerFrame.stackRoot = nullptr;
}
```

### `popAsyncStackFrameCallee`

该函数用于调用完成了某个协程后，将协程栈从链表中删除，其逻辑如下：

```c++
// calleeFrame表示被调的协程栈
inline void popAsyncStackFrameCallee(
    folly::AsyncStackFrame& calleeFrame) noexcept {
  checkAsyncStackFrameIsActive(calleeFrame);
  // 获取调用者的协程栈
  auto* callerFrame = calleeFrame.parentFrame;
  // 获取当前线程的AsyncStackRoot
  auto* stackRoot = calleeFrame.stackRoot;
  // 如果存在调用者，则线程的AsyncStackRoot交由其持有。
  if (callerFrame != nullptr) {
    callerFrame->stackRoot = stackRoot;
  }
  // 设置当前线程的栈顶为调用者栈帧
  stackRoot->topFrame.store(callerFrame, std::memory_order_release);

  // Clearing out non-top-frame's stackRoot is not strictly necessary
  // but it may help with debugging.
  calleeFrame.stackRoot = nullptr;
}
```

### `ScopedAsyncStackRoot`

`ScopedAsyncStackRoot`不是一个函数，而是一个类，其用来维护线程的`AsyncStackRoot`。其定义如下：

```c++
class ScopedAsyncStackRoot {
 public:
  explicit ScopedAsyncStackRoot(
      void* framePointer = FOLLY_ASYNC_STACK_FRAME_POINTER(),
      void* returnAddress = FOLLY_ASYNC_STACK_RETURN_ADDRESS()) noexcept;
  ~ScopedAsyncStackRoot();

  void activateFrame(AsyncStackFrame& frame) noexcept {
    folly::activateAsyncStackFrame(root_, frame);
  }

 private:
  AsyncStackRoot root_;
};
```

这里的`framePointer`与`returnAddress`都是之前提到的编译器函数对其赋值。其构造函数为：

```c++
static thread_local AsyncStackRootHolder currentThreadAsyncStackRoot;

ScopedAsyncStackRoot::ScopedAsyncStackRoot(
    void* framePointer, void* returnAddress) noexcept {
  root_.setStackFrameContext(framePointer, returnAddress);
  root_.nextRoot = currentThreadAsyncStackRoot.get();
  currentThreadAsyncStackRoot.set(&root_);
}
```

首先初始化一个`AsyncStackRoot`，之后挺好当前线程的`AsyncStackRoot`为新建的`root`。析构函数为：

```c++
ScopedAsyncStackRoot::~ScopedAsyncStackRoot() {
  assert(currentThreadAsyncStackRoot.get() == &root_);
  assert(root_.topFrame.load(std::memory_order_relaxed) == nullptr);
  currentThreadAsyncStackRoot.set_relaxed(root_.nextRoot);
}
```

在析构函数中还原会原来线程的`AsyncStackRoot`。

成员函数`activateFrame`方法为：

```c++
inline void activateAsyncStackFrame(
    folly::AsyncStackRoot& root, folly::AsyncStackFrame& frame) noexcept {
  assert(tryGetCurrentAsyncStackRoot() == &root);
  root.setTopFrame(frame);
}
```

设置当前`root`的栈顶帧。

该类是在一个线程上新起一个协程方法时被调用，folly中目前主要是`resumeCoroutineWithNewAsyncStackRoot`方法使用，其实现如下:

```c++
FOLLY_NOINLINE void resumeCoroutineWithNewAsyncStackRoot(
    coro::coroutine_handle<> h, folly::AsyncStackFrame& frame) noexcept {
  detail::ScopedAsyncStackRoot root;
  root.activateFrame(frame);
  h.resume();
}
```

该函数的意思是使用一个新的`AsyncStackRoot`来恢复执行一个协程。并且该协程与当前线程中的协程栈没有关系，因此需要维护一个新的`AsyncStackRoot`，在该协程调用完成之后，再恢复原来的`AsyncStackRoot`。

## CancellationToken

`CancellationToken`用于向函数或者操作进行信息传递，用于取消操作。其定义如下：

```c
// A CancellationToken is an object that can be passed into an function or
// operation that allows the caller to later request that the operation be
// cancelled.
//
// A CancellationToken object can be obtained by calling the .getToken()
// method on a CancellationSource or by copying another CancellationToken
// object. All CancellationToken objects obtained from the same original
// CancellationSource object all reference the same underlying cancellation
// state and will all be cancelled together.
//
// If your function needs to be cancellable but does not need to request
// cancellation then you should take a CancellationToken as a parameter.
// If your function needs to be able to request cancellation then you
// should instead take a CancellationSource as a parameter.
class CancellationToken {
 public:
  // Constructs to a token that can never be cancelled.
  //
  // Pass a default-constructed CancellationToken into an operation that
  // you never intend to cancel. These objects are very cheap to create.
  CancellationToken() noexcept = default;

  // Construct a copy of the token that shares the same underlying state.
  CancellationToken(const CancellationToken& other) noexcept;
  CancellationToken(CancellationToken&& other) noexcept;

  CancellationToken& operator=(const CancellationToken& other) noexcept;
  CancellationToken& operator=(CancellationToken&& other) noexcept;

  // Query whether someone has called .requestCancellation() on an instance
  // of CancellationSource object associated with this CancellationToken.
  bool isCancellationRequested() const noexcept;

  // Query whether this CancellationToken can ever have cancellation requested
  // on it.
  //
  // This will return false if the CancellationToken is not associated with a
  // CancellationSource object. eg. because the CancellationToken was
  // default-constructed, has been moved-from or because the last
  // CancellationSource object associated with the underlying cancellation state
  // has been destroyed and the operation has not yet been cancelled and so
  // never will be.
  //
  // Implementations of operations may be able to take more efficient code-paths
  // if they know they can never be cancelled.
  bool canBeCancelled() const noexcept;

  // Obtain a CancellationToken linked to any number of other
  // CancellationTokens.
  //
  // This token will have cancellation requested when any of the passed-in
  // tokens do.
  // This token is cancellable if any of the passed-in tokens are at the time of
  // construction.
  template <typename... Ts>
  static CancellationToken merge(Ts&&... tokens);

  void swap(CancellationToken& other) noexcept;

  friend bool operator==(
      const CancellationToken& a, const CancellationToken& b) noexcept;

 private:
  friend class CancellationCallback;
  friend class CancellationSource;

  explicit CancellationToken(detail::CancellationStateTokenPtr state) noexcept;

  detail::CancellationStateTokenPtr state_;
};
```

其需要配合`CancellationSource`使用。这里不展开介绍，核心是`CancellationSource`负责管理`cancel`逻辑,其存在`requestCancellation`和`getToken`两个核心接口。其中`requestCancellation`用于设置取消逻辑（`CancellationToken`不能设置取消，只能判断是否被取消），`getToken`用于生成`CancellationToken`。所以通过同一个`CancellationSource`生成的`CancellationToken`被统一管理，当`CancellationSource`被设置cancel状态时，所以的`CancellationToken`都被置为cancel状态。

其中存在`merge`接口，其输入是多个`CancellationToken`，并生成一个新的`CancellationToken`。这里的逻辑是，聚合多个`CancellationToken`，有应该被置为cancel状态时，新的这个cancel就会被置为cancel状态。其内部实现是新建了一个`CancellationSource`，将其与参数中的`CancellationToken`对应的`CancellationSource`绑定，并设置回调函数。

介绍完了成员变量的类型，我们再来看对应使用该类作为`promise_type`的协程来说，其执行逻辑。

## 分配`Coroutine state`

`TaskPromiseBase`自定义了分配`Coroutine state`的函数。

```c++
  static void* operator new(std::size_t size) {
    return ::folly_coro_async_malloc(size);
  }

  static void operator delete(void* ptr, std::size_t size) {
    ::folly_coro_async_free(ptr, size);
  }

FOLLY_NOINLINE
void* folly_coro_async_malloc(std::size_t size) {
  auto p = folly::operator_new(size);

  // Add this after the call to prevent the compiler from
  // turning the call to operator new() into a tailcall.
  folly::compiler_must_not_elide(p);

  return p;
}

FOLLY_NOINLINE
void folly_coro_async_free(void* ptr, std::size_t size) {
  folly::operator_delete(ptr, size);

  // Add this after the call to prevent the compiler from
  // turning the call to operator delete() into a tailcall.
  folly::compiler_must_not_elide(size);
}


struct compiler_must_not_elide_fn {
  template <typename T>
  FOLLY_ALWAYS_INLINE void operator()(T const& t) const noexcept;
};
FOLLY_INLINE_VARIABLE constexpr compiler_must_not_elide_fn
    compiler_must_not_elide{};

template <typename T>
FOLLY_ALWAYS_INLINE void compiler_must_not_elide_fn::operator()(
    T const& t) const noexcept {
  using i = detail::compiler_must_force_indirect<T>;
  detail::compiler_must_not_elide(t, i{});
}

template <typename T>
FOLLY_ALWAYS_INLINE void compiler_must_not_elide(T const& t, std::false_type) {
  // the "r" constraint forces the compiler to make the value available in a
  // register to the asm block, which means that it must first have been
  // computed or loaded
  //
  // used for small trivial values which the compiler will put into registers
  //
  // avoided for pointers to avoid fallout in calling code which mistakenly
  // applies the hint to the address of a value but not to the value itself
  asm volatile("" : : "r"(t));
}

template <typename T>
FOLLY_ALWAYS_INLINE void compiler_must_not_elide(T const& t, std::true_type) {
  // tells the compiler that the asm block will read the value from memory,
  // and that in addition it might read or write from any memory location
  //
  // if the memory clobber could be split into input and output, that would be
  // preferrable
  asm volatile("" : : "m"(t) : "memory");
}
```

其`new`使用的是`__builtin_operator_new`。`delete`使用的是`__builtin_operator_delete`。这里后面的函数主要作用是进行尾调用优化。具体可以参数[尾调用](https://zh.wikipedia.org/zh-hans/%E5%B0%BE%E8%B0%83%E7%94%A8)。

## 懒加载

`TaskPromiseBase`被默认构造。之后会获取函数返回值，这里不是在`TaskPromiseBase`中实现，而是在其派生类中实现，这里不做介绍。

当创建完成返回值后，会执行`initial_suspend`判断，来决定是否可以立即执行协程。其实现为懒加载，即始终不会立即执行。：

```c++
suspend_always initial_suspend() noexcept { return {}; }
```

## `co_await`时获取`awaitable`和`awaiter`

当协程内执行`co_await`时，会调用`await_transform`方法，这里实现了一系列的方法：

```c++
template <typename Awaitable>
  auto await_transform(Awaitable&& awaitable) {
    bypassExceptionThrowing_ =
        bypassExceptionThrowing_ == BypassExceptionThrowing::REQUESTED
        ? BypassExceptionThrowing::ACTIVE
        : BypassExceptionThrowing::INACTIVE;

    return folly::coro::co_withAsyncStack(folly::coro::co_viaIfAsync(
        executor_.get_alias(),
        folly::coro::co_withCancellation(
            cancelToken_, static_cast<Awaitable&&>(awaitable))));
  }

  template <typename Awaitable>
  auto await_transform(NothrowAwaitable<Awaitable>&& awaitable) {
    bypassExceptionThrowing_ = BypassExceptionThrowing::REQUESTED;
    return await_transform(awaitable.unwrap());
  }
  
  // 只针对co_current_executor_t，后面会介绍，获取协程的executor_，不会被挂起
  auto await_transform(co_current_executor_t) noexcept {
    return ready_awaitable<folly::Executor*>{executor_.get()};
  }

  // 只针对co_current_cancellation_token_t，后面会介绍，获取协程的cancelToken_，不会被挂起
  auto await_transform(co_current_cancellation_token_t) noexcept {
    return ready_awaitable<const folly::CancellationToken&>{cancelToken_};
  }
```

其核心是第一个函数，即实际调用的是`folly::coro::co_withAsyncStack`和`co_viaIfAsync`以及`co_withCancellation`方法。

对于`Task`来说，这几个方法都重写了，这里看一下这几个函数的默认方法。

### co_withCancellation

其方法核心是将cancel与协程任务绑定，对于Task相关结构来说，其绑定是没有问题的，但是默认情况下不清楚awaitable类型没办法绑定，因此默认的改函数实现是：

```c++
FOLLY_DEFINE_CPO(detail::adl::WithCancellationFunction, co_withCancellation)

struct WithCancellationFunction {
  template <typename Awaitable>
  auto operator()(
      const folly::CancellationToken& cancelToken, Awaitable&& awaitable) const
      noexcept(
          noexcept(co_withCancellation(cancelToken, (Awaitable &&) awaitable)))
          -> decltype(co_withCancellation(
              cancelToken, (Awaitable &&) awaitable)) {
    return co_withCancellation(cancelToken, (Awaitable &&) awaitable);
  }

  template <typename Awaitable>
  auto operator()(folly::CancellationToken&& cancelToken, Awaitable&& awaitable)
      const noexcept(noexcept(co_withCancellation(
          std::move(cancelToken), (Awaitable &&) awaitable)))
          -> decltype(co_withCancellation(
              std::move(cancelToken), (Awaitable &&) awaitable)) {
    return co_withCancellation(
        std::move(cancelToken), (Awaitable &&) awaitable);
  }
};

template <typename Awaitable>
Awaitable&& co_withCancellation(
    const folly::CancellationToken&, Awaitable&& awaitable) noexcept {
  return (Awaitable &&) awaitable;
}
```

即默认情况下直接返回`awaitable`。

这里，`task`，`TaskWithExecutor`有实现该方法（后面会介绍该类），其实现如下：

```c++
  friend Task co_withCancellation(
      folly::CancellationToken cancelToken, Task&& task) noexcept {
    DCHECK(task.coro_);
    task.coro_.promise().setCancelToken(std::move(cancelToken));
    return std::move(task);
  }


  friend TaskWithExecutor co_withCancellation(
      folly::CancellationToken cancelToken, TaskWithExecutor&& task) noexcept {
    DCHECK(task.coro_);
    task.coro_.promise().setCancelToken(std::move(cancelToken));
    return std::move(task);
  }
```

可以看到这里是直接将cancel与协程的promise绑定。

### co_viaIfAsync

`co_viaIfAsync`的作用是保证调用者协程能够始终在指定的`executor`(线程池)中执行。

其具体实现是对协程再通过框架封装一层框架定义的协程，在框架定义的这一层来实现当前协程被`suspend`后和在其恢复时依然在原线程池上执行。其实现较为复杂，可以先阅读后面部分，对`coro`协程有整体了解后再回来看其具体实现。

使用一个简单例子来进行描述：

```c++
Task<void> funca() {
	co_return;
}

Task<void> funcb() {
  xxx;
  co_await funca().scheduleOn(executor1);
}

int main() {
  folly::coro::blockingWait(b().scheduleOn(exectutor2));
}
```

在上面的伪代码中，我们希望`funcb`在线程池`exectutor2`中执行，但是希望`funca`在线程池`executor1`中执行。在执行`funcb`时，当其被挂起后，执行`funca`，由于`funca`被指定在线程池`executor1`中执行，当`funca`执行完成后，恢复`funcb`的执行时，如果没有`co_viaIfAsync`的协助，`funcb`剩下的部分也将直接在`executor1`中被执行，通过`co_viaIfAsync`，可以保证`funcb`均在指定线程池中执行。

下面我们来详细了解其实现逻辑。

```c++
FOLLY_DEFINE_CPO(detail::adl::ViaIfAsyncFunction, co_viaIfAsync)
  
#define FOLLY_DEFINE_CPO(Type, Name) \
  namespace folly_cpo__ {            \
  inline constexpr Type Name{};      \
  }                                  \
  using namespace folly_cpo__;
  
struct ViaIfAsyncFunction {
  template <typename Awaitable>
  auto operator()(folly::Executor::KeepAlive<> executor, Awaitable&& awaitable)
      const noexcept(noexcept(co_viaIfAsync(
          std::move(executor), static_cast<Awaitable&&>(awaitable))))
          -> decltype(co_viaIfAsync(
              std::move(executor), static_cast<Awaitable&&>(awaitable))) {
    return co_viaIfAsync(
        std::move(executor), static_cast<Awaitable&&>(awaitable));
  }
};


auto co_viaIfAsync(
    folly::Executor::KeepAlive<> executor,
    SemiAwaitable&&
        awaitable) noexcept(noexcept(static_cast<SemiAwaitable&&>(awaitable)
                                         .viaIfAsync(std::move(executor))))
    -> decltype(static_cast<SemiAwaitable&&>(awaitable).viaIfAsync(
        std::move(executor))) {
  return static_cast<SemiAwaitable&&>(awaitable).viaIfAsync(
      std::move(executor));
}

template <
    typename Awaitable,
    std::enable_if_t<
        is_awaitable_v<Awaitable> && !HasViaIfAsyncMethod<Awaitable>::value,
        int> = 0>
auto co_viaIfAsync(folly::Executor::KeepAlive<> executor, Awaitable&& awaitable)
    -> ViaIfAsyncAwaitable<Awaitable> {
  return ViaIfAsyncAwaitable<Awaitable>{
      std::move(executor), static_cast<Awaitable&&>(awaitable)};
}
```

如果用户实现了自己的`co_viaIfAsync`方法则优先调用用户自己的方法。之后如果用户实现了`awaitable`的`viaIfAsync`方法，则会调用该方法，否则，调用`ViaIfAsyncAwaitable`。下面主要看`ViaIfAsyncAwaitable`的实现。

```c++
class ViaIfAsyncAwaitable {
 public:
  explicit ViaIfAsyncAwaitable(
      folly::Executor::KeepAlive<> executor,
      Awaitable&&
          awaitable) noexcept(std::is_nothrow_move_constructible<Awaitable>::
                                  value)
      : executor_(std::move(executor)),
        awaitable_(static_cast<Awaitable&&>(awaitable)) {}

  ViaIfAsyncAwaiter<false, Awaitable> operator co_await() && {
    return ViaIfAsyncAwaiter<false, Awaitable>{
        std::move(executor_), static_cast<Awaitable&&>(awaitable_)};
  }

  friend StackAwareViaIfAsyncAwaitable<Awaitable> tag_invoke(
      cpo_t<co_withAsyncStack>, ViaIfAsyncAwaitable&& self) {
    return StackAwareViaIfAsyncAwaitable<Awaitable>{
        std::move(self.executor_), static_cast<Awaitable&&>(self.awaitable_)};
  }

 private:
  folly::Executor::KeepAlive<> executor_;
  Awaitable awaitable_;
};
```

当用户直接`co_await folly::coro::co_viaIfAsync(executor_.get_alias(),awaitable)`时，实际执行的就变成了`co_await ViaIfAsyncAwaitable`了（这里假设协程的`promise_type`没有`await_transform`方法，这个逻辑一般是直接在`await_transform`中返回`ViaIfAsyncAwaitable`）。之后通过`ViaIfAsyncAwaitable::co_await`方法获取`awaiter`，该方法只创建一个`ViaIfAsyncAwaiter`。`ViaIfAsyncAwaiter`即为这里实际的`awaiter`。

下面看一下`ViaIfAsyncAwaiter`的实现：

```c++
template <bool IsCallerAsyncStackAware, typename Awaitable>
class ViaIfAsyncAwaiter {
  using Awaiter = folly::coro::awaiter_type_t<Awaitable>;
  using CoroutineType = detail::ViaCoroutine<false>;
  using CoroutinePromise = typename CoroutineType::promise_type;
  using WrapperHandle = coroutine_handle<CoroutinePromise>;

  using await_suspend_result_t =
      decltype(std::declval<Awaiter&>().await_suspend(
          std::declval<WrapperHandle>()));

 public:
  explicit ViaIfAsyncAwaiter(
      folly::Executor::KeepAlive<> executor, Awaitable&& awaitable)
      : viaCoroutine_(CoroutineType::create(std::move(executor))),
        awaiter_(
            folly::coro::get_awaiter(static_cast<Awaitable&&>(awaitable))) {}

 private:
  CoroutineType viaCoroutine_;
  Awaiter awaiter_;
};
```

在创建`ViaIfAsyncAwaiter`时会调用`CoroutineType::create(std::move(executor))`方法和`folly::coro::get_awaiter(static_cast<Awaitable&&>(awaitable))`方法。其中第一个方法的实现为：

```c++
  static ViaCoroutine createImpl() { co_return; }
  static ViaCoroutine create(folly::Executor::KeepAlive<> executor) {
    ViaCoroutine coroutine = createImpl();
    coroutine.setExecutor(std::move(executor));
    return coroutine;
  }
```

可以看到在执行第一个方法的时候，调用的是一个空的协程，这里就完成了对原来协程的一层封装，相当于在原协程上又封装了一层协程。该协程对应的`ViaCoroutine`结构为：

```c++
class ViaCoroutinePromiseBase {
 public:
  static void* operator new(std::size_t size) {
    return ::folly_coro_async_malloc(size);
  }

  static void operator delete(void* ptr, std::size_t size) {
    ::folly_coro_async_free(ptr, size);
  }

  suspend_always initial_suspend() noexcept { return {}; }

  void return_void() noexcept {}

  [[noreturn]] void unhandled_exception() noexcept {
    folly::assume_unreachable();
  }

  void setExecutor(folly::Executor::KeepAlive<> executor) noexcept {
    executor_ = std::move(executor);
  }

  void setContinuation(ExtendedCoroutineHandle continuation) noexcept {
    continuation_ = continuation;
  }

  void setAsyncFrame(folly::AsyncStackFrame& frame) noexcept {
    asyncFrame_ = &frame;
  }

  void setRequestContext(
      std::shared_ptr<folly::RequestContext> context) noexcept {
    context_ = std::move(context);
  }

 protected:
  void scheduleContinuation() noexcept {
    // Pass the coroutine's RequestContext to Executor::add(), in case the
    // Executor implementation wants to know what runs on it (e.g. for stats).
    RequestContextScopeGuard contextScope{context_};

    executor_->add([this]() noexcept { this->executeContinuation(); });
  }

 private:
  void executeContinuation() noexcept {
    RequestContextScopeGuard contextScope{std::move(context_)};
    if (asyncFrame_ != nullptr) {
      folly::resumeCoroutineWithNewAsyncStackRoot(
          continuation_.getHandle(), *asyncFrame_);
    } else {
      continuation_.resume();
    }
  }

 protected:
  virtual ~ViaCoroutinePromiseBase() = default;

  folly::Executor::KeepAlive<> executor_;
  ExtendedCoroutineHandle continuation_;
  folly::AsyncStackFrame* asyncFrame_ = nullptr;
  std::shared_ptr<RequestContext> context_;
};

template <bool IsStackAware>
class ViaCoroutine {
 public:
  class promise_type final : public ViaCoroutinePromiseBase,
                             public ExtendedCoroutinePromiseImpl<promise_type> {
    struct FinalAwaiter;

    FinalAwaiter final_suspend() noexcept { return {}; }

    template <
        bool IsStackAware2 = IsStackAware,
        std::enable_if_t<IsStackAware2, int> = 0>
    folly::AsyncStackFrame& getAsyncFrame() noexcept {
      DCHECK(this->asyncFrame_ != nullptr);
      return *this->asyncFrame_;
    }

    std::pair<ExtendedCoroutineHandle, AsyncStackFrame*> getErrorHandle(
        exception_wrapper& ex) override {
      auto [handle, frame] = continuation_.getErrorHandle(ex);
      setContinuation(handle);
      if (frame && IsStackAware) {
        asyncFrame_ = frame;
      }
      return {promise_type::getHandle(), nullptr};
    }
  };

  ViaCoroutine(ViaCoroutine&& other) noexcept
      : coro_(std::exchange(other.coro_, {})) {}

  ~ViaCoroutine() {
    if (coro_) {
      coro_.destroy();
    }
  }

  static ViaCoroutine create(folly::Executor::KeepAlive<> executor) {
    ViaCoroutine coroutine = createImpl();
    coroutine.setExecutor(std::move(executor));
    return coroutine;
  }

  void setExecutor(folly::Executor::KeepAlive<> executor) noexcept {
    coro_.promise().setExecutor(std::move(executor));
  }

  void setContinuation(ExtendedCoroutineHandle continuation) noexcept {
    coro_.promise().setContinuation(continuation);
  }

  void setAsyncFrame(folly::AsyncStackFrame& frame) noexcept {
    coro_.promise().setAsyncFrame(frame);
  }

  void destroy() noexcept {
    if (coro_) {
      std::exchange(coro_, {}).destroy();
    }
  }

  void saveContext() noexcept {
    coro_.promise().setRequestContext(folly::RequestContext::saveContext());
  }

  coroutine_handle<promise_type> getHandle() noexcept { return coro_; }

 private:
  explicit ViaCoroutine(coroutine_handle<promise_type> coro) noexcept
      : coro_(coro) {}

  static ViaCoroutine createImpl() { co_return; }

  coroutine_handle<promise_type> coro_;
};
```

当创建`ViaIfAsyncAwaiter`时，首先会创建`ViaCoroutine::promise_type`。之后调用`ViaCoroutine::promise_type::get_return_object`方法创建`ViaCoroutine`(这时拿到当前协程的`coroutine_handle`)。之后挂起。

可以看到创建`ViaIfAsyncAwaiter`会起一个新的协程，并且`suspend`在对`viaCoroutine_`的赋值上。

创建`ViaIfAsyncAwaiter`的第二个函数执行逻辑为：

```c++
template <
    typename Awaitable,
    std::enable_if_t<
        folly::Conjunction<
            is_awaiter<Awaitable>,
            folly::Negation<detail::_has_free_operator_co_await<Awaitable>>,
            folly::Negation<detail::_has_member_operator_co_await<Awaitable>>>::
            value,
        int> = 0>
Awaitable& get_awaiter(Awaitable&& awaitable) {
  return awaitable;
}

template <
    typename Awaitable,
    std::enable_if_t<
        detail::_has_member_operator_co_await<Awaitable>::value,
        int> = 0>
decltype(auto) get_awaiter(Awaitable&& awaitable) {
  return static_cast<Awaitable&&>(awaitable).operator co_await();
}

template <
    typename Awaitable,
    std::enable_if_t<
        folly::Conjunction<
            detail::_has_free_operator_co_await<Awaitable>,
            folly::Negation<detail::_has_member_operator_co_await<Awaitable>>>::
            value,
        int> = 0>
decltype(auto) get_awaiter(Awaitable&& awaitable) {
  return operator co_await(static_cast<Awaitable&&>(awaitable));
}
```

这里是根据`Awaitable`获取到`awaiter`，其实现与协程实现一致，根据是否存在`co_await`函数来决定执行逻辑。

到此，完成了`ViaIfAsyncAwaiter`的创建于获取。只会执行`co_await`对`awaiter`的操作。

首先执行`await_ready`函数，其实现如下：

```c++
decltype(auto) await_ready() noexcept(noexcept(awaiter_.await_ready())) {
    return awaiter_.await_ready();
  }
```

直接根据`co_awaiter coro`的那个协程（被调协程）来决定是否ready，如果已经ready了，直接执行。

正常情况下ready都是false，此时会调用`await_suspend`来触发实际执行。其逻辑为：

```c++
template <typename Promise>
  auto await_suspend(coroutine_handle<Promise> continuation) noexcept(noexcept(
      std::declval<Awaiter&>().await_suspend(std::declval<WrapperHandle>())))
      -> await_suspend_result_t {
    viaCoroutine_.setContinuation(continuation);

    if constexpr (!detail::is_coroutine_handle_v<await_suspend_result_t>) {
      viaCoroutine_.saveContext();
    }

    if constexpr (IsCallerAsyncStackAware) {
      auto& asyncFrame = continuation.promise().getAsyncFrame();
      auto& stackRoot = *asyncFrame.getStackRoot();

      viaCoroutine_.setAsyncFrame(asyncFrame);

      folly::deactivateAsyncStackFrame(asyncFrame);

      // Reactivate the stack-frame before we resume.
      auto rollback =
          makeGuard([&] { activateAsyncStackFrame(stackRoot, asyncFrame); });
      if constexpr (std::is_same_v<await_suspend_result_t, bool>) {
        if (!awaiter_.await_suspend(viaCoroutine_.getHandle())) {
          return false;
        }
        rollback.dismiss();
        return true;
      } else if constexpr (std::is_same_v<await_suspend_result_t, void>) {
        awaiter_.await_suspend(viaCoroutine_.getHandle());
        rollback.dismiss();
        return;
      } else {
        auto ret = awaiter_.await_suspend(viaCoroutine_.getHandle());
        rollback.dismiss();
        return ret;
      }
    } else {
      return awaiter_.await_suspend(viaCoroutine_.getHandle());
    }
  }
```

这里`IsCallerAsyncStackAware`是false，可以不考虑该逻辑。参数的`continuation`为调用者协程的`coroutine_handle`，即我们需要保证执行位置的协程。将`continuation`存储到`viaCoroutine_`，之后执行`return awaiter_.await_suspend(viaCoroutine_.getHandle());`。将架构封装的这层协程的`coroutine_handle`作为参数执行被调协程。这时，正常来说会立即执行被调协程，并且在被调协程执行完成之后，会唤醒调用者协程，这里的调用者协程就是架构封装的这一层协程。

这时才会执行完成创建`ViaIfAsyncAwaiter`时的`createImpl`函数。在执行完成该函数后析构该协程前，将会执行`co_await ViaIfAsyncAwaiter::promise_type::final_suspend`，这里将返回`ViaIfAsyncAwaiter::promise_type::FinalAwaiter`，其定义如下：

```c++
struct FinalAwaiter {
bool await_ready() noexcept { return false; }

// This code runs immediately after the inner awaitable resumes its fake
// continuation, and it schedules the real continuation on the awaiter's
// executor
FOLLY_CORO_AWAIT_SUSPEND_NONTRIVIAL_ATTRIBUTES void await_suspend(
    coroutine_handle<promise_type> h) noexcept {
  auto& promise = h.promise();
  if (!promise.context_) {
    promise.setRequestContext(RequestContext::saveContext());
  }

  if constexpr (IsStackAware) {
    folly::deactivateAsyncStackFrame(promise.getAsyncFrame());
  }

  promise.scheduleContinuation();
}

[[noreturn]] void await_resume() noexcept { folly::assume_unreachable(); }
};
```

这里`await_suspend`的参数是架构这层协程的`coroutine_handle`。

其首先获取`promise`，设置`RequestContext`，之后执行`scheduleContinuation`，其实现如下：

```c++
void scheduleContinuation() noexcept {
  // Pass the coroutine's RequestContext to Executor::add(), in case the
  // Executor implementation wants to know what runs on it (e.g. for stats).
  RequestContextScopeGuard contextScope{context_};

  executor_->add([this]() noexcept { this->executeContinuation(); });
}

void executeContinuation() noexcept {
  RequestContextScopeGuard contextScope{std::move(context_)};
  if (asyncFrame_ != nullptr) {
    folly::resumeCoroutineWithNewAsyncStackRoot(
        continuation_.getHandle(), *asyncFrame_);
  } else {
    continuation_.resume();
  }
}
```

可以看到，其实现就是把`continuation_`的`resume`添加到指定线程池中执行，这里的`continuation_`即为调用者协程的`coroutine_handle`，即我们需要保证执行位置的协程。

至此完成了保证协程在指定线程池上执行的全部逻辑，可以看到，整体实现相当精妙，里面使用了C++的很多特性，值得深入研究。

### co_withAsyncStack

`co_withAsyncStack`与`co_viaIfAsync`的作用类似，其被用于`await_transform()`内部，用于在当前协程`suspend`时将当前协程的调用栈信息暂存起来，在`resume`时恢复，其实现也于`co_viaIfAsync`类似，架构封装了一层协程实现，这里不过多介绍，详细信息可以看相关代码。

这里特别注意的是，对于希望自己维护调用栈关系的`Awaitables`，可以定义`tag_invoke`函数来自己控制，类似如下代码：

```c++
class MyAwaitable {
     friend MyAwaitable&& tag_invoke(
         cpo_t<folly::coro::co_withAsyncStack>, MyAwaitable&& awaitable) {
       return std::move(awaitable);
     }

     ...
};
```

## 对`awaiter`的处理

获取到`awaiter`后，会调用其`await_ready`，`await_suspend`以及最后的`await_resume`作为`co_await`的返回结果。这里都不在`TaskPromiseBase`的控制范畴，会在后续部分详细介绍。

## 协程结束

协程结束时，如果协程执行过程中跑出了异常，则会先执行`unhandled_exception`，这里其定义不在`TaskPromiseBase`而在`TaskPromise`，其定义为：

```c++
void unhandled_exception() noexcept {
    result_.emplaceException(exception_wrapper{std::current_exception()});
  }
```

即设置对应`result_`为异常。

如果没有异常(有异常也执行)，则直接执行`co_await TaskPromiseBase::final_suspend()`。这里我们看一下其定义：

```c++
class FinalAwaiter {
 public:
  bool await_ready() noexcept { return false; }

  template <typename Promise>
  FOLLY_CORO_AWAIT_SUSPEND_NONTRIVIAL_ATTRIBUTES coroutine_handle<>
  await_suspend(coroutine_handle<Promise> coro) noexcept {
    auto& promise = coro.promise();
    // If the continuation has been exchanged, then we expect that the
    // exchanger will handle the lifetime of the async stack. See
    // ScopeExitTaskPromise's FinalAwaiter for more details.
    //
    // This is a bit untidy, and hopefully something we can replace with
    // a virtual wrapper over coroutine_handle that handles the pop for us.
    if (promise.ownsAsyncFrame_) {
      folly::popAsyncStackFrameCallee(promise.asyncFrame_);
    }
    if (promise.result_.hasException()) {
      auto [handle, frame] =
          promise.continuation_.getErrorHandle(promise.result_.exception());
      return handle.getHandle();
    }
    return promise.continuation_.getHandle();
  }

  [[noreturn]] void await_resume() noexcept { folly::assume_unreachable(); }
};
```

`await_ready()`返回false表示为懒加载。因此`co_await`会执行`await_suspend`，这里`coro`是当前协程的`coroutine_handle`。

其逻辑是首先获取当前协程的promise，这里就是`TaskPromise`。调用`popAsyncStackFrameCallee`将当前协程从协程栈中出栈。如果存在异常，则调用continuation_的异常处理函数，并返回`coroutine_handle`。

当没有异常时，返回`continuation_`的`coroutine_handle`。这里的`continuation_`是调用者协程的`coroutine_handle`，其会在后续介绍的`awaiter`的`await_suspend`中赋值。通过这个逻辑，实现了被调者完成处理后唤醒调用者。

协程结束时需要析构`handler`，folly的实现是析构交由`awaiter`来实现。

# TaskPromise

`Task`的`promise_type`并不是直接使用`TaskPromiseBase`而是使用的`TaskPromise`，其在`TaskPromiseBase`基础上增加一些协程的相关函数。

其定义如下：

```c++
template <typename T>
class TaskPromise final : public TaskPromiseBase,
                          public ExtendedCoroutinePromiseImpl<TaskPromise<T>> {
 public:
  static_assert(
      !std::is_rvalue_reference_v<T>,
      "Task<T&&> is not supported. "
      "Consider using Task<T> or Task<std::unique_ptr<T>> instead.");
  friend class TaskPromiseBase;

  using StorageType = detail::lift_lvalue_reference_t<T>;

  TaskPromise() noexcept = default;

  Task<T> get_return_object() noexcept;

  void unhandled_exception() noexcept {
    result_.emplaceException(exception_wrapper{std::current_exception()});
  }

  template <typename U = T>
  void return_value(U&& value) {
    if constexpr (std::is_same_v<remove_cvref_t<U>, Try<StorageType>>) {
      DCHECK(value.hasValue() || value.hasException());
      result_ = static_cast<U&&>(value);
    } else if constexpr (
        std::is_same_v<remove_cvref_t<U>, Try<void>> &&
        std::is_same_v<remove_cvref_t<T>, Unit>) {
      // special-case to make task -> semifuture -> task preserve void type
      DCHECK(value.hasValue() || value.hasException());
      result_ = static_cast<Try<Unit>>(static_cast<U&&>(value));
    } else {
      static_assert(
          std::is_convertible<U&&, StorageType>::value,
          "cannot convert return value to type T");
      result_.emplace(static_cast<U&&>(value));
    }
  }

  Try<StorageType>& result() { return result_; }

  using TaskPromiseBase::await_transform;

  std::pair<ExtendedCoroutineHandle, AsyncStackFrame*> getErrorHandle(
      exception_wrapper& ex) override {
    if (bypassExceptionThrowing_ == BypassExceptionThrowing::ACTIVE) {
      auto finalAwaiter = yield_value(co_error(std::move(ex)));
      DCHECK(!finalAwaiter.await_ready());
      return {
          finalAwaiter.await_suspend(
              coroutine_handle<TaskPromise>::from_promise(*this)),
          // finalAwaiter.await_suspend pops a frame
          getAsyncFrame().getParentFrame()};
    }
    return {coroutine_handle<TaskPromise>::from_promise(*this), nullptr};
  }

 private:
  Try<StorageType> result_;
};

template <typename Promise>
class ExtendedCoroutinePromiseImpl : public ExtendedCoroutinePromise {
 public:
  coroutine_handle<> getHandle() final {
    return coroutine_handle<Promise>::from_promise(
        *static_cast<Promise*>(this));
  }

  std::pair<ExtendedCoroutineHandle, AsyncStackFrame*> getErrorHandle(
      exception_wrapper&) override {
    return {getHandle(), nullptr};
  }

 protected:
  ~ExtendedCoroutinePromiseImpl() = default;
};
```

这里，其主要增加了一个`Try<StorageType> result_`用来存储协程返回值。实现了异常处理函数`unhandled_exception`。获取协程返回值`get_return_object`

```c++
template <typename T>
Task<T> detail::TaskPromise<T>::get_return_object() noexcept {
  return Task<T>{coroutine_handle<detail::TaskPromise<T>>::from_promise(*this)};
}
```

用户调用`co_return`是设置协程返回值`return_value`。这里设置的返回值会是`co_await coro`的最终返回值，即`awaiter`的`await_resume`最后会返回该值。

`TaskPromise`还有一个`TaskPromise<void>`的偏例化，其主要实现了void相关的接口，这里不详细介绍。

# Task

`task`是folly coro的核心类，一般对于协程函数返回值都应该是Task。通过task，folly将协程调用链串连起来。其定义如下：

```c++
template <typename T>
class FOLLY_NODISCARD Task {
 public:
  using promise_type = detail::TaskPromise<T>;
  using StorageType = typename promise_type::StorageType;

 private:
  class Awaiter;
  using handle_t = coroutine_handle<promise_type>;

  void setExecutor(folly::Executor::KeepAlive<>&& e) noexcept {
    DCHECK(coro_);
    DCHECK(e);
    coro_.promise().executor_ = std::move(e);
  }

 public:
  Task(const Task& t) = delete;

  /// Create a Task, invalidating the original Task in the process.
  Task(Task&& t) noexcept : coro_(std::exchange(t.coro_, {})) {}

  /// @private
  // 析构时负责析构协程的handler
  ~Task() {
    if (coro_) {
      coro_.destroy();
    }
  }

  Task& operator=(Task t) noexcept {
    swap(t);
    return *this;
  }

  void swap(Task& t) noexcept { std::swap(coro_, t.coro_); }

  /// Specify the executor that this task should execute on.
  /// @param executor An Executor::KeepAlive object, which can be implicity
  /// constructed from Executor
  /// @returns a new TaskWithExecutor object, which represents the existing Task
  /// bound to an executor
  FOLLY_NODISCARD
  TaskWithExecutor<T> scheduleOn(Executor::KeepAlive<> executor) && noexcept {
    setExecutor(std::move(executor));
    DCHECK(coro_);
    return TaskWithExecutor<T>{std::exchange(coro_, {})};
  }

 private:
  friend class detail::TaskPromiseBase;
  friend class detail::TaskPromise<T>;

  class Awaiter {
   public:
    explicit Awaiter(handle_t coro) noexcept : coro_(coro) {}

    Awaiter(Awaiter&& other) noexcept : coro_(std::exchange(other.coro_, {})) {}

    Awaiter(const Awaiter&) = delete;

    ~Awaiter() {
      if (coro_) {
        coro_.destroy();
      }
    }

    bool await_ready() noexcept { return false; }

    template <typename Promise>
    FOLLY_NOINLINE auto await_suspend(
        coroutine_handle<Promise> continuation) noexcept {
      DCHECK(coro_);
      auto& promise = coro_.promise();

      promise.continuation_ = continuation;

      auto& calleeFrame = promise.getAsyncFrame();
      calleeFrame.setReturnAddress();

      if constexpr (detail::promiseHasAsyncFrame_v<Promise>) {
        auto& callerFrame = continuation.promise().getAsyncFrame();
        folly::pushAsyncStackFrameCallerCallee(callerFrame, calleeFrame);
        return coro_;
      } else {
        folly::resumeCoroutineWithNewAsyncStackRoot(coro_);
        return;
      }
    }

    T await_resume() {
      DCHECK(coro_);
      SCOPE_EXIT { std::exchange(coro_, {}).destroy(); };
      return std::move(coro_.promise().result()).value();
    }

    folly::Try<StorageType> await_resume_try() {
      DCHECK(coro_);
      SCOPE_EXIT { std::exchange(coro_, {}).destroy(); };
      return std::move(coro_.promise().result());
    }

   private:
    // This overload needed as Awaiter is returned from co_viaIfAsync() which is
    // then passed into co_withAsyncStack().
    friend Awaiter tag_invoke(
        cpo_t<co_withAsyncStack>, Awaiter&& awaiter) noexcept {
      return std::move(awaiter);
    }

    handle_t coro_;
  };

  Task(handle_t coro) noexcept : coro_(coro) {}

  handle_t coro_;
};
```

根据之前关于`promise_type`的介绍，这里当`co_await Task`时，获取到的`awaiter`是`Task::Awaiter`。

这里其`await_ready`始终返回false。一定会执行`await_suspend`，这里其参数`continuation`是调用者协程的`coroutine_handle`，而不是`co_await Task`里面的这个`Task`。

在`await_suspend`，首先设置`promise`的`continuation_`为调用者协程的`coroutine_handle`，配合上面介绍的`TaskPromiseBase::FinalAwaiter`实现被调协程完成后唤醒调用者协程。这里的`if constexpr (detail::promiseHasAsyncFrame_v<Promise>)`为true，这里的操作是维护协程的调用栈，将当前协程的调用栈追加到调用链路中。之后返回当前协程的`coroutine_handle`，这会立即执行当前协程。

当协程执行结束，会调用`await_resume`获取`co_await Task`的最终返回值，即`co_return expr`设置的值。这里实际执行的是`Try::value`。如果没有设置value，则会抛出异常，但是如果是`void`值，则不会抛出异常，这是由于`Try<void>`默认是有值的。因此对于返回`Task<void>`的协程，可以不执行`co_return`，对于其他类型的返回一定要执行`co_return expr`。

对于返回`Task<void>`的协程来说，一定要特别注意是，虽然可以不执行`co_return`，但是一定要保证函数是协程，即至少要出现`co_await`，`co_return`或者`co_yield`。由于`Task`没有将默认构造函数delete，因此如果没有出现这三个关键字，则该函数就不是协程函数，不会按照协程的方式执行（不要问我是怎么知道的...）。例如：

```c
Task<void> a() {
   std::cout<<"process func a"<<std::endl;
}
```

这就不是一个协程函数。

# TaskWithExecutor

`Task`协程默认运行在调用者线程中，但在对延迟较敏感的服务中，我们需要将不同协程执行在不同线程中，也就是一般说的`M:N`模式（`brpc`中`bthread`也是这种模式，将m个用户态线程映射到n个实际liunx线程中，m远大于n）。为提供该功能，`Task`提供接口`scheduleOn`及`TaskWithExecutor`类。

```c++
TaskWithExecutor<T> scheduleOn(Executor::KeepAlive<> executor) && noexcept {
    setExecutor(std::move(executor));
    DCHECK(coro_);
    return TaskWithExecutor<T>{std::exchange(coro_, {})};
}
```

接口提供一个线程池，设置改task执行在该线程池中。返回`TaskWithExecutor`，其定义如下：

```c++
template <typename T>
class FOLLY_NODISCARD TaskWithExecutor {
  using handle_t = coroutine_handle<detail::TaskPromise<T>>;
  using StorageType = typename detail::TaskPromise<T>::StorageType;

 public:
  /// @private
  ~TaskWithExecutor() {
    if (coro_) {
      coro_.destroy();
    }
  }

  TaskWithExecutor(TaskWithExecutor&& t) noexcept
      : coro_(std::exchange(t.coro_, {})) {}

  TaskWithExecutor& operator=(TaskWithExecutor t) noexcept {
    swap(t);
    return *this;
  }
  /// Returns the executor that the task is bound to
  folly::Executor* executor() const noexcept {
    return coro_.promise().executor_.get();
  }

  void swap(TaskWithExecutor& t) noexcept { std::swap(coro_, t.coro_); }

  /// Start eager execution of this task.
  ///
  /// This starts execution of the Task on the bound executor.
  /// @returns folly::SemiFuture<T> that will complete with the result.
  FOLLY_NOINLINE SemiFuture<lift_unit_t<StorageType>> start() && {
    folly::Promise<lift_unit_t<StorageType>> p;

    auto sf = p.getSemiFuture();

    std::move(*this).startImpl(
        [promise = std::move(p)](Try<StorageType>&& result) mutable {
          promise.setTry(std::move(result));
        },
        folly::CancellationToken{},
        FOLLY_ASYNC_STACK_RETURN_ADDRESS());

    return sf;
  }

  /// Start eager execution of the task and call the passed callback on
  /// completion
  ///
  /// This starts execution of the Task on the bound executor, and call the
  /// passed callback upon completion. The callback takes a Try<T> which
  /// represents either th value returned by the Task on success or an
  /// exeception thrown by the Task
  /// @param tryCallback a function that takes in a Try<T>
  /// @param cancelToken a CancelationToken object
  template <typename F>
  FOLLY_NOINLINE void start(
      F&& tryCallback, folly::CancellationToken cancelToken = {}) && {
    std::move(*this).startImpl(
        static_cast<F&&>(tryCallback),
        std::move(cancelToken),
        FOLLY_ASYNC_STACK_RETURN_ADDRESS());
  }

  /// Start eager execution of this task on this thread.
  ///
  /// Assumes the current thread is already on the executor associated with the
  /// Task. Refer to TaskWithExecuter::start(F&& tryCallback,
  /// folly::CancellationToken cancelToken = {}) for more information.
  template <typename F>
  FOLLY_NOINLINE void startInlineUnsafe(
      F&& tryCallback, folly::CancellationToken cancelToken = {}) && {
    std::move(*this).startInlineImpl(
        static_cast<F&&>(tryCallback),
        std::move(cancelToken),
        FOLLY_ASYNC_STACK_RETURN_ADDRESS());
  }

  /// Start eager execution of this task on this thread.
  ///
  /// Assumes the current thread is already on the executor associated with the
  /// Task. Refer to TaskWithExecuter::start() for more information.
  FOLLY_NOINLINE SemiFuture<lift_unit_t<StorageType>> startInlineUnsafe() && {
    folly::Promise<lift_unit_t<StorageType>> p;

    auto sf = p.getSemiFuture();

    std::move(*this).startInlineImpl(
        [promise = std::move(p)](Try<StorageType>&& result) mutable {
          promise.setTry(std::move(result));
        },
        folly::CancellationToken{},
        FOLLY_ASYNC_STACK_RETURN_ADDRESS());

    return sf;
  }

 private:
  template <typename F>
  void startImpl(
      F&& tryCallback,
      folly::CancellationToken cancelToken,
      void* returnAddress) && {
    coro_.promise().setCancelToken(std::move(cancelToken));
    startImpl(std::move(*this), static_cast<F&&>(tryCallback))
        .start(returnAddress);
  }

  template <typename F>
  void startInlineImpl(
      F&& tryCallback,
      folly::CancellationToken cancelToken,
      void* returnAddress) && {
    coro_.promise().setCancelToken(std::move(cancelToken));
    RequestContextScopeGuard contextScope{RequestContext::saveContext()};
    startInlineImpl(std::move(*this), static_cast<F&&>(tryCallback))
        .start(returnAddress);
  }

  template <typename F>
  detail::InlineTaskDetached startImpl(TaskWithExecutor task, F cb) {
    try {
      cb(co_await folly::coro::co_awaitTry(std::move(task)));
    } catch (...) {
      cb(Try<StorageType>(exception_wrapper(std::current_exception())));
    }
  }

  template <typename F>
  detail::InlineTaskDetached startInlineImpl(TaskWithExecutor task, F cb) {
    try {
      cb(co_await InlineTryAwaitable{std::exchange(task.coro_, {})});
    } catch (...) {
      cb(Try<StorageType>(exception_wrapper(std::current_exception())));
    }
  }

 public:
  class Awaiter {
   public:
    explicit Awaiter(handle_t coro) noexcept : coro_(coro) {}

    Awaiter(Awaiter&& other) noexcept : coro_(std::exchange(other.coro_, {})) {}

    ~Awaiter() {
      if (coro_) {
        coro_.destroy();
      }
    }

    bool await_ready() const { return false; }

    template <typename Promise>
    FOLLY_NOINLINE void await_suspend(
        coroutine_handle<Promise> continuation) noexcept {
      DCHECK(coro_);
      auto& promise = coro_.promise();
      DCHECK(!promise.continuation_);
      DCHECK(promise.executor_);
      DCHECK(!dynamic_cast<folly::InlineExecutor*>(promise.executor_.get()))
          << "InlineExecutor is not safe and is not supported for coro::Task. "
          << "If you need to run a task inline in a unit-test, you should use "
          << "coro::blockingWait instead.";
      DCHECK(!dynamic_cast<folly::QueuedImmediateExecutor*>(
          promise.executor_.get()))
          << "QueuedImmediateExecutor is not safe and is not supported for coro::Task. "
          << "If you need to run a task inline in a unit-test, you should use "
          << "coro::blockingWait instead.";
      if constexpr (kIsDebug) {
        if (dynamic_cast<InlineLikeExecutor*>(promise.executor_.get())) {
          FB_LOG_ONCE(ERROR)
              << "InlineLikeExecutor is not safe and is not supported for coro::Task. "
              << "If you need to run a task inline in a unit-test, you should use "
              << "coro::blockingWait or write your test using the CO_TEST* macros instead."
              << "If you are using folly::getCPUExecutor, switch to getGlobalCPUExecutor "
              << "or be sure to call setCPUExecutor first.";
        }
      }
      // 维护协程调用栈
      auto& calleeFrame = promise.getAsyncFrame();
      calleeFrame.setReturnAddress();
      // 这里返回true，同样是在维护协程调用栈
      if constexpr (detail::promiseHasAsyncFrame_v<Promise>) {
        auto& callerFrame = continuation.promise().getAsyncFrame();
        calleeFrame.setParentFrame(callerFrame);
        folly::deactivateAsyncStackFrame(callerFrame);
      }
      // 设置调用者协程为continuation_值，在当前线程执行完后恢复调用者协程调用。
      promise.continuation_ = continuation;
      // 将协程丢到线程池中执行，同时传递ctx
      promise.executor_->add(
          [coro = coro_, ctx = RequestContext::saveContext()]() mutable {
            RequestContextScopeGuard contextScope{std::move(ctx)};
            folly::resumeCoroutineWithNewAsyncStackRoot(coro);
          });
    }
    // 返回对应的result
    T await_resume() {
      DCHECK(coro_);
      // Eagerly destroy the coroutine-frame once we have retrieved the result.
      SCOPE_EXIT { std::exchange(coro_, {}).destroy(); };
      return std::move(coro_.promise().result()).value();
    }

    folly::Try<StorageType> await_resume_try() {
      SCOPE_EXIT { std::exchange(coro_, {}).destroy(); };
      return std::move(coro_.promise().result());
    }

   private:
    handle_t coro_;
  };

  class InlineTryAwaitable {
   public:
    InlineTryAwaitable(handle_t coro) noexcept : coro_(coro) {}

    InlineTryAwaitable(InlineTryAwaitable&& other) noexcept
        : coro_(std::exchange(other.coro_, {})) {}

    ~InlineTryAwaitable() {
      if (coro_) {
        coro_.destroy();
      }
    }

    bool await_ready() { return false; }

    template <typename Promise>
    FOLLY_NOINLINE coroutine_handle<> await_suspend(
        coroutine_handle<Promise> continuation) {
      DCHECK(coro_);
      auto& promise = coro_.promise();
      DCHECK(!promise.continuation_);
      DCHECK(promise.executor_);

      promise.continuation_ = continuation;

      auto& calleeFrame = promise.getAsyncFrame();
      calleeFrame.setReturnAddress();

      // This awaitable is only ever awaited from a DetachedInlineTask
      // which is an async-stack-aware coroutine.
      //
      // Assume it has a .getAsyncFrame() and that this frame is currently
      // active.
      auto& callerFrame = continuation.promise().getAsyncFrame();
      folly::pushAsyncStackFrameCallerCallee(callerFrame, calleeFrame);
      return coro_;
    }

    folly::Try<StorageType> await_resume() {
      DCHECK(coro_);
      // Eagerly destroy the coroutine-frame once we have retrieved the result.
      SCOPE_EXIT { std::exchange(coro_, {}).destroy(); };
      return std::move(coro_.promise().result());
    }

   private:
    friend InlineTryAwaitable tag_invoke(
        cpo_t<co_withAsyncStack>, InlineTryAwaitable&& awaitable) noexcept {
      return std::move(awaitable);
    }

    handle_t coro_;
  };

 public:
  Awaiter operator co_await() && noexcept {
    DCHECK(coro_);
    return Awaiter{std::exchange(coro_, {})};
  }

  friend TaskWithExecutor co_withCancellation(
      folly::CancellationToken cancelToken, TaskWithExecutor&& task) noexcept {
    DCHECK(task.coro_);
    task.coro_.promise().setCancelToken(std::move(cancelToken));
    return std::move(task);
  }

  friend TaskWithExecutor tag_invoke(
      cpo_t<co_withAsyncStack>, TaskWithExecutor&& task) noexcept {
    return std::move(task);
  }

 private:
  friend class Task<T>;

  explicit TaskWithExecutor(handle_t coro) noexcept : coro_(coro) {}

  handle_t coro_;
};
```

先来看当`co_await TaskWithExecutor`时返回的`awaiter`执行逻辑，这里根据之前`TaskPromise`描述可以推断出这里返回的`Awaiter`，其执行核心为`Awaiter::await_suspend`函数，其和Task对应的awaiter核心区别是执行协程的逻辑是在被丢到指定线程池中执行，同时需要维护一下`ctx`，对于ctx的作用可以参考上一篇介绍future的文档：[RequestContext][https://www.yinkuiwang.cn/2023/01/08/folly%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E4%B8%8EDAG/#RequestContext]。

这里由于被调协程会被丢到线程池中执行，因此调用者协程如果直接在被调协程后被`resume`则会破坏调用者协程指定执行位置（线程池），因此需要`co_viaIfAsync`函数（上面有介绍）。

函数`Awaiter::await_suspend`还有一点需要注意的是，这里返回的是空，表示会挂起调用`co_await`方法的协程，并返回到调用该协程的地方。加入到线程池的函数逻辑较为简单，只有如下两句：

```
RequestContextScopeGuard contextScope{std::move(ctx)};
folly::resumeCoroutineWithNewAsyncStackRoot(coro);
```

这里需要注意的是，调用`folly::resumeCoroutineWithNewAsyncStackRoot(coro);`时会恢复当前task绑定的协程，如果恢复协程后，协程内部执行`co_await`返空了，则调用回到`folly::resumeCoroutineWithNewAsyncStackRoot(coro)`函数中的`h.resume()`语句，这时体现到线程上，这个函数就执行完成了，不会出现阻塞线程的情况。

这里的还有一个`InlineTryAwaitable`，似乎只有显示调用`startInlineUnsafe`时会使用，其`await_suspend`也是不加到线程池里直接调用，这一般是指调用者和被调者使用的是同一个线程池。

`TaskWithExecutor`的另一个核心接口是`start`，其含义是执行当前协程，并返回一个`SemiFuture`，用户使用`SemiFuture`来等待调用结束。

其核心是将协程执行状态和一个`promise`绑定，当协程执行完成后，对promise的`SemiFuture`赋值。这里的核心点是如何触发协程的执行，其实现方式是再加一层协程，这里就是

```c++
template <typename F>
  detail::InlineTaskDetached startImpl(TaskWithExecutor task, F cb) {
    try {
      cb(co_await folly::coro::co_awaitTry(std::move(task)));
    } catch (...) {
      cb(Try<StorageType>(exception_wrapper(std::current_exception())));
    }
  }
```

当调用start时，最终会执行到该方法，其返回`InlineTaskDetached`定义如下：

```c++
struct InlineTaskDetached {
  class promise_type {
    struct FinalAwaiter {
      bool await_ready() noexcept { return false; }
      void await_suspend(coroutine_handle<promise_type> h) noexcept {
        folly::deactivateAsyncStackFrame(h.promise().getAsyncFrame());
        h.destroy();
      }
      [[noreturn]] void await_resume() noexcept { folly::assume_unreachable(); }
    };

   public:
    static void* operator new(std::size_t size) {
      return ::folly_coro_async_malloc(size);
    }

    static void operator delete(void* ptr, std::size_t size) {
      ::folly_coro_async_free(ptr, size);
    }

    promise_type() noexcept {
      asyncFrame_.setParentFrame(folly::getDetachedRootAsyncStackFrame());
    }

    InlineTaskDetached get_return_object() noexcept {
      return InlineTaskDetached{
          coroutine_handle<promise_type>::from_promise(*this)};
    }

    suspend_always initial_suspend() noexcept { return {}; }

    FinalAwaiter final_suspend() noexcept { return {}; }

    void return_void() noexcept {}

    [[noreturn]] void unhandled_exception() noexcept { std::terminate(); }

    template <typename Awaitable>
    decltype(auto) await_transform(Awaitable&& awaitable) {
      return folly::coro::co_withAsyncStack(
          static_cast<Awaitable&&>(awaitable));
    }

    folly::AsyncStackFrame& getAsyncFrame() noexcept { return asyncFrame_; }

   private:
    folly::AsyncStackFrame asyncFrame_;
  };

  InlineTaskDetached(InlineTaskDetached&& other) noexcept
      : coro_(std::exchange(other.coro_, {})) {}

  ~InlineTaskDetached() {
    if (coro_) {
      coro_.destroy();
    }
  }

  FOLLY_NOINLINE void start() noexcept {
    start(FOLLY_ASYNC_STACK_RETURN_ADDRESS());
  }

  void start(void* returnAddress) noexcept {
    coro_.promise().getAsyncFrame().setReturnAddress(returnAddress);
    folly::resumeCoroutineWithNewAsyncStackRoot(std::exchange(coro_, {}));
  }

 private:
  explicit InlineTaskDetached(coroutine_handle<promise_type> h) noexcept
      : coro_(h) {}

  coroutine_handle<promise_type> coro_;
};
```

其实现较为简单，`await_transform`方法只是对原`awaitable`增加了一层`co_withAsyncStack`。最终的协程结束处理(`FinalAwaiter`)也没干什么，只是维护了一下调用栈并且析构了一下资源。调用该函数返回`InlineTaskDetached`后会立即调用其start方法。该方法直接将自己持有的协程`resume`，这时就会执行`cb(co_await folly::coro::co_awaitTry(std::move(task)));`，从而触发我们实际要等到的协程的执行。而返回给调用者的`SemiFuture`则给用户做判断是否执行完成，当协程执行完成后，`cb`函数会完成对`promise`的`setTry`，这时调用者获得的`SemiFuture`就变成完成状态。

# 等待协程执行结束

folly官方文档介绍等待协程执行结束有两种方式：

1. 协程调用`scheduleOn().start()`
2. `folly::coro::blockingWait(std::move(task).scheduleOn())`

第一种方式在`TaskWithExecutor`中已经介绍过了，这里再来看一下`blockingWait`的实现。

```c++
inline constexpr blocking_wait_fn blocking_wait{};
static constexpr blocking_wait_fn const& blockingWait =
    blocking_wait; // backcompat
    
struct blocking_wait_fn {
  template <typename Awaitable>
  FOLLY_NOINLINE auto operator()(Awaitable&& awaitable) const
      -> detail::decay_rvalue_reference_t<await_result_t<Awaitable>> {
    folly::AsyncStackFrame frame;
    frame.setReturnAddress();

    folly::AsyncStackRoot stackRoot;
    stackRoot.setNextRoot(folly::tryGetCurrentAsyncStackRoot());
    stackRoot.setStackFrameContext();
    stackRoot.setTopFrame(frame);

    return static_cast<std::add_rvalue_reference_t<await_result_t<Awaitable>>>(
        detail::makeRefBlockingWaitTask(static_cast<Awaitable&&>(awaitable))
            .get(frame));
  }

  template <typename SemiAwaitable>
  FOLLY_NOINLINE auto operator()(
      SemiAwaitable&& awaitable, folly::DrivableExecutor* executor) const
      -> detail::decay_rvalue_reference_t<semi_await_result_t<SemiAwaitable>> {
    folly::AsyncStackFrame frame;
    frame.setReturnAddress();

    folly::AsyncStackRoot stackRoot;
    stackRoot.setNextRoot(folly::tryGetCurrentAsyncStackRoot());
    stackRoot.setStackFrameContext();
    stackRoot.setTopFrame(frame);

    return static_cast<
        std::add_rvalue_reference_t<semi_await_result_t<SemiAwaitable>>>(
        detail::makeRefBlockingWaitTask(
            folly::coro::co_viaIfAsync(
                folly::getKeepAliveToken(executor),
                static_cast<SemiAwaitable&&>(awaitable)))
            .getVia(executor, frame));
  }

  template <
      typename SemiAwaitable,
      std::enable_if_t<!is_awaitable_v<SemiAwaitable>, int> = 0>
  auto operator()(SemiAwaitable&& awaitable) const
      -> detail::decay_rvalue_reference_t<semi_await_result_t<SemiAwaitable>> {
    std::exception_ptr eptr;
    {
      detail::BlockingWaitExecutor executor;
      try {
        return operator()(static_cast<SemiAwaitable&&>(awaitable), &executor);
      } catch (...) {
        eptr = std::current_exception();
      }
    }
    std::rethrow_exception(eptr);
  }
};
```

这里执行的主要是第一个`operator`方法。其中`makeRefBlockingWaitTask`定义如下：

```
template <
    typename Awaitable,
    typename Result = await_result_t<Awaitable>,
    std::enable_if_t<std::is_void<Result>::value, int> = 0>
BlockingWaitTask<void> makeRefBlockingWaitTask(Awaitable&& awaitable) {
  co_await static_cast<Awaitable&&>(awaitable);
}

template <
    typename Awaitable,
    typename Result = await_result_t<Awaitable>,
    std::enable_if_t<!std::is_void<Result>::value, int> = 0>
auto makeRefBlockingWaitTask(Awaitable&& awaitable)
    -> BlockingWaitTask<std::add_lvalue_reference_t<Result>> {
  co_yield co_await static_cast<Awaitable&&>(awaitable);
}
```

这里`BlockingWaitTask`是一个协程返回值。其定义如下：

```c++
template <typename T>
class BlockingWaitTask {
 public:
  using promise_type = BlockingWaitPromise<T>;
  using handle_t = coroutine_handle<promise_type>;

  explicit BlockingWaitTask(handle_t coro) noexcept : coro_(coro) {}

  BlockingWaitTask(BlockingWaitTask&& other) noexcept
      : coro_(std::exchange(other.coro_, {})) {}

  BlockingWaitTask& operator=(BlockingWaitTask&& other) noexcept = delete;

  ~BlockingWaitTask() {
    if (coro_) {
      coro_.destroy();
    }
  }

  FOLLY_NOINLINE T get(folly::AsyncStackFrame& parentFrame) && {
    folly::Try<detail::lift_lvalue_reference_t<T>> result;
    auto& promise = coro_.promise();
    promise.setTry(&result);

    auto& asyncFrame = promise.getAsyncFrame();
    asyncFrame.setParentFrame(parentFrame);
    asyncFrame.setReturnAddress();
    {
      RequestContextScopeGuard guard{RequestContext::saveContext()};
      folly::resumeCoroutineWithNewAsyncStackRoot(coro_);
    }
    promise.wait();
    return std::move(result).value();
  }

 private:
  handle_t coro_;
};
```

当调用get方法时，会将自己持有的协程`resume`。这里实际执行的就是`co_await static_cast<Awaitable&&>(awaitable);`，也即我们要等到执行结束的协程。之后执行`promise.wait()`等待协程执行结束。这里需要看一下`promise`的实现。

```c++
class BlockingWaitPromiseBase {
  struct FinalAwaiter {
    bool await_ready() noexcept { return false; }
    template <typename Promise>
    void await_suspend(coroutine_handle<Promise> coro) noexcept {
      BlockingWaitPromiseBase& promise = coro.promise();
      folly::deactivateAsyncStackFrame(promise.getAsyncFrame());
      promise.baton_.post();
    }
    void await_resume() noexcept {}
  };

 public:
  BlockingWaitPromiseBase() noexcept = default;

  static void* operator new(std::size_t size) {
    return ::folly_coro_async_malloc(size);
  }

  static void operator delete(void* ptr, std::size_t size) {
    ::folly_coro_async_free(ptr, size);
  }

  suspend_always initial_suspend() { return {}; }

  FinalAwaiter final_suspend() noexcept { return {}; }

  template <typename Awaitable>
  decltype(auto) await_transform(Awaitable&& awaitable) {
    return folly::coro::co_withAsyncStack(static_cast<Awaitable&&>(awaitable));
  }

  bool done() const noexcept { return baton_.ready(); }

  void wait() noexcept { baton_.wait(); }

  folly::AsyncStackFrame& getAsyncFrame() noexcept { return asyncFrame_; }

 private:
  folly::fibers::Baton baton_;
  folly::AsyncStackFrame asyncFrame_;
};
```

其中核心在`BlockingWaitPromiseBase`中，其存在一个`folly::fibers::Baton`类。该类的实现使用的是`futex`，可以参考该文档[futex](https://man7.org/linux/man-pages/man2/futex.2.html)。这里不展开介绍（其实是不太会orz）。核心是一个同步原语。folly对其封装了一下，核心是两个接口，一个是`post`，另一个是`wait`。其中`wait`用于等待同步信号，`post`用于发送信号，在post发送信号前，调用`wait`的线程会被阻塞，直到另一个线程发送了`post`信号（有点像条件变量的感觉）。因此这里在将我们等待的协程resum后，就通过`wait`接口等待协程完成。

这里有一点需要注意，正常来说`BlockingWaitPromiseBase`持有的协程被恢复后，如果执行完成了，我们等待的协程应该执行结束了才对，为啥还需要使用`baton`来进行同步呢？这时因为，将`BlockingWaitPromiseBase`持有的协程resume，该协程不一定（在这里是大概率）表示该协程被执行完成了。当我们resume `BlockingWaitPromiseBase`持有的协程时，执行`co_await static_cast<Awaitable&&>(awaitable);`，当我们`co_await`的协程被指定到特定线程池执行时，执行`co_await`时调用的`await_suspend`方法返回就是空（可以看`TaskWithExecutor`的`Awaiter::await_suspend`方法），这时会立即回到`resume`协程的地方，这里就是回到了即回到`folly::resumeCoroutineWithNewAsyncStackRoot(coro_)`里面的`h.resume()`这条语句这里，接着执行后面的逻辑。而`BlockingWaitPromiseBase`持有的协程被挂起，当我们等待的协程执行结束时，会重新唤醒`BlockingWaitPromiseBase`持有的协程，进行后续处理。因此在这里我们使用`promise.wait();`等待`BlockingWaitPromiseBase`被重新唤醒并被执行完成（在执行完成时`FinalAwaiter`的`await_suspend`发送`post`信号告知执行结束），这样才能保证我们等待的协程确定被执行完成。

# clollectAll

想DAG中依赖关系一样，一个协程依赖的数据产出可能需要多个协程生成，这时，如果我们按照协程实际依赖的数据，每次都`co_await`对应的协程，将会导致依赖的协程顺序被触发，串行执行，这在对耗时较为敏感的系统中是不可接受的，我们需要有统一触发多个协程并发执行的接口，这就是这一节要介绍的`collectAll`接口。其传递的参数是一个`Task`的list，如果task是异步的，即指定线程池执行，则所有的task会被异步执行，如果task是同步的（没有转换为`TaskWithExecutor`，则会被同步执行）。

其实现如下：

```c++
template <
    typename InputRange,
    std::enable_if_t<
        !std::is_void_v<
            semi_await_result_t<detail::range_reference_t<InputRange>>>,
        int>>
auto collectAllRange(InputRange awaitables)
    -> folly::coro::Task<std::vector<detail::collect_all_range_component_t<
        detail::range_reference_t<InputRange>>>> {
  // 利用task promise的await_transform获取到当前协程的executor
  const folly::Executor::KeepAlive<> executor = co_await co_current_executor;
  const CancellationSource cancelSource;
  // 获取当前协程的cancellation，并和自己协程创建的cancellation进行merge
  const CancellationToken cancelToken = CancellationToken::merge(
      co_await co_current_cancellation_token, cancelSource.getToken());
  //使用try list 存储结果
  std::vector<detail::collect_all_try_range_component_t<
      detail::range_reference_t<InputRange>>>
      tryResults;

  exception_wrapper firstException;

  using awaitable_type = remove_cvref_t<detail::range_reference_t<InputRange>>;
  // 对每一个task封装一层协程，协程返回值是detail::BarrierTask
  auto makeTask = [&](awaitable_type semiAwaitable,
                      std::size_t index) -> detail::BarrierTask {
    assert(index < tryResults.size());

    try {
      // co_viaIfAsync限制我们创建的协程函数始终在当前协程设置的executor上执行
      tryResults[index].emplace(co_await co_viaIfAsync(
          executor.get_alias(),
          co_withCancellation(cancelToken, std::move(semiAwaitable))));
    } catch (...) {
      if (!cancelSource.requestCancellation()) {
        firstException = exception_wrapper{std::current_exception()};
      }
    }
  };
  // 遍历awaitables，按照makeTask函数创建协程
  auto tasks = detail::collectMakeInnerTaskVec(awaitables, makeTask);

  tryResults.resize(tasks.size());

  // Save the initial context and restore it after starting each task
  // as the task may have modified the context before suspending and we
  // want to make sure the next task is started with the same initial
  // context.
  const auto context = RequestContext::saveContext();

  auto& asyncFrame = co_await detail::co_current_async_stack_frame;

  // Launch the tasks and wait for them all to finish.
  {
    // 设置一个屏障，值为我们要等待的协程数+1，这里的1是collectAll自己协程处理的那个1，即下面的co_await语句
    detail::Barrier barrier{tasks.size() + 1};
    for (auto&& task : tasks) {
      // resume我们创建的协程
      task.start(&barrier, asyncFrame);
      // 恢复当前线程的上下文，因为task.start可能会变更当前线程的context
      RequestContext::setContext(context);
    }
    // 设置所有协程执行完成后唤醒当前协程，并等待返回。
    co_await detail::UnsafeResumeInlineSemiAwaitable{barrier.arriveAndWait()};
  }
  // 异常处理
  // Check if there were any exceptions and rethrow the first one.
  if (firstException) {
    co_yield co_error(std::move(firstException));
  }

  // 从try中获取结果
  std::vector<detail::collect_all_range_component_t<
      detail::range_reference_t<InputRange>>>
      results;
  results.reserve(tryResults.size());
  for (auto& result : tryResults) {
    results.emplace_back(std::move(result).value());
  }
  // 设置结果
  co_return results;
}

template <
    typename InputRange,
    typename Make,
    typename Iter = invoke_result_t<access::begin_fn, InputRange&>,
    typename Elem = remove_cvref_t<decltype(*std::declval<Iter&>())>,
    typename RTask = invoke_result_t<Make&, Elem, std::size_t>>
std::vector<RTask> collectMakeInnerTaskVec(InputRange& awaitables, Make& make) {
  std::vector<RTask> tasks;

  auto abegin = access::begin(awaitables);
  auto aend = access::end(awaitables);

  if constexpr (is_invocable_v<folly::access::size_fn, InputRange&>) {
    tasks.reserve(static_cast<std::size_t>(folly::access::size(awaitables)));
  } else if constexpr (range_has_known_distance_v<InputRange&>) {
    tasks.reserve(static_cast<std::size_t>(std::distance(abegin, aend)));
  }

  std::size_t index = 0;
  for (auto aiter = abegin; aiter != aend; ++aiter) {
    tasks.push_back(make(std::move(*aiter), index++));
  }

  return tasks;
}

template <typename Awaitable>
class UnsafeResumeInlineSemiAwaitable {
 public:
  explicit UnsafeResumeInlineSemiAwaitable(Awaitable&& awaitable) noexcept
      : awaitable_(awaitable) {}

  Awaitable&& viaIfAsync(folly::Executor::KeepAlive<>) && noexcept {
    return static_cast<Awaitable&&>(awaitable_);
  }

 private:
  Awaitable awaitable_;
};
```

可以看到，`collectAllRange`本身也是一个协程函数，返回值是一个`Task`。因此我们调用该方法时，拿到task后，还需要`co_await task`。

具体的每步执行逻辑上面都进行了注解。这里核心需要关注的是`detail::Barrier`和`detail::BarrierTask`。

## Barrier

`Barrier`是一个屏障，当所以协程执行完成后，才就绪，其定义如下。

```c++
class Barrier {
 public:
  // 屏障计数
  explicit Barrier(std::size_t initialCount = 0) noexcept
      : count_(initialCount) {}

  void add(std::size_t count = 1) noexcept {
    [[maybe_unused]] std::size_t oldCount =
        count_.fetch_add(count, std::memory_order_relaxed);
    // Check we didn't overflow the count.
    assert(SIZE_MAX - oldCount >= count);
  }

  // Query the number of remaining tasks that the barrier is waiting
  // for. This indicates the number of arrive() calls that must be
  // made before the Barrier will be released.
  //
  // Note that this should just be used as an approximate guide
  // for the number of outstanding tasks. This value may be out
  // of date immediately upon being returned.
  std::size_t remaining() const noexcept {
    return count_.load(std::memory_order_acquire);
  }
  
  // 接受一个信号（这里一般表示一个协程结束），将计数减一，并根据是否所以条件均满足来决定是否唤醒等待的协程
  [[nodiscard]] coroutine_handle<> arrive(
      folly::AsyncStackFrame& currentFrame) noexcept {
    auto& stackRoot = *currentFrame.getStackRoot();
    folly::deactivateAsyncStackFrame(currentFrame);

    const std::size_t oldCount = count_.fetch_sub(1, std::memory_order_acq_rel);

    // Invalid to call arrive() if you haven't previously incremented the
    // counter using .add().
    assert(oldCount >= 1);

    if (oldCount == 1) {
      if (asyncFrame_ != nullptr) {
        folly::activateAsyncStackFrame(stackRoot, *asyncFrame_);
      }
      return std::exchange(continuation_, {});
    } else {
      return coro::noop_coroutine();
    }
  }

  // 接受一个信号（这里一般表示一个协程结束），将计数减一，并根据是否所以条件均满足来决定是否唤醒等待的协程
  [[nodiscard]] coroutine_handle<> arrive() noexcept {
    const std::size_t oldCount = count_.fetch_sub(1, std::memory_order_acq_rel);

    // Invalid to call arrive() if you haven't previously incremented the
    // counter using .add().
    assert(oldCount >= 1);

    if (oldCount == 1) {
      auto coro = std::exchange(continuation_, {});
      if (asyncFrame_ != nullptr) {
        folly::resumeCoroutineWithNewAsyncStackRoot(coro, *asyncFrame_);
        return coro::noop_coroutine();
      } else {
        return coro;
      }
    } else {
      return coro::noop_coroutine();
    }
  }

 private:
  class Awaiter {
   public:
    explicit Awaiter(Barrier& barrier) noexcept : barrier_(barrier) {}

    bool await_ready() { return false; }
    
    // co_await时设置最终要唤醒的协程，并将屏障计数-1，如果条件已经满足，则会立即唤醒对应协程
    template <typename Promise>
    coroutine_handle<> await_suspend(
        coroutine_handle<Promise> continuation) noexcept {
      if constexpr (detail::promiseHasAsyncFrame_v<Promise>) {
        barrier_.setContinuation(
            continuation, &continuation.promise().getAsyncFrame());
        return barrier_.arrive(continuation.promise().getAsyncFrame());
      } else {
        barrier_.setContinuation(continuation, nullptr);
        return barrier_.arrive();
      }
    }

    void await_resume() noexcept {}

   private:
    friend Awaiter tag_invoke(
        cpo_t<co_withAsyncStack>, Awaiter&& awaiter) noexcept {
      return Awaiter{awaiter.barrier_};
    }

    Barrier& barrier_;
  };

 public:
  // 返回一个Awaiter，并使用this初始化
  auto arriveAndWait() noexcept { return Awaiter{*this}; }

  // 设置屏障结束时需要唤醒的协程
  void setContinuation(
      coroutine_handle<> continuation,
      folly::AsyncStackFrame* parentFrame) noexcept {
    assert(!continuation_);
    continuation_ = continuation;
    asyncFrame_ = parentFrame;
  }

 private:
  std::atomic<std::size_t> count_;
  coroutine_handle<> continuation_;
  folly::AsyncStackFrame* asyncFrame_ = nullptr;
};
```

其存在三个成员变量。

1. `count_`计数，用来记录当前还未执行完成的协程数量。
2. `continuation_`表示当所有条件就绪后需要执行唤醒的协程。
3. `asyncFrame_`用于维护协程栈。

这里的`Awaiter`就是collectAll中最后我们`co_await`的`awaiter`。当`collectAll` `co_await`时，首先设置了最终要唤醒`collectAll`函数，并将引用计数减一。当每个协程执行结束时，也会将引用计数减一。

## BarrierTask

`BarrierTask`是`collectAll`中对每一个`task`包的一层协程。其定义如下：

```c++
class BarrierTask {
 public:
  class promise_type {
    struct FinalAwaiter {
      bool await_ready() noexcept { return false; }

      coroutine_handle<> await_suspend(
          coroutine_handle<promise_type> h) noexcept {
        auto& promise = h.promise();
        assert(promise.barrier_ != nullptr);
        return promise.barrier_->arrive(promise.asyncFrame_);
      }

      void await_resume() noexcept {}
    };

   public:
    static void* operator new(std::size_t size) {
      return ::folly_coro_async_malloc(size);
    }

    static void operator delete(void* ptr, std::size_t size) {
      ::folly_coro_async_free(ptr, size);
    }

    BarrierTask get_return_object() noexcept {
      return BarrierTask{coroutine_handle<promise_type>::from_promise(*this)};
    }

    suspend_always initial_suspend() noexcept { return {}; }

    FinalAwaiter final_suspend() noexcept { return {}; }

    template <typename Awaitable>
    auto await_transform(Awaitable&& awaitable) {
      return folly::coro::co_withAsyncStack(
          static_cast<Awaitable&&>(awaitable));
    }

    void return_void() noexcept {}

    [[noreturn]] void unhandled_exception() noexcept { std::terminate(); }

    void setBarrier(Barrier* barrier) noexcept {
      assert(barrier_ == nullptr);
      barrier_ = barrier;
    }

    folly::AsyncStackFrame& getAsyncFrame() noexcept { return asyncFrame_; }

   private:
    folly::AsyncStackFrame asyncFrame_;
    Barrier* barrier_ = nullptr;
  };

 private:
  using handle_t = coroutine_handle<promise_type>;

  explicit BarrierTask(handle_t coro) noexcept : coro_(coro) {}

 public:
  BarrierTask(BarrierTask&& other) noexcept
      : coro_(std::exchange(other.coro_, {})) {}

  ~BarrierTask() {
    if (coro_) {
      coro_.destroy();
    }
  }

  BarrierTask& operator=(BarrierTask other) noexcept {
    swap(other);
    return *this;
  }

  void swap(BarrierTask& b) noexcept { std::swap(coro_, b.coro_); }

  FOLLY_NOINLINE void start(Barrier* barrier) noexcept {
    start(barrier, folly::getDetachedRootAsyncStackFrame());
  }
  
  // 执行当前协程
  FOLLY_NOINLINE void start(
      Barrier* barrier, folly::AsyncStackFrame& parentFrame) noexcept {
    assert(coro_);
    auto& calleeFrame = coro_.promise().getAsyncFrame();
    calleeFrame.setParentFrame(parentFrame);
    calleeFrame.setReturnAddress();
    coro_.promise().setBarrier(barrier);

    folly::resumeCoroutineWithNewAsyncStackRoot(coro_);
  }

 private:
  // 当前协程的coro
  handle_t coro_;
};
```

`promise_type`持有一个`barrier`的指针，当我们调用`BarrierTask`的start函数时，传递`barrier`，将在`collectAll`中创建的`barrier`传递到`promise_type`。调用`start`函数后，就会恢复当前协程的执行。在执行完成当前协程后，`final_suspend`返回`FinalAwaiter`。当`co_await`该`awaiter`时，执行`await_suspend`函数，执行`barrier`的`arrive`函数，将`barrier`计数减一，并根据是否已经减到0了来决定是否唤醒等待的协程。

这里在`collectAll`中设置`barrier`为task数量+1中的+1，是collectAll调用`co_await detail::UnsafeResumeInlineSemiAwaitable{barrier.arriveAndWait()};`使用。调用这个函数，会设置barrier满足条件后唤醒的`collectAll`，并将当前协程挂起，等所以协程都执行结束后再唤醒该协程。

这里需要注意的是，`BarrierTask`执行start函数唤醒当前协程时，会执行在`collectAll`中`makeTask`函数，即执行

```c++
co_await co_viaIfAsync(
          executor.get_alias(),
          co_withCancellation(cancelToken, std::move(semiAwaitable)))
```

如果这里的`semiAwaitable`不是异步协程，则会陷入`semiAwaitable`协程执行对应协程的处理逻辑，执行完成后再返回到`makeTask`函数。而如果`semiAwaitable`是异步协程，则执行上述语句后会立即返回到`start`函数中，继续执行for循环后的其他start方法。

因此，当我们`collectAll`时，为了让所有协程并发执行，一定要传异步协程，否则所有协程一样会被串行执行，和一个一个`co_await`没区别（甚至因为套了一层协程更废了）。

# promise&future&SharedPromise

在上一节，我们解决了一个协程依赖多个协程产生的数据。但是在实际执行中，往往还有另一个问题，如果应该协程产生的数据被多个协程依赖如何处理。根据前面的学习，我们知道，肯定不能是多个协程同时`co_await`同一个task（协程正常应该只需要执行一次，并且task上线中，co_await时生成的awaiter都使用了`std::exchange(t.coro_, {})`方式，不支持多次co_await）。

目前对于该需求的实现，和之前介绍的future方法实现一致，即使用`SharedPromise`。具体可以参考上一篇文档：[future&SharedPromise](https://www.yinkuiwang.cn/2023/01/08/folly%E5%BC%82%E6%AD%A5%E6%A1%86%E6%9E%B6%E4%B8%8EDAG/#future-amp-promise)。

将协程的task绑定到一个`SharedPromise`上，当某个协程依赖该协程使用数据时，从`SharedPromise`中获取一个`future`，`co_await`该future即可。当协程执行完成后，设置`SharedPromise`状态为完成状态，这时所有等待的协程都会被唤醒。

这里的关键是对`co_await future`的实现，要保证在`co_await future`时协程挂起不阻塞。其实现也较为简单：

```c++
template <typename T>
inline detail::FutureAwaiter<T>
/* implicit */ operator co_await(Future<T>&& future) noexcept {
  return detail::FutureAwaiter<T>(std::move(future));
}
```

当`co_await future`是，返回的`awaiter`是`detail::FutureAwaiter`，其实现如下：

```c++
template <typename T>
class FutureAwaiter {
 public:
  explicit FutureAwaiter(folly::Future<T>&& future) noexcept
      : future_(std::move(future)) {}

  bool await_ready() {
    if (future_.isReady()) {
      result_ = std::move(future_.result());
      return true;
    }
    return false;
  }

  T await_resume() { return std::move(result_).value(); }

  Try<drop_unit_t<T>> await_resume_try() {
    return static_cast<Try<drop_unit_t<T>>>(std::move(result_));
  }

  FOLLY_CORO_AWAIT_SUSPEND_NONTRIVIAL_ATTRIBUTES void await_suspend(
      coro::coroutine_handle<> h) {
    // FutureAwaiter may get destroyed as soon as the callback is executed.
    // Make sure the future object doesn't get destroyed until setCallback_
    // returns.
    auto future = std::move(future_);
    future.setCallback_(
        [this, h](Executor::KeepAlive<>&&, Try<T>&& result) mutable {
          result_ = std::move(result);
          h.resume();
        });
  }

 private:
  folly::Future<T> future_;
  folly::Try<T> result_;
};
```

其实现也较为简单，当`co_await future`时，如果`future`已经ready，则立即继续执行就好了，因此`await_ready`中先判断就绪状态。如果future未就绪，则执行`await_suspend`时，在future后增加一个`callback`，其逻辑是在future就绪后唤醒当前协程，同时返回空，挂起当前协程。

例子：

```c++
#include "folly/experimental/coro/BlockingWait.h"
#include <atomic>
#include <coroutine>
#include <mutex>
#include <string>
#include <folly/executors/GlobalExecutor.h>
#include <iostream>
#include <folly/init/Init.h>
#include<folly/experimental/coro/Task.h>
#include <folly/experimental/coro/Collect.h>
#include <sys/select.h>
#include <folly/experimental/coro/SharedMutex.h>
#include "folly/executors/CPUThreadPoolExecutor.h"
#include "folly/experimental/coro/SharedPromise.h"
#include "folly/futures/SharedPromise.h"

std::map<std::string, folly::SharedPromise<folly::Unit>> global_maping;
std::atomic_int value = 0;

folly::coro::SharedMutexFair lock;

folly::coro::Task<void> create_depends(std::map<std::string, folly::SharedPromise<folly::Unit>>& map) {
    folly::SharedPromise<folly::Unit>* promise = nullptr;
    {
        auto share_lock = co_await lock.co_scoped_lock_shared();
        if(map.find("create_depends") != map.end()) {
            promise = &map["create_depends"];
        }
    }
    // 由于重复对同一个SharedPromise调用setValue会抛异常，因此这里加了写锁后再确认一下
    if(promise == nullptr) {
        {
            auto share_lock = co_await lock.co_scoped_lock();
            if(map.find("create_depends") != map.end()) {
                // 读
                promise = &map["create_depends"];
            } else {
                // 写
                promise = &map["create_depends"];
            }
        }
    }
    // 保证只执行一次
    if(!promise->isFulfilled()) {
        std::cout<<"create_depends value is "<<value++<<std::endl;
        promise->setValue();
    }
    co_return;
}

folly::coro::Task<void> use_depends_a(std::map<std::string, folly::SharedPromise<folly::Unit>>& map) {
    folly::SharedPromise<folly::Unit>* promise = nullptr;
    {
        auto share_lock = co_await lock.co_scoped_lock_shared();
        if(map.find("create_depends") != map.end()) {
            promise = &map["create_depends"];
        }
    }
    if(promise == nullptr) {
        // 由于读写锁生命周期问题，需要有这个标识来避免在写锁的生命周期内执行新的协程
        bool need_co_await = false;
        {
            auto share_lock = co_await lock.co_scoped_lock();
            if(map.find("create_depends") != map.end()) {
                // 读
                promise = &map["create_depends"];
            } else {
                /* 
                写，保证依赖的协程一定会被触发，执行一次对其的co_await 
                因此create_depends最多会被调用两次，一次是其依赖的协程对其添加，一次是最初的collectAll时添加
                因此create_depends需要被设计为可重入函数
                */
                promise = &map["create_depends"];
                need_co_await = true;
            }   
        }
        co_await create_depends(map).scheduleOn(folly::getGlobalCPUExecutor());
    }
    co_await promise->getFuture();
    std::cout<<"use_depends_a value is "<<value<<std::endl;
    co_return;
}

folly::coro::Task<void> use_depends_b(std::map<std::string, folly::SharedPromise<folly::Unit>>& map) {
    folly::SharedPromise<folly::Unit>* promise = nullptr;
    {
        auto share_lock = co_await lock.co_scoped_lock_shared();
        if(map.find("create_depends") != map.end()) {
            promise = &map["create_depends"];
        }
    }
    if(promise == nullptr) {
        bool need_co_await = false;
        {
            auto share_lock = co_await lock.co_scoped_lock();
            if(map.find("create_depends") != map.end()) {
                // 读
                promise = &map["create_depends"];
            } else {
                /* 
                写，保证依赖的协程一定会被触发，执行一次对其的co_await 
                因此create_depends最多会被调用两次，一次是其依赖的协程对其添加，一次是最初的collectAll时添加
                因此create_depends需要被设计为可重入函数
                */
                promise = &map["create_depends"];
                need_co_await = true;
            }   
        }
        co_await create_depends(map).scheduleOn(folly::getGlobalCPUExecutor());
    }
    co_await promise->getFuture();
    std::cout<<"use_depends_b value is "<<value<<std::endl;
    co_return;
}

folly::coro::Task<void> get_all(std::map<std::string, folly::SharedPromise<folly::Unit>>& map) {
    std::vector<folly::coro::TaskWithExecutor<void>> sum;
    sum.push_back(use_depends_a(map).scheduleOn(folly::getGlobalCPUExecutor()));
    sum.push_back(use_depends_b(map).scheduleOn(folly::getGlobalCPUExecutor()));
    co_await folly::coro::collectAllRange(std::move(sum));
}

folly::coro::Task<void> get_all_v2(std::map<std::string, folly::SharedPromise<folly::Unit>>& map) {
    std::vector<folly::coro::TaskWithExecutor<void>> sum;
    sum.push_back(use_depends_a(map).scheduleOn(folly::getGlobalCPUExecutor()));
    sum.push_back(use_depends_b(map).scheduleOn(folly::getGlobalCPUExecutor()));
    sum.push_back(create_depends(map).scheduleOn(folly::getGlobalCPUExecutor()));
    co_await folly::coro::collectAllRange(std::move(sum));
}


int main(int argc, char *argv[])
{
    folly::init(&argc, &argv);
    auto task1 = get_all(global_maping);
    folly::coro::blockingWait(std::move(task1).scheduleOn(folly::getGlobalCPUExecutor()));
    global_maping.clear();

    auto task2 = get_all_v2(global_maping);
    folly::coro::blockingWait(std::move(task2).scheduleOn(folly::getGlobalCPUExecutor()));

    return 0;
}

// g++ -L/opt/lib -I/opt/include test_coro.cpp -std=c++20 -lfolly -lglog -lgflags -lpthread -ldl -ldouble-conversion -lfmt -levent -lboost_context


./a.out -folly_global_cpu_executor_threads=5
create_depends value is 0
use_depends_b value is 1
use_depends_a value is 1
create_depends value is 1
use_depends_a value is 2
use_depends_b value is 2
$
```

这里借助`SharedPromise`，我们将协程函数设计成了可重入函数。其中使用的`folly::coro::SharedMutexFair`是协程的读写锁，后面会进行介绍。其他相关逻辑都有注释介绍，这里就不一一解释了。

# 核心点

使用协程，并且每个协程都分配一个线程池执行的情况下，执行层面优化点在哪，其实每次的调度都是需要把协程丢到线程池队列去执行，那每一个协程的实际执行需要通过线程池的分发。虽然需要通过线程池来进行分发协程任务，但是线程池在执行协程时基本不会阻塞，这就大大减少了内核调度的开销，在线程池分发协程任务时，可能会由于使用有锁队列造成一定的线程阻塞，但大部分情况来说（协程任务不是特别碎的情况下）这部分开销相对较小，因此使用协程还是会有较大收益。

这时我们再考虑一下folly的future，其实如果我们能够完全按照规范使用future，在任务内部不要死等，而都交于future的调度，其实也不会造成线程阻塞，效率理论上来说和协程应该不会差距很多。因此只要解决当前future实现的异步方法中死等的问题即可，利用coro就很好实现了。所以folly的coro兼容future，只需要在原来任务内部`future.get()`的地方改成`co_await future`，并且函数返回值是`task`即可，这样原来的future和现在的core其实新能应该差距不大。

# 避免阻塞

协程核心就是避免阻塞造成操作系统对线程的切换开销。因此如何避免阻塞就是协程库需要考虑的核心问题了，`folly`对原阻塞方法都提供了相应的非阻塞方法，下面我们针对性的进行介绍。

## sleep

`sleep`方法时明显的阻塞调用。`folly`利用事件驱动框架封装了一个非阻塞版本的`sleep`。其实现如下：

```c++
Task<void> sleep(HighResDuration d, Timekeeper* tk = nullptr);

inline Task<void> sleep(HighResDuration d, Timekeeper* tk) {
  bool cancelled{false};
  folly::coro::Baton baton;
  Try<Unit> result;
  auto future = folly::futures::sleep(d, tk).toUnsafeFuture();
  future.setCallback_(
      [&result, &baton](Executor::KeepAlive<>&&, Try<Unit>&& t) {
        result = std::move(t);
        baton.post();
      });

  {
    CancellationCallback cancelCallback(
        co_await co_current_cancellation_token, [&]() noexcept {
          cancelled = true;
          future.cancel();
        });
    co_await baton;
  }
  if (cancelled) {
    co_yield co_cancelled;
  }
  co_yield co_result(std::move(result));
}
```

`folly::coro::Baton`在介绍等待协程结束章节已经介绍过了，这里不做赘述。这里的核心是`folly::futures::sleep(d, tk).toUnsafeFuture();`该方法返回一个`future`。其实直接将其返回，我们`co_await future`就可以了，但这里为了返回`task`，直接将`co_await future`的实现封装到了函数内部。

`folly::futures::sleep(d, tk)`方法将超时时间加到全局的事件驱动框架中（`EventBase`类），该事件驱动框架基于`libevent`实现，这个没看过，不过可以参考`nginx`的事件驱动框架[nginx时间驱动框架](https://www.yinkuiwang.cn/2021/09/30/nginx%E6%9E%B6%E6%9E%84%E5%8F%8A%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/#%E4%BA%8B%E4%BB%B6%E6%A8%A1%E5%9D%97)。这里就不展开介绍了，后续如果有时间，可以研究一下。

## IO

IO是阻塞的重灾区，对于同步io，是没有办法解决阻塞问题的，只能改成使用异步IO。Facebook开源的thrift rpc支持异步io。可以将异步io返回值设置成`future`，当io完成后，设置对应的promise即可。这时在业务代码中，我们只需要`co_await future`即可。

对`future`支持`co_await`上面已经介绍过了，这里就不再赘述了。

## 锁

使用一般的线程锁，当出现锁冲突时，未获取到锁的线程将会被挂起。为了避免由于锁导致的阻塞问题，folly提供了协程锁（协程锁的设计不是为了避免协程切换导致的死锁，而是为了避免协程阻塞）。这里解释上文使用过的协程锁`SharedMutexFair`。

`SharedMutexFair`实现时基于自旋锁和原子变量实现的，实际以原子变量状态控制锁信息，自旋锁只在读写原子变量时使用。每次读写完原子变量立即释放锁，避免阻塞。

这里只介绍核心接口及其实现。

```c++
class SharedMutexFair : private folly::NonCopyableNonMovable {
  template <typename Awaiter>
  class LockOperation;
  class LockAwaiter;
  class ScopedLockAwaiter;
  class LockSharedAwaiter;
  class ScopedLockSharedAwaiter;
  
 public:
  SharedMutexFair() noexcept = default;

  ~SharedMutexFair();
  // 尝试获取读锁
  bool SharedMutexFair::try_lock() noexcept {
    auto lock = state_.contextualLock();
    // 仅锁不被任何协程持有时可以加写锁
    if (lock->lockedFlagAndReaderCount_ == kUnlocked) {
      lock->lockedFlagAndReaderCount_ = kExclusiveLockFlag;
      return true;
    }
    return false;
  }
  // 尝试获取读锁
  bool SharedMutexFair::try_lock_shared() noexcept {
    auto lock = state_.contextualLock();
    /* 
    如果没人持有任何锁，或者有人持有读锁，但是没有正在等待的协程，则表示可以成功获取读锁，在lockedFlagAndReaderCount_增加计数
    由于1-63位表示读锁数量，因此一次加2。
    这里分析一下为啥大于读锁数量并且没有等待协程时可以加锁
    当读锁被别的协程持有时，获取写锁的协程一定时在等待队列中
    当lockedFlagAndReaderCount_ > kSharedLockCountIncrement时，表示当前有协程持有读锁，正常来说可以直接加读锁即可
    但是如果一直优先读锁，将会导致写锁被饿死，永远无法被调度。
    因此这里先判断一下是否等待队列是否为空，如果不为空，则表示现在有协程在尝试获取写锁了，这时就不让再加读锁了，优先写锁。
    如果没有写锁请求，则可以直接加锁
    */
    if (lock->lockedFlagAndReaderCount_ == kUnlocked ||
        (lock->lockedFlagAndReaderCount_ >= kSharedLockCountIncrement &&
         lock->waitersHead_ == nullptr)) {
      lock->lockedFlagAndReaderCount_ += kSharedLockCountIncrement;
      return true;
    }
    return false;
  }
  // 获取写锁，需要用户自己释放
  [[nodiscard]] LockOperation<LockAwaiter> co_lock() noexcept;
  // 获取写锁，当返回值生命周期结束，自动释放锁
  [[nodiscard]] LockOperation<ScopedLockAwaiter> co_scoped_lock() noexcept;
  // 获取读锁，需要用户自己释放
  [[nodiscard]] LockOperation<LockSharedAwaiter> co_lock_shared() noexcept;
  // 获取读锁，当生命周期结束，自动释放
  [[nodiscard]] LockOperation<ScopedLockSharedAwaiter>
  co_scoped_lock_shared() noexcept;
  
  // 释放写死
  void unlock() noexcept;
  
  // 释放读锁
  void unlock_shared() noexcept;
private:
  // 标识锁状态
  enum class LockType : std::uint8_t { EXCLUSIVE, SHARED };
  struct State {
    State() noexcept
        : lockedFlagAndReaderCount_(kUnlocked),
          waitersHead_(nullptr),
          waitersTailNext_(&waitersHead_) {}

    // bit 0          - locked by writer
    // bits 1-[31/63] - reader lock count
    // 标识锁状态，第0位标识是否被被加了读锁，1-63位标识读锁数量
    std::size_t lockedFlagAndReaderCount_;
    // 将等待锁的协程使用链表串连
    LockAwaiterBase* waitersHead_;
    LockAwaiterBase** waitersTailNext_;
  };
  // 等待锁的协程awaiter
  class LockAwaiterBase {
   protected:
    friend class SharedMutexFair;

    explicit LockAwaiterBase(SharedMutexFair& mutex, LockType lockType) noexcept
        : mutex_(&mutex), nextAwaiter_(nullptr), lockType_(lockType) {}

    void resume() noexcept { continuation_.resume(); }
		// 等待的锁
    SharedMutexFair* mutex_;
    // 执行下一个awaiter，串连成链表
    LockAwaiterBase* nextAwaiter_;
    // 好像没用到
    LockAwaiterBase* nextReader_;
    // 协程的coroutine_handle
    coroutine_handle<> continuation_;
    // 需要获取锁类型
    LockType lockType_;
  };
  
  /* 
  对awaiter提供viaIfAsync方法，由于在锁释放后会会串行执行一些列的协程resume，使用viaIfAsync提供异步支持，保证resume的协程都在原指定的线程池运行，并且是异步运行，避免所有阻塞的协程串行
  */
  class LockOperation {
   public:
    explicit LockOperation(SharedMutexFair& mutex) noexcept : mutex_(mutex) {}

    auto viaIfAsync(folly::Executor::KeepAlive<> executor) const {
      return folly::coro::co_viaIfAsync(std::move(executor), Awaiter{mutex_});
    }

   private:
    SharedMutexFair& mutex_;
  };

  static constexpr std::size_t kUnlocked = 0;
  static constexpr std::size_t kExclusiveLockFlag = 1;
  static constexpr std::size_t kSharedLockCountIncrement = 2;
  /* 
  使用自旋锁的同步原语，当调用auto lock = state_.contextualLock()时，State被folly::SpinLock加锁保护
  当lock生命周期结束，自动释放锁
  */
  folly::Synchronized<State, folly::SpinLock> state_;
}
```

这里有一个核心结构`folly::Synchronized<State, folly::SpinLock>`。这里不展开启实现细节，我们只需要知道其是同步原语即可，其持有一个`state`数据，访问其中数据都应该通过`auto lock = state_.contextualLock()`的`lock`访问，`state`里面的数据都可以通过`lock`利用`->`操作符直接访问到。调用`auto lock = state_.contextualLock()`时，不仅获取到了对应存储的数据，同时获取了对应的`folly::SpinLock`锁，即获取的数据被`folly::SpinLock`锁保护。这里使用的是folly实现的自旋锁，这里也不展开介绍了，其可以理解为就是linux提供的自旋锁。自旋锁理论上是更废cpu的，但是这里为什么要是有自旋锁呢。这就要考虑自旋锁使用的场景了，自旋锁一般用于预期很快就能获取到锁的场景，这样可以避免像互斥锁一样需要将线程先挂起，再恢复的操作。这正是协程调度时所需要的，由于线程池较少，一般不会有很多锁竞争，即使有锁竞争也应该很快会获取到锁，并且要避免执行协程的线程阻塞，因此这里选取的是自旋锁。

对于可自动释放的锁来说，其实现就比不自动释放的增加了在`await_resume`中返回自动释放的class，其他没啥区别。下面我们来依次介绍锁的获取与释放。

### 读锁

```c++
inline SharedMutexFair::LockOperation<SharedMutexFair::LockSharedAwaiter>
SharedMutexFair::co_lock_shared() noexcept {
  return LockOperation<LockSharedAwaiter>{*this};
}
```

当`co_await co_lock_shared()`时，获取的`awaiter`是`LockSharedAwaiter`，其定义如下：

```c++
class LockSharedAwaiter : public LockAwaiterBase {
   public:
    explicit LockSharedAwaiter(SharedMutexFair& mutex) noexcept
        : LockAwaiterBase(mutex, LockType::SHARED) {}

    bool await_ready() noexcept { return mutex_->try_lock_shared(); }

    FOLLY_CORO_AWAIT_SUSPEND_NONTRIVIAL_ATTRIBUTES bool await_suspend(
        coroutine_handle<> continuation) noexcept {
      auto lock = mutex_->state_.contextualLock();

      // shared-lock can be acquired if it's either unlocked or it is
      // currently locked shared and there is no queued waiters.
      if (lock->lockedFlagAndReaderCount_ == kUnlocked ||
          (lock->lockedFlagAndReaderCount_ != kExclusiveLockFlag &&
           lock->waitersHead_ == nullptr)) {
        lock->lockedFlagAndReaderCount_ += kSharedLockCountIncrement;
        return false;
      }

      // Lock not available immediately.
      // Queue up for later resumption.
      continuation_ = continuation;
      *lock->waitersTailNext_ = this;
      lock->waitersTailNext_ = &nextAwaiter_;
      return true;
    }

    void await_resume() noexcept {}
  };
```

首先尝试获取读锁，如果获取成功，则继续执行协程。如果失败，则执行`await_suspend`。在`await_suspend`中，再次执行一次可以直接上锁的判断，如果可以上锁，则不suspend协程。否则，记录当前协程的`continuation_`，将当前协程加入到等待列表中。其中如下两句语句比较绕：

```c++
*lock->waitersTailNext_ = this;
lock->waitersTailNext_ = &nextAwaiter_;
```

第一句是把队尾的指针赋值为当前`awaiter`，关键是第二句，这里`waitersTailNext_`是一个双重指针，即`LockAwaiterBase**`这里将`waitersTailNext_`指向了当前awaiter的`nextAwaiter_`结构，则下次再向列表中添加元素时，执行的还是这两个语句，这时，第一条语句`*lock->waitersTailNext_ = this;`，就是将这一次的`nextAwaiter_`赋值为指向添加的`awaiter`。这样，每次对`*lock->waitersTailNext_`赋值，都是在对链表最后一个`awaiter`的`nextAwaiter_`赋值，以此达到串连所有`awaiter`的目的（妙啊）。这里还有个问题是起始指针，即`State`的`waitersHead_`变量，这就要再来看一下`State`的初始化了：

```
State() noexcept
        : lockedFlagAndReaderCount_(kUnlocked),
          waitersHead_(nullptr),
          waitersTailNext_(&waitersHead_) {}
```

可以看到，`waitersTailNext_`初始化执行`waitersHead_`，则第一次执行`*lock->waitersTailNext_`就是对`waitersHead_`赋值（好家伙，指针是被他玩明白了）。

至此，完成了等待协程`awaiter`的串连。

将等待读锁添加到等待链表后，当写锁释放时会遍历链表，对等待的协程加读锁。

### 释放读锁

释放读锁逻辑如下

```c++
void SharedMutexFair::unlock_shared() noexcept {
  LockAwaiterBase* awaitersToResume = nullptr;
  {
    // 获取自旋锁
    auto lockedState = state_.contextualLock();
    assert(lockedState->lockedFlagAndReaderCount_ >= kSharedLockCountIncrement);
    // 计数-2，标记释放读锁
    lockedState->lockedFlagAndReaderCount_ -= kSharedLockCountIncrement;
    // 如果还有锁在（这里一定是读锁），则直接返回
    if (lockedState->lockedFlagAndReaderCount_ != kUnlocked) {
      return;
    }
    // 如果已经没有锁了，则遍历等待队列，查询能够获得锁的协程，加锁
    awaitersToResume = unlockOrGetNextWaitersToResume(*lockedState);
  }
  // 唤醒协程
  resumeWaiters(awaitersToResume);
}
```

这里的核心逻辑是`unlockOrGetNextWaitersToResume`函数，其作用是获取可以获得锁的列表，其逻辑如下：

```c++
SharedMutexFair::LockAwaiterBase*
SharedMutexFair::unlockOrGetNextWaitersToResume(
    SharedMutexFair::State& state) noexcept {
  // 头指针指向等待队列头部
  auto* head = state.waitersHead_;
  // 不为空表示存在等等获取锁的协程
  if (head != nullptr) {
    // 如果第一个是要获取写锁，则直接标记锁状态为写锁，让其获得写锁
    if (head->lockType_ == LockType::EXCLUSIVE) {
      // 让头指针指向之后的列表
      state.waitersHead_ = std::exchange(head->nextAwaiter_, nullptr);
      // 标记状态为写锁被获取
      state.lockedFlagAndReaderCount_ = kExclusiveLockFlag;
    } else {
      // 如果第一个是读锁，则遍历出所有连续的读锁，加到队列中
      std::size_t newState = kSharedLockCountIncrement;

      // Scan for a run of SHARED lock types.
      auto* last = head;
      auto* next = last->nextAwaiter_;
      while (next != nullptr && next->lockType_ == LockType::SHARED) {
        last = next;
        next = next->nextAwaiter_;
        newState += kSharedLockCountIncrement;
      }
      // 输出对了最后是nullpter
      last->nextAwaiter_ = nullptr;
      // 标记获取了多少把读锁
      state.lockedFlagAndReaderCount_ = newState;
      // 维持头指针指向最后不满足条件的awaiter，要么是写锁，要么是nullptr
      state.waitersHead_ = next;
    }
    // 如果最后是空，则要维持waitersTailNext_指向waitersHead_，保证添加时正常
    if (state.waitersHead_ == nullptr) {
      state.waitersTailNext_ = &state.waitersHead_;
    }
  } else {
    // 如果为空，表示当前不需要加锁，标记状态
    state.lockedFlagAndReaderCount_ = kUnlocked;
  }

  return head;
}
```

可以看到，其逻辑是按照等等的头部属性来拉取满足条件的`awaiter`，同时加锁。

`resumeWaiters`逻辑较为简单：

```c++
void SharedMutexFair::resumeWaiters(LockAwaiterBase* awaiters) noexcept {
  while (awaiters != nullptr) {
    std::exchange(awaiters, awaiters->nextAwaiter_)->resume();
  }
}
```

遍历获取的`awaiters`，`resume`即可。但这里有个问题，如果`resume`协程后，协程串行执行，将会导致效率低下，即使协程本身绑定了`executor`，也不能保证被挂起后执行依然是异步的，这时就需要使用`co_viaIfAsync`方法，即在调用`co_await`时，对`awaiter`增加一层`co_viaIfAsync`封装，这就保证协程始终时异步协程（如果`executor`不为空），并且是被执行在指定的线程池上。这也是为什么返回的`awaiter`都由`LockOperation`包一层，因为其定义了`viaIfAsync`方法。对于`task`来说，这些是不必要的，但是如果是自己定义的协程`promise_type`就需要注意，执行锁获取应该使用

```c++
const folly::Executor::KeepAlive<> executor = co_await co_current_executor;
auto lock = co_await share_mutex_fair.co_scoped_lock_shared().viaIfAsync(executor);
```

避免被唤醒的协程被串行执行。

### 自动释放的读锁

自动释放的读锁不需要用户显示调用`unlock_shared()`，在返回值的生命周期结束会自动释放，接口是:

```c++
 [[nodiscard]] LockOperation<ScopedLockSharedAwaiter>
  co_scoped_lock_shared() noexcept;
```

其中实现如下：

```c++
  class ScopedLockSharedAwaiter : public LockSharedAwaiter {
   public:
    using LockSharedAwaiter::LockSharedAwaiter;

    [[nodiscard]] SharedLock<SharedMutexFair> await_resume() noexcept {
      LockSharedAwaiter::await_resume();
      return SharedLock<SharedMutexFair>{*mutex_, std::adopt_lock};
    }
  };
```

可以看到，其相对`LockSharedAwaiter`唯一区别是其增加了返回值，该返回值将`mutex_`包起来，在析构时，调用释放锁的函数：

```c++
  ~SharedLock() {
    if (locked_) {
      mutex_->unlock_shared();
    }
  }
```

其他与正常读锁没什么区别。

### 写锁

介绍完了读锁，写锁就简单很多了。获取锁接口有两个，一个会自动释放，一个不会，这里只简单介绍不自动释放的。

获取写锁：

```c++
inline SharedMutexFair::LockOperation<SharedMutexFair::LockAwaiter>
SharedMutexFair::co_lock() noexcept {
  return LockOperation<LockAwaiter>{*this};
}

class LockAwaiter : public LockAwaiterBase {
   public:
    explicit LockAwaiter(SharedMutexFair& mutex) noexcept
      // 标记为写锁
        : LockAwaiterBase(mutex, LockType::EXCLUSIVE) {}
    // 尝试获取写锁
    bool await_ready() noexcept { return mutex_->try_lock(); }

    FOLLY_CORO_AWAIT_SUSPEND_NONTRIVIAL_ATTRIBUTES bool await_suspend(
        coroutine_handle<> continuation) noexcept {
      auto lock = mutex_->state_.contextualLock();
      // 再次判断是否可以加写锁
      // Exclusive lock can only be acquired if it's currently unlocked.
      if (lock->lockedFlagAndReaderCount_ == kUnlocked) {
        lock->lockedFlagAndReaderCount_ = kExclusiveLockFlag;
        return false;
      }
      // 将当前的awaiter添加到链表末尾
      // Append to the end of the waiters queue.
      continuation_ = continuation;
      *lock->waitersTailNext_ = this;
      lock->waitersTailNext_ = &nextAwaiter_;
      return true;
    }

    void await_resume() noexcept {}
  };
```

释放写锁：

```c++
void SharedMutexFair::unlock() noexcept {
  LockAwaiterBase* awaitersToResume = nullptr;
  {
    auto lockedState = state_.contextualLock();
    assert(lockedState->lockedFlagAndReaderCount_ == kExclusiveLockFlag);
    awaitersToResume = unlockOrGetNextWaitersToResume(*lockedState);
  }

  resumeWaiters(awaitersToResume);
}
```

这里直接获取可以添加的队列而没有标记`lockedFlagAndReaderCount_`为`kUnlocked`是因为`unlockOrGetNextWaitersToResume`实现时是直接对`unlockOrGetNextWaitersToResume`赋值的，而不是再远基础上加减，因此没有必要执行这一步。

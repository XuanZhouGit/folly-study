
# 1 简介
Folly(Facebook Open-source Library)是facebook的类似std, boost的基础组件库，其中使用了很多c++ 11的新特性，主要目的是为了满足大规模高性能的需求。其中promise/future是folly实现的一个高性能的异步组件, c++ 11中std, boost都有promise/future的实现， 但std, boost中提供的接口都比较简单，Folly提供的接口比较多，支持via, then等方法，使链式调用更加方便，可读性更强

# 2 例子
## 2.1 makeFuture
```
string func1(int x) {
    return to_string(x);
}
void func2(const string& str) {
    cout << str << endl;
}
  
cout << "makeFuture" << endl;
auto f = makeFuture<int>(42).thenMulti(func1, func2);
cout << "Future ready? " << f.isReady()  << endl;
```

## 2.2 Future with Promise
```
cout << "making Promise" << endl;
Promise<int> p;
auto f = p.getSemiFuture();
folly::InlineExecutor e;
f.via(&e).then([](int x){ cout << "foo(" << x << ")" << endl; });
cout << "Future chain made" << endl;
cout << "Future ready? " << f.isReady() << endl;
cout << "fulfilling Promise" << endl;
p.setValue(42);
cout << "Future ready? " << f.isReady() << endl;
```
## 2.3 map/reduce
```
vector<int> vec = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
vector<Future<int>> futures;
for (int i : vec) {
    futures.push_back(makeFuture<int>(std::move(i)));
}
auto results = futures::map(futures, [](int val) {
    return val * 2;
});
auto f = reduce(results, 0, [](int sum, int&& val){
    return sum + val;
});
cout << f.get() << endl;
```
## 2.4 collect
```
vector<int> vec = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
SharedPromise<vector<int>> p;
CPUThreadPoolExecutor pool(0, 2);
auto f1 = p.getFuture().via(&pool).then([](const vector<int>& vals) {
    string str;
    for (int val : vals) {
        str += to_string(val) + " + ";
    }
    str.resize(str.size() - 3);
    str += " = ";
    return str;
});
auto f2 = p.getFuture().via(&pool).then([](const vector<int>& vals) {
    auto sum = 0;
    for (int val : vals) {
        sum += val;
    }
    return sum;
});
collectAll(f1, f2).then([](const tuple<Try<string>, Try<int>>& results){
    auto str = get<0>(results).value();
    str += to_string(get<1>(results).value());
    cout << str << endl;
});
p.setValue(vec);
pool.setNumThreads(2);
pool.join();
```

# 3 数据结构
## 3.1 总体结构
![structure](https://github.com/XuanZhouGit/folly-study/blob/master/folly1.PNG)

## 3.2 Core
Core是Promise和Future的公共成员变量，Core内部使用State维护类的当前状态，当条件满足时完成函数调用：
```
enum class State : uint8_t {
  Start,
  OnlyResult,
  OnlyCallback,
  Armed,
  Done,
};
```
状态转换流程：
![flow](https://github.com/XuanZhouGit/folly-study/blob/master/folly2.PNG)
callback的函数实现：
```
void doCallback() {
  Executor* x = executor_;
  // initialize, solely to appease clang's -Wconditional-uninitialized
  int8_t priority = 0;
  if (x) {
    if (!executorLock_.try_lock()) {
      executorLock_.lock();
    }
    x = executor_;
    priority = priority_;
    executorLock_.unlock();
  }
 
  if (x) {
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
    attached_ += 2;
    callbackReferences_ += 2;
    CoreAndCallbackReference guard_local_scope(this);
    CoreAndCallbackReference guard_lambda(this);
    try {
      if (LIKELY(x->getNumPriorities() == 1)) {
        x->add([core_ref = std::move(guard_lambda)]() mutable {
          auto cr = std::move(core_ref);
          Core* const core = cr.getCore();
          RequestContextScopeGuard rctx(core->context_);
          core->callback_(std::move(*core->result_));
        });
      } else {
        x->addWithPriority(
            [core_ref = std::move(guard_lambda)]() mutable {
              auto cr = std::move(core_ref);
              Core* const core = cr.getCore();
              RequestContextScopeGuard rctx(core->context_);
              core->callback_(std::move(*core->result_));
            },
            priority);
      }
    } catch (const std::exception& e) {
      ew = exception_wrapper(std::current_exception(), e);
    } catch (...) {
      ew = exception_wrapper(std::current_exception());
    }
    if (ew) {
      RequestContextScopeGuard rctx(context_);
      result_ = Try<T>(std::move(ew));
      callback_(std::move(*result_));
    }
  } else {
    attached_++;
    SCOPE_EXIT {
      callback_ = {};
      detachOne();
    };
    RequestContextScopeGuard rctx(context_);
    callback_(std::move(*result_));
  }
}
```
## 3.3 DeferredExecutor
DeferredExecutor是Future内部的Executor，通过add对func进行推迟调用，因为func会被异步执行，所以上面的doCallback用CoreAndCallbackReference来保证函数执行时，Core是有效的，baton是folly/synchronization里设计的信号量
```
 void add(Func func) override {
     auto state = state_.load(std::memory_order_acquire);
     if (state == State::HAS_FUNCTION) {
         // This means we are inside runAndDestroy, just run the function inline
         func();
         return;
     }
  
     func_ = std::move(func);
     std::shared_ptr<FutureBatonType> baton;
     do {
         if (state == State::HAS_EXECUTOR) {
             state_.store(State::HAS_FUNCTION, std::memory_order_release);
             executor_->add([this] { this->runAndDestroy(); });
             return;
         }
         if (state == State::DETACHED) {
             // Function destructor may trigger more functions to be added to the
             // Executor. They should be run inline.
             state = State::HAS_FUNCTION;
             func_ = nullptr;
             delete this;
             return;
         }
         if (state == State::HAS_BATON) {
             baton = baton_.copy();
         }
         assert(state == State::EMPTY || state == State::HAS_BATON);
     } while (!state_.compare_exchange_weak(
     state,
     State::HAS_FUNCTION,
     std::memory_order_release,
     std::memory_order_acquire));
  
     // After compare_exchange_weak is complete, we can no longer use this
     // object since it may be destroyed from another thread.
     if (baton) {
         baton->post();
     }
}
```
## 3.3 Promise
### 3.3.1 getSemiFuture
```
template <class T>
SemiFuture<T> Promise<T>::getSemiFuture() {
  throwIfRetrieved();
  retrieved_ = true;
  return SemiFuture<T>(core_);
}
```
getSemiFuture是Promise的核心函数，用来返回一个共享Promise的Core的SemiFuture，这个函数只能调用一次，也就是Promise只能有一个Future与之对应，Promise还有一个getFuture的接口，只是为了跟以前的接口兼容，现在不推荐使用，它可以不指定executor，使用内置的executor

### 3.3.2 SharedPromise
SharedPromise提供了所有Promise相同的接口，但可以多次调用getSemiFuture，其中makeFuture是生成Future的方式
```

template <class T>
SemiFuture<T> SharedPromise<T>::getSemiFuture() {
  std::lock_guard<std::mutex> g(mutex_);
  size_++;
  if (hasValue_) {
    return makeFuture<T>(Try<T>(try_));
  } else {
    promises_.emplace_back();
    if (interruptHandler_) {
      promises_.back().setInterruptHandler(interruptHandler_);
    }
    return promises_.back().getSemiFuture();
  }
}
```
## 3.4 SemiFuture
### 3.4.1 wait
```

template <class T>
SemiFuture<T>& SemiFuture<T>::wait() & {
  if (auto deferredExecutor = getDeferredExecutor()) {
    deferredExecutor->wait(); // wait for function adding to executor
    deferredExecutor->runAndDestroy(); // implement func and destory deferredExecutor
    this->core_->setExecutor(nullptr);
  } else {
    futures::detail::waitImpl(*this);
  }
  return *this;
}
  
template <class FutureType, typename T = typename FutureType::value_type>
void waitImpl(FutureType& f) {
  // short-circuit if there's nothing to do
  if (f.isReady()) {
    return;
  }
 
  FutureBatonType baton;
  f.setCallback_([&](const Try<T>& /* t */) { baton.post(); }); // post will wake update wait
  baton.wait();
  assert(f.isReady());
}
```
wait的功能是等待函数执行，如果设置了deferredExecutor，deferredExecutor会wait直到function ready, runAndDestroy执行function, 然后销毁deferredExecutor，如果没有设置Executor，在Core的状态机转换到Armed状态时，baton.post()会作为callback被调用，唤醒baton.wait()

### 3.4.2 then
then的核心函数是thenImplementation：
```
template <class T>
template <typename F, typename R, bool isTry, typename... Args>
typename std::enable_if<!R::ReturnsFuture::value, typename R::Return>::type
FutureBase<T>::thenImplementation(
    F&& func,
    futures::detail::argResult<isTry, F, Args...>) {
  static_assert(sizeof...(Args) <= 1, "Then must take zero/one argument");
  typedef typename R::ReturnsFuture::Inner B;
  this->throwIfInvalid();
  Promise<B> p;
  p.core_->setInterruptHandlerNoLock(this->core_->getInterruptHandler());
 
  // grab the Future now before we lose our handle on the Promise
  auto f = p.getFuture();
  f.core_->setExecutorNoLock(this->getExecutor());
  /* This is a bit tricky.
     We can't just close over *this in case this Future gets moved. So we
     make a new dummy Future. We could figure out something more
     sophisticated that avoids making a new Future object when it can, as an
     optimization. But this is correct.
     core_ can't be moved, it is explicitly disallowed (as is copying). But
     if there's ever a reason to allow it, this is one place that makes that
     assumption and would need to be fixed. We use a standard shared pointer
     for core_ (by copying it in), which means in essence obj holds a shared
     pointer to itself.  But this shouldn't leak because Promise will not
     outlive the continuation, because Promise will setException() with a
     broken Promise if it is destructed before completed. We could use a
     weak pointer but it would have to be converted to a shared pointer when
     func is executed (because the Future returned by func may possibly
     persist beyond the callback, if it gets moved), and so it is an
     optimization to just make it shared from the get-go.
     Two subtle but important points about this design. futures::detail::Core
     has no back pointers to Future or Promise, so if Future or Promise get
     moved (and they will be moved in performant code) we don't have to do
     anything fancy. And because we store the continuation in the
     futures::detail::Core, not in the Future, we can execute the continuation
     even after the Future has gone out of scope. This is an intentional design
     decision. It is likely we will want to be able to cancel a continuation
     in some circumstances, but I think it should be explicit not implicit
     in the destruction of the Future used to create it.
     */
  this->setCallback_(
      [state = futures::detail::makeCoreCallbackState(
           std::move(p), std::forward<F>(func))](Try<T>&& t) mutable {
        if (!isTry && t.hasException()) {
          state.setException(std::move(t.exception()));
        } else {
          state.setTry(makeTryWith(
              [&] { return state.invoke(t.template get<isTry, Args>()...); }));
        }
      });
  return f;
}
```
这个函数使用了一个比较tricky的方法，这里把then中要执行的func和next promise存在了CoreCallbackState中，如果当前promise调用setValue，这个包含next promise的callback会被调用，它的输入是当前promise的result, 这时function会被调用，通过state.setTry将结果存到当前Promise的core中，如果core中已经有callback，就会触发doCallback, 如果callback存的仍是then的这个function，就会继续触发里面Promise的core的doCallback，这样链式调用就能通过返回的future.get()拿到链式调用的
这个函
数使用了一个比较tricky的方法，这里把then中要执行的func和next promise存在了CoreCallbackState中，如果当前promise调用setValue，这个包含next promise的callback会被调用，它的输入是当前promise的result, 这时function会被调用，通过state.setTry将结果存到当前Promise的core中，如果core中已经有callback，就会触发doCallback, 如果callback存的仍是then的这个function，就会继续触发里面Promise的core的doCallback，这样链式调用就能通过返回的future.get()拿到链式调用的
### 3.4.3 via
```
template <class T>
inline Future<T> SemiFuture<T>::via(Executor* executor, int8_t priority) && {
  throwIfInvalid();
  if (!executor) {
    throwNoExecutor();
  }
 
  if (auto deferredExecutor = getDeferredExecutor()) {
    deferredExecutor->setExecutor(executor);
  }
 
  auto newFuture = Future<T>(this->core_);
  this->core_ = nullptr;
  newFuture.setExecutor(executor, priority);
  return newFuture;
}
```
因为Core是不允许复制的，因此Future也不允许左值复制，只是把core移到新的Future中，deferredExecutor->setExecutor的目的是为了将原来executor里的func移到新的executor中

# 4 QFuture
![QFuture](https://github.com/XuanZhouGit/folly-study/blob/master/folly3.PNG)

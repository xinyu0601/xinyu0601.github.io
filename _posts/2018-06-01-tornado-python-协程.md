---
title: 深入理解tornado框架之协程
layout: post
date: '2018-06-02 22:48:00'
image: "/assets/images/markdown.jpg"
tag:
- tornado
- 协程
- python
author: stanley huang
New field 11: "/assets/images/markdown.jpg"
category: blog
---

## 深入tornado中的协程

tornado的coroutine装饰器，使得回调函数可以用同步的方式实现，极大提高了代码的可读性。它的实现涉及到了yield，ioloop和Future的模块。

#### 基本概念

* yield的基本概念
*  Future的用法
*  ioloop的常用接口
* gen.coroutine的应用
*  gen.coroutine的源码
*  Runner类的实现

#### yield
python中使用yield实现了生成器函数，同样yield.send( )方法也实现了控制权在函数间的跳转。


#### Future

Future是用来在异步过程中，存放结果的容器。它可以在结果出来时，进行回调。

常用方法如下

`add_done_callback(self, fn)`： 添加回调函数fn，函数fn必须以future作为参数。

`set_result(self, result)`： 存放结果，此函数会执行回调函数。

`set_exception(self, exception)`：存放异常，此函数会执行回调函数。


####  ioloop

tornado框架的底层核心类，位于tornado的ioloop模块。功能方面类似win32窗口的消息循环。每个窗口可以绑定一个窗口过程。窗口过程主要是一个消息循环在执行。消息循环主要任务是利用PeekMessage系统调用，从消息队列中取出各种类型的消息，判断消息的类型，然后交给特定的消息handler进行执行。

tornado中的IOLoop与此相比具有很大的相似性，在协程运行环境中担任着协程调度器的角色, 和win32的消息循环本质上都是一种事件循环，等待事件，然后运行对应的事件处理器(handler）。不过IOLoop主要调度处理的是IO事件(如读，写，错误)。除此之外，还能调度callback和timeout事件。

常用接口

`add_future(self, future, callback)`： 当future返回结果时，会将callback登记到ioloop中，由ioloop在下次轮询时调用。

`add_callback(self, callback, *args, **kwargs)`： 将callback登记到ioloop中，由ioloop在下次轮询时调用。

`add_timeout(self, deadline, callback, *args, **kwargs)`： 将callback登记到ioloop中，由ioloop在指定的deadline时间调用。

`def call_at(self, when, callback, *args, **kwargs)`： 将callback登记到ioloop中，由ioloop在指定的when时间调用。

`def call_later(self, delay, callback, *args, **kwargs)`： 将callback登记到ioloop中， 由ioloop在指定的延迟delay秒后调用。

`add_handler(self, fd, handler, events)`： 将文件符fd的event事件和回调函数handler登记到ioloop中，当事件发生时，触发。

ioloop.py:

```python
def add_future(self, future, callback):

    assert is_future(future)
    callback = stack_context.wrap(callback)
    future.add_done_callback(
        lambda future: self.add_callback(callback, future))

```
我们可以看到，`add_future`为future添加了一个回调函数，当`future`完成的时候，也就是调用了`future.set_result`或`future.set_exception`后向`IOLoop`注册了callback函数，当`IOLoop`下一轮poll之前就会调用这个`callback函数`。下面是一个简单的使用例子，耗时任务使用另外的线程完成。

```python
import time
import threading
import tornado.ioloop
from tornado.concurrent import Future

ioloop = tornado.ioloop.IOLoop.current()

def long_task(future, sec=5):
    print("long task start")
    time.sleep(sec)
    future.set_result("long task done in %s sec" % sec)

def after_task_done(future):
    print(future.result())

def test_future():
    future = Future()
    threading.Thread(target=long_task, args=(future,)).start()
    ioloop.add_future(future, after_task_done)

if __name__ == "__main__":
    ioloop.add_callback(test_future)
    ioloop.start()
```


#### gen.coroutine的应用

普通的回调函数的方式：
```python
class AsyncHandler(RequestHandler):
    @asynchronous
    def get(self):
        http_client = AsyncHTTPClient()
        http_client.fetch("http://example.com",
                          callback=self.on_fetch)
 
    def on_fetch(self, response):
        do_something_with_response(response)
        self.render("template.html")
```
同步的方式
```python
class GenAsyncHandler(RequestHandler):
    @gen.coroutine
    def get(self):
        http_client = AsyncHTTPClient()
        response = yield http_client.fetch("http://example.com")
        do_something_with_response(response)
        self.render("template.html")
```
> 可以看出代码明显清晰，简单多了。如果有深层次的回调，效果会更明显。

#### gen.coroutine的源码

```python
def coroutine(func, replace_callback=True):
    return _make_coroutine_wrapper(func, replace_callback=True)
```

接着看`_make_coroutine_wrapper`的源码：

```python
def _make_coroutine_wrapper(func, replace_callback):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        future = TracebackFuture()

		# 有callback关键词的话为callback添加一个future，future完成时执行
        if replace_callback and 'callback' in kwargs:
            callback = kwargs.pop('callback')
            IOLoop.current().add_future(
                future, lambda future: callback(future.result()))
        
        # 调用func，根据结果的不同进行不同的处理
        try:
            result = func(*args, **kwargs)
        except (Return, StopIteration) as e:
            result = getattr(e, 'value', None)
        except Exception:
            future.set_exc_info(sys.exc_info())
            return future
        else:
            if isinstance(result, types.GeneratorType):
                # Inline the first iteration of Runner.run.  This lets us
                # avoid the cost of creating a Runner when the coroutine
                # never actually yields, which in turn allows us to
                # use "optional" coroutines in critical path code without
                # performance penalty for the synchronous case.
                try:
                    orig_stack_contexts = stack_context._state.contexts
                    
                    # 若result是一个生成器，则会调用一次next，在根据结果进行不同处理
                    yielded = next(result)
                    if stack_context._state.contexts is not orig_stack_contexts:
                        yielded = TracebackFuture()
                        yielded.set_exception(
                            stack_context.StackContextInconsistentError(
                                'stack_context inconsistency (probably caused '
                                'by yield within a "with StackContext" block)'))
                except (StopIteration, Return) as e:
                    future.set_result(getattr(e, 'value', None))
                except Exception:
                    future.set_exc_info(sys.exc_info())
                else:
                    Runner(result, future, yielded)
                try:
                    return future
                finally:
                    # Subtle memory optimization: if next() raised an exception,
                    # the future's exc_info contains a traceback which
                    # includes this stack frame.  This creates a cycle,
                    # which will be collected at the next full GC but has
                    # been shown to greatly increase memory usage of
                    # benchmarks (relative to the refcount-based scheme
                    # used in the absence of cycles).  We can avoid the
                    # cycle by clearing the local variable after we return it.
                    future = None
        future.set_result(result)
        return future
    return wrapper
```

我们可以看到，`add_future`为future添加了一个回调函数，当future完成的时候，也就是调用了`future.set_result`或`future.set_exception`后向`IOLoop`注册了`callback函数`，当`IOLoop`下一轮poll之前就会调用这个`callback函数`。下面是一个简单的使用例子，耗时任务使用另外的线程完成。

被`gen.coroutine`装饰的函数返回结果都会变成一个`Future`,若func不是一个生成器，则将结果或异常放入`future`中返回。若func返回一个生成器，调用`next`一次，如果还有后续，则交给`Runner`这个类来执行。结合demo上的来看，此时的`yielded`是`http_client.fetch("http://example.com")`执行都结果，必须是一个`Future`(或者`YieldPoint`，早期版本使用的，后来由Future代替，保留只为了兼容旧版本，这里不再讨论)，后面我们可以看到。

来看`Runner`类， 也在`gen.py`里面，在`Runner`里面最关键的是`run`和`handle_yield`方法。

```python
class Runner(object):
    def __init__(self, gen, result_future, first_yielded):
        self.gen = gen
        self.result_future = result_future
        self.future = _null_future
        self.yield_point = None
        self.pending_callbacks = None
        self.results = None
        self.running = False
        self.finished = False
        self.had_exception = False
        self.io_loop = IOLoop.current()

        self.stack_context_deactivate = None
        
        # 若first_yielded的future以完成，则最少会run一次
        if self.handle_yield(first_yielded):
            self.run()
            
    # ...省略...
    
    def handle_yield(self, yielded):
        # Lists containing YieldPoints require stack contexts;
        # other lists are handled via multi_future in convert_yielded.
        if (isinstance(yielded, list) and
                any(isinstance(f, YieldPoint) for f in yielded)):
            yielded = Multi(yielded)
        elif (isinstance(yielded, dict) and
              any(isinstance(f, YieldPoint) for f in yielded.values())):
            yielded = Multi(yielded)

        # 处理旧版本的YieldPoints的兼容，可以忽略...
		
		# future未完成则向ioloop注册这个future，当future完成时调用run继续执行
        if not self.future.done() or self.future is moment:
            self.io_loop.add_future(
                self.future, lambda f: self.run())
            return False
        return True

    def run(self):
        """Starts or resumes the generator, running until it reaches a
        yield point that is not ready.
        """
        if self.running or self.finished:
            return
        try:
            self.running = True
            while True:
                future = self.future
                if not future.done():
                    return
                self.future = None
                try:
                    orig_stack_contexts = stack_context._state.contexts
                    exc_info = None

                    try:
                        value = future.result()
                    except Exception:
                        self.had_exception = True
                        exc_info = sys.exc_info()
					
					# 有异常则设置异常
                    if exc_info is not None:
                        yielded = self.gen.throw(*exc_info)
                        exc_info = None
                   	# 将结果send回去继续执行
                    else:
                        yielded = self.gen.send(value)

                    if stack_context._state.contexts is not orig_stack_contexts:
                        self.gen.throw(
                            stack_context.StackContextInconsistentError(
                                'stack_context inconsistency (probably caused '
                                'by yield within a "with StackContext" block)'))
                except (StopIteration, Return) as e:
                    self.finished = True
                    self.future = _null_future
                    if self.pending_callbacks and not self.had_exception:
                        # If we ran cleanly without waiting on all callbacks
                        # raise an error (really more of a warning).  If we
                        # had an exception then some callbacks may have been
                        # orphaned, so skip the check in that case.
                        raise LeakedCallbackError(
                            "finished without waiting for callbacks %r" %
                            self.pending_callbacks)
                    self.result_future.set_result(getattr(e, 'value', None))
                    self.result_future = None
                    self._deactivate_stack_context()
                    return
                except Exception:
                    self.finished = True
                    self.future = _null_future
                    self.result_future.set_exc_info(sys.exc_info())
                    self.result_future = None
                    self._deactivate_stack_context()
                    return
                # handle_yield检查yielded，若还未完成则继续向IOLoop注册future等待完成。
                if not self.handle_yield(yielded):
                    return
        finally:
            self.running = False

		# .......
```

在`Runner`中`handle_yield`帮我们向`IOLoop`注册了`future`,当`future`完成时也就是`http_client.fetch`的结果返回之后，`IOLoop`就好执行`Runner.run()`使我们从断点继续执行。在`Runner.run()`里将`future`的结果取出，发送进生成器里面继续执行。
---
title: 深入理解 tornado 之底层 ioloop 实现
date: '2018-11-15 20:01:29'
layout: post
category: blog
image: "/assets/images/markdown.jpg"
tag:
- IOLoop
- Python
- tornado
---

## 概述
`tornado` 优秀的大并发处理能力得益于它的` web server `从底层开始就自己实现了一整套基于` epoll `的单线程异步架构。那么 `tornado.ioloop `就是 `tornado web server` 最底层的实现。
> ` ioloop` 实际上是对` epoll` 的封装，并加入了一些对上层事件的处理和` server `相关的底层处理。

## epoll
`ioloop` 的实现基于` epoll` ，那么什么是` epoll`？ `epoll `是`Linux内核`为处理大批量文件描述符而作了改进的` poll `。
那么什么又是 `poll` ？ 首先，我们回顾一下， `socket `通信时的服务端，当它接受（` accept `）一个连接并建立通信后（ `connection` ）就进行通信，而此时我们并不知道连接的客户端有没有信息发完。 这时候我们有两种选择：

1. 一直在这里等着直到收发数据结束；
1. 每隔一定时间来看看这里有没有数据；

第二种办法要比第一种好一些，多个连接可以统一在一定时间内轮流看一遍里面有没有数据要读写，看上去我们可以处理多个连接了，这个方式就是 poll / `select` 的解决方案。 看起来似乎解决了问题，但实际上，随着连接越来越多，轮询所花费的时间将越来越长，而服务器连接的 `socket` 大多不是活跃的，所以轮询所花费的大部分时间将是无用的。为了解决这个问题， `epoll` 被创造出来，它的概念和 `poll` 类似，不过每次轮询时，他只会把有数据活跃的 `socket` 挑出来轮询，这样在有大量连接时轮询就节省了大量时间。

对于 `epoll` 的操作，其实也很简单，只要 4 个 API 就可以完全操作它。

#### epoll_create

用来创建一个 `epoll `描述符（ 就是创建了一个` epoll `）

| 参数   |      含义      | 
|----------|:-------------:|
| EPOLL_CTL_ADD |  添加一个新的epoll事件 |
| EPOLL_CTL_DEL |    删除一个epoll事件   |  
| EPOLL_CTL_MOD | 改变一个事件的监听方式 |  

#### epoll_ctl

操作` epoll` 中的 `event`；可用参数有：

| 宏定义   |      含义      | 
|----------|:-------------:|
| EPOLLIN |  缓冲区满，有数据可读 |
| EPOLLOUT |   缓冲区空，可写数据   |  
| EPOLLERR | 发生错误 |  

#### epoll_wait

就是让 epoll 开始工作，里面有个参数 timeout，当设置为非 0 正整数时，会监听（阻塞） timeout 秒；设置为 0 时立即返回，设置为 -1 时一直监听。

在监听时有数据活跃的连接时其返回活跃的文件句柄列表（此处为 socket 文件句柄）。

#### close

关闭 `epoll`

现在了解了` epoll` 后，我们就可以来看 `ioloop `了


## tornado.ioloop
很多初学者一定好奇 `tornado `运行服务器最后那一句` tornado.ioloop.IOLoop.current().start()` 到底是干什么的。 我们先不解释作用，来看看这一句代码背后到底都在干什么。

```python

from __future__ import absolute_import, division, print_function, with_statement
 
import datetime
import errno
import functools
import heapq       # 最小堆
import itertools
import logging
import numbers
import os
import select
import sys
import threading
import time
import traceback
import math
 
from tornado.concurrent import TracebackFuture, is_future
from tornado.log import app_log, gen_log
from tornado.platform.auto import set_close_exec, Waker
from tornado import stack_context
from tornado.util import PY3, Configurable, errno_from_exception, timedelta_to_seconds
 
try:
    import signal
except ImportError:
    signal = None
 
 
if PY3:
    import _thread as thread
else:
    import thread
 
 
_POLL_TIMEOUT = 3600.0
 
 
class TimeoutError(Exception):
    pass
 
 
class IOLoop(Configurable):
    _EPOLLIN = 0x001
    _EPOLLPRI = 0x002
    _EPOLLOUT = 0x004
    _EPOLLERR = 0x008
    _EPOLLHUP = 0x010
    _EPOLLRDHUP = 0x2000
    _EPOLLONESHOT = (1 << 30)
    _EPOLLET = (1 << 31)
 
    # Our events map exactly to the epoll events
    NONE = 0
    READ = _EPOLLIN
    WRITE = _EPOLLOUT
    ERROR = _EPOLLERR | _EPOLLHUP
 
    # Global lock for creating global IOLoop instance
    _instance_lock = threading.Lock()
 
    _current = threading.local()
 
    @staticmethod
    def instance():
        if not hasattr(IOLoop, "_instance"):
            with IOLoop._instance_lock:
                if not hasattr(IOLoop, "_instance"):
                    # New instance after double check
                    IOLoop._instance = IOLoop()
        return IOLoop._instance
 
    @staticmethod
    def initialized():
        """Returns true if the singleton instance has been created."""
        return hasattr(IOLoop, "_instance")
 
    def install(self):
        assert not IOLoop.initialized()
        IOLoop._instance = self
 
    @staticmethod
    def clear_instance():
        """Clear the global `IOLoop` instance.
        .. versionadded:: 4.0
        """
        if hasattr(IOLoop, "_instance"):
            del IOLoop._instance
 
    @staticmethod
    def current(instance=True):
        current = getattr(IOLoop._current, "instance", None)
        if current is None and instance:
            return IOLoop.instance()
        return current
 
    def make_current(self):
        IOLoop._current.instance = self
 
    @staticmethod
    def clear_current():
        IOLoop._current.instance = None
 
    @classmethod
    def configurable_base(cls):
        return IOLoop
 
    @classmethod
    def configurable_default(cls):
        if hasattr(select, "epoll"):
            from tornado.platform.epoll import EPollIOLoop
            return EPollIOLoop
        if hasattr(select, "kqueue"):
            # Python 2.6+ on BSD or Mac
            from tornado.platform.kqueue import KQueueIOLoop
            return KQueueIOLoop
        from tornado.platform.select import SelectIOLoop
        return SelectIOLoop
 
    def initialize(self, make_current=None):
        if make_current is None:
            if IOLoop.current(instance=False) is None:
                self.make_current()
        elif make_current:
            if IOLoop.current(instance=False) is not None:
                raise RuntimeError("current IOLoop already exists")
            self.make_current()
 
    def close(self, all_fds=False):
        raise NotImplementedError()
 
    def add_handler(self, fd, handler, events):
        raise NotImplementedError()
 
    def update_handler(self, fd, events):
        raise NotImplementedError()
 
    def remove_handler(self, fd):
        raise NotImplementedError()
 
    def set_blocking_signal_threshold(self, seconds, action):
        raise NotImplementedError()
 
    def set_blocking_log_threshold(self, seconds):
        self.set_blocking_signal_threshold(seconds, self.log_stack)
 
    def log_stack(self, signal, frame):
        gen_log.warning('IOLoop blocked for %f seconds in\n%s',
                        self._blocking_signal_threshold,
                        ''.join(traceback.format_stack(frame)))
 
    def start(self):
        raise NotImplementedError()
 
    def _setup_logging(self):
        if not any([logging.getLogger().handlers,
                    logging.getLogger('tornado').handlers,
                    logging.getLogger('tornado.application').handlers]):
            logging.basicConfig()
 
    def stop(self):
        raise NotImplementedError()
 
    def run_sync(self, func, timeout=None):
        future_cell = [None]
 
        def run():
            try:
                result = func()
                if result is not None:
                    from tornado.gen import convert_yielded
                    result = convert_yielded(result)
            except Exception:
                future_cell[0] = TracebackFuture()
                future_cell[0].set_exc_info(sys.exc_info())
            else:
                if is_future(result):
                    future_cell[0] = result
                else:
                    future_cell[0] = TracebackFuture()
                    future_cell[0].set_result(result)
            self.add_future(future_cell[0], lambda future: self.stop())
        self.add_callback(run)
        if timeout is not None:
            timeout_handle = self.add_timeout(self.time() + timeout, self.stop)
        self.start()
        if timeout is not None:
            self.remove_timeout(timeout_handle)
        if not future_cell[0].done():
            raise TimeoutError('Operation timed out after %s seconds' % timeout)
        return future_cell[0].result()
 
    def time(self):
        return time.time()
...

```

`IOLoop `类首先声明了 `epoll` 监听事件的宏定义，当然，如前文所说，我们只要关心其中的 `EPOLLIN 、 EPOLLOUT 、 EPOLLERR` 就行。


现在版本的 tornado.ioloop 则继承自 `Configurable` 看起来现在的` IOLoop` 已经成为了一个基类，只定义了接口。 所以接着看 `Configurable` 代码：

#### tornado.util.Configurable

```python

class Configurable(object):
    __impl_class = None
    __impl_kwargs = None
 
    def __new__(cls, *args, **kwargs):
        base = cls.configurable_base()
        init_kwargs = {}
        if cls is base:
            impl = cls.configured_class()
            if base.__impl_kwargs:
                init_kwargs.update(base.__impl_kwargs)
        else:
            impl = cls
        init_kwargs.update(kwargs)
        instance = super(Configurable, cls).__new__(impl)
        # initialize vs __init__ chosen for compatibility with AsyncHTTPClient
        # singleton magic.  If we get rid of that we can switch to __init__
        # here too.
        instance.initialize(*args, **init_kwargs)
        return instance
 
    @classmethod
    def configurable_base(cls):
        """Returns the base class of a configurable hierarchy.
 
        This will normally return the class in which it is defined.
        (which is *not* necessarily the same as the cls classmethod parameter).
        """
        raise NotImplementedError()
 
    @classmethod
    def configurable_default(cls):
        """Returns the implementation class to be used if none is configured."""
        raise NotImplementedError()
 
    def initialize(self):
        """Initialize a `Configurable` subclass instance.
 
        Configurable classes should use `initialize` instead of ``__init__``.
 
        .. versionchanged:: 4.2
           Now accepts positional arguments in addition to keyword arguments.
        """
 
    @classmethod
    def configure(cls, impl, **kwargs):
        """Sets the class to use when the base class is instantiated.
 
        Keyword arguments will be saved and added to the arguments passed
        to the constructor.  This can be used to set global defaults for
        some parameters.
        """
        base = cls.configurable_base()
        if isinstance(impl, (unicode_type, bytes)):
            impl = import_object(impl)
        if impl is not None and not issubclass(impl, cls):
            raise ValueError("Invalid subclass of %s" % cls)
        base.__impl_class = impl
        base.__impl_kwargs = kwargs
 
    @classmethod
    def configured_class(cls):
        """Returns the currently configured class."""
        base = cls.configurable_base()
        if cls.__impl_class is None:
            base.__impl_class = cls.configurable_default()
        return base.__impl_class
 
    @classmethod
    def _save_configuration(cls):
        base = cls.configurable_base()
        return (base.__impl_class, base.__impl_kwargs)
 
    @classmethod
    def _restore_configuration(cls, saved):
        base = cls.configurable_base()
        base.__impl_class = saved[0]
        base.__impl_kwargs = saved[1]
```

#### start
`ioloop`最核心的部分：

```python

def start(self):
        if self._running:       # 判断是否已经运行
            raise RuntimeError("IOLoop is already running")
        self._setup_logging()
        if self._stopped:
            self._stopped = False  # 设置停止为假
            return
        old_current = getattr(IOLoop._current, "instance", None)
        IOLoop._current.instance = self
        self._thread_ident = thread.get_ident()  # 获得当前线程标识符
        self._running = True # 设置运行
 
        old_wakeup_fd = None
        if hasattr(signal, 'set_wakeup_fd') and os.name == 'posix':
            try:
                old_wakeup_fd = signal.set_wakeup_fd(self._waker.write_fileno())
                if old_wakeup_fd != -1:
                    signal.set_wakeup_fd(old_wakeup_fd)
                    old_wakeup_fd = None
            except ValueError:
                old_wakeup_fd = None
 
        try:
            while True:  # 服务器进程正式开始，类似于其他服务器的 serve_forever
                with self._callback_lock: # 加锁，_callbacks 做为临界区不加锁进行读写会产生脏数据
                    callbacks = self._callbacks # 读取 _callbacks
                    self._callbacks = []. # 清空 _callbacks
                due_timeouts = [] # 用于存放这个周期内已过期（ 已超时 ）的任务
                if self._timeouts: # 判断 _timeouts 里是否有数据
                    now = self.time() # 获取当前时间，用来判断 _timeouts 里的任务有没有超时
                    while self._timeouts: # _timeouts 有数据时一直循环, _timeouts 是个最小堆，第一个数据永远是最小的， 这里第一个数据永远是最接近超时或已超时的
                        if self._timeouts[0].callback is None: # 超时任务无回调
                            heapq.heappop(self._timeouts) # 直接弹出
                            self._cancellations -= 1 # 超时计数器 －1
                        elif self._timeouts[0].deadline  512
                            and self._cancellations > (len(self._timeouts) >> 1)):  # 当超时计数器大于 512 并且 大于 _timeouts 长度一半（ >> 为右移运算， 相当于十进制数据被除 2 ）时，清零计数器，并剔除 _timeouts 中无 callbacks 的任务
                        self._cancellations = 0
                        self._timeouts = [x for x in self._timeouts
                                          if x.callback is not None]
                        heapq.heapify(self._timeouts) # 进行 _timeouts 最小堆化
 
                for callback in callbacks:
                    self._run_callback(callback) # 运行 callbacks 里所有的 calllback
                for timeout in due_timeouts:
                    if timeout.callback is not None:
                        self._run_callback(timeout.callback) # 运行所有已过期任务的 callback
                callbacks = callback = due_timeouts = timeout = None # 释放内存
 
                if self._callbacks: # _callbacks 里有数据时
                    poll_timeout = 0.0 # 设置 epoll_wait 时间为0（ 立即返回 ）
                elif self._timeouts: # _timeouts 里有数据时
                    poll_timeout = self._timeouts[0].deadline - self.time() 
                    # 取最小过期时间当 epoll_wait 等待时间，这样当第一个任务过期时立即返回
                    poll_timeout = max(0, min(poll_timeout, _POLL_TIMEOUT))
                    # 如果最小过期时间大于默认等待时间 _POLL_TIMEOUT ＝ 3600，则用 3600，如果最小过期时间小于0 就设置为0 立即返回。
                else:
                    poll_timeout = _POLL_TIMEOUT # 默认 3600 s 等待时间
 
                if not self._running: # 检查是否有系统信号中断运行，有则中断，无则继续
                    break
 
                if self._blocking_signal_threshold is not None:
                    signal.setitimer(signal.ITIMER_REAL, 0, 0) # 开始 epoll_wait 之前确保 signal alarm 都被清空（ 这样在 epoll_wait 过程中不会被 signal alarm 打断 ）
 
                try:
                    event_pairs = self._impl.poll(poll_timeout) # 获取返回的活跃事件队
                except Exception as e:
                    if errno_from_exception(e) == errno.EINTR:
                        continue
                    else:
                        raise
 
                if self._blocking_signal_threshold is not None:
                    signal.setitimer(signal.ITIMER_REAL,
                                     self._blocking_signal_threshold, 0) #  epoll_wait 结束， 再设置 signal alarm
                self._events.update(event_pairs) # 将活跃事件加入 _events
                while self._events:
                    fd, events = self._events.popitem() # 循环弹出事件
                    try:
                        fd_obj, handler_func = self._handlers[fd] # 处理事件
                        handler_func(fd_obj, events)
                    except (OSError, IOError) as e:
                        if errno_from_exception(e) == errno.EPIPE:
                            pass
                        else:
                            self.handle_callback_exception(self._handlers.get(fd))
                    except Exception:
                        self.handle_callback_exception(self._handlers.get(fd))
                fd_obj = handler_func = None
 
        finally:
            self._stopped = False # 确保发生异常也继续运行
            if self._blocking_signal_threshold is not None:
                signal.setitimer(signal.ITIMER_REAL, 0, 0) # 清空 signal alarm
            IOLoop._current.instance = old_current 
            if old_wakeup_fd is not None:
                signal.set_wakeup_fd(old_wakeup_fd)   # 和 start 开头部分对应，但是不是很清楚作用，求老司机带带路
```
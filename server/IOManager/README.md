# IO协程调度模块

> IO协程调度解决了调度器在idle状态下忙等待导致CPU占用率高的问题。IO协程调度器使用一对管道fd来tickle调度协程，当调度器空闲时，idle协程通过epoll_wait阻塞在管道的读描述符上，等管道的可读事件。添加新任务时，tickle方法写管道，idle协程检测到管道可读后退出，调度器执行调度。

## 1. 模块设计

IO协程调度模块基于epoll实现，只支持Linux平台。对每个fd，支持两类事件，一类是可读事件，对应EPOLLIN，一类是可写事件，对应EPOLLOUT，事件枚举值直接继承自epoll。

当然epoll本身除了支持了EPOLLIN和EPOLLOUT两类事件外，还支持其他事件，比如EPOLLRDHUP, EPOLLERR, EPOLLHUP等，对于这些事件，做法是将其进行归类，分别对应到EPOLLIN和EPOLLOUT中，也就是所有的事件都可以表示为可读或可写事件，甚至有的事件还可以同时表示可读及可写事件，比如EPOLLERR事件发生时，fd将同时触发可读和可写事件。

对于IO协程调度来说，每次调度都包含一个三元组信息，分别是描述符-事件类型（可读或可写）-回调函数，调度器记录全部需要调度的三元组信息，其中描述符和事件类型用于epoll_wait，回调函数用于协程调度。这个三元组信息在源码上通过FdContext结构体来存储，在执行epoll_wait时通过epoll_event的私有数据指针data.ptr来保存FdContext结构体信息。

IO协程调度器在idle时会epoll_wait所有注册的fd，如果有fd满足条件，epoll_wait返回，从私有数据中拿到fd的上下文信息，并且执行其中的回调函数。（实际是idle协程只负责收集所有已触发的fd的回调函数并将其加入调度器的任务队列，真正的执行时机是idle协程退出后，调度器在下一轮调度时执行）

与协程调度器不一样的是，IO协程调度器支持取消事件。取消事件表示不关心某个fd的某个事件了，如果某个fd的可读或可写事件都被取消了，那这个fd会从调度器的epoll_wait中删除。

## 2. 模块实现

### 2.1 IOManager

> IO协程调度器

```C++
class IOManager : public Scheduler {
public:
    typedef std::shared_ptr<IOManager> ptr;
    typedef RWMutex RWMutexType;
...
}
```

### 2.2 读写事件

> IO事件，只关心socket fd的读和写事件，其他epoll事件会归类到这两类事件中

```C++
enum Event {
    /// 无事件
    NONE = 0x0,
    /// 读事件(EPOLLIN)
    READ = 0x1,
    /// 写事件(EPOLLOUT)
    WRITE = 0x4,
};
```

### 2.3 三元组

> 描述符-事件类型-回调函数三元组的定义，这个三元组也称为fd上下文，使用结构体FdContext来表示。由于fd有可读和可写两种事件，每种事件的回调函数也可以不一样，所以每个fd都需要保存两个事件类型-回调函数组合。

```C++
struct FdContext {
    typedef Mutex MutexType;
    struct EventContext {
        Scheduler *scheduler = nullptr;
        Fiber::ptr fiber;
        std::function<void()> cb;
    };
 
    EventContext &getEventContext(Event event);
 
    void resetEventContext(EventContext &ctx);

    void triggerEvent(Event event);
 
    EventContext read;
    EventContext write;
    int fd = 0;
    Event events = NONE;
    MutexType mutex;
```

### 2.4 事件处理

- 注册事件回调addEvent
- 删除事件回调delEvent
- 取消事件回调cancelEvent
- 以及取消全部事件cancelAll



## 3. 总结

1. IO协程调度模块可分为两部分，第一部分是对协程调度器的改造，将epoll与协程调度融合，重新实现tickle和idle，并保证原有的功能不变。第二部分是基于epoll实现IO事件的添加、删除、调度、取消等功能。
2. IO协程调度关注的是FdContext信息，也就是描述符-事件-回调函数三元组，IOManager需要保存所有关注的三元组，并且在epoll_wait检测到描述符事件就绪时执行对应的回调函数。
3. FdContext的寻址问题，直接使用fd的值作为FdContext数组的下标，这样可以快速找到一个fd对应的FdContext。由于关闭的fd会被重复利用，所以这里也不用担心FdContext数组膨胀太快，或是利用率低的问题。
4. IO协程调度器的退出，不但所有协程要完成调度，所有IO事件也要完成调度。
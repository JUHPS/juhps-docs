# Hook模块

> hook系统底层和socket相关的API，socket IO相关的API，以及sleep系列的API。hook的开启控制是线程粒度的，可以自由选择。通过hook模块，可以使一些不具异步功能的API，展现出异步的性。

## 1. 模块设计

hook功能以线程为单位，可自由设置当前线程是否使用hook。默认情况下，协程调度器的调度线程会开启hook，而其他线程则不会开启。

以下函数进行了hook，并且只对socket fd进行了hook，如果操作的不是socket fd，那会直接调用系统原本的API，而不是hook之后的API：

- sleep
- usleep
- nanosleep
- socket
- connect
- accept
- read
- readv
- recv
- recvfrom
- recvmsg
- write
- writev
- send
- sendto
- sendmsg
- close
- fcntl
- ioctl
- getsockopt
- setsockopt

增加了一个 connect_with_timeout 接口用于实现带超时的connect。

为了管理所有的socket fd，设计了一个FdManager类来记录所有分配过的fd的上下文，这是一个单例类，每个socket fd上下文记录了当前fd的读写超时，是否设置非阻塞等信息。

关于hook模块和IO协程调度的整合。一共有三类接口需要hook，如下：

1. sleep延时系列接口，包括sleep/usleep/nanosleep。对于这些接口的hook，只需要给IO协程调度器注册一个定时事件，在定时事件触发后再继续执行当前协程即可。当前协程在注册完定时事件后即可yield让出执行权。

2. socket IO系列接口，包括read/write/recv/send...等，connect及accept也可以归到这类接口中。这类接口的hook首先需要判断操作的fd是否是socket fd，以及用户是否显式地对该fd设置过非阻塞模式，如果不是socket fd或是用户显式设置过非阻塞模式，那么就不需要hook了，直接调用操作系统的IO接口即可。如果需要hook，那么首先在IO协程调度器上注册对应的读写事件，等事件发生后再继续执行当前协程。当前协程在注册完IO事件即可yield让出执行权。

3. socket/fcntl/ioctl/close等接口，这类接口主要处理的是边缘情况，比如分配fd上下文，处理超时及用户显式设置非阻塞问题。


## 2. 模块实现

### 2.1 FdCtx

> 文件句柄上下文类，管理文件句柄类型（是否socket，是否阻塞，是否关闭，读/写超时时间）

FdCtx类在用户态记录了fd的读写超时和非阻塞信息，其中非阻塞包括用户显式设置的非阻塞和hook内部设置的非阻塞，区分这两种非阻塞可以有效应对用户对fd设置/获取NONBLOCK模式的情形。


```C++
class FdCtx : public std::enable_shared_from_this<FdCtx> {
public:
    typedef std::shared_ptr<FdCtx> ptr;
    FdCtx(int fd);
    ~FdCtx();
    ....
private:
    bool m_isInit: 1;
    bool m_isSocket: 1;
    bool m_sysNonblock: 1;
    bool m_userNonblock: 1;
    bool m_isClosed: 1;
    int m_fd;
    uint64_t m_recvTimeout;
    uint64_t m_sendTimeout;
};
```

### 2.2 FdManager

> 文件句柄管理类

FdManager类对FdCtx的寻址采用了和IOManager中对FdContext的寻址一样的寻址方式，直接用fd作为数组下标进行寻址。


```C++
class FdManager {
public:
    typedef RWMutex RWMutexType;

    FdManager();
    FdCtx::ptr get(int fd, bool auto_create = false);
    void del(int fd);
private:
    RWMutexType m_mutex;
    std::vector<FdCtx::ptr> m_datas;
};
```

### 2.3 hook

首先定义线程局部变量t_hook_enable，用于表示当前线程是否启用hook，使用线程局部变量表示hook模块是线程粒度的，各个线程可单独启用或关闭hook。然后是获取各个被hook的接口的原始地址， 这里要借助dlsym来获取。

具体通过宏来简化代码：

```C++
#define HOOK_FUN(XX) \
    XX(sleep) \
    XX(usleep) \
    XX(nanosleep) \
    XX(socket) \
    XX(connect) \
    XX(accept) \
    XX(read) \
    XX(readv) \
    XX(recv) \
    XX(recvfrom) \
    XX(recvmsg) \
    XX(write) \
    XX(writev) \
    XX(send) \
    XX(sendto) \
    XX(sendmsg) \
    XX(close) \
    XX(fcntl) \
    XX(ioctl) \
    XX(getsockopt) \
    XX(setsockopt)
 
extern "C" {
#define XX(name) name ## _fun name ## _f = nullptr;
    HOOK_FUN(XX);
#undef XX
}
 
void hook_init() {
    static bool is_inited = false;
    if(is_inited) {
        return;
    }
#define XX(name) name ## _f = (name ## _fun)dlsym(RTLD_NEXT, #name);
    HOOK_FUN(XX);
#undef XX
}
```

hook_init() 放在一个静态对象的构造函数中调用，这表示在main函数运行之前就会获取各个符号的地址并保存在全局变量中。



## 3. 总结

1. 由于定时器模块只支持毫秒级定时，所以被hook后的nanosleep()实际精度只能达到毫秒级，而不是纳秒级。
2. 非调度线程不支持启用hook。
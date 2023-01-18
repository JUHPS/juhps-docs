# 线程模块

> 提供线程类和线程同步类，基于pthread线程库实现。

## 1. 线程类

### 1.1 Thread

> 构造函数传入线程入口函数和线程名称，线程入口函数类型为void()，如果带参数，则需要用std::bind进行绑定。线程类构造之后线程即开始运行，构造函数在线程真正开始运行之后返回。


线程类不提供默认复制、拷贝构造与左值构造，所以将三个默认函数删除掉。
线程类提供五个属性:
1. m_id为pid_t类型，为全局线程id，可以用syscall(SYS_gettid)获取。
2. m_thread是pthread_t类型，在当前进程中唯一标识线程，可以用pthread_self()函数获取。
3. m_cb是function类型的属性，是当前线程最终调用的函数。
4. m_name是当前线程名称可以用于日志输出，调试等等。
5. m_semaphere是信号量，保护临界资源。

```C++
class Thread {
public:
    typedef std::shared_ptr<Thread> ptr;
    Thread(std::function<void()> cb, const std::string& name);
    ~Thread();

    pid_t getId() const { return m_id; }
    const std::string& getName() const { return m_name; }

    void join();
    static Thread* GetThis();
    static const std::string& GetName();
    static void SetName(const std::string& name);
private:
    Thread(const Thread&) = delete;
    Thread(const Thread&&) = delete;
    Thread& operator=(const Thread&) = delete;

    static void* run(void* arg);
private:
    pid_t m_id = -1;
    pthread_t m_thread = 0;
    std::function<void()> m_cb;
    std::string m_name;

    Semaphere m_semaphere;
};
```

Thread类中提供三个静态方法，用于获取当前`t_thread`和`t_thread_name`。

`Thread::GetThis()`、`Thread::GetName()`、`Thread::run()`在`pthread_create()`之前注册，在`run()`中会完成对两个静态属性的初始化。

`pthread_setname_np(pthread_self(), thread->m_name.substr(0, 15).c_str())`能够完成在操作系统设置线程名称的任务。

`m_semaphore.wait()`，在`run()`中调用真正线程函数之前调用`thread->m_semaphore.notify()`。是为了保证在完成线程类的构造函数之前，线程已经开始执行了，不然在构造完成之后，线程可能还没开始执行，在离开作用域时会执行析构函数，造成线程还没开始就被pthread_detach()了。

```C++
static thread_local Thread* t_thread = nullptr;
static thread_local std::string t_thread_name = "UNKNOW";

Thread* Thread::GetThis() {
    return t_thread;
}
const std::string& Thread::GetName() {
    return t_thread_name;
}

void Thread::SetName(const std::string& name) {
    if (t_thread) {
        t_thread -> m_name = name;
    } else {
        t_thread_name = name;
    }
}

void* Thread::run(void* arg) {
    Thread* thread = (Thread*)arg;
    t_thread = thread;
    t_thread_name = thread -> m_name;
    thread -> m_id = jujimeizuo::GetThreadId();
    pthread_setname_np(pthread_self(), thread -> m_name.substr(0, 15).c_str());

    std::function<void()> cb;
    cb.swap(thread -> m_cb);

    thread -> m_semaphere.notify();

    cb();
    return 0;
}
```

## 2. 线程同步类

### 2.1 Semaphore

> 计数信号量，基于sem_t实现

信号量同样不允许默认拷贝，复制构造，左值构造，所以将默认函数删除。
其中只用一个属性为m_semaphore，为sem_t类型。

在构造函数中`sem_init()`，在析构函数中`sem_destroy()`，通过`wait()获取`，`notify()`释放。

```C++
class Semaphere {
public:
    Semaphere(uint32_t count = 0);
    ~Semaphere();
    
    void wait();
    void notify();
private:
    Semaphere(const Semaphere&) = delete;
    Semaphere(const Semaphere&&) = delete;
    Semaphere& operator=(const Semaphere&) = delete;

private:
    sem_t m_semaphere;
};
```

### 2.2 Mutex

> 互斥锁，基于pthread_mutex_t实现

### 2.3 RWMutex

> 读写锁，基于pthread_rwlock_t实现

### 2.4 Spinlock

> 自旋锁，基于pthread_spinlock_t实现

### 2.5 CASLock

> 原子锁，基于std::atomic_flag实现

### 2.6 ScopedLockImpl

> 锁的统一模版封装，对所有锁类统一使用，如果需要换锁，直接替换相应关键字即可。

## 3. 总结

在日志系统与配置系统使用互斥量是为了保证线程安全，因为日志系统写多读少所以使用`Mutex`，但有性能要求，所以使用的是`Spinlock`,在配置系统中是读多写少，所以使用的是读写锁`RWmutex`。速度大概在每秒25M左右。

在所有需要读取/更改数据的地方都要加锁。
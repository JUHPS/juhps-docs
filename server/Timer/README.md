# 定时器模块

> 基于epoll超时实现定时器功能，精度毫秒级，支持在指定超时时间结束之后执行回调函数。通过定时器可以实现给服务器注册定时事件，这是服务器上经常要处理的一类事件，比如3秒后关闭一个连接，或是定期检测一个客户端的连接状态。


## 1. 模块设计

定时器采用最小堆设计，所有定时器根据绝对的超时时间点进行排序，每次取出离当前时间最近的一个超时时间点，计算出超时需要等待的时间，然后等待超时。超时时间到后，获取当前的绝对时间点，然后把最小堆里超时时间点小于这个时间点的定时器都收集起来，执行它们的回调函数。

注意，在注册定时事件时，一般提供的是相对时间，比如相对当前时间3秒后执行。会根据传入的相对时间和当前的绝对时间计算出定时器超时时的绝对时间点，然后根据这个绝对时间点对定时器进行最小堆排序。因为依赖的是系统绝对时间，所以需要考虑校时因素。

定时器的超时等待基于epoll_wait，精度只支持毫秒级，因为epoll_wait的超时精度也只有毫秒级。

关于定时器和IO协程调度器的整合。IO协程调度器的idle协程会在调度器空闲时阻塞在epoll_wait上，等待IO事件发生。在之前的代码里，epoll_wait具有固定的超时时间，这个值是5秒钟。加入定时器功能后，epoll_wait的超时时间改用当前定时器的最小超时时间来代替。epoll_wait返回后，根据当前的绝对时间把已超时的所有定时器收集起来，执行它们的回调函数。

由于epoll_wait的返回并不一定是超时引起的，也有可能是IO事件唤醒的，所以在epoll_wait返回后不能想当然地假设定时器已经超时了，而是要再判断一下定时器有没有超时，这时绝对时间的好处就体现出来了，通过比较当前的绝对时间和定时器的绝对超时时间，就可以确定一个定时器到底有没有超时。

## 2. 模块实现

### 2.1 Timer

> 定时器

```C++
class TimerManager;
class Timer : public std::enable_shared_from_this<Timer> {
friend class TimerManager;
public:
    typedef std::shared_ptr<Timer> ptr;

    bool cancel();
    bool refresh();
    bool reset(uint64_t ms, bool from_now);
private:
    Timer(uint64_t ms, std::function<void()> cb,
          bool recurring, TimerManager* manager);
    Timer(uint64_t next);
private:
    bool m_recurring = false;
    uint64_t m_ms = 0;
    uint64_t m_next = 0;
    std::function<void()> m_cb;
    TimerManager* m_manager = nullptr;
private:
    struct Comparator {
        bool operator()(const Timer::ptr& lhs, const Timer::ptr& rhs) const;
    };
};
```

### 2.2 TimerManager

> 定时器管理器

```C++
class TimerManager {
friend class Timer;

public:
    typedef RWMutex RWMutexType;
 
    TimerManager();
    virtual ~TimerManager();
    Timer::ptr addTimer(uint64_t ms, std::function<void()> cb
                        ,bool recurring = false);
     Timer::ptr addConditionTimer(uint64_t ms, std::function<void()> cb
                        ,std::weak_ptr<void> weak_cond
                        ,bool recurring = false);
    uint64_t getNextTimer();
    void listExpiredCb(std::vector<std::function<void()> >& cbs);
    bool hasTimer();

protected:
    virtual void onTimerInsertedAtFront() = 0;
    void addTimer(Timer::ptr val, RWMutexType::WriteLock& lock);

private:
    bool detectClockRollover(uint64_t now_ms);

private:
    RWMutexType m_mutex;
    std::set<Timer::ptr, Timer::Comparator> m_timers;
    bool m_tickled = false;
    uint64_t m_previouseTime = 0;
};
```
IOManager也需要继承TimerManager获得一个定时器管理模块。

```C++
class IOManager : public Scheduler, public TimerManager {};
```

## 3. 总结

1. 创建定时器时只传入了相对超时时间，内部要先进行转换，根据当前时间把相对时间转化成绝对时间。
2. 支持创建条件定时器，也就是在创建定时器时绑定一个变量，在定时器触发时判断一下该变量是否仍然有效，如果变量无效，那就取消触发。

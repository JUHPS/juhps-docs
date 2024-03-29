# 环境变量模块

> 提供程序运行时的环境变量管理功能。这里的环境变量不仅包括系统环境变量，还包括程序自定义环境变量，命令行参数，帮助选项与描述，以及程序运行路径相关的信息。

所谓环境变量就是程序运行时可直接获取和设置的一组变量，它们往往代表一些特定的含义。所有的环境变量都以key-value的形式存储，key和value都是字符串形式。这里可以参考系统环境变量来理解，在程序运行时，可以通过调用getenv()/setenv()接口来获取/设置系统环境变量，比如getenv("PWD")来获取当前路径。在shell中可以通过printenv命令来打印当前所有的环境变量，并且在当前shell中运行的所有程序都共享这组环境变量值。

其他类型的环境变量也可以类比系统环境变量，只不过系统环境变量由shell来保存，而其他类型的环境变量由程序自己内部存储，但两者效果是一样的。具体地，定义了以下几类环境变量：

1. 系统环境变量，由shell保存，sylar环境变量模块提供getEnv()/setEnv()方法用于操作系统环境变量。

2. 程序自定义环境变量，对应get()/add()/has()/del()接口，自定义环境变量保存在程序自己的内存空间中，在内部实现为一个`std::map<std::string, std::string>`结构。

3. 命令行参数，通过解析main函数的参数得到。所有参数都被解析成选项-选项值的形式，选项只能以-开头，后面跟选项值。如果一个参数只有选项没有值，那么值为空字符串。命令行参数也保存在程序自定义环境变量中。

4. 帮助选项与描述。这里是为了统一生成程序的命令行帮助信息，在执行程序时如果指定了-h选项，那么就打印这些帮助信息。帮助选项与描述也是存储在程序自己的内存空间中，在内部实现为一个`std::vector<std::pair<std::string, std::string>>`结构。

5. 与程序运行路径相关的信息，包括记录程序名，程序路径，当前路径，这些由单独的成员变量来存储。

# 1. 模块设计

与环境变量相关的类只有一个class Env，并且这个类被包装成了单例模式。通过单例可以保证程序的环境变量是全局唯一的，便于统一管理。Env类提供以下方法：

1. init: 环境变量模块初始化，需要将main函数的参数原样传入init接口中，以便于从main函数参数中提取命令行选项与值，以及通过argv[0]参数获取命令行程序名称。

2. add/get/has/del：用于操作程序自定义环境变量，参数为key-value，get操作支持传入默认值，在对应的环境变量不存在时，返回这个默认值。

3. setEnv/getEnv: 用于操作系统环境变量，对应标准库的setenv/getenv操作。

3. addHelp/removeHelp/printHelp: 用于操作帮助选项和描述信息。

4. getExe/getCwd/getAbsolutePath: 用于获取程序名称，程序路径，绝对路径。

5. getConfigPath: 获取配置文件夹路径，配置文件夹路径由命令行-c选项传入。

# 2. 模块实现

## 2.1 Env

> 环境变量类

1. 获取程序的bin文件绝对路径是通过/proc/$pid/目录下exe软链接文件指向的路径来确定的，用到了readlink(2)系统调用。
2. 通过bin文件绝对路径可以得到bin文件所在的目录，只需要将最后的文件名部分去掉即可。
3. 通过argv[0]获得命令行输入的程序路径，注意这里的路径可能是以./开头的相对路径。
4. 通过setenv/getenv操作系统环境变量，参考setenv(3), getenv(3)。
5. 提供getAbsolutePath方法，传入一个相对于bin文件的路径，返回这个路径的绝对路径。比如默认的配置文件路径就是通过getAbsolutePath(get("c", "conf"))来获取的，也就是配置文件夹默认在bin文件所在目录的conf文件夹。
6. 按使用惯例，main函数执行的第一条语句应该就是调用Env的init方法初始化命令行参数。

```C++
class Env {
public:
    typedef RWMutex RWMutexType;

    bool init(int argc, char **argv);
    void add(const std::string &key, const std::string &val);
    bool has(const std::string &key);
    void del(const std::string &key);
    std::string get(const std::string &key, const std::string &default_value = "");
    void addHelp(const std::string &key, const std::string &desc);
    void removeHelp(const std::string &key);
    void printHelp();
    const std::string &getExe() const { return m_exe; }
    const std::string &getCwd() const { return m_cwd; }
    bool setEnv(const std::string &key, const std::string &val);
    std::string getEnv(const std::string &key, const std::string &default_value = "");
    std::string getAbsolutePath(const std::string &path) const;
    std::string getAbsoluteWorkPath(const std::string& path) const;
    std::string getConfigPath();

private:
    /// Mutex
    RWMutexType m_mutex;
    /// 存储程序的自定义环境变量
    std::map<std::string, std::string> m_args;
    /// 存储帮助选项与描述
    std::vector<std::pair<std::string, std::string>> m_helps;

    /// 程序名，也就是argv[0]
    std::string m_program;
    /// 程序完整路径名，也就是/proc/$pid/exe软链接指定的路径 
    std::string m_exe;
    /// 当前路径，从argv[0]中获取
    std::string m_cwd;
};
```

## 2.2 守护进程

将进程与终端解绑，转到后台运行，除此外，还实现了双进程唤醒功能，父进程作为守护进程的同时会检测子进程是否退出，如果子进程退出，则会定时重新拉起子进程。

以下是守护进程的实现步骤：

调用daemon(1, 0)将当前进程以守护进程的形式运行；
守护进程fork子进程，在子进程运行主业务；
父进程通过waitpid()检测子进程是否退出，如果子进程退出，则重新拉起子进程；


```C++
static int real_daemon(int argc, char** argv,
                     std::function<int(int argc, char** argv)> main_cb) {
    daemon(1, 0);
    ProcessInfoMgr::GetInstance()->parent_id = getpid();
    ProcessInfoMgr::GetInstance()->parent_start_time = time(0);
    while(true) {
        pid_t pid = fork();
        if(pid == 0) {
            //子进程返回
            ProcessInfoMgr::GetInstance()->main_id = getpid();
            ProcessInfoMgr::GetInstance()->main_start_time  = time(0);
            JUJIMEIZUO_LOG_INFO(g_logger) << "process start pid=" << getpid();
            return real_start(argc, argv, main_cb);
        } else if(pid < 0) {
            JUJIMEIZUO_LOG_ERROR(g_logger) << "fork fail return=" << pid
                << " errno=" << errno << " errstr=" << strerror(errno);
            return -1;
        } else {
            //父进程返回
            int status = 0;
            waitpid(pid, &status, 0);
            if(status) {
                JUJIMEIZUO_LOG_ERROR(g_logger) << "child crash pid=" << pid
                    << " status=" << status;
            } else {
                JUJIMEIZUO_LOG_INFO(g_logger) << "child finished pid=" << pid;
                break;
            }
            ProcessInfoMgr::GetInstance()->restart_count += 1;
            sleep(g_daemon_restart_interval->getValue());
        }
    }
    return 0;
}
```

# 3. 总结

在解析命令行参数时，没有使用getopt()/getopt_long()接口，而是使用了自己编写的解析代码，这就导致sylar的命令行参数不支持长选项和选项合并，像ps -aux这样的多个选项组合在一起的命令行参数以及ps --help这样的长选项是不支持的。
# Http模块

> 提供Http服务

# 1. 模块设计

采用Ragel（有限状态机，性能媲美汇编），实现了HTTP/1.1的简单协议实现和uri的解析。基于SocketStream实现了HttpConnection(HTTP的客户端)和HttpSession（HTTP服务器端的链接）。基于TcpServer实现了HttpServer。提供了完整的HTTP的客户端API请求功能，HTTP基础API服务器功能。

主要包含以下几个模块：

- HTTP常量定义，包括HTTP方法HttpMethod与HTTP状态HttpStatus。
- HTTP请求与响应结构，对应HttpRequest和HttpResponse。
- HTTP解析器，包含HTTP请求解析器与HTTP响应解析器，对应HttpRequestParser和HttpResponseParser。
- HTTP会话结构，对应HttpSession。
- HTTP服务器。
- HTTP Servlet。
- HTTP客户端HttpConnection，用于发起GET/POST等请求，支持连接池。

# 2. 模块实现

## 2.1 Http常量定义

### 2.1.1 http_parser

```C++
/* Request Methods */
#define HTTP_METHOD_MAP(XX)         \
  XX(0,  DELETE,      DELETE)       \
  XX(1,  GET,         GET)          \
  XX(2,  HEAD,        HEAD)         \
  XX(3,  POST,        POST)         \
  XX(4,  PUT,         PUT)          \
...
 
/* Status Codes */
#define HTTP_STATUS_MAP(XX)                                                 \
  XX(100, CONTINUE,                        Continue)                        \
  XX(101, SWITCHING_PROTOCOLS,             Switching Protocols)             \
  XX(102, PROCESSING,                      Processing)                      \
  XX(200, OK,                              OK)                              \
  XX(201, CREATED,                         Created)                         \
  XX(202, ACCEPTED,                        Accepted)                        \
  XX(203, NON_AUTHORITATIVE_INFORMATION,   Non-Authoritative Information)   \
...
```

### 2.1.2 http

```C++
/**
 * @brief HTTP方法枚举
 */
enum class HttpMethod {
#define XX(num, name, string) name = num,
    HTTP_METHOD_MAP(XX)
#undef XX
    INVALID_METHOD
};
 
/**
 * @brief HTTP状态枚举
 */
enum class HttpStatus {
#define XX(code, name, desc) name = code,
    HTTP_STATUS_MAP(XX)
#undef XX
};
```

## 2.2 Http请求与响应结构

包括HttpRequest和HttpResponse两个结构，用于封装HTTP请求与响应。

对于HTTP请求，需要关注HTTP方法，请求路径和参数，HTTP版本，HTTP头部的key-value结构，Cookies，以及HTTP Body内容。

对于HTTP响应，需要关注HTTP版本，响应状态码，响应字符串，响应头部的key-value结构，以及响应的Body内容。

## 2.3 HttpParser

> Http解析器

输入字节流，解析HTTP消息，包括HttpRequestParser和HttpResponseParser两个结构。

HTTP解析器基于nodejs/http-parser实现，通过套接字读到HTTP消息后将消息内容传递给解析器，解析器通过回调的形式通知调用方HTTP解析的内容。


### 2.3.1 HTTP请求解析类
```C++
class HttpRequestParser {
public:
    /// HTTP解析类的智能指针
    typedef std::shared_ptr<HttpRequestParser> ptr;

...
private:
    /// http_parser
    http_parser m_parser;
    /// HttpRequest
    HttpRequest::ptr m_data;
    /// 错误码，参考http_errno
    int m_error;
    /// 是否解析结束
    bool m_finished;
    /// 当前的HTTP头部field，http-parser解析HTTP头部是field和value分两次返回
    std::string m_field;
};
```

### 2.3.2 Http响应解析结构体

```C++
class HttpResponseParser {
public:
    /// 智能指针类型
    typedef std::shared_ptr<HttpResponseParser> ptr;

...
private:
    /// HTTP响应解析器
    http_parser m_parser;
    /// HTTP响应对象
    HttpResponse::ptr m_data;
    /// 错误码
    int m_error;
    /// 是否解析结束
    bool m_finished;
    /// 当前的HTTP头部field
    std::string m_field;
};
```

## 2.4 HttpSession

> Http会话结构

继承自SocketStream，实现了在套接字流上读取HTTP请求与发送HTTP响应的功能，在读取HTTP请求时需要借助HTTP解析器，以便于将套接字流上的内容解析成HTTP请求。

```C++
class HttpSession : public SocketStream {
public:
    typedef std::shared_ptr<HttpSession> ptr;

    HttpSession(Socket::ptr sock, bool owner = true);
    HttpRequest::ptr recvRequest();
    int sendResponse(HttpResponse::ptr rsp);
};
```

## 2.5 HttpServer

> Http服务器

继承自TcpServer，重载handleClient方法，将accept后得到的客户端套接字封装成HttpSession结构，以便于接收和发送HTTP消息。

```C++
class HttpServer : public TcpServer {
public:
    typedef std::shared_ptr<HttpServer> ptr;

    HttpServer(bool keepalive = false
               ,jujimeizuo::IOManager* worker = jujimeizuo::IOManager::GetThis()
               ,jujimeizuo::IOManager* io_worker = jujimeizuo::IOManager::GetThis()
               ,jujimeizuo::IOManager* accept_worker = jujimeizuo::IOManager::GetThis());

    ServletDispatch::ptr getServletDispatch() const { return m_dispatch;}
    void setServletDispatch(ServletDispatch::ptr v) { m_dispatch = v;}

    virtual void setName(const std::string& v) override;
protected:
    virtual void handleClient(Socket::ptr client) override;
private:
    /// 是否支持长连接
    bool m_isKeepalive;
    /// Servlet分发器
    ServletDispatch::ptr m_dispatch;
};
```

## 2.6 Http Servlet

> Http请求

提供HTTP请求路径到处理类的映射，用于规范化的HTTP消息处理流程。

HTTP Servlet包括两部分，第一部分是Servlet对象，每个Servlet对象表示一种处理HTTP消息的方法，第二部分是ServletDispatch，它包含一个请求路径到Servlet对象的映射，用于指定一个请求路径该用哪个Servlet来处理。


### 2.6.1 Servlet

> Servlet类

```C++
class Servlet {
public:
    typedef std::shared_ptr<Servlet> ptr;

    Servlet(const std::string& name)
        :m_name(name) {}
    virtual ~Servlet() {}
    virtual int32_t handle(jujimeizuo::http::HttpRequest::ptr request
                   , jujimeizuo::http::HttpResponse::ptr response
                   , jujimeizuo::http::HttpSession::ptr session) = 0;
    const std::string& getName() const { return m_name;}
protected:
    std::string m_name;
};
```

### 2.6.2 ServletDispatch

> Servlet分发器

```C++
class ServletDispatch : public Servlet {
public:
    typedef std::shared_ptr<ServletDispatch> ptr;
    typedef RWMutex RWMutexType;

...
private:
    /// 读写互斥量
    RWMutexType m_mutex;
    /// 精准匹配servlet MAP
    /// uri(/jujimeizuo/xxx) -> servlet
    std::unordered_map<std::string, IServletCreator::ptr> m_datas;
    /// 模糊匹配servlet 数组
    /// uri(/jujimeizuo/*) -> servlet
    std::vector<std::pair<std::string, IServletCreator::ptr> > m_globs;
    /// 默认servlet，所有路径都没匹配到时使用
    Servlet::ptr m_default;
};
```

## 2.7 HttpConnection

> Http客户端

用于发起GET/POST等请求并获取响应，支持设置超时，keep-alive，支持连接池。

HTTP服务端的业务模型是接收请求→ 发送响应，而HTTP客户端的业务模型是发送请求→ 接收响应。

关于连接池，是指提前预备好一系列已接建立连接的socket，这样，在发起请求时，可以直接从中选择一个进行通信，而不用重复创建套接字→ 发起connect→ 发起请求 的流程。

连接池与发起请求时的keep-alive参数有关，如果使用连接池来发起GET/POST请求，在未设置keep-alive时，连接池并没有什么卵用。

```C++
class HttpConnection : public SocketStream {
friend class HttpConnectionPool;
public:
    typedef std::shared_ptr<HttpConnection> ptr;

...
private:
    /// 创建时间
    uint64_t m_createTime = 0;
    /// 该连接已使用的次数，只在使用连接池的情况下有用
    uint64_t m_request = 0;
};
```

# 3. 总结

Http的各个模块相互联系、不可分割，从请求和响应，到解析，包括会话，再到服务端和客户端。

HTTP模块依赖nodejs/http-parser提供的HTTP解析器，并且直接复用了nodejs/http-parser中定义的HTTP方法与状态枚举。

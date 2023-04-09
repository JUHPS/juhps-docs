# 网络地址模块

> 提供网络地址相关的类，支持与网络地址相关的操作。

![网络地址模块UML](../../img/AddressUML.jpeg)

## 1. 模块设计

所有网络地址的方法，包括网络地址查询及网卡地址查询功能等等。包括IPv4、IPv6、Unix、Unknown地址类，操作各成员的网络地址和端口，以及获取子码掩码等操作。

Linux使用Berkeley套接字接口进行网络编程，这套接口是事实上的标准网络套接字编程接口，在基本所有的系统上都支持。Berkeley套接字接口提供了一系列用于网络编程的通用API，通过这些API可以实现跨主机之间网络通信，或是在本机上通过Unix域套接字进行进程间通信。

所有的套接字API都是以指针形式接收sockaddr参数，并且额外需要一个地址长度参数，这可以保证当sockaddr本身不足以容纳一个具体的地址时，可以通过指针取到全部的内容。比如上面的地址内容占14字节，这并不足以容纳一个128位16字节的IPv6地址。但当以指针形式传入时，完全可以通过指针取到适合IPv6的长度。

除sockaddr外，套接字接口还定义了一系列具体的网络地址结构，比如sockaddr_in表示IPv4地址，sockaddr_in6表示IPv6地址，sockaddr_un表示Unix域套接字地址。

## 2. 模块实现

### 2.1 Address

网络地址类的基类，同时是一个抽象类，对应sockaddr，表示通用网络地址。Address类还提供了地址解析与本机网卡地址查询的功能，地址解析功能可以实现域名解析，网卡地址查询可以获取本机指定网卡的IP地址。

```C++
class Address {
public:
    typedef std::shared_ptr<Address> ptr;

    static Address::ptr Create(const sockaddr *addr, socklen_t addrlen);
    static bool Lookup(std::vector<Address::ptr> &result, const std::string &host,
                       int family = AF_INET, int type = 0, int protocol = 0);
    static Address::ptr LookupAny(const std::string &host,
                                  int family = AF_INET, int type = 0, int protocol = 0);
    static std::shared_ptr<IPAddress> LookupAnyIPAddress(const std::string &host,
                                                         int family = AF_INET, int type = 0, int protocol = 0);
    static bool GetInterfaceAddresses(std::multimap<std::string, std::pair<Address::ptr, uint32_t>> &result,
                                      int family = AF_INET);
    static bool GetInterfaceAddresses(std::vector<std::pair<Address::ptr, uint32_t>> &result, const std::string &iface, int family = AF_INET);
    virtual ~Address() {}
    int getFamily() const;
    virtual const sockaddr *getAddr() const = 0;
    virtual sockaddr *getAddr() = 0;
    virtual socklen_t getAddrLen() const = 0;
    virtual std::ostream &insert(std::ostream &os) const = 0;
    std::string toString() const;
    bool operator<(const Address &rhs) const;
    bool operator==(const Address &rhs) const;
    bool operator!=(const Address &rhs) const;
};
```

### 2.2 IPAdress

继承自Address类，表示一个IP地址，同样是一个抽象类，因为IP地址包含IPv4地址和IPv6地址。IPAddress类提供了IP地址相关的端口和掩码、网段地址、网络地址操作，无论是IPv4还是IPv6都支持这些操作，但这些方法都是抽象方法，需要由继承类来实现。

### 2.3 IPv4Address

继承自IPAddress类，表示一个IPv4地址，到这一步，IPv4Address就是一个实体类了，它包含一个sockaddr_in类型的成员，并且提供具体的端口设置/获取，掩码、网段、网络地址设置/获取操作。

### 2.4 IPv6Address

继承自IPAddress类，表示一个IPv6地址，也是一个实体类，实现思路和IPv4Address一致。

### 2.5 UnixAddress

继承自Address类，表示一个Unix域套接字地址，是一个实体类，可以用于实例化对象。UnixAddress类包含一个sockaddr_un对象以及一个路径字符串长度。

### 2.6 UnknownAddress

继承自Address类，包含一个sockaddr成员，表示未知的地址类型。

## 3. 总结

整个网络地址模块可以处理任何形式的地址，包括IPv4、IPv6、Unix和未知的Unknown地址，没有针对每种类型的地址都制定一套对应的API接口，而是拟定了一个通用的套接字地址结构sockaddr，用于表示任意类型的地址，所以的套接字API在传入地址参数时都只需要传入sockaddr类型，保证接口的通用性。还有一系列表示具体的网络地址的结构，这些具体的网络地址结构用于用户赋值，但在使用时，都要转化成sockaddr的形式。
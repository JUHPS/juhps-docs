# 字节流模块

> 封装流式的统一接口。将文件，socket封装成统一的接口。使用的时候，采用统一的风格操作。基于统一的风格，可以提供更灵活的扩展。

![字节流模块UML](../../img/StreamUML.jpeg)

# 1. 模块设计

所有的流结构都继承自抽象类Stream，Stream类规定了一个流必须具备read/write接口和readFixSize/writeFixSize接口，继承自Stream的类必须实现这些接口。

- read：读数据，接收内存或ByteArray
- write：写数据，接收内存或ByteArray
- readFixSize 写固定长度的数据，接收内存或ByteArray
- writeFixSize 读固定长度的数据，接收内存或ByteArray

# 2. 模块实现

## 2.1 Stream

> 流结构

```C++
class Stream {
public:
    typedef std::shared_ptr<Stream> ptr;

    virtual ~Stream() {}
    virtual int read(void* buffer, size_t length) = 0;
    virtual int read(ByteArray::ptr ba, size_t length) = 0;
    virtual int readFixSize(void* buffer, size_t length);
    virtual int readFixSize(ByteArray::ptr ba, size_t length);
    virtual int write(const void* buffer, size_t length) = 0;
    virtual int write(ByteArray::ptr ba, size_t length) = 0;
    virtual int writeFixSize(const void* buffer, size_t length);
    virtual int writeFixSize(ByteArray::ptr ba, size_t length);
    virtual void close() = 0;
};
```

## 2.2 SocketStream

> Socket流

```C++
class SocketStream : public Stream {
public:
    typedef std::shared_ptr<SocketStream> ptr;

   ...
protected:
    /// Socket类
    Socket::ptr m_socket;
    /// 是否主控
    bool m_owner;
};
```

# 3. 总结

SocketStream类将套接字封装成流结构，以支持Stream接口规范，除此外，SocketStream还支持套接字关闭操作以及获取本地/远端地址的操作。
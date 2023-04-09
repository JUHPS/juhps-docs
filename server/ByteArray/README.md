# ByteArray序列化模块

> ByteArray二进制序列化模块，提供对二进制数据的常用操作。读写入基础类型int8_t,int16_t,int32_t,int64_t等，支持Varint,std::string的读写支持,支持字节序转化,支持序列化到文件，以及从文件反序列化等功能。

# 1. 模块设计

ByteArray的底层存储是固定大小的块，以链表形式组织。每次写入数据时，将数据写入到链表最后一个块中，如果最后一个块不足以容纳数据，则分配一个新的块并添加到链表结尾，再写入数据。ByteArray会记录当前的操作位置，每次写入数据时，该操作位置按写入大小往后偏移，如果要读取数据，则必须调用setPosition重新设置当前的操作位置。

ByteArray支持基础类型的序列化与反序列化功能，并且支持将序列化的结果写入文件，以及从文件中读取内容进行反序列化。ByteArray支持以下类型的序列化与反序列化：

1. 固定长度的有符号/无符号8位、16位、32位、64位整数
2. 不固定长度的有符号/无符号32位、64位整数
3. float、double类型
4. 字符串，包含字符串长度，长度范围支持16位、32位、64位。
5. 字符串，不包含长度。
以上所有的类型都支持读写。

ByteArray还支持设置序列化时的大小端顺序。

# 2. 模块实现

### 2.1 ByteArray

> 二进制序列化类

二进制数组,提供基础类型的序列化,反序列化功能。

```C++
class ByteArray {
public:
    typedef std::shared_ptr<ByteArray> ptr;

    struct Node {
        Node(size_t s);
        Node();
        ~Node();
        char* ptr;
        Node* next;
        size_t size;
    };

...
private:
    size_t m_baseSize;
    size_t m_position;
    size_t m_capacity;
    size_t m_size;
    int8_t m_endian;
    Node* m_root;
    Node* m_cur;
};

```

### 2.2 endian

> 8/4/2字节类型的字节序转化

```C++
template <class T>
typename std::enable_if<sizeof(T) == sizeof(uintXX_t), T>::type
byteswap(T value) {
    return (T)bswap_XX((uintXX_t)value);
}
```

# 3. 总结

1. ByteArray在序列化不固定长度的有符号/无符号32位、64位整数时使用了zigzag算法。
2. ByteArray在序列化字符串时使用TLV中的Length和Value。
3. 可以自适应在大端机器和小端机器执行byteswap。
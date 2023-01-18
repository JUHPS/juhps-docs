# 配置模块

> 用于定义/声明配置项，并且从配置文件中加载用户配置，体现约定大于配置的思想，配置文件采用yml，并用yaml-cpp进行解析，用片特化的方式实现数据类型的序列化和反序列，并用回调函数的机制使其生效。

参考谷歌开源的基于命令行的C++配置库：gflags

![配置模块UML](../../img/ConfigUML.jpeg)

## 1. ConfigVarBase

> 配置项基类

每个配置项都包括`name`和`description`，以及`toString()`和`fromString()`两个纯虚函数，用于和YAML字符串进行相互转换。但并不包括配置项类型和值，由继承类实现。

不区分大小写，全部用`transform()`转成小写进行配置。

## 2. ConfigVar

> 配置参数类（继承ConfigVarBase）

### 2.1 定义成模版类

`template <class T, class FromStr = LexicalCast<std::string, T> , class ToStr = LexicalCast<T, std::string> >`

`T`为配置项的类型，`FromStr()`和`ToStr()`用于YAML字符串的转换，并根据不同的`T`实现不同的片特化。

### 2.2 支持变更配置

`setValue/getValue`方法用于获取/更新配置值（更新配置时会一并触发全部的配置变更回调函数）\
`addListener/delListener`方法用于添加或删除配置变更回调函数。

## 2. Config

> 配置管理类

负责托管ConfigVar对象。

所有成员都为`static`，保证全局只有一个实例。

### 2.1 Lookup

> 用于根据配置名称查询配置项。

如果调用Lookup查询时同时提供了默认值和配置项的描述信息，那么在未找到对应的配置时，会自动创建一个对应的配置项，这样就保证了配置模块定义即可用的特性。

### 2.2 LoadFromYaml

> 从YAML对象加载配置。

## 3. yaml-cpp

项目的配置文件采用yml，并用yaml-cpp库进行解析。

对于每种类型的配置，在对应的`ConfigVar`模版类实例化时都要提供`FromStr`和`ToStr`两个仿函数，用于实现该类型和YAML字符串的相互转换。

对于每种数据类型，包括自定义数据类型，都需要片特化。从一个基本类型的转换类开始，特化出其他类型的转换类。

```C++
template<class F, class T>
class LexicalCast {
public:
    /**
     * @brief 类型转换
     * @param[in] v 源类型值
     * @return 返回v转换后的目标类型
     * @exception 当类型不可转换时抛出异常
     */
    T operator()(const F& v) {
        return boost::lexical_cast<T>(v);
    }
};
```

实现了`int`、`vector`、`set`、`map`等类型，根据这些类型的搭配，还实现其他复杂类型，例如`vector<set>`、`set<vector>`等等。

## 4. Usage

```bash
YAML::Node root = YAML::LoadFile("url/*.yml");
jujimeizuo::Config::LoadFromYaml(root);
```
### 4.1 unordered_map做配置

```C++
jujimeizuo::ConfigVar<std::unordered_map<std::string, int> >::ptr g_str_int_unordered_map_value_config =
	jujimeizuo::Config::Lookup("system.str_int_unordered_map", std::unordered_map<std::string, int>{{"k", 2}}, "system str int unordered_map");
```

```bash
2022-09-01 00:03:24     4294967295      0       [INFO]  [root]  /Users/fengzetao/Desktop/WebServer/tests/test_config.cc:142     after str_int_unordered_map: {k2 - 20}
```

## 5. 总结

采用约定由于配置的思想。定义即可使用。不需要单独去解析。支持变更通知功能。使用YAML文件做为配置内容。支持级别格式的数据类型，支持STL容器(vector,list,set,map等等),支持自定义类型的支持（需要实现序列化和反序列化方法)。

通过配置系统对日志进行配置需要对日志里的类型进行片特化处理，这样在处理序列化与反序列化的时候才能识别yml文件中的log配置。具体在`LogDefine`和`LogAppenderDefine`。
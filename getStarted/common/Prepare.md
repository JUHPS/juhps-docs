# 操作系统

Centos 或 Ubuntu
下文以Ubantu举例

# 编译工具

```bash
apt install cmake gcc g++
```

# 依赖库

## boost库

```bash
apt install libboost-dev
```

## yaml-cpp库

```bash
git clone git@github.com:jbeder/yaml-cpp.git
cd yaml-cpp
mkdir build
cd build
cmake ..
make & make install
```

## openssl库

```bash
apt-get install libssl-dev
```

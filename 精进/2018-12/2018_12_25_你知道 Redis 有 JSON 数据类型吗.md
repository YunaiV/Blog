title: 你知道 Redis 有 JSON 数据类型吗
date: 2018-12-25
tags:
categories: 精进
permalink: Fight/Did-you-know-that-Redis-has-JSON-data-types
author: dys
from_url: https://cloud.tencent.com/developer/article/1084698
wechat_url: https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247485871&idx=2&sn=f41c537c4e73534e7367936fe3cfbb72&chksm=fa49761ecd3eff08eadea07ebcaa8127801d30a877a246b09c8ac54ef5b990e46772b8426833&token=396846451&lang=zh_CN#rd

-------

摘要: 原创出处 https://cloud.tencent.com/developer/article/1084698 「dys」欢迎转载，保留摘要，谢谢！

- [1. 简介](http://www.iocoder.cn/Fight/Did-you-know-that-Redis-has-JSON-data-types/)
- [2. 示例](http://www.iocoder.cn/Fight/Did-you-know-that-Redis-has-JSON-data-types/)
  - [2.1 基础操作](http://www.iocoder.cn/Fight/Did-you-know-that-Redis-has-JSON-data-types/)
  - [2.2 json 内部操作](http://www.iocoder.cn/Fight/Did-you-know-that-Redis-has-JSON-data-types/)
- [3. 安装](http://www.iocoder.cn/Fight/Did-you-know-that-Redis-has-JSON-data-types/)
  - [3.1 安装流程](http://www.iocoder.cn/Fight/Did-you-know-that-Redis-has-JSON-data-types/)
  - [3.2 详细安装过程](http://www.iocoder.cn/Fight/Did-you-know-that-Redis-has-JSON-data-types/)
- [4. 小结](http://www.iocoder.cn/Fight/Did-you-know-that-Redis-has-JSON-data-types/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

# 1. 简介

Redis 本身有比较丰富的数据类型，例如 String、Hash、Set、List

JSON 是我们常用的数据类型，当我们需要在 Redis 中保存 json 数据时是怎么存放的呢？

一般是用 String 或者 Hash，但还是不太方便，无法灵活的操作 json 数据

在 Redis 4.0 中，有一个重大改进：**modules 模块系统**，可以让我们开发新的功能，集成到 redis 中

`rejson` 就是一个新的模块，为 redis 提供了 json 存储能力

# 2. 示例

## 2.1 基础操作

```javascript
127.0.0.1:6379> JSON.SET object . '{"foo": "bar", "ans": 42}'
OK
127.0.0.1:6379> JSON.GET object
"{\"foo\":\"bar",\"ans\":42}"
```

先看下第一条命令的含义：

- `JSON.SET` 是json设置命令
- `object` 是 key
- `.` 是json文档的root，后面的一串是具体的 json 数据值

第二条命令是获取 key 为 `object` 的json数据

## 2.2 json 内部操作

- 获取某字段的值

```javascript
127.0.0.1:6379> JSON.GET object .ans
"42"
```

命令中的 `.ans` 是目标路径，表示 `root` 下面的 `ans`

- 设置某字段值

```javascript
127.0.0.1:6379> json.set object .name '"bill"'
OK
127.0.0.1:6379> json.get object
"{\"foo\":\"bar\",\"ans\":42,\"hi\":\"hello\",\"name\":\"bill\"}"
```

这个命令是在 root 下新增了一个字段 `name`，值为 bill

也可以修改已有字段的值，用法相同

- 删除字段

```javascript
127.0.0.1:6379> json.del object .name
(integer) 1
127.0.0.1:6379> json.get object
"{\"foo\":\"bar\",\"ans\":42,\"hi\":\"hello\"}"
```

这个命令使用 `del` 把 root 下的 `name` 字段删除了

- 数字操作

`ans` 字段是数字类型，值为 42，下面对其执行 +3 操作

```javascript
127.0.0.1:6379> json.numincrby object .ans 3
"45"
127.0.0.1:6379> json.get object
"{\"foo\":\"bar\",\"ans\":45,\"hi\":\"hello\"}"
```

还可以进行乘法操作

```javascript
127.0.0.1:6379> json.nummultby object .ans 2
"90"
127.0.0.1:6379> json.get object
"{\"foo\":\"bar\",\"ans\":90,\"hi\":\"hello\"}"
```

还有很多其他操作命令，具体可以查看项目文档

# 3. 安装

因为使用了模块功能，所以需要 redis 4.0 以上版本

## 3.1 安装流程

1. 安装 redis 4.0
2. 安装相关系统依赖
3. 安装 rejson 模块
4. redis 加载 rejson 模块

## 3.2 详细安装过程

安装 redis 4.0

```javascript
wget https://github.com/antirez/redis/archive/4.0-rc2.tar.gz
tar xzf 4.0-rc2.tar.gz
cd redis-4.0-rc2/
make
```

安装依赖

```javascript
yum groupinstall "Development Tools"
```

（这是 centos 中的安装方法，ubuntu 可以使用这个命令 apt-get install build-essential ）

安装cmake

```javascript
# wget https://cmake.org/files/v3.8/cmake-3.8.0-rc3.tar.gz
# tar -xzvf cmake-2.8.11.2.tar.gz
# cd cmake-2.8.11.2
# ./bootstrap
# make
# make install
```

安装 rejson 模块

```javascript
git clone https://github.com/RedisLabsModules/rejson.git
cd rejson
./bootstrap.sh
cmake --build build --target rejson
```

安装完成后，rejson 目录中的 `lib` 下便会生成 `rejson.so`

启动 redis 时加载 rejson.so

```javascript
redis-server --loadmodule /path/to/module/rejson.so
```

在启动信息中会看到 rejson 的相关信息

```javascript
...
<ReJSON> JSON data type for Redis
...
```

安装完成，可以登录 redis 执行 json 命令了

# 4. 小结

rejson 让我们可以在 redis 中存储和操作 json 数据，非常方便

而且通过体验 rejson 模块，还可以感受到 redis 模块系统的强大，以后将会出现各种基于redis的强大功能

rejson 项目地址：

```javascript
https://redislabsmodules.github.io/rejson/
```
# 1. 概述

本文主要解析 `Namesrv`、`Broker` 如何实现高可用，`Producer`、`Consumer` 怎么与它们通信保证高可用。

# 2. `Namesrv` 高可用

**通过启动多个 `Namesrv` 实现高可用。**

相较于 `Zookeeper`、`Consul`、`Etcd` 等，`Namesrv` 是一个**轻量级**的注册中心。

## 2.1 `Broker` 注册到 `Namesrv`

* ✏️ **多个 `Namesrv` 之间，无任何通信与数据同步。通过 `Broker` 循环注册多个 `Namesrv`。**



# 3. `Broker` 高可用


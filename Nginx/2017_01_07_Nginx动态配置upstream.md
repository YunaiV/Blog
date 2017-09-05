title: Nginx 动态配置 upstream
date: 2017-01-07
tags:
categories: Nginx
permalink: Nginx/nginx-dynamic-upstream

-------

![](http://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

随着SOA、微服务越来越流行，注册发现服务已经成为架构里的标配。无论是在选择Dubbo、Dubbox、Spring Cloud都提供了对应的方案，我们不需要每次新增一个节点，就去修改对应配置。那么在使用Nginx的时候我们该怎么做呢？
参考类似服务发现的方案，我们选择了微博开源的nginx插件：https://github.com/weibocom/nginx-upsync-module 。

## 基于tengine安装 ##
* tengine：2.2.0（基于nginx1.8.1+）
* nginx-upsync-module：v1.0.0（基于nginx1.8.1+）
* 注册发现服务：etcd 3.0.15

问题：

* nginx的error.log不断报错`recv() failed (104: Connection reset by peer)` 。
    
    原因：该问题使用etcd会存在。作者在commit#89bf40fb60268062aaeac3780ad8d56f0834c400已经修复。由于该修复的提交是基于nginx 1.9.1+，我们无法直接在tegine2.2.0上使用。等待作者回复如何解决中。
    补充说明：即使报该错误，实际不影响插件的功能。
    
    
## 基于nginx安装 ##

*由于基于tegine2.2.0会报错，因此我们尝试使用nginx官方版本安装*

* nginx：1.10.1
* nginx-upsync-module：commit#6e1a9fe9117361539837efc81ac45303d7494dbe（基于nginx1.9.1+）
* 注册发现服务：etcd 3.0.15   

问题：

* 如果我们需要使用`nginx_upstream_check_module` 插件，需要使用https://github.com/yaoweibin/nginx_upstream_check_module 下的，不然`make`时会报错。另外在`patch -p1 < check.patch`时，请cd nginx的目录下，输入`patch -p1 < /path/to/nginx_http_upstream_check_module/check.patch`。这点README.md也说了，当时太着急，没去看，卡了半小时。

    >  
        $ wget 'http://nginx.org/download/nginx-1.0.14.tar.gz'
        $ tar -xzvf nginx-1.0.14.tar.gz
        $ cd nginx-1.0.14/
        $ patch -p1 < /path/to/nginx_http_upstream_check_module/check.patch
        $ ./configure --add-module=/path/to/nginx_http_upstream_check_module
        $ make
        $ make install

## 基于docker安装 ##

TODO

## 原理 ##

1. 实时性：upstream更新依赖注册中心提供的查询的http接口。这点和Dubbo、Spring Cloud的注册发现的实时性有些不同。如果对实时性要求特别高，可以调整`upsync_interval`参数。
2. Zookeeper的支持：目前官方暂时未提供相应的支持。考虑到很多团队是使用Zookeeper作为服务发现，另外维护一个etcd、consul集群是有相应学习成本和运维成本的。那么怎么办？我们可以封装Zookeeper提供http接口给`nginx-upsync-module`使用，当然接口形式得满足consual或者etcd提供的http接口。

## 参考文章 ##

1. nginx动态配置及服务发现那些事：http://xiaorui.cc/2016/10/16/nginx动态配置及服务发现那些事/


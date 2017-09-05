title: Docker 网络 —— Flannel 配置
date: 2017-02-04
tags:
categories: Docker
permalink: Docker/docker-network-flannel

-------

![](http://www.yunai.me/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

todo flannel简介

## A. 安装Etcd

😉网络上一大抄，我相信你能找到。

##B. 安装Flannel

基于CentOS 7.2

### 1. 下载Flannel


```
$ mkdir /root/work/flannel -p
$ cd /root/work/flannel
$ wget https://github.com/coreos/flannel/releases/download/v0.7.0/flannel-v0.7.0-linux-amd64.tar.gz
$ tar -zxvf flannel-v0.7.0-linux-amd64.tar.gz
$ ls -ls
```
    
结果：![](http://www.yunai.me/images/Docker/2017_02_04/00AD067C-53F7-43EF-A8CA-F77CD72471BA.png)    
    
### 2. 启动Flannel


```
$ nohup ./flanneld -etcd-endpoints=http://10.29.76.96:2379 &  
# 说明：http://10.29.76.96:2379 为etcd请求地址，需要改成你自己的噢
$ cat /run/flannel/subnet.env
```

结果：![](http://www.yunai.me/images/Docker/2017_02_04/D7087C8C-E6D7-408E-A088-3517E454A592.png)    

ps：重要！重要！重要！目前该方式仅仅用于测试，如果正式使用，请将Flannel配置到Systemd。

### 3. 配置Docker

* Docker版本：Docker version 1.12.5, build 047e51b/1.12.5
* Docker安装方式：yum

**该步骤可能Docker安装方式不同，配置方式不同。目的是修改Docker的bip、mtu。**

```
$ vi /etc/sysconfig/docker-network
# 说明：DOCKER_NETWORK_OPTIONS=' --bip=10.1.77.1/24 --mtu=1472 '
# 说明：bip为subnet.env里的FLANNEL_SUBNET，mtu为subnet.env里的FLANNEL_MTU
$ systemctl daemon-reload
$ systemctl restart docker
$ ifconfig docker0
$ ifconfig flannel0
```

结果：![](http://www.yunai.me/images/Docker/2017_02_04/3B5B0FEC-E65D-4D02-87FD-70C443952845.png)

### 4. 验证网络是否通畅 

* 节点1：ip=10.1.97.1/24
* 节点2：ip=10.1.77.1/24

节点1：

```
$ docker run -it --rm ubuntu
# 假设ip为10.1.97.2
```

节点2:

```
$ docker run -it --rm ubuntu
# 假设ip为10.1.77.2
$ apt-get update
$ apt-get install iputils-ping
$ ping 10.1.97.2
# ping通即配置成功！
```

## C. 原理

1. [浅析flannel与docker结合的机制和原理](https://xuxinkun.github.io/2016/07/18/flannel-docker/)
2. [DockOne技术分享（十八）：一篇文章带你了解Flannel](http://dockone.io/article/618)

## D. 性能

* ![](http://www.yunai.me/images/Docker/2017_02_04/F7429465-2334-48C6-B84E-4C7FC735F5F9.png)
（来自文章[干货|你想要的百分点大规模Kubernetes集群的应用实践来了](http://mp.weixin.qq.com/s?__biz=MjM5MzI5NjY2MA==&mid=2653782073&idx=1&sn=6db70559acabae67e35e13af7883e1d5&chksm=bd4018428a37915415ffda36c4f9f5e31088063ef3ad83e325d3e4ecd4eccf8d202709ac9629&mpshare=1&scene=1&srcid=0203g7cy4y9XpVhqA9fr5PGp#rd)）

* ![](http://www.yunai.me/images/Docker/2017_02_04/0BF283C9-C26C-46C1-9BDA-604EAD67B2E2.png)
（来自文章[Weave is kinda slow](http://www.generictestdomain.net/docker/weave/networking/stupidity/2015/04/05/weave-is-kinda-slow/）

* [Docker或Kubernets的网络模型](http://www.do1618.com/archives/869)

* [Kubernets,Flannel,Docker网络性能深度测试](http://pangxiekr.com/kubernetsflannel-wang-luo-xing-neng-ce-shi-ji-diao-you/)

总结的话：host-gw > vxlan > udp。
kubernetes在阿里云已经有host-gw + vpc的支持。

## E. 碰到的问题
1. Flannel刚安装好，配置完Docker后，启动的Container无法被ping通，而Docker0的IP可以被ping通。
    TODO: 已解决，但是原因未知。目前猜测Kubernetes可能对Flannel第一次初始化有些影响。
    * 新开一台服务器，重新安装并进行配置(yum安装docker1.12版本），未重现该情况。出问题的机子都安装过Kubernetes。
    * 在出问题的机子上使用`tcpdump -i flannel0`，命令行输出了`01:57:55.680586 IP 10.1.77.0 > 10.1.11.2: ICMP echo request, id 12091, seq 514, length 64`。可以得出网络本身应该是通的，但是未进行响应。进入Container，`tcpdump -i eth0`，未输出任何东西。得出结论：`flannel0`未将ping转到Container。
    * 关闭有问题机子的Kubernetes相关的服务。重启启动Container，结果可以ping通了。此时重启服务器，结果这个时候，即使Kubernetes启动着，依然可以ping通。好怪T T。
    


## F. 参考文章列表

1. [Dokcer 使用 Flannel 跨主机通讯](https://mritd.me/2016/09/03/Dokcer-%E4%BD%BF%E7%94%A8-Flannel-%E8%B7%A8%E4%B8%BB%E6%9C%BA%E9%80%9A%E8%AE%AF/)
2. [kubernetes入门1：kubernetes+flannel+etcd环境搭建(通用安装)](http://zdevops.blog.51cto.com/2579684/1735492)
3. [用 Flannel 配置 Kubernetes 网络](http://dockone.io/article/1186)



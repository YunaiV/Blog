title: 为 Spring Cloud Config 插上管理的翅膀
date: 2018-01-01
tag:
categories: Spring-Cloud-Config
permalink: Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release/
author: 程序猿DD
from_url: http://blog.didispace.com/spring-cloud-config-admin-1-0-0-release/
wechat_url:

-------

摘要: 原创出处 http://blog.didispace.com/spring-cloud-config-admin-1-0-0-release/ 「程序猿DD」欢迎转载，保留摘要，谢谢！

- [简介](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
- [架构概览](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
- [部署方式](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
  - [全分离模式](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
  - [半分离模式](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
  - [All-In-One模式](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
- [配置详解](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
  - [Spring Cloud配置中心的构建与配置](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
- [config server with jdbc](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
- [Datasource, share with scca-rest-server](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
  - [SCCA REST服务端的构建与配置](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
- [if config server use git, need config these properties](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
- [Datasource](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
- [Datasource](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
  - [SCCA UI服务端的构建与配置](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
- [管理功能](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
  - [系统配置](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
  - [配置中心](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
- [客户端接入](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
  - [客户端加载](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)
  - [其他参考](http://www.iocoder.cn/Spring-Cloud-Config/didi/spring-cloud-config-admin-1-0-0-release//)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------


> 最近一致在更新Spring Cloud Config的相关内容，主要也是为这篇埋个伏笔，相信不少调研过Spring Cloud Config的用户都会吐槽它的管理能力太弱。因此，就有了下面为讲推荐的这个开源项目，希望对已经入坑Spring Cloud Config的童鞋们有所帮助！

# 简介

在Spring Cloud的微服务架构方案中虽然提供了Spring Cloud Config来担任配置中心的角色，但是该项目的功能在配置的管理层面还是非常欠缺的。初期我们可以依赖选取的配置存储系统（比如：Gitlab、Github）给我们提供的配置管理界面来操作所有的配置信息，但是这样的管理还是非常粗粒度的，因此这个项目的目的就是解决这个问题，通过此项目，我们将提供一套基于Spring Cloud Config配置中心的可视化管理系统。

在该项目中，我们对于服务治理、配置存储、可视化操作都做了抽象，只要目的就是为了尽可能的兼容所有Spring Cloud Config的用户。任何Spring Cloud Config仅需要通过一些简单的配置，或者迁移工具就能将原来正在使用的配置中心统一的管理起来。

**项目地址**

- Github: https://github.com/dyc87112/spring-cloud-config-admin
- Gitee：https://gitee.com/didispace/spring-cloud-config-admin

- 前端Github: https://github.com/stone-jin/spring-cloud-config-admin-web
- 前端Gitee: https://gitee.com/stone-jin/spring-cloud-config-admin-web

> 如果您觉得该项目对您有用，欢迎Star、Follow支持我们！


# 架构概览

本项目采用了前后端分离的架构，通过core模块抽象了前端需要的操作，再通过persistence和discovery模块隔离不同的配置仓库和不同的服务注册中心，从而达到前端程序不需要关心到底使用了什么存储配置以及使用了什么注册中心，这样用户可以根据自己的需要自由的组合不同的配置存储和服务治理机制，尽可能的匹配大部分Spring Cloud用户的需求。

![](https://github.com/dyc87112/spring-cloud-config-admin/raw/master/statics/images/scca-arch.png)

# 部署方式

由于SCCA的架构对各个功能模块做了比较细致的拆分，所以它存在多种不同模式的部署方式，所以它既可以为已经在使用Spring Cloud Config提供服务，也可以为从零开始使用Spring Cloud Config的用户。

在SCCA中我们的可部署内容自底向上分为三个部分：

- `Spring Cloud 配置中心`：基于Spring Cloud Config构建的配置中心服务端。
- `SCCA REST 服务端`：SCCA的核心模块，实现了SCCA配置管理的持久化内容以及所有的管理操作API。
- `SCCA UI 服务端`：SCCA的前端模块，实现了可视化的配置管理操作界面。

下面我们来看看SCCA支持哪些多样的部署方式。

## 全分离模式

全分离模式就是将上述三个部分都以独立的进程进行部署，每一个部分都可以做高可用，具体部署结构可以如下图所示：

![](https://github.com/dyc87112/spring-cloud-config-admin/blob/master/statics/images/scca-deploy-1.png)

这种模式既可以适用于已经在使用Spring Cloud Config的用户，也适用于正准备开始适用的用户。其中，位于最底层的`Spring Cloud配置中心`就是一个最原始的Spring Cloud Config Server。所以，对于已经在使用Spring Cloud Config的用户只需要再部署一套`SCCA REST 服务端`和`SCCA UI 服务端`，并做一些配置就可以使用SCCA来管理所有的配置信息了。

**案例**

- [SCCA UI 服务端](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-ui-server)
- [SCCA REST 服务端（对接DB存储的配置中心）](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-rest-server-access-config-server-db)
- [SCCA REST 服务端（对接Git存储的配置中心）](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-rest-server-access-config-server-git)
- [Spring Cloud 配置中心（DB存储）](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-config-server-db)
- [Spring Cloud 配置中心（Git存储）](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-config-server-git)

## 半分离模式

所谓的半分离模式就是将上述的三个模块中的两个进行组合部署，以降低复杂度的部署方式。
**SCCA UI模块与SCCA REST模块合并**

如下图所示，我们可以将`SCCA UI服务端`与`SCCA REST服务端`组合在一个程序中来部署，这样可以有效的降低全分离模式的部署复杂度，同时对于已经在使用Spring Cloud Config的用户来说非常友好，已经部署的配置中心可以继续沿用。

![](https://github.com/dyc87112/spring-cloud-config-admin/blob/master/statics/images/scca-deploy-2.png)

**案例**

- [SCCA UI与SCCA REST合并的服务端](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-ui-server-with-rest-server)
- [Spring Cloud 配置中心（DB存储）](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-config-server-db)
- [Spring Cloud 配置中心（Git存储）](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-config-server-git)

> 注意：对接不同存储配置中心的配置参考分离部署中两个SCCA REST服务端的不同配置内容进行调整。

## All-In-One模式

最后介绍一种比较暴力的使用模式，SCCA支持将所有三个模块整合在一起使用和部署，在一个Spring Boot应用中同时包含：`Spring Cloud 配置中心`、`SCCA REST 服务端`以及`SCCA UI 服务端`，具体如下所示：

![](https://github.com/dyc87112/spring-cloud-config-admin/blob/master/statics/images/scca-deploy-4.png)

**案例**

- [All-In-One服务端](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-ui-server-with-all)


# 配置详解

本章节分别对三个核心模块的构建方式以及核心的配置内容。下面所有的构建都是基于Spring Boot构建的，所以您需要对Spring Boot项目的构建有基本的认识，这里不做介绍。

## Spring Cloud配置中心的构建与配置

在SCCA的架构中，配置中心的核心完全采用Spring Cloud Config，所以如何构建一个配置中心完全遵循Spring Cloud Config的使用方法。由于目前SCCA的REST模块主要实现了对Git存储和DB存储的综合管理，所以对于Spring Cloud Config的使用也只能支持这两种模式。下面分别介绍两种配置中心的搭建与配置。

### Git存储模式

这里主要介绍几种主要的并且SCCA能够比较好支持的配置模式：

**第一种：多个项目使用多个不同Git仓库存储的模式**

```properties
spring.cloud.config.server.git.uri=https://github.com/dyc87112/{application}.git
spring.cloud.config.server.git.username=
spring.cloud.config.server.git.password=
```

这种模式下不同的项目会对应的不同的Git仓库，如果项目中`spring.application.name=user-service`，那么它的配置仓库会定位到`https://github.com/dyc87112/user-service.git`仓库下的配置。配置文件按`application-{profile}.properties`的格式存储，`{profile}`代表环境名。

**第二种：多个项目公用一个Git仓库不同目录的存储模式**

```properties
spring.cloud.config.server.git.uri=https://github.com/dyc87112/config-repo.git
spring.cloud.config.server.git.search-paths=/{application}
spring.cloud.config.server.git.username=
spring.cloud.config.server.git.password=
```

这种模式下不同的项目会对应到`https://github.com/dyc87112/config-repo.git`仓库下的不同目录，如果项目中`spring.application.name=user-service`，那么它的配置仓库会定位到`https://github.com/dyc87112/config-repo.git`仓库下的`/user-service`目录。配置文件按`application-{profile}.properties`的格式存储，`{profile}`代表环境名。

> 案例：[Spring Cloud 配置中心（Git存储）](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-config-server-git)

### Db存储模式

在使用Db存储模式的时候，必须使用Spring Cloud的Edgware版本以上。比如，可以采用下面的配置：

```properties
# config server with jdbc
spring.profiles.active=jdbc
spring.cloud.config.server.jdbc.sql=SELECT `p_key`, `p_value` FROM property a, project b, env c, label d where a.project_id=b.id and a.env_id=c.id and a.label_id=d.id and b.name=? and c.name=? and d.name=?

# Datasource, share with scca-rest-server
spring.datasource.url=jdbc:mysql://localhost:3306/config-db
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

主要分为两个部分：

- 激活采用DB存储的模式：将`spring.profiles.active`设置为jdbc，同时指定获取配置的SQL，用户直接复制采用一样的配置即可。
- 指定存储配置的DB连接信息，除了mysql之外也可以使用其他主流关系型数据库。

**这里需要注意的，使用的DB要与后续介绍的SCCA REST模块采用同一个DB**

> 案例：[Spring Cloud 配置中心（DB存储）](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-config-server-db)

## SCCA REST服务端的构建与配置

在构建SCCA REST服务端的时候针对对接不同的配置存储有一些不同的配置要求，所以下面按目前支持的存储模式做不同的介绍。

### Git存储模式

当对接的配置中心采用Git存储的时候，需要引入以下核心依赖：

```xml
<dependency>
    <groupId>com.didispace</groupId>
    <artifactId>scca-rest</artifactId>
    <version>1.0.0-RELEASE</version>
</dependency>

<!-- scca persistence dependency -->
<dependency>
    <groupId>com.didispace</groupId>
    <artifactId>scca-persistence-git</artifactId>
    <version>1.0.0-RELEASE</version>
</dependency>
```

需要按如下配置：

```properties
# if config server use git, need config these properties
scca.git.username=
scca.git.password=
scca.git.repo-uri=https://github.com/dyc87112/{application}.git
scca.git.base-path=
scca.git.file-pattern=application-{profile}.properties

# Datasource
spring.datasource.url=jdbc:mysql://localhost:3306/config-db
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

主要两部分：

- 对接的git存储的配置：
  - `scca.git.username`：访问git的用户名
  - `scca.git.password`：访问git的密码
  - `scca.git.repo-uri`：配置git仓库的地址，与配置中心的`spring.cloud.config.server.git.uri`配置一致
  - `scca.git.base-path`：配置文件存储的相对路径，与配置中心的`spring.cloud.config.server.git.search-paths`配置一致
  - `scca.git.file-pattern`：配置文件的命名规则
- SCCA内部逻辑的存储库数据源信息

> 案例：[SCCA REST 服务端（对接Git存储的配置中心）](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-rest-server-access-config-server-git)

### Db存储模式

当对接的配置中心采用Git存储的时候，需要引入以下核心依赖：

```xml
<dependency>
    <groupId>com.didispace</groupId>
    <artifactId>scca-rest</artifactId>
    <version>1.0.0-RELEASE</version>
</dependency>

<!-- scca persistence dependency -->
<dependency>
    <groupId>com.didispace</groupId>
    <artifactId>scca-persistence-db</artifactId>
    <version>1.0.0-RELEASE</version>
</dependency>
```

需要按如下配置：

```properties
# Datasource
spring.datasource.url=jdbc:mysql://localhost:3306/config-db
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

**需要注意，当配置中心采用DB存储的时候，这里的数据源需要一致**

> 案例：[SCCA REST 服务端（对接DB存储的配置中心）](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-rest-server-access-config-server-db)

### 服务发现支持

如果SCCA REST模块在访问配置中心的时候基于服务发现的话还需要引入对应的支持依赖和配置

#### 与Eureka的整合

如果使用eureak，那么需要引入如下依赖：

```xml
<!-- scca discovery dependency-->
<dependency>
    <groupId>com.didispace</groupId>
    <artifactId>scca-discovery-eureka</artifactId>
    <version>1.0.0-RELEASE</version>
</dependency>
```

并且在配置中加入eureka的配置，比如：

```properties
eureka.client.serviceUrl.defaultZone=http://eureka.didispace.com/eureka/
```

**更多相关配置请参与Spring Cloud Netflix Eureka的配置文档。**

#### 与Consul的整合

如果使用consul，那么需要引入如下依赖：

```xml
<!-- scca discovery dependency-->
<dependency>
    <groupId>com.didispace</groupId>
    <artifactId>scca-discovery-consul</artifactId>
    <version>1.0.0-RELEASE</version>
</dependency>
```

并且在配置中加入consul的相关配置，比如：

```properties
spring.cloud.consul.host=localhost
spring.cloud.consul.port=8500
```

**更多相关配置请参与Spring Cloud Consul的配置文档。**

### 公共配置

SCCA REST模块还有一个特别的配置`scca.rest.context-path=/xhr`，该配置主要用来配置所有SCCA REST模块接口的前缀，该接口主要用于与SCCA UI模块对接时候使用，两边必须对接一致才能顺利对接。

## SCCA UI服务端的构建与配置

SCCA UI服务端需要引入以下核心依赖：

```xml
<dependency>
    <groupId>com.didispace</groupId>
    <artifactId>scca-ui</artifactId>
    <version>1.0.0-RELEASE</version>
</dependency>
```

另外，还需要在配置中指定具体要访问的SCCA REST模块的位置，主要有两种模式：

- 指定地址的配置：

```properties
scca.ui.rest-server-url=http://localhost:10130
```

- 基于服务发现的配置：

```properties
scca.ui.rest-server-name=scca-rest-server
```

**除了上面的配置之后，还需要引入eureka或consul的依赖以及做对应的配置**

最后，还有一个`scca.ui.rest-server-context-path=/xhr`配置，用来描述要访问的SCCA REST模块接口的前缀，与SCCA REST服务端的`scca.rest.context-path=/xhr`配置相对应。

> 案例：[SCCA UI 服务端](https://github.com/dyc87112/spring-cloud-config-admin/tree/master/scca-use-cases/scca-ui-server)


# 管理功能

通过之前介绍的任何一个部署方式搭建了配置中心和管理端之后，我们就可以打开浏览器访问我们的UI模块实现对配置中心的管理了。

访问地址为：`http://localhost:10032/admin/`，ip与端口根据实际部署UI模块的情况进行调整。

## 系统配置

在管理各个项目的配置之前，我们需要先做一些基础配置，比如：环境的配置、环境所属的参数配置，加密相关的配置等。

### 环境配置

环境配置主要用来维护要使用SCCA统一管理的环境以及对应的Spring Cloud Config服务端信息。

![环境配置](https://github.com/dyc87112/spring-cloud-config-admin/raw/master/statics/images/user-guilder-1.png)

如上图所示，通过“新增环境”按钮可以添加一个部署环境。当我们使用了Eureka、Consul等注册中心时，只需要配置注册中心的访问地址和配置中心的服务名以及配置中心访问的前缀，后续就可以方便的使用这个环境的配置中心来进行加密解密、拉取配置等一系列的操作了。

> 如果不采用服务发现的机制取找到配置中心，也可以将注册中心地址留空，配置中心服务名一栏直接配置访问注册中心的URL即可。

### 环境参数配置

环境参数配置主要用来配置每个环境所属的一些特有配置信息，比如：redis的地址，eureka的地址等等。这些配置信息将用户后续为各项目在各个环境配置的时候给予参考和快捷的替换操作提供元数据。

![环境参数配置](https://github.com/dyc87112/spring-cloud-config-admin/raw/master/statics/images/user-guilder-2.png)

### 加密管理

加密管理主要用来维护一些通常需要加密的Key，这样可以在后续编辑配置内容的时候，方便的进行批量加密操作。

![加密管理](https://github.com/dyc87112/spring-cloud-config-admin/raw/master/statics/images/user-guilder-3.png)

## 配置中心

在完成了上面的系统配置之后，用户就可以进入配置中心模块，这里会提供具体的管理配置内容的功能。目前主要有两部分组成：项目管理和配置管理。

### 项目管理

项目管理主要用来维护需要在各个环境部署的应用的配置信息，这里可以维护这个项目需要部署在什么环境，有多少配置的版本。

![项目管理](https://github.com/dyc87112/spring-cloud-config-admin/raw/master/statics/images/user-guilder-4.png)

这里的三个基本概念与Spring Cloud Config的几个概念的对应关系如下：

- 项目名称：application
- 部署环境：profile
- 配置版本：label

> 这里配置版本（label），我们会默认采用`master`。需要同时存在多个配置版本，实现灰度配置的时候，用户也可以自己添加label。

### 配置管理

配置管理功能是SCCA的核心，在这里用户可以方便对各个应用、各个环境、各个版本的配置进行编辑、加密等操作。同时，也提供了一些快捷的操作，比如：根据环境参数配置一键替换、根据加密Key清单实现一键加密、通过配置中心可以加载到的配置信息等（更多便捷功能持续添加中...）

![配置管理](https://github.com/dyc87112/spring-cloud-config-admin/raw/master/statics/images/user-guilder-5.png)


# 客户端接入

本页主要提供给没有使用过Spring Cloud Config的用户阅读。如果您已经使用过Spring Cloud Config，那么客户端如何通过Spring Cloud Config的配置中心加载配置相信已经掌握，在使用本项目的时候，无非就是搭建SCCA-REST模块和SCCA-UI模块来帮助管理您目前的配置内容。

## 客户端加载

通过前面几节内容，如果您已经完成了SCCA中几个要素的搭建，下面就来看看如何创建一个Spring Boot项目并通过配置中心来加载配置信息。

### 绝对地址接入

**1. 创建一个基本的Spring Boot项目，并在pom.xml中引入依赖**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

**2. 创建应用主类**

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}
```

**3. 创建`bootstrap.properties`配置文件（也可以使用yaml可以）**

```yaml
spring.application.name=config-client
server.port=12000

spring.cloud.config.uri=http://localhost:10032/scca-config-server
spring.cloud.config.profile=stage
spring.cloud.config.label=master
```

上述配置参数与scca中维护元素的对应关系如下：

- spring.application.name：对应scca中的项目名
- spring.cloud.config.profile：项目配置的环境名
- spring.cloud.config.label：项目配置的版本名
- spring.cloud.config.uri：配置中心的访问绝对地址

### 服务发现接入

**1. 创建一个基本的Spring Boot项目，并在pom.xml中引入依赖**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

> 上面以通过eureka做注册中心的依赖，如果用consul，只需要将`spring-cloud-starter-eureka`换成`spring-cloud-starter-consul-discovery`即可。

**2. 创建应用主类**

```java
@EnableDiscoveryClient
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}
```

**3. 创建`bootstrap.properties`配置文件（也可以使用yaml可以）**

```yaml
spring.application.name=config-client
server.port=12000

spring.cloud.config.discovery.enabled=true
spring.cloud.config.discovery.serviceId=config-server
spring.cloud.config.profile=stage
spring.cloud.config.label=master
```

上述配置参数与scca中维护元素的对应关系如下：

- spring.application.name：对应scca中的项目名
- spring.cloud.config.profile：项目配置的环境名
- spring.cloud.config.label：项目配置的版本名
- spring.cloud.config.discovery.enabled：开启服务发现功能
- spring.cloud.config.discovery.serviceId：配置中心的服务名

### 读取配置

通过上面的两种方式从配置中心拉取配置之后，在Spring Boot项目中就可以轻松的使用所有配置内容了，比如：

```java
@RefreshScope
@RestController
public class TestController {

    @Value("${a.b.c}")
    private String abc;

    @RequestMapping("/abc")
    public String abc() {
        return this.abc;
    }

}
```

两个主要注解的说明：

- `@Value("${a.b.c}")`：读取配置key为`a.b.c`的value值
- `@RefreshScope`：下面的配置信息可以通过`/refresh`端点实现动态刷新

## 其他参考

如果您还不了解Spring Cloud Config，您也可以阅读下面的几篇了解一下最原始的Spring Cloud Config配置中心和客户端接入方式

- [Spring Cloud构建微服务架构：分布式配置中心](http://blog.didispace.com/spring-cloud-starter-dalston-3/)
- [Spring Cloud构建微服务架构：分布式配置中心（加密与解密）](http://blog.didispace.com/spring-cloud-starter-dalston-3-2)
- [Spring Cloud构建微服务架构：分布式配置中心（高可用与动态刷新）](http://blog.didispace.com/springcloud4-2/)

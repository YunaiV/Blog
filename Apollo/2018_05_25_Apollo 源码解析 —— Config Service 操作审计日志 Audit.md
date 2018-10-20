title: Apollo 源码解析 —— Config Service 操作审计日志 Audit
date: 2018-05-25
tags:
categories: Apollo
permalink: Apollo/config-service-audit

-------

摘要: 原创出处 http://www.iocoder.cn/Apollo/config-service-audit/ 「芋道源码」欢迎转载，保留摘要，谢谢！

- [1. 概述](http://www.iocoder.cn/Apollo/config-service-audit/)
- [2. Audit](http://www.iocoder.cn/Apollo/config-service-audit/)
- [3. AuditService](http://www.iocoder.cn/Apollo/config-service-audit/)
- [4. AuditRepository](http://www.iocoder.cn/Apollo/config-service-audit/)
- [666. 彩蛋](http://www.iocoder.cn/Apollo/config-service-audit/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2018_05_18.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：  
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表  
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**  
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。  
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。  
> 5. **认真的**源码交流微信群。

-------

# 1. 概述

> 老艿艿：本系列假定胖友已经阅读过 [《Apollo 官方 wiki 文档》](https://github.com/ctripcorp/apollo/wiki/)

本文分享 Config Service **操作审计日志 Audit** 。在每次在做 ConfigDB 写操作( 增、删、改 )操作时，都会记录一条 Audit 日志，用于未来的审计追溯。

> 老艿艿：这种实践方式，非常适用于我们做的**管理平台**。

# 2. Audit

`com.ctrip.framework.apollo.biz.entity.Audit` ，继承 BaseEntity 抽象类，Audit **实体**。代码如下：

```Java
@Entity
@Table(name = "Audit")
@SQLDelete(sql = "Update Audit set isDeleted = 1 where id = ?")
@Where(clause = "isDeleted = 0")
public class Audit extends BaseEntity {

    /**
     * 操作枚举
     */
    public enum OP {
        INSERT, UPDATE, DELETE
    }

    /**
     * 实体名
     */
    @Column(name = "EntityName", nullable = false)
    private String entityName;
    /**
     * 实体编号
     */
    @Column(name = "EntityId")
    private Long entityId;
    /**
     * 操作名
     */
    @Column(name = "OpName", nullable = false)
    private String opName;
    /**
     * 备注
     */
    @Column(name = "Comment")
    private String comment;

}
```

* `entityName` + `entityId` 字段，确实一个实体对象。
* `opName` 字段，操作**名**。分成 INSERT、UPDATE、DELETE 三种，在 **OP** 中枚举。
* `comment` 字段，备注。
* 例如：![例子](http://www.iocoder.cn/images/Apollo/2018_05_25/01.png)

> 老艿艿：在**管理平台**中，我比较喜欢再增加几个字段
> 
> * `ip` 字段，请求方的 IP 。
> * `ua` 字段，请求的 User-Agent 。
> * `extras` 字段，数据结果为 **Map** 进行 **JSON** 化，存储**重要**字段。例如，更新用户手机号，那么会存储 `mobile=15601691024` 到 `extras` 字段中。

# 3. AuditService

在 `apollo-biz` 项目中，`com.ctrip.framework.apollo.biz.service.AuditService` ，提供 Aduit  的 **Service** 逻辑给 Admin Service 和 Config Service 。

```Java
@Service
public class AuditService {

    @Autowired
    private AuditRepository auditRepository;

    List<Audit> findByOwner(String owner) {
        return auditRepository.findByOwner(owner);
    }

    List<Audit> find(String owner, String entity, String op) {
        return auditRepository.findAudits(owner, entity, op);
    }

    @Transactional
    void audit(String entityName, Long entityId, Audit.OP op, String owner) {
        Audit audit = new Audit();
        audit.setEntityName(entityName);
        audit.setEntityId(entityId);
        audit.setOpName(op.name());
        audit.setDataChangeCreatedBy(owner);
        auditRepository.save(audit);
    }

    @Transactional
    void audit(Audit audit) {
        auditRepository.save(audit);
    }

}
```

# 4. AuditRepository

`com.ctrip.framework.apollo.biz.repository.AuditRepository` ，继承 `org.springframework.data.repository.PagingAndSortingRepository` 接口，提供 Audit 的**数据访问** 给 Admin Service 和 Config Service 。代码如下：

```Java
public interface AuditRepository extends PagingAndSortingRepository<Audit, Long> {

    @Query("SELECT a from Audit a WHERE a.dataChangeCreatedBy = :owner")
    List<Audit> findByOwner(@Param("owner") String owner);

    @Query("SELECT a from Audit a WHERE a.dataChangeCreatedBy = :owner AND a.entityName =:entity AND a.opName = :op")
    List<Audit> findAudits(@Param("owner") String owner, @Param("entity") String entity, @Param("op") String op);

}
```

# 666. 彩蛋

水更一小篇，美滋滋。

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)



title: Spring Boot 整合 MyBatis
date: 2019-01-01
tag:
categories: Spring Boot
permalink: Spring-Boot/didi/spring-boot-mybatis
author: 翟永超
from_url: http://blog.didispace.com/springbootmybatis/
wechat_url:

-------

摘要: 原创出处 http://blog.didispace.com/springbootmybatis/ 「翟永超」欢迎转载，保留摘要，谢谢！

- [整合MyBatis](http://www.iocoder.cn/Spring-Boot/didi/spring-boot-mybatis/)
- [使用MyBatis](http://www.iocoder.cn/Spring-Boot/didi/spring-boot-mybatis/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

最近项目原因可能会继续开始使用MyBatis，已经习惯于spring-data的风格，再回头看xml的映射配置总觉得不是特别舒服，接口定义与映射离散在不同文件中，使得阅读起来并不是特别方便。

Spring中整合MyBatis就不多说了，最近大量使用Spring Boot，因此整理一下Spring Boot中整合MyBatis的步骤。搜了一下Spring Boot整合MyBatis的文章，方法都比较老，比较繁琐。查了一下文档，实际已经支持较为简单的整合与使用。下面就来详细介绍如何在Spring Boot中整合MyBatis，并通过注解方式实现映射。

# 整合MyBatis

- 新建Spring Boot项目，或以[Chapter1](https://git.oschina.net/didispace/SpringBoot-Learning)为基础来操作

- `pom.xml`中引入依赖

  - 这里用到spring-boot-starter基础和spring-boot-starter-test用来做单元测试验证数据访问

  - 引入连接mysql的必要依赖mysql-connector-java

  - 引入整合MyBatis的核心依赖mybatis-spring-boot-starter

  - 这里不引入spring-boot-starter-jdbc依赖，是由于mybatis-spring-boot-starter中已经包含了此依赖

    ```XML
    <parent>
    	<groupId>org.springframework.boot</groupId>
    	<artifactId>spring-boot-starter-parent</artifactId>
    	<version>1.3.2.RELEASE</version>
    	<relativePath/> <!-- lookup parent from repository -->
    </parent>
    
    <dependencies>
    
    	<dependency>
    		<groupId>org.springframework.boot</groupId>
    		<artifactId>spring-boot-starter</artifactId>
    	</dependency>
    
    	<dependency>
    		<groupId>org.springframework.boot</groupId>
    		<artifactId>spring-boot-starter-test</artifactId>
    		<scope>test</scope>
    	</dependency>
    
    	<dependency>
    		<groupId>org.mybatis.spring.boot</groupId>
    		<artifactId>mybatis-spring-boot-starter</artifactId>
    		<version>1.1.1</version>
    	</dependency>
    
    	<dependency>
    		<groupId>mysql</groupId>
    		<artifactId>mysql-connector-java</artifactId>
    		<version>5.1.21</version>
    	</dependency>
    
    </dependencies>
    ```

- 同之前介绍的使用jdbc和spring-data连接数据库一样，在`application.properties`中配置mysql的连接配置

    ```Properteis
    spring.datasource.url=jdbc:mysql://localhost:3306/test
    spring.datasource.username=root
    spring.datasource.password=123456
    spring.datasource.driver-class-name=com.mysql.jdbc.Driver
    ```

同其他Spring Boot工程一样，简单且简洁的的完成了基本配置，下面看看如何在这个基础下轻松方便的使用MyBatis访问数据库。

# 使用MyBatis

- 在Mysql中创建User表，包含id(BIGINT)、name(INT)、age(VARCHAR)字段。同时，创建映射对象User

    ```Java
    public class User {

        private Long id;
        private String name;
        private Integer age;

        // 省略getter和setter

    }
    ```

- 创建User映射的操作UserMapper，为了后续单元测试验证，实现插入和查询操作

    ```Java
    @Mapper
    public interface UserMapper {

        @Select("SELECT * FROM USER WHERE NAME = #{name}")
        User findByName(@Param("name") String name);

        @Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name}, #{age})")
        int insert(@Param("name") String name, @Param("age") Integer age);

    }
    ```

- 创建Spring Boot主类

    ```Java
    @SpringBootApplication
    public class Application {

        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);
        }

    }
```

- 创建单元测试
  - 测试逻辑：插入一条name=AAA，age=20的记录，然后根据name=AAA查询，并判断age是否为20
  - 测试结束回滚数据，保证测试单元每次运行的数据环境独立

    ```Java
    @RunWith(SpringJUnit4ClassRunner.class)
    @SpringApplicationConfiguration(classes = Application.class)
    public class ApplicationTests {

        @Autowired
        private UserMapper userMapper;

        @Test
        @Rollback
        public void findByName() throws Exception {
            userMapper.insert("AAA", 20);
            User u = userMapper.findByName("AAA");
            Assert.assertEquals(20, u.getAge().intValue());
        }

    }
    ```

完整示例[Chapter3-2-7](https://git.oschina.net/didispace/SpringBoot-Learning)
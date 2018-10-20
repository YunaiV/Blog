title: 【龙飞】使用 Spring MVC 测试 Spring Security OAuth2 API
date: 2019-05-16
tag:
categories: Spring Security
permalink: Spring-Security/longfei/Use-Spring-MVC-to-test-the-Spring-Security-OAuth2-API
author: 龙飞
from_url: http://niocoder.com/2018/05/16/使用Spring-MVC测试Spring-Security-Oauth2-API/

-------

摘要: 原创出处 http://niocoder.com/2018/05/16/使用Spring-MVC测试Spring-Security-Oauth2-API/ 「龙飞」欢迎转载，保留摘要，谢谢！

- [1. 前言](http://www.iocoder.cn/Spring-Security/longfei/Use-Spring-MVC-to-test-the-Spring-Security-OAuth2-API/)
  - [1.1 修改pom.xml](http://www.iocoder.cn/Spring-Security/longfei/Use-Spring-MVC-to-test-the-Spring-Security-OAuth2-API/)
  - [1.2 修改MerryyouResourceServerConfig配置](http://www.iocoder.cn/Spring-Security/longfei/Use-Spring-MVC-to-test-the-Spring-Security-OAuth2-API/)
  - [1.3 增加/user和/forbidden请求映射](http://www.iocoder.cn/Spring-Security/longfei/Use-Spring-MVC-to-test-the-Spring-Security-OAuth2-API/)
  - [1.4 增加测试类SecurityOauth2Test](http://www.iocoder.cn/Spring-Security/longfei/Use-Spring-MVC-to-test-the-Spring-Security-OAuth2-API/)
- [2. 代码下载](http://www.iocoder.cn/Spring-Security/longfei/Use-Spring-MVC-to-test-the-Spring-Security-OAuth2-API/)
- [3. 推荐文章](http://www.iocoder.cn/Spring-Security/longfei/Use-Spring-MVC-to-test-the-Spring-Security-OAuth2-API/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> 不是因为看到希望了才去坚持，而坚持了才知道没有希望。

[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/spring-security-OAuth206.png](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-OAuth206.png "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-OAuth206.png")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-OAuth206.png "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-OAuth206.png")

# 1. 前言

在[Spring Security源码分析十一：Spring Security OAuth2整合JWT](https://longfeizheng.github.io/2018/01/23/Spring-Security%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E5%8D%81%E4%B8%80-Spring-Security-OAuth2%E6%95%B4%E5%90%88JWT/)和[Spring Boot 2.0 整合 Spring Security Oauth2](https://longfeizheng.github.io/2018/04/29/Spring-Boot-2.0-%E6%95%B4%E5%90%88-Spring-Security-Oauth2/)中，我们都是使用[Restlet Client - REST API Testing](https://chrome.google.com/webstore/detail/restlet-client-rest-api-t/aejoelaoggembcahagimdiliamlcdmfm)测试被`Oauth2`保护的`API`。在本章中，我们将展示如何使用`MockMvc`测试`Oauth2`的`API`。

## 1.1 修改pom.xml

添加`spring-security-test`依赖

```xml
 		<dependency>
			<groupId>org.springframework.security</groupId>
			<artifactId>spring-security-test</artifactId>
		</dependency>
```

## 1.2 修改MerryyouResourceServerConfig配置

```java
   @Override
    public void configure(HttpSecurity http) throws Exception {

        // @formatter:off
        http.formLogin()
                .successHandler(appLoginInSuccessHandler)//登录成功处理器
                .and()
                .authorizeRequests()
                .antMatchers("/user").hasRole("USER")
                .antMatchers("/forbidden").hasRole("ADMIN")
                .anyRequest().authenticated().and()
                .csrf().disable();

        // @formatter:ON
    }

```

- 修改`MerryyouResourceServerConfig`配置，增加对指定路径的角色校验。
- 默认角色为`ROLE_USER`，详见`MyUserDetailsService`
```java
@Component
public class MyUserDetailsService implements UserDetailsService {

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return new User(username, "123456", AuthorityUtils.commaSeparatedStringToAuthorityList("ROLE_USER"));
    }
}
```
- 关于为何是`hasRole("USER")`而不是`hasRole("ROLE_USER")`请参考：[Spring Security源码分析十三：Spring Security 基于表达式的权限控制](https://longfeizheng.github.io/2018/01/30/Spring-Security%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E5%8D%81%E4%B8%89-Spring-Security%E6%9D%83%E9%99%90%E6%8E%A7%E5%88%B6/#%E5%B8%B8%E8%A7%81%E7%9A%84%E8%A1%A8%E8%BE%BE%E5%BC%8F)
- ` @formatter:off`和`@formatter:ON`此段代码不进行格式化

## 1.3 增加/user和/forbidden请求映射

```java
@GetMapping("/user")
    public Object getCurrentUser1(Authentication authentication, HttpServletRequest request) throws UnsupportedEncodingException {
        log.info("【SecurityOauth2Application】 getCurrentUser1 authenticaiton={}", JsonUtil.toJson(authentication));

        String header = request.getHeader("Authorization");
        String token = StringUtils.substringAfter(header, "bearer ");

        Claims claims = Jwts.parser().setSigningKey(oAuth2Properties.getJwtSigningKey().getBytes("UTF-8")).parseClaimsJws(token).getBody();
        String blog = (String) claims.get("blog");
        log.info("【SecurityOauth2Application】 getCurrentUser1 blog={}", blog);

        return authentication;
    }

    @GetMapping("/forbidden")
    public String getForbidden() {
        return "forbidden";
    }
```

- `/user`请求需要`USER`角色
- `/forbidden`请求需要`ADMIN`角色

## 1.4 增加测试类SecurityOauth2Test

```java
@RunWith(SpringRunner.class)
@WebAppConfiguration
@SpringBootTest(classes = SecurityOauth2Application.class)
@Slf4j
public class Oauth2MvcTest {

    @Autowired
    private WebApplicationContext wac;

    @Autowired
    private FilterChainProxy springSecurityFilterChain;

    private MockMvc mockMvc;

    //clientId
    final static String CLIENT_ID = "merryyou";
    //clientSecret
    final static String CLIENT_SECRET = "merryyou";
    //用户名
    final static String USERNAME = "admin";
    //密码
    final static String PASSWORD = "123456";

    private static final String CONTENT_TYPE = "application/json;charset=UTF-8";

    @Before
    public void setup() {
        this.mockMvc = MockMvcBuilders.webAppContextSetup(this.wac).addFilter(springSecurityFilterChain).build();//初始化MockMvc对象,添加Security过滤器链
    }
```

- 初始化`Oauth2`信息


### 1.4.1 obtainAccessToken

```java
  public String obtainAccessToken() throws Exception {
        final MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("grant_type", "password");
        params.add("client_id", CLIENT_ID);
        params.add("username", USERNAME);
        params.add("password", PASSWORD);

        // @formatter:off

        ResultActions result = mockMvc.perform(post("/oauth/token")
                .params(params)
                .with(httpBasic(CLIENT_ID, CLIENT_SECRET))
                .accept(CONTENT_TYPE))
                .andExpect(status().isOk())
                .andExpect(content().contentType(CONTENT_TYPE));

        // @formatter:on

        String resultString = result.andReturn().getResponse().getContentAsString();

        JacksonJsonParser jsonParser = new JacksonJsonParser();
//        System.out.println(jsonParser.parseMap(resultString).get("access_token").toString());
        return jsonParser.parseMap(resultString).get("access_token").toString();
    }
```

### 1.4.2 测试obtainAccessToken

```java
 @Test
    public void getAccessToken() throws Exception {
        final String accessToken = obtainAccessToken();
        log.info("access_token={}", accessToken);
    }
```

控制台打印：

```java
access_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJhZG1pbiIsInNjb3BlIjpbImFsbCJdLCJleHAiOjE1MjY0NjEwMzgsImJsb2ciOiJodHRwczovL2xvbmdmZWl6aGVuZy5naXRodWIuaW8vIiwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImE1MmE2NDI4LTcwNzctNDcwZC05M2MwLTc0ZWNlNjFhYTlkMCIsImNsaWVudF9pZCI6Im1lcnJ5eW91In0.CPmkZmfOkgDII29RMIoMO7ufAe5WFrQDB7SaMDKa128
```

### 1.4.3 UnauthorizedTest

```java
    /**
     * 未授权 401
     *
     * @throws Exception
     */
    @Test
    public void UnauthorizedTest() throws Exception {
//        mockMvc.perform(get("/user")).andExpect(status().isUnauthorized());
        ResultActions actions = mockMvc.perform(get("/user"));
        int status = actions.andReturn().getResponse().getStatus();
        Assert.assertTrue(status == HttpStatus.UNAUTHORIZED.value());
    }
```

- 未授权 `401`

### 1.4.4 forbiddenTest

```java
   /**
     * 禁止访问 403
     *
     * @throws Exception
     */
    @Test
    public void forbiddenTest() throws Exception {
        final String accessToken = obtainAccessToken();
        log.info("access_token={}", accessToken);
        mockMvc.perform(get("/forbidden").header("Authorization", "bearer " + accessToken)).andExpect(status().isForbidden());
    }
```

- 禁止访问 `403`

### 1.4.5 accessTokenOk

```java
    /**
     * 允许访问 200
     *
     * @throws Exception
     */
    @Test
    public void accessTokenOk() throws Exception {
        final String accessToken = obtainAccessToken();
        log.info("access_token={}", accessToken);
        mockMvc.perform(get("/user").header("Authorization", "bearer " + accessToken)).andExpect(status().isOk());
    }
```

- 允许访问 `200`

# 2. 代码下载

- github:[https://github.com/longfeizheng/security-oauth2](https://github.com/longfeizheng/security-oauth2)
- gitee:[https://gitee.com/merryyou/security-oauth2](https://gitee.com/merryyou/security-oauth2)

# 3. 推荐文章

1. [Java创建区块链系列](https://longfeizheng.github.io/categories/#%E5%8C%BA%E5%9D%97%E9%93%BE)
2. [Spring Security源码分析系列](https://longfeizheng.github.io/categories/#Security)
3. [Spring Data Jpa 系列](https://longfeizheng.github.io/categories/#JPA)
4. [【译】数据结构中关于树的一切（java版）](https://longfeizheng.github.io/2018/04/16/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E4%B8%AD%E4%BD%A0%E9%9C%80%E8%A6%81%E7%9F%A5%E9%81%93%E7%9A%84%E5%85%B3%E4%BA%8E%E6%A0%91%E7%9A%84%E4%B8%80%E5%88%87/)
5. [SpringBoot+Docker+Git+Jenkins实现简易的持续集成和持续部署](https://longfeizheng.github.io/2018/04/22/SpringBoot+Docker+Git+Jenkins%E5%AE%9E%E7%8E%B0%E7%AE%80%E6%98%93%E7%9A%84%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E5%92%8C%E9%83%A8%E7%BD%B2/)
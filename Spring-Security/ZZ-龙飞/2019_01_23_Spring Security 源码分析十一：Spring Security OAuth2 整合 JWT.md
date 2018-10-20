title: 【龙飞】Spring Security 源码分析十一：Spring Security OAuth2 整合 JWT
date: 2019-01-23
tag:
categories: Spring Security
permalink: Spring-Security/longfei/Spring-Security-OAuth2-integrates-JWT
author: 龙飞
from_url: http://niocoder.com/2018/01/23/Spring-Security源码分析十一-Spring-Security-OAuth2整合JWT/

-------

摘要: 原创出处 http://niocoder.com/2018/01/23/Spring-Security源码分析十一-Spring-Security-OAuth2整合JWT/ 「龙飞」欢迎转载，保留摘要，谢谢！

- [1. JWT组成](http://www.iocoder.cn/Spring-Security/longfei/Spring-Security-OAuth2-integrates-JWT/)
  - [1.1 Header](http://www.iocoder.cn/Spring-Security/longfei/Spring-Security-OAuth2-integrates-JWT/)
  - [1.2  Claims (Payload)](http://www.iocoder.cn/Spring-Security/longfei/Spring-Security-OAuth2-integrates-JWT/)
  - [1.3 Signature](http://www.iocoder.cn/Spring-Security/longfei/Spring-Security-OAuth2-integrates-JWT/)
  - [1.4 JWT流程示意图](http://www.iocoder.cn/Spring-Security/longfei/Spring-Security-OAuth2-integrates-JWT/)
  - [1.5 Spring Security Oauth2 实现JWT](http://www.iocoder.cn/Spring-Security/longfei/Spring-Security-OAuth2-integrates-JWT/)
- [2. 代码下载](http://www.iocoder.cn/Spring-Security/longfei/Spring-Security-OAuth2-integrates-JWT/)

-------

![](http://www.iocoder.cn/images/common/wechat_mp_2017_07_31.jpg)

> 🙂🙂🙂关注**微信公众号：【芋道源码】**有福利：
> 1. RocketMQ / MyCAT / Sharding-JDBC **所有**源码分析文章列表
> 2. RocketMQ / MyCAT / Sharding-JDBC **中文注释源码 GitHub 地址**
> 3. 您对于源码的疑问每条留言**都**将得到**认真**回复。**甚至不知道如何读源码也可以请教噢**。
> 4. **新的**源码解析文章**实时**收到通知。**每周更新一篇左右**。
> 5. **认真的**源码交流微信群。

-------

> Json web token (JWT), 是为了在网络应用环境间传递声明而执行的一种基于JSON的开放标准（[RFC 7519](https://tools.ietf.org/html/rfc7519)).该token被设计为紧凑且安全的，特别适用于分布式站点的单点登录（SSO）场景。JWT的声明一般被用来在身份提供者和服务提供者间传递被认证的用户身份信息，以便于从资源服务器获取资源，也可以增加一些额外的其它业务逻辑所必须的声明信息，该token也可直接被用于认证，也可被加密。

# 1. JWT组成

[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt01.png](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt01.png "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt01.png")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt01.png "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt01.png")
```shell
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJhZG1pbiIsInNjb3BlIjpbImFsbCJdLCJleHAiOjE1MTY3MjY4MTMsImJsb2ciOiJodHRwczovL2xvbmdmZWl6aGVuZy5naXRodWIuaW8vIiwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImJmZmY0NjRjLTFiNTktNGZkNy1hNTE4LWU3YjY5MDFiNzU3YyIsImNsaWVudF9pZCI6Im1lcnJ5eW91In0.gp5t9nY9mGp5O2-yqdflc0nEAsTeCQG7VugA8q8XcF4
```

## 1.1 Header

Header 包含了一些元数据，至少会表明 token 类型以及 签名方法。
```json
{
 "typ": "JWT",
 "alg": "HS256"
}
```

## 1.2  Claims (Payload)
Claims 部分包含了一些跟这个 token 有关的重要信息。
```json
{
 "user_name": "admin",
 "scope": [
  "all"
 ],
 "exp": 1516726813,
 "blog": "https://longfeizheng.github.io/",
 "authorities": [
  "ROLE_USER"
 ],
 "jti": "bfff464c-1b59-4fd7-a518-e7b6901b757c",
 "client_id": "merryyou"
}
```

## 1.3 Signature

JWT 标准遵照 JSON Web Signature (JWS) 标准来生成签名。签名主要用于验证 token 是否有效，是否被篡改。

## 1.4 JWT流程示意图

[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt02.jpg](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt02.jpg "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt02.jpg")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt02.jpg "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt02.jpg")

## 1.5 Spring Security Oauth2 实现JWT

### 1.5.1 配置TokenStoreConfig用于存储Token

```java
@Configuration
public class TokenStoreConfig {
    /**
     * redis连接工厂
     */
    @Autowired
    private RedisConnectionFactory redisConnectionFactory;

    /**
     * 用于存放token
     * @return
     */
    @Bean
    @ConditionalOnProperty(prefix = "merryyou.security.oauth2", name = "storeType", havingValue = "redis")
    public TokenStore redisTokenStore() {
        return new RedisTokenStore(redisConnectionFactory);
    }

    /**
     * jwt TOKEN配置信息
     */
    @Configuration
    @ConditionalOnProperty(prefix = "merryyou.security.oauth2", name = "storeType", havingValue = "jwt", matchIfMissing = true)
    public static class JwtTokenCofnig{

        /**
         * 使用jwtTokenStore存储token
         * @return
         */
        @Bean
        public TokenStore jwtTokenStore(){
            return new JwtTokenStore(jwtAccessTokenConverter());
        }

        /**
         * 用于生成jwt
         * @return
         */
        @Bean
        public JwtAccessTokenConverter jwtAccessTokenConverter(){
            JwtAccessTokenConverter accessTokenConverter = new JwtAccessTokenConverter();
            accessTokenConverter.setSigningKey("merryyou");//生成签名的key
            return accessTokenConverter;
        }

        /**
         * 用于扩展JWT
         * @return
         */
        @Bean
        @ConditionalOnMissingBean(name = "jwtTokenEnhancer")
        public TokenEnhancer jwtTokenEnhancer(){
            return new MerryyouJwtTokenEnhancer();
        }

    }
}

```

#### 1.5.1.1 MerryyouJwtTokenEnhancer

```java
public class MerryyouJwtTokenEnhancer implements TokenEnhancer {
    @Override
    public OAuth2AccessToken enhance(OAuth2AccessToken accessToken, OAuth2Authentication authentication) {
        Map<String, Object> info = new HashMap<>();
        info.put("blog", "https://longfeizheng.github.io/");//扩展返回的token
        ((DefaultOAuth2AccessToken) accessToken).setAdditionalInformation(info);
        return accessToken;
    }
}

```

### 1.5.2 配置认证服务器

```java
@Configuration
@EnableAuthorizationServer
public class MerryyouAuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private UserDetailsService userDetailsService;

    @Autowired
    private TokenStore tokenStore;

    @Autowired(required = false)
    private JwtAccessTokenConverter jwtAccessTokenConverter;

    @Autowired(required = false)
    private TokenEnhancer jwtTokenEnhancer;

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.tokenStore(tokenStore)
                .authenticationManager(authenticationManager)
                .userDetailsService(userDetailsService);
        //扩展token返回结果
        if (jwtAccessTokenConverter != null && jwtTokenEnhancer != null) {
            TokenEnhancerChain tokenEnhancerChain = new TokenEnhancerChain();
            List<TokenEnhancer> enhancerList = new ArrayList();
            enhancerList.add(jwtTokenEnhancer);
            enhancerList.add(jwtAccessTokenConverter);
            tokenEnhancerChain.setTokenEnhancers(enhancerList);
            //jwt
            endpoints.tokenEnhancer(tokenEnhancerChain)
                    .accessTokenConverter(jwtAccessTokenConverter);
        }
    }

    /**
     * 配置客户端一些信息
     * @param clients
     * @throws Exception
     */
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("merryyou")
                .secret("merryyou")
                .accessTokenValiditySeconds(7200)
                .authorizedGrantTypes("refresh_token", "password", "authorization_code")//OAuth2支持的验证模式
                .scopes("all");
    }
}
```

### 1.5.3 配置资源服务器

```java
@Configuration
@EnableResourceServer
public class MerryyouResourceServerConfig extends ResourceServerConfigurerAdapter {

    /**
     * 自定义登录成功处理器
     */
    @Autowired
    private AuthenticationSuccessHandler appLoginInSuccessHandler;

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.formLogin()
                .successHandler(appLoginInSuccessHandler)//登录成功处理器
                .and()
                .authorizeRequests().anyRequest().authenticated().and()
                .csrf().disable();
    }

}
```

### 1.5.4 解析扩展的Token

```java
 @GetMapping("/user")
    public Object getCurrentUser1(Authentication authentication, HttpServletRequest request) throws UnsupportedEncodingException {
        log.info("【SecurityOauth2Application】 getCurrentUser1 authenticaiton={}", JsonUtil.toJson(authentication));

        String header = request.getHeader("Authorization");
        String token = StringUtils.substringAfter(header, "bearer ");

        Claims claims = Jwts.parser().setSigningKey("merryyou".getBytes("UTF-8")).parseClaimsJws(token).getBody();
        String blog = (String) claims.get("blog");
        log.info("【SecurityOauth2Application】 getCurrentUser1 blog={}", blog);

        return authentication;
    }
```

### 1.5.5 测试方法

```java
    @Test
    public void signInTest() throws Exception {
        RestTemplate rest = new RestTemplate();
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        headers.add("authorization", getBasicAuthHeader());

        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("username", "admin");
        params.add("password", "123456");

        HttpEntity<?> entity = new HttpEntity(params, headers);
        // pay attention, if using get with headers, should use exchange instead of getForEntity / getForObject
        ResponseEntity<String> result = rest.exchange(SIGN_IN_URI, HttpMethod.POST, entity, String.class, new Object[]{null});
        log.info("登录信息返回的结果={}", JsonUtil.toJson(result));
    }
```

打印：

```java
 "body": "{\"access_token\":\"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJhZG1pbiIsInNjb3BlIjpbImFsbCJdLCJleHAiOjE1MTY3MjkxNDIsImJsb2ciOiJodHRwczovL2xvbmdmZWl6aGVuZy5naXRodWIuaW8vIiwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6IjMzOTUxNDk1LTBjOGYtNGQ5NS1iZDYyLTAxMjEyYWNjZDU1ZCIsImNsaWVudF9pZCI6Im1lcnJ5eW91In0.7Lrpmn3CaNweqcMeADJeZJGDTEZYN-gg5OpAzbKIEqQ\",\"token_type\":\"bearer\",\"refresh_token\":\"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJhZG1pbiIsInNjb3BlIjpbImFsbCJdLCJhdGkiOiIzMzk1MTQ5NS0wYzhmLTRkOTUtYmQ2Mi0wMTIxMmFjY2Q1NWQiLCJleHAiOjE1MTkzMTM5NDIsImJsb2ciOiJodHRwczovL2xvbmdmZWl6aGVuZy5naXRodWIuaW8vIiwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6IjFlMjI1YzE5LTE5NDMtNGNjMi1iYTdjLTM1MzdmZDA1M2E4MyIsImNsaWVudF9pZCI6Im1lcnJ5eW91In0.lKHgXd2HSPCp2cK6S-ZvLUwXRjnXEX9wryDWV4CmSGw\",\"expires_in\":7199,\"scope\":\"all\",\"blog\":\"https://longfeizheng.github.io/\",\"jti\":\"33951495-0c8f-4d95-bd62-01212accd55d\"}"
```

效果如下：

[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt03.gif](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt03.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt03.gif")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt03.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-jwt03.gif")

# 2. 代码下载

从我的 github 中下载，[https://github.com/longfeizheng/security-oauth2](https://github.com/longfeizheng/security-oauth2)




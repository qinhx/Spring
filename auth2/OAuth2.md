### Auth2开发指南



` @EnableAuthorizationServer`  注解用于配置auth2.0的授权服务器的机制，与任何实现`AuthorizationServerConfigurer`  的`bean` 一起完成配置

**通常是通过继承 ** `AuthorizationServerConfigurerAdapter` 

```java
public class AuthorizationServerConfigurerAdapter implements AuthorizationServerConfigurer {
    public AuthorizationServerConfigurerAdapter() {
    }

    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
    }

    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    }

    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
    }
}
```




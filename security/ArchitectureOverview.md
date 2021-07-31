<center> <h1>Architecture Overview</h1></center>

### **1.核心组件**

* **1.1 SecurityContextHolder**

  用于存储安全相关的上下文，当前操作的用户是谁等

因为身份信息是与线程绑定的，所以可以在程序的任何地方使用静态方法获取用户信息。一个典型的获取当前登录用户的姓名的例子如下所示：

```java
Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
if (principal instanceof UserDetails) {   
    String username = ((UserDetails)principal).getUsername();
} else {   
    String username = principal.toString();
}
```

getAuthentication()返回了认证信息，再次getPrincipal()返回了身份信息，UserDetails便是Spring对身份信息封装的一个接口。Authentication和UserDetails的介绍在下面的小节具体讲解，本节重要的内容是介绍SecurityContextHolder这个容器。

```java
public class SecurityContextHolder {
    public static final String MODE_THREADLOCAL = "MODE_THREADLOCAL";
    public static final String MODE_INHERITABLETHREADLOCAL = "MODE_INHERITABLETHREADLOCAL";
    public static final String MODE_GLOBAL = "MODE_GLOBAL";
    public static final String SYSTEM_PROPERTY = "spring.security.strategy";
    private static String strategyName = System.getProperty("spring.security.strategy");
    private static SecurityContextHolderStrategy strategy;
    private static int initializeCount = 0;

    public SecurityContextHolder() {
    }

    public static void clearContext() {
        strategy.clearContext();
    }

    public static SecurityContext getContext() {
        return strategy.getContext();
    }

    public static int getInitializeCount() {
        return initializeCount;
    }

    private static void initialize() {
        if (!StringUtils.hasText(strategyName)) {
            strategyName = "MODE_THREADLOCAL";
        }

        if (strategyName.equals("MODE_THREADLOCAL")) {
            strategy = new ThreadLocalSecurityContextHolderStrategy();
        } else if (strategyName.equals("MODE_INHERITABLETHREADLOCAL")) {
            strategy = new InheritableThreadLocalSecurityContextHolderStrategy();
        } else if (strategyName.equals("MODE_GLOBAL")) {
            strategy = new GlobalSecurityContextHolderStrategy();
        } else {
            try {
                Class<?> clazz = Class.forName(strategyName);
                Constructor<?> customStrategy = clazz.getConstructor();
                strategy = (SecurityContextHolderStrategy)customStrategy.newInstance();
            } catch (Exception var2) {
                ReflectionUtils.handleReflectionException(var2);
            }
        }

        ++initializeCount;
    }
    //.......
}
```

* **Authentication**

```java
public interface Authentication extends Principal, Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();

    Object getCredentials();

    Object getDetails();

    Object getPrincipal();

    boolean isAuthenticated();

    void setAuthenticated(boolean var1) throws IllegalArgumentException;
}

```

- getAuthorities()，权限信息列表，默认是GrantedAuthority接口的一些实现类，通常是代表权限信息的一系列字符串。
- getCredentials()，密码信息，用户输入的密码字符串，在认证过后通常会被移除，用于保障安全。
- getDetails()，细节信息，web应用中的实现接口通常为 WebAuthenticationDetails，它记录了访问者的ip地址和sessionId的值。
- getPrincipal()大部分情况下返回的是UserDetails接口的实现类，也是框架中的常用接口之一。UserDetails接口将会在下面的小节重点介绍。

<center> <h2>Spring Security是如何完成身份认证的？</h2> </center>

1 用户名和密码被过滤器获取到，封装成`Authentication`,通常情况下是`UsernamePasswordAuthenticationToken`这个实现类。

2 `AuthenticationManager` 身份管理器负责验证这个`Authentication`

3 认证成功后，`AuthenticationManager`身份管理器返回一个被填充满了信息的（包括上面提到的权限信息，身份信息，细节信息，但密码通常会被移除）`Authentication`实例。

4 `SecurityContextHolder`安全上下文容器将第3步填充了信息的`Authentication`，通过SecurityContextHolder.getContext().setAuthentication(…)方法，设置到其中。

```java
public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean {
    private static final Log logger = LogFactory.getLog(ProviderManager.class);
    private AuthenticationEventPublisher eventPublisher;
    private List<AuthenticationProvider> providers;
    protected MessageSourceAccessor messages;
    private AuthenticationManager parent;
    private boolean eraseCredentialsAfterAuthentication;
    //....

    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        Class<? extends Authentication> toTest = authentication.getClass();
        AuthenticationException lastException = null;
        AuthenticationException parentException = null;
        Authentication result = null;
        Authentication parentResult = null;
        boolean debug = logger.isDebugEnabled();
        Iterator var8 = this.getProviders().iterator();

        while(var8.hasNext()) {
            AuthenticationProvider provider = (AuthenticationProvider)var8.next();
            if (provider.supports(toTest)) {
                if (debug) {
                    logger.debug("Authentication attempt using " + provider.getClass().getName());
                }

                try {
                    result = provider.authenticate(authentication);
                    if (result != null) {
                        this.copyDetails(authentication, result);
                        break;
                    }
                } catch (InternalAuthenticationServiceException | AccountStatusException var13) {
                    this.prepareException(var13, authentication);
                    throw var13;
                } catch (AuthenticationException var14) {
                    lastException = var14;
                }
            }
        }

        if (result == null && this.parent != null) {
            try {
                result = parentResult = this.parent.authenticate(authentication);
            } catch (ProviderNotFoundException var11) {
            } catch (AuthenticationException var12) {
                parentException = var12;
                lastException = var12;
            }
        }

        if (result != null) {
            if (this.eraseCredentialsAfterAuthentication && result instanceof CredentialsContainer) {
                ((CredentialsContainer)result).eraseCredentials();
            }

            if (parentResult == null) {
                this.eventPublisher.publishAuthenticationSuccess(result);
            }

            return result;
        } else {
            if (lastException == null) {
                lastException = new ProviderNotFoundException(this.messages.getMessage("ProviderManager.providerNotFound", new Object[]{toTest.getName()}, "No AuthenticationProvider found for {0}"));
            }

            if (parentException == null) {
                this.prepareException((AuthenticationException)lastException, authentication);
            }

            throw lastException;
        }
    }
     //....

}
```

`ProviderManager` 中的List，会依照次序去认证，认证成功则立即返回，若认证失败则返回null，下一个AuthenticationProvider会继续尝试认证，如果所有认证器都无法认证成功，则`ProviderManager` 会抛出一个ProviderNotFoundException异常。

到这里，如果不纠结于AuthenticationProvider的实现细节以及安全相关的过滤器，认证相关的核心类其实都已经介绍完毕了：身份信息的存放容器SecurityContextHolder，身份信息的抽象Authentication，身份认证器AuthenticationManager及其认证流程。姑且在这里做一个分隔线。下面来介绍下AuthenticationProvider接口的具体实现。


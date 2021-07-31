<center> <h1>security</h1> </center>

认证核心接口AuthcationManager 接收的对象Authcation

实现该接口的ProviderManager

AuthcationProvider  主要是进行认证

AbstractUserDetailsAuthcationProvider 是进行用户密码验证的，其子类DaoAuthcationProvider是进行

UserDetailsService 提供接口,用来加载UserDetails 

InMemoryDetailsManager  提供基于内存对User 进行增删查改



Authcation 就是钥匙的形状

AuthcationProvider 就是钥匙

ProviderManager 就是钥匙串

UserDetailService就是钥匙孔

UserDetails就是锁的内部




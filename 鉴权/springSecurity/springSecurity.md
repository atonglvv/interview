# spring security 优缺点

## 优点

和spring无缝整合。

全面权限控制。

资源社区活跃，丰富。

## 缺点

重量级。



# spring security 基本原理

本质是一个过滤器链。

UsernamePasswordAuthenticationFilter。（认证）

ExceptionTranslationFilter。（异常处理）

FilterSecurityInterceptor。（鉴权，从SecurityContextHolder获取Authentication，获取权限信息判断是否拥有访问当前资源所需的权限）

# spring security 是如何加载的？

DelegatingFilterProxy --> doFilter() --> initDelegate()。



# UserDetailsService接口

查询数据库用户名和密码的过程在这个接口的实现类里自己实现。

# UsernamePasswordAuthenticationFilter

继承UsernamePasswordAuthenticationFilter，重写attemptAuthentication()，successfulAuthentication()，unsuccessfulAuthentication()。

# PasswordEncoder



# 在spring security中自定义用户名与密码

## 配置文件

## 配置类

## 自定义

### 配置类，声明UserDetailsService的实现类

### 编写UserDetailsService的实现类，返回User对象

## 

# 自定义设置登录页面



# 自定义无权限访问页面

```java
http.exceptionHandling().accessDeniedPage("/xxx. html");
```

# hasAuthority 与 hasAnyAuthority的区别？



# hasAuthority 跟 hasRole的区别？



# spring security 中的注解

使用一下注解，主要开启：@EnableGlobalMethodSecurity

## @Secured

## @PreAuthorize

## @PostAuthorize

## @PostFilter

## @PreFilter



# 如何实现自动登录？



# 统一认证授权异常处理

认证异常：AuthenticationEntryPoint（实现该接口）

授权异常：AccessDeniedHandler（实现该接口）



```java
//配置异常处理器
http.exceptionHandling()
	//配置认证失败处理器
	.authenticationEntryPoint(authenticationEntryPoint)
	.accessDeniedHandler(accessDeniedHandler);
```



# spring security 允许跨域

```java
//允许跨域
http.cors();
```



# 如何自定义权限校验方法？

```java
@PreAuthorize("@beanName.methodName(arg)")
```



# 什么是CSRF（cross-site request forgery）？

CSRF是指跨站请求伪造(Cross-site request forgery)，是web常见的攻击之一。








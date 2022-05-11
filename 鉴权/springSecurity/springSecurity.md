# spring security 优缺点

## 优点

和spring无缝整合

全面权限控制

## 缺点

重量级



# spring security 基本原理

本质是一个过滤器链。

FilterSecurityInterceptor。

ExceptionTranslationFilter。

UsernamePasswordAuthenticationFilter。

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



# 什么是CSRF（cross-site request forgery）？

CsrfFilter








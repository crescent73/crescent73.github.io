---
title: spring-security
date: 2022-01-17 15:30:45
categories: 
- web
tags:
- web
- spring
- spring security
---
# spring security

## 1. 简介
### 1.1 概要
Spring是非常流行和成功的Java应用开发框架，Spring Security正是Spring家族中的成员。Spring Security基于Spring框架，提供了一套Web应用安全性的完整解决方案。|

正如你可能知道的关于安全方面的两个主要区域是“认证”和“授权”（或者访问控制），一般来说，Web应用的安全性包括用户认证( Authentication)和用户授权( Authorization)两个部分，这两点也是Spring Security重要核心功能。

1. 用户认证指的是:验证某个用户是否为系统中的合法主体，也就是说用户能否访问该系统。用户认证一般要求用户提供用户名和密码。系统通过校验用户名和密码来完成认证过程。通俗点说就是系统认为**用户是否能登录**
2. 用户授权指的是验证某个用户是否有权限执行某个操作。在一个系统中，不同用户所具有的权限是不同的。比如对一个文件来说，有的用户只能进行读取，而有的用户可以进行修改。一般来说，系统会为不同的用户分配不同的角色，而每个角色则对应一系列的权限。通俗点讲就是系统判断**用户是否有权限**去做某些事情。

### 1.2 对比
Spring Security是 Spring家族中的一个安全管理框架，实际上，在Spring Boot出现之前，Spring Security就已经发展了多年了，但是使用的并不多，安全管理这个领域，一直是Shiro的天下。

相对于Shiro，在SSM中整合Spring Security都是比较麻烦的操作，所以，SpringSecurity虽然功能比Shiro强大，但是使用反而没有Shiro多( Shiro虽然功能没有Spring Security多，但是对于大部分项目而言，Shiro也够用了)。

自从有了Spring Boot之后，Spring Boot对于Spring Security提供了**自动化配置**方案，可以使用更少的配置来使用Spring Security。

因此，一般来说，常见的安全管理技术栈的组合是这样的:
+ SSM + Shiro
+ Spring Boot/Spring Cloud + Spring Security

### 1.3 基本原理
SpringSecurity本质是一个过滤器链，有十多个过滤器。
+ FilterSecurityInterceptor:是一个方法级的权限过滤器,基本位于过滤链的最底部。
+ ExceptionTranslationFilter:是个异常过滤器，用来处理在认证授权过程中抛出的异常
+ usernamePasswordAuthenticationFilter : 对/login的 POST请求做拦截，校验表单中用户名，密码。

### 1.4 UserDetailsService接口
当什么也没有配置的时候，账号和密码是由Spring Security定义生成的。而在实际项目中账号和密码都是从数据库中查询出来的。所以我们要通过自定义逻辑控制认证逻辑。

如果需要自定义逻辑时，只需要实现 UserDetailsService接口即可。

过程：
1. 创建类继承UsernamePasswordAuthenticationFilter，重写三个方法
2. 创建类实现UserDetailService，编写查询数据过程，返回User对象，这个User对象是安全框架提供对象

### 1.5 PasswordEncoder 接口
提供一个加密密码的接口
```
//表示把参数按照特定的解析规则进行解析,
String encode(CharSequence rawPassword);

//表示验证从存储中获取的编码密码与编码后提交的原始密码是否匹配。如果密码匹配，则返回true;如果不匹配，则返回false。第一个参数表示需要被解析的密码。第二个参数表示存储的密码。。
boolean matches(CharSequence rawPassword,String encodedPassword);
//表示如果解析的密码能够再次进行解析且达到更安全的结果则返回true，否则返回false。默认返回false。
default boolean upgradeEncoding(String encodedPassword);
```
BCryptPasswordEncoder是 Spring Security官方推荐的密码解析器，平时多使用这个解析器。

BCryptPasswordEncoder是对 bcrypt.强散列方法的具体实现。是基于Hash算法实现的单向加密。可以通过strength控制加密强度，默认10.

## 2. Spring Security Web权限方案
### 2.1 设置登录系统的账号密码
#### 2.1.1 通过配置文件
```
spring.security.user.name=username
spring.security.user.password=password
```

#### 2.1.2 通过配置类
```
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
        String password = passwordEncoder.encode("123");
        auth.inMemoryAuthentication()
            .withUser("username").password(password).roles("admin");
    }

    @Bean
    PasswordEncoder password() {
        return new BCryptPasswordEncoder();
    }
}

```

#### 2.1.3 自定义编写实现类
1. 创建配置类，设置使用哪个UserDetailService实现类
```
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(password());
    }

    @Bean
    PasswordEncoder password() {
        return new BCryptPasswordEncoder();
    }
}

```
2. 编写实现类，返回User对象，User对象有用户名密码和操作权限
```
@Service("userDetailsService")
public class MyUserDetailsService implements UserDetailsService {
    @Autowired
    private UsersMapper usersMapper;

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        //调用usersMapper方法，根据用户名查询数据库
        QueryWrapper<Users> wrapper = new QueryWrapper();
        // where username=?
        wrapper.eq("username", username);
        Users users = usersMapper.selectOne(wrapper);
        if(users == null) {
            //数据库没有用户名，认证失败
            throw new UsernameNotFoundException("用户名不存在!");
        }
        List<GrantedAuthority> auths = AuthorityUtils.
            commaSeparatedStringToAuthorityList("role");
        return new User(users.getUsername(), new BCryptPasswordEncoder().encode(users.getPassword()), auths);
    }
}

```

### 2.2 自定义登录页面
1. 在配置类实现相关的配置
```
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(password());
    }

    @Bean
    PasswordEncoder password() {
        return new BCryptPasswordEncoder();
    }

    Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin(）  //自定义自己编写的登录页面
            .loginPage("/login.html")  //登录页面设置
            .loginProcessingUrl("/user/login"")  //登录访问路径
            .defaultSuccessUrl("/test/index").permitAll()  //登录成功之后，跳转路径
            .and().authorizeRequests()
                .antMatchers("/"," /test/hello","/user/login') 
                //当前登录用户,只有具有admins权限才可以访问这个路径
                .antMatchers("/test/index").hasAuthority("admins")
            .permitAll()  //设置哪些路径可以直接访问，不需要认证
            .anyRequest().authenticated() // 除了以上请求都需要认证
            .and().csrf().disable();  //关闭csrf防护
    }
}
```

2. 创建相关页面、Controller

### 2.3 基于角色或权限进行访问控制
#### 2.3.1 hasAuthority方法
如果当前的主体具有指定的权限，则返回true,否则返回false

1. 在配置类设置当前访问地址哪些有权限
```
//当前登录用户,只有具有admins权限才可以访问这个路径
.antMatchers("/test/index").hasAuthority("admins")
```
2. 在UserDetailsService,把返回user对象设置权限
```
// 此处角色名称和上面设置的角色名称一致
List<GrantedAuthority> auths = AuthorityUtils.commaSeparatedStringToAuthorityList("admins");
```
错误码403，表示没有访问权限

#### 2.3.2 hasAnyAuthority方法
如果当前的主体有任何提供的角色(给定的作为一个逗号分隔的字符串列表)的话，返回true
1. 在配置类设置当前访问地址哪些有权限
```
//当前登录用户,具有admins和manager权限才可以访问这个路径
.antMatchers("/test/index").hasAnyAuthority("admins,manager")
```

#### 2.3.3 hasRole方法
如果用户具备给定角色就允许访问,否则出现403。
如果当前主体具有指定的角色，则返回true。
1. 在配置类设置当前访问地址哪些有权限
```
//当前登录用户,具有admins和manager权限才可以访问这个路径
.antMatchers("/test/index").hasRole("sale")
```
2. 在UserDetailsService,把返回user对象设置权限
注意：在UserDetailsService里的角色前缀需要添加"ROLE_"前缀
```
// 此处角色名称和上面设置的角色名称一致
List<GrantedAuthority> auths = AuthorityUtils.commaSeparatedStringToAuthorityList("ROLE_sale");
```

#### 2.3.4 hasAnyRole
表示用户具备任何一个条件都可以访问。

#### 2.3.5 自定义403权限访问页面
修改访问配置类
```
http.exceptionHandling().accessDeniedPage("/unauth.html");
```

### 2.4 注解使用
#### 2.4.1 @Secured
判断是否具有角色，另外需要注意的是这里匹配的字符串需要添加前缀"ROLE_ "

使用注解先要开启注解功能!
`EnableGlobalMethodSecurity(securedEnabled=true)`

在Controller上添加注解
```
@RequestMapping("testSecured")
@ResponseBody:
@Secured({"ROLE_normal","ROLE_admin"})
public string helloUser({
    return "hello,user";
}
```

#### 2.4.2 PreAuthorize
先开启注解功能:
`@EnableGlobalMethodSecurity(prePostEnabled = true)`

@PreAuthorize:注解适合进入方法前的权限验证，@PreAuthorize可以将登录用户的roles/permissions参数传到方法中。
```
@ReguestMapping("/preAuthorize")
@ResponseBody
//@PreAuthorize("hasRoLe("ROLE_管理员")")
@PreAuthorize("hasAnyAuthority('menu:system")")
public string preAuthorize(){
    System.out.println("preAuthorize");
    return "preAuthorize";
}
```

#### 2.4.3 PostAuthorize
先开启注解功能:`@EnableGlobalMethodSecurity(prePostEnabled = true)`

@PostAuthorize 注解使用并不多，在方法执行后再进行权限验证，适合验证带有**返回值**的权限.
心
```
@ReguestMapping("/testPostAuthorize")
@ResponseBody
@PostAuthorize("hasAnyAuthority('menu:system")")
public string postAuthorize(){
    System.out.println("postAuthorize");
    return "postAuthorize";
}
```

#### 2.4.4 PostFilter
@PostFilter :权限验证之后对数据进行过滤留下用户名是admin1的数据，
**方法返回的数据进行过滤**
表达式中的filterObject 引用的是方法返回值List中的某一个元素
```
ReguestMapping("getAll")
@PreAuthorize("hasRole("ROLE_管理员")")
@PostFilter("filterObject.username== 'admin1'")
@ResponseBody
public List<UserInfo> getAllUser(){
    ArrayList<userInfo> list = new ArrayList<>();
    list.add(new UserInfo(11, "admin1", "6666"));
    list.add(new UserInfo(21, "admin2" , "888" ));
    return list;
}

```

#### 2.4.5 PreFilter
@PreFilter:进入控制器之前对数据进行过滤，
**对传入方法前的数据进行过滤**
```
ReguestMapping("getTestPreFilter")
@PreAuthorize("hasRole("ROLE_管理员")")
@PreFilter(value = "filterObject.id % 2 == 0")
@ResponseBody
public List<UserInfo> getTestPreFilter(@RequestBody List<UserInfo> list){
    list.forEach(t -> {
        System.out.println(t.getId()+"\t"+t.getUsername());
    })
    return list;
}

```

### 2.4 用户注销
1. 在配置类中添加推出映射地址
```
http.logout().logoutUrl("/1ogout").logoutSuccessUrl("/index.html").permitAll();
```

### 2.5 自动登录
自动登录
+ cookie技术
+ spring scurity实现自动登录

实现原理
![1636425586861.png](https://s2.loli.net/2022/01/19/FUAB7Y8lEtSzxyf.png)

具体实现
1. 创建数据库表
2. 配置类，注入数据源，配置操作数据库对象
```
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    //注入数据源
    @Autowired
    private DataSource dataSource;

    @Bean
    public PersistentTokenRepository persistentTokenRepository() {
        JdbcTokenRepositoryImpl jdbcTokenRepository = new JdbcTokenRepositoryImpl();jdbcTokenRepository.setDataSource(dataSource);
        return jdbcTokenRepository ;
    }
 
}
```
3. 配置类配置自动登录
```
Override
protected void configure(HttpSecurity http) throws Exception {
    http.formLogin(）  //自定义自己编写的登录页面
        .loginPage("/login.html")  //登录页面设置
        .loginProcessingUrl("/user/login"")  //登录访问路径
        .defaultSuccessUrl("/test/index").permitAll()  //登录成功之后，跳转路径
        .and().authorizeRequests()
            .antMatchers("/"," /test/hello","/user/login') 
            //当前登录用户,只有具有admins权限才可以访问这个路径
            .antMatchers("/test/index").hasAuthority("admins")
            .and().rememberMe().tokenRepository(persistentTokenRepository()).
            .tokenValiditySeconds(60) // 设置有效时长
            .userDetailsService(userDetailsService)
        .permitAll()  //设置哪些路径可以直接访问，不需要认证
        .anyRequest().authenticated() // 除了以上请求都需要认证
        .and().csrf().disable();  //关闭csrf防护
}
```

4. 在登录页面添加复选框
`<input type="checkbox" name="remember-me" />`

## 2.6 SCRF
跨站请求伪造(英语:Cross-site request forgery)，也被称为one-click attack或者session riding，通常缩写为CSRF或者XSRF，是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。跟跨网站脚本（XSS）相比，XSS利用的是用户对指定网站的信任，CSRF 利用的是网站对用户网页浏览器的信任。

跨站请求攻击，简单地说，是攻击者通过一些技术手段欺骗用户的浏览器去访问一个自己曾经认证过的网站并运行一些操作（如发邮件，发消息，甚至财产操作如转账和购买商品）。由于浏览器曾经认证过，所以被访问的网站会认为是真正的用户操作而去运行。这利用了web中用户身份验证的一个漏洞:简单的身份验证只能保证请求发自某个用户的浏览器，却不能保证请求本身是用户自愿发出的。

从Spring Security 4.0开始，默认情况下会启用CSRF保护，以防止CSRF攻击应用程序，Spring Security CSRF 会针对PATCH，POST，PUT和DELETE方法进行防护。

## 3. Spring Security 微服务
### 3.1 微服务
1. 介绍
微服务最早由Martin Fowler与James Lewis于2014年共同提出，微服务架构风格是一种使用一套小服务来开发单个应用的方式途径，每个服务运行在自己的进程中,并使用轻量级机制通信，通常是HTTP API，这些服务基于业务能力构建，并能够通过自动化部署机制来独立部署，这些服务使用不同的编程语言实现，以及不同数据存储技术，并保持最低限度的集中式管理。

2. 微服务优势
+ 微服务每个模块就相当于一个单独的项目，代码量明显减少，遇到问题也相对来说比较好解决。,
+ 微服务每个模块都可以使用不同的存储方式（比如有的用redis，有的用mysql等），数据库也是单个模块对应自己的数据库。
+ 微服务每个模块都可以使用不同的开发技术，开发模式更灵活。

3. 微服务本质
+ 微服务，关键其实不仅仅是微服务本身，而是系统要提供一套基础的架构，这种架构使得微服务可以独立的部署、运行、升级，不仅如此，这个系统架构还让微服务与微服务之间在结构上“松耦合”，而在功能上则表现为一个统一的整体。这种所谓的“统一的整体”表现出来的是统一风格的界面，统一的权限管理，统一的安全策略，统一的上线过程，统一的日志和审计方法，统一的调度方式，统一的访问入口等等。
+ 微服务的目的是有效的拆分应用，实现敏捷开发和部署。

4. 实现过程
![1636428445210.png](https://s2.loli.net/2022/01/19/aqvjJU7gKD2HzFC.png)

1. 微服务权限管理案例主要功能：
+ 登录（认证）
+ 添加角色
+ 为角色分配菜单
+ 添加用户
+ 为用户分配角色

### 3.2 实现思路
#### 3.2.1 权限管理数据模型
+ 菜单表 permission(id, pid, name, type, permission_value, path, component, icon, gmt_create, gmt_modifited)
+ 角色表 role(id, role_name, role_code, remark, is_deleted, gmt_create, gmt_modified)
+ 用户表 user(id, username, password, nick_name, salt, token, is_deleted, gmt_create, gmt_modified)
+ 角色菜单关系表 role_permission(id, role_id, permission_id, is_deleted, gmt_create, gmt_modifited)
+ 用户角色关系表 user_role(id, role_id, user_id, is_deleted, gmt_create, gmt_modified)

#### 3.2.2 项目中涉及的技术
+ maven: 项目依赖管理（创建父工程，管理项目依赖版本；创建子模块，使用具体依赖）
+ Spring Boot: 本质上就是spring
+ MyBatisPlus：操作数据库框架
+ Spring Cloud：GateWay网关；注册中心（Nacos）
+ redis: 缓存
+ Jwt：生成token
+ Swagger：接口文档，用于测试
+ vue：前端

### 3.3 项目搭建
#### 3.3.1 搭建项目工程
1. 创建父工程，管理依赖版本
2. 在父工程创建子模块
    + common(service_base工具类；spring_scurity权限配置) 
    + infastructure(api_gateway网关)
    + service(service_acl权限管理微服务模块)
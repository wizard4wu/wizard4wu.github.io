---
title: 从0开始学Spring Security
key: AAA-2022-05-08-from0studySpringSecurity
tags: [Spring-Security]
comment: true
footer: true
show_edit_on_github: false
pageview: true
lightbox: true
aside:
toc: true
show_subscribe: false
lightbox: true
---
### 前置入口

本文仓库：[点击进入](https://github.com/Encyclopedias/wizard-2.0/tree/main/CustomSpringSecurity)

学习文档：[点击进入](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Spring%20Security%20%E8%AF%A6%E8%A7%A3%E4%B8%8E%E5%AE%9E%E6%93%8D)

官方Spring-Security项目地址：[点击进入](https://github.com/spring-projects/spring-security)

## 1. 入门准备

当你加入Springboot 的web依赖包的时候，然后创建一个如下Controller，启动服务后便可通过`localhost:8080/ready/security/test`访问获得Hello World的结果

```java
@RestController
public class ReadySecurityController {

    @GetMapping("/ready/security/test")
    public String testReadySecurity(){
        return "Hello World";
    }
}
```

但是当你的模块引入了spring-security的依赖后，再通过上述的访问会跳到一个登陆页面，如下图所示

```java
spring-security的依赖：
   该依赖在parent的中存在并且已经指定了版本号，因此我们引入即可。
       <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
```

![image-20220429223540654]({{"assets/picture/2022-05/image-20220429223540654.png" | absolute_url}})

出现上述的图片的原因是我们还没有重写`WebSecurityConfigurerAdapter`中的`configure(HttpSecurity http)`方法。该方法的默认实现：

```
  protected void configure(HttpSecurity http) throws Exception {
        this.logger.debug("Using default configure(HttpSecurity). If subclassed this will potentially override subclass configure(HttpSecurity).");
        http.authorizeRequests((requests) -> {
            ((AuthorizedUrl)requests.anyRequest()).authenticated();
        });
        http.formLogin();
        http.httpBasic();
    }

代码中的formLogin()是提供了上述的默认登录页面
```

跳到登陆页面后需要你输入登陆用户名和密码后才能正确返回该API的返回值的，<font color=red>用户名</font>是：User，<font color=red>密码</font>是在服务启动的时候随机生成的字符串，如下图所示：

![image-20220429223815346]({{"assets/picture/2022-05/image-20220429223815346.png" | absolute_url}})

当你在application.yml文件加上你的自己的用户名和密码后，即可输入自己的配置用户名密码即可通过校验获取该API返回的数据，配置如下：

```java
spring:
  security:
    user:
      name: wizard
      password: 1234
```

加上配置后，重新启动服务就不会生成一个随机的密码在控制台输出的。

##  2. 认证与配置访问权限

<font color= red>认证是授权的前置条件:</font>认证是你这次的请求是否合法，授权是当前请求合法，但是我只会给予你部分的访问的资源。例如：同一个网站的管理员和普通用户都可以访问该网站，这就是认证合法的结果；但是管理员可以查看所有的普通用户信息而普通用户只能查看自己的信息，授予不同的权利决定访问资源的范围。

在根据文档创建内存用户时遇到了问题，创建的用户数据代码如下：

```java
自定义的 SecurityConfig 继承了 WebSecurityConfigurerAdapter 并重写：
Note：注意配置的参数

   @Override
    protected void configure(AuthenticationManagerBuilder builder) throws Exception {
        builder.inMemoryAuthentication()
                .withUser("wizard1").password("1234").roles("USER")
                .and()
                .withUser("wizard2").password("12345").roles("USER", "ADMIN");
    }
```

上述是可以在服务启动的时候初始化两个用户到内存中，但是实际上当你用用户名和密码去访问时会报异常：<font color = red>java.lang.IllegalArgumentException: There is no PasswordEncoder mapped for the id "null"</font>，异常显示这个用户没有获取到加密算法，通过debug和[查阅官网](https://docs.spring.io/spring-security/reference/features/authentication/password-storage.html)得知，在spring5.0以前默认使用的是`PlaintextPasswordEncoder`,Spring 5.0以后使用的是`DelegatingPasswordEncoder`。 前者为纯文本的密码比较，可以理解成equals；后者是需要开发者创建用户是要设置加密算法。解决方法的方式很多种，我列举比较容易的两种方法：

```java
解决方案一：
来源于官网
   @Override
    protected void configure(AuthenticationManagerBuilder builder) throws Exception {
        builder.inMemoryAuthentication()
                .withUser("wizard1").password("{noop}1234").roles("USER")
                .and()
                .withUser("wizard2").password("{noop}12345").roles("USER", "ADMIN");
    }

解决方案二：
    @Override
    protected void configure(AuthenticationManagerBuilder builder) throws Exception {
        builder.inMemoryAuthentication().passwordEncoder(NoOpPasswordEncoder.getInstance())
                .withUser("wizard1").password("1234").roles("USER")
                .and()
                .withUser("wizard2").password("12345").roles("USER", "ADMIN");
    }
```

两种方法都是解决用户的密码加密算法问题，第二种方案是易于理解的，直接塞了一个`NoOpPasswordEncoder`，这个spring 5.0以前的`PlaintextPasswordEncoder`都是属于纯文本比较，这种方式肯定不会用于实际开发中的；对于第一种方案我是很好奇，为啥要在前面配置一个花括号，本着求真的精神一探究竟。

<font color=green>5.0以前：</font>

![image-20220501161657353]({{"assets/picture/2022-05/image-20220501161657353.png" | absolute_url}})

<font color=green>5.0以后：</font>

![image-20220501161757001]({{"assets/picture/2022-05/image-20220501161757001.png" | absolute_url}})

**<font size=5px weight=bold>{加密算法}密码背后的原理</font>**:

```java
DelegatingPasswordEncoder 中的matches的方法

	@Override
	public boolean matches(CharSequence rawPassword, String prefixEncodedPassword) {
		if (rawPassword == null && prefixEncodedPassword == null) {
			return true;
		}
        //获取{}内的加密算法，这个也是服务在启动时加载到了一个map中了，如下图所示
		String id = extractId(prefixEncodedPassword);
        //根据key获取到具体的PasswordEncoder的对象
		PasswordEncoder delegate = this.idToPasswordEncoder.get(id);
		if (delegate == null) {
			return this.defaultPasswordEncoderForMatches.matches(rawPassword, prefixEncodedPassword);
		}
        //剔除{}内的所有内容，获取密码文本，拿着加密算法和两个待比较的密码进行校验
		String encodedPassword = extractEncodedPassword(prefixEncodedPassword);
		return delegate.matches(rawPassword, encodedPassword);
	}

PasswordEncoderFactories中的创建的加密算法缓存：
	public static PasswordEncoder createDelegatingPasswordEncoder() {
		String encodingId = "bcrypt";
		Map<String, PasswordEncoder> encoders = new HashMap<>();
		encoders.put(encodingId, new BCryptPasswordEncoder());
		encoders.put("ldap", new org.springframework.security.crypto.password.LdapShaPasswordEncoder());
		encoders.put("MD4", new org.springframework.security.crypto.password.Md4PasswordEncoder());
		encoders.put("MD5", new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("MD5"));
		encoders.put("noop", org.springframework.security.crypto.password.NoOpPasswordEncoder.getInstance());
		encoders.put("pbkdf2", new Pbkdf2PasswordEncoder());
		encoders.put("scrypt", new SCryptPasswordEncoder());
		encoders.put("SHA-1", new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("SHA-1"));
		encoders.put("SHA-256",
				new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("SHA-256"));
		encoders.put("sha256", new org.springframework.security.crypto.password.StandardPasswordEncoder());
		encoders.put("argon2", new Argon2PasswordEncoder());
		return new DelegatingPasswordEncoder(encodingId, encoders);
	}

DaoAuthenticationProvider会在初始化的时候将该所有的加密算法缓存到一个DelegatingPasswordEncoder对象中。
	public DaoAuthenticationProvider() {
		setPasswordEncoder(PasswordEncoderFactories.createDelegatingPasswordEncoder());
	}
```

![image-20220501163255347]({{"assets/picture/2022-05/image-20220501163255347.png" | absolute_url}})

### 2.1 权限和角色访问权限

#### 2.1.1权限 ---> authorities

```java
//对于任何请求只有当请求的用户有CREATE的权限才可以访问
http.authorizeRequests().anyRequest().hasAuthority("CREATE");

//对于任何请求只有当请求的用户有DELETE的权限才可以访问
http.authorizeRequests().anyRequest().hasAuthority("CREATE", "DELETE");

//access方法可以提供EL表达式的写法，因此支持很多复杂的逻辑的，功能强大
//该EL表达式的逻辑：有CREATE的权限但没有RETRIEVE的权限才可以访问
String expression = "hasAuthority('CREATE') and !hasAuthority('Retrieve')";
http.authorizeRequests().anyRequest().access(expression);

/******************************************************************************************/
   protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().anyRequest().hasAnyAuthority("CREATE")
                .and().formLogin()
                .and().httpBasic();
    }

创建用户时赋予的权限：
    protected void configure(AuthenticationManagerBuilder builder) throws Exception {

    UserDetails adminUser = User.withUsername("wizard1").password("1234").authorities("CREATE").build();
    builder.inMemoryAuthentication().passwordEncoder(NoOpPasswordEncoder.getInstance()).withUser(adminUser);
    }
上述可以通过该用户调用API：localhost:8080/ready/security/tes返回Hello World，如果将UserDetail中的authorities中的CREATE转换成DELETE，就会导致访问失败，返回403.

    hasAnyAuthority(String)	允许具有任一权限的用户进行访问
    hasAuthority(String)	允许具有特定权限的用户进行访问

```

#### 2.1.2 角色 --->role

```java
//只允许Admin权限的用户访问
http.authorizeRequests().anyRequest().hasRole("ADMIN");

/******************************************************************************************/

//create a new admin user
 UserDetails adminUser = User.withUsername("wizard1").password("1234").roles("ADMIN").build();
        builder.inMemoryAuthentication().passwordEncoder(NoOpPasswordEncoder.getInstance()).withUser(adminUser);

把ADMIN改成USER就会提示403.

    hasRole(String)	允许具有特定角色的用户进行访问
    hasAnyRole(String)	允许具有任一角色的用户进行访问
    denyAll()	无条件禁止一切访问
    permitAll()	无条件允许一切访问
    anonymous()	允许匿名访问
    authenticated()	允许认证用户访问

```

总结：无论是authorities还是role都是都是控制当前用户的访问的资源，不同的权限方法的资源范围不一样。 因此，上述是从用户角度出发来实现访问的安全。



### 2.2 匹配器

匹配器是从API的角度出发，限制哪些用户可以访问，哪些不能访问，哪些路径下的API都是可以访问的。

#### 2.2.1 MVC匹配器

```java
 http.authorizeRequests()
                .mvcMatchers("/ready/security/get").hasRole("USER")
                .mvcMatchers("/ready/security/test").hasRole("ADMIN")
                .and().httpBasic();
对于上述的配置，有ADMIN角色的用户可以使用localhost:8080/ready/security/test访问成功，USER角色用户可以使用localhost:8080/ready/security/get访问成功，注意的是开始处没有"/"。
```

Note：必须要带上httpBasic()，上述的配置才会生效，没有httpBasic()会被当做匿名处理，这是文档上没有涉及的，当然你也可以自定义认证方式。

1. 由于上述配置没有配置其他路径，因此该服务下的其他API都可以直接被访问，相当于`anyRequest().permitAll()`。当然如果不想要其他API都能被访问的话可以设置成`anyRequest().authenticated()`.

2. API的路径完全一致，只是请求方法不同处理:可以使用`mvcMatchers(HttpMethod method, String... mvcPatterns)`, 如果请求方法和请求路径都相同的话，服务在启动的时候会报错。

   ```java
        http.csrf().disable().authorizeRequests()
                   .mvcMatchers( HttpMethod.GET,"/ready/security/samePath/test").hasRole("USER")
                   .mvcMatchers( HttpMethod.DELETE, "/ready/security/samePath/test").hasRole("ADMIN")
                   .and().httpBasic();
   Note：csrf在POST、PUT、DELETE、PATCH这些写数据的请求方式中默认开启，所以在测试请求方式不同路径相同时，存在写数据的请求方式请把csrf关闭。通常在实践中也是会关闭的，毕竟每个公司都想使用自己的自定义的实现CSRF的方式。
       通配符匹配： /ready/**  ---> 匹配0个或多个目录，可以访问该路径下所有的API
                  /ready/*   ---> 匹配0个或多个字符可以访问该路径下一个路径为任意的API， 例如：/ready/a, /ready/b, 							   /ready/c都可以，但是/ready/a/a就不可以了；
                 /ready/?    ---> 还可以使用？进行单个字符匹配；
   ```



#### 2.2.2 ANT匹配器

方法与mvc匹配器类似， 区别：antMatcher(String antPattern)- 允许配置HttpSecurity仅在匹配提供的蚂蚁模式时调用。mvcMatcher(String mvcPattern)- 允许配置HttpSecurity仅在匹配提供的 Spring MVC 模式时调用。通常mvcMatcher比antMatcher. 举个例子：antMatchers("/secured")只匹配确切的 /securedURL，mvcMatchers("/secured")匹配/secured以及/secured/,/secured.html,/secured.xyz

#### 2.2.3 正则匹配器

正则是可以使用正则表达式来实现访问的API的匹配，功能更加强大。

### 2.3授权原理

![image-20220503204009252]({{"assets/picture/2022-05/image-20220503204009252.png" | absolute_url}})

![image-20220503205307679]({{"assets/picture/2022-05/image-20220503205307679.png" | absolute_url}})

+  `AffirmativeBased` ---- 一票赞成，一票否决，取决投票器的优先级(默认的策略)
+ `UnanimousBased` ---- 一票否决
+ `ConsensusBased` ----少数服从多数

原理在这篇文档中有了很详细的描述:[点击进入](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Spring%20Security%20%E8%AF%A6%E8%A7%A3%E4%B8%8E%E5%AE%9E%E6%93%8D/06%20%20%E6%9D%83%E9%99%90%E7%AE%A1%E7%90%86%EF%BC%9A%E5%A6%82%E4%BD%95%E5%89%96%E6%9E%90%20Spring%20Security%20%E7%9A%84%E6%8E%88%E6%9D%83%E5%8E%9F%E7%90%86%EF%BC%9F.md)

## 3.自定义认证授权

### 3.1 自定义认证方法

这种经常被用于生产实践中，实现起来也比较简单，分为两步：

```java
//第一步：实现AuthenticationProvider接口；
@Component
public class AuthenticationProviderBySessionId implements AuthenticationProvider {
    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String name = (String)authentication.getPrincipal();
        UserDetails user = userDetailsService.loadUserByUsername(name);
        return new UsernamePasswordAuthenticationToken(user.getUsername(), user.getPassword());
    }

    @Override
    public boolean supports(Class<?> authentication) {
        //填写对应的token验证
        return authentication.equals(UsernamePasswordAuthenticationToken.class);
    }
}
//第二步：服务启动的时候注入AuthenticationProvider接口的该实现类到配置中

  @Autowired
      private AuthenticationProvider authenticationProvider;
 @Override
    protected void configure(AuthenticationManagerBuilder builder) throws Exception {
        builder.authenticationProvider(authenticationProvider).userDetailsService(userService);
    }
```

### 3.2 自定义过滤器

自定义过滤器也分成两步：

```java
//第一步：实现Filter接口，并生成一个Bean对象
@Slf4j
@Component
public class PrintLogFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        log.info("Start to PrintLogFilter, Authenticate Successfully");
        filterChain.doFilter(servletRequest, servletResponse);
    }
}

//第二步：在配置类中加入这个Filter，
    @Autowired
    private PrintLogFilter printLogFilter;
    @Autowired
    private CustomTokenFilter customTokenFilter;

    protected void configure(HttpSecurity http) throws Exception {

        http.authorizeRequests().anyRequest().authenticated()
                .and().formLogin()
                .and().httpBasic();
        http.addFilterAfter(customTokenFilter, BasicAuthenticationFilter.class);
        http.addFilterAfter(printLogFilter, BasicAuthenticationFilter.class);
    }

addFilterAfter

addFilter

```

添加过滤器的方法：

+ addFilterAfter — 在指定的过滤器后面添加一个过滤器，通过将order + 1的方式实现；
+ addFilterBefore — 在指定的过滤器的前面添加一个过滤器，通过将order - 1的方式实现；
+ addFilter — 添加一个已经存在的过滤器；
+ addFilterAt — 和指定的过滤器的值order是一样的，但是指定的过滤器会先执行；

所有的过滤器按照order的值由小到大依次执行，当order的值相同时，按照list中的index顺序执行，换句话说，就是越早加载到list中越早执行。

### 3.3自定义Token

一般情况下，自定义token要配合自定义filter和自定义认证方法的使用。在生产实践中，使用SSO从第三方登录时或者是无服务之间互相建立通信时，都需要使用token进行认证。当token认证成功后，构建好上下文的数据存入缓存，这就是通常意义上的session，将缓存的key作为sessionId返回，下次发送请求就可以通过sessionId的方式来访问服务了。 上述描述的是两种认证，一种是第一次访问时没有session数据但是有token的访问，另一种是建立session后通过sessionId进行访问的方式。接下来我将会提供第一种的测试方式：

```java
自定义的Filter：

@Slf4j
public class CustomTokenFilter extends AbstractAuthenticationProcessingFilter {

    public CustomTokenFilter(String defaultFilterProcessesUrl) {
        super(defaultFilterProcessesUrl);
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException, IOException, ServletException {
        //String token = request.getHeader("header_token");

        String token = request.getParameter("jwt_token");

        System.out.println("=============" + request.getRequestURL());

        JWTAuthToken auth = new JWTAuthToken(token, null);
        Authentication result = getAuthenticationManager().authenticate(auth);
        //SecurityContextHolder.getContext().setAuthentication(result);
        return result;
    }

    //必须重写该方法
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain
            chain, Authentication authResult) throws IOException, ServletException {
        if (authResult.getPrincipal() != null) {
            SecurityContext context = SecurityContextHolder.createEmptyContext();
            context.setAuthentication(authResult);
            SecurityContextHolder.setContext(context);
        }
        chain.doFilter(request, response);
    }
}

在自定义的 serurity config中配置如下代码：
       public CustomTokenFilter customTokenFilter(){
        CustomTokenFilter customTokenFilter = new CustomTokenFilter("/**"); // /** 表示该filter过滤所有请求
        customTokenFilter.setAuthenticationManager(authenticationManager);
        return customTokenFilter;
    }


自定义的认证方法：

@Component
public class JWTAuthenticationProvider implements AuthenticationProvider {
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String token = (String) authentication.getCredentials();

        LoginUser user = JWTUtil.parseToken(token);
        return new JWTAuthToken(token, user);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return authentication.equals(JWTAuthToken.class);
    }
}

自定义的Token对象
public class JWTAuthToken extends PreAuthenticatedAuthenticationToken {

    private String jwtToken;

    private LoginUser loginUser;

    public JWTAuthToken(String jwtToken, LoginUser loginUser) {
        super(jwtToken, loginUser);
        this.jwtToken = jwtToken;
        this.loginUser = loginUser;
        Optional.ofNullable(loginUser).ifPresent( user ->  super.setAuthenticated(true));
    }

    @Override
    public Object getCredentials() {
        return this.jwtToken;
    }

    @Override
    public Object getPrincipal() {
        return this.loginUser;
    }
}
loginUser即为认证的主体，其主体也可以提供整个上下文数据，由于在认证成功后执行代码 SecurityContextHolder.setContext(context),这可以使我们在认证后的任何环节都可以通过SecurityContextHolder.getContext().getAuthentication()获取到上下文数据(异步不可取)。
```

上述是实现了整个自定义Token的认证方式主题，分别介绍一下自定义部分的作用：

+ 自定义Filter：所有需要认证的请求都会经过该filter，那么我们可以根据请求的路径做很好的规划处理。我们可以通过请求的header是否有sessionId然后决定使用session数据的认证方式，可以通过获取请求的参数或者header里面是否存在token然后决定做Token认证处理。换而言之，filter决定使用哪一种认证方式；
+ 自定义认证方式：这是实现具体的认证过程，其中包含构建pricipal，也就是上下文数据；
+ 自定义Token：该类通过supports(Class<?> authentication)方法来绑定一种认证方式，在filter中主要通过传入对应token来执行相应的认证方式。

<font color = red>踩坑记录：</font>

如果在filter中没有重写`successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult)`方法，这会导致授权成功后会再次发送一次请求`localhost:8080/`, 而这次请求是没有带token的这就会导致认证出错，所以需要实现。

<font color=blue>1. 如何访问API：</font>

1. 找到`JWTUtil`后运行main方法，将输出的token赋值好；
2. 启动服务；
3. 将路径中的jwt_token=后的替换成刚刚复制的token，执行即可，localhost:8080/ready/security/post?jwt_token={{token}}。

<font color=blue> 2.如何实现自定义注解获取上下文数据：</font>

```java
    @PostMapping("/ready/security/post")
    public String testPost(@CurrentUser LoginUser loginUser){
        System.out.println(loginUser);
        return "Test Post";
    }
```

在上述的API接口中，当请求成功访问后会打印当前的用户上下文的。实现方式如下：

```java
//自定义一个注解，加上@AuthenticationPrincipal就好了。

@Target({ElementType.PARAMETER, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@AuthenticationPrincipal
public @interface CurrentUser {
}
```

直接在方法参数上使用@AuthenticationPrincipal也是能获取上下文数据的，但是没有语义比较难理解，所以可以自定义一个和业务相关的注解的方式。



## 4.总结

该篇基本是从系统密码登陆到自定义Token进行认证的实践过程，尤其是第三章节基本的方案是可以完全用于生产实践中。对于spring security的学习，踩到了很多坑，比如版本不一样的坑，最后通过debug一步一步查出问题。出现问题不要慌，要有耐心，不要出现报错后就手足无措。






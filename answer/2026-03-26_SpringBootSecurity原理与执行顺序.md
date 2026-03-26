# Spring Boot Security 原理及执行顺序

## 一、核心架构概览

Spring Security 采用**过滤器链（Filter Chain）**模式，通过一系列过滤器对请求进行拦截、认证、授权处理。

```
HTTP Request
    ↓
SecurityFilterChain (过滤器链)
    ├─ TokenAuthenticationFilter (Token 认证)
    ├─ ExceptionTranslationFilter (异常处理)
    ├─ FilterSecurityInterceptor (权限校验)
    └─ ...其他过滤器
    ↓
Controller / Handler
    ↓
HTTP Response
```

---

## 二、zrk-cloud 项目的 Security 实现

### 2.1 核心组件

| 组件 | 职责 | 位置 |
|------|------|------|
| **TokenAuthenticationFilter** | Token 提取、验证、用户信息构建 | `core/filter/` |
| **ZrkWebSecurityConfigurerAdapter** | Security 配置、URL 权限映射 | `config/` |
| **ZrkSecurityAutoConfiguration** | Bean 自动配置 | `config/` |
| **SecurityFrameworkUtils** | 上下文工具、用户信息获取 | `core/util/` |
| **AuthenticationEntryPointImpl** | 未认证异常处理 | `core/handler/` |
| **AccessDeniedHandlerImpl** | 权限不足异常处理 | `core/handler/` |
| **LoginUser** | 登录用户信息模型 | `core/` |

---

## 三、请求执行流程（详细步骤）

### 3.1 请求到达时的处理流程

```
1. HTTP 请求进入
   ↓
2. SecurityFilterChain 开始处理
   ├─ 2.1 TokenAuthenticationFilter.doFilterInternal()
   │   ├─ 2.1.1 尝试从 Header[login-user] 获取用户（Gateway 透传）
   │   ├─ 2.1.2 若无，从 Authorization Header 或 Parameter 提取 Token
   │   ├─ 2.1.3 调用 OAuth2TokenCommonApi.checkAccessToken() 验证 Token
   │   ├─ 2.1.4 构建 LoginUser 对象
   │   ├─ 2.1.5 调用 SecurityFrameworkUtils.setLoginUser() 设置上下文
   │   └─ 2.1.6 继续过滤链 chain.doFilter()
   │
   ├─ 2.2 ExceptionTranslationFilter
   │   └─ 捕获认证/授权异常
   │
   ├─ 2.3 FilterSecurityInterceptor
   │   ├─ 检查 URL 是否需要认证
   │   ├─ 若需要，检查用户是否已认证
   │   ├─ 若未认证，触发 AuthenticationEntryPoint
   │   ├─ 若已认证，检查权限
   │   └─ 若权限不足，触发 AccessDeniedHandler
   │
   └─ 2.4 其他过滤器...
   ↓
3. 到达 Controller
   ↓
4. 返回响应
```

---

## 四、核心执行顺序详解

### 4.1 TokenAuthenticationFilter 执行流程

**文件**: `TokenAuthenticationFilter.java`

```java
protected void doFilterInternal(HttpServletRequest request,
                                HttpServletResponse response,
                                FilterChain chain) {
    // 步骤 1: 尝试从 Header 获取用户（来自 Gateway 或其他服务透传）
    LoginUser loginUser = buildLoginUserByHeader(request);

    // 步骤 2: 若无，从 Token 获取用户
    if (loginUser == null) {
        String token = SecurityFrameworkUtils.obtainAuthorization(request, ...);
        if (token != null) {
            Integer userType = WebFrameworkUtils.getLoginUserType(request);
            try {
                // 步骤 2.1: 验证 Token 有效性
                loginUser = buildLoginUserByToken(token, userType);

                // 步骤 2.2: 若 Token 无效，尝试模拟登录（开发调试用）
                if (loginUser == null) {
                    loginUser = mockLoginUser(request, token, userType);
                }
            } catch (Throwable ex) {
                // 异常处理
                CommonResult<?> result = globalExceptionHandler.allExceptionHandler(request, ex);
                ServletUtils.writeJSON(response, result);
                return;
            }
        }
    }

    // 步骤 3: 设置当前用户到 Spring Security 上下文
    if (loginUser != null) {
        SecurityFrameworkUtils.setLoginUser(loginUser, request);
    }

    // 步骤 4: 继续过滤链
    chain.doFilter(request, response);
}
```

### 4.2 Token 验证流程

```java
private LoginUser buildLoginUserByToken(String token, Integer userType) {
    try {
        // 1. 调用 OAuth2 服务验证 Token
        OAuth2AccessTokenCheckRespDTO accessToken =
            oauth2TokenApi.checkAccessToken(token).getCheckedData();

        if (accessToken == null) {
            return null;  // Token 无效
        }

        // 2. 检查用户类型是否匹配
        if (userType != null &&
            !accessToken.getUserType().equals(userType)) {
            throw new AccessDeniedException("错误的用户类型");
        }

        // 3. 构建 LoginUser 对象
        return new LoginUser()
            .setId(accessToken.getUserId())
            .setUserType(accessToken.getUserType())
            .setInfo(accessToken.getUserInfo())
            .setTenantId(accessToken.getTenantId())
            .setScopes(accessToken.getScopes())
            .setExpiresTime(accessToken.getExpiresTime());

    } catch (ServiceException ex) {
        // Token 校验失败，返回 null（允许无登录访问）
        return null;
    }
}
```

### 4.3 用户上下文设置流程

**文件**: `SecurityFrameworkUtils.java`

```java
public static void setLoginUser(LoginUser loginUser, HttpServletRequest request) {
    // 步骤 1: 创建 Authentication 对象
    Authentication authentication = buildAuthentication(loginUser, request);

    // 步骤 2: 设置到 SecurityContextHolder（ThreadLocal 存储）
    SecurityContextHolder.getContext().setAuthentication(authentication);

    // 步骤 3: 额外设置到 request 中（用于日志记录）
    if (request != null) {
        WebFrameworkUtils.setLoginUserId(request, loginUser.getId());
        WebFrameworkUtils.setLoginUserType(request, loginUser.getUserType());
    }
}

private static Authentication buildAuthentication(LoginUser loginUser,
                                                   HttpServletRequest request) {
    // 创建 UsernamePasswordAuthenticationToken
    UsernamePasswordAuthenticationToken authenticationToken =
        new UsernamePasswordAuthenticationToken(
            loginUser,           // principal（用户信息）
            null,                // credentials（密码，这里为 null）
            Collections.emptyList()  // authorities（权限，这里为空）
        );

    // 设置请求详情
    authenticationToken.setDetails(
        new WebAuthenticationDetailsSource().buildDetails(request)
    );

    return authenticationToken;
}
```

---

## 五、Security 配置流程

### 5.1 自动配置类初始化顺序

**文件**: `ZrkSecurityAutoConfiguration.java`

```java
@AutoConfiguration
@AutoConfigureOrder(-1)  // 优先级最高，先于 Spring Security 自动配置
@EnableConfigurationProperties(SecurityProperties.class)
public class ZrkSecurityAutoConfiguration {

    // 1. 创建认证失败处理器
    @Bean
    public AuthenticationEntryPoint authenticationEntryPoint() {
        return new AuthenticationEntryPointImpl();
    }

    // 2. 创建权限不足处理器
    @Bean
    public AccessDeniedHandler accessDeniedHandler() {
        return new AccessDeniedHandlerImpl();
    }

    // 3. 创建密码编码器
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(securityProperties.getPasswordEncoderLength());
    }

    // 4. 创建 Token 认证过滤器
    @Bean
    public TokenAuthenticationFilter authenticationTokenFilter(...) {
        return new TokenAuthenticationFilter(securityProperties,
                                            globalExceptionHandler,
                                            oauth2TokenApi);
    }

    // 5. 设置 SecurityContextHolder 策略为 TransmittableThreadLocal
    @Bean
    public MethodInvokingFactoryBean securityContextHolderMethodInvokingFactoryBean() {
        MethodInvokingFactoryBean bean = new MethodInvokingFactoryBean();
        bean.setTargetClass(SecurityContextHolder.class);
        bean.setTargetMethod("setStrategyName");
        bean.setArguments(
            TransmittableThreadLocalSecurityContextHolderStrategy.class.getName()
        );
        return bean;
    }
}
```

### 5.2 Web Security 配置

**文件**: `ZrkWebSecurityConfigurerAdapter.java`

```java
@Bean
protected SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
    httpSecurity
        // 1. 启用 CORS
        .cors(Customizer.withDefaults())

        // 2. 禁用 CSRF（因为使用 Token 认证）
        .csrf(AbstractHttpConfigurer::disable)

        // 3. 禁用 Session（无状态认证）
        .sessionManagement(session ->
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        )

        // 4. 配置异常处理
        .exceptionHandling(exception ->
            exception
                .authenticationEntryPoint(authenticationEntryPoint)
                .accessDeniedHandler(accessDeniedHandler)
        )

        // 5. 配置 URL 权限映射
        .authorizeHttpRequests(authorize -> {
            // 5.1 获取所有 @PermitAll 标注的 URL
            Multimap<HttpMethod, String> permitAllUrls =
                getPermitAllUrls();

            // 5.2 配置免登录 URL
            permitAllUrls.forEach((method, url) -> {
                authorize.requestMatchers(method, url).permitAll();
            });

            // 5.3 其他 URL 需要认证
            authorize.anyRequest().authenticated();
        })

        // 6. 添加 Token 认证过滤器
        .addFilterBefore(authenticationTokenFilter,
                        UsernamePasswordAuthenticationFilter.class);

    return httpSecurity.build();
}
```

---

## 六、异常处理流程

### 6.1 未认证异常处理

**文件**: `AuthenticationEntryPointImpl.java`

```java
@Override
public void commence(HttpServletRequest request,
                     HttpServletResponse response,
                     AuthenticationException e) {
    log.debug("[commence][访问 URL({}) 时，没有登录]",
              request.getRequestURI(), e);

    // 返回 401 Unauthorized
    ServletUtils.writeJSON(response,
        CommonResult.error(GlobalErrorCodeConstants.UNAUTHORIZED)
    );
}
```

### 6.2 权限不足异常处理

**文件**: `AccessDeniedHandlerImpl.java`

```java
@Override
public void handle(HttpServletRequest request,
                   HttpServletResponse response,
                   AccessDeniedException e) {
    log.debug("[handle][访问 URL({}) 时，权限不足]",
              request.getRequestURI(), e);

    // 返回 403 Forbidden
    ServletUtils.writeJSON(response,
        CommonResult.error(GlobalErrorCodeConstants.FORBIDDEN)
    );
}
```

---

## 七、关键设计点

### 7.1 三层用户获取机制

```
优先级 1: Header[login-user]（来自 Gateway 透传）
    ↓ 若无
优先级 2: Authorization Header / Parameter 中的 Token
    ↓ 若无
优先级 3: 模拟登录（开发调试，生产禁用）
```

### 7.2 用户类型校验

```java
// 检查用户类型是否匹配请求路径
if (userType != null &&
    !accessToken.getUserType().equals(userType)) {
    throw new AccessDeniedException("错误的用户类型");
}
```

- `/admin-api/*` → 需要 Admin 用户类型
- `/app-api/*` → 需要 App 用户类型
- `/ws/*` → 不需要检查用户类型

### 7.3 TransmittableThreadLocal 策略

```java
// 使用 TransmittableThreadLocal 而非普通 ThreadLocal
SecurityContextHolder.setStrategyName(
    TransmittableThreadLocalSecurityContextHolderStrategy.class.getName()
);
```

**优势**: 在异步任务、线程池中能正确传递用户上下文

### 7.4 免登录 URL 自动扫描

```java
// 扫描所有 @PermitAll 标注的方法
private Multimap<HttpMethod, String> getPermitAllUrls() {
    RequestMappingHandlerMapping mapping =
        applicationContext.getBean(RequestMappingHandlerMapping.class);

    Multimap<HttpMethod, String> result = HashMultimap.create();

    mapping.getHandlerMethods().forEach((info, method) -> {
        // 检查是否有 @PermitAll 注解
        if (method.hasMethodAnnotation(PermitAll.class)) {
            // 添加到免登录列表
        }
    });

    return result;
}
```

---

## 八、完整请求生命周期

```
1. 请求到达 Servlet 容器
   ↓
2. DelegatingFilterProxy 转发到 FilterChainProxy
   ↓
3. SecurityFilterChain 开始处理
   ├─ TokenAuthenticationFilter
   │  ├─ 提取 Token
   │  ├─ 验证 Token（调用 OAuth2 服务）
   │  ├─ 构建 LoginUser
   │  └─ 设置到 SecurityContextHolder
   │
   ├─ ExceptionTranslationFilter
   │  └─ 捕获异常
   │
   ├─ FilterSecurityInterceptor
   │  ├─ 检查 URL 是否需要认证
   │  ├─ 检查用户是否已认证
   │  ├─ 检查用户权限
   │  └─ 触发异常处理器（如需要）
   │
   └─ 其他过滤器...
   ↓
4. DispatcherServlet 处理请求
   ├─ 查找 Handler
   ├─ 执行 Interceptor.preHandle()
   ├─ 执行 Controller 方法
   │  └─ 可通过 SecurityFrameworkUtils.getLoginUser() 获取用户
   ├─ 执行 Interceptor.postHandle()
   └─ 返回 ModelAndView
   ↓
5. 视图渲染 / 响应序列化
   ↓
6. 返回 HTTP 响应
```

---

## 九、常用工具方法

```java
// 获取当前登录用户
LoginUser user = SecurityFrameworkUtils.getLoginUser();

// 获取当前用户 ID
Long userId = SecurityFrameworkUtils.getLoginUserId();

// 获取当前用户昵称
String nickname = SecurityFrameworkUtils.getLoginUserNickname();

// 获取当前用户部门 ID
Long deptId = SecurityFrameworkUtils.getLoginUserDeptId();

// 获取当前认证信息
Authentication auth = SecurityFrameworkUtils.getAuthentication();

// 从请求中提取 Token
String token = SecurityFrameworkUtils.obtainAuthorization(request,
                                                          headerName,
                                                          parameterName);
```

---

## 十、总结

| 阶段 | 关键组件 | 职责 |
|------|---------|------|
| **请求进入** | TokenAuthenticationFilter | 提取 Token，验证有效性 |
| **用户构建** | OAuth2TokenCommonApi | 验证 Token，返回用户信息 |
| **上下文设置** | SecurityFrameworkUtils | 构建 Authentication，存入 ThreadLocal |
| **权限检查** | FilterSecurityInterceptor | 检查 URL 权限要求 |
| **异常处理** | AuthenticationEntryPoint / AccessDeniedHandler | 处理认证/授权异常 |
| **业务处理** | Controller | 通过工具类获取用户信息 |

**核心流程**: Token 提取 → Token 验证 → 用户构建 → 上下文设置 → 权限检查 → 业务处理

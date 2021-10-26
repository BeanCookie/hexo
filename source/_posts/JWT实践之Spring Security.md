---
title: JWT实践之Spring Security
date: 2020-07-21 10:44:26
categories:
- 认证
tags:
- JWT
- Spring
- Spring Security
---

#### Spring Security概述

**Spring Security**的核心原理就是运用Filters在请求之前对请求进行验证和过滤从而保证资源安全

![https://beancookie.github.io/images/JWT实践之SpringSecurity-01.png](https://beancookie.github.io/images/JWT实践之SpringSecurity-01.png)

#### JWT核心配置

1. 最主要的配置就是需要实现一个继承自**WebSecurityConfigurerAdapter**的**Bean**

```java
@EnableGlobalMethodSecurity(prePostEnabled = true, jsr250Enabled = true)
@EnableWebSecurity
public class SpringSecurityConfig extends WebSecurityConfigurerAdapter {


    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                // 忽略所有OPTIONS类型的请求
                .antMatchers(HttpMethod.OPTIONS).permitAll()
                // 忽略登录请求
                .antMatchers("/api/auth/login", "/api/auth/refresh").permitAll()
                // 拦截剩余的所有请求
                .anyRequest().authenticated()
                // 使用自定义的 accessDecisionManager
                .accessDecisionManager(accessDecisionManager())
                .and()
                .exceptionHandling()
                .accessDeniedHandler(restfulAccessDeniedHandler())
                .authenticationEntryPoint(restAuthenticationEntryPoint())
                .and()
                .addFilterBefore(jwtTokenFilter(), UsernamePasswordAuthenticationFilter.class)
                .cors()
                .and()
                .csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
    }

    @Bean
    public JwtTokenFilter jwtTokenFilter() {
        return new JwtTokenFilter();
    }

    @Bean
    public RestfulAccessDeniedHandler restfulAccessDeniedHandler() {
        return new RestfulAccessDeniedHandler();
    }

    @Bean
    public RestAuthenticationEntryPoint restAuthenticationEntryPoint() {
        return new RestAuthenticationEntryPoint();
    }

    @Override
    @Bean
    public AuthenticationManager authenticationManager() throws Exception {
        return super.authenticationManager();
    }

    /**
     * 指定bcrypt算法进行密码加密
     * @return
     */
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    /**
     * 自定义permission投票器
     * @return
     */
    @Bean
    public AccessDecisionVoter<FilterInvocation> accessDecisionProcessor() {
        return new AccessDecisionProcessor();
    }

    @Bean
    public AccessDecisionManager accessDecisionManager() {
        // 构造一个新的AccessDecisionManager 放入两个投票器
        List<AccessDecisionVoter<?>> decisionVoters = Arrays.asList(new WebExpressionVoter(), accessDecisionProcessor());
        return new UnanimousBased(decisionVoters);
    }
}
```

2. 在Spring Security中使用**JWT**非常简单只需要自定义一个**JWT Filter**就可以了
3. 在定义**Filter**之前还需要创建一个**JWT**的解析器

> 使用 jjwt 进行解析

```groovy
compile 'io.jsonwebtoken:jjwt-impl'
compile 'io.jsonwebtoken:jjwt-jackson'

compile 'cn.hutool:hutool-all'
```

```java
@Slf4j
@Component
public class JwtProvider {
    private static final Key KEY = Keys.secretKeyFor(SignatureAlgorithm.HS512);

    @Resource
    private JwtProperties jwtProperties;

    /**
     * 从request中获取token
     * @param request
     * @return
     */
    public Optional<String> getTokenFromRequest(HttpServletRequest request) {
        String authorization = request.getHeader(jwtProperties.getRequestHeader());
        if (StringUtils.isNotEmpty(authorization)) {
            return Optional.of(authorization.substring(jwtProperties.getTokenPrefix().length()));
        }
        return Optional.empty();
    }

    /**
     * 从token中拿到负载信息
     */
    private Claims getClaimsFromToken(String token) {
        return Jwts.parserBuilder()
                .deserializeJsonWith(new JacksonDeserializer<>())
                .base64UrlDecodeWith(Decoders.BASE64)
                .setSigningKey(KEY)
                .build()
                .parseClaimsJws(token)
                .getBody();
    }

    /**
     * 从token中获取subject
     * @param token
     * @return
     */
    public Optional<String> getSubjectFromToken(String token) {
        if (StringUtils.isNotEmpty(token)) {
            Claims claims = this.getClaimsFromToken(token);
            if (Objects.nonNull(claims)) {
                return Optional.ofNullable(claims.getSubject());
            }
        }
        return Optional.empty();
    }

    /**
     * 根据UserDetail生成jwt，并封装成AccessToken
     * @param userDetail
     * @return
     */
    public AccessToken createToken(UserDetails userDetail) {
        return this.createToken(userDetail.getUsername());
    }

    /**
     * 根据subject生成jwt，并封装成AccessToken
     * @param subject
     * @return
     */
    public AccessToken createToken(String subject) {
        // 当前时间
        final Date now = new Date();
        // 过期时间
        final Date expirationDate = new Date(now.getTime() + jwtProperties.getExpirationTime() * 1000);
        String token = Jwts.builder()
                .serializeToJsonWith(new JacksonSerializer<>())
                .base64UrlEncodeWith(Encoders.BASE64)
                .setSubject(subject)
                .setIssuedAt(now)
                .setExpiration(expirationDate)
                .signWith(KEY)
                .compact();

        return AccessToken.builder()
                .loginAccount(subject)
                .token(token)
                .expirationTime(expirationDate)
                .build();
    }

    /**
     * 验证token是否有效
     * @param token
     * @param userDetails
     * @return
     */
    public boolean validateToken(String token, UserDetails userDetails) {
        Claims claims = this.getClaimsFromToken(token);
        return claims.getSubject().equals(userDetails.getUsername()) && !this.isTokenExpired(claims);
    }

    /**
     * 判断jwt时效性
     * @param claims
     * @return
     */
    public boolean isTokenExpired(Claims claims) {
        return claims.getIssuedAt().before(new Date());
    }

    /**
     * 判断token在指定时间内是否刚刚刷新过
     */
    private boolean tokenRefreshJustBefore(Claims claims) {
        Date refreshDate = new Date();
        //刷新时间在创建时间的指定时间内
        if (refreshDate.after(claims.getExpiration()) && refreshDate.before(DateUtil.offsetSecond(claims.getExpiration(), jwtProperties.getExpirationTime()))) {
            return true;
        }
        return false;
    }

    public AccessToken refreshToken(String token) {
        Claims claims = this.getClaimsFromToken(token);
        //如果token在30分钟之内刚刷新过，返回原token
        if (tokenRefreshJustBefore(claims)) {
            return AccessToken.builder()
                    .loginAccount(claims.getSubject())
                    .token(token)
                    .expirationTime(claims.getExpiration()).build();
        } else {
            return createToken(claims.getSubject());
        }
    }
}
```

> 通过定义定义OncePerRequestFilter拦截器可以实现在每个请求之前检查Request Header中的JWT token

```java
public class JwtTokenFilter extends OncePerRequestFilter {
    @Resource
    private JwtProvider jwtProvider;
    @Resource
    private AuthService authService;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        Optional<String> authToken = jwtProvider.getTokenFromRequest(request);
        authToken.ifPresent(token -> {
            Optional<String> subject = jwtProvider.getSubjectFromToken(token);
            if (subject.isPresent()
                && Objects.isNull(SecurityContextHolder.getContext().getAuthentication())
            ) {
                UserDetail userDetail = authService.login(subject.get(), null);
                if (Objects.nonNull(userDetail) && jwtProvider.validateToken(token, userDetail)) {
                    // 后面的拦截器里面会用到 grantedAuthorities 方法
                    UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userDetail, userDetail.getPassword(), userDetail.getAuthorities());

                    // 将authentication信息放入到上下文对象中
                    SecurityContextHolder.getContext().setAuthentication(authentication);

                }
            }
        });
        filterChain.doFilter(request, response);
    }
}
```

3. 最后再定义Rest接口对外提供登陆/登出和刷新Token的接口

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {
    @Resource
    private AuthService authService;

    @Resource
    private JwtProvider jwtProvider;

    @PostMapping("/login")
    public R<AccessToken> login(@Valid @RequestBody LoginQuery loginQuery) {
        UserDetail userDetail = authService.login(loginQuery.getLoginAccount(), loginQuery.getPassword());
        AccessToken accessToken = jwtProvider.createToken(userDetail);
        return R.success(accessToken);
    }

    @PostMapping("/logout")
    public R<?> logout() {
        authService.logout();
        return R.success();
    }

    @PostMapping("/refresh")
    public R<AccessToken> refresh(HttpServletRequest request) {
        AccessToken accessToken = authService.refresh(jwtProvider.getTokenFromRequest(request).orElseThrow(AuthorizationException::new));
        return R.success(accessToken);
    }
}
```

#### 运用投票器实现动态鉴权

Spring Security使用投票器进行鉴权，我们可以定义多个投票器对每个请求进行鉴定，每个投票器可以返回一票**反对** **通过** **弃权**，最后通过统计票数判断请求是否具有权限。再定义投票器之前需要定义权限表并与角色进行关联，权限表中需要包含请求的URL和Http方法。

```java
public class AccessDecisionProcessor implements AccessDecisionVoter<FilterInvocation> {
    @Resource
    private Cache<String, PermissionBO> permissionCache;

    @Override
    public boolean supports(ConfigAttribute attribute) {
        return true;
    }

    @Override
    public boolean supports(Class<?> clazz) {
        return true;
    }

    @Override
    public int vote(Authentication authentication, FilterInvocation invocation, Collection<ConfigAttribute> attributes) {
        String url = invocation.getRequestUrl();
        String method = invocation.getRequest().getMethod();
        String cacheKey = String.format("%s:%s", url, method);
        PermissionBO permissionBO = permissionCache.get(cacheKey);
        if (Objects.isNull(permissionBO)) {
            return ACCESS_ABSTAIN;
        }
        UserDetail userDetail = (UserDetail) authentication.getPrincipal();

        if (Objects.nonNull(userDetail) && userDetail.getRoles().contains(permissionBO.getRoleCode())) {
            return ACCESS_GRANTED;
        }

        return ACCESS_DENIED;
    }
}
```

```java
@EnableGlobalMethodSecurity(prePostEnabled = true, jsr250Enabled = true)
@EnableWebSecurity
public class SpringSecurityConfig extends WebSecurityConfigurerAdapter {
    /**
     * 自定义permission投票器
     * @return
     */
    @Bean
    public AccessDecisionVoter<FilterInvocation> accessDecisionProcessor() {
        return new AccessDecisionProcessor();
    }

    @Bean
    public AccessDecisionManager accessDecisionManager() {
        // 构造一个新的AccessDecisionManager 放入两个投票器
        List<AccessDecisionVoter<?>> decisionVoters = Arrays.asList(new WebExpressionVoter(), accessDecisionProcessor());
        return new UnanimousBased(decisionVoters);
    }
}
```


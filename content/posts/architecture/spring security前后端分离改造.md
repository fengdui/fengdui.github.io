---
title: "spring security前后端分离改造"
date: "2019-01-06"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
原先是spring security+jsp 现在要做前后端分离项目 前端是vue 后端是spring boot

Spring Security 默认是为传统的Web应用（如JSP、Thymeleaf等）设计的，
其中后端通常负责生成HTML视图。在前后端分离的架构中，后端仅提供API接口，
前端通过Ajax请求与后端交互。因此，我们需要对Spring Security进行一些配置调整，以支持前后端分离

```
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {
    
    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())  // 前后端分离，禁用 CSRF
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)  // 无状态
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()  // 公开接口
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .anyRequest().authenticated()  // 其他接口需要认证
            )
            .addFilterBefore(jwtAuthenticationFilter, 
                           UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)
                .accessDeniedHandler(jwtAccessDeniedHandler)
            );
        
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
``` 

实习filter 用于处理JWT认证
```
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private final JwtUtil jwtUtil;
    private final UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {
        
        final String authHeader = request.getHeader("Authorization");
        final String jwt;
        final String username;
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }
        
        jwt = authHeader.substring(7);
        username = jwtUtil.extractUsername(jwt);
        
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);
            
            if (jwtUtil.validateToken(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken = 
                    new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,
                        userDetails.getAuthorities()
                    );
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        filterChain.doFilter(request, response);
    }
}
```
实现AuthenticationEntryPoint 用于处理未认证的请求
```
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {
    
    @Override
    public void commence(HttpServletRequest request,
                        HttpServletResponse response,
                        AuthenticationException authException) 
                        throws IOException {
        
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        
        response.getWriter().write(JSON.toJSONString(
            ApiResponse.error(401, "未认证，请先登录")
        ));
    }
}

@Component
public class JwtAccessDeniedHandler implements AccessDeniedHandler {
    
    @Override
    public void handle(HttpServletRequest request,
                      HttpServletResponse response,
                      AccessDeniedException accessDeniedException) 
                      throws IOException {
        
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        
        response.getWriter().write(JSON.toJSONString(
            ApiResponse.error(403, "权限不足")
        ));
    }
}
```
spring security xml 配置
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:security="http://www.springframework.org/schema/security"
       xsi:schemaLocation="
          http://www.springframework.org/schema/beans
          http://www.springframework.org/schema/beans/spring-beans.xsd
          http://www.springframework.org/schema/security
          http://www.springframework.org/schema/security/spring-security.xsd">

    <!-- 1. 配置HTTP安全规则 (核心) -->
    <!-- 关键属性说明：
         security="none"：完全不经过Spring Security过滤器链，用于完全公开的资源（如登录接口本身）。
         create-session="stateless"：不创建HttpSession，实现无状态。
         disable-url-rewriting="true"：防止在URL中重写Session ID。
         entry-point-ref：定义认证入口点（处理未认证请求）。
    -->
    <security:http pattern="/api/auth/login" security="none"/>
    <security:http create-session="stateless" entry-point-ref="jwtAuthenticationEntryPoint"
                    disable-url-rewriting="true">

        <!-- 1.1 自定义过滤器链：在标准的身份验证过滤器前插入JWT过滤器 -->
        <security:custom-filter ref="jwtAuthenticationFilter" position="PRE_AUTH_FILTER"/>

        <!-- 1.2 配置URL访问规则 -->
        <security:intercept-url pattern="/api/public/**" access="permitAll" />
        <security:intercept-url pattern="/api/admin/**" access="hasRole('ADMIN')" />
        <!-- 其他所有API请求都需要认证 -->
        <security:intercept-url pattern="/api/**" access="isAuthenticated()" />

        <!-- 1.3 禁用CSRF防护（适用于无状态的API） -->
        <security:csrf disabled="true"/>

        <!-- 1.4 配置退出登录处理（可选，在无状态下通常由前端丢弃Token实现） -->
        <security:logout logout-url="/api/auth/logout" logout-success-url="/" delete-cookies="JSESSIONID"/>
    </security:http>

    <!-- 2. 配置认证管理器 -->
    <!-- 这里通常引用一个自定义的UserDetailsService实现（从数据库加载用户） -->
    <security:authentication-manager alias="authenticationManager">
        <security:authentication-provider user-service-ref="customUserDetailsService">
            <!-- 配置密码编码器 -->
            <security:password-encoder ref="passwordEncoder"/>
        </security:authentication-provider>
    </security:authentication-manager>

    <!-- 3. 声明方案一所需的自定义Bean (关键难点) -->
    <!-- 注意：这些Bean的实现类（JwtUtil, JwtAuthenticationFilter等）需要你另外用Java编写 -->
    <!-- 然后将它们通过@Component或@Bean注解交由Spring管理，或在此处用<bean>标签定义 -->
    <bean id="passwordEncoder" class="org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder"/>
    
    <!-- JWT工具类 -->
    <bean id="jwtUtil" class="com.your.security.JwtUtil"/>
    
    <!-- 自定义JWT认证过滤器 -->
    <bean id="jwtAuthenticationFilter" class="com.your.security.JwtAuthenticationFilter">
        <property name="jwtUtil" ref="jwtUtil"/>
        <property name="userDetailsService" ref="customUserDetailsService"/>
    </bean>
    
    <!-- 认证入口点（返回401 JSON响应） -->
    <bean id="jwtAuthenticationEntryPoint" class="com.your.security.JwtAuthenticationEntryPoint"/>
    
    <!-- 自定义的UserDetailsService -->
    <bean id="customUserDetailsService" class="com.your.service.CustomUserDetailsServiceImpl"/>
</beans>


<filter>
    <filter-name>springSecurityFilterChain</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>
<filter-mapping>
    <filter-name>springSecurityFilterChain</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/applicationContext-security.xml</param-value>
</context-param>
```
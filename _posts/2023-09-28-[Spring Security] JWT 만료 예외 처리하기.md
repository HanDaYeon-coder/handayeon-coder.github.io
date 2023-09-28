---
title: "[Spring Security] JWT 만료 예외 처리하기"
date: 2023-09-28
categories: [Spring Security, JWT]
---

# 0️⃣ 서론

이전에 Spring Security를 사용하여 JWT를 구현하였습니다.

https://velog.io/@da_na/Spring-Security-OAuth2-로그인-JWT-구현하기

따라서 JWT를 사용하여 인증과 인가가 성공적으로 구현되었고, 그대로 프로젝트 개발을 진행하였습니다.

그러다가,,, JWT 만료된 시간이 다가왔고, 만료된 JWT를 사용하여 API를 요청했을 때 아래와 같이 500 Internal Server Error가 나왔습니다. 🥲

![Alt text](/assets/img/2023-09-28/image-1.png)

따라서 클라이언트쪽에서는 위의 에러가 어떠한 에러인지 자세하게 알 수 없기 때문에 다시 JWT를 발급하기 위해서 로그인으로 돌아가는 과정을 추가할 수 없었습니다.

즉, JWT 예외가 발생하는 경우에 이전에 유효성 검사와 같이 에러 응답과 동일한 형태로 아래의 사진 형태처럼 resultCode와 message를 전달해야했습니다.

![Alt text](/assets/img/2023-09-28/image-3.png)

하지만 유효성 검사의 에러 응답 같은 경우에는 @RestControllerAdvice에서 처리가 가능했지만, 이번에 발생한 JWT 만료 에러의 경우에는 Spring Security 필터에서 발생한 에러이기 때문에 Controller 단까지 가지 않기 때문에 Filter 자체에서 에러를 처리하는 로직이 추가되어야만 했습니다.

이전에 작성한 JWT 처리하는 로직을 천천히 다시 살펴보면서 어떤 부분을 추가해야 할지를 생각해보겠습니다! 그리고 이 과정에서 발생한 문제점과 해결책을 같이 이야기하겠습니다.

---

# 1️⃣ 본론

## 1. JWT 만료 예외 메시지를 살펴보자.

위의 JWT 만료시에 스프링 부트 내에서 발생한 오류 기록을 보면, **ExpiredJwtException**이 발생한 것을 알 수 있습니다.

![Alt text](/assets/img/2023-09-28/image-4.png)

- 즉, JwtAuthFilter에서 Jwt 토큰의 만료기한을 확인하는 validateToken에서 만료기한이 넘으면 ExpiredJwtException 예외가 발생함을 알 수 있습니다.

---

## 2. 잘못된 JWT 만료 처리 추가

따라서 Spring Security의 Filter에서 예외가 발생할 경우, 예외를 처리해줄 부분을 추가해주어야 했습니다.

JWT 만료 처리를 하는 방법이 아래와 같이 2가지로 많이 있어서, 2가지를 추가해주었습니다.

`.exceptionHandling().authenticationEntryPoint(new JwtAuthenticationEntryPoint())`와 `.addFilterBefore(new JwtExceptionFilter(), JwtAuthFilter.class)` 중에서 addFilterBefore를 사용하여 SecurityFilterChain의 filterChain에 추가하였습니다.

### addFilterBefore(new JwtExceptionFilter(), JwtAuthFilter.class)
- JwtAuthFilter에서 발생한 예외를 JwtExceptionFilter에서 잡아내고, ExpiredJwtException 예외에 대한 request.setAttribute("exception", e.getMessage());를 설정하여 JwtAuthenticationEntryPoint에서 commence에서 에러 응답을 클라이언트에 보내도록 하였습니다.

```java
@RequiredArgsConstructor
@EnableWebSecurity
public class SecurityConfig {

    private final CustomOAuth2UserService customOAuth2UserService;
    private final TokenService tokenService;
    private final OAuth2SuccessHandler oAuth2SuccessHandler;
    private final OAuth2FailureHandler oAuth2FailureHandler;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

        http
                .csrf().disable()
                .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .cors().configurationSource(corsConfigurationSource())
                .and()
                .headers().frameOptions().disable()
                .and()
                .authorizeRequests()
                .antMatchers("/h2-console/**", "/actuator/**",
                        "/", "/api-docs/**", "/swagger-ui/**").permitAll()
                .antMatchers("/v1/**", "/login/**").hasRole(Role.USER.name())
                .anyRequest().authenticated()
                .and()
                .logout()
                .logoutSuccessUrl("/")
                .and()
                .addFilterBefore(new JwtAuthFilter(tokenService),
                        UsernamePasswordAuthenticationFilter.class)
                .addFilterBefore(new JwtExceptionFilter(), JwtAuthFilter.class)
                .oauth2Login()
                .successHandler(oAuth2SuccessHandler)
                .failureHandler(oAuth2FailureHandler)
                .userInfoEndpoint()
                .userService(customOAuth2UserService);

        return http.build();
    }
}
```

```java
@Slf4j
public class JwtExceptionFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        response.setCharacterEncoding("utf-8");

        try{
            filterChain.doFilter(request, response);
        } catch (ExpiredJwtException e){
            request.setAttribute("exception", e.getMessage());
        }
        filterChain.doFilter(request, response);

    }
}
```

```java
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException authException) throws IOException {

        response.sendError(HttpServletResponse.SC_UNAUTHORIZED);
    }
}
```

```java
@RequiredArgsConstructor
public class JwtAuthFilter extends GenericFilterBean {

    private final TokenService tokenService;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException {

        String token = tokenService.resolveToken((HttpServletRequest) request);

        if (token != null && tokenService.validateToken(token)) {
            String email = tokenService.getEmail(token);

            Authentication auth = new UsernamePasswordAuthenticationToken(email, "",
                    Arrays.asList(new SimpleGrantedAuthority("ROLE_USER")));

            SecurityContextHolder.getContext().setAuthentication(auth);
        }

        chain.doFilter(request, response);
    }
}
```

- JWT 만료시에도 Internal Server가 아니라, 위에서 입력한 대로 JwtAuthenticationEntryPoint에서 처리한 대로 SC_UNAUTHORIZED가 나옴을 알 수 있습니다.
- 따라서 JWT 만료의 경우, 원하는 대로 에러를 처리할 수 있게 되었습니다.
![Alt text](/assets/img/2023-09-28/image-2.png)

- 그런데 오히려 JWT 만료되지 않는 경우가 에러가 발생했습니다...
![Alt text](/assets/img/2023-09-28/image-5.png)

- NullPointerException이 null로 출력되었습니다. 그리고 Usercontroller의 getUserInfo 메소드(UserController의 29번째 줄)인 String email = authetication.getName() 부분이 null임을 알 수 있습니다. 따라서 authetication 부분이 null로 들어가고 있는 것이었습니다.
- authetication 부분은 인증을 받고 나서 SecurityContextHolder 부분에 들어가 있는 부분입니다.
![Alt text](/assets/img/2023-09-28/image-10.png)
![Alt text](/assets/img/2023-09-28/image-7.png)


- 오류를 찾기 위해서, 다시 localhost에 API를 호출하였는데, 1개의 API에 2개의 응답이 오고 있었습니다.
![Alt text](/assets/img/2023-09-28/image-6.png)

- 다른 API를 호출하였을 때에는 호출을 제대로 하고, 다음으로는 null을 반환하였습니다.
- 즉, 1개의 API가 2번 수행되고 있고, 1번의 수행은 제대로 되었지만, 2번째의 수행에서는 제대로 되지 않았음을 알 수 있습니다.
![Alt text](/assets/img/2023-09-28/image-13.png)

따라서 JwtExceptionFilter에서 filterChain.doFilter를 1개 지워졌더니, null에러가 나오지 않았지만 JWT 만료시에도 예외처리가 되지 않은 모습을 볼 수 있었습니다.

```java
public class JwtExceptionFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        response.setCharacterEncoding("utf-8");

        try{
            filterChain.doFilter(request, response);
        } catch (ExpiredJwtException e){
            //만료 에러
            request.setAttribute("exception", e.getMessage());

        }
//        filterChain.doFilter(request, response);

    }
}
```
![Alt text](/assets/img/2023-09-28/image-8.png)
![Alt text](/assets/img/2023-09-28/image-9.png)

따라서, JwtExceptionFilter에서 dofilter가 2번 되어 있기 때문에 만료되지 않은 JWT를 사용하면 2번의 API가 실행됨을 알 수 있습니다.
하지만 dofilter를 지우게 되면 만료된 경우를 예외 처리할 수 없게 됩니다.

![Alt text](/assets/img/2023-09-28/image-11.png)

### 참고 자료
- https://velog.io/@jkijki12/JWT-스프링-시큐리티-JWT-예외처리하기-및-고찰
- https://ailiartsua.tistory.com/25
- https://velog.io/@hellonayeon/spring-boot-jwt-expire-exception
- https://yoo-dev.tistory.com/28

---

## 3. 성공한 JWT 만료 처리 추가 벙법

### exceptionHandling().authenticationEntryPoint(new JwtAuthenticationEntryPoint())
- filter에서 발생한 예외를 handling 할 수 있도록 추가해줍니다.
- 이때, 저희는 Jwt 인증 관련 예외를 처리할 예정이기 때문에, 인증 예외를 처리하는 authenticationEntryPoint를 추가해주겠습니다.
- 저희는 기본 오류 페이지가 아닌 만료된 JWT임을 알려주는 JSON 데이터 등으로 응답해야 하기 때문에, 직접 AuthenticationEntryPoint 인터페이스를 구현하고 구현체를 시큐리티에 등록하여 사용하겠습니다.
- 즉, AuthenticationEntryPoint 인터페이스는 인증되지 않은 사용자가 인증이 필요한 요청 엔드포인트로 접근하려 할 때, 예외를 핸들링 할 수 있도록 도와줍니다.

```java
@RequiredArgsConstructor
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .csrf().disable()
                .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .cors().configurationSource(corsConfigurationSource())
                .and()
                .headers().frameOptions().disable()
                .and()
                .authorizeRequests()
                .antMatchers("/h2-console/**", "/actuator/**",
                        "/", "/api-docs/**", "/swagger-ui/**").permitAll()
                .antMatchers("/v1/**", "/login/**").hasRole(Role.USER.name())
                .anyRequest().authenticated()
                .and()
                 .logout()
                 .logoutSuccessUrl("/")
                 .and()
                 .exceptionHandling()
                 .authenticationEntryPoint(new JwtAuthenticationEntryPoint())
                 .and()
                 .addFilterBefore(new JwtAuthFilter(tokenService),
                         UsernamePasswordAuthenticationFilter.class)
                 .oauth2Login()
                .successHandler(oAuth2SuccessHandler)
                .failureHandler(oAuth2FailureHandler)
                .userInfoEndpoint()
                .userService(customOAuth2UserService);
        return http.build();
    }
}
```

```java
public class JwtAuthFilter extends GenericFilterBean {

    private final TokenService tokenService;

    @Override
     public void doFilter(ServletRequest request, ServletResponse response,
             FilterChain chain) throws IOException, ServletException {

         try {
             String token = tokenService.resolveToken((HttpServletRequest) request);

             if (token != null && tokenService.validateToken(token)) {
                 String email = tokenService.getEmail(token);

                 Authentication auth = new UsernamePasswordAuthenticationToken(email, "",
                         Arrays.asList(new SimpleGrantedAuthority("ROLE_USER")));

                 SecurityContextHolder.getContext().setAuthentication(auth);
             }
         } catch (Exception e) {
             request.setAttribute("exception", e.getMessage());
         }

         chain.doFilter(request, response);
     }
}

@Service
@Slf4j
public class TokenService {

    public boolean validateToken(String token) {
        Jws<Claims> claims = Jwts.parserBuilder().setSigningKey(secretKey)
                .build().parseClaimsJws(token);

        return claims.getBody().getExpiration()
                .after(new Date(System.currentTimeMillis()));
    }
}
```
```java
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException authException) throws IOException {

        response.sendError(HttpServletResponse.SC_UNAUTHORIZED);
    }
}
```

- 아래의 사진처럼 성공적으로 JWT 만료시에는 401 에러가 나오고, JWT가 만료되지 않았다면 200 성공 응답과 관련 json이 응답됨을 알 수 있습니다.

![Alt text](/assets/img/2023-09-28/image-2.png)

![Alt text](/assets/img/2023-09-28/image-12.png)

- 이때, 저희는 아래와 같이 401 에러가 아닌 400 에러와 함께 json으로 응답 메시지까지 설정해주어서 동일한 에러 응답 형태로 반환할 수 있게 되었습니다!

![Alt text](/assets/img/2023-09-28/image.png)

```java
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException authException) throws IOException {

        response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
        response.setCharacterEncoding("utf-8");
        response.setContentType("application/json");

        JwtErrorResponse jwtErrorResponse = new JwtErrorResponse(
                ErrorCode.INVALID_AUTH_TOKEN.getCode(), ErrorCode.INVALID_AUTH_TOKEN.getMessage());
        ObjectMapper objectMapper = new ObjectMapper();
        String result = objectMapper.writeValueAsString(jwtErrorResponse);

        response.getWriter().write(result);
    }
}
```

---

# 2️⃣ 결론

- 이번 계기로 JWT 도입과 같이 새로운 기능을 추가하게 되는 경우, 모든 에러 처리를 대비하는 것이 매우 중요하다는 것을 알게 되었습니다.
- 다행히도 출시 전인 개발 과정에서 에러 처리를 추가할 수 있게 되었지만, 만약에 출시 후였다면 모든 회원들에게 영향을 미칠 정도로 매우 큰 에러이기 때문에 다음부터는 더 세세하게 많은 에러의 상황을 고려하고 전반적인 에러 처리를 해야겠다는 다짐을 할 수 있게 되었습니다.
- 에러를 처리할 때에도 여러 개의 예시들을 찾아보면서 나의 프로젝트에 맞는 에러 방법을 찾아가고, 예외를 처리를 추가하다가 오히려 JWT 만료되지 않은 경우와 같이 정상적으로 동작하는 로직을 건들여서 에러를 발생할 수 있는 것처럼 추가한 상황에 대한 테스트뿐만 아니라 이전의 모든 테스트 상황도 테스트가 완벽하게 되는지 확인하고 코드를 추가해야겠다는 다짐을 한 번 더 하게 되었습니다!!
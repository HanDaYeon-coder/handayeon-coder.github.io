---
title: "[Spring Security] OAuth2 로그인 + JWT 구현하기"
date: 2023-07-15
author: 다나
categories: [OAuth2, JWT, Spring Security]
---

### PART 1. Spring Security 설정하기
**Spring Security**를 사용하여, 구글 소셜 로그인을 먼저 구현하고자 합니다.
- 자료들을 참고하면서 직접 구현한 코드를 보면서 해당 코드에 대해서 상세하게 살펴보겠습니다!

첫번째로 살펴볼 코드는 **시큐리티 관련 설정 코드**를 작성한 파일(**SecurityConfig**)입니다.
- 이때, Spring Security에 관련된 소셜 로그인과 JWT 설정을 같이 해주고 있습니다.

스프링 스큐리티의 특징을 잘 알면, 코드에 대해서 쉽게 이해할 수 있습니다.

<br/>

💭 **스프링 스큐리티** 관련 설정에 왜 Filter라는 단어가 있을까요?? 어떠한 관계가 있을까요??

스프링 스큐리티는 가장 중요한 것은 Filter라고 할 수 있을 만큼 떼어낼 수 없는 사이입니다.

따라서, 스프링 스큐리티는 **서블릿 필터 체인**을 구성하고, 요청을 거치게 됩니다.
즉, 필터 체인을 통과하게 되면 자원의 해당 servlet을 접근할 수 있습니다.

**SecurityConfig 파일**에서는 이러한 **필터들에 대한 설정을 커스텀하고 새로운 필터를 추가**할 수도 있습니다.
👍 해당 프로젝트가 스프링 스큐리티를 사용했다면, SecurityConfig를 잘 살펴보면 어떠한 흐름으로 인증과 인가를 하고 있는지 살펴볼 수 있습니다. 

<br/>


![](https://velog.velcdn.com/images/da_na/post/f7ea5da9-4c10-4560-9f44-238980b22997/image.png)

이미지 출처 : https://docs.spring.io/spring-security/reference/servlet/architecture.html#servlet-securityfilterchain

![](https://velog.velcdn.com/images/da_na/post/81c7f2b7-876a-436f-8c81-ad3338c88eb9/image.png)

이미지 출처 : https://youmekko.github.io/2018/04/26/2018-04-26-Filter/


```java
@RequiredArgsConstructor
@EnableWebSecurity
public class SecurityConfig {

    private final CustomOAuth2UserService customOAuth2UserService;
    private final TokenService tokenService;
    private final OAuth2SuccessHandler oAuth2SuccessHandler;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

         http
                .csrf().disable()
                 .sessionManagement()
                    .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                 .and()
                .headers().frameOptions().disable()
                 .and()
                    .authorizeRequests()
                        .antMatchers("/h2-console/**", "/",).permitAll()
                        .antMatchers("/api/v1/**").hasRole(Role.USER.name())
                        .anyRequest().authenticated()
                 .and()
                    .logout()
                        .logoutSuccessUrl("/")
                 .and()
                    .addFilterBefore(new JwtAuthFilter(tokenService),
                            UsernamePasswordAuthenticationFilter.class)
                    .oauth2Login()
                        .successHandler(oAuth2SuccessHandler)
                            .userInfoEndpoint()
                            .userService(customOAuth2UserService);

         return http.build();
    }
}
```
1️⃣ `@EnableWebSecurity`
- Spring Security 설정들을 활성화시켜줍니다.

2️⃣ `sessionManagement()
                    .sessionCreationPolicy(SessionCreationPolicy.STATELESS)`
- 세션을 스프링 시큐리티가 생성하지도 않고 존재해도 사용하지 않습니다.
- 저는 세션이 아닌 JWT를 사용하여 인증과 인가를 진행할 예정이기 때문에, 세션을 사용하지 않는다고 설정하였습니다.

3️⃣ `.authorizeRequests()
                        .antMatchers("/h2-console/**", "/",).permitAll()
                        .antMatchers("/api/v1/**").hasRole(Role.USER.name())
                        .anyRequest().authenticated()`
- URL 별로 권한 관리를 설정하는 부분입니다.
- "/h2-console/**"과 "/"로 시작하는 URL은 현재 누구든지 볼 수 있도록 전체 권한을 부여하였습니다. (permitAll)
- 그러나 현재 저희는 로그인한 회원만이 저희 서비스를 사용할 수 있도록 USER 권한이 있는 사람들에게만 "/api/v1/**"으로 시작하는 URL에 접근할 수 있습니다.
- 그리고 나머지 URL에도 인증된 사용자만 사용할 수 있습니다.
- ☝️ 이때, 중요한 점은 인증된 사용자만 사용할 수 있도록 되어 있기 때문에 JWT를 사용하면 해당 접근 토큰을 사용하지 않고 url에 접근하면 소셜 로그인을 하라고 리다이렉트 시킨다는 점입니다. 

4️⃣ `.logout().logoutSuccessUrl("/")`
- 로그아웃과 관련된 설정을 하는 사항입니다.
이때, 로그인에 성공한다면 "/" 주소로 이동하게 됩니다.
- 해당 로그아웃 필터를 커스텀하고 싶다면 위의 코드처럼 로그아웃 설정을 해줄 수 있습니다.
- 그리고 로그아웃 Url을 따로 만들 필요 없이, "/logout"을 요청하면 스프링 스큐리티에서 직접 logout filter를 사용해서 해당 세션 무효화, 인증토큰 삭제, 인증토큰을 갖고 있던 SecurityContext 또한 삭제, 쿠키정보 삭제, 로그인 페이지로 리다이렉트를 시켜줍니다.

5️⃣ `.addFilterBefore(new JwtAuthFilter(tokenService),UsernamePasswordAuthenticationFilter.class)`
- JWT토큰 필터를 UsernamePasswordAuthenticationFilter 앞에 추가하여 JWT 토큰의 유효값 체크 및 해당 사용자 관련 JWT 인증을 하게 됩니다. 
- JwtAuthFilter는 제가 정의한 필터입니다! 이처럼 사용자가 정의한 필터를 추가할 수 있습니다!

  <br/>
![](https://velog.velcdn.com/images/da_na/post/c874b3d1-a678-4cab-b9fd-63cdc5fd4354/image.png)

그림출처 : https://atin.tistory.com/590

6️⃣ `.oauth2Login().successHandler(oAuth2SuccessHandler).userInfoEndpoint().userService(customOAuth2UserService)`
- OAuth2 로그인 기능에 대한 설정사항입니다.
- 만약에 OAuth2 로그인에 성공했다면, 사용자가 정의한 oAuth2SuccessHandler에서 JWT를 발급하고 client에게 Access Token을 url로 알려줄 수 있습니다.
- userInfoEndPoint는 소셜 로그인 성공 이후 사용자의 정보를 가져올 때의 설정을 담당하고, userService에서 사용자를 회원가입 시키거나, 사용자가 회원가입이 되어 있는지 확인할 수 있습니다.

---
### PART 2. OAuth2UserService
먼저, 소셜 로그인을 살펴보겠습니다.
아래의 첫번째 코드는 소셜 로그인의 서비스 로직을 나타냈습니다.

```java
@RequiredArgsConstructor
@Service
public class CustomOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {

    private final UserRepository userRepository;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {

        OAuth2UserService<OAuth2UserRequest, OAuth2User> delegate = new DefaultOAuth2UserService();
        OAuth2User oAuth2User = delegate.loadUser(userRequest);

		//로그인 진행중인 서비스를 구분하는 ID -> 여러 개의 소셜 로그인할 때 사용하는 ID
        String registrationId = userRequest.getClientRegistration().getRegistrationId();
        
        //OAuth2 로그인 진행 시 키가 되는 필드값(Primary Key) -> 구글은 기본적으로 해당 값 지원("sub")
        //그러나, 네이버, 카카오 로그인 시 필요한 값
        String userNameAttributeName = userRequest.getClientRegistration()
                .getProviderDetails().getUserInfoEndpoint().getUserNameAttributeName();

		//OAuth2UserSevice를 통해 가져온 OAuth2User의 attribute를 담은 클래스
        OAuthAttributes attributes = OAuthAttributes.of(registrationId,
                userNameAttributeName, oAuth2User.getAttributes());
		
        //우리의 서비스에 회원가입이나 기존 회원의 정보를 업데이트를 한다.
        User user = saveOrUpdate(attributes);

        return new DefaultOAuth2User(
                Collections.singleton(new SimpleGrantedAuthority(user.getRoleKey())),
                attributes.getAttributes(), attributes.getNameAttributeKey());
    }

    private User saveOrUpdate(OAuthAttributes attributes) {
		//소셜 로그인의 회원 정보가 업데이트 되었다면, 기존 DB에 저장된 회원의 이름을 업데이트해줍니다.
        User user = userRepository.findByEmail(attributes.getEmail())
                .map(entity -> entity.update(attributes.getName()))
                .orElse(attributes.toEntity());
                
		//만약에 DB에 등록된 이메일이 아니라면, save하여 DB에 등록(회원가입)을 진행시켜준다.
        return userRepository.save(user);
    }
}
```
- 아래의 코드는 User 테이블(Entity)에 대한 파일입니다.
role은 현재 User만 부여하고 있지만, 추후에는 관리자와 같은 다른 역할(권한)을 부여할 수도 있습니다.

```java
@Getter
@NoArgsConstructor
@Entity
public class User extends BaseTimeEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "USER_ID")
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String email;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Role role;

    @Builder
    public User(String name, String email, Role role) {
        this.name = name;
        this.email = email;
        this.role = role;
    }

    public User update(String name) {
        this.name = name;
        return this;
    }

    public String getRoleKey() {
        return this.role.getKey();
    }
}
```
```java
@Getter
@RequiredArgsConstructor
public enum Role {

    USER("ROLE_USER", "일반 사용자");

    private final String key;
    private final String title;
}

```

- 아래의 코드는 **OAuth2 소셜 로그인시 사용하는 Dto**입니다.

☝️ 구글과 네이버 등 해당 기업의 서버에 회원 정보를 요청할 때 사용하므로 기업에서 제공하는 정보에 맞게 Dto를 구성해야 합니다. 
예를 들어, 구글은 프로필 이미지 Url을 제공할 수 있지만, 다른 소셜 기업은 이미지 Url을 제공하지 않을 수 있습니다.

```java
@Getter
public class OAuthAttributes {

    private Map<String, Object> attributes;
    private String nameAttributeKey;
    private String name;
    private String email;
    private String pictureURL;

    @Builder
    public OAuthAttributes(Map<String, Object> attributes,
            String nameAttributeKey,
            String name, String email, String pictureURL) {

        this.attributes = attributes;
        this.nameAttributeKey = nameAttributeKey;
        this.name = name;
        this.email = email;
        this.pictureURL = pictureURL;
    }

    public static OAuthAttributes of(String registrationId,
            String userNameAttributeName,
            Map<String, Object> attributes) {

        return ofGoogle(userNameAttributeName, attributes);
    }

    private static OAuthAttributes ofGoogle(String userNameAttributeName,
            Map<String, Object> attributes) {

        return OAuthAttributes.builder()
                .name((String) attributes.get("name"))
                .email((String) attributes.get("email"))
                .pictureURL((String) attributes.get("picture"))
                .attributes(attributes)
                .nameAttributeKey(userNameAttributeName)
                .build();
    }

    public User toEntity() {
        return User.builder()
                .name(name)
                .email(email)
                .role(Role.USER)
                .build();
    }
}

```

- 🗝️ 아래의 코드는 OAuth2 소셜 로그인이 성공적으로 되었다면, 실행되는 Handler입니다.

소셜 로그인 성공 후에는 token을 생성하여 클라이언트쪽에서도 앞으로 api를 요청할 때마다 해당 토큰 같이 넘겨주면 인증과 인가를 하지 않아도 됩니다. 따라서 리다이렉트된 url에서 해당 토큰을 파싱해야 합니다.

```java
@Slf4j
@RequiredArgsConstructor
@Component
public class OAuth2SuccessHandler extends SimpleUrlAuthenticationSuccessHandler {

    private final TokenService tokenService;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response
            , Authentication authentication) throws IOException {

        OAuth2User oAuth2User = (OAuth2User) authentication.getPrincipal();
        String email = oAuth2User.getAttribute("email");
        String name = oAuth2User.getAttribute("name");

        String token = tokenService.generateToken(name, email, "USER");

        String targetUrl = UriComponentsBuilder.fromUriString("/")
                .queryParam("token", token)
                .build().toUriString();

        getRedirectStrategy().sendRedirect(request, response, targetUrl);
    }
}

```
---
### PART 3. JWT 발급 로직
Part1에서 FilterChain에 추가한 JwtAuthFilter를 직접 정의해보겠습니다.
- 여기에서 중요한 점은 **SecurityContextHolder**에 인증된 회원의 정보를 저장해놓습니다.
그리고 앞으로 서비스에서 authenticate된 principal에 접근하고 싶으면, SecurityContextHolder 에 접근하면 됩니다.

![](https://velog.velcdn.com/images/da_na/post/57f47ecd-ffa1-4326-82a3-80f5690a2b6b/image.png)

```java
@RequiredArgsConstructor
public class JwtAuthFilter extends GenericFilterBean {

    private final TokenService tokenService;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException {
		//HttpServletRequest에서 헤더에 "X-AUTH-TOKEN"에 작성된 token을 가져옵니다.
        String token = tokenService.resolveToken((HttpServletRequest) request);
		
        //헤더에 작성된 토큰이 있는지 확인하고, 토큰이 만료되었는지 확인합니다.
        if (token != null && tokenService.validateToken(token)) {
        	//토큰에서 secret key를 사용하여 회원의 이메일을 가져옵니다.
            String email = tokenService.getEmail(token);
			
            //인증된 회원의 정보를 SecurityContextHolder에 저장합니다.
            //현재는 역할이 ROLE_USER뿐이라서 권한을 직접 주는 형태로 하였으나, 권한이 여러개인 경우 변경해야 합니다.
            Authentication auth = new UsernamePasswordAuthenticationToken(email, "",
                    Arrays.asList(new SimpleGrantedAuthority("ROLE_USER")));
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        
        //토큰이 없거나 만료된 토큰이라면, 다시 소셜 로그인을 진행하는 과정을 수행합니다.
        chain.doFilter(request, response);
    }
}
```

- ☝️ 이때, 중요한 점은 SECRET_KEY와 EXPIRE_LENGTH는 가장 중요한 정보이기 때문에 절대로 다른 곳에 유출되어서도 공개 되어서도 안됩니다. 따라서, 저는 yml을 따로 선언하여 해당 yml에서 가져오는 방식을 선택하였습니다. (해당 yml은 .gitignore 파일에 추가하여 깃허브에도 올라가지 않아야 합니다.)
- 그리고 지금은 토큰에 이메일, 이름, 역할(권한)을 넣었는데, 토큰은 언제든지 탈취당할 수 있으므로 비밀번호와 같이 민감한 개인정보는 절대로 넣지 말아야 합니다!!

```java
@Service
public class TokenService {

    private Key secretKey;
	
    @Value("${security.jwt.token.secret-key}")
    private String SECRET_KEY;

    @Value("${security.jwt.token.expire-length}")
    private Long EXPIRE_LENGTH;

    @PostConstruct
    protected void init() {
        secretKey = Keys.hmacShaKeyFor(Decoders.BASE64.decode(SECRET_KEY));
    }

    public String generateToken(String name, String email, String role) {
        Claims claims = Jwts.claims().setSubject(email);
        claims.put("name", name);
        claims.put("role", role);

        return Jwts.builder().setClaims(claims)
                .setIssuedAt(new Date(System.currentTimeMillis()))
                .setExpiration(new Date(System.currentTimeMillis() + EXPIRE_LENGTH))
                .signWith(secretKey, SignatureAlgorithm.HS256)
                .compact();
    }

    public boolean validateToken(String token) {
        try {
            Jws<Claims> claims = Jwts.parserBuilder().setSigningKey(secretKey)
                    .build().parseClaimsJws(token);

            return claims.getBody().getExpiration()
                    .after(new Date(System.currentTimeMillis()));

        } catch (Exception e) {
            return false;
        }
    }

    public String getEmail(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(secretKey)
                .build().parseClaimsJws(token).getBody().getSubject();
    }

    public String resolveToken(HttpServletRequest request) {
        return request.getHeader("X-AUTH-TOKEN");
    }
}
```
---
### 참고 자료
- https://velog.io/@jkijki12/Spring-Boot-OAuth2-JWT-%EC%A0%81%EC%9A%A9%ED%95%B4%EB%B3%B4%EB%A6%AC%EA%B8%B0
- https://hou27.tistory.com/entry/Spring-Security-JWT
- https://imbf.github.io/spring/2020/06/29/Spring-Security-with-JWT.html
- https://velog.io/@dailylifecoding/spring-security-logout-feature
- 책 : 스프링 부트와 AWS로 혼자 구현하는 웹 서비스

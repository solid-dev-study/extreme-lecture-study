# Section 9. 인가 프로세스
## 요청 기반 권한 부여 - HttpSecurity.authorizeRequests()
- `HttpServletRequest`에 대한 권한 부여
- `authorizeHttpRequests()`를 사용
  - 요청 엔드포인트와 접근에 필요한 권한을 매핑시키기 위한 규칙을 설정
- 요청에 대한 권한 규칙을 설정하면 내부적으로 `AuthorizationFilter`가 요청에 대한 권한 검사 및 승인 작업 수행

```java
@Bean
SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) {
    http.authorizeHttpRequests(authorize -> authorize
        .anyRequest().authenticated());
        // (모든 요청에 대해).(인증이 되어야 함)

    return http.build();
}
```

### requestMatchers()
- 요청에 대한 보안을 세밀하게 적용
- `.requestMatchers(String... urlPatterns)`
  - 보호가 필요한 자원을 N개 정의
- `.requestMatchers(RequestMatcher... requestMachers)`
  - `AntPathRequestMatcher`, `MvcRequestMatcher` 등의 구현체를 사용하여 보호가 필요한 자원을 N개 정의
- `.requestMatchers(HttpMethod method, String... urlPatterns)`
  - HttpMethod와 보호가 필요한 자원을 N개 정의
- `.hasRole()`을 사용하여 필요 권한 정의 
  - ex. `requestMatchers("/admin").hasRole("ADMIN")`
    - "보호해야 할 자원"은 "특정 권한"이 필요

![image001.png](./images/image001.png)
- 순서대로 처리되므로, 순서대로 조회 중 조건에 걸릴경우 이후 조건은 검사하지 않음
  - 따라서 좁은 범위의 경로를 먼저 정의하고, 그 뒤에 그것보다 큰 범위를 정의

![image002.png](./images/image002.png)

> `hasRole()`과 `hasAuthority()`의 차이
>
> - `hasRole()`은 "ROLE_" 접두사를 자동으로 붙여서 검사하기에, 명확한 역할이 필요 (따라서 "ROLE_" 접두사를 붙이면 안됨)
>
> - `hasAuthority()`는 "ROLE_"접두사를 붙이지 않기 때문에 유연

## 표현식 및 커스텀 권한 구현
- `WebExpressionAuthorizationManager`를 통해 표현식으로 권한 규칙을 설정

```java
requestMatchers("/admin/{name}").access(new WebExpressionAuthorizationManager("#name == authentication.name"))

requestMatchers("/admin/db").access(new WebExpressionAuthorizationManager("hasAuthority('DB') or hasRole('ADMIN')"))

// same
requestMatchers("/admin/db").access(anyOf(hasAuthority("db"), hasRole("ADMIN")))
```

### Custom 권한 표현식 구현

```java
SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http, ApplicationContext context) {
    // 이 핸들러가 표현식 구문을 처리
    DefaultHttpSecurityExpressionHandler expressionHandler = new DefaultHttpSecurityExpressionHandler();
    expressionHandler.setApplicationContext(context);

    WebExpressionAuthorizationManager expressManager = new WebExpressionAuthorizationManager("@customWebSecurity.check(authentication, request)");
    expressionManager.setExpressionHandler(expressionHadler);

    // AuthorizationManager를 등록
    http.authorizeHttpRequests(authorize -> authorize
        .requestMatchers("/resource/**").access(expressionManager));

    return http.build();
}

@Component("customWebSecurity")
public class CustomWebSecurity {
    // 클래스명과 메서드명은 아무렇게나 만들어도 됨
    public boolean check(Authentication authentication, HttpServletRequest request) {
        return authentication.isAuthenticated();
    }
}
```

### Custom RequestMatcher 구현

```java
public class CustomRequestMatcher implements RequestMatcher {
    private final String urlPattern;
    public CustomRequestMatcher(String urlPattern) {
        this.urlPattern = urlPattern;
    }

    @Override
    public boolean matches(HttpServletRequest request) {
        String requestURI = request.getRequestURI();
        return requestURI.startsWith(urlPattern);
    }
}
```

## 요청 기반 권한 부여 - HttpSecurity.securityMatcher()
- 특정 패턴에 해당하는 요청에만 보안 규칙(엔드포인트에 대한 규칙)을 적용
- 중복해서 사용할 경우 마지막에 설정한 것으로 대체
- `securityMatcher(String... urlPatterns)`
- `securityMatcher(RequestMatcher... requestMatchers)`
- `SecurityFilterChain`은 `RequestMatcher`와 `Filter`를 들고 있는데, `RequestMatcher`에 부합해야 `Filter`가 수행 됨

```java
// 단일 패턴
http.securityMatcher("/api/**").authorizeHttpRequests(auth -> auth.requestMatchers(...))
```

```java
// 다중 패턴 1
http.securityMatchers((matchers) -> matchers.requestMatchers("/api/**", "/oauth/**"));

// 다중 패턴 2
http.securityMatchers((matchers) -> matchers.requestMatchers("/api/**").requestMatchers("/oauth/**"));

// 다중 패턴 3
http.securityMatchers((matchers) -> matchers.requestMatchers("/api/**"))
    .securityMatchers((matchers) -> matchers.requestMatchers("/oauth/**"));
```

# 이벤트 처리
## 인증 이벤트
- 인증 성공/실패시 이벤트를 발생
  - 인증 성공 시 : `AuthenticationSuccessEvent`
  - 인증 실패 시 : `AuthenticationFailureEvent`
- `AuthenticationEventPublisher`를 통해 발행하며, 기본 구현체로 `DefaultAuthenticationEventPublisher`를 제공
### 이벤트 종류
- 최상위 이벤트
  - `AbstractAuthenticationEvent` : 모든 인증 이벤트의 최상위 추상 클래스
  - `ABstractAuthenticationFailureEvent` : 모든 인증 실패 이벤트의 최상위 추상 클래스
- 인증 성공 이벤트
  - `AuthenticationSuccessEvent` : 사용자 인증이 성공적으로 완료되었을 때 발생
  - `InteractiveAuthenticationSuccessEvent` : UI를 통한 대화형 인증이 성공했을 때 발생
- 인증 실패 이벤트
  - `AuthenticationFailureBadCredentialsEvent` : 잘못된 사용자 이름 또는 비밀번호로 인증에 실패했을 때 발생
  - `AuthenticationFailureCredentialsExpiredEvent` : 사용자의 비밀번호가 만료되어 인증에 실패했을 때 발생 
  - `AuthenticationFailureDisabledEvent` : 사용자 계정이 비활성화되어 인증에 실패했을 때 발생
  - `AuthenticationFailureExpiredEvent` : 사용자 계정 자체가 만료되어 인증에 실패했을 때 발생
  - `AuthenticationFailureLockedEvent` : 사용자 계정이 잠금 상태여서 인증에 실패했을 때 발생
  - `AuthenticationFailureProviderNotFoundEvent` : 요청된 인증을 처리할 인증 공급자를 찾을 수 없을 때 발생
  - `AuthenticationFailureProxyUntrustedEvent` : 프록시를 통한 인증 시 신뢰할 수 없는 프록시 요청이 왔을 때 발생
  - `AuthenticationFailureServiceExceptionEvent` : 인증 과정 중 예상치 못한 내부 서비스 예외가 발생하여 인증에 실패했을 때 발생

### 인증 성공 이벤트
```java
// 커스텀 이벤트 발행 (Handler 사용)
AuthenticationSuccessHandler() {
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException {
        // publisher는 빈으로 주입받음
        applicationEventPublisher.publishEvent(new CustomAuthenticationSuccessEvent(authentication));
    }
}
```

```java
// 이벤트 수신
@Component
public class AuthenticationSuccessEvents {
    // 파라미터로 수신받을 이벤트 객체 선언
    @EventListener
    public void onSuccess(AuthenticationSuccessEvent success) {
        ...
    }

    @EventListener
    public void onSuccess(CustomAuthenticationSuccessEvent success) {
        ...
    }
}
```

### 인증 실패 이벤트
```java
// 인증 실패 이벤트 발행 (기본 클래스)

// 스프링 자체에서 제공(범용 발행자) -> 이벤트를 파라미터로
applicationEventPublisher.publishEvent(
    new AuthenticationFailureDisabledEvent(
        authentication, 
        new DisabledException("message")
    )
);

// 스프링 시큐리티 전용 -> 예외를 파라미터로
authenticationEventPublisher.publishAuthenticationFailure(
    new BadCredentialsException("message"),
    authentication
);
```

```java
// 이벤트 수신
@Component
public class AuthenticationFailureEvents {
    @EventListener
    public void onFailure(AbstractAuthenticationFailureEvent failures) {
        ...
    }
}
```

### 커스텀 예외
```java
// Publisher에 커스텀 예외를 추가
@Bean
public AuthenticationEventPublisher customAuthenticationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
    Map<Class<? extends AuthenticationException>, Class<? extends AbstractAuthenticationFailureEvent>> mapping = Collections.singletonMap (CustomException.class, CustomAuthenticationFailureEvent.class);

    DefaultAuthenticationEventPublisher authenticationEventPublisher = new DefaultAuthenticationEventPublisher(applicationEventPublisher);
    authenticationEventPublisher.setAdditionalExceptionMappings(mapping);

    return authenticationEventPublisher;
}

// 예외 발행
authenticationEventPublisher.publishAuthenticationFailure(
    new CustomException("message"), 
    authentication
);

// 수신
@EventListener
public void onFailure(CustomAuthenticationFailureEvent failure) {
    ...
}
```

## 인증 이벤트
- "권한"이 부여되거나 거부되는 경우 발생
- `AuthorizationEventPublisher`를 통해 발행하며, 기본 구현체로 `SpringAuthorizationEventPublisher` 제공

```java
// 인증 이벤트 발행

// Bean 정의 필요
@Bean
public AuthorizationEventPublisher authorizationEventPubliser(ApplicationEventPublisher publisher) {
    return new SpringAuthorizationEventPublisher(publisher);
}

authorizationEventPublisher.publishAuthorizationEvent(
    () -> authentication, // 사용자 인증정보 Supplier
    "resourceName", // 인가가 필요한 리소스
    new AuthorizationDecision(false) // 인가 결정의 최종 결과 -> 이벤트 타입이 결정 됨
    // true -> AuthorizationGrantedEvent (인가 허용 이벤트)
    // false -> AuthorizationDeniedEvent (인가 거부 이벤트)
)

// 인증 이벤트 수신
@Component
public class AuthorizationEvents {
    @EventListener
    public void onAuthorization(AuthorizationEvent event) {
        ...
    }
}
```

```java
// 커스텀 AuthorizationEventPublisher
@Component
public class MyAuthorizationEventPublisher implements AuthorizationEventPublisher {
    @Override
    public <T> void publishAuthorizationEvent(
        Supplier<Authentication> authentication,
        T object, 
        AuthorizationDecision decision
    ) {
        if (decision == null) return;

        if (!decision.isGranted()) { // 인가 실패 이벤트를 발행한다
            this.delegate.publishAuthorizationEvent(authentication, object, decision);
            return;
        }
        if (shouldThisEventBePublished(decision)) {
            this.publisher.publishEvent(
                new AuthorizationGrantedEvent(authentication, object, decision)
            );
        }
    }

    private boolean shouldThisEventBePublished(AuthorizationDecision decision) {
        if (!(decision instanceof AuthorityAuthorizationDecision)) {
            return false;
        }

        Collection<GrantedAuthority> authorities = 
            ((AuthorityAuthorizationDecision) decision).getAuthorities();

        for (GrantedAuthority authority : authorities) {
            if ("ROLE_ADMIN".equals(authority.getAuthority())) {
                return true;
            }
            return false;
        }
    }
}
```

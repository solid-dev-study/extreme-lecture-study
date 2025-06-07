# SecurityContextRepository
## 역할
- SecurityContext의 영속 저장/복원 담당

## 동작
- Security Context를 HttpSession에 SPRING_SECURITY_CONTEXT 키로 저장하고 복원

## 흐름
- [인증 요청] -> [AuthenticationFilter] -> [SecurityContext] -> [SecurityContextRepository] -> [HttpSession]
- [인증 후 요청] -> [SecurityContextHolderFilter] -> [SecurityContextPersistenceFilter] -> [HttpSession] -> [SecurityContext]

# SecurityContextHolderFilter
## 역할
- ThreadLocal 기반 SecurityContextHolder 초기화 및 정리 (메모리 누수 방지)

# SecurityContextPersistenceFilter
- SecurityContext의 영속화 담당
- 사용자가 로그인한 이후 -> 다음 요청에서도 인증 상태 유지

# 스프링 MVC 인증 구현
- 수동으로 사용자 인증 시 스프링 MVC 컨트롤러 엔드포인트 사용 가능

```java
@RestController
@RequestMapping("/api/v1/auth/login")
@RequiredArgsConstructor
public class AuthController {

    private final AuthenticationManager authenticationManager;

    @PostMapping
    public ResponseEntity<ApiResponse<Void>> login(@RequestBody LoginRequest request, HttpServletRequest httpRequest) {
        UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(request.email(), request.password());

        Authentication authentication = authenticationManager.authenticate(token);

        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(authentication);
        SecurityContextHolder.setContext(context);

        HttpSession session = httpRequest.getSession(true);
        session.setAttribute(HttpSessionSecurityContextRepository.SPRING_SECURITY_CONTEXT_KEY, context);

        return ApiResponseFactory.success();
    }
}
```
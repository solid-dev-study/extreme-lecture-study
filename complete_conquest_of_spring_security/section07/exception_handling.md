# 예외 처리
- 필터 체인 내에서 발생하는 예외 의미
  - 인증 예외 (AuthenticationException)
  - 인가 예외 (AccessDeniedException)
- ExceptionTranslationFilter로 예외 처리 수행

# 예외 처리 유형
- 인증 실패 시 -> AuthenticationException 발생
  1. SecurityContext 인증 정보 삭제 
  2. AuthenticationEntryPoint 실행
  3. 인증 프로세스 요청 정보 저장하고 검색
- 인가 실패 시 -> AccessDeniedException 발생 
  - AccessDeniedHandler 호출

# ExceptionTranslationFilter
![img.png](image/ExceptionTranslationFilter.png)
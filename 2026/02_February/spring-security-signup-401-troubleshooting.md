배경

React + Spring Boot 기반 프로젝트에서 회원가입 요청 시
정상적인 요청임에도 불구하고 401 Unauthorized 오류가 발생했다.
로그인 이전 단계인 회원가입 API에서 401이 발생하는 것은 명백히 인증/보안 설정 문제라고 판단했다.

문제 상황

/api/auth/register 요청 시 401 반환

Postman, 프론트엔드 모두 동일 증상

컨트롤러 로직 자체는 정상적으로 호출되지 않음

원인 분석
1. Spring Security 설정 문제

SecurityFilterChain에서 회원가입 API가 permitAll()로 열려 있지 않았음

JWT 필터가 모든 요청에 대해 인증을 요구하고 있었음

.requestMatchers("/api/auth/**").permitAll()


이 설정이 누락되어 회원가입 요청조차 인증 대상이 되고 있었음.

2. CORS 설정 누락

프론트엔드(React)에서 요청 시 브라우저 단계에서 차단

CorsConfigurationSource 미등록 상태

configuration.setAllowedOrigins(List.of("http://localhost:5173"));
configuration.setAllowedMethods(List.of("GET","POST","PUT","DELETE","OPTIONS"));
configuration.setAllowedHeaders(List.of("*"));
configuration.setAllowCredentials(true);

3. 프론트엔드 토큰 처리 혼선

회원가입/로그인 이전 요청인데도
axios 인터셉터에서 무조건 Authorization 헤더를 붙이고 있었음

토큰이 없는 상태 → 잘못된 헤더 → 보안 필터에서 차단

해결 방법 정리

회원가입·로그인 API를 permitAll()로 명확히 분리

CORS 설정을 SecurityConfig에 명시적으로 등록

axios 인터셉터에서 토큰이 있을 때만 Authorization 헤더 추가

인증이 필요한 API / 필요 없는 API를 명확히 구분

배운 점

401 오류는 “로그인이 안 됐다”가 아니라 Security 흐름 전체를 의심해야 한다

Spring Security에서
URL 접근 권한 → 필터 순서 → 프론트 요청 구조는 항상 함께 봐야 한다

회원가입 API도 보안 설정을 잘못하면 쉽게 막힌다

다음에 개선할 점

인증이 필요 없는 API 목록을 상수로 관리

Security 설정 변경 시 Postman 테스트를 먼저 진행

프론트/백엔드 인증 책임을 명확히 분리
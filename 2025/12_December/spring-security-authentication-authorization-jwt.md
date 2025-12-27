[Spring Security] 인증(Authentication)과 인가(Authorization) 정리 + JWT 적용 흐름
1. 배경

Spring Boot 프로젝트에서 “로그인/회원가입”을 구현할 때 가장 자주 혼동하는 개념이 인증(Authentication) 과 인가(Authorization) 이다.
또한 Spring Security는 기본적으로 필터 체인을 통해 보안 로직을 처리하기 때문에, Controller/Service 레벨에서만 접근하면 전체 흐름이 끊겨 보인다.

이번 TIL에서는 다음을 목표로 정리한다.

인증과 인가의 차이

Spring Security가 요청을 처리하는 흐름(필터 체인)

JWT(Access/Refresh) 기반 인증을 어떻게 붙이는지

실무에서 자주 터지는 포인트(403/401, SecurityContext, CORS)

2. 인증(Authentication) vs 인가(Authorization)
2.1 인증(Authentication)

사용자가 누구인지 확인하는 과정

예: 로그인(아이디/비번 검증), 토큰 유효성 검증, 세션 검증

결과

“이 사용자는 userId=3인 사용자다”처럼 사용자 식별 정보가 확정됨

2.2 인가(Authorization)

인증된 사용자가 무엇을 할 수 있는지 결정하는 과정

예: ROLE_USER는 조회만 가능, ROLE_ADMIN은 삭제 가능

결과

“이 사용자는 해당 API 접근 권한이 있다/없다”로 판단됨

정리하면,

인증: Identity

인가: Permission

3. Spring Security 요청 처리 흐름(개념)

Spring Security는 요청이 들어올 때 다음 순서를 통해 “통과/차단”을 결정한다.

클라이언트 요청

Security Filter Chain 진입

인증이 필요한 요청인지 확인

인증 필요 시: 인증 절차 수행 (예: JWT 검증)

인증 성공하면 SecurityContext에 Authentication 저장

인가 규칙에 따라 접근 허용/차단

Controller 도달

즉 Controller는 보안의 “시작점”이 아니라, 보안 필터를 통과한 이후에 호출되는 영역이다.

4. JWT 기반 인증 흐름 (실무형)
4.1 토큰 종류

Access Token: API 호출에 사용, 만료 짧게(예: 15분~1시간)

Refresh Token: Access 재발급용, 만료 길게(예: 7~30일)

4.2 로그인 흐름

사용자 로그인 요청 (email/password)

서버에서 사용자 검증

Access/Refresh 발급

클라이언트 저장

Access: 메모리/스토리지 등(전략에 따라 다름)

Refresh: 보통 HttpOnly Cookie 또는 안전 저장소

4.3 API 호출 흐름

클라이언트가 Authorization: Bearer <accessToken>로 API 요청

서버는 Security Filter에서 토큰 검증

토큰 유효하면 인증 객체 생성 후 SecurityContext에 저장

인가 규칙 통과하면 Controller 실행

5. 필수 구성 요소(구현 관점)
5.1 SecurityConfig의 역할

어떤 요청을 허용할지/막을지 정의

어떤 필터를 체인에 넣을지 정의

세션 사용 여부(Stateful vs Stateless) 결정

JWT 기반이라면 일반적으로:

sessionCreationPolicy(STATELESS)로 세션 사용 안 함

로그인/회원가입은 permitAll

나머지는 authenticated 또는 role 기반 제한

5.2 JWT Filter의 역할

요청 헤더에서 토큰 추출

토큰 유효성 검증 (서명/만료/클레임)

유효하면 Authentication 객체 만들어 SecurityContext에 주입

5.3 UserDetails / UserDetailsService

Spring Security가 인증된 사용자 정보를 다루기 위한 표준 인터페이스

DB에서 사용자를 조회해 Security가 이해할 수 있는 형태로 변환

6. 401 vs 403 자주 헷갈리는 포인트

401 Unauthorized

인증 자체가 안 됨 (토큰 없음/만료/위조)

즉 “누구인지 모름”

403 Forbidden

인증은 됐지만 권한이 없음 (ROLE 부족 등)

즉 “누구인지는 아는데, 못 들어감”

디버깅 시 빠른 체크:

토큰이 아예 없거나, 만료면 401

토큰은 유효한데 역할이 부족하면 403

7. @AuthenticationPrincipal을 통한 사용자 정보 활용

Controller에서는 인증 완료 후 SecurityContext에 저장된 사용자 정보를 꺼내 쓸 수 있다.

목적: API 호출자(userId)를 파라미터로 받지 않고 “현재 로그인 사용자” 기준으로 처리

장점: userId 변조(다른 사람 userId 넣기) 같은 공격을 줄임

활용 예:

“내 플레이리스트 생성”

“내 플레이리스트 수정/삭제”

“좋아요/구독 등 사용자 행동”

8. 실무 주의사항
8.1 CORS

프론트엔드 분리(React 등)면 CORS 설정이 반드시 필요

특히 Authorization 헤더를 사용하면 preflight가 발생할 수 있음

8.2 로그/디버깅

필터에서 토큰 파싱 실패/만료/서명 오류를 로그로 남기면 원인 파악이 빨라짐

단, 토큰 원문 로그는 보안상 위험하므로 최소화

8.3 Refresh Token 저장 전략

로컬스토리지 저장은 XSS 위험이 커서 주의

HttpOnly Cookie는 XSS에는 강하지만 CSRF 설계가 필요

9. 결론(오늘의 인사이트)

인증과 인가를 구분하면 Security 설정/에러 원인(401/403)이 선명해진다.

Spring Security는 Controller 밖(Filter Chain)에서 이미 대부분의 보안 처리가 끝난다.

JWT 인증은 “요청마다 토큰 검증 → SecurityContext 주입”이 핵심이다.

사용자 식별은 파라미터가 아니라 @AuthenticationPrincipal 기반이 안전하다.
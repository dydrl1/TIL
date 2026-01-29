# [Spring Security] JWT Secret 관리 실수로 발생한 Base64 디코딩 오류 트러블슈팅

## 1. 문제 상황
## 2. 에러 로그 분석
## 3. 원인 분석
## 4. 잘못된 설정 방식
## 5. 해결 방법
## 6. 적용 후 구조 정리
## 7. 정리 & 배운 점


# [Spring Security] JWT Secret 관리 실수로 발생한 Base64 디코딩 오류 트러블슈팅

플레이리스트 프로젝트에서 JWT 인증 로직을 적용하던 중,
애플리케이션 실행 단계에서 Bean 생성 실패 에러가 발생했다.
단순한 오타가 아니라 **JWT Secret 관리 방식 자체의 문제**였고,
보안 설정과 환경변수 관리의 중요성을 다시 한 번 체감한 사례였다.

---

## 1. 문제 상황

Spring Boot 애플리케이션 실행 시 아래와 같은 에러가 발생했다.

```text
Caused by: io.jsonwebtoken.io.DecodingException: Illegal base64 character: '_'
에러는 JwtTokenProvider 빈 생성 과정에서 발생했으며,
애플리케이션이 아예 기동되지 않았다.

2. 에러 로그 분석
에러 발생 지점은 JWT Secret을 Key로 변환하는 로직이었다.

Key key = Keys.hmacShaKeyFor(keyBytes);
Keys.hmacShaKeyFor()는 내부적으로 Base64 디코딩 가능한 문자열을 기대하는데,
전달된 문자열에 Base64 규칙에 맞지 않는 문자가 포함되어 있었다.

3. 원인 분석
원인은 다음 두 가지였다.

jwt.secret 값을 평문 문자열로 사용

해당 값을 application.yaml에 직접 하드코딩

jwt:
  secret: my_jwt_secret_key_1234
JWT 라이브러리는 Base64 인코딩된 Secret을 기대하지만,
해당 문자열에는 _ 등 Base64 규칙에 어긋나는 문자가 포함되어 있었고,
이로 인해 디코딩 단계에서 예외가 발생했다.

4. 잘못된 설정 방식의 문제점
Secret을 설정 파일에 직접 노출

환경별(local/dev/prod) 분리 불가능

보안 사고 시 즉각적인 키 교체 어려움

실무 환경에 부적합한 구조

단순히 에러를 고치는 문제가 아니라,
보안 설정 전략 자체를 수정해야 하는 상황이었다.

5. 해결 방법
① Base64 인코딩된 Secret 사용
JWT Secret은 충분한 길이의 Base64 문자열로 생성한다.

openssl rand -base64 64
② application.yaml에서 secret 제거
jwt:
  access-token-expiration: 3600000
③ 환경변수로 secret 주입
export JWT_SECRET=생성한_Base64_문자열
④ JwtTokenProvider 수정
public JwtTokenProvider(
        @Value("${jwt.secret}") String secretKey,
        @Value("${jwt.access-token-expiration}") long accessTokenExpirationMillis,
        UserDetailsService userDetailsService
) {
    byte[] keyBytes = Decoders.BASE64.decode(secretKey);
    this.key = Keys.hmacShaKeyFor(keyBytes);
    this.accessTokenExpirationMillis = accessTokenExpirationMillis;
    this.userDetailsService = userDetailsService;
}
6. 적용 후 구조 정리
민감 정보는 환경변수로만 관리

Git 저장소에는 절대 노출하지 않음

로컬 / 배포 환경 모두 동일한 방식 적용

JWT Key 생성 로직 명확화

7. 정리 & 배운 점
JWT 에러의 원인은 코드보다 설정 방식인 경우가 많다

Secret은 반드시 Base64 인코딩 + 충분한 길이를 보장해야 한다

application.yaml은 설정 파일이지 보안 저장소가 아니다

보안 설정은 “동작하면 끝”이 아니라 운영까지 고려해야 한다
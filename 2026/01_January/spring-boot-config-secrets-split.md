# Spring Boot에서 application.yaml 보안 설정 분리하기 (local profile + gitignore)

## 배경
프로젝트를 진행하면서 `application.yaml`에 아래와 같은 민감 정보가 포함될 수 있다.

- DB 접속 정보 (특히 `password`)
- JWT 서명 키 (`jwt.secret`)
- 외부 API Key / Secret
- 운영 환경의 Redis/DB 호스트 정보 등

이 값들이 그대로 GitHub에 커밋되면 **비밀정보 유출**로 이어질 수 있으므로, 환경별 설정을 분리하고 Git 추적에서 제외하는 전략이 필요했다.

---

## 목표
1. GitHub에 올라가도 되는 **공용 설정**만 `application.yaml`에 둔다.
2. 로컬에서만 쓰는 **민감/환경 의존 설정**은 `application-local.yaml`로 분리한다.
3. `application-local.yaml`은 `.gitignore`로 Git 추적에서 제외한다.
4. 로컬 실행 시 profile을 `local`로 지정해 `application-local.yaml`이 적용되게 한다.

---

## 1) GitHub에 올려도 되는 설정 vs 올리면 안 되는 설정

### ✅ GitHub에 올려도 되는 것 (공용)
- `server.port`
- `spring.application.name`
- JPA 공통 옵션 (format_sql, batch_fetch_size 등)
- 로깅 레벨 (SQL debug 등)
- profile 그룹/활성화 구조 (단, 값 자체는 민감하지 않아야 함)

### ❌ GitHub에 올리면 안 되는 것 (민감 정보)
- `spring.datasource.password`
- `jwt.secret`
- 외부 API Key (YouTube, Kakao 등)
- 운영 DB/Redis의 실제 host 정보(환경에 따라 달라지는 값)

---

## 2) 파일 분리 구조

### 권장 파일 구성
- `application.yaml` : 공용/안전 설정만 포함 (Git 추적)
- `application-local.yaml` : 로컬 전용 민감 설정 (Git 추적 제외)
- (옵션) `application-prod.yaml` : 운영 설정 (배포 파이프라인에서 환경변수로 주입 권장)

---

## 3) 예시 설정

### 3-1. application.yaml (공용)
> **민감 정보는 제거**하고, 공용 설정만 남긴다.

```yaml
server:
  port: 8080

spring:
  application:
    name: backend

  jpa:
    hibernate:
      ddl-auto: update   # 개발 중 임시용. 운영은 validate 권장
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 100
    open-in-view: false

logging:
  level:
    org.hibernate.SQL: debug
    org.hibernate.type: trace

---
spring:
  config:
    activate:
      on-profile: local
위 on-profile: local 블록은 "local 프로필을 사용할 때 추가 설정을 읽겠다"는 트리거로 활용한다.
실제 민감 값은 application-local.yaml에 둔다.

3-2. application-local.yaml (로컬 전용, Git에 올리지 않음)
spring:
  datasource:
    url: jdbc:mariadb://localhost:3306/playlist_db?serverTimezone=Asia/Seoul&characterEncoding=utf8
    username: myapp_user
    password: myapp1234
    driver-class-name: org.mariadb.jdbc.Driver

  data:
    redis:
      host: localhost
      port: 6379

jwt:
  secret: "LOCAL_DEV_ONLY_CHANGE_ME"
4) .gitignore 설정 (핵심)
프로젝트 루트의 .gitignore에 아래를 추가한다.

# local config (secrets)
src/main/resources/application-local.yaml
src/main/resources/application.local.yaml
파일명 컨벤션은 한 가지로 통일하는 것을 권장한다.
예: application-local.yaml로 고정

5) 이미 Git에 올라간 경우의 대응
.gitignore에 추가해도 이미 커밋된 파일은 추적이 계속된다.
따라서 캐시에서 제거해야 한다.

git rm --cached src/main/resources/application-local.yaml
git commit -m "chore: stop tracking local config"
git push
6) 로컬에서 local profile 적용 방법
방법 A) IntelliJ Run Configuration
Run/Debug Configurations 열기

Environment variables 또는 Active profiles에 local 지정

Spring Boot 3 기준 권장:

SPRING_PROFILES_ACTIVE=local

방법 B) 실행 명령으로 지정
./gradlew bootRun --args='--spring.profiles.active=local'
7) 운영 환경에서의 권장 방식 (추가)
운영(prod)에서는 application-prod.yaml에 비밀번호를 쓰기보다,
CI/CD에서 환경변수로 주입하는 방식이 안전하다.

예:

SPRING_DATASOURCE_URL

SPRING_DATASOURCE_USERNAME

SPRING_DATASOURCE_PASSWORD

JWT_SECRET

8) 오늘 적용 결과
application.yaml은 공용 설정만 남겨 GitHub에 커밋 가능하게 정리

로컬 민감 설정은 application-local.yaml로 이동

.gitignore로 로컬 설정 파일이 커밋되지 않도록 차단

IntelliJ 실행 시 SPRING_PROFILES_ACTIVE=local로 로컬 실행 환경 구성
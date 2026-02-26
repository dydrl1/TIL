# [Troubleshooting] 스프링 부트 설정 중복으로 인한 빈 의존성 주입 실패 해결

## 1. 문제 상황 (Situation)
- 프로젝트 실행 시 `UnsatisfiedDependencyException` 발생.
- `AuthService` -> `SecurityConfig` -> `JwtUtil` 순으로 의존성이 엮여 있는 구조에서 최하단 빈인 `JwtUtil` 생성이 실패하며 애플리케이션 구동이 중단됨.

## 2. 에러 로그 (Task)
```text
Caused by: org.springframework.beans.factory.UnsatisfiedDependencyException: 
Error creating bean with name 'authService' ... 
Unsatisfied dependency expressed through constructor parameter 1: 
Error creating bean with name 'securityConfig' ... 
Error creating bean with name 'jwtUtil' ... 
Unexpected exception during bean creation


3. 원인 분석 (Action)
가설 설정 및 검증
가설 1: JwtUtil 코드 자체의 로직 오류일 것이다. -> 코드상 특이점 없음.

가설 2: @Value를 통해 주입받는 환경 변수가 누락되었을 것이다. -> application.yml 확인 중 설정 중복(Override) 발견.

근본 원인 (Root Cause)
application.yml 파일 내에 spring: 섹션이 두 번 정의되어 있었음.

스프링 부트는 YAML을 읽을 때 같은 키값이 존재할 경우 **나중에 정의된 블록으로 덮어쓰기(Override)**를 수행함.

이로 인해 상단에 정의했던 profiles.active, jpa, datasource 등의 핵심 설정이 모두 증발하여 JwtUtil이 필요한 설정값을 읽지 못해 예외를 던진 것임.

4. 해결 방법 (Result)
설정 구조 단일화: 중복된 spring: 섹션을 하나로 통합하여 설정 유실 방지.

관심사 분리: 보안이 필요한 민감 정보(JWT Secret, API Key)는 application-local.yml로 분리하고, 공통 구조는 application.yml에 유지.

정상 작동: 설정 파일 로드 순서가 정립되면서 빈 의존성 주입이 정상적으로 완료되어 서버 구동 성공.

5. 회고 및 깨달은 점 (Learned)
스프링 부트의 **설정 파일 로딩 메커니즘(Override 우선순위)**에 대해 깊이 이해하게 됨.

복잡한 UnsatisfiedDependencyException이 발생했을 때, 스택 트레이스의 최하단을 추적하여 **근본 원인(Root Cause)**을 찾아내는 디버깅 능력을 배양함.

협업과 보안을 위해 로컬 설정 파일을 분리하는 표준적인 방식의 중요성을 체감함.
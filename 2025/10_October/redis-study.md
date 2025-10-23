# 🧠 Redis Study Log - October 2025

## 📅 날짜
2025년 10월 23일

## 🧩 학습 주제
- Redis 기본 개념 복습  
- 캐싱 구조 이해  
- TTL / Expire 실습  
- Spring Boot 연동  

---

## 🧠 Redis 핵심 개념
- Redis는 메모리 기반의 Key-Value 데이터 저장소
- String, List, Set, Sorted Set, Hash 등 다양한 자료형 지원
- TTL(Time To Live)을 이용한 캐시 만료 관리 가능
- 싱글 스레드 기반이지만 I/O 멀티플렉싱으로 고성능 처리 가능

---

## ⚙️ 실습 명령어
```bash
# Redis 서버 실행
docker run -d -p 6379:6379 --name redis redis:latest

# 데이터 저장
SET user:1 "GodLife"

# 데이터 조회
GET user:1

# TTL 설정
EXPIRE user:1 60

🧰 Spring Boot 연동 정리

@Configuration
@EnableCaching
public class RedisConfig {
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory("localhost", 6379);
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        return template;
    }
}

💡 느낀점

Redis 캐싱은 단순히 속도 향상 도구가 아니라 데이터 흐름의 효율성을 설계하는 핵심 기술

TTL을 활용한 임시 저장 구조 설계의 중요성 깨달음

갓생로그 프로젝트에서 Redis 장애 대응 전략 복습 필요
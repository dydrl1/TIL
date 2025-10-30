# 🧩 Redis 연결 실패 (RedisConnectionFailureException)

## 📅 발생일시
2025-10-30

## 🧱 환경
- **Backend:** Spring Boot 3.x  
- **Database:** MariaDB  
- **Cache:** Redis (Docker)  
- **Infra:** Docker Compose / Nginx Reverse Proxy  
- **Branch:** `dev`

---

## ⚠️ 현상
서버 부팅 시 다음과 같은 예외가 발생하며, 애플리케이션이 Redis와 연결되지 않음.

org.springframework.data.redis.RedisConnectionFailureException:
Unable to connect to Redis; nested exception is io.lettuce.core.RedisConnectionException:
Unable to connect to 127.0.0.1:6379

yaml
코드 복사

- Redis 캐시 기능이 모두 동작하지 않음  
- 관리자 페이지에서 캐시 데이터가 로드되지 않음  
- DB 데이터는 정상 조회되지만 응답 속도가 느려짐  
- `docker-compose logs`에서 Redis 로그가 비어 있음

---

## 🔍 원인 분석
1. `spring.redis.host=localhost` 로 설정되어 있었음  
   → Docker Compose 환경에서는 **컨테이너 이름(`redis`)** 으로 지정해야 함.  
2. `depends_on` 옵션 누락으로 인해 **백엔드 컨테이너가 Redis보다 먼저 실행됨**.  
3. `redis` 서비스가 **같은 네트워크에 포함되지 않음** (`bridge` 누락).

---

## 🧠 해결 방법

### ✅ 1. `application.yml` 수정
```yaml
spring:
  redis:
    host: redis
    port: 6379


✅ 2. docker-compose.yml 수정
yaml
코드 복사
services:
  backend:
    image: godlife-backend
    depends_on:
      - redis
    networks:
      - godlife-network

  redis:
    image: redis:7
    container_name: redis
    ports:
      - "6379:6379"
    networks:
      - godlife-network

networks:
  godlife-network:
    driver: bridge


✅ 3. Redis 연결 테스트
bash
코드 복사
docker exec -it redis redis-cli ping
# → PONG


✅ 4. Spring Boot 로그 확인
yaml
코드 복사
o.s.d.r.c.RedisConnectionUtils : Redis Connection established successfully
🧾 참고 로그
pgsql
코드 복사
Caused by: io.lettuce.core.RedisConnectionException: Unable to connect to 127.0.0.1:6379
	at io.lettuce.core.RedisConnectionException.create(RedisConnectionException.java:78)
	at io.lettuce.core.RedisChannelHandler.lambda$initializeChannelAsync0$4(RedisChannelHandler.java:504)


💡 개선 계획
Docker Compose에 depends_on 명시하여 Redis 먼저 구동

.env 파일로 Redis 환경 변수 통합 관리

Redis 장애 시 DB fallback 로직 추가

healthcheck 설정 추가로 서비스 자동 감시


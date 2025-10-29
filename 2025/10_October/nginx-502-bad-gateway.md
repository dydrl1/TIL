cat > 2025-10-29-nginx-502-bad-gateway.md <<'EOF'
# 🐳 Nginx 502 Bad Gateway 원인 분석 및 해결

**작성일:** 2025-10-29  
**분류:** ☁️ DevOps / Docker / Nginx  
**태그:** #Nginx #Docker #ReverseProxy #GodLifeProject

---

## 🧩 문제 상황

EC2 환경에서 **Nginx → Backend(Spring Boot)** 리버스 프록시 구성 시  
웹 접속 시 `502 Bad Gateway` 에러 발생.

- 프론트엔드는 정상 작동  
- 백엔드 API 호출 시 실패 (nginx error log에서 upstream 연결 실패 확인)

---

## 🔍 원인 분석

1. `nginx.conf` 에서 `proxy_pass` 대상이 `localhost:8080` 으로 설정됨  
   → Docker 내부에서는 각 컨테이너가 **격리된 네트워크** 환경에 있기 때문에 `localhost` 접근 불가  
2. `docker-compose.yml` 에 **공통 네트워크 미설정**  
   → 컨테이너 간 서비스명(DNS) 인식이 불가능하여 연결 실패

---

## 🛠 해결 방법

### ✅ 1. Nginx 설정 수정
```nginx
# Before
location /api/ {
    proxy_pass http://localhost:8080; # ❌ 컨테이너 내부에서는 localhost 불가
}

# After
upstream backend_upstream {
    server backend:8080;  # ✅ backend 컨테이너명으로 접근
    keepalive 32;
}

location /api/ {
    proxy_set_header Host $host;
    proxy_pass http://backend_upstream;
}



 2. Docker Compose 네트워크 명시
networks:
  godlife-network:
    driver: bridge

services:
  backend:
    image: godlife-backend
    expose:
      - "8080"
    networks:
      - godlife-network

  frontend:
    image: godlife-frontend
    networks:
      - godlife-network

  nginx:
    image: nginx:stable
    ports:
      - "80:80"
    depends_on:
      - backend
      - frontend
    networks:
      - godlife-network



 3. 컨테이너 재배포
docker-compose down
docker-compose up -d --build
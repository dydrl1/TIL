!

📌 HTTP → HTTPS 자동 리디렉션 동작 방식 정리
🔎 1. 왜 자동 리디렉션이 필요한가?

HTTP(http://)는 암호화되지 않은 프로토콜이고,
HTTPS(https://)는 SSL/TLS 암호화를 사용한 보안 프로토콜이다.

보안 강화를 위해 대부분의 웹사이트는 실제 서비스에서 HTTPS를 사용하지만,
사용자가 HTTP 주소를 입력해도 자동으로 HTTPS로 이동해야 한다.

이때 사용되는 것이 HTTP → HTTPS 자동 리디렉션이다.

🔄 2. 리디렉션이란 무엇인가?

리디렉션(Redirect)이란
서버가 클라이언트에게 “다른 주소로 가라”고 알려주는 동작이다.

HTTP 상태 코드:

코드	의미	설명
301 Moved Permanently	영구 이동	HTTPS로 항상 이동하도록 고정
302 Found (or 307)	임시 이동	상황에 따라 변할 수 있는 주소로 이동

일반적으로 HTTP → HTTPS 리디렉션은 301을 사용한다.

⚙️ 3. HTTP → HTTPS 리디렉션이 실제로 어떻게 동작하는가?
📌 흐름 요약

사용자가 http://example.com에 접속

서버(웹 서버 또는 애플리케이션)가 요청을 받음

서버가 301 리디렉션 응답을 보냄

브라우저가 서버가 알려준 https://example.com으로 재요청

HTTPS로 암호화된 접속이 이루어짐

📉 흐름도
브라우저 → http://example.com 요청
서버 → 301 Moved Permanently (Location: https://example.com)
브라우저 → https://example.com 재요청
서버 → HTTPS 응답

🛠 4. 실제 서버 설정에서의 리디렉션 예시
✔ 4-1. Nginx 설정
server {
    listen 80;
    server_name example.com;

    # 모든 http 요청을 https로 리디렉션
    return 301 https://$host$request_uri;
}

✔ 4-2. Apache 설정
<VirtualHost *:80>
    ServerName example.com
    Redirect permanent / https://example.com/
</VirtualHost>

✔ 4-3. Spring Boot (Java) 설정
① WebSecurity 설정을 통한 HTTPS 강제
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
        .requiresChannel()
        .anyRequest()
        .requiresSecure();
}

② Tomcat Connector로 HTTP → HTTPS 리디렉션(스프링부트)
@Bean
public TomcatServletWebServerFactory servletContainer() {
    TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory();

    Connector connector = new Connector(TomcatServletWebServerFactory.DEFAULT_PROTOCOL);
    connector.setScheme("http");
    connector.setPort(8080);
    connector.setSecure(false);
    connector.setRedirectPort(8443);  // HTTPS 포트로 리디렉션
    tomcat.addAdditionalTomcatConnectors(connector);

    return tomcat;
}

🔐 5. 리디렉션과 HSTS의 차이
항목	Redirect	HSTS
동작 위치	서버가 HTTP 요청을 받고 처리	브라우저가 자체적으로 HTTPS 강제
최초 요청	HTTP로 올 수 있음	HTTP 접속 자체를 차단
보안성	중간자 공격(MiTM)에 취약할 수 있음	더 안전함
적용 방식	서버 301 응답	HTTP response header
HSTS 헤더 예시
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload


이 헤더를 통해 브라우저는 앞으로 1년 동안 HTTP 요청 자체를 시도하지 않고 HTTPS만 요청한다.

🧠 6. 정리
내용	설명
HTTP → HTTPS 자동 리디렉션	서버가 브라우저에게 HTTPS 주소로 이동하라고 알려주는 방식
주로 사용하는 코드	301 Moved Permanently
동작 흐름	http 요청 → 301 응답 → https 재요청
서버 설정	Nginx/Apache/Spring 등에서 수행 가능
보안을 더 강화하려면	HSTS 사용
📚 참고 자료

MDN Web Docs – Redirection

Nginx Documentation – Return Directive

Spring Security Docs – requiresSecure 사용
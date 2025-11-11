### 📘 Spring Boot — @RequestParam vs @RequestBody

#### 🔹 개요  
Spring Boot에서 클라이언트로부터 데이터를 받을 때는 주로  
`@RequestParam`, `@PathVariable`, `@RequestBody` 를 사용한다.  
이 중 `@RequestParam`과 `@RequestBody`는 자주 혼동되므로 차이를 정리했다.

---

### 💻 1. @RequestParam — 쿼리스트링이나 form-data용

#### ✔️ 사용 예시
```java
@GetMapping("/user")
public String getUser(@RequestParam String name, @RequestParam int age) {
    return name + "은(는) " + age + "살입니다.";
}

✔️ 요청 예시
pgsql
코드 복사
GET /user?name=만두&age=25

✔️ 특징
주로 GET 요청이나 **폼(form)**에서 데이터를 받을 때 사용

key=value 형태로 전달

단일 값 혹은 간단한 파라미터를 받을 때 적합

자동 형변환 지원 (String → int 등)

💻 2. @RequestBody — JSON 형태의 데이터용

✔️ 사용 예시
java
코드 복사
@PostMapping("/user")
public String createUser(@RequestBody UserDTO user) {
    return user.getName() + " 등록 완료!";
}

✔️ 요청 예시 (JSON)
json
코드 복사
POST /user
{
  "name": "만두",
  "age": 25
}

✔️ 특징
주로 POST, PUT, PATCH 요청에서 사용

Body에 담긴 JSON 데이터를 객체로 매핑

내부적으로 HttpMessageConverter가 작동하여 JSON ↔ 객체 변환 수행

복잡한 구조의 데이터를 받을 때 유용

⚖️ 비교 요약표
구분	@RequestParam	@RequestBody
데이터 위치	URL 쿼리스트링, form-data	HTTP Body
주요 요청 방식	GET, POST(form)	POST, PUT, PATCH
데이터 포맷	key=value	JSON, XML 등
매핑 대상	단일 값, 간단한 파라미터	DTO, 복합 객체
예시	/user?name=만두&age=25	{ "name": "만두", "age": 25 }

💡 TIL 요약
오늘은 @RequestParam과 @RequestBody의 차이를 실습하며 이해했다.
단순한 데이터는 파라미터로, 복잡한 JSON 객체는 RequestBody로 받는 것이 명확하다.
실제 프로젝트에서는 GET 요청에는 @RequestParam,
POST/PUT 요청에는 @RequestBody를 주로 사용했다.

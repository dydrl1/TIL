# Controller–Service–Repository 책임 분리 원칙 (Spring Boot)

취준/실무에서 가장 자주 검증되는 역량 중 하나는 “계층형 아키텍처에서 각 레이어의 책임을 정확히 분리했는가”이다.  
Controller–Service–Repository를 분리하는 목적은 **변경에 강하고(유지보수), 테스트 가능하며(품질), 도메인 로직이 응집된(확장성) 구조**를 만드는 데 있다.

---

## 1. 각 레이어의 역할 정의

### 1) Controller (Presentation Layer)
**역할**
- HTTP 요청을 받고(입력), 검증하고
- Service를 호출하여 결과를 받고(처리 위임)
- HTTP 응답 형태로 변환하여 반환(출력)

**Controller가 해야 하는 것**
- Request DTO 바인딩, 기본 유효성 검증(`@Valid`)
- 인증/인가 정보 전달(예: `@AuthenticationPrincipal`)
- Service 호출 및 Response DTO 반환
- 상태코드, 헤더, 응답 포맷 결정

**Controller가 하면 안 되는 것**
- 트랜잭션 처리
- 비즈니스 규칙(도메인 로직) 구현
- DB 조회/저장 직접 수행(JPA Repository 직접 호출 남발)
- 복잡한 예외 분기(전역 예외 처리로 이동)

---

### 2) Service (Application/Domain Layer)
**역할**
- 유스케이스(Use Case)를 수행하는 **비즈니스 로직의 중심**
- 트랜잭션 경계 설정(`@Transactional`)
- 도메인 규칙 검증 및 상태 변경
- 여러 Repository/외부 시스템을 조합하여 기능 완성

**Service가 해야 하는 것**
- “무엇을 할지”에 대한 흐름(업무 시나리오) 구현
- 트랜잭션 관리
- 권한 검증(예: “이 Playlist의 소유자인가?”)
- 도메인 객체 생성/변경(엔티티 상태 변경)
- 예외를 비즈니스 의미로 변환(커스텀 예외)

**Service가 하면 안 되는 것**
- HTTP 관련 세부 사항(`HttpServletRequest`, `ResponseEntity` 등) 다루기
- View/JSON 포맷 결정
- DB 쿼리 최적화 디테일을 지나치게 품기(Repository로 위임)

---

### 3) Repository (Persistence Layer)
**역할**
- 영속성(저장/조회) 책임
- 도메인 객체를 DB와 매핑하여 CRUD 제공

**Repository가 해야 하는 것**
- 조회/저장/삭제 등 데이터 접근
- 쿼리 메서드 / JPQL / QueryDSL 등으로 조회 로직 캡슐화
- 성능을 고려한 조회 전략 제공(필요시 fetch join 등)

**Repository가 하면 안 되는 것**
- 비즈니스 규칙 판단(“공개 가능한 상태인지” 같은 것)
- 트랜잭션 흐름 제어(서비스가 담당)
- DTO 변환 로직(특별한 조회 DTO는 예외적으로 허용 가능)

---

## 2. 책임 분리 체크리스트 (면접에서 바로 쓰는 기준)

### Controller 체크
- [ ] Controller가 repository를 직접 호출하지 않는다.
- [ ] Controller는 DTO ↔ Service 호출 ↔ DTO 반환만 수행한다.
- [ ] 검증은 `@Valid` + Validator로 처리하고, 비즈니스 검증은 Service로 간다.

### Service 체크
- [ ] 비즈니스 규칙이 Service에 응집되어 있다.
- [ ] 트랜잭션 경계가 Service에 있다.
- [ ] “권한/소유자 검증”은 Service에서 수행한다.
- [ ] 여러 repository/외부 API 조합은 Service에서 오케스트레이션한다.

### Repository 체크
- [ ] 데이터 접근 로직이 캡슐화되어 있고, 서비스는 “무슨 데이터를 가져올지”만 요구한다.
- [ ] 조회 조건이 복잡해질수록 Repository 메서드로 추상화한다.

---

## 3. 잘못된 예시 vs 개선 예시

### ❌ Bad: Controller에 비즈니스 로직 + Repository 직접 호출
```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/playlists")
public class PlaylistController {

    private final PlaylistRepository playlistRepository;

    @PostMapping("/{id}/publish")
    public ResponseEntity<?> publish(@PathVariable Long id) {
        Playlist playlist = playlistRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("not found"));

        // 비즈니스 규칙이 Controller에 존재 (문제)
        if (playlist.getTracks().isEmpty()) {
            return ResponseEntity.badRequest().body("트랙이 없으면 공개할 수 없음");
        }

        playlist.publish(); // 상태 변경
        playlistRepository.save(playlist);

        return ResponseEntity.ok().build();
    }
}
문제점

컨트롤러가 규칙을 알아야 해서 변경에 취약

테스트가 어려움(HTTP 레이어 테스트로 비즈니스까지 같이 테스트)

예외/응답 포맷이 분산됨

✅ Good: Controller는 요청/응답, Service는 규칙/트랜잭션
Controller

java
코드 복사
@RestController
@RequiredArgsConstructor
@RequestMapping("/playlists")
public class PlaylistController {

    private final PlaylistService playlistService;

    @PostMapping("/{id}/publish")
    public ResponseEntity<Void> publish(@PathVariable Long id) {
        playlistService.publish(id);
        return ResponseEntity.ok().build();
    }
}
Service

java
코드 복사
@Service
@RequiredArgsConstructor
public class PlaylistService {

    private final PlaylistRepository playlistRepository;

    @Transactional
    public void publish(Long playlistId) {
        Playlist playlist = playlistRepository.findById(playlistId)
                .orElseThrow(() -> new NotFoundException("PLAYLIST_NOT_FOUND"));

        // 비즈니스 규칙은 Service에 위치
        if (playlist.getPlaylistTracks().isEmpty()) {
            throw new InvalidRequestException("PLAYLIST_EMPTY_CANNOT_PUBLISH");
        }

        playlist.publish(); // 엔티티 상태 변경
    }
}
장점

변경(규칙 수정)이 Service에 집중됨

Controller는 얇아지고 API 계층이 단순해짐

Service 단위 테스트가 쉬워짐

전역 예외 처리(@ControllerAdvice)와 결합하기 좋음

4. “얇은 Controller”를 유지하는 팁
DTO 변환은 Controller/Service 경계에서 끝낸다

Controller: Request DTO → Service 파라미터

Service: 도메인 처리 후 → Response DTO로 변환(혹은 Facade/Assembler 사용)

예외는 커스텀 예외 + 전역 처리로 일관화

Controller에서 try-catch로 응답 만들지 않기

@ControllerAdvice에서 ErrorResponse 표준화

권한/소유자 체크는 Service로

“내가 만든 playlist인지” 같은 검증은 HTTP와 무관한 비즈니스 규칙
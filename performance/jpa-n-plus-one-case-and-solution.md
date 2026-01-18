# JPA N+1 문제: 실제 사례와 해결 과정 (운영/성능 관점)

JPA를 사용하면 연관관계 조회에서 의도치 않게 쿼리가 폭증하는 **N+1 문제**가 자주 발생한다.  
개발 환경에서는 데이터가 적어 티가 안 나지만, 운영에서는 **응답 지연 + DB 부하 급증 + 커넥션 고갈**로 이어질 수 있어 반드시 원인/증상/해결책을 정리해 두는 것이 좋다.

---

## 1. N+1 문제란?

- “1번의 쿼리로 N개의 엔티티를 가져왔는데”
- 각 엔티티의 연관 엔티티를 접근하는 순간
- **추가 쿼리가 N번 더 발생**하는 문제

즉,
- 최초 조회 1번 + 연관 조회 N번 → 총 N+1번

---

## 2. 실제 사례 (Playlist 상세 조회에 Track 포함)

### 요구사항
- `GET /playlists/{id}` 응답에
- 플레이리스트 정보 + 플레이리스트에 담긴 트랙 목록(순서 포함)을 내려준다.

구조(예시)
- `Playlist (1) - (N) PlaylistTrack - (N) Track`
- PlaylistTrack에 `trackOrder` 같은 추가 컬럼이 존재

---

## 3. 문제 발생 코드 (증상 재현)

### 기본 엔티티 매핑 (LAZY)
```java
@Entity
public class Playlist {
    @OneToMany(mappedBy = "playlist", fetch = FetchType.LAZY)
    private List<PlaylistTrack> playlistTracks = new ArrayList<>();
}

@Entity
public class PlaylistTrack {
    @ManyToOne(fetch = FetchType.LAZY)
    private Track track;

    private Integer trackOrder;
}
문제를 만든 조회/변환 흐름
public PlaylistDetailResponse getDetail(Long id) {
    Playlist playlist = playlistRepository.findById(id)
            .orElseThrow();

    // DTO 변환 과정에서 연관 엔티티 접근
    List<TrackItem> items = playlist.getPlaylistTracks().stream()
            .map(pt -> new TrackItem(
                    pt.getTrack().getId(),       // 여기서 Track 접근
                    pt.getTrack().getTitle(),
                    pt.getTrackOrder()
            ))
            .toList();

    return PlaylistDetailResponse.of(playlist, items);
}
실제로 터지는 쿼리 패턴(예시)
select * from playlist where id = ?; (1)

select * from playlist_track where playlist_id = ?; (1)

select * from track where id = ?; (N) ← 여기서 폭발

트랙이 50개면 Track 조회가 50번 더 나간다.

4. 운영에서의 영향
트래픽이 늘어날수록 쿼리 수가 선형 증가

DB CPU/IO 사용량 증가

커넥션 풀 점유 증가 → 다른 요청 대기/타임아웃 발생

API 응답 시간 급증(체감 성능 저하)

운영 기준으로는 “기능이 된다”보다
쿼리 횟수/지연/부하를 통제할 수 있느냐가 훨씬 중요하다.

5. 진단 방법 (실무에서 바로 쓰는 체크)
SQL 로그 확인

Hibernate SQL 로그를 켜고 API 1번 호출했을 때

동일 테이블에 대한 select가 반복되면 의심

트랜잭션 범위 밖에서 LAZY 접근 시도

LazyInitializationException이 나면

“현재 연관 조회가 런타임에 터지고 있다”는 신호

“DTO 변환 과정”을 의심

stream/map에서 연관 접근이 있는지 확인

6. 해결 과정 1: Fetch Join으로 한 번에 당겨오기 (권장)
핵심 아이디어
Playlist를 조회할 때

PlaylistTrack과 Track까지 같이 가져오면

Track을 접근해도 추가 쿼리가 발생하지 않는다.

Repository에 fetch join 쿼리 추가
public interface PlaylistRepository extends JpaRepository<Playlist, Long> {

    @Query("""
        select distinct p
        from Playlist p
        left join fetch p.playlistTracks pt
        left join fetch pt.track t
        where p.id = :id
    """)
    Optional<Playlist> findDetailById(@Param("id") Long id);
}
distinct를 붙여 중복 row로 인해 Playlist가 중복 생성되는 것을 방지

left join fetch로 연관 엔티티를 한 번에 로딩

Service에서 fetch join 메서드 사용
@Transactional(readOnly = true)
public PlaylistDetailResponse getDetail(Long id) {
    Playlist playlist = playlistRepository.findDetailById(id)
            .orElseThrow();

    // DTO 변환 시 추가 쿼리 없음
    ...
}
효과

쿼리 수: (Playlist + PlaylistTrack + Track) → 사실상 1번(조인)으로 축소

응답 시간 안정화

DB 부하 감소

7. 해결 과정 2: Batch Size로 N을 줄이기 (대안)
Fetch join이 항상 정답은 아니다.

페이징이 필요하거나

컬렉션 fetch join이 복잡해질 때

그럴 땐 batch fetch를 통해
N번의 쿼리를 “묶어서” 줄일 수 있다.

설정 예시
application.yml

spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
효과

Track 조회가 N번 나가던 것이

IN 쿼리로 여러 개가 묶여 호출됨

완전한 제거는 아니지만 실무에서 체감 성능 크게 개선

8. 해결 과정 3: DTO 직접 조회 (쿼리 최적화의 끝판왕)
특정 화면/API에서 필요한 값만 명확하다면

엔티티 그래프를 억지로 구성하기보다

조회 전용 DTO를 JPQL/QueryDSL로 바로 뽑는 방식이 가장 효율적일 때가 있다.

단, 코드 복잡도/재사용성 트레이드오프가 있어

“핵심 API/핵심 조회”에 제한적으로 적용하는 편이 일반적이다.

9. 내가 선택한 기준 (정리)
단건 상세 조회: fetch join 우선

목록 + 페이징: batch size / 쿼리 분리 / DTO 조회 고려

운영 성능이 중요: DTO 직접 조회도 옵션


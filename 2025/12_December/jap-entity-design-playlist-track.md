JPA에서 중간 테이블(연결 엔티티)을 사용하는 이유
Playlist ↔ Track 설계 정리
1. 문제 상황

플레이리스트 서비스에서 다음과 같은 요구사항이 있었다.

하나의 플레이리스트에는 여러 곡이 들어갈 수 있다.

하나의 곡은 여러 플레이리스트에 포함될 수 있다.

플레이리스트 안에서 곡의 순서(trackOrder) 를 관리해야 한다.

같은 플레이리스트에 동일한 곡이 중복으로 들어가면 안 된다.

관계만 보면 Playlist ↔ Track 은 다대다(ManyToMany) 관계처럼 보인다.

2. ManyToMany를 사용하지 않은 이유

JPA에서 @ManyToMany 는 다음과 같은 한계를 가진다.

❌ 실무에서의 문제점

중간 테이블에 추가 컬럼을 둘 수 없음

순서, 등록일, 메타 정보 관리 불가능

비즈니스 로직이 커질수록 확장 불가

중간 테이블을 직접 제어하기 어려움

이번 설계에서는
곡의 순서(trackOrder) 라는 명확한 추가 정보가 필요했기 때문에
@ManyToMany 는 적합하지 않았다.

3. 해결 방법: 연결 엔티티(중간 테이블) 도입

다대다 관계를 두 개의 일대다 / 다대일 관계로 풀어냈다.

설계 구조
Playlist 1 ─── N PlaylistTrack N ─── 1 Track


PlaylistTrack 이 연결 엔티티

중간 테이블이지만, JPA에서는 독립적인 엔티티로 관리

4. PlaylistTrack 엔티티 설계
@Entity
@Table(
    name = "PLAYLIST_TRACK",
    uniqueConstraints = {
        @UniqueConstraint(
            name = "uk_playlist_track",
            columnNames = {"playlist_id", "track_id"}
        )
    }
)
public class PlaylistTrack {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "playlist_id")
    private Playlist playlist;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "track_id")
    private Track track;

    @Column(nullable = false)
    private Integer trackOrder;
}

핵심 포인트

Playlist 와 Track 을 각각 @ManyToOne 으로 연결

trackOrder 로 플레이리스트 내 곡 순서 관리

(playlist_id + track_id) 조합에 유니크 제약조건 추가

동일 플레이리스트에 같은 곡 중복 방지

5. Playlist 엔티티 설계 (읽기 전용 연관관계)
@OneToMany(mappedBy = "playlist", cascade = CascadeType.ALL, orphanRemoval = true)
private List<PlaylistTrack> playlistTracks = new ArrayList<>();

설계 의도

연관관계의 주인은 PlaylistTrack

Playlist 쪽에서는 조회용(읽기 전용) 으로만 사용

실제 추가/삭제/순서 변경은 PlaylistTrack 기준으로 처리

6. Track 엔티티는 단순 엔티티로 유지
@Entity
@Table(name = "TRACK")
public class Track {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 200)
    private String title;

    @Column(nullable = false, length = 100)
    private String artist;

    private Integer durationSec;
}


Track 자체는 순서나 소속 개념을 알 필요가 없음

관계 책임은 연결 엔티티가 담당

7. 이 설계의 장점
✅ 확장성

추후 addedAt, addedBy, playCount 등 컬럼 추가 가능

✅ 명확한 책임 분리

Playlist: 플레이리스트 자체의 정보

Track: 곡 자체의 정보

PlaylistTrack: “이 플레이리스트에 이 곡이 어떤 순서로 들어있는가”

✅ 실무 친화적 설계

정렬, 삭제, 순서 변경 로직 구현이 쉬움

Repository 분리 (PlaylistTrackRepository) 가능

✅ JPA 권장 방식

공식적으로 권장되는 다대다 해소 패턴

복잡한 비즈니스 로직에 대응 가능

8. 정리

다대다 관계에서 비즈니스 속성이 하나라도 필요하다면
@ManyToMany 대신 연결 엔티티를 사용하는 것이 정석이다.

이번 Playlist ↔ Track 설계에서는
곡의 순서 관리라는 요구사항 때문에
PlaylistTrack 이라는 연결 엔티티를 도입했고,
그 결과 확장성과 유지보수성이 높은 구조를 만들 수 있었다.

# [JPA] 플레이리스트 - 트랙 엔티티 설계 & 다대다 관계 풀기

---

## 0. 작업 시작 & 브랜치 상태 확인

플레이리스트 기능 구현을 진행하면서  
JPA 엔티티 설계부터 정리하고 TIL로 남기기로 했다.

```bash
git status
git branch
1. 오늘 한 작업 요약
Playlist, Track, PlaylistTrack 엔티티 설계

다대다 관계를 중간 엔티티로 분리

플레이리스트 내 곡 순서를 위한 trackOrder 필드 추가

동일 플레이리스트 내 동일 트랙 중복 방지 제약 설정

작업 내용을 정리해두지 않으면
나중에 “왜 이렇게 설계했더라?”라는 고민을 다시 하게 될 것 같아 TIL로 기록.

2. 요구사항 정리
플레이리스트 도메인의 요구사항은 다음과 같았다.

하나의 플레이리스트에는 여러 곡이 들어간다.

하나의 곡도 여러 플레이리스트에 포함될 수 있다.

플레이리스트 안에서 곡의 순서가 중요하다.

같은 플레이리스트에 같은 곡을 중복 추가할 수 없다.

겉으로 보면 N:N 관계지만
순서(trackOrder) 라는 추가 정보가 필요한 순간
단순 @ManyToMany는 부적합하다고 판단했다.

3. @ManyToMany 설계의 한계
처음 떠올릴 수 있는 설계는 다음과 같다.

java
코드 복사
@ManyToMany
@JoinTable(
    name = "PLAYLIST_TRACK",
    joinColumns = @JoinColumn(name = "playlist_id"),
    inverseJoinColumns = @JoinColumn(name = "track_id")
)
private List<Track> tracks;
하지만 이 방식에는 분명한 문제점이 있다.

관계 테이블에 컬럼을 추가할 수 없음

관계 자체를 도메인으로 다루기 어려움

실무에서 확장성이 매우 떨어짐

👉 결론: 중간 엔티티를 두고 풀자

4. 중간 엔티티로 N:N 관계 풀기
설계 구조는 다음과 같다.

Playlist 1 : N PlaylistTrack

Track 1 : N PlaylistTrack

이 시점에서 엔티티 구조를 수정했고,
변경 사항을 잃어버리지 않기 위해 수시로 Git 상태를 확인했다.

bash
코드 복사
git status
5. 엔티티 코드 정리
5.1 Track 엔티티
java
코드 복사
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

    @Column(length = 100)
    private String album;

    private Integer durationSec;

    protected Track() {}

    public Track(String title, String artist, String album, Integer durationSec) {
        this.title = title;
        this.artist = artist;
        this.album = album;
        this.durationSec = durationSec;
    }
}
Track 엔티티 작성 후 중간 저장.

bash
코드 복사
git add Track.java
git commit -m "feat: Track 엔티티 기본 구조 추가"
5.2 Playlist 엔티티
java
코드 복사
@Entity
@Table(name = "PLAYLIST")
public class Playlist {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(optional = false)
    @JoinColumn(name = "user_id")
    private User user;

    @Column(nullable = false, length = 100)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(nullable = false)
    private boolean isPublic = true;

    @OneToMany(mappedBy = "playlist", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<PlaylistTrack> playlistTracks = new ArrayList<>();

    protected Playlist() {}

    public Playlist(User user, String title, String description, boolean isPublic) {
        this.user = user;
        this.title = title;
        this.description = description;
        this.isPublic = isPublic;
    }

    public void addPlaylistTrack(PlaylistTrack playlistTrack) {
        playlistTracks.add(playlistTrack);
        playlistTrack.setPlaylist(this);
    }
}
양방향 연관관계에 대비해
편의 메서드를 함께 정의했다.

bash
코드 복사
git add Playlist.java
git commit -m "feat: Playlist 엔티티 및 연관관계 편의 메서드 추가"
5.3 PlaylistTrack 엔티티 (핵심)
java
코드 복사
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

    protected PlaylistTrack() {}

    public PlaylistTrack(Playlist playlist, Track track, Integer trackOrder) {
        this.playlist = playlist;
        this.track = track;
        this.trackOrder = trackOrder;
    }

    public void setPlaylist(Playlist playlist) {
        this.playlist = playlist;
    }
}
관계에 의미(trackOrder)가 생김

DB 레벨에서 중복 방지 가능

bash
코드 복사
git add PlaylistTrack.java
git commit -m "feat: PlaylistTrack 중간 엔티티 및 순서 관리 필드 추가"
6. 설계 포인트 정리
6.1 중간 엔티티를 사용한 이유
관계에 데이터가 필요함

확장 가능성 확보

실무에서 흔히 사용하는 패턴

6.2 유니크 제약으로 중복 방지
java
코드 복사
uniqueConstraints = {
    @UniqueConstraint(columnNames = {"playlist_id", "track_id"})
}
애플리케이션 로직 + DB 제약의 이중 방어.

7. 회고
JPA에서 ManyToMany는 학습용,
실무에서는 거의 항상 중간 엔티티로 푼다.

“관계도 하나의 도메인이다”라는 관점을 체감했다.

엔티티 설계 단계에서 시간을 쓰는 것이
이후 API/쿼리 설계를 훨씬 편하게 만들어준다.


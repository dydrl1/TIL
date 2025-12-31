[JPA] 연관관계의 주인과 mappedBy를 읽기 전용으로 두는 이유
1. TIL 주제 선정 이유

플레이리스트 기능을 구현하면서
단순히 엔티티를 연결하는 것보다 “누가 연관관계를 관리해야 하는가” 가 훨씬 중요하다는 것을 체감했다.

특히 다음 코드에서 자연스럽게 의문이 생겼다.

@OneToMany(mappedBy = "playlist")
private List<PlaylistTrack> playlistTracks;


왜 Playlist 쪽에는 mappedBy 가 붙어 있을까?

왜 여기서는 @JoinColumn 을 사용하지 않을까?

실제 INSERT / UPDATE 는 누가 담당할까?

이 의문을 정리하기 위해 연관관계의 주인 개념을 TIL로 남긴다.

2. 연관관계의 주인이란?

JPA에서 연관관계의 주인이란,

외래 키(FK)를 실제로 관리하는 엔티티

를 의미한다.

핵심 규칙

연관관계의 주인만이 FK를 변경한다

주인이 아닌 쪽은 조회 전용(read-only) 이다

mappedBy 가 붙은 쪽은 연관관계의 주인이 아니다

3. Playlist ↔ PlaylistTrack 관계에서의 주인

현재 설계 구조는 다음과 같다.

Playlist 1 ── N PlaylistTrack

PlaylistTrack 엔티티
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "playlist_id")
private Playlist playlist;

Playlist 엔티티
@OneToMany(mappedBy = "playlist")
private List<PlaylistTrack> playlistTracks;

연관관계의 주인

👉 PlaylistTrack

이유:

실제 FK(playlist_id)를 가지고 있음

DB 테이블 기준으로 FK는 PLAYLIST_TRACK에 존재

4. 왜 Playlist를 연관관계의 주인으로 두지 않았는가?

만약 Playlist 쪽에서 이렇게 설계했다면?

@OneToMany
@JoinColumn(name = "playlist_id")
private List<PlaylistTrack> playlistTracks;

문제점

FK를 Playlist가 관리하게 됨

엔티티 구조상 책임이 어색해짐

관계 변경 시 예상치 못한 UPDATE 쿼리 발생 가능

중간 엔티티의 의미가 약해짐

👉 “관계를 소유한 엔티티”는 PlaylistTrack이 더 자연스럽다

5. mappedBy를 사용한 이유
@OneToMany(mappedBy = "playlist")
private List<PlaylistTrack> playlistTracks;


이 설정의 의미는 다음과 같다.

Playlist는 연관관계의 주인이 아니다

playlist 필드는 PlaylistTrack에 의해 관리된다

Playlist에서는 조회만 담당한다

즉,

“이 관계는 내가 관리하지 않는다.
실제 관리는 PlaylistTrack이 한다.”

라는 선언이다.

6. 연관관계 편의 메서드가 필요한 이유

연관관계의 주인이 PlaylistTrack이더라도,
객체 관점에서는 양쪽 관계를 동시에 맞춰주는 것이 중요하다.

public void addPlaylistTrack(PlaylistTrack playlistTrack) {
    playlistTracks.add(playlistTrack);
    playlistTrack.setPlaylist(this);
}

이 메서드의 역할

객체 그래프 정합성 유지

한쪽만 설정해서 생기는 버그 방지

트랜잭션 내에서 예측 가능한 상태 유지

👉 JPA는 객체지향이기 때문에 양쪽을 맞춰주는 책임이 개발자에게 있다

7. 설계 포인트 정리

연관관계의 주인은 FK 기준으로 판단한다

mappedBy가 붙은 쪽은 읽기 전용이다

중간 엔티티가 존재할 경우, 그 엔티티가 주인이 되는 것이 자연스럽다

편의 메서드는 선택이 아니라 사실상 필수다

8. 회고

처음에는
“그냥 연결만 하면 되지 않나?”라고 생각했지만,

연관관계의 주인을 명확히 나누지 않으면

쿼리 흐름이 예측되지 않고

디버깅이 어려워지며

설계 의도가 코드에 드러나지 않는다

엔티티 설계는
DB 설계 + 객체 설계를 동시에 고려해야 한다는 것을 다시 한번 느꼈다.
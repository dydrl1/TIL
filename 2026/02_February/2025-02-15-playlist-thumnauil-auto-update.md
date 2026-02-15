[TIL] 엔티티 응집도 향상을 통한 플레이리스트 썸네일 자동화 로직 구현
1. 배경 및 문제 상황
문제: 플레이리스트(Playlist) 목록 조회 시, 대표 이미지(Thumbnail)가 비어 있어 UI상에서 일관된 경험을 주지 못함.

기존 방식: 플레이리스트 생성 시 별도의 이미지를 등록하지 않으면 null로 유지됨. 조회 시점에 매번 첫 번째 트랙의 이미지를 찾아야 하는 성능 저하 우려(N+1 문제 가능성).

2. 해결 방법: 도메인 주도 설계(DDD) 기반의 상태 관리
단순히 Service 계층에서 필드를 수정하는 것이 아니라, 엔티티 내부에서 비즈니스 규칙을 관리하도록 개선함.

핵심 로직: Playlist 엔티티의 응집도 강화
플레이리스트에 트랙이 추가되는 시점에 썸네일 존재 여부를 판단하여 자동으로 업데이트하는 로직을 엔티티 내부 메서드(addTrack)로 캡슐화함.

Java
public void addTrack(PlaylistTrack playlistTrack) {
    this.playlistTracks.add(playlistTrack);
    playlistTrack.setPlaylist(this);

    // [비즈니스 로직] 썸네일이 비어있는 경우, 첫 번째로 추가되는 곡의 이미지를 대표 이미지로 설정
    if (this.thumbnailUrl == null || this.thumbnailUrl.isBlank()) {
        this.thumbnailUrl = playlistTrack.getTrack().getImageUrl();
    }
}
3. 주요 학습 내용
① 복합 인덱스(Composite Index)의 이해와 설계
성능 최적화를 위해 인덱스 설계를 검토함.

카디널리티(Cardinality): 중복도가 낮은(값의 종류가 많은) 컬럼을 인덱스 앞쪽에 배치하는 것이 탐색 효율에 유리함.

조회 패턴 우선순위: WHERE 절에 반드시 포함되는 컬럼이 있다면, 해당 컬럼을 인덱스의 선행 컬럼으로 설정해야 인덱스가 정상적으로 작동함.

② JPA Dirty Checking (변경 감지)
Service 계층에서 @Transactional 어노테이션을 활용.

엔티티의 상태(Thumbnail)만 변경하면, 트랜잭션 종료 시점에 JPA가 변경 사항을 감지하여 자동으로 UPDATE 쿼리를 실행함. 이를 통해 불필요한 repository.save() 호출을 줄이고 객체 중심의 개발을 유지함.

③ 데이터 정합성 유지 (삭제 시나리오)
트랙 삭제 시, 삭제되는 트랙이 현재 플레이리스트의 대표 이미지인 경우를 대비함.

삭제 후 다음 트랙의 이미지로 썸네일을 동기화하는 로직을 추가하여 데이터 무결성을 보장함.

4. 회고 및 깨달은 점
객체에게 책임을 부여하기: 단순히 Service에서 Setter를 호출하는 것보다 엔티티 내부에 비즈니스 메서드를 만드는 것이 코드의 가독성을 높이고 실수를 방지한다는 것을 체감함.

인덱스의 중요성: 데이터가 많아질수록 인덱스 설계 한 줄이 시스템 전체 성능에 지대한 영향을 미칠 수 있음을 인지하고, 쿼리 실행 계획을 고려하는 습관을 들여야겠다고 다짐함.
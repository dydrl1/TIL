# [Strategy] Redis 캐싱 전략 비교와 프로젝트 적용 사례

## 1. 개요: 왜 캐싱 전략이 중요한가?

백엔드 시스템에서 모든 데이터를 메모리(Redis)에 적재할 수는 없습니다.  
서비스의 트래픽 특성이 **Read-heavy(조회 중심)** 인지, **Write-heavy(쓰기 중심)** 인지에 따라 적절한 캐싱 전략을 선택해야 합니다.

잘못된 캐싱 전략은 다음과 같은 문제를 유발합니다.

- 데이터 불일치(Inconsistency)
- 캐시 스탬피드(Cache Stampede)
- 시스템 복잡도 증가
- 장애 전파 리스크

결국 캐싱 전략은 **성능 최적화 기술이 아니라 아키텍처 설계 영역**에 가깝습니다.

---

## 2. Redis 캐싱 전략 상세 비교

| 전략 | 작동 방식 | 장점 | 단점 |
|------|------------|------|------|
| **Look-aside (Cache-Aside)** | 애플리케이션이 캐시를 먼저 조회 → 없으면 DB 조회 후 캐시에 저장 | DB 부하 감소, 캐시 장애 시 DB fallback 가능 | 최초 조회 시 Cache Miss 지연, 데이터 불일치 가능 |
| **Write-through** | DB 저장과 동시에 Cache에도 저장 | 강한 데이터 일관성 | 쓰기 성능 저하, 불필요한 데이터까지 캐싱 |
| **Write-behind (Write-back)** | Cache에 먼저 저장 후 비동기적으로 DB에 반영 | 쓰기 성능 극대화 | 캐시 장애 시 데이터 손실 위험, 구현 복잡 |

---

## 3. [Deep Dive] 내 프로젝트에는 왜 Look-aside인가?

### 📌 사례 1: 갓생로그 (루틴/챌린지 조회 최적화)

**상황**

- 사용자 행동 패턴이 조회 중심(Read-heavy)
- 루틴 탐색, 랭킹 조회 트래픽이 압도적
- 실시간성보다는 응답 속도와 안정성이 중요

**적용 전략**


Look-aside + Cache Eviction


**선택 이유**

- 캐시 장애 발생 시 DB fallback 가능 (High Availability 확보)
- DB 부하 감소
- 랭킹 데이터는 초 단위 실시간성이 필수는 아님

**해결한 문제**

데이터 수정 시 발생하는 데이터 불일치를 방지하기 위해  
`@CacheEvict`를 활용하여 업데이트 시 캐시를 즉시 제거하도록 설계했습니다.

→ 다음 조회 시 DB에서 최신 데이터를 가져와 자동 재적재

---

### 📌 사례 2: Youtube Playlist (API Quota 최적화)

**상황**

- YouTube Data API v3는 일일 Quota 제한이 매우 엄격
- 동일한 키워드 검색이 반복적으로 발생

**적용 전략**


Look-aside + TTL(Time To Live)


**선택 이유**

- 동일 검색어에 대해 매번 외부 API 호출은 비효율적
- API 비용 절감 및 응답 속도 개선 필요

**효과**

- Redis에 결과가 존재하면 외부 API 호출 생략
- API Quota 소모를 거의 0 수준으로 최적화
- 평균 응답 시간 대폭 단축

---

## 4. 데이터 정합성(Consistency)과 TTL 설계

캐싱 도입 시 가장 고민했던 질문:

> "이 데이터는 얼마나 오래 살아 있어야 하는가?"

TTL은 단순 숫자가 아니라 **데이터 생명 주기 설계**입니다.

### TTL 설정 기준

- 랭킹 데이터: **10분**
  - 실시간성 일부 유지
  - DB 부하 대폭 감소

- 유튜브 검색 결과: **1시간**
  - 트렌드 변화 반영
  - API 할당량 보호

### Cache Invalidation 전략

- 데이터 수정/삭제 시 명확한 Key 식별
- 서비스 레이어에서 `@CacheEvict` 처리
- 일부 로직은 인터셉터 레벨에서 분리 설계

핵심은 다음입니다.


성능 최적화 < 데이터 정합성 보장


---

## 5. 깨달은 점

이번 프로젝트를 통해 깨달은 점은 다음과 같습니다.

- Redis를 쓴다고 무조건 빨라지는 것이 아니다.
- 캐싱은 기술이 아니라 트레이드오프 설계다.
- 백엔드 엔지니어의 역할은 TTL과 정합성 정책을 결정하는 것이다.
- 가용성(Availability)과 일관성(Consistency) 사이에서 균형을 맞추는 것이 핵심이다.

CS에서 배운 **메모리 계층 구조 원리**가 실제 인프라 설계와 직결된다는 점을 실무에서 체감했습니다.

---

## 💡 구현 코드 (Snippet)

```java
// 챌린지 조회 최적화
@Cacheable(value = "challenges", key = "#challengeId", unless = "#result == null")
public ChallengeResponseDto getChallenge(Long challengeId) {
    return challengeRepository.findById(challengeId)
            .map(ChallengeResponseDto::from)
            .orElseThrow(() -> new EntityNotFoundException("Challenge not found"));
}

// 데이터 수정 시 정합성 유지
@CacheEvict(value = "challenges", key = "#challengeId")
public void updateChallenge(Long challengeId, ChallengeUpdateDto updateDto) {
    // DB 업데이트 로직...
}
🔚 결론

캐싱 전략은 단순 성능 개선 도구가 아닙니다.
서비스 트래픽 특성 분석 → TTL 설계 → Invalidation 정책 수립 → 장애 대비 설계까지 포함하는 아키텍처 의사결정입니다.
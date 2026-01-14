외부 서비스 연동을 위한 Provider 패턴 적용기
1. 배경

플레이리스트 서비스에서 곡을 검색할 때
단일 플랫폼(YouTube)에만 의존하지 않고,
향후 Spotify, Apple Music 등 외부 서비스 확장을 고려한 구조가 필요했다.

처음부터 특정 API에 종속된 구조를 만들면

서비스 로직이 외부 API 스펙에 묶이고

새로운 플랫폼 추가 시 대규모 수정이 발생한다.

이를 방지하기 위해 Provider 패턴(전략 패턴 기반) 을 적용해
외부 검색 서비스를 확장 가능한 구조로 설계했다.

2. 기존 방식의 문제점
// ❌ 안 좋은 예
public List<TrackDto> search(String query) {
    return youTubeClient.search(query);
}


이 구조의 한계:

서비스가 YouTube API 구조를 직접 앎

Spotify 추가 시 if/else 또는 switch 분기 폭증

테스트 시 외부 API 의존성 증가

3. Provider 패턴 적용 구조
전체 아키텍처
Controller
   ↓
ExternalSearchFacade
   ↓
ExternalSearchService (Interface)
   ↓
YouTubeSearchService / SpotifySearchService ...
   ↓
각 Provider Client

4. 핵심 설계 요소
1) Provider 식별자
public enum ProviderType {
    YOUTUBE,
    SPOTIFY
}

2) 공통 인터페이스 정의
public interface ExternalSearchService {
    ProviderType provider();
    List<ExternalTrackDto> search(String query, int limit);
}


모든 외부 검색 서비스는 동일한 계약(Contract) 을 가진다.

서비스 로직은 “어디서 가져왔는지”를 몰라도 된다.

3) YouTube 구현체
@Service
@RequiredArgsConstructor
public class YouTubeSearchService implements ExternalSearchService {

    private final YouTubeClient youTubeClient;

    @Override
    public ProviderType provider() {
        return ProviderType.YOUTUBE;
    }

    @Override
    public List<ExternalTrackDto> search(String query, int limit) {
        YouTubeSearchResponse res = youTubeClient.search(query, limit);

        if (res == null || res.getItems() == null) {
            return List.of();
        }

        return res.getItems().stream()
                .map(item -> ExternalTrackDto.builder()
                        .provider(ProviderType.YOUTUBE)
                        .externalId(item.getId().getVideoId())
                        .title(item.getSnippet().getTitle())
                        .artist(item.getSnippet().getChannelTitle())
                        .thumbnailUrl(item.getSnippet().getThumbnails().getDefault().getUrl())
                        .build())
                .toList();
    }
}

4) Facade에서 Provider 선택
@Component
@RequiredArgsConstructor
public class ExternalSearchFacade {

    private final List<ExternalSearchService> services;

    public List<ExternalTrackDto> search(ProviderType type, String query, int limit) {
        return services.stream()
                .filter(s -> s.provider() == type)
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("지원하지 않는 Provider"))
                .search(query, limit);
    }
}


Spring이 ExternalSearchService 구현체를 자동 주입 →
런타임에 Provider 선택 가능한 구조 완성.

5. 이 구조의 장점
1️⃣ 확장성 (OCP 준수)

새로운 플랫폼 추가 시
→ 기존 코드 수정 없음
→ XXXSearchService 구현체만 추가

2️⃣ 결합도 감소

Controller / Service는
→ YouTube API 구조를 전혀 모름
→ 내부 DTO만 의존

3️⃣ 테스트 용이성

외부 API 없이
→ FakeSearchService로 테스트 가능

4️⃣ 책임 분리 명확화
계층	책임
Facade	Provider 선택, 흐름 제어
Service	비즈니스 로직, DTO 변환
Client	HTTP 통신, 인증, 에러 해석
6. 적용 전/후 비교
Before

if / else 로 Provider 분기

서비스가 외부 API 구조에 종속

플랫폼 추가 시 수정 범위 큼

After

Provider 패턴으로 전략 분리

서비스는 추상화에만 의존

확장 비용 거의 0

7. 실무에서의 활용 포인트

이 패턴은 음악 서비스뿐 아니라 다음 상황에도 그대로 적용 가능하다.

결제 연동

KakaoPay / Toss / Payco

메시지 발송

SMS / Email / Slack

로그인

Google / Kakao / Naver OAuth

공통점:

“하나의 역할, 여러 외부 구현체”
→ Provider 패턴이 가장 이상적인 구조.

8. 이번 작업에서 배운 점

외부 연동은 처음부터 추상화하지 않으면 나중에 반드시 발목을 잡는다.

Provider 패턴은 단순 구조가 아니라 운영 비용을 줄이는 설계다.

확장보다 중요한 건
→ 확장하지 않아도 되는 구조를 미리 만드는 것.
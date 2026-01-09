📘 TIL — 외부 API 에러를 커스텀 예외 + 공통 응답 구조로 설계하기
1. 배경

프로젝트에서 YouTube API 연동 중
Quota Exceeded (429) 에러가 발생했다.

문제는 단순히 에러를 던지는 것이 아니라:

클라이언트에게 일관된 형식으로

서비스 내부 기준에 맞는 에러 코드와 함께

의미 있는 메시지를 내려줘야 한다는 점이었다.

그래서 다음 구조로 에러 처리 체계를 정리했다.

외부 API 에러
   ↓
커스텀 예외
   ↓
공통 ErrorCode
   ↓
ErrorResponse
   ↓
ControllerAdvice

2. 커스텀 예외 설계
2-1. 외부 API 전용 예외
public class ExternalApiQuotaExceededException extends RuntimeException {
    public ExternalApiQuotaExceededException(String message) {
        super(message);
    }
}

왜 RuntimeException인가?

외부 API 실패는 복구 불가능한 비즈니스 예외

호출부마다 try-catch 강제하기보다
→ 전역 예외 처리로 위임하는 것이 더 적절

3. 에러 코드 체계
3-1. ErrorCode enum
public enum ErrorCode {

    // 500
    INTERNAL_SERVER_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "S001", "서버 내부 오류입니다."),
    EXTERNAL_API_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "S002", "외부 서비스 연동 중 오류가 발생했습니다."),
    INVALID_REQUEST(HttpStatus.BAD_REQUEST, "S003", "잘못된 요청입니다."),

    // 외부 API
    YOUTUBE_QUOTA_EXCEEDED(
        HttpStatus.TOO_MANY_REQUESTS,
        "YOUTUBE_429",
        "현재 유튜브 검색 요청이 많아 잠시 후 다시 시도해 주세요."
    );

    private final HttpStatus status;
    private final String code;
    private final String message;
}

설계 포인트

HTTP Status: 프로토콜 레벨 의미

내부 코드: 서비스 도메인 기준 식별자

메시지: 사용자/프론트에 노출되는 문구

→ 상태 코드가 같아도 에러 의미를 코드로 구분할 수 있다.

4. 공통 에러 응답 구조
4-1. ErrorResponse
public class ErrorResponse {

    private final String code;
    private final String message;

    public ErrorResponse(String code, String message) {
        this.code = code;
        this.message = message;
    }

    // ErrorCode → ErrorResponse 변환 책임을 한 곳으로 집중
    public static ErrorResponse from(ErrorCode errorCode) {
        return new ErrorResponse(
            errorCode.getCode(),
            errorCode.getMessage()
        );
    }
}

핵심 설계 포인트

변환 로직을 컨트롤러나 핸들러가 아니라
응답 객체 자체가 책임지게 함

에러 응답 포맷이 바뀌어도
→ 수정 지점은 ErrorResponse 한 곳

5. 전역 예외 처리
5-1. ControllerAdvice
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ExternalApiQuotaExceededException.class)
    public ResponseEntity<ErrorResponse> handleQuotaExceeded(
            ExternalApiQuotaExceededException e) {

        ErrorCode errorCode = ErrorCode.YOUTUBE_QUOTA_EXCEEDED;

        return ResponseEntity
                .status(errorCode.getStatus())
                .body(ErrorResponse.from(errorCode));
    }
}

이 구조의 장점

컨트롤러가 깔끔해짐

모든 에러 응답 포맷이 완전히 통일

외부 API가 늘어나도
→ 예외 + ErrorCode만 추가하면 확장 가능

6. 최종 구조 요약
단계	역할
외부 API 실패	실제 에러 발생 지점
커스텀 예외	도메인 의미 부여
ErrorCode	서비스 표준 에러 정의
ErrorResponse	클라이언트 계약(Response Contract)
ControllerAdvice	전역 제어 지점
7. 정리

이번 구조를 통해 얻은 가장 큰 장점은
“에러도 하나의 API 스펙이다” 라는 관점을 코드로 정리했다는 점이다.

단순 try-catch가 아니라

정책 → 구조 → 코드 순서로 에러 처리를 설계하면

서비스 규모가 커져도 흔들리지 않는 기반이 된다.
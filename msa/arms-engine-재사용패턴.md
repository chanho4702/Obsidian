# ARMS 엔진 — 재사용 가능한 패턴/기법 ⭐

> 상위 노트: [[arms-engine-분석]] · 관련: [[arms-engine-아키텍처]]
> 다른 프로젝트(특히 [[msa-roadmap]] 템플릿)에 그대로/약간 손봐서 가져올 만한 것들.

---

## 1. 표준 API 응답 — `CommonResponse.ApiResult<T>`
성공/실패를 하나의 제네릭 래퍼로 통일. 정적 팩토리만 노출하고 생성자는 private.
```java
public static <T> ApiResult<T> success(T response) { return new ApiResult<>(true, response, null); }
public static ApiResult<?> error(ErrorCode errorCode, HttpStatus status) { ... }
// ApiResult<T> { boolean success; T response; ApiError error; }
// ApiError { String message; String errorCode; int status; }
```
**좋은 점**: success/error 진입점이 한 곳, 클라이언트는 `success` 불리언만 보면 됨.
**가져올 때**: 이게 곧 Phase1 W4의 "공통 응답(ApiResponse)". `ApiError`에 `traceId` 필드 하나만 추가하면 분산추적과 바로 연결됨.

## 2. 에러 카탈로그 — `ErrorCode` enum
메시지를 enum 상수로 중앙집중. `getErrorMsg(Object... arg)`가 `String.format`을 태워 파라미터화 메시지 지원.
```java
@Getter @RequiredArgsConstructor
public enum ErrorCode {
    ISSUE_CREATION_ERROR("이슈 생성시 오류 발생하여..."),
    DOCUMENT_NULL_ERROR("Document가 Null입니다.");
    private final String errorMsg;
    public String getErrorMsg(Object... arg) { return String.format(errorMsg, arg); }
}
```
**가져올 때**: enum에 `httpStatus`, `code`(예: `ARMS-1001`)까지 묶으면 더 강력. 현재는 메시지만 들고 있음.

## 3. 전역 예외 처리 — `@RestControllerAdvice`
`ErrorControllerAdvice`가 컨트롤러 예외를 잡아 `ApiResult`로 변환 + Slack 통보.
- `@ExceptionHandler(Exception.class)` → 500 + Slack
- `MissingPathVariableException` → `connectId` 누락 같은 도메인 케이스 분기
- `MethodArgumentNotValidException`(@Valid), `HttpMessageNotReadableException`(잘못된 JSON) → 400
> ⚠️ **반면교사**: 핸들러 메서드명을 한글(`전송데이터_유효성체크`)로 작성. 동작엔 문제없지만 협업/스택트레이스 가독성에서 비추천.

## 4. 횡단 로깅 + 장애 알림 — `@Aspect LoggingAdvice`
포인트컷으로 모든 controller/service 메서드를 감쌈:
```java
@Pointcut("execution(* com.arms..controller.*.*(..))") public void controller() {}
@Pointcut("execution(* com.arms..service.*.*(..))")    public void service() {}

@Before(...)         // [Class :: method] :: Start
@AfterReturning(...) // [Class :: method] :: End
@Around(...)         // 예외 시 Slack 통보 + 파라미터를 JSON 직렬화해 상세 로깅 후 rethrow
```
배울 점:
- `RequestFacade`(서블릿 요청) 인자를 필터링해 세션ID만 추출하고 나머지 파라미터만 JSON 직렬화.
- `ObjectMapper`에 `NON_NULL` 적용해 로그 잡음 제거.
- 원시/래퍼 타입은 `toString`, 객체는 JSON으로 — `convertToJsonOrDefault`.
> MSA에선 이 "예외→알림" 역할을 Sentry/Slack webhook + traceId 로깅으로 옮기면 됨.

## 5. 어노테이션 + AOP로 부가작업 선언화 — `IndexStatusSaveAspect`
`@IndexStatusSnapShot(clazz=..., value=...)`를 메서드에 붙이면, 그 메서드 실행 **전/후로 ES 인덱스 상태(문서 수 등) 스냅샷을 자동 저장**.
```java
@Around("@annotation(indexStatusSnapShot)")
public Object indexStatusAspect(ProceedingJoinPoint jp, IndexStatusSnapShot ann) throws Throwable {
    var repo = findRepository.findRepositoryByClass(ann.clazz());
    String jobId = UUID.randomUUID().toString();
    saveIndexStatus(..., jobId, JobStatus.BEFORE);
    Object result = jp.proceed();
    saveIndexStatus(..., jobId, JobStatus.AFTER);   // 같은 jobId로 before/after 페어링
    return result;
}
```
**재사용 가치 높음**: "메서드에 어노테이션만 붙이면 감사/스냅샷이 따라온다"는 선언적 부가기능 패턴의 좋은 예.
> ⚠️ `System.out.println(target)` 디버그 잔존 — 가져올 때 제거.

## 6. 타입 안전 키-값 쿼리 컨테이너 — `EsQuery` + `EsQueryBuilder`
`ParameterizedTypeReference`를 키로 쓰는 `ConcurrentHashMap<Type, Object>`. 제네릭 타입별로 쿼리 조각을 안전하게 보관/조회.
```java
public abstract class EsQuery {
    private ConcurrentHashMap<Type, Object> map = new ConcurrentHashMap<>();
    public <T> void setQuery(ParameterizedTypeReference<T> ref, T t) { map.put(ref.getType(), t); }
    public <T> T getQuery(ParameterizedTypeReference<T> ref) { /* type-safe cast */ }
}
```
빌더가 `bool / sort / highlight / matchAll`을 플루언트하게 누적:
```java
new EsQueryBuilder().matchAll().sort(esSort).highlight(esHl);
```
**기법 포인트**: 이질적 타입을 한 컨테이너에 넣되 타입 안전성을 `ParameterizedTypeReference`로 확보 — 타입 토큰(Typesafe Heterogeneous Container) 패턴.

## 7. 필터/불리언 쿼리의 합성 구조 — `Filter` 추상클래스
`Filter<T>`가 `EsBoolQuery`를 상속, 서브클래스(`TermQueryFilter`, `RangeQueryFilter`, `MatchQueryFilter`, `WildCardQueryFilter` …)는 `abstractQueryBuilder()`만 구현하면 됨.
```java
public abstract class Filter<T extends AbstractQueryBuilder<T>> extends EsBoolQuery {
    public abstract AbstractQueryBuilder<T> abstractQueryBuilder();
    @Override public void boolQueryBuilder(BoolQueryBuilder b) {
        if (abstractQueryBuilder() != null) b.filter(abstractQueryBuilder());
    }
}
```
must / mustnot / filter 디렉터리로 절(clause)별 클래스를 나눠 ES 쿼리 DSL을 OOP로 조립 — **빌더 + 합성(Composite)** 의 교과서적 적용.

## 8. 제네릭 Repository 추상화 — `EsCommonRepository<T,U>` / `EsCommonRepositoryWrapper<E>`
- 인터페이스가 scroll API, `search_after` 페이지네이션, 집계(일/주별), rolling index reindex/삭제, `cat indices` 등 **운영에 필요한 ES 연산을 한 계약에 모음**.
- `Wrapper<E extends BaseEntity>`가 엔티티 클래스를 들고 `FindRepository`로 적절한 repo를 런타임 조회 → 도메인 코드는 DTO만 넘기면 쿼리 팩토리가 알아서 쿼리 생성.
```java
public <T extends SearchDocDTO> DocumentResultWrapper<E> findRecentHits(T dto) {
    return getRepository().findRecentHits(SearchDocQueryFactory.searchDoc(dto).createForRecentTrue(entityClass));
}
```
**배울 점**: 대용량 ES 조회의 정석(스크롤 vs search_after, 최신본만 보는 `recent` 개념)을 한 곳에 캡슐화.

## 9. 정적 빈 접근 — `ApplicationContextProvider`
`ApplicationContextAware`를 구현해 `static getBean(Class)` 제공. 스프링이 관리 못 하는 지점(정적 컨텍스트, 레거시 코드)에서 빈을 끌어 쓸 때 사용.
> 안티패턴에 가깝지만 **레거시/프레임워크 경계에서 현실적인 탈출구**. 남용 주의.

## 10. 소소하지만 유용한 유틸
- `StreamUtil.toStream(Iterable)` — null-safe하게 `Iterable → Stream`(빈 스트림 반환).
- `UUIDUtil.createShortUUID()` — UUID를 SHA-256 해시 후 앞 4바이트(8 hex)만 → 짧은 식별자.
- `AES256` 계열 — 대칭키 암복호화 유틸(서버 자격증명 등 보호).
- `DateRangeUtil`, `ParseUtil`, `ReflectionUtil` — 날짜 구간/파싱/리플렉션 보조.

---

## 가져갈 우선순위 (내 MSA 템플릿 기준)
1. **즉시**: 1·2·3 (공통 응답/에러코드/전역핸들러) → starter-core 모듈의 기본.
2. **곧**: 4 (로깅 Aspect, 단 traceId 기반으로 현대화).
3. **검색 도메인 생기면**: 6·7·8 (ES 쿼리 빌더/Repo 추상화).
4. **선택**: 5 (어노테이션 기반 감사/스냅샷) — 감사 요건 생길 때.

# 플랫폼 코어 감사 보고서 (2026-10-18)

범위: auth-server · gateway-server · eureka-server · platform-backend(org-service, common-proto, common-starter, search-service 일부) · board-service · migration-service(보류 — 치명만) · design-system · myFront
방식: 읽기 전용 코드 정독(테스트 미실행). 확정 설계 결정(Keycloak 리다이렉트, 백채널 로그아웃, 수동 ClientRegistration, PG+Redis Streams, stdout JSON, +10000, common-starter, SEARCH_MODE/MAIL_MODE)은 전제로 두고 되돌리는 제안은 하지 않는다.

## 요약 표 (심각도순)

| ID | 분류 | 심각도 | 한 줄 |
|---|---|---|---|
| P-01 | 보안 | High | org-service gRPC(:9131)가 무인증 — `CreateGrant`로 아무 사용자에게 GLOBAL ADMIN 부여 가능, 운영에 리플렉션까지 켜짐 |
| P-02 | 보안/정합 | High | 구성원 비활성(퇴사)이 auth-server로 전파되지 않음 — RT 최대 90일·PAT 교환 계속 동작, `ListUserGrants`는 상태 미검사 |
| P-03 | 보안(PAT) | High | `admin` 스코프가 `/api/<제품>/admin/**`만 지킴 — `org:write` PAT로 권한 부여·구성원 비활성·메일 설정 변경 가능 |
| P-04 | 복원력 | High | search-service의 gRPC 호출 전부에 deadline 없음 — org/wiki 정지 시 스레드 무기한 점유 |
| P-05 | 복원력 | Medium | 공통 JWT 디코더 JWKS 조회에 타임아웃 없음(common-starter·게이트웨이) |
| P-06 | 보안(잠재) | Medium | common-starter가 `platform.jwt.*` 누락 시 조용히 물러남 → Boot 기본 디코더(audience 검증 없음)로 강등 |
| P-07 | 보안/중복 | Medium | 내부 시크릿 필터 3벌 복제·의미 불일치(trim 여부·헤더명), 하나의 `AGENT_INTERNAL_SECRET`이 PAT 교환·서비스 토큰 발급·에이전트 생성을 모두 엶 |
| P-08 | 보안/성능 | Medium | PAT 교환 증폭 — 무작위 `chanho_pat_` 토큰은 부정 캐시를 우회해 매 요청 auth-server+DB를 때림, 해당 경로 rate-limit 없음 |
| P-09 | 보안(내부망) | Medium | Eureka 무인증 등록 + docker에서도 wiki는 `lb://wiki-backend` → 내부 컨테이너 하나로 위키 트래픽(JWT 포함) 탈취 가능, Redis 무비밀번호 |
| P-10 | 복원력/계약 | Medium | CB·fallback·Retry가 board에만 — wiki/alm/org/agent 다운 시 SCG 기본 500/503 본문(계약 밖), 전역 10s 응답 타임아웃이 PDF 내보내기 등 장시간 요청을 504로 자름 |
| P-11 | 오류 계약 | Medium | common-starter 오류 계약 구멍 — `IllegalArgumentException` 원문 노출, 타입 불일치·파라미터 누락·AccessDenied·낙관적 락·미처리 예외가 계약 밖 모양 |
| P-12 | 관측성 | Medium | 게이트웨이 `RequestLoggingFilter`가 GlobalFilter라 401/403·PAT 거부·`/api/platform/**` 응답이 로그에 안 남음, 응답에 `X-Request-Id` 미반환 |
| P-13 | 계약/중복 | Medium | board-service가 공용 예외를 복제(`NotFoundException`)하고 오류 본문이 `{error:코드, message}` — 확정 결정과 충돌(사용자 결정 필요) |
| P-14 | 문서 드리프트 | Medium | myFront CLAUDE.md가 비밀번호 로그인·회원가입(`loginWithPassword`/`SignUpPage`)을 설명 — 실코드·확정 결정과 반대, `.env` 기본 `:8000`은 compose가 더 이상 발행 안 함 |
| P-15 | 버전 드리프트 | Low | org-admin 0.2.2 발행됐으나 wiki-front·alm-front는 0.2.1, DS README 버전·소비 방식 전면 구식, changeset baseBranch `master`(실제 main) |
| P-16 | 보안(CORS) | Low | CORS 허용 메서드에 PATCH 없음(org·agent가 PATCH 사용), 단일 오리진 — myFront dev는 프록시 없이 교차 오리진 호출 |
| P-17 | 문서 드리프트 | Low | 루트 CLAUDE.md "백엔드 8" vs README "백엔드 12", 변경이력 버전 표기 0.15.0 vs 현재 0.16.0 |
| P-18 | 위생 | Low | search-service 죽은 코드(`isUnavailable` + `@SuppressWarnings("unused")`), 503 메시지에 내부 정보(`page=… status=…`)가 사용자에게 노출 |
| P-19 | 의존성 | Low | grpc 1.69.0·protobuf 4.29.3을 8개 build.gradle이 각자 하드코딩 — 버전 카탈로그/BOM 부재 |
| P-20 | 보안(설정) | Low | auth-server 기본 자격증명(`platform-bff-secret`, `keycloak/keycloak`) fail-fast 없음, 서명키 회전 수단 없음(JWKS 단일 키) |
| P-21 | 위생(PAT) | Low | PatScopeWebFilter의 "스코프 클레임 없는 구버전 PAT 통과" 분기 — V5/V6 백필 완료로 불필요한 열린 문 |
| P-22 | 성능 | Low | org-service `MemberMirrorFilter`가 모든 요청에서 쓰기 트랜잭션(+AdminGate가 `/api/org/me` 호출) |
| P-23 | 설정 | Low | 게이트웨이 `trusted-proxies`가 "로컬 dev 전용" 주석과 달리 base yml에 있어 docker에도 사설대역 전부 신뢰 |
| P-24 | (보류) migration | Low | dev 프로필 issuer `:19000` 불일치(dev에서 전부 401), 원본 URL SSRF 가드 없음 — 치명 아님, 기록만 |

긍정 확인(결함 아님): 게이트웨이 헬스 집계(20s `Mono.cache` 공유·프로브/집계 이중 타임아웃), PAT 교환 캐시 설계(TTL 3종·만료 가드·fail-closed), AdminGate의 503/403 분리, auth-server 아웃바운드 HTTP 타임아웃(2s/3s), Spring Boot 4.0.6·Cloud 2025.1.2·common-proto/starter 0.16.0이 전 서비스 동일(버전 드리프트 없음), JWK 볼륨 영속화, Redis·Postgres 루프백 발행.

---

## P-01 org-service gRPC 무인증 — 권한 쓰기 RPC 포함
- 분류: 보안 / 심각도: **High**
- 근거:
  - `platform-backend/org-service/src/main/java/com/platform/orgservice/grpc/GrpcServerLifecycle.java:36-40` — `ServerBuilder.forPort(port).addService(permissionGrpcService).addService(ProtoReflectionService.newInstance())`, 인터셉터·인증 없음.
  - `.../grpc/PermissionGrpcService.java:260-283` `createGrant` — 호출자 검증 없이 `GrantEntry.of(USER, userId, kind, resourceId, role)` 저장(GLOBAL ADMIN 포함), REST 경로에 있는 감사(GrantAudit) 기록도 없음. `:285-304` `revokeGrant`도 동일.
  - `common-proto/src/main/proto/platform/org/v1/org.proto:13,15` — `CreateGrant`/`RevokeGrant`가 공개 계약.
  - `org-service/build.gradle:42` — 운영 이미지에 `grpc-services`(리플렉션) 포함.
- 왜: 게이트웨이는 "zero-trust 이중 검증"을 표방하지만(gateway `SecurityConfig.java:26`) 권한의 정본인 org의 쓰기 RPC는 네트워크 위치만으로 열려 있다. 같은 docker 네트워크에 공개 표면을 가진 `docs-backend`, 외부 URL을 끌어오는 `migration-service`, LLM 작업을 조율하는 `agent-service`가 있다 — 하나만 뚫려도 `grpcurl`(리플렉션 덕에 스키마 불필요)로 자신에게 GLOBAL ADMIN을 줄 수 있다.
- 제안(M):
  1. 공유 토큰 기반 `ServerInterceptor`(메타데이터 `x-internal-token`, `MessageDigest.isEqual`, 빈 값이면 fail-closed) — 클라이언트 쪽은 `ClientInterceptor` 하나를 common-starter(또는 common-proto 헬퍼)로 배포.
  2. 쓰기 RPC(`CreateGrant`/`RevokeGrant`)는 호출 서비스 allowlist(wiki/alm만)로 더 좁히고, REST와 같은 `GrantAudit` 기록.
  3. 리플렉션은 `platform.grpc.reflection=false` 기본, dev 프로필에서만 켬.

## P-02 구성원 비활성이 인증 계층으로 전파되지 않음
- 분류: 보안/정합 / 심각도: **High**
- 근거:
  - `auth-server/.../user/User.java:40` `enabled = true` — auth-server 전체에 `setEnabled` 호출이 하나도 없다(grep 0건). 그런데 `token/RefreshTokenService.java:64`, `pat/PersonalAccessTokenService.java:151`은 이 플래그만 본다 → 항상 통과.
  - `platform-backend/org-service/.../member/MemberAdminService.java:204-217` `deactivate` — org 상태 변경 + Keycloak 비활성만 하고 auth-server에는 아무것도 알리지 않음(주석: "다음 로그인은 어차피 차단된다").
  - `.../grpc/PermissionGrpcService.java:247-257` `listUserGrants`는 `statusOf` 검사 없음(`checkPermission`만 `:214`에서 검사). search-service는 이 RPC로 접근 가능 스페이스를 계산(`search-service/.../permission/GrpcPermissionClient.java:39`).
- 왜: 자체 RT는 슬라이딩 14일·절대 90일(`application.yml` `session-absolute-ttl-seconds`)이라 퇴사자도 기존 브라우저 세션으로 90일까지 AT를 재발급받고, PAT는 만료일(최대 365일)까지 교환된다. org를 묻지 않는 board-service·`/api/me`는 그대로 열리고, 검색은 `ListUserGrants` 경로로 결과가 나올 수 있다(위키 필터가 상태를 보는지에 의존).
- 제안(M): org `deactivate`/`suspend` 시 auth-server 내부 API(`POST /internal/users/{id}/disable` — 기존 `/internal/**` 체인 재사용) 또는 Redis Streams `MemberStatusChanged` 이벤트로 ① `users.enabled=false` ② RT 가족 전부 폐기 ③ PAT 일괄 폐기. 재활성 시 `enabled=true`. `listUserGrants`도 비활성이면 빈 목록 + 사유(혹은 호출측이 `CheckPermission`과 같은 기준 적용). 회귀 테스트: 비활성 직후 refresh→401, PAT 교환→401.

## P-03 PAT `admin` 스코프가 관리 작업 대부분을 덮지 못함
- 분류: 보안(PAT) / 심각도: **High**
- 근거: `gateway-server/.../filter/PatScopeRules.java:34,36,85` — 제품 경로는 `<제품>:read|write`, `admin`은 두 번째 세그먼트가 정확히 `admin`일 때만 추가. org-service의 관리 API는 대부분 `/api/org/admin` 밖에 있다: `/api/org/grants`(권한 부여), `/api/org/members`(상태 PATCH=비활성·정지), `/api/org/settings/mail`(SMTP 설정), `/api/org/invitations`, `/api/org/teams`(org-service `@RequestMapping` 목록).
- 왜: 전역 관리자가 "읽기/쓰기만" 의도로 `org:write` PAT를 만들면 그 토큰으로 권한 부여·계정 비활성·메일 릴레이 변경까지 된다. 스코프 모델의 존재 이유(유출 시 피해 반경 축소)가 org에서 무력화된다. wiki·alm도 스페이스/프로젝트 권한 관리 경로가 `/admin` 아래가 아니면 같은 문제다.
- 제안(S~M): `PatScopeRules`에 "관리 경로 표"를 명시 추가 — `/api/org/grants/**`·`/api/org/members/**`(쓰기)·`/api/org/settings/**`·`/api/org/invitations/**`·`/api/org/teams/**`(쓰기)와 wiki/alm의 권한 관리 경로에 `admin` 요구. 표 단위 테스트(`PatScopeRulesTest`)에 케이스 추가. 장기적으로는 서비스가 `scope` 클레임을 직접 보는 `@RequiresScope`를 common-starter에 둬 게이트웨이 표와 이중화.

## P-04 search-service gRPC deadline 부재
- 분류: 성능/복원력 / 심각도: **High**
- 근거: `platform-backend/search-service/.../permission/GrpcPermissionClient.java:39,64`, `.../content/GrpcWikiContentClient.java:30,60,83` — 모든 블로킹 스텁 호출에 `withDeadlineAfter` 없음. 채널도 기본값(`config/GrpcClientConfig.java:33,41`). 반면 wiki/alm/agent는 전부 deadline을 건다(`wiki-backend/.../GrpcPermissionClient.java:142`, `alm-backend/.../GrpcPermissionClient.java:138`).
- 왜: org나 wiki가 행(hang)하면 검색 요청 스레드가 무기한 대기 → 게이트웨이는 10s에 끊지만 서버 스레드·톰캣 풀은 남아 검색 서비스 전체가 굳는다. 색인 이벤트 소비(`getPageContent`)도 멈춰 스트림 컨슈머가 정지한다.
- 제안(S): 다른 서비스와 같은 `platform.org-grpc.deadline-seconds`(단건 2~3s), 스트림(`listPageContents`)은 별도 긴 deadline. 채널에 `keepAliveTime`·`idleTimeout` 지정. `DEADLINE_EXCEEDED`는 기존처럼 503.

## P-05 JWKS 조회 타임아웃 없음
- 분류: 복원력 / 심각도: Medium
- 근거: `platform-backend/common-starter/.../security/PlatformResourceServerAutoConfiguration.java:39` `NimbusJwtDecoder.withJwkSetUri(jwkSetUri).build()` — `restOperations` 미지정 시 Spring Security 기본 `new RestTemplate()`(연결·읽기 타임아웃 무한; 버전별 기본값 재확인 권장). 게이트웨이도 `gateway-server/.../config/SecurityConfig.java:109` `NimbusReactiveJwtDecoder.withJwkSetUri(...)` 기본 WebClient.
- 왜: JWKS 캐시 만료·미지 kid 시점에 auth-server가 느리면 6개 리소스 서버의 인증 경로 전체가 함께 멈춘다. 다른 아웃바운드는 모두 2s/3s 타임아웃을 갖췄는데 가장 뜨거운 경로만 빠져 있다.
- 제안(S): starter에서 `RestTemplate`(connect 2s/read 3s)을 `.restOperations(...)`로 주입, 게이트웨이는 타임아웃 건 `WebClient`를 `.webClient(...)`로. 0.17.0으로 발행 후 전 서비스 동시 갱신(공유 버전 하나 원칙).

## P-06 common-starter의 조용한 강등
- 분류: 보안(잠재) / 심각도: Medium
- 근거: `PlatformResourceServerAutoConfiguration.java:30` `@ConditionalOnProperty(prefix="platform.jwt", name={"issuer","audience"})`. 현재는 전 서비스 yml에 값이 있어 정상(org `application.yml:74-75` 등)이나, 새 서비스가 키를 빠뜨리면 이 구성이 통째로 빠지고 Boot 기본 디코더(서명·만료만, **audience 미검증**, roles→ROLE_ 변환도 없음)가 뜬다. 클래스 주석(:23-24)이 경고하는 바로 그 실패가 설정 누락으로도 재현된다.
- 제안(S): 조건을 `jwk-set-uri` 존재로 바꾸고 issuer/audience가 비면 `IllegalStateException`으로 기동 실패(fail-fast). 또는 `@ConfigurationProperties` + `@Validated @NotBlank`.

## P-07 내부 시크릿 필터 복제·불일치, 시크릿 하나의 과다 권한
- 분류: 보안/중복 / 심각도: Medium
- 근거:
  - `auth-server/.../agent/InternalSecretFilter.java` — 헤더 `X-Internal-Secret`, 설정·헤더 모두 `trim()`.
  - `platform-backend/org-service/.../security/InternalTokenFilter.java:27-42` — 헤더 `X-Internal-Token`, **trim 없음**. `wiki-backend/.../config/InternalTokenFilter.java` 3번째 사본.
  - `AGENT_INTERNAL_SECRET` 하나가 게이트웨이(PAT 교환)·agent-service에 같이 주입(`infra/keycloak/docker-compose.yml:262,309,639`)되고 auth-server `/internal/**` 전체 — `/internal/pat/exchange`, `/internal/service-tokens`(에이전트 JWT 발급, `AgentTokenController.java`), `/internal/agents`, `/internal/pat/stats` — 를 연다.
- 왜: auth-server는 ".env 트레일링 스페이스" 사고를 trim으로 막았지만 org/wiki는 같은 사고가 그대로 남아 있다. 공용 보안 코드가 서비스마다 갈라지는 것은 common-starter 도입 이유(S-02)와 정면으로 어긋난다. 또 게이트웨이가 쥔 비밀로 에이전트 JWT 발급까지 가능하다.
- 제안(M): `com.platform.common.security.InternalTokenFilter`(헤더명·trim·상수시간·fail-closed·`{"error"}` 본문)를 starter로 올리고 3곳 교체. 시크릿은 호출자별로 분리(`GATEWAY_PAT_SECRET`, `AGENT_SERVICE_SECRET`)하고 경로별 허용 호출자 매핑.

## P-08 PAT 교환 증폭(DoS)
- 분류: 보안/성능 / 심각도: Medium
- 근거: `gateway-server/.../filter/PatExchangeWebFilter.java:108,117,126-145` — 캐시 키가 토큰 해시라 매번 다른 무작위 `chanho_pat_…`는 부정 캐시(10s)를 항상 빗나가 `PatExchangeClient.exchange`(auth-server, 2s 타임아웃) + DB 조회(`PersonalAccessTokenService.java:134`)가 일어난다. rate limiter는 auth·search 라우트에만 있고(`application.yml:57-100,205-209`) `/api/wiki/**` 등에는 없다.
- 왜: 인증 없이 아무 경로로 auth-server와 authdb에 요청 1:1 부하를 실을 수 있다. 503 캐시(2s)도 키별이라 보호가 안 된다.
- 제안(S): PAT 교환 직전 IP 기준 토큰버킷(기존 `ipKeyResolver` + Redis 또는 Caffeine 로컬 버킷, 예: 미스 10/s) — 초과 시 429 `{"error":"rate_limited"}`. 형식 사전검증(접두사 뒤 43자 base64url)으로 쓰레기 입력은 auth-server까지 보내지 않기.

## P-09 내부망 무인증 컴포넌트(Eureka·Redis)
- 분류: 보안(내부망) / 심각도: Medium
- 근거: `eureka-server/src/main/resources/application.yml` — 보안 의존성·인증 없음. compose 게이트웨이 설정 주석 "wiki는 Eureka에 N개 인스턴스를 등록하고 기본 lb://wiki-backend 라우트를 사용"(`infra/keycloak/docker-compose.yml` gateway 블록, `WIKI_SERVICE_URI` 미주입). Redis는 `command: ["redis-server","--appendonly","yes"]`(:86) — 비밀번호 없음, 그런데 협업 1회 티켓 인증(GETDEL)·이벤트 스트림·rate-limit이 여기 있다.
- 왜: 내부 컨테이너 하나가 `wiki-backend` 이름으로 Eureka에 등록하면 게이트웨이가 사용자 요청(Authorization JWT 포함)을 그쪽으로 라운드로빈한다. Redis 쓰기 권한이면 협업 티켓 위조·도메인 이벤트 위조가 된다. P-01과 같은 "네트워크 위치=신뢰" 가정의 연장.
- 제안(M): Eureka에 basic auth(spring-security + `defaultZone: http://user:pass@eureka:8761/eureka`), Redis `requirepass`/ACL(서비스별 사용자), 둘 다 compose env로 주입. 확정 결정(Redis Streams)은 유지.

## P-10 게이트웨이 장애 격리·타임아웃이 board에만
- 분류: 복원력/계약 / 심각도: Medium
- 근거: `gateway-server/src/main/resources/application.yml:40-56` board만 `CircuitBreaker`+`Retry`+`fallbackUri`; `org`(:119), `wiki`(:139), `alm`(:144), `agent`(:158) 라우트는 필터 없음. `FallbackController.java`는 `/fallback/board` 하나. 전역 `response-timeout: 10s`(:27) — 예외는 SSE 한 라우트(:152-157)뿐. wiki `GET /api/wiki/pages/{id}/export.pdf`(`PdfExportController.java:37`), 20MB 첨부 업로드도 10s 상한.
- 왜: 핵심 제품(wiki/alm/org)이 죽으면 프론트가 받는 것은 SCG 기본 오류 본문(`{"timestamp","path","status","error":"Internal Server Error"}` 또는 연결 거부 500)이라 "503 → 장애 안내" 계약(platform-integration)이 깨진다. 큰 문서 PDF는 504로 잘린다.
- 제안(S~M): wiki/alm/org/agent에 `CircuitBreaker`(+GET 한정 Retry, `exceptions: java.io.IOException`) 및 공용 `/fallback/{service}` → `503 {"error":"<service>_unavailable"}`. `timelimiter` 인스턴스별 11s. PDF 내보내기·업로드 라우트는 `metadata.response-timeout`을 60s로 분리.

## P-11 common-starter 오류 계약 구멍
- 분류: 오류 계약 / 심각도: Medium
- 근거: `platform-backend/common-starter/.../web/PlatformApiExceptionHandler.java:44-46` — `IllegalArgumentException`의 원문을 그대로 사용자에게(라이브러리 발 메시지 "No enum constant com.platform…", "For input string" 등 노출). 다음이 미처리: `MethodArgumentTypeMismatchException`(경로 id에 문자), `MissingServletRequestParameterException`, `HandlerMethodValidationException`, `ObjectOptimisticLockingFailureException`(409 계약), `AccessDeniedException`(org `common/ApiExceptionHandler.java:17-21`, board가 각자 복제), 그 밖의 `Exception` → Boot 기본 `/error` 모양. 401/403은 Security 기본 엔트리포인트라 본문 없음.
- 왜: "본문은 `{"error": 한국어 메시지}`" 계약을 starter가 보장한다고 선언했는데 흔한 4xx 절반이 계약 밖으로 샌다. 프론트는 `error` 필드를 그대로 화면에 띄우므로 영어·내부 클래스명이 사용자에게 보인다.
- 제안(S): 위 예외별 핸들러 추가(한국어 고정 문구), `IllegalArgumentException`은 서비스가 의도적으로 던진 것만 노출하도록 `BadRequestException`(공용 예외)로 대체 유도 + IAE는 고정 문구, 최후 `Exception` 핸들러 500 `{"error":"일시적인 오류가 발생했습니다"}`(로그엔 원인). `AuthenticationEntryPoint`/`AccessDeniedHandler` 빈도 starter에서 `{"error"}`로 제공(서비스 체인이 `.exceptionHandling`으로 꽂기만).

## P-12 게이트웨이 요청 로그 누락
- 분류: 관측성 / 심각도: Medium
- 근거: `gateway-server/.../filter/RequestLoggingFilter.java:21-22` `implements GlobalFilter` + `HIGHEST_PRECEDENCE` — GlobalFilter는 라우팅 단계라 보안 체인(-100)·PAT 필터(-101)가 끊은 401/403/503과, 라우트가 아닌 `PlatformController`(`/api/platform/**`)·`FallbackController` 응답은 `request.in/out` 줄이 없다. `X-Request-Id`는 다운스트림 헤더에만 넣고 응답에는 안 싣는다.
- 왜: 인증 실패 폭주·PAT 거부·관리자 API 감사가 Loki에서 보이지 않는다(문제 조사 시 가장 필요한 것들). 사용자가 오류 화면에서 요청 ID를 보고할 수도 없다.
- 제안(S): `WebFilter`(order -102, PAT 필터보다 앞)로 바꾸고 `exchange.getResponse().beforeCommit`에서 `X-Request-Id` 응답 헤더 추가. 기존 `RequestLoggingFilterTest` 이관.

## P-13 board-service 공용 예외 복제·오류 모양 이탈
- 분류: 계약/중복 / 심각도: Medium (사용자 결정 필요)
- 근거: `board-service/.../common/NotFoundException.java:3`(공용 `com.platform.common.error.NotFoundException`과 중복), `.../common/ApiExceptionHandler.java:12-34` — `{"error":"not_found","message":…}` 등 코드+메시지 2필드.
- 왜: 루트 확정 결정 "공용 예외 … 서비스에 다시 복제하지 않는다", "`{"error": 메시지}` 오류 계약"과 충돌. 주석은 "myFront 계약"이라 하므로 의도된 예외일 수 있다 — 임의 판단하지 않고 결정을 올린다.
- 제안(S): (a) myFront `boardStore`를 `{error}` 단일 필드로 맞추고 board 핸들러·예외 삭제(스타터 사용), 또는 (b) 예외로 공식화해 루트 CLAUDE.md/AGENTS.md에 명시. 어느 쪽이든 board `NotFoundException`은 공용 것으로 교체.

## P-14 myFront 문서 드리프트(확정 결정과 반대 서술)
- 분류: 문서 드리프트 / 심각도: Medium
- 근거: `myFront/CLAUDE.md:45`("wired to the `oauth-oidc-login` backend"), `:55`(`loginWithPassword, signUp, completeOAuthLogin`), `:59-60`(`LoginPage (form + Google)`, `SignUpPage (/register, email + password×2)`). 실제: `src/app/pages/`에 `LoginPage·NotFoundPage·RouteErrorPage`뿐, `src/auth/index.ts:11,13`은 `loginUrl`/`googleLoginUrl`(Keycloak 리다이렉트). 변경이력도 2026-06-20 1줄. `myFront/.env` `VITE_API_BASE=http://localhost:8000` + README `:91,102` — 그러나 compose 게이트웨이는 호스트 포트를 발행하지 않는다(`docker-compose.yml` gateway 블록 주석).
- 왜: 에이전트가 CLAUDE.md를 정본으로 읽어 자체 로그인 폼·ROPC를 되살릴 위험(확정 결정 위반 유도). dev 기본값으로는 도커 스택에 붙지 않는다.
- 제안(S): CLAUDE.md auth 절을 실코드(Keycloak 리다이렉트·`post_login_redirect` 쿠키·silent refresh)로 재작성, 기본 `.env`를 dev 오프셋 `:18000`(또는 vite proxy + 빈 base)로, README 동기화.

## P-15 디자인시스템 버전·문서 드리프트
- 분류: 버전/문서 드리프트 / 심각도: Low
- 근거: `design-system/packages/org-admin/package.json:3` `0.2.2`(태그 `org-admin-v0.2.2` 존재, 관리자 아바타 변경 UI 포함) vs `wiki-front/package.json:14`, `alm-front/package.json:13` `org-admin@0.2.1`. `design-system/README.md:14,20-27,121-164` — "패키지 2개", tokens 0.3.0/react 0.5.0, "tarball 방식 소비", 소비 표 React 0.3.0~0.5.0(실제 react 0.11.0·tokens 0.4.0, GitHub Packages alias). `.changeset/config.json` `"baseBranch": "master"`(리포는 main). myFront는 DS 미사용(MUI v9, vite ^6 vs 다른 프론트 ^7) — 의도라면 README에 명시.
- 제안(S): 두 프론트 org-admin 0.2.2로 동시 갱신(백엔드 아바타 관리 API 배포 여부 먼저 확인), README 패키지 표·소비 표 갱신 및 tarball 절은 "대안"으로 축소, baseBranch `main`.

## P-16 CORS 설정의 잠재 결함
- 분류: 보안(CORS) / 심각도: Low
- 근거: `gateway-server/.../config/SecurityConfig.java:125-126` 오리진 1개·메서드 `GET,POST,PUT,DELETE,OPTIONS`. PATCH 사용처: org `GrantController`·`MemberController`·`TeamController`, agent `PersonaController.java:52,62`. compose는 `CORS_ALLOWED_ORIGIN` 미주입(기본 `http://localhost:5173`). myFront `vite.config.ts`는 프록시가 없어 dev에서 교차 오리진 호출.
- 왜: 운영은 nginx 동일 오리진이라 지금은 무해하지만, 교차 오리진 경로(myFront dev)에서 PATCH 화면이 붙는 순간 preflight 실패. 여러 dev 프론트(5173/5174/…) 동시 사용도 불가.
- 제안(S): `PATCH` 추가, `cors-allowed-origins` 콤마 목록 지원(`setAllowedOrigins(List)`), compose에서 명시.

## P-17 루트 문서 서비스 목록·버전 드리프트
- 분류: 문서 드리프트 / 심각도: Low
- 근거: `CLAUDE.md:3` "백엔드 8(...)" — alm-backend·collaboration-service·search-service(platform-backend 모듈)·docs-backend 누락. `README.md:40` "백엔드 12개". `CLAUDE.md:39-40` 변경이력의 공유 버전 표기 0.15.0, 실제 전 서비스 0.16.0(`wiki-backend/build.gradle:42,44`, 태그 `v0.16.0`).
- 제안(S): 루트 CLAUDE.md·AGENTS.md 서비스 목록을 README와 일치시키고, 현재 공유 버전을 한 곳(확정 결정 줄)에 "현재 0.16.0"으로 기재(두 파일 함께 갱신 규칙).

## P-18 search-service 위생
- 분류: 위생 / 심각도: Low
- 근거: `search-service/.../permission/GrpcPermissionClient.java:87-94` `@SuppressWarnings("unused") isUnavailable` — 미사용. `.../content/GrpcWikiContentClient.java` 503 메시지 `"위키 콘텐츠 조달 실패: page=" + pageId + " status=" + code` 등이 starter 핸들러를 통해 그대로 사용자 `error`로 나감.
- 제안(S): 죽은 코드 삭제(또는 로그 분기에 실제 사용), 사용자 메시지는 고정 문구·세부는 로그로.

## P-19 gRPC/protobuf 버전 하드코딩 분산
- 분류: 의존성 / 심각도: Low
- 근거: `def grpcVersion = '1.69.0'`이 common-proto·org·search·wiki·alm·agent·migration build.gradle 각자(예: `org-service/build.gradle:18`, `wiki-backend/build.gradle:38`), protobuf `4.29.3`은 `common-proto/build.gradle:12`. grpc 1.69(2025-01)은 1년 이상 묵음.
- 왜: common-proto가 `api grpc-stub/protobuf`를 노출하지만 각 서비스의 `grpc-netty-shaded`·`grpc-inprocess`는 따로 고정 — proto 쪽만 올리면 런타임 불일치가 생긴다.
- 제안(S): common-proto와 함께 `com.platform:platform-bom`(java-platform) 발행 또는 common-proto가 `grpc-bom`을 `api platform(...)`으로 노출, 서비스는 버전 없이 선언. 그 후 grpc 최신 마이너로 일괄 상향.

## P-20 auth-server 기본 비밀값·키 회전
- 분류: 보안(설정) / 심각도: Low
- 근거: `auth-server/src/main/resources/application.yml` — `OIDC_CLIENT_SECRET:platform-bff-secret`, `AUTH_DB_USERNAME/PASSWORD:keycloak`(주석 "운영에서는 반드시 주입"이지만 강제 없음). `jwt/JwtKeyProvider.java:30-44` — 단일 RSA 키, 회전/다중 kid 미지원.
- 제안(S/M): docker 프로필에서 기본값이면 기동 실패하는 검증 빈(S). 키 회전은 JWKS에 이전 키 병행 게시 + 신규 키 서명(M, 우선순위 낮음).

## P-21 구버전 PAT 통과 분기
- 분류: 위생(PAT) / 심각도: Low
- 근거: `gateway-server/.../filter/PatScopeWebFilter.java:68-70` — `scope` 클레임이 없으면 무조건 통과. auth-server V5/V6가 기존 행을 백필했고 교환 JWT TTL은 300s라 이미 해당 토큰은 존재할 수 없다.
- 제안(S): 분기 제거(클레임 없음 = 빈 스코프 = `/api/me` 외 거부), 테스트 갱신.

## P-22 org-service 요청마다 쓰기 트랜잭션
- 분류: 성능 / 심각도: Low
- 근거: `org-service/.../security/MemberMirrorFilter.java:55-58` → `member/MemberService.java:50-62` `@Transactional mirror`(조회 + `refresh` 더티체킹)가 모든 org 요청에서 실행. 게이트웨이 `AdminGate`도 판정마다 `/api/org/me` 호출(30s 캐시).
- 제안(S): 이름·이메일이 같으면 쓰기 없이 반환하는 읽기 전용 빠른 경로(`@Transactional(readOnly=true)` 조회 후 변경 시에만 쓰기 메서드), ACTIVE 결과 짧은 로컬 캐시.

## P-23 trusted-proxies 범위
- 분류: 설정 / 심각도: Low
- 근거: `gateway-server/src/main/resources/application.yml:20-24` — 주석 "로컬 dev 전용"이나 base 파일이라 docker 프로필에도 적용(사설대역 전체 신뢰).
- 제안(S): docker 프로필에서는 nginx 컨테이너 대역(또는 고정 IP)만 신뢰하도록 `application-docker.yml`에서 덮어쓰기, 주석 정정.

## P-24 (보류 모듈) migration-service — 치명 아님, 기록만
- 근거: `migration-service/src/main/resources/application.yml:110-112` dev 프로필 `platform.jwt.issuer: http://localhost:19000` — dev auth-server는 issuer를 9000으로 유지(`auth-server/.../application-dev.yml:2`)하므로 dev 오프셋에서 migration API 전부 401. `ConfluenceDcClient` 원본 URL에 내부 주소 차단(SSRF 가드) 없음(관리자 전용이라 영향 제한). gRPC deadline도 없음(P-04와 동일 패턴).
- 제안: 보류 해제 시 issuer 기본값 9000으로 정정, 사설/루프백 주소 차단, deadline 추가.

---

## 권장 처리 순서
1. P-01·P-02·P-03 (권한·세션 무결성) — 한 묶음, auth-server·org-service·gateway 동시 변경.
2. P-04·P-05·P-10 (장애 전파·타임아웃) — common-starter 0.17.0 발행에 P-06·P-07·P-11을 같이 실어 전 서비스 1회 갱신.
3. P-08·P-09·P-12 (게이트웨이·내부망 하드닝).
4. 문서/버전 정리 P-13~P-17 (P-13은 사용자 결정 선행).

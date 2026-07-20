---
tags: [msa, template, security, auth, refresh-token, hardening, testcontainers]
작성일: 2026-07-19
상태: 구현 완료 (feature/auth-hardening 푸시됨, main 머지 전)
---

# 16단계 — auth-server RT 하드닝: 동시성·롤백 버그·수명 관리

상위: [[00 개요 — 전체 구조]] · 관련: [[01 인증 플랫폼 (Keycloak BFF + 자체 JWT)]], [[07 보안 감사 + 하드닝 (2026-07-02)]] · 설계: [[2026-07-19-auth-server-hardening-design]] · 계획: [[2026-07-19-auth-server-hardening]]

[[auth-server 개선점 정리]](2026-07-18 분석)의 코드 완결 항목 6개 + 구현 중 신규 발견 1건을 서브에이전트 주도 개발(태스크별 구현→리뷰 게이트)로 처리했다. 브랜치 `feature/auth-hardening`, 9커밋, **테스트 43/43 green** (H2 슬라이스 + Testcontainers 실 Postgres).

## 신규 발견 — 문서에 없던 P0 버그

**`revokeFamily` 롤백 무효화**: `ReuseDetectedException`(RuntimeException)이 `@Transactional rotate()`에서 전파되면 직전의 가족 폐기가 **롤백**됐다. 즉 재사용 탐지가 발동해도 도둑의 살아있는 토큰은 계속 유효 — 탐지 메커니즘이 통째로 무력이었다. 기존 H2 테스트가 통과한 이유는 `@DataJpaTest`가 전체를 한 트랜잭션으로 묶어 롤백 전 변경이 보였기 때문(운영과 다름).

- 수정: `@Transactional(noRollbackFor = ReuseDetectedException.class)` — 예외에도 폐기 커밋.
- 검증: 실 PG에서 예외 후 **별도 트랜잭션** 조회로 폐기 커밋 확인(회귀 방지 테스트).
- **교훈**: "보안 기능이 있다 ≠ 트랜잭션 경계에서도 성립한다" — 07단계의 "경쟁 조건에서도 성립한다"의 자매. 슬라이스 테스트의 트랜잭션 동작은 운영과 다를 수 있다.

## 적용된 강화 (커밋순)

| 커밋 | 내용 |
|---|---|
| `a2ddc2a` | 롤백 버그 수정(noRollbackFor) + 예외에 userId/familyId 페이로드 |
| `5a04ce5` | refresh 실패 3분기 — 도난 WARN(userId/familyId), 경쟁 패배는 쿠키 보존, 응답 body는 3분기 동일(정보 노출 최소화) |
| `139b56c` | V3 마이그레이션(`replaced_at`, `family_created_at`+백필) + `markRotated` 조건부 UPDATE |
| `10dd240` | **rotate() 재설계** — 원자 선점 + grace 30초 + enabled 검사 + 절대 세션 상한 90일 |
| `0b9d324` | Testcontainers 실 PG 검증 — 동시 rotate 1승 1패, grace 관용, noRollbackFor 커밋, V3 실측 |
| `fdcc020` | KC 호출 타임아웃 connect 2s/read 3s — 로그아웃 + 로그인 토큰 교환 |
| `add548c` | RT 청소 일 배치 — 가족 단위 삭제, 보존 기간=탐지 유효 기간 |
| `e65212c` | 청소 회귀 방지 보강 — 자체 만료 지난 증거물 행도 가족 생존 시 보존 검증 |
| `f05bdb7` | README 환경변수 표 반영 |

### rotate() 재설계 핵심

- check-and-set을 `UPDATE ... WHERE id=? AND revoked=false` **원자 연산 1번**으로 — 동시 요청 중 정확히 하나만 선점. 락 없음, 격리수준 무관.
- **grace period(30초)는 원자화와 분리 불가능한 세트**: 원자화만 하면 멀티탭 경쟁 패자가 도난으로 오판돼 가족 전멸(정상 사용자 강제 로그아웃). grace 이내 재사용 = `ConcurrentRotationException`(가족 생존, **쿠키 삭제 안 함** — 승자 탭이 심은 새 쿠키 보호), grace 경과 = 도난 판정(가족 전멸 + WARN). Auth0 등 상용 IdP leeway 방식.
- 검사 순서: 만료/절대상한/enabled 검사를 **선점 전에** — 실패할 요청이 토큰을 소모하지 않도록.
- `enabled=false` 사용자는 rotate 거부(기존엔 차단해도 14일간 refresh 가능했음).
- 절대 상한: `family_created_at + 90일` — sliding 만료의 영구 세션 차단.

### 청소 배치의 함정 회피

폐기된 옛 RT 행은 재사용 탐지의 **증거물** — 성급히 지우면 도난 토큰이 "알 수 없는 토큰"(단순 401)으로 강등돼 가족 폐기가 미발동한다. 그래서 행 단위가 아닌 **가족 단위**(`MAX(expires_at) < now - 14일`)로만 삭제. 회귀 방지 테스트가 "자체 만료는 지났지만 가족은 살아있는 증거물 행"의 보존을 판별한다(행 단위 삭제로 퇴행하면 실패).

## 구현 중 확인된 계획-현실 차이 (Boot 4.0.6 실측)

- **토큰 교환 타임아웃**: `spring.http.client.*` yml 프로퍼티는 바인딩은 되지만(deprecated, 정식은 `spring.http.clients.*`) **Spring Security 7의 토큰 교환 클라이언트에 전혀 연결되지 않음**(바이트코드 확인 — `OAuth2LoginConfigurer`가 Boot 자동구성과 무관하게 bare `RestClient.builder()` 사용). 유일하게 실효 있는 경로는 `OAuth2AccessTokenResponseClient` 빈 등록 — SecurityConfig에 구현.
- `ClientHttpRequestFactorySettings`는 Boot 4.0.6에서 별도 모듈(`spring-boot-http-client`)이라 classpath에 없음 → `JdkClientHttpRequestFactory`(spring-web 내장)로 구현.
- Testcontainers: Boot 4.0.6 BOM이 TC 2.0.5 관리 — 아티팩트명이 `testcontainers-junit-jupiter`/`testcontainers-postgresql`로 변경됨(1.x의 `junit-jupiter`/`postgresql` 아님).

## 신규 환경변수 (교차 계약 추가분)

| env | 기본값 | 용도 |
|---|---|---|
| `ROTATION_GRACE_SECONDS` | `30` | 경쟁(멀티탭) vs 도난 구분 창 |
| `SESSION_ABSOLUTE_TTL_SECONDS` | `7776000` (90일) | 가족 생성 기준 절대 세션 상한 |
| `TOKEN_CLEANUP_CRON` | `0 0 4 * * *` | RT 청소 배치 주기 |

## 테스트 체계 (신규 자산)

- **Testcontainers 실 PG 테스트**(`RefreshTokenServicePostgresTest`, 6개): H2가 못 보는 것만 — 진짜 2스레드 경쟁(CyclicBarrier), noRollbackFor 커밋(별도 트랜잭션 조회), Flyway V1~V3 실측, 청소 배치 판별. `@ActiveProfiles("test")`를 **쓰지 않는 것**이 핵심(test 프로필은 H2+Flyway off로 바꿔버림).
- 기존 H2 슬라이스 테스트는 유지, grace 주입(`graceSeconds=0` → 즉시 도난 판정)으로 보정.

## 리뷰 결과

**태스크별 게이트**: 전 태스크 스펙 준수 ✅ / 품질 승인. 리뷰 루프 1회 발동: 청소 배치 테스트가 가족 단위 vs 행 단위 삭제를 판별 못 한다는 Important 지적(계획의 테스트 설계 자체 허점) → 판별 테스트 추가로 해소.

**최종 전체 브랜치 리뷰: READY** — Critical 0 / Important 0, 스펙 7항목 전부 ✅, 43/43 테스트 강제 재실행 그린. 교차 태스크 합성도 확인: 타임아웃 빈이 docker split-horizon 프로필(수동 ClientRegistration)에도 적용됨, issue가 `family_created_at`을 심고 rotate가 승계하는 흐름 정합.

수용한 Minor(머지 게이트 아님): AuthController 로거 FQN 스타일, ttl `@Value` 3중 주입(리포 관례 — 추후 `@ConfigurationProperties` 후보), IllegalArgument 분기 전용 테스트 없음, grace 판정이 가족 전멸 여부보다 먼저라 로그아웃 직후 30초 내 재사용이 "경쟁"으로 로깅될 수 있음(토큰 획득 불가라 실해 없음), 타임아웃 테스트의 wall-clock 어서션.

**프론트 계약 후속 메모(myFront에 전달할 것)**: 경쟁 패자 401은 쿠키를 보존하므로, 프론트가 refresh 401을 "즉시 로그아웃"으로 처리하면 멀티탭 이점이 줄어든다 — **401 시 새 쿠키로 1회 재시도** 후 로그아웃 판단이 이상적.

## 다음 로드맵 (운영 전, [[auth-server 개선점 정리]] 잔여분)

1. **#3 키 관리**: 개인키 시크릿 매니저/env 주입, KeyProvider 키 리스트화(서명 1+유예 N) + `JWKSource` 디코더로 kid 매칭 → 무중단 키 로테이션.
2. **#8 스케일아웃**: 키 중앙 주입 + OIDC state 무상태화(쿠키 기반 `AuthorizationRequestRepository`).
3. P2: JWKS 캐시 헤더, `/api/me` role 우선순위 규칙, PKCE, rate limit(게이트웨이 레벨) 등.

---
tags: [backend, auth, security, jwt, refresh-token, design, spec]
updated: 2026-07-19
---

# auth-server 보안 강화 — 설계

- **작성일:** 2026-07-19
- **상태:** 승인됨 (구현 계획 작성 전)
- **홈:** `C:\MSA_TEMPLATE\auth-server`
- **기반:** [[auth-server 개선점 정리]] (2026-07-18 심층 분석) — 코드 대조 검증 결과 전 항목 사실로 확인.

## 0. 범위

**이번 구현 (코드+마이그레이션으로 완결되는 항목):**

| # | 항목 | 원문서 순위 |
|---|---|---|
| §2 | `rotate()` 동시성 — 조건부 UPDATE + grace period | P0-1 |
| §3 | 재사용 탐지 로깅 + 예외 페이로드 | P0-2 |
| §2 | `enabled` 검사 | P0-4 |
| §2 | RT 절대 세션 상한 (sliding 만료 보완) | P1-5 |
| §4 | HTTP 타임아웃 (로그아웃 + 로그인 토큰 교환) | P1-6 |
| §5 | RT 청소 배치 (가족 단위) | P1-7 |
| §1 | **[신규 발견]** `revokeFamily` 롤백 버그 | 원문서에 없음 |

**범위 외:** #3 키 관리, #8 스케일아웃 (시크릿 매니저/Redis 등 인프라 결정 필요 — §8에 방향만 기록), P2 항목.

**테스트 기반:** Testcontainers(Postgres) 도입 — 동시성·벌크·트랜잭션 동작은 실 DB로 검증. 기존 H2 테스트는 유지.

## 1. [신규 발견] revokeFamily 롤백 버그 🔴

`ReuseDetectedException`은 `RuntimeException`(`ReuseDetectedException.java:3`)이고 `rotate()`는 `@Transactional`. 운영에서는 요청마다 독립 트랜잭션이므로 **예외 전파 순간 직전의 `revokeFamily`가 롤백**된다 — 도난 탐지 시 401만 반환되고 가족 폐기는 DB에 남지 않아, 도둑의 살아있는 토큰이 계속 유효하다.

기존 테스트 `reuseDetectionRevokesWholeFamily`가 통과하는 이유: `@DataJpaTest`가 테스트 전체를 한 트랜잭션으로 묶어 롤백 전까지 변경이 보이기 때문. 운영 동작과 다르다.

**수정:** `@Transactional(noRollbackFor = ReuseDetectedException.class)` + Postgres 실 DB에서 "예외 후 별도 트랜잭션에서 가족 폐기가 커밋돼 있는지" 검증하는 테스트.

**교훈:** 원문서 #1의 교훈("보안 기능이 있다 ≠ 경쟁 조건에서도 성립한다")의 자매 — "보안 기능이 있다 ≠ 트랜잭션 경계에서도 성립한다". 슬라이스 테스트의 트랜잭션 동작은 운영과 다를 수 있다.

## 2. rotate() 재설계 — 동시성 + enabled + 절대 상한 통합

### 2.1 데이터 모델 (V3 마이그레이션)

`refresh_tokens`에 컬럼 2개 추가:

```sql
ALTER TABLE refresh_tokens ADD COLUMN replaced_at TIMESTAMP;          -- grace 판정 기준
ALTER TABLE refresh_tokens ADD COLUMN family_created_at TIMESTAMP;   -- 절대 상한 기준
-- 백필: 가족별 최초 생성 시각
UPDATE refresh_tokens rt SET family_created_at = f.min_created
FROM (SELECT family_id, MIN(created_at) AS min_created FROM refresh_tokens GROUP BY family_id) f
WHERE rt.family_id = f.family_id;
ALTER TABLE refresh_tokens ALTER COLUMN family_created_at SET NOT NULL;
```

테스트 프로필은 H2 + `ddl-auto: create-drop` + Flyway off → 엔티티 변경만으로 반영. V3 SQL 자체는 Testcontainers 테스트(Flyway on)에서 실 Postgres로 검증.

### 2.2 흐름

```
rotate(raw):
  current = findByTokenHash(sha256(raw))        // 없으면 무효 401
  if current.revoked:
      if current.replacedBy != null → handleReuse(current)
      throw 무효                                 // 가족 폐기로 부수 무효화된 토큰
  if 만료 → throw 무효
  if now > current.familyCreatedAt + 절대상한 → throw 무효   // #5
  user 조회; if !user.enabled → throw 무효                    // #4
  next = 선생성(ID 확정, familyCreatedAt·kc토큰 승계)
  n = markRotated(current.id, next.id, now)      // 조건부 UPDATE (원자 선점)
  if n == 0:                                     // 경쟁 패배
      fresh = findById(current.id)               // clearAutomatically로 컨텍스트가 비워졌으므로 재조회 필수
      handleReuse(fresh)                         // 승자가 심은 replaced_at 기준으로 grace 판정
  next INSERT, Rotated 반환

handleReuse(t):
  if t.replacedAt != null && now - t.replacedAt <= grace(30초):
      throw ConcurrentRotationException          // 멀티탭 경쟁 — 가족 생존
  revokeFamily(t.familyId)
  throw ReuseDetectedException(userId, familyId) // 도난 판정
```

Repository 추가:

```java
@Modifying(flushAutomatically = true, clearAutomatically = true)
@Query("update RefreshToken t set t.revoked = true, t.replacedBy = :nextId, t.replacedAt = :now " +
       "where t.id = :id and t.revoked = false")
int markRotated(@Param("id") UUID id, @Param("nextId") UUID nextId, @Param("now") Instant now);
```

### 2.3 설계 결정과 근거

- **조건부 UPDATE 선택** (비관적 락 대신): 락 없음·비차단·격리수준 무관, 패자는 영향 행 0으로 즉시 감지. `RefreshToken.id`가 앱 생성 UUID(`RefreshToken.java:53`)라 next ID 선확정 가능 — INSERT 전에 선점이 가능한 전제 성립.
- **grace period는 선택이 아니라 필수**: 원자화만 하면 "탐지 우회"(공격)는 막지만, 경쟁 패자(멀티탭 탭 B)를 재사용 경로로 보내면 가족 전멸 오탐이 확정 재현된다. grace 이내 재사용은 경쟁으로 관용(Auth0 등 상용 IdP leeway 방식). grace 이내엔 공격자도 관용되는 트레이드오프는 수용.
- **grace 판정은 두 경로 공통**: 첫 조회에서 revoked 발견 / 조건부 UPDATE 0행 — 모두 `handleReuse`로 수렴.
- **enabled·절대상한 검사는 선점 전에**: 실패할 요청이 토큰을 선점(소모)하지 않도록.
- `@Transactional(noRollbackFor = ReuseDetectedException.class)` (§1).

### 2.4 설정 추가 (application.yml `platform.*`)

| 키 | 기본값 | 용도 |
|---|---|---|
| `rotation-grace-seconds` | 30 | 경쟁/도난 구분 창 |
| `session-absolute-ttl-seconds` | 7776000 (90일) | 가족 생성 기준 절대 세션 상한 |

## 3. 컨트롤러 응답·로깅 정책 (#2)

`ReuseDetectedException`에 `userId`, `familyId` 필드 추가(throw 지점에서 `current`가 손에 있음). `AuthController.refresh()` catch 3분기:

| 예외 | 로그 | 응답 | 쿠키 |
|---|---|---|---|
| `ReuseDetectedException` | `WARN` (userId, familyId) — 계정 탈취 의심 | 401 `invalid_refresh_token` | 삭제 |
| `ConcurrentRotationException` | `DEBUG` | 401 `invalid_refresh_token` | **삭제 안 함** |
| `IllegalArgumentException` | `DEBUG` | 401 `invalid_refresh_token` | 삭제 |

- 응답 body는 3분기 동일 — 정보 노출 최소화 원칙 유지.
- 경쟁 패배 시 쿠키를 삭제하면 승자(탭 A)가 방금 심은 새 쿠키를 지워버린다. 쿠키는 브라우저 공유이므로 탭 B는 새 쿠키로 재시도하면 성공 — 클라이언트 수정 불필요.
- 메트릭 카운터(`reuse_detected_total` 등)는 이번 범위에서 로그까지만, 카운터는 게이트웨이/모니터링 정비 때.

## 4. HTTP 타임아웃 (#6 + 로그인 경로)

- **`KeycloakLogoutClient`**: connect 2초 / read 3초 설정한 request factory로 `RestClient` 생성 (`RestClient.create()` 대체). best-effort 철학 유지 — 예외는 삼키되 행(hang)은 타임아웃으로 차단.
- **로그인 토큰 교환** (Spring Security 자동구성 클라이언트): `spring.http.client.connect-timeout` / `read-timeout` 프로퍼티로 Boot 자동구성 HTTP 클라이언트에 타임아웃 적용. 구현 시 실제 적용 여부 확인(통합 테스트 또는 수동 검증), 안 먹히면 커스텀 token response client 빈으로 대체.

## 5. RT 청소 배치 (#7)

- `@EnableScheduling` + `TokenCleanupJob` 컴포넌트, `@Scheduled(cron = "${platform.token-cleanup-cron:0 0 4 * * *}")`.
- **가족 단위 삭제**: `MAX(expires_at) < now - 14일(RT TTL 버퍼)`인 가족만 통째로 삭제. 폐기된 옛 행은 재사용 탐지의 증거물 — **보존 기간 = 탐지 유효 기간** 원칙(원문서 #7의 함정 회피).

```sql
DELETE FROM refresh_tokens WHERE family_id IN (
  SELECT family_id FROM refresh_tokens GROUP BY family_id HAVING MAX(expires_at) < :cutoff)
```

- 삭제 건수 `INFO` 로그.

## 6. 테스트 전략

**의존성 추가:** `spring-boot-testcontainers`, `org.testcontainers:junit-jupiter`, `org.testcontainers:postgresql` (testImplementation/testRuntimeOnly).

**신규: Postgres 실 DB 테스트** (`@ServiceConnection` PostgreSQLContainer, Flyway on → V1~V3 실측):
1. 같은 rawToken 동시 rotate 2개(`ExecutorService` + latch) → 살아있는 토큰 정확히 1개, 가족 생존, 예외는 `ConcurrentRotationException`만
2. grace 이내 재사용 → `ConcurrentRotationException`, 가족 생존
3. grace 경과 재사용(`replaced_at` 과거로 조작) → `ReuseDetectedException` + **별도 트랜잭션에서 가족 폐기 커밋 확인** (§1 회귀 방지)
4. `enabled=false` → rotate 거부 / 절대 상한 초과(`family_created_at` 과거 조작) → rotate 거부
5. 청소 배치 — 죽은 가족 삭제, 산 가족(폐기 행 포함) 보존

**기존 H2 테스트:** 유지하되 동작 변경 반영 — 재사용 판정 테스트는 grace를 0으로 주입하거나 `replaced_at` 조작으로 보정. 새 `@Value` 필드는 기존 방식대로 `ReflectionTestUtils` 주입.

## 7. 에러 처리 요약

- 모든 rotate 실패는 클라이언트에 401 + 동일 body. 서버 측에서만 로그 레벨·쿠키 처리로 구분.
- 배치 실패는 로그만 (다음 주기 재시도, 데이터 축적은 점진적 문제).
- 타임아웃 발생 시: 로그아웃은 best-effort 유지(경고 로그), 로그인 토큰 교환은 OAuth2 표준 에러 흐름.

## 8. 범위 외 항목의 방향 (기록만)

- **#3 키 관리**: 운영 전 시크릿 매니저/env 주입(`platform.jwk-path` 주입 지점 활용), KeyProvider를 키 리스트(서명 1 + 검증 유예 N)로 확장, 디코더를 `JWKSource` 기반으로 교체해 kid 매칭. AT TTL 15분이라 유예 하루면 충분.
- **#8 스케일아웃**: 키 중앙 주입(#3과 동일 해법) + OIDC state 무상태화(쿠키 기반 `AuthorizationRequestRepository` 권장 — 코드로 해결 가능, 인프라 협의 후 착수).

## 9. 구현 순서 (계획 문서에서 상세화)

1. §1 noRollbackFor + §3 예외 페이로드/로깅 (기반 정비)
2. V3 마이그레이션 + 엔티티 확장
3. §2 rotate() 재설계 (Testcontainers 동시성 테스트와 함께 — TDD)
4. §4 타임아웃
5. §5 청소 배치
6. 문서 갱신 (원문서 상태 반영)

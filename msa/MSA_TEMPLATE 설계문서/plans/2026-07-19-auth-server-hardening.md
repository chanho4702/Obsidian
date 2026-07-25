# auth-server 보안 강화 Implementation Plan

> **상태: 전 태스크 구현 완료 (2026-07-19).** feature/auth-hardening 9커밋, 43/43 테스트 green, 최종 리뷰 READY. 실측 차이(타임아웃 빈 경로, TC 2.0.5 아티팩트명, 청소 테스트 보강)는 스펙 §10 참고.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** refresh token 회전의 동시성 결함·롤백 버그·미검사 항목을 수정하고, 절대 세션 상한·HTTP 타임아웃·청소 배치를 추가한다.

**Architecture:** 구조 변경 없는 지점 수정. `rotate()`를 조건부 UPDATE(원자 선점) + grace period(30초)로 재설계하고, 예외 3분기(도난/경쟁/무효)를 컨트롤러 응답 정책과 짝 맞춘다. 검증은 Testcontainers(Postgres 실 DB)로 한다.

**Tech Stack:** Spring Boot 4.0.6, Java 24, Spring Data JPA, Flyway, PostgreSQL, H2(기존 테스트), Testcontainers(신규), Lombok.

**Spec:** `C:\myBrain\내 로컬\msa\MSA_TEMPLATE 설계문서\specs\2026-07-19-auth-server-hardening-design.md`

## Global Constraints

- 홈 디렉터리: `C:\MSA_TEMPLATE\auth-server` (모든 상대경로의 기준)
- Spring Boot 4.0.6 / Java 24 — Boot 4 특이사항: `@DataJpaTest`는 `org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest`(새 패키지), `RestClient.Builder` 빈 자동구성 없음
- 테스트 프로필(`test`)은 H2 + `ddl-auto: create-drop` + Flyway off. **PG 테스트 클래스는 `@ActiveProfiles("test")를 쓰지 않는다**(Flyway로 V1~V3 실측이 목적)
- 클라이언트 응답은 모든 rotate 실패에서 동일: 401 + body `{"error":"invalid_refresh_token"}` — 분기는 로그·쿠키 처리로만
- 주석·로그·커밋 메시지는 기존 코드처럼 한국어. 커밋 메시지 말미에 `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>` 한 줄
- 테스트 실행: Git Bash에서 `./gradlew test` (Windows)
- 설정 키는 기존 스타일대로 env 오버라이드 가능하게: `${ENV_NAME:기본값}`

---

### Task 1: ReuseDetectedException 페이로드 + noRollbackFor 롤백 버그 수정

**Files:**
- Modify: `src/main/java/com/platform/authserver/token/ReuseDetectedException.java`
- Modify: `src/main/java/com/platform/authserver/token/RefreshTokenService.java:40-49`
- Test: `src/test/java/com/platform/authserver/token/RefreshTokenServiceTest.java`

**Interfaces:**
- Produces: `ReuseDetectedException(String message, Long userId, UUID familyId)` + `getUserId(): Long` + `getFamilyId(): UUID` — Task 2(컨트롤러 로깅)와 Task 4(재설계된 throw 지점)가 이 시그니처를 쓴다.
- Produces: `rotate()`에 `@Transactional(noRollbackFor = ReuseDetectedException.class)` — Task 5의 PG 테스트가 커밋 여부를 검증한다.

- [ ] **Step 1: 실패하는 테스트 작성** — `RefreshTokenServiceTest`에 추가:

```java
@Test
void reuseDetectedExceptionCarriesUserAndFamily() {
    ReflectionTestUtils.setField(service, "ttlSeconds", 1209600L);
    User u = newUser();
    var issued = service.issue(u, "kc-id-token", "kc-refresh-token");
    service.rotate(issued.rawToken());

    assertThatThrownBy(() -> service.rotate(issued.rawToken()))
            .isInstanceOfSatisfying(ReuseDetectedException.class, e -> {
                assertThat(e.getUserId()).isEqualTo(u.getId());
                assertThat(e.getFamilyId()).isNotNull();
            });
}
```

- [ ] **Step 2: 실패 확인**

Run: `./gradlew test --tests "com.platform.authserver.token.RefreshTokenServiceTest.reuseDetectedExceptionCarriesUserAndFamily"`
Expected: 컴파일 실패 — `getUserId()` 미정의.

- [ ] **Step 3: 예외 확장** — `ReuseDetectedException.java` 전체 교체:

```java
package com.platform.authserver.token;

import java.util.UUID;

/** RT 재사용 탐지 = 계정 탈취 의심. 로깅·알림용으로 소유자/가족 식별자를 실어 던진다. */
public class ReuseDetectedException extends RuntimeException {

    private final Long userId;
    private final UUID familyId;

    public ReuseDetectedException(String message, Long userId, UUID familyId) {
        super(message);
        this.userId = userId;
        this.familyId = familyId;
    }

    public Long getUserId() {
        return userId;
    }

    public UUID getFamilyId() {
        return familyId;
    }
}
```

- [ ] **Step 4: throw 지점·트랜잭션 수정** — `RefreshTokenService.java`의 `rotate()`:

`@Transactional` → `@Transactional(noRollbackFor = ReuseDetectedException.class)` 로 교체하고(**핵심 버그 수정** — 예외 롤백이 `revokeFamily`를 무효화하던 문제), throw를 다음으로 교체:

```java
throw new ReuseDetectedException("refresh token 재사용 탐지",
        current.getUserId(), current.getFamilyId());
```

주석도 한 줄 보강: `// noRollbackFor: 이 예외는 가족 폐기를 커밋한 채 전파되어야 한다(롤백되면 탐지가 무효).`

- [ ] **Step 5: 전체 테스트 통과 확인**

Run: `./gradlew test`
Expected: BUILD SUCCESSFUL (기존 테스트 포함 전부 PASS)

- [ ] **Step 6: Commit**

```bash
git add -A src/
git commit -m "fix(token): 재사용 탐지 시 revokeFamily가 예외 롤백에 무효화되던 버그 수정 + 예외에 userId/familyId 페이로드

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: ConcurrentRotationException + 컨트롤러 3분기 (로깅·쿠키 정책)

**Files:**
- Create: `src/main/java/com/platform/authserver/token/ConcurrentRotationException.java`
- Modify: `src/main/java/com/platform/authserver/auth/AuthController.java`
- Test: `src/test/java/com/platform/authserver/auth/AuthControllerTest.java`

**Interfaces:**
- Consumes: Task 1의 `ReuseDetectedException(message, userId, familyId)`
- Produces: `ConcurrentRotationException(String message)` (RuntimeException) — Task 4의 `rotate()`가 grace 이내 경쟁 패배 시 던진다.
- Produces: 컨트롤러 응답 정책 — 도난: 401+쿠키삭제+WARN / 경쟁: 401+**쿠키유지**+DEBUG / 무효: 401+쿠키삭제+DEBUG. body는 셋 다 `{"error":"invalid_refresh_token"}`.

- [ ] **Step 1: 실패하는 테스트 작성** — `AuthControllerTest`에 추가 (import: `com.platform.authserver.token.ConcurrentRotationException`, `com.platform.authserver.token.ReuseDetectedException`, `java.util.UUID`):

```java
@Test
void reuseDetectionReturns401AndDeletesCookie() throws Exception {
    when(refreshTokenService.rotate(eq("stolen-rt")))
            .thenThrow(new ReuseDetectedException("재사용", 42L, UUID.randomUUID()));

    mvc.perform(post("/api/auth/refresh").cookie(new Cookie("refresh_token", "stolen-rt")))
            .andExpect(status().isUnauthorized())
            .andExpect(header().string("Set-Cookie", org.hamcrest.Matchers.containsString("refresh_token=;")))
            .andExpect(jsonPath("$.error").value("invalid_refresh_token"));
}

@Test
void concurrentRotationReturns401ButKeepsCookie() throws Exception {
    // 경쟁 패배(멀티탭)는 승자가 심은 새 쿠키를 지우면 안 된다 — Set-Cookie 자체가 없어야 함
    when(refreshTokenService.rotate(eq("raced-rt")))
            .thenThrow(new ConcurrentRotationException("경쟁 패배"));

    mvc.perform(post("/api/auth/refresh").cookie(new Cookie("refresh_token", "raced-rt")))
            .andExpect(status().isUnauthorized())
            .andExpect(header().doesNotExist("Set-Cookie"))
            .andExpect(jsonPath("$.error").value("invalid_refresh_token"));
}
```

- [ ] **Step 2: 실패 확인**

Run: `./gradlew test --tests "com.platform.authserver.auth.AuthControllerTest"`
Expected: 컴파일 실패 — `ConcurrentRotationException` 미정의.

- [ ] **Step 3: 예외 클래스 생성** — `ConcurrentRotationException.java`:

```java
package com.platform.authserver.token;

/**
 * rotate 경쟁 패배(grace 이내 재사용) — 멀티탭 등 정상 시나리오로 관용한다.
 * 도난(ReuseDetectedException)과 달리 가족을 살려두고, 쿠키도 지우지 않는다.
 */
public class ConcurrentRotationException extends RuntimeException {
    public ConcurrentRotationException(String message) {
        super(message);
    }
}
```

- [ ] **Step 4: 컨트롤러 3분기** — `AuthController.java`에 Logger 추가 후 `refresh()`의 catch를 교체:

```java
// 클래스 필드에 추가
private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(AuthController.class);
```

```java
} catch (ReuseDetectedException e) {
    // 최고 등급 보안 이벤트 — 탈취 의심. 응답은 다른 실패와 동일(정보 노출 최소화).
    log.warn("RT 재사용 탐지 — 계정 탈취 의심. userId={}, familyId={}", e.getUserId(), e.getFamilyId());
    return unauthorizedWithCookieDelete();
} catch (ConcurrentRotationException e) {
    // 멀티탭 경쟁 패배 — 승자가 심은 새 쿠키를 지우면 안 되므로 Set-Cookie 없이 401만.
    log.debug("RT rotate 경쟁 패배: {}", e.getMessage());
    return ResponseEntity.status(401).body(Map.of("error", "invalid_refresh_token"));
} catch (IllegalArgumentException e) {
    log.debug("무효 RT: {}", e.getMessage());
    return unauthorizedWithCookieDelete();
}
```

공통 헬퍼를 클래스에 추가:

```java
private ResponseEntity<?> unauthorizedWithCookieDelete() {
    return ResponseEntity.status(401)
            .header(HttpHeaders.SET_COOKIE, cookieFactory.deleteCookie().toString())
            .body(Map.of("error", "invalid_refresh_token"));
}
```

import에 `com.platform.authserver.token.ConcurrentRotationException` 추가.

- [ ] **Step 5: 통과 확인**

Run: `./gradlew test --tests "com.platform.authserver.auth.AuthControllerTest"`
Expected: PASS (기존 3개 + 신규 2개)

- [ ] **Step 6: Commit**

```bash
git add -A src/
git commit -m "feat(auth): refresh 실패 3분기 — 도난 WARN 로깅, 경쟁 패배는 쿠키 보존, 응답 body는 동일 유지

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: V3 마이그레이션 + 엔티티 확장 + markRotated (원자 선점 쿼리)

**Files:**
- Create: `src/main/resources/db/migration/V3__rotation_grace_and_absolute_expiry.sql`
- Modify: `src/main/java/com/platform/authserver/token/RefreshToken.java`
- Modify: `src/main/java/com/platform/authserver/token/RefreshTokenRepository.java`
- Modify: `src/main/java/com/platform/authserver/token/RefreshTokenService.java` (`persist`/`issue` — 생성자 변경 반영)
- Modify: `src/main/java/com/platform/authserver/user/User.java` (`enabled`에 `@Setter` — Task 4·5 테스트용)
- Test: `src/test/java/com/platform/authserver/token/RefreshTokenServiceTest.java`

**Interfaces:**
- Produces: `RefreshToken` 생성자 — `RefreshToken(Long userId, String tokenHash, UUID familyId, String kcIdToken, String kcRefreshToken, Instant expiresAt, Instant familyCreatedAt)` (마지막 파라미터 추가)
- Produces: 엔티티 필드 `replacedAt: Instant`(@Setter), `familyCreatedAt: Instant`(@Setter) + getter
- Produces: `RefreshTokenRepository.markRotated(UUID id, UUID nextId, Instant now): int` — Task 4의 rotate()가 선점에 쓴다. 영향 행 1=선점 성공, 0=경쟁 패배.
- Produces: `User.setEnabled(boolean)`

- [ ] **Step 1: 실패하는 테스트 작성** — `RefreshTokenServiceTest`에 추가 (import `java.time.Instant`, `java.util.UUID`):

```java
@Test
void markRotatedClaimsTokenExactlyOnce() {
    ReflectionTestUtils.setField(service, "ttlSeconds", 1209600L);
    User u = newUser();
    var issued = service.issue(u, "kc-id-token", "kc-refresh-token");
    RefreshToken t = tokenRepository.findByTokenHash(RefreshTokenService.sha256(issued.rawToken())).orElseThrow();

    int first = tokenRepository.markRotated(t.getId(), UUID.randomUUID(), Instant.now());
    int second = tokenRepository.markRotated(t.getId(), UUID.randomUUID(), Instant.now());

    assertThat(first).isEqualTo(1);   // 선점 성공
    assertThat(second).isZero();      // 이미 선점됨 — 원자성의 근거
}
```

- [ ] **Step 2: 실패 확인**

Run: `./gradlew test --tests "com.platform.authserver.token.RefreshTokenServiceTest.markRotatedClaimsTokenExactlyOnce"`
Expected: 컴파일 실패 — `markRotated` 미정의.

- [ ] **Step 3: V3 마이그레이션 작성** — `V3__rotation_grace_and_absolute_expiry.sql`:

```sql
-- replaced_at: rotate 선점 시각 — grace period(경쟁 vs 도난) 판정 기준
ALTER TABLE refresh_tokens ADD COLUMN replaced_at TIMESTAMP;

-- family_created_at: 가족 최초 생성 시각 — sliding 만료 보완용 절대 세션 상한 기준
ALTER TABLE refresh_tokens ADD COLUMN family_created_at TIMESTAMP;

-- 기존 행 백필: 가족별 최초 생성 시각
UPDATE refresh_tokens rt SET family_created_at = f.min_created
FROM (SELECT family_id, MIN(created_at) AS min_created
      FROM refresh_tokens GROUP BY family_id) f
WHERE rt.family_id = f.family_id;

ALTER TABLE refresh_tokens ALTER COLUMN family_created_at SET NOT NULL;
```

- [ ] **Step 4: 엔티티 확장** — `RefreshToken.java`에 필드 추가(기존 `revoked` 필드 아래):

```java
// rotate 선점 시각. grace 판정 기준 — markRotated 쿼리가 채우고, 테스트만 setter로 조작한다.
@Setter
@Column(name = "replaced_at")
private Instant replacedAt;

// 가족 최초 생성 시각. rotate 시 승계 — 절대 세션 상한 판정 기준.
@Setter
@Column(name = "family_created_at", nullable = false)
private Instant familyCreatedAt;
```

생성자 교체(파라미터 1개 추가):

```java
public RefreshToken(Long userId, String tokenHash, UUID familyId,
                    String kcIdToken, String kcRefreshToken, Instant expiresAt, Instant familyCreatedAt) {
    this.id = UUID.randomUUID();
    this.userId = userId;
    this.tokenHash = tokenHash;
    this.familyId = familyId;
    this.kcIdToken = kcIdToken;
    this.kcRefreshToken = kcRefreshToken;
    this.expiresAt = expiresAt;
    this.familyCreatedAt = familyCreatedAt;
    this.createdAt = Instant.now();
}
```

- [ ] **Step 5: 리포지토리에 markRotated 추가** — `RefreshTokenRepository.java`:

```java
// 조건부 UPDATE = check-and-set의 DB 원자 연산. 동시 rotate 중 정확히 한 요청만 1을 받는다.
@Modifying(flushAutomatically = true, clearAutomatically = true)
@Query("update RefreshToken t set t.revoked = true, t.replacedBy = :nextId, t.replacedAt = :now " +
       "where t.id = :id and t.revoked = false")
int markRotated(@Param("id") UUID id, @Param("nextId") UUID nextId, @Param("now") Instant now);
```

import에 `java.time.Instant` 추가.

- [ ] **Step 6: 생성자 호출부 반영** — `RefreshTokenService.java`:

`persist()` 시그니처에 `Instant familyCreatedAt` 추가, 생성자에 전달:

```java
private RefreshToken persist(Long userId, String raw, UUID familyId,
                             String kcIdToken, String kcRefreshToken, Instant familyCreatedAt) {
    RefreshToken token = new RefreshToken(
            userId, sha256(raw), familyId, kcIdToken, kcRefreshToken,
            Instant.now().plusSeconds(ttlSeconds), familyCreatedAt);
    return tokenRepository.save(token);
}
```

- `issue()`: `persist(user.getId(), raw, familyId, kcIdToken, kcRefreshToken, Instant.now())` — 새 가족은 지금이 최초.
- `rotate()`: `persist(user.getId(), newRaw, current.getFamilyId(), current.getKcIdToken(), current.getKcRefreshToken(), current.getFamilyCreatedAt())` — 승계. (rotate 본체 재설계는 Task 4에서.)

- [ ] **Step 7: User에 setter 추가** — `User.java:39`의 `enabled` 필드에 `@Setter` 추가 (차단 테스트·추후 어드민 기능용):

```java
@Setter
@Column(nullable = false)
private boolean enabled = true;
```

- [ ] **Step 8: 전체 테스트 통과 확인**

Run: `./gradlew test`
Expected: BUILD SUCCESSFUL. (H2는 `ddl-auto: create-drop`이라 엔티티 변경 자동 반영, V3 SQL은 Task 5의 PG 테스트에서 실측된다.)

- [ ] **Step 9: Commit**

```bash
git add -A src/
git commit -m "feat(token): V3 — replaced_at(grace 판정)·family_created_at(절대 상한) + markRotated 원자 선점 쿼리

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: rotate() 재설계 — 조건부 UPDATE + grace + enabled + 절대 상한

**Files:**
- Modify: `src/main/java/com/platform/authserver/token/RefreshTokenService.java`
- Modify: `src/main/resources/application.yml` (`platform.*` 키 2개 추가)
- Test: `src/test/java/com/platform/authserver/token/RefreshTokenServiceTest.java`

**Interfaces:**
- Consumes: Task 1 `ReuseDetectedException(msg, userId, familyId)`, Task 2 `ConcurrentRotationException(msg)`, Task 3 `markRotated`/엔티티 필드/`User.setEnabled`
- Produces: `rotate()` 최종 동작 — Task 5의 PG 테스트가 검증하는 계약:
  - grace(기본 30초) 이내 재사용 → `ConcurrentRotationException`, 가족 생존
  - grace 경과 재사용 → `ReuseDetectedException`, 가족 폐기 **커밋**
  - `!user.enabled` / 절대 상한(기본 90일) 초과 / 만료 → `IllegalArgumentException`
- Produces: `@Value` 필드명 — `graceSeconds`, `absoluteTtlSeconds` (테스트가 ReflectionTestUtils로 주입)

- [ ] **Step 1: 실패하는 테스트 작성** — `RefreshTokenServiceTest` 전체 교체 (기존 검증 유지 + 신규 시나리오; grace 주입으로 동작 제어):

```java
package com.platform.authserver.token;

import com.platform.authserver.user.User;
import com.platform.authserver.user.UserRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest; // Spring Boot 4: 새 패키지(+ testImpl spring-boot-data-jpa-test)
import org.springframework.context.annotation.Import;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.util.ReflectionTestUtils;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.*;

@DataJpaTest
@ActiveProfiles("test")
@Import(RefreshTokenService.class)
class RefreshTokenServiceTest {

    @Autowired RefreshTokenService service;
    @Autowired UserRepository userRepository;
    @Autowired RefreshTokenRepository tokenRepository;

    @BeforeEach
    void injectConfig() {
        // 슬라이스 테스트라 @Value 미적용 — 직접 주입. grace=0: 재사용 즉시 도난 판정(기존 동작 검증용).
        ReflectionTestUtils.setField(service, "ttlSeconds", 1209600L);
        ReflectionTestUtils.setField(service, "graceSeconds", 0L);
        ReflectionTestUtils.setField(service, "absoluteTtlSeconds", 7776000L);
    }

    private User newUser() {
        User u = new User("kc-sub-1");
        u.setRoles(List.of("USER"));
        return userRepository.save(u);
    }

    @Test
    void rotateReturnsNewTokenAndRevokesOld() {
        User u = newUser();
        var issued = service.issue(u, "kc-id-token", "kc-refresh-token");

        var rotated = service.rotate(issued.rawToken());

        assertThat(rotated.user().getId()).isEqualTo(u.getId());
        assertThat(rotated.newRawToken()).isNotEqualTo(issued.rawToken());
        assertThat(rotated.kcIdToken()).isEqualTo("kc-id-token");
        // 백채널 로그아웃용 KC refresh_token 은 rotate 후에도 회수 가능해야 한다(패밀리 승계).
        assertThat(service.revokeFamilyByRawToken(rotated.newRawToken())).isEqualTo("kc-refresh-token");
        // 옛 토큰 재사용은 이제 도난으로 간주(grace=0)
        assertThatThrownBy(() -> service.rotate(issued.rawToken()))
                .isInstanceOf(ReuseDetectedException.class);
    }

    @Test
    void reuseDetectionRevokesWholeFamily() {
        User u = newUser();
        var issued = service.issue(u, "kc-id-token", "kc-refresh-token");
        var rotated = service.rotate(issued.rawToken());

        assertThatThrownBy(() -> service.rotate(issued.rawToken()))
                .isInstanceOf(ReuseDetectedException.class);
        // 패밀리가 폐기됐으므로 방금 회전한 정상 토큰도 더는 못 씀
        assertThatThrownBy(() -> service.rotate(rotated.newRawToken()))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void reuseDetectedExceptionCarriesUserAndFamily() {
        User u = newUser();
        var issued = service.issue(u, "kc-id-token", "kc-refresh-token");
        service.rotate(issued.rawToken());

        assertThatThrownBy(() -> service.rotate(issued.rawToken()))
                .isInstanceOfSatisfying(ReuseDetectedException.class, e -> {
                    assertThat(e.getUserId()).isEqualTo(u.getId());
                    assertThat(e.getFamilyId()).isNotNull();
                });
    }

    @Test
    void markRotatedClaimsTokenExactlyOnce() {
        User u = newUser();
        var issued = service.issue(u, "kc-id-token", "kc-refresh-token");
        RefreshToken t = tokenRepository.findByTokenHash(RefreshTokenService.sha256(issued.rawToken())).orElseThrow();

        int first = tokenRepository.markRotated(t.getId(), UUID.randomUUID(), Instant.now());
        int second = tokenRepository.markRotated(t.getId(), UUID.randomUUID(), Instant.now());

        assertThat(first).isEqualTo(1);
        assertThat(second).isZero();
    }

    @Test
    void reuseWithinGraceIsToleratedAndFamilySurvives() {
        ReflectionTestUtils.setField(service, "graceSeconds", 30L);
        User u = newUser();
        var issued = service.issue(u, "kc-id-token", "kc-refresh-token");
        var rotated = service.rotate(issued.rawToken());

        // 방금(grace 이내) 교체된 토큰 재사용 = 멀티탭 경쟁으로 관용
        assertThatThrownBy(() -> service.rotate(issued.rawToken()))
                .isInstanceOf(ConcurrentRotationException.class);
        // 가족은 생존 — 최신 토큰은 계속 회전 가능
        assertThatCode(() -> service.rotate(rotated.newRawToken())).doesNotThrowAnyException();
    }

    @Test
    void reuseAfterGraceIsTheft() {
        ReflectionTestUtils.setField(service, "graceSeconds", 30L);
        User u = newUser();
        var issued = service.issue(u, "kc-id-token", "kc-refresh-token");
        service.rotate(issued.rawToken());

        // replaced_at 을 grace 밖(2분 전)으로 조작 → 도난 판정
        RefreshToken old = tokenRepository.findByTokenHash(RefreshTokenService.sha256(issued.rawToken())).orElseThrow();
        old.setReplacedAt(Instant.now().minusSeconds(120));
        tokenRepository.save(old);

        assertThatThrownBy(() -> service.rotate(issued.rawToken()))
                .isInstanceOf(ReuseDetectedException.class);
    }

    @Test
    void disabledUserCannotRotate() {
        User u = newUser();
        var issued = service.issue(u, "kc-id-token", "kc-refresh-token");
        u.setEnabled(false);
        userRepository.save(u);

        assertThatThrownBy(() -> service.rotate(issued.rawToken()))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("비활성화");
    }

    @Test
    void familyPastAbsoluteTtlCannotRotate() {
        User u = newUser();
        var issued = service.issue(u, "kc-id-token", "kc-refresh-token");
        RefreshToken t = tokenRepository.findByTokenHash(RefreshTokenService.sha256(issued.rawToken())).orElseThrow();
        t.setFamilyCreatedAt(Instant.now().minusSeconds(7776000L + 60)); // 90일 + 1분 전
        tokenRepository.save(t);

        assertThatThrownBy(() -> service.rotate(issued.rawToken()))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("절대");
    }

    @Test
    void invalidTokenRejected() {
        assertThatThrownBy(() -> service.rotate("not-a-real-token"))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: 실패 확인**

Run: `./gradlew test --tests "com.platform.authserver.token.RefreshTokenServiceTest"`
Expected: 컴파일 실패 — `graceSeconds` 필드 미정의.

- [ ] **Step 3: rotate() 재설계** — `RefreshTokenService.java`의 필드·`rotate()`를 교체:

필드 추가 (`ttlSeconds` 아래):

```java
@Value("${platform.rotation-grace-seconds}")
private long graceSeconds;

@Value("${platform.session-absolute-ttl-seconds}")
private long absoluteTtlSeconds;
```

`rotate()` 전체 교체:

```java
@Transactional(noRollbackFor = ReuseDetectedException.class)
public Rotated rotate(String rawToken) {
    RefreshToken current = tokenRepository.findByTokenHash(sha256(rawToken))
            .orElseThrow(() -> new IllegalArgumentException("알 수 없는 refresh token"));

    if (current.isRevoked()) {
        handleRevoked(current); // 항상 throw
    }
    if (current.getExpiresAt().isBefore(Instant.now())) {
        throw new IllegalArgumentException("만료된 refresh token");
    }
    // sliding 만료 보완: 가족 생성 후 절대 상한을 넘기면 재로그인 유도
    if (Instant.now().isAfter(current.getFamilyCreatedAt().plusSeconds(absoluteTtlSeconds))) {
        throw new IllegalArgumentException("세션 절대 상한 초과");
    }

    User user = userRepository.findById(current.getUserId())
            .orElseThrow(() -> new IllegalArgumentException("사용자 없음"));
    if (!user.isEnabled()) {
        throw new IllegalArgumentException("비활성화된 사용자");
    }

    // 선점(조건부 UPDATE) → INSERT 순서. next ID를 선확정해야 선점 쿼리에 실을 수 있다.
    String newRaw = newRawToken();
    RefreshToken next = new RefreshToken(user.getId(), sha256(newRaw), current.getFamilyId(),
            current.getKcIdToken(), current.getKcRefreshToken(),
            Instant.now().plusSeconds(ttlSeconds), current.getFamilyCreatedAt());

    int claimed = tokenRepository.markRotated(current.getId(), next.getId(), Instant.now());
    if (claimed == 0) {
        // 경쟁 패배 — clearAutomatically 로 컨텍스트가 비워졌으므로 재조회 후 grace 판정
        RefreshToken fresh = tokenRepository.findById(current.getId())
                .orElseThrow(() -> new IllegalArgumentException("유효하지 않은 refresh token"));
        handleRevoked(fresh); // 항상 throw
    }
    tokenRepository.save(next);

    return new Rotated(user, newRaw, next.getKcIdToken());
}

/** 폐기된 토큰 처리: grace 이내 재사용=경쟁 관용, 경과=도난(가족 폐기), 교체 이력 없으면 단순 무효. */
private void handleRevoked(RefreshToken t) {
    if (t.getReplacedBy() != null) {
        if (t.getReplacedAt() != null
                && t.getReplacedAt().isAfter(Instant.now().minusSeconds(graceSeconds))) {
            throw new ConcurrentRotationException("grace 이내 재사용 — 멀티탭 경쟁으로 관용");
        }
        // grace 밖 재사용 = 도난 → 패밀리 전체 폐기 (noRollbackFor 로 커밋 보장)
        tokenRepository.revokeFamily(t.getFamilyId());
        throw new ReuseDetectedException("refresh token 재사용 탐지", t.getUserId(), t.getFamilyId());
    }
    // 패밀리 폐기로 부수적으로 무효화된 토큰 → 단순 무효
    throw new IllegalArgumentException("유효하지 않은 refresh token");
}
```

(기존 `current.setRevoked(true)` / `current.setReplacedBy(...)` / `tokenRepository.save(current)` 3줄은 markRotated 로 대체되어 삭제. `rotate()` 안의 기존 `persist(...)` 호출도 위의 인라인 생성 + `save(next)` 로 대체.)

- [ ] **Step 4: 설정 키 추가** — `application.yml`의 `platform:` 블록에:

```yaml
  # rotate 경쟁(멀티탭) vs 도난 구분 창. 이내 재사용은 관용(가족 생존), 경과 재사용은 가족 전멸.
  rotation-grace-seconds: ${ROTATION_GRACE_SECONDS:30}
  # 가족 생성 기준 절대 세션 상한(기본 90일) — sliding 만료의 영구 세션 방지
  session-absolute-ttl-seconds: ${SESSION_ABSOLUTE_TTL_SECONDS:7776000}
```

- [ ] **Step 5: 전체 테스트 통과 확인**

Run: `./gradlew test`
Expected: BUILD SUCCESSFUL — RefreshTokenServiceTest 9개 전부 PASS, AuthControllerTest·기타 기존 테스트 PASS.

- [ ] **Step 6: Commit**

```bash
git add -A src/
git commit -m "feat(token): rotate 원자 선점(조건부 UPDATE)+grace 30초 — 재사용 탐지 우회·멀티탭 오탐 동시 해소, enabled·절대 상한(90일) 검사

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 5: Testcontainers — Postgres 실 DB 검증 (동시성·noRollbackFor·V3)

**Files:**
- Modify: `build.gradle` (테스트 의존성 3개)
- Create: `src/test/java/com/platform/authserver/token/RefreshTokenServicePostgresTest.java`

**Interfaces:**
- Consumes: Task 4의 rotate() 계약 전부, Task 3의 엔티티 setter들, `TestOAuth2ClientConfig`(기존 — OIDC discovery 차단용)
- Produces: PG 컨테이너 테스트 패턴(`@ServiceConnection`, 프로필 미사용으로 Flyway V1~V3 실측) — Task 7의 배치 테스트가 같은 클래스에 추가된다.

- [ ] **Step 1: 의존성 추가** — `build.gradle`의 dependencies 블록에:

```groovy
testImplementation 'org.springframework.boot:spring-boot-testcontainers'
testImplementation 'org.testcontainers:junit-jupiter'
testImplementation 'org.testcontainers:postgresql'
```

- [ ] **Step 2: 실패하는 테스트 작성** — `RefreshTokenServicePostgresTest.java` (전체 신규):

```java
package com.platform.authserver.token;

import com.platform.authserver.TestOAuth2ClientConfig;
import com.platform.authserver.user.User;
import com.platform.authserver.user.UserRepository;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.context.annotation.Import;
import org.springframework.test.context.TestPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.time.Instant;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

import static org.assertj.core.api.Assertions.*;

/**
 * 실 Postgres 검증 — H2 슬라이스 테스트가 못 보는 것들:
 * 동시 rotate 경쟁, noRollbackFor 커밋 동작, V3 마이그레이션(Flyway on).
 * test 프로필을 쓰지 않는다(프로필이 H2+Flyway off 로 바꿔버림).
 */
@Testcontainers
@SpringBootTest
@Import(TestOAuth2ClientConfig.class)
@TestPropertySource(properties = {"eureka.client.enabled=false"})
class RefreshTokenServicePostgresTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Autowired RefreshTokenService service;
    @Autowired UserRepository userRepository;
    @Autowired RefreshTokenRepository tokenRepository;

    @AfterEach
    void clean() {
        tokenRepository.deleteAll();
        userRepository.deleteAll();
    }

    private User newUser() {
        User u = new User("kc-sub-pg-" + System.nanoTime());
        u.setRoles(List.of("USER"));
        return userRepository.save(u);
    }

    @Test
    void concurrentRotateLeavesExactlyOneLiveToken() throws Exception {
        User u = newUser();
        var issued = service.issue(u, "kc-id", "kc-rt");

        var barrier = new CyclicBarrier(2);
        Callable<Object> attempt = () -> {
            barrier.await();
            try {
                return service.rotate(issued.rawToken());
            } catch (RuntimeException e) {
                return e;
            }
        };
        ExecutorService pool = Executors.newFixedThreadPool(2);
        List<Object> results;
        try {
            results = pool.invokeAll(List.of(attempt, attempt)).stream()
                    .map(f -> {
                        try { return f.get(); } catch (Exception e) { throw new IllegalStateException(e); }
                    }).toList();
        } finally {
            pool.shutdown();
        }

        // 정확히 1승 1패 — 패자는 경쟁 관용(도난 오판 아님)
        assertThat(results).filteredOn(r -> r instanceof RefreshTokenService.Rotated).hasSize(1);
        assertThat(results).filteredOn(r -> r instanceof ConcurrentRotationException).hasSize(1);
        // DB: 살아있는 토큰 정확히 1개, 가족 생존
        assertThat(tokenRepository.findAll()).filteredOn(t -> !t.isRevoked()).hasSize(1);
    }

    @Test
    void reuseWithinGraceKeepsFamilyAlive() {
        User u = newUser();
        var issued = service.issue(u, "kc-id", "kc-rt");
        var rotated = service.rotate(issued.rawToken());

        assertThatThrownBy(() -> service.rotate(issued.rawToken()))
                .isInstanceOf(ConcurrentRotationException.class);
        assertThatCode(() -> service.rotate(rotated.newRawToken())).doesNotThrowAnyException();
    }

    @Test
    void reuseAfterGraceRevokesFamilyAndCommitsDespiteException() {
        User u = newUser();
        var issued = service.issue(u, "kc-id", "kc-rt");
        service.rotate(issued.rawToken());

        RefreshToken old = tokenRepository.findByTokenHash(RefreshTokenService.sha256(issued.rawToken())).orElseThrow();
        old.setReplacedAt(Instant.now().minusSeconds(120)); // grace(30초) 밖
        tokenRepository.save(old);

        assertThatThrownBy(() -> service.rotate(issued.rawToken()))
                .isInstanceOf(ReuseDetectedException.class);

        // 핵심(§1 롤백 버그 회귀 방지): 예외에도 불구하고 가족 폐기가 '커밋'되어 있어야 한다.
        // 이 조회는 별도 트랜잭션 — 롤백됐다면 살아있는 토큰이 보인다.
        assertThat(tokenRepository.findAll()).allMatch(RefreshToken::isRevoked);
    }

    @Test
    void disabledUserAndExpiredFamilyAreRejected() {
        User u = newUser();
        var issued = service.issue(u, "kc-id", "kc-rt");
        u.setEnabled(false);
        userRepository.save(u);
        assertThatThrownBy(() -> service.rotate(issued.rawToken()))
                .isInstanceOf(IllegalArgumentException.class);

        User u2 = newUser();
        var issued2 = service.issue(u2, "kc-id", "kc-rt");
        RefreshToken t = tokenRepository.findByTokenHash(RefreshTokenService.sha256(issued2.rawToken())).orElseThrow();
        t.setFamilyCreatedAt(Instant.now().minusSeconds(7776000L + 60));
        tokenRepository.save(t);
        assertThatThrownBy(() -> service.rotate(issued2.rawToken()))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 3: 실행·통과 확인** (Docker Desktop 기동 필요)

Run: `./gradlew test --tests "com.platform.authserver.token.RefreshTokenServicePostgresTest"`
Expected: PASS 4개 — Flyway가 V1~V3를 실 PG에 적용(마이그레이션 검증 겸함). 실패 시 원인 파악 우선(특히 `concurrentRotate`가 간헐 실패하면 경쟁 로직 버그일 가능성 — 재시도로 덮지 말 것).

- [ ] **Step 4: 전체 테스트 확인**

Run: `./gradlew test`
Expected: BUILD SUCCESSFUL (H2 + PG 전부)

- [ ] **Step 5: Commit**

```bash
git add -A src/ build.gradle
git commit -m "test(token): Testcontainers 실 PG 검증 — 동시 rotate 1승1패, grace 관용, noRollbackFor 커밋, V3 실측

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 6: HTTP 타임아웃 — KeycloakLogoutClient + 로그인 토큰 교환

**Files:**
- Modify: `src/main/java/com/platform/authserver/auth/KeycloakLogoutClient.java:39-41`
- Modify: `src/main/resources/application.yml`
- Test: `src/test/java/com/platform/authserver/auth/KeycloakLogoutClientTest.java` (Create)

**Interfaces:**
- Consumes: 없음 (독립)
- Produces: 없음 (동작 특성만 변경 — 시그니처 불변)

- [ ] **Step 1: 실패(행 검증)하는 테스트 작성** — `KeycloakLogoutClientTest.java` (신규):

```java
package com.platform.authserver.auth;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.Timeout;

import java.net.ServerSocket;

import static org.assertj.core.api.Assertions.assertThat;

class KeycloakLogoutClientTest {

    @Test
    @Timeout(15) // 타임아웃 미설정이면 read가 무한 대기 → 이 테스트가 행 재현이자 회귀 방지
    void logoutReturnsInsteadOfHangingWhenKeycloakStalls() throws Exception {
        // accept 는 OS 백로그가 받아주지만 아무도 응답하지 않는 서버 — KC 행(hang) 시뮬레이션
        try (ServerSocket stalled = new ServerSocket(0)) {
            var client = new KeycloakLogoutClient(
                    "http://localhost:" + stalled.getLocalPort() + "/realms/x", "cid", "secret");

            long start = System.nanoTime();
            client.logout("some-refresh-token"); // best-effort — 예외는 삼키고 복귀해야 한다
            long elapsedMs = (System.nanoTime() - start) / 1_000_000;

            assertThat(elapsedMs).isLessThan(10_000); // connect 2s + read 3s + 여유
        }
    }
}
```

- [ ] **Step 2: 실패 확인**

Run: `./gradlew test --tests "com.platform.authserver.auth.KeycloakLogoutClientTest"`
Expected: FAIL — `@Timeout(15)` 초과(현재 `RestClient.create()`는 read 무한 대기).

- [ ] **Step 3: 타임아웃 설정** — `KeycloakLogoutClient.java` 생성자의 `RestClient.create()` 를 교체:

```java
// KC 행(hang) 시 서블릿 스레드 동반 고갈 방지 — best-effort 는 예외는 삼켜도 행은 못 삼킨다.
var settings = org.springframework.boot.http.client.ClientHttpRequestFactorySettings.defaults()
        .withConnectTimeout(java.time.Duration.ofSeconds(2))
        .withReadTimeout(java.time.Duration.ofSeconds(3));
this.restClient = RestClient.builder()
        .requestFactory(org.springframework.boot.http.client.ClientHttpRequestFactoryBuilder.detect().build(settings))
        .build();
```

(컴파일 실패 시 Boot 4.0.6에서 `ClientHttpRequestFactorySettings`/`ClientHttpRequestFactoryBuilder`의 실제 패키지를 IDE/javadoc으로 확인해 import만 조정 — Boot 3.4에서 `org.springframework.boot.http.client`로 이동했고 Boot 4도 동일 계열이다. 클래스가 아예 없으면 `JdkClientHttpRequestFactory` 직접 생성으로 대체: `new JdkClientHttpRequestFactory()` + `setReadTimeout(Duration.ofSeconds(3))`, connect 는 `HttpClient.newBuilder().connectTimeout(...)` 로.)

- [ ] **Step 4: 로그인 토큰 교환 타임아웃** — `application.yml`의 `spring:` 블록에:

```yaml
  # Boot 자동구성 HTTP 클라이언트(OIDC 토큰 교환 포함) 공통 타임아웃 — KC 행 시 로그인 스레드 고갈 방지
  http:
    client:
      connect-timeout: 2s
      read-timeout: 3s
```

적용 검증: `./gradlew bootRun` 후 정상 기동 확인 + Boot 4 configuration metadata에서 키 존재 확인(자동완성/문서). 키가 Boot 4에서 지원되지 않으면(바인딩 안 됨) 이 yml 블록을 제거하고 다음 빈을 `SecurityConfig`에 추가하는 것으로 대체:

```java
@Bean
OAuth2AccessTokenResponseClient<OAuth2AuthorizationCodeGrantRequest> tokenResponseClient() {
    var settings = ClientHttpRequestFactorySettings.defaults()
            .withConnectTimeout(Duration.ofSeconds(2))
            .withReadTimeout(Duration.ofSeconds(3));
    RestClient restClient = RestClient.builder()
            .requestFactory(ClientHttpRequestFactoryBuilder.detect().build(settings))
            .messageConverters(converters -> {
                converters.clear();
                converters.add(new FormHttpMessageConverter());
                converters.add(new OAuth2AccessTokenResponseHttpMessageConverter());
            })
            .defaultStatusHandler(new OAuth2ErrorResponseErrorHandler())
            .build();
    var client = new RestClientAuthorizationCodeTokenResponseClient();
    client.setRestClient(restClient);
    return client;
}
```

- [ ] **Step 5: 통과 확인**

Run: `./gradlew test`
Expected: BUILD SUCCESSFUL — `KeycloakLogoutClientTest` 가 10초 이내 복귀 검증.

- [ ] **Step 6: Commit**

```bash
git add -A src/
git commit -m "fix(auth): KC 호출 타임아웃(connect 2s/read 3s) — 로그아웃·토큰 교환 행(hang)으로 인한 스레드풀 고갈 방지

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 7: RT 청소 배치 — 가족 단위, 보존 기간 = 탐지 유효 기간

**Files:**
- Modify: `src/main/java/com/platform/authserver/AuthServerApplication.java` (`@EnableScheduling`)
- Modify: `src/main/java/com/platform/authserver/token/RefreshTokenRepository.java`
- Create: `src/main/java/com/platform/authserver/token/TokenCleanupJob.java`
- Modify: `src/main/resources/application.yml`
- Test: `src/test/java/com/platform/authserver/token/RefreshTokenServicePostgresTest.java` (테스트 추가)

**Interfaces:**
- Consumes: Task 5의 PG 테스트 클래스
- Produces: `RefreshTokenRepository.deleteDeadFamilies(Instant cutoff): int`, `TokenCleanupJob.cleanup(): void`

- [ ] **Step 1: 실패하는 테스트 작성** — `RefreshTokenServicePostgresTest`에 추가 (필드에 `@Autowired TokenCleanupJob cleanupJob;` 추가):

```java
@Test
void cleanupDeletesDeadFamiliesButKeepsLiveEvidence() {
    User u = newUser();

    // 죽은 가족: 모든 토큰의 expires_at 이 (now - 14일) 보다 과거 → 탐지 유효 기간도 끝남
    RefreshToken dead = new RefreshToken(u.getId(), "dead-hash", java.util.UUID.randomUUID(),
            null, null, Instant.now().minusSeconds(15L * 24 * 3600), Instant.now().minusSeconds(40L * 24 * 3600));
    dead.setRevoked(true);
    tokenRepository.save(dead);

    // 산 가족: 최신 토큰이 살아있음 — 폐기된 옛 행(재사용 탐지 증거물)도 함께 보존돼야 한다
    var issued = service.issue(u, "kc-id", "kc-rt");
    service.rotate(issued.rawToken()); // 가족에 폐기 1 + 활성 1

    cleanupJob.cleanup();

    assertThat(tokenRepository.findAll())
            .noneMatch(t -> "dead-hash".equals(t.getTokenHash()))  // 죽은 가족 삭제
            .hasSize(2);                                            // 산 가족은 증거물 포함 전부 보존
}
```

- [ ] **Step 2: 실패 확인**

Run: `./gradlew test --tests "com.platform.authserver.token.RefreshTokenServicePostgresTest.cleanupDeletesDeadFamiliesButKeepsLiveEvidence"`
Expected: 컴파일 실패 — `TokenCleanupJob` 미정의.

- [ ] **Step 3: 리포지토리 쿼리 추가** — `RefreshTokenRepository.java`:

```java
// 가족 단위 삭제 — 가족의 '최신' 만료가 cutoff 를 지난 경우만. 폐기된 옛 행은 재사용 탐지의
// 증거물이므로 가족이 살아있는 동안은 절대 지우지 않는다(보존 기간 = 탐지 유효 기간).
@Modifying(flushAutomatically = true, clearAutomatically = true)
@Query("delete from RefreshToken t where t.familyId in " +
       "(select t2.familyId from RefreshToken t2 group by t2.familyId having max(t2.expiresAt) < :cutoff)")
int deleteDeadFamilies(@Param("cutoff") Instant cutoff);
```

- [ ] **Step 4: 배치 컴포넌트 생성** — `TokenCleanupJob.java`:

```java
package com.platform.authserver.token;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;

/** refresh_tokens 무한 증식 방지 일 배치. 가족 단위 삭제 — 근거는 deleteDeadFamilies 주석 참고. */
@Component
public class TokenCleanupJob {

    private static final Logger log = LoggerFactory.getLogger(TokenCleanupJob.class);

    private final RefreshTokenRepository tokenRepository;

    @Value("${platform.refresh-token-ttl-seconds}")
    private long ttlSeconds;

    public TokenCleanupJob(RefreshTokenRepository tokenRepository) {
        this.tokenRepository = tokenRepository;
    }

    @Scheduled(cron = "${platform.token-cleanup-cron:0 0 4 * * *}")
    @Transactional
    public void cleanup() {
        // cutoff = now - RT TTL(버퍼): 가족 만료 후에도 탐지 유효 기간만큼 더 보존
        Instant cutoff = Instant.now().minusSeconds(ttlSeconds);
        int deleted = tokenRepository.deleteDeadFamilies(cutoff);
        if (deleted > 0) {
            log.info("RT 청소 배치: 죽은 가족 토큰 {}건 삭제", deleted);
        }
    }
}
```

- [ ] **Step 5: 스케줄링 활성화 + cron 설정** — `AuthServerApplication.java`에 `@org.springframework.scheduling.annotation.EnableScheduling` 추가, `application.yml`의 `platform:` 블록에:

```yaml
  # RT 청소 배치 cron (기본 매일 04:00)
  token-cleanup-cron: ${TOKEN_CLEANUP_CRON:0 0 4 * * *}
```

- [ ] **Step 6: 통과 확인**

Run: `./gradlew test`
Expected: BUILD SUCCESSFUL

- [ ] **Step 7: Commit**

```bash
git add -A src/
git commit -m "feat(token): RT 청소 일 배치 — 가족 단위 삭제, 보존 기간=탐지 유효 기간(증거물 보존)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 8: 문서 갱신 — README 환경변수 표 + 스펙 상태

**Files:**
- Modify: `README.md` (환경변수 표 — 실제 표 위치·형식은 파일 열어 확인 후 기존 형식 그대로 행 추가)
- Modify: `C:\myBrain\내 로컬\msa\MSA_TEMPLATE 설계문서\specs\2026-07-19-auth-server-hardening-design.md` (상태 갱신)

**Interfaces:**
- Consumes: Task 4·6·7이 추가한 설정 키

- [ ] **Step 1: README 환경변수 표에 행 추가** (기존 표 형식에 맞춰):

| 변수 | 기본값 | 설명 |
|---|---|---|
| `ROTATION_GRACE_SECONDS` | `30` | RT 회전 경쟁(멀티탭) 관용 창 — 이내 재사용은 도난 아님 |
| `SESSION_ABSOLUTE_TTL_SECONDS` | `7776000` | 가족 생성 기준 절대 세션 상한(90일) |
| `TOKEN_CLEANUP_CRON` | `0 0 4 * * *` | RT 청소 배치 주기 |

인증 플로우 설명부에 한 줄 추가: 재사용 탐지는 grace 30초 이내를 경쟁으로 관용하며, 도난 판정 시 가족 전체를 폐기하고 WARN 로그를 남긴다.

- [ ] **Step 2: 스펙 상태 갱신** — 스펙 문서 헤더의 `- **상태:** 승인됨 (구현 계획 작성 전)` 을 `- **상태:** 구현 완료 (2026-07-19, 이 plans 문서 기준)` 으로 교체.

- [ ] **Step 3: 최종 전체 검증**

Run: `./gradlew test`
Expected: BUILD SUCCESSFUL — 전체 그린.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: 보안 강화 설정 3종(grace·절대 상한·청소 cron) 환경변수 표 반영

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

## 스펙 커버리지 맵 (self-review 근거)

| 스펙 | 태스크 |
|---|---|
| §1 롤백 버그 (noRollbackFor) | Task 1 (수정) + Task 5 (커밋 검증) |
| §2.1 V3 마이그레이션 | Task 3 (작성) + Task 5 (Flyway 실측) |
| §2.2~2.4 rotate 재설계·설정 | Task 4 |
| §3 컨트롤러 3분기·예외 페이로드 | Task 1 + Task 2 |
| §4 타임아웃 (로그아웃+로그인) | Task 6 |
| §5 청소 배치 | Task 7 |
| §6 테스트 전략 | Task 4 (H2) + Task 5 (PG) + Task 6·7 내 테스트 |
| §7 에러 정책 | Task 2 (응답 동일성) |
| §9-6 문서 갱신 | Task 8 |

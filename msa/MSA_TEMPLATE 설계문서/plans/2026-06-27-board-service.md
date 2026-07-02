# board-service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** auth-server가 발급한 JWT를 자체 검증하는 리소스 서버로서, 게시글+댓글 CRUD와 "본인 OR ADMIN" 인가를 제공하는 board-service를 만든다.

**Architecture:** `MSA_TEMPLATE/board-service` 독립 Gradle 프로젝트. Spring Security resource-server가 `jwk-set-uri`로 auth-server의 공개키를 가져와 JWT를 검증하고, `JwtGrantedAuthoritiesConverter`로 `roles` claim을 `ROLE_*` 권한에 매핑한다. 조회는 공개, 쓰기는 인증 필요, 수정/삭제는 서비스 계층 `AccessGuard`가 작성자 본인 또는 ADMIN인지 판정한다.

**Tech Stack:** Spring Boot 4.0.6, Java 24, Spring Web/Security(resource-server)/Data JPA/Validation, Flyway(PostgreSQL), Lombok, PostgreSQL(runtime), H2(test), spring-security-test.

## Global Constraints

- Spring Boot `4.0.6`, `io.spring.dependency-management` `1.1.7`, Java toolchain `24`.
- 베이스 패키지: `com.platform.boardservice`. group `com.platform`.
- 서버 포트: `9100`. DB: `jdbc:postgresql://localhost:5433/boarddb`, user/pw `keycloak`/`keycloak`.
- 토큰 검증: `spring.security.oauth2.resourceserver.jwt.jwk-set-uri = http://localhost:9000/.well-known/jwks.json`. **`issuer-uri`는 설정하지 않는다**(auth-server가 OIDC discovery 문서를 노출하지 않아 기동 실패).
- Role 매핑: claim 이름 `roles`, prefix `ROLE_` (auth-server와 동일).
- Lombok `1.18.46` (`compileOnly` + `annotationProcessor`).
- 테스트 프로파일(`test`): H2 인메모리(`MODE=PostgreSQL`), `ddl-auto=create-drop`, `flyway.enabled=false`. MockMvc 인증은 `spring-security-test`의 `jwt()` post-processor 사용.
- 엔티티 컨벤션: `@NoArgsConstructor(access = PROTECTED)`, `@GeneratedValue(strategy = IDENTITY)`, 시간은 `java.time.Instant`, 컬럼 `created_at`/`updated_at`.

---

## File Structure

```
board-service/
  build.gradle, settings.gradle
  src/main/resources/application.yml
  src/main/resources/db/migration/V1__init.sql        (post 테이블)
  src/main/resources/db/migration/V2__add_comment.sql (comment 테이블)
  src/main/java/com/platform/boardservice/
    BoardServiceApplication.java
    config/SecurityConfig.java          리소스 서버 + Method Security + roles 컨버터
    security/AccessGuard.java           현재 사용자 id/name/admin 판정, 본인-OR-ADMIN 가드
    common/NotFoundException.java
    common/ApiExceptionHandler.java     401/403/404/400 일관 응답
    post/Post.java, PostRepository.java, PostService.java, PostController.java
    post/dto/{PostCreateRequest,PostUpdateRequest,PostResponse,PostSummaryResponse}.java
    comment/Comment.java, CommentRepository.java, CommentService.java, CommentController.java
    comment/dto/{CommentCreateRequest,CommentUpdateRequest,CommentResponse}.java
  src/test/resources/application-test.yml
  src/test/java/com/platform/boardservice/
    BoardServiceApplicationTests.java
    TestAuth.java                       테스트용 JwtAuthenticationToken 팩토리
    security/AccessGuardTest.java
    post/PostServiceTest.java, post/PostControllerTest.java
    comment/CommentServiceTest.java, comment/CommentControllerTest.java
```

---

### Task 1: 프로젝트 스캐폴드 & 부팅

**Files:**
- Create: `board-service/settings.gradle`, `board-service/build.gradle`
- Create: `board-service/src/main/resources/application.yml`
- Create: `board-service/src/test/resources/application-test.yml`
- Create: `board-service/src/main/java/com/platform/boardservice/BoardServiceApplication.java`
- Create: `board-service/src/main/java/com/platform/boardservice/config/SecurityConfig.java`
- Test: `board-service/src/test/java/com/platform/boardservice/BoardServiceApplicationTests.java`

**Interfaces:**
- Produces: `SecurityConfig` 빈 `JwtAuthenticationConverter jwtAuthenticationConverter()`, `SecurityFilterChain filterChain(...)`. 패키지 `com.platform.boardservice`.

- [ ] **Step 1: settings.gradle 작성**

```gradle
rootProject.name = 'board-service'
```

- [ ] **Step 2: build.gradle 작성**

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.0.6'
    id 'io.spring.dependency-management' version '1.1.7'
}
group = 'com.platform'
version = '0.0.1-SNAPSHOT'
java { toolchain { languageVersion = JavaLanguageVersion.of(24) } }
repositories { mavenCentral() }
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-flyway'
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-database-postgresql'
    runtimeOnly 'org.postgresql:postgresql'
    compileOnly 'org.projectlombok:lombok:1.18.46'
    annotationProcessor 'org.projectlombok:lombok:1.18.46'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.boot:spring-boot-data-jpa-test'
    testImplementation 'org.springframework.security:spring-security-test'
    testRuntimeOnly 'com.h2database:h2'
}
tasks.named('test') { useJUnitPlatform() }
```

- [ ] **Step 3: application.yml 작성**

```yaml
server:
  port: 9100
spring:
  application:
    name: board-service
  datasource:
    url: jdbc:postgresql://localhost:5433/boarddb
    username: keycloak
    password: keycloak
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
  flyway:
    enabled: true
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: http://localhost:9000/.well-known/jwks.json
```

> 실제 PostgreSQL 구동 시 `boarddb`가 있어야 한다(테스트는 H2라 불필요). 최초 1회: psql에 접속해 `CREATE DATABASE boarddb;` 후 소유권을 `keycloak`에 부여(`ALTER DATABASE boarddb OWNER TO keycloak;`). 이 단계는 테스트 통과의 전제조건이 아니다.

- [ ] **Step 4: application-test.yml 작성**

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:boarddb;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
    driver-class-name: org.h2.Driver
    username: sa
    password: ""
  jpa:
    hibernate:
      ddl-auto: create-drop
  flyway:
    enabled: false
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: http://localhost:9000/.well-known/jwks.json
```

> `jwk-set-uri`는 디코더 빈 생성에만 쓰이고 `jwt()` post-processor 테스트에서는 실제 호출되지 않으므로 네트워크 접속이 없다.

- [ ] **Step 5: BoardServiceApplication 작성**

```java
package com.platform.boardservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class BoardServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(BoardServiceApplication.class, args);
    }
}
```

- [ ] **Step 6: SecurityConfig 작성**

```java
package com.platform.boardservice.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableMethodSecurity // 수정/삭제는 AccessGuard(서비스 계층)로 판정. 대안: @PreAuthorize 표현식.
public class SecurityConfig {

    /** auth-server와 동일: roles claim → ROLE_* 권한. */
    @Bean
    JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authorities = new JwtGrantedAuthoritiesConverter();
        authorities.setAuthoritiesClaimName("roles");
        authorities.setAuthorityPrefix("ROLE_");
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authorities);
        return converter;
    }

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http, JwtAuthenticationConverter converter) throws Exception {
        http
                .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers(HttpMethod.GET, "/api/posts/**").permitAll()
                        .anyRequest().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(converter)));
        return http.build();
    }
}
```

- [ ] **Step 7: contextLoads 테스트 작성**

```java
package com.platform.boardservice;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest
@ActiveProfiles("test")
class BoardServiceApplicationTests {
    @Test
    void contextLoads() {
    }
}
```

- [ ] **Step 8: 테스트 실행(통과 확인)**

Run: `cd board-service && ./gradlew test --tests "com.platform.boardservice.BoardServiceApplicationTests"`
(Windows: `./gradlew.bat`; Gradle wrapper가 없으면 `gradle wrapper --gradle-version 8.14` 후 재실행)
Expected: PASS — 컨텍스트가 네트워크 호출 없이 로드됨.

- [ ] **Step 9: Commit**

```bash
git add board-service
git commit -m "feat(board-service): 리소스 서버 스캐폴드 + Method Security 기본 설정"
```

---

### Task 2: Post 엔티티 + 마이그레이션 + 리포지토리

**Files:**
- Create: `board-service/src/main/resources/db/migration/V1__init.sql`
- Create: `board-service/src/main/java/com/platform/boardservice/post/Post.java`
- Create: `board-service/src/main/java/com/platform/boardservice/post/PostRepository.java`
- Test: `board-service/src/test/java/com/platform/boardservice/post/PostRepositoryTest.java`

**Interfaces:**
- Produces: `Post(String title, String content, Long authorId, String authorName)` 생성자, getter `getId/getTitle/getContent/getAuthorId/getAuthorName/getCreatedAt/getUpdatedAt`, `void edit(String title, String content)`. `PostRepository extends JpaRepository<Post, Long>`.

- [ ] **Step 1: 실패 테스트 작성**

```java
package com.platform.boardservice.post;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest; // Spring Boot 4 패키지
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@ActiveProfiles("test")
class PostRepositoryTest {

    @Autowired PostRepository repository;

    @Test
    void savesAndReadsPost() {
        Post saved = repository.save(new Post("제목", "내용", 42L, "Alice"));
        assertThat(saved.getId()).isNotNull();
        Post found = repository.findById(saved.getId()).orElseThrow();
        assertThat(found.getTitle()).isEqualTo("제목");
        assertThat(found.getAuthorId()).isEqualTo(42L);
        assertThat(found.getCreatedAt()).isNotNull();
    }
}
```

- [ ] **Step 2: 테스트 실행(실패 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.post.PostRepositoryTest"`
Expected: FAIL — `Post`/`PostRepository` 컴파일 불가.

- [ ] **Step 3: V1__init.sql 작성**

```sql
CREATE TABLE post (
    id          BIGSERIAL PRIMARY KEY,
    title       VARCHAR(255) NOT NULL,
    content     TEXT NOT NULL,
    author_id   BIGINT NOT NULL,
    author_name VARCHAR(255) NOT NULL,
    created_at  TIMESTAMP NOT NULL,
    updated_at  TIMESTAMP NOT NULL
);
```

- [ ] **Step 4: Post 엔티티 작성**

```java
package com.platform.boardservice.post;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;

import java.time.Instant;

@Entity
@Table(name = "post")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(nullable = false, columnDefinition = "text")
    private String content;

    @Column(name = "author_id", nullable = false)
    private Long authorId;

    @Column(name = "author_name", nullable = false)
    private String authorName;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    public Post(String title, String content, Long authorId, String authorName) {
        this.title = title;
        this.content = content;
        this.authorId = authorId;
        this.authorName = authorName;
        Instant now = Instant.now();
        this.createdAt = now;
        this.updatedAt = now;
    }

    public void edit(String title, String content) {
        this.title = title;
        this.content = content;
        this.updatedAt = Instant.now();
    }
}
```

- [ ] **Step 5: PostRepository 작성**

```java
package com.platform.boardservice.post;

import org.springframework.data.jpa.repository.JpaRepository;

public interface PostRepository extends JpaRepository<Post, Long> {
}
```

- [ ] **Step 6: 테스트 실행(통과 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.post.PostRepositoryTest"`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add board-service
git commit -m "feat(board-service): Post 엔티티/마이그레이션/리포지토리"
```

---

### Task 3: 인증 헬퍼 AccessGuard + 테스트 팩토리

**Files:**
- Create: `board-service/src/main/java/com/platform/boardservice/security/AccessGuard.java`
- Create: `board-service/src/test/java/com/platform/boardservice/TestAuth.java`
- Test: `board-service/src/test/java/com/platform/boardservice/security/AccessGuardTest.java`

**Interfaces:**
- Produces: `AccessGuard`(@Component) — `long currentUserId(Authentication)`, `String currentUserName(Authentication)`, `boolean isAdmin(Authentication)`, `void requireOwnerOrAdmin(long authorId, Authentication)`(아니면 `AccessDeniedException`).
- Produces: `TestAuth.user(long id, String name)`, `TestAuth.admin(long id, String name)` → `JwtAuthenticationToken`. 후속 서비스 테스트가 이 팩토리를 사용한다.

- [ ] **Step 1: TestAuth 팩토리 작성** (테스트 헬퍼 — 먼저 작성)

```java
package com.platform.boardservice;

import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;

import java.util.List;

public final class TestAuth {
    private TestAuth() {}

    public static JwtAuthenticationToken user(long userId, String name) {
        return token(userId, name, "USER");
    }

    public static JwtAuthenticationToken admin(long userId, String name) {
        return token(userId, name, "ADMIN");
    }

    private static JwtAuthenticationToken token(long userId, String name, String role) {
        Jwt jwt = Jwt.withTokenValue("test-token")
                .header("alg", "none")
                .subject(String.valueOf(userId))
                .claim("name", name)
                .claim("roles", List.of(role))
                .build();
        return new JwtAuthenticationToken(jwt, List.of(new SimpleGrantedAuthority("ROLE_" + role)));
    }
}
```

- [ ] **Step 2: 실패 테스트 작성**

```java
package com.platform.boardservice.security;

import com.platform.boardservice.TestAuth;
import org.junit.jupiter.api.Test;
import org.springframework.security.access.AccessDeniedException;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.assertThatCode;

class AccessGuardTest {

    AccessGuard guard = new AccessGuard();

    @Test
    void readsUserIdAndName() {
        var auth = TestAuth.user(42L, "Alice");
        assertThat(guard.currentUserId(auth)).isEqualTo(42L);
        assertThat(guard.currentUserName(auth)).isEqualTo("Alice");
        assertThat(guard.isAdmin(auth)).isFalse();
    }

    @Test
    void ownerPasses() {
        assertThatCode(() -> guard.requireOwnerOrAdmin(42L, TestAuth.user(42L, "Alice")))
                .doesNotThrowAnyException();
    }

    @Test
    void adminPassesForOthersResource() {
        assertThatCode(() -> guard.requireOwnerOrAdmin(99L, TestAuth.admin(1L, "Root")))
                .doesNotThrowAnyException();
    }

    @Test
    void otherUserRejected() {
        assertThatThrownBy(() -> guard.requireOwnerOrAdmin(99L, TestAuth.user(42L, "Alice")))
                .isInstanceOf(AccessDeniedException.class);
    }
}
```

- [ ] **Step 3: 테스트 실행(실패 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.security.AccessGuardTest"`
Expected: FAIL — `AccessGuard` 없음.

- [ ] **Step 4: AccessGuard 작성**

```java
package com.platform.boardservice.security;

import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Component;

@Component
public class AccessGuard {

    /** JWT sub(=userId). */
    public long currentUserId(Authentication auth) {
        return Long.parseLong(auth.getName());
    }

    /** JWT name claim(표시용). */
    public String currentUserName(Authentication auth) {
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            String name = jwtAuth.getToken().getClaimAsString("name");
            if (name != null) return name;
        }
        return auth.getName();
    }

    public boolean isAdmin(Authentication auth) {
        for (GrantedAuthority a : auth.getAuthorities()) {
            if ("ROLE_ADMIN".equals(a.getAuthority())) return true;
        }
        return false;
    }

    /** 작성자 본인 또는 ADMIN이 아니면 403. */
    public void requireOwnerOrAdmin(long authorId, Authentication auth) {
        if (isAdmin(auth)) return;
        if (currentUserId(auth) == authorId) return;
        throw new AccessDeniedException("작성자 본인 또는 ADMIN만 가능합니다.");
    }
}
```

- [ ] **Step 5: 테스트 실행(통과 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.security.AccessGuardTest"`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add board-service
git commit -m "feat(board-service): AccessGuard(본인 OR ADMIN) + 테스트 팩토리"
```

---

### Task 4: Post DTO + 생성/조회/목록 서비스

**Files:**
- Create: `board-service/src/main/java/com/platform/boardservice/post/dto/PostCreateRequest.java`
- Create: `board-service/src/main/java/com/platform/boardservice/post/dto/PostUpdateRequest.java`
- Create: `board-service/src/main/java/com/platform/boardservice/post/dto/PostResponse.java`
- Create: `board-service/src/main/java/com/platform/boardservice/post/dto/PostSummaryResponse.java`
- Create: `board-service/src/main/java/com/platform/boardservice/common/NotFoundException.java`
- Create: `board-service/src/main/java/com/platform/boardservice/post/PostService.java`
- Test: `board-service/src/test/java/com/platform/boardservice/post/PostServiceTest.java`

**Interfaces:**
- Produces: records `PostCreateRequest(String title, String content)`, `PostUpdateRequest(String title, String content)`(둘 다 `@NotBlank`), `PostResponse(Long id, String title, String content, Long authorId, String authorName, Instant createdAt, Instant updatedAt)` + `static from(Post)`, `PostSummaryResponse(Long id, String title, String authorName, Instant createdAt)` + `static from(Post)`.
- Produces: `NotFoundException extends RuntimeException`.
- Produces: `PostService` — `PostResponse create(PostCreateRequest, Authentication)`, `PostResponse get(Long)`, `Page<PostSummaryResponse> list(Pageable)`. (update/delete는 Task 5에서 추가)
- Consumes: `AccessGuard`(Task 3), `PostRepository`/`Post`(Task 2).

- [ ] **Step 1: DTO 4종 작성**

`PostCreateRequest.java`:
```java
package com.platform.boardservice.post.dto;

import jakarta.validation.constraints.NotBlank;

public record PostCreateRequest(
        @NotBlank(message = "title은 필수입니다.") String title,
        @NotBlank(message = "content는 필수입니다.") String content) {
}
```

`PostUpdateRequest.java`:
```java
package com.platform.boardservice.post.dto;

import jakarta.validation.constraints.NotBlank;

public record PostUpdateRequest(
        @NotBlank(message = "title은 필수입니다.") String title,
        @NotBlank(message = "content는 필수입니다.") String content) {
}
```

`PostResponse.java`:
```java
package com.platform.boardservice.post.dto;

import com.platform.boardservice.post.Post;

import java.time.Instant;

public record PostResponse(Long id, String title, String content, Long authorId,
                           String authorName, Instant createdAt, Instant updatedAt) {
    public static PostResponse from(Post p) {
        return new PostResponse(p.getId(), p.getTitle(), p.getContent(),
                p.getAuthorId(), p.getAuthorName(), p.getCreatedAt(), p.getUpdatedAt());
    }
}
```

`PostSummaryResponse.java`:
```java
package com.platform.boardservice.post.dto;

import com.platform.boardservice.post.Post;

import java.time.Instant;

public record PostSummaryResponse(Long id, String title, String authorName, Instant createdAt) {
    public static PostSummaryResponse from(Post p) {
        return new PostSummaryResponse(p.getId(), p.getTitle(), p.getAuthorName(), p.getCreatedAt());
    }
}
```

- [ ] **Step 2: NotFoundException 작성**

```java
package com.platform.boardservice.common;

public class NotFoundException extends RuntimeException {
    public NotFoundException(String message) {
        super(message);
    }
}
```

- [ ] **Step 3: 실패 테스트 작성**

```java
package com.platform.boardservice.post;

import com.platform.boardservice.TestAuth;
import com.platform.boardservice.common.NotFoundException;
import com.platform.boardservice.post.dto.PostCreateRequest;
import com.platform.boardservice.post.dto.PostResponse;
import com.platform.boardservice.security.AccessGuard;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest; // Spring Boot 4 패키지
import org.springframework.context.annotation.Import;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@DataJpaTest
@ActiveProfiles("test")
@Import({PostService.class, AccessGuard.class})
class PostServiceTest {

    @Autowired PostService service;

    @Test
    void createsPostWithAuthorFromToken() {
        PostResponse res = service.create(new PostCreateRequest("제목", "내용"), TestAuth.user(42L, "Alice"));
        assertThat(res.id()).isNotNull();
        assertThat(res.authorId()).isEqualTo(42L);
        assertThat(res.authorName()).isEqualTo("Alice");
    }

    @Test
    void getMissingThrowsNotFound() {
        assertThatThrownBy(() -> service.get(999L)).isInstanceOf(NotFoundException.class);
    }
}
```

- [ ] **Step 4: 테스트 실행(실패 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.post.PostServiceTest"`
Expected: FAIL — `PostService` 없음.

- [ ] **Step 5: PostService 작성**

```java
package com.platform.boardservice.post;

import com.platform.boardservice.common.NotFoundException;
import com.platform.boardservice.post.dto.PostCreateRequest;
import com.platform.boardservice.post.dto.PostResponse;
import com.platform.boardservice.post.dto.PostSummaryResponse;
import com.platform.boardservice.security.AccessGuard;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class PostService {

    private final PostRepository repository;
    private final AccessGuard accessGuard;

    public PostService(PostRepository repository, AccessGuard accessGuard) {
        this.repository = repository;
        this.accessGuard = accessGuard;
    }

    @Transactional
    public PostResponse create(PostCreateRequest req, Authentication auth) {
        Post post = new Post(req.title(), req.content(),
                accessGuard.currentUserId(auth), accessGuard.currentUserName(auth));
        return PostResponse.from(repository.save(post));
    }

    @Transactional(readOnly = true)
    public PostResponse get(Long id) {
        return PostResponse.from(find(id));
    }

    @Transactional(readOnly = true)
    public Page<PostSummaryResponse> list(Pageable pageable) {
        return repository.findAll(pageable).map(PostSummaryResponse::from);
    }

    Post find(Long id) {
        return repository.findById(id)
                .orElseThrow(() -> new NotFoundException("게시글을 찾을 수 없습니다: " + id));
    }
}
```

- [ ] **Step 6: 테스트 실행(통과 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.post.PostServiceTest"`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add board-service
git commit -m "feat(board-service): Post DTO + 생성/조회/목록 서비스"
```

---

### Task 5: Post 수정/삭제 서비스 (본인 OR ADMIN)

**Files:**
- Modify: `board-service/src/main/java/com/platform/boardservice/post/PostService.java`
- Modify: `board-service/src/test/java/com/platform/boardservice/post/PostServiceTest.java`

**Interfaces:**
- Produces: `PostService.update(Long id, PostUpdateRequest req, Authentication auth)` → `PostResponse`; `PostService.delete(Long id, Authentication auth)` → `void`. 둘 다 `AccessGuard.requireOwnerOrAdmin` 적용.

- [ ] **Step 1: 실패 테스트 추가** (PostServiceTest에 메서드 추가)

```java
    @Test
    void ownerCanUpdate() {
        var created = service.create(new com.platform.boardservice.post.dto.PostCreateRequest("t", "c"),
                TestAuth.user(42L, "Alice"));
        var res = service.update(created.id(),
                new com.platform.boardservice.post.dto.PostUpdateRequest("t2", "c2"),
                TestAuth.user(42L, "Alice"));
        assertThat(res.title()).isEqualTo("t2");
    }

    @Test
    void otherUserCannotUpdate() {
        var created = service.create(new com.platform.boardservice.post.dto.PostCreateRequest("t", "c"),
                TestAuth.user(42L, "Alice"));
        assertThatThrownBy(() -> service.update(created.id(),
                new com.platform.boardservice.post.dto.PostUpdateRequest("x", "y"),
                TestAuth.user(7L, "Mallory")))
                .isInstanceOf(org.springframework.security.access.AccessDeniedException.class);
    }

    @Test
    void adminCanDeleteOthersPost() {
        var created = service.create(new com.platform.boardservice.post.dto.PostCreateRequest("t", "c"),
                TestAuth.user(42L, "Alice"));
        service.delete(created.id(), TestAuth.admin(1L, "Root"));
        assertThatThrownBy(() -> service.get(created.id())).isInstanceOf(NotFoundException.class);
    }
```

- [ ] **Step 2: 테스트 실행(실패 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.post.PostServiceTest"`
Expected: FAIL — `update`/`delete` 없음.

- [ ] **Step 3: PostService에 update/delete 추가**

`import` 추가: `com.platform.boardservice.post.dto.PostUpdateRequest;`
클래스 본문에 메서드 추가:
```java
    @Transactional
    public PostResponse update(Long id, PostUpdateRequest req, Authentication auth) {
        Post post = find(id);
        accessGuard.requireOwnerOrAdmin(post.getAuthorId(), auth);
        post.edit(req.title(), req.content());
        return PostResponse.from(post);
    }

    @Transactional
    public void delete(Long id, Authentication auth) {
        Post post = find(id);
        accessGuard.requireOwnerOrAdmin(post.getAuthorId(), auth);
        repository.delete(post);
    }
```

- [ ] **Step 4: 테스트 실행(통과 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.post.PostServiceTest"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add board-service
git commit -m "feat(board-service): Post 수정/삭제 (본인 OR ADMIN)"
```

---

### Task 6: PostController + 에러 핸들러 + MockMvc 테스트

**Files:**
- Create: `board-service/src/main/java/com/platform/boardservice/common/ApiExceptionHandler.java`
- Create: `board-service/src/main/java/com/platform/boardservice/post/PostController.java`
- Test: `board-service/src/test/java/com/platform/boardservice/post/PostControllerTest.java`

**Interfaces:**
- Produces: REST 엔드포인트 `GET /api/posts`, `GET /api/posts/{id}`, `POST /api/posts`(201), `PUT /api/posts/{id}`, `DELETE /api/posts/{id}`(204). 목록 기본 정렬 `createdAt DESC`, size 20.
- Produces: `ApiExceptionHandler` — 404/403/400 매핑.

- [ ] **Step 1: ApiExceptionHandler 작성**

```java
package com.platform.boardservice.common;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.Map;

@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    ResponseEntity<Map<String, Object>> notFound(NotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(Map.of("error", "not_found", "message", e.getMessage()));
    }

    @ExceptionHandler(AccessDeniedException.class)
    ResponseEntity<Map<String, Object>> denied(AccessDeniedException e) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
                .body(Map.of("error", "forbidden", "message", e.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    ResponseEntity<Map<String, Object>> invalid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getAllErrors().get(0).getDefaultMessage();
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(Map.of("error", "validation_failed", "message", msg));
    }
}
```

- [ ] **Step 2: PostController 작성**

```java
package com.platform.boardservice.post;

import com.platform.boardservice.post.dto.PostCreateRequest;
import com.platform.boardservice.post.dto.PostResponse;
import com.platform.boardservice.post.dto.PostSummaryResponse;
import com.platform.boardservice.post.dto.PostUpdateRequest;
import jakarta.validation.Valid;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/posts")
public class PostController {

    private final PostService service;

    public PostController(PostService service) {
        this.service = service;
    }

    @GetMapping
    public Page<PostSummaryResponse> list(
            @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC) Pageable pageable) {
        return service.list(pageable);
    }

    @GetMapping("/{id}")
    public PostResponse get(@PathVariable Long id) {
        return service.get(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public PostResponse create(@Valid @RequestBody PostCreateRequest req, Authentication auth) {
        return service.create(req, auth);
    }

    @PutMapping("/{id}")
    public PostResponse update(@PathVariable Long id, @Valid @RequestBody PostUpdateRequest req, Authentication auth) {
        return service.update(id, req, auth);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id, Authentication auth) {
        service.delete(id, auth);
    }
}
```

- [ ] **Step 3: MockMvc 테스트 작성**

```java
package com.platform.boardservice.post;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;
import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@ActiveProfiles("test")
class PostControllerTest {

    @Autowired WebApplicationContext context;
    MockMvc mvc;

    @BeforeEach
    void setup() {
        mvc = MockMvcBuilders.webAppContextSetup(context).apply(springSecurity()).build();
    }

    // sub=userId, ROLE을 명시(jwt() post-processor는 컨버터를 거치지 않으므로 authorities 직접 지정).
    private static org.springframework.test.web.servlet.request.RequestPostProcessor asUser(long id, String name) {
        return jwt().jwt(j -> j.subject(String.valueOf(id)).claim("name", name).claim("roles", java.util.List.of("USER")))
                .authorities(new SimpleGrantedAuthority("ROLE_USER"));
    }

    private static org.springframework.test.web.servlet.request.RequestPostProcessor asAdmin(long id, String name) {
        return jwt().jwt(j -> j.subject(String.valueOf(id)).claim("name", name).claim("roles", java.util.List.of("ADMIN")))
                .authorities(new SimpleGrantedAuthority("ROLE_ADMIN"));
    }

    private Long createPost(long userId) throws Exception {
        String body = mvc.perform(post("/api/posts").with(asUser(userId, "Alice"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"title\":\"t\",\"content\":\"c\"}"))
                .andExpect(status().isCreated())
                .andReturn().getResponse().getContentAsString();
        return com.jayway.jsonpath.JsonPath.parse(body).read("$.id", Long.class);
    }

    @Test
    void listIsPublic() throws Exception {
        mvc.perform(get("/api/posts")).andExpect(status().isOk());
    }

    @Test
    void createRequiresAuth() throws Exception {
        mvc.perform(post("/api/posts").contentType(MediaType.APPLICATION_JSON)
                        .content("{\"title\":\"t\",\"content\":\"c\"}"))
                .andExpect(status().isUnauthorized());
    }

    @Test
    void blankTitleIsRejected() throws Exception {
        mvc.perform(post("/api/posts").with(asUser(42L, "Alice"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"title\":\"\",\"content\":\"c\"}"))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.error").value("validation_failed"));
    }

    @Test
    void ownerCanUpdate() throws Exception {
        Long id = createPost(42L);
        mvc.perform(put("/api/posts/" + id).with(asUser(42L, "Alice"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"title\":\"t2\",\"content\":\"c2\"}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.title").value("t2"));
    }

    @Test
    void otherUserGetsForbidden() throws Exception {
        Long id = createPost(42L);
        mvc.perform(put("/api/posts/" + id).with(asUser(7L, "Mallory"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"title\":\"x\",\"content\":\"y\"}"))
                .andExpect(status().isForbidden());
    }

    @Test
    void adminCanDeleteOthersPost() throws Exception {
        Long id = createPost(42L);
        mvc.perform(delete("/api/posts/" + id).with(asAdmin(1L, "Root")))
                .andExpect(status().isNoContent());
    }

    @Test
    void missingPostReturns404() throws Exception {
        mvc.perform(get("/api/posts/999999")).andExpect(status().isNotFound());
    }
}
```

- [ ] **Step 4: 테스트 실행(통과 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.post.PostControllerTest"`
Expected: PASS (7개 테스트).

> AccessDeniedException이 403으로 매핑되는지 확인. 만약 403이 아니라 500이면 `ApiExceptionHandler`의 `AccessDeniedException` import가 `org.springframework.security.access.AccessDeniedException`인지 확인(동일 클래스를 던지고 받아야 함).

- [ ] **Step 5: Commit**

```bash
git add board-service
git commit -m "feat(board-service): PostController + 에러 핸들러 + 인가 통합 테스트"
```

---

### Task 7: Comment 엔티티 + 마이그레이션 + 리포지토리

**Files:**
- Create: `board-service/src/main/resources/db/migration/V2__add_comment.sql`
- Create: `board-service/src/main/java/com/platform/boardservice/comment/Comment.java`
- Create: `board-service/src/main/java/com/platform/boardservice/comment/CommentRepository.java`
- Test: `board-service/src/test/java/com/platform/boardservice/comment/CommentRepositoryTest.java`

**Interfaces:**
- Produces: `Comment(Long postId, String content, Long authorId, String authorName)` 생성자, getter `getId/getPostId/getContent/getAuthorId/getAuthorName/getCreatedAt/getUpdatedAt`, `void edit(String content)`.
- Produces: `CommentRepository extends JpaRepository<Comment, Long>` — `Page<Comment> findByPostId(Long postId, Pageable pageable)`, `void deleteByPostId(Long postId)`.

- [ ] **Step 1: 실패 테스트 작성**

```java
package com.platform.boardservice.comment;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest; // Spring Boot 4 패키지
import org.springframework.data.domain.PageRequest;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@ActiveProfiles("test")
class CommentRepositoryTest {

    @Autowired CommentRepository repository;

    @Test
    void savesAndFindsByPost() {
        repository.save(new Comment(1L, "댓글", 42L, "Alice"));
        repository.save(new Comment(1L, "댓글2", 43L, "Bob"));
        repository.save(new Comment(2L, "다른글댓글", 42L, "Alice"));
        assertThat(repository.findByPostId(1L, PageRequest.of(0, 10)).getTotalElements()).isEqualTo(2);
    }

    @Test
    void deletesByPost() {
        repository.save(new Comment(1L, "댓글", 42L, "Alice"));
        repository.deleteByPostId(1L);
        assertThat(repository.findByPostId(1L, PageRequest.of(0, 10)).getTotalElements()).isZero();
    }
}
```

- [ ] **Step 2: 테스트 실행(실패 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.comment.CommentRepositoryTest"`
Expected: FAIL — `Comment`/`CommentRepository` 없음.

- [ ] **Step 3: V2__add_comment.sql 작성**

```sql
CREATE TABLE comment (
    id          BIGSERIAL PRIMARY KEY,
    post_id     BIGINT NOT NULL REFERENCES post(id) ON DELETE CASCADE,
    content     TEXT NOT NULL,
    author_id   BIGINT NOT NULL,
    author_name VARCHAR(255) NOT NULL,
    created_at  TIMESTAMP NOT NULL,
    updated_at  TIMESTAMP NOT NULL
);
CREATE INDEX idx_comment_post ON comment(post_id);
```

- [ ] **Step 4: Comment 엔티티 작성**

```java
package com.platform.boardservice.comment;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;

import java.time.Instant;

@Entity
@Table(name = "comment")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Comment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "post_id", nullable = false)
    private Long postId;

    @Column(nullable = false, columnDefinition = "text")
    private String content;

    @Column(name = "author_id", nullable = false)
    private Long authorId;

    @Column(name = "author_name", nullable = false)
    private String authorName;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    public Comment(Long postId, String content, Long authorId, String authorName) {
        this.postId = postId;
        this.content = content;
        this.authorId = authorId;
        this.authorName = authorName;
        Instant now = Instant.now();
        this.createdAt = now;
        this.updatedAt = now;
    }

    public void edit(String content) {
        this.content = content;
        this.updatedAt = Instant.now();
    }
}
```

- [ ] **Step 5: CommentRepository 작성**

```java
package com.platform.boardservice.comment;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface CommentRepository extends JpaRepository<Comment, Long> {
    Page<Comment> findByPostId(Long postId, Pageable pageable);
    void deleteByPostId(Long postId);
}
```

- [ ] **Step 6: 테스트 실행(통과 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.comment.CommentRepositoryTest"`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add board-service
git commit -m "feat(board-service): Comment 엔티티/마이그레이션/리포지토리"
```

---

### Task 8: Comment DTO + 서비스 + 글 삭제 시 댓글 정리

**Files:**
- Create: `board-service/src/main/java/com/platform/boardservice/comment/dto/CommentCreateRequest.java`
- Create: `board-service/src/main/java/com/platform/boardservice/comment/dto/CommentUpdateRequest.java`
- Create: `board-service/src/main/java/com/platform/boardservice/comment/dto/CommentResponse.java`
- Create: `board-service/src/main/java/com/platform/boardservice/comment/CommentService.java`
- Modify: `board-service/src/main/java/com/platform/boardservice/post/PostService.java` (delete 시 댓글 정리)
- Test: `board-service/src/test/java/com/platform/boardservice/comment/CommentServiceTest.java`

**Interfaces:**
- Produces: records `CommentCreateRequest(String content)`, `CommentUpdateRequest(String content)`(둘 다 `@NotBlank`), `CommentResponse(Long id, Long postId, String content, Long authorId, String authorName, Instant createdAt, Instant updatedAt)` + `static from(Comment)`.
- Produces: `CommentService` — `CommentResponse create(Long postId, CommentCreateRequest, Authentication)`(post 없으면 `NotFoundException`), `Page<CommentResponse> list(Long postId, Pageable)`, `CommentResponse update(Long id, CommentUpdateRequest, Authentication)`, `void delete(Long id, Authentication)`. 수정/삭제는 `AccessGuard.requireOwnerOrAdmin`.
- Consumes: `PostRepository`(존재 확인), `CommentRepository`, `AccessGuard`.
- Produces(modify): `PostService.delete`가 `CommentRepository.deleteByPostId(id)` 호출 후 글 삭제.

- [ ] **Step 1: DTO 3종 작성**

`CommentCreateRequest.java`:
```java
package com.platform.boardservice.comment.dto;

import jakarta.validation.constraints.NotBlank;

public record CommentCreateRequest(@NotBlank(message = "content는 필수입니다.") String content) {
}
```

`CommentUpdateRequest.java`:
```java
package com.platform.boardservice.comment.dto;

import jakarta.validation.constraints.NotBlank;

public record CommentUpdateRequest(@NotBlank(message = "content는 필수입니다.") String content) {
}
```

`CommentResponse.java`:
```java
package com.platform.boardservice.comment.dto;

import com.platform.boardservice.comment.Comment;

import java.time.Instant;

public record CommentResponse(Long id, Long postId, String content, Long authorId,
                              String authorName, Instant createdAt, Instant updatedAt) {
    public static CommentResponse from(Comment c) {
        return new CommentResponse(c.getId(), c.getPostId(), c.getContent(),
                c.getAuthorId(), c.getAuthorName(), c.getCreatedAt(), c.getUpdatedAt());
    }
}
```

- [ ] **Step 2: 실패 테스트 작성**

```java
package com.platform.boardservice.comment;

import com.platform.boardservice.TestAuth;
import com.platform.boardservice.common.NotFoundException;
import com.platform.boardservice.comment.dto.CommentCreateRequest;
import com.platform.boardservice.comment.dto.CommentResponse;
import com.platform.boardservice.post.Post;
import com.platform.boardservice.post.PostRepository;
import com.platform.boardservice.security.AccessGuard;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest; // Spring Boot 4 패키지
import org.springframework.context.annotation.Import;
import org.springframework.data.domain.PageRequest;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.*;

@DataJpaTest
@ActiveProfiles("test")
@Import({CommentService.class, AccessGuard.class})
class CommentServiceTest {

    @Autowired CommentService service;
    @Autowired PostRepository postRepository;

    private Long newPost() {
        return postRepository.save(new Post("t", "c", 42L, "Alice")).getId();
    }

    @Test
    void createsCommentUnderExistingPost() {
        Long postId = newPost();
        CommentResponse res = service.create(postId, new CommentCreateRequest("좋아요"), TestAuth.user(7L, "Bob"));
        assertThat(res.id()).isNotNull();
        assertThat(res.postId()).isEqualTo(postId);
        assertThat(res.authorId()).isEqualTo(7L);
    }

    @Test
    void createOnMissingPostThrows() {
        assertThatThrownBy(() -> service.create(999L, new CommentCreateRequest("x"), TestAuth.user(7L, "Bob")))
                .isInstanceOf(NotFoundException.class);
    }

    @Test
    void otherUserCannotUpdateComment() {
        Long postId = newPost();
        var c = service.create(postId, new CommentCreateRequest("hi"), TestAuth.user(7L, "Bob"));
        assertThatThrownBy(() -> service.update(c.id(),
                new com.platform.boardservice.comment.dto.CommentUpdateRequest("edit"),
                TestAuth.user(8L, "Eve")))
                .isInstanceOf(AccessDeniedException.class);
    }

    @Test
    void listsCommentsByPost() {
        Long postId = newPost();
        service.create(postId, new CommentCreateRequest("a"), TestAuth.user(7L, "Bob"));
        service.create(postId, new CommentCreateRequest("b"), TestAuth.user(7L, "Bob"));
        assertThat(service.list(postId, PageRequest.of(0, 10)).getTotalElements()).isEqualTo(2);
    }
}
```

- [ ] **Step 3: 테스트 실행(실패 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.comment.CommentServiceTest"`
Expected: FAIL — `CommentService` 없음.

- [ ] **Step 4: CommentService 작성**

```java
package com.platform.boardservice.comment;

import com.platform.boardservice.comment.dto.CommentCreateRequest;
import com.platform.boardservice.comment.dto.CommentResponse;
import com.platform.boardservice.comment.dto.CommentUpdateRequest;
import com.platform.boardservice.common.NotFoundException;
import com.platform.boardservice.post.PostRepository;
import com.platform.boardservice.security.AccessGuard;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class CommentService {

    private final CommentRepository repository;
    private final PostRepository postRepository;
    private final AccessGuard accessGuard;

    public CommentService(CommentRepository repository, PostRepository postRepository, AccessGuard accessGuard) {
        this.repository = repository;
        this.postRepository = postRepository;
        this.accessGuard = accessGuard;
    }

    @Transactional
    public CommentResponse create(Long postId, CommentCreateRequest req, Authentication auth) {
        if (!postRepository.existsById(postId)) {
            throw new NotFoundException("게시글을 찾을 수 없습니다: " + postId);
        }
        Comment comment = new Comment(postId, req.content(),
                accessGuard.currentUserId(auth), accessGuard.currentUserName(auth));
        return CommentResponse.from(repository.save(comment));
    }

    @Transactional(readOnly = true)
    public Page<CommentResponse> list(Long postId, Pageable pageable) {
        return repository.findByPostId(postId, pageable).map(CommentResponse::from);
    }

    @Transactional
    public CommentResponse update(Long id, CommentUpdateRequest req, Authentication auth) {
        Comment comment = find(id);
        accessGuard.requireOwnerOrAdmin(comment.getAuthorId(), auth);
        comment.edit(req.content());
        return CommentResponse.from(comment);
    }

    @Transactional
    public void delete(Long id, Authentication auth) {
        Comment comment = find(id);
        accessGuard.requireOwnerOrAdmin(comment.getAuthorId(), auth);
        repository.delete(comment);
    }

    private Comment find(Long id) {
        return repository.findById(id)
                .orElseThrow(() -> new NotFoundException("댓글을 찾을 수 없습니다: " + id));
    }
}
```

- [ ] **Step 5: PostService.delete가 댓글을 먼저 정리하도록 수정**

`PostService`에 `CommentRepository` 의존성 추가(생성자 주입) 후 `delete`에서 `commentRepository.deleteByPostId(id)` 호출.
import 추가: `import com.platform.boardservice.comment.CommentRepository;`
필드/생성자 변경:
```java
    private final PostRepository repository;
    private final AccessGuard accessGuard;
    private final CommentRepository commentRepository;

    public PostService(PostRepository repository, AccessGuard accessGuard, CommentRepository commentRepository) {
        this.repository = repository;
        this.accessGuard = accessGuard;
        this.commentRepository = commentRepository;
    }
```
`delete` 메서드 변경:
```java
    @Transactional
    public void delete(Long id, Authentication auth) {
        Post post = find(id);
        accessGuard.requireOwnerOrAdmin(post.getAuthorId(), auth);
        commentRepository.deleteByPostId(id); // 글 삭제 시 댓글 cascade 정리(H2/PG 모두 보장)
        repository.delete(post);
    }
```

> 기존 `PostServiceTest`의 `@Import`에 `CommentRepository`는 `@DataJpaTest`가 모든 리포지토리를 자동 등록하므로 별도 추가 불필요. 단 `@Import({PostService.class, AccessGuard.class})`는 그대로 두면 된다(`CommentRepository`는 자동 스캔).

- [ ] **Step 6: 전체 테스트 실행(통과 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.comment.CommentServiceTest" --tests "com.platform.boardservice.post.PostServiceTest"`
Expected: PASS — CommentService 4개 + PostService 5개. (PostService.delete 시그니처는 동일하므로 기존 테스트 영향 없음)

- [ ] **Step 7: Commit**

```bash
git add board-service
git commit -m "feat(board-service): Comment DTO/서비스 + 글 삭제 시 댓글 정리"
```

---

### Task 9: CommentController + MockMvc 테스트

**Files:**
- Create: `board-service/src/main/java/com/platform/boardservice/comment/CommentController.java`
- Test: `board-service/src/test/java/com/platform/boardservice/comment/CommentControllerTest.java`

**Interfaces:**
- Produces: `POST /api/posts/{postId}/comments`(201, 인증), `GET /api/posts/{postId}/comments`(공개), `PUT /api/comments/{id}`(인증+소유), `DELETE /api/comments/{id}`(204, 인증+소유).

- [ ] **Step 1: CommentController 작성**

```java
package com.platform.boardservice.comment;

import com.platform.boardservice.comment.dto.CommentCreateRequest;
import com.platform.boardservice.comment.dto.CommentResponse;
import com.platform.boardservice.comment.dto.CommentUpdateRequest;
import jakarta.validation.Valid;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api")
public class CommentController {

    private final CommentService service;

    public CommentController(CommentService service) {
        this.service = service;
    }

    @GetMapping("/posts/{postId}/comments")
    public Page<CommentResponse> list(@PathVariable Long postId,
            @PageableDefault(size = 50, sort = "createdAt", direction = Sort.Direction.ASC) Pageable pageable) {
        return service.list(postId, pageable);
    }

    @PostMapping("/posts/{postId}/comments")
    @ResponseStatus(HttpStatus.CREATED)
    public CommentResponse create(@PathVariable Long postId,
            @Valid @RequestBody CommentCreateRequest req, Authentication auth) {
        return service.create(postId, req, auth);
    }

    @PutMapping("/comments/{id}")
    public CommentResponse update(@PathVariable Long id,
            @Valid @RequestBody CommentUpdateRequest req, Authentication auth) {
        return service.update(id, req, auth);
    }

    @DeleteMapping("/comments/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id, Authentication auth) {
        service.delete(id, auth);
    }
}
```

> `GET /api/posts/{postId}/comments`는 SecurityConfig의 `GET /api/posts/**` permitAll에 포함되어 공개. `PUT/DELETE /api/comments/**`는 anyRequest authenticated에 걸려 인증 필요.

- [ ] **Step 2: MockMvc 테스트 작성**

```java
package com.platform.boardservice.comment;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.RequestPostProcessor;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import java.util.List;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;
import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@ActiveProfiles("test")
class CommentControllerTest {

    @Autowired WebApplicationContext context;
    MockMvc mvc;

    @BeforeEach
    void setup() {
        mvc = MockMvcBuilders.webAppContextSetup(context).apply(springSecurity()).build();
    }

    private static RequestPostProcessor asUser(long id, String name) {
        return jwt().jwt(j -> j.subject(String.valueOf(id)).claim("name", name).claim("roles", List.of("USER")))
                .authorities(new SimpleGrantedAuthority("ROLE_USER"));
    }

    private Long createPost(long userId) throws Exception {
        String body = mvc.perform(post("/api/posts").with(asUser(userId, "Alice"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"title\":\"t\",\"content\":\"c\"}"))
                .andExpect(status().isCreated())
                .andReturn().getResponse().getContentAsString();
        return com.jayway.jsonpath.JsonPath.parse(body).read("$.id", Long.class);
    }

    private Long createComment(long postId, long userId) throws Exception {
        String body = mvc.perform(post("/api/posts/" + postId + "/comments").with(asUser(userId, "Bob"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"content\":\"hi\"}"))
                .andExpect(status().isCreated())
                .andReturn().getResponse().getContentAsString();
        return com.jayway.jsonpath.JsonPath.parse(body).read("$.id", Long.class);
    }

    @Test
    void createRequiresAuth() throws Exception {
        Long postId = createPost(42L);
        mvc.perform(post("/api/posts/" + postId + "/comments")
                        .contentType(MediaType.APPLICATION_JSON).content("{\"content\":\"hi\"}"))
                .andExpect(status().isUnauthorized());
    }

    @Test
    void listIsPublic() throws Exception {
        Long postId = createPost(42L);
        createComment(postId, 7L);
        mvc.perform(get("/api/posts/" + postId + "/comments"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.totalElements").value(1));
    }

    @Test
    void ownerCanUpdateComment() throws Exception {
        Long postId = createPost(42L);
        Long cid = createComment(postId, 7L);
        mvc.perform(put("/api/comments/" + cid).with(asUser(7L, "Bob"))
                        .contentType(MediaType.APPLICATION_JSON).content("{\"content\":\"edited\"}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.content").value("edited"));
    }

    @Test
    void otherUserGetsForbidden() throws Exception {
        Long postId = createPost(42L);
        Long cid = createComment(postId, 7L);
        mvc.perform(delete("/api/comments/" + cid).with(asUser(8L, "Eve")))
                .andExpect(status().isForbidden());
    }

    @Test
    void commentOnMissingPostReturns404() throws Exception {
        mvc.perform(post("/api/posts/999999/comments").with(asUser(7L, "Bob"))
                        .contentType(MediaType.APPLICATION_JSON).content("{\"content\":\"hi\"}"))
                .andExpect(status().isNotFound());
    }
}
```

- [ ] **Step 3: 테스트 실행(통과 확인)**

Run: `./gradlew test --tests "com.platform.boardservice.comment.CommentControllerTest"`
Expected: PASS (5개).

- [ ] **Step 4: 전체 테스트 실행**

Run: `./gradlew test`
Expected: 전체 PASS (Application/Repository/Service/Controller 테스트 모두 green).

- [ ] **Step 5: Commit**

```bash
git add board-service
git commit -m "feat(board-service): CommentController + 인가 통합 테스트"
```

---

## Self-Review Notes

**Spec coverage 확인:**
- 게시글 CRUD → Task 4(생성/조회/목록), 5(수정/삭제), 6(컨트롤러). ✅
- 댓글 CRUD → Task 7~9. ✅
- JWT 검증(jwk-set-uri) → Task 1 SecurityConfig + application.yml. ✅
- Role 매핑(roles→ROLE_*) → Task 1 `jwtAuthenticationConverter`. ✅
- Method Security(@EnableMethodSecurity) + 본인 OR ADMIN → Task 1 + Task 3 AccessGuard, Task 5/8 적용. ✅
- 공개 GET / 인증 쓰기 → Task 1 authorizeHttpRequests + Task 6/9 테스트. ✅
- 댓글 cascade 삭제 → Task 7 DB FK + Task 8 `deleteByPostId`. ✅
- 에러 401/403/404/400 → Task 6 ApiExceptionHandler + 테스트. ✅
- 독립 디렉터리/포트 9100/boarddb → Task 1. ✅
- 테스트(jwt() post-processor, H2) → 전 태스크. ✅

**범위 밖(YAGNI) 확인:** 게이트웨이/검색/카테고리/조회수/첨부/좋아요/soft-delete 미포함. ✅

**타입 일관성 확인:** `AccessGuard.requireOwnerOrAdmin(long, Authentication)`, `currentUserId/currentUserName/isAdmin(Authentication)` — Task 3 정의와 Task 5/8 사용 일치. `PostResponse.from`/`PostSummaryResponse.from`/`CommentResponse.from` 시그니처 일관. `PostService.delete(Long, Authentication)` 시그니처는 Task 5→8 동일(내부만 변경).

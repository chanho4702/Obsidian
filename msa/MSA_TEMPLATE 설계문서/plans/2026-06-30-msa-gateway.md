# MSA 게이트웨이 도입 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 단일 진입점 API 게이트웨이(`gateway-server`, :8000)를 도입해 myFront가 게이트웨이 하나만 바라보게 하고, board API 경로를 서비스명 prefix(`/api/board/**`)로 통일한다.

**Architecture:** Spring Cloud Gateway(WebFlux)가 순수 라우팅 + CORS 중앙화 + 요청 로깅/traceId 전파를 담당한다. 토큰 검증은 하지 않는다(각 서비스가 JWKS로 자체 검증, 이미 배선됨). 다운스트림 위치는 정적 설정 + Docker DNS로 해소한다.

**Tech Stack:** Spring Boot 4.0.6 · Java 24 · Gradle · Spring Cloud 2025.1.2 (Oakwood) · spring-cloud-starter-gateway-server-webflux

## Global Constraints

- Spring Boot: `4.0.6` (모든 백엔드 모듈 동일)
- Java toolchain: `24`
- Spring Cloud BOM: `2025.1.2` (Boot 4.0.x 호환) — 게이트웨이 모듈에만 적용
- 게이트웨이 라우팅: **No StripPrefix** (클라이언트 경로 = 서비스 경로)
- 게이트웨이 포트: `8000` / auth-server `9000` / board-service `9100` / Keycloak `8080` / Postgres `5433` / myFront `5173`
- 게이트웨이에는 **Spring Security 미탑재** (토큰 검증 안 함)
- 컴파일 인자: 기존 모듈과 동일하게 `-parameters` 유지(파라미터 이름 보존)
- 커밋 메시지 말미: `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`
- 우산 repo는 `master` 브랜치. auth-server는 별도 git repo(`main`), board-service·myFront·infra는 우산 repo에 포함

## 모듈별 파일 책임

| 모듈 | 파일 | 책임 |
|---|---|---|
| gateway-server (신규) | `build.gradle`, `settings.gradle` | Gateway/Cloud BOM 의존성, 빌드 정의 |
| | `src/main/java/.../gateway/GatewayApplication.java` | 부트 진입점 |
| | `src/main/resources/application.yml` | 라우트(정적) + 다운스트림 URI(환경변수) + CORS |
| | `src/main/java/.../gateway/filter/RequestLoggingFilter.java` | X-Request-Id 생성/전파 + 요청 로깅 |
| board-service | `post/PostController.java`, `comment/CommentController.java` | `/api/board/**`로 매핑 변경 |
| | `config/SecurityConfig.java` | GET permit 경로 변경 + CORS 제거 |
| | 테스트 2종 | 경로 변경 / `CorsConfigTest` 삭제 |
| auth-server | `config/SecurityConfig.java` | CORS 빈/`.cors()`/필드 제거 |
| infra | `keycloak/realm-export.json` | `:8000` redirect URI 추가 |
| myFront | `.env`, `src/app/board/boardStore.ts` | base URL 통합 + board 경로 변경 |

---

## Task 1: board-service — `/api/posts` → `/api/board/**` 리네임

**Files:**
- Modify: `board-service/src/main/java/com/platform/boardservice/post/PostController.java:17`
- Modify: `board-service/src/main/java/com/platform/boardservice/comment/CommentController.java:16`
- Modify: `board-service/src/main/java/com/platform/boardservice/config/SecurityConfig.java:48`
- Test: `board-service/src/test/java/com/platform/boardservice/post/PostControllerTest.java`
- Test: `board-service/src/test/java/com/platform/boardservice/comment/CommentControllerTest.java`

**Interfaces:**
- Produces: board API가 `/api/board/posts`, `/api/board/posts/{postId}/comments`, `/api/board/comments/{id}` 경로로 노출됨 (게이트웨이 Task 4, 프론트 Task 8이 의존)

- [ ] **Step 1: 테스트 경로를 먼저 새 경로로 바꿔 실패시킨다 (PostControllerTest)**

`PostControllerTest.java`의 모든 `"/api/posts"` 리터럴을 `"/api/board/posts"`로 치환한다. 예:

```java
// 변경 전: mvc.perform(post("/api/posts").with(asUser(userId, "Alice"))
mvc.perform(post("/api/board/posts").with(asUser(userId, "Alice"))
// 변경 전: mvc.perform(get("/api/posts")).andExpect(status().isOk());
mvc.perform(get("/api/board/posts")).andExpect(status().isOk());
// 변경 전: put("/api/posts/" + id) / delete("/api/posts/" + id) / get("/api/posts/999999")
put("/api/board/posts/" + id)
delete("/api/board/posts/" + id)
get("/api/board/posts/999999")
```

- [ ] **Step 2: 테스트 경로를 새 경로로 바꿔 실패시킨다 (CommentControllerTest)**

`CommentControllerTest.java`에서 치환:

```java
post("/api/board/posts")                                  // 글 생성(셋업)
post("/api/board/posts/" + postId + "/comments")          // 댓글 생성
get("/api/board/posts/" + postId + "/comments")           // 댓글 목록
put("/api/board/comments/" + cid)                          // 댓글 수정
delete("/api/board/comments/" + cid)                       // 댓글 삭제
post("/api/board/posts/999999/comments")                   // 없는 글
```

- [ ] **Step 3: 테스트 실행해 실패 확인**

Run: `cd board-service && ./gradlew.bat test --tests "*PostControllerTest" --tests "*CommentControllerTest"`
(PowerShell: `$env:JAVA_HOME='C:\Program Files\Java\jdk-24'` 선행)
Expected: FAIL — 새 경로는 아직 컨트롤러에 없어 404(`status().isNotFound()` 등 불일치)

- [ ] **Step 4: PostController 매핑 변경**

`PostController.java:17`:

```java
@RestController
@RequestMapping("/api/board/posts")   // 변경 전: "/api/posts"
public class PostController {
```

- [ ] **Step 5: CommentController 매핑 변경**

`CommentController.java:16`의 클래스 레벨 prefix만 바꾼다(메서드 레벨 `@GetMapping("/posts/...")` 등은 그대로):

```java
@RestController
@RequestMapping("/api/board")   // 변경 전: "/api"
public class CommentController {
```

→ 결과 경로: `/api/board/posts/{postId}/comments`, `/api/board/comments/{id}`

- [ ] **Step 6: SecurityConfig의 GET permit 경로 변경**

`SecurityConfig.java:48`:

```java
.requestMatchers(HttpMethod.GET, "/api/board/posts/**").permitAll()   // 변경 전: "/api/posts/**"
```

- [ ] **Step 7: 테스트 실행해 통과 확인**

Run: `cd board-service && ./gradlew.bat test --tests "*PostControllerTest" --tests "*CommentControllerTest"`
Expected: PASS

- [ ] **Step 8: 커밋**

> board-service는 자체 repo(`backend-server`, branch `main`). 커밋은 board-service 디렉토리에서 수행.

```bash
git -C board-service add \
        src/main/java/com/platform/boardservice/post/PostController.java \
        src/main/java/com/platform/boardservice/comment/CommentController.java \
        src/main/java/com/platform/boardservice/config/SecurityConfig.java \
        src/test/java/com/platform/boardservice/post/PostControllerTest.java \
        src/test/java/com/platform/boardservice/comment/CommentControllerTest.java
git -C board-service commit -m "refactor: API 경로를 /api/board/** prefix로 리네임

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 2: board-service — 자체 CORS 제거 (게이트웨이로 일원화)

**Files:**
- Modify: `board-service/src/main/java/com/platform/boardservice/config/SecurityConfig.java`
- Delete: `board-service/src/test/java/com/platform/boardservice/config/CorsConfigTest.java`

**Interfaces:**
- Consumes: Task 1의 새 경로
- Produces: board-service는 CORS를 처리하지 않음(게이트웨이가 담당). 외부 직접 호출 시 CORS 헤더 없음

- [ ] **Step 1: CorsConfigTest 삭제**

`CorsConfigTest.java`를 삭제한다(CORS는 더 이상 board 책임이 아니므로 이 테스트는 무효).

```bash
git -C board-service rm src/test/java/com/platform/boardservice/config/CorsConfigTest.java
```

- [ ] **Step 2: SecurityConfig에서 CORS 제거**

`SecurityConfig.java`를 다음으로 만든다(생성자 `allowedOrigin` 필드, `.cors(...)`, `corsConfigurationSource` 빈, 관련 import 제거):

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
@EnableMethodSecurity // 수정/삭제는 AccessGuard(서비스 계층)로 판정.
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
                        .requestMatchers(HttpMethod.GET, "/api/board/posts/**").permitAll()
                        .anyRequest().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(converter)));
        return http.build();
    }
}
```

- [ ] **Step 3: 전체 테스트 실행해 통과 확인**

Run: `cd board-service && ./gradlew.bat test`
Expected: PASS (CorsConfigTest는 사라지고 나머지 통과)

- [ ] **Step 4: 커밋**

```bash
git -C board-service add src/main/java/com/platform/boardservice/config/SecurityConfig.java
git -C board-service rm src/test/java/com/platform/boardservice/config/CorsConfigTest.java
git -C board-service commit -m "refactor: 자체 CORS 제거 (게이트웨이로 일원화)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 3: auth-server — 자체 CORS 제거

**Files:**
- Modify: `auth-server/src/main/java/com/platform/authserver/config/SecurityConfig.java`

**Interfaces:**
- Produces: auth-server는 CORS를 처리하지 않음(게이트웨이가 담당)

> auth-server는 별도 git repo(`main` 브랜치). 커밋은 auth-server repo에서 수행.

- [ ] **Step 1: SecurityConfig에서 CORS 제거**

`SecurityConfig.java`에서 다음을 제거한다:
- 생성자와 `allowedOrigin` 필드 전체 (`SecurityConfig(Environment env)` 포함)
- `apiChain`·`webChain`의 `.cors(Customizer.withDefaults())` 라인
- `corsConfigurationSource()` 빈 메서드 전체
- 미사용 import (`CorsConfiguration`, `CorsConfigurationSource`, `UrlBasedCorsConfigurationSource`, `List`, `Customizer`)

결과 파일:

```java
package com.platform.authserver.config;

import com.platform.authserver.auth.LoginSuccessHandler;
import com.platform.authserver.jwt.JwtKeyProvider;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.security.oauth2.jwt.NimbusJwtDecoder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    JwtDecoder jwtDecoder(JwtKeyProvider keyProvider) throws Exception {
        return NimbusJwtDecoder.withPublicKey(keyProvider.publicKey()).build();
    }

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
    @Order(Ordered.HIGHEST_PRECEDENCE)
    SecurityFilterChain apiChain(HttpSecurity http, JwtAuthenticationConverter converter) throws Exception {
        http
                .securityMatcher("/api/me")
                .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(converter)));
        return http.build();
    }

    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE + 1)
    SecurityFilterChain webChain(HttpSecurity http, LoginSuccessHandler successHandler) throws Exception {
        http
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/auth/**", "/.well-known/**", "/error").permitAll()
                        .anyRequest().authenticated())
                .oauth2Login(oauth2 -> oauth2.successHandler(successHandler));
        return http.build();
    }
}
```

- [ ] **Step 2: 전체 테스트 실행해 통과 확인**

Run: `cd auth-server && ./gradlew.bat test`
Expected: PASS (auth-server엔 CORS 전용 테스트 없음 — 기존 14개 그대로 통과)

- [ ] **Step 3: 커밋 (auth-server repo)**

```bash
git -C auth-server add src/main/java/com/platform/authserver/config/SecurityConfig.java
git -C auth-server commit -m "refactor(auth-server): 자체 CORS 제거 (게이트웨이로 일원화)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 4: gateway-server — 모듈 스캐폴드 + 라우팅

**Files:**
- Create: `gateway-server/settings.gradle`
- Create: `gateway-server/build.gradle`
- Create: `gateway-server/src/main/java/com/platform/gateway/GatewayApplication.java`
- Create: `gateway-server/src/main/resources/application.yml`
- Create: `gateway-server/src/test/java/com/platform/gateway/RouteConfigTest.java`
- Copy: auth-server의 `gradlew`, `gradlew.bat`, `gradle/wrapper/*`를 `gateway-server/`로 복사

**Interfaces:**
- Produces: `:8000`에서 `/api/board/**`→board, `/oauth2/**`·`/login/**`·`/api/auth/**`·`/.well-known/**`·`/api/me`→auth 라우팅. 라우트 ID: `board`, `auth-oauth2`, `auth-login`, `auth-api`, `auth-jwks`, `auth-me`

> **저장소 구조:** gateway-server는 auth-server처럼 **자체 git repo**다. 우산 repo에서는
> gitignore되고, 모든 gateway-server 커밋은 gateway-server 자체 repo에서 수행한다(`git -C gateway-server ...`).

- [ ] **Step 0: 디렉토리/래퍼 준비 + git init + 우산 gitignore 등록**

auth-server의 래퍼를 그대로 복사(같은 Gradle 9.5.1)하고 자체 repo로 초기화:

```bash
mkdir -p gateway-server/gradle/wrapper
cp auth-server/gradlew gateway-server/gradlew
cp auth-server/gradlew.bat gateway-server/gradlew.bat
cp auth-server/gradle/wrapper/gradle-wrapper.jar gateway-server/gradle/wrapper/gradle-wrapper.jar
cp auth-server/gradle/wrapper/gradle-wrapper.properties gateway-server/gradle/wrapper/gradle-wrapper.properties
# 자체 repo 초기화
git -C gateway-server init
# 우산 .gitignore에 gateway-server 등록(auth-server와 동일 패턴)
printf '\n# gateway-server는 별도 repo\n/gateway-server/\n' >> .gitignore
git add .gitignore && git commit -m "chore: gateway-server를 별도 repo로 분리(gitignore)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

`gateway-server/.gitignore`도 생성(빌드 산출물 제외): `build/`, `.gradle/`, `out/`.

- [ ] **Step 2: settings.gradle 작성**

`gateway-server/settings.gradle`:

```groovy
rootProject.name = 'gateway-server'
```

- [ ] **Step 3: build.gradle 작성**

`gateway-server/build.gradle`:

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.0.6'
    id 'io.spring.dependency-management' version '1.1.7'
}
group = 'com.platform'
version = '0.0.1-SNAPSHOT'
java { toolchain { languageVersion = JavaLanguageVersion.of(24) } }
tasks.withType(JavaCompile).configureEach { options.compilerArgs << '-parameters' }
repositories { mavenCentral() }

ext {
    set('springCloudVersion', '2025.1.2')
}
dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-gateway-server-webflux'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.boot:spring-boot-starter-webflux' // WebTestClient
}
tasks.named('test') { useJUnitPlatform() }
```

- [ ] **Step 4: GatewayApplication 작성**

`gateway-server/src/main/java/com/platform/gateway/GatewayApplication.java`:

```java
package com.platform.gateway;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

- [ ] **Step 5: application.yml 작성 (라우트 정적 정의)**

`gateway-server/src/main/resources/application.yml`:

```yaml
server:
  port: 8000
spring:
  application:
    name: gateway-server
  cloud:
    gateway:
      server:
        webflux:
          routes:
            # board-service — 게시판/댓글. No StripPrefix(경로 그대로 전달).
            - id: board
              uri: ${BOARD_SERVICE_URI:http://localhost:9100}
              predicates:
                - Path=/api/board/**
            # auth-server — OIDC 로그인 리다이렉트 흐름
            - id: auth-oauth2
              uri: ${AUTH_SERVER_URI:http://localhost:9000}
              predicates:
                - Path=/oauth2/**
            - id: auth-login
              uri: ${AUTH_SERVER_URI:http://localhost:9000}
              predicates:
                - Path=/login/**
            # auth-server — refresh/logout
            - id: auth-api
              uri: ${AUTH_SERVER_URI:http://localhost:9000}
              predicates:
                - Path=/api/auth/**
            # auth-server — JWKS 공개키
            - id: auth-jwks
              uri: ${AUTH_SERVER_URI:http://localhost:9000}
              predicates:
                - Path=/.well-known/**
            # auth-server — 자체 JWT 검증 사용자정보
            - id: auth-me
              uri: ${AUTH_SERVER_URI:http://localhost:9000}
              predicates:
                - Path=/api/me
```

> Docker compose에서는 `BOARD_SERVICE_URI=http://board-service:9100`, `AUTH_SERVER_URI=http://auth-server:9000` 환경변수로 주입 — 같은 코드로 로컬/컨테이너 양쪽 동작(서비스명이 곧 DNS 호스트명).

- [ ] **Step 6: 라우트 검증 테스트 작성 (실패 확인용)**

`gateway-server/src/test/java/com/platform/gateway/RouteConfigTest.java`:

```java
package com.platform.gateway;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.gateway.server.mvc.common.Shortcut; // not used; placeholder removed below
import org.springframework.cloud.gateway.route.RouteLocator;
import reactor.test.StepVerifier;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
class RouteConfigTest {

    @Autowired
    RouteLocator routeLocator;

    @Test
    void allExpectedRoutesAreConfigured() {
        List<String> ids = routeLocator.getRoutes()
                .map(r -> r.getId())
                .collectList()
                .block();
        assertThat(ids).contains(
                "board", "auth-oauth2", "auth-login", "auth-api", "auth-jwks", "auth-me");
    }
}
```

> 주의: import `org.springframework.cloud.gateway.server.mvc.common.Shortcut`는 잘못된 예시다 — **삭제**하고, `RouteLocator`는 reactive Gateway 패키지(`org.springframework.cloud.gateway.route.RouteLocator`)를 쓴다. `StepVerifier` import도 사용하지 않으면 제거한다. (TDD 컴파일 단계에서 IDE/컴파일러 안내에 따라 미사용 import 정리.)

- [ ] **Step 7: 테스트 실행해 실패/통과 흐름 확인**

Run: `cd gateway-server && ./gradlew.bat test --tests "*RouteConfigTest"`
Expected: 처음엔 컴파일 에러(잘못된 import) → 정리 후 PASS. 라우트 6개가 모두 등록돼 있으면 통과.

- [ ] **Step 8: 로컬 부팅 스모크 체크**

Run: `cd gateway-server && ./gradlew.bat bootRun` (별도 터미널에서 auth-server·board-service·infra 기동 상태)
Expected: `:8000` 기동. `curl http://localhost:8000/.well-known/jwks.json` → auth-server의 JWK Set 200 응답(게이트웨이 경유 라우팅 확인).

- [ ] **Step 9: 커밋 (gateway-server 자체 repo)**

```bash
git -C gateway-server add .
git -C gateway-server commit -m "feat: Spring Cloud Gateway 모듈 + 정적 라우팅(:8000)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 5: gateway-server — CORS 중앙화

**Files:**
- Modify: `gateway-server/src/main/resources/application.yml`
- Test: `gateway-server/src/test/java/com/platform/gateway/CorsTest.java`

**Interfaces:**
- Consumes: Task 4의 라우트
- Produces: 게이트웨이가 `http://localhost:5173` 출처의 CORS 프리플라이트를 허용(credentials 포함)

- [ ] **Step 1: CORS 프리플라이트 테스트 작성 (실패 확인용)**

`gateway-server/src/test/java/com/platform/gateway/CorsTest.java`:

```java
package com.platform.gateway;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.test.web.reactive.server.WebTestClient;

import static org.springframework.boot.test.context.SpringBootTest.WebEnvironment.RANDOM_PORT;

@SpringBootTest(webEnvironment = RANDOM_PORT)
class CorsTest {

    @Autowired
    WebTestClient client;

    @Test
    void preflightFromFrontendIsAllowed() {
        client.method(HttpMethod.OPTIONS).uri("/api/board/posts")
                .header(HttpHeaders.ORIGIN, "http://localhost:5173")
                .header(HttpHeaders.ACCESS_CONTROL_REQUEST_METHOD, "POST")
                .exchange()
                .expectStatus().isOk()
                .expectHeader().valueEquals("Access-Control-Allow-Origin", "http://localhost:5173");
    }
}
```

- [ ] **Step 2: 테스트 실행해 실패 확인**

Run: `cd gateway-server && ./gradlew.bat test --tests "*CorsTest"`
Expected: FAIL — CORS 미설정이라 `Access-Control-Allow-Origin` 헤더 없음 (또는 프리플라이트가 라우팅으로 넘어가 실패)

- [ ] **Step 3: application.yml에 globalcors 추가**

`spring.cloud.gateway.server.webflux` 아래(`routes`와 동일 레벨)에 추가:

```yaml
spring:
  cloud:
    gateway:
      server:
        webflux:
          globalcors:
            cors-configurations:
              '[/**]':
                allowedOrigins: "${CORS_ALLOWED_ORIGIN:http://localhost:5173}"
                allowedMethods: [GET, POST, PUT, DELETE, OPTIONS]
                allowedHeaders: "*"
                allowCredentials: true
          routes:
            # ... (Task 4의 routes 그대로)
```

- [ ] **Step 4: 테스트 실행해 통과 확인**

Run: `cd gateway-server && ./gradlew.bat test --tests "*CorsTest"`
Expected: PASS

- [ ] **Step 5: 커밋 (gateway-server 자체 repo)**

```bash
git -C gateway-server add src/main/resources/application.yml src/test/java/com/platform/gateway/CorsTest.java
git -C gateway-server commit -m "feat: CORS 중앙화(globalcors)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 6: gateway-server — 요청 로깅 + traceId 전파

**Files:**
- Create: `gateway-server/src/main/java/com/platform/gateway/filter/RequestLoggingFilter.java`
- Test: `gateway-server/src/test/java/com/platform/gateway/filter/RequestLoggingFilterTest.java`

**Interfaces:**
- Consumes: 게이트웨이 요청 파이프라인
- Produces: 모든 요청에 `X-Request-Id` 헤더 보장(없으면 생성)하여 다운스트림으로 전파. 요청당 1줄 로깅

- [ ] **Step 1: 필터 테스트 작성 (실패 확인용)**

`gateway-server/src/test/java/com/platform/gateway/filter/RequestLoggingFilterTest.java`:

```java
package com.platform.gateway.filter;

import org.junit.jupiter.api.Test;
import org.springframework.mock.http.server.reactive.MockServerHttpRequest;
import org.springframework.mock.web.server.MockServerWebExchange;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import static org.assertj.core.api.Assertions.assertThat;

class RequestLoggingFilterTest {

    @Test
    void generatesRequestIdWhenAbsent() {
        RequestLoggingFilter filter = new RequestLoggingFilter();
        MockServerWebExchange exchange = MockServerWebExchange.from(
                MockServerHttpRequest.get("/api/board/posts").build());

        // chain: 전달된 exchange에서 X-Request-Id 헤더가 채워졌는지 캡처
        filter.filter(exchange, ex -> {
            String id = ex.getRequest().getHeaders().getFirst("X-Request-Id");
            assertThat(id).isNotBlank();
            return Mono.empty();
        }).block();
    }

    @Test
    void preservesExistingRequestId() {
        RequestLoggingFilter filter = new RequestLoggingFilter();
        MockServerWebExchange exchange = MockServerWebExchange.from(
                MockServerHttpRequest.get("/api/board/posts")
                        .header("X-Request-Id", "fixed-123").build());

        filter.filter(exchange, ex -> {
            assertThat(ex.getRequest().getHeaders().getFirst("X-Request-Id")).isEqualTo("fixed-123");
            return Mono.empty();
        }).block();
    }
}
```

- [ ] **Step 2: 테스트 실행해 실패 확인**

Run: `cd gateway-server && ./gradlew.bat test --tests "*RequestLoggingFilterTest"`
Expected: FAIL — `RequestLoggingFilter` 클래스 없음(컴파일 에러)

- [ ] **Step 3: RequestLoggingFilter 구현**

`gateway-server/src/main/java/com/platform/gateway/filter/RequestLoggingFilter.java`:

```java
package com.platform.gateway.filter;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.UUID;

/** 모든 요청에 X-Request-Id를 보장(없으면 생성)해 다운스트림으로 전파하고, 요청을 1줄 로깅한다. */
@Component
public class RequestLoggingFilter implements GlobalFilter, Ordered {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);
    static final String REQUEST_ID = "X-Request-Id";

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String requestId = exchange.getRequest().getHeaders().getFirst(REQUEST_ID);
        if (requestId == null || requestId.isBlank()) {
            requestId = UUID.randomUUID().toString();
        }
        final String id = requestId;
        ServerWebExchange mutated = exchange.mutate()
                .request(r -> r.header(REQUEST_ID, id))
                .build();
        log.info("{} {} ({})",
                mutated.getRequest().getMethod(),
                mutated.getRequest().getURI().getPath(),
                id);
        return chain.filter(mutated);
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```

> 테스트의 `chain` 람다는 `GatewayFilterChain`(단일 메서드 `filter(exchange)`)에 대입된다. 시그니처가 맞지 않으면 테스트에서 `GatewayFilterChain` 익명 구현으로 바꾼다.

- [ ] **Step 4: 테스트 실행해 통과 확인**

Run: `cd gateway-server && ./gradlew.bat test --tests "*RequestLoggingFilterTest"`
Expected: PASS

- [ ] **Step 5: 전체 테스트 실행**

Run: `cd gateway-server && ./gradlew.bat test`
Expected: PASS (RouteConfigTest, CorsTest, RequestLoggingFilterTest 모두 통과)

- [ ] **Step 6: 커밋 (gateway-server 자체 repo)**

```bash
git -C gateway-server add src/main/java/com/platform/gateway/filter/RequestLoggingFilter.java src/test/java/com/platform/gateway/filter/RequestLoggingFilterTest.java
git -C gateway-server commit -m "feat: 요청 로깅 + X-Request-Id 전파 GlobalFilter

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 7: infra — Keycloak redirect URI에 게이트웨이(:8000) 추가

**Files:**
- Modify: `infra/keycloak/realm-export.json`

**Interfaces:**
- Produces: `platform-bff` 클라이언트가 `http://localhost:8000/login/oauth2/code/keycloak`로의 리다이렉트를 허용

- [ ] **Step 1: 현재 redirectUris 확인**

Run: `grep -n "redirectUris\|9000" infra/keycloak/realm-export.json`
Expected: `platform-bff` 클라이언트의 `redirectUris` 배열에 `http://localhost:9000/login/oauth2/code/keycloak` 존재 확인

- [ ] **Step 2: :8000 redirect URI 추가**

`platform-bff` 클라이언트의 `redirectUris` 배열에 항목 추가(기존 `:9000`은 직접 호출 호환용으로 유지). 예:

```json
"redirectUris": [
  "http://localhost:9000/login/oauth2/code/keycloak",
  "http://localhost:8000/login/oauth2/code/keycloak"
],
```

(`webOrigins`에 `"+"`가 아니라 명시 출처가 있다면 `http://localhost:8000`도 추가.)

- [ ] **Step 3: realm 재초기화로 반영 확인**

Run:
```bash
cd infra/keycloak && docker compose down -v && docker compose up -d
# 기동 후
curl -s http://localhost:8080/realms/sso-demo/.well-known/openid-configuration | head -c 200
```
Expected: 200 응답. (realm은 빈 볼륨 최초 기동 때만 import되므로 `down -v` 필요.)

- [ ] **Step 4: 커밋**

```bash
git add infra/keycloak/realm-export.json
git commit -m "chore(infra): platform-bff redirect URI에 게이트웨이(:8000) 추가

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 8: myFront — base URL 통합 + board 경로 변경

**Files:**
- Modify: `myFront/.env`
- Modify: `myFront/src/app/board/boardStore.ts`
- (확인) `myFront/src/auth/authClient.ts`, `client.ts` — base만 게이트웨이로(코드는 `VITE_API_BASE` 사용 중이라 .env만 바꾸면 됨)

**Interfaces:**
- Consumes: 게이트웨이(:8000) 라우트, board 새 경로(`/api/board/posts`)
- Produces: 프론트가 게이트웨이 하나만 호출

- [ ] **Step 1: .env 통합**

`myFront/.env`:

```
VITE_API_BASE=http://localhost:8000
```

(`VITE_BOARD_API_BASE` 라인 삭제.)

- [ ] **Step 2: boardStore.ts의 BASE를 VITE_API_BASE로 변경**

`boardStore.ts`:

```ts
const BASE = (import.meta.env.VITE_API_BASE as string).replace(/\/+$/, '');
```

(상단 주석의 `board-service(:9100) REST 호출` 설명도 `게이트웨이(:8000) 경유` 취지로 갱신.)

- [ ] **Step 3: boardStore.ts의 호출 경로를 /api/board/posts로 변경**

`/api/posts`를 `/api/board/posts`로 치환(5곳: list/get/create/update/delete):

```ts
boardFetch(`/api/board/posts?page=${page}&size=${size}`)   // listPosts
boardFetch(`/api/board/posts/${id}`)                        // getPost
boardFetch('/api/board/posts', jsonInit('POST', input))    // createPost
boardFetch(`/api/board/posts/${id}`, jsonInit('PUT', input)) // updatePost
boardFetch(`/api/board/posts/${id}`, { method: 'DELETE' })  // deletePost
```

- [ ] **Step 4: 댓글 호출처 확인 및 변경**

Run: `grep -rn "/api/posts\|/api/comments\|VITE_BOARD_API_BASE" myFront/src`
Expected: 남은 `/api/posts`·`/api/comments`·`VITE_BOARD_API_BASE` 참조가 있으면 각각 `/api/board/...`·`VITE_API_BASE`로 변경. (없으면 이 단계 skip.)

- [ ] **Step 5: 프론트 타입체크/빌드**

Run: `cd myFront && npm run build`
Expected: 빌드 성공(타입 에러 없음).

- [ ] **Step 6: 엔드투엔드 수동 확인**

infra·auth-server·board-service·gateway-server·myFront 모두 기동 후:
1. 로그인 → `:8000/oauth2/authorization/keycloak` 경유 정상 로그인
2. 게시판 목록/글쓰기/수정/삭제가 `:8000/api/board/posts`로 동작
Expected: 모든 흐름이 게이트웨이(:8000)만 통해 동작.

- [ ] **Step 7: 커밋**

```bash
git add myFront/.env myFront/src/app/board/boardStore.ts
git commit -m "feat(myFront): API base를 게이트웨이(:8000)로 통합, board 경로 /api/board/**

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 9: 문서 갱신

**Files:**
- Create: `gateway-server/README.md`
- Modify: `auth-server/README.md` (CORS 이전 언급), `board-service/README.md`(있으면), `infra/README.md`(redirect URI), 루트 `README.md`(토폴로지)

**Interfaces:**
- Produces: 게이트웨이 경유 흐름·Docker DNS·확장 포인트 문서화

- [ ] **Step 1: gateway-server/README.md 작성**

역할(순수 라우팅 + CORS + 로깅, 토큰검증 X), 포트(:8000), 라우트 표, 환경변수(`AUTH_SERVER_URI`/`BOARD_SERVICE_URI`/`CORS_ALLOWED_ORIGIN`)와 Docker DNS 설명, 확장 포인트(Rate limiting/서킷브레이커/디스커버리/분산추적)를 정리. 설계 문서 `docs/superpowers/specs/2026-06-30-msa-gateway-design.md` 링크.

- [ ] **Step 2: 기타 README의 CORS·경로·진입점 언급 갱신**

- auth-server/board-service README: "CORS는 게이트웨이가 담당" 한 줄 + board 경로 `/api/board/**`.
- 루트 README: 토폴로지에 게이트웨이(:8000) 단일 진입점 반영.

- [ ] **Step 3: 커밋**

```bash
# 우산 repo (board-service README가 있으면 포함)
git add README.md infra/README.md
git commit -m "docs: 게이트웨이 도입 반영(토폴로지·CORS·경로·확장 포인트)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
# gateway-server README는 자체 repo
git -C gateway-server add README.md
git -C gateway-server commit -m "docs: 역할·라우트·환경변수·확장 포인트 정리

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
# auth-server README는 별도 repo
git -C auth-server add README.md
git -C auth-server commit -m "docs: CORS는 게이트웨이가 담당함을 명시

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Self-Review 메모

- **Spec 커버리지:** §1 토폴로지/라우팅→Task4, §2 게이트웨이 구성→Task4·5·6, §3.1 redirect→Task7, §3.3 myFront→Task8, §3.4 CORS→Task2·3·5, §4 board 리네임→Task1, §6 테스트→각 Task 내 TDD, §7 작업목록→Task1~9 전부 매핑됨.
- **확장 포인트(Rate limit/서킷브레이커/디스커버리)**: spec §0·§2대로 구현 제외, Task9 문서화로만 반영 — 의도된 비목표.
- **타입/이름 일관성:** 라우트 ID(`board`/`auth-*`)가 Task4 정의와 Task5·6에서 동일. board 경로 `/api/board/**`가 Task1·4·8에서 일관.
- **알려진 리스크(실행 중 조정 가능):** ① Spring Cloud Gateway WebFlux의 `RouteLocator`/`GlobalFilter`/`GatewayFilterChain` 정확한 패키지·시그니처는 2025.1.2에서 확인하며 import 정리(Task4 Step6, Task6 Step3에 주석). ② `application.yml`의 `spring.cloud.gateway.server.webflux` 경로 키가 2025.1.x 기준 — 부팅 안 되면 `spring.cloud.gateway.routes`(구 경로)로 폴백 확인. ③ §3.1 redirect-uri의 `{baseUrl}` 해석은 게이트웨이 뒤에서 `X-Forwarded-*`에 의존 — 로그인 리다이렉트가 :9000으로 튀면 auth-server에 `ForwardedHeaderFilter`/`server.forward-headers-strategy: framework` 추가(실행 중 보강).

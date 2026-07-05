# 게이트웨이 트래픽 제어 + 인증 조기차단 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 얇은 프록시였던 gateway-server(:8000)에 rate limiting·타임아웃·서킷브레이커·JWT 조기차단을 추가해 실질적인 API 게이트웨이로 만든다.

**Architecture:** (1) Redis 기반 `RequestRateLimiter`를 인증 경로에 적용, (2) 전역 httpclient 타임아웃, (3) board 라우트에 CircuitBreaker+Retry+fallback, (4) Spring Security reactive resource server로 보호 경로의 무토큰/위조 토큰을 게이트웨이에서 401 조기 차단(서비스 측 검증은 그대로 유지 — 이중 검증). CORS는 yml globalcors에서 Java `CorsConfigurationSource` 빈으로 이전해 **Security의 401 응답에도 CORS 헤더가 붙게** 한다(프론트 401→refresh 흐름 보존에 필수).

**Tech Stack:** Spring Cloud Gateway WebFlux(BOM 2025.1.2, Boot 4.0.6, JDK 24), Redis 7(rate limit), Resilience4j(reactor), spring-boot-starter-oauth2-resource-server(reactive).

## Global Constraints

- 저장소: `C:\MSA_TEMPLATE\gateway-server`(별도 repo, branch master) + `C:\MSA_TEMPLATE`(umbrella, infra). 문서는 Obsidian에만.
- 빌드/테스트: PowerShell에서 `$env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test` (기본 JDK 11이라 JAVA_HOME 필수).
- SCG 설정 prefix는 **`spring.cloud.gateway.server.webflux.*`** (Boot4/2025.x — 구버전 `spring.cloud.gateway.routes` 붙여넣으면 조용히 무시됨).
- 교차 계약 env(기존): `PLATFORM_ISSUER`(기본 `http://localhost:9000`), `PLATFORM_AUDIENCE`(기본 `platform-api`), `AUTH_JWKS_URI`(기본 `http://localhost:9000/.well-known/jwks.json`), `CORS_ALLOWED_ORIGIN`(기본 `http://localhost:5173`).
- dev 기본값으로 5서비스 로컬 기동이 계속 동작해야 함. Redis가 없으면 rate limiter는 fail-open(요청 통과)이므로 dev 기동 순서 강제 없음.
- 게이트웨이 검증이 서비스 검증을 **대체하지 않는다** — board-service/auth-server의 JWT 검증은 그대로(zero-trust 이중 검증).
- 커밋 footer: `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`

## File Structure (최종)

```
gateway-server/src/main/java/com/platform/gateway/
├─ GatewayApplication.java              (기존)
├─ filter/RequestLoggingFilter.java     (기존)
├─ config/
│  ├─ RateLimitConfig.java              (신규 — ipKeyResolver 빈)
│  ├─ SecurityConfig.java               (신규 — 보안체인+JWT디코더+CORS빈)
├─ security/AudienceValidator.java      (신규 — board-service와 동일 패턴)
└─ web/FallbackController.java          (신규 — /fallback/board 503)
gateway-server/src/main/resources/application.yml   (수정 — redis·타임아웃·필터·CORS 제거·platform.*)
infra/keycloak/docker-compose.yml                   (수정 — redis 서비스)
```

---

### Task 1: infra에 Redis 추가 (config-only, TDD 예외)

**Files:**
- Modify: `C:\MSA_TEMPLATE\infra\keycloak\docker-compose.yml`

**Interfaces:**
- Produces: `localhost:6379` Redis (Task 3의 rate limiter가 사용)

- [ ] **Step 1: compose에 redis 서비스 추가** — `volumes:` 블록 바로 앞에 삽입:

```yaml
  redis:
    image: redis:7-alpine
    container_name: platform-redis
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10
```

- [ ] **Step 2: 문법 검증**

Run: `cd C:\MSA_TEMPLATE\infra\keycloak; docker compose config --quiet`
Expected: 출력 없음(성공)

- [ ] **Step 3: Commit (umbrella repo)**

```
chore(infra): rate limiting용 Redis 추가 (gateway RequestRateLimiter 백엔드)
```

---

### Task 2: 전역 httpclient 타임아웃

**Files:**
- Modify: `gateway-server/src/main/resources/application.yml`
- Test: `gateway-server/src/test/java/com/platform/gateway/HttpClientTimeoutTest.java` (신규)

**Interfaces:**
- Produces: connect 3s / response 10s 전역 타임아웃 (Task 4 CB가 타임아웃 예외를 fallback으로 변환)

- [ ] **Step 1: 실패 테스트 작성**

```java
package com.platform.gateway;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.gateway.config.HttpClientProperties;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

/** 다운스트림 무한대기 방지 — 전역 connect/response 타임아웃이 바인딩되는지. */
@SpringBootTest
class HttpClientTimeoutTest {

    @Autowired HttpClientProperties httpClientProperties;

    @Test
    void globalTimeoutsAreConfigured() {
        assertThat(httpClientProperties.getConnectTimeout()).isEqualTo(3000);
        assertThat(httpClientProperties.getResponseTimeout()).isEqualTo(Duration.ofSeconds(10));
    }
}
```

- [ ] **Step 2: RED 확인** — `.\gradlew.bat test --tests '*HttpClientTimeoutTest'` → FAIL (기본값 null/45s)

- [ ] **Step 3: application.yml의 `webflux:` 아래(globalcors와 같은 레벨)에 추가**

```yaml
          httpclient:
            connect-timeout: 3000      # ms — 다운스트림 연결 3초
            response-timeout: 10s      # 응답 전체 10초 (무한대기 방지)
```

- [ ] **Step 4: GREEN 확인** — 같은 명령 → PASS
- [ ] **Step 5: Commit** — `feat(resilience): 전역 httpclient connect/response 타임아웃`

---

### Task 3: 인증 경로 rate limiting (Redis RequestRateLimiter)

**Files:**
- Modify: `gateway-server/build.gradle` (redis reactive 의존성)
- Create: `gateway-server/src/main/java/com/platform/gateway/config/RateLimitConfig.java`
- Modify: `gateway-server/src/main/resources/application.yml` (redis 접속 + auth 라우트 필터)
- Test: `gateway-server/src/test/java/com/platform/gateway/config/IpKeyResolverTest.java` (신규), `RouteConfigTest.java` (필터 존재 어서션 추가)

**Interfaces:**
- Produces: `@Bean KeyResolver ipKeyResolver` — yml에서 `key-resolver: "#{@ipKeyResolver}"`로 참조. 클라이언트 IP 기준 키, 주소 없으면 `"unknown"`.
- Redis 미기동 시 RedisRateLimiter는 에러 로그 후 **허용(fail-open)** — dev 흐름 안 깨짐.

- [ ] **Step 1: build.gradle dependencies에 추가**

```gradle
implementation 'org.springframework.boot:spring-boot-starter-data-redis-reactive'
```

- [ ] **Step 2: 실패 테스트 작성 — IpKeyResolverTest**

```java
package com.platform.gateway.config;

import org.junit.jupiter.api.Test;
import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.mock.http.server.reactive.MockServerHttpRequest;
import org.springframework.mock.web.server.MockServerWebExchange;

import java.net.InetSocketAddress;

import static org.assertj.core.api.Assertions.assertThat;

class IpKeyResolverTest {

    private final KeyResolver resolver = new RateLimitConfig().ipKeyResolver();

    @Test
    void resolvesClientIp() {
        MockServerWebExchange exchange = MockServerWebExchange.from(
                MockServerHttpRequest.get("/api/auth/refresh")
                        .remoteAddress(new InetSocketAddress("10.1.2.3", 51234)).build());
        assertThat(resolver.resolve(exchange).block()).isEqualTo("10.1.2.3");
    }

    @Test
    void fallsBackToUnknownWhenNoRemoteAddress() {
        MockServerWebExchange exchange = MockServerWebExchange.from(
                MockServerHttpRequest.get("/api/auth/refresh").build());
        assertThat(resolver.resolve(exchange).block()).isEqualTo("unknown");
    }
}
```

- [ ] **Step 3: RED 확인** (RateLimitConfig 없음 — 컴파일 실패가 RED)

- [ ] **Step 4: RateLimitConfig 구현**

```java
package com.platform.gateway.config;

import org.springframework.cloud.gateway.filter.ratelimit.KeyResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import reactor.core.publisher.Mono;

@Configuration
public class RateLimitConfig {

    /** 클라이언트 IP 기준 rate limit 키. 프록시 뒤에 게이트웨이를 또 두면 X-Forwarded-For 고려 필요. */
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
                exchange.getRequest().getRemoteAddress() != null
                        ? exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
                        : "unknown");
    }
}
```

- [ ] **Step 5: application.yml — redis 접속 + auth 라우트 3개에 필터**

`spring:` 아래에:
```yaml
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
```

`auth-api`(/api/auth/**) 라우트에 (refresh 브루트포스 방어 — 엄격):
```yaml
              filters:
                - name: RequestRateLimiter
                  args:
                    redis-rate-limiter.replenishRate: 5
                    redis-rate-limiter.burstCapacity: 10
                    key-resolver: "#{@ipKeyResolver}"
```

`auth-oauth2`(/oauth2/**)와 `auth-login`(/login/**) 라우트에 (로그인 리다이렉트 — 완화):
```yaml
              filters:
                - name: RequestRateLimiter
                  args:
                    redis-rate-limiter.replenishRate: 10
                    redis-rate-limiter.burstCapacity: 20
                    key-resolver: "#{@ipKeyResolver}"
```

- [ ] **Step 6: RouteConfigTest에 어서션 추가** (기존 6개 라우트 id 검증 유지 + 아래):

```java
@Test
void authRoutesHaveRateLimiterFilter() {
    Route authApi = routes().stream().filter(r -> r.getId().equals("auth-api")).findFirst().orElseThrow();
    assertThat(authApi.getFilters().toString()).contains("RequestRateLimiter");
}
```
(기존 테스트의 RouteLocator 수집 헬퍼를 재사용. 헬퍼가 없으면 `routeLocator.getRoutes().collectList().block()`.)

- [ ] **Step 7: GREEN 확인** — `.\gradlew.bat test` 전체 → PASS (Redis 없이도 컨텍스트 기동됨 — lazy 연결)
- [ ] **Step 8: Commit** — `feat(security): 인증 경로 IP 기준 rate limiting (Redis, fail-open)`

---

### Task 4: board 라우트 CircuitBreaker + Retry + fallback

**Files:**
- Modify: `gateway-server/build.gradle`
- Create: `gateway-server/src/main/java/com/platform/gateway/web/FallbackController.java`
- Modify: `gateway-server/src/main/resources/application.yml` (board 라우트 필터)
- Test: `gateway-server/src/test/java/com/platform/gateway/web/BoardFallbackTest.java` (신규)

**Interfaces:**
- Produces: `GET|POST... /fallback/board` → 503 `{"error":"board_unavailable"}`. Task 5 보안체인에서 `/fallback/**` permitAll 필요.

- [ ] **Step 1: build.gradle dependencies에 추가**

```gradle
implementation 'org.springframework.cloud:spring-cloud-starter-circuitbreaker-reactor-resilience4j'
```

- [ ] **Step 2: 실패 테스트 작성 — BoardFallbackTest**

```java
package com.platform.gateway.web;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.test.web.reactive.server.WebTestClient;

import java.time.Duration;

/** board-service 다운 시 무한대기/원시 5xx 대신 fallback 503 JSON. */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT,
        properties = {"BOARD_SERVICE_URI=http://localhost:59999"}) // 닫힌 포트 → connection refused
class BoardFallbackTest {

    @LocalServerPort int port;

    private WebTestClient client() {
        return WebTestClient.bindToServer()
                .baseUrl("http://localhost:" + port)
                .responseTimeout(Duration.ofSeconds(30))
                .build();
    }

    @Test
    void boardDownReturnsFallback503Json() {
        client().get().uri("/api/board/posts").exchange()
                .expectStatus().isEqualTo(503)
                .expectBody().jsonPath("$.error").isEqualTo("board_unavailable");
    }

    @Test
    void fallbackEndpointDirectly() {
        client().get().uri("/fallback/board").exchange()
                .expectStatus().isEqualTo(503)
                .expectBody().jsonPath("$.error").isEqualTo("board_unavailable");
    }
}
```

- [ ] **Step 3: RED 확인** — FAIL (fallback 없음: 직접 호출 404, 라우트는 500)

- [ ] **Step 4: FallbackController 구현**

```java
package com.platform.gateway.web;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

/** CircuitBreaker fallbackUri(forward:/fallback/board) 대상 — 다운스트림 불능 시 표준 503. */
@RestController
public class FallbackController {

    @RequestMapping("/fallback/board")
    public ResponseEntity<Map<String, String>> board() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                .body(Map.of("error", "board_unavailable"));
    }
}
```

- [ ] **Step 5: application.yml board 라우트에 필터 추가** (CB가 바깥, Retry가 안쪽):

```yaml
              filters:
                - name: CircuitBreaker
                  args:
                    name: board
                    fallbackUri: forward:/fallback/board
                - name: Retry
                  args:
                    retries: 2
                    methods: GET          # 멱등 GET만 — POST 재시도는 중복 생성 사고
                    series: SERVER_ERROR
```

- [ ] **Step 6: GREEN 확인** — `.\gradlew.bat test` 전체 → PASS
- [ ] **Step 7: Commit** — `feat(resilience): board 라우트 CircuitBreaker+GET Retry+fallback 503`

---

### Task 5: JWT 조기차단 (Security resource server) + CORS를 빈으로 이전

**Files:**
- Modify: `gateway-server/build.gradle`
- Create: `gateway-server/src/main/java/com/platform/gateway/security/AudienceValidator.java`
- Create: `gateway-server/src/main/java/com/platform/gateway/config/SecurityConfig.java`
- Modify: `gateway-server/src/main/resources/application.yml` (globalcors 삭제, platform.* 추가)
- Test: `gateway-server/src/test/java/com/platform/gateway/config/SecurityConfigTest.java` (신규), `security/AudienceValidatorTest.java` (신규, board와 동일), 기존 `CorsTest` 그대로 green이어야 함

**Interfaces:**
- Consumes: Task 4의 `/fallback/**` (permitAll 목록에 포함)
- Produces: 보호 경로 무토큰/위조토큰 → 게이트웨이가 401 (CORS 헤더 포함). 공개 경로(GET /api/board/posts/**, /oauth2/**, /login/**, /api/auth/**, /.well-known/**, /fallback/**, OPTIONS)는 통과.
- **경로 정책은 board-service SecurityConfig와 동기 유지** — board에 공개 경로를 추가하면 여기도 추가해야 함(안 하면 게이트웨이가 먼저 401). 코드 주석으로 명시.

- [ ] **Step 1: build.gradle dependencies에 추가**

```gradle
implementation 'org.springframework.boot:spring-boot-starter-security'
implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
```

- [ ] **Step 2: 실패 테스트 작성 — AudienceValidatorTest** (board-service 것과 동일 3케이스: 일치 성공/불일치 실패/부재 실패 — `Jwt.withTokenValue("t").header("alg","none").subject("1")` 빌더 사용)

- [ ] **Step 3: 실패 테스트 작성 — SecurityConfigTest**

```java
package com.platform.gateway.config;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.test.web.reactive.server.WebTestClient;

import java.time.Duration;

/**
 * 게이트웨이 JWT 조기차단. 보호 경로는 다운스트림 도달 전에 401이어야 하고(다운스트림 없이 테스트 가능),
 * 공개 경로는 보안을 통과해 프록시까지 가야 한다(다운스트림 없으니 401만 아니면 됨).
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class SecurityConfigTest {

    @LocalServerPort int port;

    private WebTestClient client() {
        return WebTestClient.bindToServer()
                .baseUrl("http://localhost:" + port)
                .responseTimeout(Duration.ofSeconds(30))
                .build();
    }

    @Test
    void protectedPathWithoutTokenIs401() {
        client().post().uri("/api/board/posts").exchange().expectStatus().isUnauthorized();
        client().get().uri("/api/me").exchange().expectStatus().isUnauthorized();
    }

    @Test
    void garbageBearerTokenIs401() {
        client().get().uri("/api/me")
                .header("Authorization", "Bearer not-a-jwt")
                .exchange().expectStatus().isUnauthorized();
    }

    @Test
    void unauthorizedResponseCarriesCorsHeaders() {
        // 401이라도 CORS 헤더가 있어야 브라우저의 fetch가 상태를 읽고 refresh 흐름을 탄다.
        client().post().uri("/api/board/posts")
                .header("Origin", "http://localhost:5173")
                .exchange()
                .expectStatus().isUnauthorized()
                .expectHeader().valueEquals("Access-Control-Allow-Origin", "http://localhost:5173");
    }

    @Test
    void publicPathsPassSecurity() {
        // 다운스트림이 없어 5xx가 나는 건 정상 — 게이트웨이 보안이 401로 막지만 않으면 된다.
        client().get().uri("/api/board/posts").exchange().expectStatus().value(s -> org.assertj.core.api.Assertions.assertThat(s).isNotEqualTo(401));
        client().post().uri("/api/auth/refresh").exchange().expectStatus().value(s -> org.assertj.core.api.Assertions.assertThat(s).isNotEqualTo(401));
        client().get().uri("/.well-known/jwks.json").exchange().expectStatus().value(s -> org.assertj.core.api.Assertions.assertThat(s).isNotEqualTo(401));
    }
}
```

- [ ] **Step 4: RED 확인** — 보호 경로 테스트 FAIL(보안 없음 → 401 아님)

- [ ] **Step 5: AudienceValidator 구현** (board-service `security/AudienceValidator.java`와 동일 — `OAuth2TokenValidator<Jwt>`, aud 미포함 시 `invalid_token` 실패. 게이트웨이 표준 패턴으로 클래스 사용.)

- [ ] **Step 6: SecurityConfig 구현**

```java
package com.platform.gateway.config;

import com.platform.gateway.security.AudienceValidator;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.reactive.EnableWebFluxSecurity;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.oauth2.core.DelegatingOAuth2TokenValidator;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.jwt.JwtValidators;
import org.springframework.security.oauth2.jwt.NimbusReactiveJwtDecoder;
import org.springframework.security.oauth2.jwt.ReactiveJwtDecoder;
import org.springframework.security.web.server.SecurityWebFilterChain;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.reactive.CorsConfigurationSource;
import org.springframework.web.cors.reactive.UrlBasedCorsConfigurationSource;

import java.util.List;

/**
 * JWT 조기차단(1차 방어). 다운스트림 서비스의 자체 검증(최종 방어)을 대체하지 않는다 — zero-trust 이중 검증.
 * 경로 정책은 board-service SecurityConfig와 동기 유지할 것(여기가 더 좁으면 게이트웨이가 먼저 401).
 */
@Configuration
@EnableWebFluxSecurity
public class SecurityConfig {

    @Bean
    SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http, CorsConfigurationSource corsSource) {
        http
                .csrf(ServerHttpSecurity.CsrfSpec::disable)
                .httpBasic(ServerHttpSecurity.HttpBasicSpec::disable)
                .formLogin(ServerHttpSecurity.FormLoginSpec::disable)
                // Security 레벨 CORS — 401 응답에도 CORS 헤더가 붙는다(프론트 401→refresh 흐름 필수).
                .cors(cors -> cors.configurationSource(corsSource))
                .authorizeExchange(auth -> auth
                        .pathMatchers(HttpMethod.OPTIONS).permitAll() // preflight
                        .pathMatchers("/oauth2/**", "/login/**", "/api/auth/**",
                                "/.well-known/**", "/fallback/**").permitAll()
                        .pathMatchers(HttpMethod.GET, "/api/board/posts/**").permitAll()
                        .anyExchange().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
        return http.build();
    }

    /** auth-server JWKS + issuer/audience 검증 — board-service 디코더와 동일 계약. */
    @Bean
    ReactiveJwtDecoder reactiveJwtDecoder(@Value("${platform.jwks-uri}") String jwksUri,
                                          @Value("${platform.issuer}") String issuer,
                                          @Value("${platform.audience}") String audience) {
        NimbusReactiveJwtDecoder decoder = NimbusReactiveJwtDecoder.withJwkSetUri(jwksUri).build();
        decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<Jwt>(
                JwtValidators.createDefaultWithIssuer(issuer),
                new AudienceValidator(audience)));
        return decoder;
    }

    /** CORS 단일 소스 — yml globalcors 대신 이 빈. credentials와 함께 쓰므로 오리진은 구체 값('*' 금지). */
    @Bean
    CorsConfigurationSource corsConfigurationSource(
            @Value("${platform.cors-allowed-origin}") String allowedOrigin) {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of(allowedOrigin));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

- [ ] **Step 7: application.yml 수정**
  - `globalcors:` 블록 전체 삭제 (CORS는 이제 SecurityConfig 빈이 단일 소스)
  - 최하단에 추가:

```yaml
platform:
  # auth-server 발급 자체 AT의 검증 계약 — auth/board와 동일 env 이름.
  jwks-uri: ${AUTH_JWKS_URI:http://localhost:9000/.well-known/jwks.json}
  issuer: ${PLATFORM_ISSUER:http://localhost:9000}
  audience: ${PLATFORM_AUDIENCE:platform-api}
  cors-allowed-origin: ${CORS_ALLOWED_ORIGIN:http://localhost:5173}
```

- [ ] **Step 8: GREEN 확인** — `.\gradlew.bat test` **전체** 실행. 신규 테스트 PASS + **기존 CorsTest가 그대로 PASS해야 함**(preflight가 이제 Security CorsWebFilter에서 응답 — 요청/응답 계약 동일). BoardFallbackTest(공개 GET)도 PASS 유지.
- [ ] **Step 9: Commit** — `feat(security): JWT 조기차단(JWKS iss/aud) + CORS를 Security 빈으로 단일화`

---

### Task 6: 문서/원장 + 최종 리뷰

**Files:**
- Modify: `gateway-server/README.md` (역할 목록 갱신: rate limit·타임아웃·CB·JWT 조기차단, "토큰 검증 없음" 문구 수정)
- Modify: `C:\MSA_TEMPLATE\.superpowers\sdd\progress.md` (세션 원장)
- Modify: Obsidian `MSA_TEMPLATE 정리/07 보안 감사 + 하드닝 (2026-07-02).md` 로드맵 체크 + `05 API 게이트웨이 설계 (미래).md`에 구현 반영 노트

- [ ] **Step 1: gateway README 갱신** — "무엇을 하나" 섹션에 4개 역할 추가, 이중 검증 원칙(게이트웨이=1차, 서비스=최종) 명시, Redis 선택적(fail-open) 명시
- [ ] **Step 2: 원장에 태스크별 커밋 SHA 기록**
- [ ] **Step 3: 전체 테스트 최종 실행** (`.\gradlew.bat test`) → 전부 PASS 확인
- [ ] **Step 4: whole-diff 코드리뷰 요청** (requesting-code-review 스킬, Critical/Important 즉시 수정)
- [ ] **Step 5: Commit** — `docs: 게이트웨이 역할 갱신(rate limit·타임아웃·CB·JWT 조기차단)` (README), umbrella에 원장 커밋

---

## 수동 검증 (자동화 제외 — 사용자)

- `docker compose up -d` (redis 포함) 후 5서비스 기동 → 브라우저 로그인 E2E 재확인.
- rate limit 실동작: `for i in $(seq 1 15); do curl -s -o /dev/null -w '%{http_code}\n' -X POST localhost:8000/api/auth/refresh; done` → 뒤쪽 요청 429.
- 유효 토큰 E2E: 로그인 후 AT로 `POST :8000/api/board/posts` → 201 (게이트웨이 통과 + board 검증 통과).

## Self-Review 결과

- 커버리지: rate limiting(T3)·타임아웃(T2)·CB/Retry(T4)·JWT 조기차단+CORS 이전(T5)·인프라(T1)·문서(T6) — 대화에서 합의한 범위 전부 매핑됨.
- 타입/이름 일관성: `ipKeyResolver` 빈명 = yml SpEL 참조, `/fallback/board` = fallbackUri = permitAll 패턴, env 4종 = 기존 계약과 동일.
- 리스크 명시: CORS 이전 시 CorsTest green 게이트(T5 Step 8), Security가 GlobalFilter보다 먼저라 401 요청은 RequestLoggingFilter 로그에 안 남음(수용), Redis fail-open.

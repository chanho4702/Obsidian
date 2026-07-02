# MSA 게이트웨이 도입 & 전체 구조 설계

- 작성일: 2026-06-30
- 대상: `MSA_TEMPLATE` (재사용 MSA 레퍼런스 템플릿)
- 목적: 단일 진입점 **API 게이트웨이**를 도입해 프론트–서비스 토폴로지를 정리하고,
  "서비스를 하나 더 추가할 때 무엇을 복제하면 되는지"가 코드로 드러나는 표준 패턴을 확립한다.

---

## 0. 핵심 원칙 (이 설계를 관통하는 결정)

| # | 결정 | 근거 |
|---|---|---|
| 1 | **재사용 템플릿**이 목적 → 단순·표준 패턴·문서화 우선, YAGNI 강하게 | repo 이름 `MSA_TEMPLATE` |
| 2 | **토큰 검증은 각 서비스(resource-server)가 JWKS로 자체 수행.** 게이트웨이는 검증하지 않음 | 분산 검증. board-service에 이미 배선됨 |
| 3 | **게이트웨이 = 순수 라우팅 + 공통 관심사(CORS·로깅)**. Spring Security 미탑재 | 책임 최소화 |
| 4 | 게이트웨이 스택 = **Spring Cloud Gateway (WebFlux)** | auth-server와 같은 Spring 스택, 템플릿 일관성 |
| 5 | 서비스 위치 해소 = **정적 설정 + Docker DNS** (디스커버리 서버 없음) | 추가 인프라 0, 템플릿에 충분 |
| 6 | 라우팅 규칙 = **서비스명 prefix (`/api/board/**`) + No StripPrefix** | 경로가 곧 서비스 지도, 디버깅 직관적 |
| 7 | Rate limiting·서킷브레이커·디스커버리·분산추적 백엔드 = **확장 포인트로 문서화만**, 구현 X | YAGNI |

---

## 1. 토폴로지

```
  myFront(:5173) ──▶ gateway-server(:8000) ──┬──▶ auth-server(:9000)   JWT발급+JWKS+/api/me
   (base URL 1개)    Spring Cloud Gateway     │
                     라우팅 / CORS / 로깅      └──▶ board-service(:9100) JWT검증(JWKS), /api/board/**
                     (토큰검증 X)
                                              infra: Keycloak(:8080) · Postgres(:5433)
```

### 포트 맵

| 컴포넌트 | 포트 | 비고 |
|---|---|---|
| myFront (React/Vite) | 5173 | 게이트웨이 주소 하나만 바라봄 |
| **gateway-server** | **8000** | 신규. 8080/9000/9100/5173/5433이 차 있어 비는 자리 |
| auth-server | 9000 | 기존 |
| board-service | 9100 | 기존 (컨트롤러·시큐리티 미완성) |
| Keycloak | 8080 | infra |
| PostgreSQL | 5433 | DB: `keycloak` / `authdb` / `boarddb` |

### 라우팅 규칙 (게이트웨이)

| 경로 패턴 | 라우팅 대상 | 경로 변형 |
|---|---|---|
| `/oauth2/**`, `/login/**` | auth-server :9000 | 없음 (OIDC 리다이렉트 흐름) |
| `/api/auth/**` | auth-server :9000 | 없음 (refresh / logout) |
| `/.well-known/**` | auth-server :9000 | 없음 (JWKS 공개키) |
| `/api/me` | auth-server :9000 | 없음 |
| `/api/board/**` | board-service :9100 | **없음 (No StripPrefix)** |
| (미래) `/api/order/**` | order-service | 라우트 한 줄 추가로 복제 |

> **No StripPrefix**: 클라이언트가 보낸 경로 = 서비스가 받는 경로. board-service 컨트롤러는
> `@RequestMapping("/api/board/posts")`로 매핑한다. 로그·디버깅이 직관적이고 학습자 혼란이 없다.

> **⚠️ 현재 상태 정정 (2026-06-30 재확인):** board-service는 골격이 아니라 **이미 완성·커밋된
> 상태**다(`PostController`/`CommentController`/`SecurityConfig`/`AccessGuard`/comment 패키지/테스트
> 풀세트). 단 현재 경로가 **`/api/posts/**`, `/api/posts/{id}/comments`, `/api/comments/**`**로,
> 위 `/api/board/**` 규칙과 다르다. 따라서 이 설계의 board 작업은 "신규 구현"이 아니라
> **기존 경로를 `/api/board/**`로 리네임**하는 것이다(섹션 4 참조).

---

## 2. 게이트웨이 구성 & 모듈 구조

신규 모듈 `gateway-server` — auth-server와 동일하게 Spring Boot 4.0.6 / Java 24 / Gradle.

```
gateway-server/
├─ build.gradle                     // spring-cloud-starter-gateway-server-webflux
├─ src/main/java/com/platform/gateway/
│  ├─ GatewayApplication.java
│  ├─ config/
│  │  └─ CorsConfig.java            // CORS 중앙화 (auth-server에서 이전)
│  └─ filter/
│     └─ RequestLoggingFilter.java  // GlobalFilter: X-Request-Id 생성/전파 + 요청 1줄 로깅
└─ src/main/resources/application.yml // 라우트 정의(정적) + 다운스트림 URI(환경변수 외부화)
```

설계 포인트:

- **라우트는 `application.yml`에 선언** (자바 DSL보다 템플릿 학습자에게 한눈에 들어옴).
- **다운스트림 URI는 환경변수로 외부화**한다. 같은 코드로 두 환경을 커버:
  - 로컬 실행: `AUTH_SERVER_URI=http://localhost:9000`, `BOARD_SERVICE_URI=http://localhost:9100`
  - Docker compose: `http://auth-server:9000`, `http://board-service:9100`
    (compose 네트워크에서 **서비스명이 곧 DNS 호스트명**이 된다 — 컨테이너끼리 서비스명으로 서로를 찾음)
- **게이트웨이엔 Spring Security를 넣지 않는다** (토큰 검증 안 함). 의존성 최소.

### 공통 관심사 (게이트웨이가 맡는 것)

1. **CORS 중앙화** — 단일 진입점이므로 CORS는 게이트웨이 한 곳에서만 처리.
   `allowedOrigin = http://localhost:5173`, `allowCredentials = true` (쿠키 전송).
2. **요청 로깅 + traceId 전파** — `GlobalFilter`에서 요청마다 `X-Request-Id`(없으면 생성)를
   다운스트림으로 전파하고, `메서드 경로 → 대상서비스 (requestId)` 한 줄을 로깅.
   분산 추적의 기본기. Zipkin/Tempo 같은 백엔드 연동은 확장 포인트.

### 확장 포인트 (위치만 표시, 구현 X)

- **Rate limiting**: `RequestRateLimiter` 필터 + Redis. → 라우트 filters에 추가하는 자리 주석.
- **서킷 브레이커**: Resilience4j `CircuitBreaker` 필터 + fallback 라우트.
- **서비스 디스커버리**: Eureka 도입 시 정적 URI를 `lb://service-name`으로 교체.
- **분산 추적 백엔드**: Micrometer Tracing + Zipkin/Tempo exporter.

---

## 3. 로그인 흐름 & 타 모듈 설정 변경

게이트웨이를 단일 진입점으로 세우면서 **로그인(OIDC 리다이렉트) 트래픽도 게이트웨이를 통과**시킨다.
프론트는 게이트웨이 주소 하나만 알면 된다. 이에 따라 3곳을 맞춘다.

### 3.1 Keycloak redirect-uri

- 현재 auth-server는 `redirect-uri: {baseUrl}/login/oauth2/code/keycloak` → 직접 호출 시 `:9000` 기준.
- 로그인 트래픽이 게이트웨이(`:8000`)를 통과하므로 **외부에 노출되는 redirect-uri가 `:8000` 기준**이 되도록 한다.
- `infra/keycloak/realm-export.json`의 `platform-bff` 클라이언트 **Valid Redirect URIs**에
  `http://localhost:8000/login/oauth2/code/keycloak`를 추가한다 (기존 `:9000`은 직접 호출 호환용으로 남겨도 됨).
- 주의: realm은 **빈 볼륨 최초 기동 때만** import된다. 반영하려면 `docker compose down -v && up -d`.

### 3.2 Refresh 쿠키 Path

- 현재 쿠키: `refresh_token`, `Path=/api/auth`, `HttpOnly`, `SameSite=Lax`, `Secure=false(dev)`.
- 게이트웨이가 `/api/auth/**`를 **경로 변형 없이** 통과시키므로 **Path는 그대로 유효**.
- dev 환경은 게이트웨이·프론트가 모두 `localhost`라 쿠키 도메인 이슈 없음.
  (운영에서 도메인이 갈리면 쿠키 도메인/Secure/SameSite 재검토 — 확장 포인트.)

### 3.3 myFront

- `.env`: `VITE_API_BASE` / `VITE_BOARD_API_BASE` 두 개 → **`VITE_API_BASE=http://localhost:8000` 하나로 통합**.
  `VITE_BOARD_API_BASE`는 제거하고 `boardStore.ts`의 `BASE`도 `VITE_API_BASE`를 쓰도록 변경.
- board 호출 경로: `boardStore.ts`의 `/api/posts/**` → **`/api/board/posts/**`** (목록/상세/생성/수정/삭제 5곳).
  댓글 호출이 별도 store에 있으면 동일하게 `/api/board/...`로 변경.
- auth 호출 경로(`/api/auth/refresh`, `/api/me`, `/api/auth/logout`)는 그대로, base만 게이트웨이로.

### 3.4 CORS 일원화

- **board-service**와 **auth-server** 둘 다 현재 자체 CORS를 가짐:
  - board: `SecurityConfig`의 `corsConfigurationSource` 빈 + `.cors()` + `allowedOrigin` 필드 + `CorsConfigTest`.
  - auth: `SecurityConfig`의 `corsConfigurationSource` 빈 + 두 체인의 `.cors()` + `allowedOrigin` 필드 (CORS 테스트 없음).
- 위를 **모두 제거**하고 CORS는 게이트웨이 한 곳으로 이전. `CorsConfigTest`는 삭제.
- 다운스트림(auth/board)은 **내부망 전제** — 외부 브라우저의 직접 호출은 게이트웨이를 통해서만.
  단일 진입점 원칙과 일치.

---

## 4. board-service 경로 리네임 (`/api/posts` → `/api/board`)

board-service는 **이미 완성**돼 있다(컨트롤러·시큐리티·comment·마이그레이션·테스트 모두 존재).
이 설계에서 할 일은 신규 구현이 아니라 **경로를 서비스명 prefix 규칙으로 리네임**하는 것뿐이다.

| 파일 | 현재 | 변경 후 |
|---|---|---|
| `post/PostController.java` | `@RequestMapping("/api/posts")` | `@RequestMapping("/api/board/posts")` |
| `comment/CommentController.java` | `@RequestMapping("/api")` (`/posts/{id}/comments`, `/comments/{id}`) | `@RequestMapping("/api/board")` |
| `config/SecurityConfig.java` | `requestMatchers(GET, "/api/posts/**")` | `requestMatchers(GET, "/api/board/posts/**")` |
| `post/PostControllerTest.java` | `/api/posts...` | `/api/board/posts...` |
| `comment/CommentControllerTest.java` | `/api/posts.../comments`, `/api/comments...` | `/api/board/posts.../comments`, `/api/board/comments...` |

> 리네임 후 board-service 자체 테스트가 그대로 통과해야 한다(경로 외 로직 불변).
> CORS 빈/`.cors()`/`CorsConfigTest`는 §3.4에 따라 제거된다(리네임과 별개 단계).

---

## 5. 에러처리

- **게이트웨이**: 다운스트림 다운 → 기본 503 통과 + `X-Request-Id` 로깅.
  (서킷브레이커 fallback은 확장 포인트.)
- **서비스**: JWT 만료/무효 → 401 (resource-server 기본 동작). 도메인 예외 → 기존 `NotFoundException` 패턴 유지(404).

---

## 6. 테스트

- **gateway-server**: 라우트 매핑 검증 위주의 가벼운 테스트 — "어떤 경로가 어느 서비스로 가는가".
- **board-service**: auth-server 테스트 패턴 복제 — H2 인메모리 + Hibernate 자동 DDL,
  오프라인 JWT 디코더로 Keycloak 없이 `@SpringBootTest` 컨텍스트 기동.

---

## 7. 구현 작업 목록 (요약)

1. `gateway-server` 모듈 신규 — Gateway 의존성, 라우트 yml(`/api/board/**`→board, 나머지→auth),
   CORS 중앙화, 요청 로깅/traceId 필터.
2. board-service — `/api/posts`·`/api/comments` → `/api/board/**` 리네임(컨트롤러 2 + 테스트 2),
   CORS 빈/`.cors()`/`CorsConfigTest` 제거.
3. auth-server — CORS 빈/`.cors()`/`allowedOrigin` 제거.
4. infra — `realm-export.json` `platform-bff` Valid Redirect URIs에 `:8000` 추가.
5. myFront — `.env` base URL 1개로 통합, `boardStore.ts` 경로 `/api/board/posts`로 변경.
6. 문서 — 각 모듈 README에 게이트웨이 경유 흐름·Docker DNS·확장 포인트 정리.

---

## 8. 비목표 (이번 설계에서 다루지 않음)

- 운영 배포(K8s), 시크릿 매니저 주입, JWK 키 회전(kid 다중키).
- Rate limiting / 서킷브레이커 / 서비스 디스커버리 / 분산추적 백엔드 **구현** (위치만 표시).
- 게이트웨이에서의 토큰 검증 (분산 검증 원칙상 의도적으로 제외).

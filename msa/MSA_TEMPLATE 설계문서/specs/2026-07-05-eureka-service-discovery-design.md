# Eureka 서비스 디스커버리 도입 — 설계

날짜: 2026-07-05
상태: 승인됨

## 배경 / 문제의식

게이트웨이 라우트가 전부 `${BOARD_SERVICE_URI:http://localhost:9100}` 식의 고정 URI(환경변수)다.
서비스 주소·포트가 바뀌거나 인스턴스가 늘면 게이트웨이 설정을 고치고 재기동해야 하며,
인스턴스 2대 이상 스케일아웃 시 분산 수단이 없다. MSA 템플릿으로서 표준 구성 요소인
서비스 디스커버리를 채워 넣는다.

## 목표

1. Eureka 서버를 독립 서비스로 신설 (단일 노드, 포트 8761)
2. auth-server·board-service가 유레카에 자기 등록
3. 게이트웨이 라우트 6개를 전부 `lb://<service-name>`으로 전환 (클라이언트 사이드 로드밸런싱)

## 비목표 (YAGNI)

- Eureka HA(peer 복제) 구성 — 단일 노드로 충분. 운영 전환 시점에 재검토
- `platform.jwks-uri`의 load-balanced WebClient 전환 — JwtDecoder의 HTTP 호출은
  게이트웨이 라우팅을 타지 않으므로 직접 URI 유지가 단순하고 충분
- Kubernetes 배포 대응 — K8s로 가면 유레카 대신 K8s Service DNS가 이 역할을 대체함을 문서에 명시

## 설계

### 1. eureka-server 신설 (`C:\MSA_TEMPLATE\eureka-server`)

기존 서비스와 동일 패턴의 독립 Gradle 프로젝트.

- Spring Boot 4.0.6 + Spring Cloud BOM 2025.1.2 + Java 24 toolchain (gateway-server와 동일)
- 의존성: `spring-cloud-starter-netflix-eureka-server` 단일
- `@EnableEurekaServer` 메인 클래스
- application.yml:
  - `server.port: 8761`
  - `eureka.client.register-with-eureka: false` / `fetch-registry: false`
    (standalone — 자기 자신을 레지스트리에 등록하지 않고, 조회할 다른 노드도 없음)

### 2. auth-server / board-service 등록

두 프로젝트 각각:

- build.gradle: Spring Cloud BOM(2025.1.2) import + `spring-cloud-starter-netflix-eureka-client`
- application.yml:
  - `spring.application.name`: `auth-server` / `board-service`
    — **게이트웨이 lb:// 서비스 이름과 반드시 일치** (유레카 등록 ID가 됨)
  - `eureka.client.service-url.defaultZone: ${EUREKA_URI:http://localhost:8761/eureka}`

### 3. gateway-server lb:// 전환

- build.gradle: `spring-cloud-starter-netflix-eureka-client` 추가
  (spring-cloud-starter-loadbalancer 전이 포함 — `lb://` 스킴 처리기)
- application.yml 라우트 6개의 `uri:` 기본값 교체 (환경변수는 유지):
  - `board` → `${BOARD_SERVICE_URI:lb://board-service}`
  - `auth-*` 5개 → `${AUTH_SERVER_URI:lb://auth-server}`
  - 환경변수를 남기는 이유: 기존 테스트(BoardFallbackTest, SlowBoardDownstreamTest)가
    `BOARD_SERVICE_URI`로 로컬 스텁 주소를 주입하는 시임(seam)으로 사용 중이고,
    운영에서도 유레카 없이 직접 URI로 돌릴 탈출구가 됨
- 게이트웨이 자신도 유레카에 등록 (기본 동작 유지) — 대시보드에서 전체 토폴로지 가시성
- `platform.jwks-uri` / `platform.issuer`는 직접 URI 그대로 유지

### 4. 장애 내성 — 기존 resilience 체계와의 합성

- 유레카 클라이언트는 레지스트리를 **로컬 캐시**하고 30초 주기로 갱신
  → 유레카 서버가 잠깐 죽어도 마지막 스냅샷으로 라우팅 지속 (유레카는 SPOF가 아님)
- 역할 분담: 유레카 = "죽은 인스턴스를 목록에서 제거"(하트비트 기반),
  CircuitBreaker = "목록엔 있지만 응답이 이상한 인스턴스 차단" — 상호보완
- 서비스가 아직 등록 안 된 시점의 요청 → `lb://` 해석 실패 503
  → board 라우트는 기존 `/fallback/board`가 받음. 기동 순서 강제 없음

## 검증 계획

1. 4개 프로젝트 각각 `gradlew build` — 기존 테스트 전부 green 유지
   (게이트웨이 기존 테스트는 유레카 없이 돌아가야 함 — 필요 시 테스트 프로파일에서
   `eureka.client.enabled: false`)
2. 수동 E2E: eureka-server → auth/board → gateway 기동
   - `http://localhost:8761` 대시보드에 GATEWAY-SERVER, AUTH-SERVER, BOARD-SERVICE 등록 확인
   - 게이트웨이 경유 curl: `GET :8000/api/board/posts`(lb://board-service),
     `GET :8000/.well-known/jwks.json`(lb://auth-server) 정상 응답
3. board-service를 내리고 30초 대기 후 → 게이트웨이 요청이 fallback 503으로 수렴하는지 확인

## 영향 범위

| 저장소 | 변경 |
|---|---|
| eureka-server (신규) | 프로젝트 전체 신설 |
| auth-server | build.gradle + application.yml (의존성·등록 설정) |
| board-service | build.gradle + application.yml (의존성·등록 설정) |
| gateway-server | build.gradle + application.yml (의존성·라우트 uri 6개) |
| MSA_TEMPLATE 루트 | README·docs 갱신 |

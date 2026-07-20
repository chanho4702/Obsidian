---
tags: [msa, template, gateway, ratelimit, x-forwarded-for, logging, observability, loki, alloy, grafana, 구현기록]
작성일: 2026-07-20
상태: 완료 · 배포·E2E 검증 완료(2026-07-20) — gateway 652aa4a · infra 4b0197e/164c10d
---

# 19 — 게이트웨이 평가 + rate-limit XFF 수정 + Loki 로그 관측 스택 (2026-07-20)

상위: [[00 개요 — 전체 구조]] · 선행: [[08 게이트웨이 트래픽 제어 + JWT 조기차단 (2026-07-03)]], [[10 게이트웨이 동작 흐름 코드 워크스루 (2026-07-05)]], [[11 nginx 통합배포 + 3앱 SSO (2026-07-14)]], [[13 백엔드 컨테이너화 + compose 통합 — CI-CD Wave 1 (2026-07-15)]]

## 무엇을 했나

"게이트웨이 평가"에서 출발 → 코드 리뷰로 **실질 버그 1건**을 찾고, "로깅을 여기서 하나?"라는 질문이 이어져 **access log 반쪽(요청만·결과 없음)** 을 발견, 마무리로 **Loki 로그 관측 스택**을 세웠다. 세 갈래를 한 번에 반영.

## 1. 평가 요약 — A− (프로덕션 근접)

잘 돼 있던 것(유지): 타임아웃 예산 순서(timelimiter 11s > response 10s > connect 3s), GET 한정·IOException 재시도, CORS를 Security 레벨에 둬 401에도 CORS 헤더, `X-Request-Id` 로그 인젝션 방지(SAFE_ID), rate limit 차등(login 10/s vs refresh 5/s), zero-trust 이중검증 + AudienceValidator(confused deputy 차단). → 대부분 [[08 게이트웨이 트래픽 제어 + JWT 조기차단 (2026-07-03)]]에서 이미 확립.

## 2. 이슈 #1 (실질 버그) — nginx 뒤에서 rate-limit 키가 무력화

### 근본 원인

`RateLimitConfig.ipKeyResolver`가 `exchange.getRequest().getRemoteAddress()`를 그대로 썼다. 그런데 통합배포는 **browser → nginx → gateway**([[11 nginx 통합배포 + 3앱 SSO (2026-07-14)]]) 구조라, 이 값은 항상 **nginx/도커 NAT IP 하나로 고정**된다. 결과:

- 전 클라이언트가 **단일 토큰 버킷을 공유** → 정상 사용자끼리 상호 차단(false positive)
- 공격자 IP를 격리 못 함 → **auth 경로 브루트포스 방어가 사실상 무력**

`RateLimitConfig`의 원래 주석("앞에 프록시를 두면 X-Forwarded-For 고려")이 이 상황을 예견했으나 미반영 상태였다.

### 수정 — 신뢰 홉 1개로 XFF 실 IP 채택

```java
private final XForwardedRemoteAddressResolver clientIpResolver =
        XForwardedRemoteAddressResolver.maxTrustedIndex(1); // nginx 1홉
```

- nginx가 `X-Forwarded-For $proxy_add_x_forwarded_for`로 실 클라이언트 IP를 **맨 오른쪽**에 append. `maxTrustedIndex(1)`은 오른쪽에서 1번째(nginx가 붙인 값)를 채택.
- **위조 방어**: 클라이언트가 XFF를 위조해도 맨 앞에만 들어가고, 신뢰 홉이 1개라 nginx append 값만 씀 → 위조 무력. 테스트 `ignoresSpoofedXForwardedForBeyondTrustedHop`으로 고정.
- gateway `trusted-proxies`(application.yml, 도커 대역 신뢰)가 XFF를 온전히 통과시키므로 성립. XFF 부재(직결 스모크)면 remoteAddress로 폴백 → `unknown` 케이스 보존.
- 패키지 확인(SCG 5.x): `org.springframework.cloud.gateway.support.ipresolver.XForwardedRemoteAddressResolver` (jar `tf`로 실측).

## 3. 로깅 — access log 반쪽을 채우다

### 진단: 요청만 찍고 결과를 버렸다

기존 `RequestLoggingFilter`는 진입 시 `METHOD PATH (id)` 한 줄만. **status·latency를 안 남겨** 401/429/503/504가 어떤 요청이었는지 로그로 못 이었다. 게이트웨이는 유일 진입점 = **access log의 주인**인데 절반만 하고 있었다.

### 수정: 완료 라인 추가 + 구조화

- `chain.filter(...).doFinally(...)`로 **완료 시 status·latencyMs·requestId** 한 줄 추가(취소/에러에도 남음).
- SLF4J fluent API `log.atInfo().addKeyValue("status", ...)...` — 이 key-value가 JSON 최상위 필드로 실려 Loki에서 질의 가능.

### 구조화 출력(ECS JSON) — 의존성 0

- `application-docker.yml` 신규: `logging.structured.format.console: ecs` — **Spring Boot 4 네이티브** 구조화 로깅. 라이브러리 추가 없음.
- compose gateway에 `SPRING_PROFILES_ACTIVE: docker` 추가. **로컬 dev(IntelliJ, 프로필 없음)는 평문 유지** — dev 편의와 운영 수집을 분리([[dev-env-ide-split]] 원칙과 일치).

## 4. Loki 로그 관측 스택

### 왜 "방법 A"(앱 디커플)인가

두 갈림길 중 **stdout JSON + Alloy scrape**를 택함:

```
각 컨테이너 stdout ─▶ Alloy(도커 소켓 scrape) ─▶ Loki(저장/검색) ─▶ Grafana(조회 UI)
```

- 앱은 **Loki를 모른다**. 새 서비스는 JSON만 뱉으면 Alloy가 자동 수집 → `MSA_TEMPLATE`처럼 서비스가 계속 느는 곳에 최적(대안 loki4j 직접 push는 서비스마다 appender+URL 필요).
- gateway가 첫 적용, 이후 auth/board/org에 **복제할 패턴**.

### 구성

- compose에 3서비스 추가: `loki`(grafana/loki:3.1.1, 기본 monolithic·볼륨 platform-loki) / `alloy`(grafana/alloy:v1.5.1, 도커 소켓 ro 마운트) / `grafana`(11.3.1, :3000, 익명 Admin·Loki 데이터소스 provisioning). 컨테이너 **8 → 12개**.
- 설정 파일 `infra/observability/`: `config.alloy`(발견→라벨 정리→Loki push), `grafana/provisioning/datasources/loki.yml`.
- **라벨은 저 카디널리티만**(job·container·service). requestId/status는 라벨이 아니라 로그라인 JSON에 두고 쿼리시 `| json` 파싱(Loki 라벨 폭발 방지 — 정석).

### 조회 (http://localhost:3000, 로그인 없음)

```logql
{service="gateway"} | json | status=`503`     # 503 응답만
{service="gateway"} | json | latencyMs > 1000  # 1초 초과 느린 요청
{job="docker"} | json | requestId=`<id>`       # 한 요청을 전 서비스에 걸쳐 추적
```

## 변경 파일

**gateway-server**
- `config/RateLimitConfig.java` — XForwardedRemoteAddressResolver(maxTrustedIndex 1)
- `filter/RequestLoggingFilter.java` — doFinally 완료 로깅 + addKeyValue 구조화
- `resources/application-docker.yml` — 신규, ECS 구조화 로깅
- `test/.../IpKeyResolverTest.java` — XFF 추출·위조 차단 케이스 2개 추가(총 4 green)

**infra**
- `keycloak/docker-compose.yml` — gateway `SPRING_PROFILES_ACTIVE: docker` + loki·alloy·grafana 3서비스 + 볼륨 2개
- `observability/config.alloy`, `observability/grafana/provisioning/datasources/loki.yml` — 신규
- `README.md` — 컨테이너 표(+org·+관측 3), 관측 스택 섹션, 트러블슈팅 2건

## 남은 것 (평가에서 나온 후속)

- **org-service·auth-me·jwks resilience 무방비** — board는 CB+Retry+fallback 3단인데 나머지는 없다. 다운 시 response-timeout 10s 매달림. board 패턴을 org에 대칭 복제 권장.
- **Redis 다운 = rate limit 침묵 해제**(fail-open) — 브루트포스 방어가 조용히 사라짐. actuator health + 경보 필요.
- **Retry 주석 정정** — `series: SERVER_ERROR`는 IOException뿐 아니라 5xx GET도 재시도한다("연결 실패만"은 부정확).
- **requestId를 전 로그 라인 MDC로**(reactor context 전파) — 현재는 access log 2줄에만 실림. gateway 내부 에러 로그까지 상관관계 걸려면 필요.

## 배포 — CI/CD와 부딪힌 함정 2개 (실측 기록)

push 후 실제 배포에서 예상 못 한 두 지점을 만났다. 둘 다 [[13 백엔드 컨테이너화 + compose 통합 — CI-CD Wave 1 (2026-07-15)]]의 파이프라인 특성에서 비롯.

**함정 ① deploy 워크플로는 서비스 단위 → 신규 obs 3서비스가 자동으로 안 뜬다.**
`deploy.yml`은 `docker compose ... up -d --no-deps $svc`로 **해당 서비스 하나만** pull+재기동한다(dispatch payload의 service). gateway push는 gateway만 갱신 → compose에 새로 추가한 loki·alloy·grafana는 **CI 배포로 뜨지 않음**. 최초 1회는 수동:
```bash
cd infra/keycloak && docker compose up -d loki alloy grafana   # 프로젝트 platform에 합류
```
프로젝트명이 `platform` 고정이라 기존 스택 네트워크에 그대로 붙고, Alloy는 도커 소켓 tail이라 **이미 돌던 컨테이너 로그까지 소급 수집**한다. → README(infra) "배포·기동 주의"에 영구 기록.

**함정 ② self-hosted 러너가 offline이면 deploy가 queued로 멈춘다.**
gateway CI(빌드→GHCR→dispatch)는 success인데 infra-settings deploy가 8분째 queued. 원인은 로컬 러너 `DESKTOP-4TUOSLF` offline(서비스가 아니라 `run.cmd` 수동 실행). 러너 기동 후 deploy success(59s, smoke 통과). 러너는 config 완료 상태라 `run.cmd`(또는 큐 1개만 소화할 땐 `--once`)로 켠다.

## 검증 (E2E 완료)

- `gateway-server` 전체 테스트 green(JDK24). IpKeyResolverTest 4/4(XFF 추출·위조 차단 포함).
- `docker compose config` OK. 배포 후 **컨테이너 12개 Up**(기존 9 + obs 3).
- **Loki 수집**: `service` 라벨에 전 12서비스(gateway·auth·board·org·nginx·keycloak·postgres·redis·eureka + loki·alloy·grafana). Alloy 소켓 접근 에러 0.
- **gateway ECS JSON**: 새 이미지 + `SPRING_PROFILES_ACTIVE=docker` 확인. `request.in`/`request.out` 이벤트에 `status`·`latencyMs`·`requestId`·`service.name` 필드 출력. 예) `GET /api/board/posts -> 200 32ms`.
- **Loki 파싱 성립**: `{service="gateway"} | json | status=200` 매칭(콜드스타트 4.5s 요청도 포착). Grafana `http://localhost:3000`(익명) Explore에서 조회 가능.
- rate limit `for` 루프 429(클라이언트별)는 브라우저 다중 IP 재현 필요 — XFF 로직은 단위테스트로 고정, 실부하 확인은 후속.

← 처음으로: [[00 개요 — 전체 구조]]

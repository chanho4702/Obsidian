# 검증 기록

이 문서는 템플릿이 보장한다고 말할 수 있는 범위와 아직 검증하지 않은 범위를 분리한다. 날짜와 환경이 달라지면 같은 절차를 다시 실행한다.

## 2026-08-15 로컬 검증

환경: Windows, JDK 21, Docker Desktop, Node.js 기반 구성기.

| 대상 | 결과 | 확인 내용 |
|---|---|---|
| Gradle 전체 프로젝트 | 통과 | 모든 starter, `sample-service`, `gateway-service` 테스트 |
| 구성 마법사 | 통과 | 프로덕션 빌드, 서버 렌더 테스트, `npm audit` 취약점 0건 |
| Compose | 통과 | 모든 profile을 함께 적용한 구성 해석 |
| PostgreSQL 복제 | 통과 | writer의 `pg_is_in_recovery() = false`, reader는 `true`, 생성 행 복제 확인 |
| DB 읽기/쓰기 라우팅 | 통과 | 쓰기는 writer, `@Transactional(readOnly=true)` 조회는 reader에서 수행 |
| writer 중단 시 읽기 | 통과 | writer 컨테이너 중단 중에도 기존 행의 read-only API가 성공 |
| 게이트웨이 | 통과 | 게이트웨이 경유 샘플 API 200, 요청 ID 전달 및 응답 헤더 단일화 |
| Redis/Kafka/Elasticsearch 이미지 | 정적 확인 | 정확한 이미지 태그와 Compose 구성 확인, 기능별 통합 부하는 미실행 |

## 아직 보장하지 않는 것

- 특정 TPS, 지연 시간 또는 가용성 수치
- Kafka broker 장애, consumer 재처리, outbox 원자성
- Elasticsearch 대량 색인과 shard 재배치
- Redis failover와 cache stampede 제어
- Kubernetes 다중 AZ 배포, HPA/PDB, backup/restore

성능 수치는 [처리량과 가용성 검증](capacity-testing.md)의 조건을 기록한 실측 결과가 생긴 뒤에만 추가한다.

## 2026-08-16 부하 시나리오와 회귀 검증

| 대상 | 결과 | 확인 내용 |
|---|---|---|
| Gradle 전체 프로젝트 | 통과 | JDK 21로 전체 starter와 서비스 테스트 재실행 |
| 구성 마법사 | 통과 | 프로덕션 빌드와 서버 렌더 테스트 재실행 |
| Compose | 통과 | 기본 구성과 모든 profile 구성 해석 |
| knee point 시나리오 | 단위 검증 | 증가 TPS 단계, 회복 단계, VU 상한 계산 |
| spike 시나리오 | 단위 검증 | baseline→3배 spike→baseline 복구 단계 |
| soak 시나리오 | 단위 검증 | 명시적 목표 TPS와 기본 4시간 지속 설정 |
| 실행 안전장치 | 단위 검증 | 비로컬 대상 opt-in, Git SHA·환경·사양·데이터셋·결과 경로 필수화 |
| 결과 계약 | 단위 검증 | 메타데이터, 입력값과 k6 원본 summary를 JSON에 보존 |
| 결과 리포트 | 단위 검증 | JSON 검증, 핵심 지표·threshold·비보장 문구 Markdown 출력 |
| 관측성 profile | 정적 검증 | localhost 전용 Prometheus/Grafana와 구성기 profile 연결 |
| Grafana dashboard | 단위 검증 | 요청률, 오류율, p95/p99, CPU, JVM heap, DB pool query 계약 |

이 검증은 k6 설정 계산을 확인한 것이며 실제 부하 결과가 아니다. 특정 TPS, 지연 시간 또는 가용성 수치를 추가로 보장하지 않는다.

## 2026-08-16 로컬 탐색 실측

이 절은 공식 C1/C2 기준선이 아니라 실행 하네스와 장애 시나리오를 확인한 **로컬 탐색 결과**다.

### 실행 환경

| 항목 | 값 |
|---|---|
| 기준 코드 | `a32a97979d71c2f7300d7def3dceae920acece45` |
| 작업 트리 | 문서와 p99 summary 보완이 포함된 dirty 상태, spike 실행 시 diff hash `0f600242bdbaefd94fbea6e070bbd6aa028f755a` |
| OS·CPU | Windows, AMD Ryzen 5 7530U, 논리 프로세서 12개 |
| 메모리 | 약 29.8 GiB |
| 애플리케이션 | sample-service 직접 호출, Gradle `bootRun`, JDK 21, DB pool 최대 10 |
| 부하 발생기 | Docker Desktop 25.0.3, k6 v2.2.0 |
| k6 이미지 | `grafana/k6@sha256:5221b620a4f874faff6e32ba597aa667c058391fe4898b1c6f6377f062c6cdec` |
| 데이터셋 | item 1건, `GET /api/v1/items` 읽기 요청 |
| 의존성 | PostgreSQL 17.6 writer+streaming reader, Prometheus 3.13.1, Grafana 13.1.0 |
| 알려진 간섭 | 같은 PC에서 다른 프로젝트 컨테이너가 실행 중이었고 8080 포트 충돌로 Gateway는 측정에서 제외 |

### 짧은 knee probe

10→25→50→100 TPS를 단계당 30초, 회복 30초로 실행했다.

| 지표 | 결과 |
|---|---:|
| 요청 수 | 5,849 |
| 전체 평균 요청률 | 38.99 req/s |
| 오류율 | 0% |
| p50 | 5.76 ms |
| p95 | 9.84 ms |
| 최대 지연 | 61.1 ms |
| dropped iteration | 0 |

이 실행은 단계가 짧고 최고 100 TPS까지만 시험했으므로 knee point를 결정하지 않는다. 첫 결과에서 k6 summary의 p99 숫자가 누락되는 문제를 발견했고, `summaryTrendStats`에 p99를 명시한 뒤 계약 테스트를 추가했다.

### 3배 spike probe

50 TPS baseline에서 150 TPS까지 올린 뒤 50 TPS로 복구했다. 총 실행 시간은 5분 30초다.

| 지표 | 결과 |
|---|---:|
| 요청 수 | 22,499 |
| 전체 평균 요청률 | 68.18 req/s |
| 오류율 | 0% |
| p50 | 6.63 ms |
| p95 | 12.28 ms |
| p99 | 24.07 ms |
| 최대 지연 | 495.4 ms |
| dropped iteration | 0 |
| 초기 threshold | 모두 PASS |

이 결과는 1건 데이터의 단순 조회를 단일 로컬 서비스에 직접 보낸 결과다. 실제 서비스 payload나 Gateway 경유 성능으로 일반화하지 않는다.

### reader 중단과 복구

20 TPS를 90초 유지하면서 reader 컨테이너를 약 17초 중단한 뒤 재시작했다.

| 지표 | 결과 |
|---|---:|
| 요청 수 | 1,713 |
| 전체 평균 요청률 | 19.03 req/s |
| 오류율 | 15.879% |
| p50 | 9.08 ms |
| p95 | 3,020.2 ms |
| p99 | 3,024.16 ms |
| 최대 지연 | 3,036.21 ms |
| dropped iteration | 88 |
| 초기 threshold | FAIL |
| reader 재시작 후 health/API | `UP`, 조회 성공 |

현재 구현은 reader URL이 비어 있을 때 시작 시 writer로 fallback하지만, 실행 중 reader 연결 장애를 writer로 자동 전환하지 않는다. 단일 reader를 중단한 동안 HikariCP connection timeout 약 3초가 지연에 반영됐고 목표 TPS도 유지하지 못했다. 운영 가용성은 관리형 reader endpoint 또는 DB proxy와 함께 별도로 검증해야 한다.

### 10분 soak 사전 점검

정식 4시간 soak 전에 50 TPS를 10분 유지했다.

| 지표 | 결과 |
|---|---:|
| 요청 수 | 30,001 |
| 요청 처리율 | 50.00 req/s |
| 오류율 | 0.007% — 연결 timeout 2건 |
| p50 | 6.99 ms |
| p95 | 11.66 ms |
| p99 | 30.02 ms |
| 최대 지연 | 245.12 ms |
| dropped iteration | 0 |
| 초기 threshold | PASS |
| process CPU 관찰 최대 | 약 2.17% |
| DB active / pending 관찰 최대 | 1 / 0 |
| heap 관찰 처음 / 마지막 / 최대 | 약 64.67 / 209.13 / 209.16 MiB |

두 요청은 Docker의 `host.docker.internal:8081` 연결 단계에서 I/O timeout이 발생했다. 기본 오류율 임계치 0.1% 이내였지만 무오류 실행은 아니다.

heap 사용량은 10분 구간에서 증가했지만, JVM이 확보한 메모리를 즉시 반환하지 않는 정상 동작과 누수를 이 길이의 실행만으로 구분할 수 없다. 정식 4시간 soak에서 GC 이후 바닥값, old generation, GC pause와 시간별 기울기를 다시 확인해야 한다.

### 앱 인스턴스 하나 제거

`capacity-ha` profile의 로컬 Nginx proxy 앞에 sample-service 두 개를 `8081`, `8083`으로 실행했다. 50 TPS를 90초 유지하고 시작 약 25초 뒤 `8083` Java 프로세스를 정확한 listening PID로 확인해 중단했다.

| 지표 | 결과 |
|---|---:|
| 요청 수 | 4,500 |
| 요청 처리율 | 50.00 req/s |
| 오류율 | 0% |
| p50 | 12.52 ms |
| p95 | 40.65 ms |
| p99 | 101.80 ms |
| 최대 지연 | 219.07 ms |
| dropped iteration | 0 |
| 초기 threshold | PASS |
| `8081` 직접 성공 로그 | 3,771건 |
| 중단 전 `8083` 성공 로그 | 708건 |
| `8083` 실패 후 `8081` 재시도 성공 | 21건 |
| 최종 proxy 5xx | 0건 |
| 제거한 `8083` 재기동 후 | health `UP`, proxy 조회 성공 |

결과 metadata의 작업 트리 diff hash는 `b75e35a6e50e6bcc3e6cc351c2068bc720cee393`이다. Nginx 로그의 `upstream_status=502, 200`은 첫 upstream 연결은 실패했지만 두 번째 upstream 응답으로 최종 HTTP 200이 반환됐다는 뜻이다.

이 검증은 멱등 GET, 단일 PC, 로컬 Nginx round-robin 조건에서만 유효하다. POST 자동 재시도, Kubernetes Service/readiness, managed load balancer와 다중 AZ 동작을 증명하지 않는다.

### 이 실측으로 아직 말할 수 없는 것

- 깨끗한 Git 커밋을 기준으로 반복 실행한 공식 기준선
- 4시간 이상 soak에서의 heap·GC 안정성 — 10분 사전 점검만 완료
- 10,000건 이상 데이터와 현실적인 요청 조합의 성능
- Gateway를 경유한 end-to-end 지연
- Kubernetes 또는 managed load balancer에서 앱 인스턴스를 제거한 뒤의 가용성 — 로컬 Nginx 실험만 완료
- C1 또는 C2 등급 충족

로컬 원본 결과는 `load-tests/results/`에 생성되며 Git에는 포함하지 않는다. 재현 절차는 [처리량과 가용성 단계별 가이드](capacity-testing.md)를 따른다.

## 2026-08-16 서비스 프론트엔드 기반 검증

이 검증은 `1c52d5593c71a189b0df5c16c664eced808c55d9` 위에 프론트 변경을 적용한 작업 트리에서 실행했다. 성능 기준선이 아니라 빌드와 연결 계약 검증이다.

| 대상 | 결과 | 확인 내용 |
|---|---|---|
| pnpm frozen install | 통과 | Node workspace의 lockfile 재현 |
| 공통 API client | 통과 | TypeScript, GET/POST, Problem Detail, validation, request ID 4개 테스트 |
| React SPA | 통과 | TypeScript, runtime config, 로딩·목록·오류 화면 7개 테스트 |
| production build | 통과 | Vite 8.2.1, JS 약 201.76 kB / gzip 63.95 kB, CSS 약 13.53 kB / gzip 4.00 kB |
| 구성 마법사 | 통과 | `none/spa/ssr` 선택을 포함한 production build와 server render |
| 실제 local 연결 | 통과 | Vite `:5173` → proxy → Gateway `:8082` → sample-service `:8081` → PostgreSQL GET/POST/재조회 |
| 의존성 취약점 | 통과 | pnpm production audit와 구성기 npm production audit에서 알려진 취약점 0건 |

로컬 `8080`은 다른 Docker/WSL process가 사용 중이라 Gateway를 `8082`로 실행하고 `GATEWAY_PROXY_TARGET`으로 Vite proxy를 맞췄다. 통합 확인 중 `frontend-qa-20260816` 항목 1건을 로컬 DB에 생성했다.

연결 가능한 인앱/외부 브라우저가 없어 자동 스크린샷 기반 시각 QA는 실행하지 못했다. 반응형 breakpoint, keyboard focus, `prefers-reduced-motion`과 상태별 컴포넌트는 코드와 단위 테스트로 확인했지만 실제 브라우저의 데스크톱·모바일 픽셀 검토는 후속 확인이 필요하다.

아직 검증하지 않은 범위:

- OIDC 로그인·token 갱신·로그아웃
- OpenAPI 생성 client와 백엔드 schema drift 검출
- 실제 브라우저 E2E
- 독립 프론트 컨테이너와 운영 ingress
- SSR adapter

## 2026-08-16 선택형 OIDC와 OpenAPI 계약 검증

이 검증은 `ed75b86f3f1cbe0aef5a5c03eeb6a87696803114` 위 작업 트리에 선택형 인증과 API 계약 변경을 적용해 실행했다. 실제 사용자 트래픽의 보안 인증이나 성능 기준선이 아니라 코드·설정·생성물 계약 검증이다.

| 대상 | 결과 | 확인 내용 |
|---|---|---|
| 전체 backend | 통과 | Java 21 `./gradlew test`, sample controller의 GET 200·POST 201·validation 400 계약 포함 |
| OpenAPI drift | 통과 | OpenAPI 3.1.2 명세에서 `openapi-typescript` 7.13.0 생성, `--check` 일치 |
| API client | 통과 | TypeScript와 GET/POST·Problem Detail·request ID·Bearer token 5개 테스트 |
| React SPA | 통과 | runtime config, OIDC 비활성·login/logout callback·silent renew, 미로그인 차단과 기존 화면 총 17개 테스트 |
| production build | 통과 | main JS 210.14 kB / gzip 66.41 kB, OIDC 별도 chunk 67.50 kB / gzip 17.04 kB, CSS 15.00 kB / gzip 4.30 kB |
| Keycloak realm | 통과 | Keycloak 26.4.2 임시 fresh import, issuer discovery, 공개 `template-spa`, Standard Flow, S256 PKCE, login/logout exact redirect와 `local-user` 확인 |
| Compose | 통과 | 기본 구성과 database-ha/cache/messaging/search/identity/observability 전체 profile config |
| 구성 마법사 | 통과 | production build와 server render 1개 테스트 |
| 부하 계약 | 통과 | 15개 Node 계약 테스트 |
| 의존성 취약점 | 통과 | pnpm production audit와 구성기 npm production audit에서 알려진 취약점 0건 |

Keycloak 검증은 기존 Compose DB를 건드리지 않도록 `18180`의 임시 컨테이너에서 수행하고 컨테이너를 제거했다. realm JSON이 fresh import에서 실제 해석된다는 사실을 확인한 것이며, 기존 realm에는 Keycloak의 `IGNORE_EXISTING` 정책 때문에 새 client가 자동 덮어써지지 않는다. 기존 realm 사용자는 [OIDC 인증 가이드](authentication.md)의 Admin Console 확인 절차를 따라야 한다.

연결 가능한 인앱/외부 브라우저가 없어 실제 `local-user` 로그인 클릭, callback 화면과 로그아웃의 브라우저 E2E·스크린샷은 실행하지 못했다. 다음 항목은 후속 환경 검증으로 남는다.

- 실제 브라우저 → Keycloak → SPA callback → Gateway JWT 검증의 end-to-end smoke
- dev/prod IdP의 HTTPS, exact redirect, MFA와 사용자 lifecycle 정책
- BFF/HTTP-only cookie와 CSRF adapter
- 프론트 독립 컨테이너와 운영 ingress/CORS

## 2026-08-19 공통 상태 처리와 브라우저 E2E 검증

이 검증은 `ad87cd0` 위 작업 트리에서 공통 실패 해석기, 공통 상태 컴포넌트와 Playwright E2E를 적용해 실행했다. 성능 기준선이 아니라 화면 상태 계약 검증이다.

환경: Windows 11, Node.js 24.16.0, pnpm 11.10.0, Playwright 1.62.1, Chromium 151.0.7922.34.

| 대상 | 결과 | 확인 내용 |
|---|---|---|
| API client | 통과 | TypeScript와 GET/POST·Problem Detail·request ID·Bearer token 5개 테스트 |
| 실패 해석 | 통과 | 401/403/404/409/422/429/5xx 분류, violation 전달, fetch TypeError, abort 판별 13개 테스트 |
| 상태 컴포넌트 | 통과 | 로딩·빈 상태·권한 안내·실패 알림의 role과 행동 버튼 8개 테스트 |
| React SPA | 통과 | runtime config, OIDC, 재시도 복구, 401 재로그인, violation 표시를 포함한 총 42개 테스트 |
| 브라우저 E2E | 통과 | Chromium 12개 시나리오: 로딩·빈 목록·목록 순서·5xx 복구·연결 실패·403·생성 성공·validation·빈 입력·인증 켬 미호출·OIDC 이동·설정 실패 |
| production build | 통과 | main JS 213.91 kB / gzip 67.60 kB, OIDC 별도 chunk 67.50 kB / gzip 17.04 kB, CSS 16.13 kB / gzip 4.52 kB |

E2E는 `vite build` 산출물을 `vite preview`로 띄우고 실제 Chromium에서 연다. `/api/v1/items`와 `/app-config.json` 응답은 브라우저 단계에서 stub으로 대체하므로 Gateway, sample-service, PostgreSQL, Keycloak을 띄우지 않아도 항상 같은 결과가 나온다. 이 방식은 화면 상태 계약을 검증하지만 Gateway routing, CORS, 실제 OIDC redirect는 검증하지 않는다.

`vite preview`의 기본 host가 IPv6 `localhost`로만 바인딩되는 환경이 있어 preview host를 `127.0.0.1`로 고정했다.

아직 검증하지 않은 범위:

- 실제 Gateway·Keycloak을 함께 띄우는 통합 E2E
- Chromium 외 브라우저와 모바일 viewport
- 시각 회귀(스크린샷 비교)
- 프론트 독립 컨테이너와 운영 ingress/CORS

## 2026-08-20 선택형 생성과 운영 설정 계약 검증

구성 마법사의 `redis`, `kafka`, `elasticsearch`, `oidc`, `observability` 선택이 서비스 dependency와 Spring 설정에 함께 반영되도록 생성기를 보완한 뒤 검증했다.

| 대상 | 결과 | 확인 내용 |
|---|---|---|
| Gradle 전체 프로젝트 | 통과 | 모든 starter와 서비스 테스트, security/observability 명시적 활성화와 adapter 자동 구성 회귀 포함 |
| PowerShell/Bash 생성기 | 통과 | 임시 저장소에서 실제 스크립트를 실행해 동일한 starter와 `application-platform-<feature>.yml` 생성 확인 |
| 설정 적용 도구 | 통과 | Redis/Kafka/Search/OIDC/Gateway 인증/관측성 활성화 환경변수 생성 |
| prod 설정 계약 | 통과 | sample-service, Gateway와 기능별 템플릿에 localhost·로컬 비밀번호·보안 비활성화 기본값이 없음을 자동 검사 |
| 구성 마법사 | 통과 | ESLint, production build와 server render 테스트 |
| Compose | 통과 | database-ha/cache/messaging/search/identity/observability/capacity-ha 전체 profile 구성 해석 |
| 부하 하네스 | 통과 | 관측성·capacity proxy와 시나리오/리포트 계약 15건 |
| 프론트 | 통과 | API client 5건, React 42건, production build, Chromium E2E 12건 |

observability starter는 `platform.observability.enabled=true`일 때만 exporter 구성을 유지한다. 값이 없거나 false이면 Prometheus·OTLP metrics, tracing과 OTLP logging exporter를 비활성화한다. security starter도 값이 없으면 JWT 보호를 자동 활성화하지 않고, 명시적으로 true인 경우에만 resource server chain을 만든다.

prod 계약 테스트는 위험한 기본값의 재유입을 막는 정적 검증이다. 실제 운영 Secret이 유효한지, 관리형 Redis·Kafka·Elasticsearch와 TLS/인증 연결이 성공하는지, 실제 IdP와 Gateway 인증 E2E가 동작하는지는 배포 환경에서 별도로 검증해야 한다.

## 2026-08-23 생성 서비스 빌드 계약과 Bash 스키마 검증

`69b6b9f` 위 작업 트리에서 P0-A(생성기 변경 마무리)를 적용한 뒤 검증했다. 환경: Windows 11, JDK 21, Node.js 24.16.0, pnpm 11.10.0, PowerShell 7.6.5, Git Bash.

| 대상 | 결과 | 확인 내용 |
|---|---|---|
| Bash 생성기 수정 | 통과 | 기능 선택 시 `spring.config.import` 상위 키(`config:`/`import:`) 없이 깨진 YAML을 만들던 버그를 수정하고 회귀 검사를 추가 |
| Bash JSON Schema 검증 | 통과 | `tools/lib/validate-template-config.mjs`(Node.js, 외부 의존성 없음)로 생성 전 스키마 검증, 위반 경로 출력 후 파일 미생성 종료 |
| 검증기 계약 | 7건 통과 | 필수 속성, type/const/enum/pattern, 추가 속성 금지, 수치 범위, 중복 environments, 잘못된 JSON, 저장소 예제 설정 |
| 생성기 계약 | 6건 통과 | PS/Bash 정상 생성 2건, PS/Bash 스키마 위반 거부 2건(생성물 없음 확인), 기능 스위치 env, prod 위험 기본값 부재 |
| 생성 서비스 선택 계약 | 통과 | PS/Bash × none·기능별 단독 5종·전체 = 14개 서비스에서 placeholder 부재, 선택한 starter·`application-platform-<feature>.yml`·import만 존재 |
| PS↔Bash 동등성 | 통과 | 조합별로 파일 목록과 정규화(개행·이름 토큰) 내용 일치 |
| 생성 서비스 Gradle 빌드 | 통과 | 7개 조합 전부 Java 21에서 `:services:<name>:test` 통과 (`pnpm tools:test:generated-build`) |
| profile 계약(생성 서비스 테스트에 포함) | 통과 | local profile은 localhost 기본값으로 설정 해석, dev/prod는 `DB_WRITER_URL` 미주입 시 placeholder 해석 단계에서 fail-fast |
| observability 실제 기동 | 통과 | `SpringApplication` 실기동에서 flag 미설정 시 Prometheus registry·OTLP span exporter 부재와 export 플래그 false, `enabled=true`일 때 Prometheus·Tracer·OTLP exporter 생성 |
| Gradle 전체 프로젝트 | 통과 | JDK 21 `./gradlew test` 전체 starter·서비스 테스트 |

한계와 관찰:

- profile 계약은 `ConfigDataEnvironmentPostProcessor`로 실제 기동과 같은 설정 체인을 해석한 검증이다. 실제 DB에 연결하는 `bootRun` 기동과 Flyway migration은 포함하지 않는다.
- Spring Boot는 `management.tracing.enabled=false`여도 no-op 성격의 `micrometerOtelTracer` bean을 유지한다. tracing 차단 계약은 Tracer bean 부재가 아니라 span exporter 부재로 검증했다.
- Bash 생성기는 설정 파일 지정 시 Node.js 22+가 필요하다. 없으면 생성 없이 안내 메시지와 함께 종료한다(요구 사항은 [모듈 카탈로그](module-catalog.md)와 [빠른 시작](quickstart.md)에 기록).
- CI에 `generated-service-build` job을 추가해 같은 계약을 Linux에서도 실행한다. 이 검증 시점에 CI 실행 자체는 하지 않았다.

## 2026-08-23 실제 IdP 인증 브라우저 E2E

`69b6b9f` 위 작업 트리에서 P0-B(SPA Authorization Code + PKCE 실제 E2E)를 적용해 검증했다. 환경: Windows 11, JDK 21, Docker Desktop, Keycloak 26.4.2, PostgreSQL 17.6, Node.js 24.16.0, Playwright Chromium.

`pnpm web:e2e:oidc`(`tools/e2e/run-oidc-e2e.mjs`)는 전용 포트(15432/18180/18081/18082/14173)에 PostgreSQL·Keycloak·sample-service·Gateway(인증 켬)·SPA preview를 격리 실행하고 실제 Chromium으로 검증한 뒤 컨테이너·프로세스를 정리한다. 브라우저 단계 개입은 정적 `app-config.json` 주입 하나뿐이며 로그인·token 교환·API 호출은 전부 실제 구성요소를 지난다.

| 시나리오 | 결과 | 확인 내용 |
|---|---|---|
| 미로그인 차단 | 통과 | 로그인 전 API 미호출, Gateway가 무토큰 요청 401 |
| 실제 로그인 흐름 | 통과 | Chromium에서 `local-user` 로그인 → `/oidc/callback` → Bearer token으로 GET 200, POST 201과 목록 반영 |
| token 갱신 | 통과 | 20초 수명 token 만료 후 refresh token 기반 silent renew로 재로그인 없이 다른 token으로 재조회 성공 |
| 로그아웃 | 통과 | end-session 왕복 뒤 로그인 전 상태 복귀, 보호 API 재차단, 저장 버튼 비활성 |
| 위조 token 거부 | 통과 | 서명 없는 문자열 Bearer 401 |
| issuer 불일치 거부 | 통과 | 같은 Keycloak의 master realm이 발급한 정상 서명 token 401 |
| 만료 token 거부 | 통과 | 2초 수명 client token이 유효 시 200, 만료 후 401 |
| 전체 오케스트레이션 | 통과 | 깨끗한 상태에서 기동→7건 테스트→정리까지 단일 명령으로 완료, 잔여 컨테이너·포트 없음 |
| 프론트 회귀 | 통과 | React 단위 42건, stub 기반 Chromium E2E 12건, TypeScript 검사 |

관찰과 한계:

- Spring Security `JwtTimestampValidator`는 기본 60초 clock skew를 허용한다. 만료 거부는 token 수명 + 60초가 지나야 판정되며, 테스트도 65초 대기 후 401을 확인한다.
- E2E는 `realm-template.json`에서 파생한 임시 realm(짧은 token 수명, E2E 전용 redirect URI, `e2e-short-token` client)을 사용한다. 공유 realm 파일과 기본 로컬 스택(5432/8180/8080/8081)은 변경하지 않는다.
- 로컬 Keycloak 전용 계정·비밀값만 사용하며 운영 Secret을 저장하지 않는다.
- CI에 `oidc-e2e` job을 추가했지만 GitHub Actions에서의 첫 실행 결과는 아직 관찰하지 않았다.
- dev/prod IdP의 HTTPS·exact redirect·MFA·사용자 lifecycle, BFF/HTTP-only cookie와 CSRF adapter는 여전히 검증 범위 밖이다. 프론트 독립 컨테이너와 로컬 ingress는 아래 절에서 검증했다.

## 2026-08-23 프론트 production image와 same-origin ingress smoke

`69b6b9f` 위 작업 트리에서 P1-A(프론트 독립 이미지·ingress)의 로컬 범위를 적용해 검증했다. 환경: Windows 11, Docker Desktop, JDK 21, nginx 1.30.4, Playwright Chromium/Firefox 153.

`pnpm web:image:smoke`(`tools/e2e/run-frontend-image-smoke.mjs`)는 `apps/web/Dockerfile` 이미지를 빌드하고 전용 포트(25432/28081/28082/28090/28091)에 web container → nginx ingress(Compose `frontend` profile) → Gateway → sample-service → PostgreSQL을 격리 실행한 뒤 브라우저 smoke를 실행하고 정리한다.

| 대상 | 결과 | 확인 내용 |
|---|---|---|
| production image 빌드 | 통과 | pnpm workspace 빌드 stage + nginx 정적 서빙, `/api` 미프록시 경계 |
| health 경계 | 통과 | web `/healthz`, ingress `/ingress-health` 200, 이미지 `HEALTHCHECK`와 Compose healthcheck |
| same-origin 흐름 | 통과 | ingress origin 하나로 SPA 로드와 `/api` GET/POST, 모든 API 요청이 ingress origin, CORS preflight(OPTIONS) 0건 |
| 환경별 config 재사용 | 통과 | 같은 이미지의 두 container가 서로 다른 mount된 `app-config.json`(local/dev)을 서빙 |
| 브라우저 범위 | 통과 | 데스크톱 Chromium, 모바일 viewport(Pixel 7), Firefox 각각 same-origin 조회·생성 성공 (6/6) |
| Compose 계약 | 통과 | `frontend` profile 포함 전체 profile `config --quiet` 해석 |

관찰과 한계:

- node:22.13 base image의 corepack이 회전된 npm 서명 키를 몰라 `corepack prepare`가 실패한다. 이미지 빌드는 `npm install -g pnpm@11.10.0`으로 pnpm을 설치한다.
- ingress는 로컬 nginx 예제다. Kubernetes ingress/LB manifest, HTTPS와 운영 CDN 조건은 P1-B 범위로 남는다.
- 스크린샷 기반 시각 회귀는 "필요한 경우" 조건이 아직 확인되지 않아 도입하지 않았다.
- CI에 `frontend-image-smoke` job을 추가했지만 GitHub Actions에서의 첫 실행 결과는 아직 관찰하지 않았다.

## 2026-08-28 P0-C 4시간 soak와 C1/C2 기준선

이 절은 `82907e3431c494dc342b6ac3d952e510f943ee65` 위 깨끗한 작업 트리(soak 실행 시점 기준)에서 [남은 작업](remaining-work.md)의 P0-C를 실측한 기록이다. `load-tests/soak.js`에 `WRITE_RATIO` 옵션을 추가해 GET/POST 혼합을 지원하도록 확장했다(계약 테스트 포함, [처리량과 가용성 단계별 가이드](capacity-testing.md) 10절에 문서화).

### 실행 환경

| 항목 | 값 |
|---|---|
| 기준 SHA | `82907e3431c494dc342b6ac3d952e510f943ee65` |
| OS·CPU | Windows 11, AMD Ryzen 5 7530U, 논리 프로세서 12개 |
| 메모리 | 약 29.8 GiB |
| JDK | 21.0.7 LTS (Temurin), `-Xmx` 등 명시적 heap 상한 없이 기본값 사용 |
| 토폴로지 | k6 → Gateway(`18082`, `platform.observability.enabled=true`) → capacity-proxy(nginx, `18084`) → sample-service `18081`/`18083`(각 HikariCP pool 10) → PostgreSQL writer(`5432`)/reader(`5434`) |
| 포트 비고 | Windows `netsh interface ipv4 show excludedportrange`가 `8025-8124`, `8286-8385`를 포함한 Hyper-V 예약 범위로 `8080-8084`를 모두 막고 있어, 문서 기본 포트 대신 `18080`대로 전체 스택을 재배치했다. `infra/capacity/nginx.conf`는 레포를 수정하지 않고 임시 사본으로 `18081`/`18083` upstream을 가리키게 했다. |
| 관측성 | Prometheus 3.13.1을 기존 `prometheus.yml`(포트 `8080`/`8081` 고정) 대신 `18081`/`18082`/`18083`를 스크레이프하는 임시 설정으로 기존 `prometheus_data` volume에 재기동, Grafana 13.1.0 |
| 데이터셋 | `items` 테이블에 `INSERT ... SELECT generate_series(1, 10000)`로 10,000행을 벌크 삽입, 기존 2행과 합쳐 10,002행에서 시작 |

### SLO, 목표 TPS와 C1/C2 판정 기준 — 실행 전 확정

`ItemController.findAll()`은 페이지네이션이 없어 매 GET 요청이 전체 행을 직렬화한다(10,002행 기준 응답 약 1.1MB). 이 특성 때문에 기본 knee probe(5→40 TPS, Gateway+capacity-proxy 경유)에서 낮은 rate(5 TPS)는 깨끗했지만 40 TPS 램프업 구간에서 http_req_duration·http_req_failed·dropped_iterations threshold가 모두 깨졌고, 16 TPS(=2배 후보)에서도 실패했다. 이 결과를 근거로 실행 전에 다음을 확정했다.

- SLO(이 실행의 합격 기준, 제품 보장 아님): 오류율 < 0.1%, p95 < 500ms, p99 < 1000ms, dropped iteration = 0.
- 목표 TPS = **8**. 90초 사전 확인에서 오류율 0%, p95 178.6ms, p99 221.4ms, dropped 0으로 SLO를 만족하는 것을 확인한 뒤 확정했다.
- C1 판정 기준: 서비스당 2개 배치는 충족하지만 인스턴스별 1~2 vCPU 제한은 적용하지 않았다(호스트 12 vCPU를 프로세스 간 공유). "목표 TPS 2배(16) 30분 유지"를 60초 사전 확인으로 먼저 시험한 결과 threshold가 깨져(아래 표) **C1 기준을 충족하지 못했다**. 정식 30분 실행은 수행하지 않았다.
- C2 판정 기준: "서비스당 3개 이상" 배치 조건 자체를 충족하지 않는다(2개만 구성). 따라서 장애 제거 실험은 진행하되 **C2 등급도 배치 조건에서 이미 미충족**으로 판정한다.
- 결론: 이 실행은 C1 또는 C2 등급을 주장하지 않는다. 4시간 soak와 장애 복구 실측 자체가 목적이다.

| 확인 | 목표 TPS | 결과 |
|---|---:|---|
| 목표 TPS 90초 사전 확인 | 8 | 오류율 0%, p95 178.6ms, p99 221.4ms, dropped 0 — PASS |
| 2배 TPS 60초 사전 확인 | 16 | threshold(`dropped_iterations`, `http_req_duration`, `http_req_failed`) 위반 — FAIL, C1 미충족 |

### 앱 인스턴스 1개 제거 — 목표 TPS(8)에서 Gateway 경유

capacity-proxy 뒤 두 sample-service 중 `18083`을 실행 약 25초 뒤 강제 종료하고 재기동했다. 실제 nginx 오류 로그 기준 연결 거부 구간은 약 45초(계획한 15초보다 김 — 재기동 스크립트가 Gradle 데몬을 통해 JVM을 다시 띄우는 데 예상보다 시간이 걸렸다)였다.

| 지표 | 결과 |
|---|---:|
| 요청 수 | 716 |
| 달성 요청률 | 7.94 req/s |
| 클라이언트 체감 오류율 | 0.000% |
| p50 | 147.5 ms |
| p95 | 873.9 ms |
| p99 | 1,897.0 ms |
| 최대 지연 | 2,973.0 ms |
| dropped iteration | 5 / 720 |
| 초기 threshold | FAIL (`http_req_duration`, `dropped_iterations`) |
| 제거 인스턴스 재기동 후 | health `UP`, proxy `UP` |

nginx의 `proxy_next_upstream`(2회 재시도, 3초 이내)이 실패한 upstream을 자동으로 다른 인스턴스로 우회해 클라이언트 오류율은 0%를 유지했지만, 재시도 비용이 p95/p99 지연에 그대로 반영됐다. 이 결과는 멱등 GET, 로컬 Nginx round-robin, 단일 PC 조건에서만 유효하다.

### reader 컨테이너 장애 — 목표 TPS(8)에서 Gateway 경유

`findAll()`은 `@Transactional(readOnly = true)`로 reader를 사용한다. 실행 약 25초 뒤 `postgres-reader` 컨테이너를 약 15초 중단했다가 재기동했다.

| 지표 | 결과 |
|---|---:|
| 요청 수 | 721 |
| 달성 요청률 | 8.00 req/s |
| 오류율 | 27.878% |
| p50 | 133.5 ms |
| p95 | 1,005.9 ms |
| p99 | 1,018.8 ms |
| 최대 지연 | 1,165.0 ms |
| dropped iteration | 0 |
| 초기 threshold | FAIL (`http_req_failed`, `http_req_duration`) |
| reader 재기동 후 | health `UP`, Gateway 경유 조회 10,002건 정상 |

현재 구현은 reader URL이 비어 있을 때만 시작 시 writer로 fallback하며, 실행 중 reader 연결 장애를 writer로 자동 전환하지 않는다. 단일 로컬 reader를 중단하는 동안 threshold `FAIL`은 스크립트 오류가 아니라 현재 가용성 한계를 측정한 결과다. 관리형 reader endpoint나 DB proxy가 있는 운영 구성에서는 다시 측정해야 한다.

### 4시간 soak — 실행했으나 호스트 경합으로 무효 처리

목표 TPS 8, `WRITE_RATIO=0.1`로 Gateway 경유 4시간 soak를 두 차례 시도했다(첫 시도는 세션 중 중단되어 재시작, 두 번째 시도 `load-tests/results/p0c-soak-4h-r2.json`가 2026-08-28T15:39:41Z에 4시간을 완주했다). 그러나 이 실행 중 같은 PC에서 이 저장소와 무관한 다른 마이크로서비스 스택(`platform-auth`, `platform-keycloak`, `platform-gateway`, `platform-opensearch`, `platform-eureka` 등 십수 개 컨테이너)이 함께 동작하고 있었다.

sample-service의 Prometheus 지표로 확인한 호스트 경합 증거(2026-08-28T11:39Z~15:40Z, 5분 간격):

| 지표 | 최소 | 평균 | 최대 |
|---|---:|---:|---:|
| `system_cpu_usage` (호스트 전체) | 0.28 | 0.85 | 1.00 |
| `process_cpu_usage` (sample-service 자신) | 0.01 | 0.04 | 0.09 |

호스트 CPU의 95% 이상을 이 실행과 무관한 프로세스가 소비했다. 그 결과 k6 요약은 요청 114,877건, 오류율 68.914%, p95 8,459.9ms, p99 11,538.1ms, 최대 30,498.0ms, dropped iteration 324건으로 나왔지만 — 이는 실행 직전 같은 토폴로지에서 확인한 목표 TPS 사전 확인 결과(오류율 0%, p95 178.6ms)와 극단적으로 어긋난다. 이 수치는 템플릿·애플리케이션의 처리 능력이 아니라 **호스트 자원 경합** 하나로 설명된다.

**따라서 이 두 실행 모두 P0-C의 공식 4시간 soak 결과로 사용하지 않는다.** 원본 JSON은 `load-tests/results/p0c-soak-4h.json`, `p0c-soak-4h-r2.json`에 남아 있으나 참고용일 뿐이며, [남은 작업](remaining-work.md)의 4시간 soak 항목은 미완료로 남긴다. 재측정은 이 저장소와 무관한 워크로드가 없는 조용한 호스트에서 다시 수행해야 한다.

### 이 실측으로 아직 말할 수 없는 것

- 깨끗한(호스트 경합 없는) 4시간 soak 결과 — 두 차례 시도 모두 무효 처리했다.
- C1 또는 C2 등급 충족 — 2배 TPS 유지 사전 확인에서 이미 실패해 명시적으로 미충족 판정했다.
- vCPU 제한을 실제로 건 컨테이너/cgroup 환경에서의 동일 실측 — 이번 실행은 호스트 코어를 프로세스 간 공유했다.
- 페이지네이션이 있는 API에서의 동일 실측 — `findAll()`의 무제한 응답 크기가 이번 실행의 지연·오류율에 함께 영향을 준 변수였다.
- 4시간 이상 또는 다중 AZ 조건에서의 soak
- Kubernetes 또는 managed load balancer에서의 인스턴스 제거 — 로컬 nginx round-robin 실험만 완료

# 남은 작업 인수인계

이 문서는 다른 에이전트가 다음 작업을 이어가기 위한 실행 기준이다. 일반적인 제품 로드맵은 [전체 로드맵](roadmap.md), 이미 확인한 범위는 [검증 기록](verification.md)을 기준으로 하고, 이 문서는 **현재 상태와 다음 실행 순서**에 집중한다.

## 1. 현재 상태

- 기준 브랜치: `main`
- 2026-08-23에 선택형 생성 마감(P0-A), 실제 IdP 브라우저 E2E(P0-B), 프론트 production image·same-origin ingress(P1-A)까지 커밋했다. 각 검증 결과는 [검증 기록](verification.md)의 2026-08-23 절들에 있다.
- 2026-08-28에 P0-C를 시도했다. knee/목표 TPS 결정, 인스턴스 제거·reader 장애 실험은 Gateway 경유로 완료했지만, 4시간 soak는 두 차례 모두 같은 PC에서 무관한 다른 마이크로서비스 스택이 함께 실행되며 호스트 CPU를 거의 다 써서(`system_cpu_usage` 평균 85%인데 `process_cpu_usage`는 평균 4%) **무효 처리했다**. 근거와 수치는 [검증 기록](verification.md)의 2026-08-28 절을 본다. P0-C는 여전히 미완료다.
- 커밋과 배포: 사용자가 별도로 요청하기 전에는 수행하지 않는다.
- 로컬 환경 주의: 시스템 `JAVA_HOME`은 `C:\java11`을 가리키지만 저장소 기준선은 Java 21이다. 검증 전 `$env:JAVA_HOME='C:\Program Files\Java\jdk-21'`을 지정한다.
- CI의 `generated-service-build`, `oidc-e2e`, `frontend-image-smoke` job은 로컬에서 같은 명령을 검증했지만 GitHub Actions에서의 첫 실행 결과는 아직 관찰하지 않았다. push 뒤 첫 CI 결과를 확인한다.

다음 작업자는 먼저 `git log`와 이 문서의 완료 표시를 검토하고 필요한 검증을 다시 실행한다.

## 2. 가장 먼저 마무리할 작업

### P0-A. 현재 생성기 변경을 병합 가능한 수준으로 닫기 — 2026-08-23 완료

`pnpm tools:test:generated-build`(CI `generated-service-build` job)가 none/기능별 단독/전체 조합을 실제로 생성하고 Java 21 Gradle로 빌드한다. 세부 결과는 [검증 기록](verification.md)의 2026-08-23 절을 본다.

- [x] `none`, 기능별 단독 선택, 전체 선택 조합으로 임시 서비스를 생성한다.
- [x] 생성 서비스에 placeholder가 남지 않고 `:services:<name>:test`가 통과하는지 자동 검증한다.
- [x] local profile은 로컬 기본값으로 실행 가능하고 dev/prod는 선택한 외부 주소·자격증명 누락 시 fail-fast하는지 확인한다. — 템플릿 테스트가 실제 기동과 같은 설정 체인을 해석해 검증한다. 실제 DB 연결 `bootRun`은 포함하지 않는다.
- [x] Bash 생성기도 PowerShell처럼 JSON Schema를 검증하게 한다. — `tools/lib/validate-template-config.mjs`(Node.js 22+ 필요, 외부 패키지 없음). Node 부재 시 실패 메시지는 문서화했다.
- [x] observability 비활성화가 실제 `SpringApplication` 기동에서도 Prometheus·OTLP exporter와 tracing을 끄는지 통합 테스트한다. — Boot는 no-op 성격의 Tracer bean을 유지하므로 exporter 부재로 검증한다.
- [x] 임시로 만든 서비스와 결과물은 제거하고 작업 트리에 남기지 않는다. — 테스트가 `services/genchk-*`를 생성 후 항상 정리한다.

이 과정에서 Bash 생성기가 기능 선택 시 `spring.config.import` 상위 키 없이 깨진 YAML을 만들던 버그를 발견해 수정했다.

완료 조건 충족 상태:

- PowerShell과 Bash의 선택 결과가 dependency, config import와 활성화 값에서 동일하다. — 조합별 파일 단위 동등성 테스트로 확인.
- 선택하지 않은 starter와 설정 파일이 생성되지 않는다. — 조합별 부재 단언으로 확인.
- 생성된 최소 조합과 전체 조합이 Java 21에서 빌드된다. — 7개 조합 전부 빌드.
- 단위/계약 테스트와 사용 예제가 같은 변경에 포함된다. — 검증기·생성기·빌드 계약 테스트와 문서 갱신 포함.

### P0-B. 실제 IdP 인증 E2E — SPA PKCE 흐름은 2026-08-23 완료

`pnpm web:e2e:oidc`(`tools/e2e/run-oidc-e2e.mjs`)가 격리 스택을 직접 띄우고 실제 Chromium E2E 7건을 실행한다. 세부는 [OIDC 인증 가이드](authentication.md) 8절과 [검증 기록](verification.md)을 본다.

- [x] PostgreSQL, Keycloak, sample-service, Gateway와 SPA를 함께 실행한다. — 전용 포트의 임시 컨테이너·프로세스로 격리 실행하고 종료 시 정리한다.
- [x] 실제 Chromium에서 `local-user` 로그인 → callback → 보호 API GET/POST를 확인한다.
- [x] Gateway가 유효한 Bearer token은 전달하고 무효·만료·issuer 불일치 token은 거부한다. — 만료 판정은 Spring Security 기본 clock skew 60초 이후에만 가능하다.
- [x] token 갱신과 로그아웃 후 보호 API 차단을 확인한다. — refresh token 기반 silent renew가 재로그인 없이 새 token으로 통과함을 확인.
- [x] 테스트가 로컬 Keycloak 전용 계정만 사용하고 운영 Secret을 저장하지 않는다. — E2E 파생 realm의 로컬 전용 값만 사용.
- [x] 전용 CI job을 추가한다. — `oidc-e2e` job을 추가했다. 로컬 실행은 검증했지만 GitHub Actions에서의 첫 실행 결과는 아직 관찰하지 않았다.

BFF를 구현한다면 추가 완료 조건:

- [ ] HTTP-only, Secure, 적절한 SameSite cookie와 세션 만료 정책을 정의한다.
- [ ] 상태 변경 요청에 CSRF 방어를 적용한다.
- [ ] SPA가 access token을 직접 보관하지 않는 경계를 계약 테스트한다.
- [ ] 기존 SPA PKCE 방식과 BFF 방식의 선택 기준을 ADR에 기록한다.

### P0-C. 깨끗한 커밋 기준 4시간 soak와 C1/C2 기준선

공식 성능 기준선은 코드 변경보다 실행 조건의 신뢰성이 중요하다. 사용자가 커밋을 요청하지 않았다면 준비까지만 하고, dirty tree에서 결과를 공식 기준선으로 기록하지 않는다.

- [x] 실행할 Git SHA, 작업 트리 상태, CPU·메모리·JVM·DB 사양을 기록한다. — 2026-08-28, [검증 기록](verification.md)의 P0-C 절.
- [x] SLO, 목표 TPS와 C1/C2 판정 기준을 실행 전에 정한다. — knee probe로 목표 TPS=8을 확정하고, 2배 TPS(16) 사전 확인에서 C1 미충족을 실행 전에 판정했다.
- [x] 10,000건 이상 데이터와 현실적인 GET/POST 비율을 준비한다. — 10,002행 벌크 시딩, soak에 `WRITE_RATIO` 옵션 추가(계약 테스트 포함).
- [x] 서비스 직접 호출이 아니라 Gateway 경유 end-to-end 부하를 포함한다. — knee/실패 실험/soak 모두 Gateway(`18082`) 경유.
- [ ] 최소 4시간 soak에서 p95/p99, 오류율, dropped iteration, CPU, heap/old generation, GC pause, DB pool을 같은 시간대로 수집한다. — 2026-08-28에 두 차례 4시간을 실제로 실행했지만 같은 PC에서 무관한 다른 컨테이너 스택이 동시에 돌며 호스트 CPU를 거의 다 써서(증거: [검증 기록](verification.md) 2026-08-28 절) **무효 처리**했다. 이 저장소와 무관한 워크로드가 없는 조용한 호스트에서 다시 실행해야 한다.
- [x] 앱 인스턴스 제거와 reader 장애 시 복구 시간·오류율을 별도 결과로 남긴다. — 목표 TPS(8)에서 각각 완료, [검증 기록](verification.md) 참조. (soak와 달리 90초짜리 짧은 실험이라 호스트 경합 창이 좁아 유효하다고 판단했다.)
- [ ] 원본 결과 위치와 Markdown 요약을 [검증 기록](verification.md)에 연결한다. — 유효한 4시간 soak가 나온 뒤 마무리.

특정 TPS 또는 C1/C2 충족은 위 근거가 모두 있을 때만 문서에 표시한다. 이번 실행은 2배 TPS 유지에 실패해 **C1/C2 등급을 주장하지 않는다** — 자세한 근거는 [검증 기록](verification.md)을 본다.

재시도 시 체크리스트: 실행 전 `docker ps`로 이 저장소와 무관한 컨테이너가 없는지 확인하고, 실행 중 5분 간격으로 `system_cpu_usage`와 `process_cpu_usage`를 비교해 호스트 경합이 없는지 같이 기록한다.

## 3. 다음 우선순위

### P1-A. 프론트 독립 이미지와 운영 ingress — 로컬 범위는 2026-08-23 완료

`pnpm web:image:smoke`(`tools/e2e/run-frontend-image-smoke.mjs`)와 Compose `frontend` profile이 담당한다. 세부는 [서비스 프론트엔드](frontend.md) 8절과 [검증 기록](verification.md)을 본다.

- [x] `apps/web` production image와 health/readiness 경계를 만든다. — nginx 정적 서빙 + `/healthz` + 이미지 `HEALTHCHECK`.
- [x] 같은 이미지를 환경별 `app-config.json`으로 재사용한다. — `/etc/msa-web/app-config.json` mount 계약, smoke가 같은 이미지의 두 container로 검증.
- [x] ingress/LB에서 SPA와 `/api`를 same-origin으로 연결한다. — Compose `frontend` profile의 nginx ingress 예제. Kubernetes ingress manifest는 P1-B 범위다.
- [x] 다른 origin을 허용해야 할 때만 명시적 origin·method·header CORS 정책을 추가한다. — same-origin에서 CORS 미추가, preflight 부재를 smoke가 단언. 분리 origin 정책은 문서 기준만 유지.
- [x] 실제 container → ingress → Gateway → sample-service smoke를 기록한다.
- [x] 모바일 viewport와 Chromium 외 브라우저를 추가한다. — Pixel 7 viewport와 Firefox project. 스크린샷 회귀는 "필요한 경우" 조건이 아직 없어 도입하지 않았다.

SSR adapter는 실제 SEO·서버 렌더 요구가 확인될 때만 추가한다.

### P1-B. Kubernetes 운영 배포 기반

- [ ] 서비스별 Helm chart와 dev/prod values
- [ ] HPA, PDB, topology spread constraints와 readiness/liveness
- [ ] 최소 권한 NetworkPolicy
- [ ] 별도 Flyway migration job과 실패·rollback runbook
- [ ] OIDC/Keycloak 선택형 values와 Secret Manager adapter 예제
- [ ] backup/restore 절차와 복원 검증

완료 조건은 dev와 prod가 계정·Secret·data plane을 공유하지 않고, 승인된 불변 image digest로 배포되는 것이다. 로컬 Compose를 운영 manifest로 변환해 사용하지 않는다.

### P1-C. 외부 인프라 장애·대용량 검증

- [ ] Redis failover, cache stampede와 TTL 집중 만료
- [ ] Kafka broker 장애, consumer 재처리, 중복 event와 lag 복구
- [ ] Elasticsearch 대량 색인, shard 재배치와 alias 전환
- [ ] 관리형 reader endpoint/DB proxy 장애 전환

현재 reader URL 미설정 시에는 writer로 시작하지만, 실행 중 reader 장애는 writer로 자동 전환하지 않는다. 자동 fallback을 추가할지는 일관성·장애 확산 위험을 비교하는 ADR 없이 결정하지 않는다.

## 4. 수요가 있을 때만 하는 확장 작업

- OpenSearch adapter
- Valkey adapter
- cloud queue adapter
- outbox/CDC 모듈
- object storage 모듈
- gRPC starter와 mTLS

각 adapter는 기존 포트를 억지로 재사용하지 않는다. Redis, Kafka, queue, 검색처럼 의미와 전달 보장이 다른 기술은 별도 계약과 계약 테스트를 가진다.

## 5. 권장 작업 분할

한 번에 전체 로드맵을 구현하지 않는다. 다음 순서로 독립 변경을 권장한다.

1. ~~현재 미커밋 생성기 변경 검토 + 생성 서비스 조합 빌드 테스트~~ — 2026-08-23 완료
2. ~~Bash JSON Schema 검증 또는 명시적 검증 계약~~ — 2026-08-23 완료
3. ~~실제 Keycloak·Gateway 브라우저 E2E~~ — 2026-08-23 완료 (BFF는 필요 확인 시 별도 ADR)
4. ~~프론트 production image와 same-origin ingress~~ — 2026-08-23 완료 (Kubernetes manifest는 6단계에서)
5. 깨끗한 SHA 기준 4시간 soak 및 C1/C2 판정
6. Helm/HPA/PDB/NetworkPolicy와 migration/rollback
7. 실제 수요가 확인된 adapter 한 개씩

각 변경은 코드, 단위/계약 테스트, 문서 예제와 검증 기록을 함께 갱신한다.

## 6. 검증 명령

PowerShell에서 Java 21을 먼저 명시한다.

```powershell
$env:JAVA_HOME='C:\Program Files\Java\jdk-21'
./gradlew test

docker compose --env-file infra/.env.versions -f infra/compose.yml `
  --profile database-ha `
  --profile cache `
  --profile messaging `
  --profile search `
  --profile identity `
  --profile observability `
  --profile capacity-ha `
  config --quiet

Set-Location tools/configurator
npm run lint
npm test
Set-Location ../..

pnpm tools:test
pnpm tools:test:generated-build

Set-Location load-tests
npm test
Set-Location ..

pnpm frontend:check
pnpm web:e2e
pnpm web:e2e:oidc
pnpm web:image:smoke
git diff --check
git status --short --branch
```

Chromium이 없으면 처음 한 번 `pnpm web:e2e:install`을 실행한다.

## 7. 다음 에이전트에게 전달할 요청문

```text
루트 AGENTS.md와 README.md, docs/architecture.md, docs/environments.md,
docs/remaining-work.md를 먼저 읽어라. 현재 작업 트리의 미커밋 변경은 이전 작업 결과이므로
초기화하거나 덮어쓰지 말고 git diff로 검토하라.

P0-A, P0-B(SPA PKCE 브라우저 E2E)와 P1-A(프론트 production image·same-origin
ingress의 로컬 범위)는 2026-08-23에 완료됐다. P0-C는 knee/목표 TPS 결정과
인스턴스 제거·reader 장애 실험까지는 2026-08-28에 완료했지만, 4시간 soak는
같은 PC의 무관한 다른 컨테이너 스택 때문에 두 차례 모두 무효 처리됐다
(docs/verification.md의 2026-08-28 절 참고). 다음 순서는 이 저장소와 무관한
워크로드가 없는 조용한 호스트에서 P0-C의 4시간 soak를 다시 실행하는 것이다.
그 다음은 P1-B(Kubernetes 운영 배포 기반)다.
BFF는 실제 요구가 확인될 때만 별도 ADR로 진행한다. 코드·테스트·문서를 함께
갱신하고, 모든 검증을 Java 21로 실행해 결과를 docs/verification.md에 기록하라.
커밋과 배포는 별도 요청 전에는 하지 마라.
```

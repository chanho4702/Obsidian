# MSA 스타터 템플릿 — 주차별 학습/구현 로드맵

> 전제: 직장 병행 **주 10~15시간** 기준. 전체 약 6~7개월(2026-06 중순 ~ 2027-01 초).
> 풀타임이면 6~8주로 압축. 페이스가 다르면 주차(W) 숫자만 비례로 조정.

## 두 가지 원칙
1. **이론은 just-in-time.** 책을 다 떼고 시작하지 말고, 템플릿에 실제 도메인을 붙여서 문제를 만났을 때 그 주제를 판다. (느린 쿼리 → 그때 인덱스/실행계획)
2. **각 주는 DoD(완료 기준)로 끝낸다.** "읽었다"가 아니라 "돌아간다"로 판정. DoD 미통과 시 다음 주로 넘어가지 않는다.

## 진행 순서 한 줄 요약
인증 마무리 → MSA 템플릿 골격 → DB 실전 → 운영 인프라 → 이벤트(Kafka) → K8s/ArgoCD → Spring AI

---

## Phase 0 — 인증 마무리 (W1–3, 6/15 ~ 7/5)
세션·JWT 직접 구현은 완료. 남은 건 OAuth2/OIDC → Keycloak → Gateway 검증. **직접 구현한 JWT는 Keycloak 도입 후 버려질 코드**라는 점 인지하고 깊이 조절.

| 주차             | 학습                                                                    | 구현                                                  | 완료 기준(DoD)                               |
| -------------- | --------------------------------------------------------------------- | --------------------------------------------------- | ---------------------------------------- |
| W1 (6/15~6/21) | OAuth2 4 grant type, OIDC(ID token vs access token), 토큰 탈취/만료/리프레시 위협 | 기존 JWT 코드 정리(교체 전제)                                 | Authorization Code + PKCE 흐름을 그림으로 설명 가능 |
| W2 (6/22~6/28) | Keycloak 개념: Realm / Client / Role / Mapper                           | Keycloak 도커 기동, Realm·Client 생성, 토큰 발급 테스트(Postman) | Keycloak access token 발급 → 디코딩해 claim 확인 |
| W3 (6/29~7/5)  | Spring resource server(JWT 검증), 검증 위치(Gateway vs 서비스)                 | Spring Boot에 Keycloak 연동, Gateway JWT 검증 필터         | 토큰 없으면 401, 통과한 요청만 서비스 도달               |

> RBAC는 서비스가 생긴 뒤라야 의미가 있으므로 Phase 1 W7에서 함께 잡는다.

---

## Phase 1 — MSA 템플릿 골격 (W4–7, 7/5 ~ 8/2)
목표: **새 서비스 하나 추가하면 바로 붙는 구조.** Phase 0의 Keycloak이 여기 compose에 들어간다.

| 주차 | 학습 | 구현 | 완료 기준(DoD) |
|---|---|---|---|
| W4 | Gradle 멀티모듈, 의존성 방향 | `starter-core/security/logging/domain` 분리, 공통 응답(ApiResponse)·ErrorCode·GlobalExceptionHandler | 새 서비스가 starter-* 의존성만 추가하면 공통 응답/예외 동작 |
| W5 | Spring Cloud Gateway route·filter, 토큰 헤더 전파 | Gateway 라우팅 + 인증 필터 + traceId 발급 시작 | Gateway가 2개 서비스로 라우팅, 토큰·traceId 전파 |
| W6 | OpenFeign vs WebClient, 타임아웃·에러 전파 | 서비스 A→B Feign 호출, 에러 표준화 | A가 B 호출, B의 에러가 표준 응답으로 전파 |
| W7 | (구현 집중) RBAC 역할 모델 | **1차 목표: `docker compose up` → Nginx+Gateway+Keycloak+MySQL+Redis+서비스2개+Prometheus+Grafana 기동.** 시크릿은 `.env`(Vault는 Phase 3) | 로그인→Gateway→서비스 호출 end-to-end 동작 + 역할 기반 접근 제어 |

**→ 여기서 "개발 시작 템플릿" 1차 완성.** 이후부터는 읽는 공부가 아니라 굴러가는 템플릿에 기능 붙이기.

---

## Phase 1.5 — CI 파이프라인 (Phase 1 직후, 약 3일)
CI는 배포 타깃과 무관하므로 K8s(Phase 5)와 분리해 **일찍 도입**. 구현은 GitHub Actions(부담 0), Jenkins/SonarQube자체/Nexus는 나중 슬롯. 상세: [[phase1.5-cicd]]

| 작업 | 구현 | 완료 기준(DoD) |
|---|---|---|
| 기본 CI | `.github/workflows/ci.yml` build+test | push 시 자동 빌드/테스트 |
| 이미지 Push | 멀티스테이지 빌드 → GHCR | main 머지 시 이미지가 GHCR에 올라감 |
| 품질 게이트 | SonarCloud 정적분석+커버리지 | 기준 미달 PR 머지 차단 |

> Compose 단계 배포는 `docker compose pull` 로 GHCR 이미지 사용 (CD 도구 불필요). ArgoCD는 Phase 5.

---

## Phase 2 — DB 실전 (W8–12, 8/2 ~ 9/6)
just-in-time 원칙 적용: 템플릿에 실제 도메인을 붙이고 쿼리를 돌리며 공부. **우선순위: 인덱스·실행계획·트랜잭션·MVCC·N+1·커넥션풀. 샤딩은 보류.**

| 주차 | 학습 | 구현 | 완료 기준(DoD) |
|---|---|---|---|
| W8 | SQL/조인/서브쿼리 복습, 트랜잭션&ACID, 격리 수준 | 도메인 엔티티·리포지토리, 트랜잭션 경계 | 격리 수준별 현상(dirty/non-repeatable/phantom) 재현 테스트 |
| W9 | Lock & MVCC, 비관/낙관 락 | 동시성 시나리오(재고 차감 등)에 락 적용 | 동시 요청에서 정합성 안 깨짐 검증 |
| W10 ★최우선 | B-tree·복합 인덱스 순서, EXPLAIN 읽기 | 느린 쿼리 EXPLAIN → 인덱스 추가 → 측정 | 풀스캔 → 인덱스 스캔 전환 + 응답시간 before/after 비교 |
| W11 | HikariCP 튜닝, N+1, fetch join, batch | N+1 탐지·해결, 풀 사이즈 설정 | 쿼리 수 감소를 로그로 확인 |
| W12 | 파티셔닝/복제/CAP **개념만 정리** | Redis 조회 캐싱 1개 적용 | 캐시 히트/미스 동작 확인 |

---

## Phase 3 — 운영 인프라 (W13–16, 9/6 ~ 10/4)
여기서 템플릿이 "장난감 → 실무형"으로 바뀐다. 관측 가능성 확보가 핵심.

| 주차 | 학습 | 구현 | 완료 기준(DoD) |
|---|---|---|---|
| W13 | Actuator metrics, Prometheus 스크랩, Grafana 패널 | JVM/HTTP 대시보드 | 요청 수·지연 대시보드가 실데이터로 표시 |
| W14 | MDC, traceId/requestId 전파(Gateway→서비스→Feign) | JSON 구조화 로그 + traceId 전파 | 하나의 요청을 traceId로 전 서비스 로그에서 추적 |
| W15 | OpenTelemetry 트레이싱, Loki 로그 수집 | OTel + Loki, Grafana 연동 | 한 요청의 분산 트레이스를 Grafana에서 확인 |
| W16 | Alertmanager 룰, Vault 시크릿 관리 | 임계치 알림 1개, DB비번·JWT secret을 `.env`→Vault 이관 | 알림 발화 테스트 + 앱이 Vault에서 시크릿 로드 |

---

## Phase 4 — 이벤트 기반 확장 (W17–20, 10/4 ~ 11/1)
Kafka는 처음부터 깊게 파지 말고 **선택 기능**으로 붙인다. 목표는 서비스 간 직접 호출 줄이기.

| 주차 | 학습 | 구현 | 완료 기준(DoD) |
|---|---|---|---|
| W17 | Topic/Partition/Consumer Group, 발행·구독 | Kafka compose 추가, 도메인 이벤트 1개 발행/구독 | A의 이벤트를 B가 비동기 소비 |
| W18 | Outbox 패턴(트랜잭션+메시지 원자성) | Outbox 테이블 + 폴러/CDC | DB 커밋과 이벤트 발행 정합성 보장 |
| W19 | 멱등성, 재시도, DLQ | 멱등 소비, 재처리, DLQ | 소비 실패가 DLQ로, 중복 메시지 멱등 처리 |
| W20 | OpenSearch 인덱싱/검색 | 이벤트로 OpenSearch 인덱싱, 검색 API | 쓰기→이벤트→검색 인덱스 반영 |

---

## Phase 5 — Kubernetes + ArgoCD (W21–26, 11/1 ~ 12/13)
**Docker Compose가 완성된 뒤에** 진입. K8s가 서비스 디스커버리를 제공하므로 Eureka는 도입하지 않는다(중복 회피). 이미지 빌드/푸시(CI)는 이미 [[phase1.5-cicd|Phase 1.5]]에서 끝났으므로, 여기는 **CD(ArgoCD)와 K8s 운영**에 집중한다.

| 주차 | 학습 | 구현 | 완료 기준(DoD) |
|---|---|---|---|
| W21 | K8s용 이미지/매니페스트 점검 | GHCR 이미지를 K8s에서 당겨오도록 imagePullSecret 등 정리 | k8s가 GHCR 이미지를 풀해서 기동 |
| W22 | Pod/Deployment/Service/ConfigMap/Secret/Ingress | 로컬 k8s(kind/minikube)에 1서비스 배포 | 서비스가 Ingress로 외부 접근 |
| W23 | readiness/liveness/health check | 전 서비스 매니페스트 작성 | 전체 스택이 k8s에서 기동 |
| W24 | Helm 또는 Kustomize, Namespace 분리 | 차트화 + dev/prod 환경 분리 | 환경별 값으로 배포 |
| W25 | ArgoCD GitOps | Git 푸시 → ArgoCD 자동 동기화 | Git 변경이 클러스터에 자동 반영 |
| W26 | HPA, Rolling Update | 오토스케일 + 무중단 롤링 배포 | 부하 시 스케일아웃, 배포 시 무중단 |

---

## Phase 6 — Spring AI (W27–28, 12/13 ~ 1/3)
기초 인프라가 다 선 마지막에. AI부터 붙이면 산으로 간다.

| 주차 | 학습 | 구현 | 완료 기준(DoD) |
|---|---|---|---|
| W27 | Spring AI 기본, LLM(Claude/OpenAI) 연동, AI Gateway 구조 | 문서 요약 엔드포인트 1개 | 요약 API 동작 |
| W28 | RAG 기본, Vector DB | 임베딩→Vector DB→검색 결합 RAG, 간단 챗봇 | 문서 기반 Q&A 동작 |

---

## 단계별 산출물 체크포인트
- [ ] Phase 0 — Gateway에서 Keycloak 토큰 검증 통과/실패 동작
- [ ] Phase 1 — `docker compose up` 한 방으로 전체 스택 기동 (= 1차 목표)
- [ ] Phase 2 — 느린 쿼리 인덱스 개선 + N+1 제거 측정치
- [ ] Phase 3 — traceId로 전 서비스 분산 추적 가능
- [ ] Phase 4 — Outbox + DLQ로 신뢰성 있는 이벤트 처리
- [ ] Phase 5 — Git 푸시로 무중단 배포(ArgoCD)
- [ ] Phase 6 — RAG 기반 문서 Q&A

## 페이스 조정 가이드
- 주 6~8시간 → 각 단계 기간 약 1.7배(전체 ~11개월)
- 주 20시간+ → 약 0.7배(전체 ~4.5개월)
- 풀타임 → 6~8주 집중(단, K8s/Kafka는 깊이 때문에 단축 폭이 작음)

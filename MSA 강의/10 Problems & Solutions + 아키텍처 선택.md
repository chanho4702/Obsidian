---
tags:
  - MSA
  - microservices
  - problem-solution
  - architecture-selection
  - 강의정리
type: note
---

# 🧩 Problems & Solutions + 아키텍처 선택

> 마이크로서비스 설계 시 마주치는 **문제→해결책 매핑**, 패턴 사용 통계,
> 트래픽 규모별 권장 설계, 그리고 아키텍처 선택 원칙 정리. (강의 종합편)

상위: [[00 MSA 디자인 패턴 강의 — 인덱스]] · 이전: [[09 마이크로서비스 배포]]

---

# 🔧 Problems & Solutions
> 각 설계 영역에서 마주치는 **문제(Problem)** 와 그에 대한 **해결책(Solution)** 매핑.

## 1️⃣ Decomposition (분해)

| 문제 | 해결책 |
|---|---|
| 트래픽 증가, 더 많은 요청 처리 | 애플리케이션을 마이크로서비스로 분할 → **확장성(Scalability)**, 수평/수직 스케일링, 스케일 업/아웃, **Load Balancer** |
| 애플리케이션을 마이크로서비스로 분할 | **Microservices Decomposition Patterns** |

## 2️⃣ Communications (통신) → [[05 마이크로서비스 통신]]

| 문제 | 해결책 |
|---|---|
| 클라이언트-서비스 직접 통신 | Well-defined API Design / Microservices Communication Patterns |
| 서비스 간 통신으로 인한 네트워크 트래픽 증가 | gRPC APIs (scalable & fast) / RPC framework로 서로 다른 기술 스택 개발 가능 |
| 서비스 간 통신으로 발생되는 연쇄적인 쿼리 사용 | Aggregate query operations / **Service Aggregator Pattern** |
| 동기 통신에서는 처리 시간이 오래 걸리는 작업 수행 불가 | Asynchronous Message Based Communications / Working with events |

## 3️⃣ Data Management (데이터 관리) → [[06 마이크로서비스 데이터 관리]]

| 문제 | 해결책 |
|---|---|
| 다양한 데이터 요구 처리·스케일링 시 DB 병목 발생 | Stateful App Horizontal Scaling / Business Boundaries 기준 Data Partitioning (Shards/Pods) / NoSQL Partitioning |
| DB 작업은 비용이 크고 성능이 낮음 | **Distributed cache** → 자주 접근하는 데이터 캐싱 |
| 분산 확장 DB에서 교차 서비스 쿼리·쓰기 명령 | Data Query Pattern / **Materialized View** / **CQRS** / **Event Sourcing** |

## 4️⃣ Transaction Management (트랜잭션 관리) → [[07 마이크로서비스 트랜잭션 관리]]

| 문제 | 해결책 |
|---|---|
| 분산 트랜잭션에서 서비스 간 일관성 관리 | Distributed Transaction 관리 패턴 / **Saga** / **Transactional Outbox** / **Compensating Transaction** / **CDC** |
| 마이크로서비스에서 수백만 개의 이벤트 처리 | **Event-Driven Architecture (EDA)** |

## 5️⃣ Deployment (배포) → [[09 마이크로서비스 배포]]

| 문제 | 해결책 |
|---|---|
| 다운타임 없이 배포 및 유연한 확장 | Containers & Orchestrators / 배포 전략(Blue-Green, Rolling, Canary) / Kubernetes Patterns(Sidecar, Service Mesh) / DevOps & CI/CD / IaC |

---

# 📊 동시접속자 수 기준 권장 설계
> 트래픽 규모에 따라 어떤 설계를 도입해야 하는지에 대한 가이드.

| 동시접속자 수 | 초당 요청 수 (RPS) | 권장 시스템 설계 및 고려사항 |
|---|---|---|
| 500 | 50 RPS | 단일 서버로 처리 가능하나, 향후 확장 대비 설계 필요 |
| 1K | 100 RPS | 로드 밸런서 도입해 2대 서버로 부하 분산 고려 |
| 2K | 200 RPS | 세션 상태 관리를 위한 분산 캐시 도입 검토 |
| 5K | 500 RPS | 데이터베이스 샤딩 또는 리플리케이션 고려 |
| 10K | 1,000 RPS | CDN 활용해 정적 자원 제공 최적화 |
| 20K | 2,000 RPS | 마이크로서비스 아키텍처 도입 검토 |
| 50K | 5,000 RPS | 서비스 메시 도입 및 오케스트레이션 도구 활용 |
| 100K | 10,000 RPS | 글로벌 로드 밸런싱 및 멀티 리전 배포 검토 |
| 500K | 50,000 RPS | 대규모 분산 시스템 설계 및 데이터 일관성 유지 전략 수립 |

> 💡 **읽는 법** — MSA는 동시접속 20K(2,000 RPS) 부근부터 본격 검토 대상. 그 이전엔 단일 서버 → LB → 캐시 → 샤딩 순으로 점진 확장.

---

# 📈 MSA 패턴 사용 통계
> 마이크로서비스 아키텍처 패턴별 사용 분포 (Pattern Usage Distribution).

| 패턴 | 사용 비중 |
|---|---|
| **API Gateway** | 23.1% |
| **Circuit Breaker** | 20.2% |
| **Database per Service** | 16.7% |
| **Strangler** | 14.4% |
| **Sidecar / Service Mesh** | 13.5% |
| **Saga** | 12.1% |

> API Gateway와 Circuit Breaker가 가장 널리 쓰이는 패턴.

---

# 🎯 Choosing Architecture (아키텍처 선택)

## 핵심 원칙
- 모든 아키텍처는 **장단점**을 가지고 있음 → SA(Solution Architect) 관점에서 아키텍처 스타일을 고려해야 함
- "애플리케이션을 위해 **가장 좋은** 아키텍처는?" → 정답은 상황에 따라 다름
- 모놀리식과 마이크로서비스 **둘 다 유효한 패턴**

> ⚠️ **`Microservices > Monolithic` 은 틀렸다 (WRONG!!!)**
> 마이크로서비스가 모놀리식보다 무조건 우월하다는 생각은 잘못됨.

## 비즈니스 요건 중심 설계
- 비즈니스 요건에 따라 아키텍처 디자인
- 요구사항 변화에 맞게 디자인을 **계속 업데이트**
- 모든 디자인 결정은 비즈니스 요구사항을 따름
- **기능적(Functional) / 비기능적(Non-Functional)** 요구사항을 명확히 정의 ([[01 이커머스 플랫폼 요구사항 정의서]])
- 애플리케이션의 **한계·제약 조건·가정**을 정의하고 비즈니스 목표를 명확히
- **Metrics**를 사용해 시작·운영에 필요한 요건을 정의

> ⚠️ 경계할 점: 엔지니어는 **최신 아키텍처 스타일로 과도하게 엔지니어링**하고 최신 도구를 무작정 쓰려는 경향이 있음 → 지양해야 함.

## Functional vs Non-Functional 요구사항

| Functional | Non-Functional |
|---|---|
| 시스템이 제공해야 할 서비스에 대한 명세 | 타이밍 제약 (Timing constraints) |
| 특정 입력에 대한 시스템 반응 | 개발 프로세스에 대한 제약 |
| 특정 상황에서의 시스템 동작 | 표준에 의해 부과되는 제약 |

---

## ✅ 전체 한 줄 정리
> "유행이 아니라 **비즈니스 요구(기능·비기능)** 에 맞춰 아키텍처를 고른다.
> 마이크로서비스가 항상 정답은 아니며, 과도한 엔지니어링을 경계한다."

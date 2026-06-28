---
tags:
  - MSA
  - architecture
  - 강의정리
type: index
source: Notion - MSA 디자인 패턴 강의 학습
---

# 🧠 MSA 디자인 패턴 강의 — 인덱스

> Notion 학습 노트를 정리·보강한 강의 자료입니다.
> 모놀리식에서 시작해 마이크로서비스의 7대 설계 영역(통신·데이터·트랜잭션·회복성·배포)까지,
> "왜 이 패턴이 필요한가"를 문제→해결 흐름으로 따라갑니다.

---

## 📚 강의 목차

### 1부. 요구사항과 모놀리식
- [[01 이커머스 플랫폼 요구사항 정의서]] — 기능/비기능 요구사항으로 설계의 출발점 잡기
- [[02 모놀리식 아키텍처 — 설계 패턴]] — Layered / MVC / Modular Monolithic, DB·API·보안·성능
- [[03 설계 원칙과 아키텍처별 설계 방법]] — KISS·SOLID 등 원칙 + 모놀리식 vs 모듈러 vs MSA 비교

### 2부. 마이크로서비스 7대 설계 영역
- [[04 마이크로서비스 아키텍처 설계 영역]] — 전체 설계 영역 한눈에 보기 (분해·통신·데이터·트랜잭션·회복성·관측성·배포)
- [[05 마이크로서비스 통신]] — 동기(REST/gRPC/Gateway) vs 비동기(Message Broker/Pub-Sub)
- [[06 마이크로서비스 데이터 관리]] — Database-per-Service, CAP, CQRS, 최종 일관성
- [[07 마이크로서비스 트랜잭션 관리]] — SAGA, Outbox, CDC, EDA
- [[08 마이크로서비스 회복성·관측성·모니터링]] — Circuit Breaker, 분산 추적, ELK·Prometheus
- [[09 마이크로서비스 배포]] — Docker/Kubernetes, Service Mesh, CI/CD, 배포 전략

### 3부. 종합
- [[10 Problems & Solutions + 아키텍처 선택]] — 문제→해결책 매핑, 트래픽별 권장 설계, 아키텍처 선택 원칙

---

## 🎯 강의를 관통하는 한 줄

> **"유행이 아니라 비즈니스 요구(기능·비기능)에 맞춰 아키텍처를 고른다.
> 마이크로서비스가 항상 정답은 아니며, 과도한 엔지니어링을 경계한다."**

## 🧩 전체 패턴 지도

```text
요구사항 정의 ──▶ 모놀리식으로 시작 ──▶ 필요 시 모듈 분리 ──▶ 마이크로서비스
   (01)            (02·03)              (Modular)            (04~09)

마이크로서비스 7대 영역
 ├─ 분해(Decomposition)      → 서비스 경계
 ├─ 통신(Communication)      → REST·gRPC·Message Broker     [[05 ...]]
 ├─ 데이터(Data)             → DB-per-Service·CQRS          [[06 ...]]
 ├─ 트랜잭션(Transaction)    → SAGA·Outbox·CDC·EDA          [[07 ...]]
 ├─ 회복성(Resilience)       → Circuit Breaker·Retry        [[08 ...]]
 ├─ 관측성(Observability)    → 분산 추적·ELK·Prometheus     [[08 ...]]
 └─ 배포(Deployment)         → Docker·K8s·CI/CD             [[09 ...]]
```

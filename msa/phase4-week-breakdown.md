---
tags: [msa, kafka, 이벤트, phase4]
status: 대기
기간: 2026-10-04 ~ 2026-11-01
관련: "[[msa-roadmap]]"
이전: "[[phase3-week-breakdown]]"
---

# Phase 4 — 이벤트 기반 확장 (행동 단위)

> 한 칸 = 1~3시간 + 미니 DoD. Kafka는 **선택 기능**으로 붙인다. 목표는 서비스 간 직접 호출 줄이기.

## W17 (10/4~10/10) — Kafka 기초
- [ ] **Kafka compose 추가** — broker + UI(kafka-ui 등)
	- DoD: 토픽 생성 가능
- [ ] **이벤트 발행** — KafkaTemplate
	- DoD: 토픽에 메시지 적재
- [ ] **이벤트 구독** — `@KafkaListener`
	- DoD: B가 이벤트 소비
- [ ] **Consumer Group 이해** — 파티션/오프셋
	- DoD: 그룹 단위 소비 확인
- [ ] **(주간 DoD)** — A의 이벤트를 B가 비동기 소비

## W18 (10/11~10/17) — Outbox 패턴
- [ ] **문제 인식** — DB 커밋과 발행의 이중 쓰기 문제
	- DoD: 왜 문제인지 3줄
- [ ] **Outbox 테이블** — 비즈니스 트랜잭션 안에서 이벤트 저장
	- DoD: 비즈니스 + outbox가 동일 트랜잭션
- [ ] **릴레이** — 폴러 또는 CDC(Debezium)
	- DoD: outbox → Kafka 발행
- [ ] **(주간 DoD)** — DB 커밋과 이벤트 발행 정합성 보장

## W19 (10/18~10/24) — 신뢰성
- [ ] **멱등 소비** — 중복 메시지 처리
	- DoD: 같은 메시지 2번 와도 1번 효과
- [ ] **재시도** — backoff 재처리
	- DoD: 일시 실패 자동 재처리
- [ ] **DLQ** — 최종 실패 격리
	- DoD: 실패 메시지가 DLQ로 이동
- [ ] **(주간 DoD)** — 실패는 DLQ, 중복은 멱등 처리

## W20 (10/25~11/1) — 검색(OpenSearch)
- [ ] **OpenSearch 기동**
	- DoD: 인덱스 생성
- [ ] **이벤트 인덱싱** — 도메인 이벤트로 색인
	- DoD: 쓰기→이벤트→색인 반영
- [ ] **검색 API**
	- DoD: 검색 쿼리 동작
- [ ] **(주간 DoD)** — 쓰기→이벤트→검색 인덱스 반영

## Phase 4 종료 기준
- [ ] Outbox + DLQ로 신뢰성 있는 이벤트 처리
- [ ] 서비스 간 동기 호출 일부를 이벤트로 대체
- [ ] 다음 → [[phase5-week-breakdown]]

---
tags: [msa, db, phase2]
status: 대기
기간: 2026-08-02 ~ 2026-09-06
관련: "[[msa-roadmap]]"
이전: "[[phase1.5-cicd]]"
---

# Phase 2 — DB 실전 (행동 단위)

> 한 칸 = 1~3시간 + 미니 DoD. **just-in-time**: 템플릿에 실제 도메인을 붙이고 쿼리를 돌리며 공부.
> 우선순위: 인덱스·실행계획·트랜잭션·MVCC·N+1·커넥션풀. **샤딩은 보류.**

## W8 (8/2~8/8) — 트랜잭션 기반
- [ ] **도메인 쿼리 작성** — SQL/조인/서브쿼리 복습을 실제 조회로
	- DoD: 도메인 핵심 조회 3개 작성
- [ ] **트랜잭션 경계** — `@Transactional` 범위 설정
	- DoD: 한 유스케이스가 원자적으로 commit/rollback
- [ ] **격리 수준 실험** — READ COMMITTED vs REPEATABLE READ
	- DoD: dirty/non-repeatable/phantom 중 2개 재현 테스트
- [ ] **(주간 DoD)** — 각 현상이 어느 격리에서 막히는지 표로 정리

## W9 (8/9~8/15) — Lock & MVCC
- [ ] **MVCC 정리** — 스냅샷 읽기 개념
	- DoD: "왜 읽기가 쓰기를 막지 않나" 2문장
- [ ] **비관적 락** — `SELECT ... FOR UPDATE`
	- DoD: 재고 차감에 락 적용
- [ ] **낙관적 락** — `@Version`
	- DoD: 충돌 시 OptimisticLockException 발생
- [ ] **동시성 테스트** — 병렬 요청
	- DoD: 동시 차감에서 재고 음수 안 됨
- [ ] **(주간 DoD)** — 동시 요청 시나리오에서 정합성 검증

## W10 (8/16~8/22) — 인덱스 + 실행계획 ★최우선
- [ ] **EXPLAIN 읽는 법** — type/key/rows/Extra
	- DoD: 한 쿼리 실행계획 해석 노트
- [ ] **느린 쿼리 특정** — 풀스캔 쿼리 1개
	- DoD: 대상 쿼리 + 현재 응답시간 기록
- [ ] **인덱스 추가** — 단일/복합, 컬럼 순서
	- DoD: EXPLAIN에서 인덱스 사용 확인
- [ ] **복합 인덱스 순서 실험** — (a,b) vs (b,a)
	- DoD: 순서가 결과에 미치는 영향 확인
- [ ] **(주간 DoD)** — 풀스캔→인덱스스캔 전환 + before/after 측정치

## W11 (8/23~8/29) — 커넥션 풀 + JPA 성능
- [ ] **HikariCP 설정** — pool size, timeout
	- DoD: 설정 근거 노트
- [ ] **N+1 재현** — 연관 조회 쿼리 폭발
	- DoD: N+1 발생 로그 확인
- [ ] **fetch join / EntityGraph**
	- DoD: 쿼리 수 N+1 → 1~2로 감소
- [ ] **batch size** — `default_batch_fetch_size`
	- DoD: IN 절 배치 적용 확인
- [ ] **(주간 DoD)** — N+1 제거 전후 쿼리 수 비교

## W12 (8/30~9/6) — 심화 개념 + 캐싱
- [ ] **파티셔닝/복제/CAP 개념 정리** — 구현 X, 노트만
	- DoD: 각 주제 핵심 3줄
- [ ] **Redis 조회 캐싱** — `@Cacheable`
	- DoD: 캐시 히트/미스 동작
- [ ] **캐시 무효화** — 갱신 시 evict
	- DoD: 데이터 변경 후 캐시 갱신 확인
- [ ] **(주간 DoD)** — 조회 API 캐시 + 무효화 동작 (샤딩 보류)

## Phase 2 종료 기준
- [ ] 느린 쿼리 인덱스 개선 측정치 확보
- [ ] N+1 제거 전후 비교 확보
- [ ] 다음 → [[phase3-week-breakdown]]

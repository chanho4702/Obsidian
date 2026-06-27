---
tags: [데이터베이스, 로드맵, MOC, 성능, 수평확장]
type: roadmap
status: 예정
updated: 2026-06-09
---

# 🗄️ 데이터베이스 학습 로드맵 (대규모 서비스 대비)

> [!abstract] 목표
> SQL → 정합성 → 성능 → 저장 구조 → 수평 확장 순으로, **"왜 느려지고 어떻게 확장하는가"를 스스로 판단**할 수 있게 된다.

[[인증 로드맵]]과는 별개 트랙. MSA 로드맵의 [[phase2-week-breakdown|Phase 2 (DB 실전)]]에서 4~7단계를 실제 템플릿에 붙여 실습한다.

---

## 단계 (13)

| 단계 | 주제 | 목표 |
|---|---|---|
| **1** | SQL 마스터 | SQL 자유 작성 + 실행계획 읽기 |
| **2** | 트랜잭션 & ACID | 정합성이 왜 깨지는지 |
| **3** | Lock & MVCC | 동시 수정이 왜 문제인지 |
| **4** ⭐ | 인덱스 | 쿼리 성능을 **설계** |
| **5** ⭐ | 실행계획 | 느린 SQL 분석 |
| **6** | 커넥션 풀 | 연결 자원 병목 제거 |
| **7** | JPA 성능 최적화 | ORM이 만드는 쿼리 제어 |
| **8** | 스토리지 엔진 내부 | DB가 내부적으로 어떻게 저장하나 |
| **9** | 파티셔닝 | 수천만~수억 건 처리 |
| **10** | 샤딩 | 여러 노드로 분산 |
| **11** | 캐싱 | 읽기 폭주 완화 |
| **12** | 복제 & 고가용성 | 읽기 성능 + 장애 대응 |
| **13** | 일관성 & CAP | 분산 환경 트레이드오프 |

⭐ = 가장 중요(성능 핵심)

---

## 1. SQL 마스터
SELECT/WHERE · JOIN(INNER/LEFT/RIGHT/FULL OUTER/SELF) · GROUP BY/HAVING · 서브쿼리(스칼라/인라인뷰/상관) · CTE(WITH)·재귀 CTE · 윈도우 함수(`ROW_NUMBER`/`RANK`/`SUM() OVER`/`LAG`/`LEAD`) · `UNION` vs `UNION ALL` · `EXISTS` vs `IN` · 인덱스 고려한 작성 · NULL과 3치 논리(`NULL = NULL`은 UNKNOWN)

## 2. 트랜잭션 & ACID
ACID(원자성/일관성/격리성/지속성) · Commit/Rollback/Savepoint · 이상현상 — **Dirty Read**(커밋 안 된 값), **Non-repeatable Read**(같은 행 다른 값, UPDATE 계열), **Phantom Read**(같은 범위 다른 건수, INSERT/DELETE 계열) · 격리수준(Read Uncommitted/Committed/Repeatable Read/Serializable)
> 내가 쓰는 DB 기본 격리수준 확인 — **MySQL InnoDB = Repeatable Read**, PostgreSQL/Oracle = Read Committed

## 3. Lock & MVCC
Lock(Shared/Exclusive/Gap/Next-Key) · Deadlock 조건과 회피 · MVCC(Undo Log, Read View) · Snapshot Read vs Current Read · 실습: `SELECT ... FOR UPDATE` / `FOR SHARE`

## 4. 인덱스 ⭐
B+Tree(구조/탐색/O(log n)) · Clustered vs Secondary · **복합 인덱스 & Leftmost Prefix Rule** (`INDEX(a,b,c)` 기준):
- 가능: `a=?` / `a=? AND b=?` / `a=? AND b=? AND c=?`
- 반쪽: `a=? AND c=?` → a로만 seek, c는 필터로만(b 건너뜀)
- 멈춤: 중간 컬럼이 범위(`>`,`BETWEEN`,`LIKE`)면 그 뒤 컬럼 못 씀
- `ORDER BY`도 인덱스 순서를 따르면 정렬 비용 제거

커버링 인덱스(테이블 접근 제거) · 선택도/카디널리티 · **쓰기 비용**(쓰기마다 인덱스 갱신 → 무조건 추가 X)

## 5. 실행계획 ⭐
`EXPLAIN`(추정) / `EXPLAIN ANALYZE`(실제 측정) · 주요 항목 — `type`, `key`, `rows`, `filtered`, `Extra` · type 등급(좋아지는 순) — `ALL < INDEX < RANGE < REF < CONST` · 조인 방식 — Nested Loop / Hash / Merge

## 6. 커넥션 풀
HikariCP 풀 사이징(무작정 키우면 역효과) · 커넥션 누수/타임아웃 · 풀 고갈과 대기(인덱스 잘 짜도 풀이 작으면 느림)

## 7. JPA 성능 최적화
Lazy vs Eager · **N+1 문제** · Fetch Join / `@EntityGraph` / `@BatchSize`(`default_batch_fetch_size`) · **OSIV 함정**(Spring Boot 기본 true) · fetch join + 페이징 동시 사용 주의(메모리 페이징 HHH000104) · 조회 전용은 DTO projection / `readOnly` 트랜잭션

## 8. 스토리지 엔진 내부
Page/Block/Buffer Pool · WAL/Redo Log/Undo Log · **B+Tree(읽기 최적) vs LSM Tree(쓰기 최적)**(예: Cassandra, RocksDB)

## 9. 파티셔닝
Range(2026-01/02/03) · Hash(`user_id % N`) · List(country) · Partition Pruning · Local vs Global Index

## 10. 샤딩
파티셔닝(한 서버 내) vs 샤딩(여러 노드) · 샤드 키 선택(잘못 고르면 핫스팟) · 리샤딩(consistent hashing) · cross-shard 조인 문제

## 11. 캐싱
Redis 기본 · Cache-Aside 패턴 · 캐시 무효화/TTL · 캐시 스탬피드(thundering herd) 대응

## 12. 복제 & 고가용성
Master-Slave(쓰기→Master, 읽기→Replica) · Replication Lag/Binlog · Failover(레플리카 승격, split-brain, 앱 재연결) · 매니지드(RDS Multi-AZ) vs 직접(Orchestrator/MHA) · Spring read/write 데이터소스 라우팅

## 13. 일관성 & CAP
CAP 정리 · 강한 일관성 vs 결과적 일관성(eventual) · read-after-write 문제(복제 지연 시 "방금 쓴 걸 못 읽음") 대응

---

## 🪜 학습 원칙

1. **4~7단계(인덱스·실행계획·커넥션풀·JPA)는 반드시 손으로.** 수십만 건 더미 데이터를 넣고 `EXPLAIN ANALYZE`로 인덱스 유무에 따라 `rows`·실제 시간이 어떻게 바뀌는지 눈으로 확인.
2. **정규화로 정확성 확보 → 측정된 병목에 한해 비정규화.**
3. 격리수준·MVCC는 "표 암기"보다 **내가 쓰는 DB의 실제 구현 확인.**

## 🔗 연결
- MSA 실습 합류점 → [[phase2-week-breakdown|Phase 2 (DB 실전)]] · [[msa-roadmap]]
- 별개 트랙: [[인증 로드맵]]

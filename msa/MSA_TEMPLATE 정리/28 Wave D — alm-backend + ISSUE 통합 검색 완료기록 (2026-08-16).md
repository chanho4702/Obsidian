---
tags: [msa, wave-d, alm-backend, opensearch, redis-streams, graphql, grpc]
작성일: 2026-08-16
상태: 구현·로컬 E2E 완료, 기능 브랜치 푸시
설계: [[2026-08-16-wave-d-alm-backend-search-design]]
계획: [[2026-08-16-wave-d-alm-backend-search]]
---

# 28 — Wave D: alm-backend + ISSUE 통합 검색 완료기록

## 결과

Wave D는 localStorage에만 있던 ALM에서 **프로젝트·이슈 최소 종단 슬라이스**를 서버 정본으로
분리하고, 기존 Wiki 검색과 같은 Redis Streams/OpenSearch 파이프라인에 `ISSUE`를 합류시켰다.

```
JWT → Gateway /api/alm/** → alm-backend(:9120) → PostgreSQL almdb
                              │
                              ├─ gRPC → org-service(:9131) PROJECT 권한
                              ├─ XADD → Redis platform:events:v1
                              └─ gRPC(:9121) ← search-service 원문 조달

search-service → OpenSearch aliases: wiki-page + wiki-attachment + alm-issue
               → GraphQL SPACE/PROJECT 권한 분기 검색
```

## 구현 범위

- 독립 `alm-backend` repo: Java 24, Spring Boot 4.0.6, PostgreSQL/Flyway
- 프로젝트 CRUD와 생성자 PROJECT ADMIN grant
- 이슈 CRUD, 프로젝트별 순번·불변 `issueKey`, `expectedVersion` 충돌 409
- Project/Issue 6종 커밋 후 Redis Streams 이벤트
- `AlmContentService` 단건/전량 gRPC 조달
- `alm-issue` Nori strict mapping, 외부 이벤트 시각 버전 upsert/delete
- SPACE와 PROJECT grant를 한 번에 읽는 통합 검색 권한
- GraphQL `DocType.ISSUE`, `projectIds`, 이슈 메타 필드
- 세 물리 인덱스를 모두 백필한 뒤 세 별칭을 한 호출로 원자 전환
- Gateway, `almdb`, Compose, healthcheck, GHCR/deploy 매핑

이번 슬라이스에서 제외한 것: 보드·스프린트·댓글·워크로그·스킴 전체 서버 이관과 alm-front의
localStorage→REST 전환. 프론트 기능을 억지로 축소하지 않고 다음 슬라이스에서 경계 어댑터로 옮긴다.

## 권한과 장애 의미

- REST는 org-service `PROJECT` VIEW/EDIT/ADMIN으로 인가한다.
- 검색은 `ListUserGrants` 한 번으로 SPACE/PROJECT 집합을 만들고 도메인별 bool 분기를 만든다.
  한쪽 grant가 0건이어도 다른 도메인 검색은 유지된다.
- org-service 가용성 장애는 503, 그 외 판정 오류는 fail-closed다.
- ALM gRPC NOT_FOUND는 삭제로 색인에서 제거하고, 전송 장애는 이벤트를 ACK하지 않아 재시도한다.
- Redis 발행 실패는 정본 트랜잭션을 롤백하지 않고 ERROR로 남기며 관리자 재색인으로 복구한다.

## 검증

| 대상 | 결과 |
|---|---|
| alm-backend | 9/9 green, H2 MVC + in-process gRPC + 실제 PostgreSQL Flyway/cascade |
| platform-backend | org 36 + search 69 green, 전체 `gradlew build` 통과 |
| gateway-server | 27/27 green |
| Compose | 16개 전부 running, healthcheck 대상 전부 healthy |
| 스모크 | 정적 배선 + 무인증 `/api/alm/**` 401 + 전체 라이브 PASS |
| ALM 양성 E2E | 관리자 JWT → 프로젝트/이슈 생성 → `ISSUE` 1 hit |
| 권한 음성 E2E | grant 없는 사용자 같은 검색 0 hit |
| 소비자 정합성 | stream 3, pending 0, DLQ 0 |
| 정리 | 테스트 프로젝트 삭제 후 `alm-issue` 잔존 문서 0 |

## 형상화 단위

- `platform-backend`: common-proto 0.6.0 + search-service ALM 확장
- `alm-backend`: 신규 독립 private repo
- `gateway-server`: `/api/alm/**` 라우트
- `infra-settings`: compose/DB/스모크/배포 매핑

## 다음 단계

1. alm-front 도메인 store를 서버 경계 어댑터로 전환
2. PROJECT 권한 관리 UI와 프로젝트 선택 기반 전역 검색 UX
3. Wave E notification-service
4. 공통 후속: transactional outbox, gRPC 서비스 인증, 재색인 dual-write/이벤트 재생

← [[27 MSA_TEMPLATE ALM-WIKI 프로젝트 산출물 — Notion 게시본 (2026-08-16)]] · [[00 개요 — 전체 구조]]

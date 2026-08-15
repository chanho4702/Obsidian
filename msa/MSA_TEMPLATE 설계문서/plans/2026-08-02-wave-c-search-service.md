---
tags: [msa, template, wave-c, search, plan]
작성일: 2026-08-02
상태: 완료 (2026-08-15)
spec: [[2026-08-02-wave-c-search-service-design]]
결과: [[25 Wave C — search-service 구현·배포 완료기록 (2026-08-15)]]
---

# Wave C 실행 계획 — search-service

repos: `platform-backend`(T1·T3·T5~T9·T11) / `wiki-backend`(T2) / `gateway-server`·우산 `MSA_TEMPLATE`(T4·T10·T11)

모든 태스크를 **테스트 그린 + 교차검증**으로 닫았다. 최종적으로 platform-backend·wiki-backend·gateway-server·infra-settings 4개 repo에 반영되었고, GHCR 이미지 푸시→`repository_dispatch`→self-hosted 배포까지 완주했다.

## 착수 전제·마감 판정

- [x] 소비·색인 경로는 Testcontainers(Redis + Nori OpenSearch)와 Docker 풀스택으로 검증
- [ ] 호스트 dev 오프셋 클러스터의 유효 JWT 검색(200 + hits) 실측은 후속 검증으로 이관
  - 네이티브 Redis 3.2가 `localhost:6379`를 선점했던 환경 문제는 배포판 풀스택 완주를 막지 않았다. search-service dev는 발행자와 맞추기 위해 Redis DB 1로 고정했다.

## 태스크

| # | 상태 | 태스크 | repo | 결과 |
|---|---|---|---|---|
| T1 | ✅ | proto v0.4.0 — `WikiContentService` | platform-backend | v0.4.0 발행 후 실구현 공백을 v0.5.0으로 보강 |
| T2 | ✅ | wiki-backend gRPC 서버 | wiki-backend | :9111, 단건·스트림 조달, CI green |
| T3 | ✅ | search-service 모듈 골격 | platform-backend | Boot·Security·Dockerfile·health 구성 |
| T4 | ✅ | OpenSearch + Nori 인프라 | 우산 + platform-backend | 내부망 전용, 인덱스·별칭 멱등 부트스트랩 |
| T5 | ✅ | 색인 도메인 | platform-backend | external version 순서 역전 방어 + 실 OpenSearch 검증 |
| T6 | ✅ | Redis Streams 이벤트 소비자 | platform-backend | PEL delivery count·XCLAIM·Lua DLQ+XACK·멱등 처리 |
| T7 | ✅ | `ListUserGrants` 권한 필터 | platform-backend | GLOBAL 분기·SPACE 교집합·fail-closed |
| T8 | ✅ | GraphQL 검색 API | platform-backend | 인증·깊이 4·복잡도 50·size 100·구조화 접근 로그 |
| T9 | ✅ | 관리자 백필/재색인 | platform-backend | GLOBAL ADMIN, 단일 비동기 잡, 2별칭 원자 전환 |
| T10 | ✅ | 게이트웨이·compose·스모크 | gateway-server, 우산 | `StripPrefix=2`, 15/15 컨테이너, 검색 3경로 401 |
| T11 | ✅ | GHCR·dispatch·배포 매핑 | platform-backend, 우산 | CI run 31818405300 + 배포 31818716588/31818717983 성공 |
| T12 | ✅ | README·원장·교차검증 | 전체 | 문서 단언 16/16 + 최종 리뷰 PASS |

## 순서 의존

```
T1 ─┬─▶ T2
    └─▶ T3 ─▶ T4 ─▶ T5 ─▶ T6 ─┐
              T7 ──────────────┼─▶ T8 ─▶ T9 ─▶ T10 ─▶ T11 ─▶ T12
```

T2와 T3~T4는 병렬 가능. T7은 T3 이후 언제든.

## 확정·반영된 되돌리기 어려운 지점

- proto **v0.4.0·v0.5.0 태그 발행 완료** — 발행본은 회수하지 않는다
- compose에 OpenSearch·search-service 편입 완료 — 전체 **15컨테이너**, OpenSearch 호스트 포트 미개방
- `deploy.yml` 반영 완료 — search-service 정확 일치 검증·태그·컨테이너 매핑·healthy 대기 포함

## 기록 규약

- 진행 중 기술 원장은 기존 `C:\MSA_TEMPLATE\.superpowers\sdd\progress.md` Wave C 섹션에 누적했다.
- 장기 보관 정본과 최종 정리본은 **이 Obsidian 보관함**이다. 리포 `docs/`에 작업기록 문서를 새로 만들지 않는다.

## 현행 갭

- 내부 gRPC 채널 인증 없음(호스트 포트 미개방 전제)
- 재색인 중 이벤트 dual-write 없음 — 별칭 전환 전후 스테일 창 가능
- 유효 JWT로 실제 200 + hits 응답, dev 오프셋 search-service 기동은 아직 미검증
- wiki-front 통합 검색 UI와 ALM 색인은 다음 웨이브

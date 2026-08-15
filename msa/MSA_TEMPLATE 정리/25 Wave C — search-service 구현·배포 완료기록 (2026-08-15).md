---
tags: [msa, template, wave-c, search-service, opensearch, redis-streams, graphql, grpc, cicd]
작성일: 2026-08-15
상태: 구현·배포 완료
repos: [platform-backend, wiki-backend, gateway-server, infra-settings]
spec: [[2026-08-02-wave-c-search-service-design]]
plan: [[2026-08-02-wave-c-search-service]]
---

# 25 — Wave C: search-service 구현·배포 완료기록 (2026-08-15)

상위: [[00 개요 — 전체 구조]] · 선행: [[17 Wave B — wiki-backend 구현기록, 과정과 원리 해설 (2026-07-21)]] · 평가: [[24 현재 아키텍처·구현 평가 + 개선 백로그 (2026-08-03)]]

## 결론

Wave C는 **위키 변경을 Redis Streams로 받아 OpenSearch에 색인하고, 사용자가 볼 수 있는 스페이스만 GraphQL로 검색하는 횡단 search-service**를 추가한 웨이브다. 색인·검색에서 끝나지 않고, 관리자 재색인, Gateway 라우트, Docker Compose, 스모크, GHCR 이미지, self-hosted runner 자동배포까지 닫았다.

**T1~T12 전부 완료.** platform-backend CI→org/search 각각 dispatch→로컬 재배포가 실제로 성공했고, 최종 스모크에서 15개 서비스가 전부 기동했다.

> [!important] 이번 범위
> 백엔드·인프라·배포까지다. wiki-front 통합 검색 화면과 ALM 색인은 후속 웨이브다.

## 1. 완성된 토폴로지

```
wiki-backend
  └─ XADD platform:events:v1
       └─▶ Redis Streams
             └─ XREADGROUP (group=search-service)
                  └─▶ search-service(:9140)
                        ├─ gRPC → wiki-backend(:9111)  본문·첨부 조달
                        ├─ gRPC → org-service(:9131)   사용자 권한 범위
                        └─ HTTP → OpenSearch(:9200)   색인·질의

브라우저 → nginx(:80) → gateway(:8000)
  └─ /api/search/** --StripPrefix=2--▶ /graphql, /admin/reindex
```

- 물리 인덱스: `wiki-page-v1`, `wiki-attachment-v1`
- 읽기·일반 쓰기 경계: 별칭 `wiki-page`, `wiki-attachment`
- 스트림: `platform:events:v1`, DLQ `platform:events:v1:dlq`
- 외부 API: `POST /api/search/graphql`
- 운영 API: `POST /api/search/admin/reindex`, `GET /api/search/admin/reindex/{jobId}`

search-service는 RDB를 갖지 않는다. PostgreSQL의 wiki 데이터가 정본이고 OpenSearch는 재색인 가능한 파생 데이터다.

## 2. 계약 — proto v0.4.0에서 v0.5.0까지

처음에는 `WikiContentService` 3개 RPC로 v0.4.0을 발행했다.

- `GetPageContent`
- `ListPageContents` (server streaming)
- `ListAttachments` (server streaming)

wiki-backend에 실제로 붙이자 초안으로 닫히지 않는 경계가 두 개 드러났다.

1. `AttachmentAdded` 이벤트만으로는 크기·업로더·스페이스 표시값을 채울 수 없었다. 이벤트마다 첨부 전량 스트림을 훑는 것도 불가했다.
2. `GetPageContent` 실패를 모두 삭제로 치면, 참조가 깨진 고아 페이지를 조용히 색인에서 지우게 된다.

그래서 v0.5.0에 `GetAttachmentMeta`를 추가하고 `AttachmentMeta` 안에 `space_key`·`space_name`을 비정규화했다. 페이지 조달은 `NOT_FOUND=삭제로 간주·ACK`, `FAILED_PRECONDITION=고아 상태·재시도/DLQ`로 분리했다.

> [!tip] 버전 발행 순서 규약
> `common-proto` 태그 발행 CI가 끝나기 전에 소비자 repo를 푸시하면 GitHub Packages에 버전이 아직 없어 CI가 실패한다. **패키지 발행 완료 → 소비자 푸시** 순서를 지킨다.

## 3. 색인 정합성과 Redis Streams 소비

이 서비스는 플랫폼의 **첫 Redis Streams 소비자**다.

- consumer group `search-service`를 `MKSTREAM`과 함께 멱등 생성
- 처리를 끝낸 뒤에만 XACK
- 재시도 횟수는 별도 메모리가 아니라 **PEL delivery count**에서 읽음
- 재기동으로 consumer name이 달라져도 `XCLAIM`으로 pending 회수
- 재시도 상한(기본 5) 초과 시 **DLQ 이동 + 원본 XACK을 Lua 한 번으로 원자화**
- 문서 `_id`를 도메인 키로 고정해 중복 소비 멱등성 확보
- `version_type=external`, `version=occurred_at` epoch millis로 뒤늦게 온 이벤트 거부

`VERSION_CONFLICT` 또는 중복·과거 이벤트는 정상 흐름으로 ACK한다. `PageDeleted`가 첨부를 연쇄 삭제하는 것은 페이지 삭제가 실제 `APPLIED`일 때만 한다. 과거 삭제 이벤트가 최신 페이지의 첨부까지 지우는 사고를 막기 위함이다.

dev에서는 wiki-backend가 Redis DB 1에 발행하므로 search-service도 `application-dev.yml`에서 DB 1을 쓴다. 둘이 다르면 소비자는 빈 스트림을 정상으로 오인하고, 표면상 장애 없이 검색만 스테일해진다.

## 4. 권한 필터 GraphQL

검색 권한은 색인에서 문서를 물리적으로 분리하지 않고, 질의 시점에 서버가 적용한다.

1. JWT `sub` → numeric `userId`
2. org-service `ListUserGrants(user_id)` 호출
3. GLOBAL grant가 있으면 스페이스 필터 없음
4. 없으면 SPACE grant만 골라 `terms(spaceId)` AND 필터
5. 클라이언트 `spaceIds`는 접근 가능 집합과 교집합을 취해 좁힐 수만 있음

`ListUserGrants` 요청에 `resource_type=SPACE`를 넣지 않는 것이 중요하다. SPACE로 먼저 좁히면 GLOBAL grant 보유자를 판별할 수 없다.

GraphQL은 실행 오류에 HTTP 200 + `errors`를 쓰는 경로가 있다. 그래서 org/OpenSearch 불능은 `extensions.code=SERVICE_UNAVAILABLE`, `httpStatus=503`으로 내보내 **빈 결과와 장애를 구분**했다. 질의 깊이 4, 복잡도 50, page size 100 상한을 걸고, 접근 로그에 operation name·오류 수·지연을 구조화 필드로 남긴다.

## 5. 원자 재색인

GLOBAL ADMIN만 재색인을 시작할 수 있다. 잡은 요청 스레드와 분리된 단일 실행기에서 돌고, 한 번에 하나만 수락한다.

```
다음 세대 페이지·첨부 인덱스 생성
  → wiki gRPC 전량 스트림
  → 500건 단위 bulk index
  → 두 인덱스 refresh
  → 페이지·첨부 별칭을 한 aliases 호출로 원자 전환
  → 구 인덱스 유지
```

둘 중 하나라도 실패하면 별칭을 건드리지 않는다. 실패한 새 인덱스는 진단을 위해 남기고 수동 삭제한다. 잡 상태는 메모리라 재기동하면 잃지만, 운영 조작을 다시 시작할 수 있어 수용했다.

## 6. Gateway·Compose·CI/CD

search-service 내부 경로는 `/graphql`·`/admin/reindex`이다. 외부에서는 `/api/search/**`로 통일하므로 Gateway의 search 라우트만 **`StripPrefix=2`**를 적용했다. org/wiki의 No StripPrefix 규약에서 유일한 예외다. 검색 비용을 제한하기 위해 rate limiter 5/15도 적용했다.

Docker에서 search-service는 Eureka에 등록하지 않고 Gateway가 `SEARCH_SERVICE_URI=http://search-service:9140`으로 직결한다. 이 환경변수가 빠지면 기본값 `lb://search-service`를 해석할 수 없어 검색 경로 전체가 503이 된다.

CI는 platform-backend 전체 `gradlew build` 후 org-service·search-service 이미지를 각각 `latest`·`sha-*`로 GHCR에 푸시하고, 서비스별 `repository_dispatch` 배포를 발사한다. 우산 `deploy.yml`의 허용 배열·서비스·태그 env·컨테이너 이름 6곳에 search-service를 정확 일치로 추가했다. 배포는 `--no-deps`로 대상만 교체하고 healthy를 대기한다.

## 7. 실측으로만 잡힌 결함

이 웨이브의 핵심 교훈은 **구현 존재와 실행 가능은 다르다**는 것이다.

| 결함 | 왜 정적 리뷰에서 안 보였나 | 보정 |
|---|---|---|
| external version 409가 장애로 처리됨 | 클라이언트가 `OpenSearchException`이 아닌 `ResponseException`을 던지는 실행 경로 | cause chain에서 두 타입의 409를 모두 식별 |
| dev 발행자 db1 / 소비자 db0 | 둘 다 연결 성공이라 표면상 정상 | search-service `application-dev.yml` db1 |
| 소비자 빈이 풀스택에서 생성 실패 | 테스트 프로필은 events bean을 아예 등록하지 않음 | 운영 생성자 `@Autowired` + 스프링 빈 생성 회귀 테스트 |
| PowerShell 5.1 스모크가 HTTP 검사를 SKIP하고 PASS | `-SkipHttpErrorCheck`가 PowerShell 6+ 전용 | 파라미터 존재 확인 + 5.1/6+ 예외에서 상태코드 추출 |
| `-RequireLive`가 11개만 세고 15개 전체로 표시 | nginx·Loki·Alloy·Grafana가 집계 대상에서 누락 | Compose가 선언한 전체 서비스를 기준으로 판정 |

또한 Windows 예약 포트(8000·8080)와 네이티브 Redis 6379 선점으로 풀스택 호스트 게시가 충돌했다. 검증 한정 인라인 Compose override로 해당 host publication만 제거하고 내부 토폴로지는 유지했다. 제품 Compose 파일은 보안 결정을 뒤집지 않았다.

## 8. 검증·배포 결과

| 게이트 | 결과 |
|---|---|
| search-service | 64/64 tests green, Redis + Nori OpenSearch Testcontainers 포함 |
| gateway-server | 27/27 tests green. search 라우트·`StripPrefix=2`·무인증 401 포함 |
| 정적 스모크 | Compose·healthcheck 의존·OpenSearch 호스트 미개방·Gateway/search env 배선 PASS |
| 풀스택 | 15/15 running, healthcheck 보유 서비스 전원 healthy |
| 인증 경계 | nginx 경유 검색 1개 + 재색인 2개 경로가 모두 401. board 양성 대조군 포함 |
| CI | platform-backend run `31818405300` success |
| 배포 | org-service `31818716588`, search-service `31818717983` success |
| 최종 문서 리뷰 | 소스 대조 단언 16/16 + Codex 최종 리뷰 PASS |

## 9. 현재 남은 갭

- **gRPC 인증 없음** — search→wiki/org, wiki→org 모두 내부망·호스트 미개방을 전제로 한다.
- **재색인 이벤트 창** — 백필 중 소비자는 구 별칭에만 쓴다. 신·구 dual-write가 없어 전환 직전 변경이 새 인덱스에서 잠깐 빠질 수 있다.
- **발행자 outbox 없음** — 소비자측 재시도·DLQ는 강하지만, wiki DB 커밋 후 Redis 발행이 실패하면 영구 누락이다. [[24 현재 아키텍처·구현 평가 + 개선 백로그 (2026-08-03)]] M-07.
- **dev gRPC 포트 예외** — wiki gRPC는 dev에서도 9111이라 +10000 규약과 어긋난다.
- **라이브 기능 검증 두 건 미완료** — 유효 JWT로 실제 200 + hits를 받는 것, dev 오프셋 클러스터에서 search-service를 기동하는 것.
- **프론트 미구현** — wiki-front 검색 화면과 ALM 도메인 색인은 다음 범위다.

## 10. 주요 커밋

| repo | 커밋 | 내용 |
|---|---|---|
| platform-backend | `56fdfd7`, `7d45dde` | wiki proto v0.4.0 → v0.5.0 |
| wiki-backend | `60ad753`, `6cbd582` | 색인 조달 gRPC + v0.5.0 계약 보강 |
| platform-backend | `0c95d15`, `872c76c` | search 모듈·색인 도메인·Redis 소비자 |
| platform-backend | `5b86b1f`, `3cc3ece` | 권한 GraphQL·관리자 재색인 |
| gateway-server | `f9ee96d` | search 라우트·인증 경계 |
| platform-backend | `cf209bb`, `7c41fcd` | 풀스택 기동 결함 수정·GHCR 배포 편입 |
| infra-settings | `133d269`, `6da91d1`, `364a480` | 스모크·배포 매핑·운영 문서 마감 |

## 11. 다음에 이어갈 것

1. 유효 JWT를 사용한 실제 검색 200 + hits E2E
2. wiki-front 통합 검색 UI
3. Wave D ALM 이벤트·색인 확장
4. 재색인 dual-write/이벤트 재생 정책
5. DB outbox→Redis Streams 재전송, gRPC 서비스 인증

← [[00 개요 — 전체 구조]]

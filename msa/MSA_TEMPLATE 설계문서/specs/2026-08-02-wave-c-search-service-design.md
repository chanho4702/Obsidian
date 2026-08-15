---
tags: [msa, template, wave-c, search, opensearch, graphql, redis-streams, grpc, 설계]
작성일: 2026-08-02
상태: 구현·배포 완료 · as-built 보정 (2026-08-15)
관련: [[15 ALM·Wiki 백엔드 요구사항 + 서비스 분할 설계 (2026-07-19)]] · [[17 Wave B — wiki-backend 구현기록, 과정과 원리 해설 (2026-07-21)]]
결과: [[25 Wave C — search-service 구현·배포 완료기록 (2026-08-15)]]
---

# Wave C — search-service 설계 (2026-08-02)

> [!success] 구현 결과
> 2026-08-15에 T1~T12가 모두 완료됐다. 아래는 초기 설계 의도를 보존하되, 구현 중 확정된 proto v0.5.0·GraphQL 오류 계약·Redis PEL/DLQ·재색인 이벤트 창을 as-built로 반영한다.

## 1. 배경과 목표

15번 문서 B안의 횡단 서비스 2호. wiki-backend가 Redis Streams로 내보내는 도메인 이벤트를 소비해
검색 색인을 만들고, 권한이 걸린 통합 검색을 제공한다.

착수 당시 상태:

- `wiki-backend`에 **검색 기능이 없다**(소스 전체 grep 0건). `wiki-front`의 "검색"은 이미 받아온 페이지 트리를 클라이언트에서 필터링하는 것뿐이라, 본문은 검색 대상이 아니다.
- 이벤트 발행은 Wave B에서 완성됐고, 08-02에 고장(네이티브 Redis 3.2 오접속)을 잡았다. **소비자는 아직 하나도 없다** — 이 웨이브가 첫 소비자다.
- `org.proto`의 `ListUserGrants`에는 이미 `// 검색 결과 권한 필터용(Wave C)` 주석이 달려 있다. 권한 필터 계약은 **신규 설계가 아니라 예약된 계약의 소비**다.

목표: 위키 본문·제목·첨부 파일명을 대상으로 한 전문 검색을, 사용자가 볼 수 있는 스페이스로 한정해서 돌려준다.

## 2. 확정 결정 (2026-08-02 대화)

| 항목 | 결정 | 근거 |
|---|---|---|
| 검색 엔진 | **OpenSearch 2.x** | 최종 목표가 온프렘 설치·운영 배포판이다. Apache 2.0이라 동봉 배포에 라이선스 검토가 필요 없다. Nori 한국어 분석기 동일 지원, ES API 호환. 15번 문서의 "Elasticsearch"는 엔진 범주 표기로 해석 |
| 검색 API | **GraphQL** (Spring for GraphQL) | 15번 문서 §통신 매트릭스의 원 설계 유지. 통합 검색은 필터 조합·필드 선택이 요구의 본질이고, Wave D에서 ALM이 합류하면 다중 도메인 조합 조회가 실제로 필요해진다 |
| 이번 범위 | **백엔드 끝까지** | 이벤트 소비→색인, 권한 필터 검색 API, 백필/재색인, compose·CI 편입. 프론트 검색 화면은 다음 웨이브 |
| 본문 조달 | **wiki-backend에 gRPC 서버 신설** (proto v0.4.0 `wiki.v1`) | 아래 §4 |
| 저장소 | search-service는 **RDB를 갖지 않는다** | 색인은 OpenSearch가 소유하고, 원본은 wiki-backend가 소유한다. 재색인으로 언제든 복구 가능한 파생 데이터뿐이라 별도 DB를 둘 이유가 없다 |

## 3. 토폴로지

```
wiki-backend ──XADD──▶ redis stream platform:events:v1
                             │
                             │ XREADGROUP (group: search-service)
                             ▼
                      search-service ──gRPC──▶ wiki-backend :9111   (본문 조달)
                        :9140 REST/GraphQL ──gRPC──▶ org-service :9131 (권한 필터)
                             │
                             ▼
                       OpenSearch :9200
                        wiki-page-v1 / wiki-attachment-v1
```

포트: org 9130(REST)/9131(gRPC), wiki 9110(REST)/**9111(gRPC 신설)**, **search 9140**, OpenSearch 9200.
dev는 org 19130/19131, wiki REST 19110, search 19140을 쓴다. **wiki gRPC는 dev에서도 9111**이라 +10000 규약의 현재 예외다. 배포 스택의 OpenSearch 9200은 호스트에 게시하지 않는다.

## 4. 본문 조달 — 왜 gRPC인가

`EventEnvelope`는 본문을 싣지 않는다. proto 주석에 못 박혀 있다:

> 본문(content)은 싣지 않는다 — 소비자는 "무엇이 변했나"만 알고, 내용은 소유 서비스 API로 가져간다.

이 원칙은 유지한다(큰 페이지가 스트림에 그대로 실리면 Redis 메모리와 재생 비용이 본문 크기에 비례해 커진다).
그래서 `PageCreated`/`PageUpdated`를 받으면 search-service가 wiki-backend에서 본문을 따로 가져와야 한다.

검토한 세 갈래:

- **REST 재사용** — `GET /api/wiki/pages/{id}`는 호출자 JWT로 권한을 검사한다. search-service는 사용자가 아니라 시스템 주체라서 통과할 토큰이 없다. 서비스 계정 토큰 발급 장치가 플랫폼에 없다(auth-server는 사용자 토큰만 낸다)
- **이벤트에 본문 포함** — 위 원칙을 뒤집는다. 되돌리기 비싼 계약 변경
- **gRPC 신설** ★ — 15번 문서 §통신 매트릭스가 "서비스 간 동기 호출 = gRPC"로 이미 정한 자리다. org-service의 gRPC 서버 패턴을 그대로 복제할 수 있고, proto 계약으로 타입이 고정된다

→ `platform/wiki/v1/wiki.proto` 신설, **proto v0.4.0** 발행. 구현 중 단건 첨부 조달과 상태 계약 공백이 드러나 **v0.5.0**으로 보강했다.

```proto
service WikiContentService {
  // 색인용 단건 조달 — 권한 검사 없음(내부 전용). 삭제됐으면 NOT_FOUND
  rpc GetPageContent(GetPageContentRequest) returns (PageContent);
  // 이벤트가 담지 않는 크기·업로더·스페이스 표시값 단건 조달(v0.5.0)
  rpc GetAttachmentMeta(GetAttachmentMetaRequest) returns (AttachmentMeta);
  // 백필용 전량 스트리밍 — 페이지 커서, 스페이스 한정 가능
  rpc ListPageContents(ListPageContentsRequest) returns (stream PageContent);
  // 첨부 백필
  rpc ListAttachments(ListAttachmentsRequest) returns (stream AttachmentMeta);
}
```

`GetPageContent`의 실패 의미도 v0.5.0에서 분리했다. `NOT_FOUND`는 이미 삭제된 페이지라 소비자가 삭제로 간주해 ACK하고, `FAILED_PRECONDITION`은 참조가 깨진 고아 페이지라 재시도·DLQ로 보낸다.

> [!warning] 인계되는 알려진 갭
> gRPC 채널에 인증이 없다. wiki-backend → org-service 호출도 같은 상태이고, Wave A 최종리뷰가 "gRPC 인증"을 defer로 남겼다. 이번 웨이브가 새로 만드는 위험이 아니라 **기존 defer의 표면이 하나 늘어나는 것**이다. 컨테이너 내부망에서만 노출하고(호스트 포트 미개방), defer 항목에 이 경로를 추가한다.

## 5. 색인 모델

인덱스 2개 + 읽기 별칭. 별칭을 두는 이유는 무중단 재색인 — 새 인덱스에 다 채운 뒤 별칭만 원자적으로 옮긴다.

| 인덱스 | 별칭 | 문서 |
|---|---|---|
| `wiki-page-v1` | `wiki-page` | 페이지 1건 = 문서 1건, `_id = {page_id}` |
| `wiki-attachment-v1` | `wiki-attachment` | 첨부 1건 = 문서 1건, `_id = {attachment_id}` |

질의는 두 별칭을 함께 지정한다(`wiki-page,wiki-attachment/_search`). Wave D의 ALM 인덱스는 별칭 추가로 붙인다.

**`wiki-page-v1` 매핑 요지**

| 필드 | 타입 | 비고 |
|---|---|---|
| `pageId` / `spaceId` / `authorId` | `long` | |
| `spaceKey` | `keyword` | |
| `spaceName` | `text`(nori) + `keyword` 서브필드 | 스페이스명 검색·집계 양쪽 |
| `title` | `text`(nori) + `keyword` | 제목 가중치 ↑ |
| `content` | `text`(nori) | 마크다운 원문. 하이라이팅 대상 |
| `type` | `keyword` | `PAGE` / `FOLDER` |
| `status` | `keyword` | `DRAFT` / `PUBLISHED` |
| `version` | `integer` | |
| `updatedAt` | `date`(epoch millis) | 정렬·외부 버전 |

한국어 분석기 **Nori**는 OpenSearch 기본 이미지에 없다 → `analysis-nori` 플러그인을 설치한 커스텀 이미지를 만든다(`infra/opensearch/Dockerfile`).

폴더(`type=FOLDER`)와 초안(`status=DRAFT`)도 색인한다. 걸러내는 건 질의 단계의 기본 필터로 두어, 나중에 "초안 포함 검색" 옵션을 켜는 게 재색인 없이 가능하게 한다.

## 6. 이벤트 소비

- Redis Streams consumer group **`search-service`**, `XREADGROUP` → 처리 → `XACK`.
- 처리 실패는 **XACK하지 않는다** → PEL의 delivery count로 재시도 횟수를 판정한다. 재기동 후 consumer name이 바뀌어도 `XCLAIM`으로 pending을 회수한다.
- 기본 5회 초과 시 `platform:events:v1:dlq`로 옮기고 원본을 XACK하는 두 동작은 **Lua 한 번**으로 원자화한다.
- **멱등**: 문서 `_id`가 도메인 키라 같은 이벤트를 두 번 받아도 같은 결과가 된다.
- **순서 역전 방어**: `version_type=external`, `version = occurred_at`(epoch millis)을 쓴다. 늦게 도착한 오래된 이벤트는 OpenSearch가 `version_conflict`로 거부하고, 그건 정상 흐름으로 취급해 XACK한다. `PageUpdated.version`은 도메인 버전이라 이벤트 순서 판정에는 쓸 수 없다(스페이스·첨부 이벤트엔 아예 없다).

**이벤트별 처리**

| 이벤트 | 처리 |
|---|---|
| `PageCreated` / `PageUpdated` | gRPC로 본문 조달 → upsert. 조달이 NOT_FOUND면 이미 삭제된 것 → 삭제로 간주하고 XACK |
| `PageDeleted` | `_id` 삭제 + 그 페이지의 첨부 `delete_by_query(pageId)` |
| `SpaceCreated` | 무동작(색인할 문서가 아직 없다). 스페이스 표시명은 페이지 문서에 비정규화돼 들어간다 |
| `SpaceUpdated` | `update_by_query`로 그 스페이스 문서들의 `spaceName`·`spaceKey` 갱신 |
| `SpaceDeleted` | 두 인덱스 모두 `delete_by_query(spaceId)` |
| `AttachmentAdded` | 첨부 문서 upsert |
| `AttachmentDeleted` | 첨부 `_id` 삭제 |

> [!note] Wave C 선행이 남긴 경계 그대로 따른다
> `AttachmentDeleted`는 **단건 삭제에만** 발행된다. 페이지·스페이스 삭제로 딸려간 첨부는 상위 `PageDeleted`/`SpaceDeleted`에서 소비자가 함께 지운다 — 위 표의 `delete_by_query`가 그 몫이다.

> [!important] 08-02 고장에서 가져온 교훈 반영
> 색인 실패는 화면이 멀쩡해서 티가 안 난다. 12일간 이벤트가 통째로 유실된 걸 몰랐던 이유가 정확히 그거였다. 그래서 ① 색인 실패는 WARN이 아니라 **ERROR** ② 기동 시 OpenSearch 연결·인덱스를 확인하고 준비되지 않으면 **기동 중단(fail-fast)** ③ 소비 경로에 **Testcontainers 통합 테스트**를 깐다(페이크만으로 검증하지 않는다 — 그게 `RedisStreamEventPublisher`가 12일간 한 번도 실행 안 된 이유였다).

## 7. 권한 필터

검색 요청의 JWT `sub` → `userId` → org-service `ListUserGrants(user_id)` gRPC 호출
→ GLOBAL grant 보유자면 필터 없음, 아니면 SPACE grant만 골라 `terms { spaceId: [...] }` 필터를 질의에 AND로 건다. 요청에 `resource_type=SPACE`를 미리 걸면 GLOBAL 보유자를 판별할 수 없어 전체 grant를 요청하는 것으로 확정했다.

- **org-service 불능 시 503 의미 전파**(fail-closed). GraphQL은 HTTP 200 + `extensions.httpStatus=503`으로 표현한다. 권한을 모르는 상태에서 빈 결과를 주면 "검색해도 안 나오네"로 조용히 오인된다.
- 필터는 서버에서 건다. 클라이언트가 보낸 `spaceIds`는 **접근 가능 집합과 교집합**을 취한다(추가 좁히기만 가능, 넓히기 불가).
- grant 조회 캐시는 이번 범위에서 제외한다(요청당 1회 gRPC). 필요해지면 짧은 TTL 캐시를 나중에.

GraphQL 실행 오류는 HTTP 200을 유지하되, 권한 서비스·검색 엔진 불능은 `errors[].extensions.code=SERVICE_UNAVAILABLE`, `httpStatus=503`으로 빈 결과와 구분한다. 질의 깊이 4, 복잡도 50, size 100 상한을 적용했다.

## 8. GraphQL 계약

엔드포인트 `POST /graphql`, 게이트웨이 라우트 `Path=/api/search/**` → search-service.

```graphql
type Query {
  search(input: SearchInput!): SearchResults!
}

input SearchInput {
  query: String!
  spaceIds: [ID!]           # 접근 가능 집합과 교집합
  docTypes: [DocType!]      # 기본: 전체
  includeDrafts: Boolean = false
  page: Int = 0
  size: Int = 20            # 상한 100
}

type SearchResults { total: Int!, tookMs: Int!, hits: [SearchHit!]! }

type SearchHit {
  id: ID!
  docType: DocType!
  spaceId: ID!
  spaceKey: String!
  spaceName: String!
  pageId: ID              # ATTACHMENT면 소속 페이지
  title: String           # PAGE
  filename: String        # ATTACHMENT
  highlights: [String!]!  # title·content 하이라이트 조각
  updatedAt: String
  score: Float!
}

enum DocType { PAGE, ATTACHMENT }
```

게이트웨이는 이 경로에 rate limiter를 건다 — 검색은 비용이 큰 질의라 보호 대상이다.

> [!note] GraphQL을 붙이면서 챙겨야 하는 것
> 플랫폼 규약(인증·라우팅·rate-limit·구조화 로그)이 전부 REST 경로 기준으로 짜여 있다. GraphQL은 단일 URL이라 경로 기반 규약이 그대로 안 먹는다. 최소한 ① Security는 `/graphql` 전체를 인증 필수로 ② 질의 깊이·복잡도 상한 ③ 접근 로그에 operation name이 남도록 — 셋은 이 웨이브에서 같이 넣는다.

## 9. 백필 / 재색인

관리자 전용 REST(GraphQL 아님 — 운영 조작이라 curl로 때릴 수 있어야 한다):

- `POST /api/search/admin/reindex` → 새 페이지·첨부 인덱스 `v{n+1}` 쌍 생성 → wiki-backend gRPC 스트림으로 전량 색인 → 두 색인이 모두 성공한 뒤 별칭을 **한 호출로 원자 전환** → 구 인덱스 유지(수동 삭제)
- `GET /api/search/admin/reindex/{jobId}` → 진행 상태

비동기 잡은 한 번에 하나만 수락하고, 상태는 메모리에 둔다(재기동하면 잃는다 — 재색인은 다시 돌리면 되는 조작이라 수용).
권한: org-service GLOBAL ADMIN.

> [!warning] as-built 갭 — 재색인 이벤트 창
> 이벤트 소비자는 재색인 중에도 현재 별칭(구 인덱스)에만 쓴다. 신·구 dual-write가 없어 전량 조달 후 별칭 전환 전에 들어온 변경이 새 인덱스에서 잠깐 빠질 수 있다. 후속 과제다.

## 10. 운영

- compose에 `opensearch`(커스텀 nori 이미지) + `search-service` 추가. 전체 스택은 **15컨테이너**다. OpenSearch는 단일 노드, `discovery.type=single-node`, 힙 512m(M 티어 dev 기준).
- 보안 플러그인은 비활성(`DISABLE_SECURITY_PLUGIN=true`) — 내부망 전용, 호스트 포트 미개방. 온프렘 배포판에서 켜는 건 별도 과제로 남긴다.
- 볼륨 `platform-opensearch-data`.
- 로그는 stdout ECS JSON + Alloy 수집(기존 규약 그대로).
- CI: platform-backend repo의 기존 GHCR push·dispatch 패턴 복제, `deploy.yml` 서비스 매핑에 `search-service` 추가(정확 일치 배열).

## 11. 실패 모드

| 무엇이 죽으면 | 무슨 일이 나나 | 설계된 반응 |
|---|---|---|
| OpenSearch | 검색 불가, 색인 실패 | GraphQL `SERVICE_UNAVAILABLE`(`httpStatus=503`). 소비자는 XACK 안 하고 pending 유지 → 복구 시 자동 따라잡음 |
| wiki-backend gRPC | 본문 조달 실패 | 재시도 → 초과 시 DLQ + ERROR. 검색은 기존 색인으로 계속 동작 |
| org-service | 권한 판정 불가 | GraphQL `SERVICE_UNAVAILABLE`(`httpStatus=503`, fail-closed) |
| Redis | 이벤트 수신 정지 | 색인이 멈춘다(검색은 스테일 상태로 계속 동작). 기동 시 스트림 지원 확인 로그는 wiki와 동일 규약 |
| search-service | 검색 불가 | wiki·org 코어는 무영향 — 15번 문서의 "소규모 설치에서 꺼도 코어는 동작" 요건 |

## 11.1 마감 검증 (2026-08-15)

- search-service 64/64, gateway 27/27 테스트 통과
- Docker 풀스택 15/15 running, healthcheck 보유 서비스 전원 healthy
- nginx→gateway 경유 검색·재색인 3경로 무인증 401 실측(PowerShell 5.1 포함)
- platform-backend CI run `31818405300`, org/search 배포 run `31818716588`·`31818717983` 성공
- 유효 JWT 200 + hits와 dev 오프셋 search-service 실행은 미검증 갭으로 유지

## 12. 이번 범위 밖

- wiki-front 검색 화면 (다음 웨이브)
- ALM 색인 (Wave D)
- notification-service (Wave E)
- Postgres FTS 폴백 모드 (S 티어 옵션 — 확장 경로만 명시)
- gRPC 채널 인증 (플랫폼 전역 defer)
- grant 조회 캐시, 검색어 로깅·인기검색어, 유사문서 추천

---
tags: [msa, wave-d, alm, search, redis-streams, opensearch, grpc, graphql]
작성일: 2026-08-16
상태: 구현 착수
관련: [[15 ALM·Wiki 백엔드 요구사항 + 서비스 분할 설계 (2026-07-19)]] · [[25 Wave C — search-service 구현·배포 완료기록 (2026-08-15)]] · [[26 wiki-front 통합 검색 UI + 라이브 JWT E2E (2026-08-15)]]
---

# Wave D — alm-backend 최소 종단 슬라이스 + 통합 검색 확장

## 1. 문제와 목표

현재 `alm-front`는 245개 테스트가 있는 완성도 높은 localStorage 제품이지만 서버 정본이 없고, `alm-backend` 저장소도 아직 존재하지 않는다. search-service는 위키만 색인하므로 “통합 검색”이라는 이름과 달리 ALM 이슈를 함께 찾을 수 없다.

이번 Wave D의 목표는 ALM 전체 기능을 한 번에 서버화하는 것이 아니라, 다음 흐름을 실제 서비스 경계로 한 번 완주하는 것이다.

`JWT 사용자 → 프로젝트/이슈 REST 쓰기 → PostgreSQL 커밋 → Redis Streams 이벤트 → search-service gRPC 조달 → OpenSearch alm-issue 색인 → org PROJECT 권한 필터 GraphQL 검색`

## 2. 성공 기준

- 유효 JWT로 프로젝트와 이슈를 생성·수정·삭제할 수 있고 PROJECT 권한이 서버에서 강제된다.
- 커밋된 이슈 변경이 `platform:events:v1`을 통해 `alm-issue` 별칭에 반영된다.
- 같은 GraphQL 검색에서 위키 PAGE/ATTACHMENT와 ALM ISSUE를 함께 조회할 수 있다.
- PROJECT grant가 없는 사용자는 해당 이슈를 검색 결과로 볼 수 없다.
- 관리자 재색인이 wiki 2개 + ALM 1개 인덱스를 모두 채운 뒤 별칭을 원자 전환한다.
- 신규/변경 서비스의 Gradle 테스트, 컨테이너 기동, Gateway 인증 경계와 실제 검색 E2E가 통과한다.

## 3. 이번 범위

### Must

1. 독립 저장소 `alm-backend` 골격(Spring Boot 4.0.6, Java 24, PostgreSQL, Flyway, JWT/JWKS, org gRPC, Redis Streams, 내부 gRPC).
2. 프로젝트 최소 CRUD와 이슈 최소 CRUD. 이슈 검색에 필요한 키·제목·설명·타입·상태·우선순위·담당자·보고자·수정시각을 정본으로 가진다.
3. 프로젝트 생성자 PROJECT ADMIN 자동 부여, VIEW/EDIT/ADMIN 서버 인가, 삭제 시 grant 회수.
4. common-proto v0.6.0: ALM 이벤트 20~25번과 `AlmContentService` 단건/전량 조달 계약.
5. search-service: `alm-issue` 인덱스, 이벤트 소비, ALM gRPC 조달, 재색인, GraphQL `ISSUE` 결과와 PROJECT 권한 필터.
6. Gateway `/api/alm/**`, Compose `almdb`·alm-backend·서비스 환경변수, CI/GHCR/배포 매핑.

### Should

- 이벤트의 외부 버전(`occurred_at`)으로 중복·순서 역전을 방어한다.
- OpenSearch/Nori 실제 통합 테스트에서 이슈 제목·설명 검색과 PROJECT 권한 경계를 검증한다.
- dev 포트는 REST 19120, gRPC 19121로 +10000 규약을 지킨다.

### Won't — 후속

- `alm-front`의 보드·스프린트·댓글·워크로그·스킴 전체를 서버로 이관하지 않는다.
- 첨부파일, 실시간 협업, 알림 영속, 활동 스트림은 기존 `alm-front/docs/BACKLOG.md`와 Wave E로 남긴다.
- 이번 슬라이스에서 localStorage 스토어를 전면 교체하지 않는다. 백엔드 계약이 안정된 뒤 하이브리드 어댑터부터 단계 이관한다.
- outbox, gRPC 인증, 재색인 dual-write는 플랫폼 공통 후속 과제로 유지한다.

## 4. 사용자 스토리와 인수조건

### 프로젝트 생성자

Given 유효한 플랫폼 JWT가 있고 프로젝트 키가 중복되지 않았을 때  
When 프로젝트를 생성하면  
Then 프로젝트가 저장되고 생성자에게 PROJECT ADMIN grant가 멱등 부여된다.

### 프로젝트 멤버

Given PROJECT VIEW grant가 있는 사용자가 접근할 때  
When 프로젝트 이슈를 조회하면  
Then 허용된 프로젝트의 이슈만 반환된다.

Given PROJECT EDIT grant가 없는 사용자가 접근할 때  
When 이슈를 생성·수정·삭제하면  
Then 서버가 403으로 거부한다.

### 검색 사용자

Given 위키와 ALM 데이터가 색인되어 있을 때  
When GraphQL 통합 검색을 실행하면  
Then PAGE/ATTACHMENT/ISSUE가 한 결과 계약으로 반환되고 각 hit의 도메인 경로를 결정할 필드가 포함된다.

Given 사용자가 특정 PROJECT grant를 갖지 않을 때  
When 그 프로젝트 이슈와 일치하는 검색어를 입력하면  
Then 해당 hit은 결과에 포함되지 않는다.

## 5. 계약

### 이벤트 번호

| 번호 | payload |
|---:|---|
| 20 | ProjectCreated |
| 21 | ProjectUpdated |
| 22 | ProjectDeleted |
| 23 | IssueCreated |
| 24 | IssueUpdated |
| 25 | IssueDeleted |

이벤트에는 본문을 싣지 않는다. search-service가 `AlmContentService`로 원문을 조달한다.

### 내부 gRPC

- `GetIssueContent(issue_id)` — 없으면 NOT_FOUND.
- `ListIssueContents(project_id=0, after_id, limit)` — 관리자 재색인용 전량 스트리밍.
- `IssueContent`는 issue/project 표시값과 검색 필드를 비정규화해 전달한다.

### OpenSearch

- 별칭 `alm-issue`, 최초 물리 인덱스 `alm-issue-v1`.
- `_id=issueId`, `docType=ISSUE`.
- Nori 대상: `issueKey`, `projectName`, `title`, `content(description)`.
- keyword/filter 대상: `projectId`, `projectKey`, `type`, `status`, `priority`, `assigneeId`, `reporterId`.

### GraphQL

- `DocType`에 `ISSUE` 추가.
- `SearchInput.projectIds` 추가. 서버가 접근 가능한 PROJECT 집합과 교집합을 취한다.
- wiki 전용 `space*` 필드는 ISSUE에서 nullable, ALM 전용 `project*`·`issueKey`·`issueType`·`status`·`priority`는 wiki hit에서 nullable.
- mixed-domain 검색은 SPACE와 PROJECT 권한 분기를 각각 적용한 뒤 OR로 합친다. 한 도메인의 권한 0건이 다른 도메인의 허용 결과까지 비우면 안 된다.

## 6. 서비스·운영 경계

| 항목 | docker | dev |
|---|---:|---:|
| alm-backend REST | 9120 | 19120 |
| alm-backend gRPC | 9121 | 19121 |

- docker 프로필은 Eureka 미등록, Gateway가 `ALM_SERVICE_URI=http://alm-backend:9120`으로 직결한다.
- 백엔드와 gRPC 포트는 호스트에 공개하지 않는다.
- 로그는 docker 프로필 ECS JSON stdout, Alloy가 자동 수집한다.
- Redis Streams는 기존 `platform:events:v1`, consumer group은 기존 search-service 하나를 유지한다.

## 7. 실패 의미

- org gRPC UNAVAILABLE/DEADLINE: REST 503, 검색 GraphQL SERVICE_UNAVAILABLE.
- 권한 거부: REST 403, 검색에서는 hit 제외.
- ALM gRPC 조달 NOT_FOUND: 이슈 색인 삭제 후 ACK.
- OpenSearch 실패: 이벤트 ACK하지 않고 PEL 재시도→DLQ.
- 이벤트 발행 실패: 쓰기 요청은 성공하되 ERROR, 관리자 재색인으로 수렴.

## 8. 영향 저장소

- 신규 `alm-backend`
- `platform-backend`(common-proto, search-service)
- `gateway-server`
- `infra-settings`(Compose, DB init, CI 배포 매핑, 스모크)
- `alm-front`는 이번 슬라이스에서 계약 문서/후속 포인터만 갱신하고 런타임 스토어 전면 교체는 하지 않는다.


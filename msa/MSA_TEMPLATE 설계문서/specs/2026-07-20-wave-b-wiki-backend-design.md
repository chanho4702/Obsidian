# Wave B 설계 — wiki-backend 코어 (스페이스·페이지·버전·첨부 + 이벤트 발행)

작성일: 2026-07-20
상위 문서: `MSA_TEMPLATE 정리/15` (설계 확정) · `MSA_TEMPLATE 정리/16` (Wave A 구현기록)

## 목적

첫 번째 **제품 서비스** — wiki-backend. 검색(Wave C)·알림(Wave E) 없이도 완결 동작하는 위키 코어:

1. **스페이스·페이지(계층 트리)·버전 히스토리·첨부파일** CRUD
2. **org-service 권한 연동** — Wave A가 만든 gRPC 계약의 첫 실소비 (+ Caffeine 캐시)
3. **도메인 이벤트 발행** — `EventPublisher` 추상화 + **Redis Streams** 구현 (15번 결정 변경 반영). 소비자는 Wave C/E
4. Wave A 패턴 검증 — 독립 repo·프로필 분기·Dockerfile·CI를 "복제 표준"으로 재사용

## 확정 결정 (2026-07-20 대화)

| 항목 | 결정 |
|---|---|
| 이벤트 버스 | **Redis Streams** (Kafka 아님 — 15번 결정 변경 노트 참조). 발행 계약은 `EventPublisher` 인터페이스로 추상화 |
| 첨부파일 | **Wave B 포함** — 로컬 볼륨 저장. MinIO 등 오브젝트 스토리지는 L 티어 확장 경로로만 명시 |
| 본문 저장 | 마크다운 **원문 text** 저장 — 렌더링은 프론트 소관, 백엔드는 포맷 무지 |
| 버전 전략 | 저장 시 **전체 스냅샷**(page_revision 행) — diff 방식 거부: 복원·비교 복잡도 대비 스토리지 이득이 M 규모에서 무의미 |

## 컨텍스트 / 전제

- wiki-backend는 `MSA_TEMPLATE/` 아래 **독립 git repo** (도메인 repo — 15번 "도메인 2 + 플랫폼 1").
- **common-proto는 GitHub Packages 발행본을 소비** — platform-backend와 달리 `project(':common-proto')` 직참조가 아닌 첫 외부 소비자. 이번 웨이브에서 proto에 `platform.events.v1` + org.v1 `CreateGrant` rpc를 추가하고 **v0.2.0 발행**이 선행된다.
- 스택: org-service와 동일 (Boot 4.0.6 / Java 24 / Spring Cloud 2025.1.2 / Flyway·PostgreSQL(wikidb) / H2 test). 포트 **HTTP 9110** (gRPC 9111은 예약만 — 이번 웨이브에 wiki gRPC 서버 없음, YAGNI).
- Eureka 프로필 분기·런타임 전용 Dockerfile·tcp 헬스체크·GHCR+build 폴백 — 전부 Wave A/기존 패턴 그대로.
- 실시간 공동편집(동시 편집 병합)은 **범위 밖** — 낙관적 잠금(버전 충돌 409)으로 단순화.

## 1. common-proto v0.2.0 (platform-backend 선행 작업)

### platform.events.v1 — 이벤트 스키마

```protobuf
syntax = "proto3";
package platform.events.v1;
option java_multiple_files = true;
option java_package = "com.platform.proto.events.v1";

// 모든 도메인 이벤트의 공통 봉투. Redis Stream 엔트리의 payload 필드에 직렬화(binary)로 실림.
message EventEnvelope {
  string event_id = 1;        // UUID — 소비자 멱등 처리용
  int64 occurred_at = 2;      // epoch millis
  int64 actor_id = 3;         // 행위자 (auth user id)
  string source = 4;          // "wiki-backend" 등 발행 서비스명
  oneof payload {
    SpaceCreated space_created = 10;
    SpaceDeleted space_deleted = 11;
    PageCreated page_created = 12;
    PageUpdated page_updated = 13;
    PageDeleted page_deleted = 14;
    AttachmentAdded attachment_added = 15;
    // alm 이벤트는 Wave D에서 20번대로 추가
  }
}

message SpaceCreated { int64 space_id = 1; string key = 2; string name = 3; }
message SpaceDeleted { int64 space_id = 1; }
message PageCreated  { int64 page_id = 1; int64 space_id = 2; string title = 3; }
message PageUpdated  { int64 page_id = 1; int64 space_id = 2; string title = 3; int32 version = 4; }
message PageDeleted  { int64 page_id = 1; int64 space_id = 2; }
message AttachmentAdded { int64 attachment_id = 1; int64 page_id = 2; int64 space_id = 3; string filename = 4; }
```

- 이벤트에 본문(content)은 **싣지 않는다** — 색인기(Wave C)는 이벤트로 "무엇이 변했나"만 알고, 본문은 wiki REST로 가져간다(이벤트 비대화 방지 + 재색인 경로와 코드 공유).
- Stream 키: **`platform:events:v1`** 단일 스트림. `XADD ... MAXLEN ~ 100000` 트림. 소비자(Wave C/E)는 consumer group으로 각자 독립 소비.

### platform.org.v1 — CreateGrant rpc 추가

```protobuf
// PermissionService에 추가 (내부 전용 — 스페이스 생성자 자동 ADMIN 부여용)
rpc CreateGrant(CreateGrantRequest) returns (CreateGrantResponse);

message CreateGrantRequest {
  int64 user_id = 1;            // 대상 사용자
  ResourceType resource_type = 2;
  string resource_id = 3;
  Role role = 4;
}
message CreateGrantResponse { bool created = 1; }  // 이미 있으면 false (멱등)
```

- 필요 사유: 스페이스 생성 시 **생성자를 해당 SPACE의 ADMIN으로 자동 부여**해야 하는데, 기존 계약은 조회(Check/List)뿐. org-service 구현은 기존 GrantService 재사용(중복이면 멱등 false), UNSPECIFIED 인자는 INVALID_ARGUMENT.
- proto는 v1 패키지 내 **필드 추가·rpc 추가는 하위호환**이므로 v0.2.0 (마이너 범프).

## 2. wiki-backend — 도메인 설계

### 데이터 모델 (wikidb, Flyway V1)

```
space
  id          BIGSERIAL PK
  key         VARCHAR(30) NOT NULL UNIQUE   -- URL용 짧은 식별자 (영문·숫자·하이픈)
  name        VARCHAR(100) NOT NULL
  description TEXT
  created_by  BIGINT NOT NULL               -- auth user id
  created_at/updated_at TIMESTAMPTZ

page
  id          BIGSERIAL PK
  space_id    BIGINT NOT NULL FK -> space ON DELETE CASCADE
  parent_id   BIGINT FK -> page ON DELETE CASCADE   -- NULL = 루트. 계층 트리
  title       VARCHAR(255) NOT NULL
  content     TEXT NOT NULL                 -- 마크다운 원문
  version     INT NOT NULL DEFAULT 1        -- 현재 버전 번호 (낙관적 잠금 겸용)
  created_by/updated_by BIGINT NOT NULL
  created_at/updated_at TIMESTAMPTZ
  INDEX (space_id), INDEX (parent_id)

page_revision                               -- 저장할 때마다 스냅샷 1행
  id          BIGSERIAL PK
  page_id     BIGINT NOT NULL FK -> page ON DELETE CASCADE
  version     INT NOT NULL
  title       VARCHAR(255) NOT NULL
  content     TEXT NOT NULL
  edited_by   BIGINT NOT NULL
  created_at  TIMESTAMPTZ NOT NULL
  UNIQUE (page_id, version)

attachment
  id           BIGSERIAL PK
  page_id      BIGINT NOT NULL FK -> page ON DELETE CASCADE
  filename     VARCHAR(255) NOT NULL        -- 원본 파일명 (표시용)
  content_type VARCHAR(100) NOT NULL
  size_bytes   BIGINT NOT NULL
  storage_key  VARCHAR(64) NOT NULL UNIQUE  -- 디스크 파일명 (UUID) — 원본명과 분리(경로 조작 차단)
  uploaded_by  BIGINT NOT NULL
  created_at   TIMESTAMPTZ NOT NULL
```

- **페이지 삭제 = hard delete** (cascade로 하위 페이지·리비전·첨부 동반 삭제). 휴지통/복원은 후속 백로그 — 대신 삭제 전 확인은 프론트 책임으로 명시.
- 트리 무결성: parent는 **같은 space** 소속이어야 함(서비스 계층 검증). 순환 방지: 이동(parent 변경) 시 조상 체인 검사.

### 버전·동시성 규칙

- `PUT /pages/{id}`는 요청에 `expectedVersion` 필수. 현재 version과 다르면 **409 Conflict**(다른 사람이 먼저 저장) — 프론트가 병합 안내.
- 저장 성공 시: version+1, page_revision에 **새 내용** 스냅샷 추가. (V1 리비전은 페이지 생성 시 즉시 1행 — "모든 버전이 리비전 테이블에 있다"는 단일 불변식)
- 복원: `POST /pages/{id}/revisions/{version}/restore` = 해당 리비전 내용으로 **새 버전을 만드는 것** (히스토리 롤백 아님 — 이력 보존).

### 권한 매핑 (org-service 연동 — gRPC 클라이언트)

| 행위 | 필요 권한 (SPACE 리소스, resource_id = space.id 문자열) |
|---|---|
| 스페이스 목록·조회, 페이지 조회·리비전 조회, 첨부 다운로드 | VIEW |
| 페이지 생성·수정·이동·삭제·복원, 첨부 업로드·삭제 | EDIT |
| 스페이스 수정·삭제 | ADMIN |
| 스페이스 생성 | 인증만 — 생성 성공 시 `CreateGrant`로 생성자에게 해당 SPACE ADMIN 자동 부여 |

- **PermissionClient**: grpc-java 채널(`ORG_GRPC_HOST:9131`, dev 기본 localhost) + **Caffeine 캐시** `(userId, spaceId, action) → allowed`, TTL 30초, 최대 10k 엔트리 — Wave A defer "권한조회 클라이언트 캐시" 이행. 권한 회수 반영 지연 최대 30초는 수용(문서화).
- org-service 불능 시: **fail-closed(403)** + 로그. 가용성보다 인가 안전 우선(온프렘 단일 서버에서 org 다운은 어차피 전면 장애).
- 스페이스 목록은 ListUserGrants로 접근 가능 스페이스만 필터.

### 이벤트 발행

```java
public interface EventPublisher { void publish(EventEnvelope event); }
// 구현: RedisStreamEventPublisher — XADD platform:events:v1 MAXLEN ~ 100000, payload=proto bytes
```

- 발행 시점: **트랜잭션 커밋 후**(`@TransactionalEventListener(AFTER_COMMIT)` 경유) — 롤백된 변경의 이벤트 유출 방지.
- 발행 실패: 로그(WARN) 후 무시 — 코어 동작 비차단. 색인 정합은 Wave C의 전체 재색인 명령으로 보정(설계상 이벤트는 best-effort, 정합의 최종 근거는 wikidb).
- Redis 내구성: infra의 redis에 `--appendonly yes` 추가(compose 수정). rate limiter와 공용 인스턴스 유지(S/M 티어 충분, L에서 분리).

### REST API (게이트웨이 경유 `/api/wiki/**`)

| 메서드/경로 | 권한 | 설명 |
|---|---|---|
| GET /api/wiki/spaces | 인증 (접근 가능만 필터) | 스페이스 목록 |
| POST /api/wiki/spaces | 인증 | 생성 + 생성자 ADMIN 자동 grant. key 중복 400 |
| GET·PUT·DELETE /api/wiki/spaces/{id} | VIEW·ADMIN·ADMIN | |
| GET /api/wiki/spaces/{id}/pages | VIEW | 트리 목록 (id·parentId·title만 — 본문 제외) |
| POST /api/wiki/pages | EDIT(대상 space) | 생성 (spaceId, parentId?, title, content) |
| GET·DELETE /api/wiki/pages/{id} | VIEW·EDIT | GET은 본문 포함 |
| PUT /api/wiki/pages/{id} | EDIT | title·content·parentId(이동), `expectedVersion` 필수 — 불일치 409 |
| GET /api/wiki/pages/{id}/revisions | VIEW | 버전 목록(메타만) |
| GET /api/wiki/pages/{id}/revisions/{version} | VIEW | 특정 버전 본문 |
| POST /api/wiki/pages/{id}/revisions/{version}/restore | EDIT | 새 버전으로 복원 |
| POST /api/wiki/pages/{id}/attachments | EDIT | multipart 업로드 (max `WIKI_MAX_ATTACHMENT_MB`, 기본 20) |
| GET /api/wiki/attachments/{id} | VIEW(소속 space) | 다운로드 — `Content-Disposition: attachment` 고정(브라우저 인라인 실행 차단) |
| DELETE /api/wiki/attachments/{id} | EDIT | 메타+파일 삭제 |

- JWT 리소스 서버·JIT 없음(멤버 미러링은 org 소관) — sub만 userId로 사용. 보안 구성은 board/org 패턴 그대로(jwk-set-uri + issuer/audience).
- 게이트웨이 라우트: `id: wiki`, `uri: ${WIKI_SERVICE_URI:lb://wiki-backend}`, `Path=/api/wiki/**` — org 패턴 동일.

### 첨부 저장

- 경로: `${WIKI_FILES_DIR:./data/attachments}/{storage_key}` — compose에서 named volume `platform-wiki-files:/data/attachments`.
- storage_key는 UUID — 원본 파일명은 DB에만(경로 조작·이름 충돌 원천 차단). 페이지 삭제 시 파일도 삭제(실패는 로그 — 고아 파일은 무해, 후속 청소 배치는 백로그).

## 3. 인프라·CI 통합

- compose: `wiki-backend` 서비스(GHCR+build 폴백, docker 프로필, tcp 9110 헬스체크) + `wikidb` 생성(init sql + 기존 볼륨 수동 CREATE) + redis `--appendonly yes` + wiki-files 볼륨 + gateway에 `WIKI_SERVICE_URI`.
- **Wave A defer 소형 2건 동승**: ① compose에 `PLATFORM_ISSUER`/`PLATFORM_AUDIENCE`를 auth·org·wiki 3서비스에 명시 주입(기본값 우연 일치 의존 제거), ② 우산 README에 `PLATFORM_BOOTSTRAP_ADMIN_ID` 운영 전 명시 설정 경고 1줄.
- CI: wiki-backend repo에 Wave A와 동일 ci.yml(build+test). 이미지 GHCR 발행은 여전히 CI/CD 웨이브로 이관(compose build 폴백으로 충분). platform-backend는 proto 변경 커밋 후 **v0.2.0 태그 발행**.

## 4. 테스트 전략

- Wave A 패턴 계승: H2(test 프로필) + @DataJpaTest + @SpringBootTest/MockMvc(springSecurity) + TestAuth 헬퍼 이식.
- **PermissionClient는 테스트에서 페이크 빈**(고정 허용/거부 맵) — org-service를 띄우지 않음. gRPC 왕복 자체는 Wave A에서 검증됨. 실연동은 성공 기준 ④(수동/스모크).
- 이벤트 발행은 **인메모리 EventPublisher 페이크**로 "무슨 이벤트가 발행됐나" 검증 + RedisStreamEventPublisher는 통합 스모크(성공 기준 ⑤)에서 XLEN 실측.
- 핵심 시나리오: 트리(같은 space 검증·순환 방지), 409 낙관 잠금, 리비전 불변식(모든 버전 존재), 복원=새 버전, 첨부 업로드/다운로드/max size 413, 권한 3레벨 가드.

## 성공 기준

1. common-proto **v0.2.0** 발행 + wiki-backend가 Packages에서 resolve (첫 외부 소비 증명)
2. 페이지 CRUD·트리·버전·복원·첨부가 테스트로 검증 (전체 green)
3. 권한 가드: VIEW/EDIT/ADMIN 3레벨 + 스페이스 생성자 자동 ADMIN(CreateGrant) — org-service 실연동 스모크에서 비인가 403
4. compose 통합: wiki-backend healthy, 게이트웨이 경유 `/api/wiki/**` 401 스모크
5. 페이지 저장 시 `platform:events:v1` 스트림에 EventEnvelope 축적 실측 (`XLEN`/`XRANGE`)
6. dev 모드(IDE+Eureka)와 docker 모드 양쪽 기동

## 범위 밖 (후속)

- 검색·색인(Wave C), 알림·활동 피드(Wave E), 휴지통/soft delete, 실시간 공동편집, 페이지 간 링크 그래프, 첨부 청소 배치, MinIO(L 티어), wiki gRPC 서버(9111 예약만)

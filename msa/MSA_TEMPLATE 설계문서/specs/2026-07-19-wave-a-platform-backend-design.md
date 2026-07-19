# Wave A 설계 — platform-backend 골격 + common-proto + org-service 최소 RBAC

작성일: 2026-07-19
상위 문서: `MSA_TEMPLATE 정리/15 ALM·Wiki 백엔드 요구사항 + 서비스 분할 설계 (2026-07-19)`

## 목적

ALM·Wiki 플랫폼의 **토대 웨이브**. 세 가지를 만든다:

1. **platform-backend** — 횡단 서비스용 Gradle 멀티모듈 repo (신규 독립 repo)
2. **common-proto** — gRPC 계약 모듈. GitHub Packages로 발행하여 이후 wiki-backend·alm-backend가 의존성으로 소비
3. **org-service** — 조직·팀·RBAC 최소 구현. 모든 서비스가 gRPC로 권한을 조회하는 첫 플랫폼 부품

이후 모든 웨이브(B: wiki, C: search, D: alm, E: notification)가 이 토대 위에 선다.

## 컨텍스트 / 아키텍처 전제

- 15번 문서에서 확정: B안 5서비스 분할, 규모 M, PostgreSQL 통일, repo는 도메인 2 + 플랫폼 1.
- platform-backend는 `MSA_TEMPLATE/` 아래 **새 형제 디렉터리이자 독립 git repo** (우산 repo에서 gitignore — 기존 서비스 repo 패턴 동일).
- 기존 서비스(gateway/eureka/auth/board)는 **수정 최소화** — 이번 웨이브에서 gateway 라우트 추가 1건만 건드린다.
- 인증은 기존 플랫폼 그대로: auth-server 발급 JWT를 JWKS(`jwk-set-uri`)로 자체 검증하는 리소스 서버 패턴(board-service와 동일).
- Eureka는 프로필 분기 원칙(15번 확정): dev 프로필 = Eureka 등록, docker 프로필 = 등록 안 함 + 컨테이너 DNS.

## 스택

기존 서비스와 동일 라인: **Spring Boot 4.0.6 / Java 24 / Spring Cloud 2025.1.2** / Spring Web / Spring Security(resource-server) / Data JPA / Validation / Flyway / PostgreSQL(runtime) / H2(test) / Lombok. group `com.platform`.

신규 도입:

| 기술 | 용도 | 비고 |
|---|---|---|
| protobuf + `com.google.protobuf` gradle 플러그인 | proto → Java 코드젠 | common-proto 모듈 |
| grpc-java (netty-shaded, protobuf, stub) | gRPC 런타임 | 버전은 protobuf 플러그인과 호환 최신 |
| **spring-grpc** (공식 `org.springframework.grpc`) | org-service gRPC 서버 부트 통합 | Boot 4 호환 최신 버전은 구현 시 확정. 호환 문제 시 폴백: `net.devh:grpc-server-spring-boot-starter` |
| maven-publish 플러그인 | common-proto → GitHub Packages 발행 | GHCR 인증과 동일한 GITHUB_TOKEN 흐름 |

## 포트 계획 (전체 웨이브 공통 — 이번에 확정)

| 서비스 | HTTP | gRPC | 비고 |
|---|---|---|---|
| board-service | 9100 | — | 기존 |
| wiki-backend | 9110 | 9111 | Wave B |
| alm-backend | 9120 | 9121 | Wave D |
| **org-service** | **9130** | **9131** | **이번 웨이브** |
| search-service | 9140 | — | Wave C (GraphQL over HTTP) |
| notification-service | 9150 | — | Wave E |

- **9200번대를 피한 이유**: Elasticsearch 기본 포트(9200/9300)와의 충돌 예방. Kafka(9092)와도 겹치지 않는다.
- 규칙: HTTP = `91x0`, gRPC = `91x1`.

## repo 구조

```
platform-backend/                (독립 git repo → github.com/chanho4702/platform-backend)
├─ settings.gradle               rootProject.name = 'platform-backend'
│                                include 'common-proto', 'org-service'
│                                (search-service·notification-service는 Wave C/E에서 include 추가)
├─ build.gradle                  공통 설정(그룹·자바 툴체인·repositories)만. 버전은 각 모듈에서
├─ common-proto/
│   ├─ build.gradle              java-library + protobuf + maven-publish
│   └─ src/main/proto/
│       └─ platform/org/v1/org.proto
└─ org-service/
    ├─ build.gradle              Spring Boot 앱. project(':common-proto') 직접 참조
    ├─ Dockerfile                기존 4서비스와 동일한 런타임 전용 패턴 (JRE + app.jar)
    ├─ .run/bootRun.run.xml      IntelliJ 공유 Run Config (기존 패턴)
    └─ src/...
```

- org-service는 같은 repo의 common-proto를 `project(':common-proto')`로 **직접 참조** — 발행 버전을 기다릴 필요 없음.
- wiki/alm(다른 repo)은 **발행된 아티팩트** `com.platform:common-proto:x.y.z`를 GitHub Packages에서 소비.

## common-proto — 계약 정의

### 버전 정책

- semver. Wave A에서 `0.1.0` 최초 발행. **1.0.0 전까지는 breaking change 허용**(우리만 쓰는 단계).
- proto 패키지는 `platform.org.v1` 처럼 **패키지에 v1을 박는다** — 향후 breaking change는 v2 패키지 신설로 대응(온프렘 설치판에서 구버전 서비스와 공존 가능해야 하므로).
- Kafka 이벤트 스키마는 **Wave B에서 `platform.events.v1`로 추가**한다. 이번 웨이브 범위 밖.

### org.proto (요지)

```protobuf
syntax = "proto3";
package platform.org.v1;
option java_multiple_files = true;
option java_package = "com.platform.proto.org.v1";

service PermissionService {
  // 단건 권한 판정 — 모든 서비스의 인가 2차 방어가 이걸 호출
  rpc CheckPermission(CheckPermissionRequest) returns (CheckPermissionResponse);
  // 검색 결과 권한 필터용(Wave C) — 사용자가 접근 가능한 리소스 목록
  rpc ListUserGrants(ListUserGrantsRequest) returns (ListUserGrantsResponse);
}

enum ResourceType { RESOURCE_TYPE_UNSPECIFIED = 0; GLOBAL = 1; SPACE = 2; PROJECT = 3; }
enum Action { ACTION_UNSPECIFIED = 0; VIEW = 1; EDIT = 2; ADMIN = 3; }
enum Role { ROLE_UNSPECIFIED = 0; VIEWER = 1; EDITOR = 2; ROLE_ADMIN = 3; }

message CheckPermissionRequest {
  int64 user_id = 1;          // JWT sub
  ResourceType resource_type = 2;
  string resource_id = 3;      // GLOBAL이면 빈 값
  Action action = 4;
}
message CheckPermissionResponse { bool allowed = 1; Role effective_role = 2; }

message ListUserGrantsRequest { int64 user_id = 1; ResourceType resource_type = 2; }
message ListUserGrantsResponse { repeated Grant grants = 1; }
message Grant { ResourceType resource_type = 1; string resource_id = 2; Role role = 3; }
```

- `resource_id`는 **string** — 리소스 소유 서비스(wiki의 space id, alm의 project id)가 각자 id 체계를 갖더라도 계약이 흔들리지 않게.
- 권한 모델: `ADMIN ⊃ EDITOR ⊃ VIEWER` 계층. Action↔Role 판정은 org-service 내부 규칙(VIEW←VIEWER↑, EDIT←EDITOR↑, ADMIN←ADMIN).

## org-service — 도메인 설계

### 결정: 단일 조직(싱글 테넌트) 전제

온프렘 설치 1개 = 조직 1개. `organization` 엔티티를 만들지 않는다. 멀티테넌시는 범위 밖(필요 시 전 테이블에 org_id 추가하는 확장 경로만 인지). — 거부안: 처음부터 멀티테넌트 스키마. SaaS 전환 계획이 확정되기 전에는 모든 쿼리·인덱스·테스트에 세금만 부과한다.

### 데이터 모델 (org_db, Flyway 관리)

```
member                              -- 플랫폼 사용자의 로컬 뷰(원본은 auth-server/Keycloak)
  id            BIGINT PK           -- auth-server user id (JWT sub) 그대로. 자체 시퀀스 없음
  display_name  VARCHAR NOT NULL    -- 표시용 스냅샷
  email         VARCHAR
  status        VARCHAR NOT NULL    -- ACTIVE | DEACTIVATED
  created_at/updated_at TIMESTAMPTZ

team
  id            BIGSERIAL PK
  name          VARCHAR NOT NULL UNIQUE
  description   TEXT
  created_at/updated_at TIMESTAMPTZ

team_member
  team_id       BIGINT FK -> team ON DELETE CASCADE
  member_id     BIGINT FK -> member ON DELETE CASCADE
  role          VARCHAR NOT NULL    -- LEAD | MEMBER
  PK (team_id, member_id)

grant_entry                         -- 권한 부여의 단일 원장 ("grant"는 SQL 예약어라 회피)
  id            BIGSERIAL PK
  subject_type  VARCHAR NOT NULL    -- USER | TEAM
  subject_id    BIGINT  NOT NULL
  resource_type VARCHAR NOT NULL    -- GLOBAL | SPACE | PROJECT
  resource_id   VARCHAR NOT NULL DEFAULT ''   -- GLOBAL이면 ''
  role          VARCHAR NOT NULL    -- VIEWER | EDITOR | ADMIN
  created_at    TIMESTAMPTZ NOT NULL
  UNIQUE (subject_type, subject_id, resource_type, resource_id)
  INDEX (resource_type, resource_id) / INDEX (subject_type, subject_id)
```

권한 판정 알고리즘(CheckPermission): 사용자 직접 grant + 소속 팀들의 grant를 모아 **최고 role**을 취하고, GLOBAL grant는 모든 리소스에 적용되는 상위 우선 규칙. 쿼리 2개(직접 + 팀 경유)면 충분하며 M 규모에서 인덱스로 p95 한 자릿수 ms.

### member 생성 시점 (결정: JIT 미러링)

auth-server가 이미 Keycloak JIT 프로비저닝으로 사용자를 만들므로, org-service는 **인증된 요청이 처음 들어올 때 JWT의 sub·name·email 클레임으로 member 행을 upsert**한다(REST 필터 레벨). 별도 사용자 동기화 배치·이벤트 없음. — 거부안: auth-server가 Kafka로 UserCreated 발행. Kafka가 Wave B 이후에나 인프라에 들어오므로 토대 웨이브가 인프라를 앞지르게 된다.

### API

**gRPC (:9131)** — 위 PermissionService 구현. 내부 전용(게이트웨이 미경유), 인증은 신뢰 네트워크 전제 + 온프렘 배포판에서는 컨테이너 네트워크 격리로 보호(mTLS는 L 티어 확장 경로).

**REST (:9130, 게이트웨이 경유 `/api/org/**`)** — 관리 화면용:

| 메서드/경로 | 권한 | 설명 |
|---|---|---|
| GET /api/org/members | 인증 | 멤버 목록(페이지네이션) |
| GET /api/org/teams · POST · PUT /{id} · DELETE /{id} | 조회=인증 · 변경=GLOBAL ADMIN | 팀 CRUD |
| PUT/DELETE /api/org/teams/{id}/members/{memberId} | GLOBAL ADMIN | 팀원 추가/제거 |
| GET /api/org/grants?resourceType=&resourceId= | GLOBAL ADMIN | 권한 부여 목록 |
| POST /api/org/grants · DELETE /api/org/grants/{id} | GLOBAL ADMIN | 권한 부여/회수 |
| GET /api/org/me/permissions | 인증 | 내 grant 목록(프론트 메뉴 제어용) |

- GLOBAL ADMIN 판정: grant_entry의 GLOBAL/ADMIN 보유자. **부트스트랩 문제**(최초 관리자는 누가?)는 Flyway seed로 해결 — 환경변수 `PLATFORM_BOOTSTRAP_ADMIN_ID`(auth user id)를 기동 시 GLOBAL ADMIN grant로 upsert.
- JWT 검증: board-service와 동일 `jwk-set-uri` 패턴(issuer-uri 사용 금지 — OIDC discovery 미노출로 기동 실패, 06-27 스펙 참조).

## 프로필 구성 (Eureka 분기 — 15번 확정사항의 첫 적용)

| 프로필 | Eureka | DB | 용도 |
|---|---|---|---|
| default (dev) | 등록 ON | localhost:5433/org_db | IntelliJ 기동 |
| docker | `eureka.client.enabled=false` | postgres:5432/org_db | compose 배포판 |

gateway-server 라우트 추가(이번 웨이브 유일한 기존 서비스 수정):

- dev: `/api/org/** → lb://org-service`
- docker 프로필: `/api/org/** → http://org-service:9130` (기존 gateway가 이미 docker 프로필 분기를 갖고 있으면 그 패턴에 편승, 없으면 라우트 uri를 env로 주입)

## CI/CD·인프라 통합

- **GitHub Actions** (platform-backend repo, 기존 self-hosted runner·GHCR 패턴 편승):
  - PR: `gradlew build` (전 모듈 테스트 포함)
  - main push: org-service 이미지 → GHCR(`ghcr.io/chanho4702/org-service`, 기존 태깅 규칙 동일)
  - **`v*` 태그 push: common-proto → GitHub Packages 발행** (버전 = 태그)
- **compose 통합** (`infra/keycloak/docker-compose.yml`):
  - `org-service` 서비스 추가 — image GHCR + build 폴백, 헬스체크(actuator), 기존 패턴 동일
  - postgres init에 `org_db` 생성 추가(기존 authdb·boarddb 생성 방식과 동일 경로)
- Kafka·Elasticsearch 컨테이너는 **이번 웨이브에 추가하지 않는다** (Wave B/C에서).

## 테스트 전략

- **단위**: 권한 판정 규칙(role 계층, GLOBAL 우선, 팀 경유 병합) — 순수 도메인 테스트로 집중 커버.
- **슬라이스**: JPA(H2) 리포지토리 쿼리, REST 컨트롤러(spring-security-test로 JWT 모킹) — board-service 기존 패턴.
- **gRPC 통합**: in-process 서버로 PermissionService 왕복 검증(네트워크 불요, CI 안정).
- **proto 소비 검증**: 발행된 아티팩트를 별도 샘플 gradle 프로젝트로 resolve — 수동 1회(성공 기준 ①), CI 자동화는 하지 않음.

## 성공 기준 (수동 E2E 포함)

1. `common-proto:0.1.0`이 GitHub Packages에 발행되고, 외부 gradle 프로젝트에서 의존성 resolve + 생성 클래스 임포트 성공
2. dev 모드: IntelliJ로 org-service 기동 → Eureka 대시보드 등록 확인 → 게이트웨이 경유 `/api/org/me/permissions` 200
3. docker 모드: compose up → org-service healthy → `grpcurl`로 CheckPermission 호출 시 부트스트랩 관리자에 `allowed=true`
4. 부트스트랩: `PLATFORM_BOOTSTRAP_ADMIN_ID`로 지정한 계정이 GLOBAL ADMIN으로 동작(팀 생성 REST 성공, 일반 계정은 403)
5. `gradlew build` 그린 (전 모듈)

## 범위 밖 (이후 웨이브)

- Kafka·ES 인프라, 이벤트 스키마(`platform.events.v1`) — Wave B/C
- search-service·notification-service 모듈 — Wave C/E
- wiki-backend·alm-backend repo — Wave B/D
- 권한조회 클라이언트 캐시(Caffeine TTL) — 소비 서비스(Wave B) 구현 시
- gRPC mTLS, 멀티테넌시, 권한 감사 로그 — L 티어/후속 확장 경로

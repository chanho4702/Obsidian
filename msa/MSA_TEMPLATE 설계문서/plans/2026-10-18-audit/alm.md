# ALM 감사 보고서 (alm-front · alm-backend) — 2026-10-18

- 범위: `C:\MSA_TEMPLATE\alm-front`, `C:\MSA_TEMPLATE\alm-backend`. 읽기 전용, 테스트 실행 없음(코드 판독 + `git show/diff` 근거).
- 기준선: **main**. 백엔드 미병합 브랜치 `feat/alm-integrity-scale-20261016`(=`feat/alm-query-scale` 포함), `fix/alm-postgres-null-params-20261016`; 프론트 `feat/alm-integrity-scale-20261016`, `feat/alm-performance-ux-20261016`을 대조해 "브랜치에서 이미 고침" 여부를 표시했다.
- 프론트 라인 번호는 `git show main:<path>` 기준이다(작업 트리는 다른 브랜치이고 P4c 미커밋 변경이 있다).
- 확정 설계 결정(Keycloak 리다이렉트·백채널 로그아웃·Redis Streams·PostgreSQL·common-starter 등)은 뒤집는 제안을 하지 않았다.
- 심각도: Critical(깨짐·데이터 손실·보안) / Important(계약·회귀·규모에서 깨짐) / Minor. 규모: S(반나절 이하) / M(1~2일) / L(3일 이상).

## 요약 표

| ID | 분류 | 심각도 | 제목 | 브랜치에서 고침 |
|---|---|---|---|---|
| A-01 | 보안 | **Critical** | 이슈 링크로 권한 없는 프로젝트의 이슈 본문 열람 | 아니오 |
| A-02 | 보안/정합성 | Important | 휴지통 프로젝트의 이슈가 검색·AQL·직접 조회·쓰기로 계속 노출됨 | 아니오 |
| A-03 | 보안 | Important | @멘션 대상 검증 없음, 비멤버에게 알림과 메일(키+제목) 전송 | 아니오 |
| A-04 | 정합성 | Important | main의 동시 수정 덮어쓰기·CSV 전부 롤백·이벤트 유실 | **예**(병합 필요) |
| A-05 | 정합성/테스트 | Important | 테스트가 H2 + Hibernate DDL(Flyway 꺼짐), PG 전용 결함이 CI를 통과 | 일부(null 파라미터만) |
| A-06 | 성능 | Important | 이슈 상세 모달이 열 때·변경할 때마다 프로젝트 전체 이슈를 읽음 | 아니오 |
| A-07 | 성능 | Important | 보드 이슈 API가 프로젝트 전체를 메모리에 올린 뒤 자바에서 거름 | 일부(N+1만) |
| A-08 | 성능/오류 | Important | 리포트가 변경 이력 전체를 무제한으로 받고, 실패하면 스피너가 멈춤 | 아니오 |
| A-09 | 정합성 | Important | 200건 조용히 잘림(하위 이슈·버전 진행률) | 일부(검색만) |
| A-10 | 성능 | Important | `statusMetaByProject` 1+N 팬아웃(홈·검색·⌘K 모달) | 아니오 |
| A-11 | 성능/버그 | Important | 일괄 편집이 이슈마다 요청 3번을 순차로 보냄, 실패 행에 키 대신 id 표시 | 일부(버전 보존만) |
| A-12 | 번들 | Important | 목업(`jiraMock` 5.5k줄)이 프로덕션 번들에 포함됨 | 아니오 |
| A-13 | 성능(DB) | Important | 담당자·라벨 인덱스 없음, `%like%` 본문 스캔, 와일드카드 이스케이프 없음 | 아니오 |
| A-14 | 보안(DoS) | Important | CSV 가져오기 항목 수 상한 없음 | 아니오 |
| A-15 | 성능 | Important | 홈·대시보드 가젯이 모든 프로젝트의 모든 이슈를 적재함 | **예** |
| A-16 | 정합성 | Minor | 프로젝트 마지막 관리자 보호가 클라이언트에만 있음 | 아니오 |
| A-17 | 기능/권한 | Minor | 댓글·워크로그는 작성자만 삭제 가능(관리자 정리 불가), 권한을 잃은 뒤에도 수정 가능 | 아니오 |
| A-18 | 정합성 | Minor | 스프린트 완료·릴리스가 클라이언트가 보낸 `doneStatuses`를 그대로 믿음 | 아니오 |
| A-19 | 정합성 | Minor | 보드 이동·랭크는 version을 올리지 않아, 전체 본문 PUT 클라이언트가 이동을 되돌릴 수 있음 | 아니오 |
| A-20 | 성능 | Minor | 링크 목록 N+1, 순환 검사 체인 N쿼리 | 아니오 |
| A-21 | 기능 공백 | Minor | 버전 수정·바로 가기 수정 API는 있는데 UI가 없음, `versionProgress`는 죽은 코드 | 아니오 |
| A-22 | 위생 | Minor | 거대 파일, 목업과 서버의 규칙 이중 구현, vitest는 목업만 검증 | 아니오 |
| A-23 | 문서 | Minor | 문서 드리프트 7건 | 아니오 |

---

## A-01 이슈 링크로 권한 없는 프로젝트의 이슈 본문 열람 — Critical (보안)

- **근거**
  - `alm-backend/src/main/java/com/platform/almbackend/collab/CollaborationService.java:187-206` `addLink`: 원본만 `require(userId, sourceId, EDIT)`로 확인하고, 대상은 `issues.findById(targetId)`로 존재만 본다. 대상 프로젝트 권한 확인이 없다.
  - 같은 파일 `:173-185` `links`: 링크 반대편 이슈를 `IssueResponse.from(other)`로 **전체 본문**(제목·설명·담당자 등)째 돌려준다. 반대편 프로젝트 권한은 확인하지 않는다.
  - `:203-204`: 대상 이슈의 활동 로그에도 행을 쓴다(권한 없는 프로젝트에 대한 쓰기).
- **이유**: 프로젝트 A에서 EDIT 권한만 있는 사용자가 이슈 id(순차 IDENTITY라 추측 가능)를 하나씩 링크하면, 볼 권한이 없는 프로젝트 B 이슈의 설명까지 `GET /api/alm/issues/{a}/links`로 읽는다. 위키 스페이스 격리 수준의 경계를 우회한다.
- **재현**: 사용자 U는 P1에서 EDIT, P2에는 grant 없음. `POST /api/alm/issues/{P1이슈}/links {targetId: <P2 이슈 id>, type:"relates"}` → 200. 이어서 `GET .../links` → `other.description`에 P2 본문이 담긴다.
- **제안(S)**: `addLink`에서 `projectService.require(userId, target.getProjectId(), VIEW)`를 확인한다(지라도 대상 Browse 권한을 요구한다). `links()`는 반대편 프로젝트를 VIEW 할 수 없으면 해당 행을 빼거나 키만 남긴 축약 응답을 준다. 사용자별로 projectId 집합을 한 번만 판정해 캐시를 활용한다. 회귀 테스트 2개(링크 생성 403, 기존 교차 링크 조회 시 본문 숨김)를 추가한다.
- **브랜치**: 고쳐지지 않음.

## A-02 휴지통 프로젝트의 이슈가 계속 노출되고 쓸 수 있음 — Important (보안/정합성)

- **근거**
  - 휴지통은 `Project`의 `@SQLRestriction("deleted_at is null")`(`domain/Project.java:20`)로만 숨긴다. `Issue`에는 프로젝트 휴지통 조건이 없다.
  - `issue/IssueSearchService.java:139-148` `scopedProjectIds`는 org grant 목록을 그대로 쓴다. 휴지통 프로젝트는 grant를 회수하지 않으므로(`ProjectService.delete:100-106`은 `trash()`만 한다) 검색 결과에 남는다. 전역 관리자는 `scope=null`이라 전부 보인다.
  - `search/aql/AqlSearchService.java:139-142` AQL도 같다(`AqlIssueRow`는 제한 없는 매핑이다).
  - `project/ProjectService.java:122-127` `require()`: 보관 가드가 `projects.findById(projectId).map(isArchived).orElse(false)`인데, 휴지통 프로젝트는 `findById`가 비어 **false**로 판정된다. 그래서 `GET /issues/{id}`, 댓글(VIEW만 요구), 워크로그·첨부·웹 링크·링크 추가(EDIT, 프로젝트 잠금 없음)가 그대로 성공한다. 프로젝트 잠금을 거치는 수정·이동은 우연히 404가 난다.
  - gRPC 백필(`grpc/AlmContentGrpcService.java:64-69`)은 휴지통 이슈를 건너뛰지만 "고아 이슈" 경고로 로그를 오염시킨다.
- **이유**: "삭제 = 휴지통" 의미(검색 색인에서 제외, 삭제 이벤트 발행)와 REST 동작이 어긋난다. 삭제했다고 믿은 프로젝트의 내용이 검색에 남고, 쓰기까지 받는다.
- **제안(M)**: `ProjectService`에 `requireActive(projectId)`를 두고, 휴지통이면 404로 `require()` 안에서 막는다(`findTrashedById` 네이티브 조회 1회. 30초 캐시 가능). 검색·AQL 범위 계산에서 `projects.findTrashedIds()`를 빼거나 `project.deleted_at is null` 서브쿼리 술어를 추가한다. 테스트는 휴지통 이동 후 검색·AQL·get·comment에 대해 404 또는 결과 제외를 확인한다.
- **브랜치**: 고쳐지지 않음(브랜치 `IssueSearchService:285`도 같은 범위 계산이다).

## A-03 @멘션 대상 검증 없음 — 비멤버 알림·메일 누출 — Important (보안)

- **근거**: `notification/NotificationService.java:115-123` `notifyMentioned`는 클라이언트가 보낸 `mentionedUserIds`를 그대로 수신자로 쓴다. 이슈 프로젝트에 대한 VIEW 권한을 확인하지 않는다. 호출부는 `IssueService.java:172,280`, `CollaborationService.java:103,115`. 메일 제목·본문은 `issue.getKey() + " " + issue.getTitle()`이다(`EmailNotifier.java:236`). 정지·비활성 계정만 걸러진다(`:198`).
- **이유**: EDIT(댓글은 VIEW) 권한자가 임의 사용자 id를 멘션해 비공개 프로젝트 이슈의 제목을 조직 전체 누구에게나 메일로 보낼 수 있다. 스팸 벡터이기도 하다. 워처도 권한을 잃은 뒤 계속 알림을 받는다(`recipients()` `:147-156`).
- **제안(S)**: 발송 직전에 `permissions.check(recipient, projectId, VIEW).allowed()`로 거른다(30초 캐시가 있어 비용이 작다). org 불능(503)이면 알림만 생략하고 저장은 계속한다. 프론트 멘션 후보도 프로젝트 멤버로 한정돼 있는지 확인한다.
- **브랜치**: 고쳐지지 않음.

## A-04 main의 동시 수정 덮어쓰기·CSV 전부 롤백·이벤트 유실 — Important (정합성), 브랜치에서 고침

- **근거(main)**
  - 백엔드: `issue/IssueService.java:211-219`는 `requireIssue`(1차 캐시에 적재) 뒤 `findByIdForUpdate`를 부른다. 잠금은 잡지만 **낡은 관리 엔티티**의 version과 비교한다. 같은 원인이 이슈 번호 카운터(`:84-86`)와 스프린트 시작·완료에도 있다.
  - 프론트: `alm-front/src/features/alm/store/jiraApi.ts:480-528` `updateIssue`는 저장 직전 GET으로 최신 version을 읽어 `expectedVersion`으로 보낸다. 결과적으로 낙관적 락이 늘 통과한다. 스프린트(`:577`)와 프로젝트(`:655`)도 같다.
  - CSV: `IssueService.java:94-131`은 단일 트랜잭션이다. 다른 `@Transactional` 빈의 검증 예외가 rollback-only를 남기면 성공한 행까지 `UnexpectedRollbackException`으로 사라진다.
  - 이벤트: `event/EventRelay.java:22-31`은 커밋 뒤 직접 XADD한다. Redis가 실패하면 로그만 남기고 색인 이벤트를 잃는다.
- **브랜치**: 백엔드 `51db504`(refresh PESSIMISTIC_WRITE·행별 REQUIRES_NEW·V26 outbox), `2f92ff6`(outbox 단일 소유), 프론트 `994e7d4`·`dc395a1`(편집 시작 version 유지·409 UX)이 고친다.
- **제안(S)**: 머지 순서는 백엔드 `fix/alm-postgres-null-params` → `feat/alm-integrity-scale`(V26) → 프론트 `feat/alm-integrity-scale` → `feat/alm-performance-ux`(이미 integrity에 포함된 3cef390과의 관계 확인)다. 머지 전에 PG 실측(A-05)으로 V26과 `skip locked`를 검증한다. **main 배포본은 지금 lost update가 가능한 상태**라는 점을 우선순위에 반영한다.

## A-05 PG ↔ H2 차이를 CI가 못 잡음 — Important (정합성/테스트)

- **근거**: `alm-backend/src/test/resources/application-test.yml:1-10`은 H2 `MODE=PostgreSQL`, `ddl-auto: create-drop`, `flyway.enabled: false`다. 행위 테스트 전부가 Flyway 스키마가 아니라 Hibernate가 만든 스키마에서 돈다. 부분 유니크 인덱스(`uq_sprint_one_active`), `ON DELETE CASCADE/SET NULL`, CHECK 제약이 테스트에 없다. Testcontainers는 schema 테스트 3개(`schema/FlywaySchemaValidationTest` 등)만 쓴다.
  - 실제 사례: `(:since is null or x >= :since)`가 PG에서 500을 냈다(`repository/AuditLogRepository.java:17-20`, `IssueChangeLogRepository.java:24-29`, 브랜치 `87257fc`가 고침). 위키에도 BLOG CHECK 제약 결함이 main에 남아 있다(메모리 기록).
- **이유**: "H2는 통과, PG는 500" 계열 결함이 반복되는 구조다. outbox `skip locked`처럼 PG 고유 동작이 늘어날수록 위험이 커진다.
- **제안(M)**: `@Tag("pg")` 통합 프로필을 만들어 `PostgreSQLContainer` + Flyway on으로 컨트롤러 테스트 묶음(최소 Issue·Search·AQL·Sprint·Admin·ChangeLog)을 돌리고, CI(`.github/workflows/ci.yml`의 `./gradlew build`)에 포함한다(ubuntu runner는 Docker를 쓸 수 있다). H2는 빠른 로컬용으로 남긴다.
- **브랜치**: null 파라미터 2건만 고침.

## A-06 이슈 상세 모달이 프로젝트 전체 이슈를 반복 적재 — Important (성능)

- **근거**: `alm-front/src/features/alm/components/IssueDetailModal.tsx:156-168` `refreshRelations`가 `listIssues(projectId)`를 부른다(`jiraApi.ts:371-379`: 200건 페이지를 **순차** 루프로 끝까지 읽는다). 모달을 열 때(`:219`)와 하위 이슈·링크·워크로그를 추가·삭제할 때마다(`:379,401,417,446,462`) 다시 돈다. 쓰임은 상위 후보 Select·조상 경로·손자 목록(`:508-528,803`)뿐이다. 같은 함수가 `listComponents`도 매번 다시 부른다.
- **이유**: 이슈가 2,000건인 프로젝트에서는 모달을 열 때마다 직렬 요청 10회와 수 MB JSON이 오간다. 상위 Select에 수천 개 옵션이 그려진다. 브랜치(`IssueDetailModal.tsx:174-182`)에도 그대로 있다.
- **제안(M)**: ① 상위 후보는 `searchIssues(text)` 기반 비동기 검색 Select로 바꾼다(DS Select가 비동기 검색을 못 하면 링크 추가 다이얼로그와 같은 검색 패턴을 쓴다). ② 조상 경로는 서버 `GET /api/alm/issues/{id}/ancestors`를 추가하거나 `parentId` 체인을 `getIssue`로 최대 깊이까지 따라간다. ③ 손자는 펼칠 때 `listChildren(childId)`로 지연 로드한다. ④ 컴포넌트·버전은 모달 수명 동안 1회만 읽는다.
- **브랜치**: 고쳐지지 않음.

## A-07 보드 이슈 API가 프로젝트 전체를 메모리에서 거름 — Important (성능)

- **근거**: `alm-backend/.../board/BoardService.java:123-149` `issues()`는 `findByProjectIdOrderBySortOrderAscKeyAsc(projectId)`로 **전 이슈**를 읽는다. 스크럼은 활성 스프린트 하나만 필요한데도 전체를 읽은 뒤 자바 스트림으로 스프린트·담당자·타입·라벨을 거른다. 칸반은 완료된 이슈까지 영구히 전부 내려준다. main은 `IssueResponse.from`이 라벨·컴포넌트를 이슈마다 지연 로딩한다(N+1).
- **이유**: 보드는 DnD·인라인 생성마다 재조회한다(`BoardPage.tsx:75-99,233`). 규모가 커지면 가장 자주 도는 경로가 가장 무겁다.
- **제안(M)**: 리포지토리 술어로 내린다(`sprintId = :active`, `assigneeId in`, `type in`, 라벨은 `EXISTS`). 칸반은 지라처럼 "완료 상태는 최근 N일(기본 14일)만" 옵션을 보드 설정에 추가한다(기본값 결정은 기획 확인 필요). 응답 매핑은 브랜치의 `IssueResponses` 일괄 로더를 재사용한다.
- **브랜치**: `1364aee`가 일괄 매핑과 `@BatchSize`로 N+1만 완화. 전체 적재는 남아 있음.

## A-08 리포트: 변경 이력 무제한 + 실패 시 무한 스피너 — Important (성능/오류 전파)

- **근거**
  - `alm-front/src/features/alm/pages/ReportsPage.tsx:93-112`: `listProjectChanges(projectId)`를 `since` 없이 부른다(`jiraApi.ts:725-733`). `Promise.all`에 catch가 없어, 하나라도 실패하면 `issues`가 `null`로 남아 로딩 상태가 고정되고 처리되지 않은 rejection이 난다.
  - 백엔드 `history/IssueChangeLogService.java:65-69` / `IssueChangeLogRepository.findHistory`는 limit·페이징이 없다. 프로젝트 전체 이력을 매번 반환한다.
- **이유**: 변경 이력은 단조 증가한다(이슈 1건당 수십 행). 1년 운영하면 리포트 진입마다 수만 행이 오간다. 503이 나도 사용자는 "로딩 중"만 본다.
- **제안(S~M)**: 프론트는 리포트 종류별로 필요한 창만 요청한다(번다운은 `sprintId`, 누적 흐름은 `since=선택 기간`). catch를 추가해 `ErrorState`와 재시도를 보여 주고, 503 문구는 서버 것을 쓴다. 백엔드는 `since` 미지정 시 기본 90일과 상한(예: 20,000행)을 두고, 넘으면 400 또는 잘림 플래그를 준다.
- **브랜치**: 고쳐지지 않음(브랜치 ReportsPage에도 catch 없음).

## A-09 200건 조용히 잘림 — Important (정합성)

- **근거**: `jiraApi.ts:2394-2403` `listChildren`(`size=200`, total 무시), `:2415-2427` `versionProgress`(`size=200`이라 진행률 분모가 틀린다), `:388-403` `queryIssues`(main, 검색 페이지). 서버 `IssueSearchService.MAX_SIZE=200`(`:39`)이다.
- **이유**: 에픽 하위가 200개를 넘으면 목록이 조용히 잘린다. "결과 200건"이 실제 total처럼 보인다.
- **제안(S)**: `listIssues`처럼 total까지 페이지를 돌거나, UI에 "N건 중 200건"과 "더 보기"를 둔다. 진행률은 브랜치의 `/issues/aggregate`(fixVersionId + statusCategory 그룹)로 서버에서 센다.
- **브랜치**: 검색 페이지 서버 페이징은 `994e7d4`가 고침. `listChildren`·`versionProgress`는 브랜치(`api_b:2517,2538`)에도 그대로 있음.

## A-10 `statusMetaByProject` 1+N 팬아웃 — Important (성능)

- **근거**: `jiraApi.ts:1581-1590`은 `listProjects()` 뒤 **프로젝트마다** `GET /projects/{id}/settings`를 부른다. 호출처는 `HomePage`, `SearchPage:116`, `SearchModal:43`(⌘K를 열 때마다, 브랜치 기준), `versionProgress`다.
- **이유**: 프로젝트가 50개면 검색 모달을 열 때마다 요청 51회가 나간다. 스킴은 공유되는 경우가 대부분이라 같은 본문을 반복해서 받는다.
- **제안(S~M)**: 서버에 `GET /api/alm/projects/settings/statuses`(접근 가능한 프로젝트 → 해석된 상태 맵, 스킴 단위 중복 제거)를 추가한다. 그 전까지는 프론트에서 세션 메모(스킴 변경 이벤트 시 무효화)로 막는다.
- **브랜치**: 고쳐지지 않음.

## A-11 일괄 편집: 이슈당 요청 3회 순차 실행, 실패 행 키 표시 버그 — Important (성능/버그)

- **근거**: `jiraApi.ts:797-822` `bulkUpdateIssues`는 이슈마다 `issueDto`(GET) → `updateIssue` 내부 GET → PUT을 **직렬로** 보낸다. 실패 행은 `{ id, key: id }`로 쌓여 키 대신 숫자 id가 보인다(`:818-819`). `bulkDeleteIssues`(`:825-841`)도 같은 패턴이다.
- **이유**: 100건 일괄 변경이면 요청 300회가 직렬로 나간다. 실패 보고서의 "키"가 숫자라 사용자가 어떤 이슈인지 알 수 없다.
- **제안**: (S) 실패 행에 이미 읽은 `current.key`를 넣고, 동시성 4~6의 풀로 병렬화한다. (M) 서버에 `POST /api/alm/issues/bulk`(행별 결과·expectedVersion 포함·행별 트랜잭션)를 추가한다.
- **브랜치**: `994e7d4`가 편집 시작 version 전달만 추가(`api_b:920-946`). 직렬 실행과 `key: id`는 그대로 있음.

## A-12 목업이 프로덕션 번들에 포함됨 — Important (번들)

- **근거**: `alm-front/src/features/alm/store/jiraStore.ts:11-12,34`는 `import * as mock` + `{ ...mock, ...rest }`다. 네임스페이스 전개라 트리 셰이킹이 불가능하다. `jiraApi.ts:107-113`도 `jiraMock`에서 상수·헬퍼를 import한다. 실측으로 `dist/assets/index-LnSXvOwK.js`(712KB, 브랜치 빌드)에 목업 전용 문자열 `"alm.jira.v1"`, `"alm.jira.ui.v1"`, `__resetForTest`가 들어 있다. REST에 없는 파사드 함수는 `defaultBoard`·`formatAvatarLimit`·`AVATAR_CHANGED_EVENT` 3개뿐이다(계산 결과).
- **이유**: 5.5k줄 목업, 시드, 샘플 데이터가 모든 사용자의 초기 로딩에 실린다. 성능 브랜치의 라우트 분할 효과를 깎는다.
- **제안(M)**: 상수·헬퍼(`assertAvatarFile`, 이벤트 이름, `DEFAULT_PREFERENCES`, `defaultBoard`)를 `store/shared.ts`로 옮긴다. 파사드는 빌드 타임에 고른다(vite `resolve.alias`로 `@impl` → `jiraApi`/`jiraMock`, 또는 `import.meta.env.PROD ? await import(...)`). `{...mock, ...rest}` 폴백은 제거하고, REST 누락은 타입(`satisfies typeof mock`)으로 잡는다. 빌드 후 `"alm.jira.v1"` 부재를 CI에서 grep으로 확인한다.
- **브랜치**: 고쳐지지 않음.

## A-13 인덱스 누락·본문 `%like%` 스캔 — Important (성능/DB)

- **근거**: 마이그레이션 전체에 `issue(assignee_id)`, `issue_label(label)`, `issue(reporter_id)` 인덱스가 없다(`V1__init.sql:22`, `V2__issue_details.sql:24-29`는 PK `(issue_id, label_order)`뿐). 홈의 "나에게 배정", 담당자 필터, AQL `assignee`, 라벨 필터·라벨 API가 전 테이블을 스캔한다. 텍스트 검색은 `lower(description) like '%x%'`다(`IssueSearchService.java:83-88`). 사용자 입력의 `%`·`_`도 이스케이프하지 않는다.
- **제안(S, Flyway V27)**: `CREATE INDEX idx_issue_assignee ON issue(assignee_id, project_id) WHERE archived_at IS NULL;`, `CREATE INDEX idx_issue_label_label ON issue_label(label);`. 텍스트는 PG `pg_trgm` GIN(`title`, `description`)을 검토한다(설치 옵션 `SEARCH_MODE=lite`의 경로이므로 효과가 크다). LIKE에는 `escape '\'`를 적용한다. 적용 전후로 `EXPLAIN ANALYZE`를 실측한다.
- **브랜치**: 고쳐지지 않음.

## A-14 CSV 가져오기 항목 수 상한 없음 — Important (DoS)

- **근거**: `issue/dto/IssueImportRequest.java:6`은 `@NotEmpty`만 있고 `@Size(max)`가 없다. main에서는 한 트랜잭션이 프로젝트 행 잠금(`IssueService.java:96`)을 쥔 채 전 항목을 처리하므로, 그동안 같은 프로젝트의 생성·수정·이동이 모두 대기한다. 브랜치는 행별 트랜잭션이라 잠금 점유는 짧아졌지만, 요청 하나가 무한정 길게 도는 문제는 남는다.
- **제안(S)**: `@Size(max = 5000)`(값은 기획 확인)을 걸고, 프론트 CSV 파서에도 같은 상한과 안내 문구를 둔다.
- **브랜치**: 상한은 없음(잠금 점유만 완화).

## A-15 홈·대시보드 가젯의 전 프로젝트 × 전 이슈 적재 — Important (성능), 브랜치에서 고침

- **근거(main)**: `alm-front/src/features/alm/pages/HomePage.tsx:58-65`는 `projectList.map((p) => listIssues(p.id))`다. `DashboardGadgets.tsx:249-250`도 같다. `docs/areas/README.md`의 "알려진 이슈 2번(N+1)"이 이것이다.
- **브랜치**: 백엔드 `1364aee`(`/issues/aggregate`, 라벨 API)와 프론트 `994e7d4`(집계·조건 검색 소비)가 고친다. 단, `DashboardGadgets`의 `listIssues(projectId)` 1건(`:218`)은 남아 있다.
- **제안**: A-04와 함께 병합한다. 병합 후 `docs/areas/README.md`의 알려진 이슈 2번을 지운다.

## A-16 프로젝트 마지막 관리자 보호가 클라이언트에만 있음 — Minor (정합성, 서비스 경계)

- **근거**: `jiraApi.ts:2606-2634`(`assertNotLastAdmin` → org `PATCH/DELETE /api/org/grants/{id}`). org-service는 **전역** 마지막 관리자만 막는다(`platform-backend/org-service/.../grant/GrantService.java:86`).
- **이유**: 두 관리자가 동시에 서로를 강등하거나, org 관리 화면·API로 직접 지우면 관리자 없는 프로젝트가 생긴다(전역 관리자만 복구할 수 있다).
- **제안(S, org-service 과제)**: org `GrantService`에 리소스(PROJECT/SPACE) 단위 마지막 ADMIN 보호(409)를 추가하고, ALM은 그 문구를 그대로 띄운다. 플랫폼 백엔드 소관이므로 기획에 에스컬레이트한다.

## A-17 댓글·워크로그 정리 권한 공백 — Minor (기능/권한)

- **근거**: `collab/CollaborationService.java:107-127,145-150`. 수정·삭제는 작성자 일치만 보고 프로젝트 권한은 다시 보지 않는다. 프로젝트 ADMIN도 남의 부적절한 댓글을 지울 수 없다.
- **제안(S)**: 삭제는 "작성자 AND 현재 VIEW" 또는 "프로젝트 ADMIN"으로 바꾼다(지라 "모든 댓글 삭제" 권한과 같다). 수정은 작성자 AND 현재 VIEW로 한다.

## A-18 서버가 클라이언트의 `doneStatuses`를 믿음 — Minor (정합성)

- **근거**: `sprint/SprintService.java:128-131`, `version/VersionService.java:101-104`. V11 이후 서버가 `completeStatusIds`를 알고 있는데도, 요청에 값이 있으면 그것으로 미완료 이관 대상을 정한다.
- **이유**: 오래된 클라이언트나 잘못된 호출이 완료 이슈를 다음 스프린트·버전으로 옮겨 리포트(번다운·벨로시티)를 오염시킨다.
- **제안(S)**: 요청 필드를 무시(또는 deprecated 후 제거)하고 서버 판정만 쓴다. 프론트 `SprintCompleteModal`·`VersionReleaseModal`의 전송을 제거한다.

## A-19 이동·랭크는 version을 올리지 않음 — 전체 본문 PUT의 되돌림 위험 — Minor (정합성)

- **근거**: `domain/Issue.java` `moveTo`·`rankTo`는 의도적으로 version을 유지한다(주석). 반면 `PUT /issues/{id}`는 전체 교체다(`IssueService.update`, `status`가 null이 아니면 덮어쓴다).
- **이유**: 프론트는 최신 GET과 병합해 보내므로 안전하다. 그러나 PAT/MCP 에이전트처럼 "읽어 둔 스냅샷으로 전체 본문 PUT"을 하는 외부 클라이언트는 그 사이의 보드 이동(상태·스프린트)을 409 없이 조용히 되돌린다.
- **제안(S)**: README·OpenAPI에 "PUT은 전체 교체이며 status·sprintId는 이동 API와 version을 공유하지 않는다"를 명시한다. 장기적으로 `PATCH`(보낸 필드만 반영)를 추가하고 MCP 도구는 PATCH를 쓴다. version 정책 변경은 사용자 결정 사항이다(기존 설계 주석과 충돌하므로).

## A-20 링크 목록 N+1·순환 검사 체인 쿼리 — Minor (성능)

- **근거**: `CollaborationService.java:176-183`(링크마다 `issues.findById`와 `linkTypes.findById`), `IssueService.java:448-456` `isDescendant`(부모 체인마다 `findById`. 계층 무제한이라 깊이만큼 쿼리가 나간다).
- **제안(S)**: 반대편 id를 모아 `findAllById` 1회, 링크 타입은 맵 캐시 1회로 바꾼다. 순환 검사는 PG 재귀 CTE(`WITH RECURSIVE`) 1쿼리로 바꾼다.

## A-21 API는 있는데 UI가 없음, 죽은 코드 — Minor (기능 공백/위생)

- **근거**: 파사드 export 중 화면에서 쓰지 않는 것: `updateVersion`, `updateProjectShortcut`, `versionProgress`(main `src` 전수 검색). `ReleasesPage.tsx:17-25`는 create·archive·delete·release만 import하고, 버전 이름·날짜·설명을 수정할 진입점이 없다. 백엔드 `PUT /api/alm/versions/{id}`와 `expectedVersion` 검증은 구현돼 있다.
- **제안**: (S) 릴리스 행 ⋯ 메뉴에 "편집"을 추가한다(지라와 같다). 바로 가기 편집도 같다. 쓰지 않는 `versionProgress`는 집계 API로 대체하거나 삭제한다.

## A-22 위생: 거대 파일, 이중 구현, 목업만 검증하는 테스트 — Minor (위생)

- **근거**: `jiraMock.ts` 5,522줄, `jiraApi.ts` 3,068줄, `app/app.css` 5,657줄, `IssueDetailModal.tsx` 1,319줄, `types.ts` 1,186줄. AQL은 서버(`search/aql/*`)와 프론트 목업용 `store/aql/evaluate.ts`(799줄)에 이중으로 구현돼 있다. 워크플로 전이·필수 필드·라벨 규칙도 목업에 복제돼 있다. `jiraStore.ts:28-31`은 "테스트는 항상 목업"이라, 화면 테스트 1,200여 개가 REST 어댑터 경로를 타지 않는다(`jiraApi.*.test.ts`의 fetch 목만 계약을 본다). `alm-front`에 에이전트 오피스 UI가 2.5만 줄(`src/features/alm/office`) 함께 있다.
- **이유**: 목업과 서버 규칙이 어긋나도 테스트는 녹색이다(예: A-04의 version 의미는 목업과 서버가 달랐다).
- **제안**: (M) `jiraApi.ts`를 도메인별(`api/issues.ts`, `api/settings.ts`, `api/agents.ts`…)로 나누고 `jiraMock`도 같은 경계로 나눈다. (M) 핵심 플로(이슈 생성·편집 409·보드 이동·검색)에 대해 MSW로 서버 응답 shape을 재현한 REST 모드 화면 테스트 소수를 추가한다. (L, 결정 필요) 오피스 UI의 리포 경계는 사용자에게 확인한다(지금은 P4c 미커밋 작업 중이라 손대지 않는다).

## A-23 문서 드리프트 — Minor (문서)

| # | 위치 | 내용 | 실제 |
|---|---|---|---|
| 1 | `alm-front/src/features/alm/store/jiraStore.ts:4-7` 헤더 주석 | "코멘트·보드·링크·워크로그·멤버·사용자·활동은 목업으로 떨어진다" | 전부 REST다(누락은 상수 3개뿐) |
| 2 | `alm-front/docs/areas/store.md:7` vs `:12` | "REST에 아직 없는 함수는 목업으로 떨어진다" ↔ "REST 미구현 함수는 이제 없다" | 앞 문장이 낡았다 |
| 3 | `docs/areas/store.md:9-10` | "역할 변경은 grant 삭제 후 재생성" | `jiraApi.ts:2620-2626`은 `PATCH` |
| 4 | `alm-front/docs/BACKLOG.md:4` | "현재 화면은 전부 jiraStore(localStorage 목업)만 호출" | 프로덕션은 REST |
| 5 | `docs/BACKLOG.md:32` | 메일은 `ALM_MAIL_HOST` 기반 | 09-07 이후 org-service 메일 허브(`OrgMailClient`) |
| 6 | `alm-backend/README.md:33,52-59` | common-proto `0.6.0` | `build.gradle:38` 기본값 `0.16.0`(common-starter 동일) |
| 7 | `alm-backend/.../issue/dto/IssueDetailsRequest.java:15` OpenAPI | "에픽→일반→하위 2단계까지만 허용" | `IssueService.java:427-430`은 계층 무제한(확정 결정) |

추가로 `docs/STATUS.md` 머리말은 "2026-08-16 기준"인데 본문은 10월 내용까지 담고 있다. **제안(S)**: 위 7곳을 한 커밋으로 정정한다. 병합 후 테스트 수(브랜치 `8cc6370`: 1266)도 README·STATUS에 반영한다.

---

## 권장 처리 순서

1. **A-01**(Critical, S) — 즉시 단독 핫픽스. 회귀 테스트 포함.
2. **A-04 / A-15** 브랜치 병합(PG 실측 선행 = A-05의 최소판) — main lost update 해소.
3. **A-02, A-03**(권한 경계, S~M).
4. 성능 묶음 **A-06, A-07, A-10, A-13**(규모 대응. V27 인덱스는 백엔드 병합 뒤 번호 확정).
5. **A-08, A-09, A-11, A-14**.
6. **A-12** 번들 분리, **A-05** PG CI 상시화.
7. Minor 정리(A-16은 org-service로 에스컬레이트, A-19는 사용자 결정).

# WIKI 제품 감사 (2026-10-18)

대상: `wiki-front`(체크아웃 `perf/wiki-loading-responsive-layout`, 미머지), `wiki-backend`(main 기준, 미머지 `perf/wiki-search-permission-pagination` 956f09a·475fc62 대조), `collaboration-service`(main).
읽기 전용으로 감사했다. 테스트는 돌리지 않았고 코드를 읽어 판정했다. 근거는 모두 main 기준 `파일:줄`이다(프론트는 현재 체크아웃 기준).
확정 설계 결정(루트 CLAUDE.md)을 뒤집는 제안은 넣지 않았다.

미머지 브랜치가 이미 고친 것(이 목록에서 뺐다): 라이트 검색의 "후보 500건 자른 뒤 권한 필터" 누락, PG 날짜 필터 null 타입 오류, 이력·댓글 전량 목록(쪽 목록 추가), TreeService 본문 과조회, 프론트 라우트 lazy 분할(main은 lazy 1개), 트리 로딩 최적화, 알림 폴링, 스페이스 라벨(V41).

심각도 표기: Critical / High / Medium / Low. 크기 표기: S(반나절 이하) / M(1~2일) / L(3일 이상).

---

## 요약 순위표

| ID | 분류 | 심각도 | 한 줄 | 브랜치에서 수정? |
|---|---|---|---|---|
| W-01 | 정확성 | High | 일반 저장(PUT)이 작업 표를 다시 만들지 않음. 줄이 밀리면 다른 사람의 체크박스가 토글됨 | 아니오 |
| W-02 | 정확성(PG) | Critical | `page_type_check`가 BLOG를 거부해 운영 PG에서 블로그 저장 실패(알려진 결함, V42 필요) | 아니오 |
| W-03 | 테스트 위생 | High | 테스트가 H2 `create-drop`이라 Flyway 제약·FK를 검증하지 않음. W-02가 CI를 통과한 원인 | 아니오 |
| W-04 | 보안 | High | 일일 요약 메일이 발송 시점에 권한을 다시 확인하지 않음. 제한된 문서 제목이 메일로 샘 | 아니오 |
| W-05 | 데이터/보안 | High | 영구 삭제·스페이스 삭제가 첨부 지난 버전 객체, Yjs 상태, 리액션을 남김 | 아니오 |
| W-06 | 보안 | High | PDF 내보내기가 본문 이미지 URL을 서버에서 가져옴(SSRF) | 아니오 |
| W-07 | 정확성 | Medium | "보관 문서는 편집 불가" 규칙이 복원·공동 초안 게시·이동 등 여러 쓰기 경로에서 빠짐 | 아니오 |
| W-08 | 정확성 | Medium | 휴지통 복원 시 부모가 다른 스페이스로 옮겨졌으면 스페이스를 넘는 부모-자식 관계가 생김 | 아니오(브랜치는 숨김 계산만 보강) |
| W-09 | 보안 | Medium | 콘텐츠 gRPC(9111)가 인증 없이 전 문서를 주고, 모든 인터페이스에 바인드되며, lite 모드에서도 켜져 있음 | 아니오 |
| W-10 | 신뢰성 | Medium | 팀·주체 디렉터리 gRPC 호출에 데드라인이 없음. org가 멈추면 요청 스레드가 무기한 대기 | 아니오 |
| W-11 | 성능 | Medium | 알림 목록이 안 읽은 알림 전체와 페이지 본문을 매번 읽음. 발송은 수신자마다 권한 질의 | 아니오(브랜치의 60초 폴링으로 호출이 늘어남) |
| W-12 | 성능 | Medium | `listUsers()`가 조직 전체 멤버를 화면마다 캐시 없이 받음(16곳) | 아니오 |
| W-13 | 성능 | Medium | 홈이 스페이스 수만큼 요청을 보내고, 최근 방문 6건마다 본문을 받음 | 아니오 |
| W-14 | 성능 | Medium | 스페이스·하위 포함 내보내기가 문서마다 `getPage`를 동시 실행 수 제한 없이 보냄 | 아니오 |
| W-15 | 성능 | Medium | 구조 변경 경로의 N+1: `collectSubtree`가 노드마다 `findById`, 스페이스 삭제는 페이지마다 쿼리 5회 | 아니오 |
| W-16 | 성능 | Medium | 블로그 목록이 페이지 나눔 없이 전 글의 본문을 읽고 정규식 9개를 돌림 | 아니오 |
| W-17 | 기능/데이터 손실 | Medium | 편집 중 앱 안 이동(사이드바 클릭)에는 이탈 가드가 없어 미저장 편집이 사라짐 | 아니오 |
| W-18 | 경계면 | Low | `listTeams`·`listUsers`가 503을 빈 목록으로 삼킴. 제한 다이얼로그에서 원인이 안 보임 | 아니오 |
| W-19 | 성능 | Low | `updatePage`가 PUT마다 본문 포함 GET을 한 번 더 보냄 | 아니오 |
| W-20 | 정확성 | Low | 라이트 검색 LIKE에서 `%`·`_`를 이스케이프하지 않음 | 아니오(브랜치도 동일) |
| W-21 | 정확성 | Low | `recordVisit` 경합으로 PK 충돌 시 조회수 증가까지 롤백 | 아니오 |
| W-22 | 기능(정책) | Low | 초안이 스페이스 열람자 전원에게 보이고 알림·검색에도 나옴. 사용자 결정 필요 | 해당 없음 |
| W-23 | 위생 | Low | 거대 파일(wikiApi 1562줄, wikiMock 2470줄, types 921줄, PageService 738줄 + FQCN 남발) | 아니오 |
| W-24 | 문서 드리프트 | Low | wiki-front CLAUDE.md 낡음, wiki-backend CLAUDE.md 없음, 루트 목록에 collaboration-service 없음 | 아니오 |

---

## 상세

### W-01 [정확성 · High] 일반 저장이 작업(W23) 표를 갱신하지 않음. 체크 토글이 엉뚱한 줄을 바꿈
- **근거**
  - `wiki-backend/src/main/java/com/platform/wikibackend/page/PageService.java:397-422` `update()`에 `taskSync.sync(p)`가 없다. 같은 파일의 create(`:90`), copy(`:152`), 공동 초안 게시(`:451`), 복원(`:733`)은 모두 호출한다.
  - `task/TaskService.java:146-150` `setDone`은 그 줄이 "어떤 작업 줄"인지만 보고 "같은 작업"인지는 보지 않는다.
  - `src/test/.../task/TaskApiTest.java` 테스트 5개 중 수정 경로를 다루는 것이 없다.
- **왜 중요한가:** 공동 편집 플래그가 꺼져 있어 거의 모든 저장이 PUT을 탄다. 저장 뒤 "내 작업"(`/tasks`)은 옛 본문 기준으로 남는다. 지운 항목이 그대로 보이고, 새 항목과 담당자·기한 변경은 반영되지 않는다. 위에 줄을 하나 넣은 뒤 목록에서 체크하면 lineNo가 가리키는 다른 작업 줄이 토글되고, 그 결과가 새 버전과 리비전으로 저장된다. 남의 문서 내용을 조용히 바꾸는 결과다.
- **수정안 (S):** `update()`에서 `revisions.save(...)` 직후 `taskSync.sync(p)`를 호출한다. 회귀 테스트도 추가한다(작성 → PUT으로 위에 줄 삽입 → `/tasks`의 lineNo 갱신 확인, 옛 lineNo로 토글하면 409). 더 안전하게 하려면 `setDone`이 요청에 항목 텍스트 해시를 받아 대조하게 한다(M).
- **브랜치:** 수정 안 됨.

### W-02 [정확성(PG) · Critical] BLOG 저장이 운영 PostgreSQL에서 실패 (알려진 결함)
- **근거:** `wiki-backend/src/main/resources/db/migration/V2__page_type_status.sql:12`에 `CHECK (type IN ('PAGE','FOLDER'))`. `domain/PageType.java:13-16`에는 BLOG가 있다. 이 제약을 바꾼 마이그레이션은 V3~V41 어디에도 없다.
- **왜 중요한가:** W24 블로그(프론트 `BlogPage`, `PageService.create`의 BLOG 분기)가 운영에서 전부 제약 위반으로 실패한다.
- **수정안 (S):** `V42__page_type_blog.sql`로 `ALTER TABLE page DROP CONSTRAINT page_type_check, ADD CONSTRAINT page_type_check CHECK (type IN ('PAGE','FOLDER','BLOG'))`. V41이 미머지 브랜치에 있으므로 번호 순서는 머지 순서와 맞춘다(브랜치 먼저 머지 후 V42). PG 회귀 테스트(`FlywaySchemaValidationTest`에 BLOG insert)를 함께 넣는다.
- **브랜치:** 수정 안 됨(475fc62 커밋 메시지가 "main의 기존 결함"으로 기록).

### W-03 [테스트 위생 · High] H2 `create-drop` 테스트가 실제 스키마를 검증하지 않음
- **근거**
  - `wiki-backend/src/test/resources/application-test.yml:10` `ddl-auto: create-drop`(Flyway 미적용), `:4` H2 MODE=PostgreSQL.
  - 코드 주석이 이 차이를 전제로 우회하고 있다. `TrashService.java:253` "H2 테스트 스키마엔 FK가 없다", `AttachmentService.java:310-312`.
  - PG 검증은 `FlywaySchemaValidationTest` 하나뿐이다.
- **왜 중요한가:** CHECK 제약(W-02), FK cascade 순서, 네이티브 쿼리의 null 타입(475fc62가 고친 결함)이 모두 H2에서는 통과하고 운영에서 깨진다. main CI가 초록인데 블로그가 깨져 있는 이유가 이것이다.
- **수정안 (M):** 적어도 Repository·서비스 쓰기 경로 테스트를 Testcontainers PG + Flyway 프로필로 한 번 더 돌린다. 예: JUnit 태그 `@Tag("pg")`로 묶고 CI에서 실행한다(CI 러너에 이미 Docker가 있다, `.github/workflows/ci.yml:28-30`). 장기적으로는 H2를 버린다.
- **브랜치:** 해당 없음.

### W-04 [보안 · High] 일일 요약 메일이 권한을 다시 확인하지 않음
- **근거:** `wiki-backend/src/main/java/com/platform/wikibackend/notification/NotificationDigestService.java:47-66`. 알림 행의 page를 읽어 `page.getTitle()`을 그대로 메일에 넣는다. space VIEW와 effective VIEW 재판정이 없다. 대조: 즉시 알림은 `NotificationService.java:177-185`에서, 알림함은 `:211-225`에서 "제한 변경 직후의 알림도 제목·경로를 노출하므로" 다시 판정한다.
- **왜 중요한가:** 알림이 만들어진 뒤 24시간 안에 문서에 VIEW 제한이 걸리거나 스페이스 권한이 회수돼도, 다음날 09시 요약 메일에 제한 문서의 제목과 링크가 나간다. 프로젝트가 다른 두 경로에서 일부러 막은 바로 그 누출이다.
- **수정안 (S):** 요약 루프에서 사용자별로 `effective.viewablePageIds(userId, pages)`를 구하고, 스페이스별로 `permissions.isAllowed(userId, spaceId, VIEW)`를 판정해 거른다(`NotificationService.list`의 판정 블록을 공용 메서드로 뽑아 재사용). org 불능(503)이면 그 사용자는 건너뛰고 `markEmailed`를 하지 않는다. 테스트: 알림 생성 → 제한 적용 → run() → 메일 본문에 제목 없음.
- **브랜치:** 수정 안 됨.

### W-05 [데이터/보안 · High] "되돌릴 수 없는 삭제"가 일부 데이터를 남김
- **근거**
  - `page/TrashService.java:241-270` `hardDelete`: 현재 첨부 객체만 `deleteAfterCommit`. 지난 버전 객체(`attachment_version`, V20)는 지우지 않는다. 대조: 단건 삭제 `AttachmentService.java:314-318`은 버전 객체까지 지운다.
  - `space/SpaceService.java:111-139` 스페이스 삭제도 같다. 버전 객체를 지우지 않고, 리액션(FK 없음, `TrashService.java:255`의 주석이 이유)도 지우지 않는다.
  - `collaboration_document`(Yjs 바이너리 상태, `V5__collaboration_document_metadata.sql:4` room text PK, page FK 없음)는 어느 삭제 경로에서도 지우지 않는다. `collaboration-service/src`에도 DELETE가 없다.
- **왜 중요한가:** 휴지통 영구 삭제의 계약은 "첨부 객체까지 되돌릴 수 없게 지운다"(`TrashService.java:127`)인데, 지난 버전 파일과 공동 편집 초안 본문(CRDT 상태)이 저장소와 DB에 무기한 남는다. 개인정보나 기밀 삭제 요청에 응할 수 없다. 저장소 용량도 샌다. 같은 id가 재사용되지 않으므로 기능상 증상은 없어서 발견이 어렵다.
- **수정안 (M):** 삭제 정리를 `PageDataPurger` 같은 한 곳으로 모으고, TrashService와 SpaceService가 공유하게 한다(현재 두 곳에 비슷한 목록이 복제돼 있고 서로 어긋나 있다). 정리 대상은 첨부와 버전 객체, 리액션(PAGE와 COMMENT), `collaboration_document where room = 'page:'||id`, 작업, 라벨, 링크, 구독, 방문. 이미 남은 고아는 `AttachmentReconciliationService`에 버전 객체 대조를 추가해 회수한다.
- **브랜치:** 수정 안 됨(브랜치 SpaceService 변경은 라벨 삭제만 추가).

### W-06 [보안 · High] PDF 내보내기의 서버 측 요청 위조(SSRF)
- **근거:** `export/MarkdownPdfRenderer.java:72-80`. `PdfRendererBuilder`에 `useUriResolver`나 `useHttpStreamImplementation` 제한이 없다. flexmark는 `![x](http://...)`을 `<img src>`로 만들고(ESCAPE_HTML은 raw HTML만 막는다), openhtmltopdf의 기본 스트림 구현은 http(s) URL을 서버에서 연다.
- **왜 중요한가:** EDIT 권한이 있는 사용자는 본문에 `![](http://org-service:9130/...)`나 `http://169.254.169.254/...` 같은 이미지를 넣고 PDF를 내보내는 것만으로 컨테이너 내부망에 GET을 보낼 수 있다. 응답이 이미지면 PDF에 그대로 박혀 돌아온다. 반대로 위키 자체 첨부 이미지(`/api/wiki/attachments/{id}/inline`, 상대 경로)는 base URI가 null이라 PDF에 나오지 않는다. 기능 결함이기도 하다.
- **수정안 (M):** URI 해석기를 하나 설치한다. 위키 첨부 inline 경로만 허용하고, 해당 첨부의 페이지 VIEW 판정 뒤 `AttachmentStorageRouter`에서 바이트를 직접 읽어 data URI로 넣는다. 나머지 스킴과 호스트는 전부 거부한다(빈 이미지). 테스트: 외부 URL 이미지는 요청이 나가지 않고, 첨부 이미지는 PDF에 포함.
- **브랜치:** 해당 없음.

### W-07 [정확성 · Medium] 보관(Archive) 불변식이 일부 쓰기 경로에만 있음
- **근거:** `isArchived()` 쓰기 차단은 `PageService.update`(`PageService.java:402`)와 `TaskService.setDone`(`TaskService.java:142`), 보관 아래 생성(`PageService.java:83`)뿐이다. 다음 경로는 검사 없이 버전을 올리거나 상태를 바꾼다.
  - `PageService.restore`(`:720-737`)
  - `PageService.commitCollaborationDraft`(`:428-457`)
  - 협업 티켓 발급(`PageService.getEditable` `:612-617`)
  - `move`의 보관 부모 아래 이동(`:250-307`)
  - `publish`·`setIcon`·첨부 업로드·라벨 변경
- **왜 중요한가:** 화면은 보관 문서를 읽기 전용으로 보여 주지만, 이력 화면의 "이 버전으로 복원"이나 이미 열린 공동 편집 세션으로 보관 문서가 계속 바뀐다. 보관 해제 전까지 변경이 없다는 전제(감사, 검증 배지)가 깨진다.
- **수정안 (S~M):** `PageService`에 `requireWritable(Page)`(삭제·보관 검사)를 두고 모든 버전 증가 경로와 메타 변경 경로에서 부른다. 협업 티켓 발급도 거부한다. 경로별 409 테스트를 추가한다.
- **브랜치:** 수정 안 됨.

### W-08 [정확성 · Medium] 휴지통 복원이 스페이스를 넘는 부모를 그대로 연결
- **근거:** `page/TrashService.java:108-110`은 부모가 존재하는지만 본다(`pages.findById(parentId).isEmpty()`). 부모가 그사이 `moveToSpace`로 다른 스페이스에 갔으면 복원된 문서는 spaceId=A, parentId=(B 스페이스 문서)가 된다. 브랜치 956f09a 메시지는 이 상태("다른 스페이스로 넘어가는 조상 체인(부모 이동 뒤 휴지통 복원)")를 권한 숨김 계산에 넣어 우회했을 뿐, 이 상태가 생기는 것 자체는 막지 않았다.
- **왜 중요한가:** A 스페이스 트리에서는 부모가 없어 고아가 되고(루트에도 안 보임), B 스페이스의 VIEW 제한이 A 문서에 상속되는 등 트리, 권한, 이동 검증이 모두 "서브트리는 한 스페이스 안에 있다"는 전제(`PageService.java:371`)와 어긋난다.
- **수정안 (S):** `reparented = parent 없음 || !parent.spaceId.equals(root.spaceId) || parent.isArchived()`로 바꿔 루트로 올린다. 이미 생긴 데이터는 일회성 점검 SQL(`page p join page q on p.parent_id=q.id where p.space_id<>q.space_id`)로 확인한다.
- **브랜치:** 근본 원인은 수정 안 됨.

### W-09 [보안 · Medium] 콘텐츠 gRPC 서버: 무인증, 전 인터페이스 바인드, lite 모드에서도 기동
- **근거:** `grpc/WikiContentGrpcService.java:6-10` 주석 "권한을 검사하지 않는다 … 컨테이너 내부망에만 노출". `grpc/GrpcServerLifecycle.java:35` `ServerBuilder.forPort(port)`(0.0.0.0). `application.yml:84-86` `WIKI_GRPC_ENABLED:true` 기본값.
- **왜 중요한가:** 보호 수단이 네트워크 격리 하나뿐이다. dev 오프셋 클러스터(호스트 bootRun)나 같은 compose 망의 다른 컨테이너(예: agent-runner)에서는 전 스페이스의 본문, 라벨, 첨부 메타를 권한 없이 읽을 수 있다. `SEARCH_MODE=lite`(기본)에서는 search-service가 없으므로 이 서버가 필요하지 않다.
- **수정안 (S):** ① 내부 import API와 같은 공유 비밀(`X-Internal-Token` → gRPC metadata)을 `ServerInterceptor`로 검사하고, 토큰이 비면 서버를 기동하지 않게(fail-closed) 한다. ② 바인드 주소를 설정으로 받는다. ③ compose에서 `WIKI_GRPC_ENABLED`를 `SEARCH_MODE` 파생값으로 둔다(확정 결정 "모드 스위치 하나"와 일치).
- **브랜치:** 해당 없음.

### W-10 [신뢰성 · Medium] 팀·주체 디렉터리 gRPC에 데드라인 없음
- **근거:** `config/SecurityConfig.java:94-103`은 `GrpcTeamDirectory`와 `GrpcPrincipalDirectory`에 데드라인 없는 stub을 넘긴다. `permission/GrpcTeamDirectory.java:36`, `GrpcPrincipalDirectory.java:45`. 대조: PermissionClient와 MemberDirectory는 `deadline-seconds`를 받는다(`:75-89`).
- **왜 중요한가:** TEAM 제한이 걸린 문서를 열 때마다 `teamsOf`를 거친다(`EffectivePermissionService.java:214`). org-service가 연결은 유지한 채 멈추면 UNAVAILABLE이 아니라 무기한 대기가 되어 톰캣 스레드가 고갈된다. 위키 전체가 멈추고, 503 전파 규약(장애 전파)도 작동하지 않는다.
- **수정안 (S):** 두 클라이언트도 `deadlineSeconds`를 받아 호출마다 `stub.withDeadlineAfter(...)`를 쓴다. DEADLINE_EXCEEDED는 기존 `isUnavailable` 경로(503)로 보낸다.
- **브랜치:** 해당 없음.

### W-11 [성능 · Medium] 알림 조회와 발송의 비용
- **근거**
  - `notification/NotificationService.java:203` `findByUserIdAndReadAtIsNull(userId)`가 안 읽은 알림을 상한 없이 전량 읽는다(건수만 필요한데 엔티티 전체).
  - `:209` `pages.findAllById`가 본문(content)까지 든 Page 엔티티를 읽는다.
  - 발송은 `:177-196`의 `deliver`가 수신자마다 `permissions.isAllowed`(gRPC)와 `effective.canView`(쿼리 2회), `prefs.immediateTarget`(쿼리)를 저장 트랜잭션 안에서 실행한다(`onPageUpdated` `:68-85`).
- **왜 중요한가:** 브랜치 d9e7264가 알림을 60초 폴링으로 바꿔서(`useNotificationFeed.ts:6`) 탭 하나당 분당 1회씩 이 비용이 든다. 알림이 쌓인 사용자(구독 스페이스가 많은 관리자)일수록 무거워진다. 스페이스 구독자가 수백 명이면 문서 저장 한 번이 수백 번의 쿼리와 gRPC 호출 뒤에 커밋된다.
- **수정안 (M):** ① 안 읽은 건수는 `count` 쿼리로 내고, 표시용 목록은 30건 쪽만 판정한다(건수는 "보이는 것만"이라는 의미를 지키려면 상한 있는 id 프로젝션 + 판정). ② 페이지는 `(id, spaceId, title)` 프로젝션으로 읽는다. ③ 발송 판정은 `viewablePageIds`처럼 수신자 묶음 단위로 한 번에 계산한다(principals만 사용자별, 체인 인덱스는 1회).
- **브랜치:** 수정 안 됨(폴링 추가로 영향이 커짐).

### W-12 [성능 · Medium] 조직 전체 멤버 목록을 화면마다 캐시 없이 받음
- **근거:** `wiki-front/src/features/wiki/store/wikiApi.ts:94-108` `listUsers()`가 `GET /api/org/members` 전량을 받는다(페이지 나눔 없음). 호출처는 16곳이다. `PageViewPage.tsx:200`, `NotificationBell.tsx:39`, `HomePage.tsx:129`, `SearchPage.tsx:112`, `FolderPage.tsx:52`, `SpaceIndexPage.tsx:115`, `PageEditPage.tsx:68`, `pageHistory.ts:133,215` 등이다. 메모이즈가 없다.
- **왜 중요한가:** 문서 하나를 열 때 PageViewPage와 NotificationBell 등에서 같은 전량 응답을 2~3번 받는다. 멤버 수천 명 규모의 온프렘에서는 문서 이동마다 수백 KB를 받고 org-service에도 부하가 간다. 같은 로직이 16번 복제돼 있다(`then(setUsers).catch(...)` 모양이 제각각이다).
- **수정안 (S):** 스토어 안에 in-flight 공유 + TTL(예: 5분) 캐시를 둔다(`wikiApi.listUsers`만 수정하므로 화면 무수정, 스토어 불변조건 준수). 이어서 `useUsers()` 훅으로 16곳을 정리한다(S). 장기적으로는 필요한 id만 조회하는 `GET /api/org/members?id=` 사용을 검토한다.
- **브랜치:** 수정 안 됨.

### W-13 [성능 · Medium] 홈 화면의 요청 팬아웃
- **근거:** `wiki-front/src/features/wiki/pages/HomePage.tsx:104-120` "최근 업데이트"가 접근 가능한 **모든 스페이스**마다 `listRecentlyUpdated`를 보낸다. `:81-83` "이어서 작업"은 localStorage 방문 6건마다 `getPage`(본문 포함)를 보낸다. 대조: 사이드바는 서버 원장 `listRecentPages`를 한 번 부른다(`GlobalSidebar.tsx:131-133`).
- **왜 중요한가:** 스페이스가 100개면 홈 진입이 100+6회의 요청이 되고, 게이트웨이 rate-limit에 걸릴 수 있다. 방문 기록이 기기마다 따로 남아 사이드바와 홈이 서로 다른 "최근"을 보여 준다.
- **수정안 (M):** 백엔드에 스페이스를 가로지르는 `GET /api/wiki/recent-updates?limit=`(접근 스페이스 IN + 제한 필터, TreeService.recentlyUpdated의 다중 스페이스판)를 둔다. 홈의 "이어서 작업"은 `listRecentPages`(서버 원장, 본문 없음)로 바꾼다. localStorage 방문 로그는 읽기 전용 인스턴스 폴백으로만 남긴다.
- **브랜치:** 수정 안 됨(d131e70은 HomePage를 건드리지 않음).

### W-14 [성능 · Medium] 내보내기의 무제한 동시 요청
- **근거:** `wiki-front/src/features/wiki/lib/exportContent.tsx:74` `Promise.all(ordered.map(getPage))`(하위 전부 동시). `:113-121` 스페이스 내보내기는 루트마다 그 과정을 반복한다. `:123` 건수 세기도 루트마다 `listDescendants`를 부른다.
- **왜 중요한가:** 1천 문서 스페이스를 Markdown으로 내보내면 브라우저가 1천 개 요청을 한꺼번에 띄운다. 게이트웨이 rate-limit에 걸려 일부가 실패하면 `loaded.filter(p !== null)`가 없는 문서를 조용히 빼서 불완전한 파일이 나온다(`getPage`가 null을 주는 경우).
- **수정안 (M):** 서버에 이미 PDF용 하위 수집기(`PdfExportService`, `MAX_DOCS=100`)가 있다. Markdown도 서버 내보내기(`GET /pages/{id}/export.md?descendants=true`, 상한과 스트리밍 포함)로 옮긴다. 단기 대안으로는 동시 실행 수 제한(예: 6) + 실패 시 전체 중단(S)이 있다.
- **브랜치:** 수정 안 됨.

### W-15 [성능 · Medium] 구조 변경 경로의 N+1
- **근거**
  - `page/PageService.java:368-386` `collectSubtree`: 주석은 "N+1을 없앴다"고 하지만 `:383`에서 노드마다 `pages.findById(id)`를 실행한다(본문 포함 엔티티). 이동, 삭제(CASCADE/PROMOTE), 스페이스 간 이동이 모두 이 경로를 쓴다.
  - `:646` `validateParent`는 조상마다 `findById`.
  - `space/SpaceService.java:118-128`: 전 페이지 엔티티(본문 포함)를 읽은 뒤 페이지마다 쿼리 5회를 실행하고, `:132` `deleteAllInBatch(all)`는 `or` 조건 수천 개짜리 DELETE를 만든다.
  - `TrashService.purgeExpired`(`:165-176`)는 루트마다 `restorableBatch → trashRows(spaceId)`로 스페이스 휴지통 전체를 다시 읽는다.
- **왜 중요한가:** 하위가 수천 개인 폴더를 옮기거나 지우면 수천 번의 단건 조회가 한 트랜잭션 안에서 돈다. 스페이스 잠금(`lockForReorder`)을 쥔 채 오래 걸려 같은 스페이스의 다른 편집이 줄을 선다. 스페이스 삭제는 대형 스페이스에서 타임아웃이나 파라미터 한도에 걸릴 수 있다.
- **수정안 (M):** `collectSubtree`는 BFS로 id만 모은 뒤 `findAllById(ids)` 한 번으로 읽는다(권한 판정에는 id만 필요하므로 `requireEditAll`을 id 기반으로 바꾸면 더 좋다). `validateParent`는 `findAncestorClosure` 재사용. 스페이스 삭제는 `delete ... where page_id in (select id from page where space_id=?)` 형태의 bulk JPQL·SQL로 테이블당 1회 실행하고, 객체 키만 프로젝션해서 읽는다. purgeExpired는 스페이스별로 묶는다.
- **브랜치:** 수정 안 됨.

### W-16 [성능 · Medium] 블로그 목록이 페이지 나눔 없이 본문 전체를 읽음
- **근거:** `page/BlogService.java:33-41`과 `repository/PageRepository.java:86-93` `findBlogPosts`가 상한 없이 Page 엔티티(본문 포함)를 읽는다. `page/dto/BlogPostView.java:23-36`은 글마다 본문 전체에 정규식 9개를 실행해 발췌 200자를 만든다.
- **왜 중요한가:** 블로그는 시간이 갈수록 계속 쌓인다. 글 500개짜리 스페이스에서 목록 한 번이 본문 수십 MB 조회와 정규식 처리가 된다. (W-02 때문에 운영에서는 아직 글이 쌓이지 않으므로, V42와 함께 지금 고치기가 쉽다.)
- **수정안 (S~M):** `createdAt, id` 키셋 커서 + limit(이력·댓글 쪽 목록과 같은 모양 `?before=&limit=`). 발췌는 `substring(content, 1, 2000)` 프로젝션에서만 만들거나, 저장할 때 `excerpt` 컬럼에 기록한다.
- **브랜치:** 수정 안 됨.

### W-17 [기능/데이터 손실 · Medium] 앱 안 이동 시 미저장 편집 유실
- **근거:** `wiki-front/src/features/wiki/pages/PageEditPage.tsx:172-180`은 `beforeunload`(새로고침·탭 닫기)만 막는다. 주석에 "선언형 Routes라 useBlocker 불가"라고 적혀 있다. `src/app/main.tsx:10,34`는 `BrowserRouter`를 쓴다. 로컬 초안 자동 저장도 없다(공동 편집 플래그가 꺼져 있으므로 서버 초안도 없다).
- **왜 중요한가:** 편집 중 사이드바 트리, 상단 바, 알림 링크를 누르면 확인 없이 편집 내용이 사라진다. 컨플루언스 대체품으로서 가장 먼저 신뢰를 잃는 지점이다.
- **수정안 (M):** `createBrowserRouter` + `RouterProvider`(data router)로 옮겨 `useBlocker`를 쓴다(App의 Routes 트리를 그대로 `createRoutesFromElements`로 감싸면 큰 변경이 없다). 보조 수단으로 편집 본문을 페이지별로 localStorage에 주기 저장하고 재진입 시 복구를 제안한다(UI 프리퍼런스가 아니라 클라이언트 임시본이므로 `lib/*` 전용 모듈로 둔다).
- **브랜치:** 수정 안 됨.

### W-18 [경계면 · Low] 디렉터리 장애를 빈 목록으로 삼킴
- **근거:** `wikiApi.ts:393-400` `listTeams`와 `:94-108` `listUsers`는 모든 예외에서 `[]`를 돌려준다. `RestrictionsDialog.tsx:37`은 그 결과로 팀 선택지를 만든다.
- **왜 중요한가:** org-service가 503일 때 제한 다이얼로그에는 "팀 없음"이 나와, 관리자가 제한을 사람 단위로 잘못 저장할 수 있다. 플랫폼 경계 규약(503 → 빈 목록 삼킴 금지)에도 어긋난다. 이름 표시용(listUsers)은 폴백이 정당하지만, 선택 UI용(listTeams)은 오류를 보여야 한다.
- **수정안 (S):** `listTeams`는 던지게 하고, 다이얼로그가 "조직 디렉터리에 연결할 수 없습니다"를 보여 주며 저장 버튼을 막는다.
- **브랜치:** 수정 안 됨.

### W-19 [성능 · Low] 저장마다 사전 GET
- **근거:** `wikiApi.ts:323` `updatePage`가 PUT 전에 `getPage(id)`(본문 포함)를 부른다. title·parentId 채우기용이다. 409가 나면 `:338`에서 한 번 더 부른다.
- **왜 중요한가:** 저장 한 번이 왕복 2회와 본문 2배 전송이 된다. 큰 문서일수록 저장이 느리다.
- **수정안 (S):** 편집 화면이 이미 들고 있는 title·parentId·version을 `UpdatePageOptions`로 넘겨 사전 GET을 없앤다(스토어 시그니처 호환 유지, 없을 때만 GET).
- **브랜치:** 수정 안 됨.

### W-20 [정확성 · Low] 라이트 검색 LIKE 와일드카드 미이스케이프
- **근거:** `search/LiteSearchService.java:118` `"%" + query.toLowerCase() + "%"`. 브랜치 `LiteSearchService.java:86`도 같다. `PageRepository`에는 ESCAPE 절이 없다.
- **왜 중요한가:** "50%"나 "snake_case"를 검색하면 `%`와 `_`가 와일드카드로 해석돼 엉뚱한 결과가 나오고, `%` 한 글자 검색은 전 문서를 훑는다(trgm 인덱스 무력화).
- **수정안 (S):** `\`, `%`, `_`를 이스케이프하고 쿼리에 `escape '\'`를 붙인다. 테스트 1건.
- **브랜치:** 수정 안 됨.

### W-21 [정확성 · Low] 최근 방문 기록 경합
- **근거:** `personal/PersonalService.java:104-107`은 조회 후 insert 하는 방식이다. `page_visit` PK는 `(user_id, page_id)`(`V21__star_and_visit.sql:26-31`). 같은 트랜잭션에서 `PageService.recordView`가 조회수를 올린다(`PageService.java:604-606`).
- **왜 중요한가:** 같은 사용자가 처음 보는 문서를 두 탭에서 거의 동시에 열면 PK 충돌로 트랜잭션 전체가 롤백되고, 조회수 증가까지 사라진다(프론트는 오류를 무시한다).
- **수정안 (S):** 네이티브 `insert ... on conflict (user_id, page_id) do update set visited_at = now()`로 바꾼다. `trim`은 별도 쿼리로 한다.
- **브랜치:** 수정 안 됨.

### W-22 [기능(정책) · Low] 초안의 노출 범위 (사용자 결정 필요)
- **근거:** 백엔드에서 `PageStatus.DRAFT`를 참조하는 곳은 통계와 proto 매핑뿐이다(`admin/AdminStatsService.java:63`, `grpc/WikiContentGrpcService.java:186`). 초안은 스페이스 열람자 전원에게 트리로 보이고, `onPageUpdated`로 구독자에게 알림이 가며, 검색 옵션(`draftsIncluded`)으로도 나온다.
- **왜 중요한가:** 컨플루언스는 초안을 작성자 전용으로 둔다. "초안=Page.status"는 확정 결정이지만 가시성 정책은 확정 기록이 없다(wiki-front-content-model 메모리의 "남은 정책 P2"). 임의로 바꾸지 말고 결정을 받아야 한다.
- **수정안:** 기획에 에스컬레이트. 선택지 A는 현행 유지 + 초안 갱신 알림만 억제(S), 선택지 B는 작성자·EDIT 권한자만 열람(M, effective 판정에 상태 추가).
- **브랜치:** 해당 없음.

### W-23 [위생 · Low] 거대 파일과 반복 패턴
- **근거:** `wiki-front/src/features/wiki/store/wikiApi.ts` 1562줄, `wikiMock.ts` 2470줄, `types.ts` 921줄, `PageTree.tsx` 702줄, `PageViewPage.tsx` 665줄. `wiki-backend/.../page/PageService.java` 738줄은 생성, 복사, 이동, 삭제, 게시, 소유자, 검증, 이력을 한 클래스에 담고, 생성자 의존성 19개가 FQCN(`com.platform.wikibackend.notification.NotificationService` 등, `:61-74`)으로 섞여 있다. 같은 파일 `:104-120`, `:244-249`, `:589-590`에는 대상 메서드와 맞지 않는 Javadoc이 남아 있다(이중 Javadoc, 옛 주석 잔존).
- **왜 중요한가:** W-01·W-07 같은 "한 경로만 빠지는" 결함이 생기는 구조적 원인이다. 버전을 올리는 쓰기 경로가 5곳에 흩어져 있고, 저장 후처리(리비전, 작업, 링크, 구독, 이벤트, 알림, 협업 리셋)를 경로마다 손으로 나열한다.
- **수정안 (M):** 저장 후처리를 `PageWriteEffects.afterContentChange(page, actor, note)` 하나로 모아 update, commit, restore, setDone, import가 함께 쓰게 한다(W-01·W-07 재발 방지). 생성·복사, 이동, 메타(소유자·검증·아이콘)로 서비스를 나눈다. 프론트 `wikiApi.ts`는 도메인별 파일(pages, comments, attachments, org…)로 쪼개고 `wikiStore.ts`가 다시 내보낸다(계약 불변). 낡은 Javadoc을 정리한다.
- **브랜치:** 해당 없음.

### W-24 [문서 드리프트 · Low] 리포 안내 문서가 코드와 어긋남
- **근거**
  - `wiki-front/CLAUDE.md:21-23`: "현재는 localStorage(`wiki.v1`) 목업이지만, 백엔드가 붙으면…". 실제로는 `wikiStore.ts:4-8`에서 이미 백엔드 듀얼모드다.
  - `:44-45`: "`/spaces/:spaceId`(WikiLayout …), 그 외 전부 첫 스페이스로 redirect". WikiLayout은 없다(AppShell). `/home`, `/search`, `/tasks`, `/admin/*` 라우트가 있다(`App.tsx:128-149`).
  - `:51`: "웨이브(W1~W7)별 App 통합 테스트". 실제로는 w2~w32다.
  - `wiki-backend`에는 CLAUDE.md가 없다. 루트 CLAUDE.md는 "각 서비스의 상세 규약은 그 리포의 CLAUDE.md, 작업 전 반드시 읽는다"고 하지만, 정작 가장 큰 위키 백엔드는 README(458줄)뿐이다.
  - 루트 `CLAUDE.md`의 백엔드 목록(8개)에 `collaboration-service`가 없다.
- **왜 중요한가:** 에이전트가 이 문서를 근거로 일하므로, 낡은 "목업 전용" 서술은 잘못된 판단(예: 스토어 밖 fetch, 목업만 수정)을 부른다.
- **수정안 (S):** wiki-front CLAUDE.md의 세 단락을 갱신한다. wiki-backend CLAUDE.md를 신설한다(명령어, 확정 규약: `@SQLRestriction` 우회는 TrashService만, page → collaboration 잠금 순서, 오류 계약은 common-starter, 저장 후처리 목록, PG 전용 주의점). 루트 목록에 collaboration-service를 추가한다(AGENTS.md도 함께 갱신, 루트 규약).
- **브랜치:** 해당 없음.

---

## 참고: 감사했지만 문제로 올리지 않은 것
- 프로덕션 번들에 목업 스토어(`wikiMock`)가 섞이는지: `dist`에서 `wiki.v1`이 검출되지 않아 트리셰이킹이 확인됐다.
- 메일 발송: 커밋 뒤 별도 executor에서 보내므로(`EmailNotifier.java:113-150`) 저장 지연 원인이 아니다.
- 내부 import 토큰: `MessageDigest.isEqual` 상수시간 비교이고, 토큰이 비면 차단된다(`InternalTokenFilter.java:43-68`).
- 공동 편집(collaboration-service): 티켓 소비, epoch 가드, 본문 상한(`bootstrapHttp.ts:40-63`)이 계약대로다. 플래그는 아직 꺼져 있고, 남은 높음 항목은 기존 추적(옵시디언 43, 메모리 wiki-realtime-collaboration-status)을 따른다. 이 감사의 W-05(Yjs 상태 미삭제)만 새로 올렸다.

## 권장 착수 순서
1. W-02 + W-03(V42와 PG 테스트 프로필), W-01(한 줄 수정 + 회귀 테스트). 셋 다 S이고 운영 결함이다.
2. W-04, W-06, W-05: 누출과 삭제 계약 위반. W-05는 W-23의 후처리 통합과 같이 하면 싸다.
3. W-07, W-08, W-09, W-10: 불변식과 신뢰성.
4. 성능 묶음 W-11~W-16. 브랜치 머지 뒤에 한다(W-11은 브랜치 폴링과 겹친다).
5. W-17(사용자 체감이 큼, M). 나머지 Low는 롤업한다.

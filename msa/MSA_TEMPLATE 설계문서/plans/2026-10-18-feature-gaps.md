---
tags: [msa-template, plan]
date: 2026-10-18
상태: 기획 초안 — P0/P1 구현 대기, 결정 대기 3건(§4)
---

# 기능 공백 백로그 (2026-10-18)

사용자 지시: "부족한 기능도 보완해줘". 같은 날 나온 감사([[2026-10-18-audit/wiki|wiki]]·[[2026-10-18-audit/alm|alm]]·[[2026-10-18-audit/platform|platform]]·[[2026-10-18-audit/ops-docs|ops-docs]])는 **결함**을 다룬다. 이 문서는 **기능 공백**만 다룬다. 감사와 겹치는 항목은 빼고, 빼면서 어느 감사 항목인지 §5에 적었다.

## 0. 문제 정의와 기준

- **문제**: 컨플루언스·지라에서 넘어온 사용자가 바로 찾는 기능이 없거나 반쯤만 있다. 특히 **API는 있는데 화면이 없는** 경우가 여럿이다. 사용자는 이런 기능이 "없다"고 느낀다.
- **대상 사용자**: 위키 편집자·열람자, ALM 팀원·프로젝트 관리자, 플랫폼 관리자(org ADMIN), AI 에이전트 감독자.
- **완료 기준**(실사용 지표 대신): 항목마다 수용 기준(AC)이 실스택(dev 오프셋 `:18000` 또는 배포본)에서 끝까지 동작한다.
- **선정 기준**(우선순위 순): ① API는 있는데 UI가 없음 ② 있는 줄 알았는데 반쯤만 됨 ③ 컨플·지라 사용자가 당장 찾는 핵심 기능. 거창한 신규 제품은 넣지 않는다.
- **우선순위**: MoSCoW 기준을 P로 표기한다. P0 = Must, 이번 세션 즉시. P1 = Should, 이번 세션에 S/M 크기로. P2 = Could, 다음 이후. 실사용 데이터가 없으므로 RICE 점수는 매기지 않는다.

### 근거 출처와 한계

- **우리 코드**: 각 리포 `main` 기준. 체크아웃이 피처 브랜치인 리포(alm-front `feat/alm-performance-ux-20261016`)는 `origin/main`으로 판정했다.
- **컨플·지라**: 참고 사이트 `tom-chkim-dev.atlassian.net`은 Atlassian MCP로 열어 보지 못했다(cloud id 미승인 오류). 그래서 Atlassian Cloud 공식 기능 이름을 근거로 삼았다(Confluence Cloud: Search filters, Copy page, Watch, Starred, Notifications, People profile. Jira Cloud: Web links, Attachment preview, Clone issue, Delete sprint, Share filter, Keyboard shortcuts). ⚠️ 확인필요: 화면 배치를 충실히 복제해야 하는 항목(F-04, F-05, F-07)은 사이트 접근을 승인한 뒤 실제 화면을 캡처해 디자이너에게 넘긴다.
- **마이그레이션 번호**: wiki-backend 다음 번호 **V42**. 단 감사 W-02(BLOG CHECK 제약)가 V42를 먼저 쓸 예정이라 V43부터일 수 있다. alm-backend 다음 번호 **V27**(미병합 V26 outbox 다음). agent-service V17, org-service V9, board-service V3, auth-server V7.
- **제외(확정 결정)**: 이관 모듈(보류), ALM 통합 검색 브랜치 재이식, 위키 초안 노출 정책(웨이브 1에서 구현), ROPC·자체 로그인 폼. 위키 공동 편집의 남은 결함(높음-1~4)도 이 문서 범위가 아니다(노트 43).

---

## 1. 요약 표

| ID | 제품 | 공백 | 범위 | 크기 | 마이그레이션 | P |
|---|---|---|---|---|---|---|
| F-01 | WIKI | 검색에 스페이스·종류 필터 없음 | 프론트 | S | 없음 | **P0** |
| F-02 | WIKI | 문서 ⋯ 메뉴에 이동·복사 없음 | 프론트 | S | 없음 | P1 |
| F-03 | WIKI | 복사할 때 대상 스페이스·위치를 고를 수 없음 | 둘 다 | M | 없음 | P1 |
| F-04 | WIKI | 별표·최근·내 초안 전체 목록 화면 없음 | 프론트 | S | 없음 | P1 |
| F-05 | WIKI | 알림이 벨 30건뿐, 전체 화면·페이징 없음 | 둘 다 | M | 없음 | P1 |
| F-06 | WIKI | 내가 구독한 문서·스페이스 목록 없음 | 둘 다 | S | 없음 | P1 |
| F-07 | WIKI | 사람 프로필 화면 없음, @멘션이 어디로도 안 감 | 프론트(+org 조회) | M | 없음 | P2 |
| F-08 | ALM | 웹 링크: 백엔드 완성, 화면 없음 | 프론트 | S | 없음 | **P0** |
| F-09 | ALM | 첨부 미리보기 없음(inline API 미사용) | 프론트 | S | 없음 | P1 |
| F-10 | ALM | 잘못 만든 스프린트를 지울 수 없음 | 둘 다 | S | 없음 | P1 |
| F-11 | ALM | 이슈 복제 없음 | 프론트 | S | 없음 | P1 |
| F-12 | ALM | 저장 필터를 팀과 공유할 수 없음 | 둘 다 | M | V27 | P1 |
| F-13 | ALM | 키보드 단축키 없음 | 프론트 | S | 없음 | P2 |
| F-14 | ALM | 이슈 목록 열 표시·숨김 없음 | 프론트 | S | 없음 | P2 |
| F-15 | AI 에이전트 | 킬 스위치를 켜고 끌 화면 없음 | 프론트 | S | 없음 | P1 |
| F-16 | AI 에이전트 | run별 비용이 안 보임 | 둘 다 | S | 없음 | P1 |
| F-17 | AI 에이전트 | 도구 호출 감사 이력 검색 불가(오늘 50건뿐) | 둘 다 | M | 인덱스 가능(V17) | P2 |
| F-18 | AI 에이전트 | 플랫폼 화면에서 페르소나 수정·전 프로젝트 run·게이트 승인 불가 | 프론트 | M | 없음 | P2 |
| F-19 | AI 에이전트 | 월 예산 상한을 화면에서 못 바꾸고 임박 알림 없음 | 둘 다 | M | V17 | P2 |
| F-20 | 포털 | 관리자 대시보드에 AI 에이전트 지표 없음 | 프론트 | S | 없음 | P1 |
| F-21 | 포털 | PAT 이름 변경·전부 폐기 없음 | 둘 다 | S | 없음 | P1 |
| F-22 | 게시판 | 댓글: 백엔드 완성, 화면 없음 | 프론트 | S | 없음 | **P0** |
| F-23 | 게시판 | 게시글 검색 없음 | 둘 다 | S | 없음 | P1 |
| F-24 | org 관리 | 권한 변경 감사 화면 없음(API 있음) | 프론트(DS) | S | 없음 | P1 |
| F-25 | org 관리 | 초대 이력 화면·구성원 CSV 내보내기 없음 | 둘 다 | M | V9(인덱스) | P2 |

P0 3개, P1 15개, P2 7개.

---

## 2. 항목 상세

### WIKI

#### F-01 검색에 스페이스·종류 필터 없음 — P0 · S · 프론트
- **문제**: "이 스페이스에서만" 또는 "첨부만·블로그만"으로 검색할 수 없다. 컨플 검색에서 가장 먼저 쓰는 필터다.
- **근거**: 검색 계약은 이미 받는다(`wiki-front/src/features/wiki/store/types.ts:302-303` `spaceIds`·`docTypes`, `wikiApi.ts:1405`가 `spaceIds` 전송). 그런데 `pages/SearchPage.tsx:131-139`는 작성자·라벨·날짜·정렬만 넘긴다. 컨플: Search filters(Space, Content type).
- **AC**:
  - Given 스페이스 A·B에 같은 단어가 든 문서가 있고, When 스페이스 필터에서 A를 고르면, Then A의 결과만 나오고 URL에 필터가 남아 새로고침해도 유지된다.
  - Given 종류 필터에서 "블로그"를 고르면, Then 블로그 글만 나온다.
  - Given 스페이스 화면에서 검색으로 들어오면, Then 그 스페이스가 미리 선택돼 있다.
- **참고**: `SEARCH_MODE=lite`와 `opensearch` 두 경로 모두에서 필터가 동작해야 한다(⚠️ 확인필요: lite 백엔드가 `docTypes`를 처리하는지 구현 전 확인).

#### F-02 문서 ⋯ 메뉴에 이동·복사 없음 — P1 · S · 프론트
- **문제**: 이동·복사는 트리의 메뉴에만 있다. 문서를 보다가 ⋯ 메뉴에서 찾으면 없다.
- **근거**: `pages/PageViewPage.tsx:473-519`(내보내기·히스토리·보관·소유자·검증·템플릿 저장·삭제). 이동·복사는 `components/PageTree.tsx:557-562`에만 있다. 컨플: 문서 ⋯ 메뉴 Move / Copy.
- **AC**:
  - Given 편집 권한이 있는 문서를 볼 때, When ⋯ 메뉴를 열면, Then "이동"과 "복사"가 있고 트리와 같은 대화상자가 열린다.
  - Given 편집 권한이 없으면, Then 두 항목이 보이지 않는다.

#### F-03 복사할 때 대상 스페이스·위치를 고를 수 없음 — P1 · M · 둘 다
- **문제**: "복제"는 늘 원본 옆에 사본을 만든다. 다른 스페이스나 다른 부모 아래로 복사할 수 없다. 이동은 스페이스 간에 이미 된다.
- **근거**: `wiki-backend/.../page/dto/CopyRequest.java:8`은 `(includeDescendants, includeRestrictions)`뿐이다. `PageMoveRequest.java:9-21`에는 `spaceId`·`beforeId`가 있다. 프론트 `wikiApi.ts:862-872`. 컨플: Copy page 대화상자의 대상 스페이스·부모 선택.
- **범위**: 백엔드는 `CopyRequest`에 대상 스페이스·부모를 선택 필드로 더하고, 생략하면 지금처럼 동작한다. 대상 스페이스의 편집 권한을 확인한다. 프론트는 복사 대화상자에 이동과 같은 위치 선택기를 붙인다.
- **AC**:
  - Given 스페이스 A의 문서를, When 스페이스 B의 문서 X 아래로 복사하면, Then 사본이 B의 X 밑에 생기고 원본은 그대로다.
  - Given B에 편집 권한이 없으면, Then 403 문구가 대화상자에 보이고 아무것도 생기지 않는다.
  - Given 대상 위치를 고르지 않으면, Then 지금처럼 원본 옆에 생긴다(호환).
- **계약 결정 대기**: 하위 포함 복사에서 제한(restrictions)을 다른 스페이스로 가져갈지. 컨플은 사용자가 고르게 한다. 우리도 기존 `includeRestrictions` 옵션을 그대로 쓰는 안을 권한다.

#### F-04 별표·최근·내 초안 전체 목록 화면 없음 — P1 · S · 프론트
- **문제**: 별표는 플라이아웃에만 있고, 최근의 "전체 보기"는 홈 대시보드로 간다. 내 초안을 한곳에서 볼 데도 없다.
- **근거**: `components/GlobalSidebar.tsx:313`(`morePath="/home"`), `:340`(`StarredFlyout`). API `GET /api/wiki/stars`, `GET /api/wiki/recent?limit=`(`PersonalController.java:36,70`). 컨플: Home의 Recent·Starred·Drafts 탭.
- **의존**: 내 초안 탭은 웨이브 1의 초안 노출 변경(작성자+관리자만) 뒤에 붙인다. 초안 목록 API가 웨이브 1에서 생기지 않으면 이 탭은 빼고 출시한다.
- **AC**:
  - Given 별표한 문서가 40개일 때, When 사이드바 별표의 "전체 보기"를 누르면, Then 전체 목록 화면이 열리고 별표 해제가 그 자리에서 된다.
  - Given 최근 "전체 보기"를 누르면, Then 홈이 아니라 최근 방문 전체 목록이 열린다.

#### F-05 알림이 벨 30건뿐, 전체 화면·페이징 없음 — P1 · M · 둘 다
- **문제**: 최근 30건만 보이고 "모두 보기"·종류별 필터·이전 알림이 없다. "새 문서 게시"는 별도 수신 설정 없이 "문서 수정" 설정에 묶여 있다.
- **근거**: `NotificationService.java:48`(`PAGE_SIZE = 30`), `NotificationController.java:27`(커서 파라미터 없음), `NotificationBell.tsx:23-24`, `NotificationPref.java:96`. 컨플: Notifications 패널의 전체 보기·필터.
- **범위**: 백엔드는 목록에 커서(또는 페이지)와 종류 필터를 더한다. `PAGE_PUBLISHED` 전용 수신 설정 분리는 같은 슬라이스에서 하되, 기존 값을 그대로 이어받는다. 실시간 푸시는 범위 밖이다(지금처럼 폴링).
- **AC**:
  - Given 알림이 100건일 때, When 벨의 "모두 보기"를 누르면, Then 전체 알림 화면에서 스크롤로 이전 알림을 계속 읽는다.
  - Given 종류 필터에서 "멘션"을 고르면, Then 멘션 알림만 나온다.
  - Given "새 문서" 알림만 끄면, Then 문서 수정 알림은 계속 온다.
- **주의**: 감사 W-11(알림 목록 성능)과 같은 코드다. 함께 고친다.

#### F-06 내가 구독한 문서·스페이스 목록 없음 — P1 · S · 둘 다
- **문제**: 구독은 할 수 있는데 무엇을 구독했는지 보거나 한꺼번에 해제할 수 없다.
- **근거**: `watch/WatchController.java:30-45`, `SpaceWatchController.java`는 단건 상태·구독·해제뿐이다. 테이블 `page_watch`·`space_watch`는 이미 있다. 컨플: 프로필 설정의 Watching 목록.
- **AC**:
  - Given 문서 5개·스페이스 2개를 구독했을 때, When 개인 설정의 "구독" 화면을 열면, Then 7개가 종류별로 나온다(권한을 잃은 문서는 제목을 가린다).
  - When 여러 개를 골라 "구독 해제"하면, Then 목록과 각 문서의 구독 버튼이 함께 해제 상태가 된다.

#### F-07 사람 프로필 화면 없음 — P2 · M · 프론트(+org 조회)
- **문제**: `@사람`을 눌러도 아무 일도 없다. 그 사람이 쓴 문서·참여 스페이스를 볼 화면이 없다.
- **근거**: `components/MarkdownView.tsx:76-78`("프로필 화면이 아직 없어 … 칩(span)으로 그린다 — 후속"), `app/App.tsx:128-216`에 사람 경로 없음. 컨플: People profile.
- **AC**:
  - Given 문서의 @멘션을 누르면, Then 그 사람의 프로필 화면(이름·아바타·최근 작성 문서 중 내가 볼 수 있는 것)이 열린다.
  - Given 비활성 구성원이면, Then "비활성" 표시와 함께 문서 목록만 보인다.
- **❓ 정책대기**: 프로필에 이메일·팀을 누구에게 보일지(조직 전원 vs 관리자). 사용자 결정 필요.

### ALM

#### F-08 웹 링크: 백엔드 완성, 화면 없음 — P0 · S · 프론트
- **문제**: 이슈에 PR·문서·위키 주소를 붙일 수 없다. 백엔드(V22·V25)는 최근에 만들어 놓았다.
- **근거**: `alm-backend/.../collab/CollaborationController.java:135,143,151`(GET/POST/DELETE). alm-front `origin/main` `src`에 `web-links` 호출이 0건. 지라: Issue links → Web link.
- **AC**:
  - Given 편집 권한이 있을 때, When 이슈 상세의 링크 영역에서 "웹 링크 추가"로 URL과 표시 이름을 넣으면, Then 목록에 생기고 새 탭으로 열린다.
  - Given 같은 URL을 다시 넣으면, Then 중복 오류 문구가 보인다(V25 unique).
  - Given 읽기 권한만 있으면, Then 목록은 보이고 추가·삭제는 없다.

#### F-09 첨부 미리보기 없음 — P1 · S · 프론트
- **문제**: 이미지·PDF도 내려받아야만 볼 수 있다. 썸네일·라이트박스가 없다.
- **근거**: `AttachmentController.java:61`(`GET /api/alm/attachments/{id}/inline`)을 아무도 안 부른다. `IssueAttachments.tsx:87-91`은 다운로드만 한다. 지라: Attachment preview.
- **AC**:
  - Given 이미지 첨부가 있으면, Then 목록에 썸네일이 보이고 누르면 큰 화면 미리보기가 열린다(Esc로 닫힘).
  - Given PDF를 누르면, Then 새 탭에서 브라우저 미리보기로 열린다.
  - Given 그 밖의 형식이면, Then 지금처럼 내려받는다.

#### F-10 스프린트 삭제 없음 — P1 · S · 둘 다
- **문제**: 잘못 만든 예정 스프린트를 지울 수 없다.
- **근거**: `sprint/SprintController.java`에 `@DeleteMapping` 없음, alm-front에 `deleteSprint` 없음. 지라: Delete sprint(백로그 스프린트 ⋯ 메뉴).
- **범위**: 예정(시작 전) 스프린트만 삭제한다. 들어 있던 이슈는 백로그로 돌아간다. 진행 중·완료 스프린트는 지우지 않는다.
- **AC**:
  - Given 이슈 3개가 든 예정 스프린트를, When 프로젝트 관리자가 ⋯ → 삭제하고 확인하면, Then 스프린트가 사라지고 이슈 3개는 백로그 맨 위에 남는다.
  - Given 진행 중 스프린트면, Then 삭제 항목이 비활성이고 서버도 409로 거부한다.

#### F-11 이슈 복제 없음 — P1 · S · 프론트
- **문제**: 비슷한 이슈를 다시 만들 때 지라의 "복제"가 없다. 상세 ⋯ 메뉴에는 보관·삭제뿐이다.
- **근거**: `IssueDetailModal.tsx:1094-1118`. 로드맵 `docs/roadmap/2026-08-28-jira-parity-requirements.md` §5.1:116 ❌. 지라: Clone issue.
- **범위**: 기존 생성 API로 처리한다. 복사하는 것: 제목("복제: " 접두), 타입, 본문, 우선순위, 라벨, 컴포넌트, 부모. 복사하지 않는 것: 댓글, 워크로그, 첨부, 상태(처음 상태로), 스프린트. 원본과 "복제됨" 링크를 건다.
- **AC**:
  - Given 이슈 상세에서, When ⋯ → 복제를 누르면, Then 새 이슈가 처음 상태로 생기고 상세로 이동하며 원본과 링크로 이어진다.
  - Given 원본에 댓글·첨부가 있어도, Then 사본에는 없다.
- **범위 밖**: 다른 프로젝트로 이동(재키잉·상태 매핑 필요, L). §3 후보로 둔다.

#### F-12 저장 필터 공유 — P1 · M · 둘 다 · V27
- **문제**: 저장 필터는 만든 사람만 본다. 팀이 "우리 팀 버그" 필터를 같이 쓸 수 없다. 대시보드는 이미 공유가 된다.
- **근거**: `domain/SavedFilter.java:37`(`ownerId`만), `domain/Dashboard.java:32`(`shared`). 로드맵 §5.5:162 ❌. 지라: Share filter.
- **범위**: 대시보드와 같은 방식(공유 여부 플래그)으로 한다. 공유 필터는 누구나 실행하지만 결과는 실행하는 사람의 권한으로 걸러진다. 수정·삭제는 소유자만 한다.
- **AC**:
  - Given A가 필터를 "공유"로 저장하면, When B가 필터 목록을 열면, Then "공유된 필터"에 보이고 실행 결과에는 B가 볼 수 있는 이슈만 나온다.
  - Given B는, Then 그 필터를 수정·삭제할 수 없다.
- **❓ 정책대기**: 공유 범위를 "조직 전체"로 할지 "프로젝트 멤버"로 할지. 대시보드 `shared`와 같게(조직 전체) 가는 안을 권한다.

#### F-13 키보드 단축키 — P2 · S · 프론트
- **문제**: 지라 사용자가 쓰는 `c`(만들기), `/`(검색), `?`(도움말)가 없다.
- **근거**: 전역 keydown은 `FilterDropdown.tsx:62`뿐, `src/app/App.tsx`에 단축키 계층 없음. 지라: Keyboard shortcuts.
- **AC**:
  - Given 입력란에 포커스가 없을 때, When `c`를 누르면, Then 이슈 만들기가 열린다. `/`는 검색, `?`는 단축키 목록.
  - Given 입력란·에디터에 포커스가 있으면, Then 단축키가 동작하지 않는다.

#### F-14 이슈 목록 열 표시·숨김 — P2 · S · 프론트
- **문제**: 열 순서·너비는 바꾸는데, 어떤 열을 보일지는 못 고른다.
- **근거**: `uiStore.ts:43-46`(`TablePrefs = {order, widths}`). 로드맵 §5.4:151 ❌. 지라: 목록 보기 Columns.
- **AC**:
  - Given 열 설정에서 "기한"을 끄면, Then 열이 사라지고 새로고침해도 유지된다.
  - Given "기본값으로"를 누르면, Then 처음 열 구성으로 돌아간다.
- **범위 밖**: 셀 인라인 편집(M). §3 후보.

### AI 에이전트

진행 중인 작업 메모: run 목록 페이징·필터와 run 상세 API 사용은 alm-front 피처 브랜치에 커밋되지 않은 작업으로 진행 중이다. 그래서 이 표에 따로 넣지 않았다. 그 작업을 먼저 마무리해 병합한다.

#### F-15 킬 스위치 조작 화면 — P1 · S · 프론트
- **문제**: 급할 때 모든 에이전트를 멈추려면 curl을 써야 한다. 화면은 상태만 보여 준다.
- **근거**: `agent-service/.../budget/BudgetController.java:39`(`POST /api/agent/kill-switch`) 호출처 없음, `AiOfficePage.tsx:759`는 표시만 한다.
- **의존**: 감사 O-09(킬 스위치가 인메모리, 실행 중 run을 안 멈춤)를 **먼저 또는 같이** 고친다. 재기동하면 풀리는 스위치에 버튼부터 달면 "껐다"고 믿게 만든다.
- **AC**:
  - Given 플랫폼 관리자가 `/app/agents`에서, When "모든 에이전트 정지"를 누르고 사유를 넣어 확인하면, Then 스위치가 켜지고 ALM 사무실 화면에도 정지 상태가 보인다.
  - Given 관리자가 아니면, Then 버튼이 없고 API도 403이다.

#### F-16 run별 비용 표시 — P1 · S · 둘 다
- **문제**: 오늘 페르소나별 비용과 월 누계만 보인다. 어떤 run이 돈을 얼마나 썼는지 모른다.
- **근거**: `usage_ledger`에 run별 `cost_usd`·`input_tokens`가 이미 쌓인다(`V3__run_gate_ledger.sql:43-44`). `RunDetailResponse`에 비용 필드가 없다.
- **범위**: run 상세 응답에 합계 비용·입력/출력 토큰을 더하고 상세 화면에 보인다. 프로젝트·모델·기간별 집계 화면은 범위 밖(F-19와 함께 P2).
- **AC**:
  - Given 끝난 run의 상세를 열면, Then 비용(USD)과 토큰 수가 보인다.
  - Given 원장 기록이 없는 run이면, Then "기록 없음"으로 보이고 0으로 속이지 않는다.

#### F-17 도구 호출 감사 이력 검색 — P2 · M · 둘 다
- **문제**: 페르소나별로 오늘 마지막 50건만 보인다. 지난 날짜·도구·상태·run으로 찾을 수 없다.
- **근거**: `agent-service/CLAUDE.md:801`(`/personas/{id}/activity` → 오늘, 최대 50), 테이블 `tool_call_audit`.
- **AC**:
  - Given 감사 화면에서 기간·도구·상태로 거르면, Then 결과가 페이지 단위로 나온다.
  - Given 프로젝트 권한이 없는 run의 감사는, Then 나오지 않는다(감사 O-24와 같은 규칙).

#### F-18 플랫폼 화면의 에이전트 관리 공백 — P2 · M · 프론트
- **문제**: 플랫폼 관리자가 `/app/agents`에서 페르소나를 고치거나 끄지 못하고, 전 프로젝트 run·대기 게이트도 볼 수 없다. 이런 일은 각 ALM 프로젝트 사무실 안에서만 된다.
- **근거**: `myFront/src/app/agents/agentsStore.ts:251-571`(PATCH 페르소나·runs·gates 호출 없음). 서버 `PersonaController.java:52,62`, `RunController.java:83`, `GateController.java:34`.
- **AC**:
  - Given 플랫폼 관리자가, When 페르소나를 비활성화하면, Then 새 run이 배정되지 않는다.
  - Given "대기 중 게이트" 탭을 열면, Then 전 프로젝트의 승인 대기가 나오고 그 자리에서 승인·거절된다.

#### F-19 예산 상한 설정·임박 알림 — P2 · M · 둘 다 · V17
- **문제**: 월 상한이 env(`BUDGET_MONTHLY_USD`) 하나뿐이고, 상한에 다가가거나 스위치가 켜져도 아무도 모른다.
- **근거**: `BudgetService.java:57-73`, `agent-service/CLAUDE.md:316,321`("예산·킬 스위치 자체의 알림은 아직 없다").
- **AC**:
  - Given 관리자가 화면에서 월 상한을 바꾸면, Then 재기동 없이 적용되고 변경 이력이 남는다.
  - Given 사용액이 상한의 80%를 넘으면, Then 플랫폼 관리자에게 메일(org 중앙 발송)과 대시보드 경고가 간다.
- **❓ 정책대기**: 상한을 플랫폼 하나로 둘지 프로젝트별로도 둘지.

### 포털·게시판·org 관리

#### F-20 관리자 대시보드에 AI 에이전트 지표 — P1 · S · 프론트
- **문제**: 관리자 홈에 에이전트 월 비용 대 상한, 실패·차단 run, 대기 게이트, 킬 스위치 상태가 없다. 에이전트 숫자는 org의 "에이전트" 구성원 수뿐이다.
- **근거**: `myFront/src/app/admin/AdminDashboardPage.tsx:37-44`(fetcher: alm·wiki·org·tokens·health), `:605`. API는 있다: `/api/agent/budget`, `/api/agent/runs?status=`, `/api/agent/gates?pending=`.
- **AC**:
  - Given 관리자 대시보드를 열면, Then "AI 에이전트" 카드에 월 비용/상한, 실패 run 수, 대기 게이트 수, 킬 스위치 상태가 보이고 누르면 `/app/agents`로 간다.
  - Given agent-service가 응답하지 않으면, Then 카드만 "확인 불가"로 보이고 다른 카드는 정상이다.

#### F-21 PAT 이름 변경·전부 폐기 — P1 · S · 둘 다
- **문제**: 토큰을 하나씩만 지울 수 있다. 유출이 의심될 때 한 번에 전부 폐기할 수 없고, 이름도 못 고친다.
- **근거**: `auth-server/.../pat/PersonalAccessTokenController.java:57-93`(GET/POST/DELETE `/{id}`만), `myFront/src/app/tokens/tokensStore.ts`.
- **AC**:
  - Given 토큰이 3개일 때, When "모두 폐기"를 누르고 확인하면, Then 3개 모두 즉시 쓸 수 없고(게이트웨이 교환 실패) 목록이 빈다.
  - Given 토큰 이름을 바꾸면, Then 목록에 바로 반영되고 토큰 값은 그대로 쓸 수 있다.
- **범위 밖**: 만료 임박 메일 알림, 관리자가 남의 토큰 보기(§3 후보). 퇴사자 토큰 폐기는 감사 P-02 몫이다.

#### F-22 게시판 댓글 화면 — P0 · S · 프론트
- **문제**: 게시판에서 댓글을 읽거나 쓸 수 없다. 백엔드 CRUD와 PAT 스코프("글·댓글")는 이미 있다.
- **근거**: `board-service/.../comment/CommentController.java:25,31,38,44`. `myFront/src/app/board/*`에 댓글 코드 0건, `myFront/src/app/tokens/scopes.ts:53`.
- **AC**:
  - Given 글 상세를 열면, Then 댓글 목록이 보이고 로그인 사용자는 댓글을 쓸 수 있다.
  - Given 내 댓글이면, Then 수정·삭제가 되고 남의 댓글에는 그 버튼이 없다.
  - Given board-service가 내려가 있으면, Then 본문은 그대로이고 댓글 영역만 오류 문구를 보인다.

#### F-23 게시판 검색 — P1 · S · 둘 다
- **문제**: 글을 최신순으로 넘겨 보기만 할 수 있다. 제목·본문으로 찾을 수 없다.
- **근거**: `PostController.java:26-30`(Pageable만, `q` 없음).
- **범위**: 목록 API에 `q`(제목·본문 부분 일치, 와일드카드 이스케이프)를 더하고 목록 화면에 검색창을 단다. 통합 검색(search-service) 색인은 범위 밖이다.
- **AC**:
  - Given "배포"가 제목에 든 글 2개가 있으면, When 검색창에 "배포"를 넣으면, Then 그 2개만 나오고 페이징이 유지된다.
  - Given `%`를 검색하면, Then 전체가 아니라 `%`가 든 글만 나온다.

#### F-24 권한 변경 감사 화면 — P1 · S · 프론트(design-system)
- **문제**: 전역 관리자가 누가 누구에게 어떤 역할을 주고 뺐는지 볼 수 없다. API는 있고 위키 스페이스 설정에서만 쓴다.
- **근거**: `platform-backend/org-service/.../grant/GrantController.java:45`(`GET /api/org/grants/audit`, GLOBAL·PROJECT 지원). `design-system/packages/org-admin/src/api/client.ts`에 호출 없음. 사용처는 `wiki-front/.../wikiApi.ts:1198`뿐.
- **범위**: `@chanho/org-admin`에 "권한 이력" 화면을 더하고 새 버전을 발행한다. wiki-front·alm-front가 그 버전으로 올린다(감사 P-15 버전 드리프트와 함께 정리).
- **AC**:
  - Given 전역 관리자가 org 관리의 "권한 이력"을 열면, Then 전역 권한 부여·회수가 시간 역순으로 누가·누구에게·무엇을과 함께 나온다.
  - Given 관리자가 아니면, Then 메뉴가 없고 API도 403이다.

#### F-25 초대 이력·구성원 CSV 내보내기 — P2 · M · 둘 다 · V9
- **문제**: 초대가 보내짐·재발송·수락·만료되는 흐름을 볼 수 없고, 구성원 목록을 내보낼 수 없다.
- **근거**: `InvitationController.java:76`(`GET /{id}/events`) 호출처 없음, `MemberController.java:111`은 구성원 한 명씩만. `V5__member_lifecycle.sql`의 인덱스는 구성원·초대 단위뿐이다.
- **AC**:
  - Given 초대 목록에서 한 건을 누르면, Then 그 초대의 이벤트 이력이 나온다.
  - Given "CSV 내보내기"를 누르면, Then 현재 필터의 구성원이 이름·이메일·상태·팀으로 내려받아진다.

---

## 3. 이번 표 밖의 후보 (근거만 남김)

총 25개 상한 때문에 뺐다. 다음 기획 때 여기서 고른다.

| 제품 | 후보 | 근거 | 크기 |
|---|---|---|---|
| WIKI | 스페이스 홈 문서 | `domain/Space.java:20-44` homepage 필드 없음 | M, 마이그레이션 |
| WIKI | 스페이스 보관 | 스페이스 상태 필드 없음, `SpaceSettingsPage.tsx:243-383` | M, 마이그레이션 |
| WIKI | 제한 문서 접근 요청 | `PageViewPage.tsx:265`에서 멈춤 | M, 마이그레이션 |
| WIKI | 작업 기한 알림 | `PageTask.java:42-43` `due_date`만 있고 스케줄러 없음 | S-M, 마이그레이션 |
| WIKI | 인라인 댓글 리액션 | `InlineCommentLayer.tsx`에 리액션 없음, API는 있음 | S |
| WIKI | 조회자 목록·추이 | `PageVisit.java:30-37` 행은 있음 | M |
| WIKI | 동영상·외부 임베드 | 파리티 문서 `:95-96` ❌ | M |
| ALM | 다른 프로젝트로 이슈 이동 | `IssueMoveRequest`는 보드 이동뿐 | L |
| ALM | 백로그 에픽·버전 패널 | `BacklogPage.tsx`, 로드맵 §5.3:143 | M |
| ALM | 목록 셀 인라인 편집 | `IssueListPage.tsx`가 `updateIssue` 미호출 | M |
| ALM | 전이 조건·검증기 | `types.ts:359-364` | M-L |
| ALM | 세분 권한 스킴 | `AlmAction.java` VIEW·EDIT·ADMIN뿐 | L |
| ALM | 복수 수정 버전·영향 버전 | `Issue.java:91` 단일 | M, 마이그레이션 |
| ALM | 릴리스·에픽 번다운 | `ReportsPage.tsx:53-58` | M |
| AI | 사무실 실시간 스트림 소비 | `OfficeStreamController.java:23`, 감사 O-15 선결 | M |
| 포털 | PAT 만료 임박 메일, 관리자 토큰 조회 | `PatCleanupJob.java:33` 알림 없음 | M |
| 포털 | 헬스 이력·가동률 | `HealthReport.java` 스냅샷만 | M-L |
| 게시판 | 첨부·카테고리·고정글 | `V1__init.sql` 제목·본문뿐 | L |

---

## 4. 열린 결정 (사용자 확인 필요)

1. ❓ **F-12 공유 필터 범위**: 조직 전체(대시보드와 같게, 권장) vs 프로젝트 멤버.
2. ❓ **F-07 프로필 공개 범위**: 이메일·팀을 조직 전원에게 보일지.
3. ❓ **F-19 예산 상한 단위**: 플랫폼 하나 vs 프로젝트별.
4. ⚠️ 확인필요: 참고 컨플루언스 사이트 Atlassian MCP 접근 승인. F-04·F-05·F-07의 화면 복제 근거 캡처에 필요하다.

---

## 5. 감사와 겹쳐서 뺀 것

| 기능 공백처럼 보이는 것 | 감사 항목 |
|---|---|
| ALM 버전 편집·바로 가기 편집 UI | A-21 |
| ALM 댓글·워크로그 관리자 삭제 | A-17 |
| ALM 일괄 편집 서버 처리 | A-11 |
| 퇴사자 토큰·세션 폐기 | P-02 |
| 킬 스위치 영속화·실행 중 run 정지 | O-09(F-15의 선결) |
| 위키 편집 중 앱 안 이동 가드 | W-17 |
| 위키 초안 노출 범위 | W-22(웨이브 1에서 구현) |
| 첨부 업로드 413 | O-05 |

---

## 6. 구현 배분 힌트 (P0·P1)

| 롤 | 항목 |
|---|---|
| 프론트(wiki-front) | F-01, F-02, F-04, F-03·F-05·F-06의 화면 |
| 백엔드(wiki-backend) | F-03 복사 대상, F-05 알림 페이징·수신 설정 분리, F-06 구독 목록 API |
| 프론트(alm-front) | F-08, F-09, F-11, F-10·F-12의 화면, F-16 표시 |
| 백엔드(alm-backend) | F-10 스프린트 삭제, F-12 공유 플래그(V27) |
| 백엔드(agent-service) | F-16 run 상세 비용 필드, O-09(F-15 선결) |
| 프론트(myFront) | F-15, F-20, F-21 화면, F-22, F-23 화면 |
| 백엔드(auth-server·board-service) | F-21 이름 변경·전부 폐기, F-23 `q` 파라미터 |
| 디자인시스템(org-admin) | F-24 + 발행, 소비 앱 버전 범프 |

권하는 순서: P0 3개(모두 프론트만, API 있음) → API가 이미 있는 P1(F-02, F-04, F-09, F-11, F-15, F-20, F-24) → 백엔드를 조금 고치는 P1(F-03, F-05, F-06, F-10, F-12, F-16, F-21, F-23).

관련: [[2026-10-18-audit/wiki]] · [[2026-10-18-audit/alm]] · [[2026-10-18-audit/platform]] · [[2026-10-18-audit/ops-docs]] · 노트 43(공동 편집 남은 것) · 노트 38 §9(org 후속 후보)

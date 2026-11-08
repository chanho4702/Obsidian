---
tags: [msa-template, 설계, 위키, alm, 스마트링크]
created: 2026-11-08
status: 배포됨(2026-11-08)
---

# W3 위키↔ALM 스마트 링크 — 기획·설계 스펙 (2026-11-08)

상태: 확정(자율 세션, 근거는 Jira–Confluence 스마트 링크 선례). 정본은 위키 '개발' 스페이스에 올려야 하나 플랫폼 MCP가 401이라 Obsidian `MSA_TEMPLATE 설계문서/plans/2026-11-08-wiki-alm-smart-link.md`에 사본을 둔다. 탐색 결과: `_workspace/w3_explore_findings.md`.

## 0. 한 줄
위키 본문에 ALM 이슈 링크가 있으면 보기 화면이 그 링크를 **이슈 키 · 제목 · 상태 배지**가 붙은 인라인 스마트 링크로 올려 그린다. ALM이 없거나 못 읽으면 그냥 링크다. 새 문법을 들이지 않는다 — 저장은 Markdown 링크 그대로.

## 1. 문제·사용자·성공 기준
- **문제:** 스펙·회의록(위키)에서 이슈를 언급할 때 지금은 URL만 붙고, 상태를 보려면 ALM으로 가야 한다. 경쟁 제품(Confluence↔Jira 스마트 링크, Linear Team documents, Notion 멘션)은 문서 안에서 상태가 보인다(옵시디언 48 §4 W3, P0).
- **사용자:** 위키 열람자 전원(읽기), 작성자(붙여넣기·슬래시 삽입).
- **성공 기준(검증 가능):** 보기 화면에서 ALM 이슈 링크가 키·제목·상태로 그려지고, 상태를 ALM에서 바꾸면 새로고침 뒤 반영된다. ALM 미설치(features) 또는 권한 없음이면 일반 링크로 보인다. 단위·통합 테스트 그린, 실스택 캡처 1장.

## 2. 범위
**포함(Must)**
1. alm-backend: 키 묶음 조회 `GET /api/alm/issues/cards?keys=` (표시명·담당자 이름까지 채운 요약 DTO, 볼 수 있는 이슈만).
2. gateway: `/api/platform/features`에 `products` 블록(설치 제품) 추가 — 설정 `platform.products`(env `PLATFORM_PRODUCTS`, 기본 전부).
3. alm-front: 키만으로 여는 주소 `/alm/browse/{KEY}` (Jira `/browse/KEY` 선례) → 상세 모달 주소로 교체 이동. 링크 복사가 이 주소를 내게.
4. wiki-front: 보기 화면(MarkdownView)에서 ALM 이슈 링크를 스마트 링크로 승격 + features 어댑터 `products` + 슬래시 메뉴 "ALM 이슈" + 목업 모드 지원 + 테스트.

**제외(이번 릴리스 아님)**
- 카드/임베드 표시(Confluence의 Card·Embed) — 인라인만. `::bookmark` 확장은 후속(Could).
- 편집기 안 스마트 렌더(편집 중에는 일반 링크로 보인다) — Should, 별도 커밋으로 가능하면.
- 키만 타이핑했을 때 자동 변환(Confluence도 안 함). 역방향(ALM 이슈에서 위키 언급 목록) — M1 문서 탭 몫.
- 실시간 갱신·웹소켓. 캐시는 보기 1회 + 60초.
- ALM 설치 스위치의 compose 프로필·404 라우팅 — K1(`PLATFORM_PRODUCTS` 전체)에서. 여기선 features 노출까지만.

## 3. 유저 스토리 + 인수조건
**S1 열람자가 상태를 본다 (Must)**
- Given 본문에 `[PLAT-12](http://host/alm/browse/PLAT-12)` 또는 `http://host/alm/projects/3/issues?issue=PLAT-12` 링크가 있고 ALM이 설치돼 있고 내가 그 프로젝트를 볼 수 있다 / When 페이지를 연다 / Then 링크 자리에 `[타입 아이콘] PLAT-12 제목 [상태 배지]`가 보이고 클릭하면 ALM 이슈가 열린다.
- Given 내가 볼 수 없는 프로젝트의 키거나 없는 키다 / When 페이지를 연다 / Then 그냥 원래 링크(키 텍스트)로 보이고 존재 여부를 알 수 없다.
- Given ALM 미설치(features.products.alm=false 또는 products 없음) 또는 조회 실패(5xx·네트워크) / Then 일반 링크, 콘솔 오류 없음.

**S2 작성자가 넣는다 (Must)**
- Given 편집 중 / When ALM 이슈 URL을 붙여넣는다 / Then 지금처럼 링크로 들어가고(붙여넣기 메뉴 변경 없음), 게시 뒤 보기에서 스마트 링크로 그려진다.
- Given 편집 중 / When 슬래시 메뉴에서 "ALM 이슈"를 고르고 키(`PLAT-12`)를 입력한다 / Then `[PLAT-12](/alm/browse/PLAT-12)` 링크가 삽입된다(상대 경로 — 같은 오리진).

**S3 운영자가 ALM을 끈 설치 (Must)**
- Given 게이트웨이 `PLATFORM_PRODUCTS=wiki,board` / When 위키가 features를 읽는다 / Then `products.alm=false`라 조회를 아예 하지 않고 일반 링크다. 슬래시 메뉴 "ALM 이슈" 항목도 숨긴다.

**S4 키로 바로 열기 (Must)**
- Given 로그인 사용자 / When `/alm/browse/PLAT-12`를 연다 / Then 해당 프로젝트 이슈 목록의 상세 모달(`/alm/projects/{id}/issues?issue=PLAT-12`)로 replace 이동한다. 403·404면 "이슈를 찾을 수 없거나 볼 권한이 없습니다" 안내와 프로젝트 목록 링크.

## 4. 계약 (백엔드 ↔ 프론트, 이 문서가 정본)
### 4.1 `GET /api/alm/issues/cards?keys=PLAT-12,WIKI-3` (alm-backend)
- GET인 이유: 게이트웨이 PAT 스코프가 `/api/alm/**` GET=`alm:read`, POST=`alm:write`(PatScopeRules) — 읽기를 POST로 두면 PAT 읽기 토큰이 못 부른다. 키 ≤ 50개면 URL 길이 문제 없음.
- 입력: `keys` 콤마 구분. 공백 trim·대문자화. 형식 `^[A-Z][A-Z0-9_-]{1,11}-[0-9]{1,9}$` 아닌 항목은 **무시**(400 아님). 중복 제거. 50개 초과 → 400 `{"error": "keys는 최대 50개"}`. 비면 `[]`.
- 출력 `200 [IssueCard]` — **볼 수 있는(프로젝트 VIEW) 기존 이슈만** 담는다. 없는 키·권한 없는 키·휴지통/삭제 프로젝트 이슈는 **조용히 빠진다**(403/404로 존재를 노출하지 않는다 — 탐색 (c)). 순서는 요청 순.
  ```json
  {
    "key": "PLAT-12", "projectId": 3, "projectKey": "PLAT", "title": "…",
    "type":     {"id": "task", "name": "작업", "icon": "check-square", "color": "info"},
    "status":   {"id": "inprogress", "name": "진행 중", "kind": "active", "color": "info", "icon": "circle"},
    "priority": {"id": "high", "name": "높음", "icon": "arrow-up", "color": "warning"},
    "assignee": {"id": 7, "displayName": "Alice Kim"},   // 없으면 null
    "dueDate": "2026-11-30",                              // 없으면 null
    "resolved": false,
    "url": "/alm/browse/PLAT-12"
  }
  ```
  - `type/status/priority`는 그 프로젝트의 **resolved 설정**(SchemeQueries, 프로젝트 스킴 반영)에서 표시명·아이콘·색을 푼다. `status.kind`는 카테고리 kind(`new|active|complete`), `status.color`는 카테고리 color(DS Lozenge appearance 이름 neutral/info/success/warning/danger — alm-front `labels.ts`와 같은 값). 레지스트리에 없는 ID면 name=id, icon·color=null.
  - `assignee.displayName`은 `MemberDirectory.lookup(ids)` 한 번으로 푼다(org 장애면 null, 카드는 그대로 낸다 — 이름 하나 때문에 카드를 버리지 않는다). `resolved`는 상태 카테고리 kind가 `complete`이거나 `resolution`이 있으면 true.
  - 구현 위치: `issue/IssueSearchController`(by-key 옆) + `IssueSearchService.cards(userId, keys)`. 권한은 프로젝트별로 `ProjectService.canView`를 **한 번씩만** 부른다(키를 프로젝트로 묶어서). DB는 키 IN 조회 1회(키 컬럼 인덱스 확인, 없으면 by-key가 쓰는 경로 재사용).
  - OpenAPI 요약 달기(기존 컨트롤러 관례). PAT: GET이라 `alm:read`. 사용자 토큰만(내부 토큰 경로 아님).
  - 테스트: 권한 없는 프로젝트 키 누락, 없는 키 누락, 형식 틀린 키 무시, 51개 400, 레지스트리 해석(스킴 적용 프로젝트), 담당자 이름·org 장애 null, 휴지통 이슈/삭제 프로젝트 누락.

### 4.2 `GET /api/platform/features` (gateway) — `products` 추가
```json
{ "search": { … 기존 그대로 … },
  "products": { "wiki": true, "alm": true, "board": true, "agent": true } }
```
- 설정 `platform.products`(`PLATFORM_PRODUCTS`, 콤마 목록, 기본 `wiki,alm,board,agent`). 모르는 이름은 무시하고 경고 로그 1줄. 이 값은 K1(제품 분리 설치)의 씨앗이다 — 여기선 **features 노출만**, 라우트 404·compose 프로필은 K1.
- 컨트롤러 Javadoc의 계약 문구에 `products` 추가. PAT 403·비로그인 401 그대로. 테스트: 기본 전부 true, `wiki,board`면 alm/agent false, 쓰레기 값 무시.

### 4.3 alm-front `/alm/browse/:key`
- 라우트(App.tsx): `by-key` 조회(`store/api/issues.ts:155` 기존 함수) → `navigate(`/projects/${projectId}/issues?issue=${KEY}`, {replace: true})`. 로딩 중 기존 LazyPage 스켈레톤 관례. 403/404/기타 → 안내 화면(“이슈를 찾을 수 없거나 볼 권한이 없습니다”, 프로젝트 목록 버튼). 키는 trim·대문자.
- `useIssueActions.tsx:63 issueLink`(링크 복사·공유가 쓰는 절대 URL)를 `/alm/browse/{KEY}`로 바꾼다 — 붙여넣는 주소가 곧 정규 주소가 되게. 기존 `?issue=` 주소도 계속 열린다(바꾸지 않음).
- 테스트: 라우트 → replace 이동, 404 안내.

### 4.4 wiki-front
- **인식 규칙(한 곳, `features/wiki/lib/almIssueLink.ts`):** href가 (a) `/alm/browse/{KEY}` 또는 (b) `/alm/projects/{n}/issues?issue={KEY}`(절대·상대, 같은 오리진 또는 호스트 무관하게 경로만 본다) → `{ key }`. KEY 정규식은 4.1과 동일(대소문자 무시, 대문자화).
- **features 어댑터(`store/platformApi.ts`):** `PlatformFeatures.products: { wiki, alm, board, agent }`. `products` 없음·JSON 아님 → 전부 **false**(= 모르면 일반 링크; 검색의 "모르면 lite"와 같은 방향). 목업 `MOCK_FEATURES`는 전부 true. docs 인스턴스는 기존대로 게이트웨이를 안 물으니 자연히 false.
- **보기(MarkdownView):** 렌더 뒤 본문 안 ALM 이슈 링크 키를 모아 `GET /api/alm/issues/cards?keys=`(sharedApiFetch, 사용자 토큰) **1회**, 모듈 캐시(키→카드, 60초). 응답에 있는 키만 `a.alm-smart-link`로 교체 렌더: `[타입 아이콘] KEY 제목 [상태 Lozenge]`. 없는 키·실패는 원래 링크 그대로. 로딩 중에는 원래 링크(스켈레톤 없음 — 인라인 텍스트가 흔들리지 않게). `products.alm=false`면 호출 자체를 안 한다.
  - 구현은 remark 플러그인이 아니라 React 쪽: 링크 렌더 컴포넌트(`components` 매핑의 `a`)가 `almIssueLink()`로 키를 얻고, 페이지 단위 훅 `useAlmIssueCards(keys)`가 배치 호출·캐시를 맡는다(링크마다 호출 금지 — 요청 폭포 규칙).
  - 목업 모드: `store/mock`에 카드 2~3개 고정 응답(키 `ALM-1`·`WIKI-7`), 테스트가 이걸 쓴다.
- **슬래시 메뉴:** "ALM 이슈"(아이콘 lucide `ticket` 또는 `square-check`) — `products.alm`일 때만 목록에 노출. 선택 → 작은 입력(기존 "이미지 (URL)" 항목이 쓰는 입력 방식 재사용) → `[KEY](/alm/browse/KEY)` 삽입. 키 형식 틀리면 삽입 안 함 + 안내.
- **테스트:** almIssueLink 파서(두 주소 형태·상대/절대·소문자·오탐), normalizeFeatures(products 없음→false), MarkdownView 승격(카드 있음/없음/ALM 꺼짐/실패), 배치 1회 호출, 슬래시 항목 노출 조건.

## 5. 구성 명세 (디자이너 → 프론트, DS 0.17.0 기준)
레퍼런스: Confluence 인라인 스마트 링크(= Jira 이슈 URL): `[이슈타입 아이콘] KEY: 요약 [상태 Lozenge]`, 완료 이슈는 키에 취소선(Jira 관례). 호버 시 밑줄.
- 마크업: `<a class="alm-smart-link" href=url data-kind=…>` 안에 `span.alm-smart-link__icon`(타입 아이콘 16px, 없으면 생략) · `span.alm-smart-link__key`(KEY, 완료면 `text-decoration: line-through`) · `span.alm-smart-link__title`(제목, 한 줄, 너무 길면 말줄임 최대 40자 상당 — `max-width` 토큰 없음 → `ch` 단위) · DS `<Lozenge appearance={status.color ?? "neutral"}>{status.name}</Lozenge>`.
- 토큰만: 색 `var(--chanho-color-link)`/텍스트 `var(--chanho-color-text)`, 간격 `var(--chanho-space-1)`, radius `var(--chanho-radius-sm)`, 배경 호버 `var(--chanho-color-background-neutral-subtle)` (실존 토큰 이름은 wiki-front `app.css`의 bookmark-card 블록과 같은 패밀리로 맞춘다 — 없으면 `--wiki-*` 확장 패밀리). 다크는 토큰이 처리.
- 아이콘: 레지스트리 `icon`은 alm-front `StatusGlyph/PriorityGlyph`가 쓰는 이름 체계 — 프론트 엔지니어가 그 매핑을 확인해 **wiki-front에 작은 매핑 복사**(제품 간 import 금지). 모르는 이름은 아이콘 생략(텍스트만으로도 의미 전달 — 색만으로 상태 구분 금지, 배지에 이름이 있다).
- 상태: 로딩=원래 링크, 실패=원래 링크, 성공=스마트 링크. 포커스 링은 기존 링크와 동일(`:focus-visible`). `title` 속성에 "담당자 · 기한" 요약(없으면 생략). 스크린리더: 링크 텍스트가 "PLAT-12 제목, 진행 중"으로 읽히도록 Lozenge 앞에 시각적으로 숨긴 ", "를 두지 말고 `aria-label`을 링크에 합성.
- 레퍼런스 없음 — 임시 적용·후속 확인: 제목 말줄임 길이(40ch), 담당자 아바타는 넣지 않음(인라인 높이 유지).

## 6. 영향 리포·크로스커팅
| 리포 | 변경 | 비고 |
|---|---|---|
| alm-backend | cards 엔드포인트·서비스·테스트 | PAT read는 GET이라 게이트웨이 규칙 변경 없음 |
| gateway-server | features `products` + `platform.products` 설정·테스트 | wiki-front 어댑터와 같은 배포 묶음 |
| alm-front | `/browse/:key` 라우트, issueLink 정규 주소 | DS 0.16.0 그대로 |
| wiki-front | 링크 인식·배치 조회·스마트 렌더·features·슬래시·목업·테스트 | DS 0.17.0 Lozenge 사용, DS 변경 없음 |
| infra | 없음(기본값 전부 설치). `.env.example`에 `PLATFORM_PRODUCTS` 주석 1줄은 K1 때 | |

배포 순서: alm-backend → gateway → alm-front → wiki-front(앞 셋이 없어도 wiki-front는 일반 링크로 안전하다).

## 7-0. 결과 (2026-11-08)
- 커밋: alm-backend eb7dd34 · gateway 280c6cc(master) · alm-front 0053317 · wiki-front 633f793 → 7ffbd34(리뷰 I1·M1 + 하이라이트 상태 카드 도착 크래시) → 0e09f20(재리뷰 R1). 리뷰어 판정 머지 가능. 배포 순서대로 전부 배포.
- 실스택 E2E 7/7(alice, 스페이스 1): browse·모달 주소 둘 다 승격, 키·제목·상태명·Lozenge·aria-label, 없는 키 일반 링크, 클릭 → `/alm/projects/3/issues?issue=AASD-6`.
- 구현이 스펙과 달리 정한 것: 서버 50키 상한은 유효 키 기준·반복 파라미터 허용·org 불능은 503(빈 배열 아님); 게이트웨이 빈 `PLATFORM_PRODUCTS`=전부 설치; 프론트 토큰 치환(--chanho-color-text-link 등)·조각 간격은 공백 한 칸·인라인 댓글 앵커는 `data-anchor-ignore`로 제목·배지 제외.
- 후속 목록은 49 작업 목록 §0 W3 항목.

## 7. 열린 결정(후속, 이번엔 임시값)
- ❓ 카드/임베드 표시 추가 여부(`::bookmark` 확장) — Could.
- ❓ 편집기 안 스마트 렌더 — Should, 별도.
- ⚠️ 60초 캐시·50키 상한 — 임시값, 실사용 뒤 조정.
- ❓ `PLATFORM_PRODUCTS`를 K1에서 compose 프로필·404로 확장 — 그 문서(`plans/2026-10-24-admin-console.md` C1) 갱신 필요.

## 8. 작업 규칙(요약)
main 직접, 파일 지정 스테이징, 커밋 키 없음(MCP 401 — 제목은 `feat(alm): … 스마트 링크` 식), 테스트 약화 금지, 게이트(프론트 `pnpm typecheck && pnpm test`, 백엔드 `gradle test`) 통과 전 커밋 금지, push는 오케스트레이터가 리뷰 뒤 한다(서브에이전트는 push 금지).

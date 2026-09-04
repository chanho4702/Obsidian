---
tags: [msa, template, alm, uiux, 필드구성, 디자인시스템, 타임라인, 이메일알림, 휴지통, 병렬세션]
작성일: 2026-09-04
상태: 구현 완료 · main 푸시 · CI 그린 (alm-front 305cd14, alm-backend ab4848d, design-system v0.8.1, infra 37df1fb)
---

# 35 — ALM 화면 손질 2차 + 필드 구성 스킴 + DS 0.8 모달 버그 + 이메일·휴지통 V19 (2026-09-04)

상위: [[00 개요 — 전체 구조]] · 직전: [[32 위키 파리티 완결 + 구조부채 S-01~03 + PDF 내보내기 (2026-08-30~09-04)]] · 같은 날 위키 쪽: [[33 위키 UI·UX 손질 + W27 템플릿·수식·다이어그램·구독·검증 + S-03 마감 (2026-09-04)]]
정본: `alm-front/docs/superpowers/specs/2026-09-04-ui-polish-pass.md`, `2026-09-04-field-configuration.md`, `alm-front/docs/STATUS.md` "UX 정비 2차"

> 시작은 "ALM 남은 작업 이어서, 프론트 화면단 전반 UI/UX 수정". 진행 중에 사용자가 두 가지를 더 던졌다 —
> "이슈 생성 모달 우측이 하나도 수정이 안 됨 + 그 항목들 전역·프로젝트별 커스텀 있어야 해", "타임라인 개구려".
> 6롤 하네스로 프론트 3 + 백엔드 1 + 리뷰 2를 병렬로 돌렸고, 같은 시간에 다른 Claude 세션 둘이 alm-front(S-03 레지스트리 전환)와
> wiki-front(W27·공개 문서 인스턴스)를 만지고 있어 세션 간 메시지로 커밋 경계를 조율했다.

> [!note] 화면 감사 방법
> Chrome 확장(claude-in-chrome)은 localhost·LAN 주소에 접근하지 못한다. 스크래치에 `playwright-core` + 이미 설치된
> `ms-playwright/chromium-1234`로 `shot.mjs`(전 라우트 1440×900, 라이트/다크)를 만들어 캡처했다. 이후 모든 검증(모달 Select 클릭,
> 타임라인 막대 클릭)도 같은 방식.

## 1. 화면 손질 2차 — 캡처 감사 → 지라 밀도·헤더 기준

| 화면 | 바꾼 것 |
|---|---|
| 공통 | 시간 표기 `components/time.ts` 하나로(날짜 `2026-09-04`·일시·상대) — 알려진 이슈 #3 해소. 프로젝트 헤더 축소(콘텐츠 y≈180→130), 별표 lucide Star |
| 보드 | 한 줄 툴바(검색·아바타·타입/라벨 라벨 시각 숨김 → 기본 선택지는 "모든 타입"), 스프린트 남은 일수 + **스프린트 완료** 버튼, 컬럼 헤더는 상태 점 + 플레인 텍스트, 카드 우선순위는 레지스트리 아이콘(`aria-label` 병행) |
| 백로그 | 제목 행(백로그 · 이슈 n개), 플랫 행(테두리 없음), 인라인 생성 토글(성공 시 닫힘) |
| 이슈 목록 | 검색 페이지와 같은 칩 드롭다운 한 줄(`FilterDropdown multiple={false}`), 대량 작업 바는 선택 시에만, 표 위 툴바 |
| 홈 | 오늘 요약 줄(배정·기한 지남·이번 주 마감 → 탭 이동), 이어서 하기 4장, 폭 1040 |
| 이슈 모달 | 헤더 한 줄(경로·키 ··· 관찰·닫기), 하위 이슈/링크 폼 기본 접힘, 속성 패널 밀도 축소, 상단 상태 Lozenge 제거 |
| 요약·리포트·릴리스·대시보드 목록 | 카드 행 stretch, 툴바 한 줄, 만들기 버튼 → 인라인 폼 |
| 프로젝트 목록 | 별표 열의 `·` 잔상 = 셀 폭 부족으로 잘린 말줄임표 → 열 56px |

칩 문구 기준(엔지니어 둘이 합의): **라벨을 숨긴 DS Select는 "모든 X"**, 필터 이름이 트리거에 남는 칩은 "전체".

## 2. "우측이 하나도 수정 안 됨" — 디자인시스템 z-index 버그

- 재현: 상세 모달 우측 Select를 열면 옵션 목록이 모달 **블랭킷 아래**에 깔려 클릭이 오버레이에 막힌다. 오늘 변경 이전(git stash)에도 동일.
- 원인: `@chanho/tokens` z 층위 `dropdown 400 < blanket 500 < modal 510`. jsdom 테스트는 포인터 가림을 모르므로 전부 통과했었다.
- 수정: tokens **0.4.0** `z.popover = 550`, react **0.8.0** Select·Dropdown·InlineEdit 팝업이 그 층 사용. v0.8.0 태그 → GitHub Packages 발행 → alm-front·wiki-front 범프.
  같은 날 react **0.8.1**: Switch disabled가 배경색을 덮어 "켜진 채 잠김"이 안 보이던 것을 opacity로.
- 교훈: **모달 안 팝업은 실제 브라우저로 확인할 것.** DS 갭은 DS에서 고치고 버전으로 소비한다(app.css 우회 금지).

## 3. 이슈 필드 구성 스킴 — 전역 + 프로젝트별 커스텀

기존 설정 모델(스킴 → 프로젝트 배정 → "이 프로젝트만 커스텀", `resolveSettings` 단일 진실)에 `SettingsBody.fields`를 얹었다. 서버 본문이 JSON TEXT라 **마이그레이션 없음**.

- 13종: description·assignee·priority·labels·components·parent·sprint·dueDate·fixVersion·resolution·estimate·attachments·links — 각 `visible`/`required`. 응답은 항상 정규화된 13개(구버전 본문은 기본값).
- 규칙(서버 400, 목업 동일 문구): 모르는/빈 id, 중복, 숨김+필수, **resolution·parent는 필수 불가**(프로젝트 단위 구성이라 parent 필수면 최상위 이슈를 못 만든다 — 타입별 스킴은 후속), `fields: []`는 기본값 복원.
- 생성 시 필수 검사(`createNumbered` — POST 이슈·CSV 가져오기 모두 통과): `"{필드}은/는 필수입니다"`. PUT은 검사 안 함. attachments·links는 필수여도 생성을 막지 않음.
- 화면: 전역 관리 **필드 구성**(스킴별 표: 표시/필수 Switch, 잠긴 행은 사유 표시), 프로젝트 설정 **필드**(커스텀 전환 재사용, 읽기 전용은 텍스트). 만들기 모달·상세 속성 패널·대량 변경이 따른다.
- 리뷰가 잡은 막다른 구성 2건: 수정 버전·예상 시간을 필수로 켜면 만들기 모달에 입력이 없어 생성 불가 → **두 입력을 모달에 추가**(서버는 생성 시 `fixVersionId`를 받고도 버리던 버그를 같이 고침), 컴포넌트가 없는 프로젝트에서 컴포넌트 필수면 안내 + 만들기 버튼 옆 "필수 항목 미입력: …" 한 줄.

## 4. 타임라인 재구성 — "개구려"

SVAR React Gantt는 유지하고 지라 타임라인 구조로: 왼쪽 한 열(글리프·키·제목, 시작/종료 열 제거), 격자 제거·주말 밴드·주/월 경계선만, 막대 24/40 radius 4(에픽 `background-inverse`·이슈 brand·완료 success), 마감 없는 이슈는 점 마커, 오늘 danger 선, 툴바 `오늘` + 일/주/월 세그먼트. 대비는 전부 4.5:1 이상(리뷰 실측).

SVAR 함정 세 가지: `markers`는 이 배포본에서 스토어가 비워 안 그려진다(`highlightTime`으로 대체) · 격자는 클래스 없는 div의 생성 배경이라 `.wx-area > div:not([class])`로만 숨길 수 있다 · **`onSelectTask`(PascalCase)여야 발화** — 소문자 `onselecttask`는 `on${string}` 인덱스 시그니처 때문에 타입은 통과하지만 절대 안 불려 막대·행 클릭이 처음부터 안 됐었다(수정, 실제 브라우저 검증).

## 5. ALM 잔여 백엔드 — V19

| 항목 | 내용 |
|---|---|
| 휴지통 자동 비우기 | `ALM_TRASH_RETENTION_DAYS`(60)·`ALM_TRASH_PURGE_CRON`(03:00)·`ALM_TRASH_PURGE_ENABLED`. 손 삭제와 같은 `purgeInternal` 경로, 프로젝트별 REQUIRES_NEW, 회차 상한 500, **삭제 직전 임계값 재확인**(선정 뒤 복원·재삭제된 프로젝트 보호 — 리뷰 Important). 응답 `purgeAt` → 휴지통 화면 "n일 후 영구 삭제" |
| 이메일 알림 | wiki W23 미러: `ALM_MAIL_*`(HOST 비면 no-op), 커밋 뒤 데몬 스레드 발송, 실패 warn. 주소는 **JWT email 클레임 스냅샷**(`user_preference.email`) — org-service proto에 이메일 RPC가 없다. 개인 설정 "이메일로도 받기"(`emailEnabled`) + 서버 구성 여부(`mailConfigured`) 안내. digest 모드는 범위 밖 |
| 인프라 | compose·`.env.example`에 `ALM_MAIL_*`·`ALM_PUBLIC_URL`·`ALM_TRASH_RETENTION_DAYS` |

## 6. 병렬 세션 조율에서 배운 것

- 같은 작업 트리를 세션 둘이 쓰면 **커밋 경계를 파일 목록으로 명시**하고 서로 알린다(alm-front: S-03 파일은 그쪽, 버전 범프 4파일은 내 커밋에 포함하기로 합의).
- 서브에이전트 병렬 편집은 `app.css`를 Edit 부분 치환만으로, 구획 분담 + 공통 유틸은 파일 끝 한 곳 — 충돌 0건.
- 리뷰어 보고가 최종 텍스트로만 남으면 오케스트레이터에 안 온다 — SendMessage로 받도록 지시.
- 테스트 4건은 병렬 워커 부하 플레이키 — 개별 `30_000` 타임아웃.

## 7. 남은 것 (2026-09-04 기준)

| 항목 | 상태 |
|---|---|
| SMTP 실제 연결 | `.env`에 `ALM_MAIL_HOST`/`WIKI_MAIL_HOST` 넣어야 발송 — **사용자 결정** |
| ALM 통합 검색 | platform-backend `feat/wave-d-alm-search`(main 대비 5커밋) 머지 여부 — **사용자 결정**(OpenSearch 의존) |
| 필드 구성 타입별 스킴 | 지라식 이슈 타입별 필드 구성은 후속 후보 — 필요 여부 **사용자 결정** |
| board-service 푸시 | `backend-server` 리포 `GH_PACKAGES_TOKEN` 시크릿 — **사용자 작업** |
| OpenSearch 재색인 | `/admin/search` 1회 — 로그인 필요, **사용자 작업** |
| Keycloak | ADMIN 롤 부여·T1 구글 E2E·T4 운영 설정 — **사용자 작업** |
| 가져오기 라이브 커넥터 | 사용자 보류 |
| DS | wiki-front·myFront를 0.8.1로 올리기(Switch 패치), Table 헤더에 노드 허용(이슈 목록 헤더 체크박스 DS 갭) |
| 문서 게시 | 이 문서를 공개 문서 인스턴스로 `npm run sync:docs`(myFront) — 인스턴스 세션 쪽 파이프라인 |

발행 이력: design-system v0.8.0(tokens 0.4.0) → v0.8.1 · alm-backend V19.

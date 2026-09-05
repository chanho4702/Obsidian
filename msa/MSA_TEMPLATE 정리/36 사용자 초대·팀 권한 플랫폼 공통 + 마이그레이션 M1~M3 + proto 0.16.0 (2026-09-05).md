---
tags: [msa, template, org-service, 초대, 팀, 권한, keycloak, org-admin, 마이그레이션, proto, 병렬세션]
작성일: 2026-09-05
상태: 사용자 초대·팀 권한 U1~U4 main 푸시·CI 그린 / 이관 M1~M3 완료(위키 밖 별도 모듈로 분리 예정, A안) / proto 0.16.0
---

# 36 — 사용자 초대·팀 권한 플랫폼 공통 + 마이그레이션 M1~M3 + proto 0.16.0 (2026-09-05)

상위: [[00 개요 — 전체 구조]] · 직전: [[33 위키 UI·UX 손질 + W27 템플릿·수식·다이어그램·구독·검증 + S-03 마감 (2026-09-04)]] · 병행: [[35 ALM 화면 손질 2차 + 필드 구성 스킴 + DS 0.8 모달 버그 + 이메일·휴지통 V19 (2026-09-04)]]
정본: `platform-backend/docs/superpowers/specs/2026-09-05-user-invite-team-permission-design.md`, `wiki-front/docs/superpowers/specs/2026-09-05-confluence-dc-migration-design.md`

> "이관보다 사용자 관련 작업이 먼저 — 사용자 추가·초대, 그룹 권한 설계·개발." 같은 날 세 세션(이 세션·ALM 세션·공개 문서 인스턴스 세션)이
> platform-backend·design-system·wiki-front를 동시에 만졌고, 분담·Flyway 번호·태그 흐름을 메시지로 맞췄다.

## 1. 결정

| 물음 | 결정 |
|---|---|
| 초대 없는 가입 | **차단** — Keycloak `registrationAllowed`는 유지(초대받은 비밀번호 사용자가 계정을 만들 길)하고 플랫폼이 PENDING으로 격리, 관리자 승인 |
| 관리 화면 위치 | **공용 패키지 `@chanho/org-admin`** 하나를 wiki `/admin/org`·alm `/settings/org`에 마운트 |
| 초대 메일 | SMTP 없으면 **초대 링크 복사**(재발송이 새 링크), 설정되면 메일 |
| 전역 관리자 판정 | **org-service grant 하나**(Keycloak `ADMIN` 역할은 부트스트랩용). ALM 백엔드는 gRPC 판정으로 전환(ALM 세션) |
| 노션·컨플 이관 | **위키 밖 별도 모듈(A: 별도 서비스 + 위키 import API)** — 사용자 작업 뒤. 그때까지 위키 `migration/`에 기능 추가 금지 |

## 2. 왜 초대가 이메일 키인가

id 사슬이 Keycloak `sub`(UUID) → auth-server `users.id`(숫자, **첫 로그인 때 생성**) → org `member.id`(첫 API 호출 때 미러)라서, 로그인 전에는 그 사람의 id가
없다. 그래서 `invitation`은 이메일(정규화)을 키로 두고, `MemberMirrorFilter`가 첫 로그인 때 PENDING 초대를 찾아 소진(팀·grant 적용, EVERYONE 팀 추가)한다.
토큰은 링크 검증·`login_hint`·수락 추적용이고, 구글 로그인은 토큰 없이도 이메일 매칭으로 수락된다. 이메일 검증(`verifyEmail`)이 없는 비밀번호 가입은 토큰 링크
경유만 수락(SMTP 생기면 완화).

## 3. 들어간 것

| 리포 | 내용 |
|---|---|
| platform-backend `c4baf46` | org-service V4 invitation(+team/grant 프리셋), V5 member 수명주기(PENDING/ACTIVE/SUSPENDED/DEACTIVATED, member_event), V6 team.kind(EVERYONE 시드·백필). API: `/me`(globalRoles·teams·status), `/members`(**배열 유지** + status/kind/q, 기본 ACTIVE·HUMAN), `/members/page`, PATCH status, approve, pending, invitations(생성·목록·재발송·철회), 내부 `/internal/org/invitations`(X-Internal-Token), PATCH grants(마지막 GLOBAL ADMIN 409), 팀 LEAD 권한, EVERYONE 수동 변경 400, 초대 메일(선택), Keycloak `platform-admin` 서비스 계정으로 DEACTIVATED 시 계정 비활성. **proto 0.16.0**: GetMembers(ids), CheckPermissionResponse.denied_reason(상태 거부는 오류가 아니라 거부 응답). 210 테스트 |
| auth-server `a81b05e` | `GET /invite/{token}` → org 내부 검증 → Keycloak 로그인(`login_hint`); 로그인 성공 시 초대 수락 호출; `/api/me` roles 배열. 85 테스트 |
| design-system `ef23a17`·`fa14271` | `packages/org-admin` → `@chanho4702/org-admin` 0.1.0→**0.1.1**(basePath 정규화). 화면 5(사용자·초대·팀·전역 역할·승인 대기) + PendingApprovalGate. 호스트가 인증 fetch 주입, 패키지는 `/api/org/*`만 안다. publish.yml 트리거 `org-admin-v*`. 65 테스트 |
| wiki-front `65253ca` | `/admin/org` 마운트, `/admin/teams` 리다이렉트·TeamsAdminPage 삭제, 승인 대기 게이트, 전역 관리자 판정을 `/api/org/me`로(전엔 검색 색인 현황을 찔러 판단 → 색인 장애가 "관리자 아님"으로 둔갑), 목업 `/api/org/*` 라우터, 스페이스 권한 대상 검색·초대 링크. 1146 테스트 |
| infra-settings `fcb08e5` | realm `platform-admin` 클라이언트(service account, view/manage-users), compose env(ORG_INTERNAL_TOKEN·KC_ADMIN_CLIENT_*·PLATFORM_MAIL_*·초대 base URL), .env.example |
| gateway-server `b190a61` | `/internal/org/**` 미라우팅 주석. nginx도 `/internal/`을 넘기지 않음(정적 폴백) |
| wiki-backend `f1a7341`·board `588a98d`·alm-backend `aaf7746` | common-proto/starter **0.16.0** 단일 버전 |

## 4. 같은 날 앞서 끝난 것 — 컨플 DC 이관 M1~M3

기획(`docs/roadmap/2026-09-05-confluence-dc-migration-module.md`) → 설계 §1~§5 → M1(DC 클라이언트·probe/discover·핸들러 5종·IR→마크다운·ImportedPageWriter·`/admin/migrations`),
M2(첨부 스테이징→첨부 레코드·본문 URL, 링크 `dc-page:` 임시 스킴 + 잡 마무리 fixup, 제한 fail-closed, 형제 순서), M3(블로그·댓글(인라인 강등)·이력 N개·"이관됨 · 원본 이름"),
주체 매핑(proto 0.15.0 LookupMembers/LookupTeams → GrpcMigrationPrincipalResolver). wiki-backend 430·wiki-front 1138 테스트. 운영 가이드는 wiki-backend README.
**단, 사용자 결정으로 이관은 위키 밖 별도 모듈(A안)로 빼야 한다** — 사용자 작업 뒤 착수.

## 5. 리뷰에서 바뀐 것 (ALM 세션)

`GET /api/org/members` shape 변경은 alm 13·wiki 10화면을 깨므로 **배열 유지 + `/members/page` 분리**. displayName 편집은 미러가 매 요청 덮어써서 제외.
gRPC 거부와 장애를 구분해야 하므로 `denied_reason` + 상태 거부는 정상 응답. id→email 방향 `GetMembers` 추가. 패키지 `basePath`는 임의(ALM은 `/settings/org`).

## 6. 병렬 세션 조율 기록

- org-service Flyway: 이 세션 V4~V6, ALM 세션 V7(member_profile). 공유 로컬 main에 ALM의 V7 커밋이 먼저 있었지만 **한 푸시에 V4~V7이 함께 pending**이면 번호순 적용이라 순서 문제 없음.
- design-system: `packages/react`·`tokens`는 ALM 세션(0.8.1→0.9.0), `packages/org-admin`·publish.yml·루트 lint:css는 이 세션. 태그는 `org-admin-v*`로 분리해 react/tokens는 skip.
- gateway-server 원격 기본 브랜치는 `master`.
- board-service: 사용자가 `GH_PACKAGES_TOKEN` 등록 → CI 그린(전엔 `[skip ci]` 커밋에 가려 한 번도 안 돌았음).

## 7. 남은 것

| 항목 | 상태 |
|---|---|
| org-admin 0.1.2 | 팀 생성 직후 선택 리셋 버그·초대 프리셋 쿼리(`?scope&resourceId`) 반영 — 패키지 레인 진행 중, 나오면 wiki 범프 |
| 실제 스택 E2E | 초대 생성 → 링크 → 구글/비밀번호 로그인 → 자동 승인·팀·권한 확인, 미초대 로그인 → 승인 대기 → 승인. compose 재기동(realm 재import: `platform-admin` 클라이언트) 필요 |
| ALM 소비 | `/settings/org` 마운트, ALM 관리자 판정 org gRPC(denied_reason, 장애 503), 이메일 GetMembers — ALM 세션 진행 중 |
| wiki-backend | `denied_reason`을 화면 문구로, `GetMembers`로 작성자 이메일 — 후속 |
| SMTP | 생기면 Keycloak `verifyEmail` 켜고 초대 수락 조건 완화, 비밀번호 사용자 사전 생성(execute-actions email) |
| 이관 모듈 분리(A) | 사용자 작업 뒤. 위키 import API 설계부터 |
| 실기 DC 실측 · 검색 재색인 · Keycloak T1/T4 | 사용자 준비 필요(그대로) |

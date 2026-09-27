---
tags: [msa, template, keycloak, oidc, 초대, 권한, org-service, auth-server, 해설]
작성일: 2026-09-27
상태: 해설 — 코드 변경 없음. 2026-09-27 main 기준 코드로 확인한 동작
최종확인: 2026-09-27
---

# 42 — 사람 추가와 권한: Keycloak 원리부터 초대·grant 판정까지 쉽게 (2026-09-27)

이전 구현 기록: [[36 사용자 초대·팀 권한 플랫폼 공통 + 마이그레이션 M1~M3 + proto 0.16.0 (2026-09-05)]] · 로그인 배경: [[14 전체 구현 심화 해설 — 배경지식으로 읽는 현재 아키텍처 (2026-07-17)]]

## 먼저, 한 줄로

**Keycloak에 계정을 우리가 만들지 않는다.** Keycloak은 "이 사람이 누구인지"만 확인한다.
"우리 회사 사람인지, 무엇을 할 수 있는지"는 **org-service**가 자기 DB(`orgdb`)로 정한다.
관리자가 "사람을 추가한다"는 건 Keycloak 계정 생성이 아니라 **초대장 발급**이다.

| 질문 | 누가 답하나 | 어디에 저장 |
|---|---|---|
| 이 사람은 누구인가 (인증) | Keycloak | Keycloak realm `sso-demo` |
| 우리 플랫폼용 신분증 발급 | auth-server | 플랫폼 JWT (auth-server 키로 서명) |
| 우리 조직 구성원인가, 상태는 | org-service | `member` 테이블 |
| 무엇을 할 수 있나 (인가) | org-service | `grant_entry` 테이블 |

---

## 1. Keycloak 원리 — 신분증 발급소

> [!note] 배경지식 — 인증과 인가
> **인증(Authentication)** 은 "너 누구야"이고 **인가(Authorization)** 는 "너 이거 해도 돼"다.
> 이 플랫폼은 인증을 Keycloak에, 인가를 org-service에 맡겨 둘을 완전히 분리했다.

### 1-1. Keycloak 기본 용어

| 용어 | 뜻 | 우리 설정 |
|---|---|---|
| **Realm** | 사용자·설정을 담는 독립 공간. 한 Keycloak에 여러 개 가능 | `sso-demo` 하나 (`infra/keycloak/realm-export.json`) |
| **User** | Keycloak에 저장된 계정. 비밀번호는 여기에만 있다 | 첫 로그인·가입 때 생김 |
| **Client** | Keycloak에 로그인을 맡기는 앱 | auth-server가 클라이언트 |
| **Identity Provider** | 외부 로그인 수단 연결 | `google` |
| **Realm Role** | realm 단위의 넓은 역할 | `USER`, `ADMIN` |
| **Service Account** | 사람 없이 앱이 쓰는 계정 | `platform-admin` (계정 잠금용) |

### 1-2. 로그인 한 번에 일어나는 일 (OIDC Authorization Code)

```
브라우저            auth-server                 Keycloak
  │ "로그인" 클릭 ───▶│                              │
  │◀── 302: Keycloak으로 가라 (client_id, redirect_uri, state)
  │ ─────────────────────────────────────────────▶│ 로그인 화면
  │                   │                              │ 비밀번호 또는 구글
  │◀── 302: auth-server로 돌아가라 (?code=일회용코드) ──│
  │ code 전달 ───────▶│                              │
  │                   │── code를 토큰으로 교환(서버끼리) ▶│
  │                   │◀── ID 토큰(sub, email, name, roles) ─│
  │                   │ 사용자 JIT 생성 + 플랫폼 JWT·RT 발급
  │◀── RT 쿠키 + /app으로 이동
```

핵심 네 가지.

1. **우리 앱은 비밀번호를 보지 않는다.** 로그인 화면은 Keycloak이 그린다. 테마로 모양만 바꾼다.
2. **브라우저가 받는 건 일회용 code뿐이다.** 진짜 토큰 교환은 auth-server와 Keycloak이 서버끼리 한다.
3. **Keycloak 토큰은 auth-server에서 멈춘다.** auth-server가 자체 플랫폼 JWT를 새로 발급하고,
   위키·ALM·게이트웨이는 이 JWT만 검증한다(JWKS·issuer·audience — `common-starter`).
   Keycloak의 `realm_access.roles`는 플랫폼 JWT의 `roles` 클레임으로 옮겨 실린다.
4. **Keycloak 계정은 사용자가 스스로 만든다.** 구글 로그인이면 Keycloak이 자동 생성하고,
   `registrationAllowed: true`라 자체 가입도 열려 있다. 즉 **Keycloak 계정은 누구나 생길 수 있다는 전제**이고,
   문지기는 org-service다.

### 1-3. 왜 권한을 Keycloak에 두지 않았나

- Keycloak 역할은 "관리자/사용자" 같은 **넓은 등급**에 맞다. "스페이스 A의 편집자, 프로젝트 B의 보기 전용"처럼
  리소스가 수백 개로 늘어나는 권한은 역할 폭발이 난다.
- **로그인 수단을 바꿔도 권한이 그대로다.** 고객사가 구글 대신 사내 AD·LDAP를 Keycloak에 붙여도
  org-service의 팀·권한은 영향이 없다. 설치형에서 가장 큰 장점이다.
- 권한 변경이 **즉시 반영**된다. JWT에 권한을 실으면 토큰이 만료될 때까지 옛 권한이 남는다.
  org-service는 요청마다 DB를 본다.

---

## 2. 사람 추가 — 초대 흐름

### 2-1. 전체 그림

```
관리자             org-service               사용자               auth-server / Keycloak
  │ 초대(이메일·팀·권한) │                         │                          │
  │──────────────────▶│ invitation 저장(PENDING)  │                          │
  │                   │ 메일 발송 또는 링크 복사 ──▶│                          │
  │                   │                         │ /invite/{토큰} 클릭 ──────▶│ 토큰 유효 확인(org에 질의)
  │                   │                         │                          │ 세션에 토큰 보관
  │                   │                         │◀── Keycloak 로그인(이메일 미리 채움, login_hint)
  │                   │                         │ 로그인·가입 ─────────────▶│ 신원 확인
  │                   │◀──────────── 초대 수락 통보(user id, 이메일, 이름) ──────│ 플랫폼 JWT 발급
  │                   │ member ACTIVE + 예약된 팀·권한 적용                       │
```

### 2-2. 단계별 설명

1. **관리자가 초대를 만든다.** 관리 화면(`@chanho4702/org-admin` — 위키 `/wiki/admin/org`, ALM `/alm/settings/org`)에서
   이메일, 들어갈 팀, 줄 권한을 고른다.
2. **org-service가 초대를 저장한다** (`V4__invitation.sql`).
   - 키는 **이메일**이다. 초대 시점엔 그 사람이 로그인한 적이 없어 user id가 없기 때문이다.
   - 팀·권한은 `invitation_team`·`invitation_grant`에 **예약**만 해 둔다. 붙일 대상이 아직 없다.
   - 링크 토큰은 **해시만** 저장한다. DB가 새도 초대 링크를 위조할 수 없다.
   - 한 이메일에 살아 있는 초대는 하나뿐이다. 다시 보내면 이전 초대는 `EXPIRED`가 된다.
3. **메일 또는 링크 복사.** `MAIL_MODE`가 켜져 있으면 메일이 가고, 아니면 관리 화면에서 링크를 복사해 전달한다.
4. **링크 클릭 — auth-server가 받는다** (`auth-server/.../invite/InviteController.java`).
   토큰이 살아 있으면 세션에 담고 Keycloak 로그인으로 보낸다. 죽은 링크면 오류 대신 안내 페이지를 보여 준다.
   초대 이메일은 `login_hint`로 넘겨 로그인 칸에 미리 채운다.
5. **로그인 성공 — 이 순간 user id가 처음 생긴다** (`LoginSuccessHandler`).
   auth-server가 사용자를 JIT 생성하고, 세션의 초대 토큰으로 org-service에 **수락을 통보**한다.
   org-service는 예약해 둔 팀·권한을 실제 `team_member`·`grant_entry`로 옮긴다.
   초대 이메일과 다른 계정으로 로그인하면 수락이 거절된다.
   통보가 실패해도 로그인은 계속한다 — 아래 이메일 대조 경로가 남아 있기 때문이다.
6. **첫 API 요청 — 한 번 더 확인** (`org-service/.../security/MemberMirrorFilter.java`).
   JWT의 sub·이메일·이름으로 `member` 행을 만들거나 갱신한다. 토큰 통보가 빠졌더라도
   그 이메일에 살아 있는 초대가 있으면 여기서 소진한다(`accepted_via = EMAIL_MATCH`).

### 2-3. 초대 없이 들어온 사람

Keycloak 로그인은 된다. 하지만 org-service가 `PENDING`(승인 대기)으로 격리한다.

- `/api/org/me`만 통과한다. 프론트가 "승인 대기" 화면을 그리려면 자기 상태는 읽어야 하기 때문이다.
- 나머지 org API는 403, 위키·ALM은 gRPC 권한 판정이 `ACTIVE`가 아니면 무조건 거부한다.
- 관리자가 관리 화면의 **승인 대기** 탭에서 승인하면 `ACTIVE`가 된다.

### 2-4. 사람의 상태

| 상태 | 뜻 | 되돌리기 |
|---|---|---|
| `PENDING` | 초대 없이 들어옴, 승인 대기 | 관리자 승인 |
| `ACTIVE` | 정상 | — |
| `SUSPENDED` | 일시 정지 | 관리자가 해제 |
| `DEACTIVATED` | 퇴사. **Keycloak 계정도 잠근다** | 재초대 |

---

## 3. 권한 — grant 원장 하나

### 3-1. 저장 구조

권한은 전부 org-service `grant_entry` 테이블 하나에 있다. 한 줄 = "누가, 어디에, 어떤 역할".

| 칸 | 값 |
|---|---|
| 누가 (`subject_type`·`subject_id`) | `USER`(사람) 또는 `TEAM`(팀) |
| 어디에 (`resource_type`·`resource_id`) | `GLOBAL`(전역, id는 빈 값), `SPACE`(위키 스페이스), `PROJECT`(ALM 프로젝트) |
| 역할 (`role`) | `VIEWER` < `COMMENTER` < `EDITOR` < `ADMIN` |

역할은 계단식이다. `ADMIN`은 편집·댓글·보기를 모두 포함하고, `COMMENTER`는 보고 댓글만 단다.
권한 변경은 `grant_audit`에 감사 기록이 남는다.

### 3-2. 판정 방법 (`PermissionFacade.check`)

위키에서 페이지를 고치려는 순간을 예로 들면 이렇다.

1. wiki-backend가 gRPC로 org-service에 묻는다: "user 7이 SPACE 12를 EDIT 할 수 있나?"
2. org-service가 모은다: user 7에게 **직접 준** grant + user 7이 속한 **팀들에 준** grant (그 스페이스 대상).
3. 그중 **가장 높은 역할**을 고른다.
4. 그 역할이 요구 수준 이상이면 허용, 아니면 거부. grant가 하나도 없으면 거부.
5. 사람이 `ACTIVE`가 아니면 grant가 뭐가 있든 거부한다(fail-closed).

예: 김대리가 직접 `VIEWER`, 소속 "개발팀"이 `EDITOR`를 받았으면 유효 역할은 `EDITOR`다.

### 3-3. 팀

- 모든 사람은 자동으로 **EVERYONE(전체 구성원)** 팀에 들어간다. 스페이스를 전사 공개하려면 이 팀에 `VIEWER`를 준다.
- 팀 안 역할은 `LEAD`·`MEMBER`다. 팀에 권한을 주면 소속원 전원에게 적용된다.
- 팀은 org-service에서 직접 만든다. **AD·LDAP 그룹과의 자동 동기화는 없다** (아래 §6).

### 3-4. 누가 권한을 줄 수 있나

- **전역 관리자**(`GLOBAL ADMIN` grant)는 전부 관리한다.
- **스페이스·프로젝트 관리자**는 그 리소스의 권한만 관리한다. 스페이스 소유자가 자기 공간에 사람을 부를 수 있어야 하기 때문이다(컨플루언스와 같다).
- 전역 권한은 전역 관리자만 줄 수 있다.
- **마지막 전역 관리자는 강등할 수 없다**(409). 관리자 없는 플랫폼을 막는다.

---

## 4. 관리자가 두 종류인 이유

헷갈리기 쉬운 부분이다. 관리자 개념이 **둘** 있고 서로 별개다 (`INSTALL.md` §6).

| | Keycloak realm 롤 `ADMIN` | org-service `GLOBAL ADMIN` grant |
|---|---|---|
| 저장 | Keycloak realm | `orgdb.grant_entry` |
| 전달 | 플랫폼 JWT `roles` → `ROLE_ADMIN` | org-service가 DB를 직접 본다 |
| 여는 것 | agent-service 페르소나·PAT·예산, 에이전트 등록, board 관리 우회 | 관리 화면 전체(사용자·초대·팀·권한·승인·메일 설정), 검색 재색인 |
| 부여 | Keycloak admin 콘솔 → Users → Role mapping | 부트스트랩 시드, 또는 기존 관리자가 화면에서 |

**사람·권한 관리의 정본은 org-service grant다.** Keycloak `ADMIN` 롤만 있다고 전역 관리자가 되지 않는다.

---

## 5. Keycloak을 직접 건드리는 곳은 딱 두 군데

### 5-1. 퇴사·복귀 시 계정 잠금 (`org-service/.../keycloak/KeycloakAdminClient.java`)

- 사람을 `DEACTIVATED`로 바꾸면 org-service가 Keycloak 관리 API로 그 계정을 **비활성화**한다. 복귀는 활성화.
- 서비스 계정 `platform-admin`이 `client_credentials`로 토큰을 받아 이메일로 사용자를 찾고 `enabled`만 바꾼다.
- 우리 쪽이 이미 막고 있지만, 계정 자체가 열려 있으면 다른 문제라서 같이 잠근다.
- **Keycloak이 죽어 있어도 퇴사 처리는 진행된다.** 실패는 `member_event`(`KEYCLOAK_DISABLED_FAILED`)에 남는다.
- 시크릿(`KC_ADMIN_CLIENT_SECRET`)이 없으면 아무것도 안 한다(dev·테스트용 no-op).

### 5-2. 그 외에는 없다

계정 생성·비밀번호·이메일 변경은 하지 않는다. 사용자가 Keycloak에서 직접 가입·로그인하고,
비밀번호 재설정도 Keycloak 화면에서 한다.

---

## 6. 첫 관리자 만들기 (새 설치)

새로 설치하면 권한을 줄 관리자가 아무도 없다. 닭과 달걀 문제를 **환경변수 하나**로 푼다.

1. `.env`에 `PLATFORM_BOOTSTRAP_ADMIN_ID`를 적는다. 값은 **auth-server user id**다.
   맨 처음 로그인한 사람이 `1`이고 compose 기본값도 `1`이다. 비우면 부트스트랩을 건너뛴다.
2. org-service가 기동할 때마다 그 id에 `GLOBAL ADMIN` grant를 upsert한다(`BootstrapAdminSeeder`).
3. 그 계정은 초대가 없어도 `PENDING`에 갇히지 않고 곧바로 `ACTIVE`(`joinedVia=BOOTSTRAP`)가 된다.
   초대해 줄 사람도 승인해 줄 사람도 없기 때문이다.
4. 에이전트·PAT 관리까지 쓰려면 Keycloak admin 콘솔에서 realm 롤 `ADMIN`도 준다(§4).
5. 두 번째 사람부터는 관리 화면의 **초대**로 한다.

확인: 로그인한 브라우저에서 `/api/org/me` → `status=ACTIVE`, `joinedVia=BOOTSTRAP`, `globalRoles=["ADMIN"]`.

> [!warning] Keycloak `ADMIN` 롤로 첫 관리자가 되는 게 아니다
> 부트스트랩은 오직 `PLATFORM_BOOTSTRAP_ADMIN_ID`로만 된다. realm을 갓 import하면 테스트 계정 `admin`이
> `USER`+`ADMIN` 롤을 갖고 있는데, 운영에서는 이 기본 비밀번호를 반드시 바꾼다.

---

## 7. 설치형 관점의 강점과 약점

**강점**
- 인증 수단 교체(구글 → AD/LDAP/사내 OIDC)가 Keycloak 설정만으로 끝나고 권한은 그대로다.
- 권한 판정이 org-service 한 곳이라 위키·ALM·게시판이 같은 기준으로 막힌다.
- 초대 없는 가입은 자동 격리, 퇴사는 Keycloak 계정까지 잠금 — 보안 심사에서 설명하기 쉽다.

**약점 (고객 요구로 자주 나올 것)**
- **AD·LDAP 그룹 → org 팀 동기화가 없다.** "AD 그룹이 곧 우리 팀"을 원하는 고객은 팀을 수동으로 다시 만들어야 한다.
- **SCIM 같은 표준 프로비저닝이 없다.** 인사 시스템에서 입·퇴사를 자동 반영할 수 없다.
- **Keycloak 쪽 퇴사 반영은 단방향이다.** 고객 IT가 AD에서 계정을 막아도 org-service `member` 상태는 그대로다
  (로그인이 안 되니 실사용은 막히지만, 관리 화면엔 `ACTIVE`로 보인다).

---

## 코드 위치 모음

| 역할 | 파일 |
|---|---|
| 초대 링크 착지 | `auth-server/src/main/java/com/platform/authserver/invite/InviteController.java` |
| 로그인 성공 후 처리·수락 통보 | `auth-server/.../auth/LoginSuccessHandler.java` |
| 첫 요청 미러링·격리 | `platform-backend/org-service/.../security/MemberMirrorFilter.java` |
| 권한 판정 | `platform-backend/org-service/.../permission/PermissionFacade.java` |
| gRPC 판정 서버 | `platform-backend/org-service/.../grpc/PermissionGrpcService.java` |
| Keycloak 계정 잠금 | `platform-backend/org-service/.../keycloak/KeycloakAdminClient.java` |
| 첫 관리자 시드 | `platform-backend/org-service/.../config/BootstrapAdminSeeder.java` |
| 스키마 | `org-service` `V1__init.sql`(member·team·grant), `V4__invitation.sql`, `V5__member_lifecycle.sql` |
| 위키 쪽 판정 클라이언트 | `wiki-backend/.../permission/GrpcPermissionClient.java` |
| 설치 절차 | `MSA_TEMPLATE/INSTALL.md` §6 |

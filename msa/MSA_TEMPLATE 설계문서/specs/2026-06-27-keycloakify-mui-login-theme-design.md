# keycloakify MUI 로그인 테마 — 설계 문서

- 날짜: 2026-06-27
- 관련 메모리: [[keycloak-jwt-platform-build]], [[auth-model-keycloak-redirect]]
- 작업 위치(코드): `C:\MSA_TEMPLATE\loginPageKeyClock\keycloakify-starter`
- 배포 대상: `C:\MSA_TEMPLATE\infra\keycloak` (docker compose, Keycloak 26)

## 목표

myFront의 `src/app/pages/LoginPage.tsx`와 **동일한 디자인 언어**를 가진 Keycloak
로그인 화면을 keycloakify(React)로 만든다. 기본 로그인(ID/비밀번호) + Google
소셜 로그인 버튼을 포함한다.

### 핵심 사실 (설계 전제)

`LoginPage.tsx`는 실제 로그인 폼이 아니라 **버튼 2개짜리 리다이렉트 화면**이다
(로그인 버튼 → auth-server OIDC 시작점, 구글 버튼 → 동일). 실제 username/password
입력은 **Keycloak 로그인 페이지**에서 일어난다. 따라서 100% 픽셀 동일은 불가능하며,
목표는 *같은 디자인 언어(그라데이션 배경 · MUI Card · Sitemark 로고 · 폰트 · 버튼
스타일)로 자연스럽게 이어지는 화면* 위에 **실제 입력 폼과 구글 소셜 버튼**을 얹는 것.

## 확정된 결정

1. **시각적 동일성**: `@mui/material` + myFront `shared-theme`/`CustomIcons`를
   keycloakify 프로젝트로 **복제(vendor)** 해 진짜 동일 컴포넌트로 렌더. (플레인 CSS
   복제 대신 — 토큰 드리프트 방지)
2. **테마 전환**: keycloakify 테마를 **새 이름 `platform-react`** 로 만들고 realm의
   `loginTheme`을 전환. 기존 FTL `platform` 테마는 롤백용으로 보존.
3. **페이지 범위**: **`login.ftl` 한 페이지만** 커스텀. 나머지(register, reset-password,
   OTP 등)는 keycloakify 기본 테마로 폴백.

## 아키텍처 / 구성요소

keycloakify-starter는 myFront와 **별도 vite 프로젝트**(독립 package.json/node_modules)다.
cross-import가 불가능하므로 필요한 MUI 자산을 vendor한다.

### 단위(unit)별 책임

1. **vendored 테마 자산** — `src/login/theme/`
   - myFront `shared-theme`에서 `AppTheme`, `themePrimitives`(컬러 토큰), 필요한
     customizations만 복제. ColorModeSelect는 로그인 화면에 불필요하면 제외(YAGNI).
   - 의존: `@mui/material`, `@emotion/react`, `@emotion/styled`.
   - 인터페이스: `<AppTheme>{children}</AppTheme>` 래퍼 + 테마 객체.

2. **vendored 아이콘** — `src/login/theme/CustomIcons.tsx`
   - myFront `sign-in/components/CustomIcons.tsx`에서 `SitemarkIcon`, `GoogleIcon`만 복제.

3. **커스텀 로그인 페이지** — `src/login/pages/Login.tsx`
   - keycloakify 기본 `keycloakify/login/pages/Login`을 베이스로 복제 후 MUI로 재작성.
   - 렌더 구성(LoginPage.tsx 룩 + 실제 폼):
     - `<AppTheme>` + 배경 그라데이션 컨테이너 + MUI `<Card variant="outlined">`
     - `<SitemarkIcon/>` → `<Typography variant="h4">로그인</Typography>` → 안내문
     - 에러 표시: `kcContext.message` (예: 잘못된 자격증명)
     - `<form action={url.loginAction} method="post">` 안에:
       - username `<TextField>` (name="username", `kcContext.login.username` 프리필,
         `realm.loginWithEmailAllowed`이면 라벨/타입 조정)
       - password `<TextField type="password">` (name="password")
       - rememberMe 체크박스 — `realm.rememberMe`일 때만
       - `<Button variant="contained" type="submit">로그인</Button>`
     - `<Divider>또는</Divider>`
     - 소셜: `kcContext.social.providers`에서 alias==="google" 항목 →
       `<Button variant="outlined" href={p.loginUrl} startIcon={<GoogleIcon/>}>Google 계정으로 로그인</Button>`
     - registration 링크: `realm.registrationAllowed && !registrationDisabled`일 때 노출
   - 의존: vendored 테마/아이콘, `kcContext`(login.ftl), `i18n`.

4. **페이지 라우팅** — `src/login/KcPage.tsx` (기존 파일 수정)
   - switch에 `case "login.ftl": return <Login .../>` 추가. `default`는 기존
     `DefaultPage` 유지(폴백). 로그인 페이지는 자체 MUI 레이아웃을 그리므로
     `doUseDefaultCss`/Template 의존을 최소화 — 커스텀 Login이 배경·카드까지 자체 렌더.

5. **빌드/배포 와이어링** — `infra/keycloak`
   - keycloakify 테마 이름을 `platform-react`로 설정(package.json의 `keycloakify.themeName`
     또는 vite-plugin 옵션).
   - `npm run build-keycloak-theme` → `dist_keycloak/keycloak-theme-for-kc-22-and-above.jar`.
   - docker-compose에 `./providers:/opt/keycloak/providers` 볼륨 추가 후 jar를 복사.
   - `realm-export.json`의 `"loginTheme": "platform"` → `"platform-react"`.

## 데이터 흐름

1. 사용자가 myFront `/login`에서 "로그인" 클릭 → auth-server(:9000) OIDC 시작 →
   Keycloak `login.ftl`(이제 `platform-react` 테마)로 리다이렉트.
2. keycloakify가 빌드한 `login.ftl`이 React 번들을 로드, `kcContext`를 읽어
   커스텀 `Login.tsx` 렌더.
3. 기본 로그인: 폼 POST → `url.loginAction` → Keycloak 인증 → auth-server 콜백 → `/app`.
4. 구글: `social.providers[google].loginUrl`로 이동 → 구글 OAuth → Keycloak → 콜백.

## 에러 처리

- 잘못된 자격증명 등은 `kcContext.message.summary`를 Card 상단 alert로 표시
  (`message.type`에 따라 error/warning/info 스타일).
- 필드 단위 에러(`messagesPerField.existsError("username","password")`)로 TextField
  `error`/`helperText` 처리.

## 테스트 / 검증

keycloakify는 **Storybook**으로 각 `kcContext` 변형을 mock 렌더할 수 있다.
- `Login.stories`에 케이스: 기본, 에러 메시지 있음, 구글 IdP 노출, rememberMe on.
- 빌드 검증: `npm run build-keycloak-theme` 성공 + jar 생성 확인.
- 통합 수동 검증(사람, T1): docker compose 재기동 → myFront `/login` → "로그인" →
  새 화면이 myFront 룩으로 뜨고 alice/alice 로그인 왕복 성공 + 구글 버튼 노출.

## 범위 밖 (YAGNI)

- register/reset-password/OTP 등 다른 Keycloak 페이지 테마링.
- ColorModeSelect(다크모드 토글) — 로그인 화면엔 불필요.
- 자체 폼 ROPC 전환 — 설계상 리다이렉트 모델 유지([[auth-model-keycloak-redirect]]).

## 미해결/주의

- keycloakify 11 + Keycloak 26 호환: jar는 `kc-22-and-above` 타깃이면 26에서 동작.
  빌드 산출물 이름/배치 경로는 빌드 후 실제 확인해 와이어링 확정.
- vendor한 `themePrimitives`가 MUI v6/v5 어느 API(`theme.applyStyles` 등)를 쓰는지
  keycloakify-starter의 MUI 버전과 맞춰야 함 — 의존성 추가 시 버전 정렬 필요.

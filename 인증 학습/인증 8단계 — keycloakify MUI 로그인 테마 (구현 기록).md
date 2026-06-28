---
작성일: 2026-06-27
태그: [인증, keycloak, keycloakify, react, mui, 테마]
상태: 빌드·배포·헤드리스 검증 완료(로컬+원격 push) · 브라우저 시각확인 대기
---

# 인증 8단계 — keycloakify MUI 로그인 테마 (구현 기록)

> [[인증 8단계 — Keycloak BFF 자체-JWT 플랫폼 (구현 기록)]]의 후속.
> 그 단계에서 만든 Keycloak 로그인 화면을 **myFront 디자인 언어로 맞춘** 작업.
> 로그인 모델 자체는 그대로 — [[인증 6-7단계 OAuth2-OIDC 로그인]]의 리다이렉트(OIDC Authorization Code) 유지.

## 왜 했나

기존 로그인 화면은 Keycloak 기본 테마에 **CSS만** 덧입힌 FTL 테마(`platform`)였다.
myFront `LoginPage`(그라데이션 배경·MUI Card·Sitemark 로고)와 톤은 맞췄지만,
"내 사이트에서 키클락 화면으로 넘어가는 이질감"을 더 줄이려면 **같은 컴포넌트(MUI)로
진짜 동일하게** 그리는 편이 낫다. 그래서 [Keycloakify](https://keycloakify.dev)
(React 코드를 Keycloak 테마 .jar로 컴파일)로 로그인 페이지를 다시 만들었다.

> ⚠️ 100% 픽셀 동일은 불가능하다. myFront `LoginPage.tsx`는 **버튼 2개짜리 리다이렉트
> 화면**이고, 실제 아이디/비밀번호 입력은 Keycloak 로그인 페이지에서 일어난다.
> 목표는 *같은 디자인 언어 위에 실제 입력 폼 + 구글 소셜 버튼*을 얹는 것.

## 무엇을 만들었나

테마명 **`platform-react`** (keycloakify, `login.ftl` 한 페이지만 커스텀, 나머지는 기본 폴백).

- **`src/login/pages/Login.tsx`** — MUI `Card` + 그라데이션 배경 위에:
  - `SitemarkIcon` → "로그인" 제목 → 안내문
  - `kcContext.message` 를 MUI `Alert`로(잘못된 자격증명 등, 서버측 한국어 번역)
  - `<form action={url.loginAction}>` 안에 username/password `TextField`
    (필드명 `username`/`password`, `messagesPerField`로 필드 에러), `credentialId` hidden
  - `realm.rememberMe`면 "로그인 상태 유지" 체크박스, `resetPasswordAllowed`면 재설정 링크
  - `Divider("또는")` + `social.providers` 매핑 → google이면 `GoogleIcon` 버튼
  - `registrationAllowed`면 회원가입 링크
- **`src/login/theme/`** — myFront `shared-theme`/`CustomIcons`를 vendor(복제).
  `AppTheme`, `themePrimitives`(컬러 토큰), inputs/feedback/surfaces customizations,
  `SitemarkIcon`·`GoogleIcon`. (별도 vite 프로젝트라 cross-import 불가 → 복제로 토큰 드리프트 방지.)
- **`src/login/KcPage.tsx`** — `login.ftl` → 커스텀 `Login` 라우팅, `default`는 기존 폴백.
- **`vite.config.ts`** — `themeName: "platform-react"`, `accountThemeImplementation: "none"`.

## repo / 위치

| 항목 | 위치 / 브랜치 |
|---|---|
| keycloakify 소스 | `loginPageKeyClock/keycloakify-starter` · `github.com/chanho4702/keycloakify-starter` · `main` (`dace7a5`) |
| infra 배포 | `MSA_TEMPLATE` · 브랜치 `feat/keycloakify-login-theme` → 원격 `infra-settings` (`94bfc22`) |
| 설계 문서 | `docs/superpowers/specs/2026-06-27-keycloakify-mui-login-theme-design.md` (master `da607cb`) |

빌드: `npm run build-keycloak-theme` → `dist_keycloak/keycloak-theme-for-kc-all-other-versions.jar`(KC26용).
배포: jar를 `infra/keycloak/providers/`에 복사 + compose `./providers:/opt/keycloak/providers` 볼륨.

## 빌드/배포에서 걸린 것

- **Maven 필요(Windows에 없음).** keycloakify build의 jar 패키징 단계가 Apache Maven을
  요구하는데 mvn·choco 둘 다 없었다. `dlcdn.apache.org` 미러는 404 → **archive.apache.org**에서
  portable Maven 3.9.9를 받아 PATH에 얹고 **`JAVA_HOME=C:\java11`(JDK11)** 로 빌드해야 통과.
  (JDK24는 jar/keytool 단계에서 이슈 가능.) Maven 체크가 리소스 생성보다 먼저라 우회 불가.
- **이미 import된 realm은 seed 변경 무시.** `start-dev --import-realm`은 realm이 이미 있으면
  `Realm 'sso-demo' already exists. Import skipped`. 그래서 `realm-export.json`의 loginTheme을
  바꿔도 실행 중 realm엔 반영 안 됨 → **kcadm으로 직접 적용**:
  ```bash
  MSYS_NO_PATHCONV=1 docker exec platform-keycloak /opt/keycloak/bin/kcadm.sh \
    config credentials --server http://localhost:8080 --realm master --user admin --password admin
  MSYS_NO_PATHCONV=1 docker exec platform-keycloak /opt/keycloak/bin/kcadm.sh \
    update realms/sso-demo -s loginTheme=platform-react
  ```
  (Git Bash 경로 변환 때문에 `MSYS_NO_PATHCONV=1` 필요.)
- **providers 볼륨은 컨테이너 재생성 필요.** 새 볼륨 마운트라 `restart`가 아니라 `up -d`(recreate).
  postgres 볼륨은 보존되어 데이터 안전.

## 검증 (2026-06-27, 헤드리스)

컨테이너 재생성 + loginTheme=platform-react 적용 후:
- ✅ realm `loginTheme` = `platform-react` (kcadm get 확인)
- ✅ 로그인 페이지 200 + keycloakify React 번들 로드:
  `resources/.../login/platform-react/dist/assets/index-*.js`, `kcContext`/`loginAction` 포함
- ✅ **폼 로그인 왕복**: `loginAction`에 `username=alice&password=alice` POST →
  **302 + `code=...`** 로 auth-server(:9000) 콜백 → OIDC code flow 완결.
  → 커스텀 `Login.tsx` 폼의 필드명·action이 정확하고 인증이 끝까지 동작함을 증명.

## 남은 것

- **브라우저 시각 확인(사람):** myFront `/login` → "로그인" → 실제로 MUI Card·Sitemark·
  구글 버튼이 myFront 룩으로 뜨는지 + **구글 실제 로그인 왕복**. (기본 폼 로그인은 위에서 증명됨.)
- 롤백 경로: realm `loginTheme`을 `platform`(FTL/CSS, `themes/platform`)으로 되돌리면 즉시 복구.

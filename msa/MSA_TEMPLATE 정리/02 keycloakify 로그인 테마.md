---
tags: [msa, template, keycloak, keycloakify, 테마, 프론트]
작성일: 2026-06-30
상태: 구현 완료
---

# 2단계 — keycloakify 로그인 테마 (platform-react)

상위: [[00 개요 — 전체 구조]]

## 정의

로그인은 Keycloak 리다이렉트 모델([[01 인증 플랫폼 (Keycloak BFF + 자체 JWT)]])을 유지하되, Keycloak 로그인 페이지를 **React + MUI 커스텀 테마**로 교체해 UX 불만을 해결. 테마명 **`platform-react`**.

## 구현

- `keycloakify-starter`(본인 fork) 기반으로 `login.ftl`을 React+MUI로 커스텀(MUI Card·Sitemark·구글 버튼).
- 빌드 `npm run build-keycloak-theme` → `dist_keycloak/*.jar`(KC26용 등).
- 배포: jar를 `infra/keycloak/providers/`에 복사 + compose에 providers 볼륨 마운트. realm `sso-demo`의 `loginTheme=platform-react`.

## 빌드 함정 (기록)

- keycloakify 빌드가 **Maven 요구** → choco 없음, dlcdn 미러 404. **archive.apache.org**에서 portable Maven 3.9.9 받아 `JAVA_HOME=C:\java11`로 빌드해야 통과(JDK24 아님).
- realm이 이미 import돼 있으면 `--import-realm`이 seed JSON 무시 → loginTheme은 kcadm으로 적용(`kcadm update realms/sso-demo -s loginTheme=platform-react`, `MSYS_NO_PATHCONV=1` 필요).
- 이 keycloakify 테마라서 **로그인 폼이 React로 렌더** → 폼 action이 평문 HTML이 아니라 **kcContext JSON의 `loginAction`**(session_code 포함 절대 URL)에 들어있음. (curl로 토큰 받을 때 이 점이 중요 → [[03 board-service (샘플 백엔드 + SSO 증명)]])

## 검증

컨테이너 재생성 후 로그인 페이지가 keycloakify 번들 로드 + alice/alice POST→302 code 왕복 성공(폼 기능 OK). **브라우저 시각 확인 + 구글 실로그인 왕복은 사람 확인 항목으로 남음.**

## 상태

`backend(theme) repo = keycloakify-starter fork`(branch main, push됨). 우산 repo에는 `infra/keycloak/providers/platform-react-keycloak-theme.jar` + realm 설정으로 배포 반영(`infra-settings/main`).

다음: [[03 board-service (샘플 백엔드 + SSO 증명)]]

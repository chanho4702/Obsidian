---
tags: [msa, template, 인증, keycloak, jwt, oidc]
작성일: 2026-06-30
상태: 구현 완료
---

# 1단계 — 인증 플랫폼 (Keycloak BFF + 자체 JWT)

상위: [[00 개요 — 전체 구조]]

## 정의

`myFront → auth-server(:9000) → Keycloak(:8080)` 구조의 **BFF(Backend for Frontend)** 인증 플랫폼. 로그인은 Keycloak OIDC 리다이렉트(+ Google IdP)로 하고, auth-server는 Keycloak 토큰을 그대로 넘기지 않고 **자체 RS256 JWT**를 발급한다.

## 로그인 흐름

1. 프론트 → `GET /oauth2/authorization/keycloak` → Keycloak 로그인 페이지(리다이렉트)
2. 사용자 인증(alice/alice 등 또는 Google) → Keycloak가 auth-server 콜백으로 `code` 전달
3. `LoginSuccessHandler`: JIT 사용자 프로비저닝 + 자체 refresh token 발급(+ KC id_token·refresh_token 보관) → **RT 쿠키 set** → 프론트 `/app` 리다이렉트
4. 프론트 마운트 시 `POST /api/auth/refresh` → **accessToken(자체 RS256 JWT) 수령** (silent restore). AT는 프론트 **메모리에만**, RT는 HttpOnly 쿠키.

## 자체 JWT (핵심 계약)

- 발급자 `iss: http://localhost:9000`, RS256 서명, JWKS는 `/.well-known/jwks.json`로 공개.
- claim: `sub`(사용자 id), `email`, `name`, `provider`(KEYCLOAK/GOOGLE), `roles`(배열).
- `/api/me`가 이 claim을 `{email,name,provider,sub,role}`로 반환(`role`=roles 첫 번째).
- refresh token **회전 + 재사용 탐지**(도난 시 family 폐기). 동시 refresh는 in-flight dedup으로 1요청 합침.

## 로그아웃 = 백채널 (중요)

- 프론트 `logout()` → `POST /api/auth/logout`만 호출(브라우저 리다이렉트 없음).
- auth-server: 자체 RT family 폐기 + RT 쿠키 삭제 + **저장한 KC refresh_token으로 `end_session` 호출**(서버-서버).
- **Why:** 예전 프론트채널(id_token_hint 리다이렉트)은 id_token 수명(수 분)이 앱 세션(자체 RT 14일)보다 짧아 **로그아웃 시 id_token 만료 → KC SSO 안 끊김 → "재로그인 즉시" 버그**. refresh_token은 SSO 세션과 함께 살아 만료 문제가 없음. → 다시 프론트채널로 되돌리지 말 것.

## 트러블슈팅 기록

- Spring Boot 4에서 `spring-boot-flyway` 모듈 누락 → authdb 스키마 미생성 런타임 버그(수정됨).

## 상태

자체 repo `github.com/chanho4702/auth-server`, branch `main`(로컬). 백엔드 통합 E2E를 curl로 검증(alice 로그인→refresh→/api/me).

다음: [[02 keycloakify 로그인 테마]] · 관련: [[03 board-service (샘플 백엔드 + SSO 증명)]]

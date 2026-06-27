---
tags: [pms, 인증, 백로그]
status: 보류 (MSA 템플릿 완성 후 착수)
관련: "[[msa-roadmap]]"
---

# PMS 백로그 (인증 / 권한)

> [!info] 이 노트의 위치
> PMS는 **MSA 템플릿 위에 올릴 첫 번째 실제 서비스**다. 순서는 **템플릿(플랫폼) 먼저 → PMS 나중**.
> 그래서 이 인증 표는 "지금 실행 계획"이 아니라 **템플릿 완성 후 꺼내 쓸 PMS 계획서**로 보존한다.
> 단, 표 안에서 **템플릿에 당장 필요한 인증 뼈대(6~9, 11)는 차용**해서 [[phase0-week-breakdown]] / [[phase1-week-breakdown]]로 빌려간다.

## 상태 범례
- **완료** — 학습 차원 완료 (제품화는 PMS 단계)
- **차용** — 템플릿 인증 뼈대로 Phase 0~1에 빌려옴
- **보류** — PMS 본체 작업 (템플릿엔 "꽂히는 구조"만, 실제 내용은 나중)

## 인증/권한 마스터 표

| 단계 | 주제 | 핵심 | 산출물 | 상태 / 위치 |
|---|---|---|---|---|
| 0 | Spring Security 기초 | Stateless/Cookie/Session, SecurityFilterChain | 동작 원리 이해 | 완료 |
| 1 | 세션 기반 로그인 | JSESSIONID, Cookie, CSRF | PMS v1 (세션) | 완료(학습) · 제품은 보류 |
| 2 | 세션의 한계와 수평 확장 | Sticky Session, Clustering, Redis Session, 수평 확장 | JWT 등장 이유 | 완료 |
| 3 | JWT 직접 구현 | Header/Payload/Signature, HMAC, 만료 | PMS v2 (JWT) | 완료(학습) · 제품은 보류 |
| 4 | 토큰 보안 (AT/RT·Rotation) | AT/RT, Rotation, Revocation, Blacklist, HttpOnly | 실무 토큰 관리 | **Phase 0** (이해 수준) |
| 5 | Spring Security 내부 구조 | UserDetailsService, Provider, OncePerRequestFilter, Method Security | 내부 구조 | **Phase 0** (이해 수준) |
| 6 | OAuth2 권한 위임 | Auth/Resource/Client, Code Flow, PKCE | 위임 이해 | **차용 → [[phase0-week-breakdown\|Phase 0 W1]]** |
| 7 | OIDC 인증 | ID Token, UserInfo, Google Login 분석 | 인증 구조 | **차용 → Phase 0 W1** |
| 8 | Keycloak 인증 서버 구축 | Docker+Keycloak+PostgreSQL, Realm/Client/User/Role | 인증 서버 | **차용 → Phase 0 W2** |
| 9 | Keycloak 연동 (Resource Server) | Resource Server, JWT Validation, Role Mapping | PMS v3 (Keycloak) | **차용 → Phase 0 W3** |
| 10 | React OIDC 연동 (프론트) | oidc-client-ts, Code Flow+PKCE | 프론트 연동 | 별도 프론트 트랙 (백엔드 템플릿과 분리) |
| 11 | Gateway 통합 인증 (SSO) | Spring Cloud Gateway, TokenRelay, Single Logout | 통합 인증 | **차용 → [[phase1-week-breakdown\|Phase 1 W5]]** |
| 12 | PMS 권한 모델(RBAC) | SYSTEM_ADMIN, PROJECT_ADMIN/PM/LEADER/DEVELOPER/VIEWER | 실무 권한 | 보류 · 템플릿엔 구조만([[phase1-week-breakdown\|Phase 1 W7]]) |
| 13 | 운영 관점 인증 | Refresh/Key Rotation, Forced Logout | 운영 수준 | 보류 → [[phase3-week-breakdown\|Phase 3]] 흡수 |
| 14 | MSA 통합 | — | — | 단일 단계 아님 = 전체 목표 그 자체 |

## 템플릿이 빌려가는 것 (= 지금 할 인증 작업)
템플릿 입장에서 인증은 **Keycloak에 Realm/Client/Role 세팅 + Gateway/서비스에서 JWT 검증**까지면 충분하다.
- 6·7 → Phase 0 W1 (OAuth2/OIDC 흐름)
- 8 → Phase 0 W2 (Keycloak 기동 + 토큰 발급)
- 9 → Phase 0 W3 (Resource server 검증)
- 11 → Phase 1 W5 (Gateway 인증 필터 + TokenRelay)
- 12 → Phase 1 W7 은 **역할 정의가 아니라 RBAC이 꽂히는 자리**만 (ADMIN/USER 정도)

## PMS 본체로 미루는 것 (템플릿 완성 후)
- PMS v1/v2 제품화 (세션/JWT 버전) — 학습은 끝났고 제품으로는 나중
- 12 RBAC 실제 역할 정의: PROJECT_ADMIN/PM/LEADER/DEVELOPER/VIEWER — PMS 도메인 권한
- 13 운영 관점: Refresh/Key Rotation, Forced Logout — PMS 운영 단계

## 무른 산출물 굳히기 (PMS 착수 때 적용)
"이해" 류 산출물은 PMS 만들 때 관측 가능한 DoD로 바꾼다. 예:
- 6 위임 이해 → "Keycloak으로 Auth Code+PKCE 받아 access token 재현"
- 7 인증 구조 → "ID token과 access token claim 차이를 디코딩해 확인"
- 4 실무 토큰 관리 → "AT 만료 후 RT로 갱신되는 흐름을 직접 호출로 확인"

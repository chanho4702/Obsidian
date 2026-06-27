---
tags: [인증, OAuth2, OIDC, SpringSecurity, JWT, 실습]
created: 2026-06-21
project: oauth-oidc-login
status: 완료 (Ready to merge)
---

# 인증 6+7단계 — OAuth2 & Google OIDC 로그인

> 스터디 커리큘럼 6단계(OAuth2 이해) + 7단계(OIDC 이해)를 하나의 동작하는 실습으로 통합.
> 기존 `jwt-security-module`(설정 기반 JWT 모듈)을 재사용하고 그 위에 Google OIDC 로그인을 얹음.

## 핵심 한 줄
OAuth는 "로그인 입구"만 담당하고, 성공 후엔 **우리 서비스의 JWT(AT 본문 + RT HttpOnly 쿠키)**로 전환된다. 이후 모든 API는 Bearer 검증·`/refresh`·`/logout`을 탄다.

---

## 6단계 — OAuth2 (인가, Authorization)

### 4역할
| 역할 | 이 시스템에서 |
|---|---|
| Resource Owner | 사용자(엔드유저) |
| Client | 우리 백엔드 `oauth-oidc-login` (client-secret 보유 → **confidential**) |
| Authorization Server | Google (accounts.google.com) |
| Resource Server | Google UserInfo (+ 로그인 후엔 우리 `/api/**`) |

### Authorization Code Flow + PKCE
```
1. Client → AuthServer: /authorize?response_type=code&client_id&redirect_uri
                        &scope&state&code_challenge&code_challenge_method=S256
2. 사용자 동의 (front-channel)
3. AuthServer → Client: redirect ?code=...&state=...   (front-channel, code만)
4. Client → AuthServer: POST /token  code + client_secret + code_verifier (back-channel)
5. AuthServer → Client: Access Token (+ OIDC면 ID Token)
```
- **front-channel** = 브라우저 경유(code만 노출), **back-channel** = 서버↔서버(토큰 교환). 토큰이 브라우저에 노출되지 않는 게 핵심.
- **PKCE**: `code_challenge = SHA256(code_verifier)`. AuthServer가 4번의 verifier와 1번의 challenge를 대조 → 탈취된 code 재사용 차단.
- **state**: CSRF 방어용 난수(인가요청↔콜백 상관관계). STATELESS라 인가요청을 **쿠키**(`OAUTH2_AUTH_REQUEST`)에 저장해 콜백에서 대조.

## 7단계 — OIDC (인증, Authentication)

| | OAuth2 | OIDC |
|---|---|---|
| 목적 | 인가 | 인증 |
| 결과물 | Access Token (불투명) | **ID Token(JWT)** + Access Token |
| 사용자 정보 | 비표준 | 표준 클레임 `iss/sub/aud/exp/email/name` + UserInfo |
| 트리거 | scope에 `openid` 없음 | scope에 **`openid`** 포함 |

- OIDC는 OAuth2 **위에 얹힌 인증 계층**. "로그인"은 본질적으로 인증이라 OIDC가 정답.
- scope에 `openid` → Spring Security `oauth2Login`이 자동으로 OIDC 모드(`OidcUserService`, principal=`OidcUser`)로 동작.
- **한 흐름에 JWT가 두 종류** (가장 중요한 포인트):
  - **Google ID Token** — `iss=https://accounts.google.com`. 입구에서 1회. "누구인가"를 Google이 서명으로 보증.
  - **우리 Access Token** — `iss=oauth-oidc-login`. 이후 모든 API. 우리 서비스의 세션 자격.

---

## 우리가 만든 전체 흐름
```
[myFront :5173]                 [oauth-oidc-login :8080]            [Google]
 Google 버튼 → /oauth2/authorization/google ───────────────────────▶ 동의
 /login/oauth2/code/google ◀── code 콜백 ◀──────────────────────────┘
   ① CustomOidcUserService: OidcUser 표준클레임 → DB find-or-create
   ② OAuth2LoginSuccessHandler: tokenService.issueFor → RT 쿠키 + 302
 /oauth/callback ◀──────────────────────────────────────────────────┘
   ③ POST /api/auth/refresh (RT 쿠키) → AT 획득 → /app
```

## 구조
**백엔드** `C:\myStudy\oauth-oidc-login` (Spring Boot 4 / Java 24 / MySQL Docker 3307)
- `oauth/` : CustomOidcUserService, OAuth2LoginSuccessHandler, CookieAuthorizationRequestRepository, OAuth2ClientConfig(PKCE resolver)
- `user/` : UserAccount(OIDC+LOCAL 공용), UserAccountRepository, DbUserDetailsService(폼로그인+시드+provisioning)
- `web/MeController` : GET /api/me (email/name/provider/sub/role)
- `security/` : 재사용한 JWT 모듈(복사). `JwtSecurityConfig.filterChain`에 `.oauth2Login(...)` 한 블록만 추가.

**프론트** `C:\MSA_TEMPLATE\myFront` (React 19 + MUI v9 + Vite)
- `lib/api.ts`(메모리 AT + 401 시 refresh 재시도 + credentials), `AuthContext`(토큰화+silent refresh), `OAuthCallbackPage`, `LoginPage`(폼 실연동+Google 버튼)

## 실행법
```powershell
# 백엔드
cd C:\myStudy\oauth-oidc-login
docker compose up -d                       # MySQL
$env:JAVA_HOME="C:\Program Files\Java\jdk-24"
$env:GOOGLE_CLIENT_ID="..."; $env:GOOGLE_CLIENT_SECRET="..."   # 같은 창에서!
.\gradlew.bat bootRun
# 프론트 (다른 창)
cd C:\MSA_TEMPLATE\myFront; npm run dev
```
- 데모 로컬 계정: `user@demo.com/password`, `admin@demo.com/admin`
- Google OAuth 클라이언트(웹): Google Cloud Console, 리디렉션 URI `http://localhost:8080/login/oauth2/code/google`
- (선택) `spring-dotenv` 추가해서 백엔드 `.env`로 creds 로딩 가능

---

## 🔧 트러블슈팅 (실제로 겪은 것)

### 1) Google 동의 화면 "401: invalid_client"
- **원인**: 환경변수가 bootRun 프로세스에 안 들어가, 앱이 `client_id`로 **문자 그대로 `${GOOGLE_CLIENT_ID}`**를 Google에 보냄.
- PowerShell `$env:...`는 **그 창에서, 설정 이후 실행된 프로세스**에만 적용됨.
- **해결**: 같은 창에서 `$env` 설정 → 그다음 `gradlew bootRun`. 확인: `GET /oauth2/authorization/google`의 302 Location에 실제 `client_id`가 실리는지.

### 2) 로그인 직후 "RT 재사용 탐지! family=... 폭파" → 로그인 실패
- **원인**: OIDC는 성공했으나, 콜백 직후 **`/api/auth/refresh`가 같은 RT로 동시 다발 호출**됨:
  `AuthProvider` 부트스트랩 refresh + `OAuthCallbackPage` refresh + React **StrictMode 이중 실행**.
  → RT 회전+재사용탐지가 동시 호출의 2번째를 "도난"으로 오인 → family 폭파.
- **해결**: `tryRefresh`에 **in-flight 중복 제거**(진행 중 요청이 있으면 같은 Promise 공유) → 동시 호출이 1요청으로 합쳐짐. 보안기능(rotation/reuse-detection)은 그대로 유지.
- 교훈: **RT 회전 + 재사용탐지** 환경에선 클라이언트가 refresh를 **단일화**해야 한다.

---

## 검증 결과 (E2E)
- 폼 로그인: login 200(AT/RT) → /api/me 200(provider=LOCAL) → refresh 200(회전) → 틀린비번 401 ✅
- OIDC 인가요청 302: scope=openid, state, nonce, **code_challenge+S256(PKCE)**, OAUTH2_AUTH_REQUEST 쿠키 ✅
- 실제 Google 동의 → /app 진입 ✅ (위 트러블슈팅 2건 해결 후)
- 최종 리뷰: **Ready to merge** (Critical/Important 없음)

## 후속 작업
- 프론트엔드 완성(회원가입·재사용 모듈화·UX 버그 3건): [[인증 6-7단계 프론트엔드 — 회원가입·모듈화·UX]]

## 참고
- 설계: `C:\myStudy\docs\superpowers\specs\2026-06-20-google-oidc-login-design.md`
- 계획: `C:\myStudy\docs\superpowers\plans\2026-06-20-oauth-oidc-login.md`
- 관련 개념 노트: [[OAuth2]] · [[OIDC]] · [[JWT]] · [[Spring Security]]

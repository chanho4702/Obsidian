# Keycloak 기반 자체-JWT 로그인 플랫폼 — 설계

- **작성일:** 2026-06-24
- **상태:** 승인됨 (구현 계획 작성 전)
- **홈:** `C:\MSA_TEMPLATE` (모노레포)

## 1. 목표와 배경

`myFront`(React 19 + MUI v9 SPA)를 **Keycloak 기반의 완전하고 재사용 가능한 로그인 플랫폼**에
연결한다. 현재 `myFront`의 auth 모듈은 커스텀 JWT 백엔드(`oauth-oidc-login`)에 묶여 있고,
별도 실습 프로젝트 `C:\myStudy\keycloak-sso`에는 Keycloak(26) + PKCE SPA + stateless 리소스
서버가 있다. 이 둘을 살려 하나의 플랫폼으로 통합한다.

### 요구사항
- 구글 OIDC 로그인 (Keycloak Identity Brokering 경유)
- SSO 지원 — 지금 증명 데모는 불필요하나, **나중에 다른 서비스를 SSO로 붙일 수 있는 토대**여야 함
- Keycloak을 인증/연합 계층으로 사용
- **플랫폼이 자체 JWT를 발급** (토큰 주권 — IdP에 종속되지 않음)
- 백엔드는 프론트와 분리된 별도 서비스. 나중에 미들 프록시(API Gateway)로 진화 가능
- 표준적이고 범용적인 방식

### 채택한 접근 (브레인스토밍 결론)
- **인증 모델 (B):** Keycloak = 인증/연합 계층, auth-server = 자체 JWT 발급 계층.
  Keycloak이 구글 브로커링·SSO·사용자 인증을 담당하고, auth-server는 confidential OIDC
  클라이언트로 로그인을 위임받아 **자체 RS256 JWT**를 발급한다.
- **자체 JWT 발급 (i):** Nimbus(JOSE) RS256 직접 발급 + `/.well-known/jwks.json` 노출.
  (Spring Authorization Server는 오버킬로 판단, 미래 업그레이드 여지로 남김)
- **미들 프록시 진화의 핵심 원칙:** 자체 JWT를 **표준 발급자(RS256 + JWKS + 표준 `iss`)** 로
  만든다. 그러면 미래의 모든 서비스/게이트웨이는 `issuer-uri`만 가리켜 검증한다.
  대칭키(HS256)였다면 비밀키를 전 서비스가 공유해야 해 분리가 불가능하다 — **RS256 + JWKS가
  분리를 가능케 하는 결정적 선택.**

## 2. 아키텍처

```
┌──────────────┐   1. 로그인 리다이렉트        ┌─────────────────────┐
│   myFront    │ ───────────────────────────► │   auth-server       │
│ (React SPA)  │                              │ (플랫폼 미니 IdP)    │
│ :5173        │ ◄─── 5. /app + RT쿠키 ─────── │ :9000               │
│ 자체JWT만 소지 │   6. /api/auth/refresh        │  • Keycloak OIDC     │
└──────────────┘   7. /api/me (Bearer 자체JWT) │    클라이언트(confid)│
                                               │  • 자체 JWT 발급      │
                                               │    (RS256 + JWKS)    │
                                               │  • refresh 회전/검증  │
                                               │  • JIT user provision│
                                               └─────────┬───────────┘
                                                         │ 2~4. OIDC code flow
                                                         ▼
                          ┌──────────────┐   authdb   ┌─────────────────────┐
                          │  PostgreSQL  │ ◄────────  │   Keycloak (26)     │
                          │  :5432(컨테이너)│  keycloak  │  :8080              │
                          └──────────────┘ ◄────────  │  • 구글 브로커링      │
                                                       │  • SSO 세션          │
                                                       │  • sso-demo realm   │
                                                       └─────────┬───────────┘
                                                                 ▼ Google OIDC
                                                            ┌─────────┐
                                                            │ Google  │
                                                            └─────────┘
```

### 컴포넌트 책임

| 컴포넌트 | 책임 | 의존 |
|---|---|---|
| **myFront** (:5173) | UI + 자체 JWT만 소지(AT 메모리/RT 쿠키). 로그인은 auth-server로 리다이렉트. | auth-server |
| **auth-server** (:9000, 신규) | ① Keycloak에 confidential OIDC 로그인 위임 ② 자체 RS256 JWT 발급 ③ `/jwks.json` 공개 ④ refresh 회전·재사용탐지 ⑤ JIT user 프로비저닝 ⑥ `/api/me` | Keycloak, PostgreSQL |
| **Keycloak** (:8080) | 구글 브로커링 + SSO 세션 + 사용자 인증. **유일한 인증 출처.** | Google |
| **PostgreSQL** | Keycloak DB(`keycloak`) + auth-server DB(`authdb`) | — |

> **범위 결정:** "완전한 로그인 플랫폼"에는 myFront + auth-server + Keycloak으로 충분하므로
> 별도 다운스트림 리소스 서버(app-a)와 SSO 증명용 두 번째 앱은 이번 범위에서 제외한다.
> auth-server 자체의 보호된 `/api/me`로 JWT 동작을 검증한다. SSO 확장은 추후 별도 요청.

### 디렉토리 레이아웃 (모노레포)
```
C:\MSA_TEMPLATE\
├─ myFront\              # 기존 SPA. auth 모듈 거의 유지, 로그인만 Keycloak 리다이렉트
├─ auth-server\          # 신규 Spring Boot 서비스
├─ infra\keycloak\       # keycloak-sso에서 포팅: docker-compose.yml + realm-export.json + .env.example
└─ docs\superpowers\specs\
```
`C:\myStudy\keycloak-sso`는 원본 보존, 거기서 Keycloak(docker+realm)을 포팅·적응시킨다.

## 3. 인증 플로우

### 3.1 로그인 (구글/Keycloak 공통)
```
1. myFront "로그인" 클릭 → window.location = {auth}/oauth2/authorization/keycloak
   ("Google로 로그인"은 ?kc_idp_hint=google 로 구글 직행)
2. auth-server(Spring OAuth2 Client) → Keycloak 로그인 화면으로 302
   └ "Sign in with Google" 노출. SSO 세션 있으면 화면 없이 통과.
3. 사용자 인증 → Keycloak이 콜백으로 code 반환 ({auth}/login/oauth2/code/keycloak)
4. auth-server: code 교환 → OIDC 사용자 확보 → AuthenticationSuccessHandler:
   • JIT: users 테이블에 keycloak_sub로 upsert (없으면 생성, 있으면 last_login/프로필 갱신)
   • 자체 access token(RS256, ~15분) + refresh token(opaque, ~14일) 발급
   • refresh token을 HttpOnly·Secure·SameSite 쿠키로 set
   • Keycloak id_token을 refresh 레코드에 보관(로그아웃용)
   • 302 redirect → myFront /app
5. myFront: AuthProvider 마운트 시 silent refresh → /api/auth/refresh → 자체 AT(메모리)
   → /api/me 로 사용자 표시
```

### 3.2 갱신 / API 호출 / 로그아웃
- **refresh** `POST /api/auth/refresh` (RT 쿠키): RT 해시 대조·검증 → 회전(옛 RT revoke,
  새 RT 쿠키 발급) + 새 AT 반환. 폐기된 RT가 다시 들어오면 재사용 = 도난 →
  같은 `family_id` 전체 revoke. (myFront `client.ts`의 in-flight dedup과 호환)
- **API 호출**: `Authorization: Bearer <자체AT>`. 401이면 client가 1회 refresh 후 재시도.
- **로그아웃** `POST /api/auth/logout`: 자체 RT revoke + 쿠키 삭제 + **Keycloak
  RP-initiated logout(end_session, id_token_hint)** 로 SSO 세션까지 종료.

### 3.3 auth-server 엔드포인트
| 메서드 | 경로 | 역할 |
|---|---|---|
| GET | `/oauth2/authorization/keycloak` | 로그인 시작 (Spring 기본) |
| GET | `/login/oauth2/code/keycloak` | OIDC 콜백 + 성공 핸들러 (Spring 기본 + 커스텀) |
| POST | `/api/auth/refresh` | RT 회전 + 새 AT |
| POST | `/api/auth/logout` | 로컬 + Keycloak 로그아웃 |
| GET | `/api/me` | 현재 사용자 (자체 AT 검증) |
| GET | `/.well-known/jwks.json` | 자체 JWT 공개키 (미래 서비스 검증용) |

## 4. 데이터 & 영속성

**DB:** docker-compose의 PostgreSQL에 별도 DB `authdb`. auth-server가 Spring Data JPA로 사용.
마이그레이션은 **Flyway**. (추후 서비스별 분리 시 컨테이너만 분리)

```
users                                  refresh_tokens
─────────────────────                  ──────────────────────────────
id            BIGINT PK   ◄──────────┐ id           UUID PK
keycloak_sub  VARCHAR UQ             └─user_id      BIGINT FK → users.id
email         VARCHAR                  token_hash   VARCHAR  (원문 저장 X, 해시만)
name          VARCHAR                  family_id    UUID     (회전 체인/재사용 탐지)
roles         VARCHAR[]               replaced_by  UUID     (회전된 다음 토큰)
created_at    TIMESTAMP               expires_at   TIMESTAMP
last_login_at TIMESTAMP               revoked      BOOLEAN
enabled       BOOLEAN                 kc_id_token  TEXT     (로그아웃 id_token_hint)
                                      created_at   TIMESTAMP
```

- **users**: Keycloak 첫 로그인 시 JIT 생성, 이후 로그인마다 동기화. 플랫폼의 사용자 권위 —
  미래 서비스의 도메인 테이블이 `users.id`를 FK로 참조.
- **refresh_tokens**: 회전 + 재사용 탐지. 토큰 원문은 해시로만 저장.

## 5. 토큰 / 키 설계

- **Access Token**: 자체 RS256 JWT, ~15분. claim: `iss`(=auth-server URL), `sub`(=users.id),
  `email`, `name`, `roles`, `iat`, `exp`. 프론트 메모리 보관.
- **Refresh Token**: opaque 랜덤(고엔트로피), ~14일, HttpOnly·Secure·SameSite=Lax 쿠키.
  DB에 해시 저장.
- **서명 키**: RSA 키페어. dev는 리소스/볼륨에 보관(운영은 외부 시크릿). `/.well-known/jwks.json`
  으로 공개키 노출 → 미래 서비스가 `issuer-uri`/`jwk-set-uri`만으로 검증.
- auth-server는 자기 토큰의 **발급자이자 검증자** — `/api/me`는 oauth2-resource-server로
  자체 공개키 JwtDecoder를 써서 검증.

## 6. 프론트엔드 변경 (myFront)

- `.env`: `VITE_API_BASE=http://localhost:9000`.
- `src/auth/client.ts`: `tryRefresh`/`fetchMe`/`apiFetch`/`logout` **그대로 재사용**.
  `login(email,password)`·`signup` 제거. `loginUrl()`(=`/oauth2/authorization/keycloak`),
  `googleLoginUrl()`(=`/oauth2/authorization/keycloak?kc_idp_hint=google`) 추가.
- `src/auth/AuthContext.tsx`: `loginWithPassword`/`signUp` 제거. `user`/`logout`/silent
  restore 유지.
- `src/app/pages/LoginPage.tsx`: 비번 폼 제거 → "로그인" + "Google로 로그인" 버튼(리다이렉트).
- `SignUpPage`: 자체 가입 폼 제거(회원가입은 Keycloak 담당). 라우트 정리.
- **콜백 페이지 불필요** — 백엔드가 `/app` 리다이렉트 후 silent refresh로 복원(기존 구조 동일).

## 7. auth-server 스택

Spring Boot 4 / Java 24 / Gradle (keycloak-sso와 통일). 스타터:
`oauth2-client`(Keycloak 로그인) + `security` + `web` + `oauth2-resource-server`(자체 AT 검증)
+ `data-jpa` + `postgresql` + `flyway`. JWT 발급은 Nimbus(RS256, Spring Security 추이 포함).
CORS: `http://localhost:5173` + credentials 허용.

## 8. Keycloak 설정

- `sso-demo` realm 유지. 공개 SPA 클라이언트 대신 **confidential 클라이언트 `platform-bff`**:
  client secret, standard flow, redirect URI `http://localhost:9000/login/oauth2/code/keycloak`,
  realm roles를 토큰 claim에 매핑.
- 데모 사용자 `alice`/`admin` + USER/ADMIN 롤 유지. realm 회원가입 활성화(선택).
- **구글 브로커링 = 수동 1회** (자동화 불가, 구글 자격증명 필요):
  Google Cloud OAuth 클라이언트 발급 → redirect URI
  `http://localhost:8080/realms/sso-demo/broker/google/endpoint` → Keycloak Identity
  providers > Google에 client id/secret 입력. 값은 `infra/keycloak/.env`.

## 9. 에러 처리

- refresh 실패 → 401 → 프론트 세션 클리어 → `/login`.
- refresh 재사용 탐지 → `family_id` 전체 revoke → 401.
- Keycloak 다운 → 로그인 리다이렉트 실패를 사용자에게 안내.
- 자체 AT 만료/위조 → `/api/me` 401 → client 1회 refresh 재시도 후 실패 시 로그아웃.

## 10. 테스트 전략

- **auth-server (JUnit + Spring Boot Test):**
  - 자체 JWT claim/RS256 서명이 JWKS로 검증 가능한가
  - refresh 회전: 옛 RT 무효화 + 새 RT 발급
  - 재사용 탐지: 폐기된 RT 재사용 시 family 전체 revoke
  - JIT 프로비저닝: 첫 로그인 user 1건 생성, 재로그인 시 중복 생성 없이 갱신
  - `/api/me`: 유효 토큰 200, 무효/없음 401
- **myFront:** 타입 게이트(`tsc -b`)가 1차 검증. 별도 러너 없음.
- **수동 E2E:**
  1. Keycloak 로그인 → `/app` 사용자 표시
  2. 구글 로그인 → federated 사용자 진입
  3. AT 만료 후 자동 refresh로 세션 유지
  4. 로그아웃 → 로컬 + Keycloak 세션 종료(재로그인 시 Keycloak 화면 다시 노출)
  5. 토큰 없이 `/api/me` → 401

## 11. 미래 확장 (이번 범위 아님)

- **미들 프록시(API Gateway):** Spring Cloud Gateway를 앞단에 추가 — 자체 JWT를 엣지에서
  검증·라우팅. auth-server는 그대로 분리 유지. 다운스트림 서비스는 `issuer-uri`만 신뢰.
- **SSO로 다른 서비스 붙이기:** 새 리소스 서버가 auth-server `issuer-uri`/JWKS로 자체 JWT
  검증. Keycloak 세션 공유로 재로그인 불필요.
- **하드닝:** refresh 시 Keycloak RT로 재인증(Keycloak이 토큰 권위 유지), 키 외부 시크릿화,
  Spring Authorization Server로 발급 계층 격상.

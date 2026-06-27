---
작성일: 2026-06-25
태그: [인증, keycloak, oidc, jwt, spring-boot, react]
상태: 코드 완료(로컬 커밋) · 수동 E2E 검증 대기
---

# 인증 8단계 — Keycloak BFF 자체-JWT 플랫폼 (구현 기록)

> [[인증 6-7단계 OAuth2-OIDC 로그인]] / [[인증 6-7단계 프론트엔드 — 회원가입·모듈화·UX]]의 연장.
> [[인증 로드맵]]에서 "Keycloak에 OIDC를 위임하고 우리는 자체 JWT를 발급한다" 모델을 실제로 구현한 단계.

## 무엇을 만들었나

myFront(React SPA)는 **자체 발급 JWT만** 소지한다(Access Token = 메모리, Refresh Token = HttpOnly 쿠키). 신규 **auth-server**(Spring Boot 4, :9000)가 Keycloak(:8080, realm `sso-demo`, confidential client `platform-bff`)에 OIDC 로그인을 위임받고, 성공 시 ① JIT로 사용자 프로비저닝 ② **RS256 자체 JWT** 발급 ③ 회전형 refresh token 쿠키 설정 후 프론트 `/app`으로 리다이렉트한다. `/.well-known/jwks.json`로 공개키를 노출해 미래 서비스가 `issuer-uri`만으로 검증 가능. refresh token은 회전 + **재사용 탐지**(폐기된 superseded 토큰 재replay 시 패밀리 전체 폐기). 사용자·refresh token은 PostgreSQL `authdb`에 영속, RT는 **SHA-256 해시만** 저장한다.

핵심 흐름:
- 로그인: 프론트 버튼 → `:9000/oauth2/authorization/keycloak` → Keycloak 화면 → 성공핸들러가 RT 쿠키 + `/app` 리다이렉트.
- 세션 복원(silent): 프론트 마운트 시 `POST /api/auth/refresh`(RT 쿠키) → `{accessToken}` 수령 → `/api/me`로 사용자 정보.
- 로그아웃: `POST /api/auth/logout` → 패밀리 폐기 + Keycloak `end_session` URL 반환 → 프론트가 전체 페이지 이동(SSO 세션 종료).

## 결과 (이번 세션, 로컬 커밋만 · push 보류)

| repo | 위치 / 브랜치 | 이번 세션 범위 | 빌드 게이트 |
|---|---|---|---|
| auth-server | `C:/MSA_TEMPLATE/auth-server` · `main` | Task 5~7 + Flyway fix (`d9b14e9`..`a6f203e`) | `./gradlew test` **14/14 green** (JDK 24) + 통합 E2E 통과 |
| myFront | `C:/MSA_TEMPLATE/myFront` · `feat/keycloak-oidc-login` | Task 9~10 (`6fcab18`..`8448f28`, 1커밋) | `npm run build`(`tsc -b && vite build`) **green** |
| MSA_TEMPLATE(우산) | `C:/MSA_TEMPLATE` · `master` | GOOGLE_SETUP.md (`53963a6`) + plan/ledger | — |

**구현 플랜:** `docs/superpowers/plans/2026-06-24-keycloak-bff-jwt-platform.md` (Task 1~11). 진행 원장: `.superpowers/sdd/progress.md`.

### 세션 중 발견·수정한 것 (플랜에 없던)
- **contextLoads 잠복 실패**: 전체 `@SpringBootTest`가 base `application.yml`의 `issuer-uri`로 Keycloak discovery(네트워크)를 시도하다 실패. Task 3~5가 포커스 테스트만 돌려 안 잡힘. → 테스트에 오프라인 `ClientRegistrationRepository` 빈(`TestOAuth2ClientConfig`, src/test)을 제공해 자동설정을 back off → discovery 미수행.
- **Spring Boot 4 적응 2건**: `@MockBean` 제거됨 → `@MockitoBean`(`org.springframework.test.context.bean.override.mockito`). `@DataJpaTest`도 새 패키지(이전 단계 반영).
- **최종 리뷰 fix**: logout URL의 `id_token_hint`도 URL 인코딩(`post_logout_redirect_uri`와 대칭).
- **Flyway 자동설정 누락(런타임 부팅 불가)**: `flyway-core`만으론 Boot 4에서 Flyway가 자동 실행 안 됨 → authdb 스키마 미생성 → `ddl-auto=validate`가 `missing table refresh_tokens`로 부팅 실패. `org.springframework.boot:spring-boot-flyway` 모듈 추가로 해결(`a6f203e`). 테스트는 H2+flyway off라 안 잡혔고, **통합검증(docker+bootRun)에서 처음 드러남** — Task 8의 존재 이유.

### 자동 통합검증 결과 (2026-06-25, curl 기반 — 브라우저 없이)
Docker(Keycloak+Postgres) + auth-server bootRun 띄우고 alice/alice 전체 OIDC 흐름을 curl로 검증:
- ✅ Keycloak discovery 200, authdb 존재, Flyway v1 적용 → `users`/`refresh_tokens` 생성
- ✅ JWKS = 공개키만(`kty/n/e`, private `d` 없음)
- ✅ `/api/me`·`/api/auth/refresh` 미인증 → 401
- ✅ alice/alice 로그인 → `:5173/app` 리다이렉트 + `refresh_token` HttpOnly 쿠키
- ✅ `/api/auth/refresh` → RS256 accessToken, `/api/me` → `{sub:"1", email:"alice@demo.com", name:"Alice Kim", role:"USER", provider:"KEYCLOAK"}`
- ✅ JIT user 영속(id=1), refresh_tokens 회전(2행 중 1 revoked), token_hash 길이 43(SHA-256 해시만)
- ✅ logout → Keycloak `end_session` URL + id_token_hint 반환
→ **백엔드 통합은 완전 검증됨.** 남은 건 브라우저 프론트 클릭 E2E와 구글 설정뿐.

### 최종 whole-branch 리뷰 (opus): **READY WITH FIXES**
교차 repo 와이어 계약(`{accessToken}` / `{keycloakLogoutUrl}` / 쿠키명·경로 / OIDC 시작 URL / `/api/me` 형태 / AT claim) **전부 일치 확인**. 회전·재사용탐지 상태머신, SHA-256-only 규칙, 2-체인 보안 포스처 모두 정상. 머지 전 필수 수정은 id_token_hint 인코딩 1건뿐이었고 **반영 완료**.

---

## ⚠️ 수동 검증 체크리스트 (자동화 불가 — 직접 실행)

자동화는 코드+빌드+문서까지였다. 아래는 **러닝 인프라 + 브라우저**가 필요해 사람이 직접 해야 한다.

### A. 백엔드 통합 검증 (Task 8) — ✅ 2026-06-25 curl로 자동 검증 완료 (아래는 재현용)
- [ ] `cd infra/keycloak; docker compose up -d` → `docker compose ps`에 postgres·keycloak 둘 다 Up
- [ ] `http://localhost:8080/realms/sso-demo/.well-known/openid-configuration` 200 + JSON
- [ ] `docker exec platform-postgres psql -U keycloak -lqt`에 `authdb`·`keycloak` 둘 다 존재
- [ ] auth-server 기동: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat bootRun` → 로그 `Started AuthServerApplication`, :9000
- [ ] `http://localhost:9000/.well-known/jwks.json` → `{"keys":[{"kty":"RSA",...}]}` (공개키만)
- [ ] `http://localhost:9000/oauth2/authorization/keycloak` → Keycloak 로그인 → `alice`/`alice` → `/app`으로 리다이렉트(프론트 미기동이면 연결실패 화면이어도 URL이 `:5173/app`이면 OK). DevTools > Application > Cookies(:9000)에 `refresh_token` HttpOnly 쿠키 존재
- [ ] DevTools 콘솔에서 refresh→me:
  ```js
  const t = (await fetch('http://localhost:9000/api/auth/refresh',{method:'POST',credentials:'include'}).then(r=>r.json())).accessToken;
  await fetch('http://localhost:9000/api/me',{headers:{Authorization:'Bearer '+t}}).then(r=>r.json())
  ```
  → `{ sub, email:"alice@demo.com", name, provider:"KEYCLOAK", role:"USER" }`

### B. 전체 E2E (Task 11)
- [ ] 3 프로세스 기동: ① `infra/keycloak` docker compose up -d ② `auth-server` bootRun ③ `cd myFront; npm install; npm run dev`
- [ ] 로컬계정 로그인: `http://localhost:5173/login` → "로그인" → Keycloak → `alice`/`alice` → `/app` 대시보드(이메일 alice@demo.com), 새로고침해도 세션 유지(silent refresh)
- [ ] 로그아웃: 대시보드에서 로그아웃 → Keycloak end_session 경유 → `/login` 복귀 → 다시 "로그인" 시 Keycloak 화면 재노출(SSO 세션 종료)
- [ ] 보호 API 음성테스트: 로그아웃 상태에서 `await fetch('http://localhost:9000/api/me',{credentials:'include'}).then(r=>r.status)` → **401**

### C. 구글 로그인 설정 (선택, 본인 Google Cloud 자격증명 필요)
가이드: `infra/keycloak/GOOGLE_SETUP.md`
- [ ] Google Cloud Console > Credentials > OAuth 2.0 Client(Web) 발급. 승인 리디렉션 URI: `http://localhost:8080/realms/sso-demo/broker/google/endpoint`
- [ ] Client ID/Secret을 `infra/keycloak/.env`에 저장(`.env.example` 참고)
- [ ] Keycloak 콘솔(admin/admin) > sso-demo > Identity providers > Add Google > Client ID/Secret 입력 > Save
- [ ] `:5173/login` → "Google 계정으로 로그인" → 구글 인증 → `/app`, `/api/me`의 `provider="GOOGLE"`

---

## Follow-up (최종 리뷰에서 accept된 폴리시 — 급하지 않음)
- 문서 드리프트: `myFront/src/auth/types.ts`·`authClient.ts` 주석의 예시 baseUrl이 아직 `:8080`; 프로젝트 `CLAUDE.md`가 아직 `useAuth()`에 `loginWithPassword/signUp/completeOAuthLogin`·`/register` SignUpPage를 설명(이제 부정확); `src/auth/index.ts`의 배럴-only-import 주석 소실.
- 테스트 보강: `CookieFactory` 단위테스트 없음; `AuthControllerTest.logout`이 쿠키 삭제(Max-Age=0) 미검증; `MeControllerTest`가 `name`/`provider` claim 미검증.
- 스타일: `SecurityConfig`가 CORS origin을 `@Value` 대신 `Environment`로 읽음(다른 클래스는 `@Value`).
- 운영 주의: `MeController`의 `role`은 `roles[0]`(첫 롤만). 단일-롤 UI엔 OK지만 멀티롤 시 첫 롤만 노출됨.

## 남은 과업 (Backlog) — "MSA 로그인 완료"까지

현재 상태: **인증 토대 + 로그인 플로우는 구현·백엔드 검증 완료.** 아래가 끝나야 "MSA 환경 로그인 완성"이라 부를 수 있음.

- [ ] **T1 — 브라우저 E2E 최종 확인 (사람)**: `http://localhost:5173/login`에서 ① 로컬계정 `alice`/`alice` ② **구글 계정** 로그인 → `/app` 진입 · 새로고침 세션유지(silent refresh) · 로그아웃 후 재로그인. **완료기준**: 두 경로 다 브라우저에서 성공. (구글 IdP·버튼은 이미 붙음. 실제 구글 왕복만 미확인.)
- [ ] **T2 — JWT 검증 샘플 마이크로서비스 1개 (MSA 핵심 증명)**: `issuer-uri=http://localhost:9000` + JWKS만으로 auth-server 자체 JWT를 검증하는 리소스 서버를 하나 붙인다. **완료기준**: auth-server가 발급한 AT로 그 서비스 보호 API가 200, 토큰 없으면 401 → "한 번 로그인 → 다른 서비스도 그 토큰으로 인가"가 도는 걸 증명. **이게 돼야 진짜 'MSA 로그인'.**
- [ ] **T3 — (선택) API 게이트웨이**: 게이트웨이단에서 JWT 검증 후 라우팅.
- [ ] **T4 — 운영 하드닝**: HTTPS, 쿠키 `Secure=true`, 시크릿 외부주입(`admin`·`platform-bff-secret`·JWK), JWK 키 회전 전략, CORS 좁히기.
- [ ] **T5 — push 결정**: auth-server `main` / myFront `feat/keycloak-oidc-login` 원격 push (현재 전부 로컬 커밋만).

기타 비차단 follow-up은 위 "follow-up" 항목 참고(문서 드리프트·테스트 보강·SecurityConfig @Value화 등).

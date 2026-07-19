---
tags: [msa, template, keycloak, oidc, split-horizon, userinfo, backchannel-logout, kc-hostname, 구현기록, 사건기록]
작성일: 2026-07-19
상태: 해결 완료 (auth-server 5fe8cae · infra-settings 8e0cc81) · E2E 검증 완료
---

# 18 — 배포 로그인·로그아웃 사건 — KC iss-호스트 대조와 split-horizon의 한계 (2026-07-19)

상위: [[00 개요 — 전체 구조]] · 선행: [[12 크로스앱 내비게이션 + redirect_uri 사건 + 운영 전환 로드맵 (2026-07-15)]], [[14 전체 구현 심화 해설 — 배경지식으로 읽는 현재 아키텍처 (2026-07-17)]]

## 무엇을 했나

배포(컨테이너) 환경에서 alm/wiki 진입 시 **모든 OIDC 로그인이 `/login?error`로 떨어지는 사건**과, 이어서 발견된 **로그아웃 후 KC SSO 세션이 살아남는 사건**을 각각 근본 원인까지 파서 해결했다. 두 사건의 뿌리는 하나 — **Keycloak(동적 hostname)은 토큰의 iss와 "요청이 들어온 호스트"를 대조한다**는 것. 14에서 만든 split-horizon 수동 ClientRegistration이 이 대조에 걸리는 두 지점(userinfo, 백채널 로그아웃)을 그동안 몰랐던 것이다.

## 0. 증상과 오해

- 증상: `http://localhost/alm/`(또는 `/wiki/`) 진입 → SPA가 `/api/auth/refresh` 401 → `/oauth2/authorization/keycloak` → KC 로그인 → 콜백 → **`/login?error`**.
- 처음엔 "alm/wiki만 안 된다"로 보였지만, 실제로는 **배포 환경의 신규 로그인 전부가 깨진 상태**였다. portal이 멀쩡해 보인 건 이미 로그인된 세션(자체 RT 쿠키)이 남아 있었기 때문.
- 12(07-15)의 "SSO 완료" 판정은 redirect_uri 거부 해소(KC 로그인 폼 200)까지였고, **콜백 이후 전 구간은 그때 검증되지 않았다**. 이번 userinfo 401은 구성상 그때도 반드시 실패했을 동작.

## 1. 사건 ① — 로그인: userinfo 401 → invalid_user_info_response

### 증거 수집 (systematic-debugging)

nginx 액세스 로그로 실패 지점을 좁혔다:

```
GET /alm/                              200
POST /api/auth/refresh                 401   ← 미로그인, 정상
GET /oauth2/authorization/keycloak     302   ← 인가 요청 정상
GET /login/oauth2/code/keycloak?...    302   ← 콜백 도달, 그런데
GET /login?error                       200   ← 즉시 실패
```

auth-server는 아무것도 로깅하지 않았다(OAuth2 로그인 실패는 DEBUG 레벨). actuator loggers는 인증에 막혀 있어, **compose override로 `LOGGING_LEVEL_ORG_SPRINGFRAMEWORK_SECURITY=DEBUG`를 주입해 재기동** 후 curl로 전체 플로우(realm의 alice 테스트 유저)를 재현했다:

```
토큰 교환 POST keycloak:8080/...(token)     → 200 ✓
JWKS GET keycloak:8080/...(certs)           → 200 ✓
userinfo GET keycloak:8080/...(userinfo)    → 401 ✗
→ OidcAuthorizationCodeAuthenticationProvider: [invalid_user_info_response]
→ /login?error
```

### 근본 원인 — userinfo는 백채널로 부를 수 없는 엔드포인트였다

KC는 userinfo 요청의 **Host를 access token의 iss와 대조**한다. 같은 토큰(iss=`http://localhost:8080/realms/sso-demo`)으로 직접 실측:

| 호출 경로 | 결과 |
|---|---|
| `localhost:8080` 경유 (iss와 일치) | **200** |
| `keycloak:8080` 경유 (컨테이너 내부 DNS) | **401** |

즉 14의 split-horizon 계약("서버-서버 = keycloak:8080")에서 token/jwks는 맞지만 **userinfo만은 예외**였다. 토큰 검증이 없는 jwks, code+secret로 인증하는 token과 달리, userinfo는 **Bearer 토큰 자체를 검증**하기 때문에 iss-호스트 대조에 걸린다.

### 수정 — userInfoUri 제거 (auth-server `5fe8cae`)

`ContainerClientRegistrationConfig`에서 `.userInfoUri(...)` 한 줄 제거. userInfoUri가 없으면 Spring `OidcUserService`는 userinfo 호출을 **건너뛰고 ID 토큰 클레임으로 principal을 구성**한다. ID 토큰에 `name`/`email`/`preferred_username`이 이미 다 실려 있음을 디코드로 확인했다(KC 기본 클레임 매퍼는 includeInIdToken). 계약 테스트를 RED→GREEN으로 갱신(`userInfoAbsentSoPrincipalComesFromIdToken`) 후 CI→GHCR→dispatch→로컬 runner 파이프라인([[13 백엔드 컨테이너화 + compose 통합 — CI-CD Wave 1 (2026-07-15)]])으로 배포.

검증: alice 로그인 → 콜백 `302 /app` + RT 쿠키 → `/api/auth/refresh` 200 → `/api/me` Bearer 200 (`name: Alice Kim, email: alice@demo.com` — ID 토큰만으로 온전).

## 2. 사건 ② — 로그아웃: 백채널 end_session invalid_grant

로그인을 고치자 곧바로 "로그아웃이 정상적이지 않다"가 드러났다. 재현 결과:

```
POST /api/auth/logout → 200 (자체 RT 폐기 + 쿠키 삭제는 정상)
auth-server WARN: KC 백채널 로그아웃 실패: 400 invalid_grant
  "Invalid token issuer. Expected 'http://keycloak:8080/realms/sso-demo'"
로그아웃 직후 재인가 요청 → 로그인 페이지 없이 code 즉시 발급 = KC SSO 세션 잔존
```

**같은 뿌리다.** KC refresh_token의 iss는 `localhost:8080`인데 end_session 호출이 `keycloak:8080`으로 가니, KC는 "이 호스트에서 발급된 토큰이 아니다"라며 거부한다. `KeycloakLogoutClient`가 best-effort 설계(실패해도 예외 안 던짐 — [[01 인증 플랫폼 (Keycloak BFF + 자체 JWT)]]의 백채널 로그아웃)라 로컬 세션만 지워지고 200이 반환됐고, 사용자 눈에는 "로그아웃했는데 다시 들어가면 그대로 로그인됨"으로 보였다.

### 수정 — KC_HOSTNAME 고정 (infra-settings `8e0cc81`)

이번엔 사건 ①처럼 "호출 제거"로 못 푼다 — **로그아웃은 토큰을 KC에 제출해야만 성립**한다. 그래서 KC 쪽을 고쳤다:

```yaml
# infra/keycloak/docker-compose.yml — keycloak 서비스
KC_HOSTNAME: http://localhost:8080
```

hostname을 고정하면 KC의 기대 issuer가 **요청 호스트와 무관하게 항상 `localhost:8080`** → 프론트채널(브라우저)이든 백채널(keycloak:8080)이든 일관된다. auth-server 코드는 무변경, KC 컨테이너만 재생성(realm은 PG 볼륨에 영속이라 무손실, 온라인 세션만 초기화).

검증 E2E: 로그인 ✓ → refresh 200 ✓ → 로그아웃 200 + WARN 소멸 ✓ → **재인가 시 로그인 페이지 복귀(HTTP 200) = SSO 세션 실제 종료** ✓.

## 3. split-horizon 계약 최종본

14의 계약이 이렇게 갱신됐다:

| 항목 | 값 | 비고 |
|---|---|---|
| authorization-uri (브라우저) | `localhost:8080` | 변화 없음 |
| token / jwks (서버-서버) | `keycloak:8080` | 변화 없음 |
| **userinfo** | **없음** | ID 토큰 클레임으로 principal — 백채널 호출은 구조적으로 401 |
| 백채널 로그아웃 | `keycloak:8080` | KC_HOSTNAME 고정으로 성립 |
| iss 검증 | `localhost:8080` | KC_HOSTNAME과 일치 |
| KC hostname | **고정** (`http://localhost:8080`) | 동적 → 고정 전환이 이번 사건의 최종 해소 |

> [!note] 왜 여전히 수동 ClientRegistration인가
> KC_HOSTNAME을 고정했어도 Boot 디스커버리로는 못 돌아간다 — 프로퍼티 `issuer-uri`(=백채널 로그아웃 URL 주입용)가 `keycloak:8080`이라, 디스커버리하면 metadata issuer(`localhost:8080`)와 불일치로 기동 실패. 14의 결론 유지.

> [!tip] 교훈 3개
> ① **"동작한다"의 검증 범위를 명시하라** — 12의 "SSO 완료"는 로그인 폼 진입까지였다. 콜백 이후가 미검증이면 완료가 아니다. E2E 체크리스트([[11 nginx 통합배포 + 3앱 SSO (2026-07-14)]] 부속)에 "콜백→/app→refresh→/api/me" 구간을 넣었어야 했다.
> ② **KC의 iss-호스트 대조는 "토큰을 제출하는 모든 엔드포인트"에 걸린다** — userinfo, end_session, introspection, revocation. split-horizon을 쓴다면 이 부류는 전부 의심 대상.
> ③ **best-effort 실패 삼킴은 증상을 지연시킨다** — 로그아웃이 200을 주니 겉보기 정상. WARN 로그가 유일한 단서였다. best-effort로 삼키는 실패는 메트릭/알림으로 승격할 가치가 있다.

## 4. 진단 기법 메모 (재사용 가치)

- **compose override로 일시 DEBUG**: `-f docker-compose.yml -f override.yml up -d <svc>` — 이미지 재빌드 없이 로깅 레벨만 주입, 원복은 override 없이 재-up. 스크래치 파일이라 repo 오염 없음.
- **curl 전체 OIDC 플로우 재현**: keycloakify(React) 테마는 form action이 HTML에 없다 — `kcContext`의 `"loginAction"` JSON 값을 파싱해야 한다. cookie jar 하나로 KC(:8080)와 nginx(:80) 쿠키가 같이 관리된다(둘 다 host `localhost`, path 분리).
- **가설 검증은 "같은 토큰, 두 호스트"**: 401의 원인이 토큰인지 호스트인지는 동일 토큰을 두 경로로 쏘면 한 방에 갈린다.
- **password grant 함정**: `directAccessGrantsEnabled: false`면 password grant 응답이 에러 JSON인데, sed 파싱은 조용히 쓰레기를 뽑는다 — `TOKEN_LEN` 같은 sanity 출력이 방어선.

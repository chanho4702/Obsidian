# nginx 통합 프론트 배포 + 3앱 SSO — 설계

날짜: 2026-07-14
상태: 승인됨 (브레인스토밍 Q&A로 갈림길 5개 전부 사용자 확정)

## 배경 / 문제의식

프론트가 3개가 됐다 — myFront(포털, auth 연동 완료), wiki-front(컨플루언스 클론 MVP),
alm-front(지라 클론 MVP). 각자 vite dev 서버(:5173 등)로만 뜨고, wiki/alm은 인증이 아예 없다
(localStorage 목 데이터). 이들을 nginx 하나로 통합 서빙하고, 기존 인증 플랫폼
(Keycloak + auth-server BFF 자체-JWT)에 물려 **한 번 로그인하면 3앱 전부 인증되는 SSO**를 만든다.

## 확정된 결정 (브레인스토밍 Q&A)

| 갈림길 | 결정 | 근거 |
|---|---|---|
| URL 구조 | **경로 기반 단일 오리진** (`/`, `/wiki/`, `/alm/`) | 오리진이 하나면 RT 쿠키(host-only, path=/api/auth)가 자동 공유 — SSO가 공짜. 서브도메인은 쿠키 Domain 수정+hosts+CORS 추가 작업 |
| wiki/alm 인증 수준 | **로그인 게이트** (미로그인 시 Keycloak으로 리다이렉트) | 진짜 SSO 체험 완성. 데이터는 localStorage 목 유지 |
| nginx 실행 | **docker compose에 서비스 추가** + dist 볼륨 마운트 | 기존 인프라(KC/PG/Redis)와 한 방 기동. 로컬 build만 다시 하면 재기동 없이 반영 |
| 토폴로지 | **nginx가 맨 앞** (정적 직접 서빙, API만 gateway로 프록시) | 정적 트래픽이 webflux를 안 거침. X-Forwarded 헤더로 OIDC redirect_uri 자동 구성 |
| 로그인 후 복귀 | **returnTo 쿠키** (`post_login_redirect`) | /wiki에서 로그인하면 /wiki로 복귀. SAML RelayState/NextAuth callbackUrl과 같은 표준 패턴 |
| auth 모듈 공유 | **앱별 복사** (myFront src/auth/ 패턴) | 파일 3~4개로 작고 세 앱이 별도 repo. 중복이 아프면 그때 패키지화 |

## 목표

1. `http://localhost`(nginx, 기본 80, `NGINX_PORT`로 조정)에서 3개 SPA + API 프록시 단일 오리진 제공
2. wiki/alm에 로그인 게이트 — myFront에서 로그인했으면 무프롬프트 통과 (SSO)
3. 로그인 후 원래 보던 경로로 복귀 (returnTo 쿠키)
4. 백채널 로그아웃 시 3앱 전부 로그아웃 (기존 계약 그대로 — 쿠키 공유라 자동)

## 비목표 (YAGNI)

- wiki/alm 데이터의 백엔드 연동 — localStorage 목 유지, 이번엔 인증만
- HTTPS/운영 배포, 프론트 Docker 멀티스테이지 빌드 — dist 볼륨 마운트로 충분
- RT 동시-refresh 유예기간 하드닝 — 리스크 섹션에 기록, 다음 하드닝 웨이브로
- @chanho/auth 패키지화 — 앱별 복사로 시작

## 토폴로지

```
브라우저 ── http://localhost (nginx, docker)
              ├─ /            → myFront/dist   (정적)
              ├─ /wiki/       → wiki-front/dist
              ├─ /alm/        → alm-front/dist
              └─ /api/**, /oauth2/**, /login/**, /.well-known/**
                              → host.docker.internal:8000 (gateway)
                                  ├─ auth-server (lb://, :9000)
                                  └─ board-service (lb://)
Keycloak :8080 (기존 그대로 — 브라우저가 직접 접근)
```

핵심 계약: RT 쿠키가 host-only(`localhost`) + `path=/api/auth`이므로 **단일 오리진이면
3앱이 쿠키를 자동 공유한다. CookieFactory는 무수정.** 같은 오리진이라 브라우저 요청에
CORS 자체가 발생하지 않고, 기존 dev 오리진(:5173) CORS 설정은 그대로 살아 있다.

## 설계

### 1. nginx (`infra/nginx/` 신설 + compose 서비스 추가)

`infra/nginx/default.conf`:
- `/` → `root /usr/share/nginx/html` + `try_files $uri /index.html` (SPA 폴백)
- `/wiki/` → 해당 dist alias + `try_files $uri /wiki/index.html`
- `/alm/` → 동일 (`/alm/index.html`)
- `/api/`, `/oauth2/`, `/login/`, `/.well-known/` → `proxy_pass http://host.docker.internal:8000`
  - `proxy_set_header X-Forwarded-Host $host:$server_port` / `X-Forwarded-Proto` / `X-Forwarded-For`
  - auth-server는 이미 `forward-headers-strategy=framework` — OIDC `redirect_uri`가
    `http://localhost/login/oauth2/code/keycloak`으로 자동 구성됨

`infra/keycloak/docker-compose.yml`에 nginx 서비스 추가:
- image: `nginx:alpine`, ports: `${NGINX_PORT:-80}:80`
- volumes: `../nginx/default.conf`(ro), `../../myFront/dist` → html,
  `../../wiki-front/dist` → html/wiki, `../../alm-front/dist` → html/alm (모두 ro)
- `extra_hosts: host.docker.internal:host-gateway` (Windows Docker Desktop은 기본 제공이나 명시)

### 2. 프론트 3앱

| 앱 | 변경 |
|---|---|
| myFront | base `/` 유지. 로그인 진입점(LoginPage)에서 리다이렉트 직전 `post_login_redirect` 쿠키 설정 한 줄 |
| wiki-front | vite `base: '/wiki/'`, 라우터 `basename="/wiki"`, auth 모듈 복사, 로그인 게이트, 헤더 사용자 표시+로그아웃 |
| alm-front | 동일 (`'/alm/'`, `"/alm"`) |

- auth 모듈: myFront `src/auth/{client,types}.ts` 복사 (in-flight refresh dedup 포함 —
  StrictMode 이중 refresh로 패밀리 폭파 방지 로직이 핵심이라 그대로 가져온다).
- 게이트 동작: 앱 마운트 → `tryRefresh()` → 실패 시 `post_login_redirect=<현재경로>` 쿠키
  (path=/, 비HttpOnly — 프론트가 심는 값) 설정 후 `location.href = /oauth2/authorization/keycloak`.
  성공 시 `/api/me` → 헤더에 사용자 표시. KC 세션이 살아 있으면 리다이렉트가 무프롬프트로
  왕복 — 이것이 SSO 체험.
- API base: 프로덕션(nginx)은 same-origin `''`, dev는 `VITE_API_BASE`(기본 `http://localhost:8000`)로 분기.
- wiki/alm 헤더 UI는 @chanho 디자인시스템으로 작성 (MUI 금지 계약 유지).

### 3. auth-server — returnTo 쿠키

`LoginSuccessHandler` 수정 (~15줄 + 테스트):
1. `post_login_redirect` 쿠키 읽기
2. 검증: `/`로 시작 **그리고** `//`로 시작 금지 (오픈 리다이렉트 방어 — 구글 `continue`
   파라미터 제한과 같은 이유). 실패/부재 시 기존 `/app` 폴백
3. `sendRedirect(frontendUrl + path)` + 쿠키 즉시 삭제(Max-Age=0, 일회용)

배포 env: `FRONTEND_URL=http://localhost`. Spring saved-request를 안 쓰는 이유:
SPA가 직접 `/oauth2/authorization/keycloak`으로 이동하는 구조라 "가로챈 원래 요청"이
없고, 커스텀 SuccessHandler가 이미 그 로직을 대체하고 있다.

### 4. Keycloak realm

`realm-export.json`: `redirectUris`/`webOrigins`에 `http://localhost/*` 계열 추가,
`post.logout.redirect.uris` 정렬. **반영은 realm 재import(`docker compose down -v && up -d`)
필요 — DB 볼륨 파괴라 수동 단계로 분리** (기존 웨이브와 동일 방식).

## 데이터 흐름 (SSO 시나리오)

```
① /wiki/pages/3 진입(미로그인) → tryRefresh 실패
② 쿠키 post_login_redirect=/wiki/pages/3 → /oauth2/authorization/keycloak
③ nginx → gateway → auth-server → KC 로그인 (KC SSO 세션 생성)
④ LoginSuccessHandler: JIT 프로비저닝 + RT 쿠키 set + 쿠키 검증 → /wiki/pages/3 복귀
⑤ 이후 /alm 진입 → 같은 RT 쿠키로 tryRefresh 즉시 성공 → 리다이렉트조차 없이 인증됨
   (RT가 만료됐어도 KC 세션이 살아 있으면 ②~④가 무프롬프트 자동 왕복)
⑥ 아무 앱에서 로그아웃 → 백채널 end_session + RT 쿠키 삭제 → 3앱 전부 로그아웃
```

## 에러 처리

- returnTo 쿠키 검증 실패(절대 URL, `//`) → 무시하고 `/app` 폴백 (테스트로 고정)
- gateway 미기동 시 nginx 프록시 502 — dev 인프라 특성상 기본 에러페이지 수용
- dist 미빌드 시 해당 경로 404 — README에 빌드 순서 명시로 대응
- 라우터 basename 밖 경로 → 각 SPA의 기존 NotFound 처리 그대로

## 알려진 리스크 (기록만, 이번 스코프 밖)

**RT 회전 + 재사용탐지 vs 다중 탭 동시 refresh**: 브라우저 세션 복원 등으로 wiki·alm 탭이
동시에 열리면 두 앱이 같은 RT로 동시에 `/api/auth/refresh`를 쳐서, 뒤진 쪽이 superseded
토큰 사용 → "도난 오인 → 패밀리 폐기 → 전체 로그아웃"이 날 수 있다. in-flight dedup은
앱 인스턴스 내부에만 작동하므로 앱이 3개가 되면서 표면이 커졌다. 정석 처방은 auth-server에
superseded 토큰 유예기간(~30초) — 다음 하드닝 웨이브 후보로 기록.

## 검증

- 단위: auth-server returnTo 테스트(유효/절대URL/`//`/부재 4케이스), wiki/alm 게이트·헤더
  테스트, 기존 게이트(wiki 77, alm 61, auth 14+) green 유지
- 빌드: 3앱 build + tsc green
- 수동 브라우저 E2E 체크리스트: 3앱 빌드 → compose up(nginx 포함) → 백엔드 4개 기동 →
  `/wiki` 진입 → KC 로그인 → `/wiki` 복귀 확인 → `/alm`·`/` 무프롬프트 진입(SSO) →
  로그아웃 → 3앱 전부 로그아웃 확인

## 대상 repo

MSA_TEMPLATE(우산: infra/nginx, realm) / auth-server / myFront / wiki-front / alm-front — 5개 교차

---
tags: [msa, template, nginx, sso, keycloak, oidc, spring-cloud-gateway, trusted-proxies, 운영전환, 구현기록]
작성일: 2026-07-15
상태: 구현·수정 완료 (커밋: my e02fdc2/fb96686, gateway 6de2fc1) · 구글 로그인 E2E만 잔존
---

# 12 — 크로스앱 내비게이션 + redirect_uri 사건 + 운영 전환 로드맵 (2026-07-15)

상위: [[00 개요 — 전체 구조]] · 선행: [[11 nginx 통합배포 + 3앱 SSO (2026-07-14)]]

## 무엇을 했나

11에서 만든 nginx 단일 오리진 위에 **앱 간 이동 진입점**을 붙였고, 첫 실제 브라우저 진입에서 터진 **"잘못된 매개변수: redirect_uri" 사건을 2중 원인까지 파서 해결**했다. 마지막으로 운영 전환 시 무엇이 바뀌는지 로드맵을 정리했다.

## 1. 크로스앱 내비게이션 (myFront)

위키/ALM은 myFront 라우터 밖의 **별도 SPA**다. react-router `navigate()`로 가면 NotFound에 걸리므로, `href` 항목은 `window.location.assign()`으로 **전체 페이지 이동**한다. 단일 오리진이라 SSO 세션은 그대로 유지된다.

진입점 4곳:

| 위치 | 내용 | 커밋 |
|---|---|---|
| `/app` 사이드메뉴 (`AppMenuContent`) | 위키/ALM 항목 — `MenuItem`에 `href?` 필드 추가 | my `e02fdc2` |
| 랜딩 헤더 (`LandingHeader`) | 위키/ALM 링크 (데스크톱 + 모바일 드로어) | my `fb96686` |
| 제품 카드 (`Home.tsx`) | ALM·WIKI 카드에 "라이브" 칩 (`Product.liveUrl`) | 〃 |
| 제품 상세 (`DetailPages`) | "라이브로 열기" contained + "소스 보기" outlined | 〃 |

> [!warning] dev 포트에서는 안 된다
> 이 링크들은 nginx 통합 오리진(`http://localhost`)에서만 동작. dev 모드(:5173/:5174/:5175)는 앱마다 오리진이 달라서 `/wiki/` 절대경로가 성립하지 않는다. alm dev 서버(:5175)에서 `/wiki/`를 열면 Vite가 "public base URL of /alm/ — did you mean /alm/wiki/?"를 내는 게 그 증상. **크로스앱 테스트는 `http://localhost`, 개별 개발은 각 dev 포트.**

## 2. redirect_uri 거부 사건 — 원인이 2중이었다

첫 실 브라우저 로그인 진입에서 Keycloak이 "잘못된 매개변수: redirect_uri"로 거부. 구간별 재현(curl)으로 파니 **독립적인 결함 2개가 겹쳐** 있었다.

### 원인 ① — SCG 4.1+가 X-Forwarded-* 를 버리고 있었다

Spring Cloud Gateway 4.1부터(이 repo는 Spring Cloud 2025.1.2) 보안 변경으로 **`trusted-proxies` 미설정 시 X-Forwarded-* 헤더를 신뢰하지 않고 제거**한다. 그래서:

```
nginx: X-Forwarded-Host: localhost 부착
  → gateway: trusted-proxies 없음 → 헤더 제거
    → auth-server(framework 전략): 폴백으로 자기가 받은 Host 사용
      = eureka prefer-ip-address 등록 IP → redirect_uri=http://172.18.240.1:9000/...
```

증거: auth-server(:9000)에 직접 `X-Forwarded-Host: localhost`를 주면 `http://localhost/...`로 정상 구성 — auth-server는 무죄, gateway가 범인.

수정: gateway `application.yml`에 `spring.cloud.gateway.server.webflux.trusted-proxies` 추가 — 루프백 + 사설대역(도커 NAT 포함) 정규식, **로컬 dev 전용** (gateway `6de2fc1`).

### 원인 ② — 실행 중인 Keycloak realm이 stale이었다

realm-export.json에는 redirectUris 3개(`:9000`/`:8000`/`localhost`)가 있지만, **`--import-realm`은 기존 realm을 덮어쓰지 않는다**. json을 고치기 전에 import된 옛 상태(`:9000` 하나)가 postgres 볼륨에 그대로 살아 있었다. DB 직접 조회로 확정.

수정: `down -v` 재import는 사용자 데이터까지 날리므로 **kcadm으로 실행 중 클라이언트만 현행화** — redirectUris/webOrigins/`post.logout.redirect.uris`를 json과 일치시킴.

> [!tip] kcadm on Windows 함정 2개
> ① Git Bash에서 컨테이너 내부 경로가 `C:/Program Files/Git/opt/...`로 변환됨 → `MSYS_NO_PATHCONV=1` 필수. ② PowerShell은 `-s 'redirectUris=[...]'` JSON 인용을 깨뜨림("Cannot parse the JSON") → JSON 파일을 `docker cp` 후 `kcadm update -f`로 전달.

### 검증

- nginx 경유 `/oauth2/authorization/keycloak` → `redirect_uri=http://localhost/login/oauth2/code/keycloak` 정상 전송
- 그 auth URL 추적 → KC 로그인 폼 HTTP 200 (keycloakify 번들 + 구글 버튼)
- 미등록 URI(`http://evil.example/cb`)는 여전히 거부 — 검증 로직 자체는 건재

> [!note] 교훈
> "seed json ≠ 실행 DB" 드리프트는 KC realm의 구조적 함정(02 테마 때도 loginTheme으로 한 번 겪음). realm 변경은 파일 수정으로 끝나지 않는다 — **kcadm으로 실행 DB까지 반영해야 한 세트.**

## 3. Q: 프론트가 3개면 nginx도 3대인가?

아니다 — **nginx 1대**. React 앱은 빌드하면 정적 파일 뭉치(`dist/`)일 뿐 실행 프로세스가 아니다. nginx 하나가 URL 경로를 보고 폴더 3개(`/srv/portal`, `/srv/apps/{wiki,alm}`) 중 어디서 파일을 줄지 분기한다. 1대로 합친 건 절약이 아니라 **단일 오리진 = SSO 쿠키 공유**라는 설계 그 자체다. dev 모드(:5173/:5174/:5175)만 진짜 서버 3개(Vite, HMR용)고, 배포 모드에선 폴더 3개 + nginx 1대로 수렴한다.

## 4. 운영으로 간다면 — 전환 로드맵

구조는 그대로, "localhost 전제"를 걷어내는 작업이 핵심.

### 층별 변경점

| 층 | 지금 (로컬) | 운영 |
|---|---|---|
| 오리진 | `http://localhost` | `https://도메인` — nginx TLS 종료, 단일 오리진 원칙 유지 |
| Keycloak | `start-dev`, redirectUris=localhost | `start` + `KC_HOSTNAME`, redirectUris를 운영 도메인으로, sslRequired |
| 쿠키 | Secure off | `COOKIE_SECURE=true` (이미 env화됨 — 값만) |
| nginx | 컨테이너 1개 | 여전히 1계층. 필요 시 N대 복제 + LB (정적 서빙은 무상태). 자산만 CDN 가능하되 **HTML 오리진은 도메인에 남긴다** |
| 백엔드 4개 | IntelliJ Run Config | 컨테이너화 (compose → 필요 시 k8s) |
| eureka | `prefer-ip-address: true` (Windows 호스트명 회피) | 컨테이너 네트워크에선 서비스명 DNS로 재검토 |
| trusted-proxies | 사설대역 전체 신뢰 (dev 광폭) | **실제 nginx/LB IP 대역으로 좁힌다** — 안 좁히면 내부망에서 X-Forwarded-Host 위조 가능 |
| 시크릿 | .env / env 기본값 | secret manager (Vault, 클라우드 SSM 등) — 외부화는 이미 완료, 주입원만 교체 |
| postgres/redis | 로컬 볼륨 | 관리형 DB or 백업 체계. **realm 변경은 kcadm/코드화**로 드리프트 방지 |

### 연쇄 주의점

- `X-Forwarded-Proto: https`가 nginx→gateway→auth-server 끝까지 전달돼야 redirect_uri가 https로 구성된다 (이번 사건과 같은 계열의 함정)
- T4 잔여 백로그와 정확히 겹침: HTTPS/TLS, JWK 키회전(kid 다중키), 시크릿 매니저, RT 만료행 정리 배치 ([[07 보안 감사 + 하드닝 (2026-07-02)]]). rate limiting·CB는 gateway에 이미 있음 ([[08 게이트웨이 트래픽 제어 + JWT 조기차단 (2026-07-03)]])

### 추천 경로 (2단계)

1. **VM 1대 + docker compose 전체 컨테이너화 + 도메인/TLS** — "운영 모양"을 먼저 완성
2. 트래픽/가용성 요구가 생기면 그때 k8s·다중 인스턴스 — 지금 구조에 이 전환을 막는 요소는 없다

## 남은 것

- 구글 실제 로그인 왕복 E2E (T1, 수동)
- gateway repo `.run/` 커밋 여부 결정 (다른 repo들은 Run Config 커밋됨)
- 운영 전환은 위 로드맵 1단계부터 — 착수 시 별도 스펙으로

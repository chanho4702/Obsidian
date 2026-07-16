---
tags: [msa, template, docker, compose, ghcr, cicd, keycloak, issuer, 구현기록]
작성일: 2026-07-17
상태: Wave 1 완료 (Task 1~5) · Wave 2~4(GitHub Actions/GHCR push/자동 배포) 대기
---

# 13 — 백엔드 컨테이너화 + compose 통합 — CI/CD Wave 1 (2026-07-15)

상위: [[00 개요 — 전체 구조]] · 선행: [[11 nginx 통합배포 + 3앱 SSO (2026-07-14)]], [[12 크로스앱 내비게이션 + redirect_uri 사건 + 운영 전환 로드맵 (2026-07-15)]]

## 무엇을 했나

[[12 크로스앱 내비게이션 + redirect_uri 사건 + 운영 전환 로드맵 (2026-07-15)|12의 운영 전환 로드맵]] 1단계("백엔드 4개 컨테이너화")를 실행에 옮긴 첫 웨이브. 전체 그림은 **GitHub Actions CI/CD: 4웨이브 13태스크** 플랜(`plans/2026-07-15-github-actions-cicd.md`)이고, 이번에 끝난 것은 그중 Wave 1 —

1. eureka / gateway / auth / board 4개 서비스 **Dockerfile 작성** (Task 1~2)
2. auth-server에 **`docker` Spring 프로필** 신설 (Task 3)
3. **docker compose에 백엔드 4서비스 통합** — 인프라 3개(postgres/keycloak/redis) + 백엔드 4개 + nginx = 8컨테이너 (Task 4)
4. **전체 스택 로컬 E2E** — 8컨테이너 healthy, 스모크 4종 200, 브라우저 SSO 확인 (Task 5)

이제 백엔드도 IntelliJ 없이 `docker compose up -d` 하나로 뜬다. 개발 모드(IDE 기동)와 컨테이너 모드가 **공존**한다.

## 1. Dockerfile — 왜 "런타임 전용"인가

4개 서비스 모두 같은 패턴:

```dockerfile
# 런타임 전용. jar는 CI(또는 로컬)의 `gradlew bootJar` 산출물을 복사한다.
FROM eclipse-temurin:24-jre
WORKDIR /app
COPY build/libs/app.jar app.jar
EXPOSE 9000
ENTRYPOINT ["java", "-jar", "app.jar"]
```

> [!note] 배경지식 — 멀티스테이지 빌드 대신 런타임 전용을 고른 이유
> Docker 이미지 빌드에는 두 유파가 있다.
> ① **멀티스테이지**: Dockerfile 안에서 gradle 빌드까지 수행(`FROM gradle AS build` → `FROM jre`). 어디서든 `docker build` 한 방이지만, 도커 레이어 캐시가 gradle 캐시보다 조악해서 CI가 느려지기 쉽다.
> ② **런타임 전용**: jar 빌드는 CI(또는 로컬)의 gradle이 담당하고, Dockerfile은 "완성된 jar를 JRE 위에 얹기"만 한다. gradle 캐시(의존성 캐싱, 증분 빌드)를 온전히 쓰고, 테스트→jar→이미지가 CI 파이프라인 단계로 자연스럽게 분리된다.
> 이 프로젝트는 **CI(GitHub Actions)가 빌드 주체**라는 전제라 ②를 택했다. 대신 "이미지를 만들려면 `gradlew bootJar`가 선행"이라는 순서 계약이 생긴다 — 이걸 어겨서 사건 1(아래)이 났다.

수용된 minor(후속 하드닝 후보): `.dockerignore` 부재(빌드 컨텍스트 비대), `USER`(비루트 실행)/`HEALTHCHECK` 지시자 없음 — 헬스체크는 compose 쪽에서 수행 중.

## 2. auth-server `docker` 프로필 (Task 3)

컨테이너 안에서는 "localhost"가 자기 자신이므로, 호스트 전제 설정을 프로필로 분리했다.

- DB·Keycloak·Eureka 주소를 컨테이너 내부 DNS(`postgres`, `keycloak`, `eureka`)로
- **JWK 발견**: 자체 JWT 서명키가 `platform.jwk-path` 프로퍼티(기본 `./auth-jwk.json`, env `PLATFORM_JWK_PATH`)로 파일 영속화됨을 확인 → compose에서 named volume `platform-auth-jwk:/data`로 마운트. **재배포해도 서명키가 유지**되어 기존 AT가 즉사하지 않는다.

> [!note] 배경지식 — 서명키가 휘발되면 생기는 일
> RS256 JWT는 auth-server의 개인키로 서명되고, board/gateway는 JWKS(공개키)로 검증한다([[01 인증 플랫폼 (Keycloak BFF + 자체 JWT)]]). 키가 컨테이너 파일시스템에만 있으면 재배포 = 새 키 = 유통 중인 모든 AT·JWKS 캐시가 무효 → 전 사용자 일시 401. 볼륨 영속화는 이를 막는 최소 장치고, 운영에선 **kid 다중 키 + 키 회전**으로 발전시킨다(T4 백로그, [[07 보안 감사 + 하드닝 (2026-07-02)]]).

## 3. compose 통합 (Task 4) — 구조 결정들

`infra/keycloak/docker-compose.yml` 하나로 8컨테이너. 핵심 결정:

| 결정 | 내용 | 이유 |
|---|---|---|
| 프로젝트명 `platform` 고정 | `name: platform` | 폴더명 의존 제거. 기존 데이터 볼륨은 `name: keycloak_platform-db` 실명 고정으로 **리네임에도 보존** |
| 이미지 = GHCR + build 폴백 | `image: ghcr.io/chanho4702/<svc>:${*_TAG:-latest}` + `build: ../../<svc>` | 평시엔 CI 산출 이미지, 로컬에선 `docker compose build`로 직접. `*_TAG` env는 **롤백용**(문제 시 `sha-<커밋>` 고정 재기동) |
| 기동 순서 = healthcheck 체인 | `depends_on: {condition: service_healthy}` | postgres→(keycloak, 백엔드), eureka→gateway 등. "떴다"가 아니라 "**준비됐다**"를 기다린다 |
| KC 헬스체크 | KC 26 관리 포트(9000) `/health/ready`를 bash `/dev/tcp`로 | 이미지에 curl이 없다. 순수 bash로 HTTP GET을 흉내내는 트릭 |
| nginx→gateway 내부 프록시 | nginx가 `host.docker.internal` 대신 `gateway:8000` 내부 DNS | 같은 compose 네트워크가 됐으니 호스트 왕복 제거 |
| 시크릿 | `POSTGRES_PASSWORD`·`KC_ADMIN_*`·`GOOGLE_*` 전부 `.env` 주입(기본값은 dev 전용) | [[07 보안 감사 + 하드닝 (2026-07-02)]]에서 시작한 외부화의 연장 |

> [!note] 배경지식 — `depends_on`은 기본적으로 "순서"만 보장한다
> compose의 `depends_on`은 기본값으로는 컨테이너 **시작 순서**만 정한다. postgres 프로세스가 떠도 DB 초기화가 끝나기 전이면 클라이언트는 연결 실패한다. `condition: service_healthy`는 healthcheck 통과까지 기다리게 하는 확장으로, 분산 시스템의 고전 문제("의존 서비스 준비 대기")를 compose 수준에서 푸는 표준 수법. postgres 헬스체크에 `-h 127.0.0.1`을 준 것도 같은 맥락 — 초기화 중 소켓 전용 임시 서버를 healthy로 오판하지 않도록 TCP를 강제한다.

## 4. E2E에서 터진 사건 2건 (Task 5)

### 사건 1 — 스테일 jar: "이미지는 빌드 시점의 스냅샷"

Task 2에서 이미지를 빌드한 **뒤에** Task 3의 yml 변경이 커밋됐다. 이미지 속 jar에는 docker 프로필이 없었고, 컨테이너는 프로필 미포함으로 기동. jar 재빌드 → 이미지 재빌드로 해소.

교훈: 런타임 전용 Dockerfile의 순서 계약(**bootJar → docker build**)은 사람이 지키기 어렵다 — 이게 바로 Wave 2(GitHub Actions)가 자동화할 대상이다.

### 사건 2 (실버그) — Keycloak은 iss를 "브라우저가 두드린 호스트"로 찍는다

설계 가정: 백채널(토큰 엔드포인트)을 `keycloak:8080`으로 부르니 토큰의 `iss`도 그 호스트일 것.
실측: **KC는 인가 요청이 들어온 호스트(브라우저 기준 `localhost:8080`)를 iss로 발급**했다. 컨테이너 안의 auth-server가 디스커버리 문서(`keycloak:8080` 기준)와 대조하면서 issuer 불일치로 실패.

- 프로퍼티 오버라이드(`issuer-uri`)로는 해결 불가 — Spring이 그 값으로 디스커버리를 강제해서 닭-달걀
- 해결: **docker 프로필 전용 수동 `ClientRegistration` 빈** — 각 엔드포인트를 명시적으로 짜맞춘다 (auth-server `1a8a88d`, RED→GREEN, 28 tests)

확정된 계약:

```
브라우저가 보는 KC     = localhost:8080   (인가 리다이렉트 URL)
백채널(토큰/로그아웃)   = keycloak:8080    (KEYCLOAK_ISSUER_URI 프로퍼티)
iss 검증 기준          = localhost:8080   (KEYCLOAK_FRONT_ISSUER env로 오버라이드 가능)
```

> [!note] 배경지식 — "이중 호스트" 문제는 컨테이너화된 OIDC의 단골
> IdP가 컨테이너 네트워크 안팎에서 다른 이름으로 보일 때(안: `keycloak:8080`, 밖: `localhost:8080`), OIDC의 issuer 일관성 검증(iss 클레임 == 디스커버리 문서의 issuer == 클라이언트 설정)이 깨진다. 정석 해법은 ① KC에 `KC_HOSTNAME`을 박아 어느 쪽에서든 같은 호스트를 쓰게 하거나(운영 방식), ② 클라이언트 쪽에서 프론트/백채널 URL을 분리 구성하는 것. 로컬 dev에서는 ①이 브라우저 접근성을 해치므로 ②(수동 ClientRegistration)를 택했다. 운영 전환 시(도메인 + `KC_HOSTNAME`) 이 분리는 자연 소멸 후보 — [[12 크로스앱 내비게이션 + redirect_uri 사건 + 운영 전환 로드맵 (2026-07-15)|12 로드맵]] 참조.

## 5. 개발 환경은 어떻게 되나 — 두 모드 공존

| | 개발 모드 (평시) | 컨테이너 모드 (통합 검증/배포 모양) |
|---|---|---|
| 인프라 3개 | `dev-up.ps1`이 compose로 | 동일 |
| 백엔드 4개 | **IntelliJ `.run/bootRun.run.xml`** (재시작·디버깅) | compose (GHCR 이미지 or 로컬 build) |
| 프론트 3개 | Windows Terminal 탭 3개 (:5173/:5174/:5175, `--strictPort`) | 빌드 후 nginx ro 마운트 (`http://localhost`) |

`scripts/dev-up.ps1` / `dev-down.ps1`이 우산 repo에 커밋됨(`d9628b6`). ps1은 UTF-8 BOM 저장 규칙.

## 남은 것

- **Wave 2~4**: GitHub Actions 워크플로(테스트→bootJar→이미지→GHCR push), 태그 규약(sha-커밋), 로컬 자동 배포. Task 5·6은 사용자 수동 개입 필요(시크릿 등)
- 하드닝 이월: `.dockerignore`, `USER`/`HEALTHCHECK` 지시자, JWK kid 다중 키 회전
- 수동분 이월: 구글 실로그인 E2E, 브라우저 체크리스트 잔여분

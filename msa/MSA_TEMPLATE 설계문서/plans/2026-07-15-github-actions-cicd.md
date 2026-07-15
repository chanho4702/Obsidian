# GitHub Actions CI/CD (GHCR + 로컬 자동 배포) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 7개 repo(백엔드 4 + 프론트 3)의 main push가 자동으로 테스트→빌드→GHCR/아티팩트→이 로컬 PC의 docker compose 스택·nginx dist에 반영되게 한다.

**Architecture:** 각 서비스 repo CI가 검증 후 GHCR 이미지(백엔드) 또는 dist 아티팩트(프론트)를 만들고 `repository_dispatch`로 infra-settings의 deploy 워크플로우를 호출한다. infra-settings에만 등록된 self-hosted runner(이 PC)가 pull + `compose up` / dist 폴더 교체를 수행한다. 백엔드 4개는 이번에 컨테이너화한다. 스펙: `specs/2026-07-15-github-actions-cicd-design.md`

**Tech Stack:** GitHub Actions, GHCR, Docker Compose, Spring Boot 4.0.6 / Java 24 / Gradle 9.5.1(wrapper), pnpm(wiki/alm/DS) + npm(myFront), PowerShell(runner 스텝)

## Global Constraints

- 실측 사실: realm 이름 `sso-demo`, Boot `4.0.6`, Java `24`, 백엔드 테스트는 H2(로컬 DB 불필요), wiki/alm은 pnpm(lockfile 있음), myFront는 npm(package-lock 있음), DS 버전 `@chanho/react 0.3.0` / `@chanho/tokens 0.2.0`
- GHCR 이미지 이름은 **repo 이름 그대로**: `ghcr.io/chanho4702/{eureka-server,gateway-server,auth-server,backend-server}` (전부 소문자 ✓)
- 이미지 태그는 항상 `latest` + `sha-<커밋sha>` 이중 태깅 (롤백용)
- 컨테이너 이름 컨벤션 유지: `platform-<svc>` (기존 platform-postgres/keycloak/redis/nginx)
- 기존 로컬 개발 흐름(IntelliJ .run으로 호스트 실행) 보존 — application.yml 기본값 무수정, docker 전용 값은 env/`docker` 프로필로만 주입
- 문서/원장 커밋 규칙: 작업기록은 Obsidian, repo에는 코드·설정만
- deploy 워크플로우 스텝은 PowerShell(`shell: pwsh`) — Windows runner
- 커밋 대상 repo가 5개(4 백엔드 + infra-settings)+4개(프론트/DS)로 나뉨 — **각 Task에 어느 repo에 커밋하는지 명시됨. 커밋 전 반드시 해당 repo 디렉토리인지 확인**

---

## Wave 1 — 백엔드 컨테이너화 + compose 통합 (CI 없이 로컬 완결)

### Task 1: eureka-server 컨테이너화 (패턴 확립)

**Files:**
- Modify: `C:\MSA_TEMPLATE\eureka-server\build.gradle` (bootJar 산출물 이름 고정)
- Create: `C:\MSA_TEMPLATE\eureka-server\Dockerfile`

**Interfaces:**
- Produces: 이미지 `ghcr.io/chanho4702/eureka-server` (로컬 태그 latest), jar 경로 계약 `build/libs/app.jar` — Task 2·8·9의 Dockerfile/CI가 동일 계약 사용

- [ ] **Step 1: bootJar 산출물 이름 고정**

`build.gradle` 끝에 추가 (Dockerfile COPY가 plain jar를 집는 사고 방지):

```groovy
// 컨테이너/CI가 집는 jar 경로를 고정한다 — 버전 문자열/plain jar 변동 차단.
tasks.named('bootJar') { archiveFileName = 'app.jar' }
```

- [ ] **Step 2: Dockerfile 작성 (런타임 전용 — jar는 밖에서 빌드)**

`eureka-server/Dockerfile`:

```dockerfile
# 런타임 전용. jar는 CI(또는 로컬)의 `gradlew bootJar` 산출물(build/libs/app.jar)을 복사한다.
# 컨테이너 안에서 gradle 빌드하지 않는 이유: CI runner의 gradle 캐시 재사용 + CRLF gradlew 이슈 회피.
FROM eclipse-temurin:24-jre
WORKDIR /app
COPY build/libs/app.jar app.jar
EXPOSE 8761
ENTRYPOINT ["java", "-jar", "app.jar"]
```

- [ ] **Step 3: 베이스 이미지 확인 + 로컬 빌드/기동 검증**

```powershell
docker pull eclipse-temurin:24-jre   # 없다고 나오면 eclipse-temurin:24-jdk로 Dockerfile 수정
cd C:\MSA_TEMPLATE\eureka-server
.\gradlew bootJar
docker build -t ghcr.io/chanho4702/eureka-server:latest .
docker run --rm -d -p 18761:8761 --name eureka-smoke ghcr.io/chanho4702/eureka-server:latest
```

30초 내: `Invoke-WebRequest http://localhost:18761 -UseBasicParsing` → StatusCode 200 (Eureka 대시보드).
확인 후 `docker rm -f eureka-smoke`.

- [ ] **Step 4: Commit (eureka-server repo)**

```powershell
cd C:\MSA_TEMPLATE\eureka-server
git add build.gradle Dockerfile
git commit -m "feat: 컨테이너화 — 런타임 전용 Dockerfile + bootJar 산출물 app.jar 고정"
```

### Task 2: gateway-server / auth-server / board-service 동일 컨테이너화

**Files:**
- Modify: `gateway-server\build.gradle`, `auth-server\build.gradle`, `board-service\build.gradle`
- Create: `gateway-server\Dockerfile`, `auth-server\Dockerfile`, `board-service\Dockerfile`

**Interfaces:**
- Consumes: Task 1의 계약 (`build/libs/app.jar`, 런타임 전용 Dockerfile 패턴)
- Produces: 이미지 `ghcr.io/chanho4702/gateway-server`(EXPOSE 8000), `ghcr.io/chanho4702/auth-server`(EXPOSE 9000), `ghcr.io/chanho4702/backend-server`(EXPOSE 9100)

- [ ] **Step 1: 3개 repo 각각 build.gradle에 동일 라인 추가**

```groovy
tasks.named('bootJar') { archiveFileName = 'app.jar' }
```

- [ ] **Step 2: Dockerfile 3개 작성 — Task 1과 동일하되 EXPOSE만 변경**

gateway-server `EXPOSE 8000`, auth-server `EXPOSE 9000`, board-service `EXPOSE 9100`:

```dockerfile
# 런타임 전용. jar는 CI(또는 로컬)의 `gradlew bootJar` 산출물(build/libs/app.jar)을 복사한다.
FROM eclipse-temurin:24-jre
WORKDIR /app
COPY build/libs/app.jar app.jar
EXPOSE 8000
ENTRYPOINT ["java", "-jar", "app.jar"]
```

- [ ] **Step 3: 3개 이미지 빌드 확인 (기동 검증은 Task 5의 전체 스택에서)**

```powershell
cd C:\MSA_TEMPLATE\gateway-server; .\gradlew bootJar; docker build -t ghcr.io/chanho4702/gateway-server:latest .
cd C:\MSA_TEMPLATE\auth-server;    .\gradlew bootJar; docker build -t ghcr.io/chanho4702/auth-server:latest .
cd C:\MSA_TEMPLATE\board-service;  .\gradlew bootJar; docker build -t ghcr.io/chanho4702/backend-server:latest .
```

각각 exit 0.

- [ ] **Step 4: repo별 커밋 (3회)**

각 repo에서: `git add build.gradle Dockerfile && git commit -m "feat: 컨테이너화 — 런타임 전용 Dockerfile + bootJar 산출물 app.jar 고정"`

### Task 3: auth-server `docker` 프로필 — Keycloak 이중 URL 해결

**Files:**
- Modify: `C:\MSA_TEMPLATE\auth-server\src\main\resources\application.yml` (프로필 문서 추가)

**Interfaces:**
- Consumes: 기존 env 계약 `KEYCLOAK_ISSUER_URI` (compose가 `http://keycloak:8080/realms/sso-demo` 주입)
- Produces: `docker` 프로필 — Task 4의 compose가 `SPRING_PROFILES_ACTIVE=docker`로 활성화

**원리 (구현자가 알아야 할 것):** Boot는 `issuer-uri`가 있으면 기동 시 디스커버리를 하고, **명시된 개별 엔드포인트 프로퍼티가 디스커버리 값을 덮어쓴다**. 컨테이너에서 issuer를 `keycloak:8080`으로 주면 토큰(iss)·token-uri·jwks·end_session이 전부 `keycloak:8080`으로 일관되고(Keycloak start-dev는 요청 호스트 기준 동적 issuer), **브라우저가 가는 authorization-uri 하나만** `localhost:8080`으로 덮어쓰면 된다. 백채널 로그아웃([[keycloak-backchannel-logout]])은 서버-서버라 keycloak:8080 그대로.

- [ ] **Step 1: JWK 파일 경로 확인 (재배포 시 토큰 무효화 여부 판단)**

```powershell
cd C:\MSA_TEMPLATE\auth-server; grep -rn "auth-jwk" src/main
```

- 경로가 설정 가능(env/property)이면: Task 4 compose에서 named volume으로 그 경로를 마운트하도록 메모(Task 4 Step 2에 반영).
- working dir 고정 상대경로면: 컨테이너 재배포 시 JWK 재생성 → 기존 AT 무효(재로그인). dev 수용 — Task 4에 마운트 불필요라고 메모하고 넘어간다.

- [ ] **Step 2: application.yml에 docker 프로필 문서 추가**

파일 끝에 추가:

```yaml
---
# docker 프로필 — 컨테이너 배포 전용 (compose가 SPRING_PROFILES_ACTIVE=docker 주입).
# 디스커버리/백채널은 KEYCLOAK_ISSUER_URI(keycloak:8080)로 일관 — iss 검증도 그 값과 일치.
# 브라우저 리다이렉트(authorization endpoint)만 호스트 오리진(localhost:8080)으로 오버라이드.
# Boot는 issuer 디스커버리 후 명시 프로퍼티로 덮어쓴다 — authorization-uri만 바뀐다.
spring:
  config:
    activate:
      on-profile: docker
  security:
    oauth2:
      client:
        provider:
          keycloak:
            authorization-uri: ${KEYCLOAK_BROWSER_BASE:http://localhost:8080}/realms/sso-demo/protocol/openid-connect/auth
```

- [ ] **Step 3: 기존 테스트 회귀 확인**

```powershell
cd C:\MSA_TEMPLATE\auth-server; .\gradlew test
```

Expected: BUILD SUCCESSFUL (프로필 문서 추가는 기본 프로필에 영향 없음).

- [ ] **Step 4: Commit (auth-server repo)**

```powershell
git add src/main/resources/application.yml
git commit -m "feat: docker 프로필 — KC 이중 URL(브라우저 authorization-uri만 localhost 오버라이드)"
```

### Task 4: compose에 백엔드 4서비스 통합 + nginx 프록시 전환 (infra-settings repo)

**Files:**
- Modify: `C:\MSA_TEMPLATE\infra\keycloak\docker-compose.yml`
- Modify: `C:\MSA_TEMPLATE\infra\nginx\default.conf` (proxy_pass 1곳)

**Interfaces:**
- Consumes: Task 1~3의 이미지·프로필
- Produces: compose 서비스명 `eureka`/`gateway`/`auth-server`/`board-service`, 컨테이너명 `platform-eureka`/`platform-gateway`/`platform-auth`/`platform-board`, 태그 env `EUREKA_TAG`/`GATEWAY_TAG`/`AUTH_TAG`/`BOARD_TAG`(기본 latest), 프로젝트명 `platform` — Task 7 deploy.yml이 이 이름들에 의존

- [ ] **Step 1: 기존 볼륨 실명 확인 (데이터 보존용 핀 값)**

```powershell
docker volume ls --filter name=platform-db
```

Expected: `keycloak_platform-db` (프로젝트명이 디렉토리명 keycloak에서 왔으므로). 다르면 그 실명을 Step 2의 `name:`에 사용.

- [ ] **Step 2: docker-compose.yml 수정**

(1) 파일 최상단에 프로젝트명 고정 (runner가 다른 경로에서 체크아웃해 실행해도 같은 스택):

```yaml
name: platform
```

(2) keycloak 서비스에 헬스체크 추가 — `environment:`에 `KC_HEALTH_ENABLED: "true"` 추가하고 서비스에:

```yaml
    healthcheck:
      # KC 26 관리 포트(9000)의 /health/ready. 이미지에 curl이 없어 bash /dev/tcp 사용.
      test: ['CMD-SHELL', 'exec 3<>/dev/tcp/127.0.0.1/9000; echo -e "GET /health/ready HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n" >&3; grep -q "200 OK" <&3']
      interval: 10s
      timeout: 5s
      retries: 30
      start_period: 30s
```

(3) nginx 서비스: `depends_on: [gateway]` 추가 (기동 시 upstream DNS 해석 실패 방지).

(4) 백엔드 4서비스 추가 (`redis:` 아래, `nginx:` 위):

```yaml
  # ── 백엔드 4서비스 — 이미지는 GHCR(CI 산출물), 로컬 개발은 build: 컨텍스트로 직접 빌드 가능.
  # 태그 env(*_TAG)는 롤백용: 기본 latest, 문제 시 sha-<커밋>으로 고정 재기동.
  eureka:
    image: ghcr.io/chanho4702/eureka-server:${EUREKA_TAG:-latest}
    build: ../../eureka-server
    container_name: platform-eureka
    healthcheck:
      test: ["CMD-SHELL", "bash -c 'exec 3<>/dev/tcp/127.0.0.1/8761'"]
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 20s

  gateway:
    image: ghcr.io/chanho4702/gateway-server:${GATEWAY_TAG:-latest}
    build: ../../gateway-server
    container_name: platform-gateway
    ports:
      - "8000:8000"   # deploy smoke + 로컬 디버깅용. nginx는 내부 DNS(gateway:8000)로 접근
    environment:
      EUREKA_URI: http://eureka:8761/eureka
      REDIS_HOST: redis
      AUTH_JWKS_URI: http://auth-server:9000/.well-known/jwks.json
    depends_on:
      eureka: { condition: service_healthy }
      redis: { condition: service_healthy }
    healthcheck:
      test: ["CMD-SHELL", "bash -c 'exec 3<>/dev/tcp/127.0.0.1/8000'"]
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 30s

  auth-server:
    image: ghcr.io/chanho4702/auth-server:${AUTH_TAG:-latest}
    build: ../../auth-server
    container_name: platform-auth
    environment:
      SPRING_PROFILES_ACTIVE: docker
      AUTH_DB_URL: jdbc:postgresql://postgres:5432/authdb
      EUREKA_URI: http://eureka:8761/eureka
      KEYCLOAK_ISSUER_URI: http://keycloak:8080/realms/sso-demo
      FRONTEND_URL: http://localhost
    depends_on:
      postgres: { condition: service_healthy }
      keycloak: { condition: service_healthy }
      eureka: { condition: service_healthy }
    healthcheck:
      test: ["CMD-SHELL", "bash -c 'exec 3<>/dev/tcp/127.0.0.1/9000'"]
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 40s

  board-service:
    image: ghcr.io/chanho4702/backend-server:${BOARD_TAG:-latest}
    build: ../../board-service
    container_name: platform-board
    environment:
      BOARD_DB_URL: jdbc:postgresql://postgres:5432/boarddb
      EUREKA_URI: http://eureka:8761/eureka
      AUTH_JWKS_URI: http://auth-server:9000/.well-known/jwks.json
    depends_on:
      postgres: { condition: service_healthy }
      eureka: { condition: service_healthy }
    healthcheck:
      test: ["CMD-SHELL", "bash -c 'exec 3<>/dev/tcp/127.0.0.1/9100'"]
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 40s
```

(Task 3 Step 1에서 JWK 경로가 설정 가능이었다면 auth-server에 named volume 추가:
`volumes: [platform-auth-jwk:<그 경로의 디렉토리>]` + 최하단 volumes에 `platform-auth-jwk:`)

(5) 프로젝트명 변경(keycloak→platform) 후에도 기존 KC/DB 데이터를 잃지 않도록 볼륨 실명 핀:

```yaml
volumes:
  platform-db:
    name: keycloak_platform-db   # Step 1에서 확인한 기존 실명 — 프로젝트 리네임에도 데이터 보존
```

- [ ] **Step 3: nginx default.conf — 게이트웨이 프록시 대상 전환**

`proxy_pass http://host.docker.internal:8000;` → `proxy_pass http://gateway:8000;`
(주석의 host.docker.internal 언급도 gateway로 갱신)

- [ ] **Step 4: 검증은 Task 5에서 통합 수행. 여기서는 구문만 확인**

```powershell
cd C:\MSA_TEMPLATE\infra\keycloak; docker compose config --quiet
```

Expected: exit 0 (경고만 허용, 에러 없음).

- [ ] **Step 5: Commit (infra-settings repo = C:\MSA_TEMPLATE)**

```powershell
cd C:\MSA_TEMPLATE
git add infra/keycloak/docker-compose.yml infra/nginx/default.conf
git commit -m "feat(infra): 백엔드 4서비스 compose 통합 — 프로젝트명 platform 고정, KC 헬스체크, nginx→gateway 내부 프록시"
```

### Task 5: 전체 스택 로컬 E2E (Wave 1 완료 게이트)

**Files:** 없음 (검증만)

- [ ] **Step 1: 호스트 백엔드 중지 + 구 프로젝트 정리**

IntelliJ에서 실행 중인 백엔드 4개 전부 중지(포트 8000/8761/9000/9100 충돌 방지 — [[dev-env-ide-split]] 흐름은 이제 선택지).

```powershell
docker compose -p keycloak down   # 구 프로젝트명 스택 제거 (볼륨은 유지됨 — -v 금지)
```

- [ ] **Step 2: 신규 스택 기동**

```powershell
cd C:\MSA_TEMPLATE\infra\keycloak
docker compose up -d --build
docker compose ps
```

Expected: 8개 컨테이너 전부 Up, 헬스체크 있는 것들 (healthy). auth-server가 재시작 루프면 `docker logs platform-auth`에서 디스커버리 실패 여부 확인(keycloak healthy 대기 문제).

- [ ] **Step 3: 스모크 3종**

```powershell
Invoke-WebRequest http://localhost:8000/.well-known/jwks.json -UseBasicParsing   # 게이트웨이→eureka→auth 체인
Invoke-WebRequest http://localhost/ -UseBasicParsing                              # nginx 정적
Invoke-WebRequest http://localhost:8080/realms/sso-demo -UseBasicParsing          # KC realm
```

Expected: 전부 200. (jwks는 eureka 등록 지연으로 최초 30초쯤 503 가능 — 재시도)

- [ ] **Step 4: 수동 SSO E2E (사용자 개입 필요 — 사용자에게 요청)**

브라우저에서 `http://localhost` → 로그인 → `/wiki/`·`/alm/` 무프롬프트 진입 → 로그아웃 → 3앱 로그아웃 확인. 게시판 API(myFront에서 board 기능) 1회 호출 확인.
**핵심 확인 포인트: 로그인 리다이렉트가 `localhost:8080`으로 가고, 콜백 후 세션이 생기는가 (이중 URL 검증).**

- [ ] **Step 5: 실패 시** — `docker logs platform-auth`의 iss 불일치/redirect_uri 오류를 우선 확인. authorization-uri 오버라이드가 안 먹으면(디스커버리가 덮어씀) Boot 4의 mapper 동작을 재확인하고, 최후 수단으로 issuer-uri 대신 5개 엔드포인트 전부 명시로 전환(스펙 리스크 섹션).

---

## Wave 2 — 파이프라인 1개 완주 (board-service → 로컬 자동 배포)

### Task 6: self-hosted runner + 시크릿 준비 (사용자 수동 단계 — 정확한 가이드 제공)

**Files:** 없음 (GitHub 설정 + 로컬 서비스 설치)

**전부 사용자가 브라우저/관리자 PowerShell에서 수행. 구현자는 아래를 안내하고 완료 확인만 한다.**

- [ ] **Step 1: PAT 2개 발급 안내**
  - `DISPATCH_PAT` (fine-grained): Resource owner `chanho4702`, 대상 repo: infra-settings, backend-server, auth-server, gateway-server, eureka-server, WIKI, ALM, ch-stack-frontend, design-system. 권한: **Contents: Read and write**(dispatch 발신용), **Actions: Read**(아티팩트 다운로드), Metadata: Read.
  - `GHCR_PULL_PAT` (classic): scope `read:packages` — GHCR docker login은 classic PAT 필요.

- [ ] **Step 2: DISPATCH_PAT를 7개 서비스 repo의 Actions secret으로 등록 안내**
  backend-server, auth-server, gateway-server, eureka-server, WIKI, ALM, ch-stack-frontend 각각: Settings → Secrets and variables → Actions → New secret, 이름 `DISPATCH_PAT`. infra-settings에도 등록(deploy가 아티팩트 다운로드에 사용).

- [ ] **Step 3: runner 설치 안내 (관리자 PowerShell)**
  infra-settings → Settings → Actions → Runners → New self-hosted runner (Windows x64)의 안내대로 다운로드/`config.cmd` 실행. 질문 중 "run as service?"에 **Y** (부팅 자동 시작). 라벨 기본값(self-hosted, Windows, X64) 유지.

- [ ] **Step 4: runner 머신에서 GHCR 로그인 1회**

```powershell
docker login ghcr.io -u chanho4702   # 비밀번호 = GHCR_PULL_PAT
```

- [ ] **Step 5: 완료 확인** — infra-settings Runners 목록에 Idle 상태 runner 표시.

### Task 7: infra-settings deploy 워크플로우

**Files:**
- Create: `C:\MSA_TEMPLATE\.github\workflows\deploy.yml`

**Interfaces:**
- Consumes: Task 4의 compose 서비스명/컨테이너명/태그 env, Task 6의 runner·secrets
- Produces: `repository_dispatch(event_type: deploy)` 수신 계약 — payload `{service, sha, run_id, repo}`. service 값: `eureka-server|gateway-server|auth-server|backend-server|myfront|wiki|alm`. Task 8·9·11의 CI가 이 계약으로 발신

- [ ] **Step 1: deploy.yml 작성**

```yaml
name: Deploy
on:
  repository_dispatch:
    types: [deploy]
  workflow_dispatch:   # dispatch 유실/수동 재배포용
    inputs:
      service: { description: "eureka-server|gateway-server|auth-server|backend-server|myfront|wiki|alm", required: true }
      run_id:  { description: "프론트: dist 아티팩트가 있는 CI run id", required: false }
      repo:    { description: "프론트: 소스 repo (예: chanho4702/WIKI)", required: false }
      tag:     { description: "백엔드: 이미지 태그 (기본 latest, 롤백 시 sha-<커밋>)", required: false, default: "latest" }

concurrency:
  group: deploy-${{ github.event.client_payload.service || inputs.service }}
  cancel-in-progress: false

env:
  SERVICE: ${{ github.event.client_payload.service || inputs.service }}
  RUN_ID:  ${{ github.event.client_payload.run_id  || inputs.run_id }}
  SRC_REPO: ${{ github.event.client_payload.repo   || inputs.repo }}
  IMAGE_TAG: ${{ inputs.tag || 'latest' }}

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4

      - name: Deploy backend (pull + up)
        if: contains('eureka-server gateway-server auth-server backend-server', env.SERVICE)
        shell: pwsh
        run: |
          $svcMap = @{ 'eureka-server'='eureka'; 'gateway-server'='gateway'; 'auth-server'='auth-server'; 'backend-server'='board-service' }
          $tagMap = @{ 'eureka-server'='EUREKA_TAG'; 'gateway-server'='GATEWAY_TAG'; 'auth-server'='AUTH_TAG'; 'backend-server'='BOARD_TAG' }
          $ctrMap = @{ 'eureka-server'='platform-eureka'; 'gateway-server'='platform-gateway'; 'auth-server'='platform-auth'; 'backend-server'='platform-board' }
          $svc = $svcMap[$env:SERVICE]
          Set-Item "env:$($tagMap[$env:SERVICE])" $env:IMAGE_TAG
          docker compose -f infra/keycloak/docker-compose.yml pull $svc
          if ($LASTEXITCODE -ne 0) { exit 1 }
          docker compose -f infra/keycloak/docker-compose.yml up -d --no-build $svc
          if ($LASTEXITCODE -ne 0) { exit 1 }
          # healthy 대기 (최대 2분)
          $ctr = $ctrMap[$env:SERVICE]
          foreach ($i in 1..24) {
            $h = docker inspect --format '{{.State.Health.Status}}' $ctr
            if ($h -eq 'healthy') { exit 0 }
            Start-Sleep -Seconds 5
          }
          Write-Error "$ctr not healthy after 120s"; exit 1

      - name: Deploy frontend (dist swap)
        if: contains('myfront wiki alm', env.SERVICE)
        shell: pwsh
        env:
          GH_TOKEN: ${{ secrets.DISPATCH_PAT }}
        run: |
          $dirMap = @{ 'myfront'='portal'; 'wiki'='wiki'; 'alm'='alm' }
          $dir = $dirMap[$env:SERVICE]
          $base = 'C:\deploy\dist'
          $new = Join-Path $base "$dir.new"; $cur = Join-Path $base $dir; $bak = Join-Path $base "$dir.bak"
          if (Test-Path $new) { Remove-Item -Recurse -Force $new }
          New-Item -ItemType Directory -Force $new | Out-Null
          gh run download $env:RUN_ID -R $env:SRC_REPO -n dist -D $new
          if ($LASTEXITCODE -ne 0) { exit 1 }
          if (Test-Path $bak) { Remove-Item -Recurse -Force $bak }
          if (Test-Path $cur) { Move-Item $cur $bak }
          Move-Item $new $cur
          # Windows 바인드 마운트가 폴더 교체 후 stale해질 수 있어 nginx만 재시작(1초 미만)
          docker compose -f infra/keycloak/docker-compose.yml restart nginx

      - name: Smoke
        shell: pwsh
        run: |
          $ok = $false
          foreach ($i in 1..30) {
            try {
              $r1 = Invoke-WebRequest http://localhost/ -UseBasicParsing -TimeoutSec 5
              $r2 = Invoke-WebRequest http://localhost:8000/.well-known/jwks.json -UseBasicParsing -TimeoutSec 5
              if ($r1.StatusCode -eq 200 -and $r2.StatusCode -eq 200) { $ok = $true; break }
            } catch {}
            Start-Sleep -Seconds 5
          }
          if (-not $ok) { Write-Error "smoke failed (nginx or gateway->auth chain)"; exit 1 }
```

주의: 프론트 스텝은 Wave 4의 `C:\deploy\dist` 전환(Task 10) 전까지는 실행되지 않는다(백엔드 service 값만 들어옴). `gh` CLI가 runner PATH에 있어야 함 — 없으면 Task 6에서 `winget install GitHub.cli` 안내 추가.

- [ ] **Step 2: Commit + push (infra-settings repo)**

```powershell
cd C:\MSA_TEMPLATE
git add .github/workflows/deploy.yml
git commit -m "feat(ci): repository_dispatch 수신 deploy 워크플로우 — self-hosted runner에서 pull+up/dist swap"
git push
```

- [ ] **Step 3: 수동 트리거로 deploy 단독 검증 (CI 없이)**

GitHub UI에서 infra-settings → Actions → Deploy → Run workflow: service=`backend-server`, tag=`latest`.
단, GHCR에 이미지가 아직 없으므로 이 시점 pull은 실패가 정상 — **Task 8 완료 후 재검증**. 여기서는 워크플로우가 runner에 잡히는 것(Queued→러너 실행)만 확인.

### Task 8: board-service CI → 파이프라인 완주

**Files:**
- Create: `C:\MSA_TEMPLATE\board-service\.github\workflows\ci.yml`

**Interfaces:**
- Consumes: Task 7의 dispatch 계약, secret `DISPATCH_PAT`
- Produces: GHCR `ghcr.io/chanho4702/backend-server:{latest, sha-<sha>}` — Wave 3 CI들의 원형

- [ ] **Step 1: ci.yml 작성**

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read
  packages: write

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "24"
      - uses: gradle/actions/setup-gradle@v4   # gradle 캐시
      - run: chmod +x gradlew                  # Windows에서 커밋된 실행비트 보정
      - run: ./gradlew test bootJar --no-daemon

      - name: Login GHCR
        if: github.ref == 'refs/heads/main'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build & push image
        if: github.ref == 'refs/heads/main'
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ghcr.io/chanho4702/backend-server:latest
            ghcr.io/chanho4702/backend-server:sha-${{ github.sha }}

      - name: Dispatch deploy
        if: github.ref == 'refs/heads/main'
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.DISPATCH_PAT }}
          repository: chanho4702/infra-settings
          event-type: deploy
          client-payload: '{"service":"backend-server","sha":"${{ github.sha }}","run_id":"${{ github.run_id }}","repo":"${{ github.repository }}"}'
```

- [ ] **Step 2: Commit + push (board-service repo)**

```powershell
cd C:\MSA_TEMPLATE\board-service
git add .github/workflows/ci.yml
git commit -m "feat(ci): test→GHCR push→infra-settings deploy dispatch"
git push
```

- [ ] **Step 3: E2E 관찰**

backend-server repo Actions에서 CI green 확인 → infra-settings Actions에 Deploy 실행 자동 생성 확인 → green 확인.

```powershell
docker inspect --format '{{.Image}}' platform-board   # 새 이미지 digest로 바뀜
docker image ls ghcr.io/chanho4702/backend-server     # latest + sha-* 태그 존재
```

- [ ] **Step 4: (사용자) 트리비얼 커밋으로 재현 확인** — 주석 한 줄 수정 push → 몇 분 뒤 로컬 반영 자동 확인. **이게 Wave 2 완료 게이트.**

- [ ] **Step 5: GHCR 패키지 비공개 확인** — github.com/chanho4702?tab=packages에서 backend-server 패키지 visibility가 Private인지 확인(공개면 Private로 변경).

---

## Wave 3 — 백엔드 3개 복제

### Task 9: eureka/gateway/auth ci.yml

**Files:**
- Create: `eureka-server\.github\workflows\ci.yml`, `gateway-server\.github\workflows\ci.yml`, `auth-server\.github\workflows\ci.yml`

**Interfaces:**
- Consumes: Task 8의 ci.yml 원형, Task 7 dispatch 계약

- [ ] **Step 1: Task 8 Step 1의 YAML을 3개 repo에 복사하되 두 곳만 치환**

| repo | 이미지 이름(tags 2곳) | client-payload service |
|---|---|---|
| eureka-server | `ghcr.io/chanho4702/eureka-server` | `eureka-server` |
| gateway-server | `ghcr.io/chanho4702/gateway-server` | `gateway-server` |
| auth-server | `ghcr.io/chanho4702/auth-server` | `auth-server` |

나머지 내용(트리거, permissions, 스텝 전부)은 Task 8과 문자 그대로 동일.

- [ ] **Step 2: repo별 commit + push (3회)** — 메시지 `feat(ci): test→GHCR push→infra-settings deploy dispatch`

- [ ] **Step 3: 3개 repo Actions green + Deploy 3건 green 확인.** auth-server 배포 후 브라우저 로그인 1회 재확인(세션/JWK 영향 확인 — Task 3 Step 1의 판단 검증).

---

## Wave 4 — 프론트 3개 + design-system

### Task 10: dist 전용 폴더 전환 (infra-settings repo)

**Files:**
- Modify: `C:\MSA_TEMPLATE\infra\keycloak\docker-compose.yml` (nginx volumes 3줄)

**Interfaces:**
- Produces: 배포 대상 고정 경로 `C:\deploy\dist\{portal,wiki,alm}` — Task 7 프론트 스텝의 $base와 일치

- [ ] **Step 1: 폴더 생성 + 현재 dist 이관**

```powershell
New-Item -ItemType Directory -Force C:\deploy\dist | Out-Null
Copy-Item -Recurse C:\MSA_TEMPLATE\myFront\dist    C:\deploy\dist\portal
Copy-Item -Recurse C:\MSA_TEMPLATE\wiki-front\dist C:\deploy\dist\wiki
Copy-Item -Recurse C:\MSA_TEMPLATE\alm-front\dist  C:\deploy\dist\alm
```

- [ ] **Step 2: compose nginx volumes 교체**

```yaml
      - ../nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - C:/deploy/dist/portal:/srv/portal:ro
      - C:/deploy/dist/wiki:/srv/apps/wiki:ro
      - C:/deploy/dist/alm:/srv/apps/alm:ro
```

(소스 트리 마운트 제거 — runner 체크아웃 위치와 무관하게 절대경로로 고정.
주석도 갱신: "dist는 deploy 워크플로우가 C:\deploy\dist에 교체 — 로컬 수동 반영은 빌드 후 해당 폴더로 복사")

- [ ] **Step 3: nginx 재생성 + 3앱 확인**

```powershell
cd C:\MSA_TEMPLATE\infra\keycloak; docker compose up -d nginx
Invoke-WebRequest http://localhost/ -UseBasicParsing        # 200
Invoke-WebRequest http://localhost/wiki/ -UseBasicParsing   # 200
Invoke-WebRequest http://localhost/alm/ -UseBasicParsing    # 200
```

- [ ] **Step 4: Commit (infra-settings)** — `git add infra/keycloak/docker-compose.yml && git commit -m "feat(infra): nginx dist 마운트를 배포 전용 C:/deploy/dist로 전환" && git push`

### Task 11: wiki/alm CI (DS checkout 포함) + myFront CI

**Files:**
- Create: `wiki-front\.github\workflows\ci.yml`, `alm-front\.github\workflows\ci.yml`, `myFront\.github\workflows\ci.yml`

**Interfaces:**
- Consumes: Task 7 dispatch 계약(service=`wiki`/`alm`/`myfront`), 아티팩트 이름 계약 `dist`, secret `DISPATCH_PAT`(dispatch + DS checkout)
- Produces: `actions/upload-artifact` 이름 `dist` — Task 7의 `gh run download -n dist`와 일치

- [ ] **Step 1: wiki-front ci.yml 작성**

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          path: wiki-front
      - uses: actions/checkout@v4
        with:
          repository: chanho4702/design-system
          token: ${{ secrets.DISPATCH_PAT }}
          path: design-system   # file:../design-system/... 상대경로를 그대로 만족
      - uses: pnpm/action-setup@v4
        with:
          version: 11   # wiki/alm package.json에 packageManager 필드 없음 — 명시 필요 (DS는 pnpm@11.10.0)
      - uses: actions/setup-node@v4
        with:
          node-version: 22

      - name: Build design-system artifacts (tgz)
        working-directory: design-system
        run: |
          pnpm install --frozen-lockfile
          pnpm build:tokens
          pnpm --filter @chanho/react build
          mkdir -p artifacts
          pnpm --filter @chanho/tokens pack --pack-destination "$GITHUB_WORKSPACE/design-system/artifacts"
          pnpm --filter @chanho/react  pack --pack-destination "$GITHUB_WORKSPACE/design-system/artifacts"
          ls artifacts   # chanho-react-0.3.0.tgz, chanho-tokens-0.2.0.tgz 확인

      - name: Test & build
        working-directory: wiki-front
        run: |
          pnpm install --no-frozen-lockfile   # file: tgz 재생성으로 integrity가 lock과 다를 수 있음
          pnpm test
          pnpm build

      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: wiki-front/dist

      - name: Dispatch deploy
        if: github.ref == 'refs/heads/main'
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.DISPATCH_PAT }}
          repository: chanho4702/infra-settings
          event-type: deploy
          client-payload: '{"service":"wiki","sha":"${{ github.sha }}","run_id":"${{ github.run_id }}","repo":"${{ github.repository }}"}'
```

- [ ] **Step 2: alm-front ci.yml** — 위와 동일하되 치환: checkout `path: alm-front`, working-directory 2곳 `alm-front`, upload path `alm-front/dist`, payload service `"alm"`.

- [ ] **Step 3: myFront ci.yml (npm, DS 의존 없음)**

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build   # tsc -b가 타입 검증 겸함 (test 스크립트 없음 — 있으면 test 스텝 추가)
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist
      - name: Dispatch deploy
        if: github.ref == 'refs/heads/main'
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.DISPATCH_PAT }}
          repository: chanho4702/infra-settings
          event-type: deploy
          client-payload: '{"service":"myfront","sha":"${{ github.sha }}","run_id":"${{ github.run_id }}","repo":"${{ github.repository }}"}'
```

- [ ] **Step 4: repo별 commit + push (3회)** — 메시지 `feat(ci): build→dist 아티팩트→infra-settings deploy dispatch`

- [ ] **Step 5: E2E** — 각 repo Actions green → Deploy green → `C:\deploy\dist\*` 타임스탬프 갱신 확인 → 브라우저에서 3앱 정상.

### Task 12: design-system CI (검증 전용)

**Files:**
- Create: `design-system\.github\workflows\ci.yml`

- [ ] **Step 1: ci.yml 작성**

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm test
      - run: pnpm build:tokens
      - run: pnpm --filter @chanho/react build
```

배포·dispatch 없음 (DS 변경은 wiki/alm의 다음 push 때 반영 — 스펙 비목표).

- [ ] **Step 2: Commit + push (design-system repo)** — `feat(ci): typecheck/test/build 검증`. Actions green 확인.

### Task 13: 마감 — 최종 E2E + 원장/메모리

- [ ] **Step 1: 전체 파이프라인 최종 확인** — 백엔드 1개(트리비얼 커밋)와 프론트 1개(문구 수정) push → 둘 다 자동 반영 확인. 로그인/SSO 1회 재확인.
- [ ] **Step 2: 원장 갱신** — `C:\MSA_TEMPLATE\.superpowers\sdd\progress.md`에 웨이브 결과 기록, infra-settings 커밋·push.
- [ ] **Step 3: 메모리 갱신** — `keycloak-jwt-platform-build` 메모리에 CI/CD 완료 사실 추가, 새 메모리(cicd 구조: dispatch 계약·runner·롤백 방법) 작성.
- [ ] **Step 4: 롤백 리허설(선택)** — Deploy 수동 실행으로 backend-server를 `sha-<이전커밋>` 태그로 재기동해 롤백 경로 1회 검증.

---

## 리스크 → 담당 Task 매핑 (스펙 커버리지)

| 스펙 항목 | Task |
|---|---|
| Keycloak 이중 URL | 3, 5 |
| 컨테이너화 + compose | 1, 2, 4 |
| self-hosted runner/시크릿 | 6 |
| 중앙 deploy + 롤백(sha 태그, .bak) | 7, 13 |
| 백엔드 CI ×4 | 8, 9 |
| 프론트 CI ×3 + DS checkout | 11 |
| DS CI | 12 |
| dist 전용 폴더 | 10 |
| dispatch 유실 대비 workflow_dispatch | 7 |
| DS 버전 하드코딩 리스크 | 11 Step 1의 ls artifacts 확인 |

# GitHub Actions CI/CD — GHCR 이미지 + 로컬 자동 배포 — 설계

날짜: 2026-07-15
상태: 승인됨 (브레인스토밍 Q&A로 갈림길 5개 전부 사용자 확정)

## 배경 / 문제의식

플랫폼이 7개 repo로 분리돼 있다 — 백엔드 4개(auth-server, backend-server=board-service,
gateway-server, eureka-server), 프론트 3개(ch-stack-frontend=myFront, WIKI=wiki-front,
ALM=alm-front), 라이브러리 1개(design-system), 인프라 1개(infra-settings=MSA_TEMPLATE).
지금은 전부 수동이다: 백엔드는 호스트에서 gradle로 직접 실행, 프론트는 로컬에서 `vite build`
후 dist를 nginx 컨테이너에 볼륨 마운트. push해도 아무 일도 안 일어난다.

**목표: main에 push하면 빌드·테스트·검증을 거쳐 이 로컬 PC의 실행 환경까지 자동 반영.**

## 확정된 결정 (브레인스토밍 Q&A)

| 갈림길 | 결정 | 근거 |
|---|---|---|
| 목표 범위 | **CI + GHCR 이미지 푸시 + 로컬 자동 배포** | "cicd 로컬에 자동 배포" — 검증만이 아니라 반영까지 |
| 대상 repo | **백엔드 4 + 프론트 3 전부** (design-system은 CI만) | 한 번에 완성 |
| 로컬 배포 방식 | **Self-hosted runner** (이 PC, Windows) | 배포 결과가 Actions UI에 보이고, dist 교체 같은 파일 작업도 같은 job에서 처리 가능. Watchtower는 컨테이너 교체만 되고 폴링 지연 있음 |
| 프론트 배포 | **dist 교체 유지** (컨테이너화 안 함) | 단일 오리진 SSO nginx(1컨테이너) 구조 무수정. CI가 dist 아티팩트 업로드 → runner가 폴더 교체 |
| DS 의존성 해결 | **CI에서 design-system을 옆에 checkout + build + pack** | wiki/alm의 `file:../design-system/artifacts/*.tgz` 의존성을 package.json 무수정으로 만족 |
| 배포 오케스트레이션 | **중앙 배포 repo(infra-settings) + repository_dispatch** | 개인 계정은 runner가 repo 단위 등록 → 1곳에만 등록하고 서비스 repo들이 dispatch로 호출. 배포 로직·이력이 한 곳에 모임 |

## 목표

1. 각 서비스 repo `main` push → 자동으로 테스트 → 빌드 → (백엔드) GHCR 이미지 푸시 / (프론트) dist 아티팩트
2. 성공 시 repository_dispatch → infra-settings의 deploy 워크플로우가 self-hosted runner에서 로컬 반영
3. 백엔드 4개 컨테이너화 — docker compose 스택에 통합 (eureka → gateway/auth/board)
4. PR에서는 CI만 (배포 없음). 배포는 main push에만
5. 롤백 가능: 이미지 `sha-<커밋>` 태그, dist 이전 버전 1개 백업

## 비목표 (YAGNI)

- 클라우드/원격 서버 배포, HTTPS, 도메인 — 로컬 PC가 유일한 배포 대상
- 프론트 컨테이너화 — nginx 1개 + dist 마운트 유지
- design-system의 npm registry 배포(GitHub Packages) — file: 의존성 + CI checkout으로 충분. registry 전환은 필요해지면
- Kubernetes, ArgoCD 류 — docker compose로 충분
- staging 환경 — main = 로컬 반영, 그게 전부

## 전체 아키텍처

```
[서비스 repo push(main)]
  ├─ 백엔드 4개: gradle test → docker build → GHCR push (latest + sha-<커밋>)
  └─ 프론트 3개: DS checkout+build+pack → vitest → vite build → dist 아티팩트 업로드
           │ (CI 성공 시)
           ▼
  repository_dispatch → infra-settings (payload: service, sha, run_id)
           ▼
[infra-settings deploy.yml — self-hosted runner(이 PC)]
  ├─ 백엔드: docker compose pull <svc> → up -d <svc> → 헬스 확인
  └─ 프론트: run_id로 아티팩트 다운로드 → dist 폴더 원자적 교체 (nginx 무재기동)
```

## 설계

### 1. 백엔드 컨테이너화 (이번 웨이브 최대 변경)

각 백엔드 repo에 멀티스테이지 Dockerfile:

```dockerfile
FROM gradle:8-jdk21 AS build        # (repo의 실제 JDK 버전에 맞춤)
COPY . .
RUN gradle bootJar --no-daemon
FROM eclipse-temurin:21-jre
COPY --from=build /home/gradle/build/libs/*.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

infra-settings compose에 4개 서비스 추가 (`image: ghcr.io/chanho4702/<repo>:latest`):
- 의존 순서: eureka(healthy) → gateway, auth-server, board-service
- 기존 postgres/redis/keycloak/nginx와 같은 네트워크
- nginx `default.conf`: `proxy_pass http://host.docker.internal:8000` → `http://gateway:8000`
- Spring `docker` 프로필 신설(환경변수 주입): eureka URL `http://eureka:8761/eureka`,
  DB `postgres:5432`, redis `redis:6379`. 기존 호스트 실행(local 프로필)은 그대로 살려둠

### 2. Keycloak 이중 URL 문제 (핵심 리스크)

컨테이너 백엔드가 Keycloak을 `http://keycloak:8080`으로 접근하면, 브라우저 플로우로 발급된
토큰의 issuer(`http://localhost:8080/realms/platform`)와 불일치 → Spring 검증 실패.

**해결: issuer-uri 디스커버리를 버리고 엔드포인트 개별 지정 (docker 프로필에서만)**
- 브라우저가 가는 곳(authorization-uri, end-session의 리다이렉트): `http://localhost:8080/...`
- 서버-서버 백채널(token-uri, jwk-set-uri, user-info-uri, 백채널 로그아웃의 end_session):
  `http://keycloak:8080/...`
- JWT 검증: `jwk-set-uri`를 keycloak:8080으로 직접 지정 + issuer 검증값은
  `http://localhost:8080/realms/platform` 고정 (jwk-set-uri가 있으면 Spring은 디스커버리 호출 안 함)

백채널 로그아웃([[keycloak-backchannel-logout]] 계약)은 서버-서버 호출이므로 keycloak:8080
쪽 — 프론트채널로 되돌리지 않는다.

### 3. 서비스 repo CI 워크플로우

**백엔드 공통 (`.github/workflows/ci.yml`, repo당 1개 — 내용 거의 동일):**
- `on: push(main), pull_request`
- job build-test: JDK 셋업 → `gradle test` → (main일 때만) `docker/build-push-action`으로
  GHCR push, 태그 `latest` + `sha-${{ github.sha }}`. 권한은 기본 `GITHUB_TOKEN`(packages: write)
- job notify-deploy (main만): `peter-evans/repository-dispatch`로 infra-settings에
  `event_type: deploy`, payload `{service, sha, run_id}`. secret `DISPATCH_PAT` 사용

**프론트 공통 (wiki/alm — DS 의존 있음):**
- checkout 자기 repo + `chanho4702/design-system`을 `../design-system`에 checkout
- DS: `pnpm install` → `pnpm build:tokens` + react 빌드 → `pnpm pack`으로
  `artifacts/chanho-react-0.3.0.tgz`, `chanho-tokens-0.2.0.tgz` 생성 (버전은 package.json의 file: 경로와 일치해야 함)
- 프론트: `npm ci` → `vitest run` → `vite build` → `actions/upload-artifact`로 dist 업로드
- main이면 dispatch (payload에 run_id — deploy가 이걸로 아티팩트를 다운로드)

**myFront:** DS 의존 없음(확인됨) → checkout 단계 생략, 나머지 동일 (`tsc -b && vite build`)

**design-system:** CI만 — `pnpm install` → `typecheck` → `test` → `build`. 배포·dispatch 없음.
단, DS가 바뀌어도 wiki/alm이 자동 재빌드되지는 않음(다음 push 때 반영) — 비목표로 수용

### 4. 중앙 배포 (infra-settings `deploy.yml`)

- `on: repository_dispatch: types: [deploy]`, `runs-on: self-hosted`
- `concurrency: deploy-${{ payload.service }}` — 같은 서비스 연속 push 시 직렬화/취소
- 분기:
  - 백엔드(eureka/gateway/auth/board): `docker compose pull <svc> && docker compose up -d <svc>`
    → healthcheck 대기 → smoke curl (gateway `/actuator/health` 등)
  - 프론트(myfront/wiki/alm): `gh run download <run_id> -R <repo>` (DISPATCH_PAT로 인증)
    → 임시 폴더에 풀기 → 기존 폴더를 `.bak`으로 이동 → 새 폴더로 rename (원자적 교체,
    nginx 재기동 불필요) → 이전 `.bak` 1개만 유지
- dist 실경로: 지금은 compose가 `../../myFront/dist` 등 소스 repo 안을 마운트 — 배포 대상
  경로를 소스 트리와 분리하기 위해 `C:\deploy\dist\{portal,wiki,alm}` 같은 전용 폴더로
  compose 마운트를 변경 (로컬 개발 중 수동 빌드 반영이 필요하면 해당 폴더로 복사)

### 5. Self-hosted runner (이 PC)

- infra-settings에 repo 단위 등록, **Windows 서비스로 설치**(부팅 시 자동 시작)
- 사전 준비 1회: `docker login ghcr.io` (GHCR pull), `gh auth` 또는 PAT 환경변수(아티팩트 다운로드)
- 보안: 이 runner는 infra-settings에만 물리고, infra-settings는 fork PR을 받지 않는 개인
  repo이므로 임의 코드 실행 리스크는 수용 범위

### 6. 시크릿

| 시크릿 | 위치 | 용도 |
|---|---|---|
| `DISPATCH_PAT` (fine-grained: infra-settings contents:write 또는 repo dispatch 권한 + 각 서비스 repo actions:read) | 서비스 repo 7개 각각 + runner 환경 | dispatch 발신, 아티팩트 크로스 repo 다운로드 |
| GHCR 인증 | CI: 기본 GITHUB_TOKEN / 로컬: docker login 1회 | 이미지 push/pull |
| Keycloak/Google `.env` | 로컬 파일만 (기존 그대로) | CI에 올리지 않음 |

## 에러 처리 / 롤백

- 테스트 실패 → 이미지 푸시·dispatch 자체가 안 됨 (배포 차단)
- 배포 후 헬스 실패 → deploy job 실패로 Actions에 빨간불. 수동 롤백:
  백엔드는 `sha-<이전커밋>` 태그로 compose 재기동, 프론트는 `.bak` 폴더 복원
- dispatch 유실(러너 꺼짐 등) → infra-settings에 `workflow_dispatch`(수동 트리거) 병행 제공

## 테스트 / 검증

- 웨이브별 완료 기준은 구현 계획에서 상세화. 최종 E2E: board-service에 트리비얼 커밋 push →
  Actions green → 로컬 `docker ps`에서 새 이미지 확인 → API 응답 확인.
  프론트: wiki에 문구 수정 push → 브라우저 새로고침으로 반영 확인
- 컨테이너화 자체 검증(웨이브 1): CI 없이 로컬에서 `docker compose up` 전체 스택 →
  기존 수동 E2E 체크리스트(SSO 로그인/로그아웃) 통과

## 진행 순서 (웨이브)

1. **백엔드 컨테이너화 + compose 통합** — Dockerfile 4개, docker 프로필, Keycloak 이중 URL,
   nginx proxy 변경. CI 없이 로컬 검증까지
2. **파이프라인 1개 완주** — runner 설치, infra-settings deploy.yml, board-service ci.yml.
   push → 자동 배포 E2E 확인
3. **백엔드 3개 복제** — auth/gateway/eureka에 ci.yml 복제
4. **프론트 3개 + DS** — dist 전용 폴더 전환, 프론트 ci.yml(DS checkout 포함), design-system CI

## 리스크

- Keycloak 이중 URL — 섹션 2로 해결하되, auth-server의 OIDC 클라이언트 설정이 예상과 다르면
  웨이브 1에서 가장 시간을 먹을 지점. realm-export의 redirect URI는 브라우저 기준이라 무수정
- Windows self-hosted runner에서 bash 스크립트 — deploy 스텝은 PowerShell 기준으로 작성
- gradle 컨테이너 빌드 시간 — 첫 빌드 느림. GHA gradle 캐시 + Docker layer 캐시(gha)로 완화
- DS 버전 하드코딩(`file:...-0.3.0.tgz`) — DS 버전 올리면 wiki/alm package.json도 같이 올려야
  CI가 깨지지 않음 (pack 산출물 파일명이 버전을 포함하므로)

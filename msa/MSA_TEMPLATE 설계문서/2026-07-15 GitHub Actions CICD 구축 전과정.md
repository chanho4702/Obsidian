# GitHub Actions CI/CD 구축 전과정 (2026-07-15 ~ )

> 스펙: [[2026-07-15-github-actions-cicd-design]] · 플랜: [[2026-07-15-github-actions-cicd]]
> 진행 원장: infra repo `.superpowers/sdd/progress.md`

## 0. 시작점과 목표

시작 시점의 상태:
- repo 9개로 분리 — 백엔드 4(auth-server, backend-server=board-service, gateway-server, eureka-server), 프론트 3(ch-stack-frontend=myFront, WIKI, ALM), design-system, infra-settings(=MSA_TEMPLATE)
- 배포는 전부 수동: 백엔드는 IntelliJ에서 호스트 실행, 프론트는 로컬 `vite build` 후 dist를 nginx 컨테이너에 볼륨 마운트. push해도 아무 일도 안 일어남

**목표: main에 push하면 테스트→빌드→검증을 거쳐 이 로컬 PC의 실행 환경까지 자동 반영.**

## 1. 브레인스토밍에서 확정한 6개 결정

| 갈림길 | 결정 | 근거 |
|---|---|---|
| 목표 범위 | CI + GHCR 이미지 푸시 + **로컬 자동 배포** | 검증만이 아니라 반영까지 |
| 대상 | 백엔드 4 + 프론트 3 전부 (DS는 CI만) | 한 번에 완성 |
| 로컬 배포 방식 | **Self-hosted runner** | 배포 결과가 Actions UI에 보임. Watchtower는 폴링 지연 + 컨테이너 교체만 가능 |
| 프론트 | **dist 교체 유지** (컨테이너화 안 함) | nginx 1개 = 단일 오리진 SSO 구조 무수정 |
| DS 의존성 | CI에서 design-system을 **옆에 checkout + build + pack** | `file:../design-system/artifacts/*.tgz` 의존을 package.json 무수정으로 만족 |
| 오케스트레이션 | **중앙 배포 repo(infra-settings) + repository_dispatch** | 개인 계정 runner는 repo 단위 등록 → 1곳 등록, 배포 로직·이력 집중. PAT 1개 |

## 2. 최종 아키텍처

```
[서비스 repo push(main)]
  ├─ 백엔드: gradle test → bootJar → docker build → GHCR push (latest + sha-<커밋>)
  └─ 프론트: (DS checkout+pack) → vitest → vite build → dist 아티팩트 업로드
           │ CI 성공 시
           ▼ repository_dispatch {service, sha, run_id, repo}
[infra-settings deploy.yml] ← self-hosted runner (이 PC)
  ├─ 백엔드: docker compose pull <svc> → up -d <svc> → healthy 대기 → smoke
  └─ 프론트: gh run download → C:\deploy\dist\<앱> 원자적 교체 → nginx restart
```

- 이미지: `ghcr.io/chanho4702/{eureka-server, gateway-server, auth-server, backend-server}` — `latest` + `sha-<커밋>` 이중 태깅(롤백용)
- compose 프로젝트명 `platform` 고정(runner가 어느 경로에서 체크아웃해도 같은 스택), 기존 DB 볼륨은 `keycloak_platform-db` 실명 핀으로 보존

## 3. Wave 1 — 백엔드 컨테이너화 (완료, 2026-07-15~17)

### 한 일
1. **Task 1~2**: 4개 repo에 런타임 전용 Dockerfile(`eclipse-temurin:24-jre`, `COPY build/libs/app.jar`) + `bootJar { archiveFileName = 'app.jar' }` 고정. 컨테이너 안에서 gradle을 돌리지 않는 이유: CI runner의 gradle 캐시 재사용 + CRLF gradlew 이슈 회피
2. **Task 3**: auth-server에 `docker` 프로필 (아래 사건 2로 최종 형태가 바뀜)
3. **Task 4**: compose에 4서비스 통합 — 의존 그래프 eureka(healthy)→gateway/auth/board, KC 헬스체크(`KC_HEALTH_ENABLED` + 관리포트 9000 `/health/ready`), nginx `proxy_pass http://gateway:8000` 전환, JWK 서명키 named volume 영속화(`PLATFORM_JWK_PATH=/data/auth-jwk.json`)
4. **Task 5**: 전체 스택 E2E — 8컨테이너 healthy, 스모크 4종(nginx/KC/JWKS체인/board API) 200, 브라우저 SSO 확인

### 사건 1 — 스테일 jar
로그인 리다이렉트가 `keycloak:8080`(브라우저 접근 불가)으로 나감. 원인: auth 이미지가 **Task 3 yml 커밋 이전에 빌드된 jar**를 담고 있었음(Task 2에서 bootJar → Task 3에서 yml만 수정, 재빌드 없음). `unzip -p app.jar`로 프로필 부재를 확인, 재빌드로 해소.
**교훈: Dockerfile이 빌드 산출물을 COPY하는 구조에서는 "yml 수정 → jar 재빌드 → 이미지 재빌드" 3단계가 전부 있어야 반영된다.**

### 사건 2 — Keycloak issuer 실측 (이 웨이브 최대 발견)
재시도하니 `/login?error`. DEBUG 로깅으로 잡은 원인: `[invalid_id_token] iss=http://localhost:8080/realms/sso-demo` — **KC는 토큰 iss를 (토큰 엔드포인트가 아니라) 브라우저 인가 요청의 호스트로 발급한다.** 설계 가정과 정반대.

프로퍼티로는 해결 불가함도 확인:
- Boot는 `provider.*.issuer-uri`가 있으면 **무조건** 기동 시 디스커버리 → `localhost:8080`은 컨테이너에서 접근 불가, `keycloak:8080`으로 디스커버리하면 metadata issuer 불일치로 기동 실패
- issuer-uri 프로퍼티는 지울 수도 없음 — `KeycloakLogoutClient`가 백채널 로그아웃 URL로 @Value 주입받음

최종 해법: **docker 프로필 전용 수동 `ClientRegistrationRepository` 빈** (`ContainerClientRegistrationConfig`, auth-server `1a8a88d`). 분리 수평(split-horizon) 계약:

| 용도 | 값 |
|---|---|
| 브라우저(authorization-uri) | `http://localhost:8080/realms/sso-demo/...` |
| 서버-서버(token/jwks/userinfo/백채널 로그아웃) | `http://keycloak:8080/realms/sso-demo/...` (`KEYCLOAK_ISSUER_URI` 프로퍼티) |
| ID 토큰 iss 검증 기준 | `http://localhost:8080/realms/sso-demo` (`KEYCLOAK_FRONT_ISSUER` env로 오버라이드 가능) |
| userNameAttribute | `sub` (디스커버리 기본과 동일) |

빈이 있으면 Boot 자동구성·디스커버리는 백오프. 단위 테스트 4개로 고정. 호스트 실행(기본 프로필)은 무변경.

## 4. Wave 2 — 파이프라인 완주 (진행 중, 2026-07-17~)

### Task 6 — 수동 셋업 (사용자 수행, 재현 절차)
1. **PAT 2개**: `DISPATCH_PAT`(fine-grained: repo 9개, Contents RW + Actions Read) / `GHCR_PULL_PAT`(classic: `read:packages` — GHCR 로그인은 classic만 지원)
2. `DISPATCH_PAT`를 8개 repo(서비스 7 + infra-settings)의 Actions secret으로 등록
3. runner: infra-settings → Settings → Actions → Runners → New self-hosted runner(Windows x64) → config.cmd
4. 로컬 1회: `docker login ghcr.io -u chanho4702` (비밀번호=GHCR_PULL_PAT) + `winget install GitHub.cli`

⚠ 현재 runner는 **서비스가 아니라 대화형 프로세스**(`C:\MSA_TEMPLATE\actions-runner`, run.cmd) — 그 터미널을 닫거나 재부팅하면 죽는다. 서비스 전환은 마감 때 재구성 예정. runner 디렉토리는 infra repo 안이라 `.gitignore`에 `/actions-runner/` 추가함.

### Task 7 — deploy.yml (완료)
`repository_dispatch(deploy)` + `workflow_dispatch`(수동/롤백용) 수신. service별 `concurrency` 직렬화. 백엔드 분기(pull→up→healthy 대기 120s)·프론트 분기(gh run download→`.bak` 백업 후 폴더 스왑→nginx restart)·smoke(nginx + JWKS 체인). 리뷰 픽스: 프론트 스텝 `$ErrorActionPreference='Stop'` + nginx restart 종료코드 검사(무음 스왑 실패 방지).

### Task 8 — board-service CI (진행 중)
`ci.yml`: PR/push에 test, main push에 한해 GHCR push(latest+sha) + dispatch. 첫 완주 관찰 중.

## 5. 남은 작업 (Wave 3~4)
- Task 9: eureka/gateway/auth에 ci.yml 복제 (이미지명·service 값만 치환)
- Task 10: nginx dist 마운트를 `C:\deploy\dist\{portal,wiki,alm}` 절대경로로 전환
- Task 11: 프론트 3 CI — wiki/alm은 DS를 옆에 checkout해 `pnpm pack`으로 tgz 재생성(`--no-frozen-lockfile` 필요: tgz integrity가 lock과 달라질 수 있음), myFront는 npm
- Task 12: design-system CI(typecheck/test/build만, 배포 없음)
- Task 13: 최종 E2E + whole-branch 리뷰 + runner 서비스 전환 + 원장/메모리 마감

## 6. 운영 메모 (구축 후 일상)
- **배포하려면**: 해당 repo main에 push. 그게 전부. Actions 탭에서 진행 확인
- **롤백(백엔드)**: infra-settings → Actions → Deploy → Run workflow → service + `tag: sha-<이전커밋>`
- **롤백(프론트)**: `C:\deploy\dist\<앱>.bak`을 원위치로 복원 (직전 1개 보존)
- **로컬 개발 모드는 그대로**: dev-up.ps1 + IntelliJ .run — 컨테이너 스택과 포트가 겹치니 동시엔 못 띄움(8000/8761/9000/9100). 컨테이너 스택을 내리려면 `infra/keycloak`에서 `docker compose down`
- **CI 실패 시 배포 안 됨**: dispatch 자체가 CI 성공 뒤에만 발신됨
- DS 버전 올릴 때: wiki/alm의 `file:...-x.y.z.tgz` 경로도 같이 올려야 CI가 안 깨짐

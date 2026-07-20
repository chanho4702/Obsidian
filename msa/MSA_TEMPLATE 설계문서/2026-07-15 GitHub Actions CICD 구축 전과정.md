# GitHub Actions CI/CD 구축 전과정 (2026-07-15 ~ 07-19 완료)

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

### Task 8 — board-service CI + 첫 완주 (완료)
`ci.yml`: PR/push에 test, main push에 한해 GHCR push(latest+sha) + dispatch. 첫 완주 과정에서 트러블슈팅 2건:

- **사건 3 — DISPATCH_PAT 권한**: dispatch 스텝만 "Repository not found" 실패. infra-settings가 비공개 repo라 PAT의 Repository access 누락이 원인 — PAT 수정으로 해소. 진단 팁: 공개 repo는 check-runs annotations API로 익명 에러 조회 가능
- **사건 4 — self-hosted runner에 pwsh 부재**: deploy job이 6초 만에 실패, runner `_diag` 로그에 `pwsh: command not found`. 클라우드 러너에는 PowerShell 7이 내장이지만 셀프호스트에는 없음 — `winget install Microsoft.PowerShell` + runner 재시작(PATH 재로딩)으로 해소

완주 실측: 빈 커밋 push → CI green → GHCR 이미지 → runner가 pull+재기동 → healthy + API 200.

## 4-5. Wave 3 — 백엔드 3개 복제 (완료)

Task 8의 ci.yml을 3개 repo에 복제(이미지명·service 값 + **eureka/gateway는 기본 브랜치 master라 트리거·if 4곳 치환**). 3개 서비스 전부 자동 재배포 실측.

- **사건 5(실버그) — .env 드리프트 재생성**: 배포 관찰 중 keycloak/postgres가 함께 재생성됨을 발견. runner 체크아웃에는 gitignore된 `.env`가 없어 `up -d`가 의존 서비스의 env 변화를 감지한 것. **deploy에 `--no-deps` 추가**로 대상 서비스만 조작하게 수정. DB 볼륨 영속이라 데이터 무손실

## 4-6. Wave 4 — 프론트 3개 + DS (완료)

- Task 10: nginx dist 마운트를 `C:\deploy\dist\{portal,wiki,alm}` 절대경로로 전환
- Task 11: 프론트 3 CI — wiki/alm은 DS를 옆에 checkout해 `pnpm pack`으로 tgz 재생성(`--no-frozen-lockfile`: tgz integrity가 lock과 달라질 수 있음), myFront는 npm. 3개 전부 dist 자동 교체(.bak 백업) + 3 URL 200 실측
- Task 12: design-system CI(typecheck/test/build만) green

## 5. 최종 리뷰 (whole-wave, 9 repo)

교차 repo 와이어 계약 전수 검증 통과(payload service↔deploy 맵↔compose 이름↔이미지↔태그 env, 아티팩트 dist, DS tgz 파일명, 브랜치 트리거). 반영한 픽스:
- **deploy.yml service 정확일치 검증 스텝** — 수동 실행(롤백 경로)의 오타가 "무서비스 pull" 또는 "배포 없이 green"이 되는 구멍 차단
- 프론트3+DS ci.yml에 `permissions: contents: read` (토큰 권한 축소)

수용된 후속 후보: dist swap try/catch 자가복구 + smoke에 /wiki/·/alm/ 추가, actions v5 일괄 범프, .dockerignore/USER/HEALTHCHECK, org-service 파이프라인 합류 시 deploy.yml 매핑.

## 6. 운영 메모 (구축 후 일상)
- **배포하려면**: 해당 repo main에 push. 그게 전부. Actions 탭에서 진행 확인
- **롤백(백엔드)**: infra-settings → Actions → Deploy → Run workflow → service + `tag: sha-<이전커밋>`
- **롤백(프론트)**: `C:\deploy\dist\<앱>.bak`을 원위치로 복원 (직전 1개 보존)
- **로컬 개발 모드는 그대로**: dev-up.ps1 + IntelliJ .run — 컨테이너 스택과 포트가 겹치니 동시엔 못 띄움(8000/8761/9000/9100). 컨테이너 스택을 내리려면 `infra/keycloak`에서 `docker compose down`
- **CI 실패 시 배포 안 됨**: dispatch 자체가 CI 성공 뒤에만 발신됨
- DS 버전 올릴 때: wiki/alm의 `file:...-x.y.z.tgz` 경로도 같이 올려야 CI가 안 깨짐

---

# 부록 A — Git/GitHub 설정 결정 상세 (무엇을, 왜, 어떻게)

> 노션 "배포" 페이지 부록과 동일 내용의 미러. 파이프라인 코드 밖에서 내린 깃/깃허브 설정 결정들의 전체 기록. 각 항목은 "선택지 → 선택 → 이유 → 실제 과정(트러블 포함)" 순서.

## A-1. Repo 토폴로지와 우산 repo의 .gitignore 컨벤션

**구조**: `C:\MSA_TEMPLATE`은 우산(umbrella) repo(=infra-settings)이면서, 그 **안에** 서비스 repo들이 디렉토리로 중첩돼 있다(auth-server, board-service, wiki-front...). 각자 독립 git repo이고, 우산 repo의 `.gitignore`가 그 디렉토리들을 전부 제외한다.

**왜 이 구조인가**: 로컬에서는 한 폴더에서 전체를 오가며 개발하고(상대경로 의존 — 예: wiki의 `file:../design-system/...`), 버전 관리·CI·배포는 repo별로 독립. 모노레포의 편의와 멀티레포의 독립 배포를 동시에 취하는 절충.

**이번 웨이브에서 추가된 ignore 2건**:
- `/actions-runner/` — runner를 우산 repo 안에 설치해버려서(설치 위치는 사용자 선택이었음) 커밋 오염 방지용으로 추가. ⚠ ignore는 커밋만 막지 `git clean -xdf`로부터는 못 지킨다 — 우산 repo에서 clean 금지
- `/platform-backend/` — 구현 서브에이전트가 지시 없이 추가하고 보고서에서 누락했던 사건. 리뷰어가 적발 → 확인해보니 실제로 사용자의 또 다른 중첩 repo(gRPC org-service)라 **컨벤션에 부합해 유지**하되, "지시 밖 변경 + 미보고"는 사건으로 기록. 교훈: diff와 보고서는 항상 대조된다

## A-2. PAT 2종 — 왜 2개이며 왜 서로 다른 종류인가

**배경 지식**: 워크플로우의 기본 `GITHUB_TOKEN`은 **자기 repo 한정**이다. 다른 repo의 워크플로우를 깨우거나(repository_dispatch), 다른 repo의 아티팩트를 받거나, 다른 비공개 repo를 checkout하려면 PAT가 필요하다.

### DISPATCH_PAT (fine-grained)

| 항목 | 값 | 이유 |
|---|---|---|
| 종류 | Fine-grained | 최소권한 원칙 — repo 9개만 지정 가능. classic은 `repo` scope가 계정 전체에 걸림 |
| Repository access | 9개 repo 전부 | 발신 7(백4+프론트3) + 수신 infra-settings + DS(프론트 CI가 checkout) |
| Contents: Read and write | dispatch 발신 인가 | repository_dispatch API가 대상 repo의 contents:write로 인가됨 (직관적이지 않은 부분!) |
| Actions: Read | 아티팩트 다운로드 | deploy가 `gh run download`로 프론트 repo의 dist 아티팩트를 크로스 repo로 받음 |

**실제 트러블(사건 ③)**: 첫 dispatch가 `Repository not found, OR token has insufficient permissions` — fine-grained PAT는 접근권 없는 repo를 403이 아니라 **404(존재 자체를 숨김)** 로 응답한다. infra-settings가 비공개 repo여서 Repository access 누락이 정확히 이 에러로 나타났다.

**진단법(재사용 가치 높음)**: `curl -H "Authorization: Bearer <PAT>" https://api.github.com/repos/chanho4702/infra-settings` → 200이면 토큰은 정상(secret에 든 값이 옛것), 404면 토큰의 repo 접근권 누락.

**흔한 함정 2개**: ① 권한 "수정"이 아니라 Regenerate를 눌러 **값이 바뀌었는데 secret은 옛 값** ② 토큰이 여러 개라 수정한 토큰 ≠ secret에 넣은 토큰.

### GHCR_PULL_PAT (classic)

- **왜 classic인가**: GHCR의 `docker login`은 사용자 네임스페이스 패키지에 대해 **classic PAT만 지원**한다. fine-grained로 통일하고 싶어도 불가.
- **scope는 `read:packages` 하나만** — 이 PC가 비공개 이미지를 pull하는 용도뿐이므로.
- 사용처: `docker login ghcr.io -u chanho4702` **로컬 1회**. 어디에도 등록 안 함(자격증명은 Windows 자격증명 관리자에 저장됨).
- 세션 함정: 비-TTY 환경에서는 `docker login`이 비밀번호 프롬프트를 못 띄운다 — 일반 터미널에서 하거나 `--password-stdin`.

## A-3. Secrets 배치 — 어디에 몇 개를 넣었나

`DISPATCH_PAT`를 **8개 repo**의 Actions secret으로 등록: 발신측 7개(백엔드 4 + 프론트 3) + **infra-settings**(deploy가 아티팩트 다운로드에 씀). design-system은 dispatch도 안 하고 남의 것을 받지도 않으므로 **등록 불필요** — 프론트 CI가 DS를 checkout할 때는 프론트 repo의 secret을 쓴다.

원칙: secret은 "그 repo의 워크플로우가 실제로 소비하는 곳"에만. 배포 안 하는 repo에 배포 토큰을 두지 않는다.

## A-4. Self-hosted runner — 등록 방식 선택과 함정들

**제약**: 개인 계정(무료)은 runner를 **repo 단위로만** 등록 가능. org-level 공유 불가.

**선택지 3개와 결정**:
1. ✅ **중앙 repo(infra-settings) 1곳 등록 + repository_dispatch 허브** — 배포 로직·이력·runner가 한 곳. PAT 1개로 해결
2. repo마다 등록(7번) — 등록 7번 + 배포 스크립트 7벌 + 동시 배포 충돌 관리
3. GitHub Organization 생성 후 repo 이전 — 가장 정석이지만 이전 작업이 큼

**실제 과정의 함정 3개**:
- **서비스 vs run.cmd**: config.cmd의 "run as service?"에서 N을 선택하면 대화형 프로세스 — 터미널 닫힘/재부팅에 죽는다. 우리가 실제로 겪은 상태. 전환 시 재등록이 필요하고, **서비스 계정은 반드시 본인 계정(chkim)으로** — Docker Desktop 파이프 접근, `docker login`·`gh` 자격증명이 전부 사용자 프로필에 묶여 있어 기본 NETWORK SERVICE로는 배포가 실패한다
- **runner에는 클라우드 러너의 소프트웨어가 없다(사건 ④)**: deploy가 `shell: pwsh`를 요구했는데 PC에는 PowerShell 5.1뿐 — `winget install Microsoft.PowerShell` 후 **runner 재시작 필수**(실행 중 프로세스는 시작 시점의 PATH 스냅샷을 물고 있다). `gh` CLI도 같은 이유로 별도 설치
- **설치 위치**: repo 밖(`C:\actions-runner`) 권장. repo 안이면 gitignore + clean 금지 규율이 추가로 필요해진다

## A-5. GHCR — 이미지 네이밍·인증·가시성

- **이미지 이름 = repo 이름 그대로** (`ghcr.io/chanho4702/backend-server` 등): 추적 가능성. GHCR은 소문자만 허용 — repo 이름이 이미 전부 소문자라 충돌 없음
- **CI에서 push는 기본 `GITHUB_TOKEN`으로 충분** (`permissions: packages: write` 선언만): 자기 계정 네임스페이스 패키지라 별도 PAT 불필요 — 시크릿 관리 부담 최소화
- **가시성 상속**: 워크플로우가 처음 push한 패키지는 **repo 가시성을 상속** — 공개 repo라 이미지도 공개로 생성됨. 노출 수준은 repo와 동일(이미지 내용물은 jar + dev 기본값뿐)이라 수용, 원하면 패키지 설정에서 Private 전환 가능
- **태그 전략 `latest` + `sha-<커밋>`**: latest는 일상 배포, sha는 롤백 앵커. 롤백은 컨테이너만 되돌리므로(GHCR latest 불변) 영구 롤백은 revert 커밋→CI

## A-6. 브랜치 현실 — main/master 혼재 대응

레포 조사 결과 **eureka-server·gateway-server는 기본 브랜치가 master**, 나머지 7개는 main이었다.

**선택**: 브랜치 통일(전부 main으로) 대신 **CI 쪽에서 흡수** — 두 repo의 ci.yml에서 트리거 `branches: [master]` + `if: github.ref == 'refs/heads/master'` 3곳, 총 4곳 치환.

**이유**: 기본 브랜치 변경은 GitHub 설정 + 로컬 추적 브랜치 + 열려있는 참조까지 건드리는 별도 작업 — CI/CD 웨이브의 범위가 아니다. 치환 4곳은 리뷰에서 전수 검증됐다. (통일하고 싶어지면 그때 ci.yml 4곳을 되돌리면 됨.)

부수 발견: board-service는 main 브랜치에 업스트림이 미설정 상태여서 첫 push에 `-u origin main` 필요했다.

## A-7. 커밋·푸시 규율 — CI 도입 전후로 의미가 달라진다

- **웨이브 전반(Wave 1)**: 태스크별 로컬 커밋만 하고 push 보류 — 로컬 E2E(Task 5) 통과 후 일괄 push. 검증 전 코드가 원격에 남지 않게
- **CI 도입 후(Wave 2~)**: **push = 배포 트리거**로 의미가 바뀐다. main에 올리는 순간 테스트→빌드→로컬 반영이 자동으로 굴러가므로, "완성 안 된 것은 main에 올리지 않는다"가 규율이 됨 (PR에서는 CI만 돌고 배포는 안 되니 작업 브랜치 활용 가능)
- **빈 커밋 트리거 기법**: `git commit --allow-empty -m "..."` → push — 코드 변경 없이 파이프라인 전 구간을 태울 때(E2E 검증, 플레이키 재확인). 파일 churn 없이 트리거만 만드는 표준 기법
- **스테일 jar 교훈(사건 ①)과 커밋 순서**: "jar를 COPY하는 Dockerfile" 구조에서는 설정 커밋 → **jar 재빌드 → 이미지 재빌드**까지 가야 반영. CI가 이 순서를 항상 강제해주는 것 자체가 CI 도입의 이유이기도 하다 — 사람은 이 순서를 까먹는다(실제로 까먹었다)

## A-8. repository_dispatch 계약 설계

- `event_type: deploy` 하나로 통일, 분기는 payload로: `{service, sha, run_id, repo}`
  - `service`: 배포 분기 키. **deploy.yml의 검증 스텝·맵·compose 서비스명과 정확 일치**해야 하는 계약값 — 최종 리뷰에서 substring 매칭의 구멍(오타→무동작 green)이 발견돼 정확일치 검증 스텝을 추가함
  - `run_id` + `repo`: 프론트 아티팩트를 `gh run download <run_id> -R <repo>`로 집어오기 위한 좌표
  - `sha`: 추적용(어느 커밋이 배포됐나)
- `workflow_dispatch`(수동 실행)를 **같은 워크플로우에 병행** — dispatch 유실 대비 + 롤백 진입점(tag 입력). env에서 `github.event.client_payload.* || inputs.*` 폴백 패턴으로 두 트리거를 한 코드로 처리
- `concurrency: deploy-<service>` — 같은 서비스 연속 push는 직렬화, 다른 서비스는 병행

## A-9. 워크플로우 permissions — 기본 토큰 권한 축소

- 백엔드 CI: `contents: read` + `packages: write` (GHCR push에 필요한 만큼만)
- 프론트·DS CI: `permissions: contents: read` — 처음엔 누락됐다가 **최종 리뷰에서 지적되어 추가**. 이 워크플로우들은 GITHUB_TOKEN을 checkout에만 쓰므로(크로스 repo 작업은 전부 DISPATCH_PAT) read면 충분
- 원칙: `permissions` 블록을 명시하지 않으면 repo 설정의 기본값(대개 광범위)이 적용된다 — 워크플로우마다 필요한 최소를 선언하는 게 정석

# 감사 보고서 — agent-service · 인프라/운영 · 문서 (2026-10-18)

범위: `C:\MSA_TEMPLATE`(infra-settings 리포), `agent-service`(미푸시 커밋 3개 포함), 서비스 간 문서 정합.
방법: 읽기 위주로 점검했다. 아래 읽기 전용 실측만 함께 했다(파일·git 상태 변경 없음).
- `docker stats`, `docker ps`, `docker images`
- 러너 컨테이너 안에서 TCP connect 프로브
- 인증 없는 nginx 2MB POST 프로브
- `C:\deploy\platform.env`에서 키 존재 여부만 확인(`grep -c`, 값은 출력하지 않음)

확정 설계 결정(Keycloak 리다이렉트, 백채널 로그아웃, 수동 ClientRegistration, PostgreSQL·Redis Streams, Alloy→Loki, +10000, Obsidian 전용 문서, ps1 BOM, common-starter, 단일 아티팩트 버전, MAIL_MODE·SEARCH_MODE)은 그대로 두는 것을 전제로 제안했다.

**정상으로 확인한 것**
- 추적 중인 `.ps1` 22개는 전부 UTF-8 BOM이다.
- DB, Redis, S3Mock, MinIO 콘솔, Grafana, docs 임포터는 127.0.0.1에만 바인딩돼 있다.
- deploy는 `platform.env`가 없으면 실패하고(fail-closed), dispatch에 sha가 없으면 거부하며, HTTP 헬스 게이트를 거친다.
- 백업 도구는 설계 품질이 높다(정합 검사, manifest 생성, ACL 제한).

---

## 요약 표 (심각도순)

| ID | 분류 | 심각도 | 한 줄 |
|---|---|---|---|
| O-01 | 보안 | Critical | LLM 워커(agent-runner)가 `host.docker.internal`로 호스트 루프백의 Postgres(5433)·Redis(16379)에 닿음 — **실측** |
| O-02 | 보안 | Critical | 운영 `platform.env`에 DB·KC admin·KC 서비스계정·MinIO·Grafana 비밀번호가 없어 compose 기본값이 적용됨. KC는 LAN(0.0.0.0:18080)에 열려 있음 |
| O-03 | 보안 | Critical | 러너 워커가 같은 uid로 돌아 `/proc/1/environ`에서 `agr_` 러너 토큰을 읽음 — **실측** |
| O-04 | 운영 | Critical | 운영 머신에서 백업이 한 번도 돌지 않음(출력 폴더·예약 작업 없음). 인벤토리에 `AGENT_CREDENTIAL_MASTER_KEY`·`platform-agent-work` 누락 |
| O-05 | 기능 결함 | Critical | nginx 일반 `/api` 경로의 본문 상한이 기본값 1MB → 위키·ALM 첨부(최대 20MB)·아바타(4MB)가 413 — **실측** |
| O-06 | 운영 | Important | 배포가 sha 태그를 프로세스 env로만 고정 → 이후 수동 `compose up`이 오래된 로컬 `latest`로 **조용히 다운그레이드** |
| O-07 | 운영 | Important | `dev-down.ps1`의 `docker compose down`이 같은 프로젝트(`platform`)인 **운영 스택 전체**를 내림 |
| O-08 | 성능·안정성 | Important | 어떤 컨테이너에도 메모리 제한·JVM 힙 플래그가 없음. 실측 합계 약 12.6GiB / 14.56GiB |
| O-09 | agent-service | Important | 킬 스위치가 인메모리(재기동 시 해제)이고 실행 중 run을 멈추지 않음 |
| O-10 | 보안 | Important | 배포판 Keycloak이 `start-dev`로 뜸. realm에 `admin/admin`(ADMIN)·`alice` 시드, 자가가입 허용, BFF 시크릿 하드코딩 |
| O-11 | 성능 | Important | nginx에 gzip·Cache-Control이 없음 → index.html 휴리스틱 캐시로 배포 뒤 청크 404·흰 화면 가능 |
| O-12 | 운영 | Important | Loki 보존 정책 없음, Loki·Grafana가 외부로 사용 통계 전송, 도커 로그 로테이션 없음 |
| O-13 | 운영 | Important | 배포 실패 시 자동 롤백 없음(백엔드: 깨진 컨테이너가 남음, 프론트: 스모크 실패해도 .bak 미복원) |
| O-14 | agent-service | Important | 운영 컨테이너의 하네스 번들이 조용히 빈 zip(`get_harness_bundle`도 빈 목록) |
| O-15 | agent-service | Important | 사무실 SSE(미푸시 066226d) — 단일 스레드 동기 전송, audit 신호 증폭 |
| O-16 | 문서 | Important | 배포된 agent-runner(AGP-69)가 README·INSTALL·infra/README·.env.example·smoke 어디에도 없음 |
| O-17 | 문서 | Important | 루트 README가 코드와 모순(ALM 통합 검색 있음, wiki Eureka 미등록). 토폴로지에 협업·에이전트·이관·러너 누락 |
| O-18 | 문서 | Important | agent-service CLAUDE.md가 1517줄 이력형이고 서두가 사실과 다름. README는 "도구 18종"(실제 24종) |
| O-19 | 문서 | Minor | 서비스 수 표기가 문서마다 다름(4·6·8·9·12). CLAUDE.md·AGENTS.md 목록에 alm-backend·collaboration-service 누락 |
| O-20 | 위생 | Minor | 루트 미추적 잡동사니 정리, 중첩 리포 `service-architecture/` gitignore 누락 |
| O-21 | 위생 | Minor | `.superpowers/sdd/progress.md`(559줄 작업 원장)가 추적되고 있음. 6MB 발표 바이너리 |
| O-22 | 위생 | Minor | `scripts/tests/*.test.ps1` 10개를 돌리는 CI가 없음 |
| O-23 | 보안·운영 | Minor | 이미지 태그가 떠다님(`postgres:16`·`redis:7-alpine`·`keycloak:26.0`), 미사용 이미지 누적 |
| O-24 | agent-service | Minor | run 목록·SSE에서 프로젝트 권한 없이 타 프로젝트 메타데이터가 노출, Dispatcher가 QUEUED 전량 조회 |
| O-25 | 운영 | Minor | nginx·loki·alloy·grafana에 healthcheck 없음, 프론트 배포 `SRC_REPO` 허용 목록 없음 |

---

## 상세

### O-01 · 보안 · Critical — 러너 격리망이 호스트 루프백을 막지 못한다

**근거**
- `infra/keycloak/docker-compose.yml:677-699`의 설계 의도: "agent-runner-net에만 붙고 postgres·redis가 보이지 않는다".
- 실측: `docker exec platform-agent-runner`에서 `host.docker.internal`(192.168.65.254)의 다음 포트가 **전부 OPEN**이었다.
  - `5433`(Postgres), `16379`(Redis), `19910`(docs 임포터), `9001`(MinIO 콘솔), `18080`(KC), `13000`(Grafana)
- 그쪽에 걸린 자격증명:
  - Postgres는 superuser `keycloak`이다(`docker-compose.yml:22-24`). O-02에 따라 비밀번호도 기본값 `keycloak`이다.
  - Redis는 `requirepass`가 비어 있다(실측 `CONFIG GET requirepass` → 빈 값).

**왜 문제인가**
- 러너는 이슈 본문(외부 입력)으로 프롬프트를 만드는 LLM이 임의 셸을 실행하는 곳이다.
- Docker Desktop은 `127.0.0.1` 게시 포트를 `host.docker.internal`로도 포워딩한다. 그래서 "루프백 전용"이 컨테이너에 대한 경계가 되지 못한다.
- 프롬프트 인젝션 한 번이면 다음이 가능해진다.
  - 9개 DB 전체 읽기·쓰기(keycloak realm 포함)와 `COPY … PROGRAM`
  - Redis Streams 이벤트 위조, 공동편집 1회 티켓 위조·삭제

**수정**
1. (S, 즉시) Postgres 비밀번호를 교체한다(INSTALL §9 절차). Redis에 `--requirepass ${REDIS_PASSWORD:?}`를 넣고 앱의 `REDIS_PASSWORD`를 배선한다.
2. (S) `agent-runner`에 `extra_hosts: ["host.docker.internal:127.0.0.1", "gateway.docker.internal:127.0.0.1"]`를 넣는다. 이름 해석만 막는 보조 수단이다.
3. (M, 근본) 러너망을 `internal: true`로 바꾸고, 송신 프록시 컨테이너(github·api.anthropic.com·npm만 허용) 한 개를 두 망에 걸친다. 또는 러너를 별도 VM·호스트로 옮긴다. 192.168.65.254 직접 접근은 2번으로 막히지 않으므로 3번이 필요하다.
4. (S) `agent-service/CLAUDE.md` §13 위협 모델에 이 경로를 명시한다.

### O-02 · 보안 · Critical — 운영 env에 핵심 비밀번호 미설정 + 기본값 가드 없음

**근거**
- `C:\deploy\platform.env`에서 아래 키의 존재 여부만 셌다(값은 보지 않음).

  | 키 | 설정된 줄 수 |
  |---|---|
  | `POSTGRES_PASSWORD` | 0 |
  | `KC_ADMIN_PASSWORD` | 0 |
  | `KC_ADMIN_CLIENT_SECRET` | 0 |
  | `MINIO_ROOT_PASSWORD` | 0 |
  | `GRAFANA_ADMIN_PASSWORD` | 0 |
  | `AGENT_CREDENTIAL_MASTER_KEY` | 1 |

- 그래서 compose 기본값이 그대로 쓰인다.
  - `keycloak`(`docker-compose.yml:24,47`)
  - `admin/admin`(`:42-43`)
  - `platform-admin-secret`(`:53,379`) — realm-management의 `manage-users` 권한을 가진 서비스 계정이다(`realm-export.json:93-96`).
  - `local-dev-secret`(`:136`), Grafana `admin`(`:993`)
- Keycloak은 `0.0.0.0:18080`으로 게시돼 있다(`docker ps` 실측, `:63`). admin 콘솔이 LAN에 열려 있다는 뜻이다.
- KC admin과 Grafana 비밀번호는 최초 기동 때만 적용된다. 그 뒤 콘솔에서 바꿨는지는 **확인 필요**하다. 서비스 계정 시크릿과 DB 비밀번호는 기본값이 확정이다.

**왜 문제인가**
- LAN 사용자 누구나 KC admin(기본값이면)으로 realm 전체를 탈취할 수 있다.
- `platform-admin` 시크릿만 알아도 전 사용자 비활성화나 비밀번호 리셋이 가능하다.

**수정**
- (S) 위 키를 즉시 생성해 설정하고 교체한다. KC 쪽은 kcadm으로 서비스 계정 시크릿을 재발급한다.
- (S) `deploy.yml`의 Deploy 스텝 앞과 `smoke-stack.ps1`에 **기본값 거부 프리플라이트**를 둔다. 해당 키가 없거나 알려진 기본값(`keycloak`, `admin`, `platform-admin-secret`, `local-dev-secret`)이면 exit 1로 막는다. 2026-09-13에 "platform.env 없으면 실패"로 바꾼 것과 같은 fail-closed 원칙이다.

### O-03 · 보안 · Critical — 러너 토큰이 워커에게 보인다

**근거**
- `agent-service/runner/Dockerfile:44-51`: 사용자가 `agent`(uid 10001) 하나뿐이고 `USER agent`이다.
- 러너 JVM과 claude 워커가 같은 uid로 돈다.
- 토큰은 env `RUNNER_TOKEN`으로 들어간다(`docker-compose.yml:691`).
- 실측: 워커 uid에서 `/proc/1/environ` 안의 `RUNNER_TOKEN`이 읽혔다(카운트 1).
- 같은 이유로 서비스 컨테이너에서는 워커 실행을 금지했다(`agent-service/CLAUDE.md:1137-1140`, `docker-compose.yml:659-662`). 그런데 러너에는 같은 구멍이 남아 있다.

**왜 문제인가**
- 탈취한 `agr_` 토큰으로 다른 프로젝트의 QUEUED SERVER run을 claim하면, 그 run의 LLM 키와 페르소나 run 토큰을 받는다.
- `RUNNER_CONCURRENCY>1`이면 옆 워커의 `/proc/<pid>/environ`에서 `ANTHROPIC_API_KEY`도 읽힌다.
- 워커가 `/work/runner.alive`를 직접 갱신해 HEALTHCHECK(`Dockerfile:61`)를 속일 수도 있다.

**수정 (M)**
- 워커는 별도 uid(예: 10002)로 낮춰 실행한다(`setpriv --reuid`).
- 러너 토큰은 env 대신 러너 uid만 읽을 수 있는 docker secret 파일(0400)로 준다.
- `/work/runner.alive`는 러너 uid 전용 디렉터리로 옮긴다.

### O-04 · 운영 · Critical — 백업이 실제로 돌지 않고, 새 키·볼륨이 인벤토리에 빠져 있다

**근거**
- 기본 출력 경로가 없다: `C:\backup\platform`(`scripts/backup-platform.ps1` param `OutDir`), `C:\backup\alm`(`backup-alm.ps1:46`). `ls /c/backup` 결과 "No such file".
- `Get-ScheduledTask`에 플랫폼 백업 작업이 없다.
- `infra/runbooks/index.md:3-5`도 "실제 백업이 만들어지는 것은 아니다"라고 스스로 경고한다.
- `infra/backup/inventory.json` `requiredSecretKeys`에 `AGENT_CREDENTIAL_MASTER_KEY`가 없다. 이 키를 잃으면 저장된 LLM 키 전체가 복호화되지 않는다(`docker-compose.yml:655-657`).
- `volumes`에 `platform-agent-work`가 없다. 계보 워크스페이스로, 재생성 뒤에도 남아야 한다고 compose `:693-695`가 말한다.

**왜 문제인가**
- 디스크 하나만 고장 나도 9개 DB, 첨부, JWK, 메일 계정이 전부 사라진다.

**수정**
- (S) 인벤토리에 위 키와 볼륨을 추가한다. `AGENT_PLATFORM_RUNNER_TOKEN`과 `KC_ADMIN_CLIENT_SECRET`도 검토한다.
- (M) 실행 체계를 만든다.
  - 정합(cold) 백업은 정지가 필요하므로, 야간 유지보수 창에 `stop writers → backup-platform -Execute -Quiesced → start` 래퍼를 작업 스케줄러에 등록한다.
  - 그 사이 간극은 `backup-alm.ps1` 같은 핫 `pg_dump`를 매일 돌려 메운다.
  - 결과(성공·실패)를 org 메일 허브로 알린다.
  - 외부 사본 경로를 하나 정한다.

### O-05 · 기능 결함 · Critical — nginx 1MB 본문 상한이 첨부 업로드를 막는다

**근거**
- `infra/nginx/default.conf:116-129`의 일반 `/(api|…)/` location에 `client_max_body_size`가 없다. 16m은 협업 경로 두 곳(`:58`, `:106`)에만 있다. nginx 기본값은 1m이다.
- 서비스 쪽 상한은 이렇다.
  - wiki `max-file-size ${WIKI_MAX_ATTACHMENT_MB:20}MB`(`wiki-backend/src/main/resources/application.yml:22`)
  - alm 20MB(`alm-backend/.../application.yml:29`)
  - org 아바타 4MB(`platform-backend/org-service/.../application.yml:27`)
- 실측(인증 없음):
  - `POST http://localhost/api/wiki/attachments-probe`에 2MB를 보내면 **413**(nginx가 끊음)
  - 같은 경로에 1바이트를 보내면 **401**(게이트웨이까지 도달)

**왜 문제인가**
- 1MB를 넘는 첨부, 이미지, 아바타가 통합 배포판(nginx 경유)에서 전부 실패한다. dev(Vite 프록시)에서는 재현되지 않아 놓치기 쉽다.

**수정 (S)**
- 일반 API location에 `client_max_body_size 25m;`를 넣는다. 서비스 상한 20MB에 멀티파트 오버헤드를 더한 값이다.
- 상한 값은 `.env`의 `*_MAX_ATTACHMENT_MB`와 함께 문서화한다.
- `smoke-stack.ps1`에 "2MB POST가 413이 아님(401)" 회귀 검사를 추가한다.

### O-06 · 운영 · Important — 이미지 태그 드리프트: 수동 `compose up`이 조용히 다운그레이드한다

**근거**
- `deploy.yml:120`은 `Set-Item env:<X>_TAG sha-…`로 태그를 **그 스텝의 프로세스 env에만** 둔다. pull도 sha 태그만 받는다(`:121`). 그래서 로컬 `latest`가 갱신되지 않는다.
- 실측:

  | 서비스 | 실행 중 | 로컬 `latest` |
  |---|---|---|
  | gateway | `sha-c2f3be2…`(f804ceb7f593) | 6b1c4b49746b(더 오래된 sha-960750e) |
  | org-service | sha 태그 | 3주 전 이미지 |
  | wiki-backend | sha 태그 | 3주 전 이미지 |
  | docs-backend | **태그 없는 dangling 이미지 537188516c42** | — |

- 태그 env 없이 compose를 다시 올리는 경로:
  - `scripts/dev-up-local.ps1:32`(`docker compose up -d`, 전체)
  - `scripts/search-up.ps1:224`(`up -d --no-deps gateway`)
  - `scripts/mail-up.ps1:198`(org-service 재기동 안내)
  - `INSTALL.md:497`(비밀번호 교체 시 앱 10개 재생성)

**왜 문제인가**
- 위 명령을 한 번 돌리면 실행 중인 서비스가 수 주 전 이미지로 되돌아간다. 헬스는 green이라 눈치채기 어렵다.

**수정**
- (S) deploy 성공 직후 `C:\deploy\image-tags.env`에 `GATEWAY_TAG=sha-…` 형식으로 갱신 기록하고, 모든 compose 호출이 `--env-file platform.env --env-file image-tags.env`를 쓰게 한다. 스크립트 3개와 INSTALL 예시를 함께 고친다.
- 대안: deploy에서 `docker tag <sha> <repo>:latest`로 로컬 latest를 맞춘다.

### O-07 · 운영 · Important — 개발 스크립트가 운영 스택을 내리고 다시 만든다

**근거**
- `scripts/dev-down.ps1:13`의 `docker compose down`은 `infra/keycloak`(프로젝트명 `platform`)에서 돈다. 배포 스택과 같은 프로젝트라 nginx와 앱 20개가 전부 내려간다.
- 주석(`:1`)은 "로컬 인프라 종료"라고만 설명한다.
- `dev-up-local.ps1:32`는 `--env-file` 없이 전체 `up -d`를 한다(O-06의 다운그레이드와 env 드리프트를 함께 유발).
- `dev-up.ps1:5`는 "postgres·keycloak·redis·nginx"만 띄운다고 적었지만, 실제로는 전체 스택을 띄운다(`:39`).

**수정 (S)**
- dev 스크립트는 백킹 서비스만 명시해 올린다: `up -d --no-deps postgres keycloak redis s3mock minio`.
- dev-down은 아무것도 내리지 않는 안내만 하거나, `-Force`일 때만 `stop <백킹>`을 한다.
- env 파일 탐색은 `search-up.ps1`의 규칙(platform.env 우선)을 재사용한다.

### O-08 · 성능·안정성 · Important — 메모리 제한·JVM 플래그 없음, 합계가 호스트 한계에 근접

**근거**
- compose 전체에 `mem_limit`·`deploy.resources`가 0건이다.
- Java 이미지 10개는 전부 `ENTRYPOINT ["java","-jar","app.jar"]`이다. 예: `gateway-server/Dockerfile:6`, `wiki-backend/Dockerfile:6`, `agent-service/Dockerfile:10`.
- 그래서 JVM 기본값인 호스트 RAM의 25%(약 3.6GB)가 컨테이너마다 최대 힙이 된다.
- 실측 `docker stats` 합계는 **약 12.6GiB / 14.56GiB**다.
  - Java 서비스 각각 450~970MiB, OpenSearch 1.35GiB
  - 보류 중인 migration-service도 618MiB, board도 500MiB를 쓴다.
- `INSTALL.md:60`은 "전체 합계 메모리는 실측 기록이 없다"고 적고 있다.

**수정 (M)**
- compose에 `x-java: &java` 앵커를 두고 `mem_limit`을 서비스별 512m~1g로 건다. 함께 `JAVA_TOOL_OPTIONS: "-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError -XX:+UseSerialGC"`를 준다(소형 서비스는 SerialGC가 RSS를 줄인다).
- `agent-runner`에도 `mem_limit`(claude CLI, git clone)을 건다.
- INSTALL 메모리 절에 실측표를 넣는다.
- 보류 중인 migration-service는 `profiles: ["migration"]`으로 옵션화하는 안을 사용자 결정으로 올린다. "배포본 유지" 지시가 있으므로 임의로 내리지 않는다.

### O-09 · agent-service · Important — 킬 스위치가 fail-open이다

**근거**
- `agent-service/src/main/java/.../budget/BudgetService.java:35`: `private volatile boolean killSwitchOn = false;`. 재기동하면 false가 된다(`CLAUDE.md:321-322`도 인정).
- heartbeat의 stop 목록은 run 상태만 본다(`.../runner/RunnerService.java:166-173`).
- 러너는 claim 응답의 `Budget`을 쓰지 않는다(`runner/src/main/java/com/platform/agentrunner/Protocol.java:44`).

**왜 문제인가**
- 비용 폭주를 멈추려고 켜도 진행 중인 워커는 timeout까지 계속 돈다.
- 배포(O-06) 한 번이면 차단이 풀린다.

**수정 (M)**
- 킬 스위치를 DB(설정 테이블)에 영속화한다.
- ON이면 heartbeat 응답에 RUNNING run 전부를 stop으로 내린다.
- 러너가 run별 상한을 CLI 인자로 강제한다.

### O-10 · 보안 · Important — 배포판 Keycloak의 개발 모드와 데모 시드

**근거**
- `docker-compose.yml:40`: `command: start-dev --import-realm`. `infra/runbooks/installation-cases.md:58`은 운영에서는 `start-dev`를 쓰지 말라고 하지만, 배포판 compose가 바로 이것이다.
- `infra/keycloak/realm-export.json`:
  - `:20` `registrationAllowed: true`
  - `:32` `"secret": "platform-bff-secret"`(하드코딩). `auth-server/src/main/resources/application.yml:31`의 기본값과 같다.
  - `:73-91` 사용자 `alice/alice`, `admin/admin`(realm ADMIN 롤) 시드
- `INSTALL.md:284`는 이 계정들을 안내만 하고, 삭제 절차는 없다.

**수정 (M)**
- 운영 프로필은 `start --optimized`로 띄우고 `KC_HTTP_ENABLED`·`KC_PROXY_HEADERS=xforwarded`를 둔다. 이미지 빌드 단계에서 `kc.sh build`를 한다. dev 오버레이만 `start-dev`를 유지한다.
- 데모 사용자는 `realm-export.dev-users.json` 같은 별도 파일이나 dev 전용 import로 분리한다.
- BFF 시크릿은 `$(env:OIDC_CLIENT_SECRET)`로 바꾸고 auth-server에도 배선한다.
- 자가가입은 제품 결정 사항이므로 사용자에게 확인한다.
- 리다이렉트 방식(확정 결정)은 바꾸지 않는다.

### O-11 · 성능 · Important — nginx 압축·캐시 정책 부재

**근거**
- `infra/nginx/default.conf` 전체에 `gzip`·`expires`·`add_header`가 0건이다.
- 실측 `curl -I http://localhost/`:
  - `Accept-Encoding: gzip`을 보내도 70961B가 원본 그대로 온다.
  - `Cache-Control`이 없다(Last-Modified·ETag만 있다).
  - `Server: nginx/1.27.5`로 버전이 노출된다.
- Cache-Control이 없으면 브라우저가 휴리스틱으로 index.html을 캐시한다(Last-Modified 경과 시간의 약 10%, 이 머신 기준 약 2일).
- 배포 스왑(`deploy.yml:204-206`)은 옛 해시 청크를 서빙 경로에서 지운다. 그런데 없는 `/wiki/assets/*.js`는 `try_files` 폴백으로 index.html이 응답된다. 결과는 MIME 오류와 흰 화면이다.

**수정 (S)**
- http 레벨: `gzip on; gzip_types text/css application/javascript application/json image/svg+xml; gzip_min_length 1024; server_tokens off;`
- 각 SPA:
  - `location ~* ^/(wiki|alm|docs)?/?assets/` → `expires 1y; add_header Cache-Control "public, immutable";`, 그리고 `try_files $uri =404;`(폴백 금지)
  - index.html → `add_header Cache-Control "no-cache";`
- 공통 보안 헤더 `X-Content-Type-Options nosniff`, `Referrer-Policy`, `X-Frame-Options SAMEORIGIN`을 추가한다.

### O-12 · 운영 · Important — 로그 저장소가 무한히 자라고, 외부로 통계를 보낸다

**근거**
- Loki는 이미지 기본 설정을 쓴다(`docker-compose.yml:954`). 실측 설정에는 `retention`·`compactor`가 없고, `analytics.reporting_enabled`가 주석 처리돼 있어 기본값 true가 적용된다. stats.grafana.org로 전송된다.
- Grafana는 `GF_ANALYTICS_REPORTING_ENABLED`가 미설정(기본 true)이다(`:986-993`).
- 어떤 서비스에도 `logging:` 옵션이 없다. 도커 json-file 로그가 로테이션 없이 쌓이는지는 **확인 필요**하다(daemon.json 기본값).

**왜 문제인가**
- 온프렘·폐쇄망 제품에서 외부로 연결을 시도한다.
- 디스크가 차면 Postgres를 포함한 전체가 멈춘다.

**수정 (S)**
- `infra/observability/loki-config.yaml`을 추가해 마운트한다: `limits_config.retention_period: 336h`, `compactor.retention_enabled: true`, `analytics.reporting_enabled: false`.
- Grafana에 `GF_ANALYTICS_REPORTING_ENABLED=false`, `GF_ANALYTICS_CHECK_FOR_UPDATES=false`를 준다.
- compose에 `x-logging: &logging {driver: json-file, options: {max-size: "20m", max-file: "5"}}`를 전 서비스에 적용한다.
- Alloy→Loki 구조(확정 결정)는 유지한다.

### O-13 · 운영 · Important — 자동 롤백 없음

**근거**
- 헬스 게이트가 실패하면 `exit 1`만 한다(`deploy.yml:144-145`). 새(깨진) 컨테이너가 그대로 남는다.
- 롤백은 수동 재실행으로 안내돼 있다(`INSTALL.md:477-478`).
- 프론트: `.bak`를 만들지만(`deploy.yml:204-206`), Smoke(`:211-232`)가 실패해도 복원하지 않는다.

**수정 (M)**
- 백엔드: `up` 전에 현재 컨테이너의 `.Config.Image`를 기록한다. 게이트가 실패하면 그 태그로 `up -d --no-deps`를 다시 하고 실패로 보고한다.
- 프론트: Smoke 실패 시 `cur↔bak`를 되돌리고 nginx를 재시작한다.
- 롤백한 태그를 O-06의 `image-tags.env`에 기록한다.

### O-14 · agent-service · Important — 운영 컨테이너의 하네스 번들이 비어 있다

**근거**
- `agent-service/src/main/resources/application.yml:76-77`의 기본값이 `C:\MSA_TEMPLATE\.claude`다.
- compose agent-service 블록(`docker-compose.yml:620-675`)에는 해당 env도 마운트도 없다.
- `HarnessBundleService.build`는 디렉터리가 없으면 오류 없이 빈 zip을 만든다(`.../runner/HarnessBundleService.java:140-142`).
- 결과적으로 SERVER run 워커는 플랫폼 규약 없이 돌고, 미푸시 bf7576f의 `get_harness_bundle`도 운영에서 빈 목록을 돌려준다.
- 반대로 dev에서는 gitignore된 마켓플레이스 스킬까지 묶어 배포한다.

**수정 (S)**
- `../../.claude`를 읽기 전용으로 마운트하고 env를 배선한다.
- 경로가 없으면 기동 시 WARN을 남기고 헬스에 반영한다.
- 허용 목록을 `agents/`와 `skills/{platform-*,product-*,code-review}`로 좁힌다(루트 `.gitignore`의 "자작분만 커밋" 규칙과 일치).

### O-15 · agent-service · Important — 사무실 SSE(미푸시 066226d)의 전송 모델

**근거**
- `.../office/stream/OfficeStreamHub.java:88-94`: 스케줄러 스레드 1개.
- `:138-148`: flush와 heartbeat를 순차적으로 동기 `emitter.send`한다. 느린 클라이언트 하나가 TCP 창이 찬 상태로 막히면 전체 heartbeat가 멈추고, 프록시 유휴 종료 뒤 재연결 폭주가 난다.
- 페르소나 감사 한 건마다 projectId null 신호가 전 구독자에게 간다(`:50-52`, 디바운스 200ms `:39`). office 재조회는 쿼리 약 10개다(`OfficeService.java:161-214`). 구독자 200명이면 초당 최대 약 1000쿼리다.
- 정상 확인: 정리 경로와 연결 상한(사용자당 5, 전체 200)은 있다.

**수정 (S)**
- 전송을 구독자별 비동기 executor로 분리하고 쓰기 타임아웃을 둔다.
- audit 사유의 디바운스를 1~2초로 늘린다.

### O-16 · 문서 · Important — 출시된 플랫폼 러너가 운영 문서에 없다

**근거**
- `grep -c agent-runner` 결과가 README.md 0, INSTALL.md 0, infra/README.md 0이다.
- `infra/keycloak/.env.example`에 다음이 없다: `AGENT_PLATFORM_RUNNER_TOKEN`, `AGENT_CREDENTIAL_MASTER_KEY`, `AGENT_RUNNER_CONCURRENCY`, `COMPOSE_PROFILES=agent-runner`.
- 반면 compose(`:655-699`)와 deploy(`:147-177`)는 이미 운영 중이다(실측: 러너 컨테이너 가동 중).
- `smoke-stack.ps1`에도 러너가 없다.

**수정 (S)**
- INSTALL §4(설치 옵션)에 "플랫폼 러너" 행을 추가한다. 켜기·끄기, 토큰 생성, 위협 모델(O-01·O-03) 링크를 넣는다.
- `.env.example`에 블록을 추가한다.
- infra/README 구성요소 표와 smoke 정적 검사에 넣는다.

### O-17 · 문서 · Important — 루트 README가 코드와 모순된다

**근거**
- ALM 통합 검색:
  - README `:66` "Wiki·ALM 통합 검색", `:72` "search-service ─gRPC→ alm-backend(:9121) 이슈 조달", `:129` "원본은 wiki-backend·alm-backend가 소유"
  - 반대 근거: `docker-compose.yml:784-786`("main의 search-service는 ALM_GRPC_HOST를 읽지 않는다"), CLAUDE.md 확정 결정(미머지 브랜치)
- wiki-backend 디스커버리:
  - README `:112` "wiki-backend는 docker에서 Eureka 등록을 끄고 *_SERVICE_URI 직결"
  - 반대 근거: compose `:240-241,433`(`WIKI_EUREKA_ENABLED: "true"`, `lb://wiki-backend`, 2노드)
  - README `:114` "*_SERVICE_URI 네 개 필수"도 틀렸다.
- 토폴로지 그림(`:46-80`)에 collaboration-service, agent-service, migration-service, agent-runner가 없다.
- infra 행의 포트 `:102`에 Grafana 3000이 적혀 있다(현재 루프백, 이 머신은 13000).

**수정 (S)**
- ALM 색인 줄을 삭제하거나 "(미구현)"으로 표시한다.
- 디스커버리 절을 "auth·board·wiki = Eureka, org·alm·agent·migration·search = DNS 직결"로 고친다.
- 토폴로지에 4개 컴포넌트를 추가한다.

### O-18 · 문서 · Important — agent-service 문서가 진입점 역할을 못 한다

**근거**
- `agent-service/CLAUDE.md`는 1517줄이고 §5~§15가 날짜별 작업 이력이다.
- 서두가 사실과 다르다: `:7-10` "MCP 관련 코드는 아직 없다", `:25-28` "PAT는 이후 태스크에서 구현".
- `agent-service/README.md:9`는 "도구 18종"이라고 적었다. 실제는 24종이다(`CLAUDE.md:199`, bf7576f). README 로드맵은 "P2 준비 중"이지만 실제로는 P4c까지 진행됐다.

**수정 (M)**
- CLAUDE.md 상단 80줄 이내에 현재 상태 요약을 다시 쓴다: 무엇을 하는지, 실행 위치 2종, 운영 스위치, 킬 스위치 한계, 문서 지도.
- 날짜별 이력은 Obsidian으로 옮긴다(확정 결정: 작업기록은 Obsidian). 리포에는 규약과 결정만 ADR식으로 남긴다.
- README의 도구 목록은 MCP 등록 코드에서 생성하거나, 개수 대신 링크로 바꾼다.

### O-19 · 문서 · Minor — 서비스 수·목록 표기 불일치

**근거**

| 위치 | 표기 |
|---|---|
| `CLAUDE.md:3`, `AGENTS.md:16` | "백엔드 8". alm-backend·collaboration-service 누락 |
| `infra/README.md:75` | "백엔드 9서비스" |
| `docker-compose.yml:199` | "백엔드 4서비스" |
| `scripts/dev-up.ps1:8,90` | "백엔드 4개" |
| `scripts/dev-up-local.ps1:12` | "백엔드 6개" |
| `README.md:39` | "12개 서비스(10 repo)" |

- `CLAUDE.md` 변경 이력은 2026-09-12에서 멈췄다. 공동편집, 러너(AGP-69), DB 분리 매뉴얼이 기록되지 않았다.

**수정 (S)**
- 서비스 목록의 정본은 README 컴포넌트 표 하나로 정한다.
- CLAUDE.md·AGENTS.md는 리포 목록만(alm-backend·collaboration-service 추가) 적고 숫자는 쓰지 않는다.
- 스크립트·compose 주석의 숫자는 지운다.
- 변경 이력에 위 3건을 추가한다.

### O-20 · 위생 · Minor — 루트 미추적 파일 판정

| 파일 | 판정 | 근거 |
|---|---|---|
| `debug.log` | **삭제** + `.gitignore`에 `debug.log` 추가 | 헤드리스 크롬 crashpad 오류 로그(09-12), 내용 무가치 |
| `nest.yml` | **삭제** | compose 중첩 기본값 `${A:-${B:-x}}` 실험용 busybox 파일. compose `:401-402` 검증 흔적 |
| `msa-template-architecture.html`(742KB), `*.visual-check.*.png`·`.html`·`.json`, `msa-template-fullstack-storage-v2.architecture.json` | **service-architecture 리포로 이동 또는 삭제** | 08-30 아키텍처 페이지 생성물과 시각 검증 캡처. infra 리포 대상 아님 |
| `service-architecture/` | **`.gitignore`에 `/service-architecture/` 추가** | 자체 `.git`이 있는 중첩 리포(GitHub Pages). 다른 중첩 리포는 전부 ignore됨 |

### O-21 · 위생 · Minor — 추적 중인 작업 원장과 바이너리

**근거**
- `.superpowers/sdd/progress.md`(559줄)가 추적되고 있다. 06월 태스크 원장이고 "우산, master" 같은 낡은 사실을 담고 있다. 작업기록은 Obsidian에만 둔다는 확정 결정과 충돌한다.
- `presentation/*.pdf`(4.6MB)와 `*.pptx`(1.3MB)가 infra 리포에 커밋돼 있다.

**수정 (S)**
- `git rm --cached .superpowers`를 하고 ignore에 추가한다. 내용은 보관함으로 옮긴다.
- 발표 산출물은 생성기(`presentation/src`)만 남긴다. 바이너리는 GitHub Release 첨부나 별도 리포로 옮긴다. 결정은 사용자에게 받는다.

### O-22 · 위생 · Minor — 스크립트 테스트가 CI에서 돌지 않는다

**근거**
- `.github/workflows/`에는 `deploy.yml` 하나뿐이다.
- `scripts/tests/`에 10개 파일이 있다(deploy-workflow, health-gate, split-db, backup 등).

**수정 (S)**
- `windows-latest`에서 `*.live-fixture.test.ps1`을 제외한 테스트를 push·PR마다 돌리는 `ci.yml`을 추가한다.
- deploy.yml을 바꾸면 deploy-workflow 테스트가 회귀를 잡도록 한다.

### O-23 · 보안·운영 · Minor — 떠다니는 태그와 오래된 이미지

**근거**
- 떠다니는 태그: `postgres:16`(`docker-compose.yml:5`), `redis:7-alpine`(`:83`), `keycloak:26.0`(`:37`, 26.0 라인은 오래됨), `nginx:1.27-alpine`(`:923`).
- `docker images`에 `<none>` 이미지가 다수 쌓여 있다.
- MinIO 커뮤니티 이미지는 배포 정책이 바뀌었다. 업데이트 경로를 **확인 필요**하다.

**수정 (S)**
- 마이너·패치까지 고정하거나 digest로 고정한다. 업데이트는 Renovate 같은 도구로 PR을 받는다.
- deploy 성공 뒤 `docker image prune -f --filter until=336h`를 돈다. 롤백용 sha 이미지 N개는 보존한다.

### O-24 · agent-service · Minor — 권한 범위·쿼리 상한

**근거**
- `GET /api/agent/runs`(미푸시 ea7a7ba, `.../run/RunController.java`)가 인증 사용자 누구에게나 전 프로젝트 run을 준다(issueKey·persona·model).
- SSE를 projectId 없이 구독하면 전 프로젝트 활동이 보인다(`OfficeStreamController.java:12-15`).
- `Dispatcher.java:77` `findByStatus(QUEUED)`가 전량 조회이고, 건마다 합계 쿼리 2개를 친다.
- 정상 확인: 페이지 크기 1~100 검증과 usage_ledger 인덱스(V3)는 있다.

**수정 (S)**
- 프로젝트 VIEWER 필터를 건다(org gRPC는 이미 배선됨).
- SSE의 projectId는 권한으로 검증한다.
- Dispatcher에 Pageable 상한을 두고, 틱 안에서 `allow()` 결과를 캐시한다.

### O-25 · 운영 · Minor — 헬스체크 공백과 입력 허용 목록

**근거**
- `nginx`(`docker-compose.yml:922-943`, 유일한 진입점), `loki`, `alloy`, `grafana`에 healthcheck가 없다. 사용자 진입점의 생사를 compose가 모른다.
- Java 서비스의 컨테이너 헬스는 TCP 연결만 본다(deploy 게이트만 HTTP).
- 프론트 배포의 `SRC_REPO`·`RUN_ID`가 payload를 그대로 쓴다(`deploy.yml:18-19,202`). DISPATCH_PAT 보유자라면 임의 리포의 dist를 배포할 수 있다.

**수정 (S)**
- nginx healthcheck를 추가한다: `wget -qO- http://127.0.0.1/ >/dev/null`.
- loki는 `/ready`, grafana는 `/api/health`를 쓴다.
- Validate 스텝에 `myfront→chanho4702/myFront`, `wiki|docs→chanho4702/WIKI`, `alm→chanho4702/ALM` 정확 일치 검사를 넣는다.

---

## 우선 실행 순서 제안

1. **오늘(S)**
   - O-02 비밀번호 설정과 기본값 거부 가드
   - O-01의 1·2단계(Redis 비밀번호, 러너 extra_hosts)
   - O-05 nginx 본문 상한
   - O-04 인벤토리 보완과 핫 `pg_dump` 일일 예약
2. **이번 주(S~M)**
   - O-06 태그 파일과 O-07 dev 스크립트를 함께 고친다(같은 원인).
   - O-03 러너 uid 분리
   - O-11 nginx 캐시·gzip
   - O-12 로그 보존·통계 끄기
   - O-16·O-17 문서
3. **다음(M)**
   - O-01의 3단계 송신 프록시
   - O-08 메모리 제한
   - O-09 킬 스위치 영속화
   - O-10 Keycloak 운영 모드
   - O-13 자동 롤백
   - O-18 agent-service 문서 재구성

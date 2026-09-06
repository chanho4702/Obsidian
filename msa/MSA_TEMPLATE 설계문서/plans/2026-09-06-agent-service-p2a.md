# agent-service P2a — 워커 + 무인 루프 구현 플랜

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans. Steps use checkbox (`- [ ]`) syntax.

**Goal:** 스케줄러가 AGP 보드에서 자동화 허용 이슈를 집어 헤드리스 Claude Code 워커(호스트 프로세스)에게 맡기고, 진행·게이트·비용이 플랫폼에 기록되는 무인 루프를 완성한다 — "밤에도 도는 팀"의 최소 실증.

**Architecture:** agent-service에 Run/Gate/UsageLedger 도메인(V3)과 디스패처(@Scheduled 루프)를 추가. 워커 = 러너 머신의 `claude -p` 서브프로세스(비-bare, per-run 임시 워크스페이스에 리포 클론 + 하네스 실체화, run-스코프 임시 PAT로 우리 MCP에 기록). 게이트는 D9(주차 후 기록 재시작). 컨테이너 격리·egress 제한은 P2b.

**Tech:** P1 스택 그대로 + ProcessBuilder, `--output-format json` 파싱(usage→원장), CLAUDE_CODE_OAUTH_TOKEN(도그푸딩)/ANTHROPIC_API_KEY(제품).

**Spec:** `specs/2026-09-05-agent-service-design.md` §3·§4·§7·§10.5 (P2a 슬라이스). **AGP 이슈:** 에픽 AGP-2, 하위 AGP-3~9(+AGP-37 라이더).

## Global Constraints (P1 것 전부 유지 + 추가)

- 워커는 **호스트 프로세스**(P2a 한정, 신뢰 리포만). 워크스페이스 루트 `${AGENT_WORK_DIR:C:\agent-work}` 하위 run별 디렉터리, run 종료 시 보존 기간 후 삭제.
- 워커 호출 계약(§10.5 고정): 비-bare `claude -p`, `--permission-mode dontAsk` + `--permission-prompts none` + `--allowedTools "Read,Edit,Write,Glob,Grep,Bash(git *),Bash(gradlew*),Bash(./gradlew*),Bash(npm *),Bash(pnpm *),Agent,mcp__agent-platform__*"`, `--strict-mcp-config` + 인라인 `--mcp-config`(run PAT), `--output-format json`, `--max-turns` 상한.
- run 토큰 = **임시 PAT 재사용**(PatService.issue, label `run:<runId>`, 디스패치 시 발급·종료 시 revoke). 새 인증 기구 금지.
- 마이그레이션: agentdb 다음 **V3**, alm-backend 다음 **V21**(현재 V20 — 착수 전 실측 재확인).
- 이 플랜의 작업 자체를 도그푸딩: 각 태스크 시작 시 해당 AGP 이슈 claim(지호), 완료 시 작업 보고서(위키)+링크 코멘트+done. MCP 호출은 프로드 `http://localhost/api/agent/mcp`.

---

### Task 1: agentdb V3 — Run·Gate·UsageLedger (AGP-4)

**Files:** `db/migration/V3__run_gate_ledger.sql`, `run/{Run,RunStatus,RunType,RunRepository}.java`, `run/{Gate,GateKind,GateRepository}.java`, `budget/{UsageLedger,UsageLedgerRepository}.java` + Flyway 정합 테스트 확장
**Interfaces (Produces):**
- `run` 테이블: id BIGSERIAL, type VARCHAR(20)(TASK만 P2a), issue_key VARCHAR(40), project_id BIGINT, persona_id BIGINT NOT NULL FK, trigger VARCHAR(20)(SCHEDULER/USER), status VARCHAR(20)(QUEUED/RUNNING/WAITING_APPROVAL/BLOCKED/DONE/FAILED/CANCELLED), harness_ref VARCHAR(200), workspace_path VARCHAR(400), session_id VARCHAR(80), pat_id BIGINT, attempt INT DEFAULT 1, error TEXT, started_at/ended_at timestamptz, created_at/updated_at timestamptz
- `gate`: id, run_id FK, kind VARCHAR(20)(MERGE/ESCALATION/PLAN), request TEXT, decided_by BIGINT, decision VARCHAR(10)(APPROVE/REJECT), requested_at/decided_at timestamptz
- `usage_ledger`: id, run_id FK, scope VARCHAR(20)(PROJECT/PLATFORM), scope_id VARCHAR(40), cost_usd NUMERIC(10,4), input_tokens BIGINT, output_tokens BIGINT, model VARCHAR(60), created_at timestamptz
- 상태 전이 메서드는 Run 엔티티에(불법 전이 IllegalState → 409 계약), `RunRepository.findByStatus`, 활성 run 카운트 쿼리
**Steps:** TDD — Testcontainers 정합(RED: 엔티티 없음) → V3 SQL(전부 timestamptz!) → 엔티티/전이 단위테스트(합법·불법 전이) → GREEN → 커밋

### Task 2: run MCP 도구 + run 토큰 (AGP-9)

**Files:** `tools/RunTools.java`, `run/RunTokenService.java`(PatService 래핑: issue/revoke, personaId 바인딩), PatToken에 `runId` nullable 컬럼은 **추가하지 않음** — label 규약(`run:<id>`)으로 식별
**Produces(MCP 도구 3):** `report_progress(runId, message)` → run 검증(RUNNING + 호출 페르소나 일치) 후 이슈에 진행 코멘트 + 감사 / `request_gate(runId, kind, request)` → Gate 생성 + run→WAITING_APPROVAL + 이슈 코멘트("⏸ 승인 대기: ...") + 워커에게 "저장 후 종료하라" 텍스트 반환 / `report_result(runId, status(DONE|FAILED|BLOCKED), summary)` → run 종결 기록. 스냅샷 테스트 18→21.
**Steps:** TDD(도구별 상태 검증·타 페르소나 거부) → 구현 → 스냅샷 갱신 → 커밋

### Task 3: 하네스 실체화 + 워커 런처 (AGP-3)

**Files:** `worker/{HarnessMaterializer,WorkerLauncher,WorkerResult}.java`, `worker/ClaudeCliRunner.java`, application.yml `platform.agent.worker.*`(work-dir, harness-bundle-path `${HARNESS_BUNDLE:C:\MSA_TEMPLATE\.claude}`, claude-bin, max-turns, timeout-minutes, allowed-tools)
**Produces:**
- `HarnessMaterializer.materialize(Path workspace)` — harness-bundle-path의 agents/skills + 루트 CLAUDE.md·AGENTS.md를 워크스페이스 `.claude/`·파일로 복사(리포 자체에 이미 있으면 리포 것 우선 — 덮어쓰지 않음)
- `WorkerLauncher.launch(Run, issueContext) → WorkerResult(exitCode, resultText, sessionId, costUsd, tokensIn/Out, model)` — 워크스페이스 생성 → `gh repo clone`(이슈 라벨/프로젝트→리포 매핑 설정 `platform.agent.worker.repos`) → 실체화 → run PAT 발급 → ProcessBuilder로 `claude -p "<프롬프트: 이슈 키·제목·본문 + 규약(툴 프로토콜·작업 보고서 의무·report_result 필수)>"` (환경: CLAUDE_CODE_OAUTH_TOKEN 또는 ANTHROPIC_API_KEY 패스스루) → stdout JSON 파싱 → PAT revoke
- 타임아웃: 프로세스 destroy + run FAILED("시간 초과")
**Steps:** HarnessMaterializer TDD(복사·리포 우선) → ClaudeCliRunner는 명령 조립 단위테스트(실행은 페이크 실행기 인터페이스로 — `CommandExecutor` 추상화, 테스트는 페이크) → 통합은 Task 7 E2E에서 실프로세스 → 커밋

### Task 4: 디스패처·스케줄러 + 자동화 정책 (AGP-5)

**Files:** `run/{Dispatcher,RunService}.java`, `run/AutomationPolicy.java`, application.yml `platform.agent.scheduler.*`(enabled 기본 false!, interval, max-concurrent-per-project 1, max-concurrent-global 2)
**Produces:** @Scheduled tick → enabled·킬스위치·예산 확인 → `search_issues` 상당(AlmClient: status=todo, label=`auto`) → 활성 run 없는 이슈 선착 → Run 생성(QUEUED→RUNNING) + claim_issue(페르소나) → WorkerLauncher 비동기 실행(@Async 풀) → 결과 처리: DONE→이슈 코멘트+원장, FAILED→attempt<3이면 재큐, 아니면 BLOCKED+에스컬레이션 코멘트. `POST /api/agent/runs/{id}/cancel`(ADMIN).
**Steps:** RunService 상태 전이·재시도 TDD(WorkerLauncher는 목) → Dispatcher 정책 테스트(동시성 상한·label 필터) → 커밋

### Task 5: 게이트 승인·재개 + 예산·킬 스위치 (AGP-6, AGP-7)

**Files:** `run/GateController.java`, `budget/{BudgetService,BudgetController}.java`, application.yml `platform.agent.budget.*`(monthly-usd-cap, per-run-usd-cap)
**Produces:** `GET /api/agent/gates?pending`(인증) / `POST /api/agent/gates/{id}/approve|reject`(ADMIN) — approve 시 새 Run(attempt+1, 같은 이슈, 프롬프트에 "게이트 승인됨 — 이슈 기록 읽고 이어서") 큐잉, reject 시 run CANCELLED+코멘트. `BudgetService.check(projectId)` 디스패처 훅 + 원장 적재 + `POST /api/agent/kill-switch`(ADMIN, 전역 enabled=false). 
**Steps:** TDD(승인→재큐, 예산 초과→디스패치 스킵+1회 경고 코멘트) → 커밋

### Task 6: git 연결 — ALM 원격링크 + 커밋 파서 (AGP-8, 리포: alm-backend + agent-service)

**alm-backend:** `V21__issue_web_link.sql`(id, issue_id FK, url VARCHAR(500), title VARCHAR(200), kind VARCHAR(20)(PR/COMMIT/WEB), created_by, created_at timestamptz) + `collab` REST: GET/POST `/api/alm/issues/{id}/web-links`, DELETE `/api/alm/web-links/{id}`(EDIT 권한) + 테스트. 커밋 (푸시는 배포 게이트 통과 후).
**agent-service:** `link_pr` 도구 승격 — 코멘트 대신 web-link POST(+제목). 워커 종료 처리에 커밋 파서: 워크스페이스 `git log --format="%H %s" origin/main..HEAD` 파싱 → `AGP-\d+` 키 매칭 → 각 이슈에 COMMIT web-link. AlmClient 메서드+테스트.
**Steps:** alm 먼저(TDD) → agent-service 소비(TDD) → 커밋 각각

### Task 7: E2E — 진짜 무인 루프 실증 (AGP-2 수용 게이트)

**산출:** 프로드 스택에서: AGP-24("요청 DTO @Size 정합" — 소형·자가수정 가능 이슈)에 label `auto` 부착 → 스케줄러 enabled → 워커가 스스로: claim→구현(agent-service 리포 클론·코드 수정·테스트)→작업 보고서(위키)→link(커밋 파서)→report_result DONE. 사람 개입 0. 검증: 이슈 타임라인·보고서·원장 row·(가능하면) 실제 커밋 품질 육안. 실패 시 원인 리포트가 곧 다음 백로그.
**주의:** 첫 실행은 `--max-turns` 보수적(예: 80), global 동시성 1, 예산 per-run cap $5. CLAUDE_CODE_OAUTH_TOKEN은 사용자에게 `claude setup-token` 1회 요청(사용자 액션).

### Task 8: 문서·마감

CLAUDE.md(agent-service)에 워커 계약·스케줄러 운영(enabled 플래그·킬 스위치)·게이트 절차 추가. compose에 스케줄러 env 기본 off 명시. 각 AGP 이슈 done+보고서 확인. 스펙 변경 이력 갱신.

## Self-Review
- AGP-3~9 전부 태스크에 매핑(3→T3, 4→T1, 5→T4, 6→T5, 7→T5, 8→T6, 9→T2) ✓. AGP-37(update_issue 도구)은 이 플랜 밖 — Epic D에서 별도.
- 인터페이스: RunTokenService↔PatService(P1), WorkerLauncher↔RunService, BudgetService↔Dispatcher 시그니처 명시 ✓. V3 timestamptz ✓. 스냅샷 18→21 ✓.
- 위험 명시: 워커 실품질(T7이 측정), Windows 프로세스 관리(destroy 트리), OAuth 토큰 만료(1년 — 운영 노트).

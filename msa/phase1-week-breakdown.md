---
tags: [msa, 템플릿, phase1]
status: 진행중 (2026-07-09 확인 — W5 상당 완료 + 계획에 없던 유레카 추가, W4/W6/W7 미착수)
기간: 2026-07-05 ~ 2026-08-02
관련: "[[msa-roadmap]]"
이전: "[[phase0-week-breakdown]]"
---

# Phase 1 — MSA 템플릿 골격 (행동 단위)

> 한 칸 = 1~3시간 + 미니 DoD. 매주 마지막 칸이 주간 DoD.
> 목표: **새 서비스 하나 추가하면 바로 붙는 구조.** Phase 0의 Keycloak이 여기 compose에 들어간다.

## W4 (7/5~7/11) — 멀티모듈 + 공통 처리
- [ ] **Gradle 멀티모듈 골격** — `settings.gradle`로 starter-core/security/logging/domain 분리
	- DoD: 빈 4개 모듈이 빌드 통과
- [ ] **공통 응답 래퍼** — `ApiResponse<T>` (starter-core)
	- DoD: 컨트롤러가 ApiResponse 형태로 응답
- [ ] **ErrorCode enum** — 코드/메시지/HTTP status 매핑
	- DoD: ErrorCode 하나로 일관된 에러 바디 생성
- [ ] **GlobalExceptionHandler** — `@RestControllerAdvice`
	- DoD: 임의 예외가 표준 에러 응답으로 변환
- [ ] **BaseEntity** — createdAt/updatedAt 감사 필드 (starter-domain)
	- DoD: 새 엔티티가 감사 필드 자동 채움
- [ ] **(주간 DoD) 새 서비스 스캐폴드** — 더미 서비스에 starter-* 의존성만 추가
	- DoD: 새 서비스에서 공통 응답/예외가 그대로 동작

## W5 (7/12~7/18) — Gateway 라우팅 + 인증 필터
- [x] **Gateway 부트스트랩** — Spring Cloud Gateway 기동 — `MSA_TEMPLATE 정리/05, 10` (gateway-server :8000)
	- DoD: Gateway가 뜸
- [x] **라우팅 규칙** — path 기반으로 2개 서비스 분기 — 실제론 6개 라우트(auth 4개 + board 1개 + jwks)로 확장 구현
	- DoD: `/a/**`, `/b/**` 가 각 서비스로 전달
- [x] **인증 필터** — JWT 검증 후 통과/차단 — `.oauth2ResourceServer(jwt)` + JWKS(`MSA_TEMPLATE 정리/10` ②③④)
	- DoD: 토큰 없으면 Gateway에서 401
- [~] **사용자 정보 전파** — 코드 확인 완료(2026-07-09): 별도 `X-User-Id`/`X-Role` 헤더 주입 **없음**. Gateway는 원본 `Authorization` 헤더를 그대로 패스스루하고, 각 서비스가 JWKS로 직접 재검증해 claim을 꺼냄(`gateway-server` README 명시, `board-service` 구조와 일치) — DoD 문구(헤더 전달)와는 다른 방식이지만 "서비스가 사용자 식별" 목적은 달성
	- DoD: 서비스가 헤더에서 사용자 식별 → (실제로는 토큰 자체 재검증으로 식별, 헤더 미사용)
- [x] **traceId 발급** — Gateway에서 생성·헤더 주입 — `RequestLoggingFilter`, `X-Request-Id` 검증+재발급(`MSA_TEMPLATE 정리/10` ⑥)
	- DoD: 응답 헤더에 traceId 존재
- [x] **(주간 DoD)** — 2개 서비스 라우팅 + 토큰·traceId 전파 확인 — E2E 통과(`MSA_TEMPLATE 정리/09` E2E 검증 결과)

> [!note] 계획에 없던 추가: 유레카 서비스 디스커버리
> W5 직후 정적 URI 라우팅의 한계(주소 변경 시 재기동)를 만나 **유레카 서비스 디스커버리(lb://)**를 도입·완료(2026-07-05, E2E 검증 포함) → [[../MSA_TEMPLATE 정리/09 유레카 서비스 디스커버리 (2026-07-05)|09 유레카 서비스 디스커버리]]. 원래 로드맵엔 없던 항목이라 W5/W6 사이 별도 작업으로 취급.

## W6 (7/19~7/25) — 서비스 간 통신
- [ ] **클라이언트 선택** — OpenFeign vs WebClient
	- DoD: 선택 근거 3줄 노트
- [ ] **호출 구현** — A→B, 토큰 전파 포함
	- DoD: A가 B를 호출 성공
- [ ] **타임아웃/에러 매핑** — 연결·읽기 타임아웃, B 에러→표준 응답
	- DoD: B 500 → A에서 표준 에러로 전파
- [ ] **실패 시나리오** — B 내려놓고 호출
	- DoD: A가 스택트레이스 노출 없이 깔끔히 실패
- [ ] **(주간 DoD)** — A→B 호출 + B 에러의 표준 전파

## W7 (7/26~8/2) — Docker Compose 1차 완성 ★ + RBAC
- [ ] **compose 스켈레톤** — Nginx, Gateway, Keycloak, MySQL, Redis, 서비스2개
	- DoD: 모든 컨테이너 healthy
- [ ] **Prometheus + Grafana 추가** — 스크랩만(대시보드는 Phase 3)
	- DoD: Grafana에서 타겟 UP
- [ ] **시크릿 .env 정리** — 하드코딩 제거(Vault는 Phase 3)
	- DoD: 비밀값이 .env로 주입됨
- [ ] **RBAC 기본** — 역할 기반 접근 엔드포인트 1개
	- DoD: ADMIN만 접근, USER는 403
- [ ] **(주간 DoD) `docker compose up` 한 방**
	- DoD: 전체 스택 기동 + 로그인→Gateway→서비스 end-to-end 동작

## Phase 1 종료 기준
- [ ] `docker compose up` 으로 전체 스택 기동 (= 1차 목표)
- [ ] 새 서비스 추가가 starter-* 의존성 + 라우팅 등록으로 끝남
- [ ] 다음 → [[phase1.5-cicd]] (CI 도입) → [[phase2-week-breakdown]]

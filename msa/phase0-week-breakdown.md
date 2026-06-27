---
tags: [msa, 인증, phase0]
status: 진행중
기간: 2026-06-15 ~ 2026-07-05
관련: "[[msa-roadmap]]"
---

# Phase 0 — 인증 마무리 (행동 단위 쪼개기)

> [!tip] 쪼개기 규칙
> 한 칸 = 한 번 앉아 끝낼 수 있는 크기(1~3시간) + 각자 **미니 DoD**.
> 매주 마지막 칸이 곧 그 주의 **주간 DoD**다.
> 토픽 "이해"는 한 칸을 따로 차지하지 않고, 행동 직전에 필요한 만큼만 끼워 넣는다.

세션·JWT 직접 구현은 완료. 남은 건 **OAuth2/OIDC → Keycloak → Gateway 검증**.
직접 구현한 JWT는 Keycloak 도입 후 버려질 코드 — 깊이 조절할 것.

---

## W1 (6/15~6/21) — OAuth2/OIDC 흐름 잡기 + 기존 JWT 정리

- [ ] **OAuth2 4 grant type 비교** — 각 grant이 언제 쓰이는지 한 문단씩
	- DoD: Authorization Code를 왜 웹/SPA에서 쓰는지 1~2문장으로 적음
- [ ] **Authorization Code + PKCE 흐름 그리기** — 클라이언트/인가서버/리소스서버 요청 순서
	- DoD: code challenge·verifier 포함한 흐름 다이어그램 1장
- [ ] **OIDC가 OAuth2에 더하는 것** — ID token vs access token, claims
	- DoD: 두 토큰의 용도 차이를 2문장으로 적음
- [ ] **토큰 보안 위협 정리** — 탈취/만료/refresh 로테이션, 저장 위치(httpOnly 쿠키 vs localStorage)
	- DoD: "access는 짧게, refresh는 안전하게"의 근거를 적음
- [ ] **기존 JWT 코드 정리** — 곧 교체될 것이므로 검증 로직을 한 곳에 모으기만, 새 기능 추가 금지
	- DoD: 교체 지점(Keycloak로 바뀔 부분)이 한눈에 보임
- [ ] **(주간 DoD) 흐름 재현** — 화이트보드에 막힘없이 그리기
	- DoD: Authorization Code + PKCE 전체 흐름을 보지 않고 그림

---

## W2 (6/22~6/28) — Keycloak 띄우고 토큰 발급까지

- [ ] **Keycloak 컨테이너 띄우기** — compose에 추가, admin 콘솔 접속
	- DoD: 콘솔 로그인 성공
- [ ] **Realm 만들기** — Realm/Client/Role/Mapper 관계 30분 정리 후 실습 Realm 1개
	- DoD: 내 Realm이 콘솔에 보임
- [ ] **Client 등록** — confidential vs public, redirect URI 설정
	- DoD: client secret 확보(또는 public client 설정 완료)
- [ ] **유저 + Role** — 유저 1명, USER/ADMIN 역할 매핑
	- DoD: 유저에 role이 부여됨
- [ ] **토큰 받아보기** — Postman으로 token endpoint 호출
	- DoD: access token 문자열 획득
- [ ] **(주간 DoD) 토큰 까보기** — jwt.io로 디코딩, `realm_access.roles` 확인
	- DoD: 내가 부여한 role이 claim에 보임

---

## W3 (6/29~7/5) — Spring 리소스 서버 + Gateway 검증

- [ ] **Resource server 연결** — `issuer-uri`로 Keycloak 연동
	- DoD: 부팅 로그에서 JWKS 로드 성공
- [ ] **보호된 엔드포인트 1개** — 인증 필요 API 작성
	- DoD: 토큰 없이 호출 시 401
- [ ] **유효 토큰 통과 확인** — W2에서 받은 토큰으로 호출
	- DoD: 200 + principal에서 사용자 확인
- [ ] **role 기반 접근** — `@PreAuthorize` 등으로 ADMIN 전용 API 1개
	- DoD: USER 토큰으로 ADMIN API 호출 시 403
- [ ] **Gateway JWT 검증** — Gateway에서 1차 검증 + 서비스로 토큰 전파
	- DoD: Gateway 통과한 요청만 서비스 도달
- [ ] **(주간 DoD) end-to-end 점검** — 로그인→토큰→Gateway→서비스
	- DoD: 토큰 없으면 Gateway에서 막히고, 유효하면 서비스까지 도달 + role 검사 동작

---

## Phase 0 종료 기준
- [ ] Gateway에서 Keycloak 토큰 검증 통과/실패가 의도대로 동작
- [ ] role 기반 접근 제어(USER/ADMIN)가 한 서비스에서 검증됨
- [ ] 다음 단계 → [[msa-roadmap#Phase 1 — MSA 템플릿 골격 W4–7 7 5 8 2|Phase 1: MSA 템플릿 골격]]

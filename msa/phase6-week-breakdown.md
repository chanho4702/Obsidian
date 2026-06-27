---
tags: [msa, spring-ai, rag, phase6]
status: 대기
기간: 2026-12-13 ~ 2027-01-03
관련: "[[msa-roadmap]]"
이전: "[[phase5-week-breakdown]]"
---

# Phase 6 — Spring AI (행동 단위)

> 한 칸 = 1~3시간 + 미니 DoD. **마지막**에. 기초 인프라 없이 AI부터 붙이면 산으로 간다.

## W27 (12/13~12/19) — 기본 + 연동
- [ ] **Spring AI 의존성/설정**
	- DoD: ChatClient 기동
- [ ] **LLM 연동** — Claude/OpenAI
	- DoD: 프롬프트 → 응답 동작
- [ ] **AI Gateway 구조** — 라우팅 + 키 관리(Vault)
	- DoD: AI 호출이 게이트웨이 경유
- [ ] **문서 요약 엔드포인트**
	- DoD: 요약 API 동작
- [ ] **(주간 DoD)** — 요약 API 동작

## W28 (12/20~1/3) — RAG
- [ ] **임베딩** — 문서 청킹 + 임베딩
	- DoD: 벡터 생성
- [ ] **Vector DB** — 저장/검색
	- DoD: 유사도 검색 동작
- [ ] **RAG 결합** — 검색 + 생성
	- DoD: 문서 기반 답변 생성
- [ ] **(주간 DoD)** — 문서 기반 Q&A 동작

## Phase 6 종료 기준
- [ ] RAG 기반 문서 Q&A 동작
- [ ] 전체 템플릿 완성 → [[msa-roadmap]] 체크포인트 전부 닫기

---

## 전체 완주 체크
- [ ] Phase 0 — 인증 (Gateway 토큰 검증)
- [ ] Phase 1 — 템플릿 골격 (`docker compose up`)
- [ ] Phase 2 — DB 성능 (인덱스/N+1)
- [ ] Phase 3 — 관측 가능성 (분산 추적)
- [ ] Phase 4 — 이벤트 (Outbox/DLQ)
- [ ] Phase 5 — 배포 (K8s/ArgoCD)
- [ ] Phase 6 — Spring AI (RAG)

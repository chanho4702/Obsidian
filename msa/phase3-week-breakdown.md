---
tags: [msa, 운영, 관측가능성, phase3]
status: 대기
기간: 2026-09-06 ~ 2026-10-04
관련: "[[msa-roadmap]]"
이전: "[[phase2-week-breakdown]]"
---

# Phase 3 — 운영 인프라 (행동 단위)

> 한 칸 = 1~3시간 + 미니 DoD. 여기서 템플릿이 **장난감 → 실무형**으로 바뀐다. 관측 가능성이 핵심.

## W13 (9/6~9/12) — 메트릭
- [ ] **Actuator 노출** — `/actuator/prometheus`
	- DoD: 메트릭 엔드포인트 노출
- [ ] **Prometheus 스크랩** — 서비스 타겟 등록
	- DoD: 타겟 UP
- [ ] **Grafana 대시보드** — JVM/HTTP 패널
	- DoD: 요청수·지연 그래프 표시
- [ ] **(주간 DoD)** — 실데이터 대시보드 1개 완성

## W14 (9/13~9/19) — 구조화 로깅
- [ ] **JSON 로그 포맷** — logback encoder
	- DoD: 로그가 JSON으로 출력
- [ ] **MDC traceId/requestId**
	- DoD: 모든 로그에 traceId 포함
- [ ] **전파** — Gateway→서비스→Feign 헤더로 traceId 전달
	- DoD: 서비스 간 동일 traceId 유지
- [ ] **(주간 DoD)** — 한 요청을 traceId로 전 서비스 로그에서 추적

## W15 (9/20~9/26) — 분산 추적 + 로그 수집
- [ ] **OpenTelemetry 계측** — 자동 계측 적용
	- DoD: 트레이스 생성
- [ ] **추적 백엔드** — Tempo 또는 Jaeger
	- DoD: 트레이스 저장/조회
- [ ] **Loki 로그 수집**
	- DoD: Grafana에서 로그 검색
- [ ] **(주간 DoD)** — 한 요청의 분산 트레이스를 Grafana에서 확인

## W16 (9/27~10/4) — 알림 + 시크릿
- [ ] **Alertmanager 룰 1개** — 에러율/지연 임계치
	- DoD: 알림 발화 테스트
- [ ] **Vault 기동** — 시크릿 1개 저장
	- DoD: Vault에 시크릿 저장됨
- [ ] **시크릿 이관** — .env → Vault (DB비번/JWT secret)
	- DoD: 앱이 Vault에서 시크릿 로드
- [ ] **(주간 DoD)** — 알림 발화 + Vault 시크릿 로드 동작

## Phase 3 종료 기준
- [ ] traceId로 전 서비스 분산 추적 가능
- [ ] 시크릿이 Vault로 이관됨
- [ ] 다음 → [[phase4-week-breakdown]]

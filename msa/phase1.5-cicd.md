---
tags: [msa, cicd, github-actions, phase1.5]
status: 대기
기간: Phase 1 완료 직후 (2026-08 초) · 약 3일
관련: "[[msa-roadmap]]"
이전: "[[phase1-week-breakdown]]"
---

# Phase 1.5 — CI 파이프라인 (행동 단위)

> [!info] 왜 여기(K8s보다 한참 앞)인가
> CI는 배포 타깃(Compose / K8s)과 무관하다. 템플릿이 `compose up`으로 도는 순간부터
> "push → 빌드 → 품질검사 → 이미지 Push"가 돌면 **이후 모든 phase가 자동 빌드 위에서** 진행된다.
> 그래서 K8s(Phase 5)와 분리해 앞으로 당긴다. 구현은 **GitHub Actions**(러너·유지보수 부담 0).
> Jenkins / SonarQube(자체) / Nexus 는 **나중 슬롯**으로 보류.

## 작업 (반나절~하루 단위)
- [ ] **기본 CI 워크플로** — `.github/workflows/ci.yml` 로 build + test 자동화
	- DoD: push 시 Actions가 자동 빌드/테스트, 실패하면 빨간불
- [ ] **이미지 빌드 + GHCR Push** — 멀티스테이지 빌드 → GitHub Container Registry에 태그 푸시
	- DoD: main 머지 시 이미지가 GHCR에 올라감 (Nexus 없이 레지스트리 확보)
- [ ] **SonarCloud 품질 게이트** — 정적 분석 + 커버리지 임계치
	- DoD: 기준 미달 PR은 머지 차단
- [ ] **(선택) 빌드 캐시** — Gradle 캐시로 빌드 단축
	- DoD: 2회차 빌드가 캐시로 빨라짐
- [ ] **(완료 DoD)** — 커밋 한 번 → 빌드 → 테스트 → 품질검사 → 이미지 Push 까지 자동

## Compose 단계 배포 방법
CD 도구(ArgoCD) 없이도 충분. GHCR에 올라간 이미지를 `docker compose pull` 로 받아 쓴다.
ArgoCD(GitOps)는 K8s 들어갈 때 → [[phase5-week-breakdown|Phase 5 W25]].

## 나중 슬롯 (보류 — 아키텍처 박스의 자체호스팅 자리)
지금은 이 자리에 GitHub Actions가 들어가 있고, "자체 호스팅 CI 구축" 자체가 목표가 되면 교체한다.
아키텍처의 의도(CI 자동화)는 살리고 구현만 가벼운 걸로 시작 — 템플릿 철학(인프라 교체/확장 용이)과 일치.
- [ ] Jenkins 자체 호스팅으로 교체 (CI 운영 학습이 목표일 때)
- [ ] SonarQube 자체 호스팅
- [ ] Nexus 아티팩트 저장소 (jar 등 별도 아티팩트 관리가 필요해질 때)

## Phase 1.5 종료 기준
- [ ] push → 자동 빌드/테스트/품질검사/이미지 Push 동작
- [ ] 이후 phase 작업이 모두 자동 빌드 위에서 진행됨
- [ ] 다음 → [[phase2-week-breakdown]]

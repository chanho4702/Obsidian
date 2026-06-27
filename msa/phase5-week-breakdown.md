---
tags: [msa, k8s, argocd, 배포, phase5]
status: 대기
기간: 2026-11-01 ~ 2026-12-13
관련: "[[msa-roadmap]]"
이전: "[[phase4-week-breakdown]]"
---

# Phase 5 — Kubernetes + ArgoCD (행동 단위)

> 한 칸 = 1~3시간 + 미니 DoD. **Docker Compose 완성 후** 진입.
> K8s가 서비스 디스커버리를 제공하므로 **Eureka는 도입하지 않는다**(중복 회피).
> 이미지 빌드/푸시(CI)는 [[phase1.5-cicd|Phase 1.5]]에서 이미 끝남 → 여기는 **CD·K8s 운영**에 집중.

## W21 (11/1~11/7) — K8s용 이미지/매니페스트 점검
- [ ] **GHCR 이미지 확인** — CI가 올린 이미지 태그 정리
	- DoD: 배포에 쓸 이미지 태그 전략 확정
- [ ] **imagePullSecret** — k8s가 GHCR에서 풀하도록 설정
	- DoD: 프라이빗 이미지 풀 성공
- [ ] **(주간 DoD)** — k8s가 GHCR 이미지를 당겨와 기동

## W22 (11/8~11/14) — K8s 기본
- [ ] **로컬 클러스터** — kind 또는 minikube
	- DoD: 클러스터 기동
- [ ] **Deployment/Service** — 1서비스 배포
	- DoD: 파드 Running
- [ ] **ConfigMap/Secret** — 설정 주입
	- DoD: 환경변수 주입 확인
- [ ] **Ingress** — 외부 노출
	- DoD: Ingress로 서비스 접근
- [ ] **(주간 DoD)** — 1서비스가 Ingress로 외부 접근

## W23 (11/15~11/21) — 전체 스택 이전 + health
- [ ] **전 서비스 매니페스트**
	- DoD: 모든 서비스 배포본 작성
- [ ] **readiness/liveness probe**
	- DoD: 비정상 파드 자동 교체
- [ ] **의존 인프라 연결** — MySQL/Redis/Kafka 배포 or 외부화
	- DoD: 서비스가 의존성에 연결
- [ ] **(주간 DoD)** — 전체 스택이 k8s에서 기동

## W24 (11/22~11/28) — 패키징 + 환경 분리
- [ ] **Helm 또는 Kustomize** — 차트/오버레이
	- DoD: 템플릿화 완료
- [ ] **dev/prod 값 분리**
	- DoD: 환경별 값으로 배포
- [ ] **Namespace 분리**
	- DoD: 환경별 네임스페이스
- [ ] **(주간 DoD)** — 환경별 값으로 배포

## W25 (11/29~12/5) — GitOps
- [ ] **ArgoCD 설치**
	- DoD: ArgoCD UI 접속
- [ ] **Application 등록** — Git 레포 연결
	- DoD: 동기화 상태 표시
- [ ] **자동 동기화**
	- DoD: Git 푸시 → 클러스터 반영
- [ ] **(주간 DoD)** — Git 변경이 자동 배포됨

## W26 (12/6~12/13) — 스케일/무중단
- [ ] **HPA** — CPU/메트릭 기반
	- DoD: 부하 시 스케일아웃
- [ ] **Rolling Update**
	- DoD: 배포 중 무중단
- [ ] **롤백**
	- DoD: 이전 버전으로 롤백 가능
- [ ] **(주간 DoD)** — 부하 시 스케일아웃 + 무중단 배포

## Phase 5 종료 기준
- [ ] Git 푸시로 무중단 배포(ArgoCD)
- [ ] 부하 시 자동 스케일
- [ ] 다음 → [[phase6-week-breakdown]]

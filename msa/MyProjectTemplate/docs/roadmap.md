# 전체 로드맵

## Phase 0 — 기준선과 문서

- [x] 범용화 원칙과 기술 기준선
- [x] local/dev/prod 환경 계약
- [x] 처리량 검증 방법과 초기 용량 등급
- [x] 구성 JSON schema
- [ ] ADR과 버전 업그레이드 정책

완료 조건: 새 사용자가 README에서 선택 기준과 비보장 범위를 이해한다.

## Phase 1 — 조립 가능한 애플리케이션 기반

- [x] Web starter
- [x] PostgreSQL/Flyway starter
- [x] writer/reader routing과 회귀 테스트
- [x] Redis cache starter
- [x] Kafka event starter
- [x] Elasticsearch search starter
- [x] security/observability starter
- [x] 게이트웨이, 샘플 서비스와 서비스 생성기

완료 조건: starter를 하나씩 제거해도 나머지 서비스가 빌드되고, 단일 DB와 writer/reader 구성이 모두 테스트된다.

## Phase 2 — 구성 마법사와 로컬 플랫폼

- [x] 단계형 옵션 UI
- [x] 권장 topology와 경고 동적 표시
- [x] `template-config.json` 내보내기
- [x] Compose profile 생성/실행 명령
- [x] 설정 적용 CLI
- [x] local DB writer/reader, 서비스, 게이트웨이 스모크

완료 조건: 빈 checkout에서 화면 선택만으로 필요한 인프라와 샘플 서비스를 실행할 수 있다.

## Phase 2.5 — 서비스 프론트엔드

- [ ] `apps/web` React 19 + Vite + TypeScript 기반
- [ ] pnpm workspace와 프론트 공통 `packages/`
- [ ] local/dev/prod API endpoint와 Gateway 연동
- [ ] 선택형 OIDC 로그인·갱신·로그아웃
- [ ] OpenAPI 기반 API client 생성과 계약 검증
- [ ] 공통 오류·로딩·권한 처리와 E2E 테스트
- [ ] 구성 마법사의 `프론트 없음 / SPA / SSR` 선택

완료 조건: 프론트와 Gateway API 변경을 같은 PR에서 검증하고, 프론트와 백엔드를 독립 이미지로 배포할 수 있다. SSR은 실제 SEO 요구가 있을 때 별도 adapter로 추가한다.

## Phase 3 — 용량 인증 하네스

- [x] k6 smoke와 constant-arrival target 시나리오
- [ ] knee point 탐색, spike, soak 시나리오
- [ ] Prometheus dashboard와 결과 리포트 생성
- [ ] 인스턴스 제거와 reader 장애 시나리오
- [ ] C1/C2 기준선 실측

완료 조건: 환경과 Git SHA가 포함된 재현 가능한 처리량 보고서가 생성된다.

## Phase 4 — 운영 배포

- [ ] Helm chart와 환경별 values
- [ ] HPA, PDB, topology spread, NetworkPolicy
- [ ] migration job과 rollback runbook
- [ ] OIDC/Keycloak 선택형 구성
- [ ] Secret Manager adapter 예제
- [ ] backup/restore 검증

완료 조건: dev와 prod가 계정·secret·data plane을 공유하지 않고, 불변 이미지로 배포된다.

## Phase 5 — 확장 adapter

- [ ] OpenSearch adapter
- [ ] Valkey adapter
- [ ] cloud queue adapter
- [ ] outbox/CDC 모듈
- [ ] object storage 모듈
- [ ] gRPC starter와 mTLS

각 adapter는 실제 수요와 계약 테스트가 있을 때만 추가한다.

---
tags: [msa, alm, wiki, notion, portfolio, architecture]
작성일: 2026-08-16
상태: 게시 원문 완성 — Notion 연결 대기
게시대상: https://app.notion.com/p/ALM-WIKI-3bda0d2516be80edaea4ca18bfb26c43
---

# ALM·Wiki 온프렘 MSA 플랫폼 — 프로젝트 산출물

> 이 문서는 Notion `ALM-WIKI` 페이지 게시용 최종 정리본이다.

## 프로젝트 한 줄 소개

Keycloak SSO, 자체 JWT BFF, API Gateway, 조직·권한, Wiki, ALM, Redis Streams, OpenSearch 한국어 통합 검색, 중앙 로그 관측을 하나의 온프렘 설치형 MSA 템플릿으로 구현한 프로젝트다.

## 해결하려는 문제

- 문서와 이슈가 서로 다른 제품에 흩어져 검색·권한·운영 방식이 갈리는 문제
- 온프렘 환경에서 서비스별 기술 스택과 배포 절차가 늘어나 운영 복잡도가 커지는 문제
- 이벤트·검색 장애가 쓰기 화면에는 드러나지 않아 데이터가 조용히 스테일해지는 문제

## 핵심 설계 결정

- 로그인: Keycloak OIDC Authorization Code + Google IdP
- 프론트 토큰 계약: auth-server가 자체 RS256 JWT 발급, RT는 HttpOnly 쿠키
- 권한: org-service가 GLOBAL/SPACE/PROJECT grant의 단일 진실 소스
- 서비스 간 동기 호출: gRPC
- 비동기 이벤트: 기존 Redis를 재사용한 Redis Streams
- 검색: OpenSearch 2.x + Nori, PostgreSQL은 정본이고 색인은 재생 가능한 파생 데이터
- 로그: 애플리케이션 stdout ECS JSON → Alloy → Loki → Grafana
- 배포: Docker Compose + GitHub Actions/GHCR + self-hosted runner

## 현재 구현 범위

### 플랫폼

- Keycloak 로그인·백채널 로그아웃·Google IdP
- auth-server BFF JWT/JWKS
- Gateway 인증·rate limit·fallback
- org-service 조직·팀·GLOBAL/SPACE/PROJECT 권한 REST/gRPC
- nginx 단일 오리진에서 포털·Wiki·ALM 3개 SPA 제공

### Wiki

- 스페이스·페이지·폴더·초안·리비전·첨부
- 마크다운/TipTap 편집과 계층 트리
- Redis Streams 변경 이벤트
- OpenSearch 제목·본문·첨부 파일명 검색
- PAGE/FOLDER/ATTACHMENT 유형별 검색 결과 라우팅

### ALM

- 프론트: 프로젝트, 보드, 백로그·스프린트, 이슈, 워크플로·스킴, 관계, 워크로그, 한국어 스마트 검색
- 백엔드: PostgreSQL 프로젝트·이슈 정본, PROJECT 권한, 불변 이슈 키, 낙관적 동시성 제어
- 프로젝트·이슈 변경 이벤트와 search-service용 gRPC 원문 조달
- OpenSearch `alm-issue` Nori 색인과 Wiki+ALM GraphQL 통합 검색
- 현재 경계: 기존 고급 ALM 프론트는 localStorage 정본이며 서버 API 전환은 다음 프론트 슬라이스

## 주요 기술적 문제와 해결

1. split-horizon OIDC 때문에 컨테이너 내부 issuer와 브라우저 issuer가 달라지는 문제를 수동 ClientRegistration으로 분리했다.
2. DB 커밋 후 검색 색인을 쓰기 요청에 결합하지 않고 Redis Streams로 분리했다.
3. 소비자는 at-least-once, PEL delivery count 재시도, XCLAIM 회수, Lua DLQ+ACK 원자화로 운영 정합성을 확보했다.
4. OpenSearch 외부 버전에 이벤트 시각을 사용해 중복과 순서 역전을 방어했다.
5. 재색인은 새 물리 인덱스를 채운 뒤 여러 별칭을 한 호출로 전환해 검색 중단과 반쪽 전환을 막았다.
6. GraphQL HTTP 200 안의 장애를 `extensions.code/httpStatus`로 구분해 빈 결과와 503을 프론트까지 전파했다.
7. 검색 highlight는 `<em>`만 React `<mark>`로 변환하고 raw HTML을 렌더하지 않아 XSS 경계를 유지했다.

## 검증 근거

- platform-backend: org-service 36 tests + search-service 69 tests green, 전체 Gradle build 통과
- alm-backend: 9 tests green, 실제 PostgreSQL Flyway·FK cascade Testcontainers와 in-process gRPC 포함
- gateway-server: 27 tests green
- wiki-front: 77 files / 593 tests green, typecheck와 production build 통과
- Docker: 16개 Compose 서비스 전부 running, healthcheck 대상 전부 healthy, 정적·라이브 스모크 통과
- 실제 인증 E2E: Keycloak 로그인 → auth-server 자체 JWT → Gateway → search-service → org 권한 → OpenSearch
- 실제 검색 E2E: `폴더` 검색 HTTP 200, 6 hits, 6건 모두 FOLDER, highlight 포함
- 실제 ALM E2E: 관리자 JWT → 프로젝트/이슈 생성 → Redis Streams 3 이벤트 → `ISSUE` 검색 1 hit
- ALM 권한 음성 대조: PROJECT grant 없는 사용자 검색 0 hit
- E2E 픽스처 삭제 후 `alm-issue` 잔존 0, consumer pending 0, DLQ 0 확인

## 로드맵

- 완료: Wave A org-service
- 완료: Wave B wiki-backend
- 완료: Wave C search-service + wiki-front 통합 검색
- 완료: Wave D alm-backend + ISSUE 통합 색인 백엔드 종단 슬라이스
- 다음: ALM 프론트 서버 연동 → Wave E notification-service + 설치판 마감
- 공통 후속: outbox, gRPC 서비스 인증, 재색인 dual-write/이벤트 재생, 메트릭·분산 추적

## 저장소

- platform-backend: common-proto, org-service, search-service
- wiki-backend / wiki-front
- alm-backend / alm-front
- auth-server / gateway-server / eureka-server / board-service
- infra-settings / design-system

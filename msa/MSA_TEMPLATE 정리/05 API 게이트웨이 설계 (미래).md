---
tags: [msa, template, gateway, spring-cloud-gateway, 설계, 미구현]
작성일: 2026-06-30
상태: 설계만 (미구현)
---

# 5단계 — API 게이트웨이 설계 (미래)

상위: [[00 개요 — 전체 구조]]

> ⚠️ **아직 구현 안 됨.** 설계 문서(`docs/superpowers/specs/2026-06-30-msa-gateway-design.md`, 별도 세션 작성)만 존재. 여기는 그 요약.

## 목적

단일 진입점 **API 게이트웨이**를 도입해 프론트–서비스 토폴로지를 정리하고, "서비스 추가 시 무엇을 복제하는지"가 코드로 드러나는 표준 패턴 확립.

## 핵심 결정

| # | 결정 | 근거 |
|---|---|---|
| 1 | 게이트웨이 스택 = **Spring Cloud Gateway (WebFlux)** | auth-server와 같은 Spring, 일관성 |
| 2 | **토큰 검증은 게이트웨이가 안 함** — 각 서비스가 JWKS로 자체 검증 유지 | 분산 검증 ([[03 board-service (샘플 백엔드 + SSO 증명)]]) |
| 3 | 게이트웨이 = **순수 라우팅 + 공통 관심사(CORS·로깅)**, Spring Security 미탑재 | 책임 최소화 |
| 4 | 서비스 위치 = **정적 설정 + Docker DNS**(디스커버리 서버 없음) | 추가 인프라 0 |
| 5 | 라우팅 = **서비스명 prefix(`/api/board/**`) + No StripPrefix** | 경로가 곧 서비스 지도 |
| 6 | Rate limit·서킷브레이커·디스커버리·분산추적 = **문서화만, 구현 X** | YAGNI |

## 목표 토폴로지

```
myFront(:5173) ──▶ gateway-server(:8000) ──┬──▶ auth-server(:9000)
 (base URL 1개)    라우팅/CORS/로깅          └──▶ board-service(:9100) /api/board/**
```

## 도입 시 변경 포인트 (계획)

- **myFront**: `VITE_API_BASE`/`VITE_BOARD_API_BASE` 2개 → **`VITE_API_BASE=:8000` 하나로 통합**. board 경로 `/api/posts/**` → `/api/board/posts/**`.
- **board-service**: 컨트롤러 경로를 `/api/board/**`로 리네임.
- **CORS 일원화**: board·auth가 각자 가진 CORS를 **모두 제거하고 게이트웨이 한 곳으로** 이전(`CorsConfigTest`도 삭제).

> 현황 정정: board-service는 골격이 아니라 **이미 완성·커밋된 상태**. 그래서 board 작업은 "신규 구현"이 아니라 **기존 경로 리네임**.

다음: [[06 저장소 구조 + 운영 원칙]]

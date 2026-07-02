---
tags: [msa, template, 프론트, react, board, cors]
작성일: 2026-06-30
상태: 구현 완료 (브라우저 시각확인은 사람 항목)
---

# 4단계 — 프론트 연동 (myFront → board-service)

상위: [[00 개요 — 전체 구조]]

## 정의

myFront의 게시판 화면(이미 완성된 스캐폴드)을 **localStorage 목업에서 board-service REST 호출로 교체**. `프론트 → 인증서버 → 백엔드` 샘플의 마지막 연결 고리.

## 핵심 통합 포인트 — 토큰 공유

board 호출은 별도 인증 인스턴스를 만들지 않고, **기존 `authClient`(:9000)가 메모리에 들고 있는 access token을 공유**한다.

- `boardStore.ts`의 `boardFetch`: `authClient.getAccessToken()`으로 Bearer 첨부 → 401이면 `authClient.tryRefresh()` 후 1회 재시도.
- 읽기(GET)는 public이라 토큰 없이도 동작, 쓰기만 Bearer 필요.
- board-service는 쿠키 미사용 → `credentials:'include'` 안 씀(Bearer 전용).
- `.env`: `VITE_BOARD_API_BASE=http://localhost:9100` 추가(auth는 `VITE_API_BASE=:9000` 유지).

## 백엔드 변경 — CORS (필수)

프론트(:5173) → board-service(:9100)는 교차 출처라 CORS 없으면 브라우저가 막음. board-service `SecurityConfig`에 CORS 추가:
- `platform.cors-allowed-origin`(기본 `http://localhost:5173`), 메서드 GET/POST/PUT/DELETE/OPTIONS.
- Bearer만 쓰므로 `allowCredentials=false`. TDD로 프리플라이트 테스트 추가(30번째 테스트).

## 화면 변경 (백엔드 계약에 맞춤)

- **목록**: 서버 페이지네이션(MUI Pagination ↔ 서버 page/totalPages). **제목 검색·조회수 컬럼 제거**(백엔드에 없음).
- **상세**: 조회수 제거, 작성자=`authorName`. **수정/삭제 버튼은 본인 또는 ADMIN만 노출**(`user.sub === String(post.authorId) || user.role==='ADMIN'`) — 서버 인가와 동일.
- **작성/수정**: 작성자 입력칸 제거(토큰 신원 자동), 제목+내용만. 에러는 토스트(`useNotify`).
- id 타입 string(UUID) → number(Long), 비동기화(loading/error 상태).

## 검증

- board-service CORS 프리플라이트(:5173→:9100) 200 ✅
- `Page` JSON 형태가 프론트 기대와 일치 ✅
- 토큰+Origin 작성 흐름 → authorName 자동 ✅
- `npm run build`(tsc 타입체크) 통과 ✅
- **브라우저 클릭 E2E(로그인→CRUD)는 사람 확인 항목으로 남음.**

## 미연결 (후속)

댓글 UI — board-service에 Comment CRUD는 있으나 프론트는 이번 범위에서 제외("이번엔 게시글만").

다음: [[05 API 게이트웨이 설계 (미래)]]

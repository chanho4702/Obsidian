---
tags: [msa, wiki-front, search, graphql, opensearch, oidc, e2e]
작성일: 2026-08-15
상태: 구현·로컬 배포·라이브 검증 완료
repos: [wiki-front, platform-backend]
---

# 26 — wiki-front 통합 검색 UI + 라이브 JWT E2E (2026-08-15)

상위: [[00 개요 — 전체 구조]] · 선행: [[25 Wave C — search-service 구현·배포 완료기록 (2026-08-15)]] · 평가: [[24 현재 아키텍처·구현 평가 + 개선 백로그 (2026-08-03)]]

## 결론

Wave C의 후속 두 건인 **wiki-front 통합 검색 UI**와 **유효 JWT 검색 200 + hits E2E**를 닫았다. 헤더에서 검색하면 `/search?q=...`로 이동하고, search-service GraphQL 결과를 페이지·폴더·첨부 유형에 맞는 경로로 연결한다. Keycloak Authorization Code 로그인부터 auth-server 자체 JWT 발급, Gateway, search-service, org 권한 필터, OpenSearch 검색까지 실제 배포 경로로 검증했다.

## 계약 보강

- GraphQL `SearchHit`에 `pageType: PAGE | FOLDER`를 추가했다. 기존에는 페이지와 폴더가 같은 문서 유형이라 프론트가 `/pages/:id`와 `/folder/:id` 중 올바른 경로를 고를 수 없었다.
- 첨부 파일명 검색어도 강조할 수 있도록 OpenSearch highlight 대상에 `filename`을 추가했다.
- 구 인덱스 문서에 `type`이 없을 때는 호환성을 위해 `PAGE`로 해석한다. 첨부 결과의 `pageType`은 `null`이다.

## 프론트 구현

- TopBar 중앙 검색창에서 Enter 또는 버튼으로 검색한다. 검색어·페이지는 URL을 정본으로 사용해 새로고침과 링크 공유가 가능하다.
- `wikiApi`와 `wikiMock`에 동일한 `searchContent` 계약을 추가했다. 프로덕션 빌드는 별도 환경변수가 없어도 same-origin 실제 API를 사용한다.
- 결과 화면은 로딩, 최초 안내, 빈 결과, 429, 503, 재시도를 구분한다.
- PAGE는 `/spaces/:spaceId/pages/:id`, FOLDER는 `/spaces/:spaceId/folder/:id`, ATTACHMENT는 소유 페이지로 이동한다.
- 서버 highlight는 `dangerouslySetInnerHTML` 없이 `<em>` 토큰만 React `<mark>`로 변환한다. 그 밖의 HTML 유사 문자열은 텍스트로 렌더링한다.

## 검증

| 게이트 | 결과 |
|---|---|
| search-service | 65/65 tests green, 실제 Nori OpenSearch Testcontainers 포함 |
| wiki-front | 77 files, 593/593 tests green |
| 정적 품질 | TypeScript typecheck 및 Vite production build PASS |
| 로컬 배포 | search-service 이미지 재빌드·healthy, `C:\deploy\dist\wiki`에 새 번들 반영, nginx 정적 경로 200 |
| OIDC E2E | Keycloak `alice` 로그인 → auth-server callback/refresh → 자체 JWT `sub=1` 확인 |
| 검색 E2E | Gateway 경유 `폴더` 검색 HTTP 200, 6 hits, 6건 모두 `pageType=FOLDER`, highlight 포함 |

재색인 전에는 기존 wiki 데이터가 이벤트 소비자 도입 이전 데이터라 별칭 문서 수가 0이었다. GLOBAL ADMIN으로 원자 재색인을 실행해 `wiki-page-v5` 17건(폴더 7건)을 활성화한 뒤 E2E를 수행했다. 실패 진단용으로 남아 있던 비활성·빈 v2/v3 인덱스는 별칭 미연결과 0건을 확인한 뒤 정리했고, 활성 v5와 롤백용 v4/v1은 보존했다.

## 배포·형상 상태

이번 작업은 로컬 Docker의 search-service와 nginx 정적 배포 폴더까지 반영했다. 스택은 실행 중이다. **커밋과 푸시는 하지 않았다.**

## 다음에 이어갈 것

1. **Wave D** — ALM 이벤트·색인 확장
2. dev +10000 오프셋 클러스터에서 search-service 라이브 검증
3. [[24 현재 아키텍처·구현 평가 + 개선 백로그 (2026-08-03)]]의 M-01~M-07 후속
4. 재색인 dual-write/이벤트 재생, DB outbox, gRPC 서비스 인증

← [[25 Wave C — search-service 구현·배포 완료기록 (2026-08-15)]] · [[00 개요 — 전체 구조]]

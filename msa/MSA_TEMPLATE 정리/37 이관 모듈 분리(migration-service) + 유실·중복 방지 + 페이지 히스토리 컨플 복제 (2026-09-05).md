---
tags: [msa, template, migration-service, wiki-backend, wiki-front, 이관, 배포, 히스토리, 병렬세션]
작성일: 2026-09-05
상태: X1~X4 완료·배포(migration-service 운영 기동) / 유실·중복 방지 보강 배포 / 페이지 히스토리 W30 푸시 / wiki-backend 후속 2건 진행 중
---

# 37 — 이관 모듈 분리(migration-service) + 유실·중복 방지 + 페이지 히스토리 컨플 복제 (2026-09-05)

상위: [[00 개요 — 전체 구조]] · 직전: [[36 사용자 초대·팀 권한 플랫폼 공통 + 마이그레이션 M1~M3 + proto 0.16.0 (2026-09-05)]]
정본: `wiki-front/docs/superpowers/specs/2026-09-05-migration-service-split-design.md`(분리), `…/2026-09-06-page-history-confluence-design.md`(히스토리)

> 사용자: "노션·컨플 마이그레이션은 따로 모듈로 제공해서 만들 거임, 위키에 있으면 안 됨" → A안(별도 서비스 + 위키 내부 import API).
> "마이그레이션 모듈은 데이터 유실 안 되는 게 정말 중요한데 API 간극·중간에 끊기는 거·장애 처리 어떻게 했어?" → 실측 답 + 구멍 3건 즉시 보강.
> "버전 히스토리 보기 어렵다, 컨플 따라하면 됨" → 모달을 전용 화면 3개로.

## 1. 분리 구조 (X1~X4)

```
migration-service (새 리포, migrationdb, :9170)             wiki-backend
 ├ 엔진 123 클래스(DC 클라이언트·핸들러·IR·정규화기·워커)   ├ /internal/wiki/import/**  ← X-Internal-Token
 ├ WikiImportClient(JDK HttpClient, 스트리밍 업로드)          │   페이지/리비전/첨부/댓글/제한/순서/본문 재작성/검증 조회
 ├ org gRPC(주체 매핑)·common-starter(JWT)                   ├ page.imported_author_name/url 유지
 ├ REST /api/migration/** (게이트웨이 라우트)                └ migration_* 6개 테이블 V37 DROP
 └ Flyway V1(엔진 원장)·V2(잡 단위 이슈)
```

| 단계 | 커밋 | 내용 |
|---|---|---|
| X1 wiki-backend | 750f0de | 내부 import API 12 엔드포인트 + `InternalTokenFilter`(빈 토큰 fail-closed) + 계약 픽스처 16개 |
| X2 migration-service | e1d09b6 … e8324f1 | 리포 신설, 엔진 이동, `WikiImportClient`, 가짜 위키 서버 테스트, springdoc("Migration API", 공통 오류 문구) |
| X3 배선 | infra a844957·e674925 / gateway 63a48e6 / wiki-front 488204d / ms d0e8b43·4ea8dfd | compose `migration-service`+`migration-db-init`+스테이징 볼륨, `MIGRATION_SERVICE_URI`, wiki-backend `WIKI_INTERNAL_TOKEN`, deploy.yml·smoke 8b·dev-up-local, 게이트웨이 `/api/migration/**`, 프론트 base path. 포트는 search-service(9140) 충돌로 **9170**(dev 19170) |
| X4 wiki-backend | 7e70162 | `migration/**` 108 클래스·테스트·픽스처 삭제, `ImportedPageWriter`만 importapi로 이동, V37, docs 프로필 차단 제거. 472 → 361 테스트 |

**배포 실측:** 사용자가 `GH_PACKAGES_TOKEN` 등록 → CI 그린 → infra deploy 수동 실행(`DISPATCH_PAT`·`DEPLOY_ENABLED` 미등록이라 자동 dispatch 없음). **첫 배포 실패**: deploy가 `up --no-deps`라 `migration-db-init`이 안 돌아 `database "migrationdb" does not exist` → init 1회 수동 실행 뒤 성공(compose 주석 e674925). 게이트웨이 경유 무토큰 401, 내부 import 경로는 nginx 정적 폴백(외부 미노출) 확인. `WIKI_INTERNAL_TOKEN`은 `.env`+`C:\deploy\platform.env` 동일 값.

## 2. 유실·중단·장애 처리 — 실측 답과 보강

**되어 있던 것**: EXTRACT 원본 스냅샷 선저장(`migration_payload`) · 단계별 체크포인트(EXTRACT→NORMALIZE→MEDIA_COPY→RESOLVE→VERIFY) · 워커 리스 만료 회수 + claim 토큰으로 옛 시도 결과 폐기 · retryable(429/5xx/타임아웃/위키 UNAVAILABLE) 지수 백오프 5회 → dead letter, permanent(404/거부)는 즉시 · 첨부 SHA-256 스테이징(`.part`→move), 위키가 같은 이름·checksum이면 UNCHANGED · 위키 쪽 페이지+리비전+라벨 REQUIRES_NEW 원자성 · 미지원 매크로 opaque+원본 참조 보존, 인라인 댓글 강등, 제한 fail-closed, DRY_RUN 쓰기 0건, 재실행 멱등(object map + checksum).

**구멍과 보강(migration-service b27e56a, 137 테스트)**:
1. **중복 페이지** — RESOLVE가 첨부·제한·댓글까지 끝난 뒤에야 object map을 썼다. 그 사이 retryable 실패/리스 만료 재실행이면 "기존 없음"으로 페이지를 하나 더 만들었다 → create/update 직후 `bindTargetPage`(REQUIRES_NEW, checksum은 PENDING)로 즉시 못박고, 재시도는 갱신 경로. 회귀 테스트: 첨부 503 1회 → 페이지 1건.
2. **링크 정리 실패 삼킴** — 잡 마감 후 `dc-page:` 정리 pass 실패가 로그뿐이었다 → `LINK_FIXUP_FAILED` 이슈를 문서별·잡 단위로(V2: `migration_issue.item_id` NULL 허용 + 부분 유니크 + 잡 FK), 잡 상세 `jobIssues[]`, `POST /api/migration/{jobId}/link-fixup` 재실행(COMPLETED/FAILED만, 멱등).
3. **VERIFY 범위** — 제목·타입·본문 유무·라벨만 → MEDIA_MANIFEST checksum ⊆ 위키 attachments, commentCount ≥ 이관 댓글 수(포함 관계 — 사람이 더한 건 어긋남 아님).
4. (진행 중) **위키 import 멱등 키** — V38 `page.import_key`, 같은 키면 `outcome: EXISTING`. object map이 사라져도 중복이 안 나게 하는 마지막 겹.

wiki-front b63162d: 잡 상세에 "잡 이슈" 섹션 + 링크 정리 재실행 버튼(COMPLETED/FAILED만 렌더).

## 3. 페이지 히스토리 컨플 복제 (W30, wiki-front 1ef8ab2, 1171 테스트)

모달(좌측 목록+우측 탭+비교 select) → 컨플 페이지 히스토리 3화면. `…/history` 표(체크박스 2개 제한 → "선택한 버전 비교", `v. N` 링크, "현재" 배지, 아바타+이름, 절대시각, 변경 요약, 복원) · `…/history/:version` 배너("이전 버전(v. N)을 보고 있습니다 · 누가 · 언제") + 현재 버전 보기·현재 버전과 비교·복원 · `…/history/compare?from&to` 메타 카드 2개 + 제목 변경 + DiffView. 복원은 확인 다이얼로그 + 변경 요약(기본 "v. N에서 복원") — 어댑터는 changeNote 있을 때만 본문, 백엔드 수용은 후속 2건 중 하나. 읽기 전용 인스턴스는 스페이스 홈으로. 레퍼런스 대비 없음(후속): 버전 삭제(API 없음), 비교 화면 이전/다음 변경 점프.

## 4. 병렬 세션 조율

- 같은 트리에서 2e 세션(wiki-backend springdoc·myFront 문서 수집)·06 세션(ALM)과 동시 작업 — 파일 지정 add, hunk 필터 스테이징(`git apply --cached`), 인덱스 blob 직접 갱신(`hash-object` + `update-index`)까지 썼다. `git add -A`는 트리가 내 변경뿐임을 확인한 X4 한 번만.
- `.gitattributes`(골든 LF 고정)는 위키에서 지우고 migration-service 1a19a4c로 이관. 2e가 복구했던 것을 되돌리며 이유 공유.
- myFront 수집기에 `migration`(9170, optional) 항목은 2e가 추가, 배포 뒤 optional 해제 요청.
- Bash 툴 heredoc에 백슬래시 이스케이프가 깨진다(`\\r\\n`이 실제 개행으로) — 파이썬 편집 스크립트는 Write 툴로 만든다.

## 5. 남은 것

| 항목 | 상태 |
|---|---|
| wiki-backend 후속 2건(restore changeNote 본문, import 멱등 키 V38) | 레인 진행 중 → migration-service가 importKey 보내도록 후속 |
| migration-service 리포 `DISPATCH_PAT`·`DEPLOY_ENABLED` | 사용자 등록 전까지 infra deploy 수동 실행 |
| 실기 컨플 DC 실측 · 노션 라이브 추출기 | 사용자 준비 / 범위 밖 |
| 검색 재색인 1회 · Keycloak T1/T4 · SMTP | 그대로 |
| wiki-backend가 org `denied_reason`·`GetMembers` 소비 | 후속 |
| 관리자 PAT `/api/auth/agents` 차단 여부 · PAT B단계 OpenAPI | 결정 대기 |

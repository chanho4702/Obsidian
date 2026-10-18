---
tags: [msa-template, plan]
created: 2026-10-18
---

# 2026-10-18 전체 검수 → 기능·성능 개선 계획

검수 원문(96건): [[2026-10-18-audit/wiki|wiki]] · [[2026-10-18-audit/alm|alm]] · [[2026-10-18-audit/platform|platform]] · [[2026-10-18-audit/ops-docs|ops-docs]]

## 사용자 결정 (10-18)
- 10-16 미병합 브랜치 → 먼저 main 병합 (완료: wiki-backend 475fc62, alm-backend e4edd75, wiki-front d9e7264, alm-front 8c5f7fd, infra f0d0b3f; agent-service P4c bf7576f 수동 배포)
- P4c(alm-front 미커밋 WIP) → 검증 후 마무리·커밋
- 반영 범위 → 게이트 통과 시 main 푸시·배포
- 운영 조치 → 코드·가드·런북·백업 예약 작업까지 Claude, 실제 비밀번호 교체는 사용자(런북)
- 위키 초안 → 작성자(와 관리자)만 보임, 게시 전 알림·검색 제외
- org gRPC 서비스 간 인증·구성원 비활성 전파 → 진행(단계적: 클라이언트 먼저 송신 → 전 서비스 배포 → 서버 강제)
- 리팩터링 → 적극적으로 분할(도메인별 모듈)
- 부족한 기능 보완 → 기획 갭 분석 후 웨이브 2

## 웨이브
| 웨이브 | 내용 | 담당 |
|---|---|---|
| 0 | 미병합 브랜치 병합·배포, 외부 스킬 7종 설치, 교육 문서 00~13 | 완료 |
| 1 | 리포별 결함·보안·성능 수정 (wiki-backend · alm-backend · gateway · platform-backend+auth · agent-service · infra) | 백엔드/운영 에이전트 6 |
| 1F | 프론트 성능·분할·P4c 마무리 (wiki-front · alm-front) | 프론트 에이전트 2 |
| 2 | 기능 공백 보완 (기획 갭 분석 결과) | 기획 → 엔지니어 |
| 3 | 교차 보안 P-01/P-02 단계 배포 마무리(서버 강제) | 백엔드 |
| 4 | 문서 정리(루트 README/CLAUDE/AGENTS·서비스 CLAUDE.md·myFront·DS), 하네스 갱신 | 운영/리뷰 |
| R | 리포별 리뷰(code-review + Codex 교차검증) → 게이트 → main 푸시 | 리뷰 |

## 운영 규칙
- 각 에이전트는 자기 리포의 별도 worktree(`improve/20261018-*` 브랜치)에서만 작업, 공유 체크아웃 미접촉.
- 동시 부하로 vitest/testcontainers가 타임아웃 → 실패 파일 단독 재실행으로 판정.
- 마이그레이션 번호: wiki V42~, alm V27~ (병합 후 기준).

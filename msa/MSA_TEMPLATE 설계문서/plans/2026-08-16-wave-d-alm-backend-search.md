---
tags: [msa, wave-d, alm, implementation-plan]
작성일: 2026-08-16
상태: 구현·검증 완료 (형상화/Notion 게시 진행)
설계: [[2026-08-16-wave-d-alm-backend-search-design]]
---

# Wave D 구현 계획 — alm-backend + 통합 검색

## T1. common-proto v0.6.0 계약

- events.proto에 Project/Issue 20~25 추가
- `platform/alm/v1/alm.proto`에 색인 조달 gRPC 추가
- 생성 코드 컴파일과 proto 회귀 테스트

## T2. alm-backend 골격

- 독립 Gradle repo, Spring Boot 4.0.6/Java 24
- JWT issuer/audience, PostgreSQL/Flyway, docker/dev 프로필, gRPC 9121/19121
- Dockerfile·CI·README

## T3. 프로젝트 도메인

- Project 엔티티/마이그레이션/REST/서비스
- 키 정규화·중복 409
- 생성자 PROJECT ADMIN 자동 grant, VIEW/EDIT/ADMIN 권한
- ProjectCreated/Updated/Deleted 커밋 후 이벤트

## T4. 이슈 도메인

- Issue 엔티티/마이그레이션/REST/서비스
- 프로젝트별 이슈 번호 발번과 불변 issueKey
- CRUD 권한, expectedVersion 낙관적 락
- IssueCreated/Updated/Deleted 이벤트

## T5. ALM 색인 조달

- AlmContentService 단건/전량 스트리밍
- NOT_FOUND와 고아 프로젝트 FAILED_PRECONDITION 분리
- 실제 gRPC 통합 테스트

## T6. search-service ALM 투영

- alm-issue Nori 매핑·부트스트랩
- ALM gRPC client, 이벤트 indexer, 외부 버전 upsert/delete
- 프로젝트 이름 갱신·프로젝트 삭제 cascade

## T7. 통합 GraphQL 권한 계약

- SPACE+PROJECT grant를 한 번에 읽는 권한 범위
- `DocType.ISSUE`, `projectIds`, mixed-domain bool 분기
- SearchHit nullable 도메인 필드와 ALM 메타
- 실제 OpenSearch 통합 테스트

## T8. 원자 재색인

- wiki-page/wiki-attachment/alm-issue 세 인덱스 생성·백필·refresh
- 세 별칭 한 호출 전환, 이슈 진행 카운트/상태 노출
- 실패 시 구 별칭 보존 테스트

## T9. Gateway·Compose·DB

- Gateway `/api/alm/**` + dev lb/docker DNS 분기 + 인증 경계 테스트
- almdb init, alm-backend 서비스, search ALM gRPC env
- 백엔드 배포 매핑·태그·컨테이너 이름·health 대기

## T10. E2E·문서·형상화

- 전체 Gradle 게이트
- Compose 기동, 유효 JWT 프로젝트/이슈 생성→검색 hit→권한 음성 대조
- Obsidian 완료기록과 Notion 게시 원문 갱신
- 저장소별 기능 브랜치 커밋·푸시

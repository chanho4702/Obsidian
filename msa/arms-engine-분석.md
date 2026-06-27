# ARMS 엔진 코드 분석 — 인덱스(MOC)

> 대상: `C:\ARMS\ENGINE` (313DEVGRP `java-service-tree-framework-engine-fire`)
> 분석일: 2026-06-18 / 분석 범위: **아키텍처 개요 · 재사용 가능한 패턴 · 설정·인프라** (비즈니스 로직 제외)
> 규모: Java 405개 파일 / 약 28,000 LOC. Spring Boot 3.4.4 · Java 21 · Spring Cloud 2024.0.1.

기존 [[msa-roadmap]] 템플릿을 만들 때 **실제로 굴러간 레퍼런스**로 쓰기 위한 분석. "이론"이 아니라 "이미 운영에 들어간 코드가 이렇게 풀었다"를 정리한다.

## 한 줄 요약
모놀리식 Spring Boot 엔진이지만, **OpenSearch(ES) 위에 자체 ORM/쿼리 프레임워크**(`egovframework.esframework`)를 얹고, **AOP·@RestControllerAdvice·공통 응답 객체**로 횡단 관심사를 표준화한 구조. Redmine/Jira(ALM) 연동과 Slack 알림이 핵심 외부 의존성.

## 분석 노트
- [[arms-engine-아키텍처]] — 패키지/레이어 구조, 모듈 경계, 데이터 흐름
- [[arms-engine-재사용패턴]] — 다른 프로젝트에 가져다 쓸 만한 기법 모음 ⭐
- [[arms-engine-설정인프라]] — Gradle·Docker·프로파일·OpenSearch·Feign·로깅 설정

## MSA 로드맵과의 연결
| 로드맵 항목 | ARMS에서 이미 구현된 형태 | 참고 노트 |
|---|---|---|
| Phase1 W4 — 공통 응답(ApiResponse)·ErrorCode·GlobalExceptionHandler | `CommonResponse.ApiResult<T>` + `ErrorCode` enum + `ErrorControllerAdvice` | [[arms-engine-재사용패턴]] |
| Phase1 W6 — OpenFeign 호출·에러 표준화 | `@FeignClient BackendCommunicator` + hc5 클라이언트 | [[arms-engine-설정인프라]] |
| Phase1.5 — 멀티스테이지 빌드/이미지 Push | `bmuschko docker-remote-api` Gradle 태스크 | [[arms-engine-설정인프라]] |
| Phase2 — 인덱스/조회 최적화 | ES scroll/search_after, rolling index, reindex | [[arms-engine-재사용패턴]] |
| 횡단 관심사(로깅/모니터링) | `@Aspect LoggingAdvice` + Slack 알림 + APM agent | [[arms-engine-재사용패턴]] |

> ⚠️ 반면교사 포인트도 함께 기록함(예: 예외 핸들러 메서드명을 한글로 작성, `System.out.println` 잔존 등). 가져올 건 가져오고 버릴 건 버린다.

# ARMS 엔진 — 아키텍처 개요

> 상위 노트: [[arms-engine-분석]]

## 패키지 구조 (com.arms)
```
com.arms
├─ Application.java              # 진입점. @EnableAspectJAutoProxy, @EnableAsync, ModelMapper 빈
├─ config/                       # 인프라 빈 (OpenSearch, Feign, Slack, Swagger, Captcha …)
├─ api/                          # 비즈니스 도메인 (controller/service/model)
│   ├─ account / admin / analysis / backoffice / bbs / dashboard
│   ├─ issue / project / requirement / report / search_engine / serverinfo / wiki
│   └─ util/                     # 공통 유틸 (응답·에러·AOP·ALM연동·암호화·Slack·MSA통신)
└─ egovframework.javaservice.esframework/   # 자체 ES 추상화 프레임워크 ⭐
    ├─ annotation/  config/  esquery/  factory/  model/  repository/  util/
```

## 레이어링 관례
도메인마다 동일한 3-레이어를 반복한다(일관성 높음):
```
{domain}/controller/{Domain}Controller.java     # REST 엔드포인트
{domain}/service/{Domain}Service.java            # 인터페이스
{domain}/service/{Domain}ServiceImpl.java        # 구현 (service 50개 중 Impl 22개 — 인터페이스+구현 분리)
{domain}/model/dto/                              # 요청/조회 파라미터 (SearchDocDTO 등 상속)
{domain}/model/vo/                               # 응답/뷰 모델
{domain}/model/entity/                           # ES 문서 매핑 (BaseEntity 상속)
```
> **DTO ↔ VO ↔ Entity 변환**은 `ModelMapper` 빈(전역)으로 처리. 변환 보일러플레이트를 줄이는 대신 매핑이 암묵적이 되는 트레이드오프.

## 데이터 흐름
```
Client
  → Controller            (@RestController, 진입 로깅: LoggingAdvice @Before)
  → Service(Impl)         (비즈니스 로직, @Around 예외→Slack 알림)
  → EsCommonRepositoryWrapper<E>   (도메인 엔티티 타입으로 ES 질의 위임)
  → EsCommonRepository / Impl      (OpenSearch RestHighLevelClient)
  → OpenSearch 클러스터
```
- 영속 저장소는 **RDB(JPA)가 아니라 OpenSearch**가 1차. (build.gradle에서 JPA는 주석 처리, MySQL 커넥터는 일부 용도로만 존재)
- 검색/집계가 본질인 제품이라 ES를 메인 스토어로 삼고 그 위에 Repository 추상화를 직접 구현.

## 모듈 경계 / 의존성 방향
- `egovframework.esframework`는 **도메인(api)에 의존하지 않는 하부 프레임워크**여야 정상인데, `IndexStatusSaveAspect`가 `com.arms.api.admin.indexstatus.service`를 import → **하부→상부 역참조**가 1곳 존재(경계 누수). MSA로 쪼갤 때 끊어야 할 지점.
- 외부 시스템 연동:
  - **ALM**: Jira(`jira-rest-java-client`), Redmine(`redmine-java-api`) — `api/util/alm`
  - **알림**: Slack(`slack-api-client`) — 에러 발생 시 채널 통보
  - **MSA 통신**: `@FeignClient`(backend-core), DWR 클라이언트 — `api/util/msa_communicate`
  - **APM**: Elastic APM agent (docker-entrypoint.sh에서 javaagent 주입)

## 핵심 횡단 관심사 (전역 빈/AOP)
| 관심사 | 구현 | 위치 |
|---|---|---|
| 진입/종료 로깅 + 예외 알림 | `LoggingAdvice` (@Aspect) | `api/util/aspect` |
| 전역 예외 → 표준 응답 | `ErrorControllerAdvice` (@RestControllerAdvice) | `api/util/erros/response` |
| 표준 응답 포맷 | `CommonResponse.ApiResult<T>` | `api/util/response` |
| ES 인덱스 작업 전후 스냅샷 | `IndexStatusSaveAspect` (@Around @annotation) | `esframework/repository/common` |
| 정적 컨텍스트 접근 | `ApplicationContextProvider` | `config` |

→ 패턴 상세는 [[arms-engine-재사용패턴]], 설정은 [[arms-engine-설정인프라]].

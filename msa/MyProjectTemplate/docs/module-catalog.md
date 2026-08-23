# 모듈 카탈로그

## starter 활성화 규칙

starter는 의존성을 추가해도 위험한 기능을 자동으로 켜지 않는다. 외부 인프라 기능은 해당 클래스가 존재하고 `platform.<feature>.enabled=true`일 때 활성화된다.

| 모듈 | 설정 스위치 | 제공 기능 |
|---|---|---|
| `platform-starter-web` | 기본 활성 | 요청 ID, UTC Clock, 표준 Problem Detail |
| `platform-starter-data-jpa` | `platform.datasource.enabled` | JPA, Flyway, writer/reader routing |
| `platform-starter-redis` | `platform.redis.enabled` | 타입 안전 JSON cache, TTL |
| `platform-starter-kafka` | `platform.kafka.enabled` | 공통 event envelope, publisher |
| `platform-starter-search` | `platform.search.enabled` | 검색 포트와 Elasticsearch adapter |
| `platform-starter-observability` | 의존성 기반 | Prometheus, OTLP tracing |

`services/gateway-service`는 외부 진입점과 circuit breaker를 제공한다. 로컬은 정적 URI를 사용하고 운영에서는 서비스 DNS 또는 플랫폼 discovery를 사용한다.

## 권장 조합

CRUD 서비스:

```groovy
implementation project(':starters:platform-starter-web')
implementation project(':starters:platform-starter-data-jpa')
```

이벤트 발행 서비스:

```groovy
implementation project(':starters:platform-starter-kafka')
```

조회 최적화 서비스:

```groovy
implementation project(':starters:platform-starter-redis')
implementation project(':starters:platform-starter-search')
```

서비스가 사용하지 않는 starter는 넣지 않는다. 모든 starter를 일괄 포함하는 `platform-starter-all`은 만들지 않는다.

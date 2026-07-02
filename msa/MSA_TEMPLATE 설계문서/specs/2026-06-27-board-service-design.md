# board-service 설계 (표준 게시판 백엔드)

작성일: 2026-06-27

## 목적

MSA_TEMPLATE에 **표준 게시판(게시글 + 댓글)** 백엔드를 추가한다. 이 서비스는
auth-server가 발급한 자체 Access Token(JWT)을 **자체 검증하는 리소스 서버**로,
앞서 학습한 JWT Validation · Role Mapping · Method Security를 실제 비즈니스
서비스에 적용하는 첫 사례다.

## 컨텍스트 / 아키텍처 전제

- 각 서비스는 `MSA_TEMPLATE` 아래 독립 Gradle 프로젝트다. board-service는
  **새 형제 디렉터리** `MSA_TEMPLATE/board-service`로 생성한다. auth-server는
  수정하지 않는다.
- auth-server는 향후 **게이트웨이 서버로 발전**시킬 예정이다. 그래도 board-service
  설계는 그대로 유효하다 — board-service는 `jwk-set-uri`로 auth-server(=장차
  게이트웨이)의 JWKS를 바라보고 토큰을 검증하므로, issuer와 JWKS 경로만 유지되면
  된다. 분리/병합 어느 쪽이든 `jwk-set-uri` 한 줄만 영향받는다.
- 인가 2단 방어 중 board-service는 **서비스 자체 인가(2차)** 를 책임진다.
  게이트웨이(1차)는 이번 범위 밖이다.

## 스택

auth-server와 동일: Spring Boot 4.0.6 / Java 24 / Spring Web / Spring Security
(resource-server) / Data JPA / Validation / Flyway(PostgreSQL) / Lombok /
PostgreSQL(runtime) / H2(test) / spring-security-test.

- 포트: **9100** (auth-server 9000)
- DB: PostgreSQL 데이터베이스 `boarddb` (auth-server는 `authdb`). Flyway로 스키마 관리.

## 토큰 검증 방식 (확정: A안)

`application.yml`에 auth-server JWKS URL을 설정한다.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: http://localhost:9000/.well-known/jwks.json
```

- 키 회전 자동 대응, 결합도 최소(공개키 복사 불필요).
- `issuer-uri`는 **설정하지 않는다** — 설정 시 Spring이 OIDC discovery
  (`/.well-known/openid-configuration`)를 시도하는데 auth-server가 그 문서를
  노출하지 않아 기동 실패한다. 대신 issuer(`http://localhost:9000`) 검증이 필요하면
  `NimbusJwtDecoder`에 `JwtValidators.createDefaultWithIssuer(...)` 같은
  `OAuth2TokenValidator`를 코드로 추가한다(구현 단계에서 결정).
- 거부안: 공개키 파일 직접 보관(수동 갱신), OIDC discovery(위 사유로 현재 불가).

## 데이터 모델

```
post
  id           BIGSERIAL PK
  title        VARCHAR  NOT NULL
  content      TEXT     NOT NULL
  author_id    BIGINT   NOT NULL   -- JWT sub (userId)
  author_name  VARCHAR  NOT NULL   -- 작성 시점 name 스냅샷
  created_at   TIMESTAMPTZ NOT NULL
  updated_at   TIMESTAMPTZ NOT NULL

comment
  id           BIGSERIAL PK
  post_id      BIGINT   NOT NULL  FK -> post(id) ON DELETE CASCADE
  content      TEXT     NOT NULL
  author_id    BIGINT   NOT NULL
  author_name  VARCHAR  NOT NULL
  created_at   TIMESTAMPTZ NOT NULL
  updated_at   TIMESTAMPTZ NOT NULL
```

- 글 삭제 시 댓글 cascade 삭제.
- `author_name`은 표시용 스냅샷(작성 시점 값 고정).

## 엔드포인트 & 권한

| 메서드 | 경로 | 권한 |
|--------|------|------|
| GET | `/api/posts` (페이징, 최신순) | 공개 |
| GET | `/api/posts/{id}` | 공개 |
| POST | `/api/posts` | 로그인 사용자 |
| PUT | `/api/posts/{id}` | 작성자 본인 OR ADMIN |
| DELETE | `/api/posts/{id}` | 작성자 본인 OR ADMIN |
| POST | `/api/posts/{id}/comments` | 로그인 사용자 |
| PUT | `/api/comments/{id}` | 작성자 본인 OR ADMIN |
| DELETE | `/api/comments/{id}` | 작성자 본인 OR ADMIN |

목록 조회는 `Pageable`(기본 최신순) 사용.

## 인가 구현 (학습 핵심)

- `SecurityConfig`
  - `oauth2ResourceServer().jwt(...)` + auth-server와 **동일한** roles→`ROLE_*`
    변환 컨버터(`JwtGrantedAuthoritiesConverter`, claim `roles`, prefix `ROLE_`).
  - `@EnableMethodSecurity` 활성화.
  - `SessionCreationPolicy.STATELESS`, csrf disable, GET `/api/posts/**` `permitAll`,
    그 외 쓰기 요청 `authenticated`.
- **본인 OR ADMIN** 판정: 서비스 계층에서 현재 사용자 `sub`와 리소스 `author_id`를
  비교하고 `ROLE_ADMIN`이면 통과, 아니면 `AccessDeniedException`(403).
  - `@PreAuthorize` 표현식 방식(`hasRole('ADMIN') or @postSecurity.isOwner(...)`)도
    주석/대안으로 함께 보여 둘 다 학습.
- 현재 사용자 추출 헬퍼: `Authentication`/`Jwt`에서 `sub`(userId)와 권한을 읽는다.

## 에러 처리

`@RestControllerAdvice`로 일관된 JSON 에러 응답:

- 401 미인증 (토큰 없음/무효) — 리소스 서버가 처리
- 403 권한 없음 (`AccessDeniedException`)
- 404 리소스 없음 (커스텀 NotFound 예외)
- 400 검증 실패 (`@Valid` → `MethodArgumentNotValidException`)

## 테스트

auth-server와 동일 방식:

- JUnit5 + `spring-security-test`의 `jwt()` post-processor로 가짜 토큰(roles/sub 지정).
- H2 runtime + Flyway(또는 테스트 프로파일)로 슬라이스 테스트.
- 시나리오: 공개 GET 통과 / 미인증 쓰기 401 / 작성자 본인 수정·삭제 통과 /
  타인 수정·삭제 403 / ADMIN 타인 글 삭제 통과 / 없는 글 404 / 검증 실패 400.

## 범위 밖 (YAGNI)

게이트웨이 서버, 검색, 카테고리/게시판 구분, 조회수, 첨부파일, 좋아요, soft-delete.
필요해지면 후속 spec으로 분리한다.

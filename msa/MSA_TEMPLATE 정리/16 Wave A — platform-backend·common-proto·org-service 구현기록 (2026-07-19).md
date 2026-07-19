---
tags: [msa, template, alm-wiki, grpc, proto, rbac, github-packages, sdd, 구현기록]
작성일: 2026-07-19
상태: Wave A 완료 (12태스크, 최종 리뷰 Ready) · 수동 E2E ②(dev모드)·④(관리자 REST) 대기 · 다음 Wave B(wiki-backend)
---

# 16 — Wave A: platform-backend·common-proto·org-service 구현기록 (2026-07-19)

상위: [[15 ALM·Wiki 백엔드 요구사항 + 서비스 분할 설계 (2026-07-19)]] · 선행: [[13 백엔드 컨테이너화 + compose 통합 — CI-CD Wave 1 (2026-07-15)]]

## 무엇을 했나

[[15 ALM·Wiki 백엔드 요구사항 + 서비스 분할 설계 (2026-07-19)|15번 설계]]의 첫 구현 웨이브. ALM·Wiki 플랫폼의 **토대 3종**을 하루에 설계→스펙→플랜→구현까지 완주했다:

1. **platform-backend** — 횡단 서비스용 Gradle 멀티모듈 독립 repo ([chanho4702/platform-backend](https://github.com/chanho4702/platform-backend))
2. **common-proto** — gRPC 계약 모듈. `com.platform:common-proto:0.1.0`으로 **GitHub Packages 발행** → 이후 wiki/alm-backend가 의존성으로 소비
3. **org-service** — 조직·팀·RBAC. 모든 서비스의 권한 판정 단일 진입점 (REST :9130 + gRPC :9131)

문서 체계: 스펙 `specs/2026-07-19-wave-a-platform-backend-design.md` · 플랜 `plans/2026-07-19-wave-a-platform-backend.md` (12태스크). 실행은 SDD(서브에이전트 구동 개발) — 태스크별 구현자→리뷰어 게이트, 마지막에 전체 브랜치 최종 리뷰.

## 1. repo·모듈 구조

```
platform-backend/            독립 repo (main, v0.1.0 태그)
├─ common-proto/             proto → Java 코드젠 + maven-publish
│   └─ src/main/proto/platform/org/v1/org.proto
└─ org-service/              Spring Boot 4.0.6 / Java 24 (board-service 계열 스택)
    ├─ src/main/java/com/platform/orgservice/
    │   ├─ domain/           enum 6종 + 엔티티 4종 (member·team·team_member·grant_entry)
    │   ├─ repository/       JPQL FQN enum 리터럴 쿼리 (H2 enum 바인딩 리스크 회피)
    │   ├─ permission/       PermissionFacade — 판정 단일 진입점
    │   ├─ security/         AudienceValidator·MemberMirrorFilter(JIT)
    │   ├─ me/ team/ member/ grant/   REST 컨트롤러·서비스
    │   ├─ grpc/             PermissionGrpcService + GrpcServerLifecycle
    │   └─ config/           SecurityConfig·BootstrapAdminSeeder
    └─ Dockerfile            런타임 전용 (기존 4서비스 패턴)
```

**포트 규칙(전 웨이브 공통 확정)**: HTTP `91x0` / gRPC `91x1` — wiki 9110/9111, alm 9120/9121, **org 9130/9131**, search 9140, notification 9150. 9200번대는 Elasticsearch(9200/9300) 충돌 예방으로 회피.

## 2. 권한 모델 — grant_entry 단일 원장

```
grant_entry: subject(USER|TEAM) × resource(GLOBAL|SPACE|PROJECT, resource_id) × role(VIEWER<EDITOR<ADMIN)
```

- **판정**: 사용자 직접 grant + 소속 팀 grant를 모아 **최고 role** 채택. GLOBAL grant는 모든 리소스에 적용. `role.rank >= action.requiredRank`(VIEW 1 / EDIT 2 / ADMIN 3)면 허용
- **resource_id는 string** — wiki의 space id, alm의 project id가 각자 체계를 가져도 계약이 안 흔들림 (Wave B~D를 위한 선계약)
- **부트스트랩**: env `PLATFORM_BOOTSTRAP_ADMIN_ID` → 기동마다 GLOBAL ADMIN upsert(강등돼 있어도 복구). 최초 관리자 문제의 해법
- **JIT 멤버 미러링**: 인증 요청의 JWT 클레임(sub·name·email)로 member 행 upsert — 원본은 auth-server/Keycloak, org는 로컬 뷰만. Kafka 없이 해결(토대 웨이브가 인프라를 앞지르지 않게)
- **싱글 테넌트 전제**: 온프렘 설치 1개 = 조직 1개. organization 엔티티 없음

> [!note] 배경지식 — 왜 판정을 org-service 한 곳에 두나
> wiki가 "이 사용자가 이 스페이스를 편집할 수 있나"를 스스로 판단하면 권한 로직이 서비스마다 복제되고 정책 변경이 전 서비스 재배포가 된다. 판정을 gRPC `CheckPermission` 한 곳에 두면 각 서비스는 "질문"만 하고, 정책(role 계층, 팀 상속, GLOBAL 우선)은 org-service만 안다. Wave C의 검색 권한 필터링도 `ListUserGrants`(사용자가 접근 가능한 리소스 목록)로 같은 원장을 쓴다.

## 3. 통신 계약 — proto v1

`platform.org.v1` / java_package `com.platform.proto.org.v1`:

- `CheckPermission(user_id, resource_type, resource_id, action) → (allowed, effective_role)` — 인가 2차 방어용
- `ListUserGrants(user_id, resource_type?) → grants[]` — 검색 필터·프론트 메뉴 제어용
- **패키지에 v1을 박음** — breaking change는 v2 패키지 신설(온프렘에서 구버전 서비스와 공존해야 하므로)

> [!warning] proto enum 함정 — `Role.ROLE_ADMIN`이 오타가 아닌 이유
> proto3의 enum 값 이름은 **패키지 스코프**(C++ 스코핑 규칙)를 쓴다. 같은 패키지에 `Action.ADMIN`이 있으면 `Role.ADMIN`은 이름 충돌로 컴파일이 안 된다. 그래서 `Role`만 `ROLE_` 접두사를 갖고, gRPC 어댑터에서 `Role.ROLE_ADMIN ↔ GrantRole.ADMIN`을 명시 매핑한다. 후속 웨이브에서 proto enum을 추가할 때 반드시 기억할 것.

**발행 파이프라인**: `v*` 태그 push → CI가 `:common-proto:publish -PprotoVersion=${태그#v}` → GitHub Packages. 소비 측은 `implementation 'com.platform:common-proto:0.1.0'` + Packages 저장소 인증(GITHUB_TOKEN, **read:packages 스코프 필수** — gh 기본 토큰엔 없어서 `gh auth refresh -s read:packages` 실측 필요했음).

## 4. gRPC 서버 — 스타터 없이 순정으로

스펙 단계에서 spring-grpc를 검토했으나 **플랜 단계에서 순정 grpc-java로 수정** — Boot 4 호환 스타터 버전 미확정 리스크 + 서비스 1개에 스타터는 YAGNI. 구현은:

- `GrpcServerLifecycle implements SmartLifecycle` — Boot 수명주기에 편입, `@ConditionalOnProperty(platform.grpc.enabled)` 로 테스트에서 오프
- `ProtoReflectionService` 포함 → **grpcurl이 -proto 파일 없이 호출 가능** (컨테이너 네트워크 안에서 실측: `docker run --rm --network platform_default fullstorydev/grpcurl ...` → `allowed: true, ROLE_ADMIN`)
- 테스트는 `InProcessServerBuilder` — 실포트 없이 실 Facade+H2 경유로 왕복 검증
- gRPC :9131은 **컨테이너 내부 전용**(ports 미발행) — 신뢰 네트워크 전제, mTLS/인터셉터는 소비자가 늘어나는 웨이브에서

## 5. Eureka 프로필 분기 — 첫 적용

[[15 ALM·Wiki 백엔드 요구사항 + 서비스 분할 설계 (2026-07-19)|15번 확정]]의 첫 실전: **default(dev) 프로필 = Eureka 등록, docker 프로필 = `eureka.client.enabled: false` + 컨테이너 DNS**. 게이트웨이 라우트는 env 오버라이드 한 줄로 양쪽을 겸한다:

```yaml
- id: org
  uri: ${ORG_SERVICE_URI:lb://org-service}   # dev: Eureka lb:// · docker: http://org-service:9130
  predicates: [ Path=/api/org/** ]
```

기존 4서비스는 무수정(게이트웨이 라우트 1건만 추가). compose는 10컨테이너가 됐다(기존 9 + org-service).

## 6. 사건·교훈 (리뷰 게이트가 잡은 것들)

| 사건 | 원인 | 교훈 |
|---|---|---|
| `@UniqueConstraint(columnNames={"teamId",...})` — 플랜 코드 자체의 버그 | Hibernate는 columnNames를 **물리 컬럼명**으로 해석 — camelCase면 EntityManagerFactory 부트스트랩 실패 | `@Column(name=...)`으로 리네임한 필드의 제약은 반드시 snake_case로. 플랜 코드도 리뷰 대상 |
| 구현자의 "테스트 green" 보고가 `11 up-to-date` 캐시 로그 | gradle up-to-date는 **실행 안 했다**는 뜻 | 신선 실행 증거는 `--rerun-tasks` + "N executed"로 요구할 것 |
| CI 첫 실행 `./gradlew: Permission denied` (exit 126) | Windows 커밋은 실행 비트를 잃는다 | `git update-index --chmod=+x gradlew` 1커밋. Windows→Actions repo의 정석 함정 |
| grpc 생성 코드가 `@javax.annotation.Generated` 요구 | grpc-java 1.69는 아직 jakarta가 아닌 javax 어노테이션 생성 | `javax.annotation-api` compileOnly 추가 (POM 미전이 확인) |
| addMember가 member 존재 미검증 | 운영(Postgres)은 FK로 500, H2 테스트(create-drop)는 통과 — **테스트/운영 거동 발산** | H2 슬라이스가 못 잡는 제약은 명시 가드로. 최종 리뷰에서 수정 |
| GLOBAL grant에 resourceId "junk" 저장 가능 | 판정 쿼리는 GLOBAL을 id 무관 매칭, 시더·requireGlobalAdmin은 `("")` 정확 일치 — 중복 행·조회 누락 | 저장 시점 정규화(GLOBAL→""). 계약의 양쪽 끝(쓰기·읽기)을 함께 볼 것 |
| Boot 4 `@DataJpaTest` 패키지 이동 | `spring-boot-data-jpa-test` 의존성 + 새 슬라이스 패키지 | 6월 auth-server에서 이미 겪은 것 — 선례 재확인됨 |

## 7. 검증 결과 (스펙 성공 기준 5개)

| # | 기준 | 결과 |
|---|---|---|
| ① | proto 발행 + 외부 gradle resolve | ✅ `com.platform:common-proto:0.1.0` BUILD SUCCESSFUL |
| ② | dev 모드: IDE 기동 → Eureka 등록 → 게이트웨이 경유 200 | ⏳ 수동 체크리스트 |
| ③ | docker 모드: grpcurl CheckPermission | ✅ `allowed: true, effectiveRole: ROLE_ADMIN` |
| ④ | 부트스트랩 관리자 REST 201 / 일반 403 | ⏳ 수동 체크리스트 (무토큰 401 스모크는 ✅) |
| ⑤ | 전 모듈 빌드·테스트 | ✅ 29/29 |

## 8. Wave B 이월 목록 (최종 리뷰 defer 판정)

- org-service **GHCR 이미지 발행** — 게이트웨이 로컬 이미지↔GHCR 발산 정리와 같은 티켓으로
- compose에 `PLATFORM_ISSUER`/`PLATFORM_AUDIENCE` **명시 주입** — 현재는 auth·org 양쪽 기본값 우연 일치에 의존
- `PLATFORM_BOOTSTRAP_ADMIN_ID` 기본값 1 — 운영 배포 전 명시 설정 경고 문서화
- gRPC 인증(인터셉터/mTLS) — 소비자 늘어날 때
- `ProtoReflectionService` → `ProtoReflectionServiceV1` 교체
- `Long.parseLong(jwt.getSubject())` 반복 → 공용 ArgumentResolver

## 커밋 지도

platform-backend(빈트리→3134b02, 18커밋): 골격 e4eaf5b → proto 2ab3967 → 부트스트랩 2a58d47 → 스키마 82475c1+8c4fae2(fix) → Facade 0e506da → 보안/JIT 4bd8e57 → 시드/팀 bd75140+7cf9cf1(테스트 보강) → grant REST 9fe3540 → gRPC fecb863 → Docker cd4c55a → CI e7c72bd+139914e(+x) → README c967ed6 → 최종 fix 3134b02
gateway-server: ea65206(라우트) · 우산: 6ec03fb(compose·orgdb) + 09d8b69(README) — **우산 push는 프론트 CI 세션 병행으로 보류 중**

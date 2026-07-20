# Wave B: wiki-backend 코어 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** wiki-backend(스페이스·페이지 트리·버전·첨부 + 권한 gRPC 연동 + Redis Streams 이벤트 발행)를 독립 repo로 구현하고 compose 스택에 통합한다.

**Architecture:** 선행으로 platform-backend의 common-proto에 `platform.events.v1`과 org.v1 `CreateGrant`를 추가해 **v0.2.0 발행**. wiki-backend는 그 발행본을 GitHub Packages에서 소비하는 첫 외부 소비자다. 권한은 org-service gRPC(:9131) + Caffeine 캐시(fail-closed 403), 이벤트는 `EventPublisher` 추상화 뒤 Redis Streams(`platform:events:v1`, 커밋 후 발행, best-effort).

**Tech Stack:** Spring Boot 4.0.6 / Java 24 / Spring Cloud 2025.1.2 / Flyway·PostgreSQL(wikidb) / H2(test) / grpc-java 1.69.0 / Caffeine(Boot BOM) / spring-boot-starter-data-redis / Lombok 1.18.46.

**스펙:** `MSA_TEMPLATE 설계문서/specs/2026-07-20-wave-b-wiki-backend-design.md`

## Global Constraints

- wiki-backend repo 루트: `C:\MSA_TEMPLATE\wiki-backend` (독립 git repo, 우산 gitignore 등록). platform-backend 작업은 `C:\MSA_TEMPLATE\platform-backend`.
- 버전 고정(Wave A 동일): Boot `4.0.6`, dependency-management `1.1.7`, Spring Cloud `2025.1.2`, Java `24`, Lombok `1.18.46`, grpc `1.69.0`, protobuf-java `4.29.3`. group `com.platform`.
- 포트: wiki-backend HTTP **9110** (gRPC 9111은 예약만 — 이번 웨이브 서버 없음). DB **wikidb**(기존 Postgres 인스턴스, 계정 keycloak — 기존 관례).
- proto: `com.platform:common-proto:0.2.0` — events.v1 추가 + org.v1 CreateGrant. **패키지 내 rpc/메시지 추가는 하위호환**(마이너 범프).
- JWT: `jwk-set-uri`만(issuer-uri 금지), issuer/audience 검증, roles 컨버터 — board/org 패턴. **JIT 미러링 없음**(org-service 소관).
- 권한 규칙: VIEW=조회·다운로드 / EDIT=페이지·첨부 변경 / ADMIN=스페이스 관리 / 스페이스 생성=인증만+생성자 SPACE ADMIN 자동 부여. org 불능 시 **fail-closed 403**.
- 이벤트: 본문(content) 미포함. 스트림 `platform:events:v1`, MAXLEN ~100000, 발행 실패는 WARN 로그 후 비차단.
- 테스트: `@ActiveProfiles("test")` — H2(MODE=PostgreSQL)+Flyway off+create-drop+eureka off. **PermissionClient·EventPublisher는 테스트에서 페이크 빈**. TDD(RED 증거 필수), 전체 회귀는 `--rerun-tasks`로 실행 수 명시.
- 로컬 빌드 주의: wiki-backend는 GitHub Packages 인증 필요 — `$env:GITHUB_TOKEN = (gh auth token)` (read:packages 스코프 확보됨). `$env:JAVA_HOME = 'C:\Program Files\Java\jdk-24'`.
- 커밋: 한국어 프리픽스 관례, 태스크 말미 커밋. gradlew 명령 timeout 600000ms.

---

### Task 1: common-proto v0.2.0 — events.v1 + CreateGrant (platform-backend)

**Files:**
- Create: `platform-backend/common-proto/src/main/proto/platform/events/v1/events.proto`
- Modify: `platform-backend/common-proto/src/main/proto/platform/org/v1/org.proto` (rpc·메시지 추가)

**Interfaces:**
- Produces (java_package `com.platform.proto.events.v1`): `EventEnvelope`(event_id, occurred_at, actor_id, source, oneof payload), `SpaceCreated/SpaceDeleted/PageCreated/PageUpdated/PageDeleted/AttachmentAdded`
- Produces (org.v1): `PermissionServiceGrpc`에 `createGrant`, `CreateGrantRequest{user_id, resource_type, resource_id, role}`, `CreateGrantResponse{created}`

- [ ] **Step 1: events.proto 작성**

`common-proto/src/main/proto/platform/events/v1/events.proto`:
```protobuf
syntax = "proto3";
package platform.events.v1;
option java_multiple_files = true;
option java_package = "com.platform.proto.events.v1";

// 모든 도메인 이벤트의 공통 봉투. Redis Stream(platform:events:v1) 엔트리의 payload 필드에 바이너리로 실린다.
// 본문(content)은 싣지 않는다 — 소비자는 "무엇이 변했나"만 알고, 내용은 소유 서비스 API로 가져간다.
message EventEnvelope {
  string event_id = 1;        // UUID — 소비자 멱등 처리용
  int64 occurred_at = 2;      // epoch millis
  int64 actor_id = 3;         // 행위자 (auth user id)
  string source = 4;          // 발행 서비스명 (예: "wiki-backend")
  oneof payload {
    SpaceCreated space_created = 10;
    SpaceDeleted space_deleted = 11;
    PageCreated page_created = 12;
    PageUpdated page_updated = 13;
    PageDeleted page_deleted = 14;
    AttachmentAdded attachment_added = 15;
    // alm 이벤트는 Wave D에서 20번대로 추가
  }
}

message SpaceCreated { int64 space_id = 1; string key = 2; string name = 3; }
message SpaceDeleted { int64 space_id = 1; }
message PageCreated  { int64 page_id = 1; int64 space_id = 2; string title = 3; }
message PageUpdated  { int64 page_id = 1; int64 space_id = 2; string title = 3; int32 version = 4; }
message PageDeleted  { int64 page_id = 1; int64 space_id = 2; }
message AttachmentAdded { int64 attachment_id = 1; int64 page_id = 2; int64 space_id = 3; string filename = 4; }
```

- [ ] **Step 2: org.proto에 CreateGrant 추가**

`org.proto`의 `service PermissionService` 블록에 rpc 추가, 파일 끝에 메시지 추가:
```protobuf
  // (service 블록 안, ListUserGrants 아래에 추가)
  // 내부 전용 — 리소스 생성자 자동 권한 부여용 (예: wiki 스페이스 생성자 → SPACE ADMIN)
  rpc CreateGrant(CreateGrantRequest) returns (CreateGrantResponse);
```
```protobuf
// (파일 끝에 추가)
message CreateGrantRequest {
  int64 user_id = 1;
  ResourceType resource_type = 2;
  string resource_id = 3;      // GLOBAL이면 빈 값으로 정규화됨
  Role role = 4;
}
message CreateGrantResponse { bool created = 1; }  // 동일 grant 존재 시 false (멱등)
```

- [ ] **Step 3: 코드젠 검증**

```powershell
cd C:\MSA_TEMPLATE\platform-backend; .\gradlew.bat :common-proto:build
```
Expected: BUILD SUCCESSFUL. `common-proto\build\generated\source\proto\main\java\com\platform\proto\events\v1\EventEnvelope.java` 및 grpc 디렉터리의 `PermissionServiceGrpc.java`에 `createGrant` 존재 확인 (Grep으로 확인).

- [ ] **Step 4: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\platform-backend add -A
git -C C:\MSA_TEMPLATE\platform-backend commit -m "feat(proto): events.v1 이벤트 스키마 + org.v1 CreateGrant — v0.2.0 준비"
```

---

### Task 2: org-service CreateGrant 구현 (platform-backend)

**Files:**
- Modify: `platform-backend/org-service/src/main/java/com/platform/orgservice/grpc/PermissionGrpcService.java`
- Test: `platform-backend/org-service/src/test/java/com/platform/orgservice/grpc/PermissionGrpcServiceTest.java` (테스트 추가)

**Interfaces:**
- Consumes: `GrantEntryRepository.findBySubjectTypeAndSubjectIdAndResourceTypeAndResourceId(...)`, `GrantEntry.of(...)`, Task 1의 `CreateGrantRequest/Response`
- Produces: gRPC `createGrant` — 멱등(존재 시 created=false), GLOBAL은 resourceId "" 정규화, UNSPECIFIED 인자 INVALID_ARGUMENT

- [ ] **Step 1: 실패하는 테스트 추가 (기존 테스트 클래스에)**

`PermissionGrpcServiceTest.java`에 추가:
```java
    @Test
    void CreateGrant_신규는_created_true_중복은_false_멱등() {
        CreateGrantRequest req = CreateGrantRequest.newBuilder()
                .setUserId(5L).setResourceType(ResourceType.SPACE).setResourceId("7")
                .setRole(Role.ROLE_ADMIN).build();

        assertThat(stub.createGrant(req).getCreated()).isTrue();
        assertThat(stub.createGrant(req).getCreated()).isFalse(); // 멱등
        assertThat(grants.findBySubjectTypeAndSubjectIdAndResourceTypeAndResourceId(
                SubjectType.USER, 5L, ResourceKind.SPACE, "7")).isPresent();
    }

    @Test
    void CreateGrant_GLOBAL은_resourceId가_빈값으로_정규화된다() {
        stub.createGrant(CreateGrantRequest.newBuilder()
                .setUserId(6L).setResourceType(ResourceType.GLOBAL).setResourceId("junk")
                .setRole(Role.ROLE_ADMIN).build());

        assertThat(grants.findBySubjectTypeAndSubjectIdAndResourceTypeAndResourceId(
                SubjectType.USER, 6L, ResourceKind.GLOBAL, "")).isPresent();
    }

    @Test
    void CreateGrant_UNSPECIFIED_인자는_INVALID_ARGUMENT() {
        assertThatThrownBy(() -> stub.createGrant(CreateGrantRequest.newBuilder()
                .setUserId(5L).setResourceType(ResourceType.RESOURCE_TYPE_UNSPECIFIED)
                .setRole(Role.ROLE_ADMIN).build()))
                .isInstanceOf(io.grpc.StatusRuntimeException.class)
                .hasMessageContaining("INVALID_ARGUMENT");
    }
```
(추가 import: `static org.assertj.core.api.Assertions.assertThatThrownBy`)

- [ ] **Step 2: RED 확인**

```powershell
cd C:\MSA_TEMPLATE\platform-backend; .\gradlew.bat :org-service:test --tests "com.platform.orgservice.grpc.*"
```
Expected: FAIL — `createGrant` 미구현(UNIMPLEMENTED).

- [ ] **Step 3: 구현**

`PermissionGrpcService.java`에 메서드 추가 (proto Role→도메인 GrantRole 역매핑 헬퍼 포함):
```java
    @Override
    public void createGrant(CreateGrantRequest req, StreamObserver<CreateGrantResponse> out) {
        ResourceKind kind = toKind(req.getResourceType());
        GrantRole role = toDomainRole(req.getRole());
        if (kind == null || role == null) {
            out.onError(Status.INVALID_ARGUMENT
                    .withDescription("resource_type/role은 UNSPECIFIED일 수 없습니다").asRuntimeException());
            return;
        }
        String resourceId = kind == ResourceKind.GLOBAL ? "" : req.getResourceId();
        boolean created = grants.findBySubjectTypeAndSubjectIdAndResourceTypeAndResourceId(
                        SubjectType.USER, req.getUserId(), kind, resourceId)
                .isEmpty();
        if (created) {
            grants.save(GrantEntry.of(SubjectType.USER, req.getUserId(), kind, resourceId, role));
        }
        out.onNext(CreateGrantResponse.newBuilder().setCreated(created).build());
        out.onCompleted();
    }

    /** proto Role → 도메인 GrantRole (ROLE_ADMIN ↔ ADMIN 비대칭 유지). */
    private static GrantRole toDomainRole(Role r) {
        return switch (r) {
            case VIEWER -> GrantRole.VIEWER;
            case EDITOR -> GrantRole.EDITOR;
            case ROLE_ADMIN -> GrantRole.ADMIN;
            default -> null;
        };
    }
```
필드 주입 추가: `private final GrantEntryRepository grants;` + `private final SubjectType`/`GrantRole`/`GrantEntry`/`SubjectType` import (`com.platform.orgservice.domain.*`, `com.platform.orgservice.repository.GrantEntryRepository`).

- [ ] **Step 4: GREEN + 전체 회귀**

```powershell
.\gradlew.bat build --rerun-tasks
```
Expected: BUILD SUCCESSFUL, **32 tests** (기존 29 + 신규 3), 0 failures.

- [ ] **Step 5: 커밋 + push**

```powershell
git -C C:\MSA_TEMPLATE\platform-backend add -A
git -C C:\MSA_TEMPLATE\platform-backend commit -m "feat: gRPC CreateGrant — 리소스 생성자 자동 권한 부여(멱등)"
git -C C:\MSA_TEMPLATE\platform-backend push
```

---

### Task 3: v0.2.0 발행 + wiki-backend의 패키지 접근 준비

**Files:** 없음 (외부 운영)

**Interfaces:**
- Produces: GitHub Packages `com.platform:common-proto:0.2.0` — Task 4부터 wiki가 소비

- [ ] **Step 1: CI green 확인 후 태그 발행**

```powershell
gh run list --repo chanho4702/platform-backend --limit 2   # push 트리거 build success 확인
cd C:\MSA_TEMPLATE\platform-backend
git tag v0.2.0; git push origin v0.2.0
gh run list --repo chanho4702/platform-backend --limit 2   # 태그 run id 확인
gh run watch <태그 run id> --repo chanho4702/platform-backend --exit-status --interval 15
```
Expected: build + publish-proto 모두 success. (timeout 600000ms)

- [ ] **Step 2: 로컬 resolve 확인**

```powershell
$env:GITHUB_TOKEN = (gh auth token)
cd $env:LOCALAPPDATA\Temp\claude\C--MSA-TEMPLATE\dc10ec23-3341-4f51-931e-8b3bfab8535d\scratchpad\proto-consume-test
# build.gradle의 버전을 0.2.0으로 수정 후:
.\gradlew.bat dependencies --configuration compileClasspath
```
Expected: `com.platform:common-proto:0.2.0` resolve, FAILED 없음.

- [ ] **Step 3: (사용자 안내 필요 — 웹 UI 1회) 패키지 Actions 접근 권한**

wiki-backend repo의 CI(GITHUB_TOKEN)가 platform-backend의 private 패키지를 읽으려면:
`github.com/users/chanho4702/packages/maven/com.platform.common-proto` → Package settings → **Manage Actions access** → `Add repository` → `wiki-backend` (Role: **Read**).
이 단계는 Task 13(CI) 전까지만 완료되면 된다 — 컨트롤러가 사용자에게 요청하고, 로컬 빌드(Task 4~12)는 개발자 토큰으로 무관하게 진행.

---

### Task 4: wiki-backend repo 골격 + Boot 부트스트랩

**Files:**
- Create: `wiki-backend/settings.gradle`, `build.gradle`, `.gitignore`, `.run/bootRun.run.xml`
- Copy: `gradlew`, `gradlew.bat`, `gradle/` (board-service에서)
- Create: `wiki-backend/src/main/java/com/platform/wikibackend/WikiBackendApplication.java`
- Create: `wiki-backend/src/main/resources/application.yml`
- Test: `wiki-backend/src/test/resources/application-test.yml`, `src/test/java/com/platform/wikibackend/WikiBackendApplicationTests.java`
- Modify: `C:\MSA_TEMPLATE\.gitignore` (`wiki-backend/` 추가 — 커밋은 Task 12에서)

**Interfaces:**
- Produces: 패키지 루트 `com.platform.wikibackend`, 설정 키 `platform.jwt.*`, `platform.org-grpc.host|port`, `platform.wiki.files-dir|max-attachment-mb`, `platform.events.enabled`, 프로필 계약(dev=Eureka/docker=DNS)

- [ ] **Step 1: repo 골격**

```powershell
New-Item -ItemType Directory -Force C:\MSA_TEMPLATE\wiki-backend
git -C C:\MSA_TEMPLATE\wiki-backend init -b main
Copy-Item C:\MSA_TEMPLATE\board-service\gradlew,C:\MSA_TEMPLATE\board-service\gradlew.bat C:\MSA_TEMPLATE\wiki-backend\
Copy-Item -Recurse C:\MSA_TEMPLATE\board-service\gradle C:\MSA_TEMPLATE\wiki-backend\gradle
git -C C:\MSA_TEMPLATE\wiki-backend update-index --add --chmod=+x gradlew 2>$null  # 이후 커밋 시 실행비트 (Wave A CI 사건 예방 — 첫 커밋 후 적용해도 됨)
```

`settings.gradle`: `rootProject.name = 'wiki-backend'`
`.gitignore` (Wave A와 동일 + SDD 스크래치):
```
.gradle/
build/
out/
.idea/
!.run/
*.iml
.superpowers/
data/
```
우산 `C:\MSA_TEMPLATE\.gitignore`의 서비스 repo 블록에 `wiki-backend/` 한 줄 추가(커밋은 Task 12).

- [ ] **Step 2: build.gradle**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.0.6'
    id 'io.spring.dependency-management' version '1.1.7'
}
group = 'com.platform'
version = '0.0.1-SNAPSHOT'
java { toolchain { languageVersion = JavaLanguageVersion.of(24) } }

repositories {
    mavenCentral()
    // common-proto 소비 — GitHub Packages (로컬: GITHUB_TOKEN env, CI: Actions access 부여 후 GITHUB_TOKEN)
    maven {
        url = uri('https://maven.pkg.github.com/chanho4702/platform-backend')
        credentials {
            username = System.getenv('GITHUB_ACTOR') ?: 'chanho4702'
            password = System.getenv('GITHUB_TOKEN') ?: providers.gradleProperty('gpr.token').getOrElse('')
        }
    }
}

ext { set('springCloudVersion', '2025.1.2') }
dependencyManagement {
    imports { mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}" }
}

def grpcVersion = '1.69.0'

dependencies {
    implementation 'com.platform:common-proto:0.2.0'
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'org.springframework.boot:spring-boot-flyway'
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-database-postgresql'
    implementation "io.grpc:grpc-netty-shaded:${grpcVersion}"
    implementation 'com.github.ben-manes.caffeine:caffeine'
    runtimeOnly 'org.postgresql:postgresql'
    compileOnly 'org.projectlombok:lombok:1.18.46'
    annotationProcessor 'org.projectlombok:lombok:1.18.46'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.boot:spring-boot-data-jpa-test'
    testImplementation 'org.springframework.security:spring-security-test'
    testRuntimeOnly 'com.h2database:h2'
}
tasks.named('test') { useJUnitPlatform() }
tasks.named('bootJar') { archiveFileName = 'app.jar' }
```

- [ ] **Step 3: Application + application.yml + 테스트 프로필**

`WikiBackendApplication.java`:
```java
package com.platform.wikibackend;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class WikiBackendApplication {
    public static void main(String[] args) {
        SpringApplication.run(WikiBackendApplication.class, args);
    }
}
```

`application.yml`:
```yaml
server:
  port: 9110
spring:
  application:
    name: wiki-backend
  datasource:
    url: ${WIKI_DB_URL:jdbc:postgresql://localhost:5433/wikidb}
    username: ${WIKI_DB_USERNAME:keycloak}
    password: ${WIKI_DB_PASSWORD:keycloak}
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
  flyway:
    enabled: true
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
  servlet:
    multipart:
      max-file-size: ${WIKI_MAX_ATTACHMENT_MB:20}MB
      max-request-size: ${WIKI_MAX_ATTACHMENT_MB:20}MB
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: ${AUTH_JWKS_URI:http://localhost:9000/.well-known/jwks.json}

eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URI:http://localhost:8761/eureka}
  instance:
    prefer-ip-address: true

platform:
  jwt:
    issuer: ${PLATFORM_ISSUER:http://localhost:9000}
    audience: ${PLATFORM_AUDIENCE:platform-api}
  org-grpc:
    host: ${ORG_GRPC_HOST:localhost}
    port: ${ORG_GRPC_PORT:9131}
  wiki:
    files-dir: ${WIKI_FILES_DIR:./data/attachments}
  events:
    enabled: true

---
spring:
  config:
    activate:
      on-profile: docker
eureka:
  client:
    enabled: false
```

`src/test/resources/application-test.yml`:
```yaml
spring:
  datasource:
    # NON_KEYWORDS=KEY: space.key 컬럼 — KEY는 H2 예약어(Postgres에선 비예약). 없으면 create-drop DDL 실패
    url: jdbc:h2:mem:wikidb;DB_CLOSE_DELAY=-1;MODE=PostgreSQL;NON_KEYWORDS=KEY
    driver-class-name: org.h2.Driver
    username: sa
    password: ""
  jpa:
    hibernate:
      ddl-auto: create-drop
  flyway:
    enabled: false
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: http://localhost:9000/.well-known/jwks.json

eureka:
  client:
    enabled: false

platform:
  jwt:
    issuer: http://localhost:9000
    audience: platform-api
  org-grpc:
    host: localhost
    port: 9131
  wiki:
    files-dir: ${java.io.tmpdir}/wiki-test-files
  events:
    enabled: false   # 실 Redis 발행 비활성 — 테스트는 인메모리 페이크 빈
```

`WikiBackendApplicationTests.java`:
```java
package com.platform.wikibackend;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest
@ActiveProfiles("test")
class WikiBackendApplicationTests {
    @Test
    void contextLoads() {
    }
}
```

`.run/bootRun.run.xml` (org-service와 동일 패턴, taskNames만 `bootRun`):
```xml
<component name="ProjectRunConfigurationManager">
  <configuration default="false" name="wiki-backend bootRun" type="GradleRunConfiguration" factoryName="Gradle">
    <ExternalSystemSettings>
      <option name="executionName" />
      <option name="externalProjectPath" value="$PROJECT_DIR$" />
      <option name="externalSystemIdString" value="GRADLE" />
      <option name="scriptParameters" value="" />
      <option name="taskDescriptions"><list /></option>
      <option name="taskNames"><list><option value="bootRun" /></list></option>
      <option name="vmOptions" value="" />
    </ExternalSystemSettings>
    <method v="2" />
  </configuration>
</component>
```

- [ ] **Step 4: contextLoads 확인**

```powershell
$env:JAVA_HOME = 'C:\Program Files\Java\jdk-24'; $env:GITHUB_TOKEN = (gh auth token)
cd C:\MSA_TEMPLATE\wiki-backend; .\gradlew.bat test
```
Expected: PASS. (proto 0.2.0 resolve 실패 시 GITHUB_TOKEN 미설정 여부 확인 — 우회 금지)

- [ ] **Step 5: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\wiki-backend add -A
git -C C:\MSA_TEMPLATE\wiki-backend update-index --chmod=+x gradlew
git -C C:\MSA_TEMPLATE\wiki-backend commit -m "chore: wiki-backend 부트스트랩 — Boot 4.0.6 + common-proto 0.2.0 소비 + 프로필 분기"
```

---

### Task 5: 스키마(Flyway V1) + 엔티티 + 리포지토리

**Files:**
- Create: `wiki-backend/src/main/resources/db/migration/V1__init.sql`
- Create: `src/main/java/com/platform/wikibackend/domain/` — `Space.java`, `Page.java`, `PageRevision.java`, `Attachment.java`
- Create: `src/main/java/com/platform/wikibackend/repository/` — `SpaceRepository.java`, `PageRepository.java`, `PageRevisionRepository.java`, `AttachmentRepository.java`
- Test: `src/test/java/com/platform/wikibackend/repository/PageRepositoryTest.java`

**Interfaces:**
- Produces:
  - `Space.of(key, name, description, createdBy)` / `space.update(name, description)`
  - `Page.of(spaceId, parentId, title, content, authorId)` / `page.edit(title, content, editorId)`(version+1) / `page.moveTo(parentId)`
  - `PageRevision.snapshotOf(Page page)` — 페이지의 **현재 상태**를 리비전으로
  - `Attachment.of(pageId, filename, contentType, sizeBytes, storageKey, uploadedBy)`
  - `PageRepository.findBySpaceIdOrderById(spaceId)` / `findByParentId(parentId)`, `PageRevisionRepository.findByPageIdOrderByVersionDesc(pageId)` / `findByPageIdAndVersion(pageId, version)`, `SpaceRepository.existsByKey(key)`, `AttachmentRepository.findByPageId(pageId)`

- [ ] **Step 1: V1__init.sql**

```sql
-- wiki 코어 스키마. 삭제는 hard delete + cascade (휴지통은 후속 백로그).
CREATE TABLE space (
    id          BIGSERIAL PRIMARY KEY,
    key         VARCHAR(30)  NOT NULL UNIQUE,
    name        VARCHAR(100) NOT NULL,
    description TEXT,
    created_by  BIGINT       NOT NULL,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE TABLE page (
    id         BIGSERIAL PRIMARY KEY,
    space_id   BIGINT       NOT NULL REFERENCES space (id) ON DELETE CASCADE,
    parent_id  BIGINT       REFERENCES page (id) ON DELETE CASCADE,   -- NULL = 루트
    title      VARCHAR(255) NOT NULL,
    content    TEXT         NOT NULL,
    version    INT          NOT NULL DEFAULT 1,
    created_by BIGINT       NOT NULL,
    updated_by BIGINT       NOT NULL,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ  NOT NULL DEFAULT now()
);
CREATE INDEX idx_page_space ON page (space_id);
CREATE INDEX idx_page_parent ON page (parent_id);

CREATE TABLE page_revision (
    id         BIGSERIAL PRIMARY KEY,
    page_id    BIGINT       NOT NULL REFERENCES page (id) ON DELETE CASCADE,
    version    INT          NOT NULL,
    title      VARCHAR(255) NOT NULL,
    content    TEXT         NOT NULL,
    edited_by  BIGINT       NOT NULL,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT now(),
    UNIQUE (page_id, version)
);

CREATE TABLE attachment (
    id           BIGSERIAL PRIMARY KEY,
    page_id      BIGINT       NOT NULL REFERENCES page (id) ON DELETE CASCADE,
    filename     VARCHAR(255) NOT NULL,
    content_type VARCHAR(100) NOT NULL,
    size_bytes   BIGINT       NOT NULL,
    storage_key  VARCHAR(64)  NOT NULL UNIQUE,
    uploaded_by  BIGINT       NOT NULL,
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT now()
);
CREATE INDEX idx_attachment_page ON attachment (page_id);
```

- [ ] **Step 2: 엔티티 4종**

`domain/Space.java`:
```java
package com.platform.wikibackend.domain;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.Instant;

@Entity
@Table(name = "space")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Space {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 30)
    private String key;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(columnDefinition = "text")
    private String description;

    @Column(name = "created_by", nullable = false, updatable = false)
    private Long createdBy;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @UpdateTimestamp
    @Column(nullable = false)
    private Instant updatedAt;

    public static Space of(String key, String name, String description, Long createdBy) {
        Space s = new Space();
        s.key = key;
        s.name = name;
        s.description = description;
        s.createdBy = createdBy;
        return s;
    }

    public void update(String name, String description) {
        this.name = name;
        this.description = description;
    }
}
```

`domain/Page.java`:
```java
package com.platform.wikibackend.domain;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.Instant;

@Entity
@Table(name = "page")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Page {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "space_id", nullable = false, updatable = false)
    private Long spaceId;

    @Column(name = "parent_id")
    private Long parentId; // NULL = 루트

    @Column(nullable = false)
    private String title;

    @Column(nullable = false, columnDefinition = "text")
    private String content;

    @Column(nullable = false)
    private Integer version; // 낙관적 잠금 겸 현재 버전 번호 — 수동 관리(@Version 아님)

    @Column(name = "created_by", nullable = false, updatable = false)
    private Long createdBy;

    @Column(name = "updated_by", nullable = false)
    private Long updatedBy;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @UpdateTimestamp
    @Column(nullable = false)
    private Instant updatedAt;

    public static Page of(Long spaceId, Long parentId, String title, String content, Long authorId) {
        Page p = new Page();
        p.spaceId = spaceId;
        p.parentId = parentId;
        p.title = title;
        p.content = content;
        p.version = 1;
        p.createdBy = authorId;
        p.updatedBy = authorId;
        return p;
    }

    /** 저장 = 새 버전. 호출부(서비스)가 expectedVersion 검사 후 호출한다. */
    public void edit(String title, String content, Long editorId) {
        this.title = title;
        this.content = content;
        this.version = this.version + 1;
        this.updatedBy = editorId;
    }

    public void moveTo(Long newParentId) {
        this.parentId = newParentId;
    }
}
```

`domain/PageRevision.java`:
```java
package com.platform.wikibackend.domain;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;

import java.time.Instant;

@Entity
@Table(name = "page_revision",
        uniqueConstraints = @UniqueConstraint(columnNames = {"page_id", "version"}))
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class PageRevision {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "page_id", nullable = false, updatable = false)
    private Long pageId;

    @Column(nullable = false, updatable = false)
    private Integer version;

    @Column(nullable = false, updatable = false)
    private String title;

    @Column(nullable = false, updatable = false, columnDefinition = "text")
    private String content;

    @Column(name = "edited_by", nullable = false, updatable = false)
    private Long editedBy;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    /** 페이지의 현재 상태를 스냅샷 — "모든 버전이 리비전에 있다" 불변식의 단일 진입점. */
    public static PageRevision snapshotOf(Page page) {
        PageRevision r = new PageRevision();
        r.pageId = page.getId();
        r.version = page.getVersion();
        r.title = page.getTitle();
        r.content = page.getContent();
        r.editedBy = page.getUpdatedBy();
        return r;
    }
}
```

`domain/Attachment.java`:
```java
package com.platform.wikibackend.domain;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;

import java.time.Instant;

@Entity
@Table(name = "attachment")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Attachment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "page_id", nullable = false, updatable = false)
    private Long pageId;

    @Column(nullable = false)
    private String filename;      // 원본 파일명 (표시용)

    @Column(name = "content_type", nullable = false, length = 100)
    private String contentType;

    @Column(name = "size_bytes", nullable = false)
    private Long sizeBytes;

    @Column(name = "storage_key", nullable = false, unique = true, length = 64)
    private String storageKey;    // 디스크 파일명(UUID) — 원본명과 분리(경로 조작 차단)

    @Column(name = "uploaded_by", nullable = false, updatable = false)
    private Long uploadedBy;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    public static Attachment of(Long pageId, String filename, String contentType,
                                Long sizeBytes, String storageKey, Long uploadedBy) {
        Attachment a = new Attachment();
        a.pageId = pageId;
        a.filename = filename;
        a.contentType = contentType;
        a.sizeBytes = sizeBytes;
        a.storageKey = storageKey;
        a.uploadedBy = uploadedBy;
        return a;
    }
}
```

- [ ] **Step 3: 리포지토리 4종**

```java
// repository/SpaceRepository.java
package com.platform.wikibackend.repository;

import com.platform.wikibackend.domain.Space;
import org.springframework.data.jpa.repository.JpaRepository;

public interface SpaceRepository extends JpaRepository<Space, Long> {
    boolean existsByKey(String key);
}
```
```java
// repository/PageRepository.java
package com.platform.wikibackend.repository;

import com.platform.wikibackend.domain.Page;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface PageRepository extends JpaRepository<Page, Long> {
    List<Page> findBySpaceIdOrderById(Long spaceId);
    List<Page> findByParentId(Long parentId);
}
```
```java
// repository/PageRevisionRepository.java
package com.platform.wikibackend.repository;

import com.platform.wikibackend.domain.PageRevision;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;
import java.util.Optional;

public interface PageRevisionRepository extends JpaRepository<PageRevision, Long> {
    List<PageRevision> findByPageIdOrderByVersionDesc(Long pageId);
    Optional<PageRevision> findByPageIdAndVersion(Long pageId, Integer version);
}
```
```java
// repository/AttachmentRepository.java
package com.platform.wikibackend.repository;

import com.platform.wikibackend.domain.Attachment;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface AttachmentRepository extends JpaRepository<Attachment, Long> {
    List<Attachment> findByPageId(Long pageId);
}
```

- [ ] **Step 4: 리포지토리 테스트 (RED→GREEN)**

`repository/PageRepositoryTest.java`:
```java
package com.platform.wikibackend.repository;

import com.platform.wikibackend.domain.Page;
import com.platform.wikibackend.domain.PageRevision;
import com.platform.wikibackend.domain.Space;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@ActiveProfiles("test")
class PageRepositoryTest {

    @Autowired SpaceRepository spaces;
    @Autowired PageRepository pages;
    @Autowired PageRevisionRepository revisions;

    @Test
    void 스페이스별_페이지와_트리_자식을_조회한다() {
        Space s = spaces.save(Space.of("dev", "개발 위키", null, 1L));
        Page root = pages.save(Page.of(s.getId(), null, "루트", "내용", 1L));
        Page child = pages.save(Page.of(s.getId(), root.getId(), "자식", "내용", 1L));

        assertThat(pages.findBySpaceIdOrderById(s.getId())).hasSize(2);
        assertThat(pages.findByParentId(root.getId())).containsExactly(child);
    }

    @Test
    void 리비전_스냅샷은_버전_역순으로_조회된다() {
        Space s = spaces.save(Space.of("ops", "운영", null, 1L));
        Page p = pages.save(Page.of(s.getId(), null, "t", "v1", 1L));
        revisions.save(PageRevision.snapshotOf(p));
        p.edit("t", "v2", 2L);
        pages.save(p);
        revisions.save(PageRevision.snapshotOf(p));

        var list = revisions.findByPageIdOrderByVersionDesc(p.getId());
        assertThat(list).hasSize(2);
        assertThat(list.get(0).getVersion()).isEqualTo(2);
        assertThat(revisions.findByPageIdAndVersion(p.getId(), 1)).isPresent()
                .get().satisfies(r -> assertThat(r.getContent()).isEqualTo("v1"));
    }
}
```
RED 확인 → 구현 존재로 GREEN:
```powershell
.\gradlew.bat test --tests "com.platform.wikibackend.repository.*"
```
Expected: PASS 2건.

- [ ] **Step 5: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\wiki-backend add -A
git -C C:\MSA_TEMPLATE\wiki-backend commit -m "feat: wiki 스키마·엔티티·리포지토리 — space/page/page_revision/attachment"
```

---

### Task 6: 보안 구성 + PermissionClient (gRPC + Caffeine, fail-closed)

**Files:**
- Create: `src/main/java/com/platform/wikibackend/config/SecurityConfig.java`
- Create: `src/main/java/com/platform/wikibackend/security/AudienceValidator.java` (org-service와 동일 — 파일 복제)
- Create: `src/main/java/com/platform/wikibackend/permission/` — `WikiAction.java`, `AccessScope.java`, `PermissionClient.java`, `GrpcPermissionClient.java`
- Create: `src/main/java/com/platform/wikibackend/common/` — `ApiExceptionHandler.java`, `NotFoundException.java`, `ForbiddenException.java`, `ConflictException.java`
- Test: `src/test/java/com/platform/wikibackend/TestAuth.java` (org-service에서 이식), `src/test/java/com/platform/wikibackend/permission/FakePermissionClient.java`, `src/test/java/com/platform/wikibackend/permission/GrpcPermissionClientTest.java`

**Interfaces:**
- Produces:
  - `enum WikiAction { VIEW, EDIT, ADMIN }`
  - `record AccessScope(boolean all, Set<Long> spaceIds)` — `contains(spaceId)` 헬퍼
  - `PermissionClient`: `boolean isAllowed(long userId, long spaceId, WikiAction action)` / `AccessScope accessibleSpaces(long userId)` / `boolean grantSpaceAdmin(long userId, long spaceId)`
  - `FakePermissionClient`(@Component, test 소스셋): `allow(userId, spaceId, action)`, `allowAll(userId)`, `grantedAdmins` 기록 리스트 — Task 7~10 테스트 공용
  - 예외 계약: `NotFoundException`→404, `ForbiddenException`→403, `ConflictException`→409, `IllegalArgumentException`→400
- Consumes: common-proto 0.2.0 `PermissionServiceGrpc`, `CheckPermissionRequest`, `ListUserGrantsRequest`, `CreateGrantRequest`

- [ ] **Step 1: 실패하는 테스트 작성 (GrpcPermissionClient — in-process 서버 스텁)**

`permission/GrpcPermissionClientTest.java`:
```java
package com.platform.wikibackend.permission;

import com.platform.proto.org.v1.*;
import io.grpc.ManagedChannel;
import io.grpc.Server;
import io.grpc.inprocess.InProcessChannelBuilder;
import io.grpc.inprocess.InProcessServerBuilder;
import io.grpc.stub.StreamObserver;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.io.IOException;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

/** GrpcPermissionClient 단위 검증 — in-process 가짜 org 서버로 매핑·캐시·fail-closed를 본다. */
class GrpcPermissionClientTest {

    static class StubOrg extends PermissionServiceGrpc.PermissionServiceImplBase {
        final AtomicInteger checkCalls = new AtomicInteger();
        volatile boolean allow = true;
        volatile boolean fail = false;

        @Override public void checkPermission(CheckPermissionRequest req, StreamObserver<CheckPermissionResponse> out) {
            checkCalls.incrementAndGet();
            if (fail) { out.onError(io.grpc.Status.UNAVAILABLE.asRuntimeException()); return; }
            out.onNext(CheckPermissionResponse.newBuilder().setAllowed(allow).build());
            out.onCompleted();
        }

        @Override public void listUserGrants(ListUserGrantsRequest req, StreamObserver<ListUserGrantsResponse> out) {
            out.onNext(ListUserGrantsResponse.newBuilder()
                    .addGrants(Grant.newBuilder().setResourceType(ResourceType.SPACE).setResourceId("3").setRole(Role.VIEWER))
                    .addGrants(Grant.newBuilder().setResourceType(ResourceType.GLOBAL).setResourceId("").setRole(Role.VIEWER))
                    .build());
            out.onCompleted();
        }

        @Override public void createGrant(CreateGrantRequest req, StreamObserver<CreateGrantResponse> out) {
            out.onNext(CreateGrantResponse.newBuilder().setCreated(true).build());
            out.onCompleted();
        }
    }

    StubOrg stubOrg = new StubOrg();
    Server server;
    ManagedChannel channel;
    GrpcPermissionClient client;

    @BeforeEach
    void setup() throws IOException {
        String name = InProcessServerBuilder.generateName();
        server = InProcessServerBuilder.forName(name).directExecutor().addService(stubOrg).build().start();
        channel = InProcessChannelBuilder.forName(name).directExecutor().build();
        client = new GrpcPermissionClient(PermissionServiceGrpc.newBlockingStub(channel));
    }

    @AfterEach
    void teardown() throws InterruptedException {
        channel.shutdownNow(); server.shutdownNow();
        channel.awaitTermination(3, java.util.concurrent.TimeUnit.SECONDS);
        server.awaitTermination();
    }

    @Test
    void 허용_판정이_전달되고_30초_캐시로_중복호출이_제거된다() {
        assertThat(client.isAllowed(1L, 5L, WikiAction.EDIT)).isTrue();
        assertThat(client.isAllowed(1L, 5L, WikiAction.EDIT)).isTrue(); // 캐시 히트
        assertThat(stubOrg.checkCalls.get()).isEqualTo(1);
    }

    @Test
    void org_불능이면_fail_closed_false() {
        stubOrg.fail = true;
        assertThat(client.isAllowed(2L, 5L, WikiAction.VIEW)).isFalse();
    }

    @Test
    void accessibleSpaces는_GLOBAL_grant를_all로_해석한다() {
        AccessScope scope = client.accessibleSpaces(1L);
        assertThat(scope.all()).isTrue();
    }

    @Test
    void grantSpaceAdmin은_CreateGrant를_호출한다() {
        assertThat(client.grantSpaceAdmin(1L, 9L)).isTrue();
    }
}
```

- [ ] **Step 2: RED 확인**

```powershell
.\gradlew.bat test --tests "com.platform.wikibackend.permission.*"
```
Expected: FAIL — 심볼 없음.

- [ ] **Step 3: 구현**

`permission/WikiAction.java`:
```java
package com.platform.wikibackend.permission;

public enum WikiAction { VIEW, EDIT, ADMIN }
```

`permission/AccessScope.java`:
```java
package com.platform.wikibackend.permission;

import java.util.Set;

/** 사용자가 접근 가능한 스페이스 범위. all=true면 전 스페이스(GLOBAL grant 보유자). */
public record AccessScope(boolean all, Set<Long> spaceIds) {
    public boolean contains(long spaceId) {
        return all || spaceIds.contains(spaceId);
    }
}
```

`permission/PermissionClient.java`:
```java
package com.platform.wikibackend.permission;

/** org-service 권한 연동 창구 — 테스트는 페이크로 대체. */
public interface PermissionClient {
    boolean isAllowed(long userId, long spaceId, WikiAction action);
    AccessScope accessibleSpaces(long userId);
    /** 스페이스 생성자 자동 ADMIN 부여. 이미 있으면 false(멱등). 실패 시 예외 아님 — false. */
    boolean grantSpaceAdmin(long userId, long spaceId);
}
```

`permission/GrpcPermissionClient.java`:
```java
package com.platform.wikibackend.permission;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import com.platform.proto.org.v1.*;
import lombok.extern.slf4j.Slf4j;

import java.time.Duration;
import java.util.Set;
import java.util.stream.Collectors;

/**
 * org-service gRPC 권한 클라이언트. 판정은 30초 Caffeine 캐시(권한 회수 반영 지연 30초 수용 — 스펙).
 * org 불능 시 fail-closed(false) — 가용성보다 인가 안전 우선.
 */
@Slf4j
public class GrpcPermissionClient implements PermissionClient {

    private record CacheKey(long userId, long spaceId, WikiAction action) {}

    private final PermissionServiceGrpc.PermissionServiceBlockingStub stub;
    private final Cache<CacheKey, Boolean> cache = Caffeine.newBuilder()
            .expireAfterWrite(Duration.ofSeconds(30))
            .maximumSize(10_000)
            .build();

    public GrpcPermissionClient(PermissionServiceGrpc.PermissionServiceBlockingStub stub) {
        this.stub = stub;
    }

    @Override
    public boolean isAllowed(long userId, long spaceId, WikiAction action) {
        return cache.get(new CacheKey(userId, spaceId, action), k -> {
            try {
                return stub.checkPermission(CheckPermissionRequest.newBuilder()
                        .setUserId(k.userId())
                        .setResourceType(ResourceType.SPACE)
                        .setResourceId(String.valueOf(k.spaceId()))
                        .setAction(toProto(k.action()))
                        .build()).getAllowed();
            } catch (Exception e) {
                log.warn("권한조회 실패 — fail-closed: user={} space={} action={}", k.userId(), k.spaceId(), k.action(), e);
                return false;
            }
        });
    }

    @Override
    public AccessScope accessibleSpaces(long userId) {
        try {
            ListUserGrantsResponse res = stub.listUserGrants(
                    ListUserGrantsRequest.newBuilder().setUserId(userId).build()); // UNSPECIFIED = 전체
            boolean global = res.getGrantsList().stream()
                    .anyMatch(g -> g.getResourceType() == ResourceType.GLOBAL);
            if (global) return new AccessScope(true, Set.of());
            Set<Long> ids = res.getGrantsList().stream()
                    .filter(g -> g.getResourceType() == ResourceType.SPACE)
                    .map(g -> Long.parseLong(g.getResourceId()))
                    .collect(Collectors.toSet());
            return new AccessScope(false, ids);
        } catch (Exception e) {
            log.warn("grant 목록 조회 실패 — fail-closed(빈 목록): user={}", userId, e);
            return new AccessScope(false, Set.of());
        }
    }

    @Override
    public boolean grantSpaceAdmin(long userId, long spaceId) {
        try {
            return stub.createGrant(CreateGrantRequest.newBuilder()
                    .setUserId(userId)
                    .setResourceType(ResourceType.SPACE)
                    .setResourceId(String.valueOf(spaceId))
                    .setRole(Role.ROLE_ADMIN)
                    .build()).getCreated();
        } catch (Exception e) {
            log.warn("자동 ADMIN 부여 실패: user={} space={}", userId, spaceId, e);
            return false;
        }
    }

    private static Action toProto(WikiAction a) {
        return switch (a) {
            case VIEW -> Action.VIEW;
            case EDIT -> Action.EDIT;
            case ADMIN -> Action.ADMIN;
        };
    }
}
```

`config/SecurityConfig.java` (org-service 패턴 + PermissionClient 빈 — JIT 필터 없음):
```java
package com.platform.wikibackend.config;

import com.platform.proto.org.v1.PermissionServiceGrpc;
import com.platform.wikibackend.permission.GrpcPermissionClient;
import com.platform.wikibackend.permission.PermissionClient;
import com.platform.wikibackend.security.AudienceValidator;
import io.grpc.ManagedChannelBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.oauth2.core.DelegatingOAuth2TokenValidator;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.security.oauth2.jwt.JwtValidators;
import org.springframework.security.oauth2.jwt.NimbusJwtDecoder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    JwtDecoder jwtDecoder(
            @Value("${spring.security.oauth2.resourceserver.jwt.jwk-set-uri}") String jwkSetUri,
            @Value("${platform.jwt.issuer}") String issuer,
            @Value("${platform.jwt.audience}") String audience) {
        NimbusJwtDecoder decoder = NimbusJwtDecoder.withJwkSetUri(jwkSetUri).build();
        decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<Jwt>(
                JwtValidators.createDefaultWithIssuer(issuer),
                new AudienceValidator(audience)));
        return decoder;
    }

    @Bean
    JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authorities = new JwtGrantedAuthoritiesConverter();
        authorities.setAuthoritiesClaimName("roles");
        authorities.setAuthorityPrefix("ROLE_");
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authorities);
        return converter;
    }

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http, JwtAuthenticationConverter converter) throws Exception {
        http
                .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .csrf(csrf -> csrf.disable())
                // 위키는 공개 엔드포인트 없음 — 조회 권한도 스페이스 단위 VIEW로 서비스 계층에서 판정
                .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(converter)));
        return http.build();
    }

    /** 테스트는 FakePermissionClient 빈이 이 빈을 대체한다(@ConditionalOnMissingBean). */
    @Bean
    @ConditionalOnMissingBean(PermissionClient.class)
    PermissionClient permissionClient(
            @Value("${platform.org-grpc.host}") String host,
            @Value("${platform.org-grpc.port}") int port) {
        var channel = ManagedChannelBuilder.forAddress(host, port).usePlaintext().build();
        return new GrpcPermissionClient(PermissionServiceGrpc.newBlockingStub(channel));
    }
}
```

`security/AudienceValidator.java` — org-service의 동일 파일을 패키지만 바꿔 복제:
```java
package com.platform.wikibackend.security;

import org.springframework.security.oauth2.core.OAuth2Error;
import org.springframework.security.oauth2.core.OAuth2TokenValidator;
import org.springframework.security.oauth2.core.OAuth2TokenValidatorResult;
import org.springframework.security.oauth2.jwt.Jwt;

public class AudienceValidator implements OAuth2TokenValidator<Jwt> {

    private final String audience;

    public AudienceValidator(String audience) { this.audience = audience; }

    @Override
    public OAuth2TokenValidatorResult validate(Jwt jwt) {
        if (jwt.getAudience() != null && jwt.getAudience().contains(audience)) {
            return OAuth2TokenValidatorResult.success();
        }
        return OAuth2TokenValidatorResult.failure(
                new OAuth2Error("invalid_token", "audience 불일치", null));
    }
}
```

`common/NotFoundException.java` / `ForbiddenException.java` / `ConflictException.java`:
```java
package com.platform.wikibackend.common;

public class NotFoundException extends RuntimeException {
    public NotFoundException(String message) { super(message); }
}
```
```java
package com.platform.wikibackend.common;

public class ForbiddenException extends RuntimeException {
    public ForbiddenException(String message) { super(message); }
}
```
```java
package com.platform.wikibackend.common;

public class ConflictException extends RuntimeException {
    public ConflictException(String message) { super(message); }
}
```

`common/ApiExceptionHandler.java`:
```java
package com.platform.wikibackend.common;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.Map;

@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, String> notFound(NotFoundException e) { return Map.of("error", e.getMessage()); }

    @ExceptionHandler(ForbiddenException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public Map<String, String> forbidden(ForbiddenException e) { return Map.of("error", e.getMessage()); }

    @ExceptionHandler(ConflictException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public Map<String, String> conflict(ConflictException e) { return Map.of("error", e.getMessage()); }

    @ExceptionHandler(IllegalArgumentException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, String> badRequest(IllegalArgumentException e) { return Map.of("error", e.getMessage()); }
}
```

테스트 소스셋 — `src/test/java/com/platform/wikibackend/TestAuth.java` (org-service와 동일 내용, 패키지만 `com.platform.wikibackend`):
```java
package com.platform.wikibackend;

import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.test.web.servlet.request.RequestPostProcessor;

import java.util.List;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;

public class TestAuth {

    public static RequestPostProcessor asUser(long id, String name) {
        return jwt().jwt(j -> j.subject(String.valueOf(id)).claim("name", name)
                        .claim("email", name.toLowerCase() + "@test.com").claim("roles", List.of("USER")))
                .authorities(new SimpleGrantedAuthority("ROLE_USER"));
    }
}
```

`src/test/java/com/platform/wikibackend/permission/FakePermissionClient.java`:
```java
package com.platform.wikibackend.permission;

import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

/**
 * 테스트 전용 페이크 — test 소스셋의 컴포넌트 스캔으로 등록된다.
 * @Primary: SecurityConfig의 @ConditionalOnMissingBean은 빈 등록 순서에 따라 gRPC 빈이
 * 함께 생길 수 있으므로(알려진 fragility), 주입 우선권으로 확실히 대체한다.
 */
@Component
@org.springframework.context.annotation.Primary
public class FakePermissionClient implements PermissionClient {

    private record Key(long userId, long spaceId, WikiAction action) {}

    private final Set<Key> allowed = new HashSet<>();
    private final Set<Long> allowAllUsers = new HashSet<>();
    public final List<long[]> grantedAdmins = new ArrayList<>(); // [userId, spaceId] 기록

    public void allow(long userId, long spaceId, WikiAction action) { allowed.add(new Key(userId, spaceId, action)); }
    public void allowAll(long userId) { allowAllUsers.add(userId); }
    public void reset() { allowed.clear(); allowAllUsers.clear(); grantedAdmins.clear(); }

    @Override
    public boolean isAllowed(long userId, long spaceId, WikiAction action) {
        return allowAllUsers.contains(userId) || allowed.contains(new Key(userId, spaceId, action));
    }

    @Override
    public AccessScope accessibleSpaces(long userId) {
        if (allowAllUsers.contains(userId)) return new AccessScope(true, Set.of());
        Set<Long> ids = new HashSet<>();
        for (Key k : allowed) if (k.userId() == userId && k.action() == WikiAction.VIEW) ids.add(k.spaceId());
        return new AccessScope(false, ids);
    }

    @Override
    public boolean grantSpaceAdmin(long userId, long spaceId) {
        grantedAdmins.add(new long[]{userId, spaceId});
        allow(userId, spaceId, WikiAction.VIEW);
        allow(userId, spaceId, WikiAction.EDIT);
        allow(userId, spaceId, WikiAction.ADMIN);
        return true;
    }
}
```

- [ ] **Step 4: GREEN + 전체 회귀**

```powershell
.\gradlew.bat test
```
Expected: PASS (permission 4건 + 기존).

- [ ] **Step 5: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\wiki-backend add -A
git -C C:\MSA_TEMPLATE\wiki-backend commit -m "feat: JWT 리소스 서버 + PermissionClient(gRPC+Caffeine 30s, fail-closed)"
```

---

### Task 7: EventPublisher + Redis Streams 발행

**Files:**
- Create: `src/main/java/com/platform/wikibackend/event/` — `EventPublisher.java`, `RedisStreamEventPublisher.java`, `EventRelay.java`, `WikiEvents.java`
- Test: `src/test/java/com/platform/wikibackend/event/RecordingEventPublisher.java`, `src/test/java/com/platform/wikibackend/event/EventRelayTest.java`

**Interfaces:**
- Produces:
  - `EventPublisher`: `void publish(EventEnvelope event)`
  - `EventRelay`: `void afterCommit(EventEnvelope event)` — 트랜잭션 커밋 후(비트랜잭션 문맥은 즉시) EventPublisher로 위임, 실패는 WARN 비차단
  - `WikiEvents` 정적 팩토리: `spaceCreated(actorId, space)`, `spaceDeleted(actorId, spaceId)`, `pageCreated(actorId, page)`, `pageUpdated(actorId, page)`, `pageDeleted(actorId, pageId, spaceId)`, `attachmentAdded(actorId, attachment, spaceId)` → `EventEnvelope`
  - `RecordingEventPublisher`(test @Component — @ConditionalOnMissingBean 대체): `List<EventEnvelope> events`, `reset()`
- Consumes: common-proto `EventEnvelope` 등, Task 5 엔티티

- [ ] **Step 1: 실패하는 테스트 (EventRelay — 커밋 후 발행)**

`event/EventRelayTest.java`:
```java
package com.platform.wikibackend.event;

import com.platform.proto.events.v1.EventEnvelope;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.transaction.support.TransactionTemplate;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@ActiveProfiles("test")
class EventRelayTest {

    @Autowired EventRelay relay;
    @Autowired RecordingEventPublisher recording;
    @Autowired org.springframework.transaction.PlatformTransactionManager txm;
    TransactionTemplate tx; // TransactionTemplate은 자동구성 빈이 아님 — 직접 생성

    @BeforeEach
    void reset() {
        tx = new TransactionTemplate(txm);
        recording.reset();
    }

    private static EventEnvelope sample() {
        return EventEnvelope.newBuilder().setEventId("e-1").setActorId(1L).setSource("wiki-backend").build();
    }

    @Test
    void 트랜잭션_안에서는_커밋_후에_발행된다() {
        tx.executeWithoutResult(status -> {
            relay.afterCommit(sample());
            assertThat(recording.events).isEmpty(); // 아직 커밋 전
        });
        assertThat(recording.events).hasSize(1);
    }

    @Test
    void 롤백되면_발행되지_않는다() {
        tx.executeWithoutResult(status -> {
            relay.afterCommit(sample());
            status.setRollbackOnly();
        });
        assertThat(recording.events).isEmpty();
    }

    @Test
    void 비트랜잭션_문맥에서는_즉시_발행된다() {
        relay.afterCommit(sample());
        assertThat(recording.events).hasSize(1);
    }
}
```

- [ ] **Step 2: RED 확인**

```powershell
.\gradlew.bat test --tests "com.platform.wikibackend.event.*"
```
Expected: FAIL — 심볼 없음.

- [ ] **Step 3: 구현**

`event/EventPublisher.java`:
```java
package com.platform.wikibackend.event;

import com.platform.proto.events.v1.EventEnvelope;

/** 이벤트 버스 창구 — 현재 구현은 Redis Streams. L 티어에서 Kafka 교체 시 이 계약 뒤만 바뀐다. */
public interface EventPublisher {
    void publish(EventEnvelope event);
}
```

`event/RedisStreamEventPublisher.java`:
```java
package com.platform.wikibackend.event;

import com.platform.proto.events.v1.EventEnvelope;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.data.redis.connection.RedisConnection;
import org.springframework.data.redis.connection.stream.MapRecord;
import org.springframework.data.redis.connection.stream.RecordId;
import org.springframework.data.redis.core.RedisCallback;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import java.nio.charset.StandardCharsets;
import java.util.Map;

import static org.springframework.data.redis.connection.RedisStreamCommands.XAddOptions;

/** XADD platform:events:v1 MAXLEN ~100000 — payload는 proto 바이너리. */
@Component
@ConditionalOnProperty(value = "platform.events.enabled", havingValue = "true", matchIfMissing = true)
public class RedisStreamEventPublisher implements EventPublisher {

    public static final String STREAM = "platform:events:v1";
    private static final long MAXLEN = 100_000;

    private final StringRedisTemplate redis;

    public RedisStreamEventPublisher(StringRedisTemplate redis) { this.redis = redis; }

    @Override
    public void publish(EventEnvelope event) {
        byte[] stream = STREAM.getBytes(StandardCharsets.UTF_8);
        Map<byte[], byte[]> fields = Map.of("payload".getBytes(StandardCharsets.UTF_8), event.toByteArray());
        redis.execute((RedisCallback<RecordId>) (RedisConnection conn) ->
                conn.streamCommands().xAdd(MapRecord.create(stream, fields),
                        XAddOptions.maxlen(MAXLEN).approximateTrimming(true)));
    }
}
```

`event/EventRelay.java`:
```java
package com.platform.wikibackend.event;

import com.platform.proto.events.v1.EventEnvelope;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.transaction.support.TransactionSynchronization;
import org.springframework.transaction.support.TransactionSynchronizationManager;

/**
 * 커밋 후 발행 릴레이. 트랜잭션 문맥이면 afterCommit 훅에, 아니면 즉시.
 * 발행 실패는 WARN 비차단 — 정합의 최종 근거는 DB, 색인은 Wave C 재색인으로 보정(스펙).
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class EventRelay {

    private final EventPublisher publisher;

    public void afterCommit(EventEnvelope event) {
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
                @Override public void afterCommit() { safePublish(event); }
            });
        } else {
            safePublish(event);
        }
    }

    private void safePublish(EventEnvelope event) {
        try {
            publisher.publish(event);
        } catch (Exception e) {
            log.warn("이벤트 발행 실패(비차단): type={} id={}", event.getPayloadCase(), event.getEventId(), e);
        }
    }
}
```

`event/WikiEvents.java`:
```java
package com.platform.wikibackend.event;

import com.platform.proto.events.v1.*;
import com.platform.wikibackend.domain.Attachment;
import com.platform.wikibackend.domain.Page;
import com.platform.wikibackend.domain.Space;

import java.util.UUID;

/** EventEnvelope 조립 — 본문(content)은 싣지 않는다(스펙). */
public final class WikiEvents {

    private WikiEvents() {}

    private static EventEnvelope.Builder base(long actorId) {
        return EventEnvelope.newBuilder()
                .setEventId(UUID.randomUUID().toString())
                .setOccurredAt(System.currentTimeMillis())
                .setActorId(actorId)
                .setSource("wiki-backend");
    }

    public static EventEnvelope spaceCreated(long actorId, Space s) {
        return base(actorId).setSpaceCreated(SpaceCreated.newBuilder()
                .setSpaceId(s.getId()).setKey(s.getKey()).setName(s.getName())).build();
    }

    public static EventEnvelope spaceDeleted(long actorId, long spaceId) {
        return base(actorId).setSpaceDeleted(SpaceDeleted.newBuilder().setSpaceId(spaceId)).build();
    }

    public static EventEnvelope pageCreated(long actorId, Page p) {
        return base(actorId).setPageCreated(PageCreated.newBuilder()
                .setPageId(p.getId()).setSpaceId(p.getSpaceId()).setTitle(p.getTitle())).build();
    }

    public static EventEnvelope pageUpdated(long actorId, Page p) {
        return base(actorId).setPageUpdated(PageUpdated.newBuilder()
                .setPageId(p.getId()).setSpaceId(p.getSpaceId()).setTitle(p.getTitle())
                .setVersion(p.getVersion())).build();
    }

    public static EventEnvelope pageDeleted(long actorId, long pageId, long spaceId) {
        return base(actorId).setPageDeleted(PageDeleted.newBuilder()
                .setPageId(pageId).setSpaceId(spaceId)).build();
    }

    public static EventEnvelope attachmentAdded(long actorId, Attachment a, long spaceId) {
        return base(actorId).setAttachmentAdded(AttachmentAdded.newBuilder()
                .setAttachmentId(a.getId()).setPageId(a.getPageId()).setSpaceId(spaceId)
                .setFilename(a.getFilename())).build();
    }
}
```

`src/test/java/com/platform/wikibackend/event/RecordingEventPublisher.java`:
```java
package com.platform.wikibackend.event;

import com.platform.proto.events.v1.EventEnvelope;
import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.List;

/** 테스트 전용 — platform.events.enabled=false로 Redis 구현이 빠진 자리를 채운다. */
@Component
public class RecordingEventPublisher implements EventPublisher {

    public final List<EventEnvelope> events = new ArrayList<>();

    public void reset() { events.clear(); }

    @Override
    public void publish(EventEnvelope event) { events.add(event); }
}
```

- [ ] **Step 4: GREEN**

```powershell
.\gradlew.bat test --tests "com.platform.wikibackend.event.*"
```
Expected: PASS 3건.

- [ ] **Step 5: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\wiki-backend add -A
git -C C:\MSA_TEMPLATE\wiki-backend commit -m "feat: EventPublisher 추상화 + Redis Streams 발행(커밋 후·비차단)"
```

---

### Task 8: 스페이스 REST (목록 필터·생성+자동 ADMIN·수정·삭제)

**Files:**
- Create: `src/main/java/com/platform/wikibackend/space/` — `SpaceController.java`, `SpaceService.java`, `dto/SpaceCreateRequest.java`, `dto/SpaceUpdateRequest.java`, `dto/SpaceResponse.java`
- Test: `src/test/java/com/platform/wikibackend/space/SpaceControllerTest.java`

**Interfaces:**
- Consumes: `PermissionClient`(isAllowed·accessibleSpaces·grantSpaceAdmin), `EventRelay.afterCommit`, `WikiEvents.spaceCreated/spaceDeleted`, `SpaceRepository`, `TestAuth.asUser`, `FakePermissionClient`, `RecordingEventPublisher`
- Produces:
  - REST: `GET /api/wiki/spaces`(접근 가능만), `POST /api/wiki/spaces`(201, key 중복 400), `GET /{id}`(VIEW), `PUT /{id}`(ADMIN), `DELETE /{id}`(ADMIN, 204)
  - `SpaceService.getForView(userId, spaceId)` — VIEW 가드 포함 조회(Task 9~10 재사용)
  - `SpaceResponse(Long id, String key, String name, String description)`

- [ ] **Step 1: 실패하는 테스트**

`space/SpaceControllerTest.java`:
```java
package com.platform.wikibackend.space;

import com.platform.wikibackend.domain.Space;
import com.platform.wikibackend.event.RecordingEventPublisher;
import com.platform.wikibackend.permission.FakePermissionClient;
import com.platform.wikibackend.permission.WikiAction;
import com.platform.wikibackend.repository.SpaceRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import static com.platform.wikibackend.TestAuth.asUser;
import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@ActiveProfiles("test")
class SpaceControllerTest {

    @Autowired WebApplicationContext context;
    @Autowired SpaceRepository spaces;
    @Autowired FakePermissionClient perms;
    @Autowired RecordingEventPublisher events;
    MockMvc mvc;

    @BeforeEach
    void setup() {
        mvc = MockMvcBuilders.webAppContextSetup(context).apply(springSecurity()).build();
        spaces.deleteAll();
        perms.reset();
        events.reset();
    }

    @Test
    void 무토큰은_401() throws Exception {
        mvc.perform(get("/api/wiki/spaces")).andExpect(status().isUnauthorized());
    }

    @Test
    void 생성하면_생성자에게_ADMIN이_자동_부여되고_이벤트가_발행된다() throws Exception {
        mvc.perform(post("/api/wiki/spaces").with(asUser(1L, "Alice"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"key\":\"dev\",\"name\":\"개발 위키\",\"description\":null}"))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.key").value("dev"));

        assertThat(perms.grantedAdmins).hasSize(1);
        assertThat(events.events).hasSize(1);
        assertThat(events.events.get(0).hasSpaceCreated()).isTrue();
    }

    @Test
    void key_중복은_400() throws Exception {
        mvc.perform(post("/api/wiki/spaces").with(asUser(1L, "Alice"))
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"key\":\"dev\",\"name\":\"a\",\"description\":null}")).andExpect(status().isCreated());
        mvc.perform(post("/api/wiki/spaces").with(asUser(1L, "Alice"))
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"key\":\"dev\",\"name\":\"b\",\"description\":null}")).andExpect(status().isBadRequest());
    }

    @Test
    void 목록은_접근_가능한_스페이스만_반환한다() throws Exception {
        Space visible = spaces.save(Space.of("a", "보임", null, 9L));
        spaces.save(Space.of("b", "안보임", null, 9L));
        perms.allow(2L, visible.getId(), WikiAction.VIEW);

        mvc.perform(get("/api/wiki/spaces").with(asUser(2L, "Bob")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.length()").value(1))
                .andExpect(jsonPath("$[0].key").value("a"));
    }

    @Test
    void 수정과_삭제는_ADMIN만_가능하다() throws Exception {
        Space s = spaces.save(Space.of("ops", "운영", null, 9L));
        perms.allow(2L, s.getId(), WikiAction.VIEW); // VIEW만

        mvc.perform(put("/api/wiki/spaces/" + s.getId()).with(asUser(2L, "Bob"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"name\":\"x\",\"description\":null}"))
                .andExpect(status().isForbidden());

        perms.allow(2L, s.getId(), WikiAction.ADMIN);
        mvc.perform(put("/api/wiki/spaces/" + s.getId()).with(asUser(2L, "Bob"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"name\":\"x\",\"description\":null}"))
                .andExpect(status().isOk());
        mvc.perform(delete("/api/wiki/spaces/" + s.getId()).with(asUser(2L, "Bob")))
                .andExpect(status().isNoContent());
        assertThat(events.events.stream().filter(e -> e.hasSpaceDeleted())).hasSize(1);
    }
}
```

- [ ] **Step 2: RED 확인**

```powershell
.\gradlew.bat test --tests "com.platform.wikibackend.space.*"
```
Expected: FAIL — 심볼 없음.

- [ ] **Step 3: 구현**

`space/dto/SpaceCreateRequest.java`:
```java
package com.platform.wikibackend.space.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;

public record SpaceCreateRequest(
        @NotBlank @Size(max = 30) @Pattern(regexp = "[a-z0-9-]+", message = "key는 소문자·숫자·하이픈만") String key,
        @NotBlank @Size(max = 100) String name,
        String description) {}
```

`space/dto/SpaceUpdateRequest.java`:
```java
package com.platform.wikibackend.space.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record SpaceUpdateRequest(@NotBlank @Size(max = 100) String name, String description) {}
```

`space/dto/SpaceResponse.java`:
```java
package com.platform.wikibackend.space.dto;

import com.platform.wikibackend.domain.Space;

public record SpaceResponse(Long id, String key, String name, String description) {
    public static SpaceResponse from(Space s) {
        return new SpaceResponse(s.getId(), s.getKey(), s.getName(), s.getDescription());
    }
}
```

`space/SpaceService.java`:
```java
package com.platform.wikibackend.space;

import com.platform.wikibackend.common.ForbiddenException;
import com.platform.wikibackend.common.NotFoundException;
import com.platform.wikibackend.domain.Space;
import com.platform.wikibackend.event.EventRelay;
import com.platform.wikibackend.event.WikiEvents;
import com.platform.wikibackend.permission.AccessScope;
import com.platform.wikibackend.permission.PermissionClient;
import com.platform.wikibackend.permission.WikiAction;
import com.platform.wikibackend.repository.SpaceRepository;
import com.platform.wikibackend.space.dto.SpaceCreateRequest;
import com.platform.wikibackend.space.dto.SpaceResponse;
import com.platform.wikibackend.space.dto.SpaceUpdateRequest;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
@Transactional
public class SpaceService {

    private final SpaceRepository spaces;
    private final PermissionClient permissions;
    private final EventRelay events;

    @Transactional(readOnly = true)
    public List<SpaceResponse> listAccessible(long userId) {
        AccessScope scope = permissions.accessibleSpaces(userId);
        return spaces.findAll().stream()
                .filter(s -> scope.contains(s.getId()))
                .map(SpaceResponse::from)
                .toList();
    }

    public SpaceResponse create(long userId, SpaceCreateRequest req) {
        if (spaces.existsByKey(req.key())) throw new IllegalArgumentException("이미 존재하는 key: " + req.key());
        Space saved = spaces.save(Space.of(req.key(), req.name(), req.description(), userId));
        // 생성자 자동 ADMIN — 실패해도 스페이스 생성은 유효(관리자가 grants REST로 수습 가능), 로그는 클라이언트가 남김
        permissions.grantSpaceAdmin(userId, saved.getId());
        events.afterCommit(WikiEvents.spaceCreated(userId, saved));
        return SpaceResponse.from(saved);
    }

    /** VIEW 가드 포함 단건 조회 — 페이지·첨부 서비스가 재사용. */
    @Transactional(readOnly = true)
    public Space getForView(long userId, long spaceId) {
        Space s = spaces.findById(spaceId).orElseThrow(() -> new NotFoundException("스페이스 없음: " + spaceId));
        require(userId, spaceId, WikiAction.VIEW);
        return s;
    }

    public SpaceResponse update(long userId, long spaceId, SpaceUpdateRequest req) {
        Space s = spaces.findById(spaceId).orElseThrow(() -> new NotFoundException("스페이스 없음: " + spaceId));
        require(userId, spaceId, WikiAction.ADMIN);
        s.update(req.name(), req.description());
        return SpaceResponse.from(s);
    }

    public void delete(long userId, long spaceId) {
        if (!spaces.existsById(spaceId)) throw new NotFoundException("스페이스 없음: " + spaceId);
        require(userId, spaceId, WikiAction.ADMIN);
        spaces.deleteById(spaceId);
        events.afterCommit(WikiEvents.spaceDeleted(userId, spaceId));
    }

    /** 다른 서비스(페이지·첨부)도 쓰는 공용 가드. */
    public void require(long userId, long spaceId, WikiAction action) {
        if (!permissions.isAllowed(userId, spaceId, action)) {
            throw new ForbiddenException(action + " 권한이 필요합니다 (space " + spaceId + ")");
        }
    }
}
```

`space/SpaceController.java`:
```java
package com.platform.wikibackend.space;

import com.platform.wikibackend.space.dto.SpaceCreateRequest;
import com.platform.wikibackend.space.dto.SpaceResponse;
import com.platform.wikibackend.space.dto.SpaceUpdateRequest;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/wiki/spaces")
@RequiredArgsConstructor
public class SpaceController {

    private final SpaceService spaces;

    @GetMapping
    public List<SpaceResponse> list(@AuthenticationPrincipal Jwt jwt) {
        return spaces.listAccessible(userId(jwt));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public SpaceResponse create(@AuthenticationPrincipal Jwt jwt, @Valid @RequestBody SpaceCreateRequest req) {
        return spaces.create(userId(jwt), req);
    }

    @GetMapping("/{id}")
    public SpaceResponse get(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id) {
        return SpaceResponse.from(spaces.getForView(userId(jwt), id));
    }

    @PutMapping("/{id}")
    public SpaceResponse update(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id,
                                @Valid @RequestBody SpaceUpdateRequest req) {
        return spaces.update(userId(jwt), id, req);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id) {
        spaces.delete(userId(jwt), id);
    }

    /** page·attachment 컨트롤러가 import static으로 공용 — 반드시 public. */
    public static long userId(Jwt jwt) { return Long.parseLong(jwt.getSubject()); }
}
```

- [ ] **Step 4: GREEN + 전체 회귀**

```powershell
.\gradlew.bat test
```
Expected: 전체 PASS (space 5건 포함).

- [ ] **Step 5: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\wiki-backend add -A
git -C C:\MSA_TEMPLATE\wiki-backend commit -m "feat: 스페이스 REST — 접근 필터 목록·생성+자동 ADMIN·수정/삭제 가드·이벤트"
```

---

### Task 9: 페이지 CRUD + 트리 (same-space·순환 검증)

**Files:**
- Create: `src/main/java/com/platform/wikibackend/page/` — `PageController.java`, `PageService.java`, `dto/PageCreateRequest.java`, `dto/PageUpdateRequest.java`, `dto/PageResponse.java`, `dto/PageTreeItem.java`
- Test: `src/test/java/com/platform/wikibackend/page/PageControllerTest.java`

**Interfaces:**
- Consumes: `SpaceService.require/getForView`, `PageRepository`, `PageRevisionRepository`, `EventRelay`, `WikiEvents.pageCreated/pageUpdated/pageDeleted`, `FakePermissionClient`, `RecordingEventPublisher`
- Produces:
  - REST: `POST /api/wiki/pages`(201), `GET /api/wiki/pages/{id}`(본문 포함), `PUT /{id}`(expectedVersion — Task 10에서 409 로직 완성, 이번엔 생성·조회·삭제·트리·이동), `DELETE /{id}`(204), `GET /api/wiki/spaces/{spaceId}/pages`(트리 항목)
  - `PageService.getOwned(pageId)` — 존재 검증만(404), `PageResponse(Long id, Long spaceId, Long parentId, String title, String content, Integer version)`
  - `PageTreeItem(Long id, Long parentId, String title)`

- [ ] **Step 1: 실패하는 테스트**

`page/PageControllerTest.java`:
```java
package com.platform.wikibackend.page;

import com.platform.wikibackend.domain.Page;
import com.platform.wikibackend.domain.Space;
import com.platform.wikibackend.event.RecordingEventPublisher;
import com.platform.wikibackend.permission.FakePermissionClient;
import com.platform.wikibackend.permission.WikiAction;
import com.platform.wikibackend.repository.PageRepository;
import com.platform.wikibackend.repository.PageRevisionRepository;
import com.platform.wikibackend.repository.SpaceRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import static com.platform.wikibackend.TestAuth.asUser;
import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@ActiveProfiles("test")
class PageControllerTest {

    @Autowired WebApplicationContext context;
    @Autowired SpaceRepository spaces;
    @Autowired PageRepository pages;
    @Autowired PageRevisionRepository revisions;
    @Autowired FakePermissionClient perms;
    @Autowired RecordingEventPublisher events;
    MockMvc mvc;

    Space space;
    static final long EDITOR = 1L;
    static final long VIEWER = 2L;

    @BeforeEach
    void setup() {
        mvc = MockMvcBuilders.webAppContextSetup(context).apply(springSecurity()).build();
        pages.deleteAll();
        spaces.deleteAll();
        perms.reset();
        events.reset();
        space = spaces.save(Space.of("dev", "개발", null, EDITOR));
        perms.allow(EDITOR, space.getId(), WikiAction.VIEW);
        perms.allow(EDITOR, space.getId(), WikiAction.EDIT);
        perms.allow(VIEWER, space.getId(), WikiAction.VIEW);
    }

    private long createPage(Long parentId, String title) throws Exception {
        String parent = parentId == null ? "null" : String.valueOf(parentId);
        String body = mvc.perform(post("/api/wiki/pages").with(asUser(EDITOR, "Alice"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"spaceId\":" + space.getId() + ",\"parentId\":" + parent
                                + ",\"title\":\"" + title + "\",\"content\":\"본문\"}"))
                .andExpect(status().isCreated())
                .andReturn().getResponse().getContentAsString();
        return com.jayway.jsonpath.JsonPath.parse(body).read("$.id", Long.class);
    }

    @Test
    void 생성하면_버전1_리비전이_생기고_이벤트가_발행된다() throws Exception {
        long id = createPage(null, "루트");

        assertThat(revisions.findByPageIdAndVersion(id, 1)).isPresent();
        assertThat(events.events).anyMatch(e -> e.hasPageCreated());
    }

    @Test
    void VIEW만_있는_사용자는_생성_불가_403_조회는_가능() throws Exception {
        long id = createPage(null, "루트");

        mvc.perform(post("/api/wiki/pages").with(asUser(VIEWER, "Bob"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"spaceId\":" + space.getId() + ",\"parentId\":null,\"title\":\"t\",\"content\":\"c\"}"))
                .andExpect(status().isForbidden());
        mvc.perform(get("/api/wiki/pages/" + id).with(asUser(VIEWER, "Bob")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.content").value("본문"));
    }

    @Test
    void 트리_목록은_본문_없이_id_parent_title만() throws Exception {
        long root = createPage(null, "루트");
        createPage(root, "자식");

        mvc.perform(get("/api/wiki/spaces/" + space.getId() + "/pages").with(asUser(VIEWER, "Bob")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.length()").value(2))
                // (int) 캐스팅: jsonPath는 작은 수를 Integer로 역직렬화 — Long 비교는 실패한다(알려진 함정)
                .andExpect(jsonPath("$[1].parentId").value((int) root))
                .andExpect(jsonPath("$[0].content").doesNotExist());
    }

    @Test
    void 다른_스페이스의_부모는_400() throws Exception {
        Space other = spaces.save(Space.of("ops", "운영", null, EDITOR));
        perms.allow(EDITOR, other.getId(), WikiAction.EDIT);
        long root = createPage(null, "루트");

        mvc.perform(post("/api/wiki/pages").with(asUser(EDITOR, "Alice"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"spaceId\":" + other.getId() + ",\"parentId\":" + root + ",\"title\":\"t\",\"content\":\"c\"}"))
                .andExpect(status().isBadRequest());
    }

    @Test
    void 삭제하면_하위와_리비전이_함께_사라지고_이벤트가_발행된다() throws Exception {
        long root = createPage(null, "루트");
        long child = createPage(root, "자식");

        mvc.perform(delete("/api/wiki/pages/" + root).with(asUser(EDITOR, "Alice")))
                .andExpect(status().isNoContent());

        assertThat(pages.findById(child)).isEmpty();
        assertThat(events.events).anyMatch(e -> e.hasPageDeleted());
    }
}
```

- [ ] **Step 2: RED 확인**

```powershell
.\gradlew.bat test --tests "com.platform.wikibackend.page.*"
```
Expected: FAIL — 심볼 없음.

- [ ] **Step 3: 구현**

`page/dto/PageCreateRequest.java`:
```java
package com.platform.wikibackend.page.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

public record PageCreateRequest(
        @NotNull Long spaceId,
        Long parentId,
        @NotBlank @Size(max = 255) String title,
        @NotNull String content) {}
```

`page/dto/PageUpdateRequest.java` (Task 10의 409 로직까지 이 DTO 사용):
```java
package com.platform.wikibackend.page.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

public record PageUpdateRequest(
        @NotBlank @Size(max = 255) String title,
        @NotNull String content,
        Long parentId,                 // null이면 루트로 이동, 값이 있으면 그 아래로 이동
        @NotNull Integer expectedVersion) {}
```

`page/dto/PageResponse.java`:
```java
package com.platform.wikibackend.page.dto;

import com.platform.wikibackend.domain.Page;

public record PageResponse(Long id, Long spaceId, Long parentId, String title, String content, Integer version) {
    public static PageResponse from(Page p) {
        return new PageResponse(p.getId(), p.getSpaceId(), p.getParentId(), p.getTitle(), p.getContent(), p.getVersion());
    }
}
```

`page/dto/PageTreeItem.java`:
```java
package com.platform.wikibackend.page.dto;

import com.platform.wikibackend.domain.Page;

public record PageTreeItem(Long id, Long parentId, String title) {
    public static PageTreeItem from(Page p) {
        return new PageTreeItem(p.getId(), p.getParentId(), p.getTitle());
    }
}
```

`page/PageService.java`:
```java
package com.platform.wikibackend.page;

import com.platform.wikibackend.common.ConflictException;
import com.platform.wikibackend.common.NotFoundException;
import com.platform.wikibackend.domain.Page;
import com.platform.wikibackend.domain.PageRevision;
import com.platform.wikibackend.event.EventRelay;
import com.platform.wikibackend.event.WikiEvents;
import com.platform.wikibackend.page.dto.PageCreateRequest;
import com.platform.wikibackend.page.dto.PageResponse;
import com.platform.wikibackend.page.dto.PageTreeItem;
import com.platform.wikibackend.page.dto.PageUpdateRequest;
import com.platform.wikibackend.permission.WikiAction;
import com.platform.wikibackend.repository.PageRepository;
import com.platform.wikibackend.repository.PageRevisionRepository;
import com.platform.wikibackend.space.SpaceService;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Objects;

@Service
@RequiredArgsConstructor
@Transactional
public class PageService {

    private final PageRepository pages;
    private final PageRevisionRepository revisions;
    private final SpaceService spaces;
    private final EventRelay events;

    public PageResponse create(long userId, PageCreateRequest req) {
        spaces.require(userId, req.spaceId(), WikiAction.EDIT);
        validateParent(req.spaceId(), req.parentId(), null);
        Page saved = pages.save(Page.of(req.spaceId(), req.parentId(), req.title(), req.content(), userId));
        revisions.save(PageRevision.snapshotOf(saved)); // 버전1도 리비전에 — "모든 버전이 리비전에 있다"
        events.afterCommit(WikiEvents.pageCreated(userId, saved));
        return PageResponse.from(saved);
    }

    @Transactional(readOnly = true)
    public PageResponse get(long userId, long pageId) {
        Page p = getOwned(pageId);
        spaces.require(userId, p.getSpaceId(), WikiAction.VIEW);
        return PageResponse.from(p);
    }

    @Transactional(readOnly = true)
    public List<PageTreeItem> tree(long userId, long spaceId) {
        spaces.getForView(userId, spaceId);
        return pages.findBySpaceIdOrderById(spaceId).stream().map(PageTreeItem::from).toList();
    }

    /** 수정 = 새 버전. expectedVersion 불일치 409. parentId 변경은 이동(순환 검증). */
    public PageResponse update(long userId, long pageId, PageUpdateRequest req) {
        Page p = getOwned(pageId);
        spaces.require(userId, p.getSpaceId(), WikiAction.EDIT);
        if (!Objects.equals(p.getVersion(), req.expectedVersion())) {
            throw new ConflictException("버전 충돌 — 현재 " + p.getVersion() + ", 요청 " + req.expectedVersion());
        }
        if (!Objects.equals(p.getParentId(), req.parentId())) {
            validateParent(p.getSpaceId(), req.parentId(), pageId);
            p.moveTo(req.parentId());
        }
        p.edit(req.title(), req.content(), userId);
        revisions.save(PageRevision.snapshotOf(p));
        events.afterCommit(WikiEvents.pageUpdated(userId, p));
        return PageResponse.from(p);
    }

    public void delete(long userId, long pageId) {
        Page p = getOwned(pageId);
        spaces.require(userId, p.getSpaceId(), WikiAction.EDIT);
        long spaceId = p.getSpaceId();
        pages.delete(p); // cascade: 하위·리비전·첨부 (첨부 파일 정리는 Task 11)
        events.afterCommit(WikiEvents.pageDeleted(userId, pageId, spaceId));
    }

    /** 존재 검증만 — 권한은 호출부가. 첨부(Task 11)·리비전(Task 10)이 재사용. */
    @Transactional(readOnly = true)
    public Page getOwned(long pageId) {
        return pages.findById(pageId).orElseThrow(() -> new NotFoundException("페이지 없음: " + pageId));
    }

    /** parent는 같은 스페이스 + (이동 시) 자기 자신·자손 금지. */
    private void validateParent(Long spaceId, Long parentId, Long movingPageId) {
        if (parentId == null) return;
        Page parent = pages.findById(parentId).orElseThrow(() -> new IllegalArgumentException("부모 페이지 없음: " + parentId));
        if (!Objects.equals(parent.getSpaceId(), spaceId)) {
            throw new IllegalArgumentException("부모는 같은 스페이스여야 합니다");
        }
        if (movingPageId != null) {
            Long cursor = parentId;
            while (cursor != null) {
                if (Objects.equals(cursor, movingPageId)) throw new IllegalArgumentException("자기 자신/자손 아래로 이동 불가");
                cursor = pages.findById(cursor).map(Page::getParentId).orElse(null);
            }
        }
    }
}
```

`page/PageController.java`:
```java
package com.platform.wikibackend.page;

import com.platform.wikibackend.page.dto.PageCreateRequest;
import com.platform.wikibackend.page.dto.PageResponse;
import com.platform.wikibackend.page.dto.PageTreeItem;
import com.platform.wikibackend.page.dto.PageUpdateRequest;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

import java.util.List;

import static com.platform.wikibackend.space.SpaceController.userId;

@RestController
@RequiredArgsConstructor
public class PageController {

    private final PageService pages;

    @PostMapping("/api/wiki/pages")
    @ResponseStatus(HttpStatus.CREATED)
    public PageResponse create(@AuthenticationPrincipal Jwt jwt, @Valid @RequestBody PageCreateRequest req) {
        return pages.create(userId(jwt), req);
    }

    @GetMapping("/api/wiki/pages/{id}")
    public PageResponse get(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id) {
        return pages.get(userId(jwt), id);
    }

    @GetMapping("/api/wiki/spaces/{spaceId}/pages")
    public List<PageTreeItem> tree(@AuthenticationPrincipal Jwt jwt, @PathVariable Long spaceId) {
        return pages.tree(userId(jwt), spaceId);
    }

    @PutMapping("/api/wiki/pages/{id}")
    public PageResponse update(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id,
                               @Valid @RequestBody PageUpdateRequest req) {
        return pages.update(userId(jwt), id, req);
    }

    @DeleteMapping("/api/wiki/pages/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id) {
        pages.delete(userId(jwt), id);
    }
}
```

- [ ] **Step 4: GREEN + 전체 회귀**

```powershell
.\gradlew.bat test
```
Expected: 전체 PASS (page 5건 포함).

- [ ] **Step 5: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\wiki-backend add -A
git -C C:\MSA_TEMPLATE\wiki-backend commit -m "feat: 페이지 CRUD·트리 — same-space/순환 검증·cascade 삭제·이벤트"
```

---

### Task 10: 버전 충돌(409)·리비전 조회·복원

**Files:**
- Create: `src/main/java/com/platform/wikibackend/page/RevisionController.java`, `page/dto/RevisionResponse.java`, `page/dto/RevisionMeta.java`
- Modify: `src/main/java/com/platform/wikibackend/page/PageService.java` (restore 추가)
- Test: `src/test/java/com/platform/wikibackend/page/RevisionTest.java`

**Interfaces:**
- Consumes: Task 9의 `PageService.update`(이미 409 구현됨 — 이번 태스크는 검증 테스트), `PageRevisionRepository`
- Produces:
  - REST: `GET /api/wiki/pages/{id}/revisions`(메타 목록), `GET .../revisions/{version}`(본문), `POST .../revisions/{version}/restore`(복원=새 버전, 200)
  - `PageService.restore(userId, pageId, version)` → `PageResponse`
  - `RevisionMeta(Integer version, Long editedBy, java.time.Instant createdAt)`, `RevisionResponse(Integer version, String title, String content, Long editedBy)`

- [ ] **Step 1: 실패하는 테스트**

`page/RevisionTest.java`:
```java
package com.platform.wikibackend.page;

import com.platform.wikibackend.domain.Space;
import com.platform.wikibackend.permission.FakePermissionClient;
import com.platform.wikibackend.permission.WikiAction;
import com.platform.wikibackend.repository.PageRepository;
import com.platform.wikibackend.repository.SpaceRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import static com.platform.wikibackend.TestAuth.asUser;
import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@ActiveProfiles("test")
class RevisionTest {

    @Autowired WebApplicationContext context;
    @Autowired SpaceRepository spaces;
    @Autowired PageRepository pages;
    @Autowired FakePermissionClient perms;
    MockMvc mvc;

    long pageId;
    static final long EDITOR = 1L;

    @BeforeEach
    void setup() throws Exception {
        mvc = MockMvcBuilders.webAppContextSetup(context).apply(springSecurity()).build();
        pages.deleteAll();
        spaces.deleteAll();
        perms.reset();
        Space s = spaces.save(Space.of("dev", "개발", null, EDITOR));
        perms.allow(EDITOR, s.getId(), WikiAction.VIEW);
        perms.allow(EDITOR, s.getId(), WikiAction.EDIT);
        String body = mvc.perform(post("/api/wiki/pages").with(asUser(EDITOR, "Alice"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"spaceId\":" + s.getId() + ",\"parentId\":null,\"title\":\"t\",\"content\":\"v1\"}"))
                .andReturn().getResponse().getContentAsString();
        pageId = com.jayway.jsonpath.JsonPath.parse(body).read("$.id", Long.class);
    }

    private void editTo(String content, int expectedVersion) throws Exception {
        mvc.perform(put("/api/wiki/pages/" + pageId).with(asUser(EDITOR, "Alice"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"title\":\"t\",\"content\":\"" + content + "\",\"parentId\":null,\"expectedVersion\":" + expectedVersion + "}"))
                .andExpect(status().isOk());
    }

    @Test
    void expectedVersion_불일치는_409() throws Exception {
        editTo("v2", 1);
        mvc.perform(put("/api/wiki/pages/" + pageId).with(asUser(EDITOR, "Alice"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"title\":\"t\",\"content\":\"늦은 저장\",\"parentId\":null,\"expectedVersion\":1}"))
                .andExpect(status().isConflict());
    }

    @Test
    void 리비전_목록과_특정_버전_본문을_조회한다() throws Exception {
        editTo("v2", 1);

        mvc.perform(get("/api/wiki/pages/" + pageId + "/revisions").with(asUser(EDITOR, "Alice")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.length()").value(2))
                .andExpect(jsonPath("$[0].version").value(2))
                .andExpect(jsonPath("$[0].content").doesNotExist()); // 목록은 메타만

        mvc.perform(get("/api/wiki/pages/" + pageId + "/revisions/1").with(asUser(EDITOR, "Alice")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.content").value("v1"));
    }

    @Test
    void 복원은_롤백이_아니라_새_버전이다() throws Exception {
        editTo("v2", 1);

        mvc.perform(post("/api/wiki/pages/" + pageId + "/revisions/1/restore").with(asUser(EDITOR, "Alice")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.version").value(3))
                .andExpect(jsonPath("$.content").value("v1"));

        mvc.perform(get("/api/wiki/pages/" + pageId + "/revisions").with(asUser(EDITOR, "Alice")))
                .andExpect(jsonPath("$.length()").value(3)); // 이력 보존
    }

    @Test
    void 없는_버전_복원은_404() throws Exception {
        mvc.perform(post("/api/wiki/pages/" + pageId + "/revisions/99/restore").with(asUser(EDITOR, "Alice")))
                .andExpect(status().isNotFound());
    }
}
```

- [ ] **Step 2: RED 확인**

```powershell
.\gradlew.bat test --tests "com.platform.wikibackend.page.RevisionTest"
```
Expected: FAIL — RevisionController/restore 없음 (409 테스트는 Task 9 구현으로 이미 PASS일 수 있음 — 정상).

- [ ] **Step 3: 구현**

`page/dto/RevisionMeta.java`:
```java
package com.platform.wikibackend.page.dto;

import com.platform.wikibackend.domain.PageRevision;

import java.time.Instant;

public record RevisionMeta(Integer version, Long editedBy, Instant createdAt) {
    public static RevisionMeta from(PageRevision r) {
        return new RevisionMeta(r.getVersion(), r.getEditedBy(), r.getCreatedAt());
    }
}
```

`page/dto/RevisionResponse.java`:
```java
package com.platform.wikibackend.page.dto;

import com.platform.wikibackend.domain.PageRevision;

public record RevisionResponse(Integer version, String title, String content, Long editedBy) {
    public static RevisionResponse from(PageRevision r) {
        return new RevisionResponse(r.getVersion(), r.getTitle(), r.getContent(), r.getEditedBy());
    }
}
```

`PageService.java`에 추가:
```java
    @Transactional(readOnly = true)
    public List<RevisionMeta> listRevisions(long userId, long pageId) {
        Page p = getOwned(pageId);
        spaces.require(userId, p.getSpaceId(), WikiAction.VIEW);
        return revisions.findByPageIdOrderByVersionDesc(pageId).stream().map(RevisionMeta::from).toList();
    }

    @Transactional(readOnly = true)
    public RevisionResponse getRevision(long userId, long pageId, int version) {
        Page p = getOwned(pageId);
        spaces.require(userId, p.getSpaceId(), WikiAction.VIEW);
        return revisions.findByPageIdAndVersion(pageId, version)
                .map(RevisionResponse::from)
                .orElseThrow(() -> new NotFoundException("리비전 없음: v" + version));
    }

    /** 복원 = 해당 리비전 내용으로 새 버전 생성(이력 보존 — 스펙). */
    public PageResponse restore(long userId, long pageId, int version) {
        Page p = getOwned(pageId);
        spaces.require(userId, p.getSpaceId(), WikiAction.EDIT);
        PageRevision target = revisions.findByPageIdAndVersion(pageId, version)
                .orElseThrow(() -> new NotFoundException("리비전 없음: v" + version));
        p.edit(target.getTitle(), target.getContent(), userId);
        revisions.save(PageRevision.snapshotOf(p));
        events.afterCommit(WikiEvents.pageUpdated(userId, p));
        return PageResponse.from(p);
    }
```
(import 추가: `com.platform.wikibackend.page.dto.RevisionMeta`, `RevisionResponse`)

`page/RevisionController.java`:
```java
package com.platform.wikibackend.page;

import com.platform.wikibackend.page.dto.PageResponse;
import com.platform.wikibackend.page.dto.RevisionMeta;
import com.platform.wikibackend.page.dto.RevisionResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

import java.util.List;

import static com.platform.wikibackend.space.SpaceController.userId;

@RestController
@RequestMapping("/api/wiki/pages/{pageId}/revisions")
@RequiredArgsConstructor
public class RevisionController {

    private final PageService pages;

    @GetMapping
    public List<RevisionMeta> list(@AuthenticationPrincipal Jwt jwt, @PathVariable Long pageId) {
        return pages.listRevisions(userId(jwt), pageId);
    }

    @GetMapping("/{version}")
    public RevisionResponse get(@AuthenticationPrincipal Jwt jwt, @PathVariable Long pageId, @PathVariable Integer version) {
        return pages.getRevision(userId(jwt), pageId, version);
    }

    @PostMapping("/{version}/restore")
    public PageResponse restore(@AuthenticationPrincipal Jwt jwt, @PathVariable Long pageId, @PathVariable Integer version) {
        return pages.restore(userId(jwt), pageId, version);
    }
}
```

- [ ] **Step 4: GREEN + 전체 회귀**

```powershell
.\gradlew.bat test
```
Expected: 전체 PASS (revision 4건 포함).

- [ ] **Step 5: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\wiki-backend add -A
git -C C:\MSA_TEMPLATE\wiki-backend commit -m "feat: 버전 충돌 409·리비전 조회·복원(새 버전, 이력 보존)"
```

---

### Task 11: 첨부파일 — 업로드·다운로드·삭제 + 로컬 스토리지

**Files:**
- Create: `src/main/java/com/platform/wikibackend/attachment/` — `AttachmentController.java`, `AttachmentService.java`, `LocalFileStorage.java`, `dto/AttachmentResponse.java`
- Test: `src/test/java/com/platform/wikibackend/attachment/AttachmentTest.java`

**Interfaces:**
- Consumes: `PageService.getOwned`, `SpaceService.require`, `AttachmentRepository`, `EventRelay`, `WikiEvents.attachmentAdded`
- Produces:
  - REST: `POST /api/wiki/pages/{pageId}/attachments`(multipart `file`, 201), `GET /api/wiki/attachments/{id}`(다운로드, Content-Disposition attachment), `GET /api/wiki/pages/{pageId}/attachments`(목록), `DELETE /api/wiki/attachments/{id}`(204)
  - `LocalFileStorage`: `String store(InputStream in)`(→storageKey UUID), `org.springframework.core.io.Resource open(String key)`, `void delete(String key)`
  - `AttachmentResponse(Long id, String filename, String contentType, Long sizeBytes)`

- [ ] **Step 1: 실패하는 테스트**

`attachment/AttachmentTest.java`:
```java
package com.platform.wikibackend.attachment;

import com.platform.wikibackend.domain.Space;
import com.platform.wikibackend.event.RecordingEventPublisher;
import com.platform.wikibackend.permission.FakePermissionClient;
import com.platform.wikibackend.permission.WikiAction;
import com.platform.wikibackend.repository.AttachmentRepository;
import com.platform.wikibackend.repository.PageRepository;
import com.platform.wikibackend.repository.SpaceRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.mock.web.MockMultipartFile;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import static com.platform.wikibackend.TestAuth.asUser;
import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@ActiveProfiles("test")
class AttachmentTest {

    @Autowired WebApplicationContext context;
    @Autowired SpaceRepository spaces;
    @Autowired PageRepository pages;
    @Autowired AttachmentRepository attachments;
    @Autowired FakePermissionClient perms;
    @Autowired RecordingEventPublisher events;
    MockMvc mvc;

    long pageId;
    static final long EDITOR = 1L;
    static final long VIEWER = 2L;

    @BeforeEach
    void setup() throws Exception {
        mvc = MockMvcBuilders.webAppContextSetup(context).apply(springSecurity()).build();
        attachments.deleteAll();
        pages.deleteAll();
        spaces.deleteAll();
        perms.reset();
        events.reset();
        Space s = spaces.save(Space.of("dev", "개발", null, EDITOR));
        perms.allow(EDITOR, s.getId(), WikiAction.VIEW);
        perms.allow(EDITOR, s.getId(), WikiAction.EDIT);
        perms.allow(VIEWER, s.getId(), WikiAction.VIEW);
        String body = mvc.perform(post("/api/wiki/pages").with(asUser(EDITOR, "Alice"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"spaceId\":" + s.getId() + ",\"parentId\":null,\"title\":\"t\",\"content\":\"c\"}"))
                .andReturn().getResponse().getContentAsString();
        pageId = com.jayway.jsonpath.JsonPath.parse(body).read("$.id", Long.class);
    }

    private MockMultipartFile png() {
        return new MockMultipartFile("file", "스크린샷.png", "image/png", new byte[]{1, 2, 3, 4});
    }

    @Test
    void EDIT_사용자만_업로드할_수_있고_이벤트가_발행된다() throws Exception {
        mvc.perform(multipart("/api/wiki/pages/" + pageId + "/attachments").file(png())
                        .with(asUser(VIEWER, "Bob")))
                .andExpect(status().isForbidden());

        mvc.perform(multipart("/api/wiki/pages/" + pageId + "/attachments").file(png())
                        .with(asUser(EDITOR, "Alice")))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.filename").value("스크린샷.png"));

        assertThat(events.events).anyMatch(e -> e.hasAttachmentAdded());
    }

    @Test
    void 다운로드는_원본_바이트와_attachment_헤더를_준다() throws Exception {
        String body = mvc.perform(multipart("/api/wiki/pages/" + pageId + "/attachments").file(png())
                        .with(asUser(EDITOR, "Alice")))
                .andReturn().getResponse().getContentAsString();
        long id = com.jayway.jsonpath.JsonPath.parse(body).read("$.id", Long.class);

        mvc.perform(get("/api/wiki/attachments/" + id).with(asUser(VIEWER, "Bob")))
                .andExpect(status().isOk())
                .andExpect(header().string("Content-Disposition", org.hamcrest.Matchers.containsString("attachment")))
                .andExpect(content().bytes(new byte[]{1, 2, 3, 4}));
    }

    @Test
    void 삭제하면_메타와_파일이_사라진다() throws Exception {
        String body = mvc.perform(multipart("/api/wiki/pages/" + pageId + "/attachments").file(png())
                        .with(asUser(EDITOR, "Alice")))
                .andReturn().getResponse().getContentAsString();
        long id = com.jayway.jsonpath.JsonPath.parse(body).read("$.id", Long.class);

        mvc.perform(delete("/api/wiki/attachments/" + id).with(asUser(EDITOR, "Alice")))
                .andExpect(status().isNoContent());
        mvc.perform(get("/api/wiki/attachments/" + id).with(asUser(EDITOR, "Alice")))
                .andExpect(status().isNotFound());
        assertThat(attachments.findByPageId(pageId)).isEmpty();
    }
}
```

- [ ] **Step 2: RED 확인**

```powershell
.\gradlew.bat test --tests "com.platform.wikibackend.attachment.*"
```
Expected: FAIL — 심볼 없음.

- [ ] **Step 3: 구현**

`attachment/LocalFileStorage.java`:
```java
package com.platform.wikibackend.attachment;

import jakarta.annotation.PostConstruct;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.PathResource;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.io.InputStream;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import java.util.UUID;

/** 로컬 디스크 첨부 저장 — 파일명은 UUID(storage_key), 원본명은 DB에만(경로 조작 차단). */
@Component
public class LocalFileStorage {

    private final Path dir;

    public LocalFileStorage(@Value("${platform.wiki.files-dir}") String filesDir) {
        this.dir = Path.of(filesDir);
    }

    @PostConstruct
    void init() throws IOException {
        Files.createDirectories(dir);
    }

    public String store(InputStream in) {
        String key = UUID.randomUUID().toString();
        try {
            Files.copy(in, dir.resolve(key), StandardCopyOption.REPLACE_EXISTING);
        } catch (IOException e) {
            throw new UncheckedIOException("첨부 저장 실패", e);
        }
        return key;
    }

    public Resource open(String key) {
        return new PathResource(dir.resolve(key));
    }

    /** 실패는 무해(고아 파일) — 호출부가 WARN 로그. */
    public boolean delete(String key) {
        try {
            return Files.deleteIfExists(dir.resolve(key));
        } catch (IOException e) {
            return false;
        }
    }
}
```

`attachment/dto/AttachmentResponse.java`:
```java
package com.platform.wikibackend.attachment.dto;

import com.platform.wikibackend.domain.Attachment;

public record AttachmentResponse(Long id, String filename, String contentType, Long sizeBytes) {
    public static AttachmentResponse from(Attachment a) {
        return new AttachmentResponse(a.getId(), a.getFilename(), a.getContentType(), a.getSizeBytes());
    }
}
```

`attachment/AttachmentService.java`:
```java
package com.platform.wikibackend.attachment;

import com.platform.wikibackend.attachment.dto.AttachmentResponse;
import com.platform.wikibackend.common.NotFoundException;
import com.platform.wikibackend.domain.Attachment;
import com.platform.wikibackend.domain.Page;
import com.platform.wikibackend.event.EventRelay;
import com.platform.wikibackend.event.WikiEvents;
import com.platform.wikibackend.page.PageService;
import com.platform.wikibackend.permission.WikiAction;
import com.platform.wikibackend.repository.AttachmentRepository;
import com.platform.wikibackend.space.SpaceService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.util.List;

@Service
@RequiredArgsConstructor
@Transactional
@Slf4j
public class AttachmentService {

    private final AttachmentRepository attachments;
    private final PageService pages;
    private final SpaceService spaces;
    private final LocalFileStorage storage;
    private final EventRelay events;

    public AttachmentResponse upload(long userId, long pageId, MultipartFile file) {
        Page page = pages.getOwned(pageId);
        spaces.require(userId, page.getSpaceId(), WikiAction.EDIT);
        String key;
        try {
            key = storage.store(file.getInputStream());
        } catch (IOException e) {
            throw new UncheckedIOException("업로드 스트림 읽기 실패", e);
        }
        String filename = file.getOriginalFilename() != null ? file.getOriginalFilename() : "unnamed";
        String contentType = file.getContentType() != null ? file.getContentType() : "application/octet-stream";
        Attachment saved = attachments.save(
                Attachment.of(pageId, filename, contentType, file.getSize(), key, userId));
        events.afterCommit(WikiEvents.attachmentAdded(userId, saved, page.getSpaceId()));
        return AttachmentResponse.from(saved);
    }

    @Transactional(readOnly = true)
    public List<AttachmentResponse> list(long userId, long pageId) {
        Page page = pages.getOwned(pageId);
        spaces.require(userId, page.getSpaceId(), WikiAction.VIEW);
        return attachments.findByPageId(pageId).stream().map(AttachmentResponse::from).toList();
    }

    /** 다운로드용 — [메타, 리소스] 쌍. */
    @Transactional(readOnly = true)
    public DownloadItem download(long userId, long attachmentId) {
        Attachment a = attachments.findById(attachmentId)
                .orElseThrow(() -> new NotFoundException("첨부 없음: " + attachmentId));
        Page page = pages.getOwned(a.getPageId());
        spaces.require(userId, page.getSpaceId(), WikiAction.VIEW);
        return new DownloadItem(a, storage.open(a.getStorageKey()));
    }

    public void delete(long userId, long attachmentId) {
        Attachment a = attachments.findById(attachmentId)
                .orElseThrow(() -> new NotFoundException("첨부 없음: " + attachmentId));
        Page page = pages.getOwned(a.getPageId());
        spaces.require(userId, page.getSpaceId(), WikiAction.EDIT);
        attachments.delete(a);
        if (!storage.delete(a.getStorageKey())) {
            log.warn("첨부 파일 삭제 실패(고아 파일 — 무해): key={}", a.getStorageKey());
        }
    }

    public record DownloadItem(Attachment meta, Resource resource) {}
}
```

`attachment/AttachmentController.java`:
```java
package com.platform.wikibackend.attachment;

import com.platform.wikibackend.attachment.dto.AttachmentResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.core.io.Resource;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.util.List;

import static com.platform.wikibackend.space.SpaceController.userId;

@RestController
@RequiredArgsConstructor
public class AttachmentController {

    private final AttachmentService service;

    @PostMapping("/api/wiki/pages/{pageId}/attachments")
    @ResponseStatus(HttpStatus.CREATED)
    public AttachmentResponse upload(@AuthenticationPrincipal Jwt jwt, @PathVariable Long pageId,
                                     @RequestParam("file") MultipartFile file) {
        return service.upload(userId(jwt), pageId, file);
    }

    @GetMapping("/api/wiki/pages/{pageId}/attachments")
    public List<AttachmentResponse> list(@AuthenticationPrincipal Jwt jwt, @PathVariable Long pageId) {
        return service.list(userId(jwt), pageId);
    }

    /** Content-Disposition attachment 고정 — 브라우저 인라인 실행(XSS) 차단(스펙). */
    @GetMapping("/api/wiki/attachments/{id}")
    public ResponseEntity<Resource> download(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id) {
        AttachmentService.DownloadItem item = service.download(userId(jwt), id);
        String encoded = URLEncoder.encode(item.meta().getFilename(), StandardCharsets.UTF_8).replace("+", "%20");
        return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename*=UTF-8''" + encoded)
                .contentType(MediaType.parseMediaType(item.meta().getContentType()))
                .body(item.resource());
    }

    @DeleteMapping("/api/wiki/attachments/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id) {
        service.delete(userId(jwt), id);
    }
}
```

- [ ] **Step 4: GREEN + 전체 회귀 (정확 집계)**

```powershell
.\gradlew.bat test --rerun-tasks
```
Expected: 전체 PASS — **contextLoads 1 + repository 2 + permission 4 + event 3 + space 5 + page 5 + revision 4 + attachment 3 = 27 tests**.

참고: 업로드 max size 413은 MockMvc에서 검증 불가(서블릿 컨테이너 레벨 제한) — yml 설정(`spring.servlet.multipart.max-file-size`) 존재로 커버하고 실검증은 Task 14 수동 체크리스트에 포함하지 않는다(스펙의 413은 설정 계약).

- [ ] **Step 5: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\wiki-backend add -A
git -C C:\MSA_TEMPLATE\wiki-backend commit -m "feat: 첨부 업로드/다운로드/삭제 — UUID 스토리지·Content-Disposition attachment"
```

---

### Task 12: Dockerfile + compose 통합 + 게이트웨이 라우트 + defer 2건

**Files:**
- Create: `wiki-backend/Dockerfile`
- Modify: `C:\MSA_TEMPLATE\infra\keycloak\init-authdb.sql` (wikidb 추가)
- Modify: `C:\MSA_TEMPLATE\infra\keycloak\docker-compose.yml` (wiki-backend 서비스 + redis appendonly + iss/aud 명시 + gateway env + 볼륨)
- Modify: `C:\MSA_TEMPLATE\gateway-server\src\main\resources\application.yml` (wiki 라우트)
- Modify: `C:\MSA_TEMPLATE\README.md` (부트스트랩 경고 1줄 — defer ②)
- Modify: `C:\MSA_TEMPLATE\.gitignore` (Task 4에서 수정분 커밋)

**Interfaces:**
- Consumes: Task 11까지 완성된 wiki-backend (bootJar)
- Produces: compose 컨테이너 `wiki-backend`, 게이트웨이 `/api/wiki/**` 라우트, redis AOF, auth·org·wiki 공통 iss/aud env

- [ ] **Step 1: Dockerfile**

`wiki-backend/Dockerfile`:
```dockerfile
# 런타임 전용. jar는 CI(또는 로컬)의 `gradlew bootJar` 산출물을 복사한다.
FROM eclipse-temurin:24-jre
WORKDIR /app
COPY build/libs/app.jar app.jar
EXPOSE 9110
ENTRYPOINT ["java", "-jar", "app.jar"]
```

- [ ] **Step 2: init-authdb.sql + 기존 볼륨 수동 생성**

`init-authdb.sql` 끝에 추가:
```sql
-- wiki-backend용. Flyway가 스키마 생성.
CREATE DATABASE wikidb;
```
기존 볼륨:
```powershell
docker exec platform-postgres psql -U keycloak -c "CREATE DATABASE wikidb;"
```
Expected: `CREATE DATABASE`.

- [ ] **Step 3: compose 수정 (Read 후 정확 삽입 — 기존 블록 무변경 원칙)**

① `redis:` 서비스에 AOF(이벤트 내구성):
```yaml
    command: ["redis-server", "--appendonly", "yes"]
```
② `org-service:` 블록 뒤에 추가:
```yaml
  # wiki-backend — 첫 제품 서비스 (Wave B). REST :9110(게이트웨이 경유), 이벤트는 redis stream 발행.
  wiki-backend:
    image: ghcr.io/chanho4702/wiki-backend:${WIKI_TAG:-latest}
    build: ../../wiki-backend
    container_name: platform-wiki
    environment:
      SPRING_PROFILES_ACTIVE: docker
      WIKI_DB_URL: jdbc:postgresql://postgres:5432/wikidb
      AUTH_JWKS_URI: http://auth-server:9000/.well-known/jwks.json
      PLATFORM_ISSUER: http://localhost:9000
      PLATFORM_AUDIENCE: platform-api
      REDIS_HOST: redis
      ORG_GRPC_HOST: org-service
      WIKI_FILES_DIR: /data/attachments
    volumes:
      - platform-wiki-files:/data/attachments
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
    healthcheck:
      test: ["CMD-SHELL", "bash -c 'exec 3<>/dev/tcp/127.0.0.1/9110'"]
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 40s
```
③ **defer ① 이행** — `auth-server:`와 `org-service:`의 environment에도 명시 추가(기본값 우연 일치 의존 제거):
```yaml
      PLATFORM_ISSUER: http://localhost:9000
      PLATFORM_AUDIENCE: platform-api
```
(auth-server는 발급자라 issuer 소스가 코드 기본값과 동일한지 먼저 Read로 확인 — auth-server가 다른 env 이름을 쓰면 해당 이름으로. 불명확하면 org/wiki 2곳만 명시하고 보고서에 기록)
④ `gateway:` environment에 추가:
```yaml
      WIKI_SERVICE_URI: http://wiki-backend:9110
```
⑤ `volumes:`에 추가:
```yaml
  platform-wiki-files:
```

- [ ] **Step 4: 게이트웨이 라우트**

`gateway-server/src/main/resources/application.yml`의 routes 마지막(org 블록 뒤)에 추가:
```yaml
            # wiki-backend — 위키 REST (Wave B). dev: lb:// · docker: WIKI_SERVICE_URI DNS 직결
            - id: wiki
              uri: ${WIKI_SERVICE_URI:lb://wiki-backend}
              predicates:
                - Path=/api/wiki/**
```

- [ ] **Step 5: README 경고 (defer ②)**

`C:\MSA_TEMPLATE\README.md`의 빠른 시작 섹션 끝에 추가:
```markdown
> ⚠️ `PLATFORM_BOOTSTRAP_ADMIN_ID`는 기본값 1(최초 가입자)이며 재기동마다 GLOBAL ADMIN으로 복구된다. **운영 배포 전 반드시 `.env`에 실제 관리자 id를 명시하거나 빈 값으로 비활성화할 것.**
```

- [ ] **Step 6: 빌드·기동 스모크**

```powershell
$env:JAVA_HOME = 'C:\Program Files\Java\jdk-24'; $env:GITHUB_TOKEN = (gh auth token)
cd C:\MSA_TEMPLATE\wiki-backend; .\gradlew.bat bootJar
cd C:\MSA_TEMPLATE\infra\keycloak
docker compose build wiki-backend
docker compose up -d
docker compose ps
docker logs platform-wiki --tail 20
```
Expected: 11컨테이너(기존 10 + wiki), `platform-wiki` healthy, 로그에 Flyway V1 적용 + Tomcat 9110 기동. 게이트웨이 라우트 스모크:
```powershell
curl.exe -s -o NUL -w "%{http_code}" http://localhost:8000/api/wiki/spaces
```
Expected: `401` (라우트 활성 + 보안 동작. 404면 gateway 재빌드 필요: `cd C:\MSA_TEMPLATE\gateway-server; .\gradlew.bat bootJar; cd C:\MSA_TEMPLATE\infra\keycloak; docker compose build gateway; docker compose up -d gateway`)

- [ ] **Step 7: 커밋 (3 repo)**

```powershell
git -C C:\MSA_TEMPLATE\wiki-backend add -A
git -C C:\MSA_TEMPLATE\wiki-backend commit -m "feat: Dockerfile(런타임 전용)"
git -C C:\MSA_TEMPLATE\gateway-server add src/main/resources/application.yml
git -C C:\MSA_TEMPLATE\gateway-server commit -m "feat: /api/wiki/** 라우트 — WIKI_SERVICE_URI 오버라이드"
git -C C:\MSA_TEMPLATE add .gitignore README.md infra/keycloak/init-authdb.sql infra/keycloak/docker-compose.yml
git -C C:\MSA_TEMPLATE commit -m "feat(infra): wiki-backend compose 통합 + wikidb + redis AOF + iss/aud 명시 + 부트스트랩 경고"
```

---

### Task 13: GitHub repo + CI

**Files:**
- Create: `wiki-backend/.github/workflows/ci.yml`

**Interfaces:**
- Produces: `github.com/chanho4702/wiki-backend` (private), CI — PR/main push `gradlew build` (GitHub Packages proto 소비 포함)

- [ ] **Step 1: workflow**

`.github/workflows/ci.yml`:
```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: read   # chanho4702/platform-backend의 common-proto 읽기 (패키지 Actions access에 wiki-backend 등록 필요)
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '24'
      - uses: gradle/actions/setup-gradle@v4
      - run: ./gradlew build
        env:
          GITHUB_ACTOR: ${{ github.actor }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- [ ] **Step 2: repo 생성 + push + CI 확인**

선행: Task 3 Step 3의 패키지 Actions access(wiki-backend, Read)가 완료돼 있어야 한다 — 미완료면 컨트롤러에 보고 후 대기.
```powershell
cd C:\MSA_TEMPLATE\wiki-backend
git add -A; git commit -m "ci: 빌드·테스트 (GitHub Packages proto 소비)"
gh repo create chanho4702/wiki-backend --private --source . --push
gh run list --repo chanho4702/wiki-backend --limit 2
gh run watch <run id> --repo chanho4702/wiki-backend --exit-status --interval 15
```
Expected: build success. 실패가 `Could not resolve com.platform:common-proto`면 패키지 Actions access 미설정 — 보고 후 대기(우회 금지).

---

### Task 14: E2E 스모크(자동화분) + README 2건

**Files:**
- Create: `wiki-backend/README.md`
- Modify: `C:\MSA_TEMPLATE\README.md` (토폴로지·컴포넌트 표)

**Interfaces:**
- Consumes: 전체 완성 상태 (스펙 성공 기준 1~6)

- [ ] **Step 1: 이벤트 스트림 실측 준비 확인 (성공 기준 5 — 토큰 불요 부분)**

```powershell
docker exec platform-redis redis-cli XLEN platform:events:v1
```
Expected: `0` 또는 에러 없이 정수(스트림 미생성 시 0). 페이지 저장 후 증가 확인은 **관리자 AT가 필요한 수동 체크리스트** — 컨트롤러가 사용자에게 전달:
1. 브라우저 로그인 → AT 복사 → `POST /api/wiki/spaces`(201) → `POST /api/wiki/pages`(201)
2. `docker exec platform-redis redis-cli XLEN platform:events:v1` → **2 이상**
3. `docker exec platform-redis redis-cli XRANGE platform:events:v1 - + COUNT 3` → payload 바이너리 확인
4. 일반 계정 AT로 남의 스페이스 `PUT` → 403 (권한 실연동 — 성공 기준 3)
5. dev 모드: IntelliJ `wiki-backend bootRun` → Eureka 등록 → 게이트웨이 경유 200 (성공 기준 6)

- [ ] **Step 2: wiki-backend README 작성**

````markdown
# wiki-backend

ALM·Wiki 플랫폼의 위키 코어 서비스. [chanho4702/platform-backend](https://github.com/chanho4702/platform-backend)의
`com.platform:common-proto`(GitHub Packages)를 소비한다 — 빌드 전 `GITHUB_TOKEN`(read:packages) 필요.

## 실행

```powershell
$env:JAVA_HOME = 'C:\Program Files\Java\jdk-24'
$env:GITHUB_TOKEN = (gh auth token)
.\gradlew.bat bootRun    # dev 모드 (:9110, Eureka 등록)
```

배포판은 infra/keycloak compose — docker 프로필(Eureka 미등록, DNS 직결).

## 도메인

- **space / page(계층 트리) / page_revision(전체 스냅샷) / attachment(UUID 로컬 스토리지)**
- 저장 = 새 버전(`expectedVersion` 낙관 잠금, 충돌 409) · 복원 = 새 버전(이력 보존)
- 권한: org-service gRPC `CheckPermission`(SPACE VIEW<EDIT<ADMIN) + Caffeine 30s, org 불능 시 fail-closed 403
- 이벤트: `EventPublisher` → Redis Streams `platform:events:v1` (커밋 후, best-effort — 색인 정합은 Wave C 재색인)

## API

`/api/wiki/spaces` · `/api/wiki/pages` · `/api/wiki/pages/{id}/revisions` · `/api/wiki/pages/{id}/attachments` — 전부 게이트웨이(:8000) 경유, JWT 필수.
````

- [ ] **Step 3: 우산 README 갱신**

- 토폴로지 gateway 분기에 `└──▶ lb://wiki-backend(:9110)` 추가, infra 줄에 wikidb 언급
- 컴포넌트 표에 행 추가:
```markdown
| **wiki-backend** | 9110 | 위키 — 스페이스·페이지 트리·버전·첨부. 권한 gRPC + Redis Streams 이벤트 | [chanho4702/wiki-backend](https://github.com/chanho4702/wiki-backend) |
```

- [ ] **Step 4: 커밋 + push**

```powershell
git -C C:\MSA_TEMPLATE\wiki-backend add README.md
git -C C:\MSA_TEMPLATE\wiki-backend commit -m "docs: wiki-backend README"
git -C C:\MSA_TEMPLATE\wiki-backend push
git -C C:\MSA_TEMPLATE add README.md
git -C C:\MSA_TEMPLATE commit -m "docs: wiki-backend 반영 — 토폴로지·컴포넌트 표"
```
(우산 push는 컨트롤러 판단 — 병행 세션 유무 확인 후)

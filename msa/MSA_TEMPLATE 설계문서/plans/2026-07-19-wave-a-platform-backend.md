# Wave A: platform-backend 골격 + common-proto + org-service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** ALM·Wiki 플랫폼의 토대 — platform-backend 멀티모듈 repo, common-proto(gRPC 계약, GitHub Packages 발행), org-service(팀·RBAC, REST+gRPC)를 만들어 compose 스택에 통합한다.

**Architecture:** platform-backend는 Gradle 멀티모듈 독립 repo(common-proto + org-service). org-service는 board-service와 동일한 JWT 리소스 서버 패턴 + Flyway/Postgres에, 순정 grpc-java `Server`(SmartLifecycle 빈)로 PermissionService gRPC(:9131)를 노출한다. 게이트웨이에는 `/api/org/**` 라우트 1건 추가(env 오버라이드로 dev=Eureka lb:// / docker=DNS 직결 분기).

**Tech Stack:** Spring Boot 4.0.6 / Java 24 / Spring Cloud 2025.1.2 / protobuf-java 4.29.3 + grpc-java 1.69.0 (protobuf gradle plugin 0.9.4) / Flyway + PostgreSQL / H2(test) / Lombok.

**스펙:** `MSA_TEMPLATE 설계문서/specs/2026-07-19-wave-a-platform-backend-design.md`

## Global Constraints

- 경로: repo 루트는 `C:\MSA_TEMPLATE\platform-backend` (우산 repo에서 gitignore되는 독립 git repo). 이하 상대경로는 이 루트 기준.
- 버전 고정: Boot `4.0.6`, dependency-management `1.1.7`, Spring Cloud `2025.1.2`, Java 툴체인 `24`, Lombok `1.18.46`, protobuf-java `4.29.3`, grpc `1.69.0`, protobuf 플러그인 `0.9.4`. group `com.platform`.
- 포트: org-service HTTP **9130** / gRPC **9131**. DB는 기존 Postgres 인스턴스의 **`orgdb`** (기존 authdb·boarddb 관례).
- proto 패키지 `platform.org.v1` / java_package `com.platform.proto.org.v1`. proto enum 값에 `Role.ROLE_ADMIN` — 같은 proto 패키지의 `Action.ADMIN`과 C++ 스코프 충돌 회피용이므로 "오타"로 보고 고치지 말 것.
- JWT 검증: `jwk-set-uri`만 사용, `issuer-uri` 금지(OIDC discovery 미노출로 기동 실패 — board-service 스펙 실측).
- 커밋 메시지는 한국어 관례(`feat:`, `chore:`, `test:` 프리픽스), 각 태스크 말미 커밋. 빌드 명령은 PowerShell 기준 `.\gradlew.bat`.
- 모든 테스트는 `@ActiveProfiles("test")` — H2(MODE=PostgreSQL) + Flyway off + ddl-auto create-drop + eureka off (board-service 패턴).

---

### Task 1: platform-backend repo 골격

**Files:**
- Create: `settings.gradle`, `build.gradle`, `.gitignore`
- Copy: `gradlew`, `gradlew.bat`, `gradle/wrapper/*` (board-service에서)
- Modify: `C:\MSA_TEMPLATE\.gitignore` (우산 repo)

**Interfaces:**
- Produces: 모듈 2개(`common-proto`, `org-service`)가 등록된 빌드 골격. 이후 모든 태스크가 이 repo 안에서 작업.

- [ ] **Step 1: 디렉터리 + git init**

```powershell
New-Item -ItemType Directory -Force C:\MSA_TEMPLATE\platform-backend
git -C C:\MSA_TEMPLATE\platform-backend init -b main
```

- [ ] **Step 2: gradle wrapper 복사 (board-service에서)**

```powershell
Copy-Item C:\MSA_TEMPLATE\board-service\gradlew,C:\MSA_TEMPLATE\board-service\gradlew.bat C:\MSA_TEMPLATE\platform-backend\
Copy-Item -Recurse C:\MSA_TEMPLATE\board-service\gradle C:\MSA_TEMPLATE\platform-backend\gradle
```

- [ ] **Step 3: settings.gradle / build.gradle / .gitignore 작성**

`settings.gradle`:
```groovy
rootProject.name = 'platform-backend'
// search-service · notification-service 모듈은 Wave C/E에서 include 추가
include 'common-proto', 'org-service'
```

`build.gradle` (루트 — 공통 최소만, 나머지는 모듈별):
```groovy
allprojects {
    group = 'com.platform'
}
```

`.gitignore`:
```
.gradle/
build/
out/
.idea/
!.run/
*.iml
```

- [ ] **Step 4: 우산 repo gitignore에 등록**

`C:\MSA_TEMPLATE\.gitignore`의 기존 서비스 repo 나열부(gateway-server/ 등)와 같은 블록에 한 줄 추가:
```
platform-backend/
```

- [ ] **Step 5: 빈 모듈 디렉터리 생성 후 빌드 확인**

```powershell
New-Item -ItemType Directory -Force C:\MSA_TEMPLATE\platform-backend\common-proto, C:\MSA_TEMPLATE\platform-backend\org-service
cd C:\MSA_TEMPLATE\platform-backend; .\gradlew.bat projects
```
Expected: `Root project 'platform-backend'` 아래 `:common-proto`, `:org-service` 출력, BUILD SUCCESSFUL.

- [ ] **Step 6: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\platform-backend add -A
git -C C:\MSA_TEMPLATE\platform-backend commit -m "chore: platform-backend 멀티모듈 골격 (common-proto + org-service)"
```
(우산 repo의 .gitignore 변경은 Task 10에서 게이트웨이·compose 변경과 함께 커밋)

---

### Task 2: common-proto 모듈 — 계약 정의 + 코드젠 + 발행 설정

**Files:**
- Create: `common-proto/build.gradle`
- Create: `common-proto/src/main/proto/platform/org/v1/org.proto`

**Interfaces:**
- Produces (이후 태스크가 사용하는 생성 클래스, java_package `com.platform.proto.org.v1`):
  - `PermissionServiceGrpc.PermissionServiceImplBase` (Task 9가 상속)
  - `CheckPermissionRequest/Response`, `ListUserGrantsRequest/Response`, `Grant`
  - enum `ResourceType { GLOBAL, SPACE, PROJECT }`, `Action { VIEW, EDIT, ADMIN }`, `Role { VIEWER, EDITOR, ROLE_ADMIN }`

- [ ] **Step 1: build.gradle 작성**

`common-proto/build.gradle`:
```groovy
plugins {
    id 'java-library'
    id 'com.google.protobuf' version '0.9.4'
    id 'maven-publish'
}
// CI가 태그 발행 시 -PprotoVersion=x.y.z 로 주입. 로컬 기본은 SNAPSHOT.
version = providers.gradleProperty('protoVersion').getOrElse('0.1.0-SNAPSHOT')
java { toolchain { languageVersion = JavaLanguageVersion.of(24) } }
repositories { mavenCentral() }

def grpcVersion = '1.69.0'
def protobufVersion = '4.29.3'

dependencies {
    api "io.grpc:grpc-stub:${grpcVersion}"
    api "io.grpc:grpc-protobuf:${grpcVersion}"
    api "com.google.protobuf:protobuf-java:${protobufVersion}"
    // 생성 코드의 @jakarta.annotation.Generated
    compileOnly 'jakarta.annotation:jakarta.annotation-api:3.0.0'
}

protobuf {
    protoc { artifact = "com.google.protobuf:protoc:${protobufVersion}" }
    plugins {
        grpc { artifact = "io.grpc:protoc-gen-grpc-java:${grpcVersion}" }
    }
    generateProtoTasks {
        all()*.plugins { grpc {} }
    }
}

publishing {
    publications {
        mavenJava(MavenPublication) { from components.java }
    }
    repositories {
        maven {
            name = 'GitHubPackages'
            url = uri('https://maven.pkg.github.com/chanho4702/platform-backend')
            credentials {
                username = System.getenv('GITHUB_ACTOR')
                password = System.getenv('GITHUB_TOKEN')
            }
        }
    }
}
```

- [ ] **Step 2: org.proto 작성 (스펙 계약 전문)**

`common-proto/src/main/proto/platform/org/v1/org.proto`:
```protobuf
syntax = "proto3";
package platform.org.v1;
option java_multiple_files = true;
option java_package = "com.platform.proto.org.v1";

// 조직/권한 계약 v1 — breaking change는 v2 패키지 신설로 대응(온프렘 구버전 공존).
service PermissionService {
  // 단건 권한 판정 — 모든 서비스의 인가 2차 방어가 호출
  rpc CheckPermission(CheckPermissionRequest) returns (CheckPermissionResponse);
  // 검색 결과 권한 필터용(Wave C) — 사용자가 접근 가능한 리소스 목록
  rpc ListUserGrants(ListUserGrantsRequest) returns (ListUserGrantsResponse);
}

enum ResourceType { RESOURCE_TYPE_UNSPECIFIED = 0; GLOBAL = 1; SPACE = 2; PROJECT = 3; }
enum Action { ACTION_UNSPECIFIED = 0; VIEW = 1; EDIT = 2; ADMIN = 3; }
// ROLE_ADMIN: 같은 proto 패키지의 Action.ADMIN과 C++ 스코프 충돌 회피(proto3 enum 값은 패키지 스코프)
enum Role { ROLE_UNSPECIFIED = 0; VIEWER = 1; EDITOR = 2; ROLE_ADMIN = 3; }

message CheckPermissionRequest {
  int64 user_id = 1;           // JWT sub
  ResourceType resource_type = 2;
  string resource_id = 3;      // GLOBAL이면 빈 값
  Action action = 4;
}
message CheckPermissionResponse {
  bool allowed = 1;
  Role effective_role = 2;     // grant 없으면 ROLE_UNSPECIFIED
}

message ListUserGrantsRequest {
  int64 user_id = 1;
  ResourceType resource_type = 2;  // UNSPECIFIED면 전체
}
message ListUserGrantsResponse { repeated Grant grants = 1; }
message Grant {
  ResourceType resource_type = 1;
  string resource_id = 2;
  Role role = 3;
}
```

- [ ] **Step 3: 코드젠 검증**

```powershell
cd C:\MSA_TEMPLATE\platform-backend; .\gradlew.bat :common-proto:build
Get-ChildItem -Recurse common-proto\build\generated\source\proto\main | Select-Object -ExpandProperty Name
```
Expected: BUILD SUCCESSFUL + `PermissionServiceGrpc.java`, `CheckPermissionRequest.java` 등 생성 확인.

- [ ] **Step 4: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\platform-backend add -A
git -C C:\MSA_TEMPLATE\platform-backend commit -m "feat: common-proto — platform.org.v1 PermissionService 계약 + GitHub Packages 발행 설정"
```

---

### Task 3: org-service 부트스트랩 (Boot 앱 + 프로필 구성)

**Files:**
- Create: `org-service/build.gradle`
- Create: `org-service/src/main/java/com/platform/orgservice/OrgServiceApplication.java`
- Create: `org-service/src/main/resources/application.yml`
- Test: `org-service/src/test/java/com/platform/orgservice/OrgServiceApplicationTests.java`
- Test: `org-service/src/test/resources/application-test.yml`

**Interfaces:**
- Consumes: `project(':common-proto')`
- Produces: 패키지 루트 `com.platform.orgservice`, 프로필 계약(default=dev/Eureka on, `docker`=Eureka off), 설정 키 `platform.grpc.port|enabled`, `platform.bootstrap-admin-id`, `platform.jwt.issuer|audience`

- [ ] **Step 1: build.gradle 작성 (board-service 계열 + gRPC)**

`org-service/build.gradle`:
```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.0.6'
    id 'io.spring.dependency-management' version '1.1.7'
}
version = '0.0.1-SNAPSHOT'
java { toolchain { languageVersion = JavaLanguageVersion.of(24) } }
repositories { mavenCentral() }

ext { set('springCloudVersion', '2025.1.2') }
dependencyManagement {
    imports { mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}" }
}

def grpcVersion = '1.69.0'

dependencies {
    implementation project(':common-proto')
    // 유레카 자기등록 — dev 모드에서 게이트웨이 lb://org-service가 이 등록을 바라본다 (docker 프로필은 off)
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-flyway'
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-database-postgresql'
    implementation "io.grpc:grpc-netty-shaded:${grpcVersion}"
    implementation "io.grpc:grpc-services:${grpcVersion}"   // 리플렉션(grpcurl 디버깅용)
    runtimeOnly 'org.postgresql:postgresql'
    compileOnly 'org.projectlombok:lombok:1.18.46'
    annotationProcessor 'org.projectlombok:lombok:1.18.46'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.boot:spring-boot-data-jpa-test'
    testImplementation 'org.springframework.security:spring-security-test'
    testImplementation "io.grpc:grpc-inprocess:${grpcVersion}"
    testRuntimeOnly 'com.h2database:h2'
}
tasks.named('test') { useJUnitPlatform() }
tasks.named('bootJar') { archiveFileName = 'app.jar' }
```

- [ ] **Step 2: Application 클래스**

`org-service/src/main/java/com/platform/orgservice/OrgServiceApplication.java`:
```java
package com.platform.orgservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OrgServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrgServiceApplication.class, args);
    }
}
```

- [ ] **Step 3: application.yml (프로필 분기 포함)**

`org-service/src/main/resources/application.yml`:
```yaml
server:
  port: 9130
spring:
  application:
    name: org-service
  # 모든 자격증명/URL은 env로 오버라이드 가능. 기본값은 로컬 dev 전용.
  datasource:
    url: ${ORG_DB_URL:jdbc:postgresql://localhost:5433/orgdb}
    username: ${ORG_DB_USERNAME:keycloak}
    password: ${ORG_DB_PASSWORD:keycloak}
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
  flyway:
    enabled: true
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
    # DNS 해석 불가 호스트명 등록 방지 — 게이트웨이 lb:// 해석 사고 예방 (기존 서비스 실측 패턴)
    prefer-ip-address: true

platform:
  jwt:
    issuer: ${PLATFORM_ISSUER:http://localhost:9000}
    audience: ${PLATFORM_AUDIENCE:platform-api}
  grpc:
    port: ${ORG_GRPC_PORT:9131}
    enabled: true
  # 최초 GLOBAL ADMIN 부트스트랩 — auth-server user id. 빈 값이면 시드 생략.
  bootstrap-admin-id: ${PLATFORM_BOOTSTRAP_ADMIN_ID:}

---
# 배포판(compose): Eureka 미등록 — 게이트웨이는 ORG_SERVICE_URI(DNS 직결)로 라우팅 (15번 문서 확정)
spring:
  config:
    activate:
      on-profile: docker
eureka:
  client:
    enabled: false
```

- [ ] **Step 4: 테스트 프로필 + contextLoads 테스트 작성**

`org-service/src/test/resources/application-test.yml`:
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:orgdb;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
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
  grpc:
    enabled: false   # 단위 테스트에서 실포트 열지 않음 — gRPC는 in-process로 검증
  bootstrap-admin-id: ""
```

`org-service/src/test/java/com/platform/orgservice/OrgServiceApplicationTests.java`:
```java
package com.platform.orgservice;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest
@ActiveProfiles("test")
class OrgServiceApplicationTests {
    @Test
    void contextLoads() {
    }
}
```

- [ ] **Step 5: 테스트 실행**

```powershell
cd C:\MSA_TEMPLATE\platform-backend; .\gradlew.bat :org-service:test
```
Expected: PASS (contextLoads).

- [ ] **Step 6: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\platform-backend add -A
git -C C:\MSA_TEMPLATE\platform-backend commit -m "feat: org-service 부트스트랩 — Boot 4.0.6 + 프로필 분기(dev=Eureka/docker=DNS)"
```

---

### Task 4: DB 스키마(Flyway) + 엔티티 + 리포지토리

**Files:**
- Create: `org-service/src/main/resources/db/migration/V1__init.sql`
- Create: `org-service/src/main/java/com/platform/orgservice/domain/` 아래 `SubjectType.java`, `ResourceKind.java`, `GrantRole.java`, `PermAction.java`, `MemberStatus.java`, `TeamRole.java`, `Member.java`, `Team.java`, `TeamMember.java`, `GrantEntry.java`
- Create: `org-service/src/main/java/com/platform/orgservice/repository/` 아래 `MemberRepository.java`, `TeamRepository.java`, `TeamMemberRepository.java`, `GrantEntryRepository.java`
- Test: `org-service/src/test/java/com/platform/orgservice/repository/GrantEntryRepositoryTest.java`

**Interfaces:**
- Produces:
  - enum `GrantRole { VIEWER, EDITOR, ADMIN }` — `int rank()`, `boolean covers(PermAction)`
  - enum `PermAction { VIEW, EDIT, ADMIN }`, `ResourceKind { GLOBAL, SPACE, PROJECT }`, `SubjectType { USER, TEAM }`
  - `GrantEntryRepository.findEffective(userId, teamIds, kind, resourceId)` / `findAllForUser(userId, teamIds)` / `findAllForUserByKind(userId, teamIds, kind)` / `findBySubjectTypeAndSubjectIdAndResourceTypeAndResourceId(...)`
  - `TeamMemberRepository.findTeamIdsByMemberId(memberId)`
  - `GrantEntry.globalAdmin(userId)` 정적 팩토리, `Member.of(id, name, email)`, `Member.refresh(name, email)`

- [ ] **Step 1: V1__init.sql 작성**

`org-service/src/main/resources/db/migration/V1__init.sql`:
```sql
-- 조직/팀/권한 스키마. 싱글 테넌트 전제(15번 문서 확정) — organization 테이블 없음.
CREATE TABLE member (
    id           BIGINT PRIMARY KEY,                 -- auth-server user id(JWT sub) 그대로
    display_name VARCHAR(255) NOT NULL,
    email        VARCHAR(255),
    status       VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE',
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE TABLE team (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_member (
    id         BIGSERIAL PRIMARY KEY,
    team_id    BIGINT NOT NULL REFERENCES team (id) ON DELETE CASCADE,
    member_id  BIGINT NOT NULL REFERENCES member (id) ON DELETE CASCADE,
    role       VARCHAR(20) NOT NULL,                 -- LEAD | MEMBER
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, member_id)
);
CREATE INDEX idx_team_member_member ON team_member (member_id);

-- 권한 부여 단일 원장. "grant"는 SQL 예약어라 grant_entry.
-- subject_id는 USER/TEAM 다형이라 FK 없음(의도).
CREATE TABLE grant_entry (
    id            BIGSERIAL PRIMARY KEY,
    subject_type  VARCHAR(10) NOT NULL,              -- USER | TEAM
    subject_id    BIGINT      NOT NULL,
    resource_type VARCHAR(20) NOT NULL,              -- GLOBAL | SPACE | PROJECT
    resource_id   VARCHAR(100) NOT NULL DEFAULT '',  -- GLOBAL이면 ''
    role          VARCHAR(20) NOT NULL,              -- VIEWER | EDITOR | ADMIN
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (subject_type, subject_id, resource_type, resource_id)
);
CREATE INDEX idx_grant_resource ON grant_entry (resource_type, resource_id);
CREATE INDEX idx_grant_subject ON grant_entry (subject_type, subject_id);
```

- [ ] **Step 2: enum 6종 작성**

`domain/SubjectType.java`:
```java
package com.platform.orgservice.domain;

public enum SubjectType { USER, TEAM }
```

`domain/ResourceKind.java` (proto의 ResourceType과 이름 충돌 방지용 명명):
```java
package com.platform.orgservice.domain;

public enum ResourceKind { GLOBAL, SPACE, PROJECT }
```

`domain/PermAction.java`:
```java
package com.platform.orgservice.domain;

public enum PermAction {
    VIEW(1), EDIT(2), ADMIN(3);

    private final int requiredRank;

    PermAction(int requiredRank) { this.requiredRank = requiredRank; }

    public int requiredRank() { return requiredRank; }
}
```

`domain/GrantRole.java`:
```java
package com.platform.orgservice.domain;

public enum GrantRole {
    VIEWER(1), EDITOR(2), ADMIN(3);

    private final int rank;

    GrantRole(int rank) { this.rank = rank; }

    public int rank() { return rank; }

    /** ADMIN ⊃ EDITOR ⊃ VIEWER 계층 — 이 role로 action이 허용되는가. */
    public boolean covers(PermAction action) { return rank >= action.requiredRank(); }
}
```

`domain/MemberStatus.java`:
```java
package com.platform.orgservice.domain;

public enum MemberStatus { ACTIVE, DEACTIVATED }
```

`domain/TeamRole.java`:
```java
package com.platform.orgservice.domain;

public enum TeamRole { LEAD, MEMBER }
```

- [ ] **Step 3: 엔티티 4종 작성**

`domain/Member.java`:
```java
package com.platform.orgservice.domain;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.Instant;

@Entity
@Table(name = "member")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Member {

    @Id
    private Long id; // auth-server user id(JWT sub) — 자체 시퀀스 없음

    @Column(nullable = false)
    private String displayName;

    private String email;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private MemberStatus status;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @UpdateTimestamp
    @Column(nullable = false)
    private Instant updatedAt;

    public static Member of(Long id, String displayName, String email) {
        Member m = new Member();
        m.id = id;
        m.displayName = displayName != null ? displayName : "user-" + id;
        m.email = email;
        m.status = MemberStatus.ACTIVE;
        return m;
    }

    /** JIT 미러링 — 클레임이 바뀐 경우만 갱신. */
    public void refresh(String displayName, String email) {
        if (displayName != null) this.displayName = displayName;
        if (email != null) this.email = email;
    }
}
```

`domain/Team.java`:
```java
package com.platform.orgservice.domain;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.Instant;

@Entity
@Table(name = "team")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Team {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;

    private String description;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @UpdateTimestamp
    @Column(nullable = false)
    private Instant updatedAt;

    public static Team of(String name, String description) {
        Team t = new Team();
        t.name = name;
        t.description = description;
        return t;
    }

    public void update(String name, String description) {
        this.name = name;
        this.description = description;
    }
}
```

`domain/TeamMember.java` (연관관계 대신 id 컬럼 직참조 — 조인 최소 도메인):
```java
package com.platform.orgservice.domain;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;

import java.time.Instant;

@Entity
@Table(name = "team_member",
        uniqueConstraints = @UniqueConstraint(columnNames = {"teamId", "memberId"}))
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class TeamMember {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "team_id", nullable = false)
    private Long teamId;

    @Column(name = "member_id", nullable = false)
    private Long memberId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private TeamRole role;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    public static TeamMember of(Long teamId, Long memberId, TeamRole role) {
        TeamMember tm = new TeamMember();
        tm.teamId = teamId;
        tm.memberId = memberId;
        tm.role = role;
        return tm;
    }
}
```

`domain/GrantEntry.java`:
```java
package com.platform.orgservice.domain;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;

import java.time.Instant;

@Entity
@Table(name = "grant_entry",
        uniqueConstraints = @UniqueConstraint(columnNames = {"subjectType", "subjectId", "resourceType", "resourceId"}))
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class GrantEntry {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING)
    @Column(name = "subject_type", nullable = false)
    private SubjectType subjectType;

    @Column(name = "subject_id", nullable = false)
    private Long subjectId;

    @Enumerated(EnumType.STRING)
    @Column(name = "resource_type", nullable = false)
    private ResourceKind resourceType;

    @Column(name = "resource_id", nullable = false)
    private String resourceId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private GrantRole role;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    public static GrantEntry of(SubjectType subjectType, Long subjectId,
                                ResourceKind resourceType, String resourceId, GrantRole role) {
        GrantEntry g = new GrantEntry();
        g.subjectType = subjectType;
        g.subjectId = subjectId;
        g.resourceType = resourceType;
        g.resourceId = resourceId != null ? resourceId : "";
        g.role = role;
        return g;
    }

    public static GrantEntry globalAdmin(Long userId) {
        return of(SubjectType.USER, userId, ResourceKind.GLOBAL, "", GrantRole.ADMIN);
    }

    /** 부트스트랩 시드가 기존 grant를 승격할 때 사용. */
    public void changeRole(GrantRole role) { this.role = role; }
}
```

- [ ] **Step 4: 리포지토리 4종 작성**

`repository/MemberRepository.java`:
```java
package com.platform.orgservice.repository;

import com.platform.orgservice.domain.Member;
import org.springframework.data.jpa.repository.JpaRepository;

public interface MemberRepository extends JpaRepository<Member, Long> {
}
```

`repository/TeamRepository.java`:
```java
package com.platform.orgservice.repository;

import com.platform.orgservice.domain.Team;
import org.springframework.data.jpa.repository.JpaRepository;

public interface TeamRepository extends JpaRepository<Team, Long> {
    boolean existsByName(String name);
}
```

`repository/TeamMemberRepository.java`:
```java
package com.platform.orgservice.repository;

import com.platform.orgservice.domain.TeamMember;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;
import java.util.Optional;

public interface TeamMemberRepository extends JpaRepository<TeamMember, Long> {

    @Query("select tm.teamId from TeamMember tm where tm.memberId = :memberId")
    List<Long> findTeamIdsByMemberId(@Param("memberId") Long memberId);

    Optional<TeamMember> findByTeamIdAndMemberId(Long teamId, Long memberId);

    List<TeamMember> findByTeamId(Long teamId);
}
```

`repository/GrantEntryRepository.java`:
```java
package com.platform.orgservice.repository;

import com.platform.orgservice.domain.GrantEntry;
import com.platform.orgservice.domain.ResourceKind;
import com.platform.orgservice.domain.SubjectType;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;
import java.util.Optional;

public interface GrantEntryRepository extends JpaRepository<GrantEntry, Long> {

    /**
     * 권한 판정용 — 사용자 직접 grant + 소속 팀 grant 중
     * GLOBAL 또는 (해당 리소스타입 + 리소스 id)에 걸리는 행 전부.
     * teamIds는 비어 있으면 호출부에서 List.of(-1L)로 채운다(빈 IN 절 회피).
     */
    @Query("""
            select g from GrantEntry g
            where ((g.subjectType = com.platform.orgservice.domain.SubjectType.USER and g.subjectId = :userId)
                or (g.subjectType = com.platform.orgservice.domain.SubjectType.TEAM and g.subjectId in :teamIds))
              and (g.resourceType = com.platform.orgservice.domain.ResourceKind.GLOBAL
                or (g.resourceType = :kind and g.resourceId = :resourceId))
            """)
    List<GrantEntry> findEffective(@Param("userId") Long userId,
                                   @Param("teamIds") List<Long> teamIds,
                                   @Param("kind") ResourceKind kind,
                                   @Param("resourceId") String resourceId);

    /** 내 grant 전체 — /me/permissions·ListUserGrants용. (":x is null or" 패턴은 H2 enum 바인딩 리스크가 있어 메서드 분리) */
    @Query("""
            select g from GrantEntry g
            where ((g.subjectType = com.platform.orgservice.domain.SubjectType.USER and g.subjectId = :userId)
                or (g.subjectType = com.platform.orgservice.domain.SubjectType.TEAM and g.subjectId in :teamIds))
            """)
    List<GrantEntry> findAllForUser(@Param("userId") Long userId,
                                    @Param("teamIds") List<Long> teamIds);

    /** 내 grant 중 특정 리소스타입만. */
    @Query("""
            select g from GrantEntry g
            where ((g.subjectType = com.platform.orgservice.domain.SubjectType.USER and g.subjectId = :userId)
                or (g.subjectType = com.platform.orgservice.domain.SubjectType.TEAM and g.subjectId in :teamIds))
              and g.resourceType = :kind
            """)
    List<GrantEntry> findAllForUserByKind(@Param("userId") Long userId,
                                          @Param("teamIds") List<Long> teamIds,
                                          @Param("kind") ResourceKind kind);

    List<GrantEntry> findByResourceTypeAndResourceId(ResourceKind resourceType, String resourceId);

    Optional<GrantEntry> findBySubjectTypeAndSubjectIdAndResourceTypeAndResourceId(
            SubjectType subjectType, Long subjectId, ResourceKind resourceType, String resourceId);
}
```

- [ ] **Step 5: 리포지토리 테스트 작성 (실패 확인 → 통과)**

`org-service/src/test/java/com/platform/orgservice/repository/GrantEntryRepositoryTest.java`:
```java
package com.platform.orgservice.repository;

import com.platform.orgservice.domain.*;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.test.context.ActiveProfiles;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@ActiveProfiles("test")
class GrantEntryRepositoryTest {

    @Autowired GrantEntryRepository grants;
    @Autowired TeamMemberRepository teamMembers;
    @Autowired TeamRepository teams;
    @Autowired MemberRepository members;

    @Test
    void findEffective_직접grant와_팀grant와_GLOBAL을_모두_수집한다() {
        members.save(Member.of(1L, "Alice", "a@x.com"));
        Team dev = teams.save(Team.of("dev", null));
        teamMembers.save(TeamMember.of(dev.getId(), 1L, TeamRole.MEMBER));

        grants.save(GrantEntry.of(SubjectType.USER, 1L, ResourceKind.SPACE, "sp-1", GrantRole.VIEWER)); // 직접
        grants.save(GrantEntry.of(SubjectType.TEAM, dev.getId(), ResourceKind.SPACE, "sp-1", GrantRole.EDITOR)); // 팀 경유
        grants.save(GrantEntry.of(SubjectType.USER, 1L, ResourceKind.GLOBAL, "", GrantRole.VIEWER)); // GLOBAL
        grants.save(GrantEntry.of(SubjectType.USER, 1L, ResourceKind.SPACE, "sp-2", GrantRole.ADMIN)); // 다른 리소스 — 제외 대상

        List<GrantEntry> found = grants.findEffective(1L, List.of(dev.getId()), ResourceKind.SPACE, "sp-1");

        assertThat(found).hasSize(3)
                .extracting(GrantEntry::getRole)
                .containsExactlyInAnyOrder(GrantRole.VIEWER, GrantRole.EDITOR, GrantRole.VIEWER);
    }

    @Test
    void findAllForUser_전체와_리소스타입_필터를_각각_반환한다() {
        grants.save(GrantEntry.of(SubjectType.USER, 2L, ResourceKind.GLOBAL, "", GrantRole.ADMIN));
        grants.save(GrantEntry.of(SubjectType.USER, 2L, ResourceKind.PROJECT, "pj-1", GrantRole.EDITOR));

        assertThat(grants.findAllForUser(2L, List.of(-1L))).hasSize(2);
        assertThat(grants.findAllForUserByKind(2L, List.of(-1L), ResourceKind.PROJECT)).hasSize(1);
    }
}
```

- [ ] **Step 6: 테스트 실행**

```powershell
cd C:\MSA_TEMPLATE\platform-backend; .\gradlew.bat :org-service:test --tests "com.platform.orgservice.repository.*"
```
Expected: PASS 2건.

- [ ] **Step 7: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\platform-backend add -A
git -C C:\MSA_TEMPLATE\platform-backend commit -m "feat: org-service 스키마·엔티티·리포지토리 — member/team/team_member/grant_entry"
```

---

### Task 5: 권한 판정 도메인 (PermissionFacade)

**Files:**
- Create: `org-service/src/main/java/com/platform/orgservice/permission/PermissionFacade.java`
- Test: `org-service/src/test/java/com/platform/orgservice/permission/PermissionFacadeTest.java`

**Interfaces:**
- Consumes: Task 4의 리포지토리·enum
- Produces:
  - `PermissionFacade.Decision(boolean allowed, GrantRole effectiveRole /*nullable*/)`
  - `Decision check(long userId, ResourceKind kind, String resourceId, PermAction action)`
  - `List<GrantEntry> grantsOf(long userId, ResourceKind kindOrNull)`
  - `void requireGlobalAdmin(long userId)` — 아니면 `AccessDeniedException`

- [ ] **Step 1: 실패하는 테스트 작성**

`org-service/src/test/java/com/platform/orgservice/permission/PermissionFacadeTest.java`:
```java
package com.platform.orgservice.permission;

import com.platform.orgservice.domain.*;
import com.platform.orgservice.repository.GrantEntryRepository;
import com.platform.orgservice.repository.MemberRepository;
import com.platform.orgservice.repository.TeamMemberRepository;
import com.platform.orgservice.repository.TeamRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.transaction.annotation.Transactional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@SpringBootTest
@ActiveProfiles("test")
@Transactional
class PermissionFacadeTest {

    @Autowired PermissionFacade facade;
    @Autowired GrantEntryRepository grants;
    @Autowired TeamRepository teams;
    @Autowired TeamMemberRepository teamMembers;
    @Autowired MemberRepository members;

    @Test
    void 직접grant와_팀grant_중_최고role로_판정한다() {
        members.save(Member.of(1L, "Alice", null));
        Team dev = teams.save(Team.of("dev", null));
        teamMembers.save(TeamMember.of(dev.getId(), 1L, TeamRole.MEMBER));
        grants.save(GrantEntry.of(SubjectType.USER, 1L, ResourceKind.SPACE, "sp-1", GrantRole.VIEWER));
        grants.save(GrantEntry.of(SubjectType.TEAM, dev.getId(), ResourceKind.SPACE, "sp-1", GrantRole.EDITOR));

        PermissionFacade.Decision d = facade.check(1L, ResourceKind.SPACE, "sp-1", PermAction.EDIT);

        assertThat(d.allowed()).isTrue();
        assertThat(d.effectiveRole()).isEqualTo(GrantRole.EDITOR);
    }

    @Test
    void 상위action은_하위role로_거부된다() {
        grants.save(GrantEntry.of(SubjectType.USER, 2L, ResourceKind.SPACE, "sp-1", GrantRole.VIEWER));

        assertThat(facade.check(2L, ResourceKind.SPACE, "sp-1", PermAction.VIEW).allowed()).isTrue();
        assertThat(facade.check(2L, ResourceKind.SPACE, "sp-1", PermAction.EDIT).allowed()).isFalse();
    }

    @Test
    void GLOBAL_grant는_모든_리소스에_적용된다() {
        grants.save(GrantEntry.of(SubjectType.USER, 3L, ResourceKind.GLOBAL, "", GrantRole.ADMIN));

        assertThat(facade.check(3L, ResourceKind.SPACE, "any", PermAction.ADMIN).allowed()).isTrue();
        assertThat(facade.check(3L, ResourceKind.PROJECT, "any", PermAction.EDIT).allowed()).isTrue();
    }

    @Test
    void grant가_없으면_거부되고_effectiveRole은_null이다() {
        PermissionFacade.Decision d = facade.check(99L, ResourceKind.SPACE, "sp-1", PermAction.VIEW);

        assertThat(d.allowed()).isFalse();
        assertThat(d.effectiveRole()).isNull();
    }

    @Test
    void requireGlobalAdmin은_비관리자에게_AccessDeniedException을_던진다() {
        grants.save(GrantEntry.of(SubjectType.USER, 4L, ResourceKind.GLOBAL, "", GrantRole.ADMIN));

        facade.requireGlobalAdmin(4L); // 통과
        assertThatThrownBy(() -> facade.requireGlobalAdmin(5L))
                .isInstanceOf(AccessDeniedException.class);
    }
}
```

- [ ] **Step 2: 실행해서 컴파일 실패 확인**

```powershell
cd C:\MSA_TEMPLATE\platform-backend; .\gradlew.bat :org-service:test --tests "com.platform.orgservice.permission.*"
```
Expected: FAIL — `PermissionFacade` 심볼 없음.

- [ ] **Step 3: PermissionFacade 구현**

`org-service/src/main/java/com/platform/orgservice/permission/PermissionFacade.java`:
```java
package com.platform.orgservice.permission;

import com.platform.orgservice.domain.GrantEntry;
import com.platform.orgservice.domain.GrantRole;
import com.platform.orgservice.domain.PermAction;
import com.platform.orgservice.domain.ResourceKind;
import com.platform.orgservice.repository.GrantEntryRepository;
import com.platform.orgservice.repository.TeamMemberRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Comparator;
import java.util.List;

/** 권한 판정 단일 진입점 — REST 가드·gRPC PermissionService가 공용. */
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class PermissionFacade {

    private final GrantEntryRepository grants;
    private final TeamMemberRepository teamMembers;

    /** effectiveRole은 grant가 하나도 없으면 null. */
    public record Decision(boolean allowed, GrantRole effectiveRole) {}

    public Decision check(long userId, ResourceKind kind, String resourceId, PermAction action) {
        GrantRole effective = grants.findEffective(userId, teamIdsOf(userId), kind, nullToEmpty(resourceId))
                .stream()
                .map(GrantEntry::getRole)
                .max(Comparator.comparingInt(GrantRole::rank))
                .orElse(null);
        return new Decision(effective != null && effective.covers(action), effective);
    }

    /** 내 grant 목록 — kind가 null이면 전체. /me/permissions·ListUserGrants 공용. */
    public List<GrantEntry> grantsOf(long userId, ResourceKind kind) {
        List<Long> teamIds = teamIdsOf(userId);
        return kind == null
                ? grants.findAllForUser(userId, teamIds)
                : grants.findAllForUserByKind(userId, teamIds, kind);
    }

    public void requireGlobalAdmin(long userId) {
        if (!check(userId, ResourceKind.GLOBAL, "", PermAction.ADMIN).allowed()) {
            throw new AccessDeniedException("GLOBAL ADMIN 권한이 필요합니다");
        }
    }

    private List<Long> teamIdsOf(long userId) {
        List<Long> ids = teamMembers.findTeamIdsByMemberId(userId);
        return ids.isEmpty() ? List.of(-1L) : ids; // 빈 IN 절 회피
    }

    private static String nullToEmpty(String s) { return s == null ? "" : s; }
}
```

- [ ] **Step 4: 테스트 통과 확인**

```powershell
.\gradlew.bat :org-service:test --tests "com.platform.orgservice.permission.*"
```
Expected: PASS 5건.

- [ ] **Step 5: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\platform-backend add -A
git -C C:\MSA_TEMPLATE\platform-backend commit -m "feat: PermissionFacade — 직접/팀/GLOBAL grant 병합 최고 role 판정"
```

---

### Task 6: 보안 구성 + JIT 멤버 미러링 + /api/org/me/permissions

**Files:**
- Create: `org-service/src/main/java/com/platform/orgservice/config/SecurityConfig.java`
- Create: `org-service/src/main/java/com/platform/orgservice/security/AudienceValidator.java`
- Create: `org-service/src/main/java/com/platform/orgservice/security/MemberMirrorFilter.java`
- Create: `org-service/src/main/java/com/platform/orgservice/member/MemberService.java`
- Create: `org-service/src/main/java/com/platform/orgservice/me/MeController.java`
- Create: `org-service/src/main/java/com/platform/orgservice/me/dto/GrantResponse.java`
- Test: `org-service/src/test/java/com/platform/orgservice/me/MeControllerTest.java`
- Test: `org-service/src/test/java/com/platform/orgservice/TestAuth.java`

**Interfaces:**
- Consumes: `PermissionFacade.grantsOf`, `MemberRepository`
- Produces:
  - `MemberService.mirror(long id, String name, String email)` — upsert
  - `GrantResponse(String resourceType, String resourceId, String role)` — grant 직렬화 공용 DTO (`GrantResponse.from(GrantEntry)`)
  - `TestAuth.asUser(long id, String name)` / `asAdmin(...)` — 이후 컨트롤러 테스트 공용
  - 보안 계약: `/api/org/**` 전부 인증 필요, JWT sub → userId(long)

- [ ] **Step 1: 실패하는 테스트 작성**

`org-service/src/test/java/com/platform/orgservice/TestAuth.java` (board-service 패턴 이식):
```java
package com.platform.orgservice;

import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.test.web.servlet.request.RequestPostProcessor;

import java.util.List;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;

public class TestAuth {

    /** sub=userId. jwt() post-processor는 컨버터를 거치지 않으므로 authorities 직접 지정. */
    public static RequestPostProcessor asUser(long id, String name) {
        return jwt().jwt(j -> j.subject(String.valueOf(id)).claim("name", name)
                        .claim("email", name.toLowerCase() + "@test.com").claim("roles", List.of("USER")))
                .authorities(new SimpleGrantedAuthority("ROLE_USER"));
    }

    public static RequestPostProcessor asAdmin(long id, String name) {
        return jwt().jwt(j -> j.subject(String.valueOf(id)).claim("name", name)
                        .claim("email", name.toLowerCase() + "@test.com").claim("roles", List.of("ADMIN")))
                .authorities(new SimpleGrantedAuthority("ROLE_ADMIN"));
    }
}
```

`org-service/src/test/java/com/platform/orgservice/me/MeControllerTest.java`:
```java
package com.platform.orgservice.me;

import com.platform.orgservice.domain.GrantEntry;
import com.platform.orgservice.domain.GrantRole;
import com.platform.orgservice.domain.ResourceKind;
import com.platform.orgservice.domain.SubjectType;
import com.platform.orgservice.repository.GrantEntryRepository;
import com.platform.orgservice.repository.MemberRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import static com.platform.orgservice.TestAuth.asUser;
import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@ActiveProfiles("test")
class MeControllerTest {

    @Autowired WebApplicationContext context;
    @Autowired GrantEntryRepository grants;
    @Autowired MemberRepository members;
    MockMvc mvc;

    @BeforeEach
    void setup() {
        mvc = MockMvcBuilders.webAppContextSetup(context).apply(springSecurity()).build();
        grants.deleteAll();
        members.deleteAll();
    }

    @Test
    void 무토큰은_401() throws Exception {
        mvc.perform(get("/api/org/me/permissions")).andExpect(status().isUnauthorized());
    }

    @Test
    void 내_grant_목록을_반환한다() throws Exception {
        grants.save(GrantEntry.of(SubjectType.USER, 1L, ResourceKind.SPACE, "sp-1", GrantRole.EDITOR));

        mvc.perform(get("/api/org/me/permissions").with(asUser(1L, "Alice")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$[0].resourceType").value("SPACE"))
                .andExpect(jsonPath("$[0].resourceId").value("sp-1"))
                .andExpect(jsonPath("$[0].role").value("EDITOR"));
    }

    @Test
    void 인증_요청이_지나가면_member가_JIT_미러링된다() throws Exception {
        mvc.perform(get("/api/org/me/permissions").with(asUser(7L, "Bob"))).andExpect(status().isOk());

        assertThat(members.findById(7L)).isPresent()
                .get().satisfies(m -> assertThat(m.getDisplayName()).isEqualTo("Bob"));
    }
}
```

- [ ] **Step 2: 실행해서 실패 확인**

```powershell
.\gradlew.bat :org-service:test --tests "com.platform.orgservice.me.*"
```
Expected: FAIL — MeController/SecurityConfig 심볼 없음.

- [ ] **Step 3: 구현 (5개 파일)**

`security/AudienceValidator.java` (board-service 동일 패턴):
```java
package com.platform.orgservice.security;

import org.springframework.security.oauth2.core.OAuth2Error;
import org.springframework.security.oauth2.core.OAuth2TokenValidator;
import org.springframework.security.oauth2.core.OAuth2TokenValidatorResult;
import org.springframework.security.oauth2.jwt.Jwt;

/** aud 클레임에 기대 audience가 포함되는지 검증 — 다른 대상 발급 토큰 거부. */
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

`config/SecurityConfig.java`:
```java
package com.platform.orgservice.config;

import com.platform.orgservice.security.AudienceValidator;
import org.springframework.beans.factory.annotation.Value;
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

    /** JWKS 서명 + issuer/audience 검증 (board-service 동일 패턴, issuer-uri 금지). */
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

    /**
     * MemberMirrorFilter를 Security 체인에 명시 편입(AuthorizationFilter 뒤 = 인증 완료 후).
     * MockMvc(springSecurity())에서도 같은 체인이 재현되고, 서블릿 컨테이너 자동 등록으로
     * 이중 실행되더라도 OncePerRequestFilter가 요청당 1회를 보장한다.
     */
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http, JwtAuthenticationConverter converter,
                                    MemberMirrorFilter memberMirrorFilter) throws Exception {
        http
                .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .csrf(csrf -> csrf.disable())
                // 관리 도메인 — 공개 엔드포인트 없음. 세부 인가(GLOBAL ADMIN)는 PermissionFacade가 판정.
                .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(converter)))
                .addFilterAfter(memberMirrorFilter, AuthorizationFilter.class);
        return http.build();
    }
}
```
(추가 import: `com.platform.orgservice.security.MemberMirrorFilter`, `org.springframework.security.web.access.intercept.AuthorizationFilter`)

`member/MemberService.java`:
```java
package com.platform.orgservice.member;

import com.platform.orgservice.domain.Member;
import com.platform.orgservice.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class MemberService {

    private final MemberRepository members;

    /** JIT 미러링 upsert. 최초 동시 요청 2건의 PK 충돌은 무시(다음 요청이 refresh). */
    @Transactional
    public void mirror(long id, String displayName, String email) {
        try {
            members.findById(id).ifPresentOrElse(
                    m -> m.refresh(displayName, email),
                    () -> members.save(Member.of(id, displayName, email)));
        } catch (DataIntegrityViolationException ignored) {
        }
    }
}
```

`security/MemberMirrorFilter.java`:
```java
package com.platform.orgservice.security;

import com.platform.orgservice.member.MemberService;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

/**
 * 인증된 요청의 JWT 클레임(sub/name/email)으로 member를 JIT 미러링.
 * SecurityConfig가 이 필터를 Security 체인(AuthorizationFilter 뒤)에 편입한다 —
 * SecurityContext가 채워진 뒤에 동작하며, MockMvc 테스트에서도 동일 체인이 재현된다.
 */
@Component
@RequiredArgsConstructor
public class MemberMirrorFilter extends OncePerRequestFilter {

    private final MemberService members;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            try {
                long id = Long.parseLong(jwtAuth.getToken().getSubject());
                members.mirror(id,
                        jwtAuth.getToken().getClaimAsString("name"),
                        jwtAuth.getToken().getClaimAsString("email"));
            } catch (NumberFormatException ignored) {
                // sub가 숫자가 아닌 토큰(외부 발급 등)은 미러링 생략
            }
        }
        chain.doFilter(request, response);
    }
}
```

`me/dto/GrantResponse.java`:
```java
package com.platform.orgservice.me.dto;

import com.platform.orgservice.domain.GrantEntry;

public record GrantResponse(String resourceType, String resourceId, String role) {
    public static GrantResponse from(GrantEntry g) {
        return new GrantResponse(g.getResourceType().name(), g.getResourceId(), g.getRole().name());
    }
}
```

`me/MeController.java`:
```java
package com.platform.orgservice.me;

import com.platform.orgservice.me.dto.GrantResponse;
import com.platform.orgservice.permission.PermissionFacade;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/api/org/me")
@RequiredArgsConstructor
public class MeController {

    private final PermissionFacade permissions;

    /** 내 grant 목록 — 프론트 메뉴/버튼 제어용. */
    @GetMapping("/permissions")
    public List<GrantResponse> myPermissions(@AuthenticationPrincipal Jwt jwt) {
        long userId = Long.parseLong(jwt.getSubject());
        return permissions.grantsOf(userId, null).stream().map(GrantResponse::from).toList();
    }
}
```

- [ ] **Step 4: 테스트 통과 확인**

```powershell
.\gradlew.bat :org-service:test --tests "com.platform.orgservice.me.*"
```
Expected: PASS 3건.

- [ ] **Step 5: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\platform-backend add -A
git -C C:\MSA_TEMPLATE\platform-backend commit -m "feat: JWT 리소스 서버 + JIT 멤버 미러링 + /api/org/me/permissions"
```

---

### Task 7: 부트스트랩 관리자 시드 + 팀 REST CRUD

**Files:**
- Create: `org-service/src/main/java/com/platform/orgservice/config/BootstrapAdminSeeder.java`
- Create: `org-service/src/main/java/com/platform/orgservice/team/TeamController.java`
- Create: `org-service/src/main/java/com/platform/orgservice/team/TeamService.java`
- Create: `org-service/src/main/java/com/platform/orgservice/team/dto/TeamCreateRequest.java`, `TeamUpdateRequest.java`, `TeamResponse.java`
- Create: `org-service/src/main/java/com/platform/orgservice/common/ApiExceptionHandler.java`, `common/NotFoundException.java`
- Test: `org-service/src/test/java/com/platform/orgservice/team/TeamControllerTest.java`
- Test: `org-service/src/test/java/com/platform/orgservice/config/BootstrapAdminSeederTest.java`

**Interfaces:**
- Consumes: `PermissionFacade.requireGlobalAdmin`, `TeamRepository`, `TeamMemberRepository`, `GrantEntry.globalAdmin`
- Produces:
  - REST: `GET/POST /api/org/teams`, `PUT/DELETE /api/org/teams/{id}`, `PUT/DELETE /api/org/teams/{id}/members/{memberId}?role=`
  - `NotFoundException` → 404, `AccessDeniedException` → 403, `IllegalArgumentException` → 400 (ApiExceptionHandler)
  - 부트스트랩 계약: `platform.bootstrap-admin-id` 설정 시 기동마다 GLOBAL ADMIN grant upsert

- [ ] **Step 1: 실패하는 테스트 작성**

`org-service/src/test/java/com/platform/orgservice/config/BootstrapAdminSeederTest.java`:
```java
package com.platform.orgservice.config;

import com.platform.orgservice.domain.GrantRole;
import com.platform.orgservice.domain.ResourceKind;
import com.platform.orgservice.domain.SubjectType;
import com.platform.orgservice.repository.GrantEntryRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(properties = "platform.bootstrap-admin-id=42")
@ActiveProfiles("test")
class BootstrapAdminSeederTest {

    @Autowired GrantEntryRepository grants;

    @Test
    void 기동_시_지정_계정이_GLOBAL_ADMIN으로_시드된다() {
        assertThat(grants.findBySubjectTypeAndSubjectIdAndResourceTypeAndResourceId(
                SubjectType.USER, 42L, ResourceKind.GLOBAL, ""))
                .isPresent()
                .get().satisfies(g -> assertThat(g.getRole()).isEqualTo(GrantRole.ADMIN));
    }
}
```

`org-service/src/test/java/com/platform/orgservice/team/TeamControllerTest.java`:
```java
package com.platform.orgservice.team;

import com.platform.orgservice.domain.GrantEntry;
import com.platform.orgservice.repository.GrantEntryRepository;
import com.platform.orgservice.repository.MemberRepository;
import com.platform.orgservice.repository.TeamMemberRepository;
import com.platform.orgservice.repository.TeamRepository;
import com.platform.orgservice.domain.Member;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import static com.platform.orgservice.TestAuth.asUser;
import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@ActiveProfiles("test")
class TeamControllerTest {

    @Autowired WebApplicationContext context;
    @Autowired GrantEntryRepository grants;
    @Autowired TeamRepository teams;
    @Autowired TeamMemberRepository teamMembers;
    @Autowired MemberRepository members;
    MockMvc mvc;

    static final long ADMIN_ID = 100L;
    static final long USER_ID = 200L;

    @BeforeEach
    void setup() {
        mvc = MockMvcBuilders.webAppContextSetup(context).apply(springSecurity()).build();
        teamMembers.deleteAll();
        teams.deleteAll();
        grants.deleteAll();
        members.deleteAll();
        grants.save(GrantEntry.globalAdmin(ADMIN_ID));
    }

    @Test
    void 팀_생성은_GLOBAL_ADMIN만_가능하다() throws Exception {
        mvc.perform(post("/api/org/teams").with(asUser(USER_ID, "Bob"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"name\":\"dev\",\"description\":\"개발팀\"}"))
                .andExpect(status().isForbidden());

        mvc.perform(post("/api/org/teams").with(asUser(ADMIN_ID, "Admin"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"name\":\"dev\",\"description\":\"개발팀\"}"))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.name").value("dev"));
    }

    @Test
    void 팀_목록은_인증만_있으면_조회된다() throws Exception {
        mvc.perform(get("/api/org/teams").with(asUser(USER_ID, "Bob")))
                .andExpect(status().isOk());
    }

    @Test
    void 팀원_추가와_제거가_동작한다() throws Exception {
        members.save(Member.of(USER_ID, "Bob", null));
        String body = mvc.perform(post("/api/org/teams").with(asUser(ADMIN_ID, "Admin"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"name\":\"dev\",\"description\":null}"))
                .andReturn().getResponse().getContentAsString();
        long teamId = com.jayway.jsonpath.JsonPath.parse(body).read("$.id", Long.class);

        mvc.perform(put("/api/org/teams/" + teamId + "/members/" + USER_ID + "?role=MEMBER")
                        .with(asUser(ADMIN_ID, "Admin")))
                .andExpect(status().isOk());

        mvc.perform(delete("/api/org/teams/" + teamId + "/members/" + USER_ID)
                        .with(asUser(ADMIN_ID, "Admin")))
                .andExpect(status().isNoContent());
    }

    @Test
    void 없는_팀_수정은_404() throws Exception {
        mvc.perform(put("/api/org/teams/99999").with(asUser(ADMIN_ID, "Admin"))
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"name\":\"x\",\"description\":null}"))
                .andExpect(status().isNotFound());
    }
}
```

- [ ] **Step 2: 실행해서 실패 확인**

```powershell
.\gradlew.bat :org-service:test --tests "com.platform.orgservice.team.*" --tests "com.platform.orgservice.config.*"
```
Expected: FAIL — TeamController/BootstrapAdminSeeder 심볼 없음.

- [ ] **Step 3: 구현**

`common/NotFoundException.java`:
```java
package com.platform.orgservice.common;

public class NotFoundException extends RuntimeException {
    public NotFoundException(String message) { super(message); }
}
```

`common/ApiExceptionHandler.java`:
```java
package com.platform.orgservice.common;

import org.springframework.http.HttpStatus;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.Map;

@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, String> notFound(NotFoundException e) {
        return Map.of("error", e.getMessage());
    }

    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public Map<String, String> forbidden(AccessDeniedException e) {
        return Map.of("error", e.getMessage());
    }

    @ExceptionHandler(IllegalArgumentException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, String> badRequest(IllegalArgumentException e) {
        return Map.of("error", e.getMessage());
    }
}
```

`config/BootstrapAdminSeeder.java`:
```java
package com.platform.orgservice.config;

import com.platform.orgservice.domain.GrantEntry;
import com.platform.orgservice.domain.GrantRole;
import com.platform.orgservice.domain.ResourceKind;
import com.platform.orgservice.domain.SubjectType;
import com.platform.orgservice.repository.GrantEntryRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

/**
 * 최초 관리자 부트스트랩 — PLATFORM_BOOTSTRAP_ADMIN_ID(auth user id)를
 * 기동마다 GLOBAL ADMIN grant로 upsert. 미설정이면 아무것도 안 함.
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class BootstrapAdminSeeder implements ApplicationRunner {

    private final GrantEntryRepository grants;

    @Value("${platform.bootstrap-admin-id:}")
    private String bootstrapAdminId;

    @Override
    @Transactional
    public void run(ApplicationArguments args) {
        if (bootstrapAdminId == null || bootstrapAdminId.isBlank()) return;
        long userId = Long.parseLong(bootstrapAdminId.trim());
        grants.findBySubjectTypeAndSubjectIdAndResourceTypeAndResourceId(
                        SubjectType.USER, userId, ResourceKind.GLOBAL, "")
                .ifPresentOrElse(
                        g -> g.changeRole(GrantRole.ADMIN), // 강등돼 있어도 기동 시 복구
                        () -> grants.save(GrantEntry.globalAdmin(userId)));
        log.info("부트스트랩 GLOBAL ADMIN 시드: userId={}", userId);
    }
}
```

`team/dto/TeamCreateRequest.java`:
```java
package com.platform.orgservice.team.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record TeamCreateRequest(@NotBlank @Size(max = 100) String name, String description) {}
```

`team/dto/TeamUpdateRequest.java`:
```java
package com.platform.orgservice.team.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record TeamUpdateRequest(@NotBlank @Size(max = 100) String name, String description) {}
```

`team/dto/TeamResponse.java`:
```java
package com.platform.orgservice.team.dto;

import com.platform.orgservice.domain.Team;

public record TeamResponse(Long id, String name, String description) {
    public static TeamResponse from(Team t) {
        return new TeamResponse(t.getId(), t.getName(), t.getDescription());
    }
}
```

`team/TeamService.java`:
```java
package com.platform.orgservice.team;

import com.platform.orgservice.common.NotFoundException;
import com.platform.orgservice.domain.Team;
import com.platform.orgservice.domain.TeamMember;
import com.platform.orgservice.domain.TeamRole;
import com.platform.orgservice.permission.PermissionFacade;
import com.platform.orgservice.repository.TeamMemberRepository;
import com.platform.orgservice.repository.TeamRepository;
import com.platform.orgservice.team.dto.TeamCreateRequest;
import com.platform.orgservice.team.dto.TeamResponse;
import com.platform.orgservice.team.dto.TeamUpdateRequest;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
@Transactional
public class TeamService {

    private final TeamRepository teams;
    private final TeamMemberRepository teamMembers;
    private final PermissionFacade permissions;

    @Transactional(readOnly = true)
    public List<TeamResponse> list() {
        return teams.findAll().stream().map(TeamResponse::from).toList();
    }

    public TeamResponse create(long actorId, TeamCreateRequest req) {
        permissions.requireGlobalAdmin(actorId);
        if (teams.existsByName(req.name())) throw new IllegalArgumentException("이미 존재하는 팀 이름: " + req.name());
        return TeamResponse.from(teams.save(Team.of(req.name(), req.description())));
    }

    public TeamResponse update(long actorId, long teamId, TeamUpdateRequest req) {
        permissions.requireGlobalAdmin(actorId);
        Team team = teams.findById(teamId).orElseThrow(() -> new NotFoundException("팀 없음: " + teamId));
        team.update(req.name(), req.description());
        return TeamResponse.from(team);
    }

    public void delete(long actorId, long teamId) {
        permissions.requireGlobalAdmin(actorId);
        if (!teams.existsById(teamId)) throw new NotFoundException("팀 없음: " + teamId);
        teams.deleteById(teamId);
    }

    public void addMember(long actorId, long teamId, long memberId, TeamRole role) {
        permissions.requireGlobalAdmin(actorId);
        if (!teams.existsById(teamId)) throw new NotFoundException("팀 없음: " + teamId);
        teamMembers.findByTeamIdAndMemberId(teamId, memberId).ifPresentOrElse(
                tm -> { /* 이미 팀원 — 멱등 */ },
                () -> teamMembers.save(TeamMember.of(teamId, memberId, role)));
    }

    public void removeMember(long actorId, long teamId, long memberId) {
        permissions.requireGlobalAdmin(actorId);
        teamMembers.findByTeamIdAndMemberId(teamId, memberId)
                .ifPresent(teamMembers::delete);
    }
}
```

`team/TeamController.java`:
```java
package com.platform.orgservice.team;

import com.platform.orgservice.domain.TeamRole;
import com.platform.orgservice.team.dto.TeamCreateRequest;
import com.platform.orgservice.team.dto.TeamResponse;
import com.platform.orgservice.team.dto.TeamUpdateRequest;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/org/teams")
@RequiredArgsConstructor
public class TeamController {

    private final TeamService teams;

    @GetMapping
    public List<TeamResponse> list() {
        return teams.list();
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public TeamResponse create(@AuthenticationPrincipal Jwt jwt, @Valid @RequestBody TeamCreateRequest req) {
        return teams.create(userId(jwt), req);
    }

    @PutMapping("/{id}")
    public TeamResponse update(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id,
                               @Valid @RequestBody TeamUpdateRequest req) {
        return teams.update(userId(jwt), id, req);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id) {
        teams.delete(userId(jwt), id);
    }

    @PutMapping("/{id}/members/{memberId}")
    public void addMember(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id,
                          @PathVariable Long memberId, @RequestParam(defaultValue = "MEMBER") TeamRole role) {
        teams.addMember(userId(jwt), id, memberId, role);
    }

    @DeleteMapping("/{id}/members/{memberId}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void removeMember(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id, @PathVariable Long memberId) {
        teams.removeMember(userId(jwt), id, memberId);
    }

    private static long userId(Jwt jwt) { return Long.parseLong(jwt.getSubject()); }
}
```

- [ ] **Step 4: 테스트 통과 확인**

```powershell
.\gradlew.bat :org-service:test --tests "com.platform.orgservice.team.*" --tests "com.platform.orgservice.config.*"
```
Expected: PASS 5건.

- [ ] **Step 5: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\platform-backend add -A
git -C C:\MSA_TEMPLATE\platform-backend commit -m "feat: 부트스트랩 GLOBAL ADMIN 시드 + 팀 REST CRUD(관리자 가드)"
```

---

### Task 8: 멤버 목록 + grant 관리 REST

**Files:**
- Create: `org-service/src/main/java/com/platform/orgservice/member/MemberController.java`, `member/dto/MemberResponse.java`
- Create: `org-service/src/main/java/com/platform/orgservice/grant/GrantController.java`, `grant/GrantService.java`, `grant/dto/GrantCreateRequest.java`, `grant/dto/GrantDetailResponse.java`
- Test: `org-service/src/test/java/com/platform/orgservice/grant/GrantControllerTest.java`

**Interfaces:**
- Consumes: `PermissionFacade.requireGlobalAdmin`, `GrantEntryRepository`, `MemberRepository`, Task 6의 `TestAuth`
- Produces:
  - `GET /api/org/members` (인증), `GET /api/org/grants?resourceType=&resourceId=` (ADMIN), `POST /api/org/grants` (ADMIN, 201), `DELETE /api/org/grants/{id}` (ADMIN, 204)

- [ ] **Step 1: 실패하는 테스트 작성**

`org-service/src/test/java/com/platform/orgservice/grant/GrantControllerTest.java`:
```java
package com.platform.orgservice.grant;

import com.platform.orgservice.domain.GrantEntry;
import com.platform.orgservice.repository.GrantEntryRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import static com.platform.orgservice.TestAuth.asUser;
import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@ActiveProfiles("test")
class GrantControllerTest {

    @Autowired WebApplicationContext context;
    @Autowired GrantEntryRepository grants;
    MockMvc mvc;

    static final long ADMIN_ID = 100L;
    static final long USER_ID = 200L;

    @BeforeEach
    void setup() {
        mvc = MockMvcBuilders.webAppContextSetup(context).apply(springSecurity()).build();
        grants.deleteAll();
        grants.save(GrantEntry.globalAdmin(ADMIN_ID));
    }

    @Test
    void grant_부여는_ADMIN만_가능하다() throws Exception {
        String body = "{\"subjectType\":\"USER\",\"subjectId\":200,\"resourceType\":\"SPACE\",\"resourceId\":\"sp-1\",\"role\":\"EDITOR\"}";

        mvc.perform(post("/api/org/grants").with(asUser(USER_ID, "Bob"))
                        .contentType(MediaType.APPLICATION_JSON).content(body))
                .andExpect(status().isForbidden());

        mvc.perform(post("/api/org/grants").with(asUser(ADMIN_ID, "Admin"))
                        .contentType(MediaType.APPLICATION_JSON).content(body))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.role").value("EDITOR"));
    }

    @Test
    void 리소스별_grant_목록을_조회하고_회수한다() throws Exception {
        String body = "{\"subjectType\":\"USER\",\"subjectId\":200,\"resourceType\":\"SPACE\",\"resourceId\":\"sp-1\",\"role\":\"VIEWER\"}";
        String created = mvc.perform(post("/api/org/grants").with(asUser(ADMIN_ID, "Admin"))
                        .contentType(MediaType.APPLICATION_JSON).content(body))
                .andReturn().getResponse().getContentAsString();
        long grantId = com.jayway.jsonpath.JsonPath.parse(created).read("$.id", Long.class);

        mvc.perform(get("/api/org/grants?resourceType=SPACE&resourceId=sp-1").with(asUser(ADMIN_ID, "Admin")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.length()").value(1));

        mvc.perform(delete("/api/org/grants/" + grantId).with(asUser(ADMIN_ID, "Admin")))
                .andExpect(status().isNoContent());
    }

    @Test
    void 중복_grant는_400() throws Exception {
        String body = "{\"subjectType\":\"USER\",\"subjectId\":200,\"resourceType\":\"SPACE\",\"resourceId\":\"sp-1\",\"role\":\"VIEWER\"}";
        mvc.perform(post("/api/org/grants").with(asUser(ADMIN_ID, "Admin"))
                .contentType(MediaType.APPLICATION_JSON).content(body)).andExpect(status().isCreated());
        mvc.perform(post("/api/org/grants").with(asUser(ADMIN_ID, "Admin"))
                .contentType(MediaType.APPLICATION_JSON).content(body)).andExpect(status().isBadRequest());
    }

    @Test
    void 멤버_목록은_인증만_있으면_조회된다() throws Exception {
        mvc.perform(get("/api/org/members").with(asUser(USER_ID, "Bob")))
                .andExpect(status().isOk());
    }
}
```

- [ ] **Step 2: 실행해서 실패 확인**

```powershell
.\gradlew.bat :org-service:test --tests "com.platform.orgservice.grant.*"
```
Expected: FAIL — GrantController 심볼 없음.

- [ ] **Step 3: 구현**

`member/dto/MemberResponse.java`:
```java
package com.platform.orgservice.member.dto;

import com.platform.orgservice.domain.Member;

public record MemberResponse(Long id, String displayName, String email, String status) {
    public static MemberResponse from(Member m) {
        return new MemberResponse(m.getId(), m.getDisplayName(), m.getEmail(), m.getStatus().name());
    }
}
```

`member/MemberController.java`:
```java
package com.platform.orgservice.member;

import com.platform.orgservice.member.dto.MemberResponse;
import com.platform.orgservice.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/api/org/members")
@RequiredArgsConstructor
public class MemberController {

    private final MemberRepository members;

    /** 멤버 목록 — 팀원 선택 UI용. M 규모(수백 명)라 페이지네이션 없이 전체 반환. */
    @GetMapping
    public List<MemberResponse> list() {
        return members.findAll().stream().map(MemberResponse::from).toList();
    }
}
```

`grant/dto/GrantCreateRequest.java`:
```java
package com.platform.orgservice.grant.dto;

import com.platform.orgservice.domain.GrantRole;
import com.platform.orgservice.domain.ResourceKind;
import com.platform.orgservice.domain.SubjectType;
import jakarta.validation.constraints.NotNull;

public record GrantCreateRequest(
        @NotNull SubjectType subjectType,
        @NotNull Long subjectId,
        @NotNull ResourceKind resourceType,
        String resourceId,   // GLOBAL이면 null/'' 허용
        @NotNull GrantRole role) {}
```

`grant/dto/GrantDetailResponse.java`:
```java
package com.platform.orgservice.grant.dto;

import com.platform.orgservice.domain.GrantEntry;

public record GrantDetailResponse(Long id, String subjectType, Long subjectId,
                                  String resourceType, String resourceId, String role) {
    public static GrantDetailResponse from(GrantEntry g) {
        return new GrantDetailResponse(g.getId(), g.getSubjectType().name(), g.getSubjectId(),
                g.getResourceType().name(), g.getResourceId(), g.getRole().name());
    }
}
```

`grant/GrantService.java`:
```java
package com.platform.orgservice.grant;

import com.platform.orgservice.common.NotFoundException;
import com.platform.orgservice.domain.GrantEntry;
import com.platform.orgservice.domain.ResourceKind;
import com.platform.orgservice.grant.dto.GrantCreateRequest;
import com.platform.orgservice.grant.dto.GrantDetailResponse;
import com.platform.orgservice.permission.PermissionFacade;
import com.platform.orgservice.repository.GrantEntryRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
@Transactional
public class GrantService {

    private final GrantEntryRepository grants;
    private final PermissionFacade permissions;

    @Transactional(readOnly = true)
    public List<GrantDetailResponse> listByResource(long actorId, ResourceKind resourceType, String resourceId) {
        permissions.requireGlobalAdmin(actorId);
        return grants.findByResourceTypeAndResourceId(resourceType, resourceId == null ? "" : resourceId)
                .stream().map(GrantDetailResponse::from).toList();
    }

    public GrantDetailResponse create(long actorId, GrantCreateRequest req) {
        permissions.requireGlobalAdmin(actorId);
        try {
            GrantEntry saved = grants.saveAndFlush(GrantEntry.of(
                    req.subjectType(), req.subjectId(), req.resourceType(), req.resourceId(), req.role()));
            return GrantDetailResponse.from(saved);
        } catch (DataIntegrityViolationException e) {
            throw new IllegalArgumentException("이미 존재하는 grant (subject·resource 조합 중복)");
        }
    }

    public void delete(long actorId, long grantId) {
        permissions.requireGlobalAdmin(actorId);
        if (!grants.existsById(grantId)) throw new NotFoundException("grant 없음: " + grantId);
        grants.deleteById(grantId);
    }
}
```

`grant/GrantController.java`:
```java
package com.platform.orgservice.grant;

import com.platform.orgservice.domain.ResourceKind;
import com.platform.orgservice.grant.dto.GrantCreateRequest;
import com.platform.orgservice.grant.dto.GrantDetailResponse;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/org/grants")
@RequiredArgsConstructor
public class GrantController {

    private final GrantService grants;

    @GetMapping
    public List<GrantDetailResponse> listByResource(@AuthenticationPrincipal Jwt jwt,
                                                    @RequestParam ResourceKind resourceType,
                                                    @RequestParam(required = false) String resourceId) {
        return grants.listByResource(userId(jwt), resourceType, resourceId);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public GrantDetailResponse create(@AuthenticationPrincipal Jwt jwt, @Valid @RequestBody GrantCreateRequest req) {
        return grants.create(userId(jwt), req);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@AuthenticationPrincipal Jwt jwt, @PathVariable Long id) {
        grants.delete(userId(jwt), id);
    }

    private static long userId(Jwt jwt) { return Long.parseLong(jwt.getSubject()); }
}
```

- [ ] **Step 4: 테스트 통과 확인 + 전체 회귀**

```powershell
.\gradlew.bat :org-service:test
```
Expected: 전체 PASS.

- [ ] **Step 5: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\platform-backend add -A
git -C C:\MSA_TEMPLATE\platform-backend commit -m "feat: 멤버 목록 + grant 부여/회수 REST(관리자 가드)"
```

---

### Task 9: gRPC PermissionService 서버

**Files:**
- Create: `org-service/src/main/java/com/platform/orgservice/grpc/PermissionGrpcService.java`
- Create: `org-service/src/main/java/com/platform/orgservice/grpc/GrpcServerLifecycle.java`
- Test: `org-service/src/test/java/com/platform/orgservice/grpc/PermissionGrpcServiceTest.java`

**Interfaces:**
- Consumes: `PermissionFacade`, common-proto 생성 클래스(`PermissionServiceGrpc.PermissionServiceImplBase`, `CheckPermissionRequest/Response`, `ListUserGrantsRequest/Response`, `Grant`, enum `ResourceType/Action/Role`)
- Produces: `:9131` gRPC 서버(리플렉션 포함, `platform.grpc.enabled=false`로 비활성 가능). proto↔도메인 enum 매핑 규칙: `Role.ROLE_ADMIN ↔ GrantRole.ADMIN`, effective_role 없으면 `ROLE_UNSPECIFIED`

- [ ] **Step 1: 실패하는 테스트 작성 (in-process)**

`org-service/src/test/java/com/platform/orgservice/grpc/PermissionGrpcServiceTest.java`:
```java
package com.platform.orgservice.grpc;

import com.platform.orgservice.domain.GrantEntry;
import com.platform.orgservice.domain.GrantRole;
import com.platform.orgservice.domain.ResourceKind;
import com.platform.orgservice.domain.SubjectType;
import com.platform.orgservice.repository.GrantEntryRepository;
import com.platform.proto.org.v1.*;
import io.grpc.ManagedChannel;
import io.grpc.Server;
import io.grpc.inprocess.InProcessChannelBuilder;
import io.grpc.inprocess.InProcessServerBuilder;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

import java.io.IOException;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@ActiveProfiles("test")
class PermissionGrpcServiceTest {

    @Autowired PermissionGrpcService grpcService;
    @Autowired GrantEntryRepository grants;

    Server server;
    ManagedChannel channel;
    PermissionServiceGrpc.PermissionServiceBlockingStub stub;

    @BeforeEach
    void setup() throws IOException {
        grants.deleteAll();
        String name = InProcessServerBuilder.generateName();
        server = InProcessServerBuilder.forName(name).directExecutor().addService(grpcService).build().start();
        channel = InProcessChannelBuilder.forName(name).directExecutor().build();
        stub = PermissionServiceGrpc.newBlockingStub(channel);
    }

    @AfterEach
    void teardown() {
        channel.shutdownNow();
        server.shutdownNow();
    }

    @Test
    void CheckPermission_grant보유자는_allowed_true() {
        grants.save(GrantEntry.of(SubjectType.USER, 1L, ResourceKind.SPACE, "sp-1", GrantRole.EDITOR));

        CheckPermissionResponse res = stub.checkPermission(CheckPermissionRequest.newBuilder()
                .setUserId(1L).setResourceType(ResourceType.SPACE).setResourceId("sp-1")
                .setAction(Action.EDIT).build());

        assertThat(res.getAllowed()).isTrue();
        assertThat(res.getEffectiveRole()).isEqualTo(Role.EDITOR);
    }

    @Test
    void CheckPermission_무grant는_allowed_false_role_UNSPECIFIED() {
        CheckPermissionResponse res = stub.checkPermission(CheckPermissionRequest.newBuilder()
                .setUserId(9L).setResourceType(ResourceType.SPACE).setResourceId("sp-1")
                .setAction(Action.VIEW).build());

        assertThat(res.getAllowed()).isFalse();
        assertThat(res.getEffectiveRole()).isEqualTo(Role.ROLE_UNSPECIFIED);
    }

    @Test
    void ListUserGrants_리소스타입_필터가_동작한다() {
        grants.save(GrantEntry.of(SubjectType.USER, 2L, ResourceKind.SPACE, "sp-1", GrantRole.VIEWER));
        grants.save(GrantEntry.of(SubjectType.USER, 2L, ResourceKind.PROJECT, "pj-1", GrantRole.ADMIN));

        ListUserGrantsResponse all = stub.listUserGrants(
                ListUserGrantsRequest.newBuilder().setUserId(2L).build());
        ListUserGrantsResponse spaceOnly = stub.listUserGrants(
                ListUserGrantsRequest.newBuilder().setUserId(2L).setResourceType(ResourceType.SPACE).build());

        assertThat(all.getGrantsList()).hasSize(2);
        assertThat(spaceOnly.getGrantsList()).hasSize(1);
        assertThat(spaceOnly.getGrants(0).getRole()).isEqualTo(Role.VIEWER);
    }
}
```

- [ ] **Step 2: 실행해서 실패 확인**

```powershell
.\gradlew.bat :org-service:test --tests "com.platform.orgservice.grpc.*"
```
Expected: FAIL — PermissionGrpcService 심볼 없음.

- [ ] **Step 3: 구현 (2개 파일)**

`grpc/PermissionGrpcService.java`:
```java
package com.platform.orgservice.grpc;

import com.platform.orgservice.domain.GrantRole;
import com.platform.orgservice.domain.PermAction;
import com.platform.orgservice.domain.ResourceKind;
import com.platform.orgservice.permission.PermissionFacade;
import com.platform.proto.org.v1.*;
import io.grpc.Status;
import io.grpc.stub.StreamObserver;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

/** platform.org.v1.PermissionService 구현 — 판정 로직은 PermissionFacade에 위임. */
@Service
@RequiredArgsConstructor
public class PermissionGrpcService extends PermissionServiceGrpc.PermissionServiceImplBase {

    private final PermissionFacade permissions;

    @Override
    public void checkPermission(CheckPermissionRequest req, StreamObserver<CheckPermissionResponse> out) {
        ResourceKind kind = toKind(req.getResourceType());
        PermAction action = toAction(req.getAction());
        if (kind == null || action == null) {
            out.onError(Status.INVALID_ARGUMENT
                    .withDescription("resource_type/action은 UNSPECIFIED일 수 없습니다").asRuntimeException());
            return;
        }
        PermissionFacade.Decision d = permissions.check(req.getUserId(), kind, req.getResourceId(), action);
        out.onNext(CheckPermissionResponse.newBuilder()
                .setAllowed(d.allowed())
                .setEffectiveRole(toProtoRole(d.effectiveRole()))
                .build());
        out.onCompleted();
    }

    @Override
    public void listUserGrants(ListUserGrantsRequest req, StreamObserver<ListUserGrantsResponse> out) {
        ResourceKind kind = toKind(req.getResourceType()); // UNSPECIFIED → null → 전체
        ListUserGrantsResponse.Builder res = ListUserGrantsResponse.newBuilder();
        permissions.grantsOf(req.getUserId(), kind).forEach(g -> res.addGrants(Grant.newBuilder()
                .setResourceType(toProtoKind(g.getResourceType()))
                .setResourceId(g.getResourceId())
                .setRole(toProtoRole(g.getRole()))
                .build()));
        out.onNext(res.build());
        out.onCompleted();
    }

    private static ResourceKind toKind(ResourceType t) {
        return switch (t) {
            case GLOBAL -> ResourceKind.GLOBAL;
            case SPACE -> ResourceKind.SPACE;
            case PROJECT -> ResourceKind.PROJECT;
            default -> null;
        };
    }

    private static PermAction toAction(Action a) {
        return switch (a) {
            case VIEW -> PermAction.VIEW;
            case EDIT -> PermAction.EDIT;
            case ADMIN -> PermAction.ADMIN;
            default -> null;
        };
    }

    private static ResourceType toProtoKind(ResourceKind k) {
        return switch (k) {
            case GLOBAL -> ResourceType.GLOBAL;
            case SPACE -> ResourceType.SPACE;
            case PROJECT -> ResourceType.PROJECT;
        };
    }

    /** proto Role.ROLE_ADMIN ↔ 도메인 GrantRole.ADMIN (proto enum 스코프 충돌 회피 명명). */
    private static Role toProtoRole(GrantRole r) {
        if (r == null) return Role.ROLE_UNSPECIFIED;
        return switch (r) {
            case VIEWER -> Role.VIEWER;
            case EDITOR -> Role.EDITOR;
            case ADMIN -> Role.ROLE_ADMIN;
        };
    }
}
```

`grpc/GrpcServerLifecycle.java`:
```java
package com.platform.orgservice.grpc;

import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.protobuf.services.ProtoReflectionService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.context.SmartLifecycle;
import org.springframework.stereotype.Component;

import java.io.IOException;

/**
 * 순정 grpc-java 서버 수명주기 — 스타터 없이 SmartLifecycle로 Boot에 편입.
 * (스펙: spring-grpc는 Boot 4 호환 확정 후 서비스 수 늘면 재평가)
 * 리플렉션 서비스 포함 — grpcurl로 계약 탐색/디버깅 가능.
 */
@Component
@ConditionalOnProperty(value = "platform.grpc.enabled", havingValue = "true", matchIfMissing = true)
@RequiredArgsConstructor
@Slf4j
public class GrpcServerLifecycle implements SmartLifecycle {

    private final PermissionGrpcService permissionGrpcService;

    @Value("${platform.grpc.port:9131}")
    private int port;

    private Server server;

    @Override
    public void start() {
        try {
            server = ServerBuilder.forPort(port)
                    .addService(permissionGrpcService)
                    .addService(ProtoReflectionService.newInstance())
                    .build()
                    .start();
            log.info("gRPC 서버 기동: :{}", port);
        } catch (IOException e) {
            throw new IllegalStateException("gRPC 서버 기동 실패 :" + port, e);
        }
    }

    @Override
    public void stop() {
        if (server != null) {
            server.shutdown();
            log.info("gRPC 서버 종료: :{}", port);
        }
    }

    @Override
    public boolean isRunning() {
        return server != null && !server.isShutdown();
    }
}
```

- [ ] **Step 4: 테스트 통과 + 전체 회귀**

```powershell
.\gradlew.bat build
```
Expected: 전 모듈 BUILD SUCCESSFUL, 테스트 전체 PASS.

- [ ] **Step 5: 커밋**

```powershell
git -C C:\MSA_TEMPLATE\platform-backend add -A
git -C C:\MSA_TEMPLATE\platform-backend commit -m "feat: gRPC PermissionService(:9131) — 순정 grpc-java Server + 리플렉션"
```

---

### Task 10: Dockerfile + compose 통합 + 게이트웨이 라우트 + orgdb

**Files:**
- Create: `org-service/Dockerfile`
- Create: `org-service/.run/bootRun.run.xml`
- Modify: `C:\MSA_TEMPLATE\infra\keycloak\init-authdb.sql`
- Modify: `C:\MSA_TEMPLATE\infra\keycloak\docker-compose.yml`
- Modify: `C:\MSA_TEMPLATE\gateway-server\src\main\resources\application.yml` (routes 블록)

**Interfaces:**
- Consumes: Task 9까지의 완성된 org-service (bootJar → `org-service/build/libs/app.jar`)
- Produces: compose 컨테이너 `org-service`(GHCR image + build 폴백), 게이트웨이 라우트 `/api/org/**`(env `ORG_SERVICE_URI` 오버라이드)

- [ ] **Step 1: Dockerfile + IntelliJ Run Config**

`org-service/Dockerfile` (기존 4서비스 런타임 전용 패턴 — 빌드 컨텍스트는 repo 루트):
```dockerfile
# 런타임 전용. jar는 CI(또는 로컬)의 `gradlew :org-service:bootJar` 산출물을 복사한다.
FROM eclipse-temurin:24-jre
WORKDIR /app
COPY org-service/build/libs/app.jar app.jar
EXPOSE 9130 9131
ENTRYPOINT ["java", "-jar", "app.jar"]
```

`org-service/.run/bootRun.run.xml` (기존 서비스 공유 Run Config 패턴):
```xml
<component name="ProjectRunConfigurationManager">
  <configuration default="false" name="org-service bootRun" type="GradleRunConfiguration" factoryName="Gradle">
    <ExternalSystemSettings>
      <option name="executionName" />
      <option name="externalProjectPath" value="$PROJECT_DIR$" />
      <option name="externalSystemIdString" value="GRADLE" />
      <option name="scriptParameters" value="" />
      <option name="taskDescriptions"><list /></option>
      <option name="taskNames"><list><option value=":org-service:bootRun" /></list></option>
      <option name="vmOptions" value="" />
    </ExternalSystemSettings>
    <method v="2" />
  </configuration>
</component>
```

- [ ] **Step 2: init-authdb.sql에 orgdb 추가**

`C:\MSA_TEMPLATE\infra\keycloak\init-authdb.sql` 끝에 추가:
```sql
-- org-service(platform-backend)용. Flyway가 스키마 생성.
CREATE DATABASE orgdb;
```

- [ ] **Step 3: 기존 볼륨에 orgdb 수동 생성 (init 스크립트는 빈 볼륨에서만 실행됨)**

```powershell
docker exec platform-postgres psql -U keycloak -c "CREATE DATABASE orgdb;"
```
Expected: `CREATE DATABASE`. (postgres 컨테이너 미기동이면 `cd C:\MSA_TEMPLATE\infra\keycloak; docker compose up -d postgres` 먼저)

- [ ] **Step 4: compose에 org-service 추가 + 게이트웨이에 ORG_SERVICE_URI**

`docker-compose.yml`의 `board-service:` 블록 아래에 추가:
```yaml
  # platform-backend 횡단 서비스 1호 — REST :9130(게이트웨이 경유) + gRPC :9131(내부 직통).
  # docker 프로필: Eureka 미등록(15번 문서 확정) — 게이트웨이가 ORG_SERVICE_URI(DNS)로 직결.
  org-service:
    image: ghcr.io/chanho4702/org-service:${ORG_TAG:-latest}
    build:
      context: ../../platform-backend
      dockerfile: org-service/Dockerfile
    container_name: platform-org
    environment:
      SPRING_PROFILES_ACTIVE: docker
      ORG_DB_URL: jdbc:postgresql://postgres:5432/orgdb
      AUTH_JWKS_URI: http://auth-server:9000/.well-known/jwks.json
      PLATFORM_BOOTSTRAP_ADMIN_ID: ${PLATFORM_BOOTSTRAP_ADMIN_ID:-1}
    depends_on:
      postgres: { condition: service_healthy }
    healthcheck:
      test: ["CMD-SHELL", "bash -c 'exec 3<>/dev/tcp/127.0.0.1/9130'"]
      interval: 5s
      timeout: 3s
      retries: 20
      start_period: 40s
```

같은 파일 `gateway:` 서비스의 `environment:`에 한 줄 추가:
```yaml
      ORG_SERVICE_URI: http://org-service:9130
```

- [ ] **Step 5: 게이트웨이 라우트 추가**

`C:\MSA_TEMPLATE\gateway-server\src\main\resources\application.yml`의 `routes:` 마지막(auth-me 블록 뒤)에 추가:
```yaml
            # org-service — 조직/팀/권한 관리 REST (Wave A).
            # dev: lb://(Eureka) · docker: ORG_SERVICE_URI로 DNS 직결 (Eureka 프로필 분기 1호)
            - id: org
              uri: ${ORG_SERVICE_URI:lb://org-service}
              predicates:
                - Path=/api/org/**
```

- [ ] **Step 6: 이미지 빌드 + compose 기동 스모크**

```powershell
cd C:\MSA_TEMPLATE\platform-backend; .\gradlew.bat :org-service:bootJar
cd C:\MSA_TEMPLATE\infra\keycloak
docker compose build org-service
docker compose up -d org-service
docker compose ps org-service
```
Expected: `platform-org` 상태 `healthy` (Flyway가 orgdb에 V1 적용, 부트스트랩 시드 로그).

```powershell
docker logs platform-org --tail 20
```
Expected: `부트스트랩 GLOBAL ADMIN 시드: userId=1`, `gRPC 서버 기동: :9131`, 기동 완료 로그.

- [ ] **Step 7: 커밋 (3개 repo)**

```powershell
git -C C:\MSA_TEMPLATE\platform-backend add -A
git -C C:\MSA_TEMPLATE\platform-backend commit -m "feat: org-service Dockerfile(런타임 전용) + IntelliJ Run Config"
git -C C:\MSA_TEMPLATE\gateway-server add src/main/resources/application.yml
git -C C:\MSA_TEMPLATE\gateway-server commit -m "feat: /api/org/** 라우트 — ORG_SERVICE_URI 오버라이드(dev lb:// / docker DNS)"
git -C C:\MSA_TEMPLATE add .gitignore infra/keycloak/init-authdb.sql infra/keycloak/docker-compose.yml
git -C C:\MSA_TEMPLATE commit -m "feat(infra): org-service compose 통합 + orgdb + platform-backend gitignore"
```

---

### Task 11: GitHub repo + CI (빌드·테스트 + proto 발행)

**Files:**
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- Produces: `github.com/chanho4702/platform-backend` (private), CI 계약 — PR/main push: `gradlew build`, `v*` 태그: common-proto를 태그 버전으로 GitHub Packages 발행. GHCR 이미지 push는 CI/CD Wave 2(전 서비스 일괄)에서 — compose의 `build:` 폴백으로 충분.

- [ ] **Step 1: workflow 작성**

`.github/workflows/ci.yml`:
```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '24'
      - uses: gradle/actions/setup-gradle@v4
      - run: ./gradlew build

  publish-proto:
    if: startsWith(github.ref, 'refs/tags/v')
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '24'
      - uses: gradle/actions/setup-gradle@v4
      - name: 태그 버전으로 common-proto 발행
        run: ./gradlew :common-proto:publish -PprotoVersion=${GITHUB_REF_NAME#v}
        env:
          GITHUB_ACTOR: ${{ github.actor }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- [ ] **Step 2: GitHub repo 생성 + push**

```powershell
cd C:\MSA_TEMPLATE\platform-backend
git add -A; git commit -m "ci: 빌드·테스트 + v* 태그 시 common-proto GitHub Packages 발행"
gh repo create chanho4702/platform-backend --private --source . --push
```
Expected: repo 생성 + main push. Actions 탭에서 `build` job 그린 확인:
```powershell
gh run watch --repo chanho4702/platform-backend
```

- [ ] **Step 3: v0.1.0 태그로 proto 최초 발행**

```powershell
git tag v0.1.0; git push origin v0.1.0
gh run watch --repo chanho4702/platform-backend
```
Expected: `publish-proto` job 그린. 확인:
```powershell
gh api /users/chanho4702/packages?package_type=maven
```
Expected: `com.platform.common-proto` 패키지 존재.

- [ ] **Step 4: 소비 검증 (성공 기준 ①)**

스크래치 디렉터리에 샘플 프로젝트로 resolve 확인:

`%TEMP%\proto-consume-test\settings.gradle`: `rootProject.name = 'proto-consume-test'`
`%TEMP%\proto-consume-test\build.gradle`:
```groovy
plugins { id 'java' }
java { toolchain { languageVersion = JavaLanguageVersion.of(24) } }
repositories {
    mavenCentral()
    maven {
        url = uri('https://maven.pkg.github.com/chanho4702/platform-backend')
        credentials {
            username = System.getenv('GITHUB_ACTOR') ?: 'chanho4702'
            password = System.getenv('GITHUB_TOKEN')
        }
    }
}
dependencies { implementation 'com.platform:common-proto:0.1.0' }
```
```powershell
$env:GITHUB_TOKEN = (gh auth token)
# platform-backend의 wrapper 재사용해 dependencies 확인
cd $env:TEMP\proto-consume-test
Copy-Item C:\MSA_TEMPLATE\platform-backend\gradlew.bat,C:\MSA_TEMPLATE\platform-backend\gradlew .; Copy-Item -Recurse C:\MSA_TEMPLATE\platform-backend\gradle .
.\gradlew.bat dependencies --configuration compileClasspath
```
Expected: `com.platform:common-proto:0.1.0` resolve 성공(FAILED 없음).

---

### Task 12: 수동 E2E — 성공 기준 검증 + README

**Files:**
- Create: `README.md` (platform-backend)
- Modify: `C:\MSA_TEMPLATE\README.md` (컴포넌트 표 + 토폴로지에 org-service 추가)

**Interfaces:**
- Consumes: 전체 완성 상태 (스펙 성공 기준 ①~⑤)

- [ ] **Step 1: dev 모드 E2E (성공 기준 ②)**

```powershell
# 인프라 + 백엔드: 기존 dev-up 절차대로 (eureka·auth·gateway는 IntelliJ 또는 기존 방식)
# org-service만 신규: IntelliJ 'org-service bootRun' 실행 (프로필 없음 = dev)
```
확인 순서:
1. `http://localhost:8761` — Eureka 대시보드에 `ORG-SERVICE` 등록됨
2. 브라우저 `http://localhost:5173` 로그인 → devtools에서 AT 복사
3. ```powershell
   $token = "<복사한 AT>"
   curl.exe -s -H "Authorization: Bearer $token" http://localhost:8000/api/org/me/permissions
   ```
   Expected: `[]` 또는 grant 목록 JSON, HTTP 200 (게이트웨이 lb:// 경유)

- [ ] **Step 2: docker 모드 E2E (성공 기준 ③·④)**

```powershell
cd C:\MSA_TEMPLATE\infra\keycloak
# PLATFORM_BOOTSTRAP_ADMIN_ID는 .env에서 실제 본인 auth user id로 설정 (기본 1)
docker compose up -d
docker compose ps
```
Expected: 9컨테이너 전부 healthy (기존 8 + org-service).

grpcurl 판정 (성공 기준 ③). gRPC 포트 9131은 컨테이너 내부 전용(호스트 포트 매핑 없음)이므로 컨테이너 네트워크 안에서 grpcurl 이미지로 호출한다:
```powershell
docker run --rm --network platform_default fullstorydev/grpcurl -plaintext -d '{\"user_id\": 1, \"resource_type\": \"GLOBAL\", \"action\": \"ADMIN\"}' org-service:9131 platform.org.v1.PermissionService/CheckPermission
```
Expected: `{ "allowed": true, "effectiveRole": "ROLE_ADMIN" }` (리플렉션 서비스 덕에 -proto 파일 없이 호출 가능)

부트스트랩 관리자 REST (성공 기준 ④) — 관리자 계정 AT로:
```powershell
curl.exe -s -X POST -H "Authorization: Bearer $adminToken" -H "Content-Type: application/json" -d '{\"name\":\"dev\",\"description\":\"개발팀\"}' http://localhost:8000/api/org/teams
```
Expected: 201 + 팀 JSON. 일반 계정 AT로 같은 요청 → 403.

- [ ] **Step 3: platform-backend README 작성**

`README.md` — 다음 구조로 작성 (기존 서비스 README 톤 준수):
````markdown
# platform-backend

ALM·Wiki 플랫폼의 횡단 서비스 멀티모듈 repo.

## 모듈

| 모듈 | 역할 | 포트 |
|---|---|---|
| common-proto | gRPC 계약(platform.org.v1) — GitHub Packages 발행 (`com.platform:common-proto`) | — |
| org-service | 조직·팀·RBAC. REST(게이트웨이 경유 /api/org/**) + gRPC PermissionService | 9130 / 9131 |

## 빌드·실행

```powershell
.\gradlew.bat build                  # 전 모듈 빌드+테스트
.\gradlew.bat :org-service:bootRun   # dev 모드 (Eureka 등록)
```

배포판은 infra/keycloak compose가 담당 — docker 프로필(Eureka 미등록, DNS 직결).

## proto 발행

`v*` 태그 push 시 CI가 태그 버전으로 GitHub Packages 발행.
소비: `implementation 'com.platform:common-proto:<버전>'` + GitHub Packages 저장소 인증.

## 권한 모델

grant_entry 단일 원장 — subject(USER|TEAM) × resource(GLOBAL|SPACE|PROJECT) × role(VIEWER<EDITOR<ADMIN).
판정: 직접+팀 grant 병합 최고 role, GLOBAL은 전 리소스 적용. 최초 관리자는 PLATFORM_BOOTSTRAP_ADMIN_ID 시드.
````

- [ ] **Step 4: 우산 README 갱신**

`C:\MSA_TEMPLATE\README.md`:
- 토폴로지 그림의 gateway 분기에 `└──▶ lb://org-service(:9130)` 추가, infra 줄에 orgdb 언급
- 컴포넌트 표에 행 추가:
```markdown
| **org-service** | 9130·9131 | 조직·팀·RBAC. REST /api/org/** + gRPC PermissionService | [chanho4702/platform-backend](https://github.com/chanho4702/platform-backend) |
```

- [ ] **Step 5: 커밋 + 결과 기록**

```powershell
git -C C:\MSA_TEMPLATE\platform-backend add README.md
git -C C:\MSA_TEMPLATE\platform-backend commit -m "docs: platform-backend README"
git -C C:\MSA_TEMPLATE\platform-backend push
git -C C:\MSA_TEMPLATE add README.md
git -C C:\MSA_TEMPLATE commit -m "docs: org-service 추가 반영 — 토폴로지·컴포넌트 표"
```
E2E 결과(성공 기준 ①~⑤ 각각 pass/fail)를 사용자에게 보고. 완료 시 Obsidian 15번 문서 상태를 "Wave A 완료"로 갱신.

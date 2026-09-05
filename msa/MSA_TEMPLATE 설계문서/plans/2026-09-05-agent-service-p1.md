# agent-service P1 — 기록 계층 + MCP 구현 플랜

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 에이전트(외부 Claude Code)가 MCP로 우리 ALM·Wiki에 페르소나 명의로 이슈를 집고, 기록하고, 문서를 쓰게 하는 기록 계층을 만든다 — 새 화면 없음.

**Architecture:** 신규 `agent-service`(Spring Boot 4.0.6 + Spring AI 2.0 MCP 서버, :9160/:19160)가 도구 계층을 제공하고, 다운스트림 alm/wiki/org REST를 **페르소나 토큰**(auth-server가 발급, sub=페르소나 멤버 id)으로 호출한다. 페르소나는 auth-server `users`(keycloak_sub=`agent:<slug>`) + org-service `member(kind=AGENT)` + agentdb `persona`의 3중 등록. 외부 CLI는 PAT(페르소나 바인딩)로 MCP에 접속한다.

**Tech Stack:** Java 24, Spring Boot 4.0.6, Spring Cloud 2025.1.2, Spring AI 2.0.0(MCP server webmvc, streamable-http), common-starter 0.15.0, PostgreSQL(`agentdb`) + Flyway, Caffeine.

**Spec:** `C:\myBrain\내 로컬\msa\MSA_TEMPLATE 설계문서\specs\2026-09-05-agent-service-design.md` (P1 = 스펙 §10 1행)

## Global Constraints

- 전 서비스 고정 버전: Boot **4.0.6**, dependency-management **1.1.7**, Java toolchain **24**, Spring Cloud **2025.1.2**, Lombok **1.18.46**, common-proto/common-starter **0.15.0**, group `com.platform`, version `0.0.1-SNAPSHOT`, `bootJar → app.jar`.
- 포트: agent-service REST **9160** prod / **19160** dev(+10000). gRPC 없음(P1).
- JWT 계약: issuer `http://localhost:9000`, audience `platform-api`, **sub는 숫자 멤버 id 필수**(모든 컨트롤러가 `Long.parseLong(sub)`), `roles`→`ROLE_*`·오류계약 `{"error":"메시지"}`·공용예외는 common-starter가 제공 — **JwtDecoder/Converter/AudienceValidator를 로컬 재선언 금지**.
- `ddl-auto: validate` + Flyway만 스키마 소유. 마이그레이션 명명 `V{n}__snake_case.sql`.
- 로깅: 컨테이너에서 stdout ECS JSON(`logging.structured.format.console: ecs`, docker 프로필). logback 파일 만들지 않는다. loki4j 직결 금지.
- 게이트웨이 라우트는 서비스가 `/api/agent` 접두사 소유(StripPrefix 없음). 게이트웨이 permitAll 경로는 서비스 SecurityConfig와 반드시 동기.
- `.ps1`은 UTF-8 **BOM**. 브랜치 만들지 않고 각 리포 main 직접 커밋(main 푸시=배포 트리거 — 게이트 통과 필수). Fable 세션은 Codex 교차리뷰 생략, 자체 검증.
- 작업기록 문서는 Obsidian에만. 리포에는 CLAUDE.md/AGENTS.md만.
- 신규 리포 visibility는 **private**(추후 공개 결정 별도), LICENSE MIT.

**리포별 커밋 대상:** agent-service(신규) / auth-server / platform-backend(org-service) / gateway-server / 루트(=infra-settings). proto 변경 없음 → 태그 발행 없음.

---

### Task 1: agent-service 리포 골격

**Files:**
- Create: `C:\MSA_TEMPLATE\agent-service\` 전체 골격(아래 단계)
- 원본: `C:\MSA_TEMPLATE\board-service\` (gradlew, .gitignore, .run, 구조)

**Interfaces:**
- Produces: 패키지 루트 `com.platform.agentservice`, 프로필 test(H2·Flyway off·Eureka off), `TestAuth.user(id,name)`/`admin(id,name)` — 이후 모든 태스크의 테스트가 사용.

- [ ] **Step 1: 골격 복사 + 신규 파일**

```powershell
New-Item -ItemType Directory -Force C:\MSA_TEMPLATE\agent-service
Copy-Item C:\MSA_TEMPLATE\board-service\gradlew,C:\MSA_TEMPLATE\board-service\gradlew.bat C:\MSA_TEMPLATE\agent-service\
Copy-Item -Recurse C:\MSA_TEMPLATE\board-service\gradle C:\MSA_TEMPLATE\agent-service\gradle
Copy-Item C:\MSA_TEMPLATE\board-service\.gitignore C:\MSA_TEMPLATE\agent-service\.gitignore
Copy-Item C:\MSA_TEMPLATE\board-service\LICENSE C:\MSA_TEMPLATE\agent-service\LICENSE
New-Item -ItemType Directory -Force C:\MSA_TEMPLATE\agent-service\.run
Copy-Item "C:\MSA_TEMPLATE\board-service\.run\bootRun.run.xml","C:\MSA_TEMPLATE\board-service\.run\bootRun (dev).run.xml" C:\MSA_TEMPLATE\agent-service\.run\
```
`.run\*.xml` 안의 `board-service` 문자열을 `agent-service`로 치환.

`settings.gradle`: `rootProject.name = 'agent-service'`

`build.gradle` — 탐색 리포트의 board-service 사본 그대로, 단 이름/포트만 교체. dependencies 블록(P1 최종형 — Redis는 P2까지 제외):

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.0.6'
    id 'io.spring.dependency-management' version '1.1.7'
}
group = 'com.platform'
version = '0.0.1-SNAPSHOT'
java { toolchain { languageVersion = JavaLanguageVersion.of(24) } }
tasks.withType(JavaCompile).configureEach { options.compilerArgs << '-parameters' }

repositories {
    if (providers.gradleProperty('useMavenLocal').isPresent()) { mavenLocal() }
    mavenCentral()
    maven {
        url = uri('https://maven.pkg.github.com/chanho4702/platform-backend')
        credentials {
            username = System.getenv('GITHUB_ACTOR') ?: 'chanho4702'
            password = System.getenv('GITHUB_TOKEN') ?: providers.gradleProperty('gpr.token').getOrElse('')
        }
    }
}

ext { set('springCloudVersion', '2025.1.2') }
ext { set('testcontainersVersion', '1.21.3') }
dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        mavenBom "org.testcontainers:testcontainers-bom:${testcontainersVersion}"
    }
}

def sharedVersion = providers.gradleProperty('commonProtoVersion').getOrElse('0.15.0')

dependencies {
    implementation "com.platform:common-starter:${sharedVersion}"
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-flyway'
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-database-postgresql'
    implementation 'com.github.ben-manes.caffeine:caffeine'
    runtimeOnly 'org.postgresql:postgresql'
    compileOnly 'org.projectlombok:lombok:1.18.46'
    annotationProcessor 'org.projectlombok:lombok:1.18.46'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
    testImplementation 'org.springframework.boot:spring-boot-testcontainers'
    testImplementation 'org.testcontainers:postgresql'
    testImplementation 'org.testcontainers:junit-jupiter'
    testRuntimeOnly 'com.h2database:h2'
}
tasks.named('test') { useJUnitPlatform() }
tasks.named('bootJar') { archiveFileName = 'app.jar' }
```

`src/main/resources/application.yml` / `application-dev.yml` / `src/test/resources/application-test.yml`: **skeleton 탐색 리포트 Part 1/3 §C의 세 블록을 그대로**(9160/19160, `agentdb`, `platform.jwt.*`, docker 프로필 ECS 로깅 포함. `platform.org-grpc`·redis 블록은 제외 — P1 미사용). 추가로 application.yml에:

```yaml
platform:
  agent:
    auth-base-url: ${AUTH_BASE_URL:http://localhost:9000}
    alm-base-url: ${ALM_BASE_URL:http://localhost:9120}
    wiki-base-url: ${WIKI_BASE_URL:http://localhost:9110}
    org-base-url: ${ORG_BASE_URL:http://localhost:9130}
    internal-secret: ${AGENT_INTERNAL_SECRET:}   # 비면 페르소나 토큰 발급 불가(fail-closed)
```
`application-dev.yml`에는 위 4개 URL의 dev 오버라이드(19000/19120/19110/19130).

`Dockerfile`: skeleton 리포트의 것(`EXPOSE 9160`).

`AgentServiceApplication.java`: bare `@SpringBootApplication`.

`config/SecurityConfig.java`: skeleton 리포트 Part 3/3 §B의 copyable 그대로(`anyRequest().authenticated()`).

`TestAuth.java`: skeleton 리포트 Part 3/3(tail) §A의 copyable을 `com.platform.agentservice` 패키지로.

`db/migration/V1__init.sql`: 임시 최소(`SELECT 1;`은 불가 — Flyway 빈 파일 금지이므로 Task 3에서 실제 스키마로 채운다. Task 1에서는 `db/migration` 디렉터리만 만들고 **마이그레이션 파일 없이** 둔다. test 프로필은 flyway off라 무방).

- [ ] **Step 2: 실패 테스트 — contextLoads**

`src/test/java/com/platform/agentservice/AgentServiceApplicationTests.java`:
```java
package com.platform.agentservice;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest
@ActiveProfiles("test")
class AgentServiceApplicationTests {
    @Test
    void contextLoads() {
    }
}
```

- [ ] **Step 3: 빌드 실행**

Run: `cd C:\MSA_TEMPLATE\agent-service; $env:GITHUB_TOKEN=<gpr PAT>; .\gradlew.bat test --no-daemon`
Expected: PASS (common-starter 0.15.0이 GitHub Packages에서 해석되고 컨텍스트 부팅).
실패 시: GH Packages 인증(`gpr.token`) 또는 common-starter autoconfig 조건(`platform.jwt.issuer/audience` 프로퍼티 존재) 확인.

- [ ] **Step 4: CLAUDE.md / AGENTS.md 작성**

`CLAUDE.md`: 서비스 한줄 정의(에이전트 기록 계층·MCP 서버), 포트 9160/19160, 페르소나 신원 체인(auth users `agent:*` → org member kind=AGENT → agentdb persona), PAT 인증, "루트 CLAUDE.md 먼저 따른다" 포인터. `AGENTS.md`: CLAUDE.md 포인터 + 확정 결정 요약(다른 리포와 동일 패턴).

- [ ] **Step 5: git init + 커밋**

```powershell
cd C:\MSA_TEMPLATE\agent-service
git init -b main; git add -A
git commit -m "feat: agent-service 골격 — Boot 4.0.6 + common-starter 0.15.0 (:9160)"
```
(GitHub 리포 생성·push는 Task 13에서.)

---

### Task 2: Spring AI 2.0 MCP 서버 + ping 도구

**Files:**
- Modify: `build.gradle`, `application.yml`
- Create: `src/main/java/com/platform/agentservice/tools/PingTools.java`, `config/McpConfig.java`
- Test: `src/test/java/com/platform/agentservice/tools/McpServerWiringTest.java`

**Interfaces:**
- Produces: `@McpTool` 메서드가 곧 MCP 도구로 노출되는 패턴(이후 태스크 8~10의 도구 등록 방식), MCP HTTP 엔드포인트 `/api/agent/mcp`.

- [ ] **Step 1: 의존성 + Boot 버전 핀**

`build.gradle`에 추가:
```gradle
dependencyManagement {
    imports {
        mavenBom "org.springframework.ai:spring-ai-bom:2.0.0"
        // ... 기존 cloud/testcontainers BOM 유지
    }
}
dependencies {
    implementation 'org.springframework.ai:spring-ai-starter-mcp-server-webmvc'
}
```
**검증(중요):** `.\gradlew.bat dependencies --configuration runtimeClasspath | Select-String "spring-boot-starter:"` — Boot 계열 아티팩트가 **4.0.6**으로 해석되는지 확인. Spring AI 2.0 스타터가 4.1.0을 끌어오면(알려진 이슈 #6465) dependency-management가 4.0.6으로 눌러주는지 본다. `contextLoads`까지 통과해야 채택.
**폴백(순서대로, 하나 되면 멈춤):** (1) Boot 아티팩트만 명시 핀(`dependencyManagement { dependencies { entry } }`), (2) 스타터 대신 MCP Java SDK(`io.modelcontextprotocol.sdk:mcp-spring-webmvc`, spring-ai-bom이 버전 관리)로 서버 수동 와이어링. Boot 4.1 승격은 P1에서 하지 않는다(Cloud 2025.1.2 호환 불확실).

- [ ] **Step 2: 실패 테스트**

```java
package com.platform.agentservice.tools;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest
@ActiveProfiles("test")
class McpServerWiringTest {
    @Autowired PingTools pingTools;

    @Test
    void ping_returns_pong_with_service_name() {
        assertThat(pingTools.ping()).contains("agent-service");
    }
}
```
Run: `.\gradlew.bat test --tests McpServerWiringTest` → Expected: FAIL (PingTools 없음)

- [ ] **Step 3: 구현**

`tools/PingTools.java` (2.0 어노테이션 API — import가 다르면 `org.springaicommunity.mcp.annotation.McpTool` ↔ `org.springframework.ai.mcp.annotation.McpTool` 중 클래스패스에 있는 쪽 사용):
```java
package com.platform.agentservice.tools;

import org.springaicommunity.mcp.annotation.McpTool;
import org.springframework.stereotype.Component;

@Component
public class PingTools {

    @McpTool(name = "ping", description = "연결 확인용. 서비스 이름과 시각을 돌려준다.")
    public String ping() {
        return "pong from agent-service at " + java.time.Instant.now();
    }
}
```

`application.yml`에 MCP 서버 설정:
```yaml
spring:
  ai:
    mcp:
      server:
        name: platform-agent
        version: 0.1.0
        protocol: STREAMABLE          # SSE 아님 — 2.0 기본
        streamable-http:
          mcp-endpoint: /api/agent/mcp
```
프로퍼티 키가 버전과 다르면 `McpServerProperties`(spring-ai-autoconfigure-mcp-server jar) 소스에서 실제 키를 확인해 맞춘다. 도구 등록은 `@Component` + `@McpTool` 스캔 자동(2.0). 자동 스캔이 안 되면 `config/McpConfig.java`에 `MethodToolCallbackProvider.builder().toolObjects(pingTools).build()` 빈으로 수동 등록.

- [ ] **Step 4: 테스트 + 수동 확인**

Run: `.\gradlew.bat test` → PASS.
수동: `.\gradlew.bat bootRun` 후 `curl -s -o NUL -w "%{http_code}" http://localhost:9160/api/agent/mcp` → **401**(Security 통과 전) — 404가 아니면 엔드포인트 자체는 살아있음. (MCP 프로토콜 왕복 검증은 Task 14 E2E에서.)

- [ ] **Step 5: Commit** — `feat: Spring AI 2.0 MCP 서버 + ping 도구 (/api/agent/mcp)`

---

### Task 3: agentdb 스키마 V1 + 도메인 엔티티

**Files:**
- Create: `db/migration/V1__init.sql`, `persona/{Persona,PersonaRole,PersonaRepository}.java`, `pat/{PatToken,PatTokenRepository}.java`, `audit/{ToolCallAudit,ToolCallAuditRepository}.java`
- Test: `schema/FlywaySchemaValidationTest.java`, `persona/PersonaRepositoryTest.java`

**Interfaces:**
- Produces: `Persona{id, memberId, slug, role, name, emoji, voicePrompt, active}`, `PatToken{id, tokenHash, label, ownerMemberId, personaId, expiresAt, lastUsedAt, revokedAt}`, `ToolCallAudit{id, personaId, actorMemberId, tool, summary, status(OK|ERROR), createdAt}` — Task 6~10이 사용.

- [ ] **Step 1: V1__init.sql**

```sql
-- agent-service P1: 페르소나 · PAT · 도구 감사. (Run/Gate/Harness는 P2)
CREATE TABLE persona (
    id            BIGSERIAL PRIMARY KEY,
    member_id     BIGINT       NOT NULL UNIQUE,  -- auth users.id = org member.id
    slug          VARCHAR(40)  NOT NULL UNIQUE,  -- agent:<slug> 의 slug
    role          VARCHAR(20)  NOT NULL,         -- PLANNER|DESIGNER|FRONTEND|BACKEND|OPS|REVIEWER
    name          VARCHAR(80)  NOT NULL,
    emoji         VARCHAR(16),
    voice_prompt  TEXT,
    active        BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at    TIMESTAMP    NOT NULL DEFAULT now(),
    updated_at    TIMESTAMP    NOT NULL DEFAULT now()
);

CREATE TABLE pat_token (
    id               BIGSERIAL PRIMARY KEY,
    token_hash       VARCHAR(64)  NOT NULL UNIQUE,  -- SHA-256 hex
    label            VARCHAR(120) NOT NULL,
    owner_member_id  BIGINT       NOT NULL,          -- 만든 사람(사람 멤버)
    persona_id       BIGINT       NOT NULL REFERENCES persona(id),
    created_at       TIMESTAMP    NOT NULL DEFAULT now(),
    expires_at       TIMESTAMP,
    last_used_at     TIMESTAMP,
    revoked_at       TIMESTAMP
);

CREATE TABLE tool_call_audit (
    id               BIGSERIAL PRIMARY KEY,
    persona_id       BIGINT REFERENCES persona(id),
    actor_member_id  BIGINT NOT NULL,
    tool             VARCHAR(60) NOT NULL,
    summary          VARCHAR(500),
    status           VARCHAR(10) NOT NULL,           -- OK | ERROR
    created_at       TIMESTAMP NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_persona_time ON tool_call_audit (persona_id, created_at DESC);
```

- [ ] **Step 2: 실패 테스트 — Testcontainers Flyway↔엔티티 정합**

`schema/FlywaySchemaValidationTest.java` (wiki/alm의 동명 테스트와 같은 패턴):
```java
package com.platform.agentservice.schema;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest
@Testcontainers
class FlywaySchemaValidationTest {
    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
        r.add("spring.flyway.enabled", () -> "true");
        r.add("spring.jpa.hibernate.ddl-auto", () -> "validate");
        r.add("eureka.client.enabled", () -> "false");
        r.add("platform.jwt.issuer", () -> "http://localhost:9000");
        r.add("platform.jwt.audience", () -> "platform-api");
        r.add("spring.security.oauth2.resourceserver.jwt.jwk-set-uri", () -> "http://localhost:9000/.well-known/jwks.json");
    }

    @Test
    void migrations_match_entities() {
        // 컨텍스트 부팅 = Flyway 실행 + validate 통과
    }
}
```
Run → Expected: FAIL (엔티티 미존재).

- [ ] **Step 3: 엔티티 3종 구현**

Lombok `@Getter`, `@NoArgsConstructor(access = PROTECTED)`, 정적 팩토리. 예 — `persona/Persona.java`:
```java
package com.platform.agentservice.persona;

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.Instant;

@Entity
@Table(name = "persona")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Persona {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false, unique = true) private Long memberId;
    @Column(nullable = false, unique = true) private String slug;
    @Enumerated(EnumType.STRING) @Column(nullable = false) private PersonaRole role;
    @Column(nullable = false) private String name;
    private String emoji;
    @Column(columnDefinition = "text") private String voicePrompt;
    @Column(nullable = false) private boolean active = true;
    @CreationTimestamp @Column(nullable = false, updatable = false) private Instant createdAt;
    @UpdateTimestamp @Column(nullable = false) private Instant updatedAt;

    public static Persona of(Long memberId, String slug, PersonaRole role, String name, String emoji, String voicePrompt) {
        Persona p = new Persona();
        p.memberId = memberId; p.slug = slug; p.role = role; p.name = name;
        p.emoji = emoji; p.voicePrompt = voicePrompt;
        return p;
    }
}
```
`PersonaRole`: `PLANNER, DESIGNER, FRONTEND, BACKEND, OPS, REVIEWER`. `PatToken`·`ToolCallAudit`도 V1 컬럼과 1:1로(타입·nullable 정확히 일치 — validate가 잡는다). Repository는 `JpaRepository` + `Optional<Persona> findBySlug(String)`, `Optional<PatToken> findByTokenHash(String)`.

- [ ] **Step 4: 테스트** — `.\gradlew.bat test` → PASS (Docker 데몬 필요).
- [ ] **Step 5: Commit** — `feat: agentdb V1 — persona·pat_token·tool_call_audit + 스키마 정합 테스트`

---

### Task 4: auth-server — 에이전트 사용자 생성 + 페르소나 토큰 발급

**Files (리포: `C:\MSA_TEMPLATE\auth-server`):**
- Create: `src/main/java/com/platform/authserver/agent/{AgentAdminController,AgentTokenController,InternalSecretFilter,AgentUserService}.java`
- Modify: `config/SecurityConfig.java`(경로 정책), `application.yml`(agent 설정)
- Test: `src/test/java/com/platform/authserver/agent/{AgentAdminControllerTest,AgentTokenControllerTest}.java`

**Interfaces:**
- Produces: `POST /api/auth/agents` `{slug, name, email?}` → 200 `{"userId": <long>, "slug": "...", "created": bool}` (ROLE_ADMIN JWT). `POST /internal/service-tokens` `{userId}` + 헤더 `X-Internal-Secret` → 200 `{"accessToken": "...", "expiresInSeconds": 900}`. Task 6의 `AuthTokenClient`/`PersonaService`가 소비.
- Consumes: 기존 `JwtService.issueAccessToken(userId, email, name, roles, provider)`, `UserRepository`.

- [ ] **Step 1: 실패 테스트 2건**

`AgentAdminControllerTest`(MockMvc + spring-security-test):
```java
@Test void admin_creates_agent_user_idempotently() {
    // POST /api/auth/agents {"slug":"jiho","name":"지호"} with ADMIN jwt → 200, userId > 0, created=true
    // 같은 요청 반복 → 200, 같은 userId, created=false
}
@Test void non_admin_forbidden() { /* USER jwt → 403 */ }
@Test void bad_slug_rejected() { /* "Jiho!" → 400 (패턴 [a-z0-9-]{2,40}) */ }
```
`AgentTokenControllerTest`:
```java
@Test void mints_token_for_agent_user_with_secret() {
    // 시크릿 설정된 컨텍스트에서 agent 유저 생성 후 POST /internal/service-tokens {"userId":X} + X-Internal-Secret
    // → 200, accessToken 디코드하면 sub == String.valueOf(X), aud=platform-api, provider="AGENT"
}
@Test void refuses_human_user() { /* keycloak_sub가 agent:* 아님 → 403 */ }
@Test void refuses_wrong_or_missing_secret() { /* → 403 */ }
@Test void disabled_when_secret_env_empty() { /* platform.agent.internal-secret 미설정 → 403 */ }
```
Run: `.\gradlew.bat test --tests "*Agent*"` → FAIL.

- [ ] **Step 2: 구현**

`AgentUserService`: `createOrGet(slug, name, email)` — `userRepository.findByKeycloakSub("agent:"+slug)` 없으면 `new User("agent:"+slug)` + name/email/`roles="USER"`/`provider="AGENT"` 저장(기존 `User` 세터 사용, `UserRepository`에 `findByKeycloakSub` 쿼리 메서드가 없으면 추가). `mint(userId)` — 유저 조회, `keycloak_sub.startsWith("agent:")` 아니면 `ForbiddenException`, `jwtService.issueAccessToken(id, email, name, roles, "AGENT")`.

`AgentAdminController`: `@PostMapping("/api/auth/agents")` + `@PreAuthorize("hasRole('ADMIN')")`. **주의:** auth-server 리소스서버 체인에 roles→ROLE_ 컨버터가 없으면(공용 스타터 미사용 서비스) `JwtAuthenticationConverter` + `JwtGrantedAuthoritiesConverter(claim "roles", prefix "ROLE_")` 빈을 이 태스크에서 추가하고 `/api/me` 기존 동작 회귀 테스트로 확인.

`InternalSecretFilter`(`OncePerRequestFilter`, `/internal/**`만): `MessageDigest.isEqual(secret, header)` — `platform.agent.internal-secret`(`${AGENT_INTERNAL_SECRET:}`)이 빈 문자열이면 무조건 403. `SecurityConfig`: `/internal/**` permitAll(필터가 인증 담당) + 필터 등록.

`application.yml`: `platform.agent.internal-secret: ${AGENT_INTERNAL_SECRET:}` (+ 주석 "비면 발급 경로 전체 차단").

- [ ] **Step 3: 테스트** → PASS. 기존 전체 테스트도 실행(`.\gradlew.bat test`) — 회귀 없음 확인.
- [ ] **Step 4: Commit(auth-server 리포)** — `feat: 에이전트 페르소나 사용자(/api/auth/agents) + 내부 서비스 토큰 발급(/internal/service-tokens)`
  main 푸시 전 CI 게이트 인지: 푸시=배포 트리거. `AGENT_INTERNAL_SECRET`는 compose에 아직 없으므로 발급 경로는 배포돼도 잠겨 있음(fail-closed) — 안전.

---

### Task 5: org-service — member.kind(AGENT) + 에이전트 멤버 등록

**Files (리포: `C:\MSA_TEMPLATE\platform-backend`):**
- Create: `org-service/src/main/resources/db/migration/V3__member_kind.sql`, `org-service/src/main/java/com/platform/orgservice/member/AgentMemberController.java`
- Modify: `domain/Member.java`, `member/MemberService.java`, `security/MemberMirrorFilter.java`, `member/dto/MemberResponse.java`
- Test: `org-service/src/test/java/com/platform/orgservice/member/AgentMemberControllerTest.java`, MirrorFilter 가드 테스트

**Interfaces:**
- Produces: `POST /api/org/members/agents` `{id, displayName, email?}` → 200 `MemberResponse{id, displayName, email, status, kind}` (ROLE_ADMIN). `GET /api/org/members` 응답에 `kind: "HUMAN"|"AGENT"` 필드 추가(**프론트 비파괴 — 필드 추가만**). Task 6이 소비.

- [ ] **Step 1: V3__member_kind.sql**
```sql
-- 에이전트 페르소나를 멤버로 구분(스펙 D6). 기존 행은 전부 사람.
ALTER TABLE member ADD COLUMN kind VARCHAR(20) NOT NULL DEFAULT 'HUMAN';
```

- [ ] **Step 2: 실패 테스트**
```java
@Test void admin_registers_agent_member_idempotently() {
    // POST /api/org/members/agents {"id":9001,"displayName":"지호"} ADMIN → 200 kind=AGENT
    // 반복 호출 → 200, displayName 갱신, 여전히 AGENT
}
@Test void non_admin_forbidden() { /* 403 */ }
@Test void mirror_filter_skips_agent_rows() {
    // kind=AGENT인 member가 있을 때 같은 id의 JWT로 아무 GET 호출
    // → displayName/email이 mirror로 덮이지 않음
}
@Test void member_list_includes_kind() { /* GET /api/org/members → 각 항목에 kind */ }
```

- [ ] **Step 3: 구현**
`Member`: `@Enumerated(STRING) MemberKind kind` 필드(+`MemberKind{HUMAN, AGENT}`), `Member.of(...)`는 HUMAN 기본, 새 팩토리 `Member.agentOf(id, displayName, email)`. `MemberService`: `registerAgent(id, name, email)` upsert(존재 시 refresh + kind=AGENT 유지), `mirror(...)`는 조회 후 `kind == AGENT`면 refresh 스킵. `AgentMemberController`: `@PreAuthorize("hasRole('ADMIN')")`. `MemberResponse`에 `kind` 추가.

- [ ] **Step 4: 테스트** → PASS + org-service 전체 테스트 회귀 확인.
- [ ] **Step 5: Commit(platform-backend 리포)** — `feat(org-service): member.kind(V3) + 에이전트 멤버 등록 API + mirror AGENT 가드`
  proto 변경 없음 → 공유 아티팩트 발행 불필요.

---

### Task 6: agent-service — 다운스트림 클라이언트 + 페르소나 부트스트랩

**Files:**
- Create: `client/{AuthTokenClient,OrgClient}.java`, `client/TokenService.java`, `persona/{PersonaService,PersonaController}.java`, `persona/dto/{PersonaCreateRequest,PersonaResponse}.java`, `config/ClientsConfig.java`
- Test: `client/TokenServiceTest.java`, `persona/PersonaServiceTest.java`(MockRestServiceServer)

**Interfaces:**
- Consumes: Task 4·5의 엔드포인트, Task 3의 `Persona`.
- Produces:
  - `AuthTokenClient.mint(long memberId) → TokenResponse(String accessToken, long expiresInSeconds)`
  - `TokenService.bearerFor(long personaMemberId) → String` (Caffeine 캐시, 만료 100초 전 갱신)
  - `OrgClient.registerAgentMember(long id, String name, String email, String adminBearer)`
  - `OrgClient.grant(String subjectId, String resourceType, String resourceId, String role, String adminBearer)` — org `POST /api/org/grants` `{subjectType:"USER", subjectId, resourceType, resourceId, role}`
  - `POST /api/agent/personas` (ROLE_ADMIN) `{slug, role, name, emoji?, voicePrompt?, email?, grants?: [{resourceType:"PROJECT"|"SPACE", resourceId, role:"EDITOR"|...}]}` → 201 `PersonaResponse{id, memberId, slug, role, name, emoji, active}`
  - `GET /api/agent/personas` (authenticated) → `List<PersonaResponse>`

- [ ] **Step 1: 실패 테스트** — `PersonaServiceTest`: MockRestServiceServer로 auth `POST /api/auth/agents`(→ userId 9001)·org `POST /api/org/members/agents`·org `POST /api/org/grants` 순서 호출 + agentdb 저장 검증, 멱등(재호출 시 기존 persona 반환). `TokenServiceTest`: mint 1회 후 두 번째 `bearerFor`는 캐시 히트(MockRestServiceServer 호출 1회만).

- [ ] **Step 2: 구현**
`ClientsConfig`: `RestClient` 빈 4개(auth/org/alm/wiki, base-url은 `platform.agent.*`). `AuthTokenClient`: `/api/auth/agents`는 **호출자(관리자)의 Authorization 헤더를 그대로 전달**(`RequestContextHolder`에서 현재 요청 헤더), `/internal/service-tokens`는 `X-Internal-Secret: ${platform.agent.internal-secret}`. `PersonaService.bootstrap(req, adminBearer)`: ① auth agents → userId ② org registerAgentMember ③ grants 반복 ④ `Persona.of(...)` 저장(slug 중복 시 기존 반환 + name 등 refresh). 오류 전파: 다운스트림 4xx/5xx는 `ConflictException`/`ServiceUnavailableException`(common-starter)으로 매핑해 `{"error":...}` 계약 유지.

- [ ] **Step 3: 테스트** → PASS.
- [ ] **Step 4: Commit** — `feat: 페르소나 부트스트랩(auth→org→agentdb) + 페르소나 토큰 캐시`

---

### Task 7: PAT — 발급·검증 필터

**Files:**
- Create: `pat/{PatService,PatController,PatAuthFilter,PatPrincipal}.java`, `pat/dto/{PatCreateRequest,PatCreatedResponse,PatSummaryResponse}.java`
- Modify: `config/SecurityConfig.java`
- Test: `pat/PatServiceTest.java`, `pat/PatAuthFilterTest.java`

**Interfaces:**
- Produces:
  - `POST /api/agent/tokens` (**ROLE_ADMIN**) `{label, personaSlug, expiresInDays?}` → 201 `{token:"agp_<43자 base64url>", id, label, personaSlug}` — **평문은 이 응답 한 번만**
  - `GET /api/agent/tokens` → `[{id,label,personaSlug,createdAt,expiresAt,lastUsedAt,revoked}]`, `DELETE /api/agent/tokens/{id}` → 204(revoked_at 스탬프)
  - `PatAuthFilter`: `/api/agent/mcp/**` 요청의 `Authorization: Bearer agp_*`를 SHA-256 조회 → 유효하면 `PatPrincipal(ownerMemberId, personaId, personaMemberId)`로 SecurityContext 채움. 만료·철회·미존재 → 401 `{"error":"유효하지 않은 토큰"}`
  - Task 8~10 도구가 `ToolActor.current()`(SecurityContext에서 PatPrincipal 꺼내는 헬퍼)로 소비.

- [ ] **Step 1: 실패 테스트** — 발급 후 해시만 저장(평문 재조회 불가), 검증 성공/만료/철회 401, `lastUsedAt` 갱신, 필터가 `/api/agent/tokens`에는 관여하지 않음(JWT 경로).
- [ ] **Step 2: 구현** — 토큰 생성 `agp_` + `SecureRandom` 32바이트 base64url. `SecurityConfig` 최종형:
```java
http.sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
    .csrf(csrf -> csrf.disable())
    .authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/agent/mcp/**").permitAll()   // PatAuthFilter가 인증(게이트웨이 permitAll과 동기 — Task 11)
        .anyRequest().authenticated())
    .oauth2ResourceServer(o -> o.jwt(jwt -> jwt.jwtAuthenticationConverter(converter)))
    .addFilterBefore(patAuthFilter, BearerTokenAuthenticationFilter.class);
```
`permitAll`이지만 필터가 401을 직접 반환하므로 실질 인증 필수. (주의: permitAll 상태에서 필터 미통과 요청이 컨트롤러/MCP 핸들러에 닿지 않도록 필터에서 **chain 진행 전 차단**을 테스트로 못박는다.)
- [ ] **Step 3: 테스트** → PASS. **Step 4: Commit** — `feat: PAT 발급/철회 + /api/agent/mcp PAT 인증 필터`

---

### Task 8: ALM 클라이언트 + 이슈 도구

**Files:**
- Create: `client/AlmClient.java`, `client/dto/`(IssueDto·CommentDto·WorklogDto — alm 응답 shape 그대로), `tools/{IssueTools,ToolActor,Audited}.java`, `audit/AuditService.java`
- Test: `client/AlmClientTest.java`(MockRestServiceServer), `tools/IssueToolsTest.java`

**Interfaces:**
- Consumes: `TokenService.bearerFor`, `PatPrincipal`, Task 3 audit 엔티티.
- Produces(MCP 도구 — 전부 페르소나 토큰으로 다운스트림 호출, 매 호출 audit 기록):

| 도구 | 시그니처(파라미터 → 반환 요약) |
|---|---|
| `search_issues` | `(projectId?, text?, statuses?, assignee?("unassigned"\|memberId), labels?, page?)` → `GET /api/alm/issues/search` 결과 요약 JSON(키·제목·상태·담당·우선순위) |
| `get_issue` | `(issueKey)` → `GET /api/alm/issues/by-key/{key}` 전체 + 코멘트 최근 10 + 워크로그 합계 |
| `create_issue` | `(projectId, title, description?, type?, priority?, assigneeSlug?, labels?, parentKey?)` → 생성된 키. description은 텍스트를 받으면 `<p>` 래핑(TipTap HTML 계약), assigneeSlug는 persona slug→memberId 해석 |
| `claim_issue` | `(issueKey)` → GET 후 `assigneeId=페르소나, status=inprogress`로 PUT(`expectedVersion` 사용, 409 시 1회 재시도) |
| `update_issue_status` | `(issueKey, status)` → GET+PUT full-replace, 스킴 전이 위반 400은 메시지 그대로 전달 |
| `add_comment` | `(issueKey, body)` → `POST .../comments` (VIEW 권한이면 가능) |
| `log_work` | `(issueKey, hours, comment?, workedOn?)` → `POST .../worklogs` (기본 workedOn=오늘) |
| `link_pr` | `(issueKey, url, note?)` → 구조화 코멘트 `🔗 PR 연결: {url}\n{note}` — P1 잠정, P2에서 원격링크 엔티티(V21)+커밋 파서로 승격 |

- [ ] **Step 1: 실패 테스트** — `AlmClientTest`: 각 메서드가 정확한 경로·바디·Authorization 헤더로 호출하는지 MockRestServiceServer로. `IssueToolsTest`: `claim_issue`의 GET→PUT 시퀀스와 409 재시도 1회, `create_issue`의 `<p>` 래핑(이미 `<`로 시작하면 미래핑), audit 행 생성(OK/ERROR).
- [ ] **Step 2: 구현** — `ToolActor.current()`: SecurityContext에서 PatPrincipal 없으면 `ForbiddenException("MCP 인증 필요")`. `Audited.run(tool, summary, supplier)`: try/catch로 audit 기록 후 예외 시 도구 결과에 `{"error": 메시지}` 문자열 반환(MCP 규약상 도구는 예외 대신 오류 텍스트가 낫다). 503(권한 서비스 다운)은 "org-service 다운 — 권한 없음이 아님, 잠시 후 재시도" 메시지로 구분.
- [ ] **Step 3: 테스트** → PASS. **Step 4: Commit** — `feat: ALM 이슈 도구 8종(search/get/create/claim/status/comment/worklog/link_pr) + 감사 로그`

---

### Task 9: Wiki 클라이언트 + 문서 도구

**Files:**
- Create: `client/WikiClient.java`, `tools/WikiTools.java`
- Test: `client/WikiClientTest.java`, `tools/WikiToolsTest.java`

**Interfaces:**
- Produces(MCP 도구):

| 도구 | 동작 |
|---|---|
| `list_spaces` | `GET /api/wiki/spaces` → `[{id,key,name}]` (페르소나가 접근 가능한 것만 — 서버가 필터) |
| `get_page` | `(pageId)` → `GET /api/wiki/pages/{id}` → 제목·마크다운 본문·version |
| `find_pages` | `(spaceId, query)` → `GET /api/wiki/spaces/{id}/pages/search?q=` (제목 검색) + `GET .../pages/children`(루트 목록) 병용 |
| `create_page` | `(spaceId, parentId?, title, contentMarkdown, draft?)` → `POST /api/wiki/pages` `{spaceId,parentId,title,content,status}` → pageId. 페르소나가 저자(`created_by=sub`) |
| `update_page` | `(pageId, title?, contentMarkdown, changeNote?)` → GET으로 version·현 제목 확보 후 PUT `{title,content,parentId:현재값,expectedVersion,changeNote}`, 409 시 1회 재시도 |
| `append_to_page` | `(pageId, sectionMarkdown, changeNote?)` → GET 본문 + `\n\n` + 추가분으로 update_page 재사용 — 작업기록 누적용 |

- [ ] **Step 1: 실패 테스트** — 경로·바디·`expectedVersion` 왕복, `append_to_page`가 기존 본문 보존, `update_page`의 `parentId` 현행 유지(400 함정 — re-parent는 별도 move라 P1 미지원 명시).
- [ ] **Step 2: 구현** — Task 8과 동일 패턴(`Audited`, 페르소나 토큰).
- [ ] **Step 3: 테스트** → PASS. **Step 4: Commit** — `feat: Wiki 문서 도구 6종 — 페르소나 저자 기록`

---

### Task 10: 컨텍스트 도구 + 서버 instructions + 도구 스냅샷

**Files:**
- Create: `tools/ContextTools.java`, `src/test/java/com/platform/agentservice/tools/ToolRegistrySnapshotTest.java`
- Modify: `application.yml`(instructions)

**Interfaces:**
- Produces: `whoami`(현재 페르소나 slug·이름·역할·memberId), `list_projects`(`GET /api/alm/projects` 요약), `get_project_context(projectId)` — 프로젝트 정보 + `GET /api/alm/projects/{id}/settings`의 유효 설정 요약(사용 가능 status/type/priority id 목록) + `GET /api/org/members` 명단(kind 포함). **create_issue 전에 이 도구를 먼저 부르라고 instructions에 명시**(스킴 400 예방).

- [ ] **Step 1: 실패 테스트** — `ToolRegistrySnapshotTest`: 컨텍스트에서 MCP 도구 콜백 목록을 꺼내(`SyncMcpToolCallbackProvider` 혹은 서버 빈) 이름 집합이 정확히 `{ping, whoami, list_projects, get_project_context, search_issues, get_issue, create_issue, claim_issue, update_issue_status, add_comment, log_work, link_pr, list_spaces, get_page, find_pages, create_page, update_page, append_to_page}` 18개와 일치함을 단언 — 도구 증발/오타 회귀 방지.
- [ ] **Step 2: 구현 + instructions**
```yaml
spring:
  ai:
    mcp:
      server:
        instructions: >
          플랫폼 ALM·Wiki 기록 도구다. 규약: (1) 작업 시작 전 get_project_context로 스킴·명단 확인,
          (2) 이슈를 집으면 claim_issue, 진행 코멘트는 add_comment, 시간은 log_work,
          (3) 결정·설계는 위키에 create_page/append_to_page로 남기고 이슈에 링크,
          (4) PR은 link_pr로 연결, (5) 완료 시 update_issue_status(done 계열)와 마무리 코멘트.
          모든 기록은 현재 페르소나 명의로 남는다.
```
- [ ] **Step 3: 테스트** → PASS. **Step 4: Commit** — `feat: 컨텍스트 도구 + MCP instructions + 도구 스냅샷 테스트`

---

### Task 11: gateway-server — agent 라우트 + MCP permitAll

**Files (리포: `C:\MSA_TEMPLATE\gateway-server`):**
- Modify: `src/main/resources/application.yml`(라우트), `src/main/java/com/platform/gateway/config/SecurityConfig.java`(permitAll)
- Test: 기존 라우팅/보안 테스트 스위트에 케이스 추가(있는 파일 패턴 따름)

- [ ] **Step 1:** 라우트 추가(skeleton 리포트 §A 스니펫 그대로 — `id: agent`, `Path=/api/agent/**`, `${AGENT_SERVICE_URI:lb://agent-service}`).
- [ ] **Step 2:** `SecurityConfig`에 `.pathMatchers("/api/agent/mcp/**").permitAll()` 추가 + 주석 "PAT 인증은 agent-service 자체 수행(서비스 SecurityConfig와 동기)". `/api/agent/tokens`·`/api/agent/personas`는 JWT 필수(기본 authenticated 유지).
- [ ] **Step 3:** 테스트/빌드 → PASS. **Step 4: Commit(gateway-server 리포)** — `feat: /api/agent/** 라우트 + MCP 경로 PAT 위임(permitAll)`

---

### Task 12: infra-settings — compose·DB·배포·스모크·dev 스크립트·문서

**Files (리포: `C:\MSA_TEMPLATE` 루트):**
- Modify: `infra/keycloak/docker-compose.yml`, `infra/keycloak/init-authdb.sql`, `.github/workflows/deploy.yml`, `scripts/smoke-stack.ps1`, `scripts/dev-up-local.ps1`, `infra/README.md`, `.env.example`, `CLAUDE.md`, `AGENTS.md`

- [ ] **Step 1: compose** — skeleton 리포트 §B의 `agent-service` 블록 + `agent-db-init`(원샷 \gexec 생성기) 그대로 추가, `agent-service`의 `depends_on`에 `agent-db-init: { condition: service_completed_successfully }` 포함, gateway env에 `AGENT_SERVICE_URI: http://agent-service:9160`. **auth-server env에 `AGENT_INTERNAL_SECRET: ${AGENT_INTERNAL_SECRET:-}` 추가, agent-service env에도 동일 주입**(같은 값 공유; `.env`/`C:\deploy\platform.env`에 실값 — Step 6에서 문서화). agent-service의 docker 프로필은 `eureka.client.enabled: false` 방식(org/search 패턴)이므로 `SPRING_PROFILES_ACTIVE: docker`로 충분한지 agent-service `application.yml`에 docker 프로필 문서 추가(Task 1의 yml에 이미 반영됐는지 검증).
- [ ] **Step 2: init-authdb.sql** — `CREATE DATABASE agentdb;` 추가(빈 볼륨 신규 설치용).
- [ ] **Step 3: deploy.yml 4개 편집** — inputs description·`$valid`·`fromJSON` 배열·`$svcMap('agent-service'='agent-service')`/`$tagMap('agent-service'='AGENT_TAG')`.
- [ ] **Step 4: smoke-stack.ps1** — `$mustBeHealthChecked`에 agent-service, gateway `AGENT_SERVICE_URI` assert, no-host-ports assert 목록에 agent-service. (UTF-8 BOM 유지 확인.)
- [ ] **Step 5: dev-up-local.ps1** — `$backends`에 `'agent-service'`, dev 포트 안내에 `agent 19160`, 기동 순서 문구에 agent-service(org 다음). BOM 유지.
- [ ] **Step 6: 문서** — `infra/README.md` 컴포넌트/포트 표 + GHCR↔TAG 표에 agent-service. `.env.example`에 `AGENT_INTERNAL_SECRET=`(주석: 비면 페르소나 토큰 발급이 잠긴다), `AGENT_TAG=`. 루트 `CLAUDE.md`: 서비스 목록에 agent-service 추가, **공유 아티팩트 버전 표기 0.14.0 → 0.15.0 정정**, 변경 이력 행 추가(P1 도입). `AGENTS.md` 동기 갱신.
- [ ] **Step 7: 검증** — `.\scripts\smoke-stack.ps1 -ConfigOnly` PASS.
- [ ] **Step 8: Commit(루트 리포)** — `ops: agent-service 편입 — compose·agentdb·deploy·smoke·dev 스크립트·문서 (proto 0.15.0 표기 정정 포함)`
  **주의:** 기존 배포 볼륨에는 init-authdb가 안 돌므로 첫 롤아웃 전에 러너 호스트에서 `docker compose up -d agent-db-init` 1회 수동 실행. 배포 순서: agent-service 먼저, gateway 마지막.

---

### Task 13: CI + GitHub 리포/시크릿

**Files (리포: agent-service):**
- Create: `.github/workflows/ci.yml` — skeleton 리포트 §D 그대로(이미지 `ghcr.io/chanho4702/agent-service`, `GH_PACKAGES_TOKEN`으로 gradle 빌드, main에서 GHCR push + infra-settings dispatch). alm처럼 dispatch를 `vars.DEPLOY_ENABLED == 'true'` 게이트로 감싼다(인프라 머지 전 안전판).

- [ ] **Step 1:** ci.yml 작성 + 커밋.
- [ ] **Step 2:** 리포 생성·푸시: `gh repo create chanho4702/agent-service --private --source . --push`
- [ ] **Step 3(사용자 액션 — 자동화 불가, 완료 보고에 명시):** agent-service 리포에 시크릿 `GH_PACKAGES_TOKEN`(read:packages PAT)·`DISPATCH_PAT` 등록, variables `DEPLOY_ENABLED=true`(인프라 커밋 후). 러너 `C:\deploy\platform.env`에 `AGENT_INTERNAL_SECRET=<랜덤 32+자>` 추가.
- [ ] **Step 4:** CI 그린 확인(`gh run watch`).

---

### Task 14: E2E 도그푸딩 스모크 + 사용 가이드

**Files:**
- Modify: `agent-service/CLAUDE.md`(MCP 접속 가이드 절)
- 산출: dev 클러스터 실기동 검증 + 첫 페르소나·PAT·이슈

- [ ] **Step 1: dev 클러스터 기동** — 도커 인프라(postgres·keycloak) + IntelliJ `bootRun (dev)`: eureka → auth(`AGENT_INTERNAL_SECRET` env 추가한 dev 실행) → org → alm → wiki → agent → gateway. agentdb는 로컬 postgres(5433)에 1회 생성: `psql -h localhost -p 5433 -U keycloak -d keycloak -c "CREATE DATABASE agentdb"`.
- [ ] **Step 2: 부트스트랩(관리자 JWT로)** — 브라우저 로그인(admin)에서 AT 획득 후:
  1. `POST :18000/api/agent/personas` `{slug:"jiho", role:"BACKEND", name:"지호", emoji:"🔧", grants:[{resourceType:"PROJECT", resourceId:"<프로젝트id>", role:"EDITOR"},{resourceType:"SPACE", resourceId:"<스페이스id>", role:"EDITOR"}]}` → 201
  2. `POST :18000/api/agent/tokens` `{label:"chkim-claude", personaSlug:"jiho"}` → `agp_*` 확보
- [ ] **Step 3: Claude Code 연결**
```
claude mcp add --transport http agent-platform http://localhost:18000/api/agent/mcp --header "Authorization: Bearer agp_..."
```
Claude Code 세션에서 확인: `ping` → pong, `whoami` → 지호, `list_projects` → 목록.
- [ ] **Step 4: 왕복 검증(수용 기준)** — MCP 도구만으로: 이슈 생성(`create_issue`) → `claim_issue` → `add_comment` → `log_work(0.5h)` → 위키 `create_page`(작업 문서) → `append_to_page` → `link_pr`(더미 URL) → `update_issue_status(done)`. 각 단계 후 alm-front(:5175)·wiki-front(:5174)에서 **지호 명의로** 보이는지 확인. `tool_call_audit`에 행 적재 확인.
- [ ] **Step 5: 가이드 작성** — `agent-service/CLAUDE.md`에 접속 절차(위 명령)·도구 규약(instructions 요약)·페르소나 추가법. 커밋.
- [ ] **Step 6: 도그푸딩 개시** — ALM에 `agent-platform` 프로젝트(키 AGP) 생성, P2~P4 + 남은 백로그(`msa-remaining-work`)를 이슈로 이관. 이후 이 플랜의 후속 작업 관리는 우리 ALM에서.

---

## Self-Review 결과

- 스펙 P1 항목 커버: 골격(T1) · 도구 계층(T8·9·10) · 페르소나 org kind=AGENT(T5) · 이슈·위키 기록(T8·9) · 외부 토큰(T7) · MCP(T2) ✓. 스펙의 `list_ready_issues`는 `search_issues`로, `read_page/search/write_page`는 `get_page/find_pages/create_page`로 명명 정리(스펙 §5 갱신 필요 없음 — 스펙은 대표 이름). `get_harness_bundle`·`report_progress`류 run 도구는 스펙 단계표대로 P2.
- 타입 일관성: `TokenService.bearerFor(long)`(T6 정의, T8·9 소비), `PatPrincipal`(T7 정의, T8 `ToolActor` 소비), org/auth 엔드포인트 shape(T4·5 정의, T6 소비) 일치 확인.
- 플레이스홀더: Task 2의 어노테이션 패키지·프로퍼티 키 확인 지점은 "실행 시 검증" 성격으로 명시(2.0 세부가 환경에서 갈릴 수 있는 유일한 지점, 폴백 2단계 명문화).

# Eureka 서비스 디스커버리 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** eureka-server 신설 + auth/board 자기등록 + 게이트웨이 라우트 6개를 lb:// 클라이언트 사이드 로드밸런싱으로 전환한다.

**Architecture:** 유레카 서버는 기존 서비스와 동일 패턴의 독립 Gradle 프로젝트(독립 git repo, 포트 8761, standalone 단일 노드). auth-server·board-service는 eureka-client로 자기 등록하고, 게이트웨이는 라우트 uri 기본값을 `lb://<서비스명>`으로 바꿔 유레카 레지스트리 기반으로 해석한다. 환경변수(`BOARD_SERVICE_URI`/`AUTH_SERVER_URI`)는 테스트 스텁 주입 시임 + 유레카 없는 환경의 탈출구로 유지.

**Tech Stack:** Spring Boot 4.0.6, Spring Cloud 2025.1.2 (spring-cloud-starter-netflix-eureka-server/client), Java 24, Gradle 9.5.1

**Spec:** `docs/superpowers/specs/2026-07-05-eureka-service-discovery-design.md`

## Global Constraints

- 모든 프로젝트: Boot `4.0.6` / dependency-management `1.1.7` / `JavaLanguageVersion.of(24)` / Spring Cloud BOM `2025.1.2` — 기존 파일과 자릿수까지 동일하게
- 멀티 repo: eureka-server·auth-server·board-service·gateway-server 각각 **자기 repo에서** 커밋. 루트(C:\MSA_TEMPLATE) repo는 .gitignore·README만
- 유레카 등록 이름은 이미 있는 `spring.application.name`(`auth-server`, `board-service`)을 그대로 사용 — 변경 금지 (게이트웨이 lb:// 이름과 일치 계약)
- 모든 단위 테스트는 **유레카 서버 없이** green이어야 함 (테스트에서 `eureka.client.enabled: false`)
- 커밋 메시지는 기존 스타일(한국어, `feat(discovery): …`)을 따르고 끝에 `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>` 추가

---

### Task 1: eureka-server 프로젝트 신설

**Files:**
- Create: `C:\MSA_TEMPLATE\eureka-server\build.gradle`
- Create: `C:\MSA_TEMPLATE\eureka-server\settings.gradle`
- Create: `C:\MSA_TEMPLATE\eureka-server\.gitignore`
- Create: `C:\MSA_TEMPLATE\eureka-server\src\main\java\com\platform\eureka\EurekaServerApplication.java`
- Create: `C:\MSA_TEMPLATE\eureka-server\src\main\resources\application.yml`
- Test: `C:\MSA_TEMPLATE\eureka-server\src\test\java\com\platform\eureka\EurekaServerApplicationTest.java`
- Copy: gateway-server의 gradle wrapper 4종 → eureka-server
- Modify: `C:\MSA_TEMPLATE\.gitignore` (루트 repo — `/eureka-server/` 추가)

**Interfaces:**
- Consumes: 없음 (신규 프로젝트)
- Produces: `http://localhost:8761/eureka` — Task 2·3·4의 `eureka.client.service-url.defaultZone`이 가리키는 레지스트리 엔드포인트

- [ ] **Step 1: 프로젝트 뼈대 + gradle wrapper 복사**

```powershell
New-Item -ItemType Directory -Force C:\MSA_TEMPLATE\eureka-server\src\main\java\com\platform\eureka
New-Item -ItemType Directory -Force C:\MSA_TEMPLATE\eureka-server\src\main\resources
New-Item -ItemType Directory -Force C:\MSA_TEMPLATE\eureka-server\src\test\java\com\platform\eureka
Copy-Item C:\MSA_TEMPLATE\gateway-server\gradlew, C:\MSA_TEMPLATE\gateway-server\gradlew.bat C:\MSA_TEMPLATE\eureka-server\
Copy-Item C:\MSA_TEMPLATE\gateway-server\gradle C:\MSA_TEMPLATE\eureka-server\gradle -Recurse
```

- [ ] **Step 2: build.gradle / settings.gradle / .gitignore 작성**

`build.gradle`:
```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.0.6'
    id 'io.spring.dependency-management' version '1.1.7'
}
group = 'com.platform'
version = '0.0.1-SNAPSHOT'
java { toolchain { languageVersion = JavaLanguageVersion.of(24) } }
repositories { mavenCentral() }

ext {
    set('springCloudVersion', '2025.1.2')
}
dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}
dependencies {
    // 서비스 레지스트리 본체 (대시보드 UI 포함, spring-boot-starter-web 전이 포함)
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-server'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
tasks.named('test') { useJUnitPlatform() }
```

`settings.gradle`:
```gradle
rootProject.name = 'eureka-server'
```

`.gitignore`:
```
.gradle/
build/
.idea/
*.iml
```

- [ ] **Step 3: 실패하는 테스트 작성** — 컨텍스트 기동 + 대시보드 200

`src\test\java\com\platform\eureka\EurekaServerApplicationTest.java`:
```java
package com.platform.eureka;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

import static org.assertj.core.api.Assertions.assertThat;

/** standalone 유레카 서버가 뜨고 대시보드(/)가 응답하는지 — 레지스트리로서의 최소 계약. */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class EurekaServerApplicationTest {

    @LocalServerPort
    int port;

    @Test
    void dashboardIsUp() throws Exception {
        HttpResponse<String> res = HttpClient.newHttpClient().send(
                HttpRequest.newBuilder(URI.create("http://localhost:" + port + "/")).GET().build(),
                HttpResponse.BodyHandlers.ofString());
        assertThat(res.statusCode()).isEqualTo(200);
    }
}
```

- [ ] **Step 4: 테스트 실패 확인**

Run: `cd C:\MSA_TEMPLATE\eureka-server; .\gradlew.bat test`
Expected: FAIL — `EurekaServerApplication` 클래스 없음(컴파일 에러) 또는 컨텍스트 기동 실패

- [ ] **Step 5: 메인 클래스 + application.yml 구현**

`src\main\java\com\platform\eureka\EurekaServerApplication.java`:
```java
package com.platform.eureka;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

`src\main\resources\application.yml`:
```yaml
server:
  port: 8761
spring:
  application:
    name: eureka-server
eureka:
  client:
    # standalone 단일 노드 — 자기 자신을 등록할 레지스트리도, 조회할 peer도 없다.
    # HA(peer 복제)는 운영 전환 시점에 재검토 (YAGNI).
    register-with-eureka: false
    fetch-registry: false
```

- [ ] **Step 6: 테스트 통과 확인**

Run: `cd C:\MSA_TEMPLATE\eureka-server; .\gradlew.bat test`
Expected: PASS (BUILD SUCCESSFUL)

- [ ] **Step 7: git init + 커밋 (eureka-server repo)**

```powershell
cd C:\MSA_TEMPLATE\eureka-server
git init
git add -A
git commit -m @'
feat(discovery): standalone Eureka 서버 신설 (:8761)

Boot 4.0.6 + Spring Cloud 2025.1.2. register-with-eureka/fetch-registry false
(단일 노드 — HA는 YAGNI). 대시보드 200 컨텍스트 테스트 포함.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
'@
```

- [ ] **Step 8: 루트 repo .gitignore에 신규 repo 제외 추가 + 커밋**

`C:\MSA_TEMPLATE\.gitignore`의 `# gateway-server는 별도 repo` 블록 아래에 추가:
```
# eureka-server는 별도 repo
/eureka-server/
```

```powershell
cd C:\MSA_TEMPLATE
git add .gitignore
git commit -m @'
chore: eureka-server 별도 repo 제외 (.gitignore)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
'@
```

---

### Task 2: board-service 유레카 자기등록

**Files:**
- Modify: `C:\MSA_TEMPLATE\board-service\build.gradle` (SC BOM + eureka-client)
- Modify: `C:\MSA_TEMPLATE\board-service\src\main\resources\application.yml` (defaultZone)
- Modify: `C:\MSA_TEMPLATE\board-service\src\test\resources\application-test.yml` (eureka 비활성)

**Interfaces:**
- Consumes: Task 1의 레지스트리 `http://localhost:8761/eureka`
- Produces: 유레카 등록 ID `BOARD-SERVICE` (기존 `spring.application.name: board-service` 그대로) — Task 4의 `lb://board-service`가 이 이름으로 해석

- [ ] **Step 1: build.gradle에 Spring Cloud BOM + eureka-client 추가**

`repositories { mavenCentral() }` 줄 아래에 삽입 (gateway-server build.gradle과 동일 패턴):
```gradle
ext {
    set('springCloudVersion', '2025.1.2')
}
dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}
```

`dependencies { … }` 블록 안 첫 줄에 추가:
```gradle
    // 유레카 자기등록 — 게이트웨이 lb://board-service가 이 등록을 바라본다
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
```

- [ ] **Step 2: application.yml에 레지스트리 주소 추가**

`spring:` 블록과 같은 최상위 레벨에 (파일 끝 `platform:` 블록 위) 추가:
```yaml
eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URI:http://localhost:8761/eureka}
```

- [ ] **Step 3: 테스트 프로파일에서 유레카 비활성**

`src\test\resources\application-test.yml`의 `spring:` 블록과 같은 최상위 레벨에 추가:
```yaml
# 단위 테스트는 유레카 서버 없이 돈다 — 등록 시도/에러 로그 차단
eureka:
  client:
    enabled: false
```

- [ ] **Step 4: 전체 테스트 green + 유레카 로그 없음 확인**

Run: `cd C:\MSA_TEMPLATE\board-service; .\gradlew.bat test`
Expected: PASS (기존 테스트 전부 green, `Connect to http://localhost:8761 failed` 류 에러 로그 없어야 함)

- [ ] **Step 5: 커밋 (board-service repo)**

```powershell
cd C:\MSA_TEMPLATE\board-service
git add build.gradle src/main/resources/application.yml src/test/resources/application-test.yml
git commit -m @'
feat(discovery): Eureka 자기등록 (board-service)

SC BOM 2025.1.2 + eureka-client. 등록 이름은 기존 spring.application.name
(board-service) — 게이트웨이 lb://board-service 계약. 테스트는 eureka 비활성.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
'@
```

---

### Task 3: auth-server 유레카 자기등록

**Files:**
- Modify: `C:\MSA_TEMPLATE\auth-server\build.gradle` (SC BOM + eureka-client)
- Modify: `C:\MSA_TEMPLATE\auth-server\src\main\resources\application.yml` (defaultZone)
- Modify: `C:\MSA_TEMPLATE\auth-server\src\test\resources\application-test.yml` (eureka 비활성)

**Interfaces:**
- Consumes: Task 1의 레지스트리 `http://localhost:8761/eureka`
- Produces: 유레카 등록 ID `AUTH-SERVER` (기존 `spring.application.name: auth-server` 그대로) — Task 4의 `lb://auth-server`가 이 이름으로 해석

- [ ] **Step 1: build.gradle에 Spring Cloud BOM + eureka-client 추가**

`repositories { mavenCentral() }` 줄 아래에 삽입:
```gradle
ext {
    set('springCloudVersion', '2025.1.2')
}
dependencyManagement {
    imports {
        mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
    }
}
```

`dependencies { … }` 블록 안 첫 줄에 추가:
```gradle
    // 유레카 자기등록 — 게이트웨이 lb://auth-server가 이 등록을 바라본다
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
```

- [ ] **Step 2: application.yml에 레지스트리 주소 추가**

`spring:` 블록과 같은 최상위 레벨에 추가:
```yaml
eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URI:http://localhost:8761/eureka}
```

- [ ] **Step 3: 테스트 프로파일에서 유레카 비활성**

`src\test\resources\application-test.yml` 최상위 레벨에 추가:
```yaml
# 단위 테스트는 유레카 서버 없이 돈다 — 등록 시도/에러 로그 차단
eureka:
  client:
    enabled: false
```

- [ ] **Step 4: 전체 테스트 green 확인**

Run: `cd C:\MSA_TEMPLATE\auth-server; .\gradlew.bat test`
Expected: PASS (기존 테스트 전부 green)

- [ ] **Step 5: 커밋 (auth-server repo)**

```powershell
cd C:\MSA_TEMPLATE\auth-server
git add build.gradle src/main/resources/application.yml src/test/resources/application-test.yml
git commit -m @'
feat(discovery): Eureka 자기등록 (auth-server)

SC BOM 2025.1.2 + eureka-client. 등록 이름은 기존 spring.application.name
(auth-server) — 게이트웨이 lb://auth-server 계약. 테스트는 eureka 비활성.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
'@
```

---

### Task 4: gateway-server 라우트 lb:// 전환

**Files:**
- Modify: `C:\MSA_TEMPLATE\gateway-server\build.gradle` (eureka-client — BOM은 이미 있음)
- Modify: `C:\MSA_TEMPLATE\gateway-server\src\main\resources\application.yml` (uri 기본값 6개)
- Create: `C:\MSA_TEMPLATE\gateway-server\src\test\resources\application.properties` (eureka 비활성)
- Test: `C:\MSA_TEMPLATE\gateway-server\src\test\java\com\platform\gateway\RouteConfigTest.java` (테스트 추가)

**Interfaces:**
- Consumes: Task 2·3이 등록하는 이름 `board-service` / `auth-server`
- Produces: 최종 라우팅 — 외부 계약(경로·필터) 변화 없음

- [ ] **Step 1: 실패하는 테스트 작성** — 라우트 기본 uri가 lb://인지

`RouteConfigTest.java`에 테스트 메서드 추가 (기존 2개 테스트 아래):
```java
    @Test
    void routesResolveViaServiceDiscoveryByDefault() {
        // env 미주입 시 기본값이 lb:// — 유레카 레지스트리에서 인스턴스를 찾는다.
        // (BOARD_SERVICE_URI/AUTH_SERVER_URI env는 테스트 스텁 주입·유레카 없는 환경용 시임으로 유지)
        List<org.springframework.cloud.gateway.route.Route> routes =
                routeLocator.getRoutes().collectList().block();
        var board = routes.stream().filter(r -> r.getId().equals("board")).findFirst().orElseThrow();
        assertThat(board.getUri()).hasScheme("lb").hasHost("board-service");
        for (String id : List.of("auth-oauth2", "auth-login", "auth-api", "auth-jwks", "auth-me")) {
            var route = routes.stream().filter(r -> r.getId().equals(id)).findFirst().orElseThrow();
            assertThat(route.getUri()).as("route %s", id).hasScheme("lb").hasHost("auth-server");
        }
    }
```

- [ ] **Step 2: 테스트 실패 확인**

Run: `cd C:\MSA_TEMPLATE\gateway-server; .\gradlew.bat test --tests com.platform.gateway.RouteConfigTest`
Expected: FAIL — `routesResolveViaServiceDiscoveryByDefault`에서 scheme이 `http`(현재 `http://localhost:9100`)

- [ ] **Step 3: build.gradle에 eureka-client 추가**

`dependencies` 블록의 gateway starter 줄 아래에 추가:
```gradle
    // 서비스 디스커버리 — lb:// 라우트 해석(LoadBalancer 전이 포함) + 게이트웨이 자기등록
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
```

- [ ] **Step 4: application.yml 라우트 uri 기본값 교체 (6곳)**

```yaml
# board 라우트 (1곳)
uri: ${BOARD_SERVICE_URI:lb://board-service}
# auth 라우트 5곳 (auth-oauth2, auth-login, auth-api, auth-jwks, auth-me) 모두
uri: ${AUTH_SERVER_URI:lb://auth-server}
```

같은 파일에 유레카 레지스트리 주소 추가 (`platform:` 블록 위, 최상위 레벨):
```yaml
eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URI:http://localhost:8761/eureka}
```

- [ ] **Step 5: 테스트 전용 유레카 비활성 파일 생성**

`src\test\resources\application.properties` (신규 — main application.yml과 병행 로드되고 키 단위로만 우선함):
```properties
# 단위 테스트는 유레카 서버 없이 돈다 — 등록/조회 시도 차단.
# lb:// 라우트는 인스턴스 0개면 503이 되는데, 테스트는 401 여부만 보거나
# BOARD_SERVICE_URI로 직접 스텁을 주입하므로 영향 없다.
eureka.client.enabled=false
```

- [ ] **Step 6: 전체 테스트 green 확인**

Run: `cd C:\MSA_TEMPLATE\gateway-server; .\gradlew.bat test`
Expected: PASS — RouteConfigTest 신규 테스트 포함 전부 green. 특히 BoardFallbackTest·SlowBoardDownstreamTest(env 시임으로 직접 URI 주입)가 기존대로 통과해야 함

- [ ] **Step 7: 커밋 (gateway-server repo)**

```powershell
cd C:\MSA_TEMPLATE\gateway-server
git add build.gradle src/main/resources/application.yml src/test/resources/application.properties src/test/java/com/platform/gateway/RouteConfigTest.java
git commit -m @'
feat(discovery): 라우트 6개 lb:// 전환 (Eureka 클라이언트 사이드 LB)

기본값만 lb://board-service·lb://auth-server로 교체 — BOARD_SERVICE_URI/
AUTH_SERVER_URI env는 테스트 스텁 시임 + 유레카 없는 환경 탈출구로 유지.
platform.jwks-uri는 라우팅을 안 타므로 직접 URI 유지. 테스트는 eureka 비활성.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
'@
```

---

### Task 5: 수동 E2E 검증 + 문서 현행화

**Files:**
- Modify: `C:\MSA_TEMPLATE\gateway-server\README.md` (라우팅 방식 서술 현행화)
- Modify: `C:\MSA_TEMPLATE\README.md` (서비스 목록에 eureka-server 추가 — 기존 서술 형식 따름)
- Modify: `C:\myBrain\내 로컬\msa\MSA_TEMPLATE 정리\09 유레카 서비스 디스커버리 (2026-07-05).md` (구현 결과 섹션 갱신)
- Copy: 스펙·플랜 → `C:\myBrain\내 로컬\msa\MSA_TEMPLATE 설계문서\` 미러

**Interfaces:**
- Consumes: Task 1~4 완료 상태
- Produces: 없음 (검증·문서)

- [ ] **Step 1: 4개 서비스 기동** (각각 별도 백그라운드 프로세스, 기동 순서 무관하나 관찰 편의상 유레카 먼저)

```powershell
cd C:\MSA_TEMPLATE\eureka-server; .\gradlew.bat bootRun   # :8761
cd C:\MSA_TEMPLATE\auth-server;   .\gradlew.bat bootRun   # :9000 (DB 필요 시 infra compose 선기동)
cd C:\MSA_TEMPLATE\board-service; .\gradlew.bat bootRun   # :9100
cd C:\MSA_TEMPLATE\gateway-server;.\gradlew.bat bootRun   # :8000
```

- [ ] **Step 2: 레지스트리 등록 확인**

Run: `curl -s http://localhost:8761/eureka/apps -H "Accept: application/json"`
Expected: `GATEWAY-SERVER`, `AUTH-SERVER`, `BOARD-SERVICE` 3개 등록 (status UP). 브라우저 `http://localhost:8761` 대시보드로도 확인 가능

- [ ] **Step 3: 게이트웨이 경유 lb:// 라우팅 확인**

```powershell
curl -s -o NUL -w "%{http_code}" http://localhost:8000/.well-known/jwks.json   # lb://auth-server
curl -s -o NUL -w "%{http_code}" http://localhost:8000/api/board/posts          # lb://board-service
```
Expected: 둘 다 200

- [ ] **Step 4: 인스턴스 다운 → fallback 수렴 확인**

board-service 프로세스 종료 → 30~90초 대기(하트비트 퇴출 + 게이트웨이 캐시 갱신) 후:
Run: `curl -s http://localhost:8000/api/board/posts`
Expected: `{"error":"board_unavailable"}` (503 fallback — 퇴출 전 과도기엔 CB가, 퇴출 후엔 no-instance가 잡음)

- [ ] **Step 5: README 2곳 현행화 + 커밋 (각 repo)**

gateway-server README의 라우팅/구성 서술에 반영: 라우트 기본 uri가 lb://이며 유레카(:8761) 레지스트리로 해석된다는 것, env var가 탈출구라는 것. 루트 README 서비스 목록에 eureka-server(:8761) 추가.

```powershell
cd C:\MSA_TEMPLATE\gateway-server
git add README.md
git commit -m @'
docs: README 현행화 — lb:// 서비스 디스커버리 라우팅

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
'@
cd C:\MSA_TEMPLATE
git add README.md
git commit -m @'
docs: README에 eureka-server(:8761) 추가

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
'@
```

- [ ] **Step 6: 옵시디언 문서 갱신**

- `09 유레카 서비스 디스커버리 (2026-07-05).md`: frontmatter `상태: 완료`, "구현 결과" 섹션에 커밋 해시 표(4개 repo)·테스트 목록·E2E 결과·교훈 기록
- 스펙/플랜 미러:
```powershell
Copy-Item C:\MSA_TEMPLATE\docs\superpowers\specs\2026-07-05-eureka-service-discovery-design.md "C:\myBrain\내 로컬\msa\MSA_TEMPLATE 설계문서\specs\" -Force
Copy-Item C:\MSA_TEMPLATE\docs\superpowers\plans\2026-07-05-eureka-service-discovery.md "C:\myBrain\내 로컬\msa\MSA_TEMPLATE 설계문서\plans\" -Force
```

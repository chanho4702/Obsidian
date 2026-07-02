# Keycloak 기반 자체-JWT 로그인 플랫폼 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** myFront(React SPA)를 Keycloak 기반 로그인 플랫폼에 연결한다 — Keycloak이 구글 OIDC·SSO·인증을 담당하고, 신규 `auth-server`가 confidential OIDC 클라이언트로 로그인을 위임받아 **자체 RS256 JWT**(+ 회전형 refresh token, JIT user provisioning)를 발급한다.

**Architecture:** myFront는 자체 JWT만 소지(AT 메모리 / RT HttpOnly 쿠키). auth-server(:9000)는 Keycloak(:8080)에 OIDC 로그인을 위임하고 성공 시 자체 JWT를 발급, `/.well-known/jwks.json`으로 공개키를 노출해 미래 서비스가 `issuer-uri`만으로 검증하게 한다. 사용자·refresh token은 PostgreSQL `authdb`에 영속화한다.

**Tech Stack:** Spring Boot 4.0.6 / Java 24 / Gradle, spring-boot-starter (web, security, oauth2-client, oauth2-resource-server, data-jpa, validation), Flyway, PostgreSQL, Nimbus JOSE+JWT(RS256), Keycloak 26 (Docker), React 19 + MUI v9 + Vite + TypeScript.

## Global Constraints

- auth-server: Spring Boot `4.0.6`, `io.spring.dependency-management` `1.1.7`, Java toolchain `24` (keycloak-sso와 동일).
- auth-server 포트 `9000`. Keycloak `8080`. myFront `5173`. PostgreSQL 호스트 포트 `5433`(컨테이너 5432).
- 자체 Access Token: RS256 JWT, 수명 15분, claim `iss=http://localhost:9000`, `sub`=users.id(문자열), `email`, `name`, `roles`(배열), `provider`, `iat`, `exp`.
- Refresh Token: opaque 랜덤(32바이트, base64url), 수명 14일, 쿠키명 `refresh_token`, `HttpOnly`, `SameSite=Lax`, `Secure=false`(dev), `Path=/api/auth`. DB에는 **SHA-256 해시만** 저장(원문 저장 금지).
- Keycloak realm `sso-demo`, confidential 클라이언트 `platform-bff`, secret `platform-bff-secret`(dev).
- auth-server Java 패키지 루트: `com.platform.authserver`.
- myFront auth 모듈은 배럴(`src/auth`)로만 import. `context/templates/`는 수정 금지.
- 빌드 게이트: auth-server `./gradlew test`(JUnit), myFront `npm run build`(tsc -b 타입체크 포함).
- Spring Boot 4 변경점: `@DataJpaTest`는 `org.springframework.boot.data.jpa.test.autoconfigure` 패키지로 이동했고 `spring-boot-data-jpa-test` testImplementation 의존성이 필요하다(옛 `org.springframework.boot.test.autoconfigure.orm.jpa`는 없음).
- Windows PowerShell 환경. gradlew는 `.\gradlew.bat`, JAVA_HOME=jdk-24 필요.

---

## File Structure

**infra/keycloak/** (keycloak-sso에서 포팅·수정)
- `docker-compose.yml` — Postgres + Keycloak. authdb 생성 init 스크립트 추가.
- `realm-export.json` — `platform-bff` confidential 클라이언트 + 롤 매퍼로 교체.
- `init-authdb.sql` — `CREATE DATABASE authdb` (신규).
- `.env.example` — 구글 자격증명 (포팅).

**auth-server/** (신규 Spring Boot)
- `build.gradle`, `settings.gradle`, `gradlew`/`gradlew.bat`/`gradle/wrapper/*`
- `src/main/resources/application.yml` — 포트, OAuth2 client(keycloak), datasource, jpa, flyway.
- `src/main/resources/db/migration/V1__init.sql` — users, refresh_tokens 스키마.
- `src/main/java/com/platform/authserver/`
  - `AuthServerApplication.java`
  - `jwt/JwtKeyProvider.java` — RSA 키 로드/생성, RSAKey(공개·비밀) 보유.
  - `jwt/JwtService.java` — AT 발급(RS256).
  - `jwt/JwksController.java` — `/.well-known/jwks.json`.
  - `user/User.java`, `user/UserRepository.java`, `user/UserService.java` — JIT provisioning.
  - `token/RefreshToken.java`, `token/RefreshTokenRepository.java`, `token/RefreshTokenService.java` — 발급/회전/재사용탐지, `RefreshResult`/`ReuseDetectedException`.
  - `token/CookieFactory.java` — RT 쿠키 생성/삭제.
  - `auth/LoginSuccessHandler.java` — OIDC 성공 → JIT + RT 발급 + 쿠키 + /app 리다이렉트.
  - `auth/AuthController.java` — `/api/auth/refresh`, `/api/auth/logout`.
  - `auth/MeController.java` — `/api/me`.
  - `auth/OidcClaims.java` — OidcUser claim 추출 헬퍼(sub/email/name/roles).
  - `config/SecurityConfig.java` — 2개 SecurityFilterChain + CORS + JwtDecoder + 롤 컨버터.
- `src/test/java/com/platform/authserver/` — 단위/슬라이스 테스트.

**myFront/** (수정)
- `.env` — `VITE_API_BASE=http://localhost:9000`.
- `src/auth/types.ts` — `AuthClient` 인터페이스에서 `login`/`signup` 제거, `loginUrl()` 추가.
- `src/auth/client.ts` — `login`/`signup` 제거, `loginUrl`/`googleLoginUrl` 추가, `logout`이 Keycloak 로그아웃 URL 처리.
- `src/auth/index.ts` — `loginUrl` 편의 export 추가.
- `src/auth/AuthContext.tsx` — `loginWithPassword`/`signUp` 제거.
- `src/app/pages/LoginPage.tsx` — 비번 폼 제거 → 리다이렉트 버튼 2개.
- `src/main.tsx` — `/register` 라우트 + SignUpPage import 제거.
- `src/app/pages/SignUpPage.tsx` — 삭제.

---

## Task 1: Keycloak 인프라 포팅 (confidential 클라이언트 + authdb)

**Files:**
- Create: `infra/keycloak/docker-compose.yml`
- Create: `infra/keycloak/realm-export.json`
- Create: `infra/keycloak/init-authdb.sql`
- Create: `infra/keycloak/.env.example`

**Interfaces:**
- Produces: Keycloak realm `sso-demo` at `http://localhost:8080`, OIDC issuer `http://localhost:8080/realms/sso-demo`, confidential client `platform-bff`/secret `platform-bff-secret`, redirect URI `http://localhost:9000/login/oauth2/code/keycloak`. PostgreSQL at `localhost:5433` with databases `keycloak` + `authdb` (user/pw `keycloak`/`keycloak`).

- [ ] **Step 1: docker-compose 작성**

`infra/keycloak/docker-compose.yml`:
```yaml
services:
  postgres:
    image: postgres:16
    container_name: platform-postgres
    ports:
      - "5433:5432"
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: keycloak
    volumes:
      - platform-db:/var/lib/postgresql/data
      - ./init-authdb.sql:/docker-entrypoint-initdb.d/init-authdb.sql

  keycloak:
    image: quay.io/keycloak/keycloak:26.0
    container_name: platform-keycloak
    command: start-dev --import-realm
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: admin
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: keycloak
    ports:
      - "8080:8080"
    volumes:
      - ./realm-export.json:/opt/keycloak/data/import/realm-export.json
    depends_on:
      - postgres

volumes:
  platform-db:
```

- [ ] **Step 2: authdb 초기화 스크립트 작성**

`infra/keycloak/init-authdb.sql`:
```sql
-- postgres 컨테이너 최초 기동(빈 볼륨) 시 1회 실행된다.
CREATE DATABASE authdb;
```

- [ ] **Step 3: realm-export 작성 (confidential 클라이언트 + 롤 매퍼)**

`infra/keycloak/realm-export.json`:
```json
{
  "realm": "sso-demo",
  "enabled": true,
  "sslRequired": "none",
  "registrationAllowed": true,
  "roles": {
    "realm": [
      { "name": "USER" },
      { "name": "ADMIN" }
    ]
  },
  "clients": [
    {
      "clientId": "platform-bff",
      "enabled": true,
      "publicClient": false,
      "secret": "platform-bff-secret",
      "standardFlowEnabled": true,
      "directAccessGrantsEnabled": false,
      "redirectUris": ["http://localhost:9000/login/oauth2/code/keycloak"],
      "webOrigins": ["http://localhost:9000"],
      "attributes": {
        "post.logout.redirect.uris": "http://localhost:5173/*##http://localhost:9000/*"
      },
      "protocolMappers": [
        {
          "name": "realm roles",
          "protocol": "openid-connect",
          "protocolMapper": "oidc-usermodel-realm-role-mapper",
          "config": {
            "claim.name": "realm_access.roles",
            "jsonType.label": "String",
            "multivalued": "true",
            "id.token.claim": "true",
            "access.token.claim": "true",
            "userinfo.token.claim": "true"
          }
        }
      ]
    }
  ],
  "users": [
    {
      "username": "alice",
      "enabled": true,
      "email": "alice@demo.com",
      "emailVerified": true,
      "firstName": "Alice",
      "lastName": "Kim",
      "credentials": [{ "type": "password", "value": "alice", "temporary": false }],
      "realmRoles": ["USER"]
    },
    {
      "username": "admin",
      "enabled": true,
      "email": "admin@demo.com",
      "emailVerified": true,
      "firstName": "Admin",
      "lastName": "Kim",
      "credentials": [{ "type": "password", "value": "admin", "temporary": false }],
      "realmRoles": ["USER", "ADMIN"]
    }
  ]
}
```

- [ ] **Step 4: .env.example 작성**

`infra/keycloak/.env.example`:
```
# Google Cloud Console > APIs & Services > Credentials > OAuth 2.0 Client(Web)
# 승인된 리디렉션 URI: http://localhost:8080/realms/sso-demo/broker/google/endpoint
# Keycloak 관리 콘솔 > sso-demo > Identity providers > Google 에 아래 값을 입력한다.
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
```

- [ ] **Step 5: 기동 및 검증**

Run (PowerShell): `cd infra/keycloak; docker compose up -d`
그다음 확인:
```
docker compose ps                 # postgres, keycloak 둘 다 Up
```
브라우저로 `http://localhost:8080/realms/sso-demo/.well-known/openid-configuration` 200 + JSON.
DB 확인: `docker exec platform-postgres psql -U keycloak -lqt` 출력에 `authdb` 와 `keycloak` 둘 다 보일 것.

Expected: realm discovery JSON 응답, authdb 존재.
> 기존 볼륨이 남아 있어 authdb가 없으면: `docker compose down -v` 후 다시 `up -d` (볼륨 초기화 시에만 init 스크립트 실행됨).

- [ ] **Step 6: Commit**

```bash
git add infra/keycloak
git commit -m "feat(infra): platform-bff confidential 클라이언트 + authdb 포함 Keycloak compose

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 2: auth-server 스캐폴드 (Gradle + Boot + DB 연결 + Flyway)

**Files:**
- Create: `auth-server/settings.gradle`, `auth-server/build.gradle`
- Create: `auth-server/gradle/wrapper/gradle-wrapper.properties`, `auth-server/gradlew`, `auth-server/gradlew.bat`, `auth-server/gradle/wrapper/gradle-wrapper.jar`
- Create: `auth-server/src/main/java/com/platform/authserver/AuthServerApplication.java`
- Create: `auth-server/src/main/resources/application.yml`
- Create: `auth-server/src/main/resources/db/migration/V1__init.sql`
- Test: `auth-server/src/test/java/com/platform/authserver/AuthServerApplicationTests.java`

**Interfaces:**
- Produces: 부팅 가능한 Spring Boot 앱(:9000), `authdb` 연결, Flyway가 `users`/`refresh_tokens` 테이블 생성.

- [ ] **Step 1: Gradle 래퍼 복사**

keycloak-sso의 래퍼를 그대로 재사용한다 (PowerShell):
```powershell
New-Item -ItemType Directory -Force auth-server\gradle\wrapper
Copy-Item C:\myStudy\keycloak-sso\app-a\gradlew auth-server\gradlew
Copy-Item C:\myStudy\keycloak-sso\app-a\gradlew.bat auth-server\gradlew.bat
Copy-Item C:\myStudy\keycloak-sso\app-a\gradle\wrapper\* auth-server\gradle\wrapper\
```

- [ ] **Step 2: settings.gradle / build.gradle 작성**

`auth-server/settings.gradle`:
```groovy
rootProject.name = 'auth-server'
```

`auth-server/build.gradle`:
```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.0.6'
    id 'io.spring.dependency-management' version '1.1.7'
}
group = 'com.platform'
version = '0.0.1-SNAPSHOT'
java { toolchain { languageVersion = JavaLanguageVersion.of(24) } }
repositories { mavenCentral() }
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-resource-server'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-database-postgresql'
    runtimeOnly 'org.postgresql:postgresql'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.boot:spring-boot-data-jpa-test'
    testImplementation 'org.springframework.security:spring-security-test'
    testRuntimeOnly 'com.h2database:h2'
}
tasks.named('test') { useJUnitPlatform() }
```

- [ ] **Step 3: application.yml 작성**

`auth-server/src/main/resources/application.yml`:
```yaml
server:
  port: 9000
spring:
  application:
    name: auth-server
  datasource:
    url: jdbc:postgresql://localhost:5433/authdb
    username: keycloak
    password: keycloak
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
  flyway:
    enabled: true
  security:
    oauth2:
      client:
        registration:
          keycloak:
            client-id: platform-bff
            client-secret: platform-bff-secret
            authorization-grant-type: authorization_code
            scope: openid,profile,email
            redirect-uri: "{baseUrl}/login/oauth2/code/keycloak"
        provider:
          keycloak:
            issuer-uri: http://localhost:8080/realms/sso-demo

platform:
  access-token-ttl-seconds: 900
  refresh-token-ttl-seconds: 1209600
  issuer: http://localhost:9000
  frontend-url: http://localhost:5173
  cors-allowed-origin: http://localhost:5173
```

- [ ] **Step 4: 메인 클래스 작성**

`auth-server/src/main/java/com/platform/authserver/AuthServerApplication.java`:
```java
package com.platform.authserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class AuthServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(AuthServerApplication.class, args);
    }
}
```

- [ ] **Step 5: Flyway 마이그레이션 작성**

`auth-server/src/main/resources/db/migration/V1__init.sql`:
```sql
CREATE TABLE users (
    id            BIGSERIAL PRIMARY KEY,
    keycloak_sub  VARCHAR(255) NOT NULL UNIQUE,
    email         VARCHAR(255),
    name          VARCHAR(255),
    roles         VARCHAR(1000) NOT NULL DEFAULT '',
    provider      VARCHAR(50),
    enabled       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at    TIMESTAMP NOT NULL,
    last_login_at TIMESTAMP
);

CREATE TABLE refresh_tokens (
    id           UUID PRIMARY KEY,
    user_id      BIGINT NOT NULL REFERENCES users(id),
    token_hash   VARCHAR(255) NOT NULL UNIQUE,
    family_id    UUID NOT NULL,
    replaced_by  UUID,
    kc_id_token  TEXT,
    expires_at   TIMESTAMP NOT NULL,
    revoked      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at   TIMESTAMP NOT NULL
);

CREATE INDEX idx_refresh_family ON refresh_tokens(family_id);
```
> `roles`는 콤마 구분 문자열(예: `USER,ADMIN`)로 저장해 DB 이식성과 테스트(H2)를 단순화한다.

- [ ] **Step 6: 컨텍스트 로드 테스트 작성 (H2 프로파일)**

`auth-server/src/test/resources/application-test.yml`:
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:authdb;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
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
      client:
        registration:
          keycloak:
            client-id: test
            client-secret: test
            authorization-grant-type: authorization_code
        provider:
          keycloak:
            authorization-uri: http://localhost:8080/auth
            token-uri: http://localhost:8080/token
            user-info-uri: http://localhost:8080/userinfo
            jwk-set-uri: http://localhost:8080/jwks
            user-name-attribute: sub
```

`auth-server/src/test/java/com/platform/authserver/AuthServerApplicationTests.java`:
```java
package com.platform.authserver;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest
@ActiveProfiles("test")
class AuthServerApplicationTests {
    @Test
    void contextLoads() {
    }
}
```
> 테스트는 H2 + Hibernate 자동 DDL을 쓰고 Flyway/실제 Keycloak에 의존하지 않는다. 운영은 Flyway+Postgres.

- [ ] **Step 7: 테스트 실행 (실패 확인 → 통과까지)**

Run (PowerShell): `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test`
Expected: 처음엔 컨텍스트 로드 실패 가능(빈 누락 등) → 위 파일들이 모두 있으면 `contextLoads` PASS.
> 이 단계에서 SecurityConfig 등 후속 빈은 아직 없다. 기본 자동설정만으로 컨텍스트가 떠야 한다. oauth2-client 자동설정이 client registration을 요구하므로 test 프로파일에 더미 registration을 넣었다.

- [ ] **Step 8: .gitignore 작성 후 Commit (auth-server 자체 repo)**

`auth-server/.gitignore` (Create):
```
.gradle/
build/
/auth-jwk.json
.idea/
```
> auth-server는 `github.com/chanho4702/auth-server` 원격이 연결된 자체 git repo다(이미 init·remote 설정됨, 브랜치 `main`). 커밋은 auth-server 디렉토리 안에서 `git add .` 로 한다. 푸시는 오케스트레이터가 리뷰 체크포인트에서 수행한다.
```bash
cd auth-server
git add .
git commit -m "feat(auth-server): authdb 연결 + Flyway 스키마를 갖춘 Spring Boot 스캐폴드

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 3: JWT 서명 키 + Access Token 발급 + JWKS

**Files:**
- Create: `auth-server/src/main/java/com/platform/authserver/jwt/JwtKeyProvider.java`
- Create: `auth-server/src/main/java/com/platform/authserver/jwt/JwtService.java`
- Create: `auth-server/src/main/java/com/platform/authserver/jwt/JwksController.java`
- Test: `auth-server/src/test/java/com/platform/authserver/jwt/JwtServiceTest.java`

**Interfaces:**
- Consumes: `platform.issuer`, `platform.access-token-ttl-seconds` (application.yml).
- Produces:
  - `JwtKeyProvider`: `RSAKey rsaKey()` (private+public), `RSAPublicKey publicKey()`, `String keyId()`.
  - `JwtService.issueAccessToken(long userId, String email, String name, java.util.List<String> roles, String provider) -> String`.
  - `JwksController`: `GET /.well-known/jwks.json` → JWK set JSON.

- [ ] **Step 1: 실패하는 테스트 작성**

`auth-server/src/test/java/com/platform/authserver/jwt/JwtServiceTest.java`:
```java
package com.platform.authserver.jwt;

import com.nimbusds.jose.JWSVerifier;
import com.nimbusds.jose.crypto.RSASSAVerifier;
import com.nimbusds.jwt.SignedJWT;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class JwtServiceTest {

    private final JwtKeyProvider keyProvider = new JwtKeyProvider(null);
    private final JwtService jwtService = new JwtService(keyProvider, "http://localhost:9000", 900);

    JwtServiceTest() throws Exception {
        keyProvider.init();
    }

    @Test
    void issuesVerifiableRs256TokenWithExpectedClaims() throws Exception {
        String token = jwtService.issueAccessToken(42L, "alice@demo.com", "Alice", List.of("USER", "ADMIN"), "GOOGLE");

        SignedJWT jwt = SignedJWT.parse(token);
        JWSVerifier verifier = new RSASSAVerifier(keyProvider.publicKey());

        assertThat(jwt.verify(verifier)).isTrue();
        assertThat(jwt.getHeader().getKeyID()).isEqualTo(keyProvider.keyId());
        assertThat(jwt.getJWTClaimsSet().getIssuer()).isEqualTo("http://localhost:9000");
        assertThat(jwt.getJWTClaimsSet().getSubject()).isEqualTo("42");
        assertThat(jwt.getJWTClaimsSet().getStringClaim("email")).isEqualTo("alice@demo.com");
        assertThat(jwt.getJWTClaimsSet().getStringClaim("name")).isEqualTo("Alice");
        assertThat(jwt.getJWTClaimsSet().getStringClaim("provider")).isEqualTo("GOOGLE");
        assertThat(jwt.getJWTClaimsSet().getStringListClaim("roles")).containsExactly("USER", "ADMIN");
        assertThat(jwt.getJWTClaimsSet().getExpirationTime()).isAfter(new java.util.Date());
    }
}
```

- [ ] **Step 2: 테스트 실행 → 컴파일 실패 확인**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test --tests '*JwtServiceTest*'`
Expected: FAIL — `JwtKeyProvider`/`JwtService` 클래스 없음(컴파일 에러).

- [ ] **Step 3: JwtKeyProvider 구현**

`auth-server/src/main/java/com/platform/authserver/jwt/JwtKeyProvider.java`:
```java
package com.platform.authserver.jwt;

import com.nimbusds.jose.jwk.RSAKey;
import com.nimbusds.jose.jwk.gen.RSAKeyGenerator;
import jakarta.annotation.PostConstruct;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.interfaces.RSAPublicKey;
import java.util.UUID;

/**
 * 자체 JWT 서명용 RSA 키. dev: keyPath 파일이 있으면 로드, 없으면 생성 후 저장(재시작해도
 * 발급한 토큰이 유효하도록 키를 고정). keyPath 가 null 이면 메모리에만 생성(테스트용).
 */
@Component
public class JwtKeyProvider {

    private final String keyPath;
    private RSAKey rsaKey;

    public JwtKeyProvider(
            @org.springframework.beans.factory.annotation.Value("${platform.jwk-path:./auth-jwk.json}") String keyPath) {
        this.keyPath = keyPath;
    }

    @PostConstruct
    public void init() throws Exception {
        if (keyPath != null) {
            Path path = Path.of(keyPath);
            if (Files.exists(path)) {
                this.rsaKey = RSAKey.parse(Files.readString(path));
                return;
            }
        }
        this.rsaKey = new RSAKeyGenerator(2048)
                .keyID(UUID.randomUUID().toString())
                .generate();
        if (keyPath != null) {
            persist(Path.of(keyPath));
        }
    }

    private void persist(Path path) throws IOException {
        Files.writeString(path, rsaKey.toJSONString());
    }

    public RSAKey rsaKey() {
        return rsaKey;
    }

    public RSAPublicKey publicKey() throws Exception {
        return rsaKey.toRSAPublicKey();
    }

    public String keyId() {
        return rsaKey.getKeyID();
    }
}
```

- [ ] **Step 4: JwtService 구현**

`auth-server/src/main/java/com/platform/authserver/jwt/JwtService.java`:
```java
package com.platform.authserver.jwt;

import com.nimbusds.jose.JOSEException;
import com.nimbusds.jose.JWSAlgorithm;
import com.nimbusds.jose.JWSHeader;
import com.nimbusds.jose.crypto.RSASSASigner;
import com.nimbusds.jwt.JWTClaimsSet;
import com.nimbusds.jwt.SignedJWT;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.Date;
import java.util.List;

@Service
public class JwtService {

    private final JwtKeyProvider keyProvider;
    private final String issuer;
    private final long ttlSeconds;

    public JwtService(JwtKeyProvider keyProvider,
                      @Value("${platform.issuer}") String issuer,
                      @Value("${platform.access-token-ttl-seconds}") long ttlSeconds) {
        this.keyProvider = keyProvider;
        this.issuer = issuer;
        this.ttlSeconds = ttlSeconds;
    }

    public String issueAccessToken(long userId, String email, String name, List<String> roles, String provider) {
        Instant now = Instant.now();
        try {
            JWTClaimsSet claims = new JWTClaimsSet.Builder()
                    .issuer(issuer)
                    .subject(String.valueOf(userId))
                    .claim("email", email)
                    .claim("name", name)
                    .claim("provider", provider)
                    .claim("roles", roles)
                    .issueTime(Date.from(now))
                    .expirationTime(Date.from(now.plusSeconds(ttlSeconds)))
                    .build();

            SignedJWT jwt = new SignedJWT(
                    new JWSHeader.Builder(JWSAlgorithm.RS256).keyID(keyProvider.keyId()).build(),
                    claims);
            jwt.sign(new RSASSASigner(keyProvider.rsaKey()));
            return jwt.serialize();
        } catch (JOSEException e) {
            throw new IllegalStateException("AT 발급 실패", e);
        }
    }
}
```

- [ ] **Step 5: JwksController 구현**

`auth-server/src/main/java/com/platform/authserver/jwt/JwksController.java`:
```java
package com.platform.authserver.jwt;

import com.nimbusds.jose.jwk.JWKSet;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
public class JwksController {

    private final JwtKeyProvider keyProvider;

    public JwksController(JwtKeyProvider keyProvider) {
        this.keyProvider = keyProvider;
    }

    /** 공개키만 노출(toPublicJWK). 미래 서비스/게이트웨이가 이 URL로 자체 JWT를 검증한다. */
    @GetMapping("/.well-known/jwks.json")
    public Map<String, Object> jwks() {
        return new JWKSet(keyProvider.rsaKey().toPublicJWK()).toJSONObject();
    }
}
```

- [ ] **Step 6: 테스트 실행 → 통과 확인**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test --tests '*JwtServiceTest*'`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
cd auth-server
git add .
git commit -m "feat(auth-server): RS256 자체 JWT 발급 + JWKS 엔드포인트

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 4: User 엔티티 + JIT 프로비저닝

**Files:**
- Create: `auth-server/src/main/java/com/platform/authserver/user/User.java`
- Create: `auth-server/src/main/java/com/platform/authserver/user/UserRepository.java`
- Create: `auth-server/src/main/java/com/platform/authserver/user/UserService.java`
- Test: `auth-server/src/test/java/com/platform/authserver/user/UserServiceTest.java`

**Interfaces:**
- Produces:
  - `User` 엔티티: `getId()`, `getKeycloakSub()`, `getEmail()`, `getName()`, `getRoles()` (List<String>), `getProvider()`.
  - `UserRepository extends JpaRepository<User, Long>`: `Optional<User> findByKeycloakSub(String sub)`.
  - `UserService.provision(String keycloakSub, String email, String name, List<String> roles, String provider) -> User` — 없으면 생성, 있으면 email/name/roles/provider/lastLoginAt 갱신.

- [ ] **Step 1: 실패하는 테스트 작성**

`auth-server/src/test/java/com/platform/authserver/user/UserServiceTest.java`:
```java
package com.platform.authserver.user;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest; // Spring Boot 4: 새 패키지(+ testImpl spring-boot-data-jpa-test)
import org.springframework.context.annotation.Import;
import org.springframework.test.context.ActiveProfiles;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@ActiveProfiles("test")
@Import(UserService.class)
class UserServiceTest {

    @Autowired UserService userService;
    @Autowired UserRepository userRepository;

    @Test
    void createsUserOnFirstLogin() {
        User u = userService.provision("kc-sub-1", "alice@demo.com", "Alice", List.of("USER"), "GOOGLE");

        assertThat(u.getId()).isNotNull();
        assertThat(u.getKeycloakSub()).isEqualTo("kc-sub-1");
        assertThat(u.getRoles()).containsExactly("USER");
        assertThat(userRepository.count()).isEqualTo(1);
    }

    @Test
    void reusesUserAndUpdatesProfileOnSecondLogin() {
        userService.provision("kc-sub-1", "alice@demo.com", "Alice", List.of("USER"), "GOOGLE");
        User again = userService.provision("kc-sub-1", "alice@new.com", "Alice K", List.of("USER", "ADMIN"), "GOOGLE");

        assertThat(userRepository.count()).isEqualTo(1);
        assertThat(again.getEmail()).isEqualTo("alice@new.com");
        assertThat(again.getRoles()).containsExactly("USER", "ADMIN");
        assertThat(again.getLastLoginAt()).isNotNull();
    }
}
```

- [ ] **Step 2: 테스트 실행 → 컴파일 실패 확인**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test --tests '*UserServiceTest*'`
Expected: FAIL — `User`/`UserRepository`/`UserService` 없음.

- [ ] **Step 3: User 엔티티 구현**

`auth-server/src/main/java/com/platform/authserver/user/User.java`:
```java
package com.platform.authserver.user;

import jakarta.persistence.*;

import java.time.Instant;
import java.util.Arrays;
import java.util.List;

@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "keycloak_sub", nullable = false, unique = true)
    private String keycloakSub;

    private String email;
    private String name;

    /** 콤마 구분 문자열로 저장(예: "USER,ADMIN"). */
    @Column(nullable = false)
    private String roles = "";

    private String provider;

    @Column(nullable = false)
    private boolean enabled = true;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @Column(name = "last_login_at")
    private Instant lastLoginAt;

    protected User() {
    }

    public User(String keycloakSub) {
        this.keycloakSub = keycloakSub;
        this.createdAt = Instant.now();
    }

    public List<String> getRoles() {
        if (roles == null || roles.isBlank()) {
            return List.of();
        }
        return Arrays.asList(roles.split(","));
    }

    public void setRoles(List<String> roleList) {
        this.roles = String.join(",", roleList);
    }

    public Long getId() { return id; }
    public String getKeycloakSub() { return keycloakSub; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getProvider() { return provider; }
    public void setProvider(String provider) { this.provider = provider; }
    public Instant getLastLoginAt() { return lastLoginAt; }
    public void setLastLoginAt(Instant t) { this.lastLoginAt = t; }
}
```

- [ ] **Step 4: UserRepository 구현**

`auth-server/src/main/java/com/platform/authserver/user/UserRepository.java`:
```java
package com.platform.authserver.user;

import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByKeycloakSub(String keycloakSub);
}
```

- [ ] **Step 5: UserService 구현**

`auth-server/src/main/java/com/platform/authserver/user/UserService.java`:
```java
package com.platform.authserver.user;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;
import java.util.List;

@Service
public class UserService {

    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    /** Keycloak 로그인마다 호출. 없으면 생성, 있으면 프로필 동기화(JIT provisioning). */
    @Transactional
    public User provision(String keycloakSub, String email, String name, List<String> roles, String provider) {
        User user = repository.findByKeycloakSub(keycloakSub).orElseGet(() -> new User(keycloakSub));
        user.setEmail(email);
        user.setName(name);
        user.setRoles(roles);
        user.setProvider(provider);
        user.setLastLoginAt(Instant.now());
        return repository.save(user);
    }
}
```

- [ ] **Step 6: 테스트 실행 → 통과 확인**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test --tests '*UserServiceTest*'`
Expected: PASS (두 테스트 모두).

- [ ] **Step 7: Commit**

```bash
cd auth-server
git add .
git commit -m "feat(auth-server): User 엔티티 + JIT 프로비저닝

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 5: Refresh Token 발급/회전/재사용 탐지

**Files:**
- Create: `auth-server/src/main/java/com/platform/authserver/token/RefreshToken.java`
- Create: `auth-server/src/main/java/com/platform/authserver/token/RefreshTokenRepository.java`
- Create: `auth-server/src/main/java/com/platform/authserver/token/RefreshTokenService.java`
- Create: `auth-server/src/main/java/com/platform/authserver/token/ReuseDetectedException.java`
- Test: `auth-server/src/test/java/com/platform/authserver/token/RefreshTokenServiceTest.java`

**Interfaces:**
- Consumes: `User` (Task 4), `platform.refresh-token-ttl-seconds`.
- Produces:
  - `RefreshTokenService.issue(User user, String kcIdToken) -> Issued` where `record Issued(String rawToken, String kcIdToken)`.
  - `RefreshTokenService.rotate(String rawToken) -> Rotated` where `record Rotated(User user, String newRawToken, String kcIdToken)`. 유효하지 않으면 `IllegalArgumentException`, 재사용 탐지 시 `ReuseDetectedException`.
  - `RefreshTokenService.revokeFamilyByRawToken(String rawToken) -> String kcIdToken` (로그아웃; 없으면 null 반환).
  - `static String sha256(String raw)`.

- [ ] **Step 1: 실패하는 테스트 작성**

`auth-server/src/test/java/com/platform/authserver/token/RefreshTokenServiceTest.java`:
```java
package com.platform.authserver.token;

import com.platform.authserver.user.User;
import com.platform.authserver.user.UserRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest; // Spring Boot 4: 새 패키지(+ testImpl spring-boot-data-jpa-test)
import org.springframework.context.annotation.Import;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.util.ReflectionTestUtils;

import java.util.List;

import static org.assertj.core.api.Assertions.*;

@DataJpaTest
@ActiveProfiles("test")
@Import(RefreshTokenService.class)
class RefreshTokenServiceTest {

    @Autowired RefreshTokenService service;
    @Autowired UserRepository userRepository;
    @Autowired RefreshTokenRepository tokenRepository;

    private User newUser() {
        User u = new User("kc-sub-1");
        u.setRoles(List.of("USER"));
        return userRepository.save(u);
    }

    @Test
    void rotateReturnsNewTokenAndRevokesOld() {
        // ttl 주입(슬라이스 테스트라 @Value 미적용)
        ReflectionTestUtils.setField(service, "ttlSeconds", 1209600L);
        User u = newUser();
        var issued = service.issue(u, "kc-id-token");

        var rotated = service.rotate(issued.rawToken());

        assertThat(rotated.user().getId()).isEqualTo(u.getId());
        assertThat(rotated.newRawToken()).isNotEqualTo(issued.rawToken());
        assertThat(rotated.kcIdToken()).isEqualTo("kc-id-token");
        // 옛 토큰 재사용은 이제 도난으로 간주
        assertThatThrownBy(() -> service.rotate(issued.rawToken()))
                .isInstanceOf(ReuseDetectedException.class);
    }

    @Test
    void reuseDetectionRevokesWholeFamily() {
        ReflectionTestUtils.setField(service, "ttlSeconds", 1209600L);
        User u = newUser();
        var issued = service.issue(u, "kc-id-token");
        var rotated = service.rotate(issued.rawToken()); // 회전 1회

        // 폐기된 옛 토큰 재사용 → 패밀리 전체 폐기
        assertThatThrownBy(() -> service.rotate(issued.rawToken()))
                .isInstanceOf(ReuseDetectedException.class);
        // 패밀리가 폐기됐으므로 방금 회전한 정상 토큰도 더는 못 씀
        assertThatThrownBy(() -> service.rotate(rotated.newRawToken()))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void invalidTokenRejected() {
        ReflectionTestUtils.setField(service, "ttlSeconds", 1209600L);
        assertThatThrownBy(() -> service.rotate("not-a-real-token"))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: 테스트 실행 → 컴파일 실패 확인**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test --tests '*RefreshTokenServiceTest*'`
Expected: FAIL — 클래스 없음.

- [ ] **Step 3: RefreshToken 엔티티 구현**

`auth-server/src/main/java/com/platform/authserver/token/RefreshToken.java`:
```java
package com.platform.authserver.token;

import jakarta.persistence.*;

import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "refresh_tokens")
public class RefreshToken {

    @Id
    private UUID id;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "token_hash", nullable = false, unique = true)
    private String tokenHash;

    @Column(name = "family_id", nullable = false)
    private UUID familyId;

    @Column(name = "replaced_by")
    private UUID replacedBy;

    @Column(name = "kc_id_token", columnDefinition = "TEXT")
    private String kcIdToken;

    @Column(name = "expires_at", nullable = false)
    private Instant expiresAt;

    @Column(nullable = false)
    private boolean revoked = false;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    protected RefreshToken() {
    }

    public RefreshToken(Long userId, String tokenHash, UUID familyId, String kcIdToken, Instant expiresAt) {
        this.id = UUID.randomUUID();
        this.userId = userId;
        this.tokenHash = tokenHash;
        this.familyId = familyId;
        this.kcIdToken = kcIdToken;
        this.expiresAt = expiresAt;
        this.createdAt = Instant.now();
    }

    public UUID getId() { return id; }
    public Long getUserId() { return userId; }
    public UUID getFamilyId() { return familyId; }
    public String getKcIdToken() { return kcIdToken; }
    public Instant getExpiresAt() { return expiresAt; }
    public boolean isRevoked() { return revoked; }
    public void setRevoked(boolean revoked) { this.revoked = revoked; }
    public void setReplacedBy(UUID replacedBy) { this.replacedBy = replacedBy; }
}
```

- [ ] **Step 4: Repository + 예외 구현**

`auth-server/src/main/java/com/platform/authserver/token/RefreshTokenRepository.java`:
```java
package com.platform.authserver.token;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;

import java.util.Optional;
import java.util.UUID;

public interface RefreshTokenRepository extends JpaRepository<RefreshToken, UUID> {
    Optional<RefreshToken> findByTokenHash(String tokenHash);

    @Modifying
    @Query("update RefreshToken t set t.revoked = true where t.familyId = :familyId")
    void revokeFamily(UUID familyId);
}
```

`auth-server/src/main/java/com/platform/authserver/token/ReuseDetectedException.java`:
```java
package com.platform.authserver.token;

public class ReuseDetectedException extends RuntimeException {
    public ReuseDetectedException(String message) {
        super(message);
    }
}
```

- [ ] **Step 5: RefreshTokenService 구현**

`auth-server/src/main/java/com/platform/authserver/token/RefreshTokenService.java`:
```java
package com.platform.authserver.token;

import com.platform.authserver.user.User;
import com.platform.authserver.user.UserRepository;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.time.Instant;
import java.util.Base64;
import java.util.UUID;

@Service
public class RefreshTokenService {

    public record Issued(String rawToken, String kcIdToken) {}
    public record Rotated(User user, String newRawToken, String kcIdToken) {}

    private static final SecureRandom RANDOM = new SecureRandom();

    private final RefreshTokenRepository tokenRepository;
    private final UserRepository userRepository;

    @Value("${platform.refresh-token-ttl-seconds}")
    private long ttlSeconds;

    public RefreshTokenService(RefreshTokenRepository tokenRepository, UserRepository userRepository) {
        this.tokenRepository = tokenRepository;
        this.userRepository = userRepository;
    }

    @Transactional
    public Issued issue(User user, String kcIdToken) {
        String raw = newRawToken();
        UUID familyId = UUID.randomUUID();
        persist(user.getId(), raw, familyId, kcIdToken);
        return new Issued(raw, kcIdToken);
    }

    @Transactional
    public Rotated rotate(String rawToken) {
        RefreshToken current = tokenRepository.findByTokenHash(sha256(rawToken))
                .orElseThrow(() -> new IllegalArgumentException("알 수 없는 refresh token"));

        if (current.isRevoked()) {
            // 폐기된 토큰 재사용 = 도난. 패밀리 전체 폐기.
            tokenRepository.revokeFamily(current.getFamilyId());
            throw new ReuseDetectedException("refresh token 재사용 탐지");
        }
        if (current.getExpiresAt().isBefore(Instant.now())) {
            throw new IllegalArgumentException("만료된 refresh token");
        }

        User user = userRepository.findById(current.getUserId())
                .orElseThrow(() -> new IllegalArgumentException("사용자 없음"));

        String newRaw = newRawToken();
        RefreshToken next = persist(user.getId(), newRaw, current.getFamilyId(), current.getKcIdToken());
        current.setRevoked(true);
        current.setReplacedBy(next.getId());
        tokenRepository.save(current);

        return new Rotated(user, newRaw, current.getKcIdToken());
    }

    /** 로그아웃: 해당 토큰의 패밀리를 폐기하고 Keycloak id_token 을 반환(없으면 null). */
    @Transactional
    public String revokeFamilyByRawToken(String rawToken) {
        if (rawToken == null) {
            return null;
        }
        return tokenRepository.findByTokenHash(sha256(rawToken))
                .map(t -> {
                    tokenRepository.revokeFamily(t.getFamilyId());
                    return t.getKcIdToken();
                })
                .orElse(null);
    }

    private RefreshToken persist(Long userId, String raw, UUID familyId, String kcIdToken) {
        RefreshToken token = new RefreshToken(
                userId, sha256(raw), familyId, kcIdToken, Instant.now().plusSeconds(ttlSeconds));
        return tokenRepository.save(token);
    }

    private static String newRawToken() {
        byte[] bytes = new byte[32];
        RANDOM.nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }

    public static String sha256(String raw) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            byte[] hash = md.digest(raw.getBytes(StandardCharsets.UTF_8));
            return Base64.getUrlEncoder().withoutPadding().encodeToString(hash);
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
    }
}
```

- [ ] **Step 6: 테스트 실행 → 통과 확인**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test --tests '*RefreshTokenServiceTest*'`
Expected: PASS (3개 테스트 모두).

- [ ] **Step 7: Commit**

```bash
cd auth-server
git add .
git commit -m "feat(auth-server): refresh 토큰 회전 + 재사용 탐지

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 6: 보안 설정 + OIDC 로그인 성공 핸들러 + 쿠키

**Files:**
- Create: `auth-server/src/main/java/com/platform/authserver/auth/OidcClaims.java`
- Create: `auth-server/src/main/java/com/platform/authserver/token/CookieFactory.java`
- Create: `auth-server/src/main/java/com/platform/authserver/auth/LoginSuccessHandler.java`
- Create: `auth-server/src/main/java/com/platform/authserver/config/SecurityConfig.java`
- Test: `auth-server/src/test/java/com/platform/authserver/auth/OidcClaimsTest.java`

**Interfaces:**
- Consumes: `UserService.provision(...)`, `RefreshTokenService.issue(...)`, `JwtKeyProvider.publicKey()`.
- Produces:
  - `OidcClaims.roles(OidcUser) -> List<String>` (realm_access.roles 추출), `OidcClaims.provider(OidcUser) -> String`.
  - `CookieFactory.refreshCookie(String rawToken) -> ResponseCookie`, `CookieFactory.deleteCookie() -> ResponseCookie`, constant `COOKIE_NAME = "refresh_token"`.
  - Security: 2개 필터체인. `/api/me` = 자체 JWT(resource server). 그 외 = oauth2Login + permitAll(/api/auth/**, /.well-known/**).

- [ ] **Step 1: OidcClaims 실패 테스트 작성**

`auth-server/src/test/java/com/platform/authserver/auth/OidcClaimsTest.java`:
```java
package com.platform.authserver.auth;

import org.junit.jupiter.api.Test;
import org.springframework.security.oauth2.core.oidc.OidcIdToken;
import org.springframework.security.oauth2.core.oidc.user.DefaultOidcUser;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class OidcClaimsTest {

    private OidcUser userWith(Map<String, Object> claims) {
        OidcIdToken idToken = new OidcIdToken(
                "tok", Instant.now(), Instant.now().plusSeconds(60),
                Map.of("sub", "kc-1", "email", "a@b.com", "name", "Alice",
                        "realm_access", Map.of("roles", List.of("USER", "ADMIN")),
                        "identity_provider", "google"));
        return new DefaultOidcUser(List.of(), idToken);
    }

    @Test
    void extractsRealmRoles() {
        OidcUser user = userWith(Map.of());
        assertThat(OidcClaims.roles(user)).containsExactlyInAnyOrder("USER", "ADMIN");
    }

    @Test
    void mapsIdentityProvider() {
        OidcUser user = userWith(Map.of());
        assertThat(OidcClaims.provider(user)).isEqualTo("GOOGLE");
    }
}
```

- [ ] **Step 2: 테스트 실행 → 컴파일 실패 확인**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test --tests '*OidcClaimsTest*'`
Expected: FAIL — `OidcClaims` 없음.

- [ ] **Step 3: OidcClaims 구현**

`auth-server/src/main/java/com/platform/authserver/auth/OidcClaims.java`:
```java
package com.platform.authserver.auth;

import org.springframework.security.oauth2.core.oidc.user.OidcUser;

import java.util.List;
import java.util.Map;

/** Keycloak OIDC 사용자 claim에서 플랫폼이 쓰는 값(roles/provider)을 뽑는다. */
public final class OidcClaims {

    private OidcClaims() {
    }

    @SuppressWarnings("unchecked")
    public static List<String> roles(OidcUser user) {
        Object realmAccess = user.getClaims().get("realm_access");
        if (realmAccess instanceof Map<?, ?> map && map.get("roles") instanceof List<?> roles) {
            return roles.stream().map(String::valueOf).toList();
        }
        return List.of();
    }

    /** Keycloak이 브로커링하면 identity_provider claim에 "google" 등이 담긴다. 없으면 KEYCLOAK. */
    public static String provider(OidcUser user) {
        Object idp = user.getClaims().get("identity_provider");
        if (idp != null && !idp.toString().isBlank()) {
            return idp.toString().toUpperCase();
        }
        return "KEYCLOAK";
    }
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test --tests '*OidcClaimsTest*'`
Expected: PASS.

- [ ] **Step 5: CookieFactory 구현**

`auth-server/src/main/java/com/platform/authserver/token/CookieFactory.java`:
```java
package com.platform.authserver.token;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.ResponseCookie;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Component
public class CookieFactory {

    public static final String COOKIE_NAME = "refresh_token";

    private final long ttlSeconds;

    public CookieFactory(@Value("${platform.refresh-token-ttl-seconds}") long ttlSeconds) {
        this.ttlSeconds = ttlSeconds;
    }

    public ResponseCookie refreshCookie(String rawToken) {
        return ResponseCookie.from(COOKIE_NAME, rawToken)
                .httpOnly(true)
                .secure(false) // dev(localhost). 운영 https에서는 true.
                .sameSite("Lax")
                .path("/api/auth")
                .maxAge(Duration.ofSeconds(ttlSeconds))
                .build();
    }

    public ResponseCookie deleteCookie() {
        return ResponseCookie.from(COOKIE_NAME, "")
                .httpOnly(true)
                .secure(false)
                .sameSite("Lax")
                .path("/api/auth")
                .maxAge(0)
                .build();
    }
}
```

- [ ] **Step 6: LoginSuccessHandler 구현**

`auth-server/src/main/java/com/platform/authserver/auth/LoginSuccessHandler.java`:
```java
package com.platform.authserver.auth;

import com.platform.authserver.token.CookieFactory;
import com.platform.authserver.token.RefreshTokenService;
import com.platform.authserver.user.User;
import com.platform.authserver.user.UserService;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpHeaders;
import org.springframework.security.core.Authentication;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;
import org.springframework.security.web.authentication.AuthenticationSuccessHandler;
import org.springframework.stereotype.Component;

import java.io.IOException;

/**
 * Keycloak OIDC 로그인 성공 후:
 * 1) JIT user 프로비저닝  2) 자체 refresh token 발급(+Keycloak id_token 보관)
 * 3) RT 쿠키 set  4) 프론트 /app 로 리다이렉트.
 * 자체 AT는 여기서 주지 않는다 — 프론트가 마운트 시 /api/auth/refresh 로 받는다(silent restore).
 */
@Component
public class LoginSuccessHandler implements AuthenticationSuccessHandler {

    private final UserService userService;
    private final RefreshTokenService refreshTokenService;
    private final CookieFactory cookieFactory;
    private final String frontendUrl;

    public LoginSuccessHandler(UserService userService,
                               RefreshTokenService refreshTokenService,
                               CookieFactory cookieFactory,
                               @Value("${platform.frontend-url}") String frontendUrl) {
        this.userService = userService;
        this.refreshTokenService = refreshTokenService;
        this.cookieFactory = cookieFactory;
        this.frontendUrl = frontendUrl;
    }

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,
                                        Authentication authentication) throws IOException {
        OidcUser oidcUser = (OidcUser) authentication.getPrincipal();

        User user = userService.provision(
                oidcUser.getSubject(),
                oidcUser.getEmail(),
                oidcUser.getFullName(),
                OidcClaims.roles(oidcUser),
                OidcClaims.provider(oidcUser));

        String kcIdToken = oidcUser.getIdToken().getTokenValue();
        var issued = refreshTokenService.issue(user, kcIdToken);

        response.addHeader(HttpHeaders.SET_COOKIE, cookieFactory.refreshCookie(issued.rawToken()).toString());
        response.sendRedirect(frontendUrl + "/app");
    }
}
```

- [ ] **Step 7: SecurityConfig 구현**

`auth-server/src/main/java/com/platform/authserver/config/SecurityConfig.java`:
```java
package com.platform.authserver.config;

import com.platform.authserver.auth.LoginSuccessHandler;
import com.platform.authserver.jwt.JwtKeyProvider;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.security.oauth2.jwt.NimbusJwtDecoder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.List;

@Configuration
public class SecurityConfig {

    private final String allowedOrigin;

    public SecurityConfig(org.springframework.core.env.Environment env) {
        this.allowedOrigin = env.getProperty("platform.cors-allowed-origin", "http://localhost:5173");
    }

    /** 자체 AT 검증용 디코더 — 우리 공개키로 RS256 검증. */
    @Bean
    JwtDecoder jwtDecoder(JwtKeyProvider keyProvider) throws Exception {
        return NimbusJwtDecoder.withPublicKey(keyProvider.publicKey()).build();
    }

    /** 자체 AT의 roles claim → ROLE_* 권한. */
    @Bean
    JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authorities = new JwtGrantedAuthoritiesConverter();
        authorities.setAuthoritiesClaimName("roles");
        authorities.setAuthorityPrefix("ROLE_");
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authorities);
        return converter;
    }

    /** 체인 A: /api/me 등 보호 API — 자체 JWT(stateless resource server). */
    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE)
    SecurityFilterChain apiChain(HttpSecurity http, JwtAuthenticationConverter converter) throws Exception {
        http
                .securityMatcher("/api/me")
                .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .csrf(csrf -> csrf.disable())
                .cors(Customizer.withDefaults())
                .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(converter)));
        return http.build();
    }

    /** 체인 B: 그 외 — Keycloak OIDC 로그인 + 커스텀 엔드포인트 permitAll. */
    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE + 1)
    SecurityFilterChain webChain(HttpSecurity http, LoginSuccessHandler successHandler) throws Exception {
        http
                .csrf(csrf -> csrf.disable())
                .cors(Customizer.withDefaults())
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/auth/**", "/.well-known/**", "/error").permitAll()
                        .anyRequest().authenticated())
                .oauth2Login(oauth2 -> oauth2.successHandler(successHandler));
        return http.build();
    }

    // CORS: myFront(:5173) 출처에서 credentials(쿠키) 포함 요청 허용.
    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of(allowedOrigin));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```
- [ ] **Step 8: 전체 테스트 실행 (컨텍스트가 새 빈들과 함께 뜨는지)**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test`
Expected: PASS — 기존 테스트 전부 + 컨텍스트 로드(SecurityConfig/핸들러 포함).

- [ ] **Step 9: Commit**

```bash
cd auth-server
git add .
git commit -m "feat(auth-server): 보안 필터체인 2개 + OIDC 로그인 성공 핸들러 + RT 쿠키

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 7: /api/auth/refresh · /api/auth/logout · /api/me 엔드포인트

**Files:**
- Create: `auth-server/src/main/java/com/platform/authserver/auth/AuthController.java`
- Create: `auth-server/src/main/java/com/platform/authserver/auth/MeController.java`
- Test: `auth-server/src/test/java/com/platform/authserver/auth/MeControllerTest.java`
- Test: `auth-server/src/test/java/com/platform/authserver/auth/AuthControllerTest.java`

**Interfaces:**
- Consumes: `RefreshTokenService`, `JwtService`, `CookieFactory`, `provider`/`issuer`/`frontendUrl` props.
- Produces:
  - `POST /api/auth/refresh` → 200 `{ "accessToken": "<jwt>" }` + 새 RT 쿠키. 실패 401.
  - `POST /api/auth/logout` → 200 `{ "keycloakLogoutUrl": "<url|null>" }` + RT 쿠키 삭제.
  - `GET /api/me` → 200 `{ email, name, sub, role, provider }` (자체 AT 필요).

- [ ] **Step 1: MeController 실패 테스트 작성**

`auth-server/src/test/java/com/platform/authserver/auth/MeControllerTest.java`:
```java
package com.platform.authserver.auth;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import java.util.List;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;
import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@ActiveProfiles("test")
class MeControllerTest {

    @Autowired WebApplicationContext context;
    MockMvc mvc;

    @org.junit.jupiter.api.BeforeEach
    void setup() {
        mvc = MockMvcBuilders.webAppContextSetup(context).apply(springSecurity()).build();
    }

    @Test
    void returnsUserFromJwtClaims() throws Exception {
        mvc.perform(get("/api/me").with(jwt().jwt(j -> j
                        .subject("42")
                        .claim("email", "alice@demo.com")
                        .claim("name", "Alice")
                        .claim("provider", "GOOGLE")
                        .claim("roles", List.of("USER", "ADMIN")))))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.sub").value("42"))
                .andExpect(jsonPath("$.email").value("alice@demo.com"))
                .andExpect(jsonPath("$.role").value("USER"));
    }

    @Test
    void rejectsWithoutToken() throws Exception {
        mvc.perform(get("/api/me")).andExpect(status().isUnauthorized());
    }
}
```

- [ ] **Step 2: 테스트 실행 → 실패 확인**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test --tests '*MeControllerTest*'`
Expected: FAIL — `/api/me` 404/없음(컨트롤러 미존재).

- [ ] **Step 3: MeController 구현**

`auth-server/src/main/java/com/platform/authserver/auth/MeController.java`:
```java
package com.platform.authserver.auth;

import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api")
public class MeController {

    /** 자체 AT claim을 프론트 AppUser 형태({email,name,provider,sub,role})로 반환. */
    @GetMapping("/me")
    public Map<String, Object> me(@AuthenticationPrincipal Jwt jwt) {
        List<String> roles = jwt.getClaimAsStringList("roles");
        Map<String, Object> body = new HashMap<>();
        body.put("sub", jwt.getSubject());
        body.put("email", jwt.getClaimAsString("email"));
        body.put("name", jwt.getClaimAsString("name"));
        body.put("provider", jwt.getClaimAsString("provider"));
        body.put("role", (roles == null || roles.isEmpty()) ? null : roles.get(0));
        return body;
    }
}
```

- [ ] **Step 4: 테스트 실행 → 통과 확인**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test --tests '*MeControllerTest*'`
Expected: PASS.

- [ ] **Step 5: AuthController 실패 테스트 작성**

`auth-server/src/test/java/com/platform/authserver/auth/AuthControllerTest.java`:
```java
package com.platform.authserver.auth;

import com.platform.authserver.jwt.JwtService;
import com.platform.authserver.token.CookieFactory;
import com.platform.authserver.token.RefreshTokenService;
import com.platform.authserver.user.User;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.util.ReflectionTestUtils;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import jakarta.servlet.http.Cookie;
import java.util.List;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.when;
import static org.springframework.security.test.web.servlet.setup.SecurityMockMvcConfigurers.springSecurity;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@ActiveProfiles("test")
class AuthControllerTest {

    @Autowired WebApplicationContext context;
    @MockBean RefreshTokenService refreshTokenService;
    MockMvc mvc;

    @BeforeEach
    void setup() {
        mvc = MockMvcBuilders.webAppContextSetup(context).apply(springSecurity()).build();
    }

    @Test
    void refreshReturnsAccessTokenAndRotatesCookie() throws Exception {
        User user = new User("kc-1");
        ReflectionTestUtils.setField(user, "id", 42L);
        user.setEmail("a@b.com");
        user.setName("Alice");
        user.setRoles(List.of("USER"));
        user.setProvider("GOOGLE");
        when(refreshTokenService.rotate(eq("raw-rt")))
                .thenReturn(new RefreshTokenService.Rotated(user, "new-rt", "kc-id"));

        mvc.perform(post("/api/auth/refresh").cookie(new Cookie("refresh_token", "raw-rt")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.accessToken").isNotEmpty())
                .andExpect(header().exists("Set-Cookie"));
    }

    @Test
    void refreshWithoutCookieIsUnauthorized() throws Exception {
        mvc.perform(post("/api/auth/refresh")).andExpect(status().isUnauthorized());
    }

    @Test
    void logoutReturnsKeycloakLogoutUrlAndClearsCookie() throws Exception {
        when(refreshTokenService.revokeFamilyByRawToken(eq("raw-rt"))).thenReturn("kc-id-token");

        mvc.perform(post("/api/auth/logout").cookie(new Cookie("refresh_token", "raw-rt")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.keycloakLogoutUrl").value(org.hamcrest.Matchers.containsString("id_token_hint=kc-id-token")))
                .andExpect(header().exists("Set-Cookie"));
    }
}
```
> `MeControllerTest`/`AuthControllerTest`는 `@SpringBootTest`라 JwtService/JwtKeyProvider 등 실제 빈을 쓴다. `refreshTokenService`만 목으로 대체.

- [ ] **Step 6: 테스트 실행 → 실패 확인**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test --tests '*AuthControllerTest*'`
Expected: FAIL — `/api/auth/refresh`/`/api/auth/logout` 없음.

- [ ] **Step 7: AuthController 구현**

`auth-server/src/main/java/com/platform/authserver/auth/AuthController.java`:
```java
package com.platform.authserver.auth;

import com.platform.authserver.jwt.JwtService;
import com.platform.authserver.token.CookieFactory;
import com.platform.authserver.token.RefreshTokenService;
import com.platform.authserver.token.ReuseDetectedException;
import com.platform.authserver.user.User;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpHeaders;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final RefreshTokenService refreshTokenService;
    private final JwtService jwtService;
    private final CookieFactory cookieFactory;
    private final String issuerUri;     // Keycloak realm issuer
    private final String frontendUrl;

    public AuthController(RefreshTokenService refreshTokenService,
                          JwtService jwtService,
                          CookieFactory cookieFactory,
                          @Value("${spring.security.oauth2.client.provider.keycloak.issuer-uri}") String issuerUri,
                          @Value("${platform.frontend-url}") String frontendUrl) {
        this.refreshTokenService = refreshTokenService;
        this.jwtService = jwtService;
        this.cookieFactory = cookieFactory;
        this.issuerUri = issuerUri;
        this.frontendUrl = frontendUrl;
    }

    @PostMapping("/refresh")
    public ResponseEntity<?> refresh(HttpServletRequest request) {
        String raw = readCookie(request);
        if (raw == null) {
            return ResponseEntity.status(401).body(Map.of("error", "no_refresh_token"));
        }
        try {
            RefreshTokenService.Rotated rotated = refreshTokenService.rotate(raw);
            User user = rotated.user();
            String accessToken = jwtService.issueAccessToken(
                    user.getId(), user.getEmail(), user.getName(), user.getRoles(), user.getProvider());

            Map<String, Object> body = new HashMap<>();
            body.put("accessToken", accessToken);
            return ResponseEntity.ok()
                    .header(HttpHeaders.SET_COOKIE, cookieFactory.refreshCookie(rotated.newRawToken()).toString())
                    .body(body);
        } catch (ReuseDetectedException | IllegalArgumentException e) {
            return ResponseEntity.status(401)
                    .header(HttpHeaders.SET_COOKIE, cookieFactory.deleteCookie().toString())
                    .body(Map.of("error", "invalid_refresh_token"));
        }
    }

    @PostMapping("/logout")
    public ResponseEntity<?> logout(HttpServletRequest request) {
        String raw = readCookie(request);
        String kcIdToken = refreshTokenService.revokeFamilyByRawToken(raw);

        String logoutUrl = null;
        if (kcIdToken != null) {
            logoutUrl = issuerUri + "/protocol/openid-connect/logout"
                    + "?id_token_hint=" + kcIdToken
                    + "&post_logout_redirect_uri=" + URLEncoder.encode(frontendUrl + "/login", StandardCharsets.UTF_8);
        }
        Map<String, Object> body = new HashMap<>();
        body.put("keycloakLogoutUrl", logoutUrl);
        return ResponseEntity.ok()
                .header(HttpHeaders.SET_COOKIE, cookieFactory.deleteCookie().toString())
                .body(body);
    }

    private String readCookie(HttpServletRequest request) {
        if (request.getCookies() == null) {
            return null;
        }
        for (var c : request.getCookies()) {
            if (CookieFactory.COOKIE_NAME.equals(c.getName())) {
                return c.getValue();
            }
        }
        return null;
    }
}
```

- [ ] **Step 8: 전체 테스트 실행 → 통과 확인**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat test`
Expected: PASS — 전체.

- [ ] **Step 9: Commit**

```bash
cd auth-server
git add .
git commit -m "feat(auth-server): refresh/logout/me 엔드포인트

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 8: 백엔드 수동 통합 검증 (Keycloak 로컬 계정)

**Files:** (코드 변경 없음 — 실행 검증)

**Interfaces:**
- Consumes: Task 1(Keycloak up) + Task 2~7(auth-server).

- [ ] **Step 1: Keycloak 기동 확인**

Run: `cd infra/keycloak; docker compose up -d`
`http://localhost:8080/realms/sso-demo/.well-known/openid-configuration` 200 확인.

- [ ] **Step 2: auth-server 기동**

Run: `cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat bootRun`
로그에 `Started AuthServerApplication`, 포트 9000 확인.

- [ ] **Step 3: JWKS 확인**

브라우저: `http://localhost:9000/.well-known/jwks.json`
Expected: `{"keys":[{"kty":"RSA",...}]}` (공개키만).

- [ ] **Step 4: 로그인 플로우 수동 확인**

브라우저: `http://localhost:9000/oauth2/authorization/keycloak`
→ Keycloak 로그인 화면 → `alice`/`alice` 로그인 → `http://localhost:5173/app` 로 리다이렉트(프론트 미기동이면 연결 실패 화면이지만 URL은 /app 이어야 함). 개발자도구 Application > Cookies(localhost:9000)에 `refresh_token` HttpOnly 쿠키 존재 확인.

- [ ] **Step 5: refresh 수동 확인 (쿠키 사용)**

개발자도구 콘솔(localhost 도메인)에서:
```js
await fetch('http://localhost:9000/api/auth/refresh', { method:'POST', credentials:'include' }).then(r=>r.json())
```
Expected: `{ accessToken: "eyJ..." }`. 그 토큰으로 `/api/me` 확인:
```js
const t = (await fetch('http://localhost:9000/api/auth/refresh',{method:'POST',credentials:'include'}).then(r=>r.json())).accessToken;
await fetch('http://localhost:9000/api/me',{headers:{Authorization:'Bearer '+t}}).then(r=>r.json())
```
Expected: `{ sub, email:"alice@demo.com", name, provider:"KEYCLOAK", role:"USER" }`.

- [ ] **Step 6: 결과 기록 (커밋 불필요)**

검증 통과 시 다음 Task로. 실패 시 systematic-debugging으로 원인 추적.

---

## Task 9: myFront auth 모듈을 Keycloak 리다이렉트로 전환

**Files:**
- Modify: `myFront/.env`
- Modify: `myFront/src/auth/types.ts`
- Modify: `myFront/src/auth/client.ts`
- Modify: `myFront/src/auth/index.ts`
- Modify: `myFront/src/auth/AuthContext.tsx`

**Interfaces:**
- Produces: `AuthClient`에서 `login`/`signup` 제거, `loginUrl()`/`googleLoginUrl()` 유지·추가, `logout()`이 Keycloak 로그아웃 URL로 전체 페이지 이동. `useAuth()` → `{ user, isAuthenticated, loading, logout }`.

- [ ] **Step 1: .env 변경**

`myFront/.env` 전체를:
```
VITE_API_BASE=http://localhost:9000
```

- [ ] **Step 2: types.ts — AuthClient 인터페이스 수정**

`myFront/src/auth/types.ts`의 `AuthClient` 인터페이스를 아래로 교체(메서드 `login`/`signup` 제거, `loginUrl` 추가). `AppUser`/`AuthClientConfig`/`SignupError`는 유지하되 `SignupError`는 더 안 쓰므로 제거 가능 — 아래에서 export도 함께 정리한다:
```ts
export interface AuthClient {
  getAccessToken(): string | null;
  setAccessToken(token: string | null): void;
  /** 백엔드 OIDC 로그인 시작 URL(Keycloak로 리다이렉트). */
  loginUrl(): string;
  /** 구글로 바로 가는 로그인 URL(Keycloak kc_idp_hint=google). */
  googleLoginUrl(): string;
  /** Authorization 헤더 자동 첨부 + 401 시 1회 refresh 후 재시도하는 fetch. */
  apiFetch(path: string, init?: RequestInit): Promise<Response>;
  /** RT 쿠키로 새 AT 획득(동시 호출은 1요청으로 합침). 성공 시 true. */
  tryRefresh(): Promise<boolean>;
  /** 현재 AT 로 사용자 정보 조회. */
  fetchMe(): Promise<AppUser>;
  /** 백엔드 로그아웃 후 AT 비움. Keycloak 로그아웃 URL이 있으면 그쪽으로 이동. */
  logout(): Promise<void>;
}
```
그리고 파일 하단의 `SignupError` 클래스는 삭제한다(더 이상 사용 안 함).

- [ ] **Step 3: client.ts — login/signup 제거, loginUrl 추가, logout 수정**

`myFront/src/auth/client.ts`에서:
- import 줄을 `import type { AppUser, AuthClient, AuthClientConfig } from './types';` 로 변경(`SignupError` import 삭제).
- `googleLoginUrl` 함수를 아래로 교체하고 `loginUrl` 추가:
```ts
  function loginUrl(): string {
    return `${baseUrl}/oauth2/authorization/keycloak`;
  }

  function googleLoginUrl(): string {
    return `${baseUrl}/oauth2/authorization/keycloak?kc_idp_hint=google`;
  }
```
- `login`/`signup` 함수 정의를 **삭제**한다.
- `logout`을 아래로 교체:
```ts
  async function logout(): Promise<void> {
    let keycloakLogoutUrl: string | null = null;
    try {
      const res = await apiFetch('/api/auth/logout', { method: 'POST' });
      if (res.ok) {
        const body = await res.json();
        keycloakLogoutUrl =
          typeof body.keycloakLogoutUrl === 'string' ? body.keycloakLogoutUrl : null;
      }
    } finally {
      setAccessToken(null);
    }
    // Keycloak SSO 세션까지 종료하려면 end_session 으로 전체 페이지 이동.
    if (keycloakLogoutUrl) {
      window.location.href = keycloakLogoutUrl;
    }
  }
```
- 맨 아래 `return { ... }` 객체에서 `login, signup` 을 빼고 `loginUrl` 을 추가:
```ts
  return {
    getAccessToken,
    setAccessToken,
    loginUrl,
    googleLoginUrl,
    apiFetch,
    tryRefresh,
    fetchMe,
    logout,
  };
```
- 파일 하단의 `readErrorMessage` 헬퍼는 더 이상 쓰이지 않으면 삭제한다(signup에서만 썼음).

- [ ] **Step 4: index.ts — export 정리**

`myFront/src/auth/index.ts`를 아래로 교체:
```ts
// auth 모듈 공개 표면.
export { AuthProvider, useAuth } from './AuthContext';
export { default as ProtectedRoute } from './ProtectedRoute';
export { default as GuestRoute } from './GuestRoute';
export { createAuthClient } from './client';
export { authClient } from './authClient';
export type { AppUser, AuthClient, AuthClientConfig } from './types';

import { authClient } from './authClient';
/** LoginPage 의 "로그인" 버튼이 쓰는 편의 함수. */
export const loginUrl = () => authClient.loginUrl();
/** LoginPage 의 "Google 로그인" 버튼이 쓰는 편의 함수. */
export const googleLoginUrl = () => authClient.googleLoginUrl();
```

- [ ] **Step 5: AuthContext.tsx — loginWithPassword/signUp 제거**

`myFront/src/auth/AuthContext.tsx`에서:
- `AuthContextValue` 인터페이스에서 `loginWithPassword`/`signUp` 줄 삭제.
- `loginWithPassword`/`signUp` `useCallback` 정의 삭제.
- `value` useMemo의 객체와 deps에서 `loginWithPassword`/`signUp` 제거. 결과:
```tsx
  const value = React.useMemo<AuthContextValue>(
    () => ({
      user,
      isAuthenticated: !!user,
      loading,
      logout,
    }),
    [user, loading, logout],
  );
```
- `AuthContextValue`는 최종적으로:
```tsx
interface AuthContextValue {
  user: AppUser | null;
  isAuthenticated: boolean;
  loading: boolean;
  logout: () => Promise<void>;
}
```

- [ ] **Step 6: 타입체크 (이 시점엔 LoginPage/SignUpPage가 깨짐 — 다음 Task에서 수정)**

Run: `cd myFront; npm run build`
Expected: FAIL — `LoginPage.tsx`(loginWithPassword 없음), `SignUpPage.tsx`(signUp/SignupError 없음), `main.tsx`(SignUpPage) 타입 에러. 다음 Task에서 해소. (이 단계는 커밋하지 않고 Task 10과 함께 커밋)

---

## Task 10: LoginPage 리다이렉트 전환 + SignUpPage/라우트 제거

**Files:**
- Modify: `myFront/src/app/pages/LoginPage.tsx`
- Delete: `myFront/src/app/pages/SignUpPage.tsx`
- Modify: `myFront/src/main.tsx`

**Interfaces:**
- Consumes: `loginUrl`/`googleLoginUrl` (Task 9), `useAuth` (이제 폼 메서드 없음).
- Produces: `/login`은 리다이렉트 버튼 2개. `/register` 라우트 제거.

- [ ] **Step 1: LoginPage.tsx 교체**

`myFront/src/app/pages/LoginPage.tsx` 전체를 아래로 교체:
```tsx
import * as React from 'react';
import Box from '@mui/material/Box';
import Button from '@mui/material/Button';
import CssBaseline from '@mui/material/CssBaseline';
import Divider from '@mui/material/Divider';
import Typography from '@mui/material/Typography';
import Stack from '@mui/material/Stack';
import MuiCard from '@mui/material/Card';
import { styled } from '@mui/material/styles';
import AppTheme from '../../context/templates/shared-theme/AppTheme';
import ColorModeSelect from '../../context/templates/shared-theme/ColorModeSelect';
import {
  GoogleIcon,
  SitemarkIcon,
} from '../../context/templates/sign-in/components/CustomIcons';
import { loginUrl, googleLoginUrl } from '../../auth';

// Keycloak 기반 로그인. 버튼을 누르면 auth-server(:9000)의 OIDC 시작점으로 리다이렉트되고,
// Keycloak 화면에서 계정/구글 로그인 후 백엔드가 /app 으로 되돌려보낸다.

const Card = styled(MuiCard)(({ theme }) => ({
  display: 'flex',
  flexDirection: 'column',
  alignSelf: 'center',
  width: '100%',
  padding: theme.spacing(4),
  gap: theme.spacing(2),
  margin: 'auto',
  [theme.breakpoints.up('sm')]: {
    maxWidth: '450px',
  },
  boxShadow:
    'hsla(220, 30%, 5%, 0.05) 0px 5px 15px 0px, hsla(220, 25%, 10%, 0.05) 0px 15px 35px -5px',
  ...theme.applyStyles('dark', {
    boxShadow:
      'hsla(220, 30%, 5%, 0.5) 0px 5px 15px 0px, hsla(220, 25%, 10%, 0.08) 0px 15px 35px -5px',
  }),
}));

const LoginContainer = styled(Stack)(({ theme }) => ({
  height: '100dvh',
  minHeight: '100%',
  padding: theme.spacing(2),
  [theme.breakpoints.up('sm')]: {
    padding: theme.spacing(4),
  },
  '&::before': {
    content: '""',
    display: 'block',
    position: 'absolute',
    zIndex: -1,
    inset: 0,
    backgroundImage:
      'radial-gradient(ellipse at 50% 50%, hsl(210, 100%, 97%), hsl(0, 0%, 100%))',
    backgroundRepeat: 'no-repeat',
    ...theme.applyStyles('dark', {
      backgroundImage:
        'radial-gradient(at 50% 50%, hsla(210, 100%, 16%, 0.5), hsl(220, 30%, 5%))',
    }),
  },
}));

export default function LoginPage(props: { disableCustomTheme?: boolean }) {
  return (
    <AppTheme {...props}>
      <CssBaseline enableColorScheme />
      <ColorModeSelect sx={{ position: 'fixed', top: '1rem', right: '1rem' }} />
      <LoginContainer direction="column" sx={{ justifyContent: 'space-between' }}>
        <Card variant="outlined">
          <SitemarkIcon />
          <Typography
            component="h1"
            variant="h4"
            sx={{ width: '100%', fontSize: 'clamp(2rem, 10vw, 2.15rem)' }}
          >
            로그인
          </Typography>
          <Typography sx={{ color: 'text.secondary' }}>
            계속하려면 로그인하세요. Keycloak 계정 또는 Google 계정을 사용할 수 있습니다.
          </Typography>
          <Box sx={{ display: 'flex', flexDirection: 'column', gap: 2 }}>
            <Button
              fullWidth
              variant="contained"
              onClick={() => {
                window.location.href = loginUrl();
              }}
            >
              로그인
            </Button>
            <Divider>또는</Divider>
            <Button
              fullWidth
              variant="outlined"
              onClick={() => {
                window.location.href = googleLoginUrl();
              }}
              startIcon={<GoogleIcon />}
            >
              Google 계정으로 로그인
            </Button>
          </Box>
        </Card>
      </LoginContainer>
    </AppTheme>
  );
}
```

- [ ] **Step 2: SignUpPage.tsx 삭제**

Run (PowerShell): `Remove-Item myFront\src\app\pages\SignUpPage.tsx`

- [ ] **Step 3: main.tsx — SignUpPage import 및 /register 라우트 제거**

`myFront/src/main.tsx`에서:
- `import SignUpPage from './app/pages/SignUpPage';` 줄 삭제.
- `/register` 라우트 객체(아래 블록)를 통째로 삭제:
```tsx
  {
    path: '/register',
    element: (
      <GuestRoute fallback={<AuthLoadingScreen />}>
        <SignUpPage />
      </GuestRoute>
    ),
    errorElement: <RouteErrorPage />,
  },
```

- [ ] **Step 4: 타입체크/빌드 통과 확인**

Run: `cd myFront; npm run build`
Expected: PASS — `tsc -b` 타입 에러 없음, vite build 성공.
> 만약 `Home.tsx` 등 다른 곳에서 `/register` 링크나 `loginWithPassword`를 참조하면 에러가 난다. 그럴 경우 해당 링크를 `/login`으로 바꾸거나 참조를 제거한다(빌드 에러 메시지를 따라 수정).

- [ ] **Step 5: Commit (Task 9 변경분과 함께)**

```bash
cd myFront
git add -A
git commit -m "feat(myFront): 인증을 Keycloak OIDC 리다이렉트로 전환 (비번/회원가입 폼 제거)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Task 11: 전체 E2E 수동 검증 + 구글 브로커링 설정 문서

**Files:**
- Create: `infra/keycloak/GOOGLE_SETUP.md`

**Interfaces:**
- Consumes: 전체 스택.

- [ ] **Step 1: 구글 브로커링 설정 문서 작성**

`infra/keycloak/GOOGLE_SETUP.md`:
```markdown
# Google Identity Brokering 설정 (수동 1회)

1. Google Cloud Console > APIs & Services > Credentials > OAuth 2.0 Client ID (Web).
   - 승인된 리디렉션 URI:
     `http://localhost:8080/realms/sso-demo/broker/google/endpoint`
2. 발급된 Client ID / Secret 을 `infra/keycloak/.env` 에 저장(.env.example 참고).
3. Keycloak 관리 콘솔(http://localhost:8080, admin/admin)
   > sso-demo realm > Identity providers > Add provider > Google
   > Client ID / Client Secret 입력 > Save.
4. 로그아웃 상태에서 `http://localhost:5173/login` > "Google 계정으로 로그인"
   (또는 `http://localhost:9000/oauth2/authorization/keycloak?kc_idp_hint=google`)
   클릭 시 구글 로그인으로 직행한다.
```

- [ ] **Step 2: 3개 프로세스 기동**

```
# 1) Keycloak + Postgres
cd infra/keycloak; docker compose up -d
# 2) auth-server
cd auth-server; $env:JAVA_HOME='C:\Program Files\Java\jdk-24'; .\gradlew.bat bootRun
# 3) myFront
cd myFront; npm install; npm run dev
```

- [ ] **Step 3: 로컬 계정 로그인 E2E**

브라우저 `http://localhost:5173/login` → "로그인" → Keycloak 화면 → `alice`/`alice`
→ `http://localhost:5173/app` 대시보드 진입, 사용자 정보 표시(이메일 alice@demo.com).
Expected: 로그인 화면 없이 /app 진입, 새로고침해도 세션 유지(silent refresh).

- [ ] **Step 4: 로그아웃 E2E**

대시보드에서 로그아웃 → Keycloak end_session 경유 → `/login` 복귀.
다시 "로그인" 클릭 시 Keycloak 로그인 화면이 **다시** 노출(SSO 세션 종료 확인).
Expected: 로그아웃 후 재로그인 시 자격증명 재입력 필요.

- [ ] **Step 5: (구글 설정했다면) 구글 로그인 E2E**

`http://localhost:5173/login` → "Google 계정으로 로그인" → 구글 인증 → `/app`.
Expected: federated 사용자로 진입, `/api/me`의 provider="GOOGLE".

- [ ] **Step 6: 보호 API 음성 테스트**

로그아웃 상태에서 콘솔:
```js
await fetch('http://localhost:9000/api/me', { credentials:'include' }).then(r=>r.status)
```
Expected: 401.

- [ ] **Step 7: Commit**

```bash
git add infra/keycloak/GOOGLE_SETUP.md
git commit -m "docs(infra): 구글 Identity Brokering 설정 가이드

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Self-Review

**Spec coverage 점검:**
- 인증 모델 B (Keycloak 위임 + 자체 JWT) → Task 6(OIDC 로그인+성공핸들러), Task 3(JWT 발급). ✓
- 자체 JWT (i) Nimbus RS256 + JWKS → Task 3. ✓
- 구글 OIDC (Keycloak 브로커링) → Task 1(realm), Task 11(설정 문서/E2E). ✓
- SSO 토대(issuer/JWKS 검증, 미래 서비스 부착) → Task 3 JWKS, application.yml issuer. ✓
- 자체 refresh 회전 + 재사용 탐지 → Task 5. ✓
- DB + JIT user → Task 2(스키마), Task 4(JIT). ✓
- 로그아웃 = 로컬 + Keycloak end_session → Task 7(logout), Task 9(프론트 redirect). ✓
- myFront 변경(리다이렉트 로그인, 폼/회원가입 제거, .env) → Task 9, 10. ✓
- 콜백 페이지 불필요(성공핸들러가 /app 리다이렉트 + silent refresh) → Task 6, 기존 AuthProvider. ✓
- 테스트(JWT/refresh/JIT/me) → Task 3,4,5,7. ✓
- CORS → Task 6 SecurityConfig. ✓

**Placeholder scan:** 모든 step에 실제 코드/명령/기대출력 포함. "TODO/적절히 처리" 없음. ✓

**Type consistency:**
- `RefreshTokenService.Issued(rawToken, kcIdToken)` / `Rotated(user, newRawToken, kcIdToken)` — Task 5 정의, Task 6(issue)·Task 7(rotate)에서 동일 시그니처 사용. ✓
- `issueAccessToken(long, String, String, List<String>, String)` — Task 3 정의, Task 7에서 동일 호출. ✓
- `CookieFactory.COOKIE_NAME="refresh_token"` — Task 6 정의, Task 7 readCookie / Task 9 쿠키명 일치. ✓
- `/api/auth/refresh` 응답 `{accessToken}`, `/api/auth/logout` 응답 `{keycloakLogoutUrl}` — Task 7 produces, Task 9 client.ts 소비 일치. ✓
- `OidcClaims.roles/provider` — Task 6 정의·사용 일치. ✓

이슈 없음.

---

## Execution Handoff

(writing-plans 종료 후 채택)

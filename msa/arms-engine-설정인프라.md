# ARMS 엔진 — 설정 / 인프라

> 상위 노트: [[arms-engine-분석]] · 관련: [[arms-engine-아키텍처]] · CI/CD는 [[phase1.5-cicd]]

## 빌드 — `build.gradle`
- **Spring Boot 3.4.4 · Java 21(toolchain) · Spring Cloud 2024.0.1**.
- 플러그인: `org.sonarqube`(정적분석), `bmuschko.docker-remote-api`(이미지 빌드/푸시), `maven-publish`(Nexus 배포), `dependency-license-report`(라이선스 인벤토리), `project-report`/`build-dashboard`.
- 사내 Nexus 저장소(`www.313.co.kr/nexus`, `allowInsecureProtocol true`)에서 의존성·배포.
- 주요 의존성 군:
  - 검색: `spring-data-opensearch`, `opensearch-java/rest-client`, `elasticsearch-java 8.11`
  - ALM: Atlassian/`jira-rest-java-client 5.1.6`, `redmine-java-api 4.0.0.rc4`
  - 통신: `spring-cloud-starter-openfeign` + `feign-hc5`(Apache HttpClient5로 교체)
  - 기타: `modelmapper`, `guava`, `slack-api-client`, `kaptcha`(캡차), `springdoc-openapi`(Swagger), `joda-time`

### 자동 버전 채번 (특이 패턴) ⭐
`ext` 블록에서 Nexus의 `maven-metadata.xml`을 `wget`/`XmlSlurper`로 파싱해 **최신 버전 +1로 patchVersion 자동 증가**.
```groovy
majorVersion=25; minorVersion=10
// metadata.xml의 latest를 읽어 major/minor 비교 후 patch 자동 산정
version = "${majorVersion}.${minorVersion}.${patchVersion}"
```
> Windows는 빌드 미지원(`*** Windows is not support build`), Mac/Linux 분기. → CI는 Linux 러너 전제.
> **반면교사**: 빌드 시 외부 네트워크(wget)에 의존하면 오프라인/CI 캐시에서 깨짐. MSA 템플릿에선 버전을 git tag/CI 변수로 빼는 게 안전.

## 컨테이너 — `Dockerfile` + `docker-entrypoint.sh`
- 베이스 `openjdk:21-jdk`, 단일 스테이지(빌드된 jar를 COPY). Gradle 태스크 체인:
  `prepareDockerContext → buildDockerImage(DockerBuildImage) → pushDockerImage` (레지스트리 `313.co.kr:5550`).
- 엔트리포인트 스크립트에서 **JVM 튜닝을 환경변수로 오버라이드 가능**하게 구성:
```sh
GC_OPTS=${GC_OPTS:="-XX:+UseNUMA -XX:+UseG1GC"}
MEM_OPTS=${MEM_OPTS:="-Xms2048m -Xmx2048m"}
NET_OPTS="-Dsun.net.inetaddr.ttl=0 ... -Djava.net.preferIPv4Stack=true"
SPRING_PROFILES_ACTIVE=${SPRING_PROFILES_ACTIVE:-live}
exec java -Duser.timezone=Asia/Seoul $GC_OPTS -jar $JVM_OPTS \
     -Dspring.profiles.active=$SPRING_PROFILES_ACTIVE javaServiceTreeFramework.jar "$@"
```
- Elastic APM agent를 `-javaagent`로 주입(프로파일별 service_name/server_urls). → 관측성은 ELK/APM 스택 전제.
**가져올 때**: JVM 옵션을 env로 외부화 + 타임존 고정 + IPv4 강제는 운영 컨테이너 기본기. 다만 **멀티스테이지 빌드 아님** → [[phase1.5-cicd]]의 멀티스테이지로 개선 대상.

## 프로파일 / 외부 설정
- `application.yml`은 앱 이름만(`javaServiceTreeFrameworkEngineFire`), 환경별은 `application-{dev,stg,live,vpn}.yml`.
- **Spring Cloud Config Server** 사용: `spring.config.import: optional:configserver:http://www.313.co.kr:33133` → 설정 외부화/중앙화.
- Actuator 노출: `refresh, env, health, beans, httptrace` (`/refresh`로 런타임 설정 리로드).
- 신규(미커밋) 프로파일 `application-vpn.yml`, `logback-vpn.xml` 존재 — VPN 환경 대응 추가 중.

## OpenSearch 클라이언트 — `OpensearchClientConfig`
```java
@EnableElasticsearchRepositories(basePackages={"com.arms"}, repositoryBaseClass = EsCommonRepositoryImpl.class)
```
핵심:
- **Repository base class를 자체 구현(`EsCommonRepositoryImpl`)으로 교체** → 모든 ES repo가 공통 운영 메서드 상속(=[[arms-engine-재사용패턴]] #8).
- connect/socket timeout 60s, keep-alive 180s, `@PreDestroy`로 클라이언트 정리.
- `refreshPolicy() = IMMEDIATE` — 쓰기 즉시 검색 반영(정합성↑, 처리량↓ 트레이드오프).

## Feign(HTTP 클라이언트) — `OpenFeignConfig` / `HttpClient5FeignConfig`
- `@EnableFeignClients("com.arms.api.util.msa_communicate")`로 통신 패키지만 스캔.
- 기본 Feign 클라이언트를 **Apache HttpClient5(`ApacheHttp5Client`)로 교체** — `@ConditionalOnProperty(feign.httpclient.hc5.enabled=true)` 가드로 안전하게 활성화.
- `@FeignClient(name="backend-core", url="${arms.backend-core.url}")` — URL을 프로퍼티로 외부화.

## 로깅 — `logback-spring.xml` + `logback-{env}.xml`
- 콘솔 패턴에 **MDC `%X{user} %X{function}`** 포함 → 요청 컨텍스트 추적.
- `RollingFileAppender` + `SizeAndTimeBasedFNATP`: 50MB/파일, 30일 보관, `.gz` 압축.
- `springProfile`로 환경별 로그레벨 분기(dev/test).
> ⚠️ 현재 로그 경로 `${app.logPathPrefix}`가 안 풀려서 `app.logPathPrefix_IS_UNDEFINED/` 디렉터리에 로그가 쌓이고 그게 git에 커밋되는 사고가 보임(레포 status 참고). **프로퍼티 미정의 시 fallback 경로**를 주거나 `.gitignore` 필요.

## 기타 인프라 빈
| 빈 | 역할 |
|---|---|
| `SlackConfig` / `SlackNotificationService` | 장애·이벤트 Slack 채널 통보 |
| `OpenApiConfig` / `SwaggerUIConfiguration` | springdoc 기반 API 문서 |
| `CaptchaConfig` | kaptcha 캡차 생성 |
| `CustomElasticsearchHealthIndicator` | Actuator health에 ES 상태 편입 |
| `ElasticsearchProperties` / `ElasticsearchIndexNameConfig` | ES 접속·인덱스명 프로퍼티 바인딩 |

---

## 운영 체크리스트로 정리 (내 템플릿용)
- [x] JVM 옵션 env 외부화, 타임존/IPv4 고정 → 차용
- [x] 설정 중앙화(Config Server) + actuator `/refresh` → 차용
- [x] Repository base class 커스터마이즈로 공통 연산 주입 → 차용
- [ ] **멀티스테이지 Docker 빌드로 전환** (현재 단일 스테이지)
- [ ] **빌드의 외부 네트워크 의존 제거**(메타데이터 wget) → CI 변수/태그 기반 버전
- [ ] 로그 경로 프로퍼티 fallback + 로그 산출물 `.gitignore`

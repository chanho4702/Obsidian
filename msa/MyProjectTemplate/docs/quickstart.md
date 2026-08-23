# 빠른 시작

## 1. 요구 사항

- JDK 21
- Docker Engine과 Docker Compose v2
- Node.js 22 이상: 구성 마법사를 사용할 때만 필요
- PowerShell 7 또는 Bash

## 2. 구성을 선택한다

```bash
cd tools/configurator
npm install
npm run dev
```

화면에서 목표 TPS와 가용성, 데이터 계층, 선택 기능을 정하고 `template-config.json`을 내려받는다. 파일을 저장소 루트에 두면 생성기와 검증 스크립트가 같은 선택을 사용한다.

```powershell
./tools/apply-config.ps1
```

이 명령은 `generated/`에 Compose 실행 명령과 애플리케이션 feature 환경변수를 만든다.

## 3. 로컬 인프라를 켠다

PostgreSQL만:

```bash
docker compose --env-file infra/.env.versions -f infra/compose.yml up -d postgres
```

Redis와 Kafka 추가:

```bash
docker compose --env-file infra/.env.versions -f infra/compose.yml --profile cache --profile messaging up -d
```

검색까지 전체 실행:

```bash
docker compose --env-file infra/.env.versions -f infra/compose.yml --profile cache --profile messaging --profile search up -d
```

profile은 기능 그룹이다. 사용하지 않는 인프라는 생성되지 않는다.

## 4. 샘플 서비스를 실행한다

```bash
./gradlew :services:sample-service:bootRun --args='--spring.profiles.active=local'
```

정상 여부:

```bash
curl http://localhost:8081/actuator/health
curl http://localhost:8081/api/v1/items
```

게이트웨이까지 실행하려면 다른 터미널에서 다음 명령을 실행하고 `http://localhost:8080/api/v1/items`를 호출한다.

```bash
./gradlew :services:gateway-service:bootRun
```

로컬 기본값은 인증을 비활성화한다. `dev`와 `prod`는 각 환경 파일의 계약에 따라 OIDC와 외부 인프라 주소를 요구한다.

## 5. 새 서비스를 만든다

```powershell
./tools/new-service.ps1 -Name order-service -BasePackage com.acme.order
```

생성된 서비스는 자동으로 Gradle 멀티프로젝트에 포함된다. 필요한 starter만 `build.gradle`에 추가한다.

## 6. 용량 기준선을 남긴다

```bash
k6 run -e BASE_URL=http://localhost:8081 load-tests/smoke.js
k6 run -e BASE_URL=http://localhost:8081 -e TARGET_TPS=100 load-tests/capacity.js
```

결과는 `load-tests/results/<날짜>-<환경>.md`에 하드웨어와 설정을 함께 기록한다. TPS 숫자만 단독으로 문서에 옮기지 않는다.

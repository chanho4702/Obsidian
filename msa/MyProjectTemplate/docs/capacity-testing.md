# 처리량과 가용성 단계별 가이드

이 문서는 실제 부하 테스트를 처음 실행하는 사람을 위한 실행 안내서다. 명령을 복사하기 전에 반드시 대상과 데이터가 테스트용인지 확인한다.

> 부하 테스트의 `PASS`는 그 실행에 입력한 임계치를 만족했다는 뜻이다. 다른 PC, 운영 환경 또는 다른 데이터 크기의 성능을 보장하지 않는다.

## 1. 먼저 이해할 개념

| 용어 | 쉬운 설명 |
|---|---|
| TPS | 1초 동안 목표로 보내는 요청 수 |
| VU | k6가 사용하는 가상 사용자 |
| p95 | 전체 요청의 95%가 이 시간 안에 끝났다는 뜻 |
| p99 | 전체 요청의 99%가 이 시간 안에 끝났다는 뜻 |
| 오류율 | HTTP 실패 요청 비율 |
| dropped iteration | 목표 TPS를 만들 VU가 부족해 시작조차 못 한 요청 수 |
| knee point | 트래픽을 올릴 때 지연·오류·자원 포화가 급격히 나빠지기 직전 후보 지점 |
| spike | 짧은 시간에 트래픽을 크게 올렸다가 다시 내리는 시험 |
| soak | 오랜 시간 같은 부하를 유지해 메모리 누수와 누적 지연을 보는 시험 |

TPS 하나만 보면 안 된다. p95/p99, 오류율, dropped iteration, CPU, JVM heap, GC와 DB connection pool을 같은 시간대에 함께 본다.

## 2. 테스트 단계와 목적

1. smoke: 1~5 TPS로 API와 스크립트가 정상인지 확인
2. probe: 짧은 단계 상승으로 실행 환경과 결과 저장을 확인
3. knee: 충분한 단계 유지 시간으로 후보 knee point 탐색
4. spike: baseline의 기본 3배 상승과 복구 확인
5. soak: 기본 4시간 동안 누적 포화 확인
6. failover: 앱 인스턴스, reader 또는 broker를 제거해 실패와 복구 관찰

처음부터 4시간 soak를 실행하지 않는다. smoke→짧은 probe→spike가 정상인 뒤 진행한다.

## 3. 실행 전에 준비할 것

### 3.1 저장소 루트 확인

```powershell
cd D:\MyProjectTemplate
Get-Location
```

### 3.2 테스트 코드 상태 확인

```powershell
git status --short
git rev-parse HEAD
```

공식 기준선은 작업 트리가 깨끗할 때만 측정한다. 변경 파일이 있으면 결과에 기준 SHA와 변경 상태를 별도로 기록하고 `probe`로만 분류한다.

### 3.3 결과 폴더 준비

```powershell
New-Item -ItemType Directory -Force load-tests/results
```

`load-tests/results/`는 Git에서 제외된다. 원본 JSON과 Markdown은 CI artifact 또는 별도 실험 저장소에 보관한다.

### 3.4 서비스와 관측성 실행

[처음부터 따라 하는 로컬 실행 가이드](quickstart.md)의 3~7단계를 먼저 수행한다.

필수 확인:

```powershell
Invoke-RestMethod http://localhost:8081/actuator/health
Invoke-WebRequest http://localhost:8081/actuator/prometheus -UseBasicParsing
```

- health의 `status`가 `UP`
- Prometheus endpoint가 HTTP 200
- <http://localhost:9090/targets>에서 `sample-service`가 `UP`

## 4. k6 실행 방법 선택

### 방법 A — PC에 설치된 k6

```powershell
k6 version
```

명령이 성공하면 문서의 `k6 run` 예제를 바로 사용할 수 있다.

### 방법 B — Docker k6

k6를 PC에 설치하지 않아도 Docker 이미지로 실행할 수 있다. 공식 사용법은 [Grafana k6 실행 문서](https://grafana.com/docs/k6/latest/get-started/running-k6/)를 참고한다.

2026-08-16 로컬 검증에 사용한 실행기는 다음과 같다.

- k6 `v2.2.0`
- 이미지 `grafana/k6@sha256:5221b620a4f874faff6e32ba597aa667c058391fe4898b1c6f6377f062c6cdec`

PowerShell 변수로 공통값을 준비한다.

```powershell
$Repo = (Get-Location).Path
$GitSha = git rev-parse HEAD
$K6Image = 'grafana/k6@sha256:5221b620a4f874faff6e32ba597aa667c058391fe4898b1c6f6377f062c6cdec'
$InstanceProfile = '1 sample-service, JDK 21, Docker Desktop, DB pool 10'
$Dataset = '1 item, read-only GET /api/v1/items'
$Dependencies = 'PostgreSQL 17.6 writer+reader; Prometheus 3.13.1; Grafana 13.1.0'
```

Docker 컨테이너에서 Windows 호스트는 `host.docker.internal`로 접근한다. 이 주소는 k6 입장에서는 `localhost`가 아니므로, 소유한 로컬 대상임을 확인한 뒤 `ALLOW_NON_LOCAL=true`를 전달한다.

## 5. 모든 시나리오가 요구하는 기록값

| 환경변수 | 의미 | 예시 |
|---|---|---|
| `GIT_SHA` | 테스트한 전체 Git SHA | `0123456789ab...` |
| `TEST_ENVIRONMENT` | 서로 구분 가능한 환경 이름 | `local-windows-probe` |
| `INSTANCE_PROFILE` | 앱·JVM·부하 발생기 사양 | `1 app, JDK 21, 2 GiB heap` |
| `DATASET_DESCRIPTION` | 행 수, payload와 요청 조합 | `10k items, read-only` |
| `RESULT_FILE` | 저장소 아래 결과 JSON 경로 | `load-tests/results/knee-local.json` |
| `DEPENDENCY_PROFILE` | DB·Redis·Kafka 등의 구성 | `PostgreSQL writer+reader` |
| `IMAGE_DIGEST` | 컨테이너 앱을 시험한 경우 불변 digest | `sha256:...` |

값을 빠뜨리면 `knee.js`, `spike.js`, `soak.js`는 시작하지 않는다.

## 6. 1단계 — smoke 테스트

목적은 성능 측정이 아니라 API와 k6 연결 확인이다.

PC에 설치된 k6:

```powershell
k6 run -e BASE_URL=http://localhost:8081 load-tests/smoke.js
```

Docker k6:

```powershell
docker run --rm -i `
  --add-host host.docker.internal:host-gateway `
  -v "${Repo}:/workspace" -w /workspace `
  --env BASE_URL=http://host.docker.internal:8081 `
  $K6Image run load-tests/smoke.js
```

정상 기준:

- HTTP check 100%
- 요청 실패 0%
- 서비스 health `UP`

## 7. 2단계 — 짧은 knee probe

각 단계를 30초만 유지해 결과 생성과 낮은 범위의 안정성을 확인한다. 이 실행만으로 knee point를 결정하지 않는다.

```powershell
docker run --rm -i `
  --add-host host.docker.internal:host-gateway `
  -v "${Repo}:/workspace" -w /workspace `
  --env BASE_URL=http://host.docker.internal:8081 `
  --env ALLOW_NON_LOCAL=true `
  --env "GIT_SHA=$GitSha" `
  --env TEST_ENVIRONMENT=local-windows-probe `
  --env "INSTANCE_PROFILE=$InstanceProfile" `
  --env "DATASET_DESCRIPTION=$Dataset" `
  --env "DEPENDENCY_PROFILE=$Dependencies" `
  --env RESULT_FILE=load-tests/results/knee-local-probe.json `
  --env KNEE_RATES=10,25,50,100 `
  --env STAGE_DURATION=30s `
  --env RECOVERY_DURATION=30s `
  $K6Image run load-tests/knee.js
```

`KNEE_RATES`는 낮은 값부터 엄격히 증가하는 양의 정수 목록이어야 한다.

## 8. 3단계 — 정식 knee 탐색

probe가 정상일 때 각 단계를 기본 2분 이상 유지한다.

```powershell
docker run --rm -i `
  --add-host host.docker.internal:host-gateway `
  -v "${Repo}:/workspace" -w /workspace `
  --env BASE_URL=http://host.docker.internal:8081 `
  --env ALLOW_NON_LOCAL=true `
  --env "GIT_SHA=$GitSha" `
  --env TEST_ENVIRONMENT=local-windows-knee `
  --env "INSTANCE_PROFILE=$InstanceProfile" `
  --env "DATASET_DESCRIPTION=$Dataset" `
  --env "DEPENDENCY_PROFILE=$Dependencies" `
  --env RESULT_FILE=load-tests/results/knee-local.json `
  --env KNEE_RATES=25,50,75,100 `
  --env STAGE_DURATION=2m `
  --env RECOVERY_DURATION=2m `
  $K6Image run load-tests/knee.js
```

처음으로 SLO를 벗어나거나 CPU·heap·DB pool이 지속 포화되는 단계의 **직전 단계**를 후보로 검토한다. 시험한 최고 TPS까지 문제가 없었다면 knee point를 찾은 것이 아니라 탐색 범위 안에서 발견하지 못한 것이다.

## 9. 4단계 — 3배 spike와 복구

기본 흐름은 다음과 같다.

```text
baseline 2분 → 30초 상승 → 3배 TPS 30초 → 30초 하강 → baseline 2분
```

```powershell
docker run --rm -i `
  --add-host host.docker.internal:host-gateway `
  -v "${Repo}:/workspace" -w /workspace `
  --env BASE_URL=http://host.docker.internal:8081 `
  --env ALLOW_NON_LOCAL=true `
  --env "GIT_SHA=$GitSha" `
  --env TEST_ENVIRONMENT=local-windows-spike `
  --env "INSTANCE_PROFILE=$InstanceProfile" `
  --env "DATASET_DESCRIPTION=$Dataset" `
  --env "DEPENDENCY_PROFILE=$Dependencies" `
  --env RESULT_FILE=load-tests/results/spike-local.json `
  --env BASELINE_TPS=50 `
  --env SPIKE_MULTIPLIER=3 `
  $K6Image run load-tests/spike.js
```

전체 평균만 보지 말고 spike 이후 오류율과 지연이 baseline 수준으로 돌아오는지 Grafana 시간축에서 확인한다.

## 10. 5단계 — soak

먼저 10분 사전 soak로 로그·결과 경로와 PC 절전 설정을 확인한다.

```powershell
docker run --rm -i `
  --add-host host.docker.internal:host-gateway `
  -v "${Repo}:/workspace" -w /workspace `
  --env BASE_URL=http://host.docker.internal:8081 `
  --env ALLOW_NON_LOCAL=true `
  --env "GIT_SHA=$GitSha" `
  --env TEST_ENVIRONMENT=local-windows-soak-preflight `
  --env "INSTANCE_PROFILE=$InstanceProfile" `
  --env "DATASET_DESCRIPTION=$Dataset" `
  --env "DEPENDENCY_PROFILE=$Dependencies" `
  --env RESULT_FILE=load-tests/results/soak-preflight-local.json `
  --env TARGET_TPS=50 `
  --env DURATION=10m `
  $K6Image run load-tests/soak.js
```

사전 soak가 정상이면 `DURATION=4h`와 별도 결과 파일명으로 정식 실행한다. 실행 중에는 PC가 절전되지 않아야 하며 Docker Desktop과 서비스 터미널을 종료하면 안 된다.

soak는 기본적으로 `GET /api/v1/items`만 보낸다. 현실적인 쓰기 비율을 섞으려면 `WRITE_RATIO`(0보다 크고 1보다 작은 값)를 지정한다. 이 비율만큼의 요청은 `POST /api/v1/items`로 새 항목을 생성하므로, 데이터셋 크기가 실행 시간 동안 계속 늘어난다.

```powershell
--env WRITE_RATIO=0.1 `
```

확인할 것:

- JVM heap 사용량이 시간에 따라 계속 증가하는가
- GC pause가 누적되는가
- HikariCP active/pending 연결이 포화되는가
- p95/p99와 오류율이 뒤로 갈수록 나빠지는가
- dropped iteration이 생기는가

## 11. 초기 임계치 바꾸기

저장소 기본 입력은 다음과 같다.

- 오류율 `< 0.1%`
- p95 `< 300ms`
- p99 `< 800ms`
- dropped iteration `= 0`

제품 SLO가 정해지면 실행 시 덮어쓴다.

```powershell
--env MAX_ERROR_RATE=0.005 `
--env P95_MS=450 `
--env P99_MS=900
```

이 값은 제품 보장이 아니라 해당 실행의 합격 조건이다.

## 12. Markdown 결과 보고서 만들기

```powershell
node load-tests/report.js `
  --input load-tests/results/spike-local.json `
  --output load-tests/results/spike-local.md
```

보고서의 주요 항목:

- 실행 SHA와 환경
- 인스턴스·데이터셋·의존성 구성
- 요청 수와 평균 요청률
- 오류율, p50/p95/p99/max
- dropped iteration
- threshold별 PASS/FAIL

`PASS`여도 데이터셋과 환경이 바뀌면 다시 측정해야 한다.

## 13. reader 장애 실험

이 실험은 일부러 장애를 만든다. 개발용 Compose에서만 실행한다. 운영 또는 공유 dev DB에서 실행하지 않는다.

### 준비

- writer와 reader가 모두 `healthy`
- sample-service가 `DB_READER_URL=jdbc:postgresql://localhost:5434/appdb`로 실행 중
- GET 요청이 reader를 사용 중

### 터미널 1 — 90초 동안 20 TPS 실행

soak 명령을 사용하되 다음 값을 준다.

```powershell
--env TEST_ENVIRONMENT=local-reader-failover `
--env RESULT_FILE=load-tests/results/reader-failover-local.json `
--env TARGET_TPS=20 `
--env DURATION=90s
```

### 터미널 2 — 25초 뒤 reader를 15초 중단

```powershell
Start-Sleep -Seconds 25
docker compose --env-file infra/.env.versions -f infra/compose.yml `
  --profile database-ha stop postgres-reader

Start-Sleep -Seconds 15
docker compose --env-file infra/.env.versions -f infra/compose.yml `
  --profile database-ha start postgres-reader
```

복구를 확인한다.

```powershell
docker compose --env-file infra/.env.versions -f infra/compose.yml ps
Invoke-RestMethod http://localhost:8081/actuator/health
Invoke-RestMethod http://localhost:8081/api/v1/items
```

현재 starter의 안전 규칙은 다음과 같다.

- reader URL이 **설정되지 않으면** 시작 시 writer로 fallback
- 실행 중 연결된 reader가 장애 나면 자동으로 writer로 전환하지 않음
- 관리형 reader endpoint나 DB proxy가 연결 전환을 담당하는 운영 구성을 권장

따라서 단일 로컬 reader를 중단하면 threshold `FAIL`이 예상된다. 이것은 스크립트 오류가 아니라 현재 가용성 한계를 측정한 결과다.

## 14. 앱 인스턴스 하나 제거 실험

`capacity-ha` profile은 운영 load balancer가 아니라 이 실험만을 위한 로컬 Nginx proxy다.

```text
k6 → capacity proxy :8084
             ├─ sample-service #1 :8081
             └─ sample-service #2 :8083
```

이 실험은 안전한 GET 요청만 사용한다. POST처럼 부작용이 있는 요청을 proxy가 자동 재시도하면 중복 생성 위험이 있으므로 그대로 적용하지 않는다.

### 14.1 첫 번째 인스턴스 실행

터미널 1:

```powershell
$env:JAVA_HOME='C:\Program Files\Java\jdk-21'
$env:PATH="$env:JAVA_HOME\bin;$env:PATH"
$env:SERVER_PORT='8081'
$env:DB_READER_URL='jdbc:postgresql://localhost:5434/appdb'
./gradlew :services:sample-service:bootRun --args='--spring.profiles.active=local'
```

### 14.2 두 번째 인스턴스 실행

터미널 2:

```powershell
$env:JAVA_HOME='C:\Program Files\Java\jdk-21'
$env:PATH="$env:JAVA_HOME\bin;$env:PATH"
$env:SERVER_PORT='8083'
$env:DB_READER_URL='jdbc:postgresql://localhost:5434/appdb'
./gradlew :services:sample-service:bootRun --args='--spring.profiles.active=local'
```

두 인스턴스를 확인한다.

```powershell
Invoke-RestMethod http://localhost:8081/actuator/health
Invoke-RestMethod http://localhost:8083/actuator/health
```

둘 다 `UP`이어야 한다.

### 14.3 capacity proxy 실행

터미널 3:

```powershell
docker compose --env-file infra/.env.versions -f infra/compose.yml `
  --profile capacity-ha up -d --wait capacity-proxy
```

확인:

```powershell
Invoke-RestMethod http://localhost:8084/proxy-health
Invoke-RestMethod http://localhost:8084/api/v1/items
```

### 14.4 두 인스턴스가 모두 사용되는지 확인

요청을 여러 번 보낸 뒤 proxy 로그를 확인한다.

```powershell
1..10 | ForEach-Object {
  Invoke-RestMethod http://localhost:8084/api/v1/items | Out-Null
}

docker compose --env-file infra/.env.versions -f infra/compose.yml `
  logs --tail 20 capacity-proxy
```

로그의 `upstream=`에 `host.docker.internal:8081`과 `host.docker.internal:8083`이 모두 보여야 한다.

### 14.5 부하 중 두 번째 인스턴스 제거

터미널 4에서 `BASE_URL`을 proxy로 바꾼 90초 soak를 시작한다.

```powershell
--env BASE_URL=http://host.docker.internal:8084 `
--env TEST_ENVIRONMENT=local-app-instance-removal `
--env RESULT_FILE=load-tests/results/app-instance-removal-local.json `
--env TARGET_TPS=50 `
--env DURATION=90s
```

25초 뒤 터미널 3에서 `8083`을 점유한 정확한 프로세스를 확인한다.

```powershell
$SecondInstancePid = (Get-NetTCPConnection -LocalPort 8083 -State Listen).OwningProcess
Get-Process -Id $SecondInstancePid
```

표시된 프로세스가 두 번째 sample-service Java 프로세스인지 확인한 뒤에만 중단한다.

```powershell
Stop-Process -Id $SecondInstancePid
```

테스트 종료 후 확인한다.

```powershell
Invoke-RestMethod http://localhost:8084/api/v1/items
docker compose --env-file infra/.env.versions -f infra/compose.yml `
  logs --tail 100 capacity-proxy
```

판정할 것:

- k6 오류율과 dropped iteration
- 중단 전 두 upstream이 모두 사용됐는지
- 중단 뒤 `8081`로 재시도됐는지
- 지연이 SLO 안에서 회복됐는지

두 번째 인스턴스는 14.2 명령으로 다시 실행한다.

이 로컬 실험은 Nginx의 단순 round-robin과 연결 실패 재시도를 확인한다. Kubernetes Service, managed load balancer, readiness 전파와 실제 다중 AZ를 증명하지 않는다. Prometheus 기본 설정은 `8081`만 수집하므로 두 인스턴스별 자원 비교도 후속 관측성 보강이 필요하다.

## 15. 결과 해석 체크리스트

- [ ] Git SHA가 실제 실행 코드와 일치한다.
- [ ] 작업 트리가 깨끗하거나 diff hash가 별도 기록됐다.
- [ ] CPU, 메모리, JVM과 DB 사양이 기록됐다.
- [ ] 데이터 행 수와 요청 payload가 기록됐다.
- [ ] Gateway 경유인지 서비스 직접 호출인지 기록됐다.
- [ ] 다른 백그라운드 프로그램이 결과에 영향을 줄 수 있는지 기록됐다.
- [ ] p95/p99, 오류율과 dropped iteration을 함께 확인했다.
- [ ] Grafana에서 CPU, heap, GC와 DB pool을 같은 시간대로 확인했다.
- [ ] 장애 실험은 중단 시각, 재시작 시각과 복구 확인을 기록했다.
- [ ] 한 번의 PASS를 다른 환경의 보장값으로 복사하지 않았다.

## 16. 종료와 결과 보관

부하가 끝나면 JSON과 Markdown을 안전한 artifact 저장소에 복사한다. 그다음 서비스 터미널에서 `Ctrl+C`를 누르고 Compose를 내린다.

```powershell
docker compose --env-file infra/.env.versions -f infra/compose.yml `
  --profile database-ha --profile observability down
```

DB와 Grafana volume은 보존된다. `down -v`는 측정 데이터와 로컬 DB를 삭제하므로 초기화 의도가 분명할 때만 사용한다.

## 17. 시나리오 자체의 자동 테스트

이 명령은 실제 부하를 만들지 않는다. 단계 계산, 안전장치, p99 결과 계약, dashboard와 보고서 생성을 검증한다.

```powershell
cd load-tests
npm test
cd ..
```

실제 측정 기록과 비보장 범위는 [검증 기록](verification.md)을 확인한다.

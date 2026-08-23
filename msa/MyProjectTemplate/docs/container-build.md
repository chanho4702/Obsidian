# 서비스 컨테이너 빌드

루트 `Dockerfile.service` 하나가 `SERVICE` build argument로 서비스 모듈을 선택한다.

```bash
docker build -f Dockerfile.service --build-arg SERVICE=sample-service -t sample-service:local .
```

운영에서는 `latest` 대신 Git SHA와 image digest를 사용한다. 런타임 이미지는 non-root 사용자로 실행하며, Secret과 Spring profile은 이미지 실행 시 주입한다.

서비스별 Dockerfile을 복제하지 않는 이유는 JDK, 보안 사용자, JVM 기준선이 서비스마다 드리프트하는 것을 막기 위해서다. 네이티브 라이브러리나 별도 런타임이 필요한 서비스만 독립 Dockerfile을 가진다.
